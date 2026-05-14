# TAFFISH Index JSON Specification

TAFFISH Index JSON is the machine-readable data format of TAFFISH Hub.
`taf update` downloads it, `taf install` reads it, and `taffish.github.io`
displays it.

Current schema version:

```text
taffish.index/v1
```

## Table Of Contents

- [File Locations](#file-locations)
- [Top-Level Object](#top-level-object)
- [`counts`](#counts)
- [`packages`](#packages)
- [Version Record](#version-record)
- [Meta](#meta)
- [Metadata Overrides](#metadata-overrides)
- [Container, Smoke, And Trust](#container-smoke-and-trust)
- [`commands`](#commands)
- [`repositories`](#repositories)
- [`warnings`](#warnings)
- [Build Reports](#build-reports)
- [Dependency Semantics](#dependency-semantics)
- [Upstream Omission Rules](#upstream-omission-rules)
- [Compatibility Principles](#compatibility-principles)

## File Locations

The `taffish-index` repository generates:

```text
index/index.json
index/packages/<package>.json
index/commands/<command>.json
index/reports/latest.json
index/reports/<timestamp>.json
```

Users download this by default:

```text
https://raw.githubusercontent.com/taffish/taffish-index/main/index/index.json
```

Since TAFFISH `0.2.0`, `taf` can read a different index URL from runtime config
or a one-off `taf update --url <INDEX-URL>` override. A mirrored index should
keep the same schema and canonical source records. Repository URL rewriting
belongs to local `taf` config, not to the index schema itself.

`index/index.json` is the complete index. Split files are used for more granular
access.

## Top-Level Object

```json
{
  "schema_version": "taffish.index/v1",
  "generated_at": "2026-05-08T00:00:00Z",
  "organization": "taffish",
  "counts": {},
  "packages": {},
  "commands": {},
  "repositories": {},
  "warnings": []
}
```

Fields:

| Field | Type | Description |
| --- | --- | --- |
| `schema_version` | string | Index schema version. |
| `generated_at` | string | UTC generation time. |
| `organization` | string/null | Scanned GitHub organization. |
| `counts` | object | Summary counts. |
| `packages` | object | Package index. |
| `commands` | object | Command-to-package index. |
| `repositories` | object | Repository-to-package index. |
| `warnings` | array | Build-time warnings. |

## `counts`

```json
{
  "packages": 12,
  "versions": 27,
  "commands": 12,
  "repositories": 12,
  "warnings": 0,
  "failed": 0
}
```

Fields:

- `packages`: number of packages.
- `versions`: total number of version records.
- `commands`: number of commands.
- `repositories`: number of repositories.
- `warnings`: number of warnings.
- `failed`: number of trust-gate failures written to the latest build report.

## `packages`

`packages` is an object keyed by package name.

```json
{
  "blast": {
    "name": "blast",
    "latest": "2.16.0-r1",
    "repository_url": "https://github.com/taffish/blast",
    "command": {
      "name": "taf-blast"
    },
    "versions": {}
  }
}
```

Package entry fields:

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | Package name. |
| `latest` | string | Latest version id. |
| `repository_url` | string | App GitHub repository. |
| `command.name` | string | Base command name. |
| `versions` | object | Mapping from version id to version record. |

## Version Record

A version record describes one concrete app version.

```json
{
  "name": "blast",
  "kind": "tool",
  "version": "2.16.0",
  "release": 1,
  "version_id": "2.16.0-r1",
  "tag": "v2.16.0-r1",
  "license": "Apache-2.0",
  "repository_url": "https://github.com/taffish/blast",
  "repository_slug": "taffish/blast",
  "command": {
    "name": "taf-blast"
  },
  "runtime": {
    "pipe": true,
    "command_mode": true
  },
  "dependencies": {},
  "platform": {
    "os": [],
    "arch": [],
    "container": "optional",
    "min_cpus": null,
    "min_memory_mb": null
  },
  "meta": {
    "domain": "bioinformatics",
    "category": "sequence-alignment",
    "categories": ["sequence-alignment"],
    "keywords": ["blast", "alignment", "sequence-search"],
    "summary": "BLAST+ wrapper for sequence similarity search.",
    "description": "BLAST+ wrapper for sequence similarity search."
  },
  "paths": {
    "main": "src/main.taf",
    "help": "docs/help.md",
    "dockerfile": "docker/Dockerfile"
  },
  "container": {
    "image": "ghcr.io/taffish/blast:2.16.0-r1",
    "dockerfile": "docker/Dockerfile",
    "image_tag": "2.16.0-r1",
    "image_tag_matches_version": true,
    "digest": "sha256:manifest-list-digest",
    "platforms": ["linux/amd64", "linux/arm64"],
    "platform_digests": {
      "linux/amd64": "sha256:...",
      "linux/arm64": "sha256:..."
    }
  },
  "smoke": {
    "backend": "docker",
    "timeout": 60,
    "exist": ["blastn"],
    "test": ["blastn -help"],
    "status": "passed",
    "checked_at": "2026-05-12T08:00:00Z",
    "backend_used": "docker"
  },
  "trust": {
    "status": "passed",
    "checked_at": "2026-05-12T08:00:00Z",
    "policy": "taffish.index/trust-v1",
    "source": "taffish-index"
  },
  "source": {
    "repository": "taffish/blast",
    "ref": "v2.16.0-r1",
    "commit": "abcdef...",
    "html_url": "https://github.com/taffish/blast/tree/v2.16.0-r1"
  }
}
```

Field descriptions:

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | Package name. |
| `kind` | string | `tool` or `flow`. |
| `version` | string | App version. |
| `release` | number | App release. |
| `version_id` | string | `<version>-r<release>`. |
| `tag` | string | Git tag, format `v<version>-r<release>`. |
| `license` | string/null | App wrapper license. |
| `repository_url` | string | App repository URL. |
| `repository_slug` | string | `owner/repo`. |
| `command.name` | string | Base command name. |
| `runtime.pipe` | boolean | Whether pipe mode is supported. |
| `runtime.command_mode` | boolean | Whether the app is command mode. |
| `dependencies` | object | Flow dependencies. |
| `platform` | object | Platform constraints. |
| `meta` | object/null | Optional discovery metadata for search, filtering, and display. |
| `paths` | object | Project-relative paths. |
| `container` | object/null | Container metadata. |
| `smoke` | object/null | Smoke metadata and result for containerized apps. |
| `trust` | object/null | Hub/index trust-gate status. |
| `source` | object | App repository source information. |
| `upstream` | object | Optional upstream software source. |

## Meta

`meta` records optional discovery metadata copied from `[meta]` in
`taffish.toml` or from index maintainer metadata overrides. It is for search,
categorization, and Hub display; installers should treat it as optional.

Current fields:

| Field | Type | Description |
| --- | --- | --- |
| `domain` | string/null | Broad domain, such as `bioinformatics`. |
| `category` | string/null | Primary category token. |
| `categories` | array | Category tokens used for filtering and browsing. |
| `keywords` | array | Search keywords and aliases. |
| `summary` | string/null | One-sentence description. |
| `description` | string/null | Same human-facing description, kept for Hub compatibility. |

TAFFISH `0.8.1` documents the compact app-side fields `category` and
`summary`. `taffish-index` also accepts the richer Hub-side aliases
`categories` and `description`, then emits both forms so older and newer
consumers can read the same record.

## Metadata Overrides

Official index maintainers may supplement already published, immutable version
records through `metadata-overrides.toml`. Overrides are intended for display
and discovery metadata only, especially `meta` and `upstream` fields such as
license, citation, DOI, and PMID.

Overrides do not change installable app identity: version ids, release tags,
source refs, container image tags, image digests, platform lists, smoke results,
and trust records still come from the app repository and index checks.

For new app releases, prefer writing `[meta]` and `[upstream]` directly in
`taffish.toml`. Use metadata overrides for historical records or index-side
metadata corrections that do not justify republishing the app.

## Container, Smoke, And Trust

`container` fields:

| Field | Type | Description |
| --- | --- | --- |
| `image` | string | Container image reference. |
| `dockerfile` | string/null | Project-relative Dockerfile path. |
| `image_tag` | string/null | Image tag parsed from `image`. |
| `image_tag_matches_version` | boolean | Whether the image tag matches `version_id`. |
| `digest` | string/null | Manifest-list digest recorded by the index builder. |
| `platforms` | array | Platforms reported for the image, such as `linux/amd64`. |
| `platform_digests` | object | Optional mapping from platform to per-platform digest. |

`smoke` fields:

| Field | Type | Description |
| --- | --- | --- |
| `backend` | string | Requested smoke backend: `docker`, `podman`, or `apptainer`. |
| `timeout` | number | Per smoke command timeout in seconds. |
| `exist` | array | Executables that should exist in the container `PATH`. |
| `test` | array | Commands that should exit with status `0`. |
| `status` | string/null | Usually `passed` for records in the main index. |
| `checked_at` | string/null | UTC smoke check time. |
| `backend_used` | string/null | Backend actually used by the index builder. |

`trust.status` values currently used by the official index:

- `passed`: container digest/platform inspection and smoke checks passed.
- `not_applicable`: non-container app version; no container smoke gate applies.

Failed new versions are not written to the main index. They appear in build
reports instead.

## `commands`

`commands` maps command names to package and latest version.

```json
{
  "taf-blast": {
    "package": "blast",
    "version": "2.16.0-r1"
  }
}
```

Used by:

- `taf install taf-blast`
- `taf info taf-blast`
- `taf search`

## `repositories`

`repositories` maps GitHub repositories to packages.

```json
{
  "taffish/blast": {
    "repository": "taffish/blast",
    "packages": ["blast"]
  }
}
```

A repository usually contains one package, but the format allows one repository
to contain multiple packages.

## `warnings`

Non-fatal problems encountered during index generation are written to
`warnings`:

```json
[
  {
    "repository": "taffish/example",
    "ref": "v0.1.0-r1",
    "message": "missing docs/help.md"
  }
]
```

Warnings do not prevent other valid apps from being written into the index. The
web UI can display them, and maintainers should review them regularly.

## Build Reports

The index builder also writes reports:

```text
index/reports/latest.json
index/reports/<timestamp>.json
```

Report schema:

```json
{
  "schema_version": "taffish.index.report/v1",
  "generated_at": "2026-05-12T08:00:00Z",
  "organization": "taffish",
  "counts": {
    "failed": 1,
    "warnings": 0
  },
  "failed": [
    {
      "repository": "taffish/my-tool",
      "ref": "v0.1.0-r1",
      "version_id": "0.1.0-r1",
      "stage": "smoke",
      "message": "smoke command failed",
      "image": "ghcr.io/taffish/my-tool:0.1.0-r1"
    }
  ],
  "warnings": []
}
```

`taf update` and `taf install` consume the stable main index, not report files.
Reports are for maintainers and future Hub UI diagnostics.

## Dependency Semantics

No dependencies:

```json
"dependencies": {}
```

Single-version dependency:

```json
"dependencies": {
  "taf-dep-tool": "0.1.0-r1"
}
```

Multi-version dependency:

```json
"dependencies": {
  "taf-x": ["0.1.0-r1", "0.1.0-r2"]
}
```

Multi-version dependency means every listed version needs to be installed. It
does not mean alternative versions.

`latest` and `*` may be treated by installers as unspecified versions, but
official flows should use exact version ids.

## Upstream Omission Rules

If neither `taffish.toml` nor index metadata overrides provide valid upstream
fields, the version record does not contain `upstream`.

When upstream metadata exists:

```json
"upstream": {
  "name": "CD-HIT",
  "type": "github",
  "url": "https://github.com/weizhongli/cdhit",
  "homepage": "https://github.com/weizhongli/cdhit",
  "repository": "weizhongli/cdhit",
  "version": "4.8.1",
  "license": "GPL-2.0",
  "doi": "10.1093/bioinformatics/bts565",
  "pmid": "23060610"
}
```

Consumers should not assume that `upstream` always exists.

## Compatibility Principles

Index JSON consumers should:

- check `schema_version`
- tolerate unknown fields
- tolerate missing optional fields
- treat `meta` as optional discovery metadata
- read `category`/`summary` and `categories`/`description` compatibly
- not display missing `upstream` as "no source"
- not interpret dependency arrays as alternatives
- use `source.ref` and `source.commit` to track app source
- use `upstream` to track the wrapped software source
- treat missing `smoke` or `trust` as unknown/legacy metadata, not as proof of failure
- read reports separately from the main installable index
