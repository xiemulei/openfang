# 第 25 节：Cron Scheduler — 定时任务调度器

> **版本**: v0.5.5 (2026-03-29)
> **核心文件**:
> - `crates/openfang-kernel/src/cron.rs`
> - `crates/openfang-types/src/scheduler.rs`

## 学习目标

- [ ] 理解 Cron Scheduler 的核心功能和架构
- [ ] 掌握定时任务的配置和管理
- [ ] 了解三种调度模式：At、Every、Cron
- [ ] 掌握任务状态管理和错误处理
- [ ] 理解任务持久化和恢复机制

---

## 1. 核心功能

Cron Scheduler 是 OpenFang 内核的定时任务调度引擎，提供以下核心功能：

- **多模式调度**：支持 At（指定时间）、Every（间隔时间）、Cron（表达式）三种调度模式
- **任务管理**：添加、删除、启用/禁用任务
- **状态跟踪**：记录任务执行状态、连续错误次数
- **自动重试**：失败任务的指数退避重试
- **自动禁用**：连续失败达到阈值时自动禁用任务
- **持久化存储**：任务状态持久化到磁盘
- **代理任务管理**：支持代理任务的重新分配和清理

---

## 2. 架构设计

### 2.1 核心结构体

```rust
// crates/openfang-kernel/src/cron.rs:69-82
pub struct CronScheduler {
    /// 所有跟踪的任务，按唯一 ID 键控
    jobs: DashMap<CronJobId, JobMeta>,
    /// 持久化文件路径 (`<home>/cron_jobs.json`)
    persist_path: PathBuf,
    /// 所有代理的总任务数上限（原子，支持热重载）
    max_total_jobs: AtomicUsize,
}
```

### 2.2 任务元数据

```rust
// crates/openfang-kernel/src/cron.rs:36-63
pub struct JobMeta {
    /// 基础任务定义
    pub job: CronJob,
    /// 是否在单次成功执行后移除
    pub one_shot: bool,
    /// 上次执行的人类可读状态（如 "ok" 或 "error: ..."）
    pub last_status: Option<String>,
    /// 连续失败次数
    pub consecutive_errors: u32,
}

impl JobMeta {
    /// 用默认元数据包装 `CronJob`
    pub fn new(job: CronJob, one_shot: bool) -> Self {
        Self {
            job,
            one_shot,
            last_status: None,
            consecutive_errors: 0,
        }
    }
}
```

---

## 3. 调度模式

### 3.1 CronSchedule 枚举

```rust
// crates/openfang-types/src/scheduler.rs
pub enum CronSchedule {
    /// 在指定时间执行一次
    At { at: chrono::DateTime<Utc> },
    /// 每隔指定秒数执行
    Every { every_secs: u32 },
    /// 使用 cron 表达式执行
    Cron { expr: String, tz: Option<String> },
}
```

### 3.2 调度模式对比

| 模式 | 描述 | 适用场景 | 示例 |
|------|------|----------|------|
| `At` | 在指定时间执行一次 | 特定时间点的一次性任务 | `2026-12-31T23:59:59Z` |
| `Every` | 每隔指定秒数执行 | 周期性任务 | `3600` (每小时) |
| `Cron` | 使用 cron 表达式 | 复杂的定时任务 | `0 9 * * 1-5` (工作日 9:00) |

### 3.3 Cron 表达式支持

Cron Scheduler 支持标准的 5 字段和 6 字段 cron 表达式：

- **5 字段**：`分 时 日 月 星期`
- **6 字段**：`秒 分 时 日 月 星期`

**时区支持**：可以指定时区，如 `America/New_York`、`Asia/Shanghai` 等

---

## 4. 任务管理

### 4.1 添加任务

```rust
pub fn add_job(&self, mut job: CronJob, one_shot: bool) -> OpenFangResult<CronJobId> {
    // 全局限制
    let max_jobs = self.max_total_jobs.load(Ordering::Relaxed);
    if self.jobs.len() >= max_jobs {
        return Err(OpenFangError::Internal(format!(
            "Global cron job limit reached ({})",
            max_jobs
        )));
    }

    // 每个代理的任务数
    let agent_count = self
        .jobs
        .iter()
        .filter(|r| r.value().job.agent_id == job.agent_id)
        .count();

    // 验证任务
    job.validate(agent_count)
        .map_err(OpenFangError::InvalidInput)?;

    // 计算初始 next_run
    job.next_run = Some(compute_next_run(&job.schedule));

    let id = job.id;
    self.jobs.insert(id, JobMeta::new(job, one_shot));
    Ok(id)
}
```

