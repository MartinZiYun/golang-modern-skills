# go-modern

一个 agent 技能（skill），帮助 AI 编码 agent 遵循现代 Go 开发规范、使用新语言特性（Go 1.19–1.27）。

## 概述

当 agent 编写、审查或生成 Go 代码时，本技能提供：

- 最新开发规范（优先使用 `slices`/`maps`/`log/slog`，避免已弃用的 API）
- 新特性指引（泛型演进、range-over-func 迭代器、generic methods、`new(expr)` 等）
- 版本升级时的行为变更提醒

采用**按需读取**设计：入口文件（SKILL.md）是精炼的索引，含高频速查表。详细内容按主题拆分到 `reference/` 下的独立文件，agent 只在相关时才加载对应文件，最大限度节省上下文。

> 注：skill 正文（SKILL.md 和 reference 文件）使用英文编写，更适合 AI 处理。本 README 为中文说明。

## 文件结构

```
go-modern/
├── SKILL.md                              入口：触发规则 + 版本速查 + 高频速查 + 按需读取索引
├── README.md                             英文说明
├── README.zh-CN.md                       本文件（中文说明）
├── locales/                              可选：显示名本地化（供支持的框架使用）
│   ├── en-US.json                        英文显示名与简介
│   └── zh-CN.json                        中文显示名与简介
└── reference/
    ├── language-features.md              语言特性（泛型、迭代器、内置函数、类型系统）
    ├── standard-library.md               标准库新包与重要 API 变更
    ├── toolchain.md                      工具链（go 命令、vet、gofmt、go fix、PGO）
    ├── code-conventions.md               编码规范（错误处理、测试、并发、日志）
    └── migration-and-deprecations.md     版本迁移（行为变更、弃用项、GODEBUG 时间线）
```

## 覆盖范围

- **版本**：Go 1.19 至 1.27（九个大版本）
- **验证版本**：Go 1.27.3（windows/amd64）——API 已通过 `go doc` 核验
- **数据来源**：各版本官方 release notes

patch 版本只含 bug 修复和安全补丁，不引入新特性。本技能以大版本为单位描述特性。

## 安装方式

本技能遵循通用的 `SKILL.md` 约定：一个包含 `SKILL.md` 文件的目录，frontmatter 为 YAML 格式（`name`、`description`、`icon`）。任何通过扫描 `SKILL.md` 文件来发现技能的 agent 框架均可使用。

安装步骤：

1. 将 `go-modern/` 文件夹放到你的 agent 框架的技能扫描目录下。
2. 如果框架需要注册自定义路径，将该文件夹路径加入配置。
3. 重启或开启新会话。

具体路径请查阅你的 agent 框架文档。例如，MiMo Desktop 使用 `.mimocode/skills/`（项目级）或 `~/.config/mimocode/skills/`（全局），也支持通过 `mimocode.jsonc` 的 `skills.paths` 字段注册自定义路径。

`locales/` 目录是可选的——支持显示名本地化的框架会使用它，其他框架会安全忽略。

## 工作原理

1. agent 框架将 Go 相关请求与技能的 `description`（触发词：Go、Golang、.go、go.mod、generics、goroutine 等）匹配，加载 SKILL.md。
2. SKILL.md 根据当前任务类型，通过查找表告知 agent 读取哪个 `reference/` 文件。
3. agent 只加载相关的那个参考文件，保持上下文精简。

## 更新技能

release notes 源文件（`Go1.XX-release-notes.md`）位于上层工作目录。Go 发布新版本后刷新技能：

1. 添加新版本的 release notes 文件。
2. 更新 SKILL.md 的版本速查表和版本信息。
3. 将新特性/新包添加到对应的 `reference/` 文件。
4. 在已安装的 Go 版本上运行 `go doc <包>.<API>` 验证新 API。
