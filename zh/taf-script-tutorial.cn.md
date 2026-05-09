# TAF 脚本教程

[English](../en/taf-script-tutorial.en.md) | [中文](taf-script-tutorial.cn.md)

这份教程面向 TAFFISH app 作者，介绍实际编写 `.taf` 脚本时会用到的语法。
它不是 TAFFISH 编译器源码开发指南。

## 目录

- [基本模型](#基本模型)
- [最小脚本](#最小脚本)
- [`ARGS` 与 `RUN`](#args-与-run)
- [必需参数](#必需参数)
- [默认值与 flag](#默认值与-flag)
- [位置参数](#位置参数)
- [参数引用](#参数引用)
- [块参数](#块参数)
- [内置参数](#内置参数)
- [容器标签](#容器标签)
- [Tool app wrapper](#tool-app-wrapper)
- [带依赖的 Flow app](#带依赖的-flow-app)
- [帮助文本](#帮助文本)
- [检查、运行、构建](#检查运行构建)
- [常见模式](#常见模式)

## 基本模型

`.taf` 文件接近 shell，但多了一层很小的结构：

- `ARGS` 声明用户可传入的参数。
- `RUN` 里写接近 shell 的命令。
- `::name::` 插入解析后的参数。
- `<container:...>` 通过容器后端运行命令。
- `<taffish>` 让 flow 可以用 `[[taf: ...]]` 调用其他 TAFFISH app。

TAFFISH 会把这些内容编译成 shell。`taf run` 运行编译后的 shell。`taf build`
把项目冻结成版本化命令 wrapper。

## 最小脚本

创建 `hello.taf`：

```taf
<shell>
echo "hello from TAFFISH"
```

编译它：

```sh
taffish hello.taf
```

在项目中运行当前 app：

```sh
taf run
```

如果第一条有效行是 `<shell>` 这样的子标签，TAFFISH 会自动补上 `RUN`。

## `ARGS` 与 `RUN`

用 `ARGS` 定义参数，然后在 `RUN` 中引用：

```taf
ARGS
<name>
  world

RUN
<shell>
echo "hello, ::name::"
```

运行：

```sh
taf run -- --name Alice
```

`<name>` 下面的正文是默认值。用户传入参数时会覆盖默认值。

## 必需参数

用 `!` 前缀声明必需参数：

```taf
ARGS
<!(--/-i)input>
<(--/-o)output>
  result.txt

RUN
<shell>
my-tool --input ::input:: --output ::output::
```

这表示：

```text
--input / -i     必需
--output / -o    可选，默认 result.txt
```

## 默认值与 flag

短默认值可以写在参数 DSL 内部：

```taf
ARGS
<(--/-t)threads=4>
<(--/-v)verbose?>

RUN
<shell>
echo "threads: ::threads::"
echo "verbose: ::verbose::"
```

`?` 表示布尔 flag。用户提供 `--verbose` 时为 true，否则为 false。

## 位置参数

用 `$1`、`$2` 等表示位置参数：

```taf
ARGS
<!$1>
<$2>
  result.txt

RUN
<shell>
cp ::$1:: ::$2::
```

公开 app 命令通常用命名参数更清楚，但位置参数适合小型 wrapper。

## 参数引用

在 `RUN` 中使用 `::name::`：

```taf
<shell>
echo "::sample::"
```

在 `ARGS` 默认值正文中，普通引用也优先使用 `::name::`：

```taf
ARGS
<sample>
  SRR000001
<prefix>
  out/::sample::
```

`@name` 和 `@{name}` 也可以用于默认值表达式。当引用后面紧跟字母、数字、
`_` 或 `-` 时，使用 `@{name}` 避免边界歧义：

```taf
ARGS
<sample>
  demo
<prefix>
  out-@{sample}
```

TAFFISH 只解释明确支持的转义形式。大多数反斜杠序列会按普通文本保留。

## 块参数

`(@:)name` 会创建一个块参数。它适合把一个分析步骤需要的一整段参数封装起来：

```taf
ARGS
<!(--/-q)query>
<(--/-d)db>
  nt

<(@:)blast-step>
  --query ::query::
  --db ::db::
  --outfmt ::(--/-f)outfmt=6::
  ::(--/-e)extra=::

RUN
<taffish>
[[taf: taf-blast-v2.16.0-r1 ::(@:)blast-step::]]
```

这样可以让 flow 主逻辑保持清楚，同时仍然暴露关键底层参数。

## 内置参数

常见内置参数：

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

对于要转发所有上游参数的 tool wrapper，`::*ARGV*::` 很常见：

```taf
<taf-app:container:ghcr.io/taffish/augustus:3.5.0-r1>
augustus ::*ARGV*::
```

## 容器标签

泛化容器后端：

```taf
<container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --help
```

app 层容器 wrapper：

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool ::*ARGV*::
```

编译时，泛化容器标签会变成 Docker、Podman 或 Apptainer 之类的后端命令。用简化的
Docker/Podman 形式表示：

```sh
podman run --rm -i \
  -w "$PWD" \
  -v "$HOME:$HOME" \
  -v "$PWD:$PWD" \
  ghcr.io/taffish/my-tool:0.1.0-r1 \
  my-tool ::*ARGV*::
```

这种自动工作目录挂载是本地输入输出路径能够自然工作的原因。它也意味着容器化 app
应该从项目目录或数据目录运行，不要从 `/usr/bin` 这类宿主机系统目录运行，因为挂载
可能遮住镜像自己的 `/usr/bin`。

本地测试时强制后端：

```sh
taf run --backend docker -- --help
taf run --backend podman -- --help
taf run --backend apptainer -- --help
```

镜像构建目前使用 Docker 或 Podman：

```sh
taf build --image --backend docker
taf build --image --backend podman
```

本地构建镜像和本地运行最好使用同一个后端，否则运行时可能找不到刚构建的镜像。

## Tool app wrapper

一个简单 tool app 通常尽量保留上游 CLI：

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool ::*ARGV*::
```

这意味着：

```sh
taf-my-tool -- --help
taf-my-tool -- --input sample.fa --output result.txt
```

除非确定生成的运行环境会经过支持 shell builtin 的 shell，否则不要随便添加
`exec` 这类 shell builtin。在容器命令 wrapper 中，`exec` 可能被当作可执行文件名，
从而导致运行失败。

## 带依赖的 Flow app

flow app 调用精确 app 版本：

```taf
ARGS
<!(--/-i)input>
<(--/-o)outdir>
  qc

RUN
<taffish>
mkdir -p ::outdir::
[[taf: taf-fastqc-v0.12.1-r1 ::input:: --outdir ::outdir::]]
[[taf: taf-multiqc-v1.19-r1 ::outdir::]]
```

在 `taffish.toml` 中声明依赖：

```toml
[dependencies]
taf-fastqc = "0.12.1-r1"
taf-multiqc = "1.19-r1"
```

如果一个 flow 需要同一个 app 的多个版本：

```toml
[dependencies]
taf-x = ["0.1.0-r1", "0.1.0-r2"]
```

数组表示所有列出的版本都需要安装，不表示可选版本。

## 帮助文本

每个 app 都应该包含 `docs/help.md`。构建后的命令在下面这种情况下会输出这个文本：

```sh
taf-my-tool --help
```

帮助文本是普通终端输出，不是渲染后的 Markdown。即使不渲染 Markdown，也应该可读。

## 检查、运行、构建

在 app 项目中：

```sh
taf check
taf run -- --help
taf build
```

容器化 app 开发时：

```sh
taf build --image --backend docker
taf run --backend docker -- --help
taf build
```

最后的 `taf build` 会写入：

```text
target/taf-my-tool-v0.1.0-r1
target/.taf-my-tool-v0.1.0-r1/
```

生成的命令使用 `target/` 里的冻结快照。

## 常见模式

把所有参数转发给上游：

```taf
<taf-app:container:ghcr.io/taffish/tool:1.0.0-r1>
tool ::*ARGV*::
```

暴露一个稳定的小接口：

```taf
ARGS
<!(--/-i)input>
<(--/-o)output>
  result.txt
<(--/-t)threads>
  4

RUN
<taf-app:container:ghcr.io/taffish/tool:1.0.0-r1>
tool --input ::input:: --output ::output:: --threads ::threads::
```

通过 `taf run` 或构建后的命令传递 option-like 参数时，使用 `--`：

```sh
taf run -- --input sample.fa
taf-my-tool -- --help
```
