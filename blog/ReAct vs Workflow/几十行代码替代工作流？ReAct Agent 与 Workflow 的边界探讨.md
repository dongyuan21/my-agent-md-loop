
## 一个震撼的对比

销售团队收到新线索时，常规流程是访问对方官网、提取业务信息、录入 CRM、通知团队。某篇教程展示了如何用 n8n 搭建自动化工作流来完成这件事：

- **阶段 1**：表单触发 → URL 清理 → 抓取首页 HTML → 解析 H1/H2 标签和导航链接
    
- **阶段 2**：AI 判断信息是否完整 → Switch 节点分流（完整/不完整）
    
- **阶段 3**（信息不完整时）：AI 识别 About 页面路径 → 构造 URL → 二次抓取 → 深度总结
    
- **阶段 4**：聚合结果 → 写入飞书多维表格 → 发送团队通知
    

这套流程需要配置十几个节点：HTTP 请求、HTML 解析器、AI 调用、Switch、数据映射、通知节点。配置文件冗长，调试时需要逐个检查节点的输入输出。

而用 deepagents 构建的 ReAct Agent，核心逻辑不到 100 行：

```python
# 定义三个工具@tooldef scrape_page(url: str) -> str:    """抓取网页内容并转为 Markdown"""    # 实现略@tooldef find_about_link(page_content: str) -> str:    """从页面内容中查找 About 链接"""    # 实现略@tooldef log_insight(business_summary: str, target_customers: str, core_products: str):    """输出销售洞察"""    # 实现略# 创建 Agentagent = create_react_agent(    llm=ChatOpenAI(model="deepseek-chat"),    tools=[scrape_page, find_about_link, log_insight],    prompt=SYSTEM_PROMPT)# 执行agent.invoke({"messages": [("user", f"请调研: {website}")]})
```

Agent 会自主决定：先爬首页、判断信息是否完整、必要时查找并爬 About 页、最后生成结构化输出。无需配置节点图，无需硬编码分支逻辑。

这引出一个问题：**ReAct Agent 能替代传统 Workflow 吗？**

## deepagents：为复杂任务设计的 Agent 框架

在讨论"替代"之前，需要理解 deepagents 的定位。它不是孤立的新工具，而是构建在成熟生态之上的增强框架。

### 技术栈的层次关系

**LangChain** 是底层工具箱，提供模型接口（ChatModel）、工具定义（@tool）、提示词模板等基础组件。你可以用它调用任何 LLM，也可以定义任何工具函数，但如何让 Agent 循环调用工具、管理状态、处理分支，需要自己写代码。

**LangGraph** 是图编排层，用状态机和有向图描述 Agent 的执行流程。你定义节点（如"模型推理节点"、"工具调用节点"），定义边（如"工具调用完成后回到推理节点"），LangGraph 负责状态管理和流程控制。它支持流式输出、人工介入、跨对话记忆。

**deepagents** 是 Agent 框架层，预设了一套标准工具集和 ReAct 循环。它基于 LangGraph 构建，但封装了常见模式，让你不用从零搭建状态机。调用 `create_deep_agent()` 就能获得一个具备规划、文件操作、子任务委托能力的 Agent。

前面的 demo 实际上用的是 LangGraph 原生的 `create_react_agent()`，并非 deepagents 的完整能力。为了讨论"Agent 替代 Workflow"的边界，我们需要先了解 deepagents 的完整工具箱。

### 核心能力 1：TodoListMiddleware

长时程任务需要规划。deepagents 提供 `write_todos` 和 `read_todos` 工具，Agent 可以在任务开始时列出计划：

```text
Todo 1: 抓取目标网站首页Todo 2: 分析业务领域和目标客户Todo 3: 查找竞品公司Todo 4: 生成对比分析报告
```

每完成一步，Agent 会标记状态为 completed，并根据进展调整后续计划。这类似项目管理中的任务看板，但由 LLM 自主维护。

**适用场景**：研究类任务、内容创作、数据分析等需要多步骤推进的场景。

**Workflow 的差异**：传统 Workflow 的"步骤"在设计时固定，deepagents 的 Todo 由 Agent 在运行时动态生成和调整。

### 核心能力 2：FilesystemMiddleware

