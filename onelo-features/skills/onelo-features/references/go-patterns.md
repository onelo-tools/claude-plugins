# Go Patterns

Detection rules, grep patterns, insertion points, and snippet templates for
Go backend projects (net/http, Gin, Echo, Fiber, Chi, gRPC).

## Contents
- Go-specific classification rules
- Grep patterns
- Symbol extraction
- Insertion point
- Snippets
- Trigger detection
- Codegen for `FeatureRegistry`
- Project structure assumptions
- Testing patterns
- Common pitfalls / anti-patterns

## Go-specific classification rules

Generic atom-suffix and atom-path filtering happens in SKILL.md Phase 2.5.
Backend Go is *all* server-side, so the "atom vs. screen" split looks
different from frontend frameworks: there are no buttons, cells, or chips
to drop. The unit of instrumentation is a **route handler** (HTTP entry
point) or a **gRPC method**. Goroutines, helper functions, and DB layer
methods are plumbing — never instrument them.

Treat each of these as a `screen` (always keep — Phase 2.5 Rule 1 applies):

- **net/http handler.** A function with the signature
  `func(http.ResponseWriter, *http.Request)`, registered via
  `http.HandleFunc(...)`, `mux.HandleFunc(...)`, `mux.Handle(...)`, or
  `http.Handle(...)`. The same shape applies to any third-party router
  built on `http.Handler` (gorilla/mux, httprouter).
- **Gin handler.** A function with signature `func(*gin.Context)`,
  registered via `r.GET(...)`, `r.POST(...)`, `group.GET(...)` etc.
- **Echo handler.** A function with signature
  `func(echo.Context) error`, registered via `e.GET(...)`, `e.POST(...)`,
  `g.GET(...)`.
- **Fiber handler.** A function with signature `func(*fiber.Ctx) error`,
  registered via `app.Get(...)`, `app.Post(...)`, `group.Get(...)`.
- **Chi handler.** Same shape as net/http (`func(http.ResponseWriter,
  *http.Request)`), registered via `r.Get(...)`, `r.Post(...)`,
  `r.Route(...)`.
- **gRPC method.** A method on a server struct that satisfies a
  generated `pb.XxxServer` interface — first arg `context.Context`,
  second arg the request message, returning the response message and
  `error`.

### Go-only atoms to drop

On top of the generic Phase 2.5 list, drop these even when they live in
`handlers/` / `routes/`:

- **Helper functions** (`func parseUser(...)`, `func writeJSON(...)`)
  — no `*http.Request` / `*gin.Context` / `*fiber.Ctx` / `echo.Context`
  parameter, or unexported and called only by other handlers.
- **Struct types** — request/response payload definitions, never an
  entry point. Skip anything that's just `type FooRequest struct { ... }`.
- **Constructors / wiring** (`func NewServer(...)`, `func NewRouter(...)`,
  `func RegisterRoutes(...)`) — bootstrap code, gates the entire app.
- **Middleware** (`func AuthMiddleware(next http.Handler) http.Handler`,
  `func Logger() gin.HandlerFunc`) — apply to every request;
  instrumenting them would gate the whole app.
- **Background workers** (functions launched in goroutines from `main`
  or via `errgroup.Group`) — async workers, not user-visible features.
  (See "Trigger detection" below for when these matter as *callers*.)
- **`init()` and `main()`** — bootstrap, never a feature gate.
- **Database / repository methods** (`func (r *UserRepo) Find(...)`) —
  storage plumbing.

## Grep patterns

Run each pattern. Collect file + line number + handler signature.

