# Version Migration, Behavior Changes & Deprecations (Go 1.19-1.27)

Reference for upgrading Go versions or reviewing old code. Grouped by "Behavior changes", "Deprecations", "GODEBUG timeline", and "Platform requirements", with versions noted.

## Behavior Changes (may affect existing code)

These changes preserve the Go 1 compatibility promise but may still affect programs depending on old behavior. Investigate before upgrading.

### for-loop variable semantics (1.22) - widest impact

Starting in 1.22, `for` loop variables are created anew each iteration.

```go
// 1.22+: each iteration's v is an independent variable
for _, v := range items {
    go func() { use(v) }()
}
```

**Impact**: code relying on the old semantics (shared variable) will behave differently. Most code is fixing a latent bug, but rare code depending on sharing is affected. `go vet` reports such references under 1.21 semantics; 1.22+ no longer reports them (they are now safe).

### Timer / Ticker behavior change (1.23)

From 1.23 (when go.mod declares `go 1.23.0` or later):

- Unreferenced `Timer`/`Ticker` become GC-eligible (even without Stop)
- `Timer.C` / `Ticker.C` channels become **unbuffered** (`cap`/`len` return 0 instead of 1)
- `Reset`/`Stop` guarantee no stale values are sent afterward

**Affected code**: using `len(timer.C) > 0` to check for a pending value breaks. Use a non-blocking receive `select { case <-timer.C: default: }` instead.

Temporarily restore old behavior with `GODEBUG=asynctimerchan=1` (auto-enabled when go.mod declares an older version). **1.27 permanently removes this GODEBUG; no fallback.**

### nil-pointer delayed-check fix (1.25)

Go 1.21-1.24 had a compiler bug: it incorrectly delayed nil-pointer checks until after the error check, so code that used a return value before checking error would侥幸 not panic. The 1.25 fix makes such code correctly panic.

```go
// This code侥幸 didn't panic in 1.21-1.24; panics from 1.25
f, err := os.Open("nonexistent")
name := f.Name()           // f is nil when err is non-nil -> panic
if err != nil { return }
```

**Fix**: move the error check before using the return value.

### Container-aware GOMAXPROCS (1.25)

On Linux, `GOMAXPROCS` now considers cgroup CPU bandwidth limits (e.g. Kubernetes CPU limit) by default. On all OSes it periodically updates available CPU count. Manually setting `GOMAXPROCS` env var or calling `runtime.GOMAXPROCS` disables this.

Disable with `GODEBUG=containermaxprocs=0` / `updatemaxprocs=0` respectively.

### panic(nil) is no longer nil (1.21)

`panic(nil)` or an untyped nil now produces `*runtime.PanicNilError`, ensuring `recover()` returns non-nil. `GODEBUG=panicnil=1` restores old behavior (auto-enabled when go.mod declares `go 1.20` or earlier).

### cgo disabled by default (1.20)

On systems without a C toolchain, `CGO_ENABLED` defaults to `0`. Affects `net`, `os/user`, `plugin` packages (already pure-Go on macOS). Set `CGO_ENABLED=1` explicitly to override.

### os/exec no longer searches current directory (1.19)

`exec.Command("prog")` and `exec.LookPath("prog")` no longer find programs in the current directory via PATH. Use `./prog` to run a program in the current directory.

### Package initialization order specified (1.21)

Initialized by import path order. Code depending on the previously-undefined order may behave differently.

### go command behavior changes

| Change | Version | Description |
|--------|---------|-------------|
| `go build -i` removed | 1.20 | `-i` flag deprecated since 1.16, no longer accepted |
| `go get` in GOPATH mode | 1.22 | `go get` no longer works under `GO111MODULE=off` |
| `go mod init` no external tool config | 1.22 | No longer imports from `Gopkg.lock`, etc. |
| `cmd/doc` removed | 1.26 | Use `go doc` |
| `go mod init` lower default | 1.26 | 1.N.X toolchain creates `go 1.(N-1).0` |
| `GOROOT_FINAL` ineffective | 1.23 | Use a symlink instead |
| `//go:linkname` restricted | 1.23 | New references to internal stdlib symbols are disallowed |
| `go test -json` build output | 1.24 | Build output also JSON (revert with `gotestjsonbuildtext=1`) |

## Deprecations & Replacements

### Standard library deprecations

