---
title: "32 - B2B Advanced Features"
description: "Deep dive into Magento 2.4.8 B2B suite: Company accounts, Shared Catalogs, Quick Order, negotiable quotes, procurement, and B2B permission management."
tags: magento2, b2b, company-accounts, shared-catalog, negotiable-quote, quick-order, procurement, b2b-permissions
rank: 14
pathways: [magento2-deep-dive]
---

# 32 — B2B Advanced Features

Magento 2.4.8 includes a comprehensive B2B suite delivered as the `magento/extension-b2b` metapackage. This article covers every major B2B module: Company accounts with role-based permissions, Shared Catalogs with per-company pricing, Quick Order pad for bulk purchasing, Requisition Lists for saved carts, Negotiable Quotes with full negotiation workflows, Company Credit for credit-limit management, and the REST/GraphQL APIs that tie it all together. Throughout, real interface contracts and database schemas are shown so you can extend and integrate the B2B stack confidently.

> **Prerequisite**: Enable B2B modules before any development:
> ```bash
> bin/magento module:enable \
>     Magento_Company Magento_SharedCatalog Magento_CompanyCredit \
>     Magento_QuickOrder Magento_ReqQuote Magento_NegotiableQuote \
>     Magento_PurchaseOrder Magento_CompanyPayment
> bin/magento setup:upgrade
> bin/magento cache:flush
> ```

---

## 1. B2B Module Overview

### 1.1 Full Module Map

The B2B metapackage installs eight production modules. Each occupies a distinct domain:

| Module | Domain | Key Entity |
|---|---|---|
| `Magento_Company` | Company structure & users | `company`, `company_roles`, `company_structure` |
| `Magento_SharedCatalog` | Catalog visibility & pricing | `shared_catalog`, `shared_catalog_product_item` |
| `Magento_CompanyCredit` | Credit lines & transactions | `company_credit`, `company_credit_history` |
| `Magento_QuickOrder` | Bulk SKU-based ordering | CSV upload + SKU pad |
| `Magento_ReqQuote` | Saved shopping lists | `requisition_list`, `requisition_list_item` |
| `Magento_NegotiableQuote` | Quote negotiation workflow | `negotiable_quote`, `negotiable_quote_history` |
| `Magento_PurchaseOrder` | PO approval workflow | `purchase_order`, `purchase_order_grid` |
| `Magento_CompanyPayment` | Payment method restrictions | `company_payment` |

### 1.2 Feature Flag Configuration

All B2B features are individually toggled via **Stores → Configuration → B2B Features**:

```bash
# CLI equivalent for enabling features
bin/magento module:enable \
    Magento_Company Magento_SharedCatalog Magento_CompanyCredit \
    Magento_QuickOrder Magento_ReqQuote Magento_NegotiableQuote \
    Magento_PurchaseOrder Magento_CompanyPayment
# B2B modules are enabled via module:enable and sequenced in app/etc/config.php
# Individual feature toggles are per-module in etc/config.xml
```

These config paths map to `etc/config.xml` defaults in each module. The values are stored in `core_config_data` and checked at runtime by the frontend block visibility system.

---

## 2. Company Accounts

### 2.1 Core Entity Relationships

A **Company** is the parent B2B entity. It owns a hierarchy of **Users** and **Teams**, and it carries a **Role** that controls what each user can do within the company.

```
company  (1)───────────────────(N)  company_roles
  │                                  └─── role_id, company_id, role_name
  │
  ├──────────────────(1)─────── customer_entity
  │                              (super_user_id = company admin)
  │
  ├──────────────────(N)─────── company_customer
  │                              (company members, not admin)
  │
  ├──────────────────(N)─────── company_structure
  │                              (tree: customers + teams)
  │
  └──────────────────(1)─────── customer_group
                                    (maps to pricing / catalog rules)
```

### 2.2 Company Repository

```php
// Magento\Company\Api\CompanyRepositoryInterface
use Magento\Company\Api\CompanyRepositoryInterface;
use Magento\Company\Api\Data\CompanyInterface;
use Magento\Framework\Api\SearchCriteriaInterface;

class CompanyService
{
    public function __construct(
        private readonly CompanyRepositoryInterface $companyRepository,
    ) {}

    public function getActiveCompanies(): array
    {
        $searchCriteria = $this->buildSearchCriteria(['status' => 1]);
        return $this->companyRepository->getList($searchCriteria)->getItems();
    }

    public function getCompanyById(int $companyId): CompanyInterface
    {
        return $this->companyRepository->get($companyId);
    }

    private function buildSearchCriteria(array $filters): SearchCriteriaInterface
    {
        // Use SearchCriteriaBuilder to construct filter groups
    }
}
```

