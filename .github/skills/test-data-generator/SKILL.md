---
name: test-data-generator
description: Generates realistic-but-fake INSERT scripts for one or more tables, respecting data types, nullability, check constraints, and foreign key dependencies.
---

# Test Data Generator

Use this skill when the user asks to generate sample/test/dummy/seed data for a table
or set of related tables (e.g., "give me 50 test rows for `dbo.Customer`").

## Required MCP Tools
- `mssql_connect_database` / `mssql_check_database_context` — retrieve the active
  connection's `DatabaseConstitution` block (naming conventions and any
  "Weird" marker rule live here).
- `mssql_schema_get_objects_for_query_writing` — get columns, data types,
  nullability, and foreign key relationships for the target table(s).
- `mssql_execute_query` — read-only lookups (e.g., `sys.check_constraints`,
  existing parent rows for FK sampling) and, only after explicit confirmation,
  execution of the generated `INSERT` statements.
- `mssql_validate_syntax` — validate generated `INSERT` scripts before presenting
  or executing them.
- `get_file` (or the skill resource loader) — load
  `references/string-pools.md`, the embedded name/address/lorem reference
  data used for semantic string generation in Step 3.

## Step 1: Resolve Scope and Row Count
1. Identify the target table(s) from the user's request. If the user names a
   single table, check its foreign keys — if any FK column is `NOT NULL` or the
   user asks for "realistic" data, include parent tables in the generation order.
2. Determine row count: use what the user specifies, or default to 10 if unstated.
3. Build a dependency order (parents before children) using
   `sys.foreign_keys` / `sys.foreign_key_columns` so generated `INSERT`s won't
   violate FK constraints.

