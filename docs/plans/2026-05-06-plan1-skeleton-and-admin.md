# Plan 1：项目骨架 + Collection / API Key 管理 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: 使用 `superpowers:subagent-driven-development`（推荐）或 `superpowers:executing-plans` 按任务逐项实现。步骤用 checkbox（`- [ ]`）跟踪。

**Goal:** 搭起 RAG 服务的项目骨架与最小可启动的管理面（collection 与业务 API key），不涉及 ingest/search 业务。

**Architecture:** Python 3.12 + FastAPI + SQLAlchemy 2.x async + asyncpg + Alembic；纯异步 web 层；admin 接口用环境变量 token 鉴权；业务 key 表本期建好但暂不消费（Plan 2 起接入）。三层（API → Service → Repository）单向依赖，DB 走 asyncpg；测试用 testcontainers 起真实 PG。

**Tech Stack:** Python 3.12, FastAPI 0.115+, SQLAlchemy 2.0+ (async), asyncpg, Alembic 1.13+, pydantic-settings v2, structlog, pytest + pytest-asyncio + pytest-cov, testcontainers-python, httpx, ruff。

**对应 spec：** `docs/specs/2026-05-06-rag-knowledge-base-design.md` 第 4、5、6.1、6.2、6.5、8、10 节。

---

## Task 1：项目骨架与依赖

> **状态（2026-05-06）：已完成。** 当前仓库已实际创建本任务所需骨架文件，并完成 `.venv` 初始化、`pip install -e ".[dev]"`、`ruff check src tests` 与 `pytest -q` 验证；结果分别为 `All checks passed!` 与 `no tests ran`。后续继续从 Task 2 开始。

**Files:**
- Create: `pyproject.toml`
- Create: `.env.example`
- Create: `.python-version`
- Create: `src/rag/__init__.py`
- Create: `src/rag/api/__init__.py`
- Create: `src/rag/service/__init__.py`
- Create: `src/rag/repository/__init__.py`
- Create: `src/rag/domain/__init__.py`
- Create: `src/rag/providers/__init__.py`
- Create: `src/rag/pipeline/__init__.py`
- Create: `src/rag/retrieval/__init__.py`
- Create: `tests/__init__.py`
- Create: `tests/unit/__init__.py`
- Create: `tests/integration/__init__.py`

- [x] **Step 1: 写 pyproject.toml**

```toml
[project]
name = "rag-knowledge-base"
version = "0.1.0"
description = "通用 RAG 底座服务"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "sqlalchemy[asyncio]>=2.0.30",
    "asyncpg>=0.29.0",
    "alembic>=1.13.0",
    "pgvector>=0.3.0",
    "pydantic>=2.7.0",
    "pydantic-settings>=2.3.0",
    "structlog>=24.1.0",
    "python-multipart>=0.0.9",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.2.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=5.0.0",
    "httpx>=0.27.0",
    "testcontainers[postgres]>=4.5.0",
    "ruff>=0.5.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/rag"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
markers = [
    "live: 需要外部服务（CI 默认跳过）",
]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "N", "RUF"]
ignore = ["E501"]
```

- [x] **Step 2: 写 .env.example**

```dotenv
# 服务
RAG_ADMIN_TOKEN=changeme-please
RAG_MAX_UPLOAD_MB=50
RAG_LOG_LEVEL=INFO

# 数据库
DATABASE_URL=postgresql+asyncpg://rag:rag@localhost:5432/rag

# MinIO（Plan 2 起使用）
MINIO_ENDPOINT=localhost:9000
MINIO_ACCESS_KEY=minio
MINIO_SECRET_KEY=minio12345
MINIO_BUCKET=rag

# Provider（Plan 2/3 起使用）
ZHIPU_API_KEY=
ZHIPU_EMBEDDING_MODEL=embedding-3
ZHIPU_RERANK_MODEL=rerank-001
DASHSCOPE_API_KEY=
DASHSCOPE_RERANK_MODEL=gte-rerank
BGE_RERANKER_URL=
```

- [x] **Step 3: 写 .python-version**

```
3.12
```

- [x] **Step 4: 创建包目录与空 `__init__.py`**

```bash
mkdir -p src/rag/{api,service,repository,domain,providers,pipeline,retrieval}
mkdir -p tests/{unit,integration}
touch src/rag/__init__.py
touch src/rag/{api,service,repository,domain,providers,pipeline,retrieval}/__init__.py
touch tests/__init__.py tests/unit/__init__.py tests/integration/__init__.py
```

- [x] **Step 5: 创建虚拟环境并安装**

```bash
python3.12 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
```

- [x] **Step 6: 跑 ruff 与 pytest 验证骨架**

```bash
ruff check src tests
pytest -q
```

期望：ruff 无报错；pytest 输出 `no tests ran`（因为还没有测试）。

- [ ] **Step 7: 提交**

```bash
git add pyproject.toml .env.example .python-version src/ tests/
git commit -m "chore: bootstrap project skeleton and dev deps"
```

---

## 今日收尾说明（2026-05-06）

- 已实际完成 Task 1 的 Step 1-6：骨架文件、`.venv`、依赖安装、`ruff`、`pytest`
- Task 1 的 Step 7（git commit）**尚未执行**，等待后续你明确要求再提交
- 后续建议从 **Task 2：Settings 配置加载** 继续，按 TDD 顺序执行：先写 `tests/unit/test_config.py`，确认失败，再写 `src/rag/config.py`

---

## Task 2：Settings 配置加载

**Files:**
- Create: `src/rag/config.py`
- Test: `tests/unit/test_config.py`

- [ ] **Step 1: 写失败测试**

```python
# tests/unit/test_config.py
import os
import pytest
from rag.config import Settings


def test_settings_loads_required_fields(monkeypatch):
    monkeypatch.setenv("RAG_ADMIN_TOKEN", "tok-abc")
    monkeypatch.setenv("DATABASE_URL", "postgresql+asyncpg://u:p@h/db")
    s = Settings()
    assert s.admin_token == "tok-abc"
    assert s.database_url == "postgresql+asyncpg://u:p@h/db"
    assert s.max_upload_mb == 50
    assert s.log_level == "INFO"


def test_settings_missing_admin_token_raises(monkeypatch):
    monkeypatch.delenv("RAG_ADMIN_TOKEN", raising=False)
    monkeypatch.setenv("DATABASE_URL", "postgresql+asyncpg://u:p@h/db")
    with pytest.raises(Exception):
        Settings()


def test_settings_max_upload_override(monkeypatch):
    monkeypatch.setenv("RAG_ADMIN_TOKEN", "x")
    monkeypatch.setenv("DATABASE_URL", "postgresql+asyncpg://u:p@h/db")
    monkeypatch.setenv("RAG_MAX_UPLOAD_MB", "200")
    assert Settings().max_upload_mb == 200
```

- [ ] **Step 2: 运行测试看失败**

```bash
pytest tests/unit/test_config.py -v
```

期望：`ModuleNotFoundError: No module named 'rag.config'`。

- [ ] **Step 3: 写最小实现**

```python
# src/rag/config.py
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="",
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
        case_sensitive=False,
    )

    admin_token: str = Field(alias="RAG_ADMIN_TOKEN")
    database_url: str = Field(alias="DATABASE_URL")
    max_upload_mb: int = Field(default=50, alias="RAG_MAX_UPLOAD_MB")
    log_level: str = Field(default="INFO", alias="RAG_LOG_LEVEL")

    # Plan 2/3 用，本期允许为空
    minio_endpoint: str | None = Field(default=None, alias="MINIO_ENDPOINT")
    minio_access_key: str | None = Field(default=None, alias="MINIO_ACCESS_KEY")
    minio_secret_key: str | None = Field(default=None, alias="MINIO_SECRET_KEY")
    minio_bucket: str = Field(default="rag", alias="MINIO_BUCKET")

    zhipu_api_key: str | None = Field(default=None, alias="ZHIPU_API_KEY")
    zhipu_embedding_model: str = Field(default="embedding-3", alias="ZHIPU_EMBEDDING_MODEL")
    zhipu_rerank_model: str | None = Field(default=None, alias="ZHIPU_RERANK_MODEL")
    dashscope_api_key: str | None = Field(default=None, alias="DASHSCOPE_API_KEY")
    dashscope_rerank_model: str = Field(default="gte-rerank", alias="DASHSCOPE_RERANK_MODEL")
    bge_reranker_url: str | None = Field(default=None, alias="BGE_RERANKER_URL")


def get_settings() -> Settings:
    return Settings()
```

- [ ] **Step 4: 运行测试看通过**

```bash
pytest tests/unit/test_config.py -v
```

期望：3 PASS。

- [ ] **Step 5: 提交**

```bash
git add src/rag/config.py tests/unit/test_config.py
git commit -m "feat(config): pydantic-settings based env loader"
```

---

## Task 3：异常分层

**Files:**
- Create: `src/rag/errors.py`
- Test: `tests/unit/test_errors.py`

- [ ] **Step 1: 写失败测试**

```python
# tests/unit/test_errors.py
import pytest
from rag.errors import (
    RagError, AuthError, PermissionError, NotFoundError, ValidationError,
    ConflictError, PayloadTooLarge, InternalError, DependencyError,
    EmbeddingProviderError, RerankProviderError, StorageError, DatabaseError,
)


def test_auth_error_attributes():
    e = AuthError("INVALID_API_KEY", "bad key")
    assert e.code == "INVALID_API_KEY"
    assert e.http_status == 401
    assert e.retryable is False
    assert e.message == "bad key"


def test_validation_error_default():
    e = ValidationError("EMPTY_QUERY", "query empty")
    assert e.http_status == 422


def test_dependency_error_retryable():
    e = EmbeddingProviderError("EMBEDDING_API_FAILED", "timeout")
    assert e.http_status == 503
    assert e.retryable is True
    assert isinstance(e, DependencyError)
    assert isinstance(e, RagError)


def test_to_dict_shape():
    e = NotFoundError("COLLECTION_NOT_FOUND", "no such", details={"name": "x"})
    d = e.to_dict(request_id="req-1")
    assert d == {
        "code": "COLLECTION_NOT_FOUND",
        "message": "no such",
        "details": {"name": "x"},
        "request_id": "req-1",
    }


def test_payload_too_large():
    assert PayloadTooLarge("FILE_OVER_LIMIT", "x").http_status == 413


def test_conflict_error():
    assert ConflictError("COLLECTION_FROZEN", "x").http_status == 409
```

