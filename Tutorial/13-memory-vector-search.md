# 第 13 节：记忆系统 — 向量搜索

> **版本**: v0.4.4 (2026-03-15)
> **核心文件**:
> - `crates/openfang-runtime/src/embedding.rs`
> - `crates/openfang-memory/src/semantic.rs`
> - `crates/openfang-memory/src/consolidation.rs`

## 学习目标

- [ ] 理解 EmbeddingDriver trait 和实现
- [ ] 掌握 8 个嵌入提供者配置
- [ ] 理解向量维度自动推断
- [ ] 掌握余弦相似度算法和测试
- [ ] 理解记忆衰减和 ConsolidationEngine
- [ ] 掌握 SQLite 向量存储方案
- [ ] 了解 Qdrant 集成扩展路径

---

## 1. EmbeddingDriver — 嵌入计算抽象

### 文件位置
`crates/openfang-runtime/src/embedding.rs:43-59`

```rust
/// Trait for computing text embeddings.
#[async_trait]
pub trait EmbeddingDriver: Send + Sync {
    /// Compute embedding vectors for a batch of texts.
    async fn embed(&self, texts: &[&str]) -> Result<Vec<Vec<f32>>, EmbeddingError>;

    /// Compute embedding for a single text.
    async fn embed_one(&self, text: &str) -> Result<Vec<f32>, EmbeddingError> {
        let results = self.embed(&[text]).await?;
        results
            .into_iter()
            .next()
            .ok_or_else(|| EmbeddingError::Parse("Empty embedding response".to_string()))
    }

    /// Return the dimensionality of embeddings produced by this driver.
    fn dimensions(&self) -> usize;
}
```

### 方法说明

| 方法 | 参数 | 返回 | 说明 |
|------|------|------|------|
| `embed()` | `&[&str]` (文本数组) | `Vec<Vec<f32>>` (向量数组) | 批量计算嵌入 |
| `embed_one()` | `&str` (单个文本) | `Vec<f32>` (单个向量) | 默认实现调用 `embed()` |
| `dimensions()` | 无 | `usize` | 返回向量维度 |

### 设计要点

**批量处理**: `embed()` 接受文本数组而非单个文本
- 利用 API 的 batch 能力（OpenAI 等支持一次多文本）
- 减少网络往返次数
- 提高吞吐量

**默认方法**: `embed_one()` 有默认实现
- 简化实现者工作
- 内部调用 `embed()` 并提取第一个结果

---

## 2. OpenAIEmbeddingDriver — OpenAI 兼容实现

### 文件位置
`crates/openfang-runtime/src/embedding.rs:61-175`

### 结构体定义

```rust
/// OpenAI-compatible embedding driver.
///
/// Works with any provider that implements the `/v1/embeddings` endpoint:
/// OpenAI, Groq, Together, Fireworks, Ollama, vLLM, LM Studio, etc.
pub struct OpenAIEmbeddingDriver {
    api_key: Zeroizing<String>,
    base_url: String,
    model: String,
    client: reqwest::Client,
    dims: usize,
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `api_key` | `Zeroizing<String>` | API Key（安全类型，自动清零） |
| `base_url` | `String` | API 基础 URL |
| `model` | `String` | 模型名称 |
| `client` | `reqwest::Client` | HTTP 客户端 |
| `dims` | `usize` | 向量维度 |

### EmbeddingConfig 配置

```rust
// embedding.rs:30-40
#[derive(Debug, Clone)]
pub struct EmbeddingConfig {
    /// Provider name (openai, groq, together, ollama, etc.).
    pub provider: String,
    /// Model name (e.g., "text-embedding-3-small", "all-MiniLM-L6-v2").
    pub model: String,
    /// API key (resolved from env var).
    pub api_key: String,
    /// Base URL for the API.
    pub base_url: String,
}
```

### 请求/响应结构

```rust
// embedding.rs:73-87
#[derive(Serialize)]
struct EmbedRequest<'a> {
    model: &'a str,
    input: &'a [&'a str],
}

