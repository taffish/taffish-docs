# TAFFISH Flow 开发者指南

本文是 TAFFISH flow app 的中文开发手册。

如果你只是想学习 `[[taf: ...]]` 依赖语法，请先读《Flow 与依赖指南》。本文关注的是一个 flow app 应该如何设计、实现、验证和发布。

## 1. Flow 是什么

TAFFISH 的根是 command-level reproducibility。taf-tool 负责把一个上游工具封装成可复现命令；taf-flow 负责把多个已经封装好的命令串成一个稳定、透明、可审计的数据路径。

taf-flow 适合做：

- 常见生信任务的一键式起步流程。
- 教学、培训、方法复现和标准 QC。
- 轻量生产流程，尤其是步骤固定、输入输出清楚的流程。
- 一个领域内常用工具组合的参考实现。

taf-flow 不适合假装自己是完整 workflow engine：

- 不追求复杂 DAG 调度、云批量执行、任务级缓存或动态模块替换。
- 不把大量未封装工具藏在一个 flow 镜像里。
- 不把大型数据库、参考基因组或易变在线资源塞进镜像。
- 不承诺替代 Nextflow、Snakemake、Cromwell 等成熟 workflow system。

一个好的 taf-flow 应该让用户清楚知道：输入是什么，做了哪些命令，输出在哪里，版本是什么，失败时看哪里，结果能解释到什么程度。

## 2. 什么时候应该做 flow

建议创建 flow app 的情况：

- 至少包含两个有明确数据衔接关系的 taf-tool。
- 目标问题在生信实践中常见，例如测序质控、比对后 QC、RNA-seq 表达定量、组装质量评估。
- 输入格式和输出结构可以稳定定义。
- 依赖工具已经在 Hub 中存在，或者可以先补齐。
- 可以用小 fixture 跑通一条真实路径。

不建议创建 flow app 的情况：

- 只是给单个工具换一个命令名。
- 需要大量人工交互或图形界面操作。
- 需要实时联网下载关键资源。
- 需要频繁替换核心模块，而这些模块又没有稳定接口。
- 领域实践还没有形成清楚共识，默认流程容易误导用户。

## 3. 标准目录结构

flow app 仍然使用普通 taf-app 目录结构：

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

`src/main.taf` 是 flow 的主入口。flow 通常不需要 tool app 的自动命令模式，而是提供稳定的领域接口，例如：

```text
taf-ngs-qc-flow --reads1 R1.fq.gz --reads2 R2.fq.gz --outdir qc-out --threads 4
```

`docs/help.md` 会被 `taf-xxx --help` 原样输出到终端，必须写成纯文本命令手册风格。不要使用 Markdown 标题、三反引号代码围栏、Markdown 链接或表格。

## 4. 输出目录规则

官方 flow app 必须使用单一输出目录模型：

```text
<outdir>/
  00_inputs/
  01_logs/
  02_intermediate/
  03_results/
  04_reports/
  run.manifest.json
```

所有新生成内容都必须放在 `<outdir>/...` 下面。flow 不应该在当前目录散落日志、临时文件、索引、报告或结果。

推荐规则：

- `--outdir` 应该是必填参数。
- 如果 `<outdir>` 已存在，默认应该拒绝运行。
- 如果支持覆盖，必须使用明确的 `--force`。
- 如果支持续跑，必须使用明确的 `--resume`，并在文档中说明哪些步骤可以复用。
- 临时文件也应该放在 `<outdir>/02_intermediate/` 或 `<outdir>/.tmp/`。
- 读取输入文件和参考文件可以在 `<outdir>` 外，但不能修改它们。

推荐输出结构：

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

具体 flow 可以调整子目录名字，但应保持层次清楚：输入快照、日志、中间结果、最终结果、报告和 provenance 分开。

## 5. 输入设计

flow 可以支持两种输入模型。

简单模式适合单样本或少量参数：

```text
taf-ngs-qc-flow --reads1 sample_R1.fq.gz --reads2 sample_R2.fq.gz --outdir qc-out
```

