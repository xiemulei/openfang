# OpenFang 学习笔记 — 25 节完整教程

> **版本**: v0.5.5 (2026-03-31)
> **状态**: ✅ 全部完成
> **总计**: 14 Crates, 145K+ LOC, 1767+ Tests, 25 节教程

---

## 学习路线概览

```
第一阶段：基础篇 (第 1-4 节)
├── 环境搭建与项目概览
├── 架构设计与 Crates 划分
├── 核心类型系统
└── 启动流程分析

第二阶段：运行时核心 (第 5-10 节)
├── Agent 循环详解 (拆分为 3 节)
├── LLM Driver 系统 (拆分为 2 节)
├── 工具执行系统 (拆分为 2 节)
└── 记忆与存储系统 (拆分为 2 节)

第三阶段：自主代理 (第 11-15 节)
├── Hands 系统 (拆分为 4 节)

第四阶段：安全与扩展 (第 20-23 节)
├── 安全系统 (拆分为 2 节)
├── Skills 系统 (拆分为 1 节)
└── Extensions 系统 (拆分为 1 节)

第五阶段：接口与应用 (第 24-25 节)
├── API 服务
└── CLI 与 Desktop
```

---

## 详细章节规划

### 第一阶段：基础篇

| 节 | 标题 | 核心内容 | 状态 |
|----|------|----------|------|
| 1 | [环境搭建与项目概览](./01-environment-setup.md) | 环境验证、代码结构、启动流程 | ✅ 已完成 |
| 2 | [14 Crates 架构解析](./02-architecture-analysis.md) | Crate 职责、依赖关系、KernelHandle 解耦 | ✅ 已完成 |
| 3 | [核心类型系统](./03-type-system.md) | AgentId/Message/Tool、污点追踪、Ed25519 签名 | ✅ 已完成 |
| 4 | [启动流程分析](./04-startup-flow.md) | CLI 入口、Kernel boot、子系统初始化 | ✅ 已完成 |

---

### 第二阶段：运行时核心

| 节 | 标题 | 核心内容 | 状态 |
|----|------|----------|------|
| 5 | [Agent 循环 — 主流程](./05-agent-loop-main.md) | run_agent_loop 主循环、迭代控制、状态流转 | ✅ 已完成 |
| 6 | [Agent 循环 — 上下文管理](./06-agent-loop-context.md) | 上下文溢出恢复、Session Repair、消息修剪 | ✅ 已完成 |
| 7 | [Agent 循环 — 错误处理](./07-agent-loop-errors.md) | 重试策略、断路器、LoopGuard | ✅ 已完成 |
| 8 | [LLM Driver — 抽象层](./08-llm-driver-abstract.md) | LlmDriver trait、CompletionRequest/Response、流式事件 | ✅ 已完成 |
| 9 | [LLM Driver — 实现](./09-llm-driver-implementations.md) | 27 个 Provider 实现、Fallback 机制、健康检查 | ✅ 已完成 |
| 10 | [工具执行 — 核心流程](./10-tool-execution-core.md) | execute_tool、工具注册、MCP 集成 | ✅ 已完成 |
| 11 | [工具执行 — 安全系统](./11-tool-execution-security.md) | Taint Tracking、shell 注入防护、RBAC 检查 | ✅ 已完成 |
| 12 | [记忆系统 — 三层存储](./12-memory-substrate.md) | Structured/Semantic/Knowledge Graph | ✅ 已完成 |
| 13 | [记忆系统 — 向量搜索](./13-memory-vector-search.md) | Embedding、向量相似度、Qdrant 集成 | ✅ 已完成 |

---

### 第三阶段：自主代理

