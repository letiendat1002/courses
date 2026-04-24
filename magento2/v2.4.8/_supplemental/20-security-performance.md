---
title: "20 - Performance Optimization"
description: "Deep dive into Magento 2.4.8 performance optimization: Redis cache configuration, Varnish full-page cache, MySQL query optimization, and CDN setup."
tags:
  - magento2
  - performance
  - redis
  - varnish
  - mysql
  - cdn
rank: 20
pathways:
  - magento2-deep-dive
duplicates_topic: 07
see_also:
  - security: "CSRF, XSS, SQL injection, ACL, and encryption patterns are covered in depth in [07-security-acl.md](./07-security-acl.md)"
---

# Performance Optimization

> **Note on scope:** Security fundamentals (CSRF, XSS, SQL injection, password hashing, file upload, admin session, 2FA) are comprehensively covered in **[07-security-acl.md](./07-security-acl.md)**. This article focuses exclusively on performance optimization: Redis cache, Varnish FPC, MySQL tuning, and CDN configuration.

---

## 1. Redis Cache Configuration

### env.php Cache Configuration

```php
<?php
// app/etc/env.php
return [
    'cache' => [
        'frontend' => [
            'default' => [
                'backend' => 'Magento\Framework\Cache\Backend\Redis',
                'backend_options' => [
                    'server' => '127.0.0.1',
                    'port' => 6379,
                    'database' => 0,
                    'timeout' => 2.5,
                    'retry_delay' => 1,
                    'lifetimelimit' => 2,
                    'password' => ''
                ]
            ],
            'page_cache' => [
                'backend' => 'Magento\Framework\Cache\Backend\Redis',
                'backend_options' => [
                    'server' => '127.0.0.1',
                    'port' => 6379,
                    'database' => 1,
                    'timeout' => 2.5,
                    'compression_threshold' => 2048,
                    'compression_lib' => 'gzip'
                ]
            ]
        ],
        'type' => [
            'layout' => ['frontend' => 'default'],
            'block_html' => ['frontend' => 'default'],
            'full_page' => ['frontend' => 'page_cache']
        ]
    ]
];
```

### Cache Flush Command

```bash
# Flush Redis cache
redis-cli FLUSHDB

# Or via Magento CLI
bin/magento cache:flush
```

---

## 2. Varnish Configuration

### default.vcl Configuration

```vcl
vcl 4.1;

import std;

backend default {
    .host = "127.0.0.1";
    .port = "8080";
    .first_byte_timeout = 86400;
    .probe = {
        .url = "/health_check";
        .timeout = 2s;
        .interval = 5s;
        .expected_response = 200;
    }
}

sub vcl_recv {
    # Remove cookies for static assets - cache them
    if (req.url ~ "\.(jpg|jpeg|png|gif|css|js|ico|svg|woff|woff2|ttf|eot)$") {
        unset req.http.Cookie;
        return (hash);
    }

    # Remove Google Analytics cookies
    if (req.http.Cookie) {
        if (req.http.Cookie ~ "__utm") {
            set req.http.Cookie = regsuball(req.http.Cookie, "__utm[^=]*=[^;]*;?\s*", "");
        }
    }

    # RemoveSID cookie for pages where it's not needed
    if (req.url ~ "\?(utm_|fbclid|gclid)") {
        set req.url = regcol(req.url, "", "1");
    }
}

sub vcl_hash {
    hash_data(req.url);
    if (req.http.host) {
        hash_data(req.http.host);
    } else {
        hash_data(server.ip);
    }
}

sub vcl_backend_response {
    # Set cache duration
    if (beresp.ttl > 0s) {
        set beresp.http.X-Cache = "HIT";
    } else {
        set beresp.http.X-Cache = "MISS";
    }

    # Gzip compression
    if (beresp.http.Content-Type ~ "text/(html|css|javascript|xml)") {
        set beresp.do_gzip = true;
    }
}

sub vcl_deliver {
    # Add cache metadata headers
    if (obj.hits > 0) {
        set resp.http.X-Cache-Hits = obj.hits;
    }
    unset resp.http.X-Powered-By;
    set resp.http.X-Frame-Options = "SAMEORIGIN";
}
```

### Varnish Health Check

```bash
# Check Varnish status
 varnishadm -S /etc/varnish/secret -T 127.0.0.1:6082 status

# Replay VCL
 varnishadm -S /etc/varnish/secret -T 127.0.0.1:6082 vcl.load myconfig /var/www/html/default.vcl
 varnishadm -S /etc/varnish/secret -T 127.0.0.1:6082 vcl.use myconfig
```

---

## 3. MySQL Query Optimization

### Enable Query Logging

```php
<?php
// app/etc/env.php
return [
    'db' => [
        'connection' => [
            'default' => [
                'host' => '127.0.0.1',
                'port' => 3306,
                'dbname' => 'magento_db',
                'username' => 'magento_user',
                'password' => 'password',
                'model' => 'mysql4',
                'engine' => 'innodb',
                'initStatements' => 'SET NAMES utf8mb4'
            ]
        ],
        'slave' => [
            'host' => '127.0.0.1',
            'port' => 3306,
            'dbname' => 'magento_db',
            'username' => 'magento_slave_user',
            'password' => 'password',
            'model' => 'mysql4',
            'engine' => 'innodb'
        ]
    ],
    'system' => [
        'default' => [
            'system' => [
                'dev' => [
                    'query_log' => 1,
                    'query_log_depth' => 3
                ]
            ]
        ]
    ]
];
```

