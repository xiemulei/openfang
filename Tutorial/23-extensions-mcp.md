# 第 23 节：Extensions 系统 — MCP 集成

> **版本**: v0.4.4 (2026-03-15)
> **核心文件**: `crates/openfang-extensions/`, `crates/openfang-runtime/src/mcp.rs`

## 学习目标

- [ ] 理解 Extensions 系统架构和 25 个 MCP 集成模板
- [ ] 掌握凭证保险箱 (Credential Vault) 的 AES-256-GCM 加密机制
- [ ] 理解 OAuth2 PKCE 流程的实现细节
- [ ] 掌握健康监控与自动重连机制

---

## 1. Extensions 系统概述

### 1.1 系统架构

OpenFang Extensions 系统提供了一站式的 MCP (Model Context Protocol) 集成能力，使用户能够通过 `openfang add <name>` 命令快速连接外部服务。

```
┌─────────────────────────────────────────────────────────────────┐
│                    Extensions System                            │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  Bundled    │  │  Registry   │  │  Installer  │             │
│  │ Integrations│  │             │  │             │             │
│  │  (25 TOML)  │  │             │  │             │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   Vault     │  │   OAuth     │  │   Health    │             │
│  │  (AES-256)  │  │  (PKCE)     │  │  Monitor    │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    MCP Client Runtime                           │
│         (crates/openfang-runtime/src/mcp.rs)                    │
└─────────────────────────────────────────────────────────────────┘
```

### 核心模块

| 模块 | 文件 | 职责 |
|------|------|------|
| **bundled** | `bundled.rs` | 25 个编译时嵌入的 TOML 集成模板 |
| **registry** | `registry.rs` | 集成注册表，管理安装状态 |
| **installer** | `installer.rs` | 一键安装流程 (`openfang add`) |
| **vault** | `vault.rs` | AES-256-GCM 加密凭证存储 |
| **credentials** | `credentials.rs` | 凭证解析链 (Vault → .env → Env → Interactive) |
| **oauth** | `oauth.rs` | OAuth2 PKCE 流程实现 |
| **health** | `health.rs` | 健康监控与自动重连 |

---

## 2. 25 个 MCP 集成模板

### 2.1 分类概览

25 个集成模板按类别分组，全部编译时嵌入到二进制中：

```rust
// crates/openfang-extensions/src/bundled.rs:7-64
pub fn bundled_integrations() -> Vec<(&'static str, &'static str)> {
    vec![
        // ── DevTools (6) ────────────────────────────────────────────────────
        ("github", include_str!("../integrations/github.toml")),
        ("gitlab", include_str!("../integrations/gitlab.toml")),
        ("linear", include_str!("../integrations/linear.toml")),
        ("jira", include_str!("../integrations/jira.toml")),
        ("bitbucket", include_str!("../integrations/bitbucket.toml")),
        ("sentry", include_str!("../integrations/sentry.toml")),
        // ── Productivity (6) ────────────────────────────────────────────────
        ("google-calendar", include_str!("../integrations/google-calendar.toml")),
        ("gmail", include_str!("../integrations/gmail.toml")),
        ("notion", include_str!("../integrations/notion.toml")),
        ("todoist", include_str!("../integrations/todoist.toml")),
        ("google-drive", include_str!("../integrations/google-drive.toml")),
        ("dropbox", include_str!("../integrations/dropbox.toml")),
        // ── Communication (3) ───────────────────────────────────────────────
        ("slack", include_str!("../integrations/slack.toml")),
        ("discord-mcp", include_str!("../integrations/discord-mcp.toml")),
        ("teams-mcp", include_str!("../integrations/teams-mcp.toml")),
        // ── Data (5) ────────────────────────────────────────────────────────
        ("postgresql", include_str!("../integrations/postgresql.toml")),
        ("sqlite-mcp", include_str!("../integrations/sqlite-mcp.toml")),
        ("mongodb", include_str!("../integrations/mongodb.toml")),
        ("redis", include_str!("../integrations/redis.toml")),
        ("elasticsearch", include_str!("../integrations/elasticsearch.toml")),
        // ── Cloud (3) ───────────────────────────────────────────────────────
        ("aws", include_str!("../integrations/aws.toml")),
        ("gcp-mcp", include_str!("../integrations/gcp-mcp.toml")),
        ("azure-mcp", include_str!("../integrations/azure-mcp.toml")),
        // ── AI & Search (2) ─────────────────────────────────────────────────
        ("brave-search", include_str!("../integrations/brave-search.toml")),
        ("exa-search", include_str!("../integrations/exa-search.toml")),
    ]
}
```

