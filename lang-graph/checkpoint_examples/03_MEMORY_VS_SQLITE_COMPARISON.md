# InMemorySaver vs SqliteSaver 源码对比

> **核心差异**: 内存存储 vs 持久化存储,三层嵌套字典 vs SQL 表结构

---

## 一、架构对比

### 1.1 存储结构差异

```python
# ===== InMemorySaver - 三层嵌套字典 =====
class InMemorySaver(BaseCheckpointSaver[str]):
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
    
    writes: defaultdict[
        tuple[str, str, str],  # (thread_id, checkpoint_ns, checkpoint_id)
        dict[
            tuple[str, int],  # (task_id, write_idx)
            tuple[str, str, tuple[str, bytes], str]  # (channel, type, value, task_path)
        ]
    ]
    
    blobs: dict[
        tuple[str, str, str, str | int | float],  # (thread_id, checkpoint_ns, channel, version)
        tuple[str, bytes]  # (type, serialized_value)
    ]

# ===== SqliteSaver - SQL 表结构 =====
CREATE TABLE checkpoints (
    thread_id TEXT NOT NULL,
    checkpoint_ns TEXT NOT NULL DEFAULT '',
    checkpoint_id TEXT NOT NULL,
    parent_checkpoint_id TEXT,
    type TEXT,  -- 序列化类型 (msgpack/json/pickle)
    checkpoint BLOB,  -- 序列化后的 checkpoint 数据
    metadata BLOB,    -- 序列化后的 metadata (JSON)
    PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id)
);

CREATE TABLE writes (
    thread_id TEXT NOT NULL,
    checkpoint_ns TEXT NOT NULL DEFAULT '',
    checkpoint_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    idx INTEGER NOT NULL,  -- WRITES_IDX_MAP: -1=ERROR, -2=SCHEDULED, -3=INTERRUPT
    channel TEXT NOT NULL,
    type TEXT,
    value BLOB,
    PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id, task_id, idx)
);
```

---

## 二、核心方法对比

### 2.1 get_tuple - 获取 Checkpoint

#### InMemorySaver 实现

```python
# 源码位置: langgraph/checkpoint/memory/__init__.py:L135-215

def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None:
    """从内存字典中获取 checkpoint"""
    thread_id: str = config["configurable"]["thread_id"]
    checkpoint_ns: str = config["configurable"].get("checkpoint_ns", "")
    
    # ===== 场景1: 精确查询 =====
    if checkpoint_id := get_checkpoint_id(config):
        # 直接索引字典 - O(1) 复杂度
        if saved := self.storage[thread_id][checkpoint_ns].get(checkpoint_id):
            checkpoint, metadata, parent_checkpoint_id = saved
            writes = self.writes[(thread_id, checkpoint_ns, checkpoint_id)].values()
            
            checkpoint_: Checkpoint = self.serde.loads_typed(checkpoint)
            
            return CheckpointTuple(
                config=config,
                checkpoint={
                    **checkpoint_,
                    "channel_values": self._load_blobs(  # ⚠️ 从 blobs 字典加载实际数据
                        thread_id, checkpoint_ns, checkpoint_["channel_versions"]
                    ),
                },
                metadata=self.serde.loads_typed(metadata),
                pending_writes=[
                    (id, c, self.serde.loads_typed(v)) for id, c, v, _ in writes
                ],
                parent_config={
                    "configurable": {
                        "thread_id": thread_id,
                        "checkpoint_ns": checkpoint_ns,
                        "checkpoint_id": parent_checkpoint_id,
                    }
                } if parent_checkpoint_id else None,
            )
    
    # ===== 场景2: 查询最新 checkpoint =====
    else:
        if checkpoints := self.storage[thread_id][checkpoint_ns]:
            # ⚠️ max() 需要遍历所有 key - O(n) 复杂度
            # 但因为 checkpoint_id 是 UUID v6 (时间递增),直接字符串比较即可
            checkpoint_id = max(checkpoints.keys())
            
            checkpoint, metadata, parent_checkpoint_id = checkpoints[checkpoint_id]
            writes = self.writes[(thread_id, checkpoint_ns, checkpoint_id)].values()
            checkpoint_ = self.serde.loads_typed(checkpoint)
            
            return CheckpointTuple(
                config={
                    "configurable": {
                        "thread_id": thread_id,
                        "checkpoint_ns": checkpoint_ns,
                        "checkpoint_id": checkpoint_id,
                    }
                },
                checkpoint={
                    **checkpoint_,
                    "channel_values": self._load_blobs(
                        thread_id, checkpoint_ns, checkpoint_["channel_versions"]
                    ),
                },
                metadata=self.serde.loads_typed(metadata),
                pending_writes=[
                    (id, c, self.serde.loads_typed(v)) for id, c, v, _ in writes
                ],
                parent_config={
                    "configurable": {
                        "thread_id": thread_id,
                        "checkpoint_ns": checkpoint_ns,
                        "checkpoint_id": parent_checkpoint_id,
                    }
                } if parent_checkpoint_id else None,
            )

def _load_blobs(
    self, thread_id: str, checkpoint_ns: str, versions: ChannelVersions
) -> dict[str, Any]:
    """从 blobs 字典加载 channel 数据"""
    channel_values: dict[str, Any] = {}
    for k, v in versions.items():
        # ⚠️ 每个 channel 都需要一次字典查询
        if blob := self.blobs.get((thread_id, checkpoint_ns, k, v)):
            channel_values[k] = self.serde.loads_typed(blob)
    return channel_values
```

