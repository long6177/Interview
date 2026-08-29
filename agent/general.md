# 通用无场景问题

## 1. Skill 和 MCP 有什么区别

**一句话定位**

> MCP 解决「连接」问题——让 Agent 能访问外部世界；Skill 解决「方法论」问题——教 Agent 怎么做某类任务。
> Anthropic 官方原话："MCP connects Claude to external services and data sources. Skills provide procedural knowledge—instructions for how to complete specific tasks or workflows."

**类比**：MCP 是 AI 的「手 / USB 接口」（能触达数据库、API、外部系统）；Skill 是 AI 的「技能书 / 训练手册」（知道一件事按什么流程、什么标准做）。

**核心对比表**

| 维度 | Skill（Agent Skill） | MCP（Model Context Protocol） |
|---|---|---|
| 本质 | 程序性知识 / 指令包 | 工具连接协议 |
| 解决的问题 | "怎么做好一件事"——上下文管理与能力封装 | "如何访问外部"——Agent 与外部工具/数据源的互操作 |
| 所在层级 | **提示/知识层**（Prompt/Knowledge Layer） | **集成层**（Integration Layer） |
| 形态 | 一个目录：SKILL.md + scripts/ + references/ + assets/ | Client-Host-Server 架构，基于 JSON-RPC 2.0 |
| 核心内容 | 指令、流程、示例、决策规则 | Tools（模型控制）、Resources（应用控制）、Prompts（用户控制） |
| 加载方式 | **渐进式加载**：元数据(~100 token) → 正文(<5k token) → 支持文件按需 | 连接后工具定义常驻上下文，随会话加载 |
| 是否执行代码 | 本身不执行，指导 Agent 执行；可附带脚本 | Server 是真实进程，执行真实操作（查库、调 API、发消息） |
| 部署方式 | 本地文件 / Git 仓库，一条命令安装 | 本地进程（stdio）或远程服务（HTTP/SSE） |
| 复杂度 | 低（以写文档为主，改 markdown 即改行为） | 中高（写代码、起服务、维护连接） |
| 上下文开销 | 低，可同时挂几十个 skill 不膨胀 | 高，每个 server 的工具定义都占 token |
| 典型场景 | 代码规范、品牌规范、SOP 工作流、专家角色 | 查数据库、接 Slack/Notion/GitHub、文件系统、实时数据 |
| 安全风险 | 恶意指令、prompt injection、隐藏脚本 | 权限过宽、凭证泄露、数据外泄 |

**架构层级关系**

```mermaid
flowchart TB
    subgraph L1["知识层 · 决定怎么做"]
        SK["Agent Skill<br/>指令 / 流程 / 示例 / 决策规则"]
    end

    subgraph L2["集成层 · 决定能访问什么"]
        TOOL["MCP Tools"]
        RES["MCP Resources"]
        PRM["MCP Prompts"]
    end

    SK -->|"内部调用"| TOOL
    TOOL --> SRV1["MCP Server · GitHub"]
    RES --> SRV2["MCP Server · PostgreSQL"]
    PRM --> SRV3["MCP Server · Notion"]
```

**两者是互补关系，不是竞争关系**

Skill 是上层能力封装，MCP 是底层工具接入——**Skill 可以在内部调用 MCP 工具**，并把"什么场景用、参数怎么组织、出错怎么处理"的使用知识一起打包。典型例子：

- **MCP 层**：`agent-mail`（发邮件）、`tencent-docs`（读写腾讯文档）等 connector 提供真实动作能力；
- **Skill 层**：`tencent-docs-routing` 告诉 Agent 遇到 Office 文件该走哪条处理链路，`wb-finance-skill` 告诉 Agent 金融类请求该用哪套方法——它们只是"说明书"，真正执行靠底层工具。

**怎么选？（决策框架）**

| 判断问题 | 答案 | 选择 |
|---|---|---|
| 本质是"知识/流程"还是"能力/访问"？ | 知识/流程 → **Skill**；能力/访问 → **MCP** | — |
| 需要连接第三方 API、数据库、实时数据？ | 是 → MCP | — |
| 只是要固定输出格式 / 遵循某套规范？ | 是 → Skill（纯提示词工程即可） | — |
| Agent 缺的是"不会做"还是"够不着"？ | 不会做 → Skill；够不着 → MCP | — |

