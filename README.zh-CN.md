# golang-modern-skills

[English](README.md) | **简体中文**

一个 agent 技能（skill），帮助 AI 编码 agent 遵循现代 Go 开发规范、掌握 Go 1.19 → 1.27 的语言变化。

技能本体位于 `skills/go-modern/`，遵循通用的 `SKILL.md` 约定，任何通过扫描 `SKILL.md` 来发现技能的 agent 框架都可直接使用。

## 提供什么

当 agent 编写、审查或生成 Go 代码时，本技能提供：

| 方面 | 典型指引 |
|------|----------|
| 现代 API | 优先 `slices`/`maps`/`cmp`、`log/slog`、`math/rand/v2`、`atomic.Int64`/`atomic.Pointer[T]`；避免已弃用写法 |
| 新语言特性 | 泛型演进、generic methods（1.27）、range-over-func 迭代器（1.23）、`new(expr)`（1.26）、逐迭代循环变量（1.22） |
| 工具链用法 | `go vet` 分析器、`go fix` 现代化改写（1.26+）、`go doc -http`、PGO、`go mod` / `tool` 指令 |
| 升级安全 | 行为变更、弃用项、可能破坏现有代码的 `GODEBUG` 时间线 |

## 设计：按需读取

上下文是稀缺资源，因此技能分层组织：

- **`SKILL.md`** —— 唯一预先加载的文件：触发规则、版本速查表、高频速查、任务→文件查找表。
- **`references/*.md`** —— 各主题详解，agent **只在**当前任务需要时才打开。

问 `go vet` 就只加载 `references/toolchain.md`，问泛型就只加载 `references/language-features.md`，其余内容不会被带进上下文。

> 注：技能正文（`SKILL.md` 与 `references/` 下的文件）使用英文编写，更适合 AI 处理；本 README 为中文说明。

## 仓库结构

```
golang-modern-skills/
├── README.md                            英文说明
├── README.zh-CN.md                      本文件（中文说明）
├── LICENSE                              Apache License 2.0
├── docs/                                原始素材：Go 1.19-1.27 官方 release notes
│   └── Go1.XX-release-notes.md
└── skills/
    └── go-modern/                       技能本体（安装时拷贝此目录）
        ├── SKILL.md                     入口：触发规则 + 版本速查 + 高频速查 + 查找表
        ├── locales/                     可选的显示名本地化
        │   ├── en-US.json
        │   └── zh-CN.json
        └── references/                  按需加载的主题文件
            ├── language-features.md
            ├── standard-library.md
            ├── toolchain.md
            ├── code-conventions.md
            └── migration-and-deprecations.md
```

## 参考文件

| 文件 | 内容 | 何时读取 |
|------|------|----------|
| `language-features.md` | 泛型、迭代器、内置函数、类型系统 | 写泛型、range 循环、`new(expr)`、类型别名时 |
| `standard-library.md` | 新包与重要 API 变更 | 选择标准库 API 时（`slices`、`maps`、`cmp`、`log/slog`、`unique`、`os` 等） |
| `toolchain.md` | `go` 命令、`vet`、`gofmt`、`go fix`、PGO | 涉及构建、测试、模块工具链时 |
| `code-conventions.md` | 错误处理、命名、测试、并发、日志 | 确认惯用写法或审查代码时 |
| `migration-and-deprecations.md` | 行为变更、弃用项、`GODEBUG` 时间线 | 升级 Go 版本或排查升级引入的问题时 |

## 覆盖范围

- **版本**：Go 1.19 至 1.27（九个大版本）
- **验证版本**：Go 1.27.3（windows/amd64）——API 已通过 `go doc` 核验
- **数据来源**：各版本官方 release notes

patch 版本（如 1.27.0 → 1.27.3）只含 bug 修复与安全补丁，因此特性以大版本为单位描述。

要确认某个 API 在当前 `go.mod` 版本下是否可用，运行 `go doc <包>.<API>`，或依赖 `go vet` 的 `stdversion` 检查。

## 安装方式

本技能遵循通用的 `SKILL.md` 约定：一个包含 `SKILL.md` 的目录，frontmatter 为 YAML 格式（`name`、`description`、`icon`）。

1. 将 `skills/go-modern/` 目录拷贝到你的 agent 框架的技能扫描目录下。
2. 如果框架需要注册自定义路径，把该目录路径加入配置。
3. 重启 agent 或开启新会话。

具体路径请查阅你的框架文档。例如 MiMo Desktop 扫描 `.mimocode/skills/`（项目级）或 `~/.config/mimocode/skills/`（全局），也支持通过 `mimocode.jsonc` 的 `skills.paths` 字段注册自定义路径。

`locales/` 目录是可选的——支持显示名本地化的框架会使用它，其他框架会安全忽略。

## 工作原理

1. 框架将 Go 相关请求与技能的 `description`（触发词：Go、Golang、`.go`、`go.mod`、`gofmt`、`go vet`、generics、goroutine、channel 等）匹配，加载 `SKILL.md`。
2. `SKILL.md` 通过查找表把当前任务类型映射到某一个 `references/` 文件。
3. agent 只读取该文件。`SKILL.md` 中的所有路径都相对于技能目录，即 `references/<name>.md`。

## 更新技能

release notes 已归档在 `docs/`。Go 发布新版本后：

1. 添加 `docs/Go1.XX-release-notes.md`。
2. 更新 `SKILL.md` 的版本速查表与版本信息。
3. 将新特性/新包补充到对应的 `references/` 文件。
4. 在已安装的 Go 工具链上用 `go doc <包>.<API>` 验证新 API。

## 许可证

[Apache License 2.0](LICENSE)
