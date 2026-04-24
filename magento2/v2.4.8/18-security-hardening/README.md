# Topic 18: Security Hardening — Production Patterns

**Goal:** Master the production security patterns missing from the course — SQL injection prevention, CSRF for custom AJAX, Redis-based rate limiting, secret management, 2FA enrollment, and security headers. These are the patterns that separate staging-ready code from production-hardened code.

---

## Topics Covered

- SQL injection via `patchWhere`, raw `getLoader()->update()`, and safe binding patterns via SearchCriteria
- CSRF protection for custom AJAX endpoints — form key validation, `_cps` helper, and custom form token handling
- Redis-based rate limiting — `[limiter]` env.php config, per-user/IP throttling middleware, Magento's State class
- Secret management — env.php credential patterns, `encrypted` config field type, Magento\EncryptionKey
- 2FA enrollment for admin, session fixation prevention, cookie security flags (HttpOnly, Secure, SameSite)
- Security headers — CORS configuration, Content-Security-Policy, X-Frame-Options for admin/API routes

---

## What You Need to Know

- Topic 04 (Data Layer) — repositories, SearchCriteria, declarative schema
- Topic 06 (Admin UI) — admin routes, ACL, form handling
- Topic 07 (API Development) — REST endpoints, authentication
- How to read Magento's `di.xml` and `acl.xml` files

---

## What You'll Learn

- Identify vulnerable vs safe data access patterns in Magento
- Implement parameterized queries using Magento's binding APIs
- Protect custom AJAX endpoints with form key validation
- Configure and use Redis rate limiting via env.php
- Store credentials and secrets using Magento's encrypted config system
- Configure admin 2FA enrollment for user accounts
- Apply security headers to HTTP responses for admin and API routes

---

## Topic Mapping

| Section | Skill Developed | Prerequisite |
|---------|----------------|--------------|
| Day 1 — SQL Injection | Identify/fix injection via patchWhere and raw queries | Topic 04 |
| Day 2 — CSRF Protection | Protect AJAX endpoints with form key validation | Topic 06 |
| Day 3 — Rate Limiting | Configure Redis limiter via env.php + middleware | Topic 07 |
| Day 4 — Secret Management | Use encrypted config fields and env.php patterns | Topics 04, 06 |
| Day 5 — Headers + 2FA | Apply CSP, CORS, X-Frame-Options, 2FA enrollment | Topics 06, 07 |

---

## Reference Exercises

- **Exercise 20.1:** Audit a repository for SQL injection — find one `patchWhere` or raw update and replace with binding
- **Exercise 20.2:** Add form key validation to a custom AJAX endpoint
- **Exercise 20.3:** Configure Redis rate limiter via env.php `[limiter]` section
- **Exercise 20.4:** Create an encrypted system config field and verify credentials are not exposed
- **Exercise 20.5:** Enforce 2FA for an admin user programmatically
- **Exercise 20.6:** Add CORS and CSP headers to a custom REST endpoint
- **Exercise 20.7:** Implement session fixation prevention on a custom admin controller

---

## Completion Criteria

- [ ] Repository uses parameterized queries via `bind()` — no string interpolation in SQL
- [ ] Custom AJAX endpoint validates form key before processing
- [ ] Redis rate limiter configured and responding with 429 on exceeded limits
- [ ] Sensitive config fields use `encrypted` type — credentials not visible in admin UI
- [ ] Admin user can be enrolled in 2FA programmatically
- [ ] Custom API endpoint sends CSP and X-Frame-Options headers
- [ ] All admin controllers set HttpOnly, Secure, SameSite cookies
- [ ] No hardcoded credentials in any module file

---

## Topics

---

### Topic 1: SQL Injection — Vulnerable Patterns vs Safe Patterns

**Why This Topic Exists:**

Magento's data layer provides safe abstraction via repositories and SearchCriteria. But when developers drop to raw SQL via `patchWhere` or resource model `getLoader()->update()`, they often replicate the exact conditions that SQL injection exploits. This topic teaches you to recognize and fix both patterns.

---

### SQL Injection via `patchWhere`

`patchWhere` is the dangerous cousin of `addFieldToFilter`. It lets you inject raw SQL `WHERE` clauses directly into collection queries. When user input reaches this method without sanitization, you have SQL injection.

**Vulnerable Pattern — Never Do This:**

```php
<?php
declare(strict_types=1);

namespace Training\Example\Model\Repository;

use Training\Example\Model\ResourceModel\ItemResourceModel;
use Magento\Framework\Model\ResourceModel\Db\Collection\AbstractCollection;

class ItemRepository
{
    public function findByCustomerInput(
        AbstractCollection $collection,
        string $customerName,
        string $statusFilter
    ): void {
        // VULNERABLE — string interpolation in WHERE clause
        // If $customerName = "' OR '1'='1", this injects arbitrary SQL
        $collection->getSelect()->where("name = '" . $customerName . "'");

        // VULNERABLE — statusFilter could contain UNION, subqueries, etc.
        $collection->getSelect()->where("status = " . $statusFilter);
    }
}
```

**Safe Pattern — Use Binding Parameters:**

