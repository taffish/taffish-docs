# TAFFISH 文档

[English](README.md) | [中文](README.cn.md)

这个仓库用于存放 TAFFISH 和 TAFFISH Hub 的开发者文档。

TAFFISH 是一个面向生信工具和流程的轻量级命令交付系统。TAFFISH Hub 是
`taf` 用来发现、安装和管理 TAFFISH app 的官方索引与注册层。

## 目录

- [先读这两份](#先读这两份)
- [开发指南](#开发指南)
- [规范文档](#规范文档)
- [仓库结构](#仓库结构)
- [在线 Hub](#在线-hub)
- [文档维护说明](#文档维护说明)
- [当前状态](#当前状态)

## 先读这两份

| 文档 | 简介 |
| --- | --- |
| [什么是 TAFFISH](zh/taffish.cn.md) | 介绍 TAFFISH 语言、编译器、CLI、app 项目结构、参数系统、容器标签和推荐开发流程。 |
| [什么是 TAFFISH Hub](zh/taffish-hub.cn.md) | 介绍 Hub 架构、GitHub 自动化、index 构建、网页版 registry、依赖处理和发布策略。 |

如果你刚开始了解 TAFFISH，建议先读这两份。

## 开发指南

| 文档 | 简介 |
| --- | --- |
| [TAFFISH app 开发者指南](zh/app-developer-guide.cn.md) | 从创建、检查、运行、构建、发布到维护 TAFFISH app 的完整实践流程。 |
| [容器化 app 最佳实践](zh/container-apps.cn.md) | Dockerfile 设计、多阶段构建、GHCR 可见性、本地 Docker/Podman 测试和 backend 一致性。 |
| [Flow 与依赖指南](zh/flow-dependencies.cn.md) | Flow app 结构、`[[taf: ...]]`、`@:` 参数块、依赖声明、多版本依赖和安装语义。 |

当你正在开发或维护 app 时，主要参考这些文档。

## 规范文档

| 文档 | 简介 |
| --- | --- |
| [`taffish.toml` 规范](zh/taffish-toml-spec.cn.md) | `taf`、TAFFISH Hub 和 index builder 共同使用的 app 元数据字段规范。 |
| [TAFFISH Index JSON 规范](zh/index-json-spec.cn.md) | `taf update`、`taf search`、`taf info` 和 `taf install` 消费的静态索引格式。 |

当你要实现工具、自动化、校验器或 Hub 消费端时，主要参考这些规范。

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

## 当前状态

官方 TAFFISH Hub 目前由 `taffish` GitHub 组织维护，暂时不是开放自助发布平台。
如果开发者希望把 app 纳入官方 Hub，需要联系维护者审核，或申请加入
`taffish` 组织。
