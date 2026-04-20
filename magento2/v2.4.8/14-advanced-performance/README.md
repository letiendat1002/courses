# Topic 14: Advanced Performance Engineering (incl. Topic 15)

**Philosophy:** Performance is not a feature — it is the foundation on which every other feature stands or falls.

---

## Overview

This module takes you beyond the basics of Magento 2 caching and search, diving into the infrastructure-level decisions that separate a development environment from a production-ready deployment. You will configure Varnish full-page caching from scratch, wire Elasticsearch into Magento's search adapter and beyond, implement advanced Redis patterns including distributed locks and session clustering, and set up profiling to identify hot spots before they reach production.

By the end of this module you will have a working Docker-based performance lab, a custom module exercising every major pattern, and the ability to diagnose and solve performance regressions in a real Magento codebase.

---

## Prerequisites (from core course)

- Topic 08 (Data Operations) or Topic 09 (Performance) completed
- Docker Compose environment with at least 4 GB RAM available
- Magento 2.4.x installed with composer access (ee or open source)
- SSH / CLI access and basic `bin/magento` competence
- Composer dependencies installed: `predis/predis` and `elasticsearch/elasticsearch` (v8)
- A Unix-like host (Linux or macOS); Varnish and Redis are native Linux services

---

## Learning Objectives

1. Explain how Varnish acts as a reverse-proxy FPC and why it outperforms Magento's built-in FPC in high-traffic scenarios.
2. Configure Varnish within a Docker Compose stack and wire it to Magento via `env.php`.
3. Interpret Magento's cache tag taxonomy (`FPC`, `CUSTOMER`, `PRODUCT_1`, `CATEGORY_5`) and implement tagged invalidation.
4. Design ESI block strategies: when to use `cacheable="false"`, when to use proper cache key design, and how request coalescing reduces thundering-herd.
5. Install and configure Elasticsearch 8.x in Docker, map Magento's product catalog, and issue queries via the official PHP client.
6. Build a custom Elasticsearch index for a non-product entity (e.g. reviews) and implement autocomplete via the completion suggester.
7. Implement Redis distributed locks using `SETNX` and the Redlock algorithm to protect concurrent operations.
8. Configure Redis sessions with proper lifetime settings, diagnose and avoid session locking issues, and set up Redis Sentinel for failover.
9. Integrate Tideways/XHProf, profile a slow `bin/magento` command, and interpret callgraphs to find the top 5 hot functions.

---

## By End of Module You Must Prove

- Varnish serves a CMS page with `HIT` headers and `X-Cache-Tags` on the second request, while the first request shows `MISS`.
- A custom review-approved observer invalidates the associated product's FPC entry by tag.
- A custom Elasticsearch index for reviews returns results filtered by `product_id` using a `bool.filter` query.
- A Redis distributed lock correctly prevents a second concurrent process from acquiring the same lock.
- `redis-cli KEYS` shows session keys matching `PHPREDIS_SESSION:` after logging into Magento's storefront.
- A Tideways/XHProf callgraph for `bin/magento indexer:reindex` identifies the top 5 functions by exclusive wall time.

---

## Assessment Criteria

| Criterion | Evidence |
|---|---|
| Varnish FPC configured and verified | `curl -I` output showing `X-Cache-Tags` and `HIT` on repeat request |
| Tagged invalidation implemented | Observer class + `CacheInterface::invalidate()` + test output |
| Elasticsearch custom index | API call showing index exists + 3 documents indexed + filtered query result |
| Redis distributed lock | Lock acquired / denied log entries from two concurrent processes |
| Redis session storage | `redis-cli KEYS "PHPREDIS_SESSION:*"` returns session keys |
| Profiling callgraph | Screenshot or text output of top-5 functions by wall time |

---

## Topics

### Topic 1: Varnish Full-Page Cache — Configuration & Tuning

#### How Varnish Works with Magento: ESI (Edge Side Includes) Fragments

Varnish Cache is an HTTP reverse-proxy that sits in front of your Magento application server (often called the "backend"). When a request arrives, Varnish checks its in-memory cache. On a cache hit it serves the response instantly without touching the backend. On a cache miss it forwards the request to Magento, stores the response in cache, and returns it to the client.

Magento's page content is not monolithic — a single rendered page is composed of dozens of layout blocks, many of which carry dynamic data (cart count, customer name, stock status). Varnish cannot cache an entire page when any single block is dynamic. This is where **Edge Side Includes (ESI)** come in.

With ESI, Varnish can cache the page shell and pull in specific blocks independently at request time:

```
<!-- Varnish fetches this from its cache -->
<html>
  <body>
    <header><!-- could be ESI --></header>
    <main><!-- served from Varnish cache --></main>
    <footer><!-- could be ESI --></footer>
  </body>
</html>
```

When Magento sends a response with the HTTP header `Surrogate-Control:ESI/1.0`, Varnish parses the response and replaces ESI surrogate tags with content fetched from the specified URL. This allows Varnish to cache pages that contain both static and dynamic fragments independently.

In practice, most Magento blocks are rendered server-side and cached as full HTML. The key insight is that Varnish caches the **full page output** including the server-side rendered block HTML. When Magento issues cache invalidation calls with tags headers, Varnish purges only the tagged entries — not the entire cache.

#### Varnish vs Built-in FPC: When Varnish Wins

Magento ships with a built-in full-page cache (`Magento\PageCache\Model\Cache`) that stores rendered pages in the configured cache backend (Redis or file system). The critical difference is **where the cache lives**:

| Characteristic | Built-in FPC | Varnish |
|---|---|---|
| Location | Same server as PHP-FPM | Separate HTTP proxy layer |
| Cache miss cost | Full PHP bootstrap + DB queries + block rendering | Proxy request to backend |
| Traffic handling | All requests hit PHP-FPM | Only cache misses hit PHP-FPM |
| TTL precision | Per-tag invalidation | Per-URL / tagged invalidation |
| Grace mode | Not built-in | Supported via `vcl_recv` |
| Saint mode | Not built-in | Built-in backend failure mode |
| ESI support | None | Full ESI/1.0 |
| HTTP/2 | Not relevant | Native multiplexing |

Varnish wins when:
- Your traffic spikes unpredictably (flash sales, product launches)
- You run multiple PHP-FPM nodes behind a load balancer (Varnish is a shared cache)
- You need grace mode to serve stale content while a backend is recovering
- Your PHP-FPM workers are the bottleneck and you want to keep them idle for real dynamic requests

For a single-server development environment the built-in FPC is sufficient. For anything going to production with more than a handful of concurrent users, Varnish is the standard.

#### Installing Varnish in Docker Compose Environment

Varnish does not run inside a PHP container — it is a standalone service that proxies HTTP to the PHP-FPM container. Here is a production-grade `docker-compose.yml` snippet:

```yaml
version: "3.8"

services:
  varnish:
    image: varnish:7.5
    container_name: magento-varnish
    volumes:
      - ./varnish/default.vcl:/etc/varnish/default.vcl:ro
    ports:
      - "80:80"
    depends_on:
      - web
    restart: unless-stopped
    environment:
      VARNISH_SIZE: 256M
      VARNISHD_THREADS: "+2"
    healthcheck:
      test: ["CMD", "varnishadm", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  web:
    image: php:8.1-fpm
    container_name: magento-web
    # ... php-fpm config, volume mounts, etc.
```

**Key points:**
- Varnish listens on port 80; `web` listens on port 9000 internally.
- The VCL file is mounted read-only into the container.
- `VARNISH_SIZE` sets the cache storage size (use at least 256M for a dev environment).
- The health check uses `varnishadm ping` to confirm Varnish is alive.

After `docker-compose up -d varnish`, confirm Varnish is running:

```bash
docker exec magento-varnish varnishadm ping
# Output: PONG 1
```

#### Magento `env.php` Varnish Configuration

In `app/etc/env.php`, the full-page cache section must point Magento's cache layer at Varnish instead of the built-in cache:

```php
<?php
return [
    // ... other sections
    'system' => [
        'default' => [
            'system' => [
                'full_page_cache' => [
                    'caching_application' => '2',   // 2 = Varnish
                    'ttl' => 86400,                  // 24 hours default TTL
                ],
            ],
        ],
    ],
    'cache' => [
        'frontend' => [
            'full_page' => [
                'backend' => 'Magento\\Framework\\Cache\\Backend\\Varnish',
                'backend_options' => [
                    'http_host'      => 'http://varnish',
                    'http_port'      => '80',
                    'http_timeout'   => 2,
                    'http_full_host' => null,       // null = use HTTP_HOST
                ],
            ],
        ],
    ],
];
```

The integer value `2` for `caching_application` selects Varnish in the admin UI dropdown (`System → Cache Management → Full Page Cache`). The `http_host` should match the hostname your Magento container uses in the Docker network.

Alternatively, use the CLI to set this without editing the file directly:

```bash
bin/magento setup:config:set \
  --http-cache-hosts=varnish:6081 \
  --full-page-cache-backend=varnish
```

Verify the configuration was written:

```bash
bin/magento config:show system/full_page_cache/caching_application
# Expected output: 2
```

#### Cache Lifetime Headers: `Cache-Control`, `X-Magento-Tags`, `X-Cache-Tags`

When Magento renders a page with Varnish enabled, it sets specific HTTP response headers that Varnish uses for caching decisions:

**`Cache-Control`**  
Controls how long the response may be cached and under what conditions it may be served stale:

```
Cache-Control: max-age=86400, public, s-maxage=86400
```

- `max-age=86400` — browser cache lifetime (seconds)
- `s-maxage=86400` — shared proxy cache lifetime (Varnish respects this)
- `public` — response may be cached by any cache (vs `private` which only allows browser caching)

**`X-Magento-Tags`**  
Comma-separated list of cache tags associated with this page. Varnish stores these tags and uses them for targeted invalidation:

```
X-Magento-Tags: FPC,CATEGORY_5,PRODUCT_1,CUSTOMER_1
```

When a product with ID 1 is updated, Magento's observer fires a DELETE request to Varnish's purge endpoint with the tag `PRODUCT_1`, causing Varnish to evict every cached page that contained that tag.

**`X-Cache-Tags`** (response from Varnish)  
When Varnish serves a cached response, it echoes back the tags it used, allowing the client to confirm cache state:

```
X-Cache-Tags: FPC,CATEGORY_5,PRODUCT_1,CUSTOMER_1
X-Cache: HIT
```

**`Pragma`** (legacy)  
Sometimes Magento still emits `Pragma: no-cache` on private pages. Varnish should be configured to ignore Pragma on public pages:

```vcl
sub vcl_backend_response {
    if (bereq.url ~ "^/(media|static|pub/)") {
        set beresp.ttl = 604800s;
        set beresp.http.Cache-Control = "public, max-age=604800";
    }
    # Ignore Pragma header for static assets
    unset beresp.http.Pragma;
}
```

#### Cache Invalidation Patterns: Tagged Invalidation vs Full Flush

**Tagged invalidation** is the preferred pattern. Instead of flushing the entire cache, you invalidate only the pages that contain a specific entity:

```php
<?php
// In an observer for catalog_product_save_after
use Magento\Framework\App\CacheInterface;
use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;

class InvalidateProductCache implements ObserverInterface
{
    public function __construct(
        private readonly CacheInterface $cache,
    ) {}

    public function execute(Observer $observer): void
    {
        $product = $observer->getEvent()->getProduct();
        $this->cache->invalidate(['PRODUCT_' . $product->getId()]);
    }
}
```

Varnish receives a `PURGE` request on its admin port or a tagged purge HTTP request and removes the matching cache entries. This keeps all other cached pages alive and reduces backend load after partial invalidations.

**Full flush** clears everything:

```bash
bin/magento cache:flush full_page
# or
curl -X PURGE http://varnish:6081/
```

Full flush is acceptable in limited scenarios:
- After a major theme change that affects every page
- During a full catalog reindex that touches all products
- As a manual emergency response when tagged invalidation misses stale data

Never use full flush as part of an automated workflow — it creates a "thundering herd" where all subsequent requests miss the cache simultaneously.

#### Grace Period, Saint Mode, and Hard Timeout Configuration

**Grace period** (`vcl_recv`) tells Varnish to serve stale cached content while a new request is being made to the backend:

```vcl
sub vcl_recv {
    # Serve stale content for up to 120 seconds while fetching
    if (req.url ~ "^/") {
        set req.http.grace = "120s";
    }
}
```

```vcl
sub vcl_backend_response {
    set beresp.grace = 120s;
}
```

This means if a page is cached but its TTL has expired, and the backend is slow or down, Varnish will still serve the stale page for up to 2 minutes rather than returning an error.

**Saint mode** is Varnish's built-in backend failure protection. When a backend is responding with errors or timing out, saint mode blacklists that backend for a configurable period and tries an alternative (if you have multiple backends):

```vcl
sub vcl_backend_response {
    if (beresp.status == 500 || beresp.status == 503) {
        set beresp.saintmode = 60s;
        return (retry);
    }
}
```

**Hard timeout** (`connect_timeout`, `first_byte_timeout`, `between_bytes_timeout`) limits how long Varnish waits for the backend:

```vcl
sub vcl_backend {
    backend default {
        .host = "web";
        .port = "9000";
        .connect_timeout = 5s;
        .first_byte_timeout = 300s;   # large for slow catalog pages
        .between_bytes_timeout = 10s;
        .probe = {
            .url = "/health_check.php";
            .timeout = 2s;
            .interval = 10s;
            .window = 5;
            .threshold = 3;
        }
    }
}
```

Set `first_byte_timeout` high enough to accommodate slow category pages with many blocks (300s is reasonable for a dev environment; production may need tuning).

#### Varnish Configuration File (VCL) Basics for Magento

The VCL is Varnish's domain-specific configuration language compiled to C at startup. Below is a production-ready `default.vcl` for Magento:

```vcl
vcl 4.1;

backend default {
    .host = "web";
    .port = "9000";
    .connect_timeout = 5s;
    .first_byte_timeout = 300s;
    .between_bytes_timeout = 10s;
    .probe = {
        .url = "/pub/health_check.php";
        .timeout = 2s;
        .interval = 10s;
        .window = 5;
        .threshold = 3;
    }
}

acl purge {
    "localhost";
    "127.0.0.1";
    "172.17.0.0/16";  # Docker bridge default   # Docker network
}

sub vcl_recv {
    # Remove cookies we don't need for caching
    if (req.url ~ "^/media/catalog/") {
        unset req.http.Cookie;
        return (hash);
    }

    # Do not cache POST requests
    if (req.method == "POST") {
        return (pass);
    }

    # Do not cache checkout, customer, and admin URIs
    if (req.url ~ "^(/checkout|/customer|/admin|/wishlist|/sendfriend)") {
        return (pass);
    }

    # PURGE request handling
    if (req.method == "PURGE") {
        if (!client.ip ~ purge) {
            return (synth(405, "Not allowed."));
        }
        return (purge);
    }

    # Ban request (alternative invalidation method)
    if (req.method == "BAN") {
        if (!client.ip ~ purge) {
            return (synth(405, "Not allowed."));
        }
        ban("obj.http.X-Magento-Tags ~ " + req.url);
        return (synth(200, "Ban added"));
    }

    # Remove tracking parameters that would bust cache
    set req.url = regsub(req.url, "\?utm_[^=]+=[^&]+", "");
    set req.url = regsub(req.url, "\?gclid=[^&]+", "");

    return (hash);
}

sub vcl_hash {
    hash_data(req.url);
    if (req.http.host) {
        hash_data(req.http.host);
    } else {
        hash_data(server.ip);
    }
    return (lookup);
}

sub vcl_backend_response {
    # Gzip content if backend doesn't
    if (beresp.http.content-type ~ "text|(application|apachelog|manifest)json") {
        set beresp.do_gzip = true;
    }

    # Cache static assets for 1 week (override TTL)
    if (bereq.url ~ "^/(media|static|pub/static)") {
        set beresp.ttl = 604800s;
        unset beresp.http.Set-Cookie;
        set beresp.http.Cache-Control = "public, max-age=604800";
    }

    # Grace mode: serve stale while fetching
    set beresp.grace = 120s;

    # Do not cache private content
    if (beresp.http.Cache-Control ~ "private") {
        set beresp.uncacheable = true;
        return (deliver);
    }

    return (deliver);
}

sub vcl_deliver {
    # Add cache hit/miss header
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
    } else {
        set resp.http.X-Cache = "MISS";
    }

    # Add tags header for debugging
    set resp.http.X-Cache-Tags = resp.http.X-Magento-Tags;

    # Remove internal headers
    unset resp.http.X-Generator;
    unset resp.http.X-Powered-By;

    return (deliver);
}

sub vcl_synth {
    if (resp.status == 503 && resp.reason ~ "Backend fetch failed") {
        set resp.status = 503;
        set resp.http.Content-Type = "text/html; charset=utf-8";
        synthetic({"<html><body><h1>Service temporarily unavailable</h1></body></html>"});
        return (deliver);
    }
}
```

---

### Topic 2: Varnish — Cache Tags & Edge Rules

#### Magento Cache Tags: Taxonomy and Naming Convention

Magento generates cache tags using a predictable naming convention:

| Tag Pattern | Meaning | Invalidation Trigger |
|---|---|---|
| `FPC` | The full-page cache root tag — present on every page | Rarely invalidated alone |
| `CUSTOMER_{id}` | Specific customer | Customer edit, logout |
| `PRODUCT_{id}` | Product page | Product save, price change, stock change |
| `CATEGORY_{id}` | Category page | Category save, product moved to/from category |
| `CMS_P_{id}` | CMS page | CMS page save |
| `CATALOG_PRODUCT_{id}` | Product in catalog (shared index) | Product save |
| `SEARCH_FILTERS_{store_id}` | Layered navigation filters | Filter attribute change |

You can inspect the tags on any page by checking the `X-Magento-Tags` response header. Example on a product page:

```
X-Magento-Tags: FPC,CATEGORY_4,CATEGORY_5,CATEGORY_8,PRODUCT_1,PRODUCT_42
```

The first tag is always `FPC`. Subsequent tags are the specific entities embedded on that page.

#### Tagged Invalidation via `CacheInterface::invalidate()`

The `Magento\Framework\App\CacheInterface` service (`cache` di alias) exposes:

```php
public function invalidate(array $tags): void;
```

When called, it issues a purge signal for each tag. With Varnish as the backend, this becomes an HTTP PURGE request to Varnish's purge endpoint.

Example: invalidating product 42's cache after a price update:

```php
<?php
declare(strict_types=1);
namespace Training\Performance\Observer;

use Magento\Framework\App\CacheInterface;
use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;

class ProductPriceUpdated implements ObserverInterface
{
    public function __construct(
        private readonly CacheInterface $cache,
        private readonly ProductRepositoryInterface $productRepository,
    ) {}

    public function execute(Observer $observer): void
    {
        /** @var \Magento\Catalog\Api\Data\ProductInterface $product */
        $product = $observer->getEvent()->getProduct();
        $cacheTags = [
            'PRODUCT_' . $product->getId(),
            'CATALOG_PRODUCT_' . $product->getId(),
        ];

        // Also invalidate all category pages this product appears in
        $categoryIds = $product->getCategoryIds();
        foreach ($categoryIds as $categoryId) {
            $cacheTags[] = 'CATEGORY_' . $categoryId;
        }

        $this->cache->invalidate($cacheTags);
    }
}
```

Register the observer in `etc/events.xml`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="catalog_product_save_after">
        <observer name="training_performance_product_price_updated"
                  instance="Training\Performance\Observer\ProductPriceUpdated"/>
    </event>
</config>
```

#### `cache_invalidate` Observer Pattern — When Magento Invalidates FPC

Magento's FPC invalidation is driven by the `cache_invalidate` event. The built-in `\Magento\PageCache\Observer\InvalidateCache` listener watches for specific events and triggers invalidation:

```php
<?php
// Simplified from Magento\PageCache\Observer\InvalidateCache
public function execute(\Magento\Framework\Event\Observer $observer): void
{
    $tags = [];

    switch ($eventName) {
        case 'catalog_product_save_after':
            /** @var ProductInterface $product */
            $product = $observer->getEvent()->getProduct();
            $tags[] = 'PRODUCT_' . $product->getId();
            break;

        case 'catalog_category_save_after':
            /** @var Category $category */
            $category = $observer->getEvent()->getCategory();
            $tags[] = 'CATEGORY_' . $category->getId();
            break;

        case 'cms_page_save_after':
            /** @var PageInterface $page */
            $page = $observer->getEvent()->getPage();
            $tags[] = 'CMS_P_' . $page->getId();
            break;

        case 'sales_order_save_after':
            // Do NOT invalidate FPC on order save — only on specific actions
            return;
    }

    if (!empty($tags)) {
        $this->cacheManager->invalidate($tags);
    }
}
```

You can hook into the same `cache_invalidate` event yourself to add custom invalidation:

```xml
<!-- etc/events.xml -->
<event name="cache_invalidate">
    <observer name="training_performance_custom_invalidate"
              instance="Training\Performance\Observer\InvalidateOnCustomEvent"/>
</event>
```

```php
<?php
declare(strict_types=1);
namespace Training\Performance\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;

class InvalidateOnCustomEvent implements ObserverInterface
{
    public function execute(Observer $observer): void
    {
        $tags = $observer->getEvent()->getTags() ?? [];
        if (!empty($tags)) {
            // Add custom tag invalidation
            $tags[] = 'CUSTOM_REVIEW_ENTITY_42';
            $observer->getEvent()->setTags($tags);
        }
    }
}
```

#### ESI Block Strategy: Which Blocks Should Be ESI vs Cached

Not every block should be ESI. Here is a decision matrix:

| Block Type | Cache Strategy | Reason |
|---|---|---|
| Header (logo, nav links) | Cached | Rarely changes per session |
| Product list (category) | Cached | Changes only on reindex |
| Recently viewed | ESI or uncached | Personalized per customer |
| Cart sidebar | ESI or uncached | Changes on every cart action |
| Customer name | ESI or uncached | Personalized |
| CMS static block | Cached | Managed via admin, invalidates on change |
| Price | Cached (with tag) | Invalidated via PRODUCT_* tag |
| Stock status | Cached (short TTL) | Can go stale; 5-15 min TTL |
| Navigation menu | Cached | Changes only on category save |

#### Making a Block ESI-Aware: `cacheable="false"` vs Proper Cache Key Design

**The wrong way** — `cacheable="false"` in layout XML:

```xml
<block class="Magento\Framework\View\Element\Template"
       name="customer.greeting"
       template="Training_Performance::greeting.phtml"
       cacheable="false"/>
```

Setting `cacheable="false"` on a block marks the **entire page** as uncacheable in Magento's built-in FPC. With Varnish, the page is still cached, but Varnish cannot ESI-include individual blocks — it has no knowledge of Magento's block structure. The `cacheable="false"` attribute tells Magento not to wrap this block in its cache layer, not to make it an ESI fragment.

**The correct way** — proper cache key design for dynamic per-customer blocks:

```php
<?php
// etc/di.xml
<type name="Magento\Customer\Block\Account\Customer">
    <arguments>
        <argument name="cache_key_info" xsi:type="array">
            <item name="customer_id" xsi:type="string">getCustomerId</item>
            <item name="website_id" xsi:type="string">getWebsiteId</item>
            <item name="store_id" xsi:type="string">getStoreId</item>
        </argument>
    </arguments>
