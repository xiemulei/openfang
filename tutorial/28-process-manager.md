# Process Manager — 进程管理器

Version: v0.5.5

## 1. 核心功能

Process Manager 是一个交互式进程管理系统，用于管理持久化的进程会话，允许代理启动长时间运行的进程并与之交互。

### 1.1 主要功能

- **持久化进程管理**：启动和管理长时间运行的进程（如 REPL、服务器、监视器等）
- **进程交互**：向进程的 stdin 写入数据，从 stdout/stderr 读取输出
- **进程控制**：杀死不需要的进程
- **资源限制**：限制每个代理的进程数量
- **自动清理**：清理超时的进程
- **跨平台支持**：支持不同操作系统的进程管理

## 2. 架构设计

### 2.1 核心结构

```rust
/// Unique process identifier.
pub type ProcessId = String;

/// A managed persistent process.
struct ManagedProcess {
    /// stdin writer.
    stdin: Option<tokio::process::ChildStdin>,
    /// Accumulated stdout output.
    stdout_buf: Arc<Mutex<Vec<String>>>,
    /// Accumulated stderr output.
    stderr_buf: Arc<Mutex<Vec<String>>>,
    /// The child process handle.
    child: tokio::process::Child,
    /// Agent that owns this process.
    agent_id: String,
    /// Command that was started.
    command: String,
    /// When the process was started.
    started_at: std::time::Instant,
}

/// Process info for listing.
#[derive(Debug, Clone)]
pub struct ProcessInfo {
    /// Process ID.
    pub id: ProcessId,
    /// Agent that owns this process.
    pub agent_id: String,
    /// Command that was started.
    pub command: String,
    /// Whether the process is still running.
    pub alive: bool,
    /// Uptime in seconds.
    pub uptime_secs: u64,
}

/// Manager for persistent agent processes.
pub struct ProcessManager {
    processes: DashMap<ProcessId, ManagedProcess>,
    max_per_agent: usize,
    next_id: std::sync::atomic::AtomicU64,
}
```

### 2.2 工作流程

1. **进程启动**：代理请求启动一个新进程
2. **资源检查**：检查代理的进程数量是否达到上限
3. **进程创建**：创建子进程并设置管道
4. **后台读取**：为 stdout/stderr 启动后台读取任务
5. **进程管理**：将进程添加到管理列表
6. **进程交互**：通过 write/read 方法与进程交互
7. **进程终止**：通过 kill 方法终止进程
8. **自动清理**：定期清理超时的进程

## 3. 核心方法

### 3.1 `start`

启动一个新的持久化进程：

```rust
pub async fn start(
    &self,
    agent_id: &str,
    command: &str,
    args: &[String],
) -> Result<ProcessId, String> {
    // Check per-agent limit
    let agent_count = self
        .processes
        .iter()
        .filter(|entry| entry.value().agent_id == agent_id)
        .count();

    if agent_count >= self.max_per_agent {
        return Err(format!(
            "Agent '{}' already has {} processes (max: {})",
            agent_id, agent_count, self.max_per_agent
        ));
    }

    let mut child = tokio::process::Command::new(command)
        .args(args)
        .stdin(Stdio::piped())
        .stdout(Stdio::piped())
        .stderr(Stdio::piped())
        .spawn()
        .map_err(|e| format!("Failed to start process '{}': {}", command, e))?;

    // 省略后续代码...
}
```

### 3.2 `write`

向进程的 stdin 写入数据：

```rust
pub async fn write(&self, process_id: &str, data: &str) -> Result<(), String> {
    let mut entry = self
        .processes
        .get_mut(process_id)
        .ok_or_else(|| format!("Process '{}' not found", process_id))?;

    let proc = entry.value_mut();
    if let Some(stdin) = &mut proc.stdin {
        stdin
            .write_all(data.as_bytes())
            .await
            .map_err(|e| format!("Write failed: {}", e))?;
        stdin
            .flush()
            .await
            .map_err(|e| format!("Flush failed: {}", e))?;
        Ok(())
    } else {
        Err("Process stdin is closed".to_string())
    }
}
```

### 3.3 `read`

读取进程的 stdout/stderr 输出：

```rust
pub async fn read(&self, process_id: &str) -> Result<(Vec<String>, Vec<String>), String> {
    let entry = self
        .processes
        .get(process_id)
        .ok_or_else(|| format!("Process '{}' not found", process_id))?;

    let mut stdout = entry.stdout_buf.lock().await;
    let mut stderr = entry.stderr_buf.lock().await;

    let out_lines: Vec<String> = stdout.drain(..).collect();
    let err_lines: Vec<String> = stderr.drain(..).collect();

    Ok((out_lines, err_lines))
}
```

### 3.4 `kill`

杀死进程：

