---
title: "04 - Database Optimization"
description: "Magento 2.4.8 database optimization: slow query analysis via var/debug/db.log, EXPLAIN analysis, EAV index pitfalls, indexer optimization, MySQL configuration tuning, N+1 query patterns, repository pattern overhead, connection pooling, read replicas, and production mysqldump strategies."
tags: [magento2, database, mysql, slow-query, explain, indexing, eav, n+1, connection-pooling, mysqldump, read-replica]
rank: 4
pathways: [magento2-deep-dive]
see_also:
  - "Indexer Management: 05-indexer-management.md"
  - "Performance Profiling: 07-performance-profiling.md"
  - "Declarative Schema: _supplemental/04-declarative-schema.md"
---

# 04 — Database Optimization

Database performance is the single most common source of Magento 2 request latency. A poorly indexed query against `catalog_product_entity` can block a frontend request for 3-8 seconds. A single N+1 pattern — `$product->getCategoryIds()` inside a product loop — can generate 500 individual SELECT statements where one would suffice. This chapter gives you the analytical tools and tactical knowledge to identify, diagnose, and fix database bottlenecks in Magento 2.4.8.

---

## 1. SQL Query Logging: `var/debug/db.log`

### Enabling Query Logging

Magento 2.4.8 logs every SQL query to `var/debug/db.log` when query logging is enabled. This is controlled by the `dev.db query_logging` configuration flag.

**Enable via CLI:**

```bash
bin/magento config:set dev/db/query_logging 1
```

**Disable when not needed** (logging has measurable performance impact):

```bash
bin/magento config:set dev/db/query_logging 0
```

**Direct database write (if locked out of admin):**

```php
// In a temporary PHP script or setup:di:compile scenario
use Magento\Framework\App\DeploymentConfig\Writer;
use Magento\Framework\Component\ComponentRegistrar;
use Magento\Framework\Filesystem\Driver\File;

$configWriter = new Writer(new File());
$configWriter->saveConfig(['dev' => ['db' => ['query_logging' => 1]]]);
```

### What Gets Logged

When enabled, every PDO SQL statement is written to `var/debug/db.log`. Each entry includes:

- **Query text** — the bound SQL with `?` placeholders replaced by actual values
- **Bind parameters** — the values passed for each placeholder
- **Execution time** — in milliseconds
- **Connection name** — `read` or `write` (see Section 7 on read replicas)

```
-- [2026-04-24 10:23:45] db-log: SELECT `e`.* FROM `catalog_product_entity` AS `e`
-- Bind: [1 => 1]
-- Time: 0.234 ms
-- Connection: write

-- [2026-04-24 10:23:45] db-log: SELECT `cat`.`category_id` FROM `catalog_category_product` AS `cat`
   WHERE (product_id = '123')
-- Time: 0.089 ms
-- Connection: read
```

### Parsing the Log

For a production site with moderate traffic, `var/debug/db.log` can grow to hundreds of megabytes per hour. Use targeted grep patterns to extract signal:

**Find slow queries (>100ms):**

```bash
grep -E "Time: [0-9]{3}\.[0-9]{3} ms" var/debug/db.log | head -20
```

**Count queries per table (identify over-queried tables):**

```bash
grep -oE "FROM \`[a-z_]+\`" var/debug/db.log \
  | sort | uniq -c | sort -rn | head -20
```

**Find queries without indexes (type=ALL in EXPLAIN terms):**

This requires correlating `db.log` queries with `EXPLAIN` output — see Section 2.

### Finding N+1 Queries in the Log

N+1 queries are the most common database performance anti-pattern in Magento. They appear in `db.log` as a repeating pattern of identical queries with different parameters:

```
-- Query 1
SELECT `cat`.`category_id` FROM `catalog_category_product` WHERE product_id = '100'
-- Query 2
SELECT `cat`.`category_id` FROM `catalog_category_product` WHERE product_id = '101'
-- Query 3
SELECT `cat`.`category_id` FROM `catalog_category_product` WHERE product_id = '102'
... (重复 300+ 次)
```

**Pattern-matching approach:**

```bash
# Extract queries grouped by query shape (stripping numeric literals)
grep "SELECT" var/debug/db.log \
  | sed "s/[0-9]\{1,\}/N/g" \
  | sort | uniq -c | sort -rn | head -20
```

If the same `SELECT ... WHERE product_id = N` pattern appears 300 times in one request, you have a confirmed N+1. See Section 8 for the full diagnosis and fix.

**Real example from `db.log` with a configurable product collection:**

```sql
-- Request: /catalog/product/view/id/456
-- Time: 0.089 ms
SELECT `child`.`entity_id` FROM `catalog_product_relation` AS `child`
WHERE (parent_id = '456')

-- Time: 0.112 ms
SELECT `e`.* FROM `catalog_product_entity` AS `e`
WHERE (entity_id = '789')

-- Time: 0.098 ms
SELECT `e`.* FROM `catalog_product_entity` AS `e`
WHERE (entity_id = '790')

-- Time: 0.095 ms
SELECT `e`.* FROM `catalog_product_entity` AS `e`
WHERE (entity_id = '791')
```

The `getChildrenIds()` call on a configurable product is triggering individual `catalog_product_entity` lookups for each child SKU — a classic N+1.

---

## 2. EXPLAIN Analysis

### Running EXPLAIN

The `EXPLAIN` statement is your primary tool for understanding how MySQL executes a query. It shows the query execution plan — which indexes MySQL considers, in what order tables are joined, and how many rows are examined.