- [ ] **Step 2: 运行测试看失败**

```bash
pytest tests/unit/test_errors.py -v
```

期望：`ModuleNotFoundError`。

- [ ] **Step 3: 写实现**

```python
# src/rag/errors.py
from __future__ import annotations
from typing import Any


class RagError(Exception):
    http_status: int = 500
    retryable: bool = False

    def __init__(self, code: str, message: str, *, details: dict[str, Any] | None = None):
        super().__init__(message)
        self.code = code
        self.message = message
        self.details = details or {}

    def to_dict(self, *, request_id: str | None = None) -> dict[str, Any]:
        return {
            "code": self.code,
            "message": self.message,
            "details": self.details,
            "request_id": request_id,
        }


class ClientError(RagError):
    http_status = 400


class AuthError(ClientError):
    http_status = 401


class PermissionError(ClientError):  # noqa: N818  与 spec 一致
    http_status = 403


class NotFoundError(ClientError):
    http_status = 404


class ValidationError(ClientError):
    http_status = 422


class ConflictError(ClientError):
    http_status = 409


class PayloadTooLarge(ClientError):
    http_status = 413


class ServerError(RagError):
    http_status = 500


class InternalError(ServerError):
    http_status = 500


class DependencyError(ServerError):
    http_status = 503
    retryable = True


class EmbeddingProviderError(DependencyError):
    pass


class RerankProviderError(DependencyError):
    pass


class StorageError(DependencyError):
    pass


class DatabaseError(DependencyError):
    pass
```

- [ ] **Step 4: 运行测试看通过**

```bash
pytest tests/unit/test_errors.py -v
```

期望：6 PASS。

- [ ] **Step 5: 提交**

```bash
git add src/rag/errors.py tests/unit/test_errors.py
git commit -m "feat(errors): layered exception hierarchy with code/message/details"
```

---

## Task 4：结构化日志与 request_id 中间件

**Files:**
- Create: `src/rag/logging.py`
- Test: `tests/unit/test_logging.py`

- [ ] **Step 1: 写失败测试**

```python
# tests/unit/test_logging.py
import logging
import structlog
from rag.logging import configure_logging, request_id_var, bind_request_id


def test_configure_logging_returns_logger():
    configure_logging("DEBUG")
    log = structlog.get_logger("test")
    assert log is not None


def test_bind_request_id_sets_contextvar():
    token = bind_request_id("req-xyz")
    try:
        assert request_id_var.get() == "req-xyz"
    finally:
        request_id_var.reset(token)


def test_request_id_default_none():
    assert request_id_var.get() is None
```

- [ ] **Step 2: 运行测试看失败**

```bash
pytest tests/unit/test_logging.py -v
```

期望：`ModuleNotFoundError`。

- [ ] **Step 3: 写实现**

```python
# src/rag/logging.py
from __future__ import annotations
import logging
import sys
from contextvars import ContextVar, Token

import structlog

request_id_var: ContextVar[str | None] = ContextVar("request_id", default=None)


def _add_request_id(_, __, event_dict: dict) -> dict:
    rid = request_id_var.get()
    if rid is not None:
        event_dict["request_id"] = rid
    return event_dict


def configure_logging(level: str = "INFO") -> None:
    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=getattr(logging, level.upper(), logging.INFO),
    )
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            _add_request_id,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(
            getattr(logging, level.upper(), logging.INFO)
        ),
        cache_logger_on_first_use=True,
    )


def bind_request_id(request_id: str) -> Token:
    return request_id_var.set(request_id)
```

- [ ] **Step 4: 运行测试看通过**

```bash
pytest tests/unit/test_logging.py -v
```

期望：3 PASS。

- [ ] **Step 5: 提交**

```bash
git add src/rag/logging.py tests/unit/test_logging.py
git commit -m "feat(logging): structlog json output with request_id contextvar"
```

---

## Task 5：Alembic 初始化

**Files:**
- Create: `alembic.ini`
- Create: `alembic/env.py`
- Create: `alembic/script.py.mako`
- Create: `alembic/versions/.gitkeep`

- [ ] **Step 1: 初始化 alembic 目录**

```bash
alembic init -t async alembic
```

会生成 `alembic.ini`、`alembic/env.py`、`alembic/script.py.mako`、`alembic/versions/`。

- [ ] **Step 2: 改 alembic.ini 关键项**

把 `sqlalchemy.url = ...` 这一行改成空：

```ini
sqlalchemy.url =
```

把 `script_location = alembic` 保留。

- [ ] **Step 3: 改 alembic/env.py 从 settings 读 URL**

把整个文件替换成：

```python
import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config

from rag.config import get_settings

config = context.config
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

settings = get_settings()
config.set_main_option("sqlalchemy.url", settings.database_url)

target_metadata = None  # MVP：手写 op.execute，不需要 autogenerate


def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection: Connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

- [ ] **Step 4: 占位 .gitkeep 防止空目录被 git 忽略**

```bash
touch alembic/versions/.gitkeep
```

- [ ] **Step 5: 提交**

```bash
git add alembic/ alembic.ini
git commit -m "chore(alembic): init async alembic with settings-driven URL"
```

---

## Task 6：第一个迁移（extensions + collections + api_keys + api_key_grants）

**Files:**
- Create: `alembic/versions/0001_init_admin_tables.py`

- [ ] **Step 1: 写迁移文件**

```python
# alembic/versions/0001_init_admin_tables.py
"""init admin tables

Revision ID: 0001
Revises:
Create Date: 2026-05-06 00:00:00
"""
from alembic import op


revision = "0001"
down_revision = None
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.execute("CREATE EXTENSION IF NOT EXISTS vector")
    op.execute("CREATE EXTENSION IF NOT EXISTS pgcrypto")

    op.execute("""
        CREATE TABLE collections (
            id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
            name          TEXT NOT NULL UNIQUE,
            description   TEXT,
            config        JSONB NOT NULL,
            embedding_dim INT  NOT NULL,
            frozen        BOOLEAN NOT NULL DEFAULT false,
            created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
        )
    """)

    op.execute("""
        CREATE TABLE api_keys (
            id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
            name       TEXT NOT NULL,
            key_hash   TEXT NOT NULL UNIQUE,
            created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
            revoked_at TIMESTAMPTZ
        )
    """)

    op.execute("""
        CREATE TABLE api_key_grants (
            api_key_id    UUID NOT NULL REFERENCES api_keys(id) ON DELETE CASCADE,
            collection_id UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
            permission    TEXT NOT NULL CHECK (permission IN ('read','write')),
            PRIMARY KEY (api_key_id, collection_id, permission)
        )
    """)


def downgrade() -> None:
    op.execute("DROP TABLE IF EXISTS api_key_grants")
    op.execute("DROP TABLE IF EXISTS api_keys")
    op.execute("DROP TABLE IF EXISTS collections")
```

- [ ] **Step 2: 起本地 PG 容器（一次性，跑迁移用）**

```bash
docker run -d --rm --name rag-pg-tmp \
  -e POSTGRES_USER=rag -e POSTGRES_PASSWORD=rag -e POSTGRES_DB=rag \
  -p 5432:5432 pgvector/pgvector:pg17
sleep 3
```

- [ ] **Step 3: 跑迁移**

```bash
RAG_ADMIN_TOKEN=x DATABASE_URL=postgresql+asyncpg://rag:rag@localhost:5432/rag \
  alembic upgrade head
```

期望：`Running upgrade  -> 0001`。

- [ ] **Step 4: 验证表存在**

```bash
docker exec rag-pg-tmp psql -U rag -d rag -c "\dt"
```

期望：列出 `alembic_version`、`api_key_grants`、`api_keys`、`collections`。

- [ ] **Step 5: 清理临时容器**

```bash
docker stop rag-pg-tmp
```

- [ ] **Step 6: 提交**

```bash
git add alembic/versions/0001_init_admin_tables.py
git commit -m "feat(db): init migration for collections / api_keys / grants"
```

---

## Task 7：SQLAlchemy ORM 模型

**Files:**
- Create: `src/rag/repository/models.py`
- Test: `tests/unit/test_models_metadata.py`

- [ ] **Step 1: 写失败测试（仅校验元数据，不连库）**

```python
# tests/unit/test_models_metadata.py
from rag.repository.models import Base, Collection, ApiKey, ApiKeyGrant


def test_tables_registered():
    names = set(Base.metadata.tables.keys())
    assert {"collections", "api_keys", "api_key_grants"} <= names


def test_collection_columns():
    cols = {c.name for c in Collection.__table__.columns}
    assert {"id", "name", "description", "config", "embedding_dim", "frozen", "created_at"} <= cols


def test_api_key_grant_pk_composite():
    pk = {c.name for c in ApiKeyGrant.__table__.primary_key.columns}
    assert pk == {"api_key_id", "collection_id", "permission"}
```

- [ ] **Step 2: 运行测试看失败**

```bash
pytest tests/unit/test_models_metadata.py -v
```

期望：`ModuleNotFoundError`。

- [ ] **Step 3: 写实现**

```python
# src/rag/repository/models.py
from __future__ import annotations
import uuid
from datetime import datetime

