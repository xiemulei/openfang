# 第 12 节：记忆系统 — 三层存储

> **版本**: v0.5.2 (2026-03-29)
> **核心文件**:
> - `crates/openfang-types/src/memory.rs`
> - `crates/openfang-memory/src/structured.rs`
> - `crates/openfang-memory/src/semantic.rs`
> - `crates/openfang-memory/src/knowledge.rs`
> - `crates/openfang-memory/src/substrate.rs`
> - `crates/openfang-memory/src/http_client.rs` (v0.5.2 新增)

## 学习目标

- [ ] 理解三层存储架构（Structured/Semantic/Knowledge Graph）
- [ ] 掌握 MemoryFragment 结构和字段含义
- [ ] 理解 MemorySource 枚举的 6 种来源
- [ ] 掌握 MemoryFilter 过滤机制
- [ ] 理解 StructuredStore KV 操作
- [ ] 理解 SemanticStore 向量搜索（余弦相似度算法）
- [ ] 理解 KnowledgeStore 实体关系
- [ ] 掌握 MemorySubstrate 统一 API
- [ ] 了解 HTTP 后端与 SQLite 双后端架构 (v0.5.2 新增)
- [ ] 了解任务队列 API (v0.5.2 新增)

---

## 1. 三层存储架构概述

OpenFang 的记忆系统采用三层架构设计，每层负责不同的存储和检索需求：

```
┌─────────────────────────────────────────────────────────────┐
│                    MemorySubstrate (统一 API)                │
│  get/set/remember/recall/add_entity/query_graph/consolidate │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐   ┌─────────────────┐   ┌─────────────────┐
│ Structured    │   │   Semantic      │   │   Knowledge     │
│ Store         │   │   Store         │   │   Store         │
│               │   │                 │   │                 │
│ • KV 存储     │   │ • 向量搜索      │   │ • 实体图谱      │
│ • Agent 持久化│   │ • 语义召回      │   │ • 关系查询      │
│ • Session 管理│   │ • 余弦相似度    │   │ • 图模式匹配    │
└───────────────┘   └─────────────────┘   └─────────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              ▼
                    ┌─────────────────┐
                    │  SQLite (单文件)│
                    │  + WAL 模式     │
                    └─────────────────┘
```

### 三层职责对比

| 层级 | 存储内容 | 查询方式 | 典型用途 |
|------|----------|----------|----------|
| **Structured Store** | KV 对、Agent 元数据、Session | 精确键查找 | 用户偏好、配置、对话历史 |
| **Semantic Store** | 带嵌入的记忆片段 | 向量相似度搜索 | 语义回忆、上下文检索 |
| **Knowledge Graph** | 实体和关系 | 图模式查询 | 事实知识、关系推理 |

---

## 2. MemoryFragment — 记忆片段结构

### 文件位置
`crates/openfang-types/src/memory.rs:52-76`

```rust
/// A single unit of memory stored in the semantic store.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MemoryFragment {
    /// Unique ID.
    pub id: MemoryId,
    /// Which agent owns this memory.
    pub agent_id: AgentId,
    /// The textual content of this memory.
    pub content: String,
    /// Vector embedding (populated by the semantic store).
    pub embedding: Option<Vec<f32>>,
    /// Arbitrary metadata.
    pub metadata: HashMap<String, serde_json::Value>,
    /// How this memory was created.
    pub source: MemorySource,
    /// Confidence score (0.0 - 1.0).
    pub confidence: f32,
    /// When this memory was created.
    pub created_at: DateTime<Utc>,
    /// When this memory was last accessed.
    pub accessed_at: DateTime<Utc>,
    /// How many times this memory has been accessed.
    pub access_count: u64,
    /// Memory scope/collection name.
    pub scope: String,
}
```

### 字段详解

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `id` | `MemoryId` | 唯一标识符（UUID v4） | `"550e8400-e29b-41d4-a716-446655440000"` |
| `agent_id` | `AgentId` | 所有者 Agent ID | `"agent_abc123"` |
| `content` | `String` | 记忆文本内容 | `"Rust 是系统编程语言"` |
| `embedding` | `Option<Vec<f32>>` | 向量嵌入（用于语义搜索） | `Some([0.1, -0.2, ...])` |
| `metadata` | `HashMap` | 任意元数据 | `{"topic": "programming"}` |
| `source` | `MemorySource` | 记忆来源（6 种枚举） | `MemorySource::Conversation` |
| `confidence` | `f32` | 置信度（0.0-1.0） | `0.95` |
| `created_at` | `DateTime<Utc>` | 创建时间 | `2026-03-15T10:30:00Z` |
| `accessed_at` | `DateTime<Utc>` | 最后访问时间 | `2026-03-15T12:00:00Z` |
| `access_count` | `u64` | 访问次数 | `42` |
| `scope` | `String` | 作用域/集合名 | `"episodic"`, `"semantic"` |

### 字段设计意图

**access_count 和 accessed_at**:
- 用于实现 **使用频率衰减**（经常访问的记忆置信度更高）
- 支持 **最近最少使用（LRU）** 淘汰策略

