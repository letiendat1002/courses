---
title: "25 - Caching (FPC/Varnish/Redis)"
description: "Deep dive into Magento 2.4.8 caching: Full Page Cache, Varnish configuration, Redis cache backend, cache types, and cache management."
tags: magento2, caching, fpc, varnish, redis, full-page-cache, cache-types, cache-management
rank: 6
pathways: [magento2-deep-dive]
---

# Caching (FPC/Varnish/Redis)

## 1. Magento Cache Architecture Overview

Magento 2.4.8 implements a **multi-layer caching architecture** designed to minimize database load and deliver fast page responses. Understanding this architecture is critical for both performance optimization and debugging cache-related issues.

### Multi-Layer Cache Flow

```
Request → Varnish (if enabled) → Application (FPC) → Redis → Database
   │              │                    │              │         │
   └── HTTP ──────┴── Full Page Cache ──┴──────────────┴─────────┘
```

**Layer breakdown:**

| Layer | Component | Purpose |
|-------|-----------|---------|
| 1st | Varnish (HTTP cache) | Caches complete HTTP responses at the reverse proxy layer |
| 2nd | Full Page Cache (FPC) | Application-level page caching via `Magento\Framework\App\PageCache` |
| 3rd | Cache Backend (Redis) | Distributed cache storage for all cache types |
| 4th | Database/File | Origin data; only hit when cache miss occurs |

### Cache Types XML Configuration

Cache types are declared in `app/etc/cache_types.xml`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Cache/etc/cache_types.xsd">
    <type name="config" instance="Magento\Framework\App\Cache\Type\Config" />
    <type name="layout" instance="Magento\Framework\App\Cache\Type\Layout" />
    <type name="block_html" instance="Magento\Framework\App\Cache\Type\Block" />
    <type name="collections" instance="Magento\Framework\App\Cache\Type\Collection" />
    <type name="reflection" instance="Magento\Framework\App\Cache\Type\Reflection" />
    <type name="db_ddl" instance="Magento\Framework\App\Cache\Type\Db_Ddl" />
    <type name="eav" instance="Magento\Framework\App\Cache\Type\Eav" />
    <type name="customer_notification" instance="Magento\Framework\App\Cache\Type\CustomerNotification" />
    <type name="full_page" instance="Magento\Framework\App\Cache\Type\Page" />
    <type name="config_integration" instance="Magento\Framework\App\Cache\Type\Integration" />
    <type name="config_integration_api" instance="Magento\Framework\App\Cache\Type\Integration\Api" />
    <type name="translate" instance="Magento\Framework\App\Cache\Type\Translate" />
    <type name="vertex" instance="Magento\Vertex\Cache\Type" />
</config>
```

Each `<type>` element registers a cache type with:
- `name`: Unique identifier for the cache type
- `instance`: Fully qualified class name implementing the cache type

---

## 2. Cache Types

Magento 2.4.8 ships with **13 built-in cache types**, each serving a specific purpose:

### Cache Types Reference Table

| Cache Type | Class | Purpose | Invalidation Triggers |
|------------|-------|---------|----------------------|
| `config` | `Magento\Framework\App\Cache\Type\Config` | Stores configuration XML (modules, environment) | Config save, `setup:upgrade` |
| `layout` | `Magento\Framework\App\Cache\Type\Layout` | Compiled page layouts (XML + PHP) | Layout XML changes, cache clean |
| `block_html` | `Magento\Framework\App\Cache\Type\Block` | Block output HTML fragments | Block cache refresh, layout update |
| `collections` | `Magento\Framework\App\Cache\Type\Collection` | Data collection metadata | Collection rebuild |
| `reflection` | `Magento\Framework\App\Cache\Type\Reflection` | API reflection data | Class changes |
| `db_ddl` | `Magento\Framework\App\Cache\Type\Db_Ddl` | Database DDL metadata | Schema changes, `setup:upgrade` |
| `eav` | `Magento\Framework\App\Cache\Type\Eav` | EAV entity type/attribute metadata | EAV attribute changes |
| `customer_notification` | `Magento\Framework\App\Cache\Type\CustomerNotification` | Customer notifications | Manual or event-based |
| `full_page` | `Magento\Framework\App\Cache\Type\Page` | Entire HTML page (FPC) | Page cache invalidation |
| `config_integration` | `Magento\Framework\App\Cache\Type\Integration` | Integration configs | Integration API changes |
| `config_integration_api` | `Magento\Framework\App\Cache\Type\Integration\Api` | Integration API configs | Integration API changes |
| `translate` | `Magento\Framework\App\Cache\Type\Translate` | Inline translation data | Translation file changes |
| `vertex` | `Magento\Vertex\Cache\Type` | Vertex tax calculation cache | Vertex-specific invalidation |

### Cache Type Priority for Performance

For performance tuning, prioritize these cache types:

```php
// High-impact caches (significant performance gain when cached)
'config'           => true,   // Avoids XML re-parsing
'full_page'        => true,   // Eliminates page render time
'layout'           => true,   // Avoids layout compilation
'block_html'       => true,   // Reduces block rendering

// Medium-impact caches
'eav'              => true,   // Reduces metadata queries
'collections'      => true,   // speeds up collection queries
'db_ddl'           => true,   // Avoids DDL queries

// Lower-impact caches
'reflection'       => true,   // Minor API overhead
'translate'        => true,   // Translation lookup
```

---

## 3. Cache Management CLI

Magento provides comprehensive CLI commands for cache management via `bin/magento cache`:

### Core Cache Commands

**Enable/Disable All Caches:**

```bash
# Disable all cache types
bin/magento cache:disable

# Enable all cache types
bin/magento cache:enable

# Example output when disabling:
# Disabled cache types:
# config, layout, block_html, collections, reflection, db_ddl, eav, customer_notification, full_page, config_integration, config_integration_api, translate, vertex
```

**Flush vs Clean — Critical Distinction:**

| Command | Behavior | Use Case |
|---------|----------|----------|
| `cache:clean` | Deletes enabled cache types only | Refresh stale data while preserving clean caches |
| `cache:flush` | Deletes ALL cache storage (all types + Redis/var/cache) | Complete cache reset,解决 corruption |

```bash
# Clean specific cache types (only enabled types are cleaned)
bin/magento cache:clean config layout