### 2.3 Company Status State Machine

Companies move through explicit statuses tracked in `company.status`:

| Status | Constant | Meaning |
|---|---|---|
| 0 | `STATUS_PENDING` | Awaiting admin approval |
| 1 | `STATUS_ACTIVE` | Approved and active |
| 2 | `STATUS_REJECTED` | Admin rejected the application |
| 3 | `STATUS_BLOCKED` | Admin blocked after being active |

Transitions are triggered in the admin panel only. Observing transitions:

```php
// etc/events.xml (global)
<event name="magento_company_save_after">
    <observer name="vendor_company_status_observer"
             instance="Vendor\Module\Observer\CompanyStatusChange"/>
</event>
```

```php
// Observer/CompanyStatusChange.php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Company\Api\Data\CompanyInterface;

class CompanyStatusChange implements ObserverInterface
{
    public function execute(Observer $observer): void
    {
        /** @var CompanyInterface $company */
        $company = $observer->getEvent()->getData('company');

        $originalStatus = (int) $company->getOrigData('status');
        $newStatus = (int) $company->getData('status');

        if ($originalStatus === $newStatus) {
            return;
        }

        match ($newStatus) {
            1 => $this->onApproved($company),
            2 => $this->onRejected($company),
            3 => $this->onBlocked($company),
            default => null,
        };
    }
}
```

### 2.4 Company Admin vs Company Users

The **Company Admin** is a `customer_entity` record linked to the company via `company.super_user_id`. All other company members are linked through `company_customer`:

```php
// Getting the company admin customer
use Magento\Company\Api\CompanyManagementInterface;

$company = $this->companyRepository->get($companyId);
$adminCustomerId = $company->getSuperUserId();
$adminCustomer = $this->customerRepository->getById($adminCustomerId);

// Getting company members
use Magento\Company\Api\CustomerManagementInterface;

$members = $this->customerManagementInterface->getCustomersByCompanyId($companyId);
// Returns array of Magento\Company\Api\Data\CustomerInterface
```

### 2.5 Team Management

Teams group company users for organisational purposes:

```php
// Magento\Company\Api\TeamRepositoryInterface
use Magento\Company\Api\Data\TeamInterfaceFactory;

class TeamService
{
    public function __construct(
        private readonly \Magento\Company\Api\TeamRepositoryInterface $teamRepository,
        private readonly TeamInterfaceFactory $teamFactory,
    ) {}

    public function createTeam(int $companyId, string $name, string $description): \Magento\Company\Api\Data\TeamInterface
    {
        $team = $this->teamFactory->create();
        $team->setCompanyId($companyId);
        $team->setName($name);
        $team->setDescription($description);

        return $this->teamRepository->save($team);
    }
}
```

---

## 3. Shared Catalogs

### 3.1 Public vs Custom Catalogs

Every installation has one immutable **Public Catalog** (type=1). Admins create additional **Custom Catalogs** (type=0) and assign companies to them:

```php
// Creating a custom shared catalog
use Magento\SharedCatalog\Api\SharedCatalogRepositoryInterface;
use Magento\SharedCatalog\Api\Data\SharedCatalogInterfaceFactory;

public function createCustomCatalog(
    int $customerGroupId,
    string $name,
    string $description
): SharedCatalogInterface {
    $catalog = $this->sharedCatalogInterfaceFactory->create();
    $catalog->setName($name);
    $catalog->setDescription($description);
    $catalog->setType(0); // 0 = custom, 1 = public (read-only)
    $catalog->setCustomerGroupId($customerGroupId);
    $catalog->setStoreId(0);

    return $this->sharedCatalogRepository->save($catalog);
}
```

### 3.2 Per-Company Pricing via Customer Group

SharedCatalog ties pricing to Magento's native `customer_group_id` mechanism. Assign a company a custom `customer_group_id`, then apply catalog price rules or tier prices scoped to that group:

```
Company A ── customer_group_id = 10 ── Tier price: 10% off on SKU-001
Company B ── customer_group_id = 11 ── Tier price: 15% off on SKU-001
```

The `shared_catalog` record stores the `customer_group_id` and all standard Magento tier/catalog price rules automatically apply at that scope.

### 3.3 Product and Category Assignment

```php
// Assigning products to a shared catalog
use Magento\SharedCatalog\Api\ProductManagementInterface;
use Magento\SharedCatalog\Api\SharedCatalogInterface;

public function assignProductsToCatalog(
    SharedCatalogInterface $catalog,
    array $skus
): void {
    $this->productManagement->assignProducts($catalog->getId(), $skus);
}

// Assigning categories
use Magento\SharedCatalog\Api\CategoryManagementInterface;

public function assignCategories(int $catalogId, array $categoryIds): void
{
    $this->categoryManagement->assignCategories($catalogId, $categoryIds);
}
```

### 3.4 SharedCatalog and CatalogPermissions

`Magento_CatalogPermissions` integrates automatically with SharedCatalog. When products are assigned to a shared catalog, `catalogpermissions` records are written automatically:

```
magento_catalogpermissions
├── permission_id
├── category_id
├── website_id
├── customer_group_id     ← from shared_catalog.customer_group_id
├── grant_catalog_category_view   ← 1 = allow
├── grant_catalog_product_price  ← 1 = allow
└── grant_checkout_items          ← 1 = allow
```

This means category-level visibility is enforced per customer group — the same mechanism that powers `Magento_Customer` group restrictions.

---

## 4. Quick Order Pad

### 4.1 How the Quick Order Pad Works

The Quick Order Pad allows buyers to add products by SKU without browsing the catalog. It has three entry modes:

1. **Single SKU per line** — paste or type SKUs with quantities
2. **CSV upload** — upload a CSV file with `sku,qty` columns
3. **Copy-paste** — paste multi-line `sku,qty` data

### 4.2 CSV Upload Format

```csv
sku,qty
MH01-XS-Black,10
WJ01-XS-Red,5
WS06-XL-Green,2
```

The CSV parser in `Magento_QuickOrder` validates each SKU against the catalog and discards invalid rows with an error report.

### 4.3 Quick Order Plugin — Filtering SKUs by Company Catalog

```php
// Plugin/QuickOrder/FilterSkuSuggestions.php
declare(strict_types=1);

namespace Vendor\Module\Plugin\QuickOrder\Model\Product;

use Magento\QuickOrder\Model\Product\Suggest;
use Magento\Framework\Api\SearchResultInterface;

/**
 * Filters Quick Order SKU suggestions to only those visible
 * in the current user's company shared catalog.
 */
class FilterSkuSuggestions
{
    public function aroundGetSuggestData(
        Suggest $subject,
        callable $proceed,
        string $query,
        int $count,
        string $scope
    ): SearchResultInterface {
        $result = $proceed($query, $count, $scope);

        // Apply company-catalog filter
        $items = array_filter(
            $result->getItems(),
            fn($item) => $this->isSkuInCompanyCatalog($item->getSku())
        );

        return $result->setItems(array_values($items));
    }

    private function isSkuInCompanyCatalog(string $sku): bool
    {
        // Inject SharedCatalog\ProductItemRepositoryInterface
        // and check if $sku exists for the company's catalog
        return true; // placeholder — implement with real check
    }
}
```

```xml
<!-- etc/di.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\QuickOrder\Model\Product\Suggest">
        <plugin name="vendorFilterSkuSuggestions"
                type="Vendor\Module\Plugin\QuickOrder\Model\Product\FilterSkuSuggestions"/>
    </type>
</config>
```

### 4.4 Adding a Custom Quick Order Source

```php
// Override the source chooser to add custom sources
use Magento\QuickOrder\Model\Config\Source\Catalog as CatalogSource;

class CustomCatalogSource extends CatalogSource
{
    public function toOptionArray(): array
    {
        $options = parent::toOptionArray();
        $options[] = [
            'value' => 'custom_erp',
            'label' => __('ERP Catalog Sync'),
        ];
        return $options;
    }
}
```

---

## 5. Requisition Lists

### 5.1 Data Model

```
requisition_list
├── entity_id
├── customer_id
├── store_id
├── name
├── description
├── updated_at
└── shared (bool)

requisition_list_item
├── entity_id
├── requisition_list_id (FK)
├── sku
├── store_id
├── qty
├── options (JSON — selected product options)
└── added_at
```

