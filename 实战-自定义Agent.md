---
aliases: [自定义Agent, Custom Agent, 手动构建Agent, StateGraph Agent]
tags: [实战, Agent, 进阶, 自定义]
---

# 🛠️ 实战：自定义 Agent

> `create_react_agent` 很方便，但结构是固定的。
> 想要**完全掌控** Agent 的每个节点、每条边、每个路由？那就手动搭建。
> 本文用 StateGraph 从零构建一个完整的 Agent，让你彻底理解 Agent 的「骨架」。

---

## 一、为什么要手动构建？

### 1.1 create_react_agent vs 手动构建

```text
create_react_agent 的固定结构：

  START → llm → [有 tool_calls？] ─是─→ tools ─┐
                  │                              │
                  否                             │
                  ▼                              │
                 END  ←──────────────────────────┘

  ✅ 简单好用
  ❌ 没法加自定义节点
  ❌ 没法自定义路由逻辑
  ❌ 没法扩展状态字段
```

```text
手动构建可以做到的：

  START → plan → [需要工具？] ─是─→ search ─┐
                    │                         │
                    否                        ▼
                    ▼                       verify ─→ [满意？] ─是─→ summarize → END
                   draft                         │                       │
                                                 否                      ▼
                                                 │                    output
                                                 └──→ revise ─┘

  ✅ 完全自定义节点和边
  ✅ 自定义路由逻辑
  ✅ 扩展状态字段
  ✅ 加审核、验证、多步推理
```

### 1.2 什么时候用哪个？

| 场景 | 选择 |
|------|------|
| 标准工具调用 Agent | `create_react_agent` ✅ |
| 需要自定义节点（如审核、规划） | 手动构建 ✅ |
| 需要自定义状态字段 | 手动构建 ✅ |
| 需要复杂路由逻辑 | 手动构建 ✅ |
| 多步推理、子任务分解 | 手动构建 ✅ |
| 快速原型验证 | `create_react_agent` ✅ |

> **建议**：先用 `create_react_agent` 跑通，发现它满足不了需求时，再手动构建。

---

## 二、手动构建的最小示例

> 最简单的 ReAct Agent：模型 → 工具 → 模型 → 结束。

### 2.1 完整代码

```python
from typing import Literal
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import ToolNode
from langchain_openai import ChatOpenAI
from langchain.tools import tool

# ═══ 1. 定义工具 ═══
@tool
def add(a: int, b: int) -> int:
    """将两个数字相加。"""
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """将两个数字相乘。"""
    return a * b

tools = [add, multiply]

# ═══ 2. 创建模型并绑定工具 ═══
model = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx",
).bind_tools(tools)

# ═══ 3. 定义节点 ═══

def llm_node(state: MessagesState):
    """模型推理节点"""
    response = model.invoke(state["messages"])
    return {"messages": [response]}

tool_node = ToolNode(tools)    # 预置的工具执行节点

# ═══ 4. 定义路由 ═══

def should_use_tool(state: MessagesState) -> Literal["tools", END]:
    """判断是否需要调用工具"""
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tools"        # 有 tool_calls → 去执行工具
    return END                # 没有 → 结束

# ═══ 5. 构建图 ═══
graph = StateGraph(MessagesState)

graph.add_node("llm", llm_node)
graph.add_node("tools", tool_node)

graph.add_edge(START, "llm")
graph.add_conditional_edges("llm", should_use_tool, ["tools", END])
graph.add_edge("tools", "llm")       # 工具执行后回到模型

# ═══ 6. 编译 ═══
app = graph.compile()

# ═══ 7. 运行 ═══
result = app.invoke({"messages": [{"role": "user", "content": "3 加 4 等于多少？再乘以 10"}]})

for msg in result["messages"]:
    print(f"{msg.type}: {msg.content}")
```

**模拟输出：**
```text
human: 3 加 4 等于多少？再乘以 10
ai: [Tool Calls: add(a=3, b=4)]
tool: 7
ai: [Tool Calls: multiply(a=7, b=10)]
tool: 70
ai: 3 + 4 = 7，7 × 10 = 70。
```

