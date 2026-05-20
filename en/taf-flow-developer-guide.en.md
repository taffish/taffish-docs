# TAFFISH Flow Developer Guide

This guide describes how to design, implement, validate, and publish TAFFISH flow apps.

If you only need the `[[taf: ...]]` dependency syntax, start with the Flow And Dependencies Guide. This document focuses on engineering conventions for official flow apps.

## 1. What a flow is

TAFFISH is rooted in command-level reproducibility. A taf-tool packages one upstream command as a reproducible executable. A taf-flow connects several already-packaged commands into a stable, transparent, auditable data path.

taf-flow apps are good for:

- Common bioinformatics starter workflows.
- Teaching, training, method reproduction, and standard QC.
- Lightweight production runs when the steps and outputs are stable.
- Reference implementations for common tool combinations in one domain.

taf-flow apps should not pretend to be full workflow engines:

- They do not aim to replace complex DAG scheduling, cloud batch execution, task-level caching, or dynamic module substitution.
- They should not hide many unpackaged tools inside one flow image.
- They should not embed large databases, reference genomes, or volatile online resources in the image.
- They should not claim to replace Nextflow, Snakemake, Cromwell, or similar workflow systems.

A good taf-flow makes the input contract, commands, versions, output directory, logs, and interpretation boundaries obvious.

## 2. When to create a flow

Create a flow app when:

- It contains at least two taf-tools with a real data handoff.
- The target task is common in bioinformatics practice, such as sequencing QC, post-alignment QC, RNA-seq expression quantification, or assembly QC.
- The input formats and output layout can be defined stably.
- Dependencies already exist in Hub, or can be added first.
- A tiny fixture can run through a real path.

Avoid creating a flow app when:

- It is only a renamed single-tool wrapper.
- It requires heavy manual interaction or GUI operation.
- It downloads critical resources online during normal execution.
- It needs frequent core module swaps without a stable interface.
- The field has no clear practice yet and a default flow would mislead users.

## 3. Standard project layout

A flow app uses the normal taf-app layout:

```text
taf-example-flow/
  taffish.toml
  Dockerfile
  src/
    main.taf
  docs/
    help.md
  README.md
  smoke.sh
  testdata/
```

`src/main.taf` is the flow entrypoint. Flow apps usually expose a stable domain interface instead of tool-app command mode:

```text
taf-ngs-qc-flow --reads1 R1.fq.gz --reads2 R2.fq.gz --outdir qc-out --threads 4
```

`docs/help.md` is printed directly by `taf-xxx --help`, so it must be plain terminal manual text. Do not use Markdown headings, fenced code blocks, Markdown links, or tables there.

## 4. Output directory rule

Official flow apps must use one explicit output directory:

```text
<outdir>/
  00_inputs/
  01_logs/
  02_intermediate/
  03_results/
  04_reports/
  run.manifest.json
```

All generated files must live under `<outdir>/...`. A flow should not scatter logs, temporary files, indexes, reports, or results in the current directory.

Recommended rules:

- `--outdir` should be required.
- If `<outdir>` already exists, refuse by default.
- If overwrite is supported, require an explicit `--force`.
- If resume is supported, require an explicit `--resume` and document which steps can be reused.
- Temporary files should live under `<outdir>/02_intermediate/` or `<outdir>/.tmp/`.
- Inputs and references may be read outside `<outdir>`, but must not be modified.

Recommended output layout:

```text
<outdir>/
  00_inputs/
    samples.tsv
    input_files.tsv
  01_logs/
    flow.log
    steps/
      01_fastqc.log
      02_trim.log
      03_multiqc.log
  02_intermediate/
    trimmed/
  03_results/
    fastqc/
    alignments/
    counts/
  04_reports/
    multiqc_report.html
    flow_summary.tsv
    methods.txt
    versions.tsv
    commands.sh
  run.manifest.json
```

Concrete flows may adjust directory names, but should keep input snapshots, logs, intermediate files, final results, reports, and provenance separate.

## 5. Input design

Flows may support a simple mode:

```text
taf-ngs-qc-flow --reads1 sample_R1.fq.gz --reads2 sample_R2.fq.gz --outdir qc-out
```

They may also support a sample-sheet mode:

```text
taf-rnaseq-expression-flow --samples samples.tsv --genome-index ref/index --gtf genes.gtf --outdir rnaseq-out
```

Sample sheets should be stable TSV files, for example:

```text
sample_id	read1	read2	condition
S1	data/S1_R1.fq.gz	data/S1_R2.fq.gz	control
S2	data/S2_R1.fq.gz	data/S2_R2.fq.gz	treatment
```

Validate at least:

- Required parameters.
- File existence and readability.
- Empty, duplicate, or unsafe sample names.
- Paired-end consistency.
- Expected sample-sheet columns.
- Relative path semantics.

Recommended rule: relative paths inside a sample sheet are resolved relative to the sample sheet location, not the process working directory.

## 6. References and databases

Reference genomes, annotations, indexes, databases, and models usually should not be baked into the flow image. Pass them as parameters:

```text
--reference genome.fa
--gtf genes.gtf
--index star-index/
--database kraken2-db/
```

Document:

- Required format and source expectations.
- Whether indexes must be prepared in advance.
- Whether indexes must match tool versions.
- How resource size affects disk, memory, and runtime.
- Whether the flow reads, creates, or reuses indexes.

Normal execution should not download critical resources from the network. Online preparation should be a separate step or a separate data app / data bundle.

## 7. Dependency declarations

Flow apps must declare taf-tool dependencies explicitly. Do not depend on bioinformatics commands from the host PATH.

Use exact versions in `taffish.toml`:

```toml
[dependencies]
"taf-fastqc" = "0.12.1-r1"
"taf-multiqc" = "1.27.1-r1"
"taf-fastp" = "0.24.1-r1"
```

Call dependencies from `src/main.taf`:

```taf
@: [[taf: taf-fastqc-v0.12.1-r1]] fastqc --version
@: [[taf: taf-fastp-v0.24.1-r1]] fastp --version
@: [[taf: taf-multiqc-v1.27.1-r1]] multiqc --version
```

Small shell/coreutils usage is acceptable for file and table handling. Core bioinformatics commands must come from explicit TAFFISH dependencies.

## 8. Parameter design

Group parameters into three layers:

- Required: input files or sample sheet, references/databases/indexes, output directory.
- Common: `--threads`, quality thresholds, `--keep-intermediate`, `--force`, `--resume`.
- Advanced: `--extra-aligner-args`, `--extra-qc-args`, `--extra-counter-args`, `--dry-run`.

Advanced parameters should not be required for normal use. All extra parameters must be recorded in `commands.sh` and `run.manifest.json`.

## 9. Data flow

Flow documentation should explain each step in input-tool-output form:

```text
Step 1. Input validation
Input: user parameters, sample sheet, reference files
Output: 00_inputs/input_files.tsv, 01_logs/flow.log

Step 2. Raw read QC
Input: FASTQ
Tool: taf-fastqc
Output: 03_results/fastqc/

Step 3. Report aggregation
Input: 03_results/fastqc/
Tool: taf-multiqc
Output: 04_reports/multiqc_report.html
```

Keep step logs separate, for example:

```text
01_logs/steps/01_validate_inputs.log
01_logs/steps/02_fastqc.log
01_logs/steps/03_multiqc.log
```

## 10. Provenance outputs

Every formal flow should produce:

- `04_reports/commands.sh`
- `04_reports/versions.tsv`
- `04_reports/methods.txt`
- `04_reports/flow_summary.tsv`
- `<outdir>/run.manifest.json`

`commands.sh` records key commands and actual parameters, without secrets. `versions.tsv` records the flow and major dependencies. `run.manifest.json` records structured inputs, parameters, dependencies, outputs, and timestamps. `methods.txt` gives a concise method description suitable for lab notes. `flow_summary.tsv` provides a fast project-level or sample-level summary.

## 11. Error handling

Flows should fail early and clearly.

Recommended checks:

- Missing required parameters show short usage.
- Missing inputs identify the exact path.
- Existing output directory is rejected by default.
- Sample-sheet errors identify line and column when possible.
- Dependency failures identify step name and log path.
- Missing or empty key outputs are treated as failure.

Avoid silently continuing after a failed step, auto-overwriting prior results, dumping only a long upstream stack trace, or leaving temporary files in the current directory.

## 12. Resource requirements