#[derive(Deserialize)]
struct EmbedResponse {
    data: Vec<EmbedData>,
}

#[derive(Deserialize)]
struct EmbedData {
    embedding: Vec<f32>,
}
```

### embed() 实现

```rust
// embedding.rs:124-170
#[async_trait]
impl EmbeddingDriver for OpenAIEmbeddingDriver {
    async fn embed(&self, texts: &[&str]) -> Result<Vec<Vec<f32>>, EmbeddingError> {
        if texts.is_empty() {
            return Ok(vec![]);
        }

        // 1. 构建请求 URL
        let url = format!("{}/embeddings", self.base_url);
        let body = EmbedRequest {
            model: &self.model,
            input: texts,
        };

        // 2. 构建 HTTP 请求
        let mut req = self.client.post(&url).json(&body);
        if !self.api_key.as_str().is_empty() {
            req = req.header("Authorization", format!("Bearer {}", self.api_key.as_str()));
        }

        // 3. 发送请求
        let resp = req
            .send()
            .await
            .map_err(|e| EmbeddingError::Http(e.to_string()))?;
        let status = resp.status().as_u16();

        // 4. 错误处理
        if status != 200 {
            let body_text = resp.text().await.unwrap_or_default();
            return Err(EmbeddingError::Api {
                status,
                message: body_text,
            });
        }

        // 5. 解析响应
        let data: EmbedResponse = resp
            .json()
            .await
            .map_err(|e| EmbeddingError::Parse(e.to_string()))?;

        let embeddings: Vec<Vec<f32>> = data.data.into_iter().map(|d| d.embedding).collect();

        debug!(
            "Embedded {} texts (dims={})",
            embeddings.len(),
            embeddings.first().map(|e| e.len()).unwrap_or(0)
        );

        Ok(embeddings)
    }

    fn dimensions(&self) -> usize {
        self.dims
    }
}
```

---

## 3. 8 个嵌入提供者配置

### 文件位置
`crates/openfang-runtime/src/embedding.rs:178-250`

```rust
pub fn create_embedding_driver(
    provider: &str,
    model: &str,
    api_key_env: &str,
    custom_base_url: Option<&str>,
) -> Result<Box<dyn EmbeddingDriver + Send + Sync>, EmbeddingError> {
    let api_key = if api_key_env.is_empty() {
        String::new()
    } else {
        std::env::var(api_key_env).unwrap_or_default()
    };

    // 1. 处理自定义 URL
    let base_url = custom_base_url
        .filter(|u| !u.is_empty())
        .map(|u| {
            let trimmed = u.trim_end_matches('/');
            // 自动追加 /v1 路径
            let needs_v1 = matches!(
                provider,
                "openai" | "groq" | "together" | "fireworks" | "mistral" | "ollama" | "vllm" | "lmstudio"
            );
            if needs_v1 && !trimmed.ends_with("/v1") {
                format!("{trimmed}/v1")
            } else {
                trimmed.to_string()
            }
        })
        .unwrap_or_else(|| match provider {
            "openai" => OPENAI_BASE_URL.to_string(),
            "groq" => GROQ_BASE_URL.to_string(),
            "together" => TOGETHER_BASE_URL.to_string(),
            "fireworks" => FIREWORKS_BASE_URL.to_string(),
            "mistral" => MISTRAL_BASE_URL.to_string(),
            "ollama" => OLLAMA_BASE_URL.to_string(),
            "vllm" => VLLM_BASE_URL.to_string(),
            "lmstudio" => LMSTUDIO_BASE_URL.to_string(),
            other => {
                warn!("Unknown embedding provider '{other}', using OpenAI-compatible format");
                format!("https://{other}/v1")
            }
        });

    // 2. 安全警告（外部 API）
    let is_local = base_url.contains("localhost")
        || base_url.contains("127.0.0.1")
        || base_url.contains("[::1]");
    if !is_local {
        warn!(
            provider = %provider,
            base_url = %base_url,
            "Embedding driver configured to send data to external API — text content will leave this machine"
        );
    }

    // 3. 创建驱动
    let config = EmbeddingConfig {
        provider: provider.to_string(),
        model: model.to_string(),
        api_key,
        base_url,
    };

    let driver = OpenAIEmbeddingDriver::new(config)?;
    Ok(Box::new(driver))
}
```

### 提供者配置表

| Provider | 默认 Base URL | 典型模型 | API Key 环境变量 |
|----------|---------------|----------|------------------|
| `openai` | `https://api.openai.com` | `text-embedding-3-small` | `OPENAI_API_KEY` |
| `groq` | `https://api.groq.com` | `nomic-embed-text` | `GROQ_API_KEY` |
| `together` | `https://api.together.xyz` | `BAAI/bge-base-en-v1.5` | `TOGETHER_API_KEY` |
| `fireworks` | `https://api.fireworks.ai` | `nomic-embed-text` | `FIREWORKS_API_KEY` |
| `mistral` | `https://api.mistral.ai` | `mistral-embed` | `MISTRAL_API_KEY` |
| `ollama` | `http://localhost:11434` | `nomic-embed-text` | 无需 |
| `vllm` | `http://localhost:8000` | 自定义 | 可选 |
| `lmstudio` | `http://localhost:1234` | 自定义 | 无需 |

