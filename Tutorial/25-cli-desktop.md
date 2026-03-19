# 第 25 节：CLI 与 Desktop 应用

> **版本**: v0.4.9 (2026-03-19)
> **核心文件**: `crates/openfang-cli/src/main.rs`, `crates/openfang-desktop/src/lib.rs`
> **新增功能**: PWA 离线支持、manifest.json、Service Worker

---

## 学习目标

- [ ] 掌握 CLI 命令结构和子命令设计
- [ ] 理解守护进程管理模式 (start/stop/status)
- [ ] 掌握 Tauri Desktop 应用架构
- [ ] 理解 IPC 命令和系统托盘集成
- [ ] 掌握自动更新机制

---

## 1. CLI 架构设计

### 文件位置
`crates/openfang-cli/src/main.rs:88-105`

```rust
/// OpenFang — the open-source Agent Operating System.
#[derive(Parser)]
#[command(
    name = "openfang",
    version,
    about = "🐍 OpenFang — Open-source Agent Operating System",
    long_about = "🐍 OpenFang — Open-source Agent Operating System\n\n\
                  Deploy, manage, and orchestrate AI agents from your terminal.\n\
                  40 channels · 60 skills · 50+ models · infinite possibilities.",
    after_help = AFTER_HELP,
)]
struct Cli {
    /// Path to config file.
    #[arg(long, global = true)]
    config: Option<PathBuf>,

    #[command(subcommand)]
    command: Option<Commands>,
}
```

**设计要点**：

| 特性 | 说明 |
|------|------|
| **Clap Parser** | 使用 `#[derive(Parser)]` 宏解析命令行参数 |
| **全局参数** | `--config` 可应用于所有子命令 |
| **子命令枚举** | `Commands` 枚举定义所有可用命令 |
| **无命令行为** | `command: Option<Commands>` 为空时启动 TUI 选择器 |

---

## 2. 核心命令分类

### 2.1 守护进程管理

| 命令 | 说明 | 关键参数 |
|------|------|----------|
| `openfang start` | 启动内核守护进程 | `--yolo` 自动批准所有工具调用 |
| `openfang stop` | 停止运行中的守护进程 | - |
| `openfang status` | 显示内核状态 | `--json` JSON 输出 |
| `openfang health` | 运行诊断健康检查 | `--json`, `--repair` |

### 2.2 Agent 管理

```rust
#[derive(Subcommand)]
enum AgentCommands {
    /// Spawn a new agent from a template (interactive or by name).
    New {
        /// Template name (e.g., "coder", "assistant"). Interactive picker if omitted.
        template: Option<String>,
    },
    /// Spawn a new agent from a manifest file.
    Spawn {
        /// Path to the agent manifest TOML file.
        manifest: PathBuf,
    },
    /// List all running agents.
    List {
        /// Output as JSON for scripting.
        #[arg(long)]
        json: bool,
    },
    /// Interactive chat with an agent.
    Chat {
        /// Agent ID (UUID).
        agent: String,
    },
    /// Kill a running agent.
    Kill {
        /// Agent ID (UUID).
        id: String,
    },
}
```

### 2.3 配置管理

| 命令 | 说明 | 示例 |
|------|------|------|
| `openfang config show` | 显示当前配置 | `openfang config show` |
| `openfang config edit` | 在编辑器中打开配置 | - |
| `openfang config get <key>` | 获取配置值 | `openfang config get default_model.provider` |
| `openfang config set <key> <value>` | 设置配置值 | `openfang config set api_listen 0.0.0.0:4200` |
| `openfang config set-key <provider>` | 保存 API Key | `openfang config set-key groq` |

### 2.4 技能与扩展

```rust
#[derive(Subcommand)]
enum SkillCommands {
    /// Install a skill from FangHub or a local directory.
    Install { source: String },
    /// List installed skills.
    List,
    /// Remove an installed skill.
    Remove { name: String },
    /// Search FangHub for skills.
    Search { query: String },
    /// Create a new skill scaffold.
    Create,
}

#[derive(Subcommand)]
enum VaultCommands {
    /// Initialize the credential vault.
    Init,
    /// Store a credential in the vault.
    Set { key: String },
    /// List all keys in the vault (values are hidden).
    List,
    /// Remove a credential from the vault.
    Remove { key: String },
}
```

---

