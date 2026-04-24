---
title: "22 - Customer Account"
description: "Magento 2.4.8 customer account: customer EAV attributes, registration flow, account management, loyalty programs, customer session"
rank: 22
pathways: [magento2-deep-dive]
see_also:
  - "Data layer (EAV): ../04-data-layer/README.md"
---

# Topic 22 — Customer Account

Magento 2.4.8's Customer module manages customer data, authentication, and account lifecycle. The module implements a standard EAV entity pattern for extensible customer and address attributes, with deep integration into the checkout and session management systems.

## 1. Customer Entity (EAV)

The customer entity uses Magento's EAV (Entity-Attribute-Value) architecture, allowing custom attributes without schema changes.

### Customer EAV Table Structure

The `customer_customer` EAV table structure spans multiple physical tables:

| Table | Purpose |
|-------|---------|
| `customer_entity` | Core customer records (entity_id, email, group_id, created_at, etc.) |
| `customer_entity_varchar` | String attributes (firstname, lastname, gender, etc.) |
| `customer_entity_int` | Integer attributes (prefix, suffix, taxvat, etc.) |
| `customer_entity_text` | Text attributes (bio, custom_attributes JSON) |
| `customer_entity_datetime` | Date attributes |
| `customer_entity_decimal` | Decimal attributes (reward_points_balance, etc.) |
| `customer_entity_eav` | Attribute metadata definitions |

### Customer EAV Attribute Metadata

The `customer_eav_attribute` table defines all available customer attributes:

```php
// Magento\Customer\Model\ResourceModel\Attribute
declare(strict_types=1);

namespace Magento\Customer\Model\ResourceModel;

class Attribute extends \Magento\Eav\Model\Entity\AbstractEntity
{
    public function getAttributeCodes(): array
    {
        $select = $this->_getReadConnection()->select()
            ->from($this->getTable('customer_eav_attribute'), 'attribute_code');
        return $this->_getReadConnection()->fetchCol($select);
    }
}
```

Attribute properties include `is_visible`, `is_required`, `is_system`, `input_filter`, `multiline_count`, `data_model`.

### Customer Entity Model

```php
// Magento\Customer\Model\ResourceModel\Customer
declare(strict_types=1);

namespace Magento\Customer\Model\ResourceModel;

class Customer extends \Magento\Eav\Model\Entity\AbstractEntity
{
    public function __construct(
        \Magento\Eav\Model\Config $eavConfig,
        \Magento\Store\Model\StoreManagerInterface $storeManager,
        \Magento\Framework\Stdlib\DateTime $dateTime
    ) {}

    public function loadByEmail($customer, $email): void
    {
        $select = $this->_init()->where('email = ?', $email);
        $this->load($customer, $select);
    }
}
```

### Default Customer Attributes

| Attribute Code | Input Type | Visible | Required |
|----------------|------------|---------|----------|
| firstname | text | Yes | Yes |
| lastname | text | Yes | Yes |
| email | text | Yes | Yes |
| gender | select | Yes | No |
| taxvat | text | No | No |
| dob | date | Yes | No |

---

## 2. Customer Address

Customer addresses are separate EAV entities with their own attribute set.

### Address EAV Tables

| Table | Purpose |
|-------|---------|
| `customer_address_entity` | Core address records |
| `customer_address_entity_varchar` | String attributes |
| `customer_address_entity_int` | Integer attributes |
| `customer_address_entity_text` | Text attributes |
| `customer_address_entity_datetime` | Date attributes |
| `customer_address_entity_decimal` | Decimal attributes |
| `customer_address_entity_eav` | Attribute metadata |

### Address Attributes

The `customer_address_entity` table stores:

```php
// Magento\Customer\Model\Address
declare(strict_types=1);

namespace Magento\Customer\Model;

class Address extends \Magento\Framework\Model\AbstractExtensibleModel
{
    public function getStreet(): array
    {
        return $this->getData('street');
    }

    public function getCity(): string
    {
        return $this->getData('city');
    }

    public function getRegion(): \Magento\Customer\Api\Data\RegionInterface
    {
        return $this->_getExtensionAttributes()->getRegion();
    }

    public function getCountryId(): string
    {
        return $this->getData('country_id');
    }

    public function getPostcode(): string
    {
        return $this->getData('postcode');
    }

    public function getTelephone(): string
    {
        return $this->getData('telephone');
    }
}
```

