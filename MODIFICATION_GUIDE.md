# 教程修改指导文档

本文档说明了对 `python/mini_claude` 源码进行的逻辑修复，以及教程文档中需要相应更新的部分。

---

## 修改概览

| 问题编号 | 问题描述 | 修复状态 | 影响范围 |
|---------|---------|---------|---------|
| #1 | DRY 原则 — 双后端重复代码 | ✅ 已修复 | agent.py |
| #2 | 状态管理混乱 | ✅ 已修复 | agent.py |
| #3 | 副作用隐藏 | ✅ 已修复 | tools.py |
| #4 | 错误处理过于宽泛 | ✅ 已修复 | agent.py, tools.py, session.py, memory.py |
| #5 | 全局状态缓存 | ⚠️ 部分修复 | skills.py, subagent.py |
| #6 | 类型定义模糊 | ✅ 已修复 | tools.py |
| #7 | 魔法数字残留 | ✅ 已修复 | agent.py, tools.py |
| #9 | 异步模式不一致 | ✅ 已确认 | agent.py |
| #10 | MessageHistory 封装性破坏 | ✅ 已修复 | agent.py |
| #11 | 缺少类型注解 | ✅ 已修复 | agent.py |
| #12 | Retry 缺少 jitter | ✅ 已修复 | agent.py |

---

## 详细修改说明

### #1 DRY 原则 — 双后端重复代码

**修改文件**: `agent.py`

**新增抽象**:
```python
class MessageHistory:
    """统一 Anthropic/OpenAI 消息格式的抽象层"""
```

**新增方法**:
- `append_user_message()` — 统一添加用户消息
- `append_assistant_message()` — 统一添加助手消息
- `append_tool_results()` — 统一添加工具结果
- `append_openai_tool_message()` — OpenAI 专用工具消息
- `update_last_user_content()` — 更新最后一条用户消息（用于记忆注入）
- `clear()` — 清空历史
- `update_system_prompt()` — 更新系统提示
- `to_dict()` / `restore()` — 序列化/反序列化

**教程需更新**:
- [docs/02-dual-backend.md](docs/02-dual-backend.md) — 需要新增一节介绍 `MessageHistory` 抽象层
- [docs/07-streaming.md](docs/07-streaming.md) — 更新消息管理相关说明

---

### #2 状态管理混乱

**修改文件**: `agent.py`

**新增数据类**:
```python
@dataclass
class AgentConfig:
    """Agent 配置（相对稳定）"""
    permission_mode: str = "default"
    model: str = "claude-opus-4-6"
    # ...

@dataclass
class AgentState:
    """Agent 运行时状态（频繁变化）"""
    total_input_tokens: int = 0
    total_output_tokens: int = 0
    # ...
```

**教程需更新**:
- [docs/01-agent-loop.md](docs/01-agent-loop.md) — 需要说明 `AgentConfig` 和 `AgentState` 的职责分离
- [docs/13-budget-control.md](docs/13-budget-control.md) — 更新 token 计数相关说明

---

### #3 副作用隐藏

**修改文件**: `tools.py`

**修改内容**:
- `_write_file()` 不再自动调用 `_auto_update_memory_index()`
- 副作用移至 `execute_tool()` 中显式调用

**教程需更新**:
- [docs/03-tools.md](docs/03-tools.md) — 需要说明工具执行的副作用管理原则

---

### #4 错误处理过于宽泛

**修改文件**: `agent.py`, `tools.py`, `session.py`, `memory.py`

**修改内容**:
- 将 `except Exception: pass` 替换为具体的异常类型
- 添加 `logging` 模块进行错误记录
- 区分 `OSError`、`json.JSONDecodeError`、`asyncio.CancelledError` 等

**教程需更新**:
- [docs/01-agent-loop.md](docs/01-agent-loop.md) — 需要说明错误处理最佳实践
- [docs/10-memory.md](docs/10-memory.md) — 更新记忆系统的错误处理说明

---

### #5 全局状态缓存（部分修复）

**现状**:
- `skills.py`、`subagent.py` 中仍有模块级缓存
- 已有 `reset_*_cache()` 函数用于测试

**建议**:
- 在教程中说明测试时需要调用 `reset_*_cache()`
- 或者在测试 fixtures 中使用依赖注入

**教程需更新**:
- [docs/16-testing.md](docs/16-testing.md) — 需要说明缓存重置的测试策略

---

### #6 类型定义模糊

**修改文件**: `tools.py`

**修改内容**:
```python
# 修改前
PermissionMode = str

# 修改后
from typing import Literal
PermissionMode = Literal["default", "plan", "acceptEdits", "bypassPermissions", "dontAsk"]
```

**教程需更新**:
- [docs/08-permissions.md](docs/08-permissions.md) — 需要说明 `Literal` 类型的使用

---

### #7 魔法数字残留

**修改文件**: `agent.py`, `tools.py`

**新增常量**:
```python
# agent.py
CONTEXT_WINDOW_SAFETY_MARGIN = 20000
AUTOCOMPACT_THRESHOLD = 0.85
COST_PER_INPUT_TOKEN = 3 / 1_000_000
COST_PER_OUTPUT_TOKEN = 15 / 1_000_000
LARGE_RESULT_THRESHOLD = 30 * 1024
LARGE_RESULT_PREVIEW_LINES = 200
MAX_RETRIES = 3
MAX_RETRY_DELAY_MS = 30000

# tools.py
MAX_LIST_FILES = 200
MAX_GREP_MATCHES = 100
MAX_GREP_TOTAL = 200
```

