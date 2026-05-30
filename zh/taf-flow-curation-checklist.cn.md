# TAFFISH Flow 精修 Checklist

本文用于 taf-flow app 的设计、实现、发布前检查。

它和《TAFFISH Flow 开发者指南》配套使用。开发者指南解释怎么做；本 checklist 用来判断一个 flow 是否已经足够清楚、可靠、可发布。

## 0. 基本结论

发布前先给出一句话结论：

```text
结论：可以发布 / 暂不发布
主要原因：
- ...
Blocker：
- ...
```

只要存在 blocker，就不要 publish。

## 1. 身份与边界

- [ ] flow 名称清楚，能表达任务，而不是只表达某个工具名。
- [ ] `taffish.toml` 中 app 类型、版本、描述和关键词准确。
- [ ] README 说明了这个 flow 解决什么问题。
- [ ] README 说明了这个 flow 不解决什么问题。
- [ ] flow 不是单个 tool app 的重复包装。
- [ ] flow 的步骤在该领域中属于常见或容易解释的实践。
- [ ] 默认参数不会制造过度承诺。

## 2. 依赖与可复现性

- [ ] 所有核心生信命令都来自显式 taf 依赖。
- [ ] `taffish.toml` 中依赖版本是精确版本。
- [ ] `src/main.taf` 中 `[[taf: ...]]` 调用和 `taffish.toml` 依赖一致。
- [ ] `[[taf: ...]]` 写在真实执行该工具的 call site，而不是集中放在开头的
      `if false` / dummy block 中，再通过 `taffish_step_*`、`*_STEP`、
      `TAF_CMD` 或裸 `taf-xxx-v...` wrapper 间接运行。
- [ ] 如果 `[[taf: ...]]` 使用运行时 shell 变量，变量在生成后的 shell 中仍会正确展开；
      没有在 TAFFISH 编译 step 时提前展开为空值或错误值。
- [ ] 使用 `'"$var"'` 的变量已在调用前 `export`；循环中的 taf 调用使用
      `</dev/null`，不会消耗 `while read ... < file` 的输入。
- [ ] 动态长度参数没有通过裸 wrapper、`eval` 或宿主机 PATH 执行；如需 helper script，
      它写在输出目录中，并通过真实 call site 的 `[[taf: ...]]` 执行。
- [ ] 没有依赖宿主机 PATH 中的 `fastqc`、`samtools`、`bwa`、`STAR` 等生信工具。
- [ ] 基础 shell/coreutils 依赖已经在 smoke 中覆盖。
- [ ] 正常运行不需要联网下载核心资源。
- [ ] 没有把大型数据库、参考基因组或模型塞进 flow 镜像。
- [ ] 平台限制、GPU 需求、商业许可或非自由许可已经在文档中说明。

## 3. 输入契约

- [ ] 必填输入参数清楚。
- [ ] 可选参数清楚，并有默认值说明。
- [ ] 样本表格式有示例。
- [ ] 样本表列名、路径解释规则和样本名规则已文档化。
- [ ] 缺失文件、重复样本、空样本名、格式错误会早失败。
- [ ] paired-end / single-end / long-read 等模式边界清楚。
- [ ] 参考文件、索引、数据库参数有格式说明。
- [ ] 文档说明了哪些输入会被读取，哪些输入可能被索引或缓存。

## 4. 输出目录与文件写入

- [ ] flow 要求显式 `--outdir`。
- [ ] 所有新生成内容都写入 `<outdir>/...`。
- [ ] 不污染当前目录。
- [ ] 不修改输入文件或参考文件。
- [ ] 输出目录已存在时默认拒绝运行。
- [ ] 如果支持 `--force`，覆盖行为清楚且不会误删输入。
- [ ] 如果支持 `--resume`，可复用步骤和风险写清楚。
- [ ] 临时文件位于 `<outdir>/02_intermediate/` 或 `<outdir>/.tmp/`。
- [ ] 失败时日志仍保留在 `<outdir>/01_logs/`。

推荐结构检查：

```text
<outdir>/
  00_inputs/
  01_logs/
  02_intermediate/
  03_results/
  04_reports/
  run.manifest.json
```

## 5. 步骤与数据流

- [ ] README 逐步说明数据如何从输入流向输出。
- [ ] 每一步都有输入、工具、输出说明。
- [ ] 主要步骤日志分开保存。
- [ ] 关键中间文件命名稳定。
- [ ] 最终结果和中间结果分目录保存。
- [ ] flow 失败时能定位到具体步骤。
- [ ] 底层工具的关键参数已经记录。

## 6. 结果与报告

- [ ] `03_results/` 中保存机器可读的主要结果。
- [ ] `04_reports/` 中保存面向用户阅读的报告。
- [ ] 有一个项目级或样本级 summary 文件，例如 `flow_summary.tsv`。
- [ ] 报告文件的用途在 README 中说明。
- [ ] HTML、PDF、TSV、JSON 等输出格式符合该领域常见使用方式。
- [ ] 空结果、低质量结果或失败样本有清楚标记。
- [ ] 文档说明哪些输出适合下游分析，哪些只适合人工检查。

## 7. Provenance