### 5.2 Requisition List Service Contract

```php
use Magento\RequisitionList\Api\RequisitionListRepositoryInterface;
use Magento\RequisitionList\Api\Data\RequisitionListInterfaceFactory;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Framework\Api\FilterBuilder;

class RequisitionListService
{
    public function __construct(
        private readonly RequisitionListRepositoryInterface $listRepository,
        private readonly RequisitionListInterfaceFactory $listFactory,
        private readonly SearchCriteriaBuilder $searchCriteriaBuilder,
    ) {}

    public function createListForCustomer(int $customerId, string $name): \Magento\RequisitionList\Api\Data\RequisitionListInterface
    {
        $list = $this->listFactory->create();
        $list->setCustomerId($customerId);
        $list->setName($name);
        $list->setStoreId(0);

        return $this->listRepository->save($list);
    }

    public function getListsForCustomer(int $customerId): array
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('customer_id', $customerId)
            ->create();

        return $this->listRepository->getList($searchCriteria)->getItems();
    }

    public function addItemToList(int $listId, string $sku, float $qty): void
    {
        // Inject RequisitionList\ItemRepositoryInterface
        // Build ItemInterface and save
    }
}
```

### 5.3 Converting a Requisition List to a Cart

```php
use Magento\RequisitionList\Model\Cart\CartCreate;

class RequisitionListCartConverter
{
    public function __construct(
        private readonly CartCreate $cartCreate,
    ) {}

    public function convertToCart(int $listId): int // returns quoteId
    {
        $this->cartCreate->setRequistionListId($listId);
        return $this->cartCreate->createCart();
    }
}
```

---

## 6. Negotiable Quotes

### 6.1 Quote Lifecycle

```
buyer creates quote
    │
    ▼
draft (status = SUBMITTED_AND_PENDING) ──── buyer edits
    │
    ├──────────────────┐
    ▼                  ▼
buyer declines    admin approves
    │                  │
    ▼                  ▼
closed           approved ────► ordered / invoiced / shipped
```

### 6.2 Quote States

| State Constant | Value | Meaning |
|---|---|---|
| `STATE_DRAFT` | 1 | Initial draft, editable |
| `STATE_SUBMITTED` | 2 | Submitted by buyer, pending seller review |
| `STATE_APPROVED` | 3 | Approved by seller |
| `STATE_REJECTED` | 4 | Rejected by seller |
| `STATE_CLOSED` | 5 | Closed/declined by buyer or expiration |

### 6.3 Negotiable Quote Data Model

```
negotiable_quote
├── quote_id
├── company_id
├── creator_id (customer_id)
├── status
├── quote_name
├── expiration_period    ← datetime
├── is_shipping_taxable
├── shipping_price
├── notes
├── extension_attributes
└── created_at / updated_at

negotiable_quote_history
├── history_id
├── negotiable_quote_id
├── action               ← SUBMIT, UPDATE, APPROVE, REJECT, DECLINE, CLOSE
├── actor_type           ← 1=admin, 2=customer, 3=system
├── actor_id
├── snapshot             ← JSON of quote state at this action
└── comment
```

### 6.4 Custom Quote Fields

Negotiable quotes support **custom fields** stored in the `negotiable_quote` table extension attributes. To add a custom field:

```php
// 1. Declare the extension attribute
// app/code/Vendor/Module/etc/extension_attributes.xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ExtensionAttributes/etc/extension_attributes.xsd">
    <extension_attributes for="Magento\NegotiableQuote\Api\Data\NegotiableQuoteInterface">
        <attribute code="customer_ref" type="string"/>
        <attribute code="approval_status" type="string"/>
    </extension_attributes>
</config>
```

```php
// 2. Set the value on a quote
use Magento\NegotiableQuote\Api\Data\NegotiableQuoteInterface;

public function setCustomFields(NegotiableQuoteInterface $quote): void
{
    $extensionAttributes = $quote->getExtensionAttributes();
    if ($extensionAttributes === null) {
        // Extension attributes must be created via the factory
        $extensionAttributes = $this->negotiableQuoteExtensionFactory->create();
    }
    $extensionAttributes->setCustomerRef('PO-2025-001');
    $extensionAttributes->setApprovalStatus('pending');
    $quote->setExtensionAttributes($extensionAttributes);
}
```