**confidence**:
- 初始值为 1.0
- 随着时间衰减（由 ConsolidationEngine 管理）
- 低置信度记忆可被自动清理

**scope**:
- 区分不同类型的记忆集合
- 常见值：`"episodic"`（事件）、`"semantic"`（事实）、`"procedural"`（技能）

---

## 3. MemorySource — 记忆来源枚举

### 文件位置
`crates/openfang-types/src/memory.rs:34-49`

```rust
/// Where a memory came from.
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum MemorySource {
    /// From a conversation/interaction.
    Conversation,
    /// From a document that was processed.
    Document,
    /// From an observation (tool output, web page, etc.).
    Observation,
    /// Inferred by the agent from existing knowledge.
    Inference,
    /// Explicitly provided by the user.
    UserProvided,
    /// From a system event.
    System,
}
```

### 6 种来源详解

| 来源 | 说明 | 典型内容 | 置信度初始值 |
|------|------|----------|--------------|
| **Conversation** | 对话/交互中产生 | "用户说喜欢 Rust" | 0.8 |
| **Document** | 处理的文档提取 | "Rust 文档第 3 章讲所有权" | 0.9 |
| **Observation** | 观察结果（工具输出/网页） | "网页显示当前时间 10:30" | 0.7 |
| **Inference** | 从现有知识推理得出 | "用户可能是 Rust 开发者" | 0.6 |
| **UserProvided** | 用户明确提供 | "记住：我的名字是 Alice" | 1.0 |
| **System** | 系统事件产生 | "Agent 启动成功" | 0.5 |

### 来源使用示例

```rust
// 对话中提取的记忆
MemoryFragment {
    source: MemorySource::Conversation,
    content: "用户正在学习第 12 节文档".to_string(),
    confidence: 0.8,
    ..
}

// 用户明确指令
MemoryFragment {
    source: MemorySource::UserProvided,
    content: "记住：我的生日是 3 月 15 日".to_string(),
    confidence: 1.0,
    ..
}

// Agent 推理得出
MemoryFragment {
    source: MemorySource::Inference,
    content: "用户可能是 Rust 初学者".to_string(),
    confidence: 0.6,
    ..
}
```

---

## 4. MemoryFilter — 记忆过滤机制

### 文件位置
`crates/openfang-types/src/memory.rs:79-113`

```rust
/// Filter criteria for memory recall.
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct MemoryFilter {
    /// Filter by agent ID.
    pub agent_id: Option<AgentId>,
    /// Filter by source type.
    pub source: Option<MemorySource>,
    /// Filter by scope.
    pub scope: Option<String>,
    /// Minimum confidence threshold.
    pub min_confidence: Option<f32>,
    /// Only memories created after this time.
    pub after: Option<DateTime<Utc>>,
    /// Only memories created before this time.
    pub before: Option<DateTime<Utc>>,
    /// Metadata key-value filters.
    pub metadata: HashMap<String, serde_json::Value>,
}

impl MemoryFilter {
    /// Create a filter for a specific agent.
    pub fn agent(agent_id: AgentId) -> Self {
        Self {
            agent_id: Some(agent_id),
            ..Default::default()
        }
    }

    /// Create a filter for a specific scope.
    pub fn scope(scope: impl Into<String>) -> Self {
        Self {
            scope: Some(scope.into()),
            ..Default::default()
        }
    }
}
```

### 过滤字段详解

| 字段 | 类型 | 说明 | SQL 对应 |
|------|------|------|----------|
| `agent_id` | `Option<AgentId>` | 按 Agent 过滤 | `WHERE agent_id = ?` |
| `source` | `Option<MemorySource>` | 按来源类型过滤 | `WHERE source = ?` |
| `scope` | `Option<String>` | 按作用域过滤 | `WHERE scope = ?` |
| `min_confidence` | `Option<f32>` | 最小置信度 | `WHERE confidence >= ?` |
| `after` | `Option<DateTime>` | 创建时间之后 | `WHERE created_at >= ?` |
| `before` | `Option<DateTime>` | 创建时间之前 | `WHERE created_at <= ?` |
| `metadata` | `HashMap` | 元数据键值匹配 | `WHERE metadata->>? = ?` |

### 使用示例

```rust
// 只查询某个 agent 的记忆
let filter = MemoryFilter::agent(my_agent_id);

// 只查询 episodic scope 的记忆
let filter = MemoryFilter::scope("episodic");

// 组合过滤
let mut filter = MemoryFilter::default();
filter.agent_id = Some(agent_id);
filter.source = Some(MemorySource::Conversation);
filter.min_confidence = Some(0.8);
filter.scope = Some("episodic".to_string());
```

---

## 5. StructuredStore — KV 存储和 Agent 持久化

### 文件位置
`crates/openfang-memory/src/structured.rs`

### 数据库表结构

