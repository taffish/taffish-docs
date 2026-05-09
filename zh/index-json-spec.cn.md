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
- [`commands`](#commands)
- [`repositories`](#repositories)
- [`warnings`](#warnings)
- [依赖语义](#依赖语义)
- [upstream 省略规则](#upstream-省略规则)
- [兼容性原则](#兼容性原则)

## 文件位置

`taffish-index` 仓库生成：

```text
index/index.json
index/packages/<package>.json
index/commands/<command>.json
```

用户默认下载：

```text
https://raw.githubusercontent.com/taffish/taffish-index/main/index/index.json
```

TAFFISH `0.3.0` 可以从运行时配置读取其他 index URL，也可以用
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
  "warnings": 0
}
```

字段：

- `packages`：package 数。
- `versions`：version record 总数。
- `commands`：command 数。
- `repositories`：仓库数。
- `warnings`：警告数。

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
  "paths": {
    "main": "src/main.taf",
    "help": "docs/help.md",
    "dockerfile": "docker/Dockerfile"
  },
  "container": {
    "image": "ghcr.io/taffish/blast:2.16.0-r1",
    "dockerfile": "docker/Dockerfile",
    "image_tag": "2.16.0-r1",
    "image_tag_matches_version": true
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
| `paths` | object | 项目内路径。 |
| `container` | object/null | 容器信息。 |
| `source` | object | app 仓库来源信息。 |
| `upstream` | object | 可选，上游软件来源。 |

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
- 不把缺失 `upstream` 显示成“无来源”。
- 不把依赖数组解释成 alternatives。
- 使用 `source.ref` 和 `source.commit` 追踪 app 自身来源。
- 使用 `upstream` 追踪被包装软件来源。
