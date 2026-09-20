# 金融研究 Agent 项目说明书

> 本文档用于指导 AI 编码助手（Codex / Claude Code）生成项目脚手架与核心实现。  
> 目标：构建一个**可审计、可追溯、多源异构数据、混合检索、带验证风控**的金融研究 Agent。

---

## 1. 项目目标

构建一个金融研究 Agent，能够：

- 接入多源金融数据（财报、公告、新闻、行情、宏观）
- 解析非结构化文档，结构化存储
- 支持混合检索（稀疏 + 稠密）+ Rerank
- 使用 LangGraph 编排多 Agent 协作
- 外置数值计算到 Python 沙箱
- 分层 Memory 管理
- 验证/风控层拦截幻觉与合规问题
- 生成带段落级引用的研究报告
- 全链路审计日志，支持回放

---

## 2. 整体架构

```
数据源 → 解析/清洗 → Chunk → Embedding → 向量库
   → 混合检索 + Rerank → Agent 编排 → Tools
   → Memory → 验证/风控 → Report → 审计日志
```

### 核心原则

- **数值计算必须走代码**，不让 LLM 心算
- **每个结论绑定来源**，可追溯
- **显式状态机**，LLM 只在节点内决策
- **分层验证**：规则 → 交叉 → 语义
- **全链路审计**：trace_id 回放

---

## 3. 模块设计

### 3.1 数据源与解析

**技术选型**

| 类型 | 来源 | 解析方案 |
|---|---|---|
| 结构化财务 | Wind、Tushare、AKShare、SEC EDGAR | SQL / API 直连 |
| 公告/财报 PDF | 巨潮、交易所、SEC | PyMuPDF、pdfplumber、Unstructured |
| 新闻/研报 | 新闻 API、券商研报库 | Trafilatura、Newspaper3k |
| 行情 | 交易所、行情 API | WebSocket / REST |
| 宏观 | 统计局、央行、FRED | API + 定时任务 |

**接口设计**

```python
class DataConnector:
    def fetch(self, params: dict) -> pd.DataFrame: ...
    def get_metadata(self) -> dict: ...
```

**工程要点**

- 每个数据源独立 Connector，统一 schema
- 增量更新：时间戳 + 唯一 ID 去重
- 数据版本：记录 `report_period`、`announce_date`、`version`
- 失败降级：主源 → 备源 → 缓存 → 明确报错
- 原始文件存对象存储，解析结果存 DB

---

### 3.2 Chunk

**技术选型**

| 策略 | 方案 |
|---|---|
| 通用递归 | LangChain `RecursiveCharacterTextSplitter` |
| 语义切分 | LlamaIndex `SemanticSplitter` |
| 结构化 | 按标题/章节/段落 |
| 表格 | pdfplumber / Camelot 单独抽取 |
| 父子块 | 小块检索，大块生成 |
| 上下文增强 | Anthropic Contextual Retrieval |

**工程要点**

- 按文档类型走不同 pipeline
- 每块元数据：`company`、`report_period`、`announce_date`、`source`、`page`、`section`
- 表格转 Markdown/JSON，单独索引
- 父子块：子块 200–500 token，父块 1000–2000 token
- 重叠 10%–20%
- 版本管理：旧块标记失效，不删除

---

### 3.3 Embedding

**技术选型**

| 模型 | 维度 | 定位 |
|---|---|---|
| Qwen3-Embedding-4B | — | 中文性价比最优 |
| BGE-M3 | 1024 | 混合检索首选 |
| OpenAI text-embedding-3-large | 3072 | 英文通用 |
| Fin-E5 | — | 金融领域专用 |

**工程要点**

- 元数据拼接进 Embedding 文本：
  ```
  [公司: 贵州茅台] [报告期: 2024年报] [章节: 管理层讨论]
  正文：本期营业收入...
  ```
- Matryoshka 截断：降低存储成本
- 版本管理：记录 `embedding_model`、`embedding_version`
- 批量推理：vLLM / TEI
- 评估：FinMTEB + 业务 Hit@K

---

### 3.4 向量库

**技术选型**

| 方案 | 适用规模 | 特点 |
|---|---|---|
| pgvector | < 1000 万 | 事务强，零新组件 |
| Qdrant | 100 万 - 1 亿 | 延迟低，过滤检索强 |
| Milvus | > 1 亿 | 分布式，GPU 加速 |

**工程要点**

- 过滤检索：payload 存 `company_id`、`report_period`、`section`
- 混合检索：Qdrant 内置稀疏向量 + BM25
- 量化压缩：标量量化 / 乘积量化，内存降 70%
- 索引：HNSW（延迟敏感）、IVF（写入频繁）
- 评估：Recall@K、P95/P99、Filter Performance

---

### 3.5 检索与 Rerank

**技术选型**

| 环节 | 方案 |
|---|---|
| 稀疏 | BM25、SPLADE |
| 稠密 | BGE-M3、Qwen3-Embedding |
| 融合 | RRF（k=60） |
| Rerank | bge-reranker-v2-m3、Cohere Rerank v3.5 |