```php
<?php
declare(strict_types=1);

namespace Training\Example\Model\Repository;

use Training\Example\Model\ResourceModel\ItemResourceModel;
use Magento\Framework\Model\ResourceModel\Db\Collection\AbstractCollection;
use Magento\Framework\DB\Adapter\AdapterInterface;

class ItemRepository
{
    public function findByCustomerInput(
        AbstractCollection $collection,
        string $customerName,
        string $statusFilter
    ): void {
        $connection = $collection->getConnection();

        // SAFE — binding via `where` with placeholder
        // Magento's DB adapter escapes the value automatically
        $collection->getSelect()->where(
            'name = ?',
            $customerName
        );

        // SAFE — for integer/status values, cast explicitly
        $collection->getSelect()->where(
            'status = ?',
            (int) $statusFilter
        );
    }
}
```

**Even Safer — Use SearchCriteria with Filters:**

```php
<?php
declare(strict_types=1);

namespace Training\Example\Model\Repository;

use Training\Example\Api\Data\ItemInterface;
use Training\Example\Api\ItemRepositoryInterface;
use Training\Example\Model\ItemFactory;
use Training\Example\Model\ResourceModel\Item as ItemResource;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Framework\Api\FilterBuilder;
use Magento\Framework\Api\SearchResultsInterface;
use Magento\Framework\Exception\NoSuchEntityException;

class ItemRepository implements ItemRepositoryInterface
{
    public function __construct(
        private readonly ItemFactory $itemFactory,
        private readonly ItemResource $resource,
        private readonly SearchCriteriaBuilder $searchCriteriaBuilder,
        private readonly FilterBuilder $filterBuilder,
    ) {}

    public function findByNameAndStatus(string $name, int $status): SearchResultsInterface
    {
        // SAFE — SearchCriteria uses binding internally
        // No string interpolation, no raw SQL exposure
        $filters = [];

        $filters[] = $this->filterBuilder
            ->setField('name')
            ->setValue($name)
            ->setConditionType('eq')
            ->create();

        $filters[] = $this->filterBuilder
            ->setField('status')
            ->setValue($status)
            ->setConditionType('eq')
            ->create();

        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilters($filters)
            ->create();

        return $this->getList($searchCriteria);
    }
}
```

---

### SQL Injection via ResourceModel `getLoader()->update()`

The resource model's direct `update()` method accepts an array of field-value pairs and a WHERE clause. If the WHERE clause uses raw string interpolation, you have injection.

**Vulnerable Pattern:**

```php
<?php
declare(strict_types=1);

namespace Training\Custom\Model\ResourceModel;

use Magento\Framework\Model\ResourceModel\Db\AbstractDb;

class ProductLog extends AbstractDb
{
    protected function _construct(): void
    {
        $this->_init('product_log', 'entity_id');
    }

    public function markAsProcessed(string $sku): int
    {
        $connection = $this->getConnection();

        // VULNERABLE — $sku interpolated directly into SQL
        // $sku = "'; DELETE FROM product_log; --" wipes the table
        $whereClause = "sku = '" . $sku . "' AND processed = 0";

        return $connection->update(
            $this->getTable('product_log'),
            ['processed' => 1, 'processed_at' => time()],
            $whereClause
        );
    }
}
```

**Safe Pattern — Binding with Array Conditions:**

```php
<?php
declare(strict_types=1);

namespace Training\Custom\Model\ResourceModel;

use Magento\Framework\Model\ResourceModel\Db\AbstractDb;

class ProductLog extends AbstractDb
{
    protected function _construct(): void
    {
        $this->_init('product_log', 'entity_id');
    }

    public function markAsProcessed(string $sku): int
    {
        $connection = $this->getConnection();

        // SAFE — bind values via array, framework escapes them
        $bind = [
            'processed' => 1,
            'processed_at' => time(),
        ];

        // SAFE — where clause uses placeholders with separate bind array
        $where = [
            'sku = ?' => $sku,
            'processed = ?' => 0,
        ];

        return $connection->update(
            $this->getTable('product_log'),
            $bind,
            $where
        );
    }
}
```

**Safe Pattern — Using the Adapter's `bind()` Method Explicitly:**

```php
<?php
declare(strict_types=1);

namespace Training\Custom\Model\ResourceModel;

use Magento\Framework\Model\ResourceModel\Db\AbstractDb;

class ProductLog extends AbstractDb
{
    public function markAsProcessedSafe(string $sku): int
    {
        $connection = $this->getConnection();

        // Explicit binding — use this when you need maximum clarity
        $sql = 'UPDATE ' . $connection->quoteIdentifier('product_log') .
               ' SET processed = :processed, processed_at = :processed_at' .
               ' WHERE sku = :sku AND processed = :not_processed';

        $bind = [
            'processed' => 1,
            'processed_at' => time(),
            'sku' => $sku,
            'not_processed' => 0,
        ];

        return $connection->query($sql, $bind)->rowCount();
    }
}
```

---

### VULNERABLE: `load()` with Custom WHERE (ResourceModel `select()` override)

```php
<?php
declare(strict_types=1);

namespace Training\Vulnerable\Model\ResourceModel;

use Magento\Framework\Model\ResourceModel\Db\AbstractDb;

class OrderItem extends AbstractDb
{
    protected function _construct(): void
    {
        $this->_init('sales_order_item', 'item_id');
    }

    // WRONG — overriding load to use raw SQL
    public function loadByOrderId(\Magento\Framework\Model\AbstractModel $model, int $orderId): void
    {
        $select = $this->getConnection()->select()
            ->from($this->getMainTable())
            ->where('order_id = ?', $orderId); // Safe — uses placeholder

        $data = $this->getConnection()->fetchRow($select);
        if ($data) {
            $model->setData($data);
        }
    }
}
```

