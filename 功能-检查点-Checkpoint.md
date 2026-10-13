---
aliases: [检查点, Checkpoint, 持久化, 持久化存储, InMemorySaver, SqliteSaver, PostgresSaver]
tags: [功能, 基础, Checkpoint, 持久化]
---

# 💾 检查点（Checkpoint）

> 程序一关，对话全没了？——那就需要**检查点**。
> 检查点 = 给 Agent 装上「存档功能」，随时保存、随时恢复、随时回溯。
> 本文从零讲清 LangGraph 的检查点机制，完全不需要前置知识。

---

## 一、检查点是什么？

### 1.1 一句话理解

> **检查点 = 游戏存档。** 每执行一步自动存一次档，关了游戏再开，进度还在。

```text
❌ 没有检查点：

  用户和 AI 聊了 10 轮 → 程序崩溃/重启 → 💥 全没了，从头再来


✅ 有检查点：

  用户和 AI 聊了 10 轮 → 每步自动存档 → 程序崩溃/重启 → 📂 从第 10 轮继续
```

### 1.2 检查点保存了什么？

| 保存内容 | 说明 |
|---------|------|
| **State（状态）** | 所有字段的当前值（对话历史、中间结果等） |
| **每一步的更新记录** | 哪个节点更新了哪些字段（writes） |
| **执行元数据** | 时间戳、步骤号、checkpoint_id 等 |
| **线程隔离** | 每个 `thread_id` 的存档互不干扰 |

### 1.3 为什么需要检查点？

| 场景 | 没检查点 | 有检查点 |
|------|---------|---------|
| 程序重启 | 💥 对话丢失 | ✅ 无缝继续 |
| 多轮对话 | ❌ 自己手动维护历史 | ✅ 自动加载历史 |
| 崩溃恢复 | ❌ 从头开始 | ✅ 回到崩溃前 |
| 时间旅行 | ❌ 不可能 | ✅ 回到任意一步 |
| 人工审批 | ❌ 做不到 | ✅ interrupt 后恢复 |

---

## 二、三种 Checkpointer

> LangGraph 提供了三种检查点存储，按**数据存到哪里**区分。

### 2.1 三种对比

| | `InMemorySaver` | `SqliteSaver` | `PostgresSaver` |
|---|:---:|:---:|:---:|
| **存储位置** | 内存 | SQLite 文件 | PostgreSQL 数据库 |
| **程序重启后** | 💥 丢失 | ✅ 保留 | ✅ 保留 |
| **需要安装** | 否 | `langgraph-checkpoint-sqlite` | `langgraph-checkpoint-postgres` |
| **需要 setup()** | 否 | 否 | ✅ 是 |
| **适合场景** | 开发测试 | 小型应用、单机 | 生产环境、多用户 |
| **并发支持** | ❌ | 一般 | ✅ 强 |

### 2.2 InMemorySaver——内存存档

> 最简单，数据只存在内存里。**适合开发调试**，程序一关就没了。

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()   # 就这么简单，无需安装额外包
```

### 2.3 SqliteSaver——文件存档

> 数据存到 SQLite 文件，程序重启不丢。**适合小型应用**。

```bash
pip install langgraph-checkpoint-sqlite
```

```python
from langgraph.checkpoint.sqlite import SqliteSaver

# 方式一：存到文件
checkpointer = SqliteSaver.from_conn_string("./chat_history.db")

# 方式二：直接用连接
import sqlite3
conn = sqlite3.connect("./chat_history.db", check_same_thread=False)
checkpointer = SqliteSaver(conn)
```

### 2.4 PostgresSaver——数据库存档

> 数据存到 PostgreSQL。**适合生产环境**，支持高并发。

```bash
pip install langgraph-checkpoint-postgres
```

```python
from langgraph.checkpoint.postgres import PostgresSaver

# ★ 必须调用 setup()！Postgres 不会自动建表
checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:password@localhost:5432/mydb"
)
checkpointer.setup()   # 第一次运行时建表，之后不会重复建
```

> **⚠️ 重要**：`PostgresSaver` **必须**调用 `setup()`，否则运行时报错（找不到表）。
> `InMemorySaver` 和 `SqliteSaver` **不需要**。

### 2.5 setup() 建的表能自定义吗？

**不能。** 表结构是框架固定的，不能改列名、类型或增删字段。

```text
checkpoints 表（框架固定，不可修改）：
  thread_id       TEXT        ← 不能改名、不能删
  checkpoint_id   TEXT
  parent_id       TEXT
  checkpoint      BYTEA
  metadata        JSONB
  created_at      TIMESTAMPTZ