**推荐链路**

```
Query → BM25 + 向量并行 → RRF 融合 → Rerank → Top-5
```

**工程要点**

- 混合检索需调权，不能朴素等权融合
- Query Rewrite：LLM 改写为 2–3 个表述
- 多跳检索：根据已有信息动态生成下一轮 query
- 评估：Hit@5、MRR、Recall@5、NDCG@K
- 金融场景：数值准确率、时效性

---

### 3.6 Agent 编排

**技术选型**

| 框架 | 定位 |
|---|---|
| LangGraph | 金融首选，状态机 + 图 |
| OpenAI Agents SDK | 轻量多 Agent |
| Pydantic AI | 类型安全工具层 |
| 自研状态机 | 完全可控 |

**架构**

```
用户 Query → Planner → Router → 子 Agent → Validation → Report → Audit
```

**工程要点**

- State 用 TypedDict 严格 schema
- Checkpointer 持久化到 Postgres，支持断点续跑
- 人在回路：`interrupt_before` / `interrupt_after`
- Supervisor 模式，子 Agent 不直接通信
- 错误处理：重试、降级、明确报错
- 成本控制：`recursion_limit`、token 预算

---

### 3.7 Tools

**技术选型**

| 工具 | 实现 |
|---|---|
| 检索 | 向量 + BM25 封装 |
| SQL | Text-to-SQL + 参数化 |
| Python 沙箱 | E2B / Docker + RestrictedPython |
| 行情 | Tushare、AKShare、Polygon |
| 估值 | DCF、可比公司，封装函数 |
| 图表 | Matplotlib / Plotly |

**Tool Calling 生命周期**

```
LLM 输出 tool_call → 参数校验 → 权限检查 → 执行（超时/重试/幂等）
→ 结果格式化 → 返回 LLM → 循环
```

**工程要点**

- JSON Schema 定义工具
- 数值计算全部外置
- 沙箱：禁网、禁写、限时、限内存、白名单库
- 错误分类处理：参数错重试，超时降级
- 可观测：记录工具名、参数、耗时、来源、时间戳
- 协议：OpenAI function calling + MCP

---

### 3.8 Memory

**技术选型**

| 类型 | 存储 |
|---|---|
| 工作记忆 | LangGraph State |
| 会话记忆 | Redis，TTL 24h |
| 长期语义 | 向量库 |
| 结构化 | PostgreSQL |
| 情景记忆 | 向量库 + 时间戳 |

**工程要点**

- 分层架构：工作 / 会话 / 长期 / 情景
- 写入策略：用户明确、新结论、偏好变化
- 冲突检测：时间优先，关键结论用户确认
- 时效衰减：`timestamp` + `expire_at`，过期降权
- 权限隔离：用户级 / 团队级 / 公司级
- 评估：召回率、准确率、任务完成率、时效准确率

---

### 3.9 验证/风控

**技术选型**

| 类型 | 方案 |
|---|---|
| 规则校验 | 自研规则引擎 |
| 交叉验证 | 多源比对 + 计算复核 |
| 幻觉检测 | NLI + LLM-as-judge |
| 合规检查 | 规则引擎 + 敏感词 |

**分层校验**

```
第 1 层：规则校验（毫秒级，必过）
第 2 层：交叉验证（秒级）
第 3 层：语义校验（LLM-as-judge，关键结论）
```

**工程要点**

- 引用绑定到段落级，反向检索验证
- 幻觉检测：NLI、Self-consistency、反向提问
- 合规红线：不给投资建议、不承诺收益、标注免责
- 失败处理：分类回退，超限则拒答
- 评估：幻觉拦截率、误拦率、延迟、红队测试

---

### 3.10 Report 与审计

**技术选型**

| 环节 | 方案 |
|---|---|
| 报告模板 | Jinja2 + 结构化 Schema |
| 引用 | 脚注 / 行内 + 原文片段 |
| 图表 | Matplotlib / Plotly |
| 审计 | OpenTelemetry + Postgres + 对象存储 |
| 可观测 | LangFuse / LangSmith |
| 版本 | 数据/模型/Prompt/代码 |

**报告结构 Schema**

```
报告 = {
  摘要: {结论, 关键数据, 风险提示}
  公司概况, 财务分析, 同行对比, 估值, 风险, 数据来源, 免责声明
}
```

**工程要点**

- 结构化 Schema 约束骨架，LLM 填充内容
- 引用到段落级：`[1]: 贵州茅台2024年报，第23页，表格3`
- 审计日志：trace_id、每步输入输出、工具调用、数据来源、版本、校验结果
- 版本管理：数据版本、模型版本、Prompt 版本、代码版本
- 评估：事实准确性、逻辑一致性、完整性、合规性

---

## 4. 技术栈清单

