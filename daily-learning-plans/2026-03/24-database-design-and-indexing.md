# Day 17 — Database Design & Indexing
**Date:** 2026-03-24
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Database Design & Indexing — Schema Design, Query Optimisation, Index Strategies**

The database is almost always the bottleneck. Today you sharpen your MySQL expertise with a system design lens — moving from "it works" to "it works at 10x traffic." You'll revisit normalisation trade-offs, understand B-Tree vs hash indexes at a structural level, learn to read EXPLAIN plans like a story, and practice designing schemas that serve read-heavy and write-heavy workloads differently.

---

## 2. Core Concepts

### 2.1 Schema Design Principles

**Normalisation vs Denormalisation — The Trade-Off:**

| Approach | Reads | Writes | Storage | Consistency |
|----------|-------|--------|---------|-------------|
| Normalised (3NF) | Slower (joins) | Faster (single write) | Less | Strong |
| Denormalised | Faster (no joins) | Slower (update multiple places) | More | Eventual (must sync) |

**Rule of thumb:** Normalise by default. Denormalise specific read paths when query latency matters and you can accept the write complexity.

```sql
-- Normalised: order + order_items + product (3 joins)
SELECT o.id, oi.quantity, p.name, p.price
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.user_id = 42;

-- Denormalised: order_details materialised view (0 joins)
SELECT id, quantity, product_name, product_price
FROM order_details_view
WHERE user_id = 42;
```

### 2.2 Index Internals — B-Tree

```
                    [50]
                   /    \
              [20,30]   [70,80]
             /  |  \    /  |  \
           [10][25][35][60][75][90]
            │   │   │   │   │   │
           rows rows rows rows rows rows
```

**B-Tree index properties:**
- Balanced — every leaf is at the same depth → O(log n) lookups.
- Sorted — supports range queries (`BETWEEN`, `>`, `<`, `ORDER BY`).
- Leftmost prefix — a composite index `(a, b, c)` supports queries on `(a)`, `(a, b)`, and `(a, b, c)`, but NOT `(b)` or `(c)` alone.

### 2.3 Index Types in MySQL

| Type | Use Case | Supports Range? | Storage |
|------|----------|----------------|---------|
| B-Tree (default) | Most queries | Yes | O(n) |
| Hash (Memory engine) | Exact match only | No | O(n) |
| Full-Text | Text search (`MATCH ... AGAINST`) | N/A | Inverted index |
| Spatial (R-Tree) | Geographic data | N/A | Bounding boxes |

### 2.4 The Composite Index Rule

```
Index: (country, city, created_at)

✅ WHERE country = 'IN'                          → uses index
✅ WHERE country = 'IN' AND city = 'BLR'         → uses index
✅ WHERE country = 'IN' AND city = 'BLR'
   AND created_at > '2026-01-01'                 → uses full index
✅ WHERE country = 'IN' ORDER BY city             → uses index for both

❌ WHERE city = 'BLR'                             → skips index (no leftmost)
❌ WHERE country = 'IN' OR city = 'BLR'           → may skip index (OR is hard)
⚠️ WHERE country = 'IN' AND created_at > '2026-01-01'
   → uses index on (country) only, skips city gap
```

### 2.5 Reading EXPLAIN Output

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 42 AND status = 'SHIPPED';
```

| Column | What It Tells You |
|--------|-------------------|
| `type` | `ALL` = full scan (bad), `ref` = index lookup (good), `range` = index range scan, `const` = single row |
| `key` | Which index MySQL actually chose |
| `rows` | Estimated rows to examine (lower is better) |
| `Extra` | `Using index` = covering index, `Using filesort` = extra sort step, `Using temporary` = temp table |

**The goal:** `type` should be `ref`, `range`, or `const`. Never `ALL` on large tables.

### 2.6 Covering Index

A covering index contains all columns needed by the query — MySQL never touches the table data.

```sql
-- Query
SELECT user_id, status, created_at FROM orders WHERE user_id = 42;

-- Covering index (all selected columns are in the index)
CREATE INDEX idx_orders_covering ON orders(user_id, status, created_at);