writes 表（框架固定，不可修改）：
  thread_id       TEXT
  checkpoint_id   TEXT
  task_id         TEXT
  channel         TEXT
  value           BYTEA
```

> 框架内部靠这些字段的精确结构来读写数据，改了就崩。

**可以做的事：**

```python
# ✅ 可以指定 schema（命名空间），避免表名冲突
checkpointer = PostgresSaver.from_conn_string(
    "postgresql://...",
    schema="langgraph"   # 表建在 langgraph.checkpoints / langgraph.writes
)
checkpointer.setup()
```

> 如果真的需要自定义存储逻辑（比如存到 Redis、MongoDB），可以实现 `BaseCheckpointSaver` 接口，但 99% 的场景不需要。

---

## 三、使用检查点的完整流程

### 3.1 三步走

```text
第 1 步：创建 checkpointer（选一种存储）
        ↓
第 2 步：graph.compile(checkpointer=checkpointer)  ← 编译时传入
        ↓
第 3 步：invoke/stream 时传 config={"configurable": {"thread_id": "..."}}
```

### 3.2 完整示例

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from langchain_openai import ChatOpenAI

# 1. 状态
class State(TypedDict):
    messages: list

# 2. 模型
model = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx",
)

# 3. 节点
def chatbot(state: State):
    response = model.invoke(state["messages"])
    return {"messages": state["messages"] + [response]}

# 4. 构建图
graph = StateGraph(State)
graph.add_node("chatbot", chatbot)
graph.add_edge(START, "chatbot")
graph.add_edge("chatbot", END)

# ★ 5. 创建 checkpointer
checkpointer = InMemorySaver()

# ★ 6. 编译时传入 checkpointer
app = graph.compile(checkpointer=checkpointer)

# ★ 7. 运行时传入 thread_id
config = {"configurable": {"thread_id": "对话-001"}}

result = app.invoke(
    {"messages": [{"role": "user", "content": "你好"}]},
    config=config,
)
print(result["messages"][-1].content)
```

> **注意**：一旦 `compile(checkpointer=...)` 了，**每次 invoke/stream 都必须传 `thread_id`**，否则报错。

### 3.3 多轮对话：自动加载历史

```python
# 第一轮
config = {"configurable": {"thread_id": "对话-001"}}
result = app.invoke(
    {"messages": [{"role": "user", "content": "我叫小明"}]},
    config=config,
)

# 第二轮——不用手动传历史！检查点自动加载
result = app.invoke(
    {"messages": [{"role": "user", "content": "我叫什么名字？"}]},
    config=config,    # ← 同一个 thread_id
)
print(result["messages"][-1].content)   # → "你叫小明。"
```

```text
第一轮 invoke:
  输入: messages = [{"role": "user", "content": "我叫小明"}]
  检查点: 自动加载（空的，第一次）
  执行后: 自动存档 → messages = [Human("我叫小明"), AI("好的，小明！")]

第二轮 invoke:
  输入: messages = [{"role": "user", "content": "我叫什么名字？"}]
  检查点: 自动加载 → [Human("我叫小明"), AI("好的，小明！")]  ← 关键！
  执行后: 自动存档 → messages = [..., Human("我叫什么名字？"), AI("你叫小明")]
```

> **核心价值**：你不需要自己维护对话历史！检查点帮你**自动存、自动取**。

---

## 四、thread_id 与检查点的关系

### 4.1 一个 thread_id = 一份存档

```text
thread_id "对话-001"  →  存档 1（小明的对话历史）
thread_id "对话-002"  →  存档 2（小红的对话历史）
thread_id "对话-003"  →  存档 3（管理员的对话历史）
```

> 每个 `thread_id` 的存档**完全隔离**，互不影响。

### 4.2 同一个 thread_id 继续对话

```python
config = {"configurable": {"thread_id": "对话-001"}}

# 第一轮
app.invoke({"messages": [{"role": "user", "content": "你好"}]}, config)

# 第二轮（自动衔接第一轮的历史）
app.invoke({"messages": [{"role": "user", "content": "继续聊"}]}, config)
```