from sqlalchemy import (
    Boolean, CheckConstraint, ForeignKey, Integer, String, Text, text,
)
from sqlalchemy.dialects.postgresql import JSONB, TIMESTAMP, UUID
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Collection(Base):
    __tablename__ = "collections"

    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), primary_key=True, server_default=text("gen_random_uuid()"),
    )
    name: Mapped[str] = mapped_column(Text, unique=True, nullable=False)
    description: Mapped[str | None] = mapped_column(Text)
    config: Mapped[dict] = mapped_column(JSONB, nullable=False)
    embedding_dim: Mapped[int] = mapped_column(Integer, nullable=False)
    frozen: Mapped[bool] = mapped_column(
        Boolean, nullable=False, server_default=text("false"),
    )
    created_at: Mapped[datetime] = mapped_column(
        TIMESTAMP(timezone=True), nullable=False, server_default=text("now()"),
    )


class ApiKey(Base):
    __tablename__ = "api_keys"

    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), primary_key=True, server_default=text("gen_random_uuid()"),
    )
    name: Mapped[str] = mapped_column(Text, nullable=False)
    key_hash: Mapped[str] = mapped_column(Text, unique=True, nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        TIMESTAMP(timezone=True), nullable=False, server_default=text("now()"),
    )
    revoked_at: Mapped[datetime | None] = mapped_column(TIMESTAMP(timezone=True))


class ApiKeyGrant(Base):
    __tablename__ = "api_key_grants"
    __table_args__ = (
        CheckConstraint("permission IN ('read','write')", name="ck_api_key_grants_permission"),
    )

    api_key_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey("api_keys.id", ondelete="CASCADE"),
        primary_key=True,
    )
    collection_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey("collections.id", ondelete="CASCADE"),
        primary_key=True,
    )
    permission: Mapped[str] = mapped_column(String(8), primary_key=True)
```

- [ ] **Step 4: 运行测试看通过**

```bash
pytest tests/unit/test_models_metadata.py -v
```

期望：3 PASS。

- [ ] **Step 5: 提交**

```bash
git add src/rag/repository/models.py tests/unit/test_models_metadata.py
git commit -m "feat(repo): SQLAlchemy ORM for admin tables"
```

---

## Task 8：domain 数据类

**Files:**
- Create: `src/rag/domain/collection.py`
- Create: `src/rag/domain/api_key.py`
- Test: `tests/unit/test_domain.py`

- [ ] **Step 1: 写失败测试**

```python
# tests/unit/test_domain.py
import uuid
from datetime import datetime, timezone

from rag.domain.collection import Collection, CollectionConfig
from rag.domain.api_key import ApiKey, ApiKeyGrant


def test_collection_config_defaults():
    c = CollectionConfig(embedding_provider="zhipu")
    assert c.chunk_size == 600
    assert c.chunk_overlap == 80
    assert c.rerank_enabled is True
    assert c.rerank_provider == "zhipu"
    assert c.search_top_k_vector == 30
    assert c.search_top_k_keyword == 30
    assert c.max_top_k == 50


def test_collection_dataclass_round_trip():
    cid = uuid.uuid4()
    cfg = CollectionConfig(embedding_provider="zhipu")
    col = Collection(
        id=cid, name="hr", description=None,
        config=cfg, embedding_dim=1024, frozen=False,
        created_at=datetime.now(timezone.utc),
    )
    assert col.id == cid
    assert col.config.embedding_provider == "zhipu"


def test_api_key_grant_permission_value():
    g = ApiKeyGrant(collection_id=uuid.uuid4(), permission="read")
    assert g.permission == "read"
```

- [ ] **Step 2: 运行测试看失败**

```bash
pytest tests/unit/test_domain.py -v
```

期望：`ModuleNotFoundError`。

- [ ] **Step 3: 写实现：CollectionConfig 与 Collection**

```python
# src/rag/domain/collection.py
from __future__ import annotations
from dataclasses import dataclass
from datetime import datetime
from typing import Literal
from uuid import UUID

from pydantic import BaseModel, Field


RerankProvider = Literal["zhipu", "dashscope", "bge_local"]
EmbeddingProvider = Literal["zhipu"]


class CollectionConfig(BaseModel):
    chunk_size: int = Field(default=600, ge=100, le=4000)
    chunk_overlap: int = Field(default=80, ge=0, le=1000)
    embedding_provider: EmbeddingProvider = "zhipu"
    rerank_enabled: bool = True
    rerank_provider: RerankProvider | None = "zhipu"
    search_top_k_vector: int = Field(default=30, ge=1, le=200)
    search_top_k_keyword: int = Field(default=30, ge=1, le=200)
    max_top_k: int = Field(default=50, ge=1, le=200)


@dataclass(slots=True)
class Collection:
    id: UUID
    name: str
    description: str | None
    config: CollectionConfig
    embedding_dim: int
    frozen: bool
    created_at: datetime
```

- [ ] **Step 4: 写实现：ApiKey 与 ApiKeyGrant**

```python
# src/rag/domain/api_key.py
from __future__ import annotations
from dataclasses import dataclass, field
from datetime import datetime
from typing import Literal
from uuid import UUID

Permission = Literal["read", "write"]


@dataclass(slots=True)
class ApiKeyGrant:
    collection_id: UUID
    permission: Permission


@dataclass(slots=True)
class ApiKey:
    id: UUID
    name: str
    key_hash: str
    created_at: datetime
    revoked_at: datetime | None = None
    grants: list[ApiKeyGrant] = field(default_factory=list)
```

- [ ] **Step 5: 运行测试看通过**

```bash
pytest tests/unit/test_domain.py -v
```

期望：3 PASS。

- [ ] **Step 6: 提交**

```bash
git add src/rag/domain/ tests/unit/test_domain.py
git commit -m "feat(domain): dataclasses for collection / api_key"
```

---

## Task 9：测试基础设施（testcontainers PG fixture）

**Files:**
- Create: `tests/conftest.py`
- Create: `tests/integration/conftest.py`

- [ ] **Step 1: 写顶层 conftest（仅设置 asyncio loop scope）**

```python
# tests/conftest.py
import pytest


@pytest.fixture(scope="session")
def anyio_backend() -> str:
    return "asyncio"
```

- [ ] **Step 2: 写 integration conftest**

```python
# tests/integration/conftest.py
from __future__ import annotations
import os
from collections.abc import AsyncIterator

import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import (
    AsyncSession, async_sessionmaker, create_async_engine,
)
from testcontainers.postgres import PostgresContainer
from alembic import command
from alembic.config import Config


@pytest.fixture(scope="session")
def pg_container():
    with PostgresContainer("pgvector/pgvector:pg17", driver="asyncpg") as pg:
        yield pg


@pytest.fixture(scope="session")
def database_url(pg_container) -> str:
    url = pg_container.get_connection_url()
    # testcontainers 给的 URL driver 是 asyncpg，已正确
    return url


@pytest.fixture(scope="session", autouse=True)
def _run_migrations(database_url, monkeypatch_session):
    monkeypatch_session.setenv("DATABASE_URL", database_url)
    monkeypatch_session.setenv("RAG_ADMIN_TOKEN", "test-admin-token")
    cfg = Config("alembic.ini")
    cfg.set_main_option("script_location", "alembic")
    cfg.set_main_option("sqlalchemy.url", database_url)
    command.upgrade(cfg, "head")


@pytest.fixture(scope="session")
def monkeypatch_session():
    from _pytest.monkeypatch import MonkeyPatch
    mp = MonkeyPatch()
    yield mp
    mp.undo()


@pytest_asyncio.fixture
async def async_session(database_url) -> AsyncIterator[AsyncSession]:
    engine = create_async_engine(database_url, echo=False, pool_pre_ping=True)
    Session = async_sessionmaker(engine, expire_on_commit=False)
    async with Session() as s:
        yield s
        await s.rollback()
    await engine.dispose()


@pytest_asyncio.fixture
async def clean_db(async_session: AsyncSession):
    """每个测试开始前清掉 admin 三张表，保持隔离。"""
    from sqlalchemy import text
    await async_session.execute(text("TRUNCATE api_key_grants, api_keys, collections CASCADE"))
    await async_session.commit()
    yield
```

- [ ] **Step 3: 验证 fixture 能起 PG**

写一个最小冒烟测试：

```python
# tests/integration/test_smoke_pg.py
import pytest
from sqlalchemy import text


@pytest.mark.asyncio
async def test_pg_alive(async_session):
    r = await async_session.execute(text("SELECT 1"))
    assert r.scalar() == 1


@pytest.mark.asyncio
async def test_tables_exist(async_session):
    r = await async_session.execute(text(
        "SELECT count(*) FROM information_schema.tables "
        "WHERE table_name IN ('collections','api_keys','api_key_grants')"
    ))
    assert r.scalar() == 3
```

- [ ] **Step 4: 跑 integration 测试**

```bash
pytest tests/integration/test_smoke_pg.py -v
```

期望：2 PASS（首次会拉镜像，慢一些）。

- [ ] **Step 5: 提交**

```bash
git add tests/conftest.py tests/integration/conftest.py tests/integration/test_smoke_pg.py
git commit -m "test: testcontainers PG fixtures with alembic upgrade"
```

---

## Task 10：CollectionsRepository

**Files:**
- Create: `src/rag/repository/collections_repo.py`
- Test: `tests/integration/test_collections_repo.py`

- [ ] **Step 1: 写失败测试**

```python
# tests/integration/test_collections_repo.py
import pytest
from rag.domain.collection import CollectionConfig
from rag.repository.collections_repo import CollectionsRepository


@pytest.mark.asyncio
async def test_create_and_get_by_name(async_session, clean_db):
    repo = CollectionsRepository(async_session)
    cfg = CollectionConfig(embedding_provider="zhipu")
    created = await repo.create(name="hr", description="HR", config=cfg, embedding_dim=1024)
    await async_session.commit()
    fetched = await repo.get_by_name("hr")
    assert fetched is not None
    assert fetched.id == created.id
    assert fetched.config.chunk_size == 600
    assert fetched.embedding_dim == 1024
    assert fetched.frozen is False


