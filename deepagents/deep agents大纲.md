1. system prompt
2. planning tool
3. sub agents
4. file system

## **deepagents 架构的四大支柱 (The Four Pillars of the deepagents Architecture)**  
deepagents 是分析现有高级智能体系统（如 Claude Code 和 Manus）的结果，这些系统（或多或少，但日益如此😁）成功地处理了复杂任务。Harrison Chase 确定了四个共同的特征，这些特征是它们有效性的关键。

**1. 首要指令：系统提示工程 (Prime Directive: System Prompt Engineering)**  
每个“深度智能体”的基础是一个异常详细的系统提示。这不是一个简单的单句指令，而是一个全面的“操作手册”，精确地定义了智能体的身份、操作原则和约束。在 deepagents 中，这个元素被极其严肃地对待，这体现在其内置的提示中，该提示深受重建的“Claude Code”系统提示的启发。分析此提示的结构表明，它远不止是一个指令。它包含详细的响应格式指南，强制执行解决问题的方法论方法（例如，通过“逐步”推理），最重要的是，包含内置工具、子代理和虚拟文件系统的详尽 API 文档。智能体不仅被告知要做什么，还被告知如何使用其可用资源。

该提示是模块化的，由几个部分组成：

- **通用指令 (General instructions):** 定义智能体的角色和首要目标。
- **工具描述 (Tool descriptions):** 为每个可用工具、其参数和预期的调用格式提供详细文档。
- **思考与行动格式 (Thinking and action format):** 强制要求响应结构，例如，要求智能体首先在 `<thinking>` 标签内呈现其计划，然后才在 `<tool_code>` 标签内调用工具。
- **自定义指令 (Custom instructions):** 用户可以注入其特定任务指令的空间。

**2. 作为上下文锚点的规划：无操作工具模式 (Planning as a Context Anchor: The No-Op Tool Pattern)**  
deepagents 中实现的最微妙但最迷人的模式之一，是使用一个规划工具，该工具从技术角度来看是一个**无操作工具 (no-op tool)**。这个名为 `TodoWrite` 的工具，灵感来源于 Claude Code 中的类似机制，实际上在外部世界不执行任何操作——它不发送电子邮件、不修改文件、也不查询 API。它的唯一目的是**迫使 LLM 将其行动计划外化**并记录在上下文窗口 (context window) 中。

标准的 LLM 进行长期的“头脑中”规划的能力有限。ReAct 循环一次只关注一个“推理-行动”步骤，鼓励战术性而非战略性思考。几次迭代后，原始目标可能会模糊，智能体开始“漂移 (drift)”，失去上下文。这种方法优雅地解决了问题。通过调用无操作工具，智能体记录其计划（例如，作为待办事项列表），该计划成为对话历史的一部分，从而成为模型在后续步骤中可以引用的持久性上下文元素（无需从头开始“重新提取”）。它充当了一种所谓的**上下文锚点 (context anchor)**，稳定了智能体的长期策略，并不断提醒其首要目标和后续步骤。

这种模式是利用工具调用机制来操纵智能体内部认知状态的一个例子，而不仅仅是与外部世界交互。它不是依赖模型脆弱的内部记忆来跟踪其计划，而是将计划“外化”，使其成为可观察环境（对话历史）的一部分。通过这种方式，LLM 有效地“欺骗”了其自身的限制，为其策略创建了一个外部记忆，从而支持执行复杂的多步骤任务。

通过理解 LLM 架构的局限性，该项目提供了一种规避这些限制的机制。


**3. 分层代理：上下文隔离与模块化 (Hierarchical Agents: Context Quarantine & Modularization)**  
随着任务复杂性的增加，单个智能体会遇到可扩展性障碍，这有两个关键原因，都与语言模型的有限上下文窗口有关。首先，其上下文窗口充斥着“信息噪音”——不再相关的先前操作结果——使得模型更难定位关键数据并导致目标漂移。其次，大量可用工具使决策过程复杂化，增加了选择次优工具的风险。deepagents 通过其内置的创建任务并将其委托给专门子代理 (sub-agents) 的能力来解决这个问题。

这种架构模式带来了两个基本好处，直接借鉴了经典软件工程中已验证的原则。  
（万一还有人怀疑“AI 时代”是否需要这样的原则。）

- **第一，上下文隔离 (Context Quarantine)。** 当主智能体遇到一个子问题（例如，“进行详细的竞争对手研究”）时，它不会自己解决并在其主上下文窗口中用数十个搜索结果和分析“污染”它，而是可以生成一个专门负责此任务的子代理。该子代理在其独立的上下文中运行，仅专注于分配的目标。完成后，它仅将浓缩的最终结果返回给主代理。这保持了主上下文的清洁，并专注于任务的战略层面。
- **第二，专业化与模块化 (specialization and modularization)。** deepagents 允许定义自定义子代理，每个子代理都有自己独特的系统提示和专门的工具集。通过这种方式，可以创建一个“专家团队”：一个用于研究的子代理（可访问搜索工具），另一个用于编写和重构代码的子代理（可访问文件系统和 linter），再一个用于数据分析的子代理（可访问分析库）。主代理然后充当项目经理，分解问题并将任务委托给合适的专家。

