# 时间旅行 (Time Travel) 实战

> **场景**: 回溯 Agent 的历史状态,从任意 checkpoint 重新执行,创建状态分支

---

## 一、核心原理

### 1.1 时间旅行流程图

```text
User Input 1         User Input 2         User Input 3
     │                    │                    │
     ▼                    ▼                    ▼
┌─────────┐          ┌─────────┐          ┌─────────┐
│  CP-1   │──────────│  CP-2   │──────────│  CP-3   │  ← 主线
│ step=-1 │          │ step=0  │          │ step=1  │
└─────────┘          └─────────┘          └─────────┘
                           │
                           │ Time Travel: 从 CP-2 创建分支
                           ▼
                      ┌─────────┐          ┌─────────┐
                      │  CP-4   │──────────│  CP-5   │  ← 新分支
                      │ step=0' │          │ step=1' │
                      └─────────┘          └─────────┘
                           ↑
                      使用 checkpoint_id="CP-2"
                      + 新的输入
```

**关键概念:**
- **Checkpoint ID** - UUID v6 (时间递增,可排序)
- **Thread ID** - 对话线程标识
- **Step** - 执行步骤编号 (-1=初始输入, 0=第一个 loop, ...)
- **Parent Checkpoint ID** - 父 checkpoint 的 ID (支持分支)

---

## 二、源码位置

### 2.1 核心文件

```text
langgraph-main/libs/langgraph/
└── graph/
    └── state.py                    # CompiledStateGraph.get_state
                                    # CompiledStateGraph.update_state

langgraph-main/libs/checkpoint/
└── langgraph/checkpoint/
    ├── base/__init__.py           # BaseCheckpointSaver.list
    └── sqlite/__init__.py         # SqliteSaver.list 实现

libs/deepagents-cli/
└── deepagents_cli/
    └── main.py                    # CLI 主程序 (可扩展时间旅行命令)
```

### 2.2 关键 API

| API | 说明 | 源码位置 |
|-----|------|----------|
| `agent.get_state(config)` | 获取指定 checkpoint 的状态 | `langgraph/graph/state.py` |
| `agent.update_state(config, values)` | 从指定 checkpoint 创建新分支 | `langgraph/graph/state.py` |
| `checkpointer.list(config)` | 列出所有历史 checkpoint | `checkpoint/base/__init__.py` |
| `checkpointer.get_tuple(config)` | 获取单个 checkpoint 详情 | `checkpoint/base/__init__.py` |

---

## 三、实战代码示例

### 3.1 基础时间旅行 - 查看历史

```python
from langgraph.checkpoint.sqlite import SqliteSaver
from deepagents.graph import create_deep_agent
import sqlite3

# 1️⃣ 创建持久化 checkpointer
conn = sqlite3.connect("checkpoints.db", check_same_thread=False)
checkpointer = SqliteSaver(conn)
checkpointer.setup()

# 2️⃣ 创建 Agent
agent = create_deep_agent(
    tools=[read_file, write_file],
    checkpointer=checkpointer
)

# 3️⃣ 执行多轮对话
config = {"configurable": {"thread_id": "session-123"}}

agent.invoke({"messages": [{"role": "user", "content": "创建 hello.txt"}]}, config)
agent.invoke({"messages": [{"role": "user", "content": "修改 hello.txt 添加内容"}]}, config)
agent.invoke({"messages": [{"role": "user", "content": "删除 hello.txt"}]}, config)

# 4️⃣ 查看所有历史 checkpoint
history = list(checkpointer.list(config))

print(f"总共 {len(history)} 个 checkpoint:\n")
for i, checkpoint_tuple in enumerate(history):
    cp = checkpoint_tuple.checkpoint
    meta = checkpoint_tuple.metadata
    
    print(f"[{i}] Checkpoint ID: {cp['id']}")
    print(f"    Time: {cp['ts']}")
    print(f"    Step: {meta.get('step', 'N/A')}")
    print(f"    Source: {meta.get('source', 'N/A')}")
    
    # 提取消息内容
    messages = cp['channel_values'].get('messages', [])
    if messages:
        last_msg = messages[-1]
        content = last_msg.content if hasattr(last_msg, 'content') else str(last_msg)
        preview = content[:50] + "..." if len(content) > 50 else content
        print(f"    Last Message: {preview}")
    
    print()

# 输出示例:
# 总共 4 个 checkpoint:
#
# [0] Checkpoint ID: 1ed2a3b4-c5d6-6789-abcd-ef0123456789
#     Time: 2024-01-15T10:30:45.123456Z
#     Step: 2
#     Source: loop
#     Last Message: 已删除文件 hello.txt
#
# [1] Checkpoint ID: 1ed2a3b4-c5d5-6789-abcd-ef0123456788
#     Time: 2024-01-15T10:30:30.123456Z
#     Step: 1
#     Source: loop
#     Last Message: 已修改文件 hello.txt,添加了新内容
#
# [2] Checkpoint ID: 1ed2a3b4-c5d4-6789-abcd-ef0123456787
#     Time: 2024-01-15T10:30:15.123456Z
#     Step: 0
#     Source: loop
#     Last Message: 已创建文件 hello.txt
#
# [3] Checkpoint ID: 1ed2a3b4-c5d3-6789-abcd-ef0123456786
#     Time: 2024-01-15T10:30:00.123456Z
#     Step: -1
#     Source: input
#     Last Message: 创建 hello.txt
```

