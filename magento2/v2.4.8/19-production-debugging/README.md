# Topic 19: Production Debugging for Magento 2 Backend

**Goal:** Master the production debugging workflow — request tracing, log correlation, query logging, API timing analysis, and common production error resolution. All topics prior to this cover profiling tools; this topic covers the actual workflow patterns you need when production is on fire.

> **Audit finding:** Topics 09 and 14 cover profiling tools (Tideways/XHProf, Blackfire, New Relic) but ALL production debugging workflow patterns are absent. This topic fills that gap.

---

## Topic Header Mapping

| Header Field | Value |
|---|---|
| **Topic** | 21 — Production Debugging |
| **Version** | Magento 2.4.x (all modes) |
| **Prerequisites** | Topics 09 (Performance) and 14 (Advanced Performance) |
| **Approximate Length** | ~1400 lines |
| **Last Updated** | 2025-04-21 |

---

## Prerequisites

- Magento 2.4.x installed with `env.php` write access
- `bin/magento` CLI competence (cache, indexer, config commands)
- SSH/server access to production host
- MySQL CLI access for query logging and `SHOW ENGINE INNODB STATUS`
- Redis CLI access for `redis-cli` diagnostics
- New Relic or equivalent APM (optional but referenced in Section 4)

---

## Learning Objectives

1. Propagate `X-Request-ID` through a Magento request lifecycle and correlate it across `debug.log`, MySQL slow query log, and HTTP response headers.
2. Enable and interpret `EnableQUERY_LOGGING` output, identify blacklisted queries, and filter the query log for slow operations.
3. Trace a slow REST or GraphQL API endpoint to its root cause using New Relic transactions, X-Magento-TAGS headers, and controller/service timing.
4. Diagnose and resolve the five most common production errors: DI cycle dependencies, memory exhaustion, MySQL lock wait timeout, Redis connection failures, and category flat table reindex failures.
5. Compare Developer, Production, and Default modes with respect to which debugging features are available in each, and safely enable developer mode on a subdirectory for targeted debugging.

---

## By End of Topic You Must Prove

- A request to the storefront produces a unique `X-Request-ID` header that appears in `debug.log` entries for that same request.
- `EnableQUERY_LOGGING` is active in `env.php` and queries from a specific controller action are captured in the query log file.
- A slow GraphQL endpoint is traced via New Relic to a specific service class whose method calls `join()` on an unbounded collection.
- A DI cycle error is resolved by restructuring a plugin preference or extracting shared logic into a joint interface.
- A Redis `connection refused` error is resolved by verifying the Redis bind address in `env.php` and checking `netstat`.
- `MAGE_MODE` is verified as `production` on the live environment, and developer mode is safely enabled on a staging subdirectory without affecting production.

---

## Assessment Criteria

| Criterion | Evidence |
|---|---|
| X-Request-ID header traced | `curl -I` showing `X-Request-ID` header + `grep` of `debug.log` showing same ID |
| Query logging active | `env.php` setting + query log file with controller-scoped entries |
| API timing traced via New Relic | Transaction breakdown showing slow service method + source file reference |
| DI cycle resolved | Before/after di.xml preference showing joint interface extraction |
| Redis error resolved | `redis-cli ping` returns `PONG` after `env.php` Redis bind fix |
| MAGE_MODE verified | `bin/magento deploy:mode:show` on production + developer mode on staging subdirectory |

---

## Reference Exercises

- **Exercise 21.1:** Add `X-Request-ID` propagation to a frontend controller and verify it appears in `var/log/debug.log`.
- **Exercise 21.2:** Enable `EnableQUERY_LOGGING` in `env.php`, hit a slow product page, and identify the top 3 slowest queries via `grep` + `EXPLAIN`.
- **Exercise 21.3:** Instrument a custom GraphQL resolver with New Relic custom events and identify a slow resolver method.
- **Exercise 21.4:** Simulate a DI cycle between two plugins, identify the cycle via the error message, and resolve it using a joint interface.
- **Exercise 21.5:** Trigger a MySQL lock wait timeout by running two concurrent reindex jobs, identify the lock via `SHOW ENGINE INNODB STATUS`, and resolve by setting indexer batch size.
- **Exercise 21.6:** Configure Redis session storage in `env.php`, trigger a connection refused error, and resolve it with proper bind configuration.
- **Exercise 21.7 (bonus):** Enable developer mode on a `/staging` subdirectory URL while keeping production mode on the main domain.

---

## Topics

---

### Section 1: End-to-End Request Tracing

Production traffic is messy. A single storefront page load generates dozens of cache operations, database queries, observer dispatches, and HTTP calls to downstream services. When something is slow — or worse, wrong — you need to follow ONE request through the entire stack. Request tracing gives you that thread.

#### The Three Headers Magento Uses for Request Tracing

Magento 2 emits three HTTP headers that, together, give you complete request visibility:

| Header | What It Contains | When It's Set |
|---|---|---|
| `X-Magento-TAGS` | Cache tags invalidated or present on the response (e.g., `FPC_PRODUCT_1,CATEGORY_5`) | On every full-page cache response |
| `X-Cache-Tags` | Same as `X-Magento-TAGS` but sent to Varnish (if Varnish is in front) | Only when Varnish is active |
| `X-Request-ID` | Unique per-request UUID generated early in the request lifecycle | On every request (must be explicitly added — see below) |

`X-Magento-TAGS` and `X-Cache-Tags` are set automatically by Magento's Http application layer. `X-Request-ID` is NOT set by default — you must add it.

#### Adding X-Request-ID to Every Request

Create a small plugin on `Magento\Framework\App\Http\Context` or `Magento\Framework\App\Request\Http` to inject the header. The cleanest approach is a `di.xml` plugin on `\Magento\Framework\App\Request\Http`:

```php
// app/code/YourVendor/DebugTracing/etc/di.xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Framework\App\Request\Http">
        <plugin name="YourVendor_DebugTracing_RequestPlugin" type="YourVendor\DebugTracing\Plugin\RequestPlugin"/>
    </type>
</config>
```

