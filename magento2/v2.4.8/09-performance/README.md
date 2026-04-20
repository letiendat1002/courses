# Topic 9: Performance Optimization

**Goal:** Understand and fix common Magento performance bottlenecks — cache layers, query optimization, indexing strategy, and profiling tools.

---

## Topics Covered

- Magento cache layers — which cache to use when and why each layer exists
- Full-page caching (Varnish vs Redis) and block caching (`cacheable="false"` caveats)
- Database query optimization — N+1 problems, eager loading, EXPLAIN analysis
- Indexing strategy — realtime vs schedule trade-offs
- Profiling with built-in tools (`bin/magento profiler`) and production tools (Blackfire/New Relic)
- Image optimization and lazy loading
- JavaScript bundling and CSS minification
- Redis configuration for session and cache (connection pooling, monitoring)
- Asynchronous operations for heavy tasks via message queue

---

## Reference Exercises

- **Exercise 8.1:** Enable all cache types, verify `var/cache` is populated
- **Exercise 8.2:** Add block cache to a custom block via `cache.xml` with proper cache tags
- **Exercise 8.3:** Identify and fix a slow query using `db_schema.xml` indexing and EXPLAIN
- **Exercise 8.4:** Configure Redis for session and cache backend with separate databases
- **Exercise 8.5:** Set up a cron job asynchronously using message queue for bulk operations
- **Exercise 8.6:** Run the Magento profiler, interpret the output and identify N+1 queries
- **Exercise 8.7 (bonus):** Set up Blackfire or New Relic on a local environment and profile a category page

---

## Completion Criteria

- [ ] Block caching configured in `cache.xml` for a custom block with proper cache tags
- [ ] Cache warms after first page load (subsequent loads served from cache)
- [ ] Slow query identified via logging or MySQL EXPLAIN, index added via `db_schema.xml`
- [ ] Redis configured as cache backend with separate databases for cache/pagecache/sessions
- [ ] Asynchronous bulk operation processes via message queue (bulk operations do not block HTTP)
- [ ] `bin/magento cache:flush` clears all cache successfully
- [ ] New Relic or Blackfire profile shows < 500ms for category page (FPC miss) and < 50ms (FPC hit)
- [ ] No `cacheable="false"` blocks on any customer-facing pages (verified via profiler output)
- [ ] All collections use `setPageSize()` with maximum limits to prevent unbounded memory growth
- [ ] Varnish or Redis FPC is verified working via `curl -I` showing `X-Magento-Cache: HIT` header

---

## Topics

---

### Topic 1: Magento Cache Layers

**Why Cache Layers Matter in Magento 2:**

Magento 2's architecture is designed around cache layers as a first-class concern. Unlike simpler applications where caching is an afterthought, Magento's entire request lifecycle is built to leverage multiple cache tiers. Understanding why each layer exists helps you make better architecture decisions:

| Cache Layer | Why It Exists | What Happens Without It |
|-------------|---------------|------------------------|
| `config` | XML files are parsed on every request | Full XML re-parse + DI compilation on each page load (~500ms+ overhead) |
| `layout` | Layout XML is compiled into PHP objects | Layout tree rebuilt from XML on every request |
| `block_html` | Block HTML is rendered via PHP templates | Every template re-executed, every PHP object re-instantiated |
| `full_page` | Entire HTML page cached | Full Stack PHP execution + template rendering on every request |
| `collections` | Collection metadata (schema, filters) | Database introspection queries repeated |

**Pro Tip: Cache Stacking**

Magento's cache system stacks multiple cache backends. The typical production stack:
```
Request → HTTP → Varnish (CDN/Edge) → Redis (FPC) → Redis (Cache) → File Cache (fallback) → MySQL
```
Each layer serves a different purpose. Varnish caches at the HTTP layer (static assets, full pages). Redis caches at the application layer (block HTML, config, sessions). Understanding this stack is critical for debugging "why isn't my cache invalidating" issues.

**Cache Types:**

