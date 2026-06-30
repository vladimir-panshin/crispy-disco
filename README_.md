# go-cache

A generic, thread-safe in-memory cache for Go, with per-key TTL and automatic background cleanup.

![Go](https://img.shields.io/badge/Go-1.26-00ADD8?logo=go&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

Type-safe via generics — `Cache[V]` stores values of a concrete type, no `interface{}`, no casts on the way out. Expired entries are removed both lazily (on access) and by an optional background sweeper.

## Install

```bash
go get github.com/vladimir-panshin/go-cache
```

## Quick start

```go
package main

import (
	"fmt"
	"time"

	"github.com/vladimir-panshin/go-cache"
)

func main() {
	// Background cleanup runs every minute. Pass 0 to disable it.
	c := cache.New[string](time.Minute)
	defer c.Close()

	c.Set("greeting", "hello", 5*time.Minute) // expires in 5 minutes
	c.Set("permanent", "stays", 0)            // never expires

	if v, ok := c.Get("greeting"); ok {
		fmt.Println(v) // hello
	}
}
```

The value type is chosen at construction, so the compiler enforces it — `Get` returns that type directly:

```go
type Session struct {
	UserID string
	Roles  []string
}

c := cache.New[Session](time.Minute)
defer c.Close()

c.Set("sess_1", Session{UserID: "u1", Roles: []string{"admin"}}, time.Hour)

s, ok := c.Get("sess_1") // s is a Session, not interface{}
```

## API

| Method | Description |
|---|---|
| `New[V](cleanupInterval)` | Create a cache. `cleanupInterval <= 0` disables the background sweeper. |
| `Set(key, value, ttl)` | Store a value. `ttl <= 0` means no expiration. |
| `SetNX(key, value, ttl)` | Store only if the key is absent or expired. Returns `false` if a live key exists. |
| `Get(key)` | Return `(value, true)` if present and not expired, otherwise `(zero, false)`. |
| `TTL(key)` | Remaining lifetime. `-1` for a key with no expiration; `ok == false` if absent. |
| `Delete(key)` | Remove a key. No-op if absent. |
| `Len()` | Number of stored keys (may include not-yet-swept expired entries). |
| `Clear()` | Remove all keys. |
| `Close()` | Stop the background sweeper. Safe to call multiple times and concurrently. |

## Expiration

A key expires when its TTL elapses. Removal happens two ways:

- **Lazily** — an expired entry found during `Get`/`TTL` is dropped on the spot, so a stale value is never returned even with the sweeper disabled.
- **In the background** — if `New` was given a positive interval, a sweeper periodically deletes expired entries so memory doesn't grow with abandoned keys.

With cleanup disabled (`New[V](0)`), expiration still works — expired keys are simply removed on access instead of proactively.

## SetNX as a lock

`SetNX` ("set if not exists") writes only when no live key is present, which makes it a simple primitive for one-shot guards — e.g. ensuring a single owner among concurrent goroutines:

```go
if c.SetNX("job:42", "running", 30*time.Second) {
	// won the slot — do the work
}
```

Under concurrent contention exactly one caller succeeds; the rest get `false`.

## Thread safety

All operations are safe for concurrent use. Reads share an `RWMutex`; writes and expiry take the write lock. The test suite exercises this with the race detector — concurrent distinct keys, contended shared keys, concurrent `SetNX`, and concurrent `Close`.

## Testing

```bash
go vet ./...
go test -race ./...
go test -bench=. -benchmem ./...
```

## License

MIT — see [LICENSE](LICENSE).
