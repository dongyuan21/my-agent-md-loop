# LangChain vs LangGraph: 关系、异同与学习路径

> **核心结论**: LangGraph 是 LangChain 的**补充扩展**,专门解决 LangChain 无法处理的**循环/分支/状态管理**问题

---

## 一、时间线和演化关系

```text
2022年10月     2024年1月      2024年至今
    │             │              │
    ▼             ▼              ▼
┌─────────┐  ┌─────────┐    ┌─────────┐
│LangChain│  │LangGraph│    │LangGraph│
│  诞生   │──│  诞生   │────│  主导   │
└─────────┘  └─────────┘    └─────────┘
    │             │              │
    │             │              │
 链式调用      循环/状态      Agent框架
  (LCEL)       (StateGraph)   (持久化/HITL)
```

### 1.1 历史演变

**LangChain (2022-10 诞生):**
- Harrison Chase 创立
- 初衷: 简化 LLM 应用开发的"胶水层"
- 核心: Prompt 模板 + LLM 调用 + 输出解析
- 早期局限: 只能处理**线性链式**调用

**LangGraph (2024-01 诞生):**
- LangChain 团队推出
- 初衷: 解决 Agent 的**循环推理**和**状态管理**问题
- 灵感来源: Google Pregel (图计算框架)
- 定位: LangChain 的"编排引擎"升级

**现状 (2024-2025):**
- LangGraph 已成为 LangChain 生态的**核心**
- LangChain 官方推荐新项目直接用 LangGraph
- LangChain 变成"组件库",LangGraph 负责"编排执行"

---

## 二、核心差异对比

### 2.1 架构模型差异

```python
# ===== LangChain: 链式调用 (DAG - 有向无环图) =====
from langchain.prompts import ChatPromptTemplate
from langchain.chat_models import init_chat_model
from langchain.schema.output_parser import StrOutputParser

# 定义链 (Runnable Sequence)
prompt = ChatPromptTemplate.from_template("翻译成{language}: {text}")
model = init_chat_model("openai:gpt-4o")
parser = StrOutputParser()

chain = prompt | model | parser  # ⚠️ 单向流,不能循环

# 执行
result = chain.invoke({"language": "中文", "text": "Hello"})
# Output: "你好"

# ❌ 无法实现:
# - "如果翻译不准确,重新翻译" (需要循环)
# - "保存翻译历史" (需要状态)
# - "人工审核翻译" (需要中断/恢复)
```

```python
# ===== LangGraph: 图式编排 (Cyclic Graph - 有向有环图) =====
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

class State(TypedDict):
    text: str
    translation: str
    attempts: int

def translate(state: State) -> dict:
    # 调用 LLM 翻译
    result = model.invoke(f"翻译成中文: {state['text']}")
    return {"translation": result, "attempts": state["attempts"] + 1}

def check_quality(state: State) -> str:
    # 检查翻译质量 (可以是 LLM 评估或人工审核)
    if "你好" in state["translation"]:
        return END  # 翻译合格,结束
    elif state["attempts"] >= 3:
        return END  # 超过重试次数,结束
    else:
        return "translate"  # 重新翻译

# 构建图
graph = StateGraph(State)
graph.add_node("translate", translate)
graph.add_edge(START, "translate")
graph.add_conditional_edges("translate", check_quality)  # ⚠️ 支持循环回 translate

# 执行
app = graph.compile()
result = app.invoke({"text": "Hello", "translation": "", "attempts": 0})
# Output: {"text": "Hello", "translation": "你好", "attempts": 1}

# ✅ 可以实现:
# - 循环重试 (conditional_edges 回到 translate 节点)
# - 保存状态 (State 自动持久化)
# - 人工审核 (添加 interrupt 中断点)
```

---

### 2.2 核心能力对比表