```php
// app/code/YourVendor/DebugTracing/Plugin/RequestPlugin.php
declare(strict_types=1);

namespace YourVendor\DebugTracing\Plugin;

use Magento\Framework\App\Request\Http as Subject;
use Magento\Framework\App\RequestInterface;
use Magento\Framework\Stdlib\DateTime\TimezoneInterface;

class RequestPlugin
{
    private const REQUEST_ID_HEADER = 'X-Request-ID';
    private const REQUEST_ID_LOG_KEY = 'request_id';

    public function beforeDispatch(Subject $subject, RequestInterface $request): void
    {
        $requestId = $subject->getHeader(self::REQUEST_ID_HEADER);
        if (!$requestId) {
            $requestId = $this->generateRequestId();
        }
        // Store in a request attribute for later log injection
        $subject->getAttributes()->set(self::REQUEST_ID_LOG_KEY, $requestId);
    }

    public function afterDispatch(Subject $subject, $result): void
    {
        $requestId = $subject->getAttribute(self::REQUEST_ID_LOG_KEY);
        if ($requestId) {
            $subject->getHeaders()->addNativeHeader(self::REQUEST_ID_HEADER, $requestId);
        }
    }

    private function generateRequestId(): string
    {
        return sprintf(
            '%04x%04x-%04x-%04x-%04x-%04x%04x%04x',
            mt_rand(0, 0xffff), mt_rand(0, 0xffff),
            mt_rand(0, 0xffff), mt_rand(0, 0x0fff) | 0x4000,
            mt_rand(0, 0x3fff) | 0x8000,
            mt_rand(0, 0xffff), mt_rand(0, 0xffff), mt_rand(0, 0xffff)
        );
    }
}
```

> **Note:** In production, avoid generating UUIDs on every request unless you need them. For debugging, this plugin is activated temporarily and then removed.

#### Verifying X-Request-ID on a Request

```bash
# Hit any storefront page and inspect response headers
curl -sI https://yourstore.example.com/ | grep -i request

# Expected output (example):
# X-Request-ID: 7f3a2c1b-9d4e-4f8a-b5c6-d7e8f9a0b1c2
```

If the header is absent, check that your plugin is registered in `app/etc/di.xml` and the module is enabled:

```bash
bin/magento module:status YourVendor_DebugTracing
bin/magento module:enable YourVendor_DebugTracing
bin/magento cache:clean
```

---

### Section 2: Log Correlation — Request ID to debug.log

`debug.log` grows rapidly in any moderately trafficked environment. Correlating log entries to a specific HTTP request is the single most effective debugging technique for production issues.

#### Injecting Request ID into All Log Entries

Magento's logger uses `Magento\Framework\Logger\LoggerAware` trait. Create a custom log handler that prepends the request ID to every log entry:

```php
// app/code/YourVendor/DebugTracing/Logger/Handler/Debug.php
declare(strict_types=1);

namespace YourVendor\DebugTracing\Logger\Handler;

use Magento\Framework\Logger\Handler\Debug as BaseDebugHandler;
use Monolog\Formatter\LineFormatter;
use Monolog\LogRecord;

class Debug extends BaseDebugHandler
{
    protected function prepareFormatter(): LineFormatter
    {
        $formatter = new LineFormatter(
            '[%extra.request_id%] %message% %context% %extra%' . PHP_EOL,
            'Y-m-d H:i:s.u'
        );
        $this->setFormatter($formatter);
        return $formatter;
    }

    public function handle(LogRecord $record): bool
    {
        $requestId = $this->getCurrentRequestId();
        if ($requestId) {
            $record->extra['request_id'] = $requestId;
        }
        return parent::handle($record);
    }

    private function getCurrentRequestId(): string
    {
        // Request ID is stored as a request attribute by the RequestPlugin
        $objectManager = \Magento\Framework\App\ObjectManager::getInstance();
        try {
            $request = $objectManager->get(\Magento\Framework\App\RequestInterface::class);
            return (string) $request->getAttribute('request_id') ?: 'no-request-id';
        } catch (\Throwable) {
            return 'no-request-id';
        }
    }
}
```

Register it in `di.xml`:

```xml
<!-- app/code/YourVendor/DebugTracing/etc/di.xml -->
<type name="Magento\Framework\Logger\Handler\Debug">
    <plugin name="YourVendor_DebugTracing_DebugHandler" type="YourVendor\DebugTracing\Logger\Handler\Debug"/>
</type>
```

> **Performance warning:** Using `ObjectManager::getInstance()` inside a log handler is expensive. In production, this handler should be only temporarily enabled for active debugging sessions. For a production-safe version, inject the request ID via a `Processor` monolog processor instead.

#### Filtering debug.log by Request ID

With the request ID injected into the `extra` context, filtering is straightforward:

```bash
# Extract all log entries for a specific request ID
REQUEST_ID="7f3a2c1b-9d4e-4f8a-b5c6-d7e8f9a0b1c2"
grep "$REQUEST_ID" var/log/debug.log

# Pretty-print with context lines (5 lines before and after each match)
grep -C 5 "$REQUEST_ID" var/log/debug.log

# Count how many log entries this request generated
grep -c "$REQUEST_ID" var/log/debug.log

# Extract just the log messages (strip the request ID prefix)
grep "$REQUEST_ID" var/log/debug.log | sed 's/.*request_id%\] //'
```

#### Grep Patterns for Common Log Searches

```bash
# All ERROR level entries across all log files
grep -rni "ERROR" var/log/

# Entries from a specific class (e.g., a plugin or observer)
grep -rni "YourVendor\\Module\\Plugin\\BeforeSave" var/log/debug.log

# Entries containing SQL queries (noting the actual query text)
grep -rni "SELECT.*FROM.*catalog_product_entity" var/log/debug.log

# Entries with exceptions in the trace
grep -rni "Exception\|Trace" var/log/exception.log | tail -50

# Entries within a time window (last hour)
awk '$1 >= "'$(date -d '1 hour ago' '+%Y-%m-%d %H:%M')'"' var/log/debug.log

# All entries referencing a specific product ID
grep -i "product.*145\|145.*product" var/log/debug.log

# Entries from a specific IP address (brute-force detection)
grep -i "192.168.1.100" var/log/debug.log
grep -i "unauthorized\|login.*failed" var/log/exception.log
```

#### Creating a Correlation Script

For active debugging sessions, create a small bash script to correlate logs by request ID:

```bash
#!/bin/bash
# scripts/correlation.sh — correlate logs by X-Request-ID
set -euo pipefail

REQUEST_ID="${1:-}"
if [[ -z "$REQUEST_ID" ]]; then
    echo "Usage: $0 <X-Request-ID>"
    exit 1
fi

echo "=== debug.log for request $REQUEST_ID ==="
grep -C 3 "$REQUEST_ID" var/log/debug.log || echo "(no debug.log entries)"

echo ""
echo "=== system.log for request $REQUEST_ID ==="
grep -C 3 "$REQUEST_ID" var/log/system.log || echo "(no system.log entries)"

echo ""
echo "=== exception.log (recent) ==="
tail -100 var/log/exception.log | grep -C 2 "$REQUEST_ID" || echo "(no exception entries)"

echo ""
echo "=== query log (recent) ==="
QUERY_LOG="${QUERY_LOG_FILE:-var/log/debug/db_repo.log}"
if [[ -f "$QUERY_LOG" ]]; then
    grep -C 1 "$REQUEST_ID" "$QUERY_LOG" | head -50 || echo "(no query log entries)"
else
    echo "(query log file not found: $QUERY_LOG)"
fi
```

---

### Section 3: Query Logging — EnableQUERY_LOGGING Deep Dive

SQL query logging is one of the most powerful tools for identifying database performance problems. Magento 2 provides `EnableQUERY_LOGGING` which writes all database queries to a dedicated log file without enabling the MySQL general log (which is too expensive for production).

#### Enabling EnableQUERY_LOGGING

Add to `env.php` under the `db` connection array:

```php
// app/etc/env.php
'db' => [
    'connection' => [
        'default' => [
            'host' => 'db',
            'dbname' => 'magento',
            'username' => 'magento',
            'password' => 'magento',
            'model' => 'mysql8',
            'engine' => 'innodb',
            'initStatements' => 'SET NAMES utf8mb4',
            'driver_options' => [
                PDO::MYSQL_ATTR_COMPRESS => true,
                PDO::MYSQL_ATTR_LOCAL_INFILE => false,
            ],
        ],
    ],
    // Enable query logging for the default connection
    'enable_query_logging' => true,
],
```

After adding this, queries are written to `var/log/debug/db_repo.log` (the default path). You can override the path:

```php
'enable_query_logging' => true,
'query_log_file' => 'var/log/debug/custom_query_log.log',
```

#### Fine-Grained Control with logQueryBlacklist

Not all queries should be logged. Magento's blacklist prevents logging of high-frequency, noisy queries that would bloat the log file:

```php
// app/etc/env.php - full query logging configuration
'db' => [
    'connection' => [
        'default' => [
            // ... connection settings
        ],
    ],
    'enable_query_logging' => true,
    'query_log_file' => 'var/log/debug/db_repo.log',

    // Array of regex patterns for queries to exclude from logging
    'log_query_blacklist' => [
        // Don't log Magento's cache metadata queries (very high frequency)
        '/SELECT.*FROM.*cms_page.*WHERE/i',
        '/SELECT.*FROM.*cms_block.*WHERE/i',
        // Don't log config cache lookups
        '/SELECT.*FROM.*cache.*WHERE/i',
        // Don't log session-related queries
        '/SELECT.*FROM.*session.*WHERE/i',
    ],
],
```

> **Practical tip:** Start with an empty blacklist, let it run for 5 minutes on a test environment, then identify your top-10 most frequent query patterns with `sort | uniq -c | sort -rn | head -20` on the raw log file. Add only those to the blacklist.

#### Analyzing the Query Log

```bash
# Count queries by type
grep -oE "^(SELECT|INSERT|UPDATE|DELETE)" var/log/debug/db_repo.log | sort | uniq -c | sort -rn

# Find the top 10 slowest queries by execution time (log format: "...took Nms")
grep -oE "took [0-9]+ms" var/log/debug/db_repo.log | sort -t ' ' -k2 -rn | head -10

# Find all queries taking > 100ms
awk '/took [0-9]+ms/ { if ($NF+0 > 100) print }' var/log/debug/db_repo.log | head -20

# Find queries with JOINs on large tables (potential N+1)
grep -i "JOIN.*catalog_product_entity" var/log/debug/db_repo.log

# Find queries missing WHERE clauses (full table scans)
grep -vE "WHERE|LIMIT [0-9]+" var/log/debug/db_repo.log | grep -E "SELECT.*FROM" | head -10

# Extract just the SQL text from a specific request ID time window
awk '/2025-04-21 14:3[0-9]/ { print }' var/log/debug/db_repo.log | grep "SELECT" | head -30

# Identify duplicate queries (same SQL executed multiple times in one request)
awk '/SELECT.*FROM/' var/log/debug/db_repo.log | cut -d'[' -f2 | cut -d']' -f1 | sort | uniq -c | sort -rn | head -15
```

#### Real Query Log Excerpt (Magento 2.4.7)

A typical entry in `var/log/debug/db_repo.log` looks like this:

```
[2025-04-21 14:32:15.123456] MainContext: SELECT
    `e`.*, `cat_index`.`position` AS `cat_index_position`
    FROM `catalog_product_entity` AS `e`
    INNER JOIN `catalog_category_product_index_store1` AS `cat_index`
        ON cat_index.product_id = e.entity_id
        AND cat_index.store_id = 1
        AND cat_index.category_id = '5'
    WHERE (`e`.`entity_id` IN (145, 146, 147))
    ORDER BY `cat_index`.`position` ASC
[took 2ms]
[2025-04-21 14:32:15.234567] MainContext: SELECT
    `e`.*, `stock_status_index`.`stock_status` AS `is_salable`
    FROM `catalog_product_entity` AS `e`
    LEFT JOIN `cataloginventory_stock_status_idx` AS `stock_status_index`
        ON e.entity_id = stock_status_index.product_id
    WHERE (e.entity_id IN (145))
[took 1ms]
[2025-04-21 14:32:16.567890] MainContext: UPDATE
    `catalog_product_entity`
    SET `updated_at` = '2025-04-21 14:32:16'
    WHERE (entity_id = 145)
[took 45ms]
```

The `[took Nms]` suffix makes it easy to find slow queries. Anything > 50ms on a production site with flash traffic is a candidate for optimization.

#### EXPLAIN Analysis Workflow

For any query taking > 100ms, run `EXPLAIN` in MySQL to identify missing indexes:

```bash
# Run EXPLAIN on the suspect query (replicate the exact params from the log)
mysql -u magento -p -D magento -e "
EXPLAIN SELECT
    e.*, cat_index.position AS cat_index_position
    FROM catalog_product_entity AS e
    INNER JOIN catalog_category_product_index_store1 AS cat_index
        ON cat_index.product_id = e.entity_id
        AND cat_index.store_id = 1
        AND cat_index.category_id = '5'
    WHERE (e.entity_id IN (145, 146, 147))
    ORDER BY cat_index.position ASC;
"

# Look for:
# - type=ALL (full table scan) — bad
# - rows > 1000 (examining too many rows) — bad
# - Using filesort / Using temporary — bad
# - key is NULL (no index used) — bad
```

