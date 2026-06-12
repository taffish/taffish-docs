# TAFFISH Flow Curation Checklist

Use this checklist before publishing or reviewing a taf-flow app.

It pairs with the TAFFISH Flow Developer Guide. The guide explains how to build a flow; this checklist decides whether the flow is clear, reliable, and publishable.

## 0. Basic conclusion

Start the release review with a clear conclusion:

```text
Conclusion: publishable / not publishable yet
Main reasons:
- ...
Blockers:
- ...
```

Do not publish while any blocker remains.

## 1. Identity and scope

- [ ] The flow name describes the task, not only a tool.
- [ ] `taffish.toml` type, version, description, and keywords are accurate.
- [ ] README explains what problem the flow solves.
- [ ] README explains what the flow does not solve.
- [ ] The flow is not a duplicate wrapper around one tool.
- [ ] The steps are common or easy to justify in the field.
- [ ] Defaults do not overpromise.

## 2. Dependencies and reproducibility

- [ ] All core bioinformatics commands come from explicit TAFFISH dependencies.
- [ ] Dependency versions in `taffish.toml` are exact.
- [ ] `[[taf: ...]]` calls in `src/main.taf` match `taffish.toml`.
- [ ] `[[taf: ...]]` appears at the real call site where the tool is executed,
      not only in an `if false` / dummy block at the top followed by indirect
      execution through `taffish_step_*`, `*_STEP`, `TAF_CMD`, or bare
      `taf-xxx-v...` wrappers.
- [ ] Each real analytical `[[taf: ...]]` call site that affects results,
      resource use, or major report content has a unique and descriptive
      `(@:)<step-name>` block, and the call site includes
      `::(@:)<step-name>::`; version probes, dependency anchors, and very
      lightweight checks have clear exception reasons.
- [ ] If the same taf app is called multiple times in one flow, `@:` blocks are
      split by call site; the flow does not use one broad `extra-step` or
      app-level block to control several semantically different steps.
- [ ] The default path runs without any `@step:` arguments and still produces
      standard results, reports, and provenance; `@:` is an expert extension
      entry point, not a required normal-use parameter.
- [ ] User-supplied step parameter blocks are recorded in `commands.sh`,
      `run.manifest.json`, or equivalent provenance for audit and reproduction.
- [ ] Runtime shell variables used inside `[[taf: ...]]` still expand in the
      generated shell script; they are not expanded too early while TAFFISH
      compiles the step.
- [ ] Variables referenced as `'"$var"'` are exported before the call; taf calls
      inside loops use `</dev/null` and cannot consume `while read ... < file`
      input.
- [ ] Variable-length argument lists are not executed through bare wrappers,
      `eval`, or the host PATH; if a helper script is needed, it lives in the
      output directory and is executed through a real call-site `[[taf: ...]]`.
- [ ] The flow does not use host `fastqc`, `samtools`, `bwa`, `STAR`, or similar tools.
- [ ] Shell/coreutils assumptions are covered by smoke tests.
- [ ] Normal execution does not download core resources from the network.
- [ ] Large databases, reference genomes, and models are not embedded in the flow image.
- [ ] Platform limits, GPU needs, commercial licenses, or non-free licenses are documented.

## 3. Input contract

- [ ] Required inputs are clear.
- [ ] Optional parameters and defaults are clear.
- [ ] Sample-sheet format has an example.
- [ ] Sample-sheet columns, path rules, and sample-name rules are documented.
- [ ] Missing files, duplicate samples, empty sample names, and malformed sheets fail early.
- [ ] Paired-end, single-end, long-read, or other mode boundaries are clear.
- [ ] Reference, index, and database parameters have format requirements.
- [ ] Documentation says which inputs are read and which may be indexed or cached.

## 4. Output directory and writes

- [ ] The flow requires explicit `--outdir`.
- [ ] All generated content is written under `<outdir>/...`.
- [ ] The current directory is not polluted.
- [ ] Input and reference files are not modified.
- [ ] Existing output directories are rejected by default.
- [ ] `--force`, if present, is documented and cannot delete inputs.
- [ ] `--resume`, if present, documents reusable steps and risks.
- [ ] Temporary files live under `<outdir>/02_intermediate/` or `<outdir>/.tmp/`.
- [ ] Logs remain available under `<outdir>/01_logs/` after failure.

Recommended layout:

```text
<outdir>/
  00_inputs/
  01_logs/
  02_intermediate/
  03_results/
  04_reports/
  run.manifest.json
```

## 5. Steps and data flow

