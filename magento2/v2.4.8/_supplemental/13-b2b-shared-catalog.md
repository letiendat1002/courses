---
title: "17 - B2B & Shared Catalog"
description: "Deep dive into Magento 2.4.8 B2B architecture: real B2B module composition, Company/SharedCatalog data structures, DataPatch patterns, and CatalogPermissions integration"
tags:
  - magento2
  - b2b
  - shared-catalog
  - company
  - catalog-permissions
  - data-patch
  - requisition-list
rank: 13
pathways:
  - magento2-deep-dive
---

# B2B & Shared Catalog

Magento Commerce B2B is a separate metapackage (`magento/extension-b2b`) installed alongside the core Commerce platform. This article covers the correct module composition, Company/SharedCatalog data structures, DataPatch patterns, and the CatalogPermissions integration that powers category-level visibility restrictions.

> **Critical Note**: `b2b_enabler` is **not** a real module and does not exist in any version of the B2B metapackage. Any reference to it in articles, blog posts, or AI-generated content is incorrect. Delete it from any configuration you encounter.

---

## 1. Real B2B Module Composition

### The Wrong Approach (Fictional Module)

Many articles and AI-generated snippets reference a non-existent module:

```xml
<!-- WRONG - b2b_enabler does NOT exist -->
<sequence>
    <module name="Magento_B2b"/>
    <module name="b2b_enabler"/>
</sequence>
```

### The Correct B2B Module Architecture

The `magento/extension-b2b` metapackage installs these real modules:

| Module | Purpose |
|---|---|
| `Magento_Company` | Company accounts, roles, permissions, hierarchy |
| `Magento_SharedCatalog` | Per-company product/price visibility |
| `Magento_CompanyCredit` | Credit limit, balance, payment on account |
| `Magento_QuickOrder` | CSV/SKU-based bulk add-to-cart |
| `Magento_ReqQuote` | Requisition lists (saved shopping lists) |
| `Magento_CompanyPayment` | Payment method restrictions per company |
| `Magento_NegotiableQuote` | Quote negotiation workflow |
| `Magento_PurchaseOrder` | Purchase order approval workflow |

Enable the full B2B suite via CLI:

```bash
bin/magento module:enable \
    Magento_Company \
    Magento_SharedCatalog \
    Magento_CompanyCredit \
    Magento_QuickOrder \
    Magento_ReqQuote \
    Magento_CompanyPayment \
    Magento_NegotiableQuote \
    Magento_PurchaseOrder

bin/magento setup:upgrade
bin/magento cache:flush
```

Selectively enable features via Admin: **Stores → Configuration → B2B Features**.

---

## 2. Company Module — Data Structures

### Database Schema

`Magento_Company` owns these primary tables:

```
company
├── entity_id (PK)
├── status          -- 0=pending, 1=active, 2=rejected, 3=blocked
├── company_name
├── legal_name
├── company_email
├── street
├── city, region_id, country_id, postcode
├── telephone
├── customer_group_id   -- maps to price rules
├── sales_representative_id
├── super_user_id       -- company admin customer_id
└── extension_attributes (EAV overlay)

company_customer
├── customer_id (FK → customer_entity.entity_id)
├── company_id  (FK → company.entity_id)
├── status
└── approved_email (website-specific)

company_roles
├── role_id
├── company_id
├── role_name
└── company_acl_resource (role → permission mapping)

company_team
├── team_id
├── company_id
├── name
└── description

company_structure
├── structure_id
├── company_id
├── entity_id   -- customer_id or team_id
├── entity_type -- 0=team, 1=customer
└── parent_id   -- tree traversal
```

### Extending Company with EAV

To add a custom attribute to the company entity, use a DataPatch (not InstallData):