复杂任务会产生大量中间结果。deepagents 提供完整的文件系统操作：

- `ls`, `read_file`, `write_file`, `edit_file`：基础文件操作
    
- `glob`, `grep`：查找和搜索
    
- `execute`：执行 shell 命令（需要 Sandbox 后端）
    

更关键的是可插拔后端设计：

- **StateBackend**（默认）：文件存储在 Agent 的临时状态中，任务结束后清空
    
- **FilesystemBackend**：真实磁盘操作，适合需要持久化的场景
    
- **StoreBackend**：跨对话持久化存储，结合 LangGraph Store 使用
    
- **CompositeBackend**：混合路由，例如 `/workspace/` 临时存储，`/memories/` 持久化
    

**实际应用**：

- Agent 将爬取的原始 HTML 保存到 `/raw/page1.md`
    
- 提取的业务信息写入 `/insights/company_profile.json`
    
- 生成的分析报告存为 `/output/report.md`
    
- 如果任务中断，下次可以读取中间状态继续
    

**Workflow 的差异**：n8n 的节点间通过变量传递数据，数据量大时会撑爆内存。deepagents 通过文件系统卸载上下文，并支持跨会话访问。

### 核心能力 3：SubAgentMiddleware

复杂任务需要分治。主 Agent 可以通过 `task()` 工具将子任务委托给专门的子 Agent：

```python
agent = create_deep_agent(    subagents=[{        "name": "research-agent",        "description": "深度研究特定主题",        "system_prompt": "你是研究专家，擅长...",        "tools": [internet_search, read_pdf]    }])
```

主 Agent 在推理时可以调用：`task(agent="research-agent", instruction="分析该公司的技术栈")`

子 Agent 有独立的上下文窗口和工具集，执行完成后将结果返回给主 Agent。

**适用场景**：

- 主 Agent 负责规划，子 Agent 负责执行（如调研、数据处理、内容生成）
    
- 需要不同专业能力的场景（如一个子 Agent 擅长代码分析，另一个擅长文本总结）
    

**Workflow 的差异**：n8n 可以调用另一个 workflow，但两者共享全局变量，容易产生副作用。deepagents 的子 Agent 是完全隔离的执行单元。

### 其他增强能力

- **SummarizationMiddleware**：上下文超过 170k token 时自动总结历史，防止溢出
    
- **AnthropicPromptCachingMiddleware**：缓存系统提示词，降低 Anthropic 模型的成本（可节省 90% 的 prompt token 费用）
    
- **HumanInTheLoopMiddleware**：配置敏感工具需要人工审批，Agent 调用前暂停等待确认
    
- **PatchToolCallsMiddleware**：修复人工中断后的悬空工具调用
    

这些能力的组合，让 deepagents 能够处理远超简单 Workflow 的复杂场景。但我的 demo 并未使用这些能力，它只是一个标准的 ReAct Agent，工具调用链很短（2-4 次），没有规划、没有文件操作、没有子任务委托。

为什么几十行代码就够了？因为这个场景本身不复杂。

## 为什么几十行代码就够了

### Prompt 重新编码业务逻辑

n8n 用节点图表达执行流程，deepagents 用自然语言。demo 的核心在于 `SYSTEM_PROMPT`：

```text
你是销售调研专家，任务是提取：- 业务总结（2 句话）- 目标客户（1 句话）- 核心产品（1 句话）严格按以下流程执行：1. 使用 scrape_page 访问首页2. 判断信息是否完整   - 完整：跳到步骤 3   - 缺失：使用 find_about_link 查找 About 页面，访问并抓取3. 使用 log_insight 输出结果约束：- 禁止跳过首页- 最多访问 2 个页面- 必须调用 log_insight 结束任务- 禁止猜测 URL
```

这段 prompt 等价于 n8n 的节点图。区别在于：

- **n8n**：用节点表示动作（HTTP 请求、AI 判断），用连线表示数据流
    
- **deepagents**：用自然语言描述步骤，LLM 理解后自主调用工具
    

### ReAct 循环 vs 节点图

n8n 的执行模型是**有向无环图遍历**：

1. 从触发节点开始
    
2. 按连线顺序执行每个节点
    
3. 遇到 Switch 节点时，根据条件选择分支
    
4. 直到所有路径终止
    

