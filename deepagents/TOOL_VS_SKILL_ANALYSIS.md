# Tool vs Skill: 数据结构与设计哲学对比分析

> "Bad programmers worry about code. Good programmers worry about data structures and their relationships."  
> —— Linus Torvalds

## TL;DR (核心判断)

**Tool** 和 **Skill** 解决**不同层次**的问题:
- **Tool** = 原子能力 (Capability) - "我能做什么"
- **Skill** = 工作流知识 (Procedure) - "如何做事情"

两者**互补而非替代**,绝不可互换。

---

## 一、数据结构对比

### 1.1 Tool 的数据结构 (LangChain 标准)

```python
# 基于 LangChain BaseTool/StructuredTool
class Tool:
    name: str                    # 工具名称 (如 "read_file")
    description: str             # 功能说明 (触发判断依据)
    func: Callable               # 实际执行函数
    args_schema: BaseModel       # 参数定义 (Pydantic Schema)
    
# 实例: 文件读取工具
@tool
def read_file(
    file_path: str,
    offset: int = 0,
    limit: int = 500
) -> str:
    """读取文件内容。
    
    Args:
        file_path: 文件路径
        offset: 起始行号
        limit: 读取行数
    
    Returns:
        文件内容字符串
    """
    return backend.read(file_path, offset, limit)
```

**关键特征:**
- **确定性**: 相同输入 → 相同输出
- **单一职责**: 一个 Tool 只做一件事
- **可组合性**: Tool 之间无依赖,可自由组合
- **类型安全**: Pydantic Schema 强制参数验证

---

### 1.2 Skill 的数据结构 (Anthropic 标准)

```python
# 元数据结构
class SkillMetadata:
    name: str                    # 技能名称 (如 "pdf-editor")
    description: str             # 使用场景/触发条件
    path: str                    # SKILL.md 文件路径
    source: Literal["user", "project"]  # 来源标识

# 文件结构
skill-name/
├── SKILL.md                     # 必需: YAML frontmatter + Markdown 指令
│   ├── --- (YAML frontmatter)
│   │   ├── name: skill-name
│   │   └── description: "何时使用此技能..."
│   └── (Markdown body)
│       └── 工作流步骤、最佳实践、示例
├── scripts/                     # 可选: 可执行脚本
│   └── rotate_pdf.py
├── references/                  # 可选: 参考文档
│   └── schema.md
└── assets/                      # 可选: 输出资源
    └── template.html
```

**SKILL.md 示例:**
```markdown
---
name: pdf-editor
description: PDF 文档处理技能,支持旋转、合并、拆分。使用场景: (1) 旋转 PDF, (2) 合并多个 PDF, (3) 提取特定页面。触发词: "旋转 PDF"、"合并文档"、"拆分 PDF"。
---

# PDF 编辑技能

## 工作流程

### 1. 旋转 PDF
1. 接收用户提供的 PDF 文件路径
2. 执行 `scripts/rotate_pdf.py --angle 90 <path>`
3. 使用 `write_file` 工具保存结果
4. 返回新文件路径

### 2. 合并 PDF
1. 接收多个 PDF 文件路径
2. 执行 `scripts/merge_pdfs.py <path1> <path2> ...`
3. 保存合并后的文件
```

**关键特征:**
- **三层渐进式加载**: 元数据 → SKILL.md 正文 → 资源文件
- **自然语言描述**: Markdown 格式,LLM 友好
- **工作流编排**: 指导如何组合多个 Tool
- **上下文感知**: 根据任务需求按需加载资源

---

## 二、相同点分析

| 维度 | Tool | Skill |
|------|------|-------|
| **触发机制** | LLM 通过 `description` 判断是否调用 | LLM 通过 `description` 判断是否读取 |
| **注入方式** | Middleware 注入到 system prompt | Middleware 注入到 system prompt |
| **元数据暴露** | 启动时全量加载 (name + description) | 启动时全量加载 (name + description) |
| **使用流程** | 1. 查看工具列表<br>2. 触发 function call | 1. 查看技能列表<br>2. 用 `read_file` 读取 SKILL.md |
| **Progressive Disclosure** | ✅ 仅暴露元数据,按需调用 | ✅ 三层加载 (元数据→正文→资源) |

