---
title: "25 - Catalog Product Types & Type Architecture"
description: "Magento 2.4.8 product type architecture: simple, configurable, bundle, grouped, virtual, downloadable products, type-specific behavior, and price calculation."
tags: magento2, product-type, catalog-product, configurable-product, bundle-product, grouped-product, virtual-product, downloadable-product, type-handler
rank: 25
pathways: [magento2-deep-dive]
see_also:
  - path: "04-data-layer/02-eav-entity-model.md"
    description: "EAV Entity Model — product entity structure"
  - path: "11-search-inventory/README.md"
    description: "Search & Inventory — configurable products in MSI context"
---

# Catalog Product Types & Type Architecture

Magento supports multiple product types — each with distinct behavior, pricing, and inventory models. Understanding these types is essential for product import, custom product builders, and any feature that interacts with the catalog.

---

## 1. Product Type Hierarchy

### The Type List

| Type | Code | Behavior |
|------|------|----------|
| Simple | `simple` | Standalone product with fixed price/inventory |
| Configurable | `configurable` | Container with variants (e.g., size/color) |
| Grouped | `grouped` | Collection of simple products sold together |
| Virtual | `virtual` | No physical shipment required |
| Downloadable | `downloadable` | Digital goods with links/samples |
| Bundle | `bundle` | Dynamically assembled product |
| Gift Card | `giftcard` | Prepaid card with value |

### Type Models

```php
<?php
// vendor/magento/module-catalog/Model/Product/Type

abstract class AbstractType
{
    protected $infoModel;
    protected $setAttributeFunctions = [];

    abstract public function save($product);

    public function isSalable($product): bool
    {
        return $product->getStatus() === Status::STATUS_ENABLED;
    }

    public function getWeight($product): ?float
    {
        return $this->getProduct($product)->getWeight();
    }
}
```

### Type Factory

```php
<?php
// Getting product type instance

/** @var \Magento\Catalog\Model\Product\TypeFactory $typeFactory */
$typeFactory = $this->typeFactory->create();

$productType = $typeFactory->getProductType($product->getTypeId());
// Returns: \Magento\Catalog\Model\Product\Type\Simple, Configurable, etc.
```

---

## 2. Simple Products

### Simple Product Behavior

Simple products are the baseline product type:
- Standalone, no variants
- Has physical inventory (unless virtual)
- Fixed price
- Single SKU

### Simple Product Model

```php
<?php
// \Magento\Catalog\Model\Product\Type\Simple

class Simple extends \Magento\Catalog\Model\Product\Type\AbstractType
{
    public function isSalable($product): bool
    {
        // Check stock status
        return parent::isSalable($product) && $this->isInStock($product);
    }

    public function checkProductConfiguration($product): bool
    {
        return true;  // Simple products have no child configuration
    }
}
```

### Simple Product Use Cases

- Single physical item with fixed price
- Physical goods, downloadable software (with inventory)
- Most e-commerce products

---

## 3. Configurable Products

### Configurable Product Architecture

A configurable product is a "parent" that represents a product with variants:

```
Configurable Product (e.g., "T-Shirt")
├── Super Attributes: size, color
├── Associated Products:
│   ├── T-Shirt Blue Small
│   ├── T-Shirt Blue Medium
│   ├── T-Shirt Red Small
│   └── T-Shirt Red Medium
└── Price: inherited from associated (or own price as fallback)
```

### Configurable Super Attributes

```php
<?php
// Super attributes define what variations exist
// Stored in catalog_product_super_attribute table

$superAttributes = [
    ['attribute_id' => 93, 'label' => 'Size', 'values' => [
        ['value_index' => 92, 'label' => 'Small'],
        ['value_index' => 93, 'label' => 'Medium'],
        ['value_index' => 94, 'label' => 'Large'],
    ]],
    ['attribute_id' => 94, 'label' => 'Color', 'values' => [
        ['value_index' => 52, 'label' => 'Blue'],
        ['value_index' => 53, 'label' => 'Red'],
    ]],
];
```

### Configurable Product Model

```php
<?php
// \Magento\ConfigurableProduct\Model\Product\Type\Configurable

class Configurable extends \Magento\Catalog\Model\Product\Type\AbstractType
{
    protected $usedProducts;  // Cached collection of associated products

    public function getUsedProducts($product, $requiredAttributeOnly = null): Collection
    {
        if ($this->usedProducts === null) {
            $collection = $this->getProductCollection()
                ->addFilter('parent_id', $product->getId());
            $this->usedProducts = $collection;
        }
        return $this->usedProducts;
    }

    public function getPrice($product): float
    {
        // Price from configurable's own price OR
        // lowest price of associated products
        $basePrice = $product->getData('price');
        $minPrice = $this->getMinimalPrice($product);

        return $minPrice ?: $basePrice;
    }

    public function getFinalPrice(float $qty, $product): float
    {
        // Get price of selected variant
        $variantId = $product->getSelectedVariantId();
        if ($variantId) {
            $variant = $this->getProductById($variantId);
            return $variant->getFinalPrice();
        }
        return parent::getFinalPrice($qty, $product);
    }
}
```

