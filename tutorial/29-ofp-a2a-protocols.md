# OFP 和 A2A 协议 — 通信系统

Version: v0.5.5

## 1. 协议概览

OpenFang 系统使用两个核心协议来实现通信：

### 1.1 OFP 协议

**OpenFang Wire Protocol (OFP)** 是 OpenFang 内核之间的点对点通信协议，提供：

- **跨机器发现**：自动发现和注册远程 Agent
- **双向认证**：HMAC-SHA256 确保双方身份可信
- **消息加密**：每会话密钥派生，防止窃听
- **防重放攻击**：Nonce 追踪和时间窗口验证
- **JSON-RPC 风格**：易调试、易扩展的消息格式

### 1.2 A2A 协议

**Agent-to-Agent (A2A) Protocol** 是一个跨框架的 Agent 互操作协议，提供：

- **跨框架互操作**：不同 Agent 框架之间可以通信和协作
- **能力发现**：通过 Agent Card 公开 Agent 的能力
- **任务协调**：基于任务的协作模式（提交 - 执行 - 完成）
- **流式支持**：支持流式响应和实时状态更新

### 1.3 协议层次关系

```
┌─────────────────────────────────────────────────────────┐
│                    OpenFang Kernel                       │
│  ┌───────────────────────────────────────────────────┐  │
│  │ A2A Protocol (应用层)                             │  │
│  │ - Agent Card 发现                                 │  │
│  │ - Task-based 协作                                 │  │
│  │ - JSON-RPC 2.0                                    │  │
│  └───────────────────────────────────────────────────┘  │
│                          ↓ uses                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │ OFP Protocol (传输层)                             │  │
│  │ - HMAC-SHA256 认证                                │  │
│  │ - 每会话密钥派生                                  │  │
│  │ - TCP 传输                                        │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## 2. OFP 协议详细分析

### 2.1 核心组件

#### 2.1.1 WireMessage — 协议消息信封

```rust
/// A wire protocol message (envelope).
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct WireMessage {
    /// Unique message ID.
    pub id: String,
    /// Message variant.
    #[serde(flatten)]
    pub kind: WireMessageKind,
}
```

#### 2.1.2 WireMessageKind — 消息类型

```rust
pub enum WireMessageKind {
    Request(WireRequest),      // 请求
    Response(WireResponse),    // 响应
    Notification(WireNotification), // 通知（单向）
}
```

#### 2.1.3 WireRequest — 请求消息

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "method")]
pub enum WireRequest {
    /// Handshake: exchange peer identity.
    #[serde(rename = "handshake")]
    Handshake {
        node_id: String,
        node_name: String,
        protocol_version: u32,
        agents: Vec<RemoteAgentInfo>,
        nonce: String,
        auth_hmac: String,
    },
    /// Discover agents matching a query.
    #[serde(rename = "discover")]
    Discover { query: String },
    /// Send a message to a specific agent.
    #[serde(rename = "agent_message")]
    AgentMessage {
        agent: String,
        message: String,
        sender: Option<String>,
    },
    /// Ping to check if the peer is alive.
    #[serde(rename = "ping")]
    Ping,
}
```

#### 2.1.4 PeerRegistry — 对等体注册表

```rust
/// Thread-safe registry of all known peers.
#[derive(Debug, Clone)]
pub struct PeerRegistry {
    peers: Arc<RwLock<HashMap<String, PeerEntry>>>,
}
```

### 2.2 安全机制

#### 2.2.1 HMAC-SHA256 认证

```rust
/// Generate HMAC-SHA256 signature for message authentication.
fn hmac_sign(secret: &str, data: &[u8]) -> String {
    let mut mac = HmacSha256::new_from_slice(secret.as_bytes())
        .expect("HMAC accepts任何大小的密钥");
    mac.update(data);
    hex::encode(mac.finalize().into_bytes())
}

/// Verify HMAC-SHA256 signature using constant-time comparison.
fn hmac_verify(secret: &str, data: &[u8], signature: &str) -> bool {
    let expected = hmac_sign(secret, data);
    subtle::ConstantTimeEq::ct_eq(expected.as_bytes(), signature.as_bytes()).into()
}
```