| 能力 | LangChain | LangGraph | 说明 |
|------|-----------|-----------|------|
| **链式调用** | ✅ 核心能力 | ✅ 支持 | LangChain 的 LCEL 更简洁 |
| **循环/重试** | ❌ 不支持 | ✅ 原生支持 | LangGraph 的 conditional_edges |
| **分支逻辑** | ⚠️ 有限 (if/else) | ✅ 灵活 (多路分支) | LangGraph 可以任意跳转 |
| **状态管理** | ❌ 无状态 | ✅ 自动持久化 | LangGraph 的 Checkpoint 机制 |
| **中断/恢复** | ❌ 不支持 | ✅ Human-in-the-Loop | 工具调用前人工审批 |
| **时间旅行** | ❌ 不支持 | ✅ 回溯历史状态 | 从任意 checkpoint 重新执行 |
| **并行执行** | ⚠️ 有限 (batch) | ✅ 原生支持 | LangGraph 的 superstep 机制 |
| **Agent 框架** | ⚠️ 简单 Agent | ✅ 复杂 Multi-Agent | LangGraph 是 Agent 的标准编排 |
| **可观测性** | ⚠️ 基础 Callbacks | ✅ LangSmith 深度集成 | 可视化图执行、状态追踪 |
| **部署** | ⚠️ 需自行部署 | ✅ LangGraph Cloud | 托管部署 + 自动扩缩容 |

---

### 2.3 典型应用场景对比

#### LangChain 适用场景

```python
# ===== 1. 简单的 RAG (检索增强生成) =====
from langchain.vectorstores import FAISS
from langchain.embeddings import OpenAIEmbeddings
from langchain.chains import RetrievalQA

vectorstore = FAISS.from_texts(["文档1", "文档2"], OpenAIEmbeddings())
qa_chain = RetrievalQA.from_chain_type(
    llm=model,
    retriever=vectorstore.as_retriever()
)

result = qa_chain.invoke("什么是 LangChain?")
# ✅ 单轮检索 + 生成,无需状态

# ===== 2. 固定流程的数据处理 =====
chain = (
    {"context": retriever, "question": lambda x: x["question"]}
    | prompt
    | model
    | StrOutputParser()
)

# ✅ 流程固定,不需要循环/分支

# ===== 3. 工具链组合 (无循环) =====
from langchain.tools import DuckDuckGoSearchRun
from langchain.agents import AgentExecutor, create_react_agent

tools = [DuckDuckGoSearchRun()]
agent = create_react_agent(model, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, max_iterations=5)

# ⚠️ 问题: max_iterations 是硬编码,无法动态调整
# ⚠️ 问题: 无法保存中间状态,重启后丢失
```

#### LangGraph 适用场景

```python
# ===== 1. 多轮对话 Agent (需要记忆) =====
from langgraph.checkpoint.sqlite import SqliteSaver

checkpointer = SqliteSaver.from_conn_string("memory.db")
agent = create_deep_agent(
    tools=[search_tool, calculator],
    checkpointer=checkpointer  # ✅ 自动保存对话历史
)

config = {"configurable": {"thread_id": "user-123"}}
agent.invoke({"messages": [{"role": "user", "content": "帮我搜索 LangGraph"}]}, config)
agent.invoke({"messages": [{"role": "user", "content": "刚才搜到了什么?"}]}, config)
# ✅ 第二次调用能记住第一次的结果

# ===== 2. 复杂的规划/执行 Agent =====
def plan(state):
    # LLM 生成 TODO 列表
    return {"todos": llm.invoke("分解任务: " + state["task"])}

def execute(state):
    # 执行第一个 TODO
    todo = state["todos"][0]
    result = execute_tool(todo)
    return {"results": state["results"] + [result], "todos": state["todos"][1:]}

def should_continue(state):
    return "execute" if state["todos"] else END

graph = StateGraph(State)
graph.add_node("plan", plan)
graph.add_node("execute", execute)
graph.add_edge(START, "plan")
graph.add_edge("plan", "execute")
graph.add_conditional_edges("execute", should_continue)  # ✅ 循环执行所有 TODO

# ===== 3. Human-in-the-Loop 工作流 =====
agent = create_deep_agent(
    tools=[delete_file, send_email],
    interrupt_on={
        "delete_file": {"allowed_decisions": ["approve", "reject"]},
        "send_email": {"allowed_decisions": ["approve", "edit", "reject"]}
    },
    checkpointer=checkpointer
)

# 第一阶段: 执行到中断点
result = agent.invoke(input, config)
assert result["__interrupt__"] is not None

# 用户审批后恢复
result2 = agent.invoke(Command(resume={"decisions": [{"type": "approve"}]}), config)
# ✅ 从中断点继续执行
```

