---
title: "06 - Zero-Downtime Database Migration"
description: "Magento 2.4.8 production database migration: declarative schema patterns, column rename strategies, safe-installer, setup:db:status, and zero-downtime deployment"
tags: [magento2, database, migration, declarative-schema, zero-downtime, deployment]
rank: 8
pathways: [magento2-deep-dive]
see_also:
  - "Declarative schema: _supplemental/04-declarative-schema.md"
  - "CI/CD deployment: _supplemental/22-ci-cd-deployment.md"
---

# Zero-Downtime Database Migration

## 1. The Problem with Traditional DB Migrations

Magento 2.3 introduced declarative schema to solve a fundamental problem: **traditional migration scripts were dangerous in production**. Every `ALTER TABLE` statement in MySQL acquires a lock, and the scope of that lock depends on the operation.

### `ALTER TABLE` Locks Blocking Reads/Writes

MySQL's InnoDB engine handles DDL operations with varying levels of locking:

| Operation | Lock Type | Blocking Behavior |
|-----------|-----------|-------------------|
| `ADD COLUMN` (nullable) | Shared lock | Blocks writes, allows reads |
| `ADD COLUMN` (NOT NULL no default) | Exclusive lock | Blocks all access |
| `DROP COLUMN` | Exclusive lock | Blocks all access |
| `CHANGE COLUMN` (rename) | Exclusive lock | Blocks all access |
| `ALTER TABLE ... ADD INDEX` | Exclusive lock | Blocks all access |
| `ALTER TABLE ... ADD FOREIGN KEY` | Exclusive lock | Blocks all access |

The danger is proportional to table size. A 10-million-row `customer_entity` table that takes 30 seconds to `CHANGE COLUMN` will hold an exclusive lock for the entire duration, making your site unavailable.

### Long-Running Migrations During Deployment

In a traditional `UpgradeSchema` approach, every deployment packages delta scripts:

```
setup/UpgradeSchema.php:
addColumn('customer_entity', 'lifetime_value', ...)  // runs at request start
```

When you have 500 concurrent requests during deployment, this means:

1. **Schema upgrade runs during traffic spike** — slow, resource-intensive
2. **Old code is still running** — serving requests while schema is in flux
3. **Rollback is nearly impossible** — partial migration state is corrupt
4. **Maintenance window required** — classic "deploy at 2 AM" anti-pattern

### The Maintenance Mode Anti-Pattern

```php
// Traditional approach that kills production:
bin/magento maintenance:enable
bin/magento setup:upgrade
bin/magento cache:flush
bin/magento maintenance:disable
```

This pattern has multiple failure modes:
- End users see errors during maintenance window
- Long-running `setup:upgrade` extends the downtime window
- If it fails midway, recovery is complex
- No easy way to verify pre-migration health

The maintenance mode anti-pattern is fundamentally incompatible with 99.99% uptime SLAs.

---

## 2. Magento's Declarative Schema Approach

Magento 2.3+ introduced declarative schema as a **target-state model**, not a delta model. Instead of writing upgrade scripts that describe *what changed*, you define *what the final schema should look like*, and Magento's infrastructure handles the diff.

### `db_schema.xml` — Target-State Definition

The `etc/db_schema.xml` file defines the **complete desired state** of a module's tables:

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/db_schema.xsd">
    <table name="customer_entity" resource="default"
           engine="innodb" identifier="id">
        <column xsi:type="int" name="entity_id" padding="10" unsigned="true"
                identity="true" nullable="false" primary="true" comment="Entity ID"/>
        <column xsi:type="varchar" name="email" length="255" nullable="false"
                default="NULL" comment="Email"/>
        <column xsi:type="varchar" name="customer_lifetime_value" length="20"
                nullable="true" default="NULL" comment="Lifetime Value"/>
        <index indexType="btree" referenceId="CUSTOMER_EMAIL">
            <column name="email"/>
        </index>
    </table>
