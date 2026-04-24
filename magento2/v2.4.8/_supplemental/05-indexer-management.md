---
title: "34 - Indexer Management"
description: "Deep dive into Magento 2.4.8 indexers: indexer architecture, MVIEW mechanism, schedule vs realtime mode, custom indexer development, and indexer optimization."
tags: magento2, indexers, mview, indexer-architecture, schedule-mode, realtime-mode, custom-indexer
rank: 5
pathways: [magento2-deep-dive]
---

# 34 - Indexer Management

Magento 2's indexer subsystem transforms normalized database structures into optimized, denormalized data representations that power high-performance storefront queries. Understanding indexers is essential for any Magento developer working with 2.4.8, as misconfigured or stale indexers are a leading cause of slow product pages, broken search, and inventory discrepancies.

## 1. Indexer Architecture

### Why Indexers Exist: The Denormalization Imperative

Relational database normalization eliminates redundancy—but query performance suffers when you need to join 5-7 tables to display a product listing. Magento's indexers pre-compute these joins and store the results in dedicated index tables, enabling single-table queries that execute in milliseconds rather than seconds.

### The catalog_product_price Join Problem

Consider the price displayed on a category listing page. In a fully normalized schema, rendering one product's price requires joining:

```
catalog_product_entity
    → catalog_product_entity_int (price attribute)
    → eav_attribute_option_value (attribute dropdown labels)
    → customer_group (price scoping)
    → catalog_product_index_price (price tier data)
    → catalog_product_index_price_idx (final price calculation)
```

This 6-table join runs once per product per page render. At 30 products per page × 100 concurrent shoppers, you're executing 3,000 complex joins per second. Indexers eliminate this by materializing the computed price into a single `catalog_product_index_price` table that joins to `catalog_product_entity` with a simple `product_id = entity_id` predicate.

### Flat Tables vs EAV

Magento historically shipped with EAV (Entity-Attribute-Value) architecture, which offered schema flexibility at the cost of query complexity. The `catalog_product_flat` indexer addresses this by creating a denormalized table where every attribute becomes a column:

```sql
-- Before (EAV): Fetching 'name' requires 3-way join
SELECT value FROM catalog_product_entity_varchar 
WHERE entity_id = 123 AND attribute_id = (
    SELECT attribute_id FROM eav_attribute 
    WHERE attribute_code = 'name'
);

-- After (Flat table): Direct column access
SELECT name FROM catalog_product_flat_1 WHERE entity_id = 123;
```

In M2.4.8, flat tables remain important for catalog queries but are gradually being superseded by EAV-optimized entity DQL repositories that leverage proper indexing without requiring full flat denormalization.

### Indexer Table Design Pattern

Each indexer follows a predictable table lifecycle:

```php
// Magento\Framework\Mview\ViewInterface implementation pattern
interface ViewInterface
{
    public function getId(): string;
    public function getChangelogName(): string;
    public function getTargetName(): string;
    public function createChangelog(): void;
    public function dropChangelog(): void;
    public function refresh(): void;
}
```

The indexer writes to a `_idx` table during processing, atomically renames it to the `_cl` (current) table upon completion, and maintains a `_changelog` table that tracks pending changes for async processing.

## 2. Indexer Types

### MODE_REALTIME vs MODE_SCHEDULE

Magento 2 supports two indexer execution modes:

| Mode | Class Constant | Behavior | Use Case |
|------|----------------|----------|----------|
| Realtime | `MODE_REAL_TIME` | Indexer runs immediately on entity save | Small catalogs, development |
| Schedule | `MODE_SCHEDULE` | Changes queued in changelog, processed via cron | Production with large catalogs |

```php
// Magento\Indexer\Model\Mode
class Mode
{
    public const MODE_REAL_TIME = 'real-time';
    public const MODE_SCHEDULE = 'schedule';

    public function getModes(): array
    {
        return [self::MODE_REAL_TIME, self::MODE_SCHEDULE];
    }

    public function isScheduleMode(): bool
    {
        return $this->_config->getValue('indexer') === self::MODE_SCHEDULE;
    }
}
```

The mode is stored in `mode` column of `mview_state` table and can be inspected via:

```bash
mysql> SELECT indexer_id, mode FROM mview_state;
+------------------+--------+
| indexer_id       | mode   |
+------------------+--------+
| catalog_product_price | schedule |
| catalogsearch_fulltext | schedule |
+------------------+--------+
```

