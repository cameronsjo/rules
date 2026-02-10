---
globs: ["**/*.go"]
alwaysApply: false
---

# Go Standards

**Runtime:** Go 1.22+ | **Tooling:** golangci-lint, go test, pprof

## Patterns

```go
// Explicit error handling with wrapping
if err != nil {
    return fmt.Errorf("failed to fetch user %s: %w", userID, err)
}

// Table-driven tests
func TestParse(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    int
        wantErr bool
    }{
        {"valid", "42", 42, false},
        {"invalid", "abc", 0, true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Parse(tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("Parse() error = %v, wantErr %v", err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("Parse() = %v, want %v", got, tt.want)
            }
        })
    }
}

// Context for cancellation
func FetchUser(ctx context.Context, id string) (*User, error) {
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }
    // ... fetch logic
}
```

## Principles

- Clear is better than clever
- Composition over inheritance (interfaces)
- Prefer standard library over dependencies
- Benchmark before optimizing (`go test -bench`)
