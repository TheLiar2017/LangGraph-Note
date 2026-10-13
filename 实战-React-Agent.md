---
aliases: [React Agent, ReAct Agent, ReAct, create_react_agent]
tags: [实战, Agent, ReAct, 基础]
---

# 🤖 实战：React Agent

> 前面学了状态、节点、边、工具、记忆……现在**全部拼起来**，做一个真正能用的 Agent！
> ReAct = **Re**asoning（推理）+ **Act**ing（行动）——AI 先想、再做、再想、再做，直到完成任务。
> 本文从零搭建一个带工具调用、记忆、流式输出的完整 Agent。

---

## 一、ReAct 是什么？

### 1.1 一句话理解

> **ReAct = AI 边想边做，循环推进，直到解决问题。**

```text
用户: "北京今天天气怎么样？需要带伞吗？"

🤖 ReAct Agent 的思考过程：

  💭 思考（Reasoning）：用户问天气，我需要查一下北京的天气数据
  🔧行动（Acting）：调用 get_weather("北京")
  📋 观察（Observation）：北京，晴天，25°C，无降水
  💭 思考（Reasoning）：晴天无降水，不需要带伞
  💬 回答（Answer）：北京今天晴天 25°C，不需要带伞 ☀️
```

### 1.2 ReAct 的循环

```text
        ┌──────────────┐
        │   💭 思考     │  ← AI 分析当前情况，决定下一步
        └──────┬───────┘
               │
               ▼
        ┌──────────────┐
        │   🔧行动     │  ← 调用工具（查数据库、搜索、计算...）
        └──────┬───────┘
               │
               ▼
        ┌──────────────┐
        │   📋 观察     │  ← 看到工具返回的结果
        └──────┬───────┘
               │
               ▼
        需要继续？──── 是 ──→ 回到「思考」
               │
               否
               │
               ▼
        ┌──────────────┐
        │   💬 回答     │  ← 给出最终答案
        └──────────────┘
```

### 1.3 ReAct vs 普通对话

| | 普通对话 | ReAct Agent |
|---|:---:|:---:|
| **能力** | 只能说 | 能想、能做、能说 |
| **工具调用** | ❌ | ✅ |
| **多步推理** | ❌ | ✅（循环多次） |
| **实时数据** | ❌ | ✅（查数据库/API） |
| **类比** | 纯聊天的顾问 | 能查资料、能干活的助理 |

---

## 二、最快上手：create_react_agent

> LangGraph 提供了 `create_react_agent`，**一行代码**创建 ReAct Agent。

### 2.1 最小示例

```python
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI
from langchain.tools import tool

# 1. 定义工具
@tool
def add(a: int, b: int) -> int:
    """将两个数字相加。

    Args:
        a: 第一个数字
        b: 第二个数字
    """
    return a + b

@tool
def get_weather(city: str) -> str:
    """获取指定城市的天气。

    Args:
        city: 城市名，如"北京"、"上海"
    """
    weather = {
        "北京": "晴天，25°C",
        "上海": "多云，22°C",
        "广州": "小雨，28°C",
    }
    return weather.get(city, f"未找到 {city} 的天气")

# 2. 创建模型
model = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx",
)

# 3. ★ 一行创建 Agent
agent = create_react_agent(model, tools=[add, get_weather])

# 4. 运行
result = agent.invoke({
    "messages": [{"role": "user", "content": "北京天气怎么样？3+4等于多少？"}]
})

for msg in result["messages"]:
    print(f"{msg.type}: {msg.content}")
```

**模拟输出：**
```text
human: 北京天气怎么样？3+4等于多少？
ai: [Tool Calls: get_weather(city="北京"), add(a=3, b=4)]
tool: 晴天，25°C
tool: 7
ai: 北京今天晴天，气温 25°C ☀️。另外 3+4=7。
```

> **就这么简单！** `create_react_agent` 帮你自动搭建了完整的 ReAct 循环图。

### 2.2 create_react_agent 做了什么？

