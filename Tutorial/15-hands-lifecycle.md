# 第 15 节：Hands 系统 — 生命周期管理

> **版本**: v0.4.4 (2026-03-16)
> **核心文件**: `crates/openfang-hands/src/registry.rs`, `crates/openfang-hands/src/manager.rs`

---

## 学习目标

- [ ] 掌握 HandRegistry 的实现和状态管理
- [ ] 理解 Hand 实例的状态流转
- [ ] 掌握状态持久化机制和恢复流程
- [ ] 理解 Dashboard Metrics 的采集和展示

---

## 1. HandRegistry 注册表

### 1.1 架构设计

**文件**: `crates/openfang-hands/src/registry.rs`

```rust
/// Hands 注册表 — 跟踪所有已激活的 Hand 实例
pub struct HandRegistry {
    /// 已激活的 Hand 实例映射
    instances: DashMap<HandId, HandInstance>,
    /// 状态存储（持久化）
    store: HandStateStore,
    /// 事件总线
    event_tx: broadcast::Sender<HandEvent>,
}

/// Hand 实例运行时状态
pub struct HandInstance {
    /// Hand ID
    pub id: HandId,
    /// 运行状态
    pub state: HandState,
    /// Agent ID（关联到 Agent Loop）
    pub agent_id: AgentId,
    /// 启动时间
    pub started_at: Instant,
    /// Dashboard 指标
    pub metrics: HandMetrics,
}
```

### 1.2 状态枚举

```rust
#[derive(Clone, Debug, Serialize, Deserialize)]
pub enum HandState {
    /// 已激活、正在运行
    Running,
    /// 已暂停（保留状态）
    Paused,
    /// 已停止（清除状态）
    Stopped,
    /// 错误状态
    Error { message: String },
}
```

**状态流转**：

```
         ┌──────────────────────────────────────────┐
         │                                          ▼
   ┌─────────┐  pause   ┌─────────┐  resume  ┌─────────┐
   │ Running │─────────▶│ Paused  │─────────▶│ Running │
   └─────────┘          └─────────┘          └─────────┘
       │  │                                     │
       │  │ deactivate                          │ error
       │  ▼                                     ▼
       │  ┌─────────┐                     ┌─────────┐
       └─▶│ Stopped │                     │  Error  │
          └─────────┘                     └─────────┘
```

---

## 2. 状态持久化

### 2.1 存储结构

**文件**: `~/.openfang/hands/{id}/state.bin`

```rust
/// 手状态序列化结构
#[derive(Serialize, Deserialize)]
pub struct HandStateData {
    /// Hand ID
    pub id: HandId,
    /// 当前迭代次数
    pub iteration: u32,
    /// 会话历史（压缩）
    pub session_history: Vec<CompressedMessage>,
    /// 采集的记忆 ID 列表
    pub memory_ids: Vec<MemoryId>,
    /// Dashboard 指标快照
    pub metrics_snapshot: HandMetrics,
    /// 最后更新时间
    pub updated_at: u64,  // Unix timestamp
}
```

### 2.2 持久化流程

```rust
impl HandRegistry {
    /// 持久化 Hand 状态
    pub async fn persist_state(&self, hand_id: &HandId) -> Result<(), HandError> {
        let instance = self.instances.get(hand_id)
            .ok_or(HandError::NotFound)?;

        let state_data = HandStateData {
            id: hand_id.clone(),
            iteration: instance.metrics.iteration,
            session_history: compress_messages(&instance.agent_history),
            memory_ids: instance.collected_memories.clone(),
            metrics_snapshot: instance.metrics.clone(),
            updated_at: std::time::SystemTime::now()
                .duration_since(std::time::UNIX_EPOCH)?
                .as_secs(),
        };

        // 序列化并写入文件
        let bytes = rmp_serde::to_vec(&state_data)?;
        let compressed = zstd::stream::encode_all(&bytes[..], 3)?;

        let state_path = self.get_state_path(hand_id);
        tokio::fs::write(&state_path, &compressed).await?;

        Ok(())
    }
}
```

### 2.3 恢复流程