**性能分析 (InMemorySaver):**
- ✅ **精确查询**: O(1) 字典索引
- ⚠️ **查询最新**: O(n) max() 遍历 (n = checkpoint 数量)
- ⚠️ **加载 blobs**: O(m) 遍历 channels (m = channel 数量)

---

#### SqliteSaver 实现

```python
# 源码位置: langgraph/checkpoint/sqlite/__init__.py:L184-286

def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None:
    """从 SQLite 数据库获取 checkpoint"""
    checkpoint_ns = config["configurable"].get("checkpoint_ns", "")
    
    with self.cursor(transaction=False) as cur:
        # ===== 场景1: 精确查询 =====
        if checkpoint_id := get_checkpoint_id(config):
            cur.execute(
                """
                SELECT thread_id, checkpoint_id, parent_checkpoint_id, type, checkpoint, metadata
                FROM checkpoints
                WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?
                """,
                (
                    str(config["configurable"]["thread_id"]),
                    checkpoint_ns,
                    checkpoint_id,
                ),
            )
        
        # ===== 场景2: 查询最新 checkpoint =====
        else:
            cur.execute(
                """
                SELECT thread_id, checkpoint_id, parent_checkpoint_id, type, checkpoint, metadata
                FROM checkpoints
                WHERE thread_id = ? AND checkpoint_ns = ?
                ORDER BY checkpoint_id DESC  -- ⚠️ UUID v6 可排序,无需时间戳字段
                LIMIT 1
                """,
                (str(config["configurable"]["thread_id"]), checkpoint_ns),
            )
        
        # ===== 反序列化结果 =====
        if value := cur.fetchone():
            (
                thread_id,
                checkpoint_id,
                parent_checkpoint_id,
                type,
                checkpoint,
                metadata,
            ) = value
            
            if not get_checkpoint_id(config):
                config = {
                    "configurable": {
                        "thread_id": thread_id,
                        "checkpoint_ns": checkpoint_ns,
                        "checkpoint_id": checkpoint_id,
                    }
                }
            
            # ===== 查询 pending writes =====
            cur.execute(
                """
                SELECT task_id, channel, type, value
                FROM writes
                WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?
                ORDER BY task_id, idx
                """,
                (
                    str(config["configurable"]["thread_id"]),
                    checkpoint_ns,
                    str(config["configurable"]["checkpoint_id"]),
                ),
            )
            
            return CheckpointTuple(
                config,
                self.serde.loads_typed((type, checkpoint)),  # ⚠️ 反序列化
                cast(
                    CheckpointMetadata,
                    json.loads(metadata) if metadata is not None else {},
                ),
                {
                    "configurable": {
                        "thread_id": thread_id,
                        "checkpoint_ns": checkpoint_ns,
                        "checkpoint_id": parent_checkpoint_id,
                    }
                } if parent_checkpoint_id else None,
                [
                    (task_id, channel, self.serde.loads_typed((type, value)))
                    for task_id, channel, type, value in cur
                ],
            )

@contextmanager
def cursor(self, transaction: bool = True) -> Iterator[sqlite3.Cursor]:
    """获取 SQLite 游标,支持事务"""
    self.setup()  # 确保表已创建
    cur = self.conn.cursor()
    try:
        yield cur
    finally:
        if transaction:
            self.conn.commit()  # ⚠️ 自动提交事务
        cur.close()
```

