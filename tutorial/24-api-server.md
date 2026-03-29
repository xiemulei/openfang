# 第 24 节：API 服务 — REST/WS 端点

> **版本**: v0.5.2 (2026-03-29)
> **核心文件**: `crates/openfang-api/src/server.rs`, `crates/openfang-api/src/routes.rs`
> **新增端点**: `/api/agents/{id}/restart`, `/api/hands/upsert`, `/api/config/schema`, `/api/comms/events/stream`, `/api/approvals` (增强版)

## 学习目标

- [ ] 理解 API 服务器架构和中间件栈
- [ ] 掌握 140+ REST 端点的分类和功能
- [ ] 理解 OpenAI 兼容 API 的实现
- [ ] 掌握 SSE 流式和 WebSocket 实时通信
- [ ] 理解 GCRA 速率限制和认证机制
- [ ] 了解 v0.4.9 新增端点 (Agent 重启、Hands 热更新等)

---

## 1. API 服务器架构

### 1.1 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      HTTP/HTTPS Server                          │
│                         (Axum + Tower)                          │
├─────────────────────────────────────────────────────────────────┤
│  Middleware Stack (入站 → 出站):                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. CORS (跨域控制)                                      │   │
│  │ 2. TraceLayer (请求追踪)                                │   │
│  │ 3. CompressionLayer (响应压缩)                          │   │
│  │ 4. SecurityHeaders (安全头)                             │   │
│  │ 5. RequestLogging (请求日志)                            │   │
│  │ 6. GCRA Rate Limiter (速率限制)                         │   │
│  │ 7. Auth Middleware (认证)                               │   │
│  └─────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                      Router (路由分发)                          │
│  ┌─────────────┬─────────────┬─────────────┬─────────────┐     │
│  │ /api/*      │ /v1/*       │ /a2a/*      │ /mcp        │     │
│  │ (REST API)  │ (OpenAI)    │ (A2A 协议)   │ (MCP 协议)   │     │
│  └─────────────┴─────────────┴─────────────┴─────────────┘     │
├─────────────────────────────────────────────────────────────────┤
│                    Handlers (业务逻辑)                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ routes.rs (140+ 端点处理函数)                            │   │
│  │ openai_compat.rs (OpenAI 兼容层)                         │   │
│  │ ws.rs (WebSocket 处理)                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    OpenFangKernel (核心)                        │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 路由构建流程

```rust
// crates/openfang-api/src/server.rs:37-54
pub async fn build_router(
    kernel: Arc<OpenFangKernel>,
    listen_addr: SocketAddr,
) -> (Router<()>, Arc<AppState>) {
    // 1. 启动 Channel Bridge (Telegram 等)
    let bridge = channel_bridge::start_channel_bridge(kernel.clone()).await;

    // 2. 创建应用状态
    let state = Arc::new(AppState {
        kernel: kernel.clone(),
        started_at: Instant::now(),
        peer_registry: kernel.peer_registry.get().map(|r| Arc::new(r.clone())),
        bridge_manager: tokio::sync::Mutex::new(bridge),
        channels_config: tokio::sync::RwLock::new(channels_config),
        shutdown_notify: Arc::new(tokio::sync::Notify::new()),
        clawhub_cache: dashmap::DashMap::new(),
        provider_probe_cache: ProbeCache::new(),
    });

    // 3. 构建中间件和路由...
}
```

### 1.3 中间件栈

```rust
// crates/openfang-api/src/server.rs:710-723
.app = app
    // ... 路由定义 ...
    .layer(axum::middleware::from_fn_with_state(
        auth_state,
        middleware::auth,  // 认证中间件
    ))
    .layer(axum::middleware::from_fn_with_state(
        gcra_limiter,
        rate_limiter::gcra_rate_limit,  // 速率限制
    ))
    .layer(axum::middleware::from_fn(middleware::security_headers))  // 安全头
    .layer(axum::middleware::from_fn(middleware::request_logging))  // 请求日志
    .layer(CompressionLayer::new())  // 响应压缩
    .layer(TraceLayer::new_for_http())  // 请求追踪
    .layer(cors)  // CORS
    .with_state(state.clone());
```

**中间件处理顺序**（请求入站）：
1. CORS → 2. TraceLayer → 3. Compression → 4. SecurityHeaders → 5. RequestLogging → 6. RateLimit → 7. Auth → Handler

---

## 2. REST API 端点大全

### 2.1 端点分类统计

| 类别 | 端点数 | 路径前缀 |
|------|--------|----------|
| **Agents** | 35+ | `/api/agents/*` |
| **Memory** | 6 | `/api/memory/*` |
| **Sessions** | 8 | `/api/sessions/*` |
| **Tools/Skills** | 12 | `/api/tools/*`, `/api/skills/*` |
| **MCP** | 4 | `/api/mcp/*`, `/mcp` |
| **A2A** | 8 | `/a2a/*`, `/api/a2a/*` |
| **Hands** | 14 | `/api/hands/*` |
| **Workflows** | 7 | `/api/workflows/*` |
| **Schedules/Cron** | 10 | `/api/schedules/*`, `/api/cron/*` |
| **Budget/Usage** | 8 | `/api/budget/*`, `/api/usage/*` |
| **Providers/Models** | 10 | `/api/providers/*`, `/api/models/*` |
| **Channels** | 8 | `/api/channels/*` |
| **Integrations** | 8 | `/api/integrations/*` |
| **Network/Peers** | 5 | `/api/peers`, `/api/network/*` |
| **Auth** | 4 | `/api/auth/*` |
| **OpenAI 兼容** | 2 | `/v1/*` |
| **其他** | 15+ | 各种管理端点 |

### 2.2 Agents 端点 (35+)

| 方法 | 路径 | 功能 | Token 成本 |
|------|------|------|-----------|
| `GET` | `/api/agents` | 列出所有 Agent | 2 |
| `POST` | `/api/agents` | 创建新 Agent | 50 |
| `GET` | `/api/agents/{id}` | 获取 Agent 详情 | 5 |
| `DELETE` | `/api/agents/{id}` | 删除 Agent | 10 |
| `PATCH` | `/api/agents/{id}` | 更新 Agent 配置 | 10 |
| `PUT` | `/api/agents/{id}/mode` | 设置运行模式 | 5 |
| `PUT` | `/api/agents/{id}/model` | 切换模型 | 5 |
| `POST` | `/api/agents/{id}/message` | 发送消息 | 30 |
| `POST` | `/api/agents/{id}/message/stream` | 流式消息 | 30 |
| `GET` | `/api/agents/{id}/session` | 获取会话 | 5 |
| `POST` | `/api/agents/{id}/session/reset` | 重置会话 | 10 |
| `POST` | `/api/agents/{id}/session/compact` | 压缩会话 | 20 |
| `GET` | `/api/agents/{id}/sessions` | 列出会话 | 5 |
| `POST` | `/api/agents/{id}/sessions` | 创建会话 | 10 |
| `POST` | `/api/agents/{id}/sessions/{session_id}/switch` | 切换会话 | 5 |
| `GET` | `/api/agents/{id}/history` | 获取历史 | 5 |
| `DELETE` | `/api/agents/{id}/history` | 清除历史 | 10 |
| `GET` | `/api/agents/{id}/tools` | 获取工具列表 | 2 |
| `PUT` | `/api/agents/{id}/tools` | 设置工具 | 10 |
| `GET` | `/api/agents/{id}/skills` | 获取技能 | 2 |
| `PUT` | `/api/agents/{id}/skills` | 设置技能 | 10 |
| `GET` | `/api/agents/{id}/mcp_servers` | 获取 MCP 服务器 | 2 |
| `PUT` | `/api/agents/{id}/mcp_servers` | 设置 MCP 服务器 | 10 |
| `PATCH` | `/api/agents/{id}/identity` | 更新身份 | 5 |
| `PATCH` | `/api/agents/{id}/config` | 更新配置 (**v0.5.1**: 改进 Provider/Model 配置同步) |
| `POST` | `/api/agents/{id}/clone` | 克隆 Agent | 20 |
| `GET` | `/api/agents/{id}/files` | 列出文件 | 2 |
| `GET` | `/api/agents/{id}/files/{filename}` | 获取文件 | 2 |
| `PUT` | `/api/agents/{id}/files/{filename}` | 设置文件 | 10 |
| `GET` | `/api/agents/{id}/deliveries` | 获取交付物 | 2 |
| `POST` | `/api/agents/{id}/upload` | 上传文件 | 20 |
| `GET` | `/api/agents/{id}/ws` | WebSocket 连接 | - |
| `POST` | `/api/agents/{id}/stop` | 停止 Agent | 10 |
| `PUT` | `/api/agents/{id}/update` | 更新 Agent | 10 |
| `GET` | `/api/agents/{id}/files/{filename}` | 文件服务 | 2 |
| `POST` | `/api/agents/{id}/restart` | 重启 Agent (v0.4.9 新增) | 10 |

### 2.3 Memory 端点

```rust
// crates/openfang-api/src/server.rs:265-274
.route(
    "/api/memory/agents/{id}/kv",
    axum::routing::get(routes::get_agent_kv),
)
.route(
    "/api/memory/agents/{id}/kv/{key}",
    axum::routing::get(routes::get_agent_kv_key)
        .put(routes::set_agent_kv_key)
        .delete(routes::delete_agent_kv_key),
)
```

| 方法 | 路径 | 功能 |
|------|------|------|
| `GET` | `/api/memory/agents/{id}/kv` | 列出所有 KV |
| `GET` | `/api/memory/agents/{id}/kv/{key}` | 获取键值 |
| `PUT` | `/api/memory/agents/{id}/kv/{key}` | 设置键值 |
| `DELETE` | `/api/memory/agents/{id}/kv/{key}` | 删除键值 |

### 2.4 Sessions 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| `GET` | `/api/sessions` | 列出所有会话 |
| `DELETE` | `/api/sessions/{id}` | 删除会话 |
| `PUT` | `/api/sessions/{id}/label` | 设置标签 |
| `GET` | `/api/agents/{id}/sessions/by-label/{label}` | 按标签查找 |

### 2.5 Tools & Skills 端点

| 方法 | 路径 | 功能 | Token 成本 |
|------|------|------|-----------|
| `GET` | `/api/tools` | 列出工具 | 1 |
| `GET` | `/api/skills` | 列出技能 | 2 |
| `POST` | `/api/skills/install` | 安装技能 | 50 |
| `POST` | `/api/skills/uninstall` | 卸载技能 | 10 |
| `POST` | `/api/skills/create` | 创建技能 | 20 |
| `GET` | `/api/marketplace/search` | 市场搜索 | 10 |

### 2.6 ClawHub (OpenClaw 生态系统)

| 方法 | 路径 | 功能 |
|------|------|------|
| `GET` | `/api/clawhub/search` | 搜索 ClawHub |
| `GET` | `/api/clawhub/browse` | 浏览 ClawHub |
| `GET` | `/api/clawhub/skill/{slug}` | 技能详情 |
| `GET` | `/api/clawhub/skill/{slug}/code` | 获取代码 |
| `POST` | `/api/clawhub/install` | 安装 ClawHub 技能 |

### 2.7 Hands 端点 (自主手系统)

| 方法 | 路径 | 功能 |
|------|------|------|
| `GET` | `/api/hands` | 列出所有 Hands |
| `POST` | `/api/hands/install` | 安装 Hand |
| `POST` | `/api/hands/upsert` | 创建/更新 Hand |
| `GET` | `/api/hands/active` | 列出活跃 Hands |
| `GET` | `/api/hands/{hand_id}` | 获取 Hand 详情 |
| `POST` | `/api/hands/{hand_id}/activate` | 激活 Hand |
| `POST` | `/api/hands/{hand_id}/check-deps` | 检查依赖 |
| `POST` | `/api/hands/{hand_id}/install-deps` | 安装依赖 |
| `GET` | `/api/hands/{hand_id}/settings` | 获取设置 |
| `PUT` | `/api/hands/{hand_id}/settings` | 更新设置 |
| `POST` | `/api/hands/instances/{id}/pause` | 暂停实例 |
| `POST` | `/api/hands/instances/{id}/resume` | 恢复实例 |
| `DELETE` | `/api/hands/instances/{id}` | 停用实例 |
| `GET` | `/api/hands/instances/{id}/stats` | 获取统计 |
| `GET` | `/api/hands/instances/{id}/browser` | 浏览器集成 |

### 2.8 Workflows 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| `GET` | `/api/workflows` | 列出工作流 |
| `POST` | `/api/workflows` | 创建工作流 (**v0.5.1**: 自动持久化到磁盘) |
| `GET` | `/api/workflows/{id}` | 获取详情 |
| `PUT` | `/api/workflows/{id}` | 更新工作流 |
| `DELETE` | `/api/workflows/{id}` | 删除工作流 |
| `POST` | `/api/workflows/{id}/run` | 运行工作流 |
| `GET` | `/api/workflows/{id}/runs` | 列出运行记录 |

**v0.5.1 新增：Workflow 持久化**

```rust
// crates/openfang-api/src/routes.rs - create_workflow
pub async fn create_workflow(
    State(state): State<Arc<AppState>>,
    Json(payload): Json<serde_json::Value>,
) -> impl IntoResponse {
    // ... 创建 workflow ...
    let id = state.kernel.register_workflow(workflow.clone()).await;

    // v0.5.1 新增：持久化到磁盘，守护进程重启后恢复 (#751)
    let wf_dir = state
        .kernel
        .config
        .workflows_dir
        .clone()
        .unwrap_or_else(|| state.kernel.config.home_dir.join("workflows"));
    if let Err(e) = std::fs::create_dir_all(&wf_dir) {
        tracing::warn!("Failed to create workflows dir: {e}");
    } else {
        let wf_path = wf_dir.join(format!("{}.json", id));
        if let Ok(json) = serde_json::to_string_pretty(&workflow) {
            if let Err(e) = std::fs::write(&wf_path, json) {
                tracing::warn!("Failed to persist workflow {id}: {e}");
            }
        }
    }

    (StatusCode::CREATED, Json(serde_json::json!({"workflow_id": id.to_string()})))
}
```

**持久化逻辑**：
1. 创建工作流后，自动序列化到 JSON
2. 保存到 `workflows_dir` 配置目录（默认 `~/.openfang/workflows/`）
3. 文件命名：`{workflow_id}.json`
4. 守护进程重启时自动加载已保存的工作流

### 2.9 Schedules & Cron 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| `GET` | `/api/schedules` | 列出调度 |
| `POST` | `/api/schedules` | 创建调度 |
| `DELETE` | `/api/schedules/{id}` | 删除调度 |
| `PUT` | `/api/schedules/{id}` | 更新调度 |
| `POST` | `/api/schedules/{id}/run` | 手动运行 |
| `GET` | `/api/cron/jobs` | 列出 Cron 任务 |
| `POST` | `/api/cron/jobs` | 创建 Cron 任务 |
| `DELETE` | `/api/cron/jobs/{id}` | 删除 Cron 任务 |
| `PUT` | `/api/cron/jobs/{id}/enable` | 启用/禁用 |
| `GET` | `/api/cron/jobs/{id}/status` | 获取状态 |

### 2.10 Budget & Usage 端点

| 方法 | 路径 | 功能 | Token 成本 |
|------|------|------|-----------|
| `GET` | `/api/budget` | 预算状态 | 3 |
| `PUT` | `/api/budget` | 更新预算 | 5 |
| `GET` | `/api/budget/agents` | Agent 花费排名 | 3 |
| `GET` | `/api/budget/agents/{id}` | Agent 预算详情 | 3 |
| `PUT` | `/api/budget/agents/{id}` | 更新 Agent 预算 | 5 |
| `GET` | `/api/usage` | 使用统计 | 3 |
| `GET` | `/api/usage/summary` | 使用摘要 | 3 |
| `GET` | `/api/usage/by-model` | 按模型统计 | 3 |
| `GET` | `/api/usage/daily` | 每日使用 | 3 |

### 2.11 Providers & Models 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| `GET` | `/api/models` | 列出模型 |
| `GET` | `/api/models/aliases` | 列出别名 |
| `POST` | `/api/models/custom` | 添加自定义模型 |
| `DELETE` | `/api/models/custom/{*id}` | 移除自定义模型 |
| `GET` | `/api/models/{*id}` | 获取模型详情 |
| `GET` | `/api/providers` | 列出 Provider |
| `POST` | `/api/providers/{name}/key` | 设置 API Key |
| `DELETE` | `/api/providers/{name}/key` | 删除 API Key |
| `POST` | `/api/providers/{name}/test` | 测试连接 |
| `PUT` | `/api/providers/{name}/url` | 设置自定义 URL |
| `POST` | `/api/providers/github-copilot/oauth/start` | Copilot OAuth |
| `GET` | `/api/providers/github-copilot/oauth/poll/{poll_id}` | OAuth 轮询 |

**v0.5.1 改进：Model/Provider 配置同步**

v0.5.1 改进了 Agent 模型配置逻辑，现在可以：
1. 通过 display_name 查找模型（如 "Grok 4" → "grok-4-0709"）
2. 自动填充 provider 的 API Key 环境变量
3. 改进的 provider 认证状态同步

**Model Catalog 改进** (`crates/openfang-runtime/src/model_catalog.rs`):

```rust
// v0.5.1: 支持通过 display_name 查找模型
pub fn find_model(&self, id_or_alias: &str) -> Option<&ModelCatalogEntry> {
    let lower = id_or_alias.to_lowercase();
    // 直接 ID 匹配
    if let Some(entry) = self.models.iter().find(|m| m.id.to_lowercase() == lower) {
        return Some(entry);
    }
    // v0.5.1 新增：display_name 匹配（用于前端 UI payload）
    if let Some(entry) = self
        .models
        .iter()
        .find(|m| m.display_name.to_lowercase() == lower)
    {
        return Some(entry);
    }
    // 别名解析
    if let Some(canonical) = self.aliases.get(&lower) {
        return self.models.iter().find(|m| m.id == *canonical);
    }
    None
}
```

**Kernel set_agent_model 改进** (`crates/openfang-kernel/src/kernel.rs`):

```rust
pub async fn set_agent_model(
    &self,
    agent_id: &str,
    model: &str,
    explicit_provider: Option<&str>,
) -> KernelResult<()> {
    // v0.5.1: 从 catalog 查找模型条目
    let catalog_entry = self
        .model_catalog
        .read()
        .ok()
        .and_then(|catalog| catalog.find_model(model).cloned());

    // 解析 provider
    let provider = if let Some(ep) = explicit_provider {
        Some(ep.to_string())
    } else {
        // 从 catalog 或模型名称推断
        catalog_entry
            .as_ref()
            .map(|entry| entry.provider.clone())
            .or_else(|| infer_provider_from_model(model))
    };

    // 规范化模型名称（去掉 provider 前缀）
    let normalized_model = /* ... */;

    if let Some(provider) = provider {
        // v0.5.1: 自动填充 API Key 环境变量
        let api_key_env = Some(self.config.resolve_api_key_env(&provider));

        // 使用新的 update_model_provider_config 方法
        self.registry
            .update_model_provider_config(
                agent_id,
                normalized_model,
                provider,
                api_key_env,
                None, // base_url
            )
            .map_err(KernelError::OpenFang)?;
    }
    // ...
}
```

**Registry 新方法** (`crates/openfang-kernel/src/registry.rs`):

```rust
/// v0.5.1 新增：一次性更新 model、provider 和连接配置
pub fn update_model_provider_config(
    &self,
    id: AgentId,
    new_model: String,
    new_provider: String,
    api_key_env: Option<String>,
    base_url: Option<String>,
) -> OpenFangResult<()> {
    let mut entry = self
        .agents
        .get_mut(&id)
        .ok_or_else(|| OpenFangError::AgentNotFound(id.to_string()))?;
    entry.manifest.model.model = new_model;
    entry.manifest.model.provider = new_provider;
    entry.manifest.model.api_key_env = api_key_env;
    entry.manifest.model.base_url = base_url;
    entry.last_active = chrono::Utc::now();
    Ok(())
}
```

**测试用例** (`crates/openfang-runtime/src/model_catalog.rs`):

```rust
#[test]
fn test_find_model_by_display_name() {
    let catalog = ModelCatalog::new();
    // 通过 display_name "Grok 4" 查找 grok-4-0709
    let entry = catalog.find_model("Grok 4").unwrap();
    assert_eq!(entry.id, "grok-4-0709");
    assert_eq!(entry.provider, "xai");
}
```

**Config Schema 改进** (`crates/openfang-types/src/config.rs`):

```rust
// v0.5.1: 添加 resolve_api_key_env 方法
impl KernelConfig {
    pub fn resolve_api_key_env(&self, provider: &str) -> String {
        // 返回对应 provider 的 API Key 环境变量名
        // 如 "anthropic" → "ANTHROPIC_API_KEY"
        // 如 "groq" → "GROQ_API_KEY"
    }
}
```

### 2.12 Channels 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| `GET` | `/api/channels` | 列出渠道 |
| `POST` | `/api/channels/{name}/configure` | 配置渠道 |
| `DELETE` | `/api/channels/{name}/remove` | 移除渠道 |
| `POST` | `/api/channels/{name}/test` | 测试渠道 |
| `POST` | `/api/channels/reload` | 重载渠道 |
| `POST` | `/api/channels/whatsapp/qr/start` | WhatsApp QR |
| `GET` | `/api/channels/whatsapp/qr/status` | QR 状态 |

### 2.13 Integrations 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| `GET` | `/api/integrations` | 列出已安装 |
| `GET` | `/api/integrations/available` | 列出可用 |
| `POST` | `/api/integrations/add` | 安装集成 |
| `DELETE` | `/api/integrations/{id}` | 移除集成 |
| `POST` | `/api/integrations/{id}/reconnect` | 重连 |
| `GET` | `/api/integrations/health` | 健康检查 |
| `POST` | `/api/integrations/reload` | 重载 |

### 2.14 Network & Peers 端点

| 方法 | 路径 | 功能 | Token 成本 |
|------|------|------|-----------|
| `GET` | `/api/peers` | 列出 Peer | 2 |
| `GET` | `/api/network/status` | 网络状态 | 2 |

### 2.15 COMMS 端点 (Agent 间通信)

| 方法 | 路径 | 功能 |
|------|------|------|
| `GET` | `/api/comms/topology` | 拓扑图 |
| `GET` | `/api/comms/events` | 事件列表 |
| `GET` | `/api/comms/events/stream` | 事件流 (SSE) |
| `POST` | `/api/comms/send` | 发送消息 |
| `POST` | `/api/comms/task` | 发送任务 |

### 2.16 Audit & Security 端点

| 方法 | 路径 | 功能 | Token 成本 |
|------|------|------|-----------|
| `GET` | `/api/audit/recent` | 最近审计 | 5 |
| `GET` | `/api/audit/verify` | 审计验证 | 5 |
| `GET` | `/api/security` | 安全仪表板 | 5 |

### 2.17 Approvals 端点 (v0.5.1 增强)

| 方法 | 路径 | 功能 |
|------|------|------|
| `GET` | `/api/approvals` | 列出待处理和最近的审批请求 (**v0.5.1**: 增加历史记录) |
| `POST` | `/api/approvals/{id}/approve` | 批准请求 |
| `POST` | `/api/approvals/{id}/deny` | 拒绝请求 |

**v0.5.1 新增：审批历史记录**

之前的实现只保留待处理的审批请求，v0.5.1 新增了历史记录功能：

```rust
// crates/openfang-kernel/src/approval.rs
pub struct ApprovalManager {
    pending: DashMap<Uuid, PendingRequest>,
    recent: std::sync::Mutex<VecDeque<ApprovalRecord>>,  // v0.5.1 新增
    policy: std::sync::RwLock<ApprovalPolicy>,
}

