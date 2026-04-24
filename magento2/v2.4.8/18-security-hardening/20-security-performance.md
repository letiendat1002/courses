---
title: "20 - Security & Performance"
description: "Deep dive into Magento 2.4.8 security patterns (CSRF, XSS, SQL injection, password hashing) and performance optimization (Redis, Varnish, MySQL, CDN)"
tags:
  - magento2
  - security
  - performance
  - csrf
  - xss
  - redis
  - varnish
rank: 20
pathways:
  - magento2-deep-dive
---

# Security & Performance

Magento 2.4.8 provides robust built-in security mechanisms and performance infrastructure. This article covers real security patterns and performance optimization techniques available in open-source Magento 2.4.8.

---

## 1. CSRF Protection

### Form Key Validation

Magento 2.4.8 implements CSRF protection via form keys. Every POST form must include a valid form key.

#### In PHTML Templates

```php
<?php
/** @var \Magento\Framework\Escaper $escaper */
/** @var \Magento\Framework\View\Element\Template $block */
?>
<form action="<?= $escaper->escapeUrl($block->getFormAction()) ?>" method="post">
    <input name="form_key" type="hidden" value="<?= $escaper->escapeHtmlAttr($block->getFormKey()) ?>" />
    <!-- other fields -->
</form>
```

#### In UI Components

```xml
<field name="custom_field">
    <argument name="data" xsi:type="array">
        <item name="config" xsi:type="array">
            <item name="formElement" xsi:type="string">input</item>
            <item name="source" xsi:type="string">custom_source</item>
        </item>
    </argument>
</field>
```

The form key is automatically validated by `Magento\Framework\Session\CsrfValidator`.

#### Custom CSRF Validation in Controllers

```php
// app/code/Vendor/Module/Controller/Adminhtml/Action/Save.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Controller\Adminhtml\Action;

use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Framework\App\Request\CsrfValidator;
use Magento\Framework\App\RequestInterface;

class Save extends \Magento\Backend\App\Action implements HttpPostActionInterface
{
    /**
     * Custom CSRF token match
     *
     * @param RequestInterface $request
     * @return bool
     */
    protected function _validateFormKey(RequestInterface $request): bool
    {
        $formKey = $request->getParam('form_key');
        $customToken = $request->getParam('custom_token');

        if ($formKey === $this->_formKeyValidator->validate($request)) {
            return true;
        }

        // Additional custom validation
        return $customToken === $this->getValidToken();
    }

    /**
     * Execute save action
     *
     * @return \Magento\Framework\Controller\ResultInterface
     */
    public function execute(): \Magento\Framework\Controller\ResultInterface
    {
        if (!$this->_validateFormKey($this->getRequest())) {
            $this->_redirect('*/*/denied');
            return $this->resultFactory->create(\Magento\Framework\Controller\ResultFactory::TYPE_REDIRECT);
        }

        // Process the save
        return $this->resultFactory->create(\Magento\Framework\Controller\ResultFactory::TYPE_PAGE);
    }
}
```

---

## 2. XSS Prevention

### Using the Escaper Class

Magento provides `Magento\Framework\Escaper` for safe output in templates.

#### In PHTML Templates

```php
<?php
/** @var \Magento\Framework\Escaper $escaper */
?>

<!-- Escape HTML content -->
<p><?= $escaper->escapeHtml($block->getUserInput()) ?></p>

<!-- Escape HTML attributes -->
<input type="text" value="<?= $escaper->escapeHtmlAttr($block->getInputValue()) ?>" />

<!-- Escape URL -->
<a href="<?= $escaper->escapeUrl($block->getProductUrl()) ?>">Link</a>

<!-- Escape JSON in HTML data attributes -->
<div data-config="<?= $escaper->escapeHtmlAttr($block->getJsonConfig()) ?>"></div>

<!-- Allow only specific HTML tags -->
<p><?= $escaper->escapeHtml($userContent, ['b', 'i', 'u']) ?></p>
```

