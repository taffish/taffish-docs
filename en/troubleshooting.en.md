# TAFFISH Troubleshooting

[English](troubleshooting.en.md) | [中文](../zh/troubleshooting.cn.md)

This page collects common problems for TAFFISH users and app developers. Start
from the error message, then check the relevant section.

## Table Of Contents

- [`taf` Or An Installed App Command Is Not Found](#taf-or-an-installed-app-command-is-not-found)
- [A System Install Looks Empty](#a-system-install-looks-empty)
- [`taffish` Cannot Find Runtime Libraries On macOS](#taffish-cannot-find-runtime-libraries-on-macos)
- [`taf update` Fails](#taf-update-fails)
- [`taf install` Cannot Find An App](#taf-install-cannot-find-an-app)
- [GitHub Authentication Or Publish Fails](#github-authentication-or-publish-fails)
- [GHCR Image Pull Fails](#ghcr-image-pull-fails)
- [Docker Permission Denied](#docker-permission-denied)
- [Podman Machine Or crun Errors](#podman-machine-or-crun-errors)
- [Apptainer Missing `mksquashfs`](#apptainer-missing-mksquashfs)
- [Apptainer Says `no writable apptainer image directory found`](#apptainer-says-no-writable-apptainer-image-directory-found)
- [Build Backend Rejects Apptainer](#build-backend-rejects-apptainer)
- [`taf check` Rejects `[smoke]`](#taf-check-rejects-smoke)
- [Smoke Command Fails Because Of Nested Quotes](#smoke-command-fails-because-of-nested-quotes)
- [Option Is Handled By The Wrapper Instead Of The Tool](#option-is-handled-by-the-wrapper-instead-of-the-tool)
- [`exec`: Executable File Not Found](#exec-executable-file-not-found)
- [Tool Exists In The Image But Is Not Found](#tool-exists-in-the-image-but-is-not-found)
- [Container Image Exists In One Backend But Not Another](#container-image-exists-in-one-backend-but-not-another)
- [Where To Get More Context](#where-to-get-more-context)

## `taf` Or An Installed App Command Is Not Found

The default user install uses `~/.local/bin` for the three core commands and
`~/.local/share/taffish/bin` for apps installed by `taf install`. Add both to
`PATH`:

```sh
export PATH="$HOME/.local/bin:$HOME/.local/share/taffish/bin:$PATH"
```

Then open a new shell or source your shell profile.

Check:

```sh
which taf
taf --version
taffish --version
taffish-mcp --version
```

If you customized `TAFFISH_USER_HOME`, also add its `bin` subdirectory. System
installs place core and app commands in `/usr/local/bin` by default; check
`taf doctor --system` when a shared command is missing.

## A System Install Looks Empty

Scope-aware commands default to user scope even when `taf` itself came from a
system installation. Use system scope explicitly:

```sh
taf doctor --system
taf config --system
taf list --system
taf which --system APP_OR_COMMAND
taf search --system QUERY
```

System writes normally require administrator permission:

```sh
sudo taf update --system
sudo taf install --system APP
```

See [TAFFISH System Administration](system-administration.en.md) for the full
scope and permissions matrix.

## `taffish` Cannot Find Runtime Libraries On macOS

Current macOS Apple Silicon binaries may require Homebrew `zstd`:

```sh
brew install zstd
```

Then retry:

```sh
taffish --version
taffish-mcp --version
taf doctor
```

## `taf update` Fails

`taf update` downloads the static TAFFISH index. Failures are usually network
or GitHub raw-content access problems.

Try:

```sh
taf update
taf update --url INDEX_URL
```

For persistent mirror use in TAFFISH `0.2.0` or later, inspect and initialize
runtime config:

```sh
taf config --user
taf config path --user
taf config init --user --china --force
taf update --user
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
taf search KEYWORD
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
Since TAFFISH `0.8.1`, only the default first-line placeholders `TODO` and
`TODO: release summary` are rejected, so normal release summaries that contain
the word `todo` are accepted.

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
docker pull ghcr.io/taffish/APP:VERSION-rRELEASE
podman pull ghcr.io/taffish/APP:VERSION-rRELEASE
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

the problem is usually Podman/crun state or a Linux kernel keyring quota, not
the TAFFISH app. The message does not necessarily mean that the filesystem is
full. Podman/crun may create keyrings for containers, and TAFFISH workflows can
start many short-lived containers when they compose several taf apps. That
pattern can expose a low per-user keyring quota.

On Linux, or inside the Linux VM used by Podman Machine on macOS, inspect the
current limits and key usage:

```sh
sysctl kernel.keys.maxkeys kernel.keys.maxbytes
cat /proc/key-users
```

On macOS, enter the Podman VM first:

```sh
podman machine ssh
```

If the key quota is too small for your workload, an administrator can raise it.
These values are examples, not universal defaults:

```sh
sudo sysctl -w kernel.keys.maxkeys=20000
sudo sysctl -w kernel.keys.maxbytes=2000000
```

For a persistent Linux setting:

```sh
sudo tee /etc/sysctl.d/99-taffish-podman-keyring.conf >/dev/null <<'EOF'
kernel.keys.maxkeys = 20000
kernel.keys.maxbytes = 2000000
EOF

sudo sysctl --system
```

Choose limits according to your local policy, user count, and workload. On
shared servers, ask the system administrator before changing kernel sysctl
settings. For background, see the Linux kernel key service documentation and
your Podman/Linux vendor documentation.

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

## Apptainer Says `no writable apptainer image directory found`

This usually happens on a shared machine after an app was installed with
`taf install --system`, but its Apptainer SIF image has not yet been cached in a
directory writable by the current user.

System install makes the app launcher visible to users. It does not
automatically pull every Apptainer SIF image during installation. On first run,
TAFFISH checks for SIF images in:

```text
${TAFFISH_SYSTEM_HOME:-/opt/taffish}/images/sif
${TAFFISH_USER_HOME:-$HOME/.local/share/taffish}/images/sif
```

If no SIF is found, at least one of those directories must be writable so
TAFFISH can pull or create the image.

Common policies:

```sh
# Per-user cache, usually safest for normal users.
taf doctor --init --user
```

```sh
# Administrator pre-cache in the system TAFFISH home.
sudo taf-APP-vVERSION-rRELEASE -- --help
```

```sh
# Shared writable system cache for trusted local users.
sudo mkdir -p /opt/taffish/images/sif
sudo chmod 1777 /opt/taffish/images/sif
```

Mode `1777` is similar to `/tmp`: local users can create files, but cannot
delete files owned by other users. Use it only when the administrator
intentionally allows trusted local users to share a writable SIF cache. On
stricter shared systems, prefer administrator pre-caching or per-user caches.

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

## `taf check` Rejects `[smoke]`

Starting with TAFFISH `0.8.0`, containerized apps must provide real `[smoke]`
metadata. `taf check` validates this metadata but does not run the smoke
commands locally.

Common causes:

- `[smoke]` is missing for a project that declares `[container].image` or
  `[container].dockerfile`.
- `exist` and `test` are both missing or empty.
- The default `TODO` placeholders from `taf new --tool --docker` were not
  replaced.
- `backend` is not one of `docker`, `podman`, or `apptainer`.
- `timeout` is not a positive integer.

Minimal example:

```toml
[smoke]
backend = "docker"
timeout = 60
exist = ["my-tool"]
test = ["my-tool --help"]
```

`backend` and `timeout` can be omitted when their defaults are fine. Keep smoke
checks cheap and deterministic. The public Hub/index automation runs them for
new containerized versions after the image has been pushed; failures appear in
the index build report instead of the main installable index.

## Smoke Command Fails Because Of Nested Quotes

Smoke `test` entries are TOML strings and are executed through `sh -lc` inside
the smoke container. When a command itself needs quotes, prefer single quotes
inside the shell snippet:

```toml
[smoke]
test = ["python -c 'import vina, rdkit, meeko, gemmi, prody'"]
```

TOML basic string escapes such as `\"` are valid, but deeply nested quoting is
harder to review and easier to copy incorrectly. If the index report shows a
Python `SyntaxError` where the code appears to start with an unexpected quote,
check the quoting in `[smoke].test` first.

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

For installed `taf-*` commands or direct `taffish` compilation, `taf run
--backend` is not involved. Set
`TAFFISH_CONTAINER_BACKEND=apptainer|podman|docker` when you need a generic
`<container:...>` tag to use a specific backend:

```sh
TAFFISH_CONTAINER_BACKEND=podman taf-my-tool-v0.1.0-r1 -- --help
```

This still does not override explicit `<docker:...>`, `<podman:...>`, or
`<apptainer:...>` tags.

If the problem is not backend selection but one-off runtime policy, use
backend-specific runtime argument variables instead. For example, on an ARM host
that needs to run an amd64 Docker/Podman image:

```sh
TAFFISH_DOCKER_RUN_ARGS="--platform linux/amd64" taf-my-tool ...
TAFFISH_PODMAN_RUN_ARGS="--platform linux/amd64" taf-my-tool ...
```

Use these variables for local policy only. If an app itself needs backend-specific
flags, such as GPU access for normal operation, the app source should declare
them with `$@[target: args]`.

## Where To Get More Context

Start with the [TAFFISH Quick Start](quick-start.en.md) or, for a shared
deployment, [TAFFISH System Administration](system-administration.en.md).

Useful commands:

```sh
taf doctor
taf config
taf history
taf which APP_OR_COMMAND
taf info APP_OR_COMMAND
```

For app projects:

```sh
taf check
taf compile -- --help
taf run -- --help
```
