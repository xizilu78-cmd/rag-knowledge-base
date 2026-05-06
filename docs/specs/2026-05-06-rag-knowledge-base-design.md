# RAG 知识库系统 · 设计文档

> 日期：2026-05-06
> 状态：设计稿 v1（待用户最终审查）
> 接续：`docs/brainstorm/2026-05-05-rag-brainstorm.md`、`docs/brainstorm/2026-05-06-rag-brainstorm-cont.md`

---

## 1. 背景与定位

构建一个**通用 RAG 底座服务**，支持上层多个智能体（HRxiaomi、个人助手等）共用同一套 ingestion 与检索能力。

**能力边界**：仅做 ingestion + 检索 + rerank，**不做生成（LLM 调用）和 prompt 组装**。生成由上层智能体自行处理。

**规模假设**：百份级起步，按千份级预留容量；100 名内部用户量级。

**部署**：单一 docker-compose，本地 macOS / 阿里云 ECS / 公司服务器三处一致。

---

## 2. 已锁定决策汇总

| 维度 | 决策 |
|---|---|
| 项目定位 | 通用 RAG 底座，多个上层智能体共用 |
| 接口形态 | HTTP REST API（FastAPI） |
| 隔离与鉴权 | Collection + 业务 API Key（read/write）；admin 用环境变量 token |
| 嵌入模型 | 智谱 AI embedding-3；代码层 `EmbeddingProvider` 接口可切换 |
| Rerank | 默认外部 API（智谱 / DashScope 二选一），代码层 `RerankProvider` 接口；本地 bge-reranker 容器作 fallback；rerank 失败时自动降级到融合分 |
| 第一期解析 | 文本可提取 PDF（pymupdf）+ Word（python-docx）+ Markdown；扫描件 PDF 检测后打标但不入库 |
| 第二阶段（不在本 spec） | 表格（Excel/CSV 三层表示）+ OCR 流水线 |
| Ingest 模式 | 异步：`/ingest` 返回 `task_id`；后台 worker 同进程线程池 + PostgreSQL 任务表，无 Celery/Redis 依赖 |
| 中文全文检索 | 应用层 jieba 分词，结果以空格连接后写入 PostgreSQL `tsvector`，不依赖 PG 中文扩展 |
| Collection | 必须预创建；创建时可配 `chunk_size` / `chunk_overlap` / `embedding_provider` / `rerank_*` 等参数；`embedding_dim` 锁定 |
| 文档去重 | 同 collection 内按文件内容 SHA256 去重，重复直接跳过返回原 `document_id` |
| 文档删除 | 软删（`documents.deleted_at`），`/v1/admin/cleanup` 做物理清理 |
| 切换 embedding 模型 | 唯一路径是新建 collection；老 collection 标记 `frozen=true`，仅允许 search |
| /search trace 字段 | 默认关闭；请求体 `"debug": true` 时返回；`latency_ms` 始终返回 |
| 单文件大小上限 | 50MB（环境变量 `RAG_MAX_UPLOAD_MB` 可调） |

---

## 3. 架构总览

```
上层智能体 (HRxiaomi / 个人助手 / ...)
    ↓ HTTP REST + Bearer <api_key> + collection name
┌──────────────────────────────────────────────────┐
│ RAG Service (FastAPI, 单容器)                     │
│  ├── API 层（路由 + 鉴权 + 参数校验）              │
│  ├── Service 层（业务编排）                       │
│  ├── Pipeline 层（ingest 流水线，由同进程 worker  │
│  │   线程池消费任务表）                            │
│  ├── Retrieval 层（向量召回 / 关键词召回 / 融合 / │
│  │   rerank，含降级）                             │
│  ├── Providers（embedding / rerank / blob 接口） │
│  └── Repository（SQLAlchemy）                    │
└──────────────────────────────────────────────────┘
       ↓                  ↓                ↓
   PostgreSQL 17      MinIO            外部 API
   + pgvector        （原始文档）      （智谱 embedding /
   （元数据 + chunk                      智谱或DashScope rerank）
    + 向量 + tsvector                   bge-reranker 本地容器
    + 任务表）                           作 fallback
```

**docker-compose 服务（MVP）**

- `postgres`（`pgvector/pgvector:pg17`）
- `minio`
- `rag-api`（自建 Python/FastAPI 镜像，内含 worker 线程池）
- `bge-reranker`（**仅在切换到本地 rerank 时启用**，第一版不带）