---

## 三、核心差异

### 3.1 本质定义

```text
Tool  → "我能做什么" (Capability)
        - 提供原子操作能力
        - 例: read_file, write_file, grep, http_request

Skill → "如何做事情" (Procedure)  
        - 提供工作流指导
        - 例: pdf-editor, web-research, code-review
```

---

### 3.2 数据流向

```text
Tool 的执行流:
LLM → Function Call → Tool 函数 → Backend → Return Value → LLM

Skill 的执行流:
LLM → read_file(SKILL.md) → LLM Context → 指导后续 Tool 调用序列
```

**实例对比:**

```python
# Tool 示例 - 原子操作
def grep(pattern: str, path: str) -> list[Match]:
    """在文件中搜索正则表达式模式。
    
    这是一个确定性操作,不包含业务逻辑。
    """
    return backend.grep_raw(pattern, path)

# Skill 示例 - 工作流编排 (Markdown)
"""
# 代码审查技能

## 审查流程
1. 使用 `grep` 工具搜索代码中的 TODO 注释
2. 使用 `read_file` 工具读取相关文件
3. 分析代码质量问题:
   - 是否有硬编码的配置?
   - 是否有未处理的异常?
   - 是否符合项目编码规范?
4. 生成审查报告

## 参考资料
详细编码规范见 `references/coding-standards.md`
"""
```

---

### 3.3 复杂度管理

#### Tool 的复杂度控制

**原则: 单一职责,零特殊情况**

```python
# ❌ 坏品味: 特殊情况堆积
def smart_file_reader(path: str, mode: str = "auto") -> str:
    """智能读取文件 (包含太多逻辑)"""
    if path.endswith(".pdf"):
        return read_pdf(path)
    elif path.endswith(".docx"):
        return read_docx(path)
    elif path.endswith(".xlsx"):
        return read_excel(path)
    # ... 50 种文件类型判断
    else:
        return read_text(path)

# ✅ 好品味: 消除特殊情况
def read_file(path: str, offset: int = 0, limit: int = 500) -> str:
    """读取文本文件内容。
    
    只做一件事,其他格式由专门的 Tool 处理。
    """
    return backend.read(path, offset, limit)

def read_pdf(path: str) -> str:
    """读取 PDF 文件内容。"""
    return pdf_backend.extract_text(path)
```

#### Skill 的复杂度控制

**原则: 三层渐进式加载,按需披露**

```markdown
❌ 坏品味: 所有内容塞进 SKILL.md (5000+ 行)
---
name: bigquery-skill
description: BigQuery 数据查询技能
---

# BigQuery 技能

## 表结构
- users 表: id, name, email, created_at...
- orders 表: id, user_id, amount, status...
- logs 表: timestamp, level, message...
(详细列出 100 个表的所有字段...)

## 查询示例
(500 个 SQL 查询示例...)

## 最佳实践
(50 条最佳实践...)

---

✅ 好品味: 分层组织,消除冗余
skill-name/
├── SKILL.md (< 500 行)
│   ---
│   name: bigquery-skill
│   description: BigQuery 数据查询技能,支持用户、订单、日志等数据分析
│   ---
│   
│   # BigQuery 技能
│   
│   ## 快速开始
│   1. 阅读 `references/schema.md` 了解表结构
│   2. 根据用户问题选择相关表
│   3. 编写 SQL 查询
│   4. 执行查询并返回结果
│   
│   ## 参考资料
│   - **表结构**: `references/schema.md`
│   - **查询示例**: `references/examples.md`
│   - **最佳实践**: `references/best-practices.md`
│
├── references/
│   ├── schema.md          # 仅在需要时加载
│   ├── examples.md        # 仅在需要时加载
│   └── best-practices.md  # 仅在需要时加载
```