**性能分析 (SqliteSaver):**
- ✅ **精确查询**: O(1) 主键索引 (假设有 B-tree 索引)
- ✅ **查询最新**: O(log n) 排序 + LIMIT 1
- ⚠️ **I/O 开销**: 磁盘读取比内存慢 100-1000 倍
- ⚠️ **序列化**: MessagePack 反序列化有 CPU 开销

---

### 2.2 list - 列出所有 Checkpoint

#### InMemorySaver 实现

```python
# 源码位置: langgraph/checkpoint/memory/__init__.py:L217-324

def list(
    self,
    config: RunnableConfig | None,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> Iterator[CheckpointTuple]:
    """列出所有 checkpoint (内存遍历)"""
    
    # ===== 1. 确定要遍历的 thread_id =====
    thread_ids = (config["configurable"]["thread_id"],) if config else self.storage
    config_checkpoint_ns = (
        config["configurable"].get("checkpoint_ns") if config else None
    )
    config_checkpoint_id = get_checkpoint_id(config) if config else None
    
    # ===== 2. 三层嵌套遍历 =====
    for thread_id in thread_ids:
        for checkpoint_ns in self.storage[thread_id].keys():
            # 过滤 checkpoint_ns
            if (
                config_checkpoint_ns is not None
                and checkpoint_ns != config_checkpoint_ns
            ):
                continue
            
            # ⚠️ 按 checkpoint_id 倒序排序 (O(n log n))
            for checkpoint_id, (
                checkpoint,
                metadata_b,
                parent_checkpoint_id,
            ) in sorted(
                self.storage[thread_id][checkpoint_ns].items(),
                key=lambda x: x[0],
                reverse=True,
            ):
                # ===== 3. 应用过滤条件 =====
                
                # 过滤: checkpoint_id 匹配
                if config_checkpoint_id and checkpoint_id != config_checkpoint_id:
                    continue
                
                # 过滤: before (只返回比 before 更早的)
                if (
                    before
                    and (before_checkpoint_id := get_checkpoint_id(before))
                    and checkpoint_id >= before_checkpoint_id
                ):
                    continue
                
                # 过滤: metadata 匹配
                metadata = self.serde.loads_typed(metadata_b)
                if filter and not all(
                    query_value == metadata.get(query_key)
                    for query_key, query_value in filter.items()
                ):
                    continue
                
                # ===== 4. 限制结果数量 =====
                if limit is not None and limit <= 0:
                    break
                elif limit is not None:
                    limit -= 1
                
                # ===== 5. 构建返回结果 =====
                writes = self.writes[
                    (thread_id, checkpoint_ns, checkpoint_id)
                ].values()
                
                checkpoint_: Checkpoint = self.serde.loads_typed(checkpoint)
                
                yield CheckpointTuple(
                    config={
                        "configurable": {
                            "thread_id": thread_id,
                            "checkpoint_ns": checkpoint_ns,
                            "checkpoint_id": checkpoint_id,
                        }
                    },
                    checkpoint={
                        **checkpoint_,
                        "channel_values": self._load_blobs(
                            thread_id,
                            checkpoint_ns,
                            checkpoint_["channel_versions"],
                        ),
                    },
                    metadata=metadata,
                    parent_config={
                        "configurable": {
                            "thread_id": thread_id,
                            "checkpoint_ns": checkpoint_ns,
                            "checkpoint_id": parent_checkpoint_id,
                        }
                    } if parent_checkpoint_id else None,
                    pending_writes=[
                        (id, c, self.serde.loads_typed(v)) for id, c, v, _ in writes
                    ],
                )
```

