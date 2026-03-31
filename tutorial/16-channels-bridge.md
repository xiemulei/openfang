# 第 16 节：Channel 系统 — 消息渠道

> **版本**: v0.5.2 (2026-03-29)
> **核心文件**:
> - `crates/openfang-channels/src/types.rs`
> - `crates/openfang-channels/src/bridge.rs`
> - `crates/openfang-channels/src/router.rs`
> - `crates/openfang-channels/src/formatter.rs`
> - `crates/openfang-channels/src/matrix.rs` (v0.5.1: 新增 auto_accept_invites 配置)
> - `crates/openfang-channels/src/mqtt.rs` (v0.5.2 新增)
> - `crates/openfang-channels/src/wecom.rs` (v0.4.9 新增)
> - `crates/openfang-channels/src/dingtalk_stream.rs` (v0.4.9 新增)

## 学习目标

- [ ] 理解 ChannelAdapter trait 的设计
- [ ] 掌握 ChannelMessage 统一消息结构
- [ ] 理解 AgentRouter 路由机制
- [ ] 掌握 OutputFormat 消息格式化
- [ ] 了解 42+ 渠道适配器架构 (v0.4.9 新增企业微信、钉钉流式)
- [ ] 了解 MQTT Pub/Sub 适配器 (v0.5.2 新增)

---

## 1. ChannelType — 渠道类型枚举

### 文件位置
`crates/openfang-channels/src/types.rs:13-27`

```rust
/// The type of messaging channel.
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum ChannelType {
    Telegram,
    WhatsApp,
    Slack,
    Discord,
    Signal,
    Matrix,
    Email,
    Teams,
    Mattermost,
    WebChat,
    CLI,
    Custom(String),
}
```

### 渠道分类

| 分类 | 渠道 | 说明 |
|------|------|------|
| **即时通讯** | Telegram, WhatsApp, Signal | 个人/群组聊天 |
| **企业协作** | Slack, Teams, Mattermost | 企业团队沟通 |
| **社区平台** | Discord, Matrix | 社区/开源项目 |
| **传统渠道** | Email, WebChat, CLI | 邮件/网页/命令行 |
| **自定义** | Custom(String) | 扩展适配器 |

---

## 2. ChannelUser — 用户结构

### 文件位置
`crates/openfang-channels/src/types.rs:29-38`

```rust
/// A user on a messaging channel.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ChannelUser {
    /// Platform-specific user ID.
    pub platform_id: String,
    /// Human-readable display name.
    pub display_name: String,
    /// Optional mapping to an OpenFang user identity.
    pub openfang_user: Option<String>,
}
```

### 字段说明

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `platform_id` | `String` | 平台特定用户 ID | Telegram: "123456789" |
| `display_name` | `String` | 人类可读显示名称 | "Alice" |
| `openfang_user` | `Option<String>` | OpenFang 用户身份映射 | Some("user_alice") |

---

## 3. ChannelContent — 内容类型

### 文件位置
`crates/openfang-channels/src/types.rs:40-71`

```rust
/// Content types that can be received from a channel.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ChannelContent {
    Text(String),
    Image {
        url: String,
        caption: Option<String>,
    },
    File {
        url: String,
        filename: String,
    },
    /// Local file data (bytes read from disk).
    FileData {
        data: Vec<u8>,
        filename: String,
        mime_type: String,
    },
    Voice {
        url: String,
        duration_seconds: u32,
    },
    Location {
        lat: f64,
        lon: f64,
    },
    Command {
        name: String,
        args: Vec<String>,
    },
}
```

### 内容类型说明

| 类型 | 字段 | 说明 |
|------|------|------|
| **Text** | `String` | 纯文本消息 |
| **Image** | `url`, `caption` | 图片（带可选说明） |
| **File** | `url`, `filename` | 文件（远程 URL） |
| **FileData** | `data`, `filename`, `mime_type` | 本地文件数据 |
| **Voice** | `url`, `duration_seconds` | 语音消息 |
| **Location** | `lat`, `lon` | 地理位置 |
| **Command** | `name`, `args` | 命令（如 `/start arg1 arg2`） |

---

## 4. ChannelMessage — 统一消息结构

### 文件位置
`crates/openfang-channels/src/types.rs:73-96`

```rust
/// A unified message from any channel.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ChannelMessage {
    /// Which channel this came from.
    pub channel: ChannelType,
    /// Platform-specific message identifier.
    pub platform_message_id: String,
    /// Who sent this message.
    pub sender: ChannelUser,
    /// The message content.
    pub content: ChannelContent,
    /// Optional target agent (if routed directly).
    pub target_agent: Option<AgentId>,
    /// When the message was sent.
    pub timestamp: DateTime<Utc>,
    /// Whether this message is from a group chat (vs DM).
    #[serde(default)]
    pub is_group: bool,
    /// Thread ID for threaded conversations (platform-specific).
    #[serde(default)]
    pub thread_id: Option<String>,
    /// Arbitrary platform metadata.
    pub metadata: HashMap<String, serde_json::Value>,
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `channel` | `ChannelType` | 来源渠道类型 |
| `platform_message_id` | `String` | 平台消息 ID |
| `sender` | `ChannelUser` | 发送者信息 |
| `content` | `ChannelContent` | 消息内容 |
| `target_agent` | `Option<AgentId>` | 目标 Agent（直接路由） |
| `timestamp` | `DateTime<Utc>` | 消息时间戳 |
| `is_group` | `bool` | 是否群聊消息 |
| `thread_id` | `Option<String>` | 线程 ID（支持线程的平台） |
| `metadata` | `HashMap` | 平台特定元数据 |

---

## 5. ChannelAdapter — 适配器 trait

### 文件位置
`crates/openfang-channels/src/types.rs:215-280`

```rust
/// Trait that every channel adapter must implement.
#[async_trait]
pub trait ChannelAdapter: Send + Sync {
    /// Human-readable name of this adapter.
    fn name(&self) -> &str;