### Analyze Slow Queries

```sql
-- Enable slow query log in MySQL
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
SET GLOBAL long_query_time = 2;

-- Find slow queries
SELECT * FROM information_schema.PROCESSLIST
WHERE Command != 'Sleep' AND Time > 5;

-- Check table indexes
SHOW INDEX FROM sales_order;

-- Use EXPLAIN for query analysis
EXPLAIN SELECT e.*, s.*
FROM catalog_product_entity e
LEFT JOIN catalog_product_entity_int s ON e.entity_id = s.entity_id
WHERE s.store_id = 0 AND s.attribute_id = 97;
```

### Index Creation

```xml
<!-- app/code/Vendor/Module/Setup/UpgradeSchema.php -->
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup;

use Magento\Framework\Setup\UpgradeSchemaInterface;
use Magento\Framework\Setup\SchemaSetupInterface;
use Magento\Framework\Setup\ModuleContextInterface;

class UpgradeSchema implements UpgradeSchemaInterface
{
    /**
     * @param SchemaSetupInterface $setup
     * @param ModuleContextInterface $context
     */
    public function upgrade(
        SchemaSetupInterface $setup,
        ModuleContextInterface $context
    ): void {
        $setup->startSetup();

        // Add composite index for order lookup
        $setup->getConnection()->addIndex(
            $setup->getTable('sales_order'),
            'IDX_ORDER_STATUS_CREATED_AT',
            ['status', 'created_at'],
            \Magento\Framework\DB\Adapter\AdapterInterface::INDEX_TYPE_INDEX
        );

        // Add fulltext index for search
        $setup->getConnection()->addIndex(
            $setup->getTable('catalog_product_entity_text'),
            'FTI_PRODUCT_DESCRIPTION',
            ['value'],
            \Magento\Framework\DB\Adapter\AdapterInterface::INDEX_TYPE_FULLTEXT
        );

        $setup->endSetup();
    }
}
```

---

## 4. CDN Configuration

### Static Content CDN

```php
<?php
// app/etc/env.php
return [
    'cache' => [
        'frontend' => [
            'default' => [
                'backend' => 'Magento\Framework\Cache\Backend\Redis',
                'backend_options' => [
                    'server' => '127.0.0.1',
                    'port' => 6379
                ]
            ]
        ]
    ],
    'remote_storage' => [
        'enabled' => true,
        'dir' => [
            'directories' => [
                [
                    'path' => 'media',
                    'rewrite_rules' => [
                        ['pattern' => '^media/', 'replacement' => 'https://cdn.example.com/media/']
                    ]
                ],
                [
                    'path' => 'static/',
                    'rewrite_rules' => [
                        ['pattern' => '^static/', 'replacement' => 'https://cdn.example.com/static/']
                    ]
                ]
            ]
        ]
    ]
];
```

### Nginx CDN Configuration

```nginx
# Upstream CDN servers
upstream cdn_backend {
    server 127.0.0.1:8080;
}

server {
    listen 443 ssl http2;
    server_name cdn.example.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    location /media/ {
        proxy_pass http://cdn_backend;
        proxy_cache_valid 200 7d;
        proxy_cache_valid 404 1m;
        expires 7d;
        add_header X-Cache-Status "HIT" always;
    }

    location /static/ {
        proxy_pass http://cdn_backend;
        proxy_cache_valid 200 1y;
        expires 1y;
        add_header X-Cache-Status "HIT" always;
    }
}
```

---

## Summary: Real Magento 2.4.8 Security & Performance

| Pattern | Implementation |
|---------|----------------|
| **CSRF Protection** | Form keys via `form_key` input, validated by `CsrfValidator` |
| **XSS Prevention** | `Magento\Framework\Escaper` - `escapeHtml()`, `escapeHtmlAttr()`, `escapeUrl()` |
| **SQL Injection** | Parameterized queries via Collection `addFieldToFilter()` |
| **File Upload** | `UploaderFactory` with allowed extensions and MIME types |
| **Session Security** | Configure in `env.php` with `session_lifetime`, use Redis backend |
| **Password Hashing** | `Magento\Framework\Crypt` interface |
| **Redis Cache** | `Magento\Framework\Cache\Backend\Redis` in `env.php` |
| **Varnish** | `default.vcl` with proper backend and cache headers |
| **MySQL Optimization** | Add indexes via `UpgradeSchema`, use `EXPLAIN` to analyze |
| **CDN** | Configure in `env.php` `remote_storage` or via Nginx |

---

## References

- [Magento Security Best Practices](https://developer.adobe.com/commerce/docs/cloud/configure/security/csrf-tokens/)
- [Magento Performance](https://developer.adobe.com/commerce/docs/cloud/configure/setup/standard-logging/)
- [Magento Cache Configuration](https://experienceleague.adobe.com/docs/commerce-operations/configuration-guide/cache/configure-redis.html)
- [Varnish Configuration](https://experienceleague.adobe.com/docs/commerce-operations/configuration-guide/cache/configure-varnish.html)

(End of file - total 856 lines)