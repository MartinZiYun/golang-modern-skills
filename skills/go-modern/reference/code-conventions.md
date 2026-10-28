# Development Conventions (Modern Go, 1.19-1.27)

Conventions the Agent should follow when writing Go code. Each item contrasts "use" vs "don't use" where possible, with the relevant version noted.

## API Selection Priority

When a modern replacement exists, prefer it in new code:

| Scenario | Prefer | Avoid |
|----------|--------|-------|
| Sort/search/delete slices | `slices.Sort`/`Contains`/`Index`/`Delete` (1.21+) | Hand-written for loops |
| Copy/equal/delete map entries | `maps.Copy`/`Equal`/`DeleteFunc` (1.21+) | Hand-written loops |
| Comparison | `cmp.Compare`/`cmp.Less` (1.21+) | Manual `if a < b` wrappers |
| Logging | `log/slog` (1.21+) | `log.Printf` / `fmt.Println` |
| Random numbers | `math/rand/v2` (1.22+) | `math/rand` (in new code) |
| Atomic values | `atomic.Int64`/`Pointer[T]` (1.19+) | `atomic.Value` + manual Load/Store |
| Object cleanup | `runtime.AddCleanup` (1.24+) | `runtime.SetFinalizer` |
| N-iteration loop | `for i := range n` (1.22+) | `for i := 0; i < n; i++` |
| Iterate slice/map | `range slices.Values(s)` / `maps.All(m)` (1.23+) | Index loop or Keys-then-iterate |
| Min/max | `min`/`max` (1.21+) | Hand-written `if` ternary |
| Clear container | `clear` (1.21+) | Hand-written zeroing loop |
| Benchmark loop | `for b.Loop()` (1.24+) | `for i := 0; i < b.N; i++` |
| User-path file ops | `os.OpenRoot` (1.24+) | Bare `os.Open` + manual path validation |
| Build host:port | `net.JoinHostPort` | `fmt.Sprintf("%s:%d", ...)` |
| Optional field pointer init | `new(expr)` (1.26+) | `v := expr; &v` |

## Error Handling

### Check error before using the return value

```go
// Correct
f, err := os.Open(path)
if err != nil {
    return err
}
defer f.Close()
name := f.Name()          // safe: err is confirmed nil

// Wrong - panics from 1.25 (compiler fixed the delayed nil-pointer check bug)
f, err := os.Open(path)
name := f.Name()          // f may be nil when err is non-nil
if err != nil { return err }
```

### Combine errors with errors.Join (1.20+)

```go
// Prefer
return errors.Join(err1, err2)

// Instead of manual formatting
return fmt.Errorf("%v; %v", err1, err2)   // loses errors.Is/As capability
```

### Wrap errors with %w

```go
return fmt.Errorf("read config: %w", err)  // preserves the error chain for errors.Is
// Note (1.27 vet): do not wrap *E with %w when E already implements error; wrap E directly
```

### Context cancellation cause (1.21+)

```go
ctx, cancel := context.WithTimeoutCause(parent, timeout, fmt.Errorf("api %s timeout", name))
defer cancel()
if err := context.Cause(ctx); err != nil {
    // Gets the specific cause rather than the generic context.DeadlineExceeded
}
```

## Loops & Concurrency

### No need to manually copy loop variables (1.22+)

```go
// 1.22+: safe, each iteration is an independent variable
for _, item := range items {
    go process(item)              // no need for item := item
}

// Only 1.21 and earlier needed the copy:
for _, item := range items {
    item := item                  // required in old versions
    go process(item)
}
```

### WaitGroup.Add outside the goroutine (vet 1.25+ checks)

```go
// Correct
var wg sync.WaitGroup
for _, item := range items {
    wg.Add(1)                     // count before starting the goroutine
    go func() { defer wg.Done(); process(item) }()
}
wg.Wait()

// Wrong (vet reports)
go func() { wg.Add(1); defer wg.Done(); ... }()  // Add inside goroutine; Wait may run before Add executes
```

