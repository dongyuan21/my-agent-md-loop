# Human-in-the-Loop 中断与恢复实战

> **场景**: Agent 在执行危险操作前需要用户批准,如删除文件、执行 Shell 命令等

---

## 一、核心原理

### 1.1 HITL 流程图

```text
┌─────────────────┐
│ 用户: 删除文件   │
└────────┬────────┘
         │
         ▼
┌────────────────────────────────────┐
│ Agent 决策: 调用 write_file 工具   │
│ (args: {path: "/file.txt",        │
│         content: None})            │  ← content=None 表示删除
└────────┬───────────────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│ 检测到工具在 interrupt_on 列表     │
│ → 触发 NodeInterrupt               │
└────────┬───────────────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│ Checkpoint 保存中断状态            │
│ - checkpoint_id: "abc123"         │
│ - pending_writes: [               │
│     (INTERRUPT, tool_call_data)   │
│   ]                               │
└────────┬───────────────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│ 返回 __interrupt__ 给用户          │
│ {                                 │
│   "action_requests": [...],       │
│   "review_configs": [...]         │
│ }                                 │
└────────┬───────────────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│ 用户审批: approve / reject / edit  │
└────────┬───────────────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│ 使用 Command(resume={...})        │
│ 从同一 checkpoint_id 恢复执行      │
└────────┬───────────────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│ 执行批准的工具调用                 │
│ → 完成删除操作                     │
└────────────────────────────────────┘
```

---

## 二、源码位置

### 2.1 核心文件

```text
libs/deepagents/deepagents/
├── middleware/
│   └── human_in_the_loop.py         # HITL 中间件实现
├── graph.py                          # create_deep_agent 集成 HITL
└── types.py                          # ActionRequest, Decision 类型定义

libs/deepagents-cli/deepagents_cli/
├── execution.py                      # prompt_for_tool_approval 用户交互
└── agent.py                          # _add_interrupt_on 配置哪些工具需要中断

libs/langchain/
└── agents/middleware.py              # HumanInTheLoopMiddleware 基类

langgraph-main/libs/langgraph/
└── pregel/loop.py                    # 检测 NodeInterrupt 并保存 checkpoint
```

### 2.2 关键代码行数

| 文件 | 函数/类 | 行数 | 说明 |
|------|--------|------|------|
| `deepagents/middleware/human_in_the_loop.py` | `HumanInTheLoopMiddleware` | 全文 | HITL 中间件实现 |
| `deepagents-cli/execution.py` | `prompt_for_tool_approval` | L40-178 | 用户交互界面 |
| `deepagents-cli/execution.py` | `execute_task` | L181-698 | 处理中断循环 |
| `deepagents-cli/agent.py` | `_add_interrupt_on` | L279-323 | 配置中断规则 |
| `deepagents/graph.py` | `create_deep_agent` | L40-161 | 集成 HITL 中间件 |

---

## 三、实战代码示例

### 3.1 最小化示例 (测试用)

