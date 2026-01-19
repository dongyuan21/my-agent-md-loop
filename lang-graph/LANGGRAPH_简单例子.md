# LangGraph 最简单示例的底层机制完全解析

> **从用户代码到底层实现**: 以 README 中的 6 行代码为例,揭示 State → Channels → PregelNode 的完整转换过程

---

## 一、用户代码 (表面层)

```python
# 来源: langgraph-main/README.md:L28-53

from langgraph.graph import START, StateGraph
from typing_extensions import TypedDict

# ===== 1️⃣ 定义 State =====
class State(TypedDict):
    text: str

# ===== 2️⃣ 定义节点函数 =====
def node_a(state: State) -> dict:
    return {"text": state["text"] + "a"}

def node_b(state: State) -> dict:
    return {"text": state["text"] + "b"}

# ===== 3️⃣ 构建图 =====
graph = StateGraph(State)
graph.add_node("node_a", node_a)
graph.add_node("node_b", node_b)
graph.add_edge(START, "node_a")
graph.add_edge("node_a", "node_b")

# ===== 4️⃣ 编译 & 执行 =====
app = graph.compile()
result = app.invoke({"text": ""})
print(result)
# Output: {'text': 'ab'}
```

**看似简单的 6 行代码,底层发生了什么?** 让我们一步步拆解。

---

## 二、第 1 步: StateGraph(State) - State 到 Channels 的转换

### 2.1 用户视角: 定义 State

```python
class State(TypedDict):
    text: str
```

**用户的意图**: 定义一个状态,包含一个字符串字段 `text`。

---

### 2.2 底层发生了什么?

```python
# 源码位置: langgraph/graph/state.py:L197-250

class StateGraph:
    def __init__(self, state_schema: type[StateT], ...):
        self.state_schema = state_schema  # ⚠️ 保存 State 类型
        
        # ===== 核心步骤: 将 State 转换为 Channels =====
        self._add_schema(self.state_schema)
```

#### 2.2.1 `_add_schema` 做了什么?

```python
# 源码位置: langgraph/graph/state.py:L257-287

def _add_schema(self, schema: type[Any], /) -> None:
    """将 State Schema 转换为 Channels"""
    
    if schema not in self.schemas:
        # ===== 1️⃣ 调用 _get_channels 解析 State =====
        channels, managed, type_hints = _get_channels(schema)
        # 对于我们的例子:
        # channels = {"text": LastValue(str)}
        # managed = {}
        # type_hints = {"text": str}
        
        # ===== 2️⃣ 保存到 self.schemas =====
        self.schemas[schema] = {**channels, **managed}
        # self.schemas[State] = {"text": LastValue(str)}
        
        # ===== 3️⃣ 注册到 self.channels =====
        for key, channel in channels.items():
            if key in self.channels:
                # 检查冲突 (如果已存在同名 channel)
                if self.channels[key] != channel:
                    if isinstance(channel, LastValue):
                        pass  # LastValue 可以共存
                    else:
                        raise ValueError(f"Channel '{key}' already exists")
            else:
                self.channels[key] = channel
        
        # 结果: self.channels = {"text": LastValue(str)}
```

#### 2.2.2 `_get_channels` 做了什么?

```python
# 源码位置: langgraph/graph/state.py:L1334-1354

def _get_channels(schema: type[dict]):
    """从 State Schema 提取 Channels"""
    
    # ===== 1. 检查是否有类型注解 =====
    if not hasattr(schema, "__annotations__"):
        # 没有注解 → 用 __root__ channel
        return ({"__root__": _get_channel("__root__", schema)}, {}, {})
    
    # ===== 2. 获取所有字段的类型注解 =====
    type_hints = get_type_hints(schema, include_extras=True)
    # 对于我们的例子:
    # type_hints = {"text": <class 'str'>}
    
    # ===== 3. 为每个字段创建 Channel =====
    all_keys = {
        name: _get_channel(name, typ)
        for name, typ in type_hints.items()
        if name != "__slots__"
    }
    # all_keys = {"text": _get_channel("text", str)}
    
    # ===== 4. 分类为 Channels 和 ManagedValues =====
    return (
        {k: v for k, v in all_keys.items() if isinstance(v, BaseChannel)},
        {k: v for k, v in all_keys.items() if is_managed_value(v)},
        type_hints,
    )
    # 返回:
    # channels = {"text": LastValue(str)}
    # managed = {}
    # type_hints = {"text": str}
```