## 3. TUI 启动器

### 文件位置
`crates/openfang-cli/src/launcher.rs:62-71`

```rust
#[derive(Clone, Copy, PartialEq, Eq)]
pub enum LauncherChoice {
    GetStarted,
    Chat,
    Dashboard,
    DesktopApp,
    TerminalUI,
    ShowHelp,
    Quit,
}
```

### 动态菜单

| 用户类型 | 菜单项 | 说明 |
|----------|--------|------|
| **首次运行** | Get started (高亮) | 配置向导、API Keys、模型选择 |
| | Chat with an agent | 快速聊天 |
| | Open dashboard | 浏览器打开 Web UI |
| | Open desktop app | 启动桌面应用 |
| | Launch terminal UI | 完整 TUI 控制台 |
| **返回用户** | Chat with an agent | 直接开始聊天 |
| | Open dashboard | 浏览器打开 Web UI |
| | Launch terminal UI | 完整 TUI 控制台 |
| | Settings | 配置管理 |

### 状态检测

```rust
// 守护进程检测
let (daemon_url, agent_count) = find_daemon();

// Provider 检测
fn detect_provider() -> Option<(&'static str, &'static str)> {
    for &(var, name) in PROVIDER_ENV_VARS {
        if std::env::var(var).is_ok() {
            return Some((name, var));
        }
    }
    None
}
```

**检测逻辑**：
1. 启动后台线程检测 `~/.openfang/config.toml` 是否存在
2. 检查环境变量中的 API Keys
3. 检测 `~/.openclaw` 目录（OpenClaw 迁移提示）
4. 尝试连接运行中的守护进程获取 agent 数量

---

## 4. 守护进程模式

### 4.1 Start 命令

**文件位置**: `crates/openfang-cli/src/main.rs`

```bash
openfang start [--yolo]
```

**执行流程**：
1. 检查单实例锁（防止重复启动）
2. 初始化 `tracing_subscriber` 日志系统
3. 加载 `KernelConfig` 从 `~/.openfang/config.toml`
4. 启动 `tokio` 运行时
5. 注册所有 channel bridges（Telegram、Slack 等）
6. 启动 background agents（心跳监控、自主 agent）
7. 绑定 API 服务器到 `0.0.0.0:4200`
8. 进入异步事件循环

### 4.2 Stop 命令

```bash
openfang stop
```

**实现**：
```rust
// 读取 PID 文件
let pid = read_pid_file()?;

// 发送 SIGTERM (Windows 使用 GenerateConsoleCtrlEvent)
send_ctrl_event(pid)?;

// 等待进程退出
wait_for_exit(pid, Duration::from_secs(10))?;

// 清理 PID 文件
remove_pid_file()?;
```

### 4.3 PID 文件管理

**位置**: `~/.openfang/daemon.pid`

```toml
# 文件格式
PID=12345
STARTED=2026-03-15T10:30:00Z
VERSION=0.4.4
```

**用途**：
- 防止重复启动
- `stop` 命令定位进程
- `status` 命令检查运行状态

---

## 5. Chat 命令

### 5.1 快速聊天

```bash
# 与默认 agent 聊天
openfang chat

# 与指定 agent 聊天
openfang chat coder-agent

# 直接发送消息
openfang message agent-id "Hello, help me write a function"
```

### 5.2 交互式聊天

**文件位置**: `crates/openfang-cli/src/tui/screens/chat.rs`

```rust
// 聊天循环
loop {
    // 1. 读取用户输入
    let input = read_line()?;

    // 2. 发送到 API /api/agents/{id}/message
    let response = client.post(&url)
        .json(&json!({"message": input}))
        .send()
        .await?;

    // 3. 流式显示响应
    while let Some(event) = stream.next().await {
        match event {
            StreamEvent::TextDelta { text } => print!("{}", text),
            StreamEvent::ToolUseStart { name } => show_tool_indicator(&name),
            StreamEvent::ToolExecutionResult { name, is_error } => {
                show_tool_result(&name, is_error)
            }
            _ => {}
        }
    }
}
```

---

## 6. TUI Dashboard

### 6.1 屏幕架构

**文件位置**: `crates/openfang-cli/src/tui/screens/mod.rs`