> **Pro Tip:** The above `loadByOrderId` is actually SAFE because it uses `?` placeholder. The danger comes when developers write `where('order_id = ' . $orderId)` (integer concatenation is common but still dangerous for larger inputs).

---

### Definition of Done — SQL Injection

- [ ] No raw SQL strings with string interpolation (`"SELECT ... WHERE x = '" . $var . "'"`)
- [ ] All `patchWhere` calls use `?` placeholders with bound values
- [ ] All `connection->update()` calls use array-based WHERE conditions with placeholders
- [ ] Integer inputs are cast `(int)` before use in SQL
- [ ] Code passes `phpcs --standard=MEQP2` with no warnings on SQL-related rules

---

### Topic 2: CSRF Protection — Form Keys for Custom AJAX Endpoints

**Why This Topic Exists:**

Topic 06 covers `_isAllowed()` ACL for admin controllers. But custom AJAX endpoints that process state-changing operations (delete, update, submit) are often left without CSRF protection. Magento's form key mechanism is the standard solution — and it must be applied correctly to custom AJAX routes.

---

### How Magento's Form Key Works

Magento's form key system involves three components:
1. **Form key input** — rendered in forms as hidden input `form_key`
2. **Session validation** — `FormKey` model validates the submitted key against the session
3. **Whitelist** — certain routes are excluded (but your custom AJAX route is NOT excluded by default)

For admin AJAX endpoints, you must manually validate the form key.

---

### The `_cps` Helper Pattern

`_cps` is `Magento\Framework\Data\Form\FormKey` — the form key model. The standard validation pattern in admin controllers:

```php
<?php
declare(strict_types=1);

namespace Training\Ajax\Controller\Adminhtml\Item;

use Magento\Backend\App\Action;
use Magento\Framework\Data\Form\FormKey;
use Magento\Framework\Exception\LocalizedException;

class Delete extends Action
{
    public function __construct(
        \Magento\Backend\App\Action\Context $context,
        private readonly FormKey $formKey,
    ) {
        parent::__construct($context);
    }

    public function execute(): \Magento\Framework\App\Response\Http
    {
        $resultRedirect = $this->resultRedirectFactory->create();

        // VULNERABLE WITHOUT THIS CHECK — CSRF attack possible
        if (!$this->formKey->getFormKey() ||
            $this->getRequest()->getParam('form_key') !== $this->formKey->getFormKey()
        ) {
            throw new LocalizedException(
                __('Invalid form key. Please refresh the page and try again.')
            );
        }

        $itemId = (int) $this->getRequest()->getParam('item_id');
        if ($itemId <= 0) {
            throw new LocalizedException(__('Invalid item ID.'));
        }

        // Safe to proceed — CSRF check passed
        $this->itemRepository->deleteById($itemId);

        $this->messageManager->addSuccessMessage(__('Item deleted successfully.'));
        return $resultRedirect->setPath('*/*/index');
    }

    protected function _isAllowed(): bool
    {
        return $this->_authorization->isAllowed('Training_Ajax::item_delete');
    }
}
```

---

### AJAX Endpoint CSRF Pattern (No Page Reload)

For AJAX endpoints that shouldn't trigger a page reload, you need to:

1. Include the form key in the AJAX request (header or body)
2. Validate server-side before processing

**Frontend (JavaScript sending the form key):**

```javascript
define([
    'Magento_Ui/js/model/messageList',
    'mage/cookies'
], function (messageList) {
    'use strict';

    return {
        deleteItem: function (itemId) {
            var formKey = $.cookie('form_key');

            return fetch(BASE_URL + 'training/ajax/delete', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'X-Form-Key': formKey  // Send via custom header
                },
                body: JSON.stringify({ item_id: itemId })
            })
            .then(function (response) {
                if (response.ok) {
                    messageList.addSuccessMessage({ message: 'Item deleted.' });
                    this.reloadGrid();
                } else {
                    messageList.addErrorMessage({ message: 'Deletion failed.' });
                }
            }.bind(this));
        }
    };
});
```

**Backend Validation:**

```php
<?php
declare(strict_types=1);

namespace Training\Ajax\Controller\Adminhtml\Ajax;

use Magento\Framework\Controller\Result\JsonFactory;
use Magento\Framework\Data\Form\FormKey\Validator;

class Delete extends \Magento\Framework\App\Action\Action
{
    public function __construct(
        \Magento\Framework\App\Action\Context $context,
        private readonly JsonFactory $resultJsonFactory,
        private readonly FormKey\Validator $formKeyValidator,
    ) {
        parent::__construct($context);
    }

    public function execute(): \Magento\Framework\App\Response\Http
    {
        $result = $this->resultJsonFactory->create();

        // Validate form key — protects against CSRF on AJAX endpoint
        if (!$this->formKeyValidator->validate($this->getRequest())) {
            return $result->setData([
                'success' => false,
                'message' => __('Invalid form key. Please refresh and try again.')
            ])->setHttpResponseCode(403);
        }

        $itemId = (int) $this->getRequest()->getParam('item_id');
        if ($itemId <= 0) {
            return $result->setData([
                'success' => false,
                'message' => __('Invalid item ID.')
            ])->setHttpResponseCode(400);
        }

        try {
            $this->itemRepository->deleteById($itemId);
            return $result->setData(['success' => true]);
        } catch (\Exception $e) {
            return $result->setData([
                'success' => false,
                'message' => $e->getMessage()
            ])->setHttpResponseCode(500);
        }
    }
}
```