### 本地 vs 外部 API

| 类型 | Provider | 特点 | 推荐场景 |
|------|----------|------|----------|
| **本地** | `ollama`, `vllm`, `lmstudio` | 无需 API Key，数据不离本地 | 隐私敏感、开发测试 |
| **外部** | `openai`, `groq`, `together` | 需要 API Key，高质量嵌入 | 生产环境、高精度需求 |

---

## 4. 向量维度自动推断

### 文件位置
`crates/openfang-runtime/src/embedding.rs:105-121`

```rust
/// Infer embedding dimensions from model name.
fn infer_dimensions(model: &str) -> usize {
    match model {
        // OpenAI
        "text-embedding-3-small" => 1536,
        "text-embedding-3-large" => 3072,
        "text-embedding-ada-002" => 1536,
        // Sentence Transformers / local models
        "all-MiniLM-L6-v2" => 384,
        "all-MiniLM-L12-v2" => 384,
        "all-mpnet-base-v2" => 768,
        "nomic-embed-text" => 768,
        "mxbai-embed-large" => 1024,
        // Default to 1536 (most common)
        _ => 1536,
    }
}
```

### 常见模型维度表

| 模型名称 | 维度 | 提供者 | 特点 |
|----------|------|--------|------|
| `text-embedding-3-small` | 1536 | OpenAI | 高性价比 |
| `text-embedding-3-large` | 3072 | OpenAI | 最高质量 |
| `text-embedding-ada-002` | 1536 | OpenAI | 经典模型 |
| `all-MiniLM-L6-v2` | 384 | SentenceBERT | 轻量快速 |
| `all-MiniLM-L12-v2` | 384 | SentenceBERT | 稍大版本 |
| `all-mpnet-base-v2` | 768 | SentenceBERT | 高质量 |
| `nomic-embed-text` | 768 | Nomic AI | 开源 SOTA |
| `mxbai-embed-large` | 1024 | MXB AI | 大型模型 |

---

## 5. 余弦相似度算法详解

### 文件位置
`crates/openfang-runtime/src/embedding.rs:252-276`

```rust
/// Compute cosine similarity between two vectors.
///
/// Returns a value in [-1.0, 1.0] where 1.0 = identical direction.
pub fn cosine_similarity(a: &[f32], b: &[f32]) -> f32 {
    if a.len() != b.len() || a.is_empty() {
        return 0.0;
    }

    let mut dot = 0.0f32;
    let mut norm_a = 0.0f32;
    let mut norm_b = 0.0f32;

    for i in 0..a.len() {
        dot += a[i] * b[i];
        norm_a += a[i] * a[i];
        norm_b += b[i] * b[i];
    }

    let denom = norm_a.sqrt() * norm_b.sqrt();
    if denom < f32::EPSILON {
        0.0
    } else {
        dot / denom
    }
}
```

### 算法分解

