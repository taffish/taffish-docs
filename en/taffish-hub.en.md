# What Is TAFFISH Hub

TAFFISH Hub is the official app registry and index layer for TAFFISH. It is not a traditional database of biological datasets. It is closer to a package registry: its records describe tools and workflows that can be installed by `taf`.

TAFFISH Hub currently relies on GitHub infrastructure:

- App repositories live under the `taffish` GitHub organization.
- Each app repository contains a valid `taffish.toml`.
- Container images are built by each app repository's GitHub Actions workflow.
- The `taffish-index` repository scans app repositories and generates static JSON index files.
- The `taffish.github.io` site provides a web interface for browsing the Hub.
- Local `taf` commands consume the JSON index for search and installation.

## Table Of Contents

- [Publishing Policy](#publishing-policy)
- [What Hub Stores](#what-hub-stores)
- [Repository Model](#repository-model)
- [Index Repository](#index-repository)
- [GitHub Actions Automation](#github-actions-automation)
- [Index Files](#index-files)
- [How `taf` Uses The Index](#how-taf-uses-the-index)
- [Dependencies](#dependencies)
- [Container Images](#container-images)
- [Upstream Metadata](#upstream-metadata)
- [Web Hub](#web-hub)
- [GitHub-Based Operation](#github-based-operation)
- [Future Server Option](#future-server-option)
- [Maintenance Workflow](#maintenance-workflow)

## Publishing Policy

The official TAFFISH Hub is not currently an open self-service publishing platform.

Only members of the `taffish` GitHub organization can create and maintain official Hub app repositories. At present, the organization is maintained by a single maintainer. Developers who want an app included in the official Hub should contact the maintainer to request organization membership or maintainer-assisted review and publication.

This policy keeps the official registry small, curated, and reproducible while TAFFISH is still young.

## What Hub Stores

For each app, Hub stores metadata such as:

- Package name.
- App kind: `tool` or `flow`.
- Version and release.
- GitHub repository URL.
- Release tag.
- Command name and versioned command name.
- Runtime flags.
- Container image metadata, if present.
- Flow dependencies, if present.
- Platform constraints, if declared.
- Upstream software metadata, if provided.

Hub does not store the actual application source as a separate backend database. The source lives in app repositories and release tags. The index records where the source is and how `taf` should install it.

## Repository Model

The current project layout separates local Hub work from published repositories.

Example local layout:

```text
taffish-hub/
  README.md
  docs/
    zh/
      taffish.cn.md
      taffish-hub.cn.md
    en/
      taffish.en.md
      taffish-hub.en.md
  repos/
    taffish-index/
    taffish.github.io/
    .github/
```

`repos/` contains repositories that are prepared locally and then published to GitHub. Each app repository can also be staged here before publication.

In the future, local maintenance data can be separated from publishable app repositories. For example:

```text
local-store/
  app-name/
    upstream.toml
    versions/
      1.0.0/
        r1/
          taf-app-project/

repos/
  app-name/
```

This keeps long-term local maintenance history separate from the clean repository that is published to GitHub.

## Index Repository

`taffish-index` is a special repository. It is not an app. It is the automation and output repository for the Hub index.

Responsibilities:

- Scan repositories under the `taffish` organization.
- Detect app repositories by looking for a valid root-level `taffish.toml`.
- Read release tags.
- Read app metadata.
- Build JSON index files.
- Commit generated index files back into the index repository.

The index repository lets the Hub work without a separate backend server.

## GitHub Actions Automation

The index repository has a GitHub Actions workflow that runs on:

- A scheduled interval.
- Manual dispatch through `workflow_dispatch`.

The workflow:

1. Checks out the index repository.
2. Installs SBCL.
3. Runs the Common Lisp index builder.
4. Writes generated JSON into `index/`.
5. Commits changes if the generated index changed.

The current schedule can be daily instead of hourly to reduce GitHub Actions usage. Manual dispatch remains useful when a maintainer publishes a new app and wants the index updated immediately.

## Index Files

The main index file uses schema:

```json
"schema": "taffish.index/v1"
```

It contains:

- `generated_at`
- `source`
- `counts`
- `packages`
- `command_index`
- `warnings`

See [TAFFISH Index JSON Specification](index-json-spec.en.md) for the detailed schema.

## How `taf` Uses The Index

`taf update` downloads the latest index and stores it locally.

`taf search` uses the local index to find packages.

`taf info` displays package, version, dependency, container, and upstream metadata.

`taf install` resolves the selected package and version, installs dependencies first, then clones or copies the indexed source ref and builds the command wrapper locally.

`taf uninstall` removes installed command wrappers and related local metadata.

`taf list` shows installed apps.

This means `taf install` does not need to scan GitHub at install time. It only needs the index record for the selected package.

## Dependencies

Flow dependencies are stored in both `taffish.toml` and the index.

`taffish.toml` example:

```toml
[dependencies]
taf-fastqc = "0.12.1-r1"
taf-align = ["2.0.0-r1", "2.1.0-r1"]
```

Index example:

```json
"dependencies": {
  "taf-fastqc": ["0.12.1-r1"],
  "taf-align": ["2.0.0-r1", "2.1.0-r1"]
}
```

Arrays mean all listed versions are required. They are not alternatives.

`taf install` should install every listed dependency before installing the flow itself.

## Container Images

Hub does not build container images.

Each app repository that uses a container should own:

```text
docker/Dockerfile
.github/workflows/build-image.yml
```

`taf new --tool --docker` can create a project skeleton with Dockerfile and GitHub Actions image workflow.

The image build workflow reads `taffish.toml`, builds the image, and pushes to GHCR.

Important requirements:

- The image tag should match `<version>-r<release>`.
- The GHCR package should be public.
- The image in `src/main.taf` should match `[container].image`.
- If local testing uses Docker or Podman, build and run should use the same backend.

## Upstream Metadata

Tool apps should describe their original upstream software when possible:

```toml
[upstream]
name = "BLAST"
version = "2.16.0"
url = "https://blast.ncbi.nlm.nih.gov/"
repository = "https://github.com/ncbi/blast_plus_docs"
license = "Public Domain"
description = "NCBI BLAST+ sequence alignment tools."
```

Hub displays this metadata so users can distinguish:

- The TAFFISH app repository.
- The original upstream software.
- The container image.
- The command installed by `taf`.

If upstream metadata is not provided, Hub should omit it rather than display "none".

## Web Hub

The web Hub lives at:

[https://taffish.github.io](https://taffish.github.io)

It provides:

- Package browsing.
- Tool / flow filtering.
- Sorting.
- Search.
- Version details.
- Install commands.
- Dependency display.
- Container and upstream metadata.

The website can be served by GitHub Pages because the index is static JSON. No custom server is required for the current design.

## GitHub-Based Operation

The current Hub design is intentionally GitHub-based:

- GitHub repositories store source.
- GitHub tags identify released versions.
- GitHub Actions builds images and index files.
- GitHub Pages hosts the web registry.
- GHCR hosts container images.

Advantages:

- No custom backend to operate.
- Public infrastructure is easy to inspect.
- Releases and tags are auditable.
- Static index files are easy for `taf` to consume.

Tradeoffs:

- GitHub connectivity can be unstable for some users.
- GitHub Actions is not unlimited.
- GHCR packages require visibility checks.
- Large scale may eventually need more dedicated infrastructure.

## Future Server Option

Later, TAFFISH Hub could move from a GitHub-only system to a dedicated server.

A future server could provide:

- Faster index queries.
- Mirror support.
- Region-specific download endpoints.
- Maintainer dashboards.
- Automated upstream update detection.
- Richer package analytics.

This is not required for the current phase. The current GitHub-based system is enough to support a curated official registry.

## Maintenance Workflow

Current practical workflow:

1. Develop or update an app project locally.
2. Run `taf check`.
3. For containerized apps, build and test the image locally.
4. Run `taf build`.
5. Run `taf publish --dry-run`.
6. Publish to the `taffish` organization.
7. Confirm GitHub Actions image build.
8. Confirm GHCR package visibility.
9. Trigger or wait for `taffish-index` update.
10. Check the app on [taffish.github.io](https://taffish.github.io).
11. Test with `taf update`, `taf search`, `taf info`, and `taf install`.

For upstream monitoring, a future local semi-automated maintenance system can keep a separate local store of app versions and upstream metadata. It can detect new upstream versions and write update reports for manual review before publication.