| 节 | 标题 | 核心内容 | 状态 |
|----|------|----------|------|
| 14 | [Hands 系统 — 配置与激活](./14-hands-config.md) | HAND.toml、Requirements、Settings | ✅ 已完成 |
| 15 | [Hands 系统 — 生命周期管理](./15-hands-lifecycle.md) | HandRegistry、状态持久化、Dashboard Metrics | ✅ 已完成 |
| 16 | [Channel 系统 — 消息渠道](./16-channels-bridge.md) | 40 个渠道适配器、RBAC、Rate Limiter | ✅ 已完成 |
| 17 | [Channel 系统 — 事件总线](./17-channels-event-bus.md) | 事件订阅、发布/订阅模式、Webhook | ✅ 已完成 |
| 18 | [OFP 协议 — P2P 通信](./18-ofp-protocol.md) | HMAC-SHA256 双向认证、Peer Registry | ✅ 已完成 |
| 19 | [A2A 协议 — Agent 间通信](./19-a2a-protocol.md) | Agent 发现、任务路由、结果追踪 | ✅ 已完成 |

---

### 第四阶段：安全与扩展

| 节 | 标题 | 核心内容 | 状态 |
|----|------|----------|------|
| 20 | [安全系统 — 污点追踪](./20-security-taint-tracking.md) | TaintLabel、TaintSink、信息流控制 | ✅ 已完成 |
| 21 | [安全系统 — 沙箱隔离](./21-security-sandbox.md) | WASM 沙箱、Docker 沙箱、子进程隔离 | ✅ 已完成 |
| 22 | [Skills 系统 — 技能市场](./22-skills-system.md) | 60 个内置技能、SKILL.md、FangHub | ✅ 已完成 |
| 23 | [Extensions 系统 — MCP 集成](./23-extensions-mcp.md) | 25 个 MCP 模板、凭证管理、OAuth2 PKCE | ✅ 已完成 |

---

### 第五阶段：接口与应用

| 节 | 标题 | 核心内容 | 状态 |
|----|------|----------|------|
| 24 | [API 服务 — REST/WS 端点](./24-api-server.md) | 140+ 端点、OpenAI 兼容 API、SSE 流式 | ✅ 已完成 |
| 25 | [CLI 与 Desktop 应用](./25-cli-desktop.md) | CLI 命令、守护进程管理、Tauri Dashboard | ✅ 已完成 |

---

## 文档更新记录

| 日期 | 变更 | 涉及文档 |
|------|------|----------|
| 2026-03-15 | 初始 10 节计划 | 全部 |
| 2026-03-15 | 扩展为 20 节详细计划 | 本文件 |
| 2026-03-16 | 添加 Hands 系统第 14-15 节、更新版本到 v0.4.4 | 14-hands-config.md, 15-hands-lifecycle.md |
| 2026-03-19 | 更新到 v0.4.9: 新增企业微信渠道、图片生成流水线、Agent 重启等 | 本文件、01/16/24 节 |
| 2026-03-29 | 更新到 v0.5.2: 合并 main 分支，大幅更新教程内容 | 全部教程 |
| 2026-03-31 | 更新到 v0.5.5: SearXNG 搜索、嵌套 XML 修复、Agent 技能重载 | 本文件、10/22/24 节 |

---

## v0.5.2 → v0.5.5 版本变更摘要

### 核心架构变更

#### 工具执行与 Agent Loop
- **嵌套 XML 工具调用恢复** (`agent_loop.rs`, +208 行): 修复 LLM 响应中嵌套 XML 格式的工具调用参数解析问题
- **测试用例** (`agent_loop.rs`, +140 行): 添加 `test_nested_xml_text_tool_call_recovery_e2e` 端到端测试

#### 搜索系统
- **SearXNG Search Provider** (`web_search.rs`, +168 行): 隐私尊重型元搜索引擎，支持 30+ 搜索类别
- **SearXNG 搜索技能** (`bundled/searxng/SKILL.md`, 70 行新增): 内置隐私搜索专家技能
- **分页与类别支持** (`web_search.rs`): SearXNG 分页、动态类别验证、噪音字段过滤
- **JSON 输出格式**: 仅向 LLM 暴露 title/url/content/published_date

#### Agent 系统
- **Agent Skills 热重载** (`kernel.rs`, +5 行): 检测技能配置变更时自动重载
- **测试覆盖** (`integration_test.rs`, +36 行): agent skills/mcp_servers TOML 解析测试