### 4.2 移除任务

```rust
pub fn remove_job(&self, id: CronJobId) -> OpenFangResult<CronJob> {
    self.jobs
        .remove(&id)
        .map(|(_, meta)| meta.job)
        .ok_or_else(|| OpenFangError::Internal(format!("Cron job {id} not found")))
}
```

### 4.3 启用/禁用任务

```rust
pub fn set_enabled(&self, id: CronJobId, enabled: bool) -> OpenFangResult<()> {
    match self.jobs.get_mut(&id) {
        Some(mut meta) => {
            meta.job.enabled = enabled;
            if enabled {
                meta.consecutive_errors = 0;
                meta.job.next_run = Some(compute_next_run(&meta.job.schedule));
            }
            Ok(())
        }
        None => Err(OpenFangError::Internal(format!("Cron job {id} not found"))),
    }
}
```

---

## 5. 任务执行

### 5.1 查找到期任务

```rust
pub fn due_jobs(&self) -> Vec<CronJob> {
    let now = Utc::now();
    let mut due = Vec::new();
    for mut entry in self.jobs.iter_mut() {
        let meta = entry.value_mut();
        if meta.job.enabled && meta.job.next_run.map(|t| t <= now).unwrap_or(false) {
            due.push(meta.job.clone());
            // 预推进 next_run，防止任务在执行期间重复触发
            meta.job.next_run = Some(compute_next_run_after(&meta.job.schedule, now));
        }
    }
    due
}
```

### 5.2 任务认领

```rust
pub fn try_claim_for_run(&self, id: CronJobId) -> Result<CronJob, ClaimError> {
    match self.jobs.get_mut(&id) {
        None => Err(ClaimError::NotFound),
        Some(mut entry) => {
            let meta = entry.value_mut();
            if !meta.job.enabled {
                return Err(ClaimError::Disabled);
            }
            let now = Utc::now();
            if meta.job.next_run.map(|t| t <= now).unwrap_or(false) {
                meta.job.next_run = Some(compute_next_run_after(&meta.job.schedule, now));
            }
            Ok(meta.job.clone())
        }
    }
}
```

### 5.3 记录执行结果

**成功执行**：
```rust
pub fn record_success(&self, id: CronJobId) {
    let should_remove = {
        if let Some(mut meta) = self.jobs.get_mut(&id) {
            meta.job.last_run = Some(Utc::now());
            meta.last_status = Some("ok".to_string());
            meta.consecutive_errors = 0;
            meta.one_shot
        } else {
            return;
        }
    };
    if should_remove {
        self.jobs.remove(&id);
    }
}
```

**失败执行**：
```rust
pub fn record_failure(&self, id: CronJobId, error_msg: &str) {
    if let Some(mut meta) = self.jobs.get_mut(&id) {
        meta.job.last_run = Some(Utc::now());
        meta.last_status = Some(format!(
            "error: {}",
            openfang_types::truncate_str(error_msg, 256)
        ));
        meta.consecutive_errors += 1;
        if meta.consecutive_errors >= MAX_CONSECUTIVE_ERRORS {
            warn!(
                job_id = %id,
                errors = meta.consecutive_errors,
                "Auto-disabling cron job after repeated failures"
            );
            meta.job.enabled = false;
        } else {
            let now = Utc::now();
            if meta.job.next_run.map(|t| t <= now).unwrap_or(true) {
                meta.job.next_run = Some(compute_next_run_after(&meta.job.schedule, now));
            }
        }
    }
}
```

---

## 6. 持久化机制

### 6.1 加载任务

```rust
pub fn load(&self) -> OpenFangResult<usize> {
    if !self.persist_path.exists() {
        return Ok(0);
    }
    let data = std::fs::read_to_string(&self.persist_path)
        .map_err(|e| OpenFangError::Internal(format!("Failed to read cron jobs: {e}")))?;
    let metas: Vec<JobMeta> = serde_json::from_str(&data)
        .map_err(|e| OpenFangError::Internal(format!("Failed to parse cron jobs: {e}")))?;
    let count = metas.len();
    for meta in metas {
        self.jobs.insert(meta.job.id, meta);
    }
    info!(count, "Loaded cron jobs from disk");
    Ok(count)
}
```

