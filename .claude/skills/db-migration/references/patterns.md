# Database Migration Patterns

Concrete examples of expand/contract patterns for common schema changes.

---

## Pattern 1: Column Rename

**Scenario:** Rename `email_confirmed_at` to `email_verified_at` for clarity.

**Why expand/contract:** Changing column names breaks existing queries. Expand/contract allows old and new code to coexist.

### Phase 1: Expand

Add new column alongside old column:

```sql
-- Migration: 20240201000000_add_email_verified_at.sql
ALTER TABLE users
ADD COLUMN email_verified_at TIMESTAMP NULL;

-- Add index if original column was indexed
CREATE INDEX CONCURRENTLY idx_users_email_verified_at
ON users(email_verified_at);
```

**Deployment:** Deploy migration before code changes.

**Application state:** Code still uses `email_confirmed_at`. New column exists but is empty.

### Phase 2: Migrate

Backfill data from old column to new column:

```sql
-- Migration: 20240202000000_backfill_email_verified_at.sql
UPDATE users
SET email_verified_at = email_confirmed_at
WHERE email_confirmed_at IS NOT NULL
  AND email_verified_at IS NULL;
```

**If table is large (>100k rows), batch the backfill:**

```sql
-- Run in batches to avoid long locks
DO $$
DECLARE
  batch_size INT := 10000;
  rows_updated INT;
BEGIN
  LOOP
    UPDATE users
    SET email_verified_at = email_confirmed_at
    WHERE id IN (
      SELECT id FROM users
      WHERE email_confirmed_at IS NOT NULL
        AND email_verified_at IS NULL
      LIMIT batch_size
    );

    GET DIAGNOSTICS rows_updated = ROW_COUNT;
    EXIT WHEN rows_updated = 0;

    -- Sleep between batches to reduce load
    PERFORM pg_sleep(0.1);
  END LOOP;
END $$;
```

**Application changes:**

Deploy code that:
1. Reads from `email_verified_at` (with fallback to `email_confirmed_at` if null)
2. Writes to BOTH columns (dual-write)

```javascript
// Example application code during migrate phase
async function setEmailVerified(userId) {
  await db.query(
    `UPDATE users
     SET email_verified_at = NOW(),
         email_confirmed_at = NOW()  -- Dual-write
     WHERE id = $1`,
    [userId]
  );
}
```

**Verification:**
```sql
-- Verify backfill complete
SELECT COUNT(*)
FROM users
WHERE email_confirmed_at IS NOT NULL
  AND email_verified_at IS NULL;
-- Should return 0
```

### Phase 3: Contract

Drop old column after all code deployed:

```sql
-- Migration: 20240205000000_drop_email_confirmed_at.sql
ALTER TABLE users
DROP COLUMN email_confirmed_at;

-- Drop old index
DROP INDEX IF EXISTS idx_users_email_confirmed_at;
```

**Before executing:**
- [ ] All application instances using `email_verified_at`
- [ ] Verified backfill 100% complete
- [ ] Searched codebase: zero references to `email_confirmed_at`
- [ ] Tested in staging

**Rollback:**

If you need to roll back the contract phase:

```sql
-- Re-add the column
ALTER TABLE users
ADD COLUMN email_confirmed_at TIMESTAMP NULL;

-- Backfill from new column
UPDATE users
SET email_confirmed_at = email_verified_at
WHERE email_verified_at IS NOT NULL;

-- Recreate index
CREATE INDEX idx_users_email_confirmed_at
ON users(email_confirmed_at);
```

---

## Pattern 2: Column Type Change

**Scenario:** Change `age` column from `TEXT` to `INTEGER`.

**Why expand/contract:** Direct type change may fail if data contains non-numeric values. Expand/contract allows validation and safe migration.

### Phase 1: Expand

Add new column with correct type:

```sql
-- Migration: 20240201000000_add_age_int.sql
ALTER TABLE users
ADD COLUMN age_int INTEGER NULL;
```

### Phase 2: Migrate

Backfill with data validation:

```sql
-- Migration: 20240202000000_backfill_age_int.sql

-- First, identify problematic data
SELECT id, age
FROM users
WHERE age IS NOT NULL
  AND age !~ '^\d+$';  -- Not a valid integer
-- Handle these rows manually or set to NULL

-- Backfill valid integers
UPDATE users
SET age_int = CAST(age AS INTEGER)
WHERE age IS NOT NULL
  AND age ~ '^\d+$'
  AND age_int IS NULL;
```

**Application changes:**

Deploy code that:
1. Reads from `age_int` (with fallback to `age`)
2. Writes to BOTH columns
3. Validates input before writing

```javascript
async function updateAge(userId, ageValue) {
  const ageInt = parseInt(ageValue, 10);
  if (isNaN(ageInt)) {
    throw new Error('Invalid age');
  }

  await db.query(
    `UPDATE users
     SET age_int = $1,
         age = $2  -- Dual-write as string
     WHERE id = $3`,
    [ageInt, String(ageInt), userId]
  );
}
```

### Phase 3: Contract

Drop old text column:

```sql
-- Migration: 20240205000000_drop_age_text.sql
ALTER TABLE users
DROP COLUMN age;

-- Rename new column to original name
ALTER TABLE users
RENAME COLUMN age_int TO age;
```

---

## Pattern 3: Add NOT NULL Constraint

**Scenario:** Make `email` column NOT NULL (currently nullable).

**Why expand/contract:** Adding NOT NULL directly fails if any rows have NULL. Must backfill first.

### Phase 1: Expand

No schema change needed. Prepare data:

```sql
-- Migration: 20240201000000_prepare_email_not_null.sql

-- First, identify rows with NULL email
SELECT COUNT(*) FROM users WHERE email IS NULL;

-- Strategy 1: Delete invalid rows (if acceptable)
-- DELETE FROM users WHERE email IS NULL;

-- Strategy 2: Set default value (if acceptable)
-- UPDATE users SET email = 'unknown@example.com' WHERE email IS NULL;

-- Strategy 3: Mark for cleanup (if manual review needed)
-- ALTER TABLE users ADD COLUMN needs_email_review BOOLEAN DEFAULT false;
-- UPDATE users SET needs_email_review = true WHERE email IS NULL;
```

**Application changes:**

Deploy code that enforces email requirement:
- Prevent creation of users without email
- Add validation to ensure email is always provided

### Phase 2: Migrate

Verify all rows have valid email:

```sql
-- Verification query
SELECT COUNT(*) FROM users WHERE email IS NULL;
-- Must return 0 before proceeding
```

Add constraint:

```sql
-- Migration: 20240202000000_add_email_not_null.sql
ALTER TABLE users
ALTER COLUMN email SET NOT NULL;
```

### Phase 3: Contract

No contract phase needed (constraint addition is the final step).

**Rollback:**

```sql
ALTER TABLE users
ALTER COLUMN email DROP NOT NULL;
```

---

## Pattern 4: Table Split

**Scenario:** Split `users` table into `users` and `user_profiles` (move bio, avatar_url, etc. to separate table).

**Why expand/contract:** Existing queries reference all columns. Must migrate data without breaking queries.

### Phase 1: Expand

Create new table:

```sql
-- Migration: 20240201000000_create_user_profiles.sql
CREATE TABLE user_profiles (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  bio TEXT,
  avatar_url TEXT,
  website TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(user_id)
);

CREATE INDEX idx_user_profiles_user_id ON user_profiles(user_id);
```

### Phase 2: Migrate

Backfill data to new table:

```sql
-- Migration: 20240202000000_backfill_user_profiles.sql
INSERT INTO user_profiles (user_id, bio, avatar_url, website, created_at)
SELECT
  id,
  bio,
  avatar_url,
  website,
  NOW()
FROM users
WHERE bio IS NOT NULL
   OR avatar_url IS NOT NULL
   OR website IS NOT NULL
ON CONFLICT (user_id) DO NOTHING;
```