    /// The channel type this adapter handles.
    fn channel_type(&self) -> ChannelType;

    /// Start receiving messages. Returns a stream of incoming messages.
    async fn start(
        &self,
    ) -> Result<Pin<Box<dyn Stream<Item = ChannelMessage> + Send>>, Box<dyn std::error::Error>>;

    /// Send a response back to a user on this channel.
    async fn send(
        &self,
        user: &ChannelUser,
        content: ChannelContent,
    ) -> Result<(), Box<dyn std::error::Error>>;

    /// Send a typing indicator (optional — default no-op).
    async fn send_typing(&self, _user: &ChannelUser) -> Result<(), Box<dyn std::error::Error>> {
        Ok(())
    }

    /// Send a lifecycle reaction to a message (optional — default no-op).
    async fn send_reaction(
        &self,
        _user: &ChannelUser,
        _message_id: &str,
        _reaction: &LifecycleReaction,
    ) -> Result<(), Box<dyn std::error::Error>> {
        Ok(())
    }

    /// Stop the adapter and clean up resources.
    async fn stop(&self) -> Result<(), Box<dyn std::error::Error>>;

    /// Get the current health status of this adapter.
    fn status(&self) -> ChannelStatus {
        ChannelStatus::default()
    }

    /// Send a response as a thread reply (optional — default falls back to `send()`).
    async fn send_in_thread(
        &self,
        user: &ChannelUser,
        content: ChannelContent,
        _thread_id: &str,
    ) -> Result<(), Box<dyn std::error::Error>> {
        self.send(user, content).await
    }

    /// Whether to suppress error responses on public channels.
    fn suppress_error_responses(&self) -> bool {
        false
    }
}
```

### 方法分类

| 方法 | 必填 | 说明 |
|------|------|------|
| `name()` | 是 | 适配器名称 |
| `channel_type()` | 是 | 渠道类型 |
| `start()` | 是 | 启动接收消息流 |
| `send()` | 是 | 发送响应消息 |
| `stop()` | 是 | 停止并清理资源 |
| `send_typing()` | 否 | 发送输入中状态 |
| `send_reaction()` | 否 | 发送生命周期反应 |
| `status()` | 否 | 获取健康状态 |
| `send_in_thread()` | 否 | 发送线程回复 |
| `suppress_error_responses()` | 否 | 是否抑制错误响应 |

---

## 6. AgentPhase — 生命周期阶段

### 文件位置
`crates/openfang-channels/src/types.rs:98-127`

```rust
/// Agent lifecycle phase for UX indicators.
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum AgentPhase {
    /// Message is queued, waiting for agent.
    Queued,
    /// Agent is calling the LLM.
    Thinking,
    /// Agent is executing a tool.
    ToolUse {
        tool_name: String,
    },
    /// Agent is streaming tokens.
    Streaming,
    /// Agent finished successfully.
    Done,
    /// Agent encountered an error.
    Error,
}
```

### 阶段说明

| 阶段 | 说明 | 默认 emoji |
|------|------|-----------|
| `Queued` | 消息等待处理 | ⏳ |
| `Thinking` | Agent 调用 LLM | 🤔 |
| `ToolUse { tool_name }` | 执行工具中 | ⚙️ |
| `Streaming` | 流式输出 tokens | ✍️ |
| `Done` | 完成 | ✅ |
| `Error` | 遇到错误 | ❌ |

### default_phase_emoji

```rust
// types.rs:152-162
pub fn default_phase_emoji(phase: &AgentPhase) -> &'static str {
    match phase {
        AgentPhase::Queued => "\u{23F3}",                 // ⏳
        AgentPhase::Thinking => "\u{1F914}",              // 🤔
        AgentPhase::ToolUse { .. } => "\u{2699}\u{FE0F}", // ⚙️
        AgentPhase::Streaming => "\u{270D}\u{FE0F}",      // ✍️
        AgentPhase::Done => "\u{2705}",                   // ✅
        AgentPhase::Error => "\u{274C}",                  // ❌
    }
}
```

---

## 7. LifecycleReaction — 生命周期反应

### 文件位置
`crates/openfang-channels/src/types.rs:129-150`

```rust
/// Reaction to show in a channel (emoji-based).
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LifecycleReaction {
    /// The agent phase this reaction represents.
    pub phase: AgentPhase,
    /// Channel-appropriate emoji.
    pub emoji: String,
    /// Whether to remove the previous phase reaction.
    pub remove_previous: bool,
}

/// Hardcoded emoji allowlist for lifecycle reactions.
pub const ALLOWED_REACTION_EMOJI: &[&str] = &[
    "\u{1F914}",        // 🤔 thinking
    "\u{2699}\u{FE0F}", // ⚙️ tool_use
    "\u{270D}\u{FE0F}", // ✍️ streaming
    "\u{2705}",         // ✅ done
    "\u{274C}",         // ❌ error
    "\u{23F3}",         // ⏳ queued
    "\u{1F504}",        // 🔄 processing
    "\u{1F440}",        // 👀 looking
];
```

---

## 8. DeliveryReceipt — 送达回执

### 文件位置
`crates/openfang-channels/src/types.rs:164-193`

```rust
/// Delivery status for outbound messages.
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum DeliveryStatus {
    Sent,      // 已发送到渠道 API
    Delivered, // 已确认送达
    Failed,    // 发送失败
    BestEffort, // 尽力投递（无确认）
}