**图结构：**
```text
START → llm → [有 tool_calls？] ─是─→ tools ─┐
                  │                              │
                  否                             │
                  ▼                              │
                 END  ←──────────────────────────┘
```

> 和 `create_react_agent` 的结构**一模一样**——但我们能完全控制每个部分了。

---

## 三、扩展：自定义状态字段

> `MessagesState` 只有 `messages`。手动构建可以加任意字段。

### 3.1 带步骤计数的 Agent

```python
from typing_extensions import TypedDict
from typing import Annotated
from langgraph.graph.message import add_messages

# ★ 自定义状态：在 messages 基础上加字段
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]   # 对话历史
    step_count: int                            # 已执行步数
    tool_calls_count: int                      # 已调用工具次数
    max_steps: int                             # 最大步数限制

def llm_node(state: AgentState):
    """模型推理（自定义状态读写）"""
    response = model.invoke(state["messages"])
    return {
        "messages": [response],
        "step_count": state["step_count"] + 1,   # 步数 +1
    }

def tool_node_custom(state: AgentState):
    """工具执行（自定义状态读写）"""
    from langchain.messages import ToolMessage
    result = []
    for tool_call in state["messages"][-1].tool_calls:
        tool = tools_by_name[tool_call["name"]]
        output = tool.invoke(tool_call["args"])
        result.append(ToolMessage(content=str(output), tool_call_id=tool_call["id"]))
    return {
        "messages": result,
        "tool_calls_count": state["tool_calls_count"] + len(result),  # 工具调用次数 +N
    }

def should_continue(state: AgentState) -> Literal["tools", END]:
    """路由：限制最大步数"""
    if state["step_count"] >= state["max_steps"]:
        return END                            # 超过最大步数，强制结束
    if state["messages"][-1].tool_calls:
        return "tools"
    return END

# 构建图
graph = StateGraph(AgentState)
graph.add_node("llm", llm_node)
graph.add_node("tools", tool_node_custom)
graph.add_edge(START, "llm")
graph.add_conditional_edges("llm", should_continue, ["tools", END])
graph.add_edge("tools", "llm")
app = graph.compile()

# 运行
result = app.invoke({
    "messages": [{"role": "user", "content": "3+4=?，再乘10"}],
    "step_count": 0,
    "tool_calls_count": 0,
    "max_steps": 10,
})
print(f"总步数: {result['step_count']}, 工具调用: {result['tool_calls_count']}")
```

**模拟输出：**
```text
总步数: 2, 工具调用: 2
```

---

## 四、扩展：自定义路由逻辑

> 路由函数可以实现任意复杂的判断逻辑。

### 4.1 分类路由

```python
from typing import Literal

def classify_and_route(state: AgentState) -> Literal["tool_call", "direct_answer", "clarify"]:
    """先判断用户意图，再路由"""
    last_msg = state["messages"][-1]

    # 有 tool_calls → 执行工具
    if last_msg.tool_calls:
        return "tool_call"

    # 消息太短，可能是不清楚的问题
    if len(last_msg.content) < 5:
        return "clarify"

    return "direct_answer"

graph.add_conditional_edges("llm", classify_and_route, {
    "tool_call": "tools",
    "direct_answer": END,
    "clarify": "ask_clarification",    # 新增：追问节点
})
```

### 4.2 工具结果验证路由

```python
def verify_tool_result(state: AgentState) -> Literal["llm", "retry_tool", END]:
    """验证工具返回结果，决定下一步"""
    last_tool_msg = state["messages"][-1]

    # 工具返回错误
    if "error" in last_tool_msg.content.lower():
        return "retry_tool"            # 重试

    # 工具返回空结果
    if not last_tool_msg.content.strip():
        return "retry_tool"            # 重试

    return "llm"                       # 正常回到模型
```

---

## 五、扩展：带审核节点的 Agent

> 生成内容后，加一个审核步骤，不通过就重新生成。

### 5.1 完整代码

