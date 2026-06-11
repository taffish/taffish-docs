# TAFFISH app 开发者指南

本文面向希望开发 TAFFISH app 的维护者。它是一份从创建项目到发布到官方 TAFFISH Hub 的操作手册。

官方 TAFFISH Hub 当前不是开放自助发布平台。只有 `taffish` GitHub 组织成员可以在官方组织下创建、发布和维护 app 仓库。目前官方维护者暂为单人维护；如果开发者希望把 app 纳入官方 Hub，需要先联系维护者申请加入组织，或由维护者协助审核与发布。

本文是通用 app 生命周期指南。精确字段含义以 [`taffish.toml` 规范](taffish-toml-spec.cn.md)
为准；Dockerfile、GHCR 和 smoke 细节以 [容器化 app 最佳实践](container-apps.cn.md)
为准；官方 Hub app 的精修 checklist 以 [官方 taf-app 精修手册](taf-app-curation-guide.cn.md)
为准；出错时优先查 [故障排查](troubleshooting.cn.md)。

## 目录

- [开发目标](#开发目标)
- [选择 app 类型](#选择-app-类型)
- [创建项目](#创建项目)
- [项目结构](#项目结构)
- [编写 `taffish.toml`](#编写-taffishtoml)
- [编写 `src/main.taf`](#编写-srcmaintaf)
- [参数设计与 `@:` 块参数](#参数设计与--块参数)
- [编写帮助文档](#编写帮助文档)
- [容器化 tool app](#容器化-tool-app)
- [flow app 与依赖](#flow-app-与依赖)
- [本地检查与运行](#本地检查与运行)
- [构建](#构建)
- [发布](#发布)
- [发布后检查](#发布后检查)
- [维护版本](#维护版本)
- [发布前检查清单](#发布前检查清单)

## 开发目标

一个合格的 TAFFISH app 应该满足：

- 有清晰的 package name 和 command name。
- 有稳定的版本号和 release 号。
- 有可运行的 `src/main.taf`。
- 有用户可读的 `docs/help.md`。
- 如果依赖容器，则镜像 tag 与 TAFFISH version id 对齐。
- 如果依赖容器，则 `[smoke]` 已替换为真实检查，而不是默认 `TODO` 占位。
- 如果准备进入公开 Hub，则尽量填写 `[meta]` 发现元数据。
- 如果包装第三方生信软件，则尽量填写 `[upstream]`。
- 如果是 flow，则准确声明 `[dependencies]`。
- 能通过 `taf check`。
- 能被 `taf build` 构建成版本化命令。
- 发布后能被 `taffish-index` 自动发现。

## 选择 app 类型

TAFFISH 当前有两类 app。

### Tool app

Tool app 用于包装一个具体工具，例如 BLAST、BWA、CD-HIT、FastQC。它通常：

- 包装一个上游软件。
- 可以绑定容器镜像。
- `runtime.pipe = true`。
- `runtime.command_mode = true`。
- 用户安装后直接获得一个工具命令。
- 当运行时第一个参数不像选项时，可以自动把用户给出的命令放进同一个 app
  运行环境/容器中执行。

### Flow app

Flow app 用于组合多个 TAFFISH app，形成分析流程。它通常：

- 使用 `<taffish>` 标签。
- 使用 `[[taf: ...]]` 调用其他 taf app。
- 在 `[dependencies]` 中声明依赖。
- `runtime.pipe = false`。
- `runtime.command_mode = false`。

简单判断：

- 如果你在包装一个已有命令行软件，选 tool。
- 如果你在串联多个 taf app，选 flow。
- 如果既需要上游工具又需要多步骤流程，通常先把上游工具拆成 tool，再用 flow 组合。

## 创建项目

创建默认 flow 项目：

```sh
taf new my-flow
```

创建 tool 项目：

```sh
taf new my-tool --tool
```

创建带 Dockerfile 和镜像构建 workflow 的 tool 项目：

```sh
taf new my-tool --tool --docker
```

指定版本、release、仓库：

```sh
taf new my-tool --tool --docker \
  --version 0.1.0 \
  --release 1 \
  --repo https://github.com/taffish/my-tool
```

## 项目结构

标准项目：

```text
my-tool/
  release.md
  taffish.toml
  src/
    main.taf
  docs/
    help.md
  target/
    .gitkeep
  README.md
  LICENSE
```

带 Dockerfile 的项目：

```text
my-tool/
  docker/
    Dockerfile
  .github/
    workflows/
      build-image.yml
```

关键文件职责：

- `taffish.toml`：app 元数据、运行方式、依赖、容器、smoke checks、平台和 upstream 信息。
- `src/main.taf`：app 的 TAFFISH 源码。
- `docs/help.md`：构建后命令的 `--help` 输出来源。
- `target/`：本地构建产物目录，不应作为源码手工维护。
- `docker/Dockerfile`：容器化 tool app 的镜像定义。
- `release.md`：被 ignore 的本地发布说明草稿，用于 publish message 和 GitHub Release notes。

`taf new` 会创建 `release.md` 并把它加入 `.gitignore`。它由
`taf publish --release` 使用，不应该作为 app 源码提交。

## 编写 `taffish.toml`

最小 tool 示例：

```toml
[package]
name = "my-tool"
kind = "tool"
version = "0.1.0"
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
```

最小 flow 示例：

```toml
[package]
name = "my-flow"
kind = "flow"
version = "0.1.0"
release = 1
license = "Apache-2.0"
main = "src/main.taf"

[repository]
url = "https://github.com/taffish/my-flow"

[command]
name = "taf-my-flow"

[runtime]
pipe = false
command_mode = false
```

`[repository].url` 应该填写 TAFFISH app 的 canonical GitHub 仓库地址。官方 Hub
package 不要在这里填写 Gitee 或内部镜像 URL。从 TAFFISH `0.2.0` 开始，镜像通过本地 `taf`
运行时配置处理，`[[source.rewrite]]` 会在 install 时重写 canonical URL。

推荐补充发现元数据：

```toml
[meta]
domain = "bioinformatics"
category = "sequence-clustering"
summary = "CD-HIT wrapper for clustering biological sequences."
keywords = ["cd-hit", "clustering", "sequence-identity"]
```

`[meta]` 对本地开发是可选的，但准备进入公开 Hub 的 app 推荐提供。它可以改善
Hub 搜索、筛选和详情页展示。

推荐补充 upstream：

```toml
[upstream]
name = "CD-HIT"
type = "github"
url = "https://github.com/weizhongli/cdhit"
homepage = "https://github.com/weizhongli/cdhit"
repository = "weizhongli/cdhit"
version = "4.8.1"
license = "GPL-2.0"
citation = "Fu et al. 2012"
doi = "10.1093/bioinformatics/bts565"
pmid = "23060610"
```

`[upstream]` 描述的是被包装软件的原始来源，不是 TAFFISH app 自己的仓库来源。
对生信工具，尽量补充已确认的学术归属信息：推荐填写 `citation` 和 `doi`；
如果 canonical paper 已被 PubMed 收录，也推荐填写 `pmid`。不能确认时宁可留空，
不要猜测。

新的公开 release 应该把发现元数据和 upstream 元数据保存在 `taffish.toml` 中。
如果已经发布且不可变的历史版本只是需要修正展示元数据，或补充已有 upstream 仓库的
归属/引用信息，官方 index 维护者可以使用 `metadata-overrides.toml`，而不是为此发布新的
app release。

更完整的字段规范见 [taffish.toml 规范](taffish-toml-spec.cn.md)。

## 编写 `src/main.taf`

Tool app 通常使用 `<taf-app:...>`：

```taf
<taf-app:shell>
my-tool ::*ARGV*::
```

容器化 tool app：

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool ::*ARGV*::
```

这种薄 wrapper 有两种常用模式：

```sh
taf-my-tool -- --help
taf-my-tool -- --input sample.fa --output result.txt
```

这些选项式调用会使用 `src/main.taf` 中的默认主体。但 tool app 还支持自动命令模式：

```sh
taf-my-tool my-tool --help
taf-my-tool helper-command --help
```

当运行时第一个参数不像选项时，TAFFISH 会把用户给出的命令放到同一个 app
运行环境/容器中执行。对于一个镜像里包含多个上游可执行程序的容器化 tool app，
优先使用这一能力。

Flow app 通常使用 `<taffish>`：

```taf
<taffish>
[[taf: taf-fastqc-v0.12.1-r1 fastqc sample.fq]]
[[taf: taf-multiqc-v1.19-r1 multiqc .]]
```

参数使用 `::...::`：

```taf
<taf-app:shell>
my-tool --threads ::threads=4:: --input ::input::
```

建议：

- 参数名尽量稳定，避免发布后频繁改 CLI。
- tool app 尽量把上游工具原始参数保留下来。
- flow app 尽量使用 exact versioned command，例如 `taf-fastqc-v0.12.1-r1`。
- 不要把大量复杂 shell 逻辑塞进一行，保持可读性。

## 参数设计与 `@:` 块参数

TAFFISH app 的参数设计不只是把 shell 字符串拼起来。对 tool app 来说，参数系统可以把上游工具的 CLI 暴露成稳定的 taf 命令；对 flow app 来说，`@:` 块参数尤其重要，因为它可以把一个分析步骤需要的一组底层参数封装成一个可复用、可扩展的步骤参数。

普通参数适合直接映射到用户输入：

```taf
ARGS
<!(--/-i)input>
<(--/-o)output>
  result.tsv
<(--/-t)threads>
  4

RUN
<taf-app:shell>
my-tool --input ::input:: --output ::output:: --threads ::threads::
```

`@:` 块参数适合作为“某一步的高级追加参数槽”：

```taf
ARGS
<!(--/-i)input>
<(--/-o)outdir>
  qc
<(--/-t)threads>
  4

<(@:)fastqc-step>

<(@:)multiqc-step>

RUN
<taffish>
mkdir -p ::outdir::
[[taf: taf-fastqc-v0.12.1-r1 fastqc --threads ::threads:: --outdir ::outdir:: ::input:: ::(@:)fastqc-step::]]
[[taf: taf-multiqc-v1.19-r1 multiqc ::outdir:: ::(@:)multiqc-step::]]
```

这个例子里：

- `input` 和 `outdir` 是 flow 的领域参数。
- `threads` 是 flow 顶层常用参数。
- `fastqc-step` 和 `multiqc-step` 默认都是空槽。
- 用户只有显式传入 `@fastqc-step: ... @:` 或 `@multiqc-step: ... @:` 时，才会追加底层工具原生参数。

这种写法的好处是：flow 的主逻辑保持清晰，普通用户调整的是稳定领域参数，高级用户仍能在必要时按步骤追加底层工具原生参数。

在 `ARGS` 的正文中，普通参数引用优先使用 `::name::`。`@name` 和 `@{name}` 也可以引用参数，但更适合用在默认值表达式或拼接场景，例如 `::(--/-p)prefix=out-@{input}::`。

建议：

- 对外暴露稳定、领域化的参数名，例如 `input`、`outdir`、`genome`、`threads`。
- 用 `(@:)step-name` 为一个工具调用步骤提供默认空的高级追加参数槽。
- 在正式 flow 中，每个会影响分析结果、资源占用或主要报告内容的真实 `[[taf: ...]]`
  call site 都应有对应的 `::(@:)step-name::` 高级参数入口。
- 同一个 taf app 多次调用时，按 call site 拆分 block 名称，例如 `salmon-index-step`
  和 `salmon-quant-step`，不要用一个宽泛的 app 级 block 同时控制多个步骤。
- `::(@:)step-name::` 默认展开为空；不要在源码、模板、shell 变量或 helper script 中预填默认参数。
- flow 自己需要的默认工具参数直接写在命令本体中，并作为正常行为文档化。
- 不传任何 `@step:` 参数时，flow 默认路径仍应完整可用；`@:` 是高级扩展入口，不是普通用户的必填参数。
- 用户实际传入的 step 参数应记录到 `commands.sh`、`run.manifest.json` 或等价 provenance。
- 发布后尽量不要修改已有参数名，必要时新增参数并保持旧参数兼容。

## 编写帮助文档

`docs/help.md` 是必需文件。构建后的命令执行：

```sh
taf-my-tool-v0.1.0-r1 --help
```

会输出该文件内容。

建议包含：

- app 简介
- 上游软件来源
- 常用命令示例
- 参数说明，包括 `@:` 块参数中暴露出的领域参数和 `extra` 参数
- 输入输出说明
- 容器镜像说明
- 版本信息
- 引用方式
- 许可证说明

最小示例：

````md
# my-tool

A TAFFISH wrapper for My Tool.

## Usage

```sh
taf-my-tool-v0.1.0-r1 --input sample.fa --output result.txt
```
````

## 容器化 tool app

如果 tool app 依赖复杂系统包、编译环境或生信软件，推荐使用容器。

最小容器元数据：

```toml
[container]
image = "ghcr.io/taffish/my-tool:0.1.0-r1"
dockerfile = "docker/Dockerfile"
build_platforms = "linux/amd64,linux/arm64"

[smoke]
backend = "docker"
timeout = 60
exist = ["my-tool"]
test = ["my-tool --help"]
```

最小入口：

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --input ::input::
```

要求：

- 镜像 tag 应与 `<version>-r<release>` 一致。
- Dockerfile 应随 app 仓库版本化。
- 镜像构建 workflow 应在 app 仓库内。
- GHCR package 应设置为 public，否则用户无法拉取。
- `[smoke]` 应包含真实检查，用于证明目标可执行文件存在，并且一个最小命令可以运行。

`taf check` 会校验 `[smoke]` 结构，并拒绝默认 `TODO` 占位。它不会在本地运行
smoke 命令。公开 Hub/index 自动化会在镜像可用后运行 smoke checks，并只把通过的新版本写入主 index。

本节只给出最小形状。Dockerfile 设计、运行时挂载、GHCR 可见性、本地 Docker/Podman
测试和 smoke 细节见 [容器化 app 最佳实践](container-apps.cn.md)。如果是在精修官方
Hub app，还应参考 [官方 taf-app 精修手册](taf-app-curation-guide.cn.md)。

## flow app 与依赖

Flow app 通过 `[[taf: ...]]` 调用其他 app：

```taf
<taffish>
[[taf: taf-trim-v1.0.0-r1 sample.fq]]
[[taf: taf-align-v2.0.0-r1 sample.trimmed.fq]]
```

依赖应写入 `taffish.toml`：

```toml
[dependencies]
taf-trim = "1.0.0-r1"
taf-align = "2.0.0-r1"
```

如果同一个 flow 需要同一 app 的多个版本：

```toml
[dependencies]
taf-align = ["2.0.0-r1", "2.1.0-r1"]
```

这表示两个版本都需要安装，不表示二选一。

更多细节见 [Flow 与依赖指南](flow-dependencies.cn.md)。

## 本地检查与运行

检查项目：

```sh
taf check
```

编译当前项目：

```sh
taf compile -- --input sample.fa
```

运行当前项目：

```sh
taf run -- --input sample.fa
```

如果是容器化 app，并且镜像还没有发布到远端 registry，通常需要先在本地构建镜像，再用相同后端试运行：

```sh
taf build --image --backend docker
taf run --backend docker -- --input sample.fa
```

`taf run --backend` 会强制泛化 `<container:...>` / `<taf-app:container:...>` 标签选择对应后端。它不会覆盖显式标签，例如 `<taf-app:podman:...>` 或 `<taf-app:docker:...>`。

命令 wrapper 安装后，或者直接使用 `taffish` 编译时，可以用
`TAFFISH_CONTAINER_BACKEND=apptainer|podman|docker` 在运行时强制泛化容器标签：

```sh
TAFFISH_CONTAINER_BACKEND=podman taf-my-tool-v0.1.0-r1 -- --help
TAFFISH_CONTAINER_BACKEND=podman taf-my-tool-v0.1.0-r1 --compile -- --help
```

这个环境变量不会覆盖显式 backend 标签。本地项目运行时，`taf run --backend ...`
的优先级高于这个环境变量。

如果 app 确实只能使用某一个后端，或者开发期只在某个本地后端中存在，建议把
`src/main.taf` 中的泛化标签改成显式后端标签：

```taf
<taf-app:podman:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --input ::input::
```

如果 app 本身可移植，但不同后端需要不同 runtime 参数，应保留泛化 `container` 标签，并使用
TAFFISH `0.9.0` 的结构化 runtime 参数：

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1$@[docker: --gpus all][podman: --device nvidia.com/gpu=all][apptainer: --nv]>
my-tool ::*ARGV*::
```

这用于 app 实现需求。例如被封装工具正常运行就需要 GPU 时，GPU 参数应该写在 app
源码中。`TAFFISH_DOCKER_RUN_ARGS`、`TAFFISH_PODMAN_RUN_ARGS` 或
`TAFFISH_APPTAINER_RUN_ARGS` 只用于本机、站点、平台或单次运行策略，不用于表达 app
自身语义。

常见检查内容：

- `taffish.toml` 必需字段。
- `src/main.taf` 是否存在并可解析。
- `docs/help.md` 是否存在。
- Dockerfile 是否存在。
- 容器镜像 tag 是否匹配当前 version/release。
- flow 依赖是否声明。

## 构建

构建版本化 command wrapper：

```sh
taf build
```

`taf build` 默认只构建 command wrapper，不会构建容器镜像。

构建容器镜像：

```sh
taf build --image
```

同时构建 command wrapper 和容器镜像：

```sh
taf build --all
```

指定镜像构建后端：

```sh
taf build --all --backend docker
```

本地开发容器化 app 时，推荐先明确后端：

```sh
taf build --image --backend docker
taf run --backend docker -- --help
taf build
```

如果希望一次性构建镜像和 command wrapper，可以使用：

```sh
taf build --all --backend docker
```

构建产物：

```text
target/taf-my-tool-v0.1.0-r1
target/.taf-my-tool-v0.1.0-r1/
```

构建后的命令使用冻结的 source snapshot，而不是实时读取当前 `src/`。这保证发布后的命令逻辑与版本一致。

## 发布

发布前，先编辑被 ignore 的 `release.md` 草稿。使用 `taf publish --release`
时，`release.md` 第一行会成为 publish commit/tag message，整个文件会成为
GitHub Release notes。正式发布前应替换默认的 `TODO` 摘要。从 TAFFISH
`0.8.1` 开始，`taf publish --release` 只会拒绝第一行仍为默认占位符
`TODO` 或 `TODO: release summary` 的 release notes；真实 summary 中正常出现
`todo` 这个词不会被误拒绝。

带 release notes 预览：

```sh
taf publish --release --dry-run
```

带 release notes 构建并发布：

```sh
taf publish --release --yes --build
```

如果远端仓库不存在，可以请求创建：

```sh
taf publish --release --yes --build --create-repo --public
```

发布会：

1. 运行 `taf check`。
2. 读取 `[repository].url`。
3. 检查远端是否已有目标 tag。
4. 检查当前 tag 是否符合 latest 策略。
5. 从 `release.md` 准备发布 message 和 GitHub Release notes。
6. commit 当前项目。
7. 创建 tag `v<version>-r<release>`。
8. push commit 和 tag，并创建或更新 GitHub Release。

默认发布策略要求当前版本是远端最新版本。如果需要发布非最新预发布版本，可以使用：

```sh
taf publish --release --yes --pre
```

不要直接发布以下项目：

- 仓库 URL 不正确。
- 镜像 tag 尚未构建。
- `taf check` 失败。
- `docs/help.md` 缺失。
- 容器化 app 缺少 `[smoke]`，或 `[smoke]` 仍然包含 `TODO`。
- `release.md` 第一行仍然是默认 `TODO` 摘要。
- release tag 已经存在。

## 发布后检查

发布后应检查：

- GitHub release tag 是否存在。
- 如果有 Dockerfile，GitHub Actions 镜像构建是否成功。
- GHCR package 是否 public。
- 如果新的容器化版本没有进入 index，检查 `taffish-index` report；可能是 smoke 或 digest gate 未通过。
- `taffish-index` 下一次自动运行后是否收录 app。
- [taffish.github.io](https://taffish.github.io) 是否能看到 app。
- 本地执行 `taf update` 后是否能 `taf install`。

手动验证：

```sh
taf update
taf search my-tool
taf info my-tool
taf install --dry-run my-tool
```

如果是公开 Hub 发布前的私有/本地测试，也可以直接从项目目录安装：

```sh
taf install --from /path/to/my-tool
taf list
taf which taf-my-tool
```

`taf install --from` 会检查本地项目、把当前工作树复制到选定的 TAFFISH home、
构建带版本的 command wrapper，并把来源记录为 `[local-project] <PROJECT-ROOT>`。
它不需要先运行 `taf update`，也不会自动安装依赖。

## 维护版本

TAFFISH 使用：

```text
version = "0.1.0"
release = 1
version_id = "0.1.0-r1"
tag = "v0.1.0-r1"
```

什么时候增加 `release`：

- TAFFISH 包装逻辑修复。
- Dockerfile 修复。
- 帮助文档修复。
- app 元数据修复。
- 上游软件版本不变，但 TAFFISH 打包方式变化。

什么时候增加 `version`：

- 上游软件版本变化。
- flow 的主要功能版本变化。
- app 行为发生需要用户明确感知的变化。

不要覆盖已经发布的 tag。需要修复时发布新的 release，例如从 `0.1.0-r1` 到 `0.1.0-r2`。

## 发布前检查清单

发布前逐项确认：

- [ ] `taffish.toml` 中 `name`、`version`、`release`、`repository.url` 正确。
- [ ] `command.name` 以 `taf-` 开头。
- [ ] `src/main.taf` 可以被 `taf check` 解析。
- [ ] `docs/help.md` 已更新。
- [ ] 如果准备进入公开 Hub，`[meta]` 中包含有用的 domain、category、summary 和 keywords。
- [ ] 如果是 tool app，确认上游软件来源写入 `[upstream]`。
- [ ] 对学术生信工具，`[upstream]` 包含已确认的 `license`、`citation`、`doi`，以及可选的 `pmid`。
- [ ] 如果是容器化 app，镜像 tag 与 `version-release` 一致。
- [ ] 如果是容器化 app，`[smoke]` 已经包含真实的 `exist` 或 `test` 检查，并且没有 `TODO` 占位。
- [ ] 如果是容器化 app，本地 `taf build --image --backend ...` 和 `taf run --backend ...` 使用一致后端。
- [ ] 如果要在公开发布前测试私有 app，`taf install --from <PROJECT-DIR>` 可以从项目根目录或子目录正常安装。
- [ ] 如果是 flow app，`[dependencies]` 与 `[[taf: ...]]` 一致。
- [ ] `taf run` 在本地通过。
- [ ] `taf build` 或 `taf build --all` 通过。
- [ ] `release.md` 第一行和 release notes 已经整理好。
- [ ] `taf publish --release --dry-run` 输出符合预期。
- [ ] 发布后检查 GitHub Actions 和 Hub index。