**`di.xml` — No special config needed, just inject the Validator:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- FormKey\Validator is already public in Magento framework -->
    <!-- No preference needed — just inject Magento\Framework\Data\Form\FormKey\Validator -->

</config>
```

---

### Custom AJAX Without Admin Context (Frontend AJAX)

If your AJAX endpoint is in the `frontend` area (not `adminhtml`), you still need CSRF protection but the mechanism differs. For frontend AJAX that handles sensitive operations:

```php
<?php
declare(strict_types=1);

namespace Training\Ajax\Controller\Frontend\Item;

use Magento\Framework\App\Action\Action;
use Magento\Framework\Data\Form\FormKey;

class Delete extends Action
{
    private FormKey $formKey;

    public function __construct(
        \Magento\Framework\App\Action\Context $context,
        FormKey $formKey
    ) {
        $this->formKey = $formKey;
        parent::__construct($context);
    }

    public function execute(): \Magento\Framework\App\Response\Http
    {
        $request = $this->getRequest();

        // Validate form key for frontend AJAX
        if (!$request->getParam('form_key') ||
            $request->getParam('form_key') !== $this->formKey->getFormKey()
        ) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Your session has expired. Please refresh the page.')
            );
        }

        // Proceed with deletion...
    }
}
```

> **Warning:** Never skip CSRF validation on state-changing endpoints just because they are AJAX. Any endpoint that creates, modifies, or deletes data is a CSRF target.

---

### Definition of Done — CSRF

- [ ] All admin AJAX endpoints validate form key before processing
- [ ] AJAX requests include form key via header or request body
- [ ] Invalid/expired form key returns HTTP 403 with JSON error message
- [ ] Frontend state-changing AJAX endpoints also validate form keys
- [ ] No state-changing admin endpoint accessible without CSRF check

---

### Topic 3: Rate Limiting — Redis-Based Throttling

**Why This Topic Exists:**

Topic 07 touches rate limiting in API configuration but does not cover the `[limiter]` env.php section or how to implement per-user/IP throttling using Magento's built-in `State` class. Production deployments need rate limiting configured before launch.

---

### Redis Rate Limiter — `[limiter]` Section in env.php

Magento 2.4 uses a Redis-based rate limiter configured in `app/etc/env.php`. This replaces the older in-memory limiter and supports distributed deployments.

**`env.php` — Limiter Configuration:**

```php
<?php
// app/etc/env.php

return [
    // ... other config sections ...

    ' limiter' => [
        'http' => [
            // Per-IP rate limiting for HTTP endpoints
            'enabled' => true,
            'limit' => 100,           // requests per window
            'window' => 60,          // window size in seconds (1 minute)
            'burst' => 150,          // max burst capacity
        ],
        'webapi' => [
            // Stricter limits for API endpoints
            'enabled' => true,
            'limit' => 60,           // more restrictive for APIs
            'window' => 60,
            'burst' => 80,
        ],
        'actions' => [
            // Per-action overrides
            'customer/account/login' => [
                'limit' => 5,         // strict limit on login attempts
                'window' => 300,      // 5 minute window
                'burst' => 10,
            ],
        ],
    ],
];
```

> **Pro Tip:** The `limiter` section uses a **space** before `limiter` key (not `limiter` at the root level). This is intentional — it's a scoped configuration section.

---

### How Magento's Rate Limiter Works

When enabled, Magento's `\Magento\Framework\RateLimiter` uses Redis to track request counts per IP or per user. When a limit is exceeded, the request is rejected with HTTP 429 (Too Many Requests).

You can also implement custom rate limiting in your own controllers using the `State` class pattern:

```php
<?php
declare(strict_types=1);

namespace Training\RateLimit\Api\RateLimit;

use Magento\Framework\RateLimiter\StateFactory;

class IpThrottler
{
    public function __construct(
        private readonly \Magento\Framework\RateLimiter\StateFactory $stateFactory,
    ) {}

    public function throttle(string $identifier, int $maxRequests, int $windowSeconds): bool
    {
        $limiterState = $this->stateFactory->create('ip_' . $identifier);

        if ($limiterState->consume()) {
            return true; // Request allowed
        }

        return false; // Request rejected — rate limit exceeded
    }
}
```

**Controller Integration:**

```php
<?php
declare(strict_types=1);

namespace Training\Ajax\Controller\Adminhtml\Export;

use Magento\Framework\Controller\Result\JsonFactory;

class Export extends \Magento\Backend\App\Action
{
    public function __construct(
        \Magento\Backend\App\Action\Context $context,
        private readonly JsonFactory $resultJsonFactory,
        private readonly \Magento\Framework\RateLimiter\StateFactory $stateFactory,
    ) {
        parent::__construct($context);
    }

    public function execute(): \Magento\Framework\App\Response\Http
    {
        $result = $this->resultJsonFactory->create();

        // Create a throttler for this specific action
        $limiter = $this->stateFactory->create('admin_export_' . $this->getUserId());

        // 10 requests per 60 seconds — strict for export operations
        $allowed = true;
        for ($i = 0; $i < 10; $i++) {
            if (!$limiter->consume()) {
                $allowed = false;
                break;
            }
        }

        if (!$allowed) {
            $this->getResponse()->setHttpResponseCode(429);
            return $result->setData([
                'success' => false,
                'message' => __('Rate limit exceeded. Please wait before trying again.')
            ]);
        }

        // Proceed with export...
        return $result->setData(['success' => true, 'download_url' => $url]);
    }