**性能分析 (InMemorySaver):**
- ⚠️ **排序开销**: O(n log n) sorted()
- ⚠️ **metadata 过滤**: 每个 checkpoint 都需要反序列化 metadata
- ✅ **内存访问**: 无 I/O 开销
- ⚠️ **内存占用**: 所有 checkpoint 都在内存中

---

#### SqliteSaver 实现

```python
# 源码位置: langgraph/checkpoint/sqlite/__init__.py:L288-378

def list(
    self,
    config: RunnableConfig | None,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> Iterator[CheckpointTuple]:
    """列出所有 checkpoint (SQL 查询)"""
    
    # ===== 1. 构建 WHERE 子句 =====
    where, param_values = search_where(config, filter, before)
    # 示例 WHERE:
    # WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id < ?
    
    # ===== 2. 构建完整 SQL =====
    query = f"""
        SELECT thread_id, checkpoint_ns, checkpoint_id, parent_checkpoint_id, type, checkpoint, metadata
        FROM checkpoints
        {where}
        ORDER BY checkpoint_id DESC
    """
    if limit is not None:
        query += " LIMIT ?"
        param_values = (*param_values, limit)
    
    # ===== 3. 执行查询 =====
    with self.cursor(transaction=False) as cur, closing(self.conn.cursor()) as wcur:
        cur.execute(query, param_values)
        
        for (
            thread_id,
            checkpoint_ns,
            checkpoint_id,
            parent_checkpoint_id,
            type,
            checkpoint,
            metadata,
        ) in cur:
            # ===== 4. 查询 pending writes =====
            wcur.execute(
                """
                SELECT task_id, channel, type, value
                FROM writes
                WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?
                ORDER BY task_id, idx
                """,
                (thread_id, checkpoint_ns, checkpoint_id),
            )
            
            yield CheckpointTuple(
                {
                    "configurable": {
                        "thread_id": thread_id,
                        "checkpoint_ns": checkpoint_ns,
                        "checkpoint_id": checkpoint_id,
                    }
                },
                self.serde.loads_typed((type, checkpoint)),
                cast(
                    CheckpointMetadata,
                    json.loads(metadata) if metadata is not None else {},
                ),
                {
                    "configurable": {
                        "thread_id": thread_id,
                        "checkpoint_ns": checkpoint_ns,
                        "checkpoint_id": parent_checkpoint_id,
                    }
                } if parent_checkpoint_id else None,
                [
                    (task_id, channel, self.serde.loads_typed((type, value)))
                    for task_id, channel, type, value in wcur
                ],
            )

# 源码位置: langgraph/checkpoint/sqlite/utils.py:L76-116
def search_where(
    config: RunnableConfig | None,
    filter: dict[str, Any] | None,
    before: RunnableConfig | None = None,
) -> tuple[str, Sequence[Any]]:
    """构建 WHERE 子句 (支持 metadata 过滤)"""
    
    where_clauses = []
    param_values = []
    
    # ===== 1. thread_id 过滤 =====
    if config and "thread_id" in config["configurable"]:
        where_clauses.append("thread_id = ?")
        param_values.append(str(config["configurable"]["thread_id"]))
    
    # ===== 2. checkpoint_ns 过滤 =====
    if config and "checkpoint_ns" in config["configurable"]:
        where_clauses.append("checkpoint_ns = ?")
        param_values.append(config["configurable"]["checkpoint_ns"])
    
    # ===== 3. before (时间范围) =====
    if before and (before_checkpoint_id := get_checkpoint_id(before)):
        where_clauses.append("checkpoint_id < ?")
        param_values.append(before_checkpoint_id)
    
    # ===== 4. metadata 过滤 (⚠️ SQLite 限制) =====
    if filter:
        where_clauses_meta, param_values_meta = _metadata_predicate(filter)
        where_clauses.extend(where_clauses_meta)
        param_values.extend(param_values_meta)
    
    where_sql = "WHERE " + " AND ".join(where_clauses) if where_clauses else ""
    
    return where_sql, param_values

def _metadata_predicate(
    metadata_filter: dict[str, Any],
) -> tuple[Sequence[str], Sequence[Any]]:
    """构建 metadata 过滤条件 (⚠️ SQLite JSON 函数限制)"""
    clauses = []
    params = []
    
    for key, value in metadata_filter.items():
        # 验证 key 安全 (防 SQL 注入)
        _validate_filter_key(key)
        
        # ⚠️ SQLite 只能做简单的等值查询
        # json_extract(metadata, '$.key') = value
        clauses.append(f"json_extract(metadata, '$.{key}') = ?")
        params.append(json.dumps(value) if not isinstance(value, str) else value)
    
    return clauses, params
```

