1. 为什么需要这个？
User Interaction Model

Resources are application-driven, giving them flexibility in how they retrieve, process, and present available context. Common interaction patterns include:

- Tree or list views for browsing resources in familiar folder-like structures
- Search and filter interfaces for finding specific resources
- Automatic context inclusion or smart suggestions based on heuristics or AI selection
- Manual or bulk selection interfaces for including single or multiple resources

Applications are free to implement resource discovery through any interface pattern that suits their needs. The protocol doesn’t mandate specific UI patterns, allowing for resource pickers with preview capabilities, smart suggestions based on current conversation context, bulk selection for including multiple resources, or integration with existing file browsers and data explorers.

当用户在 UI 上选了某个 resource（比如 `calendar://my-calendar/June-2024`），你的 client 要能去对应的 server 请求这个 URI 的数据，然后把它拼进发给 LLM 的上下文里。选择界面怎么做，是你产品设计的问题，不是协议的问题。
2. sandbox 是把所需数据复制了一份吗？还是怎么样做到隔离？
	- 沙箱保护的是执行环境，这套机制保护的是数据本身。
	- 处理方式是：**shadow write（影子写入）** 不直接写真实数据，先写一份副本，验证副本没问题再替换；**只给只读权限 + 单独写入审批** 最保守但最安全的设计：沙箱内的代码只能读数据，所有写操作都变成"写操作请求"排队，人工审批后才真正执行。这就是 MCP 里 elicitation 发挥作用的场景。
3. prompt engineering是从MCP的server概念提出来的吗？
	- 基于这个，为什么Prompt要单独划分出来，它不和tool/ resource一样是接口吗？区分开的意义：**把调试好的 prompt engineering 成果打包进 server**
	![](images/CleanShot%202026-05-26%20at%2017.50.52@2x.png)
4. https://modelcontextprotocol.io/docs/learn/client-concepts
	 sampling 和之前的roots到底在讲什么
- **Roots：server 问 client "我能碰哪些文件"**
- **Sampling：server 问 client "帮我调一次 LLM"**
- 

client 告诉 server：你只能在这几个目录里找文件。
2. what do I get ?
	- Prompts are structured templates that define expected inputs and interaction patterns. They are user-controlled, requiring explicit invocation rather than automatic triggering. Prompts can be context-aware, referencing available resources and tools to create comprehensive workflows. Similar to resources, prompts support parameter completion to help users discover valid argument values.
	- the _host_ is the application users interact with, while _clients_ are the protocol-level components that enable server connections.
	- what I get from the website:https://mp.weixin.qq.com/s/SgToU2BbiXNSrKkN8dPAfg
		 注册香港和新加坡公司