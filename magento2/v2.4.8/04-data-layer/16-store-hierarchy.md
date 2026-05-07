---
title: "16 - Store Hierarchy & Scope Resolution"
description: "Magento 2.4.8 store hierarchy: Website → Store → Store View architecture, scope resolution, store context APIs, and multi-store data scoping patterns."
tags: magento2, store-hierarchy, website, store, store-view, scope-resolution, multistore, SCOPE_STORE, SCOPE_WEBSITE
rank: 16
pathways: [magento2-deep-dive]
see_also:
  - path: "02-eav-entity-model.md"
    description: "EAV attribute scoping — SCOPE_STORE vs SCOPE_WEBSITE directly ties to store hierarchy"
  - path: "15-event-observer.md"
    description: "Event system — store context events dispatch based on current store resolution"
  - path: "README.md"
    description: "Data layer overview — store scope is fundamental to how Magento scopes all data"
---

# Store Hierarchy & Scope Resolution

Magento's multi-store architecture is not an afterthought — it's a first-class feature woven into every layer: EAV attributes, configuration, pricing, CMS content, indexing, and more. Understanding the **Website → Store → Store View** hierarchy and how Magento resolves the "current" store context is prerequisite to working correctly with any of these systems.

This chapter fills the gap left by scattered references throughout this course, giving you a coherent mental model for store hierarchy before you encounter it in EAV, pricing, or indexing contexts.

---

## 1. The Three-Level Hierarchy

### Magento's Store Hierarchy

```
Website (WebsiteFactory / \Magento\Store\Model\Website)
  └── Store Group (GroupFactory / \Magento\Store\Model\Group)
        └── Store View (StoreFactory / \Magento\Store\Model\Store)
```

| Level | Model Class | Database Table | Key Property |
|-------|-------------|----------------|--------------|
| **Website** | `\Magento\Store\Model\Website` | `store_website` | `website_id`, `code`, `name`, `sort_order`, `is_default` |
| **Store (Group)** | `\Magento\Store\Model\Group` | `store_group` | `group_id`, `website_id`, `name`, `root_category_id` |
| **Store View** | `\Magento\Store\Model\Store` | `store` | `store_id`, `group_id`, `code`, `name`, `is_active`, `sort_order` |

### Physical Meaning

- **Website** — The top level. Often corresponds to a brand or legal entity. Can have its own domain, base URL, and customer pool. Customers are typically scoped to a **website** (not store view). `customer_entity.website_id` links a customer to a website.

- **Store (Group)** — Groups of store views sharing the same root category. A website might have one Store called "US Store" with views for English, Spanish, French. Or a website might have separate Stores for different product lines.

- **Store View** — The actual storefront the customer sees. This is where **language**, **locale**, **currency**, and **per-store configuration** apply. Product prices can be scoped to store view.

### How They Relate

```
US Website (magento.com)
  └── US Store Group
        ├── English Store View (en_US) ← product prices at THIS level
        ├── Spanish Store View (es_ES)
        └── French Store View (fr_FR)

EU Website (magento.eu)
  └── EU Store Group
        ├── German Store View (de_DE)
        ├── Italian Store View (it_IT)
        └── Polish Store View (pl_PL)
```

### Why Not Just Store Views?

A common beginner question: "Why do we need Websites and Stores if we just want different language views?"

The answer is **customer scoping** and **pricing control**:
- A **customer account** belongs to a **website**, not a store view
- **Price scope** (`SCOPE_WEBSITE` vs `SCOPE_STORE`) determines whether a price change applies to one store or all stores under a website
- **Inventory / Stock** is tied to **website** (via MSI sales channel)
- **Customer segments** and **promotions** filter by `website_id`

If you only need language differences with shared customers/prices/inventory, you can use a single Website with multiple store views. But when you need separate customer pools or independent pricing, you need separate websites.

---

## 2. The Store Resolver — How Magento Determines "Current Store"

### StoreResolverInterface

Magento determines the current store via `\Magento\Store\Api\StoreResolverInterface`:

```php
<?php
// vendor/magento/module-store/Model/StoreResolver.php
declare(strict_types=1);

namespace Magento\Store\Model;

class StoreResolver implements \Magento\Store\Api\StoreResolverInterface
{
    public function getCurrentStore(): \Magento\Store\Api\Data\StoreInterface
    {
        // Resolution order:
        // 1. `___store` request parameter (explicit store code override)
        // 2. Store cookie (previously selected store)
        // 3. Referer URL parsing (store from HTTP referer)
        // 4. Default store for website (fallback)
    }
}
```

### Resolution Priority (Precedence Order)

| Priority | Source | Example |
|----------|--------|---------|
| 1 | `___store` URL parameter | `https://example.com/?___store=de_DE` |
| 2 | Store cookie | `store=de_DE` in cookies |
| 3 | HTTP Referer | Visiting from a linked page with a specific store URL |
| 4 | Website default | Falls back to the website's default store view |

> **Important for debugging:** If your code behaves differently in different contexts, the store resolver is often why. The `___store` parameter overrides everything — it's used by the admin "Store Switcher" when previewing a different store.

### Accessing the Current Store

```php
<?php
// Inject \Magento\Store\Api\StoreResolverInterface or \Magento\Store\Model\StoreManagerInterface

public function __construct(
    \Magento\Store\Model\StoreManagerInterface $storeManager
) {
    $this->storeManager = $storeManager;
}

// Get current store ID
$storeId = $this->storeManager->getStore()->getId();

// Get current store code
$storeCode = $this->storeManager->getStore()->getCode();

// Get current website ID
$websiteId = $this->storeManager->getStore()->getWebsiteId();

// Get current store group ID (store / group)
$groupId = $this->storeManager->getStore()->getGroupId();

// Get website
$website = $this->storeManager->getWebsite();

// Get store group
$group = $this->storeManager->getGroup();
```

### The Store Cookie

When a customer selects a store view (e.g., switches to German), Magento stores `store` cookie:

```php
// The cookie name and lifetime are configurable
// Default: 'store' cookie, 3600 seconds (1 hour)

// To force a store programmatically:
$this->storeManager->setCurrentStore('de_DE');
```

> **Performance note:** The store cookie is read on every request. If you have hundreds of store views, consider the cookie size — each cookie is sent with every HTTP request to your domain.

---

## 3. ScopeConfigInterface — Reading Scoped Configuration

### What Is ScopeConfig?

`\Magento\Framework\App\Config\ScopeConfigInterface` is how Magento reads system configuration values that can vary by scope (global, website, store):

```php
<?php
public function __construct(
    \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig
) {
    $this->scopeConfig = $scopeConfig;
}

// Reading config — automatically resolves correct scope
$value = $this->scopeConfig->getValue(
    'carriers/fedex/path',       // Config path
    \Magento\Store\Model\ScopeInterface::SCOPE_STORE  // Scope type
);
```

### Scope Priority (How Config Values Are Resolved)

Magento merges configuration from multiple levels. The **most specific scope wins**:

```
1. Store View (most specific)
      ↓ (fallback)
   Store Group / Store
      ↓ (fallback)
   Website
      ↓ (fallback)
   Global (least specific)
```

If a config value is not set at Store View level, Magento falls back to Store → Website → Global.

### ScopeInterface Constants

```php
<?php
// Magento\Store\Model\ScopeInterface
const SCOPE_STORES   = 'stores';   // Store view level
const SCOPE_GROUPS   = 'groups';   // Store (group) level
const SCOPE_WEBSITES  = 'websites'; // Website level
const SCOPE_DEFAULT  = 'default';  // Global level
```

### Reading at Different Scopes

```php
<?php
// Store view scope (most specific)
$storeValue = $this->scopeConfig->getValue(
    'section/group/field',
    \Magento\Store\Model\ScopeInterface::SCOPE_STORES,
    $storeId  // e.g., 1 for admin store or actual store_id
);

// Website scope
$websiteValue = $this->scopeConfig->getValue(
    'section/group/field',
    \Magento\Store\Model\ScopeInterface::SCOPE_WEBSITES,
    $websiteId
);

// Store (group) scope
$groupValue = $this->scopeConfig->getValue(
    'section/group/field',
    \Magento\Store\Model\ScopeInterface::SCOPE_GROUPS,
    $groupId
);

// Default (global) scope
$defaultValue = $this->scopeConfig->getValue(
    'section/group/field',
    \Magento\Store\Model\ScopeInterface::SCOPE_DEFAULT
);
```

