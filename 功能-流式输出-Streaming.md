---
aliases: [流式输出, Streaming, stream, astream, 流式]
tags: [功能, 基础, Streaming, 流式]
---

# 🌊 流式输出（Streaming）

> 你用 ChatGPT 时看到字一个一个蹦出来——那就是**流式输出**。
> 如果用 `invoke`，你要干等全部生成完才能看到结果；用 `stream`，**边生成边看到**。
> 本文从零讲清 LangGraph 的流式输出，完全不需要前置知识。

---

## 一、为什么需要流式输出？

### 1.1 两种体验的对比

```text
❌ invoke（非流式）：

   用户: "帮我写一首关于春天的诗"
   ⏳ 等待中...（10 秒，屏幕一片空白，用户以为卡了）
   💬 "春风拂面花满枝，细雨润泥燕归时……"（整段一次性出现）


✅ stream（流式）：

   用户: "帮我写一首关于春天的诗"
   💬 "春"
   💬 "春风"
   💬 "春风拂面"
   💬 "春风拂面花满枝，"
   💬 "春风拂面花满枝，细雨润泥燕归时……"（边生成边显示）
```

### 1.2 流式输出的好处

| 好处 | 说明 |
|------|------|
| **用户体验好** | 立刻看到输出，不用干等 |
| **感知速度快** | 虽然总时间一样，但「看起来快很多」 |
| **可以中途干预** | 看到方向不对可以随时打断 |
| **适合长文本** | 写长文章时优势尤其明显 |
| **实时反馈** | 聊天机器人、写作助手的标准体验 |

> **一句话**：`invoke` = 等整锅饭做好再吃；`stream` = 边煮边吃。

---

## 二、最基本的用法：invoke vs stream

### 2.1 invoke：一次性拿结果

```python
from langchain_openai import ChatOpenAI

model = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx",
)

result = model.invoke("用一句话介绍春天")
print(result.content)   # 全部生成完才打印
```

**模拟输出：**
```text
春天是万物复苏的季节，大地回春，百花盛开。
```

### 2.2 stream：逐块拿结果

```python
# 把 invoke 换成 stream
for chunk in model.stream("用一句话介绍春天"):
    print(chunk.content, end="", flush=True)
```

**模拟输出（一个字一个字蹦出来）：**
```text
春天是万物复苏的季节，大地回春，百花盛开。
```

> 看起来一样？因为模拟输出把过程压缩了。实际运行时你会看到字**一个一个**蹦出来。

### 2.3 看看 chunk 里到底是什么

```python
for chunk in model.stream("你好"):
    print(repr(chunk))
```

**模拟输出：**
```text
AIMessageChunk(content='你')
AIMessageChunk(content='好')
AIMessageChunk(content='！')
AIMessageChunk(content='有什么')
AIMessageChunk(content='可以帮你的')
AIMessageChunk(content='？')
```

> **关键理解**：`stream` 返回的不是完整消息，而是很多个**小碎片（chunk）**。
> 每个 chunk 是一个 `AIMessageChunk` 对象，把它们拼起来才是完整回答。

```text
chunk 1: "春"     ─┐
chunk 2: "风"      │
chunk 3: "拂"      ├──→  拼接后："春风拂面"
chunk 4: "面"     ─┘
```

### 2.4 invoke vs stream 对比

| | `invoke` | `stream` |
|---|:---:|:---:|
| **返回** | 完整结果（1 个对象） | 多个 chunk（生成器） |
| **等待** | 全部完成才返回 | 边生成边返回 |
| **用户体验** | 差（干等） | 好（实时） |
| **代码复杂度** | 简单 | 稍复杂（要循环） |
| **适用场景** | 后台任务、脚本 | 聊天界面、写作工具 |

---

## 三、在 LangGraph 中使用 stream

### 3.1 最简单的例子

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI

# 定义状态
class State(TypedDict):
    question: str
    answer: str

# 创建模型
model = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx",
)

# 定义节点
def chatbot(state: State):
    response = model.invoke(state["question"])
    return {"answer": response.content}

