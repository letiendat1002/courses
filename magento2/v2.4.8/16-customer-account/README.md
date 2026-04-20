# Topic 16: Customer Account Management

**Philosophy:** Customer accounts are the identity layer of any Magento store. Master the entity model, registration flows, address management, and security events.

---

## Overview

Customer account management covers the full lifecycle of customer identity in Magento 2 — from EAV-based entity architecture through registration, address management, group segmentation, website permissions, and authentication security. This module teaches the backend patterns for building account-centric features without touching storefront templates.

---

## Prerequisites

- Topic 4 (Data Layer) — customer uses EAV attributes and service contracts the same way products do
- Topic 7 (REST APIs) — customer API endpoints are auto-generated via service contracts; understanding `webapi.xml` and `SearchCriteria` helps
- Topic 6 (Admin UI) helpful for admin customer management

---

## Learning Objectives

- Understand the customer EAV entity model and its relationship to address entities
- Create customer accounts programmatically with full validation
- Manage billing/shipping addresses with default address flags
- Build CLI tools for bulk customer operations
- Use plugins and observers on the account management layer
- Add custom attributes to the customer API via extension attributes
- Implement customer group logic and segmentation
- Handle authentication security events

---

## By End of Module You Must Prove

- [ ] `CustomerRepositoryInterface::save()` used to create a customer programmatically with hashed password
- [ ] Customer address created and set as default billing/shipping
- [ ] Custom CLI commands (`training:customer:create`, `training:customer:address:dump`) working without errors
- [ ] Plugin on `CustomerRepositoryInterface::save()` that auto-assigns customer to group based on email domain
- [ ] Observer on `customer_customer_authenticated` that logs login event to custom table
- [ ] Custom customer attribute created via Data Patch and visible in API response via extension attributes
- [ ] All code passes PHPCS with zero errors
- [ ] DoD assessment passed

---

## Assessment Criteria

| Test | Time | Criteria |
|------|------|----------|
| Customer Creation | 20 min | CLI creates customer, password hashed, `customer_entity` row inserted |
| Address Management | 20 min | Address created, set as default, retrievable via `AddressRepository` |
| Group Plugin | 15 min | Plugin intercepts save, assigns wholesale group by email domain |
| Login Observer | 15 min | Login event logged to `training_customer_login_log` |
| Custom Attribute + API | 25 min | Attribute created, appears in `/V1/customers/me` response |
| DoD Assessment | 30 min | All criteria met, PHPCS zero errors |

---

## Topics

---

### Topic 1: Customer Architecture — Entity Model

**EAV Entity Structure:**

Customer is an EAV entity — just like products and categories. It has core attributes and EAV value tables.

**Core Tables:**

| Table | Purpose |
|-------|---------|
| `customer_entity` | Core customer data (entity_id, email, group_id, store_id, created_at, etc.) |
| `customer_entity_varchar` | String attribute values |
| `customer_entity_int` | Integer attribute values |
| `customer_entity_decimal` | Decimal attribute values |
| `customer_entity_text` | Text attribute values |
| `customer_entity_datetime` | Date/time attribute values |
| `customer_address_entity` | Address sub-entity |
| `customer_address_entity_*` | Address EAV value tables (same pattern) |
| `customer_group` | Group definitions |

**Customer Model:**

```php
<?php
// In any service class
use Magento\Customer\Model\Customer;
use Magento\Customer\Api\Data\CustomerInterface;

class CustomerService
{
    public function getCustomer(int $customerId): CustomerInterface
    {
        return $this->customerRepository->getById($customerId);
    }

    public function getCustomerByEmail(string $email, int $websiteId): CustomerInterface
    {
        // Search by email — website-scoped
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('email', $email)
            ->addFilter('website_id', $websiteId)
            ->create();

        $list = $this->customerRepository->getList($searchCriteria);
        $items = $list->getItems();
        return reset($items);
    }
}
```

**Customer Session:**

```php
<?php
use Magento\Customer\Model\Session as CustomerSession;

class MyBlock
{
    protected $customerSession;

    public function __construct(CustomerSession $customerSession)
    {
        $this->customerSession = $customerSession;
    }

    public function getCurrentCustomer(): ?CustomerInterface
    {
        if ($this->customerSession->isLoggedIn()) {
            return $this->customerSession->getCustomer();
        }
        return null;
    }

    public function getCustomerId(): ?int
    {
        return $this->customerSession->getCustomerId();
    }
}
```

