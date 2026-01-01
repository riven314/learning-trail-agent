# Go Error Interface: Implementation and Common Pitfalls

## Context
Understanding the error interface design in Go and its implications for error handling patterns. This complements the typed nil problem discussed in [go-interface-boxing-nil-internals.md](go-interface-boxing-nil-internals.md).

## Core Concept: error is an Interface Type

In Go, `error` is **not a concrete type** but an interface defined in the builtin package:

```go
type error interface {
    Error() string
}
```

Any type that implements an `Error() string` method automatically satisfies the error interface.

## Common Concrete Types Implementing error

### 1. `*errors.errorString` (from errors.New())
```go
err := errors.New("something went wrong")
// underlying concrete type: *errors.errorString
```

### 2. `*fmt.wrapError` (from fmt.Errorf() with %w)
```go
err := fmt.Errorf("wrapped: %w", originalErr)
// enables error unwrapping via errors.Unwrap()
```

### 3. Custom Error Types
```go
type MyError struct {
    Code    int
    Message string
}

func (e *MyError) Error() string {
    return fmt.Sprintf("code %d: %s", e.Code, e.Message)
}

var err error = &MyError{404, "not found"}
```

## Extracting Concrete Types: errors.As()

Since error is an interface, you can extract the underlying concrete type using type assertions or `errors.As()`:

```go
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    fmt.Println("Path:", pathErr.Path)
    fmt.Println("Op:", pathErr.Op)
}
```

**Why errors.As() is preferred over type assertions:**
- Works with wrapped errors (follows the error chain)
- Safer than direct type assertions
- Cleaner API for multi-level error inspection

## The Typed Nil Gotcha

The interface nature of error leads to a classic pitfall:

```go
func getError() error {
    var err *MyError = nil
    return err  // returns non-nil interface holding nil pointer!
}

err := getError()
if err != nil {  // TRUE! Interface is not nil
    fmt.Println("oops, unexpected branch")
}
```

**Root cause:** The interface value stores the concrete type (`*MyError`) in its type compartment, making the interface non-nil even though the underlying pointer is nil.

**Fix:** Return the interface type directly:
```go
func getError() error {
    return nil  // returns true nil (no concrete type attached)
}
```

See [go-interface-boxing-nil-internals.md](go-interface-boxing-nil-internals.md) for the complete explanation of interface boxing and typed nil mechanics.

## Key Takeaways

- `error` is an interface, not a concrete type - any type with `Error() string` satisfies it
- Common implementations: `errors.New()`, `fmt.Errorf()`, custom error structs
- Use `errors.As()` to extract concrete error types from the error chain
- Always return `error` interface directly from functions to avoid typed nil issues
- The interface nature enables flexible error handling but requires understanding boxing behavior

#GOLANG
