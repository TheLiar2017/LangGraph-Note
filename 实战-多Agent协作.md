---
aliases: [多Agent协作, Multi-Agent, Supervisor, 多Agent, Agent协作]
tags: [实战, Agent, 进阶, 多Agent]
---

# 🤝 实战：多 Agent 协作

> 一个 Agent 什么都能干？也许吧——但**不如几个专家分工合作**。
> 多 Agent 协作 = 把大任务拆给不同的「专家 Agent」，由一个「调度员」统筹。
> 本文讲清 LangGraph 中多 Agent 的常见模式和实现方式。

---

## 一、为什么需要多 Agent？

### 1.1 一个 Agent 的局限

```text
❌ 一个 Agent 干所有事：

  用户: "帮我查北京天气，写一首关于天气的诗，再总结成一份报告"

  🤖 一个 Agent：
    - 既查天气，又写诗，又写报告
    - 提示词越来越长，模型越做越糊涂
    - 工具太多，选错工具
    - 没有专注，质量不高
```

### 1.2 多 Agent 分工

```text
✅ 多 Agent 分工合作：

  用户: "帮我查北京天气，写一首关于天气的诗，再总结成一份报告"

  🎯 调度员（Supervisor）：拆解任务
     ├── 🌤️ 天气 Agent → 查北京天气
     ├── ✍️ 诗人 Agent → 写诗
     └── 📊 报告 Agent → 总结报告

  各司其职，每个 Agent 专注自己的领域
```

### 1.3 多 Agent 的优势

| 优势 | 说明 |
|------|------|
| **专注** | 每个 Agent 只管一个领域，做得更精 |
| **清晰** | 提示词短而明确，不互相干扰 |
| **可维护** | 改一个 Agent 不影响其他 |
| **可扩展** | 加新功能 = 加新 Agent |
| **权限隔离** | 不同 Agent 可以给不同的工具权限 |

---

## 二、模式一：Supervisor（调度员模式）

> 最常用的模式。一个 **Supervisor** 负责分析任务、分配给子 Agent、汇总结果。

### 2.1 架构

```text
                    ┌─────────────────┐
                    │    Supervisor    │  ← 调度员：分析任务、分配工作
                    │   （路由 Agent）  │
                    └───────┬─────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ 天气 Agent│ │ 诗人 Agent│ │ 报告 Agent│  ← 专家 Agent
        └─────┬────┘ └─────┬────┘ └─────┬────┘
              │             │             │
              └─────────────┼─────────────┘
                            ▼
                    ┌─────────────────┐
                    │    Supervisor    │  ← 汇总结果、决定下一步
                    └─────────────────┘
```

### 2.2 完整代码

```python
from typing import Literal
from typing_extensions import TypedDict
from typing import Annotated
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain_openai import ChatOpenAI
from langchain.messages import SystemMessage

# ═══ 1. 状态 ═══
class SupervisorState(TypedDict):
    messages: Annotated[list, add_messages]
    next_agent: str              # 下一个要调用的 Agent
    task_done: bool              # 任务是否完成

# ═══ 2. 模型 ═══
model = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx",
)

# ═══ 3. Supervisor（调度员）═══
def supervisor(state: SupervisorState):
    """分析当前进度，决定下一个 Agent"""
    response = model.invoke([
        SystemMessage(content="""你是任务调度员。根据对话历史，决定下一步由哪个 Agent 处理。

可选 Agent：
- weather：天气查询
- writer：写作、文案
- summary：总结、报告
- done：任务全部完成

只回复一个词：weather / writer / summary / done
"""),
        *state["messages"],
    ])

    agent_name = response.content.strip().lower()
    return {"next_agent": agent_name}

# ═══ 4. 专家 Agent ═══

def weather_agent(state: SupervisorState):
    """天气 Agent"""
    response = model.invoke([
        SystemMessage(content="你是天气专家。只回答天气相关问题，简洁明了。"),
        *state["messages"],
    ])
    return {"messages": [response]}

def writer_agent(state: SupervisorState):
    """写作 Agent"""
    response = model.invoke([
        SystemMessage(content="你是写作专家。擅长写诗、写文案。内容优美生动。"),
        *state["messages"],
    ])
    return {"messages": [response]}

def summary_agent(state: SupervisorState):
    """总结 Agent"""
    response = model.invoke([
        SystemMessage(content="你是总结专家。擅长提炼要点，生成报告。条理清晰。"),
        *state["messages"],
    ])
    return {"messages": [response]}

# ═══ 5. 路由 ═══
def route_to_agent(state: SupervisorState) -> str:
    if state["next_agent"] == "done":
        return END
    return state["next_agent"]

# ═══ 6. 构建图 ═══
graph = StateGraph(SupervisorState)

graph.add_node("supervisor", supervisor)
graph.add_node("weather", weather_agent)
graph.add_node("writer", writer_agent)
graph.add_node("summary", summary_agent)

graph.add_edge(START, "supervisor")
graph.add_conditional_edges("supervisor", route_to_agent, {
    "weather": "weather",
    "writer": "writer",
    "summary": "summary",
    END: END,
})
graph.add_edge("weather", "supervisor")    # 完成后回到 supervisor
graph.add_edge("writer", "supervisor")
graph.add_edge("summary", "supervisor")

app = graph.compile()

# ═══ 7. 运行 ═══
result = app.invoke({
    "messages": [{"role": "user", "content": "查一下北京天气，然后写一首关于春天的诗"}]
})

for msg in result["messages"]:
    if msg.type == "ai" and msg.content:
        print(f"🤖: {msg.content}")
```