**Key Distinction — Customer vs Guest:**

> **Why this matters:** In Magento, guests and customers are fundamentally different entities. A guest has no `customer_id` — their cart and order are tracked by a quote linked to an email address. When that guest registers, Magento attempts to match the existing quote to the new customer account. Understanding this flow is critical for implementing loyalty programs, abandoned cart emails, and order merging.

| Aspect | Customer | Guest |
|--------|----------|-------|
| `customer_id` | Has one | No |
| Quote ownership | `quote.customer_id` set | `quote.customer_id` = 0 |
| Order association | Linked to account | Linked to email only |
| Address book | Persistent | Ephemeral |
| Password | Hashed (bcrypt) | None |
| Session | Persistent across visits | Reset on browser close (unless cookie lifetime set) |

**Customer Session Management — Deep Dive:**

The `Magento\Customer\Model\Session` class manages customer sessions. Key methods:

```php
<?php
use Magento\Customer\Model\Session as CustomerSession;

class CustomerSessionService
{
    public function __construct(
        private readonly CustomerSession $customerSession,
    ) {}

    public function isLoggedIn(): bool
    {
        // Checks both customer ID and customer data are valid
        return $this->customerSession->isLoggedIn();
    }

    public function getCustomerId(): ?int
    {
        return $this->customerSession->getCustomerId();
    }

    public function getCustomer(): ?\Magento\Customer\Api\Data\CustomerInterface
    {
        return $this->customerSession->getCustomer();
    }

    public function setCustomer(\Magento\Customer\Api\Data\CustomerInterface $customer): void
    {
        // Persists the customer model in the session
        $this->customerSession->setCustomer($customer);
    }

    public function getGroupId(): int
    {
        // Returns the customer's group ID (1=General, 2=Wholesale, etc.)
        return (int) $this->customerSession->getCustomer()->getGroupId();
    }

    public function invalidate(): void
    {
        // Clears the session — used on logout
        $this->customerSession->destroy();
    }
}
```

**Session data stored in Redis (for multi-node deployments):**

When using Redis session storage, the session key format is `PHPREDIS_SESSION:<session_id>`. The session data stored includes:

```
[customer_id] => 123
[customer_group_id] => 2
[website_id] => 1
[store_id] => 1
[_session_validator_data] => [remote_addr, http_user_agent, session_start_time]
```

> **Pro Tip:** Never store large objects in the customer session. `$_SESSION['customer_model'] = $customerModel` serializes the entire EAV model with all attributes, bloating the session storage and slowing session deserialization. Store only the ID and load the model on demand.

**Authentication vs Authorization — The Two-Step Security Model:**

These are often confused but represent distinct security concepts:

| Concept | What It Answers | Magento Implementation |
|---------|---------------|----------------------|
| **Authentication** | "Who are you?" | Password verification, 2FA, login token |
| **Authorization** | "What are you allowed to do?" | ACL resources, customer group permissions |

```php
<?php
// Authentication — verifying identity
public function authenticate(string $email, string $password): bool
{
    /** @var AccountManagementInterface $accountManagement */
    $isValid = $this->accountManagement->authenticate($email, $password);
    return $isValid;
}

// Authorization — checking permissions
public function canAccess(\Magento\Customer\Model\Customer $customer, string $resource): bool
{
    /** @var \Magento\Framework\AuthorizationInterface $authorization */
    return $this->authorization->isAllowed($resource);
}
```

**Password Hashing — bcrypt and Why It Matters:**

Magento uses bcrypt via `Magento\Framework\Crypt\HashInterface`. Never store passwords in plain text or with weak hashing (MD5, SHA1).

```php
<?php
use Magento\Framework\Crypt\HashInterface;

// Getting a hash (for registration or password change)
public function hashPassword(string $plainPassword): string
{
    $hash = $this->hash->getHash($plainPassword, $this->hash->getHashAlgorithm());
    return $hash; // e.g., "$2y$12$KwH4dGK..."
}

// Verifying a password (for login)
public function verifyPassword(int $customerId, string $plainPassword): bool
{
    $customer = $this->customerRepository->getById($customerId);
    return $this->accountManagement->validatePassword($customerId, $plainPassword);
}
```

**How bcrypt works in Magento:**

1. `getHashAlgorithm()` returns `PASSWORD_BCRYPT` (cost factor 12 in Magento 2.4.x)
2. `password_hash()` is used with `PASSWORD_BCRYPT` and cost factor 12
3. The resulting hash is stored in `customer_entity.password_hash`
4. On login, `password_verify()` is used to compare