### indexer.xml Configuration

Custom indexers declare themselves in `etc/indexer.xml`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Indexer/etc/indexer.xsd">
    <indexer id="custom_product_indexer" view_id="custom_indexer" class="Vendor\Module\Indexer\CustomProductIndexer">
        <title translate="true">Custom Product Indexer</title>
        <description translate="true">Indexes custom product attributes for fast display</description>
    </indexer>
</config>
```

The `id` attribute must be unique across all indexers. The `view_id` references the corresponding entry in `mview.xml`. The `class` attribute specifies the `IndexerInterface` implementation.

### IndexerInterface Contract

All indexers must implement `Magento\Framework\Indexer\IndexerInterface`:

```php
<?php
declare(strict_types=1);

namespace Magento\Framework\Indexer;

interface IndexerInterface
{
    /**
     * Get indexer identifier
     */
    public function getId(): string;

    /**
     * Get indexer name
     */
    public function getName(): string;

    /**
     * Get indexer description
     */
    public function getDescription(): string;

    /**
     * Execute mass reindex
     *
     * @param int[] $ids Null means all entities
     * @return void
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function executeMass(array $ids): void;

    /**
     * Execute single reindex
     *
     * @param int $id
     * @return void
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function execute($id): void;

    /**
     * Execute full reindex
     *
     * @return void
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function executeFull(): void;
}
```

The abstract base class `Magento\Framework\Indexer\Action` provides shared functionality:

```php
<?php
declare(strict_types=1);

namespace Magento\Framework\Indexer;

use Magento\Framework\DataObject;

abstract class Action
{
    /**
     * Execute row reindex
     */
    abstract public function executeRow(int $id): void;

    /**
     * Execute list reindex
     */
    abstract public function executeList(array $ids): void;

    /**
     * Execute full reindex
     */
    abstract public function executeFull(): void;
}
```

## 3. MVIEW Mechanism

### The Change Tracking Problem

Imagine a product's price changes. To update the price index, you need to know:
1. Which products changed
2. Which indexers depend on that entity
3. When to run the indexer (immediately vs batched)

MVIEW (Materialized View) solves this through a changelog-based architecture.

### View Interface and Implementation

`Magento\Framework\Mview\View` manages the relationship between source entity changes and target index updates:

```php
<?php
declare(strict_types=1);

namespace Magento\Framework\Mview;

use Magento\Framework\App\ResourceConnection;

class View implements ViewInterface
{
    private string $id;
    private string $changelogName;
    private string $targetName;
    private Connection $connection;
    private ConfigInterface $config;

    public function __construct(
        \Magento\Framework\Mview\ConfigInterface $config,
        \Magento\Framework\App\ResourceConnection $resource,
        string $viewId
    ) {
        $this->id = $viewId;
        $this->connection = $resource->getConnection();
        $this->config = $config->getViewConfig($viewId);
        $this->changelogName = $this->config->getChangelogName();
        $this->targetName = $this->config->getTargetName();
    }

    public function getId(): string
    {
        return $this->id;
    }

    public function getChangelogName(): string
    {
        return $this->changelogName;
    }

    public function getTargetName(): string
    {
        return $this->targetName;
    }

    /**
     * Process pending changes from changelog
     */
    public function update(): void
    {
        $select = $this->connection->select()
            ->from($this->changelogName, ['entity_id'])
            ->where('version_id > ?', $this->getLastVersionId());

        $ids = $this->connection->fetchCol($select);

        if (!empty($ids)) {
            $this->getIndexer()->execute($ids);
            $this->markAsProcessed($ids);
        }
    }
}
```

### Changelog Table Structure

Each MVIEW creates a corresponding changelog table with a predictable schema:

```sql
-- Changelog table for catalog_product_price indexer
CREATE TABLE catalog_product_price_cl (
    version_id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    entity_id INT UNSIGNED NOT NULL,
    operation varchar(10),  -- 'save' | 'delete'
    version_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_entity (entity_id),
    INDEX idx_version (version_id)
) ENGINE=InnoDB;
```

The `version_id` column provides monotonic versioning for change detection. When cron runs `indexer:update`, it queries:

```sql
SELECT entity_id FROM catalog_product_price_cl 
WHERE version_id > (SELECT MAX(version_id) FROM mview_state WHERE view_id = 'catalog_product_price');
```

This returns only the changes since last processing—no full table scan required.

### Trigger-Based Change Capture

Entity changes populate the changelog via database triggers:

```sql
-- Automatically generated trigger for catalog_product_entity
DELIMITER //
CREATE TRIGGER trg_catalog_product_entity_update
AFTER UPDATE ON catalog_product_entity
FOR EACH ROW
BEGIN
    INSERT INTO catalog_product_price_cl (entity_id, operation, version_time)
    VALUES (NEW.entity_id, 'save', NOW());
