# TAFFISH 快速开始

[English](../en/quick-start.en.md) | [中文](quick-start.cn.md)

从[安装 TAFFISH](#安装-taffish) 开始，运行你的第一个 app：Augustus。你不需要编写
`.taf` 脚本。安装这个 app 需要 Git；运行它需要 Docker、Podman 或 Apptainer，任选一个可用的即可。

**版本说明：** 本文适用于正式发布的 TAFFISH 0.12.0，下方在线安装器提供此版本。
已经取得平台包时，可用[本地与离线安装方式](#本地与离线安装)。

## 目录

- [你会安装什么](#你会安装什么)
- [安装 TAFFISH](#安装-taffish)
  - [设置命令路径](#设置命令路径)
  - [首次运行 Augustus](#首次运行-augustus)
  - [其他安装方式](#其他安装方式)
  - [本地与离线安装](#本地与离线安装)
- [选择用户级或系统级 Scope](#选择用户级或系统级-scope)
- [检查环境](#检查环境)
- [更新 Hub 索引](#更新-hub-索引)
- [查找 app](#查找-app)
- [安装 app](#安装-app)
- [运行已安装 app](#运行已安装-app)
- [维护已安装 app](#维护已安装-app)
- [列出、定位和卸载 app](#列出定位和卸载-app)
- [容器后端](#容器后端)
- [运行时配置与镜像源](#运行时配置与镜像源)
- [Shell 自动补全和 Vim 文件](#shell-自动补全和-vim-文件)
- [网络说明](#网络说明)
- [查看 Flow 路线](#查看-flow-路线)
- [继续阅读](#继续阅读)

## 你会安装什么

用 `taf` 从 TAFFISH Hub 查找和安装 app。安装后，每个 app 都有自己的 shell 命令，
例如 `taf-augustus`。安装器还提供 `taffish`（把 `.taf` 脚本编译成 shell）和
`taffish-mcp`（供 AI 客户端使用）；运行 Augustus 不需要手动调用这两个命令。

## 安装 TAFFISH

安装到自己的账号，不需要 `sudo`。若要在服务器上共享安装、固定版本或改用
Gitee 镜像，请看[其他安装方式](#其他安装方式)。

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --user
```

### 设置命令路径

用户级安装将核心命令和 app 命令放在两个目录中。安装器不会修改 shell 配置，
请先在当前终端把这两个目录加入 `PATH`：

```sh
export PATH="$HOME/.local/bin:$HOME/.local/share/taffish/bin:$PATH"
```

这条 `export` 只影响当前 shell 及其子进程，不会为今后的终端保存设置。要持久生效，
把同一行加入 `~/.zshrc`（Zsh）或 `~/.bashrc`（交互式 Bash）。Bash 登录 shell 应改放
`~/.bash_profile`，或让该文件加载 `~/.bashrc`。新开终端后，检查实际使用的版本：

```sh
taf --version
taffish --version
taffish-mcp --version
```

下方 app 命令也适用于 0.11.0。安装器选项随版本变化，应查看所选安装器的 `--help`。

### 首次运行 Augustus

刷新本地 Hub 索引，查找 Augustus，查看包信息，然后安装索引推荐的版本：

```sh
taf update --user
taf search --user augustus
taf info --user augustus
taf install --user augustus
```

在平常存放分析文件的工作目录中运行：

```sh
taf-augustus -- --help
```

看到 **AUGUSTUS 自己的帮助**，就说明工具已经运行。第一个 `--` 让外层命令把
`--help` 交给容器中的 AUGUSTUS。若本机还没有镜像，容器后端需要能访问镜像仓库。
如果容器没有启动，请看[容器后端](#容器后端)。

### 其他安装方式

系统级安装，适合共享服务器：

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sudo sh -s -- --system
```

如果系统级安装会配合 Apptainer 给多用户运行 app，需要注意：安装后的 `taf-*`
命令可以对所有用户可见，但首次运行时的 SIF 镜像缓存仍然需要一个可写镜像目录。
多用户部署前建议先阅读
[Apptainer SIF 缓存权限说明](troubleshooting.cn.md#apptainer-提示-no-writable-apptainer-image-directory-found)。

如需固定安装 0.12.0，可使用该标签内的安装器。以下命令要求本机有 `gpg` 和 `gpgv`：

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/v0.12.0/install/install-taffish.sh | sh -s -- --version 0.12.0 --user --verify gpg
```

如需用默认安装器选择其他已发布版本，可传入 `--version X.Y.Z`。

发布版本下载、源码构建和校验说明见
[taffish/taffish README](https://github.com/taffish/taffish)。

中国大陆用户访问 GitHub raw 可能较慢或被阻断。Gitee 安装器会从 Gitee 镜像下载
文件，并在没有配置文件时自动初始化中国镜像配置：

```sh
curl -fsSL https://gitee.com/taffish-org/taffish/raw/main/install/install-taffish.gitee.sh | sh -s -- --user
```

如果需要把已有配置替换为 Gitee/中国镜像 profile：

```sh
curl -fsSL https://gitee.com/taffish-org/taffish/raw/main/install/install-taffish.gitee.sh | sh -s -- --user --force-config
```

**仅对 0.11.0：** 在 Apple 芯片的 Mac 上，如果未登录 Gitee 时下载单个二进制文件遇到
HTTP 403，可将 [v0.11.0 标签归档](https://gitee.com/taffish-org/taffish/repository/archive/v0.11.0.tar.gz)
保存为 `taffish-v0.11.0.tar.gz`，并将[同一标签的安装器](https://gitee.com/taffish-org/taffish/raw/v0.11.0/install/install-taffish.gitee.sh)
下载到同一目录。确认本机有 GPG 后执行：

```sh
sh install-taffish.gitee.sh --user --archive taffish-v0.11.0.tar.gz --verify gpg
```

完成用户级安装后，回到[设置命令路径](#设置命令路径)继续。

<a id="安装本地-0120-候选"></a>

### 本地与离线安装

正式 0.12.0 平台包同时包含 `taf`、`taffish`、`taffish-mcp`，已内置 Han Lisp 0.3.0
和所需 Lisp 运行时，用户无需另装 Lisp。

| 平台 | 完整平台包 | 解包工具 |
| --- | --- | --- |
| Apple Silicon，macOS 15 或更新版本 | `taffish-0.12.0-darwin-arm64.tar.xz` | 系统 `/usr/bin/tar` |
| 基于 glibc 的 x86-64 Linux | `taffish-0.12.0-linux-amd64.tar.gz` | `tar` 和 `gzip` |

macOS 包不再需要 Homebrew 或 `zstd`。但从 0.11.0 升级时，请保留旧 `zstd` 直到
升级完成：安装器替换前需要运行旧命令来识别它们。

通过可信渠道取得匹配的 0.12.0 安装器和完整平台包。把 `install-taffish.sh`、平台包、
`ARCHIVE-SHA256SUMS`、`ARCHIVE-SHA256SUMS.asc` 和 `TAFFISH-RELEASE-KEY.asc`
放在同一目录。确认 `gpg` 和 `gpgv` 可用后，在该目录运行：

```sh
sh install-taffish.sh --local --version 0.12.0 --user --verify gpg
```

`--local` 从本地选择匹配的平台包，不下载载荷，也跳过安装后的索引更新。
`--verify gpg` 要求归档清单和程序清单的签名均有效，并分别在解包和安装程序前核对摘要；
它不认证安装脚本本身。安装后设置 `PATH`，需要获取 Hub 索引时再主动运行 `taf update --user`。

Linux 安装会显示随包运行时条款并要求确认。安装前先阅读；无人值守安装须显式传入
`--accept-runtime-license`，见[管理员说明](system-administration.cn.md#本地安装与运行时条款)。
macOS 无需确认运行时条款。

0.12.0 Gitee 安装器会自动合并 macOS 平台包的 `.part-*` 小文件，再校验完整归档；
GitHub 和 Linux 使用完整归档。本地安装也需要完整归档，不能直接使用 `.parts` 清单。
改名包、手工合并和校验细节见[安装指南](https://github.com/taffish/taffish/blob/v0.12.0/install/README-CN.md)。
设置 `PATH` 后，确认 `taf --version`、`taffish --version` 和 `taffish-mcp --version`
均显示 0.12.0。

## 选择用户级或系统级 Scope

本快速开始后续使用 user scope。支持 scope 的 `taf` 命令即使在核心二进制系统级安装后，
每次调用时仍默认使用 `--user`。

管理员维护共享状态时必须始终显式使用 `--system`：

```sh
sudo taf update --system
taf search --system fastqc
sudo taf install --system fastqc
taf list --system
```

在部署共享 app、全局 shell/编辑器集成或 HPC 后端前，请先阅读
[TAFFISH 系统管理员指南](system-administration.cn.md)。

## 检查环境

运行：

```sh
taf doctor --user
```

当前安装器通常会在安装时初始化标准 TAFFISH 目录。如果是手动安装或修复安装，可以初始化缺失目录：

```sh
taf doctor --init --user
```

`taf doctor` 会检查路径和常见命令，例如 `git`、`docker`、`podman`、
`apptainer` 和 `taffish`。不是所有可选工具都必须安装。如果你只使用 Docker，
就不需要同时安装 Podman。

在 0.11.0 中，除非显式传入 `--init`，否则 `taf doctor` 只读。检查本地索引缓存
用 `taf doctor index`；检查已安装的应用可用 `taf doctor hub augustus`。用
`taf config language` 查看语言设置；只想临时看中文帮助可运行
`TAFFISH_LANGUAGE=zh taf --help`。中英文均内置，下载镜像源不决定界面语言。

## 更新 Hub 索引

本地 `taf` 命令从本地缓存的 TAFFISH Hub index 安装 app。第一次使用前先更新：

```sh
taf update --user
```

它默认下载官方静态 index，并写入当前 TAFFISH home。

如果网络不稳定，或者要测试其他 index：

```sh
taf update --url INDEX_URL
```

也可以用 `TAFFISH_INDEX_URL` 覆盖默认 index URL。

## 查找 app

搜索：

```sh
taf search augustus
taf search blast
```

查看 app 信息：

```sh
taf info augustus
taf info taf-augustus
```

列出本地 index 缓存中的在线 app：

```sh
taf list --online
```

列出本地已经安装的 app：

```sh
taf list
```

## 安装 app

安装本地索引缓存当前推荐的版本：

```sh
taf install augustus
```

也可以使用命令名：

```sh
taf install taf-augustus
```

安装精确版本：

```sh
taf install augustus 3.5.0-r1
taf install taf-augustus-v3.5.0-r1
```

只预览安装计划，不改动文件：

```sh
taf install --dry-run augustus
```

如果目标版本声明了依赖，`taf install` 会先安装依赖。

安装尚未发布到公开 Hub 的私有/本地 TAFFISH app 项目：

```sh
taf install --from /path/to/my-private-tool
taf list
taf which taf-my-private-tool
```

`taf install --from` 会读取本地项目的 `taffish.toml`、检查项目、把当前工作树
复制到选定的 TAFFISH home、构建带版本的 command wrapper，并把来源记录为
`[local-project] <PROJECT-ROOT>`。路径可以是项目根目录，也可以是项目内任意
子目录。这个模式不需要先运行 `taf update`，也不会自动安装依赖。

## 运行已安装 app

TAFFISH app 命令就是普通 shell 命令。例如：

```sh
taf-augustus -- --help
taf-augustus -- --species=help
taf-augustus -- --species=human genome.fa
```

开头的 `--` 让 wrapper 不再处理 `--help` 这类选项。上面三个示例都是在容器中
运行 `augustus`，并把所列选项传给它。最后一个示例需要当前工作目录中有名为
`genome.fa` 的 FASTA 文件。

这个 app 也支持命令模式：`taf-augustus echo 1` 会在容器中运行 `echo 1`，
既不会在宿主机上执行 `echo`，也不会把 `echo 1` 当作参数传给 AUGUSTUS。

安装后的命令通常包括：

```text
taf-<name>                  非版本化别名
taf-<name>-v<version>-r<N>   精确版本命令
```

非版本化别名指向该命令**本地已安装**的最新版本，不一定等于 index 推荐版本。

## 维护已安装 app

本节命令维护从 Hub index 安装的 app，不会升级 `taf`、`taffish` 或
`taffish-mcp` 核心二进制。升级核心时应使用相同 scope 重新运行安装器，并指定
`--version`。

预览安装 index 中的全部 app：

```sh
taf install --all
taf install --all --tools
taf install --all --flows
```

`taf install --all` 默认只是 dry-run。加上 `--yes` 才会真正安装计划中的 app：

```sh
taf install --all --tools --yes
```

检查本地已安装 Hub app 是否有更新版本：

```sh
taf update
taf outdated
taf outdated --json
```

预览或执行升级：

```sh
taf upgrade
taf upgrade --yes
taf upgrade taf-augustus --yes --prune-old
```

`taf upgrade` 默认也是 dry-run。它会用本地安装元数据和当前本地 index 比较，
所以建议先运行 `taf update`。通过 `taf install --from` 安装的本地/私有 app
不会被公开 index 中的版本静默替换。

在 0.11.0 中，批量安装和升级以本地 index 的默认推荐 `latest` 为目标，而不是
最近发布的 tag；不会自动降级。`prune` 只在该推荐版本已安装、同源且验证通过时
清理能可靠判定更旧的版本。推荐目标缺失或安装证据不充分时会保护相应的包，而不是
按本地版本号猜测。请先预览再确认：

```sh
taf prune
taf prune --yes
taf prune --kind flow --yes
```

`taf uninstall`、`taf upgrade --prune-old` 和 `taf prune` 只删除 TAFFISH app
源码与 wrapper 文件，不会删除共享的 Docker、Podman、Apptainer 或 SIF 镜像缓存。

## 列出、定位和卸载 app

列出本地安装：

```sh
taf list
```

查看安装路径：

```sh
taf which taf-augustus
taf which taf-augustus-v3.5.0-r1
```

卸载：

```sh
taf uninstall taf-augustus
taf uninstall taf-augustus-v3.5.0-r1
```

当本地有多个版本时，建议使用精确版本命令。

## 容器后端

大多数生信 tool app 会在容器中运行。TAFFISH 可以根据 app 和本地环境使用
Docker、Podman 或 Apptainer。

常见检查：

```sh
docker run --rm hello-world
podman run --rm hello-world
apptainer --version
```

你只需要安装自己打算使用的后端。在 HPC 或共享 Linux 服务器上，Apptainer
通常更合适；在个人电脑上，Docker 或 Podman 通常更简单。

如果是系统级安装并使用 Apptainer，需要区分 app 命令可见性和 SIF 镜像缓存权限。
普通用户看到 `no writable apptainer image directory found` 时，见
[Apptainer SIF 缓存 FAQ](troubleshooting.cn.md#apptainer-提示-no-writable-apptainer-image-directory-found)。

对于已经安装好的 `taf-*` 命令，或者直接使用 `taffish` 编译时，可以设置
`TAFFISH_CONTAINER_BACKEND=apptainer|podman|docker`，在运行时强制通用
`<container:...>` 标签使用指定后端：

```sh
TAFFISH_CONTAINER_BACKEND=podman taf-augustus-v3.5.0-r1 -- --help
TAFFISH_CONTAINER_BACKEND=podman taf-augustus-v3.5.0-r1 --compile --help
```

这个环境变量不会覆盖显式的 `<docker:...>`、`<podman:...>` 或
`<apptainer:...>` 标签。本地项目运行时，`taf run --backend ...` 的优先级
高于这个环境变量。

TAFFISH `0.9.0` 还支持 backend-specific container runtime arguments。app
自身运行需求应使用结构化 `.taf` 标签参数，例如某个 app 本身需要 GPU：

```taf
<container:ghcr.io/taffish/my-gpu-tool:1.0.0-r1$@[docker: --gpus all][podman: --device nvidia.com/gpu=all][apptainer: --nv]>
  my-gpu-tool --help
```

机器、站点或单次运行差异则使用运行时环境变量，不应写进 app 实现：

```sh
TAFFISH_DOCKER_RUN_ARGS="--platform linux/amd64" taf-my-tool ...
TAFFISH_PODMAN_RUN_ARGS="--platform linux/amd64" taf-my-tool ...
TAFFISH_APPTAINER_RUN_ARGS="--bind /scratch:/scratch" taf-my-tool ...
```

简言之：app 特有运行需求写进 `.taf`；本地执行策略放进环境变量。

如果容器后端在 app 启动前就报错，先检查后端本身。例如：

```sh
podman system df
podman ps -a
podman machine stop
podman machine start
```

如果 Podman 报 `crun: create keyring ... Disk quota exceeded`，见
[Podman machine 或 crun 报错](troubleshooting.cn.md#podman-machine-或-crun-报错)。
这可能是 Linux keyring quota 限制，不一定是文件系统磁盘空间不足。

容器化 app 建议从项目目录、数据目录或临时工作目录运行。不要从 `/usr/bin` 或
`/usr/local/bin` 这类宿主机系统目录运行：TAFFISH 会把当前 workdir 挂载进容器，
这个挂载可能遮住镜像内部的重要目录。

TAFFISH 仍然会使用 app package 中声明的容器镜像，通常是 GHCR。镜像配置目前
覆盖 index 下载和 app 仓库 clone；运行 app 的机器仍然需要能访问对应容器镜像。

## 运行时配置与镜像源

`taf` 提供一个很小的运行时配置文件，用来稳定支持镜像源和自定义来源。

默认配置路径：

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

中国镜像 profile 是一个模板：

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

`[index].url` 控制 `taf update` 从哪里下载静态 index。`[[source.rewrite]]`
会在 `taf install` clone app 仓库时重写 canonical app 仓库 URL。这也让内部镜像
成为可能：同步 app 仓库和 index schema 后，直接编辑 `config.toml`，让 `from`
指向 canonical GitHub 前缀，让 `to` 指向内部 Git 服务前缀。

## Shell 自动补全和 Vim 文件

安装器会把 shell 自动补全文件和 Vim 语法文件复制到 TAFFISH home，但不会自动启用它们。

用户级安装时，补全文件通常位于：

```text
~/.local/share/taffish/share/completions
```

Bash 示例：

```sh
source ~/.local/share/taffish/share/completions/bash/taf
source ~/.local/share/taffish/share/completions/bash/taffish
```

Zsh 示例：

```sh
fpath=(~/.local/share/taffish/share/completions/zsh $fpath)
autoload -Uz compinit
compinit
```

Fish 示例：

```sh
mkdir -p ~/.config/fish/completions
cp ~/.local/share/taffish/share/completions/fish/taf.fish ~/.config/fish/completions/
cp ~/.local/share/taffish/share/completions/fish/taffish.fish ~/.config/fish/completions/
```

Vim 语法文件通常位于：

```text
~/.local/share/taffish/share/vim
```

将 `syntax/taf.vim` 和 `ftdetect/taf.vim` 复制到对应的 Vim 或 Neovim 用户 runtime。
系统级启用方式见 [系统管理员指南](system-administration.cn.md)。

## 网络说明

如果你的环境访问 GitHub 不稳定：

- 稍后重试 `taf update`；
- 配置系统代理或 Git 代理；
- 初始化中国镜像 profile，或用 `taf update --url <INDEX-URL>` 做一次性 index 覆盖；
- 确认运行 app 的机器可以访问 GHCR 镜像。

## 查看 Flow 路线

如果想看 TAFFISH app 如何组成策划型分析路线，可以打开
[TAFFISH Flow Portal](https://taffish.github.io/flows/)。它链接官方 flow family、路线页面、
Hub 入口和示例报告。

- [TAFFISH Flow Portal](https://taffish.github.io/flows/)
- [RNA-seq Flow Family 门户](https://taffish.github.io/rnaseq-flows/)
- [Yeast 标准报告](https://taffish.github.io/rnaseq-flows/examples/yeast-standard-report/04_reports/rnaseq_report.html)
- [报告解读手册](https://taffish.github.io/rnaseq-flows/examples/yeast-standard-report/04_reports/report_interpretation.html)

RNA-seq 专题站仍然是当前最完整的公开 flow-family 案例。它覆盖参考基因组准备、
基于 Salmon 的表达定量、差异表达、富集分析、可选 alignment/count/QC 证据路线、
无参模式，以及最终报告生成。这些页面展示的是如何把已安装的 `taf-*` 命令组合成
command-level reproducible 的轻量 flow，而不是替代传统 workflow engine。

## 继续阅读

- [什么是 TAFFISH](taffish.cn.md)
- [TAFFISH 系统管理员指南](system-administration.cn.md)
- [TAFFISH app 开发者指南](app-developer-guide.cn.md)
- [容器化 app 最佳实践](container-apps.cn.md)
- [Flow 与依赖指南](flow-dependencies.cn.md)
- [故障排查](troubleshooting.cn.md)
- [从源码构建](https://github.com/taffish/taffish/blob/main/docs/dev/zh-CN/build-from-source.md)