pub struct ApprovalRecord {
    pub request: ApprovalRequest,
    pub decision: ApprovalDecision,
    pub decided_at: chrono::DateTime<Utc>,
    pub decided_by: Option<String>,
}
```

**新增 API 响应字段**：
- `status`: `"pending"`, `"approved"`, `"rejected"`, `"expired"`
- `decided_at`: 决策时间
- `decided_by`: 决策者用户名

**前端自动刷新** (approvals.js):
```javascript
function approvalsPage() {
  return {
    init() {
      this.loadData();
      // 每 5 秒自动刷新
      this.refreshTimer = setInterval(() => this.loadData(), 5000);
    },
    destroy() {
      if (this.refreshTimer) clearInterval(this.refreshTimer);
    }
  };
}
```

### 2.18 MCP 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| `GET` | `/api/mcp/servers` | 列出 MCP 服务器 |
| `POST` | `/mcp` | MCP JSON-RPC over HTTP |

### 2.19 A2A 端点 (Agent-to-Agent Protocol)

| 方法 | 路径 | 功能 |
|------|------|------|
| `GET` | `/.well-known/agent.json` | Agent Card |
| `GET` | `/a2a/agents` | 列出 Agents |
| `POST` | `/a2a/tasks/send` | 发送任务 |
| `GET` | `/a2a/tasks/{id}` | 获取任务状态 |
| `POST` | `/a2a/tasks/{id}/cancel` | 取消任务 |
| `GET` | `/api/a2a/agents` | 外部 Agents |
| `POST` | `/api/a2a/discover` | 发现外部 Agent |
| `POST` | `/api/a2a/send` | 发送到外部 |
| `GET` | `/api/a2a/tasks/{id}/status` | 外部任务状态 |

### 2.20 Auth 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| `POST` | `/api/auth/login` | 登录 |
| `POST` | `/api/auth/logout` | 登出 |
| `GET` | `/api/auth/check` | 检查认证状态 |

### 2.21 Health & Status 端点

| 方法 | 路径 | 功能 | Token 成本 |
|------|------|------|-----------|
| `GET` | `/api/health` | 健康检查 | 1 |
| `GET` | `/api/health/detail` | 详细健康 | 2 |
| `GET` | `/api/status` | 状态 | 1 |
| `GET` | `/api/version` | 版本 | 1 |
| `GET` | `/api/metrics` | Prometheus 指标 | - |
| `GET` | `/api/logs/stream` | 日志流 (SSE) | - |

### 2.22 Config 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| `GET` | `/api/config` | 获取配置 |
| `GET` | `/api/config/schema` | 配置 Schema |
| `POST` | `/api/config/set` | 设置配置 |
| `POST` | `/api/config/reload` | 重载配置 |

### 2.23 OpenAI 兼容 API

| 方法 | 路径 | 功能 |
|------|------|------|
| `POST` | `/v1/chat/completions` | Chat 补全 |
| `GET` | `/v1/models` | 列出模型 |

### 2.24 Webhook 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| `POST` | `/hooks/wake` | 唤醒 Agent |
| `POST` | `/hooks/agent` | Agent 触发 |

### 2.25 其他管理端点

| 方法 | 路径 | 功能 |
|------|------|------|
| `POST` | `/api/shutdown` | 关闭服务 |

### 2.26 前端通知功能 (v0.5.1)

v0.5.1 在前端添加了 Approvals 自动刷新和通知功能。

**App 状态** (`crates/openfang-api/static/js/app.js`):

```javascript
document.addEventListener('alpine:init', function() {
  Alpine.store('app', {
    pendingApprovalCount: 0,           // v0.5.1 新增
    lastPendingApprovalSignature: '',  // v0.5.1 新增

    // v0.5.1 新增：刷新审批列表
    async refreshApprovals() {
      try {
        var data = await OpenFangAPI.get('/api/approvals');
        var approvals = Array.isArray(data) ? data : (data.approvals || []);
        var pending = approvals.filter(function(a) { return a.status === 'pending'; });

        // 生成签名用于检测变化
        var signature = pending
          .map(function(a) { return a.id; })
          .sort()
          .join(',');

        // 检测到新的审批请求时显示通知
        if (pending.length > 0 && signature !== this.lastPendingApprovalSignature) {
          OpenFangToast.warn('An agent is waiting for approval. Open Approvals to review.');
        }
        this.pendingApprovalCount = pending.length;
        this.lastPendingApprovalSignature = signature;
      } catch(e) { /* silent */ }
    },

    async pollStatus() {
      // ... 原有的状态轮询 ...
    }
  });

  // 每 5 秒刷新状态和审批
  setInterval(function() {
    self.pollStatus();
    Alpine.store('app').refreshApprovals();  // v0.5.1 新增
  }, 5000);
});
```

**通知逻辑**：
1. 每 5 秒自动调用 `/api/approvals` 获取最新审批列表
2. 通过比对 `pendingApprovalCount` 和 `lastPendingApprovalSignature` 检测新请求
3. 当检测到新的待处理审批时，显示 Toast 警告通知用户
4. 侧边栏显示待处理审批数量（待实现）

### 2.27 Dotenv 缓存清理 (v0.5.1)

当用户通过 Dashboard 删除 API Key 时，v0.5.1 会同步清理内存中的 dotenv 缓存，避免返回过期值。

**Credentials 清理方法** (`crates/openfang-extensions/src/credentials.rs`):

```rust
impl CredentialResolver {
    // ... 其他方法 ...