### 2.2 类别分布

| 类别 | 数量 | 集成列表 |
|------|------|----------|
| **DevTools** | 6 | GitHub, GitLab, Linear, Jira, Bitbucket, Sentry |
| **Productivity** | 6 | Google Calendar, Gmail, Notion, Todoist, Google Drive, Dropbox |
| **Communication** | 3 | Slack, Discord MCP, Teams MCP |
| **Data** | 5 | PostgreSQL, SQLite MCP, MongoDB, Redis, Elasticsearch |
| **Cloud** | 3 | AWS, GCP MCP, Azure MCP |
| **AI & Search** | 2 | Brave Search, Exa Search |

### 2.3 模板结构

每个 TOML 模板包含完整的集成元数据：

```toml
# crates/openfang-extensions/integrations/github.toml
id = "github"
name = "GitHub"
description = "Access GitHub repositories, issues, pull requests, and organizations through the official MCP server"
category = "devtools"
icon = "🐙"
tags = ["git", "vcs", "code", "issues", "pull-requests", "ci"]

[transport]
type = "stdio"
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]

[[required_env]]
name = "GITHUB_PERSONAL_ACCESS_TOKEN"
label = "GitHub Personal Access Token"
help = "A fine-grained or classic PAT with repo and read:org scopes"
is_secret = true
get_url = "https://github.com/settings/tokens"

[oauth]
provider = "github"
scopes = ["repo", "read:org"]
auth_url = "https://github.com/login/oauth/authorize"
token_url = "https://github.com/login/oauth/access_token"

[health_check]
interval_secs = 60
unhealthy_threshold = 3

setup_instructions = """
1. Go to https://github.com/settings/tokens and create a Personal Access Token (classic or fine-grained) with 'repo' and 'read:org' scopes.
2. Paste the token into the GITHUB_PERSONAL_ACCESS_TOKEN field above.
3. Alternatively, use the OAuth flow to authorize OpenFang directly with your GitHub account.
"""
```

### 2.4 IntegrationTemplate 结构

```rust
// crates/openfang-extensions/src/lib.rs:147-177
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct IntegrationTemplate {
    /// 唯一标识符 (如 "github")
    pub id: String,
    /// 人类可读名称 (如 "GitHub")
    pub name: String,
    /// 简短描述
    pub description: String,
    /// 类别 (用于浏览)
    pub category: IntegrationCategory,
    /// 图标 (emoji)
    #[serde(default)]
    pub icon: String,
    /// MCP 传输配置
    pub transport: McpTransportTemplate,
    /// 所需凭证列表
    #[serde(default)]
    pub required_env: Vec<RequiredEnvVar>,
    /// OAuth 配置 (None = 仅 API key)
    #[serde(default)]
    pub oauth: Option<OAuthTemplate>,
    /// 可搜索标签
    #[serde(default)]
    pub tags: Vec<String>,
    /// 安装说明 (在 TUI 详情视图中显示)
    #[serde(default)]
    pub setup_instructions: String,
    /// 健康检查配置
    #[serde(default)]
    pub health_check: HealthCheckConfig,
}
```

---

## 3. 凭证保险箱 (Credential Vault)

### 3.1 加密架构

CredentialVault 使用 AES-256-GCM 加密存储在 `~/.openfang/vault.enc` 文件中：

```rust
// crates/openfang-extensions/src/vault.rs:55-65
pub struct CredentialVault {
    /// vault.enc 文件路径
    path: PathBuf,
    /// 解密后的条目 (零化保护)
    entries: HashMap<String, Zeroizing<String>>,
    /// 是否已解锁
    unlocked: bool,
    /// 缓存的主密钥 (零化保护)
    cached_key: Option<Zeroizing<[u8; 32]>>,
}
```

