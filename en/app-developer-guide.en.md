# TAFFISH App Developer Guide

This guide is for maintainers who want to develop TAFFISH apps. It covers the workflow from project creation to publishing into the official TAFFISH Hub.

The official TAFFISH Hub is not currently an open self-service publishing platform. Only members of the `taffish` GitHub organization can create, publish, and maintain official app repositories. At the moment, official maintenance is handled by a single maintainer. Developers who want an app included in the official Hub should contact the maintainer to request organization membership or request maintainer-assisted review and publishing.

This is the general app lifecycle guide. Exact field semantics live in the
[`taffish.toml` Specification](taffish-toml-spec.en.md). Dockerfile, GHCR, and
smoke details live in [Containerized App Best Practices](container-apps.en.md).
Official Hub curation checklists live in the
[Official taf-app Curation Guide](taf-app-curation-guide.en.md). When something
fails, start with [Troubleshooting](troubleshooting.en.md).

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
- Optional `[meta]` metadata when it is intended for public Hub discovery.
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
- Can automatically run a user-provided command inside the same app runtime
  when the first runtime argument is not option-like.

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

Recommended discovery metadata:

```toml
[meta]
domain = "bioinformatics"
category = "sequence-clustering"
summary = "CD-HIT wrapper for clustering biological sequences."
keywords = ["cd-hit", "clustering", "sequence-identity"]
```

`[meta]` is optional for local development, but public Hub apps should provide
it when useful. It improves Hub search, filtering, and app detail pages.

Recommended upstream metadata:

```toml
[upstream]
name = "CD-HIT"
type = "github"
url = "https://github.com/weizhongli/cdhit"
homepage = "https://github.com/weizhongli/cdhit"
repository = "weizhongli/cdhit"
version = "4.8.1"
license = "GPL-2.0"
citation = "Fu et al. 2012"
doi = "10.1093/bioinformatics/bts565"
pmid = "23060610"
```

`[upstream]` describes the original software being wrapped. It is not the TAFFISH app repository.
For bioinformatics tools, include verified scholarly attribution when available:
`citation` and `doi` are recommended, and `pmid` is useful when the canonical
paper is indexed in PubMed. Leave these fields out rather than guessing.

For new public releases, keep discovery and upstream metadata in
`taffish.toml`. If an already published immutable release only needs display
metadata corrected or existing upstream attribution metadata supplemented,
official index maintainers can use `metadata-overrides.toml` instead of
publishing a new app release.

See [`taffish.toml` Specification](taffish-toml-spec.en.md) for the full field reference.

## Writing `src/main.taf`

Tool apps usually use `<taf-app:...>`:

```taf
<taf-app:shell>
my-tool ::*ARGV*::
```

Containerized tool app:

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool ::*ARGV*::
```

This thin wrapper has two useful modes:

```sh
taf-my-tool -- --help
taf-my-tool -- --input sample.fa --output result.txt
```

These option-like calls use the default body in `src/main.taf`. But tool apps
also have automatic command mode:

```sh
taf-my-tool my-tool --help
taf-my-tool helper-command --help
```

When the first runtime argument is not option-like, TAFFISH runs the
user-provided command in the same app runtime/container. Use this for
containerized tool apps that expose several upstream executables.

Flow apps usually use `<taffish>`:

```taf
<taffish>
[[taf: taf-fastqc-v0.12.1-r1 fastqc sample.fq]]
[[taf: taf-multiqc-v1.35-r2 multiqc .]]
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

`@:` block parameters are good for providing one step's advanced append-only
argument slot:

```taf
ARGS
<!(--/-i)input>
<(--/-o)outdir>
  qc
<(--/-t)threads>
  4

<(@:)fastqc-step>

<(@:)multiqc-step>

RUN
<taffish>
mkdir -p ::outdir::
[[taf: taf-fastqc-v0.12.1-r1 fastqc --threads ::threads:: --outdir ::outdir:: ::input:: ::(@:)fastqc-step::]]
[[taf: taf-multiqc-v1.35-r2 multiqc ::outdir:: ::(@:)multiqc-step::]]
```

In this example:

- `input` and `outdir` are flow domain parameters.
- `threads` is a common top-level flow parameter.
- `fastqc-step` and `multiqc-step` are empty slots by default.
- Native low-level tool arguments are appended only when the user explicitly
  passes `@fastqc-step: ... @:` or `@multiqc-step: ... @:`.

This keeps the flow logic clear: ordinary users control stable domain
parameters, while advanced users can append native low-level tool arguments at a
specific step when needed.

Inside an `ARGS` body, prefer `::name::` for ordinary parameter references. `@name` and `@{name}` also work, but are better suited for default-expression composition, such as `::(--/-p)prefix=out-@{input}::`.

Advice:

- Expose stable domain parameters such as `input`, `outdir`, `genome`, and `threads`.
- Use `(@:)step-name` to provide an empty advanced append-only slot for one
  tool-call step.
- In formal flows, each real `[[taf: ...]]` call site that affects results,
  resource use, or major report content should have a corresponding
  `::(@:)step-name::` advanced parameter entry point.
- If the same taf app is called multiple times, split block names by call site,
  such as `salmon-index-step` and `salmon-quant-step`, instead of using one
  broad app-level block for several steps.
- `::(@:)step-name::` expands to nothing by default; do not prefill it in source
  code, templates, shell variables, or helper scripts.
- Default tool arguments that belong to the flow should be written directly in
  the command body and documented as normal behavior.
- The default flow path should remain complete without any `@step:` arguments;
  `@:` is an advanced extension entry point, not a required normal-use
  parameter.
