# LangGraph Checkpoint 深度解析

> **核心思想**: Checkpoint 就是 LangGraph 的 Git - 为 AI Agent 的状态提供版本控制和时间旅行能力

---

## 一、Checkpoint 是什么?

### 1.1 类比理解

```text
Git                    vs     LangGraph Checkpoint
────────────────────        ────────────────────────
commit ID                   checkpoint_id (UUID v6)
commit message              CheckpointMetadata
file contents               channel_values (状态快照)
branch/tag                  thread_id (对话线程)
git log                     checkpointer.list()
git checkout <commit>       agent.get_state(checkpoint_id=...)
git revert                  agent.update_state()
```

### 1.2 真实问题

**没有 Checkpoint 的 Agent:**
```python
# 每次调用都是新对话,完全失忆
agent.invoke("帮我分析这个 bug")  # ✅ 回复正常
agent.invoke("刚才分析的结论是什么?") # ❌ "我没有上下文"
```

**有 Checkpoint 的 Agent:**
```python
config = {"configurable": {"thread_id": "debug-session-123"}}
agent.invoke("帮我分析这个 bug", config)  # ✅ 保存到 checkpoint
agent.invoke("刚才分析的结论是什么?", config) # ✅ "根据之前的分析..."
```

---

## 二、为什么需要 Checkpoint?

### 2.1 核心价值 (Linus 式判断)

**✅ 真实问题:**
1. **多轮对话记忆** - Agent 需要记住之前说了什么
2. **人机协作中断** - 用户审批工具调用后需要恢复执行
3. **故障恢复** - 进程崩溃后能从上次状态继续
4. **状态审计** - 追踪 Agent 每一步的决策过程

**❌ 过度设计的部分:**
- Metadata 的 JSON 嵌套查询 (SQLite 不适合)
- 同步/异步双重实现 (AsyncSqliteSaver 包装 SqliteSaver)

---

### 2.2 在 LangGraph 执行循环中的作用

```python
# LangGraph Pregel 执行循环的 Checkpoint 插入点
while not is_done():
    # 1️⃣ 加载上次 checkpoint (如果存在)
    checkpoint = checkpointer.get_tuple(config)
    
    # 2️⃣ 执行一个 superstep (所有并行节点)
    state = execute_superstep(checkpoint)
    
    # 3️⃣ 保存新 checkpoint
    new_checkpoint = checkpointer.put(
        config, 
        checkpoint=state.snapshot,
        metadata={"step": step_num, "source": "loop"}
    )
    
    # 4️⃣ 检查中断 (Human-in-the-Loop)
    if has_interrupt(state):
        # 保存中断状态,等待 resume
        checkpointer.put_writes(
            config,
            writes=[(INTERRUPT, tool_call_data)],
            task_id="hitl"
        )
        break  # 退出循环,等待用户决策
```

**关键机制:**
- `superstep` = LangGraph 一次执行所有可并行的节点
- 每个 superstep 后都生成一个 checkpoint
- `checkpoint_id` 使用 UUID v6 → 可排序 (时间递增)

---

### 2.3 Human-in-the-Loop 的中断与恢复

```python
# ===== 第一阶段: 执行到中断点 =====
result = agent.invoke(
    {"messages": [HumanMessage("删除 /重要文件.txt")]},
    config={"configurable": {"thread_id": "session-1"}}
)

# 检测到需要用户批准的工具调用
assert result["__interrupt__"] is not None
interrupt = result["__interrupt__"][0]
# interrupt.value = {
#     "action_requests": [{
#         "name": "write_file",
#         "args": {"path": "/重要文件.txt", "content": None}  # content=None 表示删除
#     }],
#     "review_configs": [{
#         "allowed_decisions": ["approve", "reject"]
#     }]
# }

# ===== 第二阶段: 用户审批后恢复 =====
from langgraph.types import Command

# 用户批准操作
result = agent.invoke(
    Command(resume={"decisions": [{"type": "approve"}]}),
    config={"configurable": {"thread_id": "session-1"}}  # ⚠️ 必须用同一个 thread_id
)

# Agent 从中断点继续执行,完成文件删除
```

