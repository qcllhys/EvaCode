# EvaCode Linux 迁移与部署教程

本文适用于将 EvaCode 从当前服务器迁移到另一台 Linux 服务器。推荐迁移源码和
`uv.lock`，然后在目标服务器重新创建虚拟环境。不要直接复制现有 `.venv`，因为其中
包含原服务器的绝对路径，并且可能与目标服务器的 Linux 发行版、CPU 架构或 Python
版本不兼容。

## 1. 应该保留哪些内容

### 必须保留

| 路径 | 用途 |
| --- | --- |
| `evacode/` | EvaCode 的 Python 源码 |
| `pyproject.toml` | 项目名称、依赖和 `evacode` 命令入口 |
| `uv.lock` | 锁定依赖版本，保证新服务器尽量复现当前环境 |
| `EVACODE.md` | EvaCode 在本项目中的指令文档 |

### 建议保留

| 路径 | 用途 |
| --- | --- |
| `.evacode/config.yaml.example` | 配置模板，不包含真实密钥 |
| `.evacode/skills/` | 当前项目自带的 Skills |
| `.evacode/agents/`、`.evacode/commands/` | 如果以后添加了自定义 Agent 或命令，需要一起迁移 |
| `tests/` | 部署后验收和后续开发使用 |
| `.gitignore` | 防止密钥、缓存和虚拟环境被提交 |
| `dist/evacode-*.whl` | 可选的安装包，可用于快速安装或版本回滚 |

如果需要保留历史会话、记忆或权限规则，还应有选择地备份：

- `~/.evacode/memory/`：用户级记忆。
- `~/.evacode/permissions.yaml`：用户级权限规则。
- 工作目录中的 `.evacode/memory/`、`.evacode/permissions*.yaml`：项目级数据。
- `.evacode/sessions/`、`.evacode/file-history/`：仅在确实需要历史记录时迁移。

### 不建议迁移

以下内容都是缓存、日志或与当前服务器绑定的环境，可以在目标服务器重新生成：

- `.venv/`
- `.pytest_cache/`
- 所有 `__pycache__/` 和 `*.pyc`
- `.evacode/debug.log`
- 临时的 `.evacode/session/`、`.evacode/plans/` 和 `.evacode/worktrees/`
- `dist/` 和 `build/`，如果目标服务器会从源码重新构建

不要把包含真实 API Key 的 `.evacode/config.yaml` 放进普通压缩包或代码仓库。推荐在
目标服务器重新创建配置，并通过环境变量提供密钥。

## 2. 在原服务器制作迁移包

在当前服务器执行：

```bash
cd ~/path/to

tar \
  --exclude='evacode/.venv' \
  --exclude='evacode/dist' \
  --exclude='evacode/build' \
  --exclude='evacode/.pytest_cache' \
  --exclude='*/__pycache__' \
  --exclude='*.pyc' \
  --exclude='evacode/.evacode/debug.log' \
  --exclude='evacode/.evacode/session' \
  --exclude='evacode/.evacode/sessions' \
  --exclude='evacode/.evacode/plans' \
  --exclude='evacode/.evacode/worktrees' \
  --exclude='evacode/.evacode/config.yaml' \
  -czf evacode-source-0.2.0.tar.gz evacode
```

把压缩包传到新服务器，例如：

```bash
scp evacode-source-0.2.0.tar.gz user@new-server:~/
```

如果必须保留记忆或会话，单独制作一个受保护的备份包，不要与公开源码包混在一起。

## 3. 目标服务器要求

- 64 位 Linux，常见的 Ubuntu、Debian、Rocky Linux 或 CentOS 均可。
- 能访问所配置的大模型 API。
- Python 3.11 或更高版本。推荐让 `uv` 自动管理 Python。
- 如果启用了基于 `npx` 的 MCP Server，还需要安装 Node.js 和 npm；未使用 MCP 时不需要。
- 交互式界面需要正常的 UTF-8 终端。