**模拟输出：**
```text
🤖: 北京今天晴天，25°C。
🤖: 春风拂面花满枝，细雨润泥燕归时。暖阳高照人欢笑，正是人间好时节。
```

**执行流程：**
```text
START → supervisor → "weather" → weather_agent → supervisor
                  → "writer"  → writer_agent  → supervisor
                  → "done"    → END
```

---

## 三、模式二：Pipeline（流水线模式）

> 任务固定顺序：A → B → C，每个 Agent 处理完传给下一个。

### 3.1 架构

```text
START → Agent A → Agent B → Agent C → END
        (收集)     (分析)     (输出)

数据依次经过每个 Agent，像流水线一样
```

### 3.2 完整代码

```python
class PipelineState(TypedDict):
    messages: Annotated[list, add_messages]
    raw_data: str
    analysis: str
    report: str

def collect_agent(state: PipelineState):
    """收集数据"""
    response = model.invoke(f"请收集关于「{state['messages'][-1].content}」的基本信息")
    return {"raw_data": response.content}

def analyze_agent(state: PipelineState):
    """分析数据"""
    response = model.invoke(f"请分析以下数据：\n{state['raw_data']}")
    return {"analysis": response.content}

def report_agent(state: PipelineState):
    """生成报告"""
    response = model.invoke(f"请根据分析生成报告：\n{state['analysis']}")
    return {"report": response.content}

# 流水线：固定顺序
graph = StateGraph(PipelineState)
graph.add_node("collect", collect_agent)
graph.add_node("analyze", analyze_agent)
graph.add_node("report", report_agent)

graph.add_edge(START, "collect")
graph.add_edge("collect", "analyze")       # 固定顺序
graph.add_edge("analyze", "report")
graph.add_edge("report", END)
app = graph.compile()
```

---

## 四、模式三：投票/竞争模式

> 多个 Agent 各自独立工作，最后选最好的结果。

### 4.1 架构

```text
         ┌──→ Agent A ──┐
START ──┼──→ Agent B ──┼──→ 评审 ─→ 最佳结果
         └──→ Agent C ──┘
```

### 4.2 代码

```python
class VoteState(TypedDict):
    messages: Annotated[list, add_messages]
    results: Annotated[list, operator.add]   # 收集多个结果
    best_result: str

def agent_a(state: VoteState):
    """Agent A：简洁风格"""
    response = model.invoke(state["messages"] + [SystemMessage(content="用最简洁的话回答")])
    return {"results": [response.content]}

def agent_b(state: VoteState):
    """Agent B：详细风格"""
    response = model.invoke(state["messages"] + [SystemMessage(content="详细地回答")])
    return {"results": [response.content]}

def agent_c(state: VoteState):
    """Agent C：创意风格"""
    response = model.invoke(state["messages"] + [SystemMessage(content="用创意方式回答")])
    return {"results": [response.content]}

def judge(state: VoteState):
    """评审：选最好的结果"""
    candidates = "\n\n".join([f"方案{i+1}: {r}" for i, r in enumerate(state["results"])])
    response = model.invoke(f"以下是三个方案，请选最好的并说明原因：\n\n{candidates}")
    return {"best_result": response.content}

# 并行执行三个 Agent
graph = StateGraph(VoteState)
graph.add_node("agent_a", agent_a)
graph.add_node("agent_b", agent_b)
graph.add_node("agent_c", agent_c)
graph.add_node("judge", judge)

graph.add_edge(START, "agent_a")     # 三个并行
graph.add_edge(START, "agent_b")
graph.add_edge(START, "agent_c")
graph.add_edge("agent_a", "judge")   # 汇总到评审
graph.add_edge("agent_b", "judge")
graph.add_edge("agent_c", "judge")
graph.add_edge("judge", END)
app = graph.compile()
```

> **注意**：`results` 字段用了 `operator.add` reducer，三个 Agent 的结果才能都收集到。

---

## 五、模式对比

| 模式 | 结构 | 适合场景 | 复杂度 |
|------|------|---------|:------:|
| **Supervisor** | 调度员 + 专家 | 动态任务分配 | ⭐⭐ |
| **Pipeline** | 固定顺序流水线 | 固定流程 | ⭐ |
| **投票/竞争** | 并行 + 评审 | 需要多种方案 | ⭐⭐ |
| **分层（Hierarchical）** | 调度员管理调度员 | 超大任务 | ⭐⭐⭐ |

