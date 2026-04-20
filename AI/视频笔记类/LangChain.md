## 1、LangChain使用概述

LangChain用于开发由LLM驱动的应用程序，Lang是指Language即大语言模型，Chain即“链”意味着将大模型与外部数据&各种组件连接成链以此来构建AI应用程序，有点Spring之于Java



常见的大模型应用开发框架：

- python：LangChain（适合复杂任务分解和单agent应用）、LlamaIndex(专注于高效的索引和检索，适合RAG场景)
- Java：SpringAI、LangChain4j



为什么需要LangChain?

我们也可以用GPT或Qwen这种模型的API直接开发，但使用LangChain可以统一规范，移植性更好，提供了现成的链式组装，让复杂逻辑更结构化、易组合扩展，有点类似JDBC



LangChain里最重要的两个模块：

- LangGraph：协调各组件完成更复杂任务
- LangSmith：链路追踪



**基于RAG架构的开发**

背景：

- 大模型的知识冻结

训练大模型时有语料，如果是语料截止时间之后的数据，大模型是不知道的

幻觉：一本正经胡说八道，还是由于语料不精确

RAG可以很精确地解决这两个问题



**基于Agent架构的开发**

充分利用llm推理决策的能力，通过增加规划、记忆、工具的能力，构建一个能够独立思考、逐步完成给定目标的智能体

Agent=LLM+memory+tools+planning+action



**神经网络了解**

神经网络是一种模拟人脑神经元之间信息传递的数学模型，由多层神经元组成，每个神经元接收来自上一层神经元的输入信号，并通过激活函数进行加权求和并输出一个结果

在神经网络中，数据通过输入层传递到隐藏层，最终到输出层，在，每一层中神经网络通过学习算法不断调整连接权重，使得神经网络能够准确地对输入数据进行分类或预测

其训练过程是通过反向传播算法进行的，计算神经网络输出结果与实际结果之间的误差也叫做损失函数，来调整连接权值，整体来看其原理是基于神经元的连接权重调整和误差反向传播的机制









## 2、Model IO

**模型的不同功能分类**

- 非对话模型（LLMs、TextModel）
- 对话模型（Chat model）
- 嵌入模型（Embedding models）

非对话模型：输入是字符串或promptValue对象 输出字符串 适合单次文本生成任务（摘要生成、翻译），不支持多轮对话上下文

对话模型：输入是消息列表List<Base,Message>或promptValue，每条消息需指定角色（System Msg、User Msg、AIMsg），输出消息对象BaseMessage子类通常是AIMessage，适合对话系统例如客服机器人、智能助手

Embedding model：文本嵌入模型，将文本作为输入并返回浮点数列表



没有最好的大模型，只有最适合的大模型



**模型调用的主要方法与参数**

在模型调用时 参数尽量放配置文件 最佳实践还是.env 然后用dotenv来加载

调用API：OpenAI提供的API&&Langchain统一方式调用API（推荐）& 各家大模型提供的API

- OpenAI/ChatOpenAI 创建模型对象（非对话类/对话类）
- model.invoke() 执行调用
- .content 提取模型返回的文本内容



模型函数调用需初始化大模型，设置必要参数：

- base_url 大模型API服务的域名
- api_key 身份验证的秘钥 大模型服务商提供
- model 指定具体大模型名称（gpt-4o-mini）

非必要参数

- temperature 温度，控制生成文本的随机性，取值范围是0-1 值越低输出越保守确定，值越高模型输出更多样
- max_token 限制生成文本的最大长度 防止文本过长 建议客服短回复128-256 常规对话512-1024



token：大模型处理文本的最小单位，相当于自然语言的词或字，LLM通常以token的数量作为收费依据，1token相当于1-1.8个汉字 3-4英文字母

[CloseAI - 亚洲规模最大的企业级AI中转平台](https://www.closeai-asia.com/)



langchain：

- systemMessage：设定AI行为规则或背景信息
- HumanMessage：来自用户输入
- AIMessage：存储AI回复的内容



FunctionMessage/ToolMessage 函数调用/工具消息 用于函数调用结果的消息类型

此外AI是没有记忆的，记忆是工程化的处理



为了尽可能简化自定义链的创建，我们实现了一个Runnable协议，关于模型的调用方法：

- invoke 处理单条输入 等待LLM完全推理完成后再返回调用结果
- stream 流式响应，逐字输出LLM的响应结果
- batch 处理批量输入

这些也有对应的异步方法，应该与await语法一起使用实现并发

- astream
- ainvoke
- abatch

在langchain中语言模型的输出分为流式和非流式，

- 非流式输出 这是LangChain与LLM交互时的默认行为，用户发出请求后，系统等待模型生成完整响应，然后一次性将结果返回
- 流式输出，更具交互感的模型输出方式，用户能看到模型逐个token地实时返回内容













