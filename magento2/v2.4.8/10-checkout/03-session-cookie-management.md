---
title: "03 - Session & Cookie Management"
description: "Comprehensive deep dive into Magento 2.4.8 session and cookie management: PHPSESSID, customer/admin sessions, cookie configuration, Redis session backend, form keys, persistent cart, GDPR compliance, and session validation"
tags:
  - magento2
  - sessions
  - cookies
  - php
  - redis
  - security
  - checkout
rank: 3
pathways:
  - magento2-deep-dive
see_also:
  - /home/letiendat1002/workspace/personal/courses/magento2/v2.4.8/10-checkout/README.md
  - /home/letiendat1002/workspace/personal/courses/magento2/v2.4.8/10-checkout/08-order-extension.md
---

# Session & Cookie Management

Magento 2's session and cookie system is the invisible infrastructure that holds the authenticated state of every user — customers browsing the storefront, administrators managing the backend, and API clients interacting via web services. Understanding this system is essential for debugging authentication issues, optimizing performance, ensuring GDPR compliance, and building extensions that correctly preserve state across requests.

This chapter covers the complete session and cookie stack in Magento 2.4.8: from the low-level Zend Framework session wrappers to the high-level `CustomerSession` and `AdminAuthSession` models, from `web/session/` configuration values that control cookie behavior to the Redis-based session handler that scales to production traffic.

---

## 1. Magento's Session Abstraction

### The Zend Framework Heritage

Magento 2 inherited its session architecture from Zend Framework 1. In ZF1, `Zend_Session` was the core session manager that wrapped PHP's native `session_*` functions with an object-oriented interface supporting namespacing, expiration, and save handlers. Magento 2 carries this pattern forward through `Magento\Framework\Session`, which provides a PSR-7-aware wrapper around Zend Framework's session components.

The key classes in Magento's session abstraction layer are:

| Class | Role |
|-------|------|
| `Magento\Framework\Session\SessionManager` | Base session manager; wraps Zend session |
| `Magento\Framework\Session\Generic` | Generic session container used by most Magento components |
| `Magento\Framework\Session\SaveHandler` | Interface for custom session save handlers |
| `Magento\Framework\Session\SaveHandlerInterface` | Contract defining `read`, `write`, `destroy`, `gc` |
| `Magento\Framework\Session\Config\ConfigInterface` | Session configuration (lifetime, name, cookie params) |

### `Magento\Framework\Session\SessionManager`

`SessionManager` is the foundation of Magento's session system. It extends `Zend\Session\SessionManager` and provides:

- Automatic session start with `registerShutdownFunction()`
- Session namespace management via `start()`, `writeClose()`
- Cookie management integration
- Session validators for security (`HttpUserAgent`, `RemoteAddr`)
- `getSessionId()`, `getName()`, `setName()` methods

The key distinction from raw PHP `session_start()` is that `SessionManager` integrates with Magento's object manager for dependency injection, making sessions testable and overridable via `di.xml`.

```php
<?php
// vendor/magento/framework/Session/SessionManager.php
declare(strict_types=1);

namespace Magento\Framework\Session;

use Zend\Session\SessionManager as ZendSessionManager;

class SessionManager extends ZendSessionManager
{
    /**
     * @param SessionConfigInterface $sessionConfig
     * @param SaveHandlerInterface $saveHandler
     * @param ValidatorChain $validatorChain
     */
    public function __construct(
        \Magento\Framework\Session\Config\ConfigInterface $sessionConfig,
        \Magento\Framework\Session\SaveHandlerInterface $saveHandler,
        \Magento\Framework\Session\ValidatorChain $validatorChain
    ) {
        $this->sessionConfig = $sessionConfig;
        $this->saveHandler = $saveHandler;
        $this->validatorChain = $validatorChain;
        parent::__construct($sessionConfig, $saveHandler, $validatorChain);
    }

    /**
     * Start the session if not already started
     */
    public function start(): void
    {
        if (!$this->getId() || !$this->isStarted()) {
            // Register shutdown function to ensure session is written on exit
            register_shutdown_function([$this, 'writeClose']);
            parent::start();
        }
    }

    /**
     * Get the current session ID
     */
    public function getSessionId(): string
    {
        return $this->getId();
    }

    /**
     * Get the session name (e.g., PHPSESSID, adminhtml)
     */
    public function getName(): string
    {
        return $this->getConfig()->getName();
    }
}
```

### `Magento\Framework\Session\Generic`

`Generic` is the most commonly used session container class. It extends `Magento\Framework\Session\SessionManager` and provides a namespaced key-value store backed by the session:

```php
<?php
// Usage in a block or model
use Magento\Framework\Session\Generic;

class MyBlock extends \Magento\Framework\View\Element\Template
{
    public function __construct(
        \Magento\Framework\View\Element\Template\Context $context,
        private readonly Generic $session,
    ) {
        parent::__construct($context);
    }

    public function setCustomData(string $value): void
    {
        // Stores in session: $_SESSION['mage-generic']['custom_key']
        $this->session->setCustomKey($value);
    }

    public function getCustomData(): ?string
    {
        return $this->session->getCustomKey();
    }

    public function unsetCustomData(): void
    {
        $this->session->unsCustomKey();
    }
}
```

`Generic` also stores flash messages, customer data, and catalog search params under its namespace.

### `Magento\Framework\Session\SaveHandler`

The `SaveHandler` class wraps the actual session data storage. In Magento 2.4.8, the default save handler is `Magento\Framework\Session\SaveHandler\File` (for `file`-based sessions) but can be replaced via `di.xml` preferences. The save handler is configured in `app/etc/env.php`:

```php
<?php
// app/etc/env.php
return [
    // ...
    'session' => [
        'save' => 'files',    // or 'db', 'redis', 'memcache'
    ],
    // ...
];
```

---

## 2. Session Names

### Default Session Names

PHP's default session name is `PHPSESSID`. Magento uses different session names to isolate different contexts:

| Session Name | Context | Set By |
|-------------|---------|--------|
| `PHPSESSID` | Frontend (default PHP fallback) | PHP core |
| `adminhtml` | Admin panel | `Magento\Backend\Model\Session` |
| `frontend` | Frontend storefront | `Magento\Customer\Model\Session` |
| `section_data` | Customer section data (cached) | `Magento\Customer\CustomerData\JsInit` |

### How Session Names Are Set

The session name is controlled by `web/session_configuration` XML or by the session manager directly. In `etc/di.xml`:

```xml
<!-- vendor/magento/module-backend/etc/di.xml -->
<type name="Magento\Framework\Session\Generic">
    <arguments>
        <argument name="sessionName" xsi:type="string">adminhtml</argument>
    </arguments>
</type>
```

### `getSessionName()` and `setSessionName()`

```php
<?php
use Magento\Framework\Session\SessionManager;

$sessionManager = $objectManager->get(SessionManager::class);

// Get current session name
$name = $sessionManager->getName(); // e.g., 'adminhtml' or 'frontend'

// Set a custom session name (rarely needed, but possible)
$sessionManager->setName('custom_session');
$newName = $sessionManager->getName(); // 'custom_session'
```

### Cookie Domain Configuration

The cookie domain is controlled by `web/cookie/cookie_domain` in core_config_data or via `etc/env.php`:

```php
<?php
// app/etc/env.php — sets cookie domain globally
return [
    'cookie' => [
        'domain' => '.example.com',  // Leading dot allows subdomain access
    ],
];
```

Or via database:

```sql
INSERT INTO core_config_data (scope, scope_id, path, value)
VALUES ('default', 0, 'web/cookie/cookie_domain', '.example.com');
```

