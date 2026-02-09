---
name: db-migration
description: Plan safe database migrations using expand/contract patterns with comprehensive risk assessment and rollback strategies.
---

# Database Migration Planning

Plan production-safe database migrations with zero-downtime strategies and complete rollback procedures.

## Scope

**Use for:** Schema changes, data migrations, column additions/removals, table restructuring, constraint modifications.

**Not for:** Executing migrations (planning only), application code changes, API modifications.

---

# The Expand/Contract Pattern

The expand/contract pattern is the foundation for zero-downtime database migrations. It breaks destructive changes into safe, reversible phases.

## What It Is

A three-phase approach to schema changes:

1. **Expand** — Add new schema elements (columns, tables, indexes) alongside existing ones
2. **Migrate** — Dual-write or backfill data from old schema to new schema
3. **Contract** — Remove old schema elements after verifying new schema works

## When To Use

**Always use expand/contract for:**
- Column renames or type changes
- Table restructuring
- Non-nullable constraint additions
- Foreign key changes
- Any change that affects existing queries

**Not required for:**
- Adding new nullable columns with defaults
- Creating new tables (no existing data)
- Adding indexes (unless they cause contention)
- Simple data-only operations

## Why It Matters

Expand/contract prevents:
- Application downtime during deployments
- Data loss from failed migrations
- Breaking existing queries mid-deployment
- Rollback complications

Without it, deploying a schema change and code change simultaneously creates a critical window where old code breaks against new schema, or new code breaks against old schema.

---

# The Three Phases

## Phase 1: Expand

**Goal:** Add new schema elements without breaking existing code.

**Operations:**
- `ALTER TABLE ... ADD COLUMN` (nullable or with default)
- `CREATE TABLE` (new tables)
- `CREATE INDEX` (non-blocking if supported)
- Add new foreign keys to new columns

**Rules:**
- New columns must be nullable OR have a default value
- No DROP, no ALTER that changes existing columns
- Old code must continue to work unchanged

**Risk:** Low. Additive changes are inherently safe.

## Phase 2: Migrate

**Goal:** Populate new schema from old schema, maintain dual-write if needed.

**Operations:**
- Backfill data from old columns to new columns
- Update application code to write to both old and new schema
- Verify data consistency

**Rules:**
- Application must write to BOTH old and new columns during this phase
- Backfill large tables in batches to avoid locking
- Monitor for data drift between old and new

**Risk:** Medium. Data volume matters. Test backfill duration.

**Duration considerations:**
- <10k rows: Immediate
- 10k-100k rows: 1-10 seconds, acceptable in transaction
- 100k-1M rows: Batch in background job
- >1M rows: Batch + monitor, may take hours

## Phase 3: Contract

**Goal:** Remove old schema after confirming new schema works.

**Operations:**
- `ALTER TABLE ... DROP COLUMN`
- `DROP TABLE` (old tables)
- `DROP INDEX` (old indexes)
- Remove old foreign keys

**Rules:**
- ALWAYS requires explicit approval
- Must verify no code references old schema
- Must verify backfill completed successfully
- Only execute after new code fully deployed

**Risk:** High. Destructive. Irreversible without backups.

**Required before contract:**
- [ ] All application instances using new schema
- [ ] Data backfill verified 100% complete
- [ ] Zero references to old columns in codebase
- [ ] Rollback plan documented and tested

---

# db-migration-plan.yaml Schema

Every migration must produce this artifact.

```yaml
# Required root-level fields
db_change:
  required: true|false  # Does this feature need DB changes?

  # Migration phases (empty array if db_change.required = false)
  migrations:
    - phase: expand|migrate|contract
      description: "Human-readable description of what this step does"
      sql: "Exact SQL statement or reference to migration file"
      deployment_dependency: "Code must deploy before|after|during this migration"
      estimated_duration: "Estimated time for this operation (e.g., '< 1s', '~30s', '5-10min')"
      blocking: true|false  # Does this lock tables or block writes?
      requires_approval: true|false  # Destructive operations require true

  # Data backfill requirements
  data_backfill:
    required: true|false
    steps:
      - description: "What data to backfill"
        row_count_estimate: 50000  # Approximate rows affected
        batching_required: true|false
        batch_size: 1000  # If batching_required = true
        sql: "Backfill query or reference to script"

  # Risk assessment
  risk:
    level: low|med|high
    data_volume: "e.g., ~50k rows, ~2GB table"
    downtime_impact: "Expected user-facing impact"
    notes:
      - "Additional risk considerations"
      - "Performance implications"
      - "Locking behavior"

  # Rollback procedure (REQUIRED for every migration)
  rollback:
    automated: true|false  # Can this be rolled back automatically?
    steps:
      - phase: "Which phase to roll back"
        action: "Exact SQL or procedure to reverse"
        data_loss_risk: "none|partial|full"
        notes: "Warnings or considerations"

# Optional: Supabase-specific migrations
supabase:
  migration_files:
    - "20240205000000_add_email_verified.sql"
  rls_changes:
    - table: "users"
      policy: "Users can read own data"
      impact: "Must update policy to reference new column"

# Optional: Index strategy
indexes:
  new:
    - table: "users"
      columns: ["email_verified", "created_at"]
      type: "btree"
      concurrent: true  # Use CREATE INDEX CONCURRENTLY (Postgres)
  removed:
    - "users_email_confirmed_at_idx"
```