```
余弦相似度 = cos(θ) = (A · B) / (||A|| × ||B||)

步骤 1: 计算点积 (dot product)
  dot = Σ(a[i] × b[i])

步骤 2: 计算 L2 范数 (向量长度)
  ||A|| = sqrt(Σ(a[i]²))
  ||B|| = sqrt(Σ(b[i]²))

步骤 3: 计算余弦值
  cos(θ) = dot / (||A|| × ||B||)
```

### 测试用例

```rust
// embedding.rs:299-347
#[test]
fn test_cosine_similarity_identical() {
    let a = vec![1.0, 0.0, 0.0];
    let b = vec![1.0, 0.0, 0.0];
    let sim = cosine_similarity(&a, &b);
    assert!((sim - 1.0).abs() < 1e-6);  // 完全相同 = 1.0
}

#[test]
fn test_cosine_similarity_orthogonal() {
    let a = vec![1.0, 0.0];
    let b = vec![0.0, 1.0];
    let sim = cosine_similarity(&a, &b);
    assert!(sim.abs() < 1e-6);  // 正交 = 0.0
}

#[test]
fn test_cosine_similarity_opposite() {
    let a = vec![1.0, 0.0];
    let b = vec![-1.0, 0.0];
    let sim = cosine_similarity(&a, &b);
    assert!((sim + 1.0).abs() < 1e-6);  // 完全相反 = -1.0
}

#[test]
fn test_cosine_similarity_real_vectors() {
    let a = vec![0.1, 0.2, 0.3, 0.4];
    let b = vec![0.1, 0.2, 0.3, 0.4];
    let sim = cosine_similarity(&a, &b);
    assert!((sim - 1.0).abs() < 1e-5);

    let c = vec![0.4, 0.3, 0.2, 0.1];
    let sim2 = cosine_similarity(&a, &c);
    assert!(sim2 > 0.0 && sim2 < 1.0);  // 相似但不完全相同
}

#[test]
fn test_cosine_similarity_empty() {
    let sim = cosine_similarity(&[], &[]);
    assert_eq!(sim, 0.0);  // 空向量返回 0.0
}

#[test]
fn test_cosine_similarity_length_mismatch() {
    let a = vec![1.0, 2.0];
    let b = vec![1.0, 2.0, 3.0];
    let sim = cosine_similarity(&a, &b);
    assert_eq!(sim, 0.0);  // 长度不匹配返回 0.0
}
```

### 相似度值解释

| 值范围 | 说明 | 示例 |
|--------|------|------|
| `1.0` | 完全相同 | 同一文本的嵌入 |
| `0.8 - 0.99` | 高度相似 | 同义词、近义句 |
| `0.5 - 0.8` | 中度相似 | 相关主题 |
| `0.2 - 0.5` | 低度相似 | 弱相关 |
| `0.0 - 0.2` | 几乎无关 | 不同主题 |
| `< 0.0` | 负相关 | 语义相反 |

---

## 6. 向量序列化 — 嵌入存储

### 文件位置
`crates/openfang-runtime/src/embedding.rs:278-293`

```rust
/// Serialize an embedding vector to bytes (for SQLite BLOB storage).
pub fn embedding_to_bytes(embedding: &[f32]) -> Vec<u8> {
    let mut bytes = Vec::with_capacity(embedding.len() * 4);
    for &val in embedding {
        bytes.extend_from_slice(&val.to_le_bytes());
    }
    bytes
}

/// Deserialize an embedding vector from bytes.
pub fn embedding_from_bytes(bytes: &[u8]) -> Vec<f32> {
    bytes
        .chunks_exact(4)
        .map(|chunk| f32::from_le_bytes([chunk[0], chunk[1], chunk[2], chunk[3]]))
        .collect()
}
```

### 序列化格式

```
f32 向量 → 小端字节序 (Little-Endian)

示例：[0.1, -0.5, 1.23456]
  0.1      → [0xCD, 0xCC, 0xCC, 0x3D]
  -0.5     → [0x00, 0x00, 0x00, 0xBF]
  1.23456  → [0x74, 0x1D, 0x9D, 0x3F]

总字节数 = 维度 × 4
```