</schema>
```

Key concept: **`db_schema.xml` does not describe changes**. It describes the *target state*. If a column exists in this file and not in the database, Magento adds it. If a column exists in the database but not in this file, Magento removes it. This is fundamentally different from UpgradeSchema scripts.

### How Magento Generates Migration SQL

When you run `bin/magento setup:upgrade`, Magento's `DeclarativeSchemaManager` performs this workflow:

```
1. Read all db_schema.xml files from active modules
2. Build the "target schema" graph
3. Compare against current database schema (from setup_module table)
4. Generate the minimal SQL statements needed to reach target
5. Execute those SQL statements
6. Record the schema version in setup_module table
```

The critical insight: **Magento generates the minimal diff**. If you added one column, Magento generates one `ALTER TABLE ADD COLUMN` — not a complete table recreation.

### `setup:db:status` — Checking Schema Currency

This command tells you whether your database schema is current:

```bash
$ bin/magento setup:db:status

┌─────────────────────────────────────────────────────────────────────────────┐
│ DB status is up to date                                                     │
│ Current DB version: 2.4.0                                                   │
│ Expected DB version: 2.4.0                                                 │
│                                                                             │
│ All modules schemas are up to date                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

Possible outputs:

| Status | Meaning |
|--------|---------|
| `All modules schemas are up to date` | No migration needed |
| `Schema generation is required` | Declarative schema hasn't been applied |
| `Current DB version: X | Expected DB version: Y` | Module at version X, needs upgrade to Y |
| `The following modules have not been loaded:` | Modules not registered yet |

**Always run this before deployment**:

```bash
# Pre-deployment check
bin/magento setup:db:status || { echo "ABORT: Schema not current"; exit 1; }
```

### The Whitelist: `.whitelist.json`

When you install a module that doesn't have `db_schema.xml` defined, but manual changes were made to the database (say, a DBA added an index directly), Magento needs to know about those manual changes so it doesn't try to drop them.

The whitelist is the **record of all database changes that are outside Magento's control**:

```json
{
    "whitelist": {
        "customer_entity": {
            "column": {
                "manual_index_column": {
                    "COLUMN_NAME": "manual_index_column",
                    "TYPE": "varchar",
                    "LENGTH": "255"
                }
            },
            "index": {
                "MANUAL_INDEX_IDX": {
                    "INDEX_NAME": "MANUAL_INDEX_IDX",
                    "COLUMNS": ["manual_index_column"],
                    "TYPE": "btree"
                }
            }
        }
    }
}
```

**Generate the whitelist**:

```bash
# Generate from current database
vendor/bin/magento setup:db-declaration:generate-whitelist \
    --module=Vendor_Module \
    --output= Vendor_Module/etc/schema.whitelist.json
```

**Why this matters**: If a column exists in the database but not in `db_schema.xml`, Magento would normally schedule it for deletion. The whitelist tells Magento "this column exists because of external changes — don't touch it."

---

## 3. Safe Column Operations

Not all schema changes are equally dangerous. Understanding which operations are safe and which require the three-step pattern is critical.

### Adding Columns: Safe (with caveats)

Adding a column is generally safe because:
- MySQL reads existing rows without consulting the new column
- Application code that doesn't reference the new column continues working
- No data needs to be moved — the column is simply added with NULL values

**Safe pattern**:

```xml
<!-- db_schema.xml -->
<column name="customer_lifetime_value" xsi:type="decimal"
        precision="12" scale="4" nullable="true" default="NULL"/>
```

**Required: always use `nullable="true"` or provide a `default` value when adding columns to existing tables in production**. Adding a `NOT NULL` column without a default will cause the table to be rebuilt (full table lock).

### Renaming Columns: NEVER Direct Rename

