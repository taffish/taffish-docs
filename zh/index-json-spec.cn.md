# TAFFISH Index JSON 规范

TAFFISH Index JSON 是 TAFFISH Hub 的机器可读数据格式。`taf update` 下载它，`taf install` 读取它，`taffish.github.io` 展示它。

当前 schema version：

```text
taffish.index/v1
```

## 目录

- [文件位置](#文件位置)
- [顶层结构](#顶层结构)
- [`counts`](#counts)
- [`packages`](#packages)
- [version record](#version-record)
- [meta](#meta)
- [container、smoke 与 trust](#containersmoke-与-trust)
- [`commands`](#commands)
- [`repositories`](#repositories)
- [`warnings`](#warnings)
- [构建报告](#构建报告)
- [依赖语义](#依赖语义)
- [upstream 省略规则](#upstream-省略规则)
- [兼容性原则](#兼容性原则)

## 文件位置

`taffish-index` 仓库生成：

```text
index/index.json
index/packages/<package>.json
index/commands/<command>.json
index/reports/latest.json
index/reports/<timestamp>.json
```

用户默认下载：

```text
https://raw.githubusercontent.com/taffish/taffish-index/main/index/index.json
```

从 TAFFISH `0.2.0` 开始，`taf` 可以从运行时配置读取其他 index URL，也可以用
`taf update --url <INDEX-URL>` 做一次性覆盖。镜像 index 应该保留相同 schema 和
canonical source records。仓库 URL 重写属于本地 `taf` 配置，不属于 index schema
本身。

`index/index.json` 是完整索引，拆分文件用于更细粒度访问。

## 顶层结构

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

字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `schema_version` | string | index schema 版本。 |
| `generated_at` | string | UTC 生成时间。 |
| `organization` | string/null | 被扫描的 GitHub 组织。 |
| `counts` | object | 统计信息。 |
| `packages` | object | package 索引。 |
| `commands` | object | command 到 package 的索引。 |
| `repositories` | object | repository 到 package 的索引。 |
| `warnings` | array | 构建时警告。 |

## `counts`

```json
{
  "packages": 12,
  "versions": 27,
  "commands": 12,
  "repositories": 12,
  "warnings": 0,
  "failed": 0
}
```

字段：

- `packages`：package 数。
- `versions`：version record 总数。
- `commands`：command 数。
- `repositories`：仓库数。
- `warnings`：警告数。
- `failed`：写入最新构建报告的 trust-gate 失败数。

## `packages`

`packages` 是一个 object，key 是 package name。

```json
{
  "blast": {
    "name": "blast",
    "latest": "2.16.0-r1",
    "repository_url": "https://github.com/taffish/blast",
    "command": {
      "name": "taf-blast"
    },
    "versions": {}
  }
}
```

package entry 字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `name` | string | package name。 |
| `latest` | string | 最新 version id。 |
| `repository_url` | string | app GitHub 仓库。 |
| `command.name` | string | 命令基础名。 |
| `versions` | object | version id 到 version record 的映射。 |

## version record

version record 描述一个具体版本。

```json
{
  "name": "blast",
  "kind": "tool",
  "version": "2.16.0",
  "release": 1,
  "version_id": "2.16.0-r1",
  "tag": "v2.16.0-r1",
  "license": "Apache-2.0",
  "repository_url": "https://github.com/taffish/blast",
  "repository_slug": "taffish/blast",
  "command": {
    "name": "taf-blast"
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
    "image": "ghcr.io/taffish/blast:2.16.0-r1",
    "dockerfile": "docker/Dockerfile",
    "image_tag": "2.16.0-r1",
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
    "exist": ["blastn"],
    "test": ["blastn -help"],
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
    "repository": "taffish/blast",
    "ref": "v2.16.0-r1",
    "commit": "abcdef...",
    "html_url": "https://github.com/taffish/blast/tree/v2.16.0-r1"
  }
}
```

字段说明：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `name` | string | package name。 |
| `kind` | string | `tool` 或 `flow`。 |
| `version` | string | app version。 |
| `release` | number | app release。 |
| `version_id` | string | `<version>-r<release>`。 |
| `tag` | string | Git tag，格式 `v<version>-r<release>`。 |
| `license` | string/null | app 包装层许可证。 |
| `repository_url` | string | app 仓库 URL。 |
| `repository_slug` | string | `owner/repo`。 |
| `command.name` | string | command 基础名。 |
| `runtime.pipe` | boolean | 是否支持 pipe。 |
| `runtime.command_mode` | boolean | 是否是 command mode。 |
| `dependencies` | object | flow 依赖。 |
| `platform` | object | 平台约束。 |
| `meta` | object/null | 可选，搜索、筛选和展示用发现元数据。 |
| `paths` | object | 项目内路径。 |
| `container` | object/null | 容器信息。 |
| `smoke` | object/null | 容器化 app 的 smoke 元数据和结果。 |
| `trust` | object/null | Hub/index 可信 gate 状态。 |
| `source` | object | app 仓库来源信息。 |
| `upstream` | object | 可选，上游软件来源。 |

## meta

`meta` 记录来自 `taffish.toml` 中 `[meta]` 或 index 维护者 override 的可选发现
元数据。它用于搜索、分类和 Hub 展示；安装器应该把它视为可选字段。

当前字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `domain` | string/null | 大领域，例如 `bioinformatics`。 |
| `category` | string/null | 主分类 token。 |
| `categories` | array | 用于筛选和浏览的分类 token。 |
| `keywords` | array | 搜索关键词和别名。 |
| `summary` | string/null | 一句话描述。 |
| `description` | string/null | 同一段面向用户的描述，为 Hub 兼容保留。 |

TAFFISH `0.8.1` 文档化了更简洁的 app 侧字段 `category` 和 `summary`。
`taffish-index` 也兼容更丰富的 Hub 侧别名 `categories` 和 `description`，
并在生成 record 时同时输出两种形式，方便新旧消费者读取。

## container、smoke 与 trust

`container` 字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `image` | string | 容器镜像引用。 |
| `dockerfile` | string/null | 项目内 Dockerfile 路径。 |
| `image_tag` | string/null | 从 `image` 解析出的镜像 tag。 |
| `image_tag_matches_version` | boolean | 镜像 tag 是否与 `version_id` 一致。 |
| `digest` | string/null | index builder 记录的 manifest-list digest。 |
| `platforms` | array | 镜像支持的平台，例如 `linux/amd64`。 |
| `platform_digests` | object | 可选，platform 到 per-platform digest 的映射。 |

`smoke` 字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `backend` | string | 请求的 smoke backend：`docker`、`podman` 或 `apptainer`。 |
| `timeout` | number | 每条 smoke command 的超时时间，单位秒。 |
| `exist` | array | 应在容器 `PATH` 中存在的可执行命令。 |
| `test` | array | 应以退出码 `0` 结束的命令。 |
| `status` | string/null | 主 index 中通常为 `passed`。 |
| `checked_at` | string/null | UTC smoke 检查时间。 |
| `backend_used` | string/null | index builder 实际使用的 backend。 |

官方 index 当前使用的 `trust.status`：

- `passed`：容器 digest/platform 检查和 smoke checks 已通过。
- `not_applicable`：非容器 app version，不适用容器 smoke gate。

失败的新版本不会写入主 index，而是进入构建报告。

## `commands`

`commands` 是 command 到 package 和 latest version 的索引。

```json
{
  "taf-blast": {
    "package": "blast",
    "version": "2.16.0-r1"
  }
}
```

用途：

- `taf install taf-blast`
- `taf info taf-blast`
- `taf search`

## `repositories`

`repositories` 是 GitHub 仓库到 packages 的索引。

```json
{
  "taffish/blast": {
    "repository": "taffish/blast",
    "packages": ["blast"]
  }
}
```

一个仓库通常对应一个 package，但格式允许一个仓库包含多个 package。

## `warnings`

构建 index 时遇到非致命问题，会写入 `warnings`：

```json
[
  {
    "repository": "taffish/example",
    "ref": "v0.1.0-r1",
    "message": "missing docs/help.md"
  }
]
```

警告不会阻止其他合法 app 被写入 index。网页会展示这些 warnings，维护者应定期检查。

## 构建报告

index builder 还会写出 reports：

```text
index/reports/latest.json
index/reports/<timestamp>.json
```

report schema：

```json
{
  "schema_version": "taffish.index.report/v1",
  "generated_at": "2026-05-12T08:00:00Z",
  "organization": "taffish",
  "counts": {
    "failed": 1,
    "warnings": 0
  },
  "failed": [
    {
      "repository": "taffish/my-tool",
      "ref": "v0.1.0-r1",
      "version_id": "0.1.0-r1",
      "stage": "smoke",
      "message": "smoke command failed",
      "image": "ghcr.io/taffish/my-tool:0.1.0-r1"
    }
  ],
  "warnings": []
}
```

`taf update` 和 `taf install` 消费稳定主 index，而不是 report 文件。Reports
主要面向维护者和未来 Hub UI 诊断。

## 依赖语义

无依赖：

```json
"dependencies": {}
```

单版本依赖：

```json
"dependencies": {
  "taf-dep-tool": "0.1.0-r1"
}
```

多版本依赖：

```json
"dependencies": {
  "taf-x": ["0.1.0-r1", "0.1.0-r2"]
}
```

多版本依赖表示多个版本都需要安装，不表示备选版本。

`latest` 和 `*` 可以被安装器视作未指定版本，但官方 flow 推荐写 exact version id。

## upstream 省略规则

如果 `taffish.toml` 没有 `[upstream]`，或 `[upstream]` 没有任何有效字段，则 version record 中不出现 `upstream`。

有 upstream 时：

```json
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
```

消费者不应假设 `upstream` 一定存在。

## 兼容性原则

Index JSON 消费者应该：

- 检查 `schema_version`。
- 对未知字段保持宽容。
- 对缺失可选字段保持宽容。
- 把 `meta` 视为可选发现元数据。
- 兼容读取 `category`/`summary` 和 `categories`/`description`。
- 不把缺失 `upstream` 显示成“无来源”。
- 不把依赖数组解释成 alternatives。
- 使用 `source.ref` 和 `source.commit` 追踪 app 自身来源。
- 使用 `upstream` 追踪被包装软件来源。
- 把缺失的 `smoke` 或 `trust` 视为未知/旧元数据，而不是失败证明。
- 把 reports 与主安装 index 分开读取。
