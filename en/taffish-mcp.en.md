# TAFFISH MCP Guide

`taffish-mcp` is the MCP stdio server distributed with TAFFISH `0.4.0` and later.
It gives AI clients a structured way to inspect TAFFISH projects, query local Hub
state, read selected resources, and prepare safe project actions.

It is intentionally conservative. The first interface is designed for inspection,
planning, and low-risk project maintenance. It does not expose `taf run`,
`taf publish`, container image builds, or other high-impact execution paths.

## Table Of Contents

- [When To Use It](#when-to-use-it)
- [Installation And Verification](#installation-and-verification)
- [MCP Client Configuration](#mcp-client-configuration)
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
- read the current project's `taffish.toml` and `src/main.taf`
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
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --version 0.4.0 --user
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

## Current Tool Surface

The current MCP tool surface mirrors safe parts of `taf` and selected project
lifecycle operations.

Environment and config:

| Tool | Purpose |
| --- | --- |
| `taffish_doctor_check` | Check local TAFFISH directories, `PATH`, and related executables without initializing or modifying the system. |
| `taffish_config_get` | Return the effective TAFFISH runtime config. |
| `taffish_config_path` | Return TAFFISH config file paths. |

Hub and index:

| Tool | Purpose |
| --- | --- |
| `taffish_update_index` | Update the local TAFFISH index. This writes index files under TAFFISH home. |
| `taffish_search_apps` | Search apps from the local TAFFISH index. |
| `taffish_get_app_info` | Resolve an app, command, or artifact from the local index and return its version record. |
| `taffish_list_apps` | List installed apps or indexed online apps. |
| `taffish_which` | Show local install paths and metadata for an installed app or command. |
| `taffish_history_list` | Read local TAFFISH run history without clearing it. |

Install and uninstall planning:

| Tool | Purpose |
| --- | --- |
| `taffish_install_app` | Install apps or commands from the local index. Defaults to `dryRun=true` for safety. |
| `taffish_uninstall_app` | Uninstall local apps or commands. Defaults to `dryRun=true` for safety. |

Project work:

| Tool | Purpose |
| --- | --- |
| `taffish_new_project` | Create a new TAFFISH app project in the current working directory. This writes files. |
| `taffish_check_project` | Check the current TAFFISH app project. Read-only. |
| `taffish_compile_project` | Compile the current project and return generated shell. It does not execute it. |
| `taffish_build_project` | Build the current project command wrapper. Container image builds are intentionally not exposed. |

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
| `taffish://docs/taf-language` | Local TAFFISH language documentation when a source checkout is available. |
| `taffish://docs/project` | Local TAFFISH project documentation when a source checkout is available. |

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
| `prepare-taffish-publish` | Prepare a project for publishing without running `taf publish`. |

## Safety Model

The first MCP interface is deliberately narrow.

It does not expose:

- `taf run`
- `taf publish`
- container image builds
- direct shell execution
- arbitrary file deletion
- arbitrary command execution

Some tools still write local files when explicitly called, such as
`taffish_update_index`, `taffish_new_project`, and `taffish_build_project`.
Install and uninstall tools default to `dryRun=true`. An AI client should still
ask for user confirmation before calling write-capable tools.

## Typical Workflows

Inspect the local setup:

1. Call `taffish_doctor_check`.
2. Read `taffish://config`.
3. If needed, call `taffish_config_path`.

Search and explain an app:

1. Call `taffish_update_index` if the index is missing or stale.
2. Call `taffish_search_apps`.
3. Call `taffish_get_app_info` for the selected app.
4. Explain versions, dependencies, source, and container image metadata.

Debug a project:

1. Call `taffish_check_project`.
2. Read `taffish://project/current/taffish.toml`.
3. Read `taffish://project/current/src/main.taf`.
4. Suggest minimal changes before editing files.

## Troubleshooting

If the MCP client cannot start the server, first verify the command directly:

```sh
taffish-mcp --version
```

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
