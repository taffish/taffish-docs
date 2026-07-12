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
[[taf: taf-fastqc-v0.12.1-r1 fastqc sample.fq]]
[[taf: taf-multiqc-v1.19-r1 multiqc .]]
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
[[taf: taf-fastqc-v0.12.1-r1 fastqc sample.fq]]
```

During development, the base command can be used:

```taf
[[taf: taf-fastqc sample.fq]]
```

For official releases, fixed versions are recommended because:

- The index can record dependencies explicitly.
- `taf install` can install the correct versions.
- Future upstream app releases will not silently change workflow behavior.

Place `[[taf: ...]]` at the actual call site where the tool is executed. A good
mental model is: write the flow as ordinary shell first, then replace each core
bioinformatics command with `[[taf: taf-xxx-vA.B.C-rN ...]]`.

Example:

```taf
export bam flagstat log
if [[taf: taf-samtools-v1.23.1-r1 samtools flagstat '"$bam"' ]] </dev/null > "$flagstat" 2> "$log"; then
    :
else
    echo "samtools flagstat failed; see $log" >&2
    exit 1
fi
```

Inside loops, call the dependency inside the loop body where that step really
runs:

```taf
while IFS="$(printf '\t')" read -r sample_id bam || [ -n "$sample_id" ]; do
    [[taf: taf-rseqc-v5.0.4-r2 bam_stat.py -i '"$bam"' -q '"$mapq"' ]] \
        </dev/null \
        > "$outdir/03_results/rseqc/$sample_id.bam_stat.txt" \
        2>> "$outdir/01_logs/steps/03_rseqc.log"
done < "$outdir/00_inputs/bam_files.tsv"
```

Do not collect all `[[taf: ...]]` calls in an `if false` block, dummy block, or
"dependency declaration" section at the top of the file and then run them
indirectly through `taffish_step_*`, `*_STEP`, `TAF_CMD`, or bare
`taf-xxx-v...` wrappers. That pattern may expose dependencies to the compiler,
but it hides the real execution path from the source code and weakens review.

In formal flows, apart from a small amount of shell/coreutils usage, any
bioinformatics, statistical, plotting, database, model, or other domain tool
that a user would otherwise need to install should first be versioned as a taf
app and then called through a real call-site `[[taf: ...]]`. In other words, the
flow source should show which taf app, which version, and which arguments are
used for a step. That information should not appear only in documentation,
`commands.sh`, or an indirect variable.

When arguments come from runtime shell variables, preserve runtime expansion.
In the examples above, `'"$bam"'` keeps `"$bam"` in the generated shell script
instead of expanding it while TAFFISH compiles the step. Export those variables
before the call, because dependency invocations may be materialized as separate
step shells. Inside `while read ... < file` loops, add `</dev/null` to taf calls
so a tool or container entrypoint cannot consume the loop input.

For variable-length argument lists, prefer a small number of explicit branches
so the `[[taf: ...]]` call remains fixed and readable. If the number of
arguments can only be known at runtime, write a concrete helper script under
`<outdir>/02_intermediate/` and execute it through the relevant taf dependency at
the real call site, for example:

```taf
[[taf: taf-tool-vX-rY sh '"$script"' ]] </dev/null
```

This keeps dependency execution auditable; `commands.sh` remains provenance text
only.

## `@:` Parameter Blocks

Flows often need to connect user-facing domain parameters with low-level tool parameters. `(@:)` block parameters are designed for this.

Example:

```taf
ARGS
<!(--/-i)input>
<(--/-o)outdir>
  qc
<(--/-t)threads>
  4

<(@:)fastqc-step>

RUN
<taffish>
mkdir -p ::outdir::
[[taf: taf-fastqc-v0.12.1-r1 fastqc --threads ::threads:: --outdir ::outdir:: ::input:: ::(@:)fastqc-step::]]
```

`(@:)fastqc-step` creates a step parameter slot named `fastqc-step`. In formal
flows, this slot should be empty by default: stable parameters managed by the
flow itself, such as `threads`, `outdir`, and `input`, are written directly in
the command body. `::(@:)fastqc-step::` is only for native FastQC arguments that
the user explicitly appends for this run.

- `threads` is exposed as a common top-level flow parameter.
- `outdir` and `input` are part of the flow-managed contract.
- `fastqc-step` is an advanced argument slot and expands only when the user
  passes `@fastqc-step: ... @:`.

Inside an `ARGS` body, prefer `::name::` for ordinary parameter references. `@name` and `@{name}` are also supported, but they are better reserved for default-value composition such as `::(--/-p)prefix=out-@{input}::`.

This pattern is useful because it:

- Exposes stable, common, result-relevant settings as top-level flow parameters.
- Keeps the upstream executable and flow-managed arguments explicit in `RUN`.
- Avoids promoting every low-level tool argument into a top-level flow concept.
- Lets advanced users pass extra tool arguments without disrupting the workflow structure.

If a flow has multiple steps, create one block parameter per step, such as `fastqc-step`, `align-step`, and `call-variants-step`. A block parameter should correspond to one clear step.

For formal flows, this should be treated as the default rule: each real
analytical `[[taf: ...]]` call site that affects results, resource use, or major
report content should have a unique and descriptive `(@:)<step-name>` parameter
block, and the call site should include `::(@:)<step-name>::`. Version probes,
dependency anchors, and very lightweight existence checks may be exceptions
when they do not carry real analytical semantics.

If the same taf app is called multiple times in one flow, split block names by
call site. For example, one `taf-salmon` dependency may need
`salmon-index-step`, `salmon-quant-pe-step`, and `salmon-quant-se-step`. Do not
use one broad `salmon-step` or `extra-step` to control semantically different
calls.

`@:` blocks are advanced entry points for passing native low-level tool
arguments. They should not be required for normal execution.
`::(@:)<step-name>::` must expand to nothing by default; do not prefill it in
flow source code, templates, shell variables, or helper scripts. Default tool
arguments that belong to the flow should be written directly in the command body
and documented as normal behavior. A flow should run its standard path, produce
results, reports, and provenance with only domain parameters and common options.
User-supplied step parameters should also be recorded in `commands.sh`,
`run.manifest.json`, or equivalent provenance.

If a step must execute the underlying tool through a helper script,
`::(@:)<step-name>::` should be included in the real tool command generated by
that helper script, not merely appended after `sh "$script"`. Otherwise the
user's low-level arguments may never reach the intended tool.

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

Do not put the version in the dependency key:

```toml
taf-fastqc-v0.12.1-r1 = "0.12.1-r1"
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