---

## 4. 模块结构与代码组织

```
rag-knowledge-base/
├── docker-compose.yml
├── docker-compose.test.yml         ← integration test 用
├── Dockerfile
├── pyproject.toml
├── alembic/                         ← 数据库迁移
├── tests/
│   ├── unit/                        ← 不连外部，每次 commit 跑
│   ├── integration/                 ← 真 PG + MinIO，testcontainers
│   └── eval/                        ← 评测集，发版前 / 夜间跑
└── src/rag/
    ├── api/                         ← FastAPI 路由层
    │   ├── deps.py                  ← Bearer 鉴权依赖、collection 权限校验
    │   ├── routes_collections.py
    │   ├── routes_admin.py          ← API key 管理 + cleanup
    │   ├── routes_ingest.py
    │   ├── routes_search.py
    │   └── routes_health.py
    ├── service/                     ← 业务编排，无框架耦合
    │   ├── ingest_service.py        ← 接文件 → 落 MinIO → 建 task
    │   ├── search_service.py        ← 双路召回 → 融合 → rerank（降级）
    │   ├── collection_service.py
    │   └── api_key_service.py
    ├── pipeline/                    ← ingest 流水线（worker 调用）
    │   ├── parser/
    │   │   ├── base.py              ← Parser 接口
    │   │   ├── pdf_text.py          ← pymupdf + 扫描件检测
    │   │   ├── docx.py
    │   │   └── markdown.py
    │   ├── chunker.py               ← 结构感知切片
    │   ├── tokenizer.py             ← jieba 分词
    │   └── runner.py                ← worker 主循环（FOR UPDATE SKIP LOCKED）
    ├── retrieval/
    │   ├── vector_search.py         ← pgvector 召回
    │   ├── keyword_search.py        ← tsvector 召回
    │   ├── merger.py                ← RRF 融合
    │   └── rerank/
    │       ├── base.py              ← RerankProvider 接口
    │       ├── zhipu.py
    │       ├── dashscope.py
    │       └── bge_local.py
    ├── providers/
    │   ├── embedding/{base,zhipu}.py
    │   ├── storage/{base,minio}.py
    │   └── llm/                     ← 预留，第一版不用
    ├── repository/
    │   ├── models.py                ← SQLAlchemy ORM
    │   ├── collections_repo.py
    │   ├── documents_repo.py
    │   ├── chunks_repo.py
    │   ├── tasks_repo.py
    │   └── api_keys_repo.py
    ├── domain/                      ← 纯数据类
    │   ├── chunk.py
    │   ├── document.py
    │   └── search_result.py
    ├── errors.py                    ← 异常分层
    ├── config.py                    ← Pydantic Settings，从 env 读
    ├── logging.py                   ← 结构化日志 + request_id
    └── main.py                      ← FastAPI 入口 + worker 启动
```

**关键边界**

- API/Service/Pipeline 三层，**单向依赖**，不能回绕
- 所有外部依赖（embedding、rerank、blob）走 `providers/` 接口，单元测试可用 fake 实现
- `pipeline/parser/` 第一版只支持 PDF（含扫描件检测）+ DOCX + Markdown，扫描件检测但不入库
- worker 与 API 同进程同容器，`main.py` 启动时 `asyncio.create_task` 拉一个 worker 协程组（轮询任务表）
- **不引 Celery/Redis**：百份-千份级用任务表 + 同进程线程池足够
- **不引 LangChain**：接口都很薄，自己写更可控

---

## 5. 数据模型

