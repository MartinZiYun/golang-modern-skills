# golang-modern-skills

**English** | [简体中文](README.zh-CN.md)

An agent skill that keeps AI coding agents on modern Go conventions and aware of language changes across **Go 1.19 → 1.27**.

The skill is `skills/go-modern/`. It follows the common `SKILL.md` convention, so any agent framework that discovers skills by scanning for `SKILL.md` files can use it.

## What It Provides

When an agent writes, reviews, or generates Go code, the skill supplies:

| Area | Typical guidance |
|------|------------------|
| Modern APIs | Prefer `slices`/`maps`/`cmp`, `log/slog`, `math/rand/v2`, `atomic.Int64`/`atomic.Pointer[T]`; avoid deprecated patterns |
| New language features | Generics evolution, generic methods (1.27), range-over-func iterators (1.23), `new(expr)` (1.26), per-iteration loop variables (1.22) |
| Toolchain usage | `go vet` analyzers, `go fix` modernizers (1.26+), `go doc -http`, PGO, `go mod` / `tool` directives |
| Upgrade safety | Behavior changes, deprecations, and the `GODEBUG` timeline that can break existing code |

## Design: On-Demand Reading

Context is the scarce resource, so the skill is layered:

- **`SKILL.md`** - the only file loaded up front: trigger rules, version cheatsheet, high-frequency quick reference, and a task→file lookup table.
- **`references/*.md`** - detailed topic files the agent opens **only** when the current task needs them.

A request about `go vet` loads `references/toolchain.md`; a request about generics loads `references/language-features.md`. Nothing else is pulled in.

## Repository Layout

```
golang-modern-skills/
├── README.md                            English readme (this file)
├── README.zh-CN.md                      Chinese readme
├── LICENSE                              Apache License 2.0
├── docs/                                Source material: official release notes, Go 1.19-1.27
│   └── Go1.XX-release-notes.md
└── skills/
    └── go-modern/                       The skill (install this folder)
        ├── SKILL.md                     Entry: triggers + version cheatsheet + quick reference + lookup table
        ├── locales/                     Optional display-name localization
        │   ├── en-US.json
        │   └── zh-CN.json
        └── references/                  Topic files, loaded on demand
            ├── language-features.md
            ├── standard-library.md
            ├── toolchain.md
            ├── code-conventions.md
            └── migration-and-deprecations.md
```

## Reference Files

| File | Covers | Read when |
|------|--------|-----------|
| `language-features.md` | Generics, iterators, built-ins, type system | Writing generics, range loops, `new(expr)`, type aliases |
| `standard-library.md` | New packages and key API changes | Choosing stdlib APIs (`slices`, `maps`, `cmp`, `log/slog`, `unique`, `os`, ...) |
| `toolchain.md` | `go` command, `vet`, `gofmt`, `go fix`, PGO | Touching build, test, or module tooling |
| `code-conventions.md` | Error handling, naming, testing, concurrency, logging | Confirming idiomatic style or reviewing code |
| `migration-and-deprecations.md` | Behavior changes, deprecations, `GODEBUG` timeline | Upgrading a Go version or diagnosing an upgrade regression |

## Coverage

- **Versions**: Go 1.19 through 1.27 (nine major versions)
- **Verified against**: Go 1.27.3 (windows/amd64) - APIs checked with `go doc`
- **Source**: official release notes for each version

Patch releases (e.g. 1.27.0 → 1.27.3) only contain bug and security fixes, so features are described at major-version granularity.

To confirm whether an API exists under your `go.mod` version, run `go doc <pkg>.<API>`, or rely on `go vet`'s `stdversion` check.

## Installation

This skill follows the common `SKILL.md` convention: a directory containing `SKILL.md` with YAML frontmatter (`name`, `description`, `icon`).

1. Copy the `skills/go-modern/` folder into your agent framework's skill scan directory.
2. If your framework requires registering custom paths, add that folder path to its config.
3. Restart the agent or start a new session.

Consult your framework's documentation for the exact discovery path. For example, MiMo Desktop scans `.mimocode/skills/` (project-level) or `~/.config/mimocode/skills/` (global), and accepts custom paths via `skills.paths` in `mimocode.jsonc`.

The `locales/` directory is optional - frameworks that support display-name localization use it; others ignore it.

## How It Works

1. The framework matches Go-related requests against the skill's `description` (trigger words: Go, Golang, `.go`, `go.mod`, `gofmt`, `go vet`, generics, goroutine, channel, ...) and loads `SKILL.md`.
2. `SKILL.md` maps the current task type to one `references/` file via its lookup table.
3. The agent reads only that file. All paths in `SKILL.md` are relative to the skill directory, i.e. `references/<name>.md`.

## Updating the Skill

Release notes are vendored under `docs/`. After a new Go release:

1. Add `docs/Go1.XX-release-notes.md`.
2. Update the version cheatsheet and version info in `SKILL.md`.
3. Add new features and packages to the matching `references/` file.
4. Verify new APIs on the installed toolchain with `go doc <pkg>.<API>`.

## License

[Apache License 2.0](LICENSE)