### 3.2 密钥派生流程

```
用户密码/随机密钥
       │
       ▼
┌──────────────────┐
│   Argon2id KDF   │ ← 随机 Salt (16 字节)
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  AES-256-GCM     │ ← 随机 Nonce (12 字节)
└──────────────────┘
       │
       ▼
  vault.enc (加密文件)
```

### 3.3 密钥存储机制

**主密钥来源优先级**：

1. **OS Keyring** (首选)
   - Windows: Windows Credential Manager
   - macOS: Keychain
   - Linux: Secret Service

2. **环境变量** (无头/CI 环境)
   ```bash
   export OPENFANG_VAULT_KEY=<base64 编码的 32 字节密钥>
   ```

3. **密钥派生代码**：
```rust
// crates/openfang-extensions/src/vault.rs:400-407
fn derive_key(master_key: &[u8; 32], salt: &[u8]) -> ExtensionResult<Zeroizing<[u8; 32]>> {
    let mut derived = Zeroizing::new([0u8; 32]);
    Argon2::default()
        .hash_password_into(master_key, salt, derived.as_mut())
        .map_err(|e| ExtensionError::Vault(format!("Key derivation failed: {e}")))?;
    Ok(derived)
}
```

### 3.4 文件加密流程

```rust
// crates/openfang-extensions/src/vault.rs:265-320
fn save(&self, master_key: &[u8; 32]) -> ExtensionResult<()> {
    // 1. 序列化条目为 JSON
    let plaintext = serde_json::to_vec(&vault_data)...;

    // 2. 生成随机 salt 和 nonce
    let mut salt = [0u8; 16];
    let mut nonce_bytes = [0u8; 12];
    OsRng.fill_bytes(&mut salt);
    OsRng.fill_bytes(&mut nonce_bytes);

    // 3. Argon2 派生密钥
    let derived_key = derive_key(master_key, &salt)?;

    // 4. AES-256-GCM 加密
    let cipher = Aes256Gcm::new_from_slice(derived_key.as_ref())?;
    let ciphertext = cipher.encrypt(nonce, plaintext.as_slice())?;

    // 5. 写入文件 (带 OFV1 魔数标记)
    let vault_file = VaultFile { version: 1, salt, nonce, ciphertext };
    let mut output = Vec::with_capacity(...);
    output.extend_from_slice(b"OFV1");  // 魔数标记
    output.extend_from_slice(content.as_bytes());
    std::fs::write(&self.path, output)?;
}
```

### 3.5 安全特性

| 特性 | 实现 |
|------|------|
| **内存零化** | 使用 `Zeroizing<T>` 包装敏感数据，drop 时自动清零 |
| **机器绑定** | Keyring 使用 SHA-256(username + hostname) 进行混淆 |
| **版本标记** | OFV1 魔数用于文件格式版本检测 |
| **向后兼容** | 支持无魔数的旧版 JSON 保险箱文件 |

---

## 4. 凭证解析链 (Credential Resolver)

### 4.1 解析优先级

CredentialResolver 按以下顺序尝试获取凭证：

```
1. Encrypted Vault (vault.enc)  ← 最优先
         │
         ▼
2. Dotenv File (~/.openfang/.env)
         │
         ▼
3. Environment Variable (进程环境变量)
         │
         ▼
4. Interactive Prompt (CLI 交互模式，最后手段)
```

### 4.2 解析器实现