---

## 三、LangChain 补足了什么能力?

### 3.1 LangChain 的核心局限

**问题1: 无法处理循环逻辑**

```python
# LangChain 的 AgentExecutor 实现 (简化版)
class AgentExecutor:
    def __init__(self, agent, tools, max_iterations=15):
        self.agent = agent
        self.tools = tools
        self.max_iterations = max_iterations  # ⚠️ 硬编码的循环次数
    
    def invoke(self, input):
        for i in range(self.max_iterations):
            # 1. LLM 决策下一步
            action = self.agent.invoke(input)
            
            # 2. 如果决定结束,跳出循环
            if action.is_finish:
                return action.output
            
            # 3. 执行工具
            result = self.tools[action.tool].invoke(action.input)
            
            # 4. 更新输入 (但无法回溯/分支)
            input = update_input(input, result)
        
        # ⚠️ 超过最大次数,强制结束
        raise MaxIterationsError()

# 问题:
# - max_iterations 是固定的,无法动态调整
# - 无法实现 "如果工具A失败,重试3次,然后尝试工具B"
# - 无法保存中间状态,重启后丢失所有进度
```

**LangGraph 的解决方案:**

```python
# LangGraph 的动态循环
def should_continue(state):
    if state["success"]:
        return END
    elif state["retry_count"] >= 3:
        return "fallback_tool"  # 尝试备用工具
    else:
        return "main_tool"  # 重试主工具

graph.add_conditional_edges("main_tool", should_continue)
# ✅ 完全动态,由状态决定下一步
```

---

**问题2: 无法中断和恢复**

```python
# LangChain 的 AgentExecutor
executor = AgentExecutor(agent, tools)
result = executor.invoke({"input": "删除所有日志文件"})

# ❌ 问题:
# - 工具会立即执行,无法中断审批
# - 如果进程崩溃,无法恢复
# - 无法查看中间状态

# LangGraph 的解决方案:
agent = create_deep_agent(
    tools=[delete_files],
    interrupt_on={"delete_files": True},  # ⚠️ 中断点
    checkpointer=checkpointer  # ⚠️ 持久化
)

# 第一阶段: 执行到中断
result = agent.invoke(input, config)
# 返回 {"__interrupt__": [...]}

# 用户审批后恢复
result2 = agent.invoke(Command(resume={...}), config)
# ✅ 从中断点继续,即使进程重启也能恢复
```

---

**问题3: 无法管理状态**

```python
# LangChain 的链式调用
chain = prompt | model | parser

# ❌ 每次调用都是新的,无法记住之前的结果
result1 = chain.invoke({"input": "计算 2+3"})  # Output: 5
result2 = chain.invoke({"input": "加上刚才的结果"})  # ❌ LLM 不知道 "刚才" 是什么

# LangGraph 的状态管理:
class State(TypedDict):
    messages: list[BaseMessage]
    result: int

graph = StateGraph(State)
# ...

checkpointer = SqliteSaver.from_conn_string("state.db")
app = graph.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "calc-session"}}
app.invoke({"messages": [HumanMessage("计算 2+3")]}, config)  # State: {"result": 5}
app.invoke({"messages": [HumanMessage("加上刚才的结果")]}, config)  # ✅ State 持久化,能访问 result=5
```

---

### 3.2 LangGraph 补足的核心能力

| 能力 | LangChain 的问题 | LangGraph 的解决方案 |
|------|-----------------|---------------------|
| **循环逻辑** | 硬编码 max_iterations | conditional_edges 动态决策 |
| **分支逻辑** | 只能简单 if/else | 多路分支,任意跳转 |
| **状态管理** | 无状态,每次重新开始 | Checkpoint 自动持久化 |
| **中断/恢复** | 无法中断,也无法恢复 | NodeInterrupt + pending_writes |
| **时间旅行** | 无历史记录 | checkpoint_id 回溯任意状态 |
| **并行执行** | batch() 有限支持 | superstep 原生并行 |
| **可观测性** | Callbacks 基础追踪 | LangSmith 可视化图执行 |
| **生产部署** | 需自行管理进程/队列 | LangGraph Cloud 托管 |