```text
你只写了：
  model + tools
       │
       ▼
create_react_agent 自动搭建了这个图：

  START → llm → [有 tool_calls？] ─是─→ tools ─┐
                  │                              │
                  否                             │
                  │                              │
                  ▼                              │
                 END  ←──────────────────────────┘
                       (回到 llm 继续推理)

  ✅ 自动创建了 llm 节点（模型推理）
  ✅ 自动创建了 tools 节点（工具执行）
  ✅ 自动添加了条件路由（要不要调用工具）
  ✅ 自动形成了循环（工具结果 → 模型继续推理）
```

---

## 三、完整实战：智能助手 Agent

> 做一个能查天气、查时间、算数学、搜索知识的智能助手。

### 3.1 完整代码

```python
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import InMemorySaver
from langchain_openai import ChatOpenAI
from langchain.tools import tool
from langchain.messages import SystemMessage
from datetime import datetime

# ═══ 1. 定义工具 ═══

@tool
def get_weather(city: str) -> str:
    """获取指定城市的天气信息。

    Args:
        city: 城市名，如"北京"、"上海"、"广州"
    """
    weather_data = {
        "北京": "晴天，25°C，湿度 40%",
        "上海": "多云，22°C，湿度 65%",
        "广州": "小雨，28°C，湿度 80%",
        "深圳": "阴天，26°C，湿度 70%",
    }
    return weather_data.get(city, f"暂无 {city} 的天气数据")

@tool
def get_current_time() -> str:
    """获取当前日期和时间。"""
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")

@tool
def calculator(expression: str) -> str:
    """计算数学表达式的结果。支持加减乘除。

    Args:
        expression: 数学表达式，如 "3 + 4"、"10 * 5"、"100 / 3"
    """
    try:
        # 只允许基本数学运算，避免安全风险
        allowed = set("0123456789+-*/(). ")
        if not all(c in allowed for c in expression):
            return "错误：只支持数字和 + - * / ( ) 运算符"
        result = eval(expression)
        return f"{expression} = {result}"
    except Exception as e:
        return f"计算错误: {str(e)}"

@tool
def search_knowledge(query: str) -> str:
    """搜索知识库，获取百科、常识等信息。

    Args:
        query: 搜索关键词，如"LangGraph 是什么"、"光速是多少"
    """
    # 模拟知识库搜索
    knowledge = {
        "langgraph": "LangGraph 是 LangChain 生态中的低层级编排框架，用于构建有状态、可持久化的 AI Agent 工作流。",
        "光速": "光在真空中的速度约为 299,792,458 米/秒（约 30 万公里/秒）。",
        "python": "Python 是一种高级编程语言，由 Guido van Rossum 于 1991 年发布。",
        "ai": "人工智能（AI）是研究如何让计算机模拟人类智能的学科。",
    }
    query_lower = query.lower()
    for key, value in knowledge.items():
        if key in query_lower:
            return value
    return f"未找到关于「{query}」的信息"

# ═══ 2. 创建模型 ═══
model = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx",
)

# ═══ 3. 系统提示 ═══
system_prompt = SystemMessage(content="""你是一个智能助手，可以帮助用户：
- 查询天气（get_weather）
- 查询时间（get_current_time）
- 数学计算（calculator）
- 搜索知识（search_knowledge）

回答要简洁、友好、准确。如果需要多次调用工具，逐步进行。
""")

# ═══ 4. 创建 Agent ═══
agent = create_react_agent(
    model,
    tools=[get_weather, get_current_time, calculator, search_knowledge],
    prompt=system_prompt,    # 系统提示
)

# ═══ 5. 运行 ═══
result = agent.invoke({
    "messages": [{"role": "user", "content": "今天几号？北京天气怎么样？123乘以456等于多少？"}]
})

for msg in result["messages"]:
    if msg.type == "ai" and msg.content:
        print("🤖:", msg.content)
```

**模拟输出：**
```text
🤖: 今天是 2026-10-13。北京天气晴天，25°C，湿度 40% ☀️。
    123 × 456 = 56,088。
```