```rust
// crates/openfang-extensions/src/credentials.rs:47-79
pub fn resolve(&self, key: &str) -> Option<Zeroizing<String>> {
    // 1. Vault
    if let Some(ref vault) = self.vault {
        if vault.is_unlocked() {
            if let Some(val) = vault.get(key) {
                debug!("Credential '{}' resolved from vault", key);
                return Some(val);
            }
        }
    }

    // 2. Dotenv
    if let Some(val) = self.dotenv.get(key) {
        debug!("Credential '{}' resolved from .env", key);
        return Some(Zeroizing::new(val.clone()));
    }

    // 3. Environment
    if let Ok(val) = std::env::var(key) {
        debug!("Credential '{}' resolved from env var", key);
        return Some(Zeroizing::new(val));
    }

    // 4. Interactive (CLI only)
    if self.interactive {
        if let Some(val) = prompt_secret(key) {
            return Some(val);
        }
    }

    None
}
```

### 4.3 Dotenv 解析

```rust
// crates/openfang-extensions/src/credentials.rs:142-166
fn load_dotenv(path: &Path) -> Result<HashMap<String, String>, std::io::Error> {
    let content = std::fs::read_to_string(path)?;
    let mut map = HashMap::new();
    for line in content.lines() {
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') { continue; }
        if let Some((key, value)) = line.split_once('=') {
            let key = key.trim();
            let mut value = value.trim().to_string();
            // 移除引号
            if (value.starts_with('"') && value.ends_with('"'))
                || (value.starts_with('\'') && value.ends_with('\'')) {
                value = value[1..value.len() - 1].to_string();
            }
            map.insert(key.to_string(), value);
        }
    }
    Ok(map)
}
```

---

## 5. OAuth2 PKCE 流程

### 5.1 PKCE 原理

OAuth2 PKCE (Proof Key for Code Exchange) 是一种无需客户端密钥的授权流程，特别适合公共客户端：

```
┌─────────┐                          ┌─────────┐
│ OpenFang│                          │  Auth   │
│  Client │                          │ Server  │
└────┬────┘                          └────┬────┘
     │                                    │
     │  1. 生成 code_verifier & challenge │
     │                                    │
     │  2. 打开浏览器 (含 challenge)      │
     │ ─────────────────────────────────► │
     │                                    │ 3. 用户授权
     │                                    │
     │  4. 回调 localhost/?code=xxx       │
     │ ◄───────────────────────────────── │
     │                                    │
     │  5. 发送 code + verifier 交换 token│
     │ ─────────────────────────────────► │
     │                                    │
     │  6. 返回 access_token              │
     │ ◄───────────────────────────────── │
     │                                    │
```

### 5.2 PKCE 代码生成

```rust
// crates/openfang-extensions/src/oauth.rs:94-108
fn generate_pkce() -> PkcePair {
    // 生成 32 字节随机 verifier
    let mut bytes = [0u8; 32];
    rand::rngs::OsRng.fill_bytes(&mut bytes);
    let verifier = Zeroizing::new(base64_url_encode(&bytes));

    // 计算 S256 challenge = base64url(sha256(verifier))
    let challenge = {
        let mut hasher = Sha256::new();
        hasher.update(verifier.as_bytes());
        let digest = hasher.finalize();
        base64_url_encode(&digest)
    };

    PkcePair { verifier, challenge }
}
```

### 5.3 完整流程实现

```rust
// crates/openfang-extensions/src/oauth.rs:130-260
pub async fn run_pkce_flow(
    oauth: &OAuthTemplate,
    client_id: &str,
) -> ExtensionResult<OAuthTokens> {
    // 1. 生成 PKCE 参数
    let pkce = generate_pkce();
    let state = generate_state();  // CSRF 保护

    // 2. 启动 localhost 回调服务器
    let listener = TcpListener::bind("127.0.0.1:0").await?;
    let port = listener.local_addr()?.port();
    let redirect_uri = format!("http://127.0.0.1:{}/callback", port);

    // 3. 构建授权 URL
    let auth_url = format!(
        "{}?client_id={}&redirect_uri={}&response_type=code&scope={}&state={}&code_challenge={}&code_challenge_method=S256",
        oauth.auth_url,
        client_id, redirect_uri,
        oauth.scopes.join(" "),
        state, pkce.challenge,
    );

    // 4. 打开浏览器
    open_browser(&auth_url)?;

    // 5. 等待回调
    let code = wait_for_callback(listener, state).await?;

    // 6. 交换 token
    let tokens = exchange_code_for_tokens(oauth, client_id, &code, &pkce.verifier).await?;

    Ok(tokens)
}
```