| Cache Type | Purpose | Default TTL | Invalidation Trigger |
|-----------|---------|------------|---------------------|
| `config` | XML config files | 86400 (24h) | `bin/magento cache:flush config`, config save |
| `layout` | Layout XML compiled | 86400 | `bin/magento cache:flush layout`, layout XML change |
| `block_html` | Block HTML output | 86400 | `bin/magento cache:flush block_html`, block cache tags |
| `full_page` | Full-page cache (FPC) | 86400 | `bin/magento cache:flush full_page`, page-specific tags |
| `collections` | Database collections | 86400 | `bin/magento cache:flush collections` |
| `reflection` | API reflection | 86400 | `bin/magento cache:flush reflection` |
| `webservice` | API introspection | 86400 | `bin/magento cache:flush webservice` |

**Why Separate Cache Types?**

Each cache type has different invalidation characteristics. The `config` cache is invalidated by system configuration changes (which happen infrequently but broadly). The `block_html` cache is invalidated by product/category changes (which happen more frequently but scoped to specific cache tags). Mixing these would cause over-invalidation.

**Viewing Cache Status:**

```bash
bin/magento cache:status
bin/magento cache:enable  layout block_html
bin/magento cache:disable config
```

**Cache Commands:**

```bash
bin/magento cache:flush     # Clear everything (all cache types)
bin/magento cache:clean    # Clear stale only
bin/magento cache:enable  # Enable specific cache
bin/magento cache:disable # Disable specific cache
```

**When to Flush Cache:**

| Change | Flush? |
|--------|--------|
| Edited PHTML template | Yes (`block_html`) |
| Changed layout XML | Yes (`layout`) |
| Changed system config | Yes (`config`) |
| Edited controller | Yes (`di`) |
| Edited block class | Yes (`block_html`) |
| New product saved | Yes (related caches) |

---

### Topic 2: Block & Full-Page Caching

**Block Caching — `cache.xml`:**

```xml
<!-- app/code/Training/Review/etc/cache.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Cache/etc/cache.xsd">
    <placeholder name="training_review_block_message">
        <tag>training_review</tag>
        <tag>catalog_product</tag>
    </placeholder>
</config>
```

**Block with Cache Tags in Code:**

```php
<?php
// Block/CachedMessage.php
namespace Training\Review\Block;

use Magento\Framework\View\Element\Template;
use Magento\Catalog\Model\Product;

class CachedMessage extends Template
{
    public function getCacheLifetime(): ?int
    {
        return 3600; // 1 hour, null = infinite
    }

    public function getCacheTags(): array
    {
        return array_merge(parent::getCacheTags(), [
            \Magento\Catalog\Model\Product::CACHE_TAG . '_' . $this->getProductId()
        ]);
    }

    public function getCacheKeyInfo(): array
    {
        return array_merge(parent::getCacheKeyInfo(), [
            'product_id' => $this->getProductId()
        ]);
    }

    public function getIdentities(): array
    {
        return array_merge(parent::getIdentities(), [
            Product::CACHE_TAG . '_' . $this->getProductId()
        ]);
    }
}
```

**Template Cache Hint:**

```php
<?php // In template, add comment to identify cache issues ?>
<!-- cacheable="true" tag="training_review" lifetime="3600" -->
```

**Important: `cacheable="false"` — The Silent Performance Killer**

The `cacheable="false"` attribute is one of the most dangerous performance anti-patterns in Magento 2. Here's why:

```xml
<!-- DANGER: This block can NOT be cached. Use sparingly. -->
<block class="Training\Review\Block\Dynamic" cacheable="false" .../>
```

**Why `cacheable="false"` Breaks Full-Page Caching:**

When any single block on a page has `cacheable="false"`, the **entire page** cannot be full-page cached. This is not an exaggeration — Magento checks for any uncacheable block and disables FPC for the whole page.

The root cause is in `Magento\Framework\View\Result\Page::renderResult()`. It checks:
```php
if (!$page->getLayout()->getNode()->getAttribute('cacheable')) {
    // FPC is disabled for this page
    $this->response->setNoRender(true);
}
```