---

### 3.2 时间旅行 - 回到过去

```python
# 5️⃣ 回到第 2 个 checkpoint (文件被修改后)
history = list(checkpointer.list(config))
checkpoint_id_at_step_1 = history[1].checkpoint["id"]

# 使用 get_state 查看该时间点的状态
past_config = {
    "configurable": {
        "thread_id": "session-123",
        "checkpoint_id": checkpoint_id_at_step_1  # ⚠️ 指定具体的 checkpoint
    }
}

past_state = agent.get_state(past_config)

print("=== 回到过去的状态 ===")
print(f"Checkpoint ID: {past_state.config['configurable']['checkpoint_id']}")
print(f"Files at that time:")
files = past_state.values.get('files', {})
for path, file_data in files.items():
    print(f"  {path}: {file_data.content[:50]}...")

# 输出:
# === 回到过去的状态 ===
# Checkpoint ID: 1ed2a3b4-c5d5-6789-abcd-ef0123456788
# Files at that time:
#   /hello.txt: ['Hello World', 'This is new content']...
```

---

### 3.3 创建分支 - 从历史点重新执行

```python
# 6️⃣ 从 step=1 的 checkpoint 创建新分支
# 场景: 用户后悔删除文件,想从"修改后"的状态重新开始

# 方式1: 使用 update_state 修改状态
agent.update_state(
    config=past_config,  # ⚠️ 使用历史 checkpoint 的 config
    values={
        "messages": [{"role": "user", "content": "[用户撤回了删除操作]"}]
    }
)

# 方式2: 直接从历史 checkpoint 继续对话 (创建 fork)
result = agent.invoke(
    {"messages": [{"role": "user", "content": "不要删除,而是追加更多内容"}]},
    config=past_config  # ⚠️ 使用历史 checkpoint 的 config
)

# 7️⃣ 查看新分支的历史
new_history = list(checkpointer.list(config))

print("=== 新分支的历史 ===")
for i, cp_tuple in enumerate(new_history):
    cp = cp_tuple.checkpoint
    meta = cp_tuple.metadata
    parent_id = cp_tuple.parent_config['configurable'].get('checkpoint_id') if cp_tuple.parent_config else None
    
    print(f"[{i}] Step {meta.get('step')}: {cp['id'][:20]}...")
    if parent_id:
        print(f"    Parent: {parent_id[:20]}...")

# 输出:
# === 新分支的历史 ===
# [0] Step 2: 1ed2a3b4-c5d7-6789...  ← 新分支的新 checkpoint
#     Parent: 1ed2a3b4-c5d5-6789...  ← 指向 step=1 的 checkpoint
# [1] Step 1: 1ed2a3b4-c5d5-6789...  ← 分叉点
#     Parent: 1ed2a3b4-c5d4-6789...
# [2] Step 0: 1ed2a3b4-c5d4-6789...
#     Parent: 1ed2a3b4-c5d3-6789...
# [3] Step -1: 1ed2a3b4-c5d3-6789...
#     Parent: None
```

---

### 3.4 可视化历史树 (进阶)