### 6.5 Quote Price Calculation Plugin

The negotiated price is applied via a plugin on the quote's address total collectors:

```php
// Plugin/NegotiableQuote/ApplyNegotiablePrice.php
declare(strict_types=1);

namespace Vendor\Module\Plugin\NegotiableQuote\Model;

use Magento\NegotiableQuote\Model\Quote\Totals\NegotiableQuoteTotals;

/**
 * Applies an additional discount on top of negotiated pricing.
 */
class ApplyNegotiablePrice
{
    public function afterCollectTotal(
        NegotiableQuoteTotals $subject,
        $result,
        $quote,
        $address
    ) {
        $negotiableQuote = $quote->getExtensionAttributes()->getNegotiableQuote();
        if (!$negotiableQuote) {
            return $result;
        }

        // Apply an extra 2% corporate discount
        $baseDiscount = $result->getBaseDiscountAmount();
        $result->setBaseDiscountAmount($baseDiscount + ($result->getBaseSubtotal() * 0.02));

        return $result;
    }
}
```

---

## 7. Company Credit

### 7.1 Credit Data Model

```
company_credit
├── id
├── company_id            ← FK → company.entity_id
├── credit_limit         ← decimal(12,4), NULL = unlimited
├── balance              ← decimal(12,4), current owed amount
├── currency_code
└── exceed_limit         ← tinyint(1): 1 = allow overspend

company_credit_history
├── id
├── company_credit_id
├── user_id
├── user_type            ← 1=admin, 2=customer
├── currency_amount     ← transaction amount
├── rate                 ← exchange rate snapshot
├── balance              ← running balance after transaction
├── credit_limit         ← limit at time of transaction
├── available_credit     ← balance - credit_limit
├── type                 ← 1=allocated, 2=updated, 3=purchased,
│                           4=reimbursed, 5=refunded, 6=reverted
└── comment
```

### 7.2 Credit Operations

```php
use Magento\CompanyCredit\Api\CreditLimitManagementInterface;
use Magento\CompanyCredit\Api\CreditLimitRepositoryInterface;
use Magento\CompanyCredit\Api\Data\CreditLimitInterfaceFactory;
use Magento\CompanyCredit\Api\Data\CreditBalanceInterfaceFactory;

class CompanyCreditService
{
    public function __construct(
        private readonly CreditLimitManagementInterface $creditLimitManagement,
        private readonly CreditLimitRepositoryInterface $creditLimitRepository,
        private readonly CreditLimitInterfaceFactory $creditLimitFactory,
    ) {}

    public function setCreditLimit(int $companyId, float $limit, string $currency): void
    {
        $credit = $this->creditLimitManagement->getCreditByCompanyId($companyId);

        if ($credit === null) {
            $credit = $this->creditLimitFactory->create();
            $credit->setCompanyId($companyId);
        }

        $credit->setCreditLimit($limit);
        $credit->setCurrencyCode($currency);
        $credit->setExceedLimit(false);

        $this->creditLimitRepository->save($credit);
    }

    public function getAvailableCredit(int $companyId): float
    {
        $credit = $this->creditLimitManagement->getCreditByCompanyId($companyId);
        if ($credit === null) {
            return 0.0;
        }
        return (float) $credit->getCreditLimit() - (float) $credit->getBalance();
    }
}
```

### 7.3 Credit Allocation Flow

When an order is placed against Company Credit:

```
Order placed
    │
    ▼
Sales Order saved ── event: sales_order_save_after
    │
    ▼
Creditmemo created ── type: 3 (purchased)
    │
    ▼
company_credit.balance += order total
```

You can observe `company_credit_update_after` to react to balance changes.

---

## 8. B2B Permissions

### 8.1 Company Role Structure

Company permissions are role-based, separate from admin ACL. Each company has its own set of roles defined by the company admin:

```
company_roles
├── role_id
├── company_id
├── role_name        ← e.g. "Buyer", "Approver", "Viewer"
└── permissions      ← JSON array of resource identifiers

company_acl_resource (pivot)
├── id
├── role_id
└── resource_id       ← e.g. "Magento_Company::company_purchases"
```

### 8.2 Permission Resources

Available B2B permission resources include:

| Resource Identifier | Controls |
|---|---|
| `Magento_Company::company_view` | View company information |
| `Magento_Company::company_edit` | Edit company details |
| `Magento_Company::company_users` | Manage company users |
| `Magento_Company::company_users_view` | View company users |
| `Magento_Company::company_roles` | Manage roles |
| `Magento_Company::company_purchases` | Make purchases |
| `Magento_Company::quick_order` | Use Quick Order pad |
| `Magento_Company::reqquote_list` | Use Requisition Lists |
| `Magento_Company::negotiable_quote_view` | View negotiable quotes |
| `Magento_Company::negotiable_quote_update` | Create/update quotes |
| `Magento_Company::negotiable_quote_send` | Submit quotes for approval |

### 8.3 Checking Permissions in Code

```php
use Magento\Company\Api\PermissionManagementInterface;

class B2bPermissionChecker
{
    public function __construct(
        private readonly PermissionManagementInterface $permissionManagement,
    ) {}

    public function canUserMakePurchases(int $customerId): bool
    {
        return $this->permissionManagement->isAllowed(
            $customerId,
            'Magento_Company::company_purchases'
        );
    }

    public function getUserPermissions(int $customerId): array
    {
        return $this->permissionManagement->getPermissionsByCustomerId($customerId);
    }
}
```

### 8.4 Custom Permission Resource

To add a custom permission resource for a module:

```xml
<!-- etc/acl.xml inside Magento_Company context -->
<!-- Extend the company permissions tree -->
<resource id="Magento_Company::company_purchases">
    <resource id="Vendor_Module::custom_purchase_approval"
              title="Request Special Approvals"
              sortOrder="50"/>
</resource>
```

```php
// Check in your custom code
if ($this->authorization->isAllowed('Vendor_Module::custom_purchase_approval')) {
    // show the custom approval workflow
}
```

---

## 9. B2B API Integration

### 9.1 REST Endpoints Summary

| Endpoint | Method | Purpose |
|---|---|---|
| `/V1/company/` | GET/POST | Manage companies |
| `/V1/company/{companyId}/users` | GET | List company users |
| `/V1/sharedCatalog/` | GET/POST | Manage shared catalogs |
| `/V1/sharedCatalog/{id}/assignProducts` | POST | Assign products |
| `/V1/sharedCatalog/{id}/assignCompanies` | POST | Assign companies |
| `/V1/negotiableQuote/` | GET/POST | Manage quotes |
| `/V1/negotiableQuote/{id}/send` | POST | Submit for approval |
| `/V1/negotiableQuote/{id}/updatePrice` | PUT | Update negotiated price |
| `/V1/requisitionList/` | GET/POST | Manage requisition lists |
| `/V1/requisitionList/{id}/items` | GET/POST/DELETE | Manage list items |
| `/V1/companycredit/` | GET/POST | Manage credit limits |

### 9.2 REST — Creating a Company

```bash
POST /V1/company
Content-Type: application/json
Authorization: Bearer {admin_token}

{
    "company": {
        "name": "Acme Wholesale Corp",
        "company_email": "admin@acme.example.com",
        "status": 1,
        "customer_group_id": 4,
        "sales_representative_id": 1,
        "address": {
            "street": ["100 Industrial Parkway"],
            "city": "Chicago",
            "region_id": "IL",
            "country_id": "US",
            "postcode": "60601",
            "telephone": "+1-312-555-0100"
        }
    }
}
```

### 9.3 REST — Working with Shared Catalogs

```bash
# Create a custom shared catalog
POST /V1/sharedCatalog
{
    "sharedCatalog": {
        "name": "Gold Partner Catalog",
        "description": "Special pricing for gold-tier partners",
        "type": 0,
        "customer_group_id": 5,
        "store_id": 0
    }
}

# Assign products
POST /V1/sharedCatalog/3/assignProducts
{
    "products": [
        {"sku": "PROD-001"},
        {"sku": "PROD-002"}
    ]
}

# Assign companies
POST /V1/sharedCatalog/3/assignCompanies
{
    "companies": [
        {"company_id": 7}
    ]
}
```

### 9.4 REST — Negotiable Quote Workflow

