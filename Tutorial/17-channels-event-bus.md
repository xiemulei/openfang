# 第 17 节：Channel 系统 — 事件总线

> **版本**: v0.4.4 (2026-03-15)
> **核心文件**:
> - `crates/openfang-kernel/src/event_bus.rs`
> - `crates/openfang-types/src/event.rs`

## 学习目标

- [ ] 理解 EventBus 发布/订阅架构
- [ ] 掌握 Event/EventId/EventTarget/EventPayload 类型系统
- [ ] 理解 Agent 专属通道机制
- [ ] 掌握历史事件环缓冲区设计
- [ ] 理解模式匹配订阅逻辑
- [ ] 掌握 Webhook 集成扩展点

---

## 1. EventBus — 发布/订阅核心

### 文件位置
`crates/openfang-kernel/src/event_bus.rs:15-22`

```rust
/// Central event bus for agent lifecycle and kernel events.
///
/// Uses a broadcast channel pattern with per-agent subscriptions.
pub struct EventBus {
    /// Global broadcast sender — all events go here
    sender: broadcast::Sender<Event>,
    /// Per-agent channels for targeted delivery
    agent_channels: DashMap<AgentId, broadcast::Sender<Event>>,
    /// History ring buffer — keeps last N events
    history: Arc<RwLock<VecDeque<Event>>>,
}

/// Number of events to keep in history
const HISTORY_SIZE: usize = 1000;
```

### 设计要点

| 字段 | 类型 | 用途 |
|------|------|------|
| `sender` | `broadcast::Sender<Event>` | 全局广播通道，所有事件都发送到这里 |
| `agent_channels` | `DashMap<AgentId, broadcast::Sender<Event>>` | 每 Agent 专属通道，用于定向投递 |
| `history` | `Arc<RwLock<VecDeque<Event>>>` | 历史环缓冲区，保留最近 1000 个事件 |

### 架构优势

```
┌─────────────────────────────────────────────────────────────┐
│                      EventBus                                │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Global Broadcast Channel                             │   │
│  │ ─────────────────────────────────────────────────────│   │
│  │ All publishers → sender → All subscribers            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Per-Agent Channels (DashMap)                         │   │
│  │ ─────────────────────────────────────────────────────│   │
│  │ Agent-A → sender_A → subscribers_A                   │   │
│  │ Agent-B → sender_B → subscribers_B                   │   │
│  │ ...                                                   │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ History Ring Buffer (1000 events)                    │   │
│  │ ─────────────────────────────────────────────────────│   │
│  │ [ev_998] ← [ev_999] ← [ev_1000] ← (new overwrites)   │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. EventId — 事件唯一标识

### 文件位置
`crates/openfang-types/src/event.rs:8-14`

```rust
/// Unique identifier for an event.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct EventId(pub Uuid);

impl EventId {
    /// Generate a new random event ID.
    pub fn new() -> Self {
        Self(Uuid::new_v4())
    }
}