**性能分析 (SqliteSaver):**
- ✅ **排序优化**: 数据库内部排序,可能用到索引
- ⚠️ **metadata 过滤**: `json_extract()` 无法使用索引,全表扫描
- ⚠️ **I/O 开销**: 每个 checkpoint 都需要读磁盘
- ✅ **内存占用**: 流式返回,不需要一次性加载所有数据

---

### 2.3 put - 保存 Checkpoint

#### InMemorySaver 实现

```python
# 源码位置: langgraph/checkpoint/memory/__init__.py:L326-370

def put(
    self,
    config: RunnableConfig,
    checkpoint: Checkpoint,
    metadata: CheckpointMetadata,
    new_versions: ChannelVersions,
) -> RunnableConfig:
    """保存 checkpoint 到内存"""
    c = checkpoint.copy()
    thread_id = config["configurable"]["thread_id"]
    checkpoint_ns = config["configurable"]["checkpoint_ns"]
    
    # ===== 1. 分离 channel_values 到 blobs 字典 =====
    values: dict[str, Any] = c.pop("channel_values")  # 从 checkpoint 中移除
    
    # 遍历新版本,保存到 blobs
    for k, v in new_versions.items():
        self.blobs[(thread_id, checkpoint_ns, k, v)] = (
            self.serde.dumps_typed(values[k]) if k in values else ("empty", b"")
        )
    
    # ===== 2. 保存 checkpoint 元数据到 storage 字典 =====
    self.storage[thread_id][checkpoint_ns].update(
        {
            checkpoint["id"]: (
                self.serde.dumps_typed(c),  # 序列化 checkpoint (不含 channel_values)
                self.serde.dumps_typed(get_checkpoint_metadata(config, metadata)),
                config["configurable"].get("checkpoint_id"),  # parent_checkpoint_id
            )
        }
    )
    
    # ===== 3. 返回新的 config =====
    return {
        "configurable": {
            "thread_id": thread_id,
            "checkpoint_ns": checkpoint_ns,
            "checkpoint_id": checkpoint["id"],
        }
    }
```

**设计要点 (InMemorySaver):**
- ✅ **Blobs 分离存储** - 避免重复存储大对象 (如文件内容)
- ✅ **O(1) 插入** - 字典 update
- ⚠️ **无持久化** - 进程重启丢失

---

#### SqliteSaver 实现

