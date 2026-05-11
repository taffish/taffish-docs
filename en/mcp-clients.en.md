# Using TAFFISH MCP With AI Clients

[English](mcp-clients.en.md) | [中文](../zh/mcp-clients.cn.md)

This guide shows how to connect `taffish-mcp` to common AI clients. For the
server capability reference, tool list, resources, prompts, and safety model,
read the [TAFFISH MCP Guide](taffish-mcp.en.md).

The configuration examples in this document were generated and last checked on 2026-05-11.
MCP client configuration formats can change. Treat these snippets as practical
starting points, and check the official client documentation when configuring a
new machine or a new client version.

## Table Of Contents

- [Prerequisites](#prerequisites)
- [Official Configuration References](#official-configuration-references)
- [Generic Stdio Configuration](#generic-stdio-configuration)
- [Codex](#codex)
- [Claude Code](#claude-code)
- [Cursor](#cursor)
- [Cline](#cline)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Safety Notes](#safety-notes)

## Prerequisites

Install TAFFISH `0.4.0` or later:

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --user
```

Verify that the MCP server is available:

```sh
taf --version
taffish --version
taffish-mcp --version
command -v taffish-mcp
```

If the AI client cannot find `taffish-mcp`, use the absolute path printed by
`command -v taffish-mcp` in the client configuration instead of just
`taffish-mcp`.

For user installs, the executable is usually under:

```sh
$HOME/.local/bin/taffish-mcp
```

## Official Configuration References

Use these official references as the source of truth for current client-specific
configuration syntax:

| Client or standard | Official reference |
| --- | --- |
| MCP stdio transport | [Model Context Protocol transport specification](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports) |
| Codex | [OpenAI Codex MCP documentation](https://developers.openai.com/codex/mcp) |
| Claude Code | [Claude Code MCP documentation](https://code.claude.com/docs/en/mcp) |
| Cursor | [Cursor MCP documentation](https://docs.cursor.com/en/context/mcp) |
| Cline | [Cline MCP server configuration](https://docs.cline.bot/mcp/adding-and-configuring-servers) and [Cline CLI configuration](https://docs.cline.bot/cline-cli/configuration) |

## Generic Stdio Configuration

Most MCP clients support a JSON shape similar to this:

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

If the client runs MCP servers with a restricted or non-login environment, use an
absolute path:

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

For project-aware work, start the client from the TAFFISH app project directory,
or set the server working directory to the project root if the client supports a
`cwd` field. This allows resources such as
`taffish://project/current/taffish.toml` and
`taffish://project/current/src/main.taf` to resolve the intended project.

After editing MCP configuration, restart the AI client or reload its MCP server
list if the client provides a reload action. Some clients read MCP server
definitions only at startup. If `taffish-mcp` works in the terminal but does not
appear inside the client immediately, the first thing to try is usually a client
restart or MCP reload rather than changing TAFFISH itself.

## Codex

Official reference: [OpenAI Codex MCP documentation](https://developers.openai.com/codex/mcp).

Add the server from the command line:

```sh
codex mcp add taffish -- taffish-mcp
```

Then inspect configured servers:

```sh
codex mcp list
```

Codex also supports configuring MCP servers in its config file. A minimal
`taffish-mcp` entry is:

```toml
[mcp_servers.taffish]
command = "taffish-mcp"
args = []
enabled = true
startup_timeout_sec = 20
```

If Codex cannot find `taffish-mcp`, replace `command = "taffish-mcp"` with the
absolute path returned by `command -v taffish-mcp`.

If the server was just added while Codex is already running, restart Codex or
start a new session before debugging the server command.

## Claude Code

Official reference: [Claude Code MCP documentation](https://code.claude.com/docs/en/mcp).

Add `taffish-mcp` as a stdio server:

```sh
claude mcp add --transport stdio taffish -- taffish-mcp
```

For a project-shared configuration, run the command from the project root and
use project scope:

```sh
claude mcp add --transport stdio --scope project taffish -- taffish-mcp
```

Then inspect the configured servers:

```sh
claude mcp list
```

Inside Claude Code, use the MCP inspection command provided by the client, such
as `/mcp`, to confirm that the TAFFISH server connected successfully.
If the server does not appear immediately after editing configuration, restart
Claude Code or reload its MCP server state before changing the `taffish-mcp`
configuration.

## Cursor

Official reference: [Cursor MCP documentation](https://docs.cursor.com/en/context/mcp).

For a project-local setup, create `.cursor/mcp.json` in the project root. For a
user-level setup, use Cursor's global MCP configuration path as documented by
Cursor.

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

Restart Cursor or reload the MCP servers after changing the configuration. If
the server fails to start, switch `command` to the absolute `taffish-mcp` path.

## Cline

Official references: [Cline MCP server configuration](https://docs.cline.bot/mcp/adding-and-configuring-servers) and [Cline CLI configuration](https://docs.cline.bot/cline-cli/configuration).

If you use the Cline CLI, add the server with:

```sh
cline mcp add taffish -- taffish-mcp
```

If you configure the VS Code extension directly, use Cline's MCP settings file
or UI and add a stdio server like this:

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

After saving the file, restart or reload the extension's MCP servers.

## Verification

After configuring the client, check whether it can see the TAFFISH tools,
resources, and prompts.

A healthy setup should expose tools such as:

- `taffish_doctor_check`
- `taffish_search_apps`
- `taffish_get_app_info`
- `taffish_check_project`
- `taffish_compile_project`

It may also expose resources such as:

- `taffish://config`
- `taffish://index/summary`
- `taffish://installed`
- `taffish://project/current/taffish.toml`
- `taffish://project/current/src/main.taf`

Not every client displays resources and prompts in the same way. If tools work
but resources or prompts are hidden, check the client UI and official docs before
assuming the server is broken.

## Troubleshooting

If the client cannot start the server, verify the command outside the client:

```sh
taffish-mcp --version
```

If that works in a terminal but not in the client, use an absolute path in the
client configuration.

If the absolute path is correct but the client still does not show TAFFISH tools,
restart the AI client or reload MCP servers. A missing tool list immediately
after editing config is often a client reload issue, not a `taffish-mcp` issue.

If Hub resources fail, update the local TAFFISH index:

```sh
taf update
```

If project resources resolve the wrong project, start the client from the app
project root or configure the MCP server working directory if the client supports
it.

If local documentation resources are unavailable, set:

```sh
export TAFFISH_SOURCE_DIR=/path/to/taffish
```

## Safety Notes

`taffish-mcp` is intentionally conservative. It does not expose `taf run`,
`taf publish`, container image builds, direct shell execution, arbitrary file
deletion, or arbitrary command execution.

Some tools can still write local files when explicitly called, such as
`taffish_update_index`, `taffish_new_project`, and `taffish_build_project`.
Install and uninstall tools default to `dryRun=true`. Treat write-capable MCP
tool calls as actions that should be reviewed before execution.
