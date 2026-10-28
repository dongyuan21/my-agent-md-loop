# 从五步 Agent Loop 演进到 Pi runLoop

> 协作底稿：你和 PIAgent的学习bot 都可以改这份文件。课上进度：已过 V0–V2；下一步讲 V3 beforeToolCall。

> 面向面试口述与宿主实现：用「一步一改」把会写的五步循环，长成 Pi 1.1.0 风格的 Harness。
>
> 对照源码：`packages/agent/src/agent-loop.ts` 的 `runLoop`（约 L163–321），版本 **pi 1.1.0**。

---

## 开篇：业界一句话，我们怎么长成宿主

业界对 Agent Loop 的短口径几乎是同一句：

> **LLM + tools in a loop**

模型说话、要工具就执行、把结果写回、再问模型——直到模型不再要工具，或预算用尽。

这句话够过面 1、面 2 开场。但虾皮面 3 爱抠「你是会画五个框，还是真写过宿主」。区别不在多背几个名词，而在控制面：跑着能不能插话、工具前有没有门禁、本轮停了还有没有下一意图排队。

本文只做一件事：**从一份会实现的五步伪代码出发，每版只加一个实际需求**，最后长得像 Pi 的 `runLoop`。不抄 300 行源码，只保留你面试能画、能讲的骨架。

你伪代码里的 `buildContext`，在 Pi 里会拆成 `prepareNextTurn` / `transformContext` / `convertToLlm` 一类钩子——上下文组装和压缩是 Harness 的职责，不是模型自己变聪明。这块我们约好以后细抠，文中先一笔带过。

---

## V0 — 最简五步（可口述开场）

先写「会跑」的版本：没有 toolCalls 就 `return`。

```ts
async function agentLoop(task: string, tools: Tool[]) {
  let messages = [{ role: "user", content: task }]
  let turn = 0
  const MAX_TURNS = 20

  while (turn < MAX_TURNS) {
    turn++
    const context = await buildContext(messages, tools)
    const reply = await callModel(context)   // 先问模型；宿主不「选」工具
    messages.push(reply)                     // assistant 写进 history

    if (!reply.toolCalls?.length) {
      return reply                           // 没工具 → 整函数结束
    }

    for (const call of reply.toolCalls) {
      const result = await execute(call)
      messages.push({                        // tool 角色写回
        role: "tool",
        toolCallId: call.id,
        content: result,
      })
    }
    // 有工具 → 下一圈再 callModel；注意：下一圈不再 push 开局那条 user
  }
}
```

口述时记住四件事：

1. **先 `callModel`**：工具是模型输出里带的，宿主只校验/执行，不替模型「选」。
2. **assistant 先入 history**，再跑工具。
3. **结果用 `role: "tool"` 写回**（带上 `toolCallId`），否则下一圈模型对不上调用。
4. **停机**：无 `toolCalls`，或 `MAX_TURNS` 顶满。

这已经是「会实现」的及格线。问题在下一拍：一旦要支持「跑着还能插话」，`return` 就把整条函数掐死了——连检查队列的机会都没有。

---

## V1 — 内层控制流：用标志位代替 `return`

**只改控制流，不加新功能。** 行为暂时和 V0 一样，但形状开始像 Pi 的内层：`hasMoreToolCalls` 为假时，是「本层该停了」，不是「函数立刻结束」。

```ts
async function agentLoop(task: string, tools: Tool[]) {
  let messages = [{ role: "user", content: task }]
  let turn = 0
  const MAX_TURNS = 20

  // 内层：有 toolCalls 就继续（steering 下一步再加）
  let hasMoreToolCalls = true
  while (hasMoreToolCalls && turn < MAX_TURNS) {
    turn++
    const context = await buildContext(messages, tools)
    const reply = await callModel(context)
    messages.push(reply)

    if (!reply.toolCalls?.length) {
      hasMoreToolCalls = false   // ← 不再 return，只是本层该停了
    } else {
      for (const call of reply.toolCalls) {
        const result = await execute(call)
        messages.push({ role: "tool", toolCallId: call.id, content: result })
      }
      // hasMoreToolCalls 仍是 true，下一圈再问模型
    }
  }
  return messages.at(-1)
}
```