Add indexes via `db_schema.xml`:

```xml
<!-- app/code/YourVendor/YourModule/etc/db_schema.xml -->
<table name="catalog_category_product_index_store1" resource="default">
    <index referenceId="CAT_IDX_STORE1_CAT_PROD">
        <field name="category_id" xsi:type="integer"/>
        <field name="product_id" xsi:type="integer"/>
        <field name="store_id" xsi:type="smallint"/>
        <field name="position" xsi:type="integer"/>
        <indexes>
            <primary>
                <field name="category_id"/>
                <field name="product_id"/>
                <field name="store_id"/>
            </primary>
        </indexes>
    </index>
</table>
```

Then run `bin/magento setup:db-declaration:generate-whitelist` and `bin/magento setup:upgrade`.

---

### Section 4: API Debugging — REST/GraphQL Timing and New Relic

Slow API endpoints are the most visible production problem because they directly affect frontend performance and mobile apps. This section covers the workflow for finding and fixing slow REST and GraphQL endpoints.

#### Timing REST Endpoints

Magento's REST API has built-in timing logging when `enable_query_logging` is active. For targeted profiling, add timing instrumentation:

```php
// In your REST controller or service contract
use Psr\Log\LoggerInterface;

public function getProduct(int $id): \Magento\Catalog\Api\Data\ProductInterface
{
    $start = microtime(true);

    $product = $this->productRepository->get($id);

    $elapsed = (microtime(true) - $start) * 1000; // ms
    $this->logger->info(sprintf(
        'ProductRepository::get(%d) took %.2fms',
        $id,
        $elapsed
    ));

    return $product;
}
```

For GraphQL, the approach is similar but using the resolver layer:

```php
// In a GraphQL resolver (e.g., ProductResolver.php)
declare(strict_types=1);

namespace YourVendor\YourModule\Model\Resolver;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Config\GraphQlAttributeFilter;
use Psr\Log\LoggerInterface;

class ProductResolver implements ResolverInterface
{
    private ProductRepositoryInterface $productRepository;
    private LoggerInterface $logger;

    public function __construct(
        ProductRepositoryInterface $productRepository,
        LoggerInterface $logger
    ) {
        $this->productRepository = $productRepository;
        $this->logger = $logger;
    }

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $resolveInfo,
        array $value = null,
        array $args = null
    ): array {
        $start = microtime(true);

        $product = $this->productRepository->get($args['id']);

        $elapsed = (microtime(true) - $start) * 1000;
        $this->logger->info(sprintf(
            'GraphQL ProductResolver[id=%d] took %.2fms',
            $args['id'],
            $elapsed
        ));

        return $this->extractProductData($product);
    }
}
```

#### Using New Relic for API Transaction Tracing

New Relic automatically instruments Magento's REST and GraphQL controllers. The key is reading the transaction breakdown:

1. **Find the slow transaction** in New Relic APM → Transactions → sort by Duration descending.
2. **Open the transaction trace** for the slow endpoint.
3. **Read the call stack** — New Relic shows each method with its exclusive time.
4. **Look for the hot spot** — a single method call taking > 80% of total time.

Common slow patterns in Magento APIs:

| Slow Pattern | Root Cause | Fix |
|---|---|---|
| `ProductRepository::get()` taking > 500ms | Missing index on `catalog_product_entity` | Add index via `db_schema.xml` |
| `CategoryRepository::get()` calling flat table rebuild | Category flat table disabled or stale index | `bin/magento indexer:reindex catalog_category_flat` |
| `GraphQL resolver` with N+1 on collection | Calling `getItems()` in a loop | Use `searchCriteria` with filter for batched fetch |
| `CartRepository::get()` loading full quote | Quote loading customer addresses redundantly | Use `QuoteRepository::getForCustomer()` with active quote only |
| `OrderRepository::get()` loading all items | Loading order items when not needed | Pass `items[]` in `searchCriteria` only when needed |

#### Finding the Slow Controller

If New Relic shows a slow transaction but doesn't pinpoint the exact method:

```bash
# Enable Magento's built-in profiler for the slow endpoint
# 1. Enable profiler in env.php
bin/magento dev:profiler:enable

# 2. Hit the slow endpoint with a specific request ID
curl -s -o /dev/null -w "Time: %{time_total}s\n" \
     -H "X-Request-ID: manual-test-001" \
     https://yourstore.example.com/rest/V1/products/145

# 3. Check profiler output
cat var/log/profiler.log | grep "manual-test-001"

# 4. Disable when done (profiler adds overhead)
bin/magento dev:profiler:disable
```

#### GraphQL Query Timing with Built-In Magento Logging

For GraphQL, Magento's GraphQL module already logs slow queries when query complexity exceeds thresholds. Check `exception.log` and `debug.log` for GraphQL-specific entries:

```bash
# Search for GraphQL timing entries
grep -i "graphql\|resolve\|query" var/log/debug.log | grep "took" | sort -t ']' -k2 -rn | head -20

# Search for GraphQL errors
grep -i "graphql" var/log/exception.log | tail -30
```

---

### Section 5: Common Production Errors and Debug Mode

This section covers the five production errors most likely to wake you up at 2 AM, how to diagnose them from their error messages, and how to fix them.

#### 5.1: DI Cycle Detection and Resolution

**What it looks like:**

```
Circular dependency: YourVendor\Module\Plugin\A depends on YourVendor\Module\Plugin\B
and vice versa.
#0 /var/www/html/vendor/yourvendor/module-plugin-a/etc/di.xml:... (line reference)
```

**Diagnosis:**

DI cycles occur when two classes depend on each other through their constructor. Magento's `ObjectManager` catches this at compile time (in developer mode) or at runtime (in production mode).

Check both `di.xml` files of the two modules involved:

```xml
<!-- app/code/YourVendor/ModulePluginA/etc/di.xml -->
<type name="YourVendor\Module\Plugin\A">
    <plugin name="B" type="YourVendor\Module\Plugin\B"/>
</type>

<!-- app/code/YourVendor/ModulePluginB/etc/di.xml -->
<type name="YourVendor\Module\Plugin\B">
    <plugin name="A" type="YourVendor\Module\Plugin\A"/>
</type>
```

**Fix — Joint Interface Pattern:**

