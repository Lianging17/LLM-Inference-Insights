1.**混合检索的实现方式**

金融场景需要 dense + sparse 混合检索。Qdrant 内置稀疏向量和 BM25 支持，不需要额外维护 Elasticsearch[](https://www.dtstack.com/zh-cn/blogs/vector-database-selection-guide/#1)。Milvus 支持多向量混合检索。pgvector 则需要自行组合，通常配合 PostgreSQL 的全文检索（`tsvector`）做混合。
2.**量化压缩控制内存成本**

金融文档库动辄百万级 chunk，1024 维 FP32 向量的内存占用很大。Qdrant 内置标量量化、乘积量化、二值量化[](https://www.dtstack.com/zh-cn/blogs/vector-database-selection-guide/#1)。实测表明，经过量化的 HNSW 索引可使内存占用降低 70%，同时保持 95% 以上的召回率[](https://developer.baidu.com/article/detail.html?id=7196080#1)。工程上需要做量化前后的 recall 对比，确保精度损失可接受。
3.**金融场景的真实案例**
	**Qdrant + 养老金咨询**：Xaver 的 AI 金融咨询平台使用 Qdrant 构建双层知识引擎，第一层是精简知识库（预总结答案，近乎瞬时调用），第二层是完整知识库（监管、金融、政策文件的深度语料），在典型对话场景下缩短了 2-3 秒响应时间[](https://qdrant.org.cn/blog/case-study-xaver/#how-xaver-built-its-ai-knowledge-engine-with-qdrant)。
    **Qdrant + 银行多模态 RAG**：某银行业知识访问系统使用 BGE-M3 生成 embedding，按模态分别存入不同的 Qdrant collection，配合 RAG-Fusion 做查询改写和多路召回融合，在内部银行数据集上准确率和可解释性均优于纯文本 RAG baseline[](https://neurips.cc/virtual/2025/loc/san-diego/133791)。
    **pgvector + 金融风控**：pgvector 继承 PostgreSQL 的 ACID 事务，适合需要强一致性的金融风控场景。某金融科技公司在 500 万维度的用户画像检索中，pgvector 查询延迟控制在 50ms 以内。
4.金融场景常用 HNSW（图索引，查询快）和 IVF（倒排索引，构建快、内存低）。选型要点：
- 延迟敏感（在线问答）→ HNSW
- 写入频繁（持续增量）→ IVF 或 HNSW + 增量更新
- 内存受限 → 量化 + DiskANN
评测时不能只看 QPS，要同时关注 Recall@K、P95/P99 Latency 和 Filter Performance[](https://cloud.tencent.cn/developer/article/2726511?policyId=1003#3)。高 QPS 但 recall 掉到 0.8 的系统，在金融场景是不可用的。
5.**为什么不能只用 FAISS？**
FAISS 是向量检索基础库，不是数据库。它不提供持久化、分布式、过滤检索、元数据管理、并发控制。金融场景需要元数据过滤（公司、报告期）、权限隔离、增量更新——这些都是数据库层的功能。面试时可以说：“FAISS 是引擎，向量库是整车。”
	“FAISS 是引擎，向量库是整车”的意思是 FAISS 只负责最核心的向量相似度数学计算（提供动力），而完整的向量数据库（如 Milvus、Qdrant、Pinecone）则为这个引擎配好了车身、刹车、仪表盘和驾驶室（即持久化存储、权限控制、元数据过滤、分布式扩展等整套生态）。
6.**过滤检索为什么重要？**

金融研究 Agent 的查询几乎总是带条件的：“某公司 + 某报告期 + 某章节”。如果向量库只能做纯 ANN 检索，过滤条件只能在检索后应用，会导致“先召回 100 条，过滤后只剩 3 条”的问题——精度严重损失。Qdrant 的 payload 过滤和 Milvus 的标量过滤能在 ANN 检索阶段就应用条件，召回精度显著更高。

**7. 向量库怎么和业务数据库配合？**

金融场景有两类数据：结构化财务数据（营收、利润、估值）和非结构化文本（公告、研报）。推荐架构：
- 结构化数据 → PostgreSQL / MySQL，Agent 通过 SQL Tool 查询
- 非结构化 chunk → 向量库，Agent 通过检索 Tool 查询
- 向量库中每条的 payload 存结构化元数据（company_id、report_period、section），支持跨库关联
    
**8. 怎么评估向量库的表现？**
- **Recall@K**：Top-K 中是否包含正确文档，金融场景 K 通常取 5-10
- **P95/P99 Latency**：尾部延迟比平均延迟更重要，对话场景要求 P99 < 50ms
- **Filter Performance**：带过滤条件时的召回率和延迟退化程度
- **Update Latency**：新公告入库后多久可检索到

