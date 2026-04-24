---
title: "07 - Performance Profiling"
description: "Magento 2.4.8 performance profiling: Blackfire.io, New Relic APM, built-in Profiler, database slow query log, cache analysis, and interpreting APE (Application Performance) reports"
tags: [magento2, profiling, blackfire, new-relic, performance, slow-query, cache]
rank: 7
pathways: [magento2-deep-dive]
see_also:
  - "Redis/Varnish caching: _supplemental/06-caching.md"
  - "Database optimization: _supplemental/04-declarative-schema.md"
---

# 07 — Performance Profiling

Performance profiling is the disciplined practice of measuring where time goes inside a running Magento request. Without profiling data, you are guessing — guessing that the database is slow, guessing that a third-party extension is the culprit, guessing that the cache is cold. Guessing wastes days. Profiling tells you exactly where to act.

This chapter covers the four tools that together give you complete visibility: the built-in Magento Profiler (zero-install, always available), Blackfire.io (targeted ad-hoc profiling with deep PHP timeline), New Relic APM (continuous production monitoring), and MySQL slow-query analysis (database-level diagnostics). You will also learn how to read cache hit rates, interpret Blackfire comparison reports, and apply the full profile → hypothesize → change → re-profile cycle to real scenarios like a slow product listing page.

---

## 1. Profiling Fundamentals

### The Four Bottleneck Categories

Every PHP request bottleneck falls into one of four categories:

| Bottleneck | Description | Magento Example |
|---|---|---|
| **CPU** | Heavy computation — loops, encryption, search | Running price rules on a catalog with 50,000 SKUs |
| **I/O — Database** | Queries waiting on disk or lock contention | N+1 queries loading product EAV attributes |
| **I/O — Network** | External API calls, Redis round-trips | Calling an ERP on every order save |
| **Memory** | Large object allocation, GC pressure | Rendering a category page with 500 products and full page cache disabled |

PHP per-request execution makes CPU and I/O the dominant factors. Unlike compiled languages where you can profile production at any moment, PHP is short-lived: each HTTP request starts fresh, does work, and dies. This means you need to reproduce the exact scenario to profile it — a blank homepage is useless for diagnosing a slow category page with 300 products and a full price rule chain.

### Amdahl's Law

Amdahl's Law governs how much speedup is possible from parallelization. If 90% of your request time is serial (must happen one step after another) and only 10% is parallelizable, adding 8 CPU cores gives you a theoretical maximum speedup of **1 / (0.9 + 0.1/8) = 1.1x** — not 8x. Before adding hardware or scaling horizontally, you must identify what fraction of time is truly serial.

For Magento, the serial portion is typically:
- Layout XML merging and block instantiation (one handle at a time)
- Database transactions with row-level locks
- Synchronous external API calls

The parallelizable portion:
- Rendering independent CMS blocks on the same page
- Bulk price indexing (indexer splits across CPU cores)
- Image resizing jobs processed by the async queue

### Profiling vs Monitoring

**APM tools** (New Relic, Datadog, Elastic) run continuously in production. They give you the big picture: which endpoints are slow, which hosts are saturated, how error rates trend over time. They are broad but shallow — they tell you the transaction was slow, not why.

**Profilers** (Blackfire, Xdebug, Tideways) are ad-hoc, point-in-time tools. You trigger a specific scenario, capture a detailed timeline, and inspect every function call, SQL query, and cache operation that occurred. They are narrow but deep — they show you the exact SQL being executed 847 times in one request.

Use APM for alerting and trend analysis in production. Use profilers for root-cause diagnosis when APM flags a problem.

---

## 2. Built-in Profiler (`Magento\Framework\Profiler`)

Magento ships a built-in profiler that requires zero external tools. It is always available and useful for quick ad-hoc checks without installing Blackfire or configuring New Relic.

### Enabling the Profiler

**Admin UI:**
Navigate to **Stores → Configuration → Advanced → Developer → Debug** and set **Enable Profiler** to `Yes`.

**CLI (recommended for automation and reproducibility):**

```bash
bin/magento config:set dev/debug/profiler 1
```

Disable with:

```bash
bin/magento config:set dev/debug/profiler 0
```

Enabling the profiler adds an HTML comment block and timing data to every rendered page. It also writes CSV timeline files to `var/log/profiler.csv`.

### What the Profiler Captures

The profiler instruments every major Magento lifecycle event:

- **Layout XML merge time** — how long each handle took to load and merge
- **Block instantiation** — each block class, its parent, and render time
- **Observer dispatch** — which observers fired on each event and how long each took
- **Plugin around-methods** — every before/around/after plugin call
- **Database queries** — query text and duration (requires separate DB logger)
- **File compilation** — PHP file stat checks and class compilation

### Profiler Output Files

When enabled, two output artifacts are produced:

```
var/log/profiler.csv          # CSV timeline, one row per event
var/profiler/                 # Directory with block-level timing HTML files
```

**Reading `profiler.csv`:**

```csv
Time[ms],Category,Operation,Details
0.145,layout,load,default
0.289,layout,load,catalog_category_view
0.441,block,instantiate,Magento\Catalog\Block\Product\ListProduct
0.889,observer,dispatch,catalog_product_load_after
1.203,plugin,around,Magento\Catalog\Model\Product::getPrice
```

The columns are:
- `Time[ms]` — cumulative time from request start in milliseconds
- `Category` — event type (`layout`, `block`, `observer`, `plugin`, `db`)
- `Operation` — the specific operation label
- `Details` — operation-specific metadata

**Reading `var/profiler/` HTML output:**

Each file in `var/profiler/` corresponds to a block and contains a self-contained HTML timeline with color-coded bars showing relative render time. Open them directly in a browser:

```bash
# View the profiler output
cat var/log/profiler.csv | head -50

# View block-level output
ls var/profiler/
```

### Adding Profiler Marks to Your Own Code

You can instrument your own classes or custom modules:

```php
use Magento\Framework\Profiler;

// Start a named timer
Profiler::start('my_custom_operation');

// ... your code here ...

// Stop the timer
Profiler::stop('my_custom_operation');
```

Nested timers are supported — the profiler handles indentation automatically in the CSV output:

```php
Profiler::start('export_products');
foreach ($productCollection as $product) {
    Profiler::start('export_product_item');
    // export one product
    Profiler::stop('export_product_item');
}
Profiler::stop('export_products');
```

The resulting CSV shows both the total `export_products` time and each nested `export_product_item` time, making it easy to identify outliers within a loop.

### The Inline Profiler Block in HTML

When `dev/debug/profiler` is enabled, the rendered HTML contains an inline block at the bottom of `<body>`:

```html
<!-- mgnlsiframesize: 0 (0%) -->
<!-- mgnlicon: 0 -->
<!-- mgnlblock: Magento\Catalog\Block\Product\ListProduct 0.234s -->
<!-- mgnlblock: Magento\Cms\Block\Block 0.012s -->
```

This shows per-block render times without needing to open separate files. In development environments with full-page caching disabled, this gives instant feedback on which blocks are slowest.

### When to Use the Built-in Profiler

| Scenario | Profiler Output |
|---|---|
| Checking which layout handles load and their order | `var/log/profiler.csv` layout rows |
| Identifying slow block rendering on a specific page | HTML comment block in page source |
| Quick check before installing Blackfire | Always available, no setup |
| Comparing two code paths in the same request | Nested `Profiler::start/stop` calls |

The built-in profiler is intentionally simpler than Blackfire — it does not show SQL query text or call counts per function. For that level of detail, use Blackfire or enable the DB logger (Section 5).

---

## 3. Blackfire.io

Blackfire is a PHP-native profiler that instruments every function call, SQL query, cache operation, and HTTP call within a request. It provides wall time, CPU time, and wait time breakdown at the function level, plus SQL query analysis and cache hit rate metrics. It is the recommended tool for diagnosing specific slow pages in development or staging.

### Installation

Blackfire requires three components:

**1. The Blackfire Agent** (background service):

```bash
# Debian/Ubuntu
wget -q -O- https://packages.blackfire.io/gpg.key | apt-key add -
echo "deb http://packages.blackfire.io/debian stable main" \
  | tee /etc/apt/sources.list.d/blackfire.list
apt-get update && apt-get install blackfire-agent
```

Configure the agent with your server credentials from the Blackfire dashboard:

```ini
# /etc/blackfire/agent
[agent]
server_id = your-server-id
server_token = your-server-token
```

**2. The Blackfire PHP Extension:**