相对 V0，**唯一实质变化**：`return reply` → `hasMoreToolCalls = false`，函数末尾再返回。

为什么要这一步？因为后面要加 steering：没有 toolCalls 时，还可能有用户插话在队列里。若直接 `return`，插话永远进不了当前轮。V1 把「停」从「退出函数」改成「退出条件可扩展」——这是内层长成双层调度的地基。

---

## V2 — 内层加 Steering

### 实际需求

Agent 还在转（等模型，或刚跑完工具、准备再问模型），用户突然插一句：「别查上月了，改成只要北京。」这句不能等整轮结束，要能插进**当前这轮**还没停的内层。

这就是 **steering（引导 / 打断）**：

| | Steering | Follow-up（后面 V5） |
|---|---|---|
| 何时消费 | **内层还在转** | **内层已经要停了** |
| 语义 | 改**当前轮**意图 | **下一意图**，本轮收束后再开 |
| 类比 Codex | 更像跑着还能「引导」 | 更像任务「排队」 |

不是一个队列两种叫法：**同一句用户输入，进哪条语义，取决于 Agent 当时停没停得下来。**

### 机制：不是 `await` 魔法

很多人问：用户输入怎么「立即」插进来？和 TypeScript 的 `await` 有关吗？

有关系，但分两截：

1. **收集**：`await callModel` / `await execute` 时，当前 loop 让出事件循环；UI 回调仍能收键盘，把消息推进一个**队列**。
2. **插入**：loop 从 await 回来后，在你写的**检查点**主动调 `getSteeringMessages()`，把队列捞进 `messages`。

所以：**`await` 只提供空档；插入点是 Harness 设的 checkpoint。** 没有第二步，队列里堆着也进不了下一轮模型。

Java 里同一套更直白——多线程 + 线程安全队列：

```java
BlockingQueue<Message> steeringQueue = new LinkedBlockingQueue<>();
// 线程 A：用户一打字就 offer
// 线程 B：agent 循环只在两个检查点 drain / poll
```

语言不同，控制面相同：**队列 + 检查点**。

### 为什么要两处捞？

用户可能落在两个不同空档：

1. **问模型前**：把已有的 `pending` 推进 history，再 `callModel`——否则模型还按旧意图答。
2. **工具跑完后再捞**：真正耗时的往往是 `execute`；用户多半在工具 I/O 时打字。漏了这里：
   - 若后面还有 toolCalls → 仍算 steering，但会先按错方向再调一次模型（晚半拍）；
   - 若已经没有 toolCalls → 内层直接停，这句插话会**丢**，或被外层当成 follow-up（语义也变了：不是改当前轮，而是停完再开）。

时间线：

```
callModel → execute（用户常在这里打字）→ ★再捞 → [下圈] 推进 history → callModel
```

### 代码（相对 V1，只加 steering）

```ts
async function agentLoop(task: string, tools: Tool[], config: {
  getSteeringMessages?: () => Promise<Message[]>
}) {
  let messages = [{ role: "user", content: task }]
  let turn = 0
  const MAX_TURNS = 20

  let hasMoreToolCalls = true
  let pendingMessages: Message[] = (await config.getSteeringMessages?.()) || []

  while ((hasMoreToolCalls || pendingMessages.length > 0) && turn < MAX_TURNS) {
    turn++

    // ★ 新：插话先入队，再问模型
    for (const msg of pendingMessages) {
      messages.push(msg)
    }
    pendingMessages = []

    const context = await buildContext(messages, tools)
    const reply = await callModel(context)
    messages.push(reply)

    if (!reply.toolCalls?.length) {
      hasMoreToolCalls = false
    } else {
      for (const call of reply.toolCalls) {
        const result = await execute(call)
        messages.push({ role: "tool", toolCallId: call.id, content: result })
      }
      hasMoreToolCalls = true
    }

    // ★ 新：工具跑完再捞一次（用户可能刚打完字）
    pendingMessages = (await config.getSteeringMessages?.()) || []
  }
  return messages.at(-1)
}
```