```bash
# net/http handlers (top-level func with the canonical signature)
grep -rnE '^func [A-Za-z_][A-Za-z0-9_]*\(\s*[a-z_]+\s+http\.ResponseWriter\s*,\s*[a-z_]+\s+\*http\.Request\s*\)' \
  --include='*.go' \
  --exclude-dir=vendor --exclude-dir=node_modules \
  --exclude-dir=testdata --exclude-dir=.git \
  . 2>/dev/null

# net/http method-based handlers (e.g. func (s *Server) Foo(w, r))
grep -rnE '^func \([a-z_]+\s+\*?[A-Z][A-Za-z0-9_]*\)\s+[A-Z][A-Za-z0-9_]*\(\s*[a-z_]+\s+http\.ResponseWriter\s*,\s*[a-z_]+\s+\*http\.Request\s*\)' \
  --include='*.go' \
  --exclude-dir=vendor --exclude-dir=testdata \
  . 2>/dev/null

# Gin handlers (func(*gin.Context) — both standalone and method receivers)
grep -rnE 'func\s+(\([^)]+\)\s+)?[A-Za-z_][A-Za-z0-9_]*\(\s*[a-z_]+\s+\*gin\.Context\s*\)' \
  --include='*.go' \
  --exclude-dir=vendor --exclude-dir=testdata \
  . 2>/dev/null

# Echo handlers (func(echo.Context) error)
grep -rnE 'func\s+(\([^)]+\)\s+)?[A-Za-z_][A-Za-z0-9_]*\(\s*[a-z_]+\s+echo\.Context\s*\)\s+error' \
  --include='*.go' \
  --exclude-dir=vendor --exclude-dir=testdata \
  . 2>/dev/null

# Fiber handlers (func(*fiber.Ctx) error)
grep -rnE 'func\s+(\([^)]+\)\s+)?[A-Za-z_][A-Za-z0-9_]*\(\s*[a-z_]+\s+\*fiber\.Ctx\s*\)\s+error' \
  --include='*.go' \
  --exclude-dir=vendor --exclude-dir=testdata \
  . 2>/dev/null

# Route registration (helps map handler funcs to URL paths for naming)
grep -rnE '\.(Get|Post|Put|Patch|Delete|Head|Options|GET|POST|PUT|PATCH|DELETE|HEAD|OPTIONS|HandleFunc|Handle)\(\s*"' \
  --include='*.go' \
  --exclude-dir=vendor --exclude-dir=testdata \
  . 2>/dev/null

# gRPC server methods (method on *Server with (ctx context.Context, ...) signature)
grep -rnE '^func \([a-z_]+\s+\*?[A-Z][A-Za-z0-9_]*Server\)\s+[A-Z][A-Za-z0-9_]*\(\s*[a-z_]+\s+context\.Context\s*,' \
  --include='*.go' \
  --exclude-dir=vendor --exclude-dir=testdata \
  . 2>/dev/null
```

**Skip paths:** `vendor/`, `testdata/`, `.git/`, `*_test.go`,
`mock_*.go`, `*.pb.go`, `*_grpc.pb.go` (generated protobuf code),
`bin/`, `dist/`, `build/`.

The route-registration grep is for **name extraction** — pair each
`.Get("/path", handlerFunc)` line with the corresponding handler function
discovered above to derive a feature name from the URL path rather than
the function symbol.

## Symbol extraction

Derive a kebab-case feature name from each match:

| Source | Example | Feature name |
|---|---|---|
| Route path (preferred) | `r.GET("/api/chat/stream", chatStream)` | `chat-stream` |
| Route with version prefix | `r.POST("/v1/users/messages", ...)` | `users-messages` |
| Path params dropped | `r.GET("/users/:id/messages", ...)` | `users-messages` |
| net/http path params | `mux.HandleFunc("/users/{id}/messages", ...)` (Go 1.22+) | `users-messages` |
| Function name fallback | `func ListMessages(w http.ResponseWriter, r *http.Request)` | `list-messages` |
| Method receiver fallback | `func (s *Server) ChatStream(...)` | `chat-stream` |
| gRPC method | `func (s *ChatServer) StreamMessages(ctx, req)` | `stream-messages` |

### Rules

1. **Prefer the route path** when present — strip leading `/`, leading
   `api/`, leading version prefix (`v1/`, `v2/`), drop `:param` /
   `{param}` placeholders, join remaining segments with `-`, lowercase.
2. **Fall back to function/method name** when the registration line
   isn't co-located (handler defined in `handlers/foo.go`, registered
   in `routes/router.go` — pair them by symbol name).
3. **Strip suffixes** from method/function names: `Handler`, `Endpoint`,
   `Controller`, `Service` (matches the generic Phase 2.5 suffix list).
