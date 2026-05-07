---
title: "26 - Store Hierarchy & Multi-Store Architecture (AUTHORITATIVE)"
description: "Canonical deep dive into Magento 2.4.8 store hierarchy: Website/Store/StoreView entities, scope resolution, context propagation, multi-store data patterns, and architectural implications for backend development."
tags: magento2, store-hierarchy, website, store, store-view, scope-resolution, multistore, store-context, SCOPE_STORE, SCOPE_WEBSITE
rank: 26
pathways: [magento2-deep-dive]
authoritative: true
---

# Store Hierarchy & Multi-Store Architecture (AUTHORITATIVE)

This is the authoritative supplemental article for the store hierarchy topic. It provides deeper architectural context, more code examples, and covers edge cases not fully addressed in the chapter version.

---

## 1. Store Hierarchy Architecture

### The Three Core Entities

Magento's multi-store system is built on three interrelated entities, each with a corresponding Model class and database table:

| Entity | Model Class | Table | Primary Key | Parent Reference |
|--------|-------------|-------|-------------|-----------------|
| **Website** | `\Magento\Store\Model\Website` | `store_website` | `website_id` | — (root) |
| **Store (Group)** | `\Magento\Store\Model\Group` | `store_group` | `group_id` | `website_id` |
| **Store View** | `\Magento\Store\Model\Store` | `store` | `store_id` | `group_id` |

The naming convention is historically inconsistent — "Store" in Magento admin UI usually means "Store View", while `\Magento\Store\Model\Group` is internally called "Store" (or "Store Group"). The `\Magento\Store\Model\Store` class represents a Store View. This naming confusion is a common source of errors in both documentation and code.

### Database Schema

```sql
-- store_website: top-level container
CREATE TABLE store_website (
    website_id     INT PRIMARY KEY AUTO_INCREMENT,
    code          VARCHAR(64) UNIQUE NOT NULL,
    name          VARCHAR(255),
    sort_order    INT DEFAULT 0,
    is_default    SMALLINT DEFAULT 0,  -- 1 = default website
    group_id      INT DEFAULT 0,      -- points to default store group
    default_store_id INT DEFAULT NULL  -- points to default store view
);

-- store_group: the "Store" in admin UI terminology
CREATE TABLE store_group (
    group_id      INT PRIMARY KEY AUTO_INCREMENT,
    website_id    INT NOT NULL,
    name          VARCHAR(255),
    root_category_id INT DEFAULT 0,   -- root category for this store's catalog
    default_store_id INT DEFAULT NULL  -- default store view
);

-- store: the "Store View" in admin UI terminology
CREATE TABLE store (
    store_id      INT PRIMARY KEY AUTO_INCREMENT,
    group_id      INT NOT NULL,
    code          VARCHAR(64) UNIQUE NOT NULL,
    name          VARCHAR(255),
    is_active     SMALLINT DEFAULT 0,
    sort_order    INT DEFAULT 0
);
```

### Physical vs Logical Hierarchy

```
store_website
├── website_id: 1
├── code: 'us_en'
├── name: 'US English Website'
└── is_default: 1

store_group
├── group_id: 1
├── website_id: 1
├── name: 'US Main Store'
└── root_category_id: 2

store
├── store_id: 1
├── group_id: 1
├── code: 'en_US'
└── name: 'English'
├── store_id: 2
├── group_id: 1
├── code: 'es_US'
└── name: 'Spanish'
```

### Entity Model Reference

