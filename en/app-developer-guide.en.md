# TAFFISH App Developer Guide

This guide is for maintainers who want to develop TAFFISH apps. It covers the workflow from project creation to publishing into the official TAFFISH Hub.

The official TAFFISH Hub is not currently an open self-service publishing platform. Only members of the `taffish` GitHub organization can create, publish, and maintain official app repositories. At the moment, official maintenance is handled by a single maintainer. Developers who want an app included in the official Hub should contact the maintainer to request organization membership or request maintainer-assisted review and publishing.

## Table Of Contents

- [Development Goals](#development-goals)
- [Choosing An App Type](#choosing-an-app-type)
- [Creating A Project](#creating-a-project)
- [Project Structure](#project-structure)
- [Writing `taffish.toml`](#writing-taffishtoml)
- [Writing `src/main.taf`](#writing-srcmaintaf)
- [Parameter Design And `@:` Blocks](#parameter-design-and--blocks)
- [Writing Help Documentation](#writing-help-documentation)
- [Containerized Tool Apps](#containerized-tool-apps)
- [Flow Apps And Dependencies](#flow-apps-and-dependencies)
- [Local Check And Run](#local-check-and-run)
- [Build](#build)
- [Publish](#publish)
- [Post-Publish Checks](#post-publish-checks)
- [Version Maintenance](#version-maintenance)
- [Pre-Publish Checklist](#pre-publish-checklist)

## Development Goals

A good TAFFISH app should have:

- A clear package name and command name.
- A stable version and release number.
- A runnable `src/main.taf`.
- A user-readable `docs/help.md`.
- If it uses a container, an image tag aligned with the TAFFISH version id.
- If it uses a container, real `[smoke]` checks instead of the default TODO placeholders.
- If it wraps third-party bioinformatics software, `[upstream]` metadata where possible.
- If it is a flow, accurate `[dependencies]`.
- A repository and release tag that can be indexed by TAFFISH Hub.
- A command that can be built with `taf build`.

## Choosing An App Type

### Tool App

Choose `tool` when the app mainly wraps one upstream tool.

A tool app usually:

- Has `kind = "tool"`.
- Uses `<taf-app:shell>` or `<taf-app:container:...>`.
- Uses `runtime.pipe = true`.
- Uses `runtime.command_mode = true`.
- Exposes a command that behaves like a normal CLI tool.

### Flow App

Choose `flow` when the app combines multiple taf apps into a workflow.

A flow app usually:

- Has `kind = "flow"`.
- Uses `<taffish>`.
- Calls other apps through `[[taf: ...]]`.
- Declares dependencies in `[dependencies]`.
- Uses `runtime.pipe = false`.
- Uses `runtime.command_mode = false`.

Basic rule:

- If you are wrapping one executable, choose tool.
- If you are composing multiple taf apps, choose flow.
- If you need both an upstream tool and a multi-step workflow, usually wrap the upstream tool as a tool app first, then compose it with a flow.

## Creating A Project

Create a default flow project:

```sh
taf new my-flow
```

Create a tool project:

```sh
taf new my-tool --tool
```

Create a tool project with Dockerfile and image build workflow:

```sh
taf new my-tool --tool --docker
```

Create a project with explicit metadata:

```sh
taf new my-tool --tool --docker \
  --version 1.0.0 \
  --release 1 \
  --license Apache-2.0 \
  --repo https://github.com/taffish/my-tool
```

## Project Structure

Typical structure:

```text
my-tool/
  README.md
  release.md
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

Important files:

- `taffish.toml`: app metadata, runtime behavior, dependencies, container info, smoke checks, platform constraints, and upstream metadata.
- `src/main.taf`: TAFFISH source code.
- `docs/help.md`: help text shown by the built command.
- `target/`: generated command wrappers and frozen snapshots.
- `docker/Dockerfile`: container image definition for containerized tool apps.
- `.github/workflows/build-image.yml`: image build automation for app repositories.
- `release.md`: ignored local draft for publish message and GitHub Release notes.

`taf new` creates `release.md` as a local draft and adds it to `.gitignore`.
It is used by `taf publish --release`; it should not be committed as app
source.

## Writing `taffish.toml`

Minimal tool example:

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

Minimal flow example:

```toml
[package]
name = "my-flow"
kind = "flow"
version = "0.1.0"
release = 1
license = "Apache-2.0"
main = "src/main.taf"

[repository]
url = "https://github.com/taffish/my-flow"

[command]
name = "taf-my-flow"

[runtime]
pipe = false
command_mode = false
```

`[repository].url` should be the canonical GitHub repository for the TAFFISH app.
Do not put a Gitee or internal mirror URL here for official Hub packages.
Since TAFFISH `0.2.0`, mirrors are handled through local `taf` runtime config,
where `[[source.rewrite]]` can rewrite the canonical URL during install.

Recommended upstream metadata:

```toml
[upstream]
name = "CD-HIT"
type = "github"
homepage = "https://github.com/weizhongli/cdhit"
repository = "weizhongli/cdhit"
version = "4.8.1"
license = "GPL-2.0"
doi = "10.1093/bioinformatics/bts565"
pmid = "23060610"
```

`[upstream]` describes the original software being wrapped. It is not the TAFFISH app repository.

See [`taffish.toml` Specification](taffish-toml-spec.en.md) for the full field reference.

## Writing `src/main.taf`

Tool apps usually use `<taf-app:...>`:

```taf
<taf-app:shell>
my-tool --input ::input:: --output ::output::
```

Containerized tool app:

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --input ::input:: --output ::output::
```

Flow apps usually use `<taffish>`:

```taf
<taffish>
[[taf: taf-fastqc-v0.12.1-r1 sample.fq]]
[[taf: taf-multiqc-v1.19-r1 .]]
```

Parameters use `::...::`:

```taf
<taf-app:shell>
my-tool --threads ::threads=4:: --input ::input::
```

Advice:

- Keep parameter names stable after release.
- Tool apps should preserve upstream tool arguments where reasonable.
- Flow apps should use exact versioned commands, such as `taf-fastqc-v0.12.1-r1`.
- Avoid cramming complex shell logic into one line.

## Parameter Design And `@:` Blocks

TAFFISH parameter design is more than string concatenation. For tool apps, the parameter system exposes an upstream CLI through a stable taf command. For flow apps, `@:` block parameters are especially important because they can package a step's low-level tool arguments into one reusable and extensible domain step parameter.

Ordinary parameters are good for direct user input:

```taf
ARGS
<!(--/-i)input>
<(--/-o)output>
  result.tsv
<(--/-t)threads>
  4

RUN
<taf-app:shell>
my-tool --input ::input:: --output ::output:: --threads ::threads::
```

`@:` block parameters are good for packaging one step's argument fragment:

```taf
ARGS
<!(--/-i)input>
<(--/-o)outdir>
  qc

<(@:)fastqc-step>
  --threads ::(--/-t)threads=4::
  --outdir ::outdir::
  ::(--/-e)fastqc-extra=::

<(@:)multiqc-step>
  ::(--/-m)multiqc-extra=::

RUN
<taffish>
mkdir -p ::outdir::
[[taf: taf-fastqc-v0.12.1-r1 ::(@:)fastqc-step:: ::input::]]
[[taf: taf-multiqc-v1.19-r1 ::(@:)multiqc-step:: ::outdir::]]
```

In this example:

- `input` and `outdir` are flow domain parameters.
- `fastqc-step` is a whole FastQC argument block.
- `multiqc-step` is a whole MultiQC argument block.
- `threads`, `fastqc-extra`, and `multiqc-extra` are adjustable parameters embedded inside blocks.

This keeps the flow logic clear, gives users useful controls, and prevents low-level tool arguments from being scattered across many `[[taf: ...]]` calls.

Inside an `ARGS` body, prefer `::name::` for ordinary parameter references. `@name` and `@{name}` also work, but are better suited for default-expression composition, such as `::(--/-p)prefix=out-@{input}::`.

Advice:

- Expose stable domain parameters such as `input`, `outdir`, `genome`, and `threads`.
- Use `(@:)step-name` to package one tool-call step's arguments.
- Preserve an `extra` parameter for each step when advanced users may need native upstream arguments.
- Do not mix unrelated tool arguments into one block.
- Avoid renaming released parameters; add compatible new parameters instead.

## Writing Help Documentation

`docs/help.md` is required. The built command prints it for:

```sh
taf-my-tool-v0.1.0-r1 --help
```

Recommended content:

- App summary.
- Upstream software source.
- Common usage examples.
- Parameter descriptions, including domain parameters and `extra` arguments exposed through `@:` blocks.
- Input and output description.
- Container image notes.
- Version information.
- Citation instructions.
- License notes.

Minimal example:

````md
# my-tool

A TAFFISH wrapper for My Tool.

## Usage

```sh
taf-my-tool-v0.1.0-r1 --input sample.fa --output result.txt
```
````

## Containerized Tool Apps

Use containers when a tool has complex system packages, compiled dependencies, or bioinformatics runtime requirements.

`taffish.toml`:

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

`src/main.taf`:

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --input ::input::
```

Requirements:

- Image tag should match `<version>-r<release>`.
- Dockerfile should be versioned in the app repository.
- Image build workflow should live in the app repository.
- GHCR package should be public so users can pull it.
- `[smoke]` should contain real checks that prove the expected executable exists and a minimal command can run.

`taf check` validates `[smoke]` structure and rejects the default `TODO`
placeholders. It does not run smoke commands locally. The public Hub/index
automation runs smoke checks for new containerized versions after the image is
available, records digest/platform/smoke metadata, and only writes passing new
versions into the main index.

See [Containerized App Best Practices](container-apps.en.md).

## Flow Apps And Dependencies

Flow apps call other apps through `[[taf: ...]]`:

```taf
<taffish>
[[taf: taf-trim-v1.0.0-r1 sample.fq]]
[[taf: taf-align-v2.0.0-r1 sample.trimmed.fq]]
```

Dependencies should be declared in `taffish.toml`:

```toml
[dependencies]
taf-trim = "1.0.0-r1"
taf-align = "2.0.0-r1"
```

If the same flow requires multiple versions of one app:

```toml
[dependencies]
taf-align = ["2.0.0-r1", "2.1.0-r1"]
```

This means both versions are required. It does not mean alternatives.

See [Flow And Dependencies Guide](flow-dependencies.en.md).

## Local Check And Run

Check the project:

```sh
taf check
```

Compile the current project:

```sh
taf compile -- --input sample.fa
```

Run the current project:

```sh
taf run -- --input sample.fa
```

For a containerized app whose image has not been published to a remote registry, build the image locally first, then run with the same backend:

```sh
taf build --image --backend docker
taf run --backend docker -- --input sample.fa
```

`taf run --backend` forces generic `<container:...>` / `<taf-app:container:...>` tags to use the selected backend. It does not override explicit tags such as `<taf-app:podman:...>` or `<taf-app:docker:...>`.

After a command wrapper is installed, or when compiling directly with
`taffish`, use `TAFFISH_CONTAINER_BACKEND=apptainer|podman|docker` to force
generic container tags at runtime:

```sh
TAFFISH_CONTAINER_BACKEND=podman taf-my-tool-v0.1.0-r1 -- --help
TAFFISH_CONTAINER_BACKEND=podman taf-my-tool-v0.1.0-r1 --compile -- --help
```

This does not override explicit backend tags. During local project runs,
`taf run --backend ...` has priority over the environment variable.

If the image is not portable across all backends, or if it only exists in one local backend store during development, use an explicit backend tag in `src/main.taf`:

```taf
<taf-app:podman:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --input ::input::
```

Common checks:

- Required `taffish.toml` fields.
- `src/main.taf` exists and can be parsed.
- `docs/help.md` exists.
- Dockerfile exists.
- Container image tag matches current version and release.
- Flow dependencies are declared.

## Build

Build the versioned command wrapper:

```sh
taf build
```

`taf build` builds only the command wrapper by default. It does not build container images.

Build the container image:

```sh
taf build --image
```

Build command wrapper and image:

```sh
taf build --all
```

Select an image build backend:

```sh
taf build --all --backend docker
```

Recommended local flow for containerized app development:

```sh
taf build --image --backend docker
taf run --backend docker -- --help
taf build
```

To build image and command wrapper together:

```sh
taf build --all --backend docker
```

Build artifacts:

```text
target/taf-my-tool-v0.1.0-r1
target/.taf-my-tool-v0.1.0-r1/
```

The built command uses the frozen source snapshot, not the live `src/` directory. This keeps released command behavior tied to the release.

## Publish

Before publishing, edit the ignored `release.md` draft. With
`taf publish --release`, the first line of `release.md` becomes the publish
commit/tag message, and the whole file becomes the GitHub Release notes. Replace
the default `TODO` summary before a real release.

Publish preview with release notes:

```sh
taf publish --release --dry-run
```

Publish with release notes:

```sh
taf publish --release --yes --build
```

Create a repository during publish when supported:

```sh
taf publish --release --yes --build --create-repo --public
```

Publishing checks the project, reads the repository URL from `taffish.toml`,
checks remote tags, prepares the release message and release notes, and then
commits, tags, pushes, and creates or updates the GitHub Release after
confirmation.

Do not publish directly if:

- The repository URL is wrong.
- The image tag has not been built yet.
- `taf check` fails.
- `docs/help.md` is missing.
- `[smoke]` is missing for a containerized app or still contains `TODO`.
- `release.md` still contains a placeholder `TODO` summary.
- The release tag already exists.

## Post-Publish Checks

After publishing:

```sh
git tag
git status
git remote -v
```

On GitHub:

- Check that the release tag exists.
- Check GitHub Actions.
- Check GHCR package visibility if the app uses a container.
- Check the `taffish-index` report if a new containerized version is not accepted; smoke or digest gates may have failed.
- Check that TAFFISH Hub can index the package after the next index run.

After the index updates:

```sh
taf update
taf search my-tool
taf info my-tool
taf install --dry-run my-tool
```

For private or local testing before a public Hub release, install directly from
the project tree:

```sh
taf install --from /path/to/my-tool
taf list
taf which taf-my-tool
```

`taf install --from` checks the local project, copies the working tree into the
selected TAFFISH home, builds the versioned command wrapper, and records the
origin as `[local-project] <PROJECT-ROOT>`. It does not require `taf update`
and does not auto-install dependencies.

## Version Maintenance

TAFFISH uses:

```text
version = "0.1.0"
release = 1
version_id = "0.1.0-r1"
tag = "v0.1.0-r1"
```

Increase `release` when:

- TAFFISH wrapper logic changes.
- Dockerfile packaging changes.
- Help documentation changes.
- App metadata changes.
- Upstream version stays the same but TAFFISH packaging changes.

Increase `version` when:

- Upstream software version changes.
- A flow's main feature version changes.
- App behavior changes in a way users should notice.

Do not overwrite published tags. Publish a new release, for example from `0.1.0-r1` to `0.1.0-r2`.

## Pre-Publish Checklist

Before publishing, confirm:

- [ ] `taffish.toml` has correct `name`, `version`, `release`, and `repository.url`.
- [ ] `command.name` starts with `taf-`.
- [ ] `src/main.taf` can be parsed by `taf check`.
- [ ] `docs/help.md` is updated.
- [ ] For a tool app, upstream source is recorded in `[upstream]` where possible.
- [ ] For a containerized app, image tag matches `version-release`.
- [ ] For a containerized app, `[smoke]` has real `exist` or `test` checks and no `TODO` placeholders.
- [ ] For a containerized app, local `taf build --image --backend ...` and `taf run --backend ...` use a consistent backend.
- [ ] If testing a private app before public release, `taf install --from <PROJECT-DIR>` works from the project root or a child directory.
- [ ] For a flow app, `[dependencies]` matches `[[taf: ...]]`.
- [ ] `taf run` passes locally.
- [ ] `taf build` or `taf build --all` passes.
- [ ] `release.md` has a real first-line summary and release notes.
- [ ] `taf publish --release --dry-run` output is as expected.
- [ ] After publishing, GitHub Actions and Hub index are checked.
