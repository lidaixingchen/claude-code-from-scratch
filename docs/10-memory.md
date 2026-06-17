# 第 10 课：项目级记忆系统

## 🎯 本节目标

为 Agent 构建跨会话的记忆机制（Persistence Memory）。使用散列哈希对项目目录进行空间隔离，实现轻量级 Frontmatter 结构解析和 `MEMORY.md` 索引文件的自动重建，并通过 System Prompt 注入指引，使 Agent 能够自主读写记忆目录实现记忆的自我进化。

---

## 🏆 最终效果

完成本节后，Agent 会“认得”你是谁，并且能够跨会话记住项目的一些核心信息：
1. **记忆自我沉淀**：当你向 Agent 提到你的偏好（如：“不要在回复末尾做多余的总结”）时，Agent 会自主决定调用 `write_file`，在后台创建一个名为 `feedback_xxx.md` 的记忆文件。
2. **跨会话感知**：你关闭 Agent 后再次启动，向其发起另一个新任务，Agent 的系统提示词中会自动带入更新后的 `MEMORY.md` 索引。模型读取到先前的记忆，便会在新的回复中自动遵守“不要总结”的习惯。

---

## 🛠️ 本节任务

1. **实现项目哈希物理隔离**：在 `memory.py` 中编写 `_project_hash()` 和 `get_memory_dir()`，使每个项目目录都映射到独立的存储路径。
2. **编写 YAML Frontmatter 解析器**：在 `frontmatter.py` 中实现 `FrontmatterResult` 数据类和 `parse_frontmatter`，供 `memory.py` 和第 11 课的 `skills.py` 共享复用。
3. **实现索引文件自动重建**：实现 `_update_memory_index` 逻辑，在新增记忆时重新生成 `MEMORY.md` 索引。
4. **编译记忆提示词段并织入 Prompt**：实现 `build_memory_prompt_section()`，并在 `prompt.py` 中连通替换 `{{memory}}` 占位符。

---

## 📦 涉及文件

修改：
- `python/mini_claude/memory.py`
- `python/mini_claude/prompt.py`

创建：
- `python/mini_claude/frontmatter.py`

---

## 🚀 开始实现

### 步骤 1：基于 CWD 路径进行记忆空间哈希隔离

#### 为什么做

如果用户在本地开发多个不同的项目，我们决不能让 A 项目的上下文记忆混入 B 项目中。我们需要通过计算当前工作目录（CWD）的 SHA-256 散列值，将记忆文件夹物理隔离存放在全局主目录下的 `.mini-claude/projects/{hash}/memory` 中。

#### 做什么

创建（或覆写）`memory.py`，编写路径生成算法：

```python
# memory.py

from __future__ import annotations

import hashlib
import re
from pathlib import Path

from dataclasses import dataclass

VALID_TYPES = {"user", "feedback", "project", "reference"}


@dataclass
class MemoryEntry:
    name: str
    description: str
    type: str
    filename: str
    content: str = ""


def _project_hash() -> str:
    # 提取 cwd 路径并计算 SHA-256 哈希的前 16 位字符
    return hashlib.sha256(str(Path.cwd()).encode()).hexdigest()[:16]


def get_memory_dir() -> Path:
    # 建立项目隔离的记忆归档夹
    d = Path.home() / ".mini-claude" / "projects" / _project_hash() / "memory"
    d.mkdir(parents=True, exist_ok=True)
    return d
```

#### 注意什么

- **物理隔离的重要性**：利用哈希值做项目目录隔离，可以避免 A 项目的开发上下文或规则泄露到 B 项目中，保护了数据安全，也让 Agent 的记忆具有了精准的范围上下文。

---

### 步骤 2：编写 YAML Frontmatter 轻量解析器

#### 为什么做

为了教大模型以结构化的形式保存记忆（如记忆名字、描述及类型），记忆文件会采用类似 Markdown 博客的 Frontmatter 格式头（夹在 `---` 之间的元数据）。
我们需要手写一个极其紧凑的解析器解析这些元数据，而不需要引入像 `PyYAML` 这样臃肿的第三方依赖。

