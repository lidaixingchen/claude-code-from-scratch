# 第 07 课：文本流式输出与 API 重试

## 🎯 本节目标

为 Agent 构建流畅的流式字符输出界面和稳健的 API 网络容错机制。实现 Anthropic 与 OpenAI 两套后端的文本流渲染（逐字显示回复），滤除大模型的 Extended Thinking 冗余 Token 以节省上下文，并为 API 请求注入带随机抖动的“指数退避”重试保护。

---

## 🏆 最终效果

完成本节后，运行 Agent 时你将看到：
- **逐字打字机效果**：大模型的回答不再是沉默等待数十秒后一次性砸向屏幕，而是字符如瀑布般顺畅流出，首字响应时间降至数百毫秒。
- **思维隐藏**：大模型的 Extended Thinking（思考链）Token 仅在流式生成期间显示，完成后自动被过滤，防止撑爆消息历史。
- **网络容错**：当遇到服务临时过载（429 报错）或网络瞬断时，终端会自动打印类似 `↻ Retry 1/3: HTTP 429` 的重试信息，自动指数退避延时后继续请求，确保执行不会意外中断。

---

## 🛠️ 本节任务

1. **实现流式字符渲染方法**：实现 `_emit_text`，处理流式文本输出以及子代理输出缓冲。
2. **实现 Anthropic 流式文本与思考过滤**：在 `_call_anthropic_stream` 中实现流式输出并过滤掉 `thinking` 块。
3. **实现 OpenAI 增量分块参数重建**：在 `_call_openai_stream` 中手动拼装增量分片的 `arguments` 并流式渲染正文。
4. **编写指数退避与抖动重试封装**：实现 `_is_retryable` 与 `_with_retry`，保护两套流式接口不受瞬时网络故障影响。

---

## 📦 涉及文件

修改：
- `agent.py`

---

## 🚀 开始实现

### 步骤 1：实现流式字符渲染接口

#### 为什么做

由于 Python 默认的 `print()` 会自动换行，且标准输出（stdout）默认有缓冲区，在不换行打印时常常不会即时显示。我们需要通过调用底层 `sys.stdout.write` 并强行刷新（`sys.stdout.flush()`）来实现实时逐字打印。此外，子代理（Sub-agent）运行时，其输出需要被静默缓冲，不能直接打在主终端上。

#### 做什么

修改 `agent.py`，实现 `_emit_text` 方法分流控制：

```python
# agent.py 中的修改

import json
import sys
from .ui import print_assistant_text, stop_spinner  # UI 库封装了 sys.stdout.write/flush


class Agent:
    # ... 在 __init__ 中定义 self._output_buffer: list[str] | None = None

    def _emit_text(self, text: str) -> None:
        # 如果是子代理，只记录在缓冲区中，不打印至控制台
        if self._output_buffer is not None:
            self._output_buffer.append(text)
        else:
            print_assistant_text(text)


#### 注意什么

- **状态存储区分**：在本节的教学简化版中我们直接将 `self._output_buffer` 定义在实例上。但在实际完整 codebase 的架构中，为了统一管理运行状态，我们将其保存在状态容器 `self.state.output_buffer` 中。
- **UI 模块结合**：这里调用的 `print_assistant_text()` 是第 5 课所创建的 `ui.py` 中定义的函数，它可以确保流式字符的输出格式整齐。
```

---

### 步骤 2：实现 Anthropic 后端文本流与思考过滤

#### 为什么做

Anthropic API 的流式响应会混杂输出 `text` 块和 `thinking` 块。
1. `thinking` 块包含模型的中间思考步骤，通常极其庞大（数千 Token），直接存入消息历史会导致后续上下文极速膨胀。我们必须在响应完全接收后过滤掉它们。
2. `text` 块是返回给用户的自然语言，需捕获它并调用 `_emit_text` 实时输出。

#### 做什么

在 `agent.py` 的 `_call_anthropic_stream` 中实现流监听及思考链清洗逻辑：