4. **Convert to kebab-case** — PascalCase → kebab-case
   (`ListMessages` → `list-messages`); camelCase → kebab-case.
5. **Truncate to 48 chars**, deduplicate with `-2`, `-3` if needed.

## Insertion point

Every insertion runs **after** auth/permission middleware has populated
the request context (so `Identify` has a real user ID) and **before**
business logic. Frame: "deny early on disabled, but only once we know
who you are."

### net/http

```go
func chatStream(w http.ResponseWriter, r *http.Request) {
    // ← INSERT HERE — auth middleware has already attached the user to the context.
    ctx := r.Context()
    if userID, ok := userIDFromContext(ctx); ok {
        client.Identify(ctx, userID)
    }
    if !client.Features.Feature(ctx, "chat-stream").IsEnabled() {
        http.NotFound(w, r)
        return
    }
    // existing business logic continues below
}
```

**Where to insert:** first line of the function body (after the
opening `{`). Pull the user ID from whatever pattern the project uses —
`r.Context().Value(authKey)`, a helper like `auth.UserFromRequest(r)`,
or a JWT claim. If the handler can be reached anonymously, skip the
`Identify` call — gate purely on the feature.

### Gin

```go
func chatStream(c *gin.Context) {
    // ← INSERT HERE — auth middleware has already populated c.Keys via c.Set("user", ...).
    ctx := c.Request.Context()
    if userID, ok := c.Get("userID"); ok {
        client.Identify(ctx, userID.(string))
    }
    if !client.Features.Feature(ctx, "chat-stream").IsEnabled() {
        c.AbortWithStatus(http.StatusNotFound)
        return
    }
    // existing logic
}
```

**Where to insert:** first line of the function body. Use
`c.Request.Context()` (not `context.Background()`) so deadlines and
cancellation propagate. Adapt the `c.Get("userID")` to whatever key
the project's auth middleware sets.

### Echo

```go
func chatStream(c echo.Context) error {
    // ← INSERT HERE — Echo middleware (e.g. echojwt) attaches claims to c.
    ctx := c.Request().Context()
    if user, ok := c.Get("user").(*jwt.Token); ok {
        claims := user.Claims.(jwt.MapClaims)
        client.Identify(ctx, claims["sub"].(string))
    }
    if !client.Features.Feature(ctx, "chat-stream").IsEnabled() {
        return echo.NewHTTPError(http.StatusNotFound)
    }
    // existing logic
}
```

**Where to insert:** first line of the function body. Always use
`c.Request().Context()` — not `context.Background()` — so the gating
read participates in the request lifecycle.

### Fiber

```go
func chatStream(c *fiber.Ctx) error {
    // ← INSERT HERE — Fiber middleware attaches the user via c.Locals(...).
    ctx := c.UserContext()
    if userID, ok := c.Locals("userID").(string); ok && userID != "" {
        client.Identify(ctx, userID)
    }
    if !client.Features.Feature(ctx, "chat-stream").IsEnabled() {
        return c.SendStatus(fiber.StatusNotFound)
    }
    // existing logic
}
```

**Where to insert:** first line of the function body. Use
`c.UserContext()` — Fiber's per-request context that propagates
deadlines from middleware.

### Chi

```go
func chatStream(w http.ResponseWriter, r *http.Request) {
    // ← INSERT HERE — Chi reuses net/http's request context.
    ctx := r.Context()
    if userID, ok := ctx.Value(authKey).(string); ok {
        client.Identify(ctx, userID)
    }
    if !client.Features.Feature(ctx, "chat-stream").IsEnabled() {
        http.NotFound(w, r)
        return
    }
    // existing logic
}
```

**Where to insert:** first line of the function body. Chi handlers
are net/http handlers — the same insertion rules apply.

### gRPC

```go
func (s *ChatServer) StreamMessages(ctx context.Context, req *pb.StreamRequest) (*pb.StreamResponse, error) {
    // ← INSERT HERE — gRPC interceptors have already attached auth metadata.
    if userID, ok := userIDFromContext(ctx); ok {
        s.onelo.Identify(ctx, userID)
    }
    if !s.onelo.Features.Feature(ctx, "stream-messages").IsEnabled() {
        return nil, status.Error(codes.Unimplemented, "feature not available")
    }
    // existing logic
}
```

