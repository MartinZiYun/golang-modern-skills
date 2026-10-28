# Go 1.20 Release Notes

## Introduction to Go 1.20

The latest Go release, version 1.20, arrives six months after [Go 1.19](/doc/go1.19).
Most of its changes are in the implementation of the toolchain, runtime, and libraries.
As always, the release maintains the Go 1 [promise of compatibility](/doc/go1compat).
We expect almost all Go programs to continue to compile and run as before.

## Changes to the language

Go 1.20 includes four changes to the language.

Go 1.17 added [conversions from slice to an array pointer](/ref/spec#Conversions_from_slice_to_array_or_array_pointer).
Go 1.20 extends this to allow conversions from a slice to an array:
given a slice `x`, `[4]byte(x)` can now be written
instead of `*(*[4]byte)(x)`.

The [`unsafe` package](/ref/spec/#Package_unsafe) defines
three new functions `SliceData`, `String`, and `StringData`.
Along with Go 1.17&rsquo;s `Slice`, these functions now provide the complete ability to
construct and deconstruct slice and string values, without depending on their exact representation.

The specification now defines that struct values are compared one field at a time,
considering fields in the order they appear in the struct type definition,
and stopping at the first mismatch.
The specification could previously have been read as if
all fields needed to be compared beyond the first mismatch.
Similarly, the specification now defines that array values are compared
one element at a time, in increasing index order.
In both cases, the difference affects whether certain comparisons must panic.
Existing programs are unchanged: the new spec wording describes
what the implementations have always done.

[Comparable types](/ref/spec#Comparison_operators) (such as ordinary interfaces)
may now satisfy `comparable` constraints, even if the type arguments
are not strictly comparable (comparison may panic at runtime).
This makes it possible to instantiate a type parameter constrained by `comparable`
(e.g., a type parameter for a user-defined generic map key) with a non-strictly comparable type argument
such as an interface type, or a composite type containing an interface type.

## Ports

### Windows

Go 1.20 is the last release that will run on any release of Windows 7, 8, Server 2008 and Server 2012.
Go 1.21 will require at least Windows 10 or Server 2016.

### Darwin and iOS

Go 1.20 is the last release that will run on macOS 10.13 High Sierra or 10.14 Mojave.
Go 1.21 will require macOS 10.15 Catalina or later.

### FreeBSD/RISC-V

Go 1.20 adds experimental support for FreeBSD on RISC-V (`GOOS=freebsd`, `GOARCH=riscv64`).

## Tools

### Go command

The directory `$GOROOT/pkg` no longer stores
pre-compiled package archives for the standard library:
`go` `install` no longer writes them,
the `go` build no longer checks for them,
and the Go distribution no longer ships them.
Instead, packages in the standard library are built as needed
and cached in the build cache, just like packages outside `GOROOT`.
This change reduces the size of the Go distribution and also
avoids C toolchain skew for packages that use cgo.

The implementation of `go` `test` `-json`
has been improved to make it more robust.
Programs that run `go` `test` `-json`
do not need any updates.
Programs that invoke `go` `tool` `test2json`
directly should now run the test binary with `-v=test2json`
(for example, `go` `test` `-v=test2json`
or `./pkg.test` `-test.v=test2json`)
instead of plain `-v`.

A related change to `go` `test` `-json`
is the addition of an event with `Action` set to `start`
at the beginning of each test program&rsquo;s execution.
When running multiple tests using the `go` command,
these start events are guaranteed to be emitted in the same order as
the packages named on the command line.

The `go` command now defines
architecture feature build tags, such as `amd64.v2`,
to allow selecting a package implementation file based on the presence
or absence of a particular architecture feature.
See [`go` `help` `buildconstraint`](/cmd/go#hdr-Build_constraints) for details.

The `go` subcommands now accept
`-C` `<dir>` to change directory to <dir>
before performing the command, which may be useful for scripts that need to
execute commands in multiple different modules.

The `go` `build` and `go` `test`
commands no longer accept the `-i` flag,
which has been [deprecated since Go 1.16](/issue/41696).

The `go` `generate` command now accepts
`-skip` `<pattern>` to skip `//go:generate` directives
matching `<pattern>`.

The `go` `test` command now accepts
`-skip` `<pattern>` to skip tests, subtests, or examples
matching `<pattern>`.

When the main module is located within `GOPATH/src`,
`go` `install` no longer installs libraries for
non-`main` packages to `GOPATH/pkg`,
and `go` `list` no longer reports a `Target`
field for such packages. (In module mode, compiled packages are stored in the
[build cache](https://pkg.go.dev/cmd/go#hdr-Build_and_test_caching)
only, but [a bug](/issue/37015) had caused
the `GOPATH` install targets to unexpectedly remain in effect.)

The `go` `build`, `go` `install`,
and other build-related commands now support a `-pgo` flag that enables
profile-guided optimization, which is described in more detail in the
[Compiler](#compiler) section below.
The `-pgo` flag specifies the file path of the profile.
Specifying `-pgo=auto` causes the `go` command to search
for a file named `default.pgo` in the main package&rsquo;s directory and
use it if present.
This mode currently requires a single main package to be specified on the
command line, but we plan to lift this restriction in a future release.
Specifying `-pgo=off` turns off profile-guided optimization.

The `go` `build`, `go` `install`,
and other build-related commands now support a `-cover`
flag that builds the specified target with code coverage instrumentation.
This is described in more detail in the
[Cover](#cover) section below.

#### `go` `version`

The `go` `version` `-m` command
now supports reading more types of Go binaries, most notably, Windows DLLs
built with `go` `build` `-buildmode=c-shared`
and Linux binaries without execute permission.

### Cgo

The `go` command now disables `cgo` by default
on systems without a C toolchain.
More specifically, when the `CGO_ENABLED` environment variable is unset,
the `CC` environment variable is unset,
and the default C compiler (typically `clang` or `gcc`)
is not found in the path,
`CGO_ENABLED` defaults to `0`.
As always, you can override the default by setting `CGO_ENABLED` explicitly.

The most important effect of the default change is that when Go is installed
on a system without a C compiler, it will now use pure Go builds for packages
in the standard library that use cgo, instead of using pre-distributed package archives
(which have been removed, as [noted above](#go-command))
or attempting to use cgo and failing.
This makes Go work better in some minimal container environments
as well as on macOS, where pre-distributed package archives have
not been used for cgo-based packages since Go 1.16.

The packages in the standard library that use cgo are [`net`](/pkg/net/),
[`os/user`](/pkg/os/user/), and
[`plugin`](/pkg/plugin/).
On macOS, the `net` and `os/user` packages have been rewritten not to use cgo:
the same code is now used for cgo and non-cgo builds as well as cross-compiled builds.
On Windows, the `net` and `os/user` packages have never used cgo.
On other systems, builds with cgo disabled will use a pure Go version of these packages.

A consequence is that, on macOS, if Go code that uses
the `net` package is built
with `-buildmode=c-archive`, linking the resulting
archive into a C program requires passing `-lresolv` when
linking the C code.

On macOS, the race detector has been rewritten not to use cgo:
race-detector-enabled programs can be built and run without Xcode.
On Linux and other Unix systems, and on Windows, a host C toolchain
is required to use the race detector.

### Cover

Go 1.20 supports collecting code coverage profiles for programs
(applications and integration tests), as opposed to just unit tests.

To collect coverage data for a program, build it with `go`
`build`&rsquo;s `-cover` flag, then run the resulting
binary with the environment variable `GOCOVERDIR` set
to an output directory for coverage profiles.
See the
[&lsquo;coverage for integration tests&rsquo; landing page](/doc/build-cover) for more on how to get started.
For details on the design and implementation, see the
[proposal](/issue/51430).

### Vet

#### Improved detection of loop variable capture by nested functions

The `vet` tool now reports references to loop variables following
a call to [`T.Parallel()`](/pkg/testing/#T.Parallel)
within subtest function bodies. Such references may observe the value of the
variable from a different iteration (typically causing test cases to be
skipped) or an invalid state due to unsynchronized concurrent access.

The tool also detects reference mistakes in more places. Previously it would
only consider the last statement of the loop body, but now it recursively
inspects the last statements within if, switch, and select statements.

#### New diagnostic for incorrect time formats

The vet tool now reports use of the time format 2006-02-01 (yyyy-dd-mm)
with [`Time.Format`](/pkg/time/#Time.Format) and
[`time.Parse`](/pkg/time/#Parse).
This format does not appear in common date standards, but is frequently
used by mistake when attempting to use the ISO 8601 date format
(yyyy-mm-dd).

## Runtime

Some of the garbage collector&rsquo;s internal data structures were reorganized to
be both more space and CPU efficient.
This change reduces memory overheads and improves overall CPU performance by
up to 2%.

The garbage collector behaves less erratically with respect to goroutine
assists in some circumstances.

Go 1.20 adds a new `runtime/coverage` package
containing APIs for writing coverage profile data at
runtime from long-running and/or server programs that
do not terminate via `os.Exit()`.

## Compiler

Go 1.20 adds preview support for profile-guided optimization (PGO).
PGO enables the toolchain to perform application- and workload-specific
optimizations based on run-time profile information.
Currently, the compiler supports pprof CPU profiles, which can be collected
through usual means, such as the `runtime/pprof` or
`net/http/pprof` packages.
To enable PGO, pass the path of a pprof profile file via the
`-pgo` flag to `go` `build`,
as mentioned [above](#go-command).
Go 1.20 uses PGO to more aggressively inline functions at hot call sites.
Benchmarks for a representative set of Go programs show enabling
profile-guided inlining optimization improves performance about 3–4%.
See the [PGO user guide](/doc/pgo) for detailed documentation.
We plan to add more profile-guided optimizations in future releases.
Note that profile-guided optimization is a preview, so please use it
with appropriate caution.

The Go 1.20 compiler upgraded its front-end to use a new way of handling the
compiler&rsquo;s internal data, which fixes several generic-types issues and enables
type declarations within generic functions and methods.

The compiler now [rejects anonymous interface cycles](/issue/56103)
with a compiler error by default.
These arise from tricky uses of [embedded interfaces](/ref/spec#Embedded_interfaces)
and have always had subtle correctness issues,
yet we have no evidence that they&rsquo;re actually used in practice.
Assuming no reports from users adversely affected by this change,
we plan to update the language specification for Go 1.22 to formally disallow them
so tools authors can stop supporting them too.

Go 1.18 and 1.19 saw regressions in build speed, largely due to the addition
of support for generics and follow-on work. Go 1.20 improves build speeds by
up to 10%, bringing it back in line with Go 1.17.
Relative to Go 1.19, generated code performance is also generally slightly improved.

## Linker

On Linux, the linker now selects the dynamic interpreter for `glibc`
or `musl` at link time.

On Windows, the Go linker now supports modern LLVM-based C toolchains.

Go 1.20 uses `go:` and `type:` prefixes for compiler-generated
symbols rather than `go.` and `type.`.
This avoids confusion for user packages whose name starts with `go.`.
The [`debug/gosym`](/pkg/debug/gosym) package understands
this new naming convention for binaries built with Go 1.20 and newer.

## Bootstrap

When building a Go release from source and `GOROOT_BOOTSTRAP` is not set,
previous versions of Go looked for a Go 1.4 or later bootstrap toolchain in the directory
`$HOME/go1.4` (`%HOMEDRIVE%%HOMEPATH%\go1.4` on Windows).
Go 1.18 and Go 1.19 looked first for `$HOME/go1.17` or `$HOME/sdk/go1.17`
before falling back to `$HOME/go1.4`,
in anticipation of requiring Go 1.17 for use when bootstrapping Go 1.20.
Go 1.20 does require a Go 1.17 release for bootstrapping, but we realized that we should
adopt the latest point release of the bootstrap toolchain, so it requires Go 1.17.13.
Go 1.20 looks for `$HOME/go1.17.13` or `$HOME/sdk/go1.17.13`
before falling back to `$HOME/go1.4`
(to support systems that hard-coded the path $HOME/go1.4 but have installed
a newer Go toolchain there).
In the future, we plan to move the bootstrap toolchain forward approximately once a year,
and in particular we expect that Go 1.22 will require the final point release of Go 1.20 for bootstrap.

## Standard library

### New crypto/ecdh package

Go 1.20 adds a new [`crypto/ecdh`](/pkg/crypto/ecdh/) package
to provide explicit support for Elliptic Curve Diffie-Hellman key exchanges
over NIST curves and Curve25519.

Programs should use `crypto/ecdh` instead of the lower-level functionality in
[`crypto/elliptic`](/pkg/crypto/elliptic/) for ECDH, and
third-party modules for more advanced use cases.

### Wrapping multiple errors

Go 1.20 expands support for error wrapping to permit an error to
wrap multiple other errors.

An error `e` can wrap more than one error by providing
an `Unwrap` method that returns a `[]error`.

The [`errors.Is`](/pkg/errors/#Is) and
[`errors.As`](/pkg/errors/#As) functions
have been updated to inspect multiply wrapped errors.

The [`fmt.Errorf`](/pkg/fmt/#Errorf) function
now supports multiple occurrences of the `%w` format verb,
which will cause it to return an error that wraps all of those error operands.

The new function [`errors.Join`](/pkg/errors/#Join)
returns an error wrapping a list of errors.

### HTTP ResponseController

The new
[`"net/http".ResponseController`](/pkg/net/http/#ResponseController)
type provides access to extended per-request functionality not handled by the
[`"net/http".ResponseWriter`](/pkg/net/http/#ResponseWriter) interface.

Previously, we have added new per-request functionality by defining optional
interfaces which a `ResponseWriter` can implement, such as
[`Flusher`](/pkg/net/http/#Flusher). These interfaces
are not discoverable and clumsy to use.

The `ResponseController` type provides a clearer, more discoverable way
to add per-handler controls. Two such controls also added in Go 1.20 are
`SetReadDeadline` and `SetWriteDeadline`, which allow setting
per-request read and write deadlines. For example:

`func RequestHandler(w ResponseWriter, r *Request) {
rc := http.NewResponseController(w)
rc.SetWriteDeadline(time.Time{}) // disable Server.WriteTimeout when sending a large response
io.Copy(w, bigData)
}
`

### New ReverseProxy Rewrite hook

The [`httputil.ReverseProxy`](/pkg/net/http/httputil/#ReverseProxy)
forwarding proxy includes a new
[`Rewrite`](/pkg/net/http/httputil/#ReverseProxy.Rewrite)
hook function, superseding the
previous `Director` hook.

The `Rewrite` hook accepts a
[`ProxyRequest`](/pkg/net/http/httputil/#ProxyRequest) parameter,
which includes both the inbound request received by the proxy and the outbound
request that it will send.
Unlike `Director` hooks, which only operate on the outbound request,
this permits `Rewrite` hooks to avoid certain scenarios where
a malicious inbound request may cause headers added by the hook
to be removed before forwarding.
See [issue #50580](/issue/50580).

The [`ProxyRequest.SetURL`](/pkg/net/http/httputil/#ProxyRequest.SetURL)
method routes the outbound request to a provided destination
and supersedes the `NewSingleHostReverseProxy` function.
Unlike `NewSingleHostReverseProxy`, `SetURL`
also sets the `Host` header of the outbound request.

The
[`ProxyRequest.SetXForwarded`](/pkg/net/http/httputil/#ProxyRequest.SetXForwarded)
method sets the `X-Forwarded-For`, `X-Forwarded-Host`,
and `X-Forwarded-Proto` headers of the outbound request.
When using a `Rewrite`, these headers are not added by default.

An example of a `Rewrite` hook using these features is:

`proxyHandler := &httputil.ReverseProxy{
Rewrite: func(r *httputil.ProxyRequest) {
r.SetURL(outboundURL) // Forward request to outboundURL.
r.SetXForwarded() // Set X-Forwarded-* headers.
r.Out.Header.Set("X-Additional-Header", "header set by the proxy")
},
}
`

[`ReverseProxy`](/pkg/net/http/httputil/#ReverseProxy) no longer adds a `User-Agent` header
to forwarded requests when the incoming request does not have one.

### Minor changes to the library

As always, there are various minor changes and updates to the library,
made with the Go 1 [promise of compatibility](/doc/go1compat)
in mind.
There are also various performance improvements, not enumerated here.

#### [archive/tar](/pkg/archive/tar/)

When the `GODEBUG=tarinsecurepath=0` environment variable is set,
[`Reader.Next`](/pkg/archive/tar/#Reader.Next) method
will now return the error [`ErrInsecurePath`](/pkg/archive/tar/#ErrInsecurePath)
for an entry with a file name that is an absolute path,
refers to a location outside the current directory, contains invalid
characters, or (on Windows) is a reserved name such as `NUL`.
A future version of Go may disable insecure paths by default.

#### [archive/zip](/pkg/archive/zip/)

When the `GODEBUG=zipinsecurepath=0` environment variable is set,
[`NewReader`](/pkg/archive/zip/#NewReader) will now return the error
[`ErrInsecurePath`](/pkg/archive/zip/#ErrInsecurePath)
when opening an archive which contains any file name that is an absolute path,
refers to a location outside the current directory, contains invalid
characters, or (on Windows) is a reserved names such as `NUL`.
A future version of Go may disable insecure paths by default.

Reading from a directory file that contains file data will now return an error.
The zip specification does not permit directory files to contain file data,
so this change only affects reading from invalid archives.

#### [bytes](/pkg/bytes/)

The new
[`CutPrefix`](/pkg/bytes/#CutPrefix) and
[`CutSuffix`](/pkg/bytes/#CutSuffix) functions
are like [`TrimPrefix`](/pkg/bytes/#TrimPrefix)
and [`TrimSuffix`](/pkg/bytes/#TrimSuffix)
but also report whether the string was trimmed.

The new [`Clone`](/pkg/bytes/#Clone) function
allocates a copy of a byte slice.

#### [context](/pkg/context/)

The new [`WithCancelCause`](/pkg/context/#WithCancelCause) function
provides a way to cancel a context with a given error.
That error can be retrieved by calling the new [`Cause`](/pkg/context/#Cause) function.

#### [crypto/ecdsa](/pkg/crypto/ecdsa/)

When using supported curves, all operations are now implemented in constant time.
This led to an increase in CPU time between 5% and 30%, mostly affecting P-384 and P-521.

The new [`PrivateKey.ECDH`](/pkg/crypto/ecdsa/#PrivateKey.ECDH) method
converts an `ecdsa.PrivateKey` to an `ecdh.PrivateKey`.

#### [crypto/ed25519](/pkg/crypto/ed25519/)

The [`PrivateKey.Sign`](/pkg/crypto/ed25519/#PrivateKey.Sign) method
and the
[`VerifyWithOptions`](/pkg/crypto/ed25519/#VerifyWithOptions) function
now support signing pre-hashed messages with Ed25519ph,
indicated by an
[`Options.HashFunc`](/pkg/crypto/ed25519/#Options.HashFunc)
that returns
[`crypto.SHA512`](/pkg/crypto/#SHA512).
They also now support Ed25519ctx and Ed25519ph with context,
indicated by setting the new
[`Options.Context`](/pkg/crypto/ed25519/#Options.Context)
field.

#### [crypto/rsa](/pkg/crypto/rsa/)

The new field [`OAEPOptions.MGFHash`](/pkg/crypto/rsa/#OAEPOptions.MGFHash)
allows configuring the MGF1 hash separately for OAEP decryption.

crypto/rsa now uses a new, safer, constant-time backend. This causes a CPU
runtime increase for decryption operations between approximately 15%
(RSA-2048 on amd64) and 45% (RSA-4096 on arm64), and more on 32-bit architectures.
Encryption operations are approximately 20x slower than before (but still 5-10x faster than decryption).
Performance is expected to improve in future releases.
Programs must not modify or manually generate the fields of
[`PrecomputedValues`](/pkg/crypto/rsa/#PrecomputedValues).

#### [crypto/subtle](/pkg/crypto/subtle/)

The new function [`XORBytes`](/pkg/crypto/subtle/#XORBytes)
XORs two byte slices together.

#### [crypto/tls](/pkg/crypto/tls/)

Parsed certificates are now shared across all clients actively using that certificate.
The memory savings can be significant in programs that make many concurrent connections to a
server or collection of servers sharing any part of their certificate chains.

For a handshake failure due to a certificate verification failure,
the TLS client and server now return an error of the new type
[`CertificateVerificationError`](/pkg/crypto/tls/#CertificateVerificationError),
which includes the presented certificates.

#### [crypto/x509](/pkg/crypto/x509/)

[`ParsePKCS8PrivateKey`](/pkg/crypto/x509/#ParsePKCS8PrivateKey)
and
[`MarshalPKCS8PrivateKey`](/pkg/crypto/x509/#MarshalPKCS8PrivateKey)
now support keys of type [`*crypto/ecdh.PrivateKey`](/pkg/crypto/ecdh.PrivateKey).
[`ParsePKIXPublicKey`](/pkg/crypto/x509/#ParsePKIXPublicKey)
and
[`MarshalPKIXPublicKey`](/pkg/crypto/x509/#MarshalPKIXPublicKey)
now support keys of type [`*crypto/ecdh.PublicKey`](/pkg/crypto/ecdh.PublicKey).
Parsing NIST curve keys still returns values of type
`*ecdsa.PublicKey` and `*ecdsa.PrivateKey`.
Use their new `ECDH` methods to convert to the `crypto/ecdh` types.

The new [`SetFallbackRoots`](/pkg/crypto/x509/#SetFallbackRoots)
function allows a program to define a set of fallback root certificates in case an
operating system verifier or standard platform root bundle is unavailable at runtime.
It will most commonly be used with a new package, [golang.org/x/crypto/x509roots/fallback](/pkg/golang.org/x/crypto/x509roots/fallback),
which will provide an up to date root bundle.

#### [debug/elf](/pkg/debug/elf/)

Attempts to read from a `SHT_NOBITS` section using
[`Section.Data`](/pkg/debug/elf/#Section.Data)
or the reader returned by [`Section.Open`](/pkg/debug/elf/#Section.Open)
now return an error.

Additional [`R_LARCH_*`](/pkg/debug/elf/#R_LARCH) constants are defined for use with LoongArch systems.

Additional [`R_PPC64_*`](/pkg/debug/elf/#R_PPC64) constants are defined for use with PPC64 ELFv2 relocations.

The constant value for [`R_PPC64_SECTOFF_LO_DS`](/pkg/debug/elf/#R_PPC64_SECTOFF_LO_DS) is corrected, from 61 to 62.

#### [debug/gosym](/pkg/debug/gosym/)

Due to a change of [Go&rsquo;s symbol naming conventions](#linker), tools that
process Go binaries should use Go 1.20&rsquo;s `debug/gosym` package to
transparently handle both old and new binaries.

#### [debug/pe](/pkg/debug/pe/)

Additional [`IMAGE_FILE_MACHINE_RISCV*`](/pkg/debug/pe/#IMAGE_FILE_MACHINE_RISCV128) constants are defined for use with RISC-V systems.

#### [encoding/binary](/pkg/encoding/binary/)

The [`ReadVarint`](/pkg/encoding/binary/#ReadVarint) and
[`ReadUvarint`](/pkg/encoding/binary/#ReadUvarint)
functions will now return `io.ErrUnexpectedEOF` after reading a partial value,
rather than `io.EOF`.

#### [encoding/xml](/pkg/encoding/xml/)

The new [`Encoder.Close`](/pkg/encoding/xml/#Encoder.Close) method
can be used to check for unclosed elements when finished encoding.

The decoder now rejects element and attribute names with more than one colon,
such as `<a:b:c>`,
as well as namespaces that resolve to an empty string, such as `xmlns:a=""`.

The decoder now rejects elements that use different namespace prefixes in the opening and closing tag,
even if those prefixes both denote the same namespace.

#### [errors](/pkg/errors/)

The new [`Join`](/pkg/errors/#Join) function returns an error wrapping a list of errors.

#### [fmt](/pkg/fmt/)

The [`Errorf`](/pkg/fmt/#Errorf) function supports multiple occurrences of
the `%w` format verb, returning an error that unwraps to the list of all arguments to `%w`.

The new [`FormatString`](/pkg/fmt/#FormatString) function recovers the
formatting directive corresponding to a [`State`](/pkg/fmt/#State),
which can be useful in [`Formatter`](/pkg/fmt/#Formatter).
implementations.

#### [go/ast](/pkg/go/ast/)

The new [`RangeStmt.Range`](/pkg/go/ast/#RangeStmt.Range) field
records the position of the `range` keyword in a range statement.

The new [`File.FileStart`](/pkg/go/ast/#File.FileStart)
and [`File.FileEnd`](/pkg/go/ast/#File.FileEnd) fields
record the position of the start and end of the entire source file.

#### [go/token](/pkg/go/token/)

The new [`FileSet.RemoveFile`](/pkg/go/token/#FileSet.RemoveFile) method
removes a file from a `FileSet`.
Long-running programs can use this to release memory associated
with files they no longer need.

#### [go/types](/pkg/go/types/)

The new [`Satisfies`](/pkg/go/types/#Satisfies) function reports
whether a type satisfies a constraint.
This change aligns with the [new language semantics](#language)
that distinguish satisfying a constraint from implementing an interface.

#### [html/template](/pkg/html/template/)

Go 1.20.3 and later
[disallow actions in ECMAScript 6 template literals.](/pkg/html/template#hdr-Security_Model)
This behavior can be reverted by the `GODEBUG=jstmpllitinterp=1` setting.

#### [io](/pkg/io/)

The new [`OffsetWriter`](/pkg/io/#OffsetWriter) wraps an underlying
[`WriterAt`](/pkg/io/#WriterAt)
and provides `Seek`, `Write`, and `WriteAt` methods
that adjust their effective file offset position by a fixed amount.

#### [io/fs](/pkg/io/fs/)

The new error [`SkipAll`](/pkg/io/fs/#SkipAll)
terminates a [`WalkDir`](/pkg/io/fs/#WalkDir)
immediately but successfully.

#### [math/big](/pkg/math/big/)

The [math/big](/pkg/math/big/) package&rsquo;s wide scope and
input-dependent timing make it ill-suited for implementing cryptography.
The cryptography packages in the standard library no longer call non-trivial
[Int](/pkg/math/big#Int) methods on attacker-controlled inputs.
In the future, the determination of whether a bug in math/big is
considered a security vulnerability will depend on its wider impact on the
standard library.

#### [math/rand](/pkg/math/rand/)

The [math/rand](/pkg/math/rand/) package now automatically seeds
the global random number generator
(used by top-level functions like `Float64` and `Int`) with a random value,
and the top-level [`Seed`](/pkg/math/rand/#Seed) function has been deprecated.
Programs that need a reproducible sequence of random numbers
should prefer to allocate their own random source, using `rand.New(rand.NewSource(seed))`.

Programs that need the earlier consistent global seeding behavior can set
`GODEBUG=randautoseed=0` in their environment.

The top-level [`Read`](/pkg/math/rand/#Read) function has been deprecated.
In almost all cases, [`crypto/rand.Read`](/pkg/crypto/rand/#Read) is more appropriate.

#### [mime](/pkg/mime/)

The [`ParseMediaType`](/pkg/mime/#ParseMediaType) function now allows duplicate parameter names,
so long as the values of the names are the same.

#### [mime/multipart](/pkg/mime/multipart/)

Methods of the [`Reader`](/pkg/mime/multipart/#Reader) type now wrap errors
returned by the underlying `io.Reader`.

In Go 1.19.8 and later, this package sets limits the size
of the MIME data it processes to protect against malicious inputs.
`Reader.NextPart` and `Reader.NextRawPart` limit the
number of headers in a part to 10000 and `Reader.ReadForm` limits
the total number of headers in all `FileHeaders` to 10000.
These limits may be adjusted with the `GODEBUG=multipartmaxheaders`
setting.
`Reader.ReadForm` further limits the number of parts in a form to 1000.
This limit may be adjusted with the `GODEBUG=multipartmaxparts`
setting.

#### [net](/pkg/net/)

The [`LookupCNAME`](/pkg/net/#LookupCNAME)
function now consistently returns the contents
of a `CNAME` record when one exists. Previously on Unix systems and
when using the pure Go resolver, `LookupCNAME` would return an error
if a `CNAME` record referred to a name that with no `A`,
`AAAA`, or `CNAME` record. This change modifies
`LookupCNAME` to match the previous behavior on Windows,
allowing `LookupCNAME` to succeed whenever a
`CNAME` exists.

[`Interface.Flags`](/pkg/net/#Interface.Flags) now includes the new flag `FlagRunning`,
indicating an operationally active interface. An interface which is administratively
configured but not active (for example, because the network cable is not connected)
will have `FlagUp` set but not `FlagRunning`.

The new [`Dialer.ControlContext`](/pkg/net/#Dialer.ControlContext) field contains a callback function
similar to the existing [`Dialer.Control`](/pkg/net/#Dialer.Control) hook, that additionally
accepts the dial context as a parameter.
`Control` is ignored when `ControlContext` is not nil.

The Go DNS resolver recognizes the `trust-ad` resolver option.
When `options trust-ad` is set in `resolv.conf`,
the Go resolver will set the AD bit in DNS queries. The resolver does not
make use of the AD bit in responses.

DNS resolution will detect changes to `/etc/nsswitch.conf`
and reload the file when it changes. Checks are made at most once every
five seconds, matching the previous handling of `/etc/hosts`
and `/etc/resolv.conf`.

#### [net/http](/pkg/net/http/)

The [`ResponseWriter.WriteHeader`](/pkg/net/http/#ResponseWriter.WriteHeader) function now supports sending
`1xx` status codes.

The new [`Server.DisableGeneralOptionsHandler`](/pkg/net/http/#Server.DisableGeneralOptionsHandler) configuration setting
allows disabling the default `OPTIONS *` handler.

The new [`Transport.OnProxyConnectResponse`](/pkg/net/http/#Transport.OnProxyConnectResponse) hook is called
when a `Transport` receives an HTTP response from a proxy
for a `CONNECT` request.

The HTTP server now accepts HEAD requests containing a body,
rather than rejecting them as invalid.

HTTP/2 stream errors returned by `net/http` functions may be converted
to a [`golang.org/x/net/http2.StreamError`](/pkg/golang.org/x/net/http2/#StreamError) using
[`errors.As`](/pkg/errors/#As).

Leading and trailing spaces are trimmed from cookie names,
rather than being rejected as invalid.
For example, a cookie setting of &ldquo;name =value&rdquo;
is now accepted as setting the cookie &ldquo;name&rdquo;.

A [`Cookie`](/pkg/net/http#Cookie) with an empty Expires field is now considered valid.
[`Cookie.Valid`](/pkg/net/http#Cookie.Valid) only checks Expires when it is set.

#### [net/netip](/pkg/net/netip/)

The new [`IPv6LinkLocalAllRouters`](/pkg/net/netip/#IPv6LinkLocalAllRouters)
and [`IPv6Loopback`](/pkg/net/netip/#IPv6Loopback) functions
are the `net/netip` equivalents of
[`net.IPv6loopback`](/pkg/net/#IPv6loopback) and
[`net.IPv6linklocalallrouters`](/pkg/net/#IPv6linklocalallrouters).

#### [os](/pkg/os/)

On Windows, the name `NUL` is no longer treated as a special case in
[`Mkdir`](/pkg/os/#Mkdir) and
[`Stat`](/pkg/os/#Stat).

On Windows, [`File.Stat`](/pkg/os/#File.Stat)
now uses the file handle to retrieve attributes when the file is a directory.
Previously it would use the path passed to
[`Open`](/pkg/os/#Open), which may no longer be the file
represented by the file handle if the file has been moved or replaced.
This change modifies `Open` to open directories without the
`FILE_SHARE_DELETE` access, which match the behavior of regular files.

On Windows, [`File.Seek`](/pkg/os/#File.Seek) now supports
seeking to the beginning of a directory.

#### [os/exec](/pkg/os/exec/)

The new [`Cmd`](/pkg/os/exec/#Cmd) fields
[`Cancel`](/pkg/os/exec/#Cmd.Cancel) and
[`WaitDelay`](/pkg/os/exec/#Cmd.WaitDelay)
specify the behavior of the `Cmd` when its associated
`Context` is canceled or its process exits with I/O pipes still
held open by a child process.

#### [path/filepath](/pkg/path/filepath/)

The new error [`SkipAll`](/pkg/path/filepath/#SkipAll)
terminates a [`Walk`](/pkg/path/filepath/#Walk)
immediately but successfully.

The new [`IsLocal`](/pkg/path/filepath/#IsLocal) function reports whether a path is
lexically local to a directory.
For example, if `IsLocal(p)` is `true`,
then `Open(p)` will refer to a file that is lexically
within the subtree rooted at the current directory.

#### [reflect](/pkg/reflect/)

The new [`Value.Comparable`](/pkg/reflect/#Value.Comparable) and
[`Value.Equal`](/pkg/reflect/#Value.Equal) methods
can be used to compare two `Value`s for equality.
`Comparable` reports whether `Equal` is a valid operation for a given `Value` receiver.

The new [`Value.Grow`](/pkg/reflect/#Value.Grow) method
extends a slice to guarantee space for another `n` elements.

The new [`Value.SetZero`](/pkg/reflect/#Value.SetZero) method
sets a value to be the zero value for its type.

Go 1.18 introduced [`Value.SetIterKey`](/pkg/reflect/#Value.SetIterKey)
and [`Value.SetIterValue`](/pkg/reflect/#Value.SetIterValue) methods.
These are optimizations: `v.SetIterKey(it)` is meant to be equivalent to `v.Set(it.Key())`.
The implementations incorrectly omitted a check for use of unexported fields that was present in the unoptimized forms.
Go 1.20 corrects these methods to include the unexported field check.

#### [regexp](/pkg/regexp/)

Go 1.19.2 and Go 1.18.7 included a security fix to the regular expression parser,
making it reject very large expressions that would consume too much memory.
Because Go patch releases do not introduce new API,
the parser returned [`syntax.ErrInternalError`](/pkg/regexp/syntax/#ErrInternalError) in this case.
Go 1.20 adds a more specific error, [`syntax.ErrLarge`](/pkg/regexp/syntax/#ErrLarge),
which the parser now returns instead.

#### [runtime/cgo](/pkg/runtime/cgo/)

Go 1.20 adds new [`Incomplete`](/pkg/runtime/cgo/#Incomplete) marker type.
Code generated by cgo will use `cgo.Incomplete` to mark an incomplete C type.

#### [runtime/metrics](/pkg/runtime/metrics/)

Go 1.20 adds new [supported metrics](/pkg/runtime/metrics/#hdr-Supported_metrics),
including the current `GOMAXPROCS` setting (`/sched/gomaxprocs:threads`),
the number of cgo calls executed (`/cgo/go-to-c-calls:calls`),
total mutex block time (`/sync/mutex/wait/total:seconds`), and various measures of time
spent in garbage collection.

Time-based histogram metrics are now less precise, but take up much less memory.

#### [runtime/pprof](/pkg/runtime/pprof/)

Mutex profile samples are now pre-scaled, fixing an issue where old mutex profile
samples would be scaled incorrectly if the sampling rate changed during execution.

Profiles collected on Windows now include memory mapping information that fixes
symbolization issues for position-independent binaries.

#### [runtime/trace](/pkg/runtime/trace/)

The garbage collector&rsquo;s background sweeper now yields less frequently,
resulting in many fewer extraneous events in execution traces.

#### [strings](/pkg/strings/)

The new
[`CutPrefix`](/pkg/strings/#CutPrefix) and
[`CutSuffix`](/pkg/strings/#CutSuffix) functions
are like [`TrimPrefix`](/pkg/strings/#TrimPrefix)
and [`TrimSuffix`](/pkg/strings/#TrimSuffix)
but also report whether the string was trimmed.

#### [sync](/pkg/sync/)

The new [`Map`](/pkg/sync/#Map) methods [`Swap`](/pkg/sync/#Map.Swap),
[`CompareAndSwap`](/pkg/sync/#Map.CompareAndSwap), and
[`CompareAndDelete`](/pkg/sync/#Map.CompareAndDelete)
allow existing map entries to be updated atomically.

#### [syscall](/pkg/syscall/)

On FreeBSD, compatibility shims needed for FreeBSD 11 and earlier have been removed.

On Linux, additional [`CLONE_*`](/pkg/syscall/#CLONE_CLEAR_SIGHAND) constants
are defined for use with the [`SysProcAttr.Cloneflags`](/pkg/syscall/#SysProcAttr.Cloneflags) field.

On Linux, the new [`SysProcAttr.CgroupFD`](/pkg/syscall/#SysProcAttr.CgroupFD)
and [`SysProcAttr.UseCgroupFD`](/pkg/syscall/#SysProcAttr.UseCgroupFD) fields
provide a way to place a child process into a specific cgroup.

#### [testing](/pkg/testing/)

The new method [`B.Elapsed`](/pkg/testing/#B.Elapsed)
reports the current elapsed time of the benchmark, which may be useful for
calculating rates to report with `ReportMetric`.

Calling [`T.Run`](/pkg/testing/#T.Run)
from a function passed
to [`T.Cleanup`](/pkg/testing/#T.Cleanup)
was never well-defined, and will now panic.

#### [time](/pkg/time/)

The new time layout constants [`DateTime`](/pkg/time/#DateTime),
[`DateOnly`](/pkg/time/#DateOnly), and
[`TimeOnly`](/pkg/time/#TimeOnly)
provide names for three of the most common layout strings used in a survey of public Go source code.

The new [`Time.Compare`](/pkg/time/#Time.Compare) method
compares two times.

[`Parse`](/pkg/time/#Parse)
now ignores sub-nanosecond precision in its input,
instead of reporting those digits as an error.

The [`Time.MarshalJSON`](/pkg/time/#Time.MarshalJSON) method
is now more strict about adherence to RFC 3339.

#### [unicode/utf16](/pkg/unicode/utf16/)

The new [`AppendRune`](/pkg/unicode/utf16/#AppendRune)
function appends the UTF-16 encoding of a given rune to a uint16 slice,
analogous to [`utf8.AppendRune`](/pkg/unicode/utf8/#AppendRune).

## Patch Release Changelogs

### go1.20.1

```text
- `7302f83` [release-branch.go1.20] cmd/compile: remove constant arithmetic overflows during typecheck (2023-02-04)
- `487be3f` [release-branch.go1.20] cmd/compile: fix inline static init arguments substitued tree (2023-02-05)
- `90b0600` [release-branch.go1.20] cmd/compile/internal/noder: stop creating TUNION types (2022-12-02)
- `9987cb6` [release-branch.go1.20] time: update windows zoneinfo_abbrs (2023-01-27)
- `fbba58a` [release-branch.go1.20] cmd/link: keep go.buildinfo even with --gc-sections (2023-02-01)
- `a943fd0` [release-branch.go1.20] runtime: skip darwin osinit_hack on ios (2023-02-08)
- `1fa2deb` [release-branch.go1.20] cmd/go: remove tests that assume lack of new versions of external modules (2023-02-09)
- `7628627` [release-branch.go1.20] cmd/go/internal/test: refresh flagdefs.go and fix test (2023-02-08)
- `00f5d30` [release-branch.go1.20] cmd/go/internal/script: retry ETXTBSY errors in scripts (2023-01-31)
- `3a04b6e` [release-branch.go1.20] cmd/compile/internal/pgo: fix hard-coded PGO sample data position (2023-02-08)
- `bdf07c2` [release-branch.go1.20] path/filepath: do not Clean("a/../c:/b") into c:\b on Windows (2022-12-13)
- `53b4360` [release-branch.go1.20] mime/multipart: limit memory/inode consumption of ReadForm (2023-01-25)
- `5286ac4` [release-branch.go1.20] crypto/tls: replace all usages of BytesOrPanic (2022-12-14)
- `8e02cff` [release-branch.go1.20] net/http: update bundled golang.org/x/net/http2 (2023-02-06)
- `202a1a5` [release-branch.go1.20] go1.20.1 (2023-02-14)
```

### go1.20.2

```text
- `828b05c` [release-branch.go1.20] all: update vendored golang.org/x/net (2023-02-14)
- `85ded85` [release-branch.go1.20] runtime: fix signature for linked functions (2023-02-08)
- `965e9ba` [release-branch.go1.20] cmd/compile: disable inline static init optimization (2023-02-09)
- `2d01f36` [release-branch.go1.20] cmd/compile: fix wrong escape analysis for go/defer generic calls (2023-02-07)
- `7b398b1` [release-branch.go1.20] runtime: check for overflow in sweep assist (2023-01-04)
- `1acd39c` [release-branch.go1.20] Revert "internal/poll: drop redundant ENOSYS in CopyFileRange" (2023-02-21)
- `ac556f3` [release-branch.go1.20] cmd/internal/cov: fix misuse of bufio.Reader.Read in read helper (2023-02-08)
- `602eeaa` [release-branch.go1.20] crypto/internal/nistec: reduce P-256 scalar (2023-02-13)
- `1362737` [release-branch.go1.20] cmd/link: better fix for arm32 trampgen problem with duff routines (2023-02-17)
- `0f4483c` [release-branch.go1.20] cmd/compile/internal/noder: correct positions for synthetic closures (2023-02-14)
- `aaace6d` [release-branch.go1.20] crypto/ecdh: explicitly reject mismatched curves in ECDH (2023-01-31)
- `ef79380` [release-branch.go1.20] crypto/internal/bigmod: flag amd64 assembly as noescape (2023-01-08)
- `aef8a8c` [release-branch.go1.20] syscall: Faccessat: check for CAP_DAC_OVERRIDE on Linux (2023-02-16)
- `bdd86bd` [release-branch.go1.20] crypto/x509: fix ParsePKCS8PrivateKey comment (2023-02-28)
- `230765a` [release-branch.go1.20] net: delete TestTCPSelfConnect (2023-01-05)
- `d2d0ee2` [release-branch.go1.20] syscall: fix invalid unsafe.Pointer conversion on Windows (2023-02-25)
- `3243f93` [release-branch.go1.20] crypto/x509: fix system root tests + darwin intermediate handling (2023-02-28)
- `9629fa1` [release-branch.go1.20] crypto/x509: fix broken tests (2023-03-01)
- `26eeaec` [release-branch.go1.20] cmd/compile: relax overly strict assertion (2023-02-21)
- `aee9a19` [release-branch.go1.20] go1.20.2 (2023-03-07)
```

### go1.20.3

```text
- `4df95d5` [release-branch.go1.20] internal/testpty: fix error handling (2023-01-25)
- `b852f39` [release-branch.go1.20] cmd/go: avoid running slow tests on non-longtest builders (2023-03-07)
- `5c7cc46` [release-branch.go1.20] Revert "cmd/compile: enable address folding for global symbols of shared library" (2023-03-07)
- `fa42da1` [release-branch.go1.20] cmd/link: use label symbols for Duff's devices on darwin/arm64 (2023-03-08)
- `3ff6dbd` [release-branch.go1.20] cmd/go,cmd/link: prefer external linking when strange cgo flags seen (2023-03-10)
- `b52a696` [release-branch.go1.20] cmd/link/internal/arm: fix off-by-1 in trampoline reachability computation (2023-03-13)
- `65fa8a6` [release-branch.go1.20] time: fix timezone lookup logic for non-DST zones (2023-03-11)
- `94c02a3` [release-branch.go1.20] cmd/compile: re-compile instantiated generic methods in linkshared mode (2023-03-17)
- `8dce4ca` [release-branch.go1.20] cmd/compile: don't assume pointer of a slice is non-nil (2023-03-28)
- `9a164d1` [release-branch.go1.20] cmd/internal/obj/ppc64: fix incorrect base reg causing segv (2023-03-27)
- `3991f6c` [release-branch.go1.20] net/textproto: avoid overpredicting the number of MIME header keys (2023-03-10)
- `ea6b5a6` [release-branch.go1.20] mime/multipart: avoid excessive copy buffer allocations in ReadForm (2023-03-16)
- `ec18f62` [release-branch.go1.20] net/textproto, mime/multipart: improve accounting of non-file data (2023-03-16)
- `bf8c7c5` [release-branch.go1.20] mime/multipart: limit parsed mime message sizes (2023-03-20)
- `e7c4b07` [release-branch.go1.20] go/scanner: reject large line and column numbers in //line directives (2023-03-22)
- `20374d1` [release-branch.go1.20] html/template: disallow actions in JS template literals (2023-03-20)
- `7c47a6b` [release-branch.go1.20] go1.20.3 (2023-04-04)
```

### go1.20.4

```text
- `5c7c20e` [release-branch.go1.20] html/template,mime/multipart: document new GODEBUG settings (2023-04-04)
- `dcc9bdf` [release-branch.go1.20] crypto/subtle: don't cast to *uintptr when word size is 0 (2023-03-30)
- `99001c4` [release-branch.go1.20] cmd/compile: don't set range expr key/value type if already set (2023-04-03)
- `1dbbac7` [release-branch.go1.20] cmd/compile: fix ir.StaticValue for ORANGE (2023-04-12)
- `ecf7e00` [release-branch.go1.20] syscall: restore original NOFILE rlimit in child process (2023-03-22)
- `0684cec` [release-branch.go1.20] cmd/compile: use correct type for byteswaps on multi-byte stores (2023-04-01)
- `446493f` [release-branch.go1.20] cmd/compile: remove broken LEA "optimization" (2023-04-06)
- `ee42d46` [release-branch.go1.20] cmd/compile: fix reproducible build of aliased generic types (2023-04-12)
- `813a811` [release-branch.go1.20] crypto/tls: fix PSK binder calculation (2023-04-12)
- `484535e` cmd/compile/internal/importer,go/internal/gcimporter: use the 'go' command from build.Default.GOROOT in lookupGorootExport (2023-04-14)
- `25b4f40` [release-branch.go1.20] Revert "net/http: FileServer method check + minimal OPTIONS implementation" (2023-04-05)
- `090590f` [release-branch.go1.20] html/template: disallow angle brackets in CSS values (2023-04-13)
- `4a28cad` [release-branch.go1.20] html/template: handle all JS whitespace characters (2023-04-11)
- `337dd75` [release-branch.go1.20] html/template: emit filterFailsafe for empty unquoted attr value (2023-04-13)
- `324c3ac` [release-branch.go1.20] go1.20.4 (2023-05-02)
```

### go1.20.5

```text
- `afbe101` [release-branch.go1.20] cmd/compile: fix bswap/load rewrite rules (2023-05-04)
- `600636e` [release-branch.go1.20] crypto/rsa: use BoringCrypto for 4096 bit keys (2023-03-08)
- `9270e3b` [release-branch.go1.20] os: if descriptor is non-blocking, retain that in Fd method (2023-05-16)
- `2274112` [release-branch.go1.20] runtime: consistently define fcntl (2023-05-15)
- `450c802` [release-branch.go1.20] runtime: change fcntl to return two values (2023-05-19)
- `31a1e19` [release-branch.go1.20] net, os: net.Conn.File.Fd should return a blocking descriptor (2023-05-19)
- `4b95fc1` [release-branch.go1.20] cmd/go: save checksums for go.mod files needed for go version lines (2023-04-26)
- `b249ec5` [release-branch.go1.20] cmd/go/internal: update documentation of go test and go generate (2023-05-10)
- `5036ba7` [release-branch.go1.20] net: skip TestFileFdBlocks if the "unix" network is not supported (2023-05-22)
- `36144ba` [release-branch.go1.20] runtime: implement SUID/SGID protections (2023-05-09)
- `fa60c38` [release-branch.go1.20] cmd/go,cmd/cgo: in _cgo_flags use one line per flag (2023-05-04)
- `356a419` [release-branch.go1.20] cmd/go: enforce flags with non-optional arguments (2023-05-05)
- `c0ed873` [release-branch.go1.20] cmd/go: disallow package directories containing newlines (2023-05-12)
- `e827d41` [release-branch.go1.20] go1.20.5 (2023-06-06)
```

### go1.20.6

```text
- `1008486` [release-branch.go1.20] cmd/cgo: correct _cgo_flags output (2023-06-06)
- `8b3acef` [release-branch.go1.20] cmd/go: omit checksums for go.mod files needed for go version lines more often in pre-1.21 modules (2023-06-08)
- `f5172dc` [release-branch.go1.20] go/build: check for invalid import paths again (2023-06-12)
- `a7a48fa` [release-branch.go1.20] crypto/ecdsa: properly truncate P-521 hashes (2023-06-12)
- `95f377d` [release-branch.go1.20] cmd/go: retain extra roots to disambiguate imports in 'go mod tidy' (2023-05-20)
- `63ad2b5` [release-branch.go1.20] cmd/compile: do not report division by error during typecheck (2023-06-08)
- `6c606fc` [release-branch.go1.20] cmd/go: fix TestScript/build_cwd_newline with CGO_ENABLED=0 (2023-05-12)
- `4719048` [release-branch.go1.20] cmd/cgo: error out if the source path used in line directives would contain a newline (2023-05-15)
- `4959424` [release-branch.go1.20] cmd/cover: error out if a requested source file contains a newline (2023-05-15)
- `d51e322` [release-branch.go1.20] go/printer: error out of Fprint when it would write a '//line' directive with a multiline file path (2023-05-15)
- `0306310` [release-branch.go1.20] text/template: set variables correctly in range assignment (2023-06-14)
- `c7b1456` [release-branch.go1.20] cmd/go: fix tests for new builder environment (2023-03-29)
- `c32f1af` [release-branch.go1.20] all: make safe for new vet analyzer (2023-06-21)
- `b59efe6` [release-branch.go1.20] net/mail: permit more characters in mail headers (2023-06-19)
- `be30960` [release-branch.go1.20] runtime: use 1-byte load for address checking in racecallatomic (2023-06-15)
- `6b45fb7` [release-branch.go1.20] runtime: fallback to TEB arbitrary pointer when TLS slots are full (2023-04-20)
- `3db4f81` [release-branch.go1.20] runtime: resolve checkdead panic by refining `startm` lock handling in caller context (2023-04-21)
- `b8e67d1` [release-branch.go1.20] cmd/go/internal/test: don't wait for previous test actions when interrupted (2023-06-15)
- `bca8175` [release-branch.go1.20] crypto/x509: tolerate multiple matching chains in testVerify (2023-06-21)
- `6509283` [release-branch.go1.20] cmd/go: skip TestScript/gccgo_link_ldflags on aix/ppc64 (2023-06-07)
- `08a58dd` [release-branch.go1.20] runtime: allow for 5 more threads in TestWindowsStackMemory* (2023-03-04)
- `4db13d7` [release-branch.go1.20] runtime: set raceignore to zero when starting a new goroutine (2023-06-22)
- `312920c` [release-branch.go1.20] net/http: validate Host header before sending (2023-06-28)
- `2c358ff` [release-branch.go1.20] go1.20.6 (2023-07-11)
```

### go1.20.7

```text
- `6211a02` [release-branch.go1.20] cmd/compile: on PPC64, fix sign/zero extension when masking (2023-07-11)
- `bd3a1f2` [release-branch.go1.20] net: tolerate permission errors in interface tests (2023-07-18)
- `10d85fa` [release-branch.go1.20] cmd/asm, cmd/internal/obj: generate proper atomic ops for riscv64 (2023-07-11)
- `659f2a2` [release-branch.go1.20] crypto/tls: restrict RSA keys in certificates to <= 8192 bits (2023-06-07)
- `adb775e` [release-branch.go1.20] go1.20.7 (2023-08-01)
```

### go1.20.8

```text
- `201f8b4` [release-branch.go1.20] cmd/go: refuse to build Go 1.22 code (2023-08-11)
- `ede3e27` [release-branch.go1.20] net/http: permit requests with invalid Host headers (2023-07-19)
- `1a91bb9` [release-branch.go1.20] cmd/go: do not index std as a module in modcache (2023-06-16)
- `095cfba` [release-branch.go1.20] crypto/tls: add GODEBUG to control max RSA key size (2023-08-09)
- `14e3c73` [release-branch.go1.20] cmd/api: rename api.go to main_test.go (2023-08-16)
- `974a3c9` [release-branch.go1.20] go/types, types2: don't panic during interface completion (2023-08-03)
- `b463668` [release-branch.go1.20] runtime: profiling on Darwin cannot use blocking reads (2023-08-11)
- `f0b8768` [release-branch.go1.20] path/filepath: don't drop .. elements when cleaning invalid Windows paths (2023-08-08)
- `612da32` [release-branch.go1.20] cmd/compile: in expandCalls, move all arg marshalling into call block (2023-08-15)
- `023b542` [release-branch.go1.20] html/template: support HTML-like comments in script contexts (2023-08-03)
- `2070531` [release-branch.go1.20] html/template: properly handle special tags within the script context (2023-08-03)
- `d5b8518` [release-branch.go1.20] go1.20.8 (2023-09-06)
```

### go1.20.9

```text
- `071f03a` [release-branch.go1.20] cmd/link: force old Apple linker in plugin mode (2023-09-12)
- `08c544d` [release-branch.go1.20] cmd/link: disable DWARF by default in c-shared mode on darwin (2023-09-12)
- `83dce45` [release-branch.go1.20] cmd/link: suppress -bind_at_load deprecation warning for ld-prime (2023-07-10)
- `31d5b60` [release-branch.go1.20] cmd/compile: use absolute file name in isCgo check (2023-09-20)
- `68f9a6e` [release-branch.go1.20] go1.20.9 (2023-10-05)
```

### go1.20.10

```text
- `e175f27` [release-branch.go1.20] net/http: regenerate h2_bundle.go (2023-10-06)
- `8042fd8` [release-branch.go1.20] go1.20.10 (2023-10-10)
```

### go1.20.11

```text
- `c8fdffb` [release-branch.go1.20] all: tidy dependency versioning after release (2023-10-10)
- `d486390` [release-branch.go1.20] cmd/link: split text sections for arm 32-bit (2023-09-29)
- `998fdce` [release-branch.go1.20] net/http: pull http2 underflow fix from x/net/http2 (2023-10-26)
- `46fb781` [release-branch.go1.20] path/filepath: fix various issues in parsing Windows paths (2023-09-01)
- `1d0d4b1` [release-branch.go1.20] go1.20.11 (2023-11-07)
```

### go1.20.12

```text
- `e1dc209` [release-branch.go1.20] cmd/go/internal/modfetch/codehost: set core.longpaths in Git repos on Windows (2023-04-06)
- `46bc338` [release-branch.go1.20] cmd/go/internal/vcs: error out if the requested repo does not support a secure protocol (2023-11-02)
- `1b59b01` [release-branch.go1.20] path/filepath: consider \\?\c: as a volume on Windows (2023-11-09)
- `1bd7657` [release-branch.go1.20] crypto/rand,runtime: switch RtlGenRandom for ProcessPrng (2023-10-18)
- `d77307f` [release-branch.go1.20] cmd/compile: fix findIndVar so it does not match disjointed loop headers (2023-11-05)
- `77397ff` [release-branch.go1.20] crypto/rand,runtime: revert "switch RtlGenRandom for ProcessPrng" (2023-11-29)
- `6446af9` [release-branch.go1.20] net/http: limit chunked data overhead (2023-11-07)
- `97c8ff8` [release-branch.go1.20] go1.20.12 (2023-12-05)
```

### go1.20.13

```text
- `59ffd3b` [release-branch.go1.20] os/signal: remove go t.Run from TestNohup (2023-11-01)
- `8cb86b5` [release-branch.go1.20] os/signal: skip nohup tests on darwin builders (2023-11-01)
- `95afc74` [release-branch.go1.20] runtime: put ReadMemStats debug assertions behind a double-check mode (2023-11-27)
- `9e4abed` [release-branch.go1.20] crypto/internal/boring: upgrade module to fips-20220613 (2023-12-14)
- `5c38c04` [release-branch.go1.20] crypto/tls: align FIPS-only mode with BoringSSL policy (2023-12-14)
- `a95136a` [release-branch.go1.20] go1.20.13 (2024-01-09)
```

### go1.20.14

```text
- `d7df7f4` [release-branch.go1.20] runtime: properly model rwmutex in lock ranking (2023-12-13)
- `746a072` [release-branch.go1.20] crypto/x509: properly gate test on macos version (2023-12-12)
- `a2f4a5a` [release-branch.go1.20] Revert "crypto/internal/boring: upgrade module to fips-20220613" +1 (2024-01-26)
- `90a870f` [release-branch.go1.20] go1.20.14 (2024-02-06)
```