```python
# agent.py（续）

    async def _call_anthropic_stream(self, on_tool_block_complete=None) -> Any:
        async def _do():
            max_output = _get_max_output_tokens(self.config.model)
            create_params: dict[str, Any] = {
                "model": self.config.model,
                "max_tokens": max_output if self.state.thinking_mode != "disabled" else 16384,
                "system": self._system_prompt,
                "tools": get_active_tool_definitions(self.tools),
                "messages": self.history.anthropic_messages,
            }

            # 根据 thinking_mode 决定是否启用 Extended Thinking
            if self.state.thinking_mode in ("adaptive", "enabled"):
                create_params["thinking"] = {"type": "enabled", "budget_tokens": max_output - 1}

            tool_blocks_by_index: dict[int, dict] = {}
            first_text = True

            # 启动 API 监听流
            async with self._anthropic_client.messages.stream(**create_params) as stream:
                async for event in stream:
                    if not hasattr(event, 'type'):
                        continue

                    if event.type == "content_block_start":
                        cb = getattr(event, 'content_block', None)
                        if cb and getattr(cb, 'type', None) == "tool_use":
                            tool_blocks_by_index[event.index] = {
                                "id": cb.id,
                                "name": cb.name,
                                "input_json": "",
                            }
                    elif event.type == "content_block_delta":
                        delta = event.delta
                        # 捕获并清洗思考链（thinking）以及普通文本并实时流式渲染
                        if hasattr(delta, 'text'):
                            if first_text:
                                stop_spinner()
                                self._emit_text("\n")  # 首字输出前先换行
                                first_text = False
                            self._emit_text(delta.text)
                        elif hasattr(delta, 'thinking'):
                            if first_text:
                                stop_spinner()
                                self._emit_text("\n  [thinking] ")
                                first_text = False
                            self._emit_text(delta.thinking)
                        elif hasattr(delta, 'partial_json'):
                            tb = tool_blocks_by_index.get(event.index)
                            if tb:
                                tb["input_json"] += delta.partial_json
                                
                    elif event.type == "content_block_stop":
                        # 当一个完整的工具块结束时，解析其 JSON 参数并回调触发流式工具抢跑
                        tb = tool_blocks_by_index.pop(event.index, None)
                        if tb and on_tool_block_complete:
                            try:
                                parsed = json.loads(tb["input_json"] or "{}")
                            except Exception:
                                parsed = {}
                            on_tool_block_complete({
                                "type": "tool_use", "id": tb["id"],
                                "name": tb["name"], "input": parsed,
                            })

                final_message = await stream.get_final_message()

            # 【核心过滤】滤除思考块（thinking），不将它们保存到对话历史消息中
            final_message.content = [b for b in final_message.content if b.type != "thinking"]
            return final_message

        # 使用步骤 4 实现遇到的重试方法进行包裹
        return await _with_retry(_do)


#### 注意什么

- **思考链与打字机 Spinner**：在 Anthropic 流式读取时，模型可能会先返回 `thinking` 类型的数据块进行思考。为了保证用户体验，我们必须在收到首个有效字符（无论是普通文本还是思考文本）时立即调用 `stop_spinner()` 来停止加载动画。
- **工具定义获取**：在实际 codebase 中，请使用 `get_active_tool_definitions(self.tools)` 动态获取当前激活的工具，以支持后续多 Agent 沙箱的限制工具列表。
```

---

### 步骤 3：实现 OpenAI 增量分块参数重建与流式输出

#### 为什么做

OpenAI 的流式格式和 Anthropic 大相径庭：
1. **工具调用切片到达**：OpenAI 的 `tool_calls` 不是完整的 JSON 块，而是打碎成极小的 `delta` 块分片推送。比如 `arguments` 属性可能会每次推送 `{"fi`、`le_`、`pa` 这样几个字符。我们必须通过 `choices[0].delta.tool_calls` 的 `index` 识别出属于第几个工具，手动累加参数字符串，最后在流结束时进行拼装。
2. **正文流输出**：捕获 `delta.content` 并进行流式打字渲染。

#### 做什么

在 `agent.py` 中实现 `_call_openai_stream` 的增量装配器：

