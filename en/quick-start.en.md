# TAFFISH Quick Start

[English](quick-start.en.md) | [中文](../zh/quick-start.cn.md)

[Install TAFFISH](#install-taffish) and use it to run your first app, Augustus.
You do not need to write a `.taf` script. You will need Git to install the app
and one working container backend (Docker, Podman, or Apptainer) to run it.

**Version note:** these instructions cover the released TAFFISH 0.12.0, which
the online installers below provide. If you already have the package, use the
[local and offline instructions](#local-and-offline-installation).

## Table Of Contents

- [What You Install](#what-you-install)
- [Install TAFFISH](#install-taffish)
  - [Set your command path](#set-your-command-path)
  - [Run Augustus for the first time](#run-augustus-for-the-first-time)
  - [Other installation options](#other-installation-options)
  - [Local and offline installation](#local-and-offline-installation)
- [Choose User Or System Scope](#choose-user-or-system-scope)
- [Check The Environment](#check-the-environment)
- [Update The Hub Index](#update-the-hub-index)
- [Find Apps](#find-apps)
- [Install An App](#install-an-app)
- [Run An Installed App](#run-an-installed-app)
- [Maintain Installed Apps](#maintain-installed-apps)
- [List, Locate, And Remove Apps](#list-locate-and-remove-apps)
- [Container Backends](#container-backends)
- [Runtime Config And Mirrors](#runtime-config-and-mirrors)
- [Shell Completion And Vim Files](#shell-completion-and-vim-files)
- [Network Notes](#network-notes)
- [Explore Flow Routes](#explore-flow-routes)
- [Where To Read Next](#where-to-read-next)

## What You Install

Use `taf` to find and install apps from TAFFISH Hub. Each installed app has
its own shell command, such as `taf-augustus`. The installer also provides
`taffish`, which compiles `.taf` scripts to shell, and `taffish-mcp` for AI
clients. You will not need to call either one to run Augustus.

## Install TAFFISH

Install for your own account (no `sudo` required). For a shared server, a
pinned release, or the Gitee mirror, see [other installation options](#other-installation-options).

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --user
```

### Set your command path

For a user install, add both the core and app command directories to the current
shell's `PATH`. The installer does not edit your shell configuration.

```sh
export PATH="$HOME/.local/bin:$HOME/.local/share/taffish/bin:$PATH"
```

This `export` affects the current shell and its child processes; it does not
save the setting for future terminals. Add the same line to `~/.zshrc` (Zsh)
or `~/.bashrc` (interactive Bash). For a Bash login shell, put it in
`~/.bash_profile` instead, or have that file load `~/.bashrc`.
Open a new terminal and check which versions you are using:

```sh
taf --version
taffish --version
taffish-mcp --version
```

The app commands below also work with 0.11.0. Installer
options differ by version; check the selected installer's `--help`.

### Run Augustus for the first time

Refresh your local copy of the Hub index, find Augustus, inspect its entry,
and install the recommended version:

```sh
taf update --user
taf search --user augustus
taf info --user augustus
taf install --user augustus
```

From a working directory where you normally keep analysis files, run:

```sh
taf-augustus -- --help
```

You should see **AUGUSTUS's own help**. The first `--` passes `--help` through
the wrapper to the tool inside the container. If the image is not cached, the
backend needs access to the container registry. See [Container Backends](#container-backends)
if the container does not start.

### Other installation options

System install, useful on shared servers:

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sudo sh -s -- --system
```

If a shared system install will run apps with Apptainer, remember that the
installed `taf-*` commands can be visible to all users while first-run SIF image
caching still needs a writable image directory. See
[Apptainer SIF cache permissions](troubleshooting.en.md#apptainer-says-no-writable-apptainer-image-directory-found)
before deploying TAFFISH for multiple users.

To install 0.12.0 specifically, use its tagged installer.
This command requires `gpg` and `gpgv`:

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/v0.12.0/install/install-taffish.sh | sh -s -- --version 0.12.0 --user --verify gpg
```

To select a different release with the default installer, pass `--version X.Y.Z`.

For release downloads, source builds, and verification, see the
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

For **0.11.0 only**, if Gitee returns HTTP 403 for a macOS Apple Silicon binary
when you are not signed in, download the [v0.11.0 archive](https://gitee.com/taffish-org/taffish/repository/archive/v0.11.0.tar.gz)
as `taffish-v0.11.0.tar.gz` and the [matching installer](https://gitee.com/taffish-org/taffish/raw/v0.11.0/install/install-taffish.gitee.sh)
into the same directory. With GPG available, run:

```sh
sh install-taffish.gitee.sh --user --archive taffish-v0.11.0.tar.gz --verify gpg
```

After a user install, return to [Set your command path](#set-your-command-path).

<a id="install-a-local-0120-candidate"></a>

### Local and offline installation

The official 0.12.0 packages contain `taf`, `taffish`, and `taffish-mcp` together.
They include Han Lisp 0.3.0 and their Lisp runtime; you do not need a separate
Lisp installation.

| Platform | Complete package | Extraction tool |
| --- | --- | --- |
| Apple Silicon, macOS 15 or later | `taffish-0.12.0-darwin-arm64.tar.xz` | System `/usr/bin/tar` |
| glibc-based x86-64 Linux | `taffish-0.12.0-linux-amd64.tar.gz` | `tar` and `gzip` |

The macOS package no longer needs Homebrew or `zstd`. When upgrading from
0.11.0, keep the old `zstd` installed until the upgrade finishes: the installer
runs the old commands to identify them before replacement.

Obtain the matching 0.12.0 installer and complete package through a trusted
channel. Put `install-taffish.sh`, the package, `ARCHIVE-SHA256SUMS`,
`ARCHIVE-SHA256SUMS.asc`, and `TAFFISH-RELEASE-KEY.asc` in one directory.
With `gpg` and `gpgv` available, run from that directory:

```sh
sh install-taffish.sh --local --version 0.12.0 --user --verify gpg
```

`--local` selects the host's package without downloading it and skips the
post-install index update. `--verify gpg` requires valid signatures for the
archive and program manifests, then checks their hashes before extraction and
installation respectively. It does not authenticate the installer script itself.
After installation, set your `PATH` and explicitly run `taf update --user`
when you want to obtain the Hub index.

On Linux, installation displays the bundled runtime terms and asks for
acceptance. Read them before installing; unattended installations must use
`--accept-runtime-license`. See the [administrator instructions](system-administration.en.md#local-installation-and-runtime-terms).
macOS does not require runtime acceptance.

The 0.12.0 Gitee installer automatically joins the macOS package's small
`.part-*` files before checking the complete archive. GitHub and Linux use
complete archives. Local installation also needs a complete archive, not a
`.parts` list.
For renamed packages, manual reassembly and verification details, see the
[installation guide](https://github.com/taffish/taffish/blob/v0.12.0/install/README.md).
After setting `PATH`, check that `taf --version`, `taffish --version`, and
`taffish-mcp --version` all report 0.12.0.

## Choose User Or System Scope

The rest of this Quick Start uses user scope. Scope-aware `taf` commands
default to `--user`, even if the core binary was installed system-wide.

Administrators must use `--system` consistently when managing shared state:

```sh
sudo taf update --system
taf search --system fastqc
sudo taf install --system fastqc
taf list --system
```

Read [TAFFISH System Administration](system-administration.en.md) before
deploying shared apps, global shell/editor integration, or an HPC backend.

## Check The Environment

Run:

```sh
taf doctor --user
```

The current installer normally initializes the standard TAFFISH directories
during installation. If this is a manual or repaired install, you can initialize
missing directories with:

```sh
taf doctor --init --user
```

`taf doctor` checks paths and common executables such as `git`, `docker`,
`podman`, `apptainer`, and `taffish`. Not every optional tool is required for
every use case. For example, if you only use Docker, you do not need Podman.

In 0.11.0, `taf doctor` is read-only unless you explicitly request `--init`.
Use `taf doctor index` for a cached index or `taf doctor hub augustus` for an
installed app. To inspect the CLI language, run `taf config language`; to try
English just once, run `TAFFISH_LANGUAGE=en taf --help`. English and Chinese
are built in; the mirror choice does not choose your language.

## Update The Hub Index

The local `taf` command installs apps from a local copy of the TAFFISH Hub index.
Update it first:

```sh
taf update --user
```

This downloads the official static index by default and stores it under the
selected TAFFISH home.

If the network is unstable or you want to test another index:

```sh
taf update --url INDEX_URL
```

`TAFFISH_INDEX_URL` can also override the default URL.

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

Install the version recommended by the cached index:

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

A leading `--` stops the wrapper from handling options such as `--help`. In the
three examples above, the container runs `augustus` with the options shown.
The last example assumes you have a FASTA file named `genome.fa` in your working directory.

This app also supports command mode: `taf-augustus echo 1` runs `echo 1` inside
the container. It does not run `echo` on the host or pass `echo 1` to AUGUSTUS.

Installed commands usually include:

```text
taf-<name>                  unversioned alias
taf-<name>-v<version>-r<N>   exact versioned command
```

The unversioned alias points to the newest **locally installed** version of
that command, which may differ from the index recommendation.

## Maintain Installed Apps

These commands maintain apps installed from the Hub index. They do not upgrade
the `taf`, `taffish`, or `taffish-mcp` core binaries. Upgrade the core by
re-running the installer with the same scope and a selected `--version`.

Install every indexed app as a preview first:

```sh
taf install --all
taf install --all --tools
taf install --all --flows
```

`taf install --all` is dry-run by default. Add `--yes` to actually install the
planned set:

```sh
taf install --all --tools --yes
```

Check whether locally installed Hub apps have newer indexed versions:

```sh
taf update
taf outdated
taf outdated --json
```

Plan or apply upgrades:

```sh
taf upgrade
taf upgrade --yes
taf upgrade taf-augustus --yes --prune-old
```

`taf upgrade` is also dry-run by default. It compares local install metadata
with the current local index, so run `taf update` first. Local/private apps
installed with `taf install --from` are not silently replaced by public-index
upgrades.

In 0.11.0, batch install and upgrade target the local index recommendation
(`latest`), not the most recently published tag, and do not automatically
downgrade. Prune requires that same-source recommendation to be installed and
verified; only reliably older versions are removal candidates. If the target
is absent or the installation evidence is inconclusive, prune protects the
affected package instead of guessing. Preview before confirming:

```sh
taf prune
taf prune --yes
taf prune --kind flow --yes
```

`taf uninstall`, `taf upgrade --prune-old`, and `taf prune` remove TAFFISH app
source/wrapper files only. They do not remove shared Docker, Podman, Apptainer,
or SIF image caches.

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

For system-wide installs with Apptainer, app commands and SIF image caches have
different permission requirements. If normal users see
`no writable apptainer image directory found`, use the
[Apptainer SIF cache FAQ](troubleshooting.en.md#apptainer-says-no-writable-apptainer-image-directory-found).

For installed `taf-*` commands or direct `taffish` compilation, set
`TAFFISH_CONTAINER_BACKEND=apptainer|podman|docker` to force generic
`<container:...>` tags at runtime:

```sh
TAFFISH_CONTAINER_BACKEND=podman taf-augustus-v3.5.0-r1 -- --help
TAFFISH_CONTAINER_BACKEND=podman taf-augustus-v3.5.0-r1 --compile --help
```

This does not override explicit `<docker:...>`, `<podman:...>`, or
`<apptainer:...>` tags. In local project runs, `taf run --backend ...` has
priority over the environment variable.

TAFFISH `0.9.0` also supports backend-specific container runtime arguments. Use
structured `.taf` tag arguments for app requirements, such as an app that needs
GPU access:

```taf
<container:ghcr.io/taffish/my-gpu-tool:1.0.0-r1$@[docker: --gpus all][podman: --device nvidia.com/gpu=all][apptainer: --nv]>
  my-gpu-tool --help
```

Use runtime environment variables for machine, site, or one-off differences
that are not part of the app implementation:

```sh
TAFFISH_DOCKER_RUN_ARGS="--platform linux/amd64" taf-my-tool ...
TAFFISH_PODMAN_RUN_ARGS="--platform linux/amd64" taf-my-tool ...
TAFFISH_APPTAINER_RUN_ARGS="--bind /scratch:/scratch" taf-my-tool ...
```

In short: put app-specific runtime needs in `.taf`; put local execution policy
in environment variables.

If a container backend reports errors before the app starts, first check the
backend itself. For example:

```sh
podman system df
podman ps -a
podman machine stop
podman machine start
```

If Podman reports `crun: create keyring ... Disk quota exceeded`, see
[Podman Machine Or crun Errors](troubleshooting.en.md#podman-machine-or-crun-errors).
It may be a Linux keyring quota limit rather than filesystem disk space.

Run containerized apps from a project directory, data directory, or scratch
directory. Avoid running them from host system directories such as `/usr/bin` or
`/usr/local/bin`: TAFFISH mounts the current workdir into the container, and that
mount can hide important directories inside the image.

TAFFISH still uses container images from the locations declared by app packages,
often GHCR. Mirror configuration currently covers index download and app
repository cloning; container image access must still be available from the
machine that runs apps.

## Runtime Config And Mirrors

`taf` includes a small runtime config file for stable mirror and custom source
support.

Default config paths:

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

## Shell Completion And Vim Files

The installer copies shell completion files and Vim syntax files under TAFFISH
home, but it does not activate them automatically.

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

Example Fish setup:

```sh
mkdir -p ~/.config/fish/completions
cp ~/.local/share/taffish/share/completions/fish/taf.fish ~/.config/fish/completions/
cp ~/.local/share/taffish/share/completions/fish/taffish.fish ~/.config/fish/completions/
```

Vim syntax files are usually under:

```text
~/.local/share/taffish/share/vim
```

Copy `syntax/taf.vim` and `ftdetect/taf.vim` into the corresponding Vim or
Neovim user runtime. For system-wide activation, use the
[System Administration Guide](system-administration.en.md).

## Network Notes

If GitHub is slow or blocked in your environment:

- retry `taf update` later;
- configure your system proxy or Git proxy;
- initialize the China profile or use `taf update --url <INDEX-URL>` for a one-off index override;
- confirm that GHCR images are reachable from the machine that will run apps.

## Explore Flow Routes

To see TAFFISH apps used as curated analysis routes, open the
[TAFFISH Flow Portal](https://taffish.github.io/flows/). It links official flow
families, route pages, Hub entries, and example reports.

- [TAFFISH Flow Portal](https://taffish.github.io/flows/)
- [RNA-seq Flow Family portal](https://taffish.github.io/rnaseq-flows/)
- [Yeast standard report](https://taffish.github.io/rnaseq-flows/examples/yeast-standard-report/04_reports/rnaseq_report.html)
- [Report interpretation guide](https://taffish.github.io/rnaseq-flows/examples/yeast-standard-report/04_reports/report_interpretation.html)

The RNA-seq site remains the most complete public flow-family case study. It
covers reference preparation, Salmon-based expression quantification,
differential expression, enrichment, optional alignment/count/QC evidence,
de novo mode, and final reporting. These pages demonstrate command-level
reproducibility composed from installed `taf-*` commands, not a replacement for
workflow engines.

## Where To Read Next

- [What Is TAFFISH](taffish.en.md)
- [TAFFISH System Administration](system-administration.en.md)
- [TAFFISH App Developer Guide](app-developer-guide.en.md)
- [Containerized App Best Practices](container-apps.en.md)
- [Flow And Dependencies Guide](flow-dependencies.en.md)
- [Troubleshooting](troubleshooting.en.md)
- [Build From Source](https://github.com/taffish/taffish/blob/main/docs/dev/en/build-from-source.md)