README and help should state minimum and recommended resources:

```text
Resources:
  Small test: 2 CPU, 4 GB RAM, 2 GB disk
  Typical run: 4-8 CPU, 8-32 GB RAM
  Disk: at least 2-5x input FASTQ size, depending on intermediate retention
```

Relate resources to data scale:

- FASTQ flows: read count and compressed file size.
- Alignment flows: genome size, index size, and sample count.
- RNA-seq flows: sample count, sequencing depth, and reference index.
- Metagenomics flows: database size.
- Assembly flows: read depth and genome size.

Call out known bottleneck steps directly.

## 13. Smoke tests and fixtures

Flow smoke tests must do more than `--help`.

They should check:

- The flow command starts.
- Main dependency versions are visible.
- A tiny fixture runs through a real path.
- Key output files exist.
- `commands.sh`, `versions.tsv`, and `run.manifest.json` exist.
- The current directory is not polluted by unexpected files.

Fixtures should be tiny. They validate packaging, dependencies, paths, and minimal function, not full scientific correctness.

For local flow maintenance, smoke tests should default to the Podman backend:

```sh
export TAFFISH_CONTAINER_BACKEND="${TAFFISH_CONTAINER_BACKEND:-podman}"
```

This supports two cases:

- When no backend is set, local flow smoke uses Podman by default.
- When a user or CI has already set `TAFFISH_CONTAINER_BACKEND`, smoke respects that setting.

Tool app builds and generated tests may continue to use Docker. Flow apps, however, usually compose existing taf-tools and locally prepared images, so smoke tests should not call `docker pull`, `podman pull`, or otherwise re-download dependency images. If a dependency image is missing, fail clearly and tell the user to prepare dependency app images or choose another backend explicitly.

## 14. Documentation requirements

Each flow app needs at least:

- README for repository and web readers.
- `docs/help.md` for terminal use.
- Input examples.
- Output directory explanation.
- Step-by-step data flow.
- Minimum and recommended resources.
- Limitations and non-goals.
- Smoke fixture scope.

README may use Markdown. `docs/help.md` must stay plain terminal manual text.

## 15. Versioning and publication

Advance the release when behavior changes, including:

- Dependency version changes.
- Default parameter changes.
- Output layout changes.
- Major step additions or removals.
- Sample-sheet format changes.
- Result-affecting bug fixes.

Published Git tags, GitHub Releases, image tags, and Hub index records should be treated as immutable. Fixes should advance the release instead of overwriting history.

## 16. Recommended first flow types

`taf-ngs-qc-flow`:

- Omics: general NGS / WGS / WES / RNA-seq pre-QC.
- Input: FASTQ or sample sheet.
- Output: FastQC results, optional trimming results, MultiQC report, sample summary.
- Value: highly reusable for teaching and pre-production checks.

`taf-bam-qc-flow`:

- Omics: aligned DNA/RNA data.
- Input: BAM/CRAM, reference genome, optional BED.
- Output: flagstat, idxstats, coverage, duplication, MultiQC report.
- Value: bridges alignment and downstream variant/expression workflows.

`taf-rnaseq-expression-flow`:

- Omics: RNA-seq expression quantification.
- Input: FASTQ sample sheet, reference index, GTF.
- Output: QC, alignment or pseudoalignment results, count matrices, MultiQC report.
- Value: high demand, but should be documented as a lite/reference flow rather than a full differential-expression project.

`taf-assembly-qc-flow`:

- Omics: genome or metagenome assembly QC.
- Input: assembly FASTA, optional reads, optional lineage/database.
- Output: contig statistics, N50, QUAST/BUSCO-style reports, summary.
- Value: common post-assembly check, with database and resource caveats.

## 17. Common anti-patterns

Avoid:

- Creating `tmp/`, `logs/`, or `multiqc_data/` in the current directory.
- Using host `samtools` or `fastqc` while claiming dependencies are packaged.
- Downloading references by default.
- Baking a 50 GB database into the image.
- Smoke tests that only check `--help`.
- Many reports but no clear `flow_summary.tsv`.
- Documentation that lists commands without explaining output value.
- Aggressive defaults without a clear scope statement.

A good flow does not have to be complex. It should be transparent, auditable, runnable, explainable, and easy to move.