### 测试用例

```rust
// embedding.rs:349-366
#[test]
fn test_embedding_roundtrip() {
    let embedding = vec![0.1, -0.5, 1.23456, 0.0, -1e10, 1e10];
    let bytes = embedding_to_bytes(&embedding);
    let recovered = embedding_from_bytes(&bytes);
    assert_eq!(embedding.len(), recovered.len());
    for (a, b) in embedding.iter().zip(recovered.iter()) {
        assert!((a - b).abs() < f32::EPSILON);
    }
}

#[test]
fn test_embedding_bytes_empty() {
    let bytes = embedding_to_bytes(&[]);
    assert!(bytes.is_empty());
    let recovered = embedding_from_bytes(&bytes);
    assert!(recovered.is_empty());
}
```

---

## 7. ConsolidationEngine — 记忆衰减引擎

### 文件位置
`crates/openfang-memory/src/consolidation.rs`

### 结构体定义

```rust
// consolidation.rs:13-24
/// Memory consolidation engine.
#[derive(Clone)]
pub struct ConsolidationEngine {
    conn: Arc<Mutex<Connection>>,
    /// Decay rate: how much to reduce confidence per consolidation cycle.
    decay_rate: f32,
}

impl ConsolidationEngine {
    /// Create a new consolidation engine.
    pub fn new(conn: Arc<Mutex<Connection>>, decay_rate: f32) -> Self {
        Self { conn, decay_rate }
    }
}
```

### consolidate() — 衰减逻辑

```rust
// consolidation.rs:26-54
pub fn consolidate(&self) -> OpenFangResult<ConsolidationReport> {
    let start = std::time::Instant::now();
    let conn = self
        .conn
        .lock()
        .map_err(|e| OpenFangError::Internal(e.to_string()))?;

    // 1. 计算 7 天前的时间 cutoff
    let cutoff = (Utc::now() - chrono::Duration::days(7)).to_rfc3339();
    let decay_factor = 1.0 - self.decay_rate as f64;

    // 2. 衰减 7 天内未访问的记忆
    let decayed = conn
        .execute(
            "UPDATE memories SET confidence = MAX(0.1, confidence * ?1)
             WHERE deleted = 0 AND accessed_at < ?2 AND confidence > 0.1",
            rusqlite::params![decay_factor, cutoff],
        )
        .map_err(|e| OpenFangError::Memory(e.to_string()))?;

    let duration_ms = start.elapsed().as_millis() as u64;

    Ok(ConsolidationReport {
        memories_merged: 0, // Phase 1: 暂无合并功能
        memories_decayed: decayed as u64,
        duration_ms,
    })
}
```

### 衰减机制详解

```
衰减公式: new_confidence = MAX(0.1, old_confidence × (1 - decay_rate))

示例 (decay_rate = 0.1):
  初始 confidence = 0.9
  7 天后: 0.9 × 0.9 = 0.81
  14 天后: 0.81 × 0.9 = 0.729
  21 天后: 0.729 × 0.9 = 0.656
  ...
  最小值: 0.1 (下限保护)
```

### 衰减条件

| 条件 | SQL | 说明 |
|------|-----|------|
| 未删除 | `deleted = 0` | 软删除的记忆不衰减 |
| 7 天未访问 | `accessed_at < cutoff` | 最近访问的记忆不衰减 |
| 置信度 > 0.1 | `confidence > 0.1` | 已達最低值的不衰减 |

### 测试用例