END//
DELIMITER ;
```

Magento generates these triggers in `Magento\Framework\Mview\TriggerFactory` during indexer initialization. The trigger inserts a row into the changelog table for every entity modification.

### mview.xml Configuration

The `etc/mview.xml` file configures which entities the MVIEW tracks:

```xml
<?xml version="1.0"?>
<mview xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:noNamespaceSchemaLocation="urn:magento:framework:Mview/etc/mview.xsd">
    <view id="catalog_product_price" 
          class="Magento\Catalog\Model\Indexer\Product\Price" 
          group="catalog">
        <subscriptions>
            <table name="catalog_product_entity" entity="product" />
        </subscriptions>
    </view>
    
    <view id="catalogsearch_fulltext" 
          class="Magento\CatalogSearch\Model\Indexer\Fulltext" 
          group="search">
        <subscriptions>
            <table name="catalog_product_entity" entity="product" />
            <table name="catalog_product_website" entity="product" />
            <table name="stock_item" entity="product" />
        </subscriptions>
    </view>
</mview>
```

Each `table` element under `subscriptions` creates a trigger that fires on INSERT/UPDATE/DELETE of that table, inserting a row into the changelog.

## 4. Built-in Indexers

Magento 2.4.8 ships with the following indexers:

| Indexer ID | Class | Mode | Purpose |
|------------|-------|------|---------|
| `catalog_product_price` | `Magento\Catalog\Model\Indexer\Product\Price` | Schedule | Price calculations including tier pricing, group pricing |
| `catalog_product_flat` | `Magento\Catalog\Model\Indexer\Product\Flat` | Schedule | Denormalized flat tables for fast product queries |
| `catalog_category_flat` | `Magento\Catalog\Model\Indexer\Category\Flat` | Schedule | Flat category tables |
| `catalog_category_product` | `Magento\Catalog\Model\Indexer\Category\Product` | Schedule | Category-product associations |
| `catalogsearch_fulltext` | `Magento\CatalogSearch\Model\Indexer\Fulltext` | Schedule | Full-text search index |
| `catalogrule_rule` | `Magento\CatalogRule\Model\Indexer\Rule` | Schedule | Catalog price rule conditions |
| `catalogrule_product` | `Magento\CatalogRule\Model\Indexer\Product\Rule` | Schedule | Applied catalog rule calculations |
| `inventory` | `Magento\InventoryIndexer\Model\Indexer` | Schedule | MSI stock indices |
| `customer_grid` | `Magento\Customer\Model\Indexer\Grid` | Schedule | Customer grid denormalization |
| `design_config_grid` | `Magento\Theme\Model\Indexer\Design\Config` | Schedule | Design configuration |
| `ee_swatch_product` | `Magento\Swatches\Model\Indexer\Product` | Schedule | Swatch attribute values |
| `url_rewrite` | `Magento\UrlRewrite\Model\Indexer\UrlRewrite` | Schedule | URL rewrite generation |
| `catalogrule_rule` | `Magento\CatalogRule\Model\Indexer\Rule` | Schedule | Catalog price rule conditions |
| `giftcard` | `Magento\GiftCard\Model\Indexer\GiftCard` | Schedule | Gift card product attributes |

### Product Price Indexer Deep Dive

The price indexer is arguably the most complex in Magento. Its structure:

```
catalog_product_index_price
├── entity_id (PK)
├── website_id (PK)
├── customer_group_id (PK)
├── version_id
├── price (final calculated price)
├── final_price
├── minimal_price
├── min_price
├── max_price
├── tier_price
└── group_price
```

The `Price` indexer (`Magento\Catalog\Model\Indexer\Product\Price`) implements `ExecuteByCollectionStrategyInterface` to process products in batches:

```php
<?php
declare(strict_types=1);

namespace Magento\Catalog\Model\Indexer\Product;

