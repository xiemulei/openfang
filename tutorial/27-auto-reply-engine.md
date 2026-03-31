# Auto-reply Engine — 自动回复引擎

Version: v0.5.5

## 1. 核心功能

Auto-reply Engine 是一个后台回复系统，用于在各种通道上自动响应消息，具有并发控制和触发条件管理功能。

### 1.1 主要功能

- **自动回复触发**：基于消息内容和配置的规则触发自动回复
- **并发控制**：限制同时执行的自动回复数量，防止系统过载
- **消息抑制**：通过配置的模式抑制不需要自动回复的消息
- **多通道支持**：支持不同类型的通信通道（如 Telegram、Discord 等）
- **超时处理**：为自动回复设置超时时间，避免无限等待
- **后台执行**：在后台执行自动回复，不阻塞主线程

## 2. 架构设计

### 2.1 核心结构

```rust
/// Where to deliver the auto-reply result.
#[derive(Debug, Clone)]
pub struct AutoReplyChannel {
    /// Channel type string (e.g., "telegram", "discord").
    pub channel_type: String,
    /// Peer/user ID to send the reply to.
    pub peer_id: String,
    /// Optional thread ID for threaded replies.
    pub thread_id: Option<String>,
}

/// Auto-reply engine with concurrency limits and suppression patterns.
pub struct AutoReplyEngine {
    config: AutoReplyConfig,
    semaphore: Arc<Semaphore>,
}
```

### 2.2 工作流程

1. **消息接收**：系统从各种通道接收到消息
2. **触发检查**：检查消息是否应该触发自动回复
3. **并发控制**：尝试获取并发许可证
4. **后台执行**：在后台执行自动回复任务
5. **代理处理**：将消息发送给代理处理
6. **回复发送**：将代理的响应发送回通道

## 3. 配置选项

### 3.1 配置结构

```rust
pub struct AutoReplyConfig {
    pub enabled: bool,                // 是否启用自动回复
    pub max_concurrent: u32,          // 最大并发回复数
    pub timeout_secs: u64,            // 回复超时时间（秒）
    pub suppress_patterns: Vec<String>, // 抑制自动回复的模式列表
}
```

### 3.2 默认配置

- **enabled**：false（默认禁用）
- **max_concurrent**：3（最大并发数）
- **timeout_secs**：120（超时时间 2 分钟）
- **suppress_patterns**：包含 "/stop" 和 "/pause"（默认抑制模式）

## 4. 核心方法

### 4.1 `should_reply`

检查消息是否应该触发自动回复：

```rust
pub fn should_reply(
    &self,
    message: &str,
    _channel_type: &str,
    agent_id: AgentId,
) -> Option<AgentId> {
    if !self.config.enabled {
        return None;
    }

    // Check suppression patterns
    let lower = message.to_lowercase();
    for pattern in &self.config.suppress_patterns {
        if lower.contains(&pattern.to_lowercase()) {
            debug!(pattern = %pattern, "Auto-reply suppressed by pattern");
            return None;
        }
    }

    Some(agent_id)
}
```

### 4.2 `execute_reply`

在后台执行自动回复：

```rust
pub async fn execute_reply<F>(
    &self,
    kernel_handle: Arc<dyn openfang_runtime::kernel_handle::KernelHandle>,
    agent_id: AgentId,
    message: String,
    reply_channel: AutoReplyChannel,
    send_fn: F,
) -> Result<tokio::task::JoinHandle<()>, String>
where
    F: Fn(String, AutoReplyChannel) -> futures::future::BoxFuture<'static, ()>
        + Send
        + Sync
        + 'static,
{
    // Try to acquire a semaphore permit
    let permit = match self.semaphore.clone().try_acquire_owned() {
        Ok(p) => p,
        Err(_) => {
            return Err(format!(
                "Auto-reply concurrency limit reached ({} max)",
                self.config.max_concurrent
            ));
        }
    };

    let timeout_secs = self.config.timeout_secs;

    let handle = tokio::spawn(async move {
        let _permit = permit; // Hold permit until task completes

        info!(
            agent = %agent_id,
            channel = %reply_channel.channel_type,
            peer = %reply_channel.peer_id,
            "Starting auto-reply"
        );

        let result = tokio::time::timeout(
            std::time::Duration::from_secs(timeout_secs),
            kernel_handle.send_to_agent(&agent_id.to_string(), &message),
        )
        .await;

        match result {
            Ok(Ok(response)) => {
                send_fn(response, reply_channel).await;
            }
            Ok(Err(e)) => {
                warn!(agent = %agent_id, error = %e, "Auto-reply agent error");
            }
            Err(_) => {
                warn!(agent = %agent_id, timeout = timeout_secs, "Auto-reply timed out");
            }
        }
    });

    Ok(handle)
}
```

