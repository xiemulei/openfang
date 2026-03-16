# 第 14 节：Hands 系统 — 配置与激活

> **版本**: v0.4.4 (2026-03-16)
> **核心文件**: `crates/openfang-hands/`, `crates/openfang-types/src/hand.rs`

---

## 学习目标

- [ ] 理解 Hands 系统的设计理念和架构
- [ ] 掌握 HAND.toml 配置文件的结构和字段
- [ ] 理解 Requirements、Settings 和 Dashboard Metrics 配置
- [ ] 掌握 Hand 的激活和配置流程

---

## 1. Hands 系统概述

### 1.1 什么是 Hands

Hands 是 OpenFang 的**自主代理能力包**——预构建的、可立即部署的 Agent 配置，能够独立运行、无需持续提示。

**传统 Agent vs Hands**：

| 传统 Agent | Hands |
|------------|-------|
| 等待用户输入 | 自主运行、主动工作 |
| 单次对话 | 持续运行、24/7 监控 |
| 通用能力 | 领域专业化 |
| 需要手动提示 | 内置 Playbook（500+ 词系统提示） |

### 1.2 设计理念

```
┌─────────────────────────────────────────────────────────────┐
│                      Hand = 能力包                           │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ HAND.toml   │  │ System      │  │ SKILL.md    │         │
│  │ 配置清单     │  │ Prompt      │  │ 领域知识     │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│  ┌─────────────┐  ┌─────────────┐                          │
│  │ Guardrails  │  │ Dashboard   │                          │
│  │ 审批门控     │  │ Metrics     │                          │
│  └─────────────┘  └─────────────┘                          │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. HAND.toml 配置文件

### 2.1 文件结构

**示例**: `crates/openfang-hands/bundled/researcher/HAND.toml`

```toml
[hand]
id = "researcher"
name = "Researcher"
version = "1.2.0"
description = "Deep autonomous researcher with multi-source validation"
author = "OpenFang Team"

[requirements]
tools = ["web_search", "web_fetch", "summarize", "cite"]
models = ["claude-sonnet-4-20250514", "gpt-4o"]
channels = ["email", "telegram"]
permissions = ["web_access", "file_write"]

[settings]
schedule = "0 6 * * *"  # Daily at 6 AM
max_iterations = 50
context_window = 200000
thinking_budget = 100000
auto_approve = false