# Flush ALL caches (including disabled ones)
bin/magento cache:flush

# Clean page cache only
bin/magento cache:clean full_page
```

**Status and Information:**

```bash
# View cache status for all types
bin/magento cache:status

# Example output:
# Current status:
# Config:          1 (enabled)
# Layout:          1 (enabled)
# Block HTML:      1 (enabled)
# Collections:     1 (enabled)
# Reflection:      1 (enabled)
# DDL:             1 (enabled)
# EAV:             1 (enabled)
# Customer Note:   1 (enabled)
# Page Cache:      1 (enabled)
# Integration:     1 (enabled)
# Integration API: 1 (enabled)
# Translate:       1 (enabled)
# Vertex:          1 (enabled)
```

**Enable/Disable Individual Cache Types:**

```bash
# Disable specific cache type
bin/magento cache:type:disable full_page

# Enable specific cache type
bin/magento cache:type:enable full_page

# Disable multiple types
bin/magento cache:type:disable block_html collections

# List all available cache types
bin/magento cache:type:list
```

### When to Use Each Command

```bash
# After modifying configuration XML → clean config
bin/magento cache:clean config

# After deploying new theme or modifying layouts → clean layout + block_html
bin/magento cache:clean layout block_html

# After product import → clean full_page + block_html
bin/magento cache:clean full_page block_html

# After module disable/enable → full flush
bin/magento cache:flush

# After database schema changes → clean db_ddl + config
bin/magento cache:clean db_ddl config
```

---

## 4. Full Page Cache (FPC)

Full Page Cache is Magento's application-level mechanism for caching rendered HTML pages, dramatically reducing server-side rendering overhead.

### Built-in FPC vs Varnish

| Feature | Built-in FPC | Varnish |
|---------|-------------|---------|
| Storage | Redis or file | Memory (RAM) |
| Scope | Single server | Distributed/multi-server |
| Speed | Fast | Extremely fast |
| Cache invalidation | Tag-based | Purge rules |
| ESI support | No | Yes |
| Grace mode | No | Yes |
| Recommended for | Development/single-server | Production/multi-server |

### FPC Classes Architecture

**Main FPC Classes:**

```php
// Application Page Cache Front Controller
// vendor/magento/framework/App/PageCache/

namespace Magento\Framework\App\PageCache;

class Cache implements \Magento\Framework\Cache\Frontend\Decorator\Tagspecific
{
    /**
     * Cache identifier prefix
     */
    const CACHE_ID_PREFIX = 'PAGE_';

    /**
     * {@inheritdoc}
     */
    public function getBackend()
    {
        return $this->_cache->getBackend();
    }

    /**
     * {@inheritdoc}
     */
    public function getFrontend()
    {
        return $this->_cache->getFrontend();
    }

    /**
     * Load data from cache
     *
     * @param string $identifier
     * @return bool
     */
    public function load($identifier)
    {
        return $this->_cache->load($this->getCacheId($identifier));
    }

    /**
     * Save data to cache
     *
     * @param string $data
     * @param string $identifier
     * @param string[] $tags
     * @param int $lifeTime
     * @return bool
     */
    public function save($data, $identifier, $tags = [], $lifeTime = null)
    {
        return $this->_cache->save(
            $data,
            $this->getCacheId($identifier),
            $this->joinTags($tags),
            $lifeTime ?: self::DEFAULT_LIFETIME
        );
    }
}
```

### How FPC Works

**Cache Key Generation:**

The cache key for a page is generated from the request:

```php
// vendor/magento/framework/App/PageCache/CacheIdentifier.php

namespace Magento\Framework\App\PageCache;

class CacheIdentifier
{
    // Uses request URL + store ID + design package as cache key
    public function getCacheKey()
    {
        return $this->cacheKeyProtocol
            . $this->generateCacheKey();
    }

    protected function generateCacheKey()
    {
        return md5(
            $this->url->getCurrentUrl() .
            $this->_storeManager->getStore()->getId() .
            $this->_appState->getAreaCode()
        );
    }
}
```

**Page Identifier (Cache Tags):**

Pages are tagged with identifiers for targeted invalidation:

```php
// vendor/magento/framework/Stdlib/CookieManager.php
// Sets X-Magento-Tags header containing product/category/cms tags
```

**Important Headers:**

| Header | Purpose | Set By |
|--------|---------|--------|
| `X-Magento-Cache-Id` | Unique page identifier | Magento FPC |
| `X-Magento-Tags` | Comma-separated cache tags | Magento FPC |
| `X-Cache-Id` | Varnish cache identifier | Varnish |
| `X-Varnish-Tags` | Varnish cache tags | Varnish |

### Hole Punching for Dynamic Content

Dynamic content within cached pages uses the "hole punching" pattern:

**Block with `cacheable="false"`:**

```xml
<!-- In layout XML -->
<referenceBlock name="catalog.compare.sidebar">
    <block name="catalog.product.comparison.list"
           template="Magento_Catalog::product/compare/list.phtml"
           cacheable="false" />
</referenceBlock>
```

**Private block content (non-cacheable):**

```php
// In Block class
namespace Magento\Catalog\Block\Product;

class View
{
    /**
     * Non-cacheable block - product info changes per user
     */
    protected function getCacheLifetime()
    {
        return null; // This block won't be cached
    }
}
```

### Private Content (Customer-Specific)

For customer-specific dynamic content, Magento uses **customer sections**:

```javascript
// define/frontend requirejs-config.js
var config = {
    "map": {
        "*": {
            "Magento_Customer/js/customer-data": {
                "sectionLoadUrl": window.checkout.customer.getCustomerDataUrl,
                "cookieLifeTime": "3600",
                "updateSessionUrl": window.checkout.customer.updateSessionUrl
            }
        }
    }
};
```

---

## 5. Varnish Configuration

Varnish is Magento's recommended reverse proxy and Full Page Cache. It operates at the HTTP layer, caching entire responses in memory.

### Varnish Architecture with Magento

```
Client Request
      │
      ▼
┌─────────────┐
│   Varnish   │  ← Caches complete HTTP responses
│   (Port 80) │
└─────────────┘
      │
      │ (cache miss or uncacheable request)
      ▼