Extract the shared behavior into a joint interface that neither plugin directly depends on:

```php
// app/code/YourVendor/Module/Api/SharedBehaviorInterface.php
declare(strict_types=1);

namespace YourVendor\Module\Api;

interface SharedBehaviorInterface
{
    public function executeBeforeSave(string $value): string;
}
```

Then refactor both plugins to depend only on the interface:

```php
// Plugin A — depends on interface, not on Plugin B directly
class PluginA
{
    public function __construct(
        private readonly SharedBehaviorInterface $sharedBehavior
    ) {}

    public function beforeSave($subject, $value): void
    {
        $this->sharedBehavior->executeBeforeSave($value);
    }
}
```

#### 5.2: Memory Limit Exhausted

**What it looks like:**

```
Fatal error: Allowed memory size of 536870912 bytes exhausted
(tried to allocate 20480 bytes) in
/var/www/html/vendor/magento/framework/DB/Adapter/Pdo/Mysql.php on line 1234
```

**Diagnosis:**

```bash
# Check current memory limit
php -r "echo ini_get('memory_limit');"

# Check if the process is hitting the limit during a specific operation
bin/magento indexer:reindex --dry-run 2>&1 | grep -i "memory\|bytes\|exhaust"

# Check PHP-FPM memory usage per worker
ps aux | grep php-fpm | awk '{print $6}' | sort -n | tail -5
```

**Fix:**

```bash
# Option 1: Temporarily raise memory limit for a specific CLI command
php -d memory_limit=2G bin/magento indexer:reindex

# Option 2: Set batch size to reduce memory per operation
bin/magento config:set catalog/frontend/grid_batch_size 100
bin/magento config:set catalog/frontend/flat_catalog_product 0  # Disable flat for categories

# Option 3: Increase global memory limit in .user.ini or php.ini
# memory_limit = 512M  ; raise to 1G or 2G for heavy operations
```

**Long-term fix:** Identify the collection that is loading unbounded data. In `searchCriteria`, always set `pageSize`:

```php
$searchCriteria = $this->searchCriteriaBuilder
    ->addFilter('category_id', $categoryId)
    ->setPageSize(100)       // Critical — prevents unbounded collection
    ->setCurrentPage(1)
    ->create();
```

#### 5.3: MySQL Lock Wait Timeout

**What it looks like:**

```
SQLSTATE[HY000]: General error: 1205 Lock wait timeout exceeded;
try restarting transaction
```

**Diagnosis:**

```sql
-- Find which transaction is holding the lock
SHOW ENGINE INNODB STATUS\G

-- In the TRANSACTION section, look for:
-- "mysql tables in use: 1, locked: 1"
-- "LOCK WAIT ..."

-- Find blocking queries
SELECT
    trx_id,
    trx_mysql_thread_id,
    trx_started,
    trx_rows_modified,
    trx_query
FROM information_schema.INNODB_TRX
WHERE trx_state = 'RUNNING';

-- Kill the blocking thread (only in emergency)
-- KILL <thread_id>;
```

**Fix:**

```bash
# Option 1: Reduce indexer batch size to prevent long transactions
bin/magento config:set indexer/match_algo batch_size 100

# Option 2: Switch to schedule mode to avoid concurrent full reindex
bin/magento indexer:set-mode schedule
bin/magento indexer:reindex   # Only runs one at a time

# Option 3: Increase lock wait timeout in MySQL (my.cnf)
# innodb_lock_wait_timeout = 120  (default is 50)
```

#### 5.4: Redis Connection Refused

**What it looks like:**

```
Redis connection refused: 127.0.0.1:6379
#0 /var/www/html/vendor/magento/framework/Cache/Core.php on line 51
```

**Diagnosis:**

```bash
# Test Redis connectivity from the app server
redis-cli ping
# Expected: PONG
# Got: Connection refused = Redis not running or wrong bind address

# Check Redis is running
systemctl status redis  # systemd
# or
ps aux | grep redis | grep -v grep

# Check what bind address Redis is listening on
redis-cli CONFIG GET bind

# Check netstat on the Redis port
netstat -tlnp | grep 6379
# Expected: 0.0.0.0:6379 or 127.0.0.1:6379

# Check Redis auth (if configured)
redis-cli AUTH yourpassword
```

**Fix — Correct env.php Redis Configuration:**

```php
// app/etc/env.php
'cache' => [
    'frontend' => [
        'default' => [
            'backend' => 'Magento\Framework\Cache\Backend\Redis',
            'backend_options' => [
                'server' => 'redis',
                'port' => 6379,
                'password' => '',  // Set if Redis requires AUTH
                'database' => 0,     // 0 = default cache, 1 = page cache, 2 = session
                'timeout' => 2.5,
                'retry_interval' => 100,
                'persistent' => '',
                'forcestandalone' => false,  // false = use PHP redis extension (faster)
            ],
        ],
        'page_cache' => [
            'backend' => 'Magento\Framework\Cache\Backend\Redis',
            'backend_options' => [
                'server' => 'redis',
                'port' => 6379,
                'database' => 1,
                'timeout' => 2.5,
            ],
        ],
    ],
    'id_prefix' => 'xyz_',
],
'session' => [
    'save' => 'redis',
    'redis' => [
        'host' => 'redis',
        'port' => 6379,
        'database' => 2,    // Separate DB for sessions
        'timeout' => 2.5,
        'ttl' => 3600,
        'persistent' => '',
    ],
],
```

> **Critical:** The `server` value in `env.php` must match the hostname of the Redis container/service, not `127.0.0.1` if Redis is in a Docker container. Use the Docker service name (`redis`) or the correct IP.

#### MySQL Lock Wait Timeout — Extended Diagnostic Workflow

Beyond `SHOW ENGINE INNODB STATUS`, use these queries to identify lock contention patterns:

```sql
-- Find all currently running transactions with their locks
SELECT
    trx_id,
    trx_mysql_thread_id AS thread_id,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS trx_age_seconds,
    trx_rows_modified,
    trx_query
FROM information_schema.INNODB_TRX
WHERE trx_state = 'RUNNING'
ORDER BY trx_started;

-- Find which locks are held by which transactions
SELECT
    pl.lock_id,
    pl.lock_mode,
    pl.lock_type,
    pl.lock_table,
    pl.lock_index,
    pl.lock_data,
    pt.requesting_thread_id,
    pt.blocking_thread_id,
    pt.blocking_lock_id
FROM information_schema.INNODB_LOCKS pl
JOIN information_schema.INNODB_LOCK_WAITS pt
    ON pl.lock_id = pt.blocking_lock_id
WHERE pt.blocking_lock_id IS NOT NULL;

-- Processlist view (MySQL 8.0+)
SELECT
    p.id AS thread_id,
    p.user,
    p.host,
    p.db,
    p.command,
    p.time,
    p.state,
    LEFT(p.info, 80) AS current_query
FROM information_schema.PROCESSLIST p
WHERE p.command != 'Sleep'
ORDER BY p.time DESC;
```