```python
# 源码位置: langgraph/checkpoint/sqlite/__init__.py:L380-436

def put(
    self,
    config: RunnableConfig,
    checkpoint: Checkpoint,
    metadata: CheckpointMetadata,
    new_versions: ChannelVersions,
) -> RunnableConfig:
    """保存 checkpoint 到 SQLite"""
    thread_id = config["configurable"]["thread_id"]
    checkpoint_ns = config["configurable"]["checkpoint_ns"]
    
    # ===== 1. 序列化 checkpoint =====
    type_, serialized_checkpoint = self.serde.dumps_typed(checkpoint)
    # type_ = "msgpack" (默认)
    # serialized_checkpoint = MessagePack 二进制数据
    
    # ===== 2. 序列化 metadata (使用 JSON) =====
    serialized_metadata = json.dumps(
        get_checkpoint_metadata(config, metadata),
        ensure_ascii=False
    ).encode("utf-8", "ignore")
    
    # ===== 3. INSERT OR REPLACE (UPSERT) =====
    with self.cursor() as cur:
        cur.execute(
            """
            INSERT OR REPLACE INTO checkpoints
            (thread_id, checkpoint_ns, checkpoint_id, parent_checkpoint_id, type, checkpoint, metadata)
            VALUES (?, ?, ?, ?, ?, ?, ?)
            """,
            (
                str(config["configurable"]["thread_id"]),
                checkpoint_ns,
                checkpoint["id"],
                config["configurable"].get("checkpoint_id"),  # parent
                type_,
                serialized_checkpoint,
                serialized_metadata,
            ),
        )
        # ⚠️ cursor() 上下文管理器会自动 commit
    
    return {
        "configurable": {
            "thread_id": thread_id,
            "checkpoint_ns": checkpoint_ns,
            "checkpoint_id": checkpoint["id"],
        }
    }
```

**设计要点 (SqliteSaver):**
- ✅ **持久化** - 数据写入磁盘,进程重启不丢失
- ✅ **UPSERT** - INSERT OR REPLACE 简化逻辑
- ⚠️ **I/O 开销** - 写磁盘比写内存慢 100-1000 倍
- ⚠️ **事务开销** - 每次 put 都 commit (可批量优化)

---

## 三、性能对比表

| 操作 | InMemorySaver | SqliteSaver | 性能差异 |
|------|---------------|-------------|----------|
| **get_tuple (精确)** | O(1) 字典索引 | O(1) 主键索引 | 内存快 100-1000x |
| **get_tuple (最新)** | O(n) max() | O(log n) + 索引 | 数据量大时 SQLite 更优 |
| **list (全部)** | O(n log n) sorted() | O(n log n) + I/O | 内存快,但占用大 |
| **list (limit=10)** | O(n log n) 仍需全排序 | O(log n + 10) | SQLite 大幅优胜 |
| **put** | O(1) 字典写入 | O(log n) B-tree 插入 + fsync | 内存快 100-1000x |
| **metadata 过滤** | ✅ 支持任意条件 | ⚠️ json_extract 无索引 | 内存更灵活 |
| **持久化** | ❌ 进程重启丢失 | ✅ 数据永久保存 | SQLite 必需 |
| **并发** | ⚠️ 单进程 | ⚠️ 单文件锁,写阻塞 | 都不适合高并发 |

---

## 四、内存占用对比

### 4.1 InMemorySaver 内存占用

```python
# 假设每个 checkpoint 占用:
# - checkpoint 数据: 10 KB (序列化后的 channel_values)
# - metadata: 1 KB
# - blobs: 50 KB (文件内容等)

# 1000 个 checkpoint:
memory_usage = 1000 * (10 + 1 + 50) = 61 MB

# ⚠️ 问题:
# - 所有数据常驻内存
# - GC 压力大 (大量小对象)
# - 无法释放旧 checkpoint
```

### 4.2 SqliteSaver 内存占用

```python
# SQLite 使用页缓存 (默认 2 MB)
# 只有热点数据在内存,冷数据在磁盘

# 查询最新 10 个 checkpoint:
memory_usage = 10 * (10 + 1) = 110 KB  # ⚠️ blobs 在 checkpoint BLOB 中

# ✅ 优点:
# - 冷数据自动换出
# - 内存占用可控
# - 支持 TB 级数据
```

---

## 五、适用场景

### 5.1 使用 InMemorySaver 的场景