### Creating Configurable Products Programmatically

```php
<?php
public function createConfigurable(
    string $name,
    string $sku,
    array $superAttributes,
    array $variants
): ProductInterface {
    /** @var \Magento\Catalog\Api\Data\ProductInterface $configurable */
    $configurable = $this->productFactory->create();

    $configurable->setTypeId(\Magento\ConfigurableProduct\Model\Product\Type\Configurable::TYPE_CODE);
    $configurable->setAttributeSetId($this->attributeSetId);
    $configurable->setName($name);
    $configurable->setSku($sku);
    $configurable->setStatus(Status::STATUS_ENABLED);
    $configurable->setVisibility(Visibility::VISIBILITY_BOTH);

    // Set super attributes (size, color, etc.)
    $attributeData = [];
    foreach ($superAttributes as $attrCode) {
        $attribute = $this->eavConfig->getAttribute('catalog_product', $attrCode);
        $attributeData[$attribute->getId()] = [
            'label' => ['value' => $attribute->getStoreLabel() ?: $attribute->getFrontendLabel()]
        ];
    }
    $configurable->getExtensionAttributes()->setConfigurableProductOptions($attributeData);

    // Create and associate variants
    foreach ($variants as $variantData) {
        $variant = $this->createSimpleProduct($variantData);
        $variant->setData('parent_id', $configurable->getId());
        $this->productRepository->save($variant);
    }

    return $this->productRepository->save($configurable);
}
```

---

## 4. Grouped Products

### Grouped Product Architecture

Grouped products are a collection of simple products sold as a unit:

```
Grouped Product "Bedroom Set"
├── Associated Products (all required or optional):
│   ├── Bed Frame (simple, qty=1)
│   ├── Mattress (simple, qty=1)
│   └── Pillows (simple, qty=2)
└── No price of its own (sum of components)
```

### Grouped Product Model

```php
<?php
// \Magento\Catalog\Model\Product\Type\Grouped

class Grouped extends \Magento\Catalog\Model\Product\Type\AbstractType
{
    public function getAssociatedProducts($product): Collection
    {
        // Returns collection of linked simple products
        // From catalog_product_link table (type='super_group')
        return $this->getProductCollection()
            ->addFilter('link_type_id', 4)  // super_group link type
            ->addFilter('product_id', $product->getId());
    }

    public function getPrice($product): float
    {
        // Sum of all associated product prices
        $total = 0;
        foreach ($this->getAssociatedProducts($product) as $assocProduct) {
            $total += $assocProduct->getPrice();
        }
        return $total;
    }
}
```

### Grouped Product Use Cases

- "Buy the set" bundles
- Pre-configured kits
- Cross-sell bundles

---

## 5. Bundle Products

### Bundle Product Architecture

Bundle products let customers configure a product from options:

```
Bundle Product "Build Your Laptop"
├── Options:
│   ├── CPU: [Intel i5, Intel i7, AMD Ryzen 7] (required)
│   ├── RAM: [8GB, 16GB, 32GB] (required)
│   ├── Storage: [256GB SSD, 512GB SSD, 1TB SSD] (required)
│   └── Accessories: [Mouse, Keyboard, None] (optional)
└── Dynamic price based on selections
```

### Bundle Product Model

```php
<?php
// \Magento\Bundle\Model\Product\Type

class Bundle extends \Magento\Catalog\Model\Product\Type\AbstractType
{
    public function getPrice($product): float
    {
        // Returns 0 for dynamic pricing
        // Or fixed price for fixed pricing bundles
        return $product->getPrice();
    }

    public function getFinalPrice(float $qty, $product): float
    {
        // Sum of selected option prices
        $options = $product->getBundleOptions();
        $total = 0;

        foreach ($options as $option) {
            $selection = $option->getDefaultSelection();
            if ($selection) {
                $total += $selection->getPrice();
            }
        }

        return $total;
    }

    public function isSalable($product): bool
    {
        // At least one option must be in stock
        return parent::isSalable($product) && $this->hasAvailableOptions($product);
    }
}
```

### Bundle Selection Options

```sql
catalog_product_bundle_option
├── option_id
├── parent_id (product_id)
├── required   -- 1 = must select, 0 = optional
└── type       -- radio, checkbox, select, multi

catalog_product_bundle_selection
├── selection_id
├── option_id
├── product_id (the associated simple product)
├── selection_price_value
├── selection_price_type  -- 0 = fixed, 1 = percent
└── selection_qty
```