---

## 四、学习路径建议

### 4.1 快速决策树

```text
你要构建的是什么?
    │
    ▼
┌────────────────────────────────┐
│ 简单的 RAG / 数据处理 Pipeline │
└────────┬───────────────────────┘
         │
         ├─ 是 → 【先学 LangChain】
         │      - LCEL (Runnable Sequence)
         │      - Prompt 模板
         │      - 输出解析
         │
         └─ 否 → 【直接学 LangGraph】
                 ↓
         ┌───────────────────────┐
         │ 需要循环/分支/状态吗? │
         └──────┬────────────────┘
                │
                ├─ 是 → LangGraph 必需
                │      - StateGraph
                │      - Checkpoint
                │      - Human-in-the-Loop
                │
                └─ 否,但未来可能需要
                       → 还是推荐 LangGraph
                         (LangChain 能做的它都能做)
```

---

### 4.2 学习顺序推荐 (2025年标准)

#### 路径1: 如果你是新手 (推荐)

```text
1️⃣ 【LangChain 基础】(1-2天)
   ├─ Prompt 模板: ChatPromptTemplate
   ├─ LLM 调用: init_chat_model
   ├─ 输出解析: StrOutputParser, JsonOutputParser
   └─ LCEL 语法: chain = prompt | model | parser

2️⃣ 【LangGraph 核心】(3-5天) ⚠️ 重点
   ├─ StateGraph 基础: State + Nodes + Edges
   ├─ 条件路由: conditional_edges
   ├─ Checkpoint 机制: SqliteSaver, 状态持久化
   ├─ Human-in-the-Loop: interrupt_on, Command(resume)
   └─ 时间旅行: get_state, update_state

3️⃣ 【实战项目】(1-2周)
   ├─ 构建一个 Agent (create_deep_agent)
   ├─ 添加自定义工具
   ├─ 实现 HITL 审批流程
   └─ 部署到 LangGraph Cloud

4️⃣ 【高级主题】(按需学习)
   ├─ Multi-Agent 系统 (SubAgentMiddleware)
   ├─ 长期记忆 (LangGraph Store)
   ├─ 并行执行优化
   └─ LangSmith 监控调试
```

**时间分配:**
- LangChain: 20%
- LangGraph: 60%
- 实战: 20%

---

#### 路径2: 如果你已经熟悉 LangChain

```text
直接学 LangGraph,重点关注:
1️⃣ StateGraph vs LCEL 的区别
2️⃣ Checkpoint 机制 (这是 LangChain 完全没有的)
3️⃣ conditional_edges 实现复杂逻辑
4️⃣ 迁移现有 LangChain 代码到 LangGraph

迁移示例:
# Before (LangChain)
chain = prompt | model | parser

# After (LangGraph)
class State(TypedDict):
    input: str
    output: str

def node(state):
    result = (prompt | model | parser).invoke(state["input"])
    return {"output": result}

graph = StateGraph(State)
graph.add_node("process", node)
graph.add_edge(START, "process")
graph.add_edge("process", END)

app = graph.compile()
```

---

### 4.3 学习资源推荐

#### LangChain (基础组件)

```text
✅ 官方文档: https://docs.langchain.com/oss/python/langchain/overview
✅ GitHub: https://github.com/langchain-ai/langchain
✅ 重点学习:
   - LCEL (Runnable Sequence)
   - Prompt 模板
   - Chat Models 集成
   - Output Parsers
   - Tools (工具定义)
```

#### LangGraph (核心编排)

```text
✅ 官方文档: https://docs.langchain.com/oss/python/langgraph/overview
✅ GitHub: https://github.com/langchain-ai/langgraph
✅ 重点学习:
   - StateGraph 基础
   - Checkpoint 机制 (重中之重!)
   - Human-in-the-Loop
   - Subgraphs (子图)
   - LangSmith 集成
```

#### DeepAgents (高级应用)