    /// v0.5.1 新增：清除 in-memory dotenv 缓存中的凭证
    /// 当通过 Dashboard 删除 API Key 时调用，防止 resolver 返回
    /// 启动时快照的陈旧值 (#736)
    pub fn clear_dotenv_cache(&mut self, key: &str) {
        self.dotenv.remove(key);
    }

    pub fn remove_from_vault(&mut self, key: &str) -> ExtensionResult<bool> {
        if let Some(ref mut vault) = self.vault {
            // ... 从 vault 删除 ...
        }
    }
}
```

**Kernel 集成** (`crates/openfang-kernel/src/kernel.rs`):

```rust
pub async fn delete_provider_key(&self, provider: &str) -> KernelResult<()> {
    let key = self.config.resolve_api_key_env(provider);

    // 从 vault 删除
    if let Err(e) = resolver.remove_from_vault(&key) {
        debug!("Vault remove skipped for {key}: {e}");
    }

    // v0.5.1 新增：同步清理 dotenv 缓存
    resolver.clear_dotenv_cache(&key);

    info!(provider = %provider, "Provider API key deleted");
    Ok(())
}
```

**问题背景 (#736)**:
- 启动时，CredentialResolver 从环境加载 dotenv 快照到内存
- 用户通过 Dashboard 删除 API Key 后，vault 更新了但内存缓存未同步
- 导致后续请求仍返回删除前的陈旧值
- v0.5.1 通过 `clear_dotenv_cache()` 方法解决此问题

---

## 3. v0.4.9 新增端点详解

### 3.1 POST /api/agents/{id}/restart — 重启 Agent

**场景**: 当 Agent 崩溃、卡住或无响应时，无需手动停止再启动。

```rust
// crates/openfang-api/src/routes.rs
pub async fn restart_agent(
    State(state): State<Arc<AppState>>,
    Path(id): Path<String>,
) -> impl IntoResponse {
    let agent_id = &id;

    // 1. 取消运行中的任务
    let was_running = state.kernel.stop_agent_run(agent_id).unwrap_or(false);

    // 2. 重置状态为 Running
    let _ = state.kernel.registry.set_state(
        agent_id,
        openfang_types::agent::AgentState::Running
    );

    // 3. 返回成功响应
    Json(json!({
        "success": true,
        "restarted": true,
        "was_running": was_running
    }))
}
```

**使用示例**:
```bash
# 重启卡住的 Agent
curl -X POST http://127.0.0.1:4200/api/agents/agent-1/restart