```python
# agent.py（续）


async def _call_openai_stream(self) -> dict:
    async def _do():
        # 启动 OpenAI 兼容端流式生成（include_usage 让最后一个 chunk 携带 token 统计）
        stream = await self._openai_client.chat.completions.create(
            model=self.config.model,
            messages=self.history.openai_messages,
            tools=_to_openai_tools(get_active_tool_definitions(self.tools)),
            stream=True,
            stream_options={"include_usage": True},
        )

        content = ""
        first_text = True
        tool_calls: dict[int, dict] = {}
        finish_reason = ""
        usage = None  # 用于记录最后一个 chunk 返回的 token 用量

        async for chunk in stream:
            # 捕获 usage 信息（通常在最后一个 chunk 中出现）
            if chunk.usage:
                usage = {
                    "prompt_tokens": chunk.usage.prompt_tokens,
                    "completion_tokens": chunk.usage.completion_tokens,
                }

            if not chunk.choices:
                continue
            delta = chunk.choices[0].delta

            # 1. 处理正文输出文本，并进行流式刷新
            if delta and delta.content:
                if first_text:
                    self._emit_text("\n")
                    first_text = False
                self._emit_text(delta.content)
                content += delta.content

            # 2. 收集与累加工具调用参数分片
            if delta and delta.tool_calls:
                for tc in delta.tool_calls:
                    existing = tool_calls.get(tc.index)
                    if existing:
                        # 累加参数字符串
                        if tc.function and tc.function.arguments:
                            existing["arguments"] += tc.function.arguments
                    else:
                        # 初始化首个参数块
                        tool_calls[tc.index] = {
                            "id": tc.id or "",
                            "name": (tc.function.name if tc.function else "") or "",
                            "arguments": (tc.function.arguments if tc.function else "") or "",
                        }

            if chunk.choices[0].finish_reason:
                finish_reason = chunk.choices[0].finish_reason

        # 3. 拼装成符合标准的 OpenAI 格式工具对象结构
        assembled = (
            [
                {
                    "id": tc["id"],
                    "type": "function",
                    "function": {"name": tc["name"], "arguments": tc["arguments"]},
                }
                for _, tc in sorted(tool_calls.items())
            ]
            if tool_calls
            else None
        )

        # 返回统一的数据包供外层主循环更新历史
        return {
            "choices": [
                {
                    "message": {
                        "role": "assistant",
                        "content": content or None,
                        "tool_calls": assembled,
                    },
                    "finish_reason": finish_reason or "stop",
                }
            ],
            "usage": usage or {"prompt_tokens": 0, "completion_tokens": 0},
        }

    return await _with_retry(_do)


#### 注意什么

- **消息历史与 OpenAI 格式规范**：在流式输出结束后，我们需要使用 `MessageHistory` 来统一更新历史记录。
  1. 对于 **Anthropic 后端**：在 `_call_anthropic_stream` 结束后，通过 `self.history.append_assistant_message()` 添加。
  2. 对于 **OpenAI 后端**：在 `_call_openai_stream` 收集完文本和 `tool_calls` 后，通过 `self.history.append_assistant_message()` 添加带有 `tool_calls` 的回复。如果是字典格式，抽象层会直接推入，防止将其包装在多余的 `assistant` 属性中导致 OpenAI API 抛出 400 Bad Request。
- **模块级函数调用**：注意 `_to_openai_tools` 是一个模块级工具函数，调用时不需要加上 `self.` 前缀。
```

---

### 步骤 4：编写指数退避与抖动重试封装

#### 为什么做

网络请求常会遇到服务器偶尔过载（HTTP 429）、服务器维护临时不可达（HTTP 503/529）或连接被外部重置（`ECONNRESET`）。
- 我们应当只重试此类“可恢复的错误”（不应重试参数错误 400 或认证失败 401 ）。
- 指数退避（每次重试等待时长翻倍，如 $1\text{s} \to 2\text{s} \to 4\text{s}$）能让下游服务器在大负荷时有喘息之机。
- 随机抖动（Jitter）则能防止多台机器在同一时间同步重试，避免形成“重试风暴”。

#### 做什么

在 `agent.py` 文件中，编写重试拦截装饰逻辑：

```python
# agent.py（续）

import asyncio
import time
from .ui import print_retry  # 导入重试渲染函数


def _is_retryable(error: Exception) -> bool:
    # 提取错误状态码
    status = getattr(error, "status_code", None) or getattr(error, "status", None)
    if status in (429, 503, 529):
        return True
    # 注意：不做 .lower()，源码直接匹配大写关键字
    msg = str(error)
    if "overloaded" in msg or "ECONNRESET" in msg or "ETIMEDOUT" in msg:
        return True
    return False


async def _with_retry(fn, max_retries: int = 3):
    for attempt in range(max_retries + 1):
        try:
            return await fn()
        except Exception as error:
            # 如果重试次数耗尽，或者错误不可恢复，直接向上抛出
            if attempt >= max_retries or not _is_retryable(error):
                raise
            
            # 指数退避计算：min(30s, 1s * 2^attempt) + 随机抖动时间
            delay = min(1.0 * (2 ** attempt), 30.0) + (hash(str(time.time())) % 1000) / 1000
            
            status = getattr(error, "status_code", None) or getattr(error, "status", None)
            reason = f"HTTP {status}" if status else "network error"
            print_retry(attempt + 1, max_retries, reason)
            
            await asyncio.sleep(delay)


#### 注意什么

- **避免惊群效应（Thundering Herd）**：在重试机制中加入随机抖动（Jitter）至关重要。当大批客户端同时因云端 API 限流（如 429）或网络瞬断而请求失败时，如果它们都采用整秒指数退避（例如 1s, 2s, 4s），它们会在相同的秒数切片处再次并发轰炸网关。加上 0 到 1 秒之间的随机抖动值 `jitter`，可以有效错开各客户端的实际重试时点，平滑流量波峰。
```

---

## ⚖️ 设计权衡

### 思考链历史保存 vs 丢弃（Filtering Thinking Tokens）