```sql
-- KV 存储表
CREATE TABLE kv_store (
    agent_id TEXT NOT NULL,
    key TEXT NOT NULL,
    value BLOB NOT NULL,        -- MessagePack 序列化的 JSON
    version INTEGER DEFAULT 1,
    updated_at TEXT NOT NULL,
    PRIMARY KEY (agent_id, key)
);

-- Agent 存储表
CREATE TABLE agents (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    manifest BLOB NOT NULL,     -- MessagePack 序列化的 AgentManifest
    state TEXT NOT NULL,        -- JSON 序列化的 AgentState
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    session_id TEXT DEFAULT '',
    identity TEXT DEFAULT '{}'
);
```

### get/set/delete 操作

```rust
// structured.rs:22-43 - Get
pub fn get(&self, agent_id: AgentId, key: &str) -> OpenFangResult<Option<serde_json::Value>> {
    let conn = self.conn.lock().map_err(|e| OpenFangError::Internal(e.to_string()))?;
    let mut stmt = conn
        .prepare("SELECT value FROM kv_store WHERE agent_id = ?1 AND key = ?2")
        .map_err(|e| OpenFangError::Memory(e.to_string()))?;
    let result = stmt.query_row(rusqlite::params![agent_id.0.to_string(), key], |row| {
        let blob: Vec<u8> = row.get(0)?;
        Ok(blob)
    });
    match result {
        Ok(blob) => {
            let value: serde_json::Value = serde_json::from_slice(&blob)
                .map_err(|e| OpenFangError::Serialization(e.to_string()))?;
            Ok(Some(value))
        }
        Err(rusqlite::Error::QueryReturnedNoRows) => Ok(None),
        Err(e) => Err(OpenFangError::Memory(e.to_string())),
    }
}

// structured.rs:46-66 - Set
pub fn set(
    &self,
    agent_id: AgentId,
    key: &str,
    value: serde_json::Value,
) -> OpenFangResult<()> {
    let conn = self.conn.lock().map_err(|e| OpenFangError::Internal(e.to_string()))?;
    let blob = serde_json::to_vec(&value)
        .map_err(|e| OpenFangError::Serialization(e.to_string()))?;
    let now = Utc::now().to_rfc3339();
    conn.execute(
        "INSERT INTO kv_store (agent_id, key, value, version, updated_at)
         VALUES (?1, ?2, ?3, 1, ?4)
         ON CONFLICT(agent_id, key) DO UPDATE SET value = ?3, version = version + 1, updated_at = ?4",
        rusqlite::params![agent_id.0.to_string(), key, blob, now],
    )
    .map_err(|e| OpenFangError::Memory(e.to_string()))?;
    Ok(())
}

// structured.rs:69-80 - Delete
pub fn delete(&self, agent_id: AgentId, key: &str) -> OpenFangResult<()> {
    let conn = self.conn.lock().map_err(|e| OpenFangError::Internal(e.to_string()))?;
    conn.execute(
        "DELETE FROM kv_store WHERE agent_id = ?1 AND key = ?2",
        rusqlite::params![agent_id.0.to_string(), key],
    )
    .map_err(|e| OpenFangError::Memory(e.to_string()))?;
    Ok(())
}
```

### 序列化策略

| 数据类型 | 序列化格式 | 说明 |
|----------|------------|------|
| KV value | `serde_json::to_vec()` | JSON 二进制 |
| Agent manifest | `rmp_serde::to_vec_named()` | MessagePack（带字段名） |
| Agent state | `serde_json::to_string()` | JSON 字符串 |

**为什么 Agent manifest 用 MessagePack**:
- 更紧凑的序列化（AgentManifest 可能很大）
- `to_vec_named()` 保留字段名，支持版本迁移

---

## 6. SemanticStore — 向量搜索和语义召回

### 文件位置
`crates/openfang-memory/src/semantic.rs`

### 数据库表结构

```sql
CREATE TABLE memories (
    id TEXT PRIMARY KEY,
    agent_id TEXT NOT NULL,
    content TEXT NOT NULL,
    source TEXT NOT NULL,           -- JSON 序列化的 MemorySource
    scope TEXT NOT NULL,
    confidence REAL DEFAULT 1.0,
    metadata TEXT NOT NULL,         -- JSON 序列化的 HashMap
    created_at TEXT NOT NULL,
    accessed_at TEXT NOT NULL,
    access_count INTEGER DEFAULT 0,
    deleted INTEGER DEFAULT 0,      -- 软删除标记
    embedding BLOB                  -- f32 向量的小端字节
);
```

### remember_with_embedding — 存储记忆

```rust
// semantic.rs:44-81
pub fn remember_with_embedding(
    &self,
    agent_id: AgentId,
    content: &str,
    source: MemorySource,
    scope: &str,
    metadata: HashMap<String, serde_json::Value>,
    embedding: Option<&[f32]>,
) -> OpenFangResult<MemoryId> {
    let conn = self.conn.lock().map_err(|e| OpenFangError::Internal(e.to_string()))?;
    let id = MemoryId::new();
    let now = Utc::now().to_rfc3339();
    let source_str = serde_json::to_string(&source)
        .map_err(|e| OpenFangError::Serialization(e.to_string()))?;
    let meta_str = serde_json::to_string(&metadata)
        .map_err(|e| OpenFangError::Serialization(e.to_string()))?;
    let embedding_bytes: Option<Vec<u8>> = embedding.map(embedding_to_bytes);

    conn.execute(
        "INSERT INTO memories (id, agent_id, content, source, scope, confidence, metadata, created_at, accessed_at, access_count, deleted, embedding)
         VALUES (?1, ?2, ?3, ?4, ?5, 1.0, ?6, ?7, ?7, 0, 0, ?8)",
        rusqlite::params![
            id.0.to_string(),
            agent_id.0.to_string(),
            content,
            source_str,
            scope,
            meta_str,
            now,
            embedding_bytes,
        ],
    )
    .map_err(|e| OpenFangError::Memory(e.to_string()))?;
    Ok(id)
}
```