**This is the most dangerous operation in production**. A direct `CHANGE COLUMN` rename:
1. Acquires an exclusive lock for the duration of the rename
2. Forces MySQL to rewrite every row (even though no data is changing)
3. Can take minutes on large tables
4. Blocks all reads and writes

**The Three-Step Rename Pattern**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      COLUMN RENAME: THREE-STEP APPROACH                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  STEP 1 (Deploy 1):                                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ ADD new_column alongside old_column                                 │   │
│  │ Old code reads old_column, new code reads new_column                 │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                         ↓                                                   │
│  STEP 2 (Deploy 2):                                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ Data patch migrates data: old_column → new_column                   │   │
│  │ Application now reads new_column (code deployed with Deploy 1)      │   │
│  │ Data migration runs asynchronously, no lock required                │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                         ↓                                                   │
│  STEP 3 (Deploy 3):                                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ Remove old_column from db_schema.xml                                │   │
│  │ Magento executes DROP COLUMN (after data confirmed migrated)         │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Why this works**:
- Step 1: No data movement, no lock (just adds a NULL column)
- Step 2: Application reads the new column (code already deployed), data migrates via `UPDATE` which is far less locking than `ALTER TABLE`
- Step 3: Old column is no longer referenced, safe to drop

### Adding NOT NULL Columns

When you must add a `NOT NULL` column:

```xml
<column name="customer_group_code" xsi:type="varchar"
        length="20" nullable="false" default="0"/>
```

The `default="0"` is **mandatory** for existing tables. This allows MySQL to add the column instantly (metadata-only change) without rewriting rows.

Without a default, MySQL must:
1. Add the column as nullable
2. Rewrite every existing row to set the column value
3. Later enforce NOT NULL constraint

### Dropping Columns: Rename to Deprecated First

**NEVER drop a column in the same deployment where it stops being used**. The sequence must be:

1. **Deployment 1**: Remove all code references to the column (but keep it in `db_schema.xml`)
2. **Deployment 2**: Remove the column from `db_schema.xml`

If you drop a column and the old code is still deployed:
- Old code will crash with "Unknown column" errors
- Rollback of the deployment won't restore the column

**The deprecation pattern**:

```xml
<!-- Deployment 1: column is now optional, code should not use it -->
<!-- db_schema.xml still has old_column -->
<!-- Code removes all references to old_column -->

<!-- Deployment 2: remove from db_schema.xml -->
<!-- DROP COLUMN executes safely -->
```

### Adding Indexes: Concurrent Construction

A standard `CREATE INDEX` acquires a shared lock for the duration of index creation. On a 50-million row table, this can take minutes.

**Safe approach**: Use `db_schema.xml` with `indexType="btree"` for standard indexes. Magento will generate the appropriate `CREATE INDEX` statement. For truly large tables, consider manual index creation during a maintenance window or use pt-online-schema-change (Percona Toolkit).

```xml
<index indexType="btree" referenceId="CUSTOMER_EMAIL">
    <column name="email"/>
</index>
```

**Note**: MySQL 8.0 supports `CREATE INDEX CONCURRENTLY` which builds the index without blocking reads/writes. Magento 2.4.x does not natively use this for declarative schema. For critical production tables, manual intervention may be required.

---

## 4. The Three-Step Column Rename Pattern (Complete Example)

This section provides a complete, working example of renaming a column `lifetime_value` → `customer_lifetime_value` on the `customer_entity` table.

### Step 1: Add the New Column (Deploy 1)

**File**: `app/code/Vendor/Customer/etc/db_schema.xml`

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/db_schema.xsd">
    <table name="customer_entity" resource="default"
           engine="innodb" identifier="id">
        <column xsi:type="decimal" name="customer_lifetime_value"
                precision="12" scale="4" nullable="true" default="NULL"
                comment="Customer Lifetime Value"/>
        <!-- existing columns remain -->
        <column xsi:type="varchar" name="email" length="255"
                nullable="false" default="NULL" comment="Email"/>
    </table>
