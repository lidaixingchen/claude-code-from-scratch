# 第 09 课：上下文分级压缩

## 🎯 本节目标

实现 Agent 消息历史自动摘要压缩机制（Auto-compaction）。当对话轮次过多、Token 消耗逼近大模型上下文窗口上限（85% 阈值）时，自动调用大模型将历史会话提炼为一段简明摘要，重构消息历史队列，在保留最新输入轮的同时瞬间释放高达 80% 的上下文空间，赋予 Agent 理论上无限长对话的能力。

---

## 🏆 最终效果

完成本节后，当 Agent 执行复杂多轮任务，或当用户在 REPL 交互终端中手动键入 `/compact` 指令时：
1. Agent 将历史对话提取并发起一次摘要生成请求。
2. 历史中的数十条消息、冗余工具结果，会被替换为一条简洁的用户摘要和模型确认：
   ```
   [Previous conversation summary]
   用户要求修改 main.py 的 bug。我通过 read_file 读取了内容，发现第 12 行有拼写错误，已通过 edit_file 完成了拼写订正，目前正准备运行 pytest 验证。
   ```
3. 腾出大量窗口空间，避免发生上下文爆表（Context Window Exceeded）的灾难性错误。

---

## 🛠️ 本节任务

1. **引入 Token 统计与阈值检测**：在 `Agent` 中加入 Token 跟踪变量，实现 `_check_and_compact()` 触发检查函数。
2. **编写双后端自动摘要器**：实现 `_compact_conversation()` 以及对应的 `_compact_anthropic()` 与 `_compact_openai()` 消息历史重构逻辑。
3. **在主循环中接入自动压缩**：将自动检测压缩织入 Agent 的轮次边界（Turn Boundary）上。
4. **连通手动压缩命令行指令**：在 `__main__.py` 的 REPL 循环中绑定 `/compact`，支持用户强制压缩。

---

## 📦 涉及文件

修改：
- `python/mini_claude/agent.py`
- `python/mini_claude/__main__.py`

---

## 🚀 开始实现

### 步骤 1：引入 Token 统计与阈值检测

#### 为什么做

我们需要在每次大模型 API 响应返回时，记录大模型返回的 `input_tokens`。
1. 以该统计值判定当前会话对窗口的占用比例。
2. 设定 `effective_window`（有效安全边界，如 `200000 - 20000 = 180000` tokens，留出 20k tokens 给当前输入与输出）。
3. 当占用率超过有效边界的 **85%** 时，发出提示并自动开启压缩流程。

#### 做什么

修改 `agent.py`，在初始化和 API 返回后记录 Token，并编写 `_check_and_compact` 方法：

```python
# agent.py 中的修改


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

        # 1. 声明 Token 跟踪和有效窗口边界（默认有效窗口设为 180,000 tokens）
        self.last_input_token_count = 0
        self.effective_window = 180000

        # ... 其余初始化代码保持不变

    async def _check_and_compact(self) -> None:
        # 当最近一次模型返回的输入 Token 超过有效窗口的 85% 时，触发压缩
        if self.last_input_token_count > self.effective_window * 0.85:
            print("  [cyan]ℹ Context window filling up, compacting conversation...[/cyan]")
            await self._compact_conversation()
```

---

### 步骤 2：实现双后端自动摘要器

#### 为什么做

这是本节最核心的技术实现。我们将利用大模型自身的理解和归纳能力来缩减对话。
1. **备份最新消息**：必须先备份当前等待回复的最后一条 `user` 消息。
2. **请求摘要**：将除最后一条外的所有历史消息发给模型，要求模型提炼成一段摘要（保留核心决策、修改过的文件、目前进展）。
3. **消息历史重构**：将历史清空，用 `[Previous conversation summary] \n 摘要` 和 `Understood...` 这一对问答覆盖历史，随后在队列最末尾追加回此前备份的最新 `user` 消息，实现无缝平滑替换。
4. **后端协议差异**：Anthropic 消息历史是纯粹的消息数组；OpenAI 的第一条消息必须是 `system` 消息，在切片和覆盖时需要予以特判和保留。

#### 做什么

在 `agent.py` 中编写 `_compact_conversation` 路由分发器及具体的后端压缩实现：