```php
<?php
// Magento\Store\Model\Website
class Website extends \Magento\Framework\Model\AbstractExtensibleModel
{
    public function getId(): ?int;          // website_id
    public function getCode(): string;      // e.g., 'us_en'
    public function getName(): string;
    public function getDefaultStore(): \Magento\Store\Api\Data\StoreInterface;
    public function getStores(): \Magento\Store\Api\Data\StoreInterface[];
    public function getWebsiteId(): int;
}

// Magento\Store\Model\Group
class Group extends \Magento\Framework\Model\AbstractExtensibleModel
{
    public function getId(): ?int;          // group_id
    public function getName(): string;
    public function getWebsiteId(): int;
    public function getDefaultStore(): \Magento\Store\Api\Data\StoreInterface;
    public function getStores(): \Magento\Store\Api\Data\StoreInterface[];
    public function getRootCategoryId(): int;
}

// Magento\Store\Model\Store
class Store extends \Magento\Framework\Model\AbstractExtensibleModel
{
    public function getId(): ?int;          // store_id
    public function getCode(): string;      // e.g., 'en_US'
    public function getName(): string;
    public function getGroupId(): int;
    public function getWebsiteId(): int;
    public function isActive(): bool;
}
```

---

## 2. StoreResolverInterface — Complete Resolution Flow

### The Resolution Chain

The store resolver follows a strict precedence chain. Understanding this chain is critical for debugging why a particular store context is active:

```php
<?php
// Magento\Store\Model\StoreResolver::getCurrentStoreId() internal flow
public function getCurrentStoreId(): ?int
{
    // Step 1: Explicit ___store parameter (highest priority)
    // Used by: store switcher in admin, preview modes, debugging
    $storeCode = $this->request->getParam('___store');
    if ($storeCode && $this->getStoreIdByCode($storeCode)) {
        return $this->getStoreIdByCode($storeCode);
    }

    // Step 2: Store cookie (persistent user preference)
    // Cookie name: 'store', domain-scoped, 1-hour default lifetime
    $cookieStoreCode = $this->cookieManager->getCookie('store');
    if ($cookieStoreCode && $this->getStoreIdByCode($cookieStoreCode)) {
        return $this->getStoreIdByCode($cookieStoreCode);
    }

    // Step 3: HTTP referer domain matching
    // If user arrived from a store-specific URL, use that store
    // This is disabled by default in most installations
    $refererStoreCode = $this->getStoreCodeFromReferer();
    if ($refererStoreCode) {
        return $this->getStoreIdByCode($refererStoreCode);
    }

    // Step 4: Default store for current website (final fallback)
    // Uses website configuration: Stores > Settings > All Stores
    // The "Default Store View" setting on the website
    $website = $this->storeManager->getWebsite();
    $defaultStore = $website->getDefaultStore();
    return $defaultStore ? $defaultStore->getId() : null;
}
```

### The ___store Parameter

The `___store` parameter is the most powerful override. It bypasses all other resolution mechanisms:

```
https://yourstore.com/women/jackets?___store=de_DE
```

This would render the page using the German store view context, regardless of cookies or default settings. In the admin, the store switcher sets this parameter when previewing a specific store.

> **Security Note:** The `___store` parameter can expose store-specific data if misconfigured. Ensure your code properly validates store access when using this parameter.

### StoreManagerInterface — Accessing Store Context

The `StoreManagerInterface` is the primary interface for store access throughout Magento:

```php
<?php
// vendor/magento/module-store/Model/StoreManagerInterface.php

interface StoreManagerInterface
{
    public function getStore($storeId = null): \Magento\Store\Api\Data\StoreInterface;
    public function getWebsite($websiteId = null): \Magento\Store\Api\Data\WebsiteInterface;
    public function getGroup($groupId = null): \Magento\Store\Api\Data\GroupInterface;

    // Iterate collections
    public function getStores($withDefault = false): array;
    public function getWebsites($withDefault = false): array;
    public function getGroups($withDefault = false): array;

    // Current store info
    public function getDefaultStoreView(): \Magento\Store\Api\Data\StoreInterface;
    public function getCurrentStore(): \Magento\Store\Api\Data\StoreInterface;
}
```

### Practical Usage Patterns