---

### 3.4 向后兼容性 (Linus 铁律: "Never break userspace")

#### Tool 的兼容性要求

```python
# ❌ 破坏性变更: 移除参数
def read_file(path: str, limit: int):  
    # 移除了 offset 参数,破坏现有调用

# ❌ 破坏性变更: 修改返回类型
def read_file(path: str) -> dict:  
    # 从 str 改为 dict,破坏现有代码

# ✅ 兼容性变更: 添加可选参数
def read_file(
    path: str,
    offset: int = 0,
    limit: int = 500,
    encoding: str = "utf-8"  # 新增可选参数
) -> str:
    """向后兼容的扩展"""
    ...
```

#### Skill 的兼容性特性

```markdown
✅ Skill 修改不破坏系统 (仅影响工作流质量)

修改前:
---
name: pdf-editor
description: PDF 编辑技能
---
# 使用 PyPDF2 库旋转 PDF
...

修改后:
---
name: pdf-editor
description: PDF 编辑技能,支持旋转、合并、拆分
---
# 使用 pypdf 库 (PyPDF2 的继任者) 旋转 PDF
...

影响: 工作流改进,但不破坏 Tool 调用或系统稳定性
```

---

## 四、实用性验证 (解决真实问题)

### 4.1 Tool 解决的真实问题

**问题**: LLM 无法直接操作文件系统、发起 HTTP 请求、执行 Shell 命令

**解决方案**: 提供确定性的原子操作

```python
# 真实场景: 搜索代码库中的 TODO 注释
@tool
def grep(pattern: str, path: str, file_pattern: str = "*") -> list[dict]:
    """在指定路径搜索正则表达式模式。
    
    这是一个真实需求,LLM 无法直接访问文件系统。
    """
    return backend.grep_search(pattern, path, file_pattern)

# 真实场景: 获取网页内容
@tool
def fetch_url(url: str) -> dict:
    """获取网页内容并转换为 Markdown。
    
    解决 LLM 无法直接访问互联网的问题。
    """
    response = requests.get(url)
    markdown = markdownify(response.text)
    return {"content": markdown, "status": response.status_code}
```

---

### 4.2 Skill 解决的真实问题

**问题**: LLM 不记得公司特定的工作流程、Schema、最佳实践

**解决方案**: 提供可复用的程序性知识

```markdown
真实场景 1: 公司 BigQuery Schema
---
name: company-analytics
description: 公司数据分析技能,包含用户、订单、日志等表的查询方法
---

# 公司数据分析技能

## 表结构
详见 `references/schema.md` (包含 50+ 表的完整定义)

## 常见查询
- **日活用户**: 见 `references/examples.md#daily-active-users`
- **收入统计**: 见 `references/examples.md#revenue-analysis`

---

真实场景 2: PDF 表单填写
---
name: pdf-form-filler
description: 自动填写 PDF 表单,支持政府文件、合同等标准表单
---

# PDF 表单填写技能

## 工作流程
1. 使用 `scripts/extract_fields.py` 提取表单字段
2. 根据用户提供的数据映射字段
3. 使用 `scripts/fill_form.py` 填写表单
4. 保存填写后的 PDF

## 支持的表单类型
- 政府税务表单 (见 `references/tax-forms.md`)
- 标准合同模板 (见 `assets/contract-template.pdf`)
```

---

## 五、组合实战: Tool + Skill 协同

### 场景: "帮我旋转这个 PDF 并上传到 S3"

```markdown
# Skill 编排 (pdf-s3-workflow/SKILL.md)
---
name: pdf-s3-workflow
description: PDF 处理并上传到 S3 的完整工作流
---

# PDF 到 S3 工作流

## 步骤
1. **旋转 PDF**
   - 调用 `rotate_pdf` Tool
   - 参数: path (PDF 路径), angle (旋转角度)

