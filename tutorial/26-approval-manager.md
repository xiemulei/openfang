# Approval Manager — 审批管理器

Version: v0.5.5

## 1. 核心功能

Approval Manager 是一个安全机制，用于管理和控制危险操作的执行，通过人类审批来确保系统安全。

### 1.1 主要功能

- **审批请求管理**：处理需要人类批准的操作请求
- **风险等级分类**：自动评估工具调用的风险等级
- **策略管理**：支持热重载的审批策略配置
- **超时处理**：对未及时处理的请求进行超时处理
- **历史记录**：追踪最近的审批决策
- **并发控制**：限制每个代理的待处理请求数量

## 2. 架构设计

### 2.1 核心结构

```rust
pub struct ApprovalManager {
    pending: DashMap<Uuid, PendingRequest>,  // 待处理请求
    recent: std::sync::Mutex<VecDeque<ApprovalRecord>>,  // 最近的审批记录
    policy: std::sync::RwLock<ApprovalPolicy>,  // 审批策略
}

struct PendingRequest {
    request: ApprovalRequest,
    sender: tokio::sync::oneshot::Sender<ApprovalDecision>,
}

pub struct ApprovalRecord {
    pub request: ApprovalRequest,
    pub decision: ApprovalDecision,
    pub decided_at: chrono::DateTime<Utc>,
    pub decided_by: Option<String>,
}
```

### 2.2 审批流程

1. 代理尝试执行需要审批的工具
2. 系统检查是否需要审批（基于当前策略）
3. 如果需要审批，创建审批请求并等待人类决策
4. 人类通过 API/UI 查看并处理审批请求
5. 决策结果通过 oneshot channel 返回给等待的代理
6. 系统记录审批历史

## 3. 风险等级分类

Approval Manager 根据工具类型自动分类风险等级：

| 风险等级 | 工具类型 | 描述 |
|---------|---------|------|
| Critical | shell_exec | 执行 shell 命令，最高风险 |
| High | file_write, file_delete | 文件写入和删除操作 |
| Medium | web_fetch, browser_navigate | 网络请求和浏览器导航 |
| Low | 其他所有工具 | 低风险操作 |

## 4. 审批策略

### 4.1 策略结构

```rust
pub struct ApprovalPolicy {
    pub require_approval: Vec<String>,  // 需要审批的工具列表
    pub timeout_secs: u64,  // 审批超时时间（秒）
    pub auto_approve_autonomous: bool,  // 是否自动批准自主代理
    pub auto_approve: bool,  // 是否自动批准所有请求
}
```

### 4.2 默认策略

- 需要审批的工具：`["shell_exec"]`
- 超时时间：60秒
- 自动批准自主代理：false
- 自动批准所有请求：false

## 5. 核心方法

### 5.1 `requires_approval`

检查工具是否需要审批：

```rust
pub fn requires_approval(&self, tool_name: &str) -> bool {
    let policy = self.policy.read().unwrap_or_else(|e| e.into_inner());
    policy.require_approval.iter().any(|t| t == tool_name)
}
```

### 5.2 `request_approval`

提交审批请求并等待决策：

```rust
pub async fn request_approval(&self, req: ApprovalRequest) -> ApprovalDecision {
    // 检查每个代理的待处理请求限制
    let agent_pending = self
        .pending
        .iter()
        .filter(|r| r.value().request.agent_id == req.agent_id)
        .count();
    if agent_pending >= MAX_PENDING_PER_AGENT {
        warn!(agent_id = %req.agent_id, "Approval request rejected: too many pending");
        return ApprovalDecision::Denied;
    }

    let timeout = std::time::Duration::from_secs(req.timeout_secs);
    let id = req.id;
    let req_for_timeout = req.clone();

    let (tx, rx) = tokio::sync::oneshot::channel();
    self.pending.insert(
        id,
        PendingRequest {
            request: req,
            sender: tx,
        },
    );

    // 等待审批决策或超时
    match tokio::time::timeout(timeout, rx).await {
        Ok(Ok(decision)) => decision,
        _ => {
            // 处理超时情况
            let request = self
                .pending
                .remove(&id)
                .map(|(_, pending)| pending.request)
                .unwrap_or(req_for_timeout);
            self.push_recent(request, ApprovalDecision::TimedOut, None, Utc::now());
            ApprovalDecision::TimedOut
        }
    }
}
```

