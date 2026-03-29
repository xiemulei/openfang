# 第 6 节：Agent 循环 — 上下文管理

> **版本**: v0.5.2 (2026-03-29)
> **核心文件**:
> - `crates/openfang-runtime/src/context_overflow.rs`
> - `crates/openfang-runtime/src/context_budget.rs` (v0.5.2 新增)
> - `crates/openfang-runtime/src/session_repair.rs`

## 学习目标

- [ ] 理解上下文溢出恢复的 4 个阶段
- [ ] 掌握 Session Repair 的验证和修复逻辑
- [ ] 理解工具结果修剪机制
- [ ] 掌握 ContextBudget 的工作原理

---

## 1. RecoveryStage — 恢复阶段枚举

### 文件位置
`crates/openfang-runtime/src/context_overflow.rs:16-28`

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum RecoveryStage {
    /// 无需恢复
    None,
    /// Stage 1: 适度修剪（保留最近 10 条）
    AutoCompaction { removed: usize },
    /// Stage 2: 激进修剪（保留最近 4 条）
    OverflowCompaction { removed: usize },
    /// Stage 3: 截断工具结果
    ToolResultTruncation { truncated: usize },
    /// Stage 4: 无法恢复 — 建议 /reset 或 /compact
    FinalError,
}
```

---

## 2. recover_from_overflow — 4 阶段恢复管道

### 文件位置
`crates/openfang-runtime/src/context_overflow.rs:38`

### 阶段 0: 估算 token 并判断是否需要恢复

```rust
// context_overflow.rs:44-51
let estimated = estimate_tokens(messages, system_prompt, tools);
let threshold_70 = (context_window as f64 * 0.70) as usize;
let threshold_90 = (context_window as f64 * 0.90) as usize;

// 无需恢复
if estimated <= threshold_70 {
    return RecoveryStage::None;
}
```

**阈值设计**：
- **70%**：触发恢复的安全线
- **90%**：触发激进恢复的警告线

### 阶段 1: 适度修剪（Moderate Trim）

```rust
// context_overflow.rs:53-70
// Stage 1: Moderate trim — keep last 10 messages
if estimated <= threshold_90 {
    let keep = 10.min(messages.len());
    let remove = messages.len() - keep;
    if remove > 0 {
        debug!(
            estimated_tokens = estimated,
            removing = remove,
            "Stage 1: moderate trim to last {keep} messages"
        );
        messages.drain(..remove);

        // 重新检查
        let new_est = estimate_tokens(messages, system_prompt, tools);
        if new_est <= threshold_70 {
            return RecoveryStage::AutoCompaction { removed: remove };
        }
    }
}
```

**策略**：
- 保留最近 10 条消息
- 删除早期对话
- 如果仍然超出 70%，进入 Stage 2

### 阶段 2: 激进修剪（Aggressive Overflow Compaction）

```rust
// context_overflow.rs:72-95
// Stage 2: Aggressive trim — keep last 4 messages + summary marker
{
    let keep = 4.min(messages.len());
    let remove = messages.len() - keep;
    if remove > 0 {
        warn!(
            estimated_tokens = estimate_tokens(messages, system_prompt, tools),
            removing = remove,
            "Stage 2: aggressive overflow compaction to last {keep} messages"
        );

        // 添加系统摘要消息
        let summary = Message::user(format!(
            "[System: {} earlier messages were removed due to context overflow. \
             The conversation continues from here. Use /compact for smarter summarization.]",
            remove
        ));
        messages.drain(..remove);
        messages.insert(0, summary);

        let new_est = estimate_tokens(messages, system_prompt, tools);
        if new_est <= threshold_90 {
            return RecoveryStage::OverflowCompaction { removed: remove };
        }
    }
}
```

**策略**：
- 仅保留最近 4 条消息
- 插入系统摘要消息（解释删除了多少条）
- 如果仍然超出 90%，进入 Stage 3

### 阶段 3: 截断工具结果（Tool Result Truncation）

```rust
// context_overflow.rs:97-132
// Stage 3: Truncate all historical tool results to 2K chars
let tool_truncation_limit = 2000;
let mut truncated = 0;

for msg in messages.iter_mut() {
    if let MessageContent::Blocks(blocks) = &mut msg.content {
        for block in blocks.iter_mut() {
            if let ContentBlock::ToolResult { content, .. } = block {
                if content.len() > tool_truncation_limit {
                    let mut safe_keep = tool_truncation_limit.saturating_sub(80);
                    // 回退到有效的 char boundary
                    while safe_keep > 0 && !content.is_char_boundary(safe_keep) {
                        safe_keep -= 1;
                    }
                    *content = format!(
                        "{}\n\n[OVERFLOW RECOVERY: truncated from {} to {} chars]",
                        &content[..safe_keep],
                        content.len(),
                        safe_keep
                    );
                    truncated += 1;
                }
            }
        }
    }
}