#### 2.2.3 `_get_channel("text", str)` 做了什么?

```python
# 源码位置: langgraph/graph/state.py:L1369-1392

def _get_channel(name: str, annotation: Any) -> BaseChannel | ManagedValueSpec:
    """根据类型注解创建 Channel"""
    
    # ===== 1. 检查是否是 Managed Value (如 Context Manager) =====
    if manager := _is_field_managed_value(name, annotation):
        return manager
    # 对于 str,不是 Managed Value,跳过
    
    # ===== 2. 检查是否是自定义 Channel =====
    elif channel := _is_field_channel(annotation):
        return channel
    # 对于 str,不是自定义 Channel,跳过
    
    # ===== 3. 检查是否是 Annotated[type, reducer] =====
    elif channel := _is_field_binop(annotation):
        return channel
    # 对于 str,不是 Annotated,跳过
    
    # ===== 4. 默认: 创建 LastValue Channel =====
    fallback: LastValue = LastValue(annotation)
    fallback.key = name
    return fallback
    # 返回: LastValue(str, key="text")
```

---

### 2.3 总结: State → Channels 的转换

```text
用户代码:
  class State(TypedDict):
      text: str
      
            ↓
            
底层转换 (_add_schema):
  1. 解析类型注解: {"text": str}
  2. 为每个字段创建 Channel:
     - "text": str → LastValue(str, key="text")
  3. 注册到 self.channels:
     - self.channels = {"text": LastValue(str)}

结果:
  graph.channels = {
      "text": LastValue(str, key="text")
  }
```

---

## 三、第 2 步: add_node - 节点函数到 PregelNode 的转换

### 3.1 用户视角: 添加节点

```python
graph.add_node("node_a", node_a)
graph.add_node("node_b", node_b)
```

**用户的意图**: 添加两个节点,分别执行 `node_a` 和 `node_b` 函数。

---

### 3.2 底层发生了什么?

```python
# 源码位置: langgraph/graph/state.py:L359-572

def add_node(self, node: str, action: Callable, ...) -> Self:
    """添加节点到 StateGraph"""
    
    # ===== 1. 提取节点名 =====
    # 已经提供了 "node_a",跳过
    
    # ===== 2. 验证节点名 =====
    if node in self.nodes:
        raise ValueError(f"Node `{node}` already present.")
    # "node_a" 不在 self.nodes 中,验证通过
    
    # ===== 3. 推断 input_schema (从函数签名) =====
    inferred_input_schema = None
    
    if isfunction(action):
        hints = get_type_hints(action)
        # hints = {"state": State, "return": dict}
        
        # 获取第一个参数名
        first_parameter_name = next(iter(inspect.signature(action).parameters.keys()))
        # first_parameter_name = "state"
        
        if input_hint := hints.get(first_parameter_name):
            # input_hint = State
            if isinstance(input_hint, type) and get_type_hints(input_hint):
                inferred_input_schema = input_hint
                # inferred_input_schema = State
    
    # ===== 4. 创建 StateNodeSpec =====
    self.nodes[node] = StateNodeSpec[StateT, ContextT](
        coerce_to_runnable(action, name=node, trace=False),  # 将函数转换为 Runnable
        metadata=None,
        input_schema=inferred_input_schema or self.state_schema,  # State
        retry_policy=None,
        cache_policy=None,
        ends=EMPTY_SEQ,
        defer=False,
    )
    
    # ===== 5. 注册 input_schema 的 Channels =====
    if inferred_input_schema is not None:
        self._add_schema(inferred_input_schema)
        # 因为 inferred_input_schema = State,
        # 而 State 已经在 __init__ 时注册过了,
        # 这里不会重复注册
    
    return self
```