impl Default for EventId {
    fn default() -> Self {
        Self::new()
    }
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `0` (内层) | `Uuid` | UUID v4 随机标识符 |

### 使用场景

```rust
// 创建新事件时生成唯一 ID
let event = Event {
    id: EventId::new(),
    // ...
};

// 关联请求 - 响应（correlation_id）
let response = Event {
    correlation_id: Some(request_event.id),  // 关联到原始请求
    // ...
};
```

---

## 3. EventTarget — 事件目标

### 文件位置
`crates/openfang-types/src/event.rs:55-67`

```rust
/// Where an event should be delivered.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum EventTarget {
    /// Deliver to a specific agent.
    Agent(AgentId),
    /// Broadcast to all agents.
    Broadcast,
    /// Pattern-based routing (e.g., tag matching).
    Pattern(String),
    /// System-level event (for kernel/internal use).
    System,
}
```

### 目标类型详解

| 变体 | 用途 | 投递行为 |
|------|------|----------|
| `Agent(id)` | 定向投递给特定 Agent | 发送到该 Agent 的专属通道 |
| `Broadcast` | 广播给所有 Agent | 发送到全局通道 + 所有 Agent 通道 |
| `Pattern(pattern)` | 基于模式匹配路由 | 广播供订阅者进行模式匹配 |
| `System` | 系统级事件 | 发送到全局通道供内核订阅 |

### 使用示例

```rust
// 1. 定向发送消息给 Agent-A
Event {
    target: EventTarget::Agent(agent_a_id),
    payload: EventPayload::Message(...),
    // ...
}

// 2. 广播系统通知
Event {
    target: EventTarget::Broadcast,
    payload: EventPayload::System(SystemEvent::Shutdown),
    // ...
}

// 3. 带标签的事件（订阅者可过滤）
Event {
    target: EventTarget::Pattern("memory.*".to_string()),
    payload: EventPayload::MemoryUpdate(...),
    // ...
}

// 4. 内核内部事件
Event {
    target: EventTarget::System,
    payload: EventPayload::System(SystemEvent::ConfigReload),
    // ...
}
```

---

## 4. EventPayload — 事件负载

### 文件位置
`crates/openfang-types/src/event.rs:69-87`

```rust
/// The actual data carried by an event.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum EventPayload {
    /// Agent-to-agent message.
    Message(AgentMessage),
    /// Tool execution result.
    ToolResult(ToolOutput),
    /// Memory substrate change notification.
    MemoryUpdate(MemoryDelta),
    /// Agent lifecycle phase change.
    Lifecycle(LifecycleEvent),
    /// Network/peer-related event.
    Network(NetworkEvent),
    /// System-level event.
    System(SystemEvent),
    /// Custom binary payload for extensions.
    Custom(Vec<u8>),
}
```

### 负载类型详解

| 变体 | 携带数据类型 | 典型场景 |
|------|-------------|----------|
| `Message` | `AgentMessage` | Agent 间通信 |
| `ToolResult` | `ToolOutput` | 工具执行完成通知 |
| `MemoryUpdate` | `MemoryDelta` | 记忆系统变更通知 |
| `Lifecycle` | `LifecycleEvent` | Agent 启动/停止/阶段变更 |
| `Network` | `NetworkEvent` |  peers 连接/断开 |
| `System` | `SystemEvent` | 配置变更/关机等 |
| `Custom` | `Vec<u8>` | 扩展自定义负载 |

---

## 5. LifecycleEvent — 生命周期事件

### 文件位置
`crates/openfang-types/src/event.rs:89-112`

```rust
/// Agent lifecycle phase changes.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LifecycleEvent {
    /// The agent that changed state.
    pub agent_id: AgentId,
    /// The new phase.
    pub phase: LifecyclePhase,
    /// Optional detail message.
    pub detail: Option<String>,
    /// Optional tool name (if phase is tool-related).
    pub tool_name: Option<String>,
}

/// Agent lifecycle phases.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum LifecyclePhase {
    /// Agent is starting up.
    Starting,
    /// Agent is idle, waiting for messages.
    Idle,
    /// Agent is processing a message.
    Processing,
    /// Agent is executing a tool.
    ExecutingTool { tool_name: String },
    /// Agent is waiting for user input.
    WaitingForInput,
    /// Agent is shutting down.
    Stopping,
}
```

### 状态流转图

```
Starting → Idle → Processing → ExecutingTool → Processing → Idle
                     ↓
              WaitingForInput → Processing → Idle
                     ↓
                  Stopping
```

---

## 6. Event — 完整事件结构

### 文件位置
`crates/openfang-types/src/event.rs:282-300`

```rust
/// A kernel event with routing and metadata.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Event {
    /// Unique event identifier.
    pub id: EventId,
    /// The agent that originated this event.
    pub source: AgentId,
    /// Where this event should be delivered.
    pub target: EventTarget,
    /// The event data.
    pub payload: EventPayload,
    /// When the event was created.
    pub timestamp: DateTime<Utc>,
    /// Optional correlation ID for request-response pairing.
    pub correlation_id: Option<EventId>,
    /// Optional time-to-live — event expires after this duration.
    pub ttl: Option<Duration>,
}
```

### 字段说明

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `id` | `EventId` | 事件唯一标识 | `EventId(uuid)` |
| `source` | `AgentId` | 事件来源 Agent | `AgentId("agent-abc123")` |
| `target` | `EventTarget` | 投递目标 | `EventTarget::Agent(id)` |
| `payload` | `EventPayload` | 事件数据 | `EventPayload::Message(...)` |
| `timestamp` | `DateTime<Utc>` | 创建时间 | `2026-03-15T10:30:00Z` |
| `correlation_id` | `Option<EventId>` | 关联的请求 ID | `Some(request_id)` |
| `ttl` | `Option<Duration>` | 生存时间 | `Some(Duration::from_secs(60))` |

### 完整事件示例

```rust
// Agent-A 发送消息给 Agent-B
let event = Event {
    id: EventId::new(),
    source: agent_a_id,
    target: EventTarget::Agent(agent_b_id),
    payload: EventPayload::Message(AgentMessage {
        content: "Hello from Agent-A!".to_string(),
        // ...
    }),
    timestamp: Utc::now(),
    correlation_id: None,
    ttl: Some(Duration::from_secs(300)),  // 5 分钟后过期
};
```

---

## 7. publish — 事件发布

### 文件位置
`crates/openfang-kernel/src/event_bus.rs:35-73`

```rust
impl EventBus {
    /// Publish an event to the bus.
    pub async fn publish(&self, event: Event) {
        // Store in history ring buffer
        {
            let mut history = self.history.write().await;
            history.push_back(event.clone());
            // Maintain ring buffer size
            while history.len() > HISTORY_SIZE {
                history.pop_front();
            }
        }

        // Route based on target
        match &event.target {
            EventTarget::Agent(agent_id) => {
                // Send to agent's dedicated channel
                if let Some(tx) = self.agent_channels.get(agent_id) {
                    let _ = tx.send(event);
                }
            }
            EventTarget::Broadcast => {
                // Send to global channel (all subscribers receive)
                let _ = self.sender.send(event.clone());
                // Also send to all agent channels
                for entry in self.agent_channels.iter() {
                    let _ = entry.value().send(event.clone());
                }
            }
            EventTarget::Pattern(_) => {
                // Broadcast for pattern matching by subscribers
                let _ = self.sender.send(event);
            }
            EventTarget::System => {
                // Send to global channel for kernel subscribers
                let _ = self.sender.send(event);
            }
        }
    }
}
```

### 路由逻辑详解

```
publish(event)
    │
    ├─→ [1] 存入历史环缓冲区
    │       └─→ 如果超过 1000 条，弹出最旧的事件
    │
    ├─→ [2] 根据 target 路由
    │       │
    │       ├─→ Agent(id): 发送到该 Agent 的专属通道
    │       │
    │       ├─→ Broadcast: 发送到全局通道 + 所有 Agent 通道
    │       │
    │       ├─→ Pattern: 发送到全局通道（订阅者自行匹配）
    │       │
    │       └─→ System: 发送到全局通道（内核订阅）
    │
    └─→ [3] 完成（忽略发送错误）
```

---

## 8. subscribe_agent — Agent 专属订阅

### 文件位置
`crates/openfang-kernel/src/event_bus.rs:75-94`

```rust
impl EventBus {
    /// Subscribe to events for a specific agent.
    ///
    /// Returns a receiver that will get events targeted to this agent
    /// via EventTarget::Agent(agent_id) or EventTarget::Broadcast.
    pub fn subscribe_agent(&self, agent_id: AgentId) -> broadcast::Receiver<Event> {
        // Create dedicated channel for this agent if not exists
        self.agent_channels.entry(agent_id).or_insert_with(|| {
            // Use same buffer size as global sender
            let (tx, rx) = broadcast::channel(self.sender.capacity());
            tx
        });

        // Get or create the sender, then subscribe
        let tx = self.agent_channels
            .entry(agent_id)
            .or_insert_with(|| {
                let (tx, rx) = broadcast::channel(self.sender.capacity());
                tx
            })
            .clone();

        tx.subscribe()
    }
}
```

### 工作流程

```
subscribe_agent(agent_id)
    │
    ├─→ [1] 检查 agent_channels 是否存在该 Agent 的通道
    │       │
    │       ├─→ 不存在：创建新的 broadcast channel
    │       │
    │       └─→ 已存在：复用现有通道
    │
    └─→ [2] 返回 receiver
            │
            └─→ 接收目标为该 Agent 或 Broadcast 的事件
```

### 使用场景

```rust
// Agent Loop 启动时订阅自己的事件通道
let mut event_rx = event_bus.subscribe_agent(my_agent_id);

// 在 Agent Loop 中处理事件
while let Ok(event) = event_rx.recv().await {
    match event.payload {
        EventPayload::Message(msg) => {
            // 处理来自其他 Agent 的消息
        }
        EventPayload::Lifecycle(lifecycle) => {
            // 处理生命周期事件（如其他 Agent 的状态变更）
        }
        _ => {}
    }
}
```

---

## 9. subscribe_all — 全局订阅

### 文件位置
`crates/openfang-kernel/src/event_bus.rs:96-104`

```rust
impl EventBus {
    /// Subscribe to all events (broadcast pattern).
    ///
    /// Returns a receiver that will get ALL events published to the bus.
    pub fn subscribe_all(&self) -> broadcast::Receiver<Event> {
        self.sender.subscribe()
    }
}
```

### 使用场景

| 订阅者 | 用途 |
|--------|------|
| **Dashboard** | 实时更新所有 Agent 状态 |
| **审计日志** | 记录所有事件到持久化存储 |
| **监控告警** | 检测异常事件模式 |
| **调试工具** | 追踪系统行为 |

### 示例：Dashboard 全局订阅

```rust
// Dashboard 后端订阅所有事件
let mut global_rx = event_bus.subscribe_all();

tokio::spawn(async move {
    while let Ok(event) = global_rx.recv().await {
        // 推送到前端（WebSocket/SSE）
        dashboard_tx.send(event).await.ok();
    }
});
```

---

## 10. history — 历史事件查询

### 文件位置
`crates/openfang-kernel/src/event_bus.rs:106-118`

```rust
impl EventBus {
    /// Get recent events from history.
    pub async fn history(&self, limit: usize) -> Vec<Event> {
        let history = self.history.read().await;
        history.iter().rev().take(limit).cloned().collect()
    }
}
```

### 设计要点

| 特性 | 说明 |
|------|------|
| **环缓冲区** | 固定大小 1000，新事件覆盖旧事件 |
| **倒序返回** | 最新事件在前（`.rev()`） |
| **只读锁** | 不阻塞发布操作 |
| **限制数量** | `take(limit)` 防止返回过多数据 |

### 使用场景

```rust
// Dashboard 加载最近 50 个事件
let recent_events = event_bus.history(50).await;

// 显示在事件时间轴上
for event in recent_events {
    render_event_timeline_item(event);
}
```

---

## 11. 模式匹配订阅

### 文件位置
`crates/openfang-kernel/src/event_bus.rs:120-145`（推断）

```rust
impl EventBus {
    /// Subscribe to events matching a pattern.
    ///
    /// Pattern syntax:
    /// - "*" matches everything
    /// - "memory.*" matches memory-related events
    /// - "lifecycle.agent_*" matches agent lifecycle events
    pub fn subscribe_pattern(
        &self,
        pattern: &str,
    ) -> Result<broadcast::Receiver<Event>, EventBusError> {
        // Subscribe to global channel
        let mut rx = self.sender.subscribe();

        // Create a filtered stream that only yields matching events
        // (Implementation depends on pattern matching library)
        let pattern_owned = pattern.to_string();
        let (tx, filtered_rx) = broadcast::channel(100);

        tokio::spawn(async move {
            while let Ok(event) = rx.recv().await {
                if matches_pattern(&pattern_owned, &event) {
                    let _ = tx.send(event);
                }
            }
        });

        Ok(filtered_rx)
    }
}

/// Check if an event matches a pattern.
fn matches_pattern(pattern: &str, event: &Event) -> bool {
    // Simple glob-style matching
    match pattern {
        "*" => true,
        "memory.*" => matches!(event.payload, EventPayload::MemoryUpdate(_)),
        "lifecycle.*" => matches!(event.payload, EventPayload::Lifecycle(_)),
        "network.*" => matches!(event.payload, EventPayload::Network(_)),
        _ => pattern == "*",  // Fallback
    }
}
```

### 模式语法

| 模式 | 匹配事件 |
|------|----------|
| `*` | 所有事件 |
| `memory.*` | 所有记忆系统事件 |
| `lifecycle.*` | 所有生命周期事件 |
| `network.*` | 所有网络事件 |
| `tool.*` | 所有工具执行事件 |

---

## 12. Webhook 集成扩展

### 文件位置
`crates/openfang-kernel/src/webhook.rs`（推断）

```rust
/// Webhook integration for external systems.
pub struct WebhookManager {
    event_bus: Arc<EventBus>,
    subscriptions: DashMap<String, WebhookSubscription>,
}

/// A webhook subscription.
pub struct WebhookSubscription {
    /// Target URL to POST events to.
    pub url: String,
    /// Which events to forward.
    pub pattern: String,
    /// Optional HMAC secret for signing payloads.
    pub secret: Option<String>,
    /// HTTP headers to include.
    pub headers: HashMap<String, String>,
}

impl WebhookManager {
    /// Register a new webhook.
    pub fn register(&self, id: String, sub: WebhookSubscription) {
        self.subscriptions.insert(id, sub);
    }

    /// Unregister a webhook.
    pub fn unregister(&self, id: &str) {
        self.subscriptions.remove(id);
    }

    /// Start forwarding events to registered webhooks.
    pub fn start_forwarding(&self) {
        let mut global_rx = self.event_bus.subscribe_all();

        let subscriptions = self.subscriptions.clone();

        tokio::spawn(async move {
            while let Ok(event) = global_rx.recv().await {
                // Forward to matching webhooks
                for entry in subscriptions.iter() {
                    let sub = entry.value();
                    if matches_pattern(&sub.pattern, &event) {
                        let client = reqwest::Client::new();
                        let payload = serde_json::to_vec(&event).ok().unwrap();

                        let mut req = client.post(&sub.url)
                            .body(payload)
                            .header("Content-Type", "application/json");

                        // Add custom headers
                        for (key, value) in &sub.headers {
                            req = req.header(key, value);
                        }

                        // Add HMAC signature if configured
                        if let Some(secret) = &sub.secret {
                            let signature = hmac_sha256(secret, &req.body);
                            req = req.header("X-Webhook-Signature", signature);
                        }

                        tokio::spawn(async move {
                            let _ = req.send().await;
                        });
                    }
                }
            }
        });
    }
}
```

### Webhook 配置示例

```toml
# ~/.openfang/webhooks.toml

[[webhooks]]
id = "slack-notify"
url = "https://hooks.slack.com/services/xxx/yyy/zzz"
pattern = "lifecycle.*"
headers = { Authorization = "Bearer xxx" }

[[webhooks]]
id = "audit-logger"
url = "http://localhost:9000/api/audit"
pattern = "*"
secret = "hmac-secret-key"
```

### 使用场景

| 场景 | Webhook 配置 |
|------|-------------|
| **Slack 通知** | `pattern = "lifecycle.Error"` |
| **审计日志** | `pattern = "*"`（所有事件） |
| **监控告警** | `pattern = "tool.*"`（工具执行） |
| **数据分析** | `pattern = "memory.*"`（记忆变更） |

---

## 13. 与 Channel Bridge 的集成

### 文件位置
`crates/openfang-channels/src/bridge.rs`（参考）

```rust
impl BridgeManager {
    /// Dispatch a channel message to the EventBus.
    pub async fn dispatch_message(&self, channel_msg: ChannelMessage) {
        // Resolve target agent via router
        let target_agent = self.router.resolve(&channel_msg);

        // Create event
        let event = Event {
            id: EventId::new(),
            source: AgentId::system(),  // External message
            target: EventTarget::Agent(target_agent),
            payload: EventPayload::Message(AgentMessage::from(channel_msg)),
            timestamp: Utc::now(),
            correlation_id: None,
            ttl: None,
        };

        // Publish to event bus
        self.event_bus.publish(event).await;
    }
}
```

### 消息流转

```
Telegram Message
       ↓
TelegramAdapter
       ↓
BridgeManager.dispatch_message()
       ↓
EventBus.publish(event)
       ↓
Agent专属通道 / Broadcast通道
       ↓
Agent Loop 接收并处理
```

---

## 14. 测试用例

### 文件位置
`crates/openfang-kernel/src/event_bus.rs:200-350`（推断）

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_publish_and_subscribe_agent() {
        let event_bus = EventBus::new();
        let agent_id = AgentId::new();

        // Subscribe
        let mut rx = event_bus.subscribe_agent(agent_id);

        // Publish
        let event = Event {
            id: EventId::new(),
            source: agent_id,
            target: EventTarget::Agent(agent_id),
            payload: EventPayload::Lifecycle(LifecycleEvent {
                agent_id,
                phase: LifecyclePhase::Starting,
                detail: None,
                tool_name: None,
            }),
            timestamp: Utc::now(),
            correlation_id: None,
            ttl: None,
        };

        event_bus.publish(event.clone()).await;

        // Receive
        let received = rx.recv().await.unwrap();
        assert_eq!(received.id, event.id);
        assert!(matches!(
            received.payload,
            EventPayload::Lifecycle(_)
        ));
    }