### Default Address Attributes

| Attribute Code | Input Type | Purpose |
|----------------|------------|---------|
| firstname | text | Contact first name |
| lastname | text | Contact last name |
| company | text | Company name |
| street | multiline | Street address (2 lines) |
| city | text | City name |
| region | text | State/province region |
| country_id | select | Country |
| postcode | text | ZIP/postal code |
| telephone | text | Phone number |
| fax | text | Fax number |

### Address Format

The address format determines which attributes display and in what order:

```php
// Magento\Customer\Model\Address\Config
declare(strict_types=1);

namespace Magento\Customer\Model\Address;

class Config
{
    public function getRendererByCode(string $formatCode): AddressRendererInterface
    {
        // Returns \Magento\Framework\View\Element\Template for rendering
    }

    public function getFormatByCode(string $code): array
    {
        // Returns format template with attribute排列
    }
}
```

Formats: `html`, `pdf`, `oneline`, `text`

---

## 3. Registration Flow

Customer registration follows a specific controller lifecycle through the AccountController.

### CreatePost Controller

The `Magento\Customer\Controller\Account\CreatePost` handles form submission:

```php
// Magento\Customer\Controller\Account\CreatePost
declare(strict_types=1);

namespace Magento\Customer\Controller\Account;

class CreatePost extends \Magento\Customer\Controller\Account\Index
{
    public function __construct(
        \Magento\Framework\App\Action\Context $context,
        \Magento\Customer\Model\AccountManagement $accountManagement,
        \Magento\Customer\Model\Session $customerSession,
        \Magento\Framework\Data\Form\FormKey\Validator $formKeyValidator
    ) {}

    public function execute(): \Magento\Framework\App\Response\Http
    {
        $formKeyIsValid = $this->formKeyValidator->validate($this->getRequest());
        $isPost = $this->getRequest()->isPost();

        if ($formKeyIsValid && $isPost) {
            $data = $this->getRequest()->getPost();
            $this->accountManagement->createAccount($data);
            // Redirect to success page
        }
    }
}
```

### AccountManagement::createAccount

```php
// Magento\Customer\Model\AccountManagement
declare(strict_types=1);

namespace Magento\Customer\Model;

class AccountManagement implements \Magento\Customer\Api\AccountManagementInterface
{
    public function createAccount(array $data): \Magento\Customer\Api\Data\CustomerInterface
    {
        // 1. Validate email uniqueness
        // 2. Hash password
        // 3. Create customer EAV entity
        // 4. Send email notification
        // 5. Login customer (if auto-login enabled)
    }

    public function validateCustomerData(array $data): bool
    {
        // Validate required attributes
        // Check email format
        // Validate password strength
    }

    public function checkCustomerDuplicate(string $email): bool
    {
        // Returns true if email exists
    }
}
```

### Registration Lifecycle

```
1. User submits form (POST to /customer/account/createpost)
        ↓
2. CreatePost::execute() validates form key
        ↓
3. AccountManagement::validateCustomerData()
        ↓
4. AccountManagement::createAccount()
        ├── Hash password with brypt
        ├── Create customer_entity record
        ├── Insert customer_eav_* attribute values
        └── Dispatch 'customer_register_success' event
        ↓
5. EmailNotification::newAccount() sends email
        ↓
6. Optional: CustomerSession::setCustomerAsLoggedIn()
```

### Events Dispatched

| Event | Purpose |
|-------|---------|
| `customer_register_signup_account` | Before account creation |
| `customer_register_success` | After successful registration |
| `customer_login` | After customer login |
| `customer_logout` | After customer logout |

---

## 4. Customer Session

The `Magento\Customer\Model\Session` manages persistent customer authentication state across requests.

### Session Model