```
crates/openfang-cli/src/tui/screens/
├── dashboard.rs      # 主仪表盘
├── agents.rs         # Agent 列表和管理
├── chat.rs           # 聊天界面
├── memory.rs         # 记忆浏览和搜索
├── sessions.rs       # 会话历史
├── workflows.rs      # 工作流管理
├── hands.rs          # Hands 系统
├── channels.rs       # Channel 状态
├── extensions.rs     # MCP 集成
├── skills.rs         # 技能市场
├── security.rs       # 安全审计
├── usage.rs          # 使用和预算统计
├── logs.rs           # 日志浏览器
├── settings.rs       # 配置编辑
└── init_wizard.rs    # 首次运行向导
```

### 6.2 主仪表盘

**文件位置**: `crates/openfang-cli/src/tui/screens/dashboard.rs`

```rust
// 仪表盘布局
╭──────────────────────────────────────────────────────────────╮
│  🐍 OpenFang v0.4.4                      ● Daemon: Running  │
├──────────────────────────────────────────────────────────────┤
│  📊 Overview                                                 │
│  ├─ Agents: 5 running    │  │  Memory: 1,234 items          │
│  ├─ Sessions: 42 active  │  │  Tools: 89 registered         │
│  ╰─ Channels: 3 connected│  │  Budget: $12.50 / $100.00     │
├──────────────────────────────────────────────────────────────┤
│  ⚡ Recent Activity                                          │
│  │ ● coder-agent completed web_search "Rust async"          │
│  │ ● assistant created memory "API endpoint design"         │
│  │ ● telegram-bridge received message from user123          │
╰──────────────────────────────────────────────────────────────╯
```

### 6.3 主题系统

**文件位置**: `crates/openfang-cli/src/tui/theme.rs`

```rust
pub const ACCENT: Color = Color::Rgb(114, 171, 255);    // 蓝色
pub const GREEN: Color = Color::Rgb(86, 182, 194);      // 青色
pub const YELLOW: Color = Color::Rgb(229, 192, 123);    // 黄色
pub const RED: Color = Color::Rgb(204, 102, 102);       // 红色
pub const BG_PRIMARY: Color = Color::Rgb(30, 30, 46);   // 深色背景
pub const TEXT_PRIMARY: Color = Color::Rgb(219, 219, 226);
pub const TEXT_SECONDARY: Color = Color::Rgb(139, 144, 159);
pub const TEXT_TERTIARY: Color = Color::Rgb(76, 81, 96);
```

---

## 7. Desktop 应用架构

### 7.1 嵌入式服务器模式

**文件位置**: `crates/openfang-desktop/src/lib.rs`

```
┌─────────────────────────────────────────────────────────┐
│                  Tauri 2.0 Process                       │
│                                                          │
│  ┌─────────────┐    ┌────────────────────────────────┐  │
│  │ Main Thread │    │ Background Thread              │  │
│  │             │    │ ("openfang-server")            │  │
│  │ WebView     │    │                                │  │
│  │ Window      │───>│ tokio runtime                  │  │
│  │ (main)      │    │ axum API server                │  │
│  │             │    │ channel bridges                │  │
│  │ System Tray │    │ background agents              │  │
│  └─────────────┘    │                                │  │
│                     │ OpenFang Kernel                │  │
│                     └────────────────────────────────┘  │
│                          │                               │
│                     http://127.0.0.1:{port}             │
└─────────────────────────────────────────────────────────┘
```

### 7.2 启动序列

**文件位置**: `crates/openfang-desktop/src/lib.rs:71-156`

