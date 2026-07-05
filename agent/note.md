### 一. 基础点

#### 1. 一个例子说明 ReACT 的的经典具体实现

**用户问**: 2026年苹果和谷歌的市值谁更高, 差多少?

要想要使得这个问题的解决靠Agent的ReAct推理模式, 需要在两个方面下功夫

1. 发给LLM的初始prompt
2. 构建Agent的代码要遵循ReAct编排

Prompt需要用来约束LLM的思考强度(不要一次性多步思考)和输出格式:

```markdown
你是一个 AI 助手，可以使用以下工具：
- search(query): 搜索互联网获取最新信息
- calculator(expr): 计算数学表达式

回答时请严格按照以下格式：
Thought: 你的思考过程（分析当前情况，决定下一步）
Action: 工具名称
Action Input: 工具的输入参数
Observation: （此行由系统填入工具返回的结果，你不用写）
... 以上可以重复多轮 ...
Final Answer: 当你确定可以回答时，在这里给出最终答案

问题：2024 年苹果公司的市值是多少？和谷歌相比谁更高？
```

代码要跑一个循环, 不断的 "调LLM, 检查输出, 执行工具, 填回结果":

```python
def react_agent(question: str, tools: dict, max_steps: int = 10):
    # 把 ReAct 格式约束和问题拼在一起，作为初始 prompt
    prompt = build_react_prompt(question, tools)
    # 用来存每一轮的对话历史，每次调 LLM 都把完整历史带上
    history = []

    for _ in range(max_steps):
        # 调 LLM，让它输出下一步的 Thought + Action
        # 注意：每次调用都把完整历史拼进去，LLM 才知道之前做了什么
        response = llm.generate(prompt + "\n".join(history))

        if "Final Answer:" in response:
            # LLM 输出了 Final Answer，说明它判断任务完成了
            return response.split("Final Answer:")[-1].strip()

        # 从 LLM 输出里解析出 Action 名称和 Action Input
        # 例如：Action: search，Action Input: 苹果公司市值 -> ("search", "苹果公司市值")
        action, action_input = parse_action(response)

        # 执行对应的工具，拿到真实结果
        if action in tools:
            observation = tools[action](action_input)
        else:
            # 如果 LLM 填了一个不存在的工具名，给它一个错误反馈
            observation = f"工具 {action} 不存在，请选择可用工具"

        # 把这一轮的 LLM 输出（含 Thought+Action）和 Observation 都追加进历史
        # 下次调 LLM 时这些内容会成为它的「记忆」
        history.append(response)
        history.append(f"Observation: {observation}")

    return "超过最大步数，任务未完成"
```

现代 LLM（GPT-4、Claude 3 之后）基本都原生支持 Function Calling / Tool Use，模型可以直接输出结构化的 JSON 工具调用，不再需要靠解析 `Action: xxx `这种文本格式。

#### 2. RAG 的整体工作流程

一个完整的RAG系统分为两个大阶段

1. **索引阶段 (离线)**

   开卷考试前把每本书贴上标签, 目录索引, 标注重点内容

2. **查询阶段 (在线)**

   考试时, 哪道题目, 先理解题目, 然后在参考书中快速找到相关章节, 基于内容组织答案

![Image_2026-07-05_15-39-14_tpzafuwx.lul](./imgs/Image_2026-07-05_15-39-14_tpzafuwx.lul.png)

### 二. 题目