┌─────────────┐
│   Magento   │  ← Application (FPC disabled or not)
│  (Port 8080)│
└─────────────┘
```

### default.vcl Structure

**Complete production-ready Varnish 4.0 VCL for Magento 2.4.8:**

```vcl
vcl 4.0;

import std;
import directors;

# Backend definition
backend default {
    .host = "127.0.0.1";
    .port = "8080";
    .first_byte_timeout = 600s;
    .between_bytes_timeout = 600s;
    .connect_timeout = 600s;
    .probe = {
        .url = "/health_check.php";
        .timeout = 5s;
        .interval = 10s;
        .window = 5;
        .threshold = 3;
    }
}

# ACL for ban requests (cache invalidation)
acl internal {
    "localhost";
    "127.0.0.1";
}

sub vcl_init {
    return (ok);
}

sub vcl_recv {
    # Normalize common headers
    if (req.http.X-Forwarded-Proto == "https") {
        set req.http.X-Forwarded-Proto = "https";
    }

    # Do not cache POST requests
    if (req.method == "POST") {
        return (pass);
    }

    # Do not cache requests with query strings (except for XHR/API)
    if (req.queryString && !req.url ~ "^(.*\?)(q=|query=|p=|search=|filter=)") {
        return (pass);
    }

    # Do not cache checkout/cart/customer pages
    if (req.url ~ "^(.*)(checkout|cart|customer|wishlist|newsletter)(.*)$") {
        return (pass);
    }

    # Do not cache POST requests to checkout
    if (req.method == "POST" && req.url ~ "/checkout/") {
        return (pass);
    }

    # Remove Google Analytics parameters for better cache hit ratio
    if (req.url ~ "\?utm_source=" || req.url ~ "\?utm_medium=" || req.url ~ "\?utm_campaign=") {
        set req.url = regsub(req.url, "\?utm_source=([^&]*)", "");
        set req.url = regsub(req.url, "&utm_medium=([^&]*)", "");
        set req.url = regsub(req.url, "&utm_campaign=([^&]*)", "");
    }

    # Do not cache logged in customers (private content)
    if (req.http.cookie ~ "customer=") {
        return (pass);
    }

    # Do not cache private content sections
    if (req.http.cookie ~ "section_data=") {
        return (pass);
    }

    # Vary header handling for proper cache differentiation
    if (req.http.Accept-Encoding) {
        if (req.url ~ "\.(jpg|jpeg|jpe|gif|png|swf|ico|pdf|mpeg|mpg|avi|zip|gz|gzip|bz2|tar|tgz|mp3|ogg|woff2?|eot)$") {
            unset req.http.Accept-Encoding;
        } elsif (req.http.Accept-Encoding ~ "gzip") {
            set req.http.Accept-Encoding = "gzip";
        } elsif (req.http.Accept-Encoding ~ "deflate") {
            set req.http.Accept-Encoding = "deflate";
        } else {
            unset req.http.Accept-Encoding;
        }
    }

    return (hash);
}

sub vcl_hash {
    hash_data(req.url);

    # Include cookie in hash for personalized pages
    if (req.http.cookie) {
        hash_data(req.http.cookie);
    }

    # Include X-Magento-Tags for proper tag-based invalidation
    if (req.http.X-Magento-Tags) {
        hash_data(req.http.X-Magento-Tags);
    }

    # Include store code for multi-store setups
    if (req.http.X-store) {
        hash_data(req.http.X-store);
    }

    return (lookup);
}

sub vcl_backend_response {
    # Set grace mode for stale-while-revalidate
    set beresp.grace = 24h;

    # Do not cache error responses (or cache briefly)
    if (beresp.status >= 500) {
        set beresp.ttl = 30s;
        set beresp.uncacheable = true;
        return (deliver);
    }

    # Do not cache private content responses
    if (beresp.http.X-Magento-Cache-Control ~ "private") {
        set beresp.uncacheable = true;
        set beresp.ttl = 0s;
        return (deliver);
    }

    # Cache static assets for long period
    if (bereq.url ~ "\.(jpg|jpeg|jpe|gif|png|swf|ico|pdf|mpeg|mpg|avi|zip|gz|gzip|bz2|tar|tgz|mp3|ogg|woff2?|eot)$") {
        set beresp.http.Cache-Control = "public, max-age=31536000";
        set beresp.ttl = 31536000s;
        set beresp.http.Pragma = "cache";
    }

    # Do not cache CMS blocks with user-specific content
    if (beresp.http.X-Magento-Cache-Debug) {
        set beresp.http.X-Cache = "HIT";
        set beresp.http.X-Cache-Hits = obj.hits;
    }

    return (deliver);
}

sub vcl_deliver {
    # Remove internal headers before sending to client
    unset resp.http.X-Varnish;
    unset resp.http.Via;
    unset resp.http.X-Forwarded-For;
    unset resp.http.X-Proxy-Original-URL;

    # Add debug header in developer mode
    if (resp.http.X-Magento-Cache-Debug) {
        set resp.http.X-Cache = "HIT";
    } else {
        set resp.http.X-Cache = "MISS";
    }

    # Add custom cache headers
    set resp.http.X-Frame-Options = "SAMEORIGIN";

    return (deliver);
}

sub vcl_backend_error {
    # Serve stale content if backend fails (grace mode)
    if (beresp.status >= 500 && beresp.ttl < 0s) {
        set beresp.ttl = 0s;
        set beresp.uncacheable = true;
        return (deliver);
    }
}

sub vcl_synth {
    if (resp.status == 503) {
        # Try grace mode during backend failures
        set resp.http Retry-After = "5";
    }
}
```

### Grace Mode Configuration

Grace mode serves stale content while fetching fresh content:

```vcl
sub vcl_backend_response {
    # Enable grace mode - serve stale content for up to 24h while revalidating
    set beresp.grace = 24h;

    # Note: In Varnish 4, beresp.backend.healthy is NOT available in vcl_backend_response.
    # Use req.backend.healthy in vcl_recv (as shown below) with proper backend probe configuration.
    # For saint mode handling of backend failures, see the Saint Mode section below.
    if (!beresp.ttl > 0s) {
        set beresp.grace = 0s;
    }
}

