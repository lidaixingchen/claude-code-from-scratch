# 第 02 课：双后端架构

## 🎯 本节目标

实现 Agent 的双后端支持。允许 Agent 既可以使用 Anthropic API，也可以使用 OpenAI 兼容的 API（如 DeepSeek、Ollama 本地模型或 GPT-4 等），且两者共享相同的工具定义和执行流程。

---

## 🏆 最终效果

完成本节后，用户可以通过设置环境变量，在不修改代码的情况下，无缝在 Anthropic 和 OpenAI 兼容的后端之间进行切换。

运行以下命令测试 OpenAI 兼容后端（以使用本地 Ollama 运行的 `qwen2.5-coder` 或在线的 DeepSeek API 为例）：

**macOS / Linux**:
```bash
cd python
export OPENAI_BASE_URL="http://localhost:11434/v1"  # 或在线 API 接口
export OPENAI_API_KEY="ollama"                    # 在线 API 请填写实际 Key
export MODEL="qwen2.5-coder"                       # 模型名称
python -m mini_claude "列出当前目录下所有 .py 文件"
```

**Windows (PowerShell)**:
```powershell
cd python
$env:OPENAI_BASE_URL="http://localhost:11434/v1"
$env:OPENAI_API_KEY="ollama"
$env:MODEL="qwen2.5-coder"
python -m mini_claude "列出当前目录下所有 .py 文件"
```

运行后你将看到与第一课相同的工具调用流程，但整个对话和决策都是由 OpenAI 兼容接口的模型驱动的：

```
🔧 list_files {"pattern": "*.py"}
  mini_claude/__init__.py
  mini_claude/__main__.py
  mini_claude/agent.py
  mini_claude/tools.py

当前目录下有以下 .py 文件：...
```

---

## 🛠️ 本节任务

1. **扩展客户端初始化**：增加 `api_base` 与 `api_key` 参数，实例化 `openai.AsyncOpenAI` 客户端。
2. **编写工具定义转换器**：实现 `_to_openai_tools`，将 Anthropic 格式的工具 Schema 自动转换为 OpenAI 格式。
3. **实现 OpenAI 专属的聊天循环 `_chat_openai`**：独立处理 OpenAI 消息列表格式及 `tool_calls` / `tool` 结果的推入。
4. **重构 `_chat` 入口**：根据客户端配置，自动分发到 `_chat_anthropic`（原 `_chat`）或新实现的 `_chat_openai`。
5. **更新 `__main__.py` 入口**：读取环境变量以支持外部参数传递。

---

## 📦 涉及文件

修改：
- `python/mini_claude/agent.py`
- `python/mini_claude/__main__.py`

---

## 🚀 开始实现

### 步骤 1：扩展 `Agent.__init__` 支持双客户端

#### 为什么做

为了支持两套 API，我们需要在初始化时检测用户是否指定了 `api_base`（如 OpenAI 接口地址）。若指定了，则创建 `openai.AsyncOpenAI` 客户端，否则沿用 `anthropic.AsyncAnthropic` 客户端。

#### 做什么

修改 `agent.py`，导入 `openai`，并在 `__init__` 中新增参数以支持实例化不同的客户端：

```python
# agent.py

from __future__ import annotations

import anthropic
import openai  # 新增导入
from .tools import execute_tool, get_tool_definitions


class Agent:
    def __init__(
        self,
        model: str = "claude-sonnet-4-6",
        api_base: str | None = None,
        api_key: str | None = None,
    ):
        self.model = model
        self.use_openai = bool(api_base)
        self._messages: list[dict] = []

        if self.use_openai:
            # 初始化 OpenAI 兼容客户端
            self._openai_client = openai.AsyncOpenAI(base_url=api_base, api_key=api_key)
            self._anthropic_client = None
        else:
            # 初始化 Anthropic 客户端
            self._anthropic_client = anthropic.AsyncAnthropic(api_key=api_key)
            self._openai_client = None
```

#### 注意什么

- `use_openai` 主要通过 `api_base` 是否存在来判断。
- 需要确保在 Python 虚拟环境中已安装 `openai` 库（可使用 `pip install openai` 安装）。

---

### 步骤 2：实现工具格式转换器 `_to_openai_tools`

#### 为什么做

Anthropic 与 OpenAI 对工具的定义 Schema 包装格式不同。Anthropic 直接接受输入 Schema 数组，而 OpenAI 要求外层包裹 `{"type": "function", "function": {...}}`，并将参数指定在 `parameters` 属性中。

#### 做什么

在 `Agent` 类中添加私有方法 `_to_openai_tools` 进行工具定义转换：

```python
# agent.py（续）

    def _to_openai_tools(self, tools: list[dict]) -> list[dict]:
        return [
            {
                "type": "function",
                "function": {
                    "name": t["name"],
                    "description": t["description"],
                    "parameters": t["input_schema"],
                },
            }
            for t in tools
        ]
```

#### 注意什么

- OpenAI 的参数定义在 `parameters` 下，这对应 Anthropic 的 `input_schema`。两者的底层 JSON Schema 字段本身是一致的。

---