#### 3.2.1 `coerce_to_runnable` 做了什么?

```python
# 源码位置: langgraph/_internal/_runnable.py

def coerce_to_runnable(thing: Any, name: str | None, trace: bool) -> Runnable:
    """将函数/Runnable 转换为标准的 Runnable"""
    
    if isinstance(thing, Runnable):
        return thing
    
    # 对于普通函数 node_a
    if callable(thing):
        # 包装为 RunnableLambda
        return RunnableLambda(thing, name=name or thing.__name__)
        # 返回: RunnableLambda(node_a, name="node_a")
```

---

### 3.3 总结: add_node 的结果

```text
用户代码:
  graph.add_node("node_a", node_a)
  
            ↓
            
底层转换:
  1. 推断 input_schema: State (从函数签名 state: State)
  2. 将函数转换为 Runnable: RunnableLambda(node_a)
  3. 创建 StateNodeSpec:
     {
       runnable: RunnableLambda(node_a),
       input_schema: State,
       metadata: None,
       retry_policy: None,
       ...
     }
  4. 存储到 self.nodes:
     self.nodes["node_a"] = StateNodeSpec(...)

结果:
  graph.nodes = {
      "node_a": StateNodeSpec(runnable=RunnableLambda(node_a), input_schema=State, ...),
      "node_b": StateNodeSpec(runnable=RunnableLambda(node_b), input_schema=State, ...),
  }
```

---

## 四、第 3 步: add_edge - 边到 Channel 写入的转换

### 4.1 用户视角: 添加边

```python
graph.add_edge(START, "node_a")
graph.add_edge("node_a", "node_b")
```

**用户的意图**: 
- START → node_a: 图的入口
- node_a → node_b: node_a 执行完后执行 node_b

---

### 4.2 底层发生了什么?

```python
# 源码位置: langgraph/graph/state.py:L574-626

def add_edge(self, start_key: str, end_key: str) -> Self:
    """添加固定边"""
    
    # ===== 验证 =====
    if start_key == END:
        raise ValueError("END cannot be a start node")
    if end_key == START:
        raise ValueError("START cannot be an end node")
    
    # ===== 存储边 =====
    self.edges.add((start_key, end_key))
    
    return self
```

**注意**: `add_edge` 在**构建阶段**只是存储边的信息,**不创建任何底层对象**。真正的转换发生在 `compile()` 时。

---

### 4.3 总结: add_edge 的结果

```text
用户代码:
  graph.add_edge(START, "node_a")
  graph.add_edge("node_a", "node_b")
  
            ↓
            
底层转换:
  存储到 self.edges:
  
结果:
  graph.edges = {
      (START, "node_a"),
      ("node_a", "node_b"),
  }
```

---

## 五、第 4 步: compile() - 编译为可执行的 Pregel

### 5.1 用户视角: 编译图

```python
app = graph.compile()
```

**用户的意图**: 将图编译为可执行的对象。

---

### 5.2 底层发生了什么?

