# Toolchain: go command / vet / gofmt / go fix (Go 1.19-1.27)

Organized by tool, with the introducing version noted. Toolchain changes directly affect development workflow and code-quality checks.

## go fix - Code Modernization (1.26 rewrite)

Starting in 1.26, `go fix` was completely rewritten as Go's **modernizers** platform: it automatically migrates code to modern APIs and patterns. Built on the same analysis framework as `go vet`.

```bash
go fix ./...          # apply all modernizer fixes
go fix -diff ./...    # show diff without modifying (where supported)
```

The initial suite includes dozens of fixers covering language features and core library API migrations. 1.27 adds:

- `atomictypes` - migrate to typed atomic values (`atomic.Int64`, etc.)
- `embedlit` - literal embedding improvements
- `slicesbackward` - migrate to `slices.Backward` and other iterator patterns
- `unsafefuncs` - modernize unsafe usage

Also includes a source-level **inliner** supporting `//go:fix inline` directives for custom API migrations (library authors can provide automated migration paths for consumers).

**Recommendation**: run `go fix ./...` after writing code to adopt modern patterns automatically.

## go vet - New Static Checks

`go vet` (and the vet subset built into `go test`) gains new analyzers across versions:

| Analyzer | Version | Description |
|----------|---------|-------------|
| `errorsas` | 1.19 | `errors.As` called with `*error` as second argument |
| loop var capture | 1.20 | References to loop variables after `T.Parallel()` in subtests (under old semantics) |
| `timeformat` | 1.20 | `Time.Format`/`time.Parse` using `2006-02-01` (yyyy-dd-mm, easily confused with ISO 8601) |
| append no values | 1.22 | `append(slice)` with no values (no-op, almost certainly a bug) |
| defer `time.Since` | 1.22 | `defer log.Println(time.Since(t))` - `time.Since` is evaluated at defer time, not when deferred fn runs |
| `log/slog` key-value | 1.22 | slog calls where a key-position arg is not string/Attr, or missing a final value |
| `stdversion` | 1.23 | References to stdlib symbols too new for the go.mod version (`go test` runs this by default from 1.27) |
| `tests` | 1.24 | Malformed test/benchmark/fuzz/example declarations (names, signatures, documenting nonexistent identifiers) |
| `printf` non-constant format | 1.24 | `fmt.Printf(s)` with no extra args (s may contain `%`; use `fmt.Print` instead) |
| `buildtag` point version | 1.24 | `//go:build go1.23.1` references a point release (should be `go1.23`) |
| `copylock` 3-clause loop | 1.24 | `for i := iter(); ...; i = next(i)` where the variable contains a `sync.Locker` |
| `waitgroup` | 1.25 | Misplaced `sync.WaitGroup.Add` calls |
| `hostport` | 1.25 | `fmt.Sprintf("%s:%d", host, port)` to build an address (breaks IPv6; use `net.JoinHostPort`) |
| `printf` %w pointer | 1.27 | `fmt.Errorf("...: %w", p)` where p is `*E` and `E` implements `error` (wrapping a pointer is usually a bug) |

## gofmt