- [ ] README explains how data moves from input to output.
- [ ] Each step has input, tool, and output descriptions.
- [ ] Major step logs are saved separately.
- [ ] Key intermediate files have stable names.
- [ ] Final results and intermediate files are separated.
- [ ] A failure can be traced to a specific step.
- [ ] Important upstream tool parameters are recorded.

## 6. Results and reports

- [ ] `03_results/` stores machine-readable primary results.
- [ ] `04_reports/` stores user-facing reports.
- [ ] A project-level or sample-level summary exists, such as `flow_summary.tsv`.
- [ ] README explains what each report is for.
- [ ] HTML, PDF, TSV, JSON, or other formats match field practice.
- [ ] If a final HTML report is produced, it is self-contained / standalone
      where possible: main CSS/JS/logo/PNG figures/key summary tables and
      explanatory text are embedded, so copying the HTML alone still preserves
      the main reading experience.
- [ ] If a formal HTML report is produced, its structure follows the shared
      TAFFISH flow-report template contract; the flow does not invent an
      incompatible one-off report shell, navigation model, language switch, or
      deliverable structure.
- [ ] External file links in the HTML are enhancements only; full matrices,
      PDFs, logs, MultiQC bundles, commands, manifests, and other audit files
      remain in the output directory but are not required for reading the main
      report.
- [ ] If the HTML report embeds MultiQC, FastQC, Qualimap, or similar QC
      subreports, it embeds the original HTML bundle generated by the tool, not
      a hand-written look-alike page; the original bundle remains in the output
      directory for audit.
- [ ] Embedded HTML/QC subreports open from the main single HTML in a separate
      local page by default; for JavaScript-heavy MultiQC/Plotly reports, the
      flow does not default to top-level `blob:`, iframe, or `srcdoc` delivery
      that may cause plot initialization failures or blank sections.
- [ ] Embedded subreports have provenance: source path, source bytes, embedded
      bytes, status, and unresolved resources are recorded; the payload JSON in
      the main HTML parses completely and is not truncated by nested
      `</script>` content.
- [ ] For self-contained single-file MultiQC/FastQC reports, the embedded
      payload matches the original HTML bytes or hash unless documentation
      explains a necessary conversion.
- [ ] Large interactive QC reports are checked primarily through static
      payload/resource validation and ordinary-browser manual inspection when
      needed; one automated browser click failure on a real MultiQC page is not
      used as the only acceptance criterion.
- [ ] If reports are published as GitHub Pages or similar public examples,
      generated HTML, manifests, commands, provenance, and embedded-report
      indexes do not leak maintainer-local absolute paths such as `/home/...`,
      `/Users/...`, or temporary directories; public examples use stable
      placeholders or relative paths.
- [ ] Generated HTML in public examples passes whitespace checks such as
      `git diff --check`, including HTML copied or embedded from real QC tools.
- [ ] Large standalone HTML files in public examples have been reviewed for
      repository size: files above GitHub's 50 MB recommendation are
      intentional and explained, and ordinary tracked files do not approach
      GitHub's 100 MB hard limit.
- [ ] Empty, low-quality, or failed samples are marked clearly.
- [ ] Documentation says which outputs are for downstream analysis and which are for manual inspection.

## 7. Provenance

- [ ] `04_reports/commands.sh` is produced.
- [ ] `04_reports/versions.tsv` is produced.
- [ ] `04_reports/methods.txt` is produced.
- [ ] `<outdir>/run.manifest.json` is produced.
- [ ] `commands.sh` records key commands and actual parameters.
- [ ] `versions.tsv` records the flow and main dependency versions.
- [ ] `run.manifest.json` records inputs, parameters, dependencies, outputs, and timestamps.
- [ ] Provenance files do not contain tokens, passwords, or other secrets.

## 8. Resource requirements

- [ ] README states minimum resources.
- [ ] README states recommended resources.
- [ ] Disk use is related to input scale.
- [ ] CPU/thread behavior is explained.
- [ ] Memory-heavy steps are identified.
- [ ] Database or index size is documented separately.
- [ ] amd64-only dependencies on arm64 hosts document emulation impact.

## 9. Smoke and validation

- [ ] Smoke does more than `--help`.
- [ ] Smoke checks that the flow command exists.
- [ ] Smoke checks version/provenance records produced by the flow, rather than
      replacing flow validation with direct dependency version commands.
- [ ] Smoke/formal runs `taf build` first and uses this flow's
      `target/taf-<flow>-v...` as the primary subject under test.
- [ ] Smoke/formal pass conditions come from outputs, logs, and provenance
      produced by the flow wrapper.