# 返回
{"success":true,"restarted":true,"was_running":false}
```

**前端调用** (agents.js):
```javascript
async restartAgent() {
  try {
    await OpenFangAPI.post(`/api/agents/${this.agent.id}/restart`);
    OpenFangToast.success('Agent restarted');
    this.loadAgents(); // 刷新列表
  } catch (e) {
    OpenFangToast.error('Failed to restart: ' + e.message);
  }
}
```

---

### 3.2 POST /api/hands/upsert — 热更新 Hands 配置

**场景**: 无需重启服务，动态创建或更新 Hands 配置。

```rust
// crates/openfang-api/src/routes.rs
pub async fn upsert_hand(
    State(state): State<Arc<AppState>>,
    Json(payload): Json<serde_json::Value>,
) -> impl IntoResponse {
    let hand_id = payload["hand_id"].as_str().ok_or("Missing hand_id")?;
    let config = payload["config"].as_object().ok_or("Missing config")?;

    // 调用 kernel 的 upsert_hand 方法
    match state.kernel.upsert_hand(hand_id, config).await {
        Ok(created) => Json(json!({
            "success": true,
            "created": created,
            "hand_id": hand_id
        })),
        Err(e) => (StatusCode::BAD_REQUEST, Json(json!({
            "error": e.to_string()
        }))),
    }
}
```

**使用示例**:
```bash
# 创建/更新 Hand
curl -X POST http://127.0.0.1:4200/api/hands/upsert \
  -H "Content-Type: application/json" \
  -d '{
    "hand_id": "researcher",
    "config": {
      "enabled": true,
      "auto_start": true,
      "max_instances": 5
    }
  }'
