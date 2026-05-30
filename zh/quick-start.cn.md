# TAFFISH 快速开始

[English](../en/quick-start.en.md) | [中文](quick-start.cn.md)

这份文档面向只想安装和使用 TAFFISH app 的用户。它不假设你要编写 `.taf`
脚本，也不假设你要维护 app 仓库。

## 目录

- [你会安装什么](#你会安装什么)
- [安装 TAFFISH](#安装-taffish)
- [检查环境](#检查环境)
- [Shell 自动补全和 Vim 文件](#shell-自动补全和-vim-文件)
- [更新 Hub 索引](#更新-hub-索引)
- [查找 app](#查找-app)
- [安装 app](#安装-app)
- [维护已安装 app](#维护已安装-app)
- [运行已安装 app](#运行已安装-app)
- [列出、定位和卸载 app](#列出定位和卸载-app)
- [容器后端](#容器后端)
- [运行时配置与镜像源](#运行时配置与镜像源)
- [网络说明](#网络说明)
- [查看 Flow 路线](#查看-flow-路线)
- [继续阅读](#继续阅读)

## 你会安装什么

TAFFISH 在本地提供三个命令：

```text
taffish     将 .taf 程序编译为 shell
taf         管理 app 项目和 TAFFISH Hub 包
taffish-mcp 向 AI 客户端暴露安全 tools/resources/prompts 和 app/project inspection
```

普通用户主要使用 `taf` 和安装后的 `taf-*` app 命令。安装器也会把 shell
自动补全文件和 Vim 语法文件复制到 TAFFISH home。

典型流程是：

```sh
taf update
taf search augustus
taf info augustus
taf install augustus
taf-augustus -- --help
```

## 安装 TAFFISH

用户级安装，推荐普通用户使用：

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --user
```

系统级安装，适合共享服务器：

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sudo sh -s -- --system
```

如果系统级安装会配合 Apptainer 给多用户运行 app，需要注意：安装后的 `taf-*`
命令可以对所有用户可见，但首次运行时的 SIF 镜像缓存仍然需要一个可写镜像目录。
多用户部署前建议先阅读
[Apptainer SIF 缓存权限说明](troubleshooting.cn.md#apptainer-提示-no-writable-apptainer-image-directory-found)。

固定安装某个版本：

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --version 0.10.0 --user
```

TAFFISH `0.10.0` 是当前公开 release。大多数用户仍然建议使用上面的安装器；源码构建和 release 校验说明见
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

安装后检查：

```sh
taf --version
taffish --version
taffish-mcp --version
```

如果提示找不到 `taf`，把用户安装目录加入 `PATH`：

```sh
export PATH="$HOME/.local/bin:$PATH"
```

把它写入 shell 配置文件后，重新打开终端，或运行 `source ~/.zshrc` /
`source ~/.bashrc`。

## 检查环境

运行：

```sh
taf doctor
```

当前安装器通常会在安装时初始化标准 TAFFISH 目录。如果是手动安装或修复安装，可以初始化缺失目录：

```sh
taf doctor --init
```

`taf doctor` 会检查路径和常见命令，例如 `git`、`docker`、`podman`、
`apptainer` 和 `taffish`。不是所有可选工具都必须安装。如果你只使用 Docker，
就不需要同时安装 Podman。

## Shell 自动补全和 Vim 文件

TAFFISH `0.5.0` 及后续版本会把 shell 自动补全文件和 Vim 语法文件安装到 TAFFISH home。

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

Vim 语法文件通常位于：

```text
~/.local/share/taffish/share/vim
```

## 更新 Hub 索引

本地 `taf` 命令从本地缓存的 TAFFISH Hub index 安装 app。第一次使用前先更新：

```sh
taf update
```

它默认下载官方静态 index，并写入当前 TAFFISH home。

如果网络不稳定，或者要测试其他 index：

```sh
taf update --url <INDEX-URL>
```

也可以用 `TAFFISH_INDEX_URL` 覆盖默认 index URL。

## 运行时配置与镜像源

从 TAFFISH `0.2.0` 开始，`taf` 提供一个很小的运行时配置文件，用来稳定支持镜像源和自定义
来源。当前公开版本是 `0.10.0`。

默认配置路径：

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

安装 index 中的最新版本：

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

## 维护已安装 app

TAFFISH `0.10.0` 增加了面向 Hub app 的保守本地维护命令。

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

移除旧的本地版本，并保留本地已安装的最新版本：

```sh
taf prune
taf prune --yes
taf prune --kind flow --yes
```

`taf uninstall`、`taf upgrade --prune-old` 和 `taf prune` 只删除 TAFFISH app
源码与 wrapper 文件，不会删除共享的 Docker、Podman、Apptainer 或 SIF 镜像缓存。

## 运行已安装 app

TAFFISH app 命令就是普通 shell 命令。例如：

```sh
taf-augustus -- --help
taf-augustus -- --species=help
taf-augustus -- --species=human genome.fa
```

`--` 分隔符表示“后面的参数传给上游工具”。当 `--help` 这类参数可能被
TAFFISH wrapper 自己处理时，使用 `--` 更稳妥。

不要把 TAFFISH tool 命令当成 shell 命令运行器。例如：

```sh
taf-augustus echo 1
```

这不是执行 shell 的 `echo`，而是把 `echo 1` 传给上游 AUGUSTUS。

安装后的命令通常包括：

```text
taf-<name>                  非版本化别名
taf-<name>-v<version>-r<N>   精确版本命令
```

非版本化别名会指向本地已安装的最新版本。

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
TAFFISH_CONTAINER_BACKEND=podman taf-augustus-v3.5.0-r1 --compile -- --help
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
- [TAFFISH app 开发者指南](app-developer-guide.cn.md)
- [容器化 app 最佳实践](container-apps.cn.md)
- [Flow 与依赖指南](flow-dependencies.cn.md)
- [故障排查](troubleshooting.cn.md)
- [从源码构建](https://github.com/taffish/taffish/blob/main/docs/dev/zh-CN/build-from-source.md)