# 构建图
graph = StateGraph(State)
graph.add_node("chatbot", chatbot)
graph.add_edge(START, "chatbot")
graph.add_edge("chatbot", END)
app = graph.compile()

# ★ 用 stream 运行
for chunk in app.stream({"question": "用一句话介绍春天"}):
    print(chunk)
```

**模拟输出：**
```text
{'chatbot': {'answer': '春天是万物复苏的季节，大地回春，百花盛开。'}}
```

> 注意：这里**看起来**和 invoke 差不多，因为节点内部用的是 `model.invoke`（一次性）。
> 真正的「字级流式」需要额外配置——见 §五。

### 3.2 stream 的输出结构

`app.stream()` 每次 yield 一个 **dict**，格式是：

```text
{ "节点名": { "状态字段": 值 } }
```

```python
# 例：图有两个节点
for chunk in app.stream(input_data):
    print(chunk)
```

**模拟输出：**
```text
{'search': {'documents': ['文档1', '文档2']}}      ← search 节点完成
{'answer': {'answer': '根据文档...'}}               ← answer 节点完成
```

> 每个 chunk 表示**某个节点刚执行完，更新了状态**。

---

## 四、stream 的五种模式（stream_mode）

> **这是流式输出最核心的配置！** 不同模式决定「流出来的是什么东西」。

### 4.1 五种模式总览

| 模式 | 流出什么 | 适合场景 |
|------|---------|---------|
| `"values"` | 每步之后的**完整状态** | 调试、看整体进展 |
| `"updates"` | 每步**更新了哪些字段** | **默认**，看节点输出 |
| `"messages"` | **LLM 生成的每个 token** | **聊天界面、打字机效果** |
| `"custom"` | 节点里手动 `stream_writer` 的内容 | 自定义进度提示 |
| `"debug"` | 详细的调试信息 | 排查问题 |

```python
# 指定模式
for chunk in app.stream(input_data, stream_mode="messages"):
    ...
```

> 可以**同时用多种模式**：`stream_mode=["updates", "messages"]`。

---

### 4.2 模式一：updates（默认）

> 流出**每个节点的状态更新**——「谁改了什么」。

```python
for chunk in app.stream(
    {"question": "你好"},
    stream_mode="updates"
):
    print(chunk)
```

**模拟输出：**
```text
{'chatbot': {'answer': '你好！有什么可以帮你的？'}}
```

**多节点的情况：**
```text
{'search':  {'documents': ['文档A', '文档B']}}
{'answer':  {'answer': '根据检索结果...'}}
{'cleanup': {}}                    ← 节点执行了但没更新字段
```

**适用场景**：想知道每个节点干了什么、更新了哪些字段。

---

### 4.3 模式二：values

> 流出**每步之后的完整状态**——「现在整个状态长什么样」。

```python
for chunk in app.stream(
    {"question": "你好", "answer": ""},
    stream_mode="values"
):
    print(chunk)
```

**模拟输出：**
```text
{'question': '你好', 'answer': ''}                              ← 初始状态
{'question': '你好', 'answer': '你好！有什么可以帮你的？'}       ← chatbot 节点执行后
```

**和 updates 的区别：**
```text
updates: 只告诉你「改了什么」   → {'chatbot': {'answer': '你好！'}}
values:  给你看「改完什么样」   → {'question': '你好', 'answer': '你好！'}
```

| | `updates` | `values` |
|---|:---:|:---:|
| **内容** | 只有更新的字段 | 整个状态 |
| **体积** | 小 | 大（状态越大越明显） |
| **用途** | 看节点输出 | 看整体进展 |

**适用场景**：调试时想看状态的完整变化过程。

---

### 4.4 模式三：messages（⭐ 最常用）

> 流出 **LLM 生成的每个 token**——这就是「打字机效果」！

```python
for token, metadata in app.stream(
    {"question": "用一句话介绍春天"},
    stream_mode="messages"
):
    if token.content:
        print(token.content, end="", flush=True)