```python
from langgraph.checkpoint.memory import InMemorySaver

# ✅ 适用场景:
# 1. 单元测试 / 集成测试
checkpointer = InMemorySaver()

# 2. 演示 / 原型开发
agent = create_deep_agent(checkpointer=InMemorySaver())

# 3. 短期会话 (< 100 个 checkpoint)
# 4. 单进程应用

# ❌ 不适用场景:
# 1. 生产环境 (进程重启丢失数据)
# 2. 多进程 / 分布式
# 3. 长期存储 (> 1000 个 checkpoint)
# 4. 审计需求
```

---

### 5.2 使用 SqliteSaver 的场景

```python
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3

# ✅ 适用场景:
# 1. 本地开发环境
conn = sqlite3.connect("checkpoints.db", check_same_thread=False)
checkpointer = SqliteSaver(conn)

# 2. 单机部署 (无高并发)
# 3. 个人工具 / 桌面应用
# 4. 需要持久化,但不需要分布式

# ❌ 不适用场景:
# 1. 高并发 Web 应用 (写锁阻塞)
# 2. 多进程 worker (文件锁冲突)
# 3. 需要复杂 metadata 查询 (JSON 函数慢)
# 4. 生产环境 (推荐 PostgreSQL)
```

---

### 5.3 生产环境推荐 PostgreSQL

```python
from langgraph.checkpoint.postgres import PostgresSaver

# ✅ 适用场景:
# 1. 生产环境 (高可用)
checkpointer = await PostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost/db"
)

# 2. 高并发 (MVCC 无锁读写)
# 3. 分布式部署 (多进程 worker)
# 4. 复杂查询 (JSONB + GIN 索引)

# 核心优势:
# - JSONB 原生支持: metadata @> '{"step": 5}'::jsonb
# - GIN 索引: 复杂查询速度快 10-100 倍
# - 并发性能: 无写锁阻塞
# - 主从复制: 高可用 + 读写分离
```

---

## 六、源码引用总结

| 功能 | InMemorySaver | SqliteSaver |
|------|---------------|-------------|
| **get_tuple** | `memory/__init__.py:L135-215` | `sqlite/__init__.py:L184-286` |
| **list** | `memory/__init__.py:L217-324` | `sqlite/__init__.py:L288-378` |
| **put** | `memory/__init__.py:L326-370` | `sqlite/__init__.py:L380-436` |
| **put_writes** | `memory/__init__.py:L372-408` | `sqlite/__init__.py:L438-475` |
| **delete_thread** | `memory/__init__.py:L410-426` | `sqlite/__init__.py:L477-494` |
| **setup** | 无需 | `sqlite/__init__.py:L122-159` |
| **search_where** | N/A | `sqlite/utils.py:L76-116` |

---

## 七、Linus 式评价

### ✅ InMemorySaver 做对的

1. **简单直接** - 嵌套字典是最朴素的实现
2. **Blobs 分离** - 避免重复存储大对象
3. **测试友好** - 零配置,开箱即用

### ❌ InMemorySaver 的问题

1. **max() 是笑话** - 查询最新 checkpoint 用 O(n) 遍历,应该维护一个 `latest_checkpoint_id` 变量
2. **sorted() 太重** - `list()` 每次都全排序,应该用 heapq 或者预排序
3. **无 GC 策略** - 旧 checkpoint 永远不会被清理

### ✅ SqliteSaver 做对的

1. **UUID v6 排序** - 直接用字符串排序,不需要额外时间戳
2. **UPSERT 简化** - INSERT OR REPLACE 一条语句搞定
3. **事务自动化** - cursor() 上下文管理器自动 commit

### ❌ SqliteSaver 的问题

1. **Metadata 查询是笑话** - `json_extract()` 全表扫描,生产环境别用
2. **每次 put 都 commit** - 批量写入时应该手动控制事务
3. **SQLite 不适合并发** - 写锁阻塞,多 worker 场景直接崩溃

---

**最终建议:**
- 测试/演示 → `InMemorySaver`
- 本地开发 → `SqliteSaver`
- 生产环境 → `PostgresSaver` (别浪费时间优化 SQLite)
