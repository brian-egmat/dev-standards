# Database Schema & Backward Compatibility Guide

Industry standards for managing database schema changes in long-term system development.

---

## Table of Contents

1. [The Core Problem](#1-the-core-problem)
2. [The Expand-Contract Pattern](#2-the-expand-contract-pattern)
3. [Safe vs Unsafe Operations](#3-safe-vs-unsafe-operations)
4. [Migration Strategies](#4-migration-strategies)
5. [Rollback Scenarios](#5-rollback-scenarios)
6. [Version Compatibility Matrix](#6-version-compatibility-matrix)
7. [Migration File Standards](#7-migration-file-standards)
8. [Tools & Automation](#8-tools--automation)
9. [Checklist](#9-checklist)
10. [Examples](#10-examples)

---

## 1. The Core Problem

### Scenario That Breaks Production

```
Timeline:
─────────────────────────────────────────────────────────────
v1.0          v1.1              v1.0 (rollback)
─────────────────────────────────────────────────────────────
Schema:       Schema:           Schema:
- users       - users           - users
  - name        - first_name      - first_name  <-- v1.0 expects 'name'
                - last_name       - last_name       but it doesn't exist!
                (dropped 'name')
─────────────────────────────────────────────────────────────
                     ^
                  BUG FOUND - Need to rollback!
                  But 'name' column is GONE!
```

**Result:** Rollback fails. Data may be lost. Production is broken.

### The Golden Rule

> **Never make schema changes that break currently running code.**
>
> Old code and new code must BOTH work with the database at every step.

---

## 2. The Expand-Contract Pattern

The industry-standard approach for safe schema changes.

### Three Phases

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     EXPAND-CONTRACT PATTERN                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  PHASE 1: EXPAND          PHASE 2: MIGRATE         PHASE 3: CONTRACT    │
│  ─────────────────        ─────────────────        ─────────────────    │
│                                                                          │
│  Add new structure        Move data & code         Remove old structure │
│  Keep old structure       to new structure         (point of no return) │
│                                                                          │
│  [OK] Rollback safe       [OK] Rollback safe       [X] NOT rollback safe│
│                                                                          │
│  Duration: Minutes        Duration: Days/Weeks     Duration: Minutes     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Phase Details

#### Phase 1: EXPAND (Add New, Keep Old)

```sql
-- Add new columns WITHOUT removing old ones
ALTER TABLE users ADD COLUMN first_name VARCHAR(100);
ALTER TABLE users ADD COLUMN last_name VARCHAR(100);
-- 'name' column still exists!
```

**Code changes:** None required. Old code keeps working.

#### Phase 2: MIGRATE (Dual-Write + Backfill)

```python
# Application code writes to BOTH old and new columns
def save_user(user):
    # Write to new columns
    user.first_name = extract_first_name(user.name)
    user.last_name = extract_last_name(user.name)
    # Keep old column updated for rollback safety
    user.name = f"{user.first_name} {user.last_name}"
    db.save(user)
```

```sql
-- Backfill existing data
UPDATE users
SET first_name = SPLIT_PART(name, ' ', 1),
    last_name = SPLIT_PART(name, ' ', 2)
WHERE first_name IS NULL;
```

**Duration:** Keep dual-write running until confident. Days or weeks.

#### Phase 3: CONTRACT (Remove Old)

```sql
-- Only after ALL code uses new columns
-- And you're confident no rollback needed
ALTER TABLE users DROP COLUMN name;
```

**WARNING:** This is the point of no return. Once you drop the column, you cannot rollback to code that needs it.

---

## 3. Safe vs Unsafe Operations

### Safe Operations (Always Backward Compatible)

| Operation | Why Safe |
|-----------|----------|
| `ADD COLUMN` (nullable) | Old code ignores new columns |
| `ADD COLUMN` (with default) | Old code ignores new columns |
| `CREATE TABLE` | Old code doesn't use new tables |
| `CREATE INDEX` | Doesn't change data structure |
| `ADD CONSTRAINT` (with NOT VALID) | Validates new rows only |

### Unsafe Operations (Breaking Changes)

| Operation | Why Unsafe | Safe Alternative |
|-----------|------------|------------------|
| `DROP COLUMN` | Old code fails | Use expand-contract |
| `RENAME COLUMN` | Old code fails | Add new, deprecate old |
| `ALTER COLUMN TYPE` | May fail/lose data | Add new column, migrate |
| `DROP TABLE` | Old code fails | Rename first, drop later |
| `NOT NULL` on existing | May fail on existing data | Add default first |

### Decision Matrix

```
Need to change schema?
         |
         v
    ┌────────────┐
    │ Is change  │──── YES ───> Just do it (single deployment)
    │ additive?  │
    └────────────┘
         | NO
         v
    ┌────────────┐
    │ Can old    │──── YES ───> Expand-Contract (multi-deployment)
    │ code work? │
    └────────────┘
         | NO
         v
    ┌────────────────────────────────────────┐
    │ STOP! Redesign the change.             │
    │ Breaking changes require maintenance   │
    │ windows and coordinated deployments.   │
    └────────────────────────────────────────┘
```

---

## 4. Migration Strategies

### Strategy 1: Additive-Only (Simplest)

**Rule:** Only add, never remove. Clean up in scheduled maintenance windows.

```sql
-- Migration 001: Add new columns
ALTER TABLE users ADD COLUMN first_name VARCHAR(100);
ALTER TABLE users ADD COLUMN last_name VARCHAR(100);

-- Migration 002 (months later, during maintenance): Cleanup
ALTER TABLE users DROP COLUMN name;
```

**Pros:** Simple, always safe
**Cons:** Schema bloat over time

### Strategy 2: Expand-Contract (Recommended)

**Rule:** Three-phase deployment with dual-write period.

```
Deployment 1: Expand
├── Add new columns
├── Deploy code that writes to both
└── Backfill existing data

Deployment 2: Switch Reads
├── Deploy code that reads from new columns
└── Keep writing to both

Deployment 3: Contract (after validation period)
├── Deploy code that only uses new columns
└── Drop old columns
```

**Pros:** Clean schema, controlled rollback
**Cons:** More complex, longer deployment cycle

### Strategy 3: Shadow Tables (For Major Restructuring)

**Rule:** Create new table structure, migrate data, swap.

```sql
-- Step 1: Create new table
CREATE TABLE users_v2 (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(100),
    last_name VARCHAR(100)
);

-- Step 2: Migrate data
INSERT INTO users_v2 (id, first_name, last_name)
SELECT id, SPLIT_PART(name, ' ', 1), SPLIT_PART(name, ' ', 2)
FROM users;

-- Step 3: Swap (in transaction)
BEGIN;
ALTER TABLE users RENAME TO users_old;
ALTER TABLE users_v2 RENAME TO users;
COMMIT;

-- Step 4: Drop old (after validation)
DROP TABLE users_old;
```

**Pros:** Clean migration, easy rollback before swap
**Cons:** Requires handling writes during migration

---

## 5. Rollback Scenarios

### Scenario A: Bug Found During Expand Phase

```
State: New columns added, old columns still exist
Action: Just rollback code. Schema is compatible with both versions.
Risk: LOW
```

### Scenario B: Bug Found During Migrate Phase

```
State: Dual-writing to both old and new columns
Action: Rollback code. Old column has current data.
Risk: LOW (if dual-write was implemented correctly)
```

### Scenario C: Bug Found After Contract Phase

```
State: Old columns dropped
Action: CANNOT simply rollback code
Options:
  1. Fix forward (patch the bug in new code)
  2. Restore from backup + replay transactions
  3. Re-add old columns + backfill (if data still derivable)
Risk: HIGH
```

### Rollback Safety Timeline

```
───────────────────────────────────────────────────────────────────────
SAFE ZONE                    | DANGER ZONE
─────────────────────────────|─────────────────────────────────────────
                             |
Expand ───> Migrate ─────────┼───> Contract
                             |
[OK] Can rollback anytime    | [X] Cannot rollback to old code
[OK] Old code works          | [X] Old code will fail
[OK] New code works          | [OK] New code works
                             |
───────────────────────────────────────────────────────────────────────
```

---

## 6. Version Compatibility Matrix

### Document Your Compatibility

Create a matrix showing which code versions work with which schema versions:

| Code Version | Schema v1 | Schema v1.1 | Schema v2 |
|--------------|-----------|-------------|-----------|
| v1.0 | Yes | Yes | No |
| v1.1 | Yes | Yes | No |
| v1.2 | No | Yes | Yes |
| v2.0 | No | No | Yes |

### Schema Version Tracking

Add a schema version table to your database:

```sql
CREATE TABLE schema_version (
    version INTEGER PRIMARY KEY,
    description TEXT,
    applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    applied_by TEXT
);

-- Track each migration
INSERT INTO schema_version (version, description, applied_by)
VALUES (1, 'Initial schema', 'deploy-script');
```

---

## 7. Migration File Standards

### Naming Convention

```
migrations/
├── 001_initial_schema.sql
├── 002_add_user_email.sql
├── 003_add_first_last_name.sql      # Expand
├── 004_backfill_first_last_name.sql  # Migrate
├── 005_drop_name_column.sql          # Contract
└── 006_add_user_preferences.sql
```

### Migration File Template

```python
# migrations/003_add_first_last_name.py
"""
Migration: Add first_name and last_name columns
Type: EXPAND (Phase 1 of name split)
Backward Compatible: YES
Rollback Safe: YES

Related migrations:
- 004: Backfill data (MIGRATE phase)
- 005: Drop name column (CONTRACT phase)
"""

VERSION = 3
DESCRIPTION = "Add first_name and last_name columns"
BACKWARD_COMPATIBLE = True
ROLLBACK_SAFE = True

def upgrade(cursor):
    cursor.execute("""
        ALTER TABLE users
        ADD COLUMN first_name VARCHAR(100);
    """)
    cursor.execute("""
        ALTER TABLE users
        ADD COLUMN last_name VARCHAR(100);
    """)

def downgrade(cursor):
    cursor.execute("ALTER TABLE users DROP COLUMN first_name;")
    cursor.execute("ALTER TABLE users DROP COLUMN last_name;")

def validate(cursor):
    """Verify migration was successful"""
    cursor.execute("""
        SELECT column_name
        FROM information_schema.columns
        WHERE table_name = 'users'
        AND column_name IN ('first_name', 'last_name');
    """)
    columns = [row[0] for row in cursor.fetchall()]
    assert 'first_name' in columns, "first_name column not found"
    assert 'last_name' in columns, "last_name column not found"
```

### Migration with Rollback Warning

```python
# migrations/005_drop_name_column.py
"""
Migration: Drop deprecated name column
Type: CONTRACT (Phase 3 of name split)
Backward Compatible: NO
Rollback Safe: NO

WARNING: This migration cannot be rolled back!
Ensure all code is using first_name/last_name before running.

Prerequisites:
- All application instances must be on v1.2+
- Verify no queries reference 'name' column
- Backup database before running
"""

VERSION = 5
DESCRIPTION = "Drop deprecated name column"
BACKWARD_COMPATIBLE = False
ROLLBACK_SAFE = False
REQUIRES_CONFIRMATION = True

def pre_check(cursor):
    """Verify it is safe to run this migration"""
    # Check no recent queries used the old column
    cursor.execute("""
        SELECT COUNT(*) FROM pg_stat_statements
        WHERE query LIKE '%users.name%'
        AND calls > 0
        AND last_call > NOW() - INTERVAL '7 days';
    """)
    recent_uses = cursor.fetchone()[0]
    if recent_uses > 0:
        raise Exception(f"Column 'name' was used {recent_uses} times in last 7 days!")

def upgrade(cursor):
    cursor.execute("ALTER TABLE users DROP COLUMN name;")

def downgrade(cursor):
    raise NotImplementedError(
        "Cannot rollback: data in 'name' column was not preserved. "
        "Restore from backup if rollback is required."
    )
```

---

## 8. Tools & Automation

### Recommended Migration Tools

| Tool | Language | Best For |
|------|----------|----------|
| **Flyway** | Java/Any | Simple SQL migrations, strong ordering |
| **Liquibase** | Java/Any | Enterprise, XML/YAML changelogs, rollback |
| **Alembic** | Python | SQLAlchemy projects, auto-generation |
| **Django Migrations** | Python | Django projects |
| **pgroll** | PostgreSQL | Automated expand-contract |
| **gh-ost** | MySQL | Zero-downtime on large tables |

### Simple Python Migration Runner

```python
# migrate.py
import os
import sqlite3
import importlib.util

def run_migrations(db_path, migrations_dir):
    conn = sqlite3.connect(db_path)
    cursor = conn.cursor()

    # Create migrations tracking table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS migrations (
            version INTEGER PRIMARY KEY,
            description TEXT,
            applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)

    # Get applied migrations
    cursor.execute("SELECT version FROM migrations")
    applied = {row[0] for row in cursor.fetchall()}

    # Find and run pending migrations
    for filename in sorted(os.listdir(migrations_dir)):
        if not filename.endswith('.py'):
            continue

        version = int(filename.split('_')[0])
        if version in applied:
            continue

        # Load and run migration
        spec = importlib.util.spec_from_file_location(
            f"migration_{version}",
            os.path.join(migrations_dir, filename)
        )
        module = importlib.util.module_from_spec(spec)
        spec.loader.exec_module(module)

        print(f"Running migration {version}: {module.DESCRIPTION}")

        # Run pre-check if exists
        if hasattr(module, 'pre_check'):
            module.pre_check(cursor)

        # Run upgrade
        module.upgrade(cursor)

        # Run validation if exists
        if hasattr(module, 'validate'):
            module.validate(cursor)

        # Record migration
        cursor.execute(
            "INSERT INTO migrations (version, description) VALUES (?, ?)",
            (version, module.DESCRIPTION)
        )
        conn.commit()
        print(f"Migration {version} complete")

    conn.close()

if __name__ == "__main__":
    run_migrations("data/app.db", "migrations/")
```

---

## 9. Checklist

### Before Creating a Migration

- [ ] Is this change backward compatible?
- [ ] Can old code still work after this migration?
- [ ] Is there a safe rollback path?
- [ ] Have you documented the compatibility matrix?

### Before Running Contract Phase

- [ ] All application instances updated?
- [ ] No recent queries using old structure?
- [ ] Database backed up?
- [ ] Team notified of point-of-no-return?
- [ ] Monitoring in place for errors?

### After Each Migration

- [ ] Validate migration succeeded
- [ ] Test application functionality
- [ ] Update version compatibility matrix
- [ ] Document in CHANGELOG.md

---

## 10. Examples

### Example 1: Adding a Required Column

**Wrong way (breaks old code):**
```sql
ALTER TABLE users ADD COLUMN email VARCHAR(255) NOT NULL;
-- Old code does not provide email - INSERT fails!
```

**Right way (backward compatible):**
```sql
-- Migration 1: Add nullable column
ALTER TABLE users ADD COLUMN email VARCHAR(255);

-- Migration 2 (after code updated): Set default for old rows
UPDATE users SET email = 'unknown@example.com' WHERE email IS NULL;

-- Migration 3 (after validation): Add constraint
ALTER TABLE users ALTER COLUMN email SET NOT NULL;
```

### Example 2: Renaming a Column

**Wrong way (breaks old code):**
```sql
ALTER TABLE users RENAME COLUMN name TO full_name;
-- Old code: SELECT name FROM users - fails!
```

**Right way (expand-contract):**
```sql
-- Migration 1 (EXPAND): Add new column
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);

-- Application: Dual-write to both columns
-- UPDATE users SET name = :val, full_name = :val

-- Migration 2 (MIGRATE): Backfill
UPDATE users SET full_name = name WHERE full_name IS NULL;

-- Application: Read from full_name, write to both

-- Migration 3 (CONTRACT): Drop old column (after all code updated)
ALTER TABLE users DROP COLUMN name;
```

### Example 3: Changing Column Type

**Wrong way:**
```sql
ALTER TABLE products ALTER COLUMN price TYPE DECIMAL(10,2);
-- May fail on existing data, may lose precision
```

**Right way:**
```sql
-- Migration 1: Add new column
ALTER TABLE products ADD COLUMN price_decimal DECIMAL(10,2);

-- Migration 2: Copy data
UPDATE products SET price_decimal = price::DECIMAL(10,2);

-- Application: Use price_decimal, dual-write

-- Migration 3: Drop old, rename new
ALTER TABLE products DROP COLUMN price;
ALTER TABLE products RENAME COLUMN price_decimal TO price;
```

---

## Quick Reference

### Safe Changes (Just Do It)
```sql
ADD COLUMN (nullable)
ADD COLUMN (with default)
CREATE TABLE
CREATE INDEX
```

### Unsafe Changes (Use Expand-Contract)
```sql
DROP COLUMN     -> Add new, migrate, drop old
RENAME COLUMN   -> Add new, migrate, drop old
CHANGE TYPE     -> Add new column, migrate, drop old
ADD NOT NULL    -> Add default first, then constraint
```

### Deployment Order
```
1. Run EXPAND migrations
2. Deploy new code (dual-write)
3. Run MIGRATE migrations (backfill)
4. Validate for days/weeks
5. Run CONTRACT migrations
6. Deploy final code (single-write)
```

---

## Sources

- [Prisma: Expand and Contract Pattern](https://www.prisma.io/dataguide/types/relational/expand-and-contract-pattern)
- [PlanetScale: Backward Compatible Database Changes](https://planetscale.com/blog/backward-compatible-databases-changes)
- [Martin Fowler: Evolutionary Database Design](https://martinfowler.com/articles/evodb.html)
- [Spring: Zero Downtime Deployment with Database](https://spring.io/blog/2016/05/31/zero-downtime-deployment-with-a-database/)
- [Liquibase: Blue-Green Deployments](https://www.liquibase.com/blog/blue-green-deployments-liquibase)

---

*Last Updated: 2026-01-08*