```python
# 源码位置: langgraph/graph/state.py:L824-929

def compile(self, ...) -> CompiledStateGraph:
    """编译 StateGraph 为可执行的 CompiledStateGraph"""
    
    # ===== 1. 验证图结构 =====
    self.validate()
    
    # ===== 2. 创建 CompiledStateGraph (继承自 Pregel) =====
    compiled = CompiledStateGraph(
        builder=self,
        nodes={},  # ⚠️ 空的,稍后填充
        channels={
            **self.channels,  # ⚠️ 复制用户的 channels
            START: EphemeralValue(self.input_schema),  # ⚠️ START 也是一个 channel
        },
        input_channels=START,
        output_channels="text",  # 从 output_schema 推断
        checkpointer=None,
        ...
    )
    
    # ===== 3. Attach 节点 (转换为 PregelNode) =====
    compiled.attach_node(START, None)  # 特殊处理 START
    for key, node in self.nodes.items():
        compiled.attach_node(key, node)
    
    # ===== 4. Attach 边 (转换为 Channel 写入) =====
    for start, end in self.edges:
        compiled.attach_edge(start, end)
    
    return compiled
```

---

### 5.3 `attach_node` 的详细实现

这是**最关键**的步骤,将 `StateNodeSpec` 转换为 `PregelNode`。

```python
# 源码位置: langgraph/graph/state.py:L970-1071

def attach_node(self, key: str, node: StateNodeSpec | None) -> None:
    """将 StateNodeSpec 转换为 PregelNode"""
    
    # ===== 场景1: START 节点 =====
    if key == START:
        output_keys = ["text"]  # 从 input_schema 提取
        
        write_entries = (
            ChannelWriteTupleEntry(mapper=_get_updates),
            ...
        )
        
        self.nodes[START] = PregelNode(
            tags=[TAG_HIDDEN],
            triggers=[START],  # ⚠️ 订阅 START channel
            channels=START,
            writers=[ChannelWrite(write_entries)],  # ⚠️ 写入到 state channels
        )
        return
    
    # ===== 场景2: node_a / node_b 节点 =====
    # 1. 确定输入 channels
    input_schema = node.input_schema  # State
    input_channels = list(self.builder.schemas[input_schema])
    # input_channels = ["text"]
    
    is_single_input = len(input_channels) == 1 and "__root__" in input_channels
    # is_single_input = False
    
    # 2. 创建 mapper (State dict → Pydantic Model)
    if input_schema in self.schema_to_mapper:
        mapper = self.schema_to_mapper[input_schema]
    else:
        mapper = _pick_mapper(input_channels, input_schema)
        # 对于 TypedDict,mapper = None (不需要转换)
        # 对于 Pydantic Model,mapper = lambda x: State(**x)
        self.schema_to_mapper[input_schema] = mapper
    
    # 3. 创建 trigger channel (用于接收边的触发信号)
    branch_channel = _CHANNEL_BRANCH_TO.format(key)
    # branch_channel = "branch:to:node_a"
    
    self.channels[branch_channel] = EphemeralValue(Any, guard=False)
    # self.channels["branch:to:node_a"] = EphemeralValue(Any)
    
    # 4. 创建 state updater (节点输出 → Channels)
    output_keys = ["text"]  # 所有 state channels
    
    write_entries = (
        ChannelWriteTupleEntry(
            mapper=_get_updates  # 将节点输出转换为 (channel, value) 元组
        ),
        ChannelWriteTupleEntry(
            mapper=_control_branch,  # 处理 Command 对象
            static=None,
        ),
    )
    
    # 5. 创建 PregelNode
    self.nodes[key] = PregelNode(
        triggers=["branch:to:node_a"],  # ⚠️ 订阅 trigger channel
        channels=["text"],               # ⚠️ 读取的 state channels
        mapper=None,                     # ⚠️ State dict → Pydantic Model (TypedDict 不需要)
        writers=[ChannelWrite(write_entries)],  # ⚠️ 写入到 state channels
        bound=node.runnable,             # ⚠️ RunnableLambda(node_a)
        metadata=None,
        retry_policy=[],
        cache_policy=None,
        tags=[],
    )
```

---

### 5.4 `attach_edge` 的详细实现

