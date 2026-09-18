
## Pipeline
	数据源 → 解析/清洗 → Chunk → Embedding → 向量库
	→ 混合检索 + Rerank → Agent 编排 → Tools
	 → Memory → 验证/风控 → Report → 审计日志


**模块**
	**数据源与解析**
		**多源接入**：Wind、Choice
		**增量更新**：用时间戳 + 唯一 ID 做去重和增量拉取
		**数据版本**：同一份财报可能有修正版，记录版本号和公告日
		 **失败降级**：主数据源挂了，自动切备用源
	**Chunk**
		切分策略
			|固定长度切分|快速 baseline|LangChain `RecursiveCharacterTextSplitter`|
			|递归切分|通用文本|按段落 → 句子 → 字符逐级切|
			|语义切分|语义连贯性要求高|LlamaIndex `SemanticSplitter`、Chonkie|
			|结构化切分|财报、公告、研报|按标题/章节/段落切，保留层级|
		  检索
			  **父子块**：子块用于检索，父块用于生成，兼顾精度和上下文
			  **上下文增强**：给每块补一句“这是某公司 2024 年报第 3 节”，提升检索命中
	 Embedding
		 中文财报/公告 → Qwen3-Embedding-4B（精度 + 成本平衡）
		 混合检索场景 → BGE-M3（dense + sparse 一套模型搞定）
		 快速验证 → OpenAI text-embedding-3-small
		 金融领域极致 → Fin-E5（需自部署，FinMTEB 排名第一）
	向量库
		MVP 阶段 → pgvector（已有 Postgres，零新组件）
		生产阶段 → Qdrant（延迟敏感，过滤检索强）
		超大规模 → Milvus（十亿级向量，高 QPS SLA）
	 


面试问题：
	chunk
		**为什么不能简单按 token 切？**： 语义切断、不利于召回和可追溯
		 **Chunk 大小怎么定？**：
			一般小块 200–500 token 用于检索，父块 1000–2000 token 用于生成
		    金融财报可以按章节，一节可能几千字，再在节内递归切
			关键是要做评估
		**金融表格怎么处理**：
			表格不能当普通文本切
			先抽取成结构化数据，比如 DataFrame 或 JSON
			再转成 Markdown 或自然语言描述，单独索引
			 检索到表格时，直接返回结构化数据
		 **怎么评估 Chunk 质量？**
			检索召回率：相关块是否被召回
		    答案忠实度：生成答案是否基于块内容
		    人工抽样：块是否语义完整、是否包含关键信息
		    A/B 测试：不同分块策略对最终答案的影响
	Embedding
		**1.Embedding 模型怎么选？面试官真正想听什么？**
		面试官不期待你报一个“最好的模型名”，而是想看你的选型逻辑。参考回答结构：
		“选型看六个维度：语言（中文优先 Qwen3/BGE）、领域（金融需要 FinMTEB 验证）、维度（精度 vs 存储成本）、最大输入长度（决定 chunk 上限）、部署方式（API vs 自部署）、成本。然后必须在自己的业务数据上跑 Hit@K 评估，不能只看 MTEB 排行榜。”
		**2.** **BGE-M3 和 Qwen3-Embedding 怎么选？**
		**BGE-M3** 的核心优势是**一套模型同时输出 dense、sparse、colbert 三种表示**，混合检索不需要额外维护 BM25 索引，工程复杂度低。1024 维存储也友好。
	    **Qwen3-Embedding** 的优势是**中文精度更高**，CMTEB 得分领先，且 8B 版本 MTEB 多语言第一如果检索质量是第一优先级且能接受更高的存储和推理成本，选 Qwen3-Embedding。
	    面试时可以这样答：“快速原型用 BGE-M3，混合检索开箱即用；精度敏感的生产系统用 Qwen3-Embedding-4B，中文场景性价比最优；如果要做金融领域极致优化，考虑 Fin-E5。”




工程上可以展示的细节
	**元数据拼接进 Embedding 文本**
	 **向量维度与存储的权衡**
	 **怎么做 Embedding 的 A/B 测试？**
	 