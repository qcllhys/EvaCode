# EvaCode 小白学习教程：从 Python 基础到多 Agent 编程助手

> 学习项目：`~/path/to/evacode`  
> 适合读者：会使用 Linux 基本命令，但 Python、LLM Agent 经验较少的学习者  
> 学习目标：先能运行和调试 EvaCode，再能读懂核心代码，最后能独立增加工具、命令、
> Skill、Hook 和 Agent 功能

---

## 开始之前：这份教程怎么学

不要从头硬读两千多行的 `agent.py`。推荐遵循下面的顺序：

1. 先掌握本教程第 1 章的 Python 语法。
2. 跑通第 2 章的模型流式输出。
3. 自己实现第 3 章的小工具。
4. 带着“模型为什么会连续调用工具”的问题学习第 4 章。
5. 再按章节逐个理解权限、MCP、上下文、记忆和多 Agent。
6. 每学完一章，完成章末练习并运行测试。

本教程位于仓库的 `docs` 目录，文中的源码链接都相对于本文件。例如：

- [命令入口](../evacode/__main__.py)
- [Agent 主循环](../evacode/agent.py)
- [工具基类](../evacode/tools/base.py)
- [测试目录](../tests)

### 学习环境准备

```bash
cd ~/path/to/evacode

# 按 uv.lock 安装项目和开发依赖
uv sync --frozen

# 查看命令入口
uv run evacode --help

# 运行一个小测试，确认环境基本正常
uv run pytest -q tests/test_hello_world.py
```

只学习代码、不调用真实模型时，不需要 API Key。调用模型前需要准备
`.evacode/config.yaml`，配置方法见项目中的
[Linux 部署教程](DEPLOYMENT.md)。

### 整体架构先看一眼

```text
用户输入
   │
   ▼
Textual UI / Remote UI / -p 单次调用
   │
   ▼
ConversationManager ── 保存消息、工具调用和工具结果
   │
   ▼
Agent.run() ── 拼 System Prompt、管理循环、权限和上下文
   │
   ▼
LLMClient ── Anthropic / OpenAI / OpenAI-compatible
   │
   ├── 模型直接回答 ───────────────► 结束本轮
   │
   └── 模型请求调用工具
          │
          ▼
      PermissionChecker
          │
          ▼
      ToolRegistry → ReadFile / WriteFile / Bash / Grep ...
          │
          ▼
      工具结果写回 ConversationManager，再问模型
```

理解 EvaCode 的关键只有一句话：

> Agent 不是一个新的模型，而是“模型 + 对话状态 + 工具 + 循环 + 安全规则”。

---

# 第1章：Python 基础，为阅读 EvaCode 打地基

## 1.1 变量和常见数据类型

Python 不需要提前声明变量类型：

```python
project_name = "EvaCode"       # str：字符串
version = 2                    # int：整数
temperature = 0.7              # float：小数
remote_enabled = False         # bool：布尔值
```

EvaCode 中最常见的容器类型：

```python
# list：有顺序，可重复
tools = ["ReadFile", "WriteFile", "Bash"]

# dict：键值映射，工具参数和配置经常用它表示
arguments = {
    "file_path": "README.md",
    "offset": 0,
    "limit": 100,
}

# set：不重复集合，适合记录禁用工具
disabled = {"Bash", "WriteFile"}

# tuple：固定组合
name_and_score = ("ReadFile", 10)
```

读取和修改：

```python
print(tools[0])                 # ReadFile
print(arguments["file_path"]) # README.md

tools.append("Grep")
arguments["limit"] = 200
disabled.add("EditFile")
```

## 1.2 条件、循环和缩进

Python 使用缩进表示代码块，通常每层 4 个空格：

```python
permission_mode = "default"

if permission_mode == "bypassPermissions":
    print("所有工具自动放行")
elif permission_mode == "plan":
    print("只允许规划和只读操作")
else:
    print("危险操作需要确认")
```

遍历工具：

```python
tools = ["ReadFile", "WriteFile", "Bash"]

for index, tool_name in enumerate(tools, start=1):
    print(index, tool_name)
```

列表推导式在项目里很多：

```python
enabled_tools = [name for name in tools if name != "Bash"]
print(enabled_tools)  # ['ReadFile', 'WriteFile']
```

## 1.3 函数、参数和返回值

```python
def build_message(role: str, content: str = "") -> dict[str, str]:
    """创建一条简单消息。"""
    return {
        "role": role,
        "content": content,
    }


message = build_message("user", "请读取 app.py")
```

这里的 `role: str` 是类型提示，`-> dict[str, str]` 表示返回值预计是字典。
类型提示主要帮助人和编辑器理解代码，Python 运行时通常不会强制检查。

常见参数形式：

```python
def demo(required, default_value=10, *args, **kwargs):
    print(required)       # 必填参数
    print(default_value)  # 有默认值
    print(args)           # 多余的位置参数，组成 tuple
    print(kwargs)         # 多余的命名参数，组成 dict
```

## 1.4 类、对象和继承

“类”是模板，“对象”是根据模板创建的实例：

```python
class Counter:
    def __init__(self, start: int = 0) -> None:
        self.value = start

    def add(self, amount: int = 1) -> int:
        self.value += amount
        return self.value


counter = Counter(start=10)
print(counter.add())   # 11
print(counter.add(5))  # 16
```

EvaCode 的所有工具都继承 `Tool`：

```python
class Tool:
    async def execute(self, params):
        raise NotImplementedError


class HelloTool(Tool):
    async def execute(self, params):
        return "hello"
```

父类规定统一接口，子类负责具体实现。这就是“面向接口编程”。Agent 不需要知道每个
工具内部怎么工作，只要知道它们都有 `execute()`。

## 1.5 dataclass：只负责保存数据的类

普通数据类会写很多重复代码，`@dataclass` 可以自动生成初始化方法：

```python
from dataclasses import dataclass


@dataclass
class ToolResult:
    output: str
    is_error: bool = False


ok = ToolResult(output="读取成功")
bad = ToolResult(output="文件不存在", is_error=True)
```

项目中的消息、工具结果和流式事件大量使用 `dataclass`。可对照
[tools/base.py](../evacode/tools/base.py) 和
[conversation.py](../evacode/conversation.py)。

## 1.6 Pydantic：校验模型生成的参数

模型可能生成错误参数，所以工具不能直接相信传入的字典。EvaCode 用 Pydantic 校验：

```python
from pydantic import BaseModel, Field, ValidationError


class ReadParams(BaseModel):
    file_path: str = Field(description="要读取的文件")
    limit: int = Field(default=100, ge=1, le=2000)


params = ReadParams(file_path="README.md", limit=20)
print(params.file_path)

try:
    ReadParams(file_path="README.md", limit=-1)
except ValidationError as exc:
    print("参数错误：", exc)
```

Pydantic 还能生成 JSON Schema。这个 Schema 会发给大模型，让模型知道工具需要哪些
参数：

```python
print(ReadParams.model_json_schema())
```

## 1.7 异常处理

文件、网络和模型接口都可能出错：

