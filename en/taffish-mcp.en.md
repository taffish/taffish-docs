# TAFFISH MCP Guide

[English](taffish-mcp.en.md) | [中文](../zh/taffish-mcp.cn.md)

`taffish-mcp` is the MCP stdio server distributed with TAFFISH `0.4.0` and later.
It gives AI clients a structured way to inspect TAFFISH projects, query local Hub
state, read selected resources, validate or compile `.taf` source without
executing it, and prepare safe project actions.

TAFFISH `0.8.0` is the current recommended release for MCP use. It keeps the
read-only TAF source/file compiler helpers introduced in `0.5.0`, the
app/project inspection and safe app invocation compilation added in `0.6.0`,
the runtime container backend override alignment from `0.7.0`, and exposes
smoke/trust metadata for app and project inspection without running containers.

It is intentionally conservative. The interface is designed for inspection,
planning, and low-risk project maintenance. It does not expose `taf run`,
`taf publish`, container image builds, or other high-impact execution paths.

## Table Of Contents

- [When To Use It](#when-to-use-it)
- [Installation And Verification](#installation-and-verification)
- [MCP Client Configuration](#mcp-client-configuration)
- [Client-Specific Setup](#client-specific-setup)
- [Current Tool Surface](#current-tool-surface)
- [Resources](#resources)
- [Prompts](#prompts)
- [Safety Model](#safety-model)
- [Typical Workflows](#typical-workflows)
- [Troubleshooting](#troubleshooting)
- [Current Boundaries](#current-boundaries)

## When To Use It

Use `taffish-mcp` when an AI client needs structured access to TAFFISH context:

- inspect the local TAFFISH environment and config
- search the local TAFFISH index
- resolve app metadata before suggesting an install
- inspect installed or indexed app source, help, containers, dependencies, smoke metadata, and trust status
- compile a candidate app invocation without running it
- read the current project's `taffish.toml` and `src/main.taf`
- summarize current project usage, help, release notes, and local artifacts
- prepare a tool or flow project plan
- debug a TAFFISH project before editing files

For ordinary command-line use, continue to use `taf` and `taffish` directly.

## Installation And Verification

`taffish-mcp` is installed by the TAFFISH installer together with `taf` and
`taffish`.

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --user
```

Pinned install:

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --version 0.8.0 --user
```

Verify:

```sh
taf --version
taffish --version
taffish-mcp --version
```

If `taffish-mcp` is not found, make sure the install bin directory is in `PATH`.
For user installs, this is usually:

```sh
export PATH="$HOME/.local/bin:$PATH"
```

## MCP Client Configuration

Use `taffish-mcp` as a stdio MCP server.

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

The exact location for this JSON depends on the MCP client. The server expects to
run in a normal shell environment where `taf`, TAFFISH home paths, and container
backend tools can be discovered when needed.

After changing MCP configuration, some clients need a restart or an explicit MCP
server reload before the new server appears. If `taffish-mcp --version` works in
a terminal but the client does not show TAFFISH tools yet, try reloading the
client-side MCP configuration first.

For Codex, Claude Code, Cursor, Cline, and generic MCP client examples, see
[Using TAFFISH MCP With AI Clients](mcp-clients.en.md).

## Client-Specific Setup

This document is the reference for what `taffish-mcp` exposes. Client setup
details are maintained separately in [Using TAFFISH MCP With AI Clients](mcp-clients.en.md).

The client configuration examples were generated and last checked on 2026-05-12. MCP client
configuration formats can change, so always compare the examples with the
official client documentation when setting up a new machine or a new client
version.

Official MCP configuration references:

| Client or standard | Official reference |
| --- | --- |
| MCP stdio transport | [Model Context Protocol transport specification](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports) |
| Codex | [OpenAI Codex MCP documentation](https://developers.openai.com/codex/mcp) |
| Claude Code | [Claude Code MCP documentation](https://code.claude.com/docs/en/mcp) |
| Cursor | [Cursor MCP documentation](https://docs.cursor.com/en/context/mcp) |
| Cline | [Cline MCP server configuration](https://docs.cline.bot/mcp/adding-and-configuring-servers) and [Cline CLI configuration](https://docs.cline.bot/cline-cli/configuration) |

## Current Tool Surface

The current MCP tool surface mirrors safe parts of `taf`, selected app
inspection operations, and selected project lifecycle operations.

Environment and config:

| Tool | Purpose |
| --- | --- |
| `taffish_get_version` | Return the installed `taffish-mcp` version. |
| `taffish_get_help` | Return concise help for the MCP server. |
| `taffish_check_environment` | Check local TAFFISH directories, `PATH`, and related executables without initializing or modifying the system. |
| `taffish_get_config` | Return the effective TAFFISH runtime config. |
| `taffish_get_config_paths` | Return TAFFISH config file paths. |

TAF source/file compiler helpers:

| Tool | Purpose |
| --- | --- |
| `taffish_validate_source` | Validate a provided `.taf` source string without executing generated shell. |
| `taffish_validate_file` | Validate a `.taf` file path without executing generated shell. |
| `taffish_compile_source` | Compile a provided `.taf` source string and return shell code without running it. |
| `taffish_compile_file` | Compile a `.taf` file path and return shell code without running it. |
| `taffish_summarize_source` | Summarize a provided `.taf` source string for quick inspection. |
| `taffish_summarize_file` | Summarize a `.taf` file path for quick inspection. |

For compile tools, pass `containerBackend` when backend choice matters. If that
argument is omitted, `TAFFISH_CONTAINER_BACKEND=apptainer|podman|docker` is
used when set in the MCP server environment. An explicit `containerBackend`
argument always has priority.

Hub and index:

| Tool | Purpose |
| --- | --- |
| `taffish_update_index` | Update the local TAFFISH index. This writes index files under TAFFISH home. |
| `taffish_search_apps` | Search apps from the local TAFFISH index. |
| `taffish_get_app_info` | Resolve an app, command, or artifact from the local index and return its version record. |
| `taffish_list_apps` | List installed apps or indexed online apps. |
| `taffish_locate_app` | Show local install paths and metadata for an installed app or command. |
| `taffish_list_history` | Read local TAFFISH run history without clearing it. |

App inspection and invocation planning:

| Tool | Purpose |
| --- | --- |
| `taffish_resolve_app` | Resolve an app, command alias, or version-pinned artifact command using local index and install metadata. |
| `taffish_inspect_app` | Inspect index metadata plus installed `taffish.toml`, `src/main.taf`, and `docs/help.md` when local source is installed. |
| `taffish_summarize_app_usage` | Return AI-oriented usage data for an app: command, args, help, containers, dependencies, smoke metadata, and trust status. |
| `taffish_compile_app_invocation` | Validate candidate app arguments and compile the installed app's `main.taf` to shell without running it. |

`taffish_compile_app_invocation` follows the same backend priority model:
explicit `containerBackend` first, then `TAFFISH_CONTAINER_BACKEND` when set,
then TAFFISH's normal runtime backend selection.

For containerized apps, inspection and usage summaries expose index-level
`container.digest`, `container.platforms`, `smoke`, `trust`, and `source.commit`
when available. MCP does not run smoke tests, pull images, or start containers;
it only surfaces metadata that local TAFFISH and the Hub/index have already
recorded.

Install and uninstall planning:

| Tool | Purpose |
| --- | --- |
| `taffish_install_app` | Install apps or commands from the local index. Defaults to `dryRun=true` for safety. |
| `taffish_uninstall_app` | Uninstall local apps or commands. Defaults to `dryRun=true` for safety. |

Project work:

| Tool | Purpose |
| --- | --- |
| `taffish_create_project` | Create a new TAFFISH app project in the current working directory. This writes files. |
| `taffish_check_project` | Check the current TAFFISH app project. Read-only. |
| `taffish_inspect_project` | Inspect the current project manifest, `src/main.taf` summary, `docs/help.md`, `release.md`, and local artifacts. Read-only. |
| `taffish_summarize_project_usage` | Return AI-oriented usage data for the current project: command, args, help, containers, dependencies, smoke metadata, and local trust notes. Read-only. |
| `taffish_compile_project` | Compile the current project and return generated shell. It does not execute it. |
| `taffish_build_project` | Build the current project command wrapper. Container image builds are intentionally not exposed. |

`taffish_compile_project` also accepts explicit backend selection for generic
container tags. This keeps AI-assisted compile previews consistent with
installed `taf-*` commands and local `taffish` compilation.

## Resources

`taffish-mcp` exposes selected read-only resources:

| URI | Content |
| --- | --- |
| `taffish://config` | Effective TAFFISH runtime configuration. |
| `taffish://index/summary` | Summary of the local TAFFISH index. |
| `taffish://installed` | Installed TAFFISH apps and commands. |
| `taffish://history` | Recent local TAFFISH execution history. |
| `taffish://project/current/taffish.toml` | Current project manifest, if the working directory is inside a TAFFISH project. |
| `taffish://project/current/src/main.taf` | Current project main `.taf` script, if available. |
| `taffish://project/current/docs/help.md` | Current project help document, if available. |
| `taffish://project/current/release.md` | Current project release notes draft, if available. |
| `taffish://project/current/summary` | Structured summary of the current TAFFISH project. |
| `taffish://docs/taf-language` | Local TAFFISH language documentation when a source checkout is available. |
| `taffish://docs/project` | Local TAFFISH project documentation when a source checkout is available. |
| `taffish://mcp/help` | Summary of TAFFISH MCP tools, resources, and safe usage. |
| `taffish://mcp/tools` | Machine-readable MCP tool schemas exposed by the server. |
| `taffish://mcp/tool-groups` | Concise grouping and intended use of MCP tools. |
| `taffish://mcp/safety` | Safety policy and boundaries for TAFFISH MCP tools. |
| `taffish://compiler/help` | How to use source/file compiler tools exposed by TAFFISH MCP. |
| `taffish://language/taf-examples` | Small TAF examples useful for AI clients. |
| `taffish://hub/install-model` | How TAFFISH MCP treats index, install, uninstall, and local commands. |
| `taffish://mcp/app-inspection-model` | How AI clients should inspect, understand, and safely compile taf-app invocations. |
| `taffish://mcp/project-inspection-model` | How AI clients should inspect, debug, and safely compile current TAFFISH projects. |

For local documentation resources, set `TAFFISH_SOURCE_DIR` to a TAFFISH source
checkout if the client is not launched from that checkout.

## Prompts

The server also provides reusable prompts for common TAFFISH work:

| Prompt | Purpose |
| --- | --- |
| `create-taffish-tool` | Guide wrapping one bioinformatics command as a TAFFISH tool project. |
| `create-taffish-flow` | Guide composing installed TAFFISH apps into a flow project. |
| `debug-taffish-project` | Analyze a TAFFISH project using project checks and source files before suggesting fixes. |
| `explain-taf-script` | Explain a TAF script in terms of tags, arguments, containers, and dependencies. |
| `explain-taf-source` | Explain a TAF source string or local `.taf` path through validate/summarize/compile helpers. |
| `write-safe-taf-source` | Draft TAF source and validate or compile it without executing generated shell. |
| `inspect-taffish-app` | Inspect an indexed or installed taf-app before suggesting usage, install, or flow composition. |
| `prepare-taffish-publish` | Prepare a project for publishing without running `taf publish`. |

## Safety Model

The MCP interface is deliberately narrow.

It does not expose:

- `taf run`
- `taf publish`
- container image builds
- direct shell execution
- arbitrary file deletion
- arbitrary command execution

TAF source/file compiler tools are read-only. They may parse, validate,
summarize, or compile source to shell text, but they do not execute the compiled
shell.

Some tools still write local files when explicitly called, such as
`taffish_update_index`, `taffish_create_project`, and `taffish_build_project`.
Install and uninstall tools default to `dryRun=true`. An AI client should still
ask for user confirmation before calling write-capable tools.

## Typical Workflows

Inspect the local setup:

1. Call `taffish_check_environment`.
2. Read `taffish://config`.
3. If needed, call `taffish_get_config_paths`.

Search and explain an app:

1. Call `taffish_update_index` if the index is missing or stale.
2. Call `taffish_search_apps`.
3. Call `taffish_resolve_app` for the selected app.
4. Call `taffish_summarize_app_usage` for command, args, help, containers, and dependencies.
5. Check the returned smoke/trust fields before recommending a containerized app in a reproducible workflow.
6. Use `taffish_inspect_app` when raw `taffish.toml`, `src/main.taf`, or `docs/help.md` context is needed.
7. Use `taffish_compile_app_invocation` to validate candidate arguments without running the app.
8. Pass `containerBackend` when the user needs a specific Docker, Podman, or Apptainer compile preview.

Debug a project:

1. Call `taffish_inspect_project`.
2. Read `taffish://project/current/taffish.toml`.
3. Read `taffish://project/current/src/main.taf`.
4. Read `taffish://project/current/docs/help.md` or `taffish://project/current/release.md` when relevant.
5. Call `taffish_check_project` for strict validation.
6. For containerized projects, review `[smoke]` metadata in the project summary and replace any placeholder before publishing.
7. Call `taffish_compile_project` to review generated shell without executing it. Pass `containerBackend` when backend-specific container code matters.
8. Suggest minimal changes before editing files.

Prepare a project for publish:

1. Call `taffish_summarize_project_usage`.
2. Call `taffish_inspect_project` to review manifest, source, help, release notes, artifacts, Dockerfile, and workflow state.
3. Call `taffish_check_project`.
4. Call `taffish_build_project` only if the user approves local writes.
5. Do not run `taf publish` through MCP.

Inspect a TAF source file:

1. Call `taffish_validate_file`.
2. Call `taffish_summarize_file` to explain the structure.
3. Call `taffish_compile_file` if shell output is needed for review.
4. Review the generated shell without executing it.

## Troubleshooting

If the MCP client cannot start the server, first verify the command directly:

```sh
taffish-mcp --version
```

If this command works but the client still does not list TAFFISH tools, restart
the AI client or reload MCP servers. Some clients do not pick up changed MCP
configuration until their MCP layer is restarted.

If resources such as `taffish://index/summary` fail, run:

```sh
taf update
```

If local documentation resources are unavailable, launch the client from a
TAFFISH source checkout or set:

```sh
export TAFFISH_SOURCE_DIR=/path/to/taffish
```

## Current Boundaries

`taffish-mcp` is not a replacement for human review, `taf publish`, or app
runtime testing. It is best treated as a structured assistant interface for
inspection, explanation, planning, and low-risk project maintenance.
