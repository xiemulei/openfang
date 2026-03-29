# 第 19 节：A2A 协议 — Agent 间通信

> **版本**: v0.5.2 (2026-03-29)
> **核心文件**:
> - `crates/openfang-runtime/src/a2a.rs`
> - `crates/openfang-api/src/routes.rs` (A2A 端点)

## 学习目标

- [ ] 理解 A2A 协议的设计目标和架构
- [ ] 掌握 AgentCard 结构和能力描述
- [ ] 掌握 A2aTask 任务生命周期和状态流转
- [ ] 理解 A2aMessage 和 A2aPart 消息格式
- [ ] 掌握 A2aTaskStore 任务存储和状态管理
- [ ] 理解 A2aClient 发现和交互外部 Agent

---

## 1. A2A 协议概览

### 协议目标

A2A (Agent-to-Agent) Protocol 是一个跨框架的 Agent 互操作协议，提供：

| 目标 | 说明 |
|------|------|
| **跨框架互操作** | 不同 Agent 框架之间可以通信和协作 |
| **能力发现** | 通过 Agent Card 公开 Agent 的能力 |
| **任务协调** | 基于任务的协作模式（提交 - 执行 - 完成） |
| **流式支持** | 支持流式响应和实时状态更新 |

### 与 OFP 协议的关系

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

**关键区别**：
- **OFP**: OpenFang 内核之间的底层通信（传输层）
- **A2A**: 跨框架的通用 Agent 协议（应用层）

---

## 2. AgentCard — Agent 能力卡片

### 文件位置
`crates/openfang-runtime/src/a2a.rs:22-46`

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

### 字段说明

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `name` | `String` | Agent 显示名称 | `"Coder Agent"` |
| `description` | `String` | 功能描述 | `"A coding assistant"` |
| `url` | `String` | Agent 端点 URL | `"https://example.com/a2a"` |
| `version` | `String` | 协议版本 | `"0.1.0"` |
| `capabilities` | `AgentCapabilities` | 能力描述 | 见下节 |
| `skills` | `Vec<AgentSkill>` | 技能列表 | 见下节 |
| `default_input_modes` | `Vec<String>` | 支持的输入模式 | `["text", "image"]` |
| `default_output_modes` | `Vec<String>` | 支持的输出模式 | `["text", "json"]` |

### 发现端点

Agent Card 通过 `/.well-known/agent.json` 端点公开：

```
GET https://example.com/.well-known/agent.json
→ 返回 AgentCard JSON
```

---

## 3. AgentCapabilities — 能力描述

### 文件位置
`crates/openfang-runtime/src/a2a.rs:48-58`

```rust
/// A2A agent capabilities.
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct AgentCapabilities {
    /// Whether this agent supports streaming responses.
    pub streaming: bool,
    /// Whether this agent supports push notifications.
    pub push_notifications: bool,
    /// Whether task status history is available.
    pub state_transition_history: bool,
}
```

### 能力标志

| 字段 | 类型 | 说明 |
|------|------|------|
| `streaming` | `bool` | 是否支持流式响应 |
| `push_notifications` | `bool` | 是否支持推送通知 |
| `state_transition_history` | `bool` | 是否提供状态历史 |

---

## 4. AgentSkill — 技能描述

### 文件位置
`crates/openfang-runtime/src/a2a.rs:60-75`

```rust
/// A2A skill descriptor (not an OpenFang skill — describes a capability).
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AgentSkill {
    /// Unique skill identifier.
    pub id: String,
    /// Display name.
    pub name: String,
    /// Description of what this skill does.
    pub description: String,
    /// Tags for discovery.
    #[serde(default)]
    pub tags: Vec<String>,
    /// Example prompts that trigger this skill.
    #[serde(default)]
    pub examples: Vec<String>,
}
```

### 字段说明

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `id` | `String` | 唯一技能标识 | `"file_read"` |
| `name` | `String` | 显示名称 | `"File Reader"` |
| `description` | `String` | 技能描述 | `"Can read files from disk"` |
| `tags` | `Vec<String>` | 分类标签 | `["io", "filesystem"]` |
| `examples` | `Vec<String>` | 示例提示 | `["Read config.toml"]` |

---

## 5. A2aTask — 任务单元