**Industry Best Practice: When to Use `cacheable="false"`**

| Use Case | Recommendation | Alternative |
|----------|----------------|-------------|
| Customer-specific content (cart, wishlist, hello name) | ❌ DON'T use `cacheable="false"` | Use `cache_lifetime` + `cache_tags` with customer-specific tags |
| Admin-related blocks | ✅ OK — admin is never FPC-cached anyway | N/A |
| Frequently changing data (stock ticker, live chat) | ❌ DON'T use `cacheable="false"` | Use AJAX/knockout loading post-render |
| Form content (login, register, contact) | ❌ DON'T use `cacheable="false"` | Use `cache_lifetime: 0` with ESI or AJAX |
| Dynamic pricing blocks | ❌ DON'T use `cacheable="false"` | Use `customer-specific` cache tags |

**Pro Tip: The `Vary` Header Pattern**

Instead of `cacheable="false"`, use cache variation headers:
```php
// In your block's getCacheKeyInfo()
return array_merge(parent::getCacheKeyInfo(), [
    'customer_group' => $this->_customerSession->getCustomerGroupId(),
    'store_id' => $this->_storeManager->getStore()->getId(),
]);
```

This generates separate cached pages per customer group, still enabling FPC.

**Full-Page Cache (FPC) Configuration:**

```bash
# Enable FPC
bin/magento cache:enable full_page

# Or in env.php:
'system' => [
    'default' => [
        'system' => [
            'full_page_cache' => [
                'caching_application' => 2  # 2 = Redis, 1 = Varnish
            ]
        ]
    ]
]
```

**FPC Deep Dive: Varnish vs Redis**

Magento 2 supports two FPC backends:

| Backend | How It Works | Pros | Cons |
|---------|--------------|------|------|
| **Varnish** (recommended) | HTTP proxy caching | Edge caching, HTTP compression, ESI support, surrogate keys | Requires separate process, more complex setup |
| **Redis** | Application-level page caching | Simple setup, integrated with Magento, no separate process | No edge caching, all requests hit app server |

**Varnish Cache Invalidation:**

Varnish uses purging (not flushing) to invalidate cached pages:
```bash
# Purge a specific URL
bin/magento cache:clean full_page

# Or via Varnish admin
varnishadm "req.url ~ /" purge
```

**Pro Tip: Tag-Based Cache Invalidation (Better Than URL-Based)**

Magento's cache infrastructure supports tag-based invalidation, which is more reliable than URL-based purging:
```bash
# Clean cache by tags
bin/magento cache:clean zenddir

# In PHP, invalidate by tags
$this->cache->clean(\Magento\Framework\App\Cache\Type\Page::CACHE_TAG, ['catalog_product_123']);
```

**Common FPC Pitfall: Private Data in Cached Pages**

If your page shows customer-specific data (name, cart count), you MUST use Knockout.js to hydrate this data client-side after the cached HTML is delivered. The FPC stores the HTML shell; private data loads via AJAX.

---

### Topic 3: Database Query Optimization

**Finding Slow Queries:**

```bash
# Enable MySQL slow query log
bin/magento setup:config:set --sql-log-enable=1

# Or in env.php:
'db' => [
    'connection' => [
        'default' => [
            'profiler' => [
                'class' => '\Magento\Framework\DB\Profiler',
                'enabled' => true,
            ]
        ]
    ]
],
```

Then check `var/log/debug.db.log`.

**EXPLAIN Analysis:**

```sql
EXPLAIN SELECT * FROM training_review WHERE product_id = 1 ORDER BY created_at DESC;
```

Look for:
- `type: ALL` — full table scan → needs index
- `rows: > 1000` — too many rows examined
- `Extra: Using filesort` — slow sort operation

**Adding Indexes via `db_schema.xml`:**