if truncated > 0 {
    let new_est = estimate_tokens(messages, system_prompt, tools);
    if new_est <= threshold_90 {
        return RecoveryStage::ToolResultTruncation { truncated };
    }
    warn!(
        estimated_tokens = new_est,
        "Stage 3 truncated {} tool results but still over threshold", truncated
    );
}
```

**策略**：
- 将所有工具结果截断到 2K 字符
- 保留头部和尾部（80 字符缓冲区用于恢复标记）
- 添加截断标记说明

### 阶段 4: 最终错误（Final Error）

```rust
// context_overflow.rs:134-137
// Stage 4: Final error — nothing more we can do automatically
warn!("Stage 4: all recovery stages exhausted, context still too large");
RecoveryStage::FinalError
```

**用户建议**：
- 使用 `/reset` 清空会话
- 使用 `/compact` 触发 LLM 总结

---

## 3. Session Repair — 会话验证和修复

### 文件位置
`crates/openfang-runtime/src/session_repair.rs:49`

### RepairStats — 修复统计

```rust
// session_repair.rs:19-33
#[derive(Debug, Clone, Default, PartialEq, Eq)]
pub struct RepairStats {
    /// 删除的孤立 ToolResult 数量
    pub orphaned_results_removed: usize,
    /// 删除的空消息数量
    pub empty_messages_removed: usize,
    /// 合并的连续同角色消息数量
    pub messages_merged: usize,
    /// 重新排序的 ToolResult 数量
    pub results_reordered: usize,
    /// 插入的 synthetic error 结果数量
    pub synthetic_results_inserted: usize,
    /// 删除的重复 ToolResult 数量
    pub duplicates_removed: usize,
}
```

### Phase 1: 收集 ToolUse IDs

```rust
// session_repair.rs:52-65
let tool_use_ids: HashSet<String> = messages
    .iter()
    .flat_map(|m| match &m.content {
        MessageContent::Blocks(blocks) => blocks
            .iter()
            .filter_map(|b| match b {
                ContentBlock::ToolUse { id, .. } => Some(id.clone()),
                _ => None,
            })
            .collect::<Vec<_>>(),
        _ => vec![],
    })
    .collect();
```

### Phase 2: 过滤孤立 ToolResult 和空消息

```rust
// session_repair.rs:67-114
let mut cleaned: Vec<Message> = Vec::with_capacity(messages.len());
for msg in messages {
    let new_content = match &msg.content {
        MessageContent::Text(s) => {
            if s.is_empty() {
                stats.empty_messages_removed += 1;
                continue;
            }
            MessageContent::Text(s.clone())
        }
        MessageContent::Blocks(blocks) => {
            let original_len = blocks.len();
            let filtered: Vec<ContentBlock> = blocks
                .iter()
                .filter(|b| match b {
                    ContentBlock::ToolResult { tool_use_id, .. } => {
                        let keep = tool_use_ids.contains(tool_use_id);
                        if !keep {
                            stats.orphaned_results_removed += 1;
                        }
                        keep
                    }
                    _ => true,
                })
                .cloned()
                .collect();
            if filtered.is_empty() {
                if original_len > 0 {
                    debug!("Dropped message: all blocks filtered out");
                }
                stats.empty_messages_removed += 1;
                continue;
            }
            MessageContent::Blocks(filtered)
        }
    };
    cleaned.push(Message {
        role: msg.role,
        content: new_content,
    });
}
```

**修复内容**：
- 删除孤立的 `ToolResult`（没有对应的 `ToolUse`）
- 删除空消息

### Phase 2b: 重新排序错位的 ToolResult

```rust
// session_repair.rs:116-118
let reordered_count = reorder_tool_results(&mut cleaned);
stats.results_reordered = reordered_count;
```

**逻辑**（`reorder_tool_results` 函数）：
1. 构建 `tool_use_id → assistant message index` 映射
2. 对于每个包含 `ToolResult` 的 user 消息
3. 检查前一条消息是否是对应的 assistant 消息
4. 如果不是，将 `ToolResult` 移动到正确位置

### Phase 2c: 插入 Synthetic Error 结果

```rust
// session_repair.rs:120-122
let synthetic_count = insert_synthetic_results(&mut cleaned);
stats.synthetic_results_inserted = synthetic_count;
```

**逻辑**（`insert_synthetic_results` 函数）：
- 遍历所有 `ToolUse` 块
- 如果没有找到对应的 `ToolResult`
- 插入 synthetic error 结果：`"Tool execution was interrupted"`

### Phase 2d: 删除重复 ToolResult

```rust
// session_repair.rs:124-126
let dedup_count = deduplicate_tool_results(&mut cleaned);
stats.duplicates_removed = dedup_count;
```

**逻辑**（`deduplicate_tool_results` 函数）：
- 追踪已见过的 `tool_use_id`
- 删除重复的 `ToolResult`（保留第一个）

### Phase 3: 合并连续同角色消息

```rust
// session_repair.rs:143-163
let pre_merge_len = cleaned.len();
let mut merged: Vec<Message> = Vec::with_capacity(cleaned.len());
for msg in cleaned {
    if let Some(last) = merged.last_mut() {
        if last.role == msg.role {
            merge_content(&mut last.content, msg.content);
            stats.messages_merged += 1;
            continue;
        }
    }
    merged.push(msg);
}
```

**原因**：
- Anthropic API 要求消息角色交替（user/assistant 交替）
- 合并连续的同角色消息避免 API 错误

### merge_content — 合并内容

```rust
// session_repair.rs
fn merge_content(first: &mut MessageContent, second: MessageContent) {
    match (first, second) {
        (MessageContent::Text(s1), MessageContent::Text(s2)) => {
            *s1 = format!("{}\n{}", s1, s2);
        }
        (MessageContent::Blocks(blocks1), MessageContent::Blocks(blocks2)) => {
            blocks1.extend(blocks2);
        }
        _ => {}
    }
}
```

---

## 4. ContextBudget — 上下文预算守卫

### 文件位置
`crates/openfang-runtime/src/context_budget.rs` (355 行)

### 4.1 结构体定义

```rust
// context_budget.rs:14-21
pub struct ContextBudget {
    pub context_window_tokens: usize,    // 上下文窗口大小（token 数）
    pub tool_chars_per_token: f64,       // 工具结果字符/token 换算系数 (默认 2.0)
    pub general_chars_per_token: f64,    // 一般文本字符/token 换算系数 (默认 4.0)
}