## Step 2: Inspect Column Metadata
For each table in dependency order, gather:
- Column name, data type, length/precision/scale, nullability, identity/computed status.
- Default constraints (skip generating a value if a default exists and the
  column isn't required to vary).
- `CHECK` constraints (`sys.check_constraints` + `definition`) — parse simple
  patterns (`IN (...)`, ranges, `LIKE`) to constrain generated values.
- Unique constraints/indexes — ensure generated values don't collide within the batch.
- Foreign keys — for each FK column, sample an existing value from the
  referenced table (via a lightweight `SELECT TOP (n) <PK> FROM ...`) or reuse
  a value just generated for a parent table in this same script.

## Step 3: Generate Values Per Data Type
Apply sensible, deterministic-but-varied generation rules:

| Data Type | Strategy |
|---|---|
| `int`/`bigint`/`smallint`/`tinyint` (non-identity) | Random within a plausible range, or 1..n |
| `varchar`/`nvarchar`/`char` | See **Semantic String Generation** below |
| `date`/`datetime`/`datetime2` | Random date within the last 1–3 years, or `GETDATE()` offset |
| `bit` | Random 0/1 |
| `decimal`/`numeric`/`money` | Random value respecting precision/scale |
| `uniqueidentifier` | `NEWID()` |
| identity columns | Omit from the column list entirely |
| computed columns | Omit from the column list entirely |

Respect `CHECK` constraint patterns discovered in Step 2 over the generic rule above.

### Semantic String Generation
No internet/web-access tools are available in this environment, so string
values are produced from a small, embedded reference dataset rather than live
external sources (e.g., online name books or address lists). Load
`references/string-pools.md` (via `get_file`/skill resource loader) once per
session and apply it as follows:

1. **Classify the column by name**, matching case-insensitively against
   patterns (substring match, not exact), in this priority order:

   | Column name contains | Reference pool used | Example output |
   |---|---|---|
   | `firstname`, `givenname`, `fname` | First Names | `Priya` |
   | `lastname`, `surname`, `lname` | Last Names | `Kowalski` |
   | `fullname`, `contactname` | First Names + " " + Last Names | `Liam Johnson` |
   | `street`, `address1`, `addressline` | Street Names + number + Suffixes | `482 Birch Ave` |
   | `city` | Cities | `Fairview` |
   | `state`, `province` | US States (respect column length: 2 vs. full name) | `CA` or `California` |
   | `zip`, `postalcode` | 5-digit numeric pattern, zero-padded | `04821` |
   | `email` | `{firstname}.{lastname}@{domain}` using First/Last Name + Domains pools | `mia.silva@example.com` |
   | `phone`, `fax` | `555-01##` pattern (fake-but-valid-looking, never a real exchange) | `555-0148` |
   | `company`, `vendor`, `employer` | Companies | `Initech` |
   | `product`, `item`, `sku`-adjacent text columns | Products / Generic Nouns | `Widget` |
   | `description`, `notes`, `comment`, `remarks` | Lorem Words (join 3–8 words) | `lorem ipsum dolor sit` |
   | `code`, `abbr`, `ref` (short, fixed-length) | Random alphanumeric token sized to column length | `A3X9` |
   | *(no match)* | Generic Lorem Words, truncated to column length | `lorem` |

2. **Respect column length/precision** — truncate or pad the chosen pool value
   to fit `(length)`; never generate a value that exceeds the column's
   declared size.

3. **Interactive fallback** — if a column name is ambiguous or ends up in the
   "no match" bucket for a column that looks business-critical (e.g., it's
   part of a unique index, FK, or the only non-nullable string column on the
   table), ask the user a brief clarifying question before generating
   (e.g., "Column `Notes` on `dbo.Customer` — any format expectations, or
   should I use generic lorem text?"). Do not block on columns that are
   clearly low-stakes (free-text comments, optional fields).

4. **Consistency across a batch** — once a first/last name pair is picked for
   a row, reuse that same pair when deriving dependent columns in the same
   row (e.g., `Email` should derive from that row's `FirstName`/`LastName`,
   not a fresh random pick).

5. Still apply the constitution's "Weird" marker rule (Step 4) after semantic
   generation — append or substitute the marker into the chosen value rather
   than overriding the whole semantic strategy.

## Step 4: Apply Constitution Rules
1. If the constitution defines naming conventions, ensure any generated script
   objects (none expected here, since this only produces `INSERT`s) would
   comply if created.
2. Prompt the users to determine if this "Weird" rule should be applied. If the answer confirms this, then it requires every generated record to contain the word
   "Weird" in at least one character/string column, inject that token into one
   `varchar`/`nvarchar`/`char` column per row (e.g., append `" (Weird)"` or use
   a `Weird`-prefixed synthetic value), choosing the first eligible
   non-constrained string column found.
3. If the constitution's persona requires stating a name in responses, do so
   once at the top of the output, not per row.

## Step 5: Output Format
1. Present one `INSERT INTO <schema>.<table> (<columns>) VALUES (...), (...), ...`
   batch per table, in dependency order (parents first), using the coding
   standards from the constitution if one is defined (uppercase keywords,
   one clause per line, aliased where applicable, trailing/leading commas per rule).
2. Wrap the full script in a single transaction:
   ```sql
   BEGIN TRY
	   BEGIN TRANSACTION;
	   -- generated INSERT statements
	   COMMIT TRANSACTION;
   END TRY
   BEGIN CATCH
	   IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
	   THROW;
   END CATCH;
   ```
3. Validate the script with `mssql_validate_syntax` before presenting it.
4. Do **not** execute the script automatically — present it to the user and
   ask for explicit confirmation before calling `mssql_execute_query`.

## Constraints
- This skill produces **data-modifying DML**. Per safety rules, NEVER execute
  the generated `INSERT` statements without explicit user confirmation.
- Never generate values that would violate `NOT NULL`, `CHECK`, `UNIQUE`, or
  `FOREIGN KEY` constraints discovered during inspection.
- Never fabricate column/type metadata — always derive it from
  `mssql_schema_get_objects_for_query_writing` or catalog views.
- If a table has no discoverable primary key or unique key and the user wants
  FK-safe child data, warn them that referential sampling may be unreliable.
- If the constitution's "Weird" marker rule exists but a table has no
  string/character column, state that the rule cannot be satisfied for that
  table and proceed without it.
