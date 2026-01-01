# Go Fundamentals

Context: Core Go language concepts learned through building a high-frequency market maker system.

## Struct Tags - Serialization Metadata

Struct field tags provide compile-time metadata for serialization/deserialization:

```go
type Config struct {
    APIKey    string `yaml:"api_key" json:"api_key"`
    APISecret string `yaml:"api_secret" json:"api_secret"`
    Symbol    string `yaml:"symbol" json:"symbol"`
}
```

**Key points:**
- Tags are string literals in backticks
- Common tags: `yaml:"field_name"`, `json:"field_name"`
- Multiple tags supported in same field
- Used by reflection-based libraries (yaml, json, etc.)

## Pointers - Memory Reference Control

Explicit control over value vs reference semantics:

```go
// Value copy - changes local copy only
func updateValue(config Config) {
    config.Symbol = "BTCUSDT"
}

// Pointer reference - changes original
func updatePointer(config *Config) {
    config.Symbol = "BTCUSDT"
}
```

**Key symbols:**
- `*Type` - pointer type declaration
- `&variable` - get address of variable
- `*pointer` - dereference pointer

**Auto-dereferencing:**
```go
config := &Config{}
config.Symbol = "BTCUSDT"  // auto-dereferences, no need for (*config).Symbol
```

**When to use pointers:**
- Modify struct in function
- Avoid copying large structs
- Interface methods need to modify receiver
- Nil-able values (pointers can be nil, values cannot)

## Error Handling - Explicit Returns

Explicit error returns instead of exceptions:

```go
func LoadConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("failed to read config: %w", err)
    }

    var config Config
    if err := yaml.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("failed to parse config: %w", err)
    }

    return &config, nil
}
```

**Pattern:**
- Functions return `(result, error)` tuple
- Check `if err != nil` immediately after each call
- `%w` wraps errors for error chain
- Early returns keep happy path unindented

## Interfaces - Implicit Satisfaction

Interfaces are satisfied implicitly (duck typing at compile time):

```go
type IExchange interface {
    PlaceOrder(symbol string, side string, price float64) error
    GetBalance() (float64, error)
}

type BinanceAdapter struct {
    apiKey string
}

// No "implements IExchange" declaration needed
func (b *BinanceAdapter) PlaceOrder(symbol string, side string, price float64) error {
    // implementation
    return nil
}

func (b *BinanceAdapter) GetBalance() (float64, error) {
    // implementation
    return 0, nil
}
```

**Key insight:** If a type has all methods of an interface, it automatically satisfies that interface.

**Naming convention:** prefix with `I` (IExchange, ILogger) or suffix with `er` (Reader, Writer)

## Design Patterns in Exchange Package

**Factory Pattern:**
```go
func CreateExchange(config *Config) (IExchange, error) {
    switch config.CurrentExchange {
    case "binance":
        return NewBinanceWrapper(config), nil
    case "bitget":
        return NewBitgetWrapper(config), nil
    default:
        return nil, fmt.Errorf("unknown exchange: %s", config.CurrentExchange)
    }
}
```

**Adapter Pattern:**
- `binance/adapter.go` - implements Binance-specific logic
- `wrapper_binance.go` - adapts BinanceAdapter to IExchange interface

**Package structure:**
```
exchange/
├── interface.go         # IExchange interface
├── factory.go           # Factory to create instances
├── binance/
│   ├── adapter.go       # Binance-specific implementation
│   └── websocket.go     # Binance WebSocket handling
└── wrapper_binance.go   # Adapts BinanceAdapter to IExchange
```

## Context - Cancellation and Timeouts

Standard way to handle cancellation, deadlines, and request-scoped values:

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()  // cleanup when function exits

// Pass context to goroutines
go func(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return  // cancelled, clean exit
        case data := <-dataChan:
            process(data)
        }
    }
}(ctx)

// Trigger cancellation
cancel()  // all goroutines receive ctx.Done() signal
```

**Common patterns:**
- `context.Background()` - root context
- `context.WithCancel(parent)` - cancellable context
- `context.WithTimeout(parent, duration)` - auto-cancel after timeout
- Pass context as first parameter to functions

## Goroutines - Lightweight Concurrency

Lightweight threads managed by Go runtime:

```go
// Start goroutine with 'go' keyword
go func() {
    processData()
}()

// Named function
go processData()

// Goroutine with parameters
go func(symbol string) {
    monitor(symbol)
}("BTCUSDT")
```

**Key characteristics:**
- Extremely cheap (2KB stack vs 2MB for OS threads)
- Multiplexed onto OS threads (M:N scheduling)
- Launched with `go` keyword
- No return values (use channels to communicate results)

## Channels - Goroutine Communication

Typed conduits for sending/receiving values between goroutines:

```go
// Create channel
priceChan := make(chan float64)

// Send to channel (blocks until received)
priceChan <- 50000.0

// Receive from channel (blocks until sent)
price := <-priceChan

// Buffered channel (non-blocking up to buffer size)
dataChan := make(chan string, 100)

// Close channel (only sender should close)
close(priceChan)

// Receive with closed check
price, ok := <-priceChan
if !ok {
    // channel closed
}
```

**Select statement** (switch for channels):
```go
select {
case price := <-priceChan:
    handlePrice(price)
case <-ctx.Done():
    return  // context cancelled
case <-time.After(5 * time.Second):
    // timeout after 5 seconds
}
```

## Defer - Cleanup Guarantee

Schedules function call to run when surrounding function returns:

```go
func processFile(path string) error {
    file, err := os.Open(path)
    if err != nil {
        return err
    }
    defer file.Close()  // guaranteed to run on return

    // multiple defers execute in LIFO order
    defer fmt.Println("done")        // runs second
    defer fmt.Println("processing")  // runs first

    return process(file)
}
```

**Common patterns:**
```go
// Context cancellation
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

// Mutex unlock
mu.Lock()
defer mu.Unlock()

// Resource cleanup
conn, _ := db.Connect()
defer conn.Close()
```

**Execution order:** LIFO (last deferred runs first), useful for symmetric setup/teardown

#GOLANG #ALGO-TRADING