</schema>
```

At this stage, `lifetime_value` still exists in the database and is still referenced by code. The `customer_lifetime_value` column exists but is empty.

### Step 2: Migrate Data (Data Patch)

**File**: `app/code/Vendor/Customer/Setup/Patch/Data/MigrateLifetimeValue.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\Customer\Setup\Patch\Data;

use Magento\Framework\DB\Adapter\AdapterInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\Patch\PatchVersionInterface;

/**
 * Data patch to migrate lifetime_value data to customer_lifetime_value.
 *
 * This patch runs AFTER schema patches during setup:upgrade.
 * Safe for production: uses batched updates to avoid long-running transactions.
 */
class MigrateLifetimeValue implements DataPatchInterface, PatchVersionInterface
{
    /** @var string */
    private static $moduleName = 'Vendor_Customer';

    /** @var int */
    private const BATCH_SIZE = 5000;

    /**
     * @param \Magento\Framework\Setup\ModuleDataSetupInterface $moduleDataSetup
     */
    public function __construct(
        private readonly \Magento\Framework\Setup\ModuleDataSetupInterface $moduleDataSetup
    ) {}

    /**
     * {@inheritdoc}
     */
    public function apply(): void
    {
        $connection = $this->moduleDataSetup->getConnection();

        if (!$connection->tableColumnExists('customer_entity', 'lifetime_value')) {
            // Migration already applied (idempotent)
            return;
        }

        if (!$connection->tableColumnExists('customer_entity', 'customer_lifetime_value')) {
            // Step 1 not yet applied — cannot migrate
            throw new \RuntimeException(
                'Column customer_lifetime_value does not exist. Apply Step 1 first.'
            );
        }

        $this->migrateDataInBatches($connection);
    }

    /**
     * Migrate data in batches to avoid long-running transactions and lock contention.
     */
    private function migrateDataInBatches(AdapterInterface $connection): void
    {
        $select = $connection->select()
            ->from('customer_entity', ['entity_id', 'lifetime_value'])
            ->where('lifetime_value IS NOT NULL')
            ->where('customer_lifetime_value IS NULL');

        $totalMigrated = 0;
        $lastId = 0;

        do {
            // Fetch batch
            $batch = $connection->fetchAll(
                $select->where('entity_id > ?', $lastId)->limit(self::BATCH_SIZE)
            );

            if (empty($batch)) {
                break;
            }

            // Update batch
            foreach ($batch as $row) {
                $connection->update(
                    'customer_entity',
                    ['customer_lifetime_value' => $row['lifetime_value']],
                    ['entity_id = ?' => $row['entity_id']]
                );
                $lastId = (int) $row['entity_id'];
            }

            $totalMigrated += count($batch);

            // Small sleep to reduce lock pressure on busy tables
            usleep(10000); // 10ms

        } while (count($batch) === self::BATCH_SIZE);
    }

    /**
     * {@inheritdoc}
     */
    public static function getDependencies(): array
    {
        return [];
    }

    /**
     * {@inheritdoc}
     */
    public function getVersion(): string
    {
        return '1.0.1';
    }
}
```

**Deploy this alongside code changes that read from `customer_lifetime_value`**. At this point:
- New code reads `customer_lifetime_value`
- Old code still reads `lifetime_value`
- Data patch migrates data from old → new

### Step 3: Remove the Old Column (Deploy 2)

After confirming data migration is complete (verify via `SELECT COUNT(*) FROM customer_entity WHERE lifetime_value IS NOT NULL AND customer_lifetime_value IS NULL` returning 0):

**File**: `app/code/Vendor/Customer/etc/db_schema.xml` (updated)

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/db_schema.xsd">
    <table name="customer_entity" resource="default"
           engine="innodb" identifier="id">
        <!-- lifetime_value column REMOVED — Magento will DROP it -->
        <column xsi:type="decimal" name="customer_lifetime_value"
                precision="12" scale="4" nullable="true" default="NULL"
                comment="Customer Lifetime Value"/>
        <column xsi:type="varchar" name="email" length="255"
                nullable="false" default="NULL" comment="Email"/>
    </table>
</schema>
```

