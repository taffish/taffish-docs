# 什么是 TAFFISH Hub

TAFFISH Hub 是 TAFFISH 生态的 app 索引与分发层。它的目标是让用户可以通过：

```sh
taf update
taf install <app>
```

获得可复现、可版本化、可追踪来源的生信工具和流程。

当前 TAFFISH Hub 主要依赖 GitHub 完成自动化，不需要独立后端服务器。它由三个核心部分组成：

- `taffish-index`：静态索引仓库，生成机器可读的 index JSON。
- `taffish.github.io`：静态网页，展示 index 中的 app。
- `.github`：组织首页仓库，控制 `github.com/taffish` 的 Overview 内容。

现在可以通过 [taffish.github.io](https://taffish.github.io) 在线查看 TAFFISH Hub。这个网页读取 `taffish-index` 的静态 JSON，因此它展示的是当前已被官方索引发现的 app。

如果想查看官方 flow family 的面向人路线图，可以打开
[TAFFISH Flow Portal](https://taffish.github.io/flows/)。如果想查看当前最完整的公开
flow-family 案例，可以打开 [RNA-seq Flow Family](https://taffish.github.io/rnaseq-flows/)
门户。

另外，每个真正的 app 仓库也会有自己的 GitHub Actions，例如基于 Dockerfile 构建 GHCR 镜像。这部分由 app 仓库自己负责，不由 `taffish-hub` 统一代管。

## 目录

- [为什么需要 Hub](#为什么需要-hub)
- [访问与发布权限](#访问与发布权限)
- [当前仓库布局](#当前仓库布局)
- [Flow 门户](#flow-门户)
- [数据流](#数据流)
- [`taffish-index`](#taffish-index)
- [app 发现规则](#app-发现规则)
- [index 数据模型](#index-数据模型)
- [依赖信息](#依赖信息)
- [平台约束](#平台约束)
- [Smoke 与 Trust 元数据](#smoke-与-trust-元数据)
- [发现元数据](#发现元数据)
- [上游来源信息](#上游来源信息)
- [index metadata overrides](#index-metadata-overrides)
- [`taffish.github.io`](#taffishgithubio)
- [`.github`](#github)
- [app 镜像构建](#app-镜像构建)
- [用户如何被服务](#用户如何被服务)
- [开发者如何发布 app](#开发者如何发布-app)
- [镜像与内部来源支持](#镜像与内部来源支持)
- [当前边界](#当前边界)

## 为什么需要 Hub

TAFFISH app 是分散在多个 GitHub 仓库中的。用户不应该手动知道每个仓库、每个 tag、每个 Docker 镜像、每个命令名。

Hub 提供一个中心索引，把分散信息整理成：

- 哪些 app 可用
- 每个 app 有哪些版本
- 最新版本是什么
- app 是 tool 还是 flow
- 安装命令是什么
- app 的 GitHub 仓库在哪里
- app 的 release tag 和 commit 是什么
- 是否有容器镜像
- 容器化版本是否通过 smoke/trust 元数据检查
- 是否有依赖
- 有哪些平台约束
- 搜索和浏览用发现元数据
- 包装的上游软件来源是什么

用户只需要更新本地 index，然后按名称安装。

## 访问与发布权限

TAFFISH Hub 的网页和静态索引面向用户公开：任何用户都可以访问 [taffish.github.io](https://taffish.github.io)，也可以通过 `taf update` 下载公开 index。

但官方 TAFFISH Hub 的 app 发布和维护不是开放自助提交模式。当前只有 `taffish` GitHub 组织成员可以在官方组织下创建、发布和维护 app 仓库；目前官方维护者暂为单人维护。如果开发者希望把自己的 app 纳入官方 TAFFISH Hub，需要先联系维护者申请加入 `taffish` 组织，或由维护者协助完成审核与发布。

这样设计是为了保证早期生态中的 app 元数据、版本 tag、Docker 镜像、许可证、upstream 来源和依赖关系尽量一致。未来如果维护者变多，可以再设计更正式的 review、ownership 和贡献流程。

## 当前仓库布局

在本地 `taffish-hub` 工作区中，所有要发布到 GitHub 的仓库都先放在 `repos/` 下：

```text
taffish-hub/
  README.md
  docs/
    README.md
    zh/
      taffish.cn.md
      taffish-hub.cn.md
  repos/
    taffish-index/
    taffish.github.io/
    .github/
```

这种结构的意思是：`taffish-hub` 是本地工厂；`repos/<name>` 才是未来或已经发布到 GitHub 的真实仓库根目录。

当前主要仓库：

```text
repos/taffish-index       -> github.com/taffish/taffish-index
repos/taffish.github.io   -> github.com/taffish/taffish.github.io
repos/.github             -> github.com/taffish/.github
```

一些公开文档和案例仓库也可以放在它描述的业务域下面。例如：

```text
repos/apps/bio/flows/rna-seq/rnaseq-flows -> github.com/taffish/rnaseq-flows
```

## Flow 门户

[TAFFISH Flow Portal](https://taffish.github.io/flows/) 属于 `taffish.github.io`。它是
官方 flow family、路线页面、示例报告和 Hub 链接的面向人地图。它不替代机器可读
index，而是解释由已索引 `taf-*` 命令组成的策划型路线。

`taffish/rnaseq-flows` 是更深入的 TAFFISH RNA-seq flow family 静态文档与示例报告
门户。它本身不是 taf-flow package，而是用来说明多个已发布 flow app 如何连接，
集中链接每个 flow 的手册，并展示公开 yeast 报告：

- [TAFFISH Flow Portal](https://taffish.github.io/flows/)
- [RNA-seq Flow Family 门户](https://taffish.github.io/rnaseq-flows/)
- [Yeast 标准报告](https://taffish.github.io/rnaseq-flows/examples/yeast-standard-report/04_reports/rnaseq_report.html)
- [报告解读手册](https://taffish.github.io/rnaseq-flows/examples/yeast-standard-report/04_reports/report_interpretation.html)

当一组 `taf-*` 命令形成了完整分析路线，并且需要面向人的手册、图示和示例输出时，
这种门户可以作为机器可读 Hub index 之外的解释层。

## 数据流

从 app 开发者到用户安装，大致流程是：

```text
开发者创建 TAFFISH app
        |
        v
app 仓库发布 release tag: v<version>-r<release>
        |
        v
taffish-index GitHub Actions 扫描 taffish 组织
        |
        v
生成 index/index.json
        |
        +----> taffish.github.io 网页读取并展示
        |
        +----> 用户 taf update 下载到本地
                         |
                         v
                  taf install 解析并安装 app
```

这个过程里没有中心数据库，也没有常驻后端服务。GitHub 仓库本身就是数据源，GitHub Actions 负责定时生成静态索引，GitHub Pages 负责展示网页。

## `taffish-index`

`taffish-index` 是一个静态索引仓库。它的核心文件是：

```text
index/index.json
index/packages/<package>.json
index/commands/<command>.json
```

其中 `index/index.json` 是完整索引，`packages/` 和 `commands/` 是拆分后的辅助索引。

索引生成器使用 Common Lisp 编写，入口是：

```sh
sbcl --script scripts/build-index.lisp -- --org "taffish" --output index
```

本地测试时可以不扫描 GitHub，只扫描本地项目：

```sh
sbcl --script scripts/build-index.lisp -- --no-org --local-repo ../../../taffish/test/my-test-tool --output index
```

### 自动化

`taffish-index` 的 GitHub Actions 位于：

```text
.github/workflows/build-index.yml
```

触发方式：

- 手动触发：`workflow_dispatch`
- 定时触发：每天一次

当前定时任务：

```yaml
schedule:
  - cron: "17 1 * * *"
```

这是 UTC 时间。工作流会：

1. checkout `taffish-index` 仓库
2. 安装 SBCL
3. 执行 `scripts/build-index.lisp`
4. 扫描 `taffish` 组织中的仓库
5. 生成 `index/`
6. 如果 index 有变化，则自动 commit 并 push

默认使用 `GITHUB_TOKEN` 扫描公开仓库。如果需要扫描私有仓库，可以设置：

```text
TAFFISH_BOT_TOKEN
```

## app 发现规则

一个 GitHub 仓库会被视为 TAFFISH app，需要满足：

- 仓库根目录存在 `taffish.toml`
- `taffish.toml` 中必需 section 和字段合法
- `[package].main` 指向存在的 `.taf` 文件
- `docs/help.md` 存在
- `[repository].url` 指向当前被扫描的 GitHub 仓库
- release tag 使用 `v<version>-r<release>` 格式

例如：

```text
v0.1.0-r1
v0.1.0-r2
v1.0.0-r1
```

索引生成器优先扫描 release tag。默认分支可以作为开发快照参与扫描，但需要手动触发时开启 `include_default_branch`。

## index 数据模型

完整 index 的顶层结构大致是：

```json
{
  "schema_version": "taffish.index/v1",
  "generated_at": "2026-05-08T00:00:00Z",
  "organization": "taffish",
  "counts": {},
  "packages": {},
  "commands": {},
  "repositories": {},
  "warnings": []
}
```

每个 package 包含：

```json
{
  "name": "my-tool",
  "latest": "0.1.0-r1",
  "repository_url": "https://github.com/taffish/my-tool",
  "command": {
    "name": "taf-my-tool"
  },
  "versions": {}
}
```

每个 version record 包含：

```json
{
  "name": "my-tool",
  "kind": "tool",
  "version": "0.1.0",
  "release": 1,
  "version_id": "0.1.0-r1",
  "tag": "v0.1.0-r1",
  "license": "Apache-2.0",
  "repository_url": "https://github.com/taffish/my-tool",
  "repository_slug": "taffish/my-tool",
  "command": {
    "name": "taf-my-tool"
  },
  "runtime": {
    "pipe": true,
    "command_mode": true
  },
  "dependencies": {},
  "platform": {
    "os": [],
    "arch": [],
    "container": "optional",
    "min_cpus": null,
    "min_memory_mb": null
  },
  "meta": {
    "domain": "bioinformatics",
    "category": "sequence-alignment",
    "categories": ["sequence-alignment"],
    "keywords": ["blast", "alignment", "sequence-search"],
    "summary": "BLAST+ wrapper for sequence similarity search.",
    "description": "BLAST+ wrapper for sequence similarity search."
  },
  "paths": {
    "main": "src/main.taf",
    "help": "docs/help.md",
    "dockerfile": "docker/Dockerfile"
  },
  "container": {
    "image": "ghcr.io/taffish/my-tool:0.1.0-r1",
    "dockerfile": "docker/Dockerfile",
    "image_tag": "0.1.0-r1",
    "image_tag_matches_version": true,
    "digest": "sha256:manifest-list-digest",
    "platforms": ["linux/amd64", "linux/arm64"],
    "platform_digests": {
      "linux/amd64": "sha256:...",
      "linux/arm64": "sha256:..."
    }
  },
  "smoke": {
    "backend": "docker",
    "timeout": 60,
    "exist": ["my-tool"],
    "test": ["my-tool --help"],
    "status": "passed",
    "checked_at": "2026-05-12T08:00:00Z",
    "backend_used": "docker"
  },
  "trust": {
    "status": "passed",
    "checked_at": "2026-05-12T08:00:00Z",
    "policy": "taffish.index/trust-v1",
    "source": "taffish-index"
  },
  "source": {
    "repository": "taffish/my-tool",
    "ref": "v0.1.0-r1",
    "commit": "...",
    "html_url": "https://github.com/taffish/my-tool/tree/v0.1.0-r1"
  }
}
```

如果 app 提供 `[upstream]`，还会有：

```json
{
  "upstream": {
    "name": "CD-HIT",
    "type": "github",
    "url": "https://github.com/weizhongli/cdhit",
    "homepage": "https://github.com/weizhongli/cdhit",
    "repository": "weizhongli/cdhit",
    "version": "4.8.1",
    "license": "GPL-2.0",
    "doi": "10.1093/bioinformatics/bts565",
    "pmid": "23060610"
  }
}
```

如果没有提供有效 upstream 信息，`upstream` 字段会被省略，而不是写成 `null` 或 `none`。

## 依赖信息

Flow app 可以在 `taffish.toml` 中声明依赖：

```toml
[dependencies]
taf-dep-tool = "0.1.0-r1"
taf-x = ["0.1.0-r1", "0.1.0-r2"]
```

index 会导出为：

```json
{
  "dependencies": {
    "taf-dep-tool": "0.1.0-r1",
    "taf-x": ["0.1.0-r1", "0.1.0-r2"]
  }
}
```

这不是“多个备选版本”，而是“这个 flow 可能需要多个不同版本”。`taf install` 会按照 index 中的信息安装依赖。

## 平台约束

app 可以声明平台约束：

```toml
[platform]
os = "linux,darwin"
arch = "amd64,arm64"
container = "required"
min_cpus = 2
min_memory_mb = 4096
```

这些信息会进入 index，用于后续安装器、网页和用户判断。

当前 `container` 支持：

```text
optional
required
forbidden
```

## Smoke 与 Trust 元数据

容器化 app 应该在 `taffish.toml` 中声明 `[smoke]`：

```toml
[smoke]
backend = "docker"
timeout = 60
exist = ["my-tool"]
test = ["my-tool --help"]
```

本地 `taf check` 会校验这部分元数据，并拒绝默认 `TODO` 占位，但不会运行 smoke
tests。TAFFISH Hub/index 自动化会在最终镜像推送后，对新的容器化版本运行 smoke
checks，并记录镜像 digest 和平台列表。

主 index 策略：

- 已经接受过的版本，如果 release tag 仍指向同一 commit，则复用旧记录。
- 新的非容器版本可以以 `trust.status = "not_applicable"` 进入 index。
- 新的容器化版本只有通过 digest/platform 检查和 smoke checks 后才进入主 index。
- 失败结果写入 `index/reports/latest.json` 和带时间戳的 reports，不进入稳定主 index。

## 发现元数据

`[meta]` 描述 TAFFISH app 的公开发现信息。它对本地开发是可选的，但对官方 Hub
package 很有用。

推荐 app 侧写法：

```toml
[meta]
domain = "bioinformatics"
category = "sequence-alignment"
summary = "BLAST+ wrapper for sequence similarity search."
keywords = ["blast", "alignment", "sequence-search"]
```

从 TAFFISH `0.8.1` 开始，更简洁的 app 侧字段 `category` 和 `summary` 已被文档化。
index 也兼容 `categories` 和 `description`，并在 JSON 中同时输出两种形式以适配 Hub 消费端。

## 上游来源信息

`[upstream]` 描述 app 包装的原始软件来源，例如 BLAST、CD-HIT、BWA 等。

示例：

```toml
[upstream]
name = "BLAST+"
type = "official"
url = "https://blast.ncbi.nlm.nih.gov/"
homepage = "https://blast.ncbi.nlm.nih.gov/"
version = "2.16.0"
```

或：

```toml
[upstream]
name = "CD-HIT"
type = "github"
url = "https://github.com/weizhongli/cdhit"
repository = "weizhongli/cdhit"
version = "4.8.1"
```

支持字段：

```text
name
type
url
homepage
repository
release_url
docker_image
version
license
citation
doi
pmid
```

这个信息对三件事很重要：

- 用户知道 taf app 包装的是哪个原始工具。
- 维护者可以基于 upstream 检查新版本。
- 生信工具可以保留引用、许可证和论文信息。

## index metadata overrides

已发布 app release 应视为不可变。如果旧的已收录版本缺少发现元数据，或缺少已经声明的
upstream 仓库的归属/引用信息，维护者不应该重写已发布 tag，也不应该只为了这些展示信息
bump app release。官方 index 可以通过 `metadata-overrides.toml` 补充这些信息。

这个机制只用于 index 侧元数据，例如 `meta` 和上游归属/引用字段（`license`、
`citation`、`doi` 和 `pmid`）。它不会创建新的 upstream object，也不改变 install
命令、source commit、容器镜像、镜像 digest、smoke 结果或 trust 状态。新的 app
release 仍然应该在自己的 `taffish.toml` 中写完整的 `[meta]` 和 `[upstream]`。

## `taffish.github.io`

`taffish.github.io` 是静态网页仓库。它从以下地址读取 index：

```text
https://raw.githubusercontent.com/taffish/taffish-index/main/index/index.json
```

它提供：

- 中英文界面
- app 搜索
- tool / flow 筛选
- 依赖筛选
- 容器镜像筛选
- 名称 / 最新版本排序
- package 详情页
- 版本列表
- 依赖列表
- 平台约束
- 发现元数据
- upstream 来源展示
- install 命令复制
- dependency-aware install chain
- index 同步失败提示

它不需要后端服务器。GitHub Pages 直接托管静态文件：

```text
index.html
styles.css
app.js
```

## `.github`

`.github` 是 GitHub 组织主页仓库。GitHub 会把：

```text
profile/README.md
```

展示在：

```text
https://github.com/taffish
```

它适合放组织入口、Hub 入口、索引仓库入口和当前项目状态。

## app 镜像构建

镜像构建不属于 `taffish-index` 的职责，也不属于 `taffish.github.io` 的职责。

每个带 Dockerfile 的 app 应该在自己的仓库里拥有 GitHub Actions，例如：

```text
app-repo/
  Dockerfile 或 docker/Dockerfile
  .github/workflows/build-image.yml
```

`taf new --tool --docker` 可以生成带 Dockerfile 和 GitHub Actions workflow 的项目骨架。

这样设计的原因是：

- 每个 app 的构建环境不同。
- 每个 app 的 Dockerfile 应该和 app 源码一起版本化。
- 每个 app 的 release tag 应该对应自己的镜像 tag。
- Hub 只负责发现和索引，不负责替 app 构建。

## 用户如何被服务

用户侧的核心路径：

```sh
taf update
taf search blast
taf install blast
taf outdated
taf upgrade
taf list
taf which taf-blast-v2.16.0-r1
```

`taf update` 会下载静态 index，写入本地缓存：

```text
index/current.json
index/snapshots/
```

`taf install` 会：

1. 读取本地 `index/current.json`
2. 根据 package name、command name 或 exact versioned command 解析目标
3. 找到对应 version record
4. 安装依赖
5. clone 对应 source tag
6. 构建版本化 command wrapper
7. 写入本地 app 数据和命令路径

从 TAFFISH `0.10.0` 开始，本地包维护命令也会消费同一份本地
index 和安装元数据：

- `taf install --all` 用于预览或安装全部已索引 app，也可以限制为 tools 或 flows。
- `taf outdated` 用于报告本地已安装 Hub app 中有更新 index 版本的项目。
- `taf upgrade` 用于预览或执行已安装 Hub app 的升级。
- `taf prune` 用于删除较旧本地版本，并保留本地已安装的最新版本。

这些都是本地操作，不会修改公开 Hub index。通过 `taf install --from` 安装的
本地/私有 app 也不会被公开 index 中的版本静默替换。

安装命令可以是：

```sh
taf install my-tool
taf install my-tool 0.1.0-r1
taf install taf-my-tool
taf install taf-my-tool v0.1.0-r1
taf install taf-my-tool-v0.1.0-r1
```

对于尚未发布到公开 Hub 的私有/本地 app 项目，用户可以绕过 index，直接从项目目录安装：

```sh
taf install --from /path/to/my-private-tool
```

这个模式会读取本地 `taffish.toml`、检查项目、把当前工作树复制到 TAFFISH home，
并构建带版本的 wrapper。它不需要先运行 `taf update`，也不会自动安装依赖。

## 开发者如何发布 app

典型路径：

```sh
taf new my-tool --tool --docker
cd my-tool
taf check
taf run -- --help
taf build --all
taf publish --release --dry-run
taf publish --release --yes --build
```

发布前，维护者应编辑 `taf new` 创建的、被 ignore 的 `release.md` 草稿。使用
`taf publish --release` 时，它的第一行会成为 publish message，整个文件会成为
GitHub Release notes。

发布后，app 仓库应该有：

```text
taffish.toml
src/main.taf
docs/help.md
release tag: v<version>-r<release>
```

如果是容器化 app，还应该有：

```text
docker/Dockerfile
GHCR image: ghcr.io/taffish/<app>:<version>-r<release>
taffish.toml 中的 [smoke]
```

当 `taffish-index` 的自动化下一次运行后，非容器 app 和通过可信 gate 的容器化 app 会被写入主 index。
失败的新容器化版本会进入维护者报告。用户随后运行 `taf update` 即可安装已接受的版本。

## 镜像与内部来源支持

TAFFISH Hub 仍然是 GitHub 优先的静态 index，而本地 `taf` 运行时镜像配置从
TAFFISH `0.2.0` 开始支持。因此用户不需要修改官方 index schema，也可以使用镜像源。

默认 GitHub 路径是：

```text
taf update -> 官方 index URL
taf install -> index 中的 canonical app 仓库 URL
```

对于中国大陆用户或内部网络用户，`config.toml` 可以覆盖这两条访问路径：

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

`[index].url` 会改变 `taf update` 读取静态 index 的位置。`[[source.rewrite]]`
会让官方 index 继续保留 canonical GitHub URL，同时允许 `taf install` 从镜像或
内部 Git 服务 clone app。

这带来两个实际结果：

- 官方 Hub 可以继续把 GitHub 作为 canonical 发布来源。
- 镜像维护者可以同步 index 和 app 仓库，而不需要改 app 元数据；前提是镜像端保留
  兼容的仓库、tag 和相同的 TAFFISH index schema。

容器镜像是另一条链路。如果某个 app version 指向 GHCR 或其他 OCI registry，运行
app 的机器仍然需要能访问该镜像地址，或者 app 元数据需要指向用户可访问的镜像来源。

## 当前边界

当前 TAFFISH Hub 的边界是：

- 它是静态索引和静态网页，不是传统数据库后端。
- 它依赖 GitHub 仓库、release tag、GitHub Actions 和 GitHub Pages。
- 它不负责长期保存用户本地状态，用户状态由本机 `taf` 管理。
- 它不负责 app 镜像构建，镜像构建由各 app 仓库管理。
- 它可以通过本地 `taf` 运行时配置支持 index 和仓库来源镜像，但不会自动镜像容器 registry。

未来如果需要独立服务器，可以把现在的静态 index 模型迁移到数据库服务；但在当前阶段，GitHub 静态部署已经足够支撑 TAFFISH app 的发现、展示和安装。