样本表模式适合多样本：

```text
taf-rnaseq-expression-flow --samples samples.tsv --genome-index ref/index --gtf genes.gtf --outdir rnaseq-out
```

样本表建议使用 TSV，并保持字段稳定。例如 paired-end FASTQ：

```text
sample_id	read1	read2	condition
S1	data/S1_R1.fq.gz	data/S1_R2.fq.gz	control
S2	data/S2_R1.fq.gz	data/S2_R2.fq.gz	treatment
```

输入检查至少应覆盖：

- 必填参数是否存在。
- 文件是否存在且可读。
- 样本名是否为空、重复或包含危险字符。
- paired-end 文件是否成对。
- 样本表列名是否符合文档。
- 相对路径如何解释。

推荐规则：样本表中的相对路径按样本表所在目录解释，而不是按运行目录解释。这样用户移动项目目录时更不容易出错。

## 6. 参考文件和数据库

参考基因组、注释文件、索引、数据库和模型通常不应该打进 flow 镜像。它们应该通过参数传入。

常见参数形式：

```text
--reference genome.fa
--gtf genes.gtf
--index star-index/
--database kraken2-db/
```

文档必须说明：

- 每个参考资源的来源和格式要求。
- 是否需要预先建索引。
- 索引版本是否需要和工具版本匹配。
- 资源大小对磁盘、内存和运行时间的影响。
- flow 是否会读取、创建或复用索引。

正常运行不应联网下载关键资源。确实需要在线资源时，应该作为独立准备步骤或独立 data app 设计，而不是隐藏在 flow 的主路径中。

## 7. 依赖声明

flow app 必须显式声明依赖的 taf-tool。不要依赖宿主机上的生信命令。

`taffish.toml` 中应该固定依赖版本：

```toml
[dependencies]
"taf-fastqc" = "0.12.1-r1"
"taf-multiqc" = "1.27.1-r1"
"taf-fastp" = "0.24.1-r1"
```

`src/main.taf` 中通过 `[[taf: ...]]` 调用依赖：

```taf
[[taf: taf-fastqc-v0.12.1-r1 fastqc --version]]
[[taf: taf-fastp-v0.24.1-r1 fastp --version]]
[[taf: taf-multiqc-v1.27.1-r1 multiqc --version]]
```

更重要的是，`[[taf: ...]]` 应写在实际执行位置，而不是集中放在文件开头做“声明”。实现 flow 时可以先像写普通 shell 一样写出控制流：

```sh
samtools flagstat "$bam" > "$flagstat"
```

然后把核心生信命令替换为 taf 调用：

```taf
[[taf: taf-samtools-v1.23.1-r1 samtools flagstat '"$bam"' ]] > "$flagstat"
```

不要把所有 `[[taf: ...]]` 放入 `if false`、dummy block 或单独“依赖声明区”，再通过
`taffish_step_*`、`*_STEP`、`TAF_CMD` 或裸 `taf-xxx-v...` wrapper 间接执行。这样的源码
不利于审计，也容易让真实数据路径和依赖调用脱节。`commands.sh` 可以记录用户可读的命令文本，
但实际执行路径必须在 `src/main.taf` 的 call site 中看到 `[[taf: ...]]`。

flow 可以使用少量 shell 基础工具整理文件和表格，例如 `mkdir`、`cp`、`printf`、`awk`、`sed`、`sort`。但核心生信分析命令必须来自显式 taf 依赖。

## 8. 参数设计

推荐把参数分成三层。

必填参数：

- 输入文件或样本表。
- 参考文件、数据库或索引。
- 输出目录。

常用参数：

- `--threads`
- `--min-quality`
- `--keep-intermediate`
- `--force`
- `--resume`

高级参数：

- `--extra-aligner-args`
- `--extra-qc-args`
- `--extra-counter-args`
- `--dry-run`