sub vcl_recv {
    # Request grace mode during revalidation
    # Note: req.backend.healthy is only valid in vcl_recv when a backend probe is configured
    if (req.backend.healthy) {
        set req.grace = 0s;
    } else {
        set req.grace = 24h;
    }
}
```

### Saint Mode (Backend Failure Handling)

Saint mode blacklists unhealthy backends temporarily:

```vcl
sub vcl_init {
    # Initialize saint mode - blacklist backend for 60 seconds after 3 failures
    new saint = directors.failure_domain();
    saint.add_backend(default, 3, 60s);
}

sub vcl_backend_response {
    # If backend returns error, use saint mode
    if (beresp.status >= 500) {
        return (abandon);
    }
}
```

### Cache TTL Settings in VCL

```vcl
sub vcl_backend_response {
    # Default TTL for cached pages
    set beresp.ttl = 86400s;  # 24 hours

    # Override TTL based on URL patterns
    if (bereq.url ~ "^/catalog/category/view/id=") {
        set beresp.ttl = 172800s;  # 48 hours for category pages
    }

    if (bereq.url ~ "^/catalog/product/view/id=") {
        set beresp.ttl = 86400s;  # 24 hours for product pages
    }

    if (bereq.url ~ "^/cms/") {
        set beresp.ttl = 604800s;  # 7 days for CMS pages
    }
}
```

### Health Check Configuration

Create `pub/health_check.php` for Varnish health probes:

```php
<?php
/**
 * Magento health check endpoint for Varnish
 *
 * @category    Magento
 * @package     Magento_Backend
 */
declare(strict_types=1);

require __DIR__ . '/../app/bootstrap.php';

$params = $_SERVER;
$objectManager = \Magento\Framework\App\ObjectManager::getInstance();

/** @var \Magento\Framework\App\State $state */
$state = $objectManager->get(\Magento\Framework\App\State::class);

try {
    $state->setAreaCode(\Magento\Framework\App\Area::AREA_FRONTEND);

    /** @var \Magento\Framework\App\Response\Http $response */
    $response = $objectManager->get(\Magento\Framework\App\Response\Http::class);
    $response->setCode(200);
    $response->setBody('OK');
    $response->sendResponse();
} catch (\Exception $e) {
    http_response_code(503);
    echo "Service Unavailable: " . $e->getMessage();
}
```

### HTTPS Handling with X-Forwarded-Proto

```vcl
sub vcl_recv {
    # Trust X-Forwarded-Proto header from load balancer
    if (req.http.X-Forwarded-Proto == "https") {
        set req.http.X-Forwarded-Proto = "https";
    } elsif (req.http.X-Forwarded-Proto == "http") {
        set req.http.X-Forwarded-Proto = "http";
    }
}

sub vcl_backend_response {
    # Set proper cache-control for HTTPS
    if (bereq.http.X-Forwarded-Proto == "https") {
        set beresp.http.X-Forwarded-Proto = "https";
    }
}
```

---

## 6. Redis Cache Backend

Redis provides a fast, distributed cache backend for Magento's cache infrastructure.

### Configuration in env.php

```php
<?php
// app/etc/env.php

return [
    // ... other config sections ...

    'cache' => [
        'frontend' => [
            // Default cache (config, layout, blocks, etc.)
            'default' => [
                'backend' => \Magento\Framework\Cache\Backend\Redis::class,
                'backend_options' => [
                    'server' => '127.0.0.1',
                    'port' => 6379,
                    'database' => '0',
                    'password' => '',
                    'compress_data' => true,
                    'compression_lib' => 'gzip',
                    'preload_keys' => [
                        'Magento\Framework\App\Bootstrap::CACHE_DIR::CONFIG::' => 1,
                        'Magento\Framework\App\Bootstrap::CACHE_DIR::LAYOUT::' => 1,
                        'Magento\Framework\App\Bootstrap::CACHE_DIR::BLOCK_HTML::' => 1,
                    ],
                ],
                'id_prefix' => 'magento2_',
            ],

            // Page cache (FPC) - separate Redis database
            'page_cache' => [
                'backend' => \Magento\Framework\Cache\Backend\Redis::class,
                'backend_options' => [
                    'server' => '127.0.0.1',
                    'port' => 6379,
                    'database' => '1',
                    'password' => '',
                    'compress_data' => false,  // FPC data is already compressed by Varnish
                    'compression_lib' => 'gzip',
                    'use_async_write' => true,
                    'slaves' => [
                        [
                            'server' => '127.0.0.1',
                            'port' => 6380,
                            'database' => '1',
                            'weight' => 1,
                        ],
                    ],
                ],
                'id_prefix' => 'magento2_',
            ],
        ],

        'allow_parallel_generation' => true,
    ],
];
```

### Redis Connection Verification

```php
<?php
// Verify Redis connection from CLI
$redis = new Redis();
$redis->connect('127.0.0.1', 6379);
echo "Redis PING: " . $redis->ping() . "\n";

// Check cache keys
$keys = $redis->keys('magento2_*');
echo "Cache keys count: " . count($keys) . "\n";

// Check memory usage
$info = $redis->info('memory');
echo "Used memory: " . $info['used_memory_human'] . "\n";
```

### Redis Cache Backend Class

**Key implementation details:**

```php
// vendor/magento/framework/Cache/Backend/Redis.php

namespace Magento\Framework\Cache\Backend;

class Redis implements \Zend_Cache_Backend, \Zend_Cache_Backend_Interface
{
    /**
     * Lua script for atomic cache cleaning by tags
     */
    protected $_luaCleanByTags = <<<LUA
local function valToString(val)
    if type(val) == "string" then
        return val
    elseif type(val) == "number" then
        return tostring(val)
    end
    return ""
end

local function getAllKeys(pattern)
    local keys = {}
    local cursor = "0"
    repeat
        local result = redis.call("SCAN", cursor, "MATCH", pattern, "COUNT", 1000)
        cursor = result[1]
        for _, key in ipairs(result[2]) do
            table.insert(keys, key)
        end
    until cursor == "0"
    return keys
end

local allKeys = getAllKeys(...)
for i, key in ipairs(allKeys) do
    redis.call("DEL", key)
end
return #allKeys
LUA;

