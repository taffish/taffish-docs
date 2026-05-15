# Containerized App Best Practices

Containerized apps are the recommended way to package tools with complex system dependencies, compiled binaries, or bioinformatics runtime environments. In TAFFISH, the app repository owns the Dockerfile and image build workflow. TAFFISH Hub only indexes metadata and releases.

This is the focused container practice guide. Exact field semantics live in the
[`taffish.toml` Specification](taffish-toml-spec.en.md). The end-to-end app
lifecycle lives in the [TAFFISH App Developer Guide](app-developer-guide.en.md).
Official Hub final review checklists live in the
[Official taf-app Curation Guide](taf-app-curation-guide.en.md).

## Table Of Contents

- [When To Use Containers](#when-to-use-containers)
- [Project Structure](#project-structure)
- [Image Naming And Tags](#image-naming-and-tags)
- [`taffish.toml`](#taffishtoml)
- [`src/main.taf`](#srcmaintaf)
- [Smoke Metadata](#smoke-metadata)
- [Runtime Mounts And Working Directory](#runtime-mounts-and-working-directory)
- [Dockerfile Advice](#dockerfile-advice)
- [Multi-Stage Builds](#multi-stage-builds)
- [APT Hygiene](#apt-hygiene)
- [Environment Variables](#environment-variables)
- [GitHub Actions](#github-actions)
- [GHCR Visibility](#ghcr-visibility)
- [Local Testing](#local-testing)
- [Common Issues](#common-issues)

## When To Use Containers

Use a container when:

- The upstream tool needs many system packages.
- The tool needs compilation.
- Runtime behavior depends on a specific Linux distribution.
- The tool is difficult to install reproducibly on user machines.
- You want users to run one stable taf command without managing the tool environment.

Avoid containers when:

- The app is a pure flow that only calls other taf apps.
- The tool is already a small portable script.
- The wrapper only needs shell features available on ordinary systems.

## Project Structure

A containerized tool app usually looks like:

```text
my-tool/
  taffish.toml
  src/
    main.taf
  docs/
    help.md
  docker/
    Dockerfile
  .github/
    workflows/
      build-image.yml
```

## Image Naming And Tags

Recommended image format:

```text
ghcr.io/taffish/<app-name>:<version>-r<release>
```

Example:

```text
ghcr.io/taffish/blast:2.16.0-r1
```

Principles:

- The image tag should match the TAFFISH version id.
- Do not use `latest` as the official app reference.
- Do not overwrite published tags.
- If the Dockerfile is fixed but the upstream version is unchanged, increase the release, for example `2.16.0-r2`.

## `taffish.toml`

Container metadata:

```toml
[container]
image = "ghcr.io/taffish/my-tool:0.1.0-r1"
dockerfile = "docker/Dockerfile"
build_platforms = "linux/amd64,linux/arm64"
```

Rules:

- `image` should match the app version id: `<version>-r<release>`.
- `dockerfile` is a project-relative path.
- `build_platforms` is a comma-separated list.
- Use `linux/amd64` as the baseline platform unless there is a strong reason not to.
- Add `linux/arm64` only when the upstream tool actually supports it.
- Containerized apps must also provide `[smoke]` metadata before they can pass `taf check` and public Hub indexing.

## `src/main.taf`

Generic container backend:

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --input ::input::
```

Explicit backend:

```taf
<taf-app:docker:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --input ::input::
```

Prefer the generic `container` tag for published apps when the image is portable across supported backends. Use explicit `docker` or `podman` tags when a tool or local development setup depends on one backend.

## Smoke Metadata

TAFFISH `0.8.0` adds `[smoke]` metadata for containerized apps. It describes
cheap checks that prove the final image contains the expected executable and can
run a minimal command.

```toml
[smoke]
backend = "docker"
timeout = 60
exist = ["my-tool"]
test = ["my-tool --help"]
```

For commands that need nested quoting, keep the outer TOML string in double
quotes and prefer single quotes inside the shell snippet:

```toml
test = ["python -c 'import vina, rdkit, meeko, gemmi, prody'"]
```

TOML escapes such as `\"` are valid, but the single-quote style is usually less
fragile because smoke `test` commands are executed through `sh -lc` inside the
container.

Guidelines:

- Keep smoke checks short and deterministic.
- Use `exist` for command availability checks.
- Use `test` for minimal commands such as `tool --help`, `tool -h`, or
  `tool --version`.
- Do not require network access, large reference data, or user credentials.
- Replace every default `TODO` placeholder before `taf check` or `taf publish`.

`taf check` validates the metadata but does not run the container. The public
Hub/index automation runs smoke checks for new containerized versions after the
image has been pushed, records digest/platform/smoke metadata, and writes
failures to index reports instead of exposing failed new versions to users.

## Runtime Mounts And Working Directory

Containerized TAFFISH apps are not isolated from the user's working directory.
At runtime, TAFFISH normally:

- sets the container working directory to the current TAFFISH workdir;
- bind-mounts the user's home directory into the same path inside the container;
- bind-mounts the current workdir into the same path inside the container;
- passes common environment variables such as `HOME` and `USER`.

In simplified Docker/Podman form, this looks like:

```sh
podman run --rm -i \
  -w "$PWD" \
  -v "$HOME:$HOME" \
  -v "$PWD:$PWD" \
  -e "HOME=$HOME" \
  -e "USER=$USER" \
  ghcr.io/taffish/my-tool:0.1.0-r1 \
  my-tool ::*ARGV*::
```

This makes local input and output paths work naturally, but it also means that a
host directory can hide an image directory with the same path. Avoid running
containerized taf apps from host system paths that are important inside images,
especially:

```text
/bin
/usr/bin
/usr/local/bin
/lib
/usr/lib
/opt
```

For example, if a container image installs `augustus` under `/usr/bin`, and the
user runs `taf-augustus` while the host current directory is also `/usr/bin`, the
bind mount may hide the container's original `/usr/bin`. The result can look
like:

```text
augustus: executable file not found in $PATH
```

Run TAFFISH apps from a project directory, data directory, or scratch directory
instead, such as:

```sh
mkdir -p ~/work/taffish-runs/augustus-test
cd ~/work/taffish-runs/augustus-test
taf-augustus -- --help
```

## Dockerfile Advice

Start with a stable base image:

```dockerfile
FROM debian:12
```

Use `--no-install-recommends`:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
      build-essential \
    && rm -rf /var/lib/apt/lists/*
```

Recommended practices:

- Pin the upstream version being downloaded.
- Use a clear working directory.
- Clean package manager caches.
- Remove source archives and temporary build files.
- Prefer `/usr/local/bin` for installed commands.
- Keep `PATH` changes minimal.
- Add labels for maintainership and source.

Avoid:

- Running `apt-get upgrade` unless there is a specific reason.
- Keeping compilers in the final image when they are not needed at runtime.
- Downloading from a moving branch such as `main` in a release image.
- Storing large test data in the image.

## Multi-Stage Builds

For compiled tools, multi-stage builds usually make the final image much smaller:

```dockerfile
FROM debian:12 AS build

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
      build-essential \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build
RUN curl -L -o tool.tar.gz https://example.org/tool-1.0.0.tar.gz \
    && tar -xzf tool.tar.gz \
    && cd tool-1.0.0 \
    && make

FROM debian:12

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY --from=build /build/tool-1.0.0/bin/tool /usr/local/bin/tool

ENV TAFFISH_ENV=TAFFISH
ENV TAFFISH_NAME=tool
```

The compiler, source tree, and intermediate files stay in the build stage.

## APT Hygiene

Use one `RUN` layer for update, install, and cleanup:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
      curl \
      perl \
    && rm -rf /var/lib/apt/lists/*
```

If the tool requires runtime libraries, install only those in the final stage.

Example:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
      libgomp1 \
      zlib1g \
    && rm -rf /var/lib/apt/lists/*
```

## Environment Variables

TAFFISH images often set:

```dockerfile
ENV TAFFISH_ENV=TAFFISH
ENV TAFFISH_NAME=my-tool
```

For a tool installed under a custom directory:

```dockerfile
ENV PATH=/opt/my-tool/bin:$PATH
```

Avoid setting environment variables that unexpectedly override user behavior, such as global thread counts, temporary directories, or language-specific runtime paths, unless the tool needs them.

## GitHub Actions

`taf new --tool --docker` generates an image build workflow. The general workflow is:

1. Checkout repository.
2. Read image and Dockerfile metadata from `taffish.toml`.
3. Login to GHCR.
4. Use buildx for multi-platform builds.
5. Push the image to `ghcr.io/taffish/...`.

The app repository owns this automation. Hub does not build images for apps. Hub reads `taffish.toml` and release tags, then writes image information into the index.

## GHCR Visibility

After publishing an image, confirm that the GHCR package is public. Otherwise users may install the app but fail to pull the image at runtime.

TAFFISH mirror config, supported since `0.2.0`, can rewrite index and app
repository access, but it does not automatically rewrite container image
registries. If users cannot access GHCR, the app must publish or declare an
image source that those users can reach.

Checklist:

- The package belongs to the `taffish` organization.
- Package visibility is public.
- The image tag exists.
- The image tag matches `taffish.toml`.

## Local Testing

`taf build` builds only the command wrapper by default. It does not build container images. For a containerized app, build the image before local test runs.

Build the image with Docker:

```sh
taf build --image --backend docker
```

Build both image and command wrapper:

```sh
taf build --all --backend docker
```

Run with the same backend:

```sh
taf run --backend docker -- --help
```

For Podman, keep build and run consistent:

```sh
taf build --image --backend podman
taf run --backend podman -- --help
```

`taf run --backend` only affects generic `<container:...>` / `<taf-app:container:...>` tags. Explicit tags fix the backend:

```taf
<taf-app:docker:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --help
```

If an app truly requires one backend, use an explicit backend tag in `src/main.taf`.
If the app is portable but needs different runtime flags on different backends,
keep the generic `container` tag and use TAFFISH `0.9.0` structured runtime
arguments instead:

```taf
<taf-app:container:ghcr.io/taffish/my-tool:1.0.0-r1$@[docker: --gpus all][podman: --device nvidia.com/gpu=all][apptainer: --nv]>
my-tool ::*ARGV*::
```

Use `$@[target: args]` for app-specific requirements. For example, if a tool app
needs GPU access as part of its normal function, the app should declare those
backend-specific GPU arguments in `src/main.taf`. Do not make every user supply
them manually.

For installed `taf-*` commands or direct `taffish` compilation, use
`TAFFISH_CONTAINER_BACKEND=apptainer|podman|docker` to force generic
container tags without editing the app:

```sh
TAFFISH_CONTAINER_BACKEND=podman taf-my-tool-v0.1.0-r1 -- --help
TAFFISH_CONTAINER_BACKEND=podman taf-my-tool-v0.1.0-r1 --compile -- --help
```

This runtime environment variable does not override explicit backend tags.
During local project testing, `taf run --backend ...` has priority over it.

For one-off local policy that is not part of the app implementation, use backend
runtime-argument environment variables:

```sh
TAFFISH_DOCKER_RUN_ARGS="--platform linux/amd64" taf-my-tool ...
TAFFISH_PODMAN_RUN_ARGS="--platform linux/amd64" taf-my-tool ...
TAFFISH_APPTAINER_RUN_ARGS="--bind /scratch:/scratch" taf-my-tool ...
```

These variables are for machine, cluster, site, platform, or single-run
differences. They append to the generated runtime command after app-declared tag
arguments.

You can also test the container directly:

```sh
docker run --rm ghcr.io/taffish/my-tool:0.1.0-r1 my-tool --help
```

## Common Issues

Image too large:

- Use multi-stage builds.
- Remove source archives and build caches.
- Use `--no-install-recommends`.
- Install runtime dependencies only.

Users cannot pull image:

- Check GHCR visibility.
- Check organization and package permissions.
- Check whether the tag exists.

`taf check` reports image tag mismatch:

- Confirm `[package].version` and `[package].release`.
- Confirm `[container].image`.
- Confirm the image in `src/main.taf`.

`taf build --image` succeeds but `taf run` cannot find the image:

- Confirm `taf build --image --backend ...` and `taf run --backend ...` use the same backend.
- If `src/main.taf` uses `<taf-app:container:...>`, runtime selection may choose a different available backend.
- During development, use `<taf-app:docker:...>` or `<taf-app:podman:...>`.
- For installed commands, set `TAFFISH_CONTAINER_BACKEND=...` when you need the same backend outside `taf run`.

Multi-platform build fails:

- First confirm that `linux/amd64` builds successfully.
- Check whether the upstream tool supports `linux/arm64`.
- Temporarily publish only `linux/amd64` when needed.