```php
// Magento\Customer\Model\Session
declare(strict_types=1);

namespace Magento\Customer\Model;

class Session implements \Magento\Customer\Api\Data\CustomerInterface
{
    public function __construct(
        \Magento\Framework\Session\SessionManagerInterface $sessionManager,
        \Magento\Customer\Api\CustomerRepositoryInterface $customerRepository,
        \Magento\Framework\Stdlib\DateTime $dateTime
    ) {}

    public function login(array $loginData): \Magento\Customer\Api\Data\CustomerInterface
    {
        // Authenticate via AccountManagement
        // Set customer in session
        // Regenerate session ID
    }

    public function logout(): void
    {
        // Clear customer from session
        // Invalidate session
        // Dispatch logout event
    }

    public function setCustomer(\Magento\Customer\Api\Data\CustomerInterface $customer): void
    {
        $this->sessionManager->setCustomerData($customer);
    }

    public function getCustomer(): \Magento\Customer\Api\Data\CustomerInterface
    {
        return $this->sessionManager->getCustomerData();
    }

    public function isLoggedIn(): bool
    {
        return $this->sessionManager->getCustomerId() !== null;
    }

    public function getId(): ?int
    {
        return $this->sessionManager->getCustomerId();
    }
}
```

### Persistent Session

When "Remember Me" is enabled, Magento creates a persistent session:

```php
// Magento\Persistent\Model\Session
declare(strict_types=1);

namespace Magento\Persistent\Model;

class Session
{
    public function setCustomerId(int $customerId): void
    {
        $this->_session->setData('customer_id', $customerId);
        // Cookie expires based on persistent_lifetime setting
    }

    public function getCustomerId(): ?int
    {
        return $this->_session->getData('customer_id');
    }
}
```

### Login Flow

```php
// POST /customer/account/login
public function loginPost(): \Magento\Framework\App\Response\Http
{
    $credentials = $this->getRequest()->getPost('login');

    // Validate credentials
    $customer = $this->accountManagement->authenticate(
        $credentials['username'],
        $credentials['password']
    );

    // Set session
    $this->customerSession->setCustomer($customer);

    // Regenerate session ID (prevent session fixation)
    $this->sessionManager->regenerateId();

    return $this->redirect Success;
}
```

### Logout Flow

```php
public function logout(): \Magento\Framework\Controller\Result\Redirect
{
    // 1. Get customer ID before clearing
    $customerId = $this->customerSession->getId();

    // 2. Clear session data
    $this->customerSession->logout();

    // 3. Invalidate session cookie
    $this->sessionManager->destroy();

    // 4. Dispatch event for dependent modules
    $this->_eventManager->dispatch('customer_logout', ['customer' => $customer]);

    return $this->redirectToLogin();
}
```

---

## 5. Account Management

The `AccountManagementInterface` defines the contract for customer account operations.

### AccountManagementInterface

```php
// Magento\Customer\Api\AccountManagementInterface
declare(strict_types=1);

namespace Magento\Customer\Api;

interface AccountManagementInterface
{
    public function createAccount(array $data): \Magento\Customer\Api\Data\CustomerInterface;
    public function authenticate(string $email, string $password): \Magento\Customer\Api\Data\CustomerInterface;
    public function changePassword(string $email, string $oldPassword, string $newPassword): bool;
    public function resetPassword(string $email, string $resetToken, string $newPassword): bool;
    public function sendPasswordResetEmail(string $email): void;
    public function validateCustomerData(array $data): bool;
    public function getConfirmationStatus(int $customerId): string;
    public function resendConfirmation(string $email): void;
}
```

### Password Reset

```php
// Password reset token flow
public function initiatePasswordReset(string $email, string $template): bool
{
    // 1. Load customer by email
    // 2. Generate reset token (cryptographically secure)
    // 3. Hash token and store in customer_entity
    // 4. Send email with reset link
}

public function resetPassword(string $email, string $resetToken, string $newPassword): bool
{
    // 1. Load customer
    // 2. Verify token matches (within expiry window)
    // 3. Hash new password
    // 4. Update customer_entity
    // 5. Invalidate token
}
```

Token stored as `rp_token` in `customer_entity`, with `rp_token_created_at` for expiry (default 1 hour).

### Email Change Flow

```php
public function initializeEmailChange(int $customerId, string $email): void
{
    // 1. Validate new email uniqueness
    // 2. Generate confirmation token
    // 3. Store pending email change
    // 4. Send confirmation to new email
}

public function confirmEmail(string $customerId, string $confirmationToken): bool
{
    // 1. Verify token
    // 2. Update email address
    // 3. Send notification to old email
}
```

---

## 6. Customer Data Section

Magento uses section-based data loading to optimize page performance and handle checkout-specific data updates.

