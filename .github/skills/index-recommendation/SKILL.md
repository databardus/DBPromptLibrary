---
name: index-recommendation
description: Analyzes a specific table's structure, data volume, missing-index DMVs, and query/usage patterns to recommend and script new indexes.
---

# Index Recommendation

Use this skill when the user asks to analyze a table and recommend indexes
(e.g., "give me index recommendations for `dbo.SalesOrders`").

## Required MCP Tools
- `mssql_connect_database` / `mssql_check_database_context` — confirm the
  active connection and retrieve any `DatabaseConstitution` naming
  conventions to apply to recommended index names.
- `mssql_schema_get_objects_for_query_writing` — get the table's columns,
  data types, nullability, existing indexes, and foreign key relationships.
- `mssql_execute_query` — read-only queries against DMVs and catalog views
  (see Step 2–4 below).
- `mssql_get_server_properties` — check server/edition version to confirm
  availability of features like columnstore or filtered indexes if relevant.
- `mssql_validate_syntax` — validate generated `CREATE INDEX` scripts.

## Step 1: Gather Table Structure
1. Resolve the target table (schema + name) from the user's request; ask if
   either is missing.
2. Retrieve column details (name, data type, length/precision, nullability),
   primary key, existing indexes (columns, included columns, uniqueness,
   filtered definition), and foreign key constraints via
   `mssql_schema_get_objects_for_query_writing` and/or
   `sys.indexes`/`sys.index_columns`/`sys.columns`.

## Step 2: Assess Data Volume
Query row count and storage size for the table, e.g. via
`sys.dm_db_partition_stats` or `sp_spaceused`-equivalent catalog queries:

```sql
SELECT
	  OBJECT_SCHEMA_NAME(p.object_id) AS SchemaName
	, OBJECT_NAME(p.object_id) AS TableName
	, SUM(p.rows) AS RowCount
	, SUM(a.total_pages) * 8 / 1024.0 AS SizeMB
FROM sys.partitions AS p
INNER JOIN sys.allocation_units AS a
	ON a.container_id = p.partition_id
WHERE
	p.object_id = OBJECT_ID(N'<schema>.<table>')
	AND p.index_id IN (0, 1)
GROUP BY
	  p.object_id
```

Note whether the table is small enough that index recommendations would have
negligible benefit (call this out to the user rather than recommending
indexes on trivially small tables).

## Step 3: Check Missing Index DMVs
Query `sys.dm_db_missing_index_details`, `sys.dm_db_missing_index_groups`, and
`sys.dm_db_missing_index_group_stats`, filtered to the target table's
`object_id`, to surface SQL Server's own missing-index suggestions
(equality columns, inequality columns, included columns, and the
`user_seeks`/`user_scans`/`avg_total_user_cost`/`avg_user_impact` stats that
justify each one). Rank by estimated impact
(`avg_total_user_cost * avg_user_impact * (user_seeks + user_scans)`).

## Step 4: Check Usage / Query Patterns
1. If Query Store is enabled (check via `sys.database_query_store_options`),
   query `sys.query_store_query` / `sys.query_store_plan` /
   `sys.query_store_runtime_stats` joined to `sys.query_store_query_text` to
   find frequent or high-cost queries referencing the table (text search on
   `query_sql_text` for the table name is a reasonable first pass).
2. If Query Store is not enabled, fall back to
   `sys.dm_exec_query_stats`/`sys.dm_exec_sql_text`/`sys.dm_exec_cached_plans`
   for similar insight, noting these are cache-dependent and may not reflect
   historical patterns.
3. Review existing index definitions from Step 1 for columns already
   well-covered, to avoid recommending redundant/overlapping indexes.

## Step 5: Formulate Recommendations
For each recommended index, state:
- The columns (key columns in order, plus any `INCLUDE` columns).
- Whether it should be unique, filtered, or a covering index.
- The supporting evidence (which missing-index DMV entry and/or which
  observed query pattern it addresses).
- The estimated benefit/tradeoff (read performance gain vs. write/maintenance
  overhead, and storage cost using the size figures from Step 2).

Generate the corresponding `CREATE INDEX` script(s), naming each index per the
database's naming conventions if the constitution defines one for indexes,
otherwise use a clear convention like `IX_<Table>_<Col1>_<Col2>`. Validate
each script with `mssql_validate_syntax`.

## Step 6: Present Results
Summarize in this order:
1. Table structure and data volume summary.
2. Missing-index DMV findings (if any).
3. Observed query/usage patterns (if any evidence was found).
4. Ranked index recommendations with scripts and rationale.

If no missing-index DMV data and no usable Query Store/cache data exists,
state this explicitly and recommend revisiting after the workload has run
long enough to populate these DMVs, rather than inventing indexes with no evidence.

## Constraints
- This is a **read-only analysis** task. Do not execute `CREATE INDEX`
  statements without explicit user confirmation, per the safety rules.
- Do not fabricate DMV data or query patterns — if a DMV returns no rows,
  report that plainly.
- If the constitution defines a persona, honor it in the response voice.