### Timer/Ticker (1.23+, GC-eligible)

From 1.23, unreferenced Timers/Tickers are GC-eligible even without Stop. But **still call Stop** for clarity:

```go
timer := time.NewTimer(duration)
defer timer.Stop()               // good habit, even though GC can reclaim it
```

Note: from 1.23 timer channels are unbuffered; `len(timer.C)` returns 0. Check for a pending value with a non-blocking receive:

```go
select {
case <-timer.C:
    // fired
default:
    // not yet
}
```

### Avoid goroutine leaks from early returns

```go
// Leak risk - early return leaves remaining goroutines blocked forever
ch := make(chan result)          // unbuffered
for range items { go func() { ch <- work() }() }
for range items {
    r := <-ch
    if r.err != nil { return r.err }   // remaining goroutines leak
}

// Fix: use a buffered channel or context cancellation
ch := make(chan result, len(items))
// or use errgroup
```

From 1.26+ the goroutine leak profile can detect such leaks.

## Testing

### Use B.Loop for benchmarks (1.24+)

```go
func BenchmarkProcess(b *testing.B) {
    data := setupLargeFixture()    // runs once
    b.Cleanup(cleanup)
    for b.Loop() {                 // compiler won't optimize away the call
        process(data)
    }
}
```

### Correct declaration format (vet tests analyzer, 1.24+)

- `TestXxx(t *testing.T)` - test function name starts uppercase
- `BenchmarkXxx(b *testing.B)`
- `FuzzXxx(f *testing.F)`
- `ExampleXxx()` - example function name format

Malformed declarations cause tests to not run; vet reports them.

### Concurrent testing with synctest (1.25+)

Use `testing/synctest` for concurrent logic involving timers/timeouts - virtual time eliminates real waiting.

## Project Structure & Dependencies

### Tool dependencies via tool directive (1.24+)

```bash
# Old way: tools.go file + blank import
# go run golang.org/x/tools/cmd/stringer

# New way (1.24+):
go get -tool golang.org/x/tools/cmd/stringer    # writes a tool directive to go.mod
go tool stringer                                 # run it
go get tool                                      # upgrade all tools
go install tool                                  # install to GOBIN
```

### go.mod version determines available features

The `go` line in go.mod is a feature gate. `go vet`'s `stdversion` analyzer flags code using too-new APIs. **Check the target go version before writing code.**

```bash
# Check the module's requirement
head -1 go.mod
# go 1.23.0  <- features from 1.23 and earlier are available; 1.24+ features are flagged by vet
```

### GODEBUG compatibility (1.21+)

Upgrading the toolchain while keeping the go.mod go line unchanged preserves old behavior. Control explicitly with the `godebug` directive in go.mod (1.23+):

```
godebug asynctimerchan=1    // declared in go.mod (1.23+)
```

## Logging Conventions

### Structured logging (1.21+)

```go
// Prefer: slog, structured key-value pairs
slog.Info("request handled",
    "method", r.Method,
    "path", r.URL.Path,
    "duration", elapsed,
    "status", status,
)

// Avoid: unstructured formatting
log.Printf("method=%s path=%s duration=%v status=%d", ...)
fmt.Printf("[%s] %s %s %v %d\n", ...)
```

Use `slog.With` to attach fixed attributes (service name, request ID) and avoid repeating them in every log call.

## Code Style

### Doc comments (1.19+ formatting)

gofmt reformats doc comments. Use the standard syntax:

```go
// Package foo provides bar functionality.
//
// Foo does X. It works by:
//   - step one
//   - step two
//
// See [pkg/path] for related types.
//
// # heading
//
// [text]: https://example.com
package foo
```

gofmt aligns list indentation and fixes heading format automatically.

### Always run go vet

`go vet` is a subset of `go test`. New analyzers across versions cover common mistakes (see `reference/toolchain.md`). **Always run `go vet ./...` before committing.**

### Modernize with go fix (1.26+)

```bash
go fix ./...     # auto-migrate to modern APIs and patterns
```

It does not change program behavior and is safe to run.
