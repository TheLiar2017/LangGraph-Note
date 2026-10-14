---
aliases: [人工介入, Human-in-the-Loop, HITL, interrupt, 中断, 人工审批]
tags: [功能, 进阶, Human-in-the-Loop, interrupt]
---

# 🙋 人工介入（Human-in-the-Loop）

> AI 不是万能的——有些操作必须**人来拍板**。
> 发邮件？删数据？转账？——AI 可以准备好，但**按下按钮的必须是人**。
> 这就是人工介入：让工作流在关键节点**暂停**，等人确认后再继续。
> 本文从零讲清 LangGraph 的人工介入机制。

---

## 一、人工介入是什么？

### 1.1 一句话理解

> **人工介入 = AI 干到一半停下来，问人「这样做行吗？」，人说行再继续。**

```text
❌ 没有人工介入：

  用户: "帮我把这封邮件发给客户"
  AI:   自动执行 → 📧 邮件已发出（内容写错了，完了...）


✅ 有人工介入：

  用户: "帮我把这封邮件发给客户"
  AI:   草稿写好了，内容如下...
        ⏸️ 暂停，等待确认
  👤 人类: 看了看草稿 → "把措辞改得正式一点"
  AI:   修改后 → ⏸️ 再次等待确认
  👤 人类: "没问题，发吧"
  AI:   📧 邮件已发出
```

### 1.2 为什么需要人工介入？

| 场景 | 风险 | 人工介入 |
|------|------|---------|
| 发送邮件/消息 | ❌ 发出去撤不回 | 人确认后再发 |
| 删除数据 | ❌ 删了就没了 | 人批准后再删 |
| 转账/支付 | ❌ 钱转错很麻烦 | 人审核后再转 |
| 修改代码 | ❌ 改坏了系统崩溃 | 人 review 后再合 |
| 调用外部 API | ❌ 可能产生费用 | 人批准后再调 |
| 高风险决策 | ❌ AI 可能判断错 | 人做最终决定 |

> **一句话**：不可逆的操作、高风险的操作、需要人类判断的操作——都需要人工介入。

### 1.3 人工介入的两种模式

```text
模式一：审批型（Approval）
  AI 准备好操作 → 暂停 → 人「批准/拒绝」→ 执行或取消

模式二：修正型（Correction）
  AI 给出结果 → 暂停 → 人「修改」→ 用修改后的结果继续
```

| | 审批型 | 修正型 |
|---|:---:|:---:|
| **人做什么** | 批准或拒绝 | 修改内容 |
| **例子** | "这封邮件可以发吗？" | "把语气改正式一点" |
| **人提供的输入** | 是/否 | 具体的修改内容 |

---

## 二、interrupt：让工作流暂停

### 2.1 interrupt 是什么？

> `interrupt` 是 LangGraph 提供的**暂停函数**——在节点里调用它，工作流就会停下来，把控制权交给人。

```python
from langgraph.types import interrupt

def my_node(state: State):
    # ... 做一些事 ...

    # ★ 调用 interrupt → 工作流暂停，等待人的输入
    human_response = interrupt("请确认：是否执行？")

    # 人回复后，从这里继续执行
    if human_response == "yes":
        do_something()
```

### 2.2 interrupt 的工作流程

```text
┌──────────────────────────────────────────────────────┐
│                    interrupt 流程                     │
│                                                      │
│  节点执行中                                           │
│     │                                                │
│     ▼                                                │
│  调用 interrupt("请确认")                             │
│     │                                                │
│     ▼                                                │
│  ⏸️ 工作流暂停，返回 interrupt 信息                    │
│     │                                                │
│     ▼                                                │
│  👤 人类看到提示，做出决定                             │
│     │                                                │
│     ▼                                                │
│  调用 app.invoke(Command(resume="yes"))              │
│     │                                                │
│     ▼                                                │
│  ▶️ 从 interrupt 处继续执行，human_response = "yes"   │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### 2.3 最简单的例子

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

# 1. 状态
class State(TypedDict):
    question: str
    answer: str
    approved: bool

# 2. 节点（带 interrupt）
def ask_human(state: State):
    # ★ interrupt 会暂停，把 prompt 传给人看
    response = interrupt(f"AI 建议回答：{state.get('answer', '')}\n是否批准？(yes/no)")

    return {"approved": response == "yes"}

# 3. 构建图
graph = StateGraph(State)
graph.add_node("ask_human", ask_human)
graph.add_edge(START, "ask_human")
graph.add_edge("ask_human", END)

# 4. 编译（必须用 checkpointer！）
checkpointer = InMemorySaver()
app = graph.compile(checkpointer=checkpointer)

# 5. 第一次运行——会暂停在 interrupt
config = {"configurable": {"thread_id": "对话-001"}}
result = app.invoke({"question": "今天天气怎么样？"}, config)

print(result["__interrupt__"])   # → interrupt(value='AI 建议回答：...', ...)
```

