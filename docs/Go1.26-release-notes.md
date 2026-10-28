# Go 1.26 Release Notes

## Introduction to Go 1.26

The latest Go release, version 1.26, arrives in [February 2026](/doc/devel/release#go1.26.0), six months after [Go 1.25](/doc/go1.25).
Most of its changes are in the implementation of the toolchain, runtime, and libraries.
As always, the release maintains the Go 1 promise of compatibility.
We expect almost all Go programs to continue to compile and run as before.

## Changes to the language

The built-in `new` function, which creates a new variable, now allows
its operand to be an expression, specifying the initial value of the
variable.

This feature is particularly useful when working with serialization
packages such as `encoding/json` or protocol buffers that use a
pointer to represent an optional value, as it enables an optional
field to be populated in a simple expression, for example:

`import "encoding/json"

type Person struct {
Name string `json:"name"`
Age *int `json:"age"` // age if known; nil otherwise
}

func personJSON(name string, born time.Time) ([]byte, error) {
return json.Marshal(Person{
Name: name,
Age: new(yearsSince(born)),
})
}

func yearsSince(t time.Time) int {
return int(time.Since(t).Hours() / (365.25 * 24)) // approximately
}
`

The restriction that a generic type may not refer to itself in its type parameter list
has been lifted.
It is now possible to specify type constraints that refer to the generic type being
constrained.
For instance, a generic type `Adder` may require that it be instantiated with a
type that is like itself:

`type Adder[A Adder[A]] interface {
Add(A) A
}

func algo[A Adder[A]](x, y A) A {
return x.Add(y)
}
`
Previously, the self-reference to `Adder` on the first line was not allowed.
Besides making type constraints more powerful, this change also simplifies the spec
rules for type parameters ever so slightly.

## Tools

### Go command

The venerable `go fix` command has been completely revamped and is now the home
of Go&rsquo;s modernizers*. It provides a dependable, push-button way to update Go
code bases to the latest idioms and core library APIs. The initial suite of
modernizers includes dozens of fixers to make use of modern features of the Go
language and library, as well a source-level inliner that allows users to
automate their own API migrations using
[`//go:fix inline` directives](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/inline#hdr-Analyzer_inline).
These fixers should not change the behavior of your program, so if you encounter
any issues with a fix performed by `go fix`, please [report it](/issue/new).

The rewritten `go fix` command builds atop the exact same [Go analysis
framework](https://pkg.go.dev/golang.org/x/tools/go/analysis) as `go vet`.
This means the same analyzers that provide diagnostics in `go vet`
can be used to suggest and apply fixes in `go fix`.
The `go fix` command&rsquo;s historical fixers, all of which were obsolete,
have been removed.

Two upcoming Go blog posts will go into more detail on modernizers, the inliner,
and how to get the most out of `go fix`.

`go mod init` now defaults to a lower `go` version in new `go.mod` files.
Running `go mod init`
using a toolchain of version `1.N.X` will create a `go.mod` file
specifying the Go version `go 1.(N-1).0`. Pre-release versions of `1.N` will
create `go.mod` files specifying `go 1.(N-2).0`. For example, the Go 1.26
release candidates will create `go.mod` files with `go 1.24.0`, and Go 1.26
and its minor releases will create `go.mod` files with `go 1.25.0`. This is intended
to encourage the creation of modules that are compatible with currently supported
versions of Go. For additional control over the `go` version in new modules,
`go mod init` can be followed up with `go get go@version`.

`cmd/doc`, and `go tool doc` have been deleted. `go doc` can be used as
a replacement for `go tool doc`: it takes the same flags and arguments and
has the same behavior.

### Pprof

The `pprof` tool web UI, enabled with the `-http` flag, now defaults to the flame graph view.
The previous graph view is available in the &ldquo;View -> Graph&rdquo; menu, or via `/ui/graph`.

## Runtime

### New garbage collector

The Green Tea garbage collector, previously available as an experiment in
Go 1.25, is now enabled by default after incorporating feedback.

This garbage collector&rsquo;s design improves the performance of marking and
scanning small objects through better locality and CPU scalability.
Benchmark results vary, but we expect somewhere between a 10–40% reduction
in garbage collection overhead in real-world programs that heavily use the
garbage collector.
Further improvements, on the order of 10% in garbage collection overhead,
are expected when running on newer amd64-based CPU platforms (Intel Ice
Lake or AMD Zen 4 and newer), as the garbage collector now leverages
vector instructions for scanning small objects when possible.

The new garbage collector may be disabled by setting
`GOEXPERIMENT=nogreenteagc` at build time.
This opt-out setting is expected to be removed in Go 1.27.
If you disable the new garbage collector for any reason related to its
performance or behavior, please [file an issue](/issue/new).

### Faster cgo calls

The baseline runtime overhead of cgo calls has been reduced by ~30%.

### Heap base address randomization

On 64-bit platforms, the runtime now randomizes the heap base address
at startup.
This is a security enhancement that makes it harder for attackers to
predict memory addresses and exploit vulnerabilities when using cgo.
This feature may be disabled by setting
`GOEXPERIMENT=norandomizedheapbase64` at build time.
This opt-out setting is expected to be removed in a future Go release.

### Experimental goroutine leak profile

A new profile type that reports leaked goroutines is now available as an
experiment.
The new profile type, named `goroutineleak` in the
[`runtime/pprof`](/pkg/runtime/pprof) package, may be enabled by setting
`GOEXPERIMENT=goroutineleakprofile` at build time.
Enabling the experiment also makes the profile available as a
[`net/http/pprof`](/pkg/net/http/pprof) endpoint,
`/debug/pprof/goroutineleak`.

A *leaked* goroutine is a goroutine blocked on some concurrency primitive
(channels, [`sync.Mutex`](/pkg/sync#Mutex), [`sync.Cond`](/pkg/sync#Cond), etc) that
cannot possibly become unblocked.
The runtime detects leaked goroutines using the garbage collector: if a
goroutine G is blocked on concurrency primitive P, and P is unreachable from
any runnable goroutine or any goroutine that *those* could unblock, then P
cannot be unblocked, so goroutine G can never wake up.
While it is impossible to detect permanently blocked goroutines in all cases,
this approach detects a large class of such leaks.

The following example showcases a real-world goroutine leak that
can be revealed by the new profile:

`type result struct {
res workResult
err error
}

func processWorkItems(ws []workItem) ([]workResult, error) {
// Process work items in parallel, aggregating results in ch.
ch := make(chan result)
for _, w := range ws {
go func() {
res, err := processWorkItem(w)
ch <- result{res, err}
}()
}

// Collect the results from ch, or return an error if one is found.
var results []workResult
for range len(ws) {
r := <-ch
if r.err != nil {
// This early return may cause goroutine leaks.
return nil, r.err
}
results = append(results, r.res)
}
return results, nil
}
`
Because `ch` is unbuffered, if `processWorkItems` returns early due to
an error, all remaining `processWorkItem` goroutines will leak.
Soon after this happens, `ch` will become unreachable to all other
goroutines not involved in the leak, allowing the runtime to detect
and report the leaked goroutines.

Because this technique builds on reachability, the runtime may fail to identify
leaks caused by blocking on concurrency primitives reachable through global
variables or the local variables of runnable goroutines.

Special thanks to Vlad Saioc at Uber for contributing this work.
The underlying theory is presented in detail in [a publication by
Saioc et al](https://dl.acm.org/doi/pdf/10.1145/3676641.3715990).

The implementation is production-ready, and is only considered an
experiment for the purposes of collecting feedback on the API,
specifically the choice to make it a new profile.
The feature is also designed to not incur any additional run-time
overhead unless it is actively in-use.

We encourage users to try out the new feature in [the Go
playground](/play/p/3C71z4Dpav-?v=gotip),
in tests, in continuous integration, and in production.
We welcome additional feedback on the [proposal
issue](/issue/74609).

We aim to enable goroutine leak profiles by default in Go 1.27.

## Compiler

The compiler can now allocate the backing store for slices on the stack in more
situations, which improves performance. If this change is causing trouble, the
[bisect tool](https://pkg.go.dev/golang.org/x/tools/cmd/bisect) can be used to
find the allocation causing trouble using the `-compile=variablemake` flag. All
such new stack allocations can also be turned off using
`-gcflags=all=-d=variablemakehash=n`.
If you encounter issues with this optimization, please [file an issue](/issue/new).

## Linker

On 64-bit ARM-based Windows (the `windows/arm64` port), the linker now supports internal
linking mode of cgo programs, which can be requested with the
`-ldflags=-linkmode=internal` flag.

There are several minor changes to executable files. These changes do
not affect running Go programs. They may affect programs that analyze
Go executables, and they may affect people who use external linking
mode with custom linker scripts.

The `moduledata` structure is now in its own section, named
`.go.module`.

- The `moduledata` `cutab` field, which is a slice, now has the
correct length; previously the length was four times too large.

- The `pcHeader` found at the start of the `.gopclntab` section no
longer records the start of the text section. That field is now
always zero.

- That `pcHeader` change was made so that the `.gopclntab` section
no longer contains any relocations. On platforms that support
relro, the section has moved from the relro segment to the rodata
segment.

- The funcdata symbols and the findfunctab have moved from the
`.rodata` section to the `.gopclntab` section.

- The `.gosymtab` section has been removed. It was previously always
present but empty.

- When using internal linking, ELF sections now appear in the
section header list sorted by address. The previous order was
somewhat unpredictable.

The references to section names here use the ELF names as seen on
Linux and other systems. The Mach-O names as seen on Darwin start with
a double underscore and do not contain any dots.

## Bootstrap

As mentioned in the [Go 1.24 release notes](/doc/go1.24#bootstrap), Go 1.26 now requires
Go 1.24.6 or later for bootstrap.
We expect that Go 1.28 will require a minor release of Go 1.26 or later for bootstrap.

## Standard library

### New crypto/hpke package

The new [`crypto/hpke`](/pkg/crypto/hpke) package implements Hybrid Public Key Encryption
(HPKE) as specified in [RFC 9180](https://rfc-editor.org/rfc/rfc9180.html), including support for post-quantum
hybrid KEMs.

### New experimental simd/archsimd package

Go 1.26 introduces a new experimental [`simd/archsimd`](/pkg/simd/archsimd/)
package, which can be enabled by setting the environment variable
`GOEXPERIMENT=simd` at build time.
This package provides access to architecture-specific SIMD operations.
It is currently available on the amd64 architecture and supports
128-bit, 256-bit, and 512-bit vector types, such as
[`Int8x16`](/pkg/simd/archsimd#Int8x16) and
[`Float64x8`](/pkg/simd/archsimd#Float64x8), with operations such as
[`Int8x16.Add`](/pkg/simd/archsimd#Int8x16.Add).
The API is not yet considered stable.

We intend to provide support for other architectures in future versions, but the
API intentionally architecture-specific and thus non-portable.
In addition, we plan to develop a high-level portable SIMD
package in the future.

See the [package documentation](/pkg/simd/archsimd) and the [proposal issue](/issue/73787) for more details.

### New experimental runtime/secret package

The new [`runtime/secret`](/pkg/runtime/secret) package is available as an experiment,
which can be enabled by setting the environment variable
`GOEXPERIMENT=runtimesecret` at build time.
It provides a facility for securely erasing temporaries used in
code that manipulates secret information—typically cryptographic in nature—such
as registers, stack, new heap allocations.
This package is intended to make it easier to ensure [forward
secrecy](https://en.wikipedia.org/wiki/Forward_secrecy).
It currently supports the amd64 and arm64 architectures on Linux.

### Minor changes to the library

#### [`bytes`](/pkg/bytes/)

The new [`Buffer.Peek`](/pkg/bytes#Buffer.Peek) method returns the next n bytes from the buffer without
advancing it.

#### [`crypto`](/pkg/crypto/)

The new [`Encapsulator`](/pkg/crypto#Encapsulator) and [`Decapsulator`](/pkg/crypto#Decapsulator) interfaces allow accepting abstract
KEM encapsulation or decapsulation keys.

#### [`crypto/dsa`](/pkg/crypto/dsa/)

The random parameter to [`GenerateKey`](/pkg/crypto/dsa#GenerateKey) is now ignored.
Instead, it now always uses a secure source of cryptographically random bytes.
For deterministic testing, use the new [`testing/cryptotest.SetGlobalRandom`](/pkg/testing/cryptotest#SetGlobalRandom) function.
The new GODEBUG setting `cryptocustomrand=1` temporarily restores the old behavior.

#### [`crypto/ecdh`](/pkg/crypto/ecdh/)

The random parameter to [`Curve.GenerateKey`](/pkg/crypto/ecdh#Curve.GenerateKey) is now ignored.
Instead, it now always uses a secure source of cryptographically random bytes.
For deterministic testing, use the new [`testing/cryptotest.SetGlobalRandom`](/pkg/testing/cryptotest#SetGlobalRandom) function.
The new GODEBUG setting `cryptocustomrand=1` temporarily restores the old behavior.

The new [`KeyExchanger`](/pkg/crypto/ecdh#KeyExchanger) interface, implemented by [`PrivateKey`](/pkg/crypto/ecdh#PrivateKey), makes it possible
to accept abstract ECDH private keys, e.g. those implemented in hardware.

#### [`crypto/ecdsa`](/pkg/crypto/ecdsa/)

The `big.Int` fields of [`PublicKey`](/pkg/crypto/ecdsa#PublicKey) and [`PrivateKey`](/pkg/crypto/ecdsa#PrivateKey) are now deprecated.

The random parameter to [`GenerateKey`](/pkg/crypto/ecdsa#GenerateKey), [`SignASN1`](/pkg/crypto/ecdsa#SignASN1), [`Sign`](/pkg/crypto/ecdsa#Sign), and [`PrivateKey.Sign`](/pkg/crypto/ecdsa#PrivateKey.Sign) is now ignored.
Instead, they now always use a secure source of cryptographically random bytes.
For deterministic testing, use the new [`testing/cryptotest.SetGlobalRandom`](/pkg/testing/cryptotest#SetGlobalRandom) function.
The new GODEBUG setting `cryptocustomrand=1` temporarily restores the old behavior.

#### [`crypto/ed25519`](/pkg/crypto/ed25519/)

If the random parameter to [`GenerateKey`](/pkg/crypto/ed25519#GenerateKey) is nil, GenerateKey now always uses a
secure source of cryptographically random bytes, instead of [`crypto/rand.Reader`](/pkg/crypto/rand#Reader)
(which could have been overridden). The new GODEBUG setting `cryptocustomrand=1`
temporarily restores the old behavior.

#### [`crypto/fips140`](/pkg/crypto/fips140/)

[FIPS 140-3 Go Cryptographic Module](/doc/security/fips140) v1.26.0 includes changes made to the `crypto/internal/fips140/...` packages up to this release, and can now be selected with GOFIPS140.

The new [`WithoutEnforcement`](/pkg/crypto/fips140#WithoutEnforcement) and [`Enforced`](/pkg/crypto/fips140#Enforced) functions now allow running
in `GODEBUG=fips140=only` mode while selectively disabling the strict FIPS 140-3 checks.

[`Version`](/pkg/crypto/fips140#Version) returns the resolved FIPS 140-3 Go Cryptographic Module version when building against a frozen module with GOFIPS140.

#### [`crypto/mlkem`](/pkg/crypto/mlkem/)

The new [`DecapsulationKey768.Encapsulator`](/pkg/crypto/mlkem#DecapsulationKey768.Encapsulator) and
[`DecapsulationKey1024.Encapsulator`](/pkg/crypto/mlkem#DecapsulationKey1024.Encapsulator) methods implement the new
[`crypto.Decapsulator`](/pkg/crypto#Decapsulator) interface.

Encapsulation and decapsultion operations are now approximately 18% faster.

#### [`crypto/mlkem/mlkemtest`](/pkg/crypto/mlkem/mlkemtest/)

The new [`crypto/mlkem/mlkemtest`](/pkg/crypto/mlkem/mlkemtest) package exposes the [`Encapsulate768`](/pkg/crypto/mlkem/mlkemtest#Encapsulate768) and
[`Encapsulate1024`](/pkg/crypto/mlkem/mlkemtest#Encapsulate1024) functions which implement derandomized ML-KEM encapsulation,
for use with known-answer tests.

#### [`crypto/rand`](/pkg/crypto/rand/)

The random parameter to [`Prime`](/pkg/crypto/rand#Prime) is now ignored.
Instead, it now always uses a secure source of cryptographically random bytes.
For deterministic testing, use the new [`testing/cryptotest.SetGlobalRandom`](/pkg/testing/cryptotest#SetGlobalRandom) function.
The new GODEBUG setting `cryptocustomrand=1` temporarily restores the old behavior.

#### [`crypto/rsa`](/pkg/crypto/rsa/)

The new [`EncryptOAEPWithOptions`](/pkg/crypto/rsa#EncryptOAEPWithOptions) function allows specifying different hash
functions for OAEP padding and MGF1 mask generation.

The random parameter to [`GenerateKey`](/pkg/crypto/rsa#GenerateKey), [`GenerateMultiPrimeKey`](/pkg/crypto/rsa#GenerateMultiPrimeKey), and [`EncryptPKCS1v15`](/pkg/crypto/rsa#EncryptPKCS1v15) is now ignored.
Instead, they now always use a secure source of cryptographically random bytes.
For deterministic testing, use the new [`testing/cryptotest.SetGlobalRandom`](/pkg/testing/cryptotest#SetGlobalRandom) function.
The new GODEBUG setting `cryptocustomrand=1` temporarily restores the old behavior.

If [`PrivateKey`](/pkg/crypto/rsa#PrivateKey) fields are modified after calling [`PrivateKey.Precompute`](/pkg/crypto/rsa#PrivateKey.Precompute),
[`PrivateKey.Validate`](/pkg/crypto/rsa#PrivateKey.Validate) now fails.

[`PrivateKey.D`](/pkg/crypto/rsa#PrivateKey.D) is now checked for consistency with precomputed values, even if
it is not used.

Unsafe PKCS #1 v1.5 encryption padding (implemented by [`EncryptPKCS1v15`](/pkg/crypto/rsa#EncryptPKCS1v15),
[`DecryptPKCS1v15`](/pkg/crypto/rsa#DecryptPKCS1v15), and [`DecryptPKCS1v15SessionKey`](/pkg/crypto/rsa#DecryptPKCS1v15SessionKey)) is now deprecated.

#### [`crypto/sha3`](/pkg/crypto/sha3/)

The zero value of [`SHA3`](/pkg/crypto/sha3#SHA3) is now a usable SHA3-256 instance, and the zero value of [`SHAKE`](/pkg/crypto/sha3#SHAKE) is now a usable SHAKE256 instance.

#### [`crypto/subtle`](/pkg/crypto/subtle)

The [`WithDataIndependentTiming`](/pkg/crypto/subtle#WithDataIndependentTiming)
function no longer locks the calling goroutine to the OS thread while executing
the passed function. Additionally, any goroutines which are spawned during the
execution of the passed function and their descendants now inherit the properties of
`WithDataIndependentTiming` for their lifetime. This change also affects cgo in
the following ways:

- Any C code called via cgo from within the function passed to
`WithDataIndependentTiming`, or from a goroutine spawned by the function passed
to `WithDataIndependentTiming` and its descendants, will also have data
independent timing enabled for the duration of the call. If the C code
disables data independent timing, it will be re-enabled on return to Go.

- If C code called via cgo, from the function passed to
`WithDataIndependentTiming` or elsewhere, enables or disables data independent
timing then calling into Go will preserve that state for the duration of the
call.

#### [`crypto/tls`](/pkg/crypto/tls/)

The hybrid [`SecP256r1MLKEM768`](/pkg/crypto/tls#SecP256r1MLKEM768) and [`SecP384r1MLKEM1024`](/pkg/crypto/tls#SecP384r1MLKEM1024) post-quantum key
exchanges are now enabled by default. They can be disabled by setting
[`Config.CurvePreferences`](/pkg/crypto/tls#Config.CurvePreferences) or with the `tlssecpmlkem=0` GODEBUG setting.

The new [`ClientHelloInfo.HelloRetryRequest`](/pkg/crypto/tls#ClientHelloInfo.HelloRetryRequest) field indicates if the ClientHello
was sent in response to a HelloRetryRequest message. The new
[`ConnectionState.HelloRetryRequest`](/pkg/crypto/tls#ConnectionState.HelloRetryRequest) field indicates if the server
sent a HelloRetryRequest, or if the client received a HelloRetryRequest,
depending on connection role.

The [`QUICConn`](/pkg/crypto/tls#QUICConn) type used by QUIC implementations includes a new event
for reporting TLS handshake errors.

If [`Certificate.PrivateKey`](/pkg/crypto/tls#Certificate.PrivateKey) implements [`crypto.MessageSigner`](/pkg/crypto#MessageSigner), its SignMessage
method is used instead of Sign in TLS 1.2 and later.

The following GODEBUG settings introduced in [Go 1.22](/doc/godebug#go-122)
and [Go 1.23](/doc/godebug#go-123) will be removed in the next major Go release.
Starting in Go 1.27, the new behavior will apply regardless of GODEBUG setting or go.mod language version.

- `tlsunsafeekm`: [`ConnectionState.ExportKeyingMaterial`](/pkg/crypto/tls#ConnectionState.ExportKeyingMaterial) will require TLS 1.3 or Extended Master Secret.

- `tlsrsakex`: legacy RSA-only key exchanges without ECDH won&rsquo;t be enabled by default.

- `tls10server`: the default minimum TLS version for both clients and servers will be TLS 1.2.

- `tls3des`: the default cipher suites will not include 3DES.

- `x509keypairleaf`: [`X509KeyPair`](/pkg/crypto/tls#X509KeyPair) and [`LoadX509KeyPair`](/pkg/crypto/tls#LoadX509KeyPair) will always populate the [`Certificate.Leaf`](/pkg/crypto/tls#Certificate.Leaf) field.

#### [`crypto/x509`](/pkg/crypto/x509/)

The [`ExtKeyUsage`](/pkg/crypto/x509#ExtKeyUsage) and [`KeyUsage`](/pkg/crypto/x509#KeyUsage) types now have `String` methods that return the
corresponding OID names as defined in RFC 5280 and other registries.

The [`ExtKeyUsage`](/pkg/crypto/x509#ExtKeyUsage) type now has an `OID` method that returns the corresponding OID for the EKU.

The new [`OIDFromASN1OID`](/pkg/crypto/x509#OIDFromASN1OID) function allows converting an [`encoding/asn1.ObjectIdentifier`](/pkg/encoding/asn1#ObjectIdentifier) into
an [`OID`](/pkg/crypto/x509#OID).

#### [`debug/elf`](/pkg/debug/elf/)

Additional `R_LARCH_*` constants from [LoongArch ELF psABI v20250521](https://github.com/loongson/la-abi-specs/blob/v2.40/laelf.adoc)
(global version v2.40) are defined for use with LoongArch systems.

#### [`errors`](/pkg/errors/)

The new [`AsType`](/pkg/errors#AsType) function is a generic version of [`As`](/pkg/errors#As). It is type-safe, faster,
and, in most cases, easier to use.

#### [`fmt`](/pkg/fmt/)

For unformatted strings, `fmt.Errorf("x")` now allocates less and generally matches
the allocations for `errors.New("x")`.

#### [`go/ast`](/pkg/go/ast/)

The new [`ParseDirective`](/pkg/go/ast#ParseDirective) function parses [directive
comments](/doc/comment#Syntax), which are comments such as `//go:generate`.
Source code tools can support their own directive comments and this new API
should help them implement the conventional syntax.

The new [`BasicLit.ValueEnd`](/pkg/go/ast#BasicLit.ValueEnd) field records the precise end position of
a literal so that the [`BasicLit.End`](/pkg/go/ast#BasicLit.End) method can now always return the
correct answer. (Previously it was computed using a heuristic that was
incorrect for multi-line raw string literals in Windows source files,
due to removal of carriage returns.)

Programs that update the `ValuePos` field of `BasicLit`s produced by
the parser may need to also update or clear the `ValueEnd` field to
avoid minor differences in formatted output.

#### [`go/token`](/pkg/go/token/)

The new [`File.End`](/pkg/go/token#File.End) convenience method returns the file&rsquo;s end position.

#### [`go/types`](/pkg/go/types/)

The `gotypesalias` GODEBUG setting introduced in [Go 1.22](/doc/godebug#go-122)
will be removed in the next major Go release.
Starting in Go 1.27, the [`go/types`](/pkg/go/types) package will always produce an
[Alias type](/pkg/go/types#Alias) for the representation of [type aliases](/ref/spec#Type_declarations)
regardless of GODEBUG setting or go.mod language version.

#### [`image/jpeg`](/pkg/image/jpeg/)

The JPEG encoder and decoder have been replaced with new, faster, more accurate implementations.
Code that expects specific bit-for-bit outputs from the encoder or decoder may need to be updated.

#### [`io`](/pkg/io/)

[`ReadAll`](/pkg/io#ReadAll) now allocates less intermediate memory and returns a minimally sized
final slice. It is often about two times faster while typically allocating around half
as much total memory, with more benefit for larger inputs.

#### [`log/slog`](/pkg/log/slog/)

The [`NewMultiHandler`](/pkg/log/slog#NewMultiHandler) function creates a
[`MultiHandler`](/pkg/log/slog#MultiHandler) that invokes all the given Handlers.
Its `Enabled` method reports whether any of the handlers&rsquo; `Enabled` methods
return true.
Its `Handle`, `WithAttrs` and `WithGroup` methods call the corresponding method
on each of the enabled handlers.

#### [`net`](/pkg/net/)

The new [`Dialer`](/pkg/net/#Dialer) methods
[`DialIP`](/pkg/net/#Dialer.DialIP),
[`DialTCP`](/pkg/net/#Dialer.DialTCP),
[`DialUDP`](/pkg/net/#Dialer.DialUDP), and
[`DialUnix`](/pkg/net/#Dialer.DialUnix)
permit dialing specific network types with context values.

#### [`net/http`](/pkg/net/http/)

The new
[`HTTP2Config.StrictMaxConcurrentRequests`](/pkg/net/http#HTTP2Config.StrictMaxConcurrentRequests)
field controls whether a new connection should be opened
if an existing HTTP/2 connection has exceeded its stream limit.

The new [`Transport.NewClientConn`](/pkg/net/http#Transport.NewClientConn) method returns a client connection
to an HTTP server.
Most users should continue to use [`Transport.RoundTrip`](/pkg/net/http#Transport.RoundTrip) to make requests,
which manages a pool of connections.
`NewClientConn` is useful for users who need to implement their own connection management.

[`Client`](/pkg/net/http#Client) now uses and sets cookies scoped to URLs with the host portion matching
[`Request.Host`](/pkg/net/http#Request.Host) when available.
Previously, the connection address host was always used.
[`ServeMux`](/pkg/net/http#ServeMux) trailing slash redirects now use HTTP status 307
(Temporary Redirect) instead of 301 (Moved Permanently).

#### [`net/http/httptest`](/pkg/net/http/httptest/)

The HTTP client returned by [`Server.Client`](/pkg/net/http/httptest#Server.Client) will now redirect requests for
`example.com` and any subdomains to the server being tested.

#### [`net/http/httputil`](/pkg/net/http/httputil/)

The [`ReverseProxy.Director`](/pkg/net/http/httputil#ReverseProxy.Director) configuration field is deprecated
in favor of [`ReverseProxy.Rewrite`](/pkg/net/http/httputil#ReverseProxy.Rewrite).

A malicious client can remove headers added by a `Director` function
by designating those headers as hop-by-hop. Since there is no way to address
this problem within the scope of the `Director` API, we added a new
`Rewrite` hook in Go 1.20. `Rewrite` hooks are provided with both the
unmodified inbound request received by the proxy and the outbound request
which will be sent by the proxy.

Since the `Director` hook is fundamentally unsafe, we are now deprecating it.

#### [`net/netip`](/pkg/net/netip/)

The new [`Prefix.Compare`](/pkg/net/netip#Prefix.Compare) method compares two prefixes.

#### [`net/url`](/pkg/net/url/)

[`Parse`](/pkg/net/url#Parse) now rejects malformed URLs containing colons in the host subcomponent,
such as `http://::1/` or `http://localhost:80:80/`.
URLs containing bracketed IPv6 addresses, such as `http://[::1]/` are still accepted.
The new GODEBUG setting `urlstrictcolons=0` restores the old behavior.

#### [`os`](/pkg/os/)

The new [`Process.WithHandle`](/pkg/os#Process.WithHandle) method provides
access to an internal process handle on supported platforms (pidfd on Linux 5.4
or later, Handle on Windows).

On Windows, the [`OpenFile`](/pkg/os#OpenFile) `flag` parameter can now contain any combination of
Windows-specific file flags, such as `FILE_FLAG_OVERLAPPED` and
`FILE_FLAG_SEQUENTIAL_SCAN`, for control of file or device caching behavior,
access modes, and other special-purpose flags.

#### [`os/signal`](/pkg/os/signal/)

[`NotifyContext`](/pkg/os/signal#NotifyContext) now cancels the returned context with [`context.CancelCauseFunc`](/pkg/context#CancelCauseFunc)
and an error indicating which signal was received.

#### [`reflect`](/pkg/reflect/)

The new methods [`Type.Fields`](/pkg/reflect#Type.Fields),
[`Type.Methods`](/pkg/reflect#Type.Methods),
[`Type.Ins`](/pkg/reflect#Type.Ins)
and [`Type.Outs`](/pkg/reflect#Type.Outs)
return iterators for a type&rsquo;s fields (for a struct type), methods,
inputs and outputs parameters (for a function type), respectively.

Similarly, the new methods [`Value.Fields`](/pkg/reflect#Value.Fields)
and [`Value.Methods`](/pkg/reflect#Value.Methods) return iterators over
a value&rsquo;s fields or methods, respectively.
Each iteration yields the type information ([`StructField`](/pkg/reflect#StructField) or
[`Method`](/pkg/reflect#Method)) of a field or method,
along with the field or method [`Value`](/pkg/reflect#Value).

#### [`runtime/metrics`](/pkg/runtime/metrics/)

Several new scheduler metrics have been added, including counts of
goroutines in various states (waiting, runnable, etc.) under the
`/sched/goroutines` prefix, the number of OS threads the runtime is
aware of with `/sched/threads:threads`, and the total number of
goroutines created by the program with
`/sched/goroutines-created:goroutines`.

#### [`testing`](/pkg/testing/)

The new methods [`T.ArtifactDir`](/pkg/testing#T.ArtifactDir), [`B.ArtifactDir`](/pkg/testing#B.ArtifactDir), and [`F.ArtifactDir`](/pkg/testing#F.ArtifactDir)
return a directory in which to write test output files (artifacts).

When the `-artifacts` flag is provided to `go test`,
this directory will be located under the output directory
(specified with `-outputdir`, or the current directory by default).
Otherwise, artifacts are stored in a temporary directory
which is removed after the test completes.

The first call to `ArtifactDir` when `-artifacts` is provided
writes the location of the directory to the test log.

For example, in a test named `TestArtifacts`,
`t.ArtifactDir()` emits:

`=== ARTIFACTS TestArtifacts /path/to/artifact/dir
`

The [`B.Loop`](/pkg/testing#B.Loop) method no longer prevents inlining in
the loop body, which could lead to unanticipated allocation and slower benchmarks.
With this fix, we expect that all benchmarks can be converted from the old
[`B.N`](/pkg/testing#B) style to the new `B.Loop` style with no ill effects.
Within the body of a `for b.Loop() { ... }` loop, function call parameters, results,
and assigned variables are still kept alive, preventing the compiler from
optimizing away entire parts of the benchmark.

#### [`testing/cryptotest`](/pkg/testing/cryptotest/)

The new [`SetGlobalRandom`](/pkg/testing/cryptotest#SetGlobalRandom) function configures a global, deterministic
cryptographic randomness source for the duration of the test. It affects
`crypto/rand`, and all implicit sources of cryptographic randomness in the
`crypto/...` packages.

#### [`time`](/pkg/time/)

The `asynctimerchan` GODEBUG setting introduced in [Go 1.23](/doc/godebug#go-123)
will be removed in the next major Go release.
Starting in Go 1.27, the [`time`](/pkg/time) package will always use unbuffered
(synchronous) channels for timers regardless of GODEBUG setting or go.mod language version.

## Ports

### Darwin

Go 1.26 is the last release that will run on macOS 12 Monterey. Go 1.27 will require macOS 13 Ventura or later.

### FreeBSD

The freebsd/riscv64 port (`GOOS=freebsd GOARCH=riscv64`) has been marked [broken](/wiki/PortingPolicy#broken-ports).
See [issue 76475](/issue/76475) for details.

### Windows

As [announced](/doc/go1.25#windows) in the Go 1.25 release notes, the
[broken](/doc/go1.24#windows) 32-bit windows/arm port (`GOOS=windows`
`GOARCH=arm`) has been removed.

### PowerPC

Go 1.26 is the last release that supports the ELFv1 ABI on the big-endian
64-bit PowerPC port on Linux (`GOOS=linux` `GOARCH=ppc64`).
It will switch to the ELFv2 ABI in Go 1.27.
As the port does not currently support linking against other ELF objects,
we expect this change to be transparent to users.

### RISC-V

The `linux/riscv64` port now supports the race detector.

### S390X

The `s390x` port now supports passing function arguments and results using registers.

### WebAssembly

The compiler now unconditionally makes use of the sign extension
and non-trapping floating-point to integer conversion instructions.
These features have been standardized since at least Wasm 2.0.
The corresponding `GOWASM` settings, `signext` and `satconv`, are now ignored.

For WebAssembly applications, the runtime now manages chunks of
heap memory in much smaller increments, leading to significantly
reduced memory usage for applications with heaps less than around
16 MiB in size.

## Patch Release Changelogs

### go1.26.1

```text
- `c283d1f` [release-branch.go1.26] os: support deleting inaccessible files in RemoveAll (2026-02-04)
- `29618c4` [release-branch.go1.26] cmd/link: don't pass -Wl,-S on illumos (2026-02-18)
- `495e6c6` [release-branch.go1.26] internal/syscall/windows: correct some enums and syscall signatures (2026-02-13)
- `167ef71` [release-branch.go1.26] cmd/compile: rewriteFixedLoad: ensure AuxInt is sign-extended (2026-02-12)
- `e8df1a6` [release-branch.go1.26] reflect: use &zeroVal[0] instead of nil for data field for zero-sized payloads (2026-02-24)
- `12c0690` [release-branch.go1.26] cmd/compile: don't drop same register twice (2026-02-14)
- `85050c9` [release-branch.go1.26] cmd/compile: fix mis-compilation for static array initialization (2026-01-21)
- `6fb3df8` [release-branch.go1.26] go/analysis/passes/modernize: add vendored code (2026-02-17)
- `4659d63` [release-branch.go1.26] cmd/compile: avoid folding 64-bit integers into 32-bit constants (2026-02-14)
- `f67a1f4` [release-branch.go1.26] cmd/go: fix pkg-config flag sanitization (2026-02-03)
- `c572319` [release-branch.go1.26] net/smtp: prevent test failures due to expired test certificate (2026-02-10)
- `155c25e` [release-branch.go1.26] cmd/compile: pointer-shaped types are SSAable even if lots of 0-sized fields (2026-02-11)
- `ef04191` [release-branch.go1.26] cmd/compile: ensure StructMake/ArrayMake1 of direct interfaces are unwrapped (2026-02-11)
- `8cce3ab` [release-branch.go1.26] os: avoid escape from Root via ReadDir or Readdir (2026-02-26)
- `4d1051f` [release-branch.go1.26] cmd: re-vendor x/tools for Go 1.26.1 (2026-03-02)
- `e28ac67` [release-branch.go1.26] cmd/go: revert update default go directive in mod or work init (2026-02-27)
- `65c7d7a` [release-branch.go1.26] net/url: reject IPv6 literal not at start of host (2026-01-28)
- `9946928` [release-branch.go1.26] html/template: properly escape URLs in meta content attributes (2026-01-09)
- `a761c9f` [release-branch.go1.26] crypto/x509: fix full email constraint matching (2026-02-11)
- `e792d6a` [release-branch.go1.26] crypto/x509: fix name constraint checking panic (2026-02-11)
- `e87b10e` [release-branch.go1.26] go1.26.1 (2026-03-06)
```

### go1.26.2

```text
- `840d86c` [release-branch.go1.26] runtime: fix printfloat, printcomplex buffer sizes (2026-02-27)
- `1368bc2` [release-branch.go1.26] cmd/compile: fix bloop test (2026-03-03)
- `ace2cf6` [release-branch.go1.26] cmd/compile/internal/typecheck: simplify tcSliceHeader (2026-03-03)
- `99df9bd` [release-branch.go1.26] net: correctly handle non-nil empty cmsg buffer on windows (2026-03-01)
- `ba8c3d0` [release-branch.go1.26] net/http: restore accidentally removed package doc (2026-03-04)
- `2b01132` [release-branch.go1.26] runtime/poll: fix race condition in Window's SendFile (2026-03-08)
- `e29f355` [release-branch.go1.26] cmd/go/internal/cache: update trim timestamp before trimming (2026-03-09)
- `3bf3014` [release-branch.go1.26] test/convert5: skip mipsle (2026-02-10)
- `1a44be4` [release-branch.go1.26] internal/poll: move rsan to heap on windows (2026-03-09)
- `943235a` [release-branch.go1.26] builtin: update new function comment (2026-02-18)
- `12bf7d8` [release-branch.go1.26] builtin: incorporate all feedback into doc string for new (2026-02-25)
- `26548d4` [release-branch.go1.26] cmd/fix: pull in CL 755980 (2026-03-23)
- `895fbae` [release-branch.go1.26] doc: remove template use in godebug.md (2026-03-18)
- `08edcb8` [release-branch.go1.26] test/fixedbugs: remove issue46234 test timeout (2026-02-20)
- `9fe1db7` [release-branch.go1.26] runtime: fix value of ENOSYS on mips from 38 to 89 (2026-02-22)
- `68f42c5` [release-branch.go1.26] cmd/compile: gate instrumentEnterExit on NoRaceFunc check (2026-02-23)
- `f14d3a8` [release-branch.go1.26] cmd/compile: skip race detector test failure for unsupported VMA (2026-03-19)
- `82b95c1` [release-branch.go1.26] cmd/compile: ternary rewrite of rewrite should skip, not panic (2026-02-25)
- `3206e14` [release-branch.go1.26] net/url: reattach package doc comment (2026-03-24)
- `cce667a` [release-branch.go1.26] cmd/compile: handle zero-sized values more generally (2026-02-17)
- `98d4c11` [release-branch.go1.26] cmd/compile: treat all zero-sized values as SSA-able (2026-02-20)
- `e27183e` [release-branch.go1.26] cmd/compile: fix internal compiler error: bad write barrier type (2026-02-26)
- `608303d` [release-branch.go1.26] cmd/link: skip special symbols for label symbol generation (2026-03-17)
- `4ef2787` [release-branch.go1.26] runtime/race: apply LLVM zero-initialization fix (2026-03-18)
- `8028a51` [release-branch.go1.26] crypto/x509: hoist policy pruning out of loop (2026-03-23)
- `7b4ed1d` [release-branch.go1.26] crypto/x509: fix signature checking limit (2026-03-05)
- `917949c` [release-branch.go1.26] net/url: permit colons in the host subcomponent of non-http/https URLs (2026-03-26)
- `b6176f4` [release-branch.go1.26] internal/syscall/unix: properly support AT_SYMLINK_NOFOLLOW on Linux (2026-03-23)
- `babb1c3` [release-branch.go1.26] html/template: properly track JS template literal brace depth across contexts (2026-03-23)
- `ceb4da6` [release-branch.go1.26] crypto/x509: fix wildcard constraint map case sensitivity (2026-03-23)
- `3a4577a` [release-branch.go1.26] cmd/compile: fix mem access overlap detection (2026-03-12)
- `c4b4bd7` [release-branch.go1.26] cmd/compile: fix loopbce overflow check logic (2026-03-06)
- `a34b5e4` [release-branch.go1.26] archive/tar: limit the number of old GNU sparse format entries (2026-03-23)
- `7cafb41` [release-branch.go1.26] crypto/tls: prevent deadlock when client sends multiple key update messages (2026-03-23)
- `096f21b` [release-branch.go1.26] cmd/go: disallow cgo trust boundary bypass (2026-02-24)
- `9c8bf0e` [release-branch.go1.26] go1.26.2 (2026-04-07)
```

### go1.26.3

```text
- `0b4d5f8` [release-branch.go1.26] cmd/link: use bfd ld 2.36+ on linux/arm64 instead of gold (2026-01-29)
- `f4e425d` [release-branch.go1.26] fix incorrect loop trip counts (2026-04-14)
- `ba4554f` [release-branch.go1.26] cmd/go: specify full path to go command when running go tool covdata (2026-03-17)
- `efdc0fb` [release-branch.go1.26] cmd/compile: handle min integer step in loop (2026-04-13)
- `710f29a` [release-branch.go1.26] runtime: add sysUnreserve to undo sysReserve (2026-04-02)
- `ec5ebec` [release-branch.go1.26] all: update x/net to 705de46f (2026-04-17)
- `be12fe1` [release-branch.go1.26] runtime: use uname version check for 64-bit time on 32-bit arch codepaths (2026-03-24)
- `9b01c04` [release-branch.go1.26] html/template: fix escaper bypass by treating empty script type as JavaScript (2026-04-27)
- `9547066` [release-branch.go1.26] net/http/httputil: reencode queries with many parameters in proxy (2026-04-24)
- `cb994d8` [release-branch.go1.26] cmd/fix: change -diff to exit 1 if diffs exist (2026-03-03)
- `c971287` [release-branch.go1.26] crypto/internal/fips140/drbg: build tag out entropy generation on Wasm (2026-03-23)
- `e137885` [release-branch.go1.26] cmd/compile/internal/devirtualize: use pointer identity for type comparison (2026-03-27)
- `50856a1` [release-branch.go1.26] cmd/go: use MkdirTemp to create temp directory for "go bug" (2026-04-08)
- `73f417a` [release-branch.go1.26] go/types, types2: handle unconstrained type parameters correctly in a few places (2026-03-12)
- `e9df527` [release-branch.go1.26] crypto/tls: wrap ML-KEM hybrids in fips140.WithoutEnforcement (2026-03-25)
- `19d2ce3` [release-branch.go1.26] runtime: fix timespec definition on 32bits systems (2026-03-04)
- `f43caf8` [release-branch.go1.26] lib/fips140: update inprocess to v1.26.0 (2026-04-27)
- `2992086` [release-branch.go1.26] lib/fips140: add certified pointing to v1.0.0-c2097c7c (2026-04-27)
- `e0f5c05` [release-branch.go1.26] net: avoid double-free of cgo pointer when handling large DNS response (2026-03-26)
- `18fa391` [release-branch.go1.26] crypto/fips140: add package docs (2026-04-28)
- `dd29b59` [release-branch.go1.26] cmd/compile: fix loopvar version detection with line directives (2026-01-21)
- `d9389d3` [release-branch.go1.26] net/mail: fix quadratic complexity in consumeComment (2026-04-04)
- `40fa774` [release-branch.go1.26] cmd/pack: refuse to extract files with directory components (2026-04-15)
- `c9f97f5` [release-branch.go1.26] html/template: fix escaping of URLs in meta content attributes (2026-04-22)
- `3ae315a` [release-branch.go1.26] os: avoid panic when RemoveAll fails to remove a symlink (2026-04-06)
- `722b68c` [release-branch.go1.26] cmd/compile: keep blank nodes alive in b.loop (2026-03-12)
- `7136366` [release-branch.go1.26] cmd/go: invalidate test cache when -coverpkg dependencies change (2026-04-09)
- `0bec633` [release-branch.go1.26] net/mail: fix quadratic consumePhrase behavior (2026-04-28)
- `3baf3ee` [release-branch.go1.26] all: avoid unsafe StringToUTF16Ptr on Windows (2026-04-24)
- `8282c62` [release-branch.go1.26] cmd/go: reject sumdb response lacking module hash (2026-04-30)
- `2dc996f` [release-branch.go1.26] go1.26.3 (2026-05-07)
```

### go1.26.4

```text
- `26c2ac5` [release-branch.go1.26] crypto/internal/fips140/drbg: fix Wasm stub (2026-05-05)
- `ccffd99` [release-branch.go1.26] cmd/fix: pull in CL 783960 (2026-05-27)
- `7f24db4` [release-branch.go1.26] mime: avoid quadratic complexity in WordDecoder.DecodeHeader (2026-05-05)
- `ec1c380` [release-branch.go1.26] net/textproto: escape arbitrary input when including them in errors (2026-05-11)
- `ce5a3e7` [release-branch.go1.26] crypto/x509: split candidate hostname only once (2026-05-27)
- `7d24489` [release-branch.go1.26] runtime/race: build race detector linux/arm64 syso with no-outline-atomics (2026-05-27)
- `babc19e` [release-branch.go1.26] cmd/compile: do not misscompile x+x << 63 to x << 0 on amd64 (2026-05-04)
- `a9ce111` [release-branch.go1.26] go1.26.4 (2026-06-02)
```

### go1.26.5

```text
- `ec988a9` [release-branch.go1.26] os: notify testlog from (*File).Chdir (2026-04-29)
- `fe91cea` [release-branch.go1.26] cmd/go: use fsys.ReadDir for IsStandardPackage (2026-04-03)
- `6efcff6` [release-branch.go1.26] os/signal: make NotifyContext Cause match context.Canceled (2026-05-19)
- `dc80258` [release-branch.go1.26] internal/runtime/gc: require AVX512DQ for greentea (2026-06-09)
- `d7c9c81` [release-branch.go1.26] runtime: in moveSliceNoCap, round size down to a whole multiple of element size (2026-06-14)
- `2055b1a` [release-branch.go1.26] syscall: add //go:norace to rawSyscall on darwin (2026-06-03)
- `8145914` [release-branch.go1.26] cmd/compile,sync/atomic: make Add And & Or SQCST on PPC64 (2026-05-04)
- `044c995` [release-branch.go1.26] runtime: tolerate vendor suffixes in Linux kernel release strings (2026-05-22)
- `7397dd7` [release-branch.go1.26] net: fix TestLookupCNAME (2026-06-30)
- `ca8ca59` [release-branch.go1.26] crypto/tls: omit PSK in ECH outer client hello (2026-05-08)
- `f9ef7f5` [release-branch.go1.26] os: properly handle trailing slashes in paths in Root (2026-05-26)
- `c19862e` [release-branch.go1.26] go1.26.5 (2026-07-07)
```

### go1.26.6

```text
- `5bbd22f` [release-branch.go1.26] net/http: apply header timeout to server's unencrypted HTTP/2 check (2026-06-24)
- `a42fec4` [release-branch.go1.26] all: update x/net (2026-07-10)
- `07a0bbb` [release-branch.go1.26] os: don't expect an error from Root.ReadFile from a directory on NetBSD (2026-07-09)
- `cc31225` [release-branch.go1.26] runtime: be sure to scan small frame introduced by (*sigctxt).pushCall (2026-07-07)
- `28f27d6` [release-branch.go1.26] os: properly handle trailing / in Root.MkdirAll (2026-07-08)
- `4238449` [release-branch.go1.26] os: strip trailing slashes in RemoveAll on Plan 9 (2026-07-09)
- `88bf579` [release-branch.go1.26] runtime: don't emit write barrier for code pointers in itabInit (2026-07-21)
- `ae63e27` [release-branch.go1.26] runtime: fix uninitialized 7th argument in mach_vm_region_trampoline (2026-07-16)
- `b643231` [release-branch.go1.26] crypto/tls: do not count handshake messages as state-advancing post-handshake (2026-07-22)
- `019fcb0` [release-branch.go1.26] cmd/compile: don't require Heapaddr for heap vars in dead code (2026-06-22)
- `0e8a244` [release-branch.go1.26] all: update x/net (2026-07-31)
- `4d522f2` [release-branch.go1.26] cmd/compile: tighten mergelocals address use analysis (2026-07-02)
- `100fb8f` [release-branch.go1.26] cmd/link: quote PE .def library name (2026-06-23)
- `605fd48` [release-branch.go1.26] crypto/tls: add fips140ems GODEBUG setting to disable EMS enforcement (2026-06-19)
- `efc7967` [release-branch.go1.26] cmd/compile: do not home mul/div results in HI/LO (2026-07-24)
- `f8d4fa6` [release-branch.go1.26] runtime: fix frame pointer adjustment around injected calls (2026-07-31)
- `f5bf46f` [release-branch.go1.26] cmd/compile: fix mips64le bigger than 32bits pointer offsets (2026-07-28)
- `db38ee7` [release-branch.go1.26] cmd/compile: fix prove to generate Const64 for 64bits slicemasks (2026-06-06)
- `52e5e2f` [release-branch.go1.26] cmd/compile: fix time traveling proofs in prove (2026-07-22)
- `13c1940` [release-branch.go1.26] cmd/compile: do not elide riscv64 sign extension of unsigned values (2026-07-27)
- `128893d` [release-branch.go1.26] net/url: avoid quadratic complexity in resolvePath (2026-07-21)
- `33ecb96` [release-branch.go1.26] html/template: fix JavaScript regexp tracking (2026-07-28)
- `9918f26` [release-branch.go1.26] encoding/xml: fix depth processing in (*Decoder).unmarshal (2026-07-20)
- `6ec908d` [release-branch.go1.26] encoding/asn1: enforce maximum recursion depth (2026-06-08)
- `9f6980f` [release-branch.go1.26] cmd/vendor: fix CVE-2026-56864 in x/mod (2026-08-07)
- `115eb47` [release-branch.go1.26] cmd/vendor: fix CVE-2026-56865 in x/mod (2026-08-07)
- `1ea5a71` [release-branch.go1.26] go1.26.6 (2026-08-13)
```

### go1.26.7

```text
- `98e6631` [release-branch.go1.26] cmd/internal/moddeps: restore tests (2026-08-13)
- `50e5b59` [release-branch.go1.26] net/http: clear ReadHeaderTimeout after accepting an unencrypted h2 conn (2026-08-17)
- `e3336a2` [release-branch.go1.26] go1.26.7 (2026-08-19)
```

### go1.26.8

```text
- `d1434db` [release-branch.go1.26] internal/cpu: disable AVX+ on netbsd/amd64 (2026-07-07)
- `dbe8fd9` [release-branch.go1.26] os: don't symlink on Root.Mkdir("symlink/") on openbsd (2026-07-14)
- `d19a1e8` [release-branch.go1.26] cmd/go/internal: fix copy paste error (2026-04-03)
- `e9fde2d` [release-branch.go1.26] debug/elf: use addend when applying ppc relocations (2026-08-25)
- `3cc00d9` [release-branch.go1.26] cmd/compile: fix mergelocals for arm32/s390x/riscv64 (2026-08-27)
- `c293dd4` [release-branch.go1.26] go1.26.8 (2026-09-01)
```

### go1.26.9

```text
- `2c99952` [release-branch.go1.26] crypto/mlkem: fix FIPS 130-3 CAST (2026-08-27)
- `5e38ca6` [release-branch.go1.26] runtime: disable mutex profile during async preemption on windows (2026-08-21)
- `25d767a` [release-branch.go1.26] runtime: netpoll: fix 32-bit little-endian eventfd type confusion (2026-08-23)
- `ee49ded` [release-branch.go1.26] cmd/compile: fix large riscv64 move/zero (2026-08-31)
- `6a97c5b` [release-branch.go1.26] net/http: vendor in CL 836505 (2026-09-25)
- `5bf869b` [release-branch.go1.26] internal/runtime/syscall/linux: sync EpollEvent with the kernel header (2026-08-20)
- `19777cc` [release-branch.go1.26] cmd/compile: don't stack-allocate a slice whose element interior escapes (2026-09-21)
- `c7183cd` [release-branch.go1.26] os: handle unsupported OBJ_DONT_REPARSE on older Windows (2026-09-07)
- `2a1834b` [release-branch.go1.26] cmd/compile: fix slice backing store analysis (2026-06-10)
- `179610a` [release-branch.go1.26] os: preserve completion notification modes for NewFile handles (2026-09-07)
- `305bfa6` [release-branch.go1.26] cmd/go: include VetxOnly in the vet action ID (2026-09-26)
- `7060ef9` [release-branch.go1.26] cmd/link: pass -no_fixup_chains to C linker when building plugin on macOS (2026-09-30)
- `e59c03e` [release-branch.go1.26] html/template: reset JS context for template expressions (2026-09-28)
- `5d84f6f` [release-branch.go1.26] html/template: recognize `yield` as regexp preceder keyword (2026-09-28)
- `5fa9de4` [release-branch.go1.26] cmd/go: bundle go.sum h1 hashes for golang.org/fips140 (2026-09-28)
- `7325b72` [release-branch.go1.26] cmd/go: always verify toolchain downloads with sumdb (2026-09-28)
- `eb1bf6e` [release-branch.go1.26] crypto/tls: validate ECH outer extension references (2026-09-28)
- `d7a7aa8` [release-branch.go1.26] net/textproto: enforce MIME memory limit on first line (2026-09-22)
- `3fe1aac` [release-branch.go1.26] net/http, net/http/httputil: avoid post-CONNECT pool corruption (2026-08-14)
- `087de1f` [release-branch.go1.26] net/http: avoid server reuse of connection after CONNECT (2026-08-14)
- `be88124` [release-branch.go1.26] os: correct Root junction handling on Windows (avoid mkdir escape) (2026-09-15)
- `eb25b4e` [release-branch.go1.26] net/http: limit the size of range headers parsed in ServeContent (2026-09-14)
- `6e04952` [release-branch.go1.26] net/http: bundled HTTP/2 vuln fixes (2026-09-30)
- `2ae494e` [release-branch.go1.26] go1.26.9 (2026-10-08)
```

### go1.26.10

```text
- `51e7ce3` [release-branch.go1.26] cmd/internal/moddeps: restore tests (2026-10-08)
- `aa64342` [release-branch.go1.26] crypto/x509: distrust a handful of intermediates (2026-10-19)
- `4e8c85f` [release-branch.go1.26] go1.26.10 (2026-10-20)
```