#### 安全与配置
- **SSRF Allowlist** (`config.rs`, +10 行): `ssrf_allowed_hosts` 配置，支持自托管 K8s 环境
- **Ollama 上下文提升**: 默认 128K 上下文 / 16K 输出
- **Embedding 自动检测扩展**: OpenAI, Groq, Mistral, Together, Fireworks, Cohere, 本地 providers

### 新增依赖
- 无新增外部依赖（SearXNG 使用现有 HTTP 客户端）

### 统计
- **5 个文件变更**: +275/-4 行 (SearXNG 相关)
- **3 个文件变更**: +41/-1 行 (Agent skills reload)
- **213 行变更**: 嵌套 XML 工具调用恢复

---

## v0.5.2 版本变更摘要 (main 分支合并)

### 核心架构变更

#### LLM Driver
- **Vertex AI 驱动** (`vertex.rs`, 794 行新增): GCP 企业版 OAuth 认证，支持所有 Gemini 模型
- **Anthropic 驱动增强**: 新增 `ensure_object()` 防止双重序列化问题
- **Gemini 驱动增强**: +439 行改进
- **Claude Code 驱动增强**: +125 行改进

#### MCP 系统 (重大重构)
- **迁移到 rmcp SDK v1.2**: 替代手写 JSON-RPC 实现，代码从 1000+ 行精简到 539 行
- **新增 Streamable HTTP 传输**: 支持 MCP 2025-03-26 协议版本
- **SSRF 防护增强**: 显式检查云元数据端点
- **自定义 HTTP 头**: 支持 Bearer 认证

#### 渠道系统
- **MQTT 适配器** (`mqtt.rs`, 604 行新增): Pub/Sub 模式，支持 QoS 0/1/2
- **飞书增强** (`feishu.rs`, +1082 行): WebSocket 接收模式

#### Kernel 系统
- **Heartbeat Monitor** (`heartbeat.rs`, 188 行新增): Agent 健康检查和自动恢复
- **Kernel 重构** (`kernel.rs`, +569 行): 新增字段和方法重构

#### Agent Loop
- **上下文管理管线升级**: 4 阶段溢出恢复 + 动态上下文预算守卫
- **ContextBudget** (`context_budget.rs`, 355 行新增): 替代硬编码截断
- **Compactor 重写** (`compactor.rs`, +98 行): 3 阶段 LLM 智能压缩

#### 记忆系统
- **HTTP 后端** (`http_client.rs`, 246 行新增): 支持 PostgreSQL + pgvector
- **双后端架构**: SQLite + HTTP 自动切换/回退
- **向量搜索增强**: 候选扩展 10x + 余弦相似度重排序
- **任务队列 API**: task_post/claim/complete/list

#### Skills 系统
- **Freeze 机制**: Stable 模式下冻结注册表
- **Workspace Skills**: 项目级技能覆盖全局技能
- **安全扫描**: SkillVerifier prompt injection 检测

### 新增依赖
- `rmcp = "1.2"` (MCP 官方 Rust SDK)
- `rumqttc` (MQTT 客户端)

### 统计
- **98 个文件变更**: +15347/-1683 行
- **新增 14 个文件**: vertex.rs, mqtt.rs, heartbeat.rs, http_client.rs 等
- **121 个 commits** 合并自 main 分支

---

## v0.4.4 → v0.4.9 版本变更摘要

### 新增功能

#### 渠道系统 (Channels)
- **企业微信适配器** (`crates/openfang-channels/src/wecom.rs`): 691 行完整实现，支持消息收发、token 自动刷新
- **钉钉流式适配器** (`crates/openfang-channels/src/dingtalk_stream.rs`): 600 行，支持钉钉卡片消息流式处理
- **飞书增强** (`crates/openfang-channels/src/feishu.rs`): +914 行，支持更多消息类型
- **邮件渠道改进**: 支持 HTML 邮件和附件
- **Telegram 改进**: 支持更多消息格式和错误恢复
- **Mastodon 轮询修复**: 更稳定的长轮询机制