高级参数应该谨慎暴露。它们可以帮助熟悉底层工具的用户，但也容易破坏 flow 的稳定假设。所有 extra 参数都必须写入 `commands.sh` 和 `run.manifest.json`。

## 9. 数据如何流动

flow 文档应把每一步的数据流说清楚。推荐用以下格式描述：

```text
Step 1. 输入检查
输入：用户参数、样本表、参考文件
输出：00_inputs/input_files.tsv、01_logs/flow.log

Step 2. 原始 reads 质控
输入：FASTQ
工具：taf-fastqc
输出：03_results/fastqc/

Step 3. 报告汇总
输入：03_results/fastqc/
工具：taf-multiqc
输出：04_reports/multiqc_report.html
```

实现时也建议按 step 写日志：

```text
01_logs/steps/01_validate_inputs.log
01_logs/steps/02_fastqc.log
01_logs/steps/03_multiqc.log
```

这样用户看到失败时，可以直接定位到具体步骤。

## 10. Provenance 输出

每个正式 flow 都应该输出以下 provenance 文件。

`commands.sh` 记录实际执行过的关键命令：

```text
04_reports/commands.sh
```

要求：

- 命令顺序和 flow 步骤一致。
- 包含实际参数。
- 不写入 token、密码等敏感信息。
- 适合作为审计记录，不要求可以在任意环境完全原样复跑。

`versions.tsv` 记录依赖版本：

```text
tool	version	source
taf-fastqc	0.12.1-r1	taffish dependency
taf-multiqc	1.27.1-r1	taffish dependency
```

`run.manifest.json` 记录本次运行的结构化信息：

```json
{
  "flow": "ngs-qc-flow",
  "flow_version": "0.1.0-r1",
  "started_at": "2026-05-20T10:00:00+08:00",
  "outdir": "qc-out",
  "inputs": [],
  "parameters": {},
  "dependencies": [],
  "outputs": []
}
```

`methods.txt` 给用户提供可复制到实验记录或方法部分的简短描述。它不是论文方法学的替代品，但应该讲清楚主要工具、版本、关键参数和结果目录。

`flow_summary.tsv` 汇总最关键的样本级或项目级结果，方便快速检查。

## 11. 错误处理

flow 应该早失败、清楚失败。

推荐检查：

- 必填参数缺失时，显示简短 usage。
- 输入文件不存在时，指出具体路径。
- 输出目录已存在时，默认拒绝。
- 样本表格式错误时，指出行号和列名。
- 依赖命令失败时，指出步骤名和日志路径。
- 关键输出文件为空或不存在时，标记为失败。

不推荐：

- 忽略失败继续生成看似正常的报告。
- 自动覆盖已有结果。
- 在错误信息里只输出底层工具的一大段原始堆栈。
- 把临时文件留在当前目录。

## 12. 资源需求

README 和 help 中应给出资源建议。

建议写法：

```text
Resources:
  Small test: 2 CPU, 4 GB RAM, 2 GB disk
  Typical run: 4-8 CPU, 8-32 GB RAM
  Disk: at least 2-5x input FASTQ size, depending on intermediate retention
```

资源说明应该结合数据规模：

- FASTQ 流程按 reads 数量和压缩文件大小说明。
- 比对流程按基因组大小、索引大小和样本数说明。
- RNA-seq 流程按样本数、测序深度和参考索引说明。
- metagenomics 流程按数据库大小说明。
- assembly 流程按 reads 深度和基因组大小说明。

如果某一步可能成为瓶颈，要直接写出来。

## 13. Smoke 与测试数据

flow 的 smoke 不只是跑 `--help`。

至少应该检查：

- flow 命令可以启动。
- 依赖命令可以被调用并输出版本。
- 极小 fixture 可以跑通一条真实路径。
- 关键输出文件存在。
- `commands.sh`、`versions.tsv`、`run.manifest.json` 存在。
- 运行结束后没有在当前目录留下非预期文件。

fixture 应该尽量小，目标是验证封装、依赖、路径和最小功能，不替代科学正确性验证。

