# TAFFISH Quick Start

[English](quick-start.en.md) | [中文](../zh/quick-start.cn.md)

This guide is for users who want to install and run TAFFISH apps. It does not
assume that you want to write `.taf` scripts or maintain app repositories.

## Table Of Contents

- [What You Install](#what-you-install)
- [Install TAFFISH](#install-taffish)
- [Check The Environment](#check-the-environment)
- [Update The Hub Index](#update-the-hub-index)
- [Find Apps](#find-apps)
- [Install An App](#install-an-app)
- [Run An Installed App](#run-an-installed-app)
- [List, Locate, And Remove Apps](#list-locate-and-remove-apps)
- [Container Backends](#container-backends)
- [Network And Mirror Notes](#network-and-mirror-notes)
- [Where To Read Next](#where-to-read-next)

## What You Install

TAFFISH provides two local commands:

```text
taffish   compile .taf programs to shell
taf       manage app projects and TAFFISH Hub packages
```

Most users mainly use `taf` and installed `taf-*` app commands.

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
curl -fsSL https://github.com/taffish/taffish/releases/latest/download/install-taffish.sh | sh -s -- --user
```

System install, useful on shared servers:

```sh
curl -fsSL https://github.com/taffish/taffish/releases/latest/download/install-taffish.sh | sudo sh -s -- --system
```

After installation, check:

```sh
taf --version
taffish --version
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

If this is a fresh user install, you can initialize missing directories with:

```sh
taf doctor --init
```

`taf doctor` checks paths and common executables such as `git`, `docker`,
`podman`, `apptainer`, and `taffish`. Not every optional tool is required for
every use case. For example, if you only use Docker, you do not need Podman.

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

If a container backend reports errors before the app starts, first check the
backend itself. For example:

```sh
podman system df
podman ps -a
podman machine stop
podman machine start
```

## Network And Mirror Notes

TAFFISH currently uses GitHub, GitHub Releases, GitHub raw content, and GHCR.
If GitHub is slow or blocked in your environment:

- retry `taf update` later;
- configure your system proxy or Git proxy;
- use `taf update --url <INDEX-URL>` when a mirror index is available;
- confirm that GHCR images are reachable from the machine that will run apps.

The index model is static, so it can be mirrored in the future without changing
the local `taf` workflow.

## Where To Read Next

- [What Is TAFFISH](taffish.en.md)
- [TAFFISH App Developer Guide](app-developer-guide.en.md)
- [Containerized App Best Practices](container-apps.en.md)
- [Flow And Dependencies Guide](flow-dependencies.en.md)
- [Troubleshooting](troubleshooting.en.md)
