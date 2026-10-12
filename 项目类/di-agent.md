### 架构

前端+后端+agent（di-ccp）
浏览器从不直连di-ccp-app，永远经ai-platform的/ccp反代
ai-platform 与 di-ccp-app之间是服务端到服务端的 x-api-key 认证 而且有两套独立的A2A入口：一套挂在ai-platform 一套是di-ccp-app自带的
前端当黑盒：
/api/* 纯CRUD 
/ccp/* 部署动作，把数据库配置同步成di-ccp真正在用的.agents.json

### 后端相关

可以看到上传skill的时候
http://di-ccp.fws.qa.nt.ctripcorp.com/v1/files
这个接口在上传 我的任务的也是一样
创建聊天
/v2/chat/stream
先建好SseEmitter 是SpringMVC里先把响应连接占住，内容稍后异步写，方法返回处理HTTP请求的Servlet线程就结束了，真正读di-ccp流、往emitter写数据的是STREAM_EXECUTOR里提交的任务
后台线程去读流streamLines 每读到一行原始文本就回调一次，这是同步阻塞的，每读到一行，throwingRelay做两件事：
是否是ccp的completed事件
转发 把di-ccp回复里的内网IP换成
请求的是：http://10.118.170.255/v2/chat/stream
ai-platform是一个纯转发的网关层（逐行读、转发）真正agent的编排全部发生在di-ccp-app里

### agent部分

di-ccp本质是基于claude code套壳子的方案
ccp回去找一个叫claude的可执行文件，
然后有个@anthropic-ai/claude-agent-sdk 这个npm包，装的不只是一层JS API，还按平台附带了一个编译好的claude原生可执行文件，
本质是把Claude Code引擎当成核心推理/工具执行内核，自己不做任何LLM推理或工具调用逻辑，
全部发生在那个被spawn起来的claude子进程里，ai-platform配置的MCP/SKILL/专家最终目的是拼出喂给这个子进程的配置，用户上传的会话历史/工作区文件也是摆在这个子进程能读到的cwd里
claude-agent-sdk本身就是claude code打包成的库-它自带完整的agent循环、built-in工具、hooks、subagents、session管理，ccp没有重新实现agent，而是在这个SDK外面包了一层多租户HTTP/SSE服务，
还有一种做法是自己去写agent编排，本质是ReAct While循环，不断调LLM的message API，模型返回tool_use就执行对应工具，把结果拼回messages数组再继续喂，而skill/mcp/知识库全部可以换存储介质，也是正是langchain/langgraph等框架，以及自研agent平台的做法
但主要的权衡是我们把Claude Code以及踩过坑、持续在维护的那一大堆东西重新做一遍

### 知识点补充

1、SSE
服务器向浏览器实时推送消息的技术，让网页内容能自动更新，很像WebSocket
区别：
SSE是单向通信，只有服务器能向客户端发送数据，WebSocket是双向通信
SSE基于标准的HTTP/HTTPS协议，本质是一个长连接的http请求，服务器保持连接打开并持续发送事件流，而WebSocket使用独立的ws：//协议 需要通过一个特定的HTTP Upgrade请求简历连接
SSE server-sent events是目前大语言模型API实现流式响应的标准技术选择
可以看到响应头Content-Type为text/event-stream;charset=utf-8