```python
# 源码位置: langgraph/graph/state.py:L1073-1097

def attach_edge(self, start: str, end: str) -> None:
    """将边转换为 Channel 写入"""
    
    if end != END:
        # ===== 为 start 节点添加写入操作 =====
        self.nodes[start].writers.append(
            ChannelWrite(
                (ChannelWriteEntry(_CHANNEL_BRANCH_TO.format(end), None),)
            )
        )
        # 效果:
        # - START 节点添加写入操作: 写入到 "branch:to:node_a" channel
        # - node_a 节点添加写入操作: 写入到 "branch:to:node_b" channel
```

---

### 5.5 编译后的完整结构

```python
# ===== CompiledStateGraph 的结构 =====

compiled.channels = {
    # ===== 用户定义的 state channels =====
    "text": LastValue(str, key="text"),
    
    # ===== 特殊 channels =====
    START: EphemeralValue(State),  # 输入 channel
    
    # ===== 触发 channels (由 attach_node 创建) =====
    "branch:to:node_a": EphemeralValue(Any),
    "branch:to:node_b": EphemeralValue(Any),
}

compiled.nodes = {
    # ===== START 节点 =====
    START: PregelNode(
        triggers=[START],              # 订阅 START channel
        channels=START,
        writers=[
            ChannelWrite([
                ChannelWriteTupleEntry(mapper=_get_updates),  # 写入 state channels
                ChannelWriteEntry("branch:to:node_a", None),  # 触发 node_a
            ])
        ],
    ),
    
    # ===== node_a 节点 =====
    "node_a": PregelNode(
        triggers=["branch:to:node_a"],  # 订阅 "branch:to:node_a" channel
        channels=["text"],               # 读取 "text" channel
        mapper=None,
        writers=[
            ChannelWrite([
                ChannelWriteTupleEntry(mapper=_get_updates),  # 写入 state channels
                ChannelWriteEntry("branch:to:node_b", None),  # 触发 node_b
            ])
        ],
        bound=RunnableLambda(node_a),
    ),
    
    # ===== node_b 节点 =====
    "node_b": PregelNode(
        triggers=["branch:to:node_b"],  # 订阅 "branch:to:node_b" channel
        channels=["text"],               # 读取 "text" channel
        mapper=None,
        writers=[
            ChannelWrite([
                ChannelWriteTupleEntry(mapper=_get_updates),  # 写入 state channels
                # ⚠️ 没有触发下一个节点 (node_b 是最后一个)
            ])
        ],
        bound=RunnableLambda(node_b),
    ),
}
```

---

### 5.6 可视化: 编译后的 Channels 和 PregelNodes

```text
用户视角:
  START → node_a → node_b

底层视角 (Channels + PregelNodes):

Channels:
  ┌─────────────────┐
  │  START channel  │  ← 输入从这里进入
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ "text" channel  │  ← 状态存储在这里
  │ LastValue(str)  │
  └─────────────────┘
  
  ┌───────────────────────┐
  │ "branch:to:node_a"    │  ← 触发 node_a
  │ EphemeralValue(Any)   │
  └───────────────────────┘
  
  ┌───────────────────────┐
  │ "branch:to:node_b"    │  ← 触发 node_b
  │ EphemeralValue(Any)   │
  └───────────────────────┘

PregelNodes:
  ┌─────────────────────┐
  │   START 节点        │
  │  订阅: START        │
  │  执行: 无           │
  │  写入:              │
  │    - "text"         │ ← 将输入写入 state channel
  │    - "branch:to:    │ ← 触发 node_a
  │      node_a"        │
  └─────────────────────┘
           │
           ▼
  ┌─────────────────────┐
  │  node_a 节点        │
  │  订阅: "branch:to:  │
  │        node_a"      │
  │  读取: "text"       │ ← 读取 state channel
  │  执行: node_a()     │
  │  写入:              │
  │    - "text"         │ ← 更新 state channel
  │    - "branch:to:    │ ← 触发 node_b
  │      node_b"        │
  └─────────────────────┘
           │
           ▼
  ┌─────────────────────┐
  │  node_b 节点        │
  │  订阅: "branch:to:  │
  │        node_b"      │
  │  读取: "text"       │ ← 读取 state channel
  │  执行: node_b()     │
  │  写入:              │
  │    - "text"         │ ← 更新 state channel
  └─────────────────────┘
```

