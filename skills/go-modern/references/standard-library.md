# Standard Library: New Packages & Key Changes (Go 1.19-1.27)

Organized by functional area, with the introducing version noted. Prefer the modern replacements listed here when choosing APIs.

## Slices / Maps / Comparison

### slices (1.21; iterator functions added in 1.23)

Generic slice operations, replacing hand-written loops. Common functions:

```go
slices.Sort(s)                          // sort (requires Ordered elements)
slices.SortFunc(s, cmp.Compare)         // sort with custom comparison
slices.SortStableFunc(s, less)          // stable sort
slices.Reverse(s)                       // reverse in place
slices.Contains(s, v)                   // containment check
slices.Index(s, v)                      // find index, -1 if not found
slices.IndexFunc(s, fn)                 // find by predicate
slices.Delete(s, i, j)                  // delete range
slices.Clone(s)                         // shallow copy
slices.Compact(s)                       // remove consecutive duplicates
slices.CompactFunc(s, eq)               // remove duplicates by predicate
slices.Concat(a, b, c)                  // 1.22+ concatenate multiple slices
slices.Sorted(seq)                      // 1.23+ collect from iterator and sort
slices.SortedFunc(seq, cmp)             // 1.23+
slices.Collect(seq)                     // 1.23+ collect iterator into new slice
slices.Chunk(s, n)                      // 1.23+ chunked iterator
```

Iterator functions from 1.23 (paired with range-over-func):

```go
for i, v := range slices.All(s) { }      // index + value
for v := range slices.Values(s) { }      // values only
for i, v := range slices.Backward(s) { } // reverse
```

### maps (1.21; iterator functions added in 1.23)

Generic map operations:

```go
maps.Copy(dst, src)                     // copy
maps.Clone(m)                           // shallow copy
maps.Equal(a, b)                        // equality
maps.EqualFunc(a, b, eq)                // equality by predicate
maps.DeleteFunc(m, fn)                  // delete by predicate
maps.Keys(m)                            // slice of keys (1.21, unordered)
maps.Values(m)                          // slice of values (1.21, unordered)
```

Iterator functions from 1.23 (paired with range-over-func, **preferred over Keys/Values slices**):

```go
for k, v := range maps.All(m) { }
for k := range maps.Keys(m) { }         // iterator version, no slice allocation
for v := range maps.Values(m) { }
m2 := maps.Collect(seq)                 // collect iterator into a map
maps.Insert(m, seq)                     // insert from iterator
```

### cmp (1.21)

Defines the `Ordered` constraint and `Compare`/`Less` functions for use with `slices.SortFunc`:

```go
slices.SortFunc(nums, cmp.Compare)       // ascending
slices.SortFunc(nums, func(a, b int) int {
    return cmp.Compare(b, a)             // descending
})
```

## Logging

### log/slog (1.21)

Structured logging with key-value pairs. **New code should use slog instead of the `log` package and `fmt.Println` for logging.**

```go
// Quick usage (default JSON handler)
slog.Info("user login", "uid", uid, "ip", ip)
slog.Error("db failed", "err", err)
slog.With("service", "auth").Info("started")  // with fixed attributes

// Custom handler
logger := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelDebug,
}))
slog.SetDefault(logger)

// Context and grouping
ctx := context.Background()
logger.InfoContext(ctx, "request", "method", "GET")
```

`testing/slogtest` (1.21) validates custom `slog.Handler` implementations.

## Random Numbers

### math/rand/v2 (1.22)

The first v2 package in the standard library. **Prefer `math/rand/v2` in new code.** Key improvements:

```go
import "math/rand/v2"

n := rand.IntN(100)                      // replaces rand.Intn (unified naming)
u := rand.Uint64()
f := rand.Float64()
dur := rand.N(5 * time.Minute)           // generic: any integer type, replaces manual Int63n
```

Differences from `math/rand`:
- Global generator is unconditionally randomly seeded (no manual `rand.Seed` needed)
- `Read` method removed (use `crypto/rand` for cryptographic randomness)
- `Source` interface simplified to a single `Uint64` method
- Uses faster algorithms (PCG, ChaCha8)
- Adds `Uint32`/`Uint32N`/`Uint64N`/`UintN`

## Atomic Operations

### Typed Atomic Values in sync/atomic (1.19)

**Prefer typed APIs** over `atomic.Value` and manual `Load`/`Store`:

```go
var counter atomic.Int64
counter.Store(0)
counter.Add(1)
v := counter.Load()

var flag atomic.Bool
flag.Store(true)

var ptr atomic.Pointer[Config]          // generic pointer, no unsafe.Pointer conversion
ptr.Store(&Config{...})
c := ptr.Load()

var u atomic.Uintptr
```

These types hide the underlying value, forcing all access through atomic APIs. `Int64`/`Uint64` are automatically 64-bit aligned even on 32-bit systems.

## context

### New Functions (1.21)

```go
// Context that does not inherit cancellation
ctx := context.WithoutCancel(parent)

// Cancellation with a cause
ctx, cancel := context.WithTimeoutCause(parent, 5*time.Second, errors.New("timeout reason"))
defer cancel()

// Retrieve the cancellation cause
err := context.Cause(ctx)  // returns the set cause rather than ctx.Err()

// Callback after context cancellation
stop := context.AfterFunc(ctx, func() { cleanup() })
defer stop()
```

## errors

### errors.Join (1.20)

Combines multiple errors into one:

```go
err := errors.Join(err1, err2, err3)
// errors.Is/As recursively checks all sub-errors of a joined error
```

## Iterators

### iter (1.23)

Base type definitions for iterators, paired with range-over-func:

```go
import "iter"

// Standard iterator types
iter.Seq[V] func(func(V) bool)            // single-value sequence
iter.Seq2[K, V] func(func(K, V) bool)     // two-value sequence (key/value)
```

