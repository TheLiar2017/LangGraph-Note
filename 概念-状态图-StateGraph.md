---
aliases: [StateGraph, 状态图]
tags: [概念, 基础, StateGraph]
---

# 📐 StateGraph 状态图详解

> **StateGraph** 是 LangGraph 最核心的概念。一切工作流都是围绕「状态」和「图」构建的。
> 理解了 StateGraph，就理解了 LangGraph 的一半。

---

## 一、StateGraph 是什么？

用一句话概括：

> **StateGraph = 一个有向图，节点是处理函数，边是流转路径，状态在图中传递和更新。**

### 与传统流程的区别

```
传统函数调用（线性）：
  输入 → func1 → func2 → func3 → 输出

StateGraph（图式）：
       ┌──────────┐
       │  START   │
       └────┬─────┘
            ▼
       ┌──────────┐
       │  Node A  │ ← 可以根据状态分支
       └─┬──────┬─┘
         ▼      ▼
   ┌────────┐ ┌────────┐
   │ Node B │ │ Node C │
   └───┬────┘ └───┬────┘
       ▼          ▼
       └────┬─────┘
            ▼
       ┌──────────┐
       │   END    │
       └──────────┘
```

### 核心思想

| 概念 | 对应物 | 说明 |
|:----:|:------:|------|
| **State**（状态） | 数据 | 在整个图中流动、被各节点读写的数据 |
| **Node**（节点） | 函数 | 接收状态，返回状态更新 |
| **Edge**（边） | 路径 | 连接节点，决定执行顺序 |
| **Graph**（图） | 流程 | 把以上三者组装成可执行的工作流 |

---

## 二、核心概念逐个击破

### 2.1 State（状态）

State 是一个 **TypedDict**，定义了图中共享的数据结构。

```python
from typing_extensions import TypedDict

class State(TypedDict):
    input: str
    output: str
    is_done: bool
```

**类比**：State 就像一张「表格」，每个字段是一个「格子」。节点函数可以读取和填写这些格子。

#### 最简单的状态

```python
class State(TypedDict):
    message: str
```

#### 复杂一点的状态

```python
class State(TypedDict):
    question: str
    context: list[str]
    answer: str
    steps: int
    is_correct: bool
```

---

### 2.2 State 的更新规则：Reducer

> **关键问题**：当节点返回 `{"steps": 1}` 时，状态里的 `steps` 是**替换**还是**累加**？

答案是——**默认替换**，但你可以用 **Reducer** 自定义行为。

#### 默认行为：直接替换

```python
class State(TypedDict):
    count: int

def add_one(state: State):
    return {"count": state["count"] + 1}  # 手动计算
```

#### 用 Annotated + operator.add 累加

```python
from typing import Annotated
import operator

class State(TypedDict):
    log: Annotated[list[str], operator.add]  # 新值追加到列表末尾

def step1(state: State):
    return {"log": ["step1 完成"]}   # log = ["step1 完成"]

def step2(state: State):
    return {"log": ["step2 完成"]}   # log = ["step1 完成", "step2 完成"]
```

#### 常见 Reducer 对比

| Reducer 效果 | 写法 | 适用场景 |
|:------------:|------|----------|
| **替换**（默认） | 不加 Annotated | 最终答案、当前状态 |
| **追加列表** | `Annotated[list, operator.add]` | 日志、消息历史 |
| **合并字典** | `Annotated[dict, operator.or_]` | 配置合并 |
| **自定义** | 自写函数 | 复杂合并逻辑 |

#### 自定义 Reducer

```python
from typing import Annotated

def keep_max(old: int, new: int) -> int:
    """保留最大值"""
    return max(old, new)

class State(TypedDict):
    best_score: Annotated[int, keep_max]

def score_node(state: State):
    # 无论新分数多少，只会保留更高的
    return {"best_score": 85}
```

---

### 2.3 Node（节点）

节点就是一个**普通的 Python 函数**，接收当前 State，返回**要更新的字段**。

#### 节点的基本写法

```python
def my_node(state: State):
    # 读取状态
    data = state["some_field"]

    # 做一些处理
    result = process(data)

    # 返回要更新的字段（只返回要改的，不用返回全部）
    return {"output": result}
```

#### 节点的几种形式

**方式一：用 `add_node("名字", 函数)`**

```python
def call_model(state: State):
    response = model.invoke(state["messages"])
    return {"messages": [response]}

graph = StateGraph(State)
graph.add_node("call_model", call_model)  # 节点名叫 "call_model"
```

**方式二：用函数名作为节点名（简写）**

```python
def call_model(state: State):
    return {"output": "hello"}

graph.add_node(call_model)  # 节点名自动用函数名 "call_model"
```

**方式三：用 lambda 或匿名函数**

```python
graph.add_node("greet", lambda state: {"message": "你好！"})
```

#### 多个节点也可以并行

