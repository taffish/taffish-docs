# What Is TAFFISH

TAFFISH is a lightweight command delivery system for bioinformatics tools and workflows. It currently has three local entry points:

- `taffish`: the TAFFISH language compiler, which compiles `.taf` files into POSIX shell scripts.
- `taf`: the developer and user CLI, used to create projects, check projects, build commands, publish apps, and install apps from TAFFISH Hub.
- `taffish-mcp`: a conservative stdio MCP server that exposes safe TAFFISH tools, resources, and prompts to AI clients.

In other words, a `.taf` file describes how a tool or workflow should run. `taffish` turns that description into shell code. `taf` organizes that code into versioned, publishable, indexable, installable TAFFISH apps. `taffish-mcp` lets AI clients inspect TAFFISH projects and Hub state through a structured interface.

## Table Of Contents

- [Design Goals](#design-goals)
- [Installation](#installation)
- [The `taffish` Compiler](#the-taffish-compiler)
- [`.taf` File Structure](#taf-file-structure)
- [Parameter Syntax](#parameter-syntax)
- [Built-In Runtime Tags](#built-in-runtime-tags)
- [The `taf` CLI](#the-taf-cli)
- [MCP / AI Integration](#mcp--ai-integration)
- [Runtime Config And Mirrors](#runtime-config-and-mirrors)
- [TAFFISH App Project Structure](#taffish-app-project-structure)
- [Recommended Development Workflow](#recommended-development-workflow)
- [Tool Versus Flow](#tool-versus-flow)
- [Current Boundaries](#current-boundaries)

## Design Goals

TAFFISH does not try to replace shell, Docker, Conda, or workflow engines. It puts them into a more stable app delivery format:

- Every app has an explicit name, version, release, and command name.
- Every app stores its runtime logic in a `.taf` file.
- Every app can optionally bind to a container image.
- Every app can be built into a versioned command, such as `taf-blast-v2.16.0-r1`.
- Every app can be indexed by TAFFISH Hub and installed with `taf update` / `taf install`.
- Flow apps can declare and call other taf apps, forming traceable workflow dependencies.

## Installation

The public `taffish` repository currently provides binary releases. The
recommended user installation uses the installer from the canonical
`taffish/taffish` repository:

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --user
```

System-wide installation:

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sudo sh -s -- --system
```

Pinned version installation:

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --version 0.4.0 --user
```

For users in China, the Gitee installer can avoid GitHub raw content during
installation and initialize the China mirror config:

```sh
curl -fsSL https://gitee.com/taffish-org/taffish/raw/main/install/install-taffish.gitee.sh | sh -s -- --user
```

After installation:

```sh
taf --version
taffish --version
taffish-mcp --version
taf doctor
```

Installed commands:

```text
taffish
taf
taffish-mcp
```

Default user paths:

```text
bin  = ~/.local/bin
home = ~/.local/share/taffish
```

Default system paths:

```text
bin  = /usr/local/bin
home = /opt/taffish
```

The installer attempts to run `taf update` to initialize the local index. If the
network is unavailable, the installer prints a warning but does not roll back
the installed binaries.

`taf doctor` checks common local dependencies such as `git`, `gh`, `docker`, `podman`, `apptainer`, `sbcl`, and shell tools.

## The `taffish` Compiler

The direct responsibility of `taffish` is to compile `.taf` code to shell.

It accepts a `.taf` file and command arguments:

```sh
taffish path/to/main.taf [ARGS...]
```

It can also read source code from standard input:

```sh
cat path/to/main.taf | taffish --
```

Check version and help:

```sh
taffish --version
taffish --help
```

A minimal `.taf` file:

```taf
<shell>
echo "hello from TAFFISH"
```

Compile it:

```sh
taffish hello.taf
```

`taffish` normally prints the generated shell script. Actual execution is usually handled by `taf run` or by a built `taf-*` command.

## `.taf` File Structure

A `.taf` file is made of lines. Each line is interpreted as one of:

- blank line
- comment line: after leading whitespace, starts with `#`
- primary tag: `ARGS` or `RUN`
- subtag: `<...>`
- ordinary code line

The full structure is usually:

```taf
ARGS
<(--/-n)name>
  world

RUN
<shell>
echo "hello, ::name::"
```

Tags:

- Primary tags: `ARGS` or `RUN`
- Subtags: `<shell>`, `<container:...>`, `<taffish>`, `<taf-app:...>`

If the first effective line is ordinary code, TAFFISH automatically wraps it as:

```taf
RUN
<taffish>
...
```

If the first effective line is a subtag, TAFFISH automatically inserts `RUN`. Therefore the shortest form can be:

```taf
<shell>
echo "hello"
```

## Parameter Syntax

TAFFISH uses `::...::` as parameter slots. During compilation, slots are resolved from user input, defaults, and built-in context.

The shortest form can place slots directly in `RUN`:

```taf
<shell>
echo "input: ::input::"
echo "name: ::(--/-n)name=world::"
```

The recommended form declares parameters explicitly in `ARGS`, then references them in `RUN`:

```taf
ARGS
<!(--/-i)input>
<(--/-t)threads>
  4

RUN
<shell>
my-tool --input ::input:: --threads ::threads::
```

Each `<...>` subtag under `ARGS` declares one parameter. The body under the subtag is its default value. User input overrides that default.

### Parameter DSL

The same parameter DSL is used inside `<...>` and `::...::`:

```text
!(--/-i)input        required option: --input / -i
(--/-t)threads=8    option with inline default
(--/-v)verbose?     boolean flag
%(--/-x)internal=1   hidden option with default
$1                   positional argument
(@:)blast-step       block / slot argument
```

Common prefixes and suffixes:

- `!` marks a required argument.
- `%` marks a hidden argument, useful for internal defaults or non-user-facing settings.
- `?` marks a boolean flag. Present means true, absent means false.
- `=` sets a default value.
- `$1`, `$2` define positional arguments.
- `--` can expand to a long option name, for example `(--/-)name` maps to `--name` and `-n`.
- `-` can expand to a short option name, for example `(--/-)threads` maps to `--threads` and `-t`.

Defaults can reference other parameters:

```taf
ARGS
<name>
  sample
<message>
  hello, ::name::
<command>
  --sample ::name:: --threads ::(--/-t)threads=4::

RUN
<shell>
echo "::message::"
echo "::command::"
```

Inside an `ARGS` body, `::name::`, `@name`, and `@{name}` can all reference another parameter. Prefer `::name::` for ordinary references, because it matches the parameter slot syntax used in `RUN` and allows TAFFISH to extract inner parameter specs from defaults.

`@name` and `@{name}` are default-expression syntax. They are best reserved for string composition, especially inside parameter DSL defaults:

```taf
ARGS
<name>
  sample

RUN
<shell>
echo ::(--/-p)prefix=out-@{name}::
```

Use `@{name}` when the reference is followed immediately by letters, numbers, `-`, or `_`, because it avoids boundary ambiguity.

### `@:` Block Parameters

`(@:)name` is an important part of TAFFISH's parameter system. It creates a named slot:

```taf
ARGS
<(@:)blast-step>
  --db ::(--/-d)db=nt::
  --query ::!(--/-q)query::
  --outfmt ::(--/-of)outfmt=6::
  ::(--/-e)extra=::

RUN
<taffish>
[[taf: taf-blast-v2.16.0-r1 ::(@:)blast-step::]]
```

Here `blast-step` is not a plain string parameter. It is a composable parameter block. Its default block can contain ordinary parameter DSL entries such as `db`, `query`, `outfmt`, and `extra`. TAFFISH extracts these inner parameters, so developers can:

- Expose domain-oriented parameters such as `query`, `db`, and `outfmt`.
- Package a group of low-level tool arguments as one step parameter.
- Keep `[[taf: ...]]` calls readable in flow apps.
- Preserve an advanced escape hatch such as `extra`.

At the command-line layer, `(@:)blast-step` corresponds to the `@blast-step:` slot. Ordinary users usually do not need to pass slot blocks directly. App authors mainly use them inside `.taf` files to organize step defaults, domain parameters, and extra arguments.

### Built-In Parameters

Built-in parameters include:

```text
::*USER*::
::*HOMEDIR*::
::*WORKDIR*::
::*LOADDIR*::
::*ARGV*::
::*CMD*::
::*CPUS*::
::*CONTAINER*::
```

Example:

```taf
<shell>
echo "user   : ::*USER*::"
echo "workdir: ::*WORKDIR*::"
echo "argv   : ::*ARGV*::"
```

TAFFISH escaping is not general shell-style escaping. Only a few syntax characters are consumed by specific parsers.

At the `.taf` lexer layer, only these escapes are recognized:

```text
\:  -> :
\<  -> <
\#  -> #
\\  -> \
```

Other backslash sequences are preserved as text. For example, `\@` is not a `.taf` lexer escape. It is only interpreted as a literal `@` inside an `ARGS` default-expression context.

`ARGS` default expressions support these escapes:

```text
\@  -> @
\{  -> {
\}  -> }
\\  -> \
```

## Built-In Runtime Tags

Runtime tags are compile-time dispatch markers. They decide how the body below
the tag is emitted into shell. The examples in this section show the source
`.taf` code and the simplified shape of the generated shell. Real generated
output also includes comments, checks, temporary paths, and backend detection.

### `<shell>`

`<shell>` means the following body is shell code:

```taf
<shell>
echo "hello"
pwd
```

Compiled shape:

```sh
#!/bin/sh
echo "hello"
pwd
```

`<shell>` does not enter a container. It runs on the host shell, so host
commands, host `PATH`, and host working directory semantics apply.

### `<container:IMAGE>`

`<container:IMAGE>` runs the following shell code through an available container backend. The generic `container` backend chooses from `apptainer`, `podman`, or `docker` according to the environment.

```taf
<container:ghcr.io/taffish/blast:2.16.0>
blastp -query ::query:: -db ::db::
```

Simplified compiled shape:

```sh
# choose a backend, for example podman
podman pull ghcr.io/taffish/blast:2.16.0
podman run --rm -i \
  -w "$PWD" \
  -v "$HOME:$HOME" \
  -v "$PWD:$PWD" \
  -e "HOME=$HOME" \
  -e "USER=$USER" \
  ghcr.io/taffish/blast:2.16.0 \
  blastp -query "$query" -db "$db"
```

The exact backend command differs between Docker, Podman, and Apptainer, but the
important model is the same:

- TAFFISH chooses a container backend.
- TAFFISH checks or pulls the image when needed.
- TAFFISH sets the container working directory to the current TAFFISH workdir.
- TAFFISH bind-mounts the user home path and the current workdir into the same
  paths inside the container.
- TAFFISH passes common environment variables such as `HOME` and `USER`.
- The command body is executed inside the container image.

Because the current workdir is mounted into the container at the same path, do
not run containerized taf apps from host paths that collide with important
container paths such as `/bin`, `/usr/bin`, `/usr/local/bin`, `/lib`, `/opt`, or
the directory where the wrapped tool is installed. A host bind mount can hide
the image's original directory at that path. For example, running a containerized
app while the host current directory is `/usr/bin` can hide the container's own
`/usr/bin`, making tools that are installed there appear to be missing. Run taf
apps from a project directory, data directory, or scratch directory instead.

You can also request a specific backend:

```taf
<docker:ghcr.io/taffish/blast:2.16.0>
blastp -query ::query:: -db ::db::
```

Available backends are determined by the runtime environment. `taf run` can force a backend:

```sh
taf run --backend docker -- --query test.fa --db nr
```

`--backend` only forces generic container tags such as `<container:...>` or `<taf-app:container:...>`. If the tag explicitly says `<docker:...>`, `<podman:...>`, `<taf-app:docker:...>`, or `<taf-app:podman:...>`, TAFFISH respects that explicit backend.

When testing an unpublished local image, build and run with the same backend:

```sh
taf build --image --backend docker
taf run --backend docker -- --help
```

If the app only works with a specific backend, or the local image only exists in one backend store, use an explicit backend tag in `src/main.taf`:

```taf
<taf-app:podman:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --help
```

### `<taffish>`

`<taffish>` is commonly used in flow apps. It allows shell code to embed other taf apps:

```taf
<taffish>
[[taf: taf-fastqc-v0.12.1-r1 sample.fq]]
[[taf: taf-multiqc-v1.19-r1 .]]
```

Embedding syntax:

```text
[[taf: taf-command args...]]
```

At compile time, TAFFISH compiles those taf apps into temporary shell steps, then executes them inside the current workflow. This makes it possible for flow apps to compose existing tools while preserving explicit dependency relationships.

Simplified compiled shape:

```sh
taffish_tmpdir=$(mktemp -d)
taf-fastqc-v0.12.1-r1 --compile sample.fq > "$taffish_tmpdir/step-1.sh"
taf-multiqc-v1.19-r1 --compile . > "$taffish_tmpdir/step-2.sh"
chmod +x "$taffish_tmpdir"/step-*.sh
"$taffish_tmpdir/step-1.sh"
"$taffish_tmpdir/step-2.sh"
```

To print a literal `[[taf: ...]]` inside a `<taffish>` block, escape the bracket with `\[` or `\]`:

```taf
<taffish>
echo \[[taf: taf-example arg]]
```

### `<taf-app:...>`

`<taf-app:...>` is the common top-level tag in app projects. Tool projects created by `taf new --tool` usually use it.

Example containerized tool app:

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --input ::input:: --output ::output::
```

The built command compiles the `.taf` file into shell and runs it. Users see an ordinary command, while the internals are still compiled and dispatched by TAFFISH.

For a shell tool app:

```taf
<taf-app:shell>
my-tool ::*ARGV*::
```

Compiled shape when the user runs `taf-my-tool -- --alpha 1`:

```sh
#!/bin/sh
my-tool --alpha 1
```

For a containerized tool app:

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool ::*ARGV*::
```

Compiled shape when the user runs `taf-my-tool -- --alpha 1`:

```sh
#!/bin/sh
# choose/pull/run container as described above
podman run --rm -i \
  -w "$PWD" \
  -v "$HOME:$HOME" \
  -v "$PWD:$PWD" \
  ghcr.io/taffish/my-tool:0.1.0-r1 \
  my-tool --alpha 1
```

`<taf-app:...>` is therefore a user-facing app wrapper around the lower-level
runtime tags. It also preserves app command semantics such as `::*ARGV*::`,
`--compile`, and package-manager integration.

## The `taf` CLI

`taf` is the developer and package-management CLI. Its commands fall into three groups.

### Project Commands

```sh
taf new
taf check
taf compile
taf run
taf build
taf publish
```

### Hub Commands

```sh
taf update
taf search
taf info
taf install
taf uninstall
taf list
taf which
```

### System Commands

```sh
taf doctor
taf config
taf history
```

`doctor` checks or initializes the local TAFFISH environment. `config` shows or
initializes runtime configuration. `history` shows or clears local run history.

Common usage:

Create a flow project:

```sh
taf new my-flow
cd my-flow
```

Create a tool project:

```sh
taf new my-tool --tool
```

Create a tool project with a Dockerfile and GitHub Actions image workflow:

```sh
taf new my-tool --tool --docker
```

Check a project:

```sh
taf check
```

Run a project:

```sh
taf run -- --input sample.fa
```

Build a versioned command:

```sh
taf build
```

Publish preview with release notes:

```sh
taf publish --release --dry-run
```

Publish with release notes:

```sh
taf publish --release --yes --build
```

`taf new` creates an ignored `release.md` draft. With
`taf publish --release`, the first line becomes the publish message and the
whole file becomes the GitHub Release notes.

Update the Hub index:

```sh
taf update
```

Install an app:

```sh
taf install blast 2.16.0-r1
```

List installed apps:

```sh
taf list
```

## MCP / AI Integration

TAFFISH `0.4.0` adds `taffish-mcp`, a conservative MCP stdio server for AI
clients. The first interface exposes safe project, Hub, config, history,
resource, and prompt operations. It does not expose `taf run`, `taf publish`, or
image-building actions.

Example MCP client configuration:

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

This lets an AI client inspect TAFFISH projects, search the local index, read
project resources, and prepare safe project actions without relying first on
unstructured terminal text.

For the focused guide to tools, resources, prompts, and safety boundaries, see
[TAFFISH MCP Guide](taffish-mcp.en.md). For Codex, Claude Code, Cursor, Cline,
and generic MCP client configuration examples, see
[Using TAFFISH MCP With AI Clients](mcp-clients.en.md).

## Runtime Config And Mirrors

Since TAFFISH `0.2.0`, `taf` provides runtime configuration for mirror and
custom source support. The current recommended release is `0.4.0`. The default
config paths are:

```text
user   = ~/.local/share/taffish/config.toml
system = /opt/taffish/config.toml
```

Inspect the effective config:

```sh
taf config
taf config path
```

Initialize the default GitHub profile:

```sh
taf config init --github
```

Initialize the China mirror profile:

```sh
taf config init --china --force
taf update
```

The China profile is a simple template:

```toml
schema_version = "taffish.config/v1"
profile = "china"
language = "en"

[index]
url = "https://gitee.com/taffish-org/taffish-index/raw/main/index/index.json"

[[source.rewrite]]
from = "https://github.com/taffish/"
to = "https://gitee.com/taffish-org/"
enabled = true
```

`[index].url` controls the default index used by `taf update`.
`[[source.rewrite]]` rewrites canonical app repository URLs when `taf install`
clones apps. Users or maintainers can adapt the same mechanism for internal Git
services, as long as the mirror provides compatible repositories, tags, and the
same TAFFISH index schema.

## TAFFISH App Project Structure

A typical app project:

```text
my-tool/
  README.md
  taffish.toml
  src/
    main.taf
  docs/
    help.md
  target/
    .gitkeep
  docker/
    Dockerfile
  .github/
    workflows/
      build-image.yml
```

Required files:

- `taffish.toml`: project metadata.
- `src/main.taf`: app runtime source.
- `docs/help.md`: help text shown by the built command.

`target/` stores built artifacts and frozen source snapshots.

Core metadata lives in `taffish.toml`:

```toml
[package]
name = "my-tool"
kind = "tool"
version = "0.1.0"
release = 1
license = "Apache-2.0"
main = "src/main.taf"

[repository]
url = "https://github.com/taffish/my-tool"

[command]
name = "taf-my-tool"

[runtime]
pipe = true
command_mode = true
```

For a flow:

```toml
[package]
name = "my-flow"
kind = "flow"
version = "0.1.0"
release = 1
license = "Apache-2.0"
main = "src/main.taf"

[runtime]
pipe = false
command_mode = false
```

For a container app:

```toml
[container]
image = "ghcr.io/taffish/my-tool:0.1.0-r1"
dockerfile = "docker/Dockerfile"
build_platforms = "linux/amd64,linux/arm64"
```

For flow dependencies:

```toml
[dependencies]
taf-fastqc = "0.12.1-r1"
taf-multiqc = "1.19-r1"
```

If the same flow requires multiple versions of the same dependency:

```toml
[dependencies]
taf-align = ["2.0.0-r1", "2.1.0-r1"]
```

That means both versions are required. It does not mean "choose one".

## Recommended Development Workflow

A common app lifecycle:

```sh
taf new my-tool --tool --docker
cd my-tool
```

Edit:

```text
taffish.toml
src/main.taf
docs/help.md
docker/Dockerfile
```

Check locally:

```sh
taf check
```

For a containerized app, build the local image first and choose the backend explicitly:

```sh
taf build --image --backend docker
```

Run locally:

```sh
taf run --backend docker -- --help
```

Build the command wrapper:

```sh
taf build
```

To build image and command wrapper together:

```sh
taf build --all --backend docker
```

Preview publishing with release notes:

```sh
taf publish --release --dry-run
```

Publish with release notes:

```sh
taf publish --release --yes --build
```

Before publishing, edit the ignored `release.md` draft. Its first line becomes
the publish message, and the full file is used as the GitHub Release notes.

After publishing, the app repository has a release tag such as:

```text
v0.1.0-r1
```

TAFFISH Hub's index automation later scans the repository and adds it to the Hub index.

## Tool Versus Flow

### Tool App

A tool app usually wraps one upstream tool. It often:

- Uses `<taf-app:shell>` or `<taf-app:container:...>`.
- Uses `runtime.pipe = true`.
- Uses `runtime.command_mode = true`.
- Exposes a command that behaves like a normal command-line tool.

### Flow App

A flow app combines multiple taf apps into an analysis workflow. It usually:

- Uses `<taffish>`.
- Calls other apps with `[[taf: ...]]`.
- Declares dependencies in `[dependencies]`.
- Uses `runtime.pipe = false`.
- Uses `runtime.command_mode = false`.

## Current Boundaries

Current practical boundaries:

- TAFFISH is not a general-purpose programming language; it is an app description language that compiles to shell.
- TAFFISH does not directly replace Docker, Podman, or Apptainer; it selects and calls them as container backends.
- The official Hub currently publishes through the `taffish` GitHub organization and is not an open self-service publishing platform.
- Container image builds are handled by each app repository's GitHub Actions workflow, not by `taffish-index`.
- Runtime mirror config can rewrite index and app repository access, but it does not automatically mirror container registries.
- The public `taffish` repository currently focuses on binary distribution; core source code is not public yet.