#### 做什么

首先，创建 `frontmatter.py` 作为独立的共享解析模块（后续第 11 课的技能系统也将复用此模块）：

```python
# frontmatter.py

from dataclasses import dataclass, field


@dataclass
class FrontmatterResult:
    meta: dict[str, str] = field(default_factory=dict)
    body: str = ""


def parse_frontmatter(content: str) -> FrontmatterResult:
    lines = content.split("\n")
    # 如果第一行不是 --- 则证明没有 Frontmatter 元数据
    if not lines or lines[0].strip() != "---":
        return FrontmatterResult(body=content)

    end_idx = -1
    for i in range(1, len(lines)):
        if lines[i].strip() == "---":
            end_idx = i
            break

    if end_idx == -1:
        return FrontmatterResult(body=content)

    meta: dict[str, str] = {}
    for i in range(1, end_idx):
        colon = lines[i].find(":")
        if colon == -1:
            continue
        key = lines[i][:colon].strip()
        value = lines[i][colon + 1:].strip()
        if key:
            meta[key] = value

    # 裁剪元数据后，重组正文部分
    body = "\n".join(lines[end_idx + 1:]).strip()
    return FrontmatterResult(meta=meta, body=body)
```

然后，在 `memory.py` 中引入解析器：

```python
# memory.py（续）

from .frontmatter import parse_frontmatter
```

#### 注意什么

- **无依赖解析器**：通过手写 YAML 键值对行切割与正则，可以极大节省包体积，避免了初学者拉取项目时必须安装 PyYAML 第三方包的步骤，降低了启动成本。

注意：`parse_frontmatter` 返回 `FrontmatterResult` 数据类，通过 `.meta` 和 `.body` 属性访问元数据与正文。后续所有调用方（包括第 11 课的技能系统）都统一使用点操作符访问。

---

### 步骤 3：实现索引文件 `MEMORY.md` 自动重建

#### 为什么做

Agent 在提问时，如果一次性将所有的记忆文件全部发给 API 会极大耗费 Token。
- 我们的设计策略是：在记忆目录下维护一个唯一的索引文件 `MEMORY.md`。
- 每次写盘新记忆后，系统会自动扫描目录下所有的 `.md` 文件，读取元数据，并在 `MEMORY.md` 写入结构化目录。
- 这样，系统启动时只需将 `MEMORY.md` 索引注入 System Prompt。Agent 可以通过索引知晓自己存了哪些记忆，当需要具体细节时，通过 `read_file` 主动读取对应文件。

#### 做什么

在 `memory.py` 中编写 `MEMORY.md` 自动重建与检索函数：

```python
# memory.py（续）


def list_memories() -> list[MemoryEntry]:
    d = get_memory_dir()
    entries: list[MemoryEntry] = []
    # 扫描所有 .md 文件并解析
    for f in sorted(d.glob("*.md")):
        if f.name == "MEMORY.md":
            continue
        try:
            result = parse_frontmatter(f.read_text(encoding="utf-8"))
            meta = result.meta
            if meta.get("name") and meta.get("type") in VALID_TYPES:
                entries.append(MemoryEntry(
                    name=meta["name"],
                    description=meta.get("description", ""),
                    type=meta["type"],
                    filename=f.name,
                    content=result.body,
                ))
        except (OSError, ValueError) as e:
            logger.debug(f"Skipping memory file {f}: {e}")
    return entries


def _update_memory_index() -> None:
    memories = list_memories()
    lines = ["# Memory Index", ""]
    for m in memories:
        lines.append(f"- **[{m.name}]({m.filename})** ({m.type}) — {m.description}")
    
    # 覆盖重写唯一的索引文件
    index_path = get_memory_dir() / "MEMORY.md"
    index_path.write_text("\n".join(lines), encoding="utf-8")
```

#### 注意什么

