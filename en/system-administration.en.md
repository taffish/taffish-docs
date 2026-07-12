# TAFFISH System Administration

[English](system-administration.en.md) | [中文](../zh/system-administration.cn.md)

This guide is for administrators who install TAFFISH on shared workstations,
servers, or HPC login nodes. It covers system scope, permissions, shared shell
integration, container backends, and the boundary between centrally managed
apps and per-user state.

TAFFISH is not a scheduler. A system-installed `taf-*` command can be called
from ordinary shell, Slurm/PBS/LSF scripts, workflow systems, Python, R, or AI
clients, while the surrounding platform remains responsible for job
scheduling and resource policy.

## Table Of Contents

- [Scope Model](#scope-model)
- [Default Layout](#default-layout)
- [Install TAFFISH System-Wide](#install-taffish-system-wide)
- [Initialize A Managed Installation](#initialize-a-managed-installation)
- [Scope-Aware Command Matrix](#scope-aware-command-matrix)
- [Configuration And Mirrors](#configuration-and-mirrors)
- [Mixed System And User Deployments](#mixed-system-and-user-deployments)
- [Command Paths](#command-paths)
- [System-Wide Shell Completion](#system-wide-shell-completion)
- [System-Wide Vim And Neovim](#system-wide-vim-and-neovim)
- [Container Backend Policy](#container-backend-policy)
- [Apptainer On Shared Systems](#apptainer-on-shared-systems)
- [Upgrade TAFFISH And Apps](#upgrade-taffish-and-apps)
- [Backup And Removal](#backup-and-removal)
- [Operational Checklist](#operational-checklist)

## Scope Model

TAFFISH supports two independent state scopes:

| Scope | Purpose | Default home |
| --- | --- | --- |
| `--user` / `-u` | State and apps owned by one user | `~/.local/share/taffish` |
| `--system` / `-s` | Shared state and apps managed for the machine | `/opt/taffish` |

Installing the `taf` binary system-wide does not change the default scope of
later commands. Scope-aware commands still default to user scope on every
invocation. An administrator who runs `taf update` without `--system` updates
root's user index, not the machine's system index.

Use explicit scope in scripts, runbooks, and administrator examples.

## Default Layout

| Item | User scope | System scope |
| --- | --- | --- |
| Core commands | `~/.local/bin` | `/usr/local/bin` |
| Installed app commands | `~/.local/share/taffish/bin` | `/usr/local/bin` |
| TAFFISH home | `~/.local/share/taffish` | `/opt/taffish` |
| Config | `~/.local/share/taffish/config.toml` | `/opt/taffish/config.toml` |
| Cached index | `<home>/index/current.json` | `<home>/index/current.json` |
| Installed app source/metadata | `<home>/apps` | `<home>/apps` |
| Apptainer SIF cache | `<home>/images/sif` | `<home>/images/sif` |

Custom deployments can set `TAFFISH_SYSTEM_HOME` and
`TAFFISH_SYSTEM_BIN_DIR`. Persist the same values for administrators and
users; otherwise `taf --system` and the installed app launchers may resolve
different locations.

## Install TAFFISH System-Wide

GitHub/default installer:

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sudo sh -s -- --system
```

China/Gitee installer:

```sh
curl -fsSL https://gitee.com/taffish-org/taffish/raw/main/install/install-taffish.gitee.sh | sudo sh -s -- --system
```

The installer writes the three core commands to `/usr/local/bin`, creates the
system home, initializes system config when requested, and attempts a system
index update. It copies completion and Vim files, but it does not edit global
shell or editor configuration.

## Initialize A Managed Installation

The installer normally performs these steps. They are also the repair path for
a manual or incomplete deployment:

```sh
sudo taf doctor --init --system
sudo taf config init --system --github
sudo taf update --system
taf doctor --system
```

For the China mirror profile:

```sh
sudo taf config init --system --china --force
sudo taf update --system
```

Install and inspect a shared app:

```sh
taf search --system fastqc
sudo taf install --system fastqc
taf list --system
taf which --system taf-fastqc
taf-fastqc -- --help
```

## Scope-Aware Command Matrix

| Command | Typical system use | Writes system state |
| --- | --- | --- |
| `taf update` | `sudo taf update --system` | yes |
| `taf search` | `taf search --system <QUERY>` | no |
| `taf info` | `taf info --system <APP>` | no |
| `taf install` | `sudo taf install --system <APP>` | yes unless dry-run |
| `taf outdated` | `taf outdated --system` | no |
| `taf upgrade` | `sudo taf upgrade --system --yes` | only when applied |
| `taf prune` | `sudo taf prune --system --yes` | only when applied |
| `taf uninstall` | `sudo taf uninstall --system <APP>` | yes unless dry-run |
| `taf list` | `taf list --system` | no |
| `taf which` | `taf which --system <COMMAND>` | no |
| `taf doctor` | `taf doctor --system` | only with `--init` |
| `taf config` | `taf config --system` | only with `init` |

Project-local commands such as `taf new`, `taf check`, `taf run`, `taf build`,
and `taf publish` are not system package-management commands.

## Configuration And Mirrors

User scope loads configuration in this order:

1. system config;
2. user config;
3. explicit `TAFFISH_CONFIG` file.

Later files override earlier files. This lets an administrator publish a site
default while allowing a user override when local policy permits it. System
scope reads the system config and an optional explicit config, but not the user
config.

Inspect both views when diagnosing a shared machine:

```sh
taf config --system
taf config path --system
taf config --user
taf config path --user
```

The source rewrite mechanism changes app repository clone URLs. It does not
rewrite container registry URLs. A private or regional deployment must ensure
that the selected container images are reachable separately.

## Mixed System And User Deployments

A common deployment installs only the three core commands system-wide, while
each researcher keeps personal apps and indexes in user scope:

```sh
taf doctor --init --user
taf update --user
taf install --user fastqc
```

The user must add both command directories to `PATH`:

```sh
export PATH="$HOME/.local/bin:$HOME/.local/share/taffish/bin:$PATH"
```

Users may also run centrally installed commands from `/usr/local/bin`. To
inspect central install metadata, they must use `taf list --system` or
`taf which --system`; default `taf list` reads their user home.

## Command Paths

System app launchers and the three core commands share `/usr/local/bin` by
default. This makes app commands visible without editing every user's `PATH`.

For user installs, core and app commands are intentionally separated:

```text
~/.local/bin                    # taf, taffish, taffish-mcp
~/.local/share/taffish/bin     # installed taf-* commands
```

`taf doctor --user` and `taf doctor --system` report whether the active command
directory is present in `PATH`.

## System-Wide Shell Completion

Installed completion files are under `/opt/taffish/share/completions`.

For Bash with `bash-completion`:

```sh
sudo install -m 0644 /opt/taffish/share/completions/bash/taf /etc/bash_completion.d/taf
sudo install -m 0644 /opt/taffish/share/completions/bash/taffish /etc/bash_completion.d/taffish
```

Alternatively, source both files from the global interactive Bash startup
file:

```sh
source /opt/taffish/share/completions/bash/taf
source /opt/taffish/share/completions/bash/taffish
```

Debian-family systems commonly use `/etc/bash.bashrc`; Red Hat-family systems
commonly use `/etc/bashrc`.

For Zsh, add the installed directory to global `fpath` before `compinit`:

```zsh
fpath=(/opt/taffish/share/completions/zsh $fpath)
autoload -Uz compinit
compinit
```

For Fish, copy the two `.fish` files into a vendor completion directory such
as `/usr/share/fish/vendor_completions.d` or
`/usr/local/share/fish/vendor_completions.d`. Shell package layouts vary, so
inspect the target system before choosing a global path.

## System-Wide Vim And Neovim

Installed Vim files are under `/opt/taffish/share/vim`. Add that directory to a
global Vim configuration:

```vim
set runtimepath+=/opt/taffish/share/vim
```

Alternatively, copy `syntax/taf.vim` and `ftdetect/taf.vim` into the site's Vim
runtime. Common locations include `/usr/share/vim/vimfiles`,
`/usr/local/share/vim/vimfiles`, and `/etc/xdg/nvim`, but the correct path is
operating-system specific.

## Container Backend Policy

TAFFISH selects an available backend or accepts an explicit backend override.
A site should document which backend is supported:

- Docker uses a daemon and site-level socket permissions. Membership in the
  Docker group is security-sensitive.
- Podman is often per-user and rootless; image stores may not be shared between
  users.
- Apptainer is usually the best fit for HPC and shared Linux systems without a
  user-accessible Docker daemon.

Machine-specific runtime arguments belong in environment variables such as
`TAFFISH_DOCKER_RUN_ARGS`, `TAFFISH_PODMAN_RUN_ARGS`, and
`TAFFISH_APPTAINER_RUN_ARGS`. Scheduler directives and resource requests stay
in the surrounding job script or workflow system.

## Apptainer On Shared Systems

System app launchers can be globally readable even when the system SIF cache is
not writable. Choose one policy explicitly:

1. Per-user cache: each user runs `taf doctor --init --user`.
2. Administrator pre-cache: an administrator runs common versioned app
   commands once so the SIF exists under the system home.
3. Shared writable cache: trusted local users share a sticky directory.

Shared writable cache example:

```sh
sudo mkdir -p /opt/taffish/images/sif
sudo chmod 1777 /opt/taffish/images/sif
```

Use `1777` only when local users are trusted to create cache files. It does not
make container content trusted and may conflict with stricter cluster policy.

## Upgrade TAFFISH And Apps

`taf upgrade` upgrades installed taf-apps only. Upgrade the three core binaries
by re-running the installer with the same scope and a selected release:

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sudo sh -s -- --system --version X.Y.Z
```

Replace `X.Y.Z` with the release version approved for the site.

Recommended app maintenance sequence:

```sh
sudo taf update --system
taf outdated --system
sudo taf upgrade --system --yes
sudo taf prune --system --yes
```

`upgrade --prune-old`, `prune`, and `uninstall` remove TAFFISH app source,
metadata, and launchers. They do not remove Docker, Podman, Apptainer, or SIF
image caches.

## Backup And Removal

Back up a customized `/opt/taffish/config.toml` and any locally managed app or
index data before changing the system home. TAFFISH currently has no dedicated
core uninstaller.

Before removing the core:

1. List and uninstall centrally managed apps with `--system` as appropriate.
2. Preserve configuration, logs, or caches required by local policy.
3. Remove `/usr/local/bin/taf`, `/usr/local/bin/taffish`, and
   `/usr/local/bin/taffish-mcp`.
4. Remove `/opt/taffish` only after confirming that no retained state remains.

Do not remove `/usr/local/bin/taf-*` by a broad wildcard on a machine that may
contain unrelated commands. Use `taf list --system`, `taf which --system`, and
the recorded install metadata to identify TAFFISH-managed launchers.

## Operational Checklist

- [ ] `taf --version` works for ordinary users.
- [ ] `taf doctor --system` reports the intended home and command bin.
- [ ] `taf config --system` shows the intended index and mirror policy.
- [ ] `taf search --system <QUERY>` reads the system index.
- [ ] A test app can be installed with `sudo taf install --system APP`.
- [ ] The app command is visible in ordinary user `PATH`.
- [ ] The selected container backend works for ordinary users.
- [ ] Apptainer cache ownership follows an explicit site policy.
- [ ] Bash/Zsh/Fish completion is enabled only in the shells the site supports.
- [ ] Vim/Neovim integration is installed only where site policy expects it.
- [ ] Core and app upgrade responsibilities are documented separately.
- [ ] Backup and removal paths are documented for the local machine.

For user workflows, continue with the [TAFFISH Quick Start](quick-start.en.md).
For runtime failures, use [TAFFISH Troubleshooting](troubleshooting.en.md).