### Section Pool Interface

```php
// Magento\Customer\CustomerData\SectionPoolInterface
declare(strict_types=1);

namespace Magento\Customer\CustomerData;

interface SectionPoolInterface
{
    /**
     * Get all section names
     *
     * @return string[]
     */
    public function getSectionNames(): array;

    /**
     * Get section data by name
     *
     * @param string $sectionName
     * @param int|null $customerId
     * @return array
     */
    public function getSectionData(string $sectionName, ?int $customerId = null): array;
}
```

### Layout Handle

The `customer/data-source` layout handle determines which sections to load:

```xml
<!-- Magento_Customer/layout/customer_account_index.xml -->
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <reference name="customer_account_dashboard">
            <block name="customer.account.link.back" />
        </reference>
    </body>
</page>
```

### Default Sections

| Section | Data Provider | Purpose |
|---------|--------------|---------|
| `customer` | CustomerData | Customer profile, group, reward points |
| `wishlist` | WishlistData | Wishlist items count |
| `checkout` | CheckoutData | Cart items, totals for checkout |
| `header` | HeaderData | Currency, locale, cart count |

### JS Customer Data

The frontend uses `mage-customer` for section management:

```javascript
// Magento_Customer/js/customer-data
define([
    'jquery',
    'storage-manager'
], function ($, storageManager) {
    'use strict';

    var sections = ['customer', 'wishlist', 'cart', 'checkout'];

    function reload(sectionNames) {
        $.ajax({
            url: '/customer/section/load',
            type: 'GET',
            data: { sections: sectionNames.join(',') },
            dataType: 'json',
            success: function (sectionsData) {
                storageManager.set(sectionsData);
            }
        });
    }

    // Expire cache after certain actions (login, logout, cart update)
    function invalidateCache(action) {
        // POST to /customer/section/ invalidate with action type
    }
});
```

### Section Load Controller

```php
// Magento\Customer\Controller\Section\Load
declare(strict_types=1);

namespace Magento\Customer\Controller\Section;

class Load implements \Magento\Framework\App\Action\HttpGetActionInterface
{
    public function execute(): \Magento\Framework\App\Response\Http
    {
        $sectionNames = explode(',', $this->getRequest()->getParam('sections', ''));

        $output = [];
        foreach ($sectionNames as $sectionName) {
            $output[$sectionName] = $this->sectionPool->getSectionData($sectionName);
        }

        $this->getResponse()->representJson(
            json_encode($output)
        );

        return $this->getResponse();
    }
}
```

---

## 7. Custom EAV Attributes

Adding custom attributes to the customer entity requires `customer_eav_attribute` registration and data setup.

### Attribute Registration

```php
// Module Setup/UpgradeData.php
declare(strict_types=1);

namespace MyVendor\CustomerModule\Setup;

use Magento\Eav\Setup\EavSetupFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class UpgradeData implements ModuleDataSetupInterface
{
    public function __construct(EavSetupFactory $eavSetupFactory)
    {
        $this->eavSetupFactory = $eavSetupFactory;
    }

    public function upgrade(ModuleDataSetupInterface $setup): void
    {
        $eavSetup = $this->eavSetupFactory->create(['setup' => $setup]);

        $eavSetup->addAttribute(
            \Magento\Customer\Model\Customer::ENTITY,
            'customer_type', // Attribute code
            [
                'type' => 'int',
                'label' => 'Customer Type',
                'input' => 'select',
                'source' => \MyVendor\CustomerModule\Model\Attribute\Source\CustomerType::class,
                'required' => false,
                'system' => 0, // Custom attribute
                'user_defined' => true,
                'position' => 100,
            ]
        );
    }
}
```

### Adminhtml Form Configuration

```xml
<!-- etc/adminhtml/di.xml -->
<virtualType name="Magento\Customer\Model\Attribute\Data\Text">
    <arguments>
        <argument name="filter" xsi:type="object">Magento\Framework\Data\Form\Filter\Trim</argument>
    </arguments>
</virtualType>

<!-- Customer form fields configuration -->
<!-- etc/adminhtml/system.xml -->
<section id="customer">
    <group id="customer_custom_attributes">
        <field id="customer_type">
            <label>Customer Type</label>
            <frontend_input>select</frontend_input>
        </field>
    </group>
</section>
```