这种模式的实现简单且声明式。开发人员将子代理列表定义为字典列表，其中每个字典通过名称、描述（供主代理理解）、提示和可选的工具列表来描述一个子代理。以下代码片段说明了这个概念：

```python

research_sub_agent = { "name": "research-agent", 
	"description": "用于深入研究问题 (Used to research more in depth questions)", 
	"prompt": "你是一名专家研究员... (You are an expert researcher...)", 
	"tools": ["tavily_search_results_json"] } 

agent = create_deep_agent(
	 tools=[...], instructions="您的主要目标是... (Your main goal is to...)", 
	subagents=[research_sub_agent] )
```
因此，deepagents 中的子代理架构是将古老而优秀的 **SOLID 单一职责原则 (Single Responsibility Principle)** 直接转化到代理系统领域。

**4. 虚拟文件系统 (Virtual File System)**  
四大支柱中的最后一个是允许智能体持久保存和读取不适合放入上下文窗口或必须在不同步骤或不同代理之间共享的信息的机制。deepagents 将其实现为一个**虚拟文件系统 (virtual file system)**。这里的关键架构决策是，这不是与计算机物理文件系统的交互。相反，它是一个虚拟结构，在 LangGraph 状态对象内部实现为一个字典 (dict)。

这种机制有两个关键作用：

- **首先，** 它充当单个智能体的**短期工作内存（“草稿本” / "scratchpad"）**。智能体可以将笔记、中间结果、代码片段或长文本写入“文件”，这样它以后可以返回到这些内容，而无需将它们保存在活动上下文窗口中。
- **其次，** 它充当主代理及其子代理之间**异步协作的媒介**。主代理可以将子代理的输入数据写入一个文件，任务完成后，子代理可以将其结果写入另一个文件。

为了启用与此虚拟空间的交互，deepagents 提供了一组模仿标准系统命令的内置工具：

- `ls`: 列出虚拟文件系统中的“文件”。
- `read_file`: 读取“文件”的内容。
- `write_file`: 创建新“文件”或覆盖现有文件。
- `edit_file`: 修改现有“文件”的内容。

将文件系统作为受管理图状态的一部分来实现，而不是作为与外部、不受控资源的交互，是一个极其简单且巧妙的解决方案，确保了智能体的**无状态性 (statelessness)** 和**可扩展性 (scalability)**。由于智能体的整个状态——包括其“文件系统”——都由 LangGraph 显式管理，因此可以在同一台机器上并行运行同一智能体的多个实例，而没有写入冲突或并发问题的风险。每个智能体调用都是完全密闭的 (hermetic)，这理论上应该简化基于 deepagents 的系统的部署（咳咳，无服务器化）、测试和扩展。

用户可以轻松地使用预定义的文件系统状态初始化智能体，并在工作完成后读取其最终状态：

```python
# 使用虚拟文件系统中的文件初始化智能体 
result = agent.invoke({ "messages":
	 [{"role": "user", "content": "分析提供的数据。 (Analyze the provided data.)"}], 
	 "files": {"input.csv": "col1,col2\nval1,val2"} # 虚拟文件系统初始内容 
}) 
# 从临时文件系统读取文件 
output_files = result["files"] 
print(output_files.get("analysis_summary.txt")) # 打印输出文件内容

```

**底层引擎——LangGraph！ (And under the hood — LangGraph!)**  
但理解 deepagents 的四大支柱——那些从 Claude “借鉴”来的——只是故事的一半。

要充分欣赏这个项目的优雅之处，我们需要降低一个抽象层次，看看它的基础：**LangGraph 库**。正是这个库实现了对前一章描述的模式进行可靠且受控的编排。

**为什么是 LangGraph 而不是 LangChain？ (Why LangGraph and not LangChain?)**  
**从 LangChain 到 LangGraph (From LangChain to LangGraph)**

对于那些还记得 LLM 热潮早期的人来说，LangChain 通过引入 **链 (Chains)** 的概念——语言模型调用和工具调用的序列——彻底改变了我们构建 LLM 应用的方式。链非常适合建模可以表示为**有向无环图 (DAGs)** 的过程，其中信息流基本上是线性的。

然而，智能体工作的性质根本不同：它是**循环的 (cyclical)**。将智能体与单纯的“LLM 应用”区分开来的是，它反复执行一个循环：推理 → 选择行动 → 执行行动 → 观察结果 → 带着新信息返回推理。

早期尝试用 LangChain 的智能体机制（相信我，确实有过这样的尝试）实现这种循环逻辑遇到了麻烦。控制逻辑常常隐藏在复杂的提示中，开发人员对执行流的可见性有限——这使得调试和确保可靠性变得困难。

