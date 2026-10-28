# Go 1.25 Release Notes

## Introduction to Go 1.25

The latest Go release, version 1.25, arrives in [August 2025](/doc/devel/release#go1.25.0), six months after [Go 1.24](/doc/go1.24).
Most of its changes are in the implementation of the toolchain, runtime, and libraries.
As always, the release maintains the Go 1 promise of compatibility.
We expect almost all Go programs to continue to compile and run as before.

## Changes to the language

There are no languages changes that affect Go programs in Go 1.25.
However, in the [language specification](/ref/spec) the notion of core types
has been removed in favor of dedicated prose.
See the respective [blog post](/blog/coretypes) for more information.

## Tools

### Go command

The `go build` `-asan` option now defaults to doing leak detection at
program exit.
This will report an error if memory allocated by C is not freed and is
not referenced by any other memory allocated by either C or Go.
These new error reports may be disabled by setting
`ASAN_OPTIONS=detect_leaks=0` in the environment when running the
program.

The Go distribution will include fewer prebuilt tool binaries. Core
toolchain binaries such as the compiler and linker will still be
included, but tools not invoked by build or test operations will be built
and run by `go tool` as needed.

The new `go.mod` `ignore` [directive](/ref/mod#go-mod-file-ignore) can be used to
specify directories the `go` command should ignore. Files in these directories
and their subdirectories will be ignored by the `go` command when matching package
patterns, such as `all` or `./...`, but will still be included in module zip files.

The new `go doc` `-http` option will start a documentation server showing
documentation for the requested object, and open the documentation in a browser
window.

The new `go version -m -json` option will print the JSON encodings of the
`runtime/debug.BuildInfo` structures embedded in the given Go binary files.

The `go` command now supports using a subdirectory of a repository as the
path for a module root, when [resolving a module path](/ref/mod#vcs-find) using the syntax
`<meta name="go-import" content="root-path vcs repo-url subdir">` to indicate
that the `root-path` corresponds to the `subdir` of the `repo-url` with
version control system `vcs`.

The new `work` package pattern matches all packages in the work (formerly called main)
modules: either the single work module in module mode or the set of workspace modules
in workspace mode.

When the go command updates the `go` line in a `go.mod` or `go.work` file,
it [no longer](/ref/mod#go-mod-file-toolchain) adds a toolchain line
specifying the command&rsquo;s current version.

### Vet

The `go vet` command includes new analyzers:

[waitgroup](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/waitgroup),
which reports misplaced calls to [`sync.WaitGroup.Add`](/pkg/sync#WaitGroup.Add); and

- [hostport](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/hostport),
which reports uses of `fmt.Sprintf("%s:%d", host, port)` to
construct addresses for [`net.Dial`](/pkg/net#Dial), as these will not work with
IPv6; instead it suggests using [`net.JoinHostPort`](/pkg/net#JoinHostPort).

## Runtime

### Container-aware `GOMAXPROCS`

The default behavior of the `GOMAXPROCS` has changed. In prior versions of Go,
`GOMAXPROCS` defaults to the number of logical CPUs available at startup
([`runtime.NumCPU`](/pkg/runtime#NumCPU)). Go 1.25 introduces two changes:

-

On Linux, the runtime considers the CPU bandwidth limit of the cgroup
containing the process, if any. If the CPU bandwidth limit is lower than the
number of logical CPUs available, `GOMAXPROCS` will default to the lower
limit. In container runtime systems like Kubernetes, cgroup CPU bandwidth
limits generally correspond to the &ldquo;CPU limit&rdquo; option. The Go runtime does
not consider the &ldquo;CPU requests&rdquo; option.

-

On all OSes, the runtime periodically updates `GOMAXPROCS` if the number
of logical CPUs available or the cgroup CPU bandwidth limit change.

Both of these behaviors are automatically disabled if `GOMAXPROCS` is set
manually via the `GOMAXPROCS` environment variable or a call to
[`runtime.GOMAXPROCS`](/pkg/runtime#GOMAXPROCS). They can also be disabled explicitly with the [GODEBUG
settings](/doc/godebug) `containermaxprocs=0` and `updatemaxprocs=0`,
respectively.

In order to support reading updated cgroup limits, the runtime will keep cached
file descriptors for the cgroup files for the duration of the process lifetime.

### New experimental garbage collector

A new garbage collector is now available as an experiment. This garbage
collector&rsquo;s design improves the performance of marking and scanning small objects
through better locality and CPU scalability. Benchmark result vary, but we expect
somewhere between a 10—40% reduction in garbage collection overhead in real-world
programs that heavily use the garbage collector.

The new garbage collector may be enabled by setting `GOEXPERIMENT=greenteagc`
at build time. We expect the design to continue to evolve and improve. To that
end, we encourage Go developers to try it out and report back their experiences.
See the [GitHub issue](/issue/73581) for more details on the design and
instructions for sharing feedback.

### Trace flight recorder

[Runtime execution traces](/pkg/runtime/trace) have long provided a powerful,
but expensive way to understand and debug the low-level behavior of an
application. Unfortunately, because of their size and the cost of continuously
writing an execution trace, they were generally impractical for debugging rare
events.

The new [`runtime/trace.FlightRecorder`](/pkg/runtime/trace#FlightRecorder) API
provides a lightweight way to capture a runtime execution trace by continuously
recording the trace into an in-memory ring buffer. When a significant event
occurs, a program can call
[`FlightRecorder.WriteTo`](/pkg/runtime/trace#FlightRecorder.WriteTo) to
snapshot the last few seconds of the trace to a file. This approach produces a
much smaller trace by enabling applications to capture only the traces that
matter.

The length of time and amount of data captured by a
[`FlightRecorder`](/pkg/runtime/trace#FlightRecorder) may be configured within
the [`FlightRecorderConfig`](/pkg/runtime/trace#FlightRecorderConfig).

### Change to unhandled panic output

The message printed when a program exits due to an unhandled panic
that was recovered and repanicked no longer repeats the text of
the panic value.

Previously, a program which panicked with `panic("PANIC")`,
recovered the panic, and then repanicked with the original
value would print:

`panic: PANIC [recovered]
panic: PANIC
`
This program will now print:

`panic: PANIC [recovered, repanicked]
`

### VMA names on Linux

On Linux systems with kernel support for anonymous virtual memory area (VMA) names
(`CONFIG_ANON_VMA_NAME`), the Go runtime will annotate anonymous memory
mappings with context about their purpose. e.g., `[anon: Go: heap]` for heap
memory. This can be disabled with the [GODEBUG setting](/doc/godebug)
`decoratemappings=0`.

## Compiler

### `nil` pointer bug

This release fixes a [compiler bug](/issue/72860), introduced in Go 1.21, that
could incorrectly delay nil pointer checks. Programs like the following, which
used to execute successfully (incorrectly), will now (correctly) panic with a
nil-pointer exception:

`package main

import "os"

func main() {
f, err := os.Open("nonExistentFile")
name := f.Name()
if err != nil {
return
}
println(name)
}
`
This program is incorrect because it uses the result of `os.Open` before
checking the error. If `err` is non-nil, then the `f` result may be nil, in
which case `f.Name()` should panic. However, in Go versions 1.21 through 1.24,
the compiler incorrectly delayed the nil check until after* the error check,
causing the program to execute successfully, in violation of the Go spec. In Go
1.25, it will no longer run successfully. If this change is affecting your code,
the solution is to put the non-nil error check earlier in your code, preferably
immediately after the error-generating statement.

### DWARF5 support

The compiler and linker in Go 1.25 now generate debug information
using [DWARF version 5](https://dwarfstd.org/dwarf5std.html). The
newer DWARF version reduces the space required for debugging
information in Go binaries, and reduces the time for linking,
especially for large Go binaries.
DWARF 5 generation can be disabled by setting the environment
variable `GOEXPERIMENT=nodwarf5` at build time
(this fallback may be removed in a future Go release).

### Faster slices

The compiler can now allocate the backing store for slices on the
stack in more situations, which improves performance. This change has
the potential to amplify the effects of incorrect
[unsafe.Pointer](/pkg/unsafe#Pointer) usage, see for example [issue
73199](/issue/73199). In order to track down these problems, the
[bisect tool](https://pkg.go.dev/golang.org/x/tools/cmd/bisect) can be
used to find the allocation causing trouble using the
`-compile=variablemake` flag. All such new stack allocations can also
be turned off using `-gcflags=all=-d=variablemakehash=n`.

## Linker

The linker now accepts a `-funcalign=N` command line option, which
specifies the alignment of function entries.
The default value is platform-dependent, and is unchanged in this
release.

## Standard library

### New testing/synctest package

The new [`testing/synctest`](/pkg/testing/synctest) package
provides support for testing concurrent code.

The [`Test`](/pkg/testing/synctest#Test) function runs a test function in an isolated
&ldquo;bubble&rdquo;. Within the bubble, time is virtualized: [`time`](/pkg/time) package
functions operate on a fake clock and the clock moves forward instantaneously if
all goroutines in the bubble are blocked.

The [`Wait`](/pkg/testing/synctest#Wait) function waits for all goroutines in the
current bubble to block.

This package was first available in Go 1.24 under `GOEXPERIMENT=synctest`, with
a slightly different API. The experiment has now graduated to general
availability. The old API is still present if `GOEXPERIMENT=synctest` is set,
but will be removed in Go 1.26.

### New experimental encoding/json/v2 package

Go 1.25 includes a new, experimental JSON implementation,
which can be enabled by setting the environment variable
`GOEXPERIMENT=jsonv2` at build time.

When enabled, two new packages are available:

- The [`encoding/json/v2`](/pkg/encoding/json/v2) package is
a major revision of the `encoding/json` package.

- The [`encoding/json/jsontext`](/pkg/encoding/json/jsontext) package
provides lower-level processing of JSON syntax.

In addition, when the &ldquo;jsonv2&rdquo; GOEXPERIMENT is enabled:

- The [`encoding/json`](/pkg/encoding/json) package
uses the new JSON implementation.
Marshaling and unmarshaling behavior is unaffected,
but the text of errors returned by package function may change.

- The [`encoding/json`](/pkg/encoding/json) package contains
a number of new options which may be used
to configure the marshaler and unmarshaler.

The new implementation performs substantially better than
the existing one under many scenarios. In general,
encoding performance is at parity between the implementations
and decoding is substantially faster in the new one.
See the [github.com/go-json-experiment/jsonbench](https://github.com/go-json-experiment/jsonbench)
repository for more detailed analysis.

See the [proposal issue](/issue/71497) for more details.

We encourage users of [`encoding/json`](/pkg/encoding/json) to test
their programs with `GOEXPERIMENT=jsonv2` enabled to help detect
any compatibility issues with the new implementation.

We expect the design of [`encoding/json/v2`](/pkg/encoding/json/v2)
to continue to evolve. We encourage developers to try out the new
API and provide feedback on the [proposal issue](/issue/71497).

### Minor changes to the library

#### [`archive/tar`](/pkg/archive/tar/)

The [`Writer.AddFS`](/pkg/archive/tar#Writer.AddFS) implementation now supports symbolic links
for filesystems that implement [`io/fs.ReadLinkFS`](/pkg/io/fs#ReadLinkFS).

#### [`encoding/asn1`](/pkg/encoding/asn1/)

[`Unmarshal`](/pkg/encoding/asn1#Unmarshal) and [`UnmarshalWithParams`](/pkg/encoding/asn1#UnmarshalWithParams)
now parse the ASN.1 types T61String and BMPString more consistently. This may
result in some previously accepted malformed encodings now being rejected.

#### [`crypto`](/pkg/crypto/)

[`MessageSigner`](/pkg/crypto#MessageSigner) is a new signing interface that can
be implemented by signers that wish to hash the message to be signed themselves.
A new function is also introduced, [`SignMessage`](/pkg/crypto#SignMessage),
which attempts to upgrade a [`Signer`](/pkg/crypto#Signer) interface to
[`MessageSigner`](/pkg/crypto#MessageSigner), using the
[`MessageSigner.SignMessage`](/pkg/crypto#MessageSigner.SignMessage) method if
successful, and [`Signer.Sign`](/pkg/crypto#Signer.Sign) if not. This can be
used when code wishes to support both [`Signer`](/pkg/crypto#Signer) and
[`MessageSigner`](/pkg/crypto#MessageSigner).

Changing the `fips140` [GODEBUG setting](/doc/godebug) after the program has started is now a no-op.
Previously, it was documented as not allowed, and could cause a panic if changed.

SHA-1, SHA-256, and SHA-512 are now slower on amd64 when AVX2 instructions are not available.
All server processors (and most others) produced since 2015 support AVX2.

#### [`crypto/ecdsa`](/pkg/crypto/ecdsa/)

The new [`ParseRawPrivateKey`](/pkg/crypto/ecdsa#ParseRawPrivateKey),
[`ParseUncompressedPublicKey`](/pkg/crypto/ecdsa#ParseUncompressedPublicKey),
[`PrivateKey.Bytes`](/pkg/crypto/ecdsa#PrivateKey.Bytes), and
[`PublicKey.Bytes`](/pkg/crypto/ecdsa#PublicKey.Bytes) functions and methods
implement low-level encodings, replacing the need to use
[`crypto/elliptic`](/pkg/crypto/elliptic) or [`math/big`](/pkg/math/big)
functions and methods.

When FIPS 140-3 mode is enabled, signing is now four times faster, matching the
performance of non-FIPS mode.

#### [`crypto/ed25519`](/pkg/crypto/ed25519/)

When FIPS 140-3 mode is enabled, signing is now four times faster, matching the
performance of non-FIPS mode.

#### [`crypto/elliptic`](/pkg/crypto/elliptic/)

The hidden and undocumented `Inverse` and `CombinedMult` methods on some
[`Curve`](/pkg/crypto/elliptic#Curve) implementations have been removed.

#### [`crypto/rsa`](/pkg/crypto/rsa/)

[`PublicKey`](/pkg/crypto/rsa#PublicKey) no longer claims that the modulus value
is treated as secret. [`VerifyPKCS1v15`](/pkg/crypto/rsa#VerifyPKCS1v15) and
[`VerifyPSS`](/pkg/crypto/rsa#VerifyPSS) already warned that all inputs are
public and could be leaked, and there are mathematical attacks that can recover
the modulus from other public values.

Key generation is now three times faster.

#### [`crypto/sha1`](/pkg/crypto/sha1/)

Hashing is now two times faster on amd64 when SHA-NI instructions are available.

#### [`crypto/sha3`](/pkg/crypto/sha3/)

The new [`SHA3.Clone`](/pkg/crypto/sha3#SHA3.Clone) method implements [`hash.Cloner`](/pkg/hash#Cloner).

Hashing is now two times faster on Apple M processors.

#### [`crypto/tls`](/pkg/crypto/tls/)

The new [`ConnectionState.CurveID`](/pkg/crypto/tls#ConnectionState.CurveID)
field exposes the key exchange mechanism used to establish the connection.

The new [`Config.GetEncryptedClientHelloKeys`](/pkg/crypto/tls#Config.GetEncryptedClientHelloKeys)
callback can be used to set the [`EncryptedClientHelloKey`](/pkg/crypto/tls#EncryptedClientHelloKey)s
for a server to use when a client sends an Encrypted Client Hello extension.

SHA-1 signature algorithms are now disallowed in TLS 1.2 handshakes, per
[RFC 9155](https://www.rfc-editor.org/rfc/rfc9155.html).
They can be re-enabled with the [GODEBUG setting](/doc/godebug) `tlssha1=1`.

When [FIPS 140-3 mode](/doc/security/fips140) is enabled, Extended Master Secret
is now required in TLS 1.2, and Ed25519 and X25519MLKEM768 are now allowed.

TLS servers now prefer the highest supported protocol version, even if it isn&rsquo;t
the client&rsquo;s most preferred protocol version.

Both TLS clients and servers are now stricter in following the specifications
and in rejecting off-spec behavior. Connections with compliant peers should be
unaffected.

#### [`crypto/x509`](/pkg/crypto/x509/)

[`CreateCertificate`](/pkg/crypto/x509#CreateCertificate),
[`CreateCertificateRequest`](/pkg/crypto/x509#CreateCertificateRequest), and
[`CreateRevocationList`](/pkg/crypto/x509#CreateRevocationList) can now accept a
[`crypto.MessageSigner`](/pkg/crypto#MessageSigner) signing interface as well as
[`crypto.Signer`](/pkg/crypto#Signer). This allows these functions to use
signers which implement &ldquo;one-shot&rdquo; signing interfaces, where hashing is done as
part of the signing operation, instead of by the caller.

[`CreateCertificate`](/pkg/crypto/x509#CreateCertificate) now uses truncated
SHA-256 to populate the `SubjectKeyId` if it is missing.
The [GODEBUG setting](/doc/godebug) `x509sha256skid=0` reverts to SHA-1.

[`ParseCertificate`](/pkg/crypto/x509#ParseCertificate) now rejects certificates
which contain a BasicConstraints extension that contains a negative pathLenConstraint.

[`ParseCertificate`](/pkg/crypto/x509#ParseCertificate) now handles strings encoded
with the ASN.1 T61String and BMPString types more consistently. This may result in
some previously accepted malformed encodings now being rejected.

#### [`debug/elf`](/pkg/debug/elf/)

The [`debug/elf`](/pkg/debug/elf) package adds two new constants:

- [`PT_RISCV_ATTRIBUTES`](/pkg/debug/elf#PT_RISCV_ATTRIBUTES)

- [`SHT_RISCV_ATTRIBUTES`](/pkg/debug/elf#SHT_RISCV_ATTRIBUTES)
for RISC-V ELF parsing.

#### [`go/ast`](/pkg/go/ast/)

The [`FilterPackage`](/pkg/ast#FilterPackage), [`PackageExports`](/pkg/ast#PackageExports), and
[`MergePackageFiles`](/pkg/ast#MergePackageFiles) functions, and the [`MergeMode`](/pkg/go/ast#MergeMode) type and its
constants, are all deprecated, as they are for use only with the
long-deprecated [`Object`](/pkg/ast#Object) and [`Package`](/pkg/ast#Package) machinery.

The new [`PreorderStack`](/pkg/go/ast#PreorderStack) function, like [`Inspect`](/pkg/go/ast#Inspect), traverses a syntax
tree and provides control over descent into subtrees, but as a
convenience it also provides the stack of enclosing nodes at each
point.

#### [`go/parser`](/pkg/go/parser/)

The [`ParseDir`](/pkg/go/parser#ParseDir) function is deprecated.

#### [`go/token`](/pkg/go/token/)

The new [`FileSet.AddExistingFiles`](/pkg/go/token#FileSet.AddExistingFiles) method enables existing
[`File`](/pkg/go/token#File)s to be added to a [`FileSet`](/pkg/go/token#FileSet),
or a [`FileSet`](/pkg/go/token#FileSet) to be constructed for an arbitrary
set of [`File`](/pkg/go/token#File)s, alleviating the problems associated with a single global
[`FileSet`](/pkg/go/token#FileSet) in long-lived applications.

#### [`go/types`](/pkg/go/types/)

[`Var`](/pkg/go/types#Var) now has a [`Var.Kind`](/pkg/go/types#Var.Kind) method that classifies the variable as one
of: package-level, receiver, parameter, result, local variable, or
a struct field.

The new [`LookupSelection`](/pkg/go/types#LookupSelection) function looks up the field or method of a
given name and receiver type, like the existing [`LookupFieldOrMethod`](/pkg/go/types#LookupFieldOrMethod)
function, but returns the result in the form of a [`Selection`](/pkg/go/types#Selection).

#### [`hash`](/pkg/hash/)

The new [`XOF`](/pkg/hash#XOF) interface can be implemented by &ldquo;extendable output
functions&rdquo;, which are hash functions with arbitrary or unlimited output length
such as [SHAKE](/pkg/crypto/sha3#SHAKE).

Hashes implementing the new [`Cloner`](/pkg/hash#Cloner) interface can return a copy of their state.
All standard library [`Hash`](/pkg/hash#Hash) implementations now implement [`Cloner`](/pkg/hash#Cloner).

#### [`hash/maphash`](/pkg/hash/maphash/)

The new [`Hash.Clone`](/pkg/hash/maphash#Hash.Clone) method implements [`hash.Cloner`](/pkg/hash#Cloner).

#### [`io/fs`](/pkg/io/fs/)

A new [`ReadLinkFS`](/pkg/io/fs#ReadLinkFS) interface provides the ability to read symbolic links in a filesystem.

#### [`log/slog`](/pkg/log/slog/)

[`GroupAttrs`](/pkg/log/slog#GroupAttrs) creates a group [`Attr`](/pkg/log/slog#Attr) from a slice of [`Attr`](/pkg/log/slog#Attr) values.

[`Record`](/pkg/log/slog#Record) now has a [`Source`](/pkg/log/slog#Record.Source) method,
returning its source location or nil if unavailable.

#### [`mime/multipart`](/pkg/mime/multipart/)

The new helper function [`FileContentDisposition`](/pkg/mime/multipart#FileContentDisposition) builds multipart
Content-Disposition header fields.

#### [`net`](/pkg/net/)

[`LookupMX`](/pkg/net#LookupMX) and [`Resolver.LookupMX`](/pkg/net#Resolver.LookupMX) now return DNS names that look
like valid IP address, as well as valid domain names.
Previously if a name server returned an IP address as a DNS name,
[`LookupMX`](/pkg/net#LookupMX) would discard it, as required by the RFCs.
However, name servers in practice do sometimes return IP addresses.

On Windows, [`ListenMulticastUDP`](/pkg/net#ListenMulticastUDP) now supports IPv6 addresses.

On Windows, it is now possible to convert between an [`os.File`](/pkg/os#File)
and a network connection. Specifically, the [`FileConn`](/pkg/net#FileConn),
[`FilePacketConn`](/pkg/net#FilePacketConn), and
[`FileListener`](/pkg/net#FileListener) functions are now implemented, and
return a network connection or listener corresponding to an open file.
Similarly, the `File` methods of [`TCPConn`](/pkg/net#TCPConn.File),
[`UDPConn`](/pkg/net#UDPConn.File), [`UnixConn`](/pkg/net#UnixConn.File),
[`IPConn`](/pkg/net#IPConn.File), [`TCPListener`](/pkg/net#TCPListener.File),
and [`UnixListener`](/pkg/net#UnixListener.File) are now implemented, and return
the underlying [`os.File`](/pkg/os#File) of a network connection.

#### [`net/http`](/pkg/net/http/)

The new [`CrossOriginProtection`](/pkg/net/http#CrossOriginProtection) implements protections against [Cross-Site
Request Forgery (CSRF)](https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/CSRF) by rejecting non-safe cross-origin browser requests.
It uses [modern browser Fetch metadata](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Sec-Fetch-Site), doesn&rsquo;t require tokens
or cookies, and supports origin-based and pattern-based bypasses.

#### [`os`](/pkg/os/)

On Windows, [`NewFile`](/pkg/os#NewFile) now supports handles opened for asynchronous I/O (that is,
[`syscall.FILE_FLAG_OVERLAPPED`](/pkg/syscall#FILE_FLAG_OVERLAPPED) is specified in the [`syscall.CreateFile`](/pkg/syscall#CreateFile) call).
These handles are associated with the Go runtime&rsquo;s I/O completion port,
which provides the following benefits for the resulting [`File`](/pkg/os#File):

- I/O methods ([`File.Read`](/pkg/os#File.Read), [`File.Write`](/pkg/os#File.Write), [`File.ReadAt`](/pkg/os#File.ReadAt), and [`File.WriteAt`](/pkg/os#File.WriteAt)) do not block an OS thread.

- Deadline methods ([`File.SetDeadline`](/pkg/os#File.SetDeadline), [`File.SetReadDeadline`](/pkg/os#File.SetReadDeadline), and [`File.SetWriteDeadline`](/pkg/os#File.SetWriteDeadline)) are supported.

This enhancement is especially beneficial for applications that communicate via named pipes on Windows.

Note that a handle can only be associated with one completion port at a time.
If the handle provided to [`NewFile`](/pkg/os#NewFile) is already associated with a completion port,
the returned [`File`](/pkg/os#File) is downgraded to synchronous I/O mode.
In this case, I/O methods will block an OS thread, and the deadline methods have no effect.

The filesystems returned by [`DirFS`](/pkg/os#DirFS) and [`Root.FS`](/pkg/os#Root.FS) implement the new [`io/fs.ReadLinkFS`](/pkg/io/fs#ReadLinkFS) interface.
[`CopyFS`](/pkg/os#CopyFS) supports symlinks when copying filesystems that implement [`io/fs.ReadLinkFS`](/pkg/io/fs#ReadLinkFS).

The [`Root`](/pkg/os#Root) type supports the following additional methods:

- [`Root.Chmod`](/pkg/os#Root.Chmod)

- [`Root.Chown`](/pkg/os#Root.Chown)

- [`Root.Chtimes`](/pkg/os#Root.Chtimes)

- [`Root.Lchown`](/pkg/os#Root.Lchown)

- [`Root.Link`](/pkg/os#Root.Link)

- [`Root.MkdirAll`](/pkg/os#Root.MkdirAll)

- [`Root.ReadFile`](/pkg/os#Root.ReadFile)

- [`Root.Readlink`](/pkg/os#Root.Readlink)

- [`Root.RemoveAll`](/pkg/os#Root.RemoveAll)

- [`Root.Rename`](/pkg/os#Root.Rename)

- [`Root.Symlink`](/pkg/os#Root.Symlink)

- [`Root.WriteFile`](/pkg/os#Root.WriteFile)

#### [`reflect`](/pkg/reflect/)

The new [`TypeAssert`](/pkg/reflect#TypeAssert) function permits converting a [`Value`](/pkg/reflect#Value) directly to a Go value
of the given type. This is like using a type assertion on the result of [`Value.Interface`](/pkg/reflect#Value.Interface),
but avoids unnecessary memory allocations.

#### [`regexp/syntax`](/pkg/regexp/syntax/)

The `\p{name}` and `\P{name}` character class syntaxes now accept the names
Any, ASCII, Assigned, Cn, and LC, as well as Unicode category aliases like `\p{Letter}` for `\pL`.
Following [Unicode TR18](https://unicode.org/reports/tr18/), they also now use
case-insensitive name lookups, ignoring spaces, underscores, and hyphens.

#### [`runtime`](/pkg/runtime/)

Cleanup functions scheduled by [`AddCleanup`](/pkg/runtime#AddCleanup) are now executed
concurrently and in parallel, making cleanups more viable for heavy
use like the [`unique`](/pkg/unique) package. Note that individual cleanups should
still shunt their work to a new goroutine if they must execute or
block for a long time to avoid blocking the cleanup queue.

A new `GODEBUG=checkfinalizers=1` setting helps find common issues with
finalizers and cleanups, such as those described [in the GC
guide](/doc/gc-guide#Finalizers_cleanups_and_weak_pointers).
In this mode, the runtime runs diagnostics on each garbage collection cycle,
and will also regularly report the finalizer and
cleanup queue lengths to stderr to help identify issues with
long-running finalizers and/or cleanups.
See the [GODEBUG documentation](https://pkg.go.dev/runtime#hdr-Environment_Variables)
for more details.

The new [`SetDefaultGOMAXPROCS`](/pkg/runtime#SetDefaultGOMAXPROCS) function sets `GOMAXPROCS` to the runtime
default value, as if the `GOMAXPROCS` environment variable is not set. This is
useful for enabling the [new `GOMAXPROCS` default](#container-aware-gomaxprocs) if it has been
disabled by the `GOMAXPROCS` environment variable or a prior call to
[`GOMAXPROCS`](/pkg/runtime#GOMAXPROCS).

#### [`runtime/pprof`](/pkg/runtime/pprof/)

The mutex profile for contention on runtime-internal locks now correctly points
to the end of the critical section that caused the delay. This matches the
profile&rsquo;s behavior for contention on `sync.Mutex` values. The
`runtimecontentionstacks` setting for `GODEBUG`, which allowed opting in to the
unusual behavior of Go 1.22 through 1.24 for this part of the profile, is now
gone.

#### [`sync`](/pkg/sync/)

The new [`WaitGroup.Go`](/pkg/sync#WaitGroup.Go) method
makes the common pattern of creating and counting goroutines more convenient.

#### [`testing`](/pkg/testing/)

The new methods [`T.Attr`](/pkg/testing#T.Attr), [`B.Attr`](/pkg/testing#B.Attr), and [`F.Attr`](/pkg/testing#F.Attr) emit an
attribute to the test log. An attribute is an arbitrary
key and value associated with a test.

For example, in a test named `TestF`,
`t.Attr("key", "value")` emits:

`=== ATTR TestF key value
`
With the `-json` flag, attributes appear as a new &ldquo;attr&rdquo; action.

The new [`Output`](/pkg/testing#T.Output) method of [`T`](/pkg/testing#T), [`B`](/pkg/testing#B) and [`F`](/pkg/testing#F) provides an [`io.Writer`](/pkg/io#Writer)
that writes to the same test output stream as [`TB.Log`](/pkg/testing#TB.Log).
Like `TB.Log`, the output is indented, but it does not include the file and line number.

The [`AllocsPerRun`](/pkg/testing#AllocsPerRun) function now panics
if parallel tests are running.
The result of [`AllocsPerRun`](/pkg/testing#AllocsPerRun) is inherently
flaky if other tests are running.
The new panicking behavior helps catch such bugs.

#### [`testing/fstest`](/pkg/testing/fstest/)

[`MapFS`](/pkg/testing/fstest#MapFS) implements the new [`io/fs.ReadLinkFS`](/pkg/io/fs#ReadLinkFS) interface.
[`TestFS`](/pkg/testing/fstest#TestFS) will verify the functionality of the [`io/fs.ReadLinkFS`](/pkg/io/fs#ReadLinkFS) interface if implemented.
[`TestFS`](/pkg/testing/fstest#TestFS) will no longer follow symlinks to avoid unbounded recursion.

#### [`unicode`](/pkg/unicode/)

The new [`CategoryAliases`](/pkg/unicode#CategoryAliases) map provides access to category alias names, such as “Letter” for “L”.

The new categories [`Cn`](/pkg/unicode#Cn) and [`LC`](/pkg/unicode#LC) define unassigned codepoints and cased letters, respectively.
These have always been defined by Unicode but were inadvertently omitted in earlier versions of Go.
The [`C`](/pkg/unicode#C) category now includes [`Cn`](/pkg/unicode#Cn), meaning it has added all unassigned code points.

#### [`unique`](/pkg/unique/)

The [`unique`](/pkg/unique) package now reclaims interned values more eagerly,
more efficiently, and in parallel. As a consequence, applications using
[`Make`](/pkg/unique#Make) are now less likely to experience memory blow-up when lots of
truly unique values are interned.

Values passed to [`Make`](/pkg/unique#Make) containing [`Handle`](/pkg/unique#Handle)s previously required multiple
garbage collection cycles to collect, proportional to the depth of the chain
of [`Handle`](/pkg/unique#Handle) values. Now, once
unused, they are collected promptly in a single cycle.

## Ports

### Darwin

As [announced](/doc/go1.24#darwin) in the Go 1.24 release notes, Go 1.25 requires macOS 12 Monterey or later.
Support for previous versions has been discontinued.

### Windows

Go 1.25 is the last release that contains the [broken](/doc/go1.24#windows) 32-bit windows/arm port (`GOOS=windows` `GOARCH=arm`). It will be removed in Go 1.26.

### AMD64

In `GOAMD64=v3` mode or higher, the compiler will now use fused
multiply-add instructions to make floating-point arithmetic faster and
more accurate. This may change the exact floating-point values that a
program generates.

To avoid fusing use an explicit `float64` cast, like `float64(a*b)+c`.

### Loong64

The linux/loong64 port now supports the race detector, gathering traceback information from C code
using [`runtime.SetCgoTraceback`](/pkg/runtime#SetCgoTraceback), and linking cgo programs with the
internal link mode.

### RISC-V

The linux/riscv64 port now supports the `plugin` build mode.

The `GORISCV64` environment variable now accepts a new value `rva23u64`,
which selects the RVA23U64 user-mode application profile.

## Patch Release Changelogs

### go1.25.1

```text
- `c72fcab` [release-branch.go1.25] cmd/go/internal/gover: fix ModIsPrerelease for toolchain versions (2025-07-29)
- `21ac81c` [release-branch.go1.25] os/exec: fix incorrect expansion of ".." in LookPath on plan9 (2025-08-07)
- `749dff8` [release-branch.go1.25] runtime: make all synctest bubble violations fatal panics (2025-08-14)
- `8995e84` [release-branch.go1.25] internal/poll: set the correct file offset in FD.Seek for Windows overlapped handles (2025-08-19)
- `cdd8cf4` [release-branch.go1.25] net: fix WriteMsgUDPAddrPort addr handling on IPv4 sockets (2025-07-25)
- `b1959cf` [release-branch.go1.25] net/http: require exact match for CrossSiteProtection bypass patterns (2025-08-26)
- `56ebf80` [release-branch.go1.25] go1.25.1 (2025-09-03)
```

### go1.25.2

```text
- `879e3cb` [release-branch.go1.25] runtime: lock mheap_.speciallock when allocating synctest specials (2025-08-26)
- `a86792b` [release-branch.go1.25] net: skip TestIPv4WriteMsgUDPAddrPort on plan9 (2025-08-14)
- `be61132` [release-branch.go1.25] cmd/compile: export to DWARF types only referenced through interfaces (2025-08-18)
- `7d57009` [release-branch.go1.25] os: fix Root.MkdirAll to handle race of directory creation (2025-08-21)
- `f75bcff` [release-branch.go1.25] os: set full name for Roots created with Root.OpenRoot (2025-08-22)
- `57bd28a` [release-branch.go1.25] crypto/internal/fips140/ecdsa: make TestingOnlyNewDRBG generic (2025-09-07)
- `bec452a` [release-branch.go1.25] crypto/internal/fips140: update frozen module version to "v1.0.0" (2025-09-10)
- `90de3b3` [release-branch.go1.25] crypto/internal/fips140: remove key import PCTs, make keygen PCTs fatal (2025-09-05)
- `b816c79` [release-branch.go1.25] lib/fips140: re-seal v1.0.0 (2025-09-07)
- `79c3081` [release-branch.go1.25] internal/poll: don't call Seek for overlapped Windows handles (2025-08-19)
- `f3dc4aa` [release-branch.go1.25] runtime: initialise debug settings much earlier in startup process (2025-09-18)
- `16fdaac` [release-branch.go1.25] sync/atomic: correct Uintptr.Or return doc (2025-09-25)
- `205d086` [release-branch.go1.25] crypto/tls: quote protocols in ALPN error message (2025-09-29)
- `7735dc9` [release-branch.go1.25] cmd/compile: don't rely on loop info when there are irreducible loops (2025-09-23)
- `0b53e41` [release-branch.go1.25] debug/pe: permit symbols with no name (2025-09-01)
- `06993c7` [release-branch.go1.25] context: don't return a non-nil from Err before Done is closed (2025-09-18)
- `28ac8d2` [release-branch.go1.25] net/http: avoid connCount underflow race (2025-09-15)
- `d6f2741` [release-branch.go1.25] spec: update spec date to match release date (2025-10-06)
- `66f6fea` [release-branch.go1.25] spec: revert "update spec date to match release date" (2025-10-06)
- `6a05732` [release-branch.go1.25] net/mail: avoid quadratic behavior in mail address parsing (2025-09-25)
- `930ce22` [release-branch.go1.25] crypto/x509: mitigate DoS vector when intermediate certificate contains DSA public key (2025-09-11)
- `5d7a787` [release-branch.go1.25] net/textproto: avoid quadratic complexity in Reader.ReadResponse (2025-09-30)
- `9fd3ac8` [release-branch.go1.25] net/url: enforce stricter parsing of bracketed IPv6 hostnames (2025-08-29)
- `f0c69db` [release-branch.go1.25] crypto/x509: improve domain name verification (2025-09-15)
- `100c5a6` [release-branch.go1.25] net/http: add httpcookiemaxnum GODEBUG option to limit number of cookies parsed (2025-09-30)
- `e0f655b` [release-branch.go1.25] encoding/asn1: prevent memory exhaustion when parsing using internal/saferio (2025-09-03)
- `90f72bd` [release-branch.go1.25] encoding/pem: make Decode complexity linear (2025-09-30)
- `2612dcf` [release-branch.go1.25] archive/tar: set a limit on the size of GNU sparse file 1.0 regions (2025-09-11)
- `bed6c81` [release-branch.go1.25] go1.25.2 (2025-10-07)
```

### go1.25.3

```text
- `79ec0c9` [release-branch.go1.25] spec: update spec date to match release date (2025-10-06)
- `e05b2c9` [release-branch.go1.25] crypto/x509: rework fix for CVE-2025-58187 (2025-10-09)
- `28622c1` [release-branch.go1.25] go1.25.3 (2025-10-13)
```

### go1.25.4

```text
- `f6db735` [release-branch.go1.25] net/url: allow IP-literals with IPv4-mapped IPv6 addresses (2025-10-09)
- `7e049e5` [release-branch.go1.25] encoding/pem: properly decode strange PEM data (2025-10-15)
- `bbb7627` [release-branch.go1.25] cmd/compile: prevent shapifying of pointer shape type (2025-09-15)
- `bf95b76` [release-branch.go1.25] runtime: use one more address bit for tagged pointers (2025-09-02)
- `cd21a7b` [release-branch.go1.25] Revert "crypto/internal/fips140/subtle: add assembly implementation of xorBytes for mipsx" (2025-10-07)
- `4942c74` [release-branch.go1.25] Revert "crypto/internal/fips140/subtle: add assembly implementation of xorBytes for mips64x" (2025-10-07)
- `8097b19` [release-branch.go1.25] os: support deleting read-only files in RemoveAll on older Windows versions (2025-10-21)
- `5ba37a3` [release-branch.go1.25] cmd/compile: don't optimize away a panicing interface comparison (2025-10-22)
- `83885f3` [release-branch.go1.25] encoding/pem: properly calculate end indexes (2025-10-23)
- `f2cd93a` [release-branch.go1.25] go1.25.4 (2025-11-05)
```

### go1.25.5

```text
- `433c01e` [release-branch.go1.25] internal/syscall/windows: fix ReOpenFile sentinel error value (2025-11-03)
- `e1ce1bf` [release-branch.go1.25] mime: parse media types that contain braces (2025-11-10)
- `287017a` [release-branch.go1.25] crypto/x509: excluded subdomain constraints preclude wildcard SANs (2025-11-24)
- `f7bce4b` [release-branch.go1.25] crypto/x509: prevent HostnameError.Error() from consuming excessive resource (2025-11-24)
- `fefb02a` [release-branch.go1.25] go1.25.5 (2025-12-02)
```

### go1.25.6

```text
- `d5bfdcb` [release-branch.go1.25] crypto/tls: use inner hello for earlyData when using QUIC and ECH (2025-11-17)
- `93f5d1c` [release-branch.go1.25] os,internal/poll: don't call IsNonblock for consoles and Stdin (2025-11-26)
- `ea603ee` [release-branch.go1.25] os: allow direntries to have zero inodes on Linux (2025-11-25)
- `6f07a57` [release-branch.go1.25] runtime/race: set missing argument frame for ppc64x atomic And/Or wrappers (2025-11-06)
- `4e531b2` [release-branch.go1.25] runtime: mark getfp as nosplit (2025-12-08)
- `14f50f6` [release-branch.go1.25] cmd/compile: handle propagating an out-of-range jump table index (2025-12-22)
- `ddcf27f` [release-branch.go1.25] Revert "errors: optimize errors.Join for single unwrappable errors" (2025-12-23)
- `525dd85` [release-branch.go1.25] crypto/tls: reject trailing messages after client/server hello (2025-11-24)
- `4be3852` [release-branch.go1.25] crypto/tls: don't copy auto-rotated session ticket keys in Config.Clone (2026-01-06)
- `082365a` [release-branch.go1.25] cmd/go: update VCS commands to use safer flag/argument syntax (2025-12-10)
- `2526187` [release-branch.go1.25] cmd/go/internal/work: sanitize flags before invoking 'pkg-config' (2025-12-04)
- `afa9b66` [release-branch.go1.25] net/url: add urlmaxqueryparams GODEBUG to limit the number of query parameters (2025-11-03)
- `9d497df` [release-branch.go1.25] archive/zip: reduce CPU usage in index construction (2025-11-05)
- `69801b2` [release-branch.go1.25] go1.25.6 (2026-01-15)
```

### go1.25.7

```text
- `738bc3a` [release-branch.go1.25] cmd/compile: during regalloc, fixedreg values are always available (2025-08-13)
- `b191009` [release-branch.go1.25] cmd/go: remove user-content from doc strings in cgo ASTs. (2026-01-06)
- `0765a9d` [release-branch.go1.25] crypto/x509: fix single label excluded name constraints handling (2026-01-27)
- `6b1110a` [release-branch.go1.25] Revert "crypto/tls: don't copy auto-rotated session ticket keys in Config.Clone" (2026-01-26)
- `c2d04c0` [release-branch.go1.25] crypto/tls: add verifiedChains expiration checking during resumption (2026-01-26)
- `d5987bf` [release-branch.go1.25] crypto/tls: check verifiedChains roots when resuming sessions (2026-01-26)
- `4512014` [release-branch.go1.25] crypto/tls: revalidate whole chain on resumption on Windows and macOS (2026-01-30)
- `c7d189e` [release-branch.go1.25] crypto/tls: document resumption behavior across Configs (2026-01-29)
- `eaf3bc7` [release-branch.go1.25] go1.25.7 (2026-02-04)
```

### go1.25.8

```text
- `0222717` [release-branch.go1.25] cmd/compile: fix mis-compilation for static array initialization (2026-01-21)
- `592530e` [release-branch.go1.25] cmd/go: fix pkg-config flag sanitization (2026-02-03)
- `0c56fa2` [release-branch.go1.25] net/smtp: prevent test failures due to expired test certificate (2026-02-10)
- `92544bb` [release-branch.go1.25] runtime: don't negate eventfd errno (2025-09-09)
- `df7331d` [release-branch.go1.25] all: update x/sys (2026-02-25)
- `9f8fa93` [release-branch.go1.25] os: support deleting inaccessible files in RemoveAll (2026-02-04)
- `0ee4ab4` [release-branch.go1.25] internal/syscall/windows: correct some enums and syscall signatures (2026-02-13)
- `4091800` [release-branch.go1.25] os: avoid escape from Root via ReadDir or Readdir (2026-02-26)
- `d8174a9` [release-branch.go1.25] net/url: reject IPv6 literal not at start of host (2026-01-28)
- `a9db31e` [release-branch.go1.25] html/template: properly escape URLs in meta content attributes (2026-01-09)
- `439ff99` [release-branch.go1.25] go1.25.8 (2026-03-06)
```

### go1.25.9

```text
- `de86eae` [release-branch.go1.25] cmd/cgo/internal/test: use (syntactic) constant for C array bound (2025-08-02)
- `814b540` [release-branch.go1.25] cmd/internal/testdir: unify link command (2025-08-02)
- `b02c84a` [release-branch.go1.25] cmd/internal/testdir: pass -buildid to link command (2025-08-02)
- `19c660d` [release-branch.go1.25] cmd/compile/internal/typecheck: simplify tcSliceHeader (2026-03-03)
- `9a6627a` [release-branch.go1.25] crypto/x509: fix wildcard domain exclusions. (2026-03-02)
- `aed3950` [release-branch.go1.25] cmd/go/internal/cache: update trim timestamp before trimming (2026-03-09)
- `4f81fbf` [release-branch.go1.25] doc: remove template use in godebug.md (2026-03-18)
- `f9532f0` [release-branch.go1.25] test/fixedbugs: remove issue46234 test timeout (2026-02-20)
- `7301bd3` [release-branch.go1.25] cmd/go: skip git sha256 tests if git < 2.29 (2025-08-24)
- `cff64a5` [release-branch.go1.25] cmd/go: tests: rename git-min-vers->git-sha256 (2025-11-14)
- `93a4f03` [release-branch.go1.25] runtime/race: apply LLVM zero-initialization fix (2026-03-18)
- `210b811` [release-branch.go1.25] crypto/x509: hoist policy pruning out of loop (2026-03-23)
- `edc1e4a` [release-branch.go1.25] crypto/x509: fix signature checking limit (2026-03-05)
- `4c79c42` [release-branch.go1.25] internal/syscall/unix: properly support AT_SYMLINK_NOFOLLOW on Linux (2026-03-23)
- `3ed3169` [release-branch.go1.25] html/template: properly track JS template literal brace depth across contexts (2026-03-23)
- `72cc336` [release-branch.go1.25] cmd/compile: fix mem access overlap detection (2026-03-12)
- `7d2dd34` [release-branch.go1.25] cmd/compile: fix loopbce overflow check logic (2026-03-06)
- `82b0cdb` [release-branch.go1.25] archive/tar: limit the number of old GNU sparse format entries (2026-03-23)
- `02f574a` [release-branch.go1.25] crypto/tls: prevent deadlock when client sends multiple key update messages (2026-03-23)
- `abaa0cb` [release-branch.go1.25] cmd/go: disallow cgo trust boundary bypass (2026-02-24)
- `7076e01` [release-branch.go1.25] go1.25.9 (2026-04-07)
```

### go1.25.10

```text
- `f61d5cb` [release-branch.go1.25] cmd/link: use bfd ld 2.36+ on linux/arm64 instead of gold (2026-01-29)
- `f093575` [release-branch.go1.25] fix incorrect loop trip counts (2026-04-14)
- `cb19c94` [release-branch.go1.25] cmd/go: specify full path to go command when running go tool covdata (2026-03-17)
- `ab12e62` [release-branch.go1.25] cmd/compile: handle min integer step in loop (2026-04-13)
- `b8b171a` [release-branch.go1.25] runtime: add sysUnreserve to undo sysReserve (2026-04-02)
- `9024c3f` [release-branch.go1.25] all: update x/net to a9171bc8 (2026-04-17)
- `24eaa90` [release-branch.go1.25] html/template: fix escaper bypass by treating empty script type as JavaScript (2026-04-27)
- `98e009d` [release-branch.go1.25] net/http/httputil: reencode queries with many parameters in proxy (2026-04-24)
- `8546a50` [release-branch.go1.25] cmd/go: use MkdirTemp to create temp directory for "go bug" (2026-04-08)
- `e50bd35` [release-branch.go1.25] go/types, types2: handle unconstrained type parameters correctly in a few places (2026-04-29)
- `0222844` [release-branch.go1.25] lib/fips140: add certified pointing to v1.0.0-c2097c7c (2026-04-27)
- `c0dfc5c` [release-branch.go1.25] net: avoid double-free of cgo pointer when handling large DNS response (2026-03-26)
- `32a7f80` [release-branch.go1.25] crypto/fips140: add package docs (2026-04-28)
- `026f000` [release-branch.go1.25] cmd/compile: fix loopvar version detection with line directives (2026-01-21)
- `9e5f069` [release-branch.go1.25] net/mail: fix quadratic complexity in consumeComment (2026-04-04)
- `9d92298` [release-branch.go1.25] cmd/pack: refuse to extract files with directory components (2026-04-15)
- `f0384f7` [release-branch.go1.25] html/template: fix escaping of URLs in meta content attributes (2026-04-22)
- `cafd419` [release-branch.go1.25] os: avoid panic when RemoveAll fails to remove a symlink (2026-04-06)
- `9027f0c` [release-branch.go1.25] cmd/go: invalidate test cache when -coverpkg dependencies change (2026-04-06)
- `300adf1` [release-branch.go1.25] net/mail: fix quadratic consumePhrase behavior (2026-04-28)
- `278a55c` [release-branch.go1.25] all: avoid unsafe StringToUTF16Ptr on Windows (2026-04-24)
- `b18ca70` [release-branch.go1.25] cmd/go: reject sumdb response lacking module hash (2026-04-30)
- `5053df5` [release-branch.go1.25] go1.25.10 (2026-05-07)
```

### go1.25.11

```text
- `b79e033` [release-branch.go1.25] mime: avoid quadratic complexity in WordDecoder.DecodeHeader (2026-05-05)
- `449dafe` [release-branch.go1.25] net/textproto: escape arbitrary input when including them in errors (2026-05-11)
- `c5d18e4` [release-branch.go1.25] crypto/x509: split candidate hostname only once (2026-05-27)
- `51f47da` [release-branch.go1.25] runtime/race: build race detector linux/arm64 syso with no-outline-atomics (2026-05-27)
- `917b60b` [release-branch.go1.25] cmd/compile: do not misscompile x+x << 63 to x << 0 on amd64 (2026-05-04)
- `d563bc4` [release-branch.go1.25] go1.25.11 (2026-06-02)
```

### go1.25.12

```text
- `eec53f3` [release-branch.go1.25] os: notify testlog from (*File).Chdir (2026-04-29)
- `2bfd69b` [release-branch.go1.25] cmd/go: use fsys.ReadDir for IsStandardPackage (2026-04-03)
- `dd5597d` [release-branch.go1.25] cmd/compile,sync/atomic: make Add And & Or SQCST on PPC64 (2026-05-04)
- `75d4e8a` [release-branch.go1.25] net: fix TestLookupCNAME (2026-06-30)
- `fc9f821` [release-branch.go1.25] crypto/tls: omit PSK in ECH outer client hello (2026-05-08)
- `c94048f` [release-branch.go1.25] os: properly handle trailing slashes in paths in Root (2026-05-26)
- `d80d9a9` [release-branch.go1.25] go1.25.12 (2026-07-07)
```

### go1.25.13

```text
- `7841324` [release-branch.go1.25] net/http: apply header timeout to server's unencrypted HTTP/2 check (2026-06-24)
- `2d5129d` [release-branch.go1.25] all: update x/net (2026-07-10)
- `21cc406` [release-branch.go1.25] os: don't expect an error from Root.ReadFile from a directory on NetBSD (2026-07-09)
- `4457b22` [release-branch.go1.25] runtime: be sure to scan small frame introduced by (*sigctxt).pushCall (2026-07-07)
- `3f94cdc` [release-branch.go1.25] os: properly handle trailing / in Root.MkdirAll (2026-07-08)
- `08408a5` [release-branch.go1.25] os: strip trailing slashes in RemoveAll on Plan 9 (2026-07-09)
- `081933d` [release-branch.go1.25] runtime: don't emit write barrier for code pointers in itabInit (2026-07-21)
- `ea16975` [release-branch.go1.25] runtime: fix uninitialized 7th argument in mach_vm_region_trampoline (2026-07-16)
- `677cfe5` [release-branch.go1.25] crypto/tls: do not count handshake messages as state-advancing post-handshake (2026-07-22)
- `5720c29` [release-branch.go1.25] cmd/compile: don't require Heapaddr for heap vars in dead code (2026-06-22)
- `8393354` [release-branch.go1.25] cmd/compile: tighten mergelocals address use analysis (2026-07-02)
- `303e06b` [release-branch.go1.25] crypto/tls: add fips140ems GODEBUG setting to disable EMS enforcement (2026-06-19)
- `8c06099` [release-branch.go1.25] cmd/compile: do not home mul/div results in HI/LO (2026-07-24)
- `d5dc717` [release-branch.go1.25] runtime: fix frame pointer adjustment around injected calls (2026-07-31)
- `3f11ff8` [release-branch.go1.25] cmd/compile: fix mips64le bigger than 32bits pointer offsets (2026-07-28)
- `271e96a` [release-branch.go1.25] cmd/compile: fix prove to generate Const64 for 64bits slicemasks (2026-06-06)
- `bff0502` [release-branch.go1.25] cmd/compile: use topo-sort in prove to correctly learn facts while walking once (2025-10-25)
- `f59f3cc` [release-branch.go1.25] cmd/compile: fix time traveling proofs in prove (2026-07-29)
- `b60d45b` [release-branch.go1.25] cmd/compile: do not elide riscv64 sign extension of unsigned values (2026-07-27)
- `962b300` [release-branch.go1.25] net/url: avoid quadratic complexity in resolvePath (2026-07-21)
- `cafd344` [release-branch.go1.25] html/template: fix JavaScript regexp tracking (2026-07-28)
- `b952d04` [release-branch.go1.25] encoding/xml: fix depth processing in (*Decoder).unmarshal (2026-07-20)
- `8d01cba` [release-branch.go1.25] encoding/asn1: enforce maximum recursion depth (2026-06-08)
- `22e0166` [release-branch.go1.25] cmd/vendor: fix CVE-2026-56864 in x/mod (2026-08-07)
- `b0b8c97` [release-branch.go1.25] cmd/vendor: fix CVE-2026-56865 in x/mod (2026-08-07)
- `f893f27` [release-branch.go1.25] go1.25.13 (2026-08-13)
```

### go1.25.14

```text
- `01955f9` [release-branch.go1.25] cmd/internal/moddeps: restore tests (2026-08-13)
- `a3c8d44` [release-branch.go1.25] net/http: clear ReadHeaderTimeout after accepting an unencrypted h2 conn (2026-08-17)
- `08118a9` [release-branch.go1.25] go1.25.14 (2026-08-19)
```