先安装常用系统工具。Ubuntu 或 Debian 示例：

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl git tar
```

Rocky Linux、AlmaLinux 或 CentOS 示例：

```bash
sudo dnf install -y ca-certificates curl git tar
```

## 4. 推荐方式：用源码和 uv 部署

以下示例将程序安装到当前用户的 `~/apps/evacode`，不需要使用 root 运行 EvaCode。

### 4.1 解压源码

```bash
mkdir -p ~/apps
tar -xzf ~/evacode-source-0.2.0.tar.gz -C ~/apps
cd ~/apps/evacode
```

### 4.2 安装 uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

重新打开终端，或者让当前终端加载默认安装路径：

```bash
export PATH="$HOME/.local/bin:$PATH"
uv --version
```

### 4.3 创建运行环境

```bash
cd ~/apps/evacode
uv sync --frozen --no-dev
```

这条命令会按照 `uv.lock` 创建新的 `.venv` 并安装运行依赖。不要从原服务器复制
`.venv`。

检查命令是否安装成功：

```bash
~/apps/evacode/.venv/bin/evacode --help
```

需要运行测试时，再安装开发依赖：

```bash
cd ~/apps/evacode
uv sync --frozen
uv run pytest -q
```

完整测试应正常通过；需要真实模型凭据的端到端测试会在未配置环境变量时自动跳过。

## 5. 配置模型服务

EvaCode 会按以下顺序加载并合并配置，后面的配置优先级更高：

1. `~/.evacode/config.yaml`
2. 当前工作目录的 `.evacode/config.yaml`
3. 当前工作目录的 `.evacode/config.local.yaml`

推荐把通用模型配置放在用户目录：

```bash
mkdir -p ~/.evacode
chmod 700 ~/.evacode
cp ~/apps/evacode/.evacode/config.yaml.example ~/.evacode/config.yaml
chmod 600 ~/.evacode/config.yaml
```

### 5.1 OpenAI 兼容服务示例

编辑 `~/.evacode/config.yaml`：

```yaml
providers:
  - name: company-model
    protocol: openai-compat
    base_url: http://MODEL_SERVER:PORT/v1
    model: MODEL_NAME
    context_window: 128000
    max_output_tokens: 8192

permission_mode: default
mcp_servers: []
enable_coordinator_mode: false
```

通过环境变量设置 Key：

```bash
export OPENAI_API_KEY='replace-with-real-key'
```

某些本地 vLLM 或兼容服务不校验 Key，但 EvaCode 客户端仍要求它非空。这种情况下可用：

```bash
export OPENAI_API_KEY='local-not-used'
```

### 5.2 Anthropic 服务示例

```yaml
providers:
  - name: anthropic-official
    protocol: anthropic
    base_url: https://api.anthropic.com
    model: claude-sonnet-4-20250514
    thinking: true

permission_mode: default
mcp_servers: []
```

```bash
export ANTHROPIC_API_KEY='replace-with-real-key'
```

生产环境不要把 Key 直接写入代码、部署文档、Shell 历史或 Git 仓库。

## 6. Skills 和工作目录

EvaCode 把启动命令所在的当前目录当作工作目录，并在这个范围内读取项目配置、Skills、
记忆和指令。因此，不要为了处理别的项目而一直在 `~/apps/evacode` 源码目录中启动。

例如，要让 EvaCode 操作 `~/workspaces/my-project`：

```bash
cd ~/workspaces/my-project
~/apps/evacode/.venv/bin/evacode
```

当前源码附带的 Skills 位于 `~/apps/evacode/.evacode/skills/`。如果希望在所有工作目录
中使用它们，可以复制到用户级目录：

```bash
mkdir -p ~/.evacode/skills
cp -a ~/apps/evacode/.evacode/skills/. ~/.evacode/skills/
```

也可以只把需要的 Skill 放入具体项目的 `.evacode/skills/`。

## 7. 启动方式

### 7.1 交互式终端界面

```bash
cd ~/workspaces/my-project
~/apps/evacode/.venv/bin/evacode
```

首次部署建议保持 `permission_mode: default`。不要在不受信任的工作目录或生产服务器上
直接使用 `bypassPermissions`。

### 7.2 单次非交互调用

```bash
cd ~/workspaces/my-project
~/apps/evacode/.venv/bin/evacode -p '只读取项目并概括目录结构'
```

需要 NDJSON 事件流时：

```bash
~/apps/evacode/.venv/bin/evacode \
  -p '检查当前项目' \
  --output-format stream-json
```

### 7.3 浏览器远程模式

```bash
cd ~/workspaces/my-project
~/apps/evacode/.venv/bin/evacode --remote
```

服务固定监听 `0.0.0.0:18888`，浏览器访问：

```text
http://SERVER_IP:18888/
```

重要：当前远程模式没有内置登录鉴权，并且 Agent 能读写工作目录、调用命令。不要把
18888 端口直接暴露到公网。至少应通过服务器防火墙限制来源 IP；更安全的方式是禁止
公网访问 18888，然后使用 SSH 隧道：

```bash
ssh -L 18888:127.0.0.1:18888 user@SERVER_IP
```

随后在本机浏览器访问 `http://127.0.0.1:18888/`。

## 8. 用 systemd 在后台运行远程模式

以下使用当前 Linux 用户的 systemd 服务，不以 root 身份运行。

### 8.1 创建环境变量文件

```bash
mkdir -p ~/.config/evacode
cat > ~/.config/evacode/env <<'EOF'
OPENAI_API_KEY=replace-with-real-key
EOF
chmod 600 ~/.config/evacode/env
```

使用 Anthropic 时把变量名改为 `ANTHROPIC_API_KEY`。