```sql
EXPLAIN SELECT e.*, cp.category_id
FROM catalog_product_entity e
LEFT JOIN catalog_category_product cp ON e.entity_id = cp.product_id
WHERE e.attribute_set_id = 4 AND cp.category_id = 12;
```

For MySQL 8.0+, `EXPLAIN ANALYZE` runs the query and returns actual runtime statistics:

```sql
EXPLAIN ANALYZE
SELECT e.*, cp.category_id
FROM catalog_product_entity e
LEFT JOIN catalog_category_product cp ON e.entity_id = cp.product_id
WHERE e.attribute_set_id = 4 AND cp.category_id = 12;
```

### Interpreting Key Columns

#### `type` — Join Type

The `type` column is the most important indicator of query efficiency:

| Type | Meaning | Acceptable? |
|------|---------|-------------|
| `system` | Table has only one row (e.g., system constants table) | ✅ Optimal |
| `const` | Table has at most one matching row (indexed PK/UK lookup) | ✅ Optimal |
| `eq_ref` | For each row in the outer table, one row is read via unique index | ✅ Good |
| `ref` | For each row in the outer table, all rows with matching index value are read | ⚠️ Acceptable for low-cardinality |
| `range` | Index range scan | ⚠️ Normal for filtered queries |
| `index` | Full index scan (not table) | ⚠️ May be acceptable |
| `ALL` | **Full table scan** | ❌ Problem — no index used |

A query with `type: ALL` on `catalog_product_entity` (which can have 100,000+ rows) is a serious performance problem. This is what you find when EAV joins bypass indexes.

#### `key` — Index Used

The `key` column shows which index MySQL actually chose. If `key` is `NULL`, MySQL is doing a table scan despite an available index. Common causes:

- The `WHERE` clause doesn't include the index leading column
- MySQL's cost estimator prefers a table scan for small tables
- The index is too selective (returns too many rows to be useful)

```sql
EXPLAIN SELECT * FROM sales_order WHERE status = 'pending';
-- key: NULL  ← no index used, doing full table scan
```

Adding a partial index fixes this:

```sql
CREATE INDEX idx_sales_order_status ON sales_order(status);
```

#### `rows` — Rows Examined

The `rows` column estimates how many rows MySQL will examine to execute the query. Compare this to the total row count of the table:

```sql
EXPLAIN SELECT * FROM catalog_product_entity WHERE attribute_set_id = 4;
-- rows: 45231  ← examining 45K rows to find ~500 matches
-- If catalog_product_entity has 50K rows, this is nearly a table scan
```

Lower is better. If `rows` approaches the table's total count, you need a better index.

#### `Extra` — Additional Information

The `Extra` column provides critical context:

| Value | Interpretation |
|-------|----------------|
| `Using filesort` | MySQL must sort results post-query — expensive on large sets |
| `Using temporary` | MySQL creates a temporary table for the query |
| `Using index condition` | Index condition pushdown is used (good) |
| `Using index` | Covering index — no table access needed (excellent) |
| `Range checked for each record` | No suitable index, re-evaluating for each row (bad) |
| `Using where` | Storage engine filters rows after retrieval (normal) |

**`Using filesort` is a red flag for product listing pages:**

```sql
EXPLAIN SELECT * FROM sales_order ORDER BY created_at DESC LIMIT 20;
-- Extra: Using filesort
-- rows: 150000  ← scanning all orders to sort by date
```

Fix with a compound index on `(created_at)` or `(status, created_at)` if filtering by status first.

### When to Add Indexes vs Rewrite Queries

**Add an index when:**
- The `type` column shows `ALL` (table scan) on a large table
- The `key` column is `NULL` despite a `WHERE` clause on an indexed column
- `rows` is high relative to actual matching rows
- The query is on a hot path (category pages, checkout, search)

**Rewrite the query when:**
- A cartesian join is being performed (query joins tables in wrong order)
- Subqueries can be replaced with JOINs
- The EAV structure requires too many joins (consider denormalization or flat table)
- A partial index can replace a full table scan

```sql
-- BEFORE (table scan): 12,000 rows examined
SELECT e.* FROM catalog_product_entity e
WHERE e.created_at > '2026-01-01';

-- AFTER (index scan): 45 rows examined
SELECT e.* FROM catalog_product_entity e
USE INDEX (idx_catalog_product_entity_created_at)
WHERE e.created_at > '2026-01-01';
```

---

## 3. Key Magento Indexes

### Index Types in MySQL

MySQL supports four index types relevant to Magento:

| Index Type | Keyword | Use Case | Can Have Duplicates? |
|------------|---------|----------|----------------------|
| Primary Key | `PRIMARY KEY` | Row identity — one per table | No |
| Unique | `UNIQUE` | Business uniqueness constraints | No |
| Regular | `INDEX` / `KEY` | Query optimization | Yes |
| Fulltext | `FULLTEXT` | Text search on VARCHAR/TEXT | Yes |

### What Core Magento Sets Up

Magento's declarative schema (`db_schema.xml`) defines the indexes for all core tables. Key examples from a default 2.4.8 install:

**`sales_order`** — Critical indexes:

```sql
-- Primary key (implicit on entity_id)
PRIMARY KEY (`entity_id`),

-- Unique index for increment_id (order number uniqueness)
UNIQUE KEY `SALES_ORDER_INCREMENT_ID` (`increment_id`),

-- Index for customer lookup
KEY `SALES_ORDER_CUSTOMER_ID` (`customer_id`),

-- Index for status filtering (used in admin grids)
KEY `SALES_ORDER_STATUS_CREATED_AT` (`status`, `created_at`),

-- Index for search
KEY `SALES_ORDER_GRID_EMAIL` (`customer_email`)
```

