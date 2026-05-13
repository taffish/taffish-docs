# TAFFISH Quick Start

[English](quick-start.en.md) | [中文](../zh/quick-start.cn.md)

This guide is for users who want to install and run TAFFISH apps. It does not
assume that you want to write `.taf` scripts or maintain app repositories.

## Table Of Contents

- [What You Install](#what-you-install)
- [Install TAFFISH](#install-taffish)
- [Check The Environment](#check-the-environment)
- [Shell Completion And Vim Files](#shell-completion-and-vim-files)
- [Update The Hub Index](#update-the-hub-index)
- [Find Apps](#find-apps)
- [Install An App](#install-an-app)
- [Run An Installed App](#run-an-installed-app)
- [List, Locate, And Remove Apps](#list-locate-and-remove-apps)
- [Container Backends](#container-backends)
- [Runtime Config And Mirrors](#runtime-config-and-mirrors)
- [Network Notes](#network-notes)
- [Where To Read Next](#where-to-read-next)

## What You Install

TAFFISH provides three local commands:

```text
taffish   compile .taf programs to shell
taf       manage app projects and TAFFISH Hub packages
taffish-mcp
          expose safe tools/resources/prompts plus app/project inspection to AI clients
```

Most users mainly use `taf` and installed `taf-*` app commands. The installer
also copies shell completion files and Vim syntax files into TAFFISH home.

Typical user flow:

```sh
taf update
taf search augustus
taf info augustus
taf install augustus
taf-augustus -- --help
```

## Install TAFFISH

User install, recommended for normal users:

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --user
```

System install, useful on shared servers:

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sudo sh -s -- --system
```

Pinned version install:

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --version 0.8.1 --user
```

TAFFISH `0.8.1` is the current stable patch release in the first open-source
`0.8.x` local CLI/compiler series. Most users should use the installer above,
but source builds and release verification are documented in the
[taffish/taffish README](https://github.com/taffish/taffish).

For users in China, GitHub raw URLs may be slow or blocked. The Gitee installer
downloads files from the Gitee mirror and initializes the China mirror config
when no config exists:

```sh
curl -fsSL https://gitee.com/taffish-org/taffish/raw/main/install/install-taffish.gitee.sh | sh -s -- --user
```

To replace an existing config with the Gitee/China mirror profile:

```sh
curl -fsSL https://gitee.com/taffish-org/taffish/raw/main/install/install-taffish.gitee.sh | sh -s -- --user --force-config
```

After installation, check:

```sh
taf --version
taffish --version
taffish-mcp --version
```

If `taf` is not found, add the user install bin directory to `PATH`:

```sh
export PATH="$HOME/.local/bin:$PATH"
```

Then open a new shell, or run `source ~/.zshrc` / `source ~/.bashrc` after
adding it to your shell profile.

## Check The Environment

Run:

```sh
taf doctor
```

The current installer normally initializes the standard TAFFISH directories
during installation. If this is a manual or repaired install, you can initialize
missing directories with:

```sh
taf doctor --init
```

`taf doctor` checks paths and common executables such as `git`, `docker`,
`podman`, `apptainer`, and `taffish`. Not every optional tool is required for
every use case. For example, if you only use Docker, you do not need Podman.

## Shell Completion And Vim Files

Starting with TAFFISH `0.5.0`, the installer also installs shell completion
files and Vim syntax files under TAFFISH home.

For a user install, completion files are usually under:

```text
~/.local/share/taffish/share/completions
```

Example Bash setup:

```sh
source ~/.local/share/taffish/share/completions/bash/taf
source ~/.local/share/taffish/share/completions/bash/taffish
```

Example Zsh setup:

```sh
fpath=(~/.local/share/taffish/share/completions/zsh $fpath)
autoload -Uz compinit
compinit
```

Vim syntax files are usually under:

```text
~/.local/share/taffish/share/vim
```

## Update The Hub Index

The local `taf` command installs apps from a local copy of the TAFFISH Hub index.
Update it first:

```sh
taf update
```

This downloads the official static index by default and stores it under the
selected TAFFISH home.

If the network is unstable or you want to test another index:

```sh
taf update --url <INDEX-URL>
```

`TAFFISH_INDEX_URL` can also override the default URL.

## Runtime Config And Mirrors

Since TAFFISH `0.2.0`, `taf` includes a small runtime config file for stable
mirror and custom source support. The current public release is `0.8.1`.

Default config paths:

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

The China profile is a template:

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

`[index].url` controls where `taf update` downloads the static index from.
`[[source.rewrite]]` rewrites canonical app repository URLs when `taf install`
clones apps. This also makes internal mirrors possible: mirror the app
repositories and index schema, then edit `config.toml` so `from` points to the
canonical GitHub prefix and `to` points to the internal Git service prefix.

## Find Apps

Search:

```sh
taf search augustus
taf search blast
```

Show app information:

```sh
taf info augustus
taf info taf-augustus
```

List indexed apps from the local index cache:

```sh
taf list --online
```

List locally installed apps:

```sh
taf list
```

## Install An App

Install the latest indexed version:

```sh
taf install augustus
```

Install by command name:

```sh
taf install taf-augustus
```

Install an exact version:

```sh
taf install augustus 3.5.0-r1
taf install taf-augustus-v3.5.0-r1
```

Preview without changing files:

```sh
taf install --dry-run augustus
```

If the selected version declares dependencies, `taf install` installs those
dependencies first.

Install a private or local TAFFISH app project without publishing it to the
public Hub:

```sh
taf install --from /path/to/my-private-tool
taf list
taf which taf-my-private-tool
```

`taf install --from` reads the local project's `taffish.toml`, checks the
project, copies the working tree into the selected TAFFISH home, builds the
versioned command wrapper, and records the origin as
`[local-project] <PROJECT-ROOT>`. The path may be the project root or any child
directory under it. This mode does not require `taf update` and does not
auto-install dependencies.

## Run An Installed App

TAFFISH app commands are normal shell commands. For example:

```sh
taf-augustus -- --help
taf-augustus -- --species=help
taf-augustus -- --species=human genome.fa
```

The `--` separator means "pass the following options to the upstream tool".
This is useful when an option such as `--help` might otherwise be handled by the
TAFFISH wrapper itself.

Do not use a TAFFISH tool command as if it were a shell command runner. For
example:

```sh
taf-augustus echo 1
```

does not execute shell `echo`. It passes `echo 1` to upstream AUGUSTUS.

Installed commands usually include:

```text
taf-<name>                  unversioned alias
taf-<name>-v<version>-r<N>   exact versioned command
```

The unversioned alias points to the latest installed local version.

## List, Locate, And Remove Apps

List local installations:

```sh
taf list
```

Find installed files:

```sh
taf which taf-augustus
taf which taf-augustus-v3.5.0-r1
```

Uninstall:

```sh
taf uninstall taf-augustus
taf uninstall taf-augustus-v3.5.0-r1
```

When more than one local version exists, prefer the exact versioned command.

## Container Backends

Most bioinformatics tool apps run inside containers. TAFFISH can use Docker,
Podman, or Apptainer depending on the app and local environment.

Common checks:

```sh
docker run --rm hello-world
podman run --rm hello-world
apptainer --version
```

You only need the backend you plan to use. On HPC or shared Linux servers,
Apptainer is often the most practical backend. On laptops, Docker or Podman are
often simpler.

For installed `taf-*` commands or direct `taffish` compilation, set
`TAFFISH_CONTAINER_BACKEND=apptainer|podman|docker` to force generic
`<container:...>` tags at runtime:

```sh
TAFFISH_CONTAINER_BACKEND=podman taf-augustus-v3.5.0-r1 -- --help
TAFFISH_CONTAINER_BACKEND=podman taf-augustus-v3.5.0-r1 --compile -- --help
```

This does not override explicit `<docker:...>`, `<podman:...>`, or
`<apptainer:...>` tags. In local project runs, `taf run --backend ...` has
priority over the environment variable.

If a container backend reports errors before the app starts, first check the
backend itself. For example:

```sh
podman system df
podman ps -a
podman machine stop
podman machine start
```

Run containerized apps from a project directory, data directory, or scratch
directory. Avoid running them from host system directories such as `/usr/bin` or
`/usr/local/bin`: TAFFISH mounts the current workdir into the container, and that
mount can hide important directories inside the image.

TAFFISH still uses container images from the locations declared by app packages,
often GHCR. Mirror configuration currently covers index download and app
repository cloning; container image access must still be available from the
machine that runs apps.

## Network Notes

If GitHub is slow or blocked in your environment:

- retry `taf update` later;
- configure your system proxy or Git proxy;
- initialize the China profile or use `taf update --url <INDEX-URL>` for a one-off index override;
- confirm that GHCR images are reachable from the machine that will run apps.

## Where To Read Next

- [What Is TAFFISH](taffish.en.md)
- [TAFFISH App Developer Guide](app-developer-guide.en.md)
- [Containerized App Best Practices](container-apps.en.md)
- [Flow And Dependencies Guide](flow-dependencies.en.md)
- [Troubleshooting](troubleshooting.en.md)
- [Build From Source](https://github.com/taffish/taffish/blob/main/docs/dev/en/build-from-source.md)
