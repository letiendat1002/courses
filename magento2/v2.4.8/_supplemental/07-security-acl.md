---
title: "26 - Security & ACL"
description: "Deep dive into Magento 2.4.8 security: CSRF protection, XSS prevention, SQL injection prevention, file upload security, ACL/permissions system, and secure coding practices."
tags: magento2, security, csrf, xss, sql-injection, acl, permissions, secure-coding
rank: 7
pathways: [magento2-deep-dive]
---

# 26 — Security & ACL in Magento 2.4.8

Security is not a feature you bolt on at the end — in Magento 2 it is woven into the framework at every layer. This module systematically walks through every built-in defence mechanism, shows how and why each works, and teaches you to extend them correctly for your own modules.

---

## Table of Contents

1. [Security Overview](#1-security-overview)
2. [CSRF Protection](#2-csrf-protection)
3. [XSS Prevention](#3-xss-prevention)
4. [SQL Injection Prevention](#4-sql-injection-prevention)
5. [File Upload Security](#5-file-upload-security)
6. [ACL System](#6-acl-access-control-list-system)
7. [Admin Session Security](#7-admin-session-security)
8. [API Security](#8-api-security)
9. [Secure Configuration](#9-secure-configuration)
10. [Security Headers](#10-security-headers)
11. [Secure Development Practices](#11-secure-development-practices)

---

## 1. Security Overview

### 1.1 OWASP Top 10 and Magento

The [OWASP Top 10](https://owasp.org/www-project-top-ten/) lists the most critical web application security risks. Magento 2 ships with mitigations for every item on this list.

| OWASP 2021 Category | Magento Mitigation |
|---|---|
| A01 – Broken Access Control | ACL system, admin controller `_isAllowed()`, role-based user management |
| A02 – Cryptographic Failures | Argon2ID password hashing, AES-256 key for sensitive config values |
| A03 – Injection | Parameterised DB queries, Escaper for output, input validators |
| A04 – Insecure Design | DI-based design, no `ObjectManager` in business logic, CSP |
| A05 – Security Misconfiguration | `env.php` secrets separation, HTTPS enforcement flags |
| A06 – Vulnerable Components | `composer audit`, Adobe Security Patches, Composer lock |
| A07 – Auth & Session Failures | Secure session cookies, 2FA, session regeneration on login |
| A08 – Software & Data Integrity | Composer signature verification, module registration |
| A09 – Logging & Monitoring | `system.log`, `exception.log`, admin action log, NewRelic integration |
| A10 – SSRF | cURL wrapper with allowlists, `Zend\Http\Client` timeouts |

### 1.2 Magento's Built-in Security Features

Magento 2.4.8 ships with all of the following out of the box:

- **Form key (CSRF tokens)** — every POST form carries a server-issued token
- **Content Security Policy** — configurable CSP in `security.xml`
- **Two-Factor Authentication** — `Magento_TwoFactorAuth` module (on by default)
- **reCAPTCHA v2/v3/Invisible** — `Magento_ReCaptchaFrontendUi` / `Magento_ReCaptchaAdminUi`
- **Argon2ID** and **bcrypt** password hashing — via `Magento\Framework\Encryption\Encryptor`
- **Parameterised SQL** — all collection/resource model queries use bound parameters
- **Output Escaper** — `Magento\Framework\Escaper` injected into every template block
- **Admin path randomisation** — configurable via `env.php` `backend/frontName`
- **URL key validation** — whitelist-based routing prevents directory traversal
- **File upload MIME-type checks** — `Magento\Framework\File\Uploader`

### 1.3 Security-Related Admin Configuration

Navigate to **Stores → Configuration → Advanced → Admin** for:

| Setting | Path | Default |
|---|---|---|
| Admin session lifetime | `admin/security/session_lifetime` | 900 s |
| Lock account after failed logins | `admin/security/lockout_failures` | 6 |
| Password change interval | `admin/security/password_reset_link_expiration_period` | 2 days |
| Add secret key to URLs | `admin/security/use_form_key` | Yes |

Navigate to **Stores → Configuration → Security** (Magento 2.4+) for:

| Setting | Path |
|---|---|
| Storefront reCAPTCHA type | `recaptcha_frontend/type_recaptcha/public_key` |
| Admin 2FA providers | `twofactorauth/general/force_providers` |
| Password complexity | `customer/password/required_character_classes_number` |

---

## 2. CSRF Protection

### 2.1 How CSRF Works

Cross-Site Request Forgery tricks a logged-in user's browser into sending a forged request to a legitimate site on an attacker's behalf. Because cookies are sent automatically, the server cannot distinguish the forged request from a genuine one — unless a secret token unknown to the attacker is required.

```
Victim visits attacker site
  → attacker site sends hidden POST to shop.example.com/checkout
  → victim's browser attaches session cookie automatically
  → without CSRF token the shop has no way to reject it
```

### 2.2 Magento's Form Key Mechanism

Magento generates a cryptographically random form key once per session. Every state-changing POST must echo that key; the `CsrfValidator` rejects requests where the key is absent or mismatched.

**Session key storage:**

```php
// vendor/magento/framework/Session/SessionManager.php
// The form key is stored in $_SESSION['_form_key']
```

**Generating the key:**

```php
// Magento\Framework\Data\Form\FormKey
declare(strict_types=1);

namespace Magento\Framework\Data\Form;

use Magento\Framework\Math\Random;
use Magento\Framework\Session\SessionManagerInterface;

class FormKey
{
    private const FORM_KEY_LENGTH = 32;

    public function __construct(
        private readonly Random $mathRandom,
        private readonly SessionManagerInterface $session,
        private readonly \Magento\Framework\Escaper $escaper
    ) {}

    public function getFormKey(): string
    {
        if (!$this->isPresent()) {
            $this->set($this->mathRandom->getRandomString(self::FORM_KEY_LENGTH));
        }
        return $this->escaper->escapeHtml((string) $this->session->getData('_form_key'));
    }

    public function isPresent(): bool
    {
        return (bool) $this->session->getData('_form_key');
    }
}
```

### 2.3 Embedding `form_key` in Every POST Form

**In a PHTML template:**

```html
<!-- app/code/Vendor/Module/view/frontend/templates/form.phtml -->
<?php
/** @var \Magento\Framework\View\Element\Template $block */
/** @var \Magento\Framework\Escaper $escaper */
?>
<form method="post" action="<?= $escaper->escapeUrl($block->getPostUrl()) ?>">
    <?= $block->getBlockHtml('formkey') ?>   <!-- renders the hidden input -->
    <input type="text" name="email" value="">
    <button type="submit"><?= $escaper->escapeHtml(__('Submit')) ?></button>
</form>
```

The `formkey` block renders:

```html
<input name="form_key" type="hidden" value="YourRandomToken123456789012345">
```

**Using the FormKey block helper directly:**

```php
// In your Block class
declare(strict_types=1);

namespace Vendor\Module\Block;

use Magento\Framework\Data\Form\FormKey;
use Magento\Framework\View\Element\Template;
use Magento\Framework\View\Element\Template\Context;

class MyForm extends Template
{
    public function __construct(
        Context $context,
        private readonly FormKey $formKey,
        array $data = []
    ) {
        parent::__construct($context, $data);
    }

    public function getFormKey(): string
    {
        return $this->formKey->getFormKey();
    }
}
```

```html
<!-- In the corresponding template -->
<input name="form_key" type="hidden" value="<?= $escaper->escapeHtmlAttr($block->getFormKey()) ?>">
```

### 2.4 Validating the Form Key in a Controller

```php
// app/code/Vendor/Module/Controller/Index/Save.php
declare(strict_types=1);

namespace Vendor\Module\Controller\Index;

use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Framework\App\CsrfAwareActionInterface;
use Magento\Framework\App\Request\InvalidRequestException;
use Magento\Framework\App\RequestInterface;
use Magento\Framework\Controller\Result\RedirectFactory;
use Magento\Framework\Data\Form\FormKey\Validator as FormKeyValidator;
use Magento\Framework\Exception\InputException;
use Magento\Framework\Message\ManagerInterface;

class Save implements HttpPostActionInterface, CsrfAwareActionInterface
{
    public function __construct(
        private readonly RequestInterface $request,
        private readonly FormKeyValidator $formKeyValidator,
        private readonly RedirectFactory $resultRedirectFactory,
        private readonly ManagerInterface $messageManager
    ) {}

    /**
     * CsrfAwareActionInterface: create the exception object Magento will throw
     * when CSRF validation fails (called by the framework, not your code).
     */
    public function createCsrfValidationException(RequestInterface $request): ?InvalidRequestException
    {
        $resultRedirect = $this->resultRedirectFactory->create();
        $resultRedirect->setPath('*/*/');

        return new InvalidRequestException(
            $resultRedirect,
            [__('Invalid Form Key. Please refresh the page.')]
        );
    }

    /**
     * CsrfAwareActionInterface: return false to let the framework do the
     * validation automatically; return true to skip (dangerous — only for
     * API endpoints that use their own auth).
     */
    public function validateForCsrf(RequestInterface $request): ?bool
    {
        return null; // null = use default Magento validation
    }

    public function execute(): \Magento\Framework\Controller\ResultInterface
    {
        // Form key already validated by the framework via CsrfAwareActionInterface.
        // Manual check below is shown for illustration; in real code you rely on the interface.
        if (!$this->formKeyValidator->validate($this->request)) {
            throw new InputException(__('Invalid form key. Please reload the page.'));
        }

        // ... process the form data
        $this->messageManager->addSuccessMessage(__('Saved successfully.'));
        $resultRedirect = $this->resultRedirectFactory->create();
        return $resultRedirect->setPath('*/*/');
    }
}
```

> **Key point:** Implement `CsrfAwareActionInterface` on every frontend `POST` controller. Magento 2.3+ uses this interface to validate tokens automatically before `execute()` is even called.

### 2.5 AJAX CSRF Handling

Frontend AJAX calls must pass the form key in the POST body or as an `X-Form-Key` header:

```javascript
// requirejs-config.js already loads Magento_Customer/js/customer-data which
// exposes the form key. Here is the pattern:

define(['jquery', 'mage/storage', 'Magento_Customer/js/customer-data'], function ($, storage, customerData) {
    'use strict';

    function postWithCsrf(url, data) {
        // Magento stores the form key in localStorage via customer-data section
        var formKey = $('input[name="form_key"]').val()
            || $.cookie('form_key'); // fallback

        return $.ajax({
            url: url,
            type: 'POST',
            data: Object.assign({ form_key: formKey }, data),
            dataType: 'json'
        });
    }

    return { post: postWithCsrf };
});
```

For GraphQL/REST API calls that use Bearer tokens, CSRF is not applicable — the token itself is the proof of intent.

### 2.6 Custom CSRF Validator (Advanced)

For custom scenarios (e.g., a webhook that uses HMAC instead of form key):

```php
// app/code/Vendor/Module/Model/WebhookCsrfValidator.php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Framework\App\CsrfAwareActionInterface;
use Magento\Framework\App\Request\InvalidRequestException;
use Magento\Framework\App\RequestInterface;
use Magento\Framework\Controller\Result\JsonFactory;

class WebhookCsrfValidator
{
    private const HMAC_HEADER = 'X-Webhook-Signature';

    public function __construct(
        private readonly JsonFactory $resultJsonFactory,
        private readonly string $webhookSecret
    ) {}

    public function validate(RequestInterface $request): bool
    {
        $signature = $request->getHeader(self::HMAC_HEADER);
        $body      = $request->getContent();

        $expected = 'sha256=' . hash_hmac('sha256', $body, $this->webhookSecret);

        return hash_equals($expected, (string) $signature);
    }
}
```

```php
// app/code/Vendor/Module/Controller/Webhook/Receive.php
declare(strict_types=1);

namespace Vendor\Module\Controller\Webhook;

use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Framework\App\CsrfAwareActionInterface;
use Magento\Framework\App\Request\InvalidRequestException;
use Magento\Framework\App\RequestInterface;
use Vendor\Module\Model\WebhookCsrfValidator;

class Receive implements HttpPostActionInterface, CsrfAwareActionInterface
{
    public function __construct(
        private readonly RequestInterface $request,
        private readonly WebhookCsrfValidator $webhookValidator,
        private readonly \Magento\Framework\Controller\Result\JsonFactory $jsonFactory
    ) {}

    public function createCsrfValidationException(RequestInterface $request): ?InvalidRequestException
    {
        $result = $this->jsonFactory->create();
        $result->setHttpResponseCode(403);
        $result->setData(['error' => 'Invalid webhook signature']);
        return new InvalidRequestException($result);
    }

    public function validateForCsrf(RequestInterface $request): ?bool
    {
        // Return true to SKIP the default form_key check and use our own validator.
        return $this->webhookValidator->validate($request);
    }

    public function execute(): \Magento\Framework\Controller\ResultInterface
    {
        // HMAC already validated before we get here.
        $payload = json_decode((string) $this->request->getContent(), true, 512, JSON_THROW_ON_ERROR);
        // process $payload …
        return $this->jsonFactory->create()->setData(['status' => 'ok']);
    }
}
```

---

## 3. XSS Prevention

### 3.1 Types of XSS

| Type | Description | Example Attack Vector |
|---|---|---|
| **Reflected** | Malicious script in URL parameter reflected in response | `?q=<script>steal()</script>` |
| **Stored** | Script saved to DB and rendered to other users | Product review containing `<img onerror=...>` |
| **DOM-based** | Script injected via JavaScript that reads `location.hash` | `#"><script>...` processed by frontend JS |

### 3.2 The `Magento\Framework\Escaper` Class

`Escaper` is the canonical XSS defence. It is automatically available in all PHTML templates as `$escaper`. Never skip it.

```php
// vendor/magento/framework/Escaper.php (simplified)
declare(strict_types=1);

namespace Magento\Framework;

use Laminas\Escaper\Escaper as LaminasEscaper;

class Escaper
{
    private LaminasEscaper $laminasEscaper;

    public function escapeHtml(mixed $data, ?array $allowedTags = null): string
    {
        // Strips dangerous tags, encodes special chars
        // $allowedTags = ['b', 'em'] allows a safe subset of HTML
    }

    public function escapeHtmlAttr(string $string, bool $escapeSingleQuote = true): string
    {
        // For use inside HTML attribute values
    }

    public function escapeUrl(string $string): string
    {
        // Encodes characters invalid in a URL context
    }

    public function escapeJs(string $string): string
    {
        // JSON-encodes for safe injection into a JS string literal
    }

    public function escapeCss(string $string): string
    {
        // For values inserted into CSS
    }
}
```

**Practical template usage:**

```php
<?php
// app/code/Vendor/Module/view/frontend/templates/product/details.phtml
/** @var \Vendor\Module\Block\Product\Details $block */
/** @var \Magento\Framework\Escaper $escaper */
?>

<!-- 1. Plain text inside HTML body -->
<h1><?= $escaper->escapeHtml($block->getProductName()) ?></h1>

<!-- 2. Inside an HTML attribute -->
<img src="<?= $escaper->escapeUrl($block->getImageUrl()) ?>"
     alt="<?= $escaper->escapeHtmlAttr($block->getProductName()) ?>">

<!-- 3. Into a JS variable (note the quotes are NOT in the PHP call) -->
<script>
    var productId = <?= (int) $block->getProductId() ?>;
    var productName = '<?= $escaper->escapeJs($block->getProductName()) ?>';
    var dataLayer = <?= /* @noEscape */ json_encode($block->getDataLayerArray()) ?>;
</script>

<!-- 4. URLs in href/src -->
<a href="<?= $escaper->escapeUrl($block->getProductUrl()) ?>">
    <?= $escaper->escapeHtml(__('View product')) ?>
</a>

<!-- 5. Rich text stored in DB (CMS blocks / product descriptions) -->
<!-- Only allow safe HTML tags: -->
<?= $escaper->escapeHtml($block->getDescription(), ['b', 'em', 'strong', 'i', 'ul', 'li', 'p', 'br']) ?>
```

### 3.3 `$block->escapeHtml()` Shortcut

Every block extending `\Magento\Framework\View\Element\AbstractBlock` proxies to the same `Escaper`:

```php
// Inside a Block class method
public function getSafeTitle(): string
{
    return $this->escapeHtml($this->getData('title'));
}
```

Both `$block->escapeHtml()` and `$escaper->escapeHtml()` call the same underlying method — use whichever reads more clearly, but prefer `$escaper` in templates for consistency with the 2.4 coding standard.

### 3.4 Why NOT to Use `strip_tags()` for XSS

`strip_tags()` is not a security function:

```php
// DANGEROUS — strip_tags does NOT protect against XSS
$name = strip_tags($_GET['name']); // attacker sends: <img src=x onerror=alert(1)>
echo $name; // outputs: <img src=x onerror=alert(1)> — strip_tags left event handler!

// CORRECT
echo $escaper->escapeHtml($_GET['name']);
// outputs: &lt;img src=x onerror=alert(1)&gt; — safely encoded
```

Reasons `strip_tags()` fails:
1. It does not strip **event attributes** on allowed tags (e.g., `<b onclick="...">`).
2. It does not encode the remaining text — ampersands, quotes, etc. remain raw.
3. Browser parser bugs can reassemble stripped content into executable HTML.

### 3.5 Content Security Policy (CSP)

Magento 2.3.5+ ships with a configurable CSP. In 2.4.8 it is **on by default in report-only mode** for storefronts and **enforce mode** in admin.

**`etc/csp_whitelist.xml`** — declare safe sources for your module:

```xml
<?xml version="1.0"?>
<csp_whitelist xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Csp/etc/csp_whitelist.xsd">
    <policies>
        <!-- Allow loading scripts from your CDN -->
        <policy id="script-src">
            <values>
                <value id="vendor-cdn" type="host">https://cdn.vendor.example.com</value>
            </values>
        </policy>

        <!-- Allow inline styles generated by your module (prefer nonce instead) -->
        <policy id="style-src">
            <values>
                <value id="vendor-styles" type="host">https://cdn.vendor.example.com</value>
            </values>
        </policy>

        <!-- Allow connecting to your API endpoint -->
        <policy id="connect-src">
            <values>
                <value id="vendor-api" type="host">https://api.vendor.example.com</value>
            </values>
        </policy>

        <!-- Allow loading images from any HTTPS source -->
        <policy id="img-src">
            <values>
                <value id="https-images" type="host">https:</value>
            </values>
        </policy>
    </policies>
</csp_whitelist>
```

**Switch CSP to report-only in `app/etc/config.php`:**

```php
// app/etc/env.php
return [
    'csp' => [
        'mode' => [
            'storefront' => [
                'report_only' => 1,         // 1 = report-only, 0 = enforce
            ],
            'admin' => [
                'report_only' => 0,
            ],
        ],
    ],
];
```

**Programmatic CSP policy addition:**

```php
// app/code/Vendor/Module/Plugin/CspPolicyPlugin.php
declare(strict_types=1);

namespace Vendor\Module\Plugin;

use Magento\Csp\Api\PolicyCollectorInterface;
use Magento\Csp\Model\Policy\FetchPolicy;

class CspPolicyPlugin
{
    public function afterCollect(
        PolicyCollectorInterface $subject,
        array $policies,
        array $defaultPolicies = []
    ): array {
        $policies[] = new FetchPolicy(
            'connect-src',
            false,
            ['https://analytics.vendor.example.com'],
            [],
            false,
            false,
            false,
            [],
            [],
            false,
            false
        );
        return $policies;
    }
}
```

### 3.6 JavaScript Escaping

When passing server-side data to JavaScript, always use `json_encode` with `JSON_HEX_TAG | JSON_HEX_APOS | JSON_HEX_QUOT | JSON_HEX_AMP`:

```php
<?php
// In a PHTML template
$jsConfig = json_encode(
    $block->getJsConfig(),
    JSON_HEX_TAG | JSON_HEX_APOS | JSON_HEX_QUOT | JSON_HEX_AMP | JSON_THROW_ON_ERROR
);
?>
<script>
    // @noEscape comment required to satisfy Magento coding standard PHPCS rule
    require(['Vendor_Module/js/widget'], function (widget) {
        widget(<?= /* @noEscape */ $jsConfig ?>);
    });
</script>
```

Or use Magento's `x-magento-init` mechanism, which is inherently safer:

```html
<script type="text/x-magento-init">
{
    "#myElement": {
        "Vendor_Module/js/widget": <?= /* @noEscape */ $block->getJsonConfig() ?>
    }
}
</script>
```

---

## 4. SQL Injection Prevention

### 4.1 How SQL Injection Works

```sql
-- Attacker supplies: email = ' OR '1'='1
SELECT * FROM customer_entity WHERE email = '' OR '1'='1' AND store_id = 1;
-- Returns every customer!
```

### 4.2 Parameterised Queries via Collections

Magento's Collection layer always uses Zend DB adapter bound parameters:

```php
// app/code/Vendor/Module/Model/ResourceModel/Widget/Collection.php
declare(strict_types=1);

namespace Vendor\Module\Model\ResourceModel\Widget;

use Magento\Framework\Model\ResourceModel\Db\Collection\AbstractCollection;

class Collection extends AbstractCollection
{
    protected function _construct(): void
    {
        $this->_init(
            \Vendor\Module\Model\Widget::class,
            \Vendor\Module\Model\ResourceModel\Widget::class
        );
    }
}
```

```php
// ✅ CORRECT — uses bound parameters (? placeholder)
$collection = $this->collectionFactory->create();
$collection->getSelect()->where('entity_id = ?', $entityId);
$collection->getSelect()->where('store_id IN (?)', [1, 2, 3]);
$collection->getSelect()->where('name LIKE ?', '%' . $this->connection->quote($searchTerm) . '%');
```

```php
// ❌ WRONG — direct string interpolation = SQL injection
$collection->getSelect()->where("entity_id = $entityId");
$collection->getSelect()->where('name LIKE "%' . $searchTerm . '%"');
```

### 4.3 `addFieldToFilter` — Safest High-Level API

```php
// Always prefer addFieldToFilter over raw where() when filtering on a single field
$collection = $this->collectionFactory->create();

// Exact match
$collection->addFieldToFilter('status', ['eq' => $status]);

// Multiple values (IN clause)
$collection->addFieldToFilter('store_id', ['in' => [1, 2, 3]]);

// Range
$collection->addFieldToFilter('price', ['gteq' => 10.00]);
$collection->addFieldToFilter('price', ['lteq' => 100.00]);

// LIKE — Magento automatically binds the value
$collection->addFieldToFilter('name', ['like' => '%' . $term . '%']);

// Combined OR conditions
$collection->addFieldToFilter(
    ['name', 'sku'],
    [
        ['like' => '%' . $term . '%'],
        ['like' => '%' . $term . '%'],
    ]
);
```

### 4.4 Direct DB Queries with Parameterisation

When you must bypass collections:

```php
// app/code/Vendor/Module/Model/ResourceModel/Widget.php
declare(strict_types=1);

namespace Vendor\Module\Model\ResourceModel;

use Magento\Framework\Model\ResourceModel\Db\AbstractDb;

class Widget extends AbstractDb
{
    protected function _construct(): void
    {
        $this->_init('vendor_widget', 'widget_id');
    }

    /**
     * @param int[] $ids
     * @return array<int, array<string, mixed>>
     */
    public function getActiveByIds(array $ids): array
    {
        $connection = $this->getConnection();
        $tableName  = $this->getMainTable();

        // ✅ quoteInto() binds the value safely
        $select = $connection->select()
            ->from($tableName, ['widget_id', 'title', 'content'])
            ->where('widget_id IN (?)', $ids)           // bound
            ->where('is_active = ?', 1)                 // bound
            ->where('store_id = ?', (int) $this->getStoreId()); // cast + bound

        return $connection->fetchAll($select);
    }

    /**
     * Named placeholders example.
     */
    public function searchByTitle(string $title, int $storeId): array
    {
        $connection = $this->getConnection();

        // Raw SQL with named placeholders — adapter binds them
        $sql = 'SELECT widget_id, title FROM ' . $this->getMainTable()
             . ' WHERE title LIKE :title AND store_id = :store_id AND is_active = 1';

        return $connection->fetchAll($sql, [
            ':title'    => '%' . $title . '%',
            ':store_id' => $storeId,
        ]);
    }
}
```

### 4.5 Why ORM Always Uses Bindings

`Magento\Framework\DB\Adapter\Pdo\Mysql` (`Laminas\Db\Adapter` internally) always routes values through `PDO::prepare()` + `PDO::bindValue()` or uses `quoteInto()`:

```php
// Simplified internals of Zend_Db_Select::where()
public function where($cond, $val = null, $type = null): static
{
    if ($val !== null) {
        $cond = $this->_adapter->quoteInto($cond, $val, $type);
        // quoteInto() calls PDO::quote() or uses prepared statement binding
    }
    $this->_parts[self::WHERE][] = $cond;
    return $this;
}
```

The ? placeholder is **never** achieved by string replacement — it calls the PDO driver binding, making injection impossible at the protocol level.

---

## 5. File Upload Security

### 5.1 Why File Uploads are Dangerous

- Uploading a PHP file → Remote Code Execution
- Path traversal in filename → overwriting critical files
- MIME-type spoofing → bypassing extension checks
- Serving user files from webroot → executing uploaded scripts
- Oversized files → Denial of Service

### 5.2 Using `Magento\Framework\File\Uploader`

```php
// app/code/Vendor/Module/Controller/Adminhtml/Widget/Upload.php
declare(strict_types=1);

namespace Vendor\Module\Controller\Adminhtml\Widget;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Framework\Controller\Result\JsonFactory;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\File\UploaderFactory;
use Magento\Framework\Filesystem;
use Magento\Framework\App\Filesystem\DirectoryList;
use Psr\Log\LoggerInterface;

class Upload extends Action
{
    private const ALLOWED_EXTENSIONS  = ['jpg', 'jpeg', 'png', 'gif', 'webp'];
    private const MAX_FILE_SIZE_BYTES  = 2 * 1024 * 1024; // 2 MB
    private const UPLOAD_SUBDIR        = 'vendor/module/images';

    public function __construct(
        Context $context,
        private readonly JsonFactory $resultJsonFactory,
        private readonly UploaderFactory $uploaderFactory,
        private readonly Filesystem $filesystem,
        private readonly LoggerInterface $logger
    ) {
        parent::__construct($context);
    }

    protected function _isAllowed(): bool
    {
        return $this->_authorization->isAllowed('Vendor_Module::manage_widgets');
    }

    public function execute(): \Magento\Framework\Controller\ResultInterface
    {
        $resultJson = $this->resultJsonFactory->create();

        try {
            /** @var \Magento\Framework\File\Uploader $uploader */
            $uploader = $this->uploaderFactory->create(['fileId' => 'image']);

            // 1. Restrict allowed file extensions (allowlist, not denylist)
            $uploader->setAllowedExtensions(self::ALLOWED_EXTENSIONS);

            // 2. Prevent executing uploaded files (forces correct MIME types)
            $uploader->setAllowRenameFiles(true);    // avoids filename collisions
            $uploader->setFilesDispersion(true);     // spreads files across subdirectories
            $uploader->setAllowCreateFolders(true);

            // 3. Validate MIME type via callback
            $uploader->addValidateCallback(
                'validate_image',
                $this,
                'validateImageMime'
            );

            // 4. Save outside webroot — pub/media is served by nginx, not PHP
            $mediaDir   = $this->filesystem->getDirectoryWrite(DirectoryList::MEDIA);
            $uploadPath = $mediaDir->getAbsolutePath(self::UPLOAD_SUBDIR);

            $result = $uploader->save($uploadPath);

            if (!$result) {
                throw new LocalizedException(__('File upload failed.'));
            }

            // 5. Return only the relative path, not the full server path
            return $resultJson->setData([
                'name'  => $result['name'],
                'url'   => $this->getUploadUrl($result['file']),
                'error' => 0,
            ]);

        } catch (LocalizedException $e) {
            $this->logger->warning('Widget image upload failed: ' . $e->getMessage());
            return $resultJson->setData(['error' => $e->getMessage(), 'errorcode' => 0]);
        } catch (\Exception $e) {
            $this->logger->critical('Widget image upload exception', ['exception' => $e]);
            return $resultJson->setData(['error' => __('An error occurred. Please try again.'), 'errorcode' => 0]);
        }
    }

    /**
     * Called by the Uploader validate callback.
     *
     * @throws LocalizedException
     */
    public function validateImageMime(string $filePath): void
    {
        // Check real MIME type using fileinfo, not the client-supplied Content-Type
        $finfo    = new \finfo(FILEINFO_MIME_TYPE);
        $mimeType = $finfo->file($filePath);

        $allowedMimeTypes = [
            'image/jpeg',
            'image/png',
            'image/gif',
            'image/webp',
        ];

        if (!in_array($mimeType, $allowedMimeTypes, true)) {
            throw new LocalizedException(
                __('File type "%1" is not allowed. Allowed types: %2', $mimeType, implode(', ', $allowedMimeTypes))
            );
        }

        // Check file size manually if needed (Uploader::setMaxFileSize exists too)
        if (filesize($filePath) > self::MAX_FILE_SIZE_BYTES) {
            throw new LocalizedException(__('File exceeds maximum allowed size of 2 MB.'));
        }
    }

    private function getUploadUrl(string $file): string
    {
        return $this->_url->getBaseUrl(['_type' => \Magento\Framework\UrlInterface::URL_TYPE_MEDIA])
            . self::UPLOAD_SUBDIR . '/' . ltrim($file, '/');
    }
}
```

### 5.3 Filename Sanitisation

The `Uploader` class sanitises filenames by default, but always double-check:

```php
// Magento\Framework\File\Uploader::_setUploadFileId() sanitises the name internally.
// It removes path separators and normalises characters.
// After save(), always use $result['name'] (the sanitised name), NOT the original.

// If you store filenames in DB, additionally sanitise before insertion:
$safeName = preg_replace('/[^a-zA-Z0-9_\-\.]/', '_', basename($result['name']));
// Remove double extensions (e.g. shell.php.jpg → shell_php.jpg)
$safeName = preg_replace('/\.(?:php[0-9]?|phtml|phar|pht|shtml|cgi)\./i', '_blocked_.', $safeName);
```

### 5.4 Nginx Configuration for Uploaded Files

Add to your Nginx config to prevent PHP execution inside `pub/media`:

```nginx
# Deny PHP execution in media directory
location ~* ^/pub/media/.*\.(php|phtml|phar|pl|py|cgi)$ {
    deny all;
}

# Disable directory listing
location /pub/media {
    autoindex off;
}
```

---

## 6. ACL (Access Control List) System

### 6.1 Concepts

| Term | Description |
|---|---|
| **Resource** | A named permission node (e.g. `Vendor_Module::manage_widgets`) |
| **Role** | A named group of resources assigned to admin users |
| **User** | An admin account assigned to one Role |
| **`_isAllowed()`** | Method in admin controllers returning bool based on ACL check |

### 6.2 Defining Resources in `etc/acl.xml`

```xml
<?xml version="1.0"?>
<!-- app/code/Vendor/Module/etc/acl.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Acl/etc/acl.xsd">
    <acl>
        <resources>
            <!-- All custom resources must be nested under Magento_Backend::admin -->
            <resource id="Magento_Backend::admin">

                <!-- Top-level menu section for your module -->
                <resource id="Vendor_Module::vendor_module"
                          title="Widget Manager"
                          sortOrder="100">

                    <!-- Sub-resource: viewing widgets -->
                    <resource id="Vendor_Module::manage_widgets"
                              title="Manage Widgets"
                              sortOrder="10"/>

                    <!-- Sub-resource: managing configuration -->
                    <resource id="Vendor_Module::configuration"
                              title="Widget Configuration"
                              sortOrder="20"/>

                    <!-- Sub-resource: deleting widgets (extra-sensitive) -->
                    <resource id="Vendor_Module::delete_widgets"
                              title="Delete Widgets"
                              sortOrder="30"/>
                </resource>

                <!-- Tie to Stores > Configuration in admin panel -->
                <resource id="Magento_Config::config">
                    <resource id="Vendor_Module::config_section"
                              title="Widget Manager Section"
                              sortOrder="150"/>
                </resource>
            </resource>
        </resources>
    </acl>
</config>
```

### 6.3 Protecting Admin Controllers

```php
// app/code/Vendor/Module/Controller/Adminhtml/Widget/Index.php
declare(strict_types=1);

namespace Vendor\Module\Controller\Adminhtml\Widget;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Framework\View\Result\PageFactory;

class Index extends Action
{
    // Magento uses this constant automatically in _isAllowed()
    public const ADMIN_RESOURCE = 'Vendor_Module::manage_widgets';

    public function __construct(
        Context $context,
        private readonly PageFactory $resultPageFactory
    ) {
        parent::__construct($context);
    }

    // The framework calls _isAllowed() before execute() for every admin action.
    // The default implementation checks $this->_authorization->isAllowed(static::ADMIN_RESOURCE).
    // You can override for custom logic:
    protected function _isAllowed(): bool
    {
        return $this->_authorization->isAllowed(static::ADMIN_RESOURCE);
    }

    public function execute(): \Magento\Framework\View\Result\Page
    {
        $resultPage = $this->resultPageFactory->create();
        $resultPage->setActiveMenu('Vendor_Module::manage_widgets');
        $resultPage->getConfig()->getTitle()->prepend(__('Widget Manager'));
        return $resultPage;
    }
}
```

```php
// Delete controller — requires a more specific permission
// app/code/Vendor/Module/Controller/Adminhtml/Widget/Delete.php
declare(strict_types=1);

namespace Vendor\Module\Controller\Adminhtml\Widget;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Vendor\Module\Api\WidgetRepositoryInterface;

class Delete extends Action
{
    public const ADMIN_RESOURCE = 'Vendor_Module::delete_widgets'; // more restrictive

    public function __construct(
        Context $context,
        private readonly WidgetRepositoryInterface $widgetRepository
    ) {
        parent::__construct($context);
    }

    public function execute(): \Magento\Framework\Controller\ResultInterface
    {
        $resultRedirect = $this->resultRedirectFactory->create();
        $widgetId = (int) $this->getRequest()->getParam('id');

        try {
            $this->widgetRepository->deleteById($widgetId);
            $this->messageManager->addSuccessMessage(__('Widget has been deleted.'));
        } catch (\Exception $e) {
            $this->messageManager->addErrorMessage(__('An error occurred while deleting the widget.'));
        }

        return $resultRedirect->setPath('*/*/');
    }
}
```

### 6.4 Checking Permissions in Non-Controller Code

```php
// app/code/Vendor/Module/Block/Adminhtml/Widget/Grid.php
declare(strict_types=1);

namespace Vendor\Module\Block\Adminhtml\Widget;

use Magento\Backend\Block\Widget\Grid\Extended;
use Magento\Framework\AuthorizationInterface;

class Grid extends Extended
{
    public function __construct(
        \Magento\Backend\Block\Template\Context $context,
        \Magento\Backend\Helper\Data $backendHelper,
        private readonly AuthorizationInterface $authorization,
        array $data = []
    ) {
        parent::__construct($context, $backendHelper, $data);
    }

    public function canDelete(): bool
    {
        return $this->authorization->isAllowed('Vendor_Module::delete_widgets');
    }
}
```

```php
// In a service class:
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Framework\AuthorizationInterface;
use Magento\Framework\Exception\AuthorizationException;

class WidgetService
{
    public function __construct(
        private readonly AuthorizationInterface $authorization
    ) {}

    public function deleteWidget(int $widgetId): void
    {
        if (!$this->authorization->isAllowed('Vendor_Module::delete_widgets')) {
            throw new AuthorizationException(__('You are not authorised to delete widgets.'));
        }
        // … proceed with deletion
    }
}
```

### 6.5 ACL in Admin Menu (`etc/adminhtml/menu.xml`)

```xml
<?xml version="1.0"?>
<!-- app/code/Vendor/Module/etc/adminhtml/menu.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/menu.xsd">
    <menu>
        <!-- Top-level menu item; hidden if user lacks the resource -->
        <add id="Vendor_Module::vendor_module"
             title="Widget Manager"
             module="Vendor_Module"
             sortOrder="100"
             resource="Vendor_Module::vendor_module"/>

        <!-- Sub-menu item -->
        <add id="Vendor_Module::widgets_index"
             title="All Widgets"
             module="Vendor_Module"
             sortOrder="10"
             parent="Vendor_Module::vendor_module"
             action="vendor_module/widget/index"
             resource="Vendor_Module::manage_widgets"/>

        <add id="Vendor_Module::widget_configuration"
             title="Configuration"
             module="Vendor_Module"
             sortOrder="20"
             parent="Vendor_Module::vendor_module"
             action="adminhtml/system_config/edit/section/vendor_module"
             resource="Vendor_Module::configuration"/>
    </menu>
</config>
```

The `resource` attribute in `menu.xml` must match a resource `id` declared in `acl.xml`. Magento automatically hides menu items the current user cannot access.

### 6.6 ACL in `system.xml` (Config Sections)

```xml
<?xml version="1.0"?>
<!-- app/code/Vendor/Module/etc/adminhtml/system.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Config:etc/system_file.xsd">
    <system>
        <section id="vendor_module"
                 translate="label"
                 sortOrder="150"
                 showInDefault="1"
                 showInWebsite="1"
                 showInStore="1">
            <label>Widget Manager</label>
            <!-- ACL resource protecting this whole config section -->
            <resource>Vendor_Module::config_section</resource>
            <tab>catalog</tab>
            <group id="general"
                   translate="label"
                   sortOrder="10"
                   showInDefault="1"
                   showInWebsite="1"
                   showInStore="1">
                <label>General</label>
                <field id="enabled"
                       translate="label"
                       type="select"
                       sortOrder="10"
                       showInDefault="1"
                       showInWebsite="1"
                       showInStore="1">
                    <label>Enable Widget Manager</label>
                    <source_model>Magento\Config\Model\Config\Source\Yesno</source_model>
                </field>
                <!-- Sensitive field — value is encrypted in DB, masked in UI -->
                <field id="api_key"
                       translate="label"
                       type="obscure"
                       sortOrder="20"
                       showInDefault="1"
                       showInWebsite="0"
                       showInStore="0">
                    <label>API Key</label>
                    <backend_model>Magento\Config\Model\Config\Backend\Encrypted</backend_model>
                </field>
            </group>
        </section>
    </system>
</config>
```

### 6.7 Role and User Management Flow

```
Admin → System → Permissions → User Roles
    → Create Role "Widget Editor"
    → Assign resources: Vendor_Module::manage_widgets (tick)
    → Save Role

Admin → System → Permissions → All Users
    → Create User "jane@example.com"
    → Assign to Role "Widget Editor"
    → Save User

Result: Jane can see Widget Manager menu and list widgets,
        but cannot delete (Vendor_Module::delete_widgets not granted).
```

---

## 7. Admin Session Security

### 7.1 Session Lifetime and Locking

```php
// app/etc/env.php — override session lifetime programmatically
return [
    'session' => [
        'save'    => 'redis',
        'redis'   => [
            'host'                => '127.0.0.1',
            'port'                => '6379',
            'password'            => '',
            'timeout'             => '2.5',
            'persistent_identifier' => '',
            'database'            => '2',
            'compression_threshold' => '2048',
            'compression_library'  => 'gzip',
            'log_level'           => '4',
            'max_concurrency'     => '6',
            'break_after_frontend' => '5',
            'break_after_adminhtml' => '30',
            'first_lifetime'       => '600',
            'bot_first_lifetime'   => '60',
            'bot_lifetime'         => '7200',
            'disable_locking'      => '0',   // 0 = locking enabled (prevents session race conditions)
            'min_lifetime'         => '60',
            'max_lifetime'         => '2592000',
        ],
    ],
];
```

### 7.2 Password Hashing

Magento 2.4.x uses **Argon2ID** by default (PHP 7.3+), with bcrypt as a fallback.

```php
// vendor/magento/framework/Encryption/Encryptor.php (simplified)
declare(strict_types=1);

namespace Magento\Framework\Encryption;

class Encryptor implements EncryptorInterface
{
    private const HASH_VERSION_ARGON2ID = 3;
    private const HASH_VERSIONS = [
        self::HASH_VERSION_ARGON2ID => PASSWORD_ARGON2ID,
        // 2 => PASSWORD_BCRYPT (legacy)
    ];

    public function getHash(string $password, bool $salt = false, int $version = self::HASH_VERSION_ARGON2ID): string
    {
        return implode(':', [
            password_hash($password, PASSWORD_ARGON2ID, [
                'memory_cost' => 65536,  // 64 MB
                'time_cost'   => 4,
                'threads'     => 3,
            ]),
            $version,
        ]);
    }

    public function validateHash(string $password, string $hash): bool
    {
        [$storedHash, ] = explode(':', $hash, 2);
        return password_verify($password, $storedHash);
    }
}
```

**Using the encryptor in your module:**

```php
// app/code/Vendor/Module/Model/ApiKeyManager.php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Framework\Encryption\EncryptorInterface;

class ApiKeyManager
{
    public function __construct(
        private readonly EncryptorInterface $encryptor
    ) {}

    /**
     * Encrypt before saving to database.
     */
    public function encryptApiKey(string $rawKey): string
    {
        return $this->encryptor->encrypt($rawKey);
    }

    /**
     * Decrypt when reading from database.
     */
    public function decryptApiKey(string $encryptedKey): string
    {
        return $this->encryptor->decrypt($encryptedKey);
    }
}
```

### 7.3 Two-Factor Authentication (2FA)

Magento 2.4+ ships `Magento_TwoFactorAuth` enabled by default. It is **mandatory** for all admin users in a production environment. You cannot disable it without accepting the security risk.

```bash
# List enabled 2FA providers
bin/magento config:show twofactorauth/general/force_providers

# Allow only Google Authenticator (TOTP)
bin/magento config:set twofactorauth/general/force_providers google

# Generate a one-time bypass for a specific admin (for CI/CD or emergency)
bin/magento security:tfa:google:set-secret <admin_username> <base32_secret>

# Disable 2FA for testing ONLY (never in production)
bin/magento module:disable Magento_TwoFactorAuth
```

Supported providers in 2.4.8:

| Provider | Module | Algorithm |
|---|---|---|
| Google Authenticator | `Magento_TwoFactorAuth` | TOTP (RFC 6238) |
| Duo Security | `Magento_TwoFactorAuth` | Duo Web SDK v4 |
| Authy | `Magento_TwoFactorAuth` | TOTP + SMS |
| U2F/WebAuthn | `Magento_TwoFactorAuth` | FIDO2 hardware key |

### 7.4 Session Regeneration

Magento regenerates the session ID on login automatically:

```php
// vendor/magento/module-customer/Model/Session.php
// After successful authentication:
$this->_session->regenerateId();      // prevents session fixation
$this->_session->setCustomerId($id);
```

For admin:

```php
// vendor/magento/module-backend/Model/Auth/Session.php
$this->setIsFirstPageAfterLogin(true);
$this->getStorage()->regenerate();   // regenerates session ID
```

---

## 8. API Security

### 8.1 Authentication Token Types

| Token Type | Issued by | Lifespan | Use Case |
|---|---|---|---|
| **Integration OAuth** | `POST /V1/integration/admin/token` (not standard; use admin panel) | Permanent until revoked | Server-to-server integrations |
| **Admin Token** | `POST /V1/integration/admin/token` | 4 h (configurable) | Admin API scripts |
| **Customer Token** | `POST /V1/integration/customer/token` | 1 h (configurable) | Customer-facing mobile apps |
| **Guest token** | No auth (scope limited) | Per-request | Cart operations by guests |

### 8.2 OAuth 1.0a for REST Integrations

```php
// Configuring an integration programmatically
// app/code/Vendor/Module/Setup/Patch/Data/CreateIntegration.php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Integration\Api\IntegrationServiceInterface;
use Magento\Integration\Api\OauthServiceInterface;

class CreateIntegration implements DataPatchInterface
{
    public function __construct(
        private readonly IntegrationServiceInterface $integrationService,
        private readonly OauthServiceInterface $oauthService
    ) {}

    public function apply(): self
    {
        $integrationData = [
            'name'                 => 'Vendor Module Integration',
            'email'                => 'integration@vendor.example.com',
            'endpoint'             => 'https://api.vendor.example.com/magento/callback',
            'status'               => \Magento\Integration\Model\Integration::STATUS_ACTIVE,
            'setup_type'           => \Magento\Integration\Model\Integration::TYPE_CONFIG,
            'resource'             => [
                'Magento_Catalog::catalog',
                'Magento_Sales::sales',
                'Vendor_Module::manage_widgets',
            ],
        ];

        $integration = $this->integrationService->create($integrationData);
        $this->oauthService->createAccessToken((int) $integration->getId(), true);

        return $this;
    }

    public static function getDependencies(): array { return []; }
    public function getAliases(): array { return []; }
}
```

### 8.3 Securing Custom REST Endpoints

```xml
<!-- app/code/Vendor/Module/etc/webapi.xml -->
<?xml version="1.0"?>
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Webapi:etc/webapi.xsd">

    <!-- Admin-only endpoint (requires admin token) -->
    <route url="/V1/vendor-module/widgets" method="GET">
        <service class="Vendor\Module\Api\WidgetRepositoryInterface" method="getList"/>
        <resources>
            <resource ref="Vendor_Module::manage_widgets"/>
        </resources>
    </route>

    <!-- Customer endpoint (requires customer token) -->
    <route url="/V1/vendor-module/my-widgets" method="GET">
        <service class="Vendor\Module\Api\CustomerWidgetRepositoryInterface" method="getMyWidgets"/>
        <resources>
            <resource ref="self"/>   <!-- "self" = authenticated customer only -->
        </resources>
    </route>

    <!-- Anonymous endpoint (public) — be very careful -->
    <route url="/V1/vendor-module/public-widgets" method="GET">
        <service class="Vendor\Module\Api\PublicWidgetRepositoryInterface" method="getPublicList"/>
        <resources>
            <resource ref="anonymous"/>   <!-- No auth required -->
        </resources>
    </route>
</routes>
```

> **Warning:** `anonymous` resources are completely public. Never expose write operations as anonymous.

### 8.4 Rate Limiting

Magento 2.4.7+ introduced native rate limiting for the REST and GraphQL APIs.

```xml
<!-- app/code/Vendor/Module/etc/webapi.xml — per-route rate limit -->
<route url="/V1/vendor-module/widgets" method="POST">
    <service class="Vendor\Module\Api\WidgetRepositoryInterface" method="save"/>
    <resources>
        <resource ref="Vendor_Module::manage_widgets"/>
    </resources>
    <data>
        <!-- 30 requests per minute per IP for this endpoint -->
        <item name="rateLimitConfig" xsi:type="array">
            <item name="limit" xsi:type="number">30</item>
            <item name="period" xsi:type="number">60</item>
        </item>
    </data>
</route>
```

Global rate limiting in `env.php`:

```php
// app/etc/env.php
return [
    'rate_limiting' => [
        'rest'     => [
            'limit'  => 1000,  // requests
            'period' => 3600,  // per hour
        ],
        'graphql'  => [
            'limit'  => 500,
            'period' => 3600,
        ],
    ],
];
```

### 8.5 CORS for GraphQL / REST

```php
// app/etc/env.php — allowed origins
return [
    'http' => [
        'cors' => [
            'allowed_origins' => [
                'https://headless.example.com',
                'https://app.example.com',
            ],
            'allowed_methods' => ['GET', 'POST', 'PUT', 'DELETE'],
            'allowed_headers' => ['Content-Type', 'Authorization', 'X-Requested-With'],
            'max_age'         => 3600,
        ],
    ],
];
```

---

## 9. Secure Configuration

### 9.1 `env.php` vs `config.php`

| File | Contains | Committed to VCS? |
|---|---|---|
| `app/etc/env.php` | DB credentials, encryption key, Redis config, admin path | **Never** |
| `app/etc/config.php` | Module list, store/website structure, non-sensitive config values | Yes |

```php
// app/etc/env.php — the secrets file (add to .gitignore!)
return [
    'crypt' => [
        'key' => 'your_32_byte_encryption_key_here',  // NEVER commit
    ],
    'db' => [
        'connection' => [
            'default' => [
                'host'     => 'localhost',
                'dbname'   => 'magento',
                'username' => 'magento_user',
                'password' => 'secret_password',  // NEVER commit
                'active'   => '1',
            ],
        ],
    ],
    'backend' => [
        'frontName' => 'admin_d7f3a9',  // randomised admin URL
    ],
];
```

### 9.2 Sensitive Config Values in `system.xml`

Mark fields as sensitive so they are excluded from `config:dump` and masked in the UI:

```xml
<field id="api_secret"
       translate="label"
       type="obscure"
       sortOrder="30"
       showInDefault="1">
    <label>API Secret</label>
    <!-- Encrypts value in core_config_data -->
    <backend_model>Magento\Config\Model\Config\Backend\Encrypted</backend_model>
    <!-- Marks as sensitive — excluded from bin/magento app:config:dump -->
    <attribute type="sensitive">1</attribute>
</field>
```

Retrieve the encrypted value:

```php
// app/code/Vendor/Module/Model/Config.php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Framework\Encryption\EncryptorInterface;
use Magento\Store\Model\ScopeInterface;

class Config
{
    private const XML_PATH_API_SECRET = 'vendor_module/general/api_secret';

    public function __construct(
        private readonly ScopeConfigInterface $scopeConfig,
        private readonly EncryptorInterface $encryptor
    ) {}

    public function getApiSecret(): string
    {
        $encrypted = (string) $this->scopeConfig->getValue(
            self::XML_PATH_API_SECRET,
            ScopeInterface::SCOPE_STORE
        );
        return $this->encryptor->decrypt($encrypted);
    }
}
```

### 9.3 HTTPS Enforcement

```bash
# Force HTTPS in store base URLs
bin/magento config:set web/secure/base_url https://shop.example.com/
bin/magento config:set web/secure/use_in_frontend 1
bin/magento config:set web/secure/use_in_adminhtml 1

# Force HTTPS redirect
bin/magento config:set web/secure/offloader_header X-Forwarded-Proto  # for load balancers
```

```php
// app/etc/env.php — additional HTTPS enforcement
return [
    'system' => [
        'default' => [
            'web' => [
                'secure' => [
                    'use_in_frontend'   => '1',
                    'use_in_adminhtml'  => '1',
                ],
            ],
        ],
    ],
];
```

---

## 10. Security Headers

### 10.1 Configuring Headers via Magento

Magento 2.4.8 lets you define custom HTTP response headers in `etc/security.xml` (for CSP) and via Nginx/Apache configuration for the remainder.

```nginx
# nginx/conf.d/magento-security-headers.conf
server {
    # Prevent your site being embedded in an iframe (clickjacking)
    add_header X-Frame-Options "SAMEORIGIN" always;

    # Prevent MIME-type sniffing
    add_header X-Content-Type-Options "nosniff" always;

    # Legacy XSS filter (still useful for old browsers)
    add_header X-XSS-Protection "1; mode=block" always;

    # Force HTTPS for 1 year (HSTS)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # Control Referer information
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Permissions / Feature Policy
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
}
```

### 10.2 Header Reference Table

| Header | Value | Threat Mitigated |
|---|---|---|
| `X-Frame-Options` | `DENY` or `SAMEORIGIN` | Clickjacking |
| `X-Content-Type-Options` | `nosniff` | MIME-type confusion attacks |
| `X-XSS-Protection` | `1; mode=block` | Reflected XSS in old browsers |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | SSL stripping / MITM |
| `Content-Security-Policy` | (see CSP section) | XSS, data injection |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | URL leakage in Referer header |
| `Permissions-Policy` | `geolocation=(), microphone=()` | Unwanted browser API access |

### 10.3 CSP Reporting Endpoint

Set up a reporting endpoint to collect CSP violations without blocking legitimate content:

```xml
<!-- app/code/Vendor/Module/etc/csp_whitelist.xml -->
<?xml version="1.0"?>
<csp_whitelist xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Csp/etc/csp_whitelist.xsd">
    <policies>
        <policy id="report-uri">
            <values>
                <value id="report-uri-value" type="host">https://csp-reports.vendor.example.com/collect</value>
            </values>
        </policy>
    </policies>
</csp_whitelist>
```

---

## 11. Secure Development Practices

### 11.1 Input Validation — Always Validate, Never Trust

```php
// app/code/Vendor/Module/Model/Widget/DataProcessor.php
declare(strict_types=1);

namespace Vendor\Module\Model\Widget;

use Magento\Framework\Exception\InputException;
use Magento\Framework\Validator\ValidateException;

class DataProcessor
{
    private const ALLOWED_WIDGET_TYPES = ['slider', 'grid', 'list'];
    private const MAX_TITLE_LENGTH     = 255;
    private const MIN_TITLE_LENGTH     = 3;

    /**
     * @param array<string, mixed> $data
     * @throws InputException
     */
    public function validate(array $data): void
    {
        $errors = [];

        // 1. Required field check
        if (empty($data['title'])) {
            $errors[] = __('Widget title is required.');
        }

        // 2. Length check
        if (isset($data['title'])) {
            $len = mb_strlen((string) $data['title'], 'UTF-8');
            if ($len < self::MIN_TITLE_LENGTH) {
                $errors[] = __('Widget title must be at least %1 characters.', self::MIN_TITLE_LENGTH);
            }
            if ($len > self::MAX_TITLE_LENGTH) {
                $errors[] = __('Widget title must not exceed %1 characters.', self::MAX_TITLE_LENGTH);
            }
        }

        // 3. Allowlist check (never denylist)
        if (isset($data['type']) && !in_array($data['type'], self::ALLOWED_WIDGET_TYPES, true)) {
            $errors[] = __('Invalid widget type "%1". Allowed: %2', $data['type'], implode(', ', self::ALLOWED_WIDGET_TYPES));
        }

        // 4. Integer check
        if (isset($data['sort_order']) && !is_numeric($data['sort_order'])) {
            $errors[] = __('Sort order must be a number.');
        }

        // 5. URL validation
        if (isset($data['redirect_url']) && !filter_var($data['redirect_url'], FILTER_VALIDATE_URL)) {
            $errors[] = __('Redirect URL is not a valid URL.');
        }

        if ($errors) {
            $exception = new InputException();
            foreach ($errors as $error) {
                $exception->addError($error);
            }
            throw $exception;
        }
    }
}
```

### 11.2 Output Encoding — Escape on Output, Not on Input

A common mistake is HTML-encoding data when it enters the system. This corrupts data for non-HTML consumers (JSON API, email, CSV export). Always store raw data and escape at the point of output:

```php
// ❌ WRONG — storing escaped data in DB
$widget->setTitle(htmlspecialchars($title));   // data corrupted for API/email

// ✅ CORRECT — store raw, escape on output
$widget->setTitle($title);   // store raw
// Then in PHTML:
// echo $escaper->escapeHtml($block->getWidget()->getTitle());
```

### 11.3 Least Privilege Principle

```xml
<!-- Always grant the MINIMUM required resources -->
<!-- ❌ WRONG: granting Magento_Backend::admin (full admin access) -->
<resource ref="Magento_Backend::admin"/>

<!-- ✅ CORRECT: grant only what the integration needs -->
<resources>
    <resource ref="Magento_Catalog::categories"/>
    <resource ref="Vendor_Module::manage_widgets"/>
</resources>
```

```php
// DB user principle: create a read-only replica user for reporting queries
// In your env.php:
return [
    'db' => [
        'connection' => [
            'default' => [        // read-write (writes + critical reads)
                'host'     => 'db-primary.internal',
                'username' => 'magento_rw',
            ],
            'indexer' => [        // read-only replica (indexers, reports)
                'host'     => 'db-replica.internal',
                'username' => 'magento_ro',
            ],
        ],
    ],
];
```

### 11.4 Secrets Management

```bash
# Never store secrets in source code.
# Use environment variables injected by your deployment tool:

# In docker-compose.yml / Kubernetes secret:
MAGENTO_DB_PASSWORD: "${DB_PASSWORD}"
MAGENTO_CRYPT_KEY:   "${CRYPT_KEY}"

# Generate a new crypt key:
bin/magento encryption:key:change

# Rotate the admin password safely:
bin/magento admin:user:create --admin-user=newadmin --admin-password='V3ry$ecure!'
```

**`.gitignore` entries (required):**

```gitignore
# Magento secrets — NEVER commit these
app/etc/env.php
var/
pub/media/
pub/static/
generated/
.env
*.log
```

### 11.5 Security Review Checklist

Use this checklist before merging any Magento module PR:

```
CSRF
  [ ] All POST controllers implement CsrfAwareActionInterface
  [ ] Form keys present in all custom POST forms
  [ ] AJAX calls pass form_key in request body

XSS
  [ ] All template output uses $escaper->escapeHtml() / escapeHtmlAttr() / escapeUrl()
  [ ] No raw echo without @noEscape comment and justification
  [ ] No innerHTML assignments in JS without sanitisation
  [ ] CSP whitelist updated for any new external domains

SQL Injection
  [ ] No raw string interpolation in SQL queries
  [ ] All collection filters use addFieldToFilter() or bound where()
  [ ] Direct queries use named/positional placeholders

File Upload
  [ ] Extension allowlist (not denylist) defined
  [ ] MIME type validated via fileinfo (not client header)
  [ ] Files saved in pub/media (not in PHP-executable dirs)
  [ ] Filenames sanitised before storage

ACL
  [ ] Every admin controller has ADMIN_RESOURCE constant defined
  [ ] ACL resources declared in etc/acl.xml
  [ ] Menu items have correct resource= attribute
  [ ] API endpoints have correct resource= in webapi.xml

Secrets
  [ ] No credentials in source code
  [ ] env.php in .gitignore
  [ ] Sensitive fields use Encrypted backend_model

Session
  [ ] 2FA not disabled on production
  [ ] Admin session lifetime <= 3600 s
  [ ] Redis session storage configured (not files)

Dependencies
  [ ] composer audit run and all critical/high CVEs addressed
  [ ] No dev dependencies in production
```

### 11.6 Useful Security CLI Commands

```bash
# Check for known vulnerabilities in dependencies
composer audit

# Scan Magento for common misconfigurations
bin/magento security:check  # Available via Magento Security Scan API

# Generate a cryptographically secure random string (for tokens, salts)
bin/magento dev:generate:secret

# Rotate the encryption key
bin/magento encryption:key:change

# Remove deprecated admin notifications (reduces attack surface)
bin/magento security:manage-admin-notifications --delete-all

# Validate module XML against XSD schemas
bin/magento dev:xml:convert

# List all registered admin accounts (audit)
bin/magento admin:user:list

# Force all admins to reset password on next login
bin/magento admin:user:reset-password --all
```

---

## Summary

| Area | Key Class / File | Magento Mechanism |
|---|---|---|
| CSRF | `Magento\Framework\Data\Form\FormKey` | Form key in every POST; `CsrfAwareActionInterface` |
| XSS | `Magento\Framework\Escaper` | Context-aware escaping; CSP via `etc/csp_whitelist.xml` |
| SQL Injection | `Magento\Framework\DB\Adapter\Pdo\Mysql` | Bound parameters via `where('col = ?', $val)` |
| File Upload | `Magento\Framework\File\Uploader` | Extension allowlist + MIME validation + media dir |
| ACL | `Magento\Framework\Acl` | `etc/acl.xml` → `ADMIN_RESOURCE` → `_isAllowed()` |
| Session | `Magento\Backend\Model\Auth\Session` | Argon2ID hashing; 2FA; Redis sessions; ID regeneration |
| API | `etc/webapi.xml` | OAuth 1.0a; Bearer tokens; `resource` attribute; rate limiting |
| Config Secrets | `app/etc/env.php` | Never in VCS; Encrypted backend_model; sensitive attribute |
| Headers | Nginx + `etc/csp_whitelist.xml` | HSTS, X-Frame-Options, nosniff, CSP |

Security in Magento 2 is layered and systematic. Every layer exists for a reason; removing any of them increases risk. The safest Magento module is one that:

1. **Validates** all input at the boundary
2. **Escapes** all output at the point of rendering
3. **Binds** all database parameters
4. **Checks** ACL before every sensitive operation
5. **Stores** no secrets in version control

These five habits, applied consistently, eliminate the vast majority of OWASP Top 10 vulnerabilities from your codebase.