```python
from pathlib import Path


try:
    text = Path("not-exists.txt").read_text(encoding="utf-8")
except FileNotFoundError:
    print("文件不存在")
except OSError as exc:
    print("其他系统错误：", exc)
```

工具一般不把异常直接抛给 UI，而是转成统一结果：

```python
try:
    text = Path("demo.txt").read_text(encoding="utf-8")
    result = ToolResult(output=text)
except Exception as exc:
    result = ToolResult(output=f"读取失败：{exc}", is_error=True)
```

## 1.8 pathlib：现代文件操作

```python
from pathlib import Path

root = Path("/tmp/evacode-study")
file_path = root / "notes" / "chapter1.txt"

file_path.parent.mkdir(parents=True, exist_ok=True)
file_path.write_text("第一章完成", encoding="utf-8")
print(file_path.read_text(encoding="utf-8"))
print(file_path.exists())
print(file_path.resolve())
```

EvaCode 是编程助手，文件操作非常多，所以熟悉 `Path` 很重要。

## 1.9 模块和 import

目录中包含 `__init__.py` 时通常可视为 Python 包：

```text
evacode/
├── __init__.py
├── agent.py
└── tools/
    ├── __init__.py
    └── base.py
```

导入方式：

```python
from evacode.agent import Agent
from evacode.tools.base import Tool, ToolResult
```

项目命令入口在 `pyproject.toml`：

```toml
[project.scripts]
evacode = "evacode.__main__:main"
```

含义是执行 `evacode` 命令时，Python 调用 `evacode/__main__.py` 中的 `main()`。

## 1.10 async/await：异步编程基础

调用模型和 MCP 时，大部分时间都在等网络。异步让程序等待期间仍能处理 UI 和其他任务：

```python
import asyncio


async def fetch_model_reply(name: str) -> str:
    print(f"开始请求 {name}")
    await asyncio.sleep(1)  # 模拟网络等待，不阻塞整个线程
    return f"{name} 返回成功"


async def main() -> None:
    results = await asyncio.gather(
        fetch_model_reply("model-a"),
        fetch_model_reply("model-b"),
    )
    print(results)


asyncio.run(main())
```

异步生成器可以一边产生结果，一边让调用方消费，特别适合模型流式输出：

```python
import asyncio
from collections.abc import AsyncIterator


async def stream_text() -> AsyncIterator[str]:
    for word in ["你好", "，", "我是", " EvaCode"]:
        await asyncio.sleep(0.2)
        yield word


async def main() -> None:
    async for chunk in stream_text():
        print(chunk, end="", flush=True)


asyncio.run(main())
```

看到 `async def`、`await`、`async for` 时，可以分别理解成“异步函数”“等待异步结果”
和“逐个读取异步流”。

### 本章练习

1. 用 `dataclass` 定义一个包含 `name`、`category`、`enabled` 的工具信息类。
2. 用 Pydantic 定义 `SearchParams`，包含字符串 `query` 和 1～100 的 `limit`。
3. 写一个异步生成器，逐字输出“我正在学习 EvaCode”。

---

# 第2章：让 AI 开口说话

## 2.1 一次模型请求由什么组成

最简单的模型调用需要：

- `base_url`：模型服务地址。
- `api_key`：访问凭据。
- `model`：模型名称。
- `messages`：对话消息。
- `system`：模型的身份和行为规则。

EvaCode 支持三种协议：

- `anthropic`
- `openai`
- `openai-compat`，适用于 vLLM、Ollama 等兼容接口

## 2.2 配置对象

[config.py](../evacode/config.py) 用 `ProviderConfig` 表示模型配置：

```python
from evacode.config import ProviderConfig


provider = ProviderConfig(
    name="local-qwen",
    protocol="openai-compat",
    base_url="http://127.0.0.1:8000/v1",
    model="qwen-model",
    api_key="local-not-used",
    context_window=128_000,
)
```

实际使用时推荐把配置写进 `.evacode/config.yaml`，不要写死在代码中。

## 2.3 客户端抽象

[client.py](../evacode/client.py) 定义统一的 `LLMClient`。不同厂商虽然请求格式不同，
但 Agent 只依赖统一的 `stream()` 接口：

```python
client = create_client(provider)

async for event in client.stream(conversation, system="你是编程助手"):
    ...
```

这体现了“依赖抽象，不依赖具体厂商”。切换服务商时，Agent 主循环不需要重写。

## 2.4 第一个流式模型示例

把下面内容保存为 `/tmp/evacode_chat.py`，修改模型配置后运行：

```python
import asyncio

from evacode.client import create_client
from evacode.config import ProviderConfig
from evacode.conversation import ConversationManager
from evacode.tools.base import StreamEnd, TextDelta, ThinkingDelta


async def main() -> None:
    provider = ProviderConfig(
        name="study-model",
        protocol="openai-compat",
        base_url="http://127.0.0.1:8000/v1",
        model="your-model-name",
        api_key="local-not-used",
    )

    conversation = ConversationManager()
    conversation.add_user_message("用一句话解释什么是 Python")
    client = create_client(provider)

    async for event in client.stream(
        conversation,
        system="你是一位耐心的 Python 老师。",
        tools=[],
    ):
        if isinstance(event, TextDelta):
            print(event.text, end="", flush=True)
        elif isinstance(event, ThinkingDelta):
            pass  # 如果模型返回思考内容，这里选择不显示
        elif isinstance(event, StreamEnd):
            print(f"\n输入 token: {event.input_tokens}")
            print(f"输出 token: {event.output_tokens}")


asyncio.run(main())
```

运行：

```bash
cd ~/path/to/evacode
uv run python /tmp/evacode_chat.py
```

## 2.5 为什么要流式输出

如果等待完整答案后一次显示，用户可能几十秒看不到任何内容。流式输出把模型生成的
小片段立即传给 UI：

```text
模型：  Py  thon  是  一种  编程  语言
事件：  Δ1  Δ2   Δ3  Δ4   Δ5   Δ6
UI：    收到一个事件就追加显示
```

EvaCode 用 `TextDelta`、`ThinkingDelta`、`ToolCallComplete` 和 `StreamEnd` 等数据类
统一表示事件。

## 2.6 对话状态

```python
from evacode.conversation import ConversationManager


conversation = ConversationManager()
conversation.add_user_message("1 加 1 等于几？")
conversation.add_assistant_message("等于 2。")
conversation.add_user_message("再加 3 呢？")

for message in conversation.history:
    print(message.role, message.content)
```

没有 `ConversationManager`，模型每次只看到当前问题，就无法理解“再加 3”的含义。

### 本章练习

1. 修改 System Prompt，让模型始终用三句话回答。
2. 连续添加两条用户消息，观察模型如何利用历史。
3. 打印所有流式事件的类名，观察服务端实际返回了哪些事件。

---

# 第3章：工具系统

## 3.1 为什么模型需要工具

模型只能生成文本，本身不能真正读取磁盘、写文件或执行命令。工具把模型的“意图”
变成程序动作：

```text
用户：读取 config.py
模型：请求调用 ReadFile({"file_path": "config.py"})
程序：执行 Python 文件读取
程序：把结果返回给模型
模型：根据文件内容回答用户
```

默认工具在 [tools/__init__.py](../evacode/tools/__init__.py) 注册，包括：