The cookie domain affects which domains receive the session cookie. A leading dot (`.example.com`) makes the cookie available to all subdomains. A specific subdomain (`www.example.com`) restricts it. For cross-domain cart sharing across subdomains, you must set the parent domain with a leading dot.

---

## 3. Customer Sessions

### `Magento\Customer\Model\Session`

The `CustomerSession` is the primary interface for customer authentication state on the frontend. It is a request-scoped singleton that lives throughout the customer browsing session:

```php
<?php
// vendor/magento/module-customer/Model/Session.php
declare(strict_types=1);

namespace Magento\Customer\Model;

use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Framework\Session\Generic;

class Session extends Generic
{
    public const CUSTOMER_GROUP_ID = 'customer_group_id';

    /**
     * @var CustomerRepositoryInterface
     */
    private $customerRepository;

    /**
     * @param \Magento\Framework\App\RequestInterface $request
     * @param \Magento\Framework\Session\EntityFactoryInterface $entityFactory
     * @param \Magento\Store\Model\StoreManagerInterface $storeManager
     * @param \Magento\Customer\Api\CustomerRepositoryInterface $customerRepository
     */
    public function __construct(
        \Magento\Framework\App\RequestInterface $request,
        \Magento\Framework\Session\EntityFactoryInterface $entityFactory,
        \Magento\Store\Model\StoreManagerInterface $storeManager,
        \Magento\Customer\Api\CustomerRepositoryInterface $customerRepository,
    ) {
        parent::__construct($request, $entityFactory, $storeManager);
        $this->customerRepository = $customerRepository;
    }

    /**
     * Check if customer is logged in
     */
    public function isLoggedIn(): bool
    {
        return $this->getCustomerId() !== null;
    }

    /**
     * Get logged-in customer ID
     */
    public function getCustomerId(): ?int
    {
        return $this->getData('customer_id');
    }

    /**
     * Get logged-in customer entity
     */
    public function getCustomer(): ?CustomerInterface
    {
        $customerId = $this->getCustomerId();
        if ($customerId) {
            return $this->customerRepository->getById($customerId);
        }
        return null;
    }

    /**
     * Set customer as logged in
     */
    public function setCustomerAsLoggedIn(CustomerInterface $customer): void
    {
        $this->setCustomerId($customer->getId());
        $this->setCustomerGroupId($customer->getGroupId());
        // Also dispatches 'customer_login' event
    }

    /**
     * Logout customer and destroy session
     */
    public function logout(): void
    {
        // Dispatches 'customer_logout' event
        $this->setCustomerId(null);
        $this->setCustomerGroupId(null);
        // Destroys session data
    }
}
```

### Key Customer Session Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `isLoggedIn()` | `bool` | `true` if customer ID is set in session |
| `getCustomerId()` | `int\|null` | The customer's entity ID |
| `getCustomer()` | `CustomerInterface\|null` | Full customer data object (lazy-loaded) |
| `setCustomerAsLoggedIn(CustomerInterface $customer)` | `void` | Marks customer as authenticated |
| `logout()` | `void` | Clears customer from session |
| `getCustomerGroupId()` | `int` | Customer's group ID (for pricing rules) |
| `setCustomerAsLoggedIn($customer)` | `void` | After login, also regenerates session ID |

### Customer Session Lifecycle

```
1. Customer visits storefront
   └─► SessionManager::start() creates session
   └─► Session namespace 'frontend' established

2. Customer logs in (POST to /customer/account/loginPost/)
   └─► \Magento\Customer\Controller\Account\LoginPost::execute()
   └─► $this->session->setCustomerAsLoggedIn($customer)
   └─► Dispatches 'customer_login' event
   └─► Session ID may be regenerated (for session fixation prevention)

3. Customer browses (session persists)
   └─► CustomerSession::isLoggedIn() returns true
   └─► Cart, wishlist, compare list use customer ID from session

4. Customer logs out (POST to /customer/account/logout/)
   └─► CustomerSession::logout()
   └─► Dispatches 'customer_logout' event
   └─► Session data cleared (except flash messages)
```

### Regenerating Session ID on Login