**面试加分点（易被追问的坑）**

1. **渐进式信息公开（Progressive Disclosure）是 Skill 的精髓**：几十个 skill 同时存在，但同一时间只加载一两个，上下文效率远高于把工具定义全量塞进上下文——这是 Skill 相对 MCP 的关键优势。
2. **MCP 不是没有代价**：工具定义吃 token（社区观察到"server 暴露过多工具把上下文窗口占满导致幻觉"）、server 需要维护、第三方 server 有安全风险。
3. **判断口诀**："If you're explaining how to do something, that's a skill. If you need Claude to access something, that's MCP."（在教它怎么做 → Skill；要让它访问什么 → MCP）

## 2. 有没有实际写过 Skill？你会怎么定义一个 Skill，以及怎么判断它到底有没有效果？

**先解决"有没有写过"这个立场问题**

面试官问这个，核心不是听你说"写过/没写过"，而是考察三点：**① 对 Skill 触发/加载/结构机制的理解；② 有没有"把经验固化成可复用资产"的方法论；③ 有没有闭环评估意识**（定义了 ≠ 有效，怎么验证）。

两种立场都可以答得好：

- **真写过**：讲真实场景 → 为什么建 → 结构怎么设计 → 上线怎么验证 → 迭代了哪些点；
- **没写过（或接触不多）**：坦诚 + 展示方法论。例如："我在 WorkBuddy / Claude Code 中深度使用过几十个现成 Skill，拆解过它们的源码结构（frontmatter、渐进式加载、目录组织），也按官方 skill-creator 流程实操过从 0 到 1 的完整创建，下面是我的方法。"

**定义一个 Skill 的完整流程（6 步）**

```mermaid
flowchart LR
    A["Step 1<br/>识别场景与示例"] --> B["Step 2<br/>规划可复用内容"]
    B --> C["Step 3<br/>初始化骨架"]
    C --> D["Step 4<br/>撰写 SKILL.md"]
    D --> E["Step 5<br/>打包与自动校验"]
    E --> F["Step 6<br/>真实任务迭代"]
```

| 步骤 | 关键动作 | 要点 |
|---|---|---|
| 1. 识别场景 | 找出"重复出现、流程固定、容易犯错"的任务，收集 3–5 个真实示例 | 触发条件必须清晰；先问清楚"什么请求应该触发它、什么不该触发" |
| 2. 规划内容 | 分析示例决定放什么：指令 / scripts（确定性操作）/ references（大文档）/ assets（模板） | 原则：**SKILL.md 保持精简，细节下沉到 references**，避免信息重复 |
| 3. 初始化骨架 | 用官方脚手架生成 skill 目录 + SKILL.md 模板 | frontmatter 必须有 `name` + `description`（这是触发判断的唯一依据） |
| 4. 撰写 SKILL.md | 用**祈使句**写指令，回答三问：目的是什么 / 何时用 / 怎么用；写清决策优先级和边界 | 好的 description 等于这个 Skill 成功了一半 |
| 5. 打包校验 | 用打包脚本自动校验 frontmatter 格式、命名规范、结构、引用完整性 | 校验失败会报错，先修再发，避免坏包 |
| 6. 迭代 | 拿真实任务跑，记录失败点 → 更新 SKILL.md → 再测 | Skill 是活的，不是一次成型 |

**怎么写好 SKILL.md（关键细节）**

- **frontmatter 决定触发**：description 用第三人称、写清触发场景。差的描述："做报告"；好的描述："当用户需要把原始数据整理成符合 XX 公司风格的分析报告，包含执行摘要、关键发现和建议时使用"。
- **正文用祈使句**："To accomplish X, do Y"，不用第二人称——让另一个 Agent 实例能直接照做。
- **写死决策优先级**：把冲突时的处理顺序显式列出（如"保留真景优先 → 简化细节 → 再加装饰"），避免 Agent 自由发挥。
- **渐进式公开**：正文控制在 5k token 以内，大资料放 `references/` 按需 grep 加载；同代码反复重写的操作固化成 `scripts/`。

**怎么判断 Skill 到底有没有效果？（评估体系）**