**实现原理:**
```python
# deepagents_cli/execution.py 的简化版
while True:
    async for chunk in agent.astream(..., config=config):
        if chunk["__interrupt__"]:
            # 检测到中断 → 保存当前 checkpoint
            pending_interrupts = chunk["__interrupt__"]
            # 提示用户审批
            decisions = prompt_user_for_approval(pending_interrupts)
            # 用 Command.resume 恢复执行
            stream_input = Command(resume={"decisions": decisions})
            continue  # 重新进入循环
        else:
            break  # 正常完成
```

---

### 2.4 时间旅行 (Time Travel)

```python
# 查看所有历史 checkpoint
history = list(checkpointer.list(config))
for checkpoint_tuple in history:
    print(f"Checkpoint ID: {checkpoint_tuple.checkpoint['id']}")
    print(f"Step: {checkpoint_tuple.metadata['step']}")
    print(f"Messages: {checkpoint_tuple.checkpoint['channel_values']['messages']}")

# 回到第 3 步的状态
third_checkpoint_id = history[2].checkpoint["id"]
state_at_step_3 = agent.get_state(
    config={"configurable": {
        "thread_id": "session-1",
        "checkpoint_id": third_checkpoint_id
    }}
)

# 从第 3 步重新执行 (创建新分支)
agent.invoke(
    {"messages": [HumanMessage("换个思路试试")]},
    config={"configurable": {
        "thread_id": "session-1",
        "checkpoint_id": third_checkpoint_id
    }}
)
```

---

## 三、Checkpoint 怎么实现的?

### 3.1 核心数据结构

```python
# langgraph/checkpoint/base/__init__.py

class Checkpoint(TypedDict):
    """状态快照 - 类似 Git commit"""
    v: int                          # 格式版本号 (当前是 2)
    id: str                         # UUID v6 (可排序!)
    ts: str                         # ISO 8601 时间戳
    
    # ===== 核心状态 =====
    channel_values: dict[str, Any]  # 所有 channel 的当前值
    # 示例: {
    #   "messages": [HumanMessage(...), AIMessage(...)],
    #   "todos": [{"content": "Fix bug", "status": "completed"}],
    #   "files": {"/app.py": FileData(...)}
    # }
    
    # ===== 版本跟踪 =====
    channel_versions: dict[str, str | int | float]  
    # 示例: {"messages": "v5", "todos": "v2"}
    # 用于检测哪些 channel 发生了变化
    
    versions_seen: dict[str, dict[str, str]]
    # 示例: {
    #   "model_node": {"messages": "v4", "todos": "v1"},
    #   "tools_node": {"messages": "v5"}
    # }
    # 用于确定哪些节点需要重新执行
    
    updated_channels: list[str] | None  # 本次更新的 channel

class CheckpointMetadata(TypedDict, total=False):
    """Checkpoint 的元数据 - 类似 Git commit message"""
    source: Literal["input", "loop", "update", "fork"]
    # input: 用户输入触发
    # loop: Pregel 循环中自动保存
    # update: 手动调用 update_state
    # fork: 基于旧 checkpoint 创建新分支
    
    step: int  # 步骤编号 (-1=初始输入, 0=第一个 loop, ...)
    parents: dict[str, str]  # 父 checkpoint ID (支持多分支)
```

**设计评价 (Linus 视角):**

✅ **好品味:**
- 用 `uuid6` 而非 `uuid4` → ID 天然有序,不需要额外的序列号
- `channel_values` 是纯数据字典 → 序列化简单,适合存数据库
- `versions_seen` 追踪依赖 → 实现了 "脏检查" (dirty checking)

❌ **可以改进:**
- `channel_versions` 混用 `str | int | float` → 类型不统一,容易出错
- `CheckpointMetadata` 用 `TypedDict` 而非 Pydantic → 缺少运行时校验
- `parents` 设计支持多分支,但实际代码未充分利用

---

### 3.2 BaseCheckpointSaver 接口