Magento regenerates the session ID when a customer logs in to prevent session fixation attacks (where an attacker sets a user's session ID before authentication):

```php
<?php
// In Magento\Customer\Controller\Account\LoginPost
public function execute(): ResultInterface
{
    // ... authentication logic ...

    if ($customer && $this->getAuthneticator()->authenticate($customer)) {
        // Regenerate session ID to prevent session fixation
        $this->sessionRegenerator->regenerate();
        $this->session->setCustomerAsLoggedIn($customer);
    }

    // ...
}
```

The session regenerator is `\Magento\Framework\Session\SessionManagerInterface` which calls `regenerateId()` on the underlying Zend session manager.

---

## 4. Admin User Sessions

### `Magento\Backend\Model\Auth\Session`

The `AuthSession` manages the admin user's authenticated state. It extends `Magento\Framework\Session\Generic` and adds admin-specific concerns: ACL cache, user object, and UI component data:

```php
<?php
// vendor/magento/module-backend/Model/Auth/Session.php
declare(strict_types=1);

namespace Magento\Backend\Model\Auth;

use Magento\Backend\Model\Session as BackendSession;
use Magento\Framework\Acl;
use Magento\Framework\Event\ManagerInterface;
use Magento\User\Model\User;

class AuthSession extends BackendSession
{
    /**
     * ACL role passed to JS my account menu
     */
    public const ADMIN_RESOURCE = 'Magento_Backend::admin';

    /**
     * @var User|null
     */
    private $user;

    /**
     * @var Acl|null
     */
    private $acl;

    /**
     * Get the logged-in admin user
     */
    public function getUser(): ?User
    {
        if ($this->user === null) {
            $userId = $this->getUserId();
            if ($userId) {
                $this->user = $this->userFactory->create()->load($userId);
            }
        }
        return $this->user;
    }

    /**
     * Check if admin user is logged in
     */
    public function isLoggedIn(): bool
    {
        return $this->getUserId() !== null;
    }

    /**
     * Get ACL object for permission checks
     */
    public function getAcl(): ?Acl
    {
        return $this->acl;
    }

    /**
     * Set ACL into session (typically done after login)
     */
    public function setAcl(Acl $acl): void
    {
        $this->acl = $acl;
    }

    /**
     * Get current user ID
     */
    public function getUserId(): ?int
    {
        return $this->getData('user_id');
    }

    /**
     * Set user as logged in
     */
    public function setUser(User $user): void
    {
        $this->setUserId($user->getId());
        $this->user = $user;
    }
}
```

### Key AuthSession Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `isLoggedIn()` | `bool` | `true` if user ID is set |
| `getUser()` | `User\|null` | The admin user model |
| `getUserId()` | `int\|null` | Admin user entity ID |
| `getAcl()` | `Acl\|null` | ACL object for permission checks |
| `setAcl(Acl $acl)` | `void` | Sets the ACL cache |
| `setUser(User $user)` | `void` | Marks user as authenticated |

### Admin Session Timeout Configuration

Admin sessions are subject to `admin/security/session_lifetime` (in seconds). The default is 900 seconds (15 minutes) of inactivity:

```sql
SELECT * FROM core_config_data WHERE path LIKE '%session_lifetime%';
-- Results:
-- default/0/admin/security/session_lifetime = 900
-- default/0/admin/security/admin_account_sharing = 1
```

When `admin_account_sharing = 0`, each admin user has a separate session scope preventing concurrent logins by the same user. When `admin_account_sharing = 1` (default), the same user can have multiple active sessions.

### Admin Session Locking

Magento 2.4.4+ introduced session locking for admin accounts to prevent concurrent session exploitation. When enabled (`admin/security/session_lifetime` > 0), the system:

1. Acquires a lock on the session file/record when the admin authenticates
2. Releases the lock on logout or session expiry
3. Prevents the same credentials from establishing multiple simultaneous sessions

Configuration path: `admin/security/session_lifetime` (integer, seconds)

---

## 5. `web/session/` Configuration

The `web/session/` configuration path in `core_config_data` controls how PHP and Magento handle session cookies. These are fundamental settings that affect every user on the storefront.

### Cookie Lifetime (`cookie_lifetime`)

**Path**: `web/session/cookie_lifetime`  
**Type**: Integer (seconds)  
**Default**: `0` (session cookie — expires when browser closes)

Setting `cookie_lifetime` to `3600` means the session cookie expires after 1 hour regardless of browser close. Setting `0` makes it a session cookie (deleted on browser close).

```php
<?php
// Programmatic access to cookie lifetime
use Magento\Framework\App\Config\ScopeConfigInterface;

$scopeConfig = $objectManager->get(ScopeConfigInterface::class);
$cookieLifetime = $scopeConfig->getValue(
    'web/session/cookie_lifetime',
    \Magento\Store\Model\ScopeInterface::SCOPE_STORE
);
// Returns int (seconds) or 0 if not set
```

### Cookie Path (`cookie_path`)

**Path**: `web/session/cookie_path`  
**Type**: String  
**Default**: `/` (entire site)

Restricts cookie transmission to a specific path on the domain. For a site installed in a subdirectory (`https://example.com/shop/`), setting `cookie_path = /shop` prevents the cookie from being sent to other paths.

```php
// In etc/env.php
'cookie' => [
    'path' => '/',
],
```

### Cookie Domain (`cookie_domain`)

**Path**: `web/session/cookie_domain`  
**Type**: String  
**Default**: empty (current hostname)

```php
'cookie' => [
    'domain' => '.example.com',  // Leading dot for subdomain access
],
```

### Cookie Secure (`cookie_secure`)

**Path**: `web/session/cookie_secure`  
**Type**: Boolean (0 or 1)  
**Default**: `0`

When enabled (`cookie_secure = 1`), the session cookie is only sent over HTTPS connections. On sites running HTTPS exclusively, enable this. On mixed HTTP/HTTPS sites (rare), keep it disabled or you risk losing sessions when a customer transitions from HTTPS to HTTP.

```php
'cookie' => [
    'secure' => true,  // Only send over HTTPS
],
```

### Cookie HttpOnly (`cookie_httponly`)

**Path**: `web/session/cookie_httponly`  
**Type**: Boolean (0 or 1)  
**Default**: `1`

When enabled, the `HttpOnly` flag is set on the session cookie, preventing JavaScript access via `document.cookie`. This mitigates XSS-based session hijacking. **Always keep this enabled unless you have a specific JavaScript reason to read the session cookie.**

### Cookie SameSite (`cookie_samesite`)

**Path**: `web/session/cookie_samesite`  
**Type**: String (`Strict`, `Lax`, `None`)  
**Default**: `Lax`

SameSite controls when the browser includes the cookie in cross-site requests:

| Value | Behavior |
|-------|----------|
| `Strict` | Cookie only sent in first-party (same-site) requests |
| `Lax` | Cookie sent in first-party requests and same-site top-level navigations |
| `None` | Cookie sent in all requests (requires `cookie_secure = true`) |

In Magento 2.4.8, the default `Lax` provides a good balance. For cross-origin AJAX calls, you may need `None` but this requires HTTPS.

### Configuration Precedence

Cookie settings are read from multiple sources with this precedence (highest to lowest):

1. `$_COOKIE` params passed to `SessionManager` constructor
2. `etc/env.php` `'cookie'` section
3. `core_config_data` `web/session/*` values
4. PHP `ini_get('session.*')` defaults

---

## 6. Cookie Model

### `Magento\Framework\Cookie\CookieManager`

`CookieManager` is Magento's abstraction over the `setcookie()` / `setrawcookie()` functions. It provides a consistent interface with automatic handling of `SameSite`, `Secure` flags, and domain configuration:

```php
<?php
// vendor/magento/framework/Cookie/CookieManager.php
declare(strict_types=1);

namespace Magento\Framework\Cookie;

use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Framework\Stdlib\Cookie\PublicCookieMetadata;
use Magento\Framework\Stdlib\Cookie\CookieMetadata;
use Magento\Framework\Stdlib\Cookie\FailureDetection;

class CookieManager
{
    public const COOKIE_EXPIRE_TIME_OFFSET = 86400; // 1 day in seconds

    /**
     * @param ScopeConfigInterface $scopeConfig
     * @param PublicCookieMetadataFactory $publicCookieMetadataFactory
     * @param CookieMetadataFactory $cookieMetadataFactory
     * @param FailureDetection $failureDetection
     */
    public function __construct(
        ScopeConfigInterface $scopeConfig,
        PublicCookieMetadataFactory $publicCookieMetadataFactory,
        CookieMetadataFactory $cookieMetadataFactory,
        FailureDetection $failureDetection
    ) {
        $this->scopeConfig = $scopeConfig;
        $this->publicCookieMetadataFactory = $publicCookieMetadataFactory;
        $this->cookieMetadataFactory = $cookieMetadataFactory;
        $this->failureDetection = $failureDetection;
    }

    /**
     * Get a cookie value by name
     */
    public function getCookie(string $cookieName, ?string $default = null): ?string
    {
        return isset($_COOKIE[$cookieName]) ? $_COOKIE[$cookieName] : $default;
    }

    /**
     * Set a cookie with explicit metadata
     *
     * @param string $name
     * @param string $value
     * @param PublicCookieMetadata|null $metadata
     */
    public function setCookie(string $name, string $value, ?PublicCookieMetadata $metadata = null): void
    {
        $expires = $metadata && $metadata->getExpirationTime()
            ? $metadata->getExpirationTime()
            : 0;

        $expiresPath = $metadata && $metadata->getPath()
            ? $metadata->getPath()
            : '/';

        $expiresDomain = $metadata && $metadata->getDomain()
            ? $metadata->getDomain()
            : null;

        $expiresSecure = $metadata && $metadata->getSecure()
            ? $metadata->getSecure()
            : (bool)$this->scopeConfig->isSetFlag(
                \Magento\Framework\Session\Config::XML_PATH_COOKIE_SECURE,
                \Magento\Store\Model\ScopeInterface::SCOPE_STORE
            );

        $httpOnly = $metadata && $metadata->getHttpOnly()
            ? $metadata->getHttpOnly()
            : (bool)$this->scopeConfig->isSetFlag(
                \Magento\Framework\Session\Config::XML_PATH_COOKIE_HTTPONLY,
                \Magento\Store\Model\ScopeInterface::SCOPE_STORE
            );

        $sameSite = $metadata && $metadata->getSameSite()
            ? $metadata->getSameSite()
            : $this->scopeConfig->getValue(
                \Magento\Framework\Session\Config::XML_PATH_COOKIE_SAMESITE,
                \Magento\Store\Model\ScopeInterface::SCOPE_STORE
            );

        $this->setCookieInternal(
            $name,
            $value,
            $expires,
            $expiresPath,
            $expiresDomain,
            $expiresSecure,
            $httpOnly,
            $sameSite
        );
    }

    /**
     * Delete a cookie
     */
    public function deleteCookie(string $cookieName, ?CookieMetadata $metadata = null): void
    {
        $path = $metadata && $metadata->getPath() ? $metadata->getPath() : '/';
        $domain = $metadata && $metadata->getDomain() ? $metadata->getDomain() : null;
        $secure = $metadata && $metadata->getSecure()
            ? $metadata->getSecure()
            : false;
        $httpOnly = $metadata && $metadata->getHttpOnly()
            ? $metadata->getHttpOnly()
            : true;

        $this->setCookieInternal($cookieName, '', -1, $path, $domain, $secure, $httpOnly, null);
    }

    /**
     * Check if cookie should be sent over secure channel only
     */
    public function isCookieSecure(): bool
    {
        return (bool)$this->scopeConfig->isSetFlag(
            \Magento\Framework\Session\Config::XML_PATH_COOKIE_SECURE,
            \Magento\Store\Model\ScopeInterface::SCOPE_STORE
        );
    }
}
```

### CookieManager vs Native `setcookie()`

**Use `CookieManager` when:**
- Setting Magento-managed cookies that should respect global cookie settings
- Setting cookies that need `SameSite`, `Secure`, or `HttpOnly` flags
- You need cookie metadata (expiration, path, domain)
- The cookie is part of the Magento application flow

**Use native `setcookie()` when:**
- You need a raw cookie without Magento's metadata processing
- Setting cookies from a standalone script or CLI context
- You need behavior that Magento's cookie layer doesn't support

### PublicCookieMetadata for Setting Cookies

When setting cookies via `CookieManager`, use `PublicCookieMetadata` to control cookie behavior:

```php
<?php
use Magento\Framework\Cookie\CookieManager;
use Magento\Framework\Stdlib\Cookie\PublicCookieMetadata;
use Magento\Framework\Stdlib\Cookie\PublicCookieMetadataFactory;

class MyService
{
    public function __construct(
        private readonly CookieManager $cookieManager,
        private readonly PublicCookieMetadataFactory $cookieMetadataFactory,
    ) {}

    public function setRememberMeCookie(string $value): void
    {
        /** @var PublicCookieMetadata $metadata */
        $metadata = $this->cookieMetadataFactory->create();

        $metadata->setPath('/')
            ->setDuration(86400 * 30)  // 30 days
            ->setSecure(true)
            ->setHttpOnly(true)
            ->setSameSite('Lax');

        $this->cookieManager->setCookie('remember_me', $value, $metadata);
    }

    public function deleteRememberMeCookie(): void
    {
        $this->cookieManager->deleteCookie('remember_me');
    }
}
```

---

## 7. Form Key Cookies

### `Magento\Framework\Data\Form\FormKey`

Form keys are Magento's primary CSRF (Cross-Site Request Forgery) protection mechanism for adminhtml and public forms. Every form that submits data to a controller must include a form key token:

```php
<?php
// vendor/magento/framework/Data/Form/FormKey.php
declare(strict_types=1);

namespace Magento\Framework\Data;

use Magento\Framework\Session\SessionManagerInterface;

class FormKey
{
    public const FORM_KEY = 'form_key';

    /**
     * @var SessionManagerInterface
     */
    private $sessionManager;

    /**
     * @param SessionManagerInterface $sessionManager
     */
    public function __construct(SessionManagerInterface $sessionManager)
    {
        $this->sessionManager = $sessionManager;
    }

    /**
     * Get the current form key
     */
    public function getFormKey(): string
    {
        $formKey = $this->sessionManager->getData(self::FORM_KEY);
        if (!$formKey) {
            $formKey = $this->generate();
            $this->sessionManager->setData(self::FORM_KEY, $formKey);
        }
        return $formKey;
    }

    /**
     * Generate a new form key
     */
    public function generate(): string
    {
        return bin2hex(random_bytes(16));
    }
}
```

### How Form Keys Work

1. When a page loads, `FormKey::getFormKey()` is called
2. If no form key exists in the session, one is generated and stored
3. The form key is rendered in every form as a hidden input: `<input name="form_key" type="hidden" value="..." />`
4. On form submission, the controller validates the form key against the session value
5. If they don't match, the request is rejected with a 400 or 403 response

### Form Key Validation in Controllers

All Magento controllers that extend `Magento\Framework\App\Action\Action` automatically validate the form key on POST requests:

```php
<?php
// In Magento\Framework\App\Action\Action::execute()
public function execute()
{
    if ($this->getRequest()->isPost()) {
        if (!$this->_formKeyValidator->validate($this->getRequest())) {
            $resultRedirect->setUrl($this->_redirect->getRefererUrl());
            return $resultRedirect;
        }
    }
    // ... continue with action
}
```

The `_formKeyValidator` is `Magento\Framework\Data\Form\FormKey\Validator`.

### AJAX Form Submissions

When submitting forms via AJAX (JavaScript `fetch` or `$.ajax`), you must include the form key in the request:

```javascript
// JavaScript ( RequireJS module )
define(['jquery', 'mage/form-key'], function ($) {
    'use strict';

    return function (formElement) {
        var form = $(formElement);
        var action = form.attr('action');
        var formData = new FormData(form[0]);

        // Ensure form key is included
        if (!formData.has('form_key')) {
            formData.append('form_key', window.FORM_KEY);
        }

        return fetch(action, {
            method: 'POST',
            body: formData,
            headers: {
                'X-Requested-With': 'XMLHttpRequest'
            }
        });
    };
});
```

The global `window.FORM_KEY` is populated by `Magento_Customer/js/view/customer` or `Magento_Ui/js/core/app` on every page load.

### Adminhtml Form Key Configuration

For adminhtml forms specifically, the form key cookie is also checked at the layout level. In `Magento_Backend` layout XML:

```xml
<formKey>always</formKey>
```

This ensures admin forms always require valid form keys. Attempting to submit a form without a valid `form_key` cookie and POST parameter results in a `Security control「Exception」` or a 400 Bad Request.

---

## 8. Persistent Shopping Cart

### Persistent Cart Configuration

Magento's persistent cart feature remembers a guest or logged-out customer's cart contents between sessions. It is controlled by these configuration paths in `core_config_data`:

| Path | Default | Description |
|------|---------|-------------|
| `persistent/options/enabled` | `0` | Enable persistent cart |
| `persistent/options/lifetime` | `86400 * 365` | Cookie lifetime in seconds (1 year) |
| `persistent/options/cleanup` | `86400 * 90` | Guest cart lifetime in seconds (90 days) |
| `persistent/options/shopping_cart` | `1` | Remember cart contents |
| `persistent/options/recently_ordered` | `1` | Remember recently ordered |

### How Persistent Sessions Differ from Regular Sessions

| Aspect | Regular Session | Persistent Session |
|--------|----------------|--------------------|
| **Storage** | `var/sessions/` or DB or Redis | `persistent_session` table in DB |
| **Lifetime** | Browser close or `cookie_lifetime` | `persistent/options/lifetime` (1 year default) |
| **Customer** | Logged-in only | Logged-in and guest |
| **Cleared on** | Logout, browser close | Explicit logout or expiration |
| **Cart merging** | Automatic on login | Merged with persistent cart |

### The `persistent/session` Cookie

When persistent cart is enabled, Magento sets a `persistent_session` cookie alongside the regular `PHPSESSID` or `frontend` cookie:

```php
<?php
// In Magento\Persistent\Model\Session
const COOKIE_NAME = 'persistent_session';
const COOKIE_lifetime = 31536000;  // 1 year

// Cookie value is the persistent session ID (not the customer ID)
// Links to persistent_session table in database
```

### Persistent Session Table

```sql
SELECT * FROM persistent_session;
-- Columns:
--   id (PK, int)
--   website_id (int)
--   customer_id (int, FK to customer_entity)
--   session_key (varchar 255, unique token)
--   created_at (datetime)
--   updated_at (datetime)
--   keep_alive (int, 1 if persistent cart is active)
```

### Merging Cart on Login

When a customer with a persistent cart logs in:

```php
<?php
// In Magento\Persistent\Helper\Cart
public function cartExists()
{
    // Check if persistent session has cart data
    // If so, merge with current customer quote
}
```

The `Magento\Persistent\Observer\SynchronizeGuestCartObserver` handles the cart merge:

```xml
<!-- etc/events.xml in Magento_Persistent -->
<event name="customer_login">
    <observer name="persistent_observer_synchronize_guest_cart"
              instance="Magento\Persistent\Observer\SynchronizeGuestCartObserver"/>
</event>
```

---

## 9. Session Storage Backends

### File-Based Sessions (Default)

The default save handler stores sessions as files in `var/sessions/`. Configure in `etc/env.php`:

```php
<?php
// app/etc/env.php
'session' => [
    'save' => 'files',
],
```

PHP's `session.save_handler` must be `files` (the default). Sessions are stored as files named `sess_<session_id>` in `var/sessions/`.

### Database Sessions (`db`)

The `Magento_Newsletter` module (and optionally others) can use the database as a session backend:

```php
<?php
// app/etc/env.php
'session' => [
    'save' => 'db',
],
```

This requires the `session` table to exist (created by `Magento\Session\Setup\SchemaProvider`):

```sql
DESCRIBE session;
-- Columns:
--   session_id (varchar 128, PK)
--   session_data (longtext, serialized session data)
--   session_expires (int unsigned, expiration timestamp)
--   connection_name (varchar 64, optional, for multi-database setups)
```

### Redis Sessions (Recommended for Production)

Redis is the recommended session backend for production environments because it:
- Supports distributed sessions across multiple application servers
- Has built-in key expiration (no `gc` needed)
- Provides sub-millisecond read/write performance
- Supports session clustering and replication

```php
<?php
// app/etc/env.php — minimal Redis configuration
'session' => [
    'save' => 'redis',
    'redis' => [
        'host' => '127.0.0.1',
        'port' => '6379',
        'password' => '',
        'timeout' => '2.5',
        'persistent_identifier' => '',
        'database' => '0',
        'priority' => '',
        'compression_threshold' => '2048',
        'compression_library' => 'gzip',
        'log_level' => '1',
        'max_concurrency' => '6',
        'break_after_frontend' => '5',
        'break_after_adminhtml' => '10',
        'first_lifetime' => '600',
        'bot_first_lifetime' => '60',
        'bot_lifetime' => '7200',
        'disable_locking' => '0',
        'min_lifetime' => '60',
        'max_lifetime' => '86400',
    ],
],
```

### Memcache Sessions

```php
<?php
// app/etc/env.php
'session' => [
    'save' => 'memcache',
    'memcache' => [
        'host' => '127.0.0.1',
        'port' => '11211',
        'timeout' => '5',
        'persistent_identifier' => '',
        'max_overflow' => '0',
        'min_lifetime' => '60',
        'max_lifetime' => '86400',
    ],
],
```

### Session Save Handler Configuration in `di.xml`

The session save handler is wired via a preference:

```xml
<!-- vendor/magento/framework/Session/etc/di.xml -->
<type name="Magento\Framework\Session\SaveHandlerInterface">
    <arguments>
        <argument name="saveHandlerType" xsi:type="string">files</argument>
        <argument name="saveHandlerData" xsi:type="array"/>
    </arguments>
</type>
```

Overriding this preference in your module's `di.xml` changes the session backend globally.

---

## 10. `Magento\Framework\Session\SaveHandler`

### The SaveHandler Interface

Magento's `SaveHandlerInterface` defines the contract for session storage:

```php
<?php
// vendor/magento/framework/Session/SaveHandlerInterface.php
declare(strict_types=1);

namespace Magento\Framework\Session;

/**
 * Session save handler interface
 * @api
 */
interface SaveHandlerInterface
{
    /**
     * Read session data
     *
     * @param string $sessionId
     * @return string
     */
    public function read(string $sessionId): string;

    /**
     * Write session data
     *
     * @param string $sessionId
     * @param string $sessionData
     * @return bool
     */
    public function write(string $sessionId, string $sessionData): bool;

    /**
     * Destroy session
     *
     * @param string $sessionId
     * @return bool
     */
    public function destroy(string $sessionId): bool;

    /**
     * Garbage collection
     *
     * @param int $lifetime
     * @return bool
     */
    public function gc(int $lifetime): bool;
}
```

### `Magento\Framework\Session\SaveHandler\File`

The default file-based save handler:

```php
<?php
// vendor/magento/framework/Session/SaveHandler/File.php
declare(strict_types=1);

namespace Magento\Framework\Session\SaveHandler;

use Magento\Framework\App\Filesystem\DirectoryList;
use Magento\Framework\Filesystem;

class File implements \Magento\Framework\Session\SaveHandlerInterface
{
    /**
     * @var Filesystem
     */
    private $filesystem;

    /**
     * @param Filesystem $filesystem
     */
    public function __construct(Filesystem $filesystem)
    {
        $this->filesystem = $filesystem;
    }

    /**
     * Read session data from file
     */
    public function read(string $sessionId): string
    {
        $sessionFile = $this->getSessionFilePath($sessionId);
        if (file_exists($sessionFile)) {
            $data = file_get_contents($sessionFile);
            return $data !== false ? $data : '';
        }
        return '';
    }

    /**
     * Write session data to file
     */
    public function write(string $sessionId, string $sessionData): bool
    {
        $sessionFile = $this->getSessionFilePath($sessionId);
        $result = file_put_contents($sessionFile, $sessionData);
        return $result !== false;
    }

    /**
     * Destroy session by deleting file
     */
    public function destroy(string $sessionId): bool
    {
        $sessionFile = $this->getSessionFilePath($sessionId);
        if (file_exists($sessionFile)) {
            return unlink($sessionFile);
        }
        return true;
    }

    /**
     * Garbage collection — remove expired session files
     */
    public function gc(int $lifetime): bool
    {
        $sessionsDir = $this->filesystem->getDirectoryWrite(DirectoryList::SESSION)
            ->getAbsolutePath();

        foreach (glob($sessionsDir . 'sess_*') as $file) {
            if (filemtime($file) < time() - $lifetime) {
                @unlink($file);
            }
        }
        return true;
    }

    private function getSessionFilePath(string $sessionId): string
    {
        return $this->filesystem->getDirectoryWrite(DirectoryList::SESSION)
            ->getAbsolutePath() . 'sess_' . $sessionId;
    }
}
```

### When to Implement a Custom Save Handler

Implement a custom `SaveHandlerInterface` when:
- Using a session clustering technology not natively supported (e.g., DynamoDB, Google Cloud Firestore)
- Building a read replicas setup where session reads go to replicas and writes to primary
- Implementing session encryption at rest

```php
<?php
// app/code/Vendor/Module/Session/SaveHandler/DynamoDb.php
declare(strict_types=1);

namespace Vendor\Module\Session\SaveHandler;

use Magento\Framework\Session\SaveHandlerInterface;

class DynamoDb implements SaveHandlerInterface
{
    private $dynamoDbClient;

    public function __construct(\Aws\DynamoDb\DynamoDbClient $dynamoDbClient)
    {
        $this->dynamoDbClient = $dynamoDbClient;
    }

    public function read(string $sessionId): string
    {
        $result = $this->dynamoDbClient->getItem([
            'TableName' => 'sessions',
            'Key' => ['session_id' => ['S' => $sessionId]],
        ]);

        return $result['Item']['session_data']['S'] ?? '';
    }

    public function write(string $sessionId, string $sessionData): bool
    {
        $this->dynamoDbClient->putItem([
            'TableName' => 'sessions',
            'Item' => [
                'session_id' => ['S' => $sessionId],
                'session_data' => ['S' => $sessionData],
                'expires_at' => ['N' => (string)(time() + 86400)],
            ],
        ]);
        return true;
    }

    public function destroy(string $sessionId): bool
    {
        $this->dynamoDbClient->deleteItem([
            'TableName' => 'sessions',
            'Key' => ['session_id' => ['S' => $sessionId]],
        ]);
        return true;
    }

    public function gc(int $lifetime): bool
    {
        // DynamoDB TTL handles expiration automatically
        return true;
    }
}
```

Wire it via `di.xml`:

```xml
<!-- app/code/Vendor/Module/etc/di.xml -->
<type name="Magento\Framework\Session\SaveHandlerInterface">
    <arguments>
        <argument name="saveHandlerType" xsi:type="string">Vendor\Module\Session\SaveHandler\DynamoDb</argument>
    </arguments>
</type>
```

---

## 11. Redis Session Configuration

### Full Redis Session Configuration

Redis provides the most scalable session backend for production Magento deployments. Here is a complete configuration:

```php
<?php
// app/etc/env.php
'session' => [
    'save' => 'redis',
    'redis' => [
        // Connection
        'host' => 'redis.internal.example.com',
        'port' => '6379',
        'password' => 'redis_secret_password',  // Set to '' if no password
        'timeout' => '2.5',                       // Connection timeout in seconds
        'persistent_identifier' => '',            // Persistent connection ID (leave empty for default)
        'database' => '0',                         // Redis database number (0-15)

        // Compression (reduces network traffic)
        'compression_threshold' => '2048',        // Disable compression for sessions < 2KB
        'compression_library' => 'lzf',           // Options: lzf, gzip, snappy (requires extension)

        // Concurrency
        'max_concurrency' => '6',                 // Max threads writing to session at once
        'break_after_frontend' => '5',            // Break lock after 5 seconds for frontend
        'break_after_adminhtml' => '10',          // Break lock after 10 seconds for admin

        // Lifetime (when to regenerate sessions)
        'first_lifetime' => '600',               // First expiry check at 10 minutes
        'bot_first_lifetime' => '60',             // Bot gets shorter lifetime (60 seconds)
        'bot_lifetime' => '7200',                 // Bot session lifetime (2 hours)
        'disable_locking' => '0',                 // Set to '1' to disable session locking

        // Expiry
        'min_lifetime' => '60',                   // Minimum session lifetime
        'max_lifetime' => '86400',                // Maximum session lifetime (1 day)

        // Log
        'log_level' => '1',                       // 0 = emergency, 1 = error, 2 = warn, 3 = info
    ],
],
```

### Session Locking in Redis

Redis-based sessions use a locking mechanism to prevent concurrent writes to the same session. When `disable_locking = 0`:

1. When a request starts a session, it acquires a lock with a TTL
2. Other requests for the same session ID wait or get `break_after_*` timeout
3. The lock is released when `writeClose()` is called

For AJAX-heavy pages or long-polling, locking can cause delays. Set `disable_locking = 1` if your storefront uses AJAX heavily and sessions aren't critical for data consistency.

### `log_based_lifetime` for Redis

In Magento 2.4.8 with Redis session handler, there's an additional `log_based_lifetime` parameter that bases session expiry on the last write time rather than creation time:

```php
'session' => [
    'redis' => [
        // ... other config ...
        'log_based_lifetime' => '86400',  // Sessions expire 24 hours after last write
    ],
],
```

---

## 12. Session Validation

### `Magento\Framework\Session\Validator`

Magento's session validation system is designed to detect session hijacking and fixation attempts. The `Validator` class checks multiple criteria before accepting a session as valid:

```php
<?php
// vendor/magento/framework/Session/Validator.php
declare(strict_types=1);

namespace Magento\Framework\Session;

use Magento\Framework\App\RequestInterface;

class Validator implements ValidatorInterface
{
    /**
     * @var RequestInterface
     */
    private $request;

    /**
     * @var \Magento\Framework\Stdlib\DateTime\TimezoneInterface
     */
    private $timezone;

    /**
     * @var int[]
     */
    private $validators;

    /**
     * @param RequestInterface $request
     * @param \Magento\Framework\Stdlib\DateTime\TimezoneInterface $timezone
     */
    public function __construct(
        RequestInterface $request,
        \Magento\Framework\Stdlib\DateTime\TimezoneInterface $timezone
    ) {
        $this->request = $request;
        $this->timezone = $timezone;
        $this->validators = [
            'http_user_agent' => true,
            'remote_addr' => true,
            'session_lifetime' => true,
        ];
    }

    /**
     * Validate session data
     */
    public function validate(array $data): bool
    {
        // Check session lifetime
        if (isset($data['session_expires'])) {
            $maxLifetime = (int) $this->timezone->scopeTimezone()->getOffset() + (int) $this->getSessionLifetime();
            if ($data['session_expires'] < time() - $maxLifetime) {
                return false;  // Session has expired
            }
        }

        // Check HTTP User-Agent (if stored and changed, session is invalid)
        if (isset($data['http_user_agent'])) {
            $validatorHttpUserAgent = $this->request->getServer('HTTP_USER_AGENT');
            if ($data['http_user_agent'] !== $validatorHttpUserAgent) {
                return false;  // User agent changed — possible session hijacking
            }
        }

        // Check Remote Addr (if stored and changed significantly, session may be invalid)
        if (isset($data['remote_addr'])) {
            $validatorRemoteAddr = $this->request->getServer('REMOTE_ADDR');
            // Some customers appear behind proxies with varying IPs
            // Only invalidate if the first 2 octets differ significantly
            if (!$this->isValidRemoteAddr($data['remote_addr'], $validatorRemoteAddr)) {
                return false;
            }
        }

        return true;
    }

    private function isValidRemoteAddr(string $stored, string $current): bool
    {
        // Simplified: check if IPs match or are in same subnet
        // Full implementation handles proxy headers and subnet matching
        return $stored === $current;
    }

    private function getSessionLifetime(): int
    {
        // Returns session lifetime from config (default 86400 for admin, 3600 for frontend)
        return (int) $this->timezone->scopeConfig->getValue(
            'admin/security/session_lifetime',
            \Magento\Store\Model\ScopeInterface::SCOPE_STORE
        ) ?: 86400;
    }
}
```

### Why Sessions Can Be Invalidated Silently

The session validation happens on every request when `SessionManager::start()` is called. If validation fails, Magento **clears the session silently** and starts a new one. This means:

1. A customer browsing with Safari Private Browsing + Chrome side-by-side will have two different user agents
2. A customer switching from WiFi to mobile data may get a different IP
3. A customer whose browser sends a slightly different User-Agent string (due to extensions) will get logged out

This is a trade-off between security (rejecting potential hijacked sessions) and usability.

### `session_lifetime` Checks

`session_lifetime` in `web/session/` controls the cookie/session expiration. But there's also `admin/security/session_lifetime` specifically for admin:

```php
<?php
// If session_expires is set and past, session is invalid
$sessionExpires = $sessionData['session_expires'] ?? null;
if ($sessionExpires !== null && (int)$sessionExpires < time()) {
    // Session expired — regenerate
    $this->regenerateId();
}
```

### HTTP User-Agent Validation

When a session is created, the User-Agent is stored:

```php
<?php
// In SessionManager::start() or child classes
$this->setData('http_user_agent', $request->getServer('HTTP_USER_AGENT'));
$this->setData('remote_addr', $request->getServer('REMOTE_ADDR'));
```

If on a subsequent request the User-Agent string differs (case sensitivity matters!), the session is invalidated. This prevents an attacker from stealing a session ID and using it from a different browser.

### Remote Addr Check

The remote IP address is stored at session start. If it changes significantly (not just port changes, but different IPs in different /16 subnets), the session may be invalidated depending on configuration. This handles customers behind large corporate proxies where requests may come from different IPs in the same range.

---

## 13. Cookie Size Limits

### The 4KB Cookie Limit

Every HTTP cookie has a size limit imposed by web browsers: **approximately 4KB (4096 bytes) for the entire cookie string** (name + value + attributes). When the session data stored in a cookie exceeds this limit, browsers reject the cookie, resulting in session loss.

In Magento, this typically manifests as:
- Customers getting logged out randomly
- Cart contents disappearing
- "400 Bad Request" errors in the browser console
- AJAX requests failing without clear error messages

### Causes of Large Session Data

Magento stores substantial data in the session:

| Session Key | Typical Size | Notes |
|------------|-------------|-------|
| `customer_id` | ~10 bytes | Small |
| `customer_session` | ~1-5 KB | Customer object (addresses, custom attributes) |
| `checkout` | ~2-10 KB | Cart items, shipping/billing addresses |
| `wishlist` | ~1-5 KB | Wishlist items |
| `catalog` | ~1-3 KB | Recently viewed, compared products |
| `mage-form-key` | ~32 bytes | Small |
| **Total** | **5-25 KB** | **Easily exceeds 4KB** |

When session data exceeds 4KB, you need either:
1. Use Redis or DB sessions (session stored server-side, not in cookie)
2. Reduce session data size
3. Implement section-based data loading

### Cookie Fragmentation

When a single cookie exceeds the 4KB limit, it cannot be stored. Browsers do **not** automatically fragment large cookies. You must split data across multiple cookies:

```php
<?php
// Split large session data across multiple cookies
$largeData = serialize($sessionData);
$chunkSize = 3000;  // Leave room for cookie metadata

$chunks = str_split($largeData, $chunkSize);
foreach ($chunks as $index => $chunk) {
    setcookie("session_part_{$index}", $chunk, time() + 3600, '/');
}
```

This approach is rarely used in practice because Redis/DB session backends solve the problem more elegantly.

### Diagnosing Cookie Size Issues

Check the response headers in your browser's Network tab for `Set-Cookie` headers. If you see cookies being set with unusually large values, check what's being stored in session. Use the Data Collector in Admin → System → Support to inspect session contents.

---

## 14. Cookie Consent / GDPR

### The EU Cookie Law (GDPR) in Magento

The EU General Data Protection Regulation (GDPR) requires websites to obtain explicit consent before storing non-essential cookies on a visitor's device. Magento 2.4.8 provides a cookie notification system.

### Cookie Notification Block

The cookie notice is rendered by `Magento\Cookie\Block\Html\Notices`:

```xml
<!-- In Magento_Cookie layout files -->
<referenceContainer name="after.body.start">
    <block class="Magento\Cookie\Block\Html\Notices" name="cookie_notices"
           template="Magento_Cookie::notice.phtml"/>
</referenceContainer>
```

The template `notice.phtml` displays the cookie notice if the customer hasn't accepted it:

```php
<?php
// view/frontend/templates/notice.phtml (simplified)
$noticeBlock = $block->getLayout()->getBlock('cookie_notices');
$noCookie = $noticeBlock && $noticeBlock->displayCookieNotice();
?>

<?php if ($noCookie): ?>
<div class="message global cookie" id="cookie-notice">
    <div class="content">
        <?= $block->escapeHtml(__(
            'We use cookies to make your experience better. ' .
            'To comply with the new EU privacy law, we need your consent to set cookies. '
        )) ?>
        <a href="<?= $block->escapeUrl($block->getUrl('privacy-policy')) ?>">
            <?= $block->escapeHtml(__('Learn more')) ?>
    </a>
    <button type="button" class="action primary accept" title="<?= __('Accept') ?>">
        <span><?= __('I accept cookies') ?></span>
    </button>
</div>
<script type="text/x-magento-init">
    {
        "*": {
            "Magento_Cookie/js/notices": {}
        }
    }
}
</script>
<?php endif; ?>
```

### `web/browser_capabilities/cookies`

Magento checks browser cookie capabilities before enabling session management. This is checked via JavaScript:

```php
<?php
// vendor/magento/module-cookie/Helper/Html/Notices.php
public function isCookieShortcutMode()
{
    $isEnabled = $this->scopeConfig->isSetFlag(
        self::XML_PATH_COOKIE_NOTICE,
        \Magento\Store\Model\ScopeInterface::SCOPE_STORE
    );

    if (!$isEnabled) {
        return true;  // Cookie notice disabled, assume cookies work
    }

    return $this->isCookieForCurrentDomain() && $this->isUserAllowedProduct();
}
```

### JavaScript Cookie Consent Pattern

When a customer clicks "Accept", Magento sets the `cookie_notice_disallow` cookie to `0` (or removes it):

```javascript
// In Magento_Cookie/js/notices.js
define(['jquery', 'mage/cookies'], function ($) {
    'use strict';

    $.widget('mage.cookieNotices', {
        options: {
            cookieName: 'cookie_notice_disallow',
            acceptButtonClass: 'action primary accept',
        },

        _create: function () {
            this.element.on('click', '.' + this.options.acceptButtonClass, $.proxy(this.accept, this));
        },

        accept: function () {
            // Remove the restriction cookie, allow analytics/tracking
            $.mage.cookies.set(this.options.cookieName, '0', {
                expires: new Date(Date.now() + (365 * 24 * 60 * 60 * 1000)),
                path: '/',
                domain: '',
                secure: false,
                sameSite: 'Lax'
            });
            this.element.fadeOut();
        }
    });
});
```

### Required Cookies vs Optional Cookies

| Cookie | Type | Can Be Blocked by Cookie Notice |
|--------|------|-------------------------------|
| `PHPSESSID` / `frontend` | Essential | **No** — required for site to function |
| `adminhtml` | Essential | **No** — required for admin access |
| `form_key` | Essential | **No** — required for CSRF protection |
| `cookie_notice_disallow` | Essential | **No** — tracks consent status |
| `_ga`, `_gat`, `_gid` (Analytics) | Optional | **Yes** — must wait for consent |
| `_fbp` (Facebook Pixel) | Optional | **Yes** — must wait for consent |
| 3rd-party advertising cookies | Optional | **Yes** — must wait for consent |

---

## 15. Admin Session Locking

### Concurrent Session Prevention

Magento 2.4+ supports admin session locking to prevent the same admin user from having multiple concurrent active sessions. This is controlled by:

```php
<?php
// etc/env.php
'session' => [
    'admin_security' => [
        'session_lifetime' => 900,  // 15 minutes
    ],
],
```

Or via `core_config_data`:

```sql
INSERT INTO core_config_data (scope, scope_id, path, value)
VALUES ('default', 0, 'admin/security/session_lifetime', 900);
```

### `admin/security/admin_account_sharing`

When set to `0`, concurrent sessions for the same admin user are prevented. When `1` (default), the same user can have multiple sessions:

```sql
SELECT * FROM core_config_data WHERE path = 'admin/security/admin_account_sharing';
-- default/0/admin/security/admin_account_sharing = 1
```

### Session Locking Implementation

The locking mechanism uses file locks (on `var/lock/` directory) or Redis locks (when using Redis sessions):

```php
<?php
// vendor/magento/framework/Lock/LockManagerInterface.php
interface LockManagerInterface
{
    /**
     * Acquire lock
     * @param string $lockName
     * @param int $timeout
     * @return bool
     */
    public function lock(string $lockName, int $timeout = 0): bool;

    /**
     * Release lock
     * @param string $lockName
     * @return bool
     */
    public function unlock(string $lockName): bool;

    /**
     * Check if lock exists
     * @param string $lockName
     * @return bool
     */
    public function isLocked(string $lockName): bool;
}
```

The backend auth session uses `Magento\Framework\Lock\LockManager` to implement the locking:

```php
<?php
// In Magento\Backend\Model\Auth\Session::login()
public function login($username, $password)
{
    // ... authentication ...

    // Acquire session lock to prevent concurrent sessions
    $lockName = 'admin_' . $user->getId();
    $this->lockManager->lock($lockName, 30);

    $this->setUser($user);
    // ...
}
```

### Lock Timeout and `break_after_adminhtml`

When using Redis sessions, `break_after_adminhtml` specifies how long to wait for a lock before giving up:

```php
'session' => [
    'redis' => [
        'break_after_adminhtml' => '10',  // Wait 10 seconds, then proceed without lock
    ],
],
```

---

## 16. Custom Session Namespaces

### `Magento\Framework\Session\SessionManager` with Custom Namespace

Every Magento session has a "namespace" — a key in the PHP `$_SESSION` array that isolates data. The default namespaces are:

- `adminhtml` — Admin panel sessions
- `frontend` — Storefront sessions
- `mage-generic` — Generic Magento data (form keys, flash messages)

To create a custom namespace for your module:

```php
<?php
// app/code/Vendor/Module/Model/Session/CustomSession.php
declare(strict_types=1);

namespace Vendor\Module\Model\Session;

use Magento\Framework\Session\Generic;

class CustomSession extends Generic
{
    /**
     * Custom namespace for this module's session data
     */
    private const CUSTOM_NAMESPACE = 'vendor_module_session';

    /**
     * Set a value in the custom namespace
     */
    public function setCustomData(string $key, $value): void
    {
        $this->setData($key, $value);
    }

    /**
     * Get a value from the custom namespace
     */
    public function getCustomData(string $key)
    {
        return $this->getData($key);
    }

    /**
     * Unset a value from the custom namespace
     */
    public function unsetCustomData(string $key): void
    {
        $this->unsData($key);
    }
}
```

### Register the Custom Session in `di.xml`

```xml
<!-- app/code/Vendor/Module/etc/di.xml -->
<type name="Vendor\Module\Model\Session\CustomSession">
    <arguments>
        <argument name="sessionName" xsi:type="string">vendor_module</argument>
    </arguments>
</type>

<!-- Make it injectable as a dependency -->
<type name="Vendor\Module\Model\SomeService">
    <arguments>
        <argument name="customSession" xsi:type="object">Vendor\Module\Model\Session\CustomSession</argument>
    </arguments>
</type>
```

### `register_shutdownFunction()` Gotchas

PHP sessions require proper shutdown handling. When you use `register_shutdown_function('session_write_close')` or rely on Magento's `SessionManager::start()`, be aware:

**Problem 1: Early Session Close**

```php
<?php
// WRONG: Calling session_write_close() too early
public function earlyClose()
{
    session_write_close();  // Session is now locked for writing
    // Any subsequent $session->setData() calls will not persist!
    sleep(10);  // Some long operation
    // Data set above will be lost because session was already written
}
```

**Problem 2: Missing Shutdown Function Registration**

```php
<?php
// WRONG: Not registering shutdown for session write
public function processAndExit()
{
    // This method does not call start() or register_shutdown_function
    // If the session was started earlier but write hasn't been registered,
    // session data may not persist on exit
    exit();
}
```

**Correct Pattern:**

```php
<?php
// CORRECT: Let Magento's SessionManager handle shutdown
class MyController extends \Magento\Framework\App\Action\Action
{
    public function execute(): void
    {
        // Session is already started by Magento's front controller
        // DO NOT call session_write_close() manually
        // DO NOT call register_shutdown_function() directly

        $this->session->setData('key', 'value');
        // On controller finish, Magento's _finish() will call writeClose()
    }
}
```

**Problem 3: Nested Session Start**

```php
<?php
// WRONG: Starting session in a service that might be called during shutdown
class ExportService
{
    public function export()
    {
        // This might be called during shutdown when session is already closed
        $session = $objectManager->get(\Magento\Framework\Session\Generic::class);
        $session->start();  // May fail if headers already sent
    }
}
```

**Solution: Check Before Session Start**

```php
<?php
// CORRECT: Check if session can be started
if (!headers_sent() && !$sessionManager->isStarted()) {
    $sessionManager->start();
}
```

### Avoiding Session Issues in CLI and cron

In CLI context (bin/magento cron, bin/magento commands), sessions are generally not available or behave differently. Always check:

```php
<?php
if ($this->appState->getAreaCode() === \Magento\Framework\App\Area::AREA_CRONTAB) {
    // Do not use session in crontab — it may not work as expected
}
```

For background jobs that need state, use database or Redis storage instead of PHP sessions.

---

## Summary: Session and Cookie Architecture

Understanding the session and cookie system is essential for building reliable Magento 2.4.8 extensions. Here are the key take-aways:

1. **Magento wraps Zend sessions** — `Magento\Framework\Session\SessionManager` provides DI integration and testability
2. **Different session names for different areas** — `adminhtml` for backend, `frontend` for storefront
3. **Customer session and admin auth session are separate** — Use `CustomerSession` and `AuthSession` respectively
4. **`web/session/` config controls cookies** — `cookie_lifetime`, `cookie_secure`, `cookie_httponly`, `cookie_samesite`
5. **Use `CookieManager` over raw `setcookie()`** — Respects Magento's global cookie settings
6. **Form keys prevent CSRF** — Every adminhtml form must include `form_key`
7. **Persistent cart uses a separate `persistent_session` cookie and DB table** — Different from regular sessions
8. **Redis is the recommended production backend** — Scalable, fast, handles expiration automatically
9. **Session validation can silently log users out** — User-Agent and IP changes trigger invalidation
10. **Cookie size limits are real** — Exceeding 4KB causes session loss; use Redis/DB sessions for complex carts
11. **GDPR requires cookie consent** — Essential cookies (session, form_key) cannot be blocked
12. **Admin sessions can be locked** — Prevents concurrent admin sessions
13. **Custom session namespaces must be registered in `di.xml`** — Avoid shutdown conflicts

---

## See Also

- [Topic 1: Order Architecture & State Machine](../08-order-extension.md) — Understanding the full checkout lifecycle that sessions support
- [Checkout README](../README.md) — Module overview and prerequisites
- [Adobe Commerce: Session Management](https://developer.adobe.com/commerce/php/development/components/sessions/) — Official documentation on session configuration
- [Adobe Commerce: Cookie Configuration](https://experienceleague.adobe.com/docs/commerce-operATIONS/configuration-guide/reference/cookie-reconfiguration.html) — Cookie path, domain, and lifetime configuration
- [Redis Session Handler](https://experienceleague.adobe.com/docs/commerce-operations/configuration-guide/cache/redis/redis-session.html) — Setting up Redis for session storage
- [GDPR Compliance in Magento](https://developer.adobe.com/commerce/pbx-gdpr-plugin/) — Cookie consent implementation guide