The cost factor of 12 means each password check requires ~250ms of CPU time, making brute-force attacks computationally expensive.

> **Common Pitfall:** If you need to check a password outside of `AccountManagement`, use `$this->accountManagement->validatePassword($customerId, $password)`. Do not call `password_verify()` directly on the stored hash — Magento's hashing includes a version prefix that must be handled.

**Two-Factor Authentication (2FA) Patterns:**

Magento supports 2FA via the `Magento_AdminAdobeIms` module. For custom 2FA implementations (e.g., requiring 2FA for specific customer groups):

```php
<?php
// Plugin on AccountManagementInterface to enforce 2FA for specific groups
class EnforceTwoFactorForWholesale
{
    public function beforeCreateAccount(
        \Magento\Customer\Api\AccountManagementInterface $subject,
        $customer,
        $password = null,
        $redirectUrl = null
    ): array {
        // For wholesale customers, 2FA is always required
        if ($customer->getGroupId() === 2) { // Wholesale
            $this->requireTwoFactor($customer);
        }
        return [$customer, $password, $redirectUrl];
    }

    private function requireTwoFactor(\Magento\Customer\Api\Data\CustomerInterface $customer): void
    {
        // Mark customer as requiring 2FA via extension attribute
        $extension = $customer->getExtensionAttributes() ?? $this->extensionFactory->create();
        $extension->setTwoFactorEnabled(false); // Set to false to force 2FA setup
        $customer->setExtensionAttributes($extension);
    }
}
```

**Customer Address Validation — Why It Matters:**

Invalid addresses cause failed shipments, returned packages, and customer frustration. Magento provides an address validation API via `Magento\Customer\Api\AddressValidationInterface`.

```php
<?php
public function validateAddressBeforeSave(AddressInterface $address): ValidationResult
{
    /** @var \Magento\Customer\Api\AddressValidationInterface $validator */
    $result = $this->validator->validate($address);

    if ($result->getErrors()) {
        foreach ($result->getErrors() as $error) {
            // Handle each validation error
            match ($error->getErrorCode()) {
                'street_empty' => $this->handleEmptyStreet($address),
                'city_invalid' => $this->handleInvalidCity($address),
                'region_invalid' => $this->handleInvalidRegion($address),
                default => $this->handleGenericError($error),
            };
        }
    }

    return $result;
}
```

> **Pro Tip:** Address validation via the built-in API requires carrier-level integration (e.g., UPS, FedEx). For a simple implementation, use Google Address Validation API or a third-party service (SmartyStreets, Lob) as a plugin on `AddressValidationInterface`.

---

| Aspect | Customer | Guest |
|--------|----------|-------|
| `customer_id` | Has one | No |
| Quote ownership | `quote.customer_id` set | `quote.customer_id` = 0 |
| Order association | Linked to account | Linked to email only |
| Address book | Persistent | Ephemeral |

---

### Topic 2: Customer Registration & Account Creation

**The Registration Service:**

```php
<?php
use Magento\Customer\Api\AccountManagementInterface;
use Magento\Customer\Api\Data\CustomerInterfaceFactory;

class RegistrationService
{
    protected $accountManagement;
    protected $customerFactory;
    protected $storeManager;

    public function __construct(
        AccountManagementInterface $accountManagement,
        CustomerInterfaceFactory $customerFactory,
        \Magento\Store\Model\StoreManagerInterface $storeManager
    ) {
        $this->accountManagement = $accountManagement;
        $this->customerFactory = $customerFactory;
        $this->storeManager = $storeManager;
    }

    public function registerCustomer(string $email, string $firstName, string $lastName): CustomerInterface
    {
        $websiteId = (int) $this->storeManager->getStore()->getWebsiteId();

        $customer = $this->customerFactory->create()
            ->setWebsiteId($websiteId)
            ->setEmail($email)
            ->setFirstname($firstName)
            ->setLastname($lastName)
            ->setGroupId(1); // General group

        // Hashing is automatic in createAccount()
        $account = $this->accountManagement->createAccount(
            $customer,
            'Password123!'  // Only needed for new accounts
        );

        return $account;
    }
}
```

**Password Hashing:**

Magento uses bcrypt by default via `Magento\Framework\Crypt\HashingInterface`:

```php
<?php
// Getting the hash for comparison
$hash = $this->hash->getHash($password, $this->hash->getHashAlgorithm());

// Verifying a password
$isValid = $this->accountManagement->validatePassword($customerId, $password);
```