### 文件位置
`crates/openfang-runtime/src/a2a.rs:81-98`

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

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `String` | 任务唯一标识 |
| `session_id` | `Option<String>` | 会话 ID（保持对话连续性） |
| `status` | `A2aTaskStatusWrapper` | 当前任务状态 |
| `messages` | `Vec<A2aMessage>` | 任务期间的消息历史 |
| `artifacts` | `Vec<A2aArtifact>` | 任务产生的工件 |

---

## 6. A2aTaskStatus — 任务状态

### 文件位置
`crates/openfang-runtime/src/a2a.rs:100-154`

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

/// Wrapper that accepts either a bare status string or object form.
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(untagged)]
pub enum A2aTaskStatusWrapper {
    /// Object form: `{"state": "completed", "message": ...}`.
    Object {
        state: A2aTaskStatus,
        #[serde(default)]
        message: Option<serde_json::Value>,
    },
    /// Bare enum form: `"completed"`.
    Enum(A2aTaskStatus),
}
```

### 状态流转图

```
Submitted → Working → InputRequired → Working → Completed
                      ↓                    ↓
                  Working              Cancelled
                                         ↓
                                      Failed
```

### 状态说明

| 状态 | 说明 | 触发条件 |
|------|------|----------|
| `Submitted` | 任务已提交但未开始 | 任务创建时 |
| `Working` | 任务正在处理 | 开始执行 |
| `InputRequired` | 需要更多输入 | Agent 请求更多信息 |
| `Completed` | 任务成功完成 | 执行成功 |
| `Cancelled` | 任务被取消 | 用户取消 |
| `Failed` | 任务失败 | 执行出错 |

### 双格式兼容

A2A 规范允许两种状态编码格式：

```json
// 格式 1: 裸字符串
"completed"

// 格式 2: 对象格式
{"state": "completed", "message": {"text": "Done!"}}
```

`A2aTaskStatusWrapper` 自动处理两种格式的反序列化。

---

## 7. A2aMessage — 任务消息

### 文件位置
`crates/openfang-runtime/src/a2a.rs:156-163`

```rust
/// A2A message in a task conversation.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct A2aMessage {
    /// Message role ("user" or "agent").
    pub role: String,
    /// Message content parts.
    pub parts: Vec<A2aPart>,
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `role` | `String` | 消息角色（`"user"` 或 `"agent"`） |
| `parts` | `Vec<A2aPart>` | 消息内容片段 |

---

## 8. A2aPart — 消息内容片段

### 文件位置
`crates/openfang-runtime/src/a2a.rs:165-182`

```rust
/// A2A message content part.
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "camelCase")]
pub enum A2aPart {
    /// Text content.
    Text { text: String },
    /// File content (base64-encoded).
    File {
        name: String,
        mime_type: String,
        data: String,
    },
    /// Structured data.
    Data {
        mime_type: String,
        data: serde_json::Value,
    },
}
```

### 内容类型

| 变体 | 用途 | 字段 |
|------|------|------|
| `Text` | 文本内容 | `text: String` |
| `File` | 文件内容（Base64） | `name`, `mime_type`, `data` |
| `Data` | 结构化数据 | `mime_type`, `data` |

### 使用示例

```rust
// 文本消息
A2aPart::Text { text: "Hello, world!".to_string() }

// 文件消息
A2aPart::File {
    name: "document.pdf".to_string(),
    mime_type: "application/pdf".to_string(),
    data: base64_content.to_string(),
}

// 结构化数据
A2aPart::Data {
    mime_type: "application/json".to_string(),
    data: serde_json::json!({"key": "value"}),
}
```

---

## 9. A2aArtifact — 任务工件

### 文件位置
`crates/openfang-runtime/src/a2a.rs:184-205`

```rust
/// A2A artifact produced by a task.
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct A2aArtifact {
    /// Artifact name (optional per spec).
    #[serde(default)]
    pub name: Option<String>,
    /// Human-readable description.
    #[serde(default)]
    pub description: Option<String>,
    /// Arbitrary metadata.
    #[serde(default)]
    pub metadata: Option<serde_json::Value>,
    /// Artifact index in the sequence.
    #[serde(default)]
    pub index: Option<u32>,
    /// Whether this is the last chunk of a streamed artifact.
    #[serde(default)]
    pub last_chunk: Option<bool>,
    /// Artifact content parts.
    pub parts: Vec<A2aPart>,
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `Option<String>` | 工件名称（可选） |
| `description` | `Option<String>` | 描述（可选） |
| `metadata` | `Option<serde_json::Value>` | 元数据（可选） |
| `index` | `Option<u32>` | 序列索引 |
| `last_chunk` | `Option<bool>` | 是否最后一个流式块 |
| `parts` | `Vec<A2aPart>` | 内容片段 |

---

## 10. A2aTaskStore — 任务存储

### 文件位置
`crates/openfang-runtime/src/a2a.rs:211-312`

```rust
/// In-memory store for tracking A2A task lifecycle.
#[derive(Debug)]
pub struct A2aTaskStore {
    tasks: Mutex<HashMap<String, A2aTask>>,
    /// Maximum number of tasks to retain (FIFO eviction).
    max_tasks: usize,
}