本地维护 flow app 时，推荐让 smoke 默认使用 Podman backend：

```sh
export TAFFISH_CONTAINER_BACKEND="${TAFFISH_CONTAINER_BACKEND:-podman}"
```

这样可以兼容两种情况：

- 维护者没有设置 backend 时，flow smoke 默认走 Podman。
- 用户或 CI 已经设置 `TAFFISH_CONTAINER_BACKEND` 时，smoke 尊重外部设置。

官方 Hub 的 tool app 构建和生成测试可以继续使用 Docker；但 flow app 通常是在已有 taf-tool 和本地镜像基础上做组合验证，不应该在 smoke 中主动 `docker pull`、`podman pull` 或重新下载依赖镜像。依赖镜像缺失时，应清楚报错，提示用户先准备依赖 app 镜像或显式切换 backend，而不是静默下载。

## 14. 文档要求

每个 flow app 至少需要：

- README：面向网页和仓库浏览。
- `docs/help.md`：面向命令行终端。
- 输入示例：单样本或样本表。
- 输出目录说明：关键结果文件逐项解释。
- 步骤说明：每一步输入、工具、输出。
- 资源需求：最小和推荐资源。
- 限制说明：哪些数据、物种、场景不适用。
- smoke 说明：测试数据覆盖了什么，不覆盖什么。

README 可以使用 Markdown；`docs/help.md` 必须保持纯文本手册风格。

## 15. 版本与发布

flow 的 release 应该随行为变化推进。

通常需要推进 release 的情况：

- 依赖工具版本变化。
- 默认参数变化。
- 输出目录结构变化。
- 新增或移除主要步骤。
- 修改输入样本表格式。
- 修复会影响结果的 bug。

已经发布的 Git tag、GitHub Release、image tag 和 Hub index record 都应视为不可变。修复应推进新 release，不覆盖历史。

## 16. 推荐优先开发的 flow 类型

在基础 taf-tool 足够扎实后，可以优先考虑以下 flow。

`taf-ngs-qc-flow`：

- 处理组学：通用 NGS / WGS / WES / RNA-seq 前置 QC。
- 输入：FASTQ 或样本表。
- 输出：FastQC 结果、可选 trimming 结果、MultiQC 报告、样本级摘要。
- 价值：最通用，教学和生产前检查都常用。

`taf-bam-qc-flow`：

- 处理组学：比对后的 DNA/RNA 数据。
- 输入：BAM/CRAM、参考基因组、可选 BED。
- 输出：flagstat、idxstats、coverage、duplication、MultiQC 报告。
- 价值：连接上游比对和下游变异/表达分析。

`taf-rnaseq-expression-flow`：

- 处理组学：RNA-seq 表达定量。
- 输入：FASTQ 样本表、参考索引、GTF。
- 输出：QC、比对或伪比对结果、gene/transcript count matrix、MultiQC 报告。
- 价值：用户需求高，但要清楚说明它是 lite/reference flow，不替代复杂差异分析项目。

`taf-assembly-qc-flow`：

- 处理组学：基因组或宏基因组组装结果 QC。
- 输入：assembly FASTA、可选 reads、可选 lineage/database。
- 输出：contig 统计、N50、BUSCO/QUAST 类报告、summary。
- 价值：组装后检查常见，但数据库和资源需求要写清楚。

## 17. 常见反模式

避免以下做法：

- flow 运行后在当前目录生成一堆 `tmp/`、`logs/`、`multiqc_data/`。
- 用宿主机上的 `samtools` 或 `fastqc`，但文档里说依赖已封装。
- 默认联网下载参考库。
- 把一个 50 GB 数据库打进镜像。
- smoke 只检查 `--help`。
- 输出报告很多，但没有一个清楚的 `flow_summary.tsv`。
- 文档只列命令，不解释每一步输出有什么用。
- 默认参数过于激进，却没有说明适用范围。

好的 flow 不一定复杂。它应该是透明、可审计、能跑通、能解释、能被用户放心迁移的。
