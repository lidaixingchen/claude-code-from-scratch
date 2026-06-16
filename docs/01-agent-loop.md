# Lesson 01：最小 Agent Loop

## 🎯 本节目标

实现 coding agent 的心脏：一个 while 循环，让 LLM 能调用工具、拿到结果、继续思考，直到任务完成。

---

## 🏆 最终效果

完成本节后，运行以下命令：

```bash
cd python
python -m mini_claude "列出当前目录下所有 .py 文件"
```

你会看到终端输出类似：

```
🔧 list_files {"pattern": "*.py"}
  mini_claude/__init__.py
  mini_claude/__main__.py
  mini_claude/agent.py
  mini_claude/tools.py
  ...

当前目录下有以下 .py 文件：...
```

Agent 自动调用了 `list_files` 工具，拿到结果后组织语言回复你——**整个过程由 LLM 决策驱动，而不是你写死的 if/else**。

---

## 🛠️ 本节任务

1. 创建 `Agent` 类骨架：消息数组 + API 客户端初始化
2. 实现 `_chat()` 核心循环：while → 调用 LLM → 检查 tool_use → 执行 → 继续
3. 注册一个最简工具 `list_files`，让 LLM 有能力操作外部世界
4. 验收：输入自然语言，Agent 自动调用工具并返回结果

---

## 📦 涉及文件

创建：
- `python/mini_claude/agent.py`
- `python/mini_claude/tools.py`

---

## 🚀 开始实现

### Step 1：消息数组——Agent 的记忆

#### 为什么做

LLM API 是无状态的——每次调用不会记住上一次说了什么。要让 Agent 能"连续对话"，必须把**完整历史消息**每次都发给 API。

消息数组就是 Agent 的"工作记忆"。

#### 做什么

创建 `agent.py`，定义 `Agent` 类的骨架：

```python
# agent.py

from __future__ import annotations

import anthropic


class Agent:
    def __init__(self, model: str = "claude-sonnet-4-6"):
        self.model = model
        self._client = anthropic.AsyncAnthropic()
        self._messages: list[dict] = []
```

`_messages` 是核心数据结构。它的增长方式：

```
用户说"列出 .py 文件"后：
  messages = [
    { role: "user",      content: "列出 .py 文件" }
  ]

LLM 返回 tool_use 后：
  messages = [
    { role: "user",      content: "列出 .py 文件" },
    { role: "assistant", content: [text + tool_use(list_files)] },
    { role: "user",      content: [tool_result("文件列表...")] },
  ]

LLM 认为任务完成后：
  messages = [
    ...前 3 条,
    { role: "assistant", content: [text("当前目录有以下文件...")] }
  ]
```

**每轮循环增长两条**：一条 assistant（LLM 的回复），一条 user（工具结果）。模型每次都能看到完整历史，这是它能"记住"之前做过什么的原因。

#### 注意什么

- 工具结果用 `role: "user"` 推入——这是 Anthropic API 的协议要求，结果必须通过 `tool_use_id` 关联回对应的调用
- 消息数组只增不减，复杂任务跑几十轮后会撑爆上下文窗口——这个问题留到第 7 章解决

---

### Step 2：核心循环——while + tool_use 检查

#### 为什么做

Agent 和普通聊天机器人的区别：**Agent 会主动做事**。

普通聊天：用户问 → LLM 答 → 结束。
Agent：用户问 → LLM 决定调用工具 → 执行工具 → 结果喂回 LLM → LLM 继续思考 → 可能再调工具 → ……直到 LLM 认为任务完成。

这个循环就是 Agent Loop。

#### 做什么

在 `Agent` 类中添加 `_chat()` 方法：

```python
# agent.py（续）

    async def _chat(self, user_message: str) -> None:
        # 1. 把用户消息推入历史
        self._messages.append({"role": "user", "content": user_message})

        while True:
            # 2. 调用 LLM
            response = await self._client.messages.create(
                model=self.model,
                max_tokens=4096,
                system="You are a helpful coding assistant with access to tools.",
                tools=get_tool_definitions(),  # 从 tools.py 导入
                messages=self._messages,
            )

            # 3. 把 LLM 的回复推入历史
            self._messages.append({
                "role": "assistant",
                "content": [self._block_to_dict(b) for b in response.content],
            })

            # 4. 检查是否有 tool_use
            tool_uses = [b for b in response.content if b.type == "tool_use"]
            if not tool_uses:
                break  # 没有工具调用 → 任务完成，退出循环

            # 5. 执行工具，把结果推入历史
            tool_results = []
            for tu in tool_uses:
                result = await execute_tool(tu.name, dict(tu.input))
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": tu.id,
                    "content": result,
                })
            self._messages.append({"role": "user", "content": tool_results})

    @staticmethod
    def _block_to_dict(block) -> dict:
        if block.type == "text":
            return {"type": "text", "text": block.text}
        if block.type == "tool_use":
            return {
                "type": "tool_use",
                "id": block.id,
                "name": block.name,
                "input": dict(block.input) if hasattr(block.input, "items") else block.input,
            }
        return {"type": block.type}
```