| 层面 | 方法 | 具体指标 |
|---|---|---|
| 客观 | 建立**固定测试集**（5–10 个代表性任务） | 任务完成率、首轮成功率、按验收清单逐项打分 |
| 对比 | A/B 测试：同一任务有/无 Skill 各跑一轮 | 成功率提升幅度、出错次数、人工修正量 |
| 成本 | 上下文消耗对比 | 同任务下 token 消耗、调试耗时是否下降 |
| 用户 | 真实使用反馈 | 是否还在主动用；误触发率 / 漏触发率 |
| 维护 | 迭代稳定性 | 改完后是否引入回归；文档与实现是否一致 |

**核心判断标准（一句话）**：装上 Skill 后，同样的任务是否"**更少解释、更少返工、更稳定地一次做对**"。如果答案是否定的，要么是 description 写得差（触发不准），要么是内容没有提供模型本来不知道的信息——那就该改，而不是删。

**面试加分点**

1. 点出"Skill 是**上下文工程（Context Engineering）**的实践"——装几十个 skill 但上下文不膨胀，靠的就是渐进式公开。
2. 强调"效果要用**测试集度量**，而不是靠感觉"——体现工程化闭环思维。
3. 讲一个真实踩坑故事最加分，例如："最初 description 写得太含糊，Agent 频繁误触发；改成明确的正向+负向触发条件后，误触发率大幅下降，测试集首轮成功率从 60% 提到 90%。"

## 3. 目前关于 LLM 的 API 服务网络请求上游格式你了解多少, Response API, ChatCompletion, Anthropic Messages你都知道吗?

**一句话总览**

> 这三个不是平行概念：**Chat Completions 和 Responses API 是 OpenAI 的"新老两代"接口**（前者 2023 年推出、已成行业事实标准；后者 2025 年推出、是面向 Agent 的新一代统一接口），**Anthropic Messages API 是 Claude 的原生接口**（设计更严格、结构更清晰）。三者都是"发一段对话历史 + 参数 → 收到回复"的 JSON-over-HTTP 协议，但**请求字段、消息模型、工具调用表示、流式事件格式都不一样**。

**演进脉络（为什么会有三个格式）**

```mermaid
flowchart LR
    A["Completions API (2022)<br/>prompt 字符串进 / 文本出"] --> B["Chat Completions (2023.3)<br/>messages 数组 + function calling<br/>行业事实标准"]
    B -->|"OpenAI 新路线"| C["Responses API (2025.3)<br/>input/output + 内置工具 + 状态<br/>Agent 化统一接口"]
    D["Anthropic Messages API (2023)<br/>顶层 system + content 块数组<br/>与 OpenAI 并行发展"] -.->|"两套生态共存"| B
```

**演进原因**：Chat Completions 一个回合只能返回"一段文本 + 一组工具调用"，做 Agent 需要开发者自己写循环（调 LLM → 执行工具 → 再调 LLM）；Responses API 把"推理、函数调用、执行结果、最终回答"作为**类型化输出项（output items）**一次性返回，并内置 web_search / code_interpreter / file_search / computer use / 远程 MCP 等工具，官方从 2025 年起推荐所有新项目使用。**Chat Completions 未废弃、仍全量支持**，但已进入"维护模式"——新功能只在 Responses 上首发；Assistants API 则计划 2026 年年中官宣废弃。

**三大接口逐个说**

### ① Chat Completions（OpenAI 经典接口，事实标准）

**1) 最简请求与完整响应**

```json
// POST https://api.openai.com/v1/chat/completions
// Authorization: Bearer <API_KEY>
{
  "model": "gpt-4o",
  "messages": [
    { "role": "system", "content": "You are a helpful assistant." },
    { "role": "user",   "content": "北京今天几度？" }
  ],
  "temperature": 0.7,      // 0-2，越大越发散
  "max_tokens": 1024,      // 单次回复最大 token 数
  "top_p": 1.0,            // nucleus sampling
  "n": 1,                  // 返回候选数（>1 时 choices[] 会有多个独立回复）
  "stream": false,         // true 时返回 SSE 流
  "user": "user-123"       // 端到端追踪 ID，OpenAI 用于滥用检测
}
// 响应
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1717171717,
  "model": "gpt-4o-2024-08-06",
  "choices": [{
    "index": 0,
    "message": { "role": "assistant", "content": "北京今天 26℃，晴。" },
    "finish_reason": "stop"   // stop / length / tool_calls / content_filter
  }],
  "usage": {
    "prompt_tokens": 23,
    "completion_tokens": 12,
    "total_tokens": 35
  }
}
```