执行路径在设计时确定，运行时只是"走一遍图"。

deepagents 的执行模型是 **ReAct 循环**：

1. **Reasoning**：LLM 根据当前状态（历史消息、上一步工具输出）和 prompt，生成下一步行动的思考
    
2. **Acting**：LLM 决定调用哪个工具，生成工具参数
    
3. **Observing**：工具执行完成，返回结果追加到消息历史
    
4. 回到第 1 步，直到 LLM 判断任务完成（或达到递归深度限制）
    

关键差异：**每一步的决策都由 LLM 实时做出**，而非预先编码。

### 动态决策的实际含义

demo 的 prompt 规定"判断信息是否完整"，但没有定义"完整"的标准。LLM 会根据上下文理解：

- 如果首页写着"Stripe 是全球领先的支付基础设施提供商，服务在线业务、SaaS 公司，核心产品包括支付 API..."，LLM 判断"信息完整"
    
- 如果首页只写"欢迎来到我们的网站"，LLM 判断"信息缺失"，调用 `find_about_link`
    

这种判断无需硬编码规则。n8n 要实现同样效果，需要：

1. AI 节点生成判断结果（"完整" 或 "不完整"）
    
2. Switch 节点读取结果
    
3. 配置两个分支的后续节点
    

而 deepagents 只需 prompt 中一句话："判断信息是否完整"。

### 关键实现思路

**工具定义**：工具是 Python 函数 + docstring。LLM 通过 docstring 理解工具用途：

```python
@tooldef scrape_page(url: str) -> str:    """抓取网页内容并转为 Markdown"""    # 1. 验证 URL 格式    # 2. 检查是否已访问（防重复）    # 3. 调用 Firecrawl API 或 BeautifulSoup    # 4. 转换为 Markdown 并截断长度    return markdown_content
```

工具内部的复杂逻辑（URL 验证、防重复、备用方案）对 LLM 透明，LLM 只需知道"这个工具能抓网页"。

**Prompt 设计**：prompt 是 Agent 的"操作手册"。好的 prompt 需要：

- 明确任务目标（提取哪些信息）
    
- 定义执行步骤（先做什么、后做什么）
    
- 设置约束条件（不能做什么）
    
- 规定输出格式（如何调用 `log_insight`）
    

**动态决策机制**：LLM 在每次循环中接收的输入包括：

- 系统提示词（任务说明、工具列表）
    
- 完整的消息历史（之前的 Thought、Tool Call、Tool Result）
    
- 用户的原始请求
    

基于这些信息，LLM 生成下一步的 Thought（如"首页信息不完整，需要查找 About 页面"）和 Tool Call（如 `find_about_link(page_content="...")`）。



## 什么时候该用 Agent

### 能替代的场景

**1. 逻辑复杂但能自然语言描述**

如果你能用几句话说清楚"要做什么"，但画成节点图需要十几个节点，Agent 是更好的选择。

demo 中的业务逻辑：

- 先爬首页
    
- 判断信息是否完整
    
- 不完整就找 About 页面再爬一次
    
- 输出结构化结果
    

这四句话翻译成 n8n 需要：HTTP 节点、HTML 解析器、AI 判断节点、Switch 节点、第二个 HTTP 节点、数据聚合节点、输出节点。配置每个节点的输入输出映射，调试时需要逐个检查。

而 Agent 只需要把这四句话写进 prompt。

**2. 需要根据内容动态决策**

demo 中"判断信息是否完整"是典型的内容相关决策。什么叫"完整"？

- 如果首页明确写了业务范围、目标客户、核心产品，就完整
    
- 如果只写了公司名称和口号，就不完整
    

这种判断很难用规则穷举。n8n 要实现，要么：

- 写复杂的正则表达式匹配关键词（不可靠）
    
- 调用 AI 生成"完整/不完整"标签，再用 Switch 节点分流（冗余）
    

Agent 直接在 Thought 中推理："首页只有口号，没有具体业务描述，需要查找 About 页面"。

**3. 长尾场景多，穷举成本高**

网站的结构千奇百怪：

- 有的 About 页面叫 `/about-us`
    
- 有的叫 `/company/profile`
    
- 有的在导航栏叫"关于我们"，有的叫"公司简介"
    