```bash
# 1. Buyer creates a draft quote
POST /V1/negotiableQuote
{
    "quote": {
        "company_id": 7,
        "quote_name": "Q3 Bulk Order Request",
        "customer_id": 42,
        "items": [
            {"sku": "PROD-001", "qty": 500}
        ]
    }
}

# 2. Seller approves the quote
PUT /V1/negotiableQuote/15
{
    "quote": {
        "status": "approved_by_admin",
        "negotiated_price_value": 4500.00,
        "negotiated_price_type": "fixed"
    }
}

# 3. Buyer places order from approved quote
POST /V1/negotiableQuote/15/createOrder
```

### 9.5 GraphQL Schema — B2B Queries

Each B2B module ships its own `.graphqls` schema file. Key B2B GraphQL types:

```graphql
# NegotiableQuote GraphQL schema excerpt
type NegotiableQuote {
    id: Int!
    quote_name: String
    status: NegotiableQuoteStatusEnum!
    created_at: String
    updated_at: String
    expiration_period: String
    notes: String
    applied_changes: [QuoteNegotiableChange!]!
    items: [NegotiableQuoteItem!]!
    seller_cart: Cart
    buyer_cart: Cart
}

enum NegotiableQuoteStatusEnum {
    DRAFT
    SUBMITTED
    PROCESSING
    APPROVED
    REJECTED
    DECLINED
    CLOSED
    EXPIRED
}

type Query {
    negotiableQuotes(
        filter: NegotiableQuoteFilterInput
        pageSize: Int = 20
        currentPage: Int = 1
    ): NegotiableQuotes

    company: Company
    sharedCatalogs: [SharedCatalog]
    requisitionLists: [RequisitionList]
}

type Mutation {
    createNegotiableQuote(input: NegotiableQuoteInput!): NegotiableQuote
    updateNegotiableQuote(input: NegotiableQuoteUpdateInput!): NegotiableQuote
    sendNegotiableQuoteForApproval(quoteId: Int!): Boolean
    addNegotiableQuoteItem(input: NegotiableQuoteItemInput!): NegotiableQuoteItem
}
```

### 9.6 GraphQL — Creating a Negotiable Quote

```graphql
mutation createQuote($input: NegotiableQuoteInput!) {
    createNegotiableQuote(input: $input) {
        id
        quote_name
        status
        items {
            sku
            qty
            price
        }
    }
}
```

```json
{
  "input": {
    "quote_name": "Q4 Office Supplies",
    "company_id": 7,
    "items": [
      { "sku": "OFFICE-DSK-001", "qty": 10 },
      { "sku": "OFFICE-CHAIR-001", "qty": 20 }
    ]
  }
}
```

### 9.7 B2B API Rate Limiting

B2B API endpoints are subject to the same rate limiting as standard Magento REST/GraphQL endpoints. Configure in `env.php`:

```php
// app/etc/env.php
return [
    'rate_limiting' => [
        'rest' => [
            'limit'  => 2000,
            'period' => 3600,
        ],
        'graphql' => [
            'limit'  => 1000,
            'period' => 3600,
        ],
    ],
];
```

---

## Quick Reference

| Task | Service Contract |
|---|---|
| Create/read company | `Magento\Company\Api\CompanyRepositoryInterface` |
| Get company for customer | `Magento\Company\Api\CompanyManagementInterface::getByCustomerId()` |
| Create/update roles | `Magento\Company\Api\RoleRepositoryInterface` |
| Check user permissions | `Magento\Company\Api\PermissionManagementInterface::isAllowed()` |
| Assign company to shared catalog | `Magento\SharedCatalog\Api\CompanyManagementInterface::assignCompanies()` |
| Set shared catalog prices | Uses `customer_group_id` + tier prices / catalog price rules |
| Get credit limit | `Magento\CompanyCredit\Api\CreditLimitManagementInterface` |
| Create requisition list | `Magento\RequisitionList\Api\RequisitionListRepositoryInterface` |
| Create negotiable quote | `Magento\NegotiableQuote\Api\NegotiableQuoteRepositoryInterface` |
| Check B2B feature flags | `Magento\Company\Api\CompanyConfigurationInterface` |
| List Quick Order SKUs | `Magento\QuickOrder\Model\Product\Suggest` |
| REST endpoints base | `/V1/company`, `/V1/sharedCatalog`, `/V1/negotiableQuote`, `/V1/requisitionList` |
| GraphQL namespace | `NegotiableQuote`, `SharedCatalog`, `RequisitionList`, `Company` |