    /**
     * {@inheritdoc}
     */
    public function clean($mode = 'all', $tags = [])
    {
        if ($mode == \Zend_Cache::CLEANING_MODE_ALL) {
            return $this->_redis->flushdb();
        }

        if ($mode == \Zend_Cache::CLEANING_MODE_OLD) {
            return $this->cleanOld();
        }

        if ($mode == \Zend_Cache::CLEANING_MODE_MATCHING_ANY_TAG ||
            $mode == \Zend_Cache::CLEANING_MODE_MATCHING_ALL_TAGS
        ) {
            return $this->cleanByTags($tags, $mode == \Zend_Cache::CLEANING_MODE_MATCHING_ANY_TAG);
        }

        return false;
    }
}
```

---

## 7. Cache Tags

Cache tags enable **targeted cache invalidation**, ensuring that related cached data is invalidated together without clearing the entire cache.

### How Cache Tags Work

**Tag Structure:**

```
Cache Entry:
  Key: "PAGE_magento2_product_123"
  Value: "<html>...</html>"
  Tags: ["P_123", "C_456", "STORE_1"]
```

**Invalidation by Tag:**

```php
// When product 123 is updated:
$cache->clean([
// Invalidate all pages containing product 123
    'P_123',
// Invalidate all pages in category 456 (if product moved)
    'C_456',
// Invalidate all pages for store 1
    'STORE_1'
]);
```

### TagSpecific Decorator

The `Tagspecific` decorator adds tag-aware functionality to any cache backend:

```php
// vendor/magento/framework/Cache/Frontend/Decorator/Tagspecific.php

namespace Magento\Framework\Cache\Frontend\Decorator;

class Tagspecific implements \Magento\Framework\Cache\Frontend\Decorator\Tagspecific
{
    /**
     * @var \Magento\Framework\Cache\Frontend\Decorator\Tagspecific
     */
    protected $_frontend;

    /**
     * @param \Magento\Framework\Cache\Frontend\Decorator\Tagspecific $frontend
     */
    public function __construct(\Magento\Framework\Cache\Frontend\Decorator\Tagspecific $frontend)
    {
        $this->_frontend = $frontend;
    }

    /**
     * {@inheritdoc}
     */
    public function save($data, $id, $tags = [], $lifetime = null)
    {
        return $this->_frontend->save($data, $id, $this->joinTags($tags), $lifetime);
    }

    /**
     * {@inheritdoc}
     */
    public function load($id)
    {
        return $this->_frontend->load($id);
    }

    /**
     * {@inheritdoc}
     */
    public function remove($id)
    {
        return $this->_frontend->remove($id);
    }

    /**
     * {@inheritdoc}
     */
    public function clean($tags = [], $mode = \Zend_Cache::CLEANING_MODE_MATCHING_ANY_TAG)
    {
        return $this->_frontend->clean($this->joinTags($tags), $mode);
    }

    /**
     * Join tags with cache type prefix
     *
     * @param string[] $tags
     * @return string[]
     */
    protected function joinTags($tags)
    {
        if (empty($tags)) {
            return $tags;
        }

        $prefix = $this->_frontend->getOption('tag_prefix') ?: '';
        $joinedTags = [];

        foreach ($tags as $tag) {
            $joinedTags[] = $prefix . $tag;
        }

        return $joinedTags;
    }
}
```

### Adding Tags to Custom Cache Entries

```php
<?php
// In a custom module's Block or Model

namespace Vendor\Module\Block;

class ProductList extends \Magento\Framework\View\Element\Template
{
    protected function getCacheTags(): array
    {
        return [
            'P_' . $this->getProduct()->getId(),
            'C_' . $this->getCategory()->getId(),
            'STORE_' . $this->_storeManager->getStore()->getId(),
        ];
    }

    public function getCacheKeyInfo()
    {
        return [
            'PRODUCT_LIST',
            $this->_storeManager->getStore()->getId(),
            $this->getCategory()->getId(),
            $this->getLayout()->getChildName($this->getNameInLayout()),
            template,
        ];
    }
}
```

### Invalidation by Tags vs by ID

```php
<?php
// Cache invalidation patterns

// 1. Invalidate by specific cache ID (exact match)
$cache->remove('my_specific_cache_key');

// 2. Invalidate by tags (all entries with ANY matching tag)
$cache->clean(
    ['P_123', 'C_456'],    // Tags to match
    \Zend_Cache::CLEANING_MODE_MATCHING_ANY_TAG
);

// 3. Invalidate by tags (only entries with ALL matching tags)
$cache->clean(
    ['P_123', 'STORE_1'],   // All tags must match
    \Zend_Cache::CLEANING_MODE_MATCHING_ALL_TAGS
);

// 4. Invalidate all
$cache->flush();
```

---

## 8. Custom Cache Type

Creating a custom cache type allows modules to cache expensive operations with proper tag-based invalidation.

### Step 1: Declare Cache Type

Create `app/etc/cache_types.xml` in your module:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Cache/etc/cache_types.xsd">
    <type name="vendor_module_custom"
          instance="Vendor\Module\Cache\Type\CustomCache"
          translate="label"
          label="Custom Module Cache">
        <label>Custom Module Cache</label>
    </type>
</config>
```

### Step 2: Implement Cache Type Class

