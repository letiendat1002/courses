---
title: "22 - Import/Export Framework"
description: "Magento 2.4.8 import/export architecture: ImportExport framework, profile-based import, custom import entities, export processing, and data flow."
tags: magento2, import, export, ImportExport, import-entity, ProfileImport, CSV, entity-adapter, data-flow
rank: 22
pathways: [magento2-deep-dive]
see_also:
  - path: "08-data-ops/README.md"
    description: "Data Operations — import/export at module level"
  - path: "04-data-layer/02-eav-entity-model.md"
    description: "EAV Entity Model — import maps to EAV attribute structure"
---

# Import/Export Framework

Magento's Import/Export framework is a sophisticated system for moving data in and out of Magento. It handles products, customers, inventory, and more. Understanding this framework is essential for building custom import adapters, extending existing imports, and debugging data migration issues.

---

## 1. Import/Export Architecture Overview

### Key Components

| Component | Class | Purpose |
|-----------|-------|---------|
| Import Export Model | `\Magento\ImportExport\Model\Export` | Base for all exports |
| Import Model | `\Magento\ImportExport\Model\Import` | Base for all imports |
| Entity Adapter | `\Magento\ImportExport\Model\Import\Entity\AbstractEntity` | Handles specific entity type |
| Behavior Handler | `\Magento\ImportExport\Model\Import\Entity\AbstractBehavior` | Controls add/update/delete behavior |
| Adapter Factory | `\Magento\ImportExport\Model\Import\Adapter` | Creates file adapters (CSV, etc.) |

### Architecture Diagram

```
ImportExport Controller (admin)
    ↓
Import Model (handles UI, validation, file processing)
    ↓
Entity Adapter (knows entity-specific logic)
    ↓
Behavior Handler (handles add/update/replace/delete)
    ↓
EAV / DB Layer (saves to Magento)
```

---

## 2. Export Framework

### Export Process

```php
<?php
// \Magento\ImportExport\Model\Export

public function export(): \Magento\Framework\Data\Collection
{
    // 1. Get entity adapter
    $entityAdapter = $this->getEntityAdapter();

    // 2. Set export filter (from UI form data)
    $entityAdapter->setFilters($this->filter);

    // 3. Export and return collection of files
    return $entityAdapter->export();
}
```

### Entity Adapter Example

```php
<?php
// \Magento\Catalog\Model\Export\Product

class Product extends AbstractEntity
{
    protected $_entityTypeCode = 'catalog_product';

    public function getEntityTypeCode(): string
    {
        return $this->_entityTypeCode;
    }

    protected function _getHeaderColumns(): array
    {
        return ['sku', 'name', 'price', 'weight', 'status', 'visibility'];
    }

    protected function _exportData(): array
    {
        $products = $this->_getEntityCollection();  // Product collection with filters

        $rows = [];
        foreach ($products as $product) {
            $rows[] = [
                'sku' => $product->getSku(),
                'name' => $product->getName(),
                'price' => $product->getPrice(),
                'weight' => $product->getWeight(),
                'status' => $product->getStatus(),
                'visibility' => $product->getVisibility()
            ];
        }

        return $rows;
    }
}
```

### Export File Format

```csv
sku,name,price,weight,status,visibility
SKU001,"Product Name 1",99.99,1.5,1,4
SKU002,"Product Name 2",149.99,2.0,1,4
```

---

## 3. Import Framework

### Import Process

```php
<?php
// \Magento\ImportExport\Model\Import

public function importSource(): bool
{
    // 1. Load file into adapter
    $sourceFile = $this->getSourceFile();
    $adapter = $this->_getDataSourceModel()->getAdapter($sourceFile);

    // 2. Read and validate data
    while ($adapter->valid()) {
        $rowData = $adapter->fetchRow();
        $this->validateRow($rowData);
    }

    // 3. Execute import based on behavior
    // $this->importData();

    return true;
}
```

### Import Behaviors

| Behavior | Meaning |
|----------|--------|
| `add` | Add new entities, skip existing |
| `add_update` | Add new entities, update existing |
| `replace` | Update existing entities (delete first) |
| `delete` | Delete entities that exist in file |
| `delete_attr` | (Special) Delete attribute values |

```php
<?php
// Behavior is set via UI or programmatic import
public function setBehavior(string $behavior): ImportInterface
{
    $this->_behavior = $behavior;
    return $this;
}
```

---

## 4. Custom Import Entity

### Create Custom Import Adapter

