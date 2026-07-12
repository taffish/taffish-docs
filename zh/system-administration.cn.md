# TAFFISH 系统管理员指南

[English](../en/system-administration.en.md) | [中文](system-administration.cn.md)

这份指南面向在共享工作站、服务器或 HPC 登录节点上安装 TAFFISH 的系统管理员。它覆盖
system scope、权限、全局 shell 集成、容器后端，以及集中管理 app 与用户个人状态之间的边界。

TAFFISH 不是调度器。系统级安装的 `taf-*` 命令可以从普通 shell、Slurm/PBS/LSF
脚本、workflow system、Python、R 或 AI 客户端调用；任务调度和资源策略仍由外层平台负责。

## 目录

- [Scope 模型](#scope-模型)
- [默认布局](#默认布局)
- [系统级安装 TAFFISH](#系统级安装-taffish)
- [初始化受管安装](#初始化受管安装)
- [支持 Scope 的命令矩阵](#支持-scope-的命令矩阵)
- [配置与镜像](#配置与镜像)
- [系统与用户混合部署](#系统与用户混合部署)
- [命令路径](#命令路径)
- [系统级 Shell 补全](#系统级-shell-补全)
- [系统级 Vim 与 Neovim](#系统级-vim-与-neovim)
- [容器后端策略](#容器后端策略)
- [共享系统上的 Apptainer](#共享系统上的-apptainer)
- [升级 TAFFISH 与 Apps](#升级-taffish-与-apps)
- [备份与移除](#备份与移除)
- [运维检查清单](#运维检查清单)

## Scope 模型

TAFFISH 支持两个相互独立的状态 scope：

| Scope | 用途 | 默认 home |
| --- | --- | --- |
| `--user` / `-u` | 单个用户拥有的状态和 app | `~/.local/share/taffish` |
| `--system` / `-s` | 由管理员为整台机器维护的共享状态和 app | `/opt/taffish` |

把 `taf` 二进制安装到系统目录，不会改变后续命令的默认 scope。所有支持 scope 的命令
每次调用时仍默认使用 user scope。管理员如果直接运行不带 `--system` 的 `taf update`，
更新的是 root 的用户级 index，而不是机器的系统级 index。

管理员脚本、runbook 和示例都应显式写出 scope。

## 默认布局

| 项目 | User Scope | System Scope |
| --- | --- | --- |
| 核心命令 | `~/.local/bin` | `/usr/local/bin` |
| 已安装 app 命令 | `~/.local/share/taffish/bin` | `/usr/local/bin` |
| TAFFISH home | `~/.local/share/taffish` | `/opt/taffish` |
| 配置文件 | `~/.local/share/taffish/config.toml` | `/opt/taffish/config.toml` |
| 缓存 index | `<home>/index/current.json` | `<home>/index/current.json` |
| 已安装 app 源码与元数据 | `<home>/apps` | `<home>/apps` |
| Apptainer SIF 缓存 | `<home>/images/sif` | `<home>/images/sif` |

自定义部署可以设置 `TAFFISH_SYSTEM_HOME` 和 `TAFFISH_SYSTEM_BIN_DIR`。管理员与
普通用户必须持久使用相同值，否则 `taf --system` 与已安装 app launcher 可能解析到不同位置。

## 系统级安装 TAFFISH

GitHub 默认安装器：

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sudo sh -s -- --system
```

中国/Gitee 安装器：

```sh
curl -fsSL https://gitee.com/taffish-org/taffish/raw/main/install/install-taffish.gitee.sh | sudo sh -s -- --system
```

安装器会把三个核心命令写入 `/usr/local/bin`，创建 system home，在需要时初始化系统配置，
并尝试更新系统 index。它会复制 completion 和 Vim 文件，但不会修改全局 shell 或编辑器配置。

## 初始化受管安装

安装器通常会自动执行这些步骤。它们也可以用于修复手动安装或未完成的部署：

```sh
sudo taf doctor --init --system
sudo taf config init --system --github
sudo taf update --system
taf doctor --system
```

中国镜像 profile：

```sh
sudo taf config init --system --china --force
sudo taf update --system
```

安装并检查一个共享 app：

```sh
taf search --system fastqc
sudo taf install --system fastqc
taf list --system
taf which --system taf-fastqc
taf-fastqc -- --help
```

## 支持 Scope 的命令矩阵

| 命令 | 典型系统用法 | 是否写入系统状态 |
| --- | --- | --- |
| `taf update` | `sudo taf update --system` | 是 |
| `taf search` | `taf search --system <QUERY>` | 否 |
| `taf info` | `taf info --system <APP>` | 否 |
| `taf install` | `sudo taf install --system <APP>` | 是，dry-run 除外 |
| `taf outdated` | `taf outdated --system` | 否 |
| `taf upgrade` | `sudo taf upgrade --system --yes` | 应用计划时写入 |
| `taf prune` | `sudo taf prune --system --yes` | 应用计划时写入 |
| `taf uninstall` | `sudo taf uninstall --system <APP>` | 是，dry-run 除外 |
| `taf list` | `taf list --system` | 否 |
| `taf which` | `taf which --system <COMMAND>` | 否 |
| `taf doctor` | `taf doctor --system` | 仅 `--init` 写入 |
| `taf config` | `taf config --system` | 仅 `init` 写入 |

`taf new`、`taf check`、`taf run`、`taf build` 和 `taf publish` 等项目命令
不是系统 package 管理命令。

## 配置与镜像

user scope 按以下顺序加载配置：

1. system config；
2. user config；
3. `TAFFISH_CONFIG` 指定的显式文件。

后加载的文件覆盖前面的配置。因此管理员可以给出站点默认值，并在本地策略允许时让用户覆盖。
system scope 读取 system config 和可选显式配置，但不读取 user config。

诊断共享机器时应分别检查：

```sh
taf config --system
taf config path --system
taf config --user
taf config path --user
```

source rewrite 只改变 app 仓库 clone URL，不会重写容器 registry URL。私有或地区化部署
仍需单独保证目标容器镜像可以访问。

## 系统与用户混合部署

一种常见部署方式是只把三个核心命令安装到系统范围，让每位研究者在 user scope 中维护
个人 app 和 index：

```sh
taf doctor --init --user
taf update --user
taf install --user fastqc
```

用户必须把两个命令目录都加入 `PATH`：

```sh
export PATH="$HOME/.local/bin:$HOME/.local/share/taffish/bin:$PATH"
```

用户也可以直接运行 `/usr/local/bin` 中集中安装的 app。要检查系统安装元数据，必须使用
`taf list --system` 或 `taf which --system`；默认 `taf list` 只读取个人 user home。

## 命令路径

系统 app launcher 和三个核心命令默认共用 `/usr/local/bin`，因此无需逐个修改用户 PATH。

用户级安装则有意分开：

```text
~/.local/bin                    # taf, taffish, taffish-mcp
~/.local/share/taffish/bin     # 已安装的 taf-* 命令
```

`taf doctor --user` 与 `taf doctor --system` 会报告当前 command bin 是否位于 `PATH`。

## 系统级 Shell 补全

安装后的 completion 文件位于 `/opt/taffish/share/completions`。

使用 `bash-completion` 的 Bash：

```sh
sudo install -m 0644 /opt/taffish/share/completions/bash/taf /etc/bash_completion.d/taf
sudo install -m 0644 /opt/taffish/share/completions/bash/taffish /etc/bash_completion.d/taffish
```

也可以在全局交互式 Bash 启动文件中加入：

```sh
source /opt/taffish/share/completions/bash/taf
source /opt/taffish/share/completions/bash/taffish
```

Debian 系通常使用 `/etc/bash.bashrc`，Red Hat 系通常使用 `/etc/bashrc`。

Zsh 应在站点 `compinit` 之前把安装目录加入全局 `fpath`：

```zsh
fpath=(/opt/taffish/share/completions/zsh $fpath)
autoload -Uz compinit
compinit
```

Fish 可以把两个 `.fish` 文件复制到 `/usr/share/fish/vendor_completions.d` 或
`/usr/local/share/fish/vendor_completions.d` 等 vendor completion 目录。Shell 包布局
会随系统变化，选择全局路径前应先检查目标机器。

## 系统级 Vim 与 Neovim

安装后的 Vim 文件位于 `/opt/taffish/share/vim`。可以在全局 Vim 配置中加入：

```vim
set runtimepath+=/opt/taffish/share/vim
```

也可以把 `syntax/taf.vim` 和 `ftdetect/taf.vim` 复制到站点 Vim runtime。常见位置包括
`/usr/share/vim/vimfiles`、`/usr/local/share/vim/vimfiles` 和 `/etc/xdg/nvim`，
但准确路径取决于操作系统。

## 容器后端策略

TAFFISH 可以自动选择可用后端，也可以接收显式 backend override。站点应明确记录支持哪种后端：

- Docker 使用 daemon 和站点级 socket 权限；Docker group 成员资格具有安全敏感性。
- Podman 常按用户使用并支持 rootless；不同用户的 image store 可能不共享。
- Apptainer 通常最适合没有普通用户 Docker daemon 权限的 HPC 和共享 Linux 系统。

机器专用 runtime 参数应放在 `TAFFISH_DOCKER_RUN_ARGS`、`TAFFISH_PODMAN_RUN_ARGS`
和 `TAFFISH_APPTAINER_RUN_ARGS` 等环境变量中。调度器指令和资源申请仍由外层 job script
或 workflow system 管理。

## 共享系统上的 Apptainer

系统 app launcher 可以全局可读，但 system SIF 缓存未必可写。管理员应明确选择一种策略：

1. 用户级缓存：每个用户运行 `taf doctor --init --user`。
2. 管理员预缓存：管理员先运行常用的版本化 app 命令，让 SIF 写入 system home。
3. 共享可写缓存：可信本地用户共用 sticky 目录。

共享可写缓存示例：

```sh
sudo mkdir -p /opt/taffish/images/sif
sudo chmod 1777 /opt/taffish/images/sif
```

只有在信任本地用户创建缓存文件时才使用 `1777`。它不会让容器内容自动可信，也可能不符合
更严格的集群策略。

## 升级 TAFFISH 与 Apps

`taf upgrade` 只升级已安装 taf-app。三个核心二进制应通过相同 scope 重新运行安装器升级：

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sudo sh -s -- --system --version X.Y.Z
```

请把 `X.Y.Z` 替换为站点批准的 release 版本。

推荐的 app 维护顺序：

```sh
sudo taf update --system
taf outdated --system
sudo taf upgrade --system --yes
sudo taf prune --system --yes
```

`upgrade --prune-old`、`prune` 和 `uninstall` 会删除 TAFFISH app 源码、元数据和 launcher，
但不会删除 Docker、Podman、Apptainer 或 SIF 镜像缓存。

## 备份与移除

修改 system home 前，应备份自定义 `/opt/taffish/config.toml` 和需要保留的本地 app、
index 数据。TAFFISH 当前没有专用核心卸载器。

移除核心前：

1. 使用 `--system` 列出并按需卸载集中管理的 app。
2. 保留站点策略要求的配置、日志或缓存。
3. 移除 `/usr/local/bin/taf`、`/usr/local/bin/taffish` 和
   `/usr/local/bin/taffish-mcp`。
4. 只有确认没有需要保留的状态后，才移除 `/opt/taffish`。

不要在可能存在其他命令的机器上使用宽泛通配符删除 `/usr/local/bin/taf-*`。应通过
`taf list --system`、`taf which --system` 和 install metadata 识别 TAFFISH 管理的 launcher。

## 运维检查清单

- [ ] 普通用户可以运行 `taf --version`。
- [ ] `taf doctor --system` 显示预期的 home 和 command bin。
- [ ] `taf config --system` 显示预期的 index 与镜像策略。
- [ ] `taf search --system <QUERY>` 可以读取系统 index。
- [ ] 可以使用 `sudo taf install --system APP` 安装测试 app。
- [ ] 普通用户的 `PATH` 可以找到 app 命令。
- [ ] 普通用户可以使用站点选择的容器后端。
- [ ] Apptainer 缓存所有权遵循明确的站点策略。
- [ ] 只为站点实际支持的 Bash/Zsh/Fish 启用补全。
- [ ] Vim/Neovim 集成安装在站点策略规定的位置。
- [ ] 核心升级和 app 升级的责任已经分开记录。
- [ ] 已为当前机器记录备份和移除路径。

普通用户工作流见 [TAFFISH 快速开始](quick-start.cn.md)，运行故障见
[TAFFISH 故障排查](troubleshooting.cn.md)。