class Price extends \Magento\Framework\Indexer\Action
    implements \Magento\Framework\Indexer\SaveHandler\GridInterface
{
    public function __construct(
        \Magento\Catalog\Model\Product\Type $productType,
        \Magento\Customer\Model\Group $groupModel,
        \Magento\Catalog\Model\ResourceModel\Product $productResource,
        \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig
    ) {
        $this->productType = $productType;
        $this->groupModel = $groupModel;
        $this->productResource = $productResource;
        $this->scopeConfig = $scopeConfig;
    }

    public function executeFull(): void
    {
        $this->reindexAll();
    }

    public function executeList(array $ids): void
    {
        if (!empty($ids)) {
            $this->reindex($ids);
        }
    }

    public function executeRow(int $id): void
    {
        $this->reindex([$id]);
    }

    private function reindex(array $productIds): void
    {
        foreach ($this->productType->getTypes() as $typeId => $typeInfo) {
            $linkedProductIds = $this->getLinkedProductIds($productIds, $typeId);
            $this->reindexProcess($linkedProductIds, $typeId);
        }
    }

    private function reindexProcess(array $productIds, string $typeId): void
    {
        $batchSize = $this->getBatchSize();
        $chunks = array_chunk($productIds, $batchSize);

        foreach ($chunks as $chunk) {
            $this->reindexPrices($chunk, $typeId);
        }
    }
}
```

### Category Product Indexer

The `catalog_category_product` indexer maintains the `catalog_category_product_index` table that powers the category product listing:

```php
<?php
declare(strict_types=1);

namespace Magento\Catalog\Model\Indexer\Category\Product;

class Indexer extends \Magento\Framework\Indexer\Action
{
    public function executeFull(): void
    {
        $this->reindexAll();
    }

    private function reindexAll(): void
    {
        $connection = $this->getConnection();
        $connection->delete($this->getMainTable());

        // Insert all category-product relationships
        $select = $connection->select()->from(
            ['cc' => $this->getTable('catalog_category_entity')],
            ['category_id' => 'cc.entity_id', 'product_id' => 'ccp.product_id']
        )->join(
            ['ccp' => $this->getTable('catalog_category_product')],
            'cc.entity_id = ccp.category_id',
            []
        );

        $connection->query($connection->insertFromSelect(
            $select,
            $this->getMainTable(),
            ['category_id', 'product_id', 'position', 'is_parent'],
            \Magento\Framework\DB\Adapter\AdapterInterface::INSERT_ON_DUPLICATE_KEY_UPDATE
        ));
    }
}
```

## 5. Indexer CLI Commands

Magento ships with a comprehensive CLI for indexer management. All commands are in `bin/magento` or accessed via `php bin/magento`.

### indexer:reindex

Reindex one or all indexers:

```bash
# Reindex all indexers
php bin/magento indexer:reindex

# Reindex specific indexer
php bin/magento indexer:reindex catalog_product_price

# Reindex multiple indexers
php bin/magento indexer:reindex catalog_product_price catalogsearch_fulltext
```

Sample output:
```
Product PriceIndexer:     47%" 47/100 products 24.5MB 4.2s
Product PriceIndexer:     94%" 94/100 products 49.2MB 8.1s
Product PriceIndexer finished
Elapsed time: 8.3421 seconds
```

Behind the scenes, `IndexerCommand` invokes `executeFull()` on each indexer:

```php
<?php
declare(strict_types=1);

namespace Magento\Indexer\Console\Command;

class ReindexCommand extends \Symfony\Component\Console\Command\Command
{
    protected function execute(
        \Symfony\Component\Console\Input\InputInterface $input,
        \Symfony\Component\Console\Output\OutputInterface $output
    ): int {
        $indexerIds = $input->getArgument('indexer_ids');

        if (empty($indexerIds) || in_array('all', $indexerIds)) {
            $indexerIds = $this->indexerRegistry->getAllIndexerIds();
        }

        foreach ($indexerIds as $indexerId) {
            $indexer = $this->indexerRegistry->get($indexerId);
            $indexer->reindexAll();
        }

        return \Magento\Framework\Console\Cli::RETURN_SUCCESS;
    }
}
```

### indexer:reset

Mark indexer state as invalid (forces full reindex on next run):

```bash
# Reset specific indexer
php bin/magento indexer:reset catalog_product_price

