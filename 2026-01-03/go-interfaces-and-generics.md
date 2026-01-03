# Go Interfaces and Generics

**Context**: Continuation of Go type system learnings. This note complements earlier learnings on type assertions and reflection by covering interface semantics and the generics system introduced in Go 1.18. Focus on understanding when to use each approach for type abstraction. Python developer perspective.

**Session Directory**: /Users/alexlwh/Desktop/crypto/archives/opensqt_market_maker

---

## Interfaces in Go

### Implicit Implementation
Go interfaces define method signatures. Types implement interfaces implicitly - no "implements" keyword needed.

**Rule**: If a type has all required methods, it satisfies the interface.

```go
type Stringer interface {
    String() string
}

type Dog struct{}
func (d Dog) String() string { return "dog" }  // Dog implements Stringer automatically
```

### Python Comparison
- Go interfaces ≈ Python's Protocol (PEP 544) or duck typing
- Key difference: Go checks at **compile-time**, Python at **runtime**

---

## The Empty Interface (`interface{}` / `any`)

### Why It Accepts Everything
- empty interface has zero methods
- every type has at least zero methods
- therefore, every type satisfies `interface{}`

```go
var x interface{}
x = 42           // ✅ int has zero+ methods
x = "hello"      // ✅ string has zero+ methods
x = Dog{}        // ✅ Dog has zero+ methods
```

### Internal Structure (Two-Compartment Model)
```
┌─────────────────────────────────┐
│        interface{} value        │
├────────────────┬────────────────┤
│  Type: int     │  Value: 42     │
└────────────────┴────────────────┘
```

To use the underlying value, must type-assert:
```go
s := x.(string)        // unsafe - panics if wrong
val, ok := x.(string)  // safe - ok=false if wrong
```

### `any` Alias (Go 1.18+)
```go
type any = interface{}  // just an alias, identical behavior
```

---

## Generics (Go 1.18+)

### Purpose
- type-parameterized functions and types
- compile-time type safety (no runtime assertions needed)
- avoid code duplication for different types

### Syntax
```go
func First[T any](items []T) T { return items[0] }
//        └─┬──┘  └──┬──┘  └┬┘
//    type param   uses T  returns T
```

| Part | Meaning |
|------|---------|
| `[T any]` | declares type parameter T, accepts any type |
| `items []T` | parameter using the type parameter |
| `T` (return) | return type matches input type |

### Constraints
Restrict which types are allowed:
```go
func Sum[T int | float64](items []T) T { ... }  // only int or float64

type Number interface {
    int | float64 | int64
}
func Sum[T Number](items []T) T { ... }  // using named constraint
```

---

## Generics vs `interface{}` Comparison

| Aspect | `interface{}` | Generics |
|--------|---------------|----------|
| type checking | runtime | compile-time |
| type assertion | required | not needed |
| type safety | can panic | errors at compile |
| performance | boxing overhead | optimized at compile |

### Example Comparison
```go
// interface{}: type erased, must assert
func FirstAny(items []interface{}) interface{} { return items[0] }
result := FirstAny([]interface{}{1, 2})
n := result.(int)  // runtime check, can panic

// generics: type preserved throughout
func First[T any](items []T) T { return items[0] }
result := First([]int{1, 2})
n := result + 10   // works directly, type is still int
```

---

## How Go Compiles Generics (GCShape Stenciling)

### The Problem
- naive approach: generate code for every type used → huge binaries
- alternative: use interface{} internally → slow (boxing, type assertions)

### Go's Hybrid Solution: GCShape Stenciling
1. analyze all call sites to find which concrete types are used
2. group types by "shape" (memory layout)
3. generate one specialized version per shape

### What is a "Shape"? (Detailed)

"Shape" refers to how a type is represented in memory — specifically its **size**, **alignment**, and **whether the garbage collector needs to scan it for pointers**.

#### Types That Share a Shape

