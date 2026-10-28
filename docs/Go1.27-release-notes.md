# Go 1.27 Release Notes

## Introduction to Go 1.27

The latest Go release, version 1.27, arrives in [August 2026](/doc/devel/release#go1.27.0), six months after [Go 1.26](/doc/go1.26).
Most of its changes are in the implementation of the toolchain, runtime, and libraries.
As always, the release maintains the Go 1 [promise of compatibility](/doc/go1compat).
We expect almost all Go programs to continue to compile and run as before.

## Changes to the language

Go 1.27 now supports [generic methods](/issue/77273):
a [method declaration](/ref/spec#Method_declarations) may declare its own
[type parameters](/ref/spec#Type_parameter_declarations).
This widely anticipated change allows adding generic functions within
the namespace of a particular data type where before one had to declare
such functions with a scope of the entire package.
As an example, `math/rand/v2` now also declares a generic method with signature
`(*Rand) N[Int intType](Int) Int`, whereas previously it had just a
generic function `N[Int intType](Int) Int` (see [below](#rand) for details).
Note that methods of [interfaces](/ref/spec#Interface_types) may not declare
type parameters nor can interface methods be implemented by generic methods.

A key in a [struct literal](/ref/spec#Composite_literals) may now be any
valid [field selector](/issue/9859) for the struct type, not just a
(top-level) field name of the struct.

Function type inference has been [generalized](/issue/77245) to apply in all
contexts where a generic function is [assigned](/ref/spec#Assignability) to a
variable of (or converted to) a matching function type.

## Tools

Response file (`@file`) parsing is now supported for the `compile`, `link`, `asm`, `cgo`, `cover`, and `pack` tools.
The response file contains whitespace-separated arguments with support for single-quoted and double-quoted strings, escape sequences, and backslash-newline line continuation.
The format is compatible with GCC&rsquo;s response file implementation to ensure interoperability with existing build systems.

### Go command

The `go` command no longer has support for the `bzr` version control system.
It will no longer be able to directly fetch modules hosted on `bzr` servers.

#### `GODEBUG`

Starting with Go 1.27, the `go` command now recognizes a `GODEBUG` setting
for which support was removed (such as `asynctimerchan`, see [below](#runtime)) if it appears in `go.mod`
files (`godebug` entries) and `.go` source files (`//go:debug` comments).
It accepts these settings if they are set to the final default value established before
the setting was removed.
If they are set to an old value, the `go` command will fail.
This change is in the spirit of the [Go 1 compatibility guarantee](/doc/go1compat)
and allows existing programs that set supported `GODEBUG` settings to continue to
build and run without changes even when the respective setting support has been removed.

#### `go test`

`go test` now invokes the `stdversion` vet check by default.
This reports the use of standard library symbols that are too new
for the Go version in force in the referring file,
as determined by `go` directive in `go.mod` and build tags on the file.

`go test -json` now annotates `"Action":"output"` lines with an optional
new field `"OutputType"`, specifying the type of output.
Currently, the possible values include &ldquo;error&rdquo;, &ldquo;error-continue&rdquo;, and &ldquo;frame&rdquo;.
See [cmd/test2json help](/cmd/test2json#hdr-Output_Format) for details.

#### `go doc`

The `go doc` command now supports `package@version` syntax, such as
`go doc example.com/pkg@v1.2.3`.

The `go doc` command now accepts the `-ex` command-line option to
list executable examples of the given package or symbol.
When an example name is passed on the command line (such as
`go doc bytes.ExampleBuffer`), `go doc` now prints the example source
code along with comments.

#### `go fix`

The `go fix` command contains several new modernizers (`atomictypes`, `embedlit`, `slicesbackward`, and `unsafefuncs`).

The existing `fmtappendf` analyzer was removed due to stylistic concerns.

The existing `waitgroup` analyzer was renamed to `waitgroupgo` to avoid ambiguity.

#### `go mod tidy`

For modules specifying `go 1.27` or later in their `go.mod` file, `go mod tidy`
now automatically merges duplicate require blocks. This ensures the file
maintains a clean, standard structure containing at most two require blocks:
one for direct dependencies and one for indirect dependencies.

Existing comment blocks attached to dependencies are preserved during this
consolidation. If a comment block is associated with a mixed set of directives
(containing both direct and indirect dependencies), the comment block is merged
and attached to the new direct dependency block.

Previously, if a `go.mod` file accumulated multiple disjoint require blocks
(often due to manual edits, unresolved Git merge conflicts, or legacy upgrades)
`go mod tidy` would leave the extra blocks intact or inadvertently create new
ones. The tool now strictly enforces the two-block layout, consolidating
disparate requirements into their respective blocks and cleaning up the
structure of the module file automatically.

### Gofmt

This release includes a fix to the logic used by `gofmt` (and
[`go/format`](/pkg/go/format)) in deciding column alignment. The
previous logic caused the behavior of `gofmt` to vary slightly across
CPU architectures. The new logic changes the alignment decision for a
small number of files, so running `gofmt` from Go 1.27 on previously
formatted code may produce minor whitespace changes.

### Vet

The [printf](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/printf)
analyzer now reports calls such as `fmt.Errorf("...: %w", p)` in which
the `%w` operand `p` has type `*E`, where the type `E` itself
implements `error`. Wrapping a pointer in this way is usually a latent bug
since it causes spurious failures of operations that test for
a particular error type, such as a type assertion or call to
[`errors.Is`](/pkg/errors#Is).

### Trace

`go tool trace`&rsquo;s `-http` command-line option now restricts the listen
address to localhost when passed only a port (e.g., `-http=:6060`).
This change makes `go tool trace` consistent with the behavior of
`go tool pprof`&rsquo;s `-http` flag.
To listen on all addresses, explicitly include the specified address
(e.g., `-http=0.0.0.0:6060`).

## Runtime

Tracebacks for modules with `go` directives configuring Go 1.27 or later will now
include [`runtime/pprof`](https://pkg.go.dev/runtime/pprof) goroutine labels in
the header line. This behavior can be disabled with `tracebacklabels=0` `GODEBUG` setting
(added in [Go 1.26](/doc/godebug#go-126)). This opt-out is expected to be
kept indefinitely in case goroutine labels acquire sensitive information that
shouldn&rsquo;t be made available in tracebacks.

The `asynctimerchan` `GODEBUG` setting (added in [Go 1.23](/doc/godebug#go-123))
has been removed permanently. Channels created by package [time](https://pkg.go.dev/time)
are now always unbuffered (synchronous), irrespective of `GODEBUG` settings.

### Faster memory allocation

The compiler now generates calls to size-specialized memory allocation
routines, reducing the cost of some small (<80 byte) memory allocations by
up to 30%.
Improvements vary depending on the workload, but the overall improvement is
expected to be ~1% in real allocation-heavy programs.
This causes the binary size to increase by about 60 KB (independent of the
workload).
Please [file an issue](/issue/new) if you notice any regressions.
You may set `GOEXPERIMENT=nosizespecializedmalloc` at build time to disable
it.
This opt-out setting is expected to be removed in Go 1.28.

### Goroutine leak profile

A new profile type that reports leaked goroutines, previously available
as an experiment in [Go 1.26](/doc/go1.26#goroutineleak-profiles), is now
generally available.
The new profile type, named `goroutineleak`, is supported in the
[`runtime/pprof`](/pkg/runtime/pprof) package.
It is also available as the [`net/http/pprof`](/pkg/net/http/pprof)
endpoint `/debug/pprof/goroutineleak`.

A leaked* goroutine is a goroutine blocked on some concurrency primitive
(channels, [`sync.Mutex`](/pkg/sync#Mutex), [`sync.Cond`](/pkg/sync#Cond), etc)
that cannot possibly become unblocked.
The runtime detects leaked goroutines using the garbage collector: if a
goroutine G is blocked on concurrency primitive P, and P is unreachable from
any runnable goroutine or any goroutine that *those* could unblock, then P
cannot be unblocked, so goroutine G can never wake up.
While it is impossible to detect permanently blocked goroutines in all cases,
this approach detects a large class of such leaks.

Because this technique builds on reachability, the runtime may fail to identify
leaks caused by blocking on concurrency primitives reachable through global
variables or the local variables of runnable goroutines.

See [Go 1.26 release notes](/doc/go1.26#goroutineleak-profiles) for an example.

Special thanks to Vlad Saioc at Uber for contributing this work.

The `goroutineleakprofile` `GOEXPERIMENT` setting is now deleted.

## Compiler

The compiler now resolves a relative filename in a `//line` or `/*line*/`
directive against the directory of the file containing the directive,
matching the behavior of [`go/scanner`](/pkg/go/scanner).
Absolute filenames are unaffected.
See [#70478](/issue/70478).

The compiler now generates simpler names for function literals
(closures).
Previously, when the containing function is inlined, the function
literal&rsquo;s name can get quite long.
Now the compiler chooses the same name for the function literal
regardless of inlining.
It may also combine multiple instances of the same function literal
(as its containing function is inlined) to share the same code in
the compiled binary.
This change does not affect the functionality of Go code.
Tests that check symbol names may need update, although it is
recommended to not depend on the names of function literals.
For programs that [incorrectly](/pkg/reflect#Value.Pointer) compare
function code pointer for equality, the issue may be more exposed
with Go 1.27, as function literals with different captured closure
data may have equal code pointers in more cases.

## Linker

When targeting macOS, the linker now accepts `-macos` and `-macsdk`
command-line options, which specify the OS and SDK versions in the
`LC_BUILD_VERSION` load command.
By default, it selects the oldest supported macOS version (currently
[13.0.0](#darwin)) and a recent SDK version (currently 26.2.0).

## Standard library

### New encoding/json/v2 and encoding/json/jsontext packages

Two new packages are now available:

The [`encoding/json/v2`](/pkg/encoding/json/v2) package is a major
revision of [`encoding/json`](/pkg/encoding/json). It provides
[`Marshal`](/pkg/encoding/json/v2#Marshal),
[`MarshalWrite`](/pkg/encoding/json/v2#MarshalWrite),
[`MarshalEncode`](/pkg/encoding/json/v2#MarshalEncode),
[`Unmarshal`](/pkg/encoding/json/v2#Unmarshal),
[`UnmarshalRead`](/pkg/encoding/json/v2#UnmarshalRead), and
[`UnmarshalDecode`](/pkg/encoding/json/v2#UnmarshalDecode),
all of which accept variadic [`Options`](/pkg/encoding/json/v2#Options)
arguments to configure marshaling and unmarshaling behavior.

-

The [`encoding/json/jsontext`](/pkg/encoding/json/jsontext) package
provides lower-level syntactic processing of JSON.
The [`Encoder`](/pkg/encoding/json/jsontext#Encoder) and
[`Decoder`](/pkg/encoding/json/jsontext#Decoder) types operate on
JSON as a sequence of
[`Token`](/pkg/encoding/json/jsontext#Token) and
[`Value`](/pkg/encoding/json/jsontext#Value),
maintaining a state machine to ensure the produced or consumed
sequence is valid JSON text.

The v2 package chooses stricter, more interoperable defaults than v1:
it rejects invalid UTF-8 in JSON strings and rejects duplicate names within
a JSON object. See the v1 [`encoding/json`](/pkg/encoding/json#hdr-Migrating_to_v2) package
documentation for the complete set of behavioral differences and
the options available to adjust them.

The [`encoding/json`](/pkg/encoding/json) package is now backed by the
v2 implementation. Marshaling and unmarshaling behavior is preserved, but
the exact text of error messages may differ. The package also gains a number of
new [`Options`](/pkg/encoding/json#Options) that can configure v2 to operate
with v1 semantics to avoid requiring a full migration to the new API.
The v1 API will continue to be supported and users are not required to migrate.

Marshal performance is broadly at parity with the previous implementation,
while unmarshal performance is significantly faster.

Users who encounter compatibility problems with the new implementation
may disable it by setting `GOEXPERIMENT=nojsonv2` at build time,
restoring the original v1 implementation.
This opt-out is expected to be removed in a future release.

Some notable changes to `encoding/json/v2` over the course of the `GOEXPERIMENT` include
the following removals:
the `format` tag option ([#79071](/issue/79071)),
the `unknown` tag option ([#77271](/issue/77271)),
the `DiscardUnknownMembers` marshal option ([#77271](/issue/77271)),
and the `SkipFunc` sentinel error ([#74324](/issue/74324)).
In addition, the `inline` tag option was renamed to `embed` ([#79985](/issue/79985)), and
behavior was updated for the [`string`](/pkg/encoding/json/v2#hdr-JSON_Representation_of_Go_structs)
tag option ([#79065](/issue/79065))
and the [`MatchCaseInsensitiveNames`](/pkg/encoding/json/v2#MatchCaseInsensitiveNames)
option ([CL 792780](/cl/792780)).
In `encoding/json/jsontext`, the numeric [`Token`](/pkg/encoding/json/jsontext#Token)
accessors were changed to also return errors ([#77666](/issue/77666)).

See the [proposal issue](/issue/71497) for background and additional detail.
If you need to disable the new implementation, [please file an issue](/issue/new).

### New crypto/mldsa package

The new [`crypto/mldsa`](/pkg/crypto/mldsa) package implements the post-quantum ML-DSA signature
scheme specified in FIPS 204.

[`crypto/x509`](/pkg/crypto/x509) now supports ML-DSA private keys, public keys, and signatures.

[`crypto/tls`](/pkg/crypto/tls) now supports ML-DSA signatures in TLS 1.3, with the new
[`MLDSA44`](/pkg/crypto/tls#MLDSA44), [`MLDSA65`](/pkg/crypto/tls#MLDSA65),
and [`MLDSA87`](/pkg/crypto/tls#MLDSA87) [`SignatureScheme`](/pkg/crypto/tls#SignatureScheme) values.

### New uuid package

The new [`uuid`](/pkg/uuid) package generates and parses UUIDs.

### New experimental simd package

Go 1.27 introduces a new experimental [`simd`](/pkg/simd)
package that provides portable and vector-size-agnostic SIMD
support. It will make use of the hardware instructions
if they are available.
This package is enabled by setting the environment variable
`GOEXPERIMENT=simd` at build time.

The `simd` package is available on all architectures, and provides
vector types of unspecified size such as [`Int8s`](/pkg/simd#Int8s) and [`Float32s`](/pkg/simd#Float32s).
It supports a &ldquo;scalable&rdquo; subset of the operations present in the
[`simd/archsimd`](/pkg/simd/archsimd) package
that are hardware-supported or easily emulated across architectures
and vector widths.

See the [proposal issue](/issue/78902) for more details.

### Experimental simd/archsimd package

Go 1.27 continues the experimental support for SIMD operations in
the [`simd/archsimd`](/pkg/simd/archsimd/) package that began in [Go 1.26](/doc/go1.26#simd).
This release revises the amd64 API and adds support for arm64 &ldquo;Neon&rdquo; 128-bit SIMD and WebAssembly 128-bit SIMD.
The `simd/archsimd` package is enabled by setting the environment variable
`GOEXPERIMENT=simd` at build time.

This package provides access to architecture-specific SIMD operations.
It supports 128-bit vector types on wasm, arm64, and amd64,
and 256-bit and 512-bit vector types on some amd64 processors.
The API is not yet considered stable.

See the [package documentation](/pkg/simd/archsimd) and the [proposal issue](/issue/73787) for more details.

We intend to provide support for additional architectures in future versions, but the
API is intentionally architecture-specific and thus non-portable.

### Minor changes to the library

#### [`bytes`](/pkg/bytes/)

The new [`CutLast`](/pkg/bytes#CutLast) function slices a `[]byte`
around the last occurrence of a separator.
It can replace and simplify some common uses of [`LastIndex`](/pkg/bytes#LastIndex).

#### [`compress/flate`](/pkg/compress/flate/)

Compression speed is improved in Go 1.27.
The exact encoded output from [`Writer`](/pkg/compress/flate#Writer) may be different from Go 1.26
as a result of the encoder implementation change.
Since DEFLATE is the underlying compression used in [`archive/zip`](/pkg/archive/zip), [`compress/gzip`](/pkg/compress/gzip), [`compress/zlib`](/pkg/compress/zlib), and [`image/png`](/pkg/image/png),
the outputs from those packages may also have changed.

#### [`crypto`](/pkg/crypto/)

The new [`MLDSAMu`](/pkg/crypto#MLDSAMu) [`Hash`](/pkg/crypto#Hash) value
is added for use as a signaling mechanism for External μ ML-DSA signing.

#### [`crypto/ecdsa`](/pkg/crypto/ecdsa/)

[`PrivateKey.Sign`](/pkg/crypto/ecdsa#PrivateKey.Sign) now checks that the length of the hash is correct,
if a non-nil [`SignerOpts`](/pkg/crypto#SignerOpts) is provided.

#### [`crypto/tls`](/pkg/crypto/tls/)

The new [`QUICConfig.ClientHelloInfoConn`](/pkg/crypto/tls#QUICConfig.ClientHelloInfoConn) field specifies the [`net.Conn`](/pkg/net#Conn) to use
for the [`ClientHelloInfo.Conn`](/pkg/crypto/tls#ClientHelloInfo.Conn) field during QUIC server handshakes.

The [`MLKEM1024`](/pkg/crypto/tls#MLKEM1024) key exchange is now supported. It can be enabled by adding it to
[`Config.CurvePreferences`](/pkg/crypto/tls#Config.CurvePreferences).

[`Config.Rand`](/pkg/crypto/tls#Config.Rand) is now deprecated.
For deterministic testing, use [`testing/cryptotest.SetGlobalRandom`](/pkg/testing/cryptotest#SetGlobalRandom).

Post-quantum hybrid key exchanges can now be explicitly enabled in
[`Config.CurvePreferences`](/pkg/crypto/tls#Config.CurvePreferences) even if the `tlsmlkem=0` or `tlssecpmlkem=0` `GODEBUG`
options are used. Those options were always meant to only apply to the default
set used when [`Config.CurvePreferences`](/pkg/crypto/tls#Config.CurvePreferences) is nil.

The new [`ConnectionState.LocalCertificate`](/pkg/crypto/tls#ConnectionState.LocalCertificate)
field contains the certificate chain presented to the connection peer during the handshake.

The `tlsunsafeekm` (added in [Go 1.22](/doc/godebug#go-122)),
`tlsrsakex` (added in [Go 1.22](/doc/godebug#go-122)),
`tls3des` (added in [Go 1.23](/doc/godebug#go-123)),
`tls10server` (added in [Go 1.22](/doc/godebug#go-122)),
and `x509keypairleaf` (added in [Go 1.23](/doc/godebug#go-123))
`GODEBUG` settings have been removed permanently.

The `tlskyber` `GODEBUG` setting (added in [Go 1.23](/doc/godebug#go-123))
was removed in [Go 1.24](/doc/godebug#go-124) but the removal was not
documented in the [release notes for Go 1.24](/doc/go1.24.md) at that time;
it is documented now.

#### [`crypto/x509`](/pkg/crypto/x509/)

When parsing into [`pkix.Name`](/pkg/crypto/x509/pkix#Name) fields,
a wider range of [`pkix.AttributeTypeAndValue.Value`](/pkg/crypto/x509/pkix#AttributeTypeAndValue.Value)
types is now supported, and unknown types are parsed into
[`asn1.RawValue`](/pkg/encoding/asn1#RawValue).

The new [`Certificate.RawSignatureAlgorithm`](/pkg/crypto/x509#Certificate.RawSignatureAlgorithm), [`CertificateRequest.RawSignatureAlgorithm`](/pkg/crypto/x509#CertificateRequest.RawSignatureAlgorithm),
and [`RevocationList.RawSignatureAlgorithm`](/pkg/crypto/x509#RevocationList.RawSignatureAlgorithm) fields expose the DER-encoded
AlgorithmIdentifier of the signature algorithm, including when the
SignatureAlgorithm field is [`UnknownSignatureAlgorithm`](/pkg/crypto/x509#UnknownSignatureAlgorithm).

[`SystemCertPool`](/pkg/crypto/x509#SystemCertPool) now respects SSL_CERT_FILE and SSL_CERT_DIR on Windows and
Darwin. When these environment variables are set, roots are loaded from disk and
instead of using the platform certificate verification APIs, the native Go
verifier is used. This behavior can be disabled with
`GODEBUG=x509sslcertoverrideplatform=0`.

#### [`crypto/x509/pkix`](/pkg/crypto/x509/pkix/)

[`RDNSequence.String`](/pkg/crypto/x509/pkix#RDNSequence.String) (and therefore [`Name.String`](/pkg/crypto/x509/pkix#Name.String)) now renders string-typed
attribute values as strings even when the attribute&rsquo;s OID is unrecognized.
Previously such values were always hex-encoded in their DER form.
See [#33093](/issue/33093).

#### [`database/sql`](/pkg/database/sql/)

The new [`ConvertAssign`](/pkg/database/sql#ConvertAssign) function gives database drivers access
to the type conversions performed by [`Rows.Scan`](/pkg/database/sql#Rows.Scan).

#### [`database/sql/driver`](/pkg/database/sql/driver/)

Drivers may implement the new [`RowsColumnScanner`](/pkg/database/sql/driver#RowsColumnScanner) interface
to scan directly into user-provided destinations.

#### [`go/constant`](/pkg/go/constant/)

The new [`StringLen`](/pkg/go/constant#StringLen) function returns
the length of a string [`Value`](/pkg/go/constant#Value) without
fully constructing the `Value`.

#### [`go/scanner`](/pkg/go/scanner/)

The scanner now allows retrieving the end position of a token via the new [`Scanner.End`](/pkg/go/scanner#Scanner.End) method.

#### [`go/token`](/pkg/go/token/)

[`File`](/pkg/go/token#File) now has a `String` method.

#### [`go/types`](/pkg/go/types/)

The [`Hasher`](/pkg/go/types#Hasher) type is an implementation of
[`maphash.Hasher`](/pkg/maphash#Hasher) for [`Type`](/pkg/go/types#Type)s
that respects the [`Identical`](/pkg/go/types#Identical) equivalence relation, allowing `Types`
to be used in hash tables and similar data structures.
[`HasherIgnoreTags`](/pkg/go/types#HasherIgnoreTags) is the analogous hasher for [`IdenticalIgnoreTags`](/pkg/go/types#IdenticalIgnoreTags).

The `gotypesalias` `GODEBUG` setting (added in [Go 1.22](/doc/godebug#go-122))
has been removed permanently and the package [`go/types`](/pkg/go/types)
now always produces an [`Alias`](/pkg/go/types#Alias) type node for
[alias declarations](/ref/spec#Alias_declarations) irrespective of `GODEBUG` settings.

#### [`hash/maphash`](/pkg/hash/maphash/)

The [`Hasher`](/pkg/hash/maphash#Hasher) interface type defines the contract between values of a
particular type and future hash-based data structures such as hash
tables and Bloom filters; see [#70471](/issue/70471).

The [`ComparableHasher`](/pkg/hash/maphash#ComparableHasher) type provides a
convenient implementation of [`Hasher`](/pkg/hash/maphash#Hasher) for
comparable types where the `Equal` method is defined as `==`.

#### [`math/big`](/pkg/math/big/)

[`Int`](/pkg/math/big#Int) now has a [`Divide`](/pkg/math/big#Int.Divide) method
to compute quotient and remainder of two [`Int`](/pkg/math/big#Int) values.
It supports rounding modes [`Trunc`](/pkg/math/big#Trunc), [`Floor`](/pkg/math/big#Floor),
[`Round`](/pkg/math/big#Round), and [`Ceil`](/pkg/math/big#Ceil).

#### [`math/rand/v2`](/pkg/math/rand/v2/)

[`Rand`](/pkg/math/rand/v2#Rand) now supports a generic method
[`N`](/pkg/math/rand/v2#Rand.N),
matching the behavior of the top-level [`N`](/pkg/math/rand/v2#N) function.

#### [`net`](/pkg/net/)

[`UnixConn`](/pkg/net#UnixConn) read methods now return [`io.EOF`](/pkg/io#EOF) directly instead of wrapping it in [`net.OpError`](/pkg/net#OpError) when the underlying read returns EOF.

#### [`net/http`](/pkg/net/http/)

[`Transport`](/pkg/net/http#Transport) and [`Server`](/pkg/net/http#Server) support TLS ALPN protocol negotiation on
user-provided [`net.Conn`](/pkg/net#Conn) connections which implement a
`ConnectionState() tls.ConnectionState` method.

HTTP/2 server now accepts client priority signals, as defined in RFC 9218,
allowing it to prioritize serving HTTP/2 streams with higher priority. If the
old behavior is preferred, where streams are served in a round-robin manner
regardless of priority, [`Server.DisableClientPriority`](/pkg/net/http#Server.DisableClientPriority) can be set to `true`.

HTTP/1 [`Response.Body`](/pkg/net/http#Response.Body) now automatically drains any unread content upon being
closed, up to a conservative limit, to allow better connection reuse. For most
programs, this change should be a no-op, or result in a performance improvement.
In rare cases, programs that do not benefit from connection reuse might
experience performance degradation if they had been improperly allowing an
excessive amount of idle connections to linger; usually by setting
[`Transport.MaxIdleConns`](/pkg/net/http#Transport.MaxIdleConns) to `0` or using different [`Client`](/pkg/net/http#Client)s for different
requests, thereby bypassing [`Transport.MaxIdleConns`](/pkg/net/http#Transport.MaxIdleConns) limit. In these cases,
setting [`Transport.DisableKeepAlives`](/pkg/net/http#Transport.DisableKeepAlives) to `true` will disable connection reuse.
However, such performance degradation usually indicates improper configuration
or usage of [`Transport`](/pkg/net/http#Transport) or [`Client`](/pkg/net/http#Client) in the first place, and a deeper look would
likely be beneficial.

The new [`Server.MaxHeaderValueCount`](/pkg/net/http#Server.MaxHeaderValueCount)
field allows HTTP servers to control the number of header values that they are
willing to accept. If unset,
[`DefaultMaxHeaderValueCount`](/pkg/net/http#DefaultMaxHeaderValueCount) is
used.

#### [`net/http/httptest`](/pkg/net/http/httptest/)

The new [`NewTestServer`](/pkg/net/http/httptest#NewTestServer) function
creates a [`Server`](/pkg/net/http/httptest#Server) configured to use an in-memory
fake network suitable for use with the [`testing/synctest`](/pkg/testing/synctest) package.

#### [`net/url`](/pkg/net/url/)

The new [`URL.Clone`](/pkg/net/url#URL.Clone) method creates a deep copy of a URL.
The new [`Values.Clone`](/pkg/net/url#Values.Clone) method creates a deep copy of Values.

#### [`runtime/secret`](/pkg/runtime/secret/)

Goroutines that are created while in [secret mode](/pkg/runtime/secret#Do)
will now themselves execute in secret mode.

#### [`strings`](/pkg/strings/)

The new [`CutLast`](/pkg/strings#CutLast) function slices a string
around the last occurrence of a separator.
It can replace and simplify some common uses of [`LastIndex`](/pkg/strings#LastIndex).

#### [`syscall`](/pkg/syscall/)

On Plan 9, the [`Errno`](/pkg/syscall#Errno) type is now defined and
implements the `error` interface, as on other platforms. Plan 9 system
calls return `ErrorString` values, so `Errno` is never returned by this
package on Plan 9. It is defined so that portable code referring to
`syscall.Errno` builds on Plan 9 without build constraints.

#### [`testing/synctest`](/pkg/testing/synctest/)

The new [`Sleep`](/pkg/testing/synctest#Sleep) helper function combines [`time.Sleep`](/pkg/time#Sleep) and [`synctest.Wait`](/pkg/testing/synctest#Wait).

#### [`unicode`](/pkg/unicode/)

The unicode package and associated support throughout the system have been upgraded from Unicode 15 to Unicode 17.
See the [Unicode 16.0.0](https://www.unicode.org/versions/Unicode16.0.0/) and
[Unicode 17.0.0](https://www.unicode.org/versions/Unicode17.0.0/)
release notes for information about the changes.

## Ports

### Darwin

As [announced](go1.26#darwin) in the Go 1.26 release notes,
Go 1.27 requires macOS 13 Ventura or later;
support for previous versions has been discontinued.

### PowerPC

On the big-endian 64-bit PowerPC port on Linux (`GOOS=linux` `GOARCH=ppc64`),
the Go toolchain now generates binaries that use the ELFv2
system ABI.
ELFv2 support requires Linux kernel 3.13 or later.
RHEL7 backported this support to its 3.10 kernel.

Cgo, position-independent executables (PIE), and external linking
are now supported.
Using these features requires an ELFv2 compatible runtime
(libc and all linked and loaded libraries).

For programs that do not use cgo, the Go toolchain still
generates static binaries with internal linking by default.
For programs that have cgo options, if a static, pure-Go binary
is needed, one can set the environment variable `CGO_ENABLED=0`
when running `go build`.

## Patch Release Changelogs

### go1.27.1

```text
- `ea06a67` [release-branch.go1.27] database/sql: document ability to scan into uuid.UUID (2026-08-21)
- `e696678` [release-branch.go1.27] cmd/compile/internal/noder: fix OMETHEXPR construction for generic methods (2026-08-25)
- `55f10fc` [release-branch.go1.27] encoding/json: preserve error from io.Reader in Decoder.Token (2026-08-21)
- `bf05097` [release-branch.go1.27] crypto/tls: fix client handshake tests hardcoding TLSv1.0 (2026-07-23)
- `a934395` [release-branch.go1.27] simd: fix emulated ShiftAll/RotateAll method signatures (2026-08-25)
- `24681db` [release-branch.go1.27] simd/archsimd: added PMULL gate on carryless multiply test (2026-08-24)
- `0d14aa5` [release-branch.go1.27] encoding/json: fix regression unmarshaling quoted null into string (2026-08-24)
- `45c5cce` [release-branch.go1.27] net/http: clear io.EOF when Close drains a server request body (2026-08-20)
- `71b3c77` [release-branch.go1.27] encoding/json/v2: update documentation about empty structs (2026-08-21)
- `923f8ef` [release-branch.go1.27] cmd: vendor changes for "go fix" (2026-08-26)
- `0929039` [release-branch.go1.27] debug/elf: use addend when applying ppc relocations (2026-08-25)
- `ae786d7` [release-branch.go1.27] cmd/compile: do not use float min max on s390x (2026-08-25)
- `12affc4` [release-branch.go1.27] cmd/compile: fix incorrect qualifying generic methods with alias receiver (2026-08-28)
- `843a92e` [release-branch.go1.27] database/sql: fix closingMutex writer coordination (2026-08-24)
- `862c888` [release-branch.go1.27] go1.27.1 (2026-09-01)
```

### go1.27.2

```text
- `113a7a0` [release-branch.go1.27] crypto/mlkem: fix FIPS 130-3 CAST (2026-08-27)
- `33c04bd` [release-branch.go1.27] runtime: disable mutex profile during async preemption on windows (2026-08-21)
- `2ee6421` [release-branch.go1.27] runtime: netpoll: fix 32-bit little-endian eventfd type confusion (2026-08-23)
- `1e44a4c` [release-branch.go1.27] runtime: don't add the itabs of a plugin twice (2026-09-01)
- `072330d` [release-branch.go1.27] cmd/compile: correctly mark interface calls as used for tail call wrappers (2026-09-04)
- `f8a45f6` [release-branch.go1.27] cmd/compile: fix large riscv64 move/zero (2026-08-31)
- `1cfd8f0` [release-branch.go1.27] encoding/json: avoid calling MarshalText on named string map keys (2026-09-04)
- `aa6fc25` [release-branch.go1.27] net/http: fix HTTP/1 deadlock on concurrent response body Read and Close (2026-09-09)
- `db53142` [release-branch.go1.27] net/http: read Response before signaling eofc in readLoop (2026-09-13)
- `8cd5a09` [release-branch.go1.27] net/http: return nil from Body.Close after a failed Read (2026-09-13)
- `2e4b38e` [release-branch.go1.27] net/http: discard trailer when draining a response body (2026-09-14)
- `072779d` [release-branch.go1.27] net/http/internal/http2: delete malformed framing-related headers (2026-09-17)
- `ff5898f` [release-branch.go1.27] internal/runtime/syscall/linux: sync EpollEvent with the kernel header (2026-08-20)
- `c407a15` [release-branch.go1.27] cmd/compile: don't stack-allocate a slice whose element interior escapes (2026-09-21)
- `e89d44f` [release-branch.go1.27] os: handle unsupported OBJ_DONT_REPARSE on older Windows (2026-09-07)
- `d16d95c` [release-branch.go1.27] compress/flate: do not emit the preset dictionary into the output (2026-07-23)
- `620e2e0` [release-branch.go1.27] runtime: keep the working directory for non-bundled ios/arm64 binaries (2026-09-10)
- `aee72ef` [release-branch.go1.27] testing: escape framing markers only in test2json mode (2026-09-17)
- `fa3bd01` [release-branch.go1.27] cmd/compile: call racefuncexit before a tail call (2026-09-14)
- `bd8f7a6` [release-branch.go1.27] cmd: vendor changes for "go fix" (2026-09-23)
- `359bec4` [release-branch.go1.27] cmd/compile: support unified V5 export data (method indices) (2026-09-01)
- `4e4eec7` [release-branch.go1.27] os: preserve completion notification modes for NewFile handles (2026-09-07)
- `bc516c2` [release-branch.go1.27] cmd/go: include VetxOnly in the vet action ID (2026-09-26)
- `322f178` [release-branch.go1.27] encoding/json/v2: avoid setting with errors parsing float (2026-08-24)
- `d995fb8` [release-branch.go1.27] net/http: populate Request.TLS from HTTP/2 connection state (2026-09-08)
- `8ada93d` [release-branch.go1.27] encoding/json: fix UnsupportedValueError.Value regression (2026-08-29)
- `b73c143` [release-branch.go1.27] cmd/compile: use a 32-bit type for rewritten rotate amounts (2026-09-23)
- `9c26a3a` [release-branch.go1.27] cmd/internal/obj/ppc64: fix async preempt safepoint bug (2026-08-25)
- `87b06e2` [release-branch.go1.27] runtime: create Windows threads on the system stack (2026-09-28)
- `ce0ff94` [release-branch.go1.27] cmd/cover: preserve statement counts when splitting coverage blocks (2026-08-21)
- `bd9992d` [release-branch.go1.27] cmd/link: pass -no_fixup_chains to C linker when building plugin on macOS (2026-09-30)
- `b26d150` [release-branch.go1.27] html/template: reset JS context for template expressions (2026-09-28)
- `6e2cfab` [release-branch.go1.27] html/template: recognize `yield` as regexp preceder keyword (2026-09-28)
- `1e4f1fe` [release-branch.go1.27] cmd/go: bundle go.sum h1 hashes for golang.org/fips140 (2026-09-28)
- `68fa769` [release-branch.go1.27] cmd/go: always verify toolchain downloads with sumdb (2026-09-28)
- `f022e61` [release-branch.go1.27] crypto/tls: validate ECH outer extension references (2026-09-28)
- `7c79b50` [release-branch.go1.27] net/textproto: enforce MIME memory limit on first line (2026-09-22)
- `76875df` [release-branch.go1.27] net/http, net/http/httputil: avoid post-CONNECT pool corruption (2026-08-14)
- `e92229e` [release-branch.go1.27] net/http: avoid server reuse of connection after CONNECT (2026-08-14)
- `9636ab0` [release-branch.go1.27] os: correct Root junction handling on Windows (avoid mkdir escape) (2026-09-15)
- `ababc3c` [release-branch.go1.27] net/http: limit the size of range headers parsed in ServeContent (2026-09-14)
- `1d58cda` [release-branch.go1.27] net/http/internal/http2: avoid excessive work on initial window changes (2026-09-16)
- `31dc9e2` [release-branch.go1.27] net/http/internal/http2: avoid 2x refund on stream close with buffered data (2026-08-28)
- `12acaf3` [release-branch.go1.27] net/http/internal/http2: prevent crash due to HPACK encoder race (2026-09-28)
- `cbb0fb8` [release-branch.go1.27] net/http: apply header limits to Trailer headers too (2026-09-21)
- `022c863` [release-branch.go1.27] go1.27.2 (2026-10-08)
```

### go1.27.3

```text
- `90617fb` [release-branch.go1.27] crypto/x509: distrust a handful of intermediates (2026-10-19)
- `2aae100` [release-branch.go1.27] go1.27.3 (2026-10-20)
```