```python
class BaseCheckpointSaver(Generic[V]):
    """Checkpoint 存储的抽象基类
    
    V = 版本号类型 (int | float | str)
    """
    
    # ===== 必须实现的核心方法 =====
    
    def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None:
        """获取指定 checkpoint
        
        如果 config 包含 checkpoint_id → 精确查询
        否则 → 返回该 thread_id 的最新 checkpoint
        """
        raise NotImplementedError
    
    def list(
        self,
        config: RunnableConfig | None,
        *,
        filter: dict[str, Any] | None = None,  # Metadata 过滤
        before: RunnableConfig | None = None,  # 只返回此 checkpoint 之前的
        limit: int | None = None
    ) -> Iterator[CheckpointTuple]:
        """列出所有 checkpoint (按时间倒序)"""
        raise NotImplementedError
    
    def put(
        self,
        config: RunnableConfig,
        checkpoint: Checkpoint,
        metadata: CheckpointMetadata,
        new_versions: ChannelVersions
    ) -> RunnableConfig:
        """保存 checkpoint → 返回更新后的 config (包含新 checkpoint_id)"""
        raise NotImplementedError
    
    def put_writes(
        self,
        config: RunnableConfig,
        writes: Sequence[tuple[str, Any]],  # (channel, value)
        task_id: str,
        task_path: str = ""
    ) -> None:
        """保存中间写入 (如 HITL 的 pending writes)"""
        raise NotImplementedError
    
    # ===== 可选的异步方法 =====
    async def aget_tuple(...): ...
    async def alist(...): ...
    async def aput(...): ...
```

**关键设计:**
- `get_tuple` 返回的是 `CheckpointTuple` (包含 config + checkpoint + metadata)
- `put` 方法返回新的 `config` → 包含新生成的 `checkpoint_id`
- `put_writes` 专门用于保存 "待处理的写入" (如 HITL 中断时的 tool call)

---

### 3.3 实现对比: Memory vs SQLite

#### 3.3.1 InMemorySaver (开发/测试用)

```python
# langgraph/checkpoint/memory/__init__.py

class InMemorySaver(BaseCheckpointSaver[str]):
    """纯内存存储 - 进程重启后数据丢失"""
    
    # 三层嵌套的存储结构
    storage: defaultdict[
        str,  # thread_id
        dict[
            str,  # checkpoint_ns (namespace)
            dict[
                str,  # checkpoint_id
                tuple[
                    tuple[str, bytes],  # (type, serialized_checkpoint)
                    tuple[str, bytes],  # (type, serialized_metadata)
                    str | None          # parent_checkpoint_id
                ]
            ]
        ]
    ]
    
    # 中间写入 (HITL)
    writes: defaultdict[
        tuple[str, str, str],  # (thread_id, checkpoint_ns, checkpoint_id)
        dict[
            tuple[str, int],  # (task_id, write_idx)
            tuple[str, str, tuple[str, bytes], str]  # (channel, type, value, task_path)
        ]
    ]
    
    def put(self, config, checkpoint, metadata, new_versions):
        thread_id = config["configurable"]["thread_id"]
        checkpoint_ns = config["configurable"].get("checkpoint_ns", "")
        checkpoint_id = checkpoint["id"]
        
        # 序列化
        type_cp, serialized_cp = self.serde.dumps_typed(checkpoint)
        type_md, serialized_md = self.serde.dumps_typed(metadata)
        
        # 存入嵌套字典
        self.storage[thread_id][checkpoint_ns][checkpoint_id] = (
            (type_cp, serialized_cp),
            (type_md, serialized_md),
            config["configurable"].get("checkpoint_id")  # parent_id
        )
        
        return {
            "configurable": {
                "thread_id": thread_id,
                "checkpoint_ns": checkpoint_ns,
                "checkpoint_id": checkpoint_id
            }
        }
```

**优点:**
- 零配置,直接用
- 适合单元测试

**缺点:**
- 数据易失 (进程重启丢失)
- 不支持多进程/分布式
- 无法审计历史记录