```text
✅ GitHub: https://github.com/langchain-ai/deepagents
✅ 重点学习:
   - create_deep_agent API
   - FilesystemMiddleware (文件操作)
   - SubAgentMiddleware (多 Agent 系统)
   - HumanInTheLoopMiddleware (HITL)
   - Agent Memory (长期记忆)
```

---

## 五、实战对比: 同一个需求的实现

### 5.1 需求: 搜索 + 总结 + 翻译

#### LangChain 实现 (链式)

```python
from langchain.prompts import ChatPromptTemplate
from langchain.chat_models import init_chat_model
from langchain.schema.output_parser import StrOutputParser
from langchain.tools import DuckDuckGoSearchRun

model = init_chat_model("openai:gpt-4o")
search = DuckDuckGoSearchRun()

# ===== 步骤1: 搜索 =====
search_chain = (
    ChatPromptTemplate.from_template("搜索: {query}")
    | model
    | StrOutputParser()
    | (lambda x: search.invoke(x))
)

# ===== 步骤2: 总结 =====
summarize_chain = (
    ChatPromptTemplate.from_template("总结以下内容: {content}")
    | model
    | StrOutputParser()
)

# ===== 步骤3: 翻译 =====
translate_chain = (
    ChatPromptTemplate.from_template("翻译成中文: {text}")
    | model
    | StrOutputParser()
)

# ===== 组合链 =====
full_chain = (
    search_chain
    | (lambda x: {"content": x})
    | summarize_chain
    | (lambda x: {"text": x})
    | translate_chain
)

result = full_chain.invoke({"query": "What is LangGraph?"})

# ❌ 问题:
# 1. 无法保存中间状态 (搜索结果、总结结果丢失)
# 2. 如果翻译失败,无法重试
# 3. 无法人工审核总结内容
# 4. 进程崩溃后无法恢复
```

#### LangGraph 实现 (图式)

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict
from langchain.tools import DuckDuckGoSearchRun

class State(TypedDict):
    query: str
    search_results: str
    summary: str
    translation: str
    retry_count: int

search = DuckDuckGoSearchRun()
model = init_chat_model("openai:gpt-4o")

# ===== 节点定义 =====
def search_node(state: State):
    results = search.invoke(state["query"])
    return {"search_results": results}

def summarize_node(state: State):
    summary = model.invoke(f"总结: {state['search_results']}")
    return {"summary": summary}

def translate_node(state: State):
    translation = model.invoke(f"翻译成中文: {state['summary']}")
    return {"translation": translation, "retry_count": state.get("retry_count", 0) + 1}

def check_translation(state: State):
    # 检查翻译质量
    if "LangGraph" in state["translation"] or state["retry_count"] >= 3:
        return END
    else:
        return "translate"  # 重新翻译

# ===== 构建图 =====
graph = StateGraph(State)
graph.add_node("search", search_node)
graph.add_node("summarize", summarize_node)
graph.add_node("translate", translate_node)

graph.add_edge(START, "search")
graph.add_edge("search", "summarize")
graph.add_edge("summarize", "translate")
graph.add_conditional_edges("translate", check_translation)  # ✅ 循环重试

# ===== 持久化 + HITL =====
from langgraph.checkpoint.sqlite import SqliteSaver

checkpointer = SqliteSaver.from_conn_string("workflow.db")
app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["summarize"]  # ✅ 人工审核总结
)

# ===== 执行 =====
config = {"configurable": {"thread_id": "workflow-1"}}

# 第一阶段: 搜索 + 中断
result = app.invoke({"query": "What is LangGraph?", "retry_count": 0}, config)
# 中断在 summarize 前,返回当前状态

# 检查搜索结果
print(result["search_results"])

# 用户批准后继续
result2 = app.invoke(None, config)  # ✅ 从中断点恢复
print(result2["translation"])

# ✅ 优势:
# 1. 所有中间状态都保存在 checkpointer
# 2. 翻译失败可以自动重试 (conditional_edges)
# 3. 人工审核总结内容 (interrupt_before)
# 4. 进程崩溃后可以从 workflow.db 恢复
```

---

## 六、常见问题

### Q1: LangChain 会被淘汰吗?

**A:** 不会,但角色转变:

```text
LangChain 的未来角色:
- ✅ 组件库 (Prompt, Parser, Tools, Embeddings)
- ✅ 集成层 (300+ LLM/Vector DB/Tool 集成)
- ✅ 简单链式调用 (LCEL 仍然好用)