**Reindex-specific batch size configuration:**

```bash
# List all indexer match algorithm settings
bin/magento indexer:show-mode

# Set batch size per indexer (default is usually 100)
bin/magento config:set catalog/search/elasticsearch6_batch_size 100

# Check current batch configuration
bin/magento config:show catalog/search/elasticsearch6_batch_size

# For very large catalogs (>100k SKUs), reduce batch size:
bin/magento config:set catalog/frontend/grid_batch_size 50
bin/magento config:set indexer/match_algo/batch_size 50
```

#### 5.4: Redis Connection Refused — Deep Diagnostic

Redis connection failures can stem from three root causes. Diagnose systematically:

```bash
# Step 1: Verify Redis process is running
ps aux | grep redis-server | grep -v grep

# Step 2: Test basic connectivity
redis-cli ping
# Expected: PONG

# Step 3: Test with explicit host/port
redis-cli -h redis.example.com -p 6379 ping
redis-cli -h 127.0.0.1 -p 6379 ping

# Step 4: Check Redis configuration
redis-cli CONFIG GET bind
redis-cli CONFIG GET requirepass
redis-cli CONFIG GET maxmemory
redis-cli CONFIG GET tcp-keepalive

# Step 5: Verify network path from app server to Redis
# From the web server:
telnet redis 6379
nc -zv redis 6379
netstat -tlnp | grep 6379

# Step 6: Check AppArmor / SELinux if they might be blocking Redis
# (Ubuntu with AppArmor):
sudo aa-status | grep redis
# (RHEL/CentOS with SELinux):
getsebool redis_connect_bind
```

**Common Redis Configuration Mistakes in env.php:**

```php
// WRONG: using 127.0.0.1 when Redis is in a Docker container with name "redis"
'server' => '127.0.0.1',   // Wrong for Docker setup

// CORRECT: use the Docker service name
'server' => 'redis',         // Correct when Redis is in docker-compose as "redis"

// WRONG: wrong port
'port' => 6380,             // Wrong port (Redis default is 6379)

// WRONG: persistent connection with forcestandalone=true
'persistent' => 'redis-db', // Can cause "already connected" errors
'forcestandalone' => true,   // This overrides persistent

// CORRECT: use PHP redis extension (faster than predis)
'forcestandalone' => false,  // false = use PHP's redis extension
```

**Session-specific Redis troubleshooting:**

```bash
# Check session keys exist after login
redis-cli KEYS "PHPREDIS_SESSION:*" | head -5

# Verify session key TTL
redis-cli TTL "PHPREDIS_SESSION:$(redis-cli KEYS 'PHPREDIS_SESSION:*' | head -1 | cut -d: -f2-)"

# Check Redis session database number matches env.php config
redis-cli SELECT 2
redis-cli DBSIZE

# Monitor Redis commands in real-time (debug mode)
redis-cli MONITOR
# ^C to exit — shows every Redis command received by the server
```

#### 5.5: Category Flat Table Reindex Failure

**What it looks like:**

```
Category Flat Data indexer process error:
optional entity exists and has not been migrated correctly.
Entity: catalog_category_flat_store1_*
```

Or:

```
SQLSTATE[42S02]: Table 'magento.catalog_category_flat_store1_1'
doesn't exist
```

**Diagnosis:**

```bash
# Check the flat table status
bin/magento indexer:status catalog_category_flat

# Verify flat tables exist
mysql -u magento -p -D magento -e "
SHOW TABLES LIKE 'catalog_category_flat%';
"

# Check if category flat tables have the right columns
mysql -u magento -p -D magento -e "
DESCRIBE catalog_category_flat_store1_1;
"
```

**Fix:**

```bash
# Option 1: Reindex the flat tables
bin/magento indexer:reindex catalog_category_flat

# Option 2: If the error persists, reset and reindex
bin/magento indexer:reset catalog_category_flat
bin/magento indexer:reindex catalog_category_flat

# Option 3: If tables are corrupted, drop and recreate
mysql -u magento -p -D magento -e "
DROP TABLE IF EXISTS catalog_category_flat_store1_1;
DROP TABLE IF EXISTS catalog_category_flat_store1_2;
-- etc.
"
bin/magento indexer:reindex catalog_category_flat

# Option 4: Disable flat tables if not needed (performance trade-off)
bin/magento config:set catalog/frontend/flat_catalog_category 0
bin/magento cache:flush
```

#### 5.5: Category Flat Table Reindex Failure

When the category flat table indexer fails, it typically cascades into missing block HTML on category pages. The error manifests as empty category pages or missing product listings:

```
Category Flat Data indexer process error:
optional entity exists and has not been migrated correctly.
Entity: catalog_category_flat_store1_*
```

**Root Cause Analysis:**

```bash
# 1. Check whether flat tables are enabled at all
bin/magento config:show catalog/frontend/flat_catalog_category
# 1 = enabled, 0 = disabled

# 2. Check if the table exists but is stale
mysql -u magento -p -D magento -e "
SELECT COUNT(*) AS row_count
FROM catalog_category_flat_store1_1;
"

# 3. Check for structural differences between EAV and flat tables
mysql -u magento -p -D magento -e "
SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'catalog_category_flat_store1_1'
ORDER BY COLUMN_NAME;
"

# 4. Compare with EAV source table
mysql -u magento -p -D magento -e "
SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'catalog_category_entity'
ORDER BY COLUMN_NAME;
"
```

**Fix — Step-by-Step:**

```bash
# Step 1: Ensure flat tables are clean before reindex
# If you see "optional entity exists" error:
# The flat table has leftover data from a partial migration

# Option A: Reset the indexer completely (most reliable)
bin/magento indexer:reset catalog_category_flat

# Step 2: Verify all flat-related tables are dropped
mysql -u magento -p -D magento -e "
SHOW TABLES LIKE 'catalog_category_flat%';
"

# Step 3: If tables still exist, drop them manually
mysql -u magento -p -D magento -e "
SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS catalog_category_flat_store1_1;
DROP TABLE IF EXISTS catalog_category_flat_store1_2;
-- Add one DROP per store view you have
SET FOREIGN_KEY_CHECKS = 1;
"

# Step 4: Reindex from clean state
bin/magento indexer:reindex catalog_category_flat

# Step 5: Verify
bin/magento indexer:status catalog_category_flat
# Should show: "Ready" (not "Working" or "Error")
```