- `ReadFile`：读文件
- `WriteFile`：写文件
- `EditFile`：局部编辑
- `Bash`：执行命令
- `Glob`：按文件名查找
- `Grep`：按文本内容查找

## 3.2 Tool 基类

[tools/base.py](../evacode/tools/base.py) 规定每个工具至少要有：

```python
class Tool:
    name: str
    description: str
    params_model: type[BaseModel]
    category: str

    async def execute(self, params: BaseModel) -> ToolResult:
        ...
```

`category` 有三类：

- `read`：只读
- `write`：写入
- `command`：执行系统命令

权限系统会根据类别决定自动放行还是询问用户。

## 3.3 从零写一个行数统计工具

先新建 `evacode/tools/count_lines.py`：

```python
from pathlib import Path

from pydantic import BaseModel, Field

from evacode.tools.base import Tool, ToolResult


class CountLinesParams(BaseModel):
    file_path: str = Field(description="要统计行数的文件路径")
    ignore_empty: bool = Field(default=False, description="是否忽略空行")


class CountLines(Tool):
    name = "CountLines"
    description = "Count lines in a UTF-8 text file."
    params_model = CountLinesParams
    category = "read"
    is_concurrency_safe = True

    async def execute(self, params: CountLinesParams) -> ToolResult:
        path = Path(params.file_path)

        if not path.is_file():
            return ToolResult(
                output=f"文件不存在：{params.file_path}",
                is_error=True,
            )

        try:
            lines = path.read_text(encoding="utf-8").splitlines()
        except (OSError, UnicodeError) as exc:
            return ToolResult(output=f"读取失败：{exc}", is_error=True)

        if params.ignore_empty:
            lines = [line for line in lines if line.strip()]

        return ToolResult(output=f"{params.file_path} 共 {len(lines)} 行")
```

在 `create_default_registry()` 中注册：

```python
from evacode.tools.count_lines import CountLines

# 其他 registry.register(...) 后面加入
registry.register(CountLines())
```

## 3.4 不启动 UI，单独测试工具

```python
import asyncio

from evacode.tools.count_lines import CountLines, CountLinesParams


async def main() -> None:
    tool = CountLines()
    params = CountLinesParams(
        file_path="pyproject.toml",
        ignore_empty=True,
    )
    result = await tool.execute(params)
    print(result.output)
    print("是否失败：", result.is_error)


asyncio.run(main())
```

## 3.5 ToolRegistry 做了什么

`ToolRegistry` 是一个以工具名为键的字典管理器：

```python
registry.register(CountLines())
tool = registry.get("CountLines")
registry.disable("Bash")
print(registry.is_enabled("Bash"))  # False
```

发送给模型的不是 Python 对象，而是 JSON Schema：

```python
schema = tool.get_schema()
print(schema)
```

大致输出：

```json
{
  "name": "CountLines",
  "description": "Count lines in a UTF-8 text file.",
  "input_schema": {
    "type": "object",
    "properties": {
      "file_path": {"type": "string"},
      "ignore_empty": {"type": "boolean", "default": false}
    },
    "required": ["file_path"]
  }
}
```

## 3.6 一个好工具的设计原则

1. 名字清楚，例如 `ReadFile`，不要叫 `DoSomething`。
2. 描述说明“什么时候用”和“返回什么”。
3. 参数尽量少，字段描述明确。
4. 不把异常泄漏到主循环，返回 `ToolResult(is_error=True)`。
5. 正确标记 `read`、`write` 或 `command`。
6. 写操作要考虑覆盖、并发和恢复。
7. 工具做一件事，复杂流程交给 Agent 组合。

### 本章练习

1. 实现 `FileSize` 工具，返回文件字节数。
2. 实现 `ListPythonFiles` 工具，返回目录下的 `.py` 文件。
3. 给两个工具分别写成功和失败测试。

---

# 第4章：让 Agent 自己干活

## 4.1 Agent 循环

模型调用一次工具还不够。一个真实任务可能是：先查找文件，再读取，再修改，最后运行
测试。Agent 使用循环反复询问模型：

```python
while True:
    response = await model(conversation, tools)

    if response 没有工具调用:
        保存最终回答
        break

    保存模型的工具调用

    for tool_call in response.tool_calls:
        result = await execute_tool(tool_call)
        保存工具结果

    # 带着工具结果进入下一轮
```

真正实现位于 [agent.py](../evacode/agent.py) 的 `Agent.run()`。

## 4.2 Agent 的主要成员

```python
agent = Agent(
    client=client,                   # 如何访问模型
    registry=registry,               # 能使用哪些工具
    protocol="openai-compat",       # 工具消息按哪种协议转换
    work_dir="/path/to/project",    # 工作范围
    permission_checker=checker,      # 工具调用前如何判断权限
    context_window=128_000,          # 上下文容量
    instructions_content="项目规则", # 项目指令
    memory_manager=memory_manager,   # 长期记忆
    hook_engine=hook_engine,         # 生命周期自动化
)
```

## 4.3 一轮执行的真实顺序

```text
1. 注入环境、项目指令和记忆
2. 执行 turn_start / pre_send Hooks
3. 获取当前可见工具的 Schema
4. 处理过大的工具结果
5. 判断是否需要压缩上下文
6. 流式调用模型
7. 收集文字、思考和工具调用事件
8. 对工具执行权限判断
9. 执行工具并把结果加入对话
10. 如果还有工具调用，回到第 2 步
11. 没有工具调用，输出最终答案并结束
```

## 4.4 yield 为什么重要

`Agent.run()` 是异步生成器。它不会只在最后返回一个大结果，而是不断 `yield` 事件：

```python
async for event in agent.run(conversation):
    if isinstance(event, StreamText):
        print(event.text, end="")
    elif isinstance(event, ToolUseEvent):
        print("模型要调用：", event.tool_name)
    elif isinstance(event, ToolResultEvent):
        print("工具结果：", event.output)
    elif isinstance(event, LoopComplete):
        print("循环结束，共", event.total_turns, "轮")
```

同一个 Agent 内核因此能同时被终端 UI、浏览器 UI 和命令行模式使用。

## 4.5 什么时候结束循环

主要结束条件是：模型响应中没有工具调用。其他保护包括：

- 达到 `max_iterations`。
- 连续多次请求不存在的工具。
- 用户取消。
- 网络、认证或上下文错误无法恢复。

Agent 还会处理 `max_tokens`，尝试提高输出上限或要求模型从中断位置继续。

## 4.6 观察 Agent 的最好方法

不要只读代码，可以运行单次任务并看 NDJSON：

```bash
cd /tmp/your-study-project
~/path/to/evacode/.venv/bin/evacode \
  -p '读取当前目录的所有 Python 文件并概括' \
  --output-format stream-json
```

重点观察 `tool_use`、`tool_result`、文本事件和 token 事件的顺序。

### 本章练习

1. 画出“用户要求修改文件”时的事件时序图。
2. 临时把 `max_iterations` 设为 2，观察复杂任务如何中断。
3. 在 `Agent.run()` 循环开头增加调试日志，打印当前轮数和消息数。

---

