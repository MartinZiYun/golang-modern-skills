# Go 1.22 Release Notes

## Introduction to Go 1.22

The latest Go release, version 1.22, arrives six months after [Go 1.21](/doc/go1.21).
Most of its changes are in the implementation of the toolchain, runtime, and libraries.
As always, the release maintains the Go 1 [promise of compatibility](/doc/go1compat).
We expect almost all Go programs to continue to compile and run as before.

## Changes to the language

Go 1.22 makes two changes to &ldquo;for&rdquo; loops.

Previously, the variables declared by a &ldquo;for&rdquo; loop were created once and updated by each iteration. In Go 1.22, each iteration of the loop creates new variables, to avoid accidental sharing bugs.
The [transition support tooling](/wiki/LoopvarExperiment#my-test-fails-with-the-change-how-can-i-debug-it)
described in the proposal continues to work in the same way it did in Go 1.21.

-

&ldquo;For&rdquo; loops may now range over integers.
For [example](/play/p/ky02zZxgk_r?v=gotip):

`package main

import "fmt"

func main() {
for i := range 10 {
fmt.Println(10 - i)
}
fmt.Println("go1.22 has lift-off!")
}
`
See the spec for [details](/ref/spec#For_range).

Go 1.22 includes a preview of a language change we are considering
for a future version of Go: [range-over-function iterators](/wiki/RangefuncExperiment).
Building with `GOEXPERIMENT=rangefunc` enables this feature.

## Tools

### Go command

Commands in [workspaces](/ref/mod#workspaces) can now
use a `vendor` directory containing the dependencies of the
workspace. The directory is created by
[`go` `work` `vendor`](/pkg/cmd/go#hdr-Make_vendored_copy_of_dependencies),
and used by build commands when the `-mod` flag is set to
`vendor`, which is the default when a workspace `vendor`
directory is present.

Note that the `vendor` directory&rsquo;s contents for a workspace are different
from those of a single module: if the directory at the root of a workspace also
contains one of the modules in the workspace, its `vendor` directory
can contain the dependencies of either the workspace or of the module,
but not both.

`go` `get` is no longer supported outside of a module in the
legacy `GOPATH` mode (that is, with `GO111MODULE=off`).
Other build commands, such as `go` `build` and
`go` `test`, will continue to work indefinitely
for legacy `GOPATH` programs.

`go` `mod` `init` no longer attempts to import
module requirements from configuration files for other vendoring tools
(such as `Gopkg.lock`).

`go` `test` `-cover` now prints coverage summaries
for covered packages that do not have their own test files. Prior to Go 1.22 a
`go` `test` `-cover` run for such a package would
report

`? mymod/mypack [no test files]`

and now with Go 1.22, functions in the package are treated as uncovered:

`mymod/mypack coverage: 0.0% of statements`

Note that if a package contains no executable code at all, we can&rsquo;t report
a meaningful coverage percentage; for such packages the `go` tool
will continue to report that there are no test files.

`go` build commands that invoke the linker now error out if an
external (C) linker will be used but cgo is not enabled. (The Go runtime
requires cgo support to ensure that it is compatible with any additional
libraries added by the C linker.)

### Trace

The `trace` tool&rsquo;s web UI has been gently refreshed as part of the
work to support the new tracer, resolving several issues and improving the
readability of various sub-pages.
The web UI now supports exploring traces in a thread-oriented view.
The trace viewer also now displays the full duration of all system calls.

These improvements only apply for viewing traces produced by programs built with
Go 1.22 or newer.
A future release will bring some of these improvements to traces produced by older
version of Go.

### Vet

#### References to loop variables

The behavior of the `vet` tool has changed to match
the new semantics (see above) of loop variables in Go 1.22.
When analyzing a file that requires Go 1.22 or newer
(due to its go.mod file or a per-file build constraint),
`vet` no longer reports references to
loop variables from within a function literal that
might outlive the iteration of the loop.
In Go 1.22, loop variables are created anew for each iteration,
so such references are no longer at risk of using a variable
after it has been updated by the loop.

#### New warnings for missing values after append

The `vet` tool now reports calls to
[`append`](/pkg/builtin/#append) that pass
no values to be appended to the slice, such as `slice = append(slice)`.
Such a statement has no effect, and experience has shown that is nearly always a mistake.

#### New warnings for deferring `time.Since`

The vet tool now reports a non-deferred call to
[`time.Since(t)`](/pkg/time/#Since) within a `defer` statement.
This is equivalent to calling `time.Now().Sub(t)` before the `defer` statement,
not when the deferred function is called. In nearly all cases, the correct code
requires deferring the `time.Since` call. For example:

`t := time.Now()
defer log.Println(time.Since(t)) // non-deferred call to time.Since
tmp := time.Since(t); defer log.Println(tmp) // equivalent to the previous defer

defer func() {
log.Println(time.Since(t)) // a correctly deferred call to time.Since
}()
`

#### New warnings for mismatched key-value pairs in `log/slog` calls

The vet tool now reports invalid arguments in calls to functions and methods
in the structured logging package, [`log/slog`](/pkg/log/slog),
that accept alternating key/value pairs.
It reports calls where an argument in a key position is neither a
`string` nor a `slog.Attr`, and where a final key is missing its value.

## Runtime

The runtime now keeps type-based garbage collection metadata nearer to each
heap object, improving the CPU performance (latency or throughput) of Go programs
by 1–3%.
This change also reduces the memory overhead of the majority Go programs by
approximately 1% by deduplicating redundant metadata.
Some programs may see a smaller improvement because this change adjusts the size
class boundaries of the memory allocator, so some objects may be moved up a size
class.

A consequence of this change is that some objects&rsquo; addresses that were previously
always aligned to a 16 byte (or higher) boundary will now only be aligned to an 8
byte boundary.
Some programs that use assembly instructions that require memory addresses to be
more than 8-byte aligned and rely on the memory allocator&rsquo;s previous alignment behavior
may break, but we expect such programs to be rare.
Such programs may be built with `GOEXPERIMENT=noallocheaders` to revert
to the old metadata layout and restore the previous alignment behavior, but package
owners should update their assembly code to avoid the alignment assumption, as this
workaround will be removed in a future release.

On the `windows/amd64 port`, programs linking or loading Go libraries built with
`-buildmode=c-archive` or `-buildmode=c-shared` can now use
the `SetUnhandledExceptionFilter` Win32 function to catch exceptions not handled
by the Go runtime. Note that this was already supported on the `windows/386` port.

## Compiler

[Profile-guided Optimization (PGO)](/doc/pgo) builds
can now devirtualize a higher proportion of calls than previously possible.
Most programs from a representative set of Go programs now see between 2 and
14% improvement at runtime from enabling PGO.

The compiler now interleaves devirtualization and inlining, so interface
method calls are better optimized.

Go 1.22 also includes a preview of an enhanced implementation of the compiler&rsquo;s inlining phase that uses heuristics to boost inlinability at call sites deemed &ldquo;important&rdquo; (for example, in loops) and discourage inlining at call sites deemed &ldquo;unimportant&rdquo; (for example, on panic paths).
Building with `GOEXPERIMENT=newinliner` enables the new call-site
heuristics; see [issue #61502](/issue/61502) for
more info and to provide feedback.

## Linker

The linker&rsquo;s `-s` and `-w` flags are now behave more
consistently across all platforms.
The `-w` flag suppresses DWARF debug information generation.
The `-s` flag suppresses symbol table generation.
The `-s` flag also implies the `-w` flag, which can be
negated with `-w=0`.
That is, `-s` `-w=0` will generate a binary with DWARF
debug information generation but without the symbol table.

On ELF platforms, the `-B` linker flag now accepts a special form:
with `-B` `gobuildid`, the linker will generate a GNU
build ID (the ELF `NT_GNU_BUILD_ID` note) derived from the Go
build ID.

On Windows, when building with `-linkmode=internal`, the linker now
preserves SEH information from C object files by copying the `.pdata`
and `.xdata` sections into the final binary.
This helps with debugging and profiling binaries using native tools, such as WinDbg.
Note that until now, C functions&rsquo; SEH exception handlers were not being honored,
so this change may cause some programs to behave differently.
`-linkmode=external` is not affected by this change, as external linkers
already preserve SEH information.

## Bootstrap

As mentioned in the [Go 1.20 release notes](/doc/go1.20#bootstrap), Go 1.22 now requires
the final point release of Go 1.20 or later for bootstrap.
We expect that Go 1.24 will require the final point release of Go 1.22 or later for bootstrap.

## Standard library

### New math/rand/v2 package

Go 1.22 includes the first “v2” package in the standard library,
[`math/rand/v2`](/pkg/math/rand/v2/).
The changes compared to [`math/rand`](/pkg/math/rand/) are
detailed in [proposal #61716](/issue/61716). The most important changes are:

- The `Read` method, deprecated in `math/rand`,
was not carried forward for `math/rand/v2`.
(It remains available in `math/rand`.)
The vast majority of calls to `Read` should use
[`crypto/rand`’s `Read`](/pkg/crypto/rand/#Read) instead.
Otherwise a custom `Read` can be constructed using the `Uint64` method.

- The global generator accessed by top-level functions is unconditionally randomly seeded.
Because the API guarantees no fixed sequence of results,
optimizations like per-thread random generator states are now possible.

- The [`Source`](/pkg/math/rand/v2/#Source)
interface now has a single `Uint64` method;
there is no `Source64` interface.

- Many methods now use faster algorithms that were not possible to adopt in `math/rand`
because they changed the output streams.

- The
`Intn`,
`Int31`,
`Int31n`,
`Int63`,
and
`Int64n`
top-level functions and methods from `math/rand`
are spelled more idiomatically in `math/rand/v2`:
`IntN`,
`Int32`,
`Int32N`,
`Int64`,
and
`Int64N`.
There are also new top-level functions and methods
`Uint32`,
`Uint32N`,
`Uint64`,
`Uint64N`,
and
`UintN`.

- The
new generic function [`N`](/pkg/math/rand/v2/#N)
is like
[`Int64N`](/pkg/math/rand/v2/#Int64N) or
[`Uint64N`](/pkg/math/rand/v2/#Uint64N)
but works for any integer type.
For example a random duration from 0 up to 5 minutes is
`rand.N(5*time.Minute)`.

- The Mitchell & Reeds LFSR generator provided by
[`math/rand`’s `Source`](/pkg/math/rand/#Source)
has been replaced by two more modern pseudo-random generator sources:
[`ChaCha8`](/pkg/math/rand/v2/#ChaCha8) and
[`PCG`](/pkg/math/rand/v2/#PCG).
ChaCha8 is a new, cryptographically strong random number generator
roughly similar to PCG in efficiency.
ChaCha8 is the algorithm used for the top-level functions in `math/rand/v2`.
As of Go 1.22, `math/rand`&rsquo;s top-level functions (when not explicitly seeded)
and the Go runtime also use ChaCha8 for randomness.

We plan to include an API migration tool in a future release, likely Go 1.23.

### New go/version package

The new [`go/version`](/pkg/go/version/) package implements functions
for validating and comparing Go version strings.

### Enhanced routing patterns

HTTP routing in the standard library is now more expressive.
The patterns used by [`net/http.ServeMux`](/pkg/net/http#ServeMux) have been enhanced to accept methods and wildcards.

Registering a handler with a method, like `"POST /items/create"`, restricts
invocations of the handler to requests with the given method. A pattern with a method takes precedence over a matching pattern without one.
As a special case, registering a handler with ` "GET"` also registers it with `"HEAD"`.

Wildcards in patterns, like `/items/{id}`, match segments of the URL path.
The actual segment value may be accessed by calling the [`Request.PathValue`](/pkg/net/http#Request.PathValue) method.
A wildcard ending in &ldquo;&hellip;&rdquo;, like `/files/{path...}`, must occur at the end of a pattern and matches all the remaining segments.

A pattern that ends in &ldquo;/&rdquo; matches all paths that have it as a prefix, as always.
To match the exact pattern including the trailing slash, end it with `{$}`,
as in `/exact/match/{$}`.

If two patterns overlap in the requests that they match, then the more specific pattern takes precedence.
If neither is more specific, the patterns conflict.
This rule generalizes the original precedence rules and maintains the property that the order in which
patterns are registered does not matter.

This change breaks backwards compatibility in small ways, some obvious—patterns with &ldquo;{&rdquo; and &ldquo;}&rdquo; behave differently—
and some less so—treatment of escaped paths has been improved.
The change is controlled by a [`GODEBUG`](/doc/godebug) field named `httpmuxgo121`.
Set `httpmuxgo121=1` to restore the old behavior.

### Minor changes to the library

As always, there are various minor changes and updates to the library,
made with the Go 1 [promise of compatibility](/doc/go1compat)
in mind.
There are also various performance improvements, not enumerated here.

[archive/tar](/pkg/archive/tar/)

The new method [`Writer.AddFS`](/pkg/archive/tar#Writer.AddFS) adds all of the files from an [`fs.FS`](/pkg/io/fs#FS) to the archive.

[archive/zip](/pkg/archive/zip/)

The new method [`Writer.AddFS`](/pkg/archive/zip#Writer.AddFS) adds all of the files from an [`fs.FS`](/pkg/io/fs#FS) to the archive.

[bufio](/pkg/bufio/)

When a [`SplitFunc`](/pkg/bufio#SplitFunc) returns [`ErrFinalToken`](/pkg/bufio#ErrFinalToken) with a `nil` token, [`Scanner`](/pkg/bufio#Scanner) will now stop immediately.
Previously, it would report a final empty token before stopping, which was usually not desired.
Callers that do want to report a final empty token can do so by returning `[]byte{}` rather than `nil`.

[cmp](/pkg/cmp/)

The new function `Or` returns the first in a sequence of values that is not the zero value.

[crypto/tls](/pkg/crypto/tls/)

[`ConnectionState.ExportKeyingMaterial`](/pkg/crypto/tls#ConnectionState.ExportKeyingMaterial) will now
return an error unless TLS 1.3 is in use, or the `extended_master_secret` extension is supported by both the server and
client. `crypto/tls` has supported this extension since Go 1.20. This can be disabled with the
`tlsunsafeekm=1` GODEBUG setting.

By default, the minimum version offered by `crypto/tls` servers is now TLS 1.2 if not specified with
[`config.MinimumVersion`](/pkg/crypto/tls#Config.MinimumVersion), matching the behavior of `crypto/tls`
clients. This change can be reverted with the `tls10server=1` GODEBUG setting.

By default, cipher suites without ECDHE support are no longer offered by either clients or servers during pre-TLS 1.3
handshakes. This change can be reverted with the `tlsrsakex=1` GODEBUG setting.

[crypto/x509](/pkg/crypto/x509/)

The new [`CertPool.AddCertWithConstraint`](/pkg/crypto/x509#CertPool.AddCertWithConstraint)
method can be used to add customized constraints to root certificates to be applied during chain building.

On Android, root certificates will now be loaded from `/data/misc/keychain/certs-added` as well as `/system/etc/security/cacerts`.

A new type, [`OID`](/pkg/crypto/x509#OID), supports ASN.1 Object Identifiers with individual
components larger than 31 bits. A new field which uses this type, [`Policies`](/pkg/crypto/x509#Certificate.Policies),
is added to the `Certificate` struct, and is now populated during parsing. Any OIDs which cannot be represented
using a [`asn1.ObjectIdentifier`](/pkg/encoding/asn1#ObjectIdentifier) will appear in `Policies`,
but not in the old `PolicyIdentifiers` field.
When calling [`CreateCertificate`](/pkg/crypto/x509#CreateCertificate), the `Policies` field is ignored, and
policies are taken from the `PolicyIdentifiers` field. Using the `x509usepolicies=1` GODEBUG setting inverts this,
populating certificate policies from the `Policies` field, and ignoring the `PolicyIdentifiers` field. We may change the
default value of `x509usepolicies` in Go 1.23, making `Policies` the default field for marshaling.

[database/sql](/pkg/database/sql/)

The new [`Null[T]`](/pkg/database/sql/#Null) type
provide a way to scan nullable columns for any column types.

[debug/elf](/pkg/debug/elf/)

Constant `R_MIPS_PC32` is defined for use with MIPS64 systems.

Additional `R_LARCH_*` constants are defined for use with LoongArch systems.

[encoding](/pkg/encoding/)

The new methods `AppendEncode` and `AppendDecode` added to
each of the `Encoding` types in the packages
[`encoding/base32`](/pkg/encoding/base32),
[`encoding/base64`](/pkg/encoding/base64), and
[`encoding/hex`](/pkg/encoding/hex)
simplify encoding and decoding from and to byte slices by taking care of byte slice buffer management.

The methods
[`base32.Encoding.WithPadding`](/pkg/encoding/base32#Encoding.WithPadding) and
[`base64.Encoding.WithPadding`](/pkg/encoding/base64#Encoding.WithPadding)
now panic if the `padding` argument is a negative value other than
`NoPadding`.

[encoding/json](/pkg/encoding/json/)

Marshaling and encoding functionality now escapes
`'\b'` and `'\f'` characters as
`\b` and `\f` instead of
`\u0008` and `\u000c`.

[go/ast](/pkg/go/ast/)

The following declarations related to
[syntactic identifier resolution](https://pkg.go.dev/go/ast#Object)
are now [deprecated](/issue/52463):
`Ident.Obj`,
`Object`,
`Scope`,
`File.Scope`,
`File.Unresolved`,
`Importer`,
`Package`,
`NewPackage`.
In general, identifiers cannot be accurately resolved without type information.
Consider, for example, the identifier `K`
in `T{K: ""}`: it could be the name of a local variable
if T is a map type, or the name of a field if T is a struct type.
New programs should use the [go/types](/pkg/go/types)
package to resolve identifiers; see
[`Object`](https://pkg.go.dev/go/types#Object),
[`Info.Uses`](https://pkg.go.dev/go/types#Info.Uses), and
[`Info.Defs`](https://pkg.go.dev/go/types#Info.Defs) for details.

The new [`ast.Unparen`](https://pkg.go.dev/go/ast#Unparen)
function removes any enclosing
[parentheses](https://pkg.go.dev/go/ast#ParenExpr) from
an [expression](https://pkg.go.dev/go/ast#Expr).

[go/types](/pkg/go/types/)

The new [`Alias`](/pkg/go/types#Alias) type represents type aliases.
Previously, type aliases were not represented explicitly, so a reference to a type alias was equivalent
to spelling out the aliased type, and the name of the alias was lost.
The new representation retains the intermediate `Alias`.
This enables improved error reporting (the name of a type alias can be reported), and allows for better handling
of cyclic type declarations involving type aliases.
In a future release, `Alias` types will also carry [type parameter information](/issue/46477).
The new function [`Unalias`](/pkg/go/types#Unalias) returns the actual type denoted by an
`Alias` type (or any other [`Type`](/pkg/go/types#Type) for that matter).

Because `Alias` types may break existing type switches that do not know to check for them,
this functionality is controlled by a [`GODEBUG`](/doc/godebug) field named `gotypesalias`.
With `gotypesalias=0`, everything behaves as before, and `Alias` types are never created.
With `gotypesalias=1`, `Alias` types are created and clients must expect them.
The default is `gotypesalias=0`.
In a future release, the default will be changed to `gotypesalias=1`.
Clients of [`go/types`](/pkg/go/types) are urged to adjust their code as soon as possible
to work with `gotypesalias=1` to eliminate problems early.*

The [`Info`](/pkg/go/types#Info) struct now exports the
[`FileVersions`](/pkg/go/types#Info.FileVersions) map
which provides per-file Go version information.

The new helper method [`PkgNameOf`](/pkg/go/types#Info.PkgNameOf) returns the local package name
for the given import declaration.

The implementation of [`SizesFor`](/pkg/go/types#SizesFor) has been adjusted to compute
the same type sizes as the compiler when the compiler argument for `SizesFor` is `"gc"`.
The default [`Sizes`](/pkg/go/types#Sizes) implementation used by the type checker is now
`types.SizesFor("gc", "amd64")`.

The start position ([`Pos`](/pkg/go/types#Scope.Pos))
of the lexical environment block ([`Scope`](/pkg/go/types#Scope))
that represents a function body has changed:
it used to start at the opening curly brace of the function body,
but now starts at the function&rsquo;s `func` token.

[html/template](/pkg/html/template/)

JavaScript template literals may now contain Go template actions, and parsing a template containing one will
no longer return `ErrJSTemplate`. Similarly the GODEBUG setting `jstmpllitinterp` no
longer has any effect.

[io](/pkg/io/)

The new [`SectionReader.Outer`](/pkg/io#SectionReader.Outer) method returns the [`ReaderAt`](/pkg/io#ReaderAt), offset, and size passed to [`NewSectionReader`](/pkg/io#NewSectionReader).

[log/slog](/pkg/log/slog/)

The new [`SetLogLoggerLevel`](/pkg/log/slog#SetLogLoggerLevel) function
controls the level for the bridge between the `slog` and `log` packages. It sets the minimum level
for calls to the top-level `slog` logging functions, and it sets the level for calls to `log.Logger`
that go through `slog`.

[math/big](/pkg/math/big/)

The new method [`Rat.FloatPrec`](/pkg/math/big#Rat.FloatPrec) computes the number of fractional decimal digits
required to represent a rational number accurately as a floating-point number, and whether accurate decimal representation
is possible in the first place.

[net](/pkg/net/)

When [`io.Copy`](/pkg/io#Copy) copies
from a `TCPConn` to a `UnixConn`,
it will now use Linux&rsquo;s `splice(2)` system call if possible,
using the new method [`TCPConn.WriteTo`](/pkg/net#TCPConn.WriteTo).

The Go DNS Resolver, used when building with &ldquo;-tags=netgo&rdquo;,
now searches for a matching name in the Windows hosts file,
located at `%SystemRoot%\System32\drivers\etc\hosts`,
before making a DNS query.

[net/http](/pkg/net/http/)

The new functions
[`ServeFileFS`](/pkg/net/http#ServeFileFS),
[`FileServerFS`](/pkg/net/http#FileServerFS), and
[`NewFileTransportFS`](/pkg/net/http#NewFileTransportFS)
are versions of the existing
`ServeFile`, `FileServer`, and `NewFileTransport`,
operating on an `fs.FS`.

The HTTP server and client now reject requests and responses containing
an invalid empty `Content-Length` header.
The previous behavior may be restored by setting
[`GODEBUG`](/doc/godebug) field `httplaxcontentlength=1`.

The new method
[`Request.PathValue`](/pkg/net/http#Request.PathValue)
returns path wildcard values from a request
and the new method
[`Request.SetPathValue`](/pkg/net/http#Request.SetPathValue)
sets path wildcard values on a request.

[net/http/cgi](/pkg/net/http/cgi/)

When executing a CGI process, the `PATH_INFO` variable is now
always set to the empty string or a value starting with a `/` character,
as required by RFC 3875. It was previously possible for some combinations of
[`Handler.Root`](/pkg/net/http/cgi#Handler.Root)
and request URL to violate this requirement.

[net/netip](/pkg/net/netip/)

The new [`AddrPort.Compare`](/pkg/net/netip#AddrPort.Compare)
method compares two `AddrPort`s.

[os](/pkg/os/)

On Windows, the [`Stat`](/pkg/os#Stat) function now follows all reparse points
that link to another named entity in the system.
It was previously only following `IO_REPARSE_TAG_SYMLINK` and
`IO_REPARSE_TAG_MOUNT_POINT` reparse points.

On Windows, passing [`O_SYNC`](/pkg/os#O_SYNC) to [`OpenFile`](/pkg/os#OpenFile) now causes write operations to go directly to disk, equivalent to `O_SYNC` on Unix platforms.

On Windows, the [`ReadDir`](/pkg/os#ReadDir),
[`File.ReadDir`](/pkg/os#File.ReadDir),
[`File.Readdir`](/pkg/os#File.Readdir),
and [`File.Readdirnames`](/pkg/os#File.Readdirnames) functions
now read directory entries in batches to reduce the number of system calls,
improving performance up to 30%.

When [`io.Copy`](/pkg/io#Copy) copies
from a `File` to a `net.UnixConn`,
it will now use Linux&rsquo;s `sendfile(2)` system call if possible,
using the new method [`File.WriteTo`](/pkg/os#File.WriteTo).

[os/exec](/pkg/os/exec/)

On Windows, [`LookPath`](/pkg/os/exec#LookPath) now
ignores empty entries in `%PATH%`, and returns
`ErrNotFound` (instead of `ErrNotExist`) if
no executable file extension is found to resolve an otherwise-unambiguous
name.

On Windows, [`Command`](/pkg/os/exec#Command) and
[`Cmd.Start`](/pkg/os/exec#Cmd.Start) no
longer call `LookPath` if the path to the executable is already
absolute and has an executable file extension. In addition,
`Cmd.Start` no longer writes the resolved extension back to
the [`Path`](/pkg/os/exec#Cmd.Path) field,
so it is now safe to call the `String` method concurrently
with a call to `Start`.

[reflect](/pkg/reflect/)

The [`Value.IsZero`](/pkg/reflect/#Value.IsZero)
method will now return true for a floating-point or complex
negative zero, and will return true for a struct value if a
blank field (a field named `_`) somehow has a
non-zero value.
These changes make `IsZero` consistent with comparing
a value to zero using the language `==` operator.

The [`PtrTo`](/pkg/reflect/#PtrTo) function is deprecated,
in favor of [`PointerTo`](/pkg/reflect/#PointerTo).

The new function [`TypeFor`](/pkg/reflect/#TypeFor)
returns the [`Type`](/pkg/reflect/#Type) that represents
the type argument T.
Previously, to get the `reflect.Type` value for a type, one had to use
`reflect.TypeOf((*T)(nil)).Elem()`.
This may now be written as `reflect.TypeFor[T]()`.

[runtime/metrics](/pkg/runtime/metrics/)

Four new histogram metrics
`/sched/pauses/stopping/gc:seconds`,
`/sched/pauses/stopping/other:seconds`,
`/sched/pauses/total/gc:seconds`, and
`/sched/pauses/total/other:seconds` provide additional details
about stop-the-world pauses.
The &ldquo;stopping&rdquo; metrics report the time taken from deciding to stop the
world until all goroutines are stopped.
The &ldquo;total&rdquo; metrics report the time taken from deciding to stop the world
until it is started again.

The `/gc/pauses:seconds` metric is deprecated, as it is
equivalent to the new `/sched/pauses/total/gc:seconds` metric.

`/sync/mutex/wait/total:seconds` now includes contention on
runtime-internal locks in addition to
[`sync.Mutex`](/pkg/sync#Mutex) and
[`sync.RWMutex`](/pkg/sync#RWMutex).

[runtime/pprof](/pkg/runtime/pprof/)

Mutex profiles now scale contention by the number of goroutines blocked on the mutex.
This provides a more accurate representation of the degree to which a mutex is a bottleneck in
a Go program.
For instance, if 100 goroutines are blocked on a mutex for 10 milliseconds, a mutex profile will
now record 1 second of delay instead of 10 milliseconds of delay.

Mutex profiles also now include contention on runtime-internal locks in addition to
[`sync.Mutex`](/pkg/sync#Mutex) and
[`sync.RWMutex`](/pkg/sync#RWMutex).
Contention on runtime-internal locks is always reported at `runtime._LostContendedRuntimeLock`.
A future release will add complete stack traces in these cases.

CPU profiles on Darwin platforms now contain the process&rsquo;s memory map, enabling the disassembly
view in the pprof tool.

[runtime/trace](/pkg/runtime/trace/)

The execution tracer has been completely overhauled in this release, resolving several long-standing
issues and paving the way for new use-cases for execution traces.

Execution traces now use the operating system&rsquo;s clock on most platforms (Windows excluded) so
it is possible to correlate them with traces produced by lower-level components.
Execution traces no longer depend on the reliability of the platform&rsquo;s clock to produce a correct trace.
Execution traces are now partitioned regularly on-the-fly and as a result may be processed in a
streamable way.
Execution traces now contain complete durations for all system calls.
Execution traces now contain information about the operating system threads that goroutines executed on.
The latency impact of starting and stopping execution traces has been dramatically reduced.
Execution traces may now begin or end during the garbage collection mark phase.

To allow Go developers to take advantage of these improvements, an experimental
trace reading package is available at [golang.org/x/exp/trace](/pkg/golang.org/x/exp/trace).
Note that this package only works on traces produced by programs built with Go 1.22 at the moment.
Please try out the package and provide feedback on
[the corresponding proposal issue](/issue/62627).

If you experience any issues with the new execution tracer implementation, you may switch back to the
old implementation by building your Go program with `GOEXPERIMENT=noexectracer2`.
If you do, please file an issue, otherwise this option will be removed in a future release.

[slices](/pkg/slices/)

The new function `Concat` concatenates multiple slices.

Functions that shrink the size of a slice (`Delete`, `DeleteFunc`, `Compact`, `CompactFunc`, and `Replace`) now zero the elements between the new length and the old length.

`Insert` now always panics if the argument `i` is out of range. Previously it did not panic in this situation if there were no elements to be inserted.

[syscall](/pkg/syscall/)

The `syscall` package has been [frozen](/s/go1.4-syscall) since Go 1.4 and was marked as deprecated in Go 1.11, causing many editors to warn about any use of the package.
However, some non-deprecated functionality requires use of the `syscall` package, such as the [`os/exec.Cmd.SysProcAttr`](/pkg/os/exec#Cmd) field.
To avoid unnecessary complaints on such code, the `syscall` package is no longer marked as deprecated.
The package remains frozen to most new functionality, and new code remains encouraged to use [`golang.org/x/sys/unix`](/pkg/golang.org/x/sys/unix) or [`golang.org/x/sys/windows`](/pkg/golang.org/x/sys/windows) where possible.

On Linux, the new [`SysProcAttr.PidFD`](/pkg/syscall#SysProcAttr) field allows obtaining a PID FD when starting a child process via [`StartProcess`](/pkg/syscall#StartProcess) or [`os/exec`](/pkg/os/exec).

On Windows, passing [`O_SYNC`](/pkg/syscall#O_SYNC) to [`Open`](/pkg/syscall#Open) now causes write operations to go directly to disk, equivalent to `O_SYNC` on Unix platforms.

[testing/slogtest](/pkg/testing/slogtest/)

The new [`Run`](/pkg/testing/slogtest#Run) function uses sub-tests to run test cases,
providing finer-grained control.

## Ports

### Darwin

On macOS on 64-bit x86 architecture (the `darwin/amd64` port),
the Go toolchain now generates position-independent executables (PIE) by default.
Non-PIE binaries can be generated by specifying the `-buildmode=exe`
build flag.
On 64-bit ARM-based macOS (the `darwin/arm64` port),
the Go toolchain already generates PIE by default.

Go 1.22 is the last release that will run on macOS 10.15 Catalina. Go 1.23 will require macOS 11 Big Sur or later.

### ARM

The `GOARM` environment variable now allows you to select whether to use software or hardware floating point.
Previously, valid `GOARM` values were `5`, `6`, or `7`. Now those same values can
be optionally followed by `,softfloat` or `,hardfloat` to select the floating-point implementation.

This new option defaults to `softfloat` for version `5` and `hardfloat` for versions
`6` and `7`.

### Loong64

The `loong64` port now supports passing function arguments and results using registers.

The `linux/loong64` port now supports the address sanitizer, memory sanitizer, new-style linker relocations, and the `plugin` build mode.

### OpenBSD

Go 1.22 adds an experimental port to OpenBSD on big-endian 64-bit PowerPC
(`openbsd/ppc64`).

## Patch Release Changelogs

### go1.22.1

```text
- `dd31ad7` [release-branch.go1.22] spec: fix typo in year (it's 2024 now) (2024-02-07)
- `53d1b73` [release-branch.go1.22] internal/testenv: allow "-noopt" anywhere in builder name in test (2024-01-25)
- `20107e0` [release-branch.go1.22] internal/testenv: support the LUCI mobile builders in tests (2024-01-25)
- `d6a2719` [release-branch.go1.22] cmd/cgo/internal/testsanitizers: disable location checking for clang (2024-02-08)
- `6fbd01a` [release-branch.go1.22] runtime: don't call traceReadCPU on the system stack (2024-02-09)
- `fb86598` [release-branch.go1.22] cmd/compile: accept -lang=go1 as -lang=go1.0 (2024-02-08)
- `6cbe522` [release-branch.go1.22] cmd/compile: fail noder.LookupFunc gracefully if function generic (2024-02-08)
- `686662f` [release-branch.go1.22] cmd/compile: make jump table symbol static (2024-02-20)
- `8fe2ad6` [release-branch.go1.22] runtime/internal/atomic: correct GOARM=7 guard at a DMB instruction (2024-02-16)
- `3b71998` [release-branch.go1.22] go/types, types2: ensure that Alias.actual is set in NewAlias (2024-02-02)
- `c33adf4` [release-branch.go1.22] cmd/trace/v2,internal/trace: use correct frame for identifying goroutines (2024-02-07)
- `d8c4239` [release-branch.go1.22] cmd/go/internal/modcmd: correctly filter out main modules in verify (2024-02-20)
- `5330cd2` [release-branch.go1.22] net/mail: properly handle special characters in phrase and obs-phrase (2024-01-10)
- `f73eba7` [release-branch.go1.22] net: work around runtime scheduler starvation on js and wasip1 (2024-01-19)
- `056b0ed` [release-branch.go1.22] html/template: escape additional tokens in MarshalJSON errors (2024-02-15)
- `16830ab` [release-branch.go1.22] net/http: add missing call to decConnsPerHost (2024-02-14)
- `337b8e9` [release-branch.go1.22] crypto/x509: make sure pub key is non-nil before interface conversion (2024-01-18)
- `3a85520` [release-branch.go1.22] net/http, net/http/cookiejar: avoid subdomain matches on IPv6 zones (2024-01-11)
- `041a477` [release-branch.go1.22] net/textproto, mime/multipart: avoid unbounded read in MIME header (2024-01-16)
- `db6097f` [release-branch.go1.22] go1.22.1 (2024-03-05)
```

### go1.22.2

```text
- `35b1a14` [release-branch.go1.22] cmd/go: respect -coverpkg for unselected no-test packages (2024-03-04)
- `9508eae` [release-branch.go1.22] cmd/internal/obj/ppc64: don't modify runtime.elf_* symbols (2024-03-12)
- `e71b0b1` [release-branch.go1.22] cmd/compile: fix copying SSA-able variables optimization (2024-03-07)
- `3826650` [release-branch.go1.22] cmd/compile: fix early deadcode with label statement (2024-02-21)
- `e23707b` [release-branch.go1.22] go/types, types2: handle Alias types in substitution (2024-02-21)
- `7fb7acb` [release-branch.go1.22] cmd/link/internal/riscv64: generate local text symbols for R_RISCV_CALL (2024-02-27)
- `abfd578` [release-branch.go1.22] runtime: fix EvFrequency event value on Windows in the new tracer (2024-02-28)
- `0c53f93` [release-branch.go1.22] cmd/go/internal/modfetch: pass "-c" arguments to git before subcommands (2024-03-06)
- `0a5b33a` [release-branch.go1.22] encoding/gob: make x509.Certificate marshalable again (2024-03-12)
- `4658748` [release-branch.go1.22] cmd/compile: fix sign/zero-extension removal (2024-03-03)
- `2c6d106` [release-branch.go1.22] cmd/compile: don't assume args are always zero-extended (2024-03-12)
- `4edf4bb` [release-branch.go1.22] all: update golang.org/x/net (2024-03-27)
- `e55d7cf` [release-branch.go1.22] net/http: update bundled golang.org/x/net/http2 (2024-03-28)
- `dddf0ae` [release-branch.go1.22] go1.22.2 (2024-04-03)
```

### go1.22.3

```text
- `a65a2bb` [release-branch.go1.22] all: tidy dependency versioning after release (2024-04-03)
- `d6c972a` [release-branch.go1.22] net/http: update bundled golang.org/x/net/http2 (2024-04-12)
- `12c1177` [release-branch.go1.22] cmd/internal/obj/ppc64: fix incorrect int to int64 conversion when checking MOVD opcodes (2024-04-22)
- `a7ff78d` [release-branch.go1.22] cmd/compile: bail PGO method lookup on interface types (2024-04-24)
- `3f4af1f` [release-branch.go1.22] runtime: use bootstrapRand to initialize hashkey (2024-04-18)
- `93d8777` [release-branch.go1.22] net: check SkipAdditional error result (2024-04-11)
- `9d2e285` [release-branch.go1.22] cmd/compile: don't combine loads in generated equality functions (2024-05-03)
- `947e43e` [release-branch.go1.22] Revert "cmd/compile: don't combine loads in generated equality functions" (2024-05-06)
- `fa0292d` [release-branch.go1.22] cmd/go: disallow -lto_library in LDFLAGS (2024-04-25)
- `adbfb67` [release-branch.go1.22] go1.22.3 (2024-05-07)
```

### go1.22.4

```text
- `00e6815` [release-branch.go1.22] cmd/go: download 1.X.0 instead of 1.X during toolchain upgrade. (2024-04-18)
- `dc8976d` [release-branch.go1.22] cmd/compile: don't combine loads in generated equality functions (2024-05-03)
- `d4a81ec` [release-branch.go1.22] cmd/go/testdata/script: clear path in gotoolchain_issue66175 (2024-05-10)
- `362dced` [release-branch.go1.22] cmd/go/testdata/script: add darwin skips for selected buildrepro tests (2024-02-20)
- `3a84293` [release-branch.go1.22] cmd/compile: avoid past-the-end pointer when zeroing (2024-05-08)
- `185457d` [release-branch.go1.22] crypto/x509: remove TestPlatformVerifierLegacy tests (2023-12-11)
- `6b89e7d` [release-branch.go1.22] cmd/compile: initialize posBaseMap correctly (2024-05-17)
- `3c96ae0` [release-branch.go1.22] runtime: update large object stats before freeSpan in sweep (2024-05-05)
- `cb55d1a` [release-branch.go1.22] cmd/link: add runtime.text.N symbols to macho symbol table in dynlink mode (2024-05-17)
- `cf501ac` [release-branch.go1.22] archive/zip: treat truncated EOCDR comment as an error (2024-05-14)
- `95389d3` [release-branch.go1.22] runtime: x_cgo_getstackbound: initialize pthread attr (2024-05-23)
- `7456575` [release-branch.go1.22] os: RemoveAll: fix symlink race for unix (2024-05-26)
- `12d5810` [release-branch.go1.22] net/netip: check if address is v6 mapped in Is methods (2024-05-28)
- `ace5bb4` [release-branch.go1.22] go1.22.4 (2024-06-04)
```

### go1.22.5

```text
- `b515c52` [release-branch.go1.22] go/internal/gccgoimporter: recognize "any" as a builtin type (2023-10-23)
- `fe9b3c3` [release-branch.go1.22] net: add GODEBUG=netedns0=0 to disable sending EDNS0 header (2024-06-11)
- `179ccb7` [release-branch.go1.22] cmd/go: fix go list -u -m all with too new retractions dependency (2024-05-20)
- `4c97e88` [release-branch.go1.22] cmd/link: put runtime.end in the last section of data segment (2024-06-11)
- `ab60a7b` [release-branch.go1.22] cmd/cgo/internal/testsanitizers: make the libfuzzer tests all short (2024-05-29)
- `14f0251` [release-branch.go1.22] cmd/cgo/internal/swig: force use of lld for LTO tests on the builders (2024-05-30)
- `81fc616` [release-branch.go1.22] crypto/tls: don't call tlsrsakex.IncNonDefault with FIPS (2024-04-29)
- `11b861e` [release-branch.go1.22] go/types, types2: report error for floating-point iteration variable (2024-04-22)
- `5159a71` [release-branch.go1.22] cmd/compile: put constants before variables in initialization order (2024-03-28)
- `3560cf0` [release-branch.go1.22] runtime: always update stack bounds on cgocallback (2024-05-09)
- `dfe4dbf` [release-branch.go1.22] os/exec: on Windows look for extensions in Run if not already done (2024-06-05)
- `ceaf26e` [release-branch.go1.22] cmd/compile: mark pointer to noalg type as noalg (2024-02-27)
- `3222951` [release-branch.go1.22] net/http: send body or close connection on expect-100-continue requests (2024-06-06)
- `c2d4f85` [release-branch.go1.22] cmd/link: handle dynamic import variables on Darwin in plugin mode (2024-06-17)
- `8e1fdea` [release-branch.go1.22] go1.22.5 (2024-07-02)
```

### go1.22.6

```text
- `ea96074` [release-branch.go1.22] os/exec: only use cachedLookExtensions if Cmd.Path is unmodified (2024-07-04)
- `49906f9` [release-branch.go1.22] cmd/go: fix build config before creating actions for 'go list -cover' (2024-02-01)
- `45f9ded` [release-branch.go1.22] cmd/compile: don't elide zero extension on top of signed values (2024-06-28)
- `4e548f2` [release-branch.go1.22] cmd/link: don't let dsymutil delete our temp directory (2024-06-21)
- `4b27560` [release-branch.go1.22] go/types: fix assertion failure when range over int is not permitted (2024-07-09)
- `9e148a4` [release-branch.go1.22] internal/bytealg: extend memchr result correctly on wasm (2024-07-17)
- `4c50f91` [release-branch.go1.22] cmd/internal/cov: close counter data files eagerly (2024-07-16)
- `2c88c1d` [release-branch.go1.22] cmd/trace/v2: handle the -pprof flag (2024-04-11)
- `70a1aae` [release-branch.go1.22] cmd/trace/v2: make the -pprof actually useful (2024-04-11)
- `8c8adff` [release-branch.go1.22] cmd/compile: add 0-sized-value simplification to copyelim (2024-02-29)
- `cb4eee6` [release-branch.go1.22] go1.22.6 (2024-08-06)
```

### go1.22.7

```text
- `0a525a3` [release-branch.go1.22] os: fix Chtimes test flakes (2024-08-08)
- `676d610` [release-branch.go1.22] cmd/fix: support go versions with patch release (2024-02-07)
- `e87be98` [release-branch.go1.22] runtime: on AIX, fix call to _cgo_sys_thread_create in _rt0_ppc64_aix_lib (2024-08-19)
- `b232596` [release-branch.go1.22] go/parser: track depth in nested element lists (2024-06-10)
- `2092294` [release-branch.go1.22] encoding/gob: cover missed cases when checking ignore depth (2024-05-03)
- `d4c5381` [release-branch.go1.22] go/build/constraint: add parsing limits (2024-06-20)
- `7529d09` [release-branch.go1.22] go1.22.7 (2024-09-05)
```

### go1.22.8

```text
- `71655f1` [release-branch.go1.22] cmd/cgo: correct padding required by alignment (2024-08-27)
- `6fab4b9` [release-branch.go1.22] runtime: size maps.Clone destination bucket array safely (2024-08-29)
- `b4086b7` [release-branch.go1.22] syscall: skip TestAmbientCapsUserns when restricted, document (2024-05-14)
- `aeccd61` [release-branch.go1.22] go1.22.8 (2024-10-01)
```

### go1.22.9

```text
- `29252e4` [release-branch.go1.22] runtime: fix TestGdbAutotmpTypes on gdb version 15 (2024-07-13)
- `e3fd4ba` [release-branch.go1.22] cmd/link: generate Mach-O UUID when -B flag is specified (2024-10-07)
- `c19e588` [release-branch.go1.22] cmd/cgo/internal/testcarchive: remove 1-minute timeout (2024-07-17)
- `8af39d3` [release-branch.go1.22] go1.22.9 (2024-11-06)
```

### go1.22.10

```text
- `3355db9` [release-branch.go1.22] time: accept "+01" in TestLoadFixed on OpenBSD (2024-11-04)
- `6f05fa7` [release-branch.go1.22] syscall: mark SyscallN as noescape (2024-11-05)
- `6d7a95a` [release-branch.go1.22] runtime: reserve 4kB for system stack on windows-386 (2024-11-12)
- `8f3f22e` [release-branch.go1.22] go1.22.10 (2024-12-03)
```

### go1.22.11

```text
- `223260b` [release-branch.go1.22] crypto/tls: fix Config.Time in tests using expired certificates (2025-01-02)
- `ae9996f` [release-branch.go1.22] runtime: hold traceAcquire across casgstatus in injectglist (2024-12-23)
- `19d2103` [release-branch.go1.22] crypto/x509: properly check for IPv6 hosts in URIs (2024-12-09)
- `b72d56f` [release-branch.go1.22] net/http: persist header stripping across repeated redirects (2024-11-22)
- `f072884` [release-branch.go1.22] go1.22.11 (2025-01-16)
```

### go1.22.12

```text
- `e0a01ac` [release-branch.go1.22] cmd/compile: fix write barrier coalescing (2025-01-11)
- `c3c6a50` [release-branch.go1.22] cmd/go/internal/modfetch: do not trust server to send all tags in shallow fetch (2025-01-14)
- `0cc45e7` [release-branch.go1.22] crypto/internal/fips140/nistec: make p256NegCond constant time on ppc64le (2025-01-22)
- `5817e65` [release-branch.go1.22] go1.22.12 (2025-02-04)
```
