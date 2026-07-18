## Level1

### Java AI Framework



Spring AI：将AI能力无缝集成到Spring生态中，提供了一套构建AI应用所需的底层原子抽象：

- 模型通信ChatClient：
- 提示词prompt
- 检索增强生成RAG
- 工具调用Function Calling
- 记忆 ChatMemory



Spring AI Alibaba：集成Spring AI生态 专为multi-agent和workflow编排设计的项目，架构包含三层：



Langchain4j：是LangChain的java版本



### **采样参数**

logits：模型每一步会给词表中每个候选token打一个分数，分数越高说明模型越觉得这个词该出现

softmax：原始分数经过softmax数学变换变成每个候选被选中的概率，最后模型根据概率分布抽签采样





**模型参数**

max_tokens：最长输出token 限制模型最多能回答多少字，只管输出，不管上下文窗口

max_context_tokens: 上下文窗口，输入+历史+提问总和上限



重复抑制：

frequency_penalty 频率惩罚 降低重复词语、复读

presence_penalty 存在惩罚 鼓励模型输出新内容



stream：流式开关 

enable_reasoning:是否开启思维链

reason_effort:推理强度

seed：随机种子 固定种子=相同问题答案几乎一致，复现测试



解码参数（Temperature、Top-p、Top-k 等）就是在这个“打分 → 概率 → 抽签”的过程中施加控制：

- Temperature：调整概率分布的“形状”，让高分选项更突出，或者让各选项更均匀。
- Top-p / Top-k：直接砍掉不靠谱的候选项，缩小“抽签池”。从前k个高频词选 
- Penalty 系列：对已经出现过的词降分，防止“复读机”



停止词：stop 遇到指定文字立刻停



**召回**

把散落在各处的东西**召回来、捞回来、找回来**

从海量数据里，快速粗筛出大概率有用的一批候选内容

先捞一批出来，再精细筛选



行业固定流程：（所有推荐、搜索、RAG）

1. 召回层：优先保覆盖率，多捞一点
2. 精排层：优先保精确度，从捞出来的选最好的
3. 业务过滤 去重、过滤、截断