```
编排：LangGraph + Pydantic AI
Embedding：BGE-M3 / Qwen3-Embedding-4B
向量库：pgvector（MVP）→ Qdrant（生产）
Rerank：bge-reranker-v2-m3
检索：BM25 + 向量 + RRF
解析：PyMuPDF + pdfplumber + Unstructured
沙箱：E2B / Docker + RestrictedPython
Memory：Redis + Postgres + 向量库
验证：自研规则引擎 + LLM-as-judge
可观测：LangFuse / LangSmith + OpenTelemetry
审计：Postgres + 对象存储
报告：Jinja2 + Matplotlib/Plotly
```

---

## 5. MVP 实现路线

| 阶段 | 目标 | 技术栈 |
|---|---|---|
| 第 1 周 | 单公司财报问答 | PyMuPDF + RecursiveCharacterTextSplitter + BGE-M3 + pgvector |
| 第 2 周 | 混合检索 + Rerank | BM25 + RRF + bge-reranker-v2-m3 |
| 第 3 周 | Agent 编排 + Tools | LangGraph + Python 沙箱 + SQL Tool |
| 第 4 周 | 验证 + 审计 | 规则校验 + 引用绑定 + LangFuse |
| 第 5 周 | Memory + 报告 | Redis + Postgres + Jinja2 |
| 第 6 周 | 评估 + 调优 | 评估集，Hit@5 / 忠实度 / 数值准确率 |

**先跑通，再优化。** 不要一上来就上 Milvus、多 Agent、GraphRAG。

---

## 6. 关键设计决策与面试要点

| 问题 | 回答要点 |
|---|---|
| 为什么需要 Memory？ | LLM 无状态，跨轮跨会话，金融需记住时效、偏好、历史结论 |
| 为什么需要 RAG？ | 模型知识截止、私有数据、可引用、可更新、可权限控制 |
| 为什么 Agent 不是简单 Prompt？ | 目标分解、工具调用、状态管理、Memory、反馈循环、护栏 |
| Tool Calling 怎么实现？ | Schema 定义 → 模型输出 tool_call → 校验 → 执行 → 返回 → 循环 |
| 为什么混合检索？ | 精确数字需 BM25，语义需向量，但需调权 |
| 为什么 Rerank？ | 双编码器 vs 交叉编码器，精度提升显著 |
| 怎么防幻觉？ | 规则 + 交叉 + 语义三层校验，引用绑定 |
| 怎么审计？ | trace_id 全链路，版本管理，可回放 |
| 怎么控制成本？ | 分层校验，缓存，摘要，token 预算 |

---

## 7. 目录结构建议

```
financial-research-agent/
├── connectors/          # 数据源接入
├── parsers/             # PDF/HTML 解析
├── chunking/            # 分块策略
├── embedding/           # Embedding 封装
├── vectorstore/         # 向量库客户端
├── retrieval/           # 混合检索 + Rerank
├── agent/               # LangGraph 编排
│   ├── nodes/
│   ├── state.py
│   └── graph.py
├── tools/               # 工具定义
│   ├── sql_tool.py
│   ├── python_sandbox.py
│   └── market_tool.py
├── memory/              # 分层记忆
├── validation/          # 验证/风控
├── report/              # 报告生成
├── audit/               # 审计日志
├── configs/             # 配置
├── tests/               # 测试
└── main.py
```

---

## 8. 配置与依赖

**核心依赖**

```
langgraph
langchain
pydantic
pydantic-ai
qdrant-client
pgvector
psycopg2-binary
redis
pymupdf
pdfplumber
unstructured
sentence-transformers
FlagEmbedding
rank-bm25
langfuse
opentelemetry-sdk
jinja2
matplotlib
plotly
pandas
numpy
```

**环境变量**

```
OPENAI_API_KEY=
QDRANT_URL=
POSTGRES_URL=
REDIS_URL=
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
```

---

## 9. 评估与测试

**评估集**

- 固定 50–100 个金融研究问题
- 标注正确答案、相关文档、引用位置

**指标**

| 模块 | 指标 |
|---|---|
| 检索 | Hit@5、MRR、Recall@5、NDCG@K |
| 生成 | 忠实度、引用正确率、数值准确率 |
| 报告 | 完整性、合规性、逻辑一致性 |
| 系统 | P95/P99 延迟、成本、错误率 |

**测试**

- 单元测试：各模块
- 集成测试：全链路
- 红队测试：注入错误数据
- 回归测试：定期跑评估集

---

## 10. 常见坑与应对

| 坑 | 应对 |
|---|---|
| 只做向量检索 | 混合检索 + 调权 |
| 让 LLM 算财务指标 | Python 沙箱 |
| 忽略数据时效 | 全链路 `report_period` + `announce_date` |
| 引用只写来源名 | 段落级引用 + 反向验证 |
| Agent 自由发挥 | 显式状态机 |
| 没有评估集 | 固定评估集 + 回归 |
| 一上来就多 Agent | 先单 Agent + 工具 |
| 忽略合规 | 规则硬拦截 + 免责声明 + 审计 |

---

> 本文档可直接交给 Codex / Claude Code，按模块逐步生成代码。  
> 建议从 MVP 第 1 周开始，先跑通单公司财报问答，再逐步扩展。