```php
<?php
// Vendor/ImportExport/Model/Import/Entity/CustomProduct.php

declare(strict_types=1);

namespace Vendor\ImportExport\Model\Import\Entity;

use Magento\ImportExport\Model\Import\Entity\AbstractEntity;
use Magento\ImportExport\Model\Import\EntityAbstractModel;

class CustomProduct extends AbstractEntity
{
    protected $_entityTypeCode = 'custom_product';

    protected array $attributes = [
        'sku',
        'name',
        'price',
        'description',
        'weight',
        'status',
        'categories'
    ];

    public function getEntityTypeCode(): string
    {
        return $this->_entityTypeCode;
    }

    public function getAttributes(): array
    {
        return $this->attributes;
    }

    public function validateData(): \Magento\Framework\Data\Collection
    {
        // Validate all rows
        // Return collection of errors
    }

    protected function _importData(): bool
    {
        while ($bunch = $this->_dataSourceModel->getNextBunch()) {
            foreach ($bunch as $rowNum => $rowData) {
                if ($this->validateRow($rowData, $rowNum)) {
                    $this->saveEntity($rowData);
                }
            }
        }
        return true;
    }

    private function saveEntity(array $rowData): void
    {
        // Use ProductRepository or ProductFactory
        $product = $this->productFactory->create();

        $product->setSku($rowData['sku']);
        $product->setName($rowData['name']);
        $product->setPrice($rowData['price']);
        $product->setWeight($rowData['weight']);
        $product->setStatus($rowData['status']);

        // Handle categories
        $categoryIds = $this->getCategoryIdsFromPath($rowData['categories']);
        $product->setCategoryIds($categoryIds);

        $this->productRepository->save($product);
    }
}
```

### Register Custom Entity

```xml
<!-- etc/di.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/di.xsd">

    <type name="Magento\ImportExport\Model\Import\EntityFactory">
        <arguments>
            <argument name="entities" xsi:type="array">
                <item name="custom_product" xsi:type="string">Vendor\ImportExport\Model\Import\Entity\CustomProduct</item>
            </argument>
        </arguments>
    </type>

</config>
```

### CSV Format for Custom Product

```csv
sku,name,price,weight,status,categories
SKU001,"Custom Product",99.99,1.5,1,"Default Category/Subcategory"
SKU002,"Another Product",149.99,2.0,1,"Default Category/Subcategory"
```

---

## 5. Product Import Adapter (Built-in)

### Product Import Processing

```php
<?php
// \Magento\CatalogImportExport\Model\Import\Product

class Product extends AbstractEntity
{
    protected function _importData(): bool
    {
        // Process in batches (bunch) to manage memory
        while ($bunch = $this->_dataSourceModel->getNextBunch()) {
            foreach ($bunch as $rowNum => $rowData) {
                if (!$this->validateRow($rowData, $rowNum)) {
                    continue;
                }

                $this->processRow($rowData);
            }

            // After batch: clean up, save images, reindex
        }
        return true;
    }

    private function processRow(array $rowData): void
    {
        // Check if SKU exists
        if ($this->skuExists($rowData['sku'])) {
            // Update existing based on behavior
            $this->updateProduct($rowData);
        } else {
            // Create new product
            $this->createProduct($rowData);
        }
    }
}
```

### Product Image Import

```php
<?php
// Images are imported via _mediaGalleryImages[] column
// Format: /path/to/image.jpg, /path/to/image2.jpg

// Image import logic:
// 1. Check if image exists in /var/import/images/
// 2. Copy to /pub/media/import/
// 3. Link to product via media gallery attribute

// Column format for images:
// _media_image,/path/to/image.jpg,_media_attribute_id,_media_position,_media_disabled
```

---

## 6. Customer Import

### Customer Import Adapter

```php
<?php
// \Magento\CustomerImportExport\Model\Import\Customer

class Customer extends AbstractEntity
{
    protected function _importData(): bool
    {
        while ($bunch = $this->_dataSourceModel->getNextBunch()) {
            foreach ($bunch as $rowData) {
                if ($this->validateRow($rowData)) {
                    $this->saveCustomer($rowData);
                }
            }
        }
        return true;
    }
}
```

### Customer Import CSV Format

```csv
email,firstname,lastname,website_id,group_id,created_at,store_id
john@example.com,John,Doe,1,1,"2024-01-15 10:00:00",1
jane@example.com,Jane,Smith,1,1,"2024-01-15 11:00:00",1
```

### Customer Address Import

```csv
_email,firstname,lastname,street,city,region,country_id,postcode,telephone
john@example.com,John,Doe,"123 Main St","Los Angeles",CA,US,90001,555-555-5555
```

---

## 7. Validation and Error Handling

### Row Validation

