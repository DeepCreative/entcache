# Entcache in the Bravo Zero Ecosystem

> **DeepCreative Fork:** `github.com/DeepCreative/entcache`
>
> This is DeepCreative's fork of [ariga.io/entcache](https://github.com/ariga/entcache), providing query caching for the [Ent ORM](https://entgo.io) framework.

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Cache Architecture](#cache-architecture)
3. [Integration with Bravo Zero Services](#integration-with-bravo-zero-services)
4. [Caching Strategies](#caching-strategies)
5. [Repository Structure](#repository-structure)
6. [Onboarding Guide](#onboarding-guide)
7. [Configuration Examples](#configuration-examples)
8. [Performance Considerations](#performance-considerations)
9. [Troubleshooting](#troubleshooting)

---

## Overview

**entcache** is a query caching layer for the Ent ORM that intercepts database queries and caches their results. In the Bravo Zero ecosystem, it provides:

- **Reduced Database Load** - Cache frequently-accessed data
- **Faster Response Times** - Serve cached results without database roundtrips
- **Request-Level Deduplication** - Eliminate duplicate queries within a single request
- **Multi-Level Caching** - LRU memory cache + Redis for distributed caching

### Why Caching Matters for Bravo Zero

GraphQL APIs often suffer from the **N+1 query problem**. Even with Ent's field collection optimizations, complex nested queries can generate redundant database calls. Entcache addresses this by:

1. Caching query results at the driver level (transparent to application code)
2. Serving identical queries from cache within a request context
3. Optionally persisting cache entries across requests (with TTL)

---

## Cache Architecture

### High-Level Design

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GraphQL Request                               │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Ent Client                                   │
│    client.User.Query().Where(user.IDEQ(123)).Only(ctx)              │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      entcache.Driver                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  1. Generate cache key from query + parameters              │   │
│  │  2. Check cache levels (Context → LRU → Redis)              │   │
│  │  3. On HIT: Return cached rows                              │   │
│  │  4. On MISS: Execute query, cache result, return            │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                          (Cache Miss)
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        PostgreSQL                                    │
│               SELECT * FROM users WHERE id = 123                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Cache Key Generation

Entcache generates a hash from:
- SQL statement (query string)
- Query parameters (bound values)

```
Query:  SELECT * FROM users WHERE id = $1 AND org_id = $2
Params: [123, "org_abc"]
Key:    sha256("SELECT...123...org_abc") → "a1b2c3d4..."
```

---

## Integration with Bravo Zero Services

### Services Using entcache

| Service | Repository | Uses entcache | Cache Strategy |
|---------|------------|---------------|----------------|
| Main Backend | `www/server` | ✅ | Context-Level + LRU |
| Dreamscape | `dreamscape-go` | ❌ (planned) | - |

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         BRAVO ZERO ECOSYSTEM                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        www/server                                │   │
│  │                       (Main Backend)                             │   │
│  │                                                                  │   │
│  │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │   │
│  │   │   GraphQL   │───▶│  Ent Client │───▶│ entcache    │        │   │
│  │   │   Request   │    │             │    │   Driver    │        │   │
│  │   └─────────────┘    └─────────────┘    └──────┬──────┘        │   │
│  │                                                 │               │   │
│  │                    ┌────────────────────────────┼────────────┐ │   │
│  │                    │         Cache Hierarchy    │            │ │   │
│  │                    │                            ▼            │ │   │
│  │                    │  ┌──────────────────────────────────┐  │ │   │
│  │                    │  │  Level 1: Context Cache          │  │ │   │
│  │                    │  │  (Per-request, eliminates N+1)   │  │ │   │
│  │                    │  └──────────────────────────────────┘  │ │   │
│  │                    │                    │ miss              │ │   │
│  │                    │                    ▼                   │ │   │
│  │                    │  ┌──────────────────────────────────┐  │ │   │
│  │                    │  │  Level 2: LRU Memory Cache       │  │ │   │
│  │                    │  │  (Process-level, fast access)    │  │ │   │
│  │                    │  └──────────────────────────────────┘  │ │   │
│  │                    │                    │ miss              │ │   │
│  │                    │                    ▼                   │ │   │
│  │                    │  ┌──────────────────────────────────┐  │ │   │
│  │                    │  │  Level 3: Redis (Optional)       │  │ │   │
│  │                    │  │  (Distributed, shared across     │  │ │   │
│  │                    │  │   multiple server instances)     │  │ │   │
│  │                    │  └──────────────────────────────────┘  │ │   │
│  │                    └────────────────────────────────────────┘ │   │
│  │                                         │ miss               │   │
│  │                                         ▼                    │   │
│  │                              ┌──────────────────┐           │   │
│  │                              │   PostgreSQL     │           │   │
│  │                              └──────────────────┘           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Go Module Configuration

```go
// www/server/go.mod
require (
    ariga.io/entcache v0.1.0
)

replace ariga.io/entcache => github.com/DeepCreative/entcache v1.0.0
```

---

## Caching Strategies

### 1. Context-Level Cache (Recommended for GraphQL)

**Best for:** Eliminating duplicate queries within a single HTTP request

The context cache attaches a per-request LRU cache to the request's `context.Context`. This is ideal for GraphQL because:
- Complex queries often fetch the same entities multiple times
- No stale data concerns (cache dies with the request)
- Zero configuration beyond wrapping the context

**How it works:**

```
GraphQL Query:
{
  users(ids: [1, 2, 3]) {
    id
    name
    createdBy {      # ← Might be user 1 again!
      id
      name
    }
  }
}

Without Context Cache:
  Query 1: SELECT * FROM users WHERE id IN (1, 2, 3)
  Query 2: SELECT * FROM users WHERE id = 1  # Duplicate!

With Context Cache:
  Query 1: SELECT * FROM users WHERE id IN (1, 2, 3)
  Query 2: [CACHE HIT] - User 1 already in context cache
```

### 2. Driver-Level Cache (Process Cache)

**Best for:** Frequently accessed, rarely-changing data

Caches query results at the Ent driver level, shared across all requests in the same process.

**Considerations:**
- Requires TTL configuration to prevent stale data
- Good for reference data (categories, settings, etc.)
- Use with caution for frequently-mutated data

### 3. Multi-Level Cache (Redis + LRU)

**Best for:** High-traffic production deployments

Combines in-memory LRU with distributed Redis cache:
1. First check: Local LRU (fastest)
2. Second check: Redis (shared across instances)
3. Final fallback: Database query

---

## Repository Structure

```
entcache/
├── driver.go              # Core cache driver implementation
├── driver_test.go         # Driver tests with mocks
├── context.go             # Context-level cache utilities
├── level.go               # Cache level abstractions (LRU, Redis)
├── go.mod                 # Module dependencies
├── go.sum
├── README.md              # Original documentation
├── BRAVO_ZERO.md          # This file
├── LICENSE
└── internal/
    └── examples/
        ├── ctxlevel/      # Context-level cache example
        │   ├── main.go
        │   └── main_test.go
        ├── multilevel/    # Multi-level cache example
        │   ├── main.go
        │   └── README.md
        └── todo/          # Full todo app with caching
            ├── ent/       # Generated Ent code
            └── *.go       # GraphQL server with cache
```

---

## Onboarding Guide

### For New Engineers / AI Agents

#### Prerequisites

1. **Go 1.19+** installed
2. Understanding of:
   - Ent ORM basics (see [ent-contrib docs](../ent-contrib/BRAVO_ZERO.md))
   - Caching concepts (TTL, LRU, cache invalidation)
   - Context propagation in Go

#### Step 1: Understand When to Use Caching

**Use entcache when:**
- GraphQL queries cause redundant database calls
- Same data is queried multiple times per request
- Reference data is accessed frequently

**Avoid entcache when:**
- Data changes frequently and staleness is unacceptable
- You need transactional consistency (mutations bypass cache)
- Debugging database queries (cache can hide issues)

#### Step 2: Explore the Examples

```bash
cd /Users/ibdrew/Documents/entcache/internal/examples

# Context-level example (most common)
cd ctxlevel
go run main.go

# Multi-level example (production pattern)
cd ../multilevel
go run main.go
```

#### Step 3: Review www/server Integration

```bash
# See how www/server configures entcache
cd /Users/ibdrew/Documents/www/server

# Look for entcache usage
grep -r "entcache" .
```

---

## Configuration Examples

### Basic Context-Level Cache

```go
import (
    "database/sql"
    "ariga.io/entcache"
    "entgo.io/ent/dialect"
    entsql "entgo.io/ent/dialect/sql"
)

func NewClient() *ent.Client {
    // Open database connection
    db, err := sql.Open("postgres", "...")
    if err != nil {
        log.Fatal(err)
    }
    
    // Wrap with entcache driver (context-level mode)
    drv := entcache.NewDriver(
        entsql.OpenDB(dialect.Postgres, db),
        entcache.ContextLevel(), // ← Per-request caching
    )
    
    return ent.NewClient(ent.Driver(drv))
}
```

### Middleware for GraphQL

```go
// Wrap GraphQL requests with cache context
srv.AroundResponses(func(ctx context.Context, next graphql.ResponseHandler) *graphql.Response {
    if op := graphql.GetOperationContext(ctx).Operation; op != nil && op.Operation == ast.Query {
        ctx = entcache.NewContext(ctx) // ← Attach cache to context
    }
    return next(ctx)
})
```

### Driver-Level Cache with TTL

```go
drv := entcache.NewDriver(
    entsql.OpenDB(dialect.Postgres, db),
    entcache.TTL(30 * time.Second), // Cache entries expire after 30s
)
```

### Multi-Level Cache (LRU + Redis)

```go
import (
    "github.com/redis/go-redis/v9"
    "ariga.io/entcache"
)

func NewClient() *ent.Client {
    // Redis connection
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })
    
    drv := entcache.NewDriver(
        entsql.OpenDB(dialect.Postgres, db),
        entcache.TTL(time.Minute),
        entcache.Levels(
            entcache.NewLRU(256),     // L1: Local LRU (256 entries)
            entcache.NewRedis(rdb),   // L2: Distributed Redis
        ),
    )
    
    return ent.NewClient(ent.Driver(drv))
}
```

### Skipping Cache for Mutations

```go
// Migrations should bypass the cache
if err := client.Schema.Create(entcache.Skip(ctx)); err != nil {
    log.Fatal(err)
}

// Explicit cache skip for specific queries
users, err := client.User.Query().
    Where(user.IDEQ(123)).
    All(entcache.Skip(ctx)) // ← Forces database query
```

---

## Performance Considerations

### Cache Hit Ratio

Monitor your cache hit ratio to ensure effectiveness:

```go
// Custom cache with metrics
type metricsCache struct {
    entcache.Cache
    hits, misses int64
}

func (c *metricsCache) Get(ctx context.Context, key string) (*entcache.Entry, error) {
    entry, err := c.Cache.Get(ctx, key)
    if entry != nil {
        atomic.AddInt64(&c.hits, 1)
    } else {
        atomic.AddInt64(&c.misses, 1)
    }
    return entry, err
}
```

### Memory Usage

- **LRU Cache:** Set appropriate size limits based on available memory
- **Context Cache:** Automatically cleaned up when request ends
- **Redis:** Monitor memory usage and configure maxmemory policy

### TTL Selection

| Data Type | Suggested TTL | Rationale |
|-----------|---------------|-----------|
| Static reference data | 5-15 minutes | Rarely changes |
| User profiles | 30-60 seconds | Changes on edit |
| Session data | No cache | Must be real-time |
| Search results | 10-30 seconds | Can be slightly stale |

---

## Troubleshooting

### Common Issues

#### Stale Data After Mutation

**Symptom:** Data doesn't update after a mutation

**Cause:** Cached query results aren't invalidated

**Solutions:**
1. Use context-level caching only (cache dies with request)
2. Set appropriate TTL for driver-level cache
3. Manually skip cache after mutations:
   ```go
   // After mutation, re-query with skip
   user, _ = client.User.Get(entcache.Skip(ctx), id)
   ```

#### High Memory Usage

**Symptom:** Server memory grows unbounded

**Cause:** LRU cache too large or no size limit

**Solution:**
```go
// Limit LRU cache size
entcache.Levels(entcache.NewLRU(1000)) // Max 1000 entries
```

#### Cache Not Working

**Symptom:** Same query hits database repeatedly

**Causes & Solutions:**
1. **Missing context wrapper:**
   ```go
   ctx = entcache.NewContext(ctx) // Add this
   ```
2. **Different query parameters:**
   - Queries with different WHERE clauses have different cache keys
3. **Mutations in between:**
   - Mutations don't go through cache, ensure you're using Query()

#### Redis Connection Issues

**Symptom:** Errors on cache operations, fallback to database

**Solution:**
```go
// Handle Redis errors gracefully
drv := entcache.NewDriver(
    db,
    entcache.Levels(
        entcache.NewLRU(256),  // Fallback to LRU if Redis fails
        entcache.NewRedis(rdb),
    ),
)
```

---

## Related Resources

- [Entcache Official Documentation](https://pkg.go.dev/ariga.io/entcache)
- [Ent Documentation](https://entgo.io)
- [N+1 Query Problem in GraphQL](https://entgo.io/docs/tutorial-todo-gql-field-collection/#problem)
- [Relay Cursor Connections](https://relay.dev/graphql/connections.htm)

---

## Relationship to Other Repos

| Repository | Relationship |
|------------|-------------|
| `ent-contrib` | Provides entgql which works with entcache for cached GraphQL APIs |
| `www/server` | Primary consumer - uses entcache for query caching |
| `dreamscape-go` | Planned integration for performance optimization |
| `Hydra` | Manages infrastructure including Redis for distributed caching |

---

## Contact

For questions about entcache usage in Bravo Zero:
- Check the `#backend` channel
- Review www/server implementation
- Consult this documentation