---

#### 3.3.2 SqliteSaver (轻量级持久化)

```python
# langgraph/checkpoint/sqlite/__init__.py

class SqliteSaver(BaseCheckpointSaver[str]):
    """SQLite 持久化 - 适合单机部署"""
    
    def setup(self):
        """创建表结构"""
        with self.cursor() as cur:
            cur.executescript("""
                PRAGMA journal_mode=WAL;  -- 并发优化
                
                CREATE TABLE IF NOT EXISTS checkpoints (
                    thread_id TEXT NOT NULL,
                    checkpoint_ns TEXT NOT NULL DEFAULT '',
                    checkpoint_id TEXT NOT NULL,
                    parent_checkpoint_id TEXT,
                    type TEXT,  -- serde 类型标识 (msgpack/json/pickle)
                    checkpoint BLOB,  -- 序列化后的二进制数据
                    metadata BLOB,
                    PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id)
                );
                
                CREATE TABLE IF NOT EXISTS writes (
                    thread_id TEXT NOT NULL,
                    checkpoint_ns TEXT NOT NULL DEFAULT '',
                    checkpoint_id TEXT NOT NULL,
                    task_id TEXT NOT NULL,
                    idx INTEGER NOT NULL,  -- WRITES_IDX_MAP: -1=ERROR, -2=SCHEDULED, ...
                    channel TEXT NOT NULL,
                    type TEXT,
                    value BLOB,
                    PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id, task_id, idx)
                );
            """)
    
    def put(self, config, checkpoint, metadata, new_versions):
        thread_id = config["configurable"]["thread_id"]
        checkpoint_ns = config["configurable"].get("checkpoint_ns", "")
        
        # 序列化
        type_cp, cp_data = self.jsonplus_serde.dumps_typed(checkpoint)
        type_md, md_data = self.jsonplus_serde.dumps_typed(metadata)
        
        # 写入数据库
        with self.cursor() as cur:
            cur.execute(
                """
                INSERT OR REPLACE INTO checkpoints 
                (thread_id, checkpoint_ns, checkpoint_id, parent_checkpoint_id, type, checkpoint, metadata)
                VALUES (?, ?, ?, ?, ?, ?, ?)
                """,
                (
                    thread_id,
                    checkpoint_ns,
                    checkpoint["id"],
                    config["configurable"].get("checkpoint_id"),  # parent
                    type_cp,
                    cp_data,
                    md_data
                )
            )
        
        return {
            "configurable": {
                "thread_id": thread_id,
                "checkpoint_ns": checkpoint_ns,
                "checkpoint_id": checkpoint["id"]
            }
        }
    
    def get_tuple(self, config):
        """获取 checkpoint (支持查询最新或指定 ID)"""
        thread_id = config["configurable"]["thread_id"]
        checkpoint_ns = config["configurable"].get("checkpoint_ns", "")
        
        with self.cursor() as cur:
            if checkpoint_id := config["configurable"].get("checkpoint_id"):
                # 精确查询
                cur.execute(
                    """
                    SELECT checkpoint_id, parent_checkpoint_id, type, checkpoint, metadata
                    FROM checkpoints
                    WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?
                    """,
                    (thread_id, checkpoint_ns, checkpoint_id)
                )
            else:
                # 查询最新
                cur.execute(
                    """
                    SELECT checkpoint_id, parent_checkpoint_id, type, checkpoint, metadata
                    FROM checkpoints
                    WHERE thread_id = ? AND checkpoint_ns = ?
                    ORDER BY checkpoint_id DESC  -- UUID v6 天然可排序!
                    LIMIT 1
                    """,
                    (thread_id, checkpoint_ns)
                )
            
            row = cur.fetchone()
            if not row:
                return None
            
            checkpoint_id, parent_id, type_, cp_data, md_data = row
            
            # 反序列化
            checkpoint = self.jsonplus_serde.loads_typed((type_, cp_data))
            metadata = self.jsonplus_serde.loads_typed((type_, md_data))
            
            # 查询 pending writes (HITL)
            cur.execute(
                """
                SELECT task_id, channel, value
                FROM writes
                WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?
                ORDER BY task_id, idx
                """,
                (thread_id, checkpoint_ns, checkpoint_id)
            )
            
            pending_writes = [
                (task_id, channel, self.jsonplus_serde.loads_typed(value))
                for task_id, channel, value in cur.fetchall()
            ]
            
            return CheckpointTuple(
                config={
                    "configurable": {
                        "thread_id": thread_id,
                        "checkpoint_ns": checkpoint_ns,
                        "checkpoint_id": checkpoint_id
                    }
                },
                checkpoint=checkpoint,
                metadata=metadata,
                parent_config={"configurable": {..., "checkpoint_id": parent_id}} if parent_id else None,
                pending_writes=pending_writes or None
            )
```