这就是项目作者创建 **LangGraph** 的原因：一个从头开始设计的新框架，用于建模循环的、有状态的图 (stateful graphs)，完美捕捉智能体的循环。LangGraph 不是将控制逻辑深埋其中，而是强制开发人员将其明确定义为节点 (nodes) 和边 (edges)，提供完全的透明度和控制权。

**StateGraph 中的状态管理 (State Management in StateGraph)**

LangGraph 中的核心概念是 **StateGraph**，一个其节点在共享状态对象上操作的图。在 deepagents 中，这个状态——`AgentState`——远比简单的对话历史丰富。它是一个聚合了智能体运作所需所有关键信息的综合性对象：

- **消息 (Messages):** 所有交互（人类、AI 和工具输出）的列表
- **文件系统 (File system):** 代表智能体虚拟文件系统的字典，如前一节所述
- **任务列表 (Task list):** 智能体的计划，分解为待处理 (pending)、进行中 (in-progress) 和已完成 (completed) 的任务

图中的每个节点代表智能体逻辑中的一个步骤，它接收当前状态，可以读取它，然后返回一个修改后的版本。LangGraph 确保这个状态在节点之间一致地传递。

更重要的是，LangGraph 提供了内置的持久化机制，称为 **检查点器 (checkpointers)**，它可以在每个步骤后自动保存图的状态。这允许智能体在发生错误时从最后一个保存点恢复——这对于长期运行、可靠的系统来说绝对是至关重要的功能。每次发生错误时都从头重启整个智能体流程，不仅代价高昂，而且在时间上也极其低效。

**流逻辑：条件节点和边 (Flow Logic: Conditional Nodes and Edges)**  
在 LangGraph 中，控制流通过由节点和边组成的图明确定义。

- **节点 (Nodes)** 通常是执行单一“工作单元”的简单 Python 函数——例如，调用 LLM 做决策、执行 LLM 选择的工具或聚合结果。
- **边 (Edges)** 连接这些节点，定义可能的执行路径。

LangGraph 的真正力量在于**条件边 (conditional edges)**。在决策节点（例如，一个查询了 LLM 的节点）执行后，你可以定义根据结果将执行路由到不同节点的逻辑——无论 LLM 决定调用工具、直接响应用户还是委托工作给子代理。这种机制使得能够显式实现智能体的循环循环，并使开发人员能够精确控制智能体在各种情况下的行为。

在 deepagents 中，这种流逻辑决定了下一步应该是工具调用、子代理调用还是完成工作并返回答案。

这就是为什么 deepagents + LangGraph **瞄准那些控制、透明度和可靠性至关重要的场景**。虽然像 CrewAI（我们目前使用的）这样的框架通过更高级别的抽象实现了更快的原型设计，而 AutoGen 在为动态多代理交互实验提供无与伦比的灵活性，但 **LangGraph 使工程师能够构建健壮的、生产级的系统，其中每一步都被明确定义并且完全可观察**。

---

**经验教训——我们能从 deepagents 中学到什么？ (Lessons Learned — What can we learn from deepagents?)**  
我在本文开篇就提到，审查别人的代码是最好的学习练习之一。对于 deepagents 架构也是如此，它从 Claude Code 中“提炼”出某些优雅、可复用的解决方案，并提供了一些有趣的技术。

**组合优于发明 (Composition over Invention)**  
从 deepagents 中学到的最重要一课可能是：Harrison Chase 展示了如何将更简单、定义明确且易于理解的组件组合成一个系统，其**涌现行为 (emergent behavior)** 远比其各部分的总和更丰富、更强大，而不是试图构建一个单一的、庞大的、能做所有事情的“超级智能”模型。

最终，deepagents 只是 LangGraph 之上的一个薄层——一个精确的提示、一个规划实用工具、模块化的子代理和一个虚拟文件系统……仅此而已。其中任何一个单独的部分都不是革命性的。但当它们通过 LangGraph 精心组合和编排时，就形成了一个能够进行深度推理的架构。整个项目只有几百行代码（不包括提示）——事实上，比描述它的这篇文章还要少。而正是这个小规模让我想在一小时内阅读和理解它——它都刚好能装进我的“本地缓存” 😁。

同时，该项目包含一些巧妙的技术。例如，`create_deep_agent` 函数就像一个模板：它接受一个配置（工具列表、自定义指令、子代理定义和选择的 LLM 模型），并在此基础上生成并返回一个功能齐全、已编译的智能体图实例。

```python
def create_deep_agent(
    tools: Sequence[Union[BaseTool, Callable, dict[str, Any]]], # 工具序列
    instructions: str, # 指令
    model: Optional[Union[str, LanguageModelLike]] = None, # 模型（可选）
    subagents: list[SubAgent] = None, # 子代理列表（可选）
    state_schema: Optional[StateSchemaType] = None, # 状态模式（可选）
):
```
