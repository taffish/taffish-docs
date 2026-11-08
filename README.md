# TAFFISH Documentation

[English](README.md) | [中文](README.cn.md)

TAFFISH packages bioinformatics tools and lightweight workflows as commands you can install by version. Use `taf` to find and install an app from TAFFISH Hub. Then run its command, such as `taf-augustus`, in your terminal or an existing script. When a script needs a specific version, use the versioned command name.

**[Install TAFFISH and run your first app](en/quick-start.en.md)** · [Browse apps in TAFFISH Hub](https://taffish.github.io)

<a id="table-of-contents"></a>

## Recommended Reading Paths

| I want to… | Start here | Next, if needed |
| --- | --- | --- |
| Use an existing app | [Quick Start](en/quick-start.en.md) | [Troubleshooting](en/troubleshooting.en.md) if a command fails |
| Set up a shared server or HPC system | [System Administration](en/system-administration.en.md) | [Security Model](en/security-model.en.md) for trust decisions |
| Learn the `.taf` language | [TAF Script Tutorial](en/taf-script-tutorial.en.md) | [What Is TAFFISH?](en/taffish.en.md) for syntax and CLI details |
| Write a tool app | [App Developer Guide](en/app-developer-guide.en.md) | [`taffish.toml` Specification](en/taffish-toml-spec.en.md) for manifest fields |
| Package a tool that runs in a container | [App Developer Guide](en/app-developer-guide.en.md) | [Containerized App Best Practices](en/container-apps.en.md) for images, mounts, and smoke tests |
| Prepare an app for the official Hub | [Official taf-app Curation Guide](en/taf-app-curation-guide.en.md) | [Augustus](https://github.com/taffish/augustus) as a worked example |
| Combine tools into a Flow | [Flow and Dependencies](en/flow-dependencies.en.md) | [Flow Developer Guide](en/taf-flow-developer-guide.en.md), then the [pre-publication checklist](en/taf-flow-curation-checklist.en.md) |
| Find a Flow for an analysis | [TAFFISH Flow Portal](https://taffish.github.io/flows/) | [RNA-seq example and reports](#flow-portal-and-complete-example) |
| Understand how an app gets into the index | [What Is TAFFISH Hub?](en/taffish-hub.en.md) | [Index JSON Specification](en/index-json-spec.en.md) for version records and test results |
| Check what TAFFISH verifies | [Security Model](en/security-model.en.md) | [Index JSON Specification](en/index-json-spec.en.md) for recorded checks |
| Connect an AI client | [MCP Client Setup](en/mcp-clients.en.md) | [MCP Guide](en/taffish-mcp.en.md) for the tools available to the client |
| Build or modify TAFFISH itself | [Build From Source](https://github.com/taffish/taffish/blob/main/docs/dev/en/build-from-source.md) | [Source development and contributing](#source-development) |

Run `taf --version` to check your installed version. For changes in 0.11.0, see the [release notes](https://github.com/taffish/taffish/releases/tag/v0.11.0).

## Flow Portal and Complete Example

The [Flow Portal](https://taffish.github.io/flows/) lists official Flow families. For a worked example, follow the [RNA-seq Flow Family](https://taffish.github.io/rnaseq-flows/) from reference preparation to the final report. It shows how versioned shell commands fit together; you can also call those commands from an existing workflow engine. Open the [yeast report](https://taffish.github.io/rnaseq-flows/examples/yeast-standard-report/04_reports/rnaseq_report.html) alongside the [interpretation guide](https://taffish.github.io/rnaseq-flows/examples/yeast-standard-report/04_reports/report_interpretation.html) to read the results.

<a id="how-the-documents-overlap"></a>

## Document Roles

Use a tutorial to try an example, a guide to carry out a task, and a specification to look up an exact field.

### Core Concepts

- [What Is TAFFISH?](en/taffish.en.md) explains how a `.taf` script becomes a shell command, how it handles arguments and containers, and where [MCP fits](en/taffish.en.md#mcp--ai-integration).
- [What Is TAFFISH Hub?](en/taffish-hub.en.md) follows an app from its GitHub release through index checks to the version a user can install.

### User Guides

- [Quick Start](en/quick-start.en.md): install and run Augustus. Return here to install a private or local app with `taf install --from`, check for updates with `taf outdated`, or manage installed versions with `taf upgrade` and `taf prune`.
- [System Administration](en/system-administration.en.md): configure shared installs, permissions, shell completion, Vim files, and container policy.
- [MCP Client Setup](en/mcp-clients.en.md): connect a client such as Codex or Claude Code to `taffish-mcp`. The [MCP Guide](en/taffish-mcp.en.md) describes its tools, inspection and compile previews, and what the server will not execute.
- [Troubleshooting](en/troubleshooting.en.md): diagnose failures in installation, mirror access, container startup, or the installed app command.

### Developer Guides

- [TAF Script Tutorial](en/taf-script-tutorial.en.md): write a `.taf` script and pass arguments to it. The [App Developer Guide](en/app-developer-guide.en.md) takes an app from `taf new` through testing, building, preparing `release.md`, and publishing.
- [Containerized App Best Practices](en/container-apps.en.md): build the image, declare smoke tests, and check behavior across Docker, Podman, and Apptainer. Use it to distinguish app requirements (`$@[target: args]`) from local backend settings (`TAFFISH_CONTAINER_BACKEND` and options such as `TAFFISH_PODMAN_RUN_ARGS`).
- [Official taf-app Curation Guide](en/taf-app-curation-guide.en.md): review a Hub app's help, version, smoke tests, and release before submitting it for inclusion.
- [Flow and Dependencies](en/flow-dependencies.en.md): declare exact dependency versions and call them with `[[taf: ...]]` and `@:` parameter blocks. Use the [Flow Developer Guide](en/taf-flow-developer-guide.en.md) to design inputs, outputs, and provenance; use the [Flow Curation Checklist](en/taf-flow-curation-checklist.en.md) before release.

### Specifications

- [`taffish.toml` Specification](en/taffish-toml-spec.en.md): look up fields an app author writes, including `[smoke]`, `[meta]`, and `[upstream]`.
- [Index JSON Specification](en/index-json-spec.en.md): look up what the Hub records for a version, including source identity, container digests, smoke results, and metadata overrides.

### Security Model

The [Security Model](en/security-model.en.md) explains what the installer verifies, how the Hub and `taf install` check releases and `source.commit`, and where trust in mirrors, containers, and MCP clients begins and ends.

## Source Development

The compiler, `taf`, and `taffish-mcp` live in [taffish/taffish](https://github.com/taffish/taffish). Use [Build From Source](https://github.com/taffish/taffish/blob/main/docs/dev/en/build-from-source.md) for an SBCL or LispWorks build, the [source-tree developer docs](https://github.com/taffish/taffish/tree/main/docs/dev/en) for architecture and release engineering, and [Contributing](https://github.com/taffish/taffish/blob/main/CONTRIBUTING.md) for tests and patches. To report a vulnerability privately, follow the [Security Policy](https://github.com/taffish/taffish/blob/main/SECURITY.md).

<a id="status"></a>

## Online Hub

Browse apps at [taffish.github.io](https://taffish.github.io). The website reads the official index; `taf update` refreshes the local copy used by `taf`. The `taffish` GitHub organization reviews apps for inclusion. To propose an app or request organization membership, contact the maintainers; publishing your own repository does not automatically list it in the official Hub.

<a id="repository-layout"></a>

## Documentation Policy

English guides are in `en/` (`*.en.md`); Chinese guides are in `zh/` (`*.cn.md`). These guides describe TAFFISH across releases. For exact behavior in a particular version, consult that release's notes and source code.

## License

The documentation is licensed under [CC BY 4.0](LICENSE).