```rust
// consolidation.rs:67-100
#[test]
fn test_consolidation_decays_old_memories() {
    let engine = setup();
    let conn = engine.conn.lock().unwrap();

    // 插入 30 天前的旧记忆
    let old_date = (Utc::now() - chrono::Duration::days(30)).to_rfc3339();
    conn.execute(
        "INSERT INTO memories (id, agent_id, content, source, scope, confidence, metadata, created_at, accessed_at, access_count, deleted)
         VALUES ('test-id', 'agent-1', 'old memory', '\"conversation\"', 'episodic', 0.9, '{}', ?1, ?1, 0, 0)",
        rusqlite::params![old_date],
    ).unwrap();
    drop(conn);

    // 运行衰减
    let report = engine.consolidate().unwrap();
    assert_eq!(report.memories_decayed, 1);

    // 验证置信度降低
    let conn = engine.conn.lock().unwrap();
    let confidence: f64 = conn
        .query_row("SELECT confidence FROM memories WHERE id = 'test-id'", [], |row| row.get(0))
        .unwrap();
    assert!(confidence < 0.9);
}
```

---

## 8. SQLite 向量存储方案

### 为什么不用专用向量数据库

| 方案 | 优点 | 缺点 |
|------|------|------|
| **SQLite BLOB** | 单文件、零依赖、事务性 | 无 HNSW 索引、线性扫描 |
| **Qdrant** | HNSW 索引、分布式、高性能 | 额外服务、运维成本 |
| **Milvus** | 高性能、多索引类型 | 复杂部署、资源消耗大 |
| **Chroma** | 简单易用 | Python 依赖、性能一般 |

### OpenFang 的选择：SQLite BLOB

**Phase 1 设计决策**:
- 使用 SQLite BLOB 存储向量
- 线性扫描 + 余弦相似度排序
- 适合中小规模（< 10 万条记忆）

**性能分析**:
```
10 万条记忆 × 1536 维度 = 614MB 向量数据
单次查询：全表扫描 + 10 万次余弦计算
耗时：~100-500ms（可接受）
```

### 存储结构

```sql
CREATE TABLE memories (
    id TEXT PRIMARY KEY,
    agent_id TEXT NOT NULL,
    content TEXT NOT NULL,
    source TEXT NOT NULL,
    scope TEXT NOT NULL,
    confidence REAL DEFAULT 1.0,
    metadata TEXT NOT NULL,
    created_at TEXT NOT NULL,
    accessed_at TEXT NOT NULL,
    access_count INTEGER DEFAULT 0,
    deleted INTEGER DEFAULT 0,
    embedding BLOB  -- f32 向量的 LE 字节
);
```

### 召回流程

```rust
// semantic.rs:95-277 - 核心逻辑
pub fn recall_with_embedding(
    &self,
    query: &str,
    limit: usize,
    filter: Option<MemoryFilter>,
    query_embedding: Option<&[f32]>,
) -> OpenFangResult<Vec<MemoryFragment>> {
    let conn = self.conn.lock()?;

    // 1. 获取候选（10 倍于 limit）
    let fetch_limit = if query_embedding.is_some() {
        (limit * 10).max(100)
    } else {
        limit
    };

    // 2. SQL 查询（基础过滤）
    let mut sql = String::from(
        "SELECT id, agent_id, content, source, scope, confidence, metadata, created_at, accessed_at, access_count, embedding
         FROM memories WHERE deleted = 0"
    );
    // 应用过滤器...

    // 3. 执行查询并解析
    let mut fragments = Vec::new();
    for row_result in rows {
        // 解析 MemoryFragment...
    }

    // 4. 余弦相似度重排序
    if let Some(qe) = query_embedding {
        fragments.sort_by(|a, b| {
            let sim_a = a.embedding.as_deref()
                .map(|e| cosine_similarity(qe, e))
                .unwrap_or(-1.0);
            let sim_b = b.embedding.as_deref()
                .map(|e| cosine_similarity(qe, e))
                .unwrap_or(-1.0);
            sim_b.partial_cmp(&sim_a).unwrap_or(Ordering::Equal)
        });
        fragments.truncate(limit);
    }

    // 5. 更新访问计数
    for frag in &fragments {
        let _ = conn.execute(
            "UPDATE memories SET access_count = access_count + 1, accessed_at = ?1 WHERE id = ?2",
            rusqlite::params![Utc::now().to_rfc3339(), frag.id.0.to_string()],
        );
    }

    Ok(fragments)
}
```

---

## 9. 向量召回完整流程

