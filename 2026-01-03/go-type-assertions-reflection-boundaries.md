# Go Type Assertions, Reflection & Boundary Patterns

**Context**: Deep dive into Go's type system while optimizing a multi-exchange trading system (market maker). Session focused on handling type conversions between exchange-specific types and common interfaces. Python developer perspective.

**Session Directory**: /Users/alexlwh/Desktop/crypto/archives/opensqt_market_maker

---

## Type Assertions

### Core Problem
`interface{}` is an unlabeled box - Go doesn't know concrete type at compile time. Need explicit extraction.

### Syntax Patterns
```go
// unsafe - panics if wrong type
value := box.(int)

// safe - comma-ok pattern
value, ok := box.(int)
if ok {
    // use value
}

// type switch - handle multiple types
switch v := box.(type) {
case int:    // v is int
case string: // v is string
default:     // v is interface{}
}
```

### vs Python
- Go type assertion ≈ Python's `isinstance()` check
- Go requires explicit extraction, Python uses duck typing
- Go fails at runtime if wrong type, Python just AttributeError

---

## Reflection

### Two-Box Model of Interfaces
Every interface value has:
- **TYPE compartment**: what type is stored
- **DATA compartment**: the actual value

Reflection inspects these:
- `reflect.TypeOf(x)` → inspects TYPE
- `reflect.ValueOf(x)` → inspects DATA

### Kind vs Type
- **Kind**: underlying category (int, struct, slice, ptr)
- **Type**: specific name (MyInt, OrderUpdate)

Example:
```go
type OrderID int64
x := OrderID(123)

reflect.TypeOf(x)  // "main.OrderID" (type name)
reflect.ValueOf(x).Kind()  // "int64" (kind)
```

### Key Methods
```go
v := reflect.ValueOf(x)
v.Kind()              // get category
v.FieldByName("Name") // find struct field by string name
v.IsValid()           // check if field exists
v.CanInt()            // check if can call Int()
v.Int()               // extract as int64
v.String()            // extract as string
v.Elem()              // dereference pointer
```

### Safety Pattern
```go
field := v.FieldByName("OrderID")
if field.IsValid() && field.CanInt() {
    value := field.Int()
}
```

### Performance Cost
- Reflection: ~100ns per operation
- Direct access: ~1ns per operation
- **~100x slower** - acceptable at boundaries, NOT in hot paths

---

## Nominal vs Structural Typing

### Go Uses Nominal Typing
Types are different if they have different NAMES, even with identical structure.

```go
type BinanceOrder struct { OrderID int64 }
type ExchangeOrder struct { OrderID int64 }

var b BinanceOrder
var e ExchangeOrder = b  // compile error - different types
```

### Python Uses Structural/Duck Typing
```python
binance_order = BinanceOrder()
process(binance_order)  # works if it has right attributes
```

### Why Go Does This
1. prevents accidental type confusion (Meters vs Feet)
2. enforces package isolation
3. makes dependencies explicit

---

## Circular Import Problem

### The Problem
```
exchange package → imports → position package
position package → imports → exchange package
❌ illegal in Go - import cycle not allowed
```

### The Solution
- use `interface{}` at package boundaries
- break cycle by not importing
- convert types at boundary layer (wrappers)

---

## Architectural Patterns for Type Boundaries

### Current Design (with reflection)
```
binance.OrderUpdate → interface{} → reflection in main.go → position.OrderUpdate
                                    (slow: ~500ns)
```

### Optimized Design (Native Type Callback)
```
binance.OrderUpdate → wrapper converts → exchange.OrderUpdate → main.go → position.OrderUpdate
                      (direct copy)       (direct copy: ~50ns total)
```

### Why Wrapper is the Right Place
1. wrapper knows both types (imports adapter package)
2. purpose-built for adaptation between layers
3. keeps main.go clean and fast

---

## Callback Patterns

### Function Types as Parameters
```go
type OrderUpdateCallback func(update OrderUpdate)  // no return = void

func StartOrderStream(ctx context.Context, callback OrderUpdateCallback) error
```

### Why Callbacks Don't Return Values
- push/event-driven pattern (fire and forget)
- WebSocket pushes data, doesn't wait for response
- errors handled inside callback

### Nested Callbacks (Bridge Pattern)
```go
// wrapper creates bridge between two callback types
func (w *wrapper) StartOrderStream(ctx, callback exchange.OrderUpdateCallback) error {
    return w.adapter.StartOrderStream(ctx, func(update binance.OrderUpdate) {
        // inner callback receives binance type
        // converts and calls outer callback with exchange type
        callback(exchange.OrderUpdate{
            OrderID: update.OrderID,
            // ... field copy
        })
    })
}
```

---

## Python → Go Mental Model Shifts

| Python | Go | Implication |
|--------|-----|------------|
| duck typing | nominal typing | must explicitly convert between types |
| no import cycles (usually) | strict no cycles | use interface{} at boundaries |
| isinstance() | type assertion | explicit type checking required |
| getattr() | reflection | much slower, use sparingly |
| callbacks just work | typed callbacks | function signatures must match |

---

## Decision Framework

### When to Use Type Assertion
- know all possible types at compile time
- limited number of types (2-5)
- use type switch

### When to Use Reflection
- types unknown at compile time
- many types with shared field names
- boundary/adapter code (not hot paths)

### When to Redesign
- reflection in hot paths
- complex nested type handling
- consider: typed callbacks, interface methods, or converter injection

---

## The Boundary Layer Concept

```
Within Package          At Boundary              Within Package
──────────────         ─────────────            ──────────────
static types     →     interface{} or      →    static types
compile-time           type conversion          compile-time
fast                   acceptable overhead      fast
```

**Core principle**: push type conversion to the BOUNDARY, keep the CORE type-safe.

---

## Key Insights

1. **Reflection is a tool for boundaries** - use at package boundaries where you're adapting between incompatible type systems, never in hot paths

2. **Nominal typing enforces architecture** - Go's type system forces you to think about package dependencies and boundaries explicitly

3. **Callbacks as adapters** - nested callbacks create clean bridges between different type systems without coupling packages

4. **Performance matters at scale** - in HFT systems, 100x slowdown from reflection matters when processing thousands of events per second

5. **Interface{} breaks type safety temporarily** - acceptable at boundaries, but convert back to concrete types ASAP

#GOLANG #QUANT #ALGO-TRADING
