---
title: "05 - WebAPI Rate Limiting & Quota"
description: "Magento 2.4.8 REST/GraphQL rate limiting: per-route limits, global quotas, IP allowlisting, webapi_limits configuration, and API security"
tags: [magento2, api, rate-limiting, quota, webapi, security, performance]
rank: 5
pathways: [magento2-deep-dive]
see_also:
  - "REST/GraphQL API reference: ../07-api-development/11-graphql-rest-api.md"
  - "Security patterns: _supplemental/07-security-acl.md"
---

# WebAPI Rate Limiting & Quota Management

## Chapter Overview

This chapter covers Magento 2.4.7+'s built-in WebAPI rate limiting and quota enforcement system. By the end, you will understand how to configure per-route and global rate limits, implement IP allowlisting through `webapi.xml`, interpret rate limit HTTP headers, and monitor usage through the `webapi_rate_limit_logs` table.

**Prerequisites:** REST API architecture (Topic 1 of this module), authentication methods, `webapi.xml` route definitions.

**Magento Version Note:** Rate limiting via `webapi.xml` `<data>` node and the `rate_limiting` env.php section requires **Magento 2.4.7 or later**. On earlier versions, rate limiting must be implemented via custom plugins or infrastructure-layer solutions (nginx, Varnish, Redis).

---

## 1. Why Rate Limiting Matters

### 1.1 Preventing API Abuse

Without rate limiting, Magento's WebAPI is vulnerable to several attack vectors:

| Attack Vector | Impact | Example |
|---------------|--------|---------|
| **Cart manipulation spam** | Resource exhaustion, inventory inconsistencies | Rapid add-to-cart via script |
| **Credential brute force** | Account takeover | `/V1/integration/admin/token` repeated calls |
| **Denial of Service (DoS)** | Site downtime | Flooding product search endpoint |
| **Price scraping** | Competitive intelligence theft | Enumerating `/V1/products` for full catalog |
| **Checkout exhaustion** | Cart reservation attacks | Abandoned cart bombarding |

### 1.2 Protecting Resource-Intensive Operations

Some Magento operations are inherently expensive. Rate limiting prevents runaway clients from exhausting server resources:

```
Request Flow WITH Rate Limiting:

Client                    Magento                    Resource Cost
─────                     ───────                    ─────────────
POST /cart                → Rate limit check         ~0.1ms
  ↓ if allowed            → Load quote               ~50ms
  ↓                       → Inventory check          ~20ms
  ↓                       → Save quote               ~30ms
  Response 200            ← Total: ~100ms

 ───────────────────────────────────────────────────────────────

Client                    Magento                    Resource Cost
─────                     ───────                    ─────────────
POST /cart (100x/sec)     → Rate limit BLOCKED       ~0.1ms
  ↓ blocked               ← HTTP 429 returned
  Response 429            ← Total: ~0.1ms each
```

Without rate limiting, a single client could fire 100 cart operations/sec, each consuming ~100ms of server time — 10 seconds of CPU per second of wall time.

### 1.3 Business Use Case: Tiered API Access

Rate limiting enables tiered API access models common in e-commerce platforms:

```
Tier Strategy:

┌─────────────────────────────────────────────────────────────────────┐
│                     API Access Tiers                                │
├───────────────┬───────────────┬───────────────┬─────────────────────┤
│   Free Tier   │  Standard     │   Premium     │   Enterprise        │
├───────────────┼───────────────┼───────────────┼─────────────────────┤
│ 100 req/hour  │ 1000 req/hour │ 5000 req/hour │ Unlimited / Custom  │
│ Anonymous     │ Customer token│ Integration   │ Dedicated infra     │
│ Read-only     │ Full access   │ Bulk + Async  │ SLA guarantees      │
│ Product read  │ All standard  │ All endpoints │ White-label        │
│ Category list │ Cart/Checkout │ Admin APIs    │ Custom rate limits  │
└───────────────┴───────────────┴───────────────┴─────────────────────┘
```

This tiering turns the WebAPI into a **product** — different pricing for different access levels.

---

## 2. Magento's Built-in Rate Limiting Architecture (M2.4.7+)

### 2.1 Architecture Overview

Magento's rate limiting is a multi-layer system:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Rate Limiting Flow                                  │
└─────────────────────────────────────────────────────────────────────────────┘

  HTTP Request
       │
       ▼
┌──────────────────┐
│ Route Matching   │  webapi.xml lookup
│ webapi.xml       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Rate Limit Check │  Check per-route limit (from <data>)
│ Service layer    │  Fall back to global limit (env.php)
└────────┬─────────┘
         │
    ┌────┴────┐
    │         │
 Over    Under
 limit   limit
    │         │
    ▼         ▼
 HTTP 429   Continue
 + headers  to service