内层条件变成：`hasMoreToolCalls || pendingMessages.length > 0`。没有工具、也没有新插话，内层才停。此时还**没有**外层 follow-up。

---

## V3 — `beforeToolCall` 门禁

### 实际需求

Data Agent 跑 SQL：不能模型说啥就 exec 啥。要有白名单、危险语句拦截、副作用门禁。工具不是直接 `execute`——这是证明 **Harness ≠ 裸循环** 的地方，虾皮面 3 爱抠。

Pi 在真正执行前调 `beforeToolCall`；返回 `{ block: true, reason }` 则拦截。**被拦也要写回一条 tool 角色的结果**（错误/拒绝），不能跳过写回——否则模型侧挂着未完成的 toolCall，下一圈对不齐。

### 只改执行那一小段

```ts
for (const call of reply.toolCalls) {
  // ★ 新：执行前门禁
  const gate = await config.beforeToolCall?.({ toolCall: call, messages })
  if (gate?.block) {
    messages.push({
      role: "tool",
      toolCallId: call.id,
      content: gate.reason || "Tool execution was blocked",
    })
    continue
  }

  const result = await execute(call)
  messages.push({ role: "tool", toolCallId: call.id, content: result })
}
```

其余循环、steering 检查点都不动。面试口述一句：**宿主不选工具，但可以在执行前 block，并保证 tool 写回完整。**

---

## V4 — 停机控制面（abort / `finishTurn` 精简版）

### 实际需求

只靠 `MAX_TURNS` 太粗。生产上还要：

- 模型报错 / 用户取消（`aborted`）→ 立刻收束，别假装还能再转；
- 业务侧明确说「这一轮结束了」（`finishTurn` → `end`），或「再给我一圈上下文-only」（`continue`）。

Pi 里：`stopReason === "error" | "aborted"` 会 `finishTurn` 后直接结束；正常走完一圈也会问 `finishTurn`，`action === "end"` 则退出。

### 在 V2/V3 骨架上只加停机

```ts
    const reply = await callModel(context)
    messages.push(reply)

    // ★ 新：错误 / 取消 → 本轮收束
    if (reply.stopReason === "error" || reply.stopReason === "aborted") {
      await config.finishTurn?.({ message: reply, messages })
      break
    }

    if (!reply.toolCalls?.length) {
      hasMoreToolCalls = false
    } else {
      // ... beforeToolCall + execute + tool 写回（同 V3）
      hasMoreToolCalls = true
    }

    // ★ 新：业务侧可强制 end / continue
    const decision = await config.finishTurn?.({ message: reply, messages })
    if (decision?.action === "end") break

    pendingMessages = (await config.getSteeringMessages?.()) || []
```

和「只有 `MAX_TURNS`」对比：步数预算仍可保留，但真正的控制面是 **abort + `finishTurn`**。Pi 没有另搞一套业务五态状态机；停机钩子就够答「你怎么优雅停」。

---

## V5 — 外层 Follow-up

### 实际需求

内层停了（没工具、没 steering），用户又发了一句新任务：「刚才那个查完了，再帮我看下退货率。」这不是改当前轮，而是**本轮已停后再开下一意图**——follow-up。

Pi 的形状：外层 `while (true)` 包住内层；内层自然停后调 `getFollowUpMessages()`，有货就把它们设成 `pendingMessages`，再进内层；没有就 `break`。

### 骨架（只看嵌套，细节沿用 V2–V4）