```rust
pub fn run() -> tauri::Result<()> {
    // 1. 初始化 tracing
    tracing_subscriber::fmt()
        .with_env_filter(
            EnvFilter::from_default_env()
                .add_directive("openfang=info".parse()?)
                .add_directive("tauri=info".parse()?),
        )
        .init();

    // 2. 启动 Kernel
    let kernel = Arc::new(OpenFangKernel::boot(None)?);
    kernel.set_self_handle();

    // 3. 绑定端口（OS 自动分配）
    let listener = std::net::TcpListener::bind("127.0.0.1:0")?;
    let port = listener.local_addr()?.port();

    // 4. 启动服务器线程
    let server_handle = ServerHandle::start(listener, Arc::clone(&kernel))?;

    // 5. 构建 Tauri 应用
    let mut builder = tauri::Builder::default();

    // 6. 添加插件
    builder = builder
        .plugin(tauri_plugin_notification::init())
        .plugin(tauri_plugin_shell::init())
        .plugin(tauri_plugin_dialog::init())
        .plugin(tauri_plugin_autostart::init(
            AutostartOptions::default(),
        ));

    // 7. 注册 IPC 命令
    builder = builder
        .invoke_handler(tauri::generate_handler![
            commands::get_port,
            commands::get_status,
            commands::get_agent_count,
            commands::import_agent_toml,
            commands::import_skill_file,
            commands::get_autostart,
            commands::set_autostart,
            commands::check_for_updates,
            commands::install_update,
            commands::open_config_dir,
            commands::open_logs_dir,
        ]);

    // 8. 创建主窗口
    let app = builder
        .manage(PortState(port))
        .manage(KernelState {
            kernel: Arc::clone(&kernel),
            started_at: Instant::now(),
        })
        .setup(|app| {
            let window = tauri::window::WindowBuilder::new(
                app,
                "main",
                WebviewUrl::External(format!("http://127.0.0.1:{port}").parse()?),
            )
            .title("OpenFang")
            .inner_size(1280.0, 800.0)
            .min_inner_size(800.0, 600.0)
            .center()
            .build()?;
            Ok(())
        })
        .on_window_event(|window, event| {
            if let tauri::WindowEvent::CloseRequested { api, .. } = event {
                let _ = window.hide();
                api.prevent_close();  // 隐藏到托盘而非退出
            }
        })
        .build(tauri::generate_context!())?;

    // 9. 运行事件循环
    app.run(|_, _| {});
}
```

---

## 8. IPC 命令系统

### 8.1 命令列表

| 命令 | 参数 | 返回值 | 说明 |
|------|------|--------|------|
| `get_port` | - | `u16` | 获取 API 服务器端口 |
| `get_status` | - | `{"status", "port", "agents", "uptime_secs"}` | 获取运行时状态 |
| `get_agent_count` | - | `usize` | Agent 数量 |
| `import_agent_toml` | - | `String` (agent name) | 导入 agent 清单 |
| `import_skill_file` | - | `String` (file name) | 导入技能文件 |
| `get_autostart` | - | `bool` | 检查开机自启 |
| `set_autostart` | `enabled: bool` | `bool` | 切换开机自启 |
| `check_for_updates` | - | `UpdateInfo` | 检查更新 |
| `install_update` | - | `()` (重启) | 安装更新 |
| `open_config_dir` | - | `()` | 打开配置目录 |
| `open_logs_dir` | - | `()` | 打开日志目录 |

### 8.2 前端调用示例

```typescript
// 获取端口
const port = await invoke("get_port");
console.log(`API at http://127.0.0.1:${port}`);

// 获取状态
const status = await invoke("get_status");
console.log(`${status.agents} agents running for ${status.uptime_secs}s`);

// 导入 Agent
try {
    const agentName = await invoke("import_agent_toml");
    console.log(`Spawned agent: ${agentName}`);
} catch (e) {
    console.error(`Import failed: ${e}`);
}

// 自动更新
const update = await invoke("check_for_updates");
if (update.available) {
    console.log(`Update ${update.version} available: ${update.body}`);
    await invoke("install_update");  // 应用重启
}
```

---

## 9. 系统托盘集成

### 9.1 托盘菜单

**文件位置**: `crates/openfang-desktop/src/tray.rs`

```
╭─────────────────────────────────────╮
│  🐍 OpenFang Agent OS               │
├─────────────────────────────────────┤
│  👁 Show Window                     │
│  🌐 Open in Browser                 │
├─────────────────────────────────────┤
│  Agents: 5 running         [info]   │
│  Status: Running (1h 23m)  [info]   │
├─────────────────────────────────────┤
│  ☑ Launch at Login                 │
│  📦 Check for Updates...            │
├─────────────────────────────────────┤
│  📁 Open Config Directory           │
│  🚪 Quit OpenFang                   │
╰─────────────────────────────────────╯
```

### 9.2 事件处理

```rust
fn on_tray_icon_event(
    tray: &tray::TrayIcon,
    event: tray::TrayIconEvent,
) {
    if let TrayIconEvent::Click {
        button: MouseButton::Left,
        button_state: MouseButtonState::Up,
        ..
    } = event {
        // 左键点击：显示窗口
        if let Some(window) = app.get_webview_window("main") {
            let _ = window.show();
            let _ = window.unminimize();
            let _ = window.set_focus();
        }
    }
}
```

---

## 10. 自动更新机制

### 10.1 更新流程

**文件位置**: `crates/openfang-desktop/src/updater.rs`

```
1. 应用启动 → 10 秒延迟
              ↓