### 4.3 不同 thread_id = 新对话

```python
# 全新对话，不看旧历史
config_new = {"configurable": {"thread_id": "对话-002"}}
app.invoke({"messages": [{"role": "user", "content": "你好"}]}, config_new)
```

### 4.4 不传 thread_id 会怎样？

```python
# ❌ 编译时用了 checkpointer，但不传 thread_id → 报错！
app.invoke({"messages": [...]})
# ValueError: checkpoint.postgres must be called with a checkpointer with a thread_id

# ✅ 传了 thread_id → 正常运行
app.invoke({"messages": [...]}, config={"configurable": {"thread_id": "对话-001"}})
```

### 4.5 用户和 thread_id 的关系：需要自己维护

> **LangGraph 只管「存档」，不管「谁的存档」。** 检查点里没有 `user_id` 字段，用户和 thread_id 的对应关系需要你在**应用层**自己维护。

```text
┌─────────────────────────────────────────────────┐
│  你的应用数据库（自己建的表）                      │
│                                                  │
│  user_threads 表                                 │
│  ┌──────────┬──────────────┬───────────────┐     │
│  │ user_id  │ thread_id    │ created_at    │     │
│  ├──────────┼──────────────┼───────────────┤     │
│  │ 小明     │ thread-001   │ 2026-10-13    │     │
│  │ 小明     │ thread-002   │ 2026-10-14    │     │
│  │ 小红     │ thread-003   │ 2026-10-13    │     │
│  └──────────┴──────────────┴───────────────┘     │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  LangGraph 的检查点（框架自动维护）               │
│                                                  │
│  checkpoints 表                                  │
│  ┌──────────────┬──────────────────┐             │
│  │ thread_id    │ checkpoint       │             │
│  ├──────────────┼──────────────────┤             │
│  │ thread-001   │ {messages: [...]}│             │
│  │ thread-002   │ {messages: [...]}│             │
│  │ thread-003   │ {messages: [...]}│             │
│  └──────────────┴──────────────────┘             │
│                                                  │
│  注意：没有 user_id 字段！                        │
└─────────────────────────────────────────────────┘
```

**典型做法：应用层维护映射关系**

```python
import uuid

def start_new_chat(user_id: str):
    """用户开始新对话"""
    thread_id = str(uuid.uuid4())

    # 存到你自己的数据库
    db.execute(
        "INSERT INTO user_threads (user_id, thread_id) VALUES (?, ?)",
        (user_id, thread_id)
    )

    config = {"configurable": {"thread_id": thread_id}}
    return config

def get_user_threads(user_id: str):
    """查看某个用户的所有对话"""
    return db.execute(
        "SELECT thread_id, created_at FROM user_threads WHERE user_id = ?",
        (user_id,)
    ).fetchall()

def continue_chat(user_id: str, thread_id: str):
    """继续某个对话（带权限检查）"""
    # 验证这个 thread_id 是否属于这个用户
    owner = db.execute(
        "SELECT user_id FROM user_threads WHERE thread_id = ?",
        (thread_id,)
    ).fetchone()

    if owner["user_id"] != user_id:
        raise PermissionError("这不是你的对话！")

    config = {"configurable": {"thread_id": thread_id}}
    return config
```

> **一句话**：你负责「谁的对话」，LangGraph 负责「对话内容」。各管各的。

---

## 五、查看和管理检查点

### 5.1 获取当前状态

```python
# 获取某个 thread 的当前状态
state = app.get_state(config)
print(state.values)    # → {'messages': [...]}
print(state.next)      # → ()  （下一步要执行的节点）
print(state.metadata)  # → {'source': 'loop', 'step': 3, ...}
```

### 5.2 获取状态历史（时间旅行）

```python
# 查看某个 thread 的所有历史快照
for checkpoint in app.get_state_history(config):
    print(f"步骤 {checkpoint.metadata['step']}: {checkpoint.values}")
```

**模拟输出：**
```text
步骤 2: {'messages': [Human('你好'), AI('你好！'), Human('我叫小明'), AI('好的小明！')]}
步骤 1: {'messages': [Human('你好'), AI('你好！')]}
步骤 0: {'messages': [Human('你好')]}
```

### 5.3 回到某个历史状态（时间旅行）

