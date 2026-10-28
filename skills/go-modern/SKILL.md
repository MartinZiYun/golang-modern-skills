---
name: go-modern
description: "Modern Go development conventions and new-features reference (Go 1.19-1.27). Use when writing, modifying, reviewing, or generating .go files; operating on go.mod / go.work; discussing Go language features, standard library API selection, or go command toolchain usage; or when you need to confirm 'which version introduced feature X' or 'what is the modern replacement for this old pattern'. Covers generics evolution, range-over-func iterators, generic methods, slices/maps/cmp/log/slog, math/rand/v2, new(expr), go fix modernizers, new vet checks, testing/synctest, and more. Trigger words: Go, Golang, .go, go.mod, gofmt, go vet, go test, generics, goroutine, channel, Go code, Go feature."
icon: "🐹"
---

# go-modern — Modern Go Development Conventions & Features

Helps the Agent follow the latest Go conventions, prefer modern APIs, and avoid deprecated patterns when developing Go code. Covers Go 1.19 through 1.27.

## Version Info

- **Coverage**: Go 1.19 - 1.27 (nine major versions)
- **Verified against**: Go 1.27.3 (windows/amd64) - APIs mentioned in this skill have been checked with `go doc` on this version
- **Source**: official release notes for each version
- Patch releases (e.g. 1.27.0 -> 1.27.3) contain only bug fixes and security patches; they do not introduce new language features or standard library packages. This skill describes features at the major-version level; patch differences do not affect accuracy.
- To confirm whether an API is available under the current go.mod version, run `go doc <pkg>.<API>` or rely on `go vet`'s `stdversion` check.

## On-Demand Reading: Which File to Open

Detailed content is split by topic into the `reference/` directory. **Do not load everything at once** - read only the file relevant to the current task:

| Current task | Read |
|---------|------|
| Writing generics, range loops, iterators, `new(expr)`, type aliases, or other **language features** | `reference/language-features.md` |
| Choosing **standard library APIs** (slices / maps / cmp / log/slog / unique / context / errors / os, etc.) | `reference/standard-library.md` |
| Using the **go command / vet / gofmt / go fix / go mod** toolchain | `reference/toolchain.md` |
| Confirming **coding conventions** (error handling, naming, testing, project structure, concurrency) | `reference/code-conventions.md` |
| **Version upgrades**, investigating behavior changes, checking which APIs are deprecated | `reference/migration-and-deprecations.md` |

All paths are relative to this skill directory, i.e. `reference/<name>.md`.

## Version Cheatsheet (one line per version)

- **1.19** - Typed atomic values in `sync/atomic` (`atomic.Int64`, `atomic.Pointer[T]`); soft memory limit (`GOMEMLIMIT`); doc comments support links/lists and are formatted by gofmt; `unix` build constraint
- **1.20** - Slice-to-array conversion (`[4]byte(s)`); `errors.Join`; PGO preview; program-level coverage (`go build -cover` + `GOCOVERDIR`); cgo disabled by default when no C toolchain
- **1.21** - `min`/`max`/`clear` built-ins; `slices`/`maps`/`cmp`/`log/slog` packages; `context` gains `WithoutCancel`/`WithTimeoutCause`/`Cause`/`AfterFunc`; GODEBUG compatibility mechanism; automatic toolchain selection; PGO generally available
- **1.22** - Per-iteration loop variables (eliminates capture bug); `range` over integers (`for i := range 10`); `math/rand/v2` (first v2 stdlib package)
- **1.23** - `range` over function iterators (`range-over-func`); `iter`/`unique`/`structs` packages; `slices`/`maps` iterator functions; `Timer`/`Ticker` become GC-eligible with unbuffered channels
- **1.24** - Generic type aliases (stable); `os.Root` directory sandbox; `testing.B.Loop()`; `runtime.AddCleanup` (replaces `SetFinalizer`); `weak` package; go.mod `tool` directive; `crypto/mlkem`/`hkdf`/`pbkdf2`/`sha3`
- **1.25** - `testing/synctest` (stable); container-aware `GOMAXPROCS`; trace flight recorder; `go doc -http`; `encoding/json/v2` (experimental)
- **1.26** - `new(expr)` with initial value; generic self-referential constraints (`type Adder[A Adder[A]]`); `go fix` rewritten as modernizers + inliner; Green Tea GC enabled by default; goroutine leak profile (experimental)
- **1.27** - Generic methods (methods can declare type parameters); `encoding/json/v2` + `jsontext` stable; `uuid` package; `crypto/mldsa`; goroutine leak profile GA; `go test` runs `stdversion` check by default