#### API 端点
- **POST /api/agents/{id}/restart**: 重启崩溃/卡住的 Agent，重置状态并取消运行任务
- **POST /api/agents/{id}/start**: 启动 Agent (别名重启)
- **POST /api/hands/upsert**: Hands 配置热更新
- **GET /api/config/schema**: 返回配置 TOML schema 供前端使用
- **GET /api/comms/events/stream**: SSE 流式事件推送

#### 图片处理流水线
- **ContentBlock::Image 增强**: 支持更多媒体类型 (png, jpeg, gif, webp)
- **Base64 内联编码**: 图片直接嵌入消息
- **临时目录存储**: 自动管理 `/tmp/openfang_uploads`
- **前端渲染**: 消息气泡内直接显示图片缩略图

#### 前端改进
- **PWA 支持**: 新增 `manifest.json` 和 `sw.js` 服务工作者
- **Agent 详情页**: 支持切换 Provider (不局限于切换模型)
- **国际化 (i18n)**: 完整的中文/英文双语切换
- **文件上传**: 拖拽上传、图片预览

### Bug 修复

| 修复内容 | 影响范围 |
|----------|----------|
| Agent 响应空值保护 | 防止前端显示空白消息 |
| Session 消息加载 | ToolResult 正确关联到工具调用 |
| Chromium 沙箱 | Root 环境下添加 `--no-sandbox` |
| Slack 链接预览 | 自动展开 unfurl 链接 |
| 工具错误引导 | 错误消息包含修复建议 |
| Agent 重命名 | 修复 ID 不一致问题 |
| 异步 Session 保存 | 防止阻塞主线程 |
| Docker 构建参数 | 支持多平台构建 |
| Codex ID Token | OAuth2 令牌刷新逻辑 |

### 配置变更

#### Config.toml 新增字段
```toml
# 企业微信配置
[wecom]
corp_id = "your_corp_id"
agent_id = "1000001"
secret = "your_secret"
token = "webhook_token"
encoding_aes_key = "your_aes_key"

# 图片上传配置
[uploads]
max_size_mb = 5
temp_dir = "/tmp/openfang_uploads"
```

### 依赖更新
- `mailparse` → 0.16.1 (更好的 MIME 支持)
- `tokio-tungstenite` → 0.28.0 (WebSocket 改进)
- `wasmtime` → 42.0.1 (WASM 沙箱升级)
- `ratatui` → 0.30.0 (TUI 界面更新)
- `rusqlite` → 0.38.0 (SQLite 绑定更新)

---

### 学习路线调整

根据新版本，建议补充以下内容:
1. **企业微信集成** (第 16 节补充)
2. **图片处理流程** (第 10 节补充)
3. **Agent 调试与重启** (第 7 节补充)
4. **PWA 离线支持** (第 25 节补充)

---

## 学习建议

1. **按顺序学习**：后续章节依赖前面的概念
2. **动手实践**：每节都有代码示例，建议本地运行验证
3. **代码对照**：文档中的代码片段都标注了文件路径和行号
4. **关注更新**：项目迭代快速，注意检查文档版本

---

## 下一步

🎉 **恭喜！25 节教程已全部完成！**

现在您拥有了完整的 OpenFang 知识体系：

**核心能力**：
- ✅ 14 Crates 架构与设计理念
- ✅ Agent 运行时与 LLM Driver 系统
- ✅ 工具执行、安全与记忆系统
- ✅ Hands 自主代理与 Channel 通信
- ✅ MCP 扩展与 A2A 协议
- ✅ API 服务器、CLI 与 Desktop 应用

**推荐学习路径**：
1. 按顺序阅读 25 节教程建立完整知识体系
2. 动手实践每节的代码示例
3. 参考文档修改和扩展功能
4. 贡献代码到 OpenFang 社区

---

*OpenFang v0.5.2 — 25 节完整教程系列*
*创建时间：2026-03-16 (更新于 2026-03-29)*
*🐍 OpenFang — Open-source Agent Operating System*