# Reset all indexers
php bin/magento indexer:reset all
```

This updates `mview_state` to set `status = 'invalid'` and `version_id = 0`:

```sql
UPDATE mview_state SET status = 'invalid', version_id = 0 
WHERE view_id = 'catalog_product_price';
```

### indexer:set-mode

Change indexer execution mode:

```bash
# Set to schedule mode (async)
php bin/magento indexer:set-mode schedule catalog_product_price

# Set to realtime mode (sync)
php bin/magento indexer:set-mode realtime catalogsearch_fulltext

# Set all indexers to schedule mode
php bin/magento indexer:set-mode schedule all
```

### indexer:status

Display current status of all indexers:

```bash
php bin/magento indexer:status
```

Sample output:
```
+------------------+------------------+--------------------+---------------------+
| Indexer          | Mode             | Status             | Updated             |
+------------------+------------------+--------------------+---------------------+
| catalog_product_price | schedule         | ready              | 2025-10-15 09:30:00 |
| catalogsearch_fulltext| schedule         | working            | 2025-10-15 09:28:45 |
| catalog_category_flat | schedule         | invalid            | 2025-10-14 22:00:00 |
+------------------+------------------+--------------------+---------------------+
```

Status values:
- **valid**: Index is up-to-date
- **invalid**: Index data is stale, requires reindex
- **working**: Indexer is currently running
- **suspended**: Indexer is paused (for maintenance)

### indexer:info

Show detailed indexer information:

```bash
php bin/magento indexer:info
```

Output:
```
catalog_product_price     Product Price Indexer
catalogsearch_fulltext    Catalog Search Fulltext Indexer
catalog_category_flat    Category Flat Indexer
...
```

### Cron-Based Scheduled Indexing

For indexers in schedule mode, `MagentoIndexer` cron group runs `indexer:update`:

```xml
<!-- etc/crontab.xml -->
<group id="indexer">
    <job name="indexer_update" instance="Magento\Indexer\Cron\Update" method="execute">
        <schedule>*/5 * * * *</schedule>
    </job>
</group>
```

The `Update` cron handler:

```php
<?php
declare(strict_types=1);

namespace Magento\Indexer\Cron;

class Update
{
    public function execute(): void
    {
        $indexers = $this->indexerRegistry->getAllIndexers();
        
        foreach ($indexers as $indexer) {
            if ($indexer->isScheduled()) {
                $indexer->reindexUpdated();
            }
        }
    }
}
```

By default, cron runs every 5 minutes. For high-volume stores, reduce to 1 minute by overriding in `etc/crontab.xml`:

```xml
<group id="indexer">
    <job name="indexer_update" instance="Magento\Indexer\Cron\Update" method="execute">
        <schedule>* * * * *</schedule>
    </job>
</group>
```

## 6. Custom Indexer Development

### Overview

Creating a custom indexer involves:
1. `IndexerInterface` implementation
2. `etc/indexer.xml` declaration
3. `etc/mview.xml` changelog configuration
4. Database changelog table creation

### Step 1: Create the Indexer Class

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Indexer;

use Magento\Framework\Indexer\ActionInterface;
use Magento\Framework\Indexer\IndexerInterface;
use Magento\Framework\Indexer\IndexerRegistry;

class CustomProductAttributeIndexer implements ActionInterface
{
    private IndexerInterface $indexer;
    private IndexerRegistry $indexerRegistry;

    public function __construct(
        IndexerRegistry $indexerRegistry
    ) {
        $this->indexerRegistry = $indexerRegistry;
        $this->indexer = $indexerRegistry->get('vendor_custom_product_attr');
    }

    public function getId(): string
    {
        return 'vendor_custom_product_attr';
    }

    public function getName(): string
    {
        return 'Custom Product Attribute Indexer';
    }

    public function getDescription(): string
    {
        return 'Indexes custom product attribute for storefront performance';
    }

    public function executeFull(): void
    {
        $this->reindexAll();
    }

    public function executeList(array $ids): void
    {
        if (!empty($ids)) {
            $this->reindexProducts($ids);
        }
    }

    public function execute($id): void
    {
        $this->reindexProducts([(int)$id]);
    }

    private function reindexAll(): void
    {
        $productCollection = $this->getProductCollection();
        $productIds = $productCollection->getAllIds();
        
        $batchSize = $this->getBatchSize();
        foreach (array_chunk($productIds, $batchSize) as $batch) {
            $this->reindexProducts($batch);
        }
    }

    private function reindexProducts(array $productIds): void
    {
        foreach ($productIds as $productId) {
            $this->indexProduct($productId);
        }
    }

    private function indexProduct(int $productId): void
    {
        // Your indexing logic here
        // Write to index table
    }

    private function getBatchSize(): int
    {
        return (int) $this->scopeConfig->getValue(
            'indexer/catalog_batch_size',
            \Magento\Framework\App\Config\ScopeConfigInterface::SCOPE_TYPE_DEFAULT
        ) ?: 1000;
    }
}
```