**Application changes:**

Deploy code that:
1. Reads from both tables (JOIN)
2. Writes to both tables (dual-write)

```javascript
// Before: Single table query
const user = await db.query(
  'SELECT id, email, bio, avatar_url FROM users WHERE id = $1',
  [userId]
);

// During migrate: JOIN both tables
const user = await db.query(`
  SELECT
    u.id, u.email,
    up.bio, up.avatar_url
  FROM users u
  LEFT JOIN user_profiles up ON up.user_id = u.id
  WHERE u.id = $1
`, [userId]);

// Dual-write on update
async function updateProfile(userId, bio, avatarUrl) {
  // Write to old table
  await db.query(
    'UPDATE users SET bio = $1, avatar_url = $2 WHERE id = $3',
    [bio, avatarUrl, userId]
  );

  // Write to new table
  await db.query(`
    INSERT INTO user_profiles (user_id, bio, avatar_url)
    VALUES ($1, $2, $3)
    ON CONFLICT (user_id)
    DO UPDATE SET bio = $2, avatar_url = $3
  `, [userId, bio, avatarUrl]);
}
```

**Verification:**

```sql
-- Check data consistency
SELECT
  u.id,
  u.bio AS old_bio,
  up.bio AS new_bio,
  u.avatar_url AS old_avatar,
  up.avatar_url AS new_avatar
FROM users u
LEFT JOIN user_profiles up ON up.user_id = u.id
WHERE u.bio != up.bio
   OR u.avatar_url != up.avatar_url;
-- Should return 0 rows
```

### Phase 3: Contract

Drop columns from original table:

```sql
-- Migration: 20240205000000_drop_user_profile_columns.sql
ALTER TABLE users
DROP COLUMN bio,
DROP COLUMN avatar_url,
DROP COLUMN website;
```

**Before executing:**
- [ ] All code reading from `user_profiles` table
- [ ] All code writing to both tables (dual-write)
- [ ] Data consistency verified
- [ ] Tested in staging

**Application changes (after contract):**

Remove dual-write, read only from `user_profiles`:

```javascript
// After contract: Read only from joined table
const user = await db.query(`
  SELECT
    u.id, u.email,
    up.bio, up.avatar_url
  FROM users u
  LEFT JOIN user_profiles up ON up.user_id = u.id
  WHERE u.id = $1
`, [userId]);

// Write only to new table
async function updateProfile(userId, bio, avatarUrl) {
  await db.query(`
    INSERT INTO user_profiles (user_id, bio, avatar_url)
    VALUES ($1, $2, $3)
    ON CONFLICT (user_id)
    DO UPDATE SET bio = $2, avatar_url = $3
  `, [userId, bio, avatarUrl]);
}
```

---

## Pattern 5: Large Data Backfill

**Scenario:** Add `email_normalized` column (lowercase email for case-insensitive search) and backfill 2 million rows.

**Why batching:** Updating 2M rows in one transaction locks the table and may timeout. Batching prevents locks and allows progress monitoring.

### Phase 1: Expand

```sql
-- Migration: 20240201000000_add_email_normalized.sql
ALTER TABLE users
ADD COLUMN email_normalized TEXT NULL;

-- Add index for case-insensitive search
CREATE INDEX CONCURRENTLY idx_users_email_normalized
ON users(email_normalized);
```

### Phase 2: Migrate (Batched)

Backfill in batches using a background job:

```sql
-- Migration: 20240202000000_backfill_email_normalized.sql
-- This migration only creates the backfill function
-- Actual execution happens via background job

CREATE OR REPLACE FUNCTION backfill_email_normalized_batch(batch_size INT)
RETURNS INT AS $$
DECLARE
  rows_updated INT;
BEGIN
  UPDATE users
  SET email_normalized = LOWER(email)
  WHERE id IN (
    SELECT id
    FROM users
    WHERE email IS NOT NULL
      AND email_normalized IS NULL
    LIMIT batch_size
    FOR UPDATE SKIP LOCKED
  );

  GET DIAGNOSTICS rows_updated = ROW_COUNT;
  RETURN rows_updated;
END;
$$ LANGUAGE plpgsql;
```