```php
// app/code/Vendor/Module/Setup/Patch/Data/AddCompanyIndustryAttribute.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\Patch\PatchRevertableInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Company\Api\Data\CompanyInterface;

class AddCompanyIndustryAttribute implements DataPatchInterface, PatchRevertableInterface
{
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup,
    ) {}

    /**
     * Return dependencies on OTHER DataPatch classes.
     * Never reference Setup\InstallData here — that pattern is pre-M2.2.
     */
    public static function getDependencies(): array
    {
        return [
            // Example: only declare if this patch MUST run after another
            // \Vendor\Module\Setup\Patch\Data\SomeOtherPatch::class,
        ];
    }

    public static function getAliases(): array
    {
        return [];
    }

    public function apply(): self
    {
        $this->moduleDataSetup->getConnection()->startSetup();

        $this->moduleDataSetup->getConnection()->addColumn(
            $this->moduleDataSetup->getTable('company'),
            'industry_code',
            [
                'type'    => \Magento\Framework\DB\Ddl\Table::TYPE_TEXT,
                'length'  => 64,
                'default' => null,
                'comment' => 'Industry classification code',
                'nullable' => true,
            ]
        );

        $this->moduleDataSetup->getConnection()->endSetup();

        return $this;
    }

    public function revert(): void
    {
        $this->moduleDataSetup->getConnection()->startSetup();
        $this->moduleDataSetup->getConnection()->dropColumn(
            $this->moduleDataSetup->getTable('company'),
            'industry_code'
        );
        $this->moduleDataSetup->getConnection()->endSetup();
    }
}
```

> **DataPatch `getDependencies()` rule**: Return an array of fully-qualified DataPatch class names (`SomeOtherPatch::class`). **Never** return `InstallData` or `UpgradeData` class references — those belong to the pre-M2.2 setup script system which DataPatches replaced entirely.

### Chained DataPatch Example

```php
// Patch B depends on Patch A running first
class SetDefaultIndustryCodes implements DataPatchInterface
{
    public static function getDependencies(): array
    {
        return [
            // Correct: reference DataPatch class, not Setup\InstallData
            \Vendor\Module\Setup\Patch\Data\AddCompanyIndustryAttribute::class,
        ];
    }

    public function apply(): self
    {
        // Safe to reference industry_code column — dependency guarantees it exists
        $this->moduleDataSetup->getConnection()->update(
            $this->moduleDataSetup->getTable('company'),
            ['industry_code' => 'GENERAL'],
            ['industry_code IS NULL']
        );

        return $this;
    }

    // ...
}
```

---

## 3. SharedCatalog — Architecture & Data Flow

### How SharedCatalog Connects to Company

A company is linked to a shared catalog via the `shared_catalog_company` join table. The company entity itself carries the `customer_group_id` that maps to pricing, but SharedCatalog manages catalog visibility separately:

```
shared_catalog
├── entity_id (PK)
├── name
├── description
├── customer_group_id  -- prices: maps to catalog_rule / tier_price scope
├── type               -- 0=custom, 1=public (default public catalog)
├── created_at
├── created_by
└── store_id

shared_catalog_company (join table)
├── sc_id      (FK → shared_catalog.entity_id)
└── company_id (FK → company.entity_id)

shared_catalog_product_item
├── entity_id
├── catalog_id (FK → shared_catalog.entity_id)
└── sku

shared_catalog_category_item
├── entity_id
├── catalog_id (FK → shared_catalog.entity_id)
└── category_id
```

The linkage flow:

```
Company ──────────────────── shared_catalog_company ── SharedCatalog
   │                                                        │
   │  customer_group_id (on company record)                 │  customer_group_id
   └─────── Tier Prices / Catalog Rules ───────────────────┘
                                                            │
                                         shared_catalog_product_item
                                         shared_catalog_category_item
                                                            │
                                              Magento_CatalogPermissions
                                         (category-level allow/deny per group)
```

### Programmatically Assigning a Company to a SharedCatalog