#### 5.6: Debug Mode vs Production Mode — Feature Comparison Table

| Feature | Developer | Production | Default |
|---|---|---|---|
| Static view files | Published on-demand (symlink) | Must run `setup:static-content:deploy` | Published on-demand |
| Exception stack traces | Shown in browser | Written to `exception.log` only | Shown in browser |
| `debug.log` writing | Active | Inactive (unless explicit config) | Active |
| `bin/magento dev:profiler` | Enabled (`profiler` route active) | Disabled | Disabled |
| DI compilation | Every request (slow) | Pre-compiled via `generated/code/` | Every request |
| Block HTML cache | Disabled (blocks never cached) | Full FPC + block cache | Disabled |
| GraphQL introspection | Full schema | Introspection disabled by default | Full schema |
| Session storage | File (default) | Redis or DB | File |
| CSS/JS processing | Source files served directly | Bundled + minified | Source files |
| Error handler verbosity | High (full details) | Low (generic messages) | High |
| API authentication errors | Detailed reason message | Generic "authentication required" | Detailed |

**Developer Mode Performance Impact:**

Developer mode is NOT suitable for load testing or performance benchmarking. Typical overhead:

- First request after cache flush: 3–8 seconds (DI compilation)
- Static file loading: Uses symlinks instead of copies (fast but fragile)
- Block rendering: No block caching means every block re-renders every request
- Full-page cache: Disabled entirely in developer mode (`cacheable="false"` on all pages)

> **Production rule:** Always diagnose performance issues in a staging environment with production-equivalent settings. Developer mode performance data is meaningless for production sizing.

#### 5.7: Safely Enabling Developer Mode — Complete Workflow

**When you need developer mode:**

- Template syntax errors causing blank white pages
- DI compilation errors that won't show in production mode
- Block rendering errors you cannot trace from logs alone
- GraphQL schema problems (introspection gives full error details)

**Step-by-step activation on a staging environment:**

```bash
# Step 1: Verify current mode
bin/magento deploy:mode:show
# Expected: "Current application mode: production"

# Step 2: Create a staging copy of the environment
# Do NOT modify production directly

# Step 3: Activate developer mode via environment variable
# In your Nginx staging vhost:
#     location ~ ^/staging {
#         fastcgi_param MAGE_MODE developer;
#     }

# Or via Apache:
#     <VirtualHost *:80>
#         ServerName staging.yourdomain.com
#         SetEnv MAGE_MODE developer
#     </VirtualHost>

# Step 4: Verify on staging
bin/magento deploy:mode:show
# Expected: "Current application mode: developer"

# Step 5: After debugging — switch back
# Revert the vhost configuration change
bin/magento deploy:mode:set production
# Expected: "Changed application mode to production"

# Step 6: Verify production is restored
bin/magento deploy:mode:show
# Expected: "Current application mode: production"

# Step 7: Flush all caches after mode change
bin/magento cache:flush
bin/magento setup:static-content:deploy  # In production mode
```

**Emergency debugging on production (last resort):**

Only if you have direct SSH access and the site is completely broken:

```bash
# CRITICAL: Create a backup first
cp app/etc/env.php app/etc/env.php.backup.$(date +%Y%m%d%H%M%S)

# Enable developer mode for 15 minutes, then auto-revert
# Set a calendar reminder before running:
timeout 900 bin/magento deploy:mode:set developer
# After your debugging window:
bin/magento deploy:mode:set production
bin/magento cache:flush

# If something goes wrong mid-debugging:
cp app/etc/env.php.backup.* app/etc/env.php
bin/magento deploy:mode:set production
```

> **Never do this on a live e-commerce site during business hours.** The performance impact affects every customer session.

---

### Section 6: Production Emergency Response Checklist

When a production issue comes in at 2 AM, follow this workflow systematically:

**Step 1 — Triage (first 60 seconds):**

```bash
# Check if the site is up at all
curl -s -o /dev/null -w "%{http_code}" https://yourstore.example.com/
# 200 = up, 000 = cannot connect, 5xx = app error

# Check recent errors
tail -20 var/log/exception.log
tail -20 var/log/system.log

# Check PHP-FPM workers
ps aux | grep php-fpm | wc -l   # Count running workers
top -b -n 1 | head -10         # System load
```

**Step 2 — Identify the scope:**

```bash
# Is it one page or all pages?
curl -s -o /dev/null -w "Homepage: %{http_code}\n" https://yourstore.example.com/
curl -s -o /dev/null -w "Product: %{http_code}\n" https://yourstore.example.com/catalog/product/view/id/145/
curl -s -o /dev/null -w "Category: %{http_code}\n" https://yourstore.example.com/category.html
curl -s -o /dev/null -w "Checkout: %{http_code}\n" https://yourstore.example.com/checkout/

# Is it database?
mysql -u magento -p -e "SELECT 1" magento
# Should return: "1" (1 row)

# Is it Redis?
redis-cli ping
# Should return: PONG
```

**Step 3 — Narrow to the root cause:**

```bash
# If 500 error — check exception.log first
grep -A 5 "Exception" var/log/exception.log | tail -50

# If timeout — check MySQL processlist
mysql -u magento -p -e "SHOW PROCESSLIST;" magento
# Look for queries with "Sleep" state lasting > 60s (idle connections)
# Look for queries with "Locked" state

# If slow — check indexer status
bin/magento indexer:status
# Any indexer showing "Working" during non-peak hours = problem

# If checkout broken — check session storage
redis-cli KEYS "PHPREDIS_SESSION:*" | wc -l
# 0 sessions after login = session storage broken
```

**Step 4 — Take action:**

| Symptom | Immediate Action | Follow-up |
|---|---|---|
| Site completely down | Check if PHP-FPM is running | Restart services, check load |
| 500 errors | `tail -f exception.log` | Identify error class, fix |
| Slow responses | Check query log + New Relic | Optimize slow query/index |
| Checkout broken | Verify Redis/session | Restore session storage |
| Lock wait timeout | `SHOW ENGINE INNODB STATUS` | Kill blocking query or restart MySQL |
| Cache unavailable | `redis-cli ping` | Restart Redis |
| Indexer stuck | `bin/magento indexer:reset` | Reindex during off-peak |