LangGraph 的核心地位:
- ✅ Agent 编排引擎 (取代 AgentExecutor)
- ✅ 复杂工作流 (循环/分支/状态)
- ✅ 生产级部署 (LangGraph Cloud)

关系:
LangChain 提供积木 (Components)
LangGraph 负责搭建 (Orchestration)
```

---

### Q2: 已经学了 LangChain,需要重新学吗?

**A:** 不需要重学,是**升级**:

```python
# LangChain 的知识在 LangGraph 中仍然有效

# ✅ Prompt 模板照常用
from langchain.prompts import ChatPromptTemplate
prompt = ChatPromptTemplate.from_template("翻译: {text}")

# ✅ LLM 调用照常用
from langchain.chat_models import init_chat_model
model = init_chat_model("openai:gpt-4o")

# ✅ 工具定义照常用
from langchain_core.tools import tool
@tool
def search(query: str):
    """搜索工具"""
    return "..."

# ⚠️ 唯一变化: 用 LangGraph 编排
from langgraph.graph import StateGraph

graph = StateGraph(State)
graph.add_node("node", lambda s: (prompt | model).invoke(s))
# ...

# 本质: LangChain 是"零件",LangGraph 是"组装工艺"
```

---

### Q3: LangGraph 比 LangChain 难学吗?

**A:** 初期学习曲线稍陡,但值得:

```text
LangChain 学习曲线:
时间 │     
    │  ╱───  (快速上手,但遇到复杂需求无解)
    │ ╱
    └──────── 难度

LangGraph 学习曲线:
时间 │         
    │      ╱─── (掌握后可以应对任何复杂度)
    │    ╱
    │  ╱
    └──────── 难度

投入产出比:
- LangChain: 1天入门,但复杂需求做不了
- LangGraph: 3天入门,可以应对生产级复杂度
```

---

## 七、最终建议 (Linus 式)

### ✅ 2025年学习路径

```text
如果你是新手:
1. 快速过一遍 LangChain 基础 (1天)
   - Prompt 模板
   - LLM 调用
   - LCEL 语法
   
2. 直接深入 LangGraph (3-5天) ⚠️ 重点
   - StateGraph 是核心
   - Checkpoint 是灵魂
   - conditional_edges 是精髓
   
3. 实战项目 (用 LangGraph 构建 Agent)

如果你已经会 LangChain:
1. 跳过基础,直接学 LangGraph
2. 重点关注 Checkpoint + HITL
3. 迁移现有项目到 LangGraph
```

---

### ❌ Linus 的批判

> **"LangChain 的 AgentExecutor 是个笑话"**
> 
> "硬编码 max_iterations=15? 这是 2010 年的编程水平。真实的 Agent 需要动态决策,需要状态管理,需要中断恢复。LangChain 团队意识到这个问题,所以做了 LangGraph。"
>
> **"LangGraph 是正确的设计"**
>
> "用图来表达控制流,这是计算机科学的基础。Pregel (Google) 和 Apache Beam 都证明了这个模式的正确性。LangGraph 只是把它应用到 LLM 领域。"
>
> **"不要浪费时间学过时的东西"**
>
> "如果你现在还在学 LangChain 的 AgentExecutor,就像在 2024 年学 Python 2 一样。直接学 LangGraph,它能做 LangChain 能做的一切,还能做 LangChain 做不到的。"

---

**总结:**

| 维度 | 结论 |
|------|------|
| **关系** | LangGraph 是 LangChain 的**编排引擎升级** |
| **时间** | LangChain (2022) → LangGraph (2024) |
| **核心差异** | 链式 vs 图式,无状态 vs 有状态 |
| **学习顺序** | 快速过 LangChain 基础 → 深入 LangGraph 核心 |
| **2025建议** | 新项目直接用 LangGraph (LangChain 作为组件库) |