#### 2.2.2 NonceTracker — 防重放攻击

```rust
/// SECURITY: Time-windowed nonce tracker to prevent OFP handshake replay attacks.
#[derive(Clone)]
pub struct NonceTracker {
    seen: Arc<DashMap<String, Instant>>,
    window: Duration,
}

impl NonceTracker {
    pub fn new() -> Self {
        Self {
            seen: Arc::new(DashMap::new()),
            window: Duration::from_secs(300), // 5 分钟
        }
    }

    pub fn check_and_record(&self, nonce: &str) -> Result<(), String> {
        let now = Instant::now();

        // 垃圾回收过期 nonce
        self.seen.retain(|_, ts| now.duration_since(*ts) < self.window);

        // 检查是否重放
        if self.seen.contains_key(nonce) {
            return Err(format!("Nonce replay detected: {}", truncate_str(nonce, 16)));
        }

        // 记录 nonce
        self.seen.insert(nonce.to_string(), now);
        Ok(())
    }
}
```

#### 2.2.3 会话密钥派生

```rust
/// SECURITY: Derive per-session key from nonces and shared secret.
fn derive_session_key(shared_secret: &str, nonce_a: &str, nonce_b: &str) -> String {
    // HKDF-like derivation: HMAC(shared_secret, nonce_a || nonce_b)
    let input = format!("{}||{}", nonce_a, nonce_b);
    hmac_sign(shared_secret, input.as_bytes())
}
```

### 2.3 握手流程

```
Client (发起连接)                        Server (监听连接)
    │                                          │
    │  1. TCP connect()                        │
    │────────────────────────────────────────→│
    │                                          │
    │  2. Handshake Request                    │
    │     { node_id, node_name, version,       │
    │       agents, nonce, auth_hmac }         │
    │────────────────────────────────────────→│
    │                                          │
    │                                          │ 验证 nonce (防重放)
    │                                          │ 验证 HMAC (认证身份)
    │                                          │
    │  3. HandshakeAck Response                │
    │     { node_id, node_name, version,       │
    │       agents, nonce, auth_hmac }         │
    │←─────────────────────────────────────────│
    │                                          │
    │ 验证 nonce (防重放)                       │
    │ 验证 HMAC (认证身份)                       │
    │ 派生会话密钥                               │
    │                                          │ 派生会话密钥
    │                                          │
    │  4. 开始加密通信                          │
    │←────────────────────────────────────────→│
    │     (所有消息使用 per-message HMAC)       │
    │                                          │
```

## 3. A2A 协议详细分析

### 3.1 核心组件

#### 3.1.1 AgentCard — Agent 能力卡片

```rust
/// A2A Agent Card — describes an agent's capabilities to external systems.
///
/// Served at `/.well-known/agent.json` per the A2A specification.
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct AgentCard {
    /// Agent display name.
    pub name: String,
    /// Human-readable description.
    pub description: String,
    /// Agent endpoint URL.
    pub url: String,
    /// Protocol version.
    pub version: String,
    /// Agent capabilities.
    pub capabilities: AgentCapabilities,
    /// Skills this agent can perform.
    pub skills: Vec<AgentSkill>,
    /// Supported input content types.
    #[serde(default)]
    pub default_input_modes: Vec<String>,
    /// Supported output content types.
    #[serde(default)]
    pub default_output_modes: Vec<String>,
}
```

#### 3.1.2 A2aTask — 任务单元

```rust
/// A2A Task — unit of work exchanged between agents.
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct A2aTask {
    /// Unique task identifier.
    pub id: String,
    /// Optional session identifier for conversation continuity.
    #[serde(default)]
    pub session_id: Option<String>,
    /// Current task status.
    pub status: A2aTaskStatusWrapper,
    /// Messages exchanged during the task.
    #[serde(default)]
    pub messages: Vec<A2aMessage>,
    /// Artifacts produced by the task.
    #[serde(default)]
    pub artifacts: Vec<A2aArtifact>,
}
```