**createAccount() Validation:**

```php
<?php
// Email uniqueness (per website)
if (!$this->accountManagement->isEmailAvailable($email, $websiteId)) {
    throw new \Magento\Framework\Exception\InvalidEmailException(__('Email already exists'));
}
```

**Events Triggered During Registration:**

| Event | When | Observer Use |
|-------|------|-------------|
| `customer_customer_authenticated` | After successful login | Audit, analytics |
| `customer_register_success` | After `createAccount()` completes | Welcome email, CRM sync |
| `customer_account_updated` | After account change | Notification flows |
| `customer_save_commit_after` | After any customer save | Segment recalculation |

**Account Confirmation (Email Verification):**

```php
<?php
// If email confirmation is required:
$this->accountManagement->createAccountWithPasswordHash(
    $customer,
    $hash, // Pre-computed hash
    null,  // Redirect URL
    true   // <-- this enables email confirmation flow
);
```

---

### Topic 3: Customer Address Management

**Address Entity:**

Address is a sub-entity — stored in `customer_address_entity` and its EAV value tables. It links to its parent customer via `parent_id`.

**Creating an Address for a Customer:**

```php
<?php
use Magento\Customer\Api\AddressRepositoryInterface;
use Magento\Customer\Api\Data\AddressInterfaceFactory;
use Magento\Customer\Api\Data\CustomerInterface;

class AddressService
{
    protected $addressRepository;
    protected $addressFactory;

    public function __construct(
        AddressRepositoryInterface $addressRepository,
        AddressInterfaceFactory $addressFactory
    ) {
        $this->addressRepository = $addressRepository;
        $this->addressFactory = $addressFactory;
    }

    public function createAddress(int $customerId, array $addressData): AddressInterface
    {
        /** @var AddressInterface $address */
        $address = $this->addressFactory->create()
            ->setParentId($customerId)
            ->setFirstname($addressData['firstname'])
            ->setLastname($addressData['lastname'])
            ->setStreet([$addressData['street']])
            ->setCity($addressData['city'])
            ->setRegion($addressData['region'])        // Region ID or string
            ->setPostcode($addressData['postcode'])
            ->setCountryId($addressData['country_id'])  // ISO code e.g., 'US'
            ->setTelephone($addressData['telephone'])
            ->setIsDefaultBilling(true)
            ->setIsDefaultShipping(false);

        return $this->addressRepository->save($address);
    }
}
```

**Setting Default Addresses:**

```php
<?php
// Set default billing on the customer object, then save customer + address
$customer = $this->customerRepository->getById($customerId);
$customer->setDefaultBilling($address->getId());
$customer->setDefaultShipping($address->getId());
$this->customerRepository->save($customer);
```

**Region Lookup:**

```php
<?php
use Magento\Directory\Api\CountryInformationAcquirerInterface;

$regions = $this->countryInfoAcquirer
    ->getCountryInfo($countryId)
    ->getAvailableRegions();

foreach ($regions as $region) {
    echo $region->getId();    // Region ID (integer)
    echo $region->getName();   // e.g., "California"
}
```

**Address Validation:**

```php
<?php
use Magento\Customer\Api\AddressValidationInterface;

$result = $this->addressValidation->validate($address);
if (!$result->getErrors()) {
    // Address is valid
} else {
    foreach ($result->getErrors() as $error) {
        echo $error->getErrorCode();   // e.g., 'street_empty'
        echo $error->getMessage();
    }
}
```

**Getting All Addresses for a Customer:**

```php
<?php
use Magento\Customer\Api\AddressRepositoryInterface;

public function getCustomerAddresses(int $customerId): array
{
    return $this->addressRepository->getAddressesByCustomerId($customerId);
}
```

---

### Topic 4: Customer REST API — Account & Address Endpoints

**Auto-Generated Customer API:**

Magento auto-generates REST endpoints for any entity with Service Contracts in `Api/`:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/V1/customers/me` | GET | Current logged-in customer |
| `/V1/customers/:id` | GET | Get customer by ID |
| `/V1/customers` | POST | Create customer |
| `/V1/customers/:id` | PUT | Update customer |
| `/V1/customers/:id/addresses` | GET | List addresses |
| `/V1/customers/:id/addresses/:addressId` | GET/DELETE/POST/PUT | Manage address |
| `/V1/customers/search` | GET | Search customers |
| `/V1/integration/customer/token` | POST | Customer login (get token) |

**Getting a Customer Token:**

```bash
curl -X POST http://localhost:8080/rest/V1/integration/customer/token \
  -H "Content-Type: application/json" \
  -d '{"username":"alice@example.com", "password":"Password123!"}'