---

## 六、第 5 步: invoke() - 执行流程

### 6.1 用户视角: 执行图

```python
result = app.invoke({"text": ""})
```

**用户的意图**: 传入初始状态 `{"text": ""}`,执行图,获取最终结果。

---

### 6.2 底层发生了什么?

```python
# 源码位置: langgraph/pregel/main.py (简化版)

def invoke(self, input: dict, config: RunnableConfig = None):
    """执行 Pregel 图"""
    
    # ===== 1. 初始化所有 Channels =====
    channels = {
        "text": LastValue(str),
        START: EphemeralValue(State),
        "branch:to:node_a": EphemeralValue(Any),
        "branch:to:node_b": EphemeralValue(Any),
    }
    # 所有 channels 都是空的
    
    # ===== 2. 写入输入到 START channel =====
    channels[START].update([{"text": ""}])
    # channels[START] = {"text": ""}
    
    # ===== Superstep 0: START 节点 =====
    # Plan: START channel 有值 → 触发 START 节点
    # Execute:
    #   - 读取: channels[START] → {"text": ""}
    #   - 执行: 无 (START 节点不执行任何操作)
    #   - 写入:
    #     - _get_updates({"text": ""}) → [("text", "")]
    #     - channels["text"].update([""]) → channels["text"] = ""
    #     - channels["branch:to:node_a"].update([None]) → 触发 node_a
    # Update: 提交所有写入
    
    # ===== Superstep 1: node_a 节点 =====
    # Plan: "branch:to:node_a" channel 有值 → 触发 node_a 节点
    # Execute:
    #   - 读取: channels["text"] → ""
    #   - 构建输入: {"text": ""}
    #   - 执行: node_a({"text": ""}) → {"text": "a"}
    #   - 写入:
    #     - _get_updates({"text": "a"}) → [("text", "a")]
    #     - channels["text"].update(["a"]) → channels["text"] = "a"
    #     - channels["branch:to:node_b"].update([None]) → 触发 node_b
    # Update: 提交所有写入
    
    # ===== Superstep 2: node_b 节点 =====
    # Plan: "branch:to:node_b" channel 有值 → 触发 node_b 节点
    # Execute:
    #   - 读取: channels["text"] → "a"
    #   - 构建输入: {"text": "a"}
    #   - 执行: node_b({"text": "a"}) → {"text": "ab"}
    #   - 写入:
    #     - _get_updates({"text": "ab"}) → [("text", "ab")]
    #     - channels["text"].update(["ab"]) → channels["text"] = "ab"
    #     - 没有触发下一个节点 (node_b 是最后一个)
    # Update: 提交所有写入
    
    # ===== Superstep 3: 无节点被触发 =====
    # Plan: 所有 trigger channels 都为空 → 没有节点被触发
    # 执行结束
    
    # ===== 3. 从 output_channels 读取最终结果 =====
    result = {
        "text": channels["text"].get()  # "ab"
    }
    
    return result
```

---

### 6.3 逐步可视化执行流程

#### Superstep 0: START 节点

```text
Channels 状态 (执行前):
  START: {"text": ""}   ← 输入
  text: <empty>
  branch:to:node_a: <empty>
  branch:to:node_b: <empty>

执行 START 节点:
  读取: START channel → {"text": ""}
  写入:
    - text channel ← ""
    - branch:to:node_a channel ← None (触发)

Channels 状态 (执行后):
  START: {"text": ""}
  text: ""              ← 更新
  branch:to:node_a: None ← 触发 node_a
  branch:to:node_b: <empty>
```

#### Superstep 1: node_a 节点

