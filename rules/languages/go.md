---
paths:
  - "**/*.go"
  - "**/go.mod"
  - "**/go.sum"
---

# Go Standards

- **Version**: Go 1.24+ (iterators stable, Swiss Tables maps, weak pointers)
- **HTTP**: stdlib net/http (1.22+ routing) or chi
- **Database**: sqlc (type-safe SQL) or pgx
- **Validation**: go-playground/validator
- **CLI**: cobra + viper
- **Testing**: stdlib + testify (testing.B.Loop for benchmarks in 1.24)
- **Linting**: golangci-lint (100+ linters, caching, parallel)
- **Observability**: OpenTelemetry + slog (stdlib, slog.DiscardHandler in 1.24)

## Core Philosophy

Go remains simple. Use stdlib over frameworks. Generics for data structures, not business logic.

## Core Requirements

- **MUST** use `gofmt` / `goimports` for formatting
- **MUST** use `golangci-lint` with strict config
- **MUST** handle all errors explicitly - no ignoring with `_`
- **MUST** use context for cancellation and timeouts (first arg to I/O functions)
- **MUST** use `errors.Is`/`errors.As` for error checking (not equality)
- **MUST** handle errors in defer statements (`defer file.Close()` can lose data on write flush failure)
- **MUST NOT** use `panic` for error handling - only for startup failures
- **MUST NOT** use package-level global variables for DB/loggers - makes testing impossible
- **SHOULD** use slog for structured logging (Go 1.21+) - not Logrus/Zap
- **SHOULD** prefer table-driven tests
- **SHOULD** accept interfaces, return structs - decouples code, keeps usage simple
- **SHOULD** use sqlc or pgx for database (explicit SQL) - avoid heavy ORMs like GORM
- **SHOULD** pass dependencies manually in `NewService()` constructors - avoid magic DI frameworks
- **SHOULD** use generics for utilities only - if >5 type params, you're overdoing it

## Project Structure

```
myproject/
├── cmd/
│   └── myapp/
│       └── main.go
├── internal/           # Private packages
│   ├── handler/
│   ├── service/
│   └── repository/
├── pkg/                # Public packages (if any)
├── go.mod
└── go.sum
```

## Error Handling

```go
// Custom errors with context
type NotFoundError struct {
    Resource string
    ID       string
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s not found: %s", e.Resource, e.ID)
}

// Wrapping errors with context
func GetUser(ctx context.Context, id string) (*User, error) {
    user, err := db.FindUser(ctx, id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, &NotFoundError{Resource: "user", ID: id}
        }
        return nil, fmt.Errorf("failed to get user %s: %w", id, err)
    }
    return user, nil
}

// Checking error types
var notFound *NotFoundError
if errors.As(err, &notFound) {
    // Handle not found
}
```

## Modern Patterns

```go
// Range-over-func iterators (1.23+)
// Iterator types: func(func() bool), func(func(V) bool), func(func(K, V) bool)
func FilterSlice[T any](s []T, pred func(T) bool) iter.Seq[T] {
    return func(yield func(T) bool) {
        for _, v := range s {
            if pred(v) && !yield(v) {
                return
            }
        }
    }
}

// Using iterators with slices package (1.23+)
for v := range slices.Values(mySlice) { }
collected := slices.Collect(myIterator)

// String iterators (1.24+)
for part := range strings.SplitSeq(s, ",") { }
for field := range strings.FieldsSeq(s) { }

// Structured logging with slog (1.21+)
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("user created",
    slog.String("user_id", user.ID),
    slog.String("email", user.Email),
)

// HTTP routing (1.22+)
mux := http.NewServeMux()
mux.HandleFunc("GET /users/{id}", getUser)
mux.HandleFunc("POST /users", createUser)

// Context with timeout
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
```

## HTTP Server Pattern

```go
type Server struct {
    db     *sql.DB
    logger *slog.Logger
}

func (s *Server) handleGetUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")  // Go 1.22+

    user, err := s.db.GetUser(r.Context(), id)
    if err != nil {
        var notFound *NotFoundError
        if errors.As(err, &notFound) {
            http.Error(w, err.Error(), http.StatusNotFound)
            return
        }
        s.logger.Error("failed to get user", slog.Any("error", err))
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    server := &Server{
        db:     connectDB(),
        logger: logger,
    }

    mux := http.NewServeMux()
    mux.HandleFunc("GET /users/{id}", server.handleGetUser)

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 10 * time.Second,
    }

    logger.Info("starting server", slog.String("addr", srv.Addr))
    if err := srv.ListenAndServe(); err != nil {
        logger.Error("server error", slog.Any("error", err))
    }
}
```

## Anti-patterns

```go
// ❌ Bad
if err != nil {
    panic(err)                    // Don't panic for errors
}
_ = json.Unmarshal(data, &v)     // Ignoring errors
go doWork()                       // Goroutine without lifecycle

// ✅ Good
if err != nil {
    return fmt.Errorf("failed: %w", err)
}
if err := json.Unmarshal(data, &v); err != nil {
    return err
}
g, ctx := errgroup.WithContext(ctx)
g.Go(func() error { return doWork(ctx) })
```

## golangci-lint Config

```yaml
# .golangci.yml
linters:
  enable:
    - errcheck
    - govet
    - staticcheck
    - unused
    - gosimple
    - ineffassign
    - misspell
    - gofmt
    - goimports

linters-settings:
  errcheck:
    check-blank: true
```
