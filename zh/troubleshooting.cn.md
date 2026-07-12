# TAFFISH 故障排查

[English](../en/troubleshooting.en.md) | [中文](troubleshooting.cn.md)

这份文档收集 TAFFISH 用户和 app 开发者常见问题。建议先根据报错信息定位到对应
章节。

## 目录

- [找不到 `taf` 或已安装的 App 命令](#找不到-taf-或已安装的-app-命令)
- [系统级安装看起来是空的](#系统级安装看起来是空的)
- [macOS 上 `taffish` 找不到运行库](#macos-上-taffish-找不到运行库)
- [`taf update` 失败](#taf-update-失败)
- [`taf install` 找不到 app](#taf-install-找不到-app)
- [GitHub 认证或发布失败](#github-认证或发布失败)
- [GHCR 镜像拉取失败](#ghcr-镜像拉取失败)
- [Docker permission denied](#docker-permission-denied)
- [Podman machine 或 crun 报错](#podman-machine-或-crun-报错)
- [Apptainer 缺少 `mksquashfs`](#apptainer-缺少-mksquashfs)
- [Apptainer 提示 `no writable apptainer image directory found`](#apptainer-提示-no-writable-apptainer-image-directory-found)
- [构建镜像时 backend 不接受 Apptainer](#构建镜像时-backend-不接受-apptainer)
- [`taf check` 拒绝 `[smoke]`](#taf-check-拒绝-smoke)
- [smoke 命令因为嵌套引号失败](#smoke-命令因为嵌套引号失败)
- [参数被 wrapper 处理而不是传给上游工具](#参数被-wrapper-处理而不是传给上游工具)
- [`exec`: executable file not found](#exec-executable-file-not-found)
- [镜像里有工具但运行时找不到](#镜像里有工具但运行时找不到)
- [镜像存在于一个后端但另一个后端找不到](#镜像存在于一个后端但另一个后端找不到)
- [获取更多上下文](#获取更多上下文)

## 找不到 `taf` 或已安装的 App 命令

默认用户级安装使用 `~/.local/bin` 保存三个核心命令，使用
`~/.local/share/taffish/bin` 保存 `taf install` 安装的 app。应把两者都加入 `PATH`：

```sh
export PATH="$HOME/.local/bin:$HOME/.local/share/taffish/bin:$PATH"
```

然后重新打开终端，或 source 你的 shell 配置文件。

检查：

```sh
which taf
taf --version
taffish --version
taffish-mcp --version
```

如果自定义了 `TAFFISH_USER_HOME`，还要加入它的 `bin` 子目录。系统级安装默认把
核心命令和 app 命令都放在 `/usr/local/bin`；共享命令找不到时运行
`taf doctor --system` 检查。

## 系统级安装看起来是空的

即使 `taf` 本身来自系统安装，支持 scope 的命令仍默认读取 user scope。应显式使用：

```sh
taf doctor --system
taf config --system
taf list --system
taf which --system APP_OR_COMMAND
taf search --system QUERY
```

系统写操作通常需要管理员权限：

```sh
sudo taf update --system
sudo taf install --system APP
```

完整 scope 与权限矩阵见 [TAFFISH 系统管理员指南](system-administration.cn.md)。

## macOS 上 `taffish` 找不到运行库

当前 macOS Apple Silicon 二进制可能需要 Homebrew 的 `zstd`：

```sh
brew install zstd
```

然后重新检查：

```sh
taffish --version
taffish-mcp --version
taf doctor
```

## `taf update` 失败

`taf update` 会下载静态 TAFFISH index。失败通常是网络问题，或 GitHub raw
content 访问不稳定。

可以尝试：

```sh
taf update
taf update --url INDEX_URL
```

如果使用 TAFFISH `0.2.0` 或更新版本，需要持久使用镜像源，可以查看并初始化运行时配置：

```sh
taf config --user
taf config path --user
taf config init --user --china --force
taf update --user
```

生成的中国镜像 profile 会改变 index URL，并在 `taf install` clone app 仓库时把
canonical GitHub 仓库 URL 重写为配置的镜像地址。如果使用内部 Git 服务，可以直接
编辑 `config.toml`，把 `[index].url` 和 `[[source.rewrite]].to` 指向内部镜像。

如果你的网络需要代理，需要在 TAFFISH 外部配置 shell、Git 或系统代理。

镜像配置不会自动镜像容器 registry。如果某个 app 使用 GHCR，运行 app 的机器仍然
需要能访问这个镜像来源。

安装器运行 `taf update` 失败时可能会给出 warning，但不会回滚已经安装的二进制。

## `taf install` 找不到 app

先更新本地 index：

```sh
taf update
```

然后搜索：

```sh
taf search KEYWORD
taf list --online
```

如果某个仓库刚发布，需要等待 `taffish-index` 自动化运行；如果你是 index
维护者，也可以手动触发 index 自动化。

可接受的安装形式：

```sh
taf install augustus
taf install taf-augustus
taf install augustus 3.5.0-r1
taf install taf-augustus-v3.5.0-r1
```

## GitHub 认证或发布失败

`taf publish` 使用 Git 和 GitHub，但不负责登录。

检查：

```sh
git remote -v
git status
gh auth status
```

如果远端 Git 操作需要在终端里询问凭据，使用：

```sh
taf publish --release --dry-run --prompt
taf publish --release --yes --prompt
```

如果远端仓库不存在，并且希望 `taf` 创建：

```sh
taf publish --release --yes --build --create-repo --public
```

正式发布时，推荐使用 release notes 工作流：

```sh
taf publish --release --dry-run
taf publish --release --yes --build
```

`taf publish --release` 会读取被 ignore 的 `release.md` 草稿。如果 GitHub
Release 标题或说明不对，编辑 `release.md`：第一行会成为 publish message，整个
文件会成为 GitHub Release notes。
从 TAFFISH `0.8.1` 开始，`taf publish --release` 只会拒绝第一行仍为默认占位符
`TODO` 或 `TODO: release summary` 的 release notes；真实 summary 中正常出现
`todo` 这个词不会被误拒绝。

## GHCR 镜像拉取失败

app 可能安装成功，但运行时因为镜像无法拉取而失败。

检查：

- `taffish.toml` 里的 image 名称；
- image tag 是否匹配 `<version>-r<release>`；
- GHCR package 是否 public；
- 运行 app 的机器是否能访问 `ghcr.io`。

手动测试：

```sh
docker pull ghcr.io/taffish/APP:VERSION-rRELEASE
podman pull ghcr.io/taffish/APP:VERSION-rRELEASE
```

## Docker permission denied

Linux 上把用户加入 `docker` 组：

```sh
sudo usermod -aG docker "$USER"
```

然后退出并重新登录。

测试：

```sh
docker run --rm hello-world
```

## Podman machine 或 crun 报错

macOS 上 Podman 通过 Linux VM 运行。如果 Podman 无法连接：

```sh
podman machine list
podman machine start
```

如果看到类似报错：

```text
crun: create keyring ... Disk quota exceeded
```

这通常是 Podman/crun 状态或 Linux kernel keyring 配额问题，不是 TAFFISH app
自身问题。这里的 `Disk quota exceeded` 不一定表示文件系统磁盘满了。Podman/crun
可能会为容器创建 keyring；而 TAFFISH flow 在组合多个 taf app 时，可能会频繁启动
短生命周期容器，因此更容易暴露较低的 per-user keyring quota。

在 Linux 上，或者 macOS Podman Machine 使用的 Linux VM 内部，可以检查当前限制和
key 使用情况：

```sh
sysctl kernel.keys.maxkeys kernel.keys.maxbytes
cat /proc/key-users
```

macOS 用户需要先进入 Podman VM：

```sh
podman machine ssh
```

如果 key quota 对当前工作负载太小，可以由管理员调高。下面只是示例值，不是通用
默认值：

```sh
sudo sysctl -w kernel.keys.maxkeys=20000
sudo sysctl -w kernel.keys.maxbytes=2000000
```

如果需要在 Linux 中持久化：

```sh
sudo tee /etc/sysctl.d/99-taffish-podman-keyring.conf >/dev/null <<'EOF'
kernel.keys.maxkeys = 20000
kernel.keys.maxbytes = 2000000
EOF

sudo sysctl --system
```

具体数值应根据本机策略、用户数量和工作负载决定。共享服务器上不要随意修改 kernel
sysctl 设置，应先联系系统管理员。更多背景可参考 Linux kernel key service 文档以及
你的 Podman/Linux 发行版文档。

谨慎检查和清理：

```sh
podman system df
podman ps -a
podman container prune
podman machine stop
podman machine start
```

更强的清理：

```sh
podman system prune
```

只有确认未使用镜像和 volume 可以删除时，才使用：

```sh
podman system prune -a --volumes
```

## Apptainer 缺少 `mksquashfs`

Apptainer 把 Docker/OCI 镜像转换成 SIF 时，可能需要 `mksquashfs`。

安装：

```sh
sudo apt-get install -y squashfs-tools
```

如果 Apptainer 每次都把 SIF 转成临时 sandbox，可以安装：

```sh
sudo apt-get install -y squashfuse
```

减少常见 warning 的可选包：

```sh
sudo apt-get install -y fuse2fs gocryptfs
```

## Apptainer 提示 `no writable apptainer image directory found`

这个问题通常发生在共享机器上：app 通过 `taf install --system` 做了系统级安装，
但它对应的 Apptainer SIF 镜像还没有缓存到当前用户可写的目录中。

系统级安装会让 app launcher 对用户可见，但不会在安装阶段自动拉取所有
Apptainer SIF 镜像。首次运行时，TAFFISH 会查找：

```text
${TAFFISH_SYSTEM_HOME:-/opt/taffish}/images/sif
${TAFFISH_USER_HOME:-$HOME/.local/share/taffish}/images/sif
```

如果没有找到已有 SIF，至少需要其中一个目录可写，TAFFISH 才能拉取或创建镜像。

常见策略：

```sh
# 每个用户使用自己的缓存，通常最稳妥。
taf doctor --init --user
```

```sh
# 管理员预先把 SIF 缓存到系统 TAFFISH home。
sudo taf-APP-vVERSION-rRELEASE -- --help
```

```sh
# 给可信本地用户共享一个可写系统缓存。
sudo mkdir -p /opt/taffish/images/sif
sudo chmod 1777 /opt/taffish/images/sif
```

`1777` 类似 `/tmp`：本地用户可以创建文件，但不能删除其他用户拥有的文件。只有
管理员明确允许可信本地用户共享可写 SIF 缓存时才建议这样设置。更严格的共享系统
应优先使用管理员预缓存或用户级缓存。

## 构建镜像时 backend 不接受 Apptainer

`taf run --backend` 可以使用：

```text
docker, podman, apptainer
```

但 `taf build --image --backend` 当前接受的镜像构建后端是：

```text
docker, podman
```

所以这样是有效的：

```sh
taf build --image --backend docker
taf run --backend docker -- --help
```

这样不能用于镜像构建：

```sh
taf build --image --backend apptainer
```

## `taf check` 拒绝 `[smoke]`

从 TAFFISH `0.8.0` 开始，容器化 app 必须提供真实 `[smoke]` 元数据。
`taf check` 会校验这部分元数据，但不会在本地运行 smoke 命令。

常见原因：

- 声明了 `[container].image` 或 `[container].dockerfile` 的项目没有 `[smoke]`。
- `exist` 和 `test` 同时缺失或为空。
- `taf new --tool --docker` 生成的默认 `TODO` 占位还没有替换。
- `backend` 不是 `docker`、`podman` 或 `apptainer`。
- `timeout` 不是正整数。

最小示例：

```toml
[smoke]
backend = "docker"
timeout = 60
exist = ["my-tool"]
test = ["my-tool --help"]
```

如果默认值足够，`backend` 和 `timeout` 可以省略。Smoke 检查应该短小、确定、可
重复。公开 Hub/index 自动化会在镜像推送后，对新的容器化版本运行这些检查；失败
结果会进入 index 构建报告，而不是进入主安装 index。

## smoke 命令因为嵌套引号失败

Smoke `test` 条目是 TOML 字符串，并会在 smoke 容器中通过 `sh -lc` 执行。如果命令
本身需要引号，推荐在 shell 片段内部使用单引号：

```toml
[smoke]
test = ["python -c 'import vina, rdkit, meeko, gemmi, prody'"]
```

`\"` 这类 TOML basic string 转义是合法的，但多层嵌套引号更难审核，也更容易复制
出错。如果 index report 中出现 Python `SyntaxError`，并且代码看起来像以多余的引号
开头，应优先检查 `[smoke].test` 中的引号写法。

## 参数被 wrapper 处理而不是传给上游工具

在上游参数前使用 `--`：

```sh
taf-my-tool -- --help
taf run -- --help
```

如果不加 `--`，`--help`、`--version` 或 `--compile` 等参数可能会被 TAFFISH
wrapper 自己处理。

## `exec`: executable file not found

简单容器 wrapper 中不要这样写：

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
exec my-tool ::*ARGV*::
```

根据生成的运行方式，`exec` 可能会被传给容器当成可执行文件名。由于 `exec` 是
shell builtin，容器运行时可能报错：

```text
exec: "exec": executable file not found in $PATH
```

使用：

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool ::*ARGV*::
```

## 镜像里有工具但运行时找不到

现象：

```text
my-tool: executable file not found in $PATH
```

或者对于具体 app：

```text
augustus: executable file not found in $PATH
```

如果镜像本身正确，并且工具确实安装在镜像里，需要检查宿主机当前目录。TAFFISH
容器标签通常会把当前 workdir 按相同路径 bind mount 进容器，并把该路径设为容器
工作目录。这让输入输出文件路径更自然，但如果宿主机路径和容器内部关键目录同名，
就可能遮住镜像里的原始目录。

高风险宿主机工作目录包括：

```text
/bin
/usr/bin
/usr/local/bin
/lib
/usr/lib
/opt
```

例如镜像把 `augustus` 安装在 `/usr/bin`，而你在宿主机 `/usr/bin` 下运行
`taf-augustus`，bind mount 可能覆盖容器原本的 `/usr/bin`。容器可以启动，但原本
在 `/usr/bin` 里的工具被遮住了。

修复方式：

```sh
mkdir -p ~/work/taffish-runs/augustus-test
cd ~/work/taffish-runs/augustus-test
taf-augustus -- --help
```

原则上，从项目目录、数据目录或临时工作目录运行 TAFFISH app。避免从系统目录运行，
尤其是这些目录在容器内部也有重要含义时。

## 镜像存在于一个后端但另一个后端找不到

如果用 Docker 构建镜像，却用 Podman 运行，Podman 可能看不到 Docker 的镜像。

本地开发时使用同一个后端：

```sh
taf build --image --backend docker
taf run --backend docker -- --help
```

或者：

```sh
taf build --image --backend podman
taf run --backend podman -- --help
```

如果 app 源码明确写了 `<taf-app:docker:...>` 或 `<taf-app:podman:...>`，
`taf run --backend` 不会覆盖这种显式标签。

对于已经安装好的 `taf-*` 命令，或者直接使用 `taffish` 编译时，不会经过
`taf run --backend`。如果需要让泛化 `<container:...>` 标签使用某个指定后端，
可以设置 `TAFFISH_CONTAINER_BACKEND=apptainer|podman|docker`：

```sh
TAFFISH_CONTAINER_BACKEND=podman taf-my-tool-v0.1.0-r1 -- --help
```

这仍然不会覆盖显式的 `<docker:...>`、`<podman:...>` 或 `<apptainer:...>` 标签。

如果问题不是后端选择，而是单次 runtime 策略，应使用 backend-specific runtime 参数环境变量。例如在 ARM 主机上需要运行 amd64 Docker/Podman 镜像：

```sh
TAFFISH_DOCKER_RUN_ARGS="--platform linux/amd64" taf-my-tool ...
TAFFISH_PODMAN_RUN_ARGS="--platform linux/amd64" taf-my-tool ...
```

这些变量只用于本地策略。如果 app 本身正常运行就需要 backend-specific 参数，例如
GPU 访问，应在 app 源码中用 `$@[target: args]` 声明。

## 获取更多上下文

普通用户先看 [TAFFISH 快速开始](quick-start.cn.md)；共享部署先看
[TAFFISH 系统管理员指南](system-administration.cn.md)。

有用的命令：

```sh
taf doctor
taf config
taf history
taf which APP_OR_COMMAND
taf info APP_OR_COMMAND
```

对于 app 项目：

```sh
taf check
taf compile -- --help
taf run -- --help
```
