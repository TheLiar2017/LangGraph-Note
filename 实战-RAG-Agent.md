---
aliases: [RAG, RAG Agent, 检索增强生成, 知识库, 检索增强]
tags: [实战, Agent, RAG, 检索]
---

# 📚 实战：RAG Agent

> AI 不知道你公司的文档、不知道最新的产品信息——它只学过公开数据。
> RAG = **R**etrieval-**A**ugmented **G**eneration（检索增强生成）——先**查资料**，再**回答问题**。
> 本文从零搭建一个带知识库检索能力的 RAG Agent。

---

## 一、RAG 是什么？

### 1.1 一句话理解

> **RAG = 开卷考试。** 先翻书找答案，再组织语言回答——而不是凭记忆瞎编。

```text
❌ 没有 RAG（闭卷考试）：

  用户: "我们公司的退款政策是什么？"
  AI:   "一般来说，退款政策是 7 天无理由退款..."   ← 在瞎编！
        （AI 不知道你公司的具体政策）


✅ 有 RAG（开卷考试）：

  用户: "我们公司的退款政策是什么？"
  🔍 检索: 在知识库中搜索 → 找到「退款政策.docx」
  AI:   "根据公司退款政策：30 天内无理由退款，需提供订单号..."
        （基于真实文档回答）
```

### 1.2 RAG 的核心流程

```text
用户提问
  │
  ▼
🔍 检索（Retrieval）         ← 在知识库中搜索相关文档
  │
  ▼
📋 找到相关文档片段
  │
  ▼
🤖 生成（Generation）         ← 把文档 + 问题一起给 LLM
  │
  ▼
💬 基于文档的回答
```

### 1.3 为什么需要 RAG？

| 问题 | RAG 怎么解决 |
|------|-------------|
| AI 不知道你的私有数据 | ✅ 把私有文档做成知识库，检索后给 AI |
| AI 的知识会过时 | ✅ 更新知识库就行，不用重新训练 |
| AI 会「幻觉」编造 | ✅ 基于检索到的真实文档回答 |
| 想知道答案的出处 | ✅ 返回来源文档，可追溯 |

---

## 二、RAG 的三大组件

```text
┌──────────────────────────────────────────────────┐
│                   RAG Agent                       │
│                                                   │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │ 📄 知识库  │  │ 🔍 检索器  │  │ 🤖 生成器  │    │
│  │           │  │           │  │           │    │
│  │ 文档片段  │  │ 向量搜索  │  │ LLM 组织  │    │
│  │ + 向量   │  │ 相似匹配  │  │ 语言回答  │    │
│  └───────────┘  └───────────┘  └───────────┘    │
│                                                   │
│  文档怎么存？   怎么找到相关的？   怎么组织回答？  │
└──────────────────────────────────────────────────┘
```

| 组件 | 作用 | 常用工具 |
|------|------|---------|
| **知识库** | 存储文档片段 + 向量 | ChromaDB、FAISS、Pinecone |
| **检索器** | 根据问题搜索相关片段 | 向量相似度搜索 |
| **生成器** | 根据检索结果生成回答 | LLM（DeepSeek/GPT/Claude） |

---

## 三、快速上手：用 LangChain 搭建 RAG

### 3.1 安装依赖

```bash
pip install langchain-openai langchain-community chromadb
```

### 3.2 完整代码：基础 RAG

```python
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.schema import Document

# ═══ 1. 准备文档 ═══
docs = [
    Document(page_content="LangGraph 是 LangChain 生态中的低层级编排框架，用于构建有状态、可持久化的 AI Agent 工作流。"),
    Document(page_content="LangGraph 基于图（Graph）架构，支持循环、分支、人工介入和长期记忆。"),
    Document(page_content="StateGraph 是 LangGraph 的核心类，用于定义状态图。"),
    Document(page_content="Checkpoint 是 LangGraph 的持久化机制，可以保存和恢复 Agent 状态。"),
    Document(page_content="工具调用（Tool Calling）允许 AI 在对话中请求执行外部函数。"),
    Document(page_content="流式输出（Streaming）可以让用户实时看到 AI 生成的内容。"),
]

# ═══ 2. 文本分割（可选，文档长时需要）═══
splitter = RecursiveCharacterTextSplitter(chunk_size=200, chunk_overlap=20)
split_docs = splitter.split_documents(docs)

# ═══ 3. 创建向量知识库 ═══
embeddings = OpenAIEmbeddings(    # 用于把文本转成向量
    model="text-embedding-v3",
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",   # 通义 embedding
    api_key="sk-xxx",
)

vectorstore = Chroma.from_documents(
    split_docs,
    embeddings,
    persist_directory="./chroma_db",    # 持久化到本地
)

# ═══ 4. 创建检索器 ═══
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})   # 取最相关的 3 条

# ═══ 5. 测试检索 ═══
results = retriever.invoke("LangGraph 是什么？")
for doc in results:
    print(f"📄 {doc.page_content}")
```