```python
def visualize_checkpoint_tree(checkpointer, thread_id: str):
    """可视化 checkpoint 的分支树"""
    config = {"configurable": {"thread_id": thread_id}}
    history = list(checkpointer.list(config))
    
    # 构建 parent-child 关系
    nodes = {}
    for cp_tuple in history:
        cp_id = cp_tuple.checkpoint["id"]
        parent_id = (cp_tuple.parent_config["configurable"].get("checkpoint_id") 
                    if cp_tuple.parent_config else None)
        
        nodes[cp_id] = {
            "step": cp_tuple.metadata.get("step"),
            "parent": parent_id,
            "children": []
        }
    
    # 建立 children 链接
    for cp_id, node in nodes.items():
        if node["parent"] in nodes:
            nodes[node["parent"]]["children"].append(cp_id)
    
    # 递归打印树
    def print_tree(cp_id, indent=0):
        node = nodes[cp_id]
        prefix = "  " * indent + ("├─ " if indent > 0 else "")
        print(f"{prefix}Step {node['step']}: {cp_id[:12]}...")
        
        for child_id in node["children"]:
            print_tree(child_id, indent + 1)
    
    # 找到根节点 (没有 parent 的)
    root_ids = [cp_id for cp_id, node in nodes.items() if node["parent"] is None]
    
    for root_id in root_ids:
        print_tree(root_id)

# 调用
visualize_checkpoint_tree(checkpointer, "session-123")

# 输出:
# Step -1: 1ed2a3b4-c5d3...
#   ├─ Step 0: 1ed2a3b4-c5d4...
#     ├─ Step 1: 1ed2a3b4-c5d5...
#       ├─ Step 2: 1ed2a3b4-c5d6...  ← 主线 (删除文件)
#       ├─ Step 2: 1ed2a3b4-c5d7...  ← 分支1 (追加内容)
```

---

## 四、高级用法

### 4.1 按 Metadata 过滤历史

```python
# 只查询 "loop" 来源的 checkpoint (排除用户输入)
history = list(checkpointer.list(
    config,
    filter={"source": "loop"}  # ⚠️ SQLite 不支持复杂查询,需要 PostgreSQL
))

# PostgreSQL 支持复杂查询:
# SELECT * FROM checkpoints WHERE metadata @> '{"source": "loop"}'::jsonb
```

---

### 4.2 限制查询数量 (分页)

```python
# 只获取最近 5 个 checkpoint
recent_history = list(checkpointer.list(config, limit=5))

# 获取某个 checkpoint 之前的历史
before_config = {
    "configurable": {
        "thread_id": "session-123",
        "checkpoint_id": "1ed2a3b4-c5d5-6789-abcd-ef0123456788"
    }
}

older_history = list(checkpointer.list(
    config,
    before=before_config,  # ⚠️ 只返回比这个 checkpoint 更早的
    limit=10
))
```

---

### 4.3 比较两个 Checkpoint 的差异

```python
def compare_checkpoints(checkpointer, thread_id: str, cp_id_1: str, cp_id_2: str):
    """比较两个 checkpoint 的状态差异"""
    
    config_1 = {"configurable": {"thread_id": thread_id, "checkpoint_id": cp_id_1}}
    config_2 = {"configurable": {"thread_id": thread_id, "checkpoint_id": cp_id_2}}
    
    cp_tuple_1 = checkpointer.get_tuple(config_1)
    cp_tuple_2 = checkpointer.get_tuple(config_2)
    
    state_1 = cp_tuple_1.checkpoint["channel_values"]
    state_2 = cp_tuple_2.checkpoint["channel_values"]
    
    print("=== Checkpoint 差异 ===")
    print(f"Checkpoint 1: {cp_id_1[:20]}... (Step {cp_tuple_1.metadata.get('step')})")
    print(f"Checkpoint 2: {cp_id_2[:20]}... (Step {cp_tuple_2.metadata.get('step')})")
    print()
    
    # 比较 messages
    messages_1 = state_1.get("messages", [])
    messages_2 = state_2.get("messages", [])
    print(f"Messages: {len(messages_1)} → {len(messages_2)} (Δ{len(messages_2) - len(messages_1)})")
    
    # 比较 files
    files_1 = state_1.get("files", {})
    files_2 = state_2.get("files", {})
    
    added_files = set(files_2.keys()) - set(files_1.keys())
    removed_files = set(files_1.keys()) - set(files_2.keys())
    modified_files = {
        path for path in files_1.keys() & files_2.keys()
        if files_1[path] != files_2[path]
    }
    
    if added_files:
        print(f"Added files: {added_files}")
    if removed_files:
        print(f"Removed files: {removed_files}")
    if modified_files:
        print(f"Modified files: {modified_files}")

# 使用
compare_checkpoints(checkpointer, "session-123", 
                   "1ed2a3b4-c5d4-6789-abcd-ef0123456787",  # Step 0
                   "1ed2a3b4-c5d5-6789-abcd-ef0123456788")  # Step 1

# 输出:
# === Checkpoint 差异 ===
# Checkpoint 1: 1ed2a3b4-c5d4-6789... (Step 0)
# Checkpoint 2: 1ed2a3b4-c5d5-6789... (Step 1)
#
# Messages: 3 → 5 (Δ2)
# Modified files: {'/hello.txt'}
```