2. check_for_update() → GitHub Releases/latest.json
              ↓
3. 可用更新？──否──→ 静默
              │
             是
              ↓
4. 发送通知："更新可用 v0.4.4"
              ↓
5. download_and_install_update()
              ↓
6. 验证签名 (Ed25519)
              ↓
7. 安装更新 → 重启应用
```

### 10.2 配置 (tauri.conf.json)

```json
{
  "plugins": {
    "updater": {
      "pubkey": "dW50cnVzdGVkIGNvbW1lbnQ6IG1pbmlzaWdu...",
      "endpoints": [
        "https://github.com/RightNow-AI/openfang/releases/latest/download/latest.json"
      ],
      "windows": {
        "installMode": "passive"
      }
    }
  }
}
```

### 10.3 签名密钥

**生成**：
```bash
cargo install tauri-cli
cargo tauri init  # 生成密钥对
```

**输出**：
- `~/.tauri/signing.pem` - 私钥（上传到 GitHub Secrets）
- `tauri.conf.json` - 公钥（已嵌入）

**CI/CD**：
```yaml
- name: Build Tauri
  uses: tauri-apps/tauri-action@v0
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    TAURI_SIGNING_PRIVATE_KEY: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY }}
```

---

## 11. 单实例与隐藏到托盘

### 11.1 单实例保护

**文件位置**: `crates/openfang-desktop/src/lib.rs`

```rust
#[cfg(desktop)]
{
    builder = builder.plugin(tauri_plugin_single_instance::init(
        |app, _args, _cwd| {
            // 第二实例启动时，聚焦现有窗口
            if let Some(w) = app.get_webview_window("main") {
                let _ = w.show();
                let _ = w.unminimize();
                let _ = w.set_focus();
            }
        },
    ));
}
```

### 11.2 关闭行为

```rust
.on_window_event(|window, event| {
    #[cfg(desktop)]
    if let tauri::WindowEvent::CloseRequested { api, .. } = event {
        let _ = window.hide();  // 隐藏而非退出
        api.prevent_close();    // 阻止关闭
    }
})
```

**真正退出方式**：系统托盘 → "Quit OpenFang"

---

## 12. 本地通知

### 12.1 事件订阅

**文件位置**: `crates/openfang-desktop/src/lib.rs`

```rust
// 订阅内核事件总线
let mut rx = kernel.subscribe_to_events();