impl ContextBudget {
    pub fn new(context_window_tokens: usize) -> Self {
        Self {
            context_window_tokens,
            tool_chars_per_token: 2.0,
            general_chars_per_token: 4.0,
        }
    }
}
```

### 4.2 预算策略

| 约束类型 | 比例/限制 | 说明 |
|----------|-----------|------|
| **单结果上限** | 50% context window | 单个工具结果不超过窗口 50% |
| **总结果上限** | 75% context window | 所有工具结果总和不超 75% |
| **每结果上限** | 30% context window | 每个工具结果默认上限（可调整） |
| **压缩后目标** | 2048 字符 | 超长结果压缩到 2K 字符 |

**关键方法**：
```rust
// context_budget.rs:33-44
pub fn per_result_cap(&self) -> usize {
    self.context_window_tokens * 3 / 10  // 30%
}

pub fn single_result_max(&self) -> usize {
    self.context_window_tokens / 2  // 50%
}

pub fn total_tool_headroom_chars(&self) -> usize {
    (self.context_window_tokens * 75 / 100) as usize * self.tool_chars_per_token as usize  // 75%
}
```

### 4.3 apply_context_guard — 两轮扫描守卫

**第一轮**：截断超过 50% 上下文窗口的单个工具结果
**第二轮**：从最早的结果开始，压缩到 2K 字符

```rust
// context_budget.rs:100-198
pub fn apply_context_guard(
    messages: &mut [openfang_types::Message],
    budget: &ContextBudget,
) -> usize {
    let mut compressed_count = 0;

    // === 第一轮：扫描所有工具结果 ===
    for msg in messages.iter_mut() {
        if let openfang_types::MessageContent::Blocks(blocks) = &mut msg.content {
            for block in blocks.iter_mut() {
                if let ContentBlock::ToolResult { content, .. } = block {
                    // 估算字符数（假设工具输出密集）
                    let char_count = content.chars().count();
                    let token_est = (char_count as f64 / budget.tool_chars_per_token).ceil() as usize;

                    // 检查是否超过 50% 窗口
                    if token_est > budget.single_result_max() {
                        let target_chars = (budget.single_result_max() as f64 * budget.tool_chars_per_token) as usize;
                        if char_count > target_chars {
                            *content = format!("{}...[truncated: {}/{} chars]",
                                &content[..target_chars.min(content.len())], target_chars, char_count);
                            compressed_count += 1;
                        }
                    }
                }
            }
        }
    }

    // === 第二轮：从最早的结果开始压缩到 2K 字符 ===
    let total_headroom = budget.total_tool_headroom_chars();
    let mut current_tool_chars = 0_usize;

    // 反向遍历（最早的消息在前）
    for msg in messages.iter_mut() {
        if let openfang_types::MessageContent::Blocks(blocks) = &mut msg.content {
            for block in blocks.iter_mut() {
                if let ContentBlock::ToolResult { content, .. } = block {
                    let char_count = content.chars().count();

                    if current_tool_chars + char_count > total_headroom {
                        // 需要压缩
                        let remaining = total_headroom.saturating_sub(current_tool_chars);
                        let target = remaining.min(2048);  // 压缩到最多 2K 字符
                        if char_count > target {
                            *content = format!("{}...[compressed: {}/{}]",
                                &content[..target.min(content.len())], target, char_count);
                            compressed_count += 1;
                        }
                    }

                    current_tool_chars += char_count;
                }
            }
        }
    }

    compressed_count
}
```

**返回值**：被压缩的工具结果数量（用于监控）

---

## 5. 恢复流程图

```mermaid
flowchart TD
    Start[recover_from_overflow] --> Estimate[估算 token 使用]
    Estimate --> Check70{estimated <= 70%?}
    Check70 -->|是 | ReturnNone[返回 RecoveryStage::None]
    Check70 -->|否 | Check90{estimated <= 90%?}

    Check90 -->|是 | Stage1[Stage 1: 适度修剪]
    Stage1 --> Keep10[保留最近 10 条消息]
    Keep10 --> Recheck1{new_est <= 70%?}
    Recheck1 -->|是 | Return1[返回 AutoCompaction]
    Recheck1 -->|否 | Stage2

    Check90 -->|否 | Stage2[Stage 2: 激进修剪]
    Stage2 --> Keep4[保留最近 4 条消息]
    Keep4 --> InsertSummary[插入系统摘要消息]
    InsertSummary --> Recheck2{new_est <= 90%?}
    Recheck2 -->|是 | Return2[返回 OverflowCompaction]
    Recheck2 -->|否 | Stage3

    Stage2 --> Stage3[Stage 3: 截断工具结果]
    Stage3 --> Truncate2K[截断到 2K 字符]
    Truncate2K --> AddMarker[添加截断标记]
    AddMarker --> Recheck3{new_est <= 90%?}
    Recheck3 -->|是 | Return3[返回 ToolResultTruncation]
    Recheck3 -->|否 | Stage4

    Stage3 --> Stage4[Stage 4: 最终错误]
    Stage4 --> LogWarn[记录警告日志]
    LogWarn --> ReturnFinal[返回 FinalError]

    Return1 --> End
    Return2 --> End
    Return3 --> End
    ReturnNone --> End
    ReturnFinal --> End