**优点:**
- 持久化存储
- 单文件部署 (`checkpoints.sqlite`)
- 支持事务 (ACID)

**缺点 (Linus 批判):**
- **Metadata 查询是笑话** - SQLite 的 JSON 函数太弱,`filter` 参数基本没用
- **并发性能差** - 单文件锁,多进程写入会阻塞
- **AsyncSqliteSaver 是包装器** - 本质还是同步 I/O

---

### 3.4 序列化: JsonPlusSerializer

```python
# langgraph/checkpoint/serde/jsonplus.py

class JsonPlusSerializer(SerializerProtocol):
    """基于 ormsgpack (MessagePack) 的高效序列化器
    
    支持的类型:
    - 基础类型: int, float, str, bool, None
    - 容器: list, dict, tuple, set, deque
    - 时间: datetime, date, time, timedelta, ZoneInfo
    - 网络: IPv4/IPv6 Address/Network/Interface
    - 其他: UUID, Decimal, Enum, Path
    - LangChain: BaseMessage, Document, Pydantic models
    - 特殊: numpy.ndarray (如果安装了 numpy)
    """
    
    def dumps_typed(self, obj: Any) -> tuple[str, bytes]:
        """序列化对象 → (类型标识, 二进制数据)"""
        if obj is None:
            return ("null", b"")
        
        # 使用 ormsgpack (C 实现,比 json 快 3-5 倍)
        data = ormsgpack.packb(
            obj,
            default=_msgpack_default,  # 自定义类型编码
            option=_option
        )
        return ("msgpack", data)
    
    def loads_typed(self, data: tuple[str, bytes]) -> Any:
        """反序列化 (type_, data_) → 原对象"""
        type_, data_ = data
        
        if type_ == "null":
            return None
        elif type_ == "msgpack":
            return ormsgpack.unpackb(
                data_,
                ext_hook=_msgpack_ext_hook,  # 自定义类型解码
                option=ormsgpack.OPT_NON_STR_KEYS
            )
        elif type_ == "json":
            return json.loads(data_, object_hook=self._reviver)
        elif type_ == "pickle":
            if not self.pickle_fallback:
                raise ValueError("Pickle deserialization disabled")
            return pickle.loads(data_)
        else:
            raise ValueError(f"Unknown type: {type_}")

# 自定义编码示例
def _msgpack_default(obj: Any) -> ormsgpack.Ext:
    # Pydantic v2 模型
    if hasattr(obj, "model_dump"):
        return ormsgpack.Ext(
            EXT_PYDANTIC_V2,
            ormsgpack.packb([
                obj.__class__.__module__,
                obj.__class__.__name__,
                obj.model_dump()
            ])
        )
    
    # Pydantic v1 模型
    elif hasattr(obj, "dict"):
        return ormsgpack.Ext(EXT_PYDANTIC_V1, ...)
    
    # 带构造函数的类 (如 Path)
    elif hasattr(obj, "__reduce_ex__"):
        return ormsgpack.Ext(EXT_CONSTRUCTOR_KW_ARGS, ...)
    
    # Fallback to pickle
    elif self.pickle_fallback:
        return pickle.dumps(obj)
    
    raise TypeError(f"Cannot serialize {type(obj)}")
```