```php
<?php
// In a class requiring store context:
public function __construct(
    \Magento\Store\Model\StoreManagerInterface $storeManager
) {
    $this->storeManager = $storeManager;
}

// Current store
$currentStore = $this->storeManager->getStore();
$storeId = $currentStore->getId();
$storeCode = $currentStore->getCode();
$websiteId = $currentStore->getWebsiteId();

// From store to website
$website = $currentStore->getWebsite();

// From website to all stores
$allStoresInWebsite = $website->getStores();

// Get all websites
$allWebsites = $this->storeManager->getWebsites();

// Default store for current website
$defaultStore = $currentStore->getWebsite()->getDefaultStore();

// All active stores (withDefault=false excludes disabled)
$activeStores = $this->storeManager->getStores(false);

// Specific store by ID
$store = $this->storeManager->getStore(3);

// Specific website by ID
$website = $this->storeManager->getWebsite(1);
```

### Admin Store Context vs Frontend Store Context

In the adminhtml area, the "current store" context behaves differently:

```php
<?php
// In admin context, $storeManager->getStore() returns:
// 1. If admin is NOT previewing a store: returns store_id = 0 (admin store)
// 2. If admin IS previewing a store: returns the previewed store

// Always check whether you're in admin preview mode
$isPreview = $this->request->getParam('___store');

// Admin store (store_id = 0, code = 'admin')
$adminStore = $this->storeManager->getStore(0);

// For store-specific preview in admin, use the ___store param
if ($previewStoreCode = $this->request->getParam('___store')) {
    $previewStore = $this->storeManager->getStore($previewStoreCode);
    // Use previewStore for context-sensitive operations
}
```

---

## 3. ScopeConfigInterface — Configuration Scoping Deep Dive

### How Configuration Merging Works

Magento's configuration system uses a layered merge approach. When you call `getValue('section/group/field', SCOPE_STORE)`, Magento:

1. Reads the default value from `etc/config.php` or `env.php`
2. Reads the default value from `etc/adminhtml/system.xml` (the declared default)
3. Overlays values from `core_config_data` at SCOPE_DEFAULT (scope='default', scope_id=0)
4. Overlays values from `core_config_data` at SCOPE_WEBSITES (scope='websites', scope_id=X)
5. Overlays values from `core_config_data` at SCOPE_GROUPS (scope='groups', scope_id=X)
6. Overlays values from `core_config_data` at SCOPE_STORES (scope='stores', scope_id=X)

Each layer overrides the previous. The most specific scope wins.

### core_config_data Table Structure

```sql
CREATE TABLE core_config_data (
    config_id     INT PRIMARY KEY AUTO_INCREMENT,
    scope          VARCHAR(32) NOT NULL DEFAULT 'default',
    scope_id       INT NOT NULL DEFAULT 0,
    path           VARCHAR(255) NOT NULL,
    value          TEXT
);

-- Examples:
-- Global default
INSERT INTO core_config_data (scope, scope_id, path, value)
VALUES ('default', 0, 'carrier/fedex/path', '/usr/bin/fedex');

-- Website-specific override
INSERT INTO core_config_data (scope, scope_id, path, value)
VALUES ('websites', 1, 'carrier/fedex/path', '/usr/bin/fedex-eu');

-- Store-view-specific override
INSERT INTO core_config_data (scope, scope_id, path, value)
VALUES ('stores', 3, 'general/locale/code', 'de_DE');
```

### Reading Configuration at Different Scopes

```php
<?php
class ConfigReader
{
    public function __construct(
        \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig,
        \Magento\Store\Model\StoreManagerInterface $storeManager
    ) {
        $this->scopeConfig = $scopeConfig;
        $this->storeManager = $storeManager;
    }

    public function getExamples(): void
    {
        // 1. Store view scope (most specific)
        $currentStoreId = $this->storeManager->getStore()->getId();
        $storeValue = $this->scopeConfig->getValue(
            'general/locale/code',
            \Magento\Store\Model\ScopeInterface::SCOPE_STORES,
            $currentStoreId
        );

        // 2. Website scope
        $websiteId = $this->storeManager->getStore()->getWebsiteId();
        $websiteValue = $this->scopeConfig->getValue(
            'general/locale/code',
            \Magento\Store\Model\ScopeInterface::SCOPE_WEBSITES,
            $websiteId
        );

        // 3. Store (group) scope
        $groupId = $this->storeManager->getStore()->getGroupId();
        $groupValue = $this->scopeConfig->getValue(
            'general/locale/code',
            \Magento\Store\Model\ScopeInterface::SCOPE_GROUPS,
            $groupId
        );

        // 4. Global default
        $defaultValue = $this->scopeConfig->getValue(
            'general/locale/code',
            \Magento\Store\Model\ScopeInterface::SCOPE_DEFAULT
        );
    }
}
```