```

### 2.2 Key Components

| Component | File/Class | Responsibility |
|-----------|-----------|----------------|
| Rate Limit Processor | `Magento\Webapi\Model\RateLimitProcessor` | Central orchestrator — checks limits before service invocation |
| Route Config Reader | `Magento\Webapi\Controller\Rest\Config` | Reads `webapi.xml` per-route attributes |
| Global Config | `app/etc/env.php` → `rate_limiting` | Fallback limits for all routes |
| Rate Limit Header Plugin | `Magento\Webapi\Controller\Rest\HeaderPlugin` | Adds `X-RateLimit-*` response headers |
| Quota Table | `webapi_rate_limit_logs` | Stores usage records for monitoring |

### 2.3 How Rate Limiting Interacts with Authentication

Rate limits are applied **after** authentication but **before** service invocation:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        Full Request Flow                                  │
└──────────────────────────────────────────────────────────────────────────┘

  HTTP Request
       │
       ▼
┌──────────────────┐
│ Authentication   │  Token/OAuth validated
│ (TokenValidator)  │  User type identified (admin/customer/integration)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ ACL Authorization │  Route ACL checked against user
│ (Authorization)   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Rate Limit Check │  Per-IP, per-token, or per-user
│ (RateLimitProcessor) │
└────────┬─────────┘
         │
    ┌────┴────┐
    │         │
 Exceeded   Pass
    │         │
    ▼         ▼
  HTTP 429   Service
  response   called
```

The rate limit key is determined by the authenticated context:

| Authentication Type | Rate Limit Key | Scope |
|---------------------|----------------|-------|
| Anonymous (no auth) | IP address | Per-IP |
| Customer token | Customer ID | Per-customer |
| Admin token | Admin user ID | Per-user |
| Integration (OAuth) | Integration ID | Per-integration |
| Custom plugin | Configurable | Per-plugin-key |

---

## 3. Global Rate Limits in env.php

### 3.1 Configuration Structure

Global rate limits are set in `app/etc/env.php` under the `rate_limiting` key:

```php
<?php
// app/etc/env.php

return [
    // ... other config ...

    'rate_limiting' => [
        // REST API global limit
        'rest' => [
            'limit'  => 1000,      // Max requests
            'period' => 3600,     // Time window in seconds (1 hour)
        ],

        // GraphQL global limit (separate from REST)
        'graphql' => [
            'limit'  => 500,       // Max requests
            'period' => 3600,      // Time window in seconds (1 hour)
        ],
    ],

    // ... other config ...
];
```

### 3.2 Configuration Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `rest.limit` | integer | `1000` | Max REST API requests per period |
| `rest.period` | integer | `3600` | Time window in seconds |
| `graphql.limit` | integer | `500` | Max GraphQL requests per period |
| `graphql.period` | integer | `3600` | Time window in seconds |

> **Performance Note:** Lower limits (e.g., 100/minute) on shared hosting; higher limits (e.g., 10,000/minute) for dedicated enterprise deployments.

### 3.3 How Global Limits Are Applied

Global limits apply **only when no per-route limit is configured**. The precedence is:

```
1. Per-route limit (from webapi.xml <data> node)
   └── If configured, this is always used

2. Global limit (from env.php)
   └── Used when route has no per-route limit

3. No limit
   └── Only if BOTH per-route and global are absent
```

### 3.4 What Happens When Limit Is Exceeded

When a client exceeds their rate limit, Magento returns:

**HTTP 429 Too Many Requests**

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json

{
    "message": "Rate limit exceeded",
    "code": 429,
    "trace": null   // Omitted in production mode
}
```

### 3.5 Rate Limit Response Headers

Every API response includes rate limit information in headers:

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1713950400
```

| Header | Description | Example |
|--------|-------------|---------|
| `X-RateLimit-Limit` | Maximum requests allowed in window | `1000` |
| `X-RateLimit-Remaining` | Requests remaining in current window | `847` |
| `X-RateLimit-Reset` | Unix timestamp when window resets | `1713950400` |

**Header insertion is handled by** `Magento\Webapi\Controller\Rest\HeaderPlugin::afterDispatch()`.

### 3.6 Reading the Reset Timestamp

The `X-RateLimit-Reset` header is a Unix epoch timestamp:

```php
// Convert to human-readable
$resetTimestamp = 1713950400;
$resetDateTime = date('Y-m-d H:i:s', $resetTimestamp);
// "2024-04-24 12:00:00"

// Calculate seconds until reset
$secondsUntilReset = $resetTimestamp - time();
```

---

## 4. Per-Route Rate Limiting

### 4.1 webapi.xml Rate Limit Configuration

Per-route rate limits override global limits and are defined in the `<data>` node of each `<route>`:

```xml
<?xml version="1.0"?>
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Webapi/etc/webapi.xsd">

    <!-- POST /V1/orders — place order, strict limit -->
    <route method="POST" url="/V1/orders">
        <service class="Magento\Sales\Api\OrderRepositoryInterface" method="save"/>
        <resources>
            <resource ref="Magento_Sales::orders"/>
        </resources>
        <data>
            <item name="rateLimitConfig" xsi:type="array">
                <item name="limit" xsi:type="number">30</item>
                <item name="period" xsi:type="number">60</item>
            </item>
        </data>
    </route>

    <!-- GET /V1/orders/:id — read order, moderate limit -->
    <route method="GET" url="/V1/orders/:id">
        <service class="Magento\Sales\Api\OrderRepositoryInterface" method="get"/>
        <resources>
            <resource ref="Magento_Sales::orders"/>
        </resources>
        <data>
            <item name="rateLimitConfig" xsi:type="array">
                <item name="limit" xsi:type="number">120</item>
                <item name="period" xsi:type="number">60</item>
            </item>
        </data>
    </route>

</routes>
```

### 4.2 Complete webapi.xml Example with Multiple Rate Limits

```xml
<?xml version="1.0"?>
<!-- etc/webapi.xml of Vendor/Module -->
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Webapi/etc/webapi.xsd">

    <!--
        High-value operation: checkout
        Very strict limit — prevents cart exhaustion attacks
    -->
    <route method="POST" url="/V1/carts/:quoteId/items">
        <service class="Magento\Quote\Api\CartItemRepositoryInterface" method="save"/>
        <resources>
            <resource ref="Magento_Customer::customer"/>
            <resource ref="self"/>
        </resources>
        <data>
            <item name="rateLimitConfig" xsi:type="array">
                <item name="limit" xsi:type="number">20</item>
                <item name="period" xsi:type="number">60</item>
            </item>
        </data>
    </route>

    <!--
        Search operation — expensive, needs protection
        60 requests per minute (1 per second)
    -->
    <route method="GET" url="/V1/products">
        <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="getList"/>
        <resources>
            <resource ref="Magento_Catalog::products"/>
        </resources>
        <data>
            <item name="rateLimitConfig" xsi:type="array">
                <item name="limit" xsi:type="number">60</item>
                <item name="period" xsi:type="number">60</item>
            </item>
        </data>
    </route>

    <!--
        Customer token generation — protect against brute force
        Very low limit: 5 per minute
    -->
    <route method="POST" url="/V1/integration/customer/token">
        <service class="Magento\Integration\Api\CustomerTokenServiceInterface" method="createCustomerAccessToken"/>
        <resources>
            <resource ref="anonymous"/>
        </resources>
        <data>
            <item name="rateLimitConfig" xsi:type="array">
                <item name="limit" xsi:type="number">5</item>
                <item name="period" xsi:type="number">60</item>
            </item>
        </data>
    </route>

    <!--
        Bulk export — high limit for legitimate batch jobs
        500 requests per hour (supports nightly batch exports)
    -->
    <route method="GET" url="/V1/products/export">
        <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="getList"/>
        <resources>
            <resource ref="Magento_Catalog::products"/>
        </resources>
        <data>
            <item name="rateLimitConfig" xsi:type="array">
                <item name="limit" xsi:type="number">500</item>
                <item name="period" xsi:type="number">3600</item>
            </item>
        </data>
    </route>

    <!--
        Anonymous product read — public, lowest limit
        200 per minute to prevent price scraping
    -->
    <route method="GET" url="/V1/products/:id">
        <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="get"/>
        <resources>
            <resource ref="anonymous"/>
        </resources>
        <data>
            <item name="rateLimitConfig" xsi:type="array">
                <item name="limit" xsi:type="number">200</item>
                <item name="period" xsi:type="number">60</item>
            </item>
        </data>
    </route>

</routes>
```

### 4.3 How Per-Route Limits Override Global Limits

```
Example: POST /V1/orders has per-route limit of 30/min

env.php global: rest.limit=1000, rest.period=3600

Request: 35 POST /V1/orders calls in 1 minute

Result:
├── Request 1-30:    Allowed  (within per-route limit)
├── Request 31-35:   BLOCKED  (per-route limit exceeded)
│                    HTTP 429 returned
│
└── Other routes (no per-route limit) still use global:
    ├── GET /V1/customers    → uses global: 1000/hour
    └── GET /V1/products     → uses global: 1000/hour
```

### 4.4 Rate Limiting for Anonymous vs Authenticated Endpoints

| Route Type | Rate Limit Key | Behavior |
|------------|---------------|----------|
| `anonymous` resource | IP address | Shared limit per IP |
| `self` or customer token | Customer ID | Per-customer limit |
| Admin ACL resource | Admin user ID | Per-admin limit |
| Integration OAuth | Integration ID | Per-integration limit |

**Example: Different limits for same route based on auth**