---

# Risk Assessment Framework

Every migration must be assessed for risk. This determines approval requirements and deployment strategy.

## Risk Levels

### Low Risk
**Characteristics:**
- Additive changes only (ADD COLUMN, CREATE TABLE)
- No data backfill required
- No locking or blocking operations
- <10k rows affected
- Instant rollback available

**Examples:**
- Add nullable column with default
- Create new table
- Add index concurrently
- Insert new seed data

**Approval:** None required, proceed with normal review.

### Medium Risk
**Characteristics:**
- Data backfill required
- 10k-1M rows affected
- Short-duration locking (<5s)
- Requires dual-write period
- Rollback requires manual steps

**Examples:**
- Backfill existing column from another column
- Add NOT NULL constraint after backfill
- Rename column (expand/contract)
- Alter column type with casting

**Approval:** Tech lead review, verify backfill tested in staging.

### High Risk
**Characteristics:**
- Destructive operations (DROP COLUMN, DROP TABLE)
- >1M rows affected
- Long-duration locking (>5s)
- Potential data loss
- Complex rollback requiring backup restore

**Examples:**
- DROP COLUMN (contract phase)
- DROP TABLE
- Change column type without safe casting
- Destructive backfill (UPDATE with data transformation)

**Approval:** Explicit sign-off required. Must document:
- Complete rollback procedure
- Backup verification
- Data loss impact assessment
- Deployment coordination plan

## Data Volume Considerations

| Row Count | Backfill Strategy | Risk Impact |
|-----------|-------------------|-------------|
| < 10k | Single transaction | Low |
| 10k - 100k | Single transaction or small batches | Low-Med |
| 100k - 1M | Batched background job | Medium |
| 1M - 10M | Batched + monitoring | Med-High |
| > 10M | Staged rollout, may require maintenance window | High |

## Downtime Impact Assessment

**Zero downtime:**
- Expand phase with nullable columns
- Non-blocking index creation
- Additive-only changes

**Minimal downtime (<1s):**
- Fast backfills under 100k rows
- Quick constraint additions
- Metadata-only operations

**Planned downtime:**
- Large table restructuring
- Complex multi-table migrations
- Operations requiring exclusive locks

---

# Rollback Requirements

**Every migration must have a documented rollback procedure.**

## What Makes a Good Rollback

A complete rollback plan includes:

1. **Exact reverse steps** — SQL statements or commands to undo each phase
2. **Data loss assessment** — What data (if any) is lost during rollback
3. **Order of operations** — Which steps to reverse first
4. **Verification steps** — How to confirm rollback succeeded

## Rollback by Phase

### Expand Phase Rollback
**Risk:** Low. New columns/tables can be dropped safely.

```yaml
rollback:
  automated: true
  steps:
    - phase: expand
      action: "DROP COLUMN email_verified"
      data_loss_risk: none
      notes: "Safe to drop; no data written yet"
```

### Migrate Phase Rollback
**Risk:** Medium. May lose data written to new schema.

```yaml
rollback:
  automated: false
  steps:
    - phase: migrate
      action: "Stop dual-write; keep old schema only"
      data_loss_risk: partial
      notes: "Loses data written to new schema during migration window"
```

### Contract Phase Rollback
**Risk:** High. Dropped columns cannot be restored without backup.

```yaml
rollback:
  automated: false
  steps:
    - phase: contract
      action: "Restore from backup taken before contract phase"
      data_loss_risk: full
      notes: "CRITICAL: Requires restore from backup. All data written after contract is lost."
    - phase: contract
      action: "Re-add email_confirmed_at column and backfill from email_verified"
      data_loss_risk: none
      notes: "If old data still exists elsewhere, can recreate column"
```

## Rollback Testing

Before approving any medium/high-risk migration:

- [ ] Document exact rollback SQL
- [ ] Test rollback in staging environment
- [ ] Verify data integrity after rollback
- [ ] Time the rollback duration
- [ ] Confirm application works after rollback

---

# Supabase-Specific Considerations

Supabase uses Postgres + additional services. Special considerations apply.

## Migration Files

Supabase migrations live in `supabase/migrations/`:

```
supabase/migrations/
  20240205000000_add_email_verified.sql
  20240205000001_backfill_email_verified.sql
  20240205000002_drop_email_confirmed_at.sql
```

**Naming convention:** `{timestamp}_{description}.sql`

**Each phase should be a separate file** to allow incremental deployment.

## Supabase CLI

Execute migrations via CLI:
```bash
supabase db push                    # Push all pending migrations
supabase db reset                   # Reset local DB (dev only)
supabase migration new <name>       # Create new migration file
```

**Production:** Migrations run automatically on `git push` to linked branch (or via dashboard).

## Row Level Security (RLS) Implications

Schema changes affect RLS policies.

**Common impacts:**
- New columns require RLS policy updates
- Dropped columns may break existing policies
- Column renames require policy SQL updates

**Always check:**
1. Does this table have RLS enabled?
2. Do existing policies reference changed columns?
3. Do new columns need to be added to SELECT policies?

**Example:**
```sql
-- Old policy
CREATE POLICY "Users read own data"
ON users FOR SELECT
USING (auth.uid() = id AND email_confirmed_at IS NOT NULL);

-- Must update after contract phase
CREATE POLICY "Users read own data"
ON users FOR SELECT
USING (auth.uid() = id AND email_verified = true);
```

## Realtime Subscriptions

If table has Realtime enabled, schema changes may affect subscriptions:
- New columns are automatically included
- Dropped columns break subscriptions referencing them
- Consider notifying clients to refresh subscriptions

## Storage Triggers

Supabase Storage may have triggers on `storage.objects` table. Verify:
- Are there triggers on this table?
- Do triggers reference columns being changed?
- Test trigger behavior in staging

---

# Blocking Rules

Certain operations are BLOCKED unless explicit approval is granted.

## Automatic Blocks

These operations trigger an automatic STOP:

1. **DROP COLUMN without expand/contract**
   - Block reason: Data loss, no rollback
   - Required: Expand/contract plan + approval

2. **DROP TABLE without backup verification**
   - Block reason: Permanent data loss
   - Required: Backup confirmation + approval + 24hr waiting period

3. **ALTER COLUMN TYPE without safe casting**
   - Block reason: Potential data loss or corruption
   - Required: Test casting in staging + approval

4. **Schema change breaks existing queries**
   - Block reason: Application breakage
   - Required: Create `spec-change-requests.yaml` entry

5. **Large backfill (>1M rows) without batching**
   - Block reason: Table locking, performance impact
   - Required: Batch strategy + monitoring plan

## Approval Requirements

When blocked, produce:

1. **Risk assessment** with level: high
2. **Complete rollback plan** including backup restore procedure
3. **Spec change request** if requirements conflict
4. **Explicit approval note** in `db-migration-plan.yaml`:

```yaml
migrations:
  - phase: contract
    sql: "DROP COLUMN email_confirmed_at"
    requires_approval: true
    approval_justification: |
      Contract phase of expand/contract migration.
      Expand phase completed: 2024-02-01
      Migrate phase completed: 2024-02-03
      All code deployed to reference new column: 2024-02-04
      Zero references to old column verified via codebase search.
      Rollback: Can re-add column and backfill from email_verified if needed.
```

## Override Process

To override a block:

1. Document why the block is being overridden
2. Provide mitigation strategy
3. Get explicit sign-off from tech lead or architect
4. Note override in `checks.yaml`:

```yaml
db_migration:
  status: pass
  blockers:
    - "DROP COLUMN email_confirmed_at (destructive)"
  overrides:
    - blocker: "DROP COLUMN email_confirmed_at (destructive)"
      approved_by: "tech-lead"
      justification: "Part of expand/contract, expand complete, migrate verified"
      date: "2024-02-05"
```

---

# Workflow

When a task involves database changes:

## 1. Analyze Requirements

Read:
- `.ops/build/v{x}/<feature-name>/specs.md` — What schema changes are needed?
- `.ops/build/v{x}/<feature-name>/tasks.yaml` — Which tasks touch the database?
- Current schema (Supabase dashboard or migration files)

Identify:
- What columns/tables are being added, modified, or removed?
- What data needs to be migrated or backfilled?
- Are there foreign keys, indexes, or constraints involved?

## 2. Choose Migration Strategy

For each change, determine:
- **Additive only?** → Single expand phase, low risk
- **Column rename/type change?** → Expand/contract, medium risk
- **Column/table removal?** → Expand/contract, high risk, requires approval
- **Large data migration?** → Assess batching needs, medium/high risk

## 3. Plan Expand/Contract Phases

Break the change into phases:

**Expand:**
- Add new columns (nullable or with defaults)
- Create new tables
- Add indexes concurrently

**Migrate:**
- Backfill data from old → new
- Deploy application code to dual-write
- Verify data consistency