- **数据类型一致性**：在 `list_memories()` 中，我们必须返回定义好的 `MemoryEntry` 对象列表，并在 `_update_memory_index` 中以点号属性访问字段（如 `m.name`），这能保证与第 5 课所实现的 REPL 快捷查看命令 `/memory` （以对象属性方式遍历打印记忆）在数据结构类型上完全一致，避免了在 REPL 下调用命令抛出 `AttributeError` 崩溃。
- **健全的错误处理与日志记录**：在扫描项目目录、读取和解析 YAML Frontmatter 记忆文件时，由于文件可能被外部程序占用、甚至被用户修改损坏，我们需要精准捕获 `OSError` 和 `ValueError`，并通过 `logger.debug` 记录具体的跳过原因，避免异常被无声隐藏，方便后期维护。

---

### 步骤 4：编译记忆提示词段并织入 System Prompt

#### 为什么做

这是促成“记忆自我进化”的魔法：**我们不需要为记忆设计任何特有工具**。
我们只需在 System Prompt 中告诉大模型：“你拥有一个位于 `{dir}` 的记忆系统，可以使用已有的 `write_file`/`edit_file` 工具往里面写文件来记住偏好；并向其展示当前的 `MEMORY.md` 索引”。模型就会自主决定什么时候需要写文件来“记住”某些规则！

#### 做什么

1. 在 `memory.py` 末尾编写 `build_memory_prompt_section`。
2. 修改 `prompt.py`，引入该方法并在 `build_system_prompt` 中替换 `{{memory}}` 占位符。

```python
# memory.py（续）


def build_memory_prompt_section() -> str:
    index_path = get_memory_dir() / "MEMORY.md"
    index = index_path.read_text(encoding="utf-8") if index_path.exists() else ""
    memory_dir = str(get_memory_dir())

    return f"""# Memory System
You have a persistent, file-based memory system at `{memory_dir}`.

## Memory Types
- **user**: User's role, preferences, knowledge level
- **feedback**: Corrections and guidance from the user
- **project**: Ongoing work, goals, deadlines, decisions
- **reference**: Pointers to external resources

## How to Save Memories
Use the write_file tool to create a memory file with YAML frontmatter:
---
name: Memory Title
description: Quick description
type: project
---
Memory details...

Save to: `{memory_dir}/`
Filename format: `{{type}}_{{slugified_name}}.md`

## What NOT to Save
- Code patterns or architecture (read the code instead)
- Git history (use git log)
- Anything already in CLAUDE.md

{"## Current Memory Index" + chr(10) + index if index else "(No memories saved yet.)"}"""
```

在 `prompt.py` 中更新引入和替换：

```python
# prompt.py 中的修改

# 1. 引入记忆编译函数
from .memory import build_memory_prompt_section

# ...

def build_system_prompt() -> str:
    replacements = {
        "{{cwd}}": str(Path.cwd()),
        "{{date}}": date.today().isoformat(),
        "{{platform}}": f"{platform.system()} {platform.machine()}",
        "{{shell}}": os.environ.get("SHELL") or os.environ.get("COMSPEC") or "unknown",
        "{{git_context}}": get_git_context(),
        "{{claude_md}}": load_claude_md(),
        # 2. 替换记忆占位符
        “{{memory}}”: build_memory_prompt_section(), 
    }
    # ... 后续替换逻辑保持不变
```

#### 注意什么

- **大模型自主记忆驱动**：在编译记忆提示词时，我们告诉大模型记忆格式和路径，由大模型在主循环决策中”自主”写文件来记住用户偏好。
- **异步语义召回与简化机制说明**：实际 codebase 实现了高级的基于向量/轻量文本相关的异步语义预取（Prefetching）与热启动过滤机制。在本章的简化教程中，我们着重于搭建文件持久化读写与 `MEMORY.md` 索引编译的基础物理架构。当学习者在最终运行 codebase 时，会接触到更为复杂的异步预取策略，以防止频繁读取引起上下文爆炸。

---

## ⚖️ 设计权衡

### 零工具的“寄生”文件读写 vs 引入专门的 MemoryTool