```python
# libs/deepagents/tests/integration_tests/test_hitl.py
from langgraph.checkpoint.memory import MemorySaver
from deepagents.graph import create_deep_agent
from langgraph.types import Command
import uuid

# 1️⃣ 定义需要中断的工具配置
SAMPLE_TOOL_CONFIG = {
    "sample_tool": {
        "allowed_decisions": ["approve", "edit", "reject"],
        "description": lambda args: f"Sample tool with args: {args}"
    },
    "get_soccer_scores": {
        "allowed_decisions": ["approve", "reject"],  # 不允许 edit
        "description": lambda args: "Get latest soccer scores"
    }
}

# 2️⃣ 创建带 HITL 的 Agent
checkpointer = MemorySaver()
agent = create_deep_agent(
    tools=[sample_tool, get_weather, get_soccer_scores],
    interrupt_on=SAMPLE_TOOL_CONFIG,  # ⚠️ 指定哪些工具需要中断
    checkpointer=checkpointer           # ⚠️ 必须提供 checkpointer
)

# 3️⃣ 第一阶段: 执行到中断点
config = {"configurable": {"thread_id": uuid.uuid4()}}
result = agent.invoke(
    {
        "messages": [{
            "role": "user",
            "content": "Call the sample tool, get the weather in New York and "
                      "get scores for the latest soccer games in parallel"
        }]
    },
    config=config
)

# 4️⃣ 检查中断状态
assert result["__interrupt__"] is not None
interrupts = result["__interrupt__"][0].value

# 中断数据结构:
# {
#   "action_requests": [
#     {
#       "name": "sample_tool",
#       "args": {...},
#       "description": "Sample tool with args: {...}"
#     },
#     {
#       "name": "get_soccer_scores",
#       "args": {},
#       "description": "Get latest soccer scores"
#     }
#   ],
#   "review_configs": [
#     {
#       "action_name": "sample_tool",
#       "allowed_decisions": ["approve", "edit", "reject"]
#     },
#     {
#       "action_name": "get_soccer_scores",
#       "allowed_decisions": ["approve", "reject"]
#     }
#   ]
# }

# 5️⃣ 用户审批后恢复
result2 = agent.invoke(
    Command(resume={
        "decisions": [
            {"type": "approve"},  # 批准 sample_tool
            {"type": "approve"}   # 批准 get_soccer_scores
        ]
    }),
    config=config  # ⚠️ 必须使用同一个 thread_id
)

# 6️⃣ 验证工具执行成功
tool_results = [msg for msg in result2.get("messages", []) if msg.type == "tool"]
assert any([tool_result.name == "sample_tool" for tool_result in tool_results])
assert any([tool_result.name == "get_weather" for tool_result in tool_results])
assert "__interrupt__" not in result2  # 已经没有中断了
```

---

### 3.2 生产环境示例 (DeepAgents CLI)

```python
# libs/deepagents-cli/deepagents_cli/execution.py (简化版)

async def execute_task(user_input: str, agent, assistant_id: str, session_state):
    """执行任务,支持 HITL 中断循环"""
    
    config = {"configurable": {"thread_id": session_state.thread_id}}
    stream_input = {"messages": [{"role": "user", "content": user_input}]}
    
    # ===== HITL 循环 =====
    while True:
        interrupt_occurred = False
        hitl_response = {}
        
        # 流式执行 Agent
        async for chunk in agent.astream(
            stream_input,
            stream_mode=["messages", "updates"],  # ⚠️ 双模式监听
            config=config
        ):
            _namespace, stream_mode, data = chunk
            
            # ===== 监听中断 =====
            if stream_mode == "updates" and "__interrupt__" in data:
                interrupts = data["__interrupt__"]
                for interrupt_obj in interrupts:
                    interrupt_id = interrupt_obj.id
                    hitl_request = interrupt_obj.value
                    
                    # 检查是否自动批准模式
                    if session_state.auto_approve:
                        # 自动批准所有工具
                        decisions = [{"type": "approve"} 
                                   for _ in hitl_request["action_requests"]]
                        hitl_response[interrupt_id] = {"decisions": decisions}
                    else:
                        # 用户手动审批
                        decisions = []
                        for action_request in hitl_request["action_requests"]:
                            decision = prompt_for_tool_approval(
                                action_request, 
                                assistant_id
                            )
                            
                            # 检查是否切换到自动批准
                            if decision.get("type") == "auto_approve_all":
                                session_state.auto_approve = True
                                console.print("✓ Auto-approve mode enabled")
                                # 批准当前及后续所有操作
                                decisions.append({"type": "approve"})
                                break
                            
                            decisions.append(decision)
                        
                        hitl_response[interrupt_id] = {"decisions": decisions}
                    
                    interrupt_occurred = True
            
            # ===== 处理消息流 =====
            elif stream_mode == "messages":
                message, _metadata = data
                # 显示工具调用、AI 回复等
                ...
        
        # ===== 恢复执行 =====
        if interrupt_occurred and hitl_response:
            # 检查是否有拒绝
            if any(d.get("type") == "reject" for r in hitl_response.values() 
                   for d in r["decisions"]):
                console.print("[yellow]Command rejected.[/yellow]")
                return
            
            # 用决策恢复执行
            stream_input = Command(resume=hitl_response)
            # 继续循环
        else:
            # 没有中断,正常结束
            break
```

