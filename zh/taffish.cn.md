# 什么是 TAFFISH

TAFFISH 是一个面向生信工具和流程的轻量级命令交付系统。它目前有三个本地入口：

- `taffish`：TAFFISH 语言编译器，把 `.taf` 文件编译成 POSIX shell 脚本。
- `taf`：面向开发者和用户的 CLI，负责创建项目、检查项目、构建命令、发布 app，以及从 TAFFISH Hub 安装 app。
- `taffish-mcp`：一个保守的 stdio MCP server，用于向 AI 客户端暴露安全的 TAFFISH tools、resources 和 prompts。

换句话说，`.taf` 是描述“这个工具或流程应该如何运行”的源代码；`taffish` 负责把它变成 shell；`taf` 负责把这种代码组织成可发布、可索引、可安装的 TAFFISH app；`taffish-mcp` 则让 AI 客户端可以通过结构化接口检查 TAFFISH 项目和 Hub 状态。

## 目录

- [设计目标](#设计目标)
- [安装](#安装)
- [`taffish` 编译器](#taffish-编译器)
- [`.taf` 文件结构](#taf-文件结构)
- [参数语法](#参数语法)
- [内置运行标签](#内置运行标签)
- [`taf` CLI](#taf-cli)
- [MCP / AI 集成](#mcp--ai-集成)
- [运行时配置与镜像源](#运行时配置与镜像源)
- [TAFFISH app 项目结构](#taffish-app-项目结构)
- [推荐开发流程](#推荐开发流程)
- [Tool 与 Flow 的区别](#tool-与-flow-的区别)
- [当前边界](#当前边界)

## 设计目标

TAFFISH 的目标不是替代 shell、Docker、Conda 或 workflow engine，而是把它们放进一个更稳定的 app 交付格式中：

- 每个 app 有明确的名称、版本、release 和命令名。
- 每个 app 的运行逻辑写在 `.taf` 文件里。
- 每个 app 可以选择绑定容器镜像。
- 每个 app 可以被构建成版本化命令，例如 `taf-blast-v2.16.0-r1`。
- 每个 app 可以被 TAFFISH Hub 索引，并通过 `taf update` / `taf install` 安装。
- flow app 可以声明并调用其他 taf app，形成可追踪的流程依赖。

## 安装

当前公开仓库提供的是二进制发布版本。用户安装推荐使用 `taffish/taffish`
仓库中的权威安装器：

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --user
```

系统安装：

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sudo sh -s -- --system
```

固定安装某个版本：

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --version 0.5.0 --user
```

中国大陆用户可以使用 Gitee 安装器，减少安装阶段对 GitHub raw content 的依赖，
并初始化中国镜像配置：

```sh
curl -fsSL https://gitee.com/taffish-org/taffish/raw/main/install/install-taffish.gitee.sh | sh -s -- --user
```

安装后会得到三个命令：

```text
taffish
taf
taffish-mcp
```

TAFFISH `0.5.0` 还会把 shell 自动补全文件和 Vim 语法文件安装到 TAFFISH home。
用户级安装时通常位于 `~/.local/share/taffish/share/completions` 和
`~/.local/share/taffish/share/vim`。

默认用户路径：

```text
bin  = ~/.local/bin
home = ~/.local/share/taffish
```

默认系统路径：

```text
bin  = /usr/local/bin
home = /opt/taffish
```

安装器默认会尝试运行 `taf doctor --init` 和 `taf update`，用于初始化本地环境和索引。
如果网络不可用，安装器会给出警告，但不会回滚已经安装的二进制文件。

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
<name>
world

RUN
<shell>
echo "hello, ::name::"
```

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
- `?` 表示布尔 flag，出现为 true，不出现为 false。
- `=` 表示默认值。
- `$1`、`$2` 表示位置参数。
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
<(@:)blast-step>
  --db ::(--/-d)db=nt::
  --query ::!(--/-q)query::
  --outfmt ::(--/-of)outfmt=6::
  ::(--/-e)extra=::

RUN
<taffish>
[[taf: taf-blast-v2.16.0-r1 ::(@:)blast-step::]]
```

这里 `blast-step` 不是一个普通字符串参数，而是一整段可组合的参数块。它的默认块中还可以继续嵌入普通参数 DSL，例如 `db`、`query`、`outfmt` 和 `extra`。TAFFISH 会从块默认值中抽取这些内层参数，使开发者可以：

- 对外暴露领域化参数，例如 `query`、`db`、`outfmt`。
- 把底层工具需要的一组参数封装成一个步骤参数。
- 在 flow 中保持 `[[taf: ...]]` 调用简洁。
- 给高级用户保留 `extra` 之类的扩展入口。

从命令行层面看，`(@:)blast-step` 对应 `@blast-step:` 这个 slot。普通用户通常不需要直接使用 slot；app 作者更多是用它在 `.taf` 内部组织默认步骤、领域参数和额外参数。

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
[[taf: taf-fastqc-v0.12.1-r1 sample.fq]]
[[taf: taf-multiqc-v1.19-r1 .]]
```

嵌入语法是：

```text
[[taf: taf-command args...]]
```

编译时，TAFFISH 会把这些 taf app 先编译成临时 shell step，然后在当前流程中执行。这个机制让 flow 可以组合已有工具，并且保留明确的依赖关系。

简化后的编译形态：

```sh
taffish_tmpdir=$(mktemp -d)
taf-fastqc-v0.12.1-r1 --compile sample.fq > "$taffish_tmpdir/step-1.sh"
taf-multiqc-v1.19-r1 --compile . > "$taffish_tmpdir/step-2.sh"
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
taf new <APP-NAME>
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
taf search <keyword>
taf info <app-or-command>
taf install <app-or-command>
taf uninstall <app-or-command>
taf list
taf which <taf-command>
```

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

TAFFISH `0.4.0` 新增了 `taffish-mcp`，这是一个保守的 stdio MCP server，面向
AI 客户端暴露 TAFFISH 能力。TAFFISH `0.5.0` 进一步为它增加了只读 TAF
源码/文件编译器辅助工具。当前 MCP 接口提供相对安全的 project、Hub、config、
history、resource、prompt、验证、编译和摘要操作，不暴露 `taf run`、`taf publish`
或镜像构建动作。

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

这样 AI 客户端可以先通过结构化接口检查 TAFFISH 项目、搜索本地 index、读取项目资源，
验证或编译 `.taf` 源码但不执行它，并准备安全的项目操作，而不是一开始就依赖非结构化终端文本。

关于 tools、resources、prompts 和安全边界的集中说明，见
[TAFFISH MCP 指南](taffish-mcp.cn.md)。Codex、Claude Code、Cursor、Cline
和通用 MCP 客户端配置示例，见 [在 AI 客户端中使用 TAFFISH MCP](mcp-clients.cn.md)。

## 运行时配置与镜像源

从 TAFFISH `0.2.0` 开始，`taf` 提供运行时配置，用于支持镜像源和自定义来源。当前推荐版本是 `0.5.0`。默认配置路径是：

```text
用户级 = ~/.local/share/taffish/config.toml
系统级 = /opt/taffish/config.toml
```

查看当前生效配置：

```sh
taf config
taf config path
```

初始化默认 GitHub profile：

```sh
taf config init --github
```

初始化中国镜像 profile：

```sh
taf config init --china --force
taf update
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
```

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

Tool app 通常包装一个具体软件，例如 BLAST、BWA、CD-HIT。它的特点是：

- 可以绑定容器镜像。
- 通常 `pipe = true`。
- 通常 `command_mode = true`。
- 用户安装后得到一个版本化命令。

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
- `taffish` 编译器当前公开仓库以二进制发布为主，核心源码暂未开源。