- [ ] 输出 `04_reports/commands.sh`。
- [ ] 输出 `04_reports/versions.tsv`。
- [ ] 输出 `04_reports/methods.txt`。
- [ ] 输出 `<outdir>/run.manifest.json`。
- [ ] `commands.sh` 记录关键实际命令和参数。
- [ ] `versions.tsv` 记录 flow 和主要依赖工具版本。
- [ ] `run.manifest.json` 记录输入、参数、依赖、输出和时间。
- [ ] provenance 文件不包含 token、密码或其他敏感信息。

## 8. 资源需求

- [ ] README 写明最小资源。
- [ ] README 写明推荐资源。
- [ ] 文档说明磁盘需求和输入规模的关系。
- [ ] 文档说明 CPU / thread 参数如何影响运行。
- [ ] 文档说明哪些步骤最耗内存。
- [ ] 数据库或索引大小已经单独说明。
- [ ] 在 arm64 主机运行 amd64-only 依赖时，文档说明了模拟运行的影响。

## 9. Smoke 与验证

- [ ] smoke 不是只跑 `--help`。
- [ ] smoke 检查 flow 命令存在。
- [ ] smoke 检查 flow 输出的版本记录/provenance，而不是直接用底层依赖版本命令替代 flow 验证。
- [ ] smoke/formal 先 `taf build`，并以本 flow 的 `target/taf-<flow>-v...` 为核心被测对象。
- [ ] smoke/formal 的通过条件来自 flow wrapper 生成的输出、日志和 provenance。
- [ ] 下游 smoke/formal 需要上游产物时，优先调用上游 subflow 的 target wrapper 生成输入。
- [ ] 测试脚本中直接调用 dependency wrapper 只用于没有合适上游 subflow 可用的 fixture 准备或环境探针，且不替代 flow wrapper 主路径验证。
- [ ] smoke 使用极小 fixture 跑通真实数据路径。
- [ ] smoke 检查关键结果文件存在。
- [ ] smoke 检查 `commands.sh`、`versions.tsv`、`run.manifest.json` 存在。
- [ ] smoke 检查当前目录没有非预期新文件。
- [ ] smoke 运行时间和数据大小适合 CI 或本地快速检查。
- [ ] 本地维护 smoke 尊重已有 `TAFFISH_CONTAINER_BACKEND`，未设置时默认使用 `podman`。
- [ ] smoke 没有写死 Docker backend。
- [ ] smoke 不主动 `docker pull`、`podman pull` 或重新下载依赖镜像。
- [ ] 依赖 app 镜像缺失时，smoke 给出清楚错误和准备方式。
- [ ] 文档说明 smoke 不等同于科学正确性验证。

## 10. 文档

- [ ] README 面向用户，解释任务、输入、输出、示例和限制。
- [ ] `docs/help.md` 是纯文本命令手册风格。
- [ ] `docs/help.md` 没有 Markdown 标题、三反引号代码围栏、Markdown 链接或表格。
- [ ] README 中有最小运行示例。
- [ ] README 中有输出目录示例。
- [ ] README 中有常见问题或故障定位提示。
- [ ] README 中列出主要依赖 app 和版本。
- [ ] 文档中没有承诺 flow 无法保证的科学结论。

## 11. Metadata 与 Hub

- [ ] `taffish.toml` 名称、版本、release、描述、license、homepage/source 字段准确。
- [ ] app keywords 覆盖领域、数据类型和主要任务。
- [ ] 依赖声明和 README 中描述一致。
- [ ] Dockerfile 没有无关的大型构建残留。
- [ ] image 平台声明准确。
- [ ] 发布前确认 Git tag、Release、image tag 和 Hub index record 不会覆盖历史。
- [ ] 如果中间执行过 `taf build`、本地 wrapper 测试或会生成 target artifact 的验证，发布或交还维护者前已经清理 `target/` 下生成的 wrapper 文件和 `.taf-*` 冻结目录，只保留 `target/.gitkeep` 等应有占位文件。

## 12. Hard blockers

以下任一问题存在时，不应发布：

- [ ] 没有显式 `--outdir`。
- [ ] 会在当前目录或输入目录写入非预期文件。
- [ ] 核心生信命令依赖宿主机环境。
- [ ] `[[taf: ...]]` 只作为开头 dummy 声明存在，真实步骤看不到直接 taf 调用。
- [ ] 依赖版本没有固定。
- [ ] 正常运行必须联网下载数据库、模型或参考文件。
- [ ] 大型数据库或参考基因组被塞进 flow 镜像。
- [ ] 没有 `commands.sh`、`versions.tsv` 或 `run.manifest.json`。
- [ ] smoke 没有真实数据路径。
- [ ] smoke 写死 Docker backend，或会主动下载依赖镜像。
- [ ] 准备发布的源码树里残留临时 `target/` wrapper、`.taf-*` 冻结目录或测试生成文件。
- [ ] 文档没有解释输入、输出、资源或限制。
- [ ] 结果文件命名或目录结构不稳定，用户难以复用。

## 13. 最终审查报告模板

发布前可以按这个格式给出结论：

```text
Flow: taf-example-flow
Version: 0.1.0-r1
结论：可以发布 / 暂不发布

入口：
- ...

依赖：
- ...

输出目录：
- ...

输入输出：
- ...

Provenance：
- ...

Smoke：
- ...

文档：
- ...

资源与限制：
- ...

Blocker：
- ...

后续建议：
- ...
```

这个报告不需要很长，但必须能让维护者判断：这个 flow 是否真的透明、可复现、可迁移。