</type>
```

With this cache key, the block output is cached **per customer** rather than being excluded from the cache entirely. The page itself remains cacheable; only this block generates per-customer content.

**ESI in Varnish** — the correct pattern if you want true dynamic sub-content within a cached page:

Varnish ESI is activated when Magento sends:

```
Surrogate-Control: content="ESI/1.0"
```

For a block to become an ESI include, Magento's `\Magento\PageCache\Model\Esi` service processes specific blocks and replaces them with an ESI include URL. In practice, the canonical pattern in Magento 2 is to use the `Magento\Framework\View\Element\Context` block's `_loadCache` / `_saveCache` methods, which integrate with the built-in FPC. When Varnish is enabled, the page output includes ESI tags for blocks that call `$block->getIdentities()`.

Blocks that implement `\Magento\Framework\DataObject\IdentityInterface` and return non-empty identities from `getIdentities()` are automatically invalidated by tag, even when served through Varnish's cache.

#### Request Coalescing (Thundering-Herd Throttling)

When a popular cached page expires, a sudden burst of concurrent requests all miss the cache simultaneously and rush to the backend — this is the thundering herd problem.

Magento's built-in FPC handles this partially with process-level locking. Varnish provides **request coalescing** (also called request collapsing) natively: when multiple requests arrive for the same uncached URL, Varnish sends only **one** request to the backend and serves all waiting clients from that single response.

```vcl
sub vcl_recv {
    # Do not coalesce PURGE or BAN requests
    if (req.method ~ "^(PURGE|BAN)$") {
        return (purge);
    }
}
```

You can confirm coalescing is active by watching Varnish's log during a cache miss:

```bash
varnishlog -g request -i ReqStart
# You will see only ONE backend request even with 50 concurrent curl calls
```

For application-level coalescing in Magento (e.g., for Redis or database calls), use the built-in `\Magento\Framework\Lock\LockManagerInterface`:

```php
<?php
public function getExpensiveData(string $identifier, callable $dataLoader): mixed
{
    $lockKey = 'lock_' . $identifier;
    if ($this->lockManager->lock($lockKey, 30)) {
        try {
            $data = $dataLoader();
            $this->cache->save($data, $identifier, ['MY_TAG'], 3600);
            return $data;
        } finally {
            $this->lockManager->unlock($lockKey);
        }
    }

    // Another process holds the lock — wait briefly then return stale
    usleep(100000); // 100ms
    return $this->cache->load($identifier) ?: $dataLoader();
}
```

---

### Topic 3: Elasticsearch — Architecture & Magento Integration

#### Why Elasticsearch for Catalog Search (vs MySQL LIKE)

MySQL's `LIKE '%query%'` and even `FULLTEXT` search are designed for general-purpose text matching, not for the complex filtering, relevance tuning, faceted navigation, and performance requirements of a high-volume e-commerce catalog.

| Feature | MySQL LIKE / FULLTEXT | Elasticsearch |
|---|---|---|
| Query latency at scale | Degrades with catalog size | Sub-millisecond at millions of documents |
| Faceted navigation | Requires complex GROUP BY queries | Aggregations are native |
| Relevance tuning | Limited | Per-field boost, synonym expansion, fuzzy matching |
| Typo tolerance | None (without third-party) | Fuzzy queries built-in |
| Autocomplete | Not supported natively | Completion suggester, search-as-you-type |
| Multi-field search | Single index at a time | Cross-index, cross-type querying |
| Indexing impact on DB | Blocks writes during index updates | Near-real-time indexing (1s refresh) |

Elasticsearch is not a replacement for MySQL — it does not store your transactional data. It is a **search engine** that maintains its own index of your product data, synchronized via Magento's indexer.

#### Elasticsearch Architecture: Index, Document, Mapping

**Cluster** — A cluster is a collection of one or more nodes that together hold all data and provide unified search. A production Magento setup uses at minimum 3 nodes for quorum-based leader election.

**Node** — A single Elasticsearch process. Types:
- **Master node** — controls cluster topology, index creation/deletion
- **Data node** — stores and queries data (search happens here)
- **Ingest node** — pre-processes documents before indexing (optional)
- **Coordinating node** — routes requests, aggregates results (load balancer role)

**Index** — A logical namespace, analogous to a database in MySQL. Contains documents. Named must be lowercase, no spaces.

**Document** — The basic unit of information. JSON object with a unique `_id`. Has a `_type` (in ES 7.x+ the type is `_doc`). In Magento 2.4's `elasticsearch7` module, all documents use `_doc`.

**Mapping** — Schema definition for an index. Describes field names, data types, and how they should be indexed/analyzed:

```json
{
  "mappings": {
    "properties": {
      "sku":            { "type": "keyword" },
      "name":           { "type": "text", "analyzer": "english" },
      "description":    { "type": "text", "analyzer": "english" },
      "price":          { "type": "float" },
      "category_ids":   { "type": "keyword" },
      "stock_qty":      { "type": "integer" },
      "created_at":     { "type": "date" }
    }
  }
}
```

**Shards** — An index is split into shards (default 1 primary + 1 replica). Sharding allows horizontal scaling. Default is 5 primary shards.

**Replica** — Copy of a primary shard. Provides redundancy and read throughput. Minimum 1 replica in production.

#### Installing Elasticsearch 8.x in Docker (Magento 2.4 Requirements)

Magento 2.4 requires **Elasticsearch 7.x** or **OpenSearch 1.x** (via a compatibility layer). Elasticsearch 8.x is not officially supported by Magento 2.4 out of the box — use **Elasticsearch 7.17.x** for full compatibility. If you need ES 8.x features, use OpenSearch 1.x or apply the official Magento patch.

Docker Compose service for Elasticsearch 7.17:

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.16
    container_name: magento-elasticsearch
    environment:
      - node.name=es-node-1
      - cluster.name=magento-cluster
      - discovery.type=single-node    # dev only; use multi-node in prod
      - bootstrap.memory_lock=true
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
      - xpack.security.enabled=false   # disable for dev; enable in prod
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    mem_limit: 1g
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9200/_cluster/health | grep -vq '\"status\":\"red\"'"]
      interval: 15s
      timeout: 10s
      retries: 5

volumes:
  es-data:
    driver: local
```

Start the service:

```bash
docker-compose up -d elasticsearch

# Wait for startup (up to 30s)
sleep 10

# Verify
curl -s http://localhost:9200 | head -5
```

Expected output:

```json
{
  "name" : "es-node-1",
  "cluster_name" : "magento-cluster",
  "cluster_uuid" : "abc123",
  "version" : {
    "number" : "7.17.16",
    ...
  }
}
```

#### Magento's Search Engine Adapter: `elasticsearch7` Module

Magento 2.4 includes the `Magento_Elasticsearch7` module that replaces the older `Elasticsearch` module. It provides:

- Search adapter implementing `\Magento\Search\Api\SearchInterface`
- Query builder for Elasticsearch DSL
- Autocomplete / suggestions adapter
- Catalog Search indexer that populates the `catalogsearch_fulltext` index

Enable the module if not already enabled:

```bash
bin/magento module:enable Magento_Elasticsearch7
bin/magento setup:upgrade
bin/magento indexer:reindex catalogsearch_fulltext
```

#### Elasticsearch Index Structure for Products

Magento creates an index named `magento2_product_1` (or `magento2_product_{website_id}_{store_id}` depending on configuration). You can inspect it directly:

```bash
curl -s 'http://localhost:9200/magento2_product_1/_mapping?pretty' | jq '.'
```

Key fields mapped by Magento's Elasticsearch adapter:

| Field Name | ES Type | Notes |
|---|---|---|
| `name` | `text` + `keyword` | Analyzed for full-text, keyword for sorting |
| `description` | `text` | English analyzer |
| `sku` | `keyword` | Exact match, not analyzed |
| `price` | `float` | For range filtering |
| `category_ids` | `keyword` | Array of category IDs |
| `visibility` | `integer` | 1=catalog,2=search,3=both,4=not visible individually |
| `store_id` | `integer` | Per-store indexing |
| `entity_id` | `integer` | Product ID |

#### Catalog Search Adapter Configuration in `env.php`

```php
<?php
return [
    // ...
    'system' => [
        'default' => [
            'catalog' => [
                'search' => [
                    'engine' => 'elasticsearch7',
                    'elasticsearch7_server_hostname' => 'localhost',
                    'elasticsearch7_server_port' => '9200',
                    'elasticsearch7_index_prefix' => 'magento2',
                    'elasticsearch7_enable_auth' => false,
                ],
            ],
        ],
    ],
];
```

Or via CLI:

```bash
bin/magento config:set catalog/search/engine elasticsearch7
bin/magento config:set catalog/search/elasticsearch7_server_hostname localhost
bin/magento config:set catalog/search/elasticsearch7_server_port 9200
```

Test the connection:

```bash
bin/magento catalog:search:reindex
# or
bin/magento indexer:reindex catalogsearch_fulltext
```

---

### Topic 4: Elasticsearch — Custom Indexing & Queries

#### Creating a Custom Index for Non-Product Entities (Reviews)

Magento's product search index is managed entirely by `catalogsearch_fulltext`. For custom entities like reviews or orders, you manage the index yourself.

Step 1: Create the index via API:

```php
<?php
declare(strict_types=1);
namespace Training\Performance\Elasticsearch;

use Elastic\Elasticsearch\Client;
use Elastic\Elasticsearch\ClientBuilder;

class ReviewIndexManager
{
    private Client $client;

    public function __construct()
    {
        $this->client = ClientBuilder::create()
            ->setHosts(['localhost:9200'])
            ->build();
    }

    public function createIndex(): void
    {
        $indexName = 'training_reviews';

        $exists = $this->client->indices()->exists(['index' => $indexName]);
        if ($exists->asBool()) {
            $this->client->indices()->delete(['index' => $indexName]);
        }

        $this->client->indices()->create([
            'index' => $indexName,
            'body' => [
                'settings' => [
                    'number_of_shards' => 1,
                    'number_of_replicas' => 0,
                    'analysis' => [
                        'analyzer' => [
                            'review_analyzer' => [
                                'type' => 'custom',
                                'tokenizer' => 'standard',
                                'filter' => ['lowercase', 'stop'],
                            ],
                        ],
                    ],
                ],
                'mappings' => [
                    'properties' => [
                        'review_id'    => ['type' => 'integer'],
                        'product_id'   => ['type' => 'integer'],
                        'customer_id'  => ['type' => 'integer'],
                        'title'        => [
                            'type' => 'text',
                            'analyzer' => 'review_analyzer',
                            'fields' => [
                                'keyword' => ['type' => 'keyword'],
                            ],
                        ],
                        'detail'       => [
                            'type' => 'text',
                            'analyzer' => 'review_analyzer',
                        ],
                        'rating'       => ['type' => 'float'],
                        'status'       => ['type' => 'keyword'],
                        'created_at'   => ['type' => 'date'],
                    ],
                ],
            ],
        ]);

        echo "Index '{$indexName}' created successfully.\n";
    }
}
```

Run it:

```bash
bin/magento queue:consume Training\Performance\Elasticsearch\ReviewIndexManager
# Or call directly via a CLI command for setup
```

#### Indexing a Document: Bulk API and Single Document Indexing

**Single document indexing:**

```php
<?php
public function indexReview(array $reviewData): void
{
    $this->client->index([
        'index' => 'training_reviews',
        'id'    => (string) $reviewData['review_id'],
        'body'  => $reviewData,
    ]);
}
```

**Bulk API** — for indexing many documents efficiently (e.g., after a mass import):

```php
<?php
public function bulkIndexReviews(array $reviews): void
{
    $params = ['body' => []];

    foreach ($reviews as $review) {
        $params['body'][] = [
            'index' => [
                '_index' => 'training_reviews',
                '_id'    => (string) $review['review_id'],
            ],
        ];
        $params['body'][] = $review;
    }

    $response = $this->client->bulk($params);
    if ($response['errors'] === true) {
        foreach ($response['items'] as $item) {
            if (isset($item['index']['error'])) {
                echo "Error indexing doc {$item['index']['_id']}: " .
                     $item['index']['error']['reason'] . "\n";
            }
        }
    }
}
```

Index 3 review documents:

```php
<?php
$reviews = [
    [
        'review_id'   => 1,
        'product_id'  => 42,
        'customer_id' => 100,
        'title'       => 'Excellent build quality',
        'detail'      => 'The product exceeded my expectations in every way.',
        'rating'      => 5.0,
        'status'      => 'approved',
        'created_at'  => '2024-01-15T10:30:00Z',
    ],
    [
        'review_id'   => 2,
        'product_id'  => 42,
        'customer_id' => 101,
        'title'       => 'Good value for money',
        'detail'      => 'Solid product with minor issues.',
        'rating'      => 4.0,
        'status'      => 'approved',
        'created_at'  => '2024-02-20T14:15:00Z',
    ],
    [
        'review_id'   => 3,
        'product_id'  => 99,
        'customer_id' => 102,
        'title'       => 'Not what I expected',
        'detail'      => 'Returns process was smooth but the product was not as described.',
        'rating'      => 2.0,
        'status'      => 'approved',
        'created_at'  => '2024-03-05T09:00:00Z',
    ],
];

$manager->bulkIndexReviews($reviews);
```

#### Searching: Bool Query (must / should / filter / must_not)

The `bool` query is the foundation of Elasticsearch queries. It combines sub-queries with boolean logic:

```json
{
  "query": {
    "bool": {
      "must":   [],     // AND — scored (affects relevance)
      "should": [],     // OR — scored
      "filter": [],     // AND — not scored (faster)
      "must_not": []    // NOT — not scored
    }
  }
}
```

- **`must`** clauses contribute to the relevance score.
- **`filter`** clauses are cached, do not score, and are significantly faster — use them for exact matches and ranges.
- **`should`** clauses are optional boosting clauses.
- **`must_not`** excludes documents.

**Query by product_id using bool filter:**

```php
<?php
public function searchReviewsByProduct(int $productId): array
{
    $response = $this->client->search([
        'index' => 'training_reviews',
        'body'  => [
            'query' => [
                'bool' => [
                    'filter' => [
                        ['term' => ['product_id' => $productId]],
                        ['term' => ['status' => 'approved']],
                    ],
                ],
            ],
            'sort' => [
                ['created_at' => 'desc'],
            ],
            'size' => 10,
        ],
    ]);

    return array_map(
        fn($hit) => $hit['_source'],
        $response['hits']['hits']
    );
}
```

#### Full-Text Search with `match` Query and `multi_match`

**`match` query** — for single-field full-text search:

```php
<?php
public function searchReviewsByText(string $query): array
{
    $response = $this->client->search([
        'index' => 'training_reviews',
        'body'  => [
            'query' => [
                'match' => [
                    'detail' => [
                        'query' => $query,
                        'operator' => 'or',
                        'minimum_should_match' => '50%',
                    ],
                ],
            ],
        ],
    ]);

    return array_map(
        fn($hit) => $hit['_source'],
        $response['hits']['hits']
    );
}
```

**`multi_match` query** — searches across multiple fields:

```php
<?php
public function multiFieldSearch(string $query): array
{
    $response = $this->client->search([
        'index' => 'training_reviews',
        'body'  => [
            'query' => [
                'multi_match' => [
                    'query'  => $query,
                    'fields' => ['title^2', 'detail'],
                    'type'   => 'best_fields',
                    'fuzziness' => 'AUTO',
                ],
            ],
        ],
    ]);

    return $response['hits']['hits'];
}
```

Here `title^2` means the title field is boosted twice as heavily as `detail`.

#### Filter-Only Queries for Faceted Navigation

Faceted navigation requires counting documents per filter value. Elasticsearch aggregations are the right tool:

```php
<?php
public function getReviewFacets(int $productId): array
{
    $response = $this->client->search([
        'index' => 'training_reviews',
        'body'  => [
            'query' => [
                'bool' => [
                    'filter' => [
                        ['term' => ['product_id' => $productId]],
                        ['term' => ['status' => 'approved']],
                    ],
                ],
            ],
            'aggs' => [
                'rating_breakdown' => [
                    'terms' => [
                        'field' => 'rating',
                        'size'  => 5,
                    ],
                ],
                'avg_rating' => [
                    'avg' => ['field' => 'rating'],
                ],
                'rating_histogram' => [
                    'histogram' => [
                        'field' => 'rating',
                        'interval' => 1,
                    ],
                ],
            ],
            'size' => 0,  // We only want aggregations, not hits
        ],
    ]);

    return [
        'total'        => $response['hits']['total']['value'],
        'avg_rating'   => $response['aggregations']['avg_rating']['value'],
        'by_rating'    => $response['aggregations']['rating_breakdown']['buckets'],
    ];
}
```

#### Autocomplete / Suggester: Completion Suggester Field Mapping

The completion suggester provides fast, type-ahead autocomplete:

```json
{
  "mappings": {
    "properties": {
      "title_suggest": {
        "type": "completion",
        "analyzer": "simple",
        "preserve_separators": true,
        "preserve_position_increments": true,
        "max_input_length": 50
      }
    }
  }
}
```

Index a document with completion data:

```php
<?php
public function indexWithSuggest(array $review): void
{
    $this->client->index([
        'index' => 'training_reviews',
        'id'    => (string) $review['review_id'],
        'body'  => [
            'review_id'   => $review['review_id'],
            'product_id'  => $review['product_id'],
            'title'       => $review['title'],
            'title_suggest' => [
                'input' => $this->generateSuggestions($review['title']),
            ],
            'status'      => $review['status'],
        ],
    ]);
}

private function generateSuggestions(string $title): array
{
    $words = preg_split('/\s+/', strtolower($title));
    $suggestions = [$title]; // always suggest the full title
    foreach ($words as $i => $word) {
        $suggestions[] = implode(' ', array_slice($words, $i));
    }
    return $suggestions;
}
```

Query suggestions:

```php
<?php
public function autocomplete(string $prefix): array
{
    $response = $this->client->search([
        'index' => 'training_reviews',
        'body'  => [
            'suggest' => [
                'title-autocomplete' => [
                    'prefix' => $prefix,
                    'completion' => [
                        'field' => 'title_suggest',
                        'size'  => 5,
                        'skip_duplicates' => true,
                    ],
                ],
            ],
        ],
    ]);

    return array_map(
        fn($option) => $option['text'],
        $response['suggest']['title-autocomplete'][0]['options']
    );
}
```

#### Search Query Optimization — Avoiding Common Elasticsearch Pitfalls

Magento's Elasticsearch integration is powerful but can produce slow queries if not designed carefully. These patterns are the most impactful for catalog search performance.

**1. Use `filter` context for exact matches (no scoring needed):**

```php
<?php
// WRONG: Using `must` for a term filter
$query = [
    'bool' => [
        'must' => [
            ['term' => ['status' => 'enabled']], // unnecessary scoring
        ],
    ],
];

// RIGHT: Use `filter` for exact matches — no scoring, cached
$query = [
    'bool' => [
        'filter' => [
            ['term' => ['status' => 'enabled']], // filter context
        ],
    ],
];
```

The `filter` context in Elasticsearch is:
- **Cached** by default (filter bitsets are stored and reused)
- **Not scored** (faster — no relevance calculation)
- **Better for constant values** (status, visibility, category_id)

**2. Avoid wildcard queries on analyzed text fields:**

```php
<?php
// WRONG: Wildcard on text field — forces full scan
$query = [
    'wildcard' => ['name' => '*phone*']  // Very slow at scale
];

// RIGHT: Use match query with fuzziness for typo tolerance
$query = [
    'match' => [
        'name' => [
            'query' => 'phone',
            'fuzziness' => 'AUTO',  // 0 for 1-2 chars, 1 for 3-5 chars, 2 for 6+ chars
        ],
    ],
];
```

**3. Pagination with `search_after` for deep pagination:**

```php
<?php
// WRONG for large offsets — ES must sort + rank ALL matching documents
$params = ['from' => 10000, 'size' => 10];

// RIGHT: Use search_after for deep pagination (stable sort required)
$params = [
    'body' => [
        'size' => 10,
        'sort' => [
            ['_id' => 'asc'],  // Must include sort on _id for tie-breaking
            // Pass last document's sort values as search_after
        ],
        'search_after' => ['10000', 'doc_id_10000'],
        'query' => ['bool' => ['filter' => [['term' => ['status' => 'enabled']]]]],
    ],
];
```

**4. Use `nested` queries for array fields:**

Products have nested objects (e.g., `custom_attributes` array). Queries on nested fields must use `nested`:

```json
{
  "query": {
    "nested": {
      "path": "custom_attributes",
      "query": {
        "bool": {
          "must": [
            { "term": { "custom_attributes.attribute_code": "color" }},
            { "term": { "custom_attributes.value": "black" }}
          ]
        }
      }
    }
  }
}
```

Failing to use `nested` when querying array fields returns unexpected results — Elasticsearch treats the array as a "contains all" match rather than an exact field=value match.

**5. Index sorting vs. query-time sorting:**

If you frequently sort by `price` or `created_at`, define these as index sorting at mapping time to avoid sort computation on every query:

```json
{
  "settings": {
    "index": {
      "sort.field": ["price", "created_at"],
      "sort.order": ["asc", "desc"]
    }
  }
}
```

This pre-sorts the index segments at write time rather than sorting at query time. Only effective for frequently-sorted, low-cardinality fields.

#### Synonyms and Stopwords Analyzer Configuration

Create a custom analyzer with synonyms:

```php
<?php
public function createIndexWithSynonyms(): void
{
    $this->client->indices()->create([
        'index' => 'training_reviews',
        'body'  => [
            'settings' => [
                'analysis' => [
                    'filter' => [
                        'review_synonyms' => [
                            'type' => 'synonym',
                            'synonyms' => [
                                'fast, quick, rapid',
                                'big, large, huge',
                                'cheap, inexpensive, affordable',
                            ],
                        ],
                    ],
                    'analyzer' => [
                        'review_analyzer' => [
                            'tokenizer' => 'standard',
                            'filter' => [
                                'lowercase',
                                'review_synonyms',
                                'stop',
                            ],
                        ],
                    ],
                ],
            ],
            'mappings' => [
                'properties' => [
                    'title'     => [
                        'type' => 'text',
                        'analyzer' => 'review_analyzer',
                    ],
                    'detail'    => [
                        'type' => 'text',
                        'analyzer' => 'review_analyzer',
                    ],
                ],
            ],
        ],
    ]);
}
```

Update synonyms at runtime without reindexing by reloading the synonym filter:

```bash
curl -X POST 'localhost:9200/training_reviews/_close'
curl -X PUT 'localhost:9200/training_reviews/_settings' -H 'Content-Type: application/json' \
  -d '{"analysis":{"filter":{"review_synonyms":{"synonyms":["fast, quick, rapid, speedy"]}}}}'
curl -X POST 'localhost:9200/training_reviews/_open'
```

---

### Topic 5: Advanced Redis — Beyond Basic Cache

#### Redis Connection: Single Instance vs Redis Cluster vs Sentinel

**Single instance** — One Redis process. Suitable for dev and small-scale staging. All data in one process. No automatic failover.

**Redis Cluster** — Automatic sharding across multiple nodes. Data is split across 16384 hash slots. Provides horizontal read/write scaling and fault tolerance. Not compatible with operations that span multiple keys atomically (e.g. `SUNIONSTORE` across slot boundaries). Use for: high-throughput, large-dataset production deployments.

**Redis Sentinel** — Watches master-replica pairs, performs automatic failover on master crash, and publishes information about the current master. Sentinel does **not** shard data — each master-replica pair holds the full dataset. Use for: HA without sharding complexity.

For a Magento production deployment on AWS/GCP, the standard recommendation is:
- **Redis Cluster** for the cache backend (high throughput, horizontal scaling)
- **Redis Sentinel** for sessions (HA with simpler semantics, smaller dataset per session)

#### Master-Slave Replication for Read Scaling

Redis replica nodes replicate from the master asynchronously. Read queries can be distributed to replicas, reducing load on the master:

```yaml
# docker-compose.yml
services:
  redis-master:
    image: redis:7-alpine
    container_name: redis-master
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis-master-data:/data

  redis-replica:
    image: redis:7-alpine
    container_name: redis-replica
    ports:
      - "6380:6379"
    command: >
      redis-server
      --replicaof redis-master 6379
      --replica-read-only yes
    depends_on:
      - redis-master
    volumes:
      - redis-replica-data:/data
```

In `env.php`, configure the cache frontend to route reads to the replica:

```php
'cache' => [
    'frontend' => [
        'default' => [
            'backend' => 'Magento\Framework\Cache\Backend\Redis',
            'backend_options' => [
                'server'            => 'redis-master',
                'port'              => 6379,
                'database'          => 0,
                'slave_server'      => 'redis-replica',
                'slave_port'        => 6380,
                'connect_timeout'   => 2.5,
                'read_timeout'      => 2.5,
                'timeout'            => 2.5,
                'retry_on_error'    => 1,
            ],
        ],
    ],
],
```

Verify replication is working:

```bash
docker exec redis-master redis-cli INFO replication
# role:master
# connected_slaves:1

docker exec redis-replica redis-cli INFO replication
# role:slave
# master_link_status:up
```

#### Redis Pipelining: Reducing Round-Trips in Batch Operations

Every Redis command involves a round-trip (RT) between the client and server. For 1000 operations, naive sequential calls mean 1000 RTTs. Pipelining bundles commands into a single TCP packet and receives all responses at once — one RT for 1000 commands.

```php
<?php
/** @var \Predis\Client $redis */
$redis = new \Predis\Client('tcp://localhost:6379');

$pipe = $redis->pipeline();
for ($i = 1; $i <= 1000; $i++) {
    $pipe->set("product:{$i}:views", 0);
    $pipe->incr("product:{$i}:views");
}
$responses = $pipe->execute();

// Responses is an array of 1000 individual responses
echo "Pipelined 1000 commands in one RT.\n";
```

Time comparison (rough numbers):

| Method | 1000 commands | Time |
|---|---|---|
| Sequential | 1000 RTTs | ~500–1000 ms |
| Pipelined | 1 RTT | ~5–15 ms |

Use pipelining for:
- Bulk cache warming on deployment
- Mass invalidation of related keys
- Any bulk write/read operation

#### Distributed Locks with Redis: `SETNX` Pattern and Redlock Algorithm

**Simple `SETNX` lock:**

```php
<?php
class SimpleRedisLock
{
    public function __construct(
        private readonly \Predis\Client $redis,
        private readonly string $ownerId,
    ) {}

    public function acquire(string $resource, int $ttlSeconds = 30): bool
    {
        $key = "lock:{$resource}";
        $acquired = $this->redis->set($key, $this->ownerId, 'EX', $ttlSeconds, 'NX');
        return $acquired !== null;
    }

    public function release(string $resource): bool
    {
        $key = "lock:{$resource}";
        // Only release if we own the lock (Lua atomic check-and-delete)
        $script = <<<'LUA'
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
        LUA;
        $result = $this->redis->eval($script, 1, $key, $this->ownerId);
        return $result === 1;
    }
}
```

**Redlock algorithm** — for production-critical locks, use the Redlock approach: acquire a majority lock across N independent Redis instances (typically N=3 or N=5) and only consider the lock acquired if you succeed on more than half the nodes. This protects against single-node failures.

```php
<?php
class RedLock
{
    private const CLOCK_DRIFT_FACTOR = 0.01;
    private const UNLOCK_SCRIPT = <<<'LUA'
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
    LUA;

    public function __construct(
        private readonly array $redisClients,
        private readonly int $quorum,
    ) {}

    public function lock(string $resource, int $ttlMs = 30000): ?string
    {
        $token = bin2hex(random_bytes(16));
        $ttl = (int) ($ttlMs / 1000);
        $drift = (int) ($ttlMs * self::CLOCK_DRIFT_FACTOR) + 2;
        $startTime = hrtime(true);

        // Try to acquire on all nodes
        $acquired = 0;
        foreach ($this->redisClients as $client) {
            if ($this->tryAcquire($client, $resource, $token, $ttl)) {
                $acquired++;
            }
        }

        // Check we got quorum and lock is still valid
        $elapsedMs = (hrtime(true) - $startTime) / 1_000_000;
        $validityTime = $ttlMs - $drift - $elapsedMs;

        if ($acquired >= $this->quorum && $validityTime > 0) {
            return $token;
        }

        // Failed — unlock all
        $this->unlockAll($resource, $token);
        return null;
    }

    private function tryAcquire(\Predis\Client $client, string $resource, string $token, int $ttl): bool
    {
        $key = "lock:{$resource}";
        $result = $client->set($key, $token, 'EX', $ttl, 'NX');
        return $result !== null;
    }

    private function unlockAll(string $resource, string $token): void
    {
        $key = "lock:{$resource}";
        foreach ($this->redisClients as $client) {
            $client->eval(self::UNLOCK_SCRIPT, 1, $key, $token);
        }
    }

    public function unlock(string $resource, string $token): void
    {
        $key = "lock:{$resource}";
        foreach ($this->redisClients as $client) {
            $client->eval(self::UNLOCK_SCRIPT, 1, $key, $token);
        }
    }
}
```