@pytest.mark.asyncio
async def test_get_by_name_missing_returns_none(async_session, clean_db):
    repo = CollectionsRepository(async_session)
    assert await repo.get_by_name("missing") is None


@pytest.mark.asyncio
async def test_unique_name(async_session, clean_db):
    from sqlalchemy.exc import IntegrityError
    repo = CollectionsRepository(async_session)
    cfg = CollectionConfig(embedding_provider="zhipu")
    await repo.create(name="hr", description=None, config=cfg, embedding_dim=1024)
    await async_session.commit()
    await repo.create(name="hr", description=None, config=cfg, embedding_dim=1024)
    with pytest.raises(IntegrityError):
        await async_session.commit()


@pytest.mark.asyncio
async def test_freeze(async_session, clean_db):
    repo = CollectionsRepository(async_session)
    cfg = CollectionConfig(embedding_provider="zhipu")
    await repo.create(name="hr", description=None, config=cfg, embedding_dim=1024)
    await async_session.commit()
    await repo.freeze("hr")
    await async_session.commit()
    c = await repo.get_by_name("hr")
    assert c.frozen is True


@pytest.mark.asyncio
async def test_update_description_and_partial_config(async_session, clean_db):
    repo = CollectionsRepository(async_session)
    cfg = CollectionConfig(embedding_provider="zhipu")
    await repo.create(name="hr", description="old", config=cfg, embedding_dim=1024)
    await async_session.commit()

    new_cfg = cfg.model_copy(update={"max_top_k": 80})
    await repo.update("hr", description="new", config=new_cfg)
    await async_session.commit()

    c = await repo.get_by_name("hr")
    assert c.description == "new"
    assert c.config.max_top_k == 80


@pytest.mark.asyncio
async def test_delete_cascades(async_session, clean_db):
    repo = CollectionsRepository(async_session)
    cfg = CollectionConfig(embedding_provider="zhipu")
    await repo.create(name="hr", description=None, config=cfg, embedding_dim=1024)
    await async_session.commit()
    deleted = await repo.delete("hr")
    await async_session.commit()
    assert deleted is True
    assert await repo.get_by_name("hr") is None
```

- [ ] **Step 2: 运行测试看失败**

```bash
pytest tests/integration/test_collections_repo.py -v
```

期望：`ModuleNotFoundError`。

- [ ] **Step 3: 写实现**

```python
# src/rag/repository/collections_repo.py
from __future__ import annotations
from datetime import datetime
from uuid import UUID

from sqlalchemy import delete, select, update
from sqlalchemy.ext.asyncio import AsyncSession

from rag.domain.collection import Collection, CollectionConfig
from rag.repository.models import Collection as CollectionORM


def _to_domain(row: CollectionORM) -> Collection:
    return Collection(
        id=row.id,
        name=row.name,
        description=row.description,
        config=CollectionConfig.model_validate(row.config),
        embedding_dim=row.embedding_dim,
        frozen=row.frozen,
        created_at=row.created_at,
    )


class CollectionsRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def create(
        self,
        *,
        name: str,
        description: str | None,
        config: CollectionConfig,
        embedding_dim: int,
    ) -> Collection:
        row = CollectionORM(
            name=name,
            description=description,
            config=config.model_dump(mode="json"),
            embedding_dim=embedding_dim,
            frozen=False,
        )
        self.session.add(row)
        await self.session.flush()
        return _to_domain(row)

    async def get_by_name(self, name: str) -> Collection | None:
        result = await self.session.execute(
            select(CollectionORM).where(CollectionORM.name == name)
        )
        row = result.scalar_one_or_none()
        return _to_domain(row) if row else None

    async def get_by_id(self, cid: UUID) -> Collection | None:
        row = await self.session.get(CollectionORM, cid)
        return _to_domain(row) if row else None

    async def update(
        self,
        name: str,
        *,
        description: str | None = None,
        config: CollectionConfig | None = None,
    ) -> bool:
        """description / config 为 None 表示不改；都为 None 时是 no-op，返回 False。"""
        values: dict = {}
        if description is not None:
            values["description"] = description
        if config is not None:
            values["config"] = config.model_dump(mode="json")
        if not values:
            return False
        result = await self.session.execute(
            update(CollectionORM).where(CollectionORM.name == name).values(**values)
        )
        return result.rowcount > 0

    async def freeze(self, name: str) -> bool:
        result = await self.session.execute(
            update(CollectionORM)
            .where(CollectionORM.name == name)
            .values(frozen=True)
        )
        return result.rowcount > 0

    async def delete(self, name: str) -> bool:
        result = await self.session.execute(
            delete(CollectionORM).where(CollectionORM.name == name)
        )
        return result.rowcount > 0
```

- [ ] **Step 4: 运行测试看通过**

```bash
pytest tests/integration/test_collections_repo.py -v
```

期望：6 PASS。

- [ ] **Step 5: 提交**

```bash
git add src/rag/repository/collections_repo.py tests/integration/test_collections_repo.py
git commit -m "feat(repo): collections repository (CRUD + freeze)"
```

---

## Task 11：ApiKeysRepository（含 grants）

**Files:**
- Create: `src/rag/repository/api_keys_repo.py`
- Test: `tests/integration/test_api_keys_repo.py`

- [ ] **Step 1: 写失败测试**

```python
# tests/integration/test_api_keys_repo.py
import pytest
from rag.domain.api_key import ApiKeyGrant
from rag.domain.collection import CollectionConfig
from rag.repository.api_keys_repo import ApiKeysRepository
from rag.repository.collections_repo import CollectionsRepository


@pytest.mark.asyncio
async def test_create_with_grants_and_lookup(async_session, clean_db):
    crepo = CollectionsRepository(async_session)
    col = await crepo.create(
        name="hr", description=None,
        config=CollectionConfig(embedding_provider="zhipu"), embedding_dim=1024,
    )
    await async_session.commit()

    repo = ApiKeysRepository(async_session)
    key = await repo.create(
        name="hrx",
        key_hash="hash-1",
        grants=[
            ApiKeyGrant(collection_id=col.id, permission="read"),
            ApiKeyGrant(collection_id=col.id, permission="write"),
        ],
    )
    await async_session.commit()
    assert key.id is not None
    assert len(key.grants) == 2

    fetched = await repo.get_by_hash("hash-1")
    assert fetched is not None
    assert {g.permission for g in fetched.grants} == {"read", "write"}


@pytest.mark.asyncio
async def test_revoke(async_session, clean_db):
    repo = ApiKeysRepository(async_session)
    key = await repo.create(name="x", key_hash="hash-2", grants=[])
    await async_session.commit()
    assert await repo.revoke(key.id) is True
    await async_session.commit()
    fetched = await repo.get_by_hash("hash-2")
    assert fetched.revoked_at is not None


@pytest.mark.asyncio
async def test_list(async_session, clean_db):
    repo = ApiKeysRepository(async_session)
    await repo.create(name="a", key_hash="h-a", grants=[])
    await repo.create(name="b", key_hash="h-b", grants=[])
    await async_session.commit()
    items = await repo.list_all()
    assert {k.name for k in items} == {"a", "b"}


@pytest.mark.asyncio
async def test_get_by_hash_missing(async_session, clean_db):
    repo = ApiKeysRepository(async_session)
    assert await repo.get_by_hash("nope") is None
```

- [ ] **Step 2: 运行测试看失败**

```bash
pytest tests/integration/test_api_keys_repo.py -v
```

期望：`ModuleNotFoundError`。

- [ ] **Step 3: 写实现**

```python
# src/rag/repository/api_keys_repo.py
from __future__ import annotations
from datetime import datetime, timezone
from uuid import UUID

from sqlalchemy import select, update
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from rag.domain.api_key import ApiKey, ApiKeyGrant
from rag.repository.models import ApiKey as ApiKeyORM, ApiKeyGrant as GrantORM


def _to_domain(row: ApiKeyORM, grants: list[GrantORM]) -> ApiKey:
    return ApiKey(
        id=row.id,
        name=row.name,
        key_hash=row.key_hash,
        created_at=row.created_at,
        revoked_at=row.revoked_at,
        grants=[
            ApiKeyGrant(collection_id=g.collection_id, permission=g.permission)
            for g in grants
        ],
    )


class ApiKeysRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def create(
        self, *, name: str, key_hash: str, grants: list[ApiKeyGrant],
    ) -> ApiKey:
        row = ApiKeyORM(name=name, key_hash=key_hash)
        self.session.add(row)
        await self.session.flush()
        for g in grants:
            self.session.add(GrantORM(
                api_key_id=row.id,
                collection_id=g.collection_id,
                permission=g.permission,
            ))
        await self.session.flush()
        grant_rows = (await self.session.execute(
            select(GrantORM).where(GrantORM.api_key_id == row.id)
        )).scalars().all()
        return _to_domain(row, list(grant_rows))

    async def get_by_hash(self, key_hash: str) -> ApiKey | None:
        result = await self.session.execute(
            select(ApiKeyORM).where(ApiKeyORM.key_hash == key_hash)
        )
        row = result.scalar_one_or_none()
        if row is None:
            return None
        grants = (await self.session.execute(
            select(GrantORM).where(GrantORM.api_key_id == row.id)
        )).scalars().all()
        return _to_domain(row, list(grants))

    async def get_by_id(self, kid: UUID) -> ApiKey | None:
        row = await self.session.get(ApiKeyORM, kid)
        if row is None:
            return None
        grants = (await self.session.execute(
            select(GrantORM).where(GrantORM.api_key_id == row.id)
        )).scalars().all()
        return _to_domain(row, list(grants))

    async def list_all(self) -> list[ApiKey]:
        rows = (await self.session.execute(select(ApiKeyORM))).scalars().all()
        result = []
        for row in rows:
            grants = (await self.session.execute(
                select(GrantORM).where(GrantORM.api_key_id == row.id)
            )).scalars().all()
            result.append(_to_domain(row, list(grants)))
        return result

    async def revoke(self, kid: UUID) -> bool:
        result = await self.session.execute(
            update(ApiKeyORM)
            .where(ApiKeyORM.id == kid, ApiKeyORM.revoked_at.is_(None))
            .values(revoked_at=datetime.now(timezone.utc))
        )
        return result.rowcount > 0