```python
# agent.py（续）

    async def _compact_conversation(self) -> None:
        if self.use_openai:
            await self._compact_openai()
        else:
            await self._compact_anthropic()

    async def _compact_anthropic(self) -> None:
        # 消息数过少（不足以进行有意义的总结）则直接返回
        if len(self._messages) < 4:
            return

        # 1. 备份最后一条当前正在处理的用户消息
        last_user_msg = self._messages[-1]

        # 2. 向上游 API 请求对除最后一条外的历史进行总结
        response = await self._anthropic_client.messages.create(
            model=self.model,
            max_tokens=1024,
            system="You are a conversation summarizer. Be concise but preserve important decisions, file paths, and context.",
            messages=self._messages[:-1],  # 排除最新的一条
        )
        summary = (
            response.content[0].text
            if response.content and response.content[0].type == "text"
            else "No summary available."
        )

        # 3. 重塑消息历史队列：一问一答，包含先前的摘要
        self._messages = [
            {"role": "user", "content": f"[Previous conversation summary]\n{summary}"},
            {
                "role": "assistant",
                "content": "Understood. I have the context of our previous conversation. How can I continue helping?",
            },
        ]

        # 4. 将最新的一条用户请求重新接驳到历史队列末端，确保当前轮继续正常流式输出
        if last_user_msg.get("role") == "user":
            self._messages.append(last_user_msg)
        
        # 重置计数器
        self.last_input_token_count = 0

    async def _compact_openai(self) -> None:
        # OpenAI 最少包含 system + 2 轮对话 + 最新用户消息 = 5 条消息
        if len(self._messages) < 5:
            return

        # 1. 备份首位 system 消息和末位 user 消息
        system_msg = self._messages[0]
        last_user_msg = self._messages[-1]

        # 2. 向上游 OpenAI API 发送总结请求
        response = await self._openai_client.chat.completions.create(
            model=self.model,
            max_tokens=1024,
            messages=[
                {"role": "system", "content": "You are a conversation summarizer. Be concise but preserve important decisions, file paths, and context."},
                *self._messages[1:-1],  # 排除首位的 system 和末位的最新 user 消息
            ],
        )
        summary = response.choices[0].message.content or "No summary available."

        # 3. 重新构造 OpenAI 历史列表
        self._messages = [
            system_msg,
            {"role": "user", "content": f"[Previous conversation summary]\n{summary}"},
            {
                "role": "assistant",
                "content": "Understood. I have the context of our previous conversation. How can I continue helping?",
            },
        ]

        # 4. 追加最新用户消息
        if last_user_msg.get("role") == "user":
            self._messages.append(last_user_msg)
            
        self.last_input_token_count = 0
```

---

### 步骤 3：在轮次边界（Turn Boundary）上接入自动检测

#### 为什么做

这是本节**最关键的避坑要点**。
- 为什么必须在 **Turn Boundary（轮次边界，即每次用户新输入被 push 进队列之后、while 主循环启动并发送新请求之前）** 触发压缩检测？
- 因为在此处，队列的最末端永远是一条干净的纯文本 `user` 消息（例如 `{"role": "user", "content": "新提问"}`）。
- 如果你在工具循环执行的过程中（`while True` 中段）触发压缩，此时队列末端可能是未配对的 `tool_result`，执行切片（`[:-1]`）会直接截断 `tool_use` 与 `tool_result` 的物理配对关系，直接导致 LLM 报错拒绝服务。

#### 做什么

修改 `agent.py` 的公共 `chat` 对外接口，在进入对话循环前，率先调用压缩审查：

```python
# agent.py 中的修改


    async def chat(self, user_message: str) -> None:
        self._aborted = False
        
        # 【触发位置】：在当前用户输入被处理之后，但在任何工具循环 API 发起之前
        # 此时历史最末尾是纯文本 user 消息，最安全
        self._messages.append({"role": "user", "content": user_message})
        
        try:
            # 审查并决定是否在此处自动压缩历史
            await self._check_and_compact()
            
            # 进入核心 while 循环（由于用户输入已经 append，需调整核心循环中不要再重复 append 用户消息）
            await self._chat_no_append() 
        finally:
            self._auto_save()
```

*(注：此处修改时，需注意确保 `_chat` 内部不再重复执行 `self._messages.append(...)` 操作)*。
另外，在 `_call_anthropic_stream` 与 `_call_openai_stream` 请求成功后，务必更新最近一次的 Token 消耗：
```python
# 例如在 _call_anthropic_stream 内部：
self.last_input_token_count = response.usage.input_tokens
```

---

### 步骤 4：连通 REPL 终端中的手动压缩指令 `/compact`

#### 为什么做

除了自动阈值检测外，我们需要向终端暴露一个直观的手动调试入口 `/compact`。当用户认为对话太长，或者发现系统有些变慢时，可以强制触发会话压缩，立刻清理历史。

#### 做什么

修改 `__main__.py` 中的 REPL 指令解析段，连通 Agent 的手动压缩接口：