```python
def fetch_from_db(state: State):
    return {"results": ["db_result"]}

def fetch_from_api(state: State):
    return {"results": ["api_result"]}

# 如果两个节点没有边相连，它们可能并行执行
graph.add_node("fetch_db", fetch_from_db)
graph.add_node("fetch_api", fetch_from_api)
```

---

### 2.4 Edge（边）

边决定节点之间的**执行顺序**。

#### 固定边：A 之后一定是 B

```python
from langgraph.graph import START, END

graph.add_edge(START, "node_a")   # 从 START 进入 node_a
graph.add_edge("node_a", "node_b")  # node_a 完成后执行 node_b
graph.add_edge("node_b", END)     # node_b 完成后结束
```

#### 条件边：根据状态动态决定下一步

```python
from typing import Literal

def should_continue(state: State) -> Literal["tool_node", END]:
    """根据最后一个消息是否有 tool_call 来决定走向"""
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tool_node"
    return END

graph.add_conditional_edges(
    "call_model",        # 从哪个节点出发
    should_continue,     # 判断函数
)
```

#### 条件边 + 路由映射

```python
def route(state: State) -> Literal["poem", "story", "joke"]:
    return state["category"]  # 直接返回目标节点名

graph.add_conditional_edges(
    "router",
    route,
    {
        "poem": "poem_writer",
        "story": "story_writer",
        "joke": "joke_writer",
    },
)
```

#### 边的类型汇总

```
START ──→ A ──→ B ──→ END
         │
         ├──→ C (条件边)
         └──→ D (条件边)
```

| 方法 | 用途 |
|------|------|
| `add_edge(START, "node")` | 设置入口节点 |
| `add_edge("node", END)` | 设置结束节点 |
| `add_edge("a", "b")` | 固定顺序执行 |
| `add_conditional_edges("a", func)` | 根据状态动态分支 |

---

## 三、从零构建第一个 StateGraph

### 示例 1：最简单的顺序执行

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

# 1️⃣ 定义状态
class State(TypedDict):
    message: str
    result: str

# 2️⃣ 定义节点
def greet(state: State):
    print(f"[greet] 收到: {state['message']}")
    return {"result": f"你好, {state['message']}!"}

def shout(state: State):
    print(f"[shout] 收到: {state['result']}")
    return {"result": state["result"].upper() + "!!!"}

# 3️⃣ 构建图
graph = StateGraph(State)
graph.add_node("greet", greet)
graph.add_node("shout", shout)
graph.add_edge(START, "greet")
graph.add_edge("greet", "shout")
graph.add_edge("shout", END)

# 4️⃣ 编译成可执行对象
app = graph.compile()

# 5️⃣ 运行
result = app.invoke({"message": "langgraph"})
print(result)
```

**输出：**
```
[greet] 收到: langgraph
[shout] 收到: 你好, langgraph!
{'message': 'langgraph', 'result': '你好, LANGGRAPH!!!'}
```

---

### 示例 2：条件分支

```python
from typing_extensions import TypedDict, Literal
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    number: int
    result: str

# 节点
def check_number(state: State):
    return {"result": ""}

def handle_even(state: State):
    return {"result": f"{state['number']} 是偶数"}

def handle_odd(state: State):
    return {"result": f"{state['number']} 是奇数"}

# 条件边的路由函数
def route_even_odd(state: State) -> Literal["handle_even", "handle_odd"]:
    if state["number"] % 2 == 0:
        return "handle_even"
    return "handle_odd"

# 构建图
graph = StateGraph(State)
graph.add_node("check_number", check_number)
graph.add_node("handle_even", handle_even)
graph.add_node("handle_odd", handle_odd)

graph.add_edge(START, "check_number")
graph.add_conditional_edges(
    "check_number",
    route_even_odd,
    {
        "handle_even": "handle_even",
        "handle_odd": "handle_odd",
    },
)
graph.add_edge("handle_even", END)
graph.add_edge("handle_odd", END)

app = graph.compile()

# 测试
print(app.invoke({"number": 4, "result": ""}))
# {'number': 4, 'result': '4 是偶数'}

print(app.invoke({"number": 7, "result": ""}))
# {'number': 7, 'result': '7 是奇数'}
```

---

### 示例 3：循环执行（LangGraph 的杀手级特性）

> 传统流程图是 DAG（有向无环图），LangGraph **支持循环**！

```python
from typing_extensions import TypedDict, Literal
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    counter: int
    messages: list[str]

def increment(state: State):
    count = state["counter"] + 1
    print(f"第 {count} 次执行")
    return {
        "counter": count,
        "messages": [f"已完成第 {count} 轮"],
    }

def should_continue(state: State) -> Literal["increment", END]:
    if state["counter"] < 5:
        return "increment"  # 继续循环
    return END              # 停止

graph = StateGraph(State)
graph.add_node("increment", increment)
graph.add_edge(START, "increment")
graph.add_conditional_edges("increment", should_continue)