```

- [ ] **Step 4: 运行测试看通过**

```bash
pytest tests/integration/test_api_keys_repo.py -v
```

期望：4 PASS。

- [ ] **Step 5: 提交**

```bash
git add src/rag/repository/api_keys_repo.py tests/integration/test_api_keys_repo.py
git commit -m "feat(repo): api_keys repository with grants"
```

---

## Task 12：CollectionService

**Files:**
- Create: `src/rag/service/collection_service.py`
- Create: `src/rag/service/_constants.py`
- Test: `tests/unit/test_collection_service.py`

- [ ] **Step 1: 写失败测试**

```python
# tests/unit/test_collection_service.py
import uuid
import pytest
from datetime import datetime, timezone

from rag.domain.collection import Collection, CollectionConfig
from rag.errors import ConflictError, NotFoundError, ValidationError
from rag.service.collection_service import CollectionService


class FakeRepo:
    def __init__(self):
        self.store: dict[str, Collection] = {}

    async def create(self, *, name, description, config, embedding_dim):
        col = Collection(
            id=uuid.uuid4(), name=name, description=description,
            config=config, embedding_dim=embedding_dim, frozen=False,
            created_at=datetime.now(timezone.utc),
        )
        self.store[name] = col
        return col

    async def get_by_name(self, name):
        return self.store.get(name)

    async def update(self, name, *, description=None, config=None):
        c = self.store.get(name)
        if not c:
            return False
        if description is not None:
            c.description = description
        if config is not None:
            c.config = config
        return True

    async def freeze(self, name):
        c = self.store.get(name)
        if not c:
            return False
        c.frozen = True
        return True

    async def delete(self, name):
        return self.store.pop(name, None) is not None


@pytest.mark.asyncio
async def test_create_valid_name():
    svc = CollectionService(repo=FakeRepo())
    col = await svc.create(
        name="hr_policy", description="d",
        config=CollectionConfig(embedding_provider="zhipu"),
    )
    assert col.name == "hr_policy"
    assert col.embedding_dim == 1024


@pytest.mark.asyncio
@pytest.mark.parametrize("bad", ["Hr", "1abc", "ab", "a" * 65, "ab-c", "中文"])
async def test_create_invalid_name(bad):
    svc = CollectionService(repo=FakeRepo())
    with pytest.raises(ValidationError):
        await svc.create(name=bad, description=None, config=CollectionConfig(embedding_provider="zhipu"))


@pytest.mark.asyncio
async def test_create_duplicate_name():
    svc = CollectionService(repo=FakeRepo())
    cfg = CollectionConfig(embedding_provider="zhipu")
    await svc.create(name="hr", description=None, config=cfg)
    with pytest.raises(ConflictError):
        await svc.create(name="hr", description=None, config=cfg)


@pytest.mark.asyncio
async def test_get_missing():
    svc = CollectionService(repo=FakeRepo())
    with pytest.raises(NotFoundError):
        await svc.get("missing")


@pytest.mark.asyncio
async def test_freeze_then_get():
    svc = CollectionService(repo=FakeRepo())
    cfg = CollectionConfig(embedding_provider="zhipu")
    await svc.create(name="hr", description=None, config=cfg)
    await svc.freeze("hr")
    col = await svc.get("hr")
    assert col.frozen is True


@pytest.mark.asyncio
async def test_patch_cannot_change_embedding_provider():
    svc = CollectionService(repo=FakeRepo())
    cfg = CollectionConfig(embedding_provider="zhipu")
    await svc.create(name="hr", description=None, config=cfg)
    with pytest.raises(ValidationError):
        await svc.patch("hr", description=None, config_patch={"embedding_provider": "other"})


@pytest.mark.asyncio
async def test_patch_allows_max_top_k_change():
    svc = CollectionService(repo=FakeRepo())
    cfg = CollectionConfig(embedding_provider="zhipu")
    await svc.create(name="hr", description=None, config=cfg)
    await svc.patch("hr", description="new desc", config_patch={"max_top_k": 80})
    col = await svc.get("hr")
    assert col.description == "new desc"
    assert col.config.max_top_k == 80
```

- [ ] **Step 2: 运行测试看失败**

```bash
pytest tests/unit/test_collection_service.py -v
```

期望：`ModuleNotFoundError`。

- [ ] **Step 3: 写常量与服务**

```python
# src/rag/service/_constants.py
import re

COLLECTION_NAME_RE = re.compile(r"^[a-z][a-z0-9_]{2,63}$")

# 每个 embedding_provider 的固定输出维度
EMBEDDING_DIMS: dict[str, int] = {
    "zhipu": 1024,
}

# PATCH 时不允许改的 config 字段
LOCKED_CONFIG_FIELDS: set[str] = {"embedding_provider"}
```

```python
# src/rag/service/collection_service.py
from __future__ import annotations
from typing import Any, Protocol
from uuid import UUID

from rag.domain.collection import Collection, CollectionConfig
from rag.errors import ConflictError, NotFoundError, ValidationError
from rag.service._constants import (
    COLLECTION_NAME_RE, EMBEDDING_DIMS, LOCKED_CONFIG_FIELDS,
)


class _CollectionsRepoLike(Protocol):
    async def create(self, *, name: str, description: str | None,
                     config: CollectionConfig, embedding_dim: int) -> Collection: ...
    async def get_by_name(self, name: str) -> Collection | None: ...
    async def update(self, name: str, *, description=..., config=None) -> bool: ...
    async def freeze(self, name: str) -> bool: ...
    async def delete(self, name: str) -> bool: ...


class CollectionService:
    def __init__(self, repo: _CollectionsRepoLike):
        self.repo = repo

    async def create(
        self,
        *,
        name: str,
        description: str | None,
        config: CollectionConfig,
    ) -> Collection:
        if not COLLECTION_NAME_RE.fullmatch(name):
            raise ValidationError(
                "INVALID_COLLECTION_NAME",
                "name must match ^[a-z][a-z0-9_]{2,63}$",
                details={"name": name},
            )
        if await self.repo.get_by_name(name):
            raise ConflictError("DUPLICATE_NAME", f"collection '{name}' already exists")
        if config.embedding_provider not in EMBEDDING_DIMS:
            raise ValidationError(
                "UNKNOWN_EMBEDDING_PROVIDER",
                f"unknown embedding_provider: {config.embedding_provider}",
            )
        embedding_dim = EMBEDDING_DIMS[config.embedding_provider]
        return await self.repo.create(
            name=name, description=description,
            config=config, embedding_dim=embedding_dim,
        )

    async def get(self, name: str) -> Collection:
        col = await self.repo.get_by_name(name)
        if col is None:
            raise NotFoundError("COLLECTION_NOT_FOUND", f"no collection '{name}'")
        return col

    async def patch(
        self,
        name: str,
        *,
        description: str | None,
        config_patch: dict[str, Any] | None,
    ) -> Collection:
        col = await self.get(name)
        new_config = None
        if config_patch:
            for locked in LOCKED_CONFIG_FIELDS:
                if locked in config_patch:
                    raise ValidationError(
                        "LOCKED_CONFIG_FIELD",
                        f"config field '{locked}' cannot be changed",
                    )
            new_config = col.config.model_copy(update=config_patch)
            # 触发 pydantic 校验
            new_config = CollectionConfig.model_validate(new_config.model_dump())
        await self.repo.update(name, description=description, config=new_config)
        return await self.get(name)

    async def freeze(self, name: str) -> Collection:
        ok = await self.repo.freeze(name)
        if not ok:
            raise NotFoundError("COLLECTION_NOT_FOUND", f"no collection '{name}'")
        return await self.get(name)

    async def delete(self, name: str) -> None:
        ok = await self.repo.delete(name)
        if not ok:
            raise NotFoundError("COLLECTION_NOT_FOUND", f"no collection '{name}'")
```

- [ ] **Step 4: 运行测试看通过**

```bash
pytest tests/unit/test_collection_service.py -v
```

期望：12 PASS（参数化 6 个 + 6 个）。

- [ ] **Step 5: 提交**

```bash
git add src/rag/service/_constants.py src/rag/service/collection_service.py tests/unit/test_collection_service.py
git commit -m "feat(service): collection service with name/config validation"
```

---

## Task 13：ApiKeyService（key 生成 + hash + grants）

**Files:**
- Create: `src/rag/service/api_key_service.py`
- Test: `tests/unit/test_api_key_service.py`

- [ ] **Step 1: 写失败测试**

```python
# tests/unit/test_api_key_service.py
import uuid
import pytest
from datetime import datetime, timezone

from rag.domain.api_key import ApiKey, ApiKeyGrant
from rag.domain.collection import Collection, CollectionConfig
from rag.errors import NotFoundError
from rag.service.api_key_service import ApiKeyService, hash_key


class FakeKeyRepo:
    def __init__(self):
        self.items: list[ApiKey] = []

    async def create(self, *, name, key_hash, grants):
        k = ApiKey(
            id=uuid.uuid4(), name=name, key_hash=key_hash,
            created_at=datetime.now(timezone.utc), revoked_at=None,
            grants=list(grants),
        )
        self.items.append(k)
        return k

    async def list_all(self):
        return list(self.items)

    async def revoke(self, kid):
        for k in self.items:
            if k.id == kid:
                k.revoked_at = datetime.now(timezone.utc)
                return True
        return False

    async def get_by_id(self, kid):
        for k in self.items:
            if k.id == kid:
                return k
        return None