**`catalog_product_entity`** — Core indexes:

```sql
PRIMARY KEY (`entity_id`),
UNIQUE KEY `URL_KEY` (`url_key`),
KEY `CATALOG_PRODUCT_ENTITY_ATTRIBUTE_SET_ID` (`attribute_set_id`),
KEY `CATALOG_PRODUCT_ENTITY_CREATED_AT` (`created_at`),
KEY `CATALOG_PRODUCT_ENTITY_UPDATED_AT` (`updated_at`)
```

**`catalog_category_product`** — The product-category link table:

```sql
PRIMARY KEY (`product_id`, `category_id`),
KEY `CATALOG_CATEGORY_PRODUCT_CATEGORY_ID` (`category_id`)
```

### Indexes Missing in a Fresh Install

Despite Magento's comprehensive schema, common performance gaps exist:

**Missing composite index for admin order grid:**

```sql
-- Admin orders grid filters by (status, created_at, store_id)
-- but many installations lack a composite index
CREATE INDEX idx_sales_order_grid_status_store
ON sales_order_grid (status, store_id, created_at);
```

**Missing index on `eav_attribute.entity_id` for join performance:**

```sql
-- Used when filtering EAV entities by attribute
CREATE INDEX idx_eav_attribute_entity_type_id
ON eav_attribute (entity_type_id, attribute_code);
```

**Missing fulltext index on `catalog_product_entity_varchar` for search:**

```sql
-- If using LIKE-based search on product names
ALTER TABLE catalog_product_entity_varchar
ADD FULLTEXT INDEX ft_product_name (value);
```

### `entity_id` vs `increment_id`

Magento uses two distinct identity systems:

**`entity_id`** — The internal auto-increment primary key. Used for all JOINs and foreign key relationships. Fast, indexed, integer-based. Never exposed to customers.

```php
// Internal use — fast, indexed
$order->getId();                    // returns entity_id (int)
$order->load($orderId);            // uses entity_id
```

**`increment_id`** — The business-facing order/product identifier (e.g., `000000001`). Exposed to customers, appears in emails, admin URLs. Unique-indexed but not the primary key.

```php
// External use — slower for joins due to string comparison
$order->getIncrementId();          // returns string like '000000001'
$order->loadByIncrementId($incrementId);

// In URLs
<a href="/sales/order/view/order_id/{$order->getId()}">
```

For performance-critical internal operations, always use `entity_id`. Use `increment_id` only when interfacing with external systems or displaying to users.

---

## 4. EAV Index Issues

### Why EAV Queries Are Inherently Slow

EAV (Entity-Attribute-Value) is Magento's schema architecture that allows dynamic attributes without schema changes. However, querying EAV entities for filtering and display requires joining multiple tables — a fundamental architectural tradeoff.

**The EAV join problem in practice:**

```sql
-- To filter products by color='red' AND size='M':
SELECT e.entity_id FROM catalog_product_entity e
INNER JOIN catalog_product_entity_int cpei_color
    ON e.entity_id = cpei_color.entity_id
    AND cpei_color.attribute_id = (
        SELECT attribute_id FROM eav_attribute
        WHERE attribute_code = 'color' AND entity_type_id = 4
    )
    AND cpei_color.value = 93  -- option_id for 'red'
INNER JOIN catalog_product_entity_int cpei_size
    ON e.entity_id = cpei_size.entity_id
    AND cpei_size.attribute_id = (
        SELECT attribute_id FROM eav_attribute
        WHERE attribute_code = 'size' AND entity_type_id = 4
    )
    AND cpei_size.value = 168  -- option_id for 'M'
WHERE e.attribute_set_id = 4;
```

This is a 4-table query with 2 correlated subqueries. Even with indexes on `(entity_id, attribute_id, value)`, MySQL must evaluate the correlated subqueries for every candidate row.

### `catalog_product_entity_*` Joins

Each EAV attribute type maps to a separate table:

| Attribute Type | Table | Example |
|----------------|-------|---------|
| `varchar` | `catalog_product_entity_varchar` | Product name, URL key |
| `int` | `catalog_product_entity_int` | Price, status flags |
| `text` | `catalog_product_entity_text` | Description, short description |
| `decimal` | `catalog_product_entity_decimal` | Price, special price |
| `datetime` | `catalog_product_entity_datetime` | news_from_date, custom date attrs |
| `float` | `catalog_product_entity_float` | Weight, rating |

Loading a single product with all its attributes requires up to 6 joins:

```php
// Magento\Catalog\Model\Product::getAttributes()
// This triggers joins to all 6 EAV tables
$product->getData('name');        // varchar
$product->getData('price');       // decimal
$product->getData('description'); // text
$product->getData('status');      // int
```

### When to Denormalize vs Live with EAV

**Stay with EAV when:**
- Attribute sets change frequently (flexibility is more valuable than query speed)
- The number of attributes per product is moderate (<50)
- Filtering is done through Magento's indexers (price, stock, category membership)

**Denormalize (use flat tables or custom tables) when:**
- You have a fixed set of 10-20 attributes queried on every product page
- You need sub-100ms queries on denormalized columns
- You're building a custom product comparison or highly filtered search

**Magento's flat table solution:**

```xml
<!-- etc/indexer.xml -->
<urn:magento:indexer:index:catalog_product_flat>
    <title>Product Flat Index</title>
    <description>Denormalized product attributes into flat table</description>
    <model>Magento\Catalog\Model\Indexer\Product\Flat</model>
</urn:magento:indexer:index:catalog_product_flat>
```