### recall_with_embedding — 向量召回

```rust
// semantic.rs:95-277 - 核心逻辑
pub fn recall_with_embedding(
    &self,
    query: &str,
    limit: usize,
    filter: Option<MemoryFilter>,
    query_embedding: Option<&[f32]>,
) -> OpenFangResult<Vec<MemoryFragment>> {
    let conn = self.conn.lock().map_err(|e| OpenFangError::Internal(e.to_string()))?;

    // 1. 获取候选（如果有 query_embedding，获取更多候选用于重排序）
    let fetch_limit = if query_embedding.is_some() {
        (limit * 10).max(100)  // 10 倍候选
    } else {
        limit
    };

    // 2. 构建 SQL 查询
    let mut sql = String::from(
        "SELECT id, agent_id, content, source, scope, confidence, metadata, created_at, accessed_at, access_count, embedding
         FROM memories WHERE deleted = 0"
    );

    // 2a. 文本搜索（仅在没有 embedding 时使用）
    if query_embedding.is_none() && !query.is_empty() {
        sql.push_str(&format!(" AND content LIKE ?{param_idx}"));
        params.push(Box::new(format!("%{query}%")));
    }

    // 2b. 应用过滤器
    if let Some(ref f) = filter {
        if let Some(agent_id) = f.agent_id {
            sql.push_str(&format!(" AND agent_id = ?{param_idx}"));
            params.push(Box::new(agent_id.0.to_string()));
        }
        if let Some(ref source) = f.source {
            let source_str = serde_json::to_string(source)...;
            sql.push_str(&format!(" AND source = ?{param_idx}"));
            params.push(Box::new(source_str));
        }
        // ... 其他过滤
    }

    // 3. 执行查询
    let mut fragments = Vec::new();
    for row_result in rows {
        // 解析 MemoryFragment...
    }

    // 4. 如果有 query_embedding，按余弦相似度重排序
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

### 向量召回流程

```
1. 查询嵌入 → fetch_limit = limit * 10 (获取 10 倍候选)
              ↓
2. SQL 查询 → 基础过滤（deleted=0, agent_id, source 等）
              ↓
3. 解析片段 → 将数据库行转为 MemoryFragment
              ↓
4. 余弦排序 → 计算 query_embedding 与每个 fragment.embedding 的相似度
              ↓
5. 截断结果 → 取前 limit 个最高相似度的片段
              ↓
6. 更新计数 → access_count++ 和 accessed_at = now
```

---

## 7. 余弦相似度算法详解

### 文件位置
`crates/openfang-memory/src/semantic.rs:309-328`

```rust
/// Compute cosine similarity between two vectors.
fn cosine_similarity(a: &[f32], b: &[f32]) -> f32 {
    if a.len() != b.len() || a.is_empty() {
        return 0.0;
    }

    // 1. 计算点积
    let mut dot = 0.0f32;
    for i in 0..a.len() {
        dot += a[i] * b[i];
    }

    // 2. 计算向量模长
    let mut norm_a = 0.0f32;
    let mut norm_b = 0.0f32;
    for i in 0..a.len() {
        norm_a += a[i] * a[i];
        norm_b += b[i] * b[i];
    }

    // 3. 计算余弦值
    let denom = norm_a.sqrt() * norm_b.sqrt();
    if denom < f32::EPSILON {
        0.0
    } else {
        dot / denom
    }
}
```

### 数学原理

```
余弦相似度 = cos(θ) = (A · B) / (||A|| × ||B||)

其中:
- A · B = Σ(a[i] × b[i])  (点积)
- ||A|| = sqrt(Σ(a[i]²))  (L2 范数)
- ||B|| = sqrt(Σ(b[i]²))