impl A2aTaskStore {
    /// Create a new task store with a capacity limit.
    pub fn new(max_tasks: usize) -> Self {
        Self {
            tasks: Mutex::new(HashMap::new()),
            max_tasks,
        }
    }
}
```

### 核心方法

| 方法 | 用途 | 返回值 |
|------|------|--------|
| `insert(task)` | 插入任务（满时淘汰最旧） | `()` |
| `get(task_id)` | 获取任务 | `Option<A2aTask>` |
| `update_status(task_id, status)` | 更新状态 | `bool` |
| `complete(task_id, response, artifacts)` | 完成任务 | `()` |
| `fail(task_id, error_message)` | 标记失败 | `()` |
| `cancel(task_id)` | 取消任务 | `bool` |
| `len()` | 任务数量 | `usize` |
| `is_empty()` | 是否为空 | `bool` |

### 淘汰策略

```rust
// a2a.rs:232-250
pub fn insert(&self, task: A2aTask) {
    let mut tasks = self.tasks.lock().unwrap_or_else(|e| e.into_inner());
    // 达到容量时淘汰已完成/失败/取消的任务
    if tasks.len() >= self.max_tasks {
        let evict_key = tasks
            .iter()
            .filter(|(_, t)| {
                matches!(
                    t.status.state(),
                    A2aTaskStatus::Completed | A2aTaskStatus::Failed | A2aTaskStatus::Cancelled
                )
            })
            .map(|(k, _)| k.clone())
            .next();
        if let Some(key) = evict_key {
            tasks.remove(&key);
        }
    }
    tasks.insert(task.id.clone(), task);
}
```

**淘汰规则**：
1. 优先淘汰已完成/失败/取消的任务
2. FIFO（先进先出）
3. 保留进行中的任务

---

## 11. A2aClient — 外部 Agent 发现

### 文件位置
`crates/openfang-runtime/src/a2a.rs:396-518`

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

    /// Get the status of a task from an external A2A agent.
    pub async fn get_task(&self, url: &str, task_id: &str) -> Result<A2aTask, String> {
        let request = serde_json::json!({
            "jsonrpc": "2.0",
            "id": 1,
            "method": "tasks/get",
            "params": {
                "id": task_id,
            }
        });

        let response = self
            .client
            .post(url)
            .json(&request)
            .send()
            .await
            .map_err(|e| format!("A2A get_task failed: {e}"))?;

        let body: serde_json::Value = response
            .json()
            .await
            .map_err(|e| format!("Invalid A2A response: {e}"))?;

        if let Some(result) = body.get("result") {
            serde_json::from_value(result.clone())
                .map_err(|e| format!("Invalid A2A task: {e}"))
        } else {
            Err("Empty A2A response".to_string())
        }
    }
}
```

### 方法说明

| 方法 | 用途 | JSON-RPC 方法 |
|------|------|--------------|
| `discover(url)` | 获取 Agent Card | N/A (HTTP GET) |
| `send_task(url, message, session_id)` | 提交任务 | `tasks/send` |
| `get_task(url, task_id)` | 获取任务状态 | `tasks/get` |

### JSON-RPC 2.0 格式

