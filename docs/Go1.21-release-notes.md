# Go 1.21 Release Notes

## Introduction to Go 1.21

The latest Go release, version 1.21, arrives six months after [Go 1.20](/doc/go1.20).
Most of its changes are in the implementation of the toolchain, runtime, and libraries.
As always, the release maintains the Go 1 [promise of compatibility](/doc/go1compat);
in fact, Go 1.21 [improves upon that promise](#tools).
We expect almost all Go programs to continue to compile and run as before.

Go 1.21 introduces a small change to the numbering of releases.
In the past, we used Go 1.N* to refer to both the overall Go language version and release family
as well as the first release in that family.
Starting in Go 1.21, the first release is now Go 1.*N*.0.
Today we are releasing both the Go 1.21 language and its initial implementation, the Go 1.21.0 release.
These notes refer to “Go 1.21”; tools like `go` `version` will report “`go1.21.0`”
(until you upgrade to Go 1.21.1).
See “[Go versions](/doc/toolchain#version)” in the “Go Toolchains” documentation for details
about the new version numbering.

## Changes to the language

Go 1.21 adds three new built-ins to the language.

The new functions `min` and `max` compute the
smallest (or largest, for `max`) value of a fixed number
of given arguments.
See the language spec for
[details](/ref/spec#Min_and_max).

-

The new function `clear` deletes all elements from a
map or zeroes all elements of a slice.
See the language spec for
[details](/ref/spec#Clear).

Package initialization order is now specified more precisely. The
new algorithm is:

- Sort all packages by import path.

- Repeat until the list of packages is empty:

Find the first package in the list for which all imports are
already initialized.

- Initialize that package and remove it from the list.

This may change the behavior of some programs that rely on a
specific initialization ordering that was not expressed by explicit
imports. The behavior of such programs was not well defined by the
spec in past releases. The new rule provides an unambiguous definition.

Multiple improvements that increase the power and precision of type inference have been made.

-

A (possibly partially instantiated generic) function may now be called with arguments that are
themselves (possibly partially instantiated) generic functions.
The compiler will attempt to infer the missing type arguments of the callee (as before) and,
for each argument that is a generic function that is not fully instantiated,
its missing type arguments (new).
Typical use cases are calls to generic functions operating on containers
(such as [slices.IndexFunc](/pkg/slices#IndexFunc)) where a function argument
may also be generic, and where the type argument of the called function and its arguments
are inferred from the container type.
More generally, a generic function may now be used without explicit instantiation when
it is assigned to a variable or returned as a result value if the type arguments can
be inferred from the assignment.

-

Type inference now also considers methods when a value is assigned to an interface:
type arguments for type parameters used in method signatures may be inferred from
the corresponding parameter types of matching methods.

-

Similarly, since a type argument must implement all the methods of its corresponding constraint,
the methods of the type argument and constraint are matched which may lead to the inference of
additional type arguments.

-

If multiple untyped constant arguments of different kinds (such as an untyped int and
an untyped floating-point constant) are passed to parameters with the same (not otherwise
specified) type parameter type, instead of an error, now type inference determines the
type using the same approach as an operator with untyped constant operands.
This change brings the types inferred from untyped constant arguments in line with the
types of constant expressions.

-

Type inference is now precise when matching corresponding types in assignments:
component types (such as the elements of slices, or the parameter types in function signatures)
must be identical (given suitable type arguments) to match, otherwise inference fails.
This change produces more accurate error messages:
where in the past type inference may have succeeded incorrectly and lead to an invalid assignment,
the compiler now reports an inference error if two types can&rsquo;t possibly match.

More generally, the description of
[type inference](/ref/spec#Type_inference)
in the language spec has been clarified.
Together, all these changes make type inference more powerful and inference failures less surprising.

Go 1.21 includes a preview of a language change we are considering for a future version of Go:
making for loop variables per-iteration instead of per-loop, to avoid accidental sharing bugs.
For details about how to try that language change, see [the LoopvarExperiment wiki page](/wiki/LoopvarExperiment).

Go 1.21 now defines that if a goroutine is panicking and recover was called directly by a deferred
function, the return value of recover is guaranteed not to be nil. To ensure this, calling panic
with a nil interface value (or an untyped nil) causes a run-time panic of type
[`*runtime.PanicNilError`](/pkg/runtime/#PanicNilError).

To support programs written for older versions of Go, nil panics can be re-enabled by setting
`GODEBUG=panicnil=1`.
This setting is enabled automatically when compiling a program whose main package
is in a module that declares `go` `1.20` or earlier.

## Tools

Go 1.21 adds improved support for backwards compatibility and forwards compatibility
in the Go toolchain.

To improve backwards compatibility, Go 1.21 formalizes
Go&rsquo;s use of the GODEBUG environment variable to control
the default behavior for changes that are non-breaking according to the
[compatibility policy](/doc/go1compat)
but nonetheless may cause existing programs to break.
(For example, programs that depend on buggy behavior may break
when a bug is fixed, but bug fixes are not considered breaking changes.)
When Go must make this kind of behavior change,
it now chooses between the old and new behavior based on the
`go` line in the workspace&rsquo;s `go.work` file
or else the main module&rsquo;s `go.mod` file.
Upgrading to a new Go toolchain but leaving the `go` line
set to its original (older) Go version preserves the behavior of the older
toolchain.
With this compatibility support, the latest Go toolchain should always
be the best, most secure, implementation of an older version of Go.
See “[Go, Backwards Compatibility, and GODEBUG](/doc/godebug)” for details.

To improve forwards compatibility, Go 1.21 now reads the `go` line
in a `go.work` or `go.mod` file as a strict
minimum requirement: `go` `1.21.0` means
that the workspace or module cannot be used with Go 1.20 or with Go 1.21rc1.
This allows projects that depend on fixes made in later versions of Go
to ensure that they are not used with earlier versions.
It also gives better error reporting for projects that make use of new Go features:
when the problem is that a newer Go version is needed,
that problem is reported clearly, instead of attempting to build the code
and printing errors about unresolved imports or syntax errors.

To make these new stricter version requirements easier to manage,
the `go` command can now invoke not just the toolchain
bundled in its own release but also other Go toolchain versions found in the PATH
or downloaded on demand.
If a `go.mod` or `go.work` `go` line
declares a minimum requirement on a newer version of Go, the `go`
command will find and run that version automatically.
The new `toolchain` directive sets a suggested minimum toolchain to use,
which may be newer than the strict `go` minimum.
See “[Go Toolchains](/doc/toolchain)” for details.

### Go command

The `-pgo` build flag now defaults to `-pgo=auto`,
and the restriction of specifying a single main package on the command
line is now removed. If a file named `default.pgo` is present
in the main package&rsquo;s directory, the `go` command will use
it to enable profile-guided optimization for building the corresponding
program.

The `-C` `dir` flag must now be the first
flag on the command-line when used.

The new `go` `test` option
`-fullpath` prints full path names in test log messages,
rather than just base names.

The `go` `test` `-c` flag now
supports writing test binaries for multiple packages, each to
`pkg.test` where `pkg` is the package name.
It is an error if more than one test package being compiled has a given package name.

The `go` `test` `-o` flag now
accepts a directory argument, in which case test binaries are written to that
directory instead of the current directory.

When using an external (C) linker with cgo enabled, the `runtime/cgo` package is
now supplied to the Go linker as an additional dependency to ensure that the Go
runtime is compatible with any additional libraries added by the C linker.

### Cgo

In files that `import "C"`, the Go toolchain now
correctly reports errors for attempts to declare Go methods on C types.

## Runtime

When printing very deep stacks, the runtime now prints the first 50
(innermost) frames followed by the bottom 50 (outermost) frames,
rather than just printing the first 100 frames. This makes it easier
to see how deeply recursive stacks started, and is especially
valuable for debugging stack overflows.

On Linux platforms that support transparent huge pages, the Go runtime
now manages which parts of the heap may be backed by huge pages more
explicitly. This leads to better utilization of memory: small heaps
should see less memory used (up to 50% in pathological cases) while
large heaps should see fewer broken huge pages for dense parts of the
heap, improving CPU usage and latency by up to 1%. A consequence of this
change is that the runtime no longer tries to work around a particular
problematic Linux configuration setting, which may result in higher
memory overheads. The recommended fix is to adjust the OS&rsquo;s huge page
settings according to the [GC guide](/doc/gc-guide#Linux_transparent_huge_pages).
However, other workarounds are available as well. See the [section on
`max_ptes_none`](/doc/gc-guide#Linux_THP_max_ptes_none_workaround).

As a result of runtime-internal garbage collection tuning,
applications may see up to a 40% reduction in application tail latency
and a small decrease in memory use. Some applications may also observe
a small loss in throughput.
The memory use decrease should be proportional to the loss in
throughput, such that the previous release&rsquo;s throughput/memory
tradeoff may be recovered (with little change to latency) by
increasing `GOGC` and/or `GOMEMLIMIT` slightly.

Calls from C to Go on threads created in C require some setup to prepare for
Go execution. On Unix platforms, this setup is now preserved across multiple
calls from the same thread. This significantly reduces the overhead of
subsequent C to Go calls from ~1-3 microseconds per call to ~100-200
nanoseconds per call.

## Compiler

Profile-guide optimization (PGO), added as a preview in Go 1.20, is now ready
for general use. PGO enables additional optimizations on code identified as
hot by profiles of production workloads. As mentioned in the
[Go command section](#go-command), PGO is enabled by default for
binaries that contain a `default.pgo` profile in the main
package directory. Performance improvements vary depending on application
behavior, with most programs from a representative set of Go programs seeing
between 2 and 7% improvement from enabling PGO. See the
[PGO user guide](/doc/pgo) for detailed documentation.

PGO builds can now devirtualize some interface method calls, adding a
concrete call to the most common callee. This enables further optimization,
such as inlining the callee.

Go 1.21 improves build speed by up to 6%, largely thanks to building the
compiler itself with PGO.

## Assembler

On amd64, frameless nosplit assembly functions are no longer automatically marked as `NOFRAME`.
Instead, the `NOFRAME` attribute must be explicitly specified if desired,
which is already the behavior on other architectures supporting frame pointers.
With this, the runtime now maintains the frame pointers for stack transitions.

The verifier that checks for incorrect uses of `R15` when dynamic linking on amd64 has been improved.

## Linker

On windows/amd64, the linker (with help from the compiler) now emits
SEH unwinding data by default, which improves the integration
of Go applications with Windows debuggers and other tools.

In Go 1.21 the linker (with help from the compiler) is now capable of
deleting dead (unreferenced) global map variables, if the number of
entries in the variable initializer is sufficiently large, and if the
initializer expressions are side-effect free.

## Standard library

### New log/slog package

The new [log/slog](/pkg/log/slog) package provides structured logging with levels.
Structured logging emits key-value pairs
to enable fast, accurate processing of large amounts of log data.
The package supports integration with popular log analysis tools and services.

### New testing/slogtest package

The new [testing/slogtest](/pkg/testing/slogtest) package can help
to validate [slog.Handler](/pkg/log/slog#Handler) implementations.

### New slices package

The new [slices](/pkg/slices) package provides many common
operations on slices, using generic functions that work with slices
of any element type.

### New maps package

The new [maps](/pkg/maps/) package provides several
common operations on maps, using generic functions that work with
maps of any key or element type.

### New cmp package

The new [cmp](/pkg/cmp/) package defines the type
constraint [`Ordered`](/pkg/cmp/#Ordered) and
two new generic functions
[`Less`](/pkg/cmp/#Less)
and [`Compare`](/pkg/cmp/#Compare) that are
useful with [ordered
types](/ref/spec/#Comparison_operators).

### Minor changes to the library

As always, there are various minor changes and updates to the library,
made with the Go 1 [promise of compatibility](/doc/go1compat)
in mind.
There are also various performance improvements, not enumerated here.

#### [archive/tar](/pkg/archive/tar/)

The implementation of the
[`io/fs.FileInfo`](/pkg/io/fs/#FileInfo)
interface returned by
[`Header.FileInfo`](/pkg/archive/tar/#Header.FileInfo)
now implements a `String` method that calls
[`io/fs.FormatFileInfo`](/pkg/io/fs/#FormatFileInfo).

#### [archive/zip](/pkg/archive/zip/)

The implementation of the
[`io/fs.FileInfo`](/pkg/io/fs/#FileInfo)
interface returned by
[`FileHeader.FileInfo`](/pkg/archive/zip/#FileHeader.FileInfo)
now implements a `String` method that calls
[`io/fs.FormatFileInfo`](/pkg/io/fs/#FormatFileInfo).

The implementation of the
[`io/fs.DirEntry`](/pkg/io/fs/#DirEntry)
interface returned by the
[`io/fs.ReadDirFile.ReadDir`](/pkg/io/fs/#ReadDirFile.ReadDir)
method of the
[`io/fs.File`](/pkg/io/fs/#File)
returned by
[`Reader.Open`](/pkg/archive/zip/#Reader.Open)
now implements a `String` method that calls
[`io/fs.FormatDirEntry`](/pkg/io/fs/#FormatDirEntry).

#### [bytes](/pkg/bytes/)

The [`Buffer`](/pkg/bytes/#Buffer) type
has two new methods:
[`Available`](/pkg/bytes/#Buffer.Available)
and [`AvailableBuffer`](/pkg/bytes/#Buffer.AvailableBuffer).
These may be used along with the
[`Write`](/pkg/bytes/#Buffer.Write)
method to append directly to the `Buffer`.

#### [context](/pkg/context/)

The new [`WithoutCancel`](/pkg/context/#WithoutCancel)
function returns a copy of a context that is not canceled when the original
context is canceled.

The new [`WithDeadlineCause`](/pkg/context/#WithDeadlineCause)
and [`WithTimeoutCause`](/pkg/context/#WithTimeoutCause)
functions provide a way to set a context cancellation cause when a deadline or
timer expires. The cause may be retrieved with the
[`Cause`](/pkg/context/#Cause) function.

The new [`AfterFunc`](/pkg/context/#AfterFunc)
function registers a function to run after a context has been cancelled.

An optimization means that the results of calling
[`Background`](/pkg/context/#Background)
and [`TODO`](/pkg/context/#TODO) and
converting them to a shared type can be considered equal.
In previous releases they were always different. Comparing
[`Context`](/pkg/context/#Context) values
for equality has never been well-defined, so this is not
considered to be an incompatible change.

#### [crypto/ecdsa](/pkg/crypto/ecdsa/)

[`PublicKey.Equal`](/pkg/crypto/ecdsa/#PublicKey.Equal) and
[`PrivateKey.Equal`](/pkg/crypto/ecdsa/#PrivateKey.Equal)
now execute in constant time.

#### [crypto/elliptic](/pkg/crypto/elliptic/)

All of the [`Curve`](/pkg/crypto/elliptic/#Curve) methods have been deprecated, along with [`GenerateKey`](/pkg/crypto/elliptic/#GenerateKey), [`Marshal`](/pkg/crypto/elliptic/#Marshal), and [`Unmarshal`](/pkg/crypto/elliptic/#Unmarshal). For ECDH operations, the new [`crypto/ecdh`](/pkg/crypto/ecdh/) package should be used instead. For lower-level operations, use third-party modules such as [filippo.io/nistec](https://pkg.go.dev/filippo.io/nistec).

#### [crypto/rand](/pkg/crypto/rand/)

The [`crypto/rand`](/pkg/crypto/rand/) package now uses the `getrandom` system call on NetBSD 10.0 and later.

#### [crypto/rsa](/pkg/crypto/rsa/)

The performance of private RSA operations (decryption and signing) is now better than Go 1.19 for `GOARCH=amd64` and `GOARCH=arm64`. It had regressed in Go 1.20.

Due to the addition of private fields to [`PrecomputedValues`](/pkg/crypto/rsa/#PrecomputedValues), [`PrivateKey.Precompute`](/pkg/crypto/rsa/#PrivateKey.Precompute) must be called for optimal performance even if deserializing (for example from JSON) a previously-precomputed private key.

[`PublicKey.Equal`](/pkg/crypto/rsa/#PublicKey.Equal) and
[`PrivateKey.Equal`](/pkg/crypto/rsa/#PrivateKey.Equal)
now execute in constant time.

The [`GenerateMultiPrimeKey`](/pkg/crypto/rsa/#GenerateMultiPrimeKey) function and the [`PrecomputedValues.CRTValues`](/pkg/crypto/rsa/#PrecomputedValues.CRTValues) field have been deprecated. [`PrecomputedValues.CRTValues`](/pkg/crypto/rsa/#PrecomputedValues.CRTValues) will still be populated when [`PrivateKey.Precompute`](/pkg/crypto/rsa/#PrivateKey.Precompute) is called, but the values will not be used during decryption operations.

#### [crypto/sha256](/pkg/crypto/sha256/)

SHA-224 and SHA-256 operations now use native instructions when available when `GOARCH=amd64`, providing a performance improvement on the order of 3-4x.

#### [crypto/tls](/pkg/crypto/tls/)

Servers now skip verifying client certificates (including not running
[`Config.VerifyPeerCertificate`](/pkg/crypto/tls/#Config.VerifyPeerCertificate))
for resumed connections, besides checking the expiration time. This makes
session tickets larger when client certificates are in use. Clients were
already skipping verification on resumption, but now check the expiration
time even if [`Config.InsecureSkipVerify`](/pkg/crypto/tls/#Config.InsecureSkipVerify)
is set.

Applications can now control the content of session tickets.

- The new [`SessionState`](/pkg/crypto/tls/#SessionState) type
describes a resumable session.

- The [`SessionState.Bytes`](/pkg/crypto/tls/#SessionState.Bytes)
method and [`ParseSessionState`](/pkg/crypto/tls/#ParseSessionState)
function serialize and deserialize a `SessionState`.

- The [`Config.WrapSession`](/pkg/crypto/tls/#Config.WrapSession) and
[`Config.UnwrapSession`](/pkg/crypto/tls/#Config.UnwrapSession)
hooks convert a `SessionState` to and from a ticket on the server side.

- The [`Config.EncryptTicket`](/pkg/crypto/tls/#Config.EncryptTicket)
and [`Config.DecryptTicket`](/pkg/crypto/tls/#Config.DecryptTicket)
methods provide a default implementation of `WrapSession` and
`UnwrapSession`.

- The [`ClientSessionState.ResumptionState`](/pkg/crypto/tls/#ClientSessionState.ResumptionState) method and
[`NewResumptionState`](/pkg/crypto/tls/#NewResumptionState) function
may be used by a `ClientSessionCache` implementation to store and
resume sessions on the client side.

To reduce the potential for session tickets to be used as a tracking
mechanism across connections, the server now issues new tickets on every
resumption (if they are supported and not disabled) and tickets don&rsquo;t bear
an identifier for the key that encrypted them anymore. If passing a large
number of keys to [`Conn.SetSessionTicketKeys`](/pkg/crypto/tls/#Conn.SetSessionTicketKeys),
this might lead to a noticeable performance cost.

Both clients and servers now implement the Extended Master Secret extension (RFC 7627).
The deprecation of [`ConnectionState.TLSUnique`](/pkg/crypto/tls/#ConnectionState.TLSUnique)
has been reverted, and is now set for resumed connections that support Extended Master Secret.

The new [`QUICConn`](/pkg/crypto/tls/#QUICConn) type
provides support for QUIC implementations, including 0-RTT support. Note
that this is not itself a QUIC implementation, and 0-RTT is still not
supported in TLS.

The new [`VersionName`](/pkg/crypto/tls/#VersionName) function
returns the name for a TLS version number.

The TLS alert codes sent from the server for client authentication failures have
been improved. Previously, these failures always resulted in a &ldquo;bad certificate&rdquo; alert.
Now, certain failures will result in more appropriate alert codes,
as defined by RFC 5246 and RFC 8446:

- For TLS 1.3 connections, if the server is configured to require client authentication using
[RequireAnyClientCert](/pkg/crypto/tls/#RequireAnyClientCert) or
[RequireAndVerifyClientCert](/pkg/crypto/tls/#RequireAndVerifyClientCert),
and the client does not provide any certificate, the server will now return the &ldquo;certificate required&rdquo; alert.

- If the client provides a certificate that is not signed by the set of trusted certificate authorities
configured on the server, the server will return the &ldquo;unknown certificate authority&rdquo; alert.

- If the client provides a certificate that is either expired or not yet valid,
the server will return the &ldquo;expired certificate&rdquo; alert.

- In all other scenarios related to client authentication failures, the server still returns &ldquo;bad certificate&rdquo;.

#### [crypto/x509](/pkg/crypto/x509/)

[`RevocationList.RevokedCertificates`](/pkg/crypto/x509/#RevocationList.RevokedCertificates) has been deprecated and replaced with the new [`RevokedCertificateEntries`](/pkg/crypto/x509/#RevocationList.RevokedCertificateEntries) field, which is a slice of [`RevocationListEntry`](/pkg/crypto/x509/#RevocationListEntry). [`RevocationListEntry`](/pkg/crypto/x509/#RevocationListEntry) contains all of the fields in [`pkix.RevokedCertificate`](/pkg/crypto/x509/pkix#RevokedCertificate), as well as the revocation reason code.

Name constraints are now correctly enforced on non-leaf certificates, and
not on the certificates where they are expressed.

#### [debug/elf](/pkg/debug/elf/)

The new
[`File.DynValue`](/pkg/debug/elf/#File.DynValue)
method may be used to retrieve the numeric values listed with a
given dynamic tag.

The constant flags permitted in a `DT_FLAGS_1`
dynamic tag are now defined with type
[`DynFlag1`](/pkg/debug/elf/#DynFlag1). These
tags have names starting with `DF_1`.

The package now defines the constant
[`COMPRESS_ZSTD`](/pkg/debug/elf/#COMPRESS_ZSTD).

The package now defines the constant
[`R_PPC64_REL24_P9NOTOC`](/pkg/debug/elf/#R_PPC64_REL24_P9NOTOC).

#### [debug/pe](/pkg/debug/pe/)

Attempts to read from a section containing uninitialized data
using
[`Section.Data`](/pkg/debug/pe/#Section.Data)
or the reader returned by [`Section.Open`](/pkg/debug/pe/#Section.Open)
now return an error.

#### [embed](/pkg/embed/)

The [`io/fs.File`](/pkg/io/fs/#File)
returned by
[`FS.Open`](/pkg/embed/#FS.Open) now
has a `ReadAt` method that
implements [`io.ReaderAt`](/pkg/io/#ReaderAt).

Calling `[FS.Open](/pkg/embed/FS.Open).[Stat](/pkg/io/fs/#File.Stat)`
will return a type that now implements a `String`
method that calls
[`io/fs.FormatFileInfo`](/pkg/io/fs/#FormatFileInfo).

#### [encoding/binary](/pkg/encoding/binary/)

The new
[`NativeEndian`](/pkg/encoding/binary/#NativeEndian)
variable may be used to convert between byte slices and integers
using the current machine&rsquo;s native endianness.

#### [errors](/pkg/errors/)

The new
[`ErrUnsupported`](/pkg/errors/#ErrUnsupported)
error provides a standardized way to indicate that a requested
operation may not be performed because it is unsupported.
For example, a call to
[`os.Link`](/pkg/os/#Link) when using a
file system that does not support hard links.

#### [flag](/pkg/flag/)

The new [`BoolFunc`](/pkg/flag/#BoolFunc)
function and
[`FlagSet.BoolFunc`](/pkg/flag/#FlagSet.BoolFunc)
method define a flag that does not require an argument and calls
a function when the flag is used. This is similar to
[`Func`](/pkg/flag/#Func) but for a
boolean flag.

A flag definition
(via [`Bool`](/pkg/flag/#Bool),
[`BoolVar`](/pkg/flag/#BoolVar),
[`Int`](/pkg/flag/#Int),
[`IntVar`](/pkg/flag/#IntVar), etc.)
will panic if [`Set`](/pkg/flag/#Set) has
already been called on a flag with the same name. This change is
intended to detect cases where [changes in
initialization order](#language) cause flag operations to occur in a
different order than expected. In many cases the fix to this
problem is to introduce a explicit package dependence to
correctly order the definition before any
[`Set`](/pkg/flag/#Set) operations.

#### [go/ast](/pkg/go/ast/)

The new [`IsGenerated`](/pkg/go/ast/#IsGenerated) predicate
reports whether a file syntax tree contains the
[special comment](/s/generatedcode)
that conventionally indicates that the file was generated by a tool.

The new
[`File.GoVersion`](/pkg/go/ast/#File.GoVersion)
field records the minimum Go version required by
any `//go:build` or `// +build`
directives.

#### [go/build](/pkg/go/build/)

The package now parses build directives (comments that start
with `//go:`) in file headers (before
the `package` declaration). These directives are
available in the new
[`Package`](/pkg/go/build#Package) fields
[`Directives`](/pkg/go/build#Package.Directives),
[`TestDirectives`](/pkg/go/build#Package.TestDirectives),
and
[`XTestDirectives`](/pkg/go/build#Package.XTestDirectives).

#### [go/build/constraint](/pkg/go/build/constraint/)

The new
[`GoVersion`](/pkg/go/build/constraint/#GoVersion)
function returns the minimum Go version implied by a build
expression.

#### [go/token](/pkg/go/token/)

The new [`File.Lines`](/pkg/go/token/#File.Lines) method
returns the file&rsquo;s line-number table in the same form as accepted by
`File.SetLines`.

#### [go/types](/pkg/go/types/)

The new [`Package.GoVersion`](/pkg/go/types/#Package.GoVersion)
method returns the Go language version used to check the package.

#### [hash/maphash](/pkg/hash/maphash/)

The `hash/maphash` package now has a pure Go implementation, selectable with the `purego` build tag.

#### [html/template](/pkg/html/template/)

The new error
[`ErrJSTemplate`](/pkg/html/template/#ErrJSTemplate)
is returned when an action appears in a JavaScript template
literal. Previously an unexported error was returned.

#### [io/fs](/pkg/io/fs/)

The new
[`FormatFileInfo`](/pkg/io/fs/#FormatFileInfo)
function returns a formatted version of a
[`FileInfo`](/pkg/io/fs/#FileInfo).
The new
[`FormatDirEntry`](/pkg/io/fs/#FormatDirEntry)
function returns a formatted version of a
[`DirEntry`](/pkg/io/fs/#FileInfo).
The implementation of
[`DirEntry`](/pkg/io/fs/#DirEntry)
returned by
[`ReadDir`](/pkg/io/fs/#ReadDir) now
implements a `String` method that calls
[`FormatDirEntry`](/pkg/io/fs/#FormatDirEntry),
and the same is true for
the [`DirEntry`](/pkg/io/fs/#DirEntry)
value passed to
[`WalkDirFunc`](/pkg/io/fs/#WalkDirFunc).

#### [math/big](/pkg/math/big/)

The new [`Int.Float64`](/pkg/math/big/#Int.Float64)
method returns the nearest floating-point value to a
multi-precision integer, along with an indication of any
rounding that occurred.

#### [net](/pkg/net/)

On Linux, the [net](/pkg/net/) package can now use
Multipath TCP when the kernel supports it. It is not used by
default. To use Multipath TCP when available on a client, call
the
[`Dialer.SetMultipathTCP`](/pkg/net/#Dialer.SetMultipathTCP)
method before calling the
[`Dialer.Dial`](/pkg/net/#Dialer.Dial) or
[`Dialer.DialContext`](/pkg/net/#Dialer.DialContext)
methods. To use Multipath TCP when available on a server, call
the
[`ListenConfig.SetMultipathTCP`](/pkg/net/#ListenConfig.SetMultipathTCP)
method before calling the
[`ListenConfig.Listen`](/pkg/net/#ListenConfig.Listen)
method. Specify the network as `"tcp"` or
`"tcp4"` or `"tcp6"` as usual. If
Multipath TCP is not supported by the kernel or the remote host,
the connection will silently fall back to TCP. To test whether a
particular connection is using Multipath TCP, use the
[`TCPConn.MultipathTCP`](/pkg/net/#TCPConn.MultipathTCP)
method.

In a future Go release we may enable Multipath TCP by default on
systems that support it.

#### [net/http](/pkg/net/http/)

The new [`ResponseController.EnableFullDuplex`](/pkg/net/http#ResponseController.EnableFullDuplex)
method allows server handlers to concurrently read from an HTTP/1
request body while writing the response. Normally, the HTTP/1 server
automatically consumes any remaining request body before starting to
write the response, to avoid deadlocking clients which attempt to
write a complete request before reading the response. The
`EnableFullDuplex` method disables this behavior.

The new [`ErrSchemeMismatch`](/pkg/net/http/#ErrSchemeMismatch) error is returned by [`Client`](/pkg/net/http/#Client) and [`Transport`](/pkg/net/http/#Transport) when the server responds to an HTTPS request with an HTTP response.

The [net/http](/pkg/net/http/) package now supports
[`errors.ErrUnsupported`](/pkg/errors/#ErrUnsupported),
in that the expression
`errors.Is(http.ErrNotSupported, errors.ErrUnsupported)`
will return true.

#### [os](/pkg/os/)

Programs may now pass an empty `time.Time` value to
the [`Chtimes`](/pkg/os/#Chtimes) function
to leave either the access time or the modification time unchanged.

On Windows the
[`File.Chdir`](/pkg/os#File.Chdir) method
now changes the current directory to the file, rather than
always returning an error.

On Unix systems, if a non-blocking descriptor is passed
to [`NewFile`](/pkg/os/#NewFile), calling
the [`File.Fd`](/pkg/os/#File.Fd) method
will now return a non-blocking descriptor. Previously the
descriptor was converted to blocking mode.

On Windows calling
[`Truncate`](/pkg/os/#Truncate) on a
non-existent file used to create an empty file. It now returns
an error indicating that the file does not exist.

On Windows calling
[`TempDir`](/pkg/os/#TempDir) now uses
GetTempPath2W when available, instead of GetTempPathW. The
new behavior is a security hardening measure that prevents
temporary files created by processes running as SYSTEM to
be accessed by non-SYSTEM processes.

On Windows the os package now supports working with files whose
names, stored as UTF-16, can&rsquo;t be represented as valid UTF-8.

On Windows [`Lstat`](/pkg/os/#Lstat) now resolves
symbolic links for paths ending with a path separator, consistent with its
behavior on POSIX platforms.

The implementation of the
[`io/fs.DirEntry`](/pkg/io/fs/#DirEntry)
interface returned by the
[`ReadDir`](/pkg/os/#ReadDir) function and
the [`File.ReadDir`](/pkg/os/#File.ReadDir)
method now implements a `String` method that calls
[`io/fs.FormatDirEntry`](/pkg/io/fs/#FormatDirEntry).

The implementation of the
[`io/fs.FS`](/pkg/io/fs/#FS) interface returned by
the [`DirFS`](/pkg/os/#DirFS) function now implements
the [`io/fs.ReadFileFS`](/pkg/io/fs/#ReadFileFS) and
the [`io/fs.ReadDirFS`](/pkg/io/fs/#ReadDirFS)
interfaces.

#### [path/filepath](/pkg/path/filepath/)

The implementation of the
[`io/fs.DirEntry`](/pkg/io/fs/#DirEntry)
interface passed to the function argument of
[`WalkDir`](/pkg/path/filepath/#WalkDir)
now implements a `String` method that calls
[`io/fs.FormatDirEntry`](/pkg/io/fs/#FormatDirEntry).

#### [reflect](/pkg/reflect/)

In Go 1.21, [`ValueOf`](/pkg/reflect/#ValueOf)
no longer forces its argument to be allocated on the heap, allowing
a `Value`&rsquo;s content to be allocated on the stack. Most
operations on a `Value` also allow the underlying value
to be stack allocated.

The new [`Value`](/pkg/reflect/#Value)
method [`Value.Clear`](/pkg/reflect/#Value.Clear)
clears the contents of a map or zeros the contents of a slice.
This corresponds to the new `clear` built-in
[added to the language](#language).

The [`SliceHeader`](/pkg/reflect/#SliceHeader)
and [`StringHeader`](/pkg/reflect/#StringHeader)
types are now deprecated. In new code
prefer [`unsafe.Slice`](/pkg/unsafe/#Slice),
[`unsafe.SliceData`](/pkg/unsafe/#SliceData),
[`unsafe.String`](/pkg/unsafe/#String),
or [`unsafe.StringData`](/pkg/unsafe/#StringData).

#### [regexp](/pkg/regexp/)

[`Regexp`](/pkg/regexp#Regexp) now defines
[`MarshalText`](/pkg/regexp#Regexp.MarshalText)
and [`UnmarshalText`](/pkg/regexp#Regexp.UnmarshalText)
methods. These implement
[`encoding.TextMarshaler`](/pkg/encoding#TextMarshaler)
and
[`encoding.TextUnmarshaler`](/pkg/encoding#TextUnmarshaler)
and will be used by packages such as
[encoding/json](/pkg/encoding/json).

#### [runtime](/pkg/runtime/)

Textual stack traces produced by Go programs, such as those
produced when crashing, calling `runtime.Stack`, or
collecting a goroutine profile with `debug=2`, now
include the IDs of the goroutines that created each goroutine in
the stack trace.

Crashing Go applications can now opt-in to Windows Error Reporting (WER) by setting the environment variable
`GOTRACEBACK=wer` or calling [`debug.SetTraceback("wer")`](/pkg/runtime/debug/#SetTraceback)
before the crash. Other than enabling WER, the runtime will behave as with `GOTRACEBACK=crash`.
On non-Windows systems, `GOTRACEBACK=wer` is ignored.

`GODEBUG=cgocheck=2`, a thorough checker of cgo pointer passing rules,
is no longer available as a [debug option](/pkg/runtime#hdr-Environment_Variables).
Instead, it is available as an experiment using `GOEXPERIMENT=cgocheck2`.
In particular this means that this mode has to be selected at build time instead of startup time.

`GODEBUG=cgocheck=1` is still available (and is still the default).

A new type `Pinner` has been added to the runtime
package. `Pinner`s may be used to &ldquo;pin&rdquo; Go memory
such that it may be used more freely by non-Go code. For instance,
passing Go values that reference pinned Go memory to C code is
now allowed. Previously, passing any such nested reference was
disallowed by the
[cgo pointer passing rules.](https://pkg.go.dev/cmd/cgo#hdr-Passing_pointers)
See [the docs](/pkg/runtime#Pinner) for more details.

#### [runtime/metrics](/pkg/runtime/metrics/)

A few previously-internal GC metrics, such as live heap size, are
now available.
`GOGC` and `GOMEMLIMIT` are also now
available as metrics.

#### [runtime/trace](/pkg/runtime/trace/)

Collecting traces on amd64 and arm64 now incurs a substantially
smaller CPU cost: up to a 10x improvement over the previous release.

Traces now contain explicit stop-the-world events for every reason
the Go runtime might stop-the-world, not just garbage collection.

#### [sync](/pkg/sync/)

The new [`OnceFunc`](/pkg/sync/#OnceFunc),
[`OnceValue`](/pkg/sync/#OnceValue), and
[`OnceValues`](/pkg/sync/#OnceValues)
functions capture a common use of [Once](/pkg/sync/#Once) to
lazily initialize a value on first use.

#### [syscall](/pkg/syscall/)

On Windows the
[`Fchdir`](/pkg/syscall#Fchdir) function
now changes the current directory to its argument, rather than
always returning an error.

On FreeBSD
[`SysProcAttr`](/pkg/syscall#SysProcAttr)
has a new field `Jail` that may be used to put the
newly created process in a jailed environment.

On Windows the syscall package now supports working with files whose
names, stored as UTF-16, can&rsquo;t be represented as valid UTF-8.
The [`UTF16ToString`](/pkg/syscall#UTF16ToString)
and [`UTF16FromString`](/pkg/syscall#UTF16FromString)
functions now convert between UTF-16 data and
[WTF-8](https://simonsapin.github.io/wtf-8/) strings.
This is backward compatible as WTF-8 is a superset of the UTF-8
format that was used in earlier releases.

Several error values match the new
[`errors.ErrUnsupported`](/pkg/errors/#ErrUnsupported),
such that `errors.Is(err, errors.ErrUnsupported)`
returns true.

- `ENOSYS`

- `ENOTSUP`

- `EOPNOTSUPP`

- `EPLAN9` (Plan 9 only)

- `ERROR_CALL_NOT_IMPLEMENTED` (Windows only)

- `ERROR_NOT_SUPPORTED` (Windows only)

- `EWINDOWS` (Windows only)

#### [testing](/pkg/testing/)

The new `-test.fullpath` option will print full path
names in test log messages, rather than just base names.

The new [`Testing`](/pkg/testing/#Testing) function reports whether the program is a test created by `go` `test`.

#### [testing/fstest](/pkg/testing/fstest/)

Calling `[Open](/pkg/testing/fstest/MapFS.Open).[Stat](/pkg/io/fs/#File.Stat)`
will return a type that now implements a `String`
method that calls
[`io/fs.FormatFileInfo`](/pkg/io/fs/#FormatFileInfo).

#### [unicode](/pkg/unicode/)

The [`unicode`](/pkg/unicode/) package and
associated support throughout the system has been upgraded to
[Unicode 15.0.0](https://www.unicode.org/versions/Unicode15.0.0/).

## Ports

### Darwin

As [announced](go1.20#darwin) in the Go 1.20 release notes,
Go 1.21 requires macOS 10.15 Catalina or later;
support for previous versions has been discontinued.

### Windows

As [announced](go1.20#windows) in the Go 1.20 release notes,
Go 1.21 requires at least Windows 10 or Windows Server 2016;
support for previous versions has been discontinued.

### ARM

When building the Go distribution with `GOARCH=arm` when not running
on an ARM system (that is, when building a cross-compiler to ARM), the
default value for the `GOARM` environment variable is now always set
to `7`.
Previously the default depended on characteristics of the build
system.

When not building a cross-compiler, the default value is determined
by examining the build system.
That was true before and remains true in Go 1.21.
What has changed is the behavior when building a cross-compiler.

### WebAssembly

The new `go:wasmimport` directive can now be used in Go programs
to import functions from the WebAssembly host.

The Go scheduler now interacts much more efficiently with the
JavaScript event loop, especially in applications that block
frequently on asynchronous events.

### WebAssembly System Interface

Go 1.21 adds an experimental port to the [
WebAssembly System Interface (WASI)](https://wasi.dev/), Preview 1
(`GOOS=wasip1`, `GOARCH=wasm`).

As a result of the addition of the new `GOOS` value
&ldquo;`wasip1`&rdquo;, Go files named `*_wasip1.go`
will now be [ignored
by Go tools](/pkg/go/build/#hdr-Build_Constraints) except when that `GOOS` value is being
used.
If you have existing filenames matching that pattern, you will
need to rename them.

### ppc64/ppc64le

On Linux, `GOPPC64=power10` now generates PC-relative instructions, prefixed
instructions, and other new Power10 instructions. On AIX, `GOPPC64=power10`
generates Power10 instructions, but does not generate PC-relative instructions.

When building position-independent binaries for `GOPPC64=power10`
`GOOS=linux` `GOARCH=ppc64le`, users can expect reduced binary
sizes in most cases, in some cases 3.5%. Position-independent binaries are built for
ppc64le with the following `-buildmode` values:
`c-archive`, `c-shared`, `shared`, `pie`, `plugin`.

### loong64

The `linux/loong64` port now supports `-buildmode=c-archive`,
`-buildmode=c-shared` and `-buildmode=pie`.

## Patch Release Changelogs

### go1.21.1

```text
- `1755d14` [release-branch.go1.21] cmd/compile: fix missing init nodes for len(string([]byte)) optimization (2023-08-05)
- `75d8be5` [release-branch.go1.21] cmd/go/internal/web: release the net token when an HTTP request fails due to CheckRedirect (2023-08-09)
- `9398951` [release-branch.go1.21] cmd/distpack: include directory entries in tar files (2023-08-10)
- `179821c` [release-branch.go1.21] net/http: permit requests with invalid Host headers (2023-07-19)
- `3475e6a` [release-branch.go1.21] cmd/go: fix missing case checking for empty slice (2023-08-07)
- `b78e8cc` [release-branch.go1.21] crypto/tls: add GODEBUG to control max RSA key size (2023-08-09)
- `ed527ec` [release-branch.go1.21] cmd/api: rename api.go to main_test.go (2023-08-16)
- `7437db1` [release-branch.go1.21] go/types, types2: use exact unification when comparing interface methods (2023-08-14)
- `d91843f` [release-branch.go1.21] go/types, types2: use correct parameter list when checking argument passing (2023-08-14)
- `4e34f2e` [release-branch.go1.21] go/types, types2: don't panic during interface completion (2023-08-03)
- `25c6dce` [release-branch.go1.21] cmd/compile: make backingArrayPtrLen to return typecheck-ed nodes (2023-08-09)
- `6634ce2` [release-branch.go1.21] runtime: profiling on Darwin cannot use blocking reads (2023-08-11)
- `25ec110` [release-branch.go1.21] cmd/compile: ensure empty blocks in write barriers are marked unpreemptible (2023-08-11)
- `31c5a23` [release-branch.go1.21] runtime: mark traceEnabled and traceShuttingDown as no:split (2023-08-13)
- `70aa116` [release-branch.go1.21] runtime/internal/wasitest: skip racy TCP echo test (2023-08-16)
- `bac083a` [release-branch.go1.21] cmd/link: don't mangle string symbol names (2023-08-18)
- `45b98bf` [release-branch.go1.21] path/filepath: don't drop .. elements when cleaning invalid Windows paths (2023-08-08)
- `cb6ea94` [release-branch.go1.21] Revert "cmd/compile: omit redundant sign/unsign extension on arm64" (2023-08-18)
- `7c97cc7` [release-branch.go1.21] Revert "os: use handle based APIs to read directories on windows" (2023-08-16)
- `2b8026f` [release-branch.go1.21] cmd/compile: in expandCalls, move all arg marshalling into call block (2023-08-15)
- `2d4746f` [release-branch.go1.21] go/types, types2: disable interface inference for versions before Go 1.21 (2023-08-15)
- `2977709` [release-branch.go1.21] context: fix synchronization in ExampleAfterFunc_cond (2023-08-21)
- `13339c7` [release-branch.go1.21] runtime: fix maps.Clone bug when cloning a map mid-grow (2023-08-23)
- `745b81b` [release-branch.go1.21] encoding/gob: prevent panic from index out of range in Decoder.typeString (2023-08-18)
- `2d07bb8` [release-branch.go1.21] encoding/xml: overriding by empty namespace when no new name declaration (2023-08-09)
- `6385a6f` [release-branch.go1.21] cmd/go: find GOROOT using os.Executable when installed to GOROOT/bin/GOOS_GOARCH (2023-08-18)
- `91a4e74` [release-branch.go1.21] crypto/tls: QUIC: fix panics when processing post-handshake messages (2023-08-24)
- `0a95821` [release-branch.go1.21] cmd/go: retry ETXTBSY errors when running test binaries (2023-08-22)
- `b120517` [release-branch.go1.21] go/types, types2: remove order dependency in inference involving channels (2023-08-21)
- `06df329` [release-branch.go1.21] runtime: avoid MADV_HUGEPAGE for heap memory (2023-08-07)
- `8dc6ad1` [release-branch.go1.21] runtime: restore caller's frame pointer when recovering from panic (2023-08-04)
- `e3ba569` [release-branch.go1.21] net/http: revert "support streaming POST content in wasm" (2023-08-25)
- `d25a935` [release-branch.go1.21] cmd/go: reject toolchain directives containing path separators (2023-08-17)
- `b0e1d3e` [release-branch.go1.21] html/template: support HTML-like comments in script contexts (2023-08-03)
- `bbd043f` [release-branch.go1.21] html/template: properly handle special tags within the script context (2023-08-03)
- `2c1e5b0` [release-branch.go1.21] go1.21.1 (2023-09-06)
```

### go1.21.2

```text
- `d7a0626` [release-branch.go1.21] runtime: ignore SPWrite on innermost traceback frame (2023-09-05)
- `cd66ca0` [release-branch.go1.21] runtime/metrics: fix /gc/scan/* metrics (2023-09-05)
- `b64dc5f` [release-branch.go1.21] cmd/go: in TestScript/gotoolchain_path, remove the user's PATH (2023-09-18)
- `556e9c3` [release-branch.go1.21] cmd/link: avoid deadcode of global map vars for programs using plugins (2023-09-06)
- `3ef4f93` [release-branch.go1.21] cmd/link: force old Apple linker in plugin mode (2023-09-12)
- `9bec49c` [release-branch.go1.21] cmd/link: disable DWARF by default in c-shared mode on darwin (2023-09-12)
- `fc57cc3` [release-branch.go1.21] cmd/link: suppress -bind_at_load deprecation warning for ld-prime (2023-07-10)
- `cd671a1` [release-branch.go1.21] runtime: increase g0 stack size in non-cgo case (2023-09-08)
- `0b6b0a2` [release-branch.go1.21] runtime: always lock OS thread in debugcall (2023-08-03)
- `41d71a5` [release-branch.go1.21] cmd/compile: reset memcombine correctly between basic blocks (2023-09-12)
- `a15ef1b` [release-branch.go1.21] cmd/compile: absorb InvertFlags into Noov comparisons (2023-09-06)
- `2ddfc04` [release-branch.go1.21] cmd/compile: use absolute file name in isCgo check (2023-09-20)
- `26b5783` [release-branch.go1.21] go1.21.2 (2023-10-05)
```

### go1.21.3

```text
- `24ae2d9` [release-branch.go1.21] net/http: regenerate h2_bundle.go (2023-10-06)
- `883f062` [release-branch.go1.21] go1.21.3 (2023-10-10)
```

### go1.21.4

```text
- `9465990` [release-branch.go1.21] all: tidy dependency versioning after release (2023-10-10)
- `236c07c` [release-branch.go1.21] cmd/link: split text sections for arm 32-bit (2023-09-29)
- `bae0152` [release-branch.go1.21] go/types, types2: don't implicitly modify an argument function's type (2023-10-02)
- `ef6993f` [release-branch.go1.21] runtime: don't eagerly collapse hugepages (2023-09-29)
- `64b6c48` [release-branch.go1.21] go/types, types2: don't use generics (2023-10-13)
- `f9a31cd` [release-branch.go1.21] cmd/compile/internal/typecheck: fix closure field naming (2023-09-09)
- `7b04d81` [release-branch.go1.21] runtime/cgo: avoid taking the address of crosscall2 in code (2023-10-07)
- `434af85` [release-branch.go1.21] net/http: pull http2 underflow fix from x/net/http2 (2023-10-26)
- `9e933c1` [release-branch.go1.21] path/filepath: fix various issues in parsing Windows paths (2023-09-01)
- `ed817f1` [release-branch.go1.21] go1.21.4 (2023-11-07)
```

### go1.21.5

```text
- `1e91861` [release-branch.go1.21] syscall: copy rlimit.go's build constraint to rlimit_test.go (2023-11-02)
- `caacf3a` [release-branch.go1.21] cmd/compile: handle constant pointer offsets in dead store elimination (2023-10-30)
- `b9f245b` [release-branch.go1.21] cmd/compile: ensure pointer arithmetic happens after the nil check (2023-10-25)
- `8ae493b` [release-branch.go1.21] internal/poll: add SPLICE_F_NONBLOCK flag for splice to avoid inconsistency with O_NONBLOCK (2023-10-17)
- `f26fa05` [release-branch.go1.21] os: report IO_REPARSE_TAG_DEDUP files as regular in Stat and Lstat (2023-10-26)
- `4952f41` [release-branch.go1.21] cmd/go: allow 'go mod download' to switch toolchains if called with explicit arguments (2023-08-30)
- `23c943e` [release-branch.go1.21] cmd/go/internal/vcs: error out if the requested repo does not support a secure protocol (2023-11-02)
- `8caf4bb` [release-branch.go1.21] path/filepath: consider \\?\c: as a volume on Windows (2023-11-09)
- `9e43850` [release-branch.go1.21] crypto/rand,runtime: switch RtlGenRandom for ProcessPrng (2023-10-18)
- `3684d19` [release-branch.go1.21] runtime: call enableMetadataHugePages and its callees on the systemstack (2023-11-10)
- `f7a79cb` [release-branch.go1.21] cmd/compile: fix findIndVar so it does not match disjointed loop headers (2023-11-05)
- `ec8c526` [release-branch.go1.21] net/http: limit chunked data overhead (2023-11-07)
- `6018ad9` [release-branch.go1.21] go1.21.5 (2023-12-05)
```

### go1.21.6

```text
- `8c840b1` [release-branch.go1.21] cmd/compile: fix memcombine pass for big endian, > 1 byte elements (2023-11-30)
- `9f8b3ac` [release-branch.go1.21] cmd/compile: fix escape analysis of string min/max (2023-12-05)
- `7dc67e8` [release-branch.go1.21] os/signal: remove go t.Run from TestNohup (2023-11-01)
- `bbab863` [release-branch.go1.21] os/signal: skip nohup tests on darwin builders (2023-11-01)
- `ad1ec60` [release-branch.go1.21] runtime/pprof: fix generics function names (2023-12-04)
- `cf65d74` [release-branch.go1.21] runtime: add the disablethp GODEBUG setting (2023-12-05)
- `4381820` [release-branch.go1.21] runtime: put ReadMemStats debug assertions behind a double-check mode (2023-11-27)
- `f6b203c` [release-branch.go1.21] maps: fix aliasing problems with Clone (2023-12-01)
- `8c6078a` [release-branch.go1.21] runtime: add race annotations in IncNonDefault (2023-12-14)
- `368e2a9` [release-branch.go1.21] crypto/internal/boring: upgrade module to fips-20220613 (2023-12-14)
- `d2cb140` [release-branch.go1.21] crypto/tls: align FIPS-only mode with BoringSSL policy (2023-12-14)
- `cc85462` [release-branch.go1.21] go1.21.6 (2024-01-09)
```

### go1.21.7

```text
- `caafb50` [release-branch.go1.21] runtime: don't print "unexpected SPWRITE" when printing traceback (2023-09-29)
- `491c1e7` [release-branch.go1.21] runtime: allow update of system stack bounds on callback from C thread (2023-09-04)
- `7e34c43` [release-branch.go1.21] runtime: clear g0 stack bounds in dropm (2023-10-25)
- `2540b14` [release-branch.go1.21] slices: explicitly discard results of some functions (2024-01-05)
- `2f91c16` [release-branch.go1.21] runtime: properly model rwmutex in lock ranking (2023-12-13)
- `00f974e` [release-branch.go1.21] go/types, types2: don't lose position info of interface embeddings (2024-01-10)
- `3960318` [release-branch.go1.21] crypto/x509: properly gate test on macos version (2023-12-12)
- `6552f3d` [release-branch.go1.21] cmd/go/internal/toolchain: apply the -modcacherw flag when downloading a module to determine what toolchain it needs (2024-01-11)
- `916e6cd` [release-branch.go1.21] runtime: fix Pinner.Pin documentation (2024-01-26)
- `db74bfb` [release-branch.go1.21] Revert "crypto/internal/boring: upgrade module to fips-20220613" +1 (2024-01-26)
- `01c93ad` [release-branch.go1.21] cmd/go/internal/generate: call modload.InitWorkFile (2024-01-29)
- `2fdad8a` [release-branch.go1.21] cmd/internal/obj/arm64: fix frame pointer restore in epilogue (2023-10-30)
- `f292080` [release-branch.go1.21] go1.21.7 (2024-02-06)
```

### go1.21.8

```text
- `f997dfd` [release-branch.go1.21] internal/testenv: allow "-noopt" anywhere in builder name in test (2024-01-25)
- `b214108` [release-branch.go1.21] internal/testenv: support the LUCI mobile builders in tests (2024-01-25)
- `f38fca3` [release-branch.go1.21] cmd/cgo/internal/testsanitizers: disable location checking for clang (2024-02-08)
- `6d31b27` [release-branch.go1.21] runtime: don't let the tests leave core files behind (2023-09-01)
- `99e44c7` [release-branch.go1.21] cmd/go/internal/modcmd: correctly filter out main modules in verify (2024-02-20)
- `263c059` [release-branch.go1.21] net/mail: properly handle special characters in phrase and obs-phrase (2024-01-10)
- `3a58877` [release-branch.go1.21] internal/syscall/windows: fix the signature of SetFileInformationByHandle (2023-12-12)
- `3643147` [release-branch.go1.21] html/template: escape additional tokens in MarshalJSON errors (2024-02-15)
- `be5b52b` [release-branch.go1.21] crypto/x509: make sure pub key is non-nil before interface conversion (2024-01-18)
- `20586c0` [release-branch.go1.21] net/http, net/http/cookiejar: avoid subdomain matches on IPv6 zones (2024-01-11)
- `bf80213` [release-branch.go1.21] net/textproto, mime/multipart: avoid unbounded read in MIME header (2024-01-16)
- `63992de` [release-branch.go1.21] go1.21.8 (2024-03-05)
```

### go1.21.9

```text
- `6d22988` [release-branch.go1.21] go/types, types2: don't do version checks for embedded types of imported interfaces (2024-03-12)
- `140b37d` [release-branch.go1.21] Revert "go/types, types2: don't do version checks for embedded types of imported interfaces" (2024-03-13)
- `0bd1a22` [release-branch.go1.21] cmd/internal/obj/ppc64: don't modify runtime.elf_* symbols (2024-03-12)
- `efb7cc4` [release-branch.go1.21] go/types, types2: don't do version checks for embedded types of imported interfaces (2024-03-12)
- `30d8550` [release-branch.go1.21] all: update golang.org/x/net (2024-03-27)
- `ae59133` [release-branch.go1.21] net/http: update bundled golang.org/x/net/http2 (2024-03-28)
- `d8392e6` [release-branch.go1.21] go1.21.9 (2024-04-03)
```

### go1.21.10

```text
- `7450117` [release-branch.go1.21] all: tidy dependency versioning after release (2024-04-03)
- `891ac91` [release-branch.go1.21] net/http: update bundled golang.org/x/net/http2 (2024-04-12)
- `58e77ad` [release-branch.go1.21] cmd/compile: don't combine loads in generated equality functions (2024-05-03)
- `78d89b2` [release-branch.go1.21] Revert "cmd/compile: don't combine loads in generated equality functions" (2024-05-06)
- `a79ea27` [release-branch.go1.21] cmd/go: disallow -lto_library in LDFLAGS (2024-04-25)
- `752b009` [release-branch.go1.21] go1.21.10 (2024-05-07)
```

### go1.21.11

```text
- `77cfa29` [release-branch.go1.21] cmd/go: download 1.X.0 instead of 1.X during toolchain upgrade. (2024-04-18)
- `8fbd4d7` [release-branch.go1.21] cmd/compile: don't combine loads in generated equality functions (2024-05-03)
- `0fa334d` [release-branch.go1.21] cmd/go/testdata/script: clear path in gotoolchain_issue66175 (2024-05-10)
- `a6023db` [release-branch.go1.21] cmd/go/internal/modfetch/codehost: add new git tags before statLocal instead of after (2023-12-04)
- `e9f3c9b` [release-branch.go1.21] cmd/go: set global .gitconfig location more robustly in tests (2023-12-07)
- `6f52195` [release-branch.go1.21] crypto/x509: remove TestPlatformVerifierLegacy tests (2023-12-11)
- `54c4745` [release-branch.go1.21] runtime: update large object stats before freeSpan in sweep (2024-05-05)
- `c8e4033` [release-branch.go1.21] archive/zip: treat truncated EOCDR comment as an error (2024-05-14)
- `9488a44` [release-branch.go1.21] os: RemoveAll: fix symlink race for unix (2024-05-26)
- `051bdf3` [release-branch.go1.21] net/netip: check if address is v6 mapped in Is methods (2024-05-28)
- `48103d9` [release-branch.go1.21] go1.21.11 (2024-06-04)
```

### go1.21.12

```text
- `7e563a0` [release-branch.go1.21] go/internal/gccgoimporter: recognize "any" as a builtin type (2023-10-23)
- `74ac37e` [release-branch.go1.21] net: add GODEBUG=netedns0=0 to disable sending EDNS0 header (2024-06-11)
- `e65014d` [release-branch.go1.21] cmd/go: fix go list -u -m all with too new retractions dependency (2024-05-20)
- `dfb0067` [release-branch.go1.21] cmd/link: put runtime.end in the last section of data segment (2024-06-11)
- `537ec6f` [release-branch.go1.21] cmd/cgo/internal/swig: force use of lld for LTO tests on the builders (2024-05-30)
- `29d378a` [release-branch.go1.21] cmd/cgo/internal/testsanitizers: make the libfuzzer tests all short (2024-05-29)
- `ac14d4d` [release-branch.go1.21] runtime: always update stack bounds on cgocallback (2024-05-09)
- `c9be6ae` [release-branch.go1.21] net/http: send body or close connection on expect-100-continue requests (2024-06-06)
- `12e9b96` [release-branch.go1.21] go1.21.12 (2024-07-02)
```

### go1.21.13

```text
- `e073feb` [release-branch.go1.21] cmd/go: fix build config before creating actions for 'go list -cover' (2024-02-01)
- `d149ce8` [release-branch.go1.21] internal/bytealg: extend memchr result correctly on wasm (2024-07-17)
- `dabed2e` [release-branch.go1.21] cmd/internal/cov: close counter data files eagerly (2024-07-16)
- `8bba868` [release-branch.go1.21] go1.21.13 (2024-08-06)
```