class FakeColRepo:
    def __init__(self):
        self.cols = {}

    async def get_by_name(self, name):
        return self.cols.get(name)


def _make_col(name="hr") -> Collection:
    return Collection(
        id=uuid.uuid4(), name=name, description=None,
        config=CollectionConfig(embedding_provider="zhipu"),
        embedding_dim=1024, frozen=False,
        created_at=datetime.now(timezone.utc),
    )


@pytest.mark.asyncio
async def test_hash_key_stable():
    assert hash_key("rag_xx") == hash_key("rag_xx")
    assert hash_key("rag_xx") != hash_key("rag_yy")
    assert len(hash_key("rag_xx")) == 64  # sha256 hex


@pytest.mark.asyncio
async def test_create_returns_plaintext_once():
    crepo = FakeColRepo()
    crepo.cols["hr"] = _make_col("hr")
    svc = ApiKeyService(api_key_repo=FakeKeyRepo(), collections_repo=crepo)
    plain, key = await svc.create(
        name="agent", grants_input=[{"collection": "hr", "permission": "read"}],
    )
    assert plain.startswith("rag_")
    assert len(plain) >= 20
    assert key.key_hash == hash_key(plain)
    assert len(key.grants) == 1


@pytest.mark.asyncio
async def test_create_unknown_collection_raises():
    svc = ApiKeyService(api_key_repo=FakeKeyRepo(), collections_repo=FakeColRepo())
    with pytest.raises(NotFoundError):
        await svc.create(
            name="x", grants_input=[{"collection": "nope", "permission": "read"}],
        )


@pytest.mark.asyncio
async def test_revoke_missing_raises():
    svc = ApiKeyService(api_key_repo=FakeKeyRepo(), collections_repo=FakeColRepo())
    with pytest.raises(NotFoundError):
        await svc.revoke(uuid.uuid4())


@pytest.mark.asyncio
async def test_list_all_returns_keys():
    crepo = FakeColRepo()
    crepo.cols["hr"] = _make_col("hr")
    svc = ApiKeyService(api_key_repo=FakeKeyRepo(), collections_repo=crepo)
    await svc.create(name="a", grants_input=[{"collection": "hr", "permission": "read"}])
    await svc.create(name="b", grants_input=[])
    items = await svc.list_all()
    assert {k.name for k in items} == {"a", "b"}
```

- [ ] **Step 2: 运行测试看失败**

```bash
pytest tests/unit/test_api_key_service.py -v
```

期望：`ModuleNotFoundError`。

- [ ] **Step 3: 写实现**

```python
# src/rag/service/api_key_service.py
from __future__ import annotations
import hashlib
import secrets
from typing import Any, Protocol
from uuid import UUID

from rag.domain.api_key import ApiKey, ApiKeyGrant
from rag.domain.collection import Collection
from rag.errors import NotFoundError, ValidationError


def hash_key(plain: str) -> str:
    return hashlib.sha256(plain.encode("utf-8")).hexdigest()


def _generate_plaintext() -> str:
    return "rag_" + secrets.token_urlsafe(24)


class _ApiKeyRepoLike(Protocol):
    async def create(self, *, name: str, key_hash: str,
                     grants: list[ApiKeyGrant]) -> ApiKey: ...
    async def list_all(self) -> list[ApiKey]: ...
    async def revoke(self, kid: UUID) -> bool: ...
    async def get_by_id(self, kid: UUID) -> ApiKey | None: ...


class _ColRepoLike(Protocol):
    async def get_by_name(self, name: str) -> Collection | None: ...


class ApiKeyService:
    def __init__(self, *, api_key_repo: _ApiKeyRepoLike, collections_repo: _ColRepoLike):
        self.repo = api_key_repo
        self.cols = collections_repo

    async def create(
        self,
        *,
        name: str,
        grants_input: list[dict[str, Any]],
    ) -> tuple[str, ApiKey]:
        if not name.strip():
            raise ValidationError("INVALID_NAME", "key name cannot be empty")
        grants: list[ApiKeyGrant] = []
        for g in grants_input:
            cname = g.get("collection")
            perm = g.get("permission")
            if perm not in ("read", "write"):
                raise ValidationError(
                    "INVALID_PERMISSION",
                    "permission must be 'read' or 'write'",
                    details={"value": perm},
                )
            col = await self.cols.get_by_name(cname)
            if col is None:
                raise NotFoundError(
                    "COLLECTION_NOT_FOUND",
                    f"no collection '{cname}'",
                )
            grants.append(ApiKeyGrant(collection_id=col.id, permission=perm))

        plain = _generate_plaintext()
        key = await self.repo.create(name=name, key_hash=hash_key(plain), grants=grants)
        return plain, key

    async def list_all(self) -> list[ApiKey]:
        return await self.repo.list_all()

    async def revoke(self, kid: UUID) -> None:
        ok = await self.repo.revoke(kid)
        if not ok:
            raise NotFoundError("API_KEY_NOT_FOUND", f"no api key {kid}")
```

- [ ] **Step 4: 运行测试看通过**

```bash
pytest tests/unit/test_api_key_service.py -v
```

期望：6 PASS。

- [ ] **Step 5: 提交**

```bash
git add src/rag/service/api_key_service.py tests/unit/test_api_key_service.py
git commit -m "feat(service): api key issuance with sha256 hashing"
```

---

## Task 14：API 鉴权依赖（admin token）

**Files:**
- Create: `src/rag/api/deps.py`
- Test: `tests/unit/test_deps_admin.py`

- [ ] **Step 1: 写失败测试**

```python
# tests/unit/test_deps_admin.py
import pytest
from fastapi import HTTPException

from rag.api.deps import require_admin_token


def test_admin_ok():
    require_admin_token(authorization="Bearer good", admin_token="good")


def test_admin_missing_header():
    with pytest.raises(HTTPException) as exc:
        require_admin_token(authorization=None, admin_token="good")
    assert exc.value.status_code == 401


def test_admin_wrong_scheme():
    with pytest.raises(HTTPException) as exc:
        require_admin_token(authorization="Basic xxx", admin_token="good")
    assert exc.value.status_code == 401


def test_admin_wrong_token():
    with pytest.raises(HTTPException) as exc:
        require_admin_token(authorization="Bearer bad", admin_token="good")
    assert exc.value.status_code == 401
```

- [ ] **Step 2: 运行测试看失败**

```bash
pytest tests/unit/test_deps_admin.py -v
```

期望：`ModuleNotFoundError`。

- [ ] **Step 3: 写实现**

```python
# src/rag/api/deps.py
from __future__ import annotations
import secrets
from typing import Annotated

from fastapi import Depends, Header, HTTPException, status

from rag.config import Settings, get_settings


def _parse_bearer(authorization: str | None) -> str | None:
    if authorization is None:
        return None
    parts = authorization.split(" ", 1)
    if len(parts) != 2 or parts[0].lower() != "bearer":
        return None
    return parts[1].strip()


def require_admin_token(
    authorization: str | None,
    admin_token: str,
) -> None:
    token = _parse_bearer(authorization)
    if token is None or not secrets.compare_digest(token, admin_token):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail={"code": "INVALID_ADMIN_TOKEN", "message": "admin token required"},
        )


def admin_dep(
    authorization: Annotated[str | None, Header()] = None,
    settings: Settings = Depends(get_settings),
) -> None:
    require_admin_token(authorization, settings.admin_token)
```

- [ ] **Step 4: 运行测试看通过**

```bash
pytest tests/unit/test_deps_admin.py -v
```

期望：4 PASS。

- [ ] **Step 5: 提交**

```bash
git add src/rag/api/deps.py tests/unit/test_deps_admin.py
git commit -m "feat(api): admin bearer auth dependency"
```

---

## Task 15：Database 会话依赖

**Files:**
- Create: `src/rag/api/db.py`

- [ ] **Step 1: 写实现（这一层无单测，集成测覆盖）**

```python
# src/rag/api/db.py
from __future__ import annotations
from collections.abc import AsyncIterator

from fastapi import Depends
from sqlalchemy.ext.asyncio import (
    AsyncEngine, AsyncSession, async_sessionmaker, create_async_engine,
)

from rag.config import Settings, get_settings


_engine: AsyncEngine | None = None
_session_factory: async_sessionmaker[AsyncSession] | None = None


def init_engine(settings: Settings) -> None:
    global _engine, _session_factory
    _engine = create_async_engine(settings.database_url, pool_pre_ping=True)
    _session_factory = async_sessionmaker(_engine, expire_on_commit=False)


async def dispose_engine() -> None:
    global _engine, _session_factory
    if _engine is not None:
        await _engine.dispose()
    _engine = None
    _session_factory = None


async def get_session() -> AsyncIterator[AsyncSession]:
    if _session_factory is None:
        raise RuntimeError("engine not initialized — call init_engine() in lifespan")
    async with _session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

- [ ] **Step 2: 提交**

```bash
git add src/rag/api/db.py
git commit -m "feat(api): async db engine/session lifespan helpers"
```

---

## Task 16：Collection 路由

**Files:**
- Create: `src/rag/api/schemas.py`
- Create: `src/rag/api/routes_collections.py`
- Test: `tests/integration/test_routes_collections.py`

- [ ] **Step 1: 写 schemas（请求/响应模型）**

