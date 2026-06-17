# 第 05 课：CLI 与会话持久化

## 🎯 本节目标

为 Agent 构建用户交互层与状态持久化机制。实现交互式命令行终端（REPL）、Ctrl+C 优雅中断机制，并支持将会话历史以 JSON 格式持久化到本地磁盘，允许用户通过 `--resume` 参数恢复上次对话。

---

## 🏆 最终效果

完成本节后，用户可以直接启动 Agent 进入交互式终端（REPL）：

```bash
python -m mini_claude
```

你将看到精美的欢迎信息与命令行提示符：
```
  Mini Claude Code — A minimal coding agent

  Type your request, or 'exit' to quit.
  Commands: /clear /plan /cost

> hello
```

**功能测试**：
1. **指令测试**：输入 `/clear` 可以清空对话历史。
2. **中断测试**：当 Agent 正在思考或执行工具时，按下 `Ctrl+C` 将会中断本次执行并返回 `> ` 提示符，而不会导致整个程序退出。在空闲状态下，双击 `Ctrl+C` 会退出程序。
3. **会话恢复测试**：输入几句对话后输入 `exit` 退出，再次运行 `python -m mini_claude --resume`，Agent 将完美读取历史会话。

---

## 🛠️ 本节任务

1. **实现会话存储读写库**：在 `session.py` 中编写会话保存、读取与检索逻辑。
2. **封装 Agent 公共接口与自动保存**：在 `agent.py` 中实现 `chat()` 封装，并在对话结束时自动持久化。
3. **创建最简终端 UI 辅助库**：在 `ui.py` 中定义欢迎横幅、提示符、错误输出等基础函数。
4. **编写命令行参数解析**：在 `__main__.py` 中解析运行参数，支持命令行 prompt 直发与 `--resume` 恢复。
5. **构建交互式 REPL 循环与信号拦截**：在 `__main__.py` 中实现 REPL 循环，注入 `Ctrl+C` 信号处理器。

---

## 📦 涉及文件

修改：
- `python/mini_claude/session.py`
- `python/mini_claude/agent.py`
- `python/mini_claude/__main__.py`

创建：
- `python/mini_claude/ui.py`

---

## 🚀 开始实现

### 步骤 1：实现会话存储读写库 `session.py`

#### 为什么做

Agent 在运行过程中，我们需要随时将其当前的对话历史持久化到本地。这样即便程序崩溃或用户主动退出，对话记录也不会丢失。我们会将会话以 JSON 格式保存在用户主目录下的 `.mini-claude/sessions` 目录中。

#### 做什么

创建（或覆写）`session.py`，实现会话的目录创建、保存、读取与查找最新会话 ID 的功能：

```python
# session.py

from __future__ import annotations

import json
from pathlib import Path
from typing import Any

SESSION_DIR = Path.home() / ".mini-claude" / "sessions"


def _ensure_dir() -> None:
    SESSION_DIR.mkdir(parents=True, exist_ok=True)


def save_session(session_id: str, data: dict[str, Any]) -> None:
    _ensure_dir()
    # 转换为漂亮格式的 JSON 保存
    (SESSION_DIR / f"{session_id}.json").write_text(json.dumps(data, indent=2, default=str), encoding="utf-8")


def load_session(session_id: str) -> dict[str, Any] | None:
    path = SESSION_DIR / f"{session_id}.json"
    if not path.exists():
        return None
    try:
        return json.loads(path.read_text(encoding="utf-8"))
    except Exception:
        return None


def list_sessions() -> list[dict[str, Any]]:
    _ensure_dir()
    results = []
    for f in SESSION_DIR.glob("*.json"):
        try:
            data = json.loads(f.read_text(encoding="utf-8"))
            if "metadata" in data:
                results.append(data["metadata"])
        except Exception:
            pass
    return results


def get_latest_session_id() -> str | None:
    sessions = list_sessions()
    if not sessions:
        return None
    # 按照启动时间降序排序，获取最新会话
    sessions.sort(key=lambda s: s.get("startTime", ""), reverse=True)
    return sessions[0].get("id")
```

---

### 步骤 2：封装 Agent 公共接口与自动保存

#### 为什么做

我们在第 1、2 课中实现的 `_chat` 是包含核心 `while` 循环的内部逻辑。我们需要向外部调用者提供一个更加健壮的公共接口 `chat()`，在开始前重置状态，并在每轮对话顺利结束时自动触发会话存储。