### Bundle Use Cases

- Customizable/tailored products
- Build-to-order products
- Gift baskets

---

## 6. Virtual Products

### Virtual Product Behavior

Virtual products have no physical form:
- No shipping required
- No weight
- Can be used with downloadable/license extensions

```php
<?php
// \Magento\Catalog\Model\Product\Type\Virtual

class Virtual extends \Magento\Catalog\Model\Product\Type\Simple
{
    public function isVirtual(bool $product = true): bool
    {
        return true;
    }

    public function hasWeight(): bool
    {
        return false;  // Virtual products have no weight
    }
}
```

### Virtual Product Use Cases

- Services (consulting, subscriptions)
- Memberships
- Extended warranties
- Booking deposits

---

## 7. Downloadable Products

### Downloadable Product Architecture

```php
<?php
// \Magento\Downloadable\Model\Product\Type

class Downloadable extends \Magento\Catalog\Model\Product\Type\Virtual
{
    public function getLinksPurchasedSeparately(): bool
    {
        return true;
    }
}
```

### Downloadable Links

```sql
downloadable_link
├── link_id
├── product_id
├── title
├── price
├── number_of_downloads  -- -1 = unlimited
└── is_shareable

downloadable_link_price
├── price_id
├── link_id
└── website_id

catalog_product_entity
├── link_file   -- path to file in var/import/downloadable
├── link_type   -- file or url
```

### Downloadable Use Cases

- Software downloads
- Digital art/ebooks
- Music/video files
- License keys

---

## 8. Price Calculation by Type

### Price Calculation Flow

```php
<?php
// \Magento\Catalog\Model\Product::getFinalPrice()

public function getFinalPrice(float $qty = null): float
{
    $price = $this->getPrice();
    $finalPrice = $price;

    // Get type-specific final price
    $type = $this->getTypeInstance();

    // Let type model calculate
    $typeFinalPrice = $type->getFinalPrice($qty, $this);

    if ($typeFinalPrice !== null) {
        $finalPrice = $typeFinalPrice;
    }

    // Apply tier pricing
    $tierPrice = $this->getTierPrice($qty);
    $finalPrice = min($finalPrice, $tierPrice);

    // Apply catalog rules
    $catalogRulePrice = $this->getCatalogRulePrice();
    $finalPrice = min($finalPrice, $catalogRulePrice);

    return $finalPrice;
}
```

### Type-Specific Pricing

| Type | Price Source |
|------|---------------|
| Simple | Own `price` attribute |
| Configurable | Lowest of associated variants |
| Grouped | Sum of associated product prices |
| Bundle | Sum of selected option prices |
| Virtual | Own `price` attribute |
| Downloadable | Own `price` attribute |

---

## 9. Stock and Inventory by Type

### Inventory Behavior

```php
<?php
// Stock check by product type

public function isInStock(ProductInterface $product): bool
{
    $type = $product->getTypeInstance();

    // Simple, Virtual, Downloadable: check inventory
    if ($type instanceof \Magento\Catalog\Model\Product\Type\Simple ||
        $type instanceof \Magento\Catalog\Model\Product\Type\Virtual) {
        return $this->stockRegistry->getStockItem($product->getId())->getIsInStock();
    }

    // Configurable: all variants must be in stock (or allow out-of-stock)
    if ($type instanceof \Magento\ConfigurableProduct\Model\Product\Type\Configurable) {
        return $this->configurableType->areAllVariantsInStock($product);
    }

    // Grouped: all associated products must be in stock
    if ($type instanceof \Magento\Catalog\Model\Product\Type\Grouped) {
        return $this->groupedType->areAllAssociatedInStock($product);
    }

    // Bundle: check option selections
    if ($type instanceof \Magento\Bundle\Model\Product\Type) {
        return $this->bundleType->isSalable($product);
    }
}
```

---

## 10. Common Mistakes to Avoid

1. ❌ Treating all products as simple → Configurable/grouped have different behavior
2. ❌ Not checking type before price operations → Type affects price calculation
3. ❌ Forgetting super attributes on configurable → No variant options shown
4. ❌ Grouped product with no associated products → Zero price, no salable
5. ❌ Bundle without required options → Customer can add with no selections
6. ❌ Using simple for non-physical goods → Use Virtual instead

---

## Reading List

- [Product types](https://experienceleague.adobe.com/docs/commerce-admin/catalog/products/product-types.html)
- [Configurable products](https://experienceleague.adobe.com/docs/commerce-admin/catalog/products/product-create-configurable.html)
- [Bundle products](https://experienceleague.adobe.com/docs/commerce-admin/catalog/products/product-create-bundle.html)

---

*Magento 2 Backend Developer Course — Topic 04 — Data Layer*