#### 3.1.3 A2aTaskStatus — 任务状态

```rust
/// A2A task status.
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub enum A2aTaskStatus {
    /// Task has been received but not started.
    Submitted,
    /// Task is being processed.
    Working,
    /// Agent needs more input from the caller.
    InputRequired,
    /// Task completed successfully.
    Completed,
    /// Task was cancelled.
    Cancelled,
    /// Task failed.
    Failed,
}
```

#### 3.1.4 A2aClient — 外部 Agent 发现

```rust
/// Client for discovering and interacting with external A2A agents.
pub struct A2aClient {
    client: reqwest::Client,
}

impl A2aClient {
    /// Create a new A2A client.
    pub fn new() -> Self {
        Self {
            client: reqwest::Client::builder()
                .timeout(std::time::Duration::from_secs(30))
                .build()
                .unwrap_or_default(),
        }
    }

    /// Discover an external agent by fetching its Agent Card.
    pub async fn discover(&self, url: &str) -> Result<AgentCard, String> {
        let agent_json_url = format!("{}/.well-known/agent.json", url.trim_end_matches('/'));

        let response = self
            .client
            .get(&agent_json_url)
            .header("User-Agent", "OpenFang/0.1 A2A")
            .send()
            .await
            .map_err(|e| format!("A2A discovery failed: {e}"))?;

        if !response.status().is_success() {
            return Err(format!("A2A discovery returned {}", response.status()));
        }

        let card: AgentCard = response
            .json()
            .await
            .map_err(|e| format!("Invalid Agent Card: {e}"))?;

        Ok(card)
    }

    /// Send a task to an external A2A agent.
    pub async fn send_task(
        &self,
        url: &str,
        message: &str,
        session_id: Option<&str>,
    ) -> Result<A2aTask, String> {
        let request = serde_json::json!({
            "jsonrpc": "2.0",
            "id": 1,
            "method": "tasks/send",
            "params": {
                "message": {
                    "role": "user",
                    "parts": [{"type": "text", "text": message}]
                },
                "sessionId": session_id,
            }
        });

        let response = self
            .client
            .post(url)
            .json(&request)
            .send()
            .await
            .map_err(|e| format!("A2A send_task failed: {e}"))?;

        let body: serde_json::Value = response
            .json()
            .await
            .map_err(|e| format!("Invalid A2A response: {e}"))?;

        if let Some(result) = body.get("result") {
            serde_json::from_value(result.clone())
                .map_err(|e| format!("Invalid A2A task response: {e}"))
        } else if let Some(error) = body.get("error") {
            Err(format!("A2A error: {}", error))
        } else {
            Err("Empty A2A response".to_string())
        }
    }
}
```

### 3.2 任务状态流转

```
Submitted → Working → InputRequired → Working → Completed
                      ↓                    ↓
                  Working              Cancelled
                                         ↓
                                      Failed
```

### 3.3 JSON-RPC 2.0 格式

```json
// 请求
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tasks/send",
  "params": {
    "message": {"role": "user", "parts": [{"type": "text", "text": "Hello"}],
    "sessionId": "session-123"
  }
}

// 响应
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "id": "task-abc",
    "status": "working",
    "messages": [],
    "artifacts": []
  }
}
```

## 4. 集成与使用

### 4.1 API 端点

#### 4.1.1 Server 端端点（本地 Agent 暴露）

| 端点 | 方法 | 用途 |
|------|------|------|
| `/.well-known/agent.json` | GET | 默认 Agent 的 AgentCard |
| `/a2a/agents` | GET | 所有本地 Agent 的 Card 列表 |
| `/a2a/tasks/send` | POST | 提交任务给本地 Agent |
| `/a2a/tasks/{id}` | GET | 获取本地任务状态 |
| `/a2a/tasks/{id}/cancel` | POST | 取消本地任务 |