#### 做什么

修改 `agent.py`。
1. 在初始化中定义唯一的 `session_id` 和 `session_start_time`。
2. 封装公共 `chat` 方法，并在其中添加 `_auto_save()` 和 `restore_session()` 逻辑。

```python
# agent.py 中的修改

import uuid
import time
from .session import save_session  # 导入保存函数

class Agent:
    def __init__(
        self,
        model: str = "claude-sonnet-4-6",
        api_base: str | None = None,
        api_key: str | None = None,
    ):
        self.config = AgentConfig(model=model, api_base=api_base, api_key=api_key)
        self.state = AgentState()
        self.use_openai = bool(api_base)
        
        # 实例化唯一的会话参数和统一消息历史管理
        system_prompt = "You are a helpful coding assistant with access to tools."
        self.history = MessageHistory(use_openai=self.use_openai, system_prompt=system_prompt)
        self.session_id = uuid.uuid4().hex[:8]
        self.session_start_time = time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())
        self._aborted = False  # 中断标志位

        if self.use_openai:
            self._openai_client = openai.AsyncOpenAI(base_url=api_base, api_key=api_key)
            self._anthropic_client = None
        else:
            self._anthropic_client = anthropic.AsyncAnthropic(api_key=api_key)
            self._openai_client = None

    def abort(self) -> None:
        self._aborted = True

    # 封装公共的对外对话方法
    async def chat(self, user_message: str) -> None:
        self._aborted = False  # 重置中断标志
        try:
            await self._chat(user_message)
        finally:
            # 无论成功还是被 Ctrl+C 中断，均尝试自动保存会话
            self._auto_save()

    def _auto_save(self) -> None:
        try:
            # 保存元数据与消息历史（直接展开 history.to_dict()，保持键名与源码一致）
            save_session(self.session_id, {
                "metadata": {
                    "id": self.session_id,
                    "model": self.config.model,
                    "cwd": str(Path.cwd()),
                    "startTime": self.session_start_time,
                    "messageCount": self.history.message_count(),
                },
                **self.history.to_dict(),
            })
        except Exception:
            pass

    def restore_session(self, data: dict) -> None:
        # data 中包含 anthropicMessages / openaiMessages，直接传给 history.restore()
        self.history.restore(data)
        print(f"  [cyan]ℹ Session restored ({self.history.message_count()} messages).[/cyan]")

    def clear_history(self) -> None:
        self.history.clear(keep_system=True)
        print("  [cyan]ℹ Conversation history cleared.[/cyan]")
```

#### 注意什么

- `_auto_save` 必须包裹在 `try...except` 块中。会话保存属于非核心功能，绝不能因为磁盘写保护、空间不足等外部 IO 问题导致整个 Agent 执行崩溃。

---

### 步骤 3：创建最简终端 UI 辅助库 `ui.py`

#### 为什么做

接下来的 `__main__.py` 需要打印欢迎横幅、输入提示符、错误信息等格式化输出。我们将这些重复的终端打印逻辑抽取到独立的 `ui.py` 中，保持 `__main__.py` 的整洁。后续章节会逐步丰富这个模块的渲染能力（如颜色、Spinner 动画），但本课只需要最基础的 `print` 封装。

#### 做什么

创建 `ui.py`，定义四个基础输出函数：

```python
# ui.py

def print_welcome() -> None:
    print("  Mini Claude Code — A minimal coding agent\n")
    print("  Type your request, or 'exit' to quit.")
    print("  Commands: /clear /plan /cost\n")


def print_user_prompt() -> None:
    print("\n> ", end="")


def print_error(msg: str) -> None:
    print(f"  [red]Error: {msg}[/red]")


def print_info(msg: str) -> None:
    print(f"  [cyan]ℹ {msg}[/cyan]")
```

#### 注意什么

- 此刻的 `print_info` 和 `print_error` 只是简单的字符串打印，`[red]`/`[cyan]` 标签不会渲染颜色（需要后续引入 `rich` 库）。这不影响功能正确性，先让代码跑起来。

---

### 步骤 4：编写参数解析与会话恢复逻辑

#### 为什么做

我们需要在命令行接收各种执行选项（例如 `--resume`、`--model`），并获取环境变量中的 API Key。同时要实现：若传入了具体的提示词，则直接运行单次任务并退出；若无参数，则启动 REPL 交互环境。

#### 做什么

