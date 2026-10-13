---
aliases: [工具调用, Tool Calling, 工具, Tool, Function Calling]
tags: [功能, 基础, 工具, Tool]
---

# 🔧 工具调用（Tool Calling）

> AI 模型只会「说话」，但你可以**教它做事**——查天气、算数学、搜数据库、发邮件……
> 这就是工具调用：让 AI 在对话中**请求执行外部函数**，拿到结果后继续回答。
> 本文从零开始讲清楚工具调用，完全不需要前置知识。

---

## 一、工具调用是什么？

### 1.1 一句话理解

> **工具调用 = AI 说"我想用某个工具"，你的代码去执行，把结果告诉 AI。**

```text
👤 用户: "3 加 4 等于多少？"

🤖 AI 模型: "我想调用 add(3, 4)"        ← 模型不直接算，而是说"我要用计算器"

💻 你的代码: 执行 add(3, 4) → 7          ← 你的代码去跑

🤖 AI 模型: "3 + 4 = 7"                  ← 模型拿到结果，组织语言回答
```

### 1.2 为什么需要工具调用？

AI 模型有局限性：

| AI 做不到的事 | 用工具解决 |
|--------------|-----------|
| 查实时数据（天气、股价） | 🔍 搜索工具 |
| 精确计算（大数乘法） | 🧮 计算器工具 |
| 读写数据库 | 🗄️ 数据库工具 |
| 调用外部 API | 🌐 API 工具 |
| 读取文件 | 📁 文件工具 |

### 1.3 工具调用的完整流程

```text
┌──────────────────────────────────────────────────────────────────┐
│                      工具调用完整流程                             │
│                                                                  │
│  用户        AI 模型              你的代码           工具函数      │
│   │            │                    │                  │         │
│   │  1.提问    │                    │                  │         │
│   │ ─────────→ │                    │                  │         │
│   │            │                    │                  │         │
│   │            │  2.返回 tool_call  │                  │         │
│   │            │  "我要用 add(3,4)" │                  │         │
│   │            │ ──────────────────→│                  │         │
│   │            │                    │  3.执行工具      │         │
│   │            │                    │ ────────────────→│         │
│   │            │                    │                  │         │
│   │            │                    │  4.返回结果 7    │         │
│   │            │                    │ ←────────────────│         │
│   │            │                    │                  │         │
│   │            │  5.返回 ToolMessage │                  │         │
│   │            │  content=7         │                  │         │
│   │            │ ←──────────────────│                  │         │
│   │            │                    │                  │         │
│   │  6.回答    │                    │                  │         │
│   │  "3+4=7"  │                    │                  │         │
│   │ ←───────── │                    │                  │         │
│   │            │                    │                  │         │
└──────────────────────────────────────────────────────────────────┘
```

---

## 二、定义工具：@tool 装饰器

### 2.1 最简单的工具

```python
from langchain.tools import tool

@tool
def add(a: int, b: int) -> int:
    """将两个数字相加。

    Args:
        a: 第一个数字
        b: 第二个数字
    """
    return a + b
```

**每个部分的作用：**

```text
@tool                     ← 装饰器：把这个函数变成"工具"
def add(a: int, b: int)   ← 函数签名：AI 看到的参数格式
    -> int:               ← 返回类型
    """将两个数字相加。     ← 描述：AI 靠它判断什么时候该用这个工具

    Args:
        a: 第一个数字      ← 参数说明：AI 知道怎么填参数
        b: 第二个数字
    """
    return a + b          ← 实际执行的代码
```

### 2.2 查看工具的元数据

```python
print(add.name)        # → "add"（工具名，默认为函数名）
print(add.description) # → "将两个数字相加。..."
print(add.args)        # → {'a': {...}, 'b': {...}}（参数定义）
```

**模拟输出：**
```text
add
将两个数字相加。

Args:
    a: 第一个数字
    b: 第二个数字
{'a': {'title': 'A', 'type': 'integer'}, 'b': {'title': 'B', 'type': 'integer'}}
```

### 2.3 自定义工具名和描述

```python
@tool("calculator")      # 自定义工具名
def calculate(expression: str) -> str:
    """计算数学表达式的结果。当用户问数学问题时使用这个工具。"""  # 描述给 AI 看
    return str(eval(expression))

print(calculate.name)  # → "calculator"（不是函数名）
```

### 2.4 多个参数的工具

