# 官方 taf-app 精修手册

本文面向 TAFFISH Hub 官方维护者，用于把一个通过 `taf new` 创建的 app
整理成可以发布、可以索引、可以作为后续 app 模板参考的成品。

这不是 `taffish.toml` 的完整规范，也不是 Dockerfile 教程。更准确地说，它是一份
curation guide：当你拿到一个 app 目录时，应该检查什么、修改什么、保持什么不动，
以及怎样判断一个 app 是否足够好。

`augustus` 是当前官方 Hub 的首个正式 taf-app，可以作为此手册的基准模板。

本文偏向维护者判断和 checklist。字段语义仍以 [`taffish.toml` 规范](taffish-toml-spec.cn.md)
为准，容器实现细节仍以 [容器化 app 最佳实践](container-apps.cn.md) 为准。

## 目录

- [定位](#定位)
- [推荐工作流](#推荐工作流)
- [以 `taf new` 为边界](#以-taf-new-为边界)
- [`taffish.toml`](#taffishtoml)
- [`src/main.taf`](#srcmaintaf)
- [`docker/Dockerfile`](#dockerdockerfile)
- [`docs/help.md`](#docshelpmd)
- [`README.md`](#readmemd)
- [`release.md`](#releasemd)
- [Smoke 设计](#smoke-设计)
- [版本与 release 策略](#版本与-release-策略)
- [发布前检查](#发布前检查)
- [发布后检查](#发布后检查)
- [不要做的事](#不要做的事)
- [Augustus 模板要点](#augustus-模板要点)

## 定位

官方 taf-app 的目标是：

- 把上游工具放进可移植、可索引、可验证的 TAFFISH 包装中。
- 让用户通过稳定的 `taf-xxx` 命令运行上游工具。
- 让 Hub index 能记录源码 commit、容器 digest、platform 和 smoke/trust 状态。
- 让 AI 客户端和维护者能读懂 app 的用途、入口、容器和检查方式。

官方 taf-app 不应该试图重新设计上游工具的 CLI。通常最好的 wrapper 是薄的：
提供一致入口、容器运行环境、帮助文档、版本元数据和 smoke gate，把具体功能仍交给
上游工具。

## 推荐工作流

推荐从 `taf new` 开始：

```sh
taf new my-tool --tool --docker \
  --version 1.0.0 \
  --release 1 \
  --repo https://github.com/taffish/my-tool
```

然后按顺序精修：

1. 阅读上游文档，确认主命令、辅助命令、版本策略和安装方式。
2. 设计 Dockerfile，让容器包含真实工作流需要的命令，而不只是主程序。
3. 设计 `src/main.taf`，优先保持薄 wrapper。
4. 精修 `taffish.toml`，尤其是 image、upstream 和 smoke。
5. 写 `docs/help.md`，让用户从 `taf-xxx --help` 能直接开始使用。
6. 写 `README.md`，让 GitHub 页面说明安装、使用、容器和维护检查。
7. 写 `release.md`，让 `taf publish --release` 生成准确的发布说明。
8. 本地检查、容器 smoke、publish dry-run。
9. 发布后等待镜像构建，再检查 index reports。

## 以 `taf new` 为边界

`taf new` 生成的文件不是普通草稿，而是当前工具链的结构契约。维护 app 时应尊重这个
边界。

通常需要维护的文件：

```text
taffish.toml
src/main.taf
docker/Dockerfile
README.md
docs/help.md
release.md
```

通常不要手工改的文件：

```text
.github/workflows/build-image.yml
.gitignore
target/
LICENSE
```

例外：

- 如果 `taf new` 模板本身升级，按新模板同步 `.github/workflows/build-image.yml`。
- 如果许可证确实变化，更新 `LICENSE` 和 `taffish.toml`。
- 如果 `target/` 是构建产物，不手工编辑；重新运行 `taf build` 或 `taf publish`。

## `taffish.toml`

`taffish.toml` 是 app 元数据核心。维护原则是克制：只使用 `taf`、Hub 和 index
明确支持的字段，不为了“看起来更完整”添加工具链读不懂的自定义字段。

推荐结构：

```toml
[package]
name = "my-tool"
kind = "tool"
version = "1.0.0"
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

[container]
image = "ghcr.io/taffish/my-tool:1.0.0-r1"
dockerfile = "docker/Dockerfile"
build_platforms = "linux/amd64,linux/arm64"

[upstream]
type = "github"
repo = "owner/project"
watch = "tags"
strip_prefix = "v"

[smoke]
backend = "docker"
timeout = 60
exist = ["my-tool"]
test = ["my-tool --help"]
```

维护要点：

- `version` 是上游工具版本。
- `release` 是 TAFFISH 包装层 release。
- `[container].image` tag 应与 `<version>-r<release>` 对齐。
- `[repository].url` 是 TAFFISH app 仓库，不是上游仓库。
- `[upstream]` 描述被包装的软件，不描述 TAFFISH app 自己。
- `command_mode = true` 对 tool app 很重要，用户可以运行容器内的辅助命令。
- 不要添加当前工具链不理解的字段。

## `src/main.taf`

大多数容器化 tool app 推荐薄 wrapper：

```taf
<taf-app:container:ghcr.io/taffish/my-tool:1.0.0-r1>
my-tool ::*ARGV*::
```

这种形式有两个入口：

```sh
taf-my-tool -- --help
taf-my-tool -- --input data.txt
```

也可以用 command mode 显式运行容器内命令：

```sh
taf-my-tool my-tool --help
taf-my-tool helper-command --help
```

维护要点：

- 优先使用 `<taf-app:container:...>`，让运行时按环境选择 Docker、Podman 或 Apptainer。
- 只有在必须固定 backend 时，才使用 `<taf-app:docker:...>` 或 `<taf-app:podman:...>`。
- 不要在 `.taf` 里复制大量上游逻辑，除非确实需要统一参数或组合步骤。
- 如果第一屏默认运行会很昂贵，考虑让 `docs/help.md` 引导用户使用 `-- --help`。

## `docker/Dockerfile`

Dockerfile 的目标是让上游工具环境自洽，而不是让 smoke 勉强通过。

建议：

- 使用明确 base image tag。
- 优先使用上游官方 release、官方二进制或发行版包。
- 容器里包含真实工作流常用辅助命令。
- 构建期运行最小检查，例如主命令 help、版本、辅助命令 help、Python import。
- 使用 `--no-install-recommends`，清理 apt lists 和构建缓存。
- 如果上游需要编译，优先使用多阶段构建。
- 不要为了缩小镜像删除用户可能需要的官方数据、配置或文档。

可接受的权衡：

- 生信镜像偏大是常见情况。只要来源清楚、功能完整、构建稳定，镜像大不一定是问题。
- 如果上游官方推荐 conda，但 Docker 本身已经提供隔离环境，可以优先考虑直接安装到镜像
  系统环境或 Python 环境中。只有在依赖确实难以复现时再引入 conda。

## `docs/help.md`

`docs/help.md` 是用户运行 `taf-xxx --help` 看到的内容。它应该像命令手册，而不是项目
介绍。

推荐结构：

```text
taf-my-tool 1.0.0-r1

One-line description.

Usage:
  taf-my-tool [TAF-APP-OPTION]
  taf-my-tool [UPSTREAM-ARGS...]
  taf-my-tool -- [UPSTREAM-ARGS...]
  taf-my-tool my-tool [UPSTREAM-ARGS...]
  taf-my-tool <COMMAND> [COMMAND-ARGS...]

TAF app options:
  -h, --help       Show this help text
  -v, --version    Show package and command version
  --compile        Print generated shell code instead of running it
  --               Pass all following arguments to upstream command

Upstream help:
  taf-my-tool -- --help

Default examples:
  ...

Explicit container command examples:
  ...

Container commands:
  ...

Notes:
  ...

Container:
  image: ghcr.io/taffish/my-tool:1.0.0-r1
  supported backends: apptainer, podman, docker

Upstream:
  project:
  source:
  docs:
```

维护要点：

- Help 应该短、准、能直接复制命令。
- 解释 `--` 的作用，因为 `--help`、`--version` 可能被 TAFFISH wrapper 自己处理。
- 明确 command mode：`taf-my-tool COMMAND ...` 会在同一容器中运行 `COMMAND`。
- 列出容器内重要辅助命令。
- 不要把大量上游手册全文复制进来，链接到上游文档即可。

## `README.md`

README 面向 GitHub 页面和人工审核，比 `docs/help.md` 更适合说明包结构。

推荐章节：

```text
# taf-my-tool

Short description.

## Installation
taf update
taf install my-tool
taf install my-tool 1.0.0-r1
taf install --from .

## Usage
show help
run upstream command
run explicit container commands

## Package
name / command / version / kind / image

## Container
installed commands
installed libraries
build-time checks

## Upstream
project / source / documentation / release page

## Maintainer Notes
taf check
taf compile -- --help
taf publish --release --dry-run
docker build --check -f docker/Dockerfile .
```

当前建议：

- 官方 app README 先统一英文。
- 中文说明优先放在中心 docs 或官网，不在每个 app README 里混合双语。
- README 不应承诺 Dockerfile 尚未实现的功能。

## `release.md`

`release.md` 是本地发布草稿，由 `taf publish --release` 使用，通常被 `.gitignore`
忽略，不提交到 app 仓库。

规则：

- 第一行成为 publish message 的一部分。
- 整个文件成为 GitHub Release notes。
- 第一行不能是默认 `TODO` 占位。
- 第一行应该表达本次 release 的真实变化。

首发示例：

```md
# Publish My Tool 1.0.0 as a TAFFISH app

This release packages My Tool 1.0.0 as `taf-my-tool`.
```

包装层修复示例：

```md
# Fix My Tool 1.0.0 smoke metadata

This release updates TAFFISH smoke metadata without changing the upstream
My Tool version.
```

重要原则：

- App release 记录 app 自身变化。
- 如果失败来自 index bug，不要为了绕过 index bug 升 app release。
- 已发布 tag 不应覆盖；如果 app 本身确实需要修复，增加 `release`。

## Smoke 设计

Smoke 是官方 index gate 的核心。它不是完整测试套件，而是低成本证明：

- 容器镜像可拉取和检查 digest/platform。
- 关键命令在 `PATH` 中。
- 主命令能启动并显示 help 或 version。
- 关键运行时依赖可 import 或可执行。
- smoke 容器在无网络环境下仍能完成检查。

推荐写法：

```toml
[smoke]
backend = "docker"
timeout = 120
exist = ["my-tool", "helper-command", "python"]
test = ["my-tool --help", "my-tool --version 2>&1 | grep -F '1.0.0' >/dev/null", "helper-command --help", "python -c \"import my_tool\""]
```

TOML basic string 中 `\"` 是合法的，标准 parser 应把它解析为普通双引号。为了让多层
shell 更易读，也可以写成：

```toml
test = ["python -c 'import my_tool'"]
```

维护要点：

- 当前 `taf` 项目解析器要求数组写成单行，维护 app 时不要把 `exist` 或 `test` 拆成
  多行 TOML 数组。
- `exist` 放命令名，不放复杂 shell。
- `test` 放短命令，不依赖网络和大型数据。
- version 检查尽量精准，但不要脆弱到上游 banner 微调就失败。
- 对于包含 Python/R/Perl 环境的工具，import check 很有价值。
- 对于包含多个上游辅助脚本的工具，应把关键脚本纳入 smoke。
- `taf check` 只检查 smoke 元数据，不运行容器。真正 smoke 由 Hub/index 自动化执行。

## 版本与 release 策略

版本判断：

```text
<upstream-version>-r<taffish-release>
```

何时保持同一个 release：

- 只是修复 index、taf、官网或文档系统 bug。
- 只是重新运行 index action。
- 没有改变 app 仓库中已发布 tag 的内容。

何时增加 release：

- Dockerfile 变了。
- `src/main.taf` 变了。
- `taffish.toml` 中影响安装、运行、容器或 smoke 的字段变了。
- README/help/release 需要对应一次新的正式 app 发布。
- 同一上游版本的 TAFFISH 包装需要修复并重新发布。

何时增加 upstream version：

- 上游软件版本变了，例如 `1.2.7` 到 `1.2.8`。

不要覆盖已发布 tag。官方 index 信任模型依赖 release ref、source commit、容器 digest 和
smoke/trust 记录的稳定性。

## 发布前检查

最低检查：

```sh
taf check
taf compile -- --help
taf publish --release --dry-run
```

容器检查：

```sh
docker build --check -f docker/Dockerfile .
docker build -t ghcr.io/taffish/my-tool:1.0.0-r1 -f docker/Dockerfile .
docker run --rm ghcr.io/taffish/my-tool:1.0.0-r1 my-tool --help
```

如果本地使用 Podman：

```sh
podman run --rm ghcr.io/taffish/my-tool:1.0.0-r1 my-tool --help
```

检查清单：

- `taffish.toml` 无 `TODO`。
- `docs/help.md` 存在并与 command/image/version 一致。
- `README.md` 与 command/image/version 一致。
- `release.md` 第一行真实、简洁、无默认占位。
- `src/main.taf` image tag 与 `[container].image` 一致。
- Dockerfile 构建出的镜像包含 smoke 中声明的所有命令。
- `taf publish --release --dry-run` 的 tag 和 message 正确。

## 发布后检查

发布后流程：

1. 等待 app 仓库的 image build workflow 成功。
2. 确认 GHCR package 可被 index workflow 读取。
3. 重新运行或等待 `taffish-index` workflow。
4. 查看 `index/reports/latest.json`。
5. 确认 app 出现在 `index/index.json`、`index/packages/<name>.json` 和命令索引中。
6. 本地运行 `taf update`、`taf info <app>`、`taf install <app>`。

如果 index 失败，先区分：

- App metadata 或 smoke 本身有问题。
- 镜像没有构建成功或 GHCR 不可读。
- Index builder 或 TOML/parser/trust gate 有 bug。

不要在未确认根因时急着发新 app release。Index bug 应修 index，app bug 才修 app。

## 不要做的事

不要：

- 在 `taffish.toml` 中添加工具链不理解的字段。
- 用 `latest` 作为正式 image tag。
- 覆盖已发布 tag。
- 因为 index bug 升 app release。
- 手工编辑 `target/` 构建产物。
- 为了 smoke 通过删除用户可能需要的功能文件。
- 写只能通过 smoke、不能支持真实用户工作流的 Dockerfile。
- 让 smoke 依赖网络、远程数据库、用户凭证或大型测试数据。
- 把上游文档大段复制进 `docs/help.md`。
- 在每个 app README 中混合维护中英文长文。

## Augustus 模板要点

`augustus` 是当前官方 Hub 的首个正式 app，适合作为模板，因为它体现了这些原则：

- `taffish.toml` 接近 `taf new` 结构，只保留工具链理解的字段。
- `src/main.taf` 是薄 wrapper，直接委托上游 `augustus`。
- Dockerfile 保留用户可能需要的 `augustus-data` 和 `augustus-doc`。
- Smoke 先验证功能入口，再验证严格版本。
- `docs/help.md` 说明默认调用、`--`、command mode 和容器信息。
- `README.md` 说明安装、使用、包元数据、容器内容、上游来源和维护检查。
- `release.md` 首行可直接作为 publish message，正文可作为 GitHub Release notes。

之后新增 app 时，应先让它达到 `augustus` 这个标准，再考虑是否需要更复杂的 wrapper、
更丰富的测试或更特殊的容器设计。