```python
# 1. 先获取历史快照
history = list(app.get_state_history(config))

# 2. 选一个旧的状态
old_state = history[-1]   # 最早的那一步

# 3. 从那个状态继续执行
result = app.invoke(None, config=old_state.config)   # 传 None = 从那个状态继续
```

```text
时间旅行示意：

  步骤0 ──→ 步骤1 ──→ 步骤2 ──→ 步骤3（当前）
                   ↑
                   └── 回到这里，重新走一遍
```

### 5.4 更新检查点状态

```python
# 直接修改某个 thread 的状态（谨慎使用！）
app.update_state(
    config,
    {"messages": [{"role": "user", "content": "我改主意了"}]}
)
```

---

## 六、检查点在数据库中的存储结构

> 详见 [[概念-消息与记忆]] 的数据库表结构章节，这里只做简要说明。

### 6.1 两张核心表

```text
┌─────────────────────────────────────────────────┐
│  checkpoints 表（检查点快照）                     │
│                                                  │
│  thread_id      → 哪个对话                        │
│  checkpoint_id  → 快照唯一 ID                     │
│  parent_id      → 上一个快照（形成链条）           │
│  checkpoint     → 状态数据（JSON/二进制）          │
│  metadata       → 步骤号、来源等                   │
│  created_at     → 创建时间                        │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  writes 表（每步的写入记录）                      │
│                                                  │
│  thread_id      → 哪个对话                        │
│  checkpoint_id  → 属于哪个快照                     │
│  task_id        → 哪个节点                        │
│  channel        → 更新哪个状态字段                 │
│  value          → 写入的值                        │
└─────────────────────────────────────────────────┘
```

### 6.2 AI 不会直接操作数据库

```text
用户提问 → LangGraph 框架 → 自动读写数据库 → 返回结果
                ↑
        你的代码只写逻辑，不写 SQL
        框架帮你处理所有数据库操作
```

> 你只管 `compile(checkpointer=...)`，框架自动帮你建表、读写。**不需要手写 SQL**。

---

## 七、实战：带记忆的聊天机器人

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph.message import add_messages
from typing import Annotated
from langchain_openai import ChatOpenAI
from langchain.messages import HumanMessage

# ═══ 1. 状态（用 add_messages 自动管理历史）═══
class State(TypedDict):
    messages: Annotated[list, add_messages]

# ═══ 2. 模型 ═══
model = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx",
)

# ═══ 3. 节点 ═══
def chatbot(state: State):
    response = model.invoke(state["messages"])
    return {"messages": [response]}   # add_messages 自动追加

# ═══ 4. 构建图 ═══
graph = StateGraph(State)
graph.add_node("chatbot", chatbot)
graph.add_edge(START, "chatbot")
graph.add_edge("chatbot", END)

# ═══ 5. 编译（带检查点）═══
checkpointer = InMemorySaver()
app = graph.compile(checkpointer=checkpointer)

# ═══ 6. 多轮对话 ═══
config = {"configurable": {"thread_id": "user-42"}}

# 第一轮
result = app.invoke({"messages": [HumanMessage(content="我叫小明")]}, config)
print("AI:", result["messages"][-1].content)

# 第二轮（AI 记得名字）
result = app.invoke({"messages": [HumanMessage(content="我叫什么？")]}, config)
print("AI:", result["messages"][-1].content)

# 第三轮（AI 记得全部历史）
result = app.invoke({"messages": [HumanMessage(content="我们刚才聊了什么？")]}, config)
print("AI:", result["messages"][-1].content)
```

**模拟输出：**
```text
AI: 你好小明！很高兴认识你。
AI: 你叫小明。
AI: 你先做了自我介绍（叫小明），然后问了我你的名字。
```

---

## 八、常见问题

### ❌ 编译时传了 checkpointer，但 invoke 报错

```python
# ❌ 忘了传 thread_id
app.invoke({"messages": [...]})
# ValueError: ... must be called with a checkpointer with a thread_id

# ✅ 必须传 config
app.invoke({"messages": [...]}, config={"configurable": {"thread_id": "对话-001"}})
```

### ❌ 程序重启后对话丢了？

检查是否用了 `InMemorySaver`（内存存储）：

```python
# ❌ 内存存储，重启就没了
checkpointer = InMemorySaver()