---

### 3.3 用户交互界面 (箭头键选择)

```python
# libs/deepagents-cli/deepagents_cli/execution.py:L40-178

def prompt_for_tool_approval(
    action_request: ActionRequest,
    assistant_id: str | None
) -> Decision | dict:
    """提示用户批准/拒绝工具操作,支持箭头键导航
    
    返回:
        Decision (ApproveDecision 或 RejectDecision) 或
        dict {"type": "auto_approve_all"} 切换到自动批准模式
    """
    description = action_request.get("description", "No description available")
    name = action_request["name"]
    args = action_request["args"]
    
    # 构建预览 (显示文件 diff 等)
    preview = build_approval_preview(name, args, assistant_id)
    
    # 显示操作信息
    console.print(
        Panel(
            "[bold yellow]⚠️  Tool Action Requires Approval[/bold yellow]\n\n"
            + preview.title + "\n" + "\n".join(preview.details),
            border_style="yellow"
        )
    )
    
    # 如果有 diff,显示差异
    if preview.diff:
        render_diff_block(preview.diff, preview.diff_title)
    
    # 选项列表
    options = ["approve", "reject", "auto-accept all going forward"]
    selected = 0  # 默认选择 approve
    
    # 进入终端原始模式 (捕获箭头键)
    fd = sys.stdin.fileno()
    old_settings = termios.tcgetattr(fd)
    tty.setraw(fd)
    sys.stdout.write("\033[?25l")  # 隐藏光标
    
    try:
        while True:
            # 显示选项 (带颜色和复选框)
            for i, option in enumerate(options):
                sys.stdout.write("\r\033[K")  # 清除当前行
                
                if i == selected:
                    if option == "approve":
                        sys.stdout.write("\033[1;32m☑ Approve\033[0m\n")
                    elif option == "reject":
                        sys.stdout.write("\033[1;31m☑ Reject\033[0m\n")
                    else:
                        sys.stdout.write("\033[1;34m☑ Auto-accept all\033[0m\n")
                else:
                    sys.stdout.write(f"\033[2m☐ {option.capitalize()}\033[0m\n")
            
            sys.stdout.flush()
            
            # 读取按键
            char = sys.stdin.read(1)
            
            if char == "\x1b":  # ESC 序列 (箭头键)
                next1 = sys.stdin.read(1)
                next2 = sys.stdin.read(1)
                if next1 == "[":
                    if next2 == "B":  # 下箭头
                        selected = (selected + 1) % len(options)
                    elif next2 == "A":  # 上箭头
                        selected = (selected - 1) % len(options)
            elif char in {"\r", "\n"}:  # Enter
                break
            elif char == "\x03":  # Ctrl+C
                raise KeyboardInterrupt
    finally:
        sys.stdout.write("\033[?25h")  # 显示光标
        termios.tcsetattr(fd, termios.TCSADRAIN, old_settings)
    
    # 返回决策
    if selected == 0:
        return ApproveDecision(type="approve")
    elif selected == 1:
        return RejectDecision(type="reject", message="User rejected")
    else:
        return {"type": "auto_approve_all"}
```

---

## 四、配置中断规则

### 4.1 DeepAgents CLI 的默认配置