```text
Supervisor（灵活分配）     Pipeline（固定流程）      投票（并行选优）
   ┌─→ A                    A → B → C                 ┌─→ A ─┐
 S─┼─→ B                START →  →  → END            ├─→ B ─┼→ 评审
   └─→ C                                               └─→ C ─┘
```

---

## 六、实战：完整的 Supervisor 系统

> 把 Supervisor 模式做完整——加记忆、加流式、加人工介入。

### 6.1 完整代码

```python
from typing import Literal
from typing_extensions import TypedDict
from typing import Annotated
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt, Command
from langchain_openai import ChatOpenAI
from langchain.messages import SystemMessage

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]
    next_agent: str
    task_done: bool
    summary: str

model = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx",
)

def supervisor(state: AgentState):
    response = model.invoke([
        SystemMessage(content="你是调度员。根据进度决定下一步：weather/writer/summary/done。只回复一个词。"),
        *state["messages"],
    ])
    return {"next_agent": response.content.strip().lower()}

def weather_agent(state: AgentState):
    response = model.invoke([
        SystemMessage(content="你是天气专家。"),
        *state["messages"],
    ])
    return {"messages": [response]}

def writer_agent(state: AgentState):
    response = model.invoke([
        SystemMessage(content="你是写作专家。"),
        *state["messages"],
    ])
    return {"messages": [response]}

def summary_agent(state: AgentState):
    response = model.invoke([
        SystemMessage(content="你是总结专家。"),
        *state["messages"],
    ])
    return {"messages": [response], "summary": response.content}

def human_review(state: AgentState):
    """人工审核最终结果"""
    approval = interrupt({
        "summary": state.get("summary", ""),
        "prompt": "请审核最终结果（yes=确认 / no=重新处理）",
    })
    return {"task_done": approval == "yes"}

def route_to_agent(state: AgentState) -> str:
    if state["task_done"]:
        return END
    return state["next_agent"]

graph = StateGraph(AgentState)
graph.add_node("supervisor", supervisor)
graph.add_node("weather", weather_agent)
graph.add_node("writer", writer_agent)
graph.add_node("summary", summary_agent)
graph.add_node("human_review", human_review)

graph.add_edge(START, "supervisor")
graph.add_conditional_edges("supervisor", route_to_agent, {
    "weather": "weather",
    "writer": "writer",
    "summary": "summary",
    END: END,
})
graph.add_edge("weather", "supervisor")
graph.add_edge("writer", "supervisor")
graph.add_edge("summary", "human_review")
graph.add_conditional_edges("human_review", route_to_agent, {
    "weather": "weather",
    "writer": "writer",
    "summary": "summary",
    END: END,
})

checkpointer = InMemorySaver()
app = graph.compile(checkpointer=checkpointer)

# 流式运行
config = {"configurable": {"thread_id": "multi-001"}}
for token, metadata in app.stream(
    {"messages": [{"role": "user", "content": "查北京天气并写一首诗"}]},
    config,
    stream_mode="messages",
):
    if token.content:
        print(token.content, end="", flush=True)
```

---

## 七、常见问题

### ❌ Supervisor 死循环，总不返回 done？

1. 在 Supervisor 的提示词里明确终止条件
2. 限制最大步数：

```python
result = app.invoke(
    {"messages": [...]},
    config={"recursion_limit": 25}   # 最多 25 步
)
```

### ❌ 子 Agent 之间看不到对方的结果？

因为所有 Agent 共享同一个 State，子 Agent 返回的 `messages` 其他 Agent 都能看到。确保结果写入 `messages`：

```python
def my_agent(state):
    response = model.invoke(...)
    return {"messages": [response]}   # ✅ 写入 messages，其他 Agent 可见
```

### ❌ 并行节点结果丢失？

用 `operator.add` reducer：

```python
results: Annotated[list, operator.add]   # ✅ 收集多个结果
```

---

## 八、速查表

### 选择模式

| 需求 | 模式 |
|------|------|
| 动态分配任务 | Supervisor |
| 固定顺序处理 | Pipeline |
| 多方案选优 | 投票/竞争 |
| 超复杂任务 | 分层 Supervisor |

### 核心代码

| 操作 | 代码 |
|------|------|
| 调度员 | 根据 state 决定 `next_agent`，返回 Agent 名 |
| 子 Agent | 处理后 `return {"messages": [...]}` |
| 路由 | `add_conditional_edges("supervisor", route, {...})` |
| 回到调度员 | `add_edge("child", "supervisor")` |
| 终止 | 路由返回 `END` |

---

## 下一步

- [[实战-RAG-Agent]] — 检索增强生成 Agent
- [[实战-自定义Agent]] — 自定义节点和路由
- [[功能-人工介入-Human-in-the-Loop]] — Agent 中加入人工审批