#### 4.1.2 Client 端端点（外部 Agent 交互）

| 端点 | 方法 | 用途 |
|------|------|------|
| `/api/a2a/agents` | GET | 已发现的外部 Agent 列表 |
| `/api/a2a/discover` | POST | 发现新的外部 Agent |
| `/api/a2a/send` | POST | 发送任务给外部 Agent |
| `/api/a2a/tasks/{id}/status` | GET | 获取外部任务状态 |

### 4.2 配置示例

#### 4.2.1 OFP 配置

```toml
[network]
# 节点唯一标识（默认自动生成）
node_id = "openfang-node-1"
# 节点名称
node_name = "OpenFang Server"
# 预共享密钥（必需）
shared_secret = "your-secret-key-here"
# 监听地址
listen_addr = "0.0.0.0:3030"
```

#### 4.2.2 A2A 外部 Agent 配置

```toml
[external_agents]
# 外部 A2A Agent 列表
[[external_agents.agents]]
name = "coding-agent"
url = "https://coding-agent.example.com"

[[external_agents.agents]]
name = "research-agent"
url = "https://research-agent.example.com"
```

## 5. 安全考虑

### 5.1 OFP 安全特性

- **双向认证**：使用 HMAC-SHA256 确保双方身份可信
- **防重放攻击**：5 分钟时间窗口的 Nonce 追踪
- **每会话密钥**：基于双方 nonce 派生的会话密钥
- **消息认证**：每条消息都有 HMAC 签名
- **常量时间比较**：防止时序攻击

### 5.2 A2A 安全考虑

- **能力发现**：Agent Card 公开能力但不暴露敏感信息
- **任务隔离**：每个任务都有唯一 ID，确保消息路由正确
- **状态管理**：明确的任务状态流转，便于跟踪和审计
- **超时处理**：客户端有 30 秒超时设置

## 6. 使用场景

### 6.1 OFP 使用场景

1. **跨机器 Agent 通信**：不同服务器上的 OpenFang 实例之间的通信
2. **Agent 发现**：自动发现网络中的其他 Agent
3. **安全消息传递**：在不安全网络中传递敏感消息
4. **集群管理**：多节点 OpenFang 集群的协调

### 6.2 A2A 使用场景

1. **跨框架协作**：与其他 A2A 兼容的 Agent 框架交互
2. **能力调用**：使用外部 Agent 的特定能力
3. **任务委派**：将复杂任务分解给专业 Agent
4. **服务集成**：将 OpenFang 集成到现有系统中

## 7. 代码优化建议

### 7.1 OFP 优化建议

1. **TLS 集成**：在 TCP 基础上添加 TLS 加密，进一步增强安全性
2. **连接池**：实现连接池，减少频繁建立连接的开销
3. **心跳机制**：增强心跳机制，及时检测连接状态
4. **错误恢复**：添加更健壮的错误恢复机制
5. **流量控制**：实现流量控制，防止过载

### 7.2 A2A 优化建议

1. **缓存机制**：缓存 Agent Card 信息，减少重复发现的开销
2. **批量操作**：支持批量任务提交和状态查询
3. **异步处理**：增强异步处理能力，提高并发性能
4. **错误处理**：更详细的错误码和错误信息
5. **认证机制**：添加可选的认证机制，保护 A2A 端点

## 8. 总结

OFP 和 A2A 协议是 OpenFang 系统中实现通信的核心组件，它们共同构成了一个完整的通信体系：

- **OFP** 作为底层传输协议，提供安全、可靠的点对点通信能力，确保不同 OpenFang 实例之间的安全通信。
- **A2A** 作为上层应用协议，提供跨框架的 Agent 互操作能力，使 OpenFang 能够与其他 Agent 系统无缝集成。

这两个协议的设计考虑了安全性、可靠性和可扩展性，为 OpenFang 系统提供了强大的通信基础。通过合理配置和使用这些协议，可以构建更加灵活、安全、高效的 Agent 协作系统。