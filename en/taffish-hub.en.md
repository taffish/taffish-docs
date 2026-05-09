# What Is TAFFISH Hub

TAFFISH Hub is the app index and distribution layer of the TAFFISH ecosystem. Its
goal is to let users obtain reproducible, versioned, source-traceable
bioinformatics tools and flows through:

```sh
taf update
taf install <app>
```

At the current stage, TAFFISH Hub mainly relies on GitHub automation and does
not require an independent backend server. It has three core parts:

- `taffish-index`: a static index repository that generates machine-readable index JSON.
- `taffish.github.io`: a static website that displays apps from the index.
- `.github`: the organization profile repository that controls the `github.com/taffish` overview page.

You can browse the current TAFFISH Hub at [taffish.github.io](https://taffish.github.io).
The website reads static JSON from `taffish-index`, so it displays apps currently
discovered by the official index.

Each real app repository can also have its own GitHub Actions workflow, for
example to build GHCR images from a Dockerfile. That responsibility belongs to
the app repository itself and is not centrally handled by `taffish-hub`.

## Table Of Contents

- [Why Hub Exists](#why-hub-exists)
- [Access And Publishing Policy](#access-and-publishing-policy)
- [Current Repository Layout](#current-repository-layout)
- [Data Flow](#data-flow)
- [`taffish-index`](#taffish-index)
- [App Discovery Rules](#app-discovery-rules)
- [Index Data Model](#index-data-model)
- [Dependencies](#dependencies)
- [Platform Constraints](#platform-constraints)
- [Upstream Source Metadata](#upstream-source-metadata)
- [`taffish.github.io`](#taffishgithubio)
- [`.github`](#github)
- [App Image Builds](#app-image-builds)
- [How Users Are Served](#how-users-are-served)
- [How Developers Publish Apps](#how-developers-publish-apps)
- [Mirror And Internal Source Support](#mirror-and-internal-source-support)
- [Current Boundaries](#current-boundaries)

## Why Hub Exists

TAFFISH apps are distributed across many GitHub repositories. Users should not
need to manually know every repository, tag, Docker image, or command name.

Hub provides a central index that organizes distributed information into:

- which apps are available
- which versions each app has
- which version is latest
- whether an app is a tool or a flow
- install command
- app GitHub repository
- app release tag and commit
- whether a container image exists
- whether dependencies exist
- platform constraints
- original upstream software source

Users only need to update the local index, then install by name.

## Access And Publishing Policy

The TAFFISH Hub website and static index are public for users. Any user can open
[taffish.github.io](https://taffish.github.io), and any user can download the
public index through `taf update`.

Official TAFFISH Hub app publishing and maintenance are not open self-service at
the moment. Currently, only members of the `taffish` GitHub organization can
create, publish, and maintain official app repositories. The official maintainer
set is currently small. Developers who want their apps included in the official
TAFFISH Hub should contact the maintainer to request organization membership or
maintainer-assisted review and publication.

This design keeps early-ecosystem app metadata, version tags, Docker images,
licenses, upstream sources, and dependency relationships consistent. If the
maintainer group grows later, a more formal review, ownership, and contribution
process can be added.

## Current Repository Layout

In the local `taffish-hub` workspace, repositories that will be published to
GitHub are staged under `repos/`:

```text
taffish-hub/
  README.md
  docs/
    README.md
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

This means `taffish-hub` is a local factory, while `repos/<name>` is the root of
a real repository that will be or has already been published to GitHub.

Current main repositories:

```text
repos/taffish-index       -> github.com/taffish/taffish-index
repos/taffish.github.io   -> github.com/taffish/taffish.github.io
repos/.github             -> github.com/taffish/.github
```

## Data Flow

From app developer to user installation, the approximate flow is:

```text
developer creates a TAFFISH app
        |
        v
app repository publishes release tag: v<version>-r<release>
        |
        v
taffish-index GitHub Actions scans the taffish organization
        |
        v
generate index/index.json
        |
        +----> taffish.github.io reads and displays it
        |
        +----> user downloads it locally with taf update
                         |
                         v
                  taf install resolves and installs the app
```

There is no central database and no long-running backend service in this path.
GitHub repositories are the data source, GitHub Actions periodically generates
the static index, and GitHub Pages displays the web interface.

## `taffish-index`

`taffish-index` is a static index repository. Its core files are:

```text
index/index.json
index/packages/<package>.json
index/commands/<command>.json
```

`index/index.json` is the complete index. `packages/` and `commands/` are split
helper indexes.

The index builder is written in Common Lisp. Its entry point is:

```sh
sbcl --script scripts/build-index.lisp -- --org "taffish" --output index
```

For local testing, you can disable GitHub scanning and scan a local project:

```sh
sbcl --script scripts/build-index.lisp -- --no-org --local-repo ../../../taffish/test/my-test-tool --output index
```

### Automation

The GitHub Actions workflow in `taffish-index` is located at:

```text
.github/workflows/build-index.yml
```

Triggers:

- Manual dispatch: `workflow_dispatch`
- Scheduled run: once per day

Current schedule:

```yaml
schedule:
  - cron: "17 1 * * *"
```

This is UTC time. The workflow:

1. checks out the `taffish-index` repository
2. installs SBCL
3. runs `scripts/build-index.lisp`
4. scans repositories in the `taffish` organization
5. generates `index/`
6. commits and pushes automatically if the index changed

By default, the workflow uses `GITHUB_TOKEN` for public repositories. If private
repositories need to be scanned, configure:

```text
TAFFISH_BOT_TOKEN
```

## App Discovery Rules

A GitHub repository is considered a TAFFISH app when it satisfies:

- root-level `taffish.toml` exists
- required `taffish.toml` sections and fields are valid
- `[package].main` points to an existing `.taf` file
- `docs/help.md` exists
- `[repository].url` points to the scanned GitHub repository
- release tags use `v<version>-r<release>`

Examples:

```text
v0.1.0-r1
v0.1.0-r2
v1.0.0-r1
```

The index builder prefers release tags. The default branch can be indexed as a
development snapshot, but this requires enabling `include_default_branch` during
manual use.

## Index Data Model

The top-level structure of the full index is approximately:

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

Each package contains:

```json
{
  "name": "my-tool",
  "latest": "0.1.0-r1",
  "repository_url": "https://github.com/taffish/my-tool",
  "command": {
    "name": "taf-my-tool"
  },
  "versions": {}
}
```

Each version record contains:

```json
{
  "name": "my-tool",
  "kind": "tool",
  "version": "0.1.0",
  "release": 1,
  "version_id": "0.1.0-r1",
  "tag": "v0.1.0-r1",
  "license": "Apache-2.0",
  "repository_url": "https://github.com/taffish/my-tool",
  "repository_slug": "taffish/my-tool",
  "command": {
    "name": "taf-my-tool"
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
    "image": "ghcr.io/taffish/my-tool:0.1.0-r1",
    "dockerfile": "docker/Dockerfile",
    "image_tag": "0.1.0-r1",
    "image_tag_matches_version": true
  },
  "source": {
    "repository": "taffish/my-tool",
    "ref": "v0.1.0-r1",
    "commit": "...",
    "html_url": "https://github.com/taffish/my-tool/tree/v0.1.0-r1"
  }
}
```

If the app provides `[upstream]`, the version record may also include:

```json
{
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
}
```

If no valid upstream information is provided, the `upstream` field is omitted
instead of being written as `null` or `none`.

## Dependencies

Flow apps can declare dependencies in `taffish.toml`:

```toml
[dependencies]
taf-dep-tool = "0.1.0-r1"
taf-x = ["0.1.0-r1", "0.1.0-r2"]
```

The index exports them as:

```json
{
  "dependencies": {
    "taf-dep-tool": "0.1.0-r1",
    "taf-x": ["0.1.0-r1", "0.1.0-r2"]
  }
}
```

This does not mean "alternative versions". It means the flow may need multiple
different versions. `taf install` installs dependencies according to the index.

## Platform Constraints

Apps can declare platform constraints:

```toml
[platform]
os = "linux,darwin"
arch = "amd64,arm64"
container = "required"
min_cpus = 2
min_memory_mb = 4096
```

This information enters the index and can be used by the installer, web UI, and
users when deciding whether an app fits an environment.

Current `container` values:

```text
optional
required
forbidden
```

## Upstream Source Metadata

`[upstream]` describes the original software source wrapped by an app, such as
BLAST, CD-HIT, BWA, and other tools.

Example:

```toml
[upstream]
name = "BLAST+"
type = "official"
homepage = "https://blast.ncbi.nlm.nih.gov/"
version = "2.16.0"
```

Or:

```toml
[upstream]
name = "CD-HIT"
type = "github"
repository = "weizhongli/cdhit"
version = "4.8.1"
```

Supported fields:

```text
name
type
homepage
repository
release_url
docker_image
version
license
citation
doi
pmid
```

This information matters because:

- users can see which original tool the taf app wraps
- maintainers can check new upstream versions
- bioinformatics tools can preserve citation, license, and paper information

## `taffish.github.io`

`taffish.github.io` is a static website repository. It reads the index from:

```text
https://raw.githubusercontent.com/taffish/taffish-index/main/index/index.json
```

It provides:

- English and Chinese interface
- app search
- tool / flow filters
- dependency filters
- container image filters
- name / latest-version sorting
- package detail view
- version list
- dependency list
- platform constraints
- upstream source display
- install command copy
- dependency-aware install chain
- index sync failure notice

It does not need a backend server. GitHub Pages directly hosts static files:

```text
index.html
styles.css
app.js
```

## `.github`

`.github` is the GitHub organization profile repository. GitHub displays:

```text
profile/README.md
```

at:

```text
https://github.com/taffish
```

It is a good place for organization entry points, Hub links, index repository
links, and current project status.

## App Image Builds

Image building is not the responsibility of `taffish-index` or
`taffish.github.io`.

Each app with a Dockerfile should have its own GitHub Actions workflow, for
example:

```text
app-repo/
  Dockerfile or docker/Dockerfile
  .github/workflows/build-image.yml
```

`taf new --tool --docker` can generate a project skeleton with a Dockerfile and
GitHub Actions workflow.

Reasons for this design:

- each app has a different build environment
- each app Dockerfile should be versioned with the app source
- each app release tag should correspond to its image tag
- Hub discovers and indexes apps; it does not build images for them

## How Users Are Served

The core user path:

```sh
taf update
taf search blast
taf install blast
taf list
taf which taf-blast-v2.16.0-r1
```

`taf update` downloads the static index into the local cache:

```text
index/current.json
index/snapshots/
```

`taf install`:

1. reads local `index/current.json`
2. resolves the target by package name, command name, or exact versioned command
3. finds the corresponding version record
4. installs dependencies
5. clones the corresponding source tag
6. builds the versioned command wrapper
7. writes local app data and command paths

Install commands can be:

```sh
taf install my-tool
taf install my-tool 0.1.0-r1
taf install taf-my-tool
taf install taf-my-tool v0.1.0-r1
taf install taf-my-tool-v0.1.0-r1
```

## How Developers Publish Apps

Typical path:

```sh
taf new my-tool --tool --docker
cd my-tool
taf check
taf run -- --help
taf build --all
taf publish --dry-run
taf publish --yes --build
```

After publication, the app repository should have:

```text
taffish.toml
src/main.taf
docs/help.md
release tag: v<version>-r<release>
```

If it is containerized, it should also have:

```text
docker/Dockerfile
GHCR image: ghcr.io/taffish/<app>:<version>-r<release>
```

When the next `taffish-index` automation run completes, the app is written into
the index. Users can then run `taf update` and install it.

## Mirror And Internal Source Support

TAFFISH Hub remains a GitHub-first static index, but TAFFISH `0.2.0` adds
runtime mirror configuration on the local `taf` side. This means users do not
have to change the official index schema to use a mirror.

The default GitHub path is:

```text
taf update -> official index URL
taf install -> canonical app repository URLs from the index
```

For users in China or in an internal network, `config.toml` can override those
two access paths:

```toml
schema_version = "taffish.config/v1"
profile = "china"
language = "en"

[index]
url = "https://gitee.com/taffish-org/taffish-index/raw/main/index/index.json"

[[source.rewrite]]
from = "https://github.com/taffish/"
to = "https://gitee.com/taffish-org/"
enabled = true
```

`[index].url` changes where `taf update` reads the static index from.
`[[source.rewrite]]` keeps canonical GitHub URLs in the official index while
allowing `taf install` to clone from a mirror or an internal Git service.

This has two practical consequences:

- The official Hub can keep GitHub as the canonical publishing source.
- Mirror operators can sync the index and app repositories without changing app
  metadata, as long as the mirror preserves compatible repositories, tags, and
  the same TAFFISH index schema.

Container images are separate. If an app version points to GHCR or another OCI
registry, the machine running the app still needs access to that image location,
or the app metadata needs to point to an image source that the user can access.

## Current Boundaries

Current TAFFISH Hub boundaries:

- It is a static index and static website, not a traditional database backend.
- It depends on GitHub repositories, release tags, GitHub Actions, and GitHub Pages.
- It does not store long-term user-local state; user state is managed by local `taf`.
- It does not build app images; image builds are managed by app repositories.
- It supports mirror-friendly index and repository access through local `taf`
  runtime config, but it does not automatically mirror container registries.

If an independent server is needed later, the current static index model can be
migrated to a database service. At the current stage, GitHub static deployment is
enough to support TAFFISH app discovery, display, and installation.