---

## 五、实际应用场景

### 5.1 调试 Agent 行为

```python
# 场景: Agent 在第 5 步做了错误决策,需要复现和调试

# 1. 回到第 4 步
config_step_4 = {
    "configurable": {
        "thread_id": "debug-session",
        "checkpoint_id": get_checkpoint_at_step(checkpointer, "debug-session", 4)
    }
}

# 2. 查看当时的状态
state = agent.get_state(config_step_4)
print("State at step 4:")
print(f"  Messages: {len(state.values['messages'])}")
print(f"  Todos: {state.values.get('todos', [])}")

# 3. 重新执行第 5 步 (可能得到不同结果)
result = agent.invoke(
    {"messages": [{"role": "user", "content": "[重新执行]"}]},
    config=config_step_4
)
```

---

### 5.2 A/B 测试不同策略

```python
# 场景: 从同一起点测试两种不同的 prompt

base_checkpoint_id = get_checkpoint_at_step(checkpointer, "session-1", 3)
base_config = {
    "configurable": {
        "thread_id": "session-1",
        "checkpoint_id": base_checkpoint_id
    }
}

# 策略 A: 直接的 prompt
agent_a = create_deep_agent(system_prompt="You are a helpful assistant.")
result_a = agent_a.invoke(
    {"messages": [{"role": "user", "content": "优化这段代码"}]},
    config=base_config
)

# 策略 B: 更详细的 prompt
agent_b = create_deep_agent(system_prompt="You are an expert code reviewer...")
result_b = agent_b.invoke(
    {"messages": [{"role": "user", "content": "优化这段代码"}]},
    config=base_config
)

# 比较结果
compare_results(result_a, result_b)
```

---

### 5.3 回滚到安全状态

```python
# 场景: Agent 误删除了重要文件,需要回滚

# 1. 查找删除操作前的最后一个 checkpoint
history = list(checkpointer.list(config))
for cp_tuple in history:
    files = cp_tuple.checkpoint["channel_values"].get("files", {})
    if "/important.txt" in files:
        # 找到了包含该文件的 checkpoint
        safe_checkpoint_id = cp_tuple.checkpoint["id"]
        break

# 2. 从安全点重新开始
safe_config = {
    "configurable": {
        "thread_id": "session-1",
        "checkpoint_id": safe_checkpoint_id
    }
}

# 3. 提醒 Agent 不要删除
result = agent.invoke(
    {"messages": [{"role": "user", "content": "继续工作,但不要删除任何文件"}]},
    config=safe_config
)
```

---

## 六、Checkpoint ID 的内部机制

### 6.1 UUID v6 的可排序性

```python
from uuid6 import uuid6
import time

# 生成多个 UUID v6
ids = []
for i in range(5):
    ids.append(str(uuid6()))
    time.sleep(0.001)

# 验证它们是递增的
print("UUID v6 is sortable:")
for i in range(len(ids) - 1):
    assert ids[i] < ids[i+1], f"{ids[i]} should be < {ids[i+1]}"
    print(f"{ids[i]} < {ids[i+1]} ✓")

# 这使得 SQLite 可以直接用字符串排序:
# SELECT * FROM checkpoints WHERE thread_id = ? ORDER BY checkpoint_id DESC
```

---

### 6.2 Checkpoint ID 在 SQLite 的存储

```sql
-- 查询某个 thread 的所有 checkpoint (按时间倒序)
SELECT 
    checkpoint_id,
    parent_checkpoint_id,
    metadata->>'$.step' AS step,
    metadata->>'$.source' AS source,
    datetime(checkpoint->>'$.ts') AS timestamp
FROM checkpoints
WHERE thread_id = 'session-123'
ORDER BY checkpoint_id DESC;  -- ⚠️ UUID v6 天然可排序

-- 输出:
-- checkpoint_id                        | parent_checkpoint_id              | step | source | timestamp
-- -------------------------------------|-----------------------------------|------|--------|-------------------
-- 1ed2a3b4-c5d6-6789-abcd-ef0123456789 | 1ed2a3b4-c5d5-6789-abcd-ef0123456788 | 2    | loop   | 2024-01-15 10:30:45
-- 1ed2a3b4-c5d5-6789-abcd-ef0123456788 | 1ed2a3b4-c5d4-6789-abcd-ef0123456787 | 1    | loop   | 2024-01-15 10:30:30
-- 1ed2a3b4-c5d4-6789-abcd-ef0123456787 | 1ed2a3b4-c5d3-6789-abcd-ef0123456786 | 0    | loop   | 2024-01-15 10:30:15
-- 1ed2a3b4-c5d3-6789-abcd-ef0123456786 | NULL                              | -1   | input  | 2024-01-15 10:30:00
```