循环逻辑只有 **5 步**，核心判断只有一个：**`tool_uses` 是否为空**。

- 有 tool_use → 执行工具 → 结果推入消息 → 继续循环
- 没有 tool_use → LLM 认为任务完成 → break

**是 LLM，而不是代码逻辑，决定任务何时完成。**

#### 注意什么

- `_block_to_dict()` 把 Anthropic SDK 的对象转成普通 dict——因为消息数组里只能存 dict，不能存 SDK 对象
- `tool_use_id` 必须原样传回，它是 LLM 关联"调用"和"结果"的唯一标识
- 所有工具调用是**串行**的（一个 for 循环逐个执行）。Claude Code 用 `StreamingToolExecutor` 在流式响应期间并行执行，但那是第 5 章的内容

---

### Step 3：第一个工具——list_files

#### 为什么做

没有工具，LLM 只是一个聊天机器人。工具是 Agent 和真实世界交互的桥梁。

LLM 本身不能读文件、不能跑命令——但我们可以**在 System Prompt 里告诉它有哪些工具**，它返回结构化的工具调用请求，我们执行后把结果喂回去。

#### 做什么

创建 `tools.py`，定义工具和执行逻辑：

```python
# tools.py

from __future__ import annotations

import os
from pathlib import Path


# 工具定义：告诉 LLM 有哪些工具可用
tool_definitions: list[dict] = [
    {
        "name": "list_files",
        "description": "List files matching a glob pattern. Returns matching file paths.",
        "input_schema": {
            "type": "object",
            "properties": {
                "pattern": {
                    "type": "string",
                    "description": 'Glob pattern to match files (e.g., "**/*.py", "src/**/*")',
                },
                "path": {
                    "type": "string",
                    "description": "Base directory to search from. Defaults to current directory.",
                },
            },
            "required": ["pattern"],
        },
    },
]


def get_tool_definitions() -> list[dict]:
    return tool_definitions


# 工具执行：根据名称分发到具体实现
async def execute_tool(name: str, inp: dict) -> str:
    if name == "list_files":
        return _list_files(inp)
    return f"Unknown tool: {name}"


def _list_files(inp: dict) -> str:
    base = Path(inp.get("path") or ".")
    pattern = inp["pattern"]
    files = []
    for p in base.glob(pattern):
        if p.is_file():
            rel = str(p.relative_to(base) if base != Path(".") else p)
            if "node_modules" in rel or ".git" in rel.split(os.sep):
                continue
            files.append(rel)
            if len(files) >= 200:
                break
    if not files:
        return "No files found matching the pattern."
    return "\n".join(files[:200])
```

工具定义的结构是 **Anthropic Tool Use 协议**要求的格式：

- `name`：工具名，LLM 返回的 `tool_use` 块里会引用这个名字
- `description`：告诉 LLM 这个工具做什么、什么时候该用它
- `input_schema`：JSON Schema，定义工具接受哪些参数

#### 注意什么

- `description` 的质量直接影响 LLM 是否正确使用工具——写得太模糊它会乱调，写得太窄它会漏调
- `glob()` 会自动跳过 `node_modules` 和 `.git`，避免返回垃圾结果
- 结果限制 200 个文件，防止超长输出撑爆上下文

---

### Step 4：组装入口

#### 为什么做

Agent 和工具就绪后，需要一个入口把它们串起来。

#### 做什么

在 `__main__.py` 中添加最简入口：

```python
# __main__.py

import asyncio
from .agent import Agent


async def main():
    agent = Agent()
    await agent._chat("列出当前目录下所有 .py 文件")


if __name__ == "__main__":
    asyncio.run(main())
```

#### 注意什么

- 这里用 `_chat()` 而不是 `chat()`——暂时不处理会话管理、中断、成本统计等，后续章节逐步添加
- 需要设置环境变量 `ANTHROPIC_API_KEY` 才能运行

---

## ⚖️ 设计权衡

### 消息数组 vs 状态机

方案 A：**消息数组**（我们用的）
- 每轮把完整历史发给 LLM
- 优点：LLM 能看到完整上下文，实现简单
- 缺点：消息越来越长，最终撑爆上下文窗口