```php
<?php
// Vendor/Module/Cache/Type/CustomCache.php

declare(strict_types=1);

namespace Vendor\Module\Cache\Type;

use Magento\Framework\App\ObjectManager\ConfigLoader;
use Magento\Framework\Cache\Frontend\Decorator\Tagspecific;
use Magento\Framework\Cache\InvalidateTracker;
use Magento\Framework\Serialize\Serializer\Json;

class CustomCache extends Tagspecific
{
    /**
     * Cache type identifier
     */
    public const CACHE_TYPE = 'vendor_module_custom';

    /**
     * @var InvalidateTracker|null
     */
    private $invalidateTracker;

    /**
     * @var Json
     */
    private $json;

    public function __construct(
        \Magento\Framework\Cache\FrontendInterface $frontend,
        InvalidateTracker $invalidateTracker,
        Json $json,
        ?string $tagPrefix = null
    ) {
        parent::__construct($frontend, $tagPrefix ?? self::CACHE_TYPE . '_');
        $this->invalidateTracker = $invalidateTracker;
        $this->json = $json;
    }

    /**
     * Get cached data by key
     *
     * @param string $key
     * @return array|null
     */
    public function get($key): ?array
    {
        $data = $this->load($this->getCacheId($key));

        if ($data === false) {
            return null;
        }

        return $this->json->unserialize($data);
    }

    /**
     * Set cached data with tags
     *
     * @param string $key
     * @param array $data
     * @param string[] $tags
     * @param int|null $lifetime
     * @return bool
     */
    public function set(string $key, array $data, array $tags = [], ?int $lifetime = null): bool
    {
        $serialized = $this->json->serialize($data);

        return $this->save(
            $serialized,
            $key,
            $this->prepareTags($tags),
            $lifetime ?? self::DEFAULT_LIFETIME
        );
    }

    /**
     * Invalidate cache by tags
     *
     * @param string[] $tags
     * @return bool
     */
    public function invalidate(array $tags): bool
    {
        return $this->clean($this->prepareTags($tags));
    }

    /**
     * Prepare tags with prefix
     *
     * @param string[] $tags
     * @return string[]
     */
    private function prepareTags(array $tags): array
    {
        $prefix = $this->getTagPrefix();
        $prepared = [];

        foreach ($tags as $tag) {
            $prepared[] = $prefix . $tag;
        }

        return $prepared;
    }
}
```

### Step 3: Use the Custom Cache

```php
<?php
// In a service class

namespace Vendor\Module\Model;

use Vendor\Module\Cache\Type\CustomCache;

class ExpensiveCalculator
{
    public function __construct(
        private readonly CustomCache $cache,
    ) {}

    public function calculate(int $productId, int $storeId): array
    {
        $cacheKey = "calculation_{$productId}_{$storeId}";

        // Try to get from cache
        $cached = $this->cache->get($cacheKey);
        if ($cached !== null) {
            return $cached;
        }

        // Perform expensive calculation
        $result = $this->doCalculation($productId, $storeId);

        // Cache the result with relevant tags
        $this->cache->set(
            $cacheKey,
            $result,
            [
                'P_' . $productId,      // Product tag
                'STORE_' . $storeId,   // Store tag
            ],
            86400 // 24 hours
        );

        return $result;
    }

    /**
     * Invalidate calculations when product is updated
     */
    public function invalidateForProduct(int $productId): void
    {
        $this->cache->invalidate(['P_' . $productId]);
    }
}
```

### Step 4: Register InvalidateHandler

```php
<?php
// etc/frontend/di.xml or etc/adminhtml/di.xml

<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Vendor\Module\Cache\Type\CustomCache">
        <arguments>
            <argument name="invalidateTracker" xsi:type="object">Magento\Framework\Cache\InvalidateTracker</argument>
        </arguments>
    </type>
</config>
```

---

## 9. Cache Invalidators

Cache invalidators are mechanisms that trigger cache cleaning based on specific events or operations.

### InvalidateHandler Pattern

```php
<?php
// vendor/magento/framework/Cache/InvalidateHandler.php

namespace Magento\Framework\Cache;

class InvalidateHandler
{
    /**
     * @var array<string, string[]>
     */
    private $trackedTags = [];

    /**
     * @var \Psr\Log\LoggerInterface
     */
    private $logger;

    public function __construct(\Psr\Log\LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    /**
     * Register a cache tag for tracking
     *
     * @param string $cacheType
     * @param string[] $tags
     * @return void
     */
    public function register(string $cacheType, array $tags): void
    {
        $this->trackedTags[$cacheType] = array_merge(
            $this->trackedTags[$cacheType] ?? [],
            $tags
        );
    }

    /**
     * Invalidate all tracked cache types by tags
     *
     * @param string[] $tags
     * @return void
     */
    public function invalidate(array $tags): void
    {
        foreach ($this->trackedTags as $cacheType => $trackedTags) {
            $matchingTags = array_intersect($tags, $trackedTags);
            if (!empty($matchingTags)) {
                try {
                    $this->getCacheFrontend($cacheType)->clean(
                        $matchingTags,
                        \Zend_Cache::CLEANING_MODE_MATCHING_ANY_TAG
                    );
                } catch (\Exception $e) {
                    $this->logger->error('Cache invalidation failed: ' . $e->getMessage());
                }
            }
        }
    }
}
```

### Event-Based Cache Cleaning

```php
<?php
// etc/events.xml

<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <!-- Clean product-related cache when product is saved -->
    <event name="catalog_product_save_commit_after">
        <observer name="invalidate_product_cache"
                  instance="Magento\Catalog\Observer\InvalidateCacheOnProductSave" />
    </event>

    <!-- Clean category-related cache when category is saved -->
    <event name="catalog_category_save_commit_after">
        <observer name="invalidate_category_cache"
                  instance="Magento\Catalog\Observer\InvalidateCacheOnCategorySave" />
    </event>

    <!-- Clean full page cache on config change -->
    <event name="admin_system_config_changed">
        <observer name="invalidate_full_page_cache"
                  instance="Magento\PageCache\Observer\InvalidateFullPageCache" />
    </event>
</config>
```

### Observer Implementation for Cache Invalidation

```php
<?php
// Magento/Catalog/Observer/InvalidateCacheOnProductSave.php

namespace Magento\Catalog\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Framework\Cache\InvalidateHandler;
use Magento\Catalog\Model\Product as ProductModel;

class InvalidateCacheOnProductSave implements ObserverInterface
{
    /**
     * @var InvalidateHandler
     */
    private $invalidateHandler;

    public function __construct(InvalidateHandler $invalidateHandler)
    {
        $this->invalidateHandler = $invalidateHandler;
    }

    /**
     * {@inheritdoc}
     */
    public function execute(Observer $observer)
    {
        /** @var ProductModel $product */
        $product = $observer->getEvent()->getProduct();

        // Get all cache tags to invalidate
        $tags = [
            'P_' . $product->getId(),                    // Product specific
            'EAV_PRODUCT_' . $product->getEntityTypeId(), // All products of this type
            'CATALOG_PRODUCT_' . $product->getId(),      // Alternative naming
        ];

        // Add category tags if product is assigned to categories
        $categoryIds = $product->getCategoryIds();
        foreach ($categoryIds as $categoryId) {
            $tags[] = 'C_' . $categoryId;
        }

        $this->invalidateHandler->invalidate($tags);
    }
}
```