### Step 2: Declare in indexer.xml

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Indexer/etc/indexer.xsd">
    <indexer id="vendor_custom_product_attr" 
             view_id="vendor_custom_product_attr" 
             class="Vendor\Module\Model\Indexer\CustomProductAttributeIndexer">
        <title translate="true">Custom Product Attribute Indexer</title>
        <description translate="true">Indexes custom product attribute</description>
    </indexer>
</config>
```

### Step 3: Configure mview.xml

```xml
<?xml version="1.0"?>
<mview xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:noNamespaceSchemaLocation="urn:magento:framework:Mview/etc/mview.xsd">
    <view id="vendor_custom_product_attr" 
          class="Vendor\Module\Model\Indexer\CustomProductAttributeIndexer" 
          group="vendor_custom">
        <subscriptions>
            <table name="catalog_product_entity" 
                   entity="product" 
                   subscription="catalog_product_entity_save" />
        </subscriptions>
    </view>
</mview>
```

### Step 4: Create Changelog Table

Create the changelog table via InstallSchema:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup;

use Magento\Framework\Setup\InstallSchemaInterface;
use Magento\Framework\Setup\ModuleContextInterface;
use Magento\Framework\Setup\SchemaSetupInterface;

class InstallSchema implements InstallSchemaInterface
{
    public function install(SchemaSetupInterface $setup, ModuleContextInterface $context): void
    {
        $setup->startSetup();

        $table = $setup->getConnection()->newTable(
            $setup->getTable('vendor_custom_product_attr_cl')
        )->addColumn(
            'version_id',
            \Magento\Framework\DB\Ddl\Table::TYPE_BIGINT,
            null,
            ['identity' => true, 'unsigned' => true, 'nullable' => false, 'primary' => true],
            'Version ID'
        )->addColumn(
            'entity_id',
            \Magento\Framework\DB\Ddl\Table::TYPE_INTEGER,
            null,
            ['nullable' => false],
            'Product ID'
        )->addColumn(
            'operation',
            \Magento\Framework\DB\Ddl\Table::TYPE_TEXT,
            10,
            ['nullable' => false],
            'Operation type'
        )->addColumn(
            'version_time',
            \Magento\Framework\DB\Ddl\Table::TYPE_TIMESTAMP,
            null,
            ['nullable' => false, 'default' => \Magento\Framework\DB\Ddl\Table::TIMESTAMP_INIT],
            'Version Timestamp'
        )->addIndex(
            'idx_entity',
            ['entity_id']
        )->addIndex(
            'idx_version',
            ['version_id']
        );

        $setup->getConnection()->createTable($table);

        $setup->endSetup();
    }
}
```

### Step 5: Create Indexer Table

The target index table where denormalized data is stored:

```php
// In InstallSchema or UpgradeSchema
$table = $setup->getConnection()->newTable(
    $setup->getTable('vendor_custom_product_attr_index')
)->addColumn(
    'product_id',
    \Magento\Framework\DB\Ddl\Table::TYPE_INTEGER,
    null,
    ['nullable' => false, 'primary' => true],
    'Product ID'
)->addColumn(
    'custom_value',
    \Magento\Framework\DB\Ddl\Table::TYPE_TEXT,
    255,
    ['nullable' => true],
    'Custom Computed Value'
)->addColumn(
    'updated_at',
    \Magento\Framework\DB\Ddl\Table::TYPE_TIMESTAMP,
    null,
    ['nullable' => false, 'default' => \Magento\Framework\DB\Ddl\Table::TIMESTAMP_INIT_UPDATE],
    'Update Time'
);

$setup->getConnection()->createTable($table);
```

### Complete Custom Indexer Example

Full working indexer with proper dependency injection:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Indexer;

use Magento\Framework\DataObject;
use Magento\Framework\Indexer\Action;
use Magento\Framework\App\ResourceConnection;
use Magento\Catalog\Model\ResourceModel\Product\CollectionFactory;