- **方案 A**：**从消息历史中过滤**（我们所用）
  - 流式结束时，从 API 返回结果的消息体（`content` 数组）中剔除 `thinking` 类型的块，只保留 `text` 块存入 `self._messages`。
  - **优点**：大幅节约上下文窗口空间，避免多轮对话时被思考文本占满，减缓大模型的生成成本。
  - **缺点**：大模型在下一轮对话中无法看到自己上一轮具体的“心路历程”（只看得到自己的最终结论和工具输出），但实践证明其决策影响极小。
- **方案 B**：**完全完整保存**
  - 不做任何过滤，将 `thinking` 和 `text` 原封不动发回。
  - **优点**：模型记忆完全一致。
  - **缺点**：上下文 Token 消耗会呈数倍爆发，很快就会逼近极限，在真实工程环境中不推荐。

**结论**：过滤丢弃是高频交互 Agent 保持 Token 经济型的最重要前置策略。

---

## ⚠️ 常见陷阱

### 1. 弯单引号引起 OpenAI JSON 反序列化失败

OpenAI 在推送 `tool_calls` 的 `arguments` 切片时，模型有时会误输出非标准的 JSON 字符串。如果在收集完毕时没有容错机制，直接使用 `json.loads()` 会发生解析崩溃。

**修正**：在 `__main__.py` 或 `execute_tool` 中，我们必须提供解析的容错处理，如发生 `JSONDecodeError` 则退回默认空字典，防止抛出未处理异常。

---

### 2. 重试风暴（Retry Storm）中漏掉抖动因子

```python
# ❌ 错误：如果只使用纯粹的指数退避，没有引入随机抖动
delay = 1.0 * (2 ** attempt)
```

**后果**：若并发客户端很多，一旦网络闪断，全部客户端都会在完全相同的物理时间点（比如第 1.0 秒、2.0 秒、4.0 秒）向 API 网关发起海量冲击重试，导致刚刚恢复的网关由于被瞬间压垮而再次挂掉。

---

## ✅ 验收点

### 输入与验证

1. 启动 Agent 进入 REPL 终端：
   ```bash
   python -m mini_claude
   ```
2. 输入一个需要较长文本回复的复杂查询（例如让模型写一段 100 行 of 算法）。
3. **观察流式效果**：仔细核对字词是否是一个一个跳出来，在输出过程中，能否通过 `Ctrl+C` 中断输出流并成功返回 `> ` 提示符。
4. **模拟重试测试**：可以通过暂时掐断网线或提供一个极低重载限流的模拟接口 base_url，验证终端是否能正确捕获网络异常并成功打印出 `↻ Retry 1/3: ...` 的提示。

### 失败时如何排查

1. **终端加载动画（Spinner）无法停止**：检查在解析 `content_block_delta` 事件时，是否遗漏了在首个 `text` 或 `thinking` 块到达时调用 `stop_spinner()`。
2. **OpenAI 模式下提示 `400 Bad Request`**：检查在流式结束后向 `MessageHistory` 回填 assistant 消息时，是否错误地将已经拼接包装好的 `choice.message` 字典又做了一次冗余的 `role` 和 `content` 包装。
3. **未定义的函数报错**：确保 `tools.py` 导出的方法名称在 `agent.py` 顶部的 import 列表中拼写正确（使用 `get_active_tool_definitions` 而不是 `get_tool_definitions`）。

---

## 🧠 思考题

1. **为什么在 `_call_openai_stream` 中，我们需要使用 `sorted(tool_calls.items())` 对累加后的工具列表进行排序？**
   *(提示：大模型流式发回切片时，即使是多个工具的 delta 包，它们也有可能会由于网络传输因素发生乱序。在拼接时按照 index 序号对其重新排序，可以保证构建出的 arguments 结构严格对应。)*
2. **在 `_is_retryable` 判断中，我们为什么要主动忽略 HTTP 401（未授权）和 HTTP 400（请求不合法）错误而不进行重试？**
   *(提示：因为这些错误由于参数或配置写死，是无法通过“等待一段时间重新发送”来自动解决的。盲目重试只会徒增 API 等待时间和算力浪费。)*

---

## 📦 本节收获

1. **SSE 打字机交互**：掌握了利用 Server-Sent Events 事件机制渲染实时字符的终端交互技术。
2. **切片数据流累加**：掌握了还原多路并行乱序切片参数包（OpenAI 格式）的数据组装算法。
3. **退避防风暴设计**：理解了指数退避加随机抖动的算法在提高云端 API 交互可用性上的重大工程作用。

---

> **下一章**：现在 Agent 既能高速操作又能实时流式沟通。但一个能运行任意 Shell 命令的 Agent 是极其危险的，我们需要构筑防卫线——权限与安全系统。