**模拟输出：**
```text
interrupt(
    value='AI 建议回答：\n是否批准？(yes/no)',
    id='...',
)
```

> **注意**：`interrupt` 需要 checkpointer！没有 checkpointer 会报错。

### 2.4 恢复执行：Command(resume=...)

```python
from langgraph.types import Command

# 人类说 "yes"，恢复执行
result = app.invoke(Command(resume="yes"), config)

print(result["approved"])   # → True
```

**模拟输出：**
```text
approved = True
```

> `Command(resume="值")` 的「值」会传给 `interrupt()` 的返回值。

### 2.5 用户输入是固定的还是随便的？

**随便输入，不限制。** `interrupt` 只是暂停，返回的值就是 `Command(resume=...)` 传入的值。LangGraph 不做任何验证——验证和解析是你的代码的事。

**方式一：固定选项 + if 判断**

```python
choice = interrupt("选择操作：approve / reject / edit")

# 你自己验证
if choice == "approve":
    return {"approved": True}
elif choice == "reject":
    return {"approved": False}
else:
    # 用户输入了 "ok"、"YES"、"批准"？只有匹配上才生效
    print(f"未知选项：{choice}")
```

**方式二：用户随便说，让 AI 理解**

```python
human_input = interrupt("请描述你的修改意见")

# 把人的自由输入交给 AI 解读
response = model.invoke(
    f"用户说：{human_input}\n原文：{state['content']}\n请根据用户意见修改"
)
return {"content": response.content}
```

> 用户可以说"语气活泼点"、"加个 emoji"、"太长了缩短"——AI 能理解自然语言。

**方式三：返回复杂结构**

```python
result = interrupt("请提供修改内容")

# result 可以是字符串、dict、list……随便你
if isinstance(result, dict):
    return {"content": result["new_content"], "reason": result["reason"]}
else:
    return {"content": result}
```

> **核心理解**：LangGraph 只管「暂停」和「恢复」，用户输入什么、怎么处理，完全是你的自由。

### 2.6 用户怎么输入？取决于你的应用界面

`interrupt` 本身**不提供任何 UI**——它只是暂停并返回数据。怎么展示给人、怎么接收输入，是你自己搭的。

**场景一：命令行（`input()`）**

```python
result = app.invoke({...}, config)
prompt = result["__interrupt__"][0].value

user_input = input(f"{prompt}\n请输入: ")        # 用户打字输入
result = app.invoke(Command(resume=user_input), config)
```

**场景二：Web 前端（按钮）**

```python
# 后端返回 interrupt 信息
@app.post("/chat")
async def chat(request: dict):
    result = app.invoke({...}, config)
    if "__interrupt__" in result:
        return {
            "status": "waiting_human",
            "prompt": result["__interrupt__"][0].value,
            "options": ["approve", "reject", "edit"],   # 前端据此渲染按钮
        }

# 用户点按钮后，前端调用
@app.post("/resume")
async def resume(choice: str):
    return app.invoke(Command(resume=choice), config)
```

```text
前端界面：
┌─────────────────────────────────┐
│  AI 草稿：张三你好，邀请你...    │
│                                  │
│  ┌──────────┐ ┌──────────┐      │
│  │  ✅ 批准  │ │  ❌ 拒绝  │      │  ← 用户点按钮
│  └──────────┘ └──────────┘      │
│  ┌──────────┐                   │
│  │  ✏️ 修改  │                   │
│  └──────────┘                   │
└─────────────────────────────────┘
```

**场景三：聊天界面（用户打字）**

```text
AI: 请审批邮件草稿：
    张三你好，邀请你参加周五的会议...
    请输入 approve / reject / edit

用户: approve              ← 直接打字

AI: 📧 邮件已发送！
```

> **一句话**：`interrupt` 的值是给**你代码**看的，不是给用户看的。你怎么展示、怎么收输入，完全自己决定。 