```

Returns: `"token_string_here"` — use as `Authorization: Bearer token_string`

**Custom Attributes in API Response — Extension Attributes:**

```php
<?php
// etc/extension_attributes.xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Api/etc/extension_attributes.xsd">
    <extension_attributes for="Magento\Customer\Api\Data\CustomerInterface">
        <attribute code="loyalty_card_number" type="string"/>
    </extension_attributes>
</config>
```

```php
<?php
// In plugin on CustomerRepositoryInterface::get()
public function afterGetById(
    CustomerRepositoryInterface $subject,
    CustomerInterface $customer
): CustomerInterface {
    $extension = $customer->getExtensionAttributes() ?? $this->extensionFactory->create();
    $extension->setLoyaltyCardNumber($this->getLoyaltyCardNumber($customer->getId()));
    $customer->setExtensionAttributes($extension);
    return $customer;
}
```

**Searching Customers:**

```bash
# Search by email
curl "http://localhost:8080/rest/V1/customers/search?searchCriteria[filterGroups][0][filters][0][field]=email&searchCriteria[filterGroups][0][filters][0][value]=alice@example.com" \
  -H "Authorization: Bearer ADMIN_TOKEN"

# Search by group
curl "http://localhost:8080/rest/V1/customers/search?searchCriteria[filterGroups][0][filters][0][field]=group_id&searchCriteria[filterGroups][0][filters][0][value]=2" \
  -H "Authorization: Bearer ADMIN_TOKEN"
```

---

### Topic 5: Customer Groups & Segmentation

**Customer Groups:**

```php
<?php
use Magento\Customer\Api\GroupManagementInterface;
use Magento\Customer\Api\Data\GroupInterface;

// Get default groups
$notLoggedIn = $this->groupManagement->getNotLoggedInGroup();
$generalGroup = $this->groupManagement->getGeneralGroup();

// Assign customer to group
$customer = $this->customerRepository->getById($customerId);
$customer->setGroupId(2); // Wholesale
$this->customerRepository->save($customer);
```

**Default Group IDs:**

| Group | ID |
|-------|----|
| NOT LOGGED IN | 0 |
| General | 1 |
| Wholesale | 2 |
| Retailer | 3 |

**Plugin: Auto-Assign Group by Email Domain:**

```php
<?php
// Plugin/CustomerGroupAssignment.php
namespace Training\Customer\Plugin;

use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Customer\Api\Data\CustomerInterface;

class CustomerGroupAssignment
{
    private $wholesaleDomain = '@wholesale';
    private $vipDomain = '@vip';

    public function beforeSave(
        CustomerRepositoryInterface $subject,
        CustomerInterface $customer
    ): array {
        $email = $customer->getEmail();

        if (str_contains($email, $this->wholesaleDomain)) {
            $customer->setGroupId(2); // Wholesale
        } elseif (str_contains($email, $this->vipDomain)) {
            $customer->setGroupId(3); // VIP (custom group)
        }

        return [$customer];
    }
}
```

```xml
<!-- di.xml -->
<type name="Magento\Customer\Api\CustomerRepositoryInterface">
    <plugin name="training_customer_group_assign" type="Training\Customer\Plugin\CustomerGroupAssignment"/>
</type>
```

**Customer Segments (Marketing):**

Segments are dynamic collections based on conditions — created in admin (Marketing → Segments). Backend developers interact with them via observer when segment membership changes:

> **Why segmentation matters for backend developers:** Customer segments enable dynamic content and promotions without hardcoding conditions in PHP. When a segment's conditions change (e.g., "customers who purchased over $500 in the last 30 days"), all customers matching those conditions automatically become part of the segment. This is used for targeted email campaigns, dynamic price lists, and personalized content.

**Working with segments programmatically:**

```php
<?php
use Magento\CustomerSegment\Model\Segment;
use Magento\CustomerSegment\Api\SegmentRepositoryInterface;

class SegmentService
{
    public function __construct(
        private readonly Segment $segmentModel,
        private readonly SegmentRepositoryInterface $segmentRepository,
    ) {}

