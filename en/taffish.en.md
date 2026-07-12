# What Is TAFFISH

TAFFISH is a shell-native reproducible execution and delivery layer for
bioinformatics command-line tools and lightweight workflows. It brings
command-level reproducibility to the shell commands that bioinformaticians
already use, without requiring them to adopt a new workflow system first.

TAFFISH has three local entry points:

- `taffish`: the TAFFISH language compiler, which compiles `.taf` files into POSIX shell scripts.
- `taf`: the developer and user CLI, used to create projects, check projects, build commands, publish apps, and install apps from TAFFISH Hub.
- `taffish-mcp`: a conservative stdio MCP server that exposes safe TAFFISH tools, resources, prompts, app inspection, and project inspection to AI clients.

In other words, a `.taf` file describes how a tool or lightweight workflow
should run. `taffish` turns that description into shell code. `taf` organizes
that code into versioned, container-resolved, publishable, indexable,
installable, and composable executable packages. `taffish-mcp` lets AI clients
inspect TAFFISH projects, apps, and Hub state through a structured interface.

## Table Of Contents

- [Design Goals](#design-goals)
- [Installation](#installation)
- [User And System Scope](#user-and-system-scope)
- [The `taffish` Compiler](#the-taffish-compiler)
- [`.taf` File Structure](#taf-file-structure)
- [Parameter Syntax](#parameter-syntax)
- [Built-In Runtime Tags](#built-in-runtime-tags)
- [The `taf` CLI](#the-taf-cli)
- [MCP / AI Integration](#mcp--ai-integration)
- [Runtime Config And Mirrors](#runtime-config-and-mirrors)
- [Open Source And Source Builds](#open-source-and-source-builds)
- [TAFFISH App Project Structure](#taffish-app-project-structure)
- [Recommended Development Workflow](#recommended-development-workflow)
- [Tool Versus Flow](#tool-versus-flow)
- [Current Boundaries](#current-boundaries)

## Design Goals

TAFFISH does not try to replace shell, containers, package managers, or
workflow engines. It uses an executable package model to make tool invocations
more stable, portable, and directly usable as ordinary shell commands:

- Every app has an explicit name, version, release, and command name.
- Every app stores its runtime logic in a `.taf` file.
- Every app can optionally bind to a container image.
- Every app can be built into a versioned command, such as `taf-blast-v2.16.0-r1`.
- Every app can be indexed by TAFFISH Hub and installed with `taf update` / `taf install`.
- Flow apps can declare and call other taf apps for lightweight composition.

TAFFISH is not a general workflow engine. Systems such as Nextflow and
Snakemake can continue to handle DAG orchestration, scheduling, caching, and
pipeline-level reproducibility while calling installed `taf-*` commands as
ordinary shell commands.

## Installation

The public `taffish` repository provides the open-source TAFFISH
implementation, installers, source-tree build documentation, and binary
release payloads. The recommended user installation uses the installer from
that canonical repository:

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --user
```

System-wide installation:

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sudo sh -s -- --system
```

Pinned version installation:

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --version X.Y.Z --user
```

Replace `X.Y.Z` with the release version you want to install.

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
taf doctor --user
```

Installed commands:

```text
taffish
taf
taffish-mcp
```

The installer also copies shell completion files and Vim syntax files under
TAFFISH home, usually
`~/.local/share/taffish/share/completions` and
`~/.local/share/taffish/share/vim` for a user install.

| Item | User installation | System installation |
| --- | --- | --- |
| Core commands | `~/.local/bin` | `/usr/local/bin` |
| Installed `taf-*` commands | `~/.local/share/taffish/bin` | `/usr/local/bin` |
| TAFFISH home | `~/.local/share/taffish` | `/opt/taffish` |

The installer attempts to initialize the matching scope and update its index.
If the network is unavailable, it prints a warning but does not roll back the
installed binaries. It copies shell and editor integration files but does not
edit personal or global startup files.

`taf doctor` checks common local dependencies such as `git`, `gh`, `docker`, `podman`, `apptainer`, `sbcl`, and shell tools.

For source builds, release verification, supported maintainer build paths, and
contribution notes, see the
[taffish/taffish README](https://github.com/taffish/taffish).

## User And System Scope

TAFFISH keeps user and system state separate. Scope-aware `taf` commands
support `--user` / `-u` and `--system` / `-s`, and default to user scope on
every invocation. A system-wide binary installation does not make later
commands use system state automatically.

Typical personal initialization:

```sh
taf doctor --init --user
taf update --user
taf install --user fastqc
```

Typical administrator workflow:

```sh
sudo taf doctor --init --system
sudo taf update --system
sudo taf install --system fastqc
taf list --system
```

Use `sudo` for commands that write system state. Read-only commands such as
`taf search --system`, `taf list --system`, and `taf which --system` normally
do not require it. For shared workstations, servers, HPC login nodes, global
completion, configuration inheritance, upgrades, and removal, see
[TAFFISH System Administration](system-administration.en.md).

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
- `?` marks a boolean flag. Present is truthy in argument binding; absent is
  empty or false-like. Do not rely on a fixed textual representation such as
  `true` or `false`.
- `=` sets a default value.
- `$1`, `$2` read positional arguments from user input.
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
<!(--/-q)query>
<(--/-d)db>
  nt
<(--/-of)outfmt>
  6

<(@:)blast-step>

RUN
<taffish>
[[taf: taf-blast-v2.16.0-r1 blastn -db ::db:: -query ::query:: -outfmt ::outfmt:: ::(@:)blast-step::]]
```

Here `blast-step` is not a plain string parameter. It is a named step parameter
slot. In formal flows, this slot should be empty by default: stable parameters
managed by the flow itself, such as `db`, `query`, and `outfmt`, are written
directly in the command body. `::(@:)blast-step::` appends native BLAST
arguments only when the user explicitly passes `@blast-step: ... @:`. This lets
developers:

- Expose domain-oriented parameters such as `query`, `db`, and `outfmt`.
- Keep flow-managed defaults separate from advanced user-supplied arguments.
- Keep the real `[[taf: ...]]` call clear and auditable.
- Preserve a per-step entry point for native low-level arguments.

At the command-line layer, `(@:)blast-step` corresponds to the `@blast-step:`
slot. Ordinary users usually do not need to pass slot blocks directly. App
authors mainly use them inside `.taf` files to provide an empty advanced append
slot for a real tool call.

In flow apps, each real analytical `[[taf: ...]]` call site should usually have
a descriptive `(@:)xxx-step` block, and the call site should include
`::(@:)xxx-step::`. This lets ordinary users work with flow-level domain
parameters while advanced users can pass native low-level tool arguments step by
step. `::(@:)xxx-step::` must expand to nothing by default and should not be
prefilled in source code, templates, shell variables, or helper scripts. A flow
should still run completely without any `@step:` arguments.

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

For installed `taf-*` commands or direct `taffish` compilation, use
`TAFFISH_CONTAINER_BACKEND=apptainer|podman|docker` to force generic
`<container:...>` tags at runtime:

```sh
TAFFISH_CONTAINER_BACKEND=podman taf-my-tool-v0.1.0-r1 [ARGS...]
TAFFISH_CONTAINER_BACKEND=podman taf-my-tool-v0.1.0-r1 --compile -- [ARGS...]
```

This environment variable does not override explicit `<docker:...>`,
`<podman:...>`, or `<apptainer:...>` tags. When using `taf run`,
`--backend` has priority over `TAFFISH_CONTAINER_BACKEND`.

TAFFISH `0.9.0` adds structured backend-specific runtime arguments inside the
container tag. They are written after `$` as `@[target: args]` blocks:

```taf
<container:ghcr.io/taffish/my-tool:1.0.0-r1$@[all: --network host][docker/podman: --security-opt=label=disable]>
my-tool --help
```

Targets can be `all`, `container` as an alias of `all`, `docker`, `podman`,
`apptainer`, or backend combinations such as `docker/podman`. These tag
arguments are for app implementation requirements. For example, if a wrapped app
requires GPU access to work correctly, the app should declare the backend-specific
GPU flags in `src/main.taf`:

```taf
<container:ghcr.io/taffish/my-gpu-tool:1.0.0-r1$@[docker: --gpus all][podman: --device nvidia.com/gpu=all][apptainer: --nv]>
my-gpu-tool --help
```

Local runtime policy belongs in environment variables instead:

```sh
TAFFISH_DOCKER_RUN_ARGS="--platform linux/amd64" taf-my-tool ...
TAFFISH_PODMAN_RUN_ARGS="--platform linux/amd64" taf-my-tool ...
TAFFISH_APPTAINER_RUN_ARGS="--bind /scratch:/scratch" taf-my-tool ...
```

These variables are for single-run, machine, cluster, platform, or site policy.
They append backend-specific runtime arguments without changing the app source.

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
[[taf: taf-fastqc-v0.12.1-r1 fastqc sample.fq]]
[[taf: taf-multiqc-v1.19-r1 multiqc .]]
```

Embedding syntax:

```text
[[taf: taf-command args...]]
```

At compile time, TAFFISH compiles those taf apps into temporary shell steps, then executes them inside the current workflow. This makes it possible for flow apps to compose existing tools while preserving explicit dependency relationships.

In formal flows, `[[taf: ...]]` should appear at the code location where the
tool is really executed. Do not collect all taf dependencies in a dummy block or
`if false` section at the top and then run them indirectly through variables,
bare wrappers, or generated scripts. Apart from a small amount of
shell/coreutils usage, non-shell-native bioinformatics, statistical, plotting,
database, and model-related commands should be called through explicit
versioned taf apps.

Simplified compiled shape:

```sh
taffish_tmpdir=$(mktemp -d)
taf-fastqc-v0.12.1-r1 --compile fastqc sample.fq > "$taffish_tmpdir/step-1.sh"
taf-multiqc-v1.19-r1 --compile multiqc . > "$taffish_tmpdir/step-2.sh"
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
taf new APP_NAME
taf check
taf compile [ARGS...]
taf run [ARGS...]
taf build
taf publish
```

### Hub Commands

```sh
taf update
taf search KEYWORD
taf info APP_OR_COMMAND
taf install APP_OR_COMMAND
taf outdated
taf upgrade
taf prune
taf uninstall APP_OR_COMMAND
taf list
taf which TAF_COMMAND
```

These commands operate on user state by default. Use `--user` or `--system`
explicitly in scripts and administrative runbooks; installing the core
binaries system-wide does not persist system scope. Commands that modify
system state normally need `sudo`, while read-only system inspection does not.

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

Compile the current project and print shell code:

```sh
taf compile -- --input sample.fa
```

Run a project:

```sh
taf run -- --input sample.fa
```

Build a versioned command:

```sh
taf build
```

The generated command is written under:

```text
target/<command-name>-v<version>-r<release>
```

For example:

```text
target/taf-my-tool-v0.1.0-r1
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

Search the cached online index:

```sh
taf search blast
taf list --online
```

Install the latest indexed version:

```sh
taf install blast
```

Install a specific version:

```sh
taf install blast 2.16.0-r1
```

Install all indexed apps as a dry-run preview, then apply the plan if desired:

```sh
taf install --all
taf install --all --tools --yes
```

Check local Hub installs against the current local index:

```sh
taf update
taf outdated
taf upgrade
taf upgrade --yes
taf prune
taf prune --yes
```

`taf outdated` reports installed apps with newer indexed versions. `taf upgrade`
is dry-run by default and requires `--yes` to modify local installs. `taf prune`
removes older local versions while keeping the newest installed version. Apps
installed with `taf install --from` are treated as local/private installs and
are not silently replaced by public-index upgrades.

Install a private or local TAFFISH app project without publishing it to the
public Hub:

```sh
taf install --from /path/to/my-private-tool
taf list
taf which taf-my-private-tool
```

`taf install --from` reads the local project's `taffish.toml`, checks the
project, copies the working tree into the selected TAFFISH home, builds the
versioned command wrapper, and records the install origin as
`[local-project] <PROJECT-ROOT>`. The path may be the project root or any child
directory under it. It does not require `taf update` and does not auto-install
dependencies.

Command names and fully versioned command names are accepted too:

```sh
taf install taf-blast
taf install taf-blast v2.16.0-r1
taf install taf-blast-v2.16.0-r1
```

Uninstall or locate an app:

```sh
taf uninstall taf-blast
taf which taf-blast-v2.16.0-r1
```

## MCP / AI Integration

`taffish-mcp` is a conservative MCP stdio server for AI clients. It exposes
structured project, app, Hub, config, history, resource, prompt, validation,
compilation, summarization, smoke/trust inspection, and package-maintenance
planning operations. It does not expose `taf run`, `taf publish`, container
execution, or image-building actions.

For MCP compile tools, pass `containerBackend` when backend choice matters. If
that argument is omitted, `TAFFISH_CONTAINER_BACKEND=apptainer|podman|docker`
is used when set; explicit `containerBackend` always has priority.

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

This lets an AI client inspect TAFFISH projects and installed/indexed apps,
search the local index, read project resources, validate or compile `.taf`
source without executing it, preview app/project shell compilation, and prepare
safe project actions without relying first on unstructured terminal text.

For the focused guide to tools, resources, prompts, and safety boundaries, see
[TAFFISH MCP Guide](taffish-mcp.en.md). For Codex, Claude Code, Cursor, Cline,
and generic MCP client configuration examples, see
[Using TAFFISH MCP With AI Clients](mcp-clients.en.md).

## Runtime Config And Mirrors

`taf` provides runtime configuration for mirror and custom source support. The
default config paths are:

```text
user   = ~/.local/share/taffish/config.toml
system = /opt/taffish/config.toml
```

Inspect the effective config:

```sh
taf config --user
taf config path --user
```

Initialize the default GitHub profile:

```sh
taf config init --user --github
```

Initialize the China mirror profile:

```sh
taf config init --user --china --force
taf update --user
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

## Open Source And Source Builds

The Common Lisp implementation is published in
[taffish/taffish](https://github.com/taffish/taffish) under Apache License 2.0.

The source repository builds three command-line entry points:

```text
taf
taffish
taffish-mcp
```

Source builds are mainly for contributors, maintainers, packagers, and users
who need to inspect or modify the implementation. Most users should still use
the prebuilt binaries installed by the standard installer.

Useful source-side documents:

- [Build From Source](https://github.com/taffish/taffish/blob/main/docs/dev/en/build-from-source.md)
- [Source-tree Developer Docs](https://github.com/taffish/taffish/tree/main/docs/dev/en)
- [Contributing](https://github.com/taffish/taffish/blob/main/CONTRIBUTING.md)
- [Security Policy](https://github.com/taffish/taffish/blob/main/SECURITY.md)

Official release assets currently cover selected macOS and Linux platforms;
all platforms supported by SBCL can build from source when the required
dependencies are available. Tagged release payloads include `SHA256SUMS`,
`SHA256SUMS.asc`, and `TAFFISH-RELEASE-KEY.asc` for manual checksum and GPG
signature verification. Consult the source README and the selected GitHub
Release for the authoritative platform matrix, build backend, version notes,
and verification procedure.

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

[smoke]
backend = "docker"
timeout = 60
exist = ["my-tool"]
test = ["my-tool --help"]
```

For containerized apps, `[smoke]` is part of the publishable metadata. `taf
check` validates it and rejects default `TODO` placeholders, while Hub/index
automation runs the actual smoke checks for new versions after the image is
available.

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

For a containerized app, replace the default `[smoke]` placeholders before this
step. `taf check` does not run smoke commands locally; it validates that the Hub
automation has enough metadata to run them later.

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
- The public `taffish` repository is now the open-source local CLI/compiler repository, while this documentation repository remains focused on user, app-author, Hub, and index material.