**Where to insert:** first line of the method body. For server-streaming
methods (signature `(req, stream)`), use `stream.Context()` instead
of an explicit `ctx` parameter. The fail-closed response is
`codes.Unimplemented` (gRPC's analog to HTTP 404) — never `Unavailable`,
which clients retry.

### Package-scope `client` instance assumption

All snippets assume a `*onelo.Client` is reachable in scope. The typical
patterns:

- **Package-level singleton** — the most common Go idiom for small
  services: a `var client *onelo.Client` declared in the same package
  as `main()`, initialized in `main()` before serving.
- **Server struct field** — for non-trivial services with DI:
  `type Server struct { onelo *onelo.Client; ... }`, reached as
  `s.onelo.Features.Feature(...)`.
- **Context value** — when handlers can't see the server struct,
  inject the client via middleware that calls
  `r = r.WithContext(context.WithValue(r.Context(), oneloKey, client))`.
  Less common; prefer the struct-field approach.

If the import isn't already present, add it at the top of the file
alongside other project imports:

```go
import (
    "github.com/onelo-tools/onelo-go"
)
```

## Snippets

> **API note:** The Go SDK is **synchronous**. There is no goroutine or
> channel involved in the read path. The `Feature` value exposes
> `IsEnabled()`, `IsVisible()`, `IsGreyed()`, `IsNew()`, `IsBeta()`,
> `IsComingSoon()` as **methods** (with parentheses), unlike the Python
> SDK's properties. They return `bool` and never block on the network.
>
> Reads from the in-process cache (~1µs); a background goroutine keeps
> the cache fresh via SSE. If no snapshot has landed yet — or no user
> is identified — the SDK fails closed and returns
> `Feature{Status: "disabled"}`.

### Choosing the right check

Default to **`IsEnabled()`** for "execute this handler or 404" gates.
It matches the dashboard's primary on/off semantics and is what the
SDK's README leads with.

| Check | True when status is | Use for |
|---|---|---|
| `IsEnabled()` | `enabled`, `new`, `beta` | **Default.** Binary gate: serve the route or return 404. Paid endpoints, A/B rollouts. |
| `IsVisible()` | anything except `hidden` | When a "coming soon" / placeholder response is more useful than a hard 404. |
| `IsGreyed()`, `IsNew()`, `IsBeta()`, `IsComingSoon()` | the matching status | Branching the response body on a specific promotional state (e.g., return `{"upsell": true}` instead of data). |

### net/http handler

```go
if !client.Features.Feature(ctx, "$NAME").IsEnabled() {
    http.NotFound(w, r)
    return
}
```

JSON 404 variant:

```go
if !client.Features.Feature(ctx, "$NAME").IsEnabled() {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusNotFound)
    _ = json.NewEncoder(w).Encode(map[string]string{"error": "feature_disabled"})
    return
}
```

### Gin handler

```go
if !client.Features.Feature(ctx, "$NAME").IsEnabled() {
    c.AbortWithStatusJSON(http.StatusNotFound, gin.H{"error": "feature_disabled"})
    return
}
```

### Echo handler

```go
if !client.Features.Feature(ctx, "$NAME").IsEnabled() {
    return echo.NewHTTPError(http.StatusNotFound, "feature_disabled")
}
```

### Fiber handler

```go
if !client.Features.Feature(ctx, "$NAME").IsEnabled() {
    return c.Status(fiber.StatusNotFound).JSON(fiber.Map{"error": "feature_disabled"})
}
```

### Chi handler

Same as net/http — Chi handlers are `func(http.ResponseWriter, *http.Request)`:

```go
if !client.Features.Feature(ctx, "$NAME").IsEnabled() {
    http.NotFound(w, r)
    return
}
```

### gRPC method

```go
if !s.onelo.Features.Feature(ctx, "$NAME").IsEnabled() {
    return nil, status.Error(codes.Unimplemented, "feature not available")
}
```

For server-streaming RPCs:

```go
if !s.onelo.Features.Feature(stream.Context(), "$NAME").IsEnabled() {
    return status.Error(codes.Unimplemented, "feature not available")
}
```

### Dev-mode default

The Go SDK's `Config` does not yet expose a `FeatureDefaultStatus`
field — newly instrumented features always start as `disabled` until
toggled in the dashboard. Until parity with the JS/Swift
`featureDefaultStatus` ships, use the dashboard's "set status to
enabled on first appearance" workflow on the staging tenant for local
development.

### Rich destination states (opt-in)

For routes that should serve a placeholder instead of a hard 404:

```go
f := client.Features.Feature(ctx, "$NAME")
switch {
case f.IsComingSoon():
    return c.JSON(http.StatusOK, gin.H{"status": "coming_soon", "message": "We're building this!"})
case f.IsGreyed():
    return c.JSON(http.StatusPaymentRequired, gin.H{"error": "upgrade_required"})
case !f.IsEnabled():
    return c.AbortWithStatus(http.StatusNotFound)
}
// real handler logic
```

This is opt-in — write it once per route where rich states matter.

## Trigger detection

Backend Go rarely has "triggers" in the frontend sense (a button opens
a screen). Routes themselves *are* the destinations. The trigger
concept maps onto **internal callers** of those routes — background
goroutines, cron jobs (e.g. `robfig/cron`), worker pools, or other
backend services hitting an internal HTTP endpoint via `net/http`,
`resty`, or `grpc.Dial`.

In most projects, **trigger detection for Go is low-value** and should
be skipped: backend services don't render UI, so there's no "orphaned
trigger" failure mode the way there is in a frontend app. Enable
trigger detection only when the developer explicitly opts in — e.g., a
cron job that should only run for tenants who have a particular feature
enabled, where you want the job itself gated to match.

### Grep patterns (opt-in)

```bash
# Outbound HTTP calls (stdlib net/http, resty)
grep -rnE '(http\.(Get|Post|Head|PostForm|Do)|client\.(Get|Post|Do)|resty\.(New|R))\b' \
  --include='*.go' \
  --exclude-dir=vendor --exclude-dir=testdata \
  . 2>/dev/null

# robfig/cron schedule registrations
grep -rnE '\.AddFunc\(|cron\.New\(' \
  --include='*.go' \
  --exclude-dir=vendor --exclude-dir=testdata \
  . 2>/dev/null

# go-co-op/gocron registrations
grep -rnE 'gocron\.NewScheduler|scheduler\.Every\(' \
  --include='*.go' \
  --exclude-dir=vendor --exclude-dir=testdata \
  . 2>/dev/null

# Goroutine launches from main / wiring (caller code)
grep -rnE '^\s*go\s+[a-zA-Z_]' \
  --include='*.go' \
  --exclude-dir=vendor --exclude-dir=testdata \
  . 2>/dev/null
```

### Trigger snippet

```go
f := client.Features.Feature(ctx, "$NAME")
if !f.IsVisible() {
    return // or `continue` in a loop, or skip the cron tick
}
// existing trigger logic (HTTP call, job enqueue, etc.)
```

`IsVisible()` (rather than `IsEnabled()`) keeps the trigger active for
non-`hidden` statuses — `greyed`, `coming_soon`, `beta`, etc. — so an
admin's intended UX renders even when the feature isn't fully on.

## Codegen for `FeatureRegistry`

### Generated file

Path: `internal/oneloreg/registry.go`. Putting it under `internal/`
prevents accidental import from outside the module; the `oneloreg`
package name is short and unambiguous.

```go
// Code generated by tool/gen-feature-registry. DO NOT EDIT.
package oneloreg

// FeatureRegistry is the canonical list of every feature name
// instrumented in this codebase. Pass it to Features.Declare on init.
var FeatureRegistry = []string{
    "chat-stream",
    "voice-stream",
    "game-think",
    // ...
}
```

### Generator script

Pure Go, no external dependencies — runs anywhere `go` is available.
Drop into `tool/gen-feature-registry/main.go`.

```go
// Scan Go sources for Features.Feature(ctx, "name") calls and emit
// the FeatureRegistry constant.
package main

import (
    "flag"
    "fmt"
    "os"
    "path/filepath"
    "regexp"
    "sort"
    "strings"
)

// Match Features.Feature(ctx, "name") and Features.Feature(ctx, 'name'),
// atom syntax only. The (?:...) keeps the leading arg flexible.
var pattern = regexp.MustCompile(`Features\.Feature\(\s*[A-Za-z_][A-Za-z0-9_.()]*\s*,\s*"([a-z][a-z0-9-]*)"`)

var skipDirs = map[string]bool{
    "vendor": true, "testdata": true, ".git": true,
    "node_modules": true, "bin": true, "dist": true, "build": true,
}

func discover(root string) (map[string]struct{}, error) {
    names := make(map[string]struct{})
    err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        if info.IsDir() {
            if skipDirs[info.Name()] {
                return filepath.SkipDir
            }
            return nil
        }
        if !strings.HasSuffix(path, ".go") || strings.HasSuffix(path, "_test.go") {
            return nil
        }
        if strings.HasSuffix(path, ".pb.go") || strings.HasSuffix(path, "_grpc.pb.go") {
            return nil
        }
        data, err := os.ReadFile(path)
        if err != nil {
            return nil // skip unreadable files
        }
        for _, m := range pattern.FindAllStringSubmatch(string(data), -1) {
            names[m[1]] = struct{}{}
        }
        return nil
    })
    return names, err
}

func emit(names map[string]struct{}, outFile string) error {
    if err := os.MkdirAll(filepath.Dir(outFile), 0o755); err != nil {
        return err
    }
    sorted := make([]string, 0, len(names))
    for n := range names {
        sorted = append(sorted, n)
    }
    sort.Strings(sorted)

    var b strings.Builder
    b.WriteString("// Code generated by tool/gen-feature-registry. DO NOT EDIT.\n")
    b.WriteString("package oneloreg\n\n")
    b.WriteString("// FeatureRegistry is the canonical list of every feature name\n")
    b.WriteString("// instrumented in this codebase. Pass it to Features.Declare on init.\n")
    b.WriteString("var FeatureRegistry = []string{\n")
    for _, n := range sorted {
        fmt.Fprintf(&b, "\t%q,\n", n)
    }
    b.WriteString("}\n")
    return os.WriteFile(outFile, []byte(b.String()), 0o644)
}

func main() {
    sources := flag.String("src", ".", "sources directory to scan")
    out := flag.String("out", "internal/oneloreg/registry.go", "output file")
    flag.Parse()

    names, err := discover(*sources)
    if err != nil {
        fmt.Fprintf(os.Stderr, "scan failed: %v\n", err)
        os.Exit(1)
    }
    if err := emit(names, *out); err != nil {
        fmt.Fprintf(os.Stderr, "emit failed: %v\n", err)
        os.Exit(1)
    }
    fmt.Printf("Generated %s with %d features\n", *out, len(names))
}
```

Run manually with `go run ./tool/gen-feature-registry` (or with
explicit args: `go run ./tool/gen-feature-registry -src=./internal -out=./internal/oneloreg/registry.go`).

### Build hook integration

Go's idiomatic codegen hook is **`go generate`**. Drop a
`//go:generate` directive at the top of any package file (commonly
`doc.go` or the file that imports the registry):

```go
//go:generate go run ./tool/gen-feature-registry -src=. -out=./internal/oneloreg/registry.go
package main
```

Then `go generate ./...` (run as part of `make generate`, CI, or a
pre-commit hook) refreshes the registry.

**`Makefile` (most portable):**

```makefile
.PHONY: generate test build run

generate:
	go generate ./...

test: generate
	go test ./...

build: generate
	go build -o bin/server ./cmd/server

run: generate
	go run ./cmd/server
```

**`pre-commit` hook (catches drift in PRs):** add to
`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: local
    hooks:
      - id: generate-feature-registry
        name: Regenerate Onelo feature registry
        entry: go generate ./...
        language: system
        pass_filenames: false
        files: \.go$
```

**GitHub Actions (verify registry is in sync):**

```yaml
- name: Verify feature registry
  run: |
    go generate ./...
    git diff --exit-code internal/oneloreg/registry.go
```

Wire it once, forget about it — every commit refreshes the registry.

## Project structure assumptions

Where the `*onelo.Client` typically lives, by framework:

| Framework | Init location | How handlers reach it |
|---|---|---|
| net/http | `cmd/server/main.go` alongside `http.ListenAndServe(...)`. Defer `client.Close(ctx)`. | Package-level `var client *onelo.Client` in `main` package, or struct field on `Server`. |
| Gin | `cmd/server/main.go` alongside `r := gin.Default()`. Defer `client.Close(ctx)`. | Struct field on the handler container (`type API struct { onelo *onelo.Client }`). |
| Echo | `cmd/server/main.go` alongside `e := echo.New()`. Defer `client.Close(ctx)`. | Struct field on the handler container. |
| Fiber | `cmd/server/main.go` alongside `app := fiber.New()`. Register `app.Hooks().OnShutdown(...)` to call `client.Close(ctx)`. | Struct field on the handler container. |
| Chi | `cmd/server/main.go` alongside `r := chi.NewRouter()`. Defer `client.Close(ctx)`. | Struct field on the handler container. |
| gRPC | `cmd/server/main.go` alongside `grpc.NewServer(...)`. Defer `client.Close(ctx)`. | Field on the server struct that satisfies the generated `pb.XxxServer` interface. |

### net/http full init

```go
package main

import (
    "context"
    "log"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/onelo-tools/onelo-go"
    "github.com/myorg/myapp/internal/oneloreg"
)

var client *onelo.Client

func main() {
    var err error
    client, err = onelo.New(onelo.Config{
        PublishableKey: os.Getenv("ONELO_PUBLISHABLE_KEY"),
        APIURL:         envOr("ONELO_API_URL", "https://app.onelo.tools"),
        Logger:         slog.Default(),
    })
    if err != nil {
        log.Fatalf("onelo init: %v", err)
    }
    defer client.Close(context.Background())

    // Pre-register every instrumented feature.
    if err := client.Features.Declare(context.Background(), oneloreg.FeatureRegistry); err != nil {
        log.Printf("declare features: %v", err)
    }

    // Block briefly for the first SSE snapshot so initial requests don't fail closed.
    readyCtx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    _ = client.Ready(readyCtx)
    cancel()

    mux := http.NewServeMux()
    mux.HandleFunc("/chat/stream", chatStream)

    srv := &http.Server{Addr: ":8080", Handler: mux}
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("listen: %v", err)
        }
    }()

    // Graceful shutdown — Close drains the SSE goroutine before exit.
    sigs := make(chan os.Signal, 1)
    signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
    <-sigs
    shutdownCtx, cancel2 := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel2()
    _ = srv.Shutdown(shutdownCtx)
}

func envOr(k, def string) string {
    if v := os.Getenv(k); v != "" {
        return v
    }
    return def
}
```

### Gin example

```go
package main

import (
    "context"
    "log"
    "os"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/onelo-tools/onelo-go"
    "github.com/myorg/myapp/internal/oneloreg"
)

type API struct {
    onelo *onelo.Client
}

func main() {
    client, err := onelo.New(onelo.Config{
        PublishableKey: os.Getenv("ONELO_PUBLISHABLE_KEY"),
        APIURL:         os.Getenv("ONELO_API_URL"),
    })
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close(context.Background())

    _ = client.Features.Declare(context.Background(), oneloreg.FeatureRegistry)
    readyCtx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    _ = client.Ready(readyCtx)
    cancel()

    api := &API{onelo: client}
    r := gin.Default()
    r.POST("/chat/stream", api.chatStream)
    _ = r.Run(":8080")
}
```

### Echo / Fiber / Chi

The same shape applies — construct the client in `main`, store it on
the handler struct, defer `client.Close(ctx)`. For Fiber, prefer
`app.Hooks().OnShutdown(...)` over `defer` since Fiber's
`app.Listen(...)` blocks but `app.Shutdown()` is the canonical exit
point.

### gRPC example

```go
package main

import (
    "context"
    "log"
    "net"
    "os"

    "google.golang.org/grpc"
    "github.com/onelo-tools/onelo-go"
    pb "github.com/myorg/myapp/proto"
    "github.com/myorg/myapp/internal/oneloreg"
)

type ChatServer struct {
    pb.UnimplementedChatServer
    onelo *onelo.Client
}

func main() {
    client, err := onelo.New(onelo.Config{
        PublishableKey: os.Getenv("ONELO_PUBLISHABLE_KEY"),
        APIURL:         os.Getenv("ONELO_API_URL"),
    })
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close(context.Background())

    _ = client.Features.Declare(context.Background(), oneloreg.FeatureRegistry)

    lis, err := net.Listen("tcp", ":9090")
    if err != nil {
        log.Fatal(err)
    }
    s := grpc.NewServer()
    pb.RegisterChatServer(s, &ChatServer{onelo: client})
    if err := s.Serve(lis); err != nil {
        log.Fatal(err)
    }
}
```

## Testing patterns

The Go SDK is goroutine-safe and ships zero runtime deps, so tests can
either skip the client entirely (gate-mode helpers) or point it at an
`httptest.Server` stub.

### Stubbed API URL

```go
func TestChatStreamHandler(t *testing.T) {
    stub := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Return a snapshot with chat-stream enabled.
        w.Header().Set("Content-Type", "application/json")
        _, _ = w.Write([]byte(`{"features":{"chat-stream":{"status":"enabled"}}}`))
    }))
    defer stub.Close()

    c, err := onelo.New(onelo.Config{
        PublishableKey: "pk_test_dummy",
        APIURL:         stub.URL,
        Strategy:       onelo.StrategyPolling,
        PollInterval:   100 * time.Millisecond,
    })
    if err != nil {
        t.Fatal(err)
    }
    defer c.Close(context.Background())

    readyCtx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()
    if err := c.Ready(readyCtx); err != nil {
        t.Fatal(err)
    }

    // ... call handler, assert it serves rather than 404s.
}
```

### Bypass the client for unit tests

For pure handler unit tests where the client isn't relevant, inject a
nil-tolerant gate function instead of the live client:

```go
type gateFunc func(ctx context.Context, name string) bool

func chatStream(gate gateFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if !gate(r.Context(), "chat-stream") {
            http.NotFound(w, r)
            return
        }
        // ...
    }
}
```

In production, `gate` is `func(ctx context.Context, name string) bool {
return client.Features.Feature(ctx, name).IsEnabled() }`. In tests,
pass a stub closure that returns `true` / `false` deterministically.

## Common pitfalls / anti-patterns

- **Don't construct a new `*onelo.Client` per request.** The client
  owns a long-lived background goroutine and SSE connection. Build it
  once in `main()` and reuse. Per-request construction leaks goroutines
  and exhausts the API quota in seconds.
- **Don't forget `defer client.Close(ctx)`.** Without it, the
  background goroutine keeps the SSE connection open past process exit
  and the SDK can't flush in-flight `Identify` calls. In `main()`,
  always pair `onelo.New(...)` with `defer client.Close(...)`.
- **Don't pass `context.Background()` to `Feature(...)`.** Pass the
  request-scoped context (`r.Context()`, `c.Request.Context()`,
  `c.UserContext()`, `stream.Context()`) so cancellations and deadlines
  propagate. The SDK won't make a network call, but downstream code
  reading from the returned `Feature` may.
- **Don't gate inside middleware.** Middleware applies to every
  request; instrumenting it gates the whole app. Gate in the handler
  body, after auth has resolved the user.
- **Don't ignore the `Identify` call return.** It's fire-and-forget,
  but on a misconfigured client (missing user, closed client) the SDK
  logs a warning. Surface that during initial integration so missing
  identification doesn't silently cause fail-closed reads.
- **Don't call `Feature(...)` before `Ready(ctx)` returns.** Reads
  before the first snapshot fail closed (`disabled`). Either block on
  `Ready` in `main()` (recommended) or accept that early requests
  serve 404 until the snapshot lands.
- **Don't shadow `ctx` accidentally.** A common Go mistake: writing
  `ctx := r.Context()` inside an `if` block scopes `ctx` only to that
  block. Declare it once at the top of the handler and reuse.