**模拟输出：**
```text
📄 LangGraph 是 LangChain 生态中的低层级编排框架，用于构建有状态、可持久化的 AI Agent 工作流。
📄 LangGraph 基于图（Graph）架构，支持循环、分支、人工介入和长期记忆。
📄 StateGraph 是 LangGraph 的核心类，用于定义状态图。
```

> **向量搜索的原理**：把文本转成一串数字（向量），语义相近的文本向量也相近。搜索时找向量最近的文档。

---

## 四、RAG + Agent：让 Agent 能查知识库

> 把检索器包装成**工具**，让 ReAct Agent 自己决定什么时候查知识库。

### 4.1 完整代码

```python
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import InMemorySaver
from langchain_openai import ChatOpenAI
from langchain.tools import tool
from langchain.messages import SystemMessage

# ═══ 1. 检索工具 ═══
@tool
def search_knowledge(query: str) -> str:
    """在知识库中搜索相关信息。当用户问到产品信息、政策、文档内容等问题时使用此工具。

    Args:
        query: 搜索关键词或问题
    """
    results = retriever.invoke(query)
    if not results:
        return "知识库中没有找到相关信息。"

    content = "\n\n".join([f"[来源 {i+1}] {doc.page_content}" for i, doc in enumerate(results)])
    return content

# ═══ 2. 创建模型 ═══
model = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx",
)

# ═══ 3. 系统提示 ═══
system_prompt = SystemMessage(content="""你是一个知识库助手。

规则：
1. 回答前必须先使用 search_knowledge 工具搜索知识库
2. 基于搜索到的内容回答，不要编造
3. 如果知识库中没有答案，明确说"知识库中没有相关信息"
4. 回答时引用来源，如"根据文档 [来源1]..."

""")

# ═══ 4. 创建 Agent ═══
rag_agent = create_react_agent(
    model,
    tools=[search_knowledge],
    prompt=system_prompt,
    checkpointer=InMemorySaver(),
)

# ═══ 5. 运行 ═══
config = {"configurable": {"thread_id": "rag-001"}}
result = rag_agent.invoke(
    {"messages": [{"role": "user", "content": "LangGraph 是什么？有什么特点？"}]},
    config,
)

print(result["messages"][-1].content)
```

**模拟输出：**
```text
根据文档，LangGraph 是 LangChain 生态中的低层级编排框架，用于构建有状态、
可持久化的 AI Agent 工作流 [来源1]。

它的主要特点包括：
- 基于图（Graph）架构 [来源2]
- 支持循环、分支、人工介入和长期记忆 [来源2]
- 通过 StateGraph 定义状态图 [来源3]
```

---

## 五、完整实战：企业知识库 Agent

> 做一个能回答公司内部问题的知识库助手——支持多文档、带引用、多轮对话。

### 5.1 完整代码