# 第5章：System Prompt 设计

## 5.1 User Prompt 与 System Prompt

- User Prompt：用户本轮说的话。
- System Prompt：开发者给模型设定的长期角色、规则和工具使用方式。

```python
system = "你是谨慎的代码助手。修改前必须先读取文件。"
user = "帮我调整配置。"
```

如果只写 User Prompt，模型不知道项目约束和安全习惯。EvaCode 的系统提示词集中在
[prompts.py](../evacode/prompts.py)。

## 5.2 好的 System Prompt 包含什么

1. 身份：你是什么助手。
2. 目标：完成编码任务。
3. 行为边界：什么能做，什么不能做。
4. 工具规则：何时读取、何时修改、如何验证。
5. 输出风格：简洁还是详细。
6. 特殊模式：Plan、Coordinator 等。

简单示例：

```python
SYSTEM_PROMPT = """
你是一个 Python 学习助手。

规则：
1. 修改文件之前先使用 ReadFile。
2. 不确定时先搜索代码，不要猜测。
3. 修改后运行最小相关测试。
4. 不执行删除根目录、磁盘格式化等危险命令。
5. 最终回答说明改了什么以及测试结果。
""".strip()
```

## 5.3 动态提示词

EvaCode 的 Prompt 不是固定字符串，还会动态加入：

- 当前工作目录和环境信息。
- 已激活的 Skills。
- 可用 Agent 清单。
- Hook 注入的提示。
- Plan Mode 规则。
- 长期记忆和项目指令。

```python
system = build_system_prompt(
    hook_prompts=["提交前运行测试"],
    coordinator_mode=False,
    agent_catalog=[("review", "负责代码审查")],
)
```

## 5.4 EVACODE.md 的作用

`EVACODE.md` 是给 Agent 看的项目说明，不是 Python 配置。加载顺序可查看
[memory/instructions.py](../evacode/memory/instructions.py)：

```text
~/.evacode/EVACODE.md
项目目录链中的 EVACODE.md / AGENTS.md
工作目录/.evacode/INSTRUCTIONS.md
工作目录/EVACODE.local.md
```

示例：

```markdown
# 项目规则

- Python 版本为 3.11。
- 修改工具时必须补充 pytest 测试。
- 不允许直接修改生产配置。
- 搜索文件优先使用 rg。
```

## 5.5 Prompt 常见错误

- 规则互相冲突。
- 只写口号，没有可执行步骤。
- 一次塞入大量无关文档，浪费 token。
- 把 API Key 写进 Prompt。
- 认为 Prompt 能代替程序权限控制。

Prompt 只能影响模型，不能保证安全。真正的安全边界必须由第 6 章的代码实现。

### 本章练习

1. 为一个“只做代码审查、不修改文件”的 Agent 写 System Prompt。
2. 在临时项目创建 `EVACODE.md`，观察回答是否遵守其中规则。
3. 制造两条冲突规则，观察模型表现并改成单一明确规则。

---

# 第6章：权限系统，给 Agent 装上安全刹车

## 6.1 为什么不能只相信模型

模型可能误解任务，也可能被文件内容中的恶意指令诱导。权限系统必须在模型之外判断
工具动作是否允许。

[permissions/modes.py](../evacode/permissions/modes.py) 定义四种模式：

| 模式 | 读取 | 写入 | 命令 |
| --- | --- | --- | --- |
| `default` | 自动允许 | 询问 | 询问 |
| `acceptEdits` | 自动允许 | 自动允许 | 询问 |
| `plan` | 主要只读，仅允许计划文件 | 受限 | 受限 |
| `bypassPermissions` | 自动允许 | 自动允许 | 自动允许 |

学习和生产环境优先使用 `default`，不要为了省事长期使用 `bypassPermissions`。

## 6.2 分层判断

`PermissionChecker.check()` 大致按以下顺序判断：

```text
Plan 模式例外
  ↓
安全只读命令
  ↓
危险命令黑名单
  ↓
OS 沙箱规则
  ↓
文件路径沙箱
  ↓
用户/项目权限规则
  ↓
会话级“始终允许”
  ↓
权限模式默认行为
  ↓
询问用户
```

靠后的“允许”不能覆盖前面已经命中的危险命令拒绝。

## 6.3 直接体验权限判断

```python
from evacode.permissions import (
    DangerousCommandDetector,
    PathSandbox,
    PermissionChecker,
    PermissionMode,
    RuleEngine,
)
from evacode.tools import create_default_registry


registry = create_default_registry()
checker = PermissionChecker(
    detector=DangerousCommandDetector(),
    sandbox=PathSandbox("/tmp/study-project"),
    rule_engine=RuleEngine(),
    mode=PermissionMode.DEFAULT,
)

read_tool = registry.get("ReadFile")
bash_tool = registry.get("Bash")

print(checker.check(read_tool, {"file_path": "/tmp/study-project/a.py"}))
print(checker.check(bash_tool, {"command": "git status"}))
print(checker.check(bash_tool, {"command": "rm -rf /"}))
```

你应该重点看 `Decision.effect`：`allow`、`deny` 或 `ask`。

## 6.4 工具类别必须正确

如果一个删除文件的工具错误标成 `read`，默认模式可能自动放行。因此新增工具时，
下面这行不是装饰：

```python
class DeleteFile(Tool):
    category = "write"  # 绝不能写成 read
```

## 6.5 权限测试示例

```python
def test_dangerous_command_is_denied(checker, registry):
    bash = registry.get("Bash")
    decision = checker.check(bash, {"command": "rm -rf /"})

    assert decision.effect == "deny"
    assert "危险" in decision.reason
```

### 本章练习

1. 比较四种 PermissionMode 对 `WriteFile` 的判断。
2. 测试工作目录外的文件路径。
3. 给第 3 章自定义工具选择正确类别并编写权限测试。

---

# 第7章：MCP 协议——开放式工具生态

## 7.1 MCP 是什么

MCP（Model Context Protocol）可以理解成“Agent 工具的通用插座”。EvaCode 不必把
数据库、浏览器、企业系统等所有工具写进自身代码，只要连接提供 MCP 的服务即可。

```text
EvaCode ── MCP Client ── MCP Server ── 外部数据或服务
```

核心目录是 [mcp/](../evacode/mcp)。

## 7.2 两种连接方式

### stdio：启动本地子进程

```yaml
mcp_servers:
  - name: context7
    command: npx
    args: ["-y", "@upstash/context7-mcp"]
```

EvaCode 启动子进程，通过标准输入输出交换 JSON-RPC 消息。

### HTTP：连接远程服务

```yaml
mcp_servers:
  - name: company-tools
    url: https://mcp.example.com/mcp
    transport: http
    headers:
      Authorization: "Bearer ${MY_MCP_TOKEN}"
```

环境变量：

```bash
export MY_MCP_TOKEN='replace-with-real-token'
```

## 7.3 工具如何进入 Registry

大致过程：

```python
manager = MCPManager()
manager.load_configs(configs)
result = await manager.connect_and_register_all_tools(registry)

for server in result.servers:
    print(server.name, server.instructions)

for error in result.errors:
    print("连接失败：", error)
```