```rust
pub async fn kill(&self, process_id: &str) -> Result<(), String> {
    let (_, mut proc) = self
        .processes
        .remove(process_id)
        .ok_or_else(|| format!("Process '{}' not found", process_id))?;

    if let Some(pid) = proc.child.id() {
        debug!(process_id, pid, "Killing persistent process");
        let _ = crate::subprocess_sandbox::kill_process_tree(pid, 3000).await;
    }
    let _ = proc.child.kill().await;
    Ok(())
}
```

### 3.5 `list`

列出指定代理的所有进程：

```rust
pub fn list(&self, agent_id: &str) -> Vec<ProcessInfo> {
    self.processes
        .iter()
        .filter(|entry| entry.value().agent_id == agent_id)
        .map(|entry| {
            let alive = entry.value().child.id().is_some();
            ProcessInfo {
                id: entry.key().clone(),
                agent_id: entry.value().agent_id.clone(),
                command: entry.value().command.clone(),
                alive,
                uptime_secs: entry.value().started_at.elapsed().as_secs(),
            }
        })
        .collect()
}
```

### 3.6 `cleanup`

清理超时的进程：

```rust
pub async fn cleanup(&self, max_age_secs: u64) {
    let to_remove: Vec<ProcessId> = self
        .processes
        .iter()
        .filter(|entry| entry.value().started_at.elapsed().as_secs() > max_age_secs)
        .map(|entry| entry.key().clone())
        .collect();

    for id in to_remove {
        warn!(process_id = %id, "Cleaning up stale process");
        let _ = self.kill(&id).await;
    }
}
```

## 4. 配置与限制

### 4.1 默认配置

- **每个代理的最大进程数**：5个
- **进程输出缓冲区大小**：最多1000行，超过后会删除最旧的100行
- **进程ID格式**：`proc_<递增数字>`

### 4.2 配置示例

```rust
// 创建一个每个代理最多可以运行3个进程的ProcessManager
let process_manager = ProcessManager::new(3);

// 使用默认配置（每个代理最多5个进程）
let default_process_manager = ProcessManager::default();
```

## 5. 输出缓冲区管理

Process Manager 为每个进程的 stdout 和 stderr 维护独立的缓冲区：

- 缓冲区大小限制为 1000 行
- 当缓冲区达到上限时，会自动删除最旧的 100 行
- 每次调用 `read` 方法会清空缓冲区并返回所有积累的输出

## 6. 跨平台支持

Process Manager 设计考虑了跨平台兼容性：

- 使用 `tokio::process` 提供的跨平台进程管理
- 在测试中考虑了 Windows 和非 Windows 系统的差异
- 支持跨平台的进程树终止

## 7. 使用场景

### 7.1 典型使用流程

1. **启动进程**：代理启动一个 Python REPL 进程
2. **交互**：向进程发送 Python 命令并读取输出
3. **长期运行**：保持进程运行以进行后续交互
4. **资源管理**：当不再需要时，杀死进程释放资源

### 7.2 具体示例

```rust
// 启动一个 Python REPL 进程
let process_id = process_manager.start("agent1", "python3", &["-i"]).await?;

// 向进程发送命令
process_manager.write(&process_id, "print('Hello, World!')\n").await?;

// 读取进程输出
let (stdout, stderr) = process_manager.read(&process_id).await?;
println!("STDOUT: {:?}", stdout);
println!("STDERR: {:?}", stderr);

// 列出代理的所有进程
let processes = process_manager.list("agent1");

// 清理超时进程
process_manager.cleanup(3600).await; // 清理运行超过1小时的进程

// 杀死进程
process_manager.kill(&process_id).await?;
```

## 8. 集成点

- **工具执行**：作为 `shell_exec` 工具的后端，支持持久化进程
- **代理交互**：允许代理与长时间运行的进程交互
- **资源管理**：通过限制和自动清理机制管理系统资源
- **监控系统**：提供进程状态和资源使用情况的监控

## 9. 代码优化建议

1. **进程状态监控**：添加更详细的进程状态监控，如 CPU 和内存使用情况
2. **进程分组**：支持进程分组，便于批量管理相关进程
3. **进程优先级**：支持设置进程优先级，确保关键进程获得足够资源
4. **输出格式控制**：提供更灵活的输出格式控制，如实时输出、带时间戳的输出等
5. **异常处理增强**：增强异常处理，提供更详细的错误信息
6. **进程生命周期钩子**：添加进程启动、退出的钩子，便于执行自定义逻辑
7. **网络进程支持**：增强对网络服务进程的支持，如端口管理、健康检查等

## 10. 总结

Process Manager 是 OpenFang 系统中的一个重要组件，为代理提供了管理持久化进程的能力。它不仅支持基本的进程启动、交互和终止操作，还提供了资源限制和自动清理机制，确保系统资源的合理使用。

通过 Process Manager，代理可以启动和管理各种长时间运行的进程，如 REPL 环境、开发服务器、监控工具等，从而扩展了代理的能力范围。同时，Process Manager 的跨平台设计确保了它在不同操作系统上的一致行为。

合理使用 Process Manager 可以显著提高代理的工作效率，特别是在需要长时间运行的任务和交互式操作场景中。