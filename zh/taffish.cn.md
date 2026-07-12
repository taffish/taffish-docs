# 什么是 TAFFISH

TAFFISH 是一个面向生物信息学命令行工具与轻量流程的 shell-native 可复现执行与
分发层。它把命令级可复现性带入生物信息学用户原本就在使用的 shell 命令中，
而不要求用户先迁入新的工作流系统。

TAFFISH 有三个本地入口：

- `taffish`：TAFFISH 语言编译器，把 `.taf` 文件编译成 POSIX shell 脚本。
- `taf`：面向开发者和用户的 CLI，负责创建项目、检查项目、构建命令、发布 app，以及从 TAFFISH Hub 安装 app。
- `taffish-mcp`：一个保守的 stdio MCP server，用于向 AI 客户端暴露安全的 TAFFISH tools、resources、prompts、app inspection 和 project inspection。

换句话说，`.taf` 描述工具或轻量流程应该如何运行；`taffish` 负责把它变成 shell；
`taf` 负责把这种代码组织成可版本化、可解析容器、可发布、可索引、可安装、
可组合的可执行包；`taffish-mcp` 则让 AI 客户端通过结构化接口检查 TAFFISH
项目、app 和 Hub 状态。

## 目录

- [设计目标](#设计目标)
- [安装](#安装)
- [用户级与系统级作用域](#用户级与系统级作用域)
- [`taffish` 编译器](#taffish-编译器)
- [`.taf` 文件结构](#taf-文件结构)
- [参数语法](#参数语法)
- [内置运行标签](#内置运行标签)
- [`taf` CLI](#taf-cli)
- [MCP / AI 集成](#mcp--ai-集成)
- [运行时配置与镜像源](#运行时配置与镜像源)
- [开源与源码构建](#开源与源码构建)
- [TAFFISH app 项目结构](#taffish-app-项目结构)
- [推荐开发流程](#推荐开发流程)
- [Tool 与 Flow 的区别](#tool-与-flow-的区别)
- [当前边界](#当前边界)

## 设计目标

TAFFISH 的目标不是替代 shell、容器、包管理器或 workflow engine，而是通过
可执行包模型，让工具调用更稳定、更易迁移，并且仍可作为普通 shell 命令直接使用：

- 每个 app 有明确的名称、版本、release 和命令名。
- 每个 app 的运行逻辑写在 `.taf` 文件里。
- 每个 app 可以选择绑定容器镜像。
- 每个 app 可以被构建成版本化命令，例如 `taf-blast-v2.16.0-r1`。
- 每个 app 可以被 TAFFISH Hub 索引，并通过 `taf update` / `taf install` 安装。
- flow app 可以声明并调用其他 taf app，完成轻量组合。

TAFFISH 不是通用工作流引擎。Nextflow、Snakemake 等系统仍可负责 DAG 编排、
调度、缓存与流程级可复现，并把已安装的 `taf-*` 命令作为普通 shell 命令调用。

## 安装

公开的 `taffish/taffish` 仓库现在提供 TAFFISH 开源实现、安装器、源码树构建文档和二进制 release 载荷。用户安装推荐使用该仓库中的权威安装器：

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --user
```

系统安装：

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sudo sh -s -- --system
```

固定安装某个版本：

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --version X.Y.Z --user
```

请把 `X.Y.Z` 替换为要安装的 release 版本。

中国大陆用户可以使用 Gitee 安装器，减少安装阶段对 GitHub raw content 的依赖，
并初始化中国镜像配置：

```sh
curl -fsSL https://gitee.com/taffish-org/taffish/raw/main/install/install-taffish.gitee.sh | sh -s -- --user
```

安装后进行检查：

```sh
taf --version
taffish --version
taffish-mcp --version
taf doctor --user
```

安装后会得到三个命令：

```text
taffish
taf
taffish-mcp
```

安装器还会把 shell 自动补全文件和 Vim 语法文件复制到 TAFFISH home。
用户级安装时通常位于 `~/.local/share/taffish/share/completions` 和
`~/.local/share/taffish/share/vim`。

| 项目 | 用户级安装 | 系统级安装 |
| --- | --- | --- |
| 核心命令 | `~/.local/bin` | `/usr/local/bin` |
| 已安装的 `taf-*` 命令 | `~/.local/share/taffish/bin` | `/usr/local/bin` |
| TAFFISH home | `~/.local/share/taffish` | `/opt/taffish` |

安装器会尝试初始化与安装方式匹配的作用域并更新其 index。如果网络不可用，
安装器会给出警告，但不会回滚已经安装的二进制文件。它只复制 shell 和编辑器
集成文件，不会直接修改个人或全局启动配置。

源码构建、release 校验、维护者构建路径和贡献说明，请参考
[taffish/taffish README](https://github.com/taffish/taffish)。

## 用户级与系统级作用域

TAFFISH 把用户级状态和系统级状态彼此分开。支持作用域的 `taf` 命令接受
`--user` / `-u` 与 `--system` / `-s`，并且每次调用都默认使用用户级作用域。
把核心二进制安装到系统目录，并不会让后续命令自动使用系统级状态。

典型个人初始化：

```sh
taf doctor --init --user
taf update --user
taf install --user fastqc
```

典型管理员操作：

```sh
sudo taf doctor --init --system
sudo taf update --system
sudo taf install --system fastqc
taf list --system
```

写入系统级状态的命令应使用 `sudo`；`taf search --system`、
`taf list --system`、`taf which --system` 等只读命令通常不需要。共享工作站、
服务器、HPC 登录节点、全局补全、配置继承、升级与卸载的完整说明见
[TAFFISH 系统管理指南](system-administration.cn.md)。

## `taffish` 编译器

`taffish` 的直接职责是把 `.taf` 文件编译成 shell：

```sh
taffish path/to/main.taf [ARGS...]
```

也可以从标准输入读取：

```sh
cat path/to/main.taf | taffish --
```

查看版本和帮助：

```sh
taffish --version
taffish --help
```

最小 `.taf` 文件：

```taf
<shell>
echo "hello from TAFFISH"
```

编译它：

```sh
taffish hello.taf
```

`taffish` 默认只输出生成后的 shell 脚本，不直接执行。实际执行通常由 `taf run` 或构建后的 `taf-*` 命令完成。

## `.taf` 文件结构

`.taf` 文件由若干行组成。每一行会被识别为以下类型之一：

- 空行
- 注释行：去掉前导空格后以 `#` 开头
- 主标签：`ARGS` 或 `RUN`
- 子标签：`<...>`
- 普通代码行

完整结构通常是：

```taf
ARGS
<(--/-n)name>
  world

RUN
<shell>
echo "hello, ::name::"
```

主标签是 `ARGS` 或 `RUN`；子标签包括 `<shell>`、`<container:...>`、
`<taffish>` 和 `<taf-app:...>`。

如果 `.taf` 文件第一条有效行就是普通代码，TAFFISH 会自动补成：

```taf
RUN
<taffish>
...
```

如果第一条有效行是 `<...>` 子标签，TAFFISH 会自动补一个 `RUN`。因此最短形式也可以直接写：

```taf
<shell>
echo "hello"
```

## 参数语法

TAFFISH 使用 `::...::` 表示参数插槽。编译时，插槽会根据传入参数、默认值和内置上下文被解析并替换。

最短形式可以直接在 `RUN` 中写参数插槽：

```taf
<shell>
echo "input: ::input::"
echo "name: ::(--/-n)name=world::"
```

更推荐的写法是使用 `ARGS` 显式声明参数，再在 `RUN` 中引用：

```taf
ARGS
<!(--/-i)input>
<(--/-t)threads>
  4

RUN
<shell>
my-tool --input ::input:: --threads ::threads::
```

`ARGS` 中每个 `<...>` 子标签声明一个参数。子标签下面的正文是默认值；如果用户传入了对应参数，用户输入会覆盖默认值。

### 参数 DSL

`<...>` 和 `::...::` 内部使用同一套参数 DSL：

```text
!(--/-i)input        required option: --input / -i
(--/-t)threads=8    option with inline default
(--/-v)verbose?     boolean flag
%(--/-x)internal=1   hidden option with default
$1                   positional argument
(@:)blast-step       block / slot argument
```

常用前缀和后缀：

- `!` 表示必需参数。
- `%` 表示隐藏参数，适合内部默认值或不希望暴露给普通用户的参数。
- `?` 表示布尔 flag。出现时在参数绑定中为真，未出现时为空或类似 false 的空值；不要依赖固定的
  `true` / `false` 文本输出。
- `=` 表示默认值。
- `$1`、`$2` 表示从用户输入中读取的位置参数。
- `--` 可以自动展开成长参数名，例如 `(--/-)name` 对应 `--name` 和 `-n`。
- `-` 可以自动展开成短参数名，例如 `(--/-)threads` 对应 `--threads` 和 `-t`。

默认值可以引用其他参数：

```taf
ARGS
<name>
  sample
<message>
  hello, ::name::
<command>
  --sample ::name:: --threads ::(--/-t)threads=4::

RUN
<shell>
echo "::message::"
echo "::command::"
```

在 `ARGS` 的正文里，`::name::`、`@name` 和 `@{name}` 都可以引用另一个参数。推荐优先使用 `::name::`，因为它和 `RUN` 中的参数插槽保持一致，也方便 TAFFISH 从默认值中抽取内层参数。

`@name` 和 `@{name}` 是默认值表达式语法，适合保留给拼接场景，尤其是写在参数 DSL 的默认值里：

```taf
ARGS
<name>
  sample

RUN
<shell>
echo ::(--/-p)prefix=out-@{name}::
```

当引用后面紧跟字母、数字、`-` 或 `_` 时，使用 `@{name}` 可以避免边界歧义。

### `@:` 块参数

`(@:)name` 是 TAFFISH 参数系统里很重要的块参数。它会创建一个命名 slot，例如：

```taf
ARGS
<!(--/-q)query>
<(--/-d)db>
  nt
<(--/-of)outfmt>
  6

<(@:)blast-step>

RUN
<taffish>
[[taf: taf-blast-v2.16.0-r1 blastn -db ::db:: -query ::query:: -outfmt ::outfmt:: ::(@:)blast-step::]]
```

这里 `blast-step` 不是一个普通字符串参数，而是一个命名 step 参数槽。正式 flow 中推荐让它默认
为空：`db`、`query`、`outfmt` 这类 flow 自己管理的稳定参数直接写在命令本体中，
`::(@:)blast-step::` 只在用户显式传入 `@blast-step: ... @:` 时追加底层 BLAST 原生参数。
这样开发者可以：

- 对外暴露领域化参数，例如 `query`、`db`、`outfmt`。
- 把 flow-managed 默认参数和高级用户追加参数区分开。
- 在 flow 中保持真实 `[[taf: ...]]` 调用清楚可审计。
- 给高级用户保留按步骤追加原生参数的入口。

从命令行层面看，`(@:)blast-step` 对应 `@blast-step:` 这个 slot。普通用户通常不需要直接使用
slot；app 作者更多是用它在 `.taf` 内部为真实工具调用提供默认空的高级追加参数入口。

在 flow app 中，推荐让每个真实分析型 `[[taf: ...]]` 调用点都有一个语义清楚的
`(@:)xxx-step` 参数块，并在调用处写入 `::(@:)xxx-step::`。这样普通用户可以继续使用
flow 的领域级参数，而高级用户仍能按步骤传递底层工具原生参数。`::(@:)xxx-step::` 默认
必须展开为空，不应在源码、模板、shell 变量或 helper script 中预填默认参数；不传任何
`@step:` 参数时，flow 仍应能完整运行。

### 内置参数

内置参数包括：

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

示例：

```taf
<shell>
echo "user   : ::*USER*::"
echo "workdir: ::*WORKDIR*::"
echo "argv   : ::*ARGV*::"
```

TAFFISH 的转义不是通用 shell 风格转义，只有少数语法字符会被特定解析器消费。

`.taf` 词法层只识别这些转义：

```text
\:  -> :
\<  -> <
\#  -> #
\\  -> \
```

其他反斜杠序列会按原样保留。比如 `\@` 不是 `.taf` 词法层转义；它只会在 `ARGS` 默认值表达式中被参数解析器解释为字面量 `@`。

`ARGS` 默认值表达式支持这些转义：

```text
\@  -> @
\{  -> {
\}  -> }
\\  -> \
```

## 内置运行标签

运行标签是编译期分发标记。它们决定标签下面的正文会以什么方式被 emit 成 shell。
本节示例会同时给出 `.taf` 源码和“简化后的编译结果形态”。真实生成脚本还会包含
注释、检查、临时路径和 backend 检测等细节。

### `<shell>`

`<shell>` 表示下面的内容就是 shell 代码：

```taf
<shell>
echo "hello"
pwd
```

简化后的编译形态：

```sh
#!/bin/sh
echo "hello"
pwd
```

`<shell>` 不会进入容器。它在宿主机 shell 上运行，因此使用宿主机命令、宿主机
`PATH` 和宿主机工作目录语义。

### `<container:IMAGE>`

`<container:IMAGE>` 表示用可用的容器后端运行下面的 shell。`container` 是泛化后端，会根据环境选择 `apptainer`、`podman` 或 `docker`。

```taf
<container:ghcr.io/taffish/blast:2.16.0>
blastp -query ::query:: -db ::db::
```

简化后的编译形态：

```sh
# 选择一个 backend，例如 podman
podman pull ghcr.io/taffish/blast:2.16.0
podman run --rm -i \
  -w "$PWD" \
  -v "$HOME:$HOME" \
  -v "$PWD:$PWD" \
  -e "HOME=$HOME" \
  -e "USER=$USER" \
  ghcr.io/taffish/blast:2.16.0 \
  blastp -query "$query" -db "$db"
```

Docker、Podman 和 Apptainer 的具体命令会不同，但重要模型是一致的：

- TAFFISH 会选择一个容器 backend。
- 需要时检查或拉取镜像。
- 把容器内工作目录设置为当前 TAFFISH workdir。
- 把用户 home 路径和当前 workdir 按相同路径 bind mount 进容器。
- 传入 `HOME`、`USER` 等常见环境变量。
- 标签正文中的命令会在容器镜像内部执行。

由于当前 workdir 会按相同路径挂载进容器，不要在会和容器内部关键路径冲突的宿主机
目录下运行容器化 taf app，例如 `/bin`、`/usr/bin`、`/usr/local/bin`、`/lib`、
`/opt`，或者上游工具实际安装所在目录。宿主机 bind mount 可能遮住镜像中原本的
同名目录。例如在宿主机 `/usr/bin` 下运行容器化 app 时，挂载可能遮住容器自己的
`/usr/bin`，导致明明安装在容器里的工具变成“找不到”。建议在项目目录、数据目录或
临时工作目录中运行 taf app。

也可以明确指定后端：

```taf
<docker:ghcr.io/taffish/blast:2.16.0>
blastp -query ::query:: -db ::db::
```

可用后端由运行环境决定。`taf run` 可以用 `--backend` 强制选择：

```sh
taf run --backend docker -- --query test.fa --db nr
```

`--backend` 只会强制泛化容器标签，例如 `<container:...>` 或 `<taf-app:container:...>`。如果标签已经明确写成 `<docker:...>`、`<podman:...>`、`<taf-app:docker:...>` 或 `<taf-app:podman:...>`，TAFFISH 会尊重标签中的显式后端。

对于已经安装好的 `taf-*` 命令，或者直接使用 `taffish` 编译时，可以设置
`TAFFISH_CONTAINER_BACKEND=apptainer|podman|docker`，在运行时强制通用
`<container:...>` 标签使用指定后端：

```sh
TAFFISH_CONTAINER_BACKEND=podman taf-my-tool-v0.1.0-r1 [ARGS...]
TAFFISH_CONTAINER_BACKEND=podman taf-my-tool-v0.1.0-r1 --compile -- [ARGS...]
```

这个环境变量不会覆盖显式的 `<docker:...>`、`<podman:...>` 或
`<apptainer:...>` 标签。使用 `taf run` 时，`--backend` 的优先级高于
`TAFFISH_CONTAINER_BACKEND`。

TAFFISH `0.9.0` 增加了容器标签内部的结构化 backend-specific runtime 参数。
它们写在 `$` 后面的 `@[target: args]` block 中：

```taf
<container:ghcr.io/taffish/my-tool:1.0.0-r1$@[all: --network host][docker/podman: --security-opt=label=disable]>
my-tool --help
```

target 可以是 `all`、作为 `all` 别名的 `container`、`docker`、`podman`、
`apptainer`，也可以是 `docker/podman` 这样的后端组合。这类 tag 参数用于 app
实现本身的运行需求。例如某个被封装的 app 必须使用 GPU 才能正常工作时，应在
`src/main.taf` 中声明对应后端的 GPU 参数：

```taf
<container:ghcr.io/taffish/my-gpu-tool:1.0.0-r1$@[docker: --gpus all][podman: --device nvidia.com/gpu=all][apptainer: --nv]>
my-gpu-tool --help
```

本地 runtime policy 则应放在环境变量里：

```sh
TAFFISH_DOCKER_RUN_ARGS="--platform linux/amd64" taf-my-tool ...
TAFFISH_PODMAN_RUN_ARGS="--platform linux/amd64" taf-my-tool ...
TAFFISH_APPTAINER_RUN_ARGS="--bind /scratch:/scratch" taf-my-tool ...
```

这些变量用于单次运行、机器、集群、平台或站点策略。它们会追加 backend-specific
runtime 参数，但不会修改 app 源码。

本地测试未发布镜像时，构建镜像和运行项目应使用一致后端：

```sh
taf build --image --backend docker
taf run --backend docker -- --help
```

如果这个 app 只适合某个后端，或者本地镜像只存在于某个后端，可以在 `src/main.taf` 里把泛化标签改成显式标签：

```taf
<taf-app:podman:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --help
```

### `<taffish>`

`<taffish>` 是 flow 常用标签。它允许在 shell 中嵌入其他 taf app：

```taf
<taffish>
[[taf: taf-fastqc-v0.12.1-r1 fastqc sample.fq]]
[[taf: taf-multiqc-v1.19-r1 multiqc .]]
```

嵌入语法是：

```text
[[taf: taf-command args...]]
```

编译时，TAFFISH 会把这些 taf app 先编译成临时 shell step，然后在当前流程中执行。这个机制让 flow 可以组合已有工具，并且保留明确的依赖关系。

在正式 flow 中，`[[taf: ...]]` 应写在真正执行该工具的代码位置。不要把所有 taf 依赖集中
放在文件开头的 dummy block 或 `if false` 中，再通过变量、裸 wrapper 或生成脚本间接运行。
除少量 shell/coreutils 通用命令外，非 shell 原生的生信、统计、绘图、数据库和模型相关
命令都应通过显式版本化的 taf app 调用。

简化后的编译形态：

```sh
taffish_tmpdir=$(mktemp -d)
taf-fastqc-v0.12.1-r1 --compile fastqc sample.fq > "$taffish_tmpdir/step-1.sh"
taf-multiqc-v1.19-r1 --compile multiqc . > "$taffish_tmpdir/step-2.sh"
chmod +x "$taffish_tmpdir"/step-*.sh
"$taffish_tmpdir/step-1.sh"
"$taffish_tmpdir/step-2.sh"
```

如果只想输出字面量 `[[taf: ...]]`，可以在 `<taffish>` block 中用 `\[` 或 `\]` 转义方括号：

```taf
<taffish>
echo \[[taf: taf-example arg]]
```

### `<taf-app:...>`

`<taf-app:...>` 是 app 项目中常见的顶层标签。`taf new --tool` 生成的工具项目通常会使用它。

例如一个容器化 tool app：

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --input ::input:: --output ::output::
```

构建后的命令会把 `.taf` 编译为 shell 并执行。用户调用时看到的是普通命令，但内部仍然由 TAFFISH 编译和调度。

对于 shell tool app：

```taf
<taf-app:shell>
my-tool ::*ARGV*::
```

当用户运行 `taf-my-tool -- --alpha 1` 时，简化后的编译形态是：

```sh
#!/bin/sh
my-tool --alpha 1
```

对于容器化 tool app：

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool ::*ARGV*::
```

当用户运行 `taf-my-tool -- --alpha 1` 时，简化后的编译形态是：

```sh
#!/bin/sh
# 按上文描述选择、拉取并运行容器
podman run --rm -i \
  -w "$PWD" \
  -v "$HOME:$HOME" \
  -v "$PWD:$PWD" \
  ghcr.io/taffish/my-tool:0.1.0-r1 \
  my-tool --alpha 1
```

因此 `<taf-app:...>` 是面向用户 app 命令的 wrapper，底层仍然使用前面的运行标签。
它还保留了 `::*ARGV*::`、`--compile` 和包管理集成等 app 命令语义。

## `taf` CLI

`taf` 是 TAFFISH 的开发和包管理命令。主要命令分为三类。

### 项目命令

```sh
taf new APP_NAME
taf check
taf compile [ARGS...]
taf run [ARGS...]
taf build
taf publish
```

创建 flow 项目：

```sh
taf new my-flow
```

创建 tool 项目：

```sh
taf new my-tool --tool
```

创建带 Dockerfile 和 GitHub Actions 镜像构建 workflow 的 tool 项目：

```sh
taf new my-tool --tool --docker
```

检查项目：

```sh
taf check
```

编译当前项目并打印 shell：

```sh
taf compile -- --input sample.fa
```

直接运行当前项目：

```sh
taf run -- --input sample.fa
```

构建版本化命令：

```sh
taf build
```

生成的命令位于：

```text
target/<command-name>-v<version>-r<release>
```

例如：

```text
target/taf-my-tool-v0.1.0-r1
```

发布项目：

```sh
taf publish --release --dry-run
taf publish --release --yes --build
```

`taf new` 会创建一个被 ignore 的 `release.md` 草稿。使用
`taf publish --release` 时，第一行会成为 publish message，整个文件会成为
GitHub Release notes。

发布会检查项目、读取 `taffish.toml` 中的仓库地址、检查远端 tag，并在确认后
commit、tag、push，并创建或更新 GitHub Release。

### Hub 命令

```sh
taf update
taf search KEYWORD
taf info APP_OR_COMMAND
taf install APP_OR_COMMAND
taf outdated
taf upgrade
taf prune
taf uninstall APP_OR_COMMAND
taf list
taf which TAF_COMMAND
```

这些命令默认操作用户级状态。在脚本和管理员 runbook 中应显式使用 `--user`
或 `--system`；系统级安装核心二进制不会永久记住系统级作用域。修改系统状态的
命令通常需要 `sudo`，只读的系统级检查通常不需要。

更新本地索引：

```sh
taf update
```

搜索在线索引缓存：

```sh
taf search blast
taf list --online
```

安装最新版本：

```sh
taf install blast
```

安装指定版本：

```sh
taf install blast 2.16.0-r1
```

先 dry-run 预览安装全部已索引 app，再按需执行：

```sh
taf install --all
taf install --all --tools --yes
```

用当前本地 index 检查和维护本地已安装 Hub app：

```sh
taf update
taf outdated
taf upgrade
taf upgrade --yes
taf prune
taf prune --yes
```

`taf outdated` 会报告本地已安装 app 中存在更新 index 版本的项目。`taf upgrade`
默认只 dry-run，需要 `--yes` 才会修改本地安装。`taf prune` 会删除较旧的本地
版本，并保留本地已安装的最新版本。通过 `taf install --from` 安装的 app 会被视为
本地/私有安装，不会被公开 index 中的版本静默替换。

安装尚未发布到公开 Hub 的私有/本地 TAFFISH app 项目：

```sh
taf install --from /path/to/my-private-tool
taf list
taf which taf-my-private-tool
```

`taf install --from` 会读取本地项目的 `taffish.toml`、检查项目、把当前工作树复制到选定的
TAFFISH home、构建带版本的 command wrapper，并把安装来源记录为
`[local-project] <PROJECT-ROOT>`。路径可以是项目根目录，也可以是项目内任意子目录；
TAFFISH 会向上搜索 `taffish.toml`。它不需要先运行 `taf update`，也不会自动安装依赖。

也可以使用命令名：

```sh
taf install taf-blast
taf install taf-blast v2.16.0-r1
taf install taf-blast-v2.16.0-r1
```

卸载和定位：

```sh
taf uninstall taf-blast
taf which taf-blast-v2.16.0-r1
```

### 系统命令

```sh
taf doctor
taf config
taf history
```

`doctor` 用于检查或初始化本地 TAFFISH 环境；`config` 显示当前配置；`history` 显示或清理本地运行历史。

## MCP / AI 集成

`taffish-mcp` 是面向 AI 客户端的保守型 MCP stdio server。它提供结构化的
project、app、Hub、config、history、resource、prompt、验证、编译、摘要、
smoke/trust 检查和包维护规划操作，不暴露 `taf run`、`taf publish`、容器执行
或镜像构建动作。

对于 MCP compile 工具，如果 backend 选择很重要，优先显式传入 `containerBackend`。
如果没有传入这个参数，则会在设置时使用
`TAFFISH_CONTAINER_BACKEND=apptainer|podman|docker`；显式 `containerBackend`
始终具有更高优先级。

MCP 客户端配置示例：

```json
{
  "mcpServers": {
    "taffish": {
      "command": "taffish-mcp",
      "args": []
    }
  }
}
```

这样 AI 客户端可以先通过结构化接口检查 TAFFISH 项目和已安装/已索引 app、搜索本地
index、读取项目资源，验证或编译 `.taf` 源码但不执行它，预览 app/project 编译出来的
shell，并准备安全的项目操作，而不是一开始就依赖非结构化终端文本。

关于 tools、resources、prompts 和安全边界的集中说明，见
[TAFFISH MCP 指南](taffish-mcp.cn.md)。Codex、Claude Code、Cursor、Cline
和通用 MCP 客户端配置示例，见 [在 AI 客户端中使用 TAFFISH MCP](mcp-clients.cn.md)。

## 运行时配置与镜像源

`taf` 提供运行时配置，用于支持镜像源和自定义来源。默认配置路径是：

```text
用户级 = ~/.local/share/taffish/config.toml
系统级 = /opt/taffish/config.toml
```

查看当前生效配置：

```sh
taf config --user
taf config path --user
```

初始化默认 GitHub profile：

```sh
taf config init --user --github
```

初始化中国镜像 profile：

```sh
taf config init --user --china --force
taf update --user
```

中国镜像 profile 是一个简单模板：

```toml
schema_version = "taffish.config/v1"
profile = "china"
language = "en"

[index]
url = "https://gitee.com/taffish-org/taffish-index/raw/main/index/index.json"

[[source.rewrite]]
from = "https://github.com/taffish/"
to = "https://gitee.com/taffish-org/"
enabled = true
```

`[index].url` 控制 `taf update` 默认下载哪个静态 index。`[[source.rewrite]]`
会在 `taf install` clone app 仓库时重写 canonical app 仓库 URL。用户或维护者
也可以用同样机制配置内部 Git 服务，只要镜像端提供兼容仓库、tag 和相同的
TAFFISH index schema。

## 开源与源码构建

Common Lisp 实现已经在 [taffish/taffish](https://github.com/taffish/taffish)
中发布，使用 Apache License 2.0 授权。

源码仓库会构建三个命令行入口：

```text
taf
taffish
taffish-mcp
```

源码构建主要面向贡献者、维护者、打包者，以及需要检查或修改实现的用户。普通用户
仍然建议使用标准安装器安装预构建二进制。

源码侧常用文档：

- [从源码构建](https://github.com/taffish/taffish/blob/main/docs/dev/zh-CN/build-from-source.md)
- [源码树开发文档](https://github.com/taffish/taffish/tree/main/docs/dev/zh-CN)
- [贡献指南](https://github.com/taffish/taffish/blob/main/CONTRIBUTING.md)
- [安全策略](https://github.com/taffish/taffish/blob/main/SECURITY.md)

官方 release 资产目前覆盖部分 macOS 和 Linux 平台；只要依赖可用，任何受到
SBCL 支持的平台仍可从源码构建。带 tag 的 release 载荷提供 `SHA256SUMS`、
`SHA256SUMS.asc` 和 `TAFFISH-RELEASE-KEY.asc`，用于手动 checksum 和 GPG
签名校验。准确的平台矩阵、构建 backend、版本变更和校验流程，应以源码仓库
README 与所选 GitHub Release 为准。

## TAFFISH app 项目结构

一个标准 TAFFISH app 项目大致如下：

```text
my-tool/
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

如果是带 Dockerfile 的 tool 项目，还会有：

```text
docker/
  Dockerfile
.github/
  workflows/
    build-image.yml
```

核心元数据位于 `taffish.toml`：

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

对于 flow：

```toml
[package]
kind = "flow"

[runtime]
pipe = false
command_mode = false
```

对于容器化 tool：

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

对于容器化 app，`[smoke]` 是可发布元数据的一部分。`taf check` 会校验它并拒绝默认
`TODO` 占位；真正的 smoke checks 由 Hub/index 自动化在镜像可用后对新版本执行。

对于 flow 依赖：

```toml
[dependencies]
taf-fastqc = "0.12.1-r1"
taf-aligner = ["1.0.0-r1", "1.0.0-r2"]
```

对于上游软件来源：

```toml
[upstream]
name = "CD-HIT"
type = "github"
homepage = "https://github.com/weizhongli/cdhit"
repository = "weizhongli/cdhit"
version = "4.8.1"
license = "GPL-2.0"
doi = "10.1093/bioinformatics/bts565"
pmid = "23060610"
```

`[upstream]` 是可选的。它描述被包装软件的原始来源，不是 TAFFISH app 自己的仓库来源。

## 推荐开发流程

一个 app 的常见生命周期：

```sh
taf new my-tool --tool --docker
cd my-tool
```

编辑：

```text
taffish.toml
src/main.taf
docs/help.md
docker/Dockerfile
```

本地检查：

```sh
taf check
```

如果是容器化 app，需要先替换默认 `[smoke]` 占位。`taf check` 不会在本地运行 smoke
命令，它只校验 Hub 自动化后续运行 smoke 所需的元数据是否足够。

如果是容器化 app，先构建本地镜像，并明确后端：

```sh
taf build --image --backend docker
```

本地试运行：

```sh
taf run --backend docker -- --help
```

构建：

```sh
taf build
```

如果想一次性构建镜像和 command wrapper，可以使用：

```sh
taf build --all --backend docker
```

发布前预览：

```sh
taf publish --release --dry-run
```

带 release notes 发布：

```sh
taf publish --release --yes --build
```

发布前应编辑被 ignore 的 `release.md` 草稿。它的第一行会成为 publish message，
整个文件会成为 GitHub Release notes。

发布完成后，GitHub 上的 app 仓库会有 release tag，例如：

```text
v0.1.0-r1
```

TAFFISH Hub 的索引自动化会扫描这个 tag，并把 app 纳入 index。用户运行 `taf update` 后即可安装。

## Tool 与 Flow 的区别

### Tool app

Tool app 通常包装一个具体软件，例如 BLAST、BWA、CD-HIT。它的特点是：

- 可以绑定容器镜像。
- 通常 `pipe = true`。
- 通常 `command_mode = true`。
- 用户安装后得到一个版本化命令。

### Flow app

Flow app 通常组合多个 taf app，描述一个分析流程。它的特点是：

- 通常使用 `<taffish>` 标签。
- 可以通过 `[[taf: ...]]` 调用其他 taf app。
- 需要在 `[dependencies]` 中声明依赖。
- 通常 `pipe = false`。
- 通常 `command_mode = false`。

## 当前边界

需要明确几件事：

- TAFFISH 不是通用编程语言，它是一种“编译到 shell 的 app 描述语言”。
- TAFFISH 不直接替代 Docker、Podman、Apptainer，而是选择和调用它们。
- 官方 Hub 当前通过 `taffish` GitHub 组织发布，不是开放自助发布平台。
- TAFFISH Hub 不负责构建每个 app 的镜像；镜像构建由各 app 仓库自己的 GitHub Actions 完成。
- 运行时镜像配置可以重写 index 和 app 仓库访问路径，但不会自动镜像容器 registry。
- 公开的 `taffish/taffish` 仓库现在是本地 CLI/编译器开源源码仓库；本 docs 仓库继续聚焦用户、app 作者、Hub 和 index 文档。