### 4.3 辅助方法

- **`is_enabled`**：检查自动回复是否启用
- **`config`**：获取当前配置
- **`available_permits`**：获取可用的并发许可证（用于监控）

## 5. 并发控制

Auto-reply Engine 使用信号量（Semaphore）来控制并发执行的自动回复数量：

- 初始化时创建信号量，许可证数量为 `max_concurrent`
- 执行自动回复前尝试获取许可证
- 如果没有可用许可证，返回错误
- 任务完成后自动释放许可证

## 6. 消息抑制

系统通过配置的抑制模式来过滤不需要自动回复的消息：

- 消息内容转换为小写进行匹配
- 检查消息是否包含任何抑制模式
- 如果匹配到抑制模式，不触发自动回复

## 7. 多通道支持

Auto-reply Engine 通过 `AutoReplyChannel` 结构支持不同类型的通道：

- **channel_type**：通道类型字符串（如 "telegram"、"discord" 等）
- **peer_id**：要发送回复的用户/对等方 ID
- **thread_id**：可选的线程 ID，用于线程化回复

## 8. 集成点

- **通道桥接**：与各种通信通道的集成
- **内核处理**：通过 KernelHandle 与代理通信
- **配置系统**：从配置文件加载自动回复设置
- **监控系统**：提供并发状态和执行情况的监控

## 9. 使用场景

### 9.1 典型使用流程

1. **消息接收**：系统从 Telegram 接收到消息 "Hello, how are you?"
2. **触发检查**：Auto-reply Engine 检查消息，确认应该触发自动回复
3. **并发控制**：获取并发许可证
4. **后台执行**：在后台执行自动回复任务
5. **代理处理**：将消息发送给代理处理
6. **回复发送**：将代理的响应 "I'm doing well, thank you!" 发送回 Telegram

### 9.2 配置示例

```rust
let auto_reply_config = AutoReplyConfig {
    enabled: true,
    max_concurrent: 5,  // 最多同时处理 5 个自动回复
    timeout_secs: 60,   // 60 秒超时
    suppress_patterns: vec![
        "/stop",
        "/pause",
        "do not reply",
        "ignore this"
    ],
};

let auto_reply_engine = AutoReplyEngine::new(auto_reply_config);
```

## 10. 代码优化建议

1. **动态配置更新**：支持运行时更新自动回复配置，无需重启系统
2. **智能触发条件**：基于更复杂的条件（如时间、用户历史、消息内容）触发自动回复
3. **回复模板**：支持预定义回复模板，提高回复速度和一致性
4. **错误处理增强**：更详细的错误处理和日志记录
5. **性能监控**：添加性能监控指标，如回复延迟、成功率等
6. **多代理支持**：根据消息内容或通道类型选择不同的代理进行回复

## 11. 总结

Auto-reply Engine 是 OpenFang 系统中的一个重要组件，为各种通信通道提供自动回复功能。它通过并发控制、消息抑制和超时处理等机制，确保自动回复的高效执行和系统稳定性。

通过合理配置自动回复参数，可以在保持系统响应速度的同时，为用户提供及时、准确的自动回复服务。Auto-reply Engine 的设计考虑了可扩展性和灵活性，能够适应不同场景的需求。