-- EXPLAIN will show: Using index (no table access)
```

---

## 3. Write-Path Optimisation

### 3.1 Index Cost on Writes

Every index slows `INSERT`/`UPDATE`/`DELETE` because the index must be maintained.

**Guidelines:**
- Read-heavy tables (90% reads): index liberally.
- Write-heavy tables (logs, events): minimal indexes, batch inserts.
- OLTP tables: 3–5 well-chosen indexes usually suffice.

### 3.2 Partitioning

Split large tables into smaller physical pieces while keeping a single logical table.

```sql
CREATE TABLE events (
    id BIGINT AUTO_INCREMENT,
    event_type VARCHAR(50),
    created_at DATETIME NOT NULL,
    payload JSON,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p2026 VALUES LESS THAN (2027),
    PARTITION pmax  VALUES LESS THAN MAXVALUE
);
```

**Benefits:** Partition pruning (queries on `created_at` only scan relevant partitions), fast `DROP PARTITION` for data retention.

---

## 4. Hands-On Exercise

### Task: Optimise an E-Commerce Schema

Given this simplified schema:

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255),
    country VARCHAR(2),
    created_at DATETIME
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    status ENUM('PENDING','SHIPPED','DELIVERED','CANCELLED'),
    total_amount DECIMAL(10,2),
    created_at DATETIME
);

CREATE TABLE order_items (
    id BIGINT PRIMARY KEY,
    order_id BIGINT,
    product_id BIGINT,
    quantity INT,
    unit_price DECIMAL(10,2)
);
```

**Queries to optimise:**

1. `SELECT * FROM orders WHERE user_id = ? AND status = 'SHIPPED' ORDER BY created_at DESC LIMIT 20;`
2. `SELECT user_id, SUM(total_amount) FROM orders WHERE created_at BETWEEN ? AND ? GROUP BY user_id;`
3. `SELECT oi.product_id, SUM(oi.quantity) FROM order_items oi JOIN orders o ON oi.order_id = o.id WHERE o.status = 'DELIVERED' GROUP BY oi.product_id;`

**For each query:**
1. Write the `CREATE INDEX` statement.
2. Run `EXPLAIN` (mentally or on a real DB) and verify the index is used.
3. Identify if a covering index is possible.

### Expected Indexes:
```sql
-- Query 1: composite on (user_id, status, created_at)
CREATE INDEX idx_orders_user_status_date ON orders(user_id, status, created_at);

-- Query 2: composite on (created_at, user_id, total_amount) — covering
CREATE INDEX idx_orders_date_user_amount ON orders(created_at, user_id, total_amount);

-- Query 3: index on order_items(order_id, product_id, quantity) + orders(status)
CREATE INDEX idx_oi_order_product_qty ON order_items(order_id, product_id, quantity);
CREATE INDEX idx_orders_status ON orders(status);
```

---

## 5. Common Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|-------------|-------------|-----|
| `SELECT *` | Prevents covering indexes | Select only needed columns |
| Index on every column | Slows writes, wastes storage | Index for actual query patterns |
| `LIKE '%search%'` | Cannot use B-Tree index | Full-text index or Elasticsearch |
| Implicit type conversion | `WHERE varchar_col = 123` skips index | Match types explicitly |
| N+1 queries | 1 query + N lookups | JOIN or batch fetch |
| UUID as primary key | Random inserts fragment B-Tree | Use ordered UUIDs (UUIDv7) or BIGINT |

---

## 6. Key Takeaways

1. **Design indexes for queries, not tables** — look at your top 10 queries and work backwards.
2. **Composite index order matters** — equality columns first, then range, then sort.
3. **Covering indexes eliminate table lookups** — include all selected columns when practical.
4. **Every index is a write penalty** — balance read speed vs write overhead.
5. **EXPLAIN is your best friend** — never deploy a new query without checking the plan.

---

## 7. Connection to Previous Days

In Phase 1, you used pgvector with HNSW indexes (Day 4) for semantic search. Today's B-Tree indexes serve the same purpose — faster lookups — but for exact and range queries. The same "index for your query pattern" principle applies to both relational and vector databases.

---

## 8. Resources

- **[Use The Index, Luke](https://use-the-index-luke.com/)** — The best free resource on SQL indexing.
- **[MySQL EXPLAIN documentation](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)** — Official reference for reading query plans.
- **[Designing Data-Intensive Applications, Ch. 3](https://dataintensive.net/)** — Storage and retrieval deep dive.

---

*Day 17. Index for your queries, not your tables.*