MCP 工具会被包装成符合 EvaCode `Tool` 接口的对象，然后注册到同一个
`ToolRegistry`。对于 Agent 来说，本地工具和 MCP 工具的调用方式一致。

## 7.4 调试 MCP

按顺序检查：

1. `command` 是否在 `PATH` 中，例如 `which npx`。
2. 子进程是否能独立启动。
3. HTTP URL、证书和代理是否正确。
4. Token 环境变量是否真的传入 EvaCode 进程。
5. 查看 `.evacode/debug.log`。
6. 在 EvaCode 中使用 `/mcp` 查看状态。

### 本章练习

1. 在配置中临时设 `mcp_servers: []`，观察启动差异。
2. 配置一个 MCP Server，并打印注册前后的工具名称。
3. 思考：为什么 MCP Server 返回的说明也需要加入 Prompt？

---

# 第8章：上下文管理——当 Token 开始烧钱

## 8.1 什么是上下文窗口

模型每次请求能看到的 token 数量有限。对话、System Prompt、工具定义、文件内容、工具
结果都会占用窗口：

```text
总上下文 = System Prompt
         + 历史消息
         + 工具 Schema
         + 工具调用与结果
         + 本轮待生成空间
```

窗口满了会导致请求失败、答案截断或成本增加。

## 8.2 粗略估算 token

EvaCode 在没有 API 精确统计时，按约 3.5 字符一个 token 粗估：

```python
from evacode.conversation import ConversationManager


conversation = ConversationManager()
conversation.add_user_message("请解释上下文窗口" * 100)
print(conversation.current_tokens())
```

API 返回真实用量后，`record_usage_anchor()` 会建立更准确的基线，只估算之后新增的消息。

## 8.3 两层控制

[context/manager.py](../evacode/context/manager.py) 主要处理两类问题：

1. 工具结果预算：超长文件内容保存到磁盘，对话中只留预览和路径。
2. 自动 Compact：接近上下文上限时，用模型总结早期历史，保留最近关键消息。

```text
很长的 ReadFile 输出
       │
       ├── 原文持久化到 .evacode/session/tool-results/
       └── 对话里保留摘要和恢复信息
```

## 8.4 为什么不能简单删除最早消息

工具调用和工具结果必须成对存在。如果只删掉工具调用，却留下工具结果，模型 API 可能
认为消息格式非法。Compact 还必须保留：

- 当前任务目标。
- 已做的重要决定。
- 修改过的文件。
- 未完成事项。
- 最近读取的文件和激活的 Skill。

## 8.5 一个简单的滑动窗口示例

下面只是教学示例，不等于项目中的完整算法：

```python
def keep_recent(messages: list[str], max_chars: int) -> list[str]:
    kept: list[str] = []
    used = 0

    for message in reversed(messages):
        if used + len(message) > max_chars:
            break
        kept.append(message)
        used += len(message)

    return list(reversed(kept))
```

### 本章练习

1. 构造 100 条消息并观察 `current_tokens()`。
2. 阅读 `apply_tool_result_budget()`，找出超长结果存在哪里。
3. 思考工具调用与工具结果为什么不能拆开。

---

# 第9章：记忆系统——跨会话的 Agent 记忆

## 9.1 上下文与记忆的区别

- 上下文：当前会话中模型直接看到的内容，容量有限。
- 记忆：保存到磁盘的长期信息，可在以后会话重新加载。

```text
今天：用户说“这个项目使用 pytest”
        ↓ 提取
.evacode/memory/project_testing.md
        ↓ 明天启动时加载
模型仍知道项目使用 pytest
```

## 9.2 两个记忆范围

- 用户级：`~/.evacode/memory/`
- 项目级：`<project>/.evacode/memory/`

用户偏好适合用户级，项目架构信息适合项目级。

## 9.3 MemoryManager 基本使用

```python
from evacode.memory import MemoryManager


manager = MemoryManager("/tmp/study-project")

print("用户记忆目录：", manager.user_mem_dir)
print("项目记忆目录：", manager.project_mem_dir)
print("注入模型的记忆：")
print(manager.load())

for memory in manager.load_all():
    print(memory)
```

真实提取由 `MemoryManager.extract()` 异步调用模型完成。Agent 不会每轮都提取，而是按
间隔后台执行，避免拖慢主回答。

## 9.4 一份记忆文件长什么样

```markdown
---
name: testing-framework
description: 项目测试框架
metadata:
  type: project
---

本项目使用 pytest，新增工具时应补充异步测试。
```

好的记忆应该是稳定事实，不应保存短期日志、猜测或密钥。

## 9.5 记忆治理

记忆越来越多时可能重复或冲突。`memory/consolidation.py` 会尝试：

- 合并重复记忆。
- 清理过期内容。
- 更新 `MEMORY.md` 索引。
- 通过锁避免多个整理任务并发修改。

### 本章练习

1. 手工创建一份项目记忆并用 `manager.load()` 读取。
2. 区分“用户喜欢中文回答”和“项目用 Python 3.11”应该放在哪里。
3. 设计一条不应该进入长期记忆的临时信息。

---

# 第10章：Slash Command——内置命令框架

## 10.1 Slash Command 与普通对话

以 `/` 开头的输入优先由本地命令框架解析，例如：

```text
/help
/status
/clear
/compact
/memory
/permission
/mcp
/session
/plan
```

这些命令不一定发送给模型。有些只操作本地 UI 或会话状态。

## 10.2 Command 数据结构

[commands/registry.py](../evacode/commands/registry.py) 中：

```python
@dataclass
class Command:
    name: str
    description: str
    type: CommandType
    handler: CommandHandler
    aliases: list[str]
    usage: str
```

`CommandRegistry` 负责注册、别名冲突检查、查找和列表展示。

## 10.3 写一个 `/hello` 命令

```python
from evacode.commands.registry import (
    Command,
    CommandContext,
    CommandType,
)


async def handle_hello(ctx: CommandContext) -> None:
    name = ctx.args.strip() or "学习者"
    ctx.ui.add_system_message(f"你好，{name}！欢迎学习 EvaCode。")


HELLO_COMMAND = Command(
    name="hello",
    description="显示学习欢迎语",
    type=CommandType.LOCAL_UI,
    handler=handle_hello,
    aliases=["hi"],
    usage="/hello [名字]",
)
```

然后在命令注册处加入：

```python
registry.register_sync(HELLO_COMMAND)
```

## 10.4 命令测试

```python
def test_hello_registered(command_registry):
    command = command_registry.find("hello")
    assert command is not None
    assert command.usage == "/hello [名字]"
    assert command_registry.find("hi") is command
```

### 本章练习

1. 实现 `/where`，显示当前工作目录。
2. 给 `/where` 增加别名 `/pwd`。
3. 尝试注册重复别名，观察 `ValueError`。

---

# 第11章：Skill 系统——可复用的技能包

## 11.1 Skill 与 Tool 的区别

- Tool 是程序能力，例如读文件、执行命令。
- Skill 是完成某类任务的方法、知识和流程，通常写在 Markdown 中。

例如“前端设计 Skill”可能要求 Agent 先分析布局，再选择色彩，最后检查响应式效果；
它会组合使用 ReadFile、WriteFile、Bash 等工具。