```
┌─────────────────────────────────────────────────────────────┐
│ 1. 用户查询："Rust 编程书籍推荐"                            │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. EmbeddingDriver.embed_one("Rust 编程书籍推荐")           │
│    → POST https://api.openai.com/v1/embeddings              │
│    → 返回：[0.023, -0.045, 0.089, ...] (1536 维)            │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. SQL 查询：SELECT * FROM memories WHERE deleted = 0       │
│    AND agent_id = ? AND scope = 'episodic'                  │
│    LIMIT 100 (fetch_limit = 10 × 10)                        │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. 对每个候选计算余弦相似度：                               │
│    sim[0] = cosine_similarity(query_emb, candidates[0].emb) │
│    sim[1] = cosine_similarity(query_emb, candidates[1].emb) │
│    ...                                                      │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. 按相似度降序排序：                                       │
│    [0.92, 0.87, 0.81, 0.76, 0.65, ...]                      │
│    截取前 10 个                                              │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. 返回结果并更新访问计数：                                 │
│    UPDATE memories SET access_count++ WHERE id IN (...)     │
└─────────────────────────────────────────────────────────────┘
```

---

## 10. Qdrant 集成扩展路径

虽然当前使用 SQLite BLOB 存储向量，但未来可以轻松扩展到 Qdrant：

### 扩展架构

```
┌─────────────────────────────────────────────────────────────┐
│                    MemorySubstrate                          │
└─────────────────────────────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ Structured    │ │   Semantic    │ │   Knowledge   │
│ Store (SQLite)│ │   Store       │ │   Store       │
│               │ │               │ │               │
│ • SQLite      │ │ • Phase 1:    │ │ • SQLite      │
│ • KV 存储     │ │   SQLite BLOB │ │ • 实体图谱    │
│               │ │ • Phase 2:    │ │               │
│               │ │   Qdrant      │ │               │
└───────────────┘ └───────────────┘ └───────────────┘
```

### Qdrant 集成代码示例（伪代码）

```rust
pub struct QdrantSemanticStore {
    client: QdrantClient,
    collection: String,
}

impl QdrantSemanticStore {
    pub async fn recall(
        &self,
        query_embedding: &[f32],
        limit: usize,
        filter: Option<MemoryFilter>,
    ) -> OpenFangResult<Vec<MemoryFragment>> {
        // 使用 Qdrant HNSW 索引进行向量搜索
        let results = self.client
            .search_points(&SearchPoints {
                collection_name: self.collection.clone(),
                vector: query_embedding.to_vec(),
                limit: limit as u64,
                with_payload: Some(true),
                filter: build_qdrant_filter(filter),
                ..Default::default()
            })
            .await?;

        // 转换为 MemoryFragment
        Ok(results.into_iter().map(|r| {
            let fragment: MemoryFragment = serde_json::from_value(r.payload).unwrap();
            fragment
        }).collect())
    }
}
```

### Qdrant 配置示例

```toml
# ~/.openfang/config.toml
[memory.vector]
provider = "qdrant"  # 或 "sqlite"
qdrant_url = "http://localhost:6333"
collection = "openfang_memories"
embedding_model = "text-embedding-3-small"
```

---

## 11. 测试用例

### EmbeddingDriver 测试

```rust
// embedding.rs:376-418
#[test]
fn test_create_embedding_driver_ollama() {
    // Ollama 无需 API Key
    let driver = create_embedding_driver("ollama", "all-MiniLM-L6-v2", "", None);
    assert!(driver.is_ok());
    assert_eq!(driver.unwrap().dimensions(), 384);
}

#[test]
fn test_create_embedding_driver_custom_url_with_v1() {
    // 自定义 URL 已包含 /v1
    let driver = create_embedding_driver(
        "ollama", "nomic-embed-text", "",
        Some("http://192.168.0.1:11434/v1"),
    );
    assert!(driver.is_ok());
}

#[test]
fn test_create_embedding_driver_custom_url_without_v1() {
    // 自定义 URL 缺少 /v1，自动追加
    let driver = create_embedding_driver(
        "ollama", "nomic-embed-text", "",
        Some("http://192.168.0.1:11434"),
    );
    assert!(driver.is_ok());
}
```

