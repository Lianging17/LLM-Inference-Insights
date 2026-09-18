1. **从LLM到Agent Skill**
	- LLM 大语言模型，底层架构是transformer，来源于Google论文《attention is all you need》。
	- 工作机制：从底层架构出发，LLM工作的流程分为两部分：用户的信息经过编码进入大模型和大模型返回信息给用户，即编码和解码过程。编码过程包括切分用户输入和经过one hot映射成为数字矩阵，即成为大模型处理的输入信息，而解码是大模型在一系列的运算之后预测下一个token出现的概率，然后输出信息。
	- Token：大模型处理文本的最基本单元
	- Context: 这个概念是处理具体问题所接受到的信息总和。这个问题并不是在llm上提出的，而是在语言学，因为具体的语义在背景环境下有其相应的意义。而处理问题，重要的是在什么背景和场景下提出。
	- RAG：本地部署大模型，建立私有数据库。（一方面为了数据私有和安全，其二是突破了context的限制）
	- Prompt：User Prompt + System Prompt
	- 平台的工作原理：
	![](images/CleanShot%202026-05-24%20at%2018.39.48@2x.png)
			tool的基本功能是提供一系列工具，让大语言模型能够感知到外部环境，是因为模型始终只会输出语言（选择工具），所以基于这个最核心的点，agent会实现调用工具和统一接口的方式来打通与模型的文本交互。
	**大模型借助工具感知外部世界，而工具又可以使用MCP这种方式来统一接入。**
	- MCP（model context protocol):模型上下文统一接口
	- agent:自主规划和自主调用工具 ——> skill
2. **RAG工作机制详解**
- 总体介绍
	- 常见场景：对私有化数据和特定数据信息建立数据库，摆脱常规模型平台context大小有限的，多次连续推理成本高延时长的短板。
- 逐步拆解：分片Chunking、索引和 召回Retrieval、重排（cross-encoder）、生成
	![](images/CleanShot%202026-05-24%20at%2019.27.12@2x.png)
	- 在这部分我问了一个问题，召回是否是callbcak（类似脱口秀）？答案是否，召回在RAG里是retrieval。retrieval 的原意是==the act of finding and bringing something back==。所以这个环节具体在做的事情是
		![](images/CleanShot%202026-05-24%20at%2019.35.23@2x.png)
  3. MCP
   **the website of MCP**
	**让你亲手把一个现成的 MCP server 接进 Claude Desktop，跑通整个流程**。