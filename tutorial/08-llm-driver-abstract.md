# 第 8 节：LLM Driver — 抽象层

> **版本**: v0.5.2 (2026-03-29)
> **核心文件**: `crates/openfang-runtime/src/llm_driver.rs`

## 学习目标

- [ ] 理解 LlmDriver trait 的设计和作用
- [ ] 掌握 CompletionRequest/Response 结构
- [ ] 理解 StreamEvent 流式事件类型
- [ ] 掌握 DriverConfig 配置结构

---

## 1. LlmDriver Trait — LLM 抽象层

### 文件位置
`crates/openfang-runtime/src/llm_driver.rs:133-159`

```rust
/// Trait for LLM drivers.
#[async_trait]
pub trait LlmDriver: Send + Sync {
    /// Send a completion request and get a response.
    async fn complete(&self, request: CompletionRequest) -> Result<CompletionResponse, LlmError>;

    /// Stream a completion request, sending incremental events to the channel.
    /// Returns the full response when complete. Default wraps `complete()`.
    async fn stream(
        &self,
        request: CompletionRequest,
        tx: tokio::sync::mpsc::Sender<StreamEvent>,
    ) -> Result<CompletionResponse, LlmError> {
        let response = self.complete(request).await?;
        let text = response.text();
        if !text.is_empty() {
            let _ = tx.send(StreamEvent::TextDelta { text }).await;
        }
        let _ = tx
            .send(StreamEvent::ContentComplete {
                stop_reason: response.stop_reason,
                usage: response.usage,
            })
            .await;
        Ok(response)
    }
}
```

**设计要点**：

| 方法 | 说明 | 默认实现 |
|------|------|----------|
| `complete()` | 发送完成请求并返回响应 | 必须由实现者提供 |
| `stream()` | 流式完成，发送增量事件 | 默认包装 `complete()` |

**关键设计**：
- `Send + Sync`：支持多线程/异步环境
- `async_trait`：异步 trait 需要宏支持
- `stream()` 默认实现：简化简单 provider 的接入（如 Ollama 无需流式）

---

## 2. CompletionRequest — 请求结构

### 文件位置
`crates/openfang-runtime/src/llm_driver.rs:51-68`

```rust
/// A request to an LLM for completion.
#[derive(Debug, Clone)]
pub struct CompletionRequest {
    /// Model identifier.
    pub model: String,
    /// Conversation messages.
    pub messages: Vec<Message>,
    /// Available tools the model can use.
    pub tools: Vec<ToolDefinition>,
    /// Maximum tokens to generate.
    pub max_tokens: u32,
    /// Sampling temperature.
    pub temperature: f32,
    /// System prompt (extracted from messages for APIs that need it separately).
    pub system: Option<String>,
    /// Extended thinking/reasoning configuration (if supported by the model).
    pub thinking: Option<openfang_types::config::ThinkingConfig>,
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `model` | `String` | 模型标识符（如 `claude-sonnet-4-20250514`） |
| `messages` | `Vec<Message>` | 对话消息历史 |
| `tools` | `Vec<ToolDefinition>` | 可用工具定义列表 |
| `max_tokens` | `u32` | 最大生成 token 数 |
| `temperature` | `f32` | 采样温度（0.0-2.0） |
| `system` | `Option<String>` | 系统提示（部分 API 需要单独传递） |
| `thinking` | `Option<ThinkingConfig>` | 扩展思考配置（如 o1 系列模型） |

### 统一抽象的意义

**问题**：不同 provider 的 API 格式不同
- Anthropic: system prompt 单独传
- OpenAI: system prompt 作为第一条消息
- Gemini: 使用不同的 JSON 结构

**解决**：统一 `CompletionRequest` 结构，各 driver 内部转换
- Agent Loop 只关心统一的请求格式
- Driver 实现负责适配各自 API

---

## 3. CompletionResponse — 响应结构

### 文件位置
`crates/openfang-runtime/src/llm_driver.rs:70-96`

```rust
/// A response from an LLM completion.
#[derive(Debug, Clone)]
pub struct CompletionResponse {
    /// The content blocks in the response.
    pub content: Vec<ContentBlock>,
    /// Why the model stopped generating.
    pub stop_reason: StopReason,
    /// Tool calls extracted from the response.
    pub tool_calls: Vec<ToolCall>,
    /// Token usage statistics.
    pub usage: TokenUsage,
}