```text
Channels 状态 (执行前):
  START: {"text": ""}
  text: ""
  branch:to:node_a: None  ← 触发 node_a
  branch:to:node_b: <empty>

执行 node_a 节点:
  订阅: branch:to:node_a channel 有值 → 触发
  读取: text channel → ""
  构建输入: {"text": ""}
  执行: node_a({"text": ""}) → {"text": "a"}
  写入:
    - text channel ← "a"
    - branch:to:node_b channel ← None (触发)

Channels 状态 (执行后):
  START: {"text": ""}
  text: "a"             ← 更新
  branch:to:node_a: <empty> ← 已消费
  branch:to:node_b: None ← 触发 node_b
```

#### Superstep 2: node_b 节点

```text
Channels 状态 (执行前):
  START: {"text": ""}
  text: "a"
  branch:to:node_a: <empty>
  branch:to:node_b: None  ← 触发 node_b

执行 node_b 节点:
  订阅: branch:to:node_b channel 有值 → 触发
  读取: text channel → "a"
  构建输入: {"text": "a"}
  执行: node_b({"text": "a"}) → {"text": "ab"}
  写入:
    - text channel ← "ab"
    - (没有触发下一个节点)

Channels 状态 (执行后):
  START: {"text": ""}
  text: "ab"            ← 更新
  branch:to:node_a: <empty>
  branch:to:node_b: <empty> ← 已消费
```

#### Superstep 3: 结束

```text
Channels 状态:
  START: {"text": ""}
  text: "ab"
  branch:to:node_a: <empty>
  branch:to:node_b: <empty>

Plan: 所有 trigger channels 都为空 → 没有节点被触发
执行结束

读取 output_channels:
  result = {"text": channels["text"].get()} = {"text": "ab"}
```

---

## 七、关键概念总结

### 7.1 Channels (通道)

**作用**: 数据的存储和传输管道。

**类型**:
1. **State Channels**: 存储用户定义的状态字段
   - 示例: `"text"` channel (LastValue)
   
2. **Trigger Channels**: 用于触发节点执行
   - 示例: `"branch:to:node_a"` channel (EphemeralValue)
   
3. **Special Channels**: 特殊用途
   - 示例: `START` channel (EphemeralValue) - 输入入口

**特点**:
- **LastValue**: 保存最后一个值,新值覆盖旧值
- **EphemeralValue**: 临时值,消费后清空

---

### 7.2 PregelNode (执行单元)

**作用**: Actor 模型的节点,订阅 Channels,执行逻辑,写入 Channels。

**结构**:
```python
PregelNode(
    triggers=["branch:to:node_a"],  # 订阅哪些 channels (触发条件)
    channels=["text"],               # 读取哪些 channels (输入)
    mapper=None,                     # 数据转换 (State dict → Pydantic Model)
    writers=[ChannelWrite(...)],     # 写入哪些 channels (输出)
    bound=RunnableLambda(node_a),    # 实际执行的逻辑
)
```

**执行流程**:
1. **订阅**: 监听 `triggers` 中的 channels
2. **触发**: 当任意一个 trigger channel 有值时,节点被触发
3. **读取**: 从 `channels` 中读取数据,构建输入
4. **执行**: 运行 `bound` (Runnable)
5. **写入**: 将输出通过 `writers` 写入 channels

---

### 7.3 ChannelWrite (写入操作)

**作用**: 将节点输出写入 Channels。

**结构**:
```python
ChannelWrite([
    ChannelWriteTupleEntry(mapper=_get_updates),  # 写入 state channels
    ChannelWriteEntry("branch:to:node_b", None),  # 触发下一个节点
])
```

**_get_updates 函数**:
```python
def _get_updates(input: dict) -> list[tuple[str, Any]]:
    """将节点输出转换为 (channel, value) 元组"""
    if isinstance(input, dict):
        return [(k, v) for k, v in input.items() if k in output_keys]
    # 示例: {"text": "a"} → [("text", "a")]
```