**消息流转过程：**
```text
messages = [
  HumanMessage("今天几号？北京天气怎么样？123乘以456等于多少？"),
  AIMessage(tool_calls=[get_current_time(), get_weather("北京"), calculator("123*456")]),
  ToolMessage("2026-10-13 10:30:00"),
  ToolMessage("晴天，25°C，湿度 40%"),
  ToolMessage("123*456 = 56088"),
  AIMessage("今天是 2026-10-13。北京天气晴天...123 × 456 = 56,088。"),
]
```

---

## 四、加上记忆：多轮对话

> Agent 不仅要能干活，还要**记得之前聊了什么**。

### 4.1 加 checkpointer

```python
from langgraph.checkpoint.memory import InMemorySaver

# 创建 checkpointer
checkpointer = InMemorySaver()

# ★ 创建 Agent 时传入 checkpointer
agent = create_react_agent(
    model,
    tools=[get_weather, get_current_time, calculator, search_knowledge],
    prompt=system_prompt,
    checkpointer=checkpointer,     # ← 加这一行
)
```

### 4.2 多轮对话

```python
config = {"configurable": {"thread_id": "user-001"}}

# 第一轮
result = agent.invoke(
    {"messages": [{"role": "user", "content": "我叫小明"}]},
    config,
)
print("🤖:", result["messages"][-1].content)

# 第二轮（Agent 记得名字）
result = agent.invoke(
    {"messages": [{"role": "user", "content": "我叫什么？"}]},
    config,
)
print("🤖:", result["messages"][-1].content)   # → "你叫小明"

# 第三轮（记得全部历史）
result = agent.invoke(
    {"messages": [{"role": "user", "content": "刚才查了什么天气？"}]},
    config,
)
print("🤖:", result["messages"][-1].content)   # → "刚才查了北京的天气..."
```

**模拟输出：**
```text
🤖: 你好小明！很高兴认识你。
🤖: 你叫小明。
🤖: 刚才你查了北京的天气（晴天 25°C），还问了 123×456 的计算结果。
```

> **加了 checkpointer，Agent 就有了记忆**——同一个 `thread_id` 下的对话会自动衔接。

---

## 五、加上流式输出

> 聊天界面需要打字机效果——用 `stream_mode="messages"`。

### 5.1 流式运行

```python
for token, metadata in agent.stream(
    {"messages": [{"role": "user", "content": "用一句话介绍 LangGraph"}]},
    config,
    stream_mode="messages",      # ★ 字级流式
):
    if token.content:
        print(token.content, end="", flush=True)
```

**模拟输出（逐字显示）：**
```text
LangGraph 是 LangChain 生态中的低层级编排框架，用于构建有状态、可持久化的 AI Agent 工作流。
```

### 5.2 流式 + 工具调用

```python
for mode, data in agent.stream(
    {"messages": [{"role": "user", "content": "北京天气怎么样？"}]},
    config,
    stream_mode=["updates", "messages"],   # 同时看节点更新和 token
):
    if mode == "messages":
        token, meta = data
        if token.content:
            print(token.content, end="", flush=True)
    elif mode == "updates":
        print(f"\n📋 节点更新: {data}")
```

**模拟输出：**
```text
📋 节点更新: {'tools': {'messages': [ToolMessage(content='晴天，25°C')]}}
北京今天晴天，气温 25°C ☀️
```

---

## 六、加上人工介入

> 高风险操作（如发邮件、删数据）让 Agent 暂停，等人批准。

### 6.1 在工具中加入审批