```ts
async function agentLoop(task: string, tools: Tool[], config: AgentLoopConfig) {
  let messages = [{ role: "user", content: task }]
  let turn = 0
  const MAX_TURNS = 20
  let pendingMessages: Message[] = (await config.getSteeringMessages?.()) || []

  // ★ 新：外层 — 本可停时，若有 follow-up 再吃一轮
  while (true) {
    let hasMoreToolCalls = true

    // 内层 — toolCalls + steering（V2–V4）
    while (
      (hasMoreToolCalls || pendingMessages.length > 0) &&
      turn < MAX_TURNS
    ) {
      turn++
      for (const msg of pendingMessages) messages.push(msg)
      pendingMessages = []

      const context = await buildContext(messages, tools)
      const reply = await callModel(context)
      messages.push(reply)

      if (reply.stopReason === "error" || reply.stopReason === "aborted") {
        await config.finishTurn?.({ message: reply, messages })
        return messages.at(-1)
      }

      if (!reply.toolCalls?.length) {
        hasMoreToolCalls = false
      } else {
        for (const call of reply.toolCalls) {
          const gate = await config.beforeToolCall?.({ toolCall: call, messages })
          if (gate?.block) {
            messages.push({
              role: "tool",
              toolCallId: call.id,
              content: gate.reason || "Tool execution was blocked",
            })
            continue
          }
          const result = await execute(call)
          messages.push({ role: "tool", toolCallId: call.id, content: result })
        }
        hasMoreToolCalls = true
      }

      const decision = await config.finishTurn?.({ message: reply, messages })
      if (decision?.action === "end") return messages.at(-1)

      pendingMessages = (await config.getSteeringMessages?.()) || []
    }

    // ★ 新：内层停了 → 看有没有 follow-up
    const followUp = (await config.getFollowUpMessages?.()) || []
    if (followUp.length > 0) {
      pendingMessages = followUp
      continue
    }
    break
  }

  return messages.at(-1)
}
```

记住一句话：**steering 改当前轮；follow-up 等当前轮停干净再开。** 答不出这层，容易被判成「只会 `while true` 调 API」。

---

## 收口：对照 Pi `runLoop`

| 我们演进到的能力 | Pi 1.1.0 里的名字 |
|---|---|
| V0 五步：`callModel` + tool 写回 | `streamAssistantResponse` + tool result 入 history |
| V1 标志位内层 | 内层 `while (hasMoreToolCalls \|\| pendingMessages.length > 0)` |
| V2 steering 双检查点 | `getSteeringMessages`（圈末 poll；圈初推进 history） |
| V3 执行前门禁 | `beforeToolCall`（`block` 时仍写回错误 tool 结果） |
| V4 停机 | `stopReason` error/aborted；`finishTurn` → `end` / `continue` |
| V5 外层排队 | `getFollowUpMessages`；外层 `while (true)` |
| （加分，本文未展开）执行策略 | `executeToolCalls` 默认可并行；`stopReason === "length"` 整批失败防截断参数 |
| （以后细抠）拼上下文 | `prepareNextTurn` → `prepareRequest`；另有 transform / convertToLlm 一类钩子 |

你代码里的 `buildContext`，对应的就是「拼上下文是 Harness 职责」——面试先认这句，细节留到下一篇。

### 口述收口（可直接用）

> 虾皮要的 Agent Loop，业界口径是「LLM + tools in a loop」。我用五步展开讲上下文与停机，再用 Pi `runLoop` 的双层调度（steering / follow-up）和工具前钩子（`beforeToolCall`）证明做过宿主，不是背五个框。

---

## 文末 Checklist

| 版本 | 相对上一版，多了什么 |
|---|---|
| **V0** | 最简五步：`callModel` → assistant → execute → `role: tool`；无工具则 `return`；`MAX_TURNS` |
| **V1** | `hasMoreToolCalls` 标志位代替 `return`，行为不变，为插话留出口 |
| **V2** | `getSteeringMessages`；问模型前推进 pending；工具后再捞；内层条件含 pending |
| **V3** | `beforeToolCall` 可 block；拒绝也写回 tool 角色 |
| **V4** | error/aborted 收束；`finishTurn` 的 end/continue |
| **V5** | 外层 `while (true)` + `getFollowUpMessages`；双层调度成型 |

基于 **pi 1.1.0** 整理。源码入口：`packages/agent/src/agent-loop.ts` 的 `runLoop`（约 L163–321）。本地对照路径示例：`/workspace/pi/packages/agent/src/agent-loop.ts`。

下一步若继续啃：把 `buildContext` 拆成 Pi 的 context 钩子链，或补上 `executeToolCalls` 的并行与 `length` 整批失败——都是「像真引擎」的加分项，可以另开一篇一步一改。