app = graph.compile()
result = app.invoke({"counter": 0, "messages": []})
print(result)
```

**输出：**
```
第 1 次执行
第 2 次执行
第 3 次执行
第 4 次执行
第 5 次执行
{'counter': 5, 'messages': ['已完成第 1 轮', '已完成第 2 轮', ..., '已完成第 5 轮']}
```

---

## 四、MessagesState：专为对话设计的状态

如果你的状态主要是**消息列表**，LangGraph 提供了现成的 `MessagesState`：

```python
from langgraph.graph import StateGraph, MessagesState, START, END

# MessagesState 自带 messages 字段和 add_messages reducer
# add_messages 会自动去重（按 ID）、追加新消息

def chatbot(state: MessagesState):
    """模拟 LLM 回复"""
    return {"messages": [{"role": "ai", "content": "我是 AI 助手！"}]}

graph = StateGraph(MessagesState)
graph.add_node("chatbot", chatbot)
graph.add_edge(START, "chatbot")
graph.add_edge("chatbot", END)

app = graph.compile()

result = app.invoke({"messages": [{"role": "user", "content": "你好"}]})
for msg in result["messages"]:
    print(f"{msg['role']}: {msg['content']}")
```

**输出：**
```
user: 你好
ai: 我是 AI 助手！
```

### 扩展 MessagesState

```python
from langgraph.graph import MessagesState

class MyState(MessagesState):
    documents: list[str]   # 自定义字段
    user_id: str           # 自定义字段
```

---

## 五、StateGraph 完整 API 速查

### 构建阶段

| 方法 | 说明 | 示例 |
|------|------|------|
| `StateGraph(schema)` | 创建图构建器 | `StateGraph(State)` |
| `add_node(name, func)` | 添加节点 | `add_node("model", call_llm)` |
| `add_edge(from, to)` | 添加固定边 | `add_edge(START, "model")` |
| `add_conditional_edges(from, func, map?)` | 添加条件边 | `add_conditional_edges("model", route)` |
| `set_entry_point(name)` | 设置入口（等同 START 边） | `set_entry_point("model")` |

### 编译与运行

| 方法 | 说明 | 示例 |
|------|------|------|
| `compile()` | 编译为可执行图 | `app = graph.compile()` |
| `app.invoke(input)` | 同步执行，返回最终状态 | `app.invoke({"messages": [...]})` |
| `app.stream(input)` | 流式执行，逐步返回状态 | `for chunk in app.stream({...}):` |
| `app.get_graph()` | 获取可视化图对象 | `app.get_graph().draw_mermaid_png()` |

### 可视化图结构

```python
# 生成 Mermaid 代码
print(app.get_graph().draw_mermaid())

# 生成图片文件
png_data = app.get_graph().draw_mermaid_png()
with open("graph.png", "wb") as f:
    f.write(png_data)
```

---

## 六、常见误区 & 注意事项

### ⚠️ 1. 节点只返回要更新的字段

```python
# ❌ 错误：返回了全部状态（虽然能跑，但不推荐）
def my_node(state: State):
    return {"input": state["input"], "output": "done"}

# ✅ 正确：只返回需要更新的字段
def my_node(state: State):
    return {"output": "done"}
```

### ⚠️ 2. 用 add_edge 连接的节点必须都已 add_node

```python
# ❌ 错误：忘了 add_node("b")
graph.add_edge("a", "b")   # 编译时报错
```

### ⚠️ 3. 条件边的函数必须返回节点名

```python
# ❌ 错误：返回了任意字符串
def route(state):
    return "ok"

# ✅ 正确：返回已注册的节点名或 END
def route(state) -> Literal["node_a", "node_b"]:
    return "node_a"
```

### ⚠️ 4. 循环时注意设置退出条件

```python
# ❌ 危险：没有退出条件，会死循环
def loop(state):
    return {"count": state["count"] + 1}
# count 永远 < 10，永远循环！

# ✅ 安全：设置最大步数
def loop(state):
    return {"count": state["count"] + 1}

def should_continue(state):
    return "loop" if state["count"] < 10 else END
```

---

## 七、一张图总结

```
┌───────────────────────────────────────────────────┐
│                  StateGraph 工作流程               │
│                                                   │
│  1. 定义 State (TypedDict + Annotated + Reducer)   │
│         │                                         │
│         ▼                                         │
│  2. 定义 Node (普通 Python 函数)                    │
│         │                                         │
│         ▼                                         │
│  3. 连接 Edge (固定边 / 条件边)                     │
│         │                                         │
│         ▼                                         │
│  4. compile() 编译成 Pregel 执行引擎                │
│         │                                         │
│         ▼                                         │
│  5. invoke() / stream() 运行                       │
│         │                                         │
│         ▼                                         │
│  6. 获取最终 State                                  │
└───────────────────────────────────────────────────┘
```

---

## 下一步

- [[概念-节点与边]] — 深入理解 Node 和 Edge 的高级用法
- [[概念-状态管理]] — Reducer、MessagesState、add_messages 详解
- [[实战-自定义Agent]] — 用 StateGraph 构建真实 Agent