```python
# libs/deepagents-cli/deepagents_cli/agent.py:L279-323

def _add_interrupt_on() -> dict[str, InterruptOnConfig]:
    """配置破坏性工具的 HITL 设置"""
    
    return {
        # Shell 命令 - 仅批准/拒绝
        "shell": {
            "allowed_decisions": ["approve", "reject"],
            "description": _format_shell_description,
        },
        
        # 沙箱执行 - 仅批准/拒绝
        "execute": {
            "allowed_decisions": ["approve", "reject"],
            "description": _format_execute_description,
        },
        
        # 写文件 - 仅批准/拒绝
        "write_file": {
            "allowed_decisions": ["approve", "reject"],
            "description": _format_write_file_description,
        },
        
        # 编辑文件 - 仅批准/拒绝
        "edit_file": {
            "allowed_decisions": ["approve", "reject"],
            "description": _format_edit_file_description,
        },
        
        # 网络搜索 - 仅批准/拒绝
        "web_search": {
            "allowed_decisions": ["approve", "reject"],
            "description": _format_web_search_description,
        },
        
        # 获取 URL - 仅批准/拒绝
        "fetch_url": {
            "allowed_decisions": ["approve", "reject"],
            "description": _format_fetch_url_description,
        },
        
        # 子任务 - 仅批准/拒绝
        "task": {
            "allowed_decisions": ["approve", "reject"],
            "description": _format_task_description,
        },
    }

# Description 格式化示例
def _format_shell_description(args: dict) -> str:
    """格式化 shell 命令描述"""
    command = args.get("command", "")
    return f"Execute shell command: {command}"

def _format_write_file_description(args: dict) -> str:
    """格式化写文件描述"""
    path = args.get("path", "")
    content = args.get("content")
    
    if content is None:
        return f"Delete file: {path}"
    else:
        lines = len(content.split("\n")) if isinstance(content, str) else "?"
        return f"Write {lines} lines to {path}"
```

### 4.2 自定义配置 (支持 edit 决策)

```python
# 支持编辑参数的配置
custom_interrupt_on = {
    "dangerous_tool": {
        "allowed_decisions": ["approve", "edit", "reject"],
        "description": lambda args: f"Dangerous operation: {args}"
    }
}

agent = create_deep_agent(
    tools=[dangerous_tool],
    interrupt_on=custom_interrupt_on,
    checkpointer=MemorySaver()
)

# 用户可以选择:
# 1. approve → 执行原始参数
# 2. edit → 修改参数后执行 (需要前端实现编辑界面)
# 3. reject → 拒绝执行
```

---

## 五、Checkpoint 中的中断数据

### 5.1 中断状态存储

```python
# 中断发生时,LangGraph 会调用:
await checkpointer.aput_writes(
    config,
    writes=[
        (INTERRUPT, {
            "action_requests": [...],
            "review_configs": [...]
        })
    ],
    task_id="__interrupt__"
)

# 存储在 SQLite 的 writes 表:
# | thread_id | checkpoint_id | task_id       | idx | channel    | value      |
# |-----------|---------------|---------------|-----|------------|------------|
# | session-1 | abc123        | __interrupt__ | -3  | INTERRUPT  | <binary>   |
#                                                     ↑
#                                          WRITES_IDX_MAP[INTERRUPT] = -3
```

### 5.2 恢复时的 Checkpoint 读取

```python
# libs/deepagents-cli/deepagents_cli/execution.py

# 恢复执行时,LangGraph 会:
checkpoint_tuple = await checkpointer.aget_tuple(config)

# 返回的 CheckpointTuple 包含:
# CheckpointTuple(
#     config={"configurable": {"thread_id": "session-1", "checkpoint_id": "abc123"}},
#     checkpoint={
#         "id": "abc123",
#         "channel_values": {"messages": [...], "files": {...}},
#         ...
#     },
#     metadata={"step": 5, "source": "loop"},
#     pending_writes=[
#         ("__interrupt__", "__interrupt__", {
#             "action_requests": [...],
#             "review_configs": [...]
#         })
#     ]
# )

# Agent 从 pending_writes 中提取中断信息,
# 结合 Command(resume={...}) 的决策继续执行
```

---

## 六、常见问题

### Q1: 如果用户关闭终端,中断状态会丢失吗?

**A:** 取决于 Checkpointer:

```python
# ❌ InMemorySaver - 进程重启后丢失
checkpointer = InMemorySaver()

# ✅ SqliteSaver - 持久化到文件
checkpointer = SqliteSaver.from_conn_string("checkpoints.db")

# ✅ PostgresSaver - 持久化到数据库 (推荐)
checkpointer = await PostgresSaver.from_conn_string("postgresql://...")
```

用户可以:
1. 重启程序
2. 用同一个 `thread_id` 继续对话
3. Agent 会检测到 `pending_writes` 中的中断
4. 重新提示用户审批

---

### Q2: 如何实现"部分批准"?

**A:** 使用不同的 `interrupt_id`:

