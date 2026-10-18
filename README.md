# PatchPilot

PatchPilot 是一个在终端中运行的 Python 编程助手。它可以读取和修改项目文件、执行命令，并通过 MCP 连接外部工具。

## 本地启动

需要 Python 3.11 或更高版本。在 Windows 上，从项目根目录打开 PowerShell 或 PyCharm 的 Terminal，运行：

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -e .
```

在本地创建 `.patchpilot/config.yaml`。以下示例使用 DeepSeek 官方接口：

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

填好配置后运行：

```powershell
.\.venv\Scripts\python.exe -m patchpilot
```

`.patchpilot/` 用于存放本地配置、会话和日志，已被 Git 忽略。

## 源码来源

本项目基于小林 coding 的 MewCode 课程源码整理。上游署名见源文件头部。