```python
@tool
def search_database(table: str, column: str, keyword: str, limit: int = 10) -> str:
    """在数据库中搜索指定内容。

    Args:
        table: 表名，如 "users", "products"
        column: 列名，如 "name", "email"
        keyword: 搜索关键词
        limit: 返回结果数量，默认 10 条
    """
    # 实际的数据库搜索逻辑
    return f"在 {table}.{column} 中搜索 '{keyword}' 的结果..."
```

> **注意**：`docstring` 中的 `Args:` 部分**非常重要**——AI 靠它理解每个参数的含义，从而正确填写参数。

---

## 三、绑定工具到 AI 模型

### 3.1 bind_tools

定义工具后，需要告诉 AI 模型它有哪些工具可用：

```python
from langchain_openai import ChatOpenAI

# 创建模型（以 DeepSeek 为例）
model = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx",
)

# 定义工具
@tool
def add(a: int, b: int) -> int:
    """将两个数字相加。"""
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """将两个数字相乘。"""
    return a * b

# ★ 绑定工具到模型
tools = [add, multiply]
model_with_tools = model.bind_tools(tools)
```

### 3.2 绑定后模型的行为变化

```python
# 不绑定工具：AI 直接回答
response = model.invoke("3 加 4 等于多少？")
print(response.content)   # → "3 + 4 = 7"（AI 自己算的）

# 绑定工具：AI 可能请求调用工具
response = model_with_tools.invoke("3 加 4 等于多少？")
print(response.tool_calls)  # → [{'name': 'add', 'args': {'a': 3, 'b': 4}, ...}]
```

**模拟输出（绑定工具后）：**
```python
[
    {
        'name': 'add',           # 要调用的工具名
        'args': {'a': 3, 'b': 4}, # 参数（AI 自动从对话中提取）
        'id': 'call_abc123',     # 调用 ID（用于匹配结果）
        'type': 'tool_call',     # 类型标记
    }
]
```

### 3.3 tool_calls 是什么？

`tool_calls` 是 AI 模型返回的**工具调用请求**，不是工具执行的结果：

```text
AI 模型返回的 tool_calls:
  {
    "name": "add",          ← 用哪个工具
    "args": {"a": 3, "b": 4}, ← 参数
    "id": "call_abc123"     ← 调用 ID
  }

注意：模型只是"说"要调用，实际上没有执行！
      需要你的代码去执行，然后把结果传回给模型。
```

---

## 四、执行工具：ToolNode

### 4.1 用 ToolNode 自动执行工具

`ToolNode` 是 LangGraph 预置的节点，自动帮你执行工具并生成 `ToolMessage`：

```python
from langgraph.prebuilt import ToolNode

# 创建工具执行节点
tool_node = ToolNode([add, multiply])

# ToolNode 自动完成：
# 1. 从 AI 消息中提取 tool_calls
# 2. 执行对应的工具函数
# 3. 生成 ToolMessage（包含执行结果）
# 4. 返回更新后的 state
```

### 4.2 手动执行工具（了解原理）

```python
from langchain.messages import ToolMessage

def manual_tool_node(state: MessagesState):
    """手动执行工具（理解原理用）"""
    result = []
    for tool_call in state["messages"][-1].tool_calls:
        # 根据名字找工具
        tool = tools_by_name[tool_call["name"]]

        # 执行工具
        observation = tool.invoke(tool_call["args"])

        # 生成 ToolMessage（用 tool_call_id 匹配）
        result.append(ToolMessage(
            content=str(observation),
            tool_call_id=tool_call["id"]    # ← 关键：用 ID 匹配
        ))

    return {"messages": result}
```

### 4.3 ToolMessage 是什么？

```text
消息类型流转：
  HumanMessage("3+4=?")       → 用户提问
  AIMessage(tool_calls=[...])  → AI 请求调用工具
  ToolMessage(content="7")     → 工具执行结果（← ToolNode 生成的）
  AIMessage("3+4=7")          → AI 拿到结果后的最终回答
```

| 消息类型 | 谁生成 | 作用 |
|----------|--------|------|
| `HumanMessage` | 用户 | 提问 |
| `AIMessage` (带 tool_calls) | AI 模型 | 请求调用工具 |
| `ToolMessage` | **ToolNode** | 工具执行结果 |
| `AIMessage` (无 tool_calls) | AI 模型 | 最终回答 |

---

## 五、路由：tools_condition

### 5.1 判断要不要调用工具

AI 模型返回后，需要判断：它是想调用工具，还是直接回答？

