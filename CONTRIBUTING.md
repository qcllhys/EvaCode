# Contributing to EvaCode

感谢参与 EvaCode。这个项目以可读性、可测试性和安全边界清晰为优先目标。

## 开发环境

```bash
git clone <your-repository-url>
cd eva
uv sync --frozen
uv run pytest -q
```

项目要求 Python 3.11 或更高版本。

## 修改原则

- 保持模块职责单一，避免把模型、UI、工具和权限逻辑耦合在一起。
- 新工具必须声明正确的 `category`，并使用 Pydantic 校验参数。
- 写入或命令能力必须经过权限系统，不能只依赖 Prompt 约束。
- 不要在源码、测试、文档或日志中加入真实服务地址、API Key、账号或本地绝对路径。
- 修改行为时添加最小相关测试；修复 Bug 时优先添加回归测试。
- 不要提交虚拟环境、缓存、会话、记忆、调试日志和构建产物。

## 测试

```bash
# 全量测试
uv run pytest -q

# 指定测试文件
uv run pytest -q tests/test_permissions.py

# 首次失败后停止并显示详细信息
uv run pytest -x -vv
```

涉及真实模型的端到端测试必须通过环境变量显式启用，默认测试不应依赖外部付费服务。

## 提交 Pull Request 前

1. 确认改动范围清楚，没有混入缓存或本地配置。
2. 运行最小相关测试和全量测试。
3. 搜索可能的密钥、内网地址和绝对路径。
4. 在 PR 中说明行为变化、测试结果和安全影响。
5. 如果改变配置或用户操作方式，同步更新 README 或 `docs/`。

一个简单的发布前扫描示例：

```bash
rg -n --hidden '/home/|/root/|192\.168\.|172\.(1[6-9]|2[0-9]|3[01])\.' \
  --glob '!.git/**' --glob '!uv.lock'
```

测试中的 `/home/user/project`、`/etc/passwd`、`localhost` 等固定安全测试样例可以保留，
但必须确保它们不是真实环境数据。
