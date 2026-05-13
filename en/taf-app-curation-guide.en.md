# Official taf-app Curation Guide

This guide is for official TAFFISH Hub maintainers who turn a `taf new` project
into a polished app that can be published, indexed, and used as a template for
future apps.

This is not the full `taffish.toml` specification, and it is not a general
Dockerfile tutorial. It is a curation guide: when you receive an app directory,
what should you inspect, what should you edit, what should you leave alone, and
how do you decide whether the app is good enough?

`augustus` is the first formal app in the official Hub and should be treated as
the current baseline template.

This guide focuses on maintainer judgment and checklists. Exact field semantics
still live in the [`taffish.toml` Specification](taffish-toml-spec.en.md), and
container implementation details still live in
[Containerized App Best Practices](container-apps.en.md).

## Table Of Contents

- [Scope](#scope)
- [Recommended Workflow](#recommended-workflow)
- [Treat `taf new` As The Boundary](#treat-taf-new-as-the-boundary)
- [`taffish.toml`](#taffishtoml)
- [`src/main.taf`](#srcmaintaf)
- [`docker/Dockerfile`](#dockerdockerfile)
- [`docs/help.md`](#docshelpmd)
- [`README.md`](#readmemd)
- [`release.md`](#releasemd)
- [Smoke Design](#smoke-design)
- [Version And Release Policy](#version-and-release-policy)
- [Pre-Publish Checks](#pre-publish-checks)
- [Post-Publish Checks](#post-publish-checks)
- [Things Not To Do](#things-not-to-do)
- [Augustus Template Notes](#augustus-template-notes)

## Scope

An official taf-app should:

- Put an upstream tool into a portable, indexable, verifiable TAFFISH package.
- Let users run the upstream tool through a stable `taf-xxx` command.
- Let the Hub index record source commit, container digest, platforms, and
  smoke/trust status.
- Let maintainers and AI clients understand the app's purpose, entry point,
  container, and checks.

An official taf-app should not redesign the upstream tool's CLI. Usually the
best wrapper is thin: provide a consistent entry point, container runtime,
help text, version metadata, and smoke gate, while leaving domain behavior to
the upstream tool.

## Recommended Workflow

Start from `taf new`:

```sh
taf new my-tool --tool --docker \
  --version 1.0.0 \
  --release 1 \
  --repo https://github.com/taffish/my-tool
```

Then polish in this order:

1. Read upstream documentation and identify the main command, helper commands,
   version policy, and recommended installation path.
2. Design the Dockerfile so the image contains commands needed by real
   workflows, not only the primary executable.
3. Design `src/main.taf`, preferably as a thin wrapper.
4. Polish `taffish.toml`, especially image, upstream, and smoke metadata.
5. Write `docs/help.md` so `taf-xxx --help` lets users start immediately.
6. Write `README.md` for installation, usage, container contents, and maintainer
   checks.
7. Write `release.md` so `taf publish --release` produces accurate release
   notes.
8. Run local checks, container smoke checks, and publish dry-run.
9. After publishing, wait for image build and inspect index reports.

## Treat `taf new` As The Boundary

The `taf new` output is not just a draft. It is the structural contract expected
by current tools.

Files maintainers usually edit:

```text
taffish.toml
src/main.taf
docker/Dockerfile
README.md
docs/help.md
release.md
```

Files maintainers usually leave alone:

```text
.github/workflows/build-image.yml
.gitignore
target/
LICENSE
```

Exceptions:

- If the `taf new` template changes, sync `.github/workflows/build-image.yml`
  to the new template.
- If the license actually changes, update both `LICENSE` and `taffish.toml`.
- If `target/` contains build artifacts, do not edit it by hand. Re-run
  `taf build` or `taf publish`.

## `taffish.toml`

`taffish.toml` is the core app metadata. Be conservative: use only fields that
`taf`, Hub, and the index builder explicitly understand.

Recommended shape:

```toml
[package]
name = "my-tool"
kind = "tool"
version = "1.0.0"
release = 1
license = "Apache-2.0"
main = "src/main.taf"

[repository]
url = "https://github.com/taffish/my-tool"

[command]
name = "taf-my-tool"

[runtime]
pipe = true
command_mode = true

[container]
image = "ghcr.io/taffish/my-tool:1.0.0-r1"
dockerfile = "docker/Dockerfile"
build_platforms = "linux/amd64,linux/arm64"

[upstream]
type = "github"
repo = "owner/project"
watch = "tags"
strip_prefix = "v"

[smoke]
backend = "docker"
timeout = 60
exist = ["my-tool"]
test = ["my-tool --help"]
```

Notes:

- `version` is the upstream tool version.
- `release` is the TAFFISH packaging release.
- `[container].image` should match `<version>-r<release>`.
- `[repository].url` is the TAFFISH app repository, not the upstream repository.
- `[upstream]` describes the wrapped software, not the TAFFISH app itself.
- `command_mode = true` matters for tool apps because users can run helper
  commands inside the same container.
- Do not add fields that current tools do not understand.

## `src/main.taf`

Most containerized tool apps should use a thin wrapper:

```taf
<taf-app:container:ghcr.io/taffish/my-tool:1.0.0-r1>
my-tool ::*ARGV*::
```

This gives users a default entry:

```sh
taf-my-tool -- --help
taf-my-tool -- --input data.txt
```

It also gives them explicit command mode:

```sh
taf-my-tool my-tool --help
taf-my-tool helper-command --help
```

Notes:

- Prefer `<taf-app:container:...>` so the runtime can select Docker, Podman, or
  Apptainer.
- Use `<taf-app:docker:...>` or `<taf-app:podman:...>` only when a backend must
  be fixed.
- Do not copy large amounts of upstream logic into `.taf` unless you are truly
  unifying parameters or composing steps.
- If the default action would be expensive, use `docs/help.md` to guide users
  toward `-- --help`.

## `docker/Dockerfile`

The Dockerfile should make the upstream tool environment self-contained. It
should not merely do enough to pass smoke checks.

Recommendations:

- Use explicit base image tags.
- Prefer upstream official releases, official binaries, or distribution
  packages.
- Include helper commands used by real workflows.
- Run minimal build-time checks: main help, version, helper help, Python import,
  or equivalent runtime checks.
- Use `--no-install-recommends`; clean apt lists and build caches.
- Prefer multi-stage builds when compilation is needed.
- Do not remove official data, config, or documentation that users may need just
  to reduce image size.

Acceptable tradeoffs:

- Bioinformatics images are often large. Size is acceptable when the image is
  complete, reliable, and explainable.
- If upstream recommends conda, Docker already provides isolation. Prefer direct
  system or Python installation when it is reproducible enough; introduce conda
  only when dependencies are otherwise difficult to reproduce.

## `docs/help.md`

`docs/help.md` is what users see from `taf-xxx --help`. It should read like a
command manual, not a project overview.

Recommended shape:

```text
taf-my-tool 1.0.0-r1

One-line description.

Usage:
  taf-my-tool [TAF-APP-OPTION]
  taf-my-tool [UPSTREAM-ARGS...]
  taf-my-tool -- [UPSTREAM-ARGS...]
  taf-my-tool my-tool [UPSTREAM-ARGS...]
  taf-my-tool <COMMAND> [COMMAND-ARGS...]

TAF app options:
  -h, --help       Show this help text
  -v, --version    Show package and command version
  --compile        Print generated shell code instead of running it
  --               Pass all following arguments to upstream command

Upstream help:
  taf-my-tool -- --help

Default examples:
  ...

Explicit container command examples:
  ...

Container commands:
  ...

Notes:
  ...

Container:
  image: ghcr.io/taffish/my-tool:1.0.0-r1
  supported backends: apptainer, podman, docker

Upstream:
  project:
  source:
  docs:
```

Notes:

- Keep help short, accurate, and copyable.
- Explain `--`, because `--help` and `--version` may be handled by the TAFFISH
  wrapper.
- Explain command mode: `taf-my-tool COMMAND ...` runs `COMMAND` inside the same
  container.
- List important helper commands available in the container.
- Do not copy large sections of upstream manuals. Link to upstream docs instead.

## `README.md`

README is for the GitHub page and human review. It can explain the package
structure more than `docs/help.md`.

Recommended sections:

```text
# taf-my-tool

Short description.

## Installation
taf update
taf install my-tool
taf install my-tool 1.0.0-r1
taf install --from .

## Usage
show help
run upstream command
run explicit container commands

## Package
name / command / version / kind / image

## Container
installed commands
installed libraries
build-time checks

## Upstream
project / source / documentation / release page

## Maintainer Notes
taf check
taf compile -- --help
taf publish --release --dry-run
docker build --check -f docker/Dockerfile .
```

Current recommendation:

- Keep official app READMEs in English.
- Put Chinese explanations in central docs or the website instead of mixing
  long bilingual content in every app README.
- Do not promise features that the Dockerfile does not yet provide.

## `release.md`

`release.md` is a local release draft used by `taf publish --release`. It is
usually ignored by git and not committed to the app repository.

Rules:

- The first line becomes part of the publish message.
- The entire file becomes GitHub Release notes.
- The first line must not be the default TODO placeholder.
- The first line should describe the real change in this release.

Initial release example:

```md
# Publish My Tool 1.0.0 as a TAFFISH app

This release packages My Tool 1.0.0 as `taf-my-tool`.
```

Packaging fix example:

```md
# Fix My Tool 1.0.0 smoke metadata

This release updates TAFFISH smoke metadata without changing the upstream
My Tool version.
```

Important principles:

- App releases describe app changes.
- If a failure comes from an index bug, do not bump the app release only to work
  around that bug.
- Do not overwrite published tags. If the app itself needs a fix, increment
  `release`.

## Smoke Design

Smoke is the core public index gate. It is not a full test suite. It cheaply
proves that:

- The container image can be pulled and inspected for digest/platform metadata.
- Key commands exist in `PATH`.
- The main command can start and print help or version.
- Key runtime dependencies can be imported or executed.
- The smoke container can pass without network access.

Recommended example:

```toml
[smoke]
backend = "docker"
timeout = 120
exist = ["my-tool", "helper-command", "python"]
test = ["my-tool --help", "my-tool --version 2>&1 | grep -F '1.0.0' >/dev/null", "helper-command --help", "python -c \"import my_tool\""]
```

In TOML basic strings, `\"` is valid and should parse to a normal double quote.
For easier shell review, this is also fine:

```toml
test = ["python -c 'import my_tool'"]
```

Notes:

- The current `taf` project parser expects arrays to be written on one line, so
  do not split `exist` or `test` into multi-line TOML arrays in app projects.
- Put command names in `exist`, not complex shell fragments.
- Put short commands in `test`; do not require network, large data, or
  credentials.
- Make version checks precise, but not so fragile that a harmless upstream
  banner change breaks indexing.
- For Python/R/Perl environments, import checks are valuable.
- For images with upstream helper scripts, include important scripts in smoke.
- `taf check` validates smoke metadata but does not run containers. Real smoke
  execution belongs to Hub/index automation.

## Version And Release Policy

Version ids have this shape:

```text
<upstream-version>-r<taffish-release>
```

Keep the same app release when:

- Only the index, `taf`, website, or documentation system is fixed.
- Only the index action is re-run.
- The already published app tag content did not change.

Increment `release` when:

- The Dockerfile changes.
- `src/main.taf` changes.
- `taffish.toml` fields that affect installation, runtime, containers, or smoke
  change.
- README/help/release changes correspond to a new formal app publication.
- The same upstream version needs a TAFFISH packaging fix and a new publication.

Increment the upstream version when:

- The wrapped upstream software changes, for example from `1.2.7` to `1.2.8`.

Do not overwrite published tags. The public index trust model depends on stable
release refs, source commits, container digests, and smoke/trust records.

## Pre-Publish Checks

Minimum checks:

```sh
taf check
taf compile -- --help
taf publish --release --dry-run
```

Container checks:

```sh
docker build --check -f docker/Dockerfile .
docker build -t ghcr.io/taffish/my-tool:1.0.0-r1 -f docker/Dockerfile .
docker run --rm ghcr.io/taffish/my-tool:1.0.0-r1 my-tool --help
```

If you use Podman locally:

```sh
podman run --rm ghcr.io/taffish/my-tool:1.0.0-r1 my-tool --help
```

Checklist:

- `taffish.toml` has no `TODO`.
- `docs/help.md` exists and matches command, image, and version.
- `README.md` matches command, image, and version.
- `release.md` first line is real, concise, and not a default placeholder.
- `src/main.taf` image tag matches `[container].image`.
- The image built by Dockerfile contains all commands declared in smoke.
- `taf publish --release --dry-run` reports the expected tag and message.

## Post-Publish Checks

After publishing:

1. Wait for the app repository image build workflow to pass.
2. Confirm that GHCR can be read by the index workflow.
3. Run or wait for the `taffish-index` workflow.
4. Inspect `index/reports/latest.json`.
5. Confirm the app appears in `index/index.json`,
   `index/packages/<name>.json`, and command index files.
6. Locally run `taf update`, `taf info <app>`, and `taf install <app>`.

If indexing fails, first decide whether the cause is:

- App metadata or smoke.
- Image build failure or GHCR visibility.
- Index builder, TOML parser, or trust gate bug.

Do not rush into a new app release before identifying the root cause. Fix index
bugs in the index; fix app bugs in the app.

## Things Not To Do

Do not:

- Add unsupported fields to `taffish.toml`.
- Use `latest` as a formal image tag.
- Overwrite published tags.
- Bump an app release because of an index bug.
- Edit `target/` artifacts by hand.
- Remove user-facing functionality only to pass smoke.
- Write a Dockerfile that only satisfies smoke but cannot support real user
  workflows.
- Make smoke depend on network access, remote databases, credentials, or large
  test data.
- Copy large upstream manual sections into `docs/help.md`.
- Maintain long bilingual content in every app README.

## Augustus Template Notes

`augustus` is the first formal app in the official Hub and is a good template
because it follows these principles:

- `taffish.toml` stays close to the `taf new` structure and uses only supported
  fields.
- `src/main.taf` is a thin wrapper that delegates to upstream `augustus`.
- Dockerfile keeps user-relevant `augustus-data` and `augustus-doc`.
- Smoke first checks a functional entry point, then checks the exact version.
- `docs/help.md` explains default calls, `--`, command mode, and container
  metadata.
- `README.md` explains installation, usage, package metadata, container
  contents, upstream source, and maintainer checks.
- `release.md` first line can be used as the publish message, and the full file
  can be used as GitHub Release notes.

New apps should first meet the `augustus` bar before adding more complex
wrappers, richer tests, or special container behavior.