```xml
<!-- Public read - anonymous rate limit -->
<route method="GET" url="/V1/products/:id">
    <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="get"/>
    <resources>
        <resource ref="anonymous"/>   <!-- IP-based limiting -->
    </resources>
    <data>
        <item name="rateLimitConfig" xsi:type="array">
            <item name="limit" xsi:type="number">100</item>
            <item name="period" xsi:type="number">60</item>
        </item>
    </data>
</route>

<!-- Same entity type - authenticated gets higher limit -->
<route method="POST" url="/V1/products/:id/reviews">
    <service class="Magento\Review\Api\ReviewRepositoryInterface" method="save"/>
    <resources>
        <resource ref="Magento_Customer::customer"/>
        <resource ref="self"/>
    </resources>
    <data>
        <item name="rateLimitConfig" xsi:type="array">
            <item name="limit" xsi:type="number">30</item>
            <item name="period" xsi:type="number">60</item>
        </item>
    </data>
</route>
```

---

## 5. GraphQL Rate Limiting

### 5.1 GraphQL-Specific Challenges

GraphQL presents unique rate limiting challenges compared to REST:

```
REST vs GraphQL Rate Limiting:

REST:
  /V1/products        → 1 request
  /V1/products/1      → 1 request
  Total: 2 requests counted separately

GraphQL:
  query { products { items { name } } }  → 1 request
  But: could return 10,000 products
  And: each field costs computation
```

### 5.2 Query Complexity Analysis

Magento's GraphQL rate limiting uses **query complexity scoring**:

```php
// di.xml — GraphQL rate limiting configuration
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:etc/di.xsd">

    <!--
        GraphQL query complexity limit
        Each field has a cost; query must stay under threshold
    -->
    <type name="Magento\Framework\GraphQl\Query\ComplexityCalculator">
        <arguments>
            <argument name="maxComplexity">1000</argument>
        </arguments>
    </type>

    <!-- Maximum query depth (prevents circular queries) -->
    <type name="Magento\Framework\GraphQl\Query\DepthCalculator">
        <arguments>
            <argument name="maxDepth">15</argument>
        </arguments>
    </type>

</config>
```

### 5.3 Complexity Calculation Example

```
Field Cost Reference:

Scalar field (String, Int, Float):   1 point
Simple object:                        5 points
List field:                          3 points per item
Connection (pagination):            10 points base + 1 per item
Nested query:                       Multiplier based on depth

Example Query Analysis:
────────────────────────
query {
  products(pageSize: 100) {     # 10 points (connection)
    items {                      # 3 points × 100 = 300 points
      name                       # 1 point × 100 = 100 points
      price {                    # 5 points × 100 = 500 points
        amount                    # 1 point × 100 = 100 points
        currency                  # 1 point × 100 = 100 points
      }
      categories {               # 5 points × 100 = 500 points
        name                      # 1 point × 100 = 100 points
      }
    }
  }
}
────────────────────────
Total Complexity: 10 + 300 + 100 + 500 + 100 + 100 + 500 + 100 + 100 = 1,810

If maxComplexity = 1000: ❌ REJECTED (too complex)
```

### 5.4 Field Cost Analysis Configuration

You can customize field costs in your module's `etc/graphql/di.xml`:

```xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:etc/di.xsd">

    <!-- Custom field cost configuration -->
    <type name="Magento\CatalogGraphQl\Model\Resolver\Product">
        <arguments>
            <argument name="complexity">5</argument>
        </arguments>
    </type>

    <!-- High-cost fields (e.g., full-text search) -->
    <type name="Magento\CatalogGraphQl\Model\Resolver\Products">
        <arguments>
            <argument name="complexity">50</argument>
        </arguments>
    </type>

</config>
```

### 5.5 Query Depth Limiting

Prevents circular/nested query attacks:

```graphql
# Maximum depth of 15

# Depth 1 - OK
query { products }

# Depth 2 - OK
query { products { category { products } } }

# Depth 3 - OK
query { products { category { parent { products } } } }

# Depth 16 - REJECTED
query {
  products {
    category {
      parent {
        products {
          category {
            parent {
              products {
                category {
                  parent { # Depth 10... }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### 5.6 webapi_limits Configuration (Adobe Commerce)

Adobe Commerce adds additional quota management via `system.xml` configuration:

```xml
<!-- etc/adminhtml/system.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:config-etc/system.xsd">
    <section id="webapi">
        <group id="limits">
            <field id="rest_rate_limit">
                <label>REST API Rate Limit</label>
                <frontend_type>text</frontend_type>
            </field>
            <field id="graphql_rate_limit">
                <label>GraphQL Rate Limit</label>
                <frontend_type>text</frontend_type>
            </field>
        </group>
    </section>