```php
// In your module's collection
$collection = $this->_productCollectionFactory->create();
$collection->addAttributeToSelect(['name', 'price', 'status', 'image']);
// Magento automatically uses flat table when available
```

---

## 5. Indexer Optimization

### `indexer` Table vs Live EAV

Magento's indexers pre-compute the results of expensive EAV joins into dedicated flat tables. The main index tables:

| Indexer | Index Table | Purpose |
|---------|-------------|---------|
| `catalog_product_price` | `catalog_product_index_price` | Pre-computed price per customer group |
| `catalogsearch_fulltext` | `catalogsearch_fulltext` | Full-text search data |
| `inventory` | `inventory_stock_*` | Stock availability per source |
| `catalog_category_product` | `catalog_category_product_index_store*` | Category-product associations |
| `sales_order_grid` | `sales_order_grid` | Denormalized order data for admin |

### `catalogsearch_fulltext` Indexer

The fulltext search indexer populates `catalogsearch_fulltext_table` and `catalogsearch_fulltext_store_*` tables. Running it:

```bash
# Full reindex
bin/magento indexer:reindex catalogsearch_fulltext

# Check status
bin/magento indexer:status

# Individual indexer info
bin/magento indexer:info
```

**Common issue: stale search index after bulk product updates**

```php
// Manually trigger fulltext reindex after product import
use Magento\CatalogSearch\Model\Indexer\Fulltext as FulltextIndexer;

$this->indexerRegistry->get(FulltextIndexer::INDEXER_ID)->reindexAll();
```

### `catalog_product_price` Indexer

Price calculation involves multiple factors:
- Base price from `catalog_product_entity_decimal`
- Tier pricing from `catalog_product_entity_tier_price`
- Special pricing from `catalog_product_entity_decimal`
- Customer group pricing from `price_index` tables
- Catalog price rules

The indexer materializes all these combinations into `catalog_product_index_price`:

```sql
SELECT * FROM catalog_product_index_price WHERE entity_id = 123 AND customer_group_id = 1;
-- Returns: website_id, customer_group_id, price, final_price, min_price, max_price, tier_price
```

**`setup:di:compile` vs `setup:upgrade`:**

- `bin/magento setup:upgrade` — Runs data and schema patches, generates the `var/generation` factory classes, enables the module in `app/etc/config.php`
- `bin/magento setup:di:compile` — Generates all compiled dependency injection classes (factories, proxies, interceptors) into `var/generation`. Must be run after any constructor signature change or after adding a new preference

For indexer changes specifically, you typically need:
```bash
bin/magento setup:upgrade
bin/magento setup:di:compile
bin/magento indexer:reindex
```

---

## 6. MySQL Configuration for Magento

### Essential `my.cnf` Parameters

**`innodb_buffer_pool_size`** — The most important setting for read-heavy workloads:

```ini
# Set to 50-70% of available RAM on a dedicated MySQL host
# 8GB RAM server → innodb_buffer_pool_size = 4G
innodb_buffer_pool_size = 4G
innodb_buffer_pool_instances = 4  # One per GB of buffer pool, up to 8
```

This caches table data and indexes in RAM. With 100GB of catalog data but only 4GB buffer pool, MySQL constantly evicts cached pages.

**`max_connections`**:

```ini
# Magento's default connection pooling allows ~150 concurrent connections per FPM worker
# Set high enough to handle peak + buffer
max_connections = 500
```

**`slow_query_log`** — Enable the slow query log for production diagnostics:

```ini
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1  # Log queries taking > 1 second
log_queries_not_using_indexes = 1
```

**`query_cache_type` was removed in MySQL 8.0.** If you're migrating from MySQL 5.7, remove any `query_cache_*` settings — they will cause startup errors in MySQL 8.

**Recommended MySQL 8 settings for Magento 2.4.8:**

```ini
[mysqld]
# Buffer and cache
innodb_buffer_pool_size = 4G
innodb_log_file_size = 1G
innodb_flush_log_at_trx_commit = 2  # 2 = flush once per second (faster, slight durability trade)
innodb_flush_method = O_DIRECT

# Connection handling
max_connections = 500
wait_timeout = 600
interactive_timeout = 600

# Logging
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = 0  # Enable temporarily for analysis, then disable

# Character set
character-set-server = utf8mb4
collation-server = utf8mb4_general_ci

# Binary logging (for replication or point-in-time recovery)
log_bin = /var/log/mysql/mysql-bin.log
expire_logs_days = 7
```

---

## 7. Connection Pooling

### `max_connections` vs Actual Usage

MySQL's `max_connections` is a hard limit on concurrent client connections. Each PHP-FPM worker can hold multiple DB connections (one per nested `Magento\Framework\DB\Adapter\Pdo\Mysql` scope).

**What actually happens in a Magento 2.4.8 request:**

```php
// Request starts
$connection = $this->_resource->getConnection();  // Connection #1

// In a plugin on \Magento\Catalog\Model\Product::getPrice
$priceConnection = $this->_resource->getConnection(); // Connection #2

// After layout rendering
$blockConnection = $this->_resource->getConnection(); // Connection #3
```

If you have 50 PHP-FPM workers, each handling a request with 3 pooled connections, you need `50 × 3 = 150` connections minimum.

**Monitoring actual connection usage:**