```python
# 每个工具调用对应一个 interrupt_id
hitl_response = {
    "interrupt-1": {"decisions": [{"type": "approve"}]},  # 批准第一个
    "interrupt-2": {"decisions": [{"type": "reject"}]},   # 拒绝第二个
}

result = agent.invoke(Command(resume=hitl_response), config=config)
```

---

### Q3: 如何跳过中断 (自动批准)?

**A:** 两种方式:

```python
# 方式1: 创建 Agent 时禁用 HITL
agent = create_deep_agent(
    tools=[...],
    interrupt_on={},  # 空字典 = 不中断任何工具
    checkpointer=checkpointer
)

# 方式2: 运行时切换 auto_approve 模式 (DeepAgents CLI)
# 用户在第一次中断时选择 "Auto-accept all going forward"
# session_state.auto_approve = True
# 后续所有工具自动批准
```

---

## 七、测试用例参考

```python
# libs/deepagents/tests/integration_tests/test_hitl.py

class TestHITL:
    def test_hitl_agent(self):
        """测试基本 HITL 流程"""
        # 创建 Agent
        checkpointer = MemorySaver()
        agent = create_deep_agent(
            tools=[sample_tool, get_weather, get_soccer_scores],
            interrupt_on=SAMPLE_TOOL_CONFIG,
            checkpointer=checkpointer
        )
        
        # 执行到中断
        config = {"configurable": {"thread_id": uuid.uuid4()}}
        result = agent.invoke({
            "messages": [{"role": "user", "content": "..."}]
        }, config=config)
        
        # 验证中断
        assert result["__interrupt__"] is not None
        
        # 恢复执行
        result2 = agent.invoke(
            Command(resume={"decisions": [{"type": "approve"}, {"type": "approve"}]}),
            config=config
        )
        assert "__interrupt__" not in result2
    
    def test_subagent_with_hitl(self):
        """测试子 Agent 的 HITL 传播"""
        # 子 Agent 继承父 Agent 的 interrupt_on 配置
        agent = create_deep_agent(
            tools=[...],
            interrupt_on=SAMPLE_TOOL_CONFIG,
            checkpointer=checkpointer
        )
        
        result = agent.invoke({
            "messages": [{
                "role": "user",
                "content": "Use the task tool to kick off the general-purpose subagent"
            }]
        }, config=config)
        
        # 子 Agent 调用的工具也会触发中断
        assert result["__interrupt__"] is not None
    
    def test_subagent_with_custom_interrupt_on(self):
        """测试子 Agent 自定义 interrupt_on"""
        agent = create_deep_agent(
            tools=[sample_tool, get_weather],
            interrupt_on={"sample_tool": True, "get_weather": True},
            checkpointer=checkpointer,
            subagents=[{
                "name": "task_handler",
                "description": "...",
                "system_prompt": "...",
                "tools": [sample_tool, get_weather],
                "interrupt_on": {
                    "sample_tool": False,      # 子 Agent 不中断 sample_tool
                    "get_weather": True        # 但仍中断 get_weather
                }
            }]
        )
        
        # 验证只有 get_weather 被中断
        ...
```

---

## 八、源码引用总结

| 功能 | 文件 | 行数 | 说明 |
|------|------|------|------|
| HITL 中间件 | `deepagents/middleware/human_in_the_loop.py` | 全文 | 拦截工具调用,触发中断 |
| 用户交互 | `deepagents-cli/execution.py` | L40-178 | 箭头键选择界面 |
| 中断循环 | `deepagents-cli/execution.py` | L181-698 | 流式监听中断,恢复执行 |
| 配置规则 | `deepagents-cli/agent.py` | L279-323 | 定义哪些工具需要中断 |
| 集成 HITL | `deepagents/graph.py` | L40-161 | create_deep_agent 添加中间件 |
| 测试用例 | `deepagents/tests/integration_tests/test_hitl.py` | L18-154 | 完整测试流程 |

---

**核心要点:**
1. **必须提供 Checkpointer** - 中断状态需要持久化
2. **thread_id 必须一致** - 恢复时使用同一个 thread_id
3. **使用 Command(resume={...})** - 传递用户决策
4. **监听 `__interrupt__`** - 在 `updates` stream_mode 中检测
5. **支持批量中断** - 一次可以处理多个工具调用
