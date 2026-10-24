---
name: etl-pattern-replication
description: Replicates an existing ETL stored procedure pattern (source -> staging -> dimension/fact) for a new source table, creating any missing staging/target tables, surrogate keys, FK lookups, and SCD logic to match the template procedure.
---

# ETL Pattern Replication

Use this skill when the user asks to build a new ETL stored procedure (and any
supporting staging/dimension/fact tables) by following the pattern of an
existing, working stored procedure in the database (e.g., "using
`dbo.usp_ProcessEmployeeData` as a template, load `dbo.Invoices` into
`stg.Invoices` and `fact.Invoices`").

## Required MCP Tools
- `mssql_connect_database` / `mssql_check_database_context` — retrieve the
  active connection's `DatabaseConstitution` block (naming conventions,
  coding standards, and persona all live here and must be honored in any
  generated DDL/T-SQL).
- `mssql_schema_get_objects_for_query_writing` — enumerate columns, data
  types, nullability, and foreign key relationships for the source table,
  any existing staging/target tables, and the template procedure's tables.
- `mssql_execute_query` — read `sys.sql_modules`/`OBJECT_DEFINITION()` for the
  template procedure; look up actual names of `ErrorLog`/`BatchLog` tables;
  check for existing rows when sampling FK lookups; and, only after explicit
  user confirmation, execute generated DDL/DML.
- `mssql_validate_syntax` — validate every generated `CREATE TABLE` and
  `CREATE OR ALTER PROCEDURE` statement before presenting or executing it.

## Step 1: Identify the Template and Parameters
Gather, from the user's request or by asking if missing:
- **Source table** (with schema), e.g. `dbo.Invoices`
- **Staging table** target name (with schema), e.g. `stg.Invoices`
- **Final table** target name (with schema), e.g. `fact.Invoices` or `dim.Invoices`
- **Template stored procedure** name (with schema) to copy the pattern from
- **Target table type**: fact or dimension. If dimension, ask/confirm whether
  it should be SCD Type 2 (look for `EffectiveStartDate`/`EffectiveEndDate`/
  `IsCurrent`-style columns on the template's dimension table as a clue).

## Step 2: Study the Template
1. Pull the template procedure's full definition via `sys.sql_modules`.
2. Pull the schema (columns, types, nullability, identity, FK constraints) of
   the template's source, staging, and dimension/fact tables so the pattern
   can be compared structurally, not just read as text.
3. Identify and note the following reusable elements in the template:
   - How staging table columns map from the source table (1:1, renamed, or
	 widened for key lookups).
   - How natural keys are replaced with surrogate keys (subquery lookup
	 pattern, including how it resolves SCD Type 2 versioning by date, if
	 applicable).
   - Identity column handling (never insert explicit values into identity columns).
   - Error handling (`TRY`/`CATCH`, error logging table and its actual name/columns).
   - Batch/run tracking logic (batch logging table and its actual name/columns).
   - Any metadata columns maintained (e.g., `CreatedDate`, `ETLBatchID`) and
	 only replicate ones that exist in the template's actual tables.

## Step 3: Build Missing Tables
Check whether the staging and final tables already exist
(`sys.tables`/`INFORMATION_SCHEMA.TABLES`). For any that don't exist, generate
`CREATE TABLE` scripts that:
- Mirror the source table's columns in the staging table, adding extra
  columns needed to hold resolved surrogate keys (per the template's pattern).
- In the final table, add a surrogate key column (identity) while retaining
  the natural key from the source.
- Rename any column that is a foreign key reference, replacing `ID` with `Key`
  per the database's naming conventions (check the constitution for the exact
  convention — do not assume if a convention is stated).
- If the final table is SCD Type 2, add effective/expiration/current-flag
  columns matching the template's dimension table.
- Only add the metadata columns that are actually present in the template's
  corresponding tables — do not invent additional ones.
- Honor the coding standards and naming conventions from the `DatabaseConstitution`
  for all generated DDL.

Present these `CREATE TABLE` scripts to the user and confirm before executing
(per safety rules — DDL always requires explicit confirmation).

## Step 4: Build the Stored Procedure
Generate a new stored procedure, following the exact structural pattern of
the template:
1. Load source data into the staging table (straight column mapping plus any
   transformation the template applies, such as trimming or default-filling).
2. From staging, resolve each foreign-key/natural-key column to its surrogate
   key via a subquery lookup against the referenced dimension table:
   - If the referenced dimension is SCD Type 2 (has effective/expiration
	 columns), the lookup must filter using the relevant date column from the
	 source record (e.g., `InvoiceDate BETWEEN EffectiveStartDate AND
	 EffectiveEndDate`) to find the correct historical version.
   - If the referenced dimension is not versioned, a simple natural-key match is sufficient.
3. Insert/merge into the final table, omitting the surrogate/identity column
   from the explicit column list.
4. If the final table is a fact table, omit any SCD Type 2 evolution logic
   (that only applies to dimension loads).
5. Replicate the template's error handling (`TRY`/`CATCH`, error logging) and
   batch tracking logic exactly, using the actual table/column names
   confirmed in Step 2 — do not guess names.
6. Apply the database's coding standards (keyword casing, clause line breaks,
   indentation, comma placement, aliasing) from the constitution to the
   generated procedure body.
7. Name the procedure per the constitution's naming convention for stored
   procedures.

Validate the generated procedure with `mssql_validate_syntax` before
presenting it.

## Step 5: Execute (Only on Explicit Request)
If the user asks you to run the generated scripts against the database:
1. Execute the `CREATE TABLE` statements first (after confirmation), then the
   procedure creation, then optionally execute the procedure itself.
2. If any statement errors, report the exact error message and number, explain
   the likely cause, and propose a corrected script rather than silently
   retrying with guesses.
3. Never insert explicit values into identity columns.

## Constraints
- Do not fabricate table/column names for `ErrorLog`, `BatchLog`, or any other
  supporting object — always confirm the actual name via the database.
- DDL and any data-modifying execution require explicit user confirmation
  per the safety rules, even if the user's original request implied it should
  "just be done."
- If the constitution defines a persona, honor it in the response voice.