### Configuration Tables

Configuration values are stored in:

| Table | Scope |
|-------|-------|
| `core_config_data` | Store / Website / Default scope |

The `core_config_data` table structure:

```sql
scope       -- stores|groups|websites|default
scope_id    -- store_id|group_id|website_id|0
path        -- e.g., 'carriers/fedex/path'
value       -- the config value
```

- `scope = 'stores'` and `scope_id = 3` → value applies to store view with ID 3
- `scope = 'websites'` and `scope_id = 1` → value applies to website with ID 1
- `scope = 'default'` and `scope_id = 0` → global default

> **Common Pitfall:** When you edit a config value in admin, it saves to `core_config_data`. If you later modify the `env.php` or `config.php` with the same path, the database value takes precedence (unless the database is cleared). Always check `core_config_data` when debugging why a config value isn't changing.

### Admin Scope Switcher

In the Magento admin, you see the current scope in the top-left corner:

```
Current Scope: Main Website > US Store > English
```

When viewing a config section, the scope determines which level you're editing. This is why "Store View" scope has a dropdown to select which store view you're editing — each store view can have different config values.

---

## 4. EAV Attribute Scoping — How Store Hierarchy Affects Attributes

### The Three Scopes for EAV Attributes

Recall from the EAV Entity Model chapter that EAV attributes have three scope levels:

```php
<?php
// Magento\Catalog\Model\ResourceModel\Eav\Attribute
const SCOPE_GLOBAL   = 'global';
const SCOPE_WEBSITE  = 'website';
const SCOPE_STORE    = 'store';
```

### What Each Scope Means

| Scope | Effect | Example |
|-------|--------|---------|
| `SCOPE_GLOBAL` | Same value for all websites, stores, views | Product `sku` — same everywhere |
| `SCOPE_WEBSITE` | Per website, inherited by stores within | Customer `loyalty_tier` — website-specific |
| `SCOPE_STORE` | Per store view | Product `name`, `description`, `price` |

### How Scope Affects Product Prices

Product prices are `SCOPE_STORE` by default. This means:

```
Website: US Website
  └── Store: US Store
        ├── English View (en_US): price = $100
        └── Spanish View (es_ES): price = €90  ← different value

Website: EU Website
  └── Store: EU Store
        ├── German View (de_DE): price = €95  ← different website, different price
        └── French View (fr_FR): price = €92
```

The price you see depends on which store view you are currently viewing. This is why the `store_id` appears in `catalog_product_entity_decimal` value rows — the price is stored per `(entity_id, attribute_id, store_id)`.

### Customer Attributes — Website Scoped

Customer attributes use `SCOPE_WEBSITE` (not `SCOPE_STORE`). The reason: customer accounts are website-global, not store-view-specific.

```sql
customer_entity
├── entity_id
├── website_id     ← NOT store_id
├── group_id       ← customer group
├── email
└── created_at
```

The `website_id` column determines which website a customer belongs to. A customer created on `magento.com` (website_id = 1) cannot log into `magento.eu` (website_id = 2) unless the accounts are linked via a shared customer hierarchy (Adobe Commerce feature).

### EAV Value Tables with store_id

For product attributes with `SCOPE_STORE`:

```sql
catalog_product_entity_decimal
├── value_id (PK)
├── attribute_id (FK)
├── store_id (FK → store.store_id, 0 = default)
├── entity_id (FK → catalog_product_entity.entity_id)
└── value
```

`store_id = 0` means "use the global default value." When you load a product in store view 3, Magento:
1. Looks for `store_id = 3` in the value table
2. If no row exists, falls back to `store_id = 0` (global)
3. If still nothing, falls back to the attribute's default value

This fallback chain is why adding a store-specific attribute value requires an explicit row — it doesn't automatically inherit from the default.

---

## 5. CMS Content Scoping — Pages and Blocks

### PageRepository Respects Store Scope