// 转发为本地通知
tauri::async_runtime::spawn(async move {
    while let Ok(event) = rx.recv().await {
        match event {
            KernelEvent::Lifecycle(LifecycleEvent::Crashed { id, error }) => {
                let _ = app.notification()
                    .builder()
                    .title("Agent Crashed")
                    .body(format!("Agent {} crashed: {}", id, error))
                    .show();
            }
            KernelEvent::Lifecycle(LifecycleEvent::Spawned { name }) => {
                let _ = app.notification()
                    .builder()
                    .title("Agent Started")
                    .body(format!("Agent \"{}\" is now running", name))
                    .show();
            }
            KernelEvent::System(HealthCheckFailed { id, secs }) => {
                let _ = app.notification()
                    .builder()
                    .title("Health Check Failed")
                    .body(format!("Agent {} unresponsive for {}s", id, secs))
                    .show();
            }
            _ => {}  // 忽略其他事件
        }
    }
});
```

### 12.2 通知类型

| 事件 | 标题 | 正文 |
|------|------|------|
| `LifecycleEvent::Crashed` | "Agent Crashed" | `Agent {id} crashed: {error}` |
| `LifecycleEvent::Spawned` | "Agent Started" | `Agent "{name}" is now running` |
| `HealthCheckFailed` | "Health Check Failed" | `Agent {id} unresponsive for {secs}s` |
| `UpdateAvailable` | "Update Available" | `Version {version} ready to install` |
| `UpdateInstalled` | "Update Installed" | `Restarting to apply update...` |

---

## 13. 构建与分发

### 13.1 开发模式

```bash
cd crates/openfang-desktop
cargo tauri dev
```

**特性**：
- 热重载支持
- 控制台窗口可见（调试输出）
- 使用本地 API 服务器

### 13.2 生产构建

```bash
cd crates/openfang-desktop
cargo tauri build
```

**输出**：

| 平台 | 产物 |
|------|------|
| **Windows** | `.msi` (Windows Installer), `.exe` (NSIS) |
| **macOS** | `.dmg`, `.app` bundle |
| **Linux** | `.deb`, `.rpm`, `.AppImage` |

**发布二进制**：
```
crates/openfang-desktop/target/release/
├── openfang-desktop      # Linux
├── openfang-desktop.app  # macOS
└── openfang-desktop.exe  # Windows
```

### 13.3 Windows 控制台隐藏

**文件位置**: `crates/openfang-desktop/src/main.rs`

```rust
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]
```

**效果**：
- Debug 构建：显示控制台窗口（便于调试）
- Release 构建：无控制台窗口（原生应用体验）

---

## 14. Tauri 插件

### 14.1 插件列表

| 插件 | 版本 | 用途 |
|------|------|------|
| `tauri-plugin-notification` | 2 | 本地 OS 通知 |
| `tauri-plugin-shell` | 2 | Shell/进程访问 |
| `tauri-plugin-dialog` | 2 | 文件选择器 |
| `tauri-plugin-single-instance` | 2 | 单实例保护 |
| `tauri-plugin-autostart` | 2 | 开机自启 |
| `tauri-plugin-updater` | 2 | 自动更新 |
| `tauri-plugin-global-shortcut` | 2 | 全局快捷键 |

### 14.2 全局快捷键

**注册**：
```rust
app.handle().plugin(
    tauri_plugin_global_shortcut::Builder::new()
        .with_handler(|app, shortcut, _event| {
            match shortcut {
                // Ctrl+Shift+O: 显示/隐藏窗口
                Shortcut::O => {
                    if let Some(w) = app.get_webview_window("main") {
                        if w.is_visible().unwrap_or(false) {
                            let _ = w.hide();
                        } else {
                            let _ = w.show();
                            let _ = w.set_focus();
                        }
                    }
                }
                // Ctrl+Shift+N: 新建聊天
                Shortcut::N => {
                    let _ = app.emit("new-chat", ());
                }
                // Ctrl+Shift+C: 复制最后一条消息
                Shortcut::C => {
                    let _ = app.emit("copy-last", ());
                }
                _ => {}
            }
        })
        .shortcut("Ctrl+Shift+O")?
        .shortcut("Ctrl+Shift+N")?
        .shortcut("Ctrl+Shift+C")?
        .build(app),
);
```

---

## 15. CSP 与安全

### 15.1 Content Security Policy

**文件位置**: `tauri.conf.json`

```
default-src 'self' http://127.0.0.1:* ws://127.0.0.1:*
    https://fonts.googleapis.com https://fonts.gstatic.com;
img-src 'self' data: blob: http://127.0.0.1:*;
style-src 'self' 'unsafe-inline'
    https://fonts.googleapis.com https://fonts.gstatic.com;
script-src 'self' 'unsafe-inline' 'unsafe-eval';
font-src 'self' https://fonts.gstatic.com;
connect-src 'self' http://127.0.0.1:* ws://127.0.0.1:*;
media-src 'self' blob: http://127.0.0.1:*;
frame-src 'self' blob: http://127.0.0.1:*;
object-src 'none';
base-uri 'self';
form-action 'self'
```

**说明**：
- 只允许加载本地 `127.0.0.1` 的资源
- 允许 Google Fonts（用于 UI 字体）
- 允许 `data:` 和 `blob:` URLs（用于图片/媒体）
- `object-src 'none'` 阻止 Flash 等插件
- `form-action 'self'` 限制表单提交目标

### 15.2 axum 安全头

**文件位置**: `crates/openfang-api/src/server.rs`

```rust
let middleware_stack = ServiceBuilder::new()
    .layer(
        SetResponseHeaderLayer::overriding(
            header::CONTENT_SECURITY_POLICY,
            HeaderValue::from_static("default-src 'self'"),
        ),
    )
    .layer(
        SetResponseHeaderLayer::overriding(
            header::X_FRAME_OPTIONS,
            HeaderValue::from_static("DENY"),
        ),
    )
    .layer(
        SetResponseHeaderLayer::overriding(
            header::X_CONTENT_TYPE_OPTIONS,
            HeaderValue::from_static("nosniff"),
        ),
    )
    .layer(
        SetResponseHeaderLayer::overriding(
            header::STRICT_TRANSPORT_SECURITY,
            HeaderValue::from_static("max-age=63072000"),
        ),
    );
