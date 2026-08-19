# RAG Knowledge Base

通用 RAG 知识库服务的设计稿和代码骨架。

RAG 是 Retrieval-Augmented Generation，也就是先从知识库检索相关内容，再把检索结果作为上下文交给大模型生成回答。这个项目的目标是提供一个可复用的知识库底座，供不同 Agent 或业务系统通过 API 接入。

## 当前状态

- 已提交：脑暴文档、总体设计文档。
- 本次补充：Python 项目骨架、配置样例、第一阶段实施计划。
- 尚未完成：实际 API、数据库迁移、检索链路、管理后台和可运行服务。

## 规划方向

- FastAPI 服务入口
- PostgreSQL + pgvector 存储
- Collection / API Key / 权限模型
- 文档上传、切片、向量化
- 召回、重排、引用来源返回
- 面向 Agent 的 REST API

## 本地准备

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
cp .env.example .env
```

## 验证

当前仍是骨架阶段，测试目录只有占位文件：

```bash
pytest
ruff check .
```