/// Receipt tracking outbound message delivery.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DeliveryReceipt {
    pub message_id: String,
    pub channel: String,
    pub recipient: String,
    pub status: DeliveryStatus,
    pub timestamp: DateTime<Utc>,
    pub error: Option<String>,
}
```

---

## 9. DeliveryTracker — 送达回执追踪器 (v0.5.5 新增)

### 文件位置
`crates/openfang-kernel/src/kernel.rs:166-270`

### 核心结构

```rust
/// Bounded in-memory delivery receipt tracker.
/// Stores up to `MAX_RECEIPTS` most recent delivery receipts per agent.
pub struct DeliveryTracker {
    receipts: dashmap::DashMap<AgentId, Vec<openfang_channels::types::DeliveryReceipt>>,
}
```

### 主要方法

| 方法 | 说明 | 参数 |
|------|------|------|
| `new()` | 创建新的追踪器 | - |
| `record()` | 记录送达回执 | `agent_id: AgentId`, `receipt: DeliveryReceipt` |
| `get_receipts()` | 获取最近的回执 | `agent_id: AgentId`, `limit: usize` |
| `sent_receipt()` | 创建已发送回执 | `channel: &str`, `recipient: &str` |
| `failed_receipt()` | 创建失败回执 | `channel: &str`, `recipient: &str`, `error: &str` |

### 实现细节

```rust
impl DeliveryTracker {
    const MAX_RECEIPTS: usize = 10_000;  // 全局上限
    const MAX_PER_AGENT: usize = 500;     // 每个 Agent 上限

    /// Record a delivery receipt for an agent.
    pub fn record(&self, agent_id: AgentId, receipt: openfang_channels::types::DeliveryReceipt) {
        let mut entry = self.receipts.entry(agent_id).or_default();
        entry.push(receipt);
        // Per-agent cap
        if entry.len() > Self::MAX_PER_AGENT {
            let drain = entry.len() - Self::MAX_PER_AGENT;
            entry.drain(..drain);
        }
        // Global cap: evict oldest agents' receipts if total exceeds limit
        drop(entry);
        let total: usize = self.receipts.iter().map(|e| e.value().len()).sum();
        if total > Self::MAX_RECEIPTS {
            // Simple eviction: remove oldest entries from first agent found
            if let Some(mut oldest) = self.receipts.iter_mut().next() {
                let to_remove = total - Self::MAX_RECEIPTS;
                let drain = to_remove.min(oldest.value().len());
                oldest.value_mut().drain(..drain);
            }
        }
    }

