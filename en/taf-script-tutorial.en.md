# TAF Script Tutorial

[English](taf-script-tutorial.en.md) | [中文](../zh/taf-script-tutorial.cn.md)

This tutorial teaches the practical `.taf` syntax used by TAFFISH app authors.
It is not a guide to TAFFISH compiler source code.

## Table Of Contents

- [Mental Model](#mental-model)
- [A Minimal Script](#a-minimal-script)
- [`ARGS` And `RUN`](#args-and-run)
- [Required Parameters](#required-parameters)
- [Defaults And Flags](#defaults-and-flags)
- [Positional Parameters](#positional-parameters)
- [Parameter References](#parameter-references)
- [Block Parameters](#block-parameters)
- [Built-In Parameters](#built-in-parameters)
- [Container Tags](#container-tags)
- [Tool App Wrapper](#tool-app-wrapper)
- [Flow App With Dependencies](#flow-app-with-dependencies)
- [Help Text](#help-text)
- [Check, Run, Build](#check-run-build)
- [Common Patterns](#common-patterns)

## Mental Model

A `.taf` file is close to shell, but with a small amount of extra structure:

- `ARGS` declares user-facing parameters.
- `RUN` contains shell-like commands.
- `::name::` inserts a resolved parameter.
- `<container:...>` runs commands through a container backend.
- `<taffish>` lets a flow call other TAFFISH apps with `[[taf: ...]]`.

TAFFISH compiles this into shell. `taf run` runs the compiled shell. `taf build`
freezes the project into a versioned command wrapper.

## A Minimal Script

Create `hello.taf`:

```taf
<shell>
echo "hello from TAFFISH"
```

Compile it:

```sh
taffish hello.taf
```

In a project, run the current app with:

```sh
taf run
```

If the first effective line is a subtag such as `<shell>`, TAFFISH inserts
`RUN` automatically.

## `ARGS` And `RUN`

Use `ARGS` to define parameters, then reference them in `RUN`:

```taf
ARGS
<name>
  world

RUN
<shell>
echo "hello, ::name::"
```

Run:

```sh
taf run -- --name Alice
```

The text under `<name>` is the default value. User input overrides it.

## Required Parameters

Prefix a parameter with `!` to make it required:

```taf
ARGS
<!(--/-i)input>
<(--/-o)output>
  result.txt

RUN
<shell>
my-tool --input ::input:: --output ::output::
```

This declares:

```text
--input / -i     required
--output / -o    optional, default result.txt
```

## Defaults And Flags

Inline defaults are useful for short values:

```taf
ARGS
<(--/-t)threads=4>
<(--/-v)verbose?>

RUN
<shell>
echo "threads: ::threads::"
echo "verbose: ::verbose::"
```

`?` declares a boolean flag. If the user provides `--verbose`, the value becomes
true; otherwise it is false.

## Positional Parameters

Use `$1`, `$2`, and so on for positional arguments:

```taf
ARGS
<!$1>
<$2>
  result.txt

RUN
<shell>
cp ::$1:: ::$2::
```

For public app commands, named options are usually clearer than positional
arguments, but positional arguments are useful for small wrappers.

## Parameter References

Inside `RUN`, use `::name::`:

```taf
<shell>
echo "::sample::"
```

Inside `ARGS` default bodies, prefer `::name::` for ordinary references:

```taf
ARGS
<sample>
  SRR000001
<prefix>
  out/::sample::
```

`@name` and `@{name}` also work in default expressions. Use `@{name}` when a
reference is adjacent to letters, numbers, `_`, or `-`:

```taf
ARGS
<sample>
  demo
<prefix>
  out-@{sample}
```

Only documented escape forms are interpreted by TAFFISH. Most backslash
sequences are preserved as ordinary text.

## Block Parameters

`(@:)name` creates a block parameter. It is useful when a flow step needs a
whole argument fragment:

```taf
ARGS
<!(--/-q)query>
<(--/-d)db>
  nt

<(@:)blast-step>
  --query ::query::
  --db ::db::
  --outfmt ::(--/-f)outfmt=6::
  ::(--/-e)extra=::

RUN
<taffish>
[[taf: taf-blast-v2.16.0-r1 ::(@:)blast-step::]]
```

This keeps the main flow readable while still exposing important low-level
options.

## Built-In Parameters

Common built-ins:

```text
::*USER*::
::*HOMEDIR*::
::*WORKDIR*::
::*LOADDIR*::
::*ARGV*::
::*CMD*::
::*CPUS*::
::*CONTAINER*::
```

For tool wrappers that forward all upstream arguments, `::*ARGV*::` is common:

```taf
<taf-app:container:ghcr.io/taffish/augustus:3.5.0-r1>
augustus ::*ARGV*::
```

## Container Tags

Generic container backend:

```taf
<container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --help
```

App-level container wrapper:

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool ::*ARGV*::
```

At compile time, a generic container tag becomes a backend command such as
Docker, Podman, or Apptainer. In simplified Docker/Podman form:

```sh
podman run --rm -i \
  -w "$PWD" \
  -v "$HOME:$HOME" \
  -v "$PWD:$PWD" \
  ghcr.io/taffish/my-tool:0.1.0-r1 \
  my-tool ::*ARGV*::
```

This automatic working-directory mount is why local input and output paths work.
It also means you should run containerized apps from project or data
directories, not from host system directories such as `/usr/bin`, because the
mount can hide the image's own `/usr/bin`.

Force a backend during local testing:

```sh
taf run --backend docker -- --help
taf run --backend podman -- --help
taf run --backend apptainer -- --help
```

Image building currently uses Docker or Podman:

```sh
taf build --image --backend docker
taf build --image --backend podman
```

Use the same backend for local image build and local run, or the runtime may not
find the image you just built.

## Tool App Wrapper

A simple tool app normally keeps the upstream CLI intact:

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool ::*ARGV*::
```

This means:

```sh
taf-my-tool -- --help
taf-my-tool -- --input sample.fa --output result.txt
```

Do not add shell builtins such as `exec` unless the generated runtime actually
runs through a shell context that supports them. In a container command wrapper,
`exec` may be treated as an executable name and fail.

## Flow App With Dependencies

A flow app calls exact app versions:

```taf
ARGS
<!(--/-i)input>
<(--/-o)outdir>
  qc

RUN
<taffish>
mkdir -p ::outdir::
[[taf: taf-fastqc-v0.12.1-r1 ::input:: --outdir ::outdir::]]
[[taf: taf-multiqc-v1.19-r1 ::outdir::]]
```

Declare dependencies in `taffish.toml`:

```toml
[dependencies]
taf-fastqc = "0.12.1-r1"
taf-multiqc = "1.19-r1"
```

If a flow requires multiple versions of the same app:

```toml
[dependencies]
taf-x = ["0.1.0-r1", "0.1.0-r2"]
```

Arrays mean all listed versions are required, not alternatives.

## Help Text

Every app should include `docs/help.md`. Built commands print this text for:

```sh
taf-my-tool --help
```

The help text is plain terminal output, not rendered Markdown. Keep it readable
without Markdown rendering.

## Check, Run, Build

Inside an app project:

```sh
taf check
taf run -- --help
taf build
```

For containerized apps during development:

```sh
taf build --image --backend docker
taf run --backend docker -- --help
taf build
```

The final `taf build` writes:

```text
target/taf-my-tool-v0.1.0-r1
target/.taf-my-tool-v0.1.0-r1/
```

The generated command uses the frozen snapshot under `target/`.

## Common Patterns

Forward all arguments to upstream:

```taf
<taf-app:container:ghcr.io/taffish/tool:1.0.0-r1>
tool ::*ARGV*::
```

Expose a small stable interface:

```taf
ARGS
<!(--/-i)input>
<(--/-o)output>
  result.txt
<(--/-t)threads>
  4

RUN
<taf-app:container:ghcr.io/taffish/tool:1.0.0-r1>
tool --input ::input:: --output ::output:: --threads ::threads::
```

Use `--` when passing option-like arguments through `taf run` or built commands:

```sh
taf run -- --input sample.fa
taf-my-tool -- --help
```