```

---

### 3.3 GET /api/config/schema — 获取配置 Schema

**场景**: 前端动态生成配置表单，验证用户输入。

```rust
// crates/openfang-api/src/routes.rs
pub async fn get_config_schema(
    State(_state): State<Arc<AppState>>,
) -> impl IntoResponse {
    // 返回 TOML 格式的 schema 定义
    let schema = r#"
[default_model]
provider = "anthropic|openai|groq|..."
model = "model-name"
api_key_env = "API_KEY_ENV_VAR"

[memory]
decay_rate = 0.05

[network]
listen_addr = "127.0.0.1:4200"

[wecom]
corp_id = "your_corp_id"
agent_id = "1000001"
secret = "your_secret"
token = "webhook_token"
encoding_aes_key = "your_aes_key"
"#;

    (StatusCode::OK, schema)
}
```

**使用示例**:
```bash
curl -s http://127.0.0.1:4200/api/config/schema
# 返回 TOML schema 供前端解析
```

---

### 3.4 GET /api/comms/events/stream — SSE 事件流

**场景**: 前端实时接收 Agent 通信事件，更新 UI。

```rust
// crates/openfang-api/src/routes.rs
pub async fn comms_events_stream(
    State(state): State<Arc<AppState>>,
) -> impl IntoResponse {
    use axum::response::sse::{Sse, Event};
    use futures::stream::Stream;

    // 订阅事件总线
    let (tx, rx) = tokio::sync::mpsc::channel(100);
    let _subscription = state.kernel.comms_bus.subscribe(move |event| {
        let _ = tx.try_send(event);
    });

    // 流式返回事件
    let stream = ReceiverStream::new(rx).map(|event| {
        Ok::<Event, Infallible>(
            Event::default().data(serde_json::to_string(&event).unwrap())
        )
    });

    Sse::new(stream)
}
```

**使用示例**:
```bash
# 监听事件流
curl -N http://127.0.0.1:4200/api/comms/events/stream