    /// Get recent delivery receipts for an agent (newest first).
    pub fn get_receipts(
        &self,
        agent_id: AgentId,
        limit: usize,
    ) -> Vec<openfang_channels::types::DeliveryReceipt> {
        self.receipts
            .get(&agent_id)
            .map(|entries| entries.iter().rev().take(limit).cloned().collect())
            .unwrap_or_default()
    }
}
```

### 安全特性

- **容量限制**: 全局最多 10,000 条回执，每个 Agent 最多 500 条
- **自动清理**: 超出限制时自动清理最旧的回执
- **数据脱敏**: `sanitize_recipient()` 方法避免 PII 日志泄露
- **错误处理**: 错误信息最大 256 字符，移除控制字符

### 使用场景

1. **消息状态跟踪**: 追踪消息的发送状态和错误信息
2. **故障排查**: 查看历史送达记录，诊断渠道问题
3. **统计分析**: 分析消息成功率和渠道性能
4. **用户反馈**: 向用户提供消息发送状态

### 集成点

- **ChannelAdapter**: 各渠道适配器发送消息后生成回执
- **BridgeManager**: 统一收集和管理送达回执
- **Kernel**: 通过 `delivery_tracker` 字段提供全局访问
- **API**: 提供查询送达状态的端点

---

## 9. ChannelStatus — 健康状态

### 文件位置
`crates/openfang-channels/src/types.rs:195-210`

```rust
/// Health status for a channel adapter.
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct ChannelStatus {
    /// Whether the adapter is currently connected/running.
    pub connected: bool,
    /// When the adapter was started (ISO 8601).
    pub started_at: Option<DateTime<Utc>>,
    /// When the last message was received.
    pub last_message_at: Option<DateTime<Utc>>,
    /// Total messages received since start.
    pub messages_received: u64,
    /// Total messages sent since start.
    pub messages_sent: u64,
    /// Last error message (if any).
    pub last_error: Option<String>,
}
```

---

## 10. AgentRouter — Agent 路由

### 文件位置
`crates/openfang-channels/src/router.rs:25-43`

```rust
/// Routes incoming messages to the correct agent.
///
/// Routing priority: bindings (most specific first) > direct routes > user defaults > system default.
pub struct AgentRouter {
    /// Default agent per user.
    user_defaults: DashMap<String, AgentId>,
    /// Direct routes: (channel_type_key, platform_user_id) -> AgentId.
    direct_routes: DashMap<(String, String), AgentId>,
    /// System-wide default agent.
    default_agent: Option<AgentId>,
    /// Per-channel-type default agent.
    channel_defaults: DashMap<String, AgentId>,
    /// Sorted bindings (most specific first).
    bindings: Mutex<Vec<(AgentBinding, String)>>,
    /// Broadcast configuration.
    broadcast: Mutex<BroadcastConfig>,
    /// Agent name -> AgentId cache.
    agent_name_cache: DashMap<String, AgentId>,
}
```

### resolve — 路由决策

```rust
// router.rs:110-159
pub fn resolve(
    &self,
    channel_type: &ChannelType,
    platform_user_id: &str,
    user_key: Option<&str>,
) -> Option<AgentId> {
    let channel_key = format!("{channel_type:?}");

    // 0. Check bindings (most specific first)
    let ctx = BindingContext {
        channel: channel_type_to_str(channel_type).to_string(),
        account_id: None,
        peer_id: platform_user_id.to_string(),
        guild_id: None,
        roles: Vec::new(),
    };
    if let Some(agent_id) = self.resolve_binding(&ctx) {
        return Some(agent_id);
    }

    // 1. Check direct routes
    if let Some(agent) = self
        .direct_routes
        .get(&(channel_key.clone(), platform_user_id.to_string()))
    {
        return Some(*agent);
    }

    // 2. Check user defaults
    if let Some(key) = user_key {
        if let Some(agent) = self.user_defaults.get(key) {
            return Some(*agent);
        }
    }
    if let Some(agent) = self.user_defaults.get(platform_user_id) {
        return Some(*agent);
    }

    // 3. Per-channel-type default
    if let Some(agent) = self.channel_defaults.get(&channel_key) {
        return Some(*agent);
    }

    // 4. System default
    self.default_agent
}
```

### 路由优先级

```
1. Bindings（最具体优先） ← 基于 channel/peer_id/guild_id/roles
2. Direct Routes（直接路由） ← (channel, user) 对
3. User Defaults（用户默认） ← 按用户 key
4. Channel Defaults（渠道默认） ← 按渠道类型
5. System Default（系统默认） ← 兜底
```

### BindingContext — 绑定上下文

```rust
// router.rs:10-23
pub struct BindingContext {
    pub channel: String,       // "telegram", "discord"
    pub account_id: Option<String>,  // Bot ID
    pub peer_id: String,       // User ID
    pub guild_id: Option<String>,  // Guild/Server ID
    pub roles: Vec<String>,    // User roles
}
```

### binding_matches — 绑定匹配逻辑

```rust
// router.rs:279-312
fn binding_matches(&self, binding: &AgentBinding, ctx: &BindingContext) -> bool {
    let rule = &binding.match_rule;

    // All specified fields must match
    if let Some(ref ch) = rule.channel {
        if ch != &ctx.channel { return false; }
    }
    if let Some(ref acc) = rule.account_id {
        if ctx.account_id.as_ref() != Some(acc) { return false; }
    }
    if let Some(ref pid) = rule.peer_id {
        if pid != &ctx.peer_id { return false; }
    }
    if let Some(ref gid) = rule.guild_id {
        if ctx.guild_id.as_ref() != Some(gid) { return false; }
    }
    if !rule.roles.is_empty() {
        // User must have at least one of the specified roles
        let has_role = rule.roles.iter().any(|r| ctx.roles.contains(r));
        if !has_role { return false; }
    }
    true
}
```

### specificity — 特异性评分

```rust
// openfang-types/config.rs (推断)
impl BindingMatchRule {
    pub fn specificity(&self) -> u32 {
        let mut score = 0;
        if self.channel.is_some() { score += 1; }
        if self.peer_id.is_some() { score += 2; }
        if self.guild_id.is_some() { score += 2; }
        if self.account_id.is_some() { score += 4; }
        score += self.roles.len() * 2;
        score
    }
}
```

---

## 11. OutputFormat — 输出格式

### 文件位置
`crates/openfang-types/src/config.rs`（推断）

```rust
/// Output format for channel messages.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "kebab-case")]
pub enum OutputFormat {
    /// Standard Markdown (passthrough).
    Markdown,
    /// Telegram HTML subset.
    TelegramHtml,
    /// Slack mrkdwn format.
    SlackMrkdwn,
    /// Plain text (strips formatting).
    PlainText,
}
```

---

## 12. format_for_channel — 消息格式化

### 文件位置
`crates/openfang-channels/src/formatter.rs:10-18`

```rust
/// Format a message for a specific channel output format.
pub fn format_for_channel(text: &str, format: OutputFormat) -> String {
    match format {
        OutputFormat::Markdown => text.to_string(),
        OutputFormat::TelegramHtml => markdown_to_telegram_html(text),
        OutputFormat::SlackMrkdwn => markdown_to_slack_mrkdwn(text),
        OutputFormat::PlainText => markdown_to_plain(text),
    }
}
```

### markdown_to_telegram_html — Telegram HTML 转换

```rust
// formatter.rs:20-146
fn markdown_to_telegram_html(text: &str) -> String {
    let normalized = text.replace("\r\n", "\n").replace('\r', "\n");
    let mut blocks = Vec::new();
    let lines: Vec<&str> = normalized.lines().collect();
    let mut i = 0;

    while i < lines.len() {
        let line = lines[i];
        let trimmed = line.trim();

        if trimmed.is_empty() {
            i += 1;
            continue;
        }

        // Fenced code block
        if let Some(fence) = fence_delimiter(trimmed) {
            // ... code block handling
            continue;
        }

        // Heading
        if let Some(content) = heading_text(trimmed) {
            blocks.push(format!("<b>{}</b>", render_inline_markdown(content)));
            continue;
        }

        // Blockquote
        if trimmed.starts_with('>') {
            // ... blockquote handling
            continue;
        }

        // List items and paragraphs...
        i += 1;
    }

    blocks.join("\n\n")
}
```

### render_inline_markdown — 行内标记渲染

```rust
// formatter.rs:148-224
fn render_inline_markdown(text: &str) -> String {
    let mut result = escape_html(text);

    // Links: [text](url) → <a href="url">text</a>
    while let Some(bracket_start) = result.find('[') {
        // ... link parsing
    }

    // Bold: **text** → <b>text</b>
    while let Some(start) = result.find("**") {
        // ... bold parsing
    }

    // Inline code: `text` → <code>text</code>
    while let Some(start) = result.find('`') {
        // ... code parsing
    }

    // Italic: *text* → <i>text</i>
    // ... italic parsing

    out
}
```

### 支持的平台格式

| 平台 | 格式 | 支持的标签 |
|------|------|-----------|
| **Telegram** | HTML | `<b>`, `<i>`, `<code>`, `<pre>`, `<a>`, `<blockquote>` |
| **Slack** | mrkdwn | `*bold*`, `_italic_`, `<url\|text>`, `` `code` `` |
| **Discord** | Markdown | 标准 Markdown |
| **Plain** | 纯文本 | 无格式 |

---

## 13. split_message — 消息分块

### 文件位置
`crates/openfang-channels/src/types.rs:282-309`

```rust
/// Split a message into chunks of at most `max_len` characters,
/// preferring to split at newline boundaries.
///
/// Shared utility used by Telegram, Discord, and Slack adapters.
pub fn split_message(text: &str, max_len: usize) -> Vec<&str> {
    if text.len() <= max_len {
        return vec![text];
    }
    let mut chunks = Vec::new();
    let mut remaining = text;
    while !remaining.is_empty() {
        if remaining.len() <= max_len {
            chunks.push(remaining);
            break;
        }
        let safe_end = openfang_types::truncate_str(remaining, max_len).len();
        let split_at = remaining[..safe_end].rfind('\n').unwrap_or(safe_end);
        let (chunk, rest) = remaining.split_at(split_at);
        chunks.push(chunk);
        remaining = rest
            .strip_prefix("\r\n")
            .or_else(|| rest.strip_prefix('\n'))
            .unwrap_or(rest);
    }
    chunks
}
```

### 用途

- Telegram: 单消息最多 4096 字符
- Discord: 单消息最多 2000 字符
- Slack: 单消息最多 4000 字符

---

## 14. 40 个渠道适配器

### 文件位置
`crates/openfang-channels/src/lib.rs`

### Wave 1 — 核心渠道（10 个）

| 适配器 | 模块 | 说明 |
|--------|------|------|
| Telegram | `telegram.rs` | Bot API 长轮询 |
| WhatsApp | `whatsapp.rs` | Business API |
| Slack | `slack.rs` | Socket Mode / Events API |
| Discord | `discord.rs` | Gateway WebSocket |
| Signal | `signal.rs` | signal-cli RPC |
| Matrix | `matrix.rs` | Client-Server API (v0.5.1: 新增 auto_accept_invites 配置) |
| Email | `email.rs` | IMAP/SMTP |
| Teams | `teams.rs` | Graph API |
| Mattermost | `mattermost.rs` | WebSocket API |
| WebChat | `webchat.rs` | 网页小部件 |

### Wave 2 — 高价值渠道（7 个）

| 适配器 | 模块 |
|--------|------|
| Bluesky | `bluesky.rs` |
| Feishu | `feishu.rs` |
| Line | `line.rs` |
| Mastodon | `mastodon.rs` |
| Messenger | `messenger.rs` |
| Reddit | `reddit.rs` |
| Revolt | `revolt.rs` |
| Viber | `viber.rs` |

### Wave 3 — 企业渠道（9 个，v0.4.9 更新）

| 适配器 | 模块 |
|--------|------|
| Flock | `flock.rs` |
| Guilded | `guilded.rs` |
| Keybase | `keybase.rs` |
| Nextcloud | `nextcloud.rs` |
| Nostr | `nostr.rs` |
| Pumble | `pumble.rs` |
| Threema | `threema.rs` |
| Twist | `twist.rs` |
| Webex | `webex.rs` |
| **企业微信** | `wecom.rs` (v0.4.9 新增) |

### Wave 4 — 小众渠道（8 个，v0.4.9 更新）

| 适配器 | 模块 |
|--------|------|
| **钉钉流式** | `dingtalk_stream.rs` (v0.4.9 新增) |
| Dingtalk | `dingtalk.rs` |
| Discourse | `discourse.rs` |
| Gitter | `gitter.rs` |
| Gotify | `gotify.rs` |
| LinkedIn | `linkedin.rs` |
| Mumble | `mumble.rs` |
| Ntfy | `ntfy.rs` |
| Webhook | `webhook.rs` |

### Wave 5 — 传统渠道（5 个）

| 适配器 | 模块 |
|--------|------|
| IRC | `irc.rs` |
| XMPP | `xmpp.rs` |
| Zulip | `zulip.rs` |
| RocketChat | `rocketchat.rs` |
| Twitch | `twitch.rs` |

### Wave 6 — Niche & Differentiating (v0.5.2 新增)

| 适配器 | 模块 |
|--------|------|
| **MQTT** | `mqtt.rs` (v0.5.2 新增) |

---

## 15. WeComAdapter — 企业微信适配器 (v0.4.9 新增)

### 文件位置
`crates/openfang-channels/src/wecom.rs` (691 行)

### 核心结构

```rust
// wecom.rs:1-50
pub struct WeComAdapter {
    corp_id: String,
    agent_id: String,
    secret: Zeroizing<String>,
    token: String,
    encoding_aes_key: Zeroizing<String>,
    client: reqwest::Client,
    access_token: Arc<RwLock<Option<String>>>,
    token_expires_at: Arc<RwLock<Option<i64>>>,
}
```

### Token 管理

```rust
// 获取访问 Token
const WECOM_TOKEN_URL: &str = "https://qyapi.weixin.qq.com/cgi-bin/gettoken";