```php
// Inject via constructor DI
use Magento\SharedCatalog\Api\SharedCatalogManagementInterface;
use Magento\SharedCatalog\Api\CompanyManagementInterface;

class AssignCompanyToSharedCatalog
{
    public function __construct(
        private readonly CompanyManagementInterface $sharedCatalogCompanyManagement,
        private readonly SharedCatalogManagementInterface $sharedCatalogManagement,
    ) {}

    public function execute(int $companyId, int $sharedCatalogId): void
    {
        // Load the shared catalog
        $publicCatalog = $this->sharedCatalogManagement->getPublicCatalog();

        // Assign company — replaces any previous catalog assignment
        $this->sharedCatalogCompanyManagement->assignCompanies(
            $sharedCatalogId,
            [$companyId]
        );
    }
}
```

### SharedCatalog via REST API

```bash
# Create a custom shared catalog
POST /V1/sharedCatalog
{
    "sharedCatalog": {
        "name": "Wholesale Catalog",
        "description": "Wholesale pricing and product set",
        "type": 0,
        "customer_group_id": 4,
        "store_id": 0
    }
}

# Assign products to catalog (by catalog ID)
POST /V1/sharedCatalog/{catalogId}/assignProducts
{
    "products": [
        {"sku": "MH01-XS-Black"},
        {"sku": "WJ01-XS-Red"}
    ]
}

# Assign company to catalog
POST /V1/sharedCatalog/{catalogId}/assignCompanies
{
    "companies": [
        {"company_id": 3}
    ]
}
```

---

## 4. Category permissions via Magento_CatalogPermissions

SharedCatalog controls which products are visible per company. Category-level visibility restrictions are enforced by `Magento_CatalogPermissions`, which the SharedCatalog module integrates with automatically.

### How It Works

When a product is assigned/unassigned from a shared catalog, `Magento_SharedCatalog` writes records to `magento_catalogpermissions`:

```
magento_catalogpermissions
├── permission_id
├── category_id
├── website_id
├── customer_group_id   -- from the shared catalog's customer_group_id
├── grant_catalog_category_view   -- -1=deny, 0=inherit, 1=allow
├── grant_catalog_product_price   -- -1=deny, 0=inherit, 1=allow
└── grant_checkout_items          -- -1=deny, 0=inherit, 1=allow
```

The `customer_group_id` on the shared catalog record bridges catalog permissions to company assignments.

### Enabling CatalogPermissions in config.xml

```xml
<!-- app/code/Vendor/Module/etc/config.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Store:etc/config.xsd">
    <default>
        <catalog>
            <magento_catalogpermissions>
                <enabled>1</enabled>
                <grant_catalog_category_view>1</grant_catalog_category_view>
                <grant_catalog_product_price>1</grant_catalog_product_price>
                <grant_checkout_items>1</grant_checkout_items>
                <!-- Groups that always see everything (comma-separated IDs) -->
                <grant_catalog_category_view_groups></grant_catalog_category_view_groups>
            </magento_catalogpermissions>
        </catalog>
    </default>
</config>
```

### Custom Permission Observer

```php
// app/code/Vendor/Module/Observer/ApplyCustomCatalogPermission.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\CatalogPermissions\Model\Permission;

class ApplyCustomCatalogPermission implements ObserverInterface
{
    public function execute(Observer $observer): void
    {
        /** @var Permission $permission */
        $permission = $observer->getEvent()->getData('permission');
        $customerGroupId = (int) $observer->getEvent()->getData('customer_group_id');

        // Example: restrict price visibility for group 5
        if ($customerGroupId === 5) {
            $permission->setGrantCatalogProductPrice(Permission::PERMISSION_DENY);
        }
    }
}
```

```xml
<!-- etc/events.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="magento_catalogpermissions_apply_after">
        <observer name="vendor_module_apply_custom_catalog_permission"
                  instance="Vendor\Module\Observer\ApplyCustomCatalogPermission"/>
    </event>
</config>
```

---

## 5. CompanyCredit — Credit Limit & Balance

`Magento_CompanyCredit` tracks credit lines and transactions separately from standard payments:

```
company_credit
├── id
├── company_id
├── credit_limit     -- decimal(12,4), NULL = unlimited
├── balance          -- current outstanding balance (negative = owes money)
├── currency_code
└── exceed_limit     -- bool: allow overspend

company_credit_history
├── id
├── company_credit_id
├── user_id          -- admin who made the change
├── user_type        -- 1=admin, 2=customer
├── currency_amount
├── rate             -- exchange rate snapshot
├── currency_display
├── balance
├── credit_limit
├── available_credit
├── type             -- 1=allocated, 2=updated, 3=purchased, 4=reimbursed, 5=refunded, 6=reverted
└── comment
```