```bash
# Install the PHP extension
php -m | grep blackfire || (
    wget -q -O blackfire-php.tgz https://blackfire.io/api/v1/releases/php/linux_amd64/php-81
    tar -xzf blackfire-php.tgz -C /tmp
    mv /tmp/blackfire-php/*.so $(php -i | grep "^extension_dir" | cut -d' ' -f3)/blackfire.so
    echo "extension=blackfire.so" > /etc/php/8.1/mods-available/blackfire.ini
    phpenmod blackfire
)
```

**3. The PHP SDK (for scenario scripting):**

```bash
cd /var/www/magento
composer require blackfire/php-sdk
```

Start the agent and restart PHP-FPM:

```bash
systemctl restart blackfire-agent
systemctl restart php8.1-fpm
```

Verify the installation:

```bash
blackfire curl https://your-magento-site.com/
```

### Writing a `.blackfire.yml` Scenario

Scenarios define the requests to profile and the assertions that must pass. Place `.blackfire.yml` in your Magento root:

```yaml
blackfire:
  client_id: your-client-id
  client_token: your-client-token

scenarios:
  # Homepage profile
  Homepage:
    path: /
    assertions:
      - main.wall_time < 2s           # Total wall clock time under 2 seconds
      - metrics.sql.queries.count < 80  # Fewer than 80 SQL queries
      - metrics.cache.hit.rate > 0.7    # Cache hit rate above 70%

  # Category page (product listing)
  Category_Lights:
    path: /lights.html
    assertions:
      - main.wall_time < 3s
      - metrics.sql.queries.count < 120
      - metrics.cache.hit.rate > 0.5
    warmup:
      - /lights.html                   # Profile the second request (cache warm)

  # Product detail page
  Product_Camera:
    path: /camera-bk01.html
    assertions:
      - main.wall_time < 1.5s
      - metrics.sql.queries.count < 60

  # Checkout flow
  Checkout:
    path: /checkout
    assertions:
      - main.wall_time < 5s
      - metrics.sql.queries.count < 150
```

### Interpreting the Timeline

The Blackfire timeline shows three time columns:

| Column | Meaning |
|---|---|
| **Wall Time** | Real clock time elapsed. High wall time with low CPU time means waiting (I/O, network, locks). |
| **CPU Time** | Actual CPU execution time. High CPU + high wall means compute-bound work. |
| **Wait Time** | Time spent waiting — database queries, HTTP calls, file I/O. |

**Reading the call graph:**

The Blackfire call graph shows parent-child function relationships. Each node displays:
- Function name and file/line
- Wall time and percentage of total
- Call count (iterations)
- SQL query count if database operations occurred

A typical slow Magento page shows:
```
main (100%)                          # Entry point
  └─ Magento\Framework\View\Result\Page::render (85%)
       └─ Magento\Catalog\Block\Product\ListProduct::_toHtml (72%)
            └─ Magento\Eav\Model\Entity\Collection::load (60%)
                 └─ PDOStatement::execute (45%)  # SQL waits show here
```

### Narrowing to Specific Requests

Use the query parameter `?fbblackfire=<profile_id>` to trigger profiling only for requests matching a specific scenario or user role. This avoids capturing profiling data from bots or cache warmup requests.

In the Blackfire companion browser extension, you can also set up "trigger on action" — e.g., only start profiling when you click the "Add to Cart" button on a PDP.

### Key Blackfire Metrics for Magento

| Metric | Description |
|---|---|
| `main.wall_time` | Total request duration |
| `metrics.sql.queries.count` | Total SELECT/INSERT/UPDATE statements |
| `metrics.sql.queries.duration` | Total time spent in database |
| `metrics.cache.hit.rate` | Redis/Varnish cache hits / total cache ops |
| `metrics.cache.hit.count` | Number of cache hits |
| `metrics.cache.miss.count` | Number of cache misses |
| `metrics.http.request.count` | External HTTP calls made during request |
| `main.memory` | Peak PHP memory usage |

### Profiling a Slow Scenario

```bash
# Profile the homepage with Blackfire CLI
blackfire run curl https://your-magento-site.com/

# Profile a specific scenario defined in .blackfire.yml
blackfire run --scenario=Homepage

# Output JSON for CI integration
blackfire run --format=json --scenario=Homepage > blackfire-report.json
```

### When to Use Blackfire vs Built-in Profiler