impl CompletionResponse {
    /// Extract text content from the response.
    pub fn text(&self) -> String {
        self.content
            .iter()
            .filter_map(|block| match block {
                ContentBlock::Text { text, .. } => Some(text.as_str()),
                ContentBlock::Thinking { .. } => None,
                _ => None,
            })
            .collect::<Vec<_>>()
            .join("")
    }
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `content` | `Vec<ContentBlock>` | 内容块列表（文本/工具/思考） |
| `stop_reason` | `StopReason` | 停止原因（EndTurn/ToolUse/MaxTokens 等） |
| `tool_calls` | `Vec<ToolCall>` | 提取的工具调用 |
| `usage` | `TokenUsage` | Token 使用统计 |

### text() 方法

```rust
// 只提取文本块，过滤思考内容和未知块
pub fn text(&self) -> String {
    self.content
        .iter()
        .filter_map(|block| match block {
            ContentBlock::Text { text, .. } => Some(text.as_str()),
            ContentBlock::Thinking { .. } => None,  // 过滤思考
            _ => None,
        })
        .collect::<Vec<_>>()
        .join("")
}
```

**用途**：Agent Loop 中提取纯文本响应用于展示和保存。

---

## 4. StreamEvent — 流式事件

### 文件位置
`crates/openfang-runtime/src/llm_driver.rs:98-131`

```rust
/// Events emitted during streaming LLM completion.
#[derive(Debug, Clone)]
pub enum StreamEvent {
    /// Incremental text content.
    TextDelta { text: String },
    /// A tool use block has started.
    ToolUseStart { id: String, name: String },
    /// Incremental JSON input for an in-progress tool use.
    ToolInputDelta { text: String },
    /// A tool use block is complete with parsed input.
    ToolUseEnd {
        id: String,
        name: String,
        input: serde_json::Value,
    },
    /// Incremental thinking/reasoning text.
    ThinkingDelta { text: String },
    /// The entire response is complete.
    ContentComplete {
        stop_reason: StopReason,
        usage: TokenUsage,
    },
    /// Agent lifecycle phase change (for UX indicators).
    PhaseChange {
        phase: String,
        detail: Option<String>,
    },
    /// Tool execution completed with result (emitted by agent loop, not LLM driver).
    ToolExecutionResult {
        name: String,
        result_preview: String,
        is_error: bool,
    },
}
```

### 事件类型分类

| 分类 | 事件 | 发送者 |
|------|------|--------|
| **文本流式** | `TextDelta` | LLM Driver |
| **工具调用** | `ToolUseStart`, `ToolInputDelta`, `ToolUseEnd` | LLM Driver |
| **思考过程** | `ThinkingDelta` | LLM Driver (支持思考的模型) |
| **完成事件** | `ContentComplete` | LLM Driver |
| **生命周期** | `PhaseChange` | Agent Loop |
| **工具结果** | `ToolExecutionResult` | Agent Loop |

### 事件流转示例

```
LLM Driver 流式输出：
  ToolUseStart { id: "t1", name: "web_search" }
  → ToolInputDelta { text: "{\"q" }
  → ToolInputDelta { text: "uery\": \"rust\"}" }
  → ToolUseEnd { id: "t1", name: "web_search", input: {...} }
  → TextDelta { text: "正在搜索..." }
  → ContentComplete { stop_reason: EndTurn, usage: {...} }

Agent Loop 追加：
  → PhaseChange { phase: "Executing", detail: Some("web_search") }
  → ToolExecutionResult { name: "web_search", result_preview: "...", is_error: false }
  → PhaseChange { phase: "Thinking" }
```

---

## 5. LlmError — 错误类型

### 文件位置
`crates/openfang-runtime/src/llm_driver.rs:12-49`

```rust
/// Error type for LLM driver operations.
#[derive(Error, Debug)]
pub enum LlmError {
    /// HTTP request failed.
    #[error("HTTP error: {0}")]
    Http(String),
    /// API returned an error.
    #[error("API error ({status}): {message}")]
    Api {
        status: u16,
        message: String,
    },
    /// Rate limited — should retry after delay.
    #[error("Rate limited, retry after {retry_after_ms}ms")]
    RateLimited { retry_after_ms: u64 },
    /// Response parsing failed.
    #[error("Parse error: {0}")]
    Parse(String),
    /// No API key configured.
    #[error("Missing API key: {0}")]
    MissingApiKey(String),
    /// Model overloaded.
    #[error("Model overloaded, retry after {retry_after_ms}ms")]
    Overloaded { retry_after_ms: u64 },
    /// Authentication failed (invalid/missing API key).
    #[error("Authentication failed: {0}")]
    AuthenticationFailed(String),
    /// Model not found.
    #[error("Model not found: {0}")]
    ModelNotFound(String),
}
```

### 错误分类与重试策略

| 错误类型 | 可重试 | 处理方式 |
|----------|--------|----------|
| `Http` | 是 | 网络波动，指数退避 |
| `RateLimited` | 是 | 等待 `retry_after_ms` 后重试 |
| `Overloaded` | 是 | 等待 `retry_after_ms` 后重试 |
| `Parse` | 否 | API 响应格式错误，需修复代码 |
| `MissingApiKey` | 否 | 配置问题，用户需设置 API key |
| `AuthenticationFailed` | 否 | API key 无效，用户需更新 |
| `ModelNotFound` | 否 | 模型名称错误或不可用 |

**注意**：第 7 节详细讲解了这些错误如何在 `call_with_retry` 中被分类和处理。

---

## 6. DriverConfig — 驱动配置

### 文件位置
`crates/openfang-runtime/src/llm_driver.rs:161-195`

```rust
/// Configuration for creating an LLM driver.
#[derive(Clone, Serialize, Deserialize)]
pub struct DriverConfig {
    /// Provider name.
    pub provider: String,
    /// API key.
    pub api_key: Option<String>,
    /// Base URL override.
    pub base_url: Option<String>,
    /// Skip interactive permission prompts (Claude Code provider only).
    #[serde(default = "default_skip_permissions")]
    pub skip_permissions: bool,
}

fn default_skip_permissions() -> bool {
    true
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `provider` | `String` | Provider 名称（如 `anthropic`, `openai`, `groq`） |
| `api_key` | `Option<String>` | API Key（部分 provider 可选，如 Ollama） |
| `base_url` | `Option<String>` | 自定义 Base URL（用于私有部署/本地模型） |
| `skip_permissions` | `bool` | 跳过权限确认（仅 Claude Code 使用） |

### SECURITY: Debug 实现脱敏

```rust
/// SECURITY: Custom Debug impl redacts the API key.
impl std::fmt::Debug for DriverConfig {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("DriverConfig")
            .field("provider", &self.provider)
            .field("api_key", &self.api_key.as_ref().map(|_| "<redacted>"))
            .field("base_url", &self.base_url)
            .field("skip_permissions", &self.skip_permissions)
            .finish()
    }
}
```

**安全设计**：
- `api_key` 字段在日志中显示为 `<redacted>`
- 防止敏感信息泄露到日志文件

---

## 7. 默认 Stream 实现详解

### 文件位置
`crates/openfang-runtime/src/llm_driver.rs:140-158`

```rust
async fn stream(
    &self,
    request: CompletionRequest,
    tx: tokio::sync::mpsc::Sender<StreamEvent>,
) -> Result<CompletionResponse, LlmError> {
    let response = self.complete(request).await?;
    let text = response.text();
    if !text.is_empty() {
        let _ = tx.send(StreamEvent::TextDelta { text }).await;
    }
    let _ = tx
        .send(StreamEvent::ContentComplete {
            stop_reason: response.stop_reason,
            usage: response.usage,
        })
        .await;
    Ok(response)
}
```

**设计意图**：
1. **简化接入**：新 provider 只需实现 `complete()` 即可获得流式能力
2. **降级处理**：不支持真正流式的 provider（如本地模型）也能工作
3. **事件兼容**：UI 层可以统一使用流式接口

**哪些 driver 使用默认实现**：
- Ollama（本地模型，无需流式）
- vLLM（本地推理，流式意义不大）
- LM Studio（本地 GUI 工具）

**哪些 driver 实现真正流式**：
- AnthropicDriver（SSE 流式）
- OpenAIDriver（SSE 流式）
- GeminiDriver（SSE 流式）

---

## 8. 测试代码示例

### 文件位置
`crates/openfang-runtime/src/llm_driver.rs:259-313`

```rust
#[tokio::test]
async fn test_default_stream_sends_events() {
    use tokio::sync::mpsc;

    struct FakeDriver;

    #[async_trait]
    impl LlmDriver for FakeDriver {
        async fn complete(
            &self,
            _request: CompletionRequest,
        ) -> Result<CompletionResponse, LlmError> {
            Ok(CompletionResponse {
                content: vec![ContentBlock::Text {
                    text: "Hello!".to_string(),
                    provider_metadata: None,
                }],
                stop_reason: StopReason::EndTurn,
                tool_calls: vec![],
                usage: TokenUsage {
                    input_tokens: 5,
                    output_tokens: 3,
                },
            })
        }
    }

    let driver = FakeDriver;
    let (tx, mut rx) = mpsc::channel(16);
    let request = CompletionRequest {
        model: "test".to_string(),
        messages: vec![],
        tools: vec![],
        max_tokens: 100,
        temperature: 0.0,
        system: None,
        thinking: None,
    };

    let response = driver.stream(request, tx).await.unwrap();
    assert_eq!(response.text(), "Hello!");

    // Should receive TextDelta then ContentComplete
    let ev1 = rx.recv().await.unwrap();
    assert!(matches!(ev1, StreamEvent::TextDelta { text } if text == "Hello!"));

    let ev2 = rx.recv().await.unwrap();
    assert!(matches!(
        ev2,
        StreamEvent::ContentComplete {
            stop_reason: StopReason::EndTurn,
            ..
        }
    ));
}
```

**测试验证**：
1. `complete()` 返回的响应正确
2. `stream()` 默认实现发送 `TextDelta` 事件
3. `stream()` 默认实现发送 `ContentComplete` 事件

---

## 9. 关键设计点

### 9.1 Trait 对象用于多态

```rust
let driver: Arc<dyn LlmDriver> = ...;  // 可以是任何实现