    protected function _isAllowed(): bool
    {
        return $this->_authorization->isAllowed('Training_RateLimit::export');
    }

    private function getUserId(): int
    {
        return (int) $this->_auth->getUser()->getId();
    }
}
```

---

### Per-IP Rate Limiting Middleware

For custom middleware-style rate limiting using Redis directly:

```php
<?php
declare(strict_types=1);

namespace Training\RateLimit\Model;

use Magento\Framework\App\RequestInterface;
use Magento\Framework\HTTP\PhpEnvironment\RemoteAddress;

class IpRateLimiter
{
    private const RATE_LIMIT_KEY_PREFIX = 'rate_limit/ip/';
    private const DEFAULT_LIMIT = 100;
    private const DEFAULT_WINDOW = 60;

    public function __construct(
        private readonly \Magento\Framework\Cache\Frontend\Adapter\Redis $redis,
        private readonly RemoteAddress $remoteAddress,
    ) {}

    public function isAllowed(string $identifier, int $limit = self::DEFAULT_LIMIT): bool
    {
        $key = self::RATE_LIMIT_KEY_PREFIX . $identifier;
        $ip = $this->remoteAddress->getRemoteAddress();

        $current = (int) $this->redis->get($key);

        if ($current >= $limit) {
            return false; // Rate limit exceeded
        }

        // Increment counter
        if ($current === 0) {
            // First request — set with TTL
            $this->redis->set($key, 1, self::DEFAULT_WINDOW);
        } else {
            $this->redis->inc($key);
        }

        return true;
    }
}
```

> **Production Note:** For distributed environments (multiple app servers), use Redis for rate limiting state. In-memory counters only work on single-server deployments and will cause inconsistent throttling under load.

---

### Definition of Done — Rate Limiting

- [ ] `[limiter]` section present in `env.php` with `http` and `webapi` configurations
- [ ] Login endpoint has a stricter per-action limit (≤5 attempts per 5 minutes)
- [ ] Custom throttler uses Redis, not in-memory storage
- [ ] Rate-limited requests return HTTP 429 with informative JSON error
- [ ] Rate limiting does not break legitimate users (limits are reasonable for use case)

---

### Topic 4: Secret Management — Encrypted Config and env.php

**Why This Topic Exists:**

Storing credentials in plain text config files is a critical security failure. Magento provides the `encrypted` field type for system.xml configuration and a dedicated `EncryptionKey` resource model. This topic shows how to use both correctly and how to avoid the common mistake of exposing credentials in admin UI.

---

### `encrypted` Field Type in system.xml

The `encrypted` type tells Magento to use the encryption key to obscure the value in the database and admin UI. When admin users view the configuration, they see `*****` instead of the actual value.

**`etc/adminhtml/system.xml`:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Config:etc/system_file.xsd">
    <system>
        <section id="training_security" translate="label" type="text" sortOrder="100" showInDefault="1" showInWebsite="1" showInStore="1">
            <label>Security Configuration</label>
            <tab>training</tab>

            <field id="api_key" translate="label comment" type="obscure" sortOrder="10" showInDefault="1" showInWebsite="0" showInStore="0">
                <label>External API Key</label>
                <comment>Stored encrypted. Leave blank to use default.</comment>
                <frontend_model>Magento\Config\Block\System\Config\Form\Field\Obscure</frontend_model>
                <backend_model>Magento\Config\Model\Config\Backend\Encrypted</backend_model>
            </field>

            <field id="webhook_secret" translate="label" type="obscure" sortOrder="20" showInDefault="1" showInWebsite="0" showInStore="0">
                <label>Webhook Secret Token</label>
                <backend_model>Magento\Config\Model\Config\Backend\Encrypted</backend_model>
            </field>

            <field id="redis_password" translate="label" type="obscure" sortOrder="30" showInDefault="1" showInWebsite="0" showInStore="0">
                <label>Redis Password</label>
                <backend_model>Magento\Config\Model\Config\Backend\Encrypted</backend_model>
            </field>

        </section>
    </system>
</config>
```

> **Key Detail:** Use `type="obscure"` for the admin UI field display. Use `backend_model="Magento\Config\Model\Config\Backend\Encrypted"` to tell Magento to encrypt the value before storing and decrypt it when reading.

---

### Reading Encrypted Config Values

```php
<?php
declare(strict_types=1);

namespace Training\Security\Service;

use Magento\Framework\App\Config\ScopeConfigInterface;

class ExternalApiService
{
    public function __construct(
        private readonly ScopeConfigInterface $scopeConfig,
    ) {}

    public function getApiKey(): string
    {
        // getValue returns the DECRYPTED value automatically
        // No manual decryption needed — Magento handles it
        return (string) $this->scopeConfig->getValue(
            'training_security/api_key',
            \Magento\Store\Model\ScopeInterface::SCOPE_STORE
        );
    }

    public function getWebhookSecret(): string
    {
        return (string) $this->scopeConfig->getValue(
            'training_security/webhook_secret',
            \Magento\Store\Model\ScopeInterface::SCOPE_STORE
        );
    }
}
```

---

### env.php Credential Patterns

env.php should store integration credentials. These are NOT encrypted by Magento — they are protected by filesystem permissions (600 on env.php). Do NOT put these in system.xml or database config.

**`app/etc/env.php`:**