    /**
     * Get all customers matching a segment.
     */
    public function getSegmentCustomers(int $segmentId): array
    {
        $segment = $this->segmentModel->load($segmentId);
        if (!$segment->getId()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Segment %1 not found', $segmentId)
            );
        }

        // The segment's customer collection is built dynamically
        // based on the segment's conditions
        return $segment->getCustomerCollection();
    }

    /**
     * Check if a specific customer matches a segment.
     */
    public function isCustomerInSegment(int $customerId, int $segmentId): bool
    {
        $segment = $this->segmentModel->load($segmentId);
        return (bool) $segment->isCustomerInSegment($customerId);
    }

    /**
     * Get all segments a customer belongs to.
     */
    public function getCustomerSegments(int $customerId): array
    {
        /** @var \Magento\CustomerSegment\Model\ResourceModel\Segment $resource */
        $resource = $this->segmentModel->getResource();
        return $resource->getCustomerSegments($customerId);
    }
}
```

**Triggering actions when a customer enters or leaves a segment:**

Use the `customer_save_commit_after` observer to check segment membership and trigger actions:

```php
<?php
// etc/events.xml
<event name="customer_save_commit_after">
    <observer name="training_segment_actions"
              instance="Training\Customer\Observer\SegmentMembershipChanged"/>
</event>
```

```php
<?php
class SegmentMembershipChanged implements ObserverInterface
{
    public function __construct(
        private readonly Segment $segmentModel,
        private readonly \Training\Customer\Api\WelcomeEmailServiceInterface $emailService,
    ) {}

    public function execute(Observer $observer): void
    {
        /** @var CustomerInterface $customer */
        $customer = $observer->getData('customer');

        // Check if customer entered the VIP segment (segment ID 10)
        $vipSegmentId = 10;
        if ($this->segmentModel->load($vipSegmentId)->isCustomerInSegment($customer->getId())) {
            $this->emailService->sendVipWelcomeEmail($customer);
            $this->logSegmentEntry($customer->getId(), $vipSegmentId);
        }
    }
}
```

```php
<?php
// Observer/CustomerSegmentChanged.php
use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;

class CustomerSegmentChanged implements ObserverInterface
{
    public function execute(Observer $observer): void
    {
        $customer = $observer->getData('customer');
        // Trigger external CRM sync, email campaign enrollment, etc.
    }
}
```

```xml
<!-- events.xml -->
<event name="customer_save_commit_after">
    <observer name="training_segment_sync"
              instance="Training\Customer\Observer\CustomerSegmentChanged"/>
</event>
```

---

### Topic 6: Account Permissions & Website Restrictions

**Account Sharing Settings:**

In Magento Admin: **Stores → Configuration → Customers → Customer Configuration → Account Sharing Options**

| Setting | Effect |
|---------|--------|
| Per Website | Email must be unique per website only |
| Global | Email must be unique across all websites |

**Per-Website Account Behavior:**

```php
<?php
// Check if email already exists on a specific website
$exists = $this->customerRepository->getList(
    $searchCriteriaBuilder->addFilter('email', $email)->create()
)->getTotalCount();
```

**Plugin to Enforce Per-Website Email:**

```php
<?php
// Plugin/EnforceWebsiteEmailUniqueness.php
use Magento\Customer\Api\Data\CustomerInterface;

class EnforceWebsiteEmailUniqueness
{
    public function beforeSave($subject, CustomerInterface $customer): void
    {
        $email = $customer->getEmail();
        $websiteId = $customer->getWebsiteId();

        if ($this->isEmailTakenOnWebsite($email, $websiteId, $customer->getId())) {
            throw new \Magento\Framework\Exception\InvalidEmailException(
                __('This email is already used on website %1', $websiteId)
            );
        }
    }
}
```

**Multi-Store Session Context:**

```php
<?php
// When a customer from website A visits website B
// Magento checks sharing settings before creating a session

// If global sharing: customer can login on any website with same email
// If per-website: customer needs separate account per website
```

---

### Topic 7: Account Security & Authentication Events

**Authenticated Event — `customer_customer_authenticated`:**

```php
<?php
// Observer/CustomerAuthenticatedLogger.php
namespace Training\Customer\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Psr\Log\LoggerInterface;

class CustomerAuthenticatedLogger implements ObserverInterface
{
    protected $logger;
    protected $loginLogFactory;

    public function __construct(
        LoggerInterface $logger,
        \Training\Customer\Model\LoginLogFactory $loginLogFactory
    ) {
        $this->logger = $logger;
        $this->loginLogFactory = $loginLogFactory;
    }