结果范围: [-1.0, 1.0]
- 1.0: 完全相同方向
- 0.0: 正交（无相关）
- -1.0: 完全相反方向
```

### 为什么用余弦相似度而非欧氏距离

| 指标 | 公式 | 特点 | 适用场景 |
|------|------|------|----------|
| **余弦相似度** | `(A·B) / (\|A\|×\|B\|)` | 只考虑方向，忽略长度 | 文本嵌入（长度不代表语义） |
| **欧氏距离** | `sqrt(Σ(a[i]-b[i])²)` | 考虑绝对位置 | 空间坐标、物理距离 |

**在记忆系统中**:
- 长文本和短文本可能有相同语义方向
- 余弦相似度对长度不敏感，更适合语义搜索

---

## 8. KnowledgeStore — 知识图谱

### 文件位置
`crates/openfang-memory/src/knowledge.rs`

### 数据库表结构

```sql
-- 实体表
CREATE TABLE entities (
    id TEXT PRIMARY KEY,
    entity_type TEXT NOT NULL,      -- JSON 序列化的 EntityType
    name TEXT NOT NULL,
    properties TEXT NOT NULL,       -- JSON 序列化的 HashMap
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

-- 关系表
CREATE TABLE relations (
    id TEXT PRIMARY KEY,
    source_entity TEXT NOT NULL,    -- 关联 entities.id
    relation_type TEXT NOT NULL,    -- JSON 序列化的 RelationType
    target_entity TEXT NOT NULL,    -- 关联 entities.id
    properties TEXT NOT NULL,       -- JSON 序列化的 HashMap
    confidence REAL DEFAULT 1.0,
    created_at TEXT NOT NULL,
    FOREIGN KEY (source_entity) REFERENCES entities(id),
    FOREIGN KEY (target_entity) REFERENCES entities(id)
);
```

### EntityType — 实体类型

```rust
// memory.rs:133-154
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum EntityType {
    Person,       // 人
    Organization, // 组织
    Project,      // 项目
    Concept,      // 概念
    Event,        // 事件
    Location,     // 地点
    Document,     // 文档
    Tool,         // 工具
    Custom(String), // 自定义
}
```

### RelationType — 关系类型

```rust
// memory.rs:173-199
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum RelationType {
    WorksAt,      // 工作于
    KnowsAbout,   // 了解
    RelatedTo,    // 相关
    DependsOn,    // 依赖
    OwnedBy,      // 所属
    CreatedBy,    // 创建者
    LocatedIn,    // 位于
    PartOf,       // 部分
    Uses,         // 使用
    Produces,     // 产生
    Custom(String), // 自定义
}
```

### add_entity — 添加实体

```rust
// knowledge.rs:28-51
pub fn add_entity(&self, entity: Entity) -> OpenFangResult<String> {
    let conn = self.conn.lock().map_err(|e| OpenFangError::Internal(e.to_string()))?;
    let id = if entity.id.is_empty() {
        Uuid::new_v4().to_string()  // 生成随机 ID
    } else {
        entity.id.clone()
    };

    let entity_type_str = serde_json::to_string(&entity.entity_type)...;
    let props_str = serde_json::to_string(&entity.properties)...;
    let now = Utc::now().to_rfc3339();

    conn.execute(
        "INSERT INTO entities (id, entity_type, name, properties, created_at, updated_at)
         VALUES (?1, ?2, ?3, ?4, ?5, ?5)
         ON CONFLICT(id) DO UPDATE SET name = ?3, properties = ?4, updated_at = ?5",
        rusqlite::params![id, entity_type_str, entity.name, props_str, now],
    )
    .map_err(|e| OpenFangError::Memory(e.to_string()))?;
    Ok(id)
}
```

### add_relation — 添加关系

```rust
// knowledge.rs:54-80
pub fn add_relation(&self, relation: Relation) -> OpenFangResult<String> {
    let conn = self.conn.lock().map_err(|e| OpenFangError::Internal(e.to_string()))?;
    let id = Uuid::new_v4().to_string();
    let rel_type_str = serde_json::to_string(&relation.relation)...;
    let props_str = serde_json::to_string(&relation.properties)...;
    let now = Utc::now().to_rfc3339();

    conn.execute(
        "INSERT INTO relations (id, source_entity, relation_type, target_entity, properties, confidence, created_at)
         VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7)",
        rusqlite::params![
            id,
            relation.source,
            rel_type_str,
            relation.target,
            props_str,
            relation.confidence as f64,
            now,
        ],
    )
    .map_err(|e| OpenFangError::Memory(e.to_string()))?;
    Ok(id)
}
```

### query_graph — 图模式查询

```rust
// knowledge.rs:83-185
pub fn query_graph(&self, pattern: GraphPattern) -> OpenFangResult<Vec<GraphMatch>> {
    let conn = self.conn.lock().map_err(|e| OpenFangError::Internal(e.to_string()))?;

    // 1. 构建动态 SQL
    let mut sql = String::from(
        "SELECT
            s.id, s.entity_type, s.name, s.properties, s.created_at, s.updated_at,
            r.id, r.source_entity, r.relation_type, r.target_entity, r.properties, r.confidence, r.created_at,
            t.id, t.entity_type, t.name, t.properties, t.created_at, t.updated_at
         FROM relations r
         JOIN entities s ON r.source_entity = s.id
         JOIN entities t ON r.target_entity = t.id
         WHERE 1=1"
    );

    // 2. 应用过滤条件
    if let Some(ref source) = pattern.source {
        sql.push_str(&format!(" AND (s.id = ?{idx} OR s.name = ?{idx})"));
        params.push(Box::new(source.clone()));
    }
    if let Some(ref relation) = pattern.relation {
        let rel_str = serde_json::to_string(relation)...;
        sql.push_str(&format!(" AND r.relation_type = ?{idx}"));
        params.push(Box::new(rel_str));
    }
    if let Some(ref target) = pattern.target {
        sql.push_str(&format!(" AND (t.id = ?{idx} OR t.name = ?{idx})"));
        params.push(Box::new(target.clone()));
    }

    // 3. 执行查询并解析结果
    let mut matches = Vec::new();
    for row in rows {
        matches.push(GraphMatch {
            source: parse_entity(...),
            relation: parse_relation(...),
            target: parse_entity(...),
        });
    }
    Ok(matches)
}
```

### GraphPattern — 图模式

```rust
// memory.rs:201-212
pub struct GraphPattern {
    /// Optional source entity filter.
    pub source: Option<String>,      // 源实体 ID 或名称
    /// Optional relation type filter.
    pub relation: Option<RelationType>, // 关系类型
    /// Optional target entity filter.
    pub target: Option<String>,      // 目标实体 ID 或名称
    /// Maximum traversal depth.
    pub max_depth: u32,              // 最大遍历深度
}
```

### 查询示例

```rust
// 查询"Alice 在哪里工作"
let pattern = GraphPattern {
    source: Some("Alice".to_string()),
    relation: Some(RelationType::WorksAt),
    target: None,
    max_depth: 1,
};

