# TAFFISH Documentation

[English](README.md) | [中文](README.cn.md)

This repository contains public documentation for TAFFISH, TAFFISH Hub,
taf-app development, the static index, and the local `taffish-mcp` AI
integration entry point.

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
- [Security Model](#security-model)
- [Source Development](#source-development)
- [How The Documents Overlap](#how-the-documents-overlap)
- [Repository Layout](#repository-layout)
- [Online Hub](#online-hub)
- [Documentation Policy](#documentation-policy)
- [License](#license)
- [Status](#status)

## Recommended Reading Paths

Choose a path based on what you want to do.

| Goal | Recommended order |
| --- | --- |
| Use existing TAFFISH apps | [Quick Start](en/quick-start.en.md) -> [Troubleshooting](en/troubleshooting.en.md) when needed -> [What Is TAFFISH Hub](en/taffish-hub.en.md) for background |
| Learn the TAFFISH language | [What Is TAFFISH](en/taffish.en.md) -> [TAF Script Tutorial](en/taf-script-tutorial.en.md) |
| Write a simple tool app | [TAF Script Tutorial](en/taf-script-tutorial.en.md) -> [App Developer Guide](en/app-developer-guide.en.md) -> [`taffish.toml` Specification](en/taffish-toml-spec.en.md) |
| Package a containerized bioinformatics tool | [App Developer Guide](en/app-developer-guide.en.md) -> [Containerized App Best Practices](en/container-apps.en.md) -> [Official taf-app Curation Guide](en/taf-app-curation-guide.en.md) -> [Troubleshooting](en/troubleshooting.en.md) |
| Curate an official Hub app | [Official taf-app Curation Guide](en/taf-app-curation-guide.en.md) -> [Augustus app template](https://github.com/taffish/augustus) -> [Containerized App Best Practices](en/container-apps.en.md) -> [`taffish.toml` Specification](en/taffish-toml-spec.en.md) |
| Write a flow app with dependencies | [TAF Script Tutorial](en/taf-script-tutorial.en.md) -> [Flow And Dependencies Guide](en/flow-dependencies.en.md) -> [`taffish.toml` Specification](en/taffish-toml-spec.en.md) |
| Understand Hub and index internals | [What Is TAFFISH Hub](en/taffish-hub.en.md) -> [TAFFISH Index JSON Specification](en/index-json-spec.en.md) -> [`taffish.toml` Specification](en/taffish-toml-spec.en.md) |
| Understand security and trust | [TAFFISH Security Model](en/security-model.en.md) -> [TAFFISH Index JSON Specification](en/index-json-spec.en.md) -> [TAFFISH MCP Guide](en/taffish-mcp.en.md) |
| Connect TAFFISH to an AI client | [Using TAFFISH MCP With AI Clients](en/mcp-clients.en.md) -> [TAFFISH MCP Guide](en/taffish-mcp.en.md) -> [What Is TAFFISH](en/taffish.en.md#mcp--ai-integration) for background |
| Build or modify TAFFISH itself | [taffish/taffish README](https://github.com/taffish/taffish) -> [Build From Source](https://github.com/taffish/taffish/blob/main/docs/dev/en/build-from-source.md) -> [Contributing](https://github.com/taffish/taffish/blob/main/CONTRIBUTING.md) |

If you are completely new, read [Quick Start](en/quick-start.en.md) first, then
[What Is TAFFISH](en/taffish.en.md).

## Document Roles

| Document | Role |
| --- | --- |
| [Quick Start](en/quick-start.en.md) | User onboarding. It intentionally repeats install, mirror config, update, search, install, run, list, locate, and uninstall basics. |
| [What Is TAFFISH](en/taffish.en.md) | Conceptual language and CLI manual. It explains the design, syntax, tags, parameters, project structure, CLI surface, and MCP/AI integration entry point. |
| [TAF Script Tutorial](en/taf-script-tutorial.en.md) | Hands-on `.taf` writing path. It teaches by building up from small scripts to app wrappers and flows. |
| [TAFFISH MCP Guide](en/taffish-mcp.en.md) | Capability reference for `taffish-mcp`, including tools, read-only compiler helpers, app/project inspection, smoke/trust metadata exposure, resources, prompts, safety boundaries, and troubleshooting. |
| [Using TAFFISH MCP With AI Clients](en/mcp-clients.en.md) | Client setup guide for Codex, Claude Code, Cursor, Cline, and generic stdio MCP clients. |
| [App Developer Guide](en/app-developer-guide.en.md) | Practical app release workflow. It focuses on `taf new`, project editing, check, run, build, `release.md`, publish, and maintenance. |
| [Containerized App Best Practices](en/container-apps.en.md) | Focused container guide. It covers Dockerfile design, smoke metadata, runtime mounts, GHCR, Docker/Podman testing, and backend consistency. |
| [Official taf-app Curation Guide](en/taf-app-curation-guide.en.md) | Official Hub maintainer guide. It uses Augustus as the baseline template and covers file boundaries, help/README/release/smoke design, version policy, publish checks, and what not to edit. |
| [Flow And Dependencies Guide](en/flow-dependencies.en.md) | Focused flow guide. It covers `[[taf: ...]]`, `@:` blocks, exact app versions, and dependency semantics. |
| [`taffish.toml` Specification](en/taffish-toml-spec.en.md) | Metadata reference for app authors, Hub maintainers, and validators, including `[smoke]` metadata for containerized apps. |
| [TAFFISH Index JSON Specification](en/index-json-spec.en.md) | Machine-readable index reference for `taf`, Hub automation, index consumers, trust metadata, container digests, smoke results, and build reports. |
| [TAFFISH Security Model](en/security-model.en.md) | Layered trust model covering source code, release payload integrity, installers, mirrors, Hub index gates, local install verification, containers, and MCP/AI boundaries. |
| [Troubleshooting](en/troubleshooting.en.md) | Problem-oriented reference. Start here when commands, smoke metadata, containers, GHCR, GitHub, or wrappers fail. |

Maintenance rule: field-level truth belongs in the
[`taffish.toml` Specification](en/taffish-toml-spec.en.md); index output truth
belongs in the [TAFFISH Index JSON Specification](en/index-json-spec.en.md);
container practice belongs in [Containerized App Best Practices](en/container-apps.en.md);
official Hub app checklists belong in the
[Official taf-app Curation Guide](en/taf-app-curation-guide.en.md); concrete
failure handling belongs in [Troubleshooting](en/troubleshooting.en.md).

## Core Concepts

| Document | Purpose |
| --- | --- |
| [What Is TAFFISH](en/taffish.en.md) | Language, compiler, CLI, `taffish-mcp`, app project structure, parameter system, container tags, and the recommended development workflow. |
| [What Is TAFFISH Hub](en/taffish-hub.en.md) | Hub architecture, GitHub-based automation, index generation, web registry, dependency handling, and publication policy. |

Read these two documents when you want the system-level picture.

## User Guides

| Document | Purpose |
| --- | --- |
| [TAFFISH Quick Start](en/quick-start.en.md) | Install TAFFISH, update the Hub index, search, install public apps, install private/local apps with `taf install --from`, run, list, locate, and uninstall apps. |
| [Using TAFFISH MCP With AI Clients](en/mcp-clients.en.md) | Configure `taffish-mcp` in Codex, Claude Code, Cursor, Cline, or a generic stdio MCP client. |
| [TAFFISH MCP Guide](en/taffish-mcp.en.md) | Understand `taffish-mcp` tools, read-only compiler helpers, app/project inspection, smoke/trust metadata exposure, resources, prompts, and safety model. |
| [TAF Script Tutorial](en/taf-script-tutorial.en.md) | Step-by-step `.taf` writing tutorial for app authors, from minimal scripts to parameters, containers, flows, and dependencies. |
| [TAFFISH Troubleshooting](en/troubleshooting.en.md) | Common installation, index, mirror config, smoke metadata, container, GHCR, Podman, Docker, Apptainer, and wrapper problems. |

Use these guides when you want a practical path from first install to daily use.

## Developer Guides

| Document | Purpose |
| --- | --- |
| [TAFFISH App Developer Guide](en/app-developer-guide.en.md) | Practical workflow for creating, checking, running, building, private/local testing with `taf install --from`, preparing `release.md`, publishing, and maintaining TAFFISH apps. |
| [Containerized App Best Practices](en/container-apps.en.md) | Dockerfile design, multi-stage builds, smoke metadata, runtime mounts, GHCR visibility, local Docker/Podman testing, backend consistency, and `TAFFISH_CONTAINER_BACKEND`. |
| [Official taf-app Curation Guide](en/taf-app-curation-guide.en.md) | Curation guide for official Hub maintainers turning `taf new` projects into publishable, indexable apps that can serve as templates. |
| [Flow And Dependencies Guide](en/flow-dependencies.en.md) | Flow app structure, `[[taf: ...]]`, `@:` parameter blocks, dependency declarations, multi-version dependencies, and install semantics. |

Use these guides when you are actively building or maintaining apps.

## Specifications

| Document | Purpose |
| --- | --- |
| [`taffish.toml` Specification](en/taffish-toml-spec.en.md) | Metadata fields consumed by `taf`, TAFFISH Hub, and the index builder, including `[smoke]`, `[meta]`, and `[upstream]`. |
| [TAFFISH Index JSON Specification](en/index-json-spec.en.md) | Static index schema consumed by `taf update`, `taf search`, `taf info`, and `taf install`, including trust reports, container smoke metadata, and index-side metadata overrides. |

Use these documents when implementing tools, automation, validators, or Hub
consumers.

## Security Model

| Document | Purpose |
| --- | --- |
| [TAFFISH Security Model](en/security-model.en.md) | End-to-end overview of release integrity, installer behavior, mirror behavior, Hub/index trust gates, `source.commit` verification, container boundaries, and MCP safety boundaries. |

Use this document when reviewing TAFFISH for a collaboration, server deployment,
private mirror, enterprise environment, or security-sensitive workflow.

## Source Development

TAFFISH `0.8.1` is the current stable patch release in the first open-source
`0.8.x` local CLI/compiler series. The source code, ASDF systems,
source-tree developer docs, release payloads, contribution guide, and security
policy live in
[taffish/taffish](https://github.com/taffish/taffish).

The `0.8.1` release keeps the `0.8.0` public interface stable, fixes
`release.md` placeholder detection in `taf publish --release`, documents
optional `[meta]` and `[upstream]` metadata, and refreshes the signed checksum
release payload.

The most relevant source-side documents are:

| Document | Purpose |
| --- | --- |
| [Build From Source](https://github.com/taffish/taffish/blob/main/docs/dev/en/build-from-source.md) | Build `taf`, `taffish`, and `taffish-mcp` from the Common Lisp source tree with SBCL or LispWorks. |
| [Source-tree Developer Docs](https://github.com/taffish/taffish/tree/main/docs/dev/en) | Architecture notes for `taffish-core`, `taf-core`, `taffish-mcp`, `han`, ASDF systems, and release engineering. |
| [Contributing](https://github.com/taffish/taffish/blob/main/CONTRIBUTING.md) | Development setup, test expectations, code boundaries, documentation expectations, and release artifact policy. |
| [Security Policy](https://github.com/taffish/taffish/blob/main/SECURITY.md) | Private vulnerability reporting and the current release/index trust model. |

This `taffish-docs` repository remains focused on user-facing, app-author, Hub,
and index documentation. Compiler implementation notes should stay close to the
source repository.

## How The Documents Overlap

Some repetition is intentional:

- Install and basic `taf` commands appear in both Quick Start and What Is
  TAFFISH so users can start quickly and later understand the full model.
- Runtime mirror configuration appears in Quick Start, What Is TAFFISH, What Is
  TAFFISH Hub, and troubleshooting because it affects both network access and
  package installation.
- `taffish-mcp` appears briefly in What Is TAFFISH. The MCP guide documents
  the server capability surface, read-only compiler helpers, app/project inspection, smoke/trust metadata exposure, safe compile previews, and safety boundaries, while the client setup
  guide documents Codex, Claude Code, Cursor, Cline, and generic MCP
  configuration examples.
- `taf run`, `taf build`, and `taf publish` appear in the app developer guide
  and the language manual because they connect syntax to project lifecycle.
- The official taf-app curation guide overlaps with the app developer guide and
  container best practices, but acts more like a maintainer checklist for
  official Hub apps using Augustus as the baseline.
- Container backend notes appear in Quick Start, container best practices, and
  troubleshooting because runtime issues are common in real deployments.
- Smoke metadata appears in the app developer guide, container best practices,
  `taffish.toml` specification, Hub guide, index JSON specification, and MCP
  guide because TAFFISH `0.8.0` connects local app metadata with Hub-side
  validation and AI-assisted inspection.
- Security appears as its own model document because release verification,
  mirrors, index trust gates, local install checks, containers, and MCP
  boundaries cut across several repositories.
- Source-building and compiler-internal documentation are linked here, but live
  in `taffish/taffish` so they evolve with the Common Lisp source tree.
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

## License

The documentation in this repository is licensed under the
[Creative Commons Attribution 4.0 International License](LICENSE) (CC BY 4.0).

## Status

The official TAFFISH Hub is currently curated by the `taffish` GitHub
organization. It is not an open self-service publishing platform yet. Developers
who want to publish apps into the official Hub should contact the maintainer for
review or organization membership.
