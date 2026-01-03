# Go Composition & Embedding

**Context**: Understanding Go's approach to code reuse through composition rather than inheritance. This came up while working on the OpenSQT Market Maker project where exchange adapters use embedding patterns extensively.

**Session Directory**: `/Users/alexlwh/Desktop/crypto/archives/opensqt_market_maker`

## Core Concept

Go has **no inheritance**. Instead, it uses **composition** ("has-a" relationships) to achieve code reuse. **Embedding** is syntactic sugar that makes composition feel like inheritance by promoting fields and methods.

## Two Ways to Compose

```go
// Named field (explicit composition)
type Car struct {
    engine Engine      // must access via car.engine.Start()
}

// Embedding (anonymous field)
type Car struct {
    Engine             // can access via car.Start() directly
}
```

## What Embedding Gives You

| Feature | Behavior |
|---------|----------|
| Field promotion | `car.Horsepower` instead of `car.Engine.Horsepower` |
| Method promotion | `car.Start()` instead of `car.Engine.Start()` |
| Explicit access | `car.Engine.Start()` always works |
| Multiple embedding | Embed multiple structs, get all their capabilities |

## Handling Collisions

When embedded structs share the same field/method name, you **must** use explicit access:

```go
car.A.Name    // not car.Name
car.B.Name
```

## Why Composition > Inheritance

| Inheritance Problems | Composition Benefits |
|---------------------|---------------------|
| Tight coupling | Loose coupling — components independent |
| Rigid hierarchies | Flexible — easy to swap/change |
| Diamond problem | Explicit resolution required |
| Forces "is-a" relationships | Natural "has-a" relationships |

## Key Patterns

```go
// Struct embedding (code reuse)
type Service struct {
    Logger
    Database
}

// Interface embedding (compose contracts)
type ReadWriter interface {
    Reader
    Writer
}

// Override by redefining method
func (s Service) Log(msg string) {
    s.Logger.Log("[SERVICE] " + msg)
}
```

## Mental Model

Think of embedding as **automatic delegation** — the outer struct delegates calls to the embedded struct without you writing boilerplate forwarding methods.

---

#GOLANG