**教程需更新**:
- [docs/09-context.md](docs/09-context.md) — 说明上下文窗口相关的常量
- [docs/13-budget-control.md](docs/13-budget-control.md) — 说明成本计算常量

---

### #9 异步模式不一致

**现状分析**:
- `asyncio.create_task` — 用于流式工具执行期间的并发
- `await` — 用于需要顺序执行的场景
- `asyncio.gather` — 用于 OpenAI 并行工具执行

**结论**: 异步模式实际上是一致的，每种模式都有其适用场景。

**教程需更新**:
- [docs/06-streaming-tools.md](docs/06-streaming-tools.md) — 需要更清晰地说明不同异步模式的使用场景

---

### #10 MessageHistory 封装性破坏

**问题描述**:
`_compact_anthropic()` 和 `_compact_openai()` 直接访问了 `MessageHistory` 的私有属性 `_anthropic_messages` 和 `_openai_messages`，破坏了封装性。

**修复方案**:
在 `MessageHistory` 中添加公共方法：
```python
def replace_anthropic_messages(self, messages: list[dict]) -> None:
    """Replace all Anthropic messages (used during compaction)."""
    self._anthropic_messages = messages

def replace_openai_messages(self, messages: list[dict]) -> None:
    """Replace all OpenAI messages (used during compaction)."""
    self._openai_messages = messages
```

**修改内容**:

- `_compact_anthropic()` 改为使用 `self.history.replace_anthropic_messages()`
- `_compact_openai()` 改为使用 `self.history.replace_openai_messages()`

---

### #11 缺少类型注解

**问题描述**:
`_build_side_query()` 方法缺少返回类型注解。

**修复方案**:

```python
# 修改前
def _build_side_query(self):

# 修改后
def _build_side_query(self) -> Callable[[str, str], Awaitable[str]] | None:
```

---

### #12 Retry 缺少 jitter

**问题描述**:
重试逻辑中的指数退避缺少随机抖动（jitter），可能导致多个客户端同时重试时产生 thundering herd 效应。

**修复方案**:

```python
# 修改前
delay = min(1000 * (2 ** attempt), MAX_RETRY_DELAY_MS) / 1000

# 修改后
base_delay = min(1000 * (2 ** attempt), MAX_RETRY_DELAY_MS) / 1000
jitter = (hash(str(time.time())) % 1000) / 1000  # 0-1s random jitter
delay = base_delay + jitter
```

**原理**:

- `base_delay`: 指数退避基础延迟
- `jitter`: 0-1 秒的随机偏移，避免同时重试
- 总延迟 = base_delay + jitter

---

## 新增内容

### MessageHistory 抽象层

在 [docs/02-dual-backend.md](docs/02-dual-backend.md) 中需要新增一节：

```markdown
## 消息历史抽象层

为了避免 Anthropic/OpenAI 双后端的代码重复，我们引入了 `MessageHistory` 类：

\```python
class MessageHistory:
    def append_user_message(self, content: str) -> None: ...
    def append_assistant_message(self, content: Any) -> None: ...
    def append_tool_results(self, results: list[dict]) -> None: ...
    def update_last_user_content(self, suffix: str) -> None: ...
    def clear(self, keep_system: bool = True) -> None: ...
    def to_dict(self) -> dict: ...
    def restore(self, data: dict) -> None: ...
\```

这个抽象层使得：
1. 工具执行循环只需写一次
2. 消息格式转换逻辑集中管理
3. 序列化/反序列化逻辑统一
```

### AgentConfig 和 AgentState

在 [docs/01-agent-loop.md](docs/01-agent-loop.md) 中需要新增一节：

```markdown
## 配置与状态分离

Agent 的属性分为两类：

### AgentConfig（配置）
- 在创建时确定，运行期间基本不变
- 包括：model, permission_mode, max_cost_usd 等

### AgentState（状态）
- 在运行期间频繁变化
- 包括：total_input_tokens, current_turns, aborted 等

这种分离使得：
1. 配置可以轻松序列化和比较
2. 状态变化更容易追踪
3. 测试时可以独立修改配置或状态
```

---

## 测试注意事项

修改后的代码在测试时需要注意：

1. **缓存重置**:
   ```python
   from mini_claude.skills import reset_skill_cache
   from mini_claude.subagent import reset_agent_cache
   from mini_claude.tools import reset_permission_cache

   def setup_function():
       reset_skill_cache()
       reset_agent_cache()
       reset_permission_cache()
   ```

2. **日志验证**:
   ```python
   import logging
   # 验证错误被正确记录
   with caplog.at_level(logging.WARNING):
       # 触发错误
       assert "expected message" in caplog.text
   ```

3. **MessageHistory 测试**:
   ```python
   from mini_claude.agent import MessageHistory

   def test_message_history():
       history = MessageHistory(use_openai=False, system_prompt="test")
       history.append_user_message("hello")
       assert history.message_count() == 1
   ```

---

## 总结

本次修改主要关注：
1. **代码质量** — 消除重复、明确副作用、改进错误处理
2. **可维护性** — 配置/状态分离、统一消息管理
3. **教学价值** — 展示良好的 Python 设计模式

修改后的代码更适合作为学习模板，展示了：
- 数据类的使用（`@dataclass`）
- 类型注解的最佳实践（`Literal`、`Any`）
- 错误处理的分层策略
- 抽象层的设计原则