### 5.4 支持的服务商

```rust
// crates/openfang-extensions/src/oauth.rs:18-27
pub fn default_client_ids() -> HashMap<&'static str, &'static str> {
    let mut m = HashMap::new();
    m.insert("google", "openfang-google-client-id");
    m.insert("github", "openfang-github-client-id");
    m.insert("microsoft", "openfang-microsoft-client-id");
    m.insert("slack", "openfang-slack-client-id");
    m
}
```

6 个集成支持 OAuth：

| 集成 | Provider | Scopes |
|------|----------|--------|
| GitHub | `github` | `repo`, `read:org` |
| Google Calendar | `google` | `calendar.readonly`, `calendar.events` |
| Gmail | `google` | `gmail.readonly`, `gmail.send` |
| Google Drive | `google` | `drive.readonly` |
| Slack | `slack` | `channels:read`, `chat:write` |
| Teams MCP | `microsoft` | `Files.Read`, `Calendars.Read` |

---

## 6. 健康监控 (Health Monitor)

### 6.1 架构设计

HealthMonitor 使用 DashMap 实现线程安全的健康状态跟踪：

```rust
// crates/openfang-extensions/src/health.rs:105-111
pub struct HealthMonitor {
    /// 键为 Integration ID 的健康记录
    health: Arc<DashMap<String, IntegrationHealth>>,
    /// 配置
    config: HealthMonitorConfig,
}
```

### 6.2 健康状态记录

```rust
// crates/openfang-extensions/src/health.rs:13-34
pub struct IntegrationHealth {
    pub id: String,
    pub status: IntegrationStatus,
    pub tool_count: usize,           // 可用工具数量
    pub last_ok: Option<DateTime<Utc>>,  // 最后成功时间
    pub last_error: Option<String>,  // 最后错误消息
    pub consecutive_failures: u32,   // 连续失败次数
    pub reconnecting: bool,          // 是否正在重连
    pub reconnect_attempts: u32,     // 重连尝试次数
    pub connected_since: Option<DateTime<Utc>>,  // 连接持续时间
}
```

### 6.3 指数退避重连

```rust
// crates/openfang-extensions/src/health.rs:158-163
pub fn backoff_duration(&self, attempt: u32) -> Duration {
    let base_secs = 5u64;
    // 5s → 10s → 20s → 40s → ... → 300s (上限)
    let backoff = base_secs.saturating_mul(1u64 << attempt.min(10));
    Duration::from_secs(backoff.min(self.config.max_backoff_secs))
}
```

**退避策略**：

| 尝试次数 | 等待时间 |
|----------|----------|
| 1 | 5 秒 |
| 2 | 10 秒 |
| 3 | 20 秒 |
| 4 | 40 秒 |
| 5 | 80 秒 |
| 6+ | 160-300 秒 (上限) |

### 6.4 重连逻辑

```rust
// crates/openfang-extensions/src/health.rs:165-176
pub fn should_reconnect(&self, id: &str) -> bool {
    if !self.config.auto_reconnect {
        return false;
    }
    if let Some(entry) = self.health.get(id) {
        matches!(entry.status, IntegrationStatus::Error(_))
            && entry.reconnect_attempts < self.config.max_reconnect_attempts
    } else {
        false
    }
}
```

**重连条件**：
1. 自动重连已启用
2. 状态为 Error
3. 未达到最大重连次数 (默认 10 次)

---

## 7. MCP 客户端集成

### 7.1 工具发现流程

```rust
// crates/openfang-runtime/src/mcp.rs:190-235
async fn discover_tools(&mut self) -> Result<(), String> {
    let response = self.send_request("tools/list", None).await?;

    if let Some(tools_array) = response.and_then(|r| r.get("tools").as_array()) {
        for tool in tools_array {
            let raw_name = tool["name"].as_str().unwrap_or("unnamed");
            let description = tool["description"].as_str().unwrap_or("");
            let input_schema = /* 解析 inputSchema */;

            // 命名空间：mcp_{server}_{tool}
            let namespaced = format_mcp_tool_name(server_name, raw_name);

            // 保存原始名称（保留连字符等）
            self.original_names.insert(namespaced.clone(), raw_name.to_string());

            self.tools.push(ToolDefinition {
                name: namespaced,
                description: format!("[MCP:{server_name}] {description}"),
                input_schema,
            });
        }
    }
    Ok(())
}
```