**All pointers have the same shape** — they're all just memory addresses (8 bytes on 64-bit systems):

```go
*int, *string, *Dog, *Cat  // all same shape: "pointer"
```

**All types that "contain a pointer" in the same way:**

```go
[]int, []string, []Dog     // slices: all same shape (pointer + len + cap)
map[string]int, map[int]Dog // maps: all same shape (pointer to runtime struct)
```

#### Types That Have Different Shapes

**Different sizes:**
```go
int8    // 1 byte
int32   // 4 bytes
int64   // 8 bytes
// All different shapes - CPU handles differently
```

**Different memory layouts:**
```go
struct{ a int }           // 8 bytes
struct{ a, b int }        // 16 bytes
struct{ a int; b string } // different layout (contains pointer for GC)
// All different shapes
```

#### Code Sharing Example

```go
func Print[T any](v T) { fmt.Println(v) }

Print(&Dog{})    // shape: pointer  ─┐
Print(&Cat{})    // shape: pointer  ─┴─ SHARE generated code

Print(int32(1))  // shape: int32   ─┐
Print(int64(2))  // shape: int64   ─┴─ SEPARATE generated code
```

#### The Key Question

Shape is essentially asking: **"Can the CPU and garbage collector treat these types identically at the machine level?"**

- **Yes** → Share one implementation (smaller binary)
- **No** → Generate separate code (type-specific optimization)

#### Shape Categories Summary

| Shape Category | Examples | Why Same Shape |
|---------------|----------|----------------|
| Pointer | `*int`, `*Dog`, `*Cat` | All 8 bytes, all need GC scanning |
| Slice | `[]int`, `[]string` | All pointer+len+cap (24 bytes) |
| Map | `map[K]V` (any K,V) | All pointer to runtime hash table |
| int64 | `int64`, `int` (on 64-bit) | Same size, no GC scanning |
| int32 | `int32`, `rune` | Same size, no GC scanning |

### Trade-off

| Approach | Binary Size | Performance |
|----------|-------------|-------------|
| full monomorphization (C++) | large | best |
| GCShape stenciling (Go) | medium | good |
| full type erasure (Java) | small | slower |

---

## When to Use What

| Scenario | Use |
|----------|-----|
| known types, need performance | generics |
| unknown types at compile time | `interface{}` + type assertion |
| collection of mixed types | `[]interface{}` |
| type-safe collections | `[]T` with generics |
| polymorphic behavior | interface with methods |

---

## Connection to Previous Learnings

This builds on the type assertion and reflection concepts from earlier today:

```
                    ┌─────────────────────────────────────┐
                    │          Go Type Handling           │
                    ├─────────────────────────────────────┤
                    │                                     │
compile-time ──────►│  Generics [T any]                  │──► best performance
                    │  type preserved, no assertions      │
                    │                                     │
                    ├─────────────────────────────────────┤
                    │                                     │
runtime ───────────►│  interface{} + type assertion      │──► moderate
                    │  value, ok := x.(Type)              │
                    │                                     │
                    ├─────────────────────────────────────┤
                    │                                     │
dynamic ───────────►│  Reflection                        │──► slowest
                    │  reflect.ValueOf(x).FieldByName()   │    (use at boundaries)
                    │                                     │
                    └─────────────────────────────────────┘
```

---

## Key Insights

1. **Interfaces are compile-time contracts** - implicit implementation means types satisfy interfaces just by having the right methods, no explicit declaration needed

2. **Empty interface is universal container** - `interface{}` (or `any`) accepts anything because every type has at least zero methods, but requires type assertions to use

3. **Generics preserve type information** - unlike `interface{}`, generics maintain compile-time type safety without runtime assertions, best for known types

4. **GCShape stenciling balances trade-offs** - Go groups types by memory layout to avoid both code bloat and performance penalties

5. **Choose based on when you know types** - if types known at compile time, use generics; if types emerge at runtime (like JSON parsing), use `interface{}`

#GOLANG