let matches = knowledge_store.query_graph(pattern)?;
// 返回: [GraphMatch { source: Alice, relation: WorksAt, target: Acme Corp }]

// 查询"谁了解 Rust"
let pattern = GraphPattern {
    source: None,
    relation: Some(RelationType::KnowsAbout),
    target: Some("Rust".to_string()),
    max_depth: 1,
};
```

---

## 9. MemorySubstrate — 统一实现

### 文件位置
`crates/openfang-memory/src/substrate.rs`

### 结构体定义

```rust
// substrate.rs:28-36
pub struct MemorySubstrate {
    conn: Arc<Mutex<Connection>>,
    structured: StructuredStore,
    semantic: SemanticStore,
    knowledge: KnowledgeStore,
    sessions: SessionStore,
    consolidation: ConsolidationEngine,
    usage: UsageStore,
}
```

### 组合模式设计

MemorySubstrate 使用 **组合模式** 将 6 个子系统组合成统一的 Memory trait 实现：

| 子系统 | 职责 | 对应方法 |
|--------|------|----------|
| `structured` | KV 存储和 Agent 持久化 | `get()`, `set()`, `delete()` |
| `semantic` | 向量记忆存储 | `remember()`, `recall()`, `forget()` |
| `knowledge` | 知识图谱 | `add_entity()`, `add_relation()`, `query_graph()` |
| `sessions` | Session 管理 | `create_session()`, `save_session()` |
| `consolidation` | 记忆 consolidation | `consolidate()` |
| `usage` | 使用统计 | 内部使用 |

### 11.1 双后端架构 (v0.5.2 新增)

v0.5.2 引入了 HTTP 后端支持，允许将记忆操作路由到外部 memory-api 服务（基于 PostgreSQL + pgvector + Jina AI embeddings）。

```rust
// substrate.rs — create_semantic_store
#[cfg(feature = "http-memory")]
if backend == "http" && http_url.is_some() {
    let client = MemoryApiClient::new(url, token);
    if client.health_check().is_ok() {
        return SemanticStore::new_with_http(conn, client);
    }
    // 失败时 warn 并回退到 SQLite
}
```

**路由逻辑**（SemanticStore）：
- `remember()` → 优先 HTTP，失败回退 SQLite
- `recall()` → 优先 HTTP，失败回退 SQLite

**MemoryApiClient**（`http_client.rs`）提供：
| 方法 | 端点 | 功能 |
|------|------|------|
| `health_check()` | `GET /health` | 服务可用性检查 |
| `store()` | `POST /memory/store` | 存储记忆（服务端负责 embedding） |
| `search()` | `POST /memory/search` | 语义搜索（服务端负责向量搜索） |

### 11.2 任务队列 API (v0.5.2 新增)

```rust
// substrate.rs — 四个任务队列方法
pub async fn task_post(title, description, assigned_to, created_by) -> Result<Uuid>
pub async fn task_claim(agent_id) -> Result<Option<Task>>
pub async fn task_complete(task_id, result) -> Result<()>
pub async fn task_list(status) -> Result<Vec<Task>>
```

任务队列使用 `task_queue` SQLite 表，支持优先级排序和自动清理。

### Memory Trait 实现

```rust
// substrate.rs:571-681
#[async_trait]
impl Memory for MemorySubstrate {
    async fn get(&self, agent_id: AgentId, key: &str) -> OpenFangResult<Option<serde_json::Value>> {
        let store = self.structured.clone();
        let key = key.to_string();
        tokio::task::spawn_blocking(move || store.get(agent_id, &key))
            .await
            .map_err(|e| OpenFangError::Internal(e.to_string()))?
    }

