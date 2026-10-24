---
name: security-audit
description: Audits a database's security posture — principals, role memberships, permissions, orphaned users, weak authentication settings, and encryption/auditing configuration — and reports findings with remediation scripts.
---

# Database Security Audit

Use this skill when the user asks to audit, review, or assess the security of
a database or server (e.g., "run a security audit on this database",
"who has access to this database and what can they do?", "check for
excessive permissions").

## Required MCP Tools
- `mssql_connect_database` / `mssql_check_database_context` — confirm the
  active connection and retrieve any `DatabaseConstitution` content that
  might define expected ownership, support contacts, or persona.
- `mssql_execute_query` — read-only queries against security catalog views
  and DMVs (see Steps 1–6 below).
- `mssql_get_server_properties` — check authentication mode, version/edition,
  and server-level security-relevant configuration.
- `mssql_schema_get_objects_for_query_writing` — identify tables containing
  sensitive-looking columns (e.g., for data classification checks in Step 6).

## Step 1: Inventory Principals
Query database-level and server-level principals:
- `sys.database_principals` (users, roles, application roles) — exclude
  `type_desc = 'DATABASE_ROLE'` system roles unless reviewing role membership.
- `sys.server_principals` (logins) joined via `sys.server_role_members` for
  server-level role membership (e.g., `sysadmin`, `securityadmin`).
- Flag any principal with `is_disabled = 0` that has no corresponding login
  (orphaned users) — compare `sys.database_principals.sid` against
  `sys.server_principals.sid` where `type IN ('S','U')`.
- Note authentication type for each login (`SQL`, `Windows`, Microsoft Entra
  ID) and flag SQL logins (password-based) for extra scrutiny (e.g., check
  `LOGINPROPERTY(name, 'IsExpired')`, `'IsLocked'`, `'PasswordLastSetTime'`
  where supported by the platform).

## Step 2: Enumerate Role Memberships
For each database user, list role memberships via
`sys.database_role_members` joined to `sys.database_principals` (twice, for
role and member). Highlight:
- Any user directly in `db_owner`, `db_securityadmin`, or `db_ddladmin` who
  isn't clearly expected to have that level of access (judge based on naming
  — e.g., an `ETL` or `reporting` account in `db_owner` is suspicious).
- Any server login in `sysadmin`/`securityadmin`/`serveradmin` server roles.

## Step 3: Enumerate Explicit Permissions
Query `sys.database_permissions` joined to `sys.database_principals` and
`OBJECT_NAME()`/`sys.objects` to list explicit `GRANT`/`DENY` statements at
the object, schema, and database level. Flag:
- Broad grants (`CONTROL`, `ALTER`, `ALTER ANY ...`) to non-admin-looking accounts.
- `GRANT ... WITH GRANT OPTION` usage (allows privilege escalation by the grantee).
- Permissions granted directly to individual users instead of via roles
  (a common anti-pattern worth calling out, even if not inherently wrong).
- Any `DENY` statements, noting they override grants and can mask access
  issues — confirm they're intentional.

## Step 4: Check Authentication & Server-Level Configuration
Via `mssql_get_server_properties` and targeted queries:
- Server authentication mode (Windows-only vs. Mixed Mode) — Mixed Mode
  increases SQL login attack surface; note if the `sa` account is enabled
  and/or has a non-default/weak-looking name check is not possible, but flag
  if `sa` is enabled (`SELECT is_disabled FROM sys.server_principals WHERE
  name = 'sa'`).
- `sys.configurations` for risky `sp_configure` options left enabled (e.g.,
  `xp_cmdshell`, `Ole Automation Procedures`, `remote access`).
- Whether guest access is revoked in user databases
  (`SELECT permission_name FROM sys.database_permissions WHERE grantee_principal_id
  = DATABASE_PRINCIPAL_ID('guest')` — `guest` should typically have no
  `CONNECT` permission except in `master`/`tempdb`).

## Step 5: Check Encryption & Auditing
- Transparent Data Encryption status via `sys.dm_database_encryption_keys`.
- Always Encrypted columns via `sys.column_encryption_keys` /
  `sys.columns.encryption_type` (if any sensitive-looking columns from Step 6
  are not encrypted, note it as a gap rather than a hard finding).
- Connection encryption requirement (`force_encryption` server property).
- SQL Server Audit / server audit specifications via `sys.server_audits` and
  `sys.server_audit_specifications` (or Azure SQL Database auditing settings
  if applicable) — note if no audit is configured.
- Query Store / login auditing settings as supplementary evidence of
  monitoring capability, not a substitute for a real audit trail.

## Step 6: Flag Sensitive Data Exposure (Best Effort)
Using `mssql_schema_get_objects_for_query_writing`, scan column names for
patterns suggesting sensitive data (`SSN`, `Password`, `CreditCard`, `DOB`,
`Email`, `Salary`, `TaxID`, etc.). Cross-reference with permission findings
from Step 3 — flag cases where broad roles/users have read access to tables
containing these columns. This is a heuristic pass, not a certified data
classification; state that explicitly in the report.

## Step 7: Present Findings
Produce a structured report:

1. **Summary** — overall risk posture in a few sentences (e.g., "3 high,
   5 medium findings").
2. **Findings table**, one row per issue:

   | Severity | Area | Finding | Recommendation |
   |----------|------|---------|-----------------|
   | High | Role Membership | `EtlService` is a member of `db_owner` | Scope to `db_datareader`/`db_datawriter` plus explicit object grants |
   | Medium | Authentication | `sa` login is enabled | Disable `sa` and use named administrative logins |

3. **Remediation scripts** — generate `REVOKE`/`ALTER SERVER ROLE ...
   DROP MEMBER`/`DISABLE`-style scripts for each finding, grouped and clearly
   labeled, but do **not** execute them.
4. **Orphaned users** list, with the standard `sp_change_users_login` or
   `ALTER USER ... WITH LOGIN =` remediation pattern, not executed.

## Constraints
- This is a **read-only audit**. Do not execute any permission-altering,
  login-altering, or configuration-altering statement
  (`GRANT`/`REVOKE`/`DENY`/`ALTER LOGIN`/`ALTER USER`/`sp_configure`/
  `RECONFIGURE`) without explicit user confirmation, per the safety rules —
  these are security-altering statements and require confirmation every time,
  not just once per session.
- Do not expose login passwords, connection strings, or secrets in the report.
- Do not fabricate findings — if a DMV/catalog view returns no rows (e.g., no
  audit configured, no TDE keys), report that plainly as the finding itself.
- If the constitution defines a persona, honor it in the response voice.
- If the platform is Azure SQL Database/Managed Instance, note that some
  server-level concepts (e.g., `sysadmin`, `xp_cmdshell`) don't apply and
  substitute the Azure-equivalent concepts (Microsoft Entra ID admin,
  firewall rules, Advanced Threat Protection) where relevant.
