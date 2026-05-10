# TAFFISH Documentation

[English](README.md) | [中文](README.cn.md)

This repository contains the developer-facing documentation for TAFFISH,
TAFFISH Hub, and the local `taffish-mcp` AI integration entry point.

TAFFISH is a lightweight command delivery system for bioinformatics tools and
workflows. TAFFISH Hub is the curated registry and index layer used by `taf` to
discover, install, and manage TAFFISH apps.

## Table of Contents

- [Recommended Reading Paths](#recommended-reading-paths)
- [Document Roles](#document-roles)
- [Core Concepts](#core-concepts)
- [User Guides](#user-guides)
- [Developer Guides](#developer-guides)
- [Specifications](#specifications)
- [How The Documents Overlap](#how-the-documents-overlap)
- [Repository Layout](#repository-layout)
- [Online Hub](#online-hub)
- [Documentation Policy](#documentation-policy)
- [Status](#status)

## Recommended Reading Paths

Choose a path based on what you want to do.

| Goal | Recommended order |
| --- | --- |
| Use existing TAFFISH apps | [Quick Start](en/quick-start.en.md) -> [Troubleshooting](en/troubleshooting.en.md) when needed -> [What Is TAFFISH Hub](en/taffish-hub.en.md) for background |
| Learn the TAFFISH language | [What Is TAFFISH](en/taffish.en.md) -> [TAF Script Tutorial](en/taf-script-tutorial.en.md) |
| Write a simple tool app | [TAF Script Tutorial](en/taf-script-tutorial.en.md) -> [App Developer Guide](en/app-developer-guide.en.md) -> [`taffish.toml` Specification](en/taffish-toml-spec.en.md) |
| Package a containerized bioinformatics tool | [App Developer Guide](en/app-developer-guide.en.md) -> [Containerized App Best Practices](en/container-apps.en.md) -> [Troubleshooting](en/troubleshooting.en.md) |
| Write a flow app with dependencies | [TAF Script Tutorial](en/taf-script-tutorial.en.md) -> [Flow And Dependencies Guide](en/flow-dependencies.en.md) -> [`taffish.toml` Specification](en/taffish-toml-spec.en.md) |
| Understand Hub and index internals | [What Is TAFFISH Hub](en/taffish-hub.en.md) -> [TAFFISH Index JSON Specification](en/index-json-spec.en.md) -> [`taffish.toml` Specification](en/taffish-toml-spec.en.md) |
| Connect TAFFISH to an AI client | [TAFFISH MCP Guide](en/taffish-mcp.en.md) -> [What Is TAFFISH](en/taffish.en.md#mcp--ai-integration) for background |

If you are completely new, read [Quick Start](en/quick-start.en.md) first, then
[What Is TAFFISH](en/taffish.en.md).

## Document Roles

| Document | Role |
| --- | --- |
| [Quick Start](en/quick-start.en.md) | User onboarding. It intentionally repeats install, mirror config, update, search, install, run, list, locate, and uninstall basics. |
| [What Is TAFFISH](en/taffish.en.md) | Conceptual language and CLI manual. It explains the design, syntax, tags, parameters, project structure, CLI surface, and MCP/AI integration entry point. |
| [TAF Script Tutorial](en/taf-script-tutorial.en.md) | Hands-on `.taf` writing path. It teaches by building up from small scripts to app wrappers and flows. |
| [TAFFISH MCP Guide](en/taffish-mcp.en.md) | AI-client integration guide for `taffish-mcp`, including client config, tools, resources, prompts, safety boundaries, and troubleshooting. |
| [App Developer Guide](en/app-developer-guide.en.md) | Practical app release workflow. It focuses on `taf new`, project editing, check, run, build, `release.md`, publish, and maintenance. |
| [Containerized App Best Practices](en/container-apps.en.md) | Focused container guide. It covers Dockerfile design, runtime mounts, GHCR, Docker/Podman testing, and backend consistency. |
| [Flow And Dependencies Guide](en/flow-dependencies.en.md) | Focused flow guide. It covers `[[taf: ...]]`, `@:` blocks, exact app versions, and dependency semantics. |
| [`taffish.toml` Specification](en/taffish-toml-spec.en.md) | Metadata reference for app authors, Hub maintainers, and validators. |
| [TAFFISH Index JSON Specification](en/index-json-spec.en.md) | Machine-readable index reference for `taf`, Hub automation, and index consumers. |
| [Troubleshooting](en/troubleshooting.en.md) | Problem-oriented reference. Start here when commands, containers, GHCR, GitHub, or wrappers fail. |

## Core Concepts

| Document | Purpose |
| --- | --- |
| [What Is TAFFISH](en/taffish.en.md) | Language, compiler, CLI, `taffish-mcp`, app project structure, parameter system, container tags, and the recommended development workflow. |
| [What Is TAFFISH Hub](en/taffish-hub.en.md) | Hub architecture, GitHub-based automation, index generation, web registry, dependency handling, and publication policy. |

Read these two documents when you want the system-level picture.

## User Guides

| Document | Purpose |
| --- | --- |
| [TAFFISH Quick Start](en/quick-start.en.md) | Install TAFFISH, update the Hub index, search, install, run, list, locate, and uninstall apps. |
| [TAFFISH MCP Guide](en/taffish-mcp.en.md) | Configure `taffish-mcp` for AI clients and understand its safety model. |
| [TAF Script Tutorial](en/taf-script-tutorial.en.md) | Step-by-step `.taf` writing tutorial for app authors, from minimal scripts to parameters, containers, flows, and dependencies. |
| [TAFFISH Troubleshooting](en/troubleshooting.en.md) | Common installation, index, mirror config, container, GHCR, Podman, Docker, Apptainer, and wrapper problems. |

Use these guides when you want a practical path from first install to daily use.

## Developer Guides

| Document | Purpose |
| --- | --- |
| [TAFFISH App Developer Guide](en/app-developer-guide.en.md) | Practical workflow for creating, checking, running, building, preparing `release.md`, publishing, and maintaining TAFFISH apps. |
| [Containerized App Best Practices](en/container-apps.en.md) | Dockerfile design, multi-stage builds, runtime mounts, GHCR visibility, local Docker/Podman testing, and backend consistency. |
| [Flow And Dependencies Guide](en/flow-dependencies.en.md) | Flow app structure, `[[taf: ...]]`, `@:` parameter blocks, dependency declarations, multi-version dependencies, and install semantics. |

Use these guides when you are actively building or maintaining apps.

## Specifications

| Document | Purpose |
| --- | --- |
| [`taffish.toml` Specification](en/taffish-toml-spec.en.md) | Metadata fields consumed by `taf`, TAFFISH Hub, and the index builder. |
| [TAFFISH Index JSON Specification](en/index-json-spec.en.md) | Static index schema consumed by `taf update`, `taf search`, `taf info`, and `taf install`. |

Use these documents when implementing tools, automation, validators, or Hub
consumers.

## How The Documents Overlap

Some repetition is intentional:

- Install and basic `taf` commands appear in both Quick Start and What Is
  TAFFISH so users can start quickly and later understand the full model.
- Runtime mirror configuration appears in Quick Start, What Is TAFFISH, What Is
  TAFFISH Hub, and troubleshooting because it affects both network access and
  package installation.
- `taffish-mcp` appears briefly in What Is TAFFISH, while the MCP guide is the
  focused source for AI-client configuration and safety boundaries.
- `taf run`, `taf build`, and `taf publish` appear in the app developer guide
  and the language manual because they connect syntax to project lifecycle.
- Container backend notes appear in Quick Start, container best practices, and
  troubleshooting because runtime issues are common in real deployments.
- Flow dependencies appear in the language manual, app developer guide, and the
  focused flow guide; the focused guide is the most detailed source.

As a rule: use tutorials to learn, guides to work, specifications to check exact
fields, and troubleshooting to diagnose failures.

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