</config>
```

---

## 6. IP Allowlisting / Denylisting

### 6.1 IP Allowlisting in webapi.xml

Restrict specific routes to trusted IP ranges:

```xml
<?xml version="1.0"?>
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Webapi/etc/webapi.xsd">

    <!--
        Admin-only endpoint — restrict to internal network
        Allow 192.168.1.0/24 subnet
    -->
    <route method="POST" url="/V1/admin/users">
        <service class="Magento\User\Api\UserRepositoryInterface" method="save"/>
        <resources>
            <resource ref="Magento_Admin::users"/>
        </resources>
        <data>
            <item name="allowedOrigins" xsi:type="array">
                <item name="ip" xsi:type="string">192.168.1.0/24</item>
            </item>
        </data>
    </route>

    <!--
        Bulk export — allow specific partner IPs
        CIDR notation supported: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
    -->
    <route method="GET" url="/V1/export/bulk">
        <service class="Magento\ImportExport\Api\ExportManagementInterface" method="export"/>
        <resources>
            <resource ref="Vendor_Module::export"/>
        </resources>
        <data>
            <item name="allowedOrigins" xsi:type="array">
                <item name="ip" xsi:type="string">10.0.0.0/8</item>
                <item name="ip" xsi:type="string">192.168.100.0/24</item>
            </item>
        </data>
    </route>

    <!--
        Allow single IP with no CIDR (exact match)
    -->
    <route method="POST" url="/V1/internal/orders/sync">
        <service class="Vendor\Module\Api\OrderSyncInterface" method="sync"/>
        <resources>
            <resource ref="Vendor_Module::internal"/>
        </resources>
        <data>
            <item name="allowedOrigins" xsi:type="array">
                <item name="ip" xsi:type="string">203.0.113.50</item>
            </item>
        </data>
    </route>

</routes>
```

### 6.2 IP Allowlist Response

When an IP is blocked by allowlist rules:

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
    "message": "Access denied from IP 203.0.113.50",
    "code": 403,
    "trace": null
}
```

### 6.3 Nginx-Level Rate Limiting as Complementary Layer

`webapi.xml` rate limiting operates at the PHP application layer. For production environments, implement nginx-level limiting as a first line of defense:

```nginx
# /etc/nginx/conf.d/rate-limiting.conf

# Define rate limit zones
limit_req_zone $binary_remote_addr zone=api_general:10m rate=30r/s;
limit_req_zone $binary_remote_addr zone=api_strict:10m rate=5r/s;
limit_req_zone $binary_remote_addr zone=api_auth:10m rate=2r/s;

# Apply to location blocks
location /rest/ {

    # Auth endpoints — strict limiting (brute force protection)
    location ~ ^/rest/V1/integration/(admin|customer)/token {
        limit_req zone=api_strict burst=3 nodelay;
        limit_req_status 429;
        proxy_pass http://php-fpm-backend;
    }

    # Public product endpoints — moderate limiting
    location ~ ^/rest/V1/products {
        limit_req zone=api_general burst=20 nodelay;
        limit_req_status 429;
        proxy_pass http://php-fpm-backend;
    }

    # Checkout/cart — moderate limiting
    location ~ ^/rest/V1/carts {
        limit_req zone=api_general burst=10 nodelay;
        limit_req_status 429;
        proxy_pass http://php-fpm-backend;
    }

    # Internal/admin — low limiting
    location ~ ^/rest/V1/admin {
        limit_req zone=api_auth burst=2 nodelay;
        limit_req_status 429;
        proxy_pass http://php-fpm-backend;
    }

    # Default
    proxy_pass http://php-fpm-backend;
}
```

**Layered approach summary:**

```
Internet Request
       │
       ▼
┌─────────────────┐
│ Nginx           │  ← First line: rough rate limiting, DoS protection
│ (limit_req_zone)│    Blocks before PHP processes request
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Magento         │  ← Second line: per-route webapi.xml limits
│ (webapi.xml)    │    Auth-aware, per-user/token limiting
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Application     │  ← Third line: custom plugin logic
│ (Plugin)        │    Business-specific abuse detection
└─────────────────┘
```

---

## 7. Quota Enforcement (Adobe Commerce)

### 7.1 The webapi_limits Table

Adobe Commerce tracks API usage in the `webapi_limits` table (and `webapi_rate_limit_logs` for detailed logging):

```sql
-- webapi_limits table schema
CREATE TABLE webapi_limits (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    identifier      VARCHAR(255) NOT NULL,           -- Rate limit key (IP, token hash, etc.)
    limit_type      ENUM('rest', 'graphql') NOT NULL,
    hits            INT UNSIGNED NOT NULL DEFAULT 0,  -- Request count in window
    window_start    BIGINT UNSIGNED NOT NULL,         -- Window start timestamp
    window_duration INT UNSIGNED NOT NULL,             -- Window duration in seconds
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY idx_identifier_type_window (identifier, limit_type, window_start),
    KEY idx_window_start (window_start)
);
```

