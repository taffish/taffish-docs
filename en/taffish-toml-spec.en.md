# `taffish.toml` Specification

`taffish.toml` is the core metadata file for a TAFFISH app. `taf check`, `taf build`, `taf publish`, `taffish-index`, and TAFFISH Hub all depend on it to understand an app.

This document describes the currently recommended fields. Fields not listed here
may be ignored by current tools and are not guaranteed to remain compatible in
the future.

This is the field-level source of truth. For the practical development workflow,
see the [TAFFISH App Developer Guide](app-developer-guide.en.md). For container
practice, see [Containerized App Best Practices](container-apps.en.md). For
official Hub curation, see the
[Official taf-app Curation Guide](taf-app-curation-guide.en.md).

## Table Of Contents

- [Basic Principles](#basic-principles)
- [`[package]`](#package)
- [`[repository]`](#repository)
- [`[command]`](#command)
- [`[runtime]`](#runtime)
- [`[container]`](#container)
- [`[smoke]`](#smoke)
- [`[dependencies]`](#dependencies)
- [`[platform]`](#platform)
- [`[meta]`](#meta)
- [`[upstream]`](#upstream)
- [Tool Example](#tool-example)
- [Flow Example](#flow-example)
- [Version And Tag Rules](#version-and-tag-rules)
- [Common Errors](#common-errors)

## Basic Principles

`taffish.toml` uses a small TOML subset:

- Sections use `[section]`.
- Strings use double quotes.
- Booleans use `true` / `false`.
- Integers are used for release, CPU, memory, and similar fields.
- String arrays are used for multi-version dependencies and similar fields.
- The current project parser expects arrays on one line, for example
  `test = ["tool --help"]`.
- Containerized apps should provide `[smoke]` checks that are cheap, deterministic, and safe to run in Hub automation.

Apps should keep metadata simple, stable, and machine-parseable.

## `[package]`

Required.

```toml
[package]
name = "my-tool"
kind = "tool"
version = "0.1.0"
release = 1
license = "Apache-2.0"
main = "src/main.taf"
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `name` | string | yes | Package name. Must not start with `-` or `.`; recommended characters are letters, digits, `-`, and `_`. |
| `kind` | string | yes | `tool` or `flow`. |
| `version` | string | yes | Upstream or flow version. Must not be empty and must not contain spaces or tabs. |
| `release` | integer | yes | TAFFISH packaging release. Must be a positive integer. |
| `license` | string | no | App wrapper license. Current `taf new` default is `Apache-2.0`. |
| `main` | string | yes | Main `.taf` path. Must be a project-relative path. |

`name` is the package name. It does not have to equal the command name, which is
defined by `[command].name`.

`kind` rules:

```toml
# tool app
kind = "tool"

# flow app
kind = "flow"
```

Use `tool` when the app wraps one executable or upstream tool. Use `flow` when the app composes multiple taf apps.

## `[repository]`

Required.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `url` | string | yes | Git repository URL for the app. |

Example:

```toml
[repository]
url = "https://github.com/taffish/my-tool"
```

The repository URL is used by:

- `taf publish`
- `taffish-index`
- TAFFISH Hub
- `taf install`

For official Hub packages, this should normally be the canonical GitHub
repository under the `taffish` organization. Do not replace this field with a
mirror URL for normal packages. Since TAFFISH `0.2.0`, mirror support is handled
by local `taf` runtime config, where `[[source.rewrite]]` can rewrite this
canonical URL during `taf install`.

## `[command]`

Required.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `name` | string | yes | Base command name. Must start with `taf-`. |

Example:

```toml
[command]
name = "taf-my-tool"
```

Built command:

```text
taf-my-tool-v0.1.0-r1
```

The index also maps the base command name to the package so that users can search and install by command.

## `[runtime]`

Required.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `pipe` | boolean | yes | Whether the command is intended to work in pipelines. |
| `command_mode` | boolean | yes | Whether the app should pass command-like argv through. |

Typical tool app:

```toml
[runtime]
pipe = true
command_mode = true
```

Typical flow app:

```toml
[runtime]
pipe = false
command_mode = false
```

Use `command_mode = true` for wrappers that should behave like ordinary tool commands. Use `command_mode = false` for structured flow commands.

## `[container]`

Optional. Mainly used by containerized tool apps.

```toml
[container]
image = "ghcr.io/taffish/my-tool:0.1.0-r1"
dockerfile = "docker/Dockerfile"
build_platforms = "linux/amd64,linux/arm64"
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `image` | string | no | Container image used by the app. |
| `dockerfile` | string | no | Dockerfile path. Must be project-relative. |
| `build_platforms` | string | no | Image build platform list, commonly `linux/amd64,linux/arm64`. |

Rules:

- If `dockerfile` is set, the file must exist.
- `image` tag should match `<version>-r<release>`.
- `taffish-index` currently exports `image` and `dockerfile`; `build_platforms` is mainly used by local builds and GitHub Actions.

`[container]` describes the image metadata. The actual runtime tag still lives in `src/main.taf`, for example:

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --help
```

## `[smoke]`

Required for containerized apps that declare `[container].image` or
`[container].dockerfile`. Optional for non-container apps.

`taf new --tool --docker` creates a placeholder section:

```toml
[smoke]
backend = "docker"
timeout = 60
exist = ["TODO"]
test = ["TODO --help"]
```

Replace the placeholders before running `taf check` or publishing:

```toml
[smoke]
backend = "docker"
timeout = 60
exist = ["my-tool"]
test = ["my-tool --help"]
```

Nested shell quoting is allowed. The recommended style is to keep TOML's outer
string quotes as double quotes and use single quotes inside shell snippets:

```toml
test = ["python -c 'import vina, rdkit, meeko, gemmi, prody'"]
```

TOML basic string escapes such as `\"` are supported by the index parser, but
the single-quote style is usually easier to read and review.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `backend` | string | no | Preferred smoke backend: `docker`, `podman`, or `apptainer`. Default is `docker`. |
| `timeout` | integer | no | Per-command timeout in seconds. Default is `60`. Must be positive. |
| `exist` | string array | no | Executable names that should be discoverable in the container `PATH`. |
| `test` | string array | no | Shell commands that should exit with status `0`. |

Rules:

- Containerized projects must define `[smoke]`.
- `exist` and `test` cannot both be empty.
- `TODO` placeholders are rejected by `taf check`.
- `taf check` validates the metadata but does not run smoke commands.
- Hub/index automation runs smoke checks for new containerized versions, records
  smoke status, and rejects failed new versions from the main public index.

## `[dependencies]`

Optional. Mainly used by flow apps.

Single dependency version:

```toml
[dependencies]
taf-dep-tool = "0.1.0-r1"
taf-align = ["1.0.0-r1", "1.1.0-r1"]
```

Rules:

- Key must be a taf command and must start with `taf-`.
- Value may be a string or a string array.
- Strings must not be empty.
- Arrays must not be empty.
- An app must not depend on itself.

Semantics:

```toml
taf-align = ["1.0.0-r1", "1.1.0-r1"]
```

means this flow needs both versions of `taf-align` to be installable. It does
not mean one of them can be chosen as an alternative.

Dependency values should normally use version ids without a leading `v`:

```text
1.0.0-r1
```

`taf build` can sync dependencies from versioned `[[taf: ...]]` calls in `src/main.taf`.

## `[platform]`

Optional. Describes platform constraints, mainly for Hub display and future
installation decisions.

Example:

```toml
[platform]
os = "linux,darwin"
arch = "amd64,arm64"
container = "required"
min_cpus = 2
min_memory_mb = 4096
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `os` | string | no | Comma-separated operating system list. |
| `arch` | string | no | Comma-separated architecture list. |
| `container` | string | no | `optional`, `required`, or `forbidden`. Default is `optional`. |
| `min_cpus` | integer | no | Suggested minimum CPU count; positive integer. |
| `min_memory_mb` | integer | no | Suggested minimum memory in MB; positive integer. |

Current `container` meanings:

- `optional`: can run in a container, but may also run locally.
- `required`: requires a container environment.
- `forbidden`: should not run inside a container.

## `[meta]`

Optional. Describes ecosystem discovery metadata for search, filtering,
documentation, and Hub display. Local `taf` commands do not require it.

Recommended app-side form:

```toml
[meta]
domain = "bioinformatics"
category = "sequence-alignment"
summary = "BLAST+ wrapper for sequence similarity search."
keywords = ["blast", "alignment", "sequence-search"]
```

| Field | Type | Description |
| --- | --- | --- |
| `domain` | string | Broad domain, for example `bioinformatics`, `chemistry`, `machine-learning`, or `general`. |
| `category` | string | More specific category token, for example `sequence-alignment`. |
| `summary` | string | One-sentence human-facing description. |
| `keywords` | string array | Search keywords and aliases. |

The default `taf new` skeleton intentionally does not create `[meta]`.
Maintainers can add it when preparing an app for public Hub/index discovery.

For new public releases, app maintainers should keep discovery metadata in
`taffish.toml`. The official index may use `metadata-overrides.toml` only to
supplement already published immutable records or index-side display metadata.

`taffish-index` also accepts the richer Hub-side aliases `categories` and
`description`. `category` is normalized into `categories`, and `summary` is
normalized into `description`; generated index records keep both forms for
consumer compatibility.

## `[upstream]`

Optional. Describes the original source of the wrapped software.

Example:

```toml
[upstream]
name = "CD-HIT"
type = "github"
url = "https://github.com/weizhongli/cdhit"
homepage = "https://github.com/weizhongli/cdhit"
repository = "weizhongli/cdhit"
release_url = "https://github.com/weizhongli/cdhit/releases"
docker_image = "docker.io/example/cdhit:4.8.1"
version = "4.8.1"
license = "GPL-2.0"
citation = "Fu et al. 2012"
doi = "10.1093/bioinformatics/bts565"
pmid = "23060610"
```

Supported fields:

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | Upstream project name. |
| `type` | string | Source type. Recommended values: `official`, `github`, `gitlab`, `archive`, `docker`, `apt`, `conda`, `other`. |
| `url` | string | General upstream homepage, repository, or documentation URL. |
| `homepage` | string | Upstream homepage. |
| `repository` | string | Upstream repository, for example `weizhongli/cdhit`. |
| `repo` | string | Compatibility alias for `repository`; index output normalizes it to `repository`. |
| `release_url` | string | Upstream release page. |
| `docker_image` | string | Existing upstream Docker image. |
| `version` | string | Upstream version. |
| `license` | string | Upstream license. |
| `citation` | string | Citation text. |
| `doi` | string | DOI. |
| `pmid` | string | PubMed ID. |

If `[upstream]` is absent, or no valid fields exist, the index omits the
`upstream` field.

For new releases, prefer storing upstream metadata here. Index-side
`metadata-overrides.toml` can supplement historical immutable records, but it
does not replace the app-side source of truth for new packages.

## Tool Example

```toml
[package]
name = "blast"
kind = "tool"
version = "2.16.0"
release = 1
license = "Apache-2.0"
main = "src/main.taf"

[repository]
url = "https://github.com/taffish/blast"

[command]
name = "taf-blast"

[runtime]
pipe = true
command_mode = true

[container]
image = "ghcr.io/taffish/blast:2.16.0-r1"
dockerfile = "docker/Dockerfile"
build_platforms = "linux/amd64,linux/arm64"

[smoke]
backend = "docker"
timeout = 60
exist = ["blastn"]
test = ["blastn -help"]

[meta]
domain = "bioinformatics"
category = "sequence-alignment"
summary = "BLAST+ wrapper for sequence similarity search."
keywords = ["blast", "alignment", "sequence-search"]

[upstream]
name = "BLAST+"
type = "official"
url = "https://blast.ncbi.nlm.nih.gov/"
homepage = "https://blast.ncbi.nlm.nih.gov/"
version = "2.16.0"
```

## Flow Example

```toml
[package]
name = "qc-flow"
kind = "flow"
version = "0.1.0"
release = 1
license = "Apache-2.0"
main = "src/main.taf"

[repository]
url = "https://github.com/taffish/qc-flow"

[command]
name = "taf-qc-flow"

[runtime]
pipe = false
command_mode = false

[dependencies]
taf-fastqc = "0.12.1-r1"
taf-multiqc = "1.19-r1"
```

## Version And Tag Rules

`version` and `release` combine into a version id:

```text
<version>-r<release>
```

For example:

```text
2.16.0-r1
```

Git tags must add a leading `v`:

```text
v2.16.0-r1
```

Image tags should normally not add the leading `v`:

```text
ghcr.io/taffish/blast:2.16.0-r1
```

## Common Errors

- `command.name` does not start with `taf-`.
- `repository.url` points to an old organization or wrong repository.
- `release` is written as a string instead of an integer.
- `main` is not a `.taf` file.
- `docs/help.md` is missing.
- Dockerfile path is wrong.
- Container image tag does not match `version-release`.
- A containerized app has no `[smoke]` section.
- `[smoke]` still contains a default `TODO` placeholder.
- A flow uses `[[taf: ...]]`, but `[dependencies]` is missing or version-mismatched.
- `[upstream]` is used for the app's own GitHub repository instead of the original wrapped software.