```xml
<index referenceId="TRAINING_REVIEW_PRODUCT_ID_INDEX"
       tableName="training_review"
       indexType="btree">
    <column name="product_id"/>
</index>

<index referenceId="TRAINING_REVIEW_CREATED_AT_INDEX"
       tableName="training_review"
       indexType="btree">
    <column name="created_at"/>
</index>

<index referenceId="TRAINING_REVIEW_COMPOSITE_INDEX"
       tableName="training_review"
       indexType="btree">
    <column name="product_id"/>
    <column name="created_at"/>
</index>
```

**Query Optimization Rules:**

| Problem | Solution |
|---------|---------|
| `WHERE product_id = X` slow | Add index on `product_id` |
| `ORDER BY created_at` slow | Composite index on `(product_id, created_at)` |
| `LIKE '%keyword%'` slow | Full-text index or Elasticsearch |
| JOIN without index | Index foreign key columns |
| `COUNT(*)` on large table | Cache count result, invalidate on change |

**Collection Optimization:**

```php
// Bad: loads all data
$collection = $this->collectionFactory->create();
foreach ($collection as $item) { /* ... */ }

// Good: select only needed columns
$collection = $this->collectionFactory->create();
$collection->addFieldToSelect(['review_id', 'reviewer_name', 'rating']);
$collection->setPageSize(20)->setCurPage(1);

// Good: use getItems() for lightweight iteration
$items = $collection->getItems();
```

**N+1 Query Problem: The Silent Scalability Killer**

The N+1 query problem is the most common database performance issue in Magento. It occurs when loading a collection, then lazily loading related entities one-by-one:

```php
// BAD: N+1 problem — 1 query for reviews + N queries for products
$reviews = $this->reviewCollectionFactory->create()->load();
foreach ($reviews as $review) {
    // Each getProduct() triggers a separate query!
    $product = $review->getProduct();
    printf("Review: %s, Product: %s\n", $review->getTitle(), $product->getName());
}
// Total queries: 1 + N (where N = number of reviews)
```

**Fix: Eager Loading with `joinTable` or `getProduct()` Preload:**

```php
// GOOD: Eager load products in a single JOIN query
$reviews = $this->reviewCollectionFactory->create();
$reviews->addAttributeToSelect(['title', 'rating']);
$reviews->getProductCollection()->addAttributeToSelect(['name', 'price']);

// OR: Use audit() to inspect query count
$reviews->load();
echo "Query count: " . $reviews->getQuery()->getSize();

// GOOD: Manually preload all products
$productIds = $reviews->getColumnValues('product_id');
$this->productRepository->getByIds($productIds); // Preload all at once
foreach ($reviews as $review) {
    $review->setProduct($this->productRepository->get($review->getProductId()));
}
```

**Industry Best Practice: Query Logging in Development**

Enable query logging to catch N+1 issues before they reach production:

```php
// In your development env.php
'dev' => [
    'db' => [
        'profiler' => [
            'class' => '\Magento\Framework\DB\Profiler',
            'enabled' => true,
        ]
    ]
],

// Then check in your code
/** @var \Magento\Framework\DB\Adapter\AdapterInterface $connection */
$connection = $this->resourceConnection->getConnection();
$profiler = $connection->getProfiler();
echo "Query count: " . $profiler->getTotalNumQueries();
```

**Query Optimization Checklist (Magento-specific):**

| Check | How to Verify | Performance Impact |
|-------|---------------|-------------------|
| `SELECT *` in collections | PHPDoc review | High — loads unused columns |
| Missing `addAttributeToSelect` | Review collection code | Medium — loads all EAV attributes |
| N+1 on product/category | Profiler query count | Critical — multiplies queries |
| No pagination on large collections | Check `setPageSize()` usage | Critical — memory exhaustion |
| Missing indexes | `EXPLAIN` output | Critical — full table scans |

---

### Topic 4: Redis Configuration

**Why Redis?**

| Storage | Without Redis | With Redis |
|---------|--------------|-----------|
| Cache | File system (slow) | Memory (fast) |
| Sessions | File system (slow, disk I/O) | Memory (fast) |
| Page cache | File system | Memory (fast) |