### ScopeInterface Constants

```php
<?php
// Magento\Store\Model\ScopeInterface
interface ScopeInterface
{
    const SCOPE_STORES  = 'stores';   // Store view level
    const SCOPE_GROUPS  = 'groups';   // Store (group) level
    const SCOPE_WEBSITES = 'websites'; // Website level
    const SCOPE_DEFAULT = 'default';  // Global level
}
```

---

## 4. EAV Attribute Scoping — Complete Reference

### Scope Definitions for EAV

```php
<?php
// Magento\Catalog\Model\ResourceModel\Eav\Attribute
class Attribute extends \Magento\Eav\Model\Entity\Attribute
{
    const SCOPE_GLOBAL  = 'global';
    const SCOPE_WEBSITE = 'website';
    const SCOPE_STORE   = 'store';

    // Usage in setup:
    'global' => \Magento\Catalog\Model\ResourceModel\Eav\Attribute::SCOPE_GLOBAL,
    'website' => \Magento\Catalog\Model\ResourceModel\Eav\Attribute::SCOPE_WEBSITE,
    'store' => \Magento\Catalog\Model\ResourceModel\Eav\Attribute::SCOPE_STORE,
}
```

### How Scope Affects Data Storage

| Entity Type | Scope | store_id Column | website_id Column | Notes |
|-------------|-------|-----------------|-------------------|-------|
| Product | `SCOPE_GLOBAL` | 0 | — | Same value everywhere |
| Product | `SCOPE_WEBSITE` | — | Used in filter | Website-level fallback |
| Product | `SCOPE_STORE` | Used in value rows | — | Per-store values with fallback to 0 |
| Customer | `SCOPE_WEBSITE` | — | `customer_entity.website_id` | Customers not scoped to store view |
| Category | `SCOPE_GLOBAL` | — | — | Categories are always global |

### Product Price Scoping — The Complete Fallback Chain

Product prices use `SCOPE_STORE`. When loading a price for store view 5:

```php
<?php
// Magento\Catalog\Model\Product::getPrice()
public function getPrice($priceCode = null)
{
    // 1. Try store_id = 5 (current store view)
    // SELECT value FROM catalog_product_entity_decimal
    // WHERE entity_id = ? AND attribute_id = ? AND store_id = 5

    // 2. If no row found, try store_id = 0 (global default)
    // SELECT value FROM catalog_product_entity_decimal
    // WHERE entity_id = ? AND attribute_id = ? AND store_id = 0

    // 3. If still nothing, use attribute's default_value in eav_attribute
}
```

This means:
- If you set a price at `store_id = 0` only, all store views see the same price
- If you set a price at `store_id = 5`, ONLY store view 5 sees that price
- Other store views still fall back to `store_id = 0`
- The price at `store_id = 5` does NOT "inherit" from `store_id = 0` — it OVERRIDES it for that specific store

### Setting Store-Scoped Attribute Values

```php
<?php
// Correct: set store context before setting attribute
$product = $this->productRepository->get($sku);

$storeId = 5;  // German store view
$product->setStoreId($storeId);
$product->setPrice(99.99);  // Saves for store_id = 5
$product->save();

// After this, for store_id = 5: price = 99.99
// For all other store views: falls back to store_id = 0 price

// WRONG: omitting setStoreId
$product->setPrice(89.99);  // Saves for store_id = 0 (global)
$product->save();

// Now ALL store views see 89.99 unless they have their own explicit price
```

### Customer website_id Scoping

Customer attributes are scoped to **website**, not store view:

```sql
customer_entity
├── entity_id
├── email
├── website_id        ← Customer belongs to this website
├── group_id          ← Customer group (retailer, wholesale, etc.)
├── created_at
└── ...other fields...

customer_entity_varchar
├── value_id
├── attribute_id
├── entity_id
├── value             ← NO store_id for customers
```

Why no `store_id` for customers? Because a customer account is shared across all store views of the same website. If you log into the US website, you have the same customer data across English and Spanish store views.

### The website_id Column in EAV Value Tables

Customer EAV value tables (e.g., `customer_entity_int`) do NOT have `store_id` — they have `website_id` (or in practice, the website context is resolved differently). The `website_id` column in `customer_entity` is the scoping key.

```php
<?php
// Customer attribute loading respects website scope
$customer->getData('loyalty_tier');
// Internally resolves:
// 1. Get customer's website_id (from customer_entity.website_id)
// 2. Load customer_entity_int where entity_id = customer_id AND attribute_id = ? AND website_id = customer's website_id
```

---

## 5. Index Structure with Store Context

### Product Flat Index Naming

Product flat indices are created per website, not per store view:

```
catalog_product_flat_1           ← Website 1 default
catalog_product_flat_1_store_2   ← Website 1, Store View 2 (if scoped)
```

The `1` in `magento2_product_1` is the `website_id`. Store view-specific indexing only happens when the store view has content that differs from the website default (e.g., store-specific name or description).

### Price Index Structure

```sql
catalog_product_index_price
├── entity_id       (PK)
├── website_id     (PK) ← Indexed per website
├── customer_group_id (PK)
├── tax_class_id
├── price
├── final_price
├── min_price
└── max_price

catalog_product_index_price_cfg_product_website
├── entity_id
├── website_id      ← Ties configurable product price to websites
└── dropdown_label
```

### Catalog Category Flat — No Store Scoping

Categories are `SCOPE_GLOBAL` only. The `catalog_category_flat_store_[store_id]` tables exist for performance (denormalized flat tables per store) but the category entity itself is global:

```php
<?php
// Category attribute scope
$category->getResource()->getAttribute('description')->getScope();
// Returns: 'global' — categories are NOT store-scoped
```

This means you cannot have different category descriptions per store view. If you need store-specific category content, you use either:
1. Store-specific CMS blocks embedded in category pages
2. Category-friendly attributes with custom logic (not standard EAV)

---

## 6. Theme Resolution with Store Hierarchy

### Theme Assignment Resolution Order

```php
<?php
// \Magento\Theme\Model\View\Design
public function getDesignTheme(): \Magento\Framework\View\Design\ThemeInterface
{
    // Resolution order (most specific wins):
    // 1. Store view theme (explicitly set on store)
    // 2. Store group (default store for the group)
    // 3. Website theme (set on website)
    // 4. Default theme from config (global)
}
```

From the supplemental theme development chapter:

```
Theme assignment per website/store:
- **Global scope** (all websites)      ← configuration default
- **Website scope** (specific website) ← explicit per-website setting
- **Store scope** (specific store view) ← explicit per-store-view setting
```

### Layout Handle Generation with Store Code

```php
<?php
// \Magento\Framework\View\Layout::getNodeHandle
// Generates handles based on store context:

// 1. Store-specific handle
"<store_code>_<store_id>_store_view"  // e.g., "en_US_1_store_view"

// 2. Store (group) handle
"<store_code>_store"                   // e.g., "us_store"

// 3. Website handle
"<website_code>_website"               // e.g., "us_en_website"

// 4. Default (all stores)
"default"
"cms_page"  // entity-specific
```

This means you can add layout XML modifications that apply only to a specific store view:

```xml
<!-- Enters effect only for en_US store view -->
<en_US_1_store_view>
    <referenceBlock name="logo">
        <arguments>
            <argument name="logo_file" xsi:type="string">images/logo-en.svg</argument>
        </arguments>
    </referenceBlock>
</en_US_1_store_view>
```

---