```

**模拟输出（一个字一个字蹦出来）：**
```text
春天是万物复苏的季节，大地回春，百花盛开。
```

#### messages 模式的返回值

`stream_mode="messages"` 返回的是**元组**：

```python
for token, metadata in app.stream(input, stream_mode="messages"):
    # token    → AIMessageChunk 对象（包含当前这一小块文本）
    # metadata → 元数据（哪个节点、哪个模型生成的）
    print(token.content, end="")
```

```text
token:    AIMessageChunk(content='春')        ← 这一块文本
metadata: {'langgraph_node': 'chatbot',        ← 哪个节点
           'langgraph_step': 1,               ← 第几步
           'ls_provider': 'openai',           ← 模型提供方
           ...}
```

#### 只想看文本？用 token.content

```python
for token, metadata in app.stream(input, stream_mode="messages"):
    if token.type == "AIMessageChunk":   # 只要 AI 生成的内容
        print(token.content, end="", flush=True)
```

**适用场景**：聊天界面、写作助手——所有需要「打字机效果」的地方。

---

### 4.5 模式四：custom

> 节点里**手动推送**自定义内容——比如进度提示、中间结果。

```python
from langgraph.types import StreamWriter

def long_task(state: State, writer: StreamWriter):
    writer({"progress": "正在搜索..."})       # ★ 手动推送
    # ... 耗时操作 ...
    writer({"progress": "正在分析..."})
    # ...
    writer({"progress": "完成！"})
    return {"answer": "最终结果"}

# 运行
for chunk in app.stream(input, stream_mode="custom"):
    print(chunk)
```

**模拟输出：**
```text
{'progress': '正在搜索...'}
{'progress': '正在分析...'}
{'progress': '完成！'}
```

> **注意**：需要在节点签名中加入 `writer: StreamWriter` 参数。

**适用场景**：长时间任务的进度提示（「正在搜索…」「正在生成…」）。

---

### 4.6 模式五：debug

> 流出**最详细的调试信息**——每一步的输入、输出、耗时。

```python
for chunk in app.stream(input, stream_mode="debug"):
    print(chunk)
```

**模拟输出（简化）：**
```text
{'type': 'task', 'timestamp': '2026-10-13T10:00:00',
 'step': 1, 'payload': {'id': '...', 'name': 'chatbot',
 'input': {'question': '你好'}, 'result': {'answer': '...'}}}

{'type': 'task_result', 'timestamp': '2026-10-13T10:00:02',
 'step': 1, 'payload': {'id': '...', 'result': {'answer': '你好！'}}}
```

**适用场景**：排查问题、分析执行流程。

---

### 4.7 同时用多种模式

```python
for chunk in app.stream(input, stream_mode=["updates", "messages"]):
    print(chunk)
```

**模拟输出：**
```text
('messages', (AIMessageChunk(content='你'), {...}))
('messages', (AIMessageChunk(content='好'), {...}))
('updates', {'chatbot': {'answer': '你好！'}})
```

> 每个 chunk 变成 `(模式名, 数据)` 的元组。

---

## 五、⭐ 字级流式输出完整实战

> 这是**最常用**的场景：LLM 的 token 一个一个流出来。

### 5.1 基础版：简单聊天

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI

# 1. 状态
class State(TypedDict):
    question: str
    answer: str

# 2. 模型
model = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx",
)

# 3. 节点（用 stream 而不是 invoke）
def chatbot(state: State):
    # ★ 关键：节点内部用 stream 生成器
    for chunk in model.stream(state["question"]):
        yield {"answer": chunk.content}   # 每个 chunk 都写入状态

# 4. 构建图
graph = StateGraph(State)
graph.add_node("chatbot", chatbot)      # 注意：generator 节点
graph.add_edge(START, "chatbot")
graph.add_edge("chatbot", END)
app = graph.compile()

# 5. 运行
for chunk in app.stream({"question": "用一句话介绍春天"}):
    print(chunk)
```

**模拟输出：**
```text
{'chatbot': {'answer': '春'}}
{'chatbot': {'answer': '风'}}
{'chatbot': {'answer': '拂'}}
{'chatbot': {'answer': '面'}}
...
```

