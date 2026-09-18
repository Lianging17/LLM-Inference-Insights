1.**混合检索的实现方式**

金融场景需要 dense + sparse 混合检索。Qdrant 内置稀疏向量和 BM25 支持，不需要额外维护 Elasticsearch[](https://www.dtstack.com/zh-cn/blogs/vector-database-selection-guide/#1)。Milvus 支持多向量混合检索。pgvector 则需要自行组合，通常配合 PostgreSQL 的全文检索（`tsvector`）做混合。
2.**量化压缩控制内存成本**

金融文档库动辄百万级 chunk，1024 维 FP32 向量的内存占用很大。Qdrant 内置标量量化、乘积量化、二值量化[](https://www.dtstack.com/zh-cn/blogs/vector-database-selection-guide/#1)。实测表明，经过量化的 HNSW 索引可使内存占用降低 70%，同时保持 95% 以上的召回率[](https://developer.baidu.com/article/detail.html?id=7196080#1)。工程上需要做量化前后的 recall 对比，确保精度损失可接受。
3.