```python
from langchain.messages import SystemMessage

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]
    draft: str
    review_passed: bool
    review_feedback: str

def generate_draft(state: AgentState):
    """生成草稿"""
    response = model.invoke(state["messages"])
    return {
        "messages": [response],
        "draft": response.content,
    }

def review_draft(state: AgentState):
    """审核草稿"""
    review_prompt = f"""请审核以下内容是否符合要求：
    
    用户需求: {state['messages'][0].content}
    生成内容: {state['draft']}
    
    判断标准：
    1. 是否回答了用户的问题
    2. 内容是否准确
    3. 是否有明显错误
    
    回复格式：APPASS 或 FAIL:原因
    """
    reviewer = ChatOpenAI(model="deepseek-chat", base_url="https://api.deepseek.com/v1", api_key="sk-xxx")
    result = reviewer.invoke(review_prompt)

    if "FAIL" in result.content:
        return {
            "review_passed": False,
            "review_feedback": result.content,
            "messages": [{"role": "system", "content": f"审核未通过，请修改：{result.content}"}],
        }
    return {"review_passed": True}

def route_after_review(state: AgentState) -> Literal["output", "generate_draft"]:
    if state["review_passed"]:
        return "output"
    return "generate_draft"            # 不通过 → 重新生成

def output(state: AgentState):
    """输出最终结果"""
    return {"messages": [{"role": "assistant", "content": f"最终结果：{state['draft']}"}]}

# 构建图
graph = StateGraph(AgentState)
graph.add_node("generate_draft", generate_draft)
graph.add_node("review_draft", review_draft)
graph.add_node("output", output)

graph.add_edge(START, "generate_draft")
graph.add_edge("generate_draft", "review_draft")
graph.add_conditional_edges("review_draft", route_after_review, ["output", "generate_draft"])
graph.add_edge("output", END)
app = graph.compile()
```

**图结构：**
```text
START → generate_draft → review_draft → [通过？] ─是─→ output → END
                                   │                       ↑
                                   否                      │
                                   └──→ generate_draft ───┘
                                         (重新生成)
```

---

## 六、扩展：多 Agent 协作（简单版）

> 用「路由节点 + 多个功能节点」实现简单的多 Agent 协作。

### 6.1 代码

```python
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]
    next_agent: str

def router(state: AgentState):
    """路由节点：根据内容决定交给哪个 Agent"""
    response = model.invoke([
        SystemMessage(content="""根据用户消息，决定交给哪个 Agent 处理。
        只回复一个词：
        - weather（天气相关）
        - math（计算相关）
        - general（其他）"""),
        state["messages"][-1],
    ])
    return {"next_agent": response.content.strip().lower()}

def weather_agent(state: AgentState):
    """天气 Agent"""
    response = model.invoke(state["messages"] + [
        SystemMessage(content="你是天气助手，只回答天气相关问题。")
    ])
    return {"messages": [response]}

def math_agent(state: AgentState):
    """数学 Agent"""
    response = model.invoke(state["messages"] + [
        SystemMessage(content="你是数学助手，只回答计算相关问题。")
    ])
    return {"messages": [response]}

def general_agent(state: AgentState):
    """通用 Agent"""
    response = model.invoke(state["messages"])
    return {"messages": [response]}

def route_to_agent(state: AgentState) -> str:
    return state["next_agent"]

# 构建图
graph = StateGraph(AgentState)
graph.add_node("router", router)
graph.add_node("weather", weather_agent)
graph.add_node("math", math_agent)
graph.add_node("general", general_agent)

graph.add_edge(START, "router")
graph.add_conditional_edges("router", route_to_agent, {
    "weather": "weather",
    "math": "math",
    "general": "general",
})
graph.add_edge("weather", END)
graph.add_edge("math", END)
graph.add_edge("general", END)
app = graph.compile()
```

**图结构：**
```text
         ┌──→ weather ──┐
START → router ─→ math ────→ END
         └──→ general ──┘
```

---

## 七、加上记忆和流式

> 手动构建的 Agent 同样支持 checkpointer 和流式输出。

### 7.1 加记忆

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
app = graph.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "user-001"}}
result = app.invoke({"messages": [...]}, config)
```

### 7.2 加流式

```python
# 流式运行
for token, metadata in app.stream(
    {"messages": [...]},
    config,
    stream_mode="messages",
):
    if token.content:
        print(token.content, end="", flush=True)