### Indexer-Related Cache Clearing

```php
<?php
// Magento/Indexer/Model/Processor.php

namespace Magento\Indexer\Model;

class Processor
{
    /**
     * Invalidate cache after indexer run
     *
     * @param string $indexerId
     * @return void
     */
    public function invalidateCache(string $indexerId): void
    {
        $tags = $this->getIndexerCacheTags($indexerId);

        foreach ($tags as $cacheType => $cacheTags) {
            $this->cacheManager->getCache($cacheType)->clean(
                $cacheTags,
                \Zend_Cache::CLEANING_MODE_MATCHING_ANY_TAG
            );
        }

        $this->logger->info("Cache invalidated for indexer: {$indexerId}");
    }

    /**
     * Get cache tags associated with indexer
     *
     * @param string $indexerId
     * @return array<string, string[]>
     */
    private function getIndexerCacheTags(string $indexerId): array
    {
        $mapping = [
            'catalog_product_price' => [
                'full_page' => ['P_', 'CONFIG_PRODUCT_PRICE_'],
                'block_html' => ['product.price'],
            ],
            'catalog_category_product' => [
                'full_page' => ['C_', 'P_'],
                'block_html' => ['category.products.list'],
            ],
            'catalogsearch_fulltext' => [
                'full_page' => ['SEARCH_'],
            ],
        ];

        return $mapping[$indexerId] ?? [];
    }
}
```

### After Plugin for Automatic Cache Invalidation

```php
<?php
// Vendor/Module/Plugin/InvalidateCacheOnSave.php

namespace Vendor\Module\Plugin;

use Vendor\Module\Model\ResourceModel\ExpensiveData as ExpensiveDataResource;
use Vendor\Module\Cache\Type\CustomCache;

class InvalidateCacheOnSave
{
    public function __construct(
        private readonly CustomCache $cache,
    ) {}

    /**
     * Invalidate cache after expensive data is saved
     *
     * @param ExpensiveDataResource $subject
     * @param callable $proceed
     * @param mixed $object
     * @return void
     */
    public function aroundSave(
        ExpensiveDataResource $subject,
        callable $proceed,
        $object
    ): void {
        $result = $proceed($object);

        // Invalidate cache based on saved data
        $this->cache->invalidate([
            'EXPENSIVE_DATA_' . $object->getId(),
            'EXPENSIVE_DATA_STORE_' . $object->getStoreId(),
        ]);

        return $result;
    }

    /**
     * Invalidate cache after expensive data is deleted
     *
     * @param ExpensiveDataResource $subject
     * @param callable $proceed
     * @param mixed $object
     * @return void
     */
    public function aroundDelete(
        ExpensiveDataResource $subject,
        callable $proceed,
        $object
    ): void {
        // Invalidate before delete to ensure cache is cleared
        $this->cache->invalidate([
            'EXPENSIVE_DATA_' . $object->getId(),
            'EXPENSIVE_DATA_STORE_' . $object->getStoreId(),
        ]);

        return $proceed($object);
    }
}
```

---

## 10. HTTP Cache Control Headers

Magento uses HTTP cache control headers to communicate caching behavior to browsers and reverse proxies.

### Header Types and Their Purpose

| Header | Purpose | Magento Sets | Varnish Respects |
|--------|---------|-------------|------------------|
| `Cache-Control` | Directives for caching (no-cache, max-age, public, private) | Yes | Yes |
| `Expires` | HTTP/1.0 absolute expiration date | Yes | Yes |
| `ETag` | Opaque identifier for specific version of content | No | Yes |
| `Last-Modified` | Date when content was last modified | No | Yes |
| `Pragma` | Legacy HTTP/1.0 cache control | Yes (no-cache) | Yes |
| `Vary` | Which request headers affect caching | Yes (Accept-Encoding) | Yes |

### Magento Cache Header Implementation

```php
<?php
// vendor/magento/framework/App/Response/Http.php

namespace Magento\Framework\App\Response;

class Http implements \Magento\Framework\App\Response\HttpInterface
{
    /**
     * Set cache control headers
     *
     * @param int $maxAge
     * @param bool $isPrivate
     * @param array $additionalFields
     * @return $this
     */
    public function setCacheControl(int $maxAge, bool $isPrivate = true, array $additionalFields = []): self
    {
        if ($isPrivate) {
            $this->setHeader('Cache-Control', 'private, max-age=' . $maxAge, true);
        } else {
            $this->setHeader(
                'Cache-Control',
                'public, max-age=' . $maxAge . ', s-maxage=' . $maxAge,
                true
            );
        }

        // Set Expires header for HTTP/1.0 compatibility
        $this->setHeader(
            'Expires',
            'Mon, 31 Mar 2026 00:00:00 GMT',
            true
        );

        return $this;
    }

    /**
     * Set no-cache headers
     *
     * @return $this
     */
    public function setNoCache(): self
    {
        $this->setHeader('Cache-Control', 'no-store, no-cache, must-revalidate, max-age=0', true);
        $this->setHeader('Pragma', 'no-cache', true);
        $this->setHeader('Expires', 'Mon, 31 Mar 2026 00:00:00 GMT', true);

        return $this;
    }
}
```

### Page Cache Headers

```php
<?php
// vendor/magento/framework/App/PageCache/Cache.php

namespace Magento\Framework\App\PageCache;

class Cache
{
    /**
     * Default cache lifetime (1 hour)
     */
    const DEFAULT_LIFETIME = 3600;

    /**
     * {@inheritdoc}
     */
    public function getHeaders()
    {
        $headers = [];

        if ($this->_cache->getFrontend()->getOption('cache')) {
            $tags = $this->getBackend()->getTags();

            $headers['X-Magento-Cache-Id'] = $this->_cacheId;
            $headers['X-Magento-Tags'] = implode(',', $tags);
            $headers['Cache-Control'] = 'max-age=' . self::DEFAULT_LIFETIME . ', public';
        }

        return $headers;
    }
}
```

### Controller Setting Cache Headers