Magento's `MAGE_MODE` dramatically changes what debugging capabilities are available:

| Feature | Developer Mode | Production Mode | Default Mode |
|---|---|---|---|
| Static view publishing | Automatic | Manual (`bin/magento setup:static-content:deploy`) | Automatic |
| Exception handling | Full stack trace | Logged only (`exception.log`) | Full stack trace |
| `var/log/debug.log` | Active | Inactive (unless query logging enabled) | Active |
| `bin/magento dev:profiler` | Enabled | Disabled | Enabled |
| DI compilation | On-demand (each request) | Pre-compiled (`generated/` pre-compiled) | On-demand |
| Symlink for static files | Yes | No (files copied) | Yes |
| Browser error reporting | Enabled | Disabled | Enabled |
| CSS/JS bundling | Source files | Published bundles | Source files |
| GraphQL introspection | Full schema exposed | Introspection disabled | Full schema exposed |
| API authentication errors | Detailed message | Generic "authentication error" | Detailed message |

**Safely Enabling Developer Mode on a Staging Subdirectory**

> **Warning:** Never change `MAGE_MODE` to `developer` on a live production environment. The performance impact is severe and error messages can expose internal paths.

The safest approach is a per-subdirectory override:

```bash
# In the document root of your staging environment, create:
# var/.maintenance.ip (list of IPs allowed during maintenance mode)
# pub/static/.htaccess (already exists — static file signing)

# Set staging to developer mode via environment variable (Apache)
# In your staging virtual host:
#   SetEnv MAGE_MODE developer

# Or in Nginx:
#   fastcgi_param MAGE_MODE developer;

# Verify the mode on staging
bin/magento deploy:mode:show
# Expected output: Current application mode: developer

# On production, mode should be production:
bin/magento deploy:mode:show
# Expected output: Current application mode: production

# NEVER run these on production:
bin/magento deploy:mode:set developer   # NEVER on production
bin/magento dev:profiler:enable         # NEVER on production
```

**What to actually debug in Developer Mode:**

- Template file syntax errors (full stack traces instead of blank pages)
- DI compilation errors (detailed error instead of silent fallback)
- Block rendering errors (verbose logging of layout.xml processing)
- GraphQL schema errors (full validation messages)

**What NOT to debug in Developer Mode (debug in staging):**

- Production performance issues (developer mode is not representative)
- Heavy collection loading (triggers extra logging overhead)
- Full-page cache behavior (developer mode disables FPC)
- Redis session locking issues (developer mode uses file sessions)

---

## Exercises

### Exercise 21.1: Request ID Propagation

1. Create module `DebugTracing` with a plugin on `\Magento\Framework\App\Request\Http`.
2. Implement `beforeDispatch` to inject a UUID into a request attribute.
3. Implement `afterDispatch` to add the header to the HTTP response.
4. `curl -I` your storefront URL and confirm `X-Request-ID` is present.
5. Add a log entry in your plugin and verify the request ID appears in `var/log/debug.log`.

### Exercise 21.2: Query Logging Identification

1. Add `enable_query_logging => true` to your `env.php`.
2. Load a category page that triggers 50+ queries.
3. Identify the top 3 slowest queries using `grep "took" var/log/debug/db_repo.log | sort -t']' -k2 -rn | head -3`.
4. Run `EXPLAIN` on each and add appropriate indexes via `db_schema.xml`.

### Exercise 21.3: New Relic API Tracing

1. If you have New Relic installed, navigate to an active slow GraphQL endpoint.
2. Open the slowest transaction trace.
3. Identify the top method by exclusive time.
4. Trace it to a specific service class method and file.
5. Create a fix (batch loading, index, or query optimization).

### Exercise 21.4: DI Cycle Resolution

1. Create a deliberate DI cycle between two plugins in a test module.
2. Confirm the error message in `var/log/debug.log`.
3. Refactor to use a joint interface.
4. Verify the cycle is resolved.

### Exercise 21.5: MySQL Lock Timeout

1. Start a full reindex on one terminal: `bin/magento indexer:reindex`.
2. Start a second reindex on another terminal simultaneously.
3. Capture the lock error.
4. Run `SHOW ENGINE INNODB STATUS\G` and identify the blocking thread.
5. Resolve by setting `indexer/match_algo/batch_size` to a lower value.

### Exercise 21.6: Redis Connection Fix

1. Intentionally misconfigure the Redis hostname in `env.php`.
2. Load a page and capture the `connection refused` error.
3. Fix the hostname to match your Docker service name.
4. Verify with `redis-cli ping` and page load succeeds.

### Exercise 21.7 (bonus): Staging Developer Mode

1. Configure a `/staging` subdomain with its own Nginx/apache virtual host.
2. Set `MAGE_MODE=developer` via environment variable for that vhost only.
3. Verify production (`maindomain.com`) remains in `MAGE_MODE=production`.
4. Confirm `bin/magento deploy:mode:show` returns different modes per environment.

---

## Reading List

- [Magento 2 Profiler](https://developer.adobe.com/commerce/guide/testing/profiling/) — Official DevDocs on Magento's built-in profiler
- [Magento 2 Logging](https://developer.adobe.com/commerce/guide-testing/logging/) — How Magento's logging system works
- [New Relic Magento Extension](https://docs.newrelic.com/docs/apm/agents/php-agent/frameworks-libraries/magento-2x/) — Installing and configuring the New Relic PHP agent for Magento
- [MySQL EXPLAIN](https://dev.mysql.com/doc/refman/8.0/en/explain.html) — MySQL 8.0 EXPLAIN syntax and interpretation
- [Redis Connection Issues](https://experienceleague.adobe.com/docs/commerce-knowledge-base.kbtotal/solve-tightening-security-on-magento-commerce.html) — Adobe Commerce KB on common Redis connection errors
- [MAGE_MODE](https://developer.adobe.com/commerce/guide/configuration/bootstrap/magento-modes.html) — DevDocs reference for Magento application modes
- [EnableQUERY_LOGGING](https://developer.adobe.com/commerce/guide/configuration/env/env-database/) — Database configuration in env.php including query logging
- [GraphQL Debugging](https://developer.adobe.com/commerce/graphql/) — GraphQL debugging techniques for Magento
- [Innodb Lock Wait Timeout](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout) — MySQL lock wait timeout configuration