| Deprecated | Version | Replacement |
|------------|---------|-------------|
| `crypto/elliptic` Curve methods, `GenerateKey`, `Marshal`, `Unmarshal` | 1.21 | `crypto/ecdh` (for ECDH) or third-party `filippo.io/nistec` |
| `crypto/rsa` `GenerateMultiPrimeKey`, `PrecomputedValues.CRTValues` | 1.21 | No longer used |
| `math/rand` `Read` | 1.22 | `crypto/rand.Read` (cryptographic use) or `math/rand/v2` |
| `runtime.SetFinalizer` | 1.24 (replacement recommended) | `runtime.AddCleanup` (more flexible, less leak-prone) |
| PKCS#1 v1.5 encryption padding (`EncryptPKCS1v15`/`DecryptPKCS1v15`) | 1.26 | OAEP (`rsa.EncryptOAEP`) |
| `crypto/ecdsa` `big.Int` fields of `PublicKey`/`PrivateKey` | 1.26 | No longer accessed directly |
| Crypto packages' `rand` parameter | 1.26 | Ignored; always uses secure source. For testing use `testing/cryptotest.SetGlobalRandom` |
| `fmtappendf` analyzer | 1.27 (go fix) | Removed |
| `waitgroup` analyzer renamed | 1.27 (go fix) | `waitgroupgo` |

### GODEBUG setting removal timeline

| GODEBUG | Introduced | Removed/permanentized | Description |
|---------|-----------|----------------------|-------------|
| `panicnil=1` | 1.21 | Retained (auto-enabled for old go.mod) | Restores panic(nil) old behavior |
| `asynctimerchan=1` | 1.23 | **1.27 permanently removed** | Time channels are always unbuffered |
| `tlsrsakex` | 1.22 | **Removed from 1.27** | Legacy RSA key exchange no longer default |
| `tls10server` | 1.22 | **Removed from 1.27** | TLS minimum version becomes 1.2 |
| `tls3des` | 1.22 | **Removed from 1.27** | Default cipher suites exclude 3DES |
| `tlsunsafeekm` | 1.22 | **Removed from 1.27** | ExportKeyingMaterial requires TLS 1.3 or EMS |
| `x509keypairleaf` | 1.22 | **Removed from 1.27** | X509KeyPair always populates Leaf |
| `cryptocustomrand=1` | 1.26 | Retained | Restores custom random source for crypto packages |

**1.27 upgrade focus**: multiple TLS-related GODEBUGs are removed; programs depending on old TLS behavior are affected.

## Experimental-to-Stable Timeline

| Feature | Experimental | Stable |
|---------|-------------|--------|
| range-over-func (iterators) | 1.22 (`GOEXPERIMENT=rangefunc`) | 1.23 |
| Generic type aliases | 1.23 (`GOEXPERIMENT=aliastypeparams`) | 1.24 |
| testing/synctest | 1.24 (`GOEXPERIMENT=synctest`) | 1.25 |
| Green Tea GC | 1.25 (`GOEXPERIMENT=greenteagc`) | 1.26 default |
| goroutine leak profile | 1.26 (`GOEXPERIMENT=goroutineleakprofile`) | 1.27 |
| encoding/json/v2 | 1.25 (`GOEXPERIMENT=jsonv2`) | 1.27 |

Still experimental in 1.27:
- `GOEXPERIMENT=simd` - portable SIMD package (`simd`); architecture-specific variant `simd/archsimd` (1.26)
- `GOEXPERIMENT=runtimesecret` - secure erasure of sensitive data (`runtime/secret`, 1.26)

## Bootstrap Requirements

| Go version | Bootstrap minimum |
|------------|-------------------|
| 1.22 | Go 1.20 final point release |
| 1.24 | Go 1.22.6+ |
| 1.26 | Go 1.24.6+ |
| 1.28 (expected) | Some release of Go 1.26 |

## Platform Support Changes

| Change | Version |
|--------|---------|
| Last support for Windows 7/8, Server 2008/2012 | 1.20 (1.21+ requires Windows 10+) |
| Last support for macOS 10.13/10.14 | 1.20 (1.21+ requires 10.15+) |
| macOS minimum 13.0 | 1.27 |
| LoongArch 64-bit (loong64) | 1.19 |
| RISC-V register ABI | 1.19 |

## Upgrade Checklist

When upgrading to a new Go version, investigate in this order:

1. **Run `go fix ./...`** (1.26+) - auto-migrates to modern APIs.
2. **Run `go vet ./...`** - check for issues found by new analyzers.
3. **Run tests** - watch for behavior changes (loop variables, timer channels, nil-pointer checks).
4. **Check GODEBUG dependencies** - before upgrading to 1.27, confirm you do not depend on the TLS GODEBUGs being removed.
5. **Check the go.mod go line** - determines which behavior changes take effect (the GODEBUG compatibility mechanism is controlled by the go line).
6. **Confirm platform requirements** - macOS minimum version, Windows version.
7. **Check cgo** - in environments without a C toolchain, confirm the `CGO_ENABLED` default.
8. **Audit `//go:linkname`** - 1.23+ disallows new references to internal stdlib symbols.