- **方案 A**：**“寄生”文件读写**（我们所用）
  - 记忆就是普通的文件。大模型使用已有的 `write_file`/`edit_file` 和 `read_file` 读写记忆。
  - **优点**：不需要编写任何新工具，降低了工具池的 Token 开销；且用户可以直接进入 `.mini-claude/projects/` 目录下用本地编辑器（VS Code/Vim）增删改记忆。
  - **缺点**：大模型需要先理解文件写入格式，写入的规范性依赖 System Prompt 的指引强度。
- **方案 B**：**引入定制的 MemoryTool 工具**
  - 在 API 中暴露类似 `save_memory(key, val)` 的强约束工具。
  - **优点**：格式强制统一。
  - **缺点**：用户无法轻易查看或离线修改；增加了额外的工具参数开销。

**结论**：方案 A 将记忆巧妙地“归一化”为文件读写，最完美地践行了“Unix 哲学：一切皆文件”的设计思想。

---

## ⚠️ 常见陷阱

### 1. `MEMORY.md` 索引自身没有被屏蔽

```python
# ❌ 错误：在 list_memories 中没有跳过 MEMORY.md 自身
for f in d.glob("*.md"):
    # 会把 MEMORY.md 的数据也读出来加入它自己，造成无限套娃
```

**修正**：在 glob 遍历时，必须写明 `if f.name == "MEMORY.md": continue` 予以过滤。

---

### 2. 记忆没有按照项目做沙箱隔离

如果简单把所有项目的记忆混着存在全局 `~/.mini-claude/memory/` 目录下，一旦大模型在前端项目 A 下检索到后端项目 B 的命名规范或文件规则，就会产生严重的牛头不对马嘴的逻辑干扰。

**修正**：必须在路径中引入当前工作目录的 hash 值进行强物理隔离。

---

## ✅ 验收点

### 输入与验证

1. 启动 Agent，向其告知一条你的角色和习惯偏好：
   ```bash
   python -m mini_claude "记住：我叫小明，负责前端 React 重构工作"
   ```
2. **观察工具调用**：大模型在返回时，应当自主产生了一次 `write_file` 的调用，在你的后台写入了一个以 `user_` 或 `project_` 开头的记忆 md 文件。
3. **验证持久化**：退出程序。
4. 在另一目录或者直接在当前目录下重新启动 Agent：
   ```bash
   python -m mini_claude "你是谁？当前项目负责人是谁？"
   ```

### 预期结果

大模型能够准确复述出：“我是 Mini Claude Code。当前项目的负责人是小明，他负责前端 React 重构。”

---

## 🧠 思考题

1. **既然我们在 `build_memory_prompt_section` 中只把索引文件 `MEMORY.md` 编译进了 System Prompt，大模型怎么知道每条记忆的具体详细内容呢？**
   *(提示：索引中有类似 `-[不总结](feedback_no_summary.md)` 的格式。模型看到文件名后，当需要查阅“如何不总结”的详情时，会自主发起一次 `read_file` 读取该文件的命令获取内容，实现了“按需拉取”的懒加载模式。)*
2. **为什么我们把 `What NOT to Save`（不要保存什么）加入到了记忆系统的 System Prompt 中？大模型会胡乱保存什么？**
   *(提示：如果不加以限制，大模型倾向于将之前的整个代码结构、Git 记录、历史代码片全部当成记忆存盘，这会导致记忆目录短时间内发生文件大爆炸。)*

---

## 📦 本节收获

1. **一切皆文件的记忆哲学**：理解了在 Agent 系统中通过 Prompt 引导，将记忆维护降维到已有文件工具上的巧妙设计。
2. **多租户空间隔离**：掌握了利用 CWD 哈希实现本地多项目数据沙箱物理隔离的开发模式。
3. **按需加载（Lazy Loading）索引**：掌握了“只曝露索引，按需加载内容”的 Token 减负与缓存友好型架构方案。

---

> **下一章**：现在 Agent 具备了跨会话记忆。下一步我们将实现技能系统——允许 Agent 发现并运行用户或团队定制的 Prompt 工作流。
