# TAFFISH 文档

[English](README.md) | [中文](README.cn.md)

这个仓库用于存放 TAFFISH、TAFFISH Hub，以及本地 `taffish-mcp` AI 集成入口的开发者文档。

TAFFISH 是一个面向生信工具和流程的轻量级命令交付系统。TAFFISH Hub 是
`taf` 用来发现、安装和管理 TAFFISH app 的官方索引与注册层。

## 目录

- [推荐阅读路径](#推荐阅读路径)
- [文档分工](#文档分工)
- [核心概念](#核心概念)
- [用户指南](#用户指南)
- [开发指南](#开发指南)
- [规范文档](#规范文档)
- [文档之间的重复关系](#文档之间的重复关系)
- [仓库结构](#仓库结构)
- [在线 Hub](#在线-hub)
- [文档维护说明](#文档维护说明)
- [许可证](#许可证)
- [当前状态](#当前状态)

## 推荐阅读路径

根据你的目标选择阅读顺序。

| 目标 | 推荐顺序 |
| --- | --- |
| 使用已有 TAFFISH app | [快速开始](zh/quick-start.cn.md) -> 遇到问题看 [故障排查](zh/troubleshooting.cn.md) -> 需要背景再看 [什么是 TAFFISH Hub](zh/taffish-hub.cn.md) |
| 学习 TAFFISH 语言 | [什么是 TAFFISH](zh/taffish.cn.md) -> [TAF 脚本教程](zh/taf-script-tutorial.cn.md) |
| 写一个简单 tool app | [TAF 脚本教程](zh/taf-script-tutorial.cn.md) -> [App 开发者指南](zh/app-developer-guide.cn.md) -> [`taffish.toml` 规范](zh/taffish-toml-spec.cn.md) |
| 封装一个容器化生信工具 | [App 开发者指南](zh/app-developer-guide.cn.md) -> [容器化 app 最佳实践](zh/container-apps.cn.md) -> [故障排查](zh/troubleshooting.cn.md) |
| 写一个带依赖的 flow app | [TAF 脚本教程](zh/taf-script-tutorial.cn.md) -> [Flow 与依赖指南](zh/flow-dependencies.cn.md) -> [`taffish.toml` 规范](zh/taffish-toml-spec.cn.md) |
| 理解 Hub 和 index 内部逻辑 | [什么是 TAFFISH Hub](zh/taffish-hub.cn.md) -> [TAFFISH Index JSON 规范](zh/index-json-spec.cn.md) -> [`taffish.toml` 规范](zh/taffish-toml-spec.cn.md) |
| 连接 TAFFISH 到 AI 客户端 | [在 AI 客户端中使用 TAFFISH MCP](zh/mcp-clients.cn.md) -> [TAFFISH MCP 指南](zh/taffish-mcp.cn.md) -> 需要背景再看 [什么是 TAFFISH](zh/taffish.cn.md#mcp--ai-集成) |

如果你完全不了解 TAFFISH，建议先读 [快速开始](zh/quick-start.cn.md)，再读
[什么是 TAFFISH](zh/taffish.cn.md)。

## 文档分工

| 文档 | 分工 |
| --- | --- |
| [快速开始](zh/quick-start.cn.md) | 用户上手入口。它会有意重复安装、镜像配置、更新、搜索、安装、运行、列出、定位和卸载等基础操作。 |
| [什么是 TAFFISH](zh/taffish.cn.md) | 概念型语言与 CLI 手册。解释设计目标、语法、标签、参数、项目结构、CLI 表面和 MCP/AI 集成入口。 |
| [TAF 脚本教程](zh/taf-script-tutorial.cn.md) | 动手写 `.taf` 的学习路径。从小脚本逐步过渡到 app wrapper 和 flow。 |
| [TAFFISH MCP 指南](zh/taffish-mcp.cn.md) | `taffish-mcp` 能力参考，覆盖 tools、resources、prompts、安全边界和故障排查。 |
| [在 AI 客户端中使用 TAFFISH MCP](zh/mcp-clients.cn.md) | 面向 Codex、Claude Code、Cursor、Cline 和通用 stdio MCP 客户端的配置教程。 |
| [TAFFISH app 开发者指南](zh/app-developer-guide.cn.md) | 实际 app 发布流程。重点是 `taf new`、项目编辑、检查、运行、构建、`release.md`、发布和维护。 |
| [容器化 app 最佳实践](zh/container-apps.cn.md) | 容器专题。覆盖 Dockerfile 设计、运行时挂载、GHCR、Docker/Podman 测试和 backend 一致性。 |
| [Flow 与依赖指南](zh/flow-dependencies.cn.md) | Flow 专题。覆盖 `[[taf: ...]]`、`@:` 块、精确 app 版本和依赖语义。 |
| [`taffish.toml` 规范](zh/taffish-toml-spec.cn.md) | 元数据参考。面向 app 作者、Hub 维护者和校验器。 |
| [TAFFISH Index JSON 规范](zh/index-json-spec.cn.md) | 机器可读 index 参考。面向 `taf`、Hub 自动化和 index 消费端。 |
| [故障排查](zh/troubleshooting.cn.md) | 问题导向参考。命令、容器、GHCR、GitHub 或 wrapper 出错时从这里开始。 |

## 核心概念

| 文档 | 简介 |
| --- | --- |
| [什么是 TAFFISH](zh/taffish.cn.md) | 介绍 TAFFISH 语言、编译器、CLI、`taffish-mcp`、app 项目结构、参数系统、容器标签和推荐开发流程。 |
| [什么是 TAFFISH Hub](zh/taffish-hub.cn.md) | 介绍 Hub 架构、GitHub 自动化、index 构建、网页版 registry、依赖处理和发布策略。 |

当你需要理解整个系统的结构时，主要阅读这两份。

## 用户指南

| 文档 | 简介 |
| --- | --- |
| [TAFFISH 快速开始](zh/quick-start.cn.md) | 安装 TAFFISH，更新 Hub 索引，搜索、安装、运行、列出、定位和卸载 app。 |
| [在 AI 客户端中使用 TAFFISH MCP](zh/mcp-clients.cn.md) | 在 Codex、Claude Code、Cursor、Cline 或通用 stdio MCP 客户端中配置 `taffish-mcp`。 |
| [TAFFISH MCP 指南](zh/taffish-mcp.cn.md) | 理解 `taffish-mcp` 的 tools、resources、prompts 和安全模型。 |
| [TAF 脚本教程](zh/taf-script-tutorial.cn.md) | 面向 app 作者的 `.taf` 分步教程，从最小脚本到参数、容器、flow 和依赖。 |
| [TAFFISH 故障排查](zh/troubleshooting.cn.md) | 常见安装、索引、镜像配置、容器、GHCR、Podman、Docker、Apptainer 和 wrapper 问题。 |

当你需要从第一次安装走到日常使用时，优先参考这些文档。

## 开发指南

| 文档 | 简介 |
| --- | --- |
| [TAFFISH app 开发者指南](zh/app-developer-guide.cn.md) | 从创建、检查、运行、构建、准备 `release.md`、发布到维护 TAFFISH app 的完整实践流程。 |
| [容器化 app 最佳实践](zh/container-apps.cn.md) | Dockerfile 设计、多阶段构建、运行时挂载、GHCR 可见性、本地 Docker/Podman 测试和 backend 一致性。 |
| [Flow 与依赖指南](zh/flow-dependencies.cn.md) | Flow app 结构、`[[taf: ...]]`、`@:` 参数块、依赖声明、多版本依赖和安装语义。 |

当你正在开发或维护 app 时，主要参考这些文档。

## 规范文档

| 文档 | 简介 |
| --- | --- |
| [`taffish.toml` 规范](zh/taffish-toml-spec.cn.md) | `taf`、TAFFISH Hub 和 index builder 共同使用的 app 元数据字段规范。 |
| [TAFFISH Index JSON 规范](zh/index-json-spec.cn.md) | `taf update`、`taf search`、`taf info` 和 `taf install` 消费的静态索引格式。 |

当你要实现工具、自动化、校验器或 Hub 消费端时，主要参考这些规范。

## 文档之间的重复关系

有些重复是有意保留的：

- 安装和基础 `taf` 命令会同时出现在快速开始和“什么是 TAFFISH”里，因为用户需要先能快速上手，之后再理解完整模型。
- 运行时镜像配置会出现在快速开始、什么是 TAFFISH、什么是 TAFFISH Hub 和故障排查里，因为它同时影响网络访问和 package 安装。
- `taffish-mcp` 会在“什么是 TAFFISH”中简要出现。MCP 指南说明 server 能力和安全边界，
  客户端接入教程则说明 Codex、Claude Code、Cursor、Cline 和通用 MCP 配置示例。
- `taf run`、`taf build`、`taf publish` 会同时出现在 app 开发指南和语言手册里，因为它们连接了语法和项目生命周期。
- 容器 backend 说明会出现在快速开始、容器最佳实践和故障排查里，因为真实部署中容器问题最常见。
- Flow 依赖会出现在语言手册、app 开发指南和 Flow 专题里；其中 Flow 专题是最详细来源。

简单说：教程用于学习，指南用于做事，规范用于查字段，故障排查用于定位问题。

## 仓库结构

```text
./
  README.md
  README.cn.md
  en/
    *.en.md
  zh/
    *.cn.md
```

默认 README 是英文版。中文文件使用 `.cn.md` 后缀，英文文件使用 `.en.md`
后缀。

## 在线 Hub

- [taffish.github.io](https://taffish.github.io)

网页版 Hub 浏览的是和 `taf` 本地命令相同的静态索引。

## 文档维护说明

这些文档描述的是 TAFFISH 和 TAFFISH Hub 当前的设计与约定。它们是作为源文档
维护的，不是绑定到某一个特定 `taf` 或 `taffish` 二进制版本的自动生成产物。

当需要确认精确行为时，对应仓库的代码和 release notes 应作为最终依据。文档的
维护历史通过 GitHub commit history 追踪。如果未来某份文档只适用于特定版本，
应该在该文档顶部明确标注。

## 许可证

本仓库中的文档使用 [Creative Commons Attribution 4.0 International License](LICENSE)
授权，即 CC BY 4.0。

## 当前状态

官方 TAFFISH Hub 目前由 `taffish` GitHub 组织维护，暂时不是开放自助发布平台。
如果开发者希望把 app 纳入官方 Hub，需要联系维护者审核，或申请加入
`taffish` 组织。