> **关键变化**：节点从普通函数变成了**生成器（yield）**！
> 每 yield 一次，`app.stream()` 就吐一个 chunk。

### 5.2 实用版：带打字机效果的聊天

```python
# 同上图构建代码...

print("🤖 AI 助手（输入 quit 退出）")
print("-" * 40)

while True:
    question = input("\n👤 你: ")
    if question.lower() == "quit":
        break

    print("🤖 AI: ", end="", flush=True)

    for chunk in app.stream(
        {"question": question},
        stream_mode="messages"      # ★ 用 messages 模式
    ):
        token, metadata = chunk
        if token.content:
            print(token.content, end="", flush=True)   # 逐字打印

    print()  # 换行
```

**模拟运行效果：**
```text
🤖 AI 助手（输入 quit 退出）
----------------------------------------

👤 你: 用一句话介绍春天
🤖 AI: 春天是万物复苏的季节，大地回春，百花盛开。

👤 你: 再写一首诗
🤖 AI: 春风拂面花满枝，细雨润泥燕归时。

👤 你: quit
```

### 5.3 带工具调用的流式

```python
from langgraph.prebuilt import create_react_agent
from langchain.tools import tool

@tool
def get_weather(city: str) -> str:
    """获取城市天气。"""
    return f"{city}: 晴天 25°C"

@tool
def add(a: int, b: int) -> int:
    """将两个数字相加。"""
    return a + b

agent = create_react_agent(model, tools=[get_weather, add])

# ★ 流式运行 Agent
for token, metadata in agent.stream(
    {"messages": [{"role": "user", "content": "北京天气怎么样？"}]},
    stream_mode="messages"
):
    if token.content:
        print(token.content, end="", flush=True)
```

**模拟输出：**
```text
（AI 先调用工具，不输出文字）
北京今天晴天，气温 25°C。                    ← 工具结果返回后，流式输出
```

> **注意**：工具调用过程中不会有 token 输出（工具不是 LLM），工具执行完后 LLM 的回答才是流式的。

---

## 六、异步流式：astream

### 6.0 什么时候需要用异步？

> **简单判断**：写脚本自己用 → `stream` 就够了；**写服务给别人用** → 用 `astream`。

**同步 stream 的问题——线程被白白占着：**

```text
假设 3 个用户同时发起聊天（同步 stream，阻塞）：

  用户A 请求 → ⏳ 处理中（独占线程，等 LLM 吐字）
  用户B 请求 → ⛔ 排队...
  用户C 请求 → ⛔ 排队...

  3 个用户 = 3 个线程，都在「干等」，服务器资源浪费严重
```

**异步 astream 的优势——等待时让出控制权：**

```text
3 个用户同时发起聊天（异步 astream，非阻塞）：

  用户A 请求 → 🔄 发起请求，立刻让出控制权
  用户B 请求 → 🔄 发起请求，立刻让出控制权
  用户C 请求 → 🔄 发起请求，立刻让出控制权
  （3 个请求几乎同时进行，共享同一个线程）
  用户A 的字到了 → 📩 推送给 A
  用户B 的字到了 → 📩 推送给 B
  ...
```

**实际使用场景：**

| 场景 | 为什么用 astream |
|------|-----------------|
| **FastAPI / Web 后端** | 一个线程服务成百上千并发用户，不能被单个请求阻塞 |
| **WebSocket 聊天室** | 多人同时在线聊天，需要并发推送 |
| **多 Agent 并行** | 多个 Agent 同时跑，各自流式输出要汇总 |
| **微服务调用** | 调 LLM 是网络 I/O，等待时可以去处理别的请求 |

> **一句话**：`stream` = 自己一个人用不排队；`astream` = 开餐厅同时接待很多客人。

### 6.1 基本用法

```python
import asyncio

async def main():
    # 同步 → 异步：stream 换 astream
    async for chunk in app.astream(
        {"question": "你好"},
        stream_mode="messages"
    ):
        token, metadata = chunk
        if token.content:
            print(token.content, end="", flush=True)

asyncio.run(main())
```

