# PatchPilot

**在终端里运行的 Python 编程助手。** 能读写项目文件、执行命令、管理 Git 工作树，可以把任务派给子 Agent 或组成团队并行推进，并通过 MCP 接入外部工具。

- 交互式 TUI（Textual）、无头 `-p` 模式、浏览器远程模式三种形态
- 四档权限模式 + 路径沙箱 + OS 级沙箱（macOS / Linux）三重防线
- 上下文自动压缩与熔断、会话断点回溯、文件修改历史
- 子 Agent（Explore / Plan / Verification）、Lead + Teammate 团队协作
- Skill 技能包、分层记忆、生命周期 Hooks、MCP、Git Worktree

---

## 目录

- [快速开始](#快速开始)
- [命令行用法](#命令行用法)
- [配置文件](#配置文件)
- [交互式界面](#交互式界面)
- [斜杠命令](#斜杠命令)
- [内置工具](#内置工具)
- [权限与沙箱](#权限与沙箱)
- [子 Agent 与团队协作](#子-agent-与团队协作)
- [Skill 技能包](#skill-技能包)
- [记忆与上下文](#记忆与上下文)
- [Hooks](#hooks)
- [MCP 扩展](#mcp-扩展)
- [Git Worktree](#git-worktree)
- [项目结构](#项目结构)
- [开发与测试](#开发与测试)
- [常见问题](#常见问题)

---

## 快速开始

需要 **Python 3.11 或更高版本**（开发环境使用 3.13 验证）。

### 方式一：uv（推荐，仓库自带 `uv.lock`）

```powershell
uv sync
uv run patchpilot
```

### 方式二：venv + pip

在项目根目录打开 PowerShell 或 PyCharm 的 Terminal：

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -e .
```

### 写入配置

在项目根目录创建 `.patchpilot/config.yaml`，示例使用 DeepSeek 官方接口：

```yaml
providers:
  - name: deepseek-official
    protocol: openai-compat
    base_url: https://api.deepseek.com
    api_key: "替换为你的 DeepSeek API Key"
    model: deepseek-flash
    thinking: false

mcp_servers: []
```

更多字段与可选项见 [配置文件](#配置文件)，也可以直接复制 `.patchpilot/config.yaml.example` 作为起点。

### 启动

```powershell
.\.venv\Scripts\python.exe -m patchpilot     # 交互式 TUI
patchpilot -p "介绍一下这个项目的目录结构"    # 无头模式，直接打印结果
```

`.patchpilot/` 用于存放本地配置、会话、记忆与日志，已被 Git 忽略；团队协作数据落在 `~/.patchpilot/teams`。

---

## 命令行用法

| 参数 | 说明 |
| --- | --- |
| `--mode <模式>` | 覆盖配置文件里的 `permission_mode`，可选 `default` / `acceptEdits` / `plan` / `bypassPermissions` |
| `-p, --prompt <PROMPT>` | 非交互模式：执行一次提示词后把结果打到 stdout，权限请求自动放行 |
| `--output-format <格式>` | 配合 `-p` 使用，`text`（默认，只输出最终文本）或 `stream-json`（逐行输出 NDJSON 事件） |
| `--remote` | 远程模式：在 `0.0.0.0:18888` 起 WebSocket 服务，浏览器打开 <http://localhost:18888> 使用 |
| `--teammate` | 内部参数，由 tmux / iTerm2 窗格拉起队友 worker 时使用，配合 `--team-name` 与 `--agent-name` |

### 无头模式与自动化

`-p` 模式适合脚本、CI 和管道调用。加上 `--output-format stream-json` 后，每个事件输出一行 JSON，便于下游消费：

```powershell
patchpilot -p "找出所有 TODO 并汇总" --output-format stream-json
```

事件类型包括 `assistant`、`thinking`、`tool_use`、`tool_result`、`usage`、`turn_complete`、`compact`、`retry`、`error`，以及收尾的 `result`（含 `duration_ms`、`num_turns`、`tool_calls`、`usage`、`stop_reason`）。

用 `--mode` 控制无人值守时的权限行为：

```powershell
patchpilot -p "修复 lint 报错" --mode acceptEdits
```

### 远程模式

```powershell
patchpilot --remote
```

服务启动后终端会打印 `Remote UI: http://localhost:18888`，浏览器端与 TUI 共用同一套 Agent、斜杠命令与会话。

---

## 配置文件

### 加载顺序

三个候选文件按顺序加载并**逐层合并**，后者覆盖前者：

1. `~/.patchpilot/config.yaml` —— 用户全局
2. `<项目>/.patchpilot/config.yaml` —— 项目级（通常入库）
3. `<项目>/.patchpilot/config.local.yaml` —— 本地私有覆盖（不入库）

找不到任何配置文件时启动会直接报错退出。`enable_fork` 是例外：它以最后一次出现的值为准，写成 `false` 才能真正关掉。

### 字段说明

| 字段 | 类型 / 取值 | 默认 | 说明 |
| --- | --- | --- | --- |
| `providers` | 列表 | 必填 | 至少一个模型提供方 |
| `permission_mode` | `default` / `acceptEdits` / `plan` / `bypassPermissions` | `default` | 权限模式 |
| `mcp_servers` | 列表 | `[]` | MCP 服务器配置 |
| `hooks` | 列表 | `[]` | 生命周期钩子 |
| `enable_fork` | 布尔 | `true` | 是否允许子 Agent 继承当前上下文（fork） |
| `enable_verification_agent` | 布尔 | `false` | 是否注册 Verification 验证 Agent |
| `worktree` | 映射 | 见下 | Git Worktree 相关 |
| `teammate_mode` | `""` / `in-process` | `""` | 强制队友运行方式 |
| `enable_coordinator_mode` | 布尔 | `false` | 开启后 Lead 只负责调度、不再直接改代码 |
| `sandbox` | 映射 | 见下 | OS 级沙箱 |

**`providers[]`**

| 字段 | 说明 |
| --- | --- |
| `name` | 提供方名称，仅用于显示 |
| `protocol` | `anthropic` / `openai` / `openai-compat` |
| `base_url` | 接口地址 |
| `model` | 模型名 |
| `api_key` | API Key，留空则回退到环境变量：`anthropic` → `ANTHROPIC_API_KEY`，`openai` / `openai-compat` → `OPENAI_API_KEY` |
| `thinking` | 是否开启思考模式，布尔值 |
| `context_window` | 上下文窗口 token 数，`0` 表示自动推断 |
| `max_output_tokens` | 单次最大输出，`0` 表示自动（思考模式 64000，否则 8192） |

字段值支持 `${ENV_VAR}` 形式的环境变量插值，便于把密钥留在环境里。

**`mcp_servers[]`** —— `command`（stdio）与 `url`（HTTP/SSE）二选一，不能同时出现：

```yaml
mcp_servers:
  - name: context7
    command: npx
    args: ["-y", "@upstash/context7-mcp"]

  - name: my-http-mcp
    url: https://example.com/mcp
    transport: http        # http（默认）或 sse
    headers:
      Authorization: "Bearer ${MY_TOKEN}"
```

**`worktree`** —— `symlink_directories`（默认 `["node_modules", ".venv", "vendor"]`）、`stale_cleanup_interval`（默认 3600 秒）、`stale_cutoff_hours`（默认 24 小时）。

**`sandbox`** —— `enabled`、`auto_allow`、`network_enabled`，默认全为 `false`。

### 上下文窗口推断

`context_window` 为 `0` 时按四层回退链解析，优先级从高到低：

1. 配置里显式写明的 `context_window`
2. 从提供方 `/v1/models` 端点自动拉取并缓存的值
3. 内置「模型名子串 → 窗口」映射表（`claude`、`gpt-4o`、`gpt-4.1`、`o1`/`o3`/`o4`、`1m` 后缀等）
4. 保守默认值：模型名含 `claude` 取 200000，其余取 128000

映射表会随模型更新而过时，遇到不准的情况直接写 `context_window` 覆盖即可。

---

## 交互式界面

输入框内：

| 快捷键 | 作用 |
| --- | --- |
| `Enter` | 提交 |
| `Shift+Enter` / `Ctrl+J` | 换行 |
| `Tab` | 补全（命令与参数） |
| `Esc` | 关闭弹窗 |
| `↑` / `↓` | 弹窗内导航 |

全局：

| 快捷键 | 作用 |
| --- | --- |
| `Ctrl+C` | 退出（空闲时安全退出） |
| `Shift+Tab` | 循环切换权限模式 |
| `Ctrl+O` | 折叠 / 展开工具调用块 |

界面采用非 alt-screen 驱动，退出后聊天记录仍保留在终端回滚缓冲里。

---

## 斜杠命令

| 命令 | 别名 | 说明 |
| --- | --- | --- |
| `/help` | `h` `?` | 显示帮助信息，可带命令名查看单条 |
| `/status` | `s` | 显示状态信息（模型、token、模式等） |
| `/compact` | `c` | 压缩上下文，可附加要保留的重点 |
| `/clear` | | 清除对话历史 |
| `/plan` | `p` | 切换到 Plan 模式，输出只读方案 |
| `/review` | | 审查代码变更，可附加关注点 |
| `/rewind` | | 回退到历史检查点 |
| `/session` | | 会话管理：`list` / `resume <id>` / `new` / `delete <id>` |
| `/memory` | | 记忆管理：`list` / `clear` / `edit` |
| `/permission` | | 权限管理：`mode <模式>` / `rules` / `add <规则>` / `reset` |
| `/sandbox` | | 沙箱管理：`1\|on-auto` / `2\|on` / `3\|off` |
| `/mcp` | | 查看 MCP 服务器状态 |
| `/skill` | `skills` | Skill 管理：`list` / `info <name>` / `reload` |
| `/tasks` | `task` | 后台任务：`info <id>` / `cancel <id>` |
| `/trace` | `tree` | 查看 Agent 父子追踪树 |
| `/worktree` | `wt` | Git Worktree：`create` / `list` / `enter` / `exit` / `status` |

Skill 目录里定义的技能会按名字自动注册成同名斜杠命令，直接 `/技能名 参数` 调用。

---

## 内置工具

默认注册表包含文件与命令基础工具，其余工具按场景追加：

| 工具 | 作用 |
| --- | --- |
| `ReadFile` | 读取文件内容 |
| `WriteFile` | 写入文件 |
| `EditFile` | 精确替换文件片段 |
| `Bash` | 执行终端命令（可被 OS 沙箱包装） |
| `Glob` | 按路径模式查找文件 |
| `Grep` | 正则检索文件内容 |
| `ToolSearch` | 按需检索被延迟加载的工具定义 |
| `Agent` | 派发子 Agent 执行子任务 |
| `TeamCreate` / `TeamDelete` | 组建 / 解散团队 |
| `SendMessage` | 团队成员之间投递消息 |
| `TaskCreate` / `TaskGet` / `TaskList` / `TaskUpdate` / `TaskStop` | 共享任务板的增删查改与停止 |
| `EnterWorktree` / `ExitWorktree` | 进入 / 退出 Git Worktree |
| `LoadSkill` / `InstallSkill` | 加载 / 安装 Skill |
| `AskUserQuestion` | 向用户提问并等待选择 |
| `ExitPlanMode` | 结束方案模式、转入执行 |
| `SyntheticOutput` | 生成合成输出 |

`ToolSearch` 让二次扩展的工具（MCP、团队工具等）延迟注入 schema，主循环只在真正需要时才展开，避免一次性塞满工具定义。

---

## 权限与沙箱

### 四档权限模式

读操作在所有模式下都直接放行，写操作与命令按模式决策：

| 模式 | 读 | 写 | 命令 |
| --- | --- | --- | --- |
| `default` | 允许 | 询问 | 询问 |
| `acceptEdits` | 允许 | 允许 | 询问 |
| `plan` | 允许 | 询问 | 询问 |
| `bypassPermissions` | 允许 | 允许 | 允许 |

`--mode` 与 `/permission mode` 都可以切换，`Shift+Tab` 在界面里循环切换。

### 规则文件

权限规则按三层加载，后者覆盖前者：

1. `~/.patchpilot/permissions.yaml` —— 用户全局
2. `<项目>/.patchpilot/permissions.yaml` —— 项目级
3. `<项目>/.patchpilot/permissions.local.yaml` —— 本地私有

危险命令检测器独立于规则引擎，对 `rm -rf /`、`mkfs.*` 格式化、`dd` 直接写磁盘、fork bomb、`curl … | sh` 管道执行远程脚本一类的高危命令单独拦截；只读命令（`ls`、`git status`、`grep` 等）则直接放行。

### OS 级沙箱

`PathSandbox` 是路径级检查，`sandbox/` 下是操作系统层面的强制隔离——即使命令绕过了路径校验，内核也会拦下越权写入：

- **macOS**：`sandbox-exec`（Seatbelt）
- **Linux**：bubblewrap（`bwrap`）
- **Windows 及其他平台**：不支持，自动退回路径检查

启用后只允许写入工作目录与 `/tmp`，并强制把 `.patchpilot/config.yaml`、`.patchpilot/permissions.local.yaml` 设为只读，网络默认关闭。`/sandbox` 可现场开关：`on-auto` 启用沙箱并自动放行命令，`on` 只启用沙箱、仍走常规权限，`off` 关闭。

---

## 子 Agent 与团队协作

### 内置子 Agent

定义在 `patchpilot/agents/builtins/`，用带 frontmatter 的 Markdown 描述，支持 `name`、`description`、`disallowedTools`、`model`、`maxTurns`、`background` 等字段：

| Agent | 定位 |
| --- | --- |
| `Explore` | 只读的快速检索 Agent，用于定位代码，禁用写入类工具 |
| `Plan` | 软件架构师，分析需求产出实现方案，不直接改代码，轮次有上限 |
| `general-purpose` | 通用子 Agent，工具齐全，适合需要完整能力但独立上下文的场景 |
| `Verification` | 验证专家，尝试击穿实现找出隐藏 bug，输出 `VERDICT` 判定 |

自定义 Agent 放在 `.patchpilot/agents/`（项目级）或 `~/.patchpilot/agents/`（用户级），同名时项目级优先。

### 团队协作

Lead 通过 `TeamCreate` 组建团队，成员在独立上下文里运行，通过 `SendMessage` 通信、通过共享任务板（`TaskCreate` / `TaskUpdate` / `TaskList`）协作：

- **后端选择**：检测到 `TMUX` 环境变量时用 tmux 窗格，检测到 `ITERM_SESSION_ID` 时用 iTerm2 窗格，否则进程内运行；Windows 一律进程内（跨平台的 `pwsh` 执行 POSIX 命令会失败）
- **`teammate_mode: in-process`** 可强制进程内运行，`-p` 非交互模式也自动走进程内
- **`enable_coordinator_mode: true`** 开启协调者模式：Lead 从第一轮起就被收窄工具集，只能调度、不能写代码
- `/trace` 查看父子 Agent 的追踪树，`/tasks` 查看与取消后台任务

---

## Skill 技能包

Skill 是一份带 frontmatter 的 `SKILL.md`，可以打包说明、脚本与参考文件，把一个领域的做法固化下来复用。三层加载，同名时项目级优先：

1. `<项目>/.patchpilot/skills/`
2. `~/.patchpilot/skills/`
3. 内置技能

```text
.patchpilot/skills/
└── my-skill/
    ├── SKILL.md          # frontmatter + 指令正文
    └── references/       # 可选：脚本与参考资料
```

管理命令：`/skill list`、`/skill info <name>`、`/skill reload`。安装新技能用 `InstallSkill` 工具，加载用 `LoadSkill`；声明为 fork 模式的技能在无执行器环境里会自动退回 inline 执行。

---

## 记忆与上下文

### 指令分层

启动时按以下顺序收集指令内容并注入系统提示：

1. 用户全局：`~/.patchpilot/PATCHPILOT.md`、`~/.patchpilot/AGENTS.md`
2. 从 git 根目录逐级到工作目录，每层的 `PATCHPILOT.md` 与 `AGENTS.md`
3. `<工作目录>/.patchpilot/INSTRUCTIONS.md`（遗留格式）

指令文件里可以用 `@./path`、`@../path`、`@~/path`、`@/path` 引用其他文件，最多递归 5 层，自动检测循环引用，代码块内的 `@` 不会被当作引用。

### 长期记忆

记忆存放在 `.patchpilot/memory/`，以 `MEMORY.md` 为入口，会话期间自动抽取并按锁文件串行固化，避免并发写坏。

### 上下文压缩

上下文接近窗口上限时自动触发摘要压缩，分软、硬两级触发线，配熔断器防止反复压缩：

- 保留尾部约 10000 token（最少 5 条消息，最多 40000 token）原文，只摘要前缀
- 前缀不足 2000 token 时放弃压缩——压缩开销大于收益
- 单条工具结果超限时落盘并以占位符回填，需要时再按路径重读
- 单条消息内多个工具结果聚合上限 200000 字符

`/compact` 可手动触发，并额外指定要保留的重点。

### 会话与回退

会话保存在 `.patchpilot/sessions/`，支持 `/session list`、`/session resume <id>` 恢复；文件修改历史落在 `.patchpilot/file-history/`，配合 `/rewind` 回到任意检查点。运行日志默认写 `.patchpilot/debug.log`。

---

## Hooks

在配置文件的 `hooks` 段挂载生命周期钩子，命令执行、文件变更、权限请求都可以在前后插入自定义逻辑：

| 层级 | 事件 |
| --- | --- |
| 会话 | `session_start`、`session_end` |
| 轮次 | `turn_start`、`turn_end` |
| 工具 | `pre_tool_use`、`post_tool_use` |
| 消息 | `pre_send`、`post_receive` |
| 系统 | `startup`、`shutdown`、`error`、`compact`、`permission_request`、`file_change`、`command_execute` |

钩子可以拦截工具调用（`pre_tool_use` 拒绝后工具不会执行），也可以把提示信息回注到对话里。

---

## MCP 扩展

通过 MCP 把外部工具接进 Agent，stdio 与 HTTP/SSE 两种传输都支持：

```yaml
mcp_servers:
  - name: filesystem
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "D:/work"]

  - name: remote-tools
    url: https://example.com/mcp
    headers:
      Authorization: "Bearer ${MY_TOKEN}"
```

`command` 与 `url` 只能二选一，同时配置或同时缺失都会在配置校验阶段报错。启动时逐个连接并注册工具，单个服务器失败不会拖垮主进程，只用 `/mcp` 查看状态即可。

---

## Git Worktree

用 `/worktree`（别名 `/wt`）在独立的 Git Worktree 里隔离改动，避免污染当前分支：

| 子命令 | 作用 |
| --- | --- |
| `create` | 基于当前仓库创建新 Worktree |
| `list` | 列出已有 Worktree |
| `enter` | 进入指定 Worktree 作为工作目录 |
| `exit` | 退出并回到主工作区 |
| `status` | 查看当前 Worktree 状态与改动 |

创建时会自动把 `node_modules`、`.venv`、`vendor` 这类体积大又无需复制的目录软链过去（可用 `worktree.symlink_directories` 调整）。超过 `stale_cutoff_hours` 的陈旧 Worktree 会按 `stale_cleanup_interval` 周期清理。

---

## 项目结构

```text
PatchPilot/
├── patchpilot/
│   ├── __main__.py        # 入口：CLI 解析 + TUI / 无头 / 远程 / 队友四条启动路径
│   ├── app.py             # Textual TUI 主应用、快捷键、状态栏
│   ├── agent.py           # Agent 主循环、事件流、工具预算与恢复
│   ├── client.py          # 多协议 LLM 客户端、context window 解析
│   ├── config.py          # 三层配置加载与合并
│   ├── validator.py       # 配置校验与模型窗口映射表
│   ├── conversation.py    # 消息模型、token 估算
│   ├── context/           # 上下文压缩、熔断器、断点恢复
│   ├── memory/            # 指令分层、长期记忆、自动固化
│   ├── permissions/       # 权限决策、规则引擎、路径沙箱、危险命令检测
│   ├── sandbox/           # OS 级沙箱（macOS seatbelt / Linux bwrap）
│   ├── tools/             # 内置工具实现
│   ├── agents/            # 子 Agent 加载、任务管理、追踪树
│   │   └── builtins/      # Explore / Plan / general-purpose / Verification
│   ├── teams/             # Lead + Teammate 团队协作
│   ├── skills/            # Skill 加载、解析、执行、安装
│   ├── commands/          # 斜杠命令注册表
│   │   └── handlers/      # 各命令处理器
│   ├── hooks/             # 生命周期钩子引擎
│   ├── mcp/               # MCP 客户端与工具适配
│   ├── worktree/          # Git Worktree 管理
│   ├── filehistory/       # 文件修改历史与回滚
│   ├── remote.py          # WebSocket + 浏览器 Web UI
│   ├── prompts.py         # 系统提示与环境上下文拼装
│   ├── driver.py          # 非 alt-screen 终端驱动
│   └── styles.tcss        # TUI 样式
├── tests/                 # pytest 测试集
├── .patchpilot/
│   └── config.yaml.example
├── pyproject.toml
└── uv.lock
```

规模参考：`patchpilot/` 与 `tests/` 合计约 167 个 Python 文件、2.5 万行代码，其中测试约 7200 行。

---

## 开发与测试

```powershell
uv sync                                              # 安装含 dev 依赖
uv run pytest                                        # 跑全部测试
uv run pytest tests/test_permissions.py -q           # 跑单个文件
```

或者用 venv（dev 依赖在 `[dependency-groups]` 里，需要单独装）：

```powershell
.\.venv\Scripts\python.exe -m pip install -e .
.\.venv\Scripts\python.exe -m pip install pytest pytest-asyncio
.\.venv\Scripts\python.exe -m pytest
```

依赖清单：`textual`（TUI）、`anthropic` / `openai`（模型协议）、`pyyaml`、`pydantic`、`mcp`、`httpx`、`websockets`；开发期另需 `pytest` 与 `pytest-asyncio`。

测试已覆盖 Agent 主循环、上下文压缩、权限与沙箱、MCP、记忆、技能、Worktree、团队协议、工具检索与子 Agent 派发等模块。团队数据目录在测试里被重定向到临时目录，不会污染真实主目录。

---

## 常见问题

**启动报 `No config file found`**
仓库里只有 `.patchpilot/config.yaml.example`，需要自己复制成 `.patchpilot/config.yaml` 并填好 `api_key`。

**改了配置没生效**
确认覆盖顺序：`config.local.yaml` > 项目 `config.yaml` > 用户全局 `config.yaml`。另外 `enable_fork` 比较特殊，以最后出现的值为准。

**上下文窗口识别不对**
自动推断有可能会落后于模型更新，直接在 provider 里写 `context_window` 覆盖。

**`/sandbox` 打开提示不支持**
OS 级沙箱只在 macOS 和 Linux 可用，Windows 上会退回路径级权限检查，这是预期行为。

**队友没有出现在独立窗格**
只有身处 tmux 或 iTerm2 会话里才会开新窗格，Windows 一律进程内运行。想要固定行为，就显式设置 `teammate_mode: in-process`。

**`-p` 模式下权限怎么处理**
无人值守，权限请求会全部自动放行。用 `--mode` 限制它能做什么。

---

## 说明

本仓库当前未附带 `LICENSE` 文件。若计划对外分发或在其中引入第三方代码，建议先补上明确的许可证声明。