- **1.19**: reformats doc comments to make their rendered meaning clearer, supporting links, lists, and headings. See [Go Doc Comments](https://go.dev/doc/comment) for syntax.
- **1.27**: fixes the column-alignment logic, which previously caused gofmt output to vary slightly across CPU architectures. Running gofmt from 1.27 on previously formatted code may produce minor whitespace changes.

## go command

### go.mod management

| Feature | Version | Description |
|---------|---------|-------------|
| `tool` directive | 1.24 | Track executable tool dependencies in go.mod, replacing the `tools.go` blank-import workaround. `go tool` runs them; `go get -tool` adds; `go get tool` / `go install tool` batch-operate |
| `godebug` directive | 1.23 | Declare GODEBUG settings in go.mod / go.work |
| `ignore` directive | 1.25 | Directories the go command should ignore when matching `all`/`./...` (still included in module zip) |
| `go mod tidy` merges require | 1.27 | go 1.27+ auto-merges duplicate require blocks into the standard two-block structure (direct + indirect) |
| `go mod tidy -diff` | 1.23 | Prints needed changes as a unified diff without modifying files |
| `go mod init` lower default | 1.26 | A 1.N.X toolchain creates `go 1.(N-1).0`, encouraging compatibility with supported versions |
| No automatic toolchain line | 1.25 | Updating the go line no longer appends a toolchain line |

### Build & test

| Feature | Version | Description |
|---------|---------|-------------|
| `-C dir` | 1.20 | Change directory before running (must be first flag from 1.21) |
| `-skip pattern` | 1.20 | `go test`/`go generate` skip matching items |
| `go build -cover` | 1.20 | Build with coverage instrumentation, paired with `GOCOVERDIR` for integration-test coverage |
| `go build -json` | 1.24 | Build output/failures reported as JSON |
| `go test -json` build output | 1.24 | Build output also emitted as JSON interleaved with test results |
| `go test -fullpath` | 1.21 | Print full paths in test logs |
| `go test -c` multi-package | 1.21 | Build test binaries for multiple packages |
| `go test` default stdversion | 1.27 | Runs the stdversion vet check by default |
| VCS version in binary | 1.24 | `go build` sets the main module version from VCS tag/commit; `+dirty` suffix for uncommitted changes |
| `GOAUTH` | 1.24 | Authentication for private module fetches |
| workspace vendor | 1.22 | `go work vendor` supports a workspace vendor directory |
| `go env -changed` | 1.23 | Print only settings that differ from defaults |
| `go version -m -json` | 1.25 | JSON output of BuildInfo embedded in binaries |
| `go build -asan` leak detection | 1.25 | Default leak detection at program exit |

### Documentation

| Feature | Version | Description |
|---------|---------|-------------|
| `go doc -http` | 1.25 | Start a documentation server and open a browser |
| `go doc pkg@version` | 1.27 | View docs for a specific package version |
| `go doc -ex` | 1.27 | List executable examples and print their source |
| `cmd/doc` removed | 1.26 | `go tool doc` deleted; use `go doc` |

### Toolchain selection & compatibility (1.21)

Important mechanisms introduced in 1.21:

- **`go` line as strict minimum**: `go 1.21.0` in go.mod means it cannot be built with 1.20.
- **Automatic toolchain selection**: the go command can download/switch to the Go version required by go.mod.
- **`toolchain` directive**: sets a suggested minimum toolchain version (may be newer than the go line).
- **GODEBUG compatibility**: upgrading the toolchain while keeping the go.mod go line unchanged preserves older behavior. The latest toolchain is always the best implementation of an older Go version.

### cgo (1.20 default change)

On systems without a C toolchain (`CGO_ENABLED` unset, `CC` unset, no default C compiler), `CGO_ENABLED` defaults to `0`. Affects `net`, `os/user`, `plugin` packages (already pure-Go on macOS).

1.24 adds cgo annotations: `#cgo noescape` / `#cgo nocallback` to optimize C function calls.

### Profile-Guided Optimization (PGO)

- **1.20**: preview. `-pgo` flag specifies a pprof profile file.
- **1.21**: generally available. `-pgo=auto` is the default; a `default.pgo` in the main package directory enables it. Typical 2-7% improvement.
- **1.22**: stronger devirtualization, 2-14% improvement.
- **1.23**: PGO build overhead reduced from 100%+ to single-digit percentages.

### Telemetry (1.23)

`go telemetry on` enables anonymous usage statistics upload; default `go telemetry local` collects locally only. Disable via `go env`.

### Coverage for programs (1.20)

Coverage collection extends beyond unit tests to applications and integration tests:

```bash
go build -cover -o myapp
GOCOVERDIR=./coverage ./myapp        # writes coverage files at runtime
go tool covdata merge -i ./coverage -o profile.cov
```

The `runtime/coverage` package (1.20) supports writing coverage for long-running programs that do not exit via `os.Exit`.