2. **上传到 S3**
   - 调用 `upload_to_s3` Tool
   - 参数: path (文件路径), bucket (S3 桶名)

3. **返回结果**
   - 返回 S3 URL 给用户
```

```python
# Tool 定义 (tools.py)
@tool
def rotate_pdf(path: str, angle: int) -> str:
    """旋转 PDF 文件。
    
    不关心为什么旋转,只负责执行操作。
    """
    output_path = f"{path}.rotated.pdf"
    subprocess.run(["pdftk", path, "rotate", str(angle), "output", output_path])
    return output_path

@tool
def upload_to_s3(path: str, bucket: str, key: str = None) -> str:
    """上传文件到 S3。
    
    不关心文件来源,只负责上传。
    """
    if key is None:
        key = os.path.basename(path)
    s3_client.upload_file(path, bucket, key)
    return f"s3://{bucket}/{key}"
```

**关键洞察:**
- **Skill** 提供 "为什么" (业务逻辑: 旋转后上传)
- **Tool** 提供 "怎么做" (技术能力: 旋转、上传)
- **分离关注点**: Skill 变更不影响 Tool,Tool 可被其他 Skill 复用

---

## 六、反模式 (避免的错误设计)

### 6.1 把工作流逻辑塞进 Tool

```python
# ❌ 反模式: Tool 包含业务逻辑
@tool
def smart_pdf_processor(
    path: str,
    operation: Literal["rotate", "merge", "split"],
    **kwargs
) -> str:
    """智能 PDF 处理器 (违反单一职责)"""
    if operation == "rotate":
        # 50 行旋转逻辑
        ...
    elif operation == "merge":
        # 50 行合并逻辑
        ...
    elif operation == "split":
        # 50 行拆分逻辑
        ...
    # 太复杂,难以测试和维护

# ✅ 正确做法: 拆分为独立 Tool
@tool
def rotate_pdf(path: str, angle: int) -> str:
    """只做旋转,3 行实现"""
    ...

@tool
def merge_pdfs(paths: list[str]) -> str:
    """只做合并,5 行实现"""
    ...
```

---

### 6.2 在 Skill 里重复实现 Tool 能力

```markdown
❌ 反模式: Skill 包含可执行代码
---
name: bad-skill
---

# 坏的技能设计

## 旋转 PDF
使用以下 Python 代码:
```python
from PyPDF2 import PdfReader, PdfWriter
reader = PdfReader(input_path)
writer = PdfWriter()
for page in reader.pages:
    writer.add_page(page.rotate(90))
with open(output_path, "wb") as f:
    writer.write(f)
```
(每次都要 LLM 重新生成这段代码,浪费 Token)

---

✅ 正确做法: Skill 调用 Tool 或脚本
---
name: good-skill
---

# 好的技能设计

## 旋转 PDF
1. 调用 `rotate_pdf` Tool (如果存在)
2. 或执行 `scripts/rotate_pdf.py` (预先测试好的脚本)
3. 保存结果
```

---

## 七、与 MCP (Model Context Protocol) 的关系

### 7.1 MCP 的定位

**MCP** = Tool 的标准化传输协议

```text
Tool (LangChain)  →  提供能力定义和执行逻辑
MCP (Anthropic)   →  提供跨平台的 Tool 调用协议

关系: MCP 是 Tool 的传输层,不改变 Tool 的本质
```

### 7.2 Skill 与 MCP 的互补

```text
MCP Server  →  暴露 Tool (如 filesystem, database, web_search)
Skill       →  指导如何组合这些 Tool 完成任务