## 7. CMS Page and Block Scoping

### Page Store Assignment Model

CMS pages are assigned to stores via the `store` link table (or `store_ids` column depending on version):

```sql
cms_page
├── page_id
├── title
├── identifier
└── content

cms_page_store  -- link table
├── page_id (FK)
└── store_id (FK)
```

The `cms_page_store` table defines which store views can display a given page. When loading a page, Magento checks if the current store ID is in the assigned store list.

### PageRepository Store Context Loading

```php
<?php
// \Magento\Cms\Model\PageRepository::getById
public function getById(int $pageId): \Magento\Cms\Api\Data\PageInterface
{
    $page = $this->pageFactory->create();
    $this->resourceModel->load($page, $pageId);

    // getStores() returns store IDs this page is assigned to
    $assignedStores = $page->getStores();

    // Current store context
    $currentStoreId = $this->storeManager->getStore()->getId();

    if (!in_array($currentStoreId, $assignedStores) && !in_array(0, $assignedStores)) {
        throw new NoSuchEntityException();
    }

    return $page;
}
```

`store_id = 0` in the assignment means "available to all stores."

### Block Store Assignment

CMS blocks follow the same pattern — they can be assigned globally or to specific stores.

---

## 8. MSI / Inventory Sales Channel (Website as Stock Boundary)

### Stock = Website-Scoped

As covered in MSI Architecture, stocks are tied to websites (sales channels):

```php
<?php
// \Magento\InventorySalesApi\Api\GetAssignedStockByWebsiteInterface
public function execute(string $websiteCode): StockInterface;
```

The `inventory_stock_sales_channel` table links stocks to websites:

```sql
inventory_stock_sales_channel
├── stock_id (FK → inventory_stock.id)
├── type     VARCHAR(64)  -- 'website'
└── code     VARCHAR(64)  -- website code
```

This means:
- Stock availability is per-website, not per-store-view
- When a customer adds to cart, the stock is resolved from the website
- A source that is "primary" for one website can be "secondary" for another

---

## 9. Complete API Reference

### StoreManagerInterface Methods

```php
<?php
interface StoreManagerInterface
{
    // Get store by ID or current store
    public function getStore($storeId = null): \Magento\Store\Api\Data\StoreInterface;

    // Get website by ID or website of current store
    public function getWebsite($websiteId = null): \Magento\Store\Api\Data\WebsiteInterface;

    // Get store group by ID or group of current store
    public function getGroup($groupId = null): \Magento\Store\Api\Data\GroupInterface;

    // Collections
    public function getStores($withDefault = false): array;
    public function getWebsites($withDefault = false): array;
    public function getGroups($withDefault = false): array;

    // Default store views
    public function getDefaultStoreView(): \Magento\Store\Api\Data\StoreInterface;
    public function getCurrentStore(): \Magento\Store\Api\Data\StoreInterface;

    // Store code to ID resolution
    public function getStoreByCode(string $code): ?int;
    public function getWebsiteByCode(string $code): ?int;
    public function getGroupByCode(string $code): ?int;
}
```

### StoreInterface Key Methods

```php
<?php
interface StoreInterface extends \Magento\Framework\DataObject\IdentityInterface
{
    public function getId(): ?int;
    public function getCode(): string;
    public function getName(): string;
    public function getGroupId(): int;
    public function getWebsiteId(): int;
    public function isActive(): bool;
    public function getSortOrder(): int;
}
```

### ScopeInterface Constants and Usage

```php
<?php
// Usage with ScopeConfigInterface
$value = $this->scopeConfig->getValue(
    'path/to/config',
    \Magento\Store\Model\ScopeInterface::SCOPE_STORES,   // 'stores'
    $storeId                                                    // int
);

$value = $this->scopeConfig->getValue(
    'path/to/config',
    \Magento\Store\Model\ScopeInterface::SCOPE_WEBSITES,  // 'websites'
    $websiteId                                                  // int
);

$value = $this->scopeConfig->getValue(
    'path/to/config',
    \Magento\Store\Model\ScopeInterface::SCOPE_GROUPS,     // 'groups'
    $groupId                                                    // int
);

$value = $this->scopeConfig->getValue(
    'path/to/config',
    \Magento\Store\Model\ScopeInterface::SCOPE_DEFAULT     // 'default'
);
```

