# Flow 与依赖指南

Flow app 是 TAFFISH 用来组合多个 app 的机制。它通过 `<taffish>` 标签和 `[[taf: ...]]` 语法调用其他 taf app，并通过 `[dependencies]` 声明安装依赖。

## 目录

- [Flow 的角色](#flow-的角色)
- [基本写法](#基本写法)
- [`[[taf: ...]]`](#taf-)
- [`@:` 参数块](#-参数块)
- [依赖声明](#依赖声明)
- [多版本依赖](#多版本依赖)
- [依赖同步](#依赖同步)
- [安装行为](#安装行为)
- [设计建议](#设计建议)
- [常见错误](#常见错误)

## Flow 的角色

Tool app 负责包装一个具体工具。Flow app 负责把多个 tool app 或 flow app 组合成流程。

Flow 适合：

- QC 流程。
- 比对流程。
- 组装流程。
- 注释流程。
- 多步骤生信分析 pipeline。

Flow 不适合：

- 包装单个上游工具。
- 把大量难以维护的 shell 逻辑堆在一起。
- 隐式依赖没有声明的外部命令。

## 基本写法

`src/main.taf`：

```taf
<taffish>
[[taf: taf-fastqc-v0.12.1-r1 fastqc sample.fq]]
[[taf: taf-multiqc-v1.19-r1 multiqc .]]
```

`taffish.toml`：

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

语法：

```text
[[taf: taf-command args...]]
```

推荐使用 exact versioned command：

```taf
[[taf: taf-fastqc-v0.12.1-r1 fastqc sample.fq]]
```

也可以在开发期写基础 command：

```taf
[[taf: taf-fastqc sample.fq]]
```

但正式发布的 flow 推荐固定版本。固定版本可以保证：

- index 能明确记录依赖。
- 用户安装时能安装正确版本。
- 未来上游 app 更新不会悄悄改变 flow 行为。

`[[taf: ...]]` 应写在真正执行该工具的位置。推荐心智模型是：先把 flow 写成普通
shell 脚本，再把核心生信命令那一行替换成 `[[taf: taf-xxx-vA.B.C-rN ...]]`。

例如：

```taf
export bam flagstat log
if [[taf: taf-samtools-v1.23.1-r1 samtools flagstat '"$bam"' ]] </dev/null > "$flagstat" 2> "$log"; then
    :
else
    echo "samtools flagstat failed; see $log" >&2
    exit 1
fi
```

循环中也应该在循环体的实际步骤处调用：

```taf
while IFS="$(printf '\t')" read -r sample_id bam || [ -n "$sample_id" ]; do
    [[taf: taf-rseqc-v5.0.4-r2 bam_stat.py -i '"$bam"' -q '"$mapq"' ]] \
        </dev/null \
        > "$outdir/03_results/rseqc/$sample_id.bam_stat.txt" \
        2>> "$outdir/01_logs/steps/03_rseqc.log"
done < "$outdir/00_inputs/bam_files.tsv"
```

不要把所有 `[[taf: ...]]` 集中放在文件开头的 `if false`、dummy block 或“依赖声明区”中，
再通过 `taffish_step_*`、`*_STEP`、`TAF_CMD` 或裸 `taf-xxx-v...` wrapper 间接运行。
这种写法虽然可能让编译器看到依赖，但会让 flow 源码的真实执行路径不清楚，也容易绕过
依赖审计。

正式 flow 中，除少量 shell/coreutils 通用命令外，任何需要用户额外安装的生信、统计、
绘图、数据库、模型或其他领域工具，都应先作为 taf app 版本化，再通过真实 call site 的
`[[taf: ...]]` 调用。也就是说，flow 源码里应该能直接看到“这个步骤调用哪个 taf app、
哪个版本、传递哪些参数”，而不是只在文档、`commands.sh` 或某个变量中间接出现。

当参数来自运行时 shell 变量时，注意保留运行时展开语义。上面例子中的 `'"$bam"'`
表示把 `"$bam"` 保留到生成后的 shell 脚本中，而不是在 TAFFISH 编译 step 时提前展开。
这些变量还应该在调用前 `export`，因为依赖调用可能被物化为独立的 step shell。
在 `while read ... < file` 循环中，taf 调用通常要接 `</dev/null`，避免底层工具或容器
入口读取 stdin 时消耗循环输入。

如果底层工具需要动态长度参数列表，优先用少量显式分支保持 `[[taf: ...]]` 参数固定可读。
确实只能运行时决定参数数目时，可在 `<outdir>/02_intermediate/` 生成包含具体参数的
helper script，再在真实 call site 用相应 taf 依赖执行，例如：

```taf
[[taf: taf-tool-vX-rY sh '"$script"' ]] </dev/null
```

这仍然保持依赖调用可审计，`commands.sh` 只记录 provenance。
`commands.sh` 中可以另外记录便于审计和复现的命令文本，但实际执行入口仍应是源码
call site 处的 `[[taf: ...]]`。

## `@:` 参数块

Flow 经常需要把“面向用户的领域参数”和“底层工具的原生参数”连接起来。`(@:)` 块参数就是为这个场景准备的。

示例：

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

`(@:)fastqc-step` 会创建一个名为 `fastqc-step` 的 step 参数槽。对正式 flow 来说，
这个槽默认应为空：`threads`、`outdir`、`input` 这类 flow 自己管理的稳定参数直接写在
命令本体中，`::(@:)fastqc-step::` 只用于用户显式追加本次运行需要的底层 FastQC 原生参数。

- `threads` 暴露为 flow 顶层常用参数。
- `outdir` 和 `input` 属于 flow-managed 契约。
- `fastqc-step` 是高级用户的追加参数入口，只有用户传入 `@fastqc-step: ... @:` 时才展开。

在 `ARGS` 正文里，普通参数引用优先使用 `::name::`。`@name` 和 `@{name}` 也可用，但更适合默认值拼接，例如 `::(--/-p)prefix=out-@{input}::`。

这种方式适合 flow：

- 把稳定、常用、影响结果解释的参数作为 flow 顶层领域参数。
- 在 `RUN` 中明确写出上游命令和 flow-managed 参数。
- 避免把每个底层工具参数都提升成 flow 顶层概念。
- 在不破坏主流程结构的情况下给用户提供扩展能力。

如果一个 flow 有多个步骤，可以为每个步骤准备一个块参数，例如 `fastqc-step`、`align-step`、`call-variants-step`。一个块参数最好只对应一个步骤，不要混入其他步骤的参数。

对正式 flow，推荐把这条规则提升为默认规范：每个会影响分析结果、资源占用或主要报告内容的
真实 `[[taf: ...]]` call site，都应有一个唯一、语义清楚的 `(@:)<step-name>` 参数块，并在
调用点使用 `::(@:)<step-name>::`。版本探针、dependency anchor、极轻量存在性检查等不承载
真实分析语义的调用可以例外。

同一个 taf app 在同一 flow 中多次调用时，也应按 call site 拆分参数块名称。例如同一个
`taf-salmon` 可以有 `salmon-index-step`、`salmon-quant-pe-step` 和
`salmon-quant-se-step`。不要用一个宽泛的 `salmon-step` 或 `extra-step` 同时控制多种
语义不同的调用。

`@:` 参数块是高级用户传递底层工具原生参数的入口，不应该成为普通运行路径的必需条件。
`::(@:)<step-name>::` 默认展开必须为空；不要在 flow 源码、模板、shell 变量或 helper
script 中给它预填默认参数。flow 固有的默认工具参数应直接写在命令本体中，并作为正常
行为文档化。不传任何 `@step:` 参数时，flow 仍应能用领域级参数和常用参数跑通标准路径，
并产出结果、报告和 provenance。用户实际传入的 step 参数也应记录到 `commands.sh`、
`run.manifest.json` 或等价 provenance 中。

如果某个步骤必须通过 helper script 间接运行底层工具，`::(@:)<step-name>::` 应体现在
helper script 生成的真实工具命令中，而不是只追加到 `sh "$script"` 后面；否则用户传入的
底层参数可能并没有真正作用到目标工具。

## 依赖声明

每个被 flow 调用的 taf app 都应在 `[dependencies]` 中声明：

```toml
[dependencies]
taf-fastqc = "0.12.1-r1"
taf-multiqc = "1.19-r1"
```

key 是 command 基础名，不带版本：

```text
taf-fastqc
```

value 是 version id：

```text
0.12.1-r1
```

它对应的带版本命令是：

```text
taf-fastqc-v0.12.1-r1
```

不要写成：

```toml
taf-fastqc-v0.12.1-r1 = "0.12.1-r1"
```

## 多版本依赖

同一个 flow 可以依赖同一个 app 的多个版本：

```taf
<taffish>
[[taf: taf-align-v2.0.0-r1 old.fq]]
[[taf: taf-align-v2.1.0-r1 new.fq]]
```

对应：

```toml
[dependencies]
taf-align = ["2.0.0-r1", "2.1.0-r1"]
```

注意：这表示两个版本都需要安装，不是“任选一个版本”。

## 依赖同步

`taf build` 会根据 flow 的 `src/main.taf` 同步 `[dependencies]`，并移除过时依赖。

建议流程：

```sh
taf check
taf build
taf check
```

例如，若 `src/main.taf` 调用：

```taf
[[taf: taf-dep-tool-v0.1.0-r1 in.fa]]
[[taf: taf-new-tool-v1.2.3-r4 out.fa]]
```

运行 `taf build` 后，`taffish.toml` 应包含：

```toml
[dependencies]
taf-dep-tool = "0.1.0-r1"
taf-new-tool = "1.2.3-r4"
```

如果 `taf check` 报依赖缺失或版本不一致，通常说明：

- `src/main.taf` 中调用了新的 taf app。
- `[dependencies]` 没有更新。
- 调用的 exact version 与声明版本不一致。

发布前一定要重新运行：

```sh
taf build
taf check
```

## 安装行为

当用户安装 flow：

```sh
taf install qc-flow
```

`taf install` 会读取 index 中的 dependencies，并先安装依赖，再安装 flow 本身。

单版本依赖：

```json
"dependencies": {
  "taf-fastqc": "0.12.1-r1"
}
```

多版本依赖：

```json
"dependencies": {
  "taf-align": ["1.0.0-r1", "2.0.0-r1"]
}
```

安装器会把数组展开为多个依赖目标。

## 设计建议

建议：

- 每个 flow 专注一个清晰分析任务。
- flow 中的 taf app 调用尽量使用 exact versioned command。
- 对外暴露的参数不要太多，优先保留关键参数。
- 复杂步骤拆成多个 tool app 或子 flow。
- 在 `docs/help.md` 中说明依赖工具和主要版本。
- 对重要中间文件命名保持稳定。
- 发布前用小型示例数据跑通。

避免：

- 在 flow 中直接调用未声明的系统命令。
- 在 flow 中混用大量环境依赖。
- 把依赖数组当作备选版本。
- 依赖未发布或未被 Hub 收录的 app。

## 常见错误

`taf check` 报缺少 dependency：

- 检查 `src/main.taf` 中是否有新的 `[[taf: ...]]`。
- 运行 `taf build` 同步。
- 手动检查 `[dependencies]`。

安装 flow 后运行失败：

- 先运行 `taf which <dependency-command>`。
- 确认依赖版本已安装。
- 确认 flow 调用的是 exact versioned command。

依赖版本不一致：

- 检查 `[[taf: taf-x-vA-rB ...]]`。
- 检查 `[dependencies].taf-x`。
- 多版本时确认数组包含所有被调用版本。

想表达“任意一个版本都可以”：

- 当前官方语义不支持 alternatives。
- 依赖数组表示全部需要。
- 如果确实需要 alternatives，应该等待未来 schema 扩展。
