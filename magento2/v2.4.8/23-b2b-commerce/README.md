---
title: "23 - B2B Commerce (Adobe Commerce)"
description: "Magento 2.4.8 Adobe Commerce B2B suite: customer segments, shared catalogs, company accounts, quick order, requisition lists, negotiable quotes, and B2B permissions."
rank: 23
pathways: [magento2-deep-dive]
see_also:
  - "Customer Account (Topic 22): ../22-customer-account/README.md"
  - "Customer Segments & Promotions: ../_supplemental/12-customer-segments-promotions.md"
  - "B2B Shared Catalog: ../_supplemental/13-b2b-shared-catalog.md"
  - "B2B Advanced Features: ../_supplemental/14-b2b-advanced.md"
---

# Topic 23 — B2B Commerce (Adobe Commerce)

Magento's B2B suite is an extension of Adobe Commerce (Enterprise) that enables business-to-business trading scenarios. It provides company account management, shared catalogs with custom pricing, quick order pad for bulk purchasing, requisition lists, and negotiable quotes with approval workflows.

This topic covers three supplemental chapters that together provide complete B2B coverage.

## B2B Module Stack

| Module | Purpose |
|--------|---------|
| Magento_Company | Company accounts, structure, user roles |
| Magento_SharedCatalog | Per-company custom catalog pricing |
| Magento_CatalogPermissions | Category/product visibility restrictions |
| Magento_QuickOrder | Bulk SKU entry and add-to-cart |
| Magento_RequisitionList | Persistent multiple wishlists for repeat orders |
| Magento_NegotiableQuote | Quotes with pricing proposals and approval |
| Magento_CompanyCredit | Credit limit management per company |
| Magento_CompanyPayment | Payment method restrictions per company |

## Key Architecture Patterns

### Company Hierarchy

```
Master Company (root)
├── Child Company 1
│   ├── User A (Buyer)
│   └── User B (Approver)
└── Child Company 2
    └── User C (Buyer)
```

### Shared Catalog Pricing

Unlike standard catalog pricing (tier prices, special prices), shared catalog pricing is **company-specific** and overrides all other pricing mechanisms:

```php
// Magento\SharedCatalog\Model\Price\Calculator
public function getPrice(ProductInterface $product, int $customerGroupId): float
{
    // Check shared catalog custom price first
    $sharedCatalogPrice = $this->getSharedCatalogPrice(
        $product,
        $customerGroupId
    );

    if ($sharedCatalogPrice !== null) {
        return $sharedCatalogPrice; // B2B price wins
    }

    // Fall back to standard catalog pricing
    return $this->standardPriceCalculator->getPrice($product);
}
```

### Negotiable Quote Workflow

```
Buyer creates quote
    → adds products, requests discount
    → submits for approval
        → Approver reviews & negotiates
        → Buyer accepts/rejects counter
            → On acceptance: converts to order
```

---

## Supplemental Chapters

This topic is organized across three supplemental chapters:

- **[Customer Segments & Promotions](../_supplemental/12-customer-segments-promotions.md)** — Customer segmentation for targeted B2B pricing and promotions
- **[B2B Shared Catalog](../_supplemental/13-b2b-shared-catalog.md)** — Shared catalog architecture, company-specific pricing, and catalog permissions
- **[B2B Advanced Features](../_supplemental/14-b2b-advanced.md)** — Quick order, requisition lists, negotiable quotes, and company credit

Each chapter is canonical and provides in-depth coverage with code examples, data structures, and implementation patterns.