| Factor | Built-in Profiler | Blackfire |
|---|---|---|
| Setup effort | None | Agent + extension + SDK |
| SQL query text | No | Yes, with full query text |
| Function-level call counts | Limited | Yes, with per-function breakdown |
| Cache hit rate | No | Yes, per-request |
| Production use | Yes (low overhead) | Yes (low overhead) |
| Comparison reports | No | Yes, before/after diff |
| Continuous monitoring | No | No (ad-hoc) |

---

## 4. New Relic APM

New Relic provides continuous application performance monitoring across every request in production. Unlike Blackfire which profiles specific scenarios on demand, New Relic captures data on every single request and lets you query it retrospectively.

### Installing the New Relic PHP Agent

```bash
# Add New Relic repository and install
curl -Ls https://download.newrelic.com/php_agent/release/newrelic-repo-1.noarch.rpm \
  | rpm -Uvh
yum install newrelic-php-agent
```

### Configuring `newrelic.ini`

Edit `/etc/php.d/newrelic.ini` or the relevant PHP-FPM ini file:

```ini
[newrelic]
newrelic.appname = "Magento 2 Production"
newrelic.license_key = "your-license-key"
newrelic.enabled = 1
newrelic.loglevel = "info"

# Transaction tracing
newrelic.transaction_tracer.enabled = 1
newrelic.transaction_tracer.detail = 1
newrelic.transaction_tracer.threshold = 0.5   # Trace requests slower than 500ms
newrelic.transaction_tracer.stack_trace_threshold = 0.2

# Distributed tracing
newrelic.distributed_tracing.enabled = 1

# Error collection
newrelic.error_collector.enabled = 1
newrelic.error_collector.ignore_exceptions = "TypeError,InvalidArgumentException"
```

For Magento running behind a reverse proxy, also set:

```ini
newrelic.browser_monitoring.auto_instrument = 0   # Disable auto-injection (use manual)
newrelic.cross_application_tracer.enabled = 1
```

Restart PHP-FPM and the agent:

```bash
systemctl restart php-fpm
systemctl restart newrelic-php-agent
```

### New Relic PHP Agent for Adobe Commerce Cloud

On Adobe Commerce Cloud (magento-cloud), install via Composer:

```bash
composer require newrelic/magento2-extension
bin/magento setup:upgrade
bin/magento config:set newrelic.logging enabled 1
```

The module automatically instruments:
- HTTP requests and response times
- Database query times (via PDO instrumentation)
- Redis calls (when Redis extension is present)
- Cron job execution times
- Message queue consumer processing times

### Transaction Traces

When a request exceeds the `transaction_tracer.threshold` (default 500ms), New Relic saves a full transaction trace — a complete recording of every function call, SQL query, and external HTTP call within that request.

To view in New Relic One:
1. Go to **APM → your app → Transactions**
2. Click **Transaction traces** tab
3. Select a trace to see the waterfall breakdown

The trace shows:
- Time spent in each function (color-coded by type: PHP, SQL, HTTP, Redis)
- SQL query text for every database call
- The full call stack leading to the slowest functions

### Key Dashboards

| Dashboard | What It Measures | Target |
|---|---|---|
| **Apdex Score** | User satisfaction with response time | > 0.85 |
| **Error Rate** | Percentage of requests that return 5xx | < 1% |
| **Throughput** | Requests per minute | Measure baseline, alert on deviation |
| **Database** | Slow queries per minute, average query time | avg query < 20ms |
| **External Services** | Response time for HTTP calls to payment gateways, ERPs | < 200ms |
| **Background Jobs** | Cron job and queue consumer duration | No job > 5 minutes |

### Distributed Tracing Across Message Queue Workers

Magento's async operations (inventory updates, email sending, inventory sync) run in queue consumers — separate PHP processes that New Relic must trace across to show the full end-to-end latency.

Enable distributed tracing in `newrelic.ini`:

```ini
newrelic.distributed_tracing.enabled = 1
newrelic.span_events.enabled = 1
```

The span events link a frontend order placement to:
1. The HTTP request that created the order (span 1)
2. The message queue publish event (span 2)
3. The consumer process that handled the inventory update (span 3)

Without distributed tracing, you see only the HTTP request time and miss the async consumer delay that causes the actual problem.

### Recording Custom Business Events

New Relic can receive custom business metrics from Magento code:

```php
// Record a custom event (e.g., an order was placed)
if (function_exists('newrelic_record_custom_event')) {
    newrelic_record_custom_event('OrderPlaced', [
        'order_id'      => $order->getId(),
        'order_total'   => $order->getGrandTotal(),
        'items_count'   => $order->getTotalItemCount(),
        'store_id'      => $order->getStoreId(),
    ]);
}
```

These events appear in New Relic as custom events linked to the transaction, letting you correlate business outcomes with performance data (e.g., "orders placed during slow DB periods have 3x higher cart abandonment").

### When to Use New Relic vs Blackfire

| Factor | New Relic | Blackfire |
|---|---|---|
| Data collection | Continuous, all requests | On-demand, specific scenarios |
| Setup | Agent + config | Agent + extension + scenarios |
| SQL query details | Yes, in transaction traces | Yes, in timeline |
| Production overhead | ~1-3% CPU | ~1-2% CPU |
| Before/after comparison | Limited (can use baselines) | Yes, built-in comparison report |
| Best for | Detecting slow endpoints in production | Root-cause diagnosis of specific slow pages |

Use New Relic to detect that `/checkout` is slow across all production users. Use Blackfire to understand exactly why — which SQL query, which plugin, which block rendering.

---

## 5. Database Slow Query Analysis

Slow database queries are the single most common cause of slow Magento requests. Even when the code looks clean, an N+1 pattern or a missing index can turn a fast page into a 10-second monolith.

### Enabling the MySQL Slow Query Log

Add to your MySQL configuration (`/etc/mysql/mysql.conf.d/magento.cnf`):

```ini
[mysqld]
slow_query_log      = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time     = 0.5              # Log queries taking more than 500ms
log_queries_not_using_indexes = 1     # Also log queries doing full scans
min_examined_row_limit = 100          # Ignore trivial queries
```

Apply without restarting MySQL:

```bash
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 0.5;
SET GLOBAL log_queries_not_using_indexes = 1;
```

Verify:

```sql
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';
```

Reading the slow query log:

```bash
tail -f /var/log/mysql/slow.log
```

Each entry shows:
1. The query text
2. Execution time
3. Rows examined vs rows returned
4. Whether an index was used

### Enabling Magento's DB Logger

Magento has its own query logger that captures every SQL statement with timing, bound parameters, and call stack. Enable it:

```bash
bin/magento config:set dev/DB/log 1
```

This writes to `var/debug/db.log`. Entries look like:

```
[2026-04-24 10:15:23] main.DEBUG: SELECT
  `e`.*,
  `cat_index`.`position` AS `cat_index_position`
FROM `catalog_product_entity` AS `e`
LEFT JOIN `catalog_category_product_index_store1` AS `cat_index` ...
WHERE (e.entity_id IN (1234, 5678, ...)) [] {"is_collector":false}
```

**Important:** DB logging has significant overhead in production. Enable only for diagnostic sessions and disable immediately after:

```bash
bin/magento config:set dev/DB/log 0
```

To avoid disk space issues on long-running diagnostic sessions, set a log rotation policy or pipe through `logrotate`.

### Identifying N+1 Queries

N+1 occurs when the code loops over a collection and triggers a separate query per item:

```php
// N+1 pattern — one query for collection, N queries for each product
$products = $productCollection->load();
foreach ($products as $product) {
    // Each call to getCategory() triggers a separate query
    $category = $product->getCategory();
}
```

In `var/debug/db.log`, N+1 shows as the same SQL executed multiple times with different IDs:

```
SELECT ... FROM `catalog_product_entity` WHERE entity_id = 1234  -- 1st iteration
SELECT ... FROM `catalog_product_entity` WHERE entity_id = 5678  -- 2nd iteration
SELECT ... FROM `catalog_product_entity` WHERE entity_id = 9012  -- 3rd iteration
... (repeated 300 times)
```

The fix is to eager-load related data using `->addAttributeToSelect()` or `joinTable()`:

```php
// Fixed — single query using joined table
$productCollection
    ->addAttributeToSelect(['category_id', 'name', 'price'])
    ->addUrlRewrite()
    ->load();

// Or use Magento's grouped product collection loader
$productCollection->setPrefetchTerms(['category']);
```

### Using EXPLAIN ANALYZE

Before changing anything, verify the query plan:

```sql
EXPLAIN ANALYZE
SELECT e.*, cat_index.position
FROM catalog_product_entity e
LEFT JOIN catalog_category_product_index_store1 cat_index ...
WHERE e.entity_id IN (1234, 5678);
```