**2) 工具调用完整往返（2 次请求）**

```json
// ===== 第 1 次请求：告诉模型有哪些工具可用 =====
// POST /v1/chat/completions
{
  "model": "gpt-4o",
  "messages": [
    { "role": "user", "content": "北京今天几度？" }
  ],
  "tools": [{
    "type": "function",
    "function": {
      "name": "get_weather",
      "description": "查询指定城市的天气",
      "parameters": {
        "type": "object",
        "properties": {
          "city":  { "type": "string", "description": "城市名" },
          "unit":  { "type": "string", "enum": ["celsius","fahrenheit"] }
        },
        "required": ["city"]
      }
    }
  }],
  "tool_choice": "auto"   // auto / none / {"type":"function","function":{"name":"get_weather"}}
}

// ===== 第 1 次响应：模型决定调工具 =====
// 注意 arguments 是 JSON 字符串，不是对象！
{
  "choices": [{
    "finish_reason": "tool_calls",
    "message": {
      "role": "assistant",
      "content": null,
      "tool_calls": [{
        "id": "call_abc123",
        "type": "function",
        "function": {
          "name": "get_weather",
          "arguments": "{\"city\":\"北京\",\"unit\":\"celsius\"}"   // 字符串，要 JSON.parse
        }
      }]
    }
  }]
}

// ===== 第 2 次请求：把工具执行结果回传 =====
// （开发者自己执行 get_weather()，得到结果）
{
  "model": "gpt-4o",
  "messages": [
    { "role": "user", "content": "北京今天几度？" },
    { "role": "assistant", "content": null,
      "tool_calls": [{ "id": "call_abc123", "type": "function",
        "function": { "name": "get_weather", "arguments": "{\"city\":\"北京\"}" }}] },
    { "role": "tool", "tool_call_id": "call_abc123",   // 用 role:"tool" 回传
      "content": "{\"temp\":26,\"unit\":\"celsius\",\"condition\":\"晴\"}" }
  ]
}

// ===== 第 2 次响应：模型基于工具结果给最终回答 =====
{
  "choices": [{
    "finish_reason": "stop",
    "message": { "role": "assistant", "content": "北京今天 26℃，晴。" }
  }]
}
```

**3) SSE 流式响应（`stream: true`）**