Magento will generate and execute `ALTER TABLE customer_entity DROP COLUMN lifetime_value`.

---

## 5. Data Patch vs Schema Patch for Migrations

Magento has two distinct patch systems, and understanding their execution order is critical.

### SchemaPatchInterface — Structural Changes

`SchemaPatchInterface` patches operate on the **database structure**. They run during the schema installation phase, **before** data patches.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Schema;

use Magento\Framework\Setup\Patch\SchemaPatchInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class AddNewIndex implements SchemaPatchInterface
{
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup
    ) {}

    public function apply(): void
    {
        $this->moduleDataSetup->getConnection()->addIndex(
            'customer_entity',
            'CUSTOMER_LIFETIME_VALUE_IDX',
            ['customer_lifetime_value'],
            \Magento\Framework\DB\Adapter\AdapterInterface::INDEX_TYPE_INDEX
        );
    }

    public static function getDependencies(): array
    {
        return [];
    }
}
```

**Execution order**: Schema patches run first. If your patch returns `['Vendor_Module\Schema\SomeOtherPatch']` from `getDependencies()`, those patches run before this one.

### DataPatchInterface — Data Changes

`DataPatchInterface` patches operate on **data**. They run after all schema patches complete.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class InitializeConfigData implements DataPatchInterface
{
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup
    ) {}

    public function apply(): void
    {
        $data = [
            ['scope' => 'default', 'scope_id' => 0, 'path' => 'customer/lifetime/calculation', 'value' => 'monthly'],
            ['scope' => 'default', 'scope_id' => 0, 'path' => 'customer/lifetime/enabled', 'value' => 1],
        ];

        foreach ($data as $row) {
            $this->moduleDataSetup->getConnection()->insertOnDuplicate('core_config_data', $row);
        }
    }

    public static function getDependencies(): array
    {
        return [];
    }
}
```

### UnifiedDataLoader — Patch Execution Order Mechanism

Magento's `SetupDeployment` command orchestrates patch execution through `UnifiedDataLoader`:

```
setup:upgrade execution order:
1. Load all SchemaPatchInterface patches from all modules
2. Build dependency graph (getDependencies())
3. Topological sort → execution order
4. Execute schema patches in dependency order
5. Load all DataPatchInterface patches from all modules
6. Build dependency graph
7. Topological sort → execution order
8. Execute data patches in dependency order
9. Update setup_module table with version
```

**Why this order matters**:
- Schema must be correct before data is manipulated
- You can depend on another module's schema patch being complete
- You can depend on another module's data patch being complete

### When to Use `setup:upgrade` vs Manual Patch Execution

| Scenario | Command | Rationale |
|----------|---------|----------|
| Normal deployment | `setup:upgrade` | Full patch discovery and execution |
| Re-running a specific patch | `setup:db-data:upgrade` | Re-applies data patches |
| Re-running schema patches | `setup:db-schema:upgrade` | Re-applies schema patches |
| Verifying patch status | `setup:db:status` | Check current state |
| Generating migration SQL | `setup:db:generate-migration` | Preview changes |

**Never run `setup:upgrade` in production without testing first**. Always use `setup:db:generate-migration` to preview the SQL that will be generated.

---

## 6. The `safe-installer=1` Flag (Adobe Commerce Cloud)

The `safe-installer=1` configuration flag is an Adobe Commerce Cloud (formerly Magento Commerce Cloud) feature that addresses a specific concurrency problem: what happens when two deployment processes try to run schema migration simultaneously.

### What It Does

When `safe-installer=1` is set in `app/etc/env.php` or `app/etc/config.php`:

```php
// app/etc/env.php
return [
    'system' => [
        'default' => [
            'system' => [
                'safe_installer' => '1'
            ]
        ]
    ]
];
```

Magento checks for a **running MySQL process** before attempting schema migration. If another `setup:upgrade` is currently running schema operations, the new process will:

1. **Skip schema migration entirely** — not attempt to apply schema patches
2. **Complete successfully** — exit without error
3. **Allow the original process to finish** — no conflict, no corruption

This prevents the catastrophic scenario where two concurrent `setup:upgrade` processes apply conflicting schema changes and corrupt the database.

### How to Set It

**Option A: `env.php`** (applies only to that environment):

```php
// app/etc/env.php
'system' => [
    'default' => [
        'system' => [
            'safe_installer' => '1'
        ]
    ]
],
```

**Option B: `config.php`** (applies across environments, committed to git):

```php
// app/etc/config.php
'system' => [
    'default' => [
        'system' => [
            'safe_installer' => '1'
        ]
    ]
],
```

### `setup:db:add-foreign-key` — Safe FK Addition

Foreign key constraints present a particular challenge because they require both tables to be in a consistent state before the FK can be added. Standard MySQL behavior:

```sql
-- This blocks until both tables are accessible
ALTER TABLE sales_order ADD CONSTRAINT fk_sales_customer
    FOREIGN KEY (customer_id) REFERENCES customer_entity (entity_id);
```

The `setup:db:add-foreign-key` command was added to handle this safely:

```bash
# Safe foreign key addition (skips if already exists)
bin/magento setup:db:add-foreign-key \
    --table=sales_order \
    --column=customer_id \
    --reference=customer_entity \
    --reference-column=entity_id
```

When `safe-installer=1` is enabled, this command will also check for concurrent processes.

---

## 7. Pre-Migration Checklist

Running this checklist before every production migration prevents the most common failure modes.

### Step 1: `setup:db:status` Verification

```bash
$ bin/magento setup:db:status

┌─────────────────────────────────────────────────────────────────────────────┐
│ DB status is up to date                                                     │
│ Current DB version: 2.4.0                                                   │
│ Expected DB version: 2.4.0                                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Must return**: `All modules schemas are up to date` (or the expected "Current DB version" matches expected)

If it returns `Schema generation is required` or shows mismatched versions, do not proceed.

### Step 2: `setup:db:generate-migration` — Preview Migration SQL

```bash
$ bin/magento setup:db:generate-migration \
    --module=Vendor_Module \
    --output=/tmp/migration.sql

Generated migration file: /tmp/migration.sql
```

This creates `/tmp/migration.sql` containing the exact SQL Magento will execute. **Review this file before deployment**. Look for:

- Any `DROP COLUMN` statements you didn't expect
- `ALTER TABLE` statements on large tables
- `CREATE INDEX` statements that might be slow

### Step 3: Verify Whitelist Completeness

```bash
$ vendor/bin/magento setup:db-declaration:generate-whitelist \
    --module=Vendor_Module

Whitelist generated successfully.
```

Compare the generated whitelist against your expected schema. Any column or index that exists in the database but is not in either `db_schema.xml` or the whitelist is at risk of being dropped.

### Step 4: Backup Strategy

**Minimum backup required before any schema migration**:

```bash
# Database backup (mysqldump)
mysqldump -h $DB_HOST -u $DB_USER -p$DB_PASSWORD $DB_NAME \
    --single-transaction \
    --quick \
    --routines \
    --triggers \
    --events \
    > /backup/pre-migration-$(date +%Y%m%d-%H%M%S).sql

# File system backup (app/etc and var directory)
tar -czf /backup/pre-migration-files-$(date +%Y%m%d-%H%M%S).tar.gz \
    /var/www/html/app/etc \
    /var/www/html/var
