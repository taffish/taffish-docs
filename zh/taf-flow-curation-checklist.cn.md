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
- [ ] 每个会影响分析结果、资源占用或主要报告内容的真实 `[[taf: ...]]` call site，
      都有唯一、语义清楚的 `(@:)<step-name>` 参数块，并在调用点使用
      `::(@:)<step-name>::` 作为高级参数入口；版本探针、dependency anchor 和
      极轻量存在性检查已有明确例外理由。
- [ ] 同一个 taf app 在同一 flow 中多次调用时，`@:` block 按 call site 拆分命名；
      没有用一个宽泛的 `extra-step` 或 app 级 block 同时控制多个语义不同的步骤。
- [ ] 如果当前 flow 调用的是 taf-subflow，父 flow 优先桥接 subflow 内部已公开的
      tool step blocks，例如
      `@child-tool-step: ::(@:)parent-child-tool-step:: @:`；没有只用一个外层
      `@<subflow>-step:` 代替内部 tool 参数开放。
- [ ] 父 flow 桥接 subflow 内部 step blocks 时，父级槽位名称带有父流程步骤、
      subflow 名称或其他清晰命名空间，避免多个 subflow 中同名 tool step 混淆。
- [ ] 不使用任何 `@step:` 参数时，flow 默认路径仍能完整运行并产出标准结果、报告和
      provenance；`@:` 只是专家扩展入口，不是正常分析必需参数。
- [ ] 每个 `::(@:)<step-name>::` 默认展开为空；源码、模板或 helper script 没有为其
      设置非空 default，也没有用变量预填内容。flow 固有默认工具参数写在命令本体中，
      只有用户显式传入 `@step-name: ... @:` 时才进入该槽位。
- [ ] 用户实际传入的 step 参数块内容会进入 `commands.sh`、`run.manifest.json` 或
      等价 provenance，便于审计和复现。
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
- [ ] 如果生成最终 HTML 报告，HTML 尽量是 self-contained / standalone：主要
      CSS/JS/logo/PNG/SVG 图/关键摘要表格/解释文本已内嵌，单独复制该 HTML 后仍能
      离线阅读主要结果。
- [ ] HTML 报告中的主图 `<img>` 均为非空 `data:image/...;base64,...`，没有
      `data:image/...;base64,` 这类空 payload；`taffish-hub` 维护者工作区中的
      shell renderer 使用模板提供的
      `repos/apps/templates/flow-report/scripts/report_helpers.sh` 生成 data URI，避免
      macOS/BSD 与 GNU `base64` 行为差异。
- [ ] 如果生成正式 HTML 报告，报告结构基于 TAFFISH 共享 flow-report 模板契约；
      没有为当前 flow 临时自造一套不兼容的报告外壳、目录、语言切换或交付结构。
- [ ] HTML 报告保留模板硬外壳：`data-template="taffish-flow-report"`、
      `.report-shell`、`.report-sidebar`、`.report-main`、`.brand-link`、
      `.language-switch`、`.section-nav`、`.sidebar-external`、`.hero`、`.section`
      和 `.report-footer` 存在且语义正确；真实 TAFFISH logo 已内嵌。
- [ ] 左侧目录顺序与正文顺序一致，长报告子目录只在当前大章节下展开，目录高亮不
      反向跳回旧父章节；子报告按钮按内容宽度显示，长路径/表格名在卡片内换行，
      `WARN`/`FAIL` 等状态词不被拆行。
- [ ] 已运行 `repos/apps/templates/flow-report/scripts/check-rendered-report.py` 对 HTML 做
      模板外壳结构检查；生产报告在真实 logo 可用时使用 `--require-logo-data-uri`。
- [ ] HTML 报告中的外部文件链接只是增强入口；完整矩阵、PDF、日志、MultiQC、
      commands、manifest 等仍保留在结果目录用于审计和二次分析，但不应成为阅读
      主要报告内容的必需依赖。
- [ ] 如果 HTML 报告嵌入 MultiQC、FastQC、Qualimap 等 QC 子报告，嵌入的是工具真实
      生成的原始 HTML bundle，而不是手写仿制页面；原始 bundle 仍保留在结果目录用于审计。
- [ ] 嵌入的 HTML/QC 子报告从单个主 HTML 中打开时，默认在独立本地页面运行；对 MultiQC/
      Plotly 等重型 JavaScript 报告，不默认使用顶层 `blob:`、iframe 或 `srcdoc`，以避免
      图表初始化异常或空白模块。
- [ ] 子报告嵌入有 provenance：记录 source path、source bytes、embedded bytes、status
      和 unresolved resources；主 HTML 中的 payload JSON 能完整解析，不会被嵌套
      `</script>` 截断。
- [ ] 对本来就是 self-contained single HTML 的 MultiQC/FastQC 报告，嵌入 payload 与
      原始 HTML 字节一致或 hash 一致，除非文档说明了必要转换。
- [ ] 大型交互式 QC 报告优先通过静态 payload/resource 检查和普通浏览器人工抽查验收；
      不把某个自动化浏览器对真实 MultiQC 大页面的点击失败作为唯一判据。
- [ ] 如果报告会作为 GitHub Pages 等公开案例发布，生成的 HTML、manifest、commands、
      provenance 和 embedded report 索引不泄漏 `/home/...`、`/Users/...`、临时目录等
      维护者本地绝对路径；公开示例使用稳定占位符或相对路径。
- [ ] 公开案例中的 generated HTML 通过 `git diff --check` 等空白检查，包括从真实
      QC 工具复制或嵌入的 HTML。
- [ ] 公开案例中的大型 standalone HTML 已做体积审查：超过 GitHub 推荐 50 MB 时原因明确，
      且普通 tracked 文件不接近 GitHub 100 MB 硬限制。
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

- [ ] 新建 flow 文档或大幅重写 README/help 时，已从
      `repos/apps/templates/app-docs/flow/` 起步；如未使用，README 中说明了原因。
- [ ] README 面向用户，解释任务、输入、输出、示例和限制。
- [ ] `docs/help.md` 是纯文本命令手册风格。
- [ ] `docs/help.md` 没有 Markdown 标题、三反引号代码围栏、Markdown 链接或表格。
- [ ] README 中有最小运行示例。
- [ ] README 中有输出目录示例。
- [ ] README 中有常见问题或故障定位提示。
- [ ] README 中列出主要依赖 app 和版本。
- [ ] 如果 flow 暴露 `@step:` passthrough 参数块，`docs/help.md` 用纯文本列出所有
      支持的槽位，说明语法、默认空和高级用途；大量槽位可以按 subflow/模块紧凑分组，
      不必在终端 help 中展开完整长表。
- [ ] 如果 flow 暴露 `@step:` passthrough 参数块，README 说明这些槽位默认展开为空、
      是可选高级逃生口、只有显式传入时才影响对应 call site，并包含槽位表、示例和
      flow 手册/checklist 链接。
- [ ] 文档中的 `@step:` 槽位和源码中的 `::(@:)<step-name>::` 完全一致；没有只藏在
      源码里、用户文档不可发现的公开槽位。
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
- [ ] 主要分析型 `[[taf: ...]]` 调用没有按 call site 暴露 `@:` step 参数块，也没有说明
      这是版本探针、dependency anchor 或轻量检查等合理例外。
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