方案 B：**状态机 + 摘要**
- 每轮只发最近 N 条消息 + 之前的摘要
- 优点：上下文占用可控
- 缺点：摘要会丢失细节，实现复杂

结论：先用消息数组，撑爆了再压缩（第 7 章）。

### 串行工具执行 vs 并行

方案 A：**串行**（我们用的）
- 一个 for 循环逐个执行工具
- 优点：简单、不会出并发问题
- 缺点：多个独立工具时浪费时间

方案 B：**并行执行**
- 用 `asyncio.gather()` 并发执行所有工具
- 优点：多个只读工具可以同时跑
- 缺点：有副作用的工具（如写文件）并行执行可能冲突

结论：先串行，第 5 章加流式输出时再引入并行。

---

## ⚠️ 常见陷阱

### 1. 忘记把 tool_result 推入消息数组

```python
# ❌ 错误：执行了工具但没把结果推入消息
for tu in tool_uses:
    result = await execute_tool(tu.name, dict(tu.input))
    # result 丢了！LLM 看不到工具结果
```

结果：LLM 不知道工具返回了什么，会反复调用同一个工具，**死循环**。

修正：执行完工具后，必须把 `tool_result` 推入 `self._messages`。

### 2. tool_use_id 没有关联上

```python
# ❌ 错误：tool_result 里没有 tool_use_id
tool_results.append({"type": "tool_result", "content": result})
```

结果：API 报错，因为 LLM 不知道这个结果对应哪次调用。

修正：每个 `tool_result` 必须包含 `tool_use_id`，值来自对应 `tool_use` 块的 `id`。

### 3. 忘记把 assistant 回复推入历史

```python
# ❌ 错误：只推了 tool_result，没推 assistant 的回复
if not tool_uses:
    break
# 漏了 self._messages.append({"role": "assistant", ...})
```

结果：下一轮 LLM 看不到自己上一轮说了什么，上下文断裂。

修正：在检查 tool_uses **之前**，先把 assistant 的完整回复推入消息数组。

---

## ✅ 验收点

### 输入

```bash
cd python
export ANTHROPIC_API_KEY=sk-ant-xxx  # 替换成你的 key
python -c "
import asyncio
from mini_claude.agent import Agent

async def main():
    agent = Agent()
    await agent._chat('列出当前目录下所有 .py 文件')

asyncio.run(main())
"
```

### 预期结果

终端输出应包含：
1. LLM 返回的 `tool_use` 块（显示 `list_files` 被调用）
2. 工具执行结果（文件列表）
3. LLM 根据文件列表组织的自然语言回复

### 失败时如何排查

| 症状 | 可能原因 |
|------|---------|
| `AuthenticationError` | API Key 未设置或已过期 |
| `tool_use` 块没有出现 | LLM 认为不需要工具（prompt 太模糊） |
| 工具返回空结果 | glob pattern 不匹配，检查当前目录 |
| 死循环：反复调用同一工具 | tool_result 没推入消息数组 |

---

## 🧠 思考题

1. **如果 LLM 一次返回多个 `tool_use` 块，当前实现会怎样处理？** 这种情况下 LLM 认为什么？（提示：想想"同时需要读两个文件"的场景）

2. **为什么工具结果要用 `role: "user"` 推入，而不是 `role: "tool"`？** 这是 Anthropic API 的设计选择——如果不这样做会有什么问题？（提示：API 要求 user/assistant 严格交替）

3. **当前实现在什么情况下会"死循环"？** 思考：如果 LLM 每次都返回 tool_use 但工具每次都返回同样的结果，循环会怎样退出？（提示：不会退出——这就是为什么后续需要预算控制）

---

## 📦 本节收获

1. **Agent Loop 的本质**：`while True` + `tool_use` 检查。有工具调用就执行并继续，没有就退出——是 LLM 决定任务何时完成。

2. **消息数组是 Agent 的"记忆"**：每轮增长两条（assistant + tool_result），模型每次都能看到完整历史。

3. **工具是 LLM 和真实世界的桥梁**：在 System Prompt 里描述工具，LLM 返回结构化调用请求，代码执行后把结果喂回去。

4. **一个最简 Agent 只需要 ~60 行代码**：消息数组 + while 循环 + 工具执行 + 结果回填——复杂性来自后续的工程优化，不是核心逻辑。

---

> **下一章**：当前只有一个 `list_files` 工具，Agent 能力很有限。我们来构建完整的工具系统——读文件、写文件、编辑文件、搜索、执行 Shell 命令——让 Agent 真正具备"编程"能力。
