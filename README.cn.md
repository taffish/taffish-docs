# TAFFISH 文档

[English](README.md) | [中文](README.cn.md)

这个仓库用于存放 TAFFISH、TAFFISH Hub、taf-app 开发、静态 index，以及本地
`taffish-mcp` AI 集成入口的公共文档。

TAFFISH 是一个面向生物信息学命令行工具与轻量流程的 shell-native 可复现执行与交付层。
它通过可执行包模型把命令级可复现性带回 shell。TAFFISH Hub 是 `taf` 用来发现、安装
和管理 TAFFISH app 的官方索引与注册层。

## 目录

- [推荐阅读路径](#推荐阅读路径)
- [Flow 门户与完整案例](#flow-门户与完整案例)
- [文档分工](#文档分工)
- [核心概念](#核心概念)
- [用户指南](#用户指南)
- [开发指南](#开发指南)
- [规范文档](#规范文档)
- [安全模型](#安全模型)
- [源码开发](#源码开发)
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
| 管理共享服务器或 HPC 安装 | [系统管理员指南](zh/system-administration.cn.md) -> [故障排查](zh/troubleshooting.cn.md) -> [安全模型](zh/security-model.cn.md) |
| 学习 TAFFISH 语言 | [什么是 TAFFISH](zh/taffish.cn.md) -> [TAF 脚本教程](zh/taf-script-tutorial.cn.md) |
| 写一个简单 tool app | [TAF 脚本教程](zh/taf-script-tutorial.cn.md) -> [App 开发者指南](zh/app-developer-guide.cn.md) -> [`taffish.toml` 规范](zh/taffish-toml-spec.cn.md) |
| 封装一个容器化生信工具 | [App 开发者指南](zh/app-developer-guide.cn.md) -> [容器化 app 最佳实践](zh/container-apps.cn.md) -> [官方 taf-app 精修手册](zh/taf-app-curation-guide.cn.md) -> [故障排查](zh/troubleshooting.cn.md) |
| 精修官方 Hub app | [官方 taf-app 精修手册](zh/taf-app-curation-guide.cn.md) -> [Augustus app 模板](https://github.com/taffish/augustus) -> [容器化 app 最佳实践](zh/container-apps.cn.md) -> [`taffish.toml` 规范](zh/taffish-toml-spec.cn.md) |
| 写一个带依赖的 flow app | [TAF 脚本教程](zh/taf-script-tutorial.cn.md) -> [Flow 与依赖指南](zh/flow-dependencies.cn.md) -> [TAFFISH Flow 开发者指南](zh/taf-flow-developer-guide.cn.md) -> [TAFFISH Flow 精修 Checklist](zh/taf-flow-curation-checklist.cn.md) -> [`taffish.toml` 规范](zh/taffish-toml-spec.cn.md) |
| 查看官方 flow 路线 | [TAFFISH Flow Portal](https://taffish.github.io/flows/) -> [RNA-seq Flow Family](https://taffish.github.io/rnaseq-flows/) -> [Flow 与依赖指南](zh/flow-dependencies.cn.md) |
| 理解 Hub 和 index 内部逻辑 | [什么是 TAFFISH Hub](zh/taffish-hub.cn.md) -> [TAFFISH Index JSON 规范](zh/index-json-spec.cn.md) -> [`taffish.toml` 规范](zh/taffish-toml-spec.cn.md) |
| 理解安全与可信模型 | [TAFFISH 安全模型](zh/security-model.cn.md) -> [TAFFISH Index JSON 规范](zh/index-json-spec.cn.md) -> [TAFFISH MCP 指南](zh/taffish-mcp.cn.md) |
| 连接 TAFFISH 到 AI 客户端 | [在 AI 客户端中使用 TAFFISH MCP](zh/mcp-clients.cn.md) -> [TAFFISH MCP 指南](zh/taffish-mcp.cn.md) -> 需要背景再看 [什么是 TAFFISH](zh/taffish.cn.md#mcp--ai-集成) |
| 构建或修改 TAFFISH 本体 | [taffish/taffish README](https://github.com/taffish/taffish) -> [从源码构建](https://github.com/taffish/taffish/blob/main/docs/dev/zh-CN/build-from-source.md) -> [贡献指南](https://github.com/taffish/taffish/blob/main/CONTRIBUTING.md) |

如果你完全不了解 TAFFISH，建议先读 [快速开始](zh/quick-start.cn.md)，再读
[什么是 TAFFISH](zh/taffish.cn.md)。

## Flow 门户与完整案例

[TAFFISH Flow Portal](https://taffish.github.io/flows/) 是官方 flow family 和策划型
分析路线的公开入口。它链接可用 flow 页面、Hub 记录、示例报告和更深入的专题案例站。

[RNA-seq Flow Family](https://taffish.github.io/rnaseq-flows/) 仍然是当前最完整的公开
flow-family 案例，展示如何把可版本化的 `taf-*` 命令组合成从参考基因组准备到最终
报告交付的分析路线。它包含每个 flow 的使用手册、yeast 示例报告，以及报告解读手册：

- [TAFFISH Flow Portal](https://taffish.github.io/flows/)
- [RNA-seq Flow Family 门户](https://taffish.github.io/rnaseq-flows/)
- [Yeast 标准报告](https://taffish.github.io/rnaseq-flows/examples/yeast-standard-report/04_reports/rnaseq_report.html)
- [报告解读手册](https://taffish.github.io/rnaseq-flows/examples/yeast-standard-report/04_reports/report_interpretation.html)

这个例子用于展示 TAFFISH 的 command-level reproducibility 在真实分析中的用法：
它把 shell-native app 命令组合成轻量 flow，但并不把 TAFFISH 变成传统 workflow
engine 的替代品。

## 文档分工

| 文档 | 分工 |
| --- | --- |
| [快速开始](zh/quick-start.cn.md) | 用户上手入口。它会有意重复安装、镜像配置、更新、搜索、安装、本地维护、运行、列出、定位和卸载等基础操作。 |
| [系统管理员指南](zh/system-administration.cn.md) | 共享工作站、服务器和 HPC 部署指南，覆盖 user/system scope、权限、命令路径、全局补全/Vim、容器策略、升级和移除。 |
| [什么是 TAFFISH](zh/taffish.cn.md) | 概念型语言与 CLI 手册。解释设计目标、语法、标签、参数、项目结构、CLI 表面和 MCP/AI 集成入口。 |
| [TAF 脚本教程](zh/taf-script-tutorial.cn.md) | 动手写 `.taf` 的学习路径。从小脚本逐步过渡到 app wrapper 和 flow。 |
| [TAFFISH MCP 指南](zh/taffish-mcp.cn.md) | `taffish-mcp` 能力参考，覆盖 tools、只读编译器辅助工具、app/project inspection、smoke/trust 元数据暴露、本地包维护规划工具、resources、prompts、安全边界和故障排查。 |
| [在 AI 客户端中使用 TAFFISH MCP](zh/mcp-clients.cn.md) | 面向 Codex、Claude Code、Cursor、Cline 和通用 stdio MCP 客户端的配置教程。 |
| [TAFFISH app 开发者指南](zh/app-developer-guide.cn.md) | 实际 app 发布流程。重点是 `taf new`、项目编辑、检查、运行、构建、`release.md`、发布和维护。 |
| [容器化 app 最佳实践](zh/container-apps.cn.md) | 容器专题。覆盖 Dockerfile 设计、smoke 元数据、运行时挂载、GHCR、Docker/Podman 测试、backend 一致性，以及 TAFFISH `0.9.0` 引入的 runtime 参数策略。 |
| [官方 taf-app 精修手册](zh/taf-app-curation-guide.cn.md) | 官方 Hub 维护者手册。以 Augustus 为模板，说明 app 文件边界、help/README/release/smoke 设计、版本策略、发布前后检查和不该改的内容。 |
| [Flow 与依赖指南](zh/flow-dependencies.cn.md) | Flow 专题。覆盖 `[[taf: ...]]`、`@:` 块、精确 app 版本和依赖语义。 |
| [TAFFISH Flow 开发者指南](zh/taf-flow-developer-guide.cn.md) | Flow app 工程手册。覆盖 flow 定位、输出目录、输入输出契约、依赖、资源、provenance、smoke 和发布边界。 |
| [TAFFISH Flow 精修 Checklist](zh/taf-flow-curation-checklist.cn.md) | 官方 flow 发布前 checklist。覆盖边界、依赖、输出目录、数据流、报告、provenance、资源、smoke、文档和 hard blockers。 |
| [`taffish.toml` 规范](zh/taffish-toml-spec.cn.md) | 元数据参考。面向 app 作者、Hub 维护者和校验器，包含容器化 app 的 `[smoke]` 元数据。 |
| [TAFFISH Index JSON 规范](zh/index-json-spec.cn.md) | 机器可读 index 参考。面向 `taf`、Hub 自动化、index 消费端、trust 元数据、容器 digest、smoke 结果和构建报告。 |
| [TAFFISH 安全模型](zh/security-model.cn.md) | 分层可信模型，覆盖源码、release 完整性、安装器、镜像、Hub index gate、本地安装校验、容器和 MCP/AI 边界。 |
| [故障排查](zh/troubleshooting.cn.md) | 问题导向参考。命令、smoke 元数据、容器、GHCR、GitHub 或 wrapper 出错时从这里开始。 |

维护规则：字段级事实以 [`taffish.toml` 规范](zh/taffish-toml-spec.cn.md) 为准；
index 输出以 [TAFFISH Index JSON 规范](zh/index-json-spec.cn.md) 为准；容器实践以
[容器化 app 最佳实践](zh/container-apps.cn.md) 为准；官方 Hub app 的最终 checklist
以 [官方 taf-app 精修手册](zh/taf-app-curation-guide.cn.md) 为准；flow app 的工程约定以
[TAFFISH Flow 开发者指南](zh/taf-flow-developer-guide.cn.md) 和
[TAFFISH Flow 精修 Checklist](zh/taf-flow-curation-checklist.cn.md) 为准；具体报错处理以
[故障排查](zh/troubleshooting.cn.md) 为准。

## 核心概念

| 文档 | 简介 |
| --- | --- |
| [什么是 TAFFISH](zh/taffish.cn.md) | 介绍 TAFFISH 语言、编译器、CLI、`taffish-mcp`、app 项目结构、参数系统、容器标签和推荐开发流程。 |
| [什么是 TAFFISH Hub](zh/taffish-hub.cn.md) | 介绍 Hub 架构、GitHub 自动化、index 构建、网页版 registry、依赖处理和发布策略。 |

当你需要理解整个系统的结构时，主要阅读这两份。

## 用户指南

| 文档 | 简介 |
| --- | --- |
| [TAFFISH 快速开始](zh/quick-start.cn.md) | 安装 TAFFISH，更新 Hub 索引，搜索、安装公开 app，用 `taf outdated`、`taf upgrade` 和 `taf prune` 维护本地安装，用 `taf install --from` 安装私有/本地 app，运行、列出、定位和卸载 app。 |
| [TAFFISH 系统管理员指南](zh/system-administration.cn.md) | 面向多用户安装和运维 TAFFISH，覆盖显式 system scope、权限、全局 shell/编辑器集成、容器策略和维护。 |
| [在 AI 客户端中使用 TAFFISH MCP](zh/mcp-clients.cn.md) | 在 Codex、Claude Code、Cursor、Cline 或通用 stdio MCP 客户端中配置 `taffish-mcp`。 |
| [TAFFISH MCP 指南](zh/taffish-mcp.cn.md) | 理解 `taffish-mcp` 的 tools、只读编译器辅助工具、app/project inspection、smoke/trust 元数据暴露、本地包维护规划工具、resources、prompts 和安全模型。 |
| [TAF 脚本教程](zh/taf-script-tutorial.cn.md) | 面向 app 作者的 `.taf` 分步教程，从最小脚本到参数、容器、flow 和依赖。 |
| [TAFFISH 故障排查](zh/troubleshooting.cn.md) | 常见安装、索引、镜像配置、smoke 元数据、容器、GHCR、Podman、Docker、Apptainer 和 wrapper 问题。 |

当你需要从第一次安装走到日常使用时，优先参考这些文档。

## 开发指南

| 文档 | 简介 |
| --- | --- |
| [TAFFISH app 开发者指南](zh/app-developer-guide.cn.md) | 从创建、检查、运行、构建、用 `taf install --from` 做私有/本地测试、准备 `release.md`、发布到维护 TAFFISH app 的完整实践流程。 |
| [容器化 app 最佳实践](zh/container-apps.cn.md) | Dockerfile 设计、多阶段构建、smoke 元数据、运行时挂载、GHCR 可见性、本地 Docker/Podman 测试、backend 一致性、`TAFFISH_CONTAINER_BACKEND`、结构化 `$@[target: args]` 和 backend runtime 参数环境变量。 |
| [官方 taf-app 精修手册](zh/taf-app-curation-guide.cn.md) | 面向官方 Hub 维护者的 curation guide，说明如何把 `taf new` 项目打磨成可发布、可索引、可复用为模板的正式 app。 |
| [Flow 与依赖指南](zh/flow-dependencies.cn.md) | Flow app 结构、`[[taf: ...]]`、`@:` 参数块、依赖声明、多版本依赖和安装语义。 |
| [TAFFISH Flow 开发者指南](zh/taf-flow-developer-guide.cn.md) | 设计、实现、验证和发布 taf-flow app 的工程手册，强调单一 `<outdir>`、输入输出契约、provenance 和资源说明。 |
| [TAFFISH Flow 精修 Checklist](zh/taf-flow-curation-checklist.cn.md) | 发布或审核 taf-flow app 前使用的 checklist 和最终审查报告模板。 |

当你正在开发或维护 app 时，主要参考这些文档。

## 规范文档

| 文档 | 简介 |
| --- | --- |
| [`taffish.toml` 规范](zh/taffish-toml-spec.cn.md) | `taf`、TAFFISH Hub 和 index builder 共同使用的 app 元数据字段规范，包括 `[smoke]`、`[meta]` 和 `[upstream]`。 |
| [TAFFISH Index JSON 规范](zh/index-json-spec.cn.md) | `taf update`、`taf search`、`taf info` 和 `taf install` 消费的静态索引格式，包括 trust reports、容器 smoke 元数据和 index 侧 metadata override。 |

当你要实现工具、自动化、校验器或 Hub 消费端时，主要参考这些规范。

## 安全模型

| 文档 | 简介 |
| --- | --- |
| [TAFFISH 安全模型](zh/security-model.cn.md) | 从整体上解释 release 完整性、安装器行为、镜像行为、Hub/index 可信 gate、`source.commit` 校验、容器边界和 MCP 安全边界。 |

当你需要评估 TAFFISH 是否适合科研合作、服务器部署、私有镜像、企业环境或安全敏感
流程时，优先阅读这份文档。

## 源码开发

权威的当前 release、源码、ASDF 系统、源码树开发文档、release 载荷、贡献指南和
安全策略都位于 [taffish/taffish](https://github.com/taffish/taffish)。具体版本的
功能历史应由该仓库的 release notes 承担，不在当前文档系统中反复维护“当前版本”段落。

源码侧最相关的文档是：

| 文档 | 简介 |
| --- | --- |
| [从源码构建](https://github.com/taffish/taffish/blob/main/docs/dev/zh-CN/build-from-source.md) | 使用 SBCL 或 LispWorks 从 Common Lisp 源码树构建 `taf`、`taffish` 和 `taffish-mcp`。 |
| [源码树开发文档](https://github.com/taffish/taffish/tree/main/docs/dev/zh-CN) | `taffish-core`、`taf-core`、`taffish-mcp`、`han`、ASDF 系统和 release engineering 的架构说明。 |
| [贡献指南](https://github.com/taffish/taffish/blob/main/CONTRIBUTING.md) | 开发环境、测试要求、代码边界、文档要求和 release artifact 策略。 |
| [安全策略](https://github.com/taffish/taffish/blob/main/SECURITY.md) | 私密安全报告方式，以及当前 release/index 可信模型。 |

本 `taffish-docs` 仓库继续聚焦用户、app 作者、Hub 和 index 文档。编译器实现细节应尽量留在源码仓库旁边维护。

## 文档之间的重复关系

有些重复是有意保留的：

- 安装和基础 `taf` 命令会同时出现在快速开始和“什么是 TAFFISH”里，因为用户需要先能快速上手，之后再理解完整模型。
- system scope 命令会在快速开始中简要出现，并在系统管理员指南中完整说明；共享权限、
  全局 shell/编辑器集成和混合部署以系统管理员指南为事实来源。
- 运行时镜像配置会出现在快速开始、什么是 TAFFISH、什么是 TAFFISH Hub 和故障排查里，因为它同时影响网络访问和 package 安装。
- `taffish-mcp` 会在“什么是 TAFFISH”中简要出现。MCP 指南说明 server 能力、只读编译器辅助工具、app/project inspection、smoke/trust 元数据暴露、本地包维护规划工具、安全编译预览和安全边界，
  客户端接入教程则说明 Codex、Claude Code、Cursor、Cline 和通用 MCP 配置示例。
- `taf run`、`taf build`、`taf publish` 会同时出现在 app 开发指南和语言手册里，因为它们连接了语法和项目生命周期。
- 官方 taf-app 精修手册与 app 开发者指南、容器最佳实践有重叠，但它更像维护者 checklist，聚焦“以 Augustus 为模板做官方 Hub app”。
- 容器 backend 和 runtime 参数说明会出现在快速开始、容器最佳实践和故障排查里，因为真实部署中容器问题最常见。app 自身运行需求应写在 `.taf` 标签的 `$@[target: args]` 中；单次本地策略应使用 `TAFFISH_DOCKER_RUN_ARGS`、`TAFFISH_PODMAN_RUN_ARGS` 或 `TAFFISH_APPTAINER_RUN_ARGS`。
- Smoke 元数据会出现在 app 开发指南、容器最佳实践、`taffish.toml` 规范、Hub 指南、Index JSON 规范和 MCP 指南里，因为 TAFFISH `0.8.0` 把本地 app 元数据、Hub 侧校验和 AI 辅助检查连接到了一起。
- 安全内容单独形成模型文档，因为 release 校验、镜像、index trust gate、本地安装校验、容器和 MCP 边界横跨多个仓库。
- 源码构建和编译器内部文档会在这里给出入口，但正文保留在
  `taffish/taffish`，这样它们可以跟 Common Lisp 源码树一起演进。
- Flow 依赖会出现在语言手册、app 开发指南和 Flow 专题里；其中《Flow 与依赖指南》是语法与依赖语义的最详细来源。
- Flow 工程约定会出现在 app 开发指南、Flow 开发者指南和 Flow checklist 里；其中 Flow 开发者指南负责输出目录、输入输出契约、provenance 和资源模型，Flow checklist 负责发布前审查。

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