## 11.2 Skill 目录

加载顺序包括：

```text
~/.evacode/skills/              用户级
<work_dir>/.evacode/skills/     项目级
内置 Skills
```

一个最小 Skill：

```text
.evacode/skills/python-review/
└── SKILL.md
```

`SKILL.md` 示例：

```markdown
---
name: python-review
description: 审查 Python 代码质量和常见错误
---

# Python Review

1. 先读取目标文件和相关测试。
2. 检查异常处理、类型提示和资源释放。
3. 检查是否存在危险的宽泛异常。
4. 只报告有证据的问题，给出文件和行号。
5. 用户未要求修改时不要编辑文件。
```

## 11.3 加载与激活

```python
from evacode.skills.loader import SkillLoader


loader = SkillLoader("/tmp/study-project")
skills = loader.load_all()

for name, skill in skills.items():
    print(name, skill.description)

review_skill = loader.get("python-review")
```

启动时通常只把 Skill 名称和简短描述告诉模型；模型判断需要时再调用 `LoadSkill` 加载
完整正文。这叫“渐进式披露”，可以节约上下文。

## 11.4 Skill 设计原则

1. 触发范围明确。
2. 步骤可执行，不写空泛口号。
3. 说明禁止事项和结束条件。
4. 大型参考材料按需读取，不全部塞入 `SKILL.md`。
5. Skill 负责流程，Tool 负责实际动作。

### 本章练习

1. 创建 `project-summary` Skill。
2. 用 `SkillLoader` 确认能扫描到它。
3. 比较“把所有 Skill 全文放入 Prompt”和按需加载的 token 差异。

---

# 第12章：Hook 系统——生命周期钩子与自动化

## 12.1 什么是 Hook

Hook 是在特定生命周期事件发生时自动执行的动作：

```text
session_start
turn_start
pre_send
post_receive
pre_tool_use
post_tool_use
turn_end
session_end
```

常见用途：记录日志、自动检查、通知、阻止某类工具调用、为 Prompt 补充规则。

## 12.2 Hook 数据结构

```python
hook = Hook(
    id="log-bash",
    event="pre_tool_use",
    action=Action(type="command", command="echo tool=$TOOL_NAME"),
    condition=parse_condition('tool == "Bash"'),
    reject=False,
)
```

`HookContext` 提供 `$EVENT`、`$TOOL_NAME`、`$FILE_PATH`、`$MESSAGE` 和
`$TOOL_ARGS.xxx` 等变量。

## 12.3 拦截危险模式示例

```python
from evacode.hooks.conditions import parse_condition
from evacode.hooks.engine import HookEngine
from evacode.hooks.models import Action, Hook, HookContext


hook = Hook(
    id="block-force-push",
    event="pre_tool_use",
    action=Action(
        type="command",
        command="echo force push is blocked",
    ),
    condition=parse_condition(
        'tool == "Bash" && args.command =~ /git\\s+push.*--force/'
    ),
    reject=True,
)

engine = HookEngine([hook])
context = HookContext(
    event_name="pre_tool_use",
    tool_name="Bash",
    tool_args={"command": "git push --force origin main"},
)
```

## 12.4 同步与异步 Hook

- 同步 Hook：主流程等待执行结束，适合 `pre_tool_use` 拒绝判断。
- 异步 Hook：后台运行，适合耗时通知，不应阻塞回答。

```python
Hook(
    id="notify-finished",
    event="turn_end",
    action=Action(type="command", command="notify-send 'EvaCode 完成'"),
    async_exec=True,
)
```

## 12.5 Hook 不是权限系统的替代品

Hook 更灵活，但核心危险命令仍应由 `PermissionChecker` 和 OS 沙箱保护。安全控制应尽量
默认拒绝，并编写测试。

### 本章练习

1. 写一个只对 `WriteFile` 生效的日志 Hook。
2. 使用正则条件匹配 `.env` 文件。
3. 测试 `once=True` 的 Hook 是否只执行一次。

---

# 第13章：SubAgent——子 Agent 与任务分发

## 13.1 为什么需要子 Agent

主 Agent 可以把相对独立的任务交给子 Agent，例如：

- 一个子 Agent 查文档。
- 一个子 Agent分析测试失败。
- 一个子 Agent审查安全问题。
- 主 Agent 汇总结果并实施。

这样能减少主上下文污染，并允许部分任务并行。

## 13.2 关键组件

核心目录为 [agents/](../evacode/agents)：

- `loader.py`：加载 Agent 定义。
- `task_manager.py`：管理任务状态。
- `trace.py`：记录父子调用链。
- `fork.py`：创建隔离执行上下文。
- `agent_tool.py`：把子 Agent 能力暴露成工具。

## 13.3 一个 Agent 定义示例

```markdown
---
name: python-test-reviewer
description: 分析 Python 测试失败，只报告原因，不修改代码
permissionMode: default
tools:
  - ReadFile
  - Grep
  - Glob
  - Bash
---

你是测试失败分析员。

1. 只运行最小相关测试。
2. 给出失败断言、调用路径和根因。
3. 不修改文件。
4. 返回简洁、可验证的结论。
```

项目级 Agent 通常放在：

```text
<work_dir>/.evacode/agents/
```

## 13.4 任务状态

典型状态流：

```text
pending → running → completed
                  ↘ failed
                  ↘ cancelled
```

主 Agent 不能只“启动后忘记”，还要接收完成通知、错误和结果。

## 13.5 什么时候不该使用子 Agent

- 任务只有一个简单读取动作。
- 子任务强依赖主 Agent 每一步的实时判断。
- 多个 Agent 会同时修改同一文件。
- 创建和汇总成本高于直接完成。

### 本章练习

1. 写一个只读的目录分析 Agent 定义。
2. 列出一个适合并行和一个不适合并行的任务。
3. 阅读 `TaskManager`，画出任务状态变化。

---

# 第14章：Worktree——Git 工作树并行开发

## 14.1 Git worktree 是什么

同一个 Git 仓库通常一次只能检出一个分支。`git worktree` 可以让多个目录分别检出不同
分支，共享同一份 Git 对象数据库：

```text
main repo/              main 分支
.evacode/worktrees/a/   feature-a 分支
.evacode/worktrees/b/   feature-b 分支
```

这适合让多个 Agent 在隔离目录中并行修改，减少文件冲突。

## 14.2 基本 Git 示例

先在普通练习仓库理解原生命令：

```bash
mkdir -p /tmp/worktree-study
cd /tmp/worktree-study
git init
git config user.email study@example.com
git config user.name Study
echo hello > README.md
git add README.md
git commit -m init

git worktree add ../worktree-feature -b feature/demo
git worktree list
```

## 14.3 EvaCode 的 WorktreeManager

[worktree/manager.py](../evacode/worktree/manager.py) 封装：

```python
manager = WorktreeManager(repo_root="/path/to/git-project")

worktree = await manager.create("feature-demo")
session = await manager.enter("feature-demo")
print(session.path)

# 完成后退出，是否保留或合并由上层流程决定
await manager.exit(...)
```