```python
# src/rag/api/schemas.py
from __future__ import annotations
from datetime import datetime
from typing import Any
from uuid import UUID

from pydantic import BaseModel, Field

from rag.domain.collection import CollectionConfig


class CreateCollectionReq(BaseModel):
    name: str
    description: str | None = None
    config: CollectionConfig


class PatchCollectionReq(BaseModel):
    description: str | None = None
    config: dict[str, Any] | None = None


class CollectionResp(BaseModel):
    id: UUID
    name: str
    description: str | None
    config: CollectionConfig
    embedding_dim: int
    frozen: bool
    created_at: datetime


class GrantReq(BaseModel):
    collection: str
    permission: str = Field(pattern="^(read|write)$")


class CreateApiKeyReq(BaseModel):
    name: str
    grants: list[GrantReq] = []


class CreatedApiKeyResp(BaseModel):
    id: UUID
    name: str
    key: str  # 明文，仅创建时返回


class ApiKeyResp(BaseModel):
    id: UUID
    name: str
    created_at: datetime
    revoked_at: datetime | None
    grants: list[dict]
```

- [ ] **Step 2: 写失败测试**

```python
# tests/integration/test_routes_collections.py
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport


@pytest_asyncio.fixture
async def client(database_url, monkeypatch):
    monkeypatch.setenv("DATABASE_URL", database_url)
    monkeypatch.setenv("RAG_ADMIN_TOKEN", "test-admin-token")
    from rag.main import build_app
    app = build_app()
    async with app.router.lifespan_context(app):
        async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
            yield c


HDRS = {"Authorization": "Bearer test-admin-token"}


@pytest.mark.asyncio
async def test_create_collection_201(client, clean_db):
    r = await client.post("/v1/collections", json={
        "name": "hr_policy",
        "description": "HR",
        "config": {"embedding_provider": "zhipu"},
    }, headers=HDRS)
    assert r.status_code == 201, r.text
    body = r.json()
    assert body["name"] == "hr_policy"
    assert body["embedding_dim"] == 1024
    assert body["frozen"] is False


@pytest.mark.asyncio
async def test_create_without_admin_401(client, clean_db):
    r = await client.post("/v1/collections", json={
        "name": "x", "config": {"embedding_provider": "zhipu"},
    })
    assert r.status_code == 401


@pytest.mark.asyncio
async def test_create_invalid_name_422(client, clean_db):
    r = await client.post("/v1/collections", json={
        "name": "Bad-Name",
        "config": {"embedding_provider": "zhipu"},
    }, headers=HDRS)
    assert r.status_code == 422
    assert r.json()["code"] == "INVALID_COLLECTION_NAME"


@pytest.mark.asyncio
async def test_get_collection_200_and_404(client, clean_db):
    await client.post("/v1/collections", json={
        "name": "hr", "config": {"embedding_provider": "zhipu"},
    }, headers=HDRS)
    r = await client.get("/v1/collections/hr", headers=HDRS)
    assert r.status_code == 200
    r = await client.get("/v1/collections/missing", headers=HDRS)
    assert r.status_code == 404
    assert r.json()["code"] == "COLLECTION_NOT_FOUND"


@pytest.mark.asyncio
async def test_patch_locked_field_422(client, clean_db):
    await client.post("/v1/collections", json={
        "name": "hr", "config": {"embedding_provider": "zhipu"},
    }, headers=HDRS)
    r = await client.patch("/v1/collections/hr", json={
        "config": {"embedding_provider": "other"},
    }, headers=HDRS)
    assert r.status_code == 422
    assert r.json()["code"] == "LOCKED_CONFIG_FIELD"


@pytest.mark.asyncio
async def test_freeze(client, clean_db):
    await client.post("/v1/collections", json={
        "name": "hr", "config": {"embedding_provider": "zhipu"},
    }, headers=HDRS)
    r = await client.post("/v1/collections/hr/freeze", headers=HDRS)
    assert r.status_code == 200
    assert r.json()["frozen"] is True


@pytest.mark.asyncio
async def test_delete(client, clean_db):
    await client.post("/v1/collections", json={
        "name": "hr", "config": {"embedding_provider": "zhipu"},
    }, headers=HDRS)
    r = await client.delete("/v1/collections/hr", headers=HDRS)
    assert r.status_code == 204
    r = await client.get("/v1/collections/hr", headers=HDRS)
    assert r.status_code == 404
```

- [ ] **Step 3: 运行测试看失败**

```bash
pytest tests/integration/test_routes_collections.py -v
```

期望：`ImportError: cannot import name 'build_app'`。

- [ ] **Step 4: 写路由**

```python
# src/rag/api/routes_collections.py
from __future__ import annotations
from typing import Annotated

from fastapi import APIRouter, Depends, status
from sqlalchemy.ext.asyncio import AsyncSession

from rag.api.db import get_session
from rag.api.deps import admin_dep
from rag.api.schemas import (
    CollectionResp, CreateCollectionReq, PatchCollectionReq,
)
from rag.repository.collections_repo import CollectionsRepository
from rag.service.collection_service import CollectionService


router = APIRouter(prefix="/v1/collections", tags=["collections"], dependencies=[Depends(admin_dep)])


def _service(session: AsyncSession) -> CollectionService:
    return CollectionService(repo=CollectionsRepository(session))


@router.post("", response_model=CollectionResp, status_code=status.HTTP_201_CREATED)
async def create_collection(
    req: CreateCollectionReq,
    session: Annotated[AsyncSession, Depends(get_session)],
):
    col = await _service(session).create(
        name=req.name, description=req.description, config=req.config,
    )
    return CollectionResp(**col.__dict__)


@router.get("/{name}", response_model=CollectionResp)
async def get_collection(name: str, session: Annotated[AsyncSession, Depends(get_session)]):
    col = await _service(session).get(name)
    return CollectionResp(**col.__dict__)


@router.patch("/{name}", response_model=CollectionResp)
async def patch_collection(
    name: str,
    req: PatchCollectionReq,
    session: Annotated[AsyncSession, Depends(get_session)],
):
    col = await _service(session).patch(
        name, description=req.description, config_patch=req.config,
    )
    return CollectionResp(**col.__dict__)


@router.post("/{name}/freeze", response_model=CollectionResp)
async def freeze_collection(name: str, session: Annotated[AsyncSession, Depends(get_session)]):
    col = await _service(session).freeze(name)
    return CollectionResp(**col.__dict__)


@router.delete("/{name}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_collection(name: str, session: Annotated[AsyncSession, Depends(get_session)]):
    await _service(session).delete(name)
    return None
```

- [ ] **Step 5: 跑测试（在 Task 18 main.py + 错误处理就绪后才会全绿）**

先记录预期；本步只跑 import 检查：

```bash
python -c "from rag.api.routes_collections import router; print(router.routes)"
```

期望：列出路由对象。

- [ ] **Step 6: 提交**

```bash
git add src/rag/api/schemas.py src/rag/api/routes_collections.py tests/integration/test_routes_collections.py
git commit -m "feat(api): collection routes (admin)"
```

---

## Task 17：Admin（API key）路由

**Files:**
- Create: `src/rag/api/routes_admin.py`
- Test: `tests/integration/test_routes_admin.py`

- [ ] **Step 1: 写失败测试**

```python
# tests/integration/test_routes_admin.py
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport


HDRS = {"Authorization": "Bearer test-admin-token"}


@pytest_asyncio.fixture
async def client(database_url, monkeypatch):
    monkeypatch.setenv("DATABASE_URL", database_url)
    monkeypatch.setenv("RAG_ADMIN_TOKEN", "test-admin-token")
    from rag.main import build_app
    app = build_app()
    async with app.router.lifespan_context(app):
        async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
            yield c


@pytest.mark.asyncio
async def test_create_api_key_returns_plaintext_once(client, clean_db):
    await client.post("/v1/collections", json={
        "name": "hr", "config": {"embedding_provider": "zhipu"},
    }, headers=HDRS)
    r = await client.post("/v1/admin/api-keys", json={
        "name": "hrx",
        "grants": [
            {"collection": "hr", "permission": "read"},
            {"collection": "hr", "permission": "write"},
        ],
    }, headers=HDRS)
    assert r.status_code == 201, r.text
    body = r.json()
    assert body["key"].startswith("rag_")
    assert "hash" not in body


@pytest.mark.asyncio
async def test_create_api_key_unknown_collection_404(client, clean_db):
    r = await client.post("/v1/admin/api-keys", json={
        "name": "x",
        "grants": [{"collection": "missing", "permission": "read"}],
    }, headers=HDRS)
    assert r.status_code == 404
    assert r.json()["code"] == "COLLECTION_NOT_FOUND"


@pytest.mark.asyncio
async def test_list_and_revoke(client, clean_db):
    r = await client.post("/v1/admin/api-keys", json={"name": "x", "grants": []}, headers=HDRS)
    kid = r.json()["id"]

    r = await client.get("/v1/admin/api-keys", headers=HDRS)
    assert r.status_code == 200
    assert any(item["id"] == kid for item in r.json())

    r = await client.delete(f"/v1/admin/api-keys/{kid}", headers=HDRS)
    assert r.status_code == 204

    r = await client.get("/v1/admin/api-keys", headers=HDRS)
    assert next(i for i in r.json() if i["id"] == kid)["revoked_at"] is not None
```

- [ ] **Step 2: 运行测试看失败**

```bash
pytest tests/integration/test_routes_admin.py -v
```

期望：`ImportError`。

- [ ] **Step 3: 写实现**