    async fn set(&self, agent_id: AgentId, key: &str, value: serde_json::Value) -> OpenFangResult<()> {
        let store = self.structured.clone();
        let key = key.to_string();
        tokio::task::spawn_blocking(move || store.set(agent_id, &key, value))
            .await
            .map_err(|e| OpenFangError::Internal(e.to_string()))?
    }

    async fn remember(&self, agent_id: AgentId, content: &str, source: MemorySource, scope: &str, metadata: HashMap<String, serde_json::Value>) -> OpenFangResult<MemoryId> {
        let store = self.semantic.clone();
        let content = content.to_string();
        let scope = scope.to_string();
        tokio::task::spawn_blocking(move || {
            store.remember(agent_id, &content, source, &scope, metadata)
        })
        .await
        .map_err(|e| OpenFangError::Internal(e.to_string()))?
    }

    async fn recall(&self, query: &str, limit: usize, filter: Option<MemoryFilter>) -> OpenFangResult<Vec<MemoryFragment>> {
        let store = self.semantic.clone();
        let query = query.to_string();
        tokio::task::spawn_blocking(move || store.recall(&query, limit, filter))
            .await
            .map_err(|e| OpenFangError::Internal(e.to_string()))?
    }

    async fn add_entity(&self, entity: Entity) -> OpenFangResult<String> {
        let store = self.knowledge.clone();
        tokio::task::spawn_blocking(move || store.add_entity(entity))
            .await
            .map_err(|e| OpenFangError::Internal(e.to_string()))?
    }

    async fn query_graph(&self, pattern: GraphPattern) -> OpenFangResult<Vec<GraphMatch>> {
        let store = self.knowledge.clone();
        tokio::task::spawn_blocking(move || store.query_graph(pattern))
            .await
            .map_err(|e| OpenFangError::Internal(e.to_string()))?
    }

    async fn consolidate(&self) -> OpenFangResult<ConsolidationReport> {
        let engine = self.consolidation.clone();
        tokio::task::spawn_blocking(move || engine.consolidate())
            .await
            .map_err(|e| OpenFangError::Internal(e.to_string()))?
    }

    // ... 其他方法
}
```

### 异步设计

**为什么使用 `spawn_blocking`**:
- SQLite 操作是同步阻塞的
- 在 tokio 异步运行时中，阻塞操作会卡住整个运行时
- `spawn_blocking` 将阻塞操作卸载到专门的线程池

### 额外方法（非 Memory Trait）

MemorySubstrate 还提供了许多额外方法用于特定功能：

```rust
// Agent 管理
pub fn save_agent(&self, entry: &AgentEntry) -> OpenFangResult<()>
pub fn load_agent(&self, agent_id: AgentId) -> OpenFangResult<Option<AgentEntry>>
pub fn load_all_agents(&self) -> OpenFangResult<Vec<AgentEntry>>

// Session 管理
pub fn create_session(&self, agent_id: AgentId) -> OpenFangResult<Session>
pub fn save_session_async(&self, session: &Session) -> OpenFangResult<()>
pub fn append_canonical(&self, agent_id: AgentId, messages: &[Message], ...) -> OpenFangResult<()>

// 嵌入操作
pub fn remember_with_embedding(&self, agent_id: AgentId, content: &str, ..., embedding: Option<&[f32]>) -> OpenFangResult<MemoryId>
pub fn recall_with_embedding(&self, query: &str, limit: usize, filter: Option<MemoryFilter>, query_embedding: Option<&[f32]>) -> OpenFangResult<Vec<MemoryFragment>>

// 任务队列
pub async fn task_post(&self, title: &str, description: &str, ...) -> OpenFangResult<String>
pub async fn task_claim(&self, agent_id: &str) -> OpenFangResult<Option<serde_json::Value>>
pub async fn task_complete(&self, task_id: &str, result: &str) -> OpenFangResult<()>
pub async fn task_list(&self, status: Option<&str>) -> OpenFangResult<Vec<serde_json::Value>>
```

---

## 10. 测试用例

### KV 测试 (StructuredStore)

```rust
// structured.rs:446-485
#[test]
fn test_kv_set_get() {
    let store = setup();
    let agent_id = AgentId::new();
    store
        .set(agent_id, "test_key", serde_json::json!("test_value"))
        .unwrap();
    let value = store.get(agent_id, "test_key").unwrap();
    assert_eq!(value, Some(serde_json::json!("test_value")));
}

