---
name: coding-standards-compliance
description: Audits stored procedures, views, and functions against the coding standards defined in the database constitution, flagging the object, line number, and rule violated.
---

# Coding Standards Compliance Audit

Use this skill when the user asks to audit, review, or check database code objects
(stored procedures, views, functions, triggers) against the coding standards defined
in the database's constitution.

## Required MCP Tools
- `mssql_connect_database` / `mssql_check_database_context` — retrieve the active
  connection's `DatabaseConstitution` block (this is where coding standards live).
- `mssql_schema_get_objects_for_query_writing` — enumerate procedures, views, functions.
- `mssql_execute_query` — pull full object definitions via `sys.sql_modules` /
  `OBJECT_DEFINITION()` (read-only, safe).
- `mssql_validate_syntax` — optional, to confirm a rewritten object is valid.

## Step 1: Extract the Coding Standards
1. Read the `DatabaseConstitution` array returned by the active connection context
   (from `mssql_connect_database` or `mssql_check_database_context`).
2. Locate the `## Coding Standards` section and the `## Naming conventions` section.
3. Parse each numbered/bulleted rule into a discrete, testable rule with a short ID
   (e.g., `CS1` = "SQL keywords in all caps", `NC1` = "Tables PascalCase singular").
4. If the constitution defines a **Persona** section, honor it in your response voice,
   but do NOT let persona rules (favorite numbers, fun facts) count as coding standards.
5. If no `## Coding Standards` or `## Naming conventions` section exists, stop and tell
   the user there is nothing to audit against.

## Step 2: Enumerate Code Objects
Query `sys.sql_modules` joined to `sys.objects` for type IN ('P','V','FN','IF','TF','TR'):

Skip system objects (`is_ms_shipped = 0`).

## Step 3: Evaluate Each Object Against Each Rule
For every object definition, split into lines (1-based line numbers matching the
`definition` text as returned). For each parsed rule, scan line-by-line and flag violations:

- **Keyword casing** — any T-SQL reserved keyword not fully uppercase (e.g. `select`, `From`, `Where`).
- **Clause line breaks** — keywords like `SELECT`, `FROM`, `WHERE`, `JOIN`, `GROUP BY`,
  `ORDER BY`, `HAVING` not starting a new line.
- **Column/predicate indentation** — lines listing columns or predicates under a clause
  that are not indented relative to the clause keyword.
- **Column name casing mismatch** — aliasing or referencing a column with different
  case/format than its catalog definition (compare against `sys.columns`).
- **Missing table aliases** — multi-table queries (`JOIN` present) where a table has no
  alias, or column references are unqualified.
- **Comma placement** — leading vs. trailing commas, per the rule's direction.
- **Naming convention violations** — object name doesn't match the required prefix/case
  pattern for its type (e.g., stored procedure not matching `usvenp_<Action><Entity>`,
  view not matching `usvenv_<Description>`, function not matching `usvenfn_<Description>`).
- **PK/FK naming** — inspect `sys.columns`/`sys.foreign_keys` for `<TableName>ID` /
  `<ReferencedTable>ID` patterns when auditing table DDL.

Record every violation with: object (schema.name), approximate line number within the
object definition, the rule ID violated, and a one-line description of the rule text.

## Step 4: Output Format
Produce a markdown table, one row per violation:

| Object | Line | Rule Violated | Description |
|--------|------|----------------|-------------|
| dbo.GetOrders | 4 | CS1 | SQL keyword `select` is not uppercase |
| dbo.GetOrders | 12 | NC3 | Procedure name does not match `usvenp_<Action><Entity>` pattern |

If an object has zero violations, omit it from the table but include it in a one-line
"Compliant objects" summary list at the end.

## Constraints
- This is a **read-only audit**. Do NOT modify any object (`ALTER`/`CREATE`) unless the
  user explicitly asks for a fix — and even then, confirm before executing DDL.
- Do not fabricate line numbers — derive them by counting newlines in the actual
  `definition` text returned from `sys.sql_modules`.
- If the constitution's persona requires stating a name in responses, do so once at
  the top of the audit report, not per violation row.