**Docker Compose Redis Service:**

```yaml
redis:
  image: redis:7-alpine
  container_name: magento2-redis
  ports:
    - "6379:6379"
```

**Magento Redis Configuration in `env.php`:**

```php
'cache' => [
    'frontend' => [
        'default' => [
            'backend' => 'Magento\Framework\Cache\Backend\Redis',
            'backend_options' => [
                'server' => 'redis',
                'port' => 6379,
                'database' => 0,
                'password' => '',
                'compress_data' => true,
                'compression_lib' => 'gzip',
            ]
        ],
        'page_cache' => [
            'backend' => 'Magento\Framework\Cache\Backend\Redis',
            'backend_options' => [
                'server' => 'redis',
                'port' => 6379,
                'database' => 1,
                'compress_data' => 2,
                'compression_lib' => 'gzip',
            ]
        ]
    ]
],

'session' => [
    'save' => 'redis',
    'redis' => [
        'host' => 'redis',
        'port' => 6379,
        'database' => 2,
        'log_level' => 'info',
    ]
],
```

**Verifying Redis:**

```bash
# Connect to Redis container
docker compose exec redis redis-cli

# Check keys
KEYS *        # All keys
KEYS magento:*  # Magento keys
INFO clients  # Number of connected clients
```

---

### Topic 5: Profiling with Magento Tools

**Why Profiling is Essential for Magento Performance**

Magento 2 is a complex, multi-tier PHP application. A single page request can execute 500+ SQL queries, render 100+ blocks, and involve dozens of event observers. Without profiling, you're optimizing blind. The difference between a slow Magento site and a fast one is almost never "which framework" — it's always "which specific query/block/event is the bottleneck."

**Industry Best Practice: Profile Before Optimizing**

Always profile before making optimization changes:
1. Run Blackfire/New Relic to identify the top 3 slowest transaction
2. For each slow transaction, identify the specific SQL query or block rendering
3. Optimize only the identified bottleneck
4. Re-profile to verify improvement

This avoids the common mistake of spending days optimizing something that accounts for 2% of request time.

**Enabling the Profiler:**

```bash
# Enable
bin/magento dev:profiler:enable

# Disable
bin/magento dev:profiler:disable
```

**Profiler Output (HTML mode):**

Access `?profile=1` on any page to see:
- Timers — how long each step took
- SQL queries — all database queries with execution time
- Block rendering — which blocks rendered and how long
- Memory usage — peak memory consumption

**Code Profiling in Your Own Classes:**

```php
<?php
use Magento\Framework\Profiler;

// In your method
Profiler::start('training_review_process');
try {
    // ... work ...
} finally {
    Profiler::stop('training_review_process');
}
```

**Production Profiling: Blackfire and New Relic Integration**

Magento 2 has built-in support for Blackfire.io and New Relic. These tools provide production-grade profiling without developer overhead.

**Blackfire.io Setup:**

```bash
# Install Blackfire PHP agent
curl -L https://blackfire.io/api/v1/releases/probe/php/linux/amd64/$(php -r 'echo phpversion();') | php

# Install Blackfire CLI
curl -L https://blackfire.io/api/v1/releases/client/linux/amd64 | php - version
```

```php
// In your PHP code (automatic via Magento integration)
// Blackfire automatically instruments:
// - SQL queries with timing + call stack
// - HTTP requests
// - Redis/Cache operations
// - Custom timers via Blackfire SDK

use BlackfirePhp\Probe;

$probe = new Probe();
$probe->start('my_operation');
// ... your code ...
$probe->end();
```

**New Relic Setup:**

```bash
# Install New Relic PHP agent
yum install -y newrelic-php5

# Configure in php.ini
newrelic.license = "YOUR_LICENSE_KEY"
newrelic.appname = "Magento2_Production"
newrelic.framework = "magento2"
```