```
data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","created":1717,
       "model":"gpt-4o","choices":[{"index":0,"delta":{"role":"assistant"},"finish_reason":null}]}

data: {"choices":[{"index":0,"delta":{"content":"北京"},"finish_reason":null}]}

data: {"choices":[{"index":0,"delta":{"content":"今天"},"finish_reason":null}]}

data: {"choices":[{"index":0,"delta":{"content":" 26℃"},"finish_reason":null}]}

data: {"choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

每个 chunk 是独立 JSON，文本增量在 `choices[0].delta.content`；首帧 `delta.role="assistant"`，末帧 `finish_reason` 有值；流结束标记 `data: [DONE]`。

**4) 多模态（vision）**

```json
{
  "model": "gpt-4o",
  "messages": [{
    "role": "user",
    "content": [
      { "type": "text",      "text": "这张图里有什么？" },
      { "type": "image_url", "image_url": { "url": "https://example.com/cat.jpg" } }
      // 也支持 base64： "url": "data:image/jpeg;base64,/9j/4AAQ..."
    ]
  }]
}
```

- **消息模型**：`messages[]`，角色 `system / user / assistant / tool`，system 提示词就是数组里的一条消息
- **工具调用**：assistant 消息带顶层 `tool_calls`（`function.arguments` 是 **JSON 字符串**）；工具结果用 `role: "tool"` + `tool_call_id` 回传
- **支持 n 个候选**：`n > 1` 时 `choices[]` 有多个独立回复
- **最大价值**：兼容生态最广——OpenRouter、vLLM、Ollama、Groq、LiteLLM 等全部实现了它，是开源模型训练/推理的事实格式

### ② Responses API（OpenAI 新一代统一接口）

```json
// POST /v1/responses
{
  "model": "gpt-5",
  "instructions": "You are a helpful assistant.",   // system 指令提到顶层
  "input": "北京今天几度？",                          // 字符串或 item 数组
  "tools": [{ "type": "web_search" }],               // 内置工具
  "store": true                                      // 服务端状态
}
// 响应
{ "output": [
  { "type": "message", "content": [{ "type": "output_text", "text": "..." }] },
  { "type": "function_call", ... }
] }
```

- **消息模型**：`input`（字符串或 item 数组）+ 顶层 `instructions`；输出是**类型化 `output[]`**：`message` / `reasoning`（思维链摘要）/ `function_call` / `function_call_output`
- **Agent 原生**：一个请求内可多次调工具；支持 `previous_response_id` / `store: true` / Conversations API 做**服务端会话状态**，不用客户端每次带全量历史
- **内置工具 + MCP**：web_search、code_interpreter、file_search、computer use、image_generation、**远程 MCP server**（`{"type": "mcp", "server_url": ...}`）
- **推理控制**：`reasoning.effort`（low/medium/high）；移除了 `n` 参数

### ③ Anthropic Messages API（Claude 原生）

```json
// POST /v1/messages
{
  "model": "claude-sonnet-4",
  "system": "You are a helpful assistant.",   // system 是顶层独立字段
  "messages": [{ "role": "user", "content": [{ "type": "text", "text": "北京今天几度？" }] }],
  "max_tokens": 1024                           // 必填
}
// 响应
{ "content": [
  { "type": "text", "text": "..." },
  { "type": "tool_use", "id": "toolu_01A", "name": "get_weather", "input": { "city": "北京" } }
], "stop_reason": "end_turn" }
```

- **设计更严格**：`messages` 只有 `user / assistant` 且**严格交替、第一条必须是 user**；`content` 统一是**内容块数组**（`text` / `image` / `tool_use` / `tool_result` / `thinking`）
- **工具调用即内容块**：模型要调工具就输出一个 `tool_use` 块，参数 `input` 是**已解析的 JSON 对象**（不是字符串）；工具结果用 user 消息里的 `tool_result` 块回传（引用 `tool_use_id`）
- **原生 thinking**：Extended Thinking 输出 `thinking` 块
- **认证**：`x-api-key` + `anthropic-version` 头（OpenAI 是 `Authorization: Bearer`）

**核心对比表（面试重点）**

| 维度 | Chat Completions | Responses API | Anthropic Messages |
|---|---|---|---|
| 端点 | `/v1/chat/completions` | `/v1/responses` | `/v1/messages` |
| 发布 | 2023.3 | 2025.3 | 2023 |
| 输入字段 | `messages[]` | `input` + 顶层 `instructions` | `system` + `messages[]` |
| system 位置 | messages 里的 role | 顶层 instructions | 顶层 system 字段 |
| 消息角色 | system/user/assistant/tool | item 体系（含 developer） | 仅 user/assistant（严格交替） |
| 输出结构 | `choices[].message` | 类型化 `output[]`（message/reasoning/function_call） | 顶层 `content[]` 块数组 |
| content 格式 | 字符串（多模态才用数组） | 字符串或 item 数组 | **永远是块数组**（有 type） |
| 工具调用 | `message.tool_calls`（arguments 是 JSON 字符串） | `function_call` item | `tool_use` 内容块（input 是对象） |
| 工具结果 | `role:"tool"` + `tool_call_id` | `function_call_output` item | user 消息内 `tool_result` 块 |
| 结束原因 | `finish_reason`（stop/length/tool_calls） | — | `stop_reason`（end_turn/max_tokens/tool_use） |
| 多候选 | 支持（n 参数） | 不支持（单 generation） | 不支持 |
| 会话状态 | 无状态，客户端管历史 | `store:true` / `previous_response_id` / Conversations | 无状态，客户端管历史 |
| 内置工具 | 无 | web_search/file_search/code_interpreter/computer use/MCP | 无（需外部接入） |
| 流式事件 | `choices[0].delta`（文本增量） | `response.output_text.delta` / `response.completed` | `content_block_start/delta/stop`、`message_delta` |
| 认证 | `Authorization: Bearer` | 同左 | `x-api-key` + `anthropic-version` |
| token 字段 | prompt/completion/total_tokens | 类似（含 cached 明细） | input/output/cache_read 等 |
| 定位 | 维护模式的事实标准 | 新项目官方推荐 | Claude 原生、生态独立 |

**面试加分点（谈深度的地方）**

1. **两个"数组"设计哲学完全不同**：Chat Completions 的 `choices[]` 是"多个平行世界的回答"（支持 n 参数、互相独立）；Anthropic 的 `content[]` 是"一个回答的多个组成零件"（文本 + 工具调用 + 思考块有序组合）。这是最容易看出你是否真懂 wire format 的点。
2. **工具调用是三大格式差异最尖锐的地方**：OpenAI 把参数序列化成**字符串**（`arguments: "{\"city\":\"北京\"}"`，要 `JSON.parse`）；Anthropic 直接给**对象**（`input: {"city":"北京"}`）；Responses 用 `function_call` item 统一表达。做统一网关/代理时（如 LiteLLM、OpenRouter、自研格式转换层）主要工作量就在这。
3. **流式（SSE）格式不兼容**：Chat Completions 增量在 `choices[0].delta.content`；Anthropic 是 `content_block_delta` 事件、且分 start/delta/stop 三段生命周期；Responses 是 `response.output_text.delta`。只改 endpoint 不改流式解析，一定会踩坑。
4. **当前工程建议**：新项目用 Responses API（官方推荐、功能全）；已有稳定项目没必要急着迁移 Chat Completions（未废弃、兼容生态最好）；要接 Claude 就用 Messages 原生格式，不要硬转成 OpenAI 格式（会丢 thinking、prefill 等能力）。多模型接入用兼容层（LiteLLM/OpenRouter）统一，但要注意它们默认按 Chat Completions 格式做归一。
5. **"上游格式"的工程含义**：LLM 应用中"上游"就是这层 wire format——它的设计决定了你的 Agent 循环怎么写（Responses 帮你省掉手写循环）、上下文怎么传（状态化 vs 无状态）、工具调用怎么解析（字符串 vs 对象）。面试时能讲清这点，就比只背 endpoint 强得多。

## 4. 关于 SSE 你了解多少? LLM 请求服务中的SSE呢?

**一句话定位**

> **SSE（Server-Sent Events，服务器发送事件）是 HTML5 标准里的一种服务端推送协议：客户端发一个普通 HTTP 请求，服务器用一个"永远不结束的响应"（`Content-Type: text/event-stream`）持续向客户端推数据，直到主动关闭。** 所有主流 LLM 厂商（OpenAI / Anthropic / Google Gemini / DeepSeek）的流式输出接口，底层用的都是 SSE。

类比：普通 HTTP 是一次"问答"（请求 → 响应 → 断开）；SSE 是一次"打电话"（拨通后服务器一直说，客户端一直听）。

**SSE 协议细节**

**工作流程**

1. 客户端发起 HTTP 请求（浏览器用 `EventSource`，请求头 `Accept: text/event-stream`）
2. 服务器返回 `200 OK` + 响应头：
   - `Content-Type: text/event-stream`（标志这是 SSE）
   - `Cache-Control: no-cache`（禁止缓存，保证实时）
   - `Connection: keep-alive`
3. **连接保持打开，服务器持续推送"事件"**
4. 事件以**两个换行符 `\n\n`** 分隔；每个事件由若干 `field: value` 行组成

**事件字段**

| 字段 | 作用 |
|---|---|
| `data:` | 数据载荷（可多行，客户端会用 `\n` 拼接；LLM 场景就是一段 JSON） |
| `event:` | 事件类型（客户端用 `addEventListener('xx')` 监听；缺省为 `message`） |
| `id:` | 事件序号（断线重连时通过 `Last-Event-ID` 请求头续传） |
| `retry:` | 重连等待毫秒数（覆盖浏览器默认值） |
| `:` | 注释行，常用于**心跳**（如 `: ping`），防止代理把空闲连接掐断 |

示例：

```
id: 42
event: update
data: {"orderId":"ord_123","status":"shipped"}
```

**三种实时方案对比（高频考点）**

| 维度 | SSE | WebSocket | 长轮询 |
|---|---|---|---|
| 通信方向 | **单向**（服务端→客户端） | 双向全双工 | 单向（模拟） |
| 底层协议 | 标准 HTTP（无升级） | 独立协议（101 升级握手） | 标准 HTTP |
| 自动重连 | **内置**（EventSource + Last-Event-ID） | 手动实现 | 手动实现 |
| 实现复杂度 | 低（普通 HTTP 路由即可） | 高（帧、心跳、状态管理） | 中 |
| 数据格式 | 仅文本（UTF-8） | 文本 + 二进制 | 文本/二进制 |
| 代理/CDN 兼容 | 极好（纯 HTTP） | 部分代理需特殊配置 | 极好 |
| HTTP/1.1 连接数限制 | 每源 6 条（HTTP/2 后消失） | 无 | 无 |
| 水平扩展 | 容易（无状态） | 难（需 Redis 等 pub/sub 广播） | 容易 |

```mermaid
sequenceDiagram
    participant C as "客户端"
    participant S as "服务器"
    C->>S: "POST /v1/chat/completions (stream:true)"
    S-->>C: "200 OK<br/>Content-Type: text/event-stream"
    loop "逐 token 生成"
        S-->>C: "data: {delta:{content:你}}"
        S-->>C: "data: {delta:{content:好}}"
        S-->>C: "data: {delta:{content:！}}"
    end
    S-->>C: "data: [DONE]"
    S-->>C: "连接关闭"