**Background job (application-level):**

```javascript
// Run this as a background job, not in migration
async function backfillEmailNormalized() {
  const BATCH_SIZE = 10000;
  const DELAY_MS = 100;  // Delay between batches

  let totalUpdated = 0;
  let batchCount = 0;

  while (true) {
    const result = await db.query(
      'SELECT backfill_email_normalized_batch($1)',
      [BATCH_SIZE]
    );

    const rowsUpdated = result.rows[0].backfill_email_normalized_batch;
    totalUpdated += rowsUpdated;
    batchCount++;

    console.log(`Batch ${batchCount}: Updated ${rowsUpdated} rows (total: ${totalUpdated})`);

    if (rowsUpdated === 0) {
      console.log('Backfill complete');
      break;
    }

    // Sleep between batches to reduce load
    await new Promise(resolve => setTimeout(resolve, DELAY_MS));
  }
}
```

**Application changes:**

Deploy code that:
1. Writes to BOTH `email` and `email_normalized`
2. Searches using `email_normalized` if available, falls back to `email`

```javascript
async function findUserByEmail(email) {
  // Prefer normalized column for case-insensitive search
  return db.query(`
    SELECT * FROM users
    WHERE email_normalized = LOWER($1)
       OR (email_normalized IS NULL AND LOWER(email) = LOWER($1))
    LIMIT 1
  `, [email]);
}

async function updateUserEmail(userId, newEmail) {
  // Dual-write
  await db.query(`
    UPDATE users
    SET email = $1,
        email_normalized = LOWER($1)
    WHERE id = $2
  `, [newEmail, userId]);
}
```

**Monitoring:**

```sql
-- Check backfill progress
SELECT
  COUNT(*) FILTER (WHERE email_normalized IS NOT NULL) AS backfilled,
  COUNT(*) FILTER (WHERE email_normalized IS NULL AND email IS NOT NULL) AS remaining,
  COUNT(*) AS total
FROM users;
```

### Phase 3: Contract

Optional: Add NOT NULL constraint after backfill complete:

```sql
-- Migration: 20240210000000_email_normalized_not_null.sql
-- Only run after backfill 100% complete

ALTER TABLE users
ALTER COLUMN email_normalized SET NOT NULL;
```

No columns to drop (additive migration).

---

## Pattern 6: Foreign Key Migration

**Scenario:** Change foreign key from `author_id` to `user_id` in `posts` table (renaming for consistency).

**Why expand/contract:** Foreign key constraints lock tables. Expand/contract minimizes lock duration.

### Phase 1: Expand

Add new column without constraint:

```sql
-- Migration: 20240201000000_add_user_id_to_posts.sql
ALTER TABLE posts
ADD COLUMN user_id UUID NULL;

-- Backfill immediately (single ALTER statement minimizes locks)
UPDATE posts SET user_id = author_id;

-- Add index first (before constraint)
CREATE INDEX idx_posts_user_id ON posts(user_id);
```

### Phase 2: Migrate

Add foreign key constraint:

```sql
-- Migration: 20240202000000_add_user_id_foreign_key.sql
ALTER TABLE posts
ADD CONSTRAINT fk_posts_user_id
FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE;
```

**Application changes:**

Deploy code that:
1. Reads from `user_id`
2. Writes to BOTH columns

```javascript
async function createPost(userId, title, content) {
  await db.query(`
    INSERT INTO posts (user_id, author_id, title, content)
    VALUES ($1, $1, $2, $3)
  `, [userId, title, content]);
}
```

### Phase 3: Contract

Drop old column and constraint:

```sql
-- Migration: 20240205000000_drop_author_id.sql
ALTER TABLE posts
DROP CONSTRAINT IF EXISTS fk_posts_author_id;

ALTER TABLE posts
DROP COLUMN author_id;
```

---

## Pattern 7: Enum to Check Constraint