#### 关键理解：前端拿到 `options` 后怎么渲染，完全是前端的自由

```text
interrupt 返回的数据（如 options）          你的前端
─────────────────────────────           ─────────────
{                                       拿到数据后，自己决定怎么展示：
  "prompt": "请选择操作",                 ├─ 渲染按钮  [✅ 批准] [❌ 拒绝]
  "options": ["approve",                ├─ 渲染下拉框
              "reject",                 ├─ 渲染卡片选择
              "edit"]                   └─ 渲染输入框
}
     │                                         │
     │  只是普通 JSON 数据                       │  用户操作后
     ▼                                         ▼
后端不关心前端怎么展示                      把选择发回后端
只关心 Command(resume=...) 传回的值        Command(resume="approve")
```

> **前端甚至可以加 `options` 里没有的选项**——比如加一个「稍后再审」按钮，只要 `Command(resume="later")` 传回来时你的后端代码能处理就行。LangGraph 不做任何限制。

---

## 三、完整实战：审批型工作流

> 场景：AI 帮用户起草邮件，人审批后才发送。

### 3.1 完整代码

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver
from langchain_openai import ChatOpenAI

# ═══ 1. 状态 ═══
class State(TypedDict):
    request: str          # 用户请求
    draft: str            # AI 草稿
    approved: bool        # 人是否批准
    final_result: str     # 最终结果

# ═══ 2. 模型 ═══
model = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx",
)

# ═══ 3. 节点 ═══

def generate_draft(state: State):
    """AI 生成邮件草稿"""
    response = model.invoke(f"写一封简短的邮件：{state['request']}")
    return {"draft": response.content}

def human_review(state: State):
    """人工审批"""
    # ★ interrupt：暂停，等人审批
    decision = interrupt({
        "action": "请审批邮件草稿",
        "draft": state["draft"],
        "options": ["approve", "reject", "edit"],
    })

    if decision == "approve":
        return {"approved": True}
    elif decision == "reject":
        return {"approved": False, "final_result": "❌ 邮件已被拒绝，未发送"}
    else:  # edit
        # 人可以传修改意见
        return {"approved": False}

def send_email(state: State):
    """发送邮件（模拟）"""
    return {"final_result": f"📧 邮件已发送！\n\n{state['draft']}"}

def handle_reject(state: State):
    """处理拒绝"""
    return {"final_result": "❌ 邮件被拒绝，未发送"}

# ═══ 4. 路由 ═══
def route_after_review(state: State):
    if state["approved"]:
        return "send_email"
    return "handle_reject"

# ═══ 5. 构建图 ═══
graph = StateGraph(State)
graph.add_node("generate_draft", generate_draft)
graph.add_node("human_review", human_review)
graph.add_node("send_email", send_email)
graph.add_node("handle_reject", handle_reject)

graph.add_edge(START, "generate_draft")
graph.add_edge("generate_draft", "human_review")
graph.add_conditional_edges("human_review", route_after_review)
graph.add_edge("send_email", END)
graph.add_edge("handle_reject", END)

# ═══ 6. 编译（必须带 checkpointer）═══
checkpointer = InMemorySaver()
app = graph.compile(checkpointer=checkpointer)
```

### 3.2 运行流程

```python
config = {"configurable": {"thread_id": "email-001"}}

# ═══ 第一步：运行到 human_review 会暂停 ═══
result = app.invoke({"request": "邀请张三参加周五的会议"}, config)

# 查看 interrupt 信息
interrupt_info = result["__interrupt__"][0]
print(interrupt_info.value)
```

**模拟输出：**
```text
{
    'action': '请审批邮件草稿',
    'draft': '张三你好，\n\n诚挚邀请你参加本周五下午 2 点的项目会议。\n\n谢谢！',
    'options': ['approve', 'reject', 'edit']
}
```

```python
# ═══ 第二步：人审批（假设批准）═══
result = app.invoke(Command(resume="approve"), config)
print(result["final_result"])
```

**模拟输出：**
```text
📧 邮件已发送！

张三你好，

诚挚邀请你参加本周五下午 2 点的项目会议。