    #[tokio::test]
    async fn test_broadcast_reaches_all_subscribers() {
        let event_bus = EventBus::new();
        let agent_a = AgentId::new();
        let agent_b = AgentId::new();

        // Subscribe both agents
        let mut rx_a = event_bus.subscribe_agent(agent_a);
        let mut rx_b = event_bus.subscribe_agent(agent_b);

        // Publish broadcast
        let event = Event {
            id: EventId::new(),
            source: agent_a,
            target: EventTarget::Broadcast,
            payload: EventPayload::System(SystemEvent::ConfigReload),
            timestamp: Utc::now(),
            correlation_id: None,
            ttl: None,
        };

        event_bus.publish(event.clone()).await;

        // Both should receive
        let recv_a = rx_a.recv().await.unwrap();
        let recv_b = rx_b.recv().await.unwrap();

        assert_eq!(recv_a.id, event.id);
        assert_eq!(recv_b.id, event.id);
    }

    #[tokio::test]
    async fn test_history_ring_buffer() {
        let event_bus = EventBus::new();
        let agent_id = AgentId::new();

        // Publish 1100 events
        for i in 0..1100 {
            let event = Event {
                id: EventId::new(),
                source: agent_id,
                target: EventTarget::Agent(agent_id),
                payload: EventPayload::System(SystemEvent::Heartbeat(i)),
                timestamp: Utc::now(),
                correlation_id: None,
                ttl: None,
            };
            event_bus.publish(event).await;
        }

        // Should only have 1000 in history
        let history = event_bus.history(2000).await;
        assert_eq!(history.len(), 1000);

        // Most recent should be event #1099
        assert!(matches!(
            history[0].payload,
            EventPayload::System(SystemEvent::Heartbeat(1099))
        ));

        // Oldest should be event #100
        assert!(matches!(
            history[999].payload,
            EventPayload::System(SystemEvent::Heartbeat(100))
        ));
    }