```python
from langchain_community.vectorstores import Chroma
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.schema import Document
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import InMemorySaver
from langchain.tools import tool
from langchain.messages import SystemMessage
import os

# ═══ 1. 加载文档 ═══

def load_documents(folder_path: str):
    """从文件夹加载文档"""
    docs = []
    for filename in os.listdir(folder_path):
        filepath = os.path.join(folder_path, filename)
        with open(filepath, "r", encoding="utf-8") as f:
            content = f.read()
            docs.append(Document(
                page_content=content,
                metadata={"source": filename}    # 记录来源文件名
            ))
    return docs

# 创建模拟文档
documents = [
    Document(
        page_content="公司退款政策：30天内无理由退款。退款需提供订单号和购买凭证。退款将在3个工作日内到账。",
        metadata={"source": "退款政策.txt"},
    ),
    Document(
        page_content="员工假期政策：正式员工享有每年15天带薪年假。年假需提前3天申请。未休年假可折算为工资。",
        metadata={"source": "假期政策.txt"},
    ),
    Document(
        page_content="技术支持流程：用户提交工单后，1级支持需在2小时内响应。复杂问题升级至2级支持。",
        metadata={"source": "技术支持.txt"},
    ),
]

# ═══ 2. 创建向量知识库 ═══
embeddings = OpenAIEmbeddings(
    model="text-embedding-v3",
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
    api_key="sk-xxx",
)

vectorstore = Chroma.from_documents(
    documents,
    embeddings,
    persist_directory="./company_kb",
)
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# ═══ 3. 检索工具（带来源）═══
@tool
def search_company_docs(query: str) -> str:
    """搜索公司内部文档（政策、流程、制度等）。当用户问到公司相关规定时使用。

    Args:
        query: 搜索关键词或问题
    """
    results = retriever.invoke(query)
    if not results:
        return "未找到相关文档。"

    output = []
    for i, doc in enumerate(results):
        source = doc.metadata.get("source", "未知")
        output.append(f"[来源 {i+1}: {source}]\n{doc.page_content}")
    return "\n\n---\n\n".join(output)

# ═══ 4. 创建 Agent ═══
model = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com/v1",
    api_key="sk-xxx",
)

system_prompt = SystemMessage(content="""你是公司知识库助手，帮助员工查询公司政策和流程。

规则：
1. 必须先使用 search_company_docs 搜索文档
2. 基于文档内容回答，不要编造
3. 每个要点标注来源文件名，如（来源：退款政策.txt）
4. 如果文档中没有答案，明确说"文档中没有找到相关信息，建议咨询 HR"
5. 回答要专业、准确、简洁
""")

rag_agent = create_react_agent(
    model,
    tools=[search_company_docs],
    prompt=system_prompt,
    checkpointer=InMemorySaver(),
)

# ═══ 5. 多轮对话 ═══
config = {"configurable": {"thread_id": "employee-001"}}

# 问题 1
result = rag_agent.invoke(
    {"messages": [{"role": "user", "content": "公司退款政策是怎样的？"}]},
    config,
)
print("👤: 公司退款政策是怎样的？")
print("🤖:", result["messages"][-1].content)
```

**模拟输出：**
```text
👤: 公司退款政策是怎样的？
🤖: 根据公司退款政策（来源：退款政策.txt）：
    1. 30 天内无理由退款
    2. 退款需提供订单号和购买凭证
    3. 退款将在 3 个工作日内到账
```

```python
# 问题 2（多轮）
result = rag_agent.invoke(
    {"messages": [{"role": "user", "content": "那年假有多少天？"}]},
    config,
)
print("\n👤: 那年假有多少天？")
print("🤖:", result["messages"][-1].content)
```

**模拟输出：**
```text
👤: 那年假有多少天？
🤖: 根据公司假期政策（来源：假期政策.txt）：
    1. 正式员工享有每年 15 天带薪年假
    2. 年假需提前 3 天申请
    3. 未休年假可折算为工资
```

---

## 六、RAG 的关键：文档预处理

> 文档的处理质量直接决定 RAG 的效果。

### 6.1 文本分割

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,      # 每段最大 500 字符
    chunk_overlap=50,    # 相邻段重叠 50 字符（避免信息断裂）
    separators=["\n\n", "\n", "。", "！", "？", "；", "，"],   # 按这些符号分割
)

chunks = splitter.split_documents(documents)
```

### 6.2 分割参数调优

| 参数 | 太小 | 太大 | 建议 |
|------|------|------|------|
| `chunk_size` | 信息碎片化 | 检索不精确 | 200~1000 |
| `chunk_overlap` | 上下文断裂 | 冗余 | chunk_size 的 10~20% |

### 6.3 支持的文档格式

```bash
pip install pypdf python-docx unstructured
```

```python
# PDF
from langchain_community.document_loaders import PyPDFLoader
loader = PyPDFLoader("公司手册.pdf")
docs = loader.load()

# Word
from langchain_community.document_loaders import Docx2txtLoader
loader = Docx2txtLoader("政策.docx")
docs = loader.load()

# 网页
from langchain_community.document_loaders import WebBaseLoader
loader = WebBaseLoader("https://example.com/docs")
docs = loader.load()
```

---

## 七、Embedding 模型选择

> 向量化（Embedding）是 RAG 的关键——用什么模型把文本变成向量。

| 模型 | base_url | 说明 |
|------|----------|------|
| 通义 text-embedding-v3 | `https://dashscope.aliyuncs.com/compatible-mode/v1` | 国内推荐 |
| OpenAI text-embedding-3 | `https://api.openai.com/v1` | 效果好 |
| BGE-M3（开源） | 本地部署 | 中文效果好，免费 |
| Ollama nomic-embed | 本地部署 | 免费，隐私 |

