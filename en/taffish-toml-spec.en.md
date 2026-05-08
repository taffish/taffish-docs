# `taffish.toml` Specification

`taffish.toml` is the core metadata file for a TAFFISH app. `taf check`, `taf build`, `taf publish`, `taffish-index`, and TAFFISH Hub all depend on it to understand an app.

## Table Of Contents

- [Format](#format)
- [`[package]`](#package)
- [`[repository]`](#repository)
- [`[command]`](#command)
- [`[runtime]`](#runtime)
- [`[container]`](#container)
- [`[dependencies]`](#dependencies)
- [`[platform]`](#platform)
- [`[upstream]`](#upstream)
- [Tool Example](#tool-example)
- [Flow Example](#flow-example)
- [Common Errors](#common-errors)

## Format

`taffish.toml` uses a small TOML subset:

- Tables use `[section]`.
- Values are strings, integers, booleans, or string arrays.
- Table arrays are not required for current official app metadata.
- Field names are lowercase with underscores where needed.

Example:

```toml
[package]
name = "my-tool"
kind = "tool"
version = "0.1.0"
release = 1
license = "Apache-2.0"
main = "src/main.taf"
```

## `[package]`

Required.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `name` | string | yes | Package name. Usually lowercase with hyphens. |
| `kind` | string | yes | `tool` or `flow`. |
| `version` | string | yes | Upstream or app version, without release suffix. |
| `release` | integer | yes | TAFFISH packaging release number. |
| `license` | string | yes | License identifier or license description. |
| `main` | string | yes | Project-relative `.taf` source path. |

Version id is derived from:

```text
<version>-r<release>
```

Example:

```toml
version = "2.16.0"
release = 1
```

Version id:

```text
2.16.0-r1
```

Git tag:

```text
v2.16.0-r1
```

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

For official Hub packages, this should normally be under the `taffish` GitHub organization.

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

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `image` | string | yes, if `[container]` exists | Container image reference. |
| `dockerfile` | string | no | Project-relative Dockerfile path. |
| `build_platforms` | string | no | Comma-separated platform list. |

Example:

```toml
[container]
image = "ghcr.io/taffish/my-tool:0.1.0-r1"
dockerfile = "docker/Dockerfile"
build_platforms = "linux/amd64,linux/arm64"
```

Rules:

- `image` tag should match `<version>-r<release>`.
- `dockerfile` should point to a file in the repository.
- `build_platforms` should use OCI platform names such as `linux/amd64`.

`[container]` describes the image metadata. The actual runtime tag still lives in `src/main.taf`, for example:

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --help
```

## `[dependencies]`

Optional. Mainly used by flow apps.

Single dependency version:

```toml
[dependencies]
taf-fastqc = "0.12.1-r1"
```

Multiple required versions of the same dependency:

```toml
[dependencies]
taf-align = ["2.0.0-r1", "2.1.0-r1"]
```

Rules:

- Key is the base command name, such as `taf-fastqc`.
- Value is a version id string or an array of version id strings.
- Array means every listed version is required. It does not mean alternatives.
- Official flow releases should use exact versions.

`taf build` can sync dependencies from versioned `[[taf: ...]]` calls in `src/main.taf`.

## `[platform]`

Optional. Describes platform constraints for the app.

Example:

```toml
[platform]
os = "linux"
arch = "x86_64"
container = true
```

Recommended fields:

| Field | Type | Description |
| --- | --- | --- |
| `os` | string | Operating system constraint, such as `linux`, `macos`, or `any`. |
| `arch` | string | Architecture constraint, such as `x86_64`, `arm64`, or `any`. |
| `container` | boolean | Whether the app expects a container backend. |

When absent, the app is treated as having no declared platform constraint.

## `[upstream]`

Optional. Describes the original software, database, workflow, or project being wrapped by the TAFFISH app.

Example:

```toml
[upstream]
name = "BLAST"
version = "2.16.0"
url = "https://blast.ncbi.nlm.nih.gov/"
repository = "https://github.com/ncbi/blast_plus_docs"
license = "Public Domain"
description = "NCBI BLAST+ sequence alignment tools."
```

Supported fields:

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | Upstream project name. |
| `version` | string | Upstream version. |
| `url` | string | Upstream homepage or download page. |
| `repository` | string | Upstream source repository. |
| `license` | string | Upstream license. |
| `description` | string | Short upstream description. |

`[upstream]` describes the original software being wrapped. It is not the TAFFISH app repository. If the fields are absent, TAFFISH Hub simply omits upstream metadata.

## Tool Example

```toml
[package]
name = "blast"
kind = "tool"
version = "2.16.0"
release = 1
license = "Public Domain"
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

[upstream]
name = "BLAST"
version = "2.16.0"
url = "https://blast.ncbi.nlm.nih.gov/"
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

## Common Errors

Missing required field:

- Check `[package]`, `[repository]`, `[command]`, and `[runtime]`.

Wrong version id:

- `version` should not include `-r1`.
- `release` should be an integer.
- The derived version id is `version + "-r" + release`.

Image mismatch:

- `[container].image` tag should match the derived version id.
- `src/main.taf` should use the same image tag.

Flow dependency mismatch:

- `src/main.taf` uses `[[taf: ...]]`, but `[dependencies]` is missing or out of sync.
- Run `taf build` to sync dependencies.

Wrong dependency array semantics:

- Arrays mean all listed versions are required.
- Arrays are not alternatives.