### 5.3 `resolve`

解决待处理的审批请求：

```rust
pub fn resolve(
    &self,
    request_id: Uuid,
    decision: ApprovalDecision,
    decided_by: Option<String>,
) -> Result<ApprovalResponse, String> {
    match self.pending.remove(&request_id) {
        Some((_, pending)) => {
            let response = ApprovalResponse {
                request_id,
                decision,
                decided_at: Utc::now(),
                decided_by,
            };
            self.push_recent(
                pending.request.clone(),
                decision,
                response.decided_by.clone(),
                response.decided_at,
            );
            // 发送决策给等待的代理
            let _ = pending.sender.send(decision);
            Ok(response)
        }
        None => Err(format!("No pending approval request with id {request_id}")),
    }
}
```

### 5.4 `classify_risk`

分类工具调用的风险等级：

```rust
pub fn classify_risk(tool_name: &str) -> RiskLevel {
    match tool_name {
        "shell_exec" => RiskLevel::Critical,
        "file_write" | "file_delete" => RiskLevel::High,
        "web_fetch" | "browser_navigate" => RiskLevel::Medium,
        _ => RiskLevel::Low,
    }
}
```

## 6. 配置与限制

### 6.1 系统限制

- **每个代理的最大待处理请求数**：5个
- **最大审批历史记录数**：100条
- **默认超时时间**：60秒

### 6.2 配置示例

```rust
let policy = ApprovalPolicy {
    require_approval: vec!["shell_exec", "file_write", "file_delete"],
    timeout_secs: 120,  // 2分钟超时
    auto_approve_autonomous: false,
    auto_approve: false,
};

let approval_manager = ApprovalManager::new(policy);
```

## 7. 使用场景

### 7.1 典型使用流程

1. **代理执行危险操作**：代理尝试执行 `shell_exec` 工具
2. **系统检查**：Approval Manager 检查该工具需要审批
3. **创建审批请求**：系统创建审批请求并暂停代理执行
4. **人类审批**：管理员通过 UI 查看请求并做出决策
5. **执行或拒绝**：根据审批决策，系统执行操作或拒绝请求

### 7.2 安全考虑

- **最小权限原则**：默认只对高风险工具要求审批
- **超时保护**：防止审批请求无限期等待
- **历史审计**：记录所有审批决策，便于审计
- **并发控制**：防止单个代理发送过多审批请求

## 8. 集成点

- **工具执行流程**：在工具执行前检查是否需要审批
- **API/UI**：提供审批请求的查看和处理接口
- **策略管理**：支持运行时更新审批策略
- **日志系统**：记录审批相关的事件和决策

## 9. 代码优化建议

1. **策略配置外部化**：将审批策略配置移至外部配置文件，便于管理
2. **审批通知**：添加审批请求通知机制，如邮件、消息推送等
3. **审批 delegation**：支持审批权限委托，当主要审批人不可用时
4. **风险评分系统**：基于操作内容和上下文进行更细粒度的风险评估
5. **审批模板**：为常见操作提供审批模板，加速审批流程

## 10. 总结

Approval Manager 是 OpenFang 系统中的重要安全组件，通过人类审批机制为危险操作提供额外的安全保障。它不仅支持灵活的策略配置，还提供了完整的审批流程管理和历史记录功能，确保系统操作的安全性和可追溯性。

通过合理配置审批策略，可以在保障安全的同时，平衡系统的可用性和响应速度，为 OpenFang 系统提供更全面的安全防护。