# ✅ 文件存储，重启不丢
checkpointer = SqliteSaver.from_conn_string("./chat.db")

# ✅ 数据库存储，重启不丢
checkpointer = PostgresSaver.from_conn_string("postgresql://...")
checkpointer.setup()
```

### ❌ PostgresSaver 报「表不存在」？

忘了调用 `setup()`：

```python
checkpointer = PostgresSaver.from_conn_string("...")
checkpointer.setup()   # ★ 必须调用！第一次运行时建表
```

### ❌ 多个用户对话互相串了？

每个用户应该用**不同的 thread_id**：

```python
# ❌ 所有人共用一个 thread_id
config = {"configurable": {"thread_id": "shared"}}

# ✅ 每人一个 thread_id
config = {"configurable": {"thread_id": f"user-{user_id}"}}
```

### ❌ 想删除某个对话的存档？

```python
# 方法一：用 SqliteSaver 时，直接删数据库文件
# 方法二：PostgresSaver 时，执行 SQL（或调用 delete_thread）
# 方法三：换一个新的 thread_id（旧的就没人用了）
```

---

## 九、Checkpointer 选型指南

```text
你的使用场景是什么？
│
├─ 「开发调试、写 Demo」
│   └─→ InMemorySaver          最简单，一行搞定
│
├─ 「小项目、单机、数据量不大」
│   └─→ SqliteSaver            文件存储，免安装数据库
│
├─ 「生产环境、多用户、高并发」
│   └─→ PostgresSaver          专业数据库，必须 setup()
│
└─ 「不确定」
    └─→ 先用 InMemorySaver，上线前换 SqliteSaver
```

---

## 十、速查表

### 创建 Checkpointer

| 存储 | 代码 |
|------|------|
| 内存 | `InMemorySaver()` |
| SQLite | `SqliteSaver.from_conn_string("./chat.db")` |
| PostgreSQL | `PostgresSaver.from_conn_string("postgresql://...")` + `.setup()` |

### 使用流程

| 步骤 | 代码 |
|------|------|
| 编译时传入 | `app = graph.compile(checkpointer=checkpointer)` |
| 运行时传 thread_id | `config={"configurable": {"thread_id": "对话-001"}}` |
| 获取当前状态 | `app.get_state(config)` |
| 获取状态历史 | `app.get_state_history(config)` |
| 回到历史状态 | `app.invoke(None, config=old_state.config)` |
| 手动更新状态 | `app.update_state(config, new_values)` |

### 常用配置

| 配置 | 代码 |
|------|------|
| 唯一会话标识 | `config={"configurable": {"thread_id": "..."}}` |
| 限制最大步数 | `config={"recursion_limit": 10}` |
| 两者都有 | `config={"configurable": {"thread_id": "..."}, "recursion_limit": 10}` |

---

## 十一、深入理解：检查点的本质

### 11.1 检查点 = 状态的快照链

```text
thread_id "对话-001" 的检查点链：

  checkpoint_id: cp_001  ←  初始状态（用户问了第一个问题）
       │
       ▼
  checkpoint_id: cp_002  ←  chatbot 节点执行后
       │
       ▼
  checkpoint_id: cp_003  ←  第二轮对话后
       │
       ▼
  checkpoint_id: cp_004  ←  当前状态

  每个 checkpoint 都有 parent_id 指向前一个 → 形成链条
```

> **时间旅行**的本质：沿着链条回退到任意节点，然后从那里重新走。

### 11.2 类比：Git 版本控制

| Git | 检查点 |
|-----|--------|
| commit | checkpoint |
| HEAD | 当前状态 |
| git log | get_state_history |
| git checkout <commit> | 时间旅行（回到旧状态） |
| branch | 不同的 thread_id |
| .git 目录 | 数据库（checkpoints 表） |

```text
Git:           代码的版本管理
检查点:        Agent 状态的版本管理

Git commit:    保存代码快照
checkpoint:    保存状态快照

git checkout:  回到旧版本代码
时间旅行:      回到旧状态继续跑
```

---

## 下一步

- [[功能-人工介入-Human-in-the-Loop]] — interrupt 中断与人工审批（依赖检查点）
- [[概念-消息与记忆]] — thread_id、数据库表结构详解
- [[实战-React-Agent]] — 构建带记忆的 Agent
