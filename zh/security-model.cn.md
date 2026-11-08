# TAFFISH 安全模型

[English](../en/security-model.en.md) | [中文](security-model.cn.md)

这份文档总结当前 TAFFISH 的安全模型。它说明源码、release 载荷、安装器、Hub
index 元数据、容器检查、本地安装和 MCP 工具之间如何组成一条可信链。

这不是“TAFFISH 已经解决全部供应链安全问题”的声明。当前模型把 Hub
smoke/trust 元数据与保守的本地维护命令组合起来；整个模型仍然分层、可审计，
并且可以逐步增强。

## 目录

- [范围](#范围)
- [源码与漏洞报告](#源码与漏洞报告)
- [Release 载荷完整性](#release-载荷完整性)
- [安装器与镜像行为](#安装器与镜像行为)
- [Hub Index 可信 Gate](#hub-index-可信-gate)
- [本地安装校验](#本地安装校验)
- [容器运行边界](#容器运行边界)
- [MCP 与 AI 安全边界](#mcp-与-ai-安全边界)
- [用户建议](#用户建议)
- [当前限制](#当前限制)
- [继续阅读](#继续阅读)

## 范围

TAFFISH 有几个主要安全表面：

- 本地 `taffish`、`taf` 和 `taffish-mcp` 命令；
- release 二进制和安装脚本；
- TAFFISH Hub index 数据；
- taf-app 源码仓库；
- taf-app 使用的容器镜像；
- 暴露给 AI 客户端的 MCP 工具。

安全模型的目标是让这些表面保持明确：用户应该知道哪些内容会被校验，哪些内容只是
记录为元数据，哪些内容仍然依赖 GitHub、Gitee、GHCR、Docker、Podman、
Apptainer、Git 或操作系统等外部系统。

## 源码与漏洞报告

本地 TAFFISH CLI/编译器实现已经在
[taffish/taffish](https://github.com/taffish/taffish) 中开源，使用 Apache
License 2.0 授权。

安全问题报告应遵循源码仓库策略：

- [Security Policy](https://github.com/taffish/taffish/blob/main/SECURITY.md)

该策略覆盖私密漏洞报告方式，以及 command 生成、shell 转义、安装器行为、source
verification、package index 元数据、MCP 安全边界和 release artifact 完整性等问题。

## Release 载荷完整性

当前带签名的 TAFFISH release 载荷包含：

```text
target/SHA256SUMS
target/SHA256SUMS.asc
target/TAFFISH-RELEASE-KEY.asc
```

使用 **0.11.0 安装器**从官方 GitHub/Gitee raw 来源安装时，默认 `auto` 模式要求
每个选中的二进制与 `SHA256SUMS` 匹配。签名、公钥、`gpg` 和 `gpgv`
均可用时，还会按固定的公钥指纹验证清单签名。`--verify gpg` 强制两项均通过；
`--verify sha256` 只要求摘要校验。显式 `--verify none` 会关闭校验，不是等价的
回退方案。安装收据 `install-receipt.toml` 记录实际校验等级。如果该路径已存在，它必须是普通文件；初装和重装均拒绝符号链接，包括目标不存在的链接。旧安装器以其自身版本的行为为准。

典型校验流程：

```sh
cd target
shasum -a 256 -c SHA256SUMS
gpg --import TAFFISH-RELEASE-KEY.asc
gpg --verify SHA256SUMS.asc SHA256SUMS
```

需要明确的边界：

- 载荷摘要或签名不能认证用 `curl | sh` 获取的安装脚本本身；自定义二进制 URL
  和 bundle 不属于官方 raw 路径，不能假定有相同的校验材料；
- 当前 release 流程还不是可复现构建声明；
- 当前 release 流程还不是 GitHub Actions provenance 或 artifact attestation 声明。

高安全环境应从独立可信渠道取得固定 tag 的安装器和发布公钥指纹，先审查脚本，并在
信任公钥前比对指纹、安装前验证载荷。

## 安装器与镜像行为

安装脚本通常从 `main` 下载，但实际安装的二进制由 `--version` 指定的固定 release
tag 选择。

示例：

```sh
curl -fsSL https://raw.githubusercontent.com/taffish/taffish/main/install/install-taffish.sh | sh -s -- --version X.Y.Z --user
```

请把 `X.Y.Z` 替换为要校验并安装的 release 版本。

这样固定的是二进制载荷，而不是从 `main` 获取的脚本。两者都需要固定时，应使用
从固定 tag 取得并已审查的安装脚本，再显式指定载荷版本。

Gitee 安装器和中国镜像 profile 用于改善 GitHub raw URL 较慢或被阻断时的访问体验。
镜像配置可以重写：

- `taf update` 使用的 index URL；
- `taf install` 使用的 app 仓库 clone URL。

镜像配置不会自动镜像容器 registry。容器镜像通常位于 GHCR，运行 app 的机器仍然
需要能访问对应镜像来源。

## Hub Index 可信 Gate

TAFFISH Hub 使用 `taffish-index` 生成的静态 index。本地命令会消费这个 index：

```sh
taf update
taf search
taf info
taf install
```

对于每个 app version，index 可以记录：

- app 仓库和 release tag；
- `source.commit`；
- dependency 元数据；
- platform 约束；
- container image 引用；
- container digest 和 platform 元数据；
- `[smoke]` 元数据和 smoke 结果；
- `trust.status`。

对于新的容器化版本，预期 index pipeline 是：

1. 扫描 app 仓库和 release tag。
2. 读取 `taffish.toml`。
3. 校验必要 app 元数据。
4. 检查容器镜像 digest 和 platforms。
5. 按策略执行必需和建议性后端的 smoke 检查。
6. 新版本通过必需检查即可收录；建议性后端失败会记录，但本身不阻止安装。

收录不等于默认安装推荐：已收录的预发布版本可显式指定安装，但不会成为默认版本。
必需和建议性检查的失败分别见：

```text
index/reports/latest.json
index/reports/<timestamp>.json
```

已经接受过的版本通常不会在每次 index 构建时重复检查。这样可以避免因为临时网络或
registry 波动导致旧版本突然从 index 消失。维护者可以在 index 逻辑变化或历史记录
需要修复时手动 force recheck。

## 本地安装校验

当本地 index 提供 `source.commit` 时，`taf install` 会在构建 command wrapper 前
校验解析到的 app source 是否处在预期 Git commit，并且对应源码工作区是否干净。

这对 source rewrite 尤其重要：

```toml
[[source.rewrite]]
from = "https://github.com/taffish/"
to = "https://gitee.com/taffish-org/"
enabled = true
```

clone URL 可以被重写到镜像，但 checkout 出来的源码仍然必须解析到 index 记录的
commit，并且不能带有未记录修改。这样镜像安装路径可以保持可审计，而不需要改写
index 里的 app source 记录。

`taf install --from <PROJECT-DIR>` 是本地/私有安装路径。它可以保留本地 smoke
元数据，但这不等于通过了公开 Hub index trust gate。

TAFFISH `0.10.0` 增加的本地包维护命令会基于本地安装元数据和当前
本地 index 工作：

```sh
taf outdated
taf upgrade
taf prune
```

`taf upgrade` 和 `taf prune` 默认保持保守，需要 `--yes` 这类显式确认才会修改
本地 app 源码或 wrapper 文件。通过 `taf install --from` 安装的本地/私有 app
不会被公开 index 中的版本静默替换。

在 0.11.0 中，`taf doctor index` 检查索引缓存，`taf doctor hub` 检查本地安装证据；
它们不会联网获取、修复或运行 app。旧安装没有构建清单是 **unverified**，不等于
已损坏；验证通过也只证明本地文件一致，不等于认证官方来源。显式修复前请先看
[故障排查](troubleshooting.cn.md)。

## 容器运行边界

TAFFISH 可以调用 Docker、Podman 或 Apptainer，但它本身不会让任意容器自动变安全。

重要运行事实：

- 容器化 app 会执行声明的容器镜像中的代码；
- TAFFISH 容器标签通常会把当前工作目录 bind mount 到容器中；
- 从 `/usr/bin`、`/usr/local/bin`、`/lib` 或 `/opt` 等宿主机系统目录运行，可能遮住镜像内部重要路径；
- 容器 registry 访问和 Git/index 镜像访问是两件事。

对 app 作者来说，smoke checks 应该：

- 短小；
- 确定；
- 适合 CI/index 自动化运行；
- 不依赖大型用户数据；
- 聚焦证明预期 executable 存在，以及一个最小命令可以运行。

对用户来说，建议从项目目录、数据目录或临时工作目录运行 TAFFISH app，而不是从宿主机
系统目录运行。

## MCP 与 AI 安全边界

`taffish-mcp` 被有意设计为保守接口。

它可以暴露只读或预览型能力，例如：

- 验证 `.taf` 源码；
- 把 `.taf` 源码编译为 shell code，但不执行；
- 摘要 `.taf` 源码；
- 检查已安装或已索引 app；
- 检查当前项目；
- 向 AI 客户端暴露 smoke/trust 元数据；
- 规划 outdated 检查、install-all、upgrade 和 prune 等本地包维护操作；
- 提供 resources 和 prompts。

它不暴露：

- `taf run`；
- `taf publish`；
- 容器镜像构建；
- smoke 执行；
- 容器启动；
- 镜像拉取。

这个边界很重要：AI 客户端可以检查和理解 TAFFISH 项目，而不会自动运行 workflow、
拉取镜像或发布变更。

## 用户建议

普通用户建议：

- 从官方 `taffish/taffish` 仓库安装；
- 需要可复现安装时固定 `--version`；
- 安装 app 前先运行 `taf update`；
- 优先使用官方 index 中出现的 app，而不是未经审核的本地拷贝；
- 当 digest、platform、smoke 或 source 元数据重要时，查看 `taf info <app>`；
- 从安全工作目录运行容器化 app。

高安全要求环境建议：

- 手动校验 release checksum 和 GPG 签名；
- 内部同步 index 和 app 仓库；
- 明确记录 source rewrite 规则；
- 使用 index 中的 `source.commit` 校验；
- 单独决定如何镜像或 allow-list 容器 registry 和镜像；
- 定期检查 index 构建报告，尤其是 smoke 或 digest gate 失败记录。

App 作者建议：

- 使用不可变的 `version-release` tag；
- 不要重写已经发布的 release tag；
- 为容器化 app 提供真实 `[smoke]` 元数据；
- 把 Dockerfile 和 app 元数据放在同一个源码仓库中版本化；
- 通过 `release.md` 保持 release notes 清晰。

## 当前限制

当前模型有意保留了几个未来工作：

- 只验摘要不等于验签；请查看收据，不要仅凭 `auto` 假定已使用 GPG；
- 官方 release 二进制仍由维护者手动构建，还不是完整自动化 provenance pipeline；
- 尚未声明可复现构建；
- 尚未声明 GitHub Actions artifact attestation；
- 容器镜像可信仍依赖 registry 和镜像发布流程；
- Gitee 或内部镜像可以改善访问，但不能替代 checksum、commit 和 digest 校验；
- 官方 Hub 是 curated registry，不是开放自助 package registry。

这些限制是当前安全模型的一部分。后续增强应该逐步减少这些限制，但不应夸大当前 release
已经提供的保证。

## 继续阅读

- [什么是 TAFFISH](taffish.cn.md)
- [什么是 TAFFISH Hub](taffish-hub.cn.md)
- [TAFFISH Index JSON 规范](index-json-spec.cn.md)
- [`taffish.toml` 规范](taffish-toml-spec.cn.md)
- [TAFFISH MCP 指南](taffish-mcp.cn.md)
- [故障排查](troubleshooting.cn.md)
- [taffish/taffish Security Policy](https://github.com/taffish/taffish/blob/main/SECURITY.md)
- [taffish/taffish 从源码构建](https://github.com/taffish/taffish/blob/main/docs/dev/zh-CN/build-from-source.md)