`EXPLAIN ANALYZE` (MySQL 8.0+) executes the query and returns actual timing data:

```
-> Nested loop left join  (cost=1234.50 rows=300) (actual time=0.123..1.456 rows=300 loops=1)
    -> Table scan on e  (cost=12.30 rows=300) (actual time=0.011..0.089 rows=300)
    -> Index lookup on cat_index using PRIMARY (entity_id=e.entity_id) (cost=0.25 rows=1)
```

Red flags in the plan:

| Pattern | Meaning | Fix |
|---|---|---|
| `Using filesort` | MySQL is sorting results in memory | Add `GROUP BY` column to an index |
| `Using temporary` | MySQL created a temp table | Rewrite to avoid temp table, or increase `tmp_table_size` |
| `Using where; Using index` | Index used for filtering but not covering | Create covering index (index includes all selected columns) |
| `rows=X` where X is huge | MySQL examines many rows | Add appropriate index |
| No index on foreign key | Full table scan on join | Add foreign key index |

### Fixing Slow Queries: Three Strategies

**1. Add a covering index:**

```sql
-- Before: query examines 50,000 rows
SELECT entity_id, name, price FROM catalog_product_entity WHERE category_id = 5;

-- After: create index covering all selected columns
CREATE INDEX idx_category_covered
  ON catalog_product_entity (category_id, entity_id, name, price);
```

**2. Rewrite the query to avoid subqueries:**

```php
// Bad: correlated subquery (runs once per row)
SELECT * FROM products p
WHERE price > (SELECT AVG(price) FROM products WHERE category_id = p.category_id);

// Good: single pass with window function (runs once)
SELECT p.*,
  AVG(p.price) OVER (PARTITION BY category_id) as avg_category_price
FROM products p;
```

**3. Denormalize for read-heavy paths:**

For the product listing page, pre-compute and store the `category_name` and `price` in a denormalized table or EAV attribute to avoid multi-table joins on every page load.

---

## 6. Cache Hit Rate Analysis

Cache hit rate directly determines how fast Magento responds. A cold `config` cache causes every request to re-parse XML layout files. A cold `full_page` cache causes the database to be hit on every page view.

### Cache Types in Magento 2.4

| Cache Type | Key | What It Stores | Cold Penalty |
|---|---|---|---|
| `config` | `CONFIG` | Merged XML configuration, store settings | Very high — full layout XML re-parsed |
| `layout` | `LAYOUT` | Compiled layout XML per handle | High — all blocks reinstantiated |
| `block_html` | `BLOCK_HTML` | Rendered HTML fragments | Medium — blocks re-render |
| `full_page` | `FULL_PAGE` | Entire page HTML | Very high for CMS, medium for catalog |
| `customer_data` | `CUSTOMER_SESSION` | Logged-in customer data | Low |

### Inspecting Cache Statistics

**Magento CLI:**

```bash
bin/magento cache:stats
```

Output:

```
config:
  Storage: Redis (RedisBackend)
  Hits:   14,523
  Misses:   1,847
  Hit rate: 88.7%

layout:
  Storage: Redis
  Hits:   31,204
  Misses:   892
  Hit rate: 97.2%

full_page:
  Storage: Redis
  Hits:    8,341
  Misses:   3,912
  Hit rate: 68.1%    # <-- Problem: below 80% threshold

customer_data:
  Storage: Redis
  Hits:    5,203
  Misses:     312
  Hit rate: 94.3%
```

A `full_page` hit rate below 80% typically means:
- Cache invalidation is too aggressive (plugin or observer calling `cleanByTags()`)
- Cookie/session scope mismatch (different cache for different customer groups)
- Varnish not configured and L2 cache cold after deployment

**Redis `INFO stats`:**

```bash
redis-cli INFO stats | grep keyspace
```

```
keyspace_hits:148392
keyspace_misses:23471
keyspace_hits_misses_ratio:6.32:1
```

A ratio above `5:1` (83% hit rate) is good. Below `3:1` (75%) indicates problematic cache misses.

### Cold Start Penalty

The first request after `bin/magento cache:flush` or a deployment is always slow because all caches are empty. Measure it:

```bash
# Flush everything
bin/magento cache:flush

# Time the first request (should be 5-20x slower than subsequent)
time curl -s https://your-magento-site.com/ > /dev/null

# Time the second request (cache warm)
time curl -s https://your-magento-site.com/ > /dev/null
```