```python
# __main__.py 中的修改

        # ... 在 run_repl 函数的命令分发处修改：
        if inp == "/clear":
            agent.clear_history()
            continue
            
        # 连通手动压缩命令
        if inp == "/compact":
            # 构造一条虚拟的用户命令，以便将当前的输入作为一个 turn boundary，触发安全压缩
            agent._messages.append({"role": "user", "content": "Please summarize our conversation."})
            await agent._compact_conversation()
            # 移除我们刚刚压入的虚拟消息，防止干扰
            agent._messages.pop()
            
            print("  [cyan]ℹ Conversation compacted successfully.[/cyan]")
            continue
```

---

## ⚖️ 设计权衡

### 本地编译 Token 估算（Tokenizer） vs 依赖 API 响应元数据

- **方案 A**：**基于 API 响应元数据（Usage）记录**（我们所用）
  - 每次请求成功后，读取大模型官方响应中携带的 `prompt_tokens` 或 `input_tokens` 精确数值存入 `self.last_input_token_count`。
  - **优点**：数据 100% 物理精准，没有任何估算误差，代码实现极其轻量。
  - **缺点**：只有在一次完整对话请求成功后，才能拿到最新数据，无法在发送前对超长的单次输入（如输入一个 10MB 的文本）进行事前阻断。
- **方案 B**：**本地引入 Tiktoken / Anthropic Tokenizer**
  - 在本地加载大模型词表分词器，对消息队列进行实时离线 Token 估算。
  - **优点**：可以在请求发出之前阻断和压缩。
  - **缺点**：需要引入大型外部依赖库，且不同后端（Anthropic vs OpenAI vs DeepSeek）的词表不同，本地估算存在 5%-10% 左右的系统误差，违背了 mini-claude 的“零第三方依赖”极简原则。

**结论**：使用 API 响应提供的真实元数据，是教学版本和简化设计下保持高精准度、零外部依赖的最佳实践。

---

## ⚠️ 常见陷阱

### 1. 错误的压缩时机切断了工具匹配对

```python
# ❌ 错误：在工具执行的 while 循环中途执行了 compact
# 此时 _messages = [..., tool_use, tool_result]
# 切片 messages[:-1] 丢弃了最后一条 tool_result，但保留了前一条 assistant 的 tool_use 块
response = await client.messages.create(...)
```

**后果**：API 服务端会直接返回 `400 BadRequest` 报错，提示 `assistant` 的工具调用缺失对应的结果回执。
**修正**：自动压缩只能位于 **Turn Boundary（轮次边界）**。如果需要在中途强制触发，必须保证当前队列的最末尾是常规文本消息。

---

### 2. 摘要中遗漏了关键的文件修改历史

在总结时，模型可能会为了简短而忽略之前修改过哪些文件，导致压缩后 Agent 忘记之前的进度而造成重复改写。

**修正**：在总结的 `system` 提示词中，要给模型施加严格的提示规则：`system="You are a conversation summarizer. Be concise but preserve important decisions, file paths, and context."`。

---

## ✅ 验收点

### 输入与执行

1. 启动 Agent REPL：
   ```bash
   python -m mini_claude
   ```
2. 进行 2-3 轮常规问答。
3. 输入手动压缩命令 `/compact`。
4. **验证压缩历史**：
   - 终端应该打印 `ℹ Conversation compacted successfully.`。
   - 接着向 Agent 提问：`“我们刚才聊了什么？”`，确认大模型是否能准确复述出压缩前的前情概要。

---

## 🧠 思考题

1. **在 `_compact_openai` 中，我们读取首位 system 消息并最终把它重新塞回到新历史 `self._messages[0]` 的首位，为什么？**
   *(提示：OpenAI 协议中，系统提示词必须一直处于整个消息历史队列的最开头。如果压缩时把 `system` 消息丢弃了，模型会失去对自身身份和工具规范的认知，直接退化为普通的无工具聊天助手。)*
2. **自动摘要压缩虽然释放了 Token 空间，但它需要我们专门调用一次大模型 API。频繁自动压缩会产生什么副作用？**
   *(提示：频繁压缩会增加额外的 API 费用和响应等待时间。因此，我们设定了高达 85% 的“高水位线”触发阈值，尽量减少总结 API 的调用频次。)*

---

## 📦 本节收获

1. **分级压缩机制**：掌握了多级降级（硬限制硬截断、高水位线自动摘要）的上下文容错架构。
2. **轮次边界控制（Turn Boundary）**：理解了在 Agent 状态机中保护 `tool_use`/`tool_result` 协议配对完整性的重要设计原理。
3. **元数据校准**：掌握了利用大模型 API 元数据反馈校正 Token 计算偏离的工程实现手段。

---

> **下一章**：现在 Agent 拥有了长期对话的无限记忆。接下来我们将为其实现长效记忆机制——通过本地持久化实现跨会话的语义召回记忆系统。