- 有的没有独立 About 页面，信息分散在多个页面
    

如果用 n8n，要么硬编码几个常见路径（覆盖率低），要么写复杂的匹配逻辑（维护成本高）。

Agent 通过 `find_about_link` 工具，从页面内容中用正则匹配所有可能的 About 链接，然后根据匹配结果决定访问哪个 URL。LLM 的泛化能力天然适合处理长尾情况。

**4. 快速原型验证**

需求不明确时，修改 prompt 比修改节点图快。

- 需求变化："不仅要业务信息，还要提取联系方式" → 在 prompt 加一句话
    
- 流程调整："如果首页有完整信息，就不要爬 About 页面了" → 修改 prompt 的步骤 2
    
- 工具增加：新写一个 `extract_contact_info` 工具，加到工具列表，prompt 提一句"如果需要联系方式，使用该工具"
    

n8n 做同样的调整需要：拖拽新节点、调整连线、配置输入输出、重新测试整个分支。

### 不该替代的场景

**1. 严格顺序要求**

银行转账流程：验证账户 → 检查余额 → 锁定金额 → 执行扣款 → 记账 → 通知对方

这五个步骤不能乱序、不能跳过、不能重复。任何一步失败，后续步骤都不能执行。

Agent 的动态决策能力在这里是负担：

- LLM 可能理解偏差，先执行扣款再验证账户
    
- 某一步失败后，LLM 可能自作主张"重试"或"跳过"
    
- 无法向监管机构证明"严格按流程执行"
    

n8n 的节点图是确定的执行路径，每个节点的输入输出明确，符合审计要求。

**2. 高频低成本任务**

数据同步：每分钟从 A 系统抓取新数据，清洗后写入 B 系统

- n8n 执行一次的成本：服务器费用（几乎可忽略）+ API 调用（$0.0001 级别）
    
- Agent 执行一次的成本：LLM 推理（每次 2-5 次调用，每次 $0.001-0.01）+ API 调用
    

每天执行 1440 次（每分钟一次）：

- n8n：$0.14/天
    
- Agent：$1.44-14.4/天
    

Agent 的成本高 10-100 倍。

更关键的是成本不可控：

- n8n 的执行路径固定，成本可以精确预算
    
- Agent 的循环次数不确定，某次可能 2 步完成，某次可能 10 步
    

**3. 实时性保证**

n8n 的每个节点执行时间相对稳定：

- HTTP 请求：50-200ms
    
- 数据转换：10ms
    
- AI 调用：1-3s
    

整个流程的总耗时可以估算：2-5s。

Agent 的执行时间不确定：

- 简单情况：2 次 LLM 调用 + 1 次工具调用 = 3-5s
    
- 复杂情况：5 次 LLM 调用 + 3 次工具调用 = 10-20s
    
- 最坏情况：触发 recursion_limit（如 10 次），可能 30s+
    

如果业务要求"必须在 5s 内返回结果"，Agent 无法保证。

**4. 监管审计**

医疗诊断辅助、金融风控、法律文书生成等场景，需要证明"每一步都按规定执行"。

n8n 的审计日志：

- 节点 1 (验证身份) 输入：{user_id: 123}，输出：{verified: true}
    
- 节点 2 (风险评分) 输入：{user_id: 123}，输出：{score: 85}
    
- 节点 3 (决策) 输入：{score: 85}，输出：{approved: true}
    

Agent 的审计日志：

- Thought 1: "需要验证用户身份"
    
- Tool Call: verify_user(user_id=123)
    
- Tool Result: {"verified": true}
    
- Thought 2: "用户已验证，现在评估风险"
    
- ...
    

Thought 链虽然可读，但不如节点图直观。更关键的是，LLM 的推理过程是黑盒，无法证明"一定按照规定步骤执行"。

### 决策矩阵

|   |   |   |
|---|---|---|
|维度|Workflow 更优|Agent 更优|
|**逻辑复杂度**|简单线性流程|复杂分支逻辑|
|**决策类型**|基于规则（if/switch）|基于内容理解|
|**执行频率**|高频（分钟级）|低频（小时/天级）|
|**成本敏感度**|严格预算控制|可接受变动成本|
|**确定性要求**|必须 100% 可预测|允许 90% 正确率|
|**审计需求**|需要结构化日志|可接受文本日志|
|**维护人员**|非技术团队|技术团队|
|**迭代速度**|需求稳定|快速试错|