Custom container types should return `iter.Seq`/`iter.Seq2` as iteration methods.

## Value Canonicalization / Weak References

### unique (1.23)

Value canonicalization (interning / hash-consing) for deduplication to save memory:

```go
import "unique"

h := unique.Make("large string that repeats")
// Equal values return equal Handles; comparison reduces to pointer comparison
h2 := unique.Make("large string that repeats")
h == h2  // true
s := h.Value()  // retrieve the original value
```

### weak (1.24)

Weak pointers for building memory-efficient caches/maps. Used with `runtime.AddCleanup` and `maphash.Comparable`:

```go
import "weak"

var p weak.Pointer[T]
p = weak.Make(&T{...})           // create weak pointer
v := p.Value()                   // retrieve strong pointer, may be nil (GC'd)
```

## Filesystem

### os.Root Directory Sandbox (1.24)

Confines file operations to a specific directory, preventing path traversal (including symlink escape):

```go
root, err := os.OpenRoot("./data")
f, err := root.Open("file.txt")          // cannot access ../etc/passwd
err = root.Mkdir("subdir", 0755)
// Methods mirror the os package: Create, Stat, Open, Mkdir, etc.
```

**Prefer `os.OpenRoot` when handling user-supplied file paths.**

## Testing

### testing.B.Loop (1.24)

A safer, more efficient benchmark loop:

```go
func BenchmarkX(b *testing.B) {
    data := setup()               // runs once
    for b.Loop() {                // replaces for i := 0; i < b.N; i++
        process(data)
    }
}
```

Advantages: setup/cleanup runs only once per count; the compiler cannot optimize away function calls in the loop body.

### testing/synctest (1.25 stable; 1.24 experimental)

Tests concurrent code: runs goroutines in an isolated "bubble" with virtualized time that advances instantly when all goroutines block.

```go
import "testing/synctest"

func TestTimeout(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        done := make(chan struct{})
        go func() {
            time.Sleep(time.Hour)       // virtual time, completes instantly
            close(done)
        }()
        synctest.Wait()                 // wait until all goroutines block
        select {
        case <-done:
        default:
            t.Fatal("should have completed")
        }
    })
}
```

## Cleanup

### runtime.AddCleanup (1.24)

A more flexible, efficient, and less error-prone cleanup mechanism than `SetFinalizer`. **Prefer it in new code.**

```go
// 1.24+
runtime.AddCleanup(obj, func(f *os.File) { f.Close() }, file)
// Multiple cleanups may be attached to one object
// May attach to interior pointers
// Does not leak on reference cycles
// Does not delay object reclamation
```

## Crypto (new in 1.24-1.27)

| Package | Version | Description |
|---------|---------|-------------|
| `crypto/mlkem` | 1.24 | Post-quantum key encapsulation (ML-KEM-768/1024, FIPS 203) |
| `crypto/hkdf` | 1.24 | HMAC-based key derivation (RFC 5869) |
| `crypto/pbkdf2` | 1.24 | Password-based key derivation (RFC 8018) |
| `crypto/sha3` | 1.24 | SHA-3 / SHAKE / cSHAKE (FIPS 202) |
| `crypto/hpke` | 1.26 | Hybrid public key encryption (RFC 9180), incl. post-quantum hybrid KEMs |
| `crypto/mldsa` | 1.27 | Post-quantum signatures (ML-DSA, FIPS 204) |

FIPS 140-3 compliance mechanisms available from 1.24 (`GOFIPS140` env var selects module version).

From 1.26, the `rand` parameter in several crypto packages is ignored (always uses a secure source); for deterministic testing use `testing/cryptotest.SetGlobalRandom`.

## JSON

### encoding/json/v2 + jsontext (1.27, stable)

`encoding/json/v2` is a major revision of `encoding/json` with better performance (decoding significantly faster) and stricter defaults (rejects invalid UTF-8, rejects duplicate names):

```go
import "encoding/json/v2"

b, err := jsonv2.Marshal(data)               // accepts variadic Options
err = jsonv2.Unmarshal(b, &data)
err = jsonv2.MarshalEncode(w, data, opts...)
```

`encoding/json/jsontext` provides lower-level JSON syntax processing (token stream, state-machine validation).

From 1.27, v1 `encoding/json` is also backed by the v2 implementation; behavior is preserved but error text may differ. v1 remains supported; migration is not required.

## Other Common Package Changes

### bytes.Buffer (1.21 new methods; 1.26 new method)

```go
// 1.21+: append directly to the buffer, avoiding extra allocation
b := &bytes.Buffer{}
b.WriteString("hello")
buf := b.AvailableBuffer()   // returns the writable underlying slice
buf = append(buf, " world"...)
// b.Available() returns remaining writable bytes

// 1.26+
peek, _ := b.Peek(5)         // peek n bytes without advancing the read position
```

### time.Timer / time.Ticker (1.23, behavior change)

- Unreferenced Timers/Tickers become GC-eligible (even if Stop was not called)
- Timer/Ticker channels are now unbuffered (`cap`/`len` return 0)
- Reset/Stop guarantees no stale values afterward

**Impact**: code that checks `len(timer.C)` to test for a pending value breaks; use a non-blocking receive `select { case <-timer.C: default: }` instead.

### os/exec (1.19, security change)

`exec.Command` and `exec.LookPath` no longer find programs in the current directory via PATH search (security hardening). To run a program in the current directory, use `./prog` explicitly.

### uuid (1.27, new package)

The new `uuid` package generates and parses UUIDs (RFC 9562).

### structs (1.23, new package)

`structs.HostLayout` marks a struct's layout as conforming to host platform expectations (for structs passed to C APIs).