### 6.2 持久化任务

```rust
pub fn persist(&self) -> OpenFangResult<()> {
    let metas: Vec<JobMeta> = self.jobs.iter().map(|r| r.value().clone()).collect();
    let data = serde_json::to_string_pretty(&metas)
        .map_err(|e| OpenFangError::Internal(format!("Failed to serialize cron jobs: {e}")))?;
    let tmp_path = self.persist_path.with_extension("json.tmp");
    std::fs::write(&tmp_path, data.as_bytes()).map_err(|e| {
        OpenFangError::Internal(format!("Failed to write cron jobs temp file: {e}"))
    })?;
    std::fs::rename(&tmp_path, &self.persist_path).map_err(|e| {
        OpenFangError::Internal(format!("Failed to rename cron jobs file: {e}"))
    })?;
    debug!(count = metas.len(), "Persisted cron jobs");
    Ok(())
}
```

---

## 7. 代理任务管理

### 7.1 重新分配代理任务

当代理重启获得新 UUID 时，重新分配任务：

```rust
pub fn reassign_agent_jobs(&self, old_agent_id: AgentId, new_agent_id: AgentId) -> usize {
    let mut count = 0;
    for mut entry in self.jobs.iter_mut() {
        if entry.value().job.agent_id == old_agent_id {
            entry.value_mut().job.agent_id = new_agent_id;
            // 重置连续错误，让任务在新代理上重新开始
            entry.value_mut().consecutive_errors = 0;
            if !entry.value().job.enabled {
                // 重新启用因代理 ID 失效而自动禁用的任务
                if entry
                    .value()
                    .last_status
                    .as_deref()
                    .is_some_and(|s| s.contains("not found") || s.contains("No such agent"))
                {
                    entry.value_mut().job.enabled = true;
                    entry.value_mut().job.next_run =
                        Some(compute_next_run(&entry.value().job.schedule));
                }
            }
            count += 1;
        }
    }
    if count > 0 {
        info!(
            old_agent = %old_agent_id,
            new_agent = %new_agent_id,
            count,
            "Reassigned cron jobs to new agent"
        );
    }
    count
}
```

### 7.2 移除代理任务

当代理被删除时，清理其任务：

```rust
pub fn remove_agent_jobs(&self, agent_id: AgentId) -> usize {
    let ids: Vec<CronJobId> = self
        .jobs
        .iter()
        .filter(|r| r.value().job.agent_id == agent_id)
        .map(|r| *r.key())
        .collect();
    let count = ids.len();
    for id in ids {
        self.jobs.remove(&id);
    }
    if count > 0 {
        info!(agent = %agent_id, count, "Removed cron jobs for deleted agent");
    }
    count
}
```

---

## 8. 错误处理

### 8.1 自动禁用机制

当任务连续失败达到 `MAX_CONSECUTIVE_ERRORS`（默认 5 次）时，会自动禁用任务：

```rust
const MAX_CONSECUTIVE_ERRORS: u32 = 5;

// 在 record_failure 中
if meta.consecutive_errors >= MAX_CONSECUTIVE_ERRORS {
    warn!(
        job_id = %id,
        errors = meta.consecutive_errors,
        "Auto-disabling cron job after repeated failures"
    );
    meta.job.enabled = false;
}
```

### 8.2 错误消息处理

错误消息会被截断到 256 个字符，以避免存储过大的错误信息：

```rust
meta.last_status = Some(format!(
    "error: {}",
    openfang_types::truncate_str(error_msg, 256)
));
```

---

## 9. 时区支持

Cron Scheduler 支持时区感知的 cron 表达式：

