# TAFFISH Documentation

[English](README.md) | [中文](README.cn.md)

This repository contains the developer-facing documentation for TAFFISH and
TAFFISH Hub.

TAFFISH is a lightweight command delivery system for bioinformatics tools and
workflows. TAFFISH Hub is the curated registry and index layer used by `taf` to
discover, install, and manage TAFFISH apps.

## Table of Contents

- [Start Here](#start-here)
- [Developer Guides](#developer-guides)
- [Specifications](#specifications)
- [Repository Layout](#repository-layout)
- [Online Hub](#online-hub)
- [Documentation Policy](#documentation-policy)
- [Status](#status)

## Start Here

| Document | Purpose |
| --- | --- |
| [What Is TAFFISH](en/taffish.en.md) | Language, compiler, CLI, app project structure, parameter system, container tags, and the recommended development workflow. |
| [What Is TAFFISH Hub](en/taffish-hub.en.md) | Hub architecture, GitHub-based automation, index generation, web registry, dependency handling, and publication policy. |

Read these two documents first if you are new to the project.

## Developer Guides

| Document | Purpose |
| --- | --- |
| [TAFFISH App Developer Guide](en/app-developer-guide.en.md) | Practical workflow for creating, checking, running, building, publishing, and maintaining TAFFISH apps. |
| [Containerized App Best Practices](en/container-apps.en.md) | Dockerfile design, multi-stage builds, GHCR visibility, local Docker/Podman testing, and backend consistency. |
| [Flow And Dependencies Guide](en/flow-dependencies.en.md) | Flow app structure, `[[taf: ...]]`, `@:` parameter blocks, dependency declarations, multi-version dependencies, and install semantics. |

Use these guides when you are actively building or maintaining apps.

## Specifications

| Document | Purpose |
| --- | --- |
| [`taffish.toml` Specification](en/taffish-toml-spec.en.md) | Metadata fields consumed by `taf`, TAFFISH Hub, and the index builder. |
| [TAFFISH Index JSON Specification](en/index-json-spec.en.md) | Static index schema consumed by `taf update`, `taf search`, `taf info`, and `taf install`. |

Use these documents when implementing tools, automation, validators, or Hub
consumers.

## Repository Layout

```text
./
  README.md
  README.cn.md
  en/
    *.en.md
  zh/
    *.cn.md
```

The default README is English. Chinese files use the `.cn.md` suffix, and
English files use the `.en.md` suffix.

## Online Hub

- [taffish.github.io](https://taffish.github.io)

The web Hub browses the same static index that `taf` consumes.

## Documentation Policy

These documents describe the current design and conventions of TAFFISH and
TAFFISH Hub. They are maintained as source documentation, not generated artifacts
bound to one specific `taf` or `taffish` binary version.

When exact behavior matters, prefer the corresponding repository code and release
notes as the final source of truth. Documentation history is tracked through
GitHub commit history. If a future document applies only to a specific version,
that constraint should be stated explicitly at the top of that document.

## Status

The official TAFFISH Hub is currently curated by the `taffish` GitHub
organization. It is not an open self-service publishing platform yet. Developers
who want to publish apps into the official Hub should contact the maintainer for
review or organization membership.