```

**LLM 为什么用 SSE 而不是 WebSocket（面试必答）**

1. **天然单向**：LLM 对话是"一问一答"——客户端发一次请求，服务器持续推 token，客户端在生成期间不需要回传数据。WebSocket 的双向能力是**过度设计（over-engineering）**，多出来的复杂度是负担不是优势。
2. **基础设施兼容**：SSE 就是普通 HTTP，能穿过所有代理、防火墙、CDN、负载均衡；WebSocket 需要 101 升级握手，部分严格代理会掐掉 ws 连接。
3. **首 token 延迟（TTFT）**：非流式要等模型把整段生成完（可能几十秒）才返回；流式第一个 token 几百毫秒就到了，**感知速度天壤之别**，还能提前打断（AbortController）。
4. **无状态易扩展**：断了重连就是一次新请求（幂等），不需要维护连接状态，水平扩展容易。
5. **生态标准**：OpenAI、Anthropic、Gemini、DeepSeek 全部用 SSE，SDK（OpenAI SDK / Anthropic SDK / Vercel AI SDK）底层都在消费 SSE。

**LLM 场景里 SSE 的实际形态**

### ① Chat Completions（OpenAI 兼容，事实标准）

**先建立心智模型：一次流式响应 = 一串"帧"**

服务器不是一次性返回完整答案，而是把答案**切碎、边生成边推**。线上传输的每一"帧"就是一行 `data:` 文本，帧与帧之间用空行（`\n\n`）隔开；`[DONE]` 是最后一帧，表示流结束。

```text
data: {...第 1 帧...}