---

### 7.4 State Schema (状态模式)

**作用**: 定义图的状态结构。

**用户定义**:
```python
class State(TypedDict):
    text: str
```

**底层转换**:
```python
channels = {
    "text": LastValue(str, key="text")
}
```

**节点使用**:
```python
# 节点输入
input = {"text": channels["text"].get()}

# 节点输出
output = {"text": "new_value"}

# 写入 channels
channels["text"].update(["new_value"])
```

---

## 八、完整流程总结图

```text
用户代码                  底层转换                      执行流程
═══════════════════════════════════════════════════════════════════

1️⃣ 定义 State             
class State(TypedDict):    → _add_schema()            
    text: str             → _get_channels()
                          → channels = {
                              "text": LastValue(str)
                            }

2️⃣ 添加节点
graph.add_node(           → coerce_to_runnable()
  "node_a", node_a)       → StateNodeSpec(
                              runnable=RunnableLambda(node_a),
                              input_schema=State,
                            )

3️⃣ 添加边
graph.add_edge(           → self.edges.add(
  START, "node_a")          (START, "node_a"))

4️⃣ 编译
app = graph.compile()     → attach_node()             
                          → PregelNode(
                              triggers=["branch:to:node_a"],
                              channels=["text"],
                              writers=[ChannelWrite(...)],
                              bound=RunnableLambda(node_a),
                            )
                          
                          → attach_edge()
                          → ChannelWrite([
                              ChannelWriteEntry(
                                "branch:to:node_b", None)
                            ])

5️⃣ 执行
result = app.invoke(      → Superstep 0:              START channel 有值
  {"text": ""})             - 触发 START 节点          → 写入 "text" = ""
                            - 写入 branch:to:node_a   → 触发 node_a
                          
                          → Superstep 1:              branch:to:node_a 有值
                            - 触发 node_a 节点         → 读取 "text" = ""
                            - 执行 node_a()           → 写入 "text" = "a"
                            - 写入 branch:to:node_b   → 触发 node_b
                          
                          → Superstep 2:              branch:to:node_b 有值
                            - 触发 node_b 节点         → 读取 "text" = "a"
                            - 执行 node_b()           → 写入 "text" = "ab"
                          
                          → Superstep 3:              无节点触发
                            - 执行结束
                          
                          → 返回:
                            {"text": "ab"}
```

---

## 九、Linus 式评价

### ✅ 设计精妙之处

**1. Channel 是数据流的本质抽象**
> "State 只是给用户的语法糖,真正的数据流是 Channels。用户定义 `text: str`,底层是 `LastValue(str)` channel。这个抽象很干净。"

**2. Trigger Channels 实现控制流**
> "用 `branch:to:node_a` channel 来触发节点,而不是硬编码 if/else,这是好品味。节点订阅 channel,channel 有值就触发,简单但强大。"

**3. PregelNode 是 Actor 模型的正确实现**
> "订阅 → 读取 → 执行 → 写入,这是 Actor 模型的标准流程。LangGraph 的实现很忠实于 Pregel 论文。"

### ❌ 可以改进的地方

**1. 用户看不到 Channels**
> "用户定义 State,但不知道底层有 Channels。这个抽象泄漏是无法避免的,但文档应该更清楚。"

**2. Trigger Channels 的命名**
> "`branch:to:node_a` 这个命名太长了。应该简化为 `trigger:node_a` 或者 `>node_a`。"

**3. attach_node 太复杂**
> "100 行代码做太多事情。应该拆成 `create_trigger_channel()` + `create_state_reader()` + `create_state_writer()` 三个函数。"

---

**最终结论:**

> "理解了 Channels → PregelNode → ChannelWrite 的转换,你就理解了 LangGraph 的 90%。State 是用户接口,Channels 是数据流,PregelNode 是执行单元,ChannelWrite 是输出。别被抽象迷惑,看源码才是王道。"