### 7.2 工具命名空间

```rust
// crates/openfang-runtime/src/mcp.rs:549-577
pub fn format_mcp_tool_name(server: &str, tool: &str) -> String {
    format!("mcp_{}_{}", normalize_name(server), normalize_name(tool))
}

pub fn normalize_name(name: &str) -> String {
    name.to_lowercase().replace('-', '_')
}
```

**示例**：
- `github` + `create_issue` → `mcp_github_create_issue`
- `my-server` + `do-thing` → `mcp_my_server_do_thing`

### 7.3 环境沙箱

MCP 子进程使用 `env_clear()` 进行环境隔离：

```rust
// crates/openfang-runtime/src/mcp.rs:453-482
let mut cmd = tokio::process::Command::new(&resolved_command);
cmd.env_clear();  // 清除所有环境变量

// 只传递白名单变量
for var_name in env_whitelist {
    if let Ok(val) = std::env::var(var_name) {
        cmd.env(var_name, val);
    }
}

// 始终传递 PATH
if let Ok(path) = std::env::var("PATH") {
    cmd.env("PATH", path);
}

// Windows 特定需求
if cfg!(windows) {
    for var in &["APPDATA", "LOCALAPPDATA", "USERPROFILE", "SystemRoot", "TEMP", "TMP"] {
        if let Ok(val) = std::env::var(var) {
            cmd.env(var, val);
        }
    }
}
```

---

## 8. 安装流程

### 8.1 `openfang add <name>` 命令

安装流程由 `installer.rs` 实现：

```
1. 查找模板 (bundled_integrations)
         │
         ▼
2. 检查是否已安装
         │
         ▼
3. 解析凭证 (CredentialResolver)
         │
         ▼
4. 如有 OAuth 配置 → 运行 PKCE 流程
         │
         ▼
5. 保存配置到 ~/.openfang/integrations.toml
         │
         ▼
6. 启动 MCP 连接 (McpConnection::connect)
         │
         ▼
7. 注册健康监控
```

### 8.2 持久化配置

```rust
// crates/openfang-extensions/src/lib.rs:208-223
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct InstalledIntegration {
    /// 模板 ID
    pub id: String,
    /// 安装时间
    pub installed_at: DateTime<Utc>,
    /// 是否启用
    #[serde(default = "default_true")]
    pub enabled: bool,
    /// OAuth Provider (如 "google")
    #[serde(default)]
    pub oauth_provider: Option<String>,
    /// 自定义配置覆盖
    #[serde(default)]
    pub config: HashMap<String, String>,
}
```

---

## 9. 安全特性

### 9.1 凭证存储安全

| 层面 | 措施 |
|------|------|
| **加密算法** | AES-256-GCM (认证加密) |
| **密钥派生** | Argon2id (防 GPU 攻击) |
| **内存保护** | Zeroizing 自动清零 |
| **机器绑定** | SHA-256(username + hostname) 混淆 |

### 9.2 MCP 子进程沙箱

| 保护 | 实现 |
|------|------|
| **环境隔离** | `env_clear()` 清除所有变量 |
| **白名单传递** | 只传递显式配置的 env vars |
| **路径遍历防护** | 拒绝含 `..` 的命令路径 |
| **SSRF 防护** | SSE URL 检查元数据端点 |

### 9.3 OAuth2 PKCE 安全

| 保护 | 说明 |
|------|------|
| **code_verifier** | 32 字节随机值，零化存储 |
| **code_challenge** | S256 SHA-256 哈希 |
| **state 参数** | 防 CSRF 攻击 |
| **localhost 回调** | 不暴露凭证到外部 |

---

## 10. 测试代码