示例:
- MCP Server 提供 `read_file`, `write_file`, `grep` 等 Tool
- Skill 提供 "如何使用这些 Tool 进行代码审查" 的工作流
```

---

## 八、Linus 式总结

### 【品味评分】🟢 好品味

Tool 和 Skill 的设计体现了 "好品味" 的核心原则:
1. **消除特殊情况**: Tool 通过单一职责,Skill 通过分层加载
2. **数据结构优先**: 两者的数据结构决定了它们的互补关系
3. **简洁执念**: Tool 函数短小精悍,Skill 文档分层组织

---

### 【数据结构】

```text
Tool  = 函数指针 + 类型签名 (C 风格的简洁)
        struct Tool {
            char *name;
            char *description;
            void (*func)(void *args);
            Schema *args_schema;
        };

Skill = 渐进式文档加载 (三层缓存策略)
        struct Skill {
            Metadata metadata;      // L1: 始终在内存
            char *skill_md_path;    // L2: 按需加载
            char **resource_paths;  // L3: 按需加载
        };
```

---

### 【复杂度】

```text
Tool:  1 层 (函数调用)
       - 输入验证 → 执行 → 返回结果
       - 无条件分支,无特殊情况

Skill: 3 层 (渐进式加载)
       - L1: 元数据 (启动时加载,~100 words)
       - L2: SKILL.md 正文 (触发时加载,< 5k words)
       - L3: 资源文件 (按需加载,无限制)
       - 通过分层消除 Token 浪费
```

---

### 【破坏性分析】

```text
Tool:  修改签名 = 破坏性变更
       - 必须保持向后兼容
       - 新增参数必须是可选的
       - 返回类型不可改变

Skill: 修改内容 = 零破坏性
       - 仅影响工作流质量
       - 不影响系统稳定性
       - 可自由迭代优化
```

---

### 【实用主义验证】

```text
Tool  解决的真实问题:
✅ LLM 无法直接操作文件系统
✅ LLM 无法发起 HTTP 请求
✅ LLM 无法执行 Shell 命令

Skill 解决的真实问题:
✅ LLM 不记得公司 SQL Schema
✅ LLM 不知道标准工作流程
✅ LLM 每次都要重新生成相同代码

两者都在解决真实存在的问题,而非臆想的威胁。
```

---

### 【终极原则】

> "Bad programmers worry about code. Good programmers worry about data structures and their relationships."

**Tool 和 Skill 的数据结构决定了它们是互补而非替代的关系:**

- **Tool** 的数据结构 (函数 + Schema) 决定了它适合提供**原子能力**
- **Skill** 的数据结构 (文档 + 资源) 决定了它适合提供**工作流知识**
- 两者组合形成完整的 Agent 能力体系

**不要试图用 Tool 做 Skill 的事,也不要用 Skill 做 Tool 的事。**

---

## 九、快速参考

### 何时使用 Tool?

- ✅ 需要确定性操作 (文件读写、HTTP 请求、数据库查询)
- ✅ 需要类型安全和参数验证
- ✅ 操作可以独立测试和复用
- ❌ 不要在 Tool 里写业务逻辑

### 何时使用 Skill?

- ✅ 需要指导多步骤工作流
- ✅ 需要提供领域知识或最佳实践
- ✅ 需要组合多个 Tool 完成任务
- ❌ 不要在 Skill 里重复实现 Tool 能力

### Tool + Skill 协同模式

```text
用户请求 → Skill 匹配 → 读取 SKILL.md → 指导 Tool 调用序列 → 完成任务
          ↓
      触发条件判断
          ↓
      工作流编排
          ↓
      Tool₁ → Tool₂ → Tool₃
```

---

## 参考资料

- **Tool 实现**: `libs/deepagents/deepagents/middleware/filesystem.py`
- **Skill 实现**: `libs/deepagents-cli/deepagents_cli/skills/middleware.py`
- **Skill 规范**: `libs/deepagents-cli/examples/skills/skill-creator/SKILL.md`
- **Anthropic Agent Skills**: https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
- **LangChain Tools**: https://python.langchain.com/docs/modules/tools/

---

**文档版本**: v1.0  
**最后更新**: 2026-01-14  
**作者**: DeepAgents 项目分析