```json
// 请求
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tasks/send",
  "params": {
    "message": {"role": "user", "parts": [{"type": "text", "text": "Hello"}]},
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

---

## 12. build_agent_card — 从 Manifest 构建 AgentCard

### 文件位置
`crates/openfang-runtime/src/a2a.rs:360-390`

```rust
/// Build an A2A Agent Card from an OpenFang agent manifest.
pub fn build_agent_card(manifest: &AgentManifest, base_url: &str) -> AgentCard {
    let tools: Vec<String> = manifest.capabilities.tools.clone();

    // 将工具名转换为 A2A skill 描述
    let skills: Vec<AgentSkill> = tools
        .iter()
        .map(|tool| AgentSkill {
            id: tool.clone(),
            name: tool.replace('_', " "),
            description: format!("Can use the {tool} tool"),
            tags: vec!["tool".to_string()],
            examples: vec![],
        })
        .collect();

    AgentCard {
        name: manifest.name.clone(),
        description: manifest.description.clone(),
        url: format!("{base_url}/a2a"),
        version: "0.1.0".to_string(),
        capabilities: AgentCapabilities {
            streaming: true,
            push_notifications: false,
            state_transition_history: true,
        },
        skills,
        default_input_modes: vec!["text".to_string()],
        default_output_modes: vec!["text".to_string()],
    }
}
```

### 转换逻辑

1. **工具名 → Skills**：将 OpenFang 工具名转换为 A2A 技能描述
2. **默认能力**：启用流式、状态历史
3. **默认模式**：仅支持文本输入/输出

---

## 13. discover_external_agents — 启动时发现

### 文件位置
`crates/openfang-runtime/src/a2a.rs:318-354`

```rust
/// Discover all configured external A2A agents and return their cards.
///
/// Called during kernel boot to populate the list of known external agents.
pub async fn discover_external_agents(
    agents: &[openfang_types::config::ExternalAgent],
) -> Vec<(String, AgentCard)> {
    let client = A2aClient::new();
    let mut discovered = Vec::new();

    for agent in agents {
        match client.discover(&agent.url).await {
            Ok(card) => {
                info!(
                    name = %agent.name,
                    url = %agent.url,
                    skills = card.skills.len(),
                    "Discovered external A2A agent"
                );
                discovered.push((agent.name.clone(), card));
            }
            Err(e) => {
                warn!(
                    name = %agent.name,
                    url = %agent.url,
                    error = %e,
                    "Failed to discover external A2A agent"
                );
            }
        }
    }

    if !discovered.is_empty() {
        info!("A2A: discovered {} external agent(s)", discovered.len());
    }

    discovered
}
```

### 调用时机

在内核启动时调用：
1. 读取配置中的 `external_agents` 列表
2. 并行发现每个外部 Agent
3. 收集成功的 Agent Card
4. 失败时记录警告日志

---

## 14. API 端点

### 文件位置
`crates/openfang-api/src/routes.rs`

### Server 端端点（本地 Agent 暴露）

| 端点 | 方法 | 用途 |
|------|------|------|
| `/.well-known/agent.json` | GET | 默认 Agent 的 AgentCard |
| `/a2a/agents` | GET | 所有本地 Agent 的 Card 列表 |
| `/a2a/tasks/send` | POST | 提交任务给本地 Agent |
| `/a2a/tasks/{id}` | GET | 获取本地任务状态 |
| `/a2a/tasks/{id}/cancel` | POST | 取消本地任务 |

### Client 端端点（外部 Agent 交互）

| 端点 | 方法 | 用途 |
|------|------|------|
| `/api/a2a/agents` | GET | 已发现的外部 Agent 列表 |
| `/api/a2a/discover` | POST | 发现新的外部 Agent |
| `/api/a2a/send` | POST | 发送任务给外部 Agent |
| `/api/a2a/tasks/{id}/status` | GET | 获取外部任务状态 |

---

## 15. 测试用例

### 文件位置
`crates/openfang-runtime/src/a2a.rs:520-755`

### AgentCard 测试

```rust
#[test]
fn test_agent_card_from_manifest() {
    let manifest = AgentManifest {
        name: "test-agent".to_string(),
        description: "A test agent".to_string(),
        ..Default::default()
    };

    let card = build_agent_card(&manifest, "https://example.com");
    assert_eq!(card.name, "test-agent");
    assert_eq!(card.description, "A test agent");
    assert!(card.url.contains("/a2a"));
    assert!(card.capabilities.streaming);
    assert_eq!(card.default_input_modes, vec!["text"]);
}
```

### 任务状态流转测试

```rust
#[test]
fn test_a2a_task_status_transitions() {
    let task = A2aTask {
        id: "task-1".to_string(),
        session_id: None,
        status: A2aTaskStatus::Submitted.into(),
        messages: vec![],
        artifacts: vec![],
    };
    assert_eq!(task.status, A2aTaskStatus::Submitted);

    // 状态流转
    let working = A2aTask { status: A2aTaskStatus::Working.into(), ..task.clone() };
    assert_eq!(working.status, A2aTaskStatus::Working);

    let completed = A2aTask { status: A2aTaskStatus::Completed.into(), ..task.clone() };
    assert_eq!(completed.status, A2aTaskStatus::Completed);

    let cancelled = A2aTask { status: A2aTaskStatus::Cancelled.into(), ..task.clone() };
    assert_eq!(cancelled.status, A2aTaskStatus::Cancelled);

    let failed = A2aTask { status: A2aTaskStatus::Failed.into(), ..task };
    assert_eq!(failed.status, A2aTaskStatus::Failed);
}
```

### 双格式状态兼容测试

```rust
#[test]
fn test_a2a_task_status_wrapper_object_form() {
    // 对象格式
    let json = r#"{"state":"completed","message":null}"#;
    let wrapper: A2aTaskStatusWrapper = serde_json::from_str(json).unwrap();
    assert_eq!(wrapper, A2aTaskStatus::Completed);

    // 带消息的对象格式
    let json_with_msg = r#"{"state":"working","message":{"text":"Processing..."}}"#;
    let wrapper2: A2aTaskStatusWrapper = serde_json::from_str(json_with_msg).unwrap();
    assert_eq!(wrapper2, A2aTaskStatus::Working);

    // 裸字符串格式
    let json_bare = r#""completed""#;
    let wrapper3: A2aTaskStatusWrapper = serde_json::from_str(json_bare).unwrap();
    assert_eq!(wrapper3, A2aTaskStatus::Completed);
}
```

### TaskStore 测试

```rust
#[test]
fn test_task_store_insert_and_get() {
    let store = A2aTaskStore::new(10);
    let task = A2aTask {
        id: "t-1".to_string(),
        session_id: None,
        status: A2aTaskStatus::Working.into(),
        messages: vec![],
        artifacts: vec![],
    };
    store.insert(task);
    assert_eq!(store.len(), 1);

    let got = store.get("t-1").unwrap();
    assert_eq!(got.status, A2aTaskStatus::Working);
}

