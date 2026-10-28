# Go 1.23 Release Notes

## Introduction to Go 1.23

The latest Go release, version 1.23, arrives six months after [Go 1.22](/doc/go1.22).
Most of its changes are in the implementation of the toolchain, runtime, and libraries.
As always, the release maintains the Go 1 [promise of compatibility](/doc/go1compat).
We expect almost all Go programs to continue to compile and run as before.

## Changes to the language

The &ldquo;range&rdquo; clause in a &ldquo;for-range&rdquo; loop now accepts iterator functions of the following types

`func(func() bool)
func(func(K) bool)
func(func(K, V) bool)
`
as range expressions.
Calls of the iterator argument function produce the iteration values for the &ldquo;for-range&rdquo; loop.
For details see the [`iter`](/pkg/iter) package documentation, the
[language spec](/ref/spec#For_range), and the [Range over Function
Types blog post](/blog/range-functions).
For motivation see the 2022 [&ldquo;range-over-func&rdquo; discussion](/issue/56413).

Go 1.23 includes preview support for [generic type aliases](/issue/46477).
Building the toolchain with `GOEXPERIMENT=aliastypeparams` enables this feature within a package.
(Using generic alias types across package boundaries is not yet supported.)

## Tools

### Telemetry

Starting in Go 1.23, the Go toolchain can collect usage and breakage
statistics that help the Go team understand how the Go toolchain is
used and how well it is working. We refer to these statistics as
[Go telemetry](/doc/telemetry).

