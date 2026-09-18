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