```python
from langgraph.prebuilt import tools_condition

# tools_condition 自动判断：
# - 有 tool_calls → 去执行工具
# - 没有 → 结束（END）
```

### 5.2 手写路由逻辑

```python
from typing import Literal
from langgraph.graph import END

def should_continue(state: MessagesState) -> Literal["tools", END]:
    """判断是否需要执行工具"""
    last_message = state["messages"][-1]

    if last_message.tool_calls:      # AI 请求了工具调用
        return "tools"                # → 去执行工具
    return END                        # → 没有工具调用，直接结束
```

---

## 六、完整示例：手动构建工具调用 Agent

```python
from typing import Literal
from typing_extensions import TypedDict, Annotated
from langchain.messages import HumanMessage, SystemMessage, ToolMessage, AnyMessage
from langchain_openai import ChatOpenAI
from langchain.tools import tool
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import ToolNode
import operator

# ═══ 1. 定义工具 ═══
@tool
def add(a: int, b: int) -> int:
    """将两个数字相加。

    Args:
        a: 第一个数字
        b: 第二个数字
    """
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """将两个数字相乘。

    Args:
        a: 第一个数字
        b: 第二个数字
    """
    return a * b

@tool
def get_weather(city: str) -> str:
    """获取指定城市的天气信息。

    Args:
        city: 城市名，如"北京"、"上海"
    """
    # 模拟天气 API
    weather_data = {
        "北京": "晴天，25°C",
        "上海": "多云，22°C",
        "广州": "小雨，28°C",
    }
    return weather_data.get(city, f"未找到 {city} 的天气数据")

# ═══ 2. 创建模型并绑定工具 ═══
model = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx",
)

tools = [add, multiply, get_weather]
tools_by_name = {tool.name: tool for tool in tools}
model_with_tools = model.bind_tools(tools)

# ═══ 3. 定义模型节点 ═══
def llm_call(state: MessagesState):
    """AI 模型推理：决定是否调用工具"""
    response = model_with_tools.invoke(
        [
            SystemMessage(content="你是一个有帮助的助手，可以计算数字和查询天气。")
        ]
        + state["messages"]
    )
    return {"messages": [response]}

# ═══ 4. 定义工具执行节点 ═══
tool_node = ToolNode(tools)

# ═══ 5. 定义路由 ═══
def should_continue(state: MessagesState) -> Literal["tools", END]:
    """有 tool_calls 就去执行工具，否则结束"""
    if state["messages"][-1].tool_calls:
        return "tools"
    return END

# ═══ 6. 构建图 ═══
graph = StateGraph(MessagesState)
graph.add_node("llm_call", llm_call)
graph.add_node("tools", tool_node)

graph.add_edge(START, "llm_call")
graph.add_conditional_edges("llm_call", should_continue, ["tools", END])
graph.add_edge("tools", "llm_call")   # 工具执行后回到模型，继续推理

app = graph.compile()

# ═══ 7. 运行 ═══
result = app.invoke({
    "messages": [HumanMessage(content="北京今天天气怎么样？")]
})

for msg in result["messages"]:
    msg.pretty_print()
```

**模拟输出：**
``` text
=============================== Human Message ===============================
北京今天天气怎么样？

================================= Ai Message ================================
Tool Calls:
  get_weather (call_abc123)
 Call ID: call_abc123
  Args:
    city: 北京

=============================== Tool Message ================================
北京：晴天，25°C

================================= Ai Message =================================
北京今天晴天，气温 25°C。
```

**消息流转过程：**
```text
messages = [
  HumanMessage("北京今天天气怎么样？"),           # 1. 用户提问
  AIMessage(tool_calls=[get_weather("北京")]),    # 2. AI 请求调用天气工具
  ToolMessage("北京：晴天，25°C"),                # 3. 工具执行结果
  AIMessage("北京今天晴天，25°C。"),              # 4. AI 最终回答
]
```

---

## 七、快捷方式：create_react_agent

不想手动构建图？用 `create_react_agent` 一行搞定：

```python
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI
from langchain.tools import tool

# 1. 定义工具
@tool
def add(a: int, b: int) -> int:
    """将两个数字相加。"""
    return a + b

@tool
def get_weather(city: str) -> str:
    """获取城市天气。"""
    return f"{city}: 晴天 25°C"

# 2. 创建模型
model = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx",
)

# 3. 一行创建 Agent
agent = create_react_agent(model, tools=[add, get_weather])

# 4. 运行
result = agent.invoke({"messages": [{"role": "user", "content": "北京天气？3+4=?"}]})
for msg in result["messages"]:
    print(msg.content)
```

