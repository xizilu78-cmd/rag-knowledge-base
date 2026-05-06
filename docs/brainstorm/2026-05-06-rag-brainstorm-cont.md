# RAG 知识库系统 - 脑暴续篇

> 日期：2026-05-06
> 状态：设计进行中（已完成需求澄清 + 方案选定，待完成详细设计）
> 接续：2026-05-05-rag-brainstorm.md

---

## 已锁定决策

| 维度 | 决策 |
|------|------|
| 项目定位 | 通用 RAG 底座，多个上层智能体共用（HRxiaomi 是其中之一） |
| 数据规模 | 未定，按小起步（百份级）、中型预留（千份级）设计 |
| 部署形态 | 单一 docker-compose，本地 macOS / 阿里云 ECS / 公司服务器三处一致 |
| 接口形态 | HTTP REST API |
| 隔离与鉴权 | Collection + API Key；用户级鉴权交给上层智能体 |
| 能力边界 | 仅做 ingestion + 检索 + rerank，不做生成和 prompt 组装 |
| 嵌入模型 | 智谱 AI embedding-3；代码抽象为 EmbeddingProvider 接口，可切换 |
| Rerank | **待定**：选项 A（本地 bge-reranker 容器，推荐）或选项 B（暂时跳过）；下次继续前需拍板 |
| 第一期解析 | 文本可提取 PDF（pymupdf）+ Word（python-docx）；扫描件 PDF 先检测打标不入库 |
| 第二阶段 | 表格（Excel/CSV 三层表示）+ OCR 流水线 |

---

## 选定方案：方案 B · 结构感知 + 混合检索 + rerank

核心思路：
- 解析保留**标题路径**（H1>H2>H3）和**页码**
- 切片先按章节/段落语义切，再按字数补切；每个 chunk 保留父级标题用于检索时上下文拼接
- **双路索引**：pgvector 向量召回 + PostgreSQL `pg_trgm`/`tsvector` 关键词召回（单库，无外部依赖）
- 合并去重 → rerank 重排 → 返回带 `source_path / section_path / page_no` 的 chunk 列表
- 评测集：20–50 条真实问题，每次改动跑一遍

---

## 已勾勒架构图

```
上层智能体 (HRxiaomi / 个人助手 / ...)
    ↓ HTTP REST  +  API Key  +  collection
RAG Service (FastAPI)
  ├── /ingest    ← 上传文档，同步解析入库（MVP 阶段）
  ├── /search    ← 混合召回 + rerank，返回 chunk 列表
  ├── /collections  ← collection 管理
  └── /health

存储层
  ├── PostgreSQL 17 + pgvector  ← 元数据 + chunk + 向量 + 关键词索引
  └── MinIO                     ← 原始文档

外部 API
  ├── 智谱 AI embedding-3        ← 嵌入（国内直连）
  └── Rerank：待定（bge-reranker 本地容器 或 暂时跳过）

docker-compose 服务（MVP）
  - postgres（pgvector/pgvector:pg17）
  - minio
  - rag-api（自建 Python/FastAPI 镜像）
  - bge-reranker（若选方案 A，加这个容器）
```

---

## 下次继续的起点

1. **先拍板 rerank 方案**（A 本地容器 / B 暂时跳过）
2. 继续详细设计第二段：**组件 + 目录结构**
3. 第三段：**数据模型 + API 接口定义**
4. 第四段：**错误处理 + 测试策略**
5. 写正式设计文档到 `docs/specs/2026-05-06-rag-knowledge-base-design.md`
6. 用户确认后转 writing-plans（实现计划）
