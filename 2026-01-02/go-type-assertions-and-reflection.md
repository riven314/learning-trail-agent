# Go Type Assertions and Reflection

**Context**: Learning Go fundamentals while understanding the OpenSQT market maker codebase. Coming from Python background, exploring how Go handles runtime type operations differently due to its static type system and package architecture constraints.

**Session**: `/Users/alexlwh/Desktop/crypto/archives/opensqt_market_maker`

## Type Assertions

### The Problem
- `interface{}` is like an unlabeled box - Go doesn't know what's inside at compile time
- Need runtime mechanism to extract the concrete type

### Basic Syntax
```go
// dangerous - panics if wrong type
value := box.(Type)

// safe - comma-ok pattern
value, ok := box.(Type)
if !ok {
    // handle type mismatch
}
```

### Type Switch
```go
switch v := box.(type) {
case int:
    // v is int
case string:
    // v is string
default:
    // unknown type
}
```

### Python Comparison
Similar to `isinstance()` checks, but Go requires explicit extraction:
```python
# Python
if isinstance(box, int):
    value = box

# Go equivalent
if value, ok := box.(int); ok {
    // use value
}
```

## Reflection

### Two Pillars
- `reflect.TypeOf(x)` - inspects TYPE compartment
- `reflect.ValueOf(x)` - inspects DATA compartment

### Kind vs Type
- **Kind**: underlying category (`reflect.Int`, `reflect.Struct`)
- **Type**: specific named type (`MyInt`, `OrderUpdate`)

### Key Methods
```go
v := reflect.ValueOf(x)

// type inspection
v.Kind()              // get category
v.Type()              // get specific type

// struct field access
v.FieldByName(name)   // find field by string name
v.IsValid()           // check if field exists

// type-safe extraction
v.CanInt()            // check before extracting
v.Int()               // get as int64
v.String()            // get as string
v.Float()             // get as float64

// pointer handling
v.Elem()              // dereference pointer
```

### Safety Pattern
Always check before extracting:
```go
field := v.FieldByName("Price")
if !field.IsValid() {
    return errors.New("field not found")
}
if !field.CanFloat() {
    return errors.New("field not a float")
}
price := field.Float()
```

### Performance Cost
- Reflection: ~100ns per operation
- Direct access: ~1ns per operation
- **~100x slower** - use sparingly

## Architectural Motivation

### Why This Design Exists in Market Maker Codebase

1. **Circular Import Problem**
   - Go forbids circular imports
   - `exchange` package can't import `position` if `position` imports `exchange`
   - Solution: use `interface{}` at boundaries to break the cycle

2. **Multiple Exchange Types**
   - Each exchange has own `OrderUpdate` struct
   - `binance.OrderUpdate ≠ bitget.OrderUpdate ≠ gate.OrderUpdate`
   - Same field NAMES, different TYPES
   - Can't pass one where another is expected

3. **Design Patterns Used**
   - **Reflection for Order Updates** (`main.go:154-199`)
     - WebSocket callback receives `interface{}`
     - Uses `reflect.FieldByName()` to extract fields by name
     - Works regardless of concrete type

   - **Type Switch for Positions** (`position/super_position_manager.go:1069-1104`)
     - Handle different return formats from different exchanges
     - Know all possible types at compile time

### Trade-off
Go makes you work harder at package boundaries, but gives compile-time safety within packages.

## Decision Tree: When to Use What

```
Know all possible types?
├─ YES → Type switch: switch v := x.(type) { case TypeA: ... }
└─ NO → Do types share field names?
        ├─ YES → Reflection: v.FieldByName("SharedField")
        └─ NO → Redesign types to share interface or use reflection
```

## Key Insight

Type assertion/reflection is the "boundary layer" where packages meet:
- **Within a package**: use static types for compile-time safety
- **At boundaries** (callbacks, interface returns): use `interface{}` + runtime type handling
- Think of it as the price Go pays for strict compile-time checking - you handle type flexibility at specific points rather than everywhere

#GOLANG