```

> **和 `create_react_agent` 完全一样**——compile 时传 checkpointer，invoke 时传 config。

---

## 八、常见错误

### ❌ 忘了加条件路由的 END

```python
# ❌ 没有 END，图永远跑不完
graph.add_conditional_edges("llm", should_continue, ["tools"])  # 缺 END

# ✅ 正确
graph.add_conditional_edges("llm", should_continue, ["tools", END])
```

### ❌ 工具节点忘记 add_node

```python
# ❌ tools 没注册
graph.add_edge("tools", "llm")   # ValueError: Node 'tools' not found

# ✅ 先注册
graph.add_node("tools", tool_node)
graph.add_edge("tools", "llm")
```

### ❌ 路由函数返回值不在目标列表里

```python
# ❌ 返回 "retry" 但没在目标列表中
def should_continue(state) -> Literal["tools", "retry", END]:
    ...
graph.add_conditional_edges("llm", should_continue, ["tools", END])  # 缺 "retry"

# ✅ 匹配
graph.add_conditional_edges("llm", should_continue, ["tools", "retry", END])
```

### ❌ 循环死循环

```python
# ❌ 没有退出条件
def should_continue(state):
    return "tools"   # 永远返回 tools，死循环

# ✅ 加退出条件
def should_continue(state):
    if state["step_count"] > 10:
        return END
    if state["messages"][-1].tool_calls:
        return "tools"
    return END
```

---

## 九、速查表

### 构建 Agent 的基本步骤

| 步骤 | 代码 |
|------|------|
| 1. 定义状态 | `class AgentState(TypedDict): ...` |
| 2. 定义节点 | `def llm_node(state): ...` |
| 3. 定义路由 | `def should_continue(state) -> Literal[...]:` |
| 4. 创建图 | `graph = StateGraph(AgentState)` |
| 5. 注册节点 | `graph.add_node("llm", llm_node)` |
| 6. 添加边 | `graph.add_edge(START, "llm")` |
| 7. 添加条件边 | `graph.add_conditional_edges("llm", route, [...])` |
| 8. 编译 | `app = graph.compile()` |
| 9. 运行 | `app.invoke({"messages": [...]})` |

### 节点类型参考

| 节点类型 | 写法 | 用途 |
|---------|------|------|
| LLM 推理 | `model.invoke(state["messages"])` | AI 推理 |
| 工具执行 | `ToolNode(tools)` 或自写 | 执行工具函数 |
| 审核 | 调用另一个模型判断 | 质量控制 |
| 路由 | 不算节点，在条件边里 | 决定下一步 |
| 转换 | 对状态做变换 | 数据处理 |
| 输出 | 格式化最终结果 | 返回给用户 |

### 路由函数模板

```python
def route(state: AgentState) -> Literal["node_a", "node_b", END]:
    if 某个条件:
        return "node_a"
    if 另一个条件:
        return "node_b"
    return END
```

---

## 十、深入理解：create_react_agent 的内部结构

> 其实 `create_react_agent` 就是帮你写了这些代码：

```python
# create_react_agent 内部大致做了这些事（简化）：

# 1. 创建 LLM 节点
def llm(state):
    response = model.invoke(state["messages"])
    return {"messages": [response]}

# 2. 创建工具节点
tool_node = ToolNode(tools)

# 3. 定义路由
def route(state):
    if state["messages"][-1].tool_calls:
        return "tools"
    return END

# 4. 构建图
graph = StateGraph(MessagesState)
graph.add_node("llm", llm)
graph.add_node("tools", tool_node)
graph.add_edge(START, "llm")
graph.add_conditional_edges("llm", route, ["tools", END])
graph.add_edge("tools", "llm")

# 5. 编译
return graph.compile(checkpointer=checkpointer)
```

> **手动构建 = 你自己写这些代码。** 好处是你可以在任何地方加自定义逻辑。

---

## 下一步

- [[实战-多Agent协作]] — Supervisor 模式、多 Agent 编排
- [[实战-RAG-Agent]] — 检索增强生成 Agent
- [[功能-人工介入-Human-in-the-Loop]] — 在自定义 Agent 中加入审批