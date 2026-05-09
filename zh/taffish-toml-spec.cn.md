# `taffish.toml` 规范

`taffish.toml` 是 TAFFISH app 的核心元数据文件。`taf check`、`taf build`、`taf publish`、`taffish-index` 和 TAFFISH Hub 都依赖它理解一个 app。

本文描述当前推荐字段。未列出的字段可能被当前工具忽略，未来也不保证兼容。

## 目录

- [基本原则](#基本原则)
- [`[package]`](#package)
- [`[repository]`](#repository)
- [`[command]`](#command)
- [`[runtime]`](#runtime)
- [`[container]`](#container)
- [`[dependencies]`](#dependencies)
- [`[platform]`](#platform)
- [`[upstream]`](#upstream)
- [Tool 示例](#tool-示例)
- [Flow 示例](#flow-示例)
- [版本与 tag 规则](#版本与-tag-规则)
- [常见错误](#常见错误)

## 基本原则

`taffish.toml` 使用一个小的 TOML 子集：

- section 使用 `[section]`。
- 字符串使用双引号。
- 布尔值使用 `true` / `false`。
- 整数用于 release、CPU 和内存等字段。
- 字符串数组用于多版本依赖等字段。

建议所有 app 都保持字段简洁、稳定、可机器解析。

## `[package]`

必需。

```toml
[package]
name = "my-tool"
kind = "tool"
version = "0.1.0"
release = 1
license = "Apache-2.0"
main = "src/main.taf"
```

字段：

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `name` | string | 是 | package 名称。不能以 `-` 或 `.` 开头，只建议使用字母、数字、`-`、`_`。 |
| `kind` | string | 是 | `tool` 或 `flow`。 |
| `version` | string | 是 | 上游或流程版本。不能为空，不能包含空格或 tab。 |
| `release` | integer | 是 | TAFFISH 打包 release，必须是正整数。 |
| `license` | string | 否 | app 包装层许可证。当前 `taf new` 默认 `Apache-2.0`。 |
| `main` | string | 是 | 主 `.taf` 文件路径，必须是项目内相对路径。 |

`name` 是 package name，不一定等于命令名。命令名由 `[command].name` 定义。

## `[repository]`

必需。

```toml
[repository]
url = "https://github.com/taffish/my-tool"
```

字段：

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `url` | string | 是 | app 自己的 GitHub 仓库 URL。 |

官方 Hub 扫描时会检查该 URL 是否匹配被扫描的仓库。也就是说，`taffish/my-tool`
仓库中的 `repository.url` 应该指向 canonical GitHub 地址
`https://github.com/taffish/my-tool`。普通 package 不应该把这里改成镜像 URL。
TAFFISH `0.3.0` 的镜像支持由本地 `taf` 运行时配置处理，`[[source.rewrite]]`
会在 `taf install` 时重写这个 canonical URL。

## `[command]`

必需。

```toml
[command]
name = "taf-my-tool"
```

字段：

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `name` | string | 是 | app 命令基础名，必须以 `taf-` 开头。 |

构建后的版本化命令格式为：

```text
<command.name>-v<version>-r<release>
```

例如：

```text
taf-my-tool-v0.1.0-r1
```

## `[runtime]`

必需。

```toml
[runtime]
pipe = true
command_mode = true
```

字段：

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `pipe` | boolean | 是 | app 是否适合参与标准输入/输出管道。 |
| `command_mode` | boolean | 是 | app 是否以命令式工具方式运行。 |

推荐：

```toml
# tool app
pipe = true
command_mode = true

# flow app
pipe = false
command_mode = false
```

## `[container]`

可选。用于容器化 app。

```toml
[container]
image = "ghcr.io/taffish/my-tool:0.1.0-r1"
dockerfile = "docker/Dockerfile"
build_platforms = "linux/amd64,linux/arm64"
```

字段：

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `image` | string | 否 | app 使用的容器镜像。 |
| `dockerfile` | string | 否 | Dockerfile 路径，必须是项目内相对路径。 |
| `build_platforms` | string | 否 | 镜像构建平台列表，当前常用 `linux/amd64,linux/arm64`。 |

注意：

- 如果写了 `dockerfile`，文件必须存在。
- 如果写了 `image`，镜像 tag 应匹配 `<version>-r<release>`。
- `taffish-index` 当前导出 `image` 和 `dockerfile`，`build_platforms` 主要用于本地构建和 GitHub Actions。

## `[dependencies]`

可选。主要用于 flow app。

```toml
[dependencies]
taf-dep-tool = "0.1.0-r1"
taf-align = ["1.0.0-r1", "1.1.0-r1"]
```

规则：

- key 必须是 taf command，且以 `taf-` 开头。
- value 可以是字符串，也可以是字符串数组。
- 字符串不能为空。
- 数组不能为空。
- app 不能依赖自己。

语义：

```toml
taf-align = ["1.0.0-r1", "1.1.0-r1"]
```

表示该 flow 需要 `taf-align` 的两个版本都可安装，不表示二选一。

依赖值推荐使用不带前导 `v` 的 version id：

```text
1.0.0-r1
```

## `[platform]`

可选。用于描述平台约束，主要服务 Hub 展示和未来安装决策。

```toml
[platform]
os = "linux,darwin"
arch = "amd64,arm64"
container = "required"
min_cpus = 2
min_memory_mb = 4096
```

字段：

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `os` | string | 否 | 逗号分隔的系统列表。 |
| `arch` | string | 否 | 逗号分隔的架构列表。 |
| `container` | string | 否 | `optional`、`required` 或 `forbidden`。默认 `optional`。 |
| `min_cpus` | integer | 否 | 建议最小 CPU 数，正整数。 |
| `min_memory_mb` | integer | 否 | 建议最小内存，单位 MB，正整数。 |

当前 `container` 的含义：

- `optional`：可容器运行，也可能可本地运行。
- `required`：需要容器环境。
- `forbidden`：不应在容器内运行。

## `[upstream]`

可选。用于描述被包装软件的原始来源。

```toml
[upstream]
name = "CD-HIT"
type = "github"
homepage = "https://github.com/weizhongli/cdhit"
repository = "weizhongli/cdhit"
release_url = "https://github.com/weizhongli/cdhit/releases"
docker_image = "docker.io/example/cdhit:4.8.1"
version = "4.8.1"
license = "GPL-2.0"
citation = "Fu et al. 2012"
doi = "10.1093/bioinformatics/bts565"
pmid = "23060610"
```

支持字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `name` | string | 上游软件名称。 |
| `type` | string | 来源类型，推荐 `official`、`github`、`gitlab`、`archive`、`docker`、`apt`、`conda`、`other`。 |
| `homepage` | string | 上游主页。 |
| `repository` | string | 上游仓库，例如 `weizhongli/cdhit`。 |
| `release_url` | string | 上游发布页。 |
| `docker_image` | string | 上游已有 Docker 镜像。 |
| `version` | string | 被包装的上游版本。 |
| `license` | string | 上游软件许可证。 |
| `citation` | string | 引用信息。 |
| `doi` | string | DOI。 |
| `pmid` | string | PubMed ID。 |

如果 `[upstream]` 不存在，或没有任何有效字段，index 中会省略 `upstream` 字段。

## Tool 示例

```toml
[package]
name = "blast"
kind = "tool"
version = "2.16.0"
release = 1
license = "Apache-2.0"
main = "src/main.taf"

[repository]
url = "https://github.com/taffish/blast"

[command]
name = "taf-blast"

[runtime]
pipe = true
command_mode = true

[container]
image = "ghcr.io/taffish/blast:2.16.0-r1"
dockerfile = "docker/Dockerfile"
build_platforms = "linux/amd64,linux/arm64"

[platform]
os = "linux,darwin"
arch = "amd64,arm64"
container = "required"

[upstream]
name = "BLAST+"
type = "official"
homepage = "https://blast.ncbi.nlm.nih.gov/"
version = "2.16.0"
```

## Flow 示例

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

## 版本与 tag 规则

`version` 与 `release` 组合成 version id：

```text
<version>-r<release>
```

例如：

```text
2.16.0-r1
```

Git tag 必须加前导 `v`：

```text
v2.16.0-r1
```

镜像 tag 推荐不加前导 `v`：

```text
ghcr.io/taffish/blast:2.16.0-r1
```

## 常见错误

常见问题：

- `command.name` 没有以 `taf-` 开头。
- `repository.url` 指向旧组织或错误仓库。
- `release` 写成字符串而不是整数。
- `main` 不是 `.taf` 文件。
- `docs/help.md` 缺失。
- Dockerfile 路径写错。
- 容器镜像 tag 与 `version-release` 不一致。
- flow 使用了 `[[taf: ...]]`，但 `[dependencies]` 缺失或版本不一致。
- 把 `[upstream]` 当成 app 自己的 GitHub 仓库信息。