Typical cold vs warm results:
- Cold: 4.2 seconds (config cache loads 2,000 XML files)
- Warm: 180ms (config served from Redis)

### Cache Warming Strategies

**Manual cache warming after deployment:**

```bash
# Warm all CMS pages and key catalog pages
bin/magento cache:warm
```

Or with a custom script for a specific set of URLs:

```php
// warm-cache.php — custom cache warmer CLI script
<?php
$urls = [
    '/',
    '/women.html',
    '/men.html',
    '/sale.html',
    '/customer/account',
];

$client = new \GuzzleHttp\Client();
foreach ($urls as $url) {
    $client->get('https://' . $_SERVER['HTTP_HOST'] . $url);
    echo "Warmed: $url\n";
}
```

**Deployment cache warming (CI/CD hook):**

```bash
# In .gitlab-ci.yml or deployment script
- curl -s https://your-magento-site.com/ > /dev/null   # warm homepage
- php bin/magento cache:warm                            # warm CMS + catalog
```

---

## 7. Reading a Blackfire Comparison Report (APE)

The Application Performance (APE) Comparison Report is Blackfire's before/after diff view. When you make a code change (index, optimization, extension removal), you profile the same scenario twice and Blackfire shows exactly what changed.

### Triggering a Comparison

1. Run the baseline profile: `blackfire run --scenario=Category_Lights`
2. Make your change (e.g., add a database index)
3. Run the comparison: `blackfire run --scenario=Category_Lights --compare-to=<baseline-id>`
4. Open the comparison link shown in the output

### Reading the Columns

| Column | Meaning |
|---|---|
| **Function** | The PHP function or method |
| **Before** | Time spent (or count) before the change |
| **After** | Time spent (or count) after the change |
| **Diff** | Absolute and percentage change |
| **Time %** | Relative time as percentage of total (only shown on the profile, not comparison) |
| **Iterations** | How many times this function was called in the request |

### Finding the Bottleneck

The bottleneck is almost always the function with the highest **Time %** value. In Magento, one of three patterns typically dominates:

**Pattern 1 — Catalog price rules recalculation:**
```
catalog_product_get_price: 890ms (45% of total)
  └─ price_indexer_recalculate: 720ms
       └─ rule_evaluator_apply: 650ms
```

Fix: Pre-calculate prices via `cron` and store in `price_index` table instead of computing on-the-fly.

**Pattern 2 — EAV attribute loading (N+1):**
```
Eav\Entity\AbstractEntity::load: 1200ms (60% of total)
  └─ (called 300 times — once per product in collection)
```

Fix: Use `addAttributeToSelect` or write a plugin that preloads all needed attributes in one query.

**Pattern 3 — Layout XML merge:**
```
Magento\Framework\View\LayoutReader::read: 800ms (40% of total)
  └─ SimpleXMLElement::merge: 750ms
```

Fix: Cache the compiled layout XML in Redis (`config` cache type) and avoid deploying with `cache:flush`.

### Comparison Report Walkthrough

```
Function                          Before    After     Diff
-----------------------------------------------------------------
Magento\Eav\Model\Entity\Collection::load  1240ms   180ms   -1060ms (-85%)
PDOStatement::execute                    1100ms   140ms   -960ms  (-87%)
catalog_product_entity table scan          980ms    30ms   -950ms  (-97%)
block_html cache write                     80ms    75ms     -5ms   (-6%)
```

In this example, the fix was adding a covering index on `catalog_product_entity(category_id, entity_id, price)` so the product collection could load without scanning the entire table. SQL time dropped from 1100ms to 140ms, reducing total request time from 1240ms to 180ms — a **7x improvement**.

---

## 8. Profiling a Real Scenario: Slow Product Listing Page

This section walks through the complete profile → hypothesize → change → re-profile cycle on a real Magento 2.4 store with a slow category page (`/lights.html`).

### Step 0 — Establish Baseline

Before changing anything, establish a baseline with Blackfire:

```yaml
# .blackfire.yml
scenarios:
  Category_Lights:
    path: /lights.html
    assertions:
      - main.wall_time < 5s
      - metrics.sql.queries.count < 200
    warmup:
      - /lights.html
```

Run the baseline:

```bash
blackfire run --scenario=Category_Lights --json > baseline.json
```

