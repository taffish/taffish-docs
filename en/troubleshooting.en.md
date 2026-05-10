# TAFFISH Troubleshooting

[English](troubleshooting.en.md) | [中文](../zh/troubleshooting.cn.md)

This page collects common problems for TAFFISH users and app developers. Start
from the error message, then check the relevant section.

## Table Of Contents

- [`taf: command not found`](#taf-command-not-found)
- [`taffish` Cannot Find Runtime Libraries On macOS](#taffish-cannot-find-runtime-libraries-on-macos)
- [`taf update` Fails](#taf-update-fails)
- [`taf install` Cannot Find An App](#taf-install-cannot-find-an-app)
- [GitHub Authentication Or Publish Fails](#github-authentication-or-publish-fails)
- [GHCR Image Pull Fails](#ghcr-image-pull-fails)
- [Docker Permission Denied](#docker-permission-denied)
- [Podman Machine Or crun Errors](#podman-machine-or-crun-errors)
- [Apptainer Missing `mksquashfs`](#apptainer-missing-mksquashfs)
- [Build Backend Rejects Apptainer](#build-backend-rejects-apptainer)
- [Option Is Handled By The Wrapper Instead Of The Tool](#option-is-handled-by-the-wrapper-instead-of-the-tool)
- [`exec`: Executable File Not Found](#exec-executable-file-not-found)
- [Tool Exists In The Image But Is Not Found](#tool-exists-in-the-image-but-is-not-found)
- [Container Image Exists In One Backend But Not Another](#container-image-exists-in-one-backend-but-not-another)
- [Where To Get More Context](#where-to-get-more-context)

## `taf: command not found`

The install bin directory is not in `PATH`.

For user install:

```sh
export PATH="$HOME/.local/bin:$PATH"
```

Then open a new shell or source your shell profile.

Check:

```sh
which taf
taf --version
```

## `taffish` Cannot Find Runtime Libraries On macOS

Current macOS Apple Silicon binaries may require Homebrew `zstd`:

```sh
brew install zstd
```

Then retry:

```sh
taffish --version
taf doctor
```

## `taf update` Fails

`taf update` downloads the static TAFFISH index. Failures are usually network
or GitHub raw-content access problems.

Try:

```sh
taf update
taf update --url <INDEX-URL>
```

For persistent mirror use in TAFFISH `0.2.0` or later, inspect and initialize
runtime config:

```sh
taf config
taf config path
taf config init --china --force
taf update
```

The generated China profile changes the index URL and rewrites canonical GitHub
app repository URLs to the configured mirror when `taf install` clones apps.
For an internal Git service, edit `config.toml` and point `[index].url` and
`[[source.rewrite]].to` at the internal mirror.

If your network requires a proxy, configure your shell, Git, or system proxy
outside TAFFISH.

Mirror config does not automatically mirror container registries. If an app
uses GHCR, the machine running the app still needs access to that image source.

The installer may warn if `taf update` fails, but it does not roll back the
installed binaries.

## `taf install` Cannot Find An App

First update the local index:

```sh
taf update
```

Then search:

```sh
taf search <keyword>
taf list --online
```

If a repository was just published, wait for the `taffish-index` automation to
run, or run the index automation manually if you maintain the index.

Check accepted install forms:

```sh
taf install augustus
taf install taf-augustus
taf install augustus 3.5.0-r1
taf install taf-augustus-v3.5.0-r1
```

## GitHub Authentication Or Publish Fails

`taf publish` uses Git and GitHub but does not manage login itself.

Check:

```sh
git remote -v
git status
gh auth status
```

If remote Git operations should ask for credentials, use:

```sh
taf publish --release --dry-run --prompt
taf publish --release --yes --prompt
```

If the repository does not exist and you want `taf` to create it:

```sh
taf publish --release --yes --build --create-repo --public
```

For real releases, prefer the release-note workflow:

```sh
taf publish --release --dry-run
taf publish --release --yes --build
```

`taf publish --release` reads the ignored `release.md` draft. If the GitHub
Release title or notes look wrong, edit `release.md`: the first line becomes the
publish message, and the whole file becomes the GitHub Release notes.

## GHCR Image Pull Fails

The app may install successfully but fail at runtime if its image cannot be
pulled.

Check:

- the image name in `taffish.toml`;
- the image tag matches `<version>-r<release>`;
- the GHCR package is public;
- the runtime machine can access `ghcr.io`.

Manual test:

```sh
docker pull ghcr.io/taffish/<app>:<version>-r<release>
podman pull ghcr.io/taffish/<app>:<version>-r<release>
```

## Docker Permission Denied

On Linux, add your user to the `docker` group:

```sh
sudo usermod -aG docker "$USER"
```

Then log out and log back in.

Test:

```sh
docker run --rm hello-world
```

## Podman Machine Or crun Errors

On macOS, Podman runs through a Linux VM. If Podman is not reachable:

```sh
podman machine list
podman machine start
```

If you see an error like:

```text
crun: create keyring ... Disk quota exceeded
```

the problem is usually Podman/crun state or resource quota, not the TAFFISH app.

Check and clean carefully:

```sh
podman system df
podman ps -a
podman container prune
podman machine stop
podman machine start
```

More aggressive cleanup:

```sh
podman system prune
```

Only if you are sure unused images and volumes can be removed:

```sh
podman system prune -a --volumes
```

## Apptainer Missing `mksquashfs`

When Apptainer converts Docker/OCI images to SIF, it may need `mksquashfs`.

Install:

```sh
sudo apt-get install -y squashfs-tools
```

If Apptainer repeatedly converts SIF to a temporary sandbox, install:

```sh
sudo apt-get install -y squashfuse
```

Optional packages that remove common warnings:

```sh
sudo apt-get install -y fuse2fs gocryptfs
```

## Build Backend Rejects Apptainer

`taf run --backend` can use:

```text
docker, podman, apptainer
```

But `taf build --image --backend` currently accepts image build backends:

```text
docker, podman
```

So this is valid:

```sh
taf build --image --backend docker
taf run --backend docker -- --help
```

This is not valid for image building:

```sh
taf build --image --backend apptainer
```

## Option Is Handled By The Wrapper Instead Of The Tool

Use `--` before upstream options:

```sh
taf-my-tool -- --help
taf run -- --help
```

Without `--`, options such as `--help`, `--version`, or `--compile` may be
handled by the TAFFISH wrapper.

## `exec`: Executable File Not Found

Do not write this in a simple container wrapper:

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
exec my-tool ::*ARGV*::
```

Depending on the generated runtime, `exec` may be passed to the container as the
executable name. Since `exec` is a shell builtin, the container runtime can fail:

```text
exec: "exec": executable file not found in $PATH
```

Use:

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool ::*ARGV*::
```

## Tool Exists In The Image But Is Not Found

Symptom:

```text
my-tool: executable file not found in $PATH
```

or, for a concrete app:

```text
augustus: executable file not found in $PATH
```

If the image is correct and the tool is installed inside the image, check the
host current directory. TAFFISH container tags normally bind-mount the current
workdir into the same path inside the container and set that path as the
container workdir. This is useful for input and output files, but it can hide
important image directories when the host path has the same name.

Risky host working directories include:

```text
/bin
/usr/bin
/usr/local/bin
/lib
/usr/lib
/opt
```

For example, if the image installs `augustus` under `/usr/bin`, and you run
`taf-augustus` while the host current directory is `/usr/bin`, the bind mount may
cover the container's original `/usr/bin`. The container then starts, but the
tool that used to be in `/usr/bin` is hidden.

Fix:

```sh
mkdir -p ~/work/taffish-runs/augustus-test
cd ~/work/taffish-runs/augustus-test
taf-augustus -- --help
```

As a rule, run TAFFISH apps from a project directory, data directory, or scratch
directory. Avoid running them from system directories that are also meaningful
inside containers.

## Container Image Exists In One Backend But Not Another

If you build an image with Docker and run with Podman, Podman may not see the
Docker image.

During local development, use the same backend:

```sh
taf build --image --backend docker
taf run --backend docker -- --help
```

or:

```sh
taf build --image --backend podman
taf run --backend podman -- --help
```

If an app source explicitly uses `<taf-app:docker:...>` or
`<taf-app:podman:...>`, `taf run --backend` does not override that explicit tag.

## Where To Get More Context

Useful commands:

```sh
taf doctor
taf config
taf history
taf which <app-or-command>
taf info <app-or-command>
```

For app projects:

```sh
taf check
taf compile -- --help
taf run -- --help
```