---

## 10. Common Mistakes and Debugging

### Mistake 1: Confusing store_id and website_id in Queries

```php
<?php
// WRONG: using store_id where website_id is needed
$collection = $productCollection->create()
    ->addFieldToFilter('store_id', $currentStore->getId());

// RIGHT: for inventory/pricing, use website_id
$collection = $productCollection->create()
    ->addWebsiteFilter($currentStore->getWebsiteId());

// RIGHT: for content/display, use store_id
$collection = $cmsPageCollection->create()
    ->addStoreFilter($currentStore->getId());
```

**Decision matrix:**
- Pricing, inventory, customer accounts → `website_id`
- Content, locale, display → `store_id`

### Mistake 2: Not Setting Store Before Saving EAV Attributes

```php
<?php
// WRONG: default save goes to store_id = 0 (global)
$product->setPrice(79.99);
$product->save();

// RIGHT: explicitly set store context
$product->setStoreId($storeId);
$product->setPrice(79.99);
$product->save();
```

### Mistake 3: Hardcoding Website IDs

```php
<?php
// WRONG
$select->where('website_id = 1');

// RIGHT
$select->where('website_id = ?', $storeManager->getStore()->getWebsiteId());
```

### Mistake 4: Admin Store Context Assumption

```php
<?php
// In admin controllers, $storeManager->getStore() returns admin store (id=0)
// unless the admin is in store preview mode

// WRONG: assuming admin store context is useful for preview
$storeId = $this->storeManager->getStore()->getId();  // Returns 0 in admin

// RIGHT: check for preview mode
$previewStoreCode = $this->request->getParam('___store');
if ($previewStoreCode) {
    $previewStore = $this->storeManager->getStore($previewStoreCode);
}
```

### Mistake 5: Config Cache Staleness

```bash
# Every time you modify core_config_data, clear the config cache
bin/magento cache:clean config

# Or flush all caches during development
bin/magento cache:flush full
```

### Debugging Store Context

```php
<?php
// Quick diagnostic in any template or controller
/** @var \Magento\Store\Model\Store $store */
$store = $this->storeManager->getStore();

echo "Store ID: " . $store->getId() . "\n";
echo "Store Code: " . $store->getCode() . "\n";
echo "Store Name: " . $store->getName() . "\n";
echo "Group ID: " . $store->getGroupId() . "\n";
echo "Website ID: " . $store->getWebsiteId() . "\n";
echo "Is Active: " . ($store->isActive() ? 'yes' : 'no') . "\n";
```

---

## 11. Migration Notes for Adobe Commerce

### Shared Customer Accounts (B2B Feature)

In Adobe Commerce B2B, customers can belong to a company hierarchy that spans websites. This is handled via `company_id` linkage, not shared customer records. Each website still has its own `customer_entity` rows.

### Visual Merchandiser (EE Only)

Adobe Commerce's Visual Merchandiser allows category rules that can vary by store view. This adds additional scope considerations beyond the standard EAV model.

---

## Reading List

- [\Magento\Store\Model\StoreManager](https://github.com/magento/magento2/blob/2.4-develop/app/code/Magento/Store/Model/StoreManager.php) — Store manager implementation
- [\Magento\Framework\App\Config\ScopeConfigInterface](https://github.com/magento/magento2/blob/2.4-develop/lib/internal/Magento/Framework/App/Config/ScopeConfigInterface.php) — Config reading interface
- [Multi-Store Documentation](https://developer.adobe.com/commerce/php/architecture/topics/store-model/) — Official store model docs
- [MSI Architecture](./17-elasticsearch-search.md) — Sales channel / website relationship

---

*Magento 2 Backend Developer Course — Supplemental Article 26 — Authoritative*