    #[tokio::test]
    async fn test_correlation_id_pairs_request_response() {
        let event_bus = EventBus::new();
        let agent_a = AgentId::new();
        let agent_b = AgentId::new();

        let mut rx_b = event_bus.subscribe_agent(agent_b);

        // Request
        let request_id = EventId::new();
        let request = Event {
            id: request_id,
            source: agent_a,
            target: EventTarget::Agent(agent_b),
            payload: EventPayload::Message(AgentMessage::request("ping")),
            timestamp: Utc::now(),
            correlation_id: None,
            ttl: None,
        };
        event_bus.publish(request).await;

        // Response (correlated)
        let response = Event {
            id: EventId::new(),
            source: agent_b,
            target: EventTarget::Agent(agent_a),
            payload: EventPayload::Message(AgentMessage::response("pong")),
            timestamp: Utc::now(),
            correlation_id: Some(request_id),
            ttl: None,
        };
        event_bus.publish(response.clone()).await;

        // Verify correlation
        let recv = rx_b.recv().await.unwrap();
        assert_eq!(recv.correlation_id, Some(request_id));
    }
}
```

---

## 15. 关键设计点

### 15.1 发布/订阅模式优势

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  Publisher  │ ──→ │  EventBus    │ ←── │ Subscriber  │
│  (解耦)     │     │  (中介)      │     │  (解耦)     │
└─────────────┘     └──────────────┘     └─────────────┘
```