data: {...第 2 帧...}

data: {...第 3 帧...}

data: [DONE]
```

**一次真实的完整流（带注释）**

```text
# 第 1 帧：角色帧——delta 里只有 role，声明"我开始回答了"
data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","created":1720000000,"model":"gpt-4o","choices":[{"index":0,"delta":{"role":"assistant"},"finish_reason":null}]}

# 第 2 帧：内容帧——delta.content 是本帧新生成的 1 个 token
data: {"choices":[{"index":0,"delta":{"content":"你"},"finish_reason":null}]}

# 第 3 帧：内容帧——继续增量
data: {"choices":[{"index":0,"delta":{"content":"好"},"finish_reason":null}]}

# 第 4 帧：结束帧——delta 为空，finish_reason 变成 "stop"，告诉你"我说完了"
data: {"choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

# 最后一帧：结束标记，它不是 JSON，就是个固定字符串
data: [DONE]
```

**每一帧里的 `{...}` 就是一个 chunk（数据块）**

把第 1 帧的 JSON 美化展开，逐字段看：

```json
{
  "id": "chatcmpl-abc",                     // 与普通响应同一个会话 id
  "object": "chat.completion.chunk",        // 注意：对象类型是 chunk（流式），不是 completion
  "created": 1720000000,
  "model": "gpt-4o",
  "choices": [{
    "index": 0,                             // 候选序号（n>1 时区分多个回答）
    "delta": { "role": "assistant" },       // ★ 流式专有字段：本帧带来的"增量"
    "finish_reason": null                   // 生成中为 null，结束时变为 stop/length/tool_calls
  }]
}
```

**核心区分：普通响应看 `message`，流式响应看 `delta`**

| 对比项 | 普通响应（stream:false） | 流式响应（stream:true） |
|---|---|---|
| choices 里装什么 | `message`：一条**完整**消息 | `delta`：本帧的**增量** |
| content | 一次给全的完整文本 | 只含本帧新生成的一小段 |
| role | 每次都有 | 只有第 1 帧有，其余帧为空 |
| 取数方式 | 直接用 | 每帧取 `delta.content` 拼接 |

**帧的生命周期（背下来，面试常问）**

| 阶段 | delta 内容 | finish_reason | 作用 |
|---|---|---|---|
| 第 1 帧 | `{"role":"assistant"}` | null | 角色帧：声明开始回答 |
| 中间帧 | `{"content":"X"}` | null | 内容帧：每生成一段就推一帧 |
| 末帧 | `{}` | `"stop"` | 结束帧：标记生成完成及原因 |
| 最后一帧 | `data: [DONE]`（非 JSON） | — | 流结束标记 |

**客户端拼装逻辑（增量 → 完整回答）**

```text
收到帧1: delta.content 为空        → 已拼文本 = ""
收到帧2: delta.content = "你"      → 已拼文本 = "你"
收到帧3: delta.content = "好"      → 已拼文本 = "你好"
收到帧4: finish_reason = "stop"    → 生成结束
收到帧5: [DONE]                    → 关闭连接，渲染已拼文本 "你好"
```

**工具调用时流长什么样（差异点）**

模型决定调工具时，delta 里不是 content 而是 `tool_calls`，而且 **arguments 也是分片字符串**：

```text
data: {"choices":[{"index":0,"delta":{"tool_calls":[{"index":0,"id":"call_abc","type":"function","function":{"name":"get_weather","arguments":""}}]},"finish_reason":null}]}

data: {"choices":[{"index":0,"delta":{"tool_calls":[{"index":0,"function":{"arguments":"{\"city\":"}}]},"finish_reason":null}]}

data: {"choices":[{"index":0,"delta":{"tool_calls":[{"index":0,"function":{"arguments":"\"北京\"}"}}]},"finish_reason":null}]}

data: {"choices":[{"index":0,"delta":{},"finish_reason":"tool_calls"}]}
```

客户端要做两件事：**① 把多帧的 `arguments` 碎片拼成完整字符串**（`{"city":"北京"}`）；**② 再 `JSON.parse` 成对象**才能用。`finish_reason="tool_calls"` 表示"模型不说话了，它想调工具"。

**想拿 usage（token 统计）？**

加参数 `stream_options: {"include_usage": true}`，usage 会出现在**最后一个 chunk**（`[DONE]` 前）里。

**面试加分点（谈深度）**

1. **"为什么是 SSE 不是 WebSocket"是 LLM 面试高频题**，标准答法：单向性 + HTTP 兼容 + 简单 + 生态标准，WebSocket 是过度设计。
2. **能讲出三种流的格式差异**（Chat Completions 裸 chunk / Responses 语义事件 / Anthropic 分块生命周期），证明你真正对接过 LLM 服务，而不是只背概念。
3. **客户端为什么用 fetch 而不是 EventSource**（POST 限制 + 自定义 header）——这题一答出来，面试官就知道你写过真实代码。
4. **性能指标意识**：流式优化的核心指标是 **TTFT（Time To First Token，首 token 延迟）** 和 tokens/s，而不是"总耗时"。
5. **结合上游格式**：SSE 是"传输层"，第 3 题的 wire format 是"业务层"——两者结合讲（如 Chat Completions 流式就是"delta 增量 + [DONE]"），体现体系化理解。