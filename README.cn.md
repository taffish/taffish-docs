# TAFFISH 文档

[English](README.md) | [中文](README.cn.md)

TAFFISH 把生物信息学工具和轻量流程封装成可以按版本安装的命令。用 `taf` 从 TAFFISH Hub 查找并安装 app 后，就可以在终端或已有脚本中运行 `taf-augustus` 这样的命令。脚本需要固定版本时，使用带版本号的命令名。

**[安装 TAFFISH，运行第一个 app](zh/quick-start.cn.md)** · [浏览 TAFFISH Hub 中的 app](https://taffish.github.io)

<a id="目录"></a>

## 推荐阅读路径

| 我想做什么 | 从这里开始 | 需要时再看 |
| --- | --- | --- |
| 使用已有 app | [快速开始](zh/quick-start.cn.md) | 命令出错时查[故障排查](zh/troubleshooting.cn.md) |
| 在共享服务器或 HPC 上部署 | [系统管理员指南](zh/system-administration.cn.md) | 用[安全模型](zh/security-model.cn.md)了解安装和运行时需要信任什么 |
| 学习 `.taf` 语言 | [TAF 脚本教程](zh/taf-script-tutorial.cn.md) | 在[什么是 TAFFISH](zh/taffish.cn.md)中查语法和命令行用法 |
| 编写工具类 app | [App 开发者指南](zh/app-developer-guide.cn.md) | 在 [`taffish.toml` 规范](zh/taffish-toml-spec.cn.md)中查字段 |
| 封装在容器中运行的工具 | [App 开发者指南](zh/app-developer-guide.cn.md) | 用[容器化 app 最佳实践](zh/container-apps.cn.md)配置镜像、挂载和 smoke 测试 |
| 准备提交官方 Hub 收录 | [官方 taf-app 精修手册](zh/taf-app-curation-guide.cn.md) | 参照 [Augustus 实例](https://github.com/taffish/augustus) |
| 把多个工具组合成 Flow | [Flow 与依赖指南](zh/flow-dependencies.cn.md) | 按 [Flow 开发者指南](zh/taf-flow-developer-guide.cn.md)实现，再对照[发布前检查清单](zh/taf-flow-curation-checklist.cn.md) |
| 为一次分析查找现有 Flow | [TAFFISH Flow 门户](https://taffish.github.io/flows/) | [RNA-seq 实例与报告](#flow-门户与完整案例) |
| 了解一个 app 如何被索引收录 | [什么是 TAFFISH Hub](zh/taffish-hub.cn.md) | 在 [Index JSON 规范](zh/index-json-spec.cn.md)中查版本记录和测试结果 |
| 了解 TAFFISH 验证了什么 | [安全模型](zh/security-model.cn.md) | 用 [Index JSON 规范](zh/index-json-spec.cn.md)解读已记录的检查结果 |
| 接入 AI 客户端 | [MCP 客户端配置](zh/mcp-clients.cn.md) | 在 [MCP 指南](zh/taffish-mcp.cn.md)中查客户端能调用哪些工具 |
| 构建或修改 TAFFISH 本身 | [从源码构建](https://github.com/taffish/taffish/blob/main/docs/dev/zh-CN/build-from-source.md) | [源码开发与贡献](#源码开发) |

用 `taf --version` 查看本机安装的版本。0.11.0 的具体变化见[发布说明](https://github.com/taffish/taffish/releases/tag/v0.11.0)。

## Flow 门户与完整案例

[Flow 门户](https://taffish.github.io/flows/)收录各个官方 Flow 系列。想看一条完整的分析流程，可以从 [RNA-seq 系列](https://taffish.github.io/rnaseq-flows/)开始：这个实例用带版本的命令串起从准备参考基因组到生成报告的各个步骤。这些命令也可以在已有的工作流引擎中调用。阅读结果时，可以对照[酵母示例报告](https://taffish.github.io/rnaseq-flows/examples/yeast-standard-report/04_reports/rnaseq_report.html)和[报告解读手册](https://taffish.github.io/rnaseq-flows/examples/yeast-standard-report/04_reports/report_interpretation.html)。

<a id="文档之间的重复关系"></a>

## 文档分工

跟着示例动手时读教程，做具体任务时读指南，核对字段时查规范。

### 核心概念

- [什么是 TAFFISH](zh/taffish.cn.md)：一个 `.taf` 脚本怎样成为 shell 命令，如何处理参数和容器，以及 [MCP 的作用](zh/taffish.cn.md#mcp--ai-集成)。
- [什么是 TAFFISH Hub](zh/taffish-hub.cn.md)：从 GitHub 发布、索引检查，到用户能够安装一个 app 版本，中间有哪些步骤。

### 用户指南

- [快速开始](zh/quick-start.cn.md)：安装并运行 Augustus。以后需要用 `taf install --from` 安装私有或本地 app、用 `taf outdated` 检查更新，或用 `taf upgrade` 和 `taf prune` 管理已安装版本，也可以在这里查。
- [系统管理员指南](zh/system-administration.cn.md)：配置共享安装、权限、shell 补全、Vim 文件和容器策略。
- [MCP 客户端配置](zh/mcp-clients.cn.md)：把 Codex、Claude Code 等客户端连接到 `taffish-mcp`。[MCP 指南](zh/taffish-mcp.cn.md)说明可用工具、检查和编译预览，以及服务不会执行哪些操作。
- [故障排查](zh/troubleshooting.cn.md)：定位安装、镜像源访问、容器启动或已安装 app 命令运行时的问题。

### 开发指南

- [TAF 脚本教程](zh/taf-script-tutorial.cn.md)：编写 `.taf` 脚本并传入参数。[App 开发者指南](zh/app-developer-guide.cn.md)从 `taf new` 开始，讲解如何测试、构建、准备 `release.md` 和发布。
- [容器化 app 最佳实践](zh/container-apps.cn.md)：构建镜像、声明 smoke 测试、核对 Docker、Podman 和 Apptainer 的运行情况。app 自身需要的参数写在 `$@[target: args]` 中；本地选用哪个后端、传入哪些额外参数，则由 `TAFFISH_CONTAINER_BACKEND`、`TAFFISH_PODMAN_RUN_ARGS` 等环境变量控制。
- [官方 taf-app 精修手册](zh/taf-app-curation-guide.cn.md)：提交 Hub 收录前，检查 app 的帮助、版本、smoke 测试和发布内容。
- [Flow 与依赖指南](zh/flow-dependencies.cn.md)：声明依赖的确切版本，使用 `[[taf: ...]]` 和 `@:` 参数块调用依赖。[Flow 开发者指南](zh/taf-flow-developer-guide.cn.md)讲解如何设计输入输出、记录运行来源；发布前使用 [Flow 检查清单](zh/taf-flow-curation-checklist.cn.md)。

### 规范文档

- [`taffish.toml` 规范](zh/taffish-toml-spec.cn.md)：查 app 作者填写的字段，如 `[smoke]`、`[meta]` 和 `[upstream]`。
- [Index JSON 规范](zh/index-json-spec.cn.md)：查 Hub 为每个版本记录的内容，如源码身份、容器镜像摘要、smoke 结果和索引侧的元数据修正。

### 安全模型

[安全模型](zh/security-model.cn.md)说明安装器会验证什么、Hub 和 `taf install` 如何检查发布内容及 `source.commit`，以及使用镜像源、容器和 MCP 客户端时，哪些环节仍需自行判断是否可信。

## 源码开发

编译器、`taf` 和 `taffish-mcp` 的源码在 [taffish/taffish](https://github.com/taffish/taffish)。使用 SBCL 或 LispWorks 构建程序时，读[从源码构建](https://github.com/taffish/taffish/blob/main/docs/dev/zh-CN/build-from-source.md)；了解架构和发布过程，读[源码树开发文档](https://github.com/taffish/taffish/tree/main/docs/dev/zh-CN)；提交补丁和测试，读[贡献指南](https://github.com/taffish/taffish/blob/main/CONTRIBUTING.md)。私下报告安全漏洞，请按[安全策略](https://github.com/taffish/taffish/blob/main/SECURITY.md)联系维护者。

<a id="当前状态"></a>

## 在线 Hub

[taffish.github.io](https://taffish.github.io) 展示官方索引；`taf update` 刷新 `taf` 使用的本地索引副本。官方 Hub 由 `taffish` GitHub 组织审核收录。希望提交 app 或申请加入组织，请联系维护者；在自己的仓库发布并不等于进入官方 Hub。

<a id="仓库结构"></a>

## 文档维护说明

英文指南在 `en/`（`*.en.md`），中文指南在 `zh/`（`*.cn.md`）。文档适用于多个版本；需要确认某个版本的具体行为时，以对应的发布说明和源码为准。

## 许可证

本文档采用 [CC BY 4.0](LICENSE) 许可。