```

---

## 6. validate_and_repair 调用时机

在主循环中的调用位置：

```rust
// 1. 初始化阶段：添加用户消息后
let mut messages = crate::session_repair::validate_and_repair(&llm_messages);

// 2. 修剪过长历史后
messages.drain(..trim_count);
messages = crate::session_repair::validate_and_repair(&messages);

// 3. 溢出恢复后
let recovery = recover_from_overflow(&mut messages, ...);
if recovery != RecoveryStage::None {
    messages = crate::session_repair::validate_and_repair(&messages);
}

// 4. 空响应重试前
if is_silent_failure {
    messages = crate::session_repair::validate_and_repair(&messages);
}
```

---

## 7. 关键设计点

### 7.1 渐进式恢复策略

```
Stage 1 (70%): 适度修剪 → Stage 2 (90%): 激进修剪 → Stage 3: 截断工具 → Stage 4: 报错
```

**优点**：
- 先尝试低成本恢复（删除消息）
- 再尝试中等成本恢复（截断工具结果）
- 最后才报错（用户体验最差）

### 7.2 会话修复多相位验证

```
Phase 1: 收集 ToolUse IDs
Phase 2: 过滤孤立 ToolResult + 空消息
Phase 2b: 重新排序错位 ToolResult
Phase 2c: 插入 Synthetic Error 结果
Phase 2d: 删除重复 ToolResult
Phase 3: 合并连续同角色消息
```

**优点**：
- 每个阶段解决一类问题
- 统计数据帮助调试
- 保证最终消息列表合法

### 7.3 Token 估算启发式

```rust
fn estimate_tokens(messages, system_prompt, tools) -> usize {
    crate::compactor::estimate_token_count(messages, Some(system_prompt), Some(tools))
}

// compactor.rs 中的实现（chars/4 启发式）
fn estimate_token_count(...) -> usize {
    let total_chars = ...;
    total_chars / 4  // 大约 4 字符 = 1 token
}
```

**优点**：
- 快速估算（无需调用 LLM）
- 足够准确用于触发恢复决策

---

## 完成检查清单

- [ ] 理解上下文溢出恢复的 4 个阶段
- [ ] 掌握 Session Repair 的验证和修复逻辑
- [ ] 理解工具结果修剪机制
- [ ] 掌握 ContextBudget 的工作原理

---

## 下一步

前往 [第 7 节：Agent 循环 — 错误处理](./07-agent-loop-errors.md)

---

*创建时间：2026-03-15 (更新于 2026-03-29 v0.5.2)*
*OpenFang v0.5.2*
