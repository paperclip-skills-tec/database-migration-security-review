---
name: database-migration-security-review
description: "Structured security review checklist for database migrations that touch grants, revokes, RLS policies, role permissions, schema creation, or row-level access controls. Use this skill whenever you are reviewing a migration file for security implications — especially when it creates tables, schemas, or roles; modifies GRANT/REVOKE chains; adds or changes Row-Level Security policies; introduces SECURITY DEFINER functions or triggers; or grants access to the anon role or equivalent public-access role. Invoke even when the migration appears routine — privilege and RLS changes are easy to overlook without a systematic trace."
---

# Database Migration Security Review

This skill guides a complete security review of a database migration. Follow all seven checks in order. Each check requires evidence from the migration file(s) and prior migration history — do not answer based on assumptions about what "should" be there.

---

## Pre-Review Setup

Before starting the checklist:

1. Identify the migration file(s) under review.
2. List all tables, schemas, roles, and functions the migration touches.
3. Find all **prior migrations** that affect the same tables/schemas/roles. Read them in full to establish the pre-migration baseline state.

If prior migrations are not available or the history is unclear, note this gap explicitly — security findings cannot be confirmed without a complete privilege chain.

---

## Checklist

### 1. Baseline Audit

Establish the current security state **before** the new migration runs.

- What grants and revokes exist on each affected table/schema, based on prior migrations?
- What RLS policies are currently in effect on each affected table?
- What roles have access, and through what grant path?

Document the baseline explicitly. Every finding in later checks is relative to this baseline.

### 2. Grant/Revoke Completeness

Examine every GRANT and REVOKE in the migration.

- Does the migration revoke all privileges it intends to restrict? Check for partial revocations (e.g., revoking `SELECT` but not `INSERT`).
- Does any new GRANT supersede or contradict an existing REVOKE from a prior migration?
- Are GRANTs scoped to specific operations (`SELECT`, `INSERT`, `UPDATE`, `DELETE`) rather than `ALL PRIVILEGES`?
- Are GRANTs to roles scoped to the minimum necessary access for that role's function?

### 3. RLS Policy Coverage

For every table the migration creates or modifies:

- Does an RLS policy exist for each role that can `SELECT`, `INSERT`, `UPDATE`, or `DELETE` from the table?
- Is `ROW LEVEL SECURITY` enabled on the table (`ALTER TABLE ... ENABLE ROW LEVEL SECURITY`)?
- Is `FORCE ROW LEVEL SECURITY` set where table owners need to be covered by RLS?
- Are there any tables where RLS was previously enforced but this migration bypasses or weakens it?

### 4. Anonymous Access Vectors

Explicitly check whether the `anon` role (or equivalent public/unauthenticated role in your stack) can reach affected rows after this migration runs.

Direct paths to check:
- Direct GRANTs to `anon` on affected tables or schemas.
- Policies that apply to `anon` or to `PUBLIC` (which includes `anon`).

Indirect paths to check:
- Functions with `SECURITY DEFINER` that `anon` can call — these execute as the function owner, potentially bypassing RLS.
- Views or materialized views accessible to `anon` that query affected tables.
- Row-level policies using `auth.role()` or equivalent that evaluate to a value `anon` can satisfy.

If `anon` had no access before and this migration is intended to preserve that, confirm there is no new path — direct or indirect.

### 5. Privilege Escalation

Review any new roles, functions, triggers, or extensions the migration introduces.

- Do new `SECURITY DEFINER` functions correctly validate the invoker's context before executing privileged operations?
- Do new triggers fire as a privileged role on actions the `anon` or `authenticated` role can initiate?
- Does the migration create a role with capabilities beyond what the described intent requires?
- Are there any `GRANT ... WITH GRANT OPTION` statements that could allow role members to re-grant privileges beyond their intended scope?

### 6. Rollback Impact

Reason about what happens if this migration is rolled back (or if the down migration runs).

- Does the rollback leave any table in a less secure state than it was before the migration ran?
- For example: if the migration adds an RLS policy and the rollback drops it, does the rollback also reinstate the prior policy — or does it leave the table with RLS enabled but no policies (effectively denying all access, or depending on default-permissive/restrictive policy type)?
- Are there orphaned grants that would remain after rollback?

### 7. Verdict

For each of the six checks above, state one of:

- **Satisfied** — evidence shows the requirement is fully met
- **Partially satisfied** — some evidence exists but the coverage is incomplete; describe the gap
- **Not addressed** — the migration does not cover this area, and it should; describe the risk
- **Not applicable** — this check does not apply to this migration; explain why

Conclude with an overall **Pass / Fail / Needs Clarification** verdict and a short summary of the most significant finding (or confirmation that no issues were found).

---

## Output Format

Structure your review as a markdown document with:

```
## Migration Security Review: <migration file name>

**Reviewer:** <agent>
**Date:** <date>
**Migration:** <file name and identifier>
**Tables/schemas affected:** <list>

---

### Baseline State
<summary of pre-migration privilege/RLS state>

### Check 1: Baseline Audit
**Result:** Satisfied / Partially satisfied / Not addressed / Not applicable
<evidence>

### Check 2: Grant/Revoke Completeness
...

### Check 3: RLS Policy Coverage
...

### Check 4: Anonymous Access Vectors
...

### Check 5: Privilege Escalation
...

### Check 6: Rollback Impact
...

### Check 7: Verdict
**Overall:** Pass / Fail / Needs Clarification
<summary>
```

Post this review as a comment on the issue, or attach it as a document if the issue supports structured documents.

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*