### 6.2 在 FastAPI 中使用（SSE 流式接口）

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import json

app_fastapi = FastAPI()

@app_fastapi.post("/chat")
async def chat(request: dict):
    question = request["question"]

    async def generate():
        async for token, metadata in agent.astream(
            {"messages": [{"role": "user", "content": question}]},
            stream_mode="messages"
        ):
            if token.content:
                # SSE 格式：data: {json}\n\n
                yield f"data: {json.dumps({'content': token.content})}\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

**前端（JavaScript）接收：**
```javascript
const response = await fetch("/chat", {
    method: "POST",
    headers: {"Content-Type": "application/json"},
    body: JSON.stringify({question: "你好"})
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
    const {done, value} = await reader.read();
    if (done) break;

    const text = decoder.decode(value);
    // 解析 "data: {...}\n\n"
    for (const line of text.split("\n\n")) {
        if (line.startsWith("data: ")) {
            const data = JSON.parse(line.slice(6));
            document.getElementById("output").textContent += data.content;
        }
    }
}
```

> **SSE（Server-Sent Events）** = 服务器向浏览器推送事件的标准协议。
> 这就是 ChatGPT 网页版打字机效果的实现原理。

---

## 七、自定义进度提示（stream_mode="custom"）

### 7.1 长任务的进度反馈

```python
from langgraph.types import StreamWriter

def research_and_answer(state: State, writer: StreamWriter):
    # 步骤 1：搜索
    writer({"progress": "🔍 正在搜索相关资料...", "percent": 20})
    documents = search_web(state["question"])

    # 步骤 2：分析
    writer({"progress": "🧠 正在分析资料...", "percent": 50})
    analysis = analyze(documents)

    # 步骤 3：生成
    writer({"progress": "✍️ 正在生成回答...", "percent": 80})
    answer = model.invoke(f"根据分析回答：{analysis}")

    writer({"progress": "✅ 完成！", "percent": 100})
    return {"answer": answer.content}

graph.add_node("research", research_and_answer)
```

**运行：**
```python
for chunk in app.stream(input, stream_mode="custom"):
    print(f"[{chunk['percent']:3d}%] {chunk['progress']}")
```

**模拟输出：**
```text
[ 20%] 🔍 正在搜索相关资料...
[ 50%] 🧠 正在分析资料...
[ 80%] ✍️ 正在生成回答...
[100%] ✅ 完成！
```

---

## 八、常见问题

### ❌ stream 输出是空的？

**可能原因**：节点内部用了 `model.invoke`（一次性），不是 `model.stream`（生成器）。

```python
# ❌ 不会流式（invoke 一次性返回）
def chatbot(state: State):
    response = model.invoke(state["question"])
    return {"answer": response.content}

# ✅ 流式（yield 逐块写入）
def chatbot(state: State):
    for chunk in model.stream(state["question"]):
        yield {"answer": chunk.content}
```

> **或者**用 `stream_mode="messages"`——它会**自动**帮你捕获 LLM 的 token 流，节点里用 `invoke` 也可以。

### ❌ 打印时字都挤在一行？

加 `flush=True` 和 `end=""`：

```python
# ❌ 字会换行，看不到打字机效果
print(token.content)

# ✅ 逐字追加
print(token.content, end="", flush=True)
```

### ❌ messages 模式拿到的 token 是空的？

`token.content` 可能为空字符串（比如思考过程、工具调用阶段）：

```python
for token, metadata in app.stream(input, stream_mode="messages"):
    if token.content:                    # ★ 过滤空内容
        print(token.content, end="", flush=True)
```

### ❌ 想看工具调用的中间过程？

用 `stream_mode=["updates", "messages"]` 同时看节点更新和 token：

```python
for mode, data in app.stream(input, stream_mode=["updates", "messages"]):
    if mode == "messages":
        token, meta = data
        if token.content:
            print(token.content, end="")
    elif mode == "updates":
        print(f"\n[节点更新] {data}")
```

**模拟输出：**
```text
[节点更新] {'tools': {'messages': [ToolMessage(content='北京：晴天 25°C')]}}
北京今天晴天，气温 25°C。
```