[dashboard]
metrics = [
    { name = "reports_generated", type = "counter" },
    { name = "sources_consulted", type = "counter" },
    { name = "avg_report_quality", type = "gauge", unit = "score" },
    { name = "last_run", type = "timestamp" },
]
```

### 2.2 字段详解

#### [hand] 元数据

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 唯一标识符（小写、无空格） |
| `name` | string | 显示名称 |
| `version` | string | 语义化版本号 |
| `description` | string | 功能描述 |
| `author` | string | 作者/团队名称 |

#### [requirements] 依赖

| 字段 | 类型 | 说明 |
|------|------|------|
| `tools` | string[] | 需要的工具列表 |
| `models` | string[] | 支持的模型列表（ fallback 顺序） |
| `channels` | string[] | 输出渠道 |
| `permissions` | string[] | 需要的权限 |

#### [settings] 运行配置

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `schedule` | string | Cron 表达式 | - |
| `max_iterations` | u32 | 最大迭代次数 | 100 |
| `context_window` | u32 | 上下文窗口大小 | 200000 |
| `thinking_budget` | u32 | 思考 token 预算 | 100000 |
| `auto_approve` | bool | 自动审批敏感操作 | false |

#### [dashboard] 监控指标

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 指标名称 |
| `type` | enum | 指标类型：`counter`, `gauge`, `timestamp` |
| `unit` | string | 单位（可选） |

---

## 3. 内置 Hands 列表

### 3.1 8 个 Bundled Hands

| Hand | 职责 | 核心能力 |
|------|------|----------|
| **Clip** | YouTube 视频剪辑 | 视频下载、精彩片段识别、竖屏裁剪、字幕生成、多平台发布 |
| **Lead** | 潜在客户挖掘 | ICP 匹配、网络调研、评分、去重、报告生成 |
| **Collector** | 开源情报收集 | 目标监控、变更检测、情感追踪、知识图谱 |
| **Predictor** | 超级预测引擎 | 信号收集、校准推理、置信区间、Brier 分数追踪 |
| **Researcher** | 深度研究 | 多源交叉验证、CRAAP 评估、引用报告、多语言支持 |
| **Twitter** |  autonomous 推特管理 | 多格式内容创作、定时发布、互动回复、表现分析 |
| **Browser** | 网页自动化 | 导航、填表、点击、多步骤工作流、购买审批门控 |
| **Trader** | 交易分析 | 市场监控、信号识别、风险评估、交易执行（需审批） |

### 3.2 代码位置

**文件**: `crates/openfang-hands/src/bundled.rs`

```rust
pub fn bundled_hands() -> Vec<(&'static str, &'static str, &'static str)> {
    vec![
        ("clip", include_str!("../bundled/clip/HAND.toml"), ...),
        ("lead", include_str!("../bundled/lead/HAND.toml"), ...),
        ("collector", include_str!("../bundled/collector/HAND.toml"), ...),
        ("predictor", include_str!("../bundled/predictor/HAND.toml"), ...),
        ("researcher", include_str!("../bundled/researcher/HAND.toml"), ...),
        ("twitter", include_str!("../bundled/twitter/HAND.toml"), ...),
        ("browser", include_str!("../bundled/browser/HAND.toml"), ...),
        ("trader", include_str!("../bundled/trader/HAND.toml"), ...),
    ]
}
```

---

## 4. 激活 Hand

### 4.1 CLI 命令

```bash
# 激活 Hand
openfang hand activate researcher

# 查看状态
openfang hand status researcher

# 暂停（保留状态）
openfang hand pause researcher

# 恢复
openfang hand resume researcher

# 停止并清除状态
openfang hand deactivate researcher
```

### 4.2 激活流程

```
1. 读取 HAND.toml
   ↓
2. 检查 Requirements（工具/模型/渠道）
   ↓
3. 加载 System Prompt + SKILL.md
   ↓
4. 注册 Dashboard Metrics
   ↓
5. 启动 Agent Loop
   ↓
6. 持久化状态到 ~/.openfang/hands/{id}/state.bin
```

---

## 5. Guardrails 审批门控

### 5.1 敏感操作列表

| 操作 | 默认行为 | 配置 |
|------|----------|------|
| 网页购买 | **要求审批** | `permissions: ["purchase_approval"]` |
| 文件写入 | 要求审批 | `permissions: ["file_write"]` |
| API 调用 | 自动（有配额限制） | `permissions: ["api_access"]` |
| 发布内容 | 要求审批（Twitter Hand） | `permissions: ["publish_approval"]` |

### 5.2 审批流程

```
Hand 触发敏感操作
   ↓
创建 Approval Request（存入数据库）
   ↓
发送通知到配置渠道（Telegram/Email）
   ↓
等待用户响应（approve/reject）
   ↓
执行或取消操作
   ↓
记录审计日志
```

---

## 6. 状态持久化

### 6.1 状态文件

**位置**: `~/.openfang/hands/{id}/state.bin`

**内容**：
- 当前迭代计数器
- 已收集的上下文/记忆
- 会话历史
- Dashboard 指标快照

### 6.2 状态恢复

Hand 被暂停后重新激活时：
1. 读取 state.bin
2. 反序列化为 `HandState`
3. 恢复到暂停前的迭代点
4. 继续执行

---

## 完成检查清单

- [ ] 理解 Hands 系统的设计理念和架构
- [ ] 掌握 HAND.toml 配置文件的结构和字段
- [ ] 理解 Requirements、Settings 和 Dashboard Metrics 配置
- [ ] 掌握 Hand 的激活和配置流程

---

## 下一步

前往 [第 15 节：Hands 系统 — 生命周期管理](./15-hands-lifecycle.md)

---

*创建时间：2026-03-16*
*OpenFang v0.4.4*