```php
<?php
return [
    'system' => [
        'default' => [
            'training_security' => [
                'api_endpoint' => 'https://api.example.com',
            ],
        ],
    ],

    // Directories and caches configuration
    'directories' => [...],
    'cache' => [...],

    // For external service credentials that should NOT go in DB:
    'integration' => [
        'stripe' => [
            'secret_key' => 'sk_live_xxxx',
            'publishable_key' => 'pk_live_xxxx',
            'webhook_secret' => 'whsec_xxxx',
        ],
        'smtp' => [
            'api_key' => 'key_prod_xxxx',
        ],
    ],
];
```

**Access pattern for env.php integration credentials:**

```php
<?php
declare(strict_types=1);

namespace Training\Security\Service;

class StripeIntegration
{
    private array $config;

    public function __construct(
        \Magento\Framework\App\DeploymentConfig $deploymentConfig,
    ) {
        // Load from env.php — not from scope config
        $this->config = $deploymentConfig->get('integration/stripe') ?? [];
    }

    public function getSecretKey(): string
    {
        return (string) ($this->config['secret_key'] ?? '');
    }

    public function getWebhookSecret(): string
    {
        return (string) ($this->config['webhook_secret'] ?? '');
    }
}
```

> **Critical:** env.php credentials are loaded via `DeploymentConfig` — they are NOT encrypted. The security comes from filesystem permissions (`chmod 600 env.php`). If an attacker can read env.php, the encryption is moot. Always pair env.php credential storage with proper file permissions.

---

### Secret Scanner — Detecting Hardcoded Credentials

A static check pattern to add to your CI pipeline:

```bash
#!/bin/bash
# scripts/detect-secrets.sh

# Patterns that indicate hardcoded credentials
PATTERNS=(
    "sk_live_[a-zA-Z0-9]\{24,\}"
    "pk_live_[a-zA-Z0-9]\{24,\}"
    "api_key.*=.*['\"][a-zA-Z0-9_-]\{20,\}['\"]"
    "password.*=.*['\"][^'\"]{8,}['\"]"
    "secret.*=.*['\"][a-zA-Z0-9_-]\{16,\}['\"]"
)

FOUND=0
for pattern in "${PATTERNS[@]}"; do
    if grep -r -E "$pattern" --include="*.php" app/code/; then
        echo "WARNING: Possible secret found: $pattern"
        FOUND=1
    fi
done

exit $FOUND
```

> **Pro Tip:** Use this script as a pre-commit hook. It won't catch all secrets (obfuscated ones still slip through), but it catches the most common hardcoded credential patterns.

---

### Encryption Key Management

Magento's encryption key is stored in `app/etc/env.php` under the `crypt` section. The key is used for encrypting sensitive data in the database (credit cards, admin passwords, encrypted config values).

**Viewing the encryption key (read-only via CLI):**

```bash
# Never show the actual key in logs — only verify it exists
php bin/magento crypto:status
```

**Rotating the encryption key (requires re-encryption of existing data):**

```bash
# Generate a new key (this does NOT automatically re-encrypt existing data)
php bin/magento crypto:generate:key

# Re-encrypt sensitive data after key rotation
php bin/magento crypto:reencrypt
```

> **Production Warning:** Encryption key rotation without re-encryption makes previously encrypted data (customer data, payment info) permanently unreadable. Always plan key rotation carefully in production.

---

### Definition of Done — Secret Management

- [ ] All credential fields in system.xml use `backend_model="Magento\Config\Model\Config\Backend\Encrypted"`
- [ ] Credentials stored in env.php use `DeploymentConfig`, not `ScopeConfig`
- [ ] env.php has filesystem permissions set to 600
- [ ] No credentials appear in module code, log files, or git history
- [ ] CI pipeline runs secret detection script before merge

---

### Topic 5: Security Headers + 2FA — CORS, CSP, X-Frame-Options, Admin 2FA

**Why This Topic Exists:**

The final layer of production hardening — enforcing browser-level security controls and strong admin authentication. This topic covers security headers for custom API endpoints and how to programmatically enroll admin users in 2FA.

---

### Content-Security-Policy Header

CSP is an HTTP response header that tells the browser which resources can be loaded. For custom Magento API endpoints, you should set a restrictive CSP.

**`etc/di.xml` — Plugin to Add CSP Header:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <type name="Magento\Framework\App\Action\Action">
        <plugin name="TrainingSecurityCspPlugin"
                type="Training\Security\Plugin\CspHeaderPlugin"
                sortOrder="10"/>
    </type>

</config>
```

**`Plugin/CspHeaderPlugin.php`:**

```php
<?php
declare(strict_types=1);

namespace Training\Security\Plugin;

use Magento\Framework\App\Action\Action;
use Magento\Framework\App\Response\Http;

class CspHeaderPlugin
{
    public function afterExecute(Action $subject, $result): void
    {
        if (!$result instanceof Http) {
            return;
        }

        // Restrictive CSP for API responses
        $cspDirectives = [
            "default-src 'none'",
            "script-src 'self'",
            "style-src 'self'",
            "img-src 'self' data:",
            "font-src 'self'",
            "connect-src 'self'",
            "frame-ancestors 'none'",
            "base-uri 'self'",
            "form-action 'self'",
        ];

        $result->setHeader('Content-Security-Policy', implode('; ', $cspDirectives));
    }
}
```

---

### X-Frame-Options Header

X-Frame-Options prevents clickjacking by controlling whether your page can be embedded in an iframe. Required for admin pages and payment processing endpoints.

```php
<?php
declare(strict_types=1);