```

---

## 16. 文件结构

### 16.1 CLI

```
crates/openfang-cli/
├── Cargo.toml
├── src/
│   ├── main.rs           # 入口、命令定义
│   ├── launcher.rs       # TUI 启动器
│   ├── ui.rs             # 终端 UI 输出
│   ├── table.rs          # 表格渲染
│   ├── progress.rs       # 进度条
│   ├── dotenv.rs         # .env 解析
│   ├── mcp.rs            # MCP helpers
│   ├── templates.rs      # 模板渲染
│   ├── bundled_agents.rs # 内置 agent 清单
│   └── tui/
│       ├── mod.rs        # TUI 框架
│       ├── event.rs      # 事件处理
│       ├── theme.rs      # 颜色主题
│       ├── chat_runner.rs# 聊天运行器
│       └── screens/
│           ├── mod.rs
│           ├── dashboard.rs
│           ├── agents.rs
│           ├── chat.rs
│           ├── memory.rs
│           ├── sessions.rs
│           ├── workflows.rs
│           ├── hands.rs
│           ├── channels.rs
│           ├── extensions.rs
│           ├── skills.rs
│           ├── security.rs
│           ├── usage.rs
│           ├── logs.rs
│           ├── settings.rs
│           ├── init_wizard.rs
│           └── welcome.rs
```

### 16.2 Desktop

```
crates/openfang-desktop/
├── Cargo.toml
├── tauri.conf.json
├── build.rs
├── capabilities/
│   └── default.json      # 权限配置
├── gen/
│   └── schemas/          # 自动生成
├── icons/
│   ├── icon.ico
│   ├── icon.png
│   ├── 32x32.png
│   ├── 128x128.png
│   └── 128x128@2x.png
└── src/
    ├── main.rs           # 二进制入口
    ├── lib.rs            # Tauri 构建器
    ├── commands.rs       # IPC 命令
    ├── server.rs         # ServerHandle
    ├── tray.rs           # 系统托盘
    ├── shortcuts.rs      # 全局快捷键
    └── updater.rs        # 自动更新
```

---

## 17. 环境变量

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `RUST_LOG` | 日志级别 | `openfang=info,tauri=info` |
| `OPENFANG_HOME` | 数据目录 | `~/.openfang/` |
| `ANTHROPIC_API_KEY` | Anthropic Key | - |
| `OPENAI_API_KEY` | OpenAI Key | - |
| `GROQ_API_KEY` | Groq Key | - |
| `GEMINI_API_KEY` | Google Key | - |
| `DEEPSEEK_API_KEY` | DeepSeek Key | - |

---

## 18. PWA 离线支持 (v0.4.9 新增)

### 18.1 manifest.json

**文件位置**: `crates/openfang-api/static/manifest.json`

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
  ],
  "categories": ["productivity", "utilities"],
  "shortcuts": [
    {
      "name": "Chat",
      "url": "/?action=chat",
      "description": "Start a new chat"
    },
    {
      "name": "Agents",
      "url": "/?view=agents",
      "description": "View all agents"
    }
  ]
}
```

### 18.2 Service Worker (sw.js)

**文件位置**: `crates/openfang-api/static/sw.js`

```javascript
// Service Worker 缓存关键资源
const CACHE_NAME = 'openfang-v1';
const ASSETS = [
  '/',
  '/index.html',
  '/js/i18n.js',
  '/js/app.js',
  '/js/pages/agents.js',
  '/js/pages/chat.js',
  '/css/styles.css'
];

// 安装时缓存
self.addEventListener('install', (e) => {
  e.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.addAll(ASSETS);
    })
  );
});

// 激活时清理旧缓存
self.addEventListener('activate', (e) => {
  e.waitUntil(
    caches.keys().then((keys) => {
      return Promise.all(
        keys.filter((k) => k !== CACHE_NAME).map((k) => caches.delete(k))
      );
    })
  );
});

// 拦截请求，缓存优先
self.addEventListener('fetch', (e) => {
  e.respondWith(
    caches.match(e.request).then((cached) => {
      return cached || fetch(e.request);
    })
  );
});
```