### 7.2 Quota Exceeded Response Format

```json
// When quota is exceeded — HTTP 429
{
    "message": "Rate limit exceeded",
    "code": 429,
    "parameters": {
        "limit": 1000,
        "period": 3600,
        "reset": 1713950400
    }
}
```

### 7.3 Clearing Rate Limit Counters (for Testing)

During development and testing, you may need to reset rate limit counters:

```sql
-- Clear all rate limit data (development only)
TRUNCATE TABLE webapi_limits;
TRUNCATE TABLE webapi_rate_limit_logs;
```

```php
// Programmatic reset via repository
<?php
namespace Vendor\Module\Console\Command;

use Magento\Framework\MessageQueue\PublisherInterface;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class ResetRateLimits extends Command
{
    protected $resourceConnection;

    public function __construct(
        \Magento\Framework\App\ResourceConnection $resourceConnection
    ) {
        $this->resourceConnection = $resourceConnection;
        parent::__construct();
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $connection = $this->resourceConnection->getConnection();
        $tableName = $connection->getTableName('webapi_limits');

        $connection->query("TRUNCATE TABLE {$tableName}");

        $output->writeln('<info>Rate limits reset successfully</info>');
    }
}
```

> **⚠️ Warning:** Never clear `webapi_limits` in production. This would allow clients to bypass rate limiting immediately after the clear.

### 7.4 Monitoring Quota Usage

```sql
-- View top API consumers (by IP or token)
SELECT
    identifier,
    limit_type,
    hits,
    window_start,
    window_duration,
    FROM_UNIXTIME(window_start) as window_start_time
FROM webapi_limits
WHERE window_start > UNIX_TIMESTAMP() - 3600
ORDER BY hits DESC
LIMIT 20;

-- View rate limit events in logs
SELECT
    identifier,
    event_type,
    created_at,
    request_count
FROM webapi_rate_limit_logs
WHERE created_at > NOW() - INTERVAL 24 HOUR
ORDER BY created_at DESC;

-- Check for IPs hitting rate limits frequently
SELECT
    identifier,
    COUNT(*) as rate_limit_events,
    SUM(hits) as total_hits
FROM webapi_limits
WHERE window_start > UNIX_TIMESTAMP() - 86400
GROUP BY identifier
HAVING COUNT(*) > 10
ORDER BY rate_limit_events DESC;
```

---

## 8. Integration with OAuth and Tokens

### 8.1 Rate Limiting by Integration Token

When using OAuth-based integration authentication, rate limits are applied per-integration:

```php
// Integration token → Integration ID → Rate limit key
$token = 'aomensof283mr...';        // OAuth access token
$integrationId = $this->tokenManager->getIntegrationIdByAccessToken($token);
// $integrationId = 5
```

**OAuth integration rate limit configuration:**

```xml
<!-- etc/webapi_rest/di.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:etc/di.xsd">

    <!--
        Override rate limiter per integration
        High-volume integrations get higher limits
    -->
    <type name="Magento\Webapi\Model\RateLimitProcessor">
        <arguments>
            <argument name="integrationLimits" xsi:type="array">
                <item name="integration_5" xsi:type="array">
                    <item name="limit" xsi:type="number">10000</item>
                    <item name="period" xsi:type="number">3600</item>
                </item>
                <item name="integration_12" xsi:type="array">
                    <item name="limit" xsi:type="number">5000</item>
                    <item name="period" xsi:type="number">3600</item>
                </item>
            </argument>
        </arguments>
    </type>

</config>
```

### 8.2 Per-Integration Rate Limits for Marketplace Apps

For marketplace/app store integrations, configure tier-based limits:

```php
<?php
// app/etc/env.php — Marketplace integration tiers
return [
    'rate_limiting' => [
        'rest' => [
            'limit' => 1000,
            'period' => 3600,
        ],
    ],

    // Marketplace-specific limits
    'integration_tiers' => [
        'marketplace_standard' => [
            'limit' => 5000,
            'period' => 3600,
        ],
        'marketplace_premium' => [
            'limit' => 20000,
            'period' => 3600,
        ],
        'marketplace_enterprise' => [
            'limit' => 100000,
            'period' => 3600,
        ],
    ],
];
```

### 8.3 Customer Token vs Admin Token Rate Limiting

Different token types get different rate limit treatment:

| Token Type | Rate Limit Key | Typical Limits |
|------------|---------------|----------------|
| Admin token | Admin user ID | High (trusted operations) |
| Customer token | Customer ID | Medium (per-customer) |
| Integration OAuth | Integration ID | High (bulk operations) |
| Anonymous (no token) | IP address | Low (shared resource) |

### 8.4 webapi_rate_limit_logs Table Schema

