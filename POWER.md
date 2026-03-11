---
name: "clickhouse"
displayName: "ClickHouse Database"
description: "Query, explore, and manage ClickHouse databases with best-practice guidance for schema design, query optimization, and data ingestion"
keywords: ["clickhouse", "database", "analytics", "olap", "sql", "columnar", "data", "query"]
author: "ClickHouse"
---

# Onboarding

Before proceeding, let the user know that the ClickHouse Cloud Remote MCP Server must be enabled on their ClickHouse Cloud service. They can enable it from the **Connect** view in the ClickHouse Cloud console. See [Enable MCP on ClickHouse](https://clickhouse.com/docs/use-cases/AI/MCP/remote_mcp).

Authentication is handled via OAuth — the user will be prompted to authenticate in their browser on first use.

For self-hosted ClickHouse, users should use the open-source MCP server (`mcp-clickhouse` via pip/uv) instead. See the Configuration section below.

# Overview

The ClickHouse Database Power provides access to ClickHouse for running SQL queries, exploring databases and tables, and inspecting schemas. It also includes comprehensive best-practice guidance for schema design, query optimization, and data ingestion — adapted from the official [ClickHouse Agent Skills](https://github.com/ClickHouse/agent-skills).

**Key capabilities:**
- **SQL Queries**: Execute read-only SQL queries against ClickHouse clusters
- **Database Discovery**: List databases and browse table schemas
- **Table Inspection**: View table structures, engines, ORDER BY keys, and column types
- **Best Practices**: 28 rules covering schema design, query optimization, and insert strategy
- **Schema Reviews**: Validate CREATE TABLE statements against ClickHouse best practices
- **Query Optimization**: JOIN algorithms, skipping indices, materialized views guidance

**Perfect for:**
- Exploring and querying ClickHouse databases
- Designing new tables with proper ORDER BY, partitioning, and data types
- Optimizing slow queries (JOINs, filters, indices)
- Building data ingestion pipelines with proper batching
- Reviewing schemas and queries against best practices

## Available Steering Files

This power has the following steering files:
- **steering** — Complete ClickHouse best-practices guide with 28 rules for schema design, query optimization, and insert strategy

## Available MCP Servers

### clickhouse
**Endpoint:** `https://mcp.clickhouse.cloud/mcp`
**Connection:** Remote MCP server via OAuth (ClickHouse Cloud)

**Tools:**

1. **run_query** — Execute SQL queries on your ClickHouse cluster
   - Required: `query` (string) — The SQL query to execute
   - Queries run in read-only mode by default
   - Returns: Query results with rows and metadata

2. **list_databases** — List all databases on your ClickHouse cluster
   - No parameters required
   - Returns: Array of database names

3. **list_tables** — List tables in a database with pagination
   - Required: `database` (string) — Database name
   - Optional: `like` (string) — LIKE filter on table names
   - Optional: `not_like` (string) — NOT LIKE filter on table names
   - Optional: `page_token` (string) — Pagination token from previous call
   - Optional: `page_size` (int, default 50) — Number of tables per page
   - Optional: `include_detailed_columns` (bool, default true) — Include column metadata
   - Returns: Tables with schemas, next_page_token, total_tables


## Tool Usage Examples

### Listing Databases

```javascript
usePower("clickhouse", "clickhouse", "list_databases", {})
// Returns: ["default", "system", "my_analytics", ...]
```

### Listing Tables

**List all tables in a database:**
```javascript
usePower("clickhouse", "clickhouse", "list_tables", {
  "database": "default"
})
// Returns: Tables with schemas, engines, ORDER BY keys
```

**Filter tables by name pattern:**
```javascript
usePower("clickhouse", "clickhouse", "list_tables", {
  "database": "default",
  "like": "%events%"
})
// Returns: Only tables matching the pattern
```

### Running Queries

**Simple SELECT:**
```javascript
usePower("clickhouse", "clickhouse", "run_query", {
  "query": "SELECT count() FROM events WHERE timestamp > now() - INTERVAL 1 HOUR"
})
// Returns: Row count
```

**Inspect table schema:**
```javascript
usePower("clickhouse", "clickhouse", "run_query", {
  "query": "DESCRIBE TABLE events"
})
// Returns: Column names, types, defaults, and comments
```

**Check table engine and ORDER BY:**
```javascript
usePower("clickhouse", "clickhouse", "run_query", {
  "query": "SELECT engine, sorting_key, partition_key FROM system.tables WHERE name = 'events' AND database = 'default'"
})
// Returns: Engine type, sorting key, partition key
```

**Analyze query performance:**
```javascript
usePower("clickhouse", "clickhouse", "run_query", {
  "query": "SELECT event_type, count() as cnt, avg(duration) as avg_duration FROM events WHERE timestamp > now() - INTERVAL 24 HOUR GROUP BY event_type ORDER BY cnt DESC LIMIT 20"
})
// Returns: Event type breakdown with counts and average duration
```

**Check part health:**
```javascript
usePower("clickhouse", "clickhouse", "run_query", {
  "query": "SELECT partition, count() as parts, sum(rows) as rows, formatReadableSize(sum(bytes_on_disk)) as size FROM system.parts WHERE table = 'events' AND active GROUP BY partition ORDER BY partition"
})
// Returns: Partition health with part counts and sizes
```

## Combining Tools (Workflows)

### Workflow 1: Database Exploration

```javascript
// Step 1: List all databases
const databases = usePower("clickhouse", "clickhouse", "list_databases", {})

// Step 2: List tables in a database
const tables = usePower("clickhouse", "clickhouse", "list_tables", {
  "database": "analytics"
})

// Step 3: Inspect a specific table
const schema = usePower("clickhouse", "clickhouse", "run_query", {
  "query": "DESCRIBE TABLE analytics.events"
})

// Step 4: Check table engine and keys
const tableInfo = usePower("clickhouse", "clickhouse", "run_query", {
  "query": "SELECT engine, sorting_key, partition_key, total_rows, formatReadableSize(total_bytes) as size FROM system.tables WHERE database = 'analytics' AND name = 'events'"
})

// Step 5: Sample data
const sample = usePower("clickhouse", "clickhouse", "run_query", {
  "query": "SELECT * FROM analytics.events LIMIT 10"
})
```

### Workflow 2: Query Performance Investigation

```javascript
// Step 1: Find slow queries in the query log
const slowQueries = usePower("clickhouse", "clickhouse", "run_query", {
  "query": "SELECT query, query_duration_ms, read_rows, read_bytes, result_rows FROM system.query_log WHERE type = 'QueryFinish' AND query_duration_ms > 1000 AND event_time > now() - INTERVAL 1 HOUR ORDER BY query_duration_ms DESC LIMIT 10"
})

// Step 2: Check if filters align with ORDER BY
const tableKeys = usePower("clickhouse", "clickhouse", "run_query", {
  "query": "SELECT sorting_key FROM system.tables WHERE database = 'analytics' AND name = 'events'"
})

// Step 3: Check index usage with EXPLAIN
const explain = usePower("clickhouse", "clickhouse", "run_query", {
  "query": "EXPLAIN indexes = 1 SELECT * FROM analytics.events WHERE user_id = 12345"
})

// Step 4: Check part count health
const partHealth = usePower("clickhouse", "clickhouse", "run_query", {
  "query": "SELECT table, count() as parts, sum(rows) as total_rows FROM system.parts WHERE active AND database = 'analytics' GROUP BY table ORDER BY parts DESC"
})
```

### Workflow 3: Schema Review

```javascript
// Step 1: Get the CREATE TABLE statement
const createTable = usePower("clickhouse", "clickhouse", "run_query", {
  "query": "SHOW CREATE TABLE analytics.events"
})

// Step 2: Check column cardinality for ORDER BY optimization
const cardinality = usePower("clickhouse", "clickhouse", "run_query", {
  "query": "SELECT uniq(event_type) as event_type_uniq, uniq(user_id) as user_id_uniq, uniq(toDate(timestamp)) as date_uniq FROM analytics.events"
})

// Step 3: Check partition count
const partitions = usePower("clickhouse", "clickhouse", "run_query", {
  "query": "SELECT partition, count() as parts, sum(rows) as rows, formatReadableSize(sum(bytes_on_disk)) as size FROM system.parts WHERE table = 'events' AND database = 'analytics' AND active GROUP BY partition ORDER BY partition"
})

// Step 4: Verify data types are optimal
const columnSizes = usePower("clickhouse", "clickhouse", "run_query", {
  "query": "SELECT column, type, formatReadableSize(sum(column_data_compressed_bytes)) as compressed, formatReadableSize(sum(column_data_uncompressed_bytes)) as uncompressed, round(sum(column_data_uncompressed_bytes) / sum(column_data_compressed_bytes), 2) as ratio FROM system.parts_columns WHERE table = 'events' AND database = 'analytics' AND active GROUP BY column, type ORDER BY sum(column_data_uncompressed_bytes) DESC"
})
```

## Best Practices

### ✅ Do:

- **Plan ORDER BY before table creation** — it's immutable after creation
- **Order columns low-to-high cardinality** in ORDER BY for effective granule skipping
- **Use native types** (UInt32, DateTime, UUID) instead of String for everything
- **Use LowCardinality(String)** for columns with fewer than 10K unique values
- **Batch inserts 10K–100K rows** per INSERT to avoid part explosion
- **Filter on ORDER BY prefix columns** in queries for index usage
- **Use ReplacingMergeTree** instead of ALTER TABLE UPDATE for frequent updates
- **Start without partitioning** and add it later for data lifecycle needs
- **Keep partition cardinality low** (100–1,000 values)
- **Use skipping indices** (bloom_filter, set, minmax) for non-ORDER BY filter columns

### ❌ Don't:

- **Use Nullable everywhere** — use DEFAULT values instead (Nullable adds storage overhead)
- **Use ALTER TABLE UPDATE/DELETE** for frequent changes — use specialized MergeTree engines
- **Run OPTIMIZE TABLE FINAL** regularly — let background merges work
- **Use String for everything** — native types give 2–10x storage reduction
- **Put high-cardinality columns first** in ORDER BY — prevents index pruning
- **Use very high partition cardinality** — causes "too many parts" errors
- **Insert single rows** — each INSERT creates a data part
- **Use wildcards inside quotes** in queries — they become literal
- **Skip query analysis** before choosing ORDER BY — wrong choice requires full migration

## Troubleshooting

### Error: "Too many parts"
**Cause:** Too many small inserts or high partition cardinality
**Solution:**
1. Check part count: `SELECT table, count() as parts FROM system.parts WHERE active GROUP BY table ORDER BY parts DESC`
2. Increase batch size to 10K–100K rows per INSERT
3. Enable async inserts for high-frequency small batches
4. Reduce partition cardinality (aim for 100–1,000 partitions)

### Error: "Memory limit exceeded"
**Cause:** Query processing too much data or wrong JOIN algorithm
**Solution:**
1. Add filters to reduce data scanned
2. Use `partial_merge` or `grace_hash` JOIN algorithm for large tables
3. Ensure smaller table is on the RIGHT side of JOIN (pre-24.12)
4. Add `LIMIT` to restrict output

### Error: "Query is too slow"
**Cause:** Filters not aligned with ORDER BY or missing indices
**Solution:**
1. Check ORDER BY: `SELECT sorting_key FROM system.tables WHERE name = 'your_table'`
2. Verify filters use ORDER BY prefix: `EXPLAIN indexes = 1 SELECT ...`
3. Add skipping indices for non-ORDER BY filter columns
4. Consider materialized views for pre-aggregation

### Error: "Cannot modify ORDER BY"
**Cause:** ORDER BY is immutable after table creation
**Solution:**
1. Create a new table with the correct ORDER BY
2. Migrate data: `INSERT INTO new_table SELECT * FROM old_table`
3. Rename tables: `RENAME TABLE old_table TO old_table_backup, new_table TO old_table`

### Error: "Authentication failed" (Cloud MCP)
**Cause:** OAuth session expired or MCP not enabled
**Solution:**
1. Re-authenticate via browser when prompted
2. Verify MCP is enabled in ClickHouse Cloud console (Connect view)
3. Check that the service is running

## Configuration

### ClickHouse Cloud (Remote MCP — Default)

Authentication is handled via OAuth. No API keys needed.

**Setup Steps:**
1. Log in to ClickHouse Cloud console
2. Click **Connect** on your service
3. Enable **Remote MCP Server**
4. The power uses `https://mcp.clickhouse.cloud/mcp` automatically
5. Authenticate via browser when prompted on first use

### Self-Hosted ClickHouse (Local MCP)

For self-hosted ClickHouse, replace the mcp.json config with the open-source MCP server:

```json
{
  "mcpServers": {
    "clickhouse": {
      "command": "uv",
      "args": [
        "run",
        "--with",
        "mcp-clickhouse",
        "--python",
        "3.10",
        "mcp-clickhouse"
      ],
      "env": {
        "CLICKHOUSE_HOST": "<your-clickhouse-host>",
        "CLICKHOUSE_PORT": "8443",
        "CLICKHOUSE_USER": "<your-user>",
        "CLICKHOUSE_PASSWORD": "<your-password>",
        "CLICKHOUSE_SECURE": "true",
        "CLICKHOUSE_VERIFY": "true",
        "CLICKHOUSE_CONNECT_TIMEOUT": "30",
        "CLICKHOUSE_SEND_RECEIVE_TIMEOUT": "300"
      }
    }
  }
}
```

**Required environment variables:**
- `CLICKHOUSE_HOST` — Hostname of your ClickHouse server
- `CLICKHOUSE_USER` — Username for authentication
- `CLICKHOUSE_PASSWORD` — Password for authentication

**Optional environment variables:**
- `CLICKHOUSE_PORT` — Port (default: 8443 for HTTPS, 8123 for HTTP)
- `CLICKHOUSE_SECURE` — Enable HTTPS (default: "true")
- `CLICKHOUSE_DATABASE` — Default database
- `CLICKHOUSE_ALLOW_WRITE_ACCESS` — Allow DDL/DML (default: "false")

### ClickHouse SQL Playground (Try It Out)

To test with the public ClickHouse SQL Playground:

```json
{
  "mcpServers": {
    "clickhouse": {
      "command": "uv",
      "args": [
        "run",
        "--with",
        "mcp-clickhouse",
        "--python",
        "3.10",
        "mcp-clickhouse"
      ],
      "env": {
        "CLICKHOUSE_HOST": "sql-clickhouse.clickhouse.com",
        "CLICKHOUSE_PORT": "8443",
        "CLICKHOUSE_USER": "demo",
        "CLICKHOUSE_PASSWORD": "",
        "CLICKHOUSE_SECURE": "true",
        "CLICKHOUSE_VERIFY": "true"
      }
    }
  }
}
```

## Tips

1. **Plan ORDER BY first** — analyze your top query patterns before creating tables
2. **Use the steering file** — comprehensive best-practices guide with 28 rules
3. **Check cardinality** — run `SELECT uniq(column) FROM table` before choosing ORDER BY order
4. **Use EXPLAIN** — `EXPLAIN indexes = 1 SELECT ...` shows whether the primary index is used
5. **Monitor parts** — query `system.parts` to catch part explosion early
6. **Leverage system tables** — `system.query_log`, `system.parts`, `system.tables` are your friends
7. **Batch inserts** — aim for 10K–100K rows per INSERT
8. **Use LowCardinality** — for string columns with fewer than 10K unique values
9. **Prefer ReplacingMergeTree** — over ALTER TABLE UPDATE for mutable data
10. **Start simple** — no partitioning initially, add it when you have clear lifecycle needs

---

## Legal & Support

### Power

- **License:** [Apache-2.0](https://spdx.org/licenses/Apache-2.0.html)
- **Privacy Policy:** [ClickHouse Privacy Policy](https://clickhouse.com/legal/privacy-policy)
- **Support:** [coding.aurora@gmail.com](mailto:coding.aurora@gmail.com) | [GitHub Issues](https://github.com/ClickHouse/mcp-clickhouse/issues)

### MCP Server — ClickHouse Cloud Remote (`mcp.clickhouse.cloud`)

- **License:** [Apache-2.0](https://spdx.org/licenses/Apache-2.0.html) ([source](https://github.com/ClickHouse/mcp-clickhouse/blob/main/LICENSE))
- **Privacy Policy:** [ClickHouse Privacy Policy](https://clickhouse.com/legal/privacy-policy)
- **Support:** [coding.aurora@gmail.com](mailto:coding.aurora@gmail.com) | [ClickHouse Support](https://clickhouse.com/support) | [GitHub Issues](https://github.com/ClickHouse/mcp-clickhouse/issues)

### MCP Server — Self-Hosted (`mcp-clickhouse` via pip/uv)

- **License:** [Apache-2.0](https://spdx.org/licenses/Apache-2.0.html) ([source](https://github.com/ClickHouse/mcp-clickhouse/blob/main/LICENSE))
- **Privacy Policy:** [ClickHouse Privacy Policy](https://clickhouse.com/legal/privacy-policy)
- **Support:** [coding.aurora@gmail.com](mailto:coding.aurora@gmail.com) | [GitHub Issues](https://github.com/ClickHouse/mcp-clickhouse/issues)

### Best Practices Content

- **Source:** [ClickHouse Agent Skills](https://github.com/ClickHouse/agent-skills)
- **License:** [Apache-2.0](https://spdx.org/licenses/Apache-2.0.html) ([source](https://github.com/ClickHouse/agent-skills/blob/main/LICENSE))