```php
<?php
// In a custom controller

namespace Vendor\Module\Controller\Index;

class Index extends \Magento\Framework\App\Action\Action
{
    /**
     * {@inheritdoc}
     */
    public function execute()
    {
        /** @var \Magento\Framework\App\Response\Http $response */
        $response = $this->getResponse();

        // Cache this page for 1 hour, public (cacheable by reverse proxy)
        $response->setHeader('Cache-Control', 'public, max-age=3600', true);

        // Set private (browser only, not reverse proxy)
        // $response->setHeader('Cache-Control', 'private, max-age=3600', true);

        // Set no-cache
        // $response->setNoCache();

        // Add ETag for content fingerprinting
        $response->setHeader('ETag', md5($content), true);

        return $this->resultPageFactory->create();
    }
}
```

### Vary Header for Multi-Device Caching

```php
<?php
// Set Vary header for proper cache differentiation

// In di.xml or around plugin
$response->setHeader('Vary', 'Accept-Encoding, X-Requested-With', true);

// For mobile-specific caching
$response->setHeader('Vary', 'User-Agent', true);
```

---

## 11. Development Cache Tips

### Never Cache User-Specific Data

**Critical Rule:** Never cache data containing:
- Customer information
- Shopping cart contents
- Wishlist items
- Session data
- Personalized recommendations
- Price adjustments for specific customers

```php
<?php
// WRONG - Never do this
public function getCustomerName(): string
{
    $cacheKey = 'customer_name_' . $this->customerSession->getCustomerId();
    $cached = $this->cache->load($cacheKey);

    if ($cached !== false) {
        return $cached;  // Could return another customer's data!
    }

    $name = $this->customer->getName();
    $this->cache->save($cacheKey, $name);  // Shared cache!
    return $name;
}

// CORRECT - Use customer sections for private data
public function getCustomerName(): string
{
    // Customer data is loaded via JavaScript customer-data section
    return $this->customer->getName();
}
```

### Hole Punching for Dynamic Content

Use ESI (Edge Side Includes) or JavaScript-based hole punching for dynamic blocks within cached pages:

```php
<?php
// In Block class - mark as non-cacheable
class DynamicCartBlock extends \Magento\Framework\View\Element\Template
{
    protected function getCacheLifetime()
    {
        // Return null = block is not cached
        return null;
    }

    protected function getCacheKeyInfo()
    {
        return [
            'DYNAMIC_CART',
            $this->customerSession->getCustomerId(),
            // Do NOT include in cache key if user-specific
        ];
    }
}
```

### Using cacheable="false" in Layout XML

```xml
<!-- In app/design/frontend/Vendor/theme/Magento_Theme/layout/default.xml -->

<!-- Non-cacheable header block (customer-specific) -->
<referenceBlock name="header.panel" remove="false">
    <block name="header.links"
           template="Magento_Theme::html/header.phtml"
           cacheable="false" />
</referenceBlock>

<!-- Cacheable footer block (same for all users) -->
<referenceBlock name="footer" remove="false">
    <block name="footer.content"
           template="Magento_Theme::html/footer.phtml"
           cacheable="true" />
</referenceBlock>
```

### Layout Cache Control

```xml
<!-- Disable layout cache for specific handles -->

<!-- In app/design/frontend/Vendor/theme/Magento_Catalog/layout/catalog_product_view.xml -->
<layout xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/layout_generic.xsd">
    <update handle="empty"/>

    <!-- Product page uses hole punching for dynamic elements -->
    <referenceBlock name="product.info.main" cacheable="true"/>
    <referenceBlock name="product.info.price" cacheable="true"/>
    <referenceBlock name="product.info.addtocart" cacheable="false"/>
    <referenceBlock name="product.info.addto" cacheable="false"/>
</layout>
```

### Debugging Cache Issues

```bash
# Enable cache debugging
bin/magento config:set system/full_page_cache/debug 1

# Check cache headers in response
curl -I https://yourstore.com/

# Look for:
# X-Magento-Cache-Id: PAGE_...
# X-Magento-Tags: P_123, C_456...
# X-Cache: HIT/MISS
```

### Cache-Friendly Development Checklist

```markdown
## Cache Optimization Checklist

### DO
- [ ] Use cache tags when saving custom cache entries
- [ ] Invalidate cache when underlying data changes
- [ ] Keep cache keys short but descriptive
- [ ] Set appropriate TTL for cache entries
- [ ] Use hole punching for user-specific content
- [ ] Test with cache disabled during development

### DON'T
- [ ] Cache customer-specific data
- [ ] Store sensitive data in cache
- [ ] Use serialized PHP objects in Redis cache (use JSON)
- [ ] Set infinite TTL for data that may change
- [ ] Skip cache invalidation after data updates
- [ ] Cache complete page responses when using hole punching
```

### CLI Commands for Cache Debugging

```bash
# View cache statistics
bin/magento cache:status

# Clean specific types
bin/magento cache:clean full_page block_html

# Flush all cache
bin/magento cache:flush

# Disable specific cache type
bin/magento cache:type:disable full_page

# Re-enable
bin/magento cache:type:enable full_page

# Check cache configuration
bin/magento config:show system/full_page_cache/caching_application
```

---

## Summary

Magento 2.4.8's caching architecture provides multiple layers of optimization:

| Component | Purpose | When to Configure |
|-----------|---------|-------------------|
| **Redis** | General-purpose cache backend | Always in production |
| **Varnish** | HTTP reverse proxy, FPC | Recommended for production |
| **Built-in FPC** | Application-level page cache | When not using Varnish |
| **Cache Tags** | Targeted invalidation | For custom cache types |
| **CLI Commands** | Cache management | During development/deployment |

**Key Takeaways:**

1. **Use Varnish** in production for best FPC performance
2. **Configure Redis** for distributed caching across servers
3. **Understand cache tags** for proper invalidation strategy
4. **Never cache user-specific data** - use hole punching
5. **Use CLI commands** appropriately: `clean` vs `flush`
6. **Test with cache disabled** during development to catch cache-dependency bugs

---

## Further Reading

- [Magento 2.4.8 Documentation: Cache Management](https://experienceleague.adobe.com/docs/commerce-operations/configuration-guide/cache/)
- [Varnish Documentation](https://www.varnish-software.com/developers/tutorials/)
- [Redis Cache Backend](https://developer.adobe.com/commerce/php/development/cache/partial/backend/redis/)