谢谢！
```

### 3.3 如果人拒绝呢？

```python
# 重新运行，这次拒绝
config2 = {"configurable": {"thread_id": "email-002"}}
app.invoke({"request": "邀请张三参加周五的会议"}, config2)
result = app.invoke(Command(resume="reject"), config2)
print(result["final_result"])
```

**模拟输出：**
```text
❌ 邮件已被拒绝，未发送
```

---

## 四、修正型工作流：人修改 AI 的输出

> 场景：AI 写了文案，人可以修改后再继续。

### 4.1 完整代码

```python
from langgraph.types import interrupt, Command

class State(TypedDict):
    topic: str
    content: str
    feedback: str

def generate_content(state: State):
    """AI 生成内容"""
    response = model.invoke(f"写一段关于{state['topic']}的简介")
    return {"content": response.content}

def human_edit(state: State):
    """人修改内容"""
    # ★ interrupt：人可以传修改意见
    feedback = interrupt({
        "current_content": state["content"],
        "prompt": "请修改内容（或输入 ok 表示满意）",
    })

    if feedback == "ok":
        return {}   # 不修改
    else:
        # 用人的修改意见重新生成
        response = model.invoke(
            f"根据以下修改意见重写内容。\n"
            f"原文：{state['content']}\n"
            f"修改意见：{feedback}"
        )
        return {"content": response.content, "feedback": feedback}

# 运行
config = {"configurable": {"thread_id": "content-001"}}
app.invoke({"topic": "LangGraph"}, config)

# 人给出修改意见
result = app.invoke(
    Command(resume="把语气改得更活泼一些，加上 emoji"),
    config,
)
print(result["content"])
```

**模拟输出：**
```text
🌊 LangGraph 超酷的！它是 LangChain 生态里的低层级编排框架 🚀
用图的方式构建 AI Agent，支持循环、分支、记忆……
让你的 AI 应用又稳又灵活 💪
```

---

## 五、interrupt 的更多用法

### 5.1 传递复杂数据给人

```python
def complex_review(state: State):
    # interrupt 的值可以是任意类型
    decision = interrupt({
        "type": "multi_step_approval",
        "steps": [
            {"step": 1, "action": "创建数据库表", "risk": "low"},
            {"step": 2, "action": "导入数据", "risk": "medium"},
            {"step": 3, "action": "修改配置", "risk": "high"},
        ],
        "prompt": "请逐步审批以上操作",
    })
    return {"approved": decision}
```

### 5.2 多次 interrupt（多轮审批）

```python
def multi_step_approval(state: State):
    # 第一步审批
    step1 = interrupt("步骤 1/3：创建数据库表。批准？(yes/no)")
    if step1 != "yes":
        return {"status": "cancelled"}

    # 第二步审批
    step2 = interrupt("步骤 2/3：导入数据。批准？(yes/no)")
    if step2 != "yes":
        return {"status": "cancelled"}

    # 第三步审批
    step3 = interrupt("步骤 3/3：修改配置。批准？(yes/no)")
    return {"status": "all_approved" if step3 == "yes" else "cancelled"}
```

**运行：**
```python
config = {"configurable": {"thread_id": "multi-001"}}
app.invoke({}, config)

app.invoke(Command(resume="yes"), config)   # 批准步骤 1
app.invoke(Command(resume="yes"), config)   # 批准步骤 2
app.invoke(Command(resume="no"), config)    # 拒绝步骤 3
```

### 5.3 在条件路由中使用人工输入

```python
def human_gate(state: State):
    """人工门控：决定走哪条路"""
    choice = interrupt({
        "question": "请选择处理方式",
        "options": ["自动处理", "人工审核", "跳过"],
    })

    if choice == "自动处理":
        return {"route": "auto"}
    elif choice == "人工审核":
        return {"route": "manual"}
    else:
        return {"route": "skip"}

def route_after_gate(state: State):
    return state["route"]   # "auto" / "manual" / "skip"
```

---

## 六、interrupt 与检查点的关系

### 6.1 为什么 interrupt 必须有 checkpointer？

```text
interrupt 的本质：
  1. 工作流执行到 interrupt
  2. 把当前状态存到检查点     ← 需要 checkpointer！
  3. 工作流暂停
  4. 人处理完，调用 Command(resume=...)
  5. 从检查点恢复状态         ← 需要 checkpointer！
  6. 从 interrupt 处继续执行
```

> **没有 checkpointer，interrupt 无法工作**——因为状态存不下来，没法恢复。

```python
# ❌ 没有 checkpointer，interrupt 报错
app = graph.compile()
app.invoke({...})   # ValueError: interrupt requires a checkpointer

