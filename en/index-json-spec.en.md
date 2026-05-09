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
- [`commands`](#commands)
- [`repositories`](#repositories)
- [`warnings`](#warnings)
- [Dependency Semantics](#dependency-semantics)
- [Upstream Omission Rules](#upstream-omission-rules)
- [Compatibility Principles](#compatibility-principles)

## File Locations

The `taffish-index` repository generates:

```text
index/index.json
index/packages/<package>.json
index/commands/<command>.json
```

Users download this by default:

```text
https://raw.githubusercontent.com/taffish/taffish-index/main/index/index.json
```

TAFFISH `0.3.0` can read a different index URL from runtime config or a one-off
`taf update --url <INDEX-URL>` override. A mirrored index should keep the same
schema and canonical source records. Repository URL rewriting belongs to local
`taf` config, not to the index schema itself.

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
  "warnings": 0
}
```

Fields:

- `packages`: number of packages.
- `versions`: total number of version records.
- `commands`: number of commands.
- `repositories`: number of repositories.
- `warnings`: number of warnings.

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
  "paths": {
    "main": "src/main.taf",
    "help": "docs/help.md",
    "dockerfile": "docker/Dockerfile"
  },
  "container": {
    "image": "ghcr.io/taffish/blast:2.16.0-r1",
    "dockerfile": "docker/Dockerfile",
    "image_tag": "2.16.0-r1",
    "image_tag_matches_version": true
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
| `paths` | object | Project-relative paths. |
| `container` | object/null | Container metadata. |
| `source` | object | App repository source information. |
| `upstream` | object | Optional upstream software source. |

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

If `taffish.toml` has no `[upstream]`, or `[upstream]` has no valid field, the
version record does not contain `upstream`.

When upstream metadata exists:

```json
"upstream": {
  "name": "CD-HIT",
  "type": "github",
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
- not display missing `upstream` as "no source"
- not interpret dependency arrays as alternatives
- use `source.ref` and `source.commit` to track app source
- use `upstream` to track the wrapped software source