**Contract:**
- Drop old columns/tables
- Remove old indexes
- Clean up deprecated constraints

## 4. Assess Risk

Determine risk level:
- Data volume (row count)
- Locking behavior (blocking vs non-blocking)
- Rollback complexity
- Potential for data loss

## 5. Document Rollback

For each phase, document:
- Exact reverse SQL
- Data loss implications
- Order of rollback steps
- Verification procedure

## 6. Write Migration Plan

Create `.ops/build/v{x}/<feature-name>/db-migration-plan.yaml` with:
- All phases
- Risk assessment
- Rollback procedure
- Supabase-specific notes (RLS, Realtime, etc.)

## 7. Update Checks

Merge into `.ops/build/v{x}/<feature-name>/checks.yaml`:

```yaml
db_migration:
  status: pass|fail
  blockers:
    - "List any automatic blocks"
  notes:
    - "Risk level: low/med/high"
    - "Estimated duration: X seconds/minutes"
    - "Deployment coordination: expand before code, contract after"
```

## 8. Block If Necessary

If destructive operation without approval:
- Set `checks.yaml` status to `fail`
- List blocker
- Request explicit approval
- Do not proceed

---

# Common Patterns

See `references/patterns.md` for detailed examples of:
- Column rename (expand/contract)
- Column type change
- Table split
- Constraint addition
- Large data backfill

---

# Examples

## Example 1: Add Nullable Column (Low Risk)

**Task:** Add `phone_number` column to `users` table.

**Migration Plan:**
```yaml
db_change:
  required: true
  migrations:
    - phase: expand
      description: "Add phone_number column"
      sql: "ALTER TABLE users ADD COLUMN phone_number TEXT NULL"
      deployment_dependency: "Can deploy before or after code"
      estimated_duration: "< 1s"
      blocking: false
      requires_approval: false
  data_backfill:
    required: false
  risk:
    level: low
    data_volume: "N/A (no backfill)"
    downtime_impact: "None"
    notes:
      - "Additive change, zero downtime"
      - "Nullable column, no default needed"
  rollback:
    automated: true
    steps:
      - phase: expand
        action: "DROP COLUMN phone_number"
        data_loss_risk: "partial"
        notes: "Loses any phone numbers added after deploy"
```

## Example 2: Column Rename (Medium Risk)

**Task:** Rename `email_confirmed_at` to `email_verified_at`.

**Migration Plan:**
```yaml
db_change:
  required: true
  migrations:
    - phase: expand
      description: "Add email_verified_at column"
      sql: "ALTER TABLE users ADD COLUMN email_verified_at TIMESTAMP NULL"
      deployment_dependency: "Before code deploy"
      estimated_duration: "< 1s"
      blocking: false
      requires_approval: false

    - phase: migrate
      description: "Backfill email_verified_at from email_confirmed_at"
      sql: "UPDATE users SET email_verified_at = email_confirmed_at WHERE email_confirmed_at IS NOT NULL"
      deployment_dependency: "After expand, before code deploy"
      estimated_duration: "~5s for 50k rows"
      blocking: true
      requires_approval: false

    - phase: migrate
      description: "Deploy application to use email_verified_at and dual-write"
      sql: "N/A (code deployment)"
      deployment_dependency: "After backfill"
      estimated_duration: "N/A"
      blocking: false
      requires_approval: false

    - phase: contract
      description: "Drop old email_confirmed_at column"
      sql: "ALTER TABLE users DROP COLUMN email_confirmed_at"
      deployment_dependency: "After code fully deployed"
      estimated_duration: "< 1s"
      blocking: false
      requires_approval: true

  data_backfill:
    required: true
    steps:
      - description: "Copy email_confirmed_at to email_verified_at"
        row_count_estimate: 50000
        batching_required: false
        sql: "UPDATE users SET email_verified_at = email_confirmed_at WHERE email_confirmed_at IS NOT NULL"

  risk:
    level: med
    data_volume: "~50k rows"
    downtime_impact: "None (expand/contract)"
    notes:
      - "Requires deployment coordination"
      - "Dual-write period between migrate and contract"
      - "Contract phase is destructive"

  rollback:
    automated: false
    steps:
      - phase: contract
        action: "Re-add email_confirmed_at and backfill from email_verified_at"
        data_loss_risk: "none"
        notes: "Can restore column from new column data"
      - phase: migrate
        action: "Revert code to use email_confirmed_at only"
        data_loss_risk: "partial"
        notes: "Loses data written to email_verified_at during dual-write"
      - phase: expand
        action: "DROP COLUMN email_verified_at"
        data_loss_risk: "none"
        notes: "Safe if no data written yet"
```

---

# Deep Dives

For more detail on specific topics:
- `references/patterns.md` — Concrete expand/contract examples