**判断标准**：

1. 如果你的场景在"Workflow 更优"列有 3 个及以上命中，用 n8n
    
2. 如果在"Agent 更优"列有 3 个及以上命中，用 deepagents
    
3. 如果各占一半，考虑混合方案
    

## 混合方案：Agent 编排 Workflow

最佳实践可能不是二选一，而是分层使用。

### 架构设计

**顶层：Agent 作为智能路由器**

用户输入是非结构化的自然语言：

- "帮我调研这家公司的技术栈"
    
- "分析最近三个月的销售数据"
    
- "生成本周的客户跟进报告"
    

Agent 理解意图后，决定调用哪个 Workflow：

```python
@tooldef execute_research_workflow(company_website: str):    """调用标准的公司调研工作流"""    # 触发 n8n 的 webhook    return requests.post(N8N_WEBHOOK, json={"website": company_website})@tooldef execute_data_analysis_workflow(date_range: str):    """调用数据分析工作流"""    return requests.post(N8N_WEBHOOK, json={"date_range": date_range})agent = create_deep_agent(    tools=[execute_research_workflow, execute_data_analysis_workflow],    system_prompt="理解用户意图，调用对应的工作流")
```

Agent 的职责是"决策"，而非"执行"。它不需要知道 Workflow 内部有多少个节点，只需要知道"什么场景调用哪个 Workflow"。

**底层：Workflow 作为可靠执行单元**

每个 Workflow 是确定的执行序列：

- 研究工作流：抓取首页 → 解析 → 判断 → 爬 About 页 → 生成报告 → 入库
    
- 数据分析工作流：查询数据库 → 数据清洗 → 统计计算 → 生成图表 → 发送邮件
    

Workflow 内部没有动态决策，严格按步骤执行，保证可靠性和可审计性。

### 案例 1：智能客服

**传统方案**：为每种问题类型配置一个 Workflow

- 用户问退货 → 触发退货流程
    
- 用户问订单状态 → 触发订单查询流程
    
- 用户问产品信息 → 触发产品搜索流程
    

问题：新增问题类型需要配置新 Workflow，意图识别规则越来越复杂。

**混合方案**：

- Agent 理解用户问题，判断问题类型
    
- 调用对应的 Workflow 处理具体业务逻辑
    

```python
agent = create_deep_agent(    tools=[        handle_refund_workflow,        query_order_workflow,        search_product_workflow    ],    system_prompt="""    你是客服助手，理解用户问题后：    - 退货相关 → 调用 handle_refund_workflow    - 订单状态 → 调用 query_order_workflow    - 产品咨询 → 调用 search_product_workflow    """)
```

**优势**：

- Agent 的自然语言理解能力处理意图识别（可泛化到新问题类型）
    
- Workflow 的确定性执行保证业务流程不出错（如退货需要验证订单状态、检查退货政策）
    

### 案例 2：数据处理流水线

**传统方案**：ETL 流程固定

- Extract：从数据库抓取
    
- Transform：按规则清洗
    
- Load：写入目标系统
    

问题：数据格式变化时需要修改 Transform 规则，边缘情况处理困难。

**混合方案**：

- Agent 检查数据质量，决定使用哪个清洗策略
    
- 调用对应的 Workflow 执行 ETL
    

```python
@tooldef execute_standard_etl():    """标准 ETL 流程"""    pass@tooldef execute_strict_etl():    """严格模式（丢弃异常数据）"""    pass@tooldef execute_repair_etl():    """修复模式（尝试修复异常数据）"""    passagent = create_deep_agent(    tools=[execute_standard_etl, execute_strict_etl, execute_repair_etl],    system_prompt="""    检查源数据质量，根据异常率选择策略：    - 异常率 < 1%：标准模式    - 1%-5%：修复模式    - > 5%：严格模式    """)
```

**优势**：

- Agent 根据数据实际情况动态选择策略（无需人工判断）
    
- Workflow 保证 ETL 的原子性和一致性（如事务处理）
    

### 架构示意