#[test]
fn test_task_store_eviction() {
    let store = A2aTaskStore::new(2);
    // 插入 2 个已完成任务
    for i in 0..2 {
        store.insert(A2aTask {
            id: format!("t-{i}"),
            status: A2aTaskStatus::Completed.into(),
            ..default_task()
        });
    }
    assert_eq!(store.len(), 2);

    // 插入第 3 个，应淘汰一个已完成任务
    store.insert(A2aTask {
        id: "t-2".to_string(),
        status: A2aTaskStatus::Working.into(),
        ..default_task()
    });
    assert!(store.len() <= 2);
}
```

---

## 16. 关键设计点

### 16.1 双格式状态兼容

```rust
#[serde(untagged)]
pub enum A2aTaskStatusWrapper {
    Object { state: A2aTaskStatus, message: Option<Value> },
    Enum(A2aTaskStatus),
}
```

**优势**：
- 兼容不同 A2A 实现
- 无需修改调用方代码
- 透明处理两种格式

### 16.2 任务存储淘汰

```
容量限制 → 优先淘汰终端状态任务 → 保留进行中任务
```

**优势**：
- 防止内存泄漏
- 保护进行中的任务
- 自动清理已完成任务

### 16.3 JSON-RPC 2.0

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "...",
  "params": {...}
}
```

**优势**：
- 标准化协议
- 错误处理统一
- 广泛工具支持

---

## 17. 与前后文的关联

### 与 OFP 协议（第 18 节）的关系

```
A2A (应用层)
  ↓ 使用
OFP (传输层)
  ↓ 使用
TCP (网络层)
```

### 与 EventBus（第 17 节）的关系

```
A2A 任务提交
  ↓
EventBus.publish(Event { payload: Message(...) })
  ↓
本地 Agent 接收
```

---

## 完成检查清单

- [ ] 理解 A2A 协议的设计目标和架构
- [ ] 掌握 AgentCard 结构和能力描述
- [ ] 掌握 A2aTask 任务生命周期和状态流转
- [ ] 理解 A2aMessage 和 A2aPart 消息格式
- [ ] 掌握 A2aTaskStore 任务存储和状态管理
- [ ] 理解 A2aClient 发现和交互外部 Agent

---

## 下一步

前往 [第 20 节：安全系统 — 污点追踪](./20-security-taint-tracking.md)

---

*创建时间：2026-03-15*
*OpenFang v0.4.4*
