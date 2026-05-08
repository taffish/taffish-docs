# Containerized App Best Practices

Containerized apps are the recommended way to package tools with complex system dependencies, compiled binaries, or bioinformatics runtime environments. In TAFFISH, the app repository owns the Dockerfile and image build workflow. TAFFISH Hub only indexes metadata and releases.

## Table Of Contents

- [When To Use Containers](#when-to-use-containers)
- [Project Structure](#project-structure)
- [`taffish.toml`](#taffishtoml)
- [`src/main.taf`](#srcmaintaf)
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

If an image only exists in a local Docker or Podman store, or if runtime arguments depend on one backend, use an explicit backend tag in `src/main.taf` instead of generic `container`.

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

Multi-platform build fails:

- First confirm that `linux/amd64` builds successfully.
- Check whether the upstream tool supports `linux/arm64`.
- Temporarily publish only `linux/amd64` when needed.