```

**For major migrations (column renames, large data moves)**:
- Full database backup mandatory
- Point-in-time recovery test on staging first
- Verify backup restoration completes and site functions

### Pre-Migration Checklist Summary

| Check | Command | Pass Criteria |
|-------|---------|--------------|
| Schema current | `setup:db:status` | All modules schemas are up to date |
| SQL preview | `setup:db:generate-migration` | No unexpected DROP COLUMN |
| Whitelist | `setup:db-declaration:generate-whitelist` | All manual changes listed |
| Backup | Manual | Backup verified, tested restore |
| Large table impact | Manual review | No ALTER on tables > 1M rows during traffic |

---

## 8. Monitoring During Migration

Even with zero-downtime patterns, schema changes can cause performance degradation during the migration window. Monitoring is essential.

### `EXPLAIN ANALYZE` for Long-Running Queries

Before running data migration patches, check the query plan:

```sql
EXPLAIN ANALYZE
SELECT entity_id, lifetime_value
FROM customer_entity
WHERE lifetime_value IS NOT NULL
  AND customer_lifetime_value IS NULL
LIMIT 1000;
```

**What to look for**:
- `type: ALL` (full table scan) — add index before migration
- `rows: N` where N is very large — use batched approach
- `Using filesort` — avoid sorting large result sets

### Lock Wait Timeouts

MySQL's `innodb_lock_wait_timeout` controls how long a transaction waits for a lock:

```sql
-- Check current setting
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';
-- Value: 50 (seconds by default)

-- Set to 5 seconds for migration window (fail fast)
SET SESSION innodb_lock_wait_timeout = 5;
```

A lower timeout means failed locks fail quickly (allowing retry) rather than blocking for 50 seconds.

**In `my.cnf` for the migration window**:

```ini
[mysqld]
innodb_lock_wait_timeout = 5
```

### Monitoring Deadlocks During Migration

MySQL can detect deadlocks and automatically roll back one transaction. Enable deadlock logging:

```sql
-- Enable InnoDB deadlock monitor
SET GLOBAL innodb_print_all_deadlocks = ON;

-- Check MySQL error log for deadlocks
-- tail -f /var/log/mysql/error.log | grep -i deadlock
```

**Deadlock during migration** typically occurs when:
1. Data patch `UPDATE` acquires lock on rows
2. Application request tries to modify same rows
3. MySQL detects cycle, kills one transaction

**Mitigation**:
- Run data migration patches during lowest-traffic window
- Use smaller batch sizes to reduce lock hold duration
- Add retry logic to patches:

```php
private function executeWithRetry(string $sql, int $maxRetries = 3): void
{
    $attempt = 0;
    while ($attempt < $maxRetries) {
        try {
            $this->connection->query($sql);
            return;
        } catch (\Magento\Framework\DB\Adapter\DeadlockException $e) {
            $attempt++;
            if ($attempt >= $maxRetries) {
                throw $e;
            }
            sleep(pow(2, $attempt)); // exponential backoff
        }
    }
}
```

---

## Summary

Zero-downtime database migration in Magento 2.4.x is achieved through disciplined use of declarative schema, the three-step column rename pattern, and careful pre-migration verification.

**Core principles**:
- Declarative schema defines *target state*, not deltas — write `db_schema.xml` for the schema you want, not the changes
- Column renames require three separate deployments: add → migrate data → drop
- Always check `setup:db:status` before deployment
- Always preview SQL with `setup:db:generate-migration`
- Maintain whitelist for all manual database changes

**Key commands**:
| Command | Purpose |
|---------|---------|
| `setup:db:status` | Verify schema is current |
| `setup:db:generate-migration` | Preview migration SQL |
| `setup:db-declaration:generate-whitelist` | Generate/verify whitelist |
| `setup:upgrade` | Execute all patches |
| `setup:db:add-foreign-key` | Safe FK addition |

The zero-downtime approach trades a longer overall migration timeline (3 deployments instead of 1) for continuous availability — the correct trade-off for any production system with meaningful traffic.