```sql
SHOW STATUS LIKE 'Threads_connected';  -- Current
SHOW STATUS LIKE 'Max_used_connections'; -- Peak used
SHOW STATUS LIKE 'Aborted_connects';     -- Failed connections
```

### Magento's `AdapterInterface` Implementation

Magento's database adapter is `Magento\Framework\DB\Adapter\Pdo\Mysql`, which wraps Zend PDO:

```php
// Magento\Framework\DB\Adapter\Pdo\Mysql
class Mysql implements \Magento\Framework\DB\Adapter\AdapterInterface
{
    // Key methods:
    public function query($sql, array $bind = []);
    public function insert($table, array $data);
    public function update($table, array $data, $where);
    public function delete($table, $where);
    public function fetchAll($sql, array $bind = []);
    public function fetchOne($sql, array $bind = []);
    public function select();
    public function beginTransaction();
    public function commit();
    public function rollBack();
}
```

**Connection acquisition in Magento:**

```php
// Magento\Framework\App\ResourceConnection
public function getConnection($connectionName = null)
{
    if ($connectionName === null) {
        $connectionName = $this->_connectionConfigMap['default'];
    }

    if (!isset($this->_connections[$connectionName])) {
        $this->_connections[$connectionName] = $this->_factory->create(
            $this->_config,
            $connectionName
        );
    }

    return $this->_connections[$connectionName];
}
```

**Connection retry logic in `Magento\Framework\DB\Adapter\Pdo\Mysql`:**

```php
protected function _retryConnection($sql, $bind = [])
{
    $retry = 0;
    $maxRetries = 3;

    while ($retry < $maxRetries) {
        try {
            $this->_connection->query($sql);
            return;
        } catch (\PDOException $e) {
            if ($e->getCode() == 2006 || $e->getCode() == 2013) { // MySQL gone away / lost connection
                $this->_connection = null;
                $this->connect();
                $retry++;
            } else {
                throw $e;
            }
        }
    }
}
```

---

## 8. N+1 Query Patterns

### The Classic: `$product->getCategoryIds()`

This is the most notorious N+1 pattern in Magento. When called on a product in a loop:

```php
// In a template or block rendering product list
foreach ($productCollection as $product) {
    $categoryIds = $product->getCategoryIds();
    // ...
}
```

What happens under the hood:

```php
// Magento\Catalog\Model\Product::getCategoryIds()
public function getCategoryIds()
{
    if ($this->hasData('category_ids')) {
        return $this->getData('category_ids');
    }

    // Lazy load — triggers one query per product
    $ids = $this->getCategoryIdsStorage()->getCategoryIds($this->getId());
    $this->setData('category_ids', $ids);
    return $ids;
}

// Magento\Catalog\Model\ResourceModel\Product\Link\ProductCategory
// Triggers: SELECT category_id FROM catalog_category_product WHERE product_id = ?
```

For a collection of 300 products, this generates 300 `SELECT ... WHERE product_id = N` queries.

### `getChildrenIds()` in Configurable Products

Configurable products store child SKUs in `catalog_product_relation`:

```php
// Loading child IDs for configurable product
$childrenIds = $configurableProduct->getTypeInstance()->getChildrenIds($configurableProduct->getId());

// Each getChildrenIds() triggers:
SELECT child_id FROM catalog_product_relation WHERE parent_id = ?
```

### Fixing N+1 with `joinTable`

**Fix 1: Eager-load category IDs in the collection query:**

```php
// In a custom ProductRepository or CollectionFactory
$collection = $this->productCollectionFactory->create();
$collection->addAttributeToSelect(['name', 'price']);

// Eager-load category_ids to avoid N+1
$collection->joinAttribute('category_ids', 'catalog_product/category_ids', 'entity_id', null, 'left');
```

**Fix 2: Use `getLoadedProductCollection()` with `setFlag`:**

```php
// In \Magento\Catalog\Block\Product\ListProduct::_getProductCollection
$collection = $this->getLoadedProductCollection();

// Before rendering, force load of category_ids
if (!$collection->hasFlag('category_ids_loaded')) {
    $collection->setFlag('category_ids_loaded', true);
    // The following doesn't work in all versions; use Fix 3 instead
}
```

**Fix 3: Direct SQL join in collection:**

```php
// Custom product collection with category join
$collection = $this->productCollectionFactory->create();
$collection->getSelect()->joinLeft(
    ['ccp' => 'catalog_category_product'],
    'e.entity_id = ccp.product_id',
    ['category_ids' => new \Zend_Db_Expr('GROUP_CONCAT(ccp.category_id)')]
)->group('e.entity_id');
```

### `getLoadedProductCollection` vs Direct Collection

```php
// Magento\Catalog\Block\Product\ListProduct
protected function _getProductCollection()
{
    if ($this->_productCollection === null) {
        // Uses \Magento\Catalog\Model\ResourceModel\Product\Collection
        // which applies all registered layers (category filter, price filter, stock)
        $this->_productCollection = $this->initializeProductCollection();
    }
    return $this->_productCollection;
}
```

The `getLoadedProductCollection()` returns a pre-initialized collection with all applicable layers. Using it directly (rather than instantiating a new collection) avoids duplicate layer application and double queries.

---

## 9. Repository Pattern Overhead

### `SearchCriteria` + Repository vs Direct Collection

Magento 2's repository pattern (`Magento\Framework\Api\SearchCriteriaInterface`) wraps collections with service contract semantics:

```php
// Repository approach (more overhead for bulk)
use Magento\Catalog\Api\ProductRepositoryInterface;

$searchCriteria = $this->searchCriteriaBuilder
    ->addFilter('attribute_set_id', 4)
    ->addFilter('status', 1)
    ->setPageSize(20)
    ->setCurrentPage(1)
    ->create();

$products = $this->productRepository->getList($searchCriteria);
// Internally: creates collection → applies filters → returns SearchResults
```

**Why repositories add overhead:**

1. `SearchCriteriaBuilder` → `SearchCriteria` object creation
2. Repository `getList()` calls `addFiltersFromSearchCriteria()` on the collection
3. `SearchResults` data object is constructed with metadata (total count, pagination)
4. For bulk operations, this metadata is unnecessary

**Direct collection approach (lower overhead for bulk):**

```php
// Direct collection (less overhead)
$collection = $this->productCollectionFactory->create();
$collection->addAttributeToFilter('attribute_set_id', 4);
$collection->addAttributeToFilter('status', 1);
$collection->setPage(1, 20);

foreach ($collection as $product) {
    // Process each product
}
```

### When to Bypass Repositories

**Use repositories when:**
- Building API endpoints (service contract compliance)
- You need `SearchResults` with total count for pagination UI
- Third-party code or GraphQL resolvers that expect interface compliance

**Bypass repositories when:**
- Processing 10,000+ products in a batch script
- Building a one-off data export
- Performance is critical and you control the query entirely

```php
// High-performance bulk processing — bypass repository
/** @var \Magento\Catalog\Model\ResourceModel\Product\Collection $collection */
$collection = $this->productCollectionFactory->create();
$collection->addAttributeToSelect(['entity_id', 'sku', 'name']);
$collection->addFieldToFilter('entity_id', ['in' => $productIds]);

$batchSize = 500;
$productIds = array_chunk($productIds, $batchSize);

foreach ($productIds as $batch) {
    $collection->clear()->addFieldToFilter('entity_id', ['in' => $batch]);
    foreach ($collection as $product) {
        // Process without repository overhead
    }
}
```

---

## 10. The `sales_order_grid` Flat Table

### Why `sales_order_grid` Exists

The `sales_order_grid` table is a denormalized copy of `sales_order` optimized for the admin order grid. Loading orders in the admin panel would otherwise require joining `sales_order`, `sales_order_address`, `sales_order_payment`, and `sales_order_item` — 4 tables per row in the grid.

```sql
-- sales_order_grid is flat — single table query
SELECT * FROM sales_order_grid
WHERE status = 'pending'
ORDER BY created_at DESC
LIMIT 20;

-- vs the normalized alternative:
SELECT so.*, soa.*, sop.*
FROM sales_order so
LEFT JOIN sales_order_address soa ON so.entity_id = soa.parent_id
LEFT JOIN sales_order_payment sop ON so.entity_id = sop.parent_id
WHERE so.status = 'pending'
LIMIT 20;  -- Still not including items!
```

### Schema Difference

`sales_order_grid` contains a subset of `sales_order` columns plus a few flattened fields:

| Column | Source | Notes |
|--------|--------|-------|
| `entity_id` | `sales_order.entity_id` | Primary key |
| `status` | `sales_order.status` | Order status |
| `increment_id` | `sales_order.increment_id` | Customer-facing order number |
| `customer_id` | `sales_order.customer_id` | FK to customer |
| `customer_email` | `sales_order.customer_email` | Flattened for search |
| `store_id` | `sales_order.store_id` | FK to store |
| `base_grand_total` | `sales_order.base_grand_total` | Currency-normalized total |
| `grand_total` | `sales_order.grand_total` | Display total |
| `created_at` | `sales_order.created_at` | Order creation time |
| `updated_at` | `sales_order.updated_at` | Last modification |

### Why Grid Data Goes Stale

After certain operations, `sales_order_grid` is not automatically updated:

- **Direct SQL updates** to `sales_order` bypassing Magento models
- **Database triggers** failing or not defined
- **Partial indexer failures** during `setup:upgrade` or `indexer:reindex`
- **Manual data manipulation** in the database directly

**Indexer responsible for grid:**

```bash
bin/magento indexer:reindex sales_order_grid
```

### Refreshing Stale Grid Data

**Manual reindex:**

```bash
# Single indexer
bin/magento indexer:reindex sales_order_grid

# All order-related indexers
bin/magento indexer:reindex sales_order_grid sales_order_address_grid
```

**Programmatic full reindex:**

```php
use Magento\Sales\Model\Indexer\Order\Grid\Runner;

$this->indexerRegistry->get(\Magento\Sales\Model\Indexer\Order\Grid\Indexer::INDEXER_ID)->reindexAll();
```

**Partial refresh for specific orders:**

```sql
-- If you know the order IDs, update them directly
UPDATE sales_order_grid sog
JOIN sales_order so ON sog.entity_id = so.entity_id
SET
    sog.status = so.status,
    sog.base_grand_total = so.base_grand_total,
    sog.grand_total = so.grand_total,
    sog.updated_at = NOW()
WHERE so.entity_id IN (123, 456, 789);
```

---

## 11. Database Slow Query Log

### `my.cnf` Configuration

Enable slow query logging in MySQL to capture queries exceeding a threshold:

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1  # Queries taking > 1 second
log_queries_not_using_indexes = 1  # Can generate noise; use temporarily
```

**For MySQL 8.0+, use dynamic variables:**

```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
```

### What to Look For

**Top offenders in slow.log:**

```bash
# Extract the query and execution time
grep "Query_time" /var/log/mysql/slow.log \
  | awk -F'Query_time: ' '{print $2}' \
  | sort -rn | head -20