async fn refresh_access_token(&self) -> Result<String, String> {
    let url = format!(
        "{}?corpid={}&corpsecret={}",
        WECOM_TOKEN_URL, self.corp_id, self.secret
    );

    let resp = self.client.get(&url).send().await?;
    let json: serde_json::Value = resp.json().await?;

    // 缓存 Token，过期前自动刷新
    let access_token = json["access_token"].as_str().unwrap().to_string();
    let expires_in = json["expires_in"].as_i64().unwrap_or(7200);

    *self.access_token.write().await = Some(access_token.clone());
    *self.token_expires_at.write().await = Some(current_timestamp() + expires_in - 300);

    Ok(access_token)
}
```

### 消息发送

```rust
// 发送文本消息
const WECOM_SEND_URL: &str = "https://qyapi.weixin.qq.com/cgi-bin/message/send";

pub async fn send_text(&self, to_user: &str, content: &str) -> Result<(), String> {
    let access_token = self.get_access_token().await?;

    let payload = serde_json::json!({
        "touser": to_user,
        "msgtype": "text",
        "agentid": self.agent_id,
        "text": {
            "content": content
        },
        "safe": 0
    });

    let url = format!("{}?access_token={}", WECOM_SEND_URL, access_token);
    self.client.post(&url).json(&payload).send().await?;

    Ok(())
}
```

### AES-CBC 解密

```rust
// 解密微信加密消息
fn decrypt_aes_cbc(key: &[u8], encrypted_base64: &str) -> Result<Vec<u8>, String> {
    let key = GenericKey::<Aes256, _>::from_slice(key);
    let encrypted = base64_decode(encrypted_base64)?;

    // PKCS#7 去填充
    let cipher = cbc::Decryptor::<Aes256CBC>::new(&key, &iv);
    let decrypted = cipher.decrypt_padded_vec_mut::<Pkcs7>(&encrypted)?;

    Ok(decrypted)
}
```

### 配置示例

```toml
# ~/.openfang/config.toml
[wecom]
corp_id = "ww1234567890"
agent_id = "1000001"
secret = "your-agent-secret"
token = "webhook-token"
encoding_aes_key = "your-aes-key-here"
```

---

## 16. DingTalkStreamAdapter — 钉钉流式适配器 (v0.4.9 新增)

### 文件位置
`crates/openfang-channels/src/dingtalk_stream.rs` (600 行)

### 核心特性

- **流式处理**: 支持钉钉卡片消息的流式更新
- **交互式卡片**: 支持按钮、表单等交互元素
- **回调机制**: 处理用户点击、提交等事件

### 卡片消息示例

```rust
// 发送交互式卡片
pub async fn send_card(&self, to_user: &str, card: DingTalkCard) -> Result<(), String> {
    let access_token = self.get_access_token().await?;

    let payload = serde_json::json!({
        "user_id": to_user,
        "msgtype": "interactive_card",
        "card": {
            "card_type": card.card_type,
            "card_content": card.content,
            "actions": card.actions
        }
    });

    // 发送卡片
    let resp = self.client.post(&send_url)
        .header("Content-Type", "application/json")
        .json(&payload)
        .send()
        .await?;

    Ok(())
}
```

### 回调处理

```rust
// 处理卡片回调
pub async fn handle_callback(
    &self,
    callback: DingTalkCallback
) -> Result<CallbackResponse, String> {
    match callback.action_type.as_str() {
        "click" => self.handle_click(callback).await,
        "submit" => self.handle_submit(callback).await,
        _ => Err("Unknown action type".into()),
    }
}
```

---

## 17. TelegramAdapter 示例

### 文件位置
`crates/openfang-channels/src/telegram.rs`

### 结构定义

```rust
// telegram.rs:30-50
pub struct TelegramAdapter {
    token: Zeroizing<String>,  // SECURITY: 退出时清零
    client: reqwest::Client,
    allowed_users: Vec<String>,
    poll_interval: Duration,
    api_base_url: String,
    bot_username: Arc<tokio::sync::RwLock<Option<String>>>,
    shutdown_tx: Arc<watch::Sender<bool>>,
    shutdown_rx: watch::Receiver<bool>,
}
```

### start — 启动长轮询

```rust
// telegram.rs (推断)
async fn start(&self) -> Result<Pin<Box<dyn Stream<Item = ChannelMessage> + Send>>, Box<dyn std::error::Error>> {
    // 1. 调用 getMe 获取 bot 用户名
    let me = self.get_me().await?;
    *self.bot_username.write().await = Some(me.username);

    // 2. 创建消息流
    let (tx, rx) = mpsc::channel(100);

    // 3. 启动轮询任务
    let token = self.token.clone();
    let client = self.client.clone();
    tokio::spawn(async move {
        let mut offset = 0i64;
        loop {
            match self.get_updates(&client, &token, offset).await {
                Ok(updates) => {
                    for update in updates {
                        offset = update.update_id + 1;
                        if let Some(message) = update.message {
                            let channel_msg = self.parse_message(message)?;
                            let _ = tx.send(channel_msg).await;
                        }
                    }
                }
                Err(e) => {
                    // 指数退避
                    tokio::time::sleep(INITIAL_BACKOFF).await;
                }
            }
        }
    });

    Ok(Box::pin(tokio_stream::wrappers::ReceiverStream::new(rx)))
}
```

### send — 发送消息

```rust
// telegram.rs (推断)
async fn send(&self, user: &ChannelUser, content: ChannelContent) -> Result<(), Box<dyn std::error::Error>> {
    match content {
        ChannelContent::Text(text) => {
            // 分割长消息
            let chunks = split_message(&text, 4096);
            for chunk in chunks {
                let formatted = format_for_channel(chunk, OutputFormat::TelegramHtml);
                self.send_message(&user.platform_id, &formatted).await?;
            }
        }
        ChannelContent::Image { url, caption } => {
            self.send_photo(&user.platform_id, &url, caption.as_deref()).await?;
        }
        // ... 其他内容类型
    }
    Ok(())
}
```

### suppress_error_responses — 公共渠道保护

```rust
// telegram.rs (推断)
fn suppress_error_responses(&self) -> bool {
    // Telegram 公共频道不发送错误消息
    // 避免污染公共时间线
    true
}
```

---

## 16. BridgeManager — 桥接管理器

### 文件位置
`crates/openfang-channels/src/bridge.rs`

```rust
/// Manages channel adapters and dispatches messages.
pub struct BridgeManager {
    /// Running adapters by instance ID.
    adapters: DashMap<String, Arc<dyn ChannelAdapter>>,
    /// Router for incoming messages.
    router: Arc<AgentRouter>,
    /// Kernel handle for sending messages.
    kernel: Arc<dyn ChannelBridgeHandle>,
    /// Output format per channel.
    formats: DashMap<ChannelType, OutputFormat>,
    /// Rate limiters per channel/user.
    rate_limiters: DashMap<String, RateLimiter>,
}
```

### dispatch_message — 消息分发

```rust
// bridge.rs (推断)
async fn dispatch_message(&self, msg: ChannelMessage) {
    // 1. 路由决策
    let agent_id = match self.router.resolve(
        &msg.channel,
        &msg.sender.platform_id,
        msg.sender.openfang_user.as_deref(),
    ) {
        Some(id) => id,
        None => {
            warn!("No agent route for message from {}", msg.sender.display_name);
            return;
        }
    };

    // 2. 发送消息到 kernel
    let response = match self.kernel.send_message(agent_id, &msg_text).await {
        Ok(r) => r,
        Err(e) => {
            error!("Failed to send message to agent: {}", e);
            return;
        }
    };

    // 3. 获取适配器发送响应
    let adapter = self.adapters.get(&adapter_key);
    if let Some(adapter) = adapter {
        let formatted = format_for_channel(&response, self.get_format(msg.channel));
        let _ = adapter.send(&msg.sender, ChannelContent::Text(formatted)).await;
    }
}
```

---

## 17. 关键设计点

### 17.1 统一消息模型

```
┌─────────────────┐
│  Telegram       │
│  WhatsApp       │
│  Slack          │
│  Discord        │
│  ... (40 个)     │
└────────┬────────┘
         │
         ↓