As mentioned in the CMS & Extensions chapter, `PageRepository` load operations use store scope:

```php
<?php
// \Magento\Cms\Model\PageRepository
public function getById(int $pageId): \Magento\Cms\Api\Data\PageInterface
{
    $page = $this->pageFactory->create();
    $this->resourceModel->load($page, $pageId);
    // The page's store_assignments determine which views it appears in
}
```

### CMS Page Store Assignment

CMS pages are assigned to stores via `cms_page_store` (the `store_ids` column or separate link table depending on version):

```php
<?php
// Checking if a page is available in current store
$page = $this->pageRepository->getById($pageId);
$page->getStores();  // Returns store IDs this page is assigned to

if (!in_array($this->storeManager->getStore()->getId(), $page->getStores())) {
    // Page not available in current store view
    throw new \Magento\Framework\Exception\NoSuchEntityException();
}
```

### Store-Specific Content

The same page can have different content per store via the `content` field (which might store different values in separate rows or use a fallback mechanism):

- **Default content**: Shown when no store-specific override exists
- **Store-specific content**: Overrides the default for that store view

This is implemented via `store_id` in the page's underlying data — when editing in admin, the scope switcher determines which store's content you're modifying.

---

## 6. Theme and Layout Scoping

### Theme Assignment Hierarchy

Themes in Magento are assigned at three levels (as covered in the supplemental Theme Development chapter):

```php
// From _supplemental/23-theme-development.md
Theme assignment per website/store:
- **Global scope** (all websites)
- **Website scope** (specific website)
- **Store scope** (specific store view)
```

The hierarchy:
1. Check store view level for theme assignment
2. If none, fall back to store (group) level
3. If none, fall back to website level
4. If none, fall back to global default theme

### Layout XML Loading Per Store

Layout handles are loaded based on the current store context:

```php
<?php
// \Magento\Framework\View\LayoutBuilder
// Handles loading:
// <store_code>_store_view_handle>
// <store_code>_store_handle>
// <website_code>_website_handle>
// <default_handle>
// <all_handle>
```

The store code becomes part of the layout handle name, allowing store-specific layout XML modifications.

---

## 7. Indexing and Store Context

### Index Naming with Store Context

As noted in the Performance chapter, product flat index names include store context:

```
magento2_product_1                      ← Default (global) index
magento2_product_1_2                    ← Website 1, Store 2
```

Each store view that has prices or content differing from the default requires its own index copy. This is why MSI (Multi-Source Inventory) ties stocks to websites — the website becomes the indexing boundary.

### Price Index and website_id

Price indexing happens per website. The `catalog_product_index_price` table (and its variants like `catalog_product_index_price_cfg_product_website`) use `website_id` as a primary indexing dimension:

```sql
catalog_product_index_price
├── entity_id (PK)
├── website_id (PK)      ← Price index is per-website
├── customer_group_id
├── tax_class_id
├── price
├── final_price
└── min_price
```

When a product's price changes, Magento reindexes the price for each website the product is assigned to, not each store view.

---

## 8. Common Pitfalls and How to Avoid Them

### Pitfall 1: Filtering by store_id When You Mean website_id

```php
<?php
// BAD — filtering products by store_id when you want website-level filtering
$collection = $this->productCollectionFactory->create();
$collection->addFieldToFilter('store_id', $this->storeManager->getStore()->getId());

// BETTER — use website_id for website-level concerns like inventory/stock
$collection = $this->productCollectionFactory->create();
$collection->addWebsiteFilter($this->storeManager->getStore()->getWebsiteId());
```

**Rule:** Use `store_id` for content, display, and language concerns. Use `website_id` for customer accounts, pricing, inventory, and promotions.

### Pitfall 2: Forgetting the Store Switcher in Admin

When writing admin code that displays store-specific data, the admin might have a store scope active that differs from the preview store you expect. Always explicitly scope:

```php
<?php
// In admin controllers or blocks, be aware that
// $this->storeManager->getStore() returns the ADMIN store by default
// unless the admin is previewing a specific store view

// If you need the actual store context (e.g., for preview), check:
// 1. Is there a `___store` parameter in the request?
// 2. Is the admin in "Store Switcher" preview mode?

$previewStoreCode = $this->request->getParam('___store');
if ($previewStoreCode) {
    // Admin is previewing a specific store view
    $previewStore = $this->storeManager->getStore($previewStoreCode);
}
```