```python
# src/rag/api/routes_admin.py
from __future__ import annotations
from typing import Annotated
from uuid import UUID

from fastapi import APIRouter, Depends, status
from sqlalchemy.ext.asyncio import AsyncSession

from rag.api.db import get_session
from rag.api.deps import admin_dep
from rag.api.schemas import ApiKeyResp, CreateApiKeyReq, CreatedApiKeyResp
from rag.repository.api_keys_repo import ApiKeysRepository
from rag.repository.collections_repo import CollectionsRepository
from rag.service.api_key_service import ApiKeyService


router = APIRouter(prefix="/v1/admin", tags=["admin"], dependencies=[Depends(admin_dep)])


def _service(session: AsyncSession) -> ApiKeyService:
    return ApiKeyService(
        api_key_repo=ApiKeysRepository(session),
        collections_repo=CollectionsRepository(session),
    )


@router.post("/api-keys", response_model=CreatedApiKeyResp, status_code=status.HTTP_201_CREATED)
async def create_api_key(
    req: CreateApiKeyReq,
    session: Annotated[AsyncSession, Depends(get_session)],
):
    plain, key = await _service(session).create(
        name=req.name,
        grants_input=[g.model_dump() for g in req.grants],
    )
    return CreatedApiKeyResp(id=key.id, name=key.name, key=plain)


@router.get("/api-keys", response_model=list[ApiKeyResp])
async def list_api_keys(session: Annotated[AsyncSession, Depends(get_session)]):
    items = await _service(session).list_all()
    return [
        ApiKeyResp(
            id=k.id, name=k.name,
            created_at=k.created_at, revoked_at=k.revoked_at,
            grants=[{"collection_id": str(g.collection_id), "permission": g.permission} for g in k.grants],
        )
        for k in items
    ]


@router.delete("/api-keys/{kid}", status_code=status.HTTP_204_NO_CONTENT)
async def revoke_api_key(kid: UUID, session: Annotated[AsyncSession, Depends(get_session)]):
    await _service(session).revoke(kid)
    return None
```

- [ ] **Step 4: 提交（测试在 Task 19 一并跑通）**

```bash
git add src/rag/api/routes_admin.py tests/integration/test_routes_admin.py
git commit -m "feat(api): admin api-key routes"
```

---

## Task 18：Health 路由

**Files:**
- Create: `src/rag/api/routes_health.py`
- Test: `tests/integration/test_routes_health.py`

- [ ] **Step 1: 写失败测试**

```python
# tests/integration/test_routes_health.py
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport


@pytest_asyncio.fixture
async def client(database_url, monkeypatch):
    monkeypatch.setenv("DATABASE_URL", database_url)
    monkeypatch.setenv("RAG_ADMIN_TOKEN", "test-admin-token")
    from rag.main import build_app
    app = build_app()
    async with app.router.lifespan_context(app):
        async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
            yield c


@pytest.mark.asyncio
async def test_health_db_ok(client):
    r = await client.get("/v1/health")
    assert r.status_code == 200
    body = r.json()
    assert body["db"] == "ok"


@pytest.mark.asyncio
async def test_version(client):
    r = await client.get("/v1/version")
    assert r.status_code == 200
    assert "version" in r.json()
```

- [ ] **Step 2: 运行测试看失败**

```bash
pytest tests/integration/test_routes_health.py -v
```

期望：`ImportError`。

- [ ] **Step 3: 写实现**

```python
# src/rag/api/routes_health.py
from __future__ import annotations
from importlib.metadata import version, PackageNotFoundError
from typing import Annotated

from fastapi import APIRouter, Depends
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

from rag.api.db import get_session


router = APIRouter(prefix="/v1", tags=["health"])


@router.get("/health")
async def health(session: Annotated[AsyncSession, Depends(get_session)]):
    db = "ok"
    try:
        r = await session.execute(text("SELECT 1"))
        if r.scalar() != 1:
            db = "fail"
    except Exception:
        db = "fail"
    return {"db": db}


@router.get("/version")
async def app_version():
    try:
        v = version("rag-knowledge-base")
    except PackageNotFoundError:
        v = "0.0.0+local"
    return {"version": v}
```

- [ ] **Step 4: 提交**

```bash
git add src/rag/api/routes_health.py tests/integration/test_routes_health.py
git commit -m "feat(api): health and version endpoints"
```

---

## Task 19：FastAPI 入口（main.py）+ 全局错误处理 + 集成测全绿

**Files:**
- Create: `src/rag/main.py`

- [ ] **Step 1: 写实现**

```python
# src/rag/main.py
from __future__ import annotations
import uuid
from contextlib import asynccontextmanager

from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

from rag.api import db as api_db
from rag.api.routes_admin import router as admin_router
from rag.api.routes_collections import router as collections_router
from rag.api.routes_health import router as health_router
from rag.config import get_settings
from rag.errors import RagError
from rag.logging import bind_request_id, configure_logging, request_id_var


@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    configure_logging(settings.log_level)
    api_db.init_engine(settings)
    yield
    await api_db.dispose_engine()


def build_app() -> FastAPI:
    app = FastAPI(title="RAG Knowledge Base", lifespan=lifespan)

    @app.middleware("http")
    async def add_request_id(request: Request, call_next):
        rid = request.headers.get("X-Request-Id") or uuid.uuid4().hex
        token = bind_request_id(rid)
        try:
            response = await call_next(request)
        finally:
            request_id_var.reset(token)
        response.headers["X-Request-Id"] = rid
        return response

    @app.exception_handler(RagError)
    async def rag_error_handler(request: Request, exc: RagError):
        rid = request_id_var.get()
        return JSONResponse(
            status_code=exc.http_status,
            content=exc.to_dict(request_id=rid),
        )

    app.include_router(health_router)
    app.include_router(collections_router)
    app.include_router(admin_router)
    return app


app = build_app()
```

- [ ] **Step 2: 跑全部 integration 测试**

```bash
pytest tests/integration -v
```

期望：所有测试通过（health / collections / admin / repos / smoke 全绿）。

- [ ] **Step 3: 跑全部单元测试**

```bash
pytest tests/unit -v
```

期望：全绿。

- [ ] **Step 4: 启服务做手工冒烟**

```bash
docker run -d --rm --name rag-pg-tmp \
  -e POSTGRES_USER=rag -e POSTGRES_PASSWORD=rag -e POSTGRES_DB=rag \
  -p 5432:5432 pgvector/pgvector:pg17
sleep 3
RAG_ADMIN_TOKEN=tok DATABASE_URL=postgresql+asyncpg://rag:rag@localhost:5432/rag \
  alembic upgrade head
RAG_ADMIN_TOKEN=tok DATABASE_URL=postgresql+asyncpg://rag:rag@localhost:5432/rag \
  uvicorn rag.main:app --port 8000 &
SERVER_PID=$!
sleep 2
curl -s http://localhost:8000/v1/health
curl -s -X POST http://localhost:8000/v1/collections \
  -H "Authorization: Bearer tok" -H "Content-Type: application/json" \
  -d '{"name":"hr","config":{"embedding_provider":"zhipu"}}'
curl -s -X POST http://localhost:8000/v1/admin/api-keys \
  -H "Authorization: Bearer tok" -H "Content-Type: application/json" \
  -d '{"name":"hrx","grants":[{"collection":"hr","permission":"read"}]}'
kill $SERVER_PID
docker stop rag-pg-tmp
```

期望：health 返回 `{"db":"ok"}`；建 collection 返回 201 含 `embedding_dim:1024`；建 key 返回 201 含 `key: "rag_..."`。

- [ ] **Step 5: 提交**

```bash
git add src/rag/main.py
git commit -m "feat(api): assemble FastAPI app with lifespan, request-id, error handler"
```

---

## Task 20：开发用 docker-compose（便于本地起）

**Files:**
- Create: `docker-compose.yml`
- Create: `Makefile`

- [ ] **Step 1: 写 docker-compose**

```yaml
# docker-compose.yml
services:
  postgres:
    image: pgvector/pgvector:pg17
    environment:
      POSTGRES_USER: rag
      POSTGRES_PASSWORD: rag
      POSTGRES_DB: rag
    ports:
      - "5432:5432"
    volumes:
      - rag_pg:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U rag"]
      interval: 3s
      timeout: 3s
      retries: 10

volumes:
  rag_pg:
```

> 说明：rag-api 服务的 Dockerfile 留到 Plan 4 写。本期本地直接 `uvicorn rag.main:app` 跑。MinIO 同样留到 Plan 2。

- [ ] **Step 2: 写 Makefile（可选但有用）**

```makefile
.PHONY: db-up db-down migrate dev test test-unit test-integ lint

db-up:
	docker compose up -d postgres

db-down:
	docker compose down

migrate:
	alembic upgrade head

dev:
	uvicorn rag.main:app --reload --port 8000

test:
	pytest -q

test-unit:
	pytest tests/unit -q

test-integ:
	pytest tests/integration -q

lint:
	ruff check src tests
	ruff format --check src tests
```

- [ ] **Step 3: 验证 compose 起得来**

```bash
docker compose up -d postgres
docker compose ps
docker compose down
```

- [ ] **Step 4: 提交**

```bash
git add docker-compose.yml Makefile
git commit -m "chore(devx): docker-compose for local pg and Makefile"
```

---

## 完成验收

跑完所有任务后，执行以下命令应当全部通过：

```bash
ruff check src tests
pytest tests/unit -v
docker compose up -d postgres
alembic upgrade head
pytest tests/integration -v
```

**人工冒烟脚本（Task 19 Step 4）应当返回：**

- `GET /v1/health` → `{"db": "ok"}`
- `POST /v1/collections`（admin）→ 201 + `embedding_dim: 1024`
- `POST /v1/admin/api-keys` → 201 + `key: "rag_<24chars>"`（明文只此一次）
- 错过 admin token → 401 `INVALID_ADMIN_TOKEN`
- 重复名 → 409 `DUPLICATE_NAME`
- 非法名 → 422 `INVALID_COLLECTION_NAME`

---

## Plan 1 范围之外（明确不做）

| 项 | 推迟到 |
|---|---|
| `/v1/admin/cleanup` | Plan 2（依赖 documents） |
| 业务 API key 鉴权依赖（read/write 校验） | Plan 2（首次有业务接口才需要） |
| Ingest 流水线、parser、chunker | Plan 2 |
| Search、retrieval、rerank | Plan 3 |
| Dockerfile、bge-reranker compose、metrics、eval | Plan 4 |

---

## 进入 Plan 2 的前提

- 本计划全部 task ✓
- 集成测试在 testcontainers 上稳定通过
- 人工冒烟成功
- `git log --oneline | head -25` 能看到约 18 次小步提交