New Relic automatically instruments:
- Transaction traces (slowest requests)
- SQL query traces with explain plans
- External service calls
- Error rates and exceptions
- Custom events via `newrelic_add_custom_parameter()`

**Pro Tip: Custom New Relic Instrumentation for Magento:**

```php
<?php
// In your observer or plugin
if (extension_loaded('newrelic')) {
    newrelic_add_custom_parameter('quote_id', $quote->getId());
    newrelic_add_custom_parameter('customer_group', $quote->getCustomerGroupId());
    newrelic_record_custom_event('CheckoutStep', [
        'step' => 'shipping_method',
        'quote_total' => $quote->getGrandTotal(),
    ]);
}
```

**Reading Profiler Output: Real-World Example**

When you access `?profile=1` on a Magento 2 page with the built-in profiler, you'll see:

```
Timer                                      Cnt       Summary
---------------------------------------------------------
mage.dispatch.controller.pre dispatch  1       0.0012s
mage.app                                      1       0.234s
    catalog_category_view_id_3              1       0.198s
        layout.render                        1       0.045s
            catalog_product_view             1       0.089s
                Magento\Catalog\Block\Product\View   12    0.023s
                    catalog_product_price    12    0.012s
```

**What to Look For:**
1. Timers with high `Cnt` (count) — called many times, potential N+1
2. Timers with high `Summary` — slow operations
3. `mage.db.statement` entries — SQL queries with execution time

**Output to File:**

```bash
bin/magento dev:profiler:enable file
tail -f var/log/profiler.log
```

---

### Topic 6: Asynchronous Operations with Message Queue

**Why Async?**

Heavy operations (bulk email, mass indexing, export) can block HTTP requests. Moving them to a queue keeps the request fast.

**Message Queue Architecture:**

```
Producer → Queue → Consumer (async worker)
```

**Queue Configuration — `queue.xml`:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:MessageQueue/etc/queue.xsd">
    <queue name="training_review_bulk_export"
           connection="db"
           exchange="magento-export"
           topic="training.review.bulk_export">
        <consumer name="Training_Review_Consumer_BulkExport"
                  queue="training_review_bulk_export"
                  maxMessages="100"
                  instance="Training\Review\Model\Async\BulkExportConsumer"/>
    </queue>
</config>
```

**Publish to Queue:**

```php
<?php
// Publisher/PublishBulkExport.php
namespace Training\Review\Model\Publisher;

use Magento\Framework\MessageQueue\PublisherInterface;

class PublishBulkExport
{
    protected $publisher;

    public function __construct(PublisherInterface $publisher)
    {
        $this->publisher = $publisher;
    }

    public function publish(array $reviewIds): void
    {
        $this->publisher->publish('training.review.bulk_export', [
            'review_ids' => $reviewIds,
            'initiator' => 'admin_user_id_' . $this->getCurrentUserId(),
            'timestamp' => time()
        ]);
    }
}
```

**Consumer:**

```php
<?php
// Model/Async/BulkExportConsumer.php
namespace Training\Review\Model\Async;

use Magento\Framework\MessageQueue\ConsumerInterface;
use Psr\Log\LoggerInterface;

class BulkExportConsumer implements ConsumerInterface
{
    protected $logger;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function processMessage(array $message): void
    {
        $reviewIds = $message['review_ids'];
        $this->logger->info('Processing bulk export for ' . count($reviewIds) . ' reviews');

        // Expensive export work
        foreach ($reviewIds as $reviewId) {
            // Generate export row...
        }

        $this->logger->info('Bulk export complete');
    }
}
```

**Running the Consumer:**

```bash
# Run continuously (for long-running worker)
bin/magento queue:consumers:start Training_Review_Consumer_BulkExport