**Scenario:** Replace PostgreSQL ENUM type with TEXT + CHECK constraint for flexibility.

**Current:**
```sql
CREATE TYPE user_status AS ENUM ('active', 'inactive', 'suspended');
ALTER TABLE users ADD COLUMN status user_status;
```

**Goal:**
```sql
ALTER TABLE users ADD COLUMN status TEXT CHECK (status IN ('active', 'inactive', 'suspended'));
```

**Why:** ENUMs are hard to modify (can't remove values, adding values requires lock). CHECK constraints are more flexible.

### Phase 1: Expand

Add new column with CHECK constraint:

```sql
-- Migration: 20240201000000_add_status_text.sql
ALTER TABLE users
ADD COLUMN status_text TEXT NULL
CHECK (status_text IN ('active', 'inactive', 'suspended'));
```

### Phase 2: Migrate

Backfill from enum to text:

```sql
-- Migration: 20240202000000_backfill_status_text.sql
UPDATE users
SET status_text = status::TEXT
WHERE status IS NOT NULL;
```

**Application changes:**

Deploy code that:
1. Reads from `status_text`
2. Writes to BOTH columns

```javascript
async function updateUserStatus(userId, status) {
  await db.query(`
    UPDATE users
    SET status_text = $1,
        status = $1::user_status
    WHERE id = $2
  `, [status, userId]);
}
```

### Phase 3: Contract

Drop old enum column and type:

```sql
-- Migration: 20240205000000_drop_status_enum.sql
ALTER TABLE users
DROP COLUMN status;

-- Rename new column
ALTER TABLE users
RENAME COLUMN status_text TO status;

-- Drop enum type (only if not used elsewhere)
DROP TYPE user_status;
```

**Adding new status values (after migration):**

```sql
-- Easy with CHECK constraint
ALTER TABLE users
DROP CONSTRAINT users_status_text_check;

ALTER TABLE users
ADD CONSTRAINT users_status_text_check
CHECK (status IN ('active', 'inactive', 'suspended', 'banned'));
```

---

## Pattern 8: Supabase RLS Policy Update

**Scenario:** After adding `email_verified` column, update RLS policies to require verification.

**Note:** RLS policy changes don't require expand/contract, but should be coordinated with schema changes.

### Before Migration

```sql
-- Old policy: No verification check
CREATE POLICY "Users can update own profile"
ON users FOR UPDATE
USING (auth.uid() = id)
WITH CHECK (auth.uid() = id);
```

### After Expand Phase

RLS still references old schema (safe during expand):

```sql
-- Policy still works with old column
CREATE POLICY "Users can update own profile"
ON users FOR UPDATE
USING (auth.uid() = id)
WITH CHECK (auth.uid() = id);
```

### After Migrate Phase

Update policy to reference new column:

```sql
-- Migration: 20240203000000_update_rls_for_email_verified.sql

-- Drop old policy
DROP POLICY IF EXISTS "Users can update own profile" ON users;

-- Create new policy with verification check
CREATE POLICY "Users can update own profile"
ON users FOR UPDATE
USING (
  auth.uid() = id
  AND (email_verified = true OR email_verified_at IS NOT NULL)
)
WITH CHECK (
  auth.uid() = id
);
```

**Timing:** Update RLS policies AFTER backfill completes, BEFORE contract phase.

---

## Summary Table

| Pattern | Risk Level | Batching Needed | Downtime |
|---------|------------|-----------------|----------|
| Column rename | Medium | No (<100k rows) | None |
| Column type change | Medium | Depends on validation | None |
| Add NOT NULL | Low-Medium | Depends on rows | None |
| Table split | High | Yes (>100k rows) | None |
| Large backfill | Medium-High | Yes (always) | None |
| Foreign key change | Medium | No | Minimal |
| Enum to check | Medium | No | None |
| RLS policy update | Low | No | None |

**Key principles:**
1. Always expand before contract
2. Batch operations on large tables
3. Dual-write during migrate phase
4. Verify before contract
5. Document rollback for every phase
