# go-modern

An agent skill that helps AI coding agents follow modern Go development conventions and use new language features (Go 1.19-1.27).

## Overview

When an agent writes, reviews, or generates Go code, this skill provides:

- Up-to-date conventions (prefer `slices`/`maps`/`log/slog`, avoid deprecated APIs)
- New-feature guidance (generics evolution, range-over-func iterators, generic methods, `new(expr)`, etc.)
- Behavior-change warnings for version upgrades

Designed for **on-demand reading**: the entry file (SKILL.md) is a concise index with a high-frequency quick reference. Detailed content is split into topic-specific files under `reference/` that the agent loads only when relevant to the current task, minimizing context usage.

## File Structure

```
go-modern/
├── SKILL.md                              Entry: trigger rules + version cheatsheet + quick reference + on-demand index
├── README.md                             This file (English)
├── README.zh-CN.md                       Chinese readme
├── locales/                              Optional: display-name localization (for frameworks that support it)
│   ├── en-US.json                        English display name & brief
│   └── zh-CN.json                        Chinese display name & brief
└── reference/
    ├── language-features.md              Generics, iterators, built-ins, type system
    ├── standard-library.md               New packages & key API changes
    ├── toolchain.md                      go command, vet, gofmt, go fix, PGO
    ├── code-conventions.md               Error handling, testing, concurrency, logging
    └── migration-and-deprecations.md     Behavior changes, deprecations, GODEBUG timeline
```

## Coverage

- **Versions**: Go 1.19 through 1.27 (nine major versions)
- **Verified against**: Go 1.27.3 (windows/amd64) - APIs checked with `go doc`
- **Source**: official release notes for each version

Patch releases contain only bug fixes and security patches; they do not introduce new features. This skill describes features at the major-version level.

## Installation

This skill follows the common `SKILL.md` convention: a directory containing a `SKILL.md` file with YAML frontmatter (`name`, `description`, `icon`). It works with any agent framework that discovers skills by scanning for `SKILL.md` files.

To install:

1. Place the `go-modern/` folder in your agent framework's skill scan directory.
2. If your framework requires registering custom paths, add the folder path to its configuration.
3. Restart or start a new session.

Consult your agent framework's documentation for the exact skill discovery path. For example, MiMo Desktop uses `.mimocode/skills/` (project-level) or `~/.config/mimocode/skills/` (global), and supports custom paths via the `skills.paths` field in `mimocode.jsonc`.

The `locales/` directory is optional - frameworks that support display-name localization will use it; others will safely ignore it.

## How It Works

1. The agent framework matches Go-related requests against the skill's `description` (trigger words: Go, Golang, .go, go.mod, generics, goroutine, etc.) and loads SKILL.md.
2. SKILL.md tells the agent which `reference/` file to read based on the current task (a lookup table maps task type to file).
3. The agent reads only the relevant reference file, keeping context lean.

## Updating the Skill

The release notes source files (`Go1.XX-release-notes.md`) live in the parent working directory. To refresh the skill after a new Go release:

1. Add the new version's release notes file.
2. Update SKILL.md's version cheatsheet and version info.
3. Add new features/packages to the relevant `reference/` file.
4. Run `go doc <pkg>.<API>` to verify new APIs on the installed Go version.