┌─────────────────┐
│  ChannelMessage │ ← 统一模型
└────────┬────────┘
         │
         ↓
┌─────────────────┐
│   Agent Router  │
└─────────────────┘
```

### 17.2 适配器分层

| 层 | 职责 |
|----|------|
| **ChannelAdapter** | trait 定义接口 |
| **具体适配器** | TelegramAdapter, SlackAdapter 等 |
| **BridgeManager** | 管理适配器生命周期 |
| **AgentRouter** | 消息路由决策 |
| **Formatter** | 格式转换 |

### 17.3 路由优先级

```
最具体 → 最通用
bindings → direct_routes → user_defaults → channel_defaults → system_default
```

### 17.4 格式化管道

```
Agent 响应 (Markdown)
    ↓
format_for_channel()
    ↓
Telegram HTML / Slack mrkdwn / Plain
    ↓
split_message() (如需要)
    ↓
发送多渠道
```

---

## 18. v0.5.1 新增：Matrix auto_accept_invites 配置

### 配置项

v0.5.1 为 Matrix 渠道添加了 `auto_accept_invites` 配置项，允许控制是否自动接受房间邀请。

**配置文件位置**: `~/.openfang/config.toml`

```toml
[channels.matrix]
enabled = true
home_server = "matrix.org"
user_id = "@bot:matrix.org"
access_token_env = "MATRIX_ACCESS_TOKEN"
allowed_rooms = []  # 留空表示允许所有房间
auto_accept_invites = false  # v0.5.1 新增，默认 false
default_agent = "assistant"
```

### 配置说明

| 值 | 说明 |
|-----|------|
| `true` | 自动接受所有房间邀请 |
| `false` | 仅响应已配置 `allowed_rooms` 中的房间 |

### 代码实现

**MatrixConfig 结构** (`crates/openfang-types/src/config.rs:1861-1883`):

```rust
pub struct MatrixConfig {
    pub home_server: String,
    pub user_id: String,
    pub access_token_env: String,
    pub allowed_rooms: Vec<String>,
    pub default_agent: Option<String>,
    /// Whether to auto-accept room invites (default: false).
    #[serde(default)]
    pub auto_accept_invites: bool,  // v0.5.1 新增
    pub overrides: ChannelOverrides,
}