```python
# 通义 Embedding
embeddings = OpenAIEmbeddings(
    model="text-embedding-v3",
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
    api_key="sk-xxx",
)

# Ollama 本地 Embedding
from langchain_ollama import OllamaEmbeddings
embeddings = OllamaEmbeddings(model="nomic-embed-text")
```

> **注意**：查询和文档必须用**同一个 Embedding 模型**，否则向量空间不一致，搜索结果不准确。

---

## 八、常见问题

### ❌ 检索到的内容不相关？

1. **检查 chunk_size**：太大或太小都会影响
2. **增加 top_k**：`search_kwargs={"k": 5}` 取更多结果
3. **优化查询**：用户问题越具体，检索越准
4. **换 Embedding 模型**：中文推荐 BGE-M3 或通义

### ❌ 回答里没有引用来源？

在系统提示中强调：

```python
system_prompt = SystemMessage(content="回答时必须标注来源文件名。")
```

### ❌ 知识库怎么更新？

```python
# 添加新文档
vectorstore.add_documents([new_doc])

# 删除某个文档
vectorstore.delete(ids=["doc_id"])

# 重建整个知识库（最简单）
Chroma.from_documents(all_docs, embeddings, persist_directory="./kb")
```

### ❌ 文档很大怎么办？

1. 先分割再入库（chunk_size=500）
2. 用更好的 Embedding 模型
3. 加 reranker（重排序）提升检索精度
4. 考虑用向量数据库（Pinecone、Weaviate）

---

## 九、速查表

### RAG 流程

| 步骤 | 代码 |
|------|------|
| 加载文档 | `PyPDFLoader(path).load()` |
| 分割文本 | `RecursiveCharacterTextSplitter().split_documents(docs)` |
| 创建向量库 | `Chroma.from_documents(docs, embeddings)` |
| 检索 | `retriever.invoke("问题")` |
| 包装成工具 | `@tool def search(query): return retriever.invoke(query)` |

### 关键参数

| 参数 | 建议值 | 说明 |
|------|--------|------|
| `chunk_size` | 200~1000 | 文本片段大小 |
| `chunk_overlap` | 50~100 | 相邻片段重叠 |
| `k` | 3~5 | 检索返回条数 |
| Embedding 模型 | 通义/BGE-M3 | 中文效果好 |

### 常用库

| 库 | 安装 | 用途 |
|-----|------|------|
| chromadb | `pip install chromadb` | 向量数据库 |
| pypdf | `pip install pypdf` | 读 PDF |
| python-docx | `pip install python-docx` | 读 Word |
| sentence-transformers | `pip install sentence-transformers` | 本地 Embedding |

---

## 十、深入理解：RAG 的本质

### 10.1 类比：查字典

```text
闭卷考试（没有 RAG）：
  老师问："光速是多少？"
  学生凭记忆答："好像是 30 万吧..."  ← 可能记错

开卷考试（有 RAG）：
  老师问："光速是多少？"
  学生翻书 → 找到相关页面 → "书上写的是 299,792,458 米/秒"  ← 有据可查

RAG Agent = 一个会自己翻书的学生
  1. 听到问题
  2. 翻书（检索知识库）
  3. 找到相关内容
  4. 组织语言回答
  5. 注明出处
```

### 10.2 RAG vs 微调 vs 提示工程

| | RAG | 微调 | 提示工程 |
|---|:---:|:---:|:---:|
| **成本** | 低 | 高 | 最低 |
| **更新知识** | ✅ 改知识库就行 | ❌ 重新训练 | ❌ 重写提示词 |
| **私有数据** | ✅ | ✅ | ⚠️ 塞进上下文 |
| **准确性** | ✅ 有据可查 | ⚠️ 可能遗忘 | ⚠️ 可能幻觉 |
| **适合场景** | 知识问答 | 领域适应 | 简单任务 |

> **一句话**：RAG 是性价比最高的「让 AI 知道你的数据」的方式。

---

## 下一步

- [[实战-自定义Agent]] — 自定义节点和路由
- [[实战-多Agent协作]] — 多 Agent 分工合作
- [[功能-工具调用-Tool-Calling]] — RAG 的检索就是一种工具调用