---

## 七、常见问题

### Q1: 时间旅行会改变历史吗?

**A:** 不会! 时间旅行是**非破坏性**的:

```python
# 原始历史:
# CP-1 → CP-2 → CP-3 → CP-4

# 从 CP-2 创建分支:
result = agent.invoke(input, config={"configurable": {"checkpoint_id": "CP-2"}})

# 新历史:
# CP-1 → CP-2 → CP-3 → CP-4  ← 主线不变
#          └──→ CP-5         ← 新分支

# 所有旧 checkpoint 仍然存在
```

---

### Q2: 如何删除旧分支?

**A:** 手动清理 (LangGraph 暂无内置 API):

```python
# 删除某个 checkpoint 及其所有子孙
def delete_checkpoint_branch(checkpointer, thread_id: str, checkpoint_id: str):
    """递归删除 checkpoint 及其子分支"""
    
    # 1. 找出所有子孙 checkpoint
    config = {"configurable": {"thread_id": thread_id}}
    history = list(checkpointer.list(config))
    
    descendants = set()
    queue = [checkpoint_id]
    
    while queue:
        current_id = queue.pop(0)
        descendants.add(current_id)
        
        # 找出 parent 是 current_id 的 checkpoint
        for cp_tuple in history:
            parent_id = (cp_tuple.parent_config["configurable"].get("checkpoint_id")
                        if cp_tuple.parent_config else None)
            if parent_id == current_id:
                queue.append(cp_tuple.checkpoint["id"])
    
    # 2. 删除这些 checkpoint (直接操作数据库)
    import sqlite3
    conn = checkpointer.conn
    
    for desc_id in descendants:
        conn.execute(
            "DELETE FROM checkpoints WHERE thread_id = ? AND checkpoint_id = ?",
            (thread_id, desc_id)
        )
        conn.execute(
            "DELETE FROM writes WHERE thread_id = ? AND checkpoint_id = ?",
            (thread_id, desc_id)
        )
    
    conn.commit()
    print(f"Deleted {len(descendants)} checkpoints")
```

---

### Q3: 时间旅行会影响性能吗?

**A:** 取决于 Checkpointer:

```python
# ❌ InMemorySaver - 所有 checkpoint 都在内存,查询快但占内存
history = list(in_memory_saver.list(config))  # O(n) 遍历

# ✅ SqliteSaver - 查询有 I/O 开销,但可以用索引优化
history = list(sqlite_saver.list(config, limit=10))  # 只查 10 个,很快

# ✅ PostgresSaver - 支持高效索引查询
# CREATE INDEX idx_checkpoints_thread_id ON checkpoints(thread_id, checkpoint_id DESC);
```

---

## 八、源码引用总结

| 功能 | 文件 | 说明 |
|------|------|------|
| `get_state` | `langgraph/graph/state.py` | 获取指定 checkpoint 的状态 |
| `update_state` | `langgraph/graph/state.py` | 从指定 checkpoint 创建新分支 |
| `list` | `checkpoint/base/__init__.py` | 列出所有历史 checkpoint |
| `get_tuple` | `checkpoint/sqlite/__init__.py:L184-286` | 获取单个 checkpoint 详情 |
| UUID v6 生成 | `checkpoint/base/id.py` | 时间递增的 checkpoint ID |

---

**核心要点:**
1. **UUID v6 天然可排序** - 按 checkpoint_id 排序等价于按时间排序
2. **非破坏性分支** - 时间旅行不会改变历史,只会创建新分支
3. **Checkpoint 即快照** - 包含完整的 `channel_values` (messages, files, todos)
4. **Thread ID 是命名空间** - 不同 thread 的 checkpoint 相互隔离
5. **PostgreSQL 查询更强** - 支持 JSONB 过滤,SQLite 只能全表扫描