```sql
-- Detailed logging table (Adobe Commerce)
CREATE TABLE webapi_rate_limit_logs (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    identifier      VARCHAR(255) NOT NULL,      -- IP, token hash, user ID
    limit_type      ENUM('rest', 'graphql') NOT NULL,
    event_type      ENUM('hit', 'exceeded', 'reset') NOT NULL,
    request_count   INT UNSIGNED NOT NULL DEFAULT 1,
    ip_address      VARCHAR(45) NULL,          -- IPv4 or IPv6
    user_agent      VARCHAR(255) NULL,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_identifier_type (identifier, limit_type),
    INDEX idx_created_at (created_at)
);
```

### 8.5 Querying Rate Limit Logs

```sql
-- Monitor API usage patterns
SELECT
    DATE(created_at) as date,
    limit_type,
    COUNT(*) as total_events,
    SUM(CASE WHEN event_type = 'exceeded' THEN 1 ELSE 0 END) as rate_limit_exceeded,
    COUNT(DISTINCT identifier) as unique_consumers
FROM webapi_rate_limit_logs
WHERE created_at > NOW() - INTERVAL 7 DAY
GROUP BY DATE(created_at), limit_type
ORDER BY date DESC;

-- Find clients frequently hitting rate limits
SELECT
    identifier,
    ip_address,
    COUNT(*) as exceeded_events,
    MAX(request_count) as max_hits_in_window
FROM webapi_rate_limit_logs
WHERE event_type = 'exceeded'
  AND created_at > NOW() - INTERVAL 24 HOUR
GROUP BY identifier, ip_address
HAVING COUNT(*) > 5
ORDER BY exceeded_events DESC;
```

---

## 9. Complete Configuration Example

### 9.1 Full env.php Configuration

```php
<?php
// app/etc/env.php
return [
    // ... other config sections ...

    'system' => [
        'websites' => [
            'base' => [
                'webapi' => [
                    'rate_limiting' => [
                        'rest_rate_limit'   => 1000,
                        'graphql_rate_limit' => 500,
                    ],
                ],
            ],
        ],
    ],

    'rate_limiting' => [
        'rest' => [
            'limit'  => 1000,
            'period' => 3600,
        ],
        'graphql' => [
            'limit'  => 500,
            'period' => 3600,
        ],
    ],
];
```

### 9.2 Complete webapi.xml with Rate Limiting

```xml
<?xml version="1.0"?>
<!-- app/code/Vendor/Catalog/etc/webapi.xml -->
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Webapi/etc/webapi.xsd">

    <!-- Product Search — expensive, moderate limit -->
    <route method="GET" url="/V1/products">
        <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="getList"/>
        <resources>
            <resource ref="Magento_Catalog::products"/>
        </resources>
        <data>
            <item name="rateLimitConfig" xsi:type="array">
                <item name="limit" xsi:type="number">120</item>
                <item name="period" xsi:type="number">60</item>
            </item>
        </data>
    </route>

    <!-- Single Product Read — public, anonymous access -->
    <route method="GET" url="/V1/products/:id">
        <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="get"/>
        <resources>
            <resource ref="anonymous"/>
        </resources>
        <data>
            <item name="rateLimitConfig" xsi:type="array">
                <item name="limit" xsi:type="number">200</item>
                <item name="period" xsi:type="number">60</item>
            </item>
        </data>
    </route>

    <!-- Product Create/Update — authenticated, moderate -->
    <route method="POST" url="/V1/products"/>
    <route method="PUT" url="/V1/products/:id">
        <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="save"/>
        <resources>
            <resource ref="Magento_Catalog::products"/>
        </resources>
        <data>
            <item name="rateLimitConfig" xsi:type="array">
                <item name="limit" xsi:type="number">60</item>
                <item name="period" xsi:type="number">60</item>
            </item>
        </data>
    </route>

    <!-- Admin Price Update — restricted to admin only, IP-allowed -->
    <route method="PUT" url="/V1/products/:id/prices">
        <service class="Magento\Catalog\Api\ProductPriceManagementInterface" method="set"/>
        <resources>
            <resource ref="Magento_Catalog::products"/>
        </resources>
        <data>
            <item name="rateLimitConfig" xsi:type="array">
                <item name="limit" xsi:type="number">30</item>
                <item name="period" xsi:type="number">60</item>
            </item>
            <item name="allowedOrigins" xsi:type="array">
                <item name="ip" xsi:type="string">10.0.0.0/8</item>
                <item name="ip" xsi:type="string">192.168.0.0/16</item>
            </item>
        </data>
    </route>

</routes>
```

---

## 10. Error Handling & Edge Cases

### 10.1 Rate Limit Error Response Format

```json
// HTTP 429 Too Many Requests
{
    "message": "Rate limit exceeded",
    "code": 429,
    "trace": null
}
```

### 10.2 Error Flow Summary