# Or run once (for cron-based)
bin/magento queue:consumers:run Training_Review_Consumer_BulkExport --max-messages=100
```

---

## Reading List

- [Caching](https://developer.adobe.com/commerce/php/development/components/cache/) — Block and page caching
- [Cache Configuration](https://experienceleague.adobe.com/docs/commerce-operations/configuration-guide/cache/configure-redis.html) — Redis setup
- [Message Queue](https://developer.adobe.com/commerce/php/development/components/message-queue/) — Async consumers
- [Profiling](https://experienceleague.adobe.com/docs/commerce-operations/configuration-guide/debug/debug.html) — Profiler setup

---

## Edge Cases & Troubleshooting

| Issue | Symptom | Solution |
|-------|---------|----------|
| Cache not working | Page still slow after FPC enabled | Check for `cacheable="false"` in layout XML — entire page affected |
| Redis not connecting | Cache falls back to file, very slow | Check Redis is running: `docker compose ps`; verify `redis-cli ping` |
| Slow query persists | EXPLAIN shows no index | Add index to `db_schema.xml`, run `setup:upgrade`, then re-run EXPLAIN |
| Profiler output messy | Too much data | Filter by timer name: `?profile=1&timer=sql` or `?profile=1&timer=block` |
| Queue consumer timeout | Messages pile up in `queue_message` table | Increase `maxMessages`, check consumer for long-running DB queries |
| Block cache not invalidating | Stale data shown | Block cache tags must match invalidated tags; use `getIdentities()` |
| N+1 query problem | 500+ queries on single page | Use eager loading (`joinTable`) or `getProductCollection()` preload |
| FPC not generating HIT headers | `curl -I` shows MISS | Check `system/full_page_cache/caching_application` config; verify cache enabled |
| Redis memory growing unbounded | `redis-cli INFO memory` shows high | Set `maxmemory-policy allkeys-lru` in Redis config |
| Collection memory issue | Out of memory on large catalog | Always use `setPageSize()`; never iterate unbounded collections |

---

## Common Mistakes to Avoid

1. ❌ Using `cacheable="false"` too broadly → Entire page can't be cached
   - **Why it matters:** Even one uncacheable block disables FPC for the entire page. A single `cacheable="false"` on a product page can add 300ms+ to every request.
   - **Fix:** Use customer-group-specific cache tags instead of `cacheable="false"`.

2. ❌ Not clearing cache after config changes → Stale config served
   - **Why it matters:** The `config` cache holds compiled system configuration. Admin changes don't auto-invalidate.
   - **Fix:** Always flush `config` cache after any `core_config_data` change via deployment script or admin.

3. ❌ Large collections without pagination → Memory exhausted
   - **Why it matters:** Magento collections lazy-load. Without `setPageSize()`, a collection with 100,000 products will instantiate all 100,000 ORM objects.
   - **Fix:** Always use `setPageSize(20-100)` and `setCurPage($page)`.

4. ❌ Missing database indexes → Slow queries on production scale
   - **Why it matters:** A query that takes 10ms with 100 rows takes 5000ms with 1,000,000 rows without proper indexing.
   - **Fix:** Run `EXPLAIN` on all queries touching large tables; add indexes via `db_schema.xml`.

5. ❌ Forgetting Redis password in production → Security vulnerability
   - **Why it matters:** Exposed Redis without password allows data exfiltration of all sessions and cache.
   - **Fix:** Always use strong Redis passwords; restrict Redis to internal network only.

6. ❌ Sync operations for bulk email → HTTP timeouts
   - **Why it matters:** Sending 1000 emails synchronously takes 100+ seconds, causing HTTP timeouts.
   - **Fix:** Use the message queue (`queue.xml`) for async email sending.

7. ❌ Using `SELECT *` in collections → Unnecessary memory + I/O
   - **Why it matters:** `SELECT *` loads all columns including heavy TEXT/BLOB fields (description, custom_attributes).
   - **Fix:** Always use `addFieldToSelect(['column1', 'column2'])` on collections.

8. ❌ Bypassing the totals collector → Inconsistent totals
   - **Why it matters:** If you manually set totals without calling `collectTotals()`, other totals become stale.
   - **Fix:** Always call `$quote->collectTotals()` after manual total modifications.

---

*Magento 2 Backend Developer Course — Topic 09*