    public function execute(Observer $observer): void
    {
        $event = $observer->getData('event');
        $passwordHash = $event->getData('password');
        // Event contains: model (customer), password hash

        $customer = $event->getData('model');
        $loginLog = $this->loginLogFactory->create();
        $loginLog->setCustomerId($customer->getId())
            ->setEmail($customer->getEmail())
            ->setLoggedInAt(date('Y-m-d H:i:s'))
            ->setWebsiteId($customer->getWebsiteId());

        $loginLog->save();
        $this->logger->info('Customer logged in', ['email' => $customer->getEmail()]);
    }
}
```

```xml
<!-- events.xml -->
<event name="customer_customer_authenticated">
    <observer name="training_auth_logger"
              instance="Training\Customer\Observer\CustomerAuthenticatedLogger"/>
</event>
```

**Failed Login Tracking — `customer_login_failure`:**

```php
<?php
public function execute(Observer $observer): void
{
    $userName = $observer->getData('username'); // email or username
    $exception = $observer->getData('exception');

    $this->logger->warning('Login failed', [
        'user' => $userName,
        'reason' => $exception->getMessage()
    ]);
}
```

```xml
<event name="customer_login_failure">
    <observer name="training_login_failure_logger"
              instance="Training\Customer\Observer\LoginFailureLogger"/>
</event>
```

**Account Locking:**

Magento's `AccountManagement` locks accounts after 6 failed attempts (configurable):

```bash
# Admin config: Stores → Configuration → Customers → Customer Configuration → Account Locking
# Max failures: 6 (default)
# Lock timeout: 30 minutes (default)
```

**Password Reset:**

```php
<?php
// Initiate password reset email
$this->accountManagement->initiatePasswordReset(
    $email,
    AccountManagement::EMAIL_RESET,  // or ::EMAIL_PASSWORD
    $websiteId
);