| Scenario | HTTP Code | Response |
|-----------|-----------|----------|
| Limit exceeded | 429 | `{"message": "Rate limit exceeded"}` |
| IP not allowed | 403 | `{"message": "Access denied from IP X.X.X.X"}` |
| Authentication failed | 401 | `{"message": "Consumer is not authorized"}` |
| ACL denied | 403 | `{"message": "ACL resource denied"}` |

### 10.3 Concurrent Request Handling

When multiple requests arrive simultaneously near the rate limit boundary:

```
Timeline (milliseconds):

t=0ms    Request A arrives → count=999/1000 → ALLOWED
t=0ms    Request B arrives → count=1000/1000 → ALLOWED  (checked same time)
t=0ms    Request C arrives → count=1001/1000 → BLOCKED  (429)
t=0ms    Request D arrives → count=1002/1000 → BLOCKED  (429)

Result:
- A and B succeed (counter was 999 when checked)
- C and D get 429 (counter was 1001 when checked)

This is eventual consistency at the counter level — acceptable trade-off.
```

### 10.4 Rate Limit Counter Race Conditions

Magento uses atomic operations for rate limit counters. The counter update uses `INSERT ... ON DUPLICATE KEY UPDATE` pattern for atomic increment:

```php
// Magento\Framework\Webapi\Model\RateLimit\Storage::hit()
// Uses atomic increment to prevent race conditions

$sql = "INSERT INTO webapi_limits
        (identifier, limit_type, hits, window_start, window_duration)
        VALUES (?, ?, 1, ?, ?)
        ON DUPLICATE KEY UPDATE hits = hits + 1";

$this->connection->query($sql, [$identifier, $type, $windowStart, $windowDuration]);
```

---

## 11. Best Practices Summary

### 11.1 Configuration Checklist

| Item | Recommendation |
|------|----------------|
| Global REST limit | 1000 req/hour (adjust for traffic) |
| Global GraphQL limit | 500 req/hour (GraphQL is more expensive) |
| Checkout endpoints | 20-30 req/min (strict) |
| Product read (anonymous) | 100-200 req/min |
| Search endpoints | 60 req/min |
| Auth/token endpoints | 5-10 req/min (strict brute force protection) |
| Bulk export | 500 req/hour (higher period) |

### 11.2 Security Hardening Checklist

| Practice | Implementation |
|----------|----------------|
| Separate limits per integration tier | env.php `integration_tiers` |
| IP allowlisting for admin endpoints | webapi.xml `<allowedOrigins>` |
| Brute force protection on auth endpoints | 5 req/min on `/V1/integration/*/token` |
| Anonymous endpoint rate limiting | Prevent price scraping, catalog enumeration |
| Monitor and alert on rate limit hits | Query `webapi_rate_limit_logs` |
| Layer nginx rate limiting | First-line defense at infrastructure |

### 11.3 Production Recommendations

1. **Start conservative** — lower limits are easier to increase than to fix abuse after the fact
2. **Monitor before limiting** — use `webapi_rate_limit_logs` to understand real traffic patterns first
3. **Implement layered defense** — nginx → Magento webapi.xml → custom plugin
4. **Use per-integration limits** — marketplace apps should have dedicated quotas
5. **Document rate limits** — provide API consumers with clear documentation of limits
6. **Test rate limit behavior** — write integration tests that verify 429 responses

---

## 12. Related Commands

```bash
# Flush rate limit counters (development only)
php bin/magento cache:flush webapi

# Check rate limit configuration
php bin/magento config:show webapi/rate_limiting

# View current rate limit hits
mysql -u magento_user -p -e "SELECT * FROM magento.webapi_limits"

# Monitor logs for rate limit events
tail -f /var/log/magento.log | grep RateLimit
```

---

## 13. Key Takeaways

1. **Magento 2.4.7+** provides native rate limiting via `webapi.xml` and `env.php`
2. **Per-route limits override global limits** — configure specific limits on expensive endpoints
3. **Rate limit key = authentication context** — anonymous uses IP, tokens use user/integration ID
4. **HTTP 429** is returned when limit is exceeded, with `X-RateLimit-*` headers on all responses
5. **IP allowlisting** is configured per-route in `<data><allowedOrigins>` block
6. **Layer your defenses** — nginx for rough limiting, Magento for fine-grained control
7. **Monitor via `webapi_rate_limit_logs`** — catch abuse patterns before they become problems

---

## Cross-References

- **REST/GraphQL API Reference:** `../07-api-development/11-graphql-rest-api.md` — API architecture, endpoint patterns
- **Security & ACL:** `_supplemental/07-security-acl.md` — ACL resources, authorization, endpoint protection
- **Request Flow:** `_supplemental/02-request-flow.md` — How Magento processes WebAPI requests
- **Message Queue:** `_supplemental/16-message-queue.md` — Async API patterns for bulk operations

---

*Magento 2.4.8 WebAPI Development — Rate Limiting & Quota Management*