> **`create_react_agent` = 自动帮你搭建了 §六 的完整图结构**（llm_call → tools → llm_call → ... 的循环）。

### 手动构建 vs create_react_agent

| | 手动构建 | `create_react_agent` |
|---|:---:|:---:|
| **灵活性** | ⭐⭐⭐ 完全自定义 | ⭐⭐ 固定结构 |
| **上手难度** | 较高 | ⭐⭐⭐ 一行搞定 |
| **适合场景** | 需要自定义节点/路由 | 标准的工具调用 Agent |
| **代码量** | 50+ 行 | 3 行 |

---

## 八、工具设计最佳实践

### 8.1 描述写清楚

```python
# ❌ 差的描述：AI 不知道什么时候该用
@tool
def do_something(x: str):
    """做点什么。"""
    pass

# ✅ 好的描述：AI 知道何时、如何使用
@tool
def search_products(keyword: str, category: str = "") -> str:
    """根据关键词搜索商品信息。当用户想查找、搜索商品时使用此工具。

    Args:
        keyword: 搜索关键词，如"手机"、"笔记本电脑"
        category: 可选的商品分类，如"电子产品"、"服饰"，不填则搜索全部分类
    """
    ...
```

### 8.2 参数类型用 Python 类型标注

```python
# ✅ 类型清晰，AI 能正确填写参数
@tool
def book_room(guest_name: str, check_in: str, nights: int, room_type: str = "standard"):
    ...

# ❌ 没有类型，AI 可能填错
@tool
def book_room(guest_name, check_in, nights, room_type):
    ...
```

### 8.3 返回值用字符串

```python
# ✅ 返回字符串，AI 容易理解
@tool
def get_stock_price(symbol: str) -> str:
    """获取股票实时价格。"""
    price = fetch_price(symbol)
    return f"{symbol} 当前价格: ${price}，涨幅: +2.3%"

# ⚠️ 返回复杂对象，AI 可能难以理解
@tool
def get_stock_price(symbol: str) -> dict:
    return {"symbol": symbol, "price": 123.45, "change": 2.3, ...}
```

### 8.4 工具不要太多

| 工具数量 | 效果 |
|:--------:|------|
| 3-5 个 | ✅ 最佳，AI 选择准确 |
| 5-10 个 | ⚠️ 可能需要更详细的描述 |
| 10+ 个 | ❌ AI 可能选错，考虑分组或分层 |

---

## 九、常见问题

### ❌ AI 不调用工具怎么办？

1. 检查工具的 `docstring` 描述是否清楚
2. 在 SystemMessage 中明确告知何时使用工具
3. 确认 `bind_tools` 已调用

```python
# 在系统提示中强调
SystemMessage(content="你是一个助手。计算数字时必须使用 add/multiply 工具。")
```

### ❌ AI 调用工具但参数填错了？

1. `docstring` 中的 `Args:` 部分写详细
2. 用 Python 类型标注（`int`, `str`, `float`）
3. 在参数描述中给示例值

### ❌ 工具执行报错？

在工具内部处理异常，返回错误信息而不是抛出：

```python
@tool
def fetch_data(url: str) -> str:
    """获取网页数据。"""
    try:
        response = requests.get(url, timeout=5)
        return response.text[:500]
    except Exception as e:
        return f"获取数据失败: {str(e)}"   # 返回错误信息给 AI
```

### ❌ 工具调用循环停不下来？

设置最大迭代次数（`recursion_limit`）：

```python
result = app.invoke(
    {"messages": [...]},
    config={"recursion_limit": 10}   # 最多 10 步，超过则报错
)
```

---

## 十、速查表

| 操作 | 代码 |
|------|------|
| 定义工具 | `@tool` 装饰器 |
| 绑定工具 | `model.bind_tools([tool1, tool2])` |
| 创建工具执行节点 | `ToolNode([tool1, tool2])` |
| 判断是否调用工具 | `tools_condition` 或自写 `should_continue` |
| 一行创建 Agent | `create_react_agent(model, tools)` |
| 查看工具调用请求 | `response.tool_calls` |
| 工具结果消息 | `ToolMessage(content=result, tool_call_id=id)` |
| 限制最大步数 | `config={"recursion_limit": 10}` |

---