```python
from langgraph.types import interrupt, Command

@tool
def send_email(to: str, subject: str, body: str) -> str:
    """发送邮件给指定收件人。

    Args:
        to: 收件人邮箱
        subject: 邮件主题
        body: 邮件正文
    """
    # ★ interrupt：暂停，等人批准
    approval = interrupt({
        "action": "发送邮件",
        "to": to,
        "subject": subject,
        "body": body,
        "prompt": "请审批此邮件（yes/no）",
    })

    if approval == "yes":
        # 实际发送逻辑
        return f"✅ 邮件已发送给 {to}"
    else:
        return "❌ 邮件被拒绝，未发送"

# 创建 Agent（必须带 checkpointer）
agent = create_react_agent(
    model,
    tools=[send_email, get_weather],
    checkpointer=InMemorySaver(),
)

# 运行
config = {"configurable": {"thread_id": "email-001"}}
result = agent.invoke(
    {"messages": [{"role": "user", "content": "发邮件给张三，告诉他明天开会"}]},
    config,
)

# 有 interrupt
if "__interrupt__" in result:
    print("⏸️ 等待审批:", result["__interrupt__"][0].value)

# 人批准后恢复
result = agent.invoke(Command(resume="yes"), config)
print(result["messages"][-1].content)
```

---

## 七、自定义 Agent 行为

### 7.1 用 prompt 控制 Agent 人格

```python
from langchain.messages import SystemMessage

# 专业客服
customer_service = SystemMessage(content="""你是一个专业的电商客服。
- 回答要礼貌、耐心
- 涉及退款、换货需要转人工
- 不要编造商品信息，没查到就说不知道
""")

# 编程助手
coding_assistant = SystemMessage(content="""你是一个编程助手。
- 代码要带注释
- 解释清楚每一步
- 如果不确定，建议用户查官方文档
""")

agent = create_react_agent(model, tools=tools, prompt=customer_service)
```

### 7.2 用 prompt 控制工具使用策略

```python
strict_prompt = SystemMessage(content="""你是一个数据查询助手。
规则：
1. 必须使用工具获取数据，不要凭记忆回答
2. 如果工具查不到数据，明确告诉用户"查不到"
3. 不要猜测或编造数据
4. 每次只调用一个工具，等结果后再决定下一步
""")
```

### 7.3 限制工具数量

```python
# 不同场景给不同的工具
research_tools = [search_knowledge, calculator]
weather_tools = [get_weather]

research_agent = create_react_agent(model, tools=research_tools, prompt=research_prompt)
weather_agent = create_react_agent(model, tools=weather_tools, prompt=weather_prompt)
```

---

## 八、create_react_agent 参数速查

```python
agent = create_react_agent(
    model,                  # 必填：LLM 模型
    tools=[...],            # 必填：工具列表
    prompt=system_message,  # 可选：系统提示（SystemMessage 或 str）
    checkpointer=...,       # 可选：检查点（加记忆）
    state_schema=...,       # 可选：自定义状态 schema
    debug=False,            # 可选：调试模式
)
```

| 参数 | 作用 | 不传会怎样 |
|------|------|-----------|
| `model` | 使用哪个 LLM | 报错 |
| `tools` | Agent 可用的工具 | 纯对话，不能调工具 |
| `prompt` | 系统提示 / 人格设定 | 用默认提示 |
| `checkpointer` | 对话记忆 | 无记忆，每次独立 |
| `state_schema` | 自定义状态字段 | 用默认 MessagesState |

---

## 九、常见问题

### ❌ Agent 不调用工具？

1. 检查工具的 `docstring` 描述是否清晰
2. 在 `prompt` 中明确告诉 Agent 何时用工具
3. 确认 `tools=[...]` 传入了工具

```python
prompt = SystemMessage(content="查询天气时必须使用 get_weather 工具，不要自己猜。")
```

### ❌ Agent 调用了错误的工具？

1. 工具描述写得更具体（包含使用场景）
2. 减少工具数量（5 个以内最佳）
3. 在 prompt 中明确工具使用规则

### ❌ 每次都从头开始，没有记忆？

```python
# ❌ 没传 checkpointer
agent = create_react_agent(model, tools=tools)

# ✅ 传了 checkpointer
agent = create_react_agent(model, tools=tools, checkpointer=InMemorySaver())
# 而且 invoke 时必须传 config={"configurable": {"thread_id": "..."}}
```

### ❌ 工具调用死循环？

```python
# 限制最大步数
result = agent.invoke(
    {"messages": [...]},
    config={"recursion_limit": 15}   # 最多 15 步
)
```