### 5.1 PostgreSQL Schema

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- ==== Collection ====
CREATE TABLE collections (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name          TEXT NOT NULL UNIQUE,
    description   TEXT,
    config        JSONB NOT NULL,
    embedding_dim INT  NOT NULL,
    frozen        BOOLEAN NOT NULL DEFAULT false,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ==== API Key（业务 key） ====
CREATE TABLE api_keys (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name          TEXT NOT NULL,
    key_hash      TEXT NOT NULL UNIQUE,         -- sha256(plain_key)
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at    TIMESTAMPTZ
);
CREATE TABLE api_key_grants (
    api_key_id    UUID REFERENCES api_keys(id) ON DELETE CASCADE,
    collection_id UUID REFERENCES collections(id) ON DELETE CASCADE,
    permission    TEXT NOT NULL CHECK (permission IN ('read','write')),
    PRIMARY KEY (api_key_id, collection_id, permission)
);

-- ==== Document ====
CREATE TABLE documents (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    filename      TEXT NOT NULL,
    content_hash  CHAR(64) NOT NULL,
    source_type   TEXT NOT NULL,                -- pdf / docx / markdown
    blob_key      TEXT NOT NULL,                -- MinIO object key
    size_bytes    BIGINT NOT NULL,
    page_count    INT,
    parse_status  TEXT NOT NULL,                -- pending / parsing / ready / failed / skipped_scanned
    metadata      JSONB NOT NULL DEFAULT '{}',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at    TIMESTAMPTZ,
    UNIQUE (collection_id, content_hash)
);
CREATE INDEX idx_documents_collection_active
    ON documents(collection_id) WHERE deleted_at IS NULL;

-- ==== Chunk ====
CREATE TABLE chunks (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id   UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    collection_id UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    chunk_index   INT NOT NULL,
    section_path  TEXT,
    page_no       INT,
    text          TEXT NOT NULL,
    text_tokens   TEXT NOT NULL,                -- jieba 分词后空格连接
    tsv           TSVECTOR GENERATED ALWAYS AS (to_tsvector('simple', text_tokens)) STORED,
    embedding     VECTOR,                       -- 维度由 collection.embedding_dim 决定
    char_count    INT NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_chunks_collection      ON chunks(collection_id);
CREATE INDEX idx_chunks_document        ON chunks(document_id);
CREATE INDEX idx_chunks_tsv             ON chunks USING GIN (tsv);
CREATE INDEX idx_chunks_embedding       ON chunks
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- ==== Ingest 任务 ====
CREATE TABLE ingest_tasks (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    document_id   UUID REFERENCES documents(id) ON DELETE SET NULL,
    status        TEXT NOT NULL,                -- queued / running / done / failed
    error_class   TEXT,
    error_msg     TEXT,
    progress      JSONB,                        -- {stage, current, total}
    attempts      INT NOT NULL DEFAULT 0,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    started_at    TIMESTAMPTZ,
    finished_at   TIMESTAMPTZ
);
CREATE INDEX idx_tasks_status_created ON ingest_tasks(status, created_at);
```

### 5.2 关键设计点

- `collections.name` 命名规则：`^[a-z][a-z0-9_]{2,63}$`（小写字母开头，字母/数字/下划线，长度 3-64）。
- `chunks.embedding` 类型不写死维度，由 `collections.embedding_dim` 锁定；切换 embedding 模型 → 新建 collection。
- `documents` 唯一约束 `(collection_id, content_hash)` 实现"同 collection 内容去重"。
- 软删通过 `documents.deleted_at`，索引 `idx_documents_collection_active` 加 `WHERE deleted_at IS NULL` 减少扫描。
- worker 用 `SELECT ... FOR UPDATE SKIP LOCKED` 抢任务，多副本部署也安全。
- 向量索引第一版用 `ivfflat (lists=100)`，规模上来再换 `hnsw`。

---

## 6. REST API 契约

所有接口要求 `Authorization: Bearer <api_key>`（业务 key）或 `Authorization: Bearer <admin_token>`（admin 接口）。
错误响应统一为 `{code, message, details, request_id}`。

### 6.1 Collection 管理（admin）

```
POST /v1/collections
Authorization: Bearer <admin_token>
{
  "name": "hrxiaomi_policy",
  "description": "HR 制度文档",
  "config": {
    "chunk_size": 600,
    "chunk_overlap": 80,
    "embedding_provider": "zhipu",        // 决定 embedding_dim
    "rerank_enabled": true,
    "rerank_provider": "zhipu",           // zhipu | dashscope | bge_local
    "search_top_k_vector": 30,        // 向量召回数（粗排）
    "search_top_k_keyword": 30,       // 关键词召回数（粗排）
    "max_top_k": 50                   // /search 请求 top_k 的硬上限
  }
}
→ 201 { "id":"...", "name":"...", "embedding_dim":1024, "frozen":false, ... }

GET    /v1/collections/{name}
DELETE /v1/collections/{name}            ← 级联删 documents/chunks/tasks
PATCH  /v1/collections/{name}            ← 仅可改 description / 部分 config（不能改 embedding_provider）
POST   /v1/collections/{name}/freeze     ← 标记 frozen=true
```

### 6.2 API Key 管理（admin）

```
POST /v1/admin/api-keys
{
  "name": "hrxiaomi-agent",
  "grants": [
    {"collection": "hrxiaomi_policy", "permission": "read"},
    {"collection": "hrxiaomi_policy", "permission": "write"}
  ]
}
→ 201 { "id":"...", "key":"rag_xxxxxxxx" }    // 明文只此一次

GET    /v1/admin/api-keys
DELETE /v1/admin/api-keys/{id}                  // revoke
POST   /v1/admin/cleanup                        // 物理清理软删 30 天前的 documents
```

### 6.3 Ingest（异步）

```
POST /v1/collections/{name}/ingest
Authorization: Bearer <api_key>             // 需要 write 权限
Content-Type: multipart/form-data
fields:
  file:     <二进制>
  metadata: {"source_url":"...", "author":"...", ...}    // 可选 JSON
→ 202 {
  "task_id": "...",
  "document_id": "...",
  "deduped": false                          // true 表示命中 hash 去重
}

GET /v1/ingest-tasks/{task_id}
→ 200 {
  "id":"...", "status":"running",
  "progress":{"stage":"embedding","current":12,"total":40},
  "document_id":"...", "error":null
}

DELETE /v1/collections/{name}/documents/{doc_id}    // 软删，需 write
```

### 6.4 Search

```
POST /v1/collections/{name}/search
Authorization: Bearer <api_key>             // 需要 read 权限
{
  "query": "试用期工资怎么发",
  "top_k": 8,
  "filters": {                              // 全部可选
    "document_ids": ["..."],
    "source_types": ["pdf","docx"],
    "ingested_after": "2026-01-01T00:00:00Z",
    "metadata": {"business_domain":"hr"}    // 精确匹配 documents.metadata 字段
  },
  "include": ["text","section_path","page_no","score","document"],
  "debug": false                            // true 返回 trace 详细计数
}
→ 200 {
  "query": "试用期工资怎么发",
  "results": [        // 长度 = 请求.top_k，受 collection.config.max_top_k 上限约束
    {
      "chunk_id":"...", "document_id":"...",
      "filename":"员工手册v3.pdf",
      "section_path":"第三章>3.2 试用期",
      "page_no": 12,
      "text":"...",
      "score": 0.84,
      "score_components":{"vector":0.78,"keyword":0.62,"rerank":0.84}
      // rerank 降级时：score = 融合分（RRF），score_components.rerank=null
    }
  ],
  "latency_ms": {"embed":120,"vec":18,"kw":9,"rerank":210,"total":380},
  "trace": {                                // 仅 debug=true 时存在
    "vector_recall": 30,
    "keyword_recall": 30,
    "merged": 47,
    "reranked": 8,
    "rerank_skipped": false
  }
}
```

### 6.5 Health

```
GET /v1/health    → DB / MinIO / embedding API / rerank API 各自连通性
GET /v1/version
```

---

## 7. 数据流

### 7.1 Ingest 流水线

```
[client]                [api]                  [service]              [worker]
  │   POST /ingest        │                       │                      │
  │ ─────────────────────►│                       │                      │
  │                       │ 鉴权(write)            │                      │
  │                       │ 计算 SHA256            │                      │
  │                       │ INSERT documents      │                      │
  │                       │   ON CONFLICT DO NOTHING                     │
  │                       │ 命中→deduped=true,无 task                    │
  │                       │ 否则→上传 MinIO,                              │
  │                       │       INSERT ingest_tasks(status='queued')   │
  │ ◄──────────── 202 ────│                       │                      │
  │  {task_id, doc_id}    │                       │                      │
  │                       │                       │   FOR UPDATE SKIP LOCKED
  │                       │                       │   抢一个 queued
  │                       │                       │   ────────────────► 解析(parser)
  │                       │                       │   ────────────────► 检测扫描件
  │                       │                       │     是 → status=skipped_scanned, done
  │                       │                       │   ────────────────► 切片(chunker)
  │                       │                       │   ────────────────► jieba 分词
  │                       │                       │   ────────────────► embedding(批量)
  │                       │                       │   ────────────────► INSERT chunks
  │                       │                       │   ────────────────► 更新 task=done
```

### 7.2 Search 流水线

```
[client]              [api]                [service]
  │  POST /search       │                     │
  │ ──────────────────► │                     │
  │                     │ 鉴权(read)           │
  │                     │ ──────────────────► │
  │                     │                     │ 1. embedding(query)         ←── 智谱
  │                     │                     │ 2. 向量召回 top_k_vector     ←── pgvector
  │                     │                     │    + 关键词召回 top_k_keyword←── tsvector + jieba(query)
  │                     │                     │ 3. RRF 融合 → 候选集
  │                     │                     │ 4. rerank(query, 候选)       ←── 外部 API / bge
  │                     │                     │    失败 → 降级:返回融合分top_k,
  │                     │                     │            trace.rerank_skipped=true
  │                     │                     │ 5. 拼装 chunk + document 元数据
  │ ◄──────────── 200 ──│                     │
```

---

## 8. 错误处理

### 8.1 异常分层

```
RagError(code, http_status, retryable)
├── ClientError (4xx)
│   ├── AuthError              401  INVALID_API_KEY / KEY_REVOKED
│   ├── PermissionError        403  NO_PERMISSION
│   ├── NotFoundError          404  COLLECTION_NOT_FOUND / DOCUMENT_NOT_FOUND / TASK_NOT_FOUND
│   ├── ValidationError        422  INVALID_FILE_TYPE / EMPTY_QUERY / TOP_K_TOO_LARGE
│   ├── ConflictError          409  COLLECTION_FROZEN / DUPLICATE_NAME
│   └── PayloadTooLarge        413  FILE_OVER_LIMIT
└── ServerError (5xx)
    ├── InternalError          500  UNEXPECTED
    └── DependencyError        503  retryable=true
        ├── EmbeddingProviderError    EMBEDDING_API_FAILED
        ├── RerankProviderError       RERANK_API_FAILED
        ├── StorageError              MINIO_UNAVAILABLE
        └── DatabaseError             DB_UNAVAILABLE
```

### 8.2 关键策略

| 场景 | 行为 |
|---|---|
| 智谱 embedding API 失败 | 指数退避重试 3 次（1s/3s/9s）；仍失败 → task `failed`，error_class=`EMBEDDING_API_FAILED`，可手动重跑 |
| Rerank API 失败 | **自动降级**：返回融合分 top_k；`trace.rerank_skipped=true`；记录 metric `rerank_skipped_total` |
| PDF 检测到扫描件 | `documents.parse_status='skipped_scanned'`，task `done`（非 failed） |
| 切片为 0（空文档） | task `failed`，error_class=`EMPTY_DOCUMENT` |
| Worker 崩溃 | 任务 `started_at` 超过 10 分钟仍 `running` → 看护把 status 改回 `queued`，attempts+=1；attempts ≥ 3 直接 `failed` |
| /search query 为空 | 422 `EMPTY_QUERY` |
| top_k > collection.max_top_k（默认 50） | 422 `TOP_K_TOO_LARGE` |
| 单文件 > 50MB | 413 `FILE_OVER_LIMIT`（`RAG_MAX_UPLOAD_MB` 可调） |
| 向 frozen collection 发 ingest | 409 `COLLECTION_FROZEN` |

### 8.3 日志与可观测

- 每请求生成 `request_id`，贯穿 service/pipeline/repository，结构化 JSON 日志输出 stdout
- Prometheus metrics 暴露在 `/v1/metrics`：
  - `ingest_tasks_total{status}`
  - `ingest_duration_seconds_bucket`
  - `search_latency_ms_bucket`
  - `embedding_api_failures_total`
  - `rerank_skipped_total`
- 第一版不接 Sentry / OTLP；公司服务器迁移阶段再评估

---

## 9. 测试策略

### 9.1 三层结构

```
tests/
├── unit/                            ← 不连外部，每次 commit 跑
│   ├── test_chunker.py
│   ├── test_tokenizer.py            ← jieba 结果稳定性
│   ├── test_merger.py               ← RRF 融合
│   ├── test_parser_pdf.py           ← fixture 小 PDF + 扫描件检测
│   ├── test_dedup.py                ← SHA256 去重
│   └── test_errors.py
├── integration/                     ← 真 PG + MinIO（compose.test.yml / testcontainers）
│   ├── test_ingest_flow.py          ← 上传 → poll task → 查 chunks
│   ├── test_search_hybrid.py
│   ├── test_collection_frozen.py
│   ├── test_api_key_grants.py
│   ├── test_rerank_fallback.py      ← 模拟 rerank 失败,验证降级
│   └── conftest.py
└── eval/
    ├── eval_dataset.yaml            ← 20–50 条真实 Q/期望
    ├── runner.py                    ← recall@5/10、MRR、p50/p95
    └── baseline.json                ← 历史基线
```

### 9.2 Provider 在测试里的处理

- `EmbeddingProvider` / `RerankProvider` 提供 fake 实现（确定性 hash → 向量、按文本长度逆序）
- 单元测试和 integration 默认用 fake；接真智谱的测试单独标 `@pytest.mark.live`，CI 默认跳过
- `BlobStore` 用 MinIO testcontainer，不写本地 fake

### 9.3 评测集（质量护栏）

```yaml
# eval_dataset.yaml 示例
- id: q001
  collection: hrxiaomi_policy
  query: 试用期工资怎么发
  expected_docs: ["员工手册v3.pdf"]
  expected_sections: ["第三章>3.2 试用期"]
  must_contain_keywords: ["试用期", "工资"]
```

**指标**：`recall@5` / `recall@10` / `MRR` / `latency_p50` / `latency_p95`。

**CI gate**：每次改切片/检索/rerank 必须跑评测集，分数不能掉。

**第一版来源**：HRxiaomi 第一批要接的业务文档（员工手册/制度/FAQ），从真实问题里挑 20 条起步。

---

## 10. 配置与部署

### 10.1 关键环境变量

```
# 服务
RAG_ADMIN_TOKEN=<random-string>           # 必填
RAG_MAX_UPLOAD_MB=50

# 数据库
DATABASE_URL=postgresql+asyncpg://rag:rag@postgres:5432/rag

# MinIO
MINIO_ENDPOINT=minio:9000
MINIO_ACCESS_KEY=...
MINIO_SECRET_KEY=...
MINIO_BUCKET=rag

# 智谱
ZHIPU_API_KEY=...
ZHIPU_EMBEDDING_MODEL=embedding-3
ZHIPU_RERANK_MODEL=...                     # 若用智谱 rerank

# DashScope（备选 rerank）
DASHSCOPE_API_KEY=...
DASHSCOPE_RERANK_MODEL=gte-rerank

# 本地 bge（fallback）
BGE_RERANKER_URL=http://bge-reranker:8000
```

### 10.2 docker-compose（MVP）

服务：
- `postgres`（pgvector/pgvector:pg17，挂载 volume）
- `minio`（挂载 volume）
- `rag-api`（自建镜像，依赖 postgres / minio）

`bge-reranker` 容器**默认不启用**，按需通过 profile 加。

### 10.3 第一次启动

1. `docker-compose up -d postgres minio`
2. `alembic upgrade head`
3. `docker-compose up -d rag-api`
4. 用 `RAG_ADMIN_TOKEN` 调 `/v1/collections` 建第一个 collection
5. 调 `/v1/admin/api-keys` 发业务 key 给上层智能体

---

## 11. 不在本期范围

明确划走，避免范围蔓延：

- 表格（Excel/CSV）三层表示
- OCR 流水线（扫描件入库）
- 文档版本管理（同名文档新版本）
- 用户级鉴权（由上层智能体处理）
- LLM 生成 / prompt 组装
- 权限标签 / 数据敏感分级
- 增量更新 / 变更监听
- 多副本 worker 部署调优（设计已支持，但第一版单容器）

---

## 12. 后续路线（仅参考，不在本 spec 设计中）

1. 第二阶段：表格三层处理 + Excel/CSV 解析
2. 第三阶段：OCR 流水线（PaddleOCR 容器）+ OCR 置信度打标
3. 切公司服务器：评估 Sentry / OTLP；评估 OpenSearch 是否替换 tsvector
4. 文档版本管理（`documents.version` / `supersedes`）

---

## 附 · 决策来源

- `docs/brainstorm/2026-05-05-rag-brainstorm.md`：基础原则、文档分类、切片、混合检索、rerank、元数据、技术选型、落地路线、踩坑
- `docs/brainstorm/2026-05-06-rag-brainstorm-cont.md`：项目定位、规模、部署、接口、隔离、能力边界、嵌入模型、解析范围、方案 B 选定
- 本 spec 撰写过程：rerank 方案、ingest 异步、中文检索、collection 预创建、SHA256 去重、API key 颗粒度、embedding 切换、软删、trace 字段、文件大小、rerank 降级
