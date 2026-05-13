# 在 AI 客户端中使用 TAFFISH MCP

[English](../en/mcp-clients.en.md) | [中文](mcp-clients.cn.md)

这份文档说明如何把 `taffish-mcp` 接入常见 AI 客户端。关于 server 本身暴露的
tools、resources、prompts 和安全模型，请阅读 [TAFFISH MCP 指南](taffish-mcp.cn.md)。

本文档中的配置示例生成并最后检查于 2026-05-12。MCP 客户端配置格式可能变化。这些示例应当作为实用起点，
在新机器或新客户端版本上配置时，应以对应客户端的官方文档为准。

## 目录

- [前置条件](#前置条件)
- [官方配置参考](#官方配置参考)
- [通用 stdio 配置](#通用-stdio-配置)
- [Codex](#codex)
- [Claude Code](#claude-code)
- [Cursor](#cursor)
- [Cline](#cline)
- [验证](#验证)
- [故障排查](#故障排查)
- [安全说明](#安全说明)

## 前置条件

安装当前 TAFFISH 版本，即 `0.8.1` 或后续版本，以获得当前 MCP 工具接口：

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --user
```

确认 MCP server 可用：

```sh
taf --version
taffish --version
taffish-mcp --version
command -v taffish-mcp
```

如果 AI 客户端找不到 `taffish-mcp`，可以把客户端配置中的 `taffish-mcp`
替换为 `command -v taffish-mcp` 输出的绝对路径。

用户级安装通常位于：

```sh
$HOME/.local/bin/taffish-mcp
```

对于 MCP compile 工具，如果 Docker、Podman 或 Apptainer 的选择很重要，AI 客户端
可以在调用工具时传入 `containerBackend`。如果工具调用没有传入该参数，而 MCP
server 环境里设置了 `TAFFISH_CONTAINER_BACKEND=apptainer|podman|docker`，
TAFFISH 会使用这个环境变量。这是 TAFFISH 运行时设置，不是某个 MCP 客户端专属功能。

## 官方配置参考

以下官方文档应作为当前客户端配置语法的权威参考：

| 客户端或标准 | 官方参考 |
| --- | --- |
| MCP stdio 传输 | [Model Context Protocol transport specification](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports) |
| Codex | [OpenAI Codex MCP documentation](https://developers.openai.com/codex/mcp) |
| Claude Code | [Claude Code MCP documentation](https://code.claude.com/docs/en/mcp) |
| Cursor | [Cursor MCP documentation](https://docs.cursor.com/en/context/mcp) |
| Cline | [Cline MCP server configuration](https://docs.cline.bot/mcp/adding-and-configuring-servers) 和 [Cline CLI configuration](https://docs.cline.bot/cline-cli/configuration) |

## 通用 stdio 配置

多数 MCP 客户端支持类似下面的 JSON 结构：

```json
{
  "mcpServers": {
    "taffish": {
      "command": "taffish-mcp",
      "args": []
    }
  }
}
```

如果客户端用受限环境或非 login shell 启动 MCP server，可以改用绝对路径：

```json
{
  "mcpServers": {
    "taffish": {
      "command": "/home/user/.local/bin/taffish-mcp",
      "args": []
    }
  }
}
```

如果要让 AI 客户端处理某个 TAFFISH app 项目，建议从该项目根目录启动客户端，
或者在客户端支持时把 MCP server 的工作目录设置到项目根目录。这样
`taffish://project/current/taffish.toml` 和
`taffish://project/current/src/main.taf` 这类资源才能解析到预期项目。

修改 MCP 配置后，如果客户端提供 reload MCP servers 之类的操作，可以先重新加载；
否则建议重启 AI 客户端。有些客户端只在启动时读取 MCP server 定义。如果
`taffish-mcp` 在终端中可以正常运行，但没有立刻出现在客户端里，优先尝试重启客户端或
reload MCP，而不是先修改 TAFFISH 本身。

## Codex

官方参考：[OpenAI Codex MCP documentation](https://developers.openai.com/codex/mcp)。

通过命令行添加 server：

```sh
codex mcp add taffish -- taffish-mcp
```

然后查看已配置 server：

```sh
codex mcp list
```

Codex 也支持在配置文件中配置 MCP server。一个最小 `taffish-mcp` 配置如下：

```toml
[mcp_servers.taffish]
command = "taffish-mcp"
args = []
enabled = true
startup_timeout_sec = 20
```

如果 Codex 找不到 `taffish-mcp`，把 `command = "taffish-mcp"` 改成
`command -v taffish-mcp` 返回的绝对路径。

如果是在 Codex 已经运行时刚刚添加 server，先重启 Codex 或开启新会话，再排查 server 命令本身。

## Claude Code

官方参考：[Claude Code MCP documentation](https://code.claude.com/docs/en/mcp)。

把 `taffish-mcp` 添加为 stdio server：

```sh
claude mcp add --transport stdio taffish -- taffish-mcp
```

如果希望把配置随项目共享，可以在项目根目录使用 project scope：

```sh
claude mcp add --transport stdio --scope project taffish -- taffish-mcp
```

然后查看已配置 server：

```sh
claude mcp list
```

在 Claude Code 内部，可以使用客户端提供的 MCP 检查命令，例如 `/mcp`，
确认 TAFFISH server 是否成功连接。
如果修改配置后 server 没有立刻出现，先重启 Claude Code 或重新加载它的 MCP server 状态，
再考虑修改 `taffish-mcp` 配置。

## Cursor

官方参考：[Cursor MCP documentation](https://docs.cursor.com/en/context/mcp)。

项目级配置可以在项目根目录创建 `.cursor/mcp.json`。用户级配置使用 Cursor 官方文档中说明的全局 MCP 配置路径。

```json
{
  "mcpServers": {
    "taffish": {
      "type": "stdio",
      "command": "taffish-mcp",
      "args": []
    }
  }
}
```

修改配置后，重启 Cursor 或重新加载 MCP servers。如果 server 启动失败，把
`command` 改成 `taffish-mcp` 的绝对路径。

## Cline

官方参考：[Cline MCP server configuration](https://docs.cline.bot/mcp/adding-and-configuring-servers) 和 [Cline CLI configuration](https://docs.cline.bot/cline-cli/configuration)。

如果使用 Cline CLI，可以这样添加 server：

```sh
cline mcp add taffish -- taffish-mcp
```

如果直接配置 VS Code 扩展，可以通过 Cline 的 MCP 设置文件或界面添加一个 stdio server：

```json
{
  "mcpServers": {
    "taffish": {
      "command": "taffish-mcp",
      "args": [],
      "disabled": false
    }
  }
}
```

保存后重启或重新加载扩展中的 MCP servers。

## 验证

配置完成后，检查客户端是否能看到 TAFFISH tools、resources 和 prompts。

正常配置通常可以看到这些 tools：

- `taffish_get_version`
- `taffish_check_environment`
- `taffish_search_apps`
- `taffish_get_app_info`
- `taffish_resolve_app`
- `taffish_inspect_app`
- `taffish_summarize_app_usage`
- `taffish_compile_app_invocation`
- `taffish_validate_file`
- `taffish_compile_file`
- `taffish_check_project`
- `taffish_inspect_project`
- `taffish_summarize_project_usage`
- `taffish_compile_project`

也可能看到这些 resources：

- `taffish://config`
- `taffish://index/summary`
- `taffish://installed`
- `taffish://project/current/taffish.toml`
- `taffish://project/current/src/main.taf`
- `taffish://project/current/docs/help.md`
- `taffish://project/current/release.md`
- `taffish://project/current/summary`
- `taffish://mcp/app-inspection-model`
- `taffish://mcp/project-inspection-model`

不同客户端展示 resources 和 prompts 的方式不完全相同。如果 tools 可用，但 resources
或 prompts 不明显，先检查客户端界面和官方文档，不要直接判断 server 出错。

## 故障排查

如果客户端无法启动 server，先在客户端外部检查命令：

```sh
taffish-mcp --version
```

如果终端中可用，但客户端中不可用，通常说明客户端环境找不到命令。此时在客户端配置中使用绝对路径。

如果绝对路径正确，但客户端仍然看不到 TAFFISH tools，先重启 AI 客户端或 reload MCP servers。
刚修改配置后工具列表没有出现，很多时候是客户端尚未重新加载配置，不是 `taffish-mcp` 本身的问题。

如果 Hub 资源失败，先更新本地 TAFFISH index：

```sh
taf update
```

如果项目资源解析到了错误项目，从 app 项目根目录启动客户端，或在客户端支持时配置 MCP server 工作目录。

如果本地文档资源不可用，可以设置：

```sh
export TAFFISH_SOURCE_DIR=/path/to/taffish
```

## 安全说明

`taffish-mcp` 的设计是保守的。它不暴露 `taf run`、`taf publish`、容器镜像构建、
直接 shell 执行、任意文件删除或任意命令执行。

部分工具被显式调用时仍然会写入本地文件，例如 `taffish_update_index`、
`taffish_create_project` 和 `taffish_build_project`。安装和卸载工具默认
`dryRun=true`。会写入文件的 MCP tool 调用仍应在执行前经过用户确认。