- [ ] Downstream smoke/formal tests that need upstream outputs prefer upstream
      subflow target wrappers to generate those inputs.
- [ ] Direct dependency wrapper calls in tests are limited to fixture setup when no
      suitable upstream subflow exists, or to environment probes, and do not replace
      the main flow-wrapper validation.
- [ ] Smoke runs a tiny fixture through a real data path.
- [ ] Smoke checks key result files.
- [ ] Smoke checks `commands.sh`, `versions.tsv`, and `run.manifest.json`.
- [ ] Smoke checks that the current directory has no unexpected new files.
- [ ] Smoke is fast and small enough for local or CI use.
- [ ] Local maintenance smoke respects an existing `TAFFISH_CONTAINER_BACKEND` and defaults to `podman` when it is unset.
- [ ] Smoke does not hard-code the Docker backend.
- [ ] Smoke does not call `docker pull`, `podman pull`, or re-download dependency images.
- [ ] Missing dependency app images fail with a clear preparation message.
- [ ] Documentation states that smoke is not scientific validation.

## 10. Documentation

- [ ] New flow docs or major README/help rewrites started from
      `repos/apps/templates/app-docs/flow/`, unless the README documents a
      reason to diverge.
- [ ] README explains task, inputs, outputs, examples, and limits.
- [ ] `docs/help.md` is plain terminal manual text.
- [ ] `docs/help.md` has no Markdown headings, fenced code blocks, Markdown links, or tables.
- [ ] README has a minimal run example.
- [ ] README has an output directory example.
- [ ] README has troubleshooting hints.
- [ ] README lists main dependency apps and versions.
- [ ] If the flow exposes `@step:` passthrough blocks, `docs/help.md` lists all
      supported slots and their call sites in plain terminal text.
- [ ] If the flow exposes `@step:` passthrough blocks, README explains that
      slots default to empty, are optional expert escape hatches, affect only
      the named call site when explicitly supplied, and includes a slot table,
      example, and links to the flow guide/checklist.
- [ ] The documented `@step:` slots match the source `::(@:)<step-name>::`
      blocks exactly; no public slot is hidden in source only.
- [ ] Documentation avoids unsupported scientific claims.

## 11. Metadata and Hub

- [ ] `taffish.toml` name, version, release, description, license, homepage, and source fields are accurate.
- [ ] Keywords cover domain, data type, and main task.
- [ ] Dependency declarations match README.
- [ ] Dockerfile has no unrelated large build residue.
- [ ] Image platform declaration is accurate.
- [ ] Publishing will not overwrite existing Git tags, GitHub Releases, image tags, or Hub index records.
- [ ] If `taf build`, local wrapper tests, or any validation generated target artifacts, generated wrapper files and `.taf-*` frozen directories under `target/` have been cleaned before publication or handoff, leaving only intended placeholders such as `target/.gitkeep`.

## 12. Hard blockers

Do not publish if any of these are true:

- [ ] No explicit `--outdir`.
- [ ] Unexpected files are written into the current directory or input directories.
- [ ] Core bioinformatics commands depend on the host environment.
- [ ] `[[taf: ...]]` exists only as a dummy declaration at the top, while real
      steps do not show direct taf calls.
- [ ] Main analytical `[[taf: ...]]` calls do not expose call-site `@:` step
      parameter blocks and the documentation does not explain a reasonable
      exception such as a version probe, dependency anchor, or lightweight
      check.
- [ ] Dependency versions are not fixed.
- [ ] Normal execution must download databases, models, or references.
- [ ] Large databases or reference genomes are embedded in the flow image.
- [ ] `commands.sh`, `versions.tsv`, or `run.manifest.json` is missing.
- [ ] Smoke does not run a real data path.
- [ ] Smoke hard-codes the Docker backend or downloads dependency images.
- [ ] The source tree prepared for publication still contains temporary `target/` wrappers, `.taf-*` frozen directories, or generated test files.
- [ ] Documentation does not explain inputs, outputs, resources, or limits.
- [ ] Result filenames or output layout are unstable enough that users cannot reuse them.

## 13. Final review template

Use this format before release:

```text
Flow: taf-example-flow
Version: 0.1.0-r1
Conclusion: publishable / not publishable yet

Entrypoint:
- ...

Dependencies:
- ...

Output directory:
- ...

Inputs and outputs:
- ...

Provenance:
- ...

Smoke:
- ...

Documentation:
- ...

Resources and limits:
- ...

Blockers:
- ...

Follow-ups:
- ...
```

The report can be short, but it must let maintainers judge whether the flow is transparent, reproducible, and portable.