```text
用户输入   ↓Agent（理解意图、动态决策）   ↓   ├─→ Workflow A（确定性执行）→ 结果 A   ├─→ Workflow B（确定性执行）→ 结果 B   └─→ Workflow C（确定性执行）→ 结果 C   ↓Agent（整合结果、生成回复）   ↓返回给用户
```

**关键点**：

- Agent 不执行具体业务逻辑，只做路由和整合
    
- Workflow 不做决策，只执行预定义流程
    
- 两者通过工具接口解耦（Agent 调用 Workflow 的 webhook/API）
    

## 给选型者的建议

### 三个判断标准

**1. 你的任务是"规划导向"还是"执行导向"？**

- **规划导向**：不确定要做几步、每步做什么，需要根据中间结果调整
    
    - 例：研究一个复杂主题，可能需要先搜索概述，再深入某个子领域，再找相关论文
        
    - 适合 Agent
        
- **执行导向**：清楚要做哪几步，只是步骤较多
    
    - 例：订单履约（验证库存 → 锁定 → 扣款 → 发货 → 更新状态）
        
    - 适合 Workflow
        

**2. 你的决策依据是"规则"还是"理解"？**

- **规则**：if 条件 A then 动作 X，可以穷举
    
    - 例：订单金额 > 1000 → 需要审批
        
    - 适合 Workflow
        
- **理解**：需要理解内容含义才能决策
    
    - 例：判断网页内容是否包含"业务描述"
        
    - 适合 Agent
        

**3. 你能容忍多大的不确定性？**

- **零容忍**：必须保证每次执行路径相同
    
    - 适合 Workflow
        
- **可接受**：90% 正确率就够，偶尔出错可以人工补救
    
    - 适合 Agent
        

### deepagents 的适用范围

**最适合的场景**：

- 研究类任务：需要多次搜索、阅读、分析、总结
    
- 内容创作：需要查资料、列大纲、分段写作、整合
    
- 复杂决策：需要综合多个信息源，权衡多种方案
    

**需要谨慎的场景**：

- 简单线性流程（用 Workflow 更简单）
    
- 高频任务（成本问题）
    
- 需要严格审计（可解释性不足）
    

**完全不适合的场景**：

- 实时交易系统（确定性要求）
    
- 安全关键系统（不能容忍 LLM 幻觉）
    
- 法律/医疗决策（责任归属问题）
    

### 实施路径建议

**阶段 1：从非关键任务开始**

- 选一个容错性高的场景（如内部调研、内容生成）
    
- 用 Agent 跑一段时间，观察成本和可靠性
    
- 积累 prompt 工程经验
    

**阶段 2：混合方案试点**

- 保留现有 Workflow 作为底层执行单元
    
- 增加 Agent 作为顶层路由
    
- 逐步将决策逻辑从 Workflow 迁移到 Agent
    

**阶段 3：根据数据决策**

- 统计 Agent 的成功率、平均成本、响应时间
    
- 对比 Workflow 的相同指标
    
- 基于 ROI 决定扩大或收缩 Agent 的使用范围
    

**避免一步到位**：不要试图用 Agent 替代所有 Workflow，而是找到合适的边界，让两者协同工作。

### 未来展望

**短期（1-2 年）**：

- LLM 成本持续下降，Agent 的成本劣势缩小
    
- 更多 Middleware 和工具生态成熟，降低开发门槛
    
- 混合架构成为主流（Agent + Workflow 协同）
    

**中期（3-5 年）**：

- LLM 的可靠性提升，可以承担更多确定性任务
    
- 出现专门的"Agent 审计"工具和标准
    
- Agent 逐步进入金融、医疗等监管行业
    

**长期（5 年以上）**：

- 是否会出现"Agent 完全替代 Workflow"的场景？
    
- 取决于：LLM 能否达到 99.99% 的可靠性、成本能否降到 Workflow 同级、监管是否认可
    

但无论如何，**理解两者的边界、选择合适的工具、设计合理的架构**，才是当下开发者应该关注的核心问题。

---

**完整代码和更多案例**：[https://github.com/dongyuan21/sales-crawl-agent](https://github.com/dongyuan21/sales-crawl-agent)

**deepagents 官方文档**：[https://docs.langchain.com/oss/python/deepagents/overview](https://docs.langchain.com/oss/python/deepagents/overview)