**设计评价:**
- ✅ 用 MessagePack 而非 JSON → 体积小 30-50%,速度快 3-5 倍
- ✅ 支持 Pydantic 模型 → 开箱即用
- ⚠️ Pickle fallback 是安全风险 → 默认禁用,仅用于调试

---

## 四、最佳实践

### 4.1 选择合适的 Checkpointer

```python
# ❌ 生产环境不要用
from langgraph.checkpoint.memory import InMemorySaver
checkpointer = InMemorySaver()  # 进程重启丢失数据

# ✅ 开发/测试 - 单机持久化
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3
conn = sqlite3.connect("checkpoints.db", check_same_thread=False)
checkpointer = SqliteSaver(conn)
checkpointer.setup()

# ✅ 生产环境 - PostgreSQL (推荐)
from langgraph.checkpoint.postgres import PostgresSaver
checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost/db"
)
await checkpointer.setup()
```

**为什么 PostgreSQL 最好?**
1. **JSONB 原生支持** - 嵌套查询高效 (`metadata @> '{"step": 5}'`)
2. **GIN 索引** - Metadata 查询速度快 10-100 倍
3. **并发性能** - MVCC 架构,多进程写入无锁
4. **生产级可靠性** - ACID + WAL + 主从复制

---

### 4.2 Thread ID 设计模式

```python
# ✅ 推荐: 用户 ID + 会话类型
thread_id = f"user-{user_id}-debug-session"

# ✅ 推荐: UUID (每次对话独立)
import uuid
thread_id = str(uuid.uuid4())

# ❌ 不推荐: 固定字符串 (所有用户共享状态)
thread_id = "default"

# ✅ 高级: 分层命名空间
thread_id = f"org-{org_id}/project-{project_id}/task-{task_id}"
```

---

### 4.3 Checkpoint 清理策略

```python
# 定期清理过期 checkpoint (避免数据库膨胀)
import datetime

def cleanup_old_checkpoints(checkpointer, days=30):
    """删除 30 天前的 checkpoint"""
    cutoff = datetime.datetime.now() - datetime.timedelta(days=days)
    cutoff_iso = cutoff.isoformat()
    
    # 遍历所有 thread
    for thread_id in get_all_threads():
        config = {"configurable": {"thread_id": thread_id}}
        
        # 获取所有 checkpoint
        checkpoints = list(checkpointer.list(config))
        
        for cp_tuple in checkpoints:
            if cp_tuple.checkpoint["ts"] < cutoff_iso:
                # 注意: BaseCheckpointSaver 没有 delete_checkpoint 方法
                # 需要直接操作数据库
                checkpointer.conn.execute(
                    "DELETE FROM checkpoints WHERE thread_id = ? AND checkpoint_id = ?",
                    (thread_id, cp_tuple.checkpoint["id"])
                )
```

---

## 五、源码解析: 关键流程

### 5.1 Checkpoint 创建时机

```python
# langgraph/pregel/loop.py (简化版)

async def astream(...):
    # 1️⃣ 加载初始 checkpoint
    checkpoint = await checkpointer.aget_tuple(config)
    
    if not checkpoint:
        # 创建空 checkpoint
        checkpoint = empty_checkpoint()
        checkpoint = await checkpointer.aput(
            config, checkpoint, 
            metadata={"source": "input", "step": -1},
            new_versions={}
        )
    
    # 2️⃣ Pregel 循环
    for step in range(recursion_limit):
        # 执行所有并行节点
        tasks = [node.ainvoke(...) for node in ready_nodes]
        results = await asyncio.gather(*tasks)
        
        # 更新状态
        for channel, value in results:
            state.channels[channel].update(value)
        
        # 3️⃣ 保存 checkpoint
        new_checkpoint = create_checkpoint(state, step)
        config = await checkpointer.aput(
            config,
            checkpoint=new_checkpoint,
            metadata={"source": "loop", "step": step},
            new_versions=state.channel_versions
        )
        
        # 4️⃣ 检查中断
        if state.has_interrupt:
            await checkpointer.aput_writes(
                config,
                writes=state.pending_writes,
                task_id="__interrupt__"
            )
            yield {"__interrupt__": state.interrupts}
            break
        
        # 检查是否结束
        if not ready_nodes:
            break
```

