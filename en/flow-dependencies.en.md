# Flow And Dependencies Guide

Flow apps are TAFFISH's mechanism for composing multiple apps. A flow uses the `<taffish>` tag and `[[taf: ...]]` syntax to call other taf apps, and declares install dependencies through `[dependencies]`.

## Table Of Contents

- [Role Of A Flow](#role-of-a-flow)
- [Basic Form](#basic-form)
- [`[[taf: ...]]`](#taf-)
- [`@:` Parameter Blocks](#-parameter-blocks)
- [Dependency Declaration](#dependency-declaration)
- [Multiple Versions](#multiple-versions)
- [Dependency Sync](#dependency-sync)
- [Install Behavior](#install-behavior)
- [Design Advice](#design-advice)
- [Common Mistakes](#common-mistakes)

## Role Of A Flow

Tool apps wrap individual upstream tools. Flow apps compose multiple tool apps or flow apps into a workflow.

Flow apps are suitable for:

- QC workflows.
- Alignment workflows.
- Assembly workflows.
- Annotation workflows.
- Multi-step bioinformatics pipelines.

Flow apps are not ideal for:

- Wrapping one upstream tool.
- Hiding a large pile of hard-to-maintain shell logic.
- Implicitly depending on undeclared external commands.

## Basic Form

`src/main.taf`:

```taf
<taffish>
[[taf: taf-fastqc-v0.12.1-r1 sample.fq]]
[[taf: taf-multiqc-v1.19-r1 .]]
```

`taffish.toml`:

```toml
[package]
name = "qc-flow"
kind = "flow"
version = "0.1.0"
release = 1
license = "Apache-2.0"
main = "src/main.taf"

[repository]
url = "https://github.com/taffish/qc-flow"

[command]
name = "taf-qc-flow"

[runtime]
pipe = false
command_mode = false

[dependencies]
taf-fastqc = "0.12.1-r1"
taf-multiqc = "1.19-r1"
```

## `[[taf: ...]]`

Syntax:

```text
[[taf: taf-command args...]]
```

Prefer exact versioned commands in published flows:

```taf
[[taf: taf-fastqc-v0.12.1-r1 sample.fq]]
```

During development, the base command can be used:

```taf
[[taf: taf-fastqc sample.fq]]
```

For official releases, fixed versions are recommended because:

- The index can record dependencies explicitly.
- `taf install` can install the correct versions.
- Future upstream app releases will not silently change workflow behavior.

## `@:` Parameter Blocks

Flows often need to connect user-facing domain parameters with low-level tool parameters. `(@:)` block parameters are designed for this.

Example:

```taf
ARGS
<!(--/-i)input>
<(--/-o)outdir>
  qc

<(@:)fastqc-step>
  --threads ::(--/-t)threads=4::
  --outdir ::outdir::
  ::(--/-e)fastqc-extra=::

RUN
<taffish>
mkdir -p ::outdir::
[[taf: taf-fastqc-v0.12.1-r1 ::(@:)fastqc-step:: ::input::]]
```

`(@:)fastqc-step` creates a block parameter named `fastqc-step`. Its default value is a whole argument fragment, not a single string. That fragment can contain ordinary parameter DSL:

- `threads` exposes thread count.
- `outdir` reuses the flow output directory.
- `fastqc-extra` keeps an advanced extra-argument entry point.

Inside an `ARGS` body, prefer `::name::` for ordinary parameter references. `@name` and `@{name}` are also supported, but they are better reserved for default-value composition such as `::(--/-p)prefix=out-@{input}::`.

This pattern is useful because it:

- Keeps arguments for one tool step together in `ARGS`.
- Keeps `[[taf: ...]]` calls short and readable.
- Avoids promoting every low-level tool argument into a top-level flow concept.
- Lets advanced users pass extra tool arguments without disrupting the workflow structure.

If a flow has multiple steps, create one block parameter per step, such as `fastqc-step`, `align-step`, and `call-variants-step`. A block parameter should correspond to one clear step.

## Dependency Declaration

Every taf app called by a flow should be declared in `[dependencies]`:

```toml
[dependencies]
taf-fastqc = "0.12.1-r1"
taf-multiqc = "1.19-r1"
```

The key is the base command name without the version:

```text
taf-fastqc
```

The value is a version id:

```text
0.12.1-r1
```

This maps to the versioned command:

```text
taf-fastqc-v0.12.1-r1
```

## Multiple Versions

One flow may require multiple versions of the same app:

```taf
<taffish>
[[taf: taf-align-v2.0.0-r1 old.fq]]
[[taf: taf-align-v2.1.0-r1 new.fq]]
```

Then `[dependencies]` should use an array:

```toml
[dependencies]
taf-align = ["2.0.0-r1", "2.1.0-r1"]
```

This means both versions are required. It is not an alternatives list.

## Dependency Sync

`taf build` syncs `[dependencies]` from `src/main.taf` and removes stale dependencies.

Typical workflow:

```sh
taf check
taf build
taf check
```

If `src/main.taf` calls:

```taf
[[taf: taf-dep-tool-v0.1.0-r1 in.fa]]
[[taf: taf-new-tool-v1.2.3-r4 out.fa]]
```

then after:

```sh
taf build
```

`taffish.toml` should contain:

```toml
[dependencies]
taf-dep-tool = "0.1.0-r1"
taf-new-tool = "1.2.3-r4"
```

`taf check` will report errors when:

- A flow uses `[[taf: ...]]` but `[dependencies]` is missing.
- The dependency version does not match the versioned command in `src/main.taf`.
- Multiple versions are needed but the dependency value is only a single string.

## Install Behavior

When a user installs a flow:

```sh
taf install qc-flow
```

`taf install` reads dependencies from the index, installs dependencies first, then installs the flow itself.

Index representation:

```json
"dependencies": {
  "taf-fastqc": ["0.12.1-r1"],
  "taf-multiqc": ["1.19-r1"]
}
```

For multiple versions:

```json
"dependencies": {
  "taf-align": ["2.0.0-r1", "2.1.0-r1"]
}
```

The installer should treat every listed version as required.

## Design Advice

- Keep each flow focused on one clear analysis task.
- Use exact versioned taf commands in published flows.
- Expose a compact set of user-facing parameters.
- Use `@:` block parameters to group per-step tool arguments.
- Split complex logic into tool apps or subflows.
- Keep `[dependencies]` synchronized with `src/main.taf`.
- Document input, output, and expected files in `docs/help.md`.

Avoid:

- Calling undeclared system commands directly in a flow.
- Mixing many environment assumptions inside one flow.
- Depending on unversioned app commands in official releases.
- Treating dependency arrays as alternatives.
- Hiding important behavior in undocumented shell fragments.

## Common Mistakes

`taf check` reports a missing dependency:

- Run `taf build` to sync dependencies.
- Check whether `src/main.taf` uses exact versioned commands.
- Check whether `[dependencies]` was edited manually.

Install succeeds but the flow fails:

- Confirm dependency commands are installed.
- Confirm the flow calls exact versioned commands.
- Confirm container images for tool dependencies are public and pullable.

Same app appears twice:

- Check `[dependencies].taf-x`.
- Use an array only when both versions are truly required.
- Do not use arrays to mean "choose one".

