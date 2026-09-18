1. OpenAI 和 Claude code使用感受
	 针对不同button的使用，openAI的聊天感受过于闲散，准确度和稳定性有限。但是对于数据的组织非常有逻辑，project一栏，是建立专门的窗口进行讨论的。比如做代码项目，是可以把需求文档、demo和需求分析、技术路线一起丢进去的。
	 而Claude code整体对于项目和代码的理解比较好，逻辑条理和答案专业。不同的是claude code在书写代码时用的是终端（生态更加原生），而codex是一个工具接口。
	 ![](images/claudecode-使用截图.png)
2. 处于用户和后端开发层之间的产品（做应用的）
	 Openhand:**专为软件工程设计的开源AI Agent平台**
		 OpenHands 更像一个“AI **Orchestrator**（编排器）”或“元平台”。它提供一个标准化的环境和架构，用来指挥不同的 AI Agent（可以是内部的CodeActAgent，也可以是外部的Claude Code等）
我告诉你它做了什么？
		 昨天尝试的第一个项目是做数据提取、分析和可视化，在5分钟之内完成，但是在云端无法部署。所以我遇到了启动项目的问题，在这个过程中我没有意识到openhands cloud 的代码是运行在云端的，所以项目路径和启动在本地终端是无法工作的。
		
		 我给了它一个指令“新建一个python工程，用来获取美股的财报数据，并把一个公司的财务信息，按照季度和年的方式，用web的方式展示出来，并将其可视化为图表。”然后他分解任务为：创建项目结构和依赖文件/实现数据获取模块 (yfinance)/创建Web服务器 (FastAPI)/实现前端页面和数据可视化/测试并运行项目。
		 创建项目结构分为：1. 建立项目介绍文档和下载相应依赖 2.建立data_fetcher.py 3.建立前段页面。项目代码完成后，执行部署、运行和测试，在它答复中，可以看见包含打开网页、查看数据等自测过程。结束之后返回结果，结果如下：

/workspace/project/stock_financial_data/
├── api/
│   ├── __init__.py
│   ├── data_fetcher.py    # 数据获取模块 (yfinance)
│   └── main.py            # FastAPI 后端服务
├── templates/
│   └── index.html         # 前端页面 + Plotly 图表
├── static/
│   ├── css/               # CSS 样式
│   └── js/                # JavaScript 文件
├── main.py                # 启动入口
├── pyproject.toml         # 项目依赖配置
└── README.md              # 使用说明

### 功能特性

1. **数据获取**: 使用 yfinance 从 Yahoo Finance 获取美股财报数据
2. **财务报表**: 支持利润表、资产负债表、现金流量表
3. **数据视图**: 支持按年度和季度查看数据
4. **图表可视化**: 使用 Plotly.js 渲染交互式图表
    - 收入趋势图
    - 净利润趋势图
    - 每股收益趋势图
    - 资产负债概况图
    - 现金流量概况图
5. **详细表格**: 显示关键财务指标数据

### 运行方式

`cd /workspace/project/stock_financial_data # 安装依赖 pip install -e . # 启动服务 python main.py`

服务启动后访问: **[http://localhost:12000](http://localhost:12000/)**

### API 端点

- `GET /` - 主页
- `GET /api/stock/{ticker}` - 获取股票基本信息
- `GET /api/financial/{ticker}?period=yearly` - 获取年度财务数据
- `GET /api/financial/{ticker}?period=quarterly` - 获取季度财务数据
- `GET /health` - 健康检查

### 支持的股票示例

AAPL, MSFT, GOOGL, AMZN, TSLA, META, NVDA 等主流美股

- 基于这个答复，我找了一段时间在终端启动服务。发现网页上的终端是 read only，而且我也不知道项目在本机的哪个位置。经过几轮问答后，我才知道目前的项目是在网页背后的sandbox，并不在我本地上。如果我想本地运行，可以给它一个打包命令。这让我想起了之前在公司工作做项目的经验，如果在服务器上打包一个项目，开放接口，那么可以直接访问网页。于是我就尝试让它直接部署，结果是部署之后我可以直接网页访问。
- 问题是网络不稳定，不仅是问答过程，还包括部署之后的网页访问。
- 为了测试边界，我尝试增加功能：获得美股的实时数据。这样我可以和网站上的数据做对比，以验证可靠性。
- 目前不知道它和agent的区别，也不知道边界在哪里。


OpenHands → 你给目标，它自己跑完整个循环 → 中间不需要你，它自己决定下一步做什么 → 你是在循环结束后才看到结果

> 一个能替你执行软件开发全流程的 Agent。

你给目标，它自己写代码、选工具、运行、测试、修复、部署。你不用在中间参与。

一个能替你执行软件开发全流程的 Agent。在任务客观、结果可验证的场景下可靠性高。边界在于运行环境的权限、网络稳定性，以及任务本身是否有客观的验证标准。


{{ { "model": "glm-4-flash", "messages": [{ "role": "user", "content": "Below is a list of news items, each in the format 'title link'.\n\nYour task: Only keep items strictly related to:\n- Artificial intelligence, machine learning, LLMs\n- Programming languages, developer tools, open source\n- Hardware, chips, quantum computing\n- Tech company news (OpenAI, Google, Apple, Meta, Nvidia, etc.)\n\nRules:\n- TikTok/YouTube entertainment content does NOT count as tech\n- Celebrity, film, music, sports do NOT count\n- If nothing qualifies, reply only with: No relevant news today\n- Do not explain, do not add anything, just output the matching items\n\nNews list:\n" + $json.body }] } }}


={{ JSON.stringify({ "model": "glm-4-flash", "messages":  }) }}