### 18.3 HTML 引用 (index_body.html)

**文件位置**: `crates/openfang-api/static/index_body.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>OpenFang Dashboard</title>

  <!-- PWA Manifest -->
  <link rel="manifest" href="/manifest.json">
  <meta name="theme-color" content="#10b981">

  <!-- Apple PWA 支持 -->
  <meta name="apple-mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
  <link rel="apple-touch-icon" href="/icon-192.png">
</head>
<body>
  <!-- 应用内容 -->

  <script>
    // 注册 Service Worker
    if ('serviceWorker' in navigator) {
      window.addEventListener('load', () => {
        navigator.serviceWorker.register('/sw.js')
          .then((registration) => {
            console.log('SW registered:', registration.scope);
          })
          .catch((error) => {
            console.log('SW registration failed:', error);
          });
      });
    }
  </script>
</body>
</html>
```

### 18.4 离线能力

| 功能 | 在线 | 离线 |
|------|------|------|
| Dashboard UI | ✅ | ✅ (缓存) |
| 查看 Agents 列表 | ✅ | ✅ (缓存) |
| 发送消息 | ✅ | ❌ (队列等待) |
| 查看历史消息 | ✅ | ✅ (IndexedDB) |
| 切换语言 | ✅ | ✅ (缓存) |

### 18.5 安装 PWA

**桌面端**:
- Chrome/Edge: 地址栏右侧出现"安装"图标
- Firefox: 右键菜单"将此站点作为应用安装"

**移动端**:
- iOS Safari: 分享 → 添加到主屏幕
- Android Chrome: 分享 → 安装应用

---

## 19. Desktop 与 PWA 对比

| 特性 | Desktop App | PWA |
|------|-------------|-----|
| **安装方式** | .msi/.dmg/.deb | 浏览器安装 |
| **更新机制** | 自动更新插件 | Service Worker |
| **系统集成** | 托盘、快捷键、通知 | 有限通知 |
| **离线能力** | ✅ (嵌入服务器) | ✅ (缓存) |
| **体积** | ~50MB | ~1MB |
| **跨平台** | ✅ | ✅ |

---

## 完成检查清单

- [ ] 掌握 CLI 命令结构和子命令设计
- [ ] 理解守护进程管理模式 (start/stop/status)
- [ ] 掌握 Tauri Desktop 应用架构
- [ ] 理解 IPC 命令和系统托盘集成
- [ ] 掌握自动更新机制
- [ ] 了解 PWA 离线支持 (v0.4.9 新增)

---

## 总结

本节完成了 OpenFang 的两种用户界面形态：

| 特性 | CLI | Desktop App |
|------|-----|-------------|
| **入口** | `openfang` | `openfang-desktop` |
| **交互** | 命令行/TUI | WebView + 原生窗口 |
| **进程模型** | 守护进程 + 客户端 | 嵌入式服务器 |
| **通知** | 终端输出 | 本地 OS 通知 |
| **托盘** | ❌ | ✅ |
| **自动更新** | ❌ | ✅ |
| **单实例** | PID 文件 | `tauri-plugin-single-instance` |
| **快捷键** | ❌ | ✅ (全局) |

**推荐使用场景**：
- **CLI**：服务器部署、CI/CD、脚本自动化
- **Desktop**：个人开发、日常管理、可视化监控

---

*创建时间：2026-03-15 (更新于 2026-03-19 v0.4.9)*
*OpenFang v0.4.9*

恭喜！您已经完成了 OpenFang 25 节完整教程系列。

现在您应该已经掌握了：
- ✅ 14 Crates 的架构设计
- ✅ Agent 运行时核心机制
- ✅ LLM Driver 抽象与实现
- ✅ 工具执行与安全系统
- ✅ 记忆与存储架构
- ✅ Hands 自主代理系统
- ✅ Channel 通信与协议
- ✅ MCP 与 A2A 集成
- ✅ API 服务器开发
- ✅ CLI 与 Desktop 应用构建
- ✅ PWA 离线支持 (v0.4.9 新增)

---

*OpenFang v0.4.9 — 25 节完整教程系列*
