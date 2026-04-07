# MySQL & Data — Daily Prompts

> Tech stack: MySQL 8, Spring Data JPA / Hibernate, Flyway, Azure Database for MySQL Flexible Server

---

## Schema Design

### Design a new table
```
Design a MySQL table for [entity] with these business requirements:
[describe requirements]

Include:
- Column types (prefer precise types: BIGINT for IDs, DECIMAL for money,
  DATETIME(3) for timestamps, VARCHAR with explicit lengths)
- Primary key strategy (AUTO_INCREMENT BIGINT vs UUID)
- NOT NULL constraints, DEFAULTs
- Indexes (covering index for [common query pattern])
- Foreign keys with ON DELETE strategy
- Audit columns: created_at, updated_at, created_by
- Soft delete: deleted_at column (nullable)
- Table options: ENGINE=InnoDB, CHARSET=utf8mb4, COLLATE=utf8mb4_unicode_ci
- Estimated row count: [X] for partitioning consideration

Show: CREATE TABLE DDL and the matching JPA @Entity class.
```

### Add a column safely (zero-downtime)
```
I need to add column [name] ([type]) to table [table] which has [X million] rows
on Azure MySQL Flexible Server.

Show me a zero-downtime migration plan:
1. Flyway migration script (V[version]__description.sql)
2. Will ALTER TABLE lock the table? How long for [X]M rows?
3. Do I need pt-online-schema-change or gh-ost? Or is MySQL 8 instant DDL enough?
4. Application code changes (nullable first, then backfill, then enforce NOT NULL)
5. Rollback plan
6. How to verify in staging before prod
```

### Design indexes for query patterns
```
Here are my most common query patterns for table [table_name]:
1. [SELECT ... WHERE column_a = ? AND column_b > ? ORDER BY column_c]
2. [SELECT ... WHERE column_d IN (?, ?) AND column_e = ?]
3. [SELECT COUNT(*) ... GROUP BY column_f WHERE column_g BETWEEN ? AND ?]

Current table has [X million] rows, grows by [Y]/day.

For each query:
- Recommend the optimal index (column order matters!)
- Explain why this column order (equality → range → sort)
- Show the CREATE INDEX statement
- Show EXPLAIN ANALYZE before/after
- Flag any index that would hurt write performance

Also flag if any indexes are redundant (left-prefix overlap).
```

---

## Query Optimization

### Analyze a slow query
```
This query is slow (~[X]s, target <[Y]ms):
[paste query]

Table stats: [table] has [X million] rows.
Current indexes: [list]

Analyze:
1. Run EXPLAIN ANALYZE — interpret the output
2. Identify: full table scan? filesort? temp table? index not used?
3. Suggest index changes
4. Rewrite the query if needed (subquery → JOIN, OR → UNION, etc.)
5. Show the optimized query and expected EXPLAIN output
6. If data volume is the issue, suggest partitioning strategy
```

### Optimize a JPA/Hibernate query
```
This JPA repository method is generating inefficient SQL:
[paste repository method or @Query]

Generated SQL (from show-sql): [paste SQL]

Diagnose and fix:
- N+1 problem? → @EntityGraph or JOIN FETCH
- Fetching unnecessary columns? → interface-based projection
- Loading entire entity for a count? → @Query with COUNT
- Sorting in memory instead of DB? → ORDER BY in query
- Missing index for the WHERE clause?

Show the fixed repository method AND the SQL it generates.
```

### Bulk operations
```
I need to [insert/update/delete] [X thousand] records in [table].

Current approach: [describe — e.g., loop with save(), single UPDATE]
Problem: [too slow / locks too long / runs out of memory]

Show me the optimal approach using:
1. Spring Data JPA batch inserts (hibernate.jdbc.batch_size)
2. @Modifying @Query for bulk UPDATE/DELETE
3. JDBC batch with JdbcTemplate.batchUpdate()
4. Chunked processing (process N records at a time, commit per chunk)

For each, show code and explain when to use it.
Which approach avoids long-running transactions that block reads?
```

---

## Flyway Migrations

### Create a migration
```
Create a Flyway migration script for:
[describe change: new table / add column / create index / data backfill]

Requirements:
- Filename: V[version]__[description].sql
- Idempotent where possible (IF NOT EXISTS)
- Include a ROLLBACK comment block (manual steps since Flyway Community doesn't auto-rollback)
- Estimate execution time for [X million] rows
- Flag if this migration needs downtime or can run online
```

### Fix a failed migration
```
Flyway migration V[X]__[name].sql failed with this error:
[paste error]

Current state: flyway_schema_history shows version [X] as failed=1.

Walk me through:
1. Root cause of the error
2. How to fix the migration SQL
3. How to repair flyway_schema_history (flyway repair)
4. Steps to re-run safely
5. How to prevent this in CI (validate before applying)
```

---

## Data Modeling Decisions

### Normalization vs denormalization
```
I'm designing the data model for [feature]. I need to store:
[describe entities and relationships]

Query patterns: [list most common reads and writes]
Scale: [estimated data volume and growth rate]

Should I normalize or denormalize? Consider:
- Read vs write ratio
- Join cost at scale
- Data consistency requirements
- Caching opportunities
- Reporting needs

Show both designs, compare with concrete query examples, and recommend one.
```

### Soft delete strategy
```
Implement soft delete for [entity] in Spring Data JPA + MySQL.

Show:
- Entity: deleted_at DATETIME(3) column (null = active)
- @Where(clause = "deleted_at IS NULL") or @SQLRestriction
- Unique constraint that allows re-creation: UNIQUE(email, deleted_at)
- Repository: add findAllIncludingDeleted() using @Query
- Service: softDelete() sets deleted_at = now(), don't JPA delete
- Scheduled cleanup: purge records where deleted_at < 90 days ago
- Index: partial index concept (MySQL workaround with generated column)
```

---

## Azure MySQL Specific

### Connection configuration
```
Generate the optimal Spring Boot application.yml for connecting to
Azure Database for MySQL Flexible Server.

Include:
- JDBC URL with SSL, serverTimezone, cachePrepStmts, useServerPrepStmts
- HikariCP pool settings for [N] pods × [M] pool size ≤ MySQL max_connections
- Hibernate dialect and batch settings
- Flyway configuration
- Azure AD authentication via Managed Identity (passwordless)
- Connection retry with spring.datasource.hikari.initializationFailTimeout

Show dev profile (local MySQL via Docker) and prod profile (Azure MySQL).
```

---

## Quick One-Liners

```
Write a MySQL query to find duplicate records in [table] based on [columns].
```

```
Convert this raw SQL to a Spring Data JPA @Query: [paste SQL]
```

```
What's the MySQL equivalent of PostgreSQL's [feature]? Show syntax.
```

```
Estimate the storage size for [table] with [X million] rows and these columns: [list].
```

```
Show me how to use MySQL's JSON column type with JPA for storing [flexible data].
```

```
Write a Flyway migration to add a composite index on (column_a, column_b) — what order and why?
```

---

*Measure twice, index once. Always EXPLAIN ANALYZE before and after.*