## 十一、深入理解：工具调用的本质

### 11.1 AI、工具、开发者，各自的角色

> **AI 是「决策者」，工具是「执行者」，开发者是「搭建者」。**

```text
开发者的工作：                        AI 的工作：
  1. 写工具函数（做什么、怎么做）        1. 判断要不要调工具（根据对话内容）
  2. 注册到系统                          2. 填参数（从对话中提取信息）
  3. 执行、返回结果                      3. 拿到结果后组织语言回答
```

| 角色 | 负责什么 | 不负责什么 |
|------|----------|-----------|
| **开发者** | 写工具代码、注册工具、搭建执行流程 | 不决定何时调用 |
| **AI 模型** | 判断何时调用、填什么参数、组织最终回答 | 不执行工具代码 |
| **LangGraph 框架** | 把 AI 的请求转发给工具、把结果传回 AI | 不写工具逻辑 |

**类比：餐厅**

```text
AI 就像一个餐厅服务员 👨‍🍳
  - 客人（用户）问："有没有红烧肉？"
  - 服务员看菜单（工具列表），说："我帮你查一下厨房"
  - 厨房（工具函数）说："有，38 元"
  - 服务员告诉客人："红烧肉有，38 元"

服务员不会做菜（AI 不执行代码），
厨房不会接待客人（工具不会和用户对话），
餐厅老板（开发者）决定菜单上有什么菜。
```

### 11.2 什么适合用工具，什么适合让 AI 做

> **工具 = AI 的「手和眼」**（帮它拿数据、执行操作）
> **AI = AI 的「大脑」**（做判断、做推理、做生成）

**判断标准：**

| | 用工具 ✅ | 让 AI 做 ✅ |
|---|:---:|:---:|
| **本质** | 确定性操作、查数据、调 API | 推理、生成、综合判断 |
| **输入确定时输出是否确定？** | ✅ 是（同一输入，同一结果） | ❌ 否（可能有多种合理方案） |
| **例子** | 查数据库、算数学、发邮件、读文件 | 写方案、写文章、分析问题、给建议 |

**反面例子：排班计划**

```text
❌ 不适合用工具：
   生成排班方案   → 需要综合多种因素推理，不是确定性操作
   解释排班理由   → 需要自然语言生成
   处理冲突和例外 → 需要判断和权衡

✅ 适合用工具：
   查员工名单     → get_employees()
   查排班规则     → get_scheduling_rules()
   查请假记录     → get_leave_records()
   查节假日       → get_holiday_list()
```

**完整流程示例：**

```python
from langchain.tools import tool

# ═══ 工具：负责查数据（确定性的）═══

@tool
def get_employees(department: str) -> str:
    """获取指定部门的员工名单。"""
    return "张三(高级), 李四(中级), 王五(初级), 赵六(高级)"

@tool
def get_leave_records(department: str, month: str) -> str:
    """获取指定部门的请假记录。"""
    return "张三 10/15-10/17 请假, 王五 10/20 请假"

@tool
def get_scheduling_rules() -> str:
    """获取排班规则。"""
    return "1. 每天至少 2 人在岗; 2. 高级员工不能连续值班超过 3 天; 3. 每人每月至少 4 天休息"

@tool
def get_holiday_list(month: str) -> str:
    """获取法定节假日。"""
    return "10/01-10/07 国庆节（法定假日）"

# ═══ AI：负责推理和生成（不确定的）═══

# AI 拿到数据后的工作流程：
# 1. 调用 get_employees("技术部")     → 拿到 4 个员工
# 2. 调用 get_leave_records(...)      → 拿到请假记录
# 3. 调用 get_scheduling_rules()      → 拿到排班规则
# 4. 调用 get_holiday_list("10月")    → 拿到节假日
#
# 5. 【AI 推理】综合以上数据，生成排班方案 ← 这步不能用工具！
#    "10月排班方案：
#     10/01-10/07: 张三+赵六（高级组合值班）
#     10/08-10/14: 李四+王五
#     ...
#     注意：张三 10/15-10/17 请假，已安排赵六顶班"
```

> **一句话总结**：让 AI 当大脑，让工具当手脚，各司其职。工具帮 AI「看到」数据，AI 自己「想」怎么做。

---

## 下一步

- [[功能-流式输出-Streaming]] — 工具调用结果的流式输出
- [[实战-React-Agent]] — 构建完整的 ReAct Agent
- [[实战-RAG-Agent]] — 检索增强生成 Agent