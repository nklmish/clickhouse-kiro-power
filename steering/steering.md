# ClickHouse Best Practices Steering Guide

This steering file provides comprehensive guidance for designing schemas, optimizing queries, and managing data ingestion in ClickHouse. It contains 28 rules across three categories, prioritized by impact.

> Adapted from the official [ClickHouse Agent Skills](https://github.com/ClickHouse/agent-skills) and [ClickHouse Best Practices](https://clickhouse.com/docs/best-practices).

## When to Use This Guide

Use these best practices when you encounter:
- `CREATE TABLE` statements or schema design discussions
- `ALTER TABLE` modifications
- `ORDER BY` or `PRIMARY KEY` decisions
- Data type selection questions
- Slow query troubleshooting
- JOIN optimization requests
- Data ingestion pipeline design
- Update/delete strategy questions
- Partitioning strategy decisions

## How to Apply

**Priority order when answering ClickHouse questions:**
1. Check for applicable rules below
2. Apply them and cite them: "Per the primary key planning rule..."
3. If no rule applies, use general ClickHouse knowledge
4. If uncertain, search ClickHouse documentation

---

## Rule Categories by Priority

| Priority | Category | Impact | Rule Count |
|----------|----------|--------|------------|
| 1 | Primary Key Selection | CRITICAL | 4 |
| 2 | Data Type Selection | CRITICAL | 5 |
| 3 | JOIN Optimization | CRITICAL | 5 |
| 4 | Insert Batching | CRITICAL | 1 |
| 5 | Mutation Avoidance | CRITICAL | 2 |
| 6 | Partitioning Strategy | HIGH | 4 |
| 7 | Skipping Indices | HIGH | 1 |
| 8 | Materialized Views | HIGH | 2 |
| 9 | Async Inserts | HIGH | 2 |
| 10 | OPTIMIZE Avoidance | HIGH | 1 |
| 11 | JSON Usage | MEDIUM | 1 |

---

## 1. Schema Design (CRITICAL)

ORDER BY is immutable after table creation. Wrong choices require full data migration.

### 1.1 Plan PRIMARY KEY Before Table Creation

ORDER BY cannot be modified after table creation. Analyze query patterns first.

**Pre-creation checklist:**
- List top 5–10 query patterns
- Identify columns in WHERE clauses with frequency
- Prioritize columns that exclude large numbers of rows
- Order columns by cardinality (low first, high last)
- Limit to 4–5 key columns

```sql
-- Step 1: Document query patterns BEFORE creating table
-- 60% queries: WHERE user_id = ? AND timestamp BETWEEN ? AND ?
-- 25% queries: WHERE event_type = ? AND timestamp > ?

-- Step 2: Create table with correct ORDER BY
CREATE TABLE events (
    event_id UUID DEFAULT generateUUIDv4(),
    user_id UInt64,
    event_type LowCardinality(String),
    timestamp DateTime,
    event_date Date DEFAULT toDate(timestamp)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (user_id, event_date, event_id);
```

### 1.2 Order Columns by Cardinality (Low to High)

Low-cardinality leading columns create more useful index entries that skip entire granules.

| Position | Cardinality | Examples |
|----------|-------------|----------|
| 1st | Low (few distinct values) | event_type, status, country |
| 2nd | Date (coarse granularity) | toDate(timestamp) |
| 3rd+ | Medium-High | user_id, session_id |
| Last | High (if needed) | event_id, uuid |

```sql
-- ❌ High cardinality first — no pruning benefit
ORDER BY (event_id, event_type, timestamp);

-- ✅ Low cardinality first — enables pruning
ORDER BY (event_type, event_date, event_id);
```

### 1.3 Prioritize Filter Columns in ORDER BY

Queries filtering on columns not in ORDER BY result in full table scans.

```sql
-- Given: ORDER BY (tenant_id, event_type, timestamp)

-- ✅ Full prefix match — best performance
SELECT * FROM events WHERE tenant_id = 123 AND event_type = 'click';

-- ✅ Partial prefix — still uses index
SELECT * FROM events WHERE tenant_id = 123;

-- ❌ Skips prefix — can't use index
SELECT * FROM events WHERE event_type = 'click';
```

### 1.4 Filter on ORDER BY Columns in Queries

Even with good schema design, queries must use ORDER BY columns to benefit.

| Filter | Index Used? |
|--------|-------------|
| `WHERE tenant_id = 123` | Full |
| `WHERE tenant_id = 123 AND event_type = 'click'` | Full |
| `WHERE event_type = 'click'` | None (skipped prefix) |
| `WHERE timestamp > '2024-01-01'` | None (skipped both) |

### 1.5 Use Native Types Instead of String

Using String for all data wastes storage and prevents compression optimization.

| Data | Use | Avoid |
|------|-----|-------|
| Sequential IDs | UInt32/UInt64 | String |
| UUIDs | UUID | String |
| Status/Category | Enum8 or LowCardinality(String) | String |
| Timestamps | DateTime | String |
| Counts | UInt8/16/32 (smallest that fits) | Int64, String |
| Money | Decimal(P,S) or Int64 (cents) | Float64, String |
| Booleans | Bool or UInt8 | String |

### 1.6 Minimize Bit-Width for Numeric Types

| Type | Range | Bytes |
|------|-------|-------|
| UInt8 | 0 to 255 | 1 |
| UInt16 | 0 to 65,535 | 2 |
| UInt32 | 0 to 4.3 billion | 4 |
| UInt64 | 0 to 18 quintillion | 8 |

```sql
-- ❌ Oversized
CREATE TABLE metrics (status_code Int64, age Int64);

-- ✅ Right-sized
CREATE TABLE metrics (status_code UInt16, age UInt8);
```

### 1.7 Use LowCardinality for Repeated Strings

Dictionary encoding for columns with fewer than 10K unique values.

```sql
-- Check cardinality first
SELECT uniq(column_name) FROM table_name;

-- ✅ Use LowCardinality for low unique counts
CREATE TABLE events (
    country LowCardinality(String),      -- ~200 unique values
    browser LowCardinality(String),      -- ~50 unique values
    event_type LowCardinality(String)    -- ~100 unique values
)
```

| Unique Values | Recommendation |
|---------------|----------------|
| < 10,000 | Use LowCardinality |
| > 10,000 | Use regular String |

### 1.8 Avoid Nullable Unless Semantically Required

Nullable adds a separate UInt8 column for tracking null values.

```sql
-- ❌ Nullable everywhere
CREATE TABLE users (
    id Nullable(UInt64),
    name Nullable(String),
    age Nullable(UInt8)
)

-- ✅ DEFAULT values, Nullable only when semantic
CREATE TABLE users (
    id UInt64,
    name String DEFAULT '',
    age UInt8 DEFAULT 0,
    deleted_at Nullable(DateTime),    -- NULL = not deleted (semantic)
    parent_id Nullable(UInt64)        -- NULL = no parent (semantic)
)
```

### 1.9 Use Enum for Finite Value Sets

Insert-time validation and natural ordering. 1–2 bytes storage.

```sql
CREATE TABLE orders (
    status Enum8('pending' = 1, 'processing' = 2, 'shipped' = 3, 'delivered' = 4)
)

-- Invalid values rejected at insert time
INSERT INTO orders VALUES ('shiped');  -- ERROR
```

### 1.10 Use JSON Type for Dynamic Schemas

For truly dynamic data only. Use typed columns for known schemas.

```sql
CREATE TABLE events (
    event_id UUID DEFAULT generateUUIDv4(),
    event_type LowCardinality(String),
    timestamp DateTime DEFAULT now(),
    properties JSON  -- Flexible schema with type inference
)
ENGINE = MergeTree()
ORDER BY (event_type, timestamp);

-- Query JSON paths directly
SELECT properties.url, properties.amount FROM events
WHERE event_type = 'purchase';
```

---

## 2. Partitioning Strategy (HIGH)

### 2.1 Keep Partition Cardinality Low (100–1,000 Values)

Too many partitions cause "too many parts" errors.

```sql
-- ❌ High cardinality — millions of partitions
PARTITION BY user_id

-- ✅ Bounded cardinality — 12 per year
PARTITION BY toStartOfMonth(timestamp)
```

### 2.2 Use Partitioning for Data Lifecycle Management

Partitioning is primarily a data management technique, not a query optimization tool.

```sql
CREATE TABLE events (...)
ENGINE = MergeTree()
PARTITION BY toStartOfMonth(timestamp)
ORDER BY (event_type, timestamp)
TTL timestamp + INTERVAL 1 YEAR DELETE;

-- Fast: metadata-only operation
ALTER TABLE events DROP PARTITION '202301';
```

### 2.3 Consider Starting Without Partitioning

Add partitioning later when you have clear lifecycle requirements.

### 2.4 Understand Partition Query Trade-offs

- Queries filtering by partition key benefit from partition pruning
- Queries spanning many partitions increase total parts scanned

---

## 3. Query Optimization (CRITICAL)

### 3.1 Choose the Right JOIN Algorithm

| Algorithm | Best For | Trade-off |
|-----------|----------|-----------|
| `parallel_hash` | Small-to-medium in-memory | Default since 24.11; fast |
| `hash` | General purpose | Single-threaded build |
| `direct` | Dictionary lookups (INNER/LEFT) | Fastest; no hash table |
| `full_sorting_merge` | Tables sorted on join key | Low memory |
| `partial_merge` | Large tables, memory-constrained | Slower execution |
| `grace_hash` | Large datasets, tunable memory | Disk-spilling |
| `auto` | Adaptive selection | Tries hash first |

```sql
-- For large-to-large joins where memory is constrained
SET join_algorithm = 'partial_merge';
SELECT * FROM large_a JOIN large_b ON large_b.id = large_a.id;
```

### 3.2 Consider Alternatives to JOINs

| Approach | Use Case | Performance |
|----------|----------|-------------|
| Dictionary | Frequent lookups to small dimension | Fastest (in-memory) |
| Denormalization | Analytics always need enriched data | Fast (no join at query) |
| IN subquery | Existence filtering | Often faster than JOIN |
| JOIN | Infrequent or complex joins | Acceptable |

```sql
-- Dictionary lookup instead of JOIN
CREATE DICTIONARY customer_dict (
    id UInt64, name String, email String
)
PRIMARY KEY id
SOURCE(CLICKHOUSE(TABLE 'customers'))
LAYOUT(HASHED())
LIFETIME(MIN 300 MAX 360);

SELECT order_id,
    dictGet('customer_dict', 'name', customer_id) as customer_name
FROM orders;
```

### 3.3 Filter Tables Before Joining

```sql
-- ❌ Join then filter
SELECT o.order_id, c.name FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at > '2024-01-01' AND c.country = 'US';

-- ✅ Filter in subqueries before joining
SELECT o.order_id, c.name FROM (
    SELECT order_id, customer_id FROM orders WHERE created_at > '2024-01-01'
) o
JOIN (
    SELECT id, name FROM customers WHERE country = 'US'
) c ON c.id = o.customer_id;
```

### 3.4 Use ANY JOIN When Only One Match Needed

```sql
-- ✅ Returns first match only — faster, less memory
SELECT o.order_id, c.name FROM orders o
LEFT ANY JOIN customers c ON c.id = o.customer_id;
```

### 3.5 Optimize NULL Handling in Outer JOINs

```sql
-- Use default values instead of NULLs for non-matching rows
SET join_use_nulls = 0;
```

### 3.6 Use Data Skipping Indices for Non-ORDER BY Filters

| Type | Best For | Example Filter |
|------|----------|----------------|
| `bloom_filter` | Equality on high-cardinality | `WHERE user_id = 123` |
| `set(N)` | Low cardinality (N unique values) | `WHERE status IN ('a','b')` |
| `minmax` | Range queries | `WHERE amount > 1000` |
| `ngrambf_v1` | Text search | `WHERE text LIKE '%term%'` |
| `tokenbf_v1` | Token search | `WHERE hasToken(text, 'word')` |

```sql
-- Add skipping index
ALTER TABLE events ADD INDEX idx_user_id user_id TYPE bloom_filter GRANULARITY 4;
ALTER TABLE events MATERIALIZE INDEX idx_user_id;

-- Verify index usage
EXPLAIN indexes = 1 SELECT * FROM events WHERE user_id = 12345;
```

### 3.7 Use Incremental MVs for Real-Time Aggregations

Pre-compute aggregations at insert time. Read thousands of rows instead of billions.

```sql
-- Target table
CREATE TABLE events_hourly (
    event_type LowCardinality(String),
    hour DateTime,
    events AggregateFunction(count),
    unique_users AggregateFunction(uniq, UInt64)
)
ENGINE = AggregatingMergeTree()
ORDER BY (event_type, hour);

-- Materialized view
CREATE MATERIALIZED VIEW events_hourly_mv TO events_hourly AS
SELECT event_type, toStartOfHour(timestamp) as hour,
    countState() as events, uniqState(user_id) as unique_users
FROM events GROUP BY event_type, hour;

-- Query pre-aggregated data
SELECT event_type, hour,
    countMerge(events) as events, uniqMerge(unique_users) as unique_users
FROM events_hourly
WHERE hour >= now() - INTERVAL 7 DAY
GROUP BY event_type, hour;
```

### 3.8 Use Refreshable MVs for Complex Joins and Batch Workflows

```sql
CREATE MATERIALIZED VIEW orders_denormalized
REFRESH EVERY 5 MINUTE
ENGINE = MergeTree()
ORDER BY (created_at, order_id)
AS SELECT o.order_id, o.created_at, o.total,
    c.name as customer_name, p.name as product_name
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id
WHERE o.created_at >= now() - INTERVAL 1 DAY;
```

---

## 4. Insert Strategy (CRITICAL)

### 4.1 Batch Inserts (10K–100K Rows)

Each INSERT creates a data part. Single-row inserts overwhelm the merge process.

| Threshold | Value |
|-----------|-------|
| Minimum | 1,000 rows |
| Ideal range | 10,000–100,000 rows |
| Insert rate (sync) | ~1 insert per second |

```sql
-- Monitor part count (>3000 per partition blocks inserts)
SELECT table, count() as parts, sum(rows) as total_rows
FROM system.parts WHERE active AND database = 'default'
GROUP BY table ORDER BY parts DESC;
```

### 4.2 Use Async Inserts for High-Frequency Small Batches

When client-side batching isn't practical:

```sql
ALTER USER my_app_user SETTINGS
    async_insert = 1,
    wait_for_async_insert = 1,
    async_insert_max_data_size = 10000000,  -- Flush at 10MB
    async_insert_busy_timeout_ms = 1000;    -- Flush after 1s
```

### 4.3 Use Native Format for Best Insert Performance

| Format | Notes |
|--------|-------|
| **Native** | Most efficient. Column-oriented, minimal parsing. |
| **RowBinary** | Efficient row-based alternative |
| **JSONEachRow** | Easier to use but expensive to parse |

### 4.4 Avoid ALTER TABLE UPDATE

Mutations rewrite entire data parts. Use ReplacingMergeTree instead.

```sql
CREATE TABLE users (
    user_id UInt64,
    name String,
    status LowCardinality(String),
    updated_at DateTime DEFAULT now()
)
ENGINE = ReplacingMergeTree(updated_at)
ORDER BY user_id;

-- "Update" by inserting new version
INSERT INTO users (user_id, name, status) VALUES (123, 'John', 'inactive');

-- Query with FINAL to get latest version
SELECT * FROM users FINAL WHERE user_id = 123;
```

### 4.5 Avoid ALTER TABLE DELETE

Use lightweight DELETE, CollapsingMergeTree, or DROP PARTITION instead.

| Method | Speed | When to Use |
|--------|-------|-------------|
| ALTER DELETE | Slow | Rare corrections only |
| CollapsingMergeTree | Fast | Frequent soft deletes |
| Lightweight DELETE | Medium | Occasional deletes |
| DROP PARTITION | Instant | Bulk deletion by partition |

### 4.6 Avoid OPTIMIZE TABLE FINAL

Let background merges work. `OPTIMIZE FINAL` rewrites entire partitions and can cause OOM.

For ReplacingMergeTree deduplication, use `FINAL` in SELECT queries instead.

---

## Schema Review Checklist

When reviewing a CREATE TABLE statement, check:

- [ ] PRIMARY KEY / ORDER BY column order (low-to-high cardinality)
- [ ] Data types match actual data ranges
- [ ] LowCardinality applied to appropriate string columns
- [ ] Partition key cardinality bounded (100–1,000 values)
- [ ] ReplacingMergeTree has version column if used
- [ ] No unnecessary Nullable columns
- [ ] Enum used for finite value sets with validation needs

## Query Review Checklist

- [ ] Filters use ORDER BY prefix columns
- [ ] JOINs filter tables before joining (not after)
- [ ] Correct JOIN algorithm for table sizes
- [ ] Skipping indices for non-ORDER BY filter columns
- [ ] ANY JOIN used when only one match needed

## Insert Review Checklist

- [ ] Batch size 10K–100K rows per INSERT
- [ ] No ALTER TABLE UPDATE for frequent changes
- [ ] ReplacingMergeTree or CollapsingMergeTree for update patterns
- [ ] Async inserts enabled for high-frequency small batches
- [ ] No scheduled OPTIMIZE TABLE FINAL

---

## Common System Table Queries

### Check Table Health
```sql
SELECT database, table, engine, sorting_key, partition_key,
    total_rows, formatReadableSize(total_bytes) as size
FROM system.tables
WHERE database NOT IN ('system', 'INFORMATION_SCHEMA', 'information_schema')
ORDER BY total_bytes DESC;
```

### Monitor Part Count
```sql
SELECT database, table, count() as parts, sum(rows) as rows,
    formatReadableSize(sum(bytes_on_disk)) as disk_size
FROM system.parts WHERE active
GROUP BY database, table
ORDER BY parts DESC;
```

### Find Slow Queries
```sql
SELECT query, query_duration_ms, read_rows, read_bytes,
    memory_usage, result_rows
FROM system.query_log
WHERE type = 'QueryFinish' AND query_duration_ms > 1000
    AND event_time > now() - INTERVAL 1 HOUR
ORDER BY query_duration_ms DESC LIMIT 20;
```

### Column Compression Analysis
```sql
SELECT column, type,
    formatReadableSize(sum(column_data_compressed_bytes)) as compressed,
    formatReadableSize(sum(column_data_uncompressed_bytes)) as uncompressed,
    round(sum(column_data_uncompressed_bytes) / sum(column_data_compressed_bytes), 2) as ratio
FROM system.parts_columns
WHERE table = 'your_table' AND database = 'your_db' AND active
GROUP BY column, type
ORDER BY sum(column_data_uncompressed_bytes) DESC;
```

### Check Mutations
```sql
SELECT database, table, mutation_id, command, create_time, is_done
FROM system.mutations
WHERE NOT is_done
ORDER BY create_time DESC;
```

---

## Additional Resources

- [ClickHouse Documentation](https://clickhouse.com/docs)
- [ClickHouse Best Practices](https://clickhouse.com/docs/best-practices)
- [ClickHouse Agent Skills](https://github.com/ClickHouse/agent-skills)
- [ClickHouse MCP Server](https://github.com/ClickHouse/mcp-clickhouse)