impl Default for MatrixConfig {
    fn default() -> Self {
        Self {
            home_server: "matrix.org".to_string(),
            user_id: "@bot:matrix.org".to_string(),
            access_token_env: "MATRIX_ACCESS_TOKEN".to_string(),
            allowed_rooms: vec![],
            default_agent: None,
            auto_accept_invites: false,  // 默认不自动接受
            overrides: ChannelOverrides::default(),
        }
    }
}
```

**MatrixAdapter 构造函数** (`crates/openfang-channels/src/matrix.rs`):

```rust
impl MatrixAdapter {
    pub fn new(
        home_server: String,
        user_id: String,
        access_token: String,
        allowed_rooms: Vec<String>,
        auto_accept_invites: bool,  // v0.5.1 新增参数
    ) -> Self {
        let (shutdown_tx, shutdown_rx) = watch::channel(false);
        Self {
            home_server,
            user_id,
            access_token,
            allowed_rooms,
            auto_accept_invites,  // 保存配置
            shutdown_tx: Arc::new(shutdown_tx),
            shutdown_rx,
            since_token: Arc::new(RwLock::new(None)),
        }
    }
}
```

**Bridge 集成** (`crates/openfang-api/src/channel_bridge.rs:1219`):

```rust
// 启动 Matrix 渠道时传递配置
if matrix_enabled && !mx_config.access_token_env.is_empty() {
    if let Some(token) = std::env::var(&mx_config.access_token_env).ok() {
        adapters.push((
            Arc::new(MatrixAdapter::new(
                mx_config.home_server.clone(),
                mx_config.user_id.clone(),
                token,
                mx_config.allowed_rooms.clone(),
                mx_config.auto_accept_invites,  // v0.5.1 传递配置
            )),
            mx_config.default_agent.clone(),
        ));
    }
}
```

### 使用场景

**场景 1: 公共客服机器人**
```toml
auto_accept_invites = true
allowed_rooms = []  # 允许所有人邀请
```

**场景 2: 私有团队机器人**
```toml
auto_accept_invites = false
allowed_rooms = ["!team-room:matrix.org"]  # 只响应指定房间
```

---

---

## 19. MqttAdapter — MQTT Pub/Sub 适配器 (v0.5.2 新增)

### 文件位置
`crates/openfang-channels/src/mqtt.rs` (604 行)

### 核心结构

```rust
pub struct MqttAdapter {
    broker_url: String,           // MQTT broker URL
    client_id: String,            // 客户端 ID（空时自动生成）
    subscribe_topic: String,      // 订阅主题（接收消息）
    publish_topic: String,        // 发布主题（发送响应）
    username: Option<String>,
    password: Option<String>,
    use_tls: bool,
    keep_alive: u16,
    clean_session: bool,
    qos: QoS,                     // QoS 级别 (0/1/2)
    shutdown_tx: Arc<watch::Sender<bool>>,
    shutdown_rx: watch::Receiver<bool>,
    publish_tx: PublishSender,    // 出站消息通道
}
```

### 配置示例

```toml
[channels.mqtt]
broker_url = "tcp://broker.hivemq.com:1883"
subscribe_topic = "openfang/inbox"
publish_topic = "openfang/outbox"
# username_env = "MQTT_USERNAME"
# password_env = "MQTT_PASSWORD"
use_tls = false
qos = 1
```

### 设计特点

| 特点 | 说明 |
|------|------|
| **Pub/Sub 模式** | 订阅主题接收消息，发布主题发送响应 |
| **消息分块** | 长消息自动分块（4096 字符上限），在换行处分割 |
| **指数退避重连** | 1s → 2s → 4s → ... → 60s（上限） |
| **JSON 负载支持** | 支持 `{"text": "..."}` 格式的 JSON 负载 |
| **命令解析** | 以 `/` 开头的消息解析为 `ChannelContent::Command` |
| **广播模式** | 所有消息标记为 `is_group = true` |
| **QoS 支持** | 支持 QoS 0 (AtMostOnce)、1 (AtLeastOnce)、2 (ExactlyOnce) |

### URL 解析

| 格式 | 端口 |
|------|------|
| `tcp://host:port` | 使用指定端口 |
| `ssl://host:port` | 使用指定端口 |
| `host` + `use_tls=false` | 1883 |
| `host` + `use_tls=true` | 8883 |

### 未实现的方法

| 方法 | 行为 |
|------|------|
| `send_typing()` | 空操作（MQTT 无 typing 概念） |
| `send_reaction()` | 默认 no-op |
| `status()` | 默认 `connected = false` |

---

- [ ] 理解 ChannelAdapter trait 的设计
- [ ] 掌握 ChannelMessage 统一消息结构
- [ ] 理解 AgentRouter 路由机制
- [ ] 掌握 OutputFormat 消息格式化
- [ ] 了解 40+ 个渠道适配器架构
- [ ] 掌握 Matrix auto_accept_invites 配置 (v0.5.1)
- [ ] 了解 MQTT Pub/Sub 适配器 (v0.5.2)

---

## 下一步

前往 [第 17 节：Channel 系统 — 事件总线](./17-channels-event-bus.md)

---

*创建时间：2026-03-15 (更新于 2026-03-29 v0.5.2)*
*OpenFang v0.5.2*