class CustomProductAttributeIndexer extends Action
{
    private ResourceConnection $resource;
    private CollectionFactory $productCollectionFactory;

    public function __construct(
        ResourceConnection $resource,
        CollectionFactory $productCollectionFactory,
        ?array $data = []
    ) {
        parent::__construct($data);
        $this->resource = $resource;
        $this->productCollectionFactory = $productCollectionFactory;
    }

    public function executeFull(): void
    {
        $this->reindexAll();
    }

    public function executeList(array $ids): void
    {
        if (!empty($ids)) {
            $this->reindexProducts($ids);
        }
    }

    public function executeRow(int $id): void
    {
        $this->reindexProducts([$id]);
    }

    private function reindexAll(): void
    {
        $collection = $this->productCollectionFactory->create();
        $collection->addFieldToFilter('custom_attribute', ['notnull' => true]);
        
        $productIds = $collection->getAllIds();
        $batchSize = 500;

        foreach (array_chunk($productIds, $batchSize) as $batch) {
            $this->reindexProducts($batch);
        }
    }

    private function reindexProducts(array $productIds): void
    {
        if (empty($productIds)) {
            return;
        }

        $connection = $this->resource->getConnection();
        $tableName = $this->resource->getTableName('vendor_custom_product_attr_index');

        // Clear existing entries
        $connection->delete($tableName, ['product_id IN (?)' => $productIds]);

        // Fetch product data
        $select = $connection->select()
            ->from(
                ['e' => $this->resource->getTableName('catalog_product_entity')],
                ['product_id' => 'entity_id']
            )
            ->join(
                ['ca' => $this->resource->getTableName('catalog_product_entity_int')],
                'e.entity_id = ca.entity_id AND ca.attribute_id = ' . $this->getCustomAttributeId(),
                ['custom_value' => 'ca.value']
            )
            ->where('e.entity_id IN (?)', $productIds);

        $insertData = $connection->fetchAll($select);

        if (!empty($insertData)) {
            foreach ($insertData as &$row) {
                $row['updated_at'] = time();
            }
            $connection->insertMultiple($tableName, $insertData);
        }
    }

    private function getCustomAttributeId(): int
    {
        return 123; // Replace with actual attribute ID lookup
    }
}
```

## 7. Indexer Performance

### Batch Size Configuration

The primary performance knob for indexers is batch size—the number of entities processed per iteration. Configure via `env.php`:

```php
// app/etc/env.php
return [
    // ...
    'indexer' => [
        'catalog_batch_size' => 5000,
        'catalogsearch_batch_size' => 100,
        'ecatalog_batch_size' => 1000,
    ],
    // ...
];
```

Or via `Magento\Framework\App\Config\ScopeConfigInterface`:

```xml
<!-- etc/env.php or admin configuration -->
<config>
    <indexer>
        <catalog_batch_size>5000</catalog_batch_size>
    </indexer>
</config>
```

### Batch Processing Implementation

```php
<?php
declare(strict_types=1);

trait BatchProcessingTrait
{
    private function processInBatches(array $ids, int $batchSize, callable $processor): void
    {
        $batches = array_chunk($ids, $batchSize);
        
        foreach ($batches as $batch) {
            $processor($batch);
            
            // Allow garbage collection between batches
            gc_collect_cycles();
        }
    }

    private function getBatchSize(string $indexerId): int
    {
        $batchSizes = [
            'catalog_product_price' => 5000,
            'catalogsearch_fulltext' => 100,
            'catalog_product_flat' => 10000,
            'inventory' => 5000,
        ];

        return $batchSizes[$indexerId] ?? 1000;
    }
}
```

### catalog_batch_size Environment Variable

For the catalog indexers, use the environment variable:

```bash
# In .env or system environment
export CATALOG_BATCH_SIZE=5000
```

Or set programmatically:

```php
<?php
declare(strict_types=1);

namespace Magento\Catalog\Model\Indexer\Product;

