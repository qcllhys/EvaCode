# Contributing to EvaCode

感谢关注 EvaCode。项目优先保证模块边界、可测试性和安全行为清晰。

## Development setup

```bash
uv sync --frozen
uv run pytest -q
```

项目要求 Python 3.11 或更高版本。默认测试必须使用 Mock Client，不应依赖外部模型、
API Key 或付费服务。

## Architecture boundaries

- UI 只消费 Agent 事件，不直接调用模型 SDK 或执行工具。
- 模型协议差异集中在 `evacode/client.py` 和序列化层。
- 工具通过 `Tool` 抽象注册，并声明准确的 `category`。
- 写入和命令能力必须经过权限系统，不能只依赖 Prompt 约束。
- 上下文压缩不得破坏 tool call 与 tool result 的配对关系。
- Agent Teams 的进度、邮箱和共享任务状态应保持可恢复。

## Change requirements

- 修复 Bug 时优先增加能够复现问题的回归测试。
- 新工具必须使用 Pydantic 参数模型，并覆盖参数错误和权限行为。
- 改变配置、命令或用户行为时，同步更新 README 或 `docs/`。
- 不要提交虚拟环境、缓存、会话、记忆、日志、构建产物或本地配置。
- 不要在源码、测试和文档中加入真实服务地址、密钥、账号或本地绝对路径。

## Validation

```bash
# 最小相关测试
uv run pytest -q tests/test_permissions.py

# 全量回归
uv run pytest -q

# 构建发布包
uv build
```

提交前请确认：

1. 改动范围单一，没有混入运行时文件；
2. 最小相关测试和全量测试通过；
3. 行为变化和安全影响已经记录；
4. 文档中的命令可以从干净环境执行；
5. 仓库中不存在凭据、内网地址和个人绝对路径。
