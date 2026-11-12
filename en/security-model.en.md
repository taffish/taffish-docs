# TAFFISH Security Model

[English](security-model.en.md) | [中文](../zh/security-model.cn.md)

This document summarizes the current TAFFISH security model. It is a practical
map of how source code, release artifacts, installers, Hub index metadata,
container checks, local installation, and MCP tools fit together.

This is not a claim that TAFFISH solves all supply-chain security problems.
The model combines Hub smoke/trust metadata with conservative local
maintenance commands, remains layered and auditable, and is designed to
improve over time.

## Table Of Contents

- [Scope](#scope)
- [Source Code And Vulnerability Reports](#source-code-and-vulnerability-reports)
- [Release Payload Integrity](#release-payload-integrity)
- [Installer And Mirror Behavior](#installer-and-mirror-behavior)
- [Hub Index Trust Gate](#hub-index-trust-gate)
- [Local Install Verification](#local-install-verification)
- [Container Runtime Boundaries](#container-runtime-boundaries)
- [MCP And AI Safety Boundaries](#mcp-and-ai-safety-boundaries)
- [User Recommendations](#user-recommendations)
- [Current Limitations](#current-limitations)
- [Where To Read Next](#where-to-read-next)

## Scope

TAFFISH has several security surfaces:

- the local `taffish`, `taf`, and `taffish-mcp` commands;
- release binaries and installation scripts;
- TAFFISH Hub index data;
- taf-app source repositories;
- container images used by taf-apps;
- MCP tools exposed to AI clients.

The goal is to keep these surfaces explicit. Users should know what is checked,
what is only recorded as metadata, and what still depends on external systems
such as GitHub, Gitee, GHCR, Docker, Podman, Apptainer, Git, or the operating
system.

## Source Code And Vulnerability Reports

The local TAFFISH CLI/compiler implementation is open source in
[taffish/taffish](https://github.com/taffish/taffish) under Apache License 2.0.

Security reports should follow the source repository policy:

- [Security Policy](https://github.com/taffish/taffish/blob/main/SECURITY.md)

That policy covers private vulnerability reporting and the expected scope for
issues involving command generation, shell escaping, installer behavior,
source verification, package index metadata, MCP safety boundaries, and release
artifact integrity.

## Release Payload Integrity

### Per-program verification in 0.11.0

The 0.11.0 release payload includes:

```text
target/SHA256SUMS
target/SHA256SUMS.asc
target/TAFFISH-RELEASE-KEY.asc
```

The **0.11.0 installer** checks each selected binary against `SHA256SUMS`
before replacing commands when installing from official GitHub/Gitee raw
sources in the default `auto` mode. It also verifies the manifest signature
when the signature, release key, `gpg`, and `gpgv` are available. Older
installer scripts have their own verification policy.

<a id="platform-archives-in-the-0120-candidate"></a>

### Platform archives in 0.12.0

Starting with [0.12.0](https://github.com/taffish/taffish/releases/tag/v0.12.0),
official platform archives use two verification layers:

1. Before extraction, check the complete archive against the adjacent
   `ARCHIVE-SHA256SUMS`; signature verification uses `ARCHIVE-SHA256SUMS.asc`
   and `TAFFISH-RELEASE-KEY.asc` according to the selected verification mode.
2. After extraction and before installation, check the three programs against
   the package's `target/SHA256SUMS`, with its own signature and release key.

This applies to packages selected with `--local` or recognized by their
standard platform filename, including packages supplied through a mirror,
`--url`, or `--archive`. Both checksum manifests are required in `auto` mode.
Other custom bundles and explicit binary URLs do not automatically receive
this archive-level protection.

### Verification modes

| Mode | Required checks |
| --- | --- |
| `auto` (default) | Checksums for official raw installs and both layers of recognized platform packages. Signatures are also checked against the pinned release key when the signature, key, `gpg`, and `gpgv` are available. Other custom sources without a program manifest can produce a warning and an unverified installation. |
| `sha256` | Checksums are mandatory for every applicable layer; signatures are not required. |
| `gpg` | Checksums and valid signatures from the pinned TAFFISH release key are mandatory for every applicable layer. Missing verification material or tools causes failure. |
| `none` | Explicitly disables verification; it is not an equivalent fallback. |

`auto` therefore does not guarantee signature verification. For an authenticated
offline installation, obtain the installer through a trusted channel.
Place the archive, both archive verification files, and the release key beside
the saved installer, then use:

```sh
sh install-taffish.sh --local --verify gpg --user
```

Package names, manual verification, and custom-source options are documented in
the [0.12.0 installation guide](https://github.com/taffish/taffish/blob/v0.12.0/install/README.md).
The install receipt, `install-receipt.toml`, records the verification level
actually achieved. If that path already exists, it must be a regular file:
both first-time installs and reinstalls reject symlinks, including dangling ones.

Important boundaries:

- a payload hash or signature does not authenticate the installer fetched by
  `curl | sh`;
- checksums alone detect mismatches but do not authenticate a manifest obtained
  from the same untrusted source as the payload;
- the current release process is not a reproducible-build claim;
- the current release process is not a GitHub Actions provenance or artifact
  attestation claim.

For high-security installation, obtain the pinned installer and the release-key
fingerprint through a separately trusted channel. Inspect the installer, compare
the key fingerprint before trusting it, and verify the payload before installation.

## Installer And Mirror Behavior

The installer script is usually downloaded from `main`, but installed binaries
are selected from a fixed release tag through `--version`.

Example:

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --version X.Y.Z --user
```

Replace `X.Y.Z` with the release version you want to verify and install.

This pins the binary payload, not the script downloaded from `main`. To pin
both, use the reviewed script from a fixed tag and specify the payload version.

The Gitee installer and China mirror profile are intended to improve access
when GitHub raw URLs are slow or blocked. Mirror configuration can rewrite:

- the index URL used by `taf update`;
- app repository clone URLs used by `taf install`.

Mirror configuration does not automatically mirror container registries.
Container images, often hosted on GHCR, must still be reachable by the machine
that runs the app.

The 0.12.0 Gitee installer reassembles macOS archive parts before
checking the complete archive. The `.parts` manifest checks transport integrity;
it does not replace the signed archive manifest. Local installation with
`--local` skips the online index update but retains the selected payload
verification policy.

## Hub Index Trust Gate

TAFFISH Hub uses a static index generated by `taffish-index`. The index is the
machine-readable source consumed by:

```sh
taf update
taf search
taf info
taf install
```

For each app version, the index can record:

- app repository and release tag;
- `source.commit`;
- dependency metadata;
- platform constraints;
- container image reference;
- container digest and platform metadata;
- `[smoke]` metadata and smoke result;
- `trust.status`.

For new containerized versions, the intended index pipeline is:

1. Scan app repositories and release tags.
2. Read `taffish.toml`.
3. Validate required app metadata.
4. Inspect the container image digest and platforms.
5. Run the configured required and advisory smoke checks.
6. Accept new versions that pass the required gates. An advisory backend
   failure is reported but does not on its own prevent installation.

Acceptance is distinct from the default install recommendation: an accepted
prerelease can be installed explicitly but is not the default. Inspect required
and advisory failures in:

```text
index/reports/latest.json
index/reports/<timestamp>.json
```

Previously accepted versions are normally reused instead of rechecked on every
index build. This avoids making old, already accepted versions disappear because
of temporary network or registry failures. Manual force recheck can be used by
maintainers when index logic changes or a historical record needs repair.

## Local Install Verification

When the local index provides `source.commit`, `taf install` verifies that the
resolved app source is at the expected Git commit and that the resolved source
worktree is clean before building the installed command wrapper.

This is especially important with source rewrite rules:

```toml
[[source.rewrite]]
from = "https://github.com/taffish/"
to = "https://gitee.com/taffish-org/"
enabled = true
```

The clone URL may be rewritten to a mirror, but the checked source must still
resolve to the commit recorded in the index and remain unmodified. This makes
mirror-based installs more auditable without rewriting app source records in the
index.

`taf install --from <PROJECT-DIR>` is a local/private installation path. It can
preserve local smoke metadata, but it is not the same as being accepted by the
public Hub index trust gate.

TAFFISH `0.10.0` adds local package-maintenance commands that operate on local
install metadata and the current local index:

```sh
taf outdated
taf upgrade
taf prune
```

`taf upgrade` and `taf prune` are conservative by default and require explicit
confirmation such as `--yes` before they modify local app source or wrapper
files. Public-index upgrades do not silently replace local/private apps
installed with `taf install --from`.

With 0.11.0, `taf doctor index` checks the cached index, and `taf doctor hub`
checks local installation evidence without fetching, repairing or running apps.
A legacy install missing its build manifest is **unverified**, not necessarily
corrupt; a verified install proves local consistency, not an official origin.
See [Troubleshooting](troubleshooting.en.md) before attempting an explicit repair.

## Container Runtime Boundaries

TAFFISH can call Docker, Podman, or Apptainer, but it does not make arbitrary
containers safe by itself.

Important runtime facts:

- containerized apps execute code from the declared container image;
- TAFFISH container tags normally bind-mount the current working directory into
  the container;
- running from host system directories such as `/usr/bin`, `/usr/local/bin`,
  `/lib`, or `/opt` can hide important paths inside the image;
- container registry access is separate from Git/index mirror access.

For app authors, smoke checks should be:

- short;
- deterministic;
- safe to run in CI/index automation;
- independent of large user datasets;
- focused on proving that the expected executable exists and a minimal command
  can run.

For users, run TAFFISH apps from project, data, or scratch directories rather
than host system directories.

## MCP And AI Safety Boundaries

`taffish-mcp` is intentionally conservative.

It can expose read-only or preview-oriented capabilities such as:

- validating `.taf` source;
- compiling `.taf` source to shell code without executing it;
- summarizing `.taf` source;
- inspecting installed or indexed apps;
- inspecting the current project;
- exposing smoke/trust metadata to AI clients;
- planning local package maintenance operations such as outdated checks,
  install-all, upgrades, and pruning;
- providing resources and prompts.

It does not expose:

- `taf run`;
- `taf publish`;
- container image builds;
- smoke execution;
- container startup;
- image pulls.

This boundary matters because AI clients can inspect and reason about TAFFISH
projects without automatically running workflows, pulling images, or publishing
changes.

## User Recommendations

For regular users:

- install from the official `taffish/taffish` repository;
- pin `--version` when reproducibility matters;
- run `taf update` before installing apps;
- prefer apps that appear in the official index over unreviewed local copies;
- inspect `taf info <app>` when digest, platform, smoke, or source metadata
  matters;
- run containerized apps from safe work directories.

For high-security environments:

- verify release checksums and GPG signatures manually;
- mirror both the index and app repositories internally;
- keep source rewrite rules explicit and auditable;
- use `source.commit` verification from the index;
- decide separately how container registries and images are mirrored or
  allow-listed;
- review index build reports, especially failed smoke or digest checks.

For app authors:

- use immutable `version-release` tags;
- avoid rewriting published release tags;
- provide real `[smoke]` metadata for containerized apps;
- keep Dockerfiles and app metadata in the same source repository;
- keep release notes clear through `release.md`.

## Current Limitations

The current model intentionally leaves some future work open:

- checksum-only verification is not signature verification; check the receipt
  rather than assuming that `auto` used GPG;
- official release binaries are manually built, not produced by a fully
  automated provenance pipeline;
- reproducible builds are not claimed yet;
- GitHub Actions artifact attestation is not claimed yet;
- container image trust still depends on the registry and image publication
  process;
- Gitee or internal mirrors improve access but do not remove the need for
  checksum, commit, and digest verification;
- the official Hub is curated, not an open self-service package registry.

These limitations are part of the published model. They should be reduced over
time without overstating what the current release already guarantees.

## Where To Read Next

- [What Is TAFFISH](taffish.en.md)
- [What Is TAFFISH Hub](taffish-hub.en.md)
- [TAFFISH Index JSON Specification](index-json-spec.en.md)
- [`taffish.toml` Specification](taffish-toml-spec.en.md)
- [TAFFISH MCP Guide](taffish-mcp.en.md)
- [Troubleshooting](troubleshooting.en.md)
- [taffish/taffish Security Policy](https://github.com/taffish/taffish/blob/main/SECURITY.md)
- [taffish/taffish Build From Source](https://github.com/taffish/taffish/blob/main/docs/dev/en/build-from-source.md)