### Programmatic Credit Operations

```php
use Magento\CompanyCredit\Api\CreditLimitManagementInterface;
use Magento\CompanyCredit\Api\CreditLimitRepositoryInterface;
use Magento\CompanyCredit\Api\Data\CreditLimitInterfaceFactory;

class ManageCompanyCredit
{
    public function __construct(
        private readonly CreditLimitManagementInterface $creditLimitManagement,
        private readonly CreditLimitRepositoryInterface $creditLimitRepository,
        private readonly CreditLimitInterfaceFactory $creditLimitFactory,
    ) {}

    public function setCreditLimit(int $companyId, float $limit): void
    {
        $credit = $this->creditLimitManagement->getCreditByCompanyId($companyId);
        $credit->setCreditLimit($limit);
        $this->creditLimitRepository->save($credit);
    }
}
```

---

## 6. QuickOrder — Bulk Add-to-Cart

`Magento_QuickOrder` provides SKU-based or CSV-based add-to-cart. Extending it:

```php
// Plugin to validate SKUs against company's shared catalog
// app/code/Vendor/Module/Plugin/QuickOrder/SkuValidator.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\QuickOrder;

use Magento\QuickOrder\Model\Product\Suggest;

class SkuValidator
{
    public function afterGetSuggestData(
        Suggest $subject,
        array $result,
        string $query
    ): array {
        // Filter suggestions to only those visible in company's shared catalog
        // Implementation depends on session company resolution
        return array_filter($result, fn($item) => $this->isVisibleInCompanyCatalog($item['sku']));
    }

    private function isVisibleInCompanyCatalog(string $sku): bool
    {
        // Inject SharedCatalog product repository and check visibility
        return true; // placeholder
    }
}
```

---

## 7. RequisitionList (Magento_ReqQuote)

Requisition lists allow B2B buyers to save product lists for repeat purchasing:

```
negotiable_quote               -- NegotiableQuote (separate from ReqQuote)

requisition_list
├── entity_id
├── customer_id
├── store_id
├── name
├── description
└── updated_at

requisition_list_item
├── entity_id
├── requisition_list_id
├── sku
├── store_id
├── qty
└── options  -- JSON: selected options
```

### Service Contract Usage

```php
use Magento\RequisitionList\Api\RequisitionListRepositoryInterface;
use Magento\RequisitionList\Api\Data\RequisitionListInterfaceFactory;
use Magento\Framework\Api\SearchCriteriaBuilder;

class RequisitionListService
{
    public function __construct(
        private readonly RequisitionListRepositoryInterface $requisitionListRepository,
        private readonly RequisitionListInterfaceFactory $requisitionListFactory,
        private readonly SearchCriteriaBuilder $searchCriteriaBuilder,
    ) {}

    public function getListsForCustomer(int $customerId): array
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('customer_id', $customerId)
            ->create();

        return $this->requisitionListRepository->getList($searchCriteria)->getItems();
    }
}
```

---

## 8. CompanyPayment — Payment Method Restrictions

`Magento_CompanyPayment` restricts which payment methods are available per company:

```
company_payment
├── company_id
└── applicable_payment_method  -- 'all', 'specific', or 'b2b_payment_methods'
```

Configuration is stored in the `company_payment` table. A company can be restricted to:
- **All enabled methods** — default
- **B2B methods only** — Payment on Account, Purchase Order
- **Specific methods** — admin-selected list per company

### Reading Payment Config for a Company