### ❌ 想看 Agent 的思考过程？

```python
for msg in result["messages"]:
    if msg.type == "ai":
        if msg.tool_calls:
            print(f"💭 思考: 调用工具 {msg.tool_calls}")
        if msg.content:
            print(f"💬 回答: {msg.content}")
    elif msg.type == "tool":
        print(f"📋 工具结果: {msg.content}")
```

---

## 十、手动构建 vs create_react_agent

| | `create_react_agent` | 手动构建 StateGraph |
|---|:---:|:---:|
| **上手难度** | ⭐ 一行搞定 | ⭐⭐⭐ 需要理解图结构 |
| **灵活性** | ⭐⭐ 固定 ReAct 结构 | ⭐⭐⭐ 完全自定义 |
| **适合场景** | 标准工具调用 Agent | 需要自定义节点/路由/状态 |
| **代码量** | 3 行 | 50+ 行 |
| **推荐** | ✅ **首选** | 需要自定义时用 |

> **建议**：先用 `create_react_agent` 跑通，需要自定义时再手动构建。大部分场景 `create_react_agent` 就够了。

---

## 十一、速查表

### 创建 Agent

| 操作 | 代码 |
|------|------|
| 最简创建 | `create_react_agent(model, tools)` |
| 带记忆 | `create_react_agent(model, tools, checkpointer=InMemorySaver())` |
| 带人格 | `create_react_agent(model, tools, prompt=SystemMessage(...))` |

### 运行 Agent

| 操作 | 代码 |
|------|------|
| 一次性 | `agent.invoke({"messages": [...]})` |
| 流式 | `agent.stream({"messages": [...]}, stream_mode="messages")` |
| 带记忆 | `agent.invoke({"messages": [...]}, config={"configurable": {"thread_id": "..."}})` |
| 限制步数 | `config={"recursion_limit": 15}` |

### 查看结果

| 需求 | 代码 |
|------|------|
| 最终回答 | `result["messages"][-1].content` |
| 所有消息 | `for msg in result["messages"]` |
| 工具调用 | `msg.tool_calls` |
| 工具结果 | `msg.content` (tool 类型消息) |
| 是否暂停 | `"__interrupt__" in result` |

---

## 十二、深入理解：ReAct Agent 的本质

### 12.1 Agent 的三个角色

```text
🧠 大脑（LLM）
   负责：推理、决策、生成回答
   不负责：执行具体操作

🔧 工具（Tools）
   负责：执行具体操作（查数据、调 API）
   不负责：决定何时调用

🎛️ 编排（LangGraph）
   负责：连接大脑和工具、管理循环、保存状态
   不负责：业务逻辑
```

### 12.2 类比：工作中的助理

```text
老板（用户）：帮我查一下明天北京的天气，如果下雨就提醒我带伞。

助理（ReAct Agent）：
  💭 老板要查天气 → 我需要查北京明天的天气
  🔧 调用天气查询工具
  📋 明天北京有小雨
  💭 下雨了 → 需要提醒老板带伞
  💬 老板，明天北京有小雨 🌧️，记得带伞哦！

助理会自己想、自己查、自己总结——不需要老板一步步指挥。
```

### 12.3 Agent 的能力边界

| 能做 ✅ | 不能做 ❌ |
|---------|----------|
| 调用你给它的工具 | 做工具列表之外的事 |
| 根据对话推理 | 真正理解世界 |
| 多步调用工具 | 保证 100% 正确 |
| 记住对话历史 | 超出上下文窗口的记忆 |
| 流式输出 | 自己学习新技能 |

> **一句话**：Agent 是一个**会用工具的 LLM**——它比纯对话强很多，但能力取决于你给了它什么工具。

---

## 下一步

- [[实战-自定义Agent]] — 手动构建 StateGraph Agent（完全自定义）
- [[实战-多Agent协作]] — 多个 Agent 分工合作
- [[实战-RAG-Agent]] — 检索增强生成 Agent
