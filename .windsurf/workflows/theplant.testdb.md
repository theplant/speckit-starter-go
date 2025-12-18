---
description: Setup and manage test databases using testcontainers for Go integration tests.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Setup and manage test databases using testcontainers for Go integration tests.

## Test Database Management

### Prerequisites

- Docker must be available (local/CI environments)
- Install testcontainers-go:
```bash
go get github.com/testcontainers/testcontainers-go
go get github.com/testcontainers/testcontainers-go/modules/postgres
```

### Setup Test Database

```go
import (
    "context"
    "testing"
    
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

func setupTestDB(t *testing.T) (*gorm.DB, func()) {
    ctx := context.Background()
    
    // Create PostgreSQL container
    pgContainer, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:15-alpine"),
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("postgres"),
        postgres.WithPassword("postgres"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2),
        ),
    )
    if err != nil {
        t.Fatalf("Failed to start PostgreSQL container: %v", err)
    }
    
    // Cleanup function
    cleanup := func() {
        if err := pgContainer.Terminate(ctx); err != nil {
            t.Logf("Failed to terminate container: %v", err)
        }
    }
    
    // Get connection string
    connStr, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
    if err != nil {
        cleanup()
        t.Fatalf("Failed to get connection string: %v", err)
    }
    
    // Connect using GORM
    db, err := gorm.Open(postgres.Open(connStr), &gorm.Config{})
    if err != nil {
        cleanup()
        t.Fatalf("Failed to connect to database: %v", err)
    }
    
    // Run GORM AutoMigrate with internal models
    if err := db.AutoMigrate(&Product{} /* add other models */); err != nil {
        cleanup()
        t.Fatalf("Failed to run migrations: %v", err)
    }
    
    return db, cleanup
}
```

### Test Isolation with Truncation

Use database truncation for test isolation:

```go
// Create helper function for truncation (reusable across all tests)
func truncateTables(db *gorm.DB, tables ...string) {
    // Truncate in reverse order (children before parents)
    for i := len(tables) - 1; i >= 0; i-- {
        db.Exec(fmt.Sprintf("TRUNCATE TABLE %s CASCADE", tables[i]))
    }
}

// Usage in tests
func TestProductCreate(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products", "categories")  // Children before parents
    
    // Test implementation...
}
```

### Benefits of Truncation Strategy

- Tests real production behavior with commits
- Works with any code structure
- Simple and fast (~1-5ms overhead)
- Handles foreign key constraints with CASCADE

### AI Agent Requirements

- Use `testcontainers.PostgresContainer` for automatic PostgreSQL lifecycle
- Setup schema with GORM `AutoMigrate` (matches production)
- Cleanup with `defer container.Terminate(ctx)` pattern
- Truncate tables after each test using `defer` pattern
- Use `TRUNCATE ... CASCADE` to handle foreign key constraints
- Truncate in reverse dependency order (children before parents)