```php
use Magento\CompanyPayment\Model\Payment\AvailabilityChecker;

// Injected into a payment method model to check company eligibility
class CustomPaymentAvailabilityPlugin
{
    public function __construct(
        private readonly AvailabilityChecker $availabilityChecker,
    ) {}

    public function afterIsAvailable(
        \Magento\Payment\Model\MethodInterface $subject,
        bool $result,
        ?\Magento\Quote\Api\Data\CartInterface $quote = null
    ): bool {
        if (!$result || $quote === null) {
            return $result;
        }

        return $this->availabilityChecker->isAvailableForCompany(
            $subject->getCode(),
            $quote
        );
    }
}
```

---

## 9. GraphQL Schema for B2B

### Correct Schema File Location

B2B modules expose their own `.graphqls` schema files under each module's `etc/` directory. Unlike XML configs, GraphQL schema files do **not** use XSD URN references — they are parsed by `Magento_GraphQl`'s schema stitching system.

The schema stitching entry point is:

```
vendor/magento/module-graph-ql/etc/graphql/di.xml
```

Each B2B module places its schema at:

```
vendor/magento/module-company/etc/schema.graphqls
vendor/magento/module-shared-catalog/etc/schema.graphqls
vendor/magento/module-negotiable-quote/etc/schema.graphqls
vendor/magento/module-purchase-order/etc/schema.graphqls
```

> **Wrong path**: `urn:magento:framework:graphql/etc/schema.graphqls` — this URN does **not** exist. GraphQL `.graphqls` files are not validated via XSD URNs. The `urn:magento:framework:*` URN namespace resolves to `lib/internal/Magento/Framework/` and there is no `graphql/etc/schema.graphqls` there. Remove any such reference from your code.

### Custom B2B GraphQL Schema

```graphql
# app/code/Vendor/Module/etc/schema.graphqls
type Query {
    companySharedCatalogInfo(company_id: Int! @doc(description: "Company entity ID")): CompanySharedCatalogOutput
        @resolver(class: "Vendor\\Module\\Model\\Resolver\\CompanySharedCatalogInfo")
        @doc(description: "Returns shared catalog assignment for a company")
        @cache(cacheable: false)
}

type CompanySharedCatalogOutput @doc(description: "Company to shared catalog mapping") {
    catalog_id: Int @doc(description: "Shared catalog entity ID")
    catalog_name: String @doc(description: "Shared catalog display name")
    customer_group_id: Int @doc(description: "Associated customer group ID")
}
```

The resolver:

```php
// app/code/Vendor/Module/Model/Resolver/CompanySharedCatalogInfo.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Resolver;

use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Exception\GraphQlInputException;
use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\SharedCatalog\Api\SharedCatalogRepositoryInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;

class CompanySharedCatalogInfo implements ResolverInterface
{
    public function __construct(
        private readonly SharedCatalogRepositoryInterface $sharedCatalogRepository,
        private readonly SearchCriteriaBuilder $searchCriteriaBuilder,
    ) {}

    public function resolve(
        Field $field,
        mixed $context,
        ResolveInfo $info,
        ?array $value = null,
        ?array $args = null
    ): array {
        $companyId = (int) ($args['company_id'] ?? 0);

        if ($companyId <= 0) {
            throw new GraphQlInputException(__('company_id must be a positive integer'));
        }

        // Locate the shared catalog assigned to this company
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('company_id', $companyId)
            ->create();

        $catalogs = $this->sharedCatalogRepository->getList($searchCriteria)->getItems();
        $catalog = reset($catalogs);

        if (!$catalog) {
            return ['catalog_id' => null, 'catalog_name' => null, 'customer_group_id' => null];
        }

        return [
            'catalog_id'        => (int) $catalog->getId(),
            'catalog_name'      => $catalog->getName(),
            'customer_group_id' => (int) $catalog->getCustomerGroupId(),
        ];
    }
}
```

---

## 10. B2B Company Status State Machine

Company accounts transition through statuses managed by `Magento\Company\Model\Company\Source\Status`:

```
pending (0)
    │
    ├─── admin approves ──► active (1)
    │
    └─── admin rejects ───► rejected (2)

active (1)
    │
    └─── admin blocks ────► blocked (3)

blocked (3)
    │
    └─── admin unblocks ──► active (1)
```

### Observing Company Status Changes