它还处理恢复会话、列出 worktree、过期清理和共享大目录的软链接配置。

## 14.4 常见风险

- 未提交修改导致切换或清理失败。
- 两个分支最终合并时仍可能冲突。
- 把 `.venv`、`node_modules` 重复复制，浪费空间。
- Agent 删除仍在使用的 worktree。
- 在非 Git 目录中调用 worktree。

### 本章练习

1. 在 `/tmp` 创建两个 worktree，并分别修改不同文件。
2. 分别修改同一行，手工体验合并冲突。
3. 阅读 `WorktreeManager.create()` 的失败回滚逻辑。

---

# 第15章：Agent Teams——从一次性子任务到长期协作

## 15.1 Team 与 SubAgent 的区别

- SubAgent：通常完成一个任务后返回结果。
- Agent Team：多个有身份的成员共享任务、邮箱和进度，适合持续协作。

```text
Lead Agent
├── researcher
├── implementer
└── reviewer

共享：任务列表、邮箱、进度、trace、可能的 worktree
```

## 15.2 核心组件

[teams/](../evacode/teams) 包括：

- `manager.py`：团队生命周期。
- `models.py`：团队和成员模型。
- `mailbox.py`：成员间消息。
- `shared_task.py`：共享任务。
- `progress.py`：进度状态。
- `coordinator.py`：Lead 协调规则。
- `spawn_inprocess.py`、`spawn_tmux.py`：不同运行后端。

## 15.3 创建团队的概念示例

```python
from evacode.teams.manager import TeamManager
from evacode.teams.models import TeammateInfo


manager = TeamManager(
    worktree_manager=worktree_manager,
    trace_manager=trace_manager,
)

team = manager.create_team(
    name="feature-team",
    lead_agent_id="lead-agent-1",
    description="完成一个 Python 功能并审查",
    teammate_mode="in-process",
    is_interactive=False,
)

manager.register_member(
    team_name="feature-team",
    member=TeammateInfo(
        name="reviewer",
        agent_id="agent-reviewer-1",
        agent_type="python-reviewer",
        model="study-model",
        worktree_path="/tmp/reviewer-worktree",
        backend_type="in-process",
        is_active=True,
    ),
)
```

这个例子只创建团队元数据并注册成员；真正启动成员还需要由上层的 Team 工具选择后端、
创建 Agent 和工作目录。

## 15.4 邮箱与共享任务

成员之间不应直接修改彼此内部状态，而是发送消息：

```text
implementer → mailbox → reviewer
reviewer → mailbox → lead
```

共享任务需要明确：

- 负责人是谁。
- 当前状态。
- 依赖哪些任务。
- 完成标准。
- 结果和错误。

## 15.5 协调模式

开启 `enable_coordinator_mode` 后，Lead 的主要职责是分解、派发、检查和汇总，而不是亲自
写代码。这样能避免 Lead 一边修改文件、一边与成员产生冲突。

## 15.6 多 Agent 的真实代价

多 Agent 不是越多越好：

- 消耗更多 token。
- 结果汇总更复杂。
- 并发写文件可能冲突。
- 错误可能在成员间传播。
- 任务边界不清时会重复工作。

优先把任务拆成边界清楚、输出可验证的子任务。

### 本章练习

1. 为“新增工具”设计 implementer、tester、reviewer 三个角色。
2. 为每个角色写输入、输出和完成标准。
3. 设计一条规则，避免两个成员同时编辑同一文件。

---

# 第16章：终端 UI、远程 UI 与事件驱动

## 16.1 三种入口共用一个内核

[__main__.py](../evacode/__main__.py) 根据参数选择运行方式：

```text
evacode                    → Textual 终端界面
evacode -p "任务"          → 非交互单次调用
evacode --remote           → WebSocket + 浏览器 UI
```

它们都使用同一套 Config、Client、Agent、Tools 和 Permission。

## 16.2 Textual 的基本思想

[app.py](../evacode/app.py) 中 `EvaCodeApp` 继承 Textual 的 `App`：

```python
class EvaCodeApp(App):
    CSS_PATH = "styles.tcss"

    def compose(self) -> ComposeResult:
        yield Static("EvaCode")
        yield VerticalScroll(id="chat-area")
        yield ChatInput(id="chat-input")

    async def on_chat_input_submitted(self, event):
        # 用户提交后启动 Agent，而不是阻塞 UI
        ...
```

这叫事件驱动：程序平时等待键盘、鼠标、网络或 Agent 事件，事件发生时调用对应处理函数。

## 16.3 为什么 UI 不应该直接调用厂商 SDK

如果 UI 直接写 OpenAI 请求：

- 远程 UI 要重复实现。
- 测试难以 Mock。
- 切换协议要修改 UI。
- 工具和权限逻辑容易分叉。

正确分层是：

```text
UI → Agent → LLMClient
```

UI 只展示事件和收集用户输入。

## 16.4 Remote UI 安全

远程模式监听 `0.0.0.0:18888`，当前没有内置登录鉴权。学习时优先本机使用，不要直接
暴露公网。部署细节见 [Linux 部署教程](DEPLOYMENT.md)。

### 本章练习

1. 找到 `EvaCodeApp.compose()` 中所有组件。
2. 找到 Agent 的 `StreamText` 如何更新聊天区域。
3. 增加一个仅显示当前模型名的静态组件。

---

# 第17章：测试、调试和阅读大型项目的方法

## 17.1 pytest 基础

最简单的测试：

```python
def add(a: int, b: int) -> int:
    return a + b


def test_add():
    assert add(1, 2) == 3
```

异步测试：

```python
import pytest


@pytest.mark.asyncio
async def test_count_lines(tmp_path):
    file_path = tmp_path / "demo.txt"
    file_path.write_text("a\n\nb\n", encoding="utf-8")

    tool = CountLines()
    result = await tool.execute(
        CountLinesParams(
            file_path=str(file_path),
            ignore_empty=True,
        )
    )

    assert result.is_error is False
    assert "2 行" in result.output
```

`tmp_path` 是 pytest 提供的临时目录，测试结束后自动清理。

## 17.2 常用测试命令

```bash
cd ~/path/to/evacode

# 全部测试
uv run pytest -q

# 单个文件
uv run pytest -q tests/test_permissions.py

# 单个测试
uv run pytest -q \
  tests/test_permissions.py::TestPermissionChecker::test_dangerous_command_denied

# 显示 print 和详细名称
uv run pytest -vv -s tests/test_commands.py

# 第一个失败后停止
uv run pytest -x -q
```

学习时要区分“原有失败”和“自己改坏”。修改前后都跑同一组测试，比较结果。

## 17.3 Mock 模型客户端

单元测试不应每次真实调用收费模型。可以实现一个假客户端：

```python
from evacode.client import LLMClient
from evacode.tools.base import StreamEnd, TextDelta


class FakeClient(LLMClient):
    async def stream(self, conversation, system="", tools=None):
        yield TextDelta(text="这是固定测试回答")
        yield StreamEnd(
            stop_reason="end_turn",
            input_tokens=10,
            output_tokens=5,
        )
```

这样测试稳定、快速、免费。

## 17.4 调试方法

### 用最小复现代替直接启动整个 UI