```rust
impl HandRegistry {
    /// 从持久化状态恢复 Hand
    pub async fn restore_hand(&self, hand_id: &HandId) -> Result<(), HandError> {
        let state_path = self.get_state_path(hand_id);

        // 读取压缩文件
        let compressed = tokio::fs::read(&state_path).await?;

        // 解压并反序列化
        let bytes = zstd::stream::decode_all(&compressed[..])?;
        let state_data: HandStateData = rmp_serde::from_slice(&bytes)?;

        // 重建 HandInstance
        let instance = HandInstance {
            id: state_data.id.clone(),
            state: HandState::Paused,  // 恢复时为暂停状态
            agent_id: AgentId::new(),
            started_at: Instant::now(),
            metrics: state_data.metrics_snapshot,
            agent_history: decompress_messages(&state_data.session_history),
            collected_memories: state_data.memory_ids,
        };

        self.instances.insert(hand_id.clone(), instance);

        tracing::info!("Restored hand {} from iteration {}",
            hand_id, state_data.iteration);

        Ok(())
    }
}
```

---

## 3. Dashboard Metrics

### 3.1 指标类型

**文件**: `crates/openfang-types/src/hand.rs`

```rust
/// Dashboard 指标定义
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct MetricDefinition {
    /// 指标名称
    pub name: String,
    /// 指标类型
    pub kind: MetricKind,
    /// 单位（可选）
    pub unit: Option<String>,
}

#[derive(Clone, Debug, Serialize, Deserialize)]
pub enum MetricKind {
    /// 计数器（只增不减）
    Counter,
    /// 仪表盘（可增可减）
    Gauge,
    /// 时间戳
    Timestamp,
    /// 字符串状态
    Status,
}
```

### 3.2 各 Hand 的指标

| Hand | 指标 1 | 指标 2 | 指标 3 | 指标 4 |
|------|--------|--------|--------|--------|
| **Clip** | videos_processed | clips_generated | total_duration | last_processed |
| **Lead** | leads_discovered | leads_qualified | avg_score | last_scan |
| **Collector** | intel_items | knowledge_nodes | alerts_sent | graph_size |
| **Predictor** | predictions_made | brier_score | active_forecasts | last_update |
| **Researcher** | reports_generated | sources_consulted | avg_quality | last_run |
| **Twitter** | posts_created | engagement_rate | followers_change | last_post |
| **Browser** | sessions_completed | forms_filled | purchases_pending | last_action |
| **Trader** | trades_analyzed | signals_detected | pnl_total | last_trade |

### 3.3 指标采集

```rust
/// 更新 Dashboard 指标
impl HandInstance {
    pub fn increment_counter(&mut self, metric_name: &str, delta: u64) {
        if let Some(metric) = self.metrics.get_mut(metric_name) {
            if let MetricValue::Counter(ref mut value) = metric.value {
                *value += delta;
            }
        }
    }

    pub fn set_gauge(&mut self, metric_name: &str, value: f64) {
        if let Some(metric) = self.metrics.get_mut(metric_name) {
            if let MetricValue::Gauge(ref mut val) = metric.value {
                *val = value;
            }
        }
    }

    pub fn set_timestamp(&mut self, metric_name: &str) {
        if let Some(metric) = self.metrics.get_mut(metric_name) {
            if let MetricValue::Timestamp(ref mut ts) = metric.value {
                *ts = std::time::SystemTime::now()
                    .duration_since(std::time::UNIX_EPOCH)
                    .unwrap()
                    .as_secs();
            }
        }
    }
}
```

---

## 4. 生命周期事件

### 4.1 事件类型

**文件**: `crates/openfang-hands/src/events.rs`

```rust
/// Hand 生命周期事件
#[derive(Clone, Debug, Serialize)]
pub enum HandEvent {
    /// Hand 已激活
    Activated { id: HandId, agent_id: AgentId },
    /// Hand 已暂停
    Paused { id: HandId },
    /// Hand 已恢复
    Resumed { id: HandId },
    /// Hand 已停止
    Stopped { id: HandId },
    /// Hand 发生错误
    Error { id: HandId, error: String },
    /// 指标更新
    MetricsUpdated { id: HandId, metric: String, value: f64 },
    /// 迭代完成
    IterationComplete { id: HandId, iteration: u32, result: String },
}
```