```php
// Register observer for: magento_company_save_after
// Event carries the company model with original and new status

class CompanyStatusChangeObserver implements ObserverInterface
{
    public function execute(Observer $observer): void
    {
        /** @var \Magento\Company\Api\Data\CompanyInterface $company */
        $company = $observer->getEvent()->getData('company');

        $origStatus = (int) $company->getOrigData('status');
        $newStatus  = (int) $company->getData('status');

        if ($origStatus === $newStatus) {
            return;
        }

        // Handle: pending → active
        if ($origStatus === 0 && $newStatus === 1) {
            $this->handleCompanyApproval($company);
        }

        // Handle: active → blocked
        if ($origStatus === 1 && $newStatus === 3) {
            $this->handleCompanyBlock($company);
        }
    }

    private function handleCompanyApproval(\Magento\Company\Api\Data\CompanyInterface $company): void
    {
        // Send custom welcome email, sync to ERP, etc.
    }

    private function handleCompanyBlock(\Magento\Company\Api\Data\CompanyInterface $company): void
    {
        // Revoke active sessions, notify account manager, etc.
    }
}
```

---

## 11. Common Pitfalls

### Pitfall 1: Referencing Non-Existent Modules

```xml
<!-- WRONG - none of these exist -->
<module name="b2b_enabler"/>
<module name="Magento_B2b"/>
<module name="Magento_B2BCore"/>
```

Use only the real module names from Section 1.

### Pitfall 2: getDependencies() Referencing InstallData

```php
// WRONG - pre-M2.2 pattern, not valid in DataPatches
public static function getDependencies(): array
{
    return [
        \Vendor\Module\Setup\InstallData::class,     // WRONG
        \Vendor\Module\Setup\UpgradeData::class,     // WRONG
    ];
}

// CORRECT - only other DataPatch classes
public static function getDependencies(): array
{
    return [
        \Vendor\Module\Setup\Patch\Data\CreateDefaultRoles::class,
    ];
}
```

### Pitfall 3: Wrong GraphQL URN in Schema Files

```graphql
# WRONG - this URN does not resolve to anything real
# urn:magento:framework:graphql/etc/schema.graphqls

# CORRECT - no URN reference needed; just write valid GraphQL SDL
type Query {
    myB2bQuery: MyType @resolver(class: "Vendor\\Module\\Model\\Resolver\\MyResolver")
}
```

### Pitfall 4: Assuming SharedCatalog Manages Pricing Directly

SharedCatalog does **not** store prices itself. It works by:
1. Assigning a `customer_group_id` to the shared catalog
2. Magento's tier price / catalog rule system uses that `customer_group_id` for pricing
3. Category visibility is enforced via `Magento_CatalogPermissions` records keyed by `customer_group_id`

To customize pricing per shared catalog, use catalog price rules or tier prices scoped to the catalog's `customer_group_id`.

### Pitfall 5: Forgetting Reindex After SharedCatalog Changes

SharedCatalog changes require reindexing CatalogPermissions:

```bash
bin/magento indexer:reindex catalogpermissions_category
bin/magento indexer:reindex catalogpermissions_product
bin/magento cache:flush
```

---

## Quick Reference

| Task | Service Contract / Class |
|---|---|
| Get company by ID | `Magento\Company\Api\CompanyRepositoryInterface::get()` |
| Get company for customer | `Magento\Company\Api\CompanyManagementInterface::getByCustomerId()` |
| Assign company to shared catalog | `Magento\SharedCatalog\Api\CompanyManagementInterface::assignCompanies()` |
| Get shared catalogs (search) | `Magento\SharedCatalog\Api\SharedCatalogRepositoryInterface::getList()` |
| Get/set credit limit | `Magento\CompanyCredit\Api\CreditLimitManagementInterface` |
| Get requisition lists | `Magento\RequisitionList\Api\RequisitionListRepositoryInterface` |
| Check payment availability | `Magento\CompanyPayment\Model\Payment\AvailabilityChecker` |
| Category permissions model | `Magento\CatalogPermissions\Model\Permission` |