class Price extends Action
{
    private function getBatchSize(): int
    {
        $envBatchSize = getenv('CATALOG_BATCH_SIZE');
        if ($envBatchSize !== false) {
            return (int) $envBatchSize;
        }

        return $this->scopeConfig->getValue(
            'indexer/catalog_batch_size',
            \Magento\Framework\App\Config\ScopeConfigInterface::SCOPE_TYPE_DEFAULT
        ) ?: 1000;
    }
}
```

### Parallel Indexing

For multi-core systems, Magento supports running independent indexers in parallel via cron configuration. Edit `etc/crontab.xml`:

```xml
<group id="indexer">
    <!-- Run inventory indexer with higher frequency -->
    <job name="indexer_update_inventory" 
         instance="Magento\InventoryIndexer\Cron\Update" 
         method="execute">
        <schedule>*/1 * * * *</schedule>
    </job>
    
    <!-- Price indexer every 5 minutes -->
    <job name="indexer_update_price" 
         instance="Magento\Indexer\Cron\Update" 
         method="execute">
        <schedule>*/5 * * * *</schedule>
        <arguments>
            <indexers>catalog_product_price</indexers>
        </arguments>
    </job>
</group>
```

### Indexer Invalidation Events

Indexers are invalidated when their source data changes. Key invalidation triggers:

```php
<?php
declare(strict_types=1);

namespace Magento\Catalog\Model\Indexer;

class ProductProcessor
{
    /**
     * Mark price indexer invalid on product save
     */
    public function markPriceIndexerInvalid(): void
    {
        $this->indexerRegistry->get('catalog_product_price')->invalidate();
    }

    /**
     * Mark flat indexer invalid on product save
     */
    public function markFlatIndexerInvalid(): void
    {
        $this->indexerRegistry->get('catalog_product_flat')->invalidate();
    }
}
```

Common invalidation patterns:

| Event | Indexer Invalidated |
|-------|---------------------|
| `catalog_product_save_commit_after` | `catalog_product_price`, `catalog_product_flat` |
| `catalog_category_save_commit_after` | `catalog_category_flat`, `catalog_category_product` |
| `catalogsearch_searchable_attribute_save_after` | `catalogsearch_fulltext` |
| `inventory_stock_item_save_commit_after` | `inventory` |
| `customer_save_commit_after` | `customer_grid` |

### Manual Invalidation via CLI

```bash
# Force invalidate specific indexer
php bin/magento indexer:reset catalog_product_price

# Check invalid indexers
php bin/magento indexer:status | grep invalid
```

### Memory Limits for Large Indexers

For catalogs with 100K+ products, adjust PHP memory limits in cron:

```xml
<!-- etc/crontab.xml -->
<group id="indexer">
    <job name="indexer_reindex_all" 
         instance="Magento\Indexer\Console\Command\ReindexCommand" 
         method="execute">
        <schedule>0 2 * * *</schedule>  <!-- 2 AM daily -->
        <config>
            <php_memory_limit>4G</php_memory_limit>
        </config>
    </job>
</group>
```

### Indexer Optimization Checklist

1. **Match mode to catalog size**:
   - Small catalog (< 10K products): Realtime mode acceptable
   - Large catalog (100K+): Schedule mode required

2. **Tune batch sizes**:
   - Product count < 50K: batch size 1000
   - Product count 50K-200K: batch size 5000
   - Product count > 200K: batch size 10000

3. **Configure cron frequency**:
   - Standard: `*/5 * * * *`
   - High-volume: `*/1 * * * *`

4. **Use appropriate storage engine**:
   - Index tables: InnoDB with proper PRIMARY KEY
   - Changelog tables: InnoDB with row-based replication

5. **Monitor reindex times**:
   ```bash
   time php bin/magento indexer:reindex catalog_product_price
   ```

6. **Pre-warm price index after catalog update**:
   ```bash
   php bin/magento indexer:reindex catalog_product_price
   ```

---

## Summary

Magento 2.4.8's indexer subsystem transforms normalized database designs into denormalized representations optimized for storefront query performance. The MVIEW mechanism provides change tracking through database triggers and changelog tables, enabling both immediate (realtime) and deferred (schedule) index updates.

Key takeaways:
- **Indexers are Materialized Views**: They pre-compute expensive JOIN operations into single-table lookups
- **MVIEW tracks changes via triggers**: Every entity modification inserts a row into the changelog
- **Schedule mode is production standard**: For catalogs > 10K products, always use schedule mode
- **Batch processing is essential**: Tune `catalog_batch_size` to match your hardware
- **Custom indexers follow the same pattern**: Implement `IndexerInterface`, declare in `indexer.xml`, configure in `mview.xml`

Understanding indexers is fundamental to troubleshooting performance issues and building custom functionality that integrates seamlessly with Magento's data architecture.