如果工具出错，只单独调用工具：

```python
result = await tool.execute(params)
print(result)
```

### 打印类型和内容

```python
print(type(event).__name__, repr(event))
```

### 使用日志

```python
import logging

log = logging.getLogger(__name__)
log.debug("tool=%s args=%r", tool.name, arguments)
```

运行日志一般在当前工作目录的 `.evacode/debug.log`。

### 搜索调用关系

```bash
rg -n 'class PermissionChecker|def check' evacode
rg -n 'PermissionChecker\(' evacode tests
rg -n 'ToolResultEvent' evacode tests
```

## 17.5 推荐的源码阅读顺序

1. `pyproject.toml`
2. `evacode/__main__.py`
3. `evacode/config.py`
4. `evacode/conversation.py`
5. `evacode/tools/base.py`
6. `evacode/tools/read_file.py`
7. `evacode/tools/__init__.py`
8. `evacode/client.py`
9. `evacode/agent.py` 的 `Agent.__init__` 和 `run`
10. `evacode/permissions/checker.py`
11. `evacode/app.py`
12. 再按兴趣学习 memory、skills、mcp、agents、teams

每读一个模块，回答四个问题：

1. 输入是什么？
2. 输出是什么？
3. 它保存了什么状态？
4. 谁调用它，它又调用谁？

---

# 第18章：四个循序渐进的实战项目

## 实战一：只读项目分析器

目标：理解 Client、Conversation 和流式事件。

要求：

1. 用户输入一个目录。
2. 只注册 `Glob`、`Grep`、`ReadFile`。
3. 禁止写工具和 Bash。
4. 让模型输出项目目录说明。
5. 打印每次工具调用和 token 数。

验收：程序不能修改任何文件。

## 实战二：增加 `CountLines` 工具

目标：掌握 Tool、Pydantic、Registry 和 pytest。

要求：

1. 实现本教程第 3 章工具。
2. 注册到默认 Registry。
3. 写正常文件、空文件、文件不存在、忽略空行四个测试。
4. 让真实 Agent 自己决定何时调用。

验收：测试通过，Schema 中字段描述清楚。

## 实战三：代码审查 Skill + `/review-python` 命令

目标：理解 Skill 和 Slash Command。

要求：

1. 创建 Python 审查 Skill。
2. 增加一个命令激活该 Skill。
3. 只允许读文件和运行只读检查。
4. 输出文件路径、行号、证据和建议。

验收：未要求修改时不能写文件。

## 实战四：安全的双 Agent 开发流程

目标：组合 SubAgent、权限、Worktree 和测试。

角色：

- `implementer`：在 worktree 实现小功能。
- `reviewer`：只读审查实现和测试。
- Lead：根据审查决定是否合并。

验收：

1. 两个角色职责不重叠。
2. reviewer 没有写权限。
3. 实现发生在隔离 worktree。
4. 合并前测试通过。
5. 所有任务状态和结果可追踪。

---

# 第19章：30 天学习路线

## 第 1～3 天：Python 基础

- 变量、容器、函数、类。
- dataclass、Pydantic、异常。
- pathlib、模块、类型提示。
- async/await 和异步生成器。

目标：能独立读懂 `tools/read_file.py`。

## 第 4～7 天：最小 Agent

- 配置一个模型。
- 完成流式聊天示例。
- 理解 ConversationManager。
- 实现两个只读工具。

目标：能解释“模型如何请求工具”。

## 第 8～12 天：Agent 主循环和安全

- 跟踪一次完整工具调用。
- 学习 System Prompt。
- 学习权限模式、规则和沙箱。
- 给自定义工具写测试。

目标：能在不破坏安全边界的情况下添加工具。

## 第 13～18 天：扩展系统

- MCP。
- Slash Command。
- Skill。
- Hook。
- 记忆和上下文。

目标：完成实战三。

## 第 19～24 天：工程能力

- Textual UI。
- Mock、pytest、日志。
- Git 和 worktree。
- 阅读错误恢复和上下文压缩。

目标：能定位一个跨模块 Bug。

## 第 25～30 天：多 Agent

- SubAgent。
- 任务状态和 Trace。
- Agent Teams、邮箱和共享任务。
- 完成实战四并写学习总结。

目标：能设计一个边界清楚、可验证的多 Agent 工作流。

---

# 附录 A：常用命令速查

```bash
cd ~/path/to/evacode

# 安装依赖
uv sync --frozen

# 启动 TUI
uv run evacode

# 单次调用
uv run evacode -p '解释当前项目结构'

# JSON 事件流
uv run evacode -p '读取 pyproject.toml' --output-format stream-json

# 远程 UI
uv run evacode --remote

# 测试
uv run pytest -q

# 格式化地查看某个失败
uv run pytest -vv -s tests/test_hooks.py

# 搜索定义和调用
rg -n 'class Agent' evacode
rg -n 'Agent\(' evacode tests

# 查看日志
tail -f .evacode/debug.log
```

# 附录 B：术语表

| 术语 | 小白解释 |
| --- | --- |
| LLM | 能理解和生成文本的大语言模型 |
| Agent | 让模型带着状态和工具循环工作的程序系统 |
| Prompt | 发送给模型的文字指令 |
| System Prompt | 优先级更高的角色与行为规则 |
| Token | 模型处理文本的基本计量单位 |
| Context Window | 一次请求最多能容纳的 token 范围 |
| Tool Calling | 模型输出结构化参数，请程序执行工具 |
| JSON Schema | 描述 JSON 字段和类型的规范 |
| Pydantic | Python 数据校验库 |
| Async | 等待网络时不阻塞其他工作的异步机制 |
| MCP | 连接外部工具服务的开放协议 |
| Skill | 可复用的任务流程和领域说明 |
| Hook | 生命周期特定时机自动触发的动作 |
| SubAgent | 被主 Agent 派去完成子任务的 Agent |
| Worktree | 同一 Git 仓库的额外工作目录 |
| Agent Team | 通过任务和消息长期协作的多个 Agent |
| TUI | 在终端中显示的图形化文本界面 |
| WebSocket | 浏览器与服务端双向实时通信协议 |
| Mock | 测试中代替真实外部系统的假对象 |

# 附录 C：遇到看不懂的代码怎么办

使用下面的固定流程，不要一次啃完整文件：

1. 先找到类或函数定义。
2. 只看参数、返回类型和 docstring。
3. 搜索它在哪里被调用。
4. 用最小输入单独运行。
5. 打印中间对象的类型和值。
6. 阅读对应测试，测试通常是最好的使用示例。
7. 最后再读异常处理和优化分支。

例如理解 `PermissionChecker.check()`：

```bash
cd ~/path/to/evacode
rg -n 'def check' evacode/permissions/checker.py
rg -n '\.check\(' evacode tests
sed -n '1,220p' tests/test_permissions.py
```

学完本教程后，你不必记住所有源码；真正需要掌握的是分层思想和追踪方法：

```text
输入从哪里来 → 状态存在哪里 → 谁做决策 → 谁执行动作 → 结果回到哪里
```

能沿着这条链路定位问题，就已经具备继续学习和改造 EvaCode 的核心能力。