### ❌ 流式中途想中断？

```python
from langgraph.types import Interrupt

for chunk in app.stream(input, stream_mode="messages"):
    token, metadata = chunk
    if should_stop(token):        # 自定义中断条件
        break                     # 直接 break 即可
```

---

## 九、stream_mode 选择指南

```text
你的需求是什么？
│
├─ 「聊天界面要打字机效果」
│   └─→ stream_mode="messages"     ⭐ 最常用
│
├─ 「想知道每个节点输出了什么」
│   └─→ stream_mode="updates"      （默认）
│
├─ 「调试时想看状态完整变化」
│   └─→ stream_mode="values"
│
├─ 「长时间任务要进度提示」
│   └─→ stream_mode="custom"
│
└─ 「排查问题，要最详细的信息」
    └─→ stream_mode="debug"
```

---

## 十、速查表

### 运行方式

| 操作 | 代码 |
|------|------|
| 同步流式 | `for chunk in app.stream(input):` |
| 异步流式 | `async for chunk in app.astream(input):` |
| 指定模式 | `app.stream(input, stream_mode="messages")` |
| 多模式 | `app.stream(input, stream_mode=["updates", "messages"])` |

### stream_mode 对照

| 模式 | 流出内容 | 返回结构 |
|------|---------|---------|
| `"updates"` | 节点状态更新 | `{节点名: {字段: 值}}` |
| `"values"` | 完整状态 | `{字段: 值}` |
| `"messages"` | LLM token | `(AIMessageChunk, metadata)` |
| `"custom"` | 手动 writer 内容 | 你 writer 的数据 |
| `"debug"` | 调试信息 | 详细执行日志 |

### 节点写法

| 写法 | 效果 |
|------|------|
| `return {"answer": text}` | 一次性写入（不流式） |
| `yield {"answer": chunk}` | 逐块写入（流式） |
| `writer({"progress": "..."})` | 自定义进度推送 |

### 常用过滤

| 需求 | 代码 |
|------|------|
| 只要文本 | `if token.content: print(...)` |
| 只要 AI 内容 | `if token.type == "AIMessageChunk"` |
| 要节点信息 | `metadata["langgraph_node"]` |
| 要步骤号 | `metadata["langgraph_step"]` |

---

## 十一、深入理解：流式的本质

### 11.1 三种「流」不要混淆

```text
1. 模型流（LLM token 流）
   模型 ──春──风──拂──面──→  你的代码
   一个字一个字生成，stream_mode="messages" 拿到的就是这个

2. 节点流（节点执行流）
   START ──节点A──节点B──节点C──→ END
   每个节点完成流出一个 chunk，stream_mode="updates" 拿到的是这个

3. 状态流（状态变化流）
   初始状态 ──→ 节点A后状态 ──→ 节点B后状态 ──→ 最终状态
   stream_mode="values" 拿到的是这个
```

| 流类型 | stream_mode | 粒度 | 看到什么 |
|--------|:-----------:|:----:|---------|
| 模型流 | `messages` | 字/词 | 文本内容 |
| 节点流 | `updates` | 节点 | 节点输出 |
| 状态流 | `values` | 步骤 | 完整状态 |

### 11.2 类比：餐厅上菜

```text
❌ invoke（非流式）：
   点菜 → ⏳ 等厨房做好全部菜 → 一次上齐

✅ stream_mode="updates"（节点流）：
   点菜 → 🥗 先上凉菜 → 🍲 再上热菜 → 🍰 最后上甜点
   （按「菜品类型」上）

✅ stream_mode="messages"（模型流）：
   点菜 → 上一盘菜，边吃边上下一盘
   （按「一口一口」上）

✅ stream_mode="values"（状态流）：
   每上一道菜，都重新看一眼整桌菜
```

---

## 下一步

- [[功能-检查点-Checkpoint]] — 流式过程中的状态持久化
- [[功能-人工介入-Human-in-the-Loop]] — 流式过程中的中断与审批
- [[实战-React-Agent]] — 构建带流式输出的 Agent