#### In Block Classes

```php
// app/code/Vendor/Module/Block/Widget.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Block;

use Magento\Framework\View\Element\Template;
use Magento\Framework\Escaper;

class Widget extends Template
{
    /**
     * Get safe user data for display
     *
     * @return string
     */
    public function getUserDisplayName(): string
    {
        /** @var Escaper $escaper */
        $escaper = $this->getEscaper();
        return $escaper->escapeHtml($this->_scopeConfig->getValue('customer/demo/display_name'));
    }
}
```

#### In JavaScript Templates

```javascript
define([
    'Magento_Ui/js/modal/confirm',
    'mage/translate'
], function(confirmation, $t) {
    'use strict';

    return function(config) {
        // Use mage/translate for JS string escaping
        var safeMessage = $t('Confirm action for: %1').replace('%1', config.itemName);

        confirmation({
            title: $t('Confirm'),
            content: safeMessage,
            actions: {
                confirm: function() {
                    // Process
                }
            }
        });
    };
});
```

---

## 3. SQL Injection Prevention

### Parameterized Queries via ResourceModels

Magento 2.4.8 uses Zend_DB prepared statements automatically through the ResourceModel pattern.

#### Safe Query via Collection

```php
// app/code/Vendor/Module/Model/ResourceModel/Order/Collection.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\ResourceModel\Order;

use Magento\Sales\Model\ResourceModel\Order\Collection as SalesCollection;

class Collection extends SalesCollection
{
    /**
     * Filter by safe status using bound parameters
     *
     * @param string $status
     * @return $this
     */
    public function filterByStatus(string $status): self
    {
        // Parameterized query - safe from SQL injection
        $this->addFieldToFilter('status', ['eq' => $status]);
        return $this;
    }

    /**
     * Filter by multiple statuses safely
     *
     * @param array $statuses
     * @return $this
     */
    public function filterByStatuses(array $statuses): self
    {
        $this->addFieldToFilter('status', ['in' => $statuses]);
        return $this;
    }
}
```

#### Safe Query with JOIN

```php
// app/code/Vendor/Module/Model/ResourceModel/Product/Link/Collection.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\ResourceModel\Product\Link;

use Magento\Catalog\Model\ResourceModel\Product\Link\Collection as ProductLinkCollection;

class Collection extends ProductLinkCollection
{
    /**
     * Join with custom table using binding
     *
     * @param string $sku
     * @param int $websiteId
     * @return $this
     */
    public function joinLinkedProducts(string $sku, int $websiteId): self
    {
        $connection = $this->getConnection();

        // Use bindings for safety
        $select = $this->getSelect();
        $select->join(
            ['link_table' => $this->getTable('catalog_product_link')],
            $connection->quoteInto(
                'link_table.product_id = e.entity_id AND link_table.linked_product_id = ?',
                $this->getProductIdBySku($sku)
            ),
            ['link_type_id', 'qty']
        );

        $select->where('link_table.website_id = ?', $websiteId);

        return $this;
    }
}
```

---

## 4. File Upload Security

### Validating Uploaded Files