#[test]
fn test_kv_update() {
    let store = setup();
    let agent_id = AgentId::new();
    store.set(agent_id, "key", serde_json::json!("v1")).unwrap();
    store.set(agent_id, "key", serde_json::json!("v2")).unwrap();
    let value = store.get(agent_id, "key").unwrap();
    assert_eq!(value, Some(serde_json::json!("v2")));
}
```

### 语义搜索测试 (SemanticStore)

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

### 知识图谱测试 (KnowledgeStore)

```rust
// knowledge.rs:282-342
#[test]
fn test_add_relation_and_query() {
    let store = setup();

    // 添加实体
    let alice_id = store.add_entity(Entity {
        id: "alice".to_string(),
        entity_type: EntityType::Person,
        name: "Alice".to_string(),
        properties: HashMap::new(),
        created_at: Utc::now(),
        updated_at: Utc::now(),
    }).unwrap();

    let company_id = store.add_entity(Entity {
        id: "acme".to_string(),
        entity_type: EntityType::Organization,
        name: "Acme Corp".to_string(),
        properties: HashMap::new(),
        created_at: Utc::now(),
        updated_at: Utc::now(),
    }).unwrap();

    // 添加关系
    store.add_relation(Relation {
        source: alice_id.clone(),
        relation: RelationType::WorksAt,
        target: company_id,
        properties: HashMap::new(),
        confidence: 0.95,
        created_at: Utc::now(),
    }).unwrap();

    // 查询图谱
    let matches = store.query_graph(GraphPattern {
        source: Some(alice_id),
        relation: Some(RelationType::WorksAt),
        target: None,
        max_depth: 1,
    }).unwrap();

    assert_eq!(matches.len(), 1);
    assert_eq!(matches[0].target.name, "Acme Corp");
}
```

### MemorySubstrate 集成测试

```rust
// substrate.rs:687-715
#[tokio::test]
async fn test_substrate_kv() {
    let substrate = MemorySubstrate::open_in_memory(0.1).unwrap();
    let agent_id = AgentId::new();
    substrate
        .set(agent_id, "key", serde_json::json!("value"))
        .await
        .unwrap();
    let val = substrate.get(agent_id, "key").await.unwrap();
    assert_eq!(val, Some(serde_json::json!("value")));
}

#[tokio::test]
async fn test_substrate_remember_recall() {
    let substrate = MemorySubstrate::open_in_memory(0.1).unwrap();
    let agent_id = AgentId::new();
    substrate
        .remember(
            agent_id,
            "Rust is a great language",
            MemorySource::Conversation,
            "episodic",
            HashMap::new(),
        )
        .await
        .unwrap();
    let results = substrate.recall("Rust", 10, None).await.unwrap();
    assert_eq!(results.len(), 1);
}
```

---

## 11. 关键设计点

### 11.1 三层分离的优势

| 方面 | 单层设计 | 三层设计 |
|------|----------|----------|
| **查询性能** | 所有查询走同一路径 | KV 精确查找 → 向量索引 → 图遍历 |
| **扩展性** | 难以针对特定类型优化 | 每层可独立优化（如向量索引、图缓存） |
| **数据隔离** | 混合存储，难以管理 | 清晰边界，便于维护 |
| **API 简洁** | 需要多个接口 | Memory trait 统一抽象 |

### 11.2 SQLite 单文件设计

**为什么不用多数据库（Redis+Postgres+Neo4j）**:
- 简化部署（单二进制 + 单文件）
- 事务一致性（跨层操作原子性）
- WAL 模式支持高并发读
- 对于中小规模应用足够（百万级记忆）

**扩展路径**:
- 结构化数据 → PostgreSQL
- 向量搜索 → Qdrant/Milvus
- 知识图谱 → Neo4j

### 11.3 软删除机制

```rust
// MemoryFragment 不直接从数据库删除
pub fn forget(&self, id: MemoryId) -> OpenFangResult<()> {
    let conn = self.conn.lock()?;
    conn.execute(
        "UPDATE memories SET deleted = 1 WHERE id = ?1",
        rusqlite::params![id.0.to_string()],
    )?;
    Ok(())
}
```

**优势**:
- 可恢复（撤销删除）
- 审计追踪（记录谁删除了什么）
- 便于 consolidation 清理（批量物理删除）

### 11.4 访问计数和衰减

```rust
// 每次 recall 都会更新访问计数
for frag in &fragments {
    let _ = conn.execute(
        "UPDATE memories SET access_count = access_count + 1, accessed_at = ?1 WHERE id = ?2",
        rusqlite::params![Utc::now().to_rfc3339(), frag.id.0.to_string()],
    );
}
```

**用途**:
- `access_count`: 热门记忆识别（经常被访问的记忆更重要）
- `accessed_at`: 最近使用（LRU 淘汰策略依据）
- ConsolidationEngine 使用这些字段计算置信度衰减

---

## 完成检查清单

- [ ] 理解三层存储架构（Structured/Semantic/Knowledge Graph）
- [ ] 掌握 MemoryFragment 结构和字段含义
- [ ] 理解 MemorySource 枚举的 6 种来源
- [ ] 掌握 MemoryFilter 过滤机制
- [ ] 理解 StructuredStore KV 操作
- [ ] 理解 SemanticStore 向量搜索（余弦相似度算法）
- [ ] 理解 KnowledgeStore 实体关系
- [ ] 掌握 MemorySubstrate 统一 API
- [ ] 了解 HTTP 后端与 SQLite 双后端架构 (v0.5.2 新增)
- [ ] 了解任务队列 API (v0.5.2 新增)

---

## 下一步

前往 [第 13 节：记忆系统 — 向量搜索](./13-memory-vector-search.md)

---

*创建时间：2026-03-15 (更新于 2026-03-29 v0.5.2)*
*OpenFang v0.5.2*