```rust
pub fn compute_next_run_after(
    schedule: &CronSchedule,
    after: chrono::DateTime<Utc>,
) -> chrono::DateTime<Utc> {
    match schedule {
        CronSchedule::Cron { expr, tz } => {
            // 转换标准 5/6 字段 cron 到 7 字段
            let trimmed = expr.trim();
            let fields: Vec<&str> = trimmed.split_whitespace().collect();
            let seven_field = match fields.len() {
                5 => format!("0 {trimmed} *"),
                6 => format!("{trimmed} *"),
                _ => expr.clone(),
            };

            // 添加 1 秒，确保 .after() 返回严格未来的时间
            let base = after + Duration::seconds(1);

            match seven_field.parse::<cron::Schedule>() {
                Ok(sched) => {
                    // 如果指定了时区，在该时区计算下一次触发时间
                    let next_utc = match tz.as_deref() {
                        Some(tz_str) if !tz_str.is_empty() && tz_str != "UTC" => {
                            match tz_str.parse::<chrono_tz::Tz>() {
                                Ok(timezone) => {
                                    let base_local = base.with_timezone(&timezone);
                                    sched
                                        .after(&base_local)
                                        .next()
                                        .map(|dt| dt.with_timezone(&Utc))
                                }
                                Err(_) => {
                                    warn!(
                                        "Invalid timezone '{}' in cron job, falling back to UTC",
                                        tz_str
                                    );
                                    sched.after(&base).next()
                                }
                            }
                        }
                        _ => sched.after(&base).next(),
                    };
                    next_utc.unwrap_or_else(|| after + Duration::hours(1))
                }
                Err(e) => {
                    warn!("Failed to parse cron expression '{}': {}", expr, e);
                    after + Duration::hours(1)
                }
            }
        }
        // 其他调度模式处理...
    }
}
```

---

## 10. 使用场景

### 10.1 常见使用场景

1. **定期数据备份**：每天凌晨执行备份任务
2. **系统维护**：每周执行系统维护任务
3. **报表生成**：每月生成业务报表
4. **监控检查**：每 5 分钟执行监控检查
5. **定时通知**：在特定时间发送通知

### 10.2 任务配置示例

**每天 9:00 执行系统维护**：
```rust
let job = CronJob {
    id: CronJobId::new(),
    agent_id: agent_id,
    name: "系统维护".into(),
    enabled: true,
    schedule: CronSchedule::Cron {
        expr: "0 9 * * *".into(),
        tz: Some("Asia/Shanghai".into()),
    },
    action: CronAction::SystemEvent {
        text: "system_maintenance".into(),
    },
    delivery: CronDelivery::None,
    created_at: Utc::now(),
    last_run: None,
    next_run: None,
};
scheduler.add_job(job, false).unwrap();
```

**每 30 分钟检查系统状态**：
```rust
let job = CronJob {
    id: CronJobId::new(),
    agent_id: agent_id,
    name: "系统状态检查".into(),
    enabled: true,
    schedule: CronSchedule::Every { every_secs: 1800 }, // 30分钟
    action: CronAction::SystemEvent {
        text: "check_system_status".into(),
    },
    delivery: CronDelivery::None,
    created_at: Utc::now(),
    last_run: None,
    next_run: None,
};
scheduler.add_job(job, false).unwrap();
```

---

## 11. 安全特性

1. **线程安全**：使用 DashMap 实现并发安全的任务管理
2. **资源限制**：全局任务数限制和每个代理的任务数限制
3. **错误处理**：连续失败自动禁用，防止无限重试
4. **持久化安全**：使用原子写入（先写临时文件，再重命名）确保文件完整性
5. **错误消息截断**：限制错误消息长度，防止存储溢出

---

## 12. 性能优化

1. **高效查找**：使用 DashMap 实现 O(1) 时间复杂度的任务查找
2. **批量处理**：批量加载和持久化任务
3. **预计算**：预先计算下一次执行时间，避免每次检查都重新计算
4. **惰性执行**：只在任务到期时才执行，避免不必要的计算
5. **指数退避**：失败任务使用指数退避策略，避免频繁重试

---

## 13. 测试覆盖

Cron Scheduler 包含全面的测试覆盖：

- **基本功能测试**：添加、删除、列出任务
- **限制测试**：全局任务数限制、每个代理任务数限制
- **执行测试**：成功执行、失败执行、自动禁用
- **持久化测试**：保存和加载任务
- **调度测试**：不同调度模式的计算
- **时区测试**：时区感知的 cron 表达式
- **代理管理测试**：重新分配和移除代理任务
- **错误处理测试**：错误消息截断、无效表达式处理

---

## 完成检查清单

- [ ] 理解 Cron Scheduler 的核心功能和架构
- [ ] 掌握定时任务的配置和管理
- [ ] 了解三种调度模式：At、Every、Cron
- [ ] 掌握任务状态管理和错误处理
- [ ] 理解任务持久化和恢复机制
- [ ] 了解代理任务管理和时区支持

---

## 下一步

前往 [第 26 节：Approval Manager — 审批管理器](./26-approval-manager.md)

---

*创建时间：2026-03-29*
*OpenFang v0.5.5*