## High-Frequency Quick Reference (usable without reading reference files)

These are the most commonly needed modern patterns - apply directly:

```go
// Built-in functions (1.21+)
m := map[string]int{"a": 1}
clear(m)           // empties the map
s := make([]int, 5)
clear(s)           // zeroes the slice
largest := max(a, b, c)
smallest := min(a, b)

// range over integers (1.22+)
for i := range 10 { }        // i: 0..9

// range over function iterators (1.23+)
for k, v := range maps.All(m) { }
for _, v := range slices.Values(s) { }
for i := range slices.Backward(s) { }

// Structured logging (1.21+) - replaces fmt.Println/log.Printf
slog.Info("user login", "uid", uid, "ip", ip)
slog.Error("db failed", "err", err)

// Slice/map utilities (1.21+) - replaces hand-written loops
slices.Sort(s)
slices.Contains(s, v)
slices.Index(s, v)
slices.SortFunc(s, cmp.Compare)
maps.Copy(dst, src)

// Random numbers (1.22+, prefer math/rand/v2)
n := rand.IntN(100)
dur := rand.N(5 * time.Minute)   // generic, any integer type

// Typed atomic values (1.19+)
var counter atomic.Int64
counter.Add(1)
var ptr atomic.Pointer[Config]
ptr.Store(&Config{...})

// Benchmark loop (1.24+)
func BenchmarkX(b *testing.B) {
    setup()            // runs once
    for b.Loop() {     // replaces for i := 0; i < b.N; i++
        work()
    }
}
```

## Core Principles (always follow when writing Go code)

1. **Prefer new stdlib packages**: use `slices`/`maps`/`cmp` instead of hand-written loops; use `log/slog` instead of `log`/`fmt` for logging.
2. **Prefer v2 packages**: `math/rand/v2` over `math/rand` (use v2 directly in new code).
3. **Prefer typed APIs**: `atomic.Int64`/`atomic.Pointer[T]` over `atomic.Value` or manual `Load`/`Store`.
4. **Prefer the new cleanup mechanism**: `runtime.AddCleanup` over `runtime.SetFinalizer` (1.24+).
5. **Modernize with `go fix`**: 1.26+ `go fix` auto-migrates to new APIs; run it after writing code.
6. **Check go.mod version first**: the `go` line in go.mod determines available features; confirm the version when unsure. `go vet`'s `stdversion` check flags code using too-new APIs.
7. **Loop variables no longer have a capture bug**: 1.22+ creates a new variable per iteration; no need to manually copy loop variables into locals.
8. **Use `errors.Join`** to combine multiple errors (1.20+); use `errors.Is`/`errors.As` to inspect.

## Behavior Change Reminders (easy to trip over)

These changes may affect existing code behavior. Always check `reference/migration-and-deprecations.md` when upgrading:

- **1.22**: for-loop variable semantics changed (new variable per iteration) - code relying on the old semantics will behave differently.
- **1.23**: `Timer`/`Ticker` channels are now unbuffered (`len`/`cap` return 0 instead of 1).
- **1.25**: compiler fix for delayed nil-pointer checks - code that uses a return value before checking error will now panic.
- **1.26**: Green Tea GC enabled by default; `go mod init` defaults to a lower go version.
- **1.27**: `asynctimerchan` GODEBUG permanently removed; time channels are always unbuffered.
