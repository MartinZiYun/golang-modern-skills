# Language Feature Evolution (Go 1.19-1.27)

Organized by topic, with the introducing version noted for each item. Prefer the modern form when writing code in a relevant scenario.

## Generics

Generics were introduced in Go 1.18 and have been enhanced continuously since 1.19.

### Generic Type Aliases (1.24, stable)

A type alias may be parameterized like a defined type. Previewed in 1.23 (`GOEXPERIMENT=aliastypeparams`), stable in 1.24.

```go
// 1.24+: generic type alias
type List[T any] = []T

type Number = cmp.Ordered // constraints can be aliased too
```

### Generic Self-Referential Constraints (1.26)

A generic type may now refer to itself in its constraint, allowing "instantiate with a type like itself":

```go
// 1.26+
type Adder[A Adder[A]] interface {
    Add(A) A
}

func algo[A Adder[A]](x, y A) A {
    return x.Add(y)
}
```

Previously the self-reference to `Adder` on the first line was disallowed.

### Generic Methods (1.27)

Method declarations may now have their own type parameters. The most significant language feature in 1.27. Previously generic functions could only be declared at package scope; now they can live in a type's namespace.

```go
// 1.27+: a method with its own type parameters
type Set[K comparable] struct{ m map[K]struct{} }

func (s *Set[K]) Filter[V any](f func(K) bool) *Set[K] {
    //     ^^^ the method's own type param, independent of receiver K
}
```

**Limitations**: interface methods may not declare type parameters; generic methods cannot implement interface methods.

### Relaxed `comparable` Constraint (1.20)

The `comparable` constraint now accepts non-strictly-comparable type arguments (e.g. interface types, composite types containing interfaces) - comparison may panic at runtime but compiles. This allows generic map key types to accept interface types.

### Improved Type Inference (1.21)

1.21 significantly improved type inference: generic function arguments may themselves be generic functions; method-based inference when assigning to interfaces; correct inference for multiple untyped constants of different kinds. Practical effect: **fewer cases require explicit type arguments**.

```go
// 1.21+: inference is more powerful, many cases need no explicit instantiation
s := slices.IndexFunc(items, func(x T) bool { return x.ID == id }) // inferred
```

### Generalized Function Type Inference (1.27)

Type inference now applies in all contexts where a generic function is assigned to or converted to a matching function type, not only direct calls.

## Loops & Iterators

### Per-Iteration Loop Variables (1.22)

**Important behavior change**. Starting in 1.22, variables declared by a `for` loop are created anew each iteration, eliminating the loop-variable-capture trap at its root.

```go
// 1.22+: safe - each iteration is an independent variable
for _, v := range items {
    go func() { process(v) }()  // no longer needs v := v
}
```

In 1.21 and earlier you had to manually copy `v := v`. **New code does not need this workaround.**

### Range Over Integers (1.22)

`range` can iterate directly over an integer expression:

```go
// 1.22+
for i := range 10 {        // i: 0,1,...,9
    fmt.Println(10 - i)
}

n := getItemCount()
for range n { work() }     // only care about the count, no need for i
```

### Range Over Function Iterators (1.23)

The range expression can be an iterator function. This is the foundation of Go's iterator pattern. Supported signatures:

```go
func(func() bool)          // no values
func(func(K) bool)         // single value
func(func(K, V) bool)      // two values (key/value)
```

The iterator calls the yield function to produce iteration values; returning `false` from yield stops iteration.

```go
// 1.23+: defining an iterator
func Backward[T any](s []T) func(func(int, T) bool) {
    return func(yield func(int, T) bool) {
        for i := len(s) - 1; i >= 0; i-- {
            if !yield(i, s[i]) {
                return
            }
        }
    }
}

// Usage
for i, v := range Backward(s) {
    fmt.Println(i, v)
}

// Using iterator functions provided by the standard library
for k, v := range maps.All(m) { }
for v := range slices.Values(s) { }
for _, c := range slices.Chunk(s, 3) { }  // chunked iteration
```

**When writing iterators**: the yield function must not be saved and called after the loop, nor called concurrently.

## Built-in Functions

### min / max (1.21)

Compute the smallest/largest of any ordered values; accepts a fixed but variable number of arguments:

```go
max(3, a, b, 9)          // 9 (if a,b < 9)
min(1.5, 2.5)            // 1.5
max("apple", "banana")   // "banana" (lexicographic)
```

### clear (1.21)

Deletes all elements from a map or zeroes all elements of a slice:

```go
clear(m)   // map is empty, len=0
clear(s)   // slice elements all zeroed, len unchanged
```

### new(expr) with Initial Value (1.26)

The operand to `new` may now be an expression specifying the initial value. Especially useful for optional fields represented as pointers:

```go
// 1.26+
age := new(yearsSince(born))  // *int, pointing to the result
p := Person{Name: name, Age: age}

// Equivalent old form:
// a := yearsSince(born); p := Person{Name: name, Age: &a}
```

## Type System Enhancements

### Slice to Array Conversion (1.20)

A slice can be converted directly to a fixed-length array (no unsafe needed):

```go
// 1.20+
s := []byte{1, 2, 3, 4, 5}
a := [4]byte(s)      // takes the first 4 elements
// 1.17 introduced conversion to array pointer: *[4]byte(s)

// Note: panics if length exceeds slice len
```

### Struct Literal Field Selector (1.27)

A struct literal key may now be any valid field selector (nested field path), not just a top-level field name:

```go
// 1.27+
type Inner struct{ X int }
type Outer struct{ Inner; Y int }

o := Outer{Inner: Inner{X: 1}, Y: 2}        // old form
o := Outer{Inner.X: 1, Y: 2}                // 1.27 new form (nested field directly)
```

### New unsafe Functions (1.20)

The `unsafe` package adds `SliceData`, `String`, and `StringData`. Together with 1.17's `Slice`, these provide complete construction/deconstruction of slices and strings without depending on their memory representation. **Prefer these over manually computing pointer offsets.**

### Struct / Array Comparison Order (1.20)

The spec now defines: structs are compared field-by-field and arrays element-by-element, in order, stopping at the first mismatch. Affects edge cases around whether certain comparisons panic. Existing program behavior is unchanged.

## Other Language Changes

### panic(nil) Is No Longer Error-Free (1.21)

`panic(nil)` or passing an untyped nil now produces a `*runtime.PanicNilError` runtime panic, ensuring `recover` returns non-nil. Old code can restore prior behavior with `GODEBUG=panicnil=1` (auto-enabled when go.mod declares `go 1.20` or earlier).

### Package Initialization Order Specified (1.21)

Packages are sorted by import path and initialized in order as their dependencies become ready. **Do not rely on the previously-undefined ordering**; cross-package side effects should be expressed via explicit init or explicit calls.

### Core Types Notion Removed (1.25)

The "core types" concept was removed from the language spec in favor of dedicated prose. No direct impact on writing code; the spec is clearer.

### Simpler Closure Naming (1.27, compiler)

The compiler generates simpler symbol names for closures and may merge identical closures' code. **Do not rely on closure symbol names for equality comparisons.**