---

### 5.2 HITL 中断的底层实现

```python
# deepagents/middleware/human_in_the_loop.py

class HumanInTheLoopMiddleware(AgentMiddleware):
    def wrap_tool_call(self, request, handler):
        tool_name = request.tool.name
        
        # 检查是否需要中断
        if tool_name in self.interrupt_on:
            # 创建中断请求
            hitl_request = {
                "action_requests": [{
                    "name": tool_name,
                    "args": request.args,
                    "description": self._format_description(tool_name, request.args)
                }],
                "review_configs": [{
                    "action_name": tool_name,
                    "allowed_decisions": ["approve", "reject", "edit"]
                }]
            }
            
            # 触发中断 (LangGraph 会自动保存 checkpoint)
            raise NodeInterrupt(hitl_request)
        
        # 正常执行
        return handler(request)

# LangGraph 内部处理
try:
    result = node.invoke(...)
except NodeInterrupt as e:
    # 保存中断状态到 checkpoint
    await checkpointer.aput_writes(
        config,
        writes=[(INTERRUPT, e.value)],
        task_id=node_name
    )
    # 返回 __interrupt__ 给调用者
    return {"__interrupt__": [Interrupt(value=e.value, id=...)]}
```

---

### 5.3 版本号生成策略

```python
# langgraph/checkpoint/base/__init__.py

class BaseCheckpointSaver:
    def get_next_version(self, current: V | None, channel: None) -> V:
        """生成下一个版本号
        
        默认实现: 整数递增
        可以重写为: 时间戳 / UUID / 自定义
        """
        if isinstance(current, str):
            raise NotImplementedError  # 必须子类实现
        elif current is None:
            return 1  # 初始版本
        else:
            return current + 1  # 递增

# SqliteSaver 的实现
class SqliteSaver(BaseCheckpointSaver[str]):
    def get_next_version(self, current: str | None, channel: None) -> str:
        """使用随机字符串作为版本号"""
        if current is None:
            current = ""
        # 生成 6 位随机字符串
        return current + "".join(random.choices("0123456789abcdef", k=6))
```

**Linus 批判:**
> "为什么版本号要支持 `str | int | float` 三种类型? 这是典型的过度灵活。直接用整数不好吗? SQLite 用随机字符串是为了避免冲突,但这引入了不必要的复杂性。如果真担心冲突,用 UUID 或者时间戳 + 序列号。"

---

## 六、常见问题

### Q1: 为什么 checkpoint_id 用 UUID v6 而非 v4?

**A:** UUID v6 的前 48 位是时间戳 (单调递增),可以直接排序:

```python
import uuid
from uuid6 import uuid6

# UUID v4 - 完全随机,无法排序
id1 = str(uuid.uuid4())  # "3c4e5f6a-..."
id2 = str(uuid.uuid4())  # "1a2b3c4d-..."
assert id1 > id2  # ❌ 无意义的比较

# UUID v6 - 时间递增,可排序
id1 = str(uuid6())  # "1ed2a3b4-c5d6-6..."
time.sleep(0.001)
id2 = str(uuid6())  # "1ed2a3b4-c5d7-6..."
assert id2 > id1  # ✅ id2 确实晚于 id1
```

这使得 SQLite 可以用 `ORDER BY checkpoint_id DESC` 直接查询最新 checkpoint,无需额外的时间戳列。

---

### Q2: 为什么有 `checkpoints` 和 `writes` 两张表?

**A:** 分离设计:
- `checkpoints` 表 - 存储完整状态快照
- `writes` 表 - 存储 "待处理的写入" (如 HITL 中断时的 tool call)

```sql
-- 查询某个 checkpoint 的状态
SELECT checkpoint FROM checkpoints WHERE checkpoint_id = ?;

-- 查询该 checkpoint 的 pending writes (HITL)
SELECT channel, value FROM writes WHERE checkpoint_id = ?;
```