### 4.2 事件订阅

```rust
// 订阅 Hand 事件
let mut rx = kernel.subscribe_to_hand_events();

tokio::spawn(async move {
    while let Ok(event) = rx.recv().await {
        match event {
            HandEvent::Activated { id, .. } => {
                tracing::info!("Hand {} activated", id);
            }
            HandEvent::MetricsUpdated { id, metric, value } => {
                tracing::debug!("Hand {} metric {}: {}", id, metric, value);
            }
            HandEvent::IterationComplete { id, iteration, result } => {
                tracing::info!("Hand {} completed iteration {}: {}", id, iteration, result);
            }
            _ => {}
        }
    }
});
```

---

## 5. CLI 命令详解

### 5.1 列出所有 Hands

```bash
openfang hand list
```

**输出**：
```
Available Hands (8 bundled):
  ● clip          - YouTube video to shorts converter
  ● lead          - Lead discovery and qualification
  ● collector     - OSINT intelligence collector
  ● predictor     - Superforecasting engine
  ● researcher    - Deep autonomous researcher
  ● twitter       - Autonomous Twitter manager
  ● browser       - Web automation agent
  ● trader        - Trading analysis agent

Active instances:
  researcher (running, iteration 42, uptime 2h 15m)
  lead (paused, iteration 17)
```

### 5.2 激活 Hand

```bash
openfang hand activate researcher
```

**执行流程**：
1. 查找 bundled Hands
2. 解析 HAND.toml
3. 检查依赖（工具、模型、渠道）
4. 创建 Agent 实例
5. 注册到 HandRegistry
6. 发送 `HandEvent::Activated`
7. 开始 Agent Loop

### 5.3 查看状态

```bash
openfang hand status researcher --json
```

**JSON 输出**：
```json
{
  "id": "researcher",
  "state": "running",
  "agent_id": "a1b2c3d4-...",
  "iteration": 42,
  "started_at": "2026-03-15T08:30:00Z",
  "uptime_secs": 8100,
  "metrics": {
    "reports_generated": 12,
    "sources_consulted": 847,
    "avg_report_quality": 0.89,
    "last_run": "2026-03-16T06:00:00Z"
  }
}
```

### 5.4 暂停/恢复

```bash
# 暂停（保留状态）
openfang hand pause researcher

# 恢复
openfang hand resume researcher
```

### 5.5 停止

```bash
openfang hand deactivate researcher
```

**效果**：
- 停止 Agent Loop
- 清除运行状态
- 保留 Dashboard 指标历史
- 发送 `HandEvent::Stopped`

---

## 6. 并发与锁

### 6.1 线程安全

`HandRegistry` 使用 `DashMap` 实现并发安全的实例管理：

```rust
pub struct HandRegistry {
    // DashMap 提供细粒度锁（每 key 独立锁）
    instances: DashMap<HandId, HandInstance>,
    ...
}

// 安全并发访问
let instance = registry.instances.get(&hand_id);
// 自动持有读锁， Drop 时释放
```

### 6.2 状态一致性

```rust
impl HandRegistry {
    /// 原子性状态转换
    pub fn transition_state(
        &self,
        hand_id: &HandId,
        from: HandState,
        to: HandState,
    ) -> Result<(), HandError> {
        let mut instance = self.instances.get_mut(hand_id)
            .ok_or(HandError::NotFound)?;

        // 检查当前状态
        if instance.state != from {
            return Err(HandError::InvalidStateTransition {
                expected: from,
                actual: instance.state.clone(),
            });
        }

        // 原子性转换
        instance.state = to;

        // 持久化
        self.persist_state(hand_id)?;

        Ok(())
    }
}
```

---

## 完成检查清单

- [ ] 掌握 HandRegistry 的实现和状态管理
- [ ] 理解 Hand 实例的状态流转
- [ ] 掌握状态持久化机制和恢复流程
- [ ] 理解 Dashboard Metrics 的采集和展示

---

## 下一步

前往 [第 16 节：Channel 系统 — 消息渠道](./16-channels-bridge.md)

---

*创建时间：2026-03-16*
*OpenFang v0.4.4*
