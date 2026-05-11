# TAFFISH MCP 指南

[English](../en/taffish-mcp.en.md) | [中文](taffish-mcp.cn.md)

`taffish-mcp` 是 TAFFISH `0.4.0` 及后续版本随安装器分发的 MCP stdio server。它让
AI 客户端可以用结构化方式检查 TAFFISH 项目、查询本地 Hub 状态、读取部分资源，并准备相对安全的项目操作。

它的设计是保守的。第一版接口主要服务于检查、规划和低风险项目维护，不暴露
`taf run`、`taf publish`、容器镜像构建或其他高影响执行路径。

## 目录

- [什么时候使用](#什么时候使用)
- [安装与验证](#安装与验证)
- [MCP 客户端配置](#mcp-客户端配置)
- [特定客户端接入](#特定客户端接入)
- [当前工具接口](#当前工具接口)
- [资源](#资源)
- [Prompts](#prompts)
- [安全模型](#安全模型)
- [典型工作流](#典型工作流)
- [故障排查](#故障排查)
- [当前边界](#当前边界)

## 什么时候使用

当 AI 客户端需要结构化访问 TAFFISH 上下文时，可以使用 `taffish-mcp`：

- 检查本地 TAFFISH 环境和配置
- 搜索本地 TAFFISH index
- 在建议安装前解析 app 元数据
- 读取当前项目的 `taffish.toml` 和 `src/main.taf`
- 准备 tool 或 flow 项目方案
- 在修改文件前调试 TAFFISH 项目

普通命令行使用仍然直接使用 `taf` 和 `taffish`。

## 安装与验证

`taffish-mcp` 会由 TAFFISH 安装器和 `taf`、`taffish` 一起安装。

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --user
```

固定版本安装：

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --version 0.4.0 --user
```

验证：

```sh
taf --version
taffish --version
taffish-mcp --version
```

如果找不到 `taffish-mcp`，先确认安装 bin 目录已经加入 `PATH`。用户级安装通常是：

```sh
export PATH="$HOME/.local/bin:$PATH"
```

## MCP 客户端配置

把 `taffish-mcp` 作为 stdio MCP server 使用。

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

不同 MCP 客户端的配置文件位置不同。server 需要运行在正常 shell 环境中，以便在需要时找到
`taf`、TAFFISH home 路径和容器 backend 工具。

修改 MCP 配置后，有些客户端需要重启或显式 reload MCP servers，新的 server 才会出现。
如果 `taffish-mcp --version` 在终端中可用，但客户端里还看不到 TAFFISH tools，
先尝试重新加载客户端侧 MCP 配置。

Codex、Claude Code、Cursor、Cline 和通用 MCP 客户端示例见
[在 AI 客户端中使用 TAFFISH MCP](mcp-clients.cn.md)。

## 特定客户端接入

本文档是 `taffish-mcp` 暴露能力的参考。具体客户端接入说明单独维护在
[在 AI 客户端中使用 TAFFISH MCP](mcp-clients.cn.md)。

客户端配置示例生成并最后检查于 2026-05-11。MCP 客户端配置格式可能变化，所以在新机器或新客户端版本上配置时，
应始终对照对应客户端的官方文档。

官方 MCP 配置参考：

| 客户端或标准 | 官方参考 |
| --- | --- |
| MCP stdio 传输 | [Model Context Protocol transport specification](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports) |
| Codex | [OpenAI Codex MCP documentation](https://developers.openai.com/codex/mcp) |
| Claude Code | [Claude Code MCP documentation](https://code.claude.com/docs/en/mcp) |
| Cursor | [Cursor MCP documentation](https://docs.cursor.com/en/context/mcp) |
| Cline | [Cline MCP server configuration](https://docs.cline.bot/mcp/adding-and-configuring-servers) 和 [Cline CLI configuration](https://docs.cline.bot/cline-cli/configuration) |

## 当前工具接口

当前 MCP tools 对应 `taf` 的安全子集，以及部分项目生命周期操作。

环境与配置：

| Tool | 作用 |
| --- | --- |
| `taffish_doctor_check` | 检查本地 TAFFISH 目录、`PATH` 和相关可执行文件，不初始化或修改系统。 |
| `taffish_config_get` | 返回当前生效的 TAFFISH 运行时配置。 |
| `taffish_config_path` | 返回 TAFFISH 配置文件路径。 |

Hub 与 index：

| Tool | 作用 |
| --- | --- |
| `taffish_update_index` | 更新本地 TAFFISH index，会向 TAFFISH home 写入 index 文件。 |
| `taffish_search_apps` | 从本地 TAFFISH index 搜索 apps。 |
| `taffish_get_app_info` | 从本地 index 解析 app、command 或 artifact，并返回版本记录。 |
| `taffish_list_apps` | 列出已安装 apps 或 index 中的 online apps。 |
| `taffish_which` | 显示已安装 app 或 command 的本地路径和元数据。 |
| `taffish_history_list` | 读取本地 TAFFISH 运行历史，不清理 history。 |

安装与卸载规划：

| Tool | 作用 |
| --- | --- |
| `taffish_install_app` | 从本地 index 安装 app 或 command。出于安全考虑默认 `dryRun=true`。 |
| `taffish_uninstall_app` | 卸载本地 app 或 command。出于安全考虑默认 `dryRun=true`。 |

项目操作：

| Tool | 作用 |
| --- | --- |
| `taffish_new_project` | 在当前目录创建新的 TAFFISH app 项目，会写入文件。 |
| `taffish_check_project` | 检查当前 TAFFISH app 项目，只读。 |
| `taffish_compile_project` | 编译当前项目并返回生成的 shell，不执行它。 |
| `taffish_build_project` | 构建当前项目的命令 wrapper。容器镜像构建刻意不暴露。 |

## 资源

`taffish-mcp` 暴露部分只读 resources：

| URI | 内容 |
| --- | --- |
| `taffish://config` | 当前生效的 TAFFISH 运行时配置。 |
| `taffish://index/summary` | 本地 TAFFISH index 摘要。 |
| `taffish://installed` | 已安装 TAFFISH apps 和 commands。 |
| `taffish://history` | 最近的本地 TAFFISH 执行历史。 |
| `taffish://project/current/taffish.toml` | 当前项目 manifest。如果工作目录位于 TAFFISH 项目内则可用。 |
| `taffish://project/current/src/main.taf` | 当前项目主 `.taf` 脚本。如果存在则可用。 |
| `taffish://docs/taf-language` | 本地 TAFFISH 语言文档，需要可用的源码 checkout。 |
| `taffish://docs/project` | 本地 TAFFISH 项目文档，需要可用的源码 checkout。 |

如果 MCP 客户端不是从 TAFFISH 源码 checkout 中启动，而又需要读取本地文档资源，可以设置
`TAFFISH_SOURCE_DIR`。

## Prompts

server 也提供一些可复用 prompts：

| Prompt | 作用 |
| --- | --- |
| `create-taffish-tool` | 引导把一个生信命令封装为 TAFFISH tool 项目。 |
| `create-taffish-flow` | 引导把已安装 TAFFISH apps 组合成 flow 项目。 |
| `debug-taffish-project` | 先使用项目检查和源文件上下文分析 TAFFISH 项目，再建议修复。 |
| `explain-taf-script` | 从 tags、参数、容器和依赖等角度解释 TAF 脚本。 |
| `prepare-taffish-publish` | 在不运行 `taf publish` 的情况下准备发布检查。 |

## 安全模型

第一版 MCP 接口刻意保持较窄。

它不暴露：

- `taf run`
- `taf publish`
- 容器镜像构建
- 直接 shell 执行
- 任意文件删除
- 任意命令执行

部分工具被显式调用时仍然会写入本地文件，例如 `taffish_update_index`、`taffish_new_project`
和 `taffish_build_project`。安装和卸载工具默认 `dryRun=true`。AI 客户端在调用会写入文件的工具前，仍应该请求用户确认。

## 典型工作流

检查本地环境：

1. 调用 `taffish_doctor_check`。
2. 读取 `taffish://config`。
3. 必要时调用 `taffish_config_path`。

搜索并解释 app：

1. 如果 index 缺失或过旧，调用 `taffish_update_index`。
2. 调用 `taffish_search_apps`。
3. 对选中的 app 调用 `taffish_get_app_info`。
4. 解释版本、依赖、来源和容器镜像元数据。

调试项目：

1. 调用 `taffish_check_project`。
2. 读取 `taffish://project/current/taffish.toml`。
3. 读取 `taffish://project/current/src/main.taf`。
4. 在编辑文件前提出最小修改建议。

## 故障排查

如果 MCP 客户端无法启动 server，先直接检查命令：

```sh
taffish-mcp --version
```

如果这个命令可用，但客户端仍然没有列出 TAFFISH tools，先重启 AI 客户端或 reload MCP servers。
有些客户端不会在 MCP 层重启前读取新的配置。

如果 `taffish://index/summary` 等资源失败，先运行：

```sh
taf update
```

如果本地文档资源不可用，可以从 TAFFISH 源码 checkout 启动客户端，或设置：

```sh
export TAFFISH_SOURCE_DIR=/path/to/taffish
```

## 当前边界

`taffish-mcp` 不是人工 review、`taf publish` 或 app 运行测试的替代品。它更适合作为结构化助手接口，用于检查、解释、规划和低风险项目维护。