```

**Common Magento slow query patterns:**

```sql
-- 1. EAV attribute filter without proper index
SELECT * FROM catalog_product_entity_int
WHERE attribute_id = 93 AND value = 1;

-- Fix: Ensure composite index on (attribute_id, value, entity_id)

-- 2. Category product join without index
SELECT e.* FROM catalog_product_entity e
INNER JOIN catalog_category_product cp ON e.entity_id = cp.product_id
WHERE cp.category_id = 12;

-- Fix: Ensure PRIMARY KEY on (product_id, category_id) and index on category_id

-- 3. Full table scan for customer search
SELECT * FROM customer_entity WHERE email LIKE '%@example.com%';

-- Fix: MySQL can't use index for leading wildcard; consider fulltext index
```

**mysqldumpslow (MySQL utility) for analysis:**

```bash
# Summarize slow queries
mysqldumpslow /var/log/mysql/slow.log

# Top 10 slowest queries
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# Queries with lock time > 0
mysqldumpslow -s l -t 10 /var/log/mysql/slow.log
```

---

## 12. Mysqldump for Production

### Essential Flags

For large Magento databases (>10GB), standard `mysqldump` can cause table locks, memory exhaustion, and long downtime windows.

**`--single-transaction`** — For InnoDB tables, wraps the dump in a transaction, providing a consistent snapshot without locking:

```bash
mysqldump --single-transaction \
  --host=localhost \
  --user=magento_user \
  --password=xxx \
  magento_database > /backup/magento_$(date +%Y%m%d).sql
```

**`--quick`** — Prevents loading the entire result set into memory. Essential for large tables:

```bash
mysqldump --single-transaction --quick \
  --host=localhost \
  magento_database \
  catalog_product_entity > /backup/catalog_product_entity.sql
```

**`--lock-tables=false`** — When used with `--single-transaction`, prevents `FLUSH TABLES WITH READ LOCK`. Required for production to avoid blocking writes:

```bash
mysqldump --single-transaction --quick --lock-tables=false \
  --host=localhost \
  magento_database > /backup/magento.sql
```

### Partial Export with WHERE Clause

Export only recent orders (last 30 days) for testing:

```bash
mysqldump --single-transaction --quick --lock-tables=false \
  --host=localhost \
  --user=magento_user \
  magento_database \
  sales_order \
  --where="created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)" \
  > /backup/recent_orders.sql
```

### Production Backup Script

```bash
#!/bin/bash
# backup_magento.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/mysql"
DB_NAME="magento_database"
DB_USER="magento_backup_user"
DB_PASS="xxx"

mkdir -p $BACKUP_DIR

# Dump all tables except log and quote tables (for smaller backup)
mysqldump --single-transaction --quick --lock-tables=false \
  --host=localhost \
  --user=$DB_USER \
  --password=$DB_PASS \
  --ignore-table=${DB_NAME}.mg_core_log \
  --ignore-table=${DB_NAME}.mg_logging_event \
  --ignore-table=${DB_NAME}.report_event \
  --ignore-table=${DB_NAME}.sales_flat_quote \
  --ignore-table=${DB_NAME}.sales_flat_quote_address \
  --ignore-table=${DB_NAME}.sales_flat_quote_item \
  --ignore-table=${DB_NAME}.sales_flat_quote_payment \
  ${DB_NAME} | gzip > ${BACKUP_DIR}/magento_${DATE}.sql.gz

# Keep last 7 days
find $BACKUP_DIR -name "magento_*.sql.gz" -mtime +7 -delete

echo "Backup complete: ${BACKUP_DIR}/magento_${DATE}.sql.gz"
```

---

## 13. Connection Issues

### `SQLSTATE[HY000] [2002] Connection refused`

This error means MySQL is not accepting connections at the socket or port level:

```
SQLSTATE[HY000] [2002] Connection refused
```

**Common causes:**

1. **MySQL not running:**

```bash
systemctl status mysql  # or mysqld
systemctl restart mysql
```

2. **Wrong socket path in `app/etc/env.php`:**

```php
// Incorrect
'connection' => [
    'host' => 'localhost',
    'unix_socket' => '/var/run/mysqld/mysqld.sock',  # Wrong path
],

// Find correct socket
mysql -e "SELECT @@socket"
# Output: /var/lib/mysql/mysql.sock
```

3. **Port blocked by firewall (if using 127.0.0.1:3306 instead of socket):**

```bash
# Check if port 3306 is listening
ss -tlnp | grep 3306
```

### Max Connection Errors

```
SQLSTATE[08006] [1040] Too many connections
```

**Cause:** All `max_connections` slots are in use.

**Immediate fix:**

```sql
-- Show current connections
SHOW PROCESSLIST;

-- Kill a stuck query
KILL 12345;