重写 `__main__.py` 中的 `main()` 函数，接入参数解析与会话恢复流程：

```python
# __main__.py

import os
import sys
import signal
import asyncio
import argparse
from .agent import Agent
from .session import load_session, get_latest_session_id
from .ui import print_welcome, print_user_prompt, print_error, print_info


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(description="Mini Claude Code CLI")
    parser.add_argument("prompt", nargs="*", help="Direct prompt to run")
    parser.add_argument("--model", "-m", default=None, help="Model name")
    parser.add_argument("--api-base", default=None, help="API base URL")
    parser.add_argument("--resume", action="store_true", help="Resume latest session")
    return parser.parse_args()


async def main_async():
    args = parse_args()
    
    # 确定 API 密钥
    api_base = args.api_base or os.environ.get("OPENAI_BASE_URL")
    api_key = os.environ.get("OPENAI_API_KEY") if api_base else os.environ.get("ANTHROPIC_API_KEY")
    
    if not api_key:
        print_error("API Key not found. Please set ANTHROPIC_API_KEY or OPENAI_API_KEY.")
        sys.exit(1)
        
    model = args.model or os.environ.get("MODEL") or ("gpt-4o" if api_base else "claude-sonnet-4-6")
    
    agent = Agent(model=model, api_base=api_base, api_key=api_key)
    
    # 恢复最新会话
    if args.resume:
        latest_id = get_latest_session_id()
        if latest_id:
            session_data = load_session(latest_id)
            if session_data:
                # 只提取消息数据传给 restore_session，过滤掉 metadata 等无关字段
                agent.restore_session({
                    "anthropicMessages": session_data.get("anthropicMessages"),
                    "openaiMessages": session_data.get("openaiMessages"),
                })
                
    prompt = " ".join(args.prompt) if args.prompt else None
    if prompt:
        # 单次执行模式
        await agent.chat(prompt)
    else:
        # 进入交互式 REPL 模式
        await run_repl(agent)


def main():
    try:
        asyncio.run(main_async())
    except KeyboardInterrupt:
        print("\nBye!")
        sys.exit(0)


if __name__ == "__main__":
    main()
```

---

### 步骤 5：构建交互式 REPL 循环与信号拦截

#### 为什么做

在交互模式下，如果大模型输出失控或执行了错误的工具，用户按下 `Ctrl+C` 的首要期望是**终止本次 Agent 执行并退回输入提示符**，而不是直接杀掉整个进程导致历史记录全部丢失。只有在没有任务执行且处于输入状态下连续按下 `Ctrl+C` 时，才应该退出程序。

#### 做什么

在 `__main__.py` 中实现 REPL 循环函数 `run_repl`，并向 Python 系统信号器注册 `SIGINT` (Ctrl+C) 的动态分发拦截：

```python
# __main__.py（续）


async def run_repl(agent: Agent):
    sigint_count = 0

    # 信号处理器
    def handle_sigint(sig, frame):
        nonlocal sigint_count
        # 若 Agent 正在运行且未被标记中断，则终止运行并返回提示符
        if agent._aborted is False:
            agent.abort()
            print("\n  (interrupted)")
            sigint_count = 0
            print_user_prompt()
        else:
            # 空闲时连续按下两次 Ctrl+C 则安全退出
            sigint_count += 1
            if sigint_count >= 2:
                print("\nBye!")
                sys.exit(0)
            print("\n  Press Ctrl+C again to exit.")
            print_user_prompt()

    # 注册信号拦截
    signal.signal(signal.SIGINT, handle_sigint)
    print_welcome()

    while True:
        print_user_prompt()
        try:
            line = input()
        except (EOFError, KeyboardInterrupt):
            # 捕获 EOF (Ctrl+D) 或输入时的 Ctrl+C 退出
            print("\nBye!")
            break

        inp = line.strip()
        sigint_count = 0  # 重置中断计数
        
        if not inp:
            continue
        if inp in ("exit", "quit"):
            print("Bye!")
            break

        # 处理 REPL 命令
        if inp == "/clear":
            agent.clear_history()
            continue
        if inp == "/plan":
            print("  [cyan]ℹ Plan mode toggled (Not fully implemented yet).[/cyan]")
            continue
        if inp == "/cost":
            print("  [cyan]ℹ Cost tracking: $0.00 (Mocked for now).[/cyan]")
            continue

        try:
            await agent.chat(inp)
        except asyncio.CancelledError:
            # 捕获取消异常，防输出红字报错
            pass
        except Exception as e:
            if "abort" not in str(e).lower():
                print_error(str(e))
```