Read the key numbers:
- Wall time: **4.2 seconds** (target: < 3s)
- SQL queries: **187** (target: < 120)
- Cache hit rate: **41%** (target: > 70%)

### Step 1 — Enable DB Logger and Capture the Profile

```bash
bin/magento config:set dev/DB/log 1
curl -s https://your-magento-site.com/lights.html > /dev/null
```

Inspect the query log:

```bash
tail -100 var/debug/db.log | grep "SELECT" | sort | uniq -c | sort -rn | head -20
```

Output shows:

```
  312  SELECT `e`.* FROM `catalog_product_entity` `e` WHERE (entity_id = 1234)
   89  SELECT `e`.* FROM `catalog_product_entity` `e` WHERE (entity_id = 5678)
    4  SELECT `e`.* FROM `catalog_product_entity` `e` WHERE (entity_id = 9012)
```

The N+1 is clear — 405 identical queries with different `entity_id` values in a single request.

### Step 2 — Hypothesis

The product listing page calls `getCategory()` on each product in the collection, triggering a separate EAV load per product. The product collection does not eager-load the `category_id` attribute.

### Step 3 — Fix the N+1 Query

Find the block that renders the category association:

```php
// In Magento\Catalog\Block\Product\ListProduct or a custom theme file
// Change from implicit lazy loading to explicit eager loading:

$productCollection
    ->addAttributeToSelect('category_id')           // Eager load category FK
    ->addAttributeToSelect('name')
    ->addAttributeToSelect('price')
    ->addUrlRewrite();                              // Pre-populate URL rewrites
```

This changes the SQL from 405 individual queries to 1 query with a joined subselect.

### Step 4 — Re-profile

```bash
bin/magento config:set dev/DB/log 0   # Disable logger — back to normal

blackfire run --scenario=Category_Lights --json > after-fix.json
```

New numbers:
- Wall time: **1.4 seconds** (was 4.2s — **3x improvement**)
- SQL queries: **62** (was 187 — **3x reduction**)
- Cache hit rate: **68%** (was 41%)

### Step 5 — Compare with Blackfire APE

```bash
blackfire compare baseline.json after-fix.json
```

Output confirms the fix:

```
Before                           After          Diff
product_collection_load: 4120ms  420ms    -3700ms (-90%)
PDO queries (count):       405     62       -343 (-85%)
```

The remaining 62 queries are expected: core CMS operations, customer session, and catalog price rules. The 420ms remaining in `product_collection_load` is genuine business logic, not an N+1 issue.

### Full Cycle Summary

| Phase | Action | Result |
|---|---|---|
| Baseline | Blackfire scenario | 4.2s, 187 queries, 41% cache hit |
| Diagnose | `dev/DB/log` + db.log | Identified N+1: 405 identical queries |
| Fix | `addAttributeToSelect('category_id')` | Reduced queries to 62 |
| Re-profile | Blackfire run | 1.4s, 62 queries, 68% cache hit |
| Compare | APE report | 3x wall time improvement, 3x fewer queries |

Repeat the cycle for the next bottleneck (likely `full_page` cache hit rate next, then EAV attribute loading).

---

## Key Commands Quick Reference

```bash
# Enable built-in profiler
bin/magento config:set dev/debug/profiler 1

# Enable Magento DB logger
bin/magento config:set dev/DB/log 1

# View cache stats
bin/magento cache:stats

# View Magento DB log
tail -f var/debug/db.log

# Blackfire CLI profiling
blackfire run curl https://your-magento-site.com/
blackfire run --scenario=Homepage
blackfire compare baseline.json after-fix.json

# MySQL slow query log
SHOW VARIABLES LIKE 'slow_query_log%';
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 0.5;

# Redis cache info
redis-cli INFO stats | grep keyspace

# New Relic custom event (PHP code)
if (function_exists('newrelic_record_custom_event')) {
    newrelic_record_custom_event('OrderPlaced', ['order_id' => $id]);
}

# Cache warming
bin/magento cache:warm
```

---

## See Also

- **Redis/Varnish caching**: `_supplemental/06-caching.md`
- **Database optimization**: `_supplemental/04-declarative-schema.md`
- **Blackfire.io documentation**: https://blackfire.io/docs
- **New Relic PHP agent**: https://docs.newrelic.com/docs/apm/agents/php-agent
- **MySQL EXPLAIN ANALYZE**: https://dev.mysql.com/doc/refman/8.0/en/explain.html