// Set new password via token (token received in email link)
$this->accountManagement->resetPassword(
    $email,
    $resetToken,
    $newPassword
);
```

**Plugin — Block Disposable Email Registration:**

> **Common Pitfall:** The `customer_login_failure` event only fires on authentication failures via `AccountManagement::authenticate()`. If you implement a custom login endpoint that bypasses `AccountManagement`, the event will not fire and your failure logging will miss those attempts. Always route authentication through `AccountManagementInterface` to benefit from Magento's built-in security events.

**Account Locking — Understanding the Lockout Mechanism:**

Magento's `AccountManagement` implements account locking after failed login attempts. This is configured in:

**Admin → Stores → Configuration → Customers → Customer Configuration → Account Lockout Settings**

```php
<?php
// Check if an account is locked before attempting authentication
public function isAccountLocked(int $customerId): bool
{
    $lockThreshold = 6; // Default: 6 failed attempts
    $lockTimeout = 30;  // Default: 30 minutes

    /** @var \Magento\Customer\Model\ResourceModel\Visitor $visitorResource */
    $visitorResource = $this->visitorResource;

    // Check failed login attempts from the customer_log table
    $failures = $this->getFailedLoginAttempts($customerId);

    if ($failures >= $lockThreshold) {
        $lastFailureTime = $this->getLastFailureTime($customerId);
        $unlockTime = strtotime("+{$lockTimeout} minutes", $lastFailureTime);

        if (time() < $unlockTime) {
            return true; // Account is still locked
        }
    }

    return false;
}
```

To unlock a customer account programmatically (e.g., after manual verification):

```php
<?php
public function unlockAccount(int $customerId): void
{
    /** @var \Magento\Customer\Model\ResourceModel\Customer $customerResource */
    $customerResource = $this->customerResource;

    // Reset failed login counter
    $customerResource->resetAuthenticationLevel($customerId);
}
```

**Password Reset — Token-Based Flow:**

Magento's password reset uses a token-based flow (not a link with the password in it). The flow:

1. Customer requests reset → `initiatePasswordReset()` generates a token (stored in `customer_entity`.`rp_token`)
2. Email sent with link containing token → customer clicks link
3. Link points to `customer/account/createPassword` with token in URL
4. Customer submits new password → `resetPassword()` verifies token and updates hash

```php
<?php
// To invalidate all existing sessions when a password is changed
// (forces re-login everywhere):
public function resetPasswordWithSessionInvalidation(
    string $email,
    string $resetToken,
    string $newPassword
): void {
    $websiteId = (int) $this->storeManager->getWebsite()->getId();

    // This invalidates the token so it can't be used again
    $this->accountManagement->resetPassword(
        $email,
        $resetToken,
        $newPassword
    );

    // Force logout of all existing sessions for this customer
    $customer = $this->customerRepository->get($email, $websiteId);
    $this->sessionManager->destroyByCustomerId($customer->getId());
}
```

---

## Reference Exercises

**Exercise 1:** Create a CLI command `bin/magento training:customer:create --email=alice@example.com --firstname=Alice --lastname=Smith --website=1 --group=2` that programmatically creates a customer account with the `CustomerAccountManagementInterface::createAccount()` method. Verify with SQL: `SELECT * FROM customer_entity WHERE email = 'alice@example.com';`

**Exercise 2:** Create a CLI command `bin/magento training:customer:address:dump <customer_id>` that fetches all addresses for the given customer via `AddressRepositoryInterface::getAddressesByCustomerId()` and prints each address as formatted output (street, city, country, default flags).

**Exercise 3:** Build a plugin on `CustomerRepositoryInterface::save()` that intercepts any customer save and automatically assigns the customer to "Wholesale" group (group_id=2) if their email contains `@wholesale`. Test by creating a customer with a `@wholesale` domain email and verify the group_id.

**Exercise 4:** Create an observer on `customer_customer_authenticated` that logs the login event (customer_id, email, timestamp, website_id) to a custom `training_customer_login_log` table. Wire the table creation via `db_schema.xml` and verify logs appear after a successful customer login.

**Exercise 5:** Add a custom customer attribute `loyalty_card_number` (varchar, max 20 chars) via `Setup\Patch\Data\AddLoyaltyCardAttribute`. Then expose it in the Customer REST API by adding it to `extension_attributes.xml` and implementing a plugin on `CustomerRepositoryInterface::get()` that sets the value from storage into the extension attribute. Verify by calling `GET /V1/customers/me` with a valid token.

**Exercise 6:** Build a CLI command `bin/magento training:customer:segment:rebuild <segment_id>` that loads a customer segment by ID and outputs the count of matching customers. Use the `Segment` model and its `getCustomerCollection()` method.

**Exercise 7 (optional):** Write a plugin on `AccountManagementInterface::createAccount()` that rejects registration from any email domain that appears in a `training_disposable_domain_blacklist` database table. The check should query the table before allowing account creation to proceed.

---

## Reading List

- [Customer API](https://developer.adobe.com/commerce/webapi/rest/use-apis/customers-wishtlists/) — Customer endpoints
- [Customer Module Architecture](https://developer.adobe.com/commerce/php/development/components/entities/) — EAV structure
- [Customer Account Management](https://developer.adobe.com/commerce/webapi/rest/tutorials/customers-wishlists/) — Registration, authentication
- [Extension Attributes](https://developer.adobe.com/commerce/php/development/components/extension-attributes/) — Adding fields to API

---

## Edge Cases & Troubleshooting

| Issue | Symptom | Solution |
|-------|--------|----------|
| Duplicate email on save | `InvalidEmailException` | Check `website_id` scope — email uniqueness is per-website |
| Address not saving | Silent failure, no exception | Verify `parent_id` = customer ID, country_id is ISO format |
| Customer API returns empty custom attribute | Attribute not in response | Must add to `extension_attributes.xml` AND set via plugin |
| Group assignment not persisted | Group reverts on next load | Plugin on `beforeSave()` must set group BEFORE the save call |
| Session not persisting | Customer logs out immediately | Check `session.save` config and cookie lifetime |
| Customer token expires | 401 on subsequent requests | Default token TTL is 1 hour; check `oauth_token_ttl` |

---

## Common Mistakes to Avoid

1. ❌ Setting `entity_id` manually on new customer → Let Magento generate it
2. ❌ Storing password in plain text → Always use `createAccount()` which hashes automatically
3. ❌ Forgetting `website_id` when creating customer → Throws `InvalidEmailException`
4. ❌ Not setting `parent_id` on address → Address has no customer link
5. ❌ Hardcoding group IDs (magic numbers) → Use named constants or config
6. ❌ Adding custom attribute without `extension_attributes.xml` → Attribute not visible in API
7. ❌ Plugin on `save()` that reads from DB → Creates circular load; use `beforeSave()` for pre-process logic only

---

*Optional Module: Customer Account Management*  
---

*Magento 2 Backend Developer Course — Topic 16*  
*Language: English*