- User-supplied step parameters should be recorded in `commands.sh`,
  `run.manifest.json`, or equivalent provenance.
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

Minimal container metadata:

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

Minimal entry point:

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
placeholders. It does not run smoke commands locally. The public Hub/index runs
smoke checks after the image is available. Required failures block a new version; advisory backend failures remain visible in the report.

This section only shows the minimal shape. Dockerfile design, runtime mounts,
GHCR visibility, local Docker/Podman testing, and smoke details live in
[Containerized App Best Practices](container-apps.en.md). For official Hub app
curation, also use the [Official taf-app Curation Guide](taf-app-curation-guide.en.md).

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
TAFFISH_CONTAINER_BACKEND=podman taf-my-tool-v0.1.0-r1 --compile --help
```

This does not override explicit backend tags. During local project runs,
`taf run --backend ...` has priority over the environment variable.

If the app truly requires one backend, or if it only exists in one local backend
store during development, use an explicit backend tag in `src/main.taf`:

```taf
<taf-app:podman:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --input ::input::
```

If the app is portable but has backend-specific runtime requirements, keep the
generic `container` tag and use TAFFISH `0.9.0` structured runtime arguments:

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1$@[docker: --gpus all][podman: --device nvidia.com/gpu=all][apptainer: --nv]>
my-tool ::*ARGV*::
```

Use this for app implementation needs. For example, GPU access required by the
wrapped tool should be declared in the app source. Use
`TAFFISH_DOCKER_RUN_ARGS`, `TAFFISH_PODMAN_RUN_ARGS`, or
`TAFFISH_APPTAINER_RUN_ARGS` only for local machine, site, platform, or one-off
runtime policy that is not part of the app itself.

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

The built command uses the frozen source snapshot, not the live `src/` directory. In 0.11.0, rerunning `taf build` verifies and reuses matching artifacts; an invalid snapshot is rebuilt in staging. A publish dry-run shows the plan, not evidence that the wrapper was built or tested.

After upgrading both `taf` and `taffish` to 0.11.0, rerun `taf build` in any app source project that still uses an older wrapper to pick up the new history writer. Upgrading the core alone does not replace old wrappers; a local rebuild does not require republishing the app.

## Publish

From the app project root, run `taf check`. Proceed only if it passes, `docs/help.md`
is present, and `[repository].url` points to the intended repository. For a containerized
app, verify that its declared image builds and that `[smoke]` contains real checks,
not scaffold placeholders. Stop if an existing tag points to a different commit or
an existing Release conflicts with what you intend to publish.

**Choose the release channel.** TAFFISH 0.11.0 does not require the new version to
sort above earlier tags. Choose a prerelease only when the app is a preview:

| What do you want to publish? | Options | Result on GitHub |
| --- | --- | --- |
| A new recommended release | `--release` (or `--release --latest`) | The new release becomes Latest. |
| A preview for testers | `--release --pre` | A prerelease, not Latest. |
| A maintenance release that should not become Latest | `--release --no-latest` | A regular release, not marked Latest. |
| A tag without a GitHub Release | Omit `--release` and the channel flags | The tag is pushed; Latest does not change. |

Use at most one of `--latest`, `--pre` and `--no-latest`. On an older installation,
check `taf --version` and `taf publish --help` for supported options.
If you use `--release`, edit the Git-ignored `release.md`: the first line supplies
the summary for the publish commit, and the full file becomes the GitHub Release
notes. Since 0.8.1, the first-line placeholders `TODO` and `TODO: release summary`
are rejected; the word `todo` inside a real summary is fine.

**Preview the exact command you intend to run.** For example, to build the wrapper
and publish a new recommended release:

```sh
taf publish --release --build --dry-run
```

Check the repository, tag, channel and build setting in the plan. A dry run does
not build the wrapper, commit, tag, push or create a GitHub Release. For a
prerelease, include `--pre` in the preview; to create a missing public repository,
include `--create-repo --public`.

**Publish only after checking the plan.** Keep all selected options and replace
`--dry-run` with `--yes`. For the recommended release above:

```sh
taf publish --release --build --yes
```

If publication is interrupted, retry with the same repository, options and
`release.md`. Matching steps are skipped; a conflicting commit, tag or Release stops
the retry. Retrying a completed release does not republish it or take
Latest back from a later release. A published tag does not mean that the app’s
image workflow has succeeded or that the Hub Index has accepted the version.

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
- [ ] If intended for public Hub discovery, `[meta]` contains useful domain, category, summary, and keywords.
- [ ] For a tool app, upstream source is recorded in `[upstream]` where possible.
- [ ] For scholarly bioinformatics tools, `[upstream]` includes verified `license`, `citation`, `doi`, and optionally `pmid`.
- [ ] For a containerized app, image tag matches `version-release`.
- [ ] For a containerized app, `[smoke]` has real `exist` or `test` checks and no `TODO` placeholders.
- [ ] For a containerized app, local `taf build --image --backend ...` and `taf run --backend ...` use a consistent backend.
- [ ] If testing a private app before public release, `taf install --from <PROJECT-DIR>` works from the project root or a child directory.
- [ ] For a flow app, `[dependencies]` matches `[[taf: ...]]`.
- [ ] `taf run` passes locally.
- [ ] `taf build` or `taf build --all` passes.
- [ ] For a GitHub Release, `release.md` has a real first-line summary and release notes.
- [ ] `taf publish` was previewed with `--dry-run` and the intended channel and build options.
- [ ] After publishing, GitHub Actions and Hub index are checked.
