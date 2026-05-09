# 容器化 app 最佳实践

生信工具经常依赖复杂系统包、编译环境、运行库和外部数据。TAFFISH 推荐把这类 tool app 容器化，再通过 `.taf` 提供稳定命令入口。

## 目录

- [什么时候使用容器](#什么时候使用容器)
- [基本结构](#基本结构)
- [镜像命名与 tag](#镜像命名与-tag)
- [运行时挂载与工作目录](#运行时挂载与工作目录)
- [Dockerfile 建议](#dockerfile-建议)
- [多阶段构建](#多阶段构建)
- [APT 安装建议](#apt-安装建议)
- [运行路径与环境变量](#运行路径与环境变量)
- [GitHub Actions](#github-actions)
- [GHCR 可见性](#ghcr-可见性)
- [本地测试](#本地测试)
- [常见问题](#常见问题)

## 什么时候使用容器

推荐使用容器的情况：

- 上游工具需要编译。
- 上游工具依赖很多系统库。
- apt/conda 安装结果随时间变化。
- 用户本地很难安装依赖。
- 希望 tool app 在 Linux/macOS 上表现一致。
- 需要同时支持 Docker、Podman 或 Apptainer。

可以不使用容器的情况：

- app 只是简单 shell 包装。
- 上游工具是纯脚本并且依赖很少。
- app 本身是 flow，只调用其他 taf app。

## 基本结构

```text
my-tool/
  taffish.toml
  src/
    main.taf
  docs/
    help.md
  docker/
    Dockerfile
  .github/
    workflows/
      build-image.yml
```

`taffish.toml`：

```toml
[container]
image = "ghcr.io/taffish/my-tool:0.1.0-r1"
dockerfile = "docker/Dockerfile"
build_platforms = "linux/amd64,linux/arm64"
```

`src/main.taf`：

```taf
<taf-app:container:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --input ::input::
```

## 镜像命名与 tag

推荐格式：

```text
ghcr.io/taffish/<app-name>:<version>-r<release>
```

例如：

```text
ghcr.io/taffish/blast:2.16.0-r1
```

原则：

- 镜像 tag 应与 TAFFISH version id 一致。
- 不要用 `latest` 作为 app 的正式引用。
- 不要覆盖已发布 tag。
- 如果 Dockerfile 修复但上游版本不变，增加 release，例如 `2.16.0-r2`。

## 运行时挂载与工作目录

容器化 TAFFISH app 并不是完全脱离用户工作目录运行。运行时，TAFFISH 通常会：

- 把容器内工作目录设置为当前 TAFFISH workdir；
- 把用户 home 目录按相同路径 bind mount 进容器；
- 把当前 workdir 按相同路径 bind mount 进容器；
- 传入 `HOME`、`USER` 等常见环境变量。

用简化的 Docker/Podman 形式表示，大致是：

```sh
podman run --rm -i \
  -w "$PWD" \
  -v "$HOME:$HOME" \
  -v "$PWD:$PWD" \
  -e "HOME=$HOME" \
  -e "USER=$USER" \
  ghcr.io/taffish/my-tool:0.1.0-r1 \
  my-tool ::*ARGV*::
```

这让本地输入输出路径可以自然工作，但也意味着宿主机目录可能遮住容器镜像里的同名
目录。不要在容器内部重要路径对应的宿主机系统目录下运行容器化 taf app，尤其是：

```text
/bin
/usr/bin
/usr/local/bin
/lib
/usr/lib
/opt
```

例如某个镜像把 `augustus` 安装在 `/usr/bin`，如果用户在宿主机 `/usr/bin` 下运行
`taf-augustus`，bind mount 可能遮住容器原本的 `/usr/bin`，最后看起来就像：

```text
augustus: executable file not found in $PATH
```

建议在项目目录、数据目录或临时工作目录中运行 TAFFISH app，例如：

```sh
mkdir -p ~/work/taffish-runs/augustus-test
cd ~/work/taffish-runs/augustus-test
taf-augustus -- --help
```

## Dockerfile 建议

基础写法：

```dockerfile
FROM debian:12

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
      build-essential \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /root

ENV TAFFISH_ENV=TAFFISH
ENV TAFFISH_NAME=my-tool
```

建议：

- 使用明确 base image tag。
- 使用 `--no-install-recommends`。
- 安装后清理 `/var/lib/apt/lists/*`。
- 固定上游 release 下载地址。
- 下载后删除源码包、临时文件和 build cache。
- 运行时只保留必要二进制和运行库。
- 不把测试数据、编译缓存、文档大包无意放进镜像。

## 多阶段构建

如果工具需要编译，推荐多阶段构建。

示例：

```dockerfile
FROM debian:12 AS build

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
      build-essential \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /build

RUN curl -L -o tool.tar.gz https://example.org/tool-1.0.0.tar.gz \
 && tar -xf tool.tar.gz \
 && cd tool-1.0.0 \
 && make

FROM debian:12

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      ca-certificates \
      libgomp1 \
 && rm -rf /var/lib/apt/lists/*

COPY --from=build /build/tool-1.0.0/bin/tool /usr/local/bin/tool

ENV TAFFISH_ENV=TAFFISH
ENV TAFFISH_NAME=my-tool
```

这样可以把编译器、源码和中间文件留在 build stage，减小最终镜像。

## APT 安装建议

推荐：

```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends pkg1 pkg2 \
 && rm -rf /var/lib/apt/lists/*
```

避免：

```dockerfile
RUN apt-get update
RUN apt-get install -y pkg1 pkg2
```

原因：

- 分层会保留 apt index。
- 镜像更大。
- 构建缓存更容易失效或不一致。

## 运行路径与环境变量

推荐把 app 二进制放在：

```text
/usr/local/bin
```

或上游目录并设置：

```dockerfile
ENV PATH=/opt/my-tool/bin:$PATH
```

TAFFISH 约定环境变量：

```dockerfile
ENV TAFFISH_ENV=TAFFISH
ENV TAFFISH_NAME=my-tool
```

这些变量不是上游软件必须项，但有助于识别镜像来自 TAFFISH app。

## GitHub Actions

带 Dockerfile 的 app 应在自己的仓库中构建镜像。

`taf new --tool --docker` 会生成相关 workflow。一般流程是：

1. checkout app 仓库。
2. 读取 `taffish.toml` 中的 image 和 Dockerfile。
3. 登录 GHCR。
4. buildx 构建多平台镜像。
5. 推送 `ghcr.io/taffish/<app>:<version>-r<release>`。

Hub 不负责替 app 构建镜像。Hub 只读取 `taffish.toml` 和 release tag，把镜像信息写入 index。

## GHCR 可见性

镜像发布后，需要确认 GHCR package 是 public。否则用户安装后运行 app 时可能无法拉取镜像。

TAFFISH `0.2.0` 的镜像配置可以重写 index 和 app 仓库访问路径，但不会自动重写
容器镜像 registry。如果用户无法访问 GHCR，app 需要发布或声明这些用户可访问的镜像来源。

检查项：

- package 属于 `taffish` 组织。
- package visibility 是 public。
- image tag 存在。
- image tag 与 `taffish.toml` 一致。

## 本地测试

`taf build` 默认只构建 command wrapper，不会构建镜像。容器化 app 本地试运行前，通常需要先构建镜像。

构建镜像，并明确使用 Docker：

```sh
taf build --image --backend docker
```

构建命令和镜像：

```sh
taf build --all --backend docker
```

运行时使用相同 backend：

```sh
taf run --backend docker -- --help
```

如果使用 Podman，本地构建和运行也要保持一致：

```sh
taf build --image --backend podman
taf run --backend podman -- --help
```

`taf run --backend` 只影响 `<container:...>` / `<taf-app:container:...>` 这种泛化标签。显式标签会固定后端：

```taf
<taf-app:docker:ghcr.io/taffish/my-tool:0.1.0-r1>
my-tool --help
```

如果一个镜像只在本地 Docker 或 Podman 中存在，或者运行参数依赖某个后端，建议在 `src/main.taf` 中使用显式后端标签，而不是泛化的 `container`。

也可以直接测试容器：

```sh
docker run --rm ghcr.io/taffish/my-tool:0.1.0-r1 my-tool --help
```

## 常见问题

镜像太大：

- 使用多阶段构建。
- 删除源码包和编译缓存。
- 使用 `--no-install-recommends`。
- 只安装运行时依赖。

用户无法拉取镜像：

- 检查 GHCR visibility。
- 检查组织和 package 权限。
- 检查 tag 是否存在。

`taf check` 报镜像 tag 不匹配：

- 确认 `[package].version` 和 `[package].release`。
- 确认 `[container].image` tag。
- 确认 `src/main.taf` 中 `<taf-app:container:...>` 的镜像。

`taf build --image` 成功，但 `taf run` 找不到镜像：

- 确认 `taf build --image --backend ...` 和 `taf run --backend ...` 使用同一个后端。
- 如果 `src/main.taf` 写的是 `<taf-app:container:...>`，运行时可能自动选择了别的可用后端。
- 开发期可以显式写 `<taf-app:docker:...>` 或 `<taf-app:podman:...>`。

多平台构建失败：

- 先确认 amd64 单平台能通过。
- 检查上游工具是否支持 arm64。
- 必要时暂时只发布 `linux/amd64`。
