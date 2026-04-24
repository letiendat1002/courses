---
title: "24 - Declarative Schema & Data Patches"
description: "Deep dive into Magento 2.4.8 database development: declarative schema (db_schema.xml), data patches, schema patches, and migration from old UpgradeSchema scripts."
tags: magento2, database, declarative-schema, db-schema, data-patches, schema-patches, migrations
rank: 4
pathways: [magento2-deep-dive]
---

# 24 — Declarative Schema & Data Patches

> **Magento version:** 2.4.8 | **PHP:** 8.3+ | **MySQL/MariaDB:** 8.0+ / 10.6+

---

## Table of Contents

1. [Declarative Schema Overview](#1-declarative-schema-overview)
2. [db\_schema.xml Structure](#2-db_schemaxml-structure)
3. [db\_schema.xml Examples](#3-db_schemaxml-examples)
4. [Data Patches](#4-data-patches)
5. [Schema Patches](#5-schema-patches)
6. [Recurring Patches](#6-recurring-patches)
7. [Patch Registration & Execution Order](#7-patch-registration--execution-order)
8. [Whitelist (db\_schema\_whitelist.json)](#8-whitelist-db_schema_whitelistjson)
9. [Declarative Schema vs Old Setup Scripts](#9-declarative-schema-vs-old-setup-scripts)
10. [Indexer Invalidation](#10-indexer-invalidation)

---

## 1. Declarative Schema Overview

### What Is Declarative Schema?

Introduced in **Magento 2.3** and now the only supported approach from **2.4.x** onward, **Declarative Schema** replaces the procedural `InstallSchema`, `UpgradeSchema`, `InstallData`, and `UpgradeData` PHP scripts with a single XML file — `db_schema.xml` — that **describes the desired final state** of the database tables owned by a module.

The Magento framework compares the current database state against the XML declaration and automatically generates and runs the necessary DDL statements (CREATE, ALTER, DROP) to converge the database to the declared state.

```
Old approach (imperative):          New approach (declarative):
───────────────────────           ───────────────────────────
Setup/InstallSchema.php             etc/db_schema.xml
Setup/UpgradeSchema.php             (one file, describes final state)
Setup/InstallData.php               Setup/Patch/Data/*.php
Setup/UpgradeData.php               Setup/Patch/Schema/*.php
```

### Why It Replaced the Old Scripts

The old `UpgradeSchema` pattern had serious problems at scale:

| Problem | Description |
|---|---|
| **Version coupling** | Every upgrade had to check `$context->getVersion()` and branch on string comparisons — fragile and error-prone |
| **No rollback** | If an upgrade failed halfway, the database was left in an inconsistent state with no automated recovery |
| **Unreadable diff** | Reading an UpgradeSchema file to understand the *current* schema required mentally replaying every version |
| **Duplicate logic** | InstallSchema created the table; each UpgradeSchema then modified it incrementally — same table described in N files |
| **Hard module removal** | No mechanism existed to cleanly drop tables when a module was removed |

Declarative schema solves all of this:

- **Single source of truth** — `db_schema.xml` always reflects the current schema intention.
- **Automatic DDL generation** — Magento diffs XML vs live DB and generates only the needed `ALTER TABLE` statements.
- **Dry-run support** — `bin/magento setup:db-declaration:generate-whitelist --module-name=Vendor_Module` and `bin/magento setup:upgrade --dry-run` let you preview changes before applying them.
- **Auto-rollback on failure** — DDL changes run inside a transaction (where the storage engine supports it) and roll back on error.
- **Clean uninstall** — The whitelist JSON enables Magento to drop tables safely when a module is removed.

### The XSD Contract

Every `db_schema.xml` must declare the XSD location so IDEs and Magento's validator can enforce correctness:

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etcSchema.xsd">
    <!-- table definitions -->
</schema>
```

The XSD lives at:
```
vendor/magento/framework/Setup/Declaration/Schema/etcSchema.xsd
```

---

## 2. db\_schema.xml Structure

### Full Element Hierarchy

```
<schema>
└── <table> (one per database table)
    ├── <column> (one per column)
    ├── <constraint>
    │   ├── xsi:type="primary"  → <column> references
    │   ├── xsi:type="unique"   → <column> references
    │   └── xsi:type="foreign"  → single column + referenceTable/referenceColumn
    └── <index>
        └── <column> references
```

### `<table>` Attributes

| Attribute | Required | Description | Example |
|---|---|---|---|
| `name` | ✅ | Table name without prefix | `vendor_module_brand` |
| `resource` | ✅ | DB connection resource | `default`, `checkout`, `sales` |
| `engine` | ✅ | Storage engine | `innodb` |
| `comment` | ❌ | Table comment | `"Brand catalog entity"` |

### Column Types and Their Attributes

#### Integer Types

| `xsi:type` | MySQL Type | Bytes | Signed Range |
|---|---|---|---|
| `smallint` | SMALLINT | 2 | -32 768 to 32 767 |
| `int` | INT | 4 | -2 147 483 648 to 2 147 483 647 |
| `bigint` | BIGINT | 8 | -9.2 × 10¹⁸ to 9.2 × 10¹⁸ |

Applicable attributes:

| Attribute | Type | Description |
|---|---|---|
| `name` | string | Column name |
| `padding` | int | Display width (e.g. `10` → `INT(10)`) |
| `unsigned` | bool | Disallow negative values |
| `nullable` | bool | Allow `NULL` |
| `default` | string | Default value |
| `identity` | bool | `AUTO_INCREMENT` |
| `comment` | string | Column comment |

```xml
<column xsi:type="int"
        name="entity_id"
        padding="10"
        unsigned="true"
        nullable="false"
        identity="true"
        comment="Entity ID"/>

<column xsi:type="smallint"
        name="status"
        padding="5"
        unsigned="true"
        nullable="false"
        default="1"
        comment="Status flag"/>

<column xsi:type="bigint"
        name="sequence_value"
        padding="20"
        unsigned="true"
        nullable="false"
        default="0"
        comment="Auto-increment sequence"/>
```

#### Decimal / Float Types

| `xsi:type` | MySQL Type | Notes |
|---|---|---|
| `decimal` | DECIMAL(P,S) | Exact; use for money |
| `float` | FLOAT | Approximate; avoid for money |
| `real` | DOUBLE | Approximate; rarely needed |

```xml
<column xsi:type="decimal"
        name="price"
        scale="4"
        precision="12"
        unsigned="false"
        nullable="false"
        default="0.0000"
        comment="Product price"/>
```

`precision` = total significant digits; `scale` = digits after decimal point.

#### String Types

| `xsi:type` | MySQL Type | Max Length |
|---|---|---|
| `varchar` | VARCHAR(N) | 65 535 bytes (row limit) |
| `char` | CHAR(N) | 255 characters |
| `text` | TEXT | 65 535 bytes |
| `mediumtext` | MEDIUMTEXT | 16 MB |
| `longtext` | LONGTEXT | 4 GB |
| `blob` | BLOB | 65 535 bytes |
| `mediumblob` | MEDIUMBLOB | 16 MB |

```xml
<column xsi:type="varchar"
        name="sku"
        nullable="false"
        length="64"
        comment="SKU"/>

<column xsi:type="text"
        name="description"
        nullable="true"
        comment="Long description"/>
```

#### Date / Time Types

| `xsi:type` | MySQL Type | Notes |
|---|---|---|
| `date` | DATE | Date only (YYYY-MM-DD) |
| `datetime` | DATETIME | No timezone awareness |
| `timestamp` | TIMESTAMP | Auto-converts to UTC; range 1970–2038 |

```xml
<column xsi:type="datetime"
        name="created_at"
        on_update="false"
        nullable="false"
        comment="Creation timestamp"/>

<column xsi:type="timestamp"
        name="updated_at"
        on_update="true"
        nullable="false"
        default="CURRENT_TIMESTAMP"
        comment="Last update timestamp"/>
```

`on_update="true"` maps to `ON UPDATE CURRENT_TIMESTAMP`.

#### Boolean

```xml
<column xsi:type="boolean"
        name="is_active"
        nullable="false"
        default="true"
        comment="Is active flag"/>
```

Maps to `SMALLINT(1)` in MySQL — Magento's convention for boolean columns.

### Constraint Types

#### Primary Key

```xml
<constraint xsi:type="primary" referenceId="PRIMARY">
    <column name="entity_id"/>
</constraint>
```

`referenceId` must always be `PRIMARY` for the primary key constraint.

#### Unique Key

```xml
<constraint xsi:type="unique" referenceId="VENDOR_MODULE_BRAND_URL_KEY">
    <column name="url_key"/>
</constraint>
```

`referenceId` becomes the constraint name in MySQL. Convention: uppercase table name + column names joined by underscores.

#### Foreign Key

```xml
<constraint xsi:type="foreign"
            referenceId="VENDOR_MODULE_BRAND_STORE_ID_STORE_STORE_ID"
            table="vendor_module_brand"
            column="store_id"
            referenceTable="store"
            referenceColumn="store_id"
            onDelete="CASCADE"
            onUpdate="NO ACTION"/>
```

| `onDelete` / `onUpdate` options | MySQL behaviour |
|---|---|
| `CASCADE` | Delete/update child rows when parent changes |
| `SET NULL` | Set child column to NULL |
| `NO ACTION` / `RESTRICT` | Prevent parent change if children exist |

### Index Types

| `xsi:type` | MySQL Type | Use Case |
|---|---|---|
| `index` | INDEX (BTREE) | General lookup |
| `fulltext` | FULLTEXT | Full-text search on text columns |

```xml
<index referenceId="VENDOR_MODULE_BRAND_NAME" indexType="btree">
    <column name="name"/>
</index>

<index referenceId="VENDOR_MODULE_PRODUCT_DESCRIPTION_FULLTEXT" indexType="fulltext">
    <column name="description"/>
</index>
```

> **Note:** Unique indexes are expressed as `<constraint xsi:type="unique">`, not `<index>`.

---

## 3. db\_schema.xml Examples

### 3.1 — Simple Table with Primary Key

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etcSchema.xsd">

    <table name="vendor_module_brand"
           resource="default"
           engine="innodb"
           comment="Brand entity table">

        <column xsi:type="int"
                name="brand_id"
                padding="10"
                unsigned="true"
                nullable="false"
                identity="true"
                comment="Brand ID"/>

        <column xsi:type="varchar"
                name="name"
                nullable="false"
                length="255"
                comment="Brand name"/>

        <column xsi:type="varchar"
                name="url_key"
                nullable="false"
                length="255"
                comment="URL key"/>

        <column xsi:type="boolean"
                name="is_active"
                nullable="false"
                default="true"
                comment="Is active"/>

        <column xsi:type="datetime"
                name="created_at"
                nullable="false"
                comment="Creation date"/>

        <column xsi:type="datetime"
                name="updated_at"
                nullable="true"
                comment="Last update date"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="brand_id"/>
        </constraint>

        <constraint xsi:type="unique" referenceId="VENDOR_MODULE_BRAND_URL_KEY">
            <column name="url_key"/>
        </constraint>

        <index referenceId="VENDOR_MODULE_BRAND_NAME" indexType="btree">
            <column name="name"/>
        </index>

    </table>

</schema>
```

### 3.2 — Table with Foreign Key to `store`

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etcSchema.xsd">

    <table name="vendor_module_brand_store"
           resource="default"
           engine="innodb"
           comment="Brand to store relationship">

        <column xsi:type="int"
                name="brand_id"
                padding="10"
                unsigned="true"
                nullable="false"
                comment="Brand ID"/>

        <column xsi:type="smallint"
                name="store_id"
                padding="5"
                unsigned="true"
                nullable="false"
                comment="Store ID"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="brand_id"/>
            <column name="store_id"/>
        </constraint>

        <!-- FK → vendor_module_brand.brand_id -->
        <constraint xsi:type="foreign"
                    referenceId="VENDOR_MODULE_BRAND_STORE_BRAND_ID_VENDOR_MODULE_BRAND_BRAND_ID"
                    table="vendor_module_brand_store"
                    column="brand_id"
                    referenceTable="vendor_module_brand"
                    referenceColumn="brand_id"
                    onDelete="CASCADE"
                    onUpdate="NO ACTION"/>

        <!-- FK → store.store_id -->
        <constraint xsi:type="foreign"
                    referenceId="VENDOR_MODULE_BRAND_STORE_STORE_ID_STORE_STORE_ID"
                    table="vendor_module_brand_store"
                    column="store_id"
                    referenceTable="store"
                    referenceColumn="store_id"
                    onDelete="CASCADE"
                    onUpdate="NO ACTION"/>

    </table>

</schema>
```

### 3.3 — Table with Decimal, Full-text Index, and Timestamps

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etcSchema.xsd">

    <table name="vendor_module_product_review"
           resource="default"
           engine="innodb"
           comment="Product review extended data">

        <column xsi:type="int"
                name="review_id"
                padding="10"
                unsigned="true"
                nullable="false"
                identity="true"
                comment="Review ID"/>

        <column xsi:type="int"
                name="product_id"
                padding="10"
                unsigned="true"
                nullable="false"
                comment="Product ID"/>

        <column xsi:type="decimal"
                name="rating"
                scale="2"
                precision="4"
                unsigned="true"
                nullable="false"
                default="0.00"
                comment="Numeric rating 0.00–5.00"/>

        <column xsi:type="varchar"
                name="title"
                nullable="false"
                length="255"
                comment="Review title"/>

        <column xsi:type="text"
                name="body"
                nullable="true"
                comment="Review body"/>

        <column xsi:type="timestamp"
                name="created_at"
                on_update="false"
                nullable="false"
                default="CURRENT_TIMESTAMP"
                comment="Created at"/>

        <column xsi:type="timestamp"
                name="updated_at"
                on_update="true"
                nullable="false"
                default="CURRENT_TIMESTAMP"
                comment="Updated at"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="review_id"/>
        </constraint>

        <constraint xsi:type="foreign"
                    referenceId="VENDOR_MODULE_PRODUCT_REVIEW_PRODUCT_ID_CATALOG_PRODUCT_ENTITY_ENTITY_ID"
                    table="vendor_module_product_review"
                    column="product_id"
                    referenceTable="catalog_product_entity"
                    referenceColumn="entity_id"
                    onDelete="CASCADE"
                    onUpdate="NO ACTION"/>

        <index referenceId="VENDOR_MODULE_PRODUCT_REVIEW_PRODUCT_ID" indexType="btree">
            <column name="product_id"/>
        </index>

        <!-- Full-text index for search -->
        <index referenceId="VENDOR_MODULE_PRODUCT_REVIEW_TITLE_BODY_FULLTEXT" indexType="fulltext">
            <column name="title"/>
            <column name="body"/>
        </index>

    </table>

</schema>
```

### 3.4 — Adding a Column to an Existing Table (Cross-module)

A module can *alter* tables owned by another module using its own `db_schema.xml` with the same table name. Magento merges the declarations:

```xml
<?xml version="1.0"?>
<!-- Vendor_BrandExtension extending Magento_Catalog's catalog_product_entity -->
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etcSchema.xsd">

    <table name="catalog_product_entity" resource="default" engine="innodb">
        <column xsi:type="int"
                name="brand_id"
                padding="10"
                unsigned="true"
                nullable="true"
                default="0"
                comment="Associated brand ID"/>

        <index referenceId="CATALOG_PRODUCT_ENTITY_BRAND_ID" indexType="btree">
            <column name="brand_id"/>
        </index>
    </table>

</schema>
```

> ⚠️ You must add the borrowed table + columns to **your module's whitelist** (`db_schema_whitelist.json`) so they are tracked correctly. See [Section 8](#8-whitelist-db_schema_whitelistjson).

---

## 4. Data Patches

### What Is a Data Patch?

A **Data Patch** is a PHP class that runs once during `setup:upgrade` and is responsible for inserting, updating, or deleting **data** in the database. It replaces the old `InstallData` / `UpgradeData` scripts.

Data patches implement `Magento\Framework\Setup\Patch\DataPatchInterface`:

```php
interface DataPatchInterface extends PatchInterface
{
    public function apply(): void;
}

interface PatchInterface
{
    public static function getDependencies(): array;
    public function getAliases(): array;
}
```

### Full Data Patch Example

```php
<?php

declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

/**
 * Seed default brand records on first install.
 */
class AddDefaultBrands implements DataPatchInterface
{
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup,
    ) {}

    /**
     * Patches that must be applied before this one.
     */
    public static function getDependencies(): array
    {
        // AddDefaultBrands has no dependencies
        return [];
    }

    /**
     * Aliases for renamed patch classes (backward compat).
     */
    public function getAliases(): array
    {
        return [];
    }

    public function apply(): void
    {
        $connection = $this->moduleDataSetup->getConnection();
        $tableName  = $this->moduleDataSetup->getTable('vendor_module_brand');

        $connection->insertMultiple($tableName, [
            [
                'name'       => 'Generic',
                'url_key'    => 'generic',
                'is_active'  => 1,
                'created_at' => date('Y-m-d H:i:s'),
            ],
            [
                'name'       => 'Premium',
                'url_key'    => 'premium',
                'is_active'  => 1,
                'created_at' => date('Y-m-d H:i:s'),
            ],
        ]);
    }
}
```

### Data Patch with EAV Attribute Registration

```php
<?php

declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Customer\Model\Customer;
use Magento\Customer\Setup\CustomerSetupFactory;
use Magento\Eav\Model\Entity\Attribute\SetFactory as AttributeSetFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class AddBrandPreferenceAttribute implements DataPatchInterface
{
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup,
        private readonly CustomerSetupFactory $customerSetupFactory,
        private readonly AttributeSetFactory $attributeSetFactory,
    ) {}

    public static function getDependencies(): array
    {
        return [];
    }

    public function getAliases(): array
    {
        return [];
    }

    public function apply(): void
    {
        $customerSetup = $this->customerSetupFactory->create(
            ['setup' => $this->moduleDataSetup]
        );

        $customerEntity   = $customerSetup->getEavConfig()->getEntityType(Customer::ENTITY);
        $attributeSetId   = $customerEntity->getDefaultAttributeSetId();
        $attributeSet     = $this->attributeSetFactory->create();
        $attributeGroupId = $attributeSet->getDefaultGroupId($attributeSetId);

        $customerSetup->addAttribute(Customer::ENTITY, 'brand_preference', [
            'type'               => 'int',
            'label'              => 'Preferred Brand',
            'input'              => 'select',
            'required'           => false,
            'visible'            => true,
            'user_defined'       => true,
            'sort_order'         => 1000,
            'position'           => 1000,
            'system'             => false,
            'source'             => \Vendor\Module\Model\Config\Source\Brand::class,
        ]);

        $attribute = $customerSetup->getEavConfig()
            ->getAttribute(Customer::ENTITY, 'brand_preference')
            ->addData([
                'attribute_set_id'   => $attributeSetId,
                'attribute_group_id' => $attributeGroupId,
                'used_in_forms'      => ['adminhtml_customer', 'customer_account_edit'],
            ]);

        $attribute->save();
    }
}
```

### Patch Dependencies

When patch B requires patch A to have run first, declare the dependency:

```php
<?php

declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class UpdateBrandSlugs implements DataPatchInterface
{
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup,
    ) {}

    /**
     * AddDefaultBrands must run first so rows exist to update.
     */
    public static function getDependencies(): array
    {
        return [
            AddDefaultBrands::class,  // fully-qualified class name
        ];
    }

    public function getAliases(): array
    {
        return [];
    }

    public function apply(): void
    {
        $connection = $this->moduleDataSetup->getConnection();
        $table      = $this->moduleDataSetup->getTable('vendor_module_brand');

        // Regenerate url_key from name for all rows
        $rows = $connection->fetchAll(
            $connection->select()->from($table, ['brand_id', 'name'])
        );

        foreach ($rows as $row) {
            $urlKey = strtolower(preg_replace('/[^a-z0-9]+/i', '-', $row['name']));
            $connection->update(
                $table,
                ['url_key' => trim($urlKey, '-')],
                ['brand_id = ?' => (int) $row['brand_id']]
            );
        }
    }
}
```

### When to Use DataPatch vs SchemaPatch

| Scenario | Use |
|---|---|
| Inserting seed/default data | `DataPatch` |
| Migrating existing row data | `DataPatch` |
| Registering EAV attributes | `DataPatch` |
| Adding/modifying a column | `db_schema.xml` (preferred) |
| Complex DDL not expressible in XML | `SchemaPatch` |
| Copying data between columns before dropping one | `DataPatch` (run *before* schema change) |

---

## 5. Schema Patches

### What Is a Schema Patch?

A **Schema Patch** is a PHP class that performs **one-time DDL operations** that cannot be expressed declaratively in `db_schema.xml`. It implements `Magento\Framework\Setup\Patch\SchemaPatchInterface`.

```php
interface SchemaPatchInterface extends PatchInterface
{
    public function apply(): void;
}
```

### When to Use SchemaPatch

- Complex `SELECT INTO` or data-dependent DDL
- Creating stored procedures or triggers (rare in Magento)
- Running DDL that depends on runtime logic impossible to express in XML
- Migrating large tables where you need per-batch control

### Schema Patch Example — Migrating a Column

Scenario: The old `vendor_module_brand` table stores `status` as a `varchar('enabled','disabled')`. We want to convert it to a `smallint`. The XML declares the final `smallint` state, but we need a patch to convert existing data *before* the column type changes.

**Step 1 — Schema Patch to copy data:**

```php
<?php

declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Schema;

use Magento\Framework\DB\Adapter\AdapterInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\SchemaPatchInterface;

/**
 * Converts varchar status column data to tinyint before
 * db_schema.xml changes the column type.
 *
 * NOTE: This patch must run BEFORE the declarative schema applies
 * the column type change. Magento applies schema patches after
 * db_schema.xml DDL, so we add a temporary column approach here.
 */
class MigrateStatusColumn implements SchemaPatchInterface
{
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup,
    ) {}

    public static function getDependencies(): array
    {
        return [];
    }

    public function getAliases(): array
    {
        return [];
    }

    public function apply(): void
    {
        $connection = $this->moduleDataSetup->getConnection();
        $table      = $this->moduleDataSetup->getTable('vendor_module_brand');

        // Guard: only run if old varchar column still exists
        $describe = $connection->describeTable($table);
        if (!isset($describe['status']) || $describe['status']['DATA_TYPE'] !== 'varchar') {
            return;
        }

        // Add temporary integer column
        $connection->addColumn($table, 'status_int', [
            'type'     => AdapterInterface::TYPE_SMALLINT,
            'unsigned' => true,
            'nullable' => false,
            'default'  => 0,
            'comment'  => 'Temporary migration column',
        ]);

        // Populate integer values from varchar
        $connection->update($table, ['status_int' => 1], "status = 'enabled'");
        $connection->update($table, ['status_int' => 0], "status = 'disabled'");

        // Drop old varchar column
        $connection->dropColumn($table, 'status');

        // Rename temp column to status
        $connection->changeColumn($table, 'status_int', 'status', [
            'type'     => AdapterInterface::TYPE_SMALLINT,
            'unsigned' => true,
            'nullable' => false,
            'default'  => 0,
            'comment'  => 'Brand status',
        ]);
    }
}
```

> ⚠️ Schema patches run **after** `db_schema.xml` DDL is applied. If the column type change in XML would break data migration, keep the old column type in XML until the patch runs, then update the XML in the same release. Coordinate carefully.

### Schema Patch vs db\_schema.xml Decision Tree

```
Need to change database structure?
         │
         ▼
Is it a column/index/FK/table change
expressible purely in XML?
    ├── YES ──► Edit db_schema.xml (preferred)
    └── NO
         │
         ▼
Does it require reading/transforming
existing row data as part of DDL?
    ├── YES ──► SchemaPatch (+ possibly DataPatch for data)
    └── NO
         │
         ▼
Is it a one-time structural change
needed for correctness?
    ├── YES ──► SchemaPatch
    └── NO ───► Re-examine — db_schema.xml likely works
```

---

## 6. Recurring Patches

### What Is a Recurring Patch?

A **Recurring Patch** runs on **every** `setup:upgrade` execution — not just once. It implements `Magento\Framework\Setup\Patch\RecurringPatchInterface` (which extends `DataPatchInterface`):

```php
interface RecurringPatchInterface extends DataPatchInterface
{
    // No additional methods — the recurring behaviour is inferred
    // from the interface type at runtime
}
```

### When to Use Recurring Patches

| Use Case | Example |
|---|---|
| Enforcing data integrity invariants | Ensure required config values always exist |
| Keeping reference data in sync | Refresh a lookup table from an external source |
| Clearing stale cache entries | Flush specific cache tags after every upgrade |
| Re-applying computed fields | Regenerate denormalised columns |

### Recurring Patch Example — Ensuring Config Values Exist

```php
<?php

declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\App\Config\Storage\WriterInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\Patch\RecurringPatchInterface;

/**
 * Ensures mandatory configuration values are present after every upgrade.
 * Runs on every bin/magento setup:upgrade.
 */
class EnsureDefaultConfig implements RecurringPatchInterface
{
    private const DEFAULTS = [
        'vendor_module/general/enabled'     => '1',
        'vendor_module/general/batch_size'  => '500',
        'vendor_module/email/template'      => 'vendor_module_email_template',
    ];

    public function __construct(
        private readonly WriterInterface $configWriter,
        private readonly \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig,
    ) {}

    public static function getDependencies(): array
    {
        return [];
    }

    public function getAliases(): array
    {
        return [];
    }

    public function apply(): void
    {
        foreach (self::DEFAULTS as $path => $value) {
            $current = $this->scopeConfig->getValue($path);
            if ($current === null) {
                $this->configWriter->save($path, $value);
            }
        }
    }
}
```

> **Important:** Because recurring patches run every time, they must be **idempotent** — calling `apply()` twice must produce the same end state as calling it once.

### Difference Between Patch Types at a Glance

| Type | Interface | Runs When | Tracked in DB? |
|---|---|---|---|
| Data Patch | `DataPatchInterface` | Once, on first `setup:upgrade` after class appears | ✅ Yes (`patch_list` table) |
| Schema Patch | `SchemaPatchInterface` | Once, on first `setup:upgrade` after class appears | ✅ Yes (`patch_list` table) |
| Recurring Patch | `RecurringPatchInterface` | Every `setup:upgrade` | ❌ No |

---

## 7. Patch Registration & Execution Order

### Auto-Discovery

Patches are discovered automatically — **no registration in XML is required**. Magento scans:

```
{Module}/Setup/Patch/Data/     → DataPatchInterface implementations
{Module}/Setup/Patch/Schema/   → SchemaPatchInterface implementations
```

Any class in these directories that implements the correct interface is picked up.

### The `patch_list` Table

Applied patches are recorded in the `patch_list` database table:

```sql
DESCRIBE patch_list;
-- patch_id   int(10) unsigned NOT NULL AUTO_INCREMENT
-- patch_name varchar(1024) NOT NULL   ← fully-qualified class name

SELECT patch_name FROM patch_list WHERE patch_name LIKE '%Vendor_Module%';
-- Vendor\Module\Setup\Patch\Data\AddDefaultBrands
-- Vendor\Module\Setup\Patch\Data\UpdateBrandSlugs
```

### Execution Order Rules

1. **Dependencies first** — If patch B declares `getDependencies()` returning `[A::class]`, Magento guarantees A runs before B within the same `setup:upgrade` invocation.
2. **No explicit ordering without dependencies** — Patches without declared dependencies may run in any order relative to each other. Never assume alphabetical.
3. **Cross-module ordering** — Magento respects module load order (`module.xml` `<sequence>`). If module B depends on module A in `module.xml`, A's patches run first.
4. **Already-applied patches are skipped** — Magento checks `patch_list` before running. If the class name is already recorded, the patch is skipped entirely.

### Forcing Re-execution (Development Only)

```bash
# Remove a specific patch record (DEV ONLY — never in production)
DELETE FROM patch_list WHERE patch_name = 'Vendor\\Module\\Setup\\Patch\\Data\\AddDefaultBrands';

# Then re-run
bin/magento setup:upgrade
```

---

## 8. Whitelist (db\_schema\_whitelist.json)

### What Is the Whitelist?

The whitelist is a JSON file that **enumerates every table, column, constraint, and index** that your module's `db_schema.xml` declares ownership of. It enables Magento to safely **drop** those database objects when the module is uninstalled.

Without the whitelist, Magento cannot distinguish between:
- A column intentionally removed from `db_schema.xml` (should be dropped from DB)
- A column that was never in this module's schema (should be ignored)

File location:
```
app/code/{Vendor}/{Module}/etc/db_schema_whitelist.json
```

### Auto-generation Command

**Always generate the whitelist automatically** — never write it by hand:

```bash
# Generate whitelist for a specific module
bin/magento setup:db-declaration:generate-whitelist --module-name=Vendor_Module

# Regenerate all whitelists (slower but thorough)
bin/magento setup:db-declaration:generate-whitelist
```

Run this command **every time you add or remove elements in `db_schema.xml`**, then commit the updated JSON alongside your XML changes.

### Example Whitelist

```json
{
    "vendor_module_brand": {
        "column": {
            "brand_id": true,
            "name": true,
            "url_key": true,
            "is_active": true,
            "created_at": true,
            "updated_at": true
        },
        "constraint": {
            "PRIMARY": true,
            "VENDOR_MODULE_BRAND_URL_KEY": true
        },
        "index": {
            "VENDOR_MODULE_BRAND_NAME": true
        }
    },
    "vendor_module_brand_store": {
        "column": {
            "brand_id": true,
            "store_id": true
        },
        "constraint": {
            "PRIMARY": true,
            "VENDOR_MODULE_BRAND_STORE_BRAND_ID_VENDOR_MODULE_BRAND_BRAND_ID": true,
            "VENDOR_MODULE_BRAND_STORE_STORE_ID_STORE_STORE_ID": true
        }
    }
}
```

### Whitelist and Column Removal

When you **remove a column** from `db_schema.xml`:

1. The column's entry in the whitelist signals to Magento: "this column was previously declared by this module."
2. Magento knows it should emit `ALTER TABLE ... DROP COLUMN ...`.
3. If the column were absent from both the XML *and* the whitelist, Magento would simply ignore it (assume it belongs to another module).

This is why regenerating the whitelist after every schema change is critical.

### Dry Run

Preview DDL that will be applied without touching the database:

```bash
bin/magento setup:upgrade --dry-run

# Output is written to:
# var/log/dry-run-installation/declarative_schema_dry_run_log.log
```

---

## 9. Declarative Schema vs Old Setup Scripts

The table below maps common tasks to old-style PHP code vs the current declarative approach.

### 9.1 — Creating a Table

**Old (UpgradeSchema.php):**

```php
// Setup/UpgradeSchema.php
public function upgrade(SchemaSetupInterface $setup, ModuleContextInterface $context): void
{
    if (version_compare($context->getVersion(), '1.0.0', '<')) {
        $setup->startSetup();

        $table = $setup->getConnection()->newTable(
            $setup->getTable('vendor_module_brand')
        )->addColumn(
            'brand_id',
            \Magento\Framework\DB\Ddl\Table::TYPE_INTEGER,
            null,
            ['identity' => true, 'unsigned' => true, 'nullable' => false, 'primary' => true],
            'Brand ID'
        )->addColumn(
            'name',
            \Magento\Framework\DB\Ddl\Table::TYPE_TEXT,
            255,
            ['nullable' => false],
            'Brand Name'
        )->addColumn(
            'url_key',
            \Magento\Framework\DB\Ddl\Table::TYPE_TEXT,
            255,
            ['nullable' => false],
            'URL Key'
        )->addIndex(
            $setup->getIdxName('vendor_module_brand', ['url_key'], \Magento\Framework\DB\Adapter\AdapterInterface::INDEX_TYPE_UNIQUE),
            ['url_key'],
            ['type' => \Magento\Framework\DB\Adapter\AdapterInterface::INDEX_TYPE_UNIQUE]
        )->setComment('Brand entity table');

        $setup->getConnection()->createTable($table);
        $setup->endSetup();
    }
}
```

**New (db\_schema.xml):**

```xml
<table name="vendor_module_brand" resource="default" engine="innodb" comment="Brand entity table">
    <column xsi:type="int" name="brand_id" padding="10" unsigned="true" nullable="false" identity="true" comment="Brand ID"/>
    <column xsi:type="varchar" name="name" nullable="false" length="255" comment="Brand Name"/>
    <column xsi:type="varchar" name="url_key" nullable="false" length="255" comment="URL Key"/>
    <constraint xsi:type="primary" referenceId="PRIMARY">
        <column name="brand_id"/>
    </constraint>
    <constraint xsi:type="unique" referenceId="VENDOR_MODULE_BRAND_URL_KEY">
        <column name="url_key"/>
    </constraint>
</table>
```

✅ 60% fewer lines. Readable at a glance. No version branching.

### 9.2 — Modifying a Column

**Old (UpgradeSchema.php):**

```php
if (version_compare($context->getVersion(), '1.1.0', '<')) {
    $setup->getConnection()->modifyColumn(
        $setup->getTable('vendor_module_brand'),
        'name',
        [
            'type'   => \Magento\Framework\DB\Ddl\Table::TYPE_TEXT,
            'length' => 512,   // was 255, now 512
        ]
    );
}
```

**New (db\_schema.xml):**

Simply change the attribute:

```xml
<!-- Before -->
<column xsi:type="varchar" name="name" nullable="false" length="255" comment="Brand Name"/>

<!-- After — just update length -->
<column xsi:type="varchar" name="name" nullable="false" length="512" comment="Brand Name"/>
```

Then run `bin/magento setup:upgrade`. Magento generates and executes:
```sql
ALTER TABLE `vendor_module_brand` MODIFY COLUMN `name` varchar(512) NOT NULL COMMENT 'Brand Name';
```

### 9.3 — Adding an Index

**Old:**

```php
$setup->getConnection()->addIndex(
    $setup->getTable('vendor_module_brand'),
    $setup->getIdxName('vendor_module_brand', ['name']),
    ['name']
);
```

**New:**

```xml
<index referenceId="VENDOR_MODULE_BRAND_NAME" indexType="btree">
    <column name="name"/>
</index>
```

### 9.4 — Adding a Foreign Key

**Old:**

```php
$setup->getConnection()->addForeignKey(
    $setup->getFkName('vendor_module_brand_store', 'brand_id', 'vendor_module_brand', 'brand_id'),
    $setup->getTable('vendor_module_brand_store'),
    'brand_id',
    $setup->getTable('vendor_module_brand'),
    'brand_id',
    \Magento\Framework\DB\Ddl\Table::ACTION_CASCADE
);
```

**New:**

```xml
<constraint xsi:type="foreign"
            referenceId="VENDOR_MODULE_BRAND_STORE_BRAND_ID_VENDOR_MODULE_BRAND_BRAND_ID"
            table="vendor_module_brand_store"
            column="brand_id"
            referenceTable="vendor_module_brand"
            referenceColumn="brand_id"
            onDelete="CASCADE"
            onUpdate="NO ACTION"/>
```

### 9.5 — Migration Summary Table

| Task | Old Approach | New Approach |
|---|---|---|
| Create table | `newTable()` + `createTable()` in `UpgradeSchema` | `<table>` in `db_schema.xml` |
| Add column | `addColumn()` in `UpgradeSchema` | Add `<column>` to `db_schema.xml` |
| Modify column | `modifyColumn()` in `UpgradeSchema` | Edit `<column>` attributes in `db_schema.xml` |
| Drop column | `dropColumn()` in `UpgradeSchema` | Remove `<column>` from `db_schema.xml` (whitelist required) |
| Add index | `addIndex()` in `UpgradeSchema` | Add `<index>` to `db_schema.xml` |
| Add unique key | `addIndex(..., UNIQUE)` in `UpgradeSchema` | Add `<constraint xsi:type="unique">` |
| Add foreign key | `addForeignKey()` in `UpgradeSchema` | Add `<constraint xsi:type="foreign">` |
| Drop table | `dropTable()` in `UpgradeSchema` | Remove `<table>` from `db_schema.xml` (whitelist required) |
| Insert data | `InstallData` / `UpgradeData` | `DataPatch` |
| One-time DDL | Versioned `UpgradeSchema` block | `SchemaPatch` |
| Repeating maintenance | _(no equivalent)_ | `RecurringPatch` |

---

## 10. Indexer Invalidation

### How Declarative Schema Affects Indexers

When `setup:upgrade` applies DDL changes through declarative schema, Magento automatically **invalidates** indexers that are known to depend on the modified tables. This is controlled by the Indexer framework's dependency graph, not by the schema system directly.

After `setup:upgrade` completes you will typically see:

```bash
bin/magento indexer:status
# catalog_category_product   Status: invalid
# catalog_product_attribute  Status: invalid
# catalog_product_price      Status: invalid
# catalogrule_rule           Status: invalid
```

### Indexer Dependency Declaration

Each indexer declares its dependencies in `indexer.xml`:

```xml
<!-- Magento/Catalog/etc/indexer.xml (simplified) -->
<indexer id="catalog_product_attribute"
         view_id="catalog_product_attribute"
         class="Magento\Catalog\Model\Indexer\Product\Eav"
         primary="catalog_product">
    <dependencies>
        <indexer id="catalog_product_category"/>
    </dependencies>
</indexer>
```

When you add a column to `catalog_product_entity` via a cross-module `db_schema.xml` addition, Magento recognises the table is touched and invalidates all indexers registered against it.

### Reindex After Setup Upgrade

Best practice after any `setup:upgrade` that modifies tables used by indexers:

```bash
# Reindex all invalidated indexers
bin/magento indexer:reindex

# Or target specific indexers
bin/magento indexer:reindex catalog_product_attribute catalog_product_price

# Check status afterward
bin/magento indexer:status
```

### Configuring Indexer Mode

Indexers can run in two modes:

```bash
# Real-time (on-save) — each save triggers partial reindex
bin/magento indexer:set-mode realtime catalog_product_price

# Scheduled (cron) — changes batched; indexer runs via cron
bin/magento indexer:set-mode schedule catalog_product_price
```

For high-traffic stores, **schedule mode** is strongly recommended. Declarative schema changes always require a full manual reindex regardless of mode setting.

### Cache Types After Schema Changes

After DDL changes it is good practice to flush caches:

```bash
bin/magento cache:flush

# Or target specific cache types
bin/magento cache:clean config block_html full_page
```

The `config` cache type is particularly important because EAV attribute metadata is cached there. Any Data Patch that registers EAV attributes must be followed by a config cache flush (Magento does this automatically at the end of `setup:upgrade`).

### Checklist After Running setup:upgrade

```
✅  bin/magento setup:upgrade
✅  bin/magento setup:di:compile          (production only)
✅  bin/magento setup:static-content:deploy (production only)
✅  bin/magento indexer:reindex           (if indexers invalidated)
✅  bin/magento cache:flush
```

---

## Quick Reference Card

### CLI Commands

```bash
# Apply all pending schema + patch changes
bin/magento setup:upgrade

# Dry-run (preview DDL without applying)
bin/magento setup:upgrade --dry-run

# Generate/update whitelist JSON
bin/magento setup:db-declaration:generate-whitelist --module-name=Vendor_Module

# Check which patches have been applied
SELECT patch_name FROM patch_list ORDER BY patch_id;

# List indexer statuses
bin/magento indexer:status

# Reindex all
bin/magento indexer:reindex

# Flush all caches
bin/magento cache:flush
```

### File Locations

```
app/code/Vendor/Module/
├── etc/
│   ├── db_schema.xml                   ← declarative schema
│   └── db_schema_whitelist.json        ← auto-generated whitelist
└── Setup/
    └── Patch/
        ├── Data/
        │   ├── AddDefaultBrands.php    ← DataPatchInterface
        │   ├── UpdateBrandSlugs.php    ← DataPatchInterface (depends on above)
        │   └── EnsureDefaultConfig.php ← RecurringPatchInterface
        └── Schema/
            └── MigrateStatusColumn.php ← SchemaPatchInterface
```

### Column Type Quick Reference

```
Integer:   smallint · int · bigint
Decimal:   decimal(precision, scale) · float · real
String:    char · varchar(length) · text · mediumtext · longtext
Binary:    blob · mediumblob
Date/Time: date · datetime · timestamp
Boolean:   boolean  (→ SMALLINT(1))
```

---

*Article part of the **2025-Q4-magento2-deep-dive** course | Rank 24 | Last updated: 2025*