// 调用时自动分发到具体实现
driver.complete(request).await?;
```

**优点**：
- Agent Loop 不关心具体 provider
- 运行时可以切换 driver（Fallback 机制）
- 易于扩展新 provider

### 9.2 统一请求/响应结构

```
┌─────────────────────────────────────────────────────┐
│                  Agent Loop                         │
│                     ↓                               │
│          CompletionRequest                          │
│                     ↓                               │
│  ┌──────────────┬──────────────┬──────────────┐    │
│  │  Anthropic   │   OpenAI     │   Gemini     │    │
│  │    Driver    │    Driver    │    Driver    │    │
│  └──────────────┴──────────────┴──────────────┘    │
│                     ↓                               │
│          CompletionResponse                         │
└─────────────────────────────────────────────────────┘
```

### 9.3 流式事件分层

| 层级 | 事件类型 | 职责 |
|------|----------|------|
| **LLM Driver** | `TextDelta`, `ToolUseStart`, `ToolInputDelta`, `ToolUseEnd`, `ThinkingDelta`, `ContentComplete` | 负责解析 API 响应并发送增量事件 |
| **Agent Loop** | `PhaseChange`, `ToolExecutionResult` | 负责生命周期管理和工具执行结果通知 |

**分层好处**：
- Driver 只关心 LLM API 交互
- Agent Loop 负责业务逻辑
- UI 层可以订阅所有事件构建丰富的 UX

### 9.4 错误分类与处理

```rust
match driver.complete(request).await {
    Ok(response) => { /* 成功处理 */ }
    Err(LlmError::RateLimited { retry_after_ms }) => {
        // 等待后重试
    }
    Err(LlmError::AuthenticationFailed(_)) => {
        // 提示用户更新 API key
    }
    Err(LlmError::ModelNotFound(_)) => {
        // 切换到 fallback provider
    }
    // ...
}
```

---

## 完成检查清单

- [ ] 理解 LlmDriver trait 的设计和作用
- [ ] 掌握 CompletionRequest/Response 结构
- [ ] 理解 StreamEvent 流式事件类型
- [ ] 掌握 DriverConfig 配置结构

---

## 下一步

前往 [第 9 节：LLM Driver — 实现](./09-llm-driver-implementations.md)

---

*创建时间：2026-03-15*
*OpenFang v0.5.2*