Usage for a "first customer to claim a promo" operation:

```php
<?php
// In a PromoClaimService
$lock = $this->redLock->lock("promo:{$promoId}", 5000);
if ($lock === null) {
    throw new \Exception('Another customer is processing this claim. Please try again.');
}

try {
    $promo = $this->promoRepository->getById($promoId);
    if (!$promo->hasClaimsRemaining()) {
        throw new \Exception('Sorry, this promo has been fully claimed.');
    }
    $promo->recordClaim($customerId);
    $this->promoRepository->save($promo);
} finally {
    $this->redLock->unlock("promo:{$promoId}", $lock);
}
```

#### Implementing Rate Limiting: Sliding Window vs Fixed Window

**Fixed window** — simplest: allow N requests per time window (e.g., per minute):

```php
<?php
class FixedWindowRateLimiter
{
    public function __construct(
        private readonly \Predis\Client $redis,
    ) {}

    public function isAllowed(string $identifier, int $limit, int $windowSeconds): bool
    {
        $key = "ratelimit:{$identifier}:" . (int)(time() / $windowSeconds);

        $count = (int) $this->redis->incr($key);
        if ($count === 1) {
            $this->redis->expire($key, $windowSeconds);
        }

        return $count <= $limit;
    }
}
```

Fixed window has a boundary burst issue: at 12:00:59 a client can make 100 requests, then at 12:01:01 another 100 — 200 requests in 2 seconds. **Sliding window** fixes this.

**Sliding window** — uses a sorted set with timestamps as scores:

```php
<?php
class SlidingWindowRateLimiter
{
    public function __construct(
        private readonly \Predis\Client $redis,
    ) {}

    public function isAllowed(string $identifier, int $limit, int $windowSeconds): bool
    {
        $key = "ratelimit:sw:{$identifier}";
        $now = microtime(true);
        $windowStart = $now - $windowSeconds;

        $pipe = $this->redis->pipeline();
        // Remove all entries older than window start
        $pipe->zremrangebyscore($key, '-inf', (string) $windowStart);
        // Count current entries
        $pipe->zcard($key);
        // Add current request
        $pipe->zadd($key, [$now . ':' . bin2hex(random_bytes(4)) => $now]);
        // Set TTL
        $pipe->expire($key, $windowSeconds + 1);
        $responses = $pipe->execute();

        $count = $responses[1]; // zcard result
        return ($count + 1) <= $limit;
    }
}
```

Use it:

```php
<?php
if (!$limiter->isAllowed('customer:42', 10, 60)) {
    http_response_code(429);
    header('Retry-After: 60');
    exit('Rate limit exceeded. Please wait 60 seconds.');
}
```

#### Redis Lua Scripting: Atomic Lock Release, Custom Counters

Lua scripts in Redis execute atomically — no other command runs during script execution. This is how you implement operations that would otherwise require WATCH/MULTI/EXEC.

**Atomic counter with floor:**

```lua
-- increment_with_floor.lua
local key = KEYS[1]
local increment = tonumber(ARGV[1])
local floor = tonumber(ARGV[2])

local current = tonumber(redis.call('GET', key) or '0')
local new_value = current + increment

if new_value < floor then
    new_value = floor
end

redis.call('SET', key, new_value)
return new_value
```

```php
<?php
$script = file_get_contents(__DIR__ . '/increment_with_floor.lua');
$result = $redis->eval($script, 1, 'inventory:sku:12345', -5, 0);
// atomically decrements but never goes below 0
```

**Atomic page view counter with daily reset:**

```php
<?php
public function recordPageView(string $pageKey): int
{
    $key = "pageviews:daily:{$pageKey}:" . date('Y-m-d');
    $count = $this->redis->incr($key);
    if ($count === 1) {
        $this->redis->expire($key, 86400 * 2); // 2 day TTL (covers day boundary)
    }
    return $count;
}
```

#### `cache_invalidate` with Redis TAG-Based Pattern Deletion

Magento's Redis cache backend supports tag-based invalidation. When you call `invalidate(['PRODUCT_1'])`, Redis deletes all cache keys whose `tag:` set contains that tag.

Verify the structure:

```bash
# Keys stored in Redis with Magento's backend
redis-cli KEYS "zc:k:training*" | head -5

# Tag sets (each tag's key set)
redis-cli KEYS "zc:ti:*" | head -5
# e.g. zc:ti:PRODUCT_1 -> {zc:k:training:product:1, zc:k:training:product:1:cache}

# Invalidate PRODUCT_1 tag
redis-cli --eval "redis.call('DEL', unpack(redis.call('SMEMBERS', KEYS[1])))" \
  "zc:ti:PRODUCT_1"
```

In Magento's `\Magento\Framework\Cache\Backend\Redis`, invalidation is handled natively when you call `$cache->invalidate(['TAG_NAME'])`.

---

### Topic 6: Redis — Session Clustering & Application-Level Session Management

#### Session Storage in Redis vs Database vs File

| Storage | Speed | Scalability | HA Support | Notes |
|---|---|---|---|---|
| Files (`var/session/`) | Fast for single server | Poor (shared filesystem required) | None | Default in dev; causes locking issues |
| Database | Slowest; DB is bottleneck | Moderate; requires shared DB | Good (shared DB) | Not recommended for >100 concurrent users |
| Redis | Fastest; in-memory | Excellent | Requires Sentinel or Cluster | Recommended for production |

Redis session storage in Magento is configured in `env.php`:

```php
'session' => [
    'save' => 'redis',
    'redis' => [
        'host'           => 'redis-session',
        'port'           => 6379,
        'database'       => 2,      // Use a separate DB from cache (DB 0)
        'password'       => null,
        'timeout'        => 2.5,
        'persistent_identifier' => '',
        'compression_threshold'  => 2048,
        'compression_library'     => 'gzip',
        'log_level'      => 1,
        'max_concurrency' => 10,
        'break_after_frontend' => 10,
        'break_after_adminhtml' => 5,
        'first_lifetime' => 600,
        'bot_first_lifetime' => 60,
        'bot_lifetime'   => 7200,
        'cookie_lifetime' => 86400,
        'cookie_path'    => '/',
        'cookie_domain'   => '',
        'serialize_handler' => 'php',
        'use_frontend_cookies' => true,
        'forever_hash_branch_tree' => true,
    ],
],
```

Set via CLI:

```bash
bin/magento config:set system/security/frontend_sessions 86400
bin/magento setup:config:set --session-save=redis \
  --session-save-redis-host=redis-session \
  --session-save-redis-port=6379 \
  --session-save-redis-db=2
```

#### Session Lifetime Configuration

Two key settings control session duration:

**`session.cookie_lifetime`** — how long the browser cookie lives (seconds). `0` means session cookie (deleted on browser close):

```php
'cookie_lifetime' => 86400, // 24 hours
```