这样可以在不修改 checkpoint 的情况下附加 pending writes。

---

### Q3: Metadata 查询为什么这么弱?

**A:** SQLite 的 JSON 函数只能做简单查询:

```python
# ❌ 复杂查询不支持
filter = {"step": {"$gte": 5}, "user_id": "123"}
checkpointer.list(config, filter=filter)  # SQLite: 报错或全扫描

# ✅ PostgreSQL JSONB 可以
# SELECT * FROM checkpoints WHERE metadata @> '{"step": 5}'::jsonb
```

**解决方案:**
1. 用 PostgreSQL (推荐)
2. 把常用字段提取到单独的列:
   ```sql
   ALTER TABLE checkpoints ADD COLUMN step INTEGER;
   CREATE INDEX idx_step ON checkpoints(step);
   ```

---

## 七、总结: Linus 式评价

### ✅ 做对的事情

1. **解决真实问题** - Agent 需要状态持久化,这不是臆想
2. **数据结构简洁** - `Checkpoint` 用 TypedDict,清晰直观
3. **UUID v6 妙用** - 天然可排序,省去序列号字段
4. **MessagePack 序列化** - 比 JSON 快 3-5 倍,体积小 30-50%

### ❌ 过度设计

1. **版本号类型混用** - `str | int | float` 引入不必要的复杂性
2. **AsyncSqliteSaver 包装** - 本质还是同步 I/O,性能提升有限
3. **Metadata 查询幻觉** - SQLite 不适合复杂 JSON 查询,别硬凑

### 🔧 改进建议

```python
# 1. 统一版本号为整数
class Checkpoint(TypedDict):
    channel_versions: dict[str, int]  # 去掉 str | float

# 2. 去掉 AsyncSqliteSaver 的同步包装
# 要么全异步,要么全同步,别混用

# 3. Metadata 字段拆出来
CREATE TABLE checkpoints (
    ...
    step INTEGER,
    source TEXT,
    user_id TEXT,
    metadata_json BLOB  -- 剩余的非结构化数据
);
CREATE INDEX idx_step ON checkpoints(step);
```

---

## 附录: 核心源码位置

```text
langgraph-main/libs/checkpoint/
├── langgraph/checkpoint/base/
│   └── __init__.py          # BaseCheckpointSaver, Checkpoint, CheckpointMetadata
├── langgraph/checkpoint/memory/
│   └── __init__.py          # InMemorySaver (纯内存)
├── langgraph/checkpoint/serde/
│   ├── base.py              # SerializerProtocol
│   └── jsonplus.py          # JsonPlusSerializer (MessagePack)
└── langgraph/checkpoint/sqlite/
    ├── __init__.py          # SqliteSaver (同步)
    └── aio.py               # AsyncSqliteSaver (异步)

langgraph-main/libs/checkpoint-sqlite/
└── langgraph/checkpoint/sqlite/
    ├── __init__.py          # SqliteSaver 实现
    ├── aio.py               # AsyncSqliteSaver 实现
    └── utils.py             # search_where (Metadata 查询)
```

---

**最后的 Linus 式总结:**

> "Checkpoint 机制是对的,但实现有太多'聪明'细节。如果你要生产用,直接上 PostgreSQL,别浪费时间优化 SQLite。如果你要本地调试,InMemorySaver 够用。SQLite 是个尴尬的中间地带 - 既不够快,也不够强。"
> 
> "AsyncSqliteSaver 包装 SqliteSaver 是在自欺欺人。SQLite 是单文件锁,异步只是把阻塞挪到线程池,本质还是同步 I/O。真要异步,换 PostgreSQL + asyncpg。"
> 
> "Metadata 查询是个笑话。SQLite 的 JSON 函数只能做最简单的等值查询,复杂条件直接全表扫描。要么用 PostgreSQL JSONB,要么把常用字段拆成列。别幻想 SQLite 能当 NoSQL 用。"