### Storefront Display

For storefront display, create a plugin on the Customer data section:

```php
// etc/di.xml
<type name="Magento\Customer\CustomerData\Customer">
    <plugin name="MyVendor_CustomerModule" type="MyVendor\CustomerModule\Plugin\CustomerDataPlugin" />
</type>
```

```php
// MyVendor\CustomerModule\Plugin\CustomerDataPlugin
declare(strict_types=1);

namespace MyVendor\CustomerModule\Plugin;

class CustomerDataPlugin
{
    public function afterGetSectionData(
        \Magento\Customer\CustomerData\Customer $customer,
        array $result
    ): array {
        $result['customAttribute'] = $customer->getCustomer()->getCustomAttribute('customer_type');
        return $result;
    }
}
```

---

## 8. Loyalty/Rewards Integration

The `Magento_Reward` module integrates loyalty points with the customer account system.

### Reward Model

```php
// Magento\Reward\Model\Reward
declare(strict_types=1);

namespace Magento\Reward\Model;

class Reward extends \Magento\Framework\Model\AbstractExtensibleModel
{
    public function getCustomerId(): int
    {
        return $this->getData('customer_id');
    }

    public function getPointsBalance(): int
    {
        return $this->getData('points_balance');
    }

    public function getCurrencyAmount(): float
    {
        return $this->getData('currency_amount');
    }

    public function getPointsRate(): int
    {
        // Points-to-currency conversion rate
        return $this->getData('points_rate');
    }
}
```

### Reward Repository

```php
// Magento\Reward\Api\RewardRepositoryInterface
declare(strict_types=1);

namespace Magento\Reward\Api;

interface RewardRepositoryInterface
{
    public function getByCustomerId(int $customerId): \Magento\Reward\Api\Data\RewardInterface;
    public function save(\Magento\Reward\Api\Data\RewardInterface $reward): void;
    public function getNotificationBalance(int $customerId): int;
}
```

### Integration Points

| Component | Purpose |
|---------|---------|
| `customer_entity` | `reward_points_balance` decimal column |
| `reward_points_history` | Transaction history table |
| `customer_eav_attribute` | Points notification threshold |
| `Reward` model | Balance and currency tracking |
| `SectionsPoolInterface` | Customer data section updates |
| `AccountManagement` | Points sync on login/logout |
| `CheckoutData` | Points discount application |

### Earning Rules

Points are awarded through the reward system:

```php
// Magento\Reward\Model\Reward\Helper
declare(strict_types=1);

namespace Magento\Reward\Model\Reward;

class Helper
{
    public function getPointsForOrder(\Magento\Sales\Model\Order $order): int
    {
        // Calculate points based on order total and reward rate
        $rate = $this->getRewardRate();
        return (int) floor($order->getGrandTotal() * $rate);
    }

    public function getRewardRate(): float
    {
        // Default: 1 point per 1 currency unit
        // Configurable via admin
    }
}
```

### Balance Notification

Customers can set a threshold for email notifications:

```php
// Points balance notification
public function checkBalanceNotification(int $customerId): void
{
    $reward = $this->rewardRepository->getByCustomerId($customerId);
    $threshold = $this->getNotificationThreshold($customerId);

    if ($reward->getPointsBalance() >= $threshold) {
        $this->sendBalanceNotificationEmail($customerId);
    }
}
```

---

## Summary

Magento 2.4.8's Customer module provides a comprehensive account management system:

| Component | Key Classes | Purpose |
|-----------|-------------|---------|
| Customer EAV | `customer_entity`, `customer_eav_attribute` | Extensible customer data model |
| Address EAV | `customer_address_entity` | Separate address attributes |
| Registration | `CreatePost`, `AccountManagement` | Account creation flow |
| Session | `Magento\Customer\Model\Session` | Persistent authentication |
| Account Mgmt | `AccountManagementInterface` | Password reset, email change |
| Data Sections | `SectionsPoolInterface` | Optimized JS data loading |
| Custom Attrs | `customer_eav_attribute` setup | Adding new attributes |
| Rewards | `Magento\Reward\Model\Reward` | Loyalty points integration |

Understanding these components enables customization of registration flows, addition of custom attributes, integration with loyalty programs, and optimization of customer data handling in checkout and account pages.

(End of file - total 1423 lines)