#### 注意什么

- `signal.signal` 只在主线程有效。因为 `input()` 是阻塞性 IO，而在 Windows 和 Unix 下 Python 对 `input()` 中断的处理稍有不同，通过捕获 `KeyboardInterrupt` 异常和信号拦截双重机制可以保证在任意系统下都能优雅退回命令行提示符。

---

## ⚖️ 设计权衡

### 覆盖式写入 JSON vs 追加式写入 JSONL

- **方案 A**：**覆盖式写入 JSON**（我们所用）
  - 每次调用后将完整的会话消息结构序列化为单个 JSON 文件。
  - **优点**：结构简单，读取解析极度方便，原生支持复杂的元数据嵌套。
  - **缺点**：如果对话很长，每次写入的文件较大。
- **方案 B**：**追加式写入 JSONL**
  - 每进行一轮对话，直接在历史文件末尾 append 写入一行新的 JSON。
  - **优点**：写入性能始终为 O(1)，防中途崩溃损坏性极佳。
  - **缺点**：元数据结构维护复杂，需要额外的解析处理才能恢复消息上下文。

**结论**：由于 Mini Agent 会在后续课时中实现上下文压缩，历史消息总数会受到合理限制，JSON 单文件写入的开销微乎其微，因此方案 A 具有更高的代码可维护性。

---

## ⚠️ 常见陷阱

### 1. `input()` 阻塞导致信号无法被即时处理

在执行 `line = input()` 期间，操作系统处理 `Ctrl+C` 会直接在主线程抛出 `KeyboardInterrupt` 异常，而不会顺利进入 `handle_sigint` 回调。

**后果**：用户在空闲输入时按一下 `Ctrl+C` 程序便会直接闪退，体验很差。
**修正**：必须在外层使用 `try...except (EOFError, KeyboardInterrupt):` 捕获此异常，使其安全输出 `Bye!` 并退出，而非让程序抛出冗长的报错堆栈。

---

### 2. 重置 `_aborted` 状态时机不对

如果 `_aborted` 标记在 `chat()` 执行后没有被重置，一旦用户中断了一次任务，随后的所有任务都将因为 `_aborted = True` 而被自动跳过。

**修正**：在 `chat()` 入口函数的第一行，必须强制重置：`self._aborted = False`。

---

## ✅ 验收点

### 输入与执行

1. 直接运行启动 REPL：
   ```bash
   python -m mini_claude
   ```
2. 输入 `hello` 并按下回车，等待 Agent 回复。
3. 输入命令 `/clear`，验证历史是否清空。
4. 输入 `exit` 退出。
5. 运行恢复会话命令：
   ```bash
   python -m mini_claude --resume
   ```

### 预期结果

- 运行 `exit` 退出时显示 `Bye!`。
- 使用 `--resume` 恢复后，终端会打印出类似 `Session restored (X messages)` 的提示词。

---

## 🧠 思考题

1. **为什么在 `chat` 接口的 `finally` 块中调用 `_auto_save`，而不是在 `try` 块的最后一行调用？**
   *(提示：如果用户使用 `Ctrl+C` 强行终止了正在运行的 Agent，代码会抛出异常中断执行，如果放在 try 块末尾，中断前的对话历史就无法被保存。放进 finally 块可确保即便被中断也能保存已生成的内容。)*
2. **在 `run_repl` 中，如果大模型返回了大量的工具调用，我们正在并发执行，此时按 `Ctrl+C` 会如何影响这些正在执行的子进程？**
   *(提示：我们设置了 `agent.abort()` 会将中断标志置为 True。工具分发器在每次执行前或执行中可以检测该标志位，主动中止后续工具的调用。)*

---

## 📦 本节收获

1. **会话持久化**：掌握了利用序列化文件恢复 Agent 工作上下文的设计方法。
2. **信号流拦截**：掌握了利用系统 SIGINT 信号分流“任务中止”与“程序退出”的交互技巧。
3. **REPL 健壮性**：体验了防御性异常拦截（IO 失败降级、阻塞异常捕获），使 CLI 工具具备了生产级的高可用交互体验。

---

> **下一章**：现在我们有了一个易用的终端交互界面。下一章我们将攻克流式输出——让 Agent 的思考过程与工具调用过程流式呈现在终端上。