### Pitfall 3: Creating Store-Scoped EAV Rows Incorrectly

When you add a store-specific EAV value, you must explicitly INSERT a row with `store_id = X`. There is no automatic inheritance — the fallback happens at READ time, not WRITE time:

```php
<?php
// To set a price for a specific store view
$price = 99.99;
$product->setStoreId($storeId);  // Set BEFORE setting attribute value
$product->setPrice($price);      // This creates/updates the row for that store_id
$product->save();
```

If you omit `setStoreId()`, the value saves to `store_id = 0` (global default), not the current store.

### Pitfall 4: Hardcoding website_id

```php
<?php
// BAD — hardcoded website ID
$select->where('website_id = 1');

// GOOD — get website ID from store context
$websiteId = $this->storeManager->getStore()->getWebsiteId();
$select->where('website_id = ?', $websiteId);
```

Hardcoded website IDs break in environments with different store structures.

### Pitfall 5: Assuming Config Changes Apply Immediately

Configuration values are cached. After changing a scoped config value:

```bash
# Clear the config cache so the new value is read
bin/magento cache:clean config
```

The `config` cache stores compiled configuration from `core_config_data`. Without clearing it, your code continues to read stale values.

---

## 9. Entity Relationships — Quick Reference

### The Key Relationships

```
store_website
├── website_id (PK)
├── code
├── name
└── is_default

store_group
├── group_id (PK)
├── website_id (FK → store_website.website_id)
├── name
└── root_category_id

store
├── store_id (PK)
├── group_id (FK → store_group.group_id)
├── code
├── name
├── is_active
└── sort_order
```

### When to Use Each

| Need | Use |
|------|-----|
| Get current store ID | `$storeManager->getStore()->getId()` |
| Get current website ID | `$storeManager->getStore()->getWebsiteId()` |
| Get current store group | `$storeManager->getStore()->getGroupId()` |
| Iterate all websites | `$storeManager->getWebsites()` |
| Iterate all stores in website | `$website->getStores()` |
| Iterate all store views in group | `$group->getStores()` |
| Get default store for website | `$website->getDefaultStore()` |
| Check if store is active | `$store->isActive()` |

---

## Reading List

- [Magento Store Model](https://developer.adobe.com/commerce/php/architecture/topics/store-model/) — Official docs on store hierarchy
- [\Magento\Store\Model\StoreManagerInterface](https://github.com/magento/magento2/blob/2.4-develop/app/code/Magento/Store/Model/StoreManagerInterface.php) — Main store access interface
- [Scope Config](https://developer.adobe.com/commerce/php/development/components/configuration/) — Configuration scoping

---

## Edge Cases & Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| Price different from expected | Wrong store scope active | Check `___store` param and store cookie |
| Customer not found in store | Customer is on different website | Verify `customer_entity.website_id` matches current |
| Config change not applying | Config cache stale | `bin/magento cache:clean config` |
| Theme not changing | Theme assigned at wrong level | Check website-level vs store-level theme assignment |
| Product not showing in store | Store assignment missing | Check `catalog_product_website` table for website assignment |
| Admin showing wrong store data | Admin store context vs preview | Check for `___store` param in request |

---

## Common Mistakes to Avoid

1. ❌ Using `store_id` for pricing/inventory filtering → Use `website_id`
2. ❌ Forgetting `setStoreId()` before saving store-scoped EAV values → Value saves to wrong scope
3. ❌ Hardcoding `website_id = 1` → Breaks in multi-website environments
4. ❌ Not clearing config cache after config changes → Stale config values
5. ❌ Assuming `getStore()` in admin returns frontend store → Returns admin store by default
6. ❌ Forgetting that `store_id = 0` in EAV tables means "global default" → Always check for fallback
7. ❌ Not checking `is_active` on store before rendering → Inactive stores can cause routing issues

---

*Magento 2 Backend Developer Course — Topic 04 — Data Layer*