### 10.1 Vault 测试

```rust
// crates/openfang-extensions/src/vault.rs:535-567
#[test]
fn vault_init_and_roundtrip() {
    let (dir, mut vault) = test_vault();
    let key = random_key();

    // 初始化创建 vault 文件
    vault.init_with_key(key.clone()).unwrap();
    assert!(vault.exists());
    assert!(vault.is_unlocked());
    assert!(vault.is_empty());

    // 存储凭证
    vault.set("GITHUB_TOKEN".to_string(), Zeroizing::new("ghp_test123".to_string())).unwrap();
    assert_eq!(vault.len(), 1);

    // 读取回来
    let val = vault.get("GITHUB_TOKEN").unwrap();
    assert_eq!(val.as_str(), "ghp_test123");

    // 新实例，用相同密钥解锁
    let mut vault2 = CredentialVault::new(dir.path().join("vault.enc"));
    vault2.unlock_with_key(key).unwrap();
    let val2 = vault2.get("GITHUB_TOKEN").unwrap();
    assert_eq!(val2.as_str(), "ghp_test123");
}
```

### 10.2 健康监控测试

```rust
// crates/openfang-extensions/src/health.rs:238-248
#[test]
fn backoff_exponential() {
    let monitor = HealthMonitor::new(HealthMonitorConfig::default());
    assert_eq!(monitor.backoff_duration(0), Duration::from_secs(5));
    assert_eq!(monitor.backoff_duration(1), Duration::from_secs(10));
    assert_eq!(monitor.backoff_duration(2), Duration::from_secs(20));
    assert_eq!(monitor.backoff_duration(3), Duration::from_secs(40));
    // 上限 300s
    assert_eq!(monitor.backoff_duration(10), Duration::from_secs(300));
    assert_eq!(monitor.backoff_duration(20), Duration::from_secs(300));
}
```

### 10.3 PKCE 测试

```rust
// crates/openfang-extensions/src/oauth.rs:329-338
#[test]
fn pkce_challenge_is_sha256() {
    let pkce = generate_pkce();
    // 验证：challenge = base64url(sha256(verifier))
    let mut hasher = Sha256::new();
    hasher.update(pkce.verifier.as_bytes());
    let digest = hasher.finalize();
    let expected = base64_url_encode(&digest);
    assert_eq!(pkce.challenge, expected);
}
```

---

## 11. 关键设计点

### 11.1 编译时嵌入

25 个 TOML 模板通过 `include_str!()` 编译时嵌入，确保：
- 零文件系统依赖
- 二进制自包含
- 启动即可用

### 11.2 凭证解析链

多层解析设计提供灵活性：
```
Vault (加密存储) → .env (开发方便) → Env (容器部署) → Interactive (CLI 友好)
```

### 11.3 健康监控架构

```
┌─────────────────────────────────────────────────────────┐
│                 HealthMonitor                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │  DashMap<String, IntegrationHealth>             │   │
│  │  - github: Ready, 12 tools                      │   │
│  │  - slack: Error, reconnecting...                │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
           │
           ▼ 后台任务 (每 60 秒)
┌─────────────────────────────────────────────────────────┐
│  for each integration:                                  │
│    if should_reconnect():                               │
│      sleep(backoff_duration(attempt))                   │
│      reconnect()                                        │
└─────────────────────────────────────────────────────────┘
```

### 11.4 OAuth PKCE 安全

- `code_verifier` 使用 `Zeroizing<String>` 存储
- `state` 参数防 CSRF
- localhost 回调确保授权码不离开本机
- 5 分钟超时防止挂起

---

## 完成检查清单

- [ ] 理解 Extensions 系统架构和 25 个 MCP 集成模板
- [ ] 掌握凭证保险箱的 AES-256-GCM 加密机制
- [ ] 理解 OAuth2 PKCE 流程的实现细节
- [ ] 掌握健康监控与自动重连机制

---

## 下一步

前往 [第 24 节：API 服务 — REST/WS 端点](./24-api-server.md)

---

*创建时间：2026-03-15*
*OpenFang v0.4.4*