```php
// app/code/Vendor/Module/Controller/Adminhtml/Upload/Save.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Controller\Adminhtml\Upload;

use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Framework\Controller\ResultFactory;

class Save extends \Magento\Backend\App\Action implements HttpPostActionInterface
{
    /**
     * @var \Magento\Framework\File\UploaderFactory
     */
    private $uploaderFactory;

    /**
     * @param \Magento\Backend\App\Action\Context $context
     * @param \Magento\Framework\File\UploaderFactory $uploaderFactory
     */
    public function __construct(
        \Magento\Backend\App\Action\Context $context,
        \Magento\Framework\File\UploaderFactory $uploaderFactory
    ) {
        parent::__construct($context);
        $this->uploaderFactory = $uploaderFactory;
    }

    /**
     * Execute file upload with validation
     *
     * @return \Magento\Framework\Controller\ResultInterface
     */
    public function execute(): \Magento\Framework\Controller\ResultInterface
    {
        $result = $this->resultFactory->create(ResultFactory::TYPE_JSON);

        try {
            $uploader = $this->uploaderFactory->create(['fileId' => 'upload']);
            $uploader->setAllowedExtensions(['jpg', 'jpeg', 'png', 'gif', 'pdf']);
            $uploader->setFilesDispersion(false);
            $uploader->setAllowCreateFolders(true);
            $uploader->setAllowedMimeTypes([
                'image/jpeg',
                'image/png',
                'image/gif',
                'application/pdf'
            ]);

            $resultUpload = $uploader->save($this->getUploadDir());

            $result->setData([
                'success' => true,
                'file' => $resultUpload['file']
            ]);
        } catch (\Exception $e) {
            $result->setData([
                'success' => false,
                'error' => $e->getMessage()
            ]);
        }

        return $result;
    }

    /**
     * Get upload directory
     *
     * @return string
     */
    private function getUploadDir(): string
    {
        return $this->_filesystem->getDirectoryRead(\Magento\Framework\App\Filesystem\DirectoryList::MEDIA)
            ->getAbsolutePath('vendor_module/upload');
    }
}
```

---

## 5. Admin Session Security

### Session Lifetime Configuration

Configure admin session timeout in `env.php`:

```php
<?php
return [
    'session' => [
        'save' => 'db',
        'redis' => [
            'host' => '127.0.0.1',
            'port' => 6379,
            'timeout' => 2.5,
            'password' => '',
            'database' => 1
        ]
    ],
    'admin' => [
        'session_lifetime' => 3600, // 1 hour in seconds
        'security' => [
            'admin_base_url' => 'admin',
            'use_expired_csrf_token' => false
        ]
    ],
    // ... other config
];
```

### Admin Route Security

```xml
<!-- app/code/Vendor/Module/etc/adminhtml/routes.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:App/etc/routes.xsd">
    <router id="admin">
        <route id="vendor_module" frontName="vendor-module">
            <module name="Vendor_Module" before="Magento_Backend" />
        </route>
    </router>
</config>
```

### Two-Factor Authentication

Magento 2.4.8 supports 2FA via `Magento\TwoFactorAuth` module. Configuration:

```xml
<!-- app/code/Vendor/Module/etc/env.php -->
<?php
return [
    'twofactorauth' => [
        'general' => [
            'force_providers' => ['google'],
            'window' => 5 // Token window in seconds
        ]
    ]
];
```

---

## 6. Password Hashing

### Using Magento Crypt Library

```php
// app/code/Vendor/Module/Model/UserPassword.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Framework\Crypt\CryptInterface;

class UserPassword
{
    /**
     * @var CryptInterface
     */
    private $crypt;

    /**
     * @param CryptInterface $crypt
     */
    public function __construct(CryptInterface $crypt)
    {
        $this->crypt = $crypt;
    }

    /**
     * Hash a password using Magento's algorithm
     *
     * @param string $password
     * @return string
     */
    public function hash(string $password): string
    {
        return $this->crypt->hash($password);
    }

    /**
     * Verify password against hash
     *
     * @param string $password
     * @param string $hash
     * @return bool
     */
    public function verify(string $password, string $hash): bool
    {
        return $this->crypt->verify($password, $hash);
    }
}
```

```xml
<!-- app/code/Vendor/Module/etc/di.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Vendor\Module\Model\UserPassword">
        <arguments>
            <argument name="crypt" xsi:type="object">Magento\Framework\Crypt</argument>
        </arguments>
    </type>
</config>
```

---

## 7. Redis Cache Configuration

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

## 8. Varnish Configuration

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

## 9. MySQL Query Optimization

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

## 10. CDN Configuration

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