**优点**：
- **发布者和订阅者互不感知**：松耦合
- **支持多订阅者**：一个事件可被多方消费
- **支持过滤**：订阅者可以选择性接收

### 15.2 双重通道设计

```
Global Channel ──→ 所有事件（供 Dashboard/审计/监控）
       │
       └─→ 与 Agent Channels 的关系
               │
               ├─→ Agent-A Channel: 只接收定向事件
               ├─→ Agent-B Channel: 只接收定向事件
               └─→ Broadcast 事件同时发送到所有通道
```

### 15.3 环缓冲区空间优化

```rust
// 固定大小 1000
const HISTORY_SIZE: usize = 1000;

// 新事件覆盖旧事件
while history.len() > HISTORY_SIZE {
    history.pop_front();  // 移除最旧
}
history.push_back(new_event);  // 添加最新
```

**空间复杂度**: O(1) — 固定内存占用

### 15.4 错误处理策略

```rust
// 忽略发送错误（使用 `_ =`）
let _ = tx.send(event);
```

**原因**：
- 接收者可能已经断开连接
- 事件丢失不影响核心功能
- 避免阻塞发布者

---

## 16. 与第 16 节的关联

### Channel Bridge → EventBus

```
Channel Message (Telegram/Discord/etc.)
       ↓
BridgeManager.dispatch_message()
       ↓
EventBus.publish(event)  ← 本章内容
       ↓
Agent Loop 接收并处理
       ↓
Response → BridgeManager.send_to_channel()  ← 第 16 节内容
```

### 完整消息流

```
用户 (Telegram)
       ↓
TelegramAdapter (第 16 节)
       ↓
ChannelMessage (第 16 节)
       ↓
BridgeManager (第 16 节)
       ↓
EventBus.publish() (本章)
       ↓
Agent Loop 接收
       ↓
LLM 处理
       ↓
响应 → BridgeManager.send_to_channel() (第 16 节)
       ↓
TelegramAdapter.send() (第 16 节)
       ↓
用户 (Telegram)
```

---

## 完成检查清单

- [ ] 理解 EventBus 发布/订阅架构
- [ ] 掌握 Event/EventId/EventTarget/EventPayload 类型系统
- [ ] 理解 Agent 专属通道机制
- [ ] 掌握历史事件环缓冲区设计
- [ ] 理解模式匹配订阅逻辑
- [ ] 掌握 Webhook 集成扩展点

---

## 下一步

前往 [第 18 节：OFP 协议 — P2P 通信](./18-ofp-protocol.md)

---

*创建时间：2026-03-15*
*OpenFang v0.4.4*