### 向量召回测试

```rust
// semantic.rs:441-492
#[test]
fn test_vector_recall_ranking() {
    let store = setup();
    let agent_id = AgentId::new();

    // 存储 3 个不同方向的嵌入
    let emb_rust = vec![0.9, 0.1, 0.0, 0.0];    // "Rust" 方向
    let emb_python = vec![0.0, 0.0, 0.9, 0.1];  // "Python" 方向
    let emb_mixed = vec![0.5, 0.5, 0.0, 0.0];   // 混合方向

    store.remember_with_embedding(
        agent_id, "Rust is a systems language",
        MemorySource::Conversation, "episodic", HashMap::new(),
        Some(&emb_rust),
    ).unwrap();

    store.remember_with_embedding(
        agent_id, "Python is interpreted",
        MemorySource::Conversation, "episodic", HashMap::new(),
        Some(&emb_python),
    ).unwrap();

    store.remember_with_embedding(
        agent_id, "Both are popular",
        MemorySource::Conversation, "episodic", HashMap::new(),
        Some(&emb_mixed),
    ).unwrap();

    // 使用 "Rust" 类似的查询嵌入
    let query_emb = vec![0.85, 0.15, 0.0, 0.0];
    let results = store
        .recall_with_embedding("", 3, None, Some(&query_emb))
        .unwrap();

    assert_eq!(results.len(), 3);
    // Rust 记忆应该排第一（最高余弦相似度）
    assert!(results[0].content.contains("Rust"));
    // Python 记忆应该排最后（最低相似度）
    assert!(results[2].content.contains("Python"));
}
```

---

## 12. 关键设计点

### 12.1 OpenAI 兼容设计

**统一接口**:
- 所有提供者使用相同的 `/v1/embeddings` 端点
- 只需配置 `base_url` 和 `api_key`
- 一处实现，多提供者复用

**支持的提供者**:
| 类型 | Provider | 使用场景 |
|------|----------|----------|
| 云服务 | OpenAI, Groq, Together | 生产环境、高质量 |
| 本地 | Ollama, vLLM, LM Studio | 开发测试、隐私敏感 |

### 12.2 安全设计

**Zeroizing 类型**:
```rust
api_key: Zeroizing<String>  // 离开作用域时自动清零内存
```

**安全警告**:
```rust
let is_local = base_url.contains("localhost")
    || base_url.contains("127.0.0.1")
    || base_url.contains("[::1]");
if !is_local {
    warn!(
        "Embedding driver configured to send data to external API — text content will leave this machine"
    );
}
```

### 12.3 批量优化

```rust
// 批量嵌入减少网络往返
async fn embed(&self, texts: &[&str]) -> Result<Vec<Vec<f32>>, EmbeddingError>

// 默认实现（单次调用）
async fn embed_one(&self, text: &str) -> Result<Vec<f32>, EmbeddingError> {
    let results = self.embed(&[text]).await?;
    results.into_iter().next()...
}
```

### 12.4 衰减策略

| 策略 | 公式 | 特点 |
|------|------|------|
| **线性衰减** | `c -= rate` | 简单，但可能变负 |
| **指数衰减** | `c *= (1 - rate)` | 自然衰减，永不归零 |
| **OpenFang** | `MAX(0.1, c × 0.9)` | 指数衰减 + 下限保护 |

---

## 完成检查清单

- [ ] 理解 EmbeddingDriver trait 和实现
- [ ] 掌握 8 个嵌入提供者配置
- [ ] 理解向量维度自动推断
- [ ] 掌握余弦相似度算法和测试
- [ ] 理解记忆衰减和 ConsolidationEngine
- [ ] 掌握 SQLite 向量存储方案
- [ ] 了解 Qdrant 集成扩展路径

---

## 下一步

前往 [第 14 节：Hands 系统 — 配置与激活](./14-hands-config.md)

---

*创建时间：2026-03-15*
*OpenFang v0.4.4*
