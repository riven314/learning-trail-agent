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

### What is a "Shape"?
Types with same memory layout share a shape:
- all pointers have same shape (8 bytes on 64-bit)
- each primitive has its own shape

```go
First([]int{...})     // shape: int
First([]*Dog{...})    // shape: pointer
First([]*Cat{...})    // shape: pointer → shares code with *Dog!
```

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