Go telemetry is an opt-in system*, controlled by the
[`go` `telemetry` command](/cmd/go/#hdr-Manage_telemetry_data_and_settings).
By default, the toolchain programs
collect statistics in counter files that can be inspected locally
but are otherwise unused (`go` `telemetry` `local`).

To help us keep Go working well and understand Go usage,
please consider opting in to Go telemetry by running
`go` `telemetry` `on`.
In that mode,
anonymous counter reports are uploaded to
[telemetry.go.dev](https://telemetry.go.dev) weekly,
where they are aggregated into graphs and also made
available for download by any Go contributors or users
wanting to analyze the data.
See “[Go Telemetry](/doc/telemetry)” for more details
about the Go Telemetry system.

### Go command

Setting the `GOROOT_FINAL` environment variable no longer has an effect
([#62047](/issue/62047)).
Distributions that install the `go` command to a location other than
`$GOROOT/bin/go` should install a symlink instead of relocating
or copying the `go` binary.

The new `go` `env` `-changed` flag causes the command to print only
those settings whose effective value differs from the default value
that would be obtained in an empty environment with no prior uses of the `-w` flag.

The new `go` `mod` `tidy` `-diff` flag causes the command not to modify
the files but instead print the necessary changes as a unified diff.
It exits with a non-zero code if updates are needed.

The `go` `list` `-m` `-json` command now includes new `Sum` and `GoModSum` fields.
This is similar to the existing behavior of the `go` `mod` `download` `-json` command.

The new `godebug` directive in `go.mod` and `go.work` declares a
[GODEBUG setting](/doc/godebug) to apply for the work module or workspace in use.

### Vet

The `go vet` subcommand now includes the
[stdversion](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/stdversion)
analyzer, which flags references to symbols that are too new for the version
of Go in effect in the referring file. (The effective version is determined
by the `go` directive in the file&rsquo;s enclosing `go.mod` file, and
by any [`//go:build` constraints](/cmd/go#hdr-Build_constraints)
in the file.)

For example, it will report a diagnostic for a reference to the
`reflect.TypeFor` function (introduced in go1.22) from a file in a
module whose go.mod file specifies `go 1.21`.

### Cgo

[`cmd/cgo`](/pkg/cmd/cgo) supports the new `-ldflags` flag for passing flags to the C linker.
The `go` command uses it automatically, avoiding &ldquo;argument list too long&rdquo;
errors with a very large `CGO_LDFLAGS`.

### Trace

The `trace` tool now better tolerates partially broken traces by attempting to
recover what trace data it can. This functionality is particularly helpful when
viewing a trace that was collected during a program crash, since the trace data
leading up to the crash will now [be recoverable](/issue/65319) under most
circumstances.

## Runtime

The traceback printed by the runtime after an unhandled panic or other
fatal error now indents the second and subsequent lines of the error
message (for example, the argument to panic) by a single tab, so that
it can be unambiguously distinguished from the stack trace of the
first goroutine. See [#64590](/issue/64590) for discussion.

## Compiler

The build time overhead to building with [Profile Guided Optimization](/doc/pgo) has been reduced significantly.
Previously, large builds could see 100%+ build time increase from enabling PGO.
In Go 1.23, overhead should be in the single digit percentages.

The compiler in Go 1.23 can now overlap the stack frame slots of local variables
accessed in disjoint regions of a function, which reduces stack usage
for Go applications.

For 386 and amd64, the compiler will use information from PGO to align certain
hot blocks in loops. This improves performance an additional 1-1.5% at
a cost of an additional 0.1% text and binary size. This is currently only implemented
on 386 and amd64 because it has not shown an improvement on other platforms.
Hot block alignment can be disabled with `-gcflags=[<packages>=]-d=alignhot=0`.

## Linker

The linker now disallows using a `//go:linkname` directive to refer to
internal symbols in the standard library (including the runtime) that
are not marked with `//go:linkname` on their definitions.
Similarly, the linker disallows references to such symbols from assembly
code.
For backward compatibility, existing usages of `//go:linkname` found in
a large open-source code corpus remain supported.
Any new references to standard library internal symbols will be disallowed.

A linker command line flag `-checklinkname=0` can be used to disable
this check, for debugging and experimenting purposes.

When building a dynamically linked ELF binary (including PIE binary), the
new `-bindnow` flag enables immediate function binding.

## Standard library

### Timer changes

Go 1.23 makes two significant changes to the implementation of
[`time.Timer`](/pkg/time#Timer) and [`time.Ticker`](/pkg/time#Ticker).

First, `Timer`s and `Ticker`s that are no longer referred to by the program
become eligible for garbage collection immediately, even if their
`Stop` methods have not been called.
Earlier versions of Go did not collect unstopped `Timer`s until after
they had fired and never collected unstopped `Ticker`s.

Second, the timer channel associated with a `Timer` or `Ticker` is
now unbuffered, with capacity 0.
The main effect of this change is that Go now guarantees
that for any call to a `Reset` or `Stop` method, no stale values
prepared before that call will be sent or received after the call.
Earlier versions of Go used channels with a one-element buffer,
making it difficult to use `Reset` and `Stop` correctly.
A visible effect of this change is that `len` and `cap` of timer channels
now returns 0 instead of 1, which may affect programs that
poll the length to decide whether a receive on the timer channel
will succeed.
Such code should use a non-blocking receive instead.

These new behaviors are only enabled when the main Go program
is in a module with a `go.mod` `go` line using Go 1.23.0 or later.
When Go 1.23 builds older programs, the old behaviors remain in effect.
The new [GODEBUG setting](/doc/godebug) [`asynctimerchan=1`](/pkg/time/#NewTimer)
can be used to revert back to asynchronous channel behaviors
even when a program names Go 1.23.0 or later in its `go.mod` file.

### New unique package

The new [`unique`](/pkg/unique) package provides facilities for
canonicalizing values (like &ldquo;interning&rdquo; or &ldquo;hash-consing&rdquo;).

Any value of comparable type may be canonicalized with the new
`Make[T]` function, which produces a reference to a canonical copy of
the value in the form of a `Handle[T]`.
Two `Handle[T]` are equal if and only if the values used to produce the
handles are equal, allowing programs to deduplicate values and reduce
their memory footprint.
Comparing two `Handle[T]` values is efficient, reducing down to a simple
pointer comparison.

### Iterators

The new [`iter`](/pkg/iter) package provides the basic definitions for working with
user-defined iterators.

The [`slices`](/pkg/slices) package adds several functions that work with iterators:

[All](/pkg/slices#All) returns an iterator over slice indexes and values.

- [Values](/pkg/slices#Values) returns an iterator over slice elements.

- [Backward](/pkg/slices#Backward) returns an iterator that loops over
a slice backward.

- [Collect](/pkg/slices#Collect) collects values from an iterator into
a new slice.

- [AppendSeq](/pkg/slices#AppendSeq) appends values from an iterator to
an existing slice.

- [Sorted](/pkg/slices#Sorted) collects values from an iterator into a
new slice, and then sorts the slice.

- [SortedFunc](/pkg/slices#SortedFunc) is like `Sorted` but with a
comparison function.

- [SortedStableFunc](/pkg/slices#SortedStableFunc) is like `SortFunc`
but uses a stable sort algorithm.

- [Chunk](/pkg/slices#Chunk) returns an iterator over consecutive
sub-slices of up to n elements of a slice.

The [`maps`](/pkg/maps) package adds several functions that work with iterators:

- [All](/pkg/maps#All) returns an iterator over key-value pairs from a map.

- [Keys](/pkg/maps#Keys) returns an iterator over keys in a map.

- [Values](/pkg/maps#Values) returns an iterator over values in a map.

- [Insert](/pkg/maps#Insert) adds the key-value pairs from an iterator to an existing map.

- [Collect](/pkg/maps#Collect) collects key-value pairs from an iterator into a new map and returns it.

### New structs package

The new [`structs`](/pkg/structs) package provides
types for struct fields that modify properties of
the containing struct type such as memory layout.

In this release, the only such type is
[`HostLayout`](/pkg/structs#HostLayout)
which indicates that a structure with a field of that
type has a layout that conforms to host platform
expectations. HostLayout should be used in types that
are passed to, returned from, or accessed
via a pointer passed to/from host APIs.
Without this marker, struct layout order is not
guaranteed by the language spec, though as of Go 1.23
the host and language layouts happen to match.

### Minor changes to the library

#### [`archive/tar`](/pkg/archive/tar/)

If the argument to [`FileInfoHeader`](/pkg/archive/tar#FileInfoHeader) implements the new [`FileInfoNames`](/pkg/archive/tar#FileInfoNames)
interface, then the interface methods will be used to set the Uname/Gname
of the file header. This allows applications to override the system-dependent
Uname/Gname lookup.

#### [`crypto/tls`](/pkg/crypto/tls/)

The TLS client now supports the Encrypted Client Hello [draft specification](https://www.ietf.org/archive/id/draft-ietf-tls-esni-18.html).
This feature can be enabled by setting the [`Config.EncryptedClientHelloConfigList`](/pkg/crypto/tls#Config.EncryptedClientHelloConfigList)
field to an encoded ECHConfigList for the host that is being connected to.

The [`QUICConn`](/pkg/crypto/tls#QUICConn) type used by QUIC implementations includes new events
reporting on the state of session resumption, and provides a way for
the QUIC layer to add data to session tickets and session cache entries.

3DES cipher suites were removed from the default list used when
[`Config.CipherSuites`](/pkg/crypto/tls#Config.CipherSuites) is nil. The default can be reverted by adding `tls3des=1` to
the GODEBUG environment variable.

The experimental post-quantum key exchange mechanism X25519Kyber768Draft00
is now enabled by default when [`Config.CurvePreferences`](/pkg/crypto/tls#Config.CurvePreferences) is nil.
The default can be reverted by adding `tlskyber=0` to the GODEBUG environment variable.
This can be useful when dealing with buggy TLS servers that do not handle large records correctly,
causing a timeout during the handshake (see [TLS post-quantum TL;DR fail](https://tldr.fail/)).

Go 1.23 changed the behavior of [`X509KeyPair`](/pkg/crypto/tls#X509KeyPair) and [`LoadX509KeyPair`](/pkg/crypto/tls#LoadX509KeyPair)
to populate the [`Certificate.Leaf`](/pkg/crypto/tls#Certificate.Leaf) field of the returned [`Certificate`](/pkg/crypto/tls#Certificate).
The new `x509keypairleaf` [GODEBUG setting](/doc/godebug) is added for this behavior.

#### [`crypto/x509`](/pkg/crypto/x509/)

[`CreateCertificateRequest`](/pkg/crypto/x509#CreateCertificateRequest) now correctly supports RSA-PSS signature algorithms.

[`CreateCertificateRequest`](/pkg/crypto/x509#CreateCertificateRequest) and [`CreateRevocationList`](/pkg/crypto/x509#CreateRevocationList) now verify the generated signature using the signer’s public key. If the signature is invalid, an error is returned. This has been the behavior of [`CreateCertificate`](/pkg/crypto/x509#CreateCertificate) since Go 1.16.

The [`x509sha1` GODEBUG setting](/pkg/crypto/x509#InsecureAlgorithmError) will
be removed in the next Go major release (Go 1.24). This will mean that `crypto/x509`
will no longer support verifying signatures on certificates that use SHA-1 based
signature algorithms.

The new [`ParseOID`](/pkg/crypto/x509#ParseOID) function parses a dot-encoded ASN.1 Object Identifier string.
The [`OID`](/pkg/crypto/x509#OID) type now implements the [`encoding.BinaryMarshaler`](/pkg/encoding#BinaryMarshaler),
[`encoding.BinaryUnmarshaler`](/pkg/encoding#BinaryUnmarshaler), [`encoding.TextMarshaler`](/pkg/encoding#TextMarshaler), [`encoding.TextUnmarshaler`](/pkg/encoding#TextUnmarshaler) interfaces.

#### [`database/sql`](/pkg/database/sql/)

Errors returned by [`driver.Valuer`](/pkg/driver#Valuer) implementations are now wrapped for
improved error handling during operations like [`DB.Query`](/pkg/database/sql#DB.Query), [`DB.Exec`](/pkg/database/sql#DB.Exec),
and [`DB.QueryRow`](/pkg/database/sql#DB.QueryRow).

#### [`debug/elf`](/pkg/debug/elf/)

The `debug/elf` package now defines [`PT_OPENBSD_NOBTCFI`](/pkg/debug/elf#PT_OPENBSD_NOBTCFI). This [`ProgType`](/pkg/debug/elf#ProgType) is
used to disable Branch Tracking Control Flow Integrity (BTCFI) enforcement
on OpenBSD binaries.

Now defines the symbol type constants [`STT_RELC`](/pkg/debug/elf#STT_RELC), [`STT_SRELC`](/pkg/debug/elf#STT_SRELC), and
[`STT_GNU_IFUNC`](/pkg/debug/elf#STT_GNU_IFUNC).

#### [`encoding/binary`](/pkg/encoding/binary/)

The new [`Encode`](/pkg/encoding/binary#Encode) and [`Decode`](/pkg/encoding/binary#Decode) functions are byte slice equivalents
to [`Read`](/pkg/encoding/binary#Read) and [`Write`](/pkg/encoding/binary#Write).
[`Append`](/pkg/encoding/binary#Append) allows marshaling multiple data into the same byte slice.

#### [`go/ast`](/pkg/go/ast/)

The new [`Preorder`](/pkg/go/ast#Preorder) function returns a convenient iterator over all the
nodes of a syntax tree.

#### [`go/types`](/pkg/go/types/)

The [`Func`](/pkg/go/types#Func) type, which represents a function or method symbol, now
has a [`Func.Signature`](/pkg/go/types#Func.Signature) method that returns the function&rsquo;s type, which
is always a `Signature`.

The [`Alias`](/pkg/go/types#Alias) type now has an [`Rhs`](/pkg/go/types#Rhs) method that returns the type on the
right-hand side of its declaration: given `type A = B`, the `Rhs` of A
is B. ([#66559](/issue/66559))

The methods [`Alias.Origin`](/pkg/go/types#Alias.Origin), [`Alias.SetTypeParams`](/pkg/go/types#Alias.SetTypeParams), [`Alias.TypeParams`](/pkg/go/types#Alias.TypeParams),
and [`Alias.TypeArgs`](/pkg/go/types#Alias.TypeArgs) have been added. They are needed for generic alias types.

By default, go/types now produces [`Alias`](/pkg/go/types#Alias) type nodes for type aliases.
This behavior can be controlled by the `GODEBUG` `gotypesalias` flag.
Its default has changed from 0 in Go 1.22 to 1 in Go 1.23.

#### [`math/rand/v2`](/pkg/math/rand/v2/)

The [`Uint`](/pkg/math/rand/v2#Uint) function and [`Rand.Uint`](/pkg/math/rand/v2#Rand.Uint) method have been added.
They were inadvertently left out of Go 1.22.

The new [`ChaCha8.Read`](/pkg/math/rand/v2#ChaCha8.Read) method implements the [`io.Reader`](/pkg/io#Reader) interface.

#### [`net`](/pkg/net/)

The new type [`KeepAliveConfig`](/pkg/net#KeepAliveConfig) permits fine-tuning the keep-alive
options for TCP connections, via a new [`TCPConn.SetKeepAliveConfig`](/pkg/net#TCPConn.SetKeepAliveConfig)
method and new KeepAliveConfig fields for [`Dialer`](/pkg/net#Dialer) and [`ListenConfig`](/pkg/net#ListenConfig).

The [`DNSError`](/pkg/net#DNSError) type now wraps errors caused by timeouts or cancellation.
For example, `errors.Is(someDNSErr, context.DeadlineExceedeed)`
will now report whether a DNS error was caused by a timeout.

The new `GODEBUG` setting `netedns0=0` disables sending EDNS0
additional headers on DNS requests, as they reportedly break the DNS
server on some modems.

#### [`net/http`](/pkg/net/http/)

[`Cookie`](/pkg/net/http#Cookie) now preserves double quotes surrounding a cookie value.
The new [`Cookie.Quoted`](/pkg/net/http#Cookie.Quoted) field indicates whether the [`Cookie.Value`](/pkg/net/http#Cookie.Value)
was originally quoted.

The new [`Request.CookiesNamed`](/pkg/net/http#Request.CookiesNamed) method retrieves all cookies that match the given name.

The new [`Cookie.Partitioned`](/pkg/net/http#Cookie.Partitioned) field identifies cookies with the Partitioned attribute.

The patterns used by [`ServeMux`](/pkg/net/http#ServeMux) now allow one or more spaces or tabs after the method name.
Previously, only a single space was permitted.

The new [`ParseCookie`](/pkg/net/http#ParseCookie) function parses a Cookie header value and
returns all the cookies which were set in it. Since the same cookie
name can appear multiple times the returned Values can contain
more than one value for a given key.

The new [`ParseSetCookie`](/pkg/net/http#ParseSetCookie) function parses a Set-Cookie header value and
returns a cookie. It returns an error on syntax error.

[`ServeContent`](/pkg/net/http#ServeContent), [`ServeFile`](/pkg/net/http#ServeFile), and [`ServeFileFS`](/pkg/net/http#ServeFileFS) now remove
the `Cache-Control`, `Content-Encoding`, `Etag`, and `Last-Modified`
headers when serving an error. These headers usually apply to the
non-error content, but not to the text of errors.

Middleware which wraps a [`ResponseWriter`](/pkg/net/http#ResponseWriter) and applies on-the-fly
encoding, such as `Content-Encoding: gzip`, will not function after
this change. The previous behavior of [`ServeContent`](/pkg/net/http#ServeContent), [`ServeFile`](/pkg/net/http#ServeFile),
and [`ServeFileFS`](/pkg/net/http#ServeFileFS) may be restored by setting
`GODEBUG=httpservecontentkeepheaders=1`.

Note that middleware which changes the size of the served content
(such as by compressing it) already does not function properly when
[`ServeContent`](/pkg/net/http#ServeContent) handles a Range request. On-the-fly compression
should use the `Transfer-Encoding` header instead of `Content-Encoding`.

For inbound requests, the new [`Request.Pattern`](/pkg/net/http#Request.Pattern) field contains the [`ServeMux`](/pkg/net/http#ServeMux)
pattern (if any) that matched the request. This field is not set when
`GODEBUG=httpmuxgo121=1` is set.

#### [`net/http/httptest`](/pkg/net/http/httptest/)

The new [`NewRequestWithContext`](/pkg/net/http/httptest#NewRequestWithContext) method creates an incoming request with
a [`context.Context`](/pkg/context#Context).

#### [`net/netip`](/pkg/net/netip/)

In Go 1.22 and earlier, using
[`reflect.DeepEqual`](/pkg/reflect#DeepEqual) to compare an
[`Addr`](/pkg/net/netip#Addr) holding an IPv4 address to one holding
the IPv4-mapped IPv6 form of that address incorrectly returned true,
even though the `Addr` values were different when comparing with `==` or
[`Addr.Compare`](/pkg/net/netip#Addr.Compare).
This bug is now fixed and all three approaches now report the same
result.

#### [`os`](/pkg/os/)

The [`Stat`](/pkg/os#Stat) function now sets the [`ModeSocket`](/pkg/os#ModeSocket) bit for
files that are Unix sockets on Windows. These files are identified
by having a reparse tag set to `IO_REPARSE_TAG_AF_UNIX`.

On Windows, the mode bits reported by [`Lstat`](/pkg/os#Lstat) and [`Stat`](/pkg/os#Stat) for
reparse points changed. Mount points no longer have [`ModeSymlink`](/pkg/os#ModeSymlink) set,
and reparse points that are not symlinks, Unix sockets, or dedup files
now always have [`ModeIrregular`](/pkg/os#ModeIrregular) set.
This behavior is controlled by the `winsymlink` setting.
For Go 1.23, it defaults to `winsymlink=1`.
Previous versions default to `winsymlink=0`.

The [`CopyFS`](/pkg/os#CopyFS) function copies an [`io/fs.FS`](/pkg/io/fs#FS) into the local filesystem.

On Windows, [`Readlink`](/pkg/os#Readlink) no longer tries to normalize volumes
to drive letters, which was not always even possible.
This behavior is controlled by the `winreadlinkvolume` setting.
For Go 1.23, it defaults to `winreadlinkvolume=1`.
Previous versions default to `winreadlinkvolume=0`.

On Linux with pidfd support (generally Linux v5.4+),
[`Process`](/pkg/os#Process)-related functions and methods use pidfd (rather
than PID) internally, eliminating potential mistargeting when a PID is reused
by the OS. Pidfd support is fully transparent to a user, except for additional
process file descriptors that a process may have.

#### [`path/filepath`](/pkg/path/filepath/)

The new [`Localize`](/pkg/path/filepath#Localize) function safely converts a slash-separated
path into an operating system path.

On Windows, [`EvalSymlinks`](/pkg/path/filepath#EvalSymlinks) no longer evaluates mount points,
which was a source of many inconsistencies and bugs.
This behavior is controlled by the `winsymlink` setting.
For Go 1.23, it defaults to `winsymlink=1`.
Previous versions default to `winsymlink=0`.

On Windows, [`EvalSymlinks`](/pkg/path/filepath#EvalSymlinks) no longer tries to normalize
volumes to drive letters, which was not always even possible.
This behavior is controlled by the `winreadlinkvolume` setting.
For Go 1.23, it defaults to `winreadlinkvolume=1`.
Previous versions default to `winreadlinkvolume=0`.

#### [`reflect`](/pkg/reflect/)

The new methods synonymous with the methods of the same name
in [`Value`](/pkg/reflect#Value) are added to [`Type`](/pkg/reflect#Type):

- [`Type.OverflowComplex`](/pkg/reflect#Type.OverflowComplex)

- [`Type.OverflowFloat`](/pkg/reflect#Type.OverflowFloat)

- [`Type.OverflowInt`](/pkg/reflect#Type.OverflowInt)

- [`Type.OverflowUint`](/pkg/reflect#Type.OverflowUint)

The new [`SliceAt`](/pkg/reflect#SliceAt) function is analogous to [`NewAt`](/pkg/reflect#NewAt), but for slices.

The [`Value.Pointer`](/pkg/reflect#Value.Pointer) and [`Value.UnsafePointer`](/pkg/reflect#Value.UnsafePointer) methods now support values of kind [`String`](/pkg/reflect#String).

The new methods [`Value.Seq`](/pkg/reflect#Value.Seq) and [`Value.Seq2`](/pkg/reflect#Value.Seq2) return sequences that iterate over the value
as though it were used in a for/range loop.
The new methods [`Type.CanSeq`](/pkg/reflect#Type.CanSeq) and [`Type.CanSeq2`](/pkg/reflect#Type.CanSeq2) report whether calling
[`Value.Seq`](/pkg/reflect#Value.Seq) and [`Value.Seq2`](/pkg/reflect#Value.Seq2), respectively, will succeed without panicking.

#### [`runtime/debug`](/pkg/runtime/debug/)

The [`SetCrashOutput`](/pkg/runtime/debug#SetCrashOutput) function allows the user to specify an alternate
file to which the runtime should write its fatal crash report.
It may be used to construct an automated reporting mechanism for all
unexpected crashes, not just those in goroutines that explicitly use
`recover`.

#### [`runtime/pprof`](/pkg/runtime/pprof/)

The maximum stack depth for `alloc`, `mutex`, `block`, `threadcreate` and `goroutine`
profiles has been raised from 32 to 128 frames.

#### [`runtime/trace`](/pkg/runtime/trace/)

The runtime now explicitly flushes trace data when a program crashes due to an
uncaught panic. This means that more complete trace data will be available in a
trace if the program crashes while tracing is active.

#### [`slices`](/pkg/slices/)

The [`Repeat`](/pkg/slices#Repeat) function returns a new slice that repeats the
provided slice the given number of times.

#### [`sync`](/pkg/sync/)

The [`Map.Clear`](/pkg/sync#Map.Clear) method deletes all the entries, resulting in
an empty [`Map`](/pkg/sync#Map). It is analogous to `clear`.

#### [`sync/atomic`](/pkg/sync/atomic/)

The new [`And`](/pkg/sync/atomic#And) and [`Or`](/pkg/sync/atomic#Or) operators apply a bitwise `AND` or `OR` to
the given input, returning the old value.

#### [`syscall`](/pkg/syscall/)

The syscall package now defines [`WSAENOPROTOOPT`](/pkg/syscall#WSAENOPROTOOPT) on Windows.

The [`GetsockoptInt`](/pkg/syscall#GetsockoptInt) function is now supported on Windows.

#### [`testing/fstest`](/pkg/testing/fstest/)

[`TestFS`](/pkg/testing/fstest#TestFS) now returns a structured error that can be unwrapped
(via method `Unwrap() []error`). This allows inspecting errors
using [`errors.Is`](/pkg/errors#Is) or [`errors.As`](/pkg/errors#As).

#### [`text/template`](/pkg/text/template/)

Templates now support the new &ldquo;else with&rdquo; action, which reduces template complexity in some use cases.

#### [`time`](/pkg/time/)

[`Parse`](/pkg/time#Parse) and [`ParseInLocation`](/pkg/time#ParseInLocation) now return an error if the time zone
offset is out of range.

On Windows, [`Timer`](/pkg/time#Timer), [`Ticker`](/pkg/time#Ticker), and functions that put the goroutine to sleep,
such as [`Sleep`](/pkg/time#Sleep), got their time resolution improved to 0.5ms instead of 15.6ms.

#### [`unicode/utf16`](/pkg/unicode/utf16/)

The [`RuneLen`](/pkg/unicode/utf16#RuneLen) function returns the number of 16-bit words in
the UTF-16 encoding of the rune. It returns -1 if the rune
is not a valid value to encode in UTF-16.

## Ports

### Darwin

As [announced](go1.22#darwin) in the Go 1.22 release notes,
Go 1.23 requires macOS 11 Big Sur or later;
support for previous versions has been discontinued.

### Linux

Go 1.23 is the last release that requires Linux kernel version 2.6.32 or later. Go 1.24 will require Linux kernel version 3.2 or later.

### OpenBSD

Go 1.23 adds experimental support for OpenBSD on 64-bit RISC-V (`GOOS=openbsd`, `GOARCH=riscv64`).

### ARM64

Go 1.23 introduces a new `GOARM64` environment variable, which specifies the minimum target version of the ARM64 architecture at compile time. Allowed values are `v8.{0-9}` and `v9.{0-5}`. This may be followed by an option specifying extensions implemented by target hardware. Valid options are `,lse` and `,crypto`.

The `GOARM64` environment variable defaults to `v8.0`.

### RISC-V

Go 1.23 introduces a new `GORISCV64` environment variable, which selects the [RISC-V user-mode application profile](https://github.com/riscv/riscv-profiles/blob/main/src/profiles.adoc) for which to compile. Allowed values are `rva20u64` and `rva22u64`.

The `GORISCV64` environment variable defaults to `rva20u64`.

### Wasm

The `go_wasip1_wasm_exec` script in `GOROOT/misc/wasm` has dropped support
for versions of `wasmtime` < 14.0.0.

## Patch Release Changelogs

### go1.23.1

```text
- `dbecb41` [release-branch.go1.23] os: fix Chtimes test flakes (2024-08-08)
- `3c93405` [release-branch.go1.23] os: use O_EXCL instead of O_TRUNC in CopyFS to disallow rewriting existing files does not exist (2024-08-16)
- `76346b3` [release-branch.go1.23] unique: use TypeFor instead of TypeOf to get type in Make (2024-08-21)
- `9166d2f` [release-branch.go1.23] go/types, types2: Named.cleanup must also handle *Alias types (2024-08-15)
- `8002845` [release-branch.go1.23] runtime: on AIX, fix call to _cgo_sys_thread_create in _rt0_ppc64_aix_lib (2024-08-19)
- `7afe17b` [release-branch.go1.23] go/types, types2: use max(fileVersion, go1.21) if fileVersion present (2024-08-16)
- `9c939a1` [release-branch.go1.23] runtime: store bp on cgocallback as unsafe.Pointer (2024-08-27)
- `9625a7f` [release-branch.go1.23] go/types, types2: unalias tilde terms in underIs (2024-08-15)
- `6de5a71` [release-branch.go1.23] database/sql: fix panic with concurrent Conn and Close (2024-08-22)
- `3d1f1f2` [release-branch.go1.23] cmd: vendor golang.org/x/telemetry@internal-branch.go1.23-vendor (2024-08-28)
- `53487e5` [release-branch.go1.23] go/parser: track depth in nested element lists (2024-06-10)
- `fa8ff1a` [release-branch.go1.23] encoding/gob: cover missed cases when checking ignore depth (2024-05-03)
- `032ac07` [release-branch.go1.23] go/build/constraint: add parsing limits (2024-06-20)
- `69234de` [release-branch.go1.23] go1.23.1 (2024-09-05)
```

### go1.23.2

```text
- `80ff7cd` [release-branch.go1.23] cmd/cgo: correct padding required by alignment (2024-08-27)
- `a886959` [release-branch.go1.23] runtime: size maps.Clone destination bucket array safely (2024-08-29)
- `82575f7` [release-branch.go1.23] internal/weak: shade pointer in weak-to-strong conversion (2024-09-04)
- `e6598e7` [release-branch.go1.23] os: dup pidfd if caller sets PidFD manually (2024-08-22)
- `a74951c` [release-branch.go1.23] unique: don't retain uncloned input as key (2024-09-04)
- `c8c6f9a` [release-branch.go1.23] syscall: on exec failure, close pidfd (2024-09-06)
- `fbddfae` [release-branch.go1.23] cmd/compile: fix wrong esacpe analysis for rangefunc (2024-09-18)
- `3b2e846` [release-branch.go1.23] runtime: if stop/reset races with running timer, return correct result (2024-09-07)
- `ed07b32` [release-branch.go1.23] go1.23.2 (2024-10-01)
```

### go1.23.3

```text
- `f8080ed` [release-branch.go1.23] runtime: fix TestGdbAutotmpTypes on gdb version 15 (2024-07-13)
- `9563300` [release-branch.go1.23] os: ignore SIGSYS in checkPidfd (2024-09-24)
- `cc16cdf` [release-branch.go1.23] runtime: clear isSending bit earlier (2024-10-02)
- `7fc8312` [release-branch.go1.23] os: add clone(CLONE_PIDFD) check to pidfd feature check (2024-06-11)
- `6495ce0` [release-branch.go1.23] syscall: use SYS_EXIT_GROUP in CLONE_PIDFD feature check child (2024-10-02)
- `35c010a` [release-branch.go1.23] runtime: fix GoroutineProfile stacks not getting null terminated (2024-08-30)
- `8d79bf7` [release-branch.go1.23] runtime: don't frob isSending for tickers (2024-10-14)
- `58babf6` [release-branch.go1.23] runtime,time: use atomic.Int32 for isSending (2024-10-21)
- `cfe0ae0` [release-branch.go1.23] runtime: uphold goroutine profile invariants in coroswitch (2024-10-23)
- `5472853` [release-branch.go1.23] cmd/link: generate Mach-O UUID when -B flag is specified (2024-10-07)
- `6ba3a8a` [release-branch.go1.23] internal/poll: keep copying after successful Sendfile return on BSD (2024-10-23)
- `958f3a0` [release-branch.go1.23] internal/poll: handle the special case of sendfile(2) sending the full chunk (2024-10-24)
- `a0d15cb` [release-branch.go1.23] runtime: fix MutexProfile missing root frames (2024-09-07)
- `1207de4` [release-branch.go1.23] runtime: reduce syscall.SyscallX stack usage (2024-10-08)
- `c390a1c` [release-branch.go1.23] go1.23.3 (2024-11-06)
```

### go1.23.4

```text
- `3726f07` [release-branch.go1.23] cmd/compile: use a non-fragile test for "does f contain closure c?" (2024-10-25)
- `777f43a` [release-branch.go1.23]time: accept "+01" in TestLoadFixed on OpenBSD (2024-11-04)
- `847cb6f` [release-branch.go1.23] syscall: mark SyscallN as noescape (2024-11-05)
- `d8adc6c` [release-branch.go1.23] runtime: prevent weak->strong conversions during mark termination (2024-11-01)
- `be062b7` [release-branch.go1.23] runtime: explicitly keep handle alive during getOrAddWeakHandle (2024-11-20)
- `25f042d` [release-branch.go1.23] runtime: reserve 4kB for system stack on windows-386 (2024-11-12)
- `5164a86` [release-branch.go1.23] cmd/trace: also show end stack traces (2024-11-26)
- `194de8f` [release-branch.go1.23] go1.23.4 (2024-12-03)
```

### go1.23.5

```text
- `69c8cfe` [release-branch.go1.23] cmd/compile/internal/importer: enable aliases (2024-08-09)
- `59b7d40` [release-branch.go1.23] runtime: update and restore g0 stack bounds at cgocallback (2024-07-22)
- `1576793` [release-branch.go1.23] net: don't use sendfile for irregular files; handle EOPNOTSUPP/ENOTSUP (2024-12-11)
- `3417000` [release-branch.go1.23] crypto/tls: fix Config.Time in tests using expired certificates (2025-01-02)
- `1dde0b4` [release-branch.go1.23] runtime: hold traceAcquire across casgstatus in injectglist (2024-12-23)
- `fdb8413` [release-branch.go1.23] crypto/x509: properly check for IPv6 hosts in URIs (2024-12-09)
- `bb8230f` [release-branch.go1.23] net/http: persist header stripping across repeated redirects (2024-11-22)
- `d04e3cb` [release-branch.go1.23] go1.23.5 (2025-01-16)
```

### go1.23.6

```text
- `9cbbf5e` [release-branch.go1.23] cmd/compile: fix write barrier coalescing (2025-01-11)
- `ab44565` [release-branch.go1.23] cmd/go/internal/modfetch: do not trust server to send all tags in shallow fetch (2025-01-14)
- `6644ed6` [release-branch.go1.23] crypto/internal/fips140/nistec: make p256NegCond constant time on ppc64le (2025-01-22)
- `a991f9c` [release-branch.go1.23] go1.23.6 (2025-02-04)
```

### go1.23.7

```text
- `22fdd35` [release-branch.go1.23] runtime: fix usleep on s390x/linux (2025-02-12)
- `2aaa388` [release-branch.go1.23] cmd/compile, runtime: use deferreturn as target PC for recover from deferrangefunc (2025-02-18)
- `c57e2bd` [release-branch.go1.23] syscall: don't send child signal when testing pidfd (2025-02-20)
- `acde84c` [release-branch.go1.23] runtime: use WCLONE when waiting on pidfd test child (2025-02-21)
- `d89fda2` [release-branch.go1.23] reflect: correctly handle method values in Seq (2025-02-21)
- `0fa31cb` [release-branch.go1.23] reflect: let Value.Seq return the iteration value correct type (2025-02-23)
- `9facf1f` [release-branch.go1.23] runtime/cgo: avoid errors from -Wdeclaration-after-statement (2025-02-26)
- `e477283` [release-branch.go1.23] runtime/cgo: avoid errors from -Wdeclaration-after-statement (2025-02-26)
- `25177ec` [release-branch.go1.23] all: updated vendored x/net with security fix (2025-02-27)
- `c01c4d4` [release-branch.go1.23] go1.23.7 (2025-03-04)
```

### go1.23.8

```text
- `45aade7` [release-branch.go1.23] all: updates vendored x/net (2025-03-04)
- `15e01a2` [release-branch.go1.23] net/http: reject newlines in chunk-size lines (2025-02-26)
- `ec6e84d` [release-branch.go1.23] internal/godebugs: fix changed version for winsymlink and winreadlinkvolume to 1.23 (2025-03-19)
- `c855149` [release-branch.go1.23] runtime: only set isExtraInC if there are no Go frames left (2025-03-14)
- `2c0a0fc` [release-branch.go1.23] runtime: skip TestCgoCallbackPprof on platforms with broken profiling (2025-03-17)
- `c72a2ba` [release-branch.go1.23] runtime: explicitly disable async preempt for internal/runtime (2025-03-05)
- `7a2cfb7` [release-branch.go1.23] go1.23.8 (2025-04-01)
```

### go1.23.9

```text
- `96537d5` [release-branch.go1.23] cmd/link: choose one with larger size for duplicated BSS symbols (2025-03-25)
- `8ff45d1` [release-branch.go1.23] runtime: fix 9-arg syscall on darwin/amd64 (2025-04-14)
- `f77084d` [release-branch.go1.23] go1.23.9 (2025-05-06)
```

### go1.23.10

```text
- `4ca7c01` [release-branch.go1.23] runtime/debug: document DefaultGODEBUG as a BuildSetting (2024-07-12)
- `8eeb134` [release-branch.go1.23] cmd/link: allow linkname reference to a TEXT symbol regardless of size (2025-05-21)
- `c2c89d9` [release-branch.go1.23] os: don't follow symlinks on Windows when O_CREATE|O_EXCL (2025-05-13)
- `b897e97` [release-branch.go1.23] net/http: strip sensitive proxy headers from redirect requests (2025-05-21)
- `d375ae5` [release-branch.go1.23] go1.23.10 (2025-06-05)
```

### go1.23.11

```text
- `3eedbde` [release-branch.go1.23] cmd/compile: do nil check before calling duff functions, on arm64 and amd64 (2025-05-29)
- `dac4567` [release-branch.go1.23] runtime: handle system goroutines later in goroutine profiling (2025-06-10)
- `136ea1a` [release-branch.go1.23] runtime: prevent mutual deadlock between GC stopTheWorld and suspendG (2025-06-14)
- `c46a0e6` [release-branch.go1.23] runtime: set mspan limit field early and eagerly (2025-06-18)
- `e919b33` [release-branch.go1.23] cmd/link: permit a larger size BSS reference to a smaller DATA symbol (2025-06-26)
- `e9d2c03` [release-branch.go1.23] cmd/go: disable support for multiple vcs in one module (2025-06-09)
- `0a75dd7` [release-branch.go1.23] go1.23.11 (2025-07-08)
```

### go1.23.12

```text
- `317a3c1` [release-branch.go1.23] runtime: stash allpSnapshot on the M (2025-06-27)
- `cea5be7` [release-branch.go1.23] cmd/cgo/internal/testsanitizers: disable ASLR for TSAN tests (2024-10-31)
- `6c9c80b` [release-branch.go1.23] cmd/go: skip known TestScript/build_trimpath_cgo failure on darwin/arm64 (2025-07-23)
- `e8794e6` [release-branch.go1.23] cmd/compile: for arm64 epilog, do SP increment with a single instruction (2025-07-21)
- `8fa31a2` [release-branch.go1.23] os/exec: fix incorrect expansion of "", "." and ".." in LookPath (2025-06-30)
- `8a924ca` [release-branch.go1.23] database/sql: avoid closing Rows while scan is in progress (2025-07-23)
- `dd8b7ad` [release-branch.go1.23] go1.23.12 (2025-08-06)
```