-- Or kill all sleeping connections
KILL CONNECTION FOR every row WHERE Command = 'Sleep';
```

**Long-term fix:**

```php
// In app/etc/env.php — reduce FPM child processes per connection
// Or increase max_connections in my.cnf
'model' => 'mysql4',
'connection' => [
    'max_connections' => 1000,  // Increase from 500
],
```

### Magento Connection Retry Logic

`Magento\Framework\DB\Adapter\Pdo\Mysql` implements automatic reconnection for transient failures:

```php
// Magento\Framework\DB\Adapter\Pdo\Mysql::connect()
public function connect()
{
    if ($this->_connection) {
        return;
    }

    try {
        $this->_connection = new \PDO(
            $this->_dsn,
            $this->_config['username'],
            $this->_config['password'],
            $this->_options
        );
        $this->_connection->setAttribute(\PDO::ATTR_ERRMODE, \PDO::ERRMODE_EXCEPTION);
    } catch (\PDOException $e) {
        // Retry logic handled in _retryConnection
        $this->_retryConnect();
    }
}
```

For Magento 2.4.8, the connection retry logic specifically handles:
- `2006` — MySQL server has gone away
- `2013` — Lost connection during query
- `2002` — Connection refused (on retry, if socket issue was transient)

---

## 14. Read Replicas

### Magento's Read/Write Connection Split

Magento 2.4.8 supports directing read queries to a separate replica connection defined in `app/etc/env.php`:

```php
// app/etc/env.php
return [
    'db' => [
        'table_prefix' => '',
        'connection' => [
            'default' => [
                'host' => '10.0.1.100',        // Primary (write)
                'port' => '3306',
                'dbname' => 'magento',
                'username' => 'magento_user',
                'password' => 'xxx',
                'model' => 'mysql4',
                'initStatements' => 'SET NAMES utf8mb4',
                'active' => '1',
            ],
            'indexer' => [
                'host' => '10.0.1.101',        // Read replica
                'port' => '3306',
                'dbname' => 'magento',
                'username' => 'magento_replica_user',
                'password' => 'xxx',
                'model' => 'mysql4',
                'initStatements' => 'SET NAMES utf8mb4',
                'active' => '1',
            ],
        ],
    ],
];
```

**How Magento routes queries:**

```php
// Magento\Framework\App\ResourceConnection
public function getConnection($connectionName = null)
{
    // If 'indexer' is specified and available, use it
    if ($connectionName === 'indexer' && isset($this->_config['indexer'])) {
        return $this->_getConnection('indexer');
    }

    // Default goes to primary
    return $this->_getConnection('default');
}
```

### What Uses Which Connection

| Operation | Connection |
|-----------|------------|
| Product save | `default` (write) |
| Order placement | `default` (write) |
| Category page load | `indexer` (read, when configured) |
| Product listing | `indexer` (read, when configured) |
| Admin grid load | `indexer` (read, when configured) |
| Indexer processes | `indexer` (read) |

### Limitations in Magento 2.4.8

Magento's read replica support is **limited and not fully implemented** in the core codebase:

1. **No automatic read/write splitting** — There is no middleware or plugin that automatically routes `SELECT` queries to the replica. The `indexer` connection is used only when explicitly requested.
2. **Session and cache** still use primary for writes
3. **Transactions** must use the primary connection
4. **Most collection loads** do not respect replica routing

For full read/write splitting, you need:
- External proxy (ProxySQL, MariaDB MaxScale)
- Or custom plugin on `Magento\Framework\DB\Adapter\Pdo\Mysql` to randomly route SELECTs

### Verifying Replica Lag

If using a real MySQL replica (not just a separate instance), monitor replication lag:

```sql
-- On replica
SHOW SLAVE STATUS\G
-- Look for: Seconds_Behind_Master: 0

-- Or in MySQL 8.0
SHOW REPLICA STATUS\G
-- Look for: Replica_IO_Running: Yes, Replica_SQL_Running: Yes
```

High lag (>30 seconds) means read queries on the replica may return stale data (e.g., an order placed 30 seconds ago not appearing in the admin grid).

---

## 15. Advanced Diagnostic Techniques

### Query Logging with Timeline

For correlating slow queries with request timeline:

```php
// Add to pub/index.php or a custom frontend controller
$profiler->start('db_query_' . microtime(true));
$connection->query($sql);
$profiler->stop('db_query_' . microtime(true));
```

### Identifying Lock Waits

```sql
-- Current lock waits
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- Which transactions are waiting
SELECT
    r.trx_id,
    r.trx_mysql_thread_id,
    r.trx_query,
    r.trx_state,
    r.trx_started,
    r.trx_rows_locked,
    r.trx_tables_locked,
    r.trx_lock_memory
FROM information_schema.INNODB_TRX r
WHERE r.trx_state = 'LOCK WAIT';

-- Kill the blocking query
KILL 12345;  -- Use the trx_mysql_thread_id
```

### Table Metadata Lock Waits

```sql
-- Show metadata locks (MySQL 8.0+)
SELECT
    mt.OBJECT_SCHEMA,
    mt.OBJECT_NAME,
    mt.LOCK_TYPE,
    mt.LOCK_STATUS,
    mt.THREAD_ID,
    pro.SQL_TEXT
FROM information_schema.METADATA_LOCK_INFO mt
JOIN performance_schema.threads pro ON mt.THREAD_ID = pro.THREAD_ID;
```

---

## See Also

- [Indexer Management: 05-indexer-management.md](05-indexer-management.md) — Deeper coverage of indexer architecture, MVIEW mechanism, and custom indexer development
- [Performance Profiling: 07-performance-profiling.md](07-performance-profiling.md) — Using Blackfire.io, New Relic APM, and the built-in Profiler for end-to-end request analysis
- [Declarative Schema: _supplemental/04-declarative-schema.md](_supplemental/04-declarative-schema.md) — `db_schema.xml` for custom table and index definition in Magento 2.4.x
- [Redis/Varnish Caching: _supplemental/20-security-performance.md](_supplemental/20-security-performance.md) — Cache backend configuration for database query result caching
- [MySQL 8.0 Reference Manual: EXPLAIN Output](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html) — Full EXPLAIN column reference
- [Percona Blog: MySQL Indexing for Performance](https://www.percona.com/blog/mysql-indexing-for-performance/) — Deep-dive on MySQL index internals
