
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
		 