namespace Training\Security\Plugin;

use Magento\Framework\App\Action\Action;
use Magento\Framework\App\Response\Http;

class XFrameOptionsPlugin
{
    public function afterExecute(Action $subject, $result): void
    {
        if (!$result instanceof Http) {
            return;
        }

        // DENY — page cannot be embedded in any iframe
        // Use SAMEORIGIN if embedding within same domain is needed
        $result->setHeader('X-Frame-Options', 'DENY');
    }
}
```

**For GraphQL endpoints specifically — add in `etc/di.xml`:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <type name="Magento\Framework\GraphQl\Controller\GraphQL">
        <plugin name="TrainingSecurityGraphQLHeaders"
                type="Training\Security\Plugin\GraphQlSecurityHeadersPlugin"
                sortOrder="10"/>
    </type>

</config>
```

---

### CORS Configuration for Custom REST Endpoints

Cross-Origin Resource Sharing (CORS) must be configured for custom REST endpoints that will be called from browser-based frontends (PWA, mobile app, or third-party integrations).

**`etc/di.xml` — CORS Plugin:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <type name="Magento\Framework\App\Action\Action">
        <plugin name="TrainingCorsPlugin"
                type="Training\Security\Plugin\CorsHeaderPlugin"
                sortOrder="5"/>
    </type>

</config>
```

**`Plugin/CorsHeaderPlugin.php`:**

```php
<?php
declare(strict_types=1);

namespace Training\Security\Plugin;

use Magento\Framework\App\Action\Action;
use Magento\Framework\App\Response\Http;
use Magento\Framework\App\RequestInterface;

class CorsHeaderPlugin
{
    public const ALLOWED_ORIGINS = [
        'https://your-pwa.example.com',
        'https://admin.example.com',
    ];

    public function beforeExecute(Action $subject, RequestInterface $request): void
    {
        // Only apply CORS to API routes
        if (!str_starts_with($request->getPathInfo(), '/training_api/')) {
            return;
        }

        $origin = $request->getHeader('Origin');

        if (in_array($origin, self::ALLOWED_ORIGINS, true)) {
            // Store for later — set headers in afterExecute
            $subject->getRequest()->setParam('_cors_origin', $origin);
        }
    }

    public function afterExecute(Action $subject, $result): void
    {
        if (!$result instanceof Http) {
            return;
        }

        $request = $subject->getRequest();
        $origin = $request->getParam('_cors_origin');

        if ($origin) {
            $result->setHeader('Access-Control-Allow-Origin', $origin);
            $result->setHeader('Access-Control-Allow-Methods', 'GET, POST, OPTIONS');
            $result->setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization, X-Form-Key');
            $result->setHeader('Access-Control-Allow-Credentials', 'true');
            $result->setHeader('Access-Control-Max-Age', '86400'); // 24 hours
        }

        // Handle preflight OPTIONS request
        if ($subject->getRequest()->getMethod() === 'OPTIONS') {
            $result->setHttpResponseCode(204);
        }
    }
}
```

> **Security Note:** Never set `Access-Control-Allow-Origin: *` for requests that include credentials (cookies, authorization headers). Use an explicit allowlist of trusted origins.

---

### Admin 2FA Enrollment — Programmatic Enrollment

You can enforce 2FA enrollment for admin users programmatically. This is useful during user provisioning or when enforcing 2FA via a setup script.

```php
<?php
declare(strict_types=1);

namespace Training\Security\Setup;

use Magento\User\Model\User;
use Magento\TwoFactorAuth\Model\AdminUserProvider;
use Magento\Framework\Setup\ModuleContextInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class AdminUser2FAEnforcement
{
    public function __construct(
        private readonly \Magento\User\Model\ResourceModel\User $userResource,
        private readonly \Magento\Backend\Model\Auth\Session $authSession,
    ) {}

    public function enforce2FAForUser(int $userId): void
    {
        /** @var User $user */
        $user = $this->userResource->load($userId);

        // Mark user as 2FA enabled
        // This forces the user to complete 2FA enrollment on next login
        $user->setData('force_2fa', true);
        $this->userResource->save($user);
    }

    public function enrollUserIn2FA(int $userId, string $provider, array $providerData): void
    {
        /** @var User $user */
        $user = $this->userResource->load($userId);

        // The actual enrollment depends on the 2FA provider
        // Google Authenticator uses TOTP — store the secret
        // For TOTP-based providers:
        $user->setData('twofactorauth_token', $providerData['secret']);
        $user->setData('twofactorauth_provider', $provider);
        $user->setData('twofactorauth_active', true);

        $this->userResource->save($user);
    }
}
```

**Forcing 2FA on all admin users via CLI command:**

```bash
#!/bin/bash
# Enforce 2FA for all admin users
php bin/magento security:2fa:enforce --role=admin
```

---

### Session Fixation Prevention

Session fixation attacks try to set or guess a user's session ID before they authenticate. Magento's admin session handling includes built-in protection, but for custom admin controllers you should regenerate the session ID after authentication.

```php
<?php
declare(strict_types=1);

namespace Training\Security\Controller\Adminhtml\Auth;

use Magento\Backend\App\Action;