### 步骤 3：实现 OpenAI 专属的聊天循环 `_chat_openai`

#### 为什么做

OpenAI API 的消息流与 Anthropic 存在两个关键协议差异：
1. **System Prompt**：OpenAI 不支持顶层的 `system` 参数，必须作为 `{"role": "system", "content": "..."}` 消息插在消息历史 `messages` 列表的开头。
2. **工具调用与结果**：模型工具调用通过 `tool_calls` 返回；执行结果必须通过 `role: "tool"` 消息返回，且必须附带 `tool_call_id`。

#### 做什么

在 `Agent` 类中实现 `_chat_openai` 循环：

```python
# agent.py（续）

    async def _chat_openai(self, user_message: str) -> None:
        # 1. 用户消息推入历史
        self._messages.append({"role": "user", "content": user_message})
        system_prompt = {"role": "system", "content": "You are a helpful coding assistant with access to tools."}

        while True:
            # 2. 调用 OpenAI 兼容 API（将 system 消息拼装在历史最前面）
            response = await self._openai_client.chat.completions.create(
                model=self.model,
                messages=[system_prompt] + self._messages,
                tools=self._to_openai_tools(get_tool_definitions()),
            )
            message = response.choices[0].message

            # 3. 构造并把 assistant 的回复（含文本及可能的 tool_calls）推入历史
            msg_dict = {"role": "assistant", "content": message.content}
            if message.tool_calls:
                msg_dict["tool_calls"] = [
                    {
                        "id": tc.id,
                        "type": "function",
                        "function": {
                            "name": tc.function.name,
                            "arguments": tc.function.arguments,
                        },
                    }
                    for tc in message.tool_calls
                ]
            self._messages.append(msg_dict)

            # 4. 检查是否有工具调用
            if not message.tool_calls:
                break

            # 5. 执行工具并将结果（role: "tool"）推入历史
            import json
            for tc in message.tool_calls:
                try:
                    args = json.loads(tc.function.arguments)
                except Exception:
                    args = {}
                
                result = await execute_tool(tc.function.name, args)
                self._messages.append({
                    "role": "tool",
                    "tool_call_id": tc.id,
                    "name": tc.function.name,
                    "content": result,
                })
```

#### 注意什么

- 保存 `assistant` 回复到 `_messages` 时，务必保留 `tool_calls` 结构。否则，下一次循环将 `tool` 结果发回时，API 会报协议错误（因为 `tool` 结果必须紧跟在包含对应 `tool_calls` 的 `assistant` 回复之后）。
- 每一个 `tool` 消息都必须包含正确的 `tool_call_id`。

---

### 步骤 4：重构 `_chat` 入口进行分发

#### 为什么做

外部调用者不需要关心底层是哪种 API，只需调用统一 of `_chat()` 入口。我们需要根据当前 Agent 的后端配置进行路由转发。

#### 做什么

重构 `agent.py` 中的 `_chat`，将原本的 Anthropic 逻辑提取到 `_chat_anthropic` 中，然后用 `_chat` 进行分发：

```python
# agent.py（续）

    async def _chat(self, user_message: str) -> None:
        if self.use_openai:
            await self._chat_openai(user_message)
        else:
            await self._chat_anthropic(user_message)

    async def _chat_anthropic(self, user_message: str) -> None:
        # 原 Lesson 01 中的实现
        self._messages.append({"role": "user", "content": user_message})

        while True:
            response = await self._anthropic_client.messages.create(
                model=self.model,
                max_tokens=4096,
                system="You are a helpful coding assistant with access to tools.",
                tools=get_tool_definitions(),
                messages=self._messages,
            )

            self._messages.append({
                "role": "assistant",
                "content": [self._block_to_dict(b) for b in response.content],
            })

            tool_uses = [b for b in response.content if b.type == "tool_use"]
            if not tool_uses:
                break

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

#### 注意什么

- 确保原 `_chat` 里的 `self._client` 改为了 `self._anthropic_client`（在 `__init__` 中已对应修改）。

---

### 步骤 5：更新 `__main__.py` 入口

#### 为什么做

我们需要在程序入口读取环境变量，让最终用户可以通过配置 `OPENAI_BASE_URL` 轻松测试 OpenAI 兼容后端，而无需手动修改代码。

#### 做什么

修改 `__main__.py`，从环境变量中读取参数来实例化 `Agent`：

```python
# __main__.py

import os
import sys
import asyncio
from .agent import Agent


async def main():
    query = sys.argv[1] if len(sys.argv) > 1 else "列出当前目录下所有 .py 文件"

    # 读取环境变量
    api_base = os.environ.get("OPENAI_BASE_URL") or os.environ.get("OPENAI_API_BASE")
    api_key = os.environ.get("OPENAI_API_KEY")
    
    # 若设置了 api_base 则默认使用 gpt-4o；否则默认使用 claude-sonnet-4-6
    default_model = "gpt-4o" if api_base else "claude-sonnet-4-6"
    model = os.environ.get("MODEL") or default_model

    agent = Agent(
        model=model,
        api_base=api_base,
        api_key=api_key,
    )
    await agent._chat(query)


if __name__ == "__main__":
    asyncio.run(main())