# ✅ 有 checkpointer，正常工作
checkpointer = InMemorySaver()
app = graph.compile(checkpointer=checkpointer)
```

### 6.2 interrupt 状态存在哪里？

```text
检查点数据：
  checkpoints 表 →  状态快照（包含 interrupt 的暂停状态）
  writes 表     →  每步写入记录

当调用 interrupt 时：
  1. 当前状态（含 interrupt 信息）被写入 checkpoints 表
  2. 当前进程可以安全退出（甚至重启）
  3. 调用 Command(resume=...) 时，从 checkpoints 表恢复
```

> **好处**：人工介入可以**跨进程**——服务器可以重启，审批的人可以明天再批。

---

## 七、常见问题

### ❌ interrupt 报错「需要 checkpointer」？

```python
# ❌ 编译时没传 checkpointer
app = graph.compile()
app.invoke({...})   # ValueError

# ✅ 编译时传 checkpointer
app = graph.compile(checkpointer=InMemorySaver())
```

### ❌ invoke 返回了 `__interrupt__` 而不是最终结果？

这是正常的！说明工作流在 `interrupt` 处暂停了：

```python
result = app.invoke({...}, config)

if "__interrupt__" in result:
    # 工作流暂停了，等待人工输入
    prompt = result["__interrupt__"][0].value
    print(f"需要人工介入：{prompt}")

    # 人工处理后恢复
    result = app.invoke(Command(resume="yes"), config)
```

### ❌ 恢复后节点从头执行了？

不会。LangGraph 从 `interrupt()` 的**返回处**继续，不会重复执行前面的代码：

```python
def my_node(state):
    print("步骤 1")              # ← 第一次运行会执行
    result = interrupt("确认？")  # ← 在这里暂停
    print("步骤 2")              # ← 恢复后从这里继续，不会重复执行步骤 1
    return {"result": result}
```

### ❌ 如何查看当前是否有 interrupt 在等待？

```python
state = app.get_state(config)
if state.next:
    # 有节点等待执行（可能在 interrupt 处暂停）
    print(f"工作流暂停在: {state.next}")
```

---

## 八、速查表

### 核心 API

| 操作 | 代码 |
|------|------|
| 暂停工作流 | `result = interrupt("提示信息")` |
| 恢复执行 | `app.invoke(Command(resume="值"), config)` |
| 查看 interrupt 信息 | `result["__interrupt__"][0].value` |
| 判断是否暂停 | `"__interrupt__" in result` |

### 恢复时的返回值

| 场景 | 代码 |
|------|------|
| 简单审批 | `Command(resume="yes")` 或 `Command(resume="no")` |
| 传递修改意见 | `Command(resume="把语气改正式一点")` |
| 传递复杂数据 | `Command(resume={"action": "edit", "content": "..."})` |

### 注意事项

| 要点 | 说明 |
|------|------|
| **必须有 checkpointer** | `interrupt` 依赖检查点存状态 |
| **必须传 thread_id** | 恢复时用同一个 `thread_id` |
| **`__interrupt__` 是正常的** | 不是错误，是「等待人工输入」的信号 |
| **支持跨进程** | 服务器重启后也能恢复 |

---

## 九、深入理解：人工介入的本质

### 9.1 人机分工

```text
AI 的职责：
  ✅ 生成草稿、分析数据、准备操作
  ✅ 做初步判断
  ❌ 做最终决定（高风险场景）
  ❌ 执行不可逆操作（未经批准）

人类的职责：
  ✅ 审批、批准、拒绝
  ✅ 修改、纠正、补充
  ✅ 做最终决定
  ✅ 对结果负责

LangGraph 的职责：
  ✅ 在正确的位置暂停
  ✅ 安全地保存状态
  ✅ 无缝恢复执行
  ✅ 连接 AI 和人类
```

### 9.2 类比：公司报销流程

```text
员工（AI）：
  填好报销单，附上发票

部门经理（人工介入）：
  看一眼，批准或退回修改

财务（工具/执行）：
  经理批准后才打款

没有审批会怎样？
  员工随便填个数字就打款了 → 💥 财务灾难
```

---

## 下一步

- [[功能-检查点-Checkpoint]] — interrupt 依赖的持久化机制
- [[功能-工具调用-Tool-Calling]] — 工具调用 + 人工审批的组合
- [[实战-自定义Agent]] — 构建带人工介入的完整 Agent