class Login extends Action
{
    public function execute(): \Magento\Framework\App\Response\Http
    {
        $resultRedirect = $this->resultRedirectFactory->create();

        $credentials = [
            'username' => $this->getRequest()->getParam('username'),
            'password' => $this->getRequest()->getParam('password'),
        ];

        if ($this->_auth->login($credentials['username'], $credentials['password'])) {
            // CRITICAL — regenerate session ID after login
            // This prevents session fixation attacks
            $this->_session->regenerateId();

            return $resultRedirect->setPath('dashboard/index');
        }

        $this->messageManager->addErrorMessage __('Invalid credentials.');
        return $resultRedirect->setPath('login');
    }
}
```

---

### Cookie Security Flags

All admin cookies should have HttpOnly, Secure, and SameSite flags set. Magento configures these by default, but for custom cookie creation you must set them explicitly.

**Creating a secure cookie in a controller:**

```php
<?php
declare(strict_types=1);

namespace Training\Security\Controller\Adminhtml;

use Magento\Framework\App\Action\Action;
use Magento\Framework\App\Response\Http;

class SetSecureCookie extends Action
{
    public function execute(): Http
    {
        $response = $this->getResponse();

        // Use Magento's cookie metadata for secure flag handling
        $metadata = $this->cookieMetadataFactory->create()
            ->setPath('/')
            ->setHttpOnly(true)           // JavaScript cannot read this cookie
            ->setSecure(true)              // Only sent over HTTPS
            ->setSameSite('Strict');       // CSRF protection via SameSite

        $this->session->setData('custom_token', $token, $metadata);

        return $response;
    }

    private function getCookieMetadataFactory(): \Magento\Framework\Stdlib\Cookie\CookieMetadataFactory
    {
        return $this->_objectManager->get(
            \Magento\Framework\Stdlib\Cookie\CookieMetadataFactory::class
        );
    }
}
```

**Configuring admin session cookies via `env.php`:**

```php
<?php
// app/etc/env.php
return [
    // ...
    'session' => [
        'save' => 'redis',
        'redis' => [
            'host' => '127.0.0.1',
            'port' => 6379,
            'password' => '',          // Set password in production
            'database' => 0,           // Session DB
            'session_lifetime' => 3600,
            'session_locking' => true,
            'log_level' => 0,
        ],
    ],

    // Cookie configuration
    'cookie' => [
        'cookie_model' => 'Magento\Framework\Stdlib\Cookie\PhpCookieManager',
        'cookie_domain' => '.example.com',
        'cookie_httponly' => true,
        'cookie_secure' => true,      // Only over HTTPS
        'cookie_samesite' => 'Strict',
    ],
];
```

---

### Security Headers Summary

| Header | Purpose | Admin Value | API Value |
|--------|---------|-------------|-----------|
| Content-Security-Policy | Prevent XSS/injection | `frame-ancestors 'none'` | `default-src 'none'` |
| X-Frame-Options | Prevent clickjacking | `DENY` | `DENY` |
| X-Content-Type-Options | Prevent MIME sniffing | `nosniff` | `nosniff` |
| X-XSS-Protection | XSS filter (legacy browsers) | `1; mode=block` | `1; mode=block` |
| Referrer-Policy | Control referrer leakage | `strict-origin-when-cross-origin` | `no-referrer` |
| Access-Control-Allow-Origin | CORS origin restriction | Not needed | Explicit allowlist only |

---

### Definition of Done — Security Headers + 2FA

- [ ] CSP header set on all custom API responses
- [ ] X-Frame-Options set to `DENY` on all admin and payment endpoints
- [ ] CORS headers set with explicit allowlist (no wildcard origins for credentialed requests)
- [ ] Admin session IDs regenerated after login
- [ ] All custom cookies have HttpOnly, Secure, and SameSite flags
- [ ] 2FA can be enforced for admin users programmatically

---

## Common Mistakes to Avoid

- **String interpolation in SQL** — `"SELECT * FROM table WHERE id = '" . $id . "'"` is always injection vulnerable, regardless of "trusted input"
- **Skipping CSRF on AJAX endpoints** — "It's just an AJAX call" is not a security justification; any state-changing operation needs CSRF validation
- **Putting credentials in system.xml** — sensitive credentials belong in env.php via DeploymentConfig, not in the database config
- **Using `type="password"` for credentials** — `type="obscure"` with `backend_model="Magento\Config\Model\Config\Backend\Encrypted"` is the correct pattern
- **Wildcard CORS origins** — `Access-Control-Allow-Origin: *` on endpoints that handle credentials is a critical misconfiguration
- **SameSite=None without Secure** — cookies with `SameSite=None` MUST also have `Secure=true`, otherwise browsers reject them
- **Skipping rate limiting for login endpoints** — brute force protection on authentication endpoints is non-negotiable in production

---

## Reading List

- [Magento Security Best Practices](https://developer.adobe.com/commerce/php/development/security/)
- [OWASP Top 10 — Injection](https://owasp.org/www-project-top-ten/2017/A1_2017-Injection)
- [OWASP CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [RFC 6749 — OAuth 2.0 Security Topics](https://datatracker.ietf.org/doc/html/rfc6749#section-10)
- [MDN — SameSite Cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
- [Chrome CSP Documentation](https://developer.chrome.com/docs/extensions/mv3/security/)
- [Magento Encryption Key Management](https://developer.adobe.com/commerce/operations/configuration-guide/security/encryption-key/)
- [Rate Limiter Implementation — Magento DevDocs](https://developer.adobe.com/commerce/php/development/rate-limiting/)