```

#### 注意什么

- 如果未配置 `OPENAI_BASE_URL`，默认行为依然会走 Anthropic 客户端，保持向后兼容。

---

## ⚖️ 设计权衡

### 方案 A：共用同一个 `_messages` 列表（我们所用的）

- **优点**：代码极其简洁，Agent 内部只维护一个统一的历史记录，在单一会话中无需复杂的跨后端转换。
- **缺点**：如果在单次会话中动态切换后端（这在实际应用中几乎不会发生），不同格式的历史消息混合会导致 API 报错。

### 方案 B：独立维护 `_anthropic_messages` 和 `_openai_messages` 两个列表

- **优点**：后端解耦更彻底，可以避免任何格式混淆的问题，且容易支持运行中动态热切换后端。
- **缺点**：初始化、持久化和状态重构的代码量会增加，对基础教学来说门槛过高。

**结论**：方案 A 足够轻量，非常适合起步教学；在后续开发完整项目时（如支持会话保存），再逐步采用方案 B 进行彻底重构。

---

## ⚠️ 常见陷阱

### 1. 遗漏 `tool_calls` 的回复内容

在推入 `assistant` 消息到 `_messages` 时，不能只保存文本内容（`content`），必须把包含工具调用的 `tool_calls` 结构一起存入：

```python
# ❌ 错误：这会导致下一次发送消息时 API 报错，因为 tool 消息找不到关联的 tool_calls
self._messages.append({"role": "assistant", "content": message.content})
```

**后果**：OpenAI 协议强制要求，所有的 `role: "tool"` 消息之前，必须紧跟一条带有对应 `tool_calls` 列表的 `assistant` 消息。

---

### 2. 系统提示词（System Prompt）的位置错误

OpenAI 协议中不支持顶层的 `system` 参数，必须作为 `{"role": "system", "content": ...}` 消息插在消息历史的最前面。

```python
# ❌ 错误：OpenAI completions 接口不支持顶层 system 参数
response = await self._openai_client.chat.completions.create(
    model=self.model,
    system="...",
    messages=self._messages
)
```

**修正**：在请求时动态将系统提示词拼装在 `messages` 列表 the 第 0 位：`messages=[system_prompt] + self._messages`。

---

## ✅ 验收点

### 输入

在终端中设置 OpenAI 兼容后端（此处以使用本地 Ollama 运行的 `qwen2.5-coder` 模型为例）：

**macOS / Linux**:
```bash
cd python
export OPENAI_BASE_URL="http://localhost:11434/v1"
export OPENAI_API_KEY="ollama"
export MODEL="qwen2.5-coder"
python -m mini_claude "列出当前目录下所有 .py 文件"
```

**Windows (PowerShell)**:
```powershell
cd python
$env:OPENAI_BASE_URL="http://localhost:11434/v1"
$env:OPENAI_API_KEY="ollama"
$env:MODEL="qwen2.5-coder"
python -m mini_claude "列出当前目录下所有 .py 文件"
```

### 预期结果

程序使用指定的本地模型启动，成功调用 `list_files` 工具，读取文件列表并最终以中文回复文件列表，过程与使用 Claude 效果相同。

### 失败时如何排查

| 症状 | 可能原因 | 排除方法 |
|---|---|---|
| `NotFoundError` / `InvalidURL` | `OPENAI_BASE_URL` 格式错误 | 检查接口地址，特别注意末尾是否漏掉 `/v1`。如 Ollama 默认为 `http://localhost:11434/v1`。 |
| `AuthenticationError` | API Key 错误 | 检查环境变量是否成功注入。如果是本地 Ollama，Key 可填写任意字符串。 |
| `BadRequestError: 'messages' must follow...` | 消息推入顺序不对 | 检查 `_chat_openai` 中保存 `assistant` 回复时，是否正确包裹了 `tool_calls`。 |

---

## 🧠 思考题

1. **为什么 OpenAI 协议中每个工具结果（`role: "tool"`）必须单独作为一个消息发送，而 Anthropic 可以在一个 `user` 消息的数组里放多个工具结果？**
   *(提示：OpenAI 要求 `role: "tool"` 消息必须单独存在，并使用 `tool_call_id` 与具体的调用进行平级关联。)*

2. **如果在单轮循环中模型决定同时调用 3 个工具，OpenAI 协议的历史中会增加多少条消息？**
   *(提示：1 条带 3 个 tool_calls 的 assistant 消息 + 3 条单独 the tool 消息 = 共 4 条消息。)*

---

## 📦 本节收获

1. **双后端架构的设计**：理解了如何通过抽象公共方法，同时支持 Anthropic 与 OpenAI 兼容协议。
2. **多协议适配的差异性**：掌握了 OpenAI 中 System Prompt、`tool_calls` 与 `role: "tool"` 的协议特殊要求。
3. **环境适应力**：使 Agent 摆脱单一闭源 API 的限制，具备了接入任意本地模型（Ollama/vLLM）或更具性价比的模型（如 DeepSeek/Qwen）的能力。

---

> **下一章**：有了双后端支持，下一步是让 Agent 具备真正的操作能力——工具系统。