### 8.2 创建用户服务

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/evacode.service <<'EOF'
[Unit]
Description=EvaCode Remote UI
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=%h/workspaces/my-project
EnvironmentFile=%h/.config/evacode/env
ExecStart=%h/apps/evacode/.venv/bin/evacode --remote
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
EOF
```

把 `WorkingDirectory` 改成 EvaCode 实际需要操作的项目目录，然后启动：

```bash
systemctl --user daemon-reload
systemctl --user enable --now evacode
systemctl --user status evacode
```

查看日志：

```bash
journalctl --user -u evacode -f
```

如需用户退出登录后继续运行，可由管理员执行：

```bash
sudo loginctl enable-linger "$(id -un)"
```

停止服务：

```bash
systemctl --user disable --now evacode
```

## 9. 只用 wheel 安装

如果只需要运行而不准备修改源码，可以传输 `dist` 中的 wheel：

```bash
python3.11 -m venv ~/apps/evacode-runtime
~/apps/evacode-runtime/bin/pip install \
  ~/evacode-0.2.0-py3-none-any.whl
~/apps/evacode-runtime/bin/evacode --help
```

这种方式会从 Python 软件源解析并下载依赖，不会严格使用 `uv.lock` 中的版本，因此生产
部署更推荐前面的源码加 `uv sync --frozen` 方案。wheel 也不会自动携带项目根目录中的
`.evacode/skills/` 和 `EVACODE.md`，需要时应单独迁移。

## 10. 离线服务器部署

wheel 文件只包含 EvaCode 本身，不包含 `textual`、`openai`、`anthropic` 等第三方依赖。
完全离线部署时，需要在一台与目标服务器具有相同 CPU 架构、Linux 平台和 Python 版本
的联网机器上提前下载依赖 wheel。

联网机器示例：

```bash
cd ~/apps/evacode
uv export \
  --frozen \
  --no-dev \
  --no-emit-project \
  --format requirements-txt \
  --output-file requirements-linux.txt

python3.11 -m pip download \
  -r requirements-linux.txt \
  -d wheelhouse
```

把以下内容一起复制到离线服务器：

- `evacode-0.2.0-py3-none-any.whl`
- `requirements-linux.txt`
- `wheelhouse/`

离线服务器安装：

```bash
python3.11 -m venv ~/apps/evacode-runtime
~/apps/evacode-runtime/bin/pip install \
  --no-index \
  --find-links=wheelhouse \
  -r requirements-linux.txt

~/apps/evacode-runtime/bin/pip install \
  --no-index \
  --no-deps \
  evacode-0.2.0-py3-none-any.whl
```

## 11. 部署验收

依次检查：

```bash
# 1. Python 和命令入口
~/apps/evacode/.venv/bin/python --version
~/apps/evacode/.venv/bin/evacode --help

# 2. 配置文件权限
ls -l ~/.evacode/config.yaml

# 3. 模型调用；此操作会真实调用配置的模型服务
cd ~/workspaces/my-project
~/apps/evacode/.venv/bin/evacode -p '只回复：EvaCode 部署成功'

# 4. 后台服务（如果启用）
systemctl --user status evacode
ss -lntp | grep 18888
```

如果命令在项目根目录执行，运行日志通常位于当前工作目录的 `.evacode/debug.log`。

## 12. 常见问题

### 提示找不到配置文件

确认存在 `~/.evacode/config.yaml` 或当前工作目录下的 `.evacode/config.yaml`。

### 提示缺少 API Key

- `anthropic` 协议读取 `ANTHROPIC_API_KEY`。
- `openai` 和 `openai-compat` 协议读取 `OPENAI_API_KEY`。
- systemd 服务不会自动读取交互式 Shell 的 `.bashrc`，需要使用 `EnvironmentFile`。

### MCP 启动失败或找不到 npx

如果不使用 MCP，把配置改为 `mcp_servers: []`。如果需要 `npx` MCP，则安装 Node.js，
并确保 systemd 服务的 `PATH` 能找到 `node` 和 `npx`。

### 换服务器后 `.venv/bin/evacode` 无法运行

删除目标服务器上的 `.venv`，然后重新执行：

```bash
cd ~/apps/evacode
uv sync --frozen --no-dev
```

### 远程页面无法访问

检查 EvaCode 进程、18888 监听状态和服务器防火墙。不要为了方便直接将无鉴权的 18888
端口开放给整个公网。

## 13. 后续升级与回滚

升级前备份配置、Skills、记忆和需要保留的会话。替换源码后重新同步锁定依赖：

```bash
cd ~/apps/evacode
uv sync --frozen --no-dev
systemctl --user restart evacode
```

建议为每次可用版本保留一份源码压缩包或 wheel，并记录对应的 `uv.lock`。回滚时应同时
恢复源码和同版本的锁文件，再重新创建或同步 `.venv`。