**`session.save_lifetime`** (actually `cookie_lifetime` in Magento's config; `session.saveLifetime` is the PHP INI setting used as fallback):

```php
'session' => [
    'save' => 'redis',
    'cookie_lifetime' => 86400,
    'cookie_path'     => '/',
    'cookie_domain'   => '',
    'cookie_secure'   => true,   // HTTPS only in production
    'cookie_httponly' => true,
    'cookie_samesite' => 'Lax',
],
```

To force session refresh on each login (ignore existing cookie):

```bash
bin/magento config:set system/security/frontend_sessions 0
# 0 = session ends when browser closes
```

#### Session Locking Issues with Redis and How to Avoid Them

PHP sessions are locked for the duration of a script's execution to prevent concurrent writes from corrupting session data. By default, PHP uses `files` save handler which locks the session file. When switching to Redis without configuration, Redis does not lock sessions — concurrent requests from the same session race:

**Request A:** `session_start()` → `$_SESSION['cart'] = ['item1']` → runs for 5s
**Request B:** `session_start()` → `$_SESSION['cart'] = ['item2']` → runs for 2s
**Result:** Request B's cart item may be lost when Request A writes last.

**Fix 1:** Enable Redis session locking in Magento's Redis session config:

```php
'session' => [
    'redis' => [
        // ...
        'locking_enabled'     => true,
        'lock_retries'         => 300,
        'lock_wait_time'       => 3000,  // microseconds
        'break_after_frontend' => 10,    // seconds to wait before giving up
    ],
],
```

This enables `redis.locking` Redis Module (requires `phpredis` extension, not `predis`) or falls back to a Lua-based lock.

**Fix 2:** Reduce session lock hold time by avoiding heavy operations in the session-dependent part of your code. Split read-heavy and write-heavy operations.

**Fix 3:** Use different session cookies for frontend vs adminhtml to prevent admin operations from locking frontend sessions.

Verify locking is active:

```bash
redis-cli KEYS "PHPREDIS_SESSION:*" | wc -l
# After login + multiple page loads
redis-cli GET "$(redis-cli KEYS 'PHPREDIS_SESSION:*' | head -1)" | head -20
```

#### Implementing a Custom Session Handler for Multi-Node Deployments

For advanced use cases (e.g., sharing sessions between Magento and a separate Node.js microservice), implement `\SessionHandlerInterface`:

```php
<?php
declare(strict_types=1);
namespace Training\Performance\Session;

use Magento\Framework\Session\SessionManager;
use Magento\Framework\Session\SaveHandler\RedisSessionManager;

class SharedSessionHandler implements \SessionHandlerInterface
{
    public function __construct(
        private readonly RedisSessionManager $redisSession,
        private readonly string $sessionPrefix = 'shared_session:',
    ) {}

    public function open(string $savePath, string $sessionName): bool
    {
        return $this->redisSession->open($savePath, $sessionName);
    }

    public function read(string $sessionId): string
    {
        return $this->redisSession->read($this->prefix($sessionId));
    }

    public function write(string $sessionId, string $sessionData): bool
    {
        return $this->redisSession->write($this->prefix($sessionId), $sessionData);
    }

    public function destroy(string $sessionId): bool
    {
        return $this->redisSession->destroy($this->prefix($sessionId));
    }

    public function gc(int $maxLifetime): int|false
    {
        return $this->redisSession->gc($maxLifetime);
    }

    private function prefix(string $sessionId): string
    {
        return $this->sessionPrefix . $sessionId;
    }
}
```

Register in `etc/di.xml`:

```xml
<type name="Magento\Framework\Session\SessionManager">
    <arguments>
        <argument name="sessionHandler" xsi:type="object">
            Training\Performance\Session\SharedSessionHandler
        </argument>
    </arguments>
</type>
```

#### Redis SENTINEL Configuration for Automatic Failover

Sentinel monitors Redis masters and replicas, and redirects clients to the new master after a failover. Configuration requires at least 3 Sentinel processes (for quorum):

```yaml
# docker-compose.yml
services:
  redis-sentinel-1:
    image: redis:7-alpine
    container_name: redis-sentinel-1
    command: >
      redis-server
      --sentinel
      --sentinel monitor mymaster redis-master 6379 2
      --sentinel down-after-milliseconds mymaster 5000
      --sentinel failover-timeout mymaster 60000
      --sentinel announce-ip redis-sentinel-1
    ports:
      - "26379:26379"

  redis-sentinel-2:
    image: redis:7-alpine
    container_name: redis-sentinel-2
    command: >
      redis-server
      --sentinel
      --sentinel monitor mymaster redis-master 6379 2
      --sentinel down-after-milliseconds mymaster 5000
      --sentinel failover-timeout mymaster 60000
      --sentinel announce-ip redis-sentinel-2
    ports:
      - "26380:26379"

  redis-master:
    image: redis:7-alpine
    container_name: redis-master
    ports:
      - "6381:6379"
    command: redis-server --appendonly yes

  redis-replica:
    image: redis:7-alpine
    container_name: redis-replica
    ports:
      - "6382:6379"
    command: >
      redis-server
      --replicaof redis-master 6379
      --replica-read-only yes
    depends_on:
      - redis-master
```

In the application, use a Sentinel-aware client:

```php
<?php
// With phpredis:
$redis = new \Redis();
$redis->connect('redis-sentinel-1', 26379);
$master = $redis->sentinel('master', 'mymaster');
$redis->connect($master['ip'], $master['port']);
```

#### Session Data You Should Never Store

Store only what is strictly necessary for session continuity. Never store:

| Data | Reason |
|---|---|
| Passwords or hashes | Never needed in session after authentication |
| Payment card numbers / CVV | PCI-DSS violation |
| Full customer objects (with addresses) | Eavmodel objects are large; store IDs and load on demand |
| Sensitive API tokens (payment gateways) | Store in encrypted vault, not session |
| Admin privilege flags | Use Magento's authorization system, not session flags |
| Raw price or cost data | Not needed for session continuity |

What IS appropriate to store:
- Customer ID (`customer_id`)
- Store / website ID
- Cart item IDs (for quick retrieval, not full cart data)
- Persistent cart token
- Recently viewed product IDs (small dataset)

---

### Topic 7: Profiling — XHProf / Tideways Integration

#### Setting Up Tideways/XHProf for Production Profiling

Tideways is the actively maintained fork of the deprecated XHProf. It provides a PHP extension plus a UI for analyzing profiling data.

**Install the Tideways daemon (for real-time profiling):**

```yaml
# docker-compose.yml
services:
  tideways-daemon:
    image: tideways-daemon
    container_name: tideways-daemon
    ports:
      - "9137:9137"
    restart: unless-stopped
```

**Install the PHP extension:**

```bash
# In the PHP container
apt-get update && apt-get install -y php-tideways

# Or via PECL
pecl install tideways-xhprof
echo "extension=tideways_xhprof.so" > /usr/local/etc/php/conf.d/tideways.ini
```

For `phpredis`, install the daemon and extension:

```bash
# Download the Tideways daemon binary
curl -sL https://github.com/tideways/php-profiler-extension/releases/latest/download/tideways.x86_64 \
  -o /usr/local/bin/tideways-daemon
chmod +x /usr/local/bin/tideways-daemon

# Start daemon
tideways-daemon --host=0.0.0.0 --port=9137 &
```

**Configure Magento to use Tideways:**

```php
// app/etc/env.php
'dev' => [
    'profiler' => [
        'enabled' => true,
        'backend' => 'Tideways\\XHProf\\Profiler\\Profiler',
        'backend_options' => [
            'tideways_api_key'     => getenv('TIDEWAYS_API_KEY'),
            'tideways_service_name' => 'magento2-app',
            'collect' => [
                'memory' => true,
                'time' => true,
                'exceptions' => true,
                'mysql' => true,
                'redis' => true,
                'elasticsearch' => true,
            ],
        ],
    ],
],
```

Or via `di.xml` for always-on application-level profiling:

```xml
<type name="Magento\Framework\Profiler\Driver\Standard\Mapper">
    <arguments>
        <argument name="backendFactory"
                   xsi:type="string">Tideways\XHProf\Profiler\ProfilerFactory::create</argument>
    </arguments>
</type>
```

#### Auto-Instrumentation: MySQL Queries, Cache Calls, HTTP Requests

Tideways auto-instruments most PHP built-ins. For MySQL (PDO/MySQLi), enable via:

```bash
# In php.ini or Tideways config
tideways.sample_rate = 100        # Sample every request (use sampling in prod)
tideways.auto_prepend_library =
```

For manual instrumentation of custom code:

```php
<?php
$tideways = new \Tideways\Profiler();
$tideways->start();

// ... your code ...

$tideways->leave('my_custom_operation');

// Manual span for Elasticsearch calls
$tideways->enterSection('elasticsearch_search');
$results = $elasticsearchClient->search([...]);
$tideways->leaveSection('elasticsearch_search');
```

In Tideways UI or the open-source [xhprof.io](https://github.com/perftools/xhprof) viewer, each function call becomes a node in the callgraph with metrics: wall time, CPU time, memory delta, and I/O time.

#### Interpreting Callgraphs: Wall Time vs CPU Time

**Wall time (Exclusive)** — Time spent in this function alone, excluding time spent in child functions. This is the most important metric for optimization: if `getProduct()` takes 200ms wall time and 195ms is spent in its children, the function itself has only 5ms of overhead.

**Wall time (Inclusive)** — Total time from entry to exit of this function, including all children.

**CPU time** — Actual CPU cycles spent in the function (not waiting on I/O). On modern servers with fast disks and Redis, CPU time is usually much less than wall time for Magento code because most time is spent in I/O (MySQL, Redis, HTTP).

**Memory delta** — Net change in memory consumption during this function's execution. Negative values mean the function freed memory (e.g., result set destructor ran).

```
Function: Magento\Framework\DB\Adapter\Pdo\Mysql::query
Inclusive: 847ms
Exclusive: 12ms   ← this is the real cost of the adapter itself
Children:  835ms   ← the query execution + result fetching
Calls:     1
```

The 12ms exclusive means the PDO adapter overhead is minimal. The 835ms in children is the actual MySQL query time. Target `exclusive` time for optimization, not `inclusive`.

#### Finding Hot Spots: The 80/20 Rule in Magento Callgraphs

The 80/20 rule: 80% of execution time is caused by 20% of functions. These are the "hot spots." Look for:

1. **Functions with high exclusive wall time** — These are your bottlenecks. Typically: `Pdo_Mysql::query`, `Elasticsearch::search`, `Redis::get`, `Plugin::after*` methods
2. **Functions called many times in a single request** — N+1 query patterns. A function called 1000 times with 0.5ms each = 500ms total
3. **Functions with high memory allocation** — Eav model loading without collection pooling

In Tideways, sort the function list by **Exclusive Wall Time** descending. The top 5 entries are your hot spots. Ignore functions below 5ms exclusive time unless they are called thousands of times.

#### Profiling a Slow API Endpoint: Step-by-Step

**Step 1:** Enable profiling for the specific request via a header or query parameter:

```php
// In a front controller plugin
public function beforeDispatch(
    \Magento\Framework\App\FrontControllerInterface $subject,
    \Magento\Framework\App\RequestInterface $request
): void {
    if ($request->getParam('profile')) {
        \Tideways\Profiler::start(['api_endpoint' => $request->getPathInfo()]);
    }
}
```

**Step 2:** Make the request with profiling enabled:

```bash
time curl -s "http://localhost/rest/V1/products?searchCriteria=foo&profile=1" | jq '.'
```

**Step 3:** Collect the profile:

```php
<?php
// Save profile at the end of the request or in an after-plugin
$tideways = new \Tideways\Profiler();
$tideways->stop();

// Save to file
$profileData = $tideways->serialize();
file_put_contents('/tmp/profile_' . uniqid() . '.json', $profileData);

// Or send to Tideways cloud
$tideways->sendToTidewaysService($apiKey, $serviceName);
```

**Step 4:** Analyze the profile. Look for:
- `PDO::query` calls taking >100ms each (missing index)
- `Magento\Eav\Model\Entity\Abstract::load` called in a loop (N+1)
- `Plugin::afterGet` intercepting every product with a DB call
- `curl_exec` calls not using async or connection pooling

#### Memory Profiling: Peak Memory Consumption, Memory Leaks

Tideways records memory allocations. In the callgraph, look for functions with large **memory delta** (positive = high allocation, negative = freeing memory):

```
Function: Magento\Framework\Model\ResourceModel\Db\Collection\AbstractCollection::_loadAll
Inclusive Memory: +2.4MB
Exclusive Memory: +128KB
```

A typical culprit: loading full EAV collections inside a loop without `clear()`:

```php
<?php
// WRONG — each iteration keeps all previous models in memory
foreach ($productIds as $id) {
    $product = $this->productRepository->getById($id);
    // ... $product is kept in memory
}
// After loop: 10,000 Product objects loaded simultaneously

// RIGHT — clear collection between batches
foreach (array_chunk($productIds, 100) as $batch) {
    foreach ($batch as $id) {
        $collection->addFieldToFilter('entity_id', $id);
    }
    $collection->load();
    foreach ($collection as $product) {
        // ...
    }
    $collection->clear(); // free memory
}
```

#### Setting Profiling Thresholds That Alert on Deployment

Integrate Tideways data with your CI/CD pipeline to alert when a deployment degrades performance:

---

### Topic 8: Database Connection Pooling & Horizontal Scaling

#### Why Database Connection Pooling Matters in Magento

Magento's database layer uses PDO Mysql, which by default opens a new TCP connection for every query. In a high-traffic environment with multiple PHP-FPM workers, this creates hundreds of short-lived connections that compete for the MySQL `max_connections` limit.

**The problem in numbers:**

| Concurrent PHP Workers | Queries per Page Load | Connection Pattern | Total Connections |
|---|---|---|---|
| 10 | 50 | No pooling | 500 (all short-lived) |
| 10 | 50 | Pooled (size=10) | 10 (persistent, reused) |

Connection pooling maintains a **fixed pool of persistent connections** that are reused across requests. When a PHP worker needs a connection, it borrows one from the pool; when done, it returns it instead of closing it.

Magento 2.4 supports connection pooling via `dbsock` and `persistent` connection options in `env.php`. For production, the recommended pattern is:

```php
<?php
// app/etc/env.php
'db' => [
    'connection' => [
        'default' => [
            'host' => 'db-primary',
            'port' => 3306,
            'dbname' => 'magento',
            'username' => 'magento_user',
            'password' => 'secret',
            'model' => 'mysql4',
            'engine' => 'innodb',
            'initStatements' => 'SET NAMES utf8mb4',
            // Persistent connection — shares a single connection per PHP worker
            'persistent' => false, // Use false with connection pooling; true with RDS Proxy
            // Use connection pool via RDS Proxy or ProxySQL
            'charset' => 'utf8mb4',
            'active' => '1',
        ],
        'indexer' => [
            'host' => 'db-replica',
            'port' => 3306,
            'dbname' => 'magento',
            'username' => 'magento_readonly',
            'password' => 'secret',
            'model' => 'mysql4',
            'charset' => 'utf8mb4',
            'active' => '1',
        ],
    ],
],
```

#### MySQL Connection Pooling via ProxySQL

**ProxySQL** is a middleware that sits between PHP and MySQL, providing connection pooling, query routing, and failover. For Magento, the typical architecture is:

```
PHP-FPM Workers → ProxySQL (connection pool) → MySQL Primary (writes)
                                                    ↓
                                              MySQL Replicas (reads)
```

**Why ProxySQL over RDS Proxy?**

RDS Proxy (AWS) is managed and simpler, but ProxySQL gives you:
- Multi-vendor compatibility (works with any MySQL/MariaDB)
- Full query control (route `SELECT` to replica, `INSERT/UPDATE/DELETE` to primary)
- Query caching
- Per-user connection limits

```ini
# docker-compose.yml for ProxySQL
services:
  proxysql:
    image: proxysql/proxysql:2.5.2
    container_name: proxysql
    ports:
      - "6032:6032"  # Admin interface
      - "6033:6033"  # MySQL interface (PHP connects here)
    volumes:
      - ./proxysql.cnf:/etc/proxysql.cnf:ro
    restart: unless-stopped
```

**ProxySQL configuration for Magento (read/write split):**

```ini
# /etc/proxysql.cnf (simplified)
databases:
  - hostname: "db-primary"
    port: 3306
    username: "magento_user"
    password: "secret"
    max_connections: 200

  - hostname: "db-replica"
    port: 3306
    username: "magento_readonly"
    password: "secret"
    max_connections: 200

# Route rules: SELECT goes to replica, everything else to primary
mysql_query_rules:
  - rule_id: 1
    match_pattern: "^SELECT.*FROM (sales_|catalog_)"
    destination_hostgroup: 1  # replica
    apply: 1
  - rule_id: 2
    match_pattern: "^INSERT|^UPDATE|^DELETE"
    destination_hostgroup: 0  # primary
    apply: 1
```

PHP connects to ProxySQL on port 6033. ProxySQL routes queries intelligently without PHP knowing about the split.

#### Async Indexing — Decoupling Indexing from Request Life

Standard Magento indexing is **synchronous**: when a product is saved, the `catalogsearch_fulltext` indexer runs immediately, blocking the request. For catalogs with millions of SKUs, this causes unacceptable latency spikes.

**Async indexing** (introduced in Magento 2.4 via the `design_config_grid` indexer pattern) separates indexing from the save operation:

```bash
# Enable async indexing for specific indexers
bin/magento indexer:set-mode realtime catalogsearch_fulltext
bin/magento indexer:set-mode schedule design_config_grid
```

When in `schedule` mode, indexers run on a cron schedule instead of on save. In `realtime` mode, they run immediately.

For a production catalog, the recommended pattern is:

| Indexer | Mode | Reason |
|---------|------|--------|
| `catalog_product_price` | Schedule (cron) | Can take 30+ min on large catalogs |
| `catalogsearch_fulltext` | Schedule (cron) | ES indexing should not block saves |
| `inventory` | Realtime | Stock status must be immediately accurate |
| `design_config_grid` | Schedule | Grid rebuild on every config change is wasteful |

Configure the cron schedule in `crontab.xml`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Cron:etc/crontab.xsd">
    <group id="index">
        <!-- Run every minute; let the indexer process queue -->
        <job name="indexer_update_all" instance="Magento\Indexer\Cron\UpdateMview"
             method="execute">
            <schedule>* * * * *</schedule>
        </job>
    </group>
</config>
```

The `UpdateMview` cron job processes the `indexer_state` table's `update_required` flag, running only the indexers that need updating since last run.

#### Cache Warmup Strategies — Avoiding Cold Cache Stampede

After a full cache flush (deployment, emergency clear), serving live traffic against a cold cache causes a **cache stampede** — every request misses the cache and hammers the backend simultaneously.

**Cache warmup strategies:**

**1. HTTP-based warmup (recommended for Varnish):**

```bash
#!/bin/bash
# warmup.sh — run AFTER cache flush, before traffic returns
PRODUCT_URLS=$(mysql -h db -u magento -psecret magento \
  -N -e "SELECT CONCAT('/catalog/product/view/id/', entity_id) 
          FROM catalog_product_entity LIMIT 500")

CATEGORY_URLS=$(mysql -h db -u magento -psecret magento \
  -N -e "SELECT CONCAT('/catalog/category/view/id/', entity_id) 
          FROM catalog_category_entity WHERE level = 2 LIMIT 100")

CMS_PAGES="home about-us contact"

for url in $PRODUCT_URLS $CATEGORY_URLS $CMS_PAGES; do
  curl -s -o /dev/null -w "%{http_code} %{url_effective}\n" \
    --cookie "X-Magento-Cache-Debug=1" \
    "https://store.example.com$url" &
done
wait
```

**2. Redis-based programmatic warmup:**

```php
<?php
// In a CLI command: bin/magento training:cache:warmup
declare(strict_types=1);
namespace Training\Performance\Command;

use Magento\Framework\App\State;
use Magento\Catalog\Model\Product\CategoryFactory;
use Magento\Framework\Cache\Frontend\Decorator\TagScope;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class CacheWarmup extends Command
{
    public function __construct(
        private readonly State $appState,
        private readonly \Magento\Catalog\Api\CategoryRepositoryInterface $categoryRepo,
    ) {
        parent::__construct();
    }

    protected function configure(): void
    {
        $this->setName('training:cache:warmup')
             ->setDescription('Warm up FPC cache for critical pages');
    }

    public function execute(InputInterface $input, OutputInterface $output): int
    {
        $this->appState->setAreaCode(\Magento\Framework\App\Area::AREA_CRONTAB);

        $output->writeln('Warming homepage...');
        $this->warmUrl('/');

        $categoryIds = [2, 3, 4, 5]; // Top-level categories
        foreach ($categoryIds as $categoryId) {
            $category = $this->categoryRepo->get($categoryId);
            $output->writeln("Warming category {$category->getName()}...");
            $this->warmUrl('/catalog/category/view/id/' . $categoryId);
        }

        $output->writeln('Warmup complete.');
        return 0;
    }

    private function warmUrl(string $path): void
    {
        $client = new \Magento\Framework\HTTP\Client\Curl();
        $client->get('http://localhost' . $path);
        // Response is cached by Varnish; subsequent live requests get HIT
    }
}
```

**3. Deployment warmup via `post_deploy` hook:**

```yaml
# app/etc/config.yaml (Magento Cloud)
hooks:
  post_deploy: |
    php bin/magento training:cache:warmup
    set -e
```

This runs AFTER the deployment finishes but BEFORE traffic switches to the new version, ensuring the cache is warm when users arrive.

#### New Relic APM Integration for Magento

New Relic provides application performance monitoring beyond what Tideways offers — distributed tracing, error grouping, and infrastructure correlation. The `newrelic` PHP agent is installed as an extension and auto-instruments Magento.

**Install New Relic agent:**

```bash
# On the server (or in Dockerfile)
curl -L https://download.newrelic.com/php_agent/release/newrelic-php5-10.12.0.490-linux.tar.gz \
  | tar -C /tmp -xzf -
cd /tmp/newrelic-php5-*/ Daemon
NR_INSTALL_KEY=YOUR_LICENSE_KEY NR_APPLICATION_NAME=magento2 \
  ./newrelic-install install

# Enable the agent
echo "extension=newrelic.so" >> /usr/local/etc/php/conf.d/newrelic.ini
echo "newrelic.appname=magento2-production" >> /usr/local/etc/php/conf.d/newrelic.ini
```

**Magento-specific configuration in `env.php`:**

```php
<?php
// app/etc/env.php — New Relic configuration
'newrelic' => [
    'reporting' => [
        'nrapikey' => 'YOUR_NR_ADMIN_API_KEY',
        'transaction_tracer' => [
            'enabled' => true,
            'threshold' => 0.5,  // ms — trace requests > 500μs
            'detail' => \NewRelic\TransactionTraceAdapter::DETAIL_ALL,
        ],
        'error_collector' => [
            'enabled' => true,
            'ignore_exceptions' => [
                \Magento\Framework\Exception\LocalizedException::class,
            ],
        ],
        'utm_params' => [
            'name' => 'request_uri',
            'scope' => 'request.params.*',
        ],
    ],
],
```

**Key New Relic dashboards for Magento:**

| Dashboard | What It Shows |
|-----------|--------------|
| **Apdex Score by Page Type** | User satisfaction score per route |
| **Slow Transactions** | Top 20 slowest endpoints by P95 |
| **Database by Caller** | Which module/controller calls the most queries |
| **External Services** | Payment gateway, shipping API response times |
| **Error Rate** | Exceptions grouped by type and stack trace |
| **Throughput** | Requests per minute by page type |

> **Pro Tip:** Use New Relic's `TransactionNaming` to override the default controller/action naming with more meaningful business names:

```php
<?php
// In a front controller plugin
public function beforeDispatch(
    \Magento\Framework\App\FrontControllerInterface $subject,
    \Magento\Framework\App\RequestInterface $request
): void {
    if (function_exists('newrelic_name_transaction')) {
        newrelic_name_transaction(
            $request->getModuleName() . '/' .
            $request.getControllerName() . '/' .
            $request.getActionName()
        );
    }
}
```

This gives you named transactions like `catalog/product/view` instead of `FrontController::dispatch`.

#### Horizontal Scaling — Read Replicas and Multiple PHP-FPM Nodes

Horizontal scaling means adding more PHP-FPM nodes behind a load balancer, with a shared cache (Redis) and a database read replica for read-heavy operations.

**Architecture for 3-node Magento deployment:**

```
                    ┌─────────────┐
                    │ Load Balancer│
                    │ (nginx / AWS ALB) │
                    └──────┬──────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ PHP-FPM  │    │ PHP-FPM  │    │ PHP-FPM  │
    │ Node 1   │    │ Node 2   │    │ Node 3   │
    └────┬─────┘    └────┬─────┘    └────┬─────┘
         │               │               │
         └───────────────┼───────────────┘
                         ▼
              ┌──────────────────┐
              │  Shared Redis     │
              │  (cache + session)│
              └──────────────────┘
                         │
              ┌──────────┴──────────┐
              ▼                      ▼
       ┌─────────────┐       ┌─────────────┐
       │ MySQL Primary│       │ MySQL Replica│
       │ (writes)     │       │ (reads)      │
       └─────────────┘       └─────────────┘
              │
              ▼
       ┌─────────────────┐
       │  Elasticsearch │
       │  (search index) │
       └─────────────────┘
```

**Magento's read/write split configuration:**

```php
<?php
// app/etc/env.php — Read replica configuration
'db' => [
    'connection' => [
        'default' => [
            'host' => 'mysql-primary',
            'port' => 3306,
            'dbname' => 'magento',
            'username' => 'magento_user',
            'password' => 'secret',
            'active' => '1',
        ],
        'checkout' => [
            'host' => 'mysql-replica',
            'port' => 3306,
            'dbname' => 'magento',
            'username' => 'magento_readonly',
            'password' => 'secret',
            'active' => '1',
        ],
    ],
],
'resource' => [
    'default_setup' => [
        'connection' => 'default',
    ],
    'checkout' => [
        'connection' => 'checkout',  # Use replica for checkout reads
    ],
],
```

> **Note:** Magento does not automatically route all reads to the replica. The `checkout` resource connection is used for specific operations. Proper read/write splitting at the ProxySQL level (as described above) is more reliable than Magento's per-resource configuration.

---

### Topic 15: Edge-Side Includes (ESI), Private Content, and Cache Stratification

**Why This Matters:** Full-page cache serves public content. But what about customer-specific content (cart count, wishlist, price) that can't be cached publicly? ESI and private content rendering solve this without sacrificing cache performance.

**Cache Stratification — Four Layers:**

| Layer | Scope | Cache Type | Invalidated When |
|-------|-------|-----------|-----------------|
| **Full-Page Cache (Public)** | Public pages (category, CMS) | Varnish/Redis | Product/category save, config change |
| **Block Cache (Semi-Private)** | Custom blocks with cache identity | Redis | Entity save or explicit tag |
| **Private Content (Per-Customer)** | Cart, wishlist, customer data | Browser +hole punch | Customer action (add to cart, login) |
| **No Cache** | Real-time data (inventory, pricing) | None | Always fetched fresh |

**Edge-Side Includes (ESI) — When to Use:**

ESI allows Varnish to assemble a page from multiple cached fragments, some cached publicly, some fetched from backend:

```
Varnish (public cache)
  ├── Header [cached 1h]
  ├── Navigation [cached 1h]  
  ├── Product Content [cached 1h]
  │     └── <esi:src="/blocks/cart-count" />  ← ESI: fetched from backend per request
  │     └── <esi:src="/blocks/wishlist-count" /> ← ESI: fetched per customer
  └── Footer [cached 1h]
```

**Magento ESI Support:**

Magento automatically generates ESI tags for blocks that:
- Use `$block->getCacheLifetime()` 
- Are rendered in a cached page context
- Need customer-specific data

Example: A `CustomerWelcome` block with ESI:
```xml
<block name="customer.welcome" template="Magento_Theme::header/welcome.phtml"
       cacheable="true" cacheable ttl="3600">
    <!-- In layout XML -->
</block>
```

The FPC will include this block with a cache key that varies by customer. But if the block has customer-specific data, Magento uses a "hole punch" — the cached page includes an ESI URL that fetches the private block on each request.

**Private Content Rendering (Customer-Specific Blocks):**

Private content is rendered client-side (browser) via Magento's `private_content_version` and Ajax-based rendering:

```javascript
// Magento's private content request
require(['Magento_Customer/js/customer-data'], function(customerData) {
    var cart = customerData.get('cart');
    cart.subscribe(function() {
        // Re-render when cart data changes
    });
});
```

The flow:
```
Cached Page Load (served from Varnish)
  ↓
Browser receives HTML with customer-specific placeholder divs
  ↓
Browser executes require(['Magento_Customer/js/customer-data'], ...)
  ↓
AJAX requests to /customer/section/load with customer cookie
  ↓
Server returns JSON with cart, wishlist, reviews, etc.
  ↓
JS updates placeholder divs with actual data
```

**Key Cache Configuration Patterns:**

**1. Public block with long TTL:**
```xml
<block name="category.breadcrumbs" 
       template="Magento_Catalog::product/breadcrumbs.phtml"
       cacheable="true"
       cache_ttl="86400"/>
```

**2. Private block (customer-specific, no FPC):**
```xml
<block name="catalog.compare.list" 
       template="Magento_Catalog::product/compare/list.phtml"
       class="Magento\Catalog\Block\Product\Compare\ListCompare"
       cacheable="false"/>
```
`cacheable="false"` cascades to the parent container — the entire page won't be cached if this block is on it.

**3. Hole-punch pattern (cached page + dynamic block):**
```php
// In block class — use cacheable="true" with a customer-dependent cache key
protected function _construct()
{
    parent::_construct();
    $this->setCacheTags(['CUSTOMER_ID_' . $this->_getCustomerId()]);
}
```

**TTL Strategy by Content Type:**

| Content Type | Public Cache (TTL) | ESI | Private Content |
|-------------|-------------------|-----|----------------|
| Category page | 1 hour | No | No |
| Product page | 1 hour | No | No |
| CMS page | 24 hours | No | No |
| Cart sidebar | — | Yes | Yes |
| Wishlist | — | Yes | Yes |
| Customer welcome | — | No | Yes |
| Recently viewed | — | No | Yes |
| Price (real-time) | — | No | No |

**Diagnosing Cache Problems:**

```bash
# Check if page is cacheable (look for X-Magento-Cache-Debug header)
curl -I https://yourstore.com/category/page
# HIT = cached, MISS = not in cache, EXPIRED = was expired

# Check ESI URL on cached pages
curl -s https://yourstore.com/category/page | grep esi

# Varnish admin (if available)
varnishlog -g request -q 'ReqURL ~ "/category/"'
```

**Common Cache Mistakes:**

1. ❌ **cacheable="false" on customer-facing page** — entire page won't be cached (homepage, category page) — causes massive performance degradation
2. ❌ **No TTL on cacheable blocks** — uses Magento default (3600s) but wrong for content type
3. ❌ **Cache tags missing on custom blocks** — blocks never invalidate when related entity changes
4. ❌ **Private content without proper invalidation** — customer sees stale cart after adding/removing items

**Pro Tips:**
- Never put `cacheable="false"` on CMS pages, category pages, or product pages — only on truly dynamic blocks
- Private content section handling in controllers: return `$resultLayout->getLayout()->getChild('content')->setData('cacheable', false)`
- For personalized content that updates in real-time (inventory, live pricing), use JavaScript Ajax calls — don't try to cache it

---

## Additional Common Mistakes to Avoid (Topic 14)

11. **Not setting `first_byte_timeout` high enough for slow category pages** — Varnish will return an error instead of waiting. Set `first_byte_timeout = 300s` for catalog-heavy pages.

12. **Using `cache:flush` in automated scripts** — Always prefer tagged invalidation. `cache:flush` causes a full cache clear and creates a thundering herd on the next deployment.

13. **Enabling Tideways in production without sampling** — Profiling every single request adds 5-15% overhead. Always use request sampling (e.g., profile 1 in 100 requests) in production.

14. **Not configuring Elasticsearch 7.x for Magento 2.4 compatibility** — ES 8.x has breaking changes. Always use 7.17.x (or apply the official Magento patch if you need ES 8.x features).

15. **Using the same Redis database for both cache and sessions** — If you accidentally run `FLUSHDB` on the cache DB, you lose all sessions. Always use separate DB numbers (DB 0 for cache, DB 2 for sessions).

---

```yaml
# .gitlab-ci.yml or GitHub Actions
script:
  - |
    # Profiling baseline from previous release
    BASELINE=$(cat baseline_profile.json | jq -r '.metrics.exclusive_wall_time.total')
    
    # Run load test against new deployment
    k6 run --out json=k6_results.json load-test.js
    
    # Extract Tideways profile from new deployment
    NEW_PROFILE=$(curl -s "http://app.internal/api/profile/latest")
    
    # Compare wall time per endpoint
    DEGRADATION=$(echo "$NEW_PROFILE" | jq -r '.endpoints[].exclusive_wall_time.total')
    
    if (( $(echo "$DEGRADATION > $BASELINE * 1.2" | bc -l) )); then
      echo "PERFORMANCE DEGRADATION DETECTED: +20% vs baseline"
      exit 1
    fi
```

Set up alerting rules in Tideways Cloud (or a self-hostedxhprof.io instance):

| Threshold | Action |
|---|---|
| Endpoint wall time > 2× baseline | Block deploy |
| MySQL query count > 100 per request | Warning |
| Memory per request > 256 MB | Alert |
| P95 latency > 5s | Page on-call |

---

## Reference Exercises

### Exercise 1: Configure Varnish in Docker, Enable FPC, Verify Cache Headers

**Objective:** Set up Varnish, point Magento to it, and verify full-page caching is working.

**Steps:**
1. Add the Varnish service to your `docker-compose.yml` (use the snippet from Topic 1).
2. Write a `varnish/default.vcl` file based on the VCL from Topic 1.
3. Update `app/etc/env.php` to set `caching_application = 2` and point `http_cache_hosts`.
4. Clear the full-page cache: `bin/magento cache:flush full_page`
5. Warm the cache: `curl -s http://localhost/cms-page-url/ > /dev/null`
6. Check headers on first request (expect `X-Cache: MISS`) and second request (expect `X-Cache: HIT`):

```bash
curl -sI http://localhost/cms-page-url/ | grep -E "^(X-Cache|X-Magento-Tags|Cache-Control)"
```

**Deliverable:** Output showing `X-Cache: MISS` on first curl and `X-Cache: HIT` on second curl with `X-Magento-Tags` populated.

---

### Exercise 2: Add Cache Tag Invalidation for a Custom Module

**Objective:** When a product review is approved, invalidate the product's FPC cache tag.

**Steps:**
1. Create module scaffold: `Training/Performance`
2. Create an observer on `review_save_after` in `etc/events.xml`
3. In the observer, get the product ID from the review entity
4. Call `$this->cache->invalidate(['PRODUCT_' . $productId])`
5. Programatically approve a review and observe the FPC invalidation

```php
<?php
// In a test script
$review = $this->reviewRepository->getById(42);
$review->setStatus(\Magento\Review\Model\Review::STATUS_APPROVED);
$this->reviewRepository->save($review);

// In Varnish log, you should see a PURGE request for PRODUCT_42
```

**Deliverable:** Observer class + event registration + log output showing `PURGE` for `PRODUCT_{id}`.

---

### Exercise 3: Install Elasticsearch 8.x in Docker, Create Custom Index for Reviews

**Objective:** Install Elasticsearch (7.17.x for Magento compatibility), create the `training_reviews` index, index 3 documents, and query by `product_id`.

**Steps:**
1. Add Elasticsearch service to `docker-compose.yml` (7.17.16 as per Topic 3).
2. Verify ES is up: `curl http://localhost:9200`
3. Create the index using the PHP code from Topic 4 (`ReviewIndexManager::createIndex()`).
4. Index 3 review documents via `bulkIndexReviews()`.
5. Search using the bool filter query:

```php
$results = $reviewSearchService->searchReviewsByProduct(42);
// Should return 2 reviews (review_id 1 and 2)
```

6. Verify with curl:

```bash
curl -s -X GET 'localhost:9200/training_reviews/_search?pretty' \
  -H 'Content-Type: application/json' \
  -d '{"query":{"bool":{"filter":[{"term":{"product_id":42}}]}}}'
```

**Deliverable:** Index created, 3 documents indexed, query output showing 2 documents for `product_id=42`.

---

### Exercise 4: Implement a Redis Distributed Lock

**Objective:** Implement a concurrent "claim" operation where only the first process wins.

**Steps:**
1. Install `predis/predis`: `composer require predis/predis`
2. Implement `SimpleRedisLock` from Topic 5.
3. Create a test script that simulates 10 concurrent processes trying to claim the same promo code:

```php
<?php
// Run this with: php -r "require 'claim_test.php';"
// from 10 different terminal tabs simultaneously
$lock = new SimpleRedisLock($redis, 'process_' . getmypid());

if (!$lock->acquire("promo:FLASH_SALE_001", 30)) {
    echo "Sorry, another customer is currently claiming this promo.\n";
    exit(1);
}

echo "Lock acquired! Processing claim...\n";
sleep(5); // simulate work
echo "Claim processed successfully.\n";
$lock->release("promo:FLASH_SALE_001");
```

4. Observe that only one process prints "Lock acquired!" and the rest print the failure message.

**Deliverable:** Log output showing exactly 1 success and 9 failures among concurrent invocations.

---

### Exercise 5: Configure Redis Session Storage, Verify Sessions in Redis

**Objective:** Configure Magento to store sessions in Redis and confirm they appear in Redis.

**Steps:**
1. Add a Redis service to `docker-compose.yml` (separate from the cache Redis).
2. Configure `env.php` with `session.save = redis` and `session.redis.*` settings.
3. Run `bin/magento cache:flush` and `bin/magento setup:config:set` to apply.
4. Start the PHP built-in server: `bin/magento serve:development`
5. Open the storefront and log in as a customer.
6. Check Redis for session keys:

```bash
redis-cli KEYS "PHPREDIS_SESSION:*"
# Expected: 1+ keys matching the session cookie

redis-cli GET "PHPREDIS_SESSION:<session_id>" | head -20
# Expected: Serialized PHP session data containing customer_id, etc.
```

**Deliverable:** `redis-cli KEYS` output showing session keys, `GET` output showing customer data in session.

---

### Exercise 6 (Optional): Use Tideways/XHProf to Profile a Slow Indexer Call

**Objective:** Profile `bin/magento indexer:reindex` and identify the top 5 functions by exclusive wall time.

**Steps:**
1. Install Tideways extension and daemon per Topic 7.
2. Configure `app/etc/env.php` with profiler settings.
3. Run the indexer with profiling: `XHPROF_ENABLED=1 bin/magento indexer:reindex catalogsearch_fulltext`
4. Find the saved profile file: `find /tmp -name '*.xhprof' -mmin -5`
5. Render the profile with `xhprof_html` or `tideways` CLI:

```bash
php $(find vendor -name 'xhprof_html' -o -name 'xhprof.php' | head -1) \
  --source=tideways \
  --report=main \
  /tmp/profile_latest.json
```

6. Identify the top 5 functions by **exclusive wall time** in the callgraph.
7. For each function, note: name, exclusive time, likely cause, and suggested fix.

**Deliverable:** Top 5 functions table with columns: `Function`, `Exclusive (ms)`, `Calls`, `Likely Cause`, `Fix`.

---

## Reading List

1. **Varnish Cache Documentation** — https://varnish-cache.org/docs/
2. **Magento 2.4 Developer Documentation: Configure Varnish** — https://experienceleague.adobe.com/docs/commerce-oper/configuration/cache/configure-varnish.html
3. **Elasticsearch: The Definitive Guide** (Clinton Gormley & Zachary Tong) — https://www.elastic.co/guide/en/elasticsearch/guide/current/
4. **Elasticsearch PHP Client v8 Documentation** — https://www.elastic.co/guide/en/elasticsearch/client/php-api/current/index.html
5. **Redis Documentation** — https://redis.io/docs/
6. **Predis Client Documentation** — https://github.com/predis/predis
7. **Tideways PHP Profiler Documentation** — https://tideways.io/profiler/docs
8. **Magento Performance Best Practices** (Adobe Commerce documentation) — https://experienceleague.adobe.com/docs/commerce-operations/performance-best-practices/overview.html
9. **Martin Fowler — Event Sourcing / Cache Stampede** — https://martinfowler.com/
10. **High Performance Browser Networking** (Ilya Grigorik) — https://hpbn.co/ — chapters on HTTP caching and transport layer

---

## Edge Cases & Troubleshooting

| Issue | Symptom | Cause | Fix |
|---|---|---|---|
| Varnish returns `MISS` on every request | Headers show `X-Cache: MISS` even on repeated requests | Cache-Control: private on the page | Ensure VCL sets `uncacheable` only for explicitly private pages |
| Varnish serves stale content after product update | Price changed but old price shown | Tag invalidation not reaching Varnish | Verify `http_cache_hosts` in env.php; check Magento `cache_invalidate` observer fires |
| Elasticsearch index not created | `index not found` error | Elasticsearch not reachable | Check `ES_JAVA_OPTS`, `max_map_count` sysctl (`sysctl -w vm.max_map_count=262144`) |
| Redis sessions not appearing | `KEYS *` returns nothing after login | Wrong Redis database number | Check `session.redis.database` = 2 (separate from cache DB 0) |
| Redis lock not releasing | Lock never expires, resource permanently locked | Lock holder crashed without TTL | Ensure TTL is always set on lock acquisition; use Redlock with auto-expiry |
| Tideways no data | Profiler started but no profile saved | Extension not loaded or daemon not running | Check `php -m | grep tideways`; check daemon `curl http://localhost:9137/ping` |
| PHP session lock blocking | Concurrent requests from same user hang | Session locking enabled but session handler is `files` | Switch session save handler to Redis; tune `break_after_frontend` |
| Varnish `vcl.load` fails | `varnishd: Child failed to start` | Syntax error in VCL or port conflict | Run `varnishd -C -f /etc/varnish/default.vcl` to validate syntax |
| Elasticsearch bulk index partially fails | `errors: true` in bulk response | Mapping conflict or document ID collision | Check `items[].index.error.reason` in response; update mapping |
| `magento indexer:reindex` very slow | Takes 30+ minutes for `catalogsearch_fulltext` | Elasticsearch not available, fallback to MySQL | Check ES connection; re-run `bin/magento config:set catalog/search/engine elasticsearch7` |

---

## Common Mistakes to Avoid

1. **Setting `cacheable="false"` on many blocks** — This does not make blocks dynamic; it marks every containing page uncacheable. Use proper cache key design instead.

2. **Not separating Redis cache DB from session DB** — Using the same Redis database for both cache and sessions risks `FLUSHDB` wiping all user sessions. Use `database` values 0 and 2 respectively.

3. **Running `cache:flush full_page` in production automated flows** — Always prefer tagged invalidation. Full flush causes a thundering herd. Reserve `cache:flush` for manual emergency clears only.

4. **Using Elasticsearch 8.x with Magento 2.4 without patches** — Magento 2.4 was tested against ES 7.x. Using ES 8.x without the official compatibility patch will cause silent search failures.

5. **Skipping `vm.max_map_count` on Linux before starting Elasticsearch** — ES will refuse to start with a confusing "max virtual memory areas" error. Pre-configure it: `echo 'vm.max_map_count=262144' >> /etc/sysctl.conf && sysctl -p`.

6. **Not using pipelining for bulk Redis operations** — Running 10,000 individual `SET` commands in a loop is a network I/O disaster. Always pipeline batch operations.

7. **Storing large objects in PHP sessions** — `$_SESSION['customer'] = $customerModel` serializes the entire EAV model including all attribute values. Store only the ID and load on demand.

8. **Not setting TTL on Redis locks** — A lock acquired without a TTL (`expire`) will permanently block the resource if the holder crashes. Always use `SET key value EX ttl NX`.

9. **Profiling in production without sampling** — Full-request profiling on every single request creates massive overhead. Use request sampling (e.g., profile 1 in every 100 requests) or enable only for specific endpoints via a header.

10. **Assuming Varnish caches POST requests** — By default Varnish does not cache POST (it passes through). Never rely on Varnish caching for form submissions; they must always reach the backend.

---

*Magento 2 Backend Developer Course — Topic 14+15 | Advanced Performance*