```php
<?php
public function validateRow(array $rowData, int $rowNum): bool
{
    // Check required fields
    if (empty($rowData['sku'])) {
        $this->addRowError('SKU is required', $rowNum);
        return false;
    }

    // Check SKU format (alphanumeric, dashes, underscores)
    if (!preg_match('/^[A-Za-z0-9_-]+$/', $rowData['sku'])) {
        $this->addRowError('Invalid SKU format', $rowNum);
        return false;
    }

    // Check duplicate SKU (if adding)
    if ($this->getBehavior() === ImportBehavior::ADD &&
        $this->skuExists($rowData['sku'])) {
        $this->addRowError('SKU already exists', $rowNum);
        return false;
    }

    // Check price is numeric
    if (isset($rowData['price']) && !is_numeric($rowData['price'])) {
        $this->addRowError('Price must be numeric', $rowNum);
        return false;
    }

    return true;
}
```

### Error Collection

```php
<?php
// \Magento\ImportExport\Model\Import

protected array $errors = [];

// Add row error
public function addRowError(string $errorCode, int $rowNum, array $errorMessages = []): void
{
    $this->errors[$rowNum][] = $errorCode;
}

// Get all errors
public function getErrors(): array
{
    return $this->errors;
}

// Get error count
public function getErrorCount(): int
{
    return count($this->errors);
}
```

### Error Report

After import fails, Magento generates an error report file:

```csv
_row,error,attribute value
5,Invalid SKU format,Invalid@SKU!
6,Price must be numeric,not-a-price
```

---

## 8. Export CLI Commands

```bash
# Export products
bin/magento export:products --entity= catalog_product
bin/magento export:products --entity=catalog_product --file=/path/to/export.csv

# Export customers
bin/magento export:customers --entity=customer

# Export all entity types
bin/magento list entity  # Show available entities
```

---

## 9. Import CLI Commands

```bash
# Import products from CSV
bin/magento import:products --entity=catalog_product --file=/path/to/import.csv

# With behavior options
bin/magento import:products --entity=catalog_product \
    --file=/path/to/import.csv \
    --behavior=add_update

# Validate only (no actual import)
bin/magento import:products --entity=catalog_product \
    --file=/path/to/import.csv \
    --dry-run
```

---

## 10. Memory and Performance Optimization

### Batch Processing

```php
<?php
// Import processes data in "bunches" to manage memory

while ($bunch = $this->_dataSourceModel->getNextBunch()) {
    // Process 100 rows at a time
    foreach ($bunch as $rowNum => $rowData) {
        // ...
    }

    // After batch: clear memory, log progress
    $this->_afterBunchProcess();
}

private function _afterBunchProcess(): void
{
    // Clean entity collection
    // Flush image cache
    // Log progress

    gc_collect_cycles();  // Force garbage collection
}
```

### Import Performance Tips

| Technique | Effect |
|-----------|--------|
| Disable indexes during import | Faster bulk inserts, rebuild afterward |
| Use `add_update` behavior | Faster than `replace` (no delete) |
| Increase PHP memory | Allows larger batches |
| Use stock JSON format | Faster than CSV for large datasets |

```php
<?php
// Disable indexes before bulk import
public function beforeImport(): void
{
    $indexerIds = [
        'catalog_product_price',
        'catalog_product_flat',
        'catalogsearch_fulltext'
    ];

    foreach ($indexerIds as $indexerId) {
        $indexer = $this->indexerRegistry->get($indexerId);
        $indexer->setIngestMode();  // Or invalidates
    }
}

public function afterImport(): void
{
    // Reindex after import completes
    $this->indexerRegistry->get('catalog_product_price')->reindexAll();
}
```

---

## Reading List

- [Import/export documentation](https://experienceleague.adobe.com/docs/commerce-admin/systems/data-transfer/data-import/data-import.html)
- [Product import](https://developer.adobe.com/commerce/php/development/components/import-export/)
- [Customer import](https://experienceleague.adobe.com/docs/commerce-admin/systems/data-transfer/data-import/process-customers.html)

---

## Common Mistakes to Avoid

1. ❌ CSV encoding not UTF-8 → Special characters get corrupted
2. ❌ Missing required columns → Import fails on first row
3. ❌ Not disabling indexes during bulk import → Extremely slow
4. ❌ Import file too large → Memory exhaustion; split into smaller files
5. ❌ Wrong behavior setting → `replace` deletes then re-inserts (slower, riskier)
6. ❌ Forgetting to reindex after import → Search/category pages out of date

---

*Magento 2 Backend Developer Course — Topic 08 — Data Operations*