# 输出
data: {"type":"agent_message","agent_id":"agent-1","content":"Hello"}
data: {"type":"task_completed","task_id":"task-123","result":"success"}
```

---

### 3.5 PWA 支持 — manifest.json 和 sw.js

**文件位置**:
- `crates/openfang-api/static/manifest.json`
- `crates/openfang-api/static/sw.js`

**manifest.json**:
```json
{
  "name": "OpenFang Dashboard",
  "short_name": "OpenFang",
  "description": "Agent Operating System Dashboard",
  "start_url": "/",
  "display": "standalone",
  "theme_color": "#10b981",
  "background_color": "#1f2937",
  "icons": [
    {
      "src": "/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

**sw.js** (Service Worker):
```javascript
// Service Worker 缓存关键资源
const CACHE_NAME = 'openfang-v1';
const ASSETS = [
  '/',
  '/index.html',
  '/js/i18n.js',
  '/js/app.js',
  '/css/styles.css'
];

self.addEventListener('install', (e) => {
  e.waitUntil(caches.open(CACHE_NAME).then(c => c.addAll(ASSETS)));
});

self.addEventListener('fetch', (e) => {
  e.respondWith(
    caches.match(e.request).then(r => r || fetch(e.request))
  );
});
```

**HTML 引用** (index_body.html):
```html
<link rel="manifest" href="/manifest.json">
<script>
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js');
  }
</script>
```

---

### 3.6 图片处理流水线

**场景**: 用户上传图片，Agent 识别并回复。

```rust
// crates/openfang-api/src/routes.rs - 图片上传处理
pub async fn upload_file(
    State(state): State<Arc<AppState>>,
    Path(agent_id): Path<String>,
    mut multipart: Multipart,
) -> Result<Json<Value>, ApiError> {
    const UPLOADS_DIR: &str = "/tmp/openfang_uploads";

    while let Some(field) = multipart.next_field().await? {
        let data = field.bytes().await?;

        // 保存为 base64
        let base64_data = base64_encode(&data);
        let mime_type = field.content_type().unwrap_or("image/png");

        // 创建 ContentBlock::Image
        let image_block = ContentBlock::Image {
            data: data.to_vec(),
            mime_type: mime_type.to_string(),
            base64_inline: true,
        };

        // 发送到 Agent
        let response = state.kernel
            .send_message(&agent_id, vec![image_block])
            .await?;
    }

    Ok(Json(json!({ "success": true })))
}
```

**前端调用**:
```javascript
async uploadImage(file) {
  const formData = new FormData();
  formData.append('file', file);

  const resp = await fetch(`/api/agents/${agentId}/upload`, {
    method: 'POST',
    body: formData
  });

  // 图片直接显示在消息气泡中
  const imageUrl = URL.createObjectURL(file);
  this.messages.push({ role: 'user', content: imageUrl, type: 'image' });
}
```

---
| `GET` | `/api/commands` | 列出命令 |
| `GET` | `/api/templates` | 列出模板 |
| `GET` | `/api/templates/{name}` | 获取模板 |
| `GET` | `/api/triggers` | 列出触发器 |
| `POST` | `/api/triggers` | 创建触发器 |
| `DELETE` | `/api/triggers/{id}` | 删除触发器 |
| `PUT` | `/api/triggers/{id}` | 更新触发器 |
| `POST` | `/api/migrate/detect` | 检测迁移 |
| `POST` | `/api/migrate/scan` | 扫描迁移 |
| `POST` | `/api/migrate` | 执行迁移 |
| `GET` | `/api/pairing/devices` | 配对设备 |
| `POST` | `/api/pairing/request` | 请求配对 |
| `POST` | `/api/pairing/complete` | 完成配对 |
| `DELETE` | `/api/pairing/devices/{id}` | 移除设备 |
| `POST` | `/api/pairing/notify` | 配对通知 |

---

*创建时间：2026-03-15 (更新于 2026-03-20 v0.5.1)*
*OpenFang v0.5.2*

## 3. GCRA 速率限制

### 3.1 算法原理

GCRA (Generic Cell Rate Algorithm) 是一种平滑的速率限制算法：

```
令牌桶容量：500 tokens/分钟
每 IP 独立计数
每操作有 token 成本
```

### 3.2 操作成本表

```rust
// crates/openfang-api/src/rate_limiter.rs:14-35
pub fn operation_cost(method: &str, path: &str) -> NonZeroU32 {
    match (method, path) {
        (_, "/api/health") => NonZeroU32::new(1).unwrap(),
        ("GET", "/api/status") => NonZeroU32::new(1).unwrap(),
        ("GET", "/api/version") => NonZeroU32::new(1).unwrap(),
        ("GET", "/api/tools") => NonZeroU32::new(1).unwrap(),
        ("GET", "/api/agents") => NonZeroU32::new(2).unwrap(),
        ("GET", "/api/skills") => NonZeroU32::new(2).unwrap(),
        ("GET", "/api/peers") => NonZeroU32::new(2).unwrap(),
        ("GET", "/api/config") => NonZeroU32::new(2).unwrap(),
        ("GET", "/api/usage") => NonZeroU32::new(3).unwrap(),
        ("GET", p) if p.starts_with("/api/audit") => NonZeroU32::new(5).unwrap(),
        ("GET", p) if p.starts_with("/api/marketplace") => NonZeroU32::new(10).unwrap(),
        ("POST", "/api/agents") => NonZeroU32::new(50).unwrap(),
        ("POST", p) if p.contains("/message") => NonZeroU32::new(30).unwrap(),
        ("POST", p) if p.contains("/run") => NonZeroU32::new(100).unwrap(),
        ("POST", "/api/skills/install") => NonZeroU32::new(50).unwrap(),
        ("POST", "/api/skills/uninstall") => NonZeroU32::new(10).unwrap(),
        ("POST", "/api/migrate") => NonZeroU32::new(100).unwrap(),
        ("PUT", p) if p.contains("/update") => NonZeroU32::new(10).unwrap(),
        _ => NonZeroU32::new(5).unwrap(),
    }
}
```

### 3.3 成本分级

| 成本 | 操作类型 | 示例 |
|------|----------|------|
| **1** | 轻量查询 | health, status, version, tools |
| **2-3** | 数据查询 | agents, skills, peers, usage |
| **5-10** | 审计/市场 | audit, marketplace, update |
| **30** | 消息发送 | /message |
| **50** | 资源创建 | spawn agent, install skill |
| **100** | 重操作 | workflow run, migrate |

### 3.4 速率限制中间件

```rust
// crates/openfang-api/src/rate_limiter.rs:52-79
pub async fn gcra_rate_limit(
    axum::extract::State(limiter): axum::extract::State<Arc<KeyedRateLimiter>>,
    request: Request<Body>,
    next: Next,
) -> Response<Body> {
    // 提取客户端 IP
    let ip = request
        .extensions()
        .get::<axum::extract::ConnectInfo<SocketAddr>>()
        .map(|ci| ci.0.ip())
        .unwrap_or(IpAddr::from([127, 0, 0, 1]));

    // 计算操作成本
    let method = request.method().as_str().to_string();
    let path = request.uri().path().to_string();
    let cost = operation_cost(&method, &path);

    // 检查速率限制
    if limiter.check_key_n(&ip, cost).is_err() {
        tracing::warn!(ip = %ip, cost = cost.get(), path = %path, "GCRA rate limit exceeded");
        return Response::builder()
            .status(StatusCode::TOO_MANY_REQUESTS)
            .header("content-type", "application/json")
            .header("retry-after", "60")
            .body(Body::from(
                serde_json::json!({"error": "Rate limit exceeded"}).to_string(),
            ))
            .unwrap_or_default();
    }

    next.run(request).await
}
```

---

## 4. 认证机制

### 4.1 认证配置

```rust
// crates/openfang-api/src/server.rs:106-118
let api_key = state.kernel.config.api_key.trim().to_string();
let auth_state = crate::middleware::AuthState {
    api_key: api_key.clone(),
    auth_enabled: state.kernel.config.auth.enabled,
    session_secret: if !api_key.is_empty() {
        api_key.clone()
    } else if state.kernel.config.auth.enabled {
        state.kernel.config.auth.password_hash.clone()
    } else {
        String::new()
    },
};
```

### 4.2 认证中间件

```rust
// 认证逻辑简化版
pub async fn auth(
    State(auth_state): State<AuthState>,
    request: Request<Body>,
    next: Next,
) -> Response {
    // 1. 如果未启用认证，直接通过
    if !auth_state.auth_enabled {
        return next.run(request).await;
    }

    // 2. 检查 Authorization 头
    let auth_header = request.headers().get("Authorization")
        .and_then(|h| h.to_str().ok());

    // 3. 验证 Bearer Token
    if let Some(header) = auth_header {
        if header.starts_with("Bearer ") {
            let token = &header[7..];
            if token == auth_state.api_key {
                return next.run(request).await;
            }
        }
    }

    // 4. 返回 401
    Response::builder()
        .status(StatusCode::UNAUTHORIZED)
        .body(Body::from("Unauthorized"))
        .unwrap()
}
```

### 4.3 免认证路径

以下路径始终公开：
- `/.well-known/agent.json` (A2A Agent Card)
- `/api/health` (健康检查)
- `/api/version` (版本信息)

---

## 5. SSE 流式端点

### 5.1 日志流

```rust
// crates/openfang-api/src/server.rs:415-416
.route("/api/logs/stream", axum::routing::get(routes::logs_stream))
```

**使用示例**：
```bash
curl -N http://127.0.0.1:4200/api/logs/stream
```

**响应格式**：
```
data: {"timestamp": "2026-03-15T10:00:00Z", "level": "INFO", "message": "..."}

data: {"timestamp": "2026-03-15T10:00:01Z", "level": "DEBUG", "message": "..."}
```

### 5.2 COMMS 事件流

```rust
// crates/openfang-api/src/server.rs:433-435
.route(
    "/api/comms/events/stream",
    axum::routing::get(routes::comms_events_stream),
)
```

**使用示例**：
```bash
curl -N http://127.0.0.1:4200/api/comms/events/stream
```

### 5.3 消息流式响应

```rust
// crates/openfang-api/src/server.rs:153-156
.route(
    "/api/agents/{id}/message/stream",
    axum::routing::post(routes::send_message_stream),
)
```

**使用示例**：
```bash
curl -X POST http://127.0.0.1:4200/api/agents/agent-1/message/stream \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello"}' \
  -N
```

---

## 6. WebSocket 端点

### 6.1 Agent WebSocket

```rust
// crates/openfang-api/src/server.rs:229
.route("/api/agents/{id}/ws", axum::routing::get(ws::agent_ws))
```

**连接示例**：
```javascript
const ws = new WebSocket("ws://127.0.0.1:4200/api/agents/agent-1/ws");

ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    console.log("Received:", data);
};

ws.send(JSON.stringify({
    type: "message",
    content: "Hello Agent"
}));
```

**消息类型**：

| 类型 | 方向 | 内容 |
|------|------|------|
| `message` | C→S | 发送消息 |
| `response` | S→C | 响应内容 |
| `status` | S→C | 状态更新 |
| `error` | S→C | 错误通知 |
| `tool_call` | S→C | 工具调用通知 |
| `tool_result` | S→C | 工具结果 |

---

## 7. OpenAI 兼容 API

### 7.1 Chat Completions

```rust
// crates/openfang-api/src/server.rs:689-692
.route(
    "/v1/chat/completions",
    axum::routing::post(crate::openai_compat::chat_completions),
)
```

**请求示例**：
```bash
curl -X POST http://127.0.0.1:4200/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <api_key>" \
  -d '{
    "model": "agent-1",
    "messages": [
      {"role": "user", "content": "Hello"}
    ],
    "stream": false
  }'
```

**响应示例**：
```json
{
  "id": "chatcmpl-123",
  "object": "chat.completion",
  "created": 1677652288,
  "model": "agent-1",
  "choices": [{
    "index": 0,
    "message": {
      "role": "assistant",
      "content": "Hello! How can I help you?"
    },
    "finish_reason": "stop"
  }],
  "usage": {
    "prompt_tokens": 9,
    "completion_tokens": 12,
    "total_tokens": 21
  }
}
```

### 7.2 Models 端点

```rust
// crates/openfang-api/src/server.rs:693-696
.route(
    "/v1/models",
    axum::routing::get(crate::openai_compat::list_models),
)
```

**响应示例**：
```json
{
  "object": "list",
  "data": [
    {
      "id": "agent-1",
      "object": "model",
      "created": 1677652288,
      "owned_by": "openfang"
    },
    {
      "id": "agent-2",
      "object": "model",
      "created": 1677652289,
      "owned_by": "openfang"
    }
  ]
}
```

---

## 8. A2A 协议端点

### 8.1 Agent Card

```rust
// crates/openfang-api/src/server.rs:605-608
.route(
    "/.well-known/agent.json",
    axum::routing::get(routes::a2a_agent_card),
)
```

**响应示例**：
```json
{
  "name": "code-reviewer",
  "description": "Reviews code for bugs and security issues",
  "url": "http://127.0.0.1:4200/a2a",
  "version": "0.1.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false,
    "stateTransitionHistory": true
  },
  "skills": [
    {
      "id": "file_read",
      "name": "file read",
      "description": "Can use the file_read tool",
      "tags": ["tool"],
      "examples": []
    }
  ],
  "defaultInputModes": ["text"],
  "defaultOutputModes": ["text"]
}
```

### 8.2 任务提交

```rust
// crates/openfang-api/src/server.rs:610-613
.route(
    "/a2a/tasks/send",
    axum::routing::post(routes::a2a_send_task),
)
```

**请求示例**：
```bash
curl -X POST http://127.0.0.1:4200/a2a/tasks/send \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tasks/send",
    "params": {
      "message": {
        "role": "user",
        "parts": [{
          "type": "text",
          "text": "Review this code for security issues"
        }]
      },
      "sessionId": "session-123"
    }
  }'
```

---

## 9. Dashboard 集成

### 9.1 WebChat 页面

```rust
// crates/openfang-api/src/server.rs:122-124
.route("/", axum::routing::get(webchat::webchat_page))
.route("/logo.png", axum::routing::get(webchat::logo_png))
.route("/favicon.ico", axum::routing::get(webchat::favicon_ico))
```

**访问**：`http://127.0.0.1:4200/`

### 9.2 静态资源

Dashboard 的静态资源通过内嵌方式提供：

| 路径 | 内容 |
|------|------|
| `/` | WebChat HTML 页面 |
| `/logo.png` | OpenFang Logo |
| `/favicon.ico` | 网站图标 |

### 9.3 KaTeX 数学公式渲染 (v0.5.1 新增)

v0.5.1 新增了 KaTeX 支持，可在聊天消息中渲染 LaTeX 数学公式。

**新增文件**：`crates/openfang-api/static/js/katex.js`

```javascript
// KaTeX 按需加载器和渲染器
var KATEX_VERSION = '0.16.21';
var KATEX_CSS_URL = 'https://cdn.jsdelivr.net/npm/katex@' + KATEX_VERSION + '/dist/katex.min.css';
var KATEX_JS_URL = 'https://cdn.jsdelivr.net/npm/katex@' + KATEX_VERSION + '/dist/katex.min.js';
var KATEX_AUTORENDER_URL =
  'https://cdn.jsdelivr.net/npm/katex@' + KATEX_VERSION + '/dist/contrib/auto-render.min.js';

// 支持的定界符
delimiters: [
  { left: '$$', right: '$$', display: true },   // 块级公式
  { left: '\\[', right: '\\]', display: true }, // 块级公式
  { left: '$', right: '$', display: false },    // 行内公式
  { left: '\\(', right: '\\)', display: false } // 行内公式
]
```

**使用示例**：
```latex
# 行内公式
$E = mc^2$

# 块级公式
$$\int_{-\infty}^{\infty} e^{-x^2} dx = \sqrt{\pi}$$

# 矩阵
$$\begin{pmatrix} a & b \\ c & d \end{pmatrix}$$
```

**集成到 WebChat** (webchat.rs):
```rust
const WEBCHAT_HTML: &str = concat!(
    include_str!("../static/js/pages/overview.js"),
    "\n",
    include_str!("../static/js/katex.js"),  // v0.5.1 新增
    "\n",
    include_str!("../static/js/pages/chat.js"),
    // ...
);
```

**前端调用** (app.js):
```javascript
// 新消息到达后自动渲染 LaTeX
function renderMarkdown(text) {
  var html = marked.parse(text);
  // 检测并渲染 LaTeX
  if (hasLatexDelimiters(text)) {
    renderLatex(messageElement);
  }
  return html;
}
```

---

## 10. 安全特性

### 10.1 安全头

```rust
// middleware::security_headers
pub async fn security_headers(
    request: Request<Body>,
    next: Next,
) -> Response {
    let mut response = next.run(request).await;
    let headers = response.headers_mut();

    // XSS 防护
    headers.insert("X-Content-Type-Options", "nosniff".parse().unwrap());
    headers.insert("X-Frame-Options", "DENY".parse().unwrap());
    headers.insert("X-XSS-Protection", "1; mode=block".parse().unwrap());

    // CSP
    headers.insert(
        "Content-Security-Policy",
        "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'"
            .parse().unwrap()
    );

    response
}
```

### 10.2 CORS 配置

```rust
// crates/openfang-api/src/server.rs:56-104
let cors = if state.kernel.config.api_key.trim().is_empty() {
    // 无认证 → 限制为 localhost
    CorsLayer::new()
        .allow_origin(vec![
            format!("http://{listen_addr}").parse().unwrap(),
            format!("http://localhost:{port}").parse().unwrap(),
            "http://127.0.0.1:3000".parse().unwrap(),
            "http://localhost:8080".parse().unwrap(),
        ])
        .allow_methods(tower_http::cors::Any)
        .allow_headers(tower_http::cors::Any)
} else {
    // 有认证 → 更严格的 CORS
    CorsLayer::new()
        .allow_origin(vec![
            format!("http://{listen_addr}").parse().unwrap(),
            "http://localhost:4200".parse().unwrap(),
        ])
        .allow_methods(tower_http::cors::Any)
        .allow_headers(tower_http::cors::Any)
};
```

### 10.3 请求日志

```rust
// middleware::request_logging
pub async fn request_logging(
    request: Request<Body>,
    next: Next,
) -> Response {
    let method = request.method().clone();
    let uri = request.uri().clone();
    let start = Instant::now();

    let response = next.run(request).await;

    let duration = start.elapsed();
    let status = response.status();

    tracing::info!(
        method = %method,
        uri = %uri,
        status = %status,
        duration_ms = %duration.as_millis(),
        "HTTP request"
    );

    response
}
```

---

## 11. 测试代码

### 11.1 速率限制测试

```rust
// crates/openfang-api/src/rate_limiter.rs:82-98
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_costs() {
        assert_eq!(operation_cost("GET", "/api/health").get(), 1);
        assert_eq!(operation_cost("GET", "/api/tools").get(), 1);
        assert_eq!(operation_cost("POST", "/api/agents/1/message").get(), 30);
        assert_eq!(operation_cost("POST", "/api/agents").get(), 50);
        assert_eq!(operation_cost("POST", "/api/workflows/1/run").get(), 100);
        assert_eq!(operation_cost("GET", "/api/agents/1/session").get(), 5);
        assert_eq!(operation_cost("GET", "/api/skills").get(), 2);
        assert_eq!(operation_cost("GET", "/api/peers").get(), 2);
    }
}
```

---

## 12. 关键设计点

### 12.1 分层路由结构

```
Router (720+ lines)
├── Chunk 1: Agents, Memory, Sessions (lines 121-300)
├── Chunk 2: Workflows, Skills, Hands (lines 300-400)
├── Chunk 3: MCP, Network, COMMS (lines 400-444)
└── Chunk 4: Tools, Config, Budget, Auth (lines 447-723)
```

**设计原因**：Axum 类型嵌套限制，需要分两个 `let app = app...` 块构建。

### 12.2 AppState 设计

```rust
// crates/openfang-api/src/routes.rs
pub struct AppState {
    pub kernel: Arc<OpenFangKernel>,
    pub started_at: Instant,
    pub peer_registry: Option<Arc<PeerRegistry>>,
    pub bridge_manager: tokio::sync::Mutex<BridgeManager>,
    pub channels_config: tokio::sync::RwLock<ChannelsConfig>,
    pub shutdown_notify: Arc<tokio::sync::Notify>,
    pub clawhub_cache: DashMap<String, ClawHubEntry>,
    pub provider_probe_cache: ProbeCache,
}
```

### 12.3 成本感知速率限制

不同于传统固定速率限制，OpenFang 根据操作成本动态计费：

```
健康检查 (1 token)    ████████░░░░░░░░░░░░░░░░░░░░  2%
Agent 列表 (2 tokens)  ████████████░░░░░░░░░░░░░░░░  4%
发送消息 (30 tokens)   ██████████████████████████████  6%
创建 Agent (50 tokens) ████████████████████████████████████  10%
运行 Workflow (100)    ████████████████████████████████████████████████████████  20%
```

---

## 完成检查清单

- [ ] 理解 API 服务器架构和中间件栈
- [ ] 掌握 140+ REST 端点的分类和功能
- [ ] 理解 OpenAI 兼容 API 的实现
- [ ] 掌握 SSE 流式和 WebSocket 实时通信
- [ ] 理解 GCRA 速率限制和认证机制
- [ ] 了解 v0.5.1 新增功能：Workflow 持久化、审批历史记录、KaTeX 渲染、Dotenv 缓存清理

---

## 下一步

前往 [第 25 节：CLI 与 Desktop 应用](./25-cli-desktop.md)

---

*创建时间：2026-03-15 (更新于 2026-03-20 v0.5.1)*
*OpenFang v0.5.2*
