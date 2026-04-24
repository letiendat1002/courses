---
title: "30 - EAV Entity Model"
description: "Deep dive into Magento 2.4.8 EAV (Entity-Attribute-Value) architecture: the 6 core EAV entities, table structure, attribute backend/source/frontend models, collections, performance patterns, and custom EAV registration."
tags: magento2, eav, entity-attribute-value, eav-entity, product-eav, customer-eav, category-eav, eav-collections, flat-tables
rank: 2
pathways: [magento2-deep-dive]
see_also:
  - path: "03-di-plugins.md"
    description: "Plugin system — plugins are often applied to EAV repository and attribute management classes"
  - path: "15-event-observer.md"
    description: "Event/Observer system — EAV save operations dispatch events that observers react to"
  - path: "README.md"
    description: "Data layer overview — EAV architecture and custom attributes are core topics in this module"
---

# EAV Entity Model

The Entity-Attribute-Value (EAV) model is Magento's answer to a hard problem: how do you store catalog products, customer profiles, and categories — entities that need radically different sets of attributes per instance — in a relational MySQL database without resorting to thousands of fixed columns or thousands of tables?

Before diving into the architecture, understand the core insight: **EAV separates "what attributes exist" (metadata) from "what values an entity has" (data)**. This separation allows admin users to create new attributes at runtime without altering the physical database schema.

In Magento 2.4.8, EAV underpins the three largest data domains in the platform:

- **Catalog products** — configurable attributes like size, color, material
- **Catalog categories** — display settings, featured flags, thumbnail images
- **Customers and their addresses** — loyalty tier, preferred shipping address, custom fields

Understanding EAV deeply is prerequisite to working effectively with product imports, custom attribute development, performance optimization, and the flat-table indexer system.

---

## 1. What EAV Solves

### The Fixed-Column Problem

A traditional flat product table has fixed columns:

```sql
CREATE TABLE flat_product (
    entity_id INT PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL(12,4),
    description TEXT,
    created_at TIMESTAMP
);
```

This works for a warehouse system where every product has the same columns. It fails for a retail catalog where:

- A **T-Shirt** needs `size`, `color`, `material`, `fit`
- A **Laptop** needs `screen_size`, `ram`, `processor`, `storage`
- A **Book** needs `author`, `isbn`, `publisher`, `page_count`

With a flat table, you'd add all possible columns and leave most NULL — a sparse, unmanageable schema. Or you'd create separate tables per product type — dozens of tables with no shared structure.

### EAV's Solution: Three-Table Pattern

EAV solves this by splitting storage into three concerns:

```
┌──────────────────┐     ┌──────────────────────┐     ┌─────────────────────┐
│  eav_attribute   │────▶│  eav_attribute_set    │     │  catalog_product_    │
│  (metadata)      │     │  (grouping)           │     │  entity (core row)   │
└──────────────────┘     └──────────────────────┘     └─────────────────────┘
        │                                                  │
        │         ┌──────────────────────────────────────┤
        │         │                                      │
        ▼         ▼                                      ▼
┌──────────────────────────────┐    ┌─────────────────────────────┐
│  catalog_product_entity_varchar  │    │  catalog_product_entity_int     │
│  entity_id | attr_id | value  │    │  entity_id | attr_id | value  │
└──────────────────────────────┘    └─────────────────────────────┘
```

1. **`eav_attribute`** — defines what attributes exist: their names, data types, validation rules, input handlers, and display labels
2. **`eav_attribute_set`** — groups attributes into named sets (e.g., "Default", "Clothing", "Electronics") so admin users can assign attribute sets to product types
3. **`catalog_product_entity_*`** (the value tables) — stores the actual values, keyed by `(entity_id, attribute_id)`

### Why Not Just JSON in a Text Column?

PostgreSQL's `jsonb` or MySQL 5.7+ JSON columns could store flexible attributes. But EAV in Magento 2.4.8 serves several purposes JSON cannot:

| Requirement | EAV | JSON Column |
|-------------|-----|-------------|
| Admin-managed attribute creation | Yes — `eav_attribute` rows, UI in admin | No — requires code changes |
| Per-attribute indexing | Yes — each type table has indexes on `(attribute_id, value)` | Limited — JSON indexes are broad |
| Attribute-set grouping | Yes — via `eav_attribute_set` + `eav_attribute_group` | No |
| Different input types per attribute | Yes — `varchar`, `int`, `decimal`, `text`, `datetime`, `gallery` | One column handles all types |
| Existing admin UI | Yes — built into Magento admin | No |
| Grid filtering and sorting | Yes — works with standard collection filtering | Complex custom SQL |

### Multi-Entity-Type Attribute Sets

The attribute-set concept deserves special attention. In the Magento admin, every product belongs to an **attribute set** — a named grouping of attributes. The "Default" attribute set might have 30 attributes. A "Clothing" attribute set inherits from Default and adds `size` and `color`.

Internally, this works through `eav_attribute_set` and `eav_attribute_group`:

```
eav_attribute_set
├── attribute_set_id: 4
├── entity_type_id: 4  (catalog_product)
└── attribute_set_name: "Clothing"

eav_attribute_group
├── attribute_group_id: 20
├── attribute_set_id: 4
└── attribute_group_name: "Basic Settings"

eav_entity_attribute
├── entity_type_id: 4
├── attribute_set_id: 4
├── attribute_group_id: 20
└── attribute_id: 134  (the 'size' attribute)
```

When a new product is created, the admin selects an attribute set. Magento uses the attribute set to determine which attributes appear in the product form and which values to load from which EAV value tables.

### Schema Flexibility in Practice

Consider a store that sells both books and apparel. In the admin:

1. Create a **Books** attribute set (inherits Default) + adds `isbn`, `author`, `publisher`
2. Create a **Apparel** attribute set (inherits Default) + adds `size`, `color`, `material`
3. When adding a new product, select the appropriate attribute set

The MySQL storage naturally accommodates this — a book product has rows in `varchar` for `isbn` and `author`, while a clothing product has rows in `varchar` for `size` and `color`. Both share the same `entity_id` core table and the same set of EAV value tables.

> **Why this matters for extensions:** Custom modules that add product attributes (e.g., a loyalty points module adding `pointsEarned` to every product) must use EAV — adding columns to `catalog_product_entity` would bypass Magento's attribute management system entirely, breaking admin UI, import/export, and indexers.

---

## 2. The 6 Core EAV Entities

Magento 2.4.8 has six formally EAV-backed entity types (though only the first four are truly EAV in current versions):

| Entity | Type Code | EAV Tables | Notes |
|--------|-----------|-------------|-------|
| `catalog_product` | `4` | Yes — full EAV | Primary product entity |
| `catalog_category` | `3` | Yes — full EAV | Category attributes |
| `customer` | `1` | Yes — full EAV | Customer entity |
| `customer_address` | `2` | Yes — full EAV | Customer address |
| `order` | `5` | **Flat now** | Historically EAV; flattened in 2.0 |
| `invoice`/`shipment`/`creditmemo` | `6`/`7`/`8` | **Flat now** | Historically EAV; flattened |

### catalog_product Entity

**Type ID: `4`**

The most complex EAV entity. Products have:

- ~30 default attributes (`name`, `description`, `price`, `weight`, `status`, `visibility`, `url_key`, etc.)
- Store-scoped values (`SCOPE_WEBSITE`, `SCOPE_STORE`) — same product can have different prices per website
- Multiple attribute sets (configurable products use their own attribute set to define configurable attributes)
- Flat table indexing via `catalog_product_flat_1` etc. (see Section 11)

**Core entity table:** `catalog_product_entity`
**Value tables:**
- `catalog_product_entity_varchar` — string values (name, description, url_key)
- `catalog_product_entity_int` — integer values (status, visibility, stock count)
- `catalog_product_entity_decimal` — decimal values (price, weight, special_price)
- `catalog_product_entity_text` — long text (description, meta_description)
- `catalog_product_entity_datetime` — date/time values (news_from_date, custom_design_from)
- `catalog_product_entity_gallery` — media gallery images

### catalog_category Entity

**Type ID: `3`**

Simpler than products. Categories have:
- ~15 default attributes (`name`, `description`, `parent_id`, `path`, `is_active`, `include_in_menu`, `image`, `thumbnail`)
- Global scope only — category attributes are not website-scoped
- Single attribute set — categories do not use attribute sets (unlike products)

**Core entity table:** `catalog_category_entity`
**Value tables:**
- `catalog_category_entity_varchar`
- `catalog_category_entity_int`
- `catalog_category_entity_decimal`
- `catalog_category_entity_text`
- `catalog_category_entity_datetime`

### customer Entity

**Type ID: `1`**

Customer accounts have:
- Core attributes from `customer_entity` (email, firstname, lastname, created_at)
- EAV attributes stored across `customer_entity_*` value tables
- **No attribute sets** — unlike products, customers do not use attribute sets
- Website-scoped — customer attributes can be scoped to specific websites in multi-store setups
- Address EAV separate — `customer_address` (type ID `2`) handles address attributes

**Core entity table:** `customer_entity`
**Value tables:**
- `customer_entity_varchar`
- `customer_entity_int`
- `customer_entity_decimal`
- `customer_entity_text`
- `customer_entity_datetime`

### customer_address Entity

**Type ID: `2`**

Customer address attributes:
- Default address attributes: `firstname`, `lastname`, `street`, `city`, `region`, `postcode`, `country_id`, `telephone`
- All address attributes are EAV — stored across the customer_address value tables
- Same attribute value tables as `customer` (they share the column type structure)

**Core entity table:** `customer_address_entity`
**Value tables:**
- `customer_entity_varchar` (shared with customer)
- `customer_entity_int` (shared with customer)
- `customer_entity_decimal` (shared with customer)
- `customer_entity_text` (shared with customer)
- `customer_entity_datetime` (shared with customer)

### order / invoice / shipment / creditmemo

These were EAV in early Magento 2.x versions. As of 2.4.x, they use flat tables (`sales_order`, `sales_invoice`, etc.) with fixed columns. They're listed here because you may encounter EAV references to these entities in older code or upgrade migration scripts.

### Type ID Registration

Every EAV entity is registered in `eav_entity_type`:

```sql
SELECT entity_type_id, entity_type_code, entity_model, attribute_model
FROM eav_entity_type
WHERE entity_type_code IN ('catalog_product', 'catalog_category', 'customer', 'customer_address');
```

```
entity_type_id: 1  entity_type_code: customer          entity_model: Magento\Customer\Model\ResourceModel\Customer
entity_type_id: 2  entity_type_code: customer_address  entity_model: Magento\Customer\Model\ResourceModel\Address
entity_type_id: 3  entity_type_code: catalog_category entity_model: Magento\Catalog\Model\ResourceModel\Category
entity_type_id: 4  entity_type_code: catalog_product  entity_model: Magento\Catalog\Model\ResourceModel\Product
entity_type_id: 5  entity_type_code: order             entity_model: Magento\Sales\Model\ResourceModel\Order
```

The `attribute_model` column optionally points to a custom attribute model class that overrides default backend/source/frontend behavior for all attributes of that entity type.

---

## 3. EAV Table Structure

Understanding the physical table layout is critical for debugging, writing raw SQL for migrations, and understanding performance characteristics.

### Core Entity Tables

Each EAV entity has a **core entity table** that stores the primary record with its own identity and shared/core fields:

```sql
-- catalog_product core table
CREATE TABLE catalog_product_entity (
    entity_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    attribute_set_id INT UNSIGNED NOT NULL DEFAULT 0,
    type_id VARCHAR(32) NOT NULL DEFAULT 'simple',
    sku VARCHAR(64) NULL,
    has_options INT NOT NULL DEFAULT 0,
    required_options INT NOT NULL DEFAULT 0,
    created_at DATETIME NOT NULL DEFAULT '0000-00-00 00:00:00',
    updated_at DATETIME NOT NULL DEFAULT '0000-00-00 00:00:00',
    PRIMARY KEY (entity_id)
) ENGINE=InnoDB;

-- catalog_category core table
CREATE TABLE catalog_category_entity (
    entity_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    entity_type_id INT UNSIGNED NOT NULL DEFAULT 3,
    attribute_set_id INT UNSIGNED NOT NULL DEFAULT 0,
    parent_id INT UNSIGNED NOT NULL DEFAULT 0,
    path VARCHAR(255) NOT NULL DEFAULT '',
    position INT NOT NULL DEFAULT 0,
    level INT NOT NULL DEFAULT 0,
    children_count INT NOT NULL DEFAULT 0,
    created_at DATETIME NOT NULL DEFAULT '0000-00-00 00:00:00',
    updated_at DATETIME NOT NULL DEFAULT '0000-00-00 00:00:00',
    PRIMARY KEY (entity_id)
) ENGINE=InnoDB;

-- customer core table
CREATE TABLE customer_entity (
    entity_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    entity_type_id INT UNSIGNED NOT NULL DEFAULT 1,
    website_id SMALLINT UNSIGNED NULL,
    email VARCHAR(255) NULL,
    group_id SMALLINT NOT NULL DEFAULT 1,
    increment_id VARCHAR(50) NULL,
    store_id SMALLINT UNSIGNED NULL DEFAULT 0,
    created_at DATETIME NOT NULL DEFAULT '0000-00-00 00:00:00',
    updated_at DATETIME NOT NULL DEFAULT '0000-00-00 00:00:00',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    PRIMARY KEY (entity_id)
) ENGINE=InnoDB;
```

### EAV Attribute Metadata Tables

The attribute definition system lives in three tables:

```sql
-- Central attribute definition table
CREATE TABLE eav_attribute (
    attribute_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    entity_type_id INT UNSIGNED NOT NULL DEFAULT 0,
    attribute_code VARCHAR(255) NOT NULL,
    attribute_model VARCHAR(255) NULL,
    backend_model VARCHAR(255) NULL,
    backend_type VARCHAR(255) NULL DEFAULT 'static',
    backend_table VARCHAR(255) NULL,
    frontend_model VARCHAR(255) NULL,
    frontend_input VARCHAR(255) NULL,
    frontend_label VARCHAR(255) NULL,
    frontend_class VARCHAR(255) NULL,
    source_model VARCHAR(255) NULL,
    is_required BOOLEAN NOT NULL DEFAULT FALSE,
    is_user_defined BOOLEAN NOT NULL DEFAULT FALSE,
    default_value TEXT NULL,
    unique_key VARCHAR(255) NULL,
    note VARCHAR(255) NULL,
    UNIQUE KEY `UNQ_EAV_ATTRIBUTE_ENTITY_TYPE_ATTRIBUTE_CODE` (entity_type_id, attribute_code),
    KEY `IDX_EAV_ATTRIBUTE_ENTITY_TYPE` (entity_type_id)
) ENGINE=InnoDB;

-- Groups attributes into sets
CREATE TABLE eav_attribute_set (
    attribute_set_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    entity_type_id INT UNSIGNED NOT NULL DEFAULT 0,
    attribute_set_name VARCHAR(255) NOT NULL,
    attribute_set_code VARCHAR(255) NOT NULL,
    sort_order INT NOT NULL DEFAULT 0,
    UNIQUE KEY `UNQ_EAV_ATTRIBUTE_SET_ENTITY_TYPE_ATTRIBUTE_SET_NAME` (entity_type_id, attribute_set_name),
    KEY `IDX_EAV_ATTRIBUTE_SET_ENTITY_TYPE` (entity_type_id)
) ENGINE=InnoDB;

-- Groups attributes within a set
CREATE TABLE eav_attribute_group (
    attribute_group_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    attribute_set_id INT UNSIGNED NOT NULL DEFAULT 0,
    attribute_group_name VARCHAR(255) NOT NULL,
    attribute_group_code VARCHAR(255) NOT NULL,
    sort_order INT NOT NULL DEFAULT 0,
    default_id INT UNSIGNED NOT NULL DEFAULT 0,
    UNIQUE KEY `UNQ_EAV_ATTRIBUTE_GROUP_ATTRIBUTE_SET_ATTRIBUTE_GROUP_NAME`
        (attribute_set_id, attribute_group_name),
    KEY `IDX_EAV_ATTRIBUTE_GROUP_ATTRIBUTE_SET` (attribute_set_id)
) ENGINE=InnoDB;

-- Maps attributes to groups within sets
CREATE TABLE eav_entity_attribute (
    entity_attribute_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    entity_type_id INT UNSIGNED NOT NULL DEFAULT 0,
    attribute_set_id INT UNSIGNED NOT NULL DEFAULT 0,
    attribute_group_id INT UNSIGNED NOT NULL DEFAULT 0,
    attribute_id INT UNSIGNED NOT NULL DEFAULT 0,
    UNIQUE KEY `UNQ_EAV_ENTITY_ATTRIBUTE` (attribute_set_id, attribute_id),
    KEY `IDX_EAV_ENTITY_ATTRIBUTE_ATTRIBUTE_SET` (attribute_set_id),
    KEY `IDX_EAV_ENTITY_ATTRIBUTE_ATTRIBUTE` (attribute_id)
) ENGINE=InnoDB;
```

### EAV Value Tables (for catalog_product)

Each data type has its own value table:

```sql
-- Common structure shared by all value tables:
-- (entity_id, attribute_id, value) + optional extra columns per type

CREATE TABLE catalog_product_entity_varchar (
    value_id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    attribute_id INT UNSIGNED NOT NULL DEFAULT 0,
    store_id INT UNSIGNED NOT NULL DEFAULT 0,
    entity_id INT UNSIGNED NOT NULL DEFAULT 0,
    value VARCHAR(255) NULL,
    UNIQUE KEY `UNQ_CAT_PRD_ENT_VARCHAR_ENTITY_ATTRIBUTE_STORE`
        (entity_id, attribute_id, store_id),
    KEY `IDX_CAT_PRD_ENT_VARCHAR_ATTRIBUTE_ID` (attribute_id),
    KEY `IDX_CAT_PRD_ENT_VARCHAR_STORE_ID` (store_id),
    KEY `IDX_CAT_PRD_ENT_VARCHAR_VALUE` (value)
) ENGINE=InnoDB;

CREATE TABLE catalog_product_entity_int (
    value_id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    attribute_id INT UNSIGNED NOT NULL DEFAULT 0,
    store_id INT UNSIGNED NOT NULL DEFAULT 0,
    entity_id INT UNSIGNED NOT NULL DEFAULT 0,
    value INT NULL,
    UNIQUE KEY `UNQ_CAT_PRD_ENT_INT_ENTITY_ATTRIBUTE_STORE`
        (entity_id, attribute_id, store_id),
    KEY `IDX_CAT_PRD_ENT_INT_ATTRIBUTE_ID` (attribute_id),
    KEY `IDX_CAT_PRD_ENT_INT_STORE_ID` (store_id),
    KEY `IDX_CAT_PRD_ENT_INT_VALUE` (value)
) ENGINE=InnoDB;

CREATE TABLE catalog_product_entity_decimal (
    value_id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    attribute_id INT UNSIGNED NOT NULL DEFAULT 0,
    store_id INT UNSIGNED NOT NULL DEFAULT 0,
    entity_id INT UNSIGNED NOT NULL DEFAULT 0,
    value DECIMAL(12,4) NULL,
    UNIQUE KEY `UNQ_CAT_PRD_ENT_DECIMAL_ENTITY_ATTRIBUTE_STORE`
        (entity_id, attribute_id, store_id),
    KEY `IDX_CAT_PRD_ENT_DECIMAL_ATTRIBUTE_ID` (attribute_id),
    KEY `IDX_CAT_PRD_ENT_DECIMAL_STORE_ID` (store_id),
    KEY `IDX_CAT_PRD_ENT_DECIMAL_VALUE` (value)
) ENGINE=InnoDB;

CREATE TABLE catalog_product_entity_text (
    value_id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    attribute_id INT UNSIGNED NOT NULL DEFAULT 0,
    store_id INT UNSIGNED NOT NULL DEFAULT 0,
    entity_id INT UNSIGNED NOT NULL DEFAULT 0,
    value TEXT NULL,
    UNIQUE KEY `UNQ_CAT_PRD_ENT_TEXT_ENTITY_ATTRIBUTE_STORE`
        (entity_id, attribute_id, store_id),
    KEY `IDX_CAT_PRD_ENT_TEXT_ATTRIBUTE_ID` (attribute_id),
    KEY `IDX_CAT_PRD_ENT_TEXT_STORE_ID` (store_id)
) ENGINE=InnoDB;

CREATE TABLE catalog_product_entity_datetime (
    value_id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    attribute_id INT UNSIGNED NOT NULL DEFAULT 0,
    store_id INT UNSIGNED NOT NULL DEFAULT 0,
    entity_id INT UNSIGNED NOT NULL DEFAULT 0,
    value DATETIME NULL,
    UNIQUE KEY `UNQ_CAT_PRD_ENT_DATETIME_ENTITY_ATTRIBUTE_STORE`
        (entity_id, attribute_id, store_id),
    KEY `IDX_CAT_PRD_ENT_DATETIME_ATTRIBUTE_ID` (attribute_id),
    KEY `IDX_CAT_PRD_ENT_DATETIME_STORE_ID` (store_id),
    KEY `IDX_CAT_PRD_ENT_DATETIME_VALUE` (value)
) ENGINE=InnoDB;

CREATE TABLE catalog_product_entity_gallery (
    value_id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    attribute_id INT UNSIGNED NOT NULL DEFAULT 0,
    store_id INT UNSIGNED NOT NULL DEFAULT 0,
    entity_id INT UNSIGNED NOT NULL DEFAULT 0,
    value VARCHAR(255) NULL,
    PRIMARY KEY (value_id),
    KEY `IDX_CAT_PRD_ENT_GALLERY_ENTITY_ATTRIBUTE` (entity_id, attribute_id),
    KEY `IDX_CAT_PRD_ENT_GALLERY_ATTRIBUTE_ID` (attribute_id)
) ENGINE=InnoDB;
```

### Customer EAV Value Tables

Customer value tables are shared between `customer` and `customer_address` entities:

```sql
CREATE TABLE customer_entity_varchar (
    value_id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    attribute_id INT UNSIGNED NOT NULL DEFAULT 0,
    entity_id INT UNSIGNED NOT NULL DEFAULT 0,
    value VARCHAR(255) NULL,
    UNIQUE KEY `UNQ_CUSTOMER_ENTITY_VARCHAR_ENTITY_ATTRIBUTE`
        (entity_id, attribute_id),
    KEY `IDX_CUSTOMER_ENTITY_VARCHAR_ATTRIBUTE_ID` (attribute_id)
) ENGINE=InnoDB;

CREATE TABLE customer_entity_int (
    value_id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    attribute_id INT UNSIGNED NOT NULL DEFAULT 0,
    entity_id INT UNSIGNED NOT NULL DEFAULT 0,
    value INT NULL,
    UNIQUE KEY `UNQ_CUSTOMER_ENTITY_INT_ENTITY_ATTRIBUTE`
        (entity_id, attribute_id),
    KEY `IDX_CUSTOMER_ENTITY_INT_ATTRIBUTE_ID` (attribute_id)
) ENGINE=InnoDB;
-- (decimal, text, datetime follow the same pattern)
```

> **Note:** Customer value tables do **not** have a `store_id` column — customer data is global, not scoped to store view (unlike products).

### How the Tables Join

Loading a single product with its EAV attributes generates a query like:

```sql
-- Simplified: loading product ID 42, attribute 'color' (varchar) and 'price' (decimal)
SELECT p.entity_id, p.sku, p.type_id,
       vv.value AS color,
       dv.value AS price
FROM catalog_product_entity p
LEFT JOIN catalog_product_entity_varchar vv
    ON p.entity_id = vv.entity_id
    AND vv.attribute_id = (
        SELECT attribute_id FROM eav_attribute
        WHERE attribute_code = 'color' AND entity_type_id = 4
    )
    AND vv.store_id = 0
LEFT JOIN catalog_product_entity_decimal dv
    ON p.entity_id = dv.entity_id
    AND dv.attribute_id = (
        SELECT attribute_id FROM eav_attribute
        WHERE attribute_code = 'price' AND entity_type_id = 4
    )
    AND dv.store_id = 0
WHERE p.entity_id = 42;
```

In practice, Magento's `Magento\Eav\Model\Entity\Collection\AbstractCollection` (Section 8) generates these joins automatically and handles the multi-store scoping logic.

---

## 4. Attribute Backend Models

Backend models determine how attribute values are stored, loaded, and validated. Every attribute can optionally declare a `backend_model` — a class that handles persistence operations.

### DefaultBackend

`Magento\Eav\Model\Entity\Attribute\Backend\DefaultBackend` is the fallback for attributes without an explicit backend model. It provides minimal behavior — just stores and retrieves the raw value:

```php
<?php
// Magento\Eav\Model\Entity\Attribute\Backend\DefaultBackend
namespace Magento\Eav\Model\Entity\Attribute\Backend;

class DefaultBackend extends \Magento\Eav\Model\Entity\Attribute\Backend\AbstractBackend
{
    public function afterLoad($object)
    {
        $attributeCode = $this->getAttribute()->getAttributeCode();
        $object->setData($attributeCode, $object->getData($attributeCode));
        return $this;
    }
}
```

### AbstractBackend

`Magento\Eav\Model\Entity\Attribute\Backend\AbstractBackend` is the base class for all backend models. Key methods:

```php
<?php
namespace Magento\Eav\Model\Entity\Attribute\Backend;

abstract class AbstractBackend
{
    protected $attribute;

    public function setAttribute(\Magento\Eav\Model\Entity\Attribute $attribute): void
    {
        $this->attribute = $attribute;
    }

    public function getAttribute(): \Magento\Eav\Model\Entity\Attribute
    {
        return $this->attribute;
    }

    /**
     * Called when entity is saved — return false to prevent save
     */
    public function beforeSave($object): self
    {
        return $this;
    }

    /**
     * Called after entity save completes
     */
    public function afterSave($object): self
    {
        return $this;
    }

    /**
     * Called when entity is loaded — populate attribute value
     */
    public function afterLoad($object): self
    {
        return $this;
    }

    /**
     * Called before entity delete
     */
    public function beforeDelete($object): self
    {
        return $this;
    }

    /**
     * Validate attribute value — return true/false or throw exception
     */
    public function validate($object): bool
    {
        return true;
    }
}
```

### Common Backend Model Examples

**`Magento\Catalog\Model\Product\Attribute\Backend\Price`**

Handles price rounding and decimal precision:

```php
<?php
// Magento\Catalog\Model\Product\Attribute\Backend\Price
class Price extends \Magento\Eav\Model\Entity\Attribute\Backend\AbstractBackend
{
    public function beforeSave($object): self
    {
        $price = $object->getData($this->getAttribute()->getAttributeCode());
        if ($price !== null) {
            // Round to 4 decimal places
            $object->setData(
                $this->getAttribute()->getAttributeCode(),
                round((float) $price, 4)
            );
        }
        return $this;
    }
}
```

**`Magento\Catalog\Model\Product\Attribute\Backend\Image`**

Handles media image upload and validation:

```php
<?php
// Magento\Catalog\Model\Product\Attribute\Backend\Image
class Image extends \Magento\Eav\Model\Entity\Attribute\Backend\AbstractBackend
{
    public function beforeSave($object): self
    {
        $attrCode = $this->getAttribute()->getAttributeCode();
        $value = $object->getData($attrCode);

        if ($this->getAttribute()->getData('scope') === 'gallery') {
            // Validate image file exists and is readable
        }

        return $this;
    }
}
```

**`Magento\Eav\Model\Entity\Attribute\Backend\Datetime`**

Handles timezone-aware date conversion between UTC storage and local display:

```php
<?php
// Magento\Eav\Model\Entity\Attribute\Backend\Datetime
class Datetime extends \Magento\Eav\Model\Entity\Attribute\Backend\AbstractBackend
{
    public function beforeSave($object): self
    {
        $attrCode = $this->getAttribute()->getAttributeCode();
        $date = $object->getData($attrCode);

        if ($date) {
            // Convert to UTC before storing
            $object->setData(
                $attrCode,
                $this->_localeDate->convertConfigTimeToUtc($date)
            );
        }

        return $this;
    }
}
```

### How catalog_product_entity Links to Backend Class

The link is made via `eav_attribute.backend_model`:

```sql
SELECT attribute_id, attribute_code, backend_model, backend_type
FROM eav_attribute
WHERE entity_type_id = 4
LIMIT 10;
```

```
attribute_id: 79   attribute_code: 'name'           backend_model: NULL               backend_type: varchar
attribute_id: 80   attribute_code: 'description'     backend_model: NULL               backend_type: text
attribute_id: 81   attribute_code: 'price'            backend_model: Magento\Catalog\Model\Product\Attribute\Backend\Price   backend_type: decimal
attribute_id: 82   attribute_code: 'image'           backend_model: Magento\Catalog\Model\Product\Attribute\Backend\Image backend_type: varchar
attribute_id: 83   attribute_code: 'news_from_date'  backend_model: Magento\Eav\Model\Entity\Attribute\Backend\Datetime   backend_type: datetime
```

When `Magento\Catalog\Model\ResourceModel\Product::save()` runs, it iterates over all attributes with values and calls:

```php
$backendModel = $this->getAttribute($attrCode)->getBackend();
$backendModel->beforeSave($this);
```

The `backend_type` column in `eav_attribute` determines which value table is used:
- `varchar` → `catalog_product_entity_varchar`
- `int` → `catalog_product_entity_int`
- `decimal` → `catalog_product_entity_decimal`
- `text` → `catalog_product_entity_text`
- `datetime` → `catalog_product_entity_datetime`
- `static` → stored directly in the entity's core table (e.g., `catalog_product_entity.sku`)

---

## 5. Attribute Source Models

Source models provide the option lists that power dropdown, multiselect, and boolean attributes. When an attribute has `input = "select"` or `input = "multiselect"`, Magento loads a source model to populate the admin dropdown options.

### Table Source

`Magento\Eav\Model\Entity\Attribute\Source\Table` is the most commonly used source model — it reads options from the EAV value tables (for options defined via `eav_attribute_option`):

```php
<?php
// Magento\Eav\Model\Entity\Attribute\Source\Table
namespace Magento\Eav\Model\Entity\Attribute\Source;

class Table extends \Magento\Eav\Model\Entity\Attribute\Source\AbstractSource
{
    // Reads options from eav_attribute_option table
}
```

Options are stored in:

```sql
CREATE TABLE eav_attribute_option (
    option_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    attribute_id INT UNSIGNED NOT NULL DEFAULT 0,
    sort_order INT NOT NULL DEFAULT 0,
    UNIQUE KEY `UNQ_EAV_ATTRIBUTE_OPTION_ATTRIBUTE_ID_OPTION_ID` (attribute_id, option_id),
    KEY `IDX_EAV_ATTRIBUTE_OPTION_ATTRIBUTE_ID` (attribute_id)
) ENGINE=InnoDB;

CREATE TABLE eav_attribute_option_value (
    option_id INT UNSIGNED NOT NULL DEFAULT 0,
    store_id SMALLINT UNSIGNED NOT NULL DEFAULT 0,
    value VARCHAR(255) NULL,
    UNIQUE KEY `UNQ_EAV_ATTRIBUTE_OPTION_VALUE_OPTION_ID_STORE_ID` (option_id, store_id)
) ENGINE=InnoDB;
```

### OptionSourceInterface

All source models implement `Magento\Eav\Model\Entity\Attribute\Source\OptionSourceInterface`:

```php
<?php
namespace Magento\Eav\Model\Entity\Attribute\Source;

interface OptionSourceInterface
{
    /**
     * Get all options for the attribute — used in admin grid filtering
     */
    public function getAllOptions(bool $withEmpty = false): array;

    /**
     * Get option text by value — used in forms for display
     */
    public function getOptionText(?int $value): mixed;
}
```

### getOptionText() vs getAllOptions()

These two methods serve different purposes:

| Method | Purpose | Called By |
|--------|---------|-----------|
| `getAllOptions()` | Returns full `[['value' => ..., 'label' => ...], ...]` array | Admin grid column filters, `addAttributeToFilter()` in collections |
| `getOptionText()` | Returns single label string for a given value | Admin product form display, `getFormattedOptionValue()` |

**Example — Boolean source model:**

```php
<?php
// Magento\Eav\Model\Entity\Attribute\Source\Boolean
namespace Magento\Eav\Model\Entity\Attribute\Source;

class Boolean implements OptionSourceInterface
{
    public function getAllOptions(bool $withEmpty = false): array
    {
        $options = [
            ['value' => '0', 'label' => __('No')],
            ['value' => '1', 'label' => __('Yes')],
        ];

        if ($withEmpty) {
            array_unshift($options, ['value' => '', 'label' => __('Please Select')]);
        }

        return $options;
    }

    public function getOptionText(?int $value): mixed
    {
        $options = $this->getAllOptions();
        foreach ($options as $option) {
            if ($option['value'] == $value) {
                return $option['label'];
            }
        }
        return null;
    }
}
```

### Custom Source Model Example

A custom source model for a loyalty tier attribute:

```php
<?php
// app/code/Vendor/Module/Model/Attribute/Source/LoyaltyTier.php
declare(strict_types=1);

namespace Vendor\Module\Model\Attribute\Source\LoyaltyTier;

use Magento\Eav\Model\Entity\Attribute\Source\AbstractSource;

class LoyaltyTier extends AbstractSource
{
    public function getAllOptions(bool $withEmpty = false): array
    {
        if ($this->_options === null) {
            $this->_options = [
                ['value' => 0, 'label' => __('Standard')],
                ['value' => 1, 'label' => __('Silver')],
                ['value' => 2, 'label' => __('Gold')],
                ['value' => 3, 'label' => __('Platinum')],
            ];
        }

        $options = $this->_options;
        if ($withEmpty) {
            array_unshift($options, ['value' => '', 'label' => __('Select Loyalty Tier')]);
        }

        return $options;
    }
}
```

Register the source model in your attribute setup patch:

```php
$eavSetup->addAttribute(
    \Magento\Customer\Model\Customer::ENTITY,
    'loyalty_tier',
    [
        'type' => 'int',
        'label' => 'Loyalty Tier',
        'input' => 'select',
        'source' => \Vendor\Module\Model\Attribute\Source\LoyaltyTier::class,
        'required' => false,
        'user_defined' => true,
    ]
);
```

### AbstractSource Base Class

`Magento\Eav\Model\Entity\Attribute\Source\AbstractSource` provides caching:

```php
<?php
namespace Magento\Eav\Model\Entity\Attribute\Source;

abstract class AbstractSource implements OptionSourceInterface
{
    protected $_options = null;

    public function getAllOptions(bool $withEmpty = false): array
    {
        // Implemented by concrete classes
    }

    public function getOptionText(?int $value): mixed
    {
        // Default implementation calls getAllOptions() and finds match
        // Subclasses override for performance
    }
}
```

The `_options` property is cached — once `getAllOptions()` is called, subsequent calls return the cached array without re-computing. This matters in collections where the source model is invoked per row.

---

## 6. Attribute Frontend Models

Frontend models control how attributes render in the admin and storefront — label display, CSS classes, input-rendering HTML, and input validation display.

### FrontendLabel

`Magento\Eav\Model\Entity\Attribute\Frontend\FrontendLabel` is the default frontend model used by most attributes for label resolution with store scope:

```php
<?php
// Magento\Eav\Model\Entity\Attribute\Frontend\FrontendLabel
namespace Magento\Eav\Model\Entity\Attribute\Frontend;

class FrontendLabel extends \Magento\Eav\Model\Entity\Attribute\Frontend\AbstractFrontend
{
    public function getLabel(\Magento\Framework\DataObject $object): string
    {
        // Returns the store-scoped label for the attribute
        // e.g., "Color" in English store, "Couleur" in French store
    }
}
```

### AbstractFrontend

```php
<?php
namespace Magento\Eav\Model\Entity\Attribute\Frontend;

abstract class AbstractFrontend
{
    protected $attribute;

    public function setAttribute(\Magento\Eav\Model\Entity\Attribute $attribute): void
    {
        $this->attribute = $attribute;
    }

    /**
     * Called when attribute value is rendered in forms
     */
    public function getLabel(): string
    {
        return $this->attribute->getFrontendLabel();
    }

    /**
     * CSS class applied to form input in admin
     */
    public function getClass(): string
    {
        return $this->attribute->getFrontendClass();
    }

    /**
     * Input type — text, select, textarea, multilselect, etc.
     */
    public function getInputType(): string
    {
        return $this->attribute->getFrontendInput();
    }
}
```

### How Frontend Affects Rendering

The frontend model controls what HTML the admin form generates:

```php
// In Magento\Adminhtml\Block\Catalog\Product\Edit\Tab\Attributes
// The form block uses the attribute's frontend model to render each field

$attribute->getFrontend()->getInputType();
// Returns: 'text', 'textarea', 'select', 'multiselect', 'date', 'boolean', 'price', 'image', etc.

$attribute->getFrontend()->getClass();
// Returns: CSS class string used on the form input element
```

Common frontend input mappings:

| `frontend_input` | Form Element Rendered |
|-----------------|----------------------|
| `text` | `<input type="text">` |
| `textarea` | `<textarea>` |
| `select` | `<select>` (uses source model options) |
| `multiselect` | `<select multiple>` |
| `boolean` | `<select>` with Yes/No options |
| `date` | Calendar date picker |
| `price` | Currency-formatted input |
| `image` | File upload with preview |
| `gallery` | Media gallery multi-upload |

The admin form rendering uses `Magento\Framework\Data\Form` which maps `frontend_input` to a data element renderer. When you create a custom attribute with `input => 'multiselect'`, Magento automatically renders a multi-select dropdown using the source model's `getAllOptions()`.

---

## 7. Attribute Validation

Validation rules are defined in `eav_attribute.validation_rule` and processed by `Magento\Eav\Model\Entity\Attribute\ValidationRule`.

### ValidationRule Model

```php
<?php
// Magento\Eav\Model\Entity\Attribute\ValidationRule
namespace Magento\Eav\Model\Entity;

class ValidationRule
{
    private $validation;
    private $options = [];

    public function __construct(array $data)
    {
        $this->validation = $data['validation'] ?? '';
        $this->options = $data['options'] ?? [];
    }

    /**
     * Validate the value against this rule
     * Returns true if valid, false if invalid
     */
    public function validate(\Magento\Framework\Model\AbstractModel $object): bool
    {
        $value = $object->getData($this->attribute->getAttributeCode());
        return $this->validateValue($value);
    }

    private function validateValue($value): bool
    {
        switch ($this->validation) {
            case 'validate-email':
                return filter_var($value, FILTER_VALIDATE_EMAIL) !== false;
            case 'validate-url':
                return filter_var($value, FILTER_VALIDATE_URL) !== false;
            case 'validate-alphanumeric':
                return preg_match('/^[a-zA-Z0-9]+$/', $value) === 1;
            case 'validate-number-range':
                return $this->validateNumberRange($value);
            case 'validate-date':
                return strtotime($value) !== false;
            default:
                return true;
        }
    }
}
```

### input_validation Field

The `eav_attribute.input_validation` column stores the validation type for an attribute:

```sql
SELECT attribute_id, attribute_code, input_validation, validation_rule
FROM eav_attribute
WHERE entity_type_id = 4
AND input_validation IS NOT NULL;
```

```
attribute_id: 93   attribute_code: 'cost'          input_validation: 'validate-number'   validation_rule: NULL
attribute_id: 94   attribute_code: 'weight'        input_validation: 'validate-number'   validation_rule: NULL
attribute_id: 95   attribute_code: 'url_key'       input_validation: 'validate-url'      validation_rule: NULL
attribute_id: 96   attribute_code: 'email'         input_validation: 'validate-email'   validation_rule: NULL
```

Built-in validation types:

| `input_validation` value | Applied Rule | Example |
|------------------------|--------------|---------|
| `validate-email` | PHP `FILTER_VALIDATE_EMAIL` | Customer email |
| `validate-url` | PHP `FILTER_VALIDATE_URL` | External link |
| `validate-number` | Numeric only | `weight`, `cost` |
| `validate-date` | `strtotime()` parsable | `news_from_date` |
| `validate-maximum-length` | Max character count | Database-level limit |
| `validate-digits` | Digits only | Phone numbers |
| `validate-alpha` | Letters only | Name fields |
| `validate-alphanum` | Letters and numbers | `url_key` |

### Validation in the Admin Form

The admin form validates on submit using JavaScript validation rules rendered from the attribute configuration. For example, an email attribute with `input_validation = 'validate-email'` renders:

```html
<input type="text" id="attribute-123" class="required-entry validate-email"/>
```

The `validate-email` JavaScript class is defined in `mage/form/validator.js` and performs client-side validation matching the server-side PHP rules.

### Adding Custom Validation

Custom validation rules are registered in `validation_rules` array in `eav_attribute`:

```php
<?php
// In a Data Patch, adding a custom validation rule
$eavSetup->addAttribute(
    \Magento\Catalog\Model\Product::ENTITY,
    'custom_field',
    [
        'type' => 'varchar',
        'label' => 'Custom Field',
        'input' => 'text',
        'user_defined' => true,
        'input_validation' => 'custom_validation', // Must match a registered rule
    ]
);
```

Custom validation rules must be registered via `Magento\Framework\Validator\ValidationRule` in a custom validator builder. This is more advanced — for most use cases, use the built-in validation types.

---

## 8. EAV Collections

`Magento\Eav\Model\Entity\Collection\AbstractCollection` is the collection foundation for all EAV entities. It extends `Magento\Framework\Model\ResourceModel\Db\Collection\AbstractCollection` and adds EAV-specific logic for lazy attribute loading, filtering, and sorting.

### AbstractCollection Architecture

```php
<?php
// Magento\Eav\Model\Entity\Collection\AbstractCollection
namespace Magento\Eav\Model\Entity\Collection;

abstract class AbstractCollection
    extends \Magento\Framework\Model\ResourceModel\Db\Collection\AbstractCollection
{
    // Extends AbstractCollection with:
    // - addAttributeToSelect()
    // - addAttributeToFilter()
    // - addAttributeToSort()
    // - _renderFilters()
    // - _renderOrders()
    // - EAV-specific _beforeLoad()
}
```

### Key Collection Methods

**`addAttributeToSelect()`**

Adds EAV attributes to the SELECT clause, generating proper JOINs:

```php
<?php
$collection = $this->productCollectionFactory->create();
$collection->addAttributeToSelect(['name', 'price', 'status', 'description']);
// Generates JOINs to catalog_product_entity_varchar, catalog_product_entity_decimal, etc.
// Selects: e.entity_id, vv.value AS name, dv.value AS price, iv.value AS status
```

**`addAttributeToFilter()`**

Filters the collection by EAV attribute values:

```php
<?php
// Filter by status = 1
$collection->addAttributeToFilter('status', 1);

// Filter with condition type
$collection->addAttributeToFilter('price', ['gteq' => 100]);

// Filter by multiple attributes (AND)
$collection->addAttributeToFilter([
    ['attribute' => 'status', 'eq' => 1],
    ['attribute' => 'price', 'gt' => 50],
]);

// Filter with OR
$collection->addAttributeToFilter(
    [['attribute' => 'name', 'like' => 'A%'],
     ['attribute' => 'name', 'like' => 'B%']]
);
```

**`addAttributeToSort()`**

Sorts by EAV attribute:

```php
<?php
$collection->addAttributeToSort('name', 'ASC');
$collection->addAttributeToSort('price', 'DESC');
```

### The Lazy-Loading Pattern

EAV collections use **lazy loading** — attributes are not selected until they're needed. The collection maintains a `_selectAttributes` set tracking which attributes to load:

```php
<?php
// Simplified internal flow
public function load($printQuery = false, $logQuery = false): self
{
    // 1. If not loaded yet, call _beforeLoad()
    // 2. Build the SELECT with all registered _selectAttributes
    // 3. _renderFilters() — applies all addAttributeToFilter() calls
    // 4. Execute query — var_export($collection->getSelect()) to see final SQL
    // 5. _afterLoad() — populates objects with fetched data
    // 6. Mark as loaded
}
```

The flow for `addAttributeToFilter()`:

```php
<?php
public function addAttributeToFilter($attribute, $condition = null): self
{
    // 1. Normalize attribute name or ID
    // 2. Store condition in _attributesFilter[$attributeCode]
    // 3. NOT executed immediately — stored in a pending queue
    // 4. On load(), _renderFilters() generates JOINs and WHERE clauses
}
```

This means `addAttributeToFilter()` is idempotent before `load()` — you can build complex filter sets incrementally:

```php
<?php
$collection = $this->productCollectionFactory->create();
$collection->addAttributeToSelect('*');         // Queue attribute
$collection->addAttributeToFilter('status', 1); // Queue filter
$collection->addAttributeToFilter('price', ['gt' => 10]);
$collection->setPageSize(20);                   // Queue pagination
// Query NOT yet executed — all operations queued
// Query executes NOW when you iterate or call load()
foreach ($collection as $product) { /* first iteration triggers load() */ }
```

### Product Collection Specifics

`Magento\Catalog\Model\ResourceModel\Product\Collection` extends the EAV collection and adds product-specific logic:

```php
<?php
// Magento\Catalog\Model\ResourceModel\Product\Collection
namespace Magento\Catalog\Model\ResourceModel;

class Product\Collection extends \Magento\Eav\Model\Entity\Collection\AbstractCollection
{
    protected function _construct(): void
    {
        $this->_init(
            \Magento\Catalog\Model\Product::class,
            \Magento\Catalog\Model\ResourceModel\Product::class
        );
        $this->_setProductFlatTable(1); // Switch to flat table if enabled
    }

    /**
     * Add price filter — special handling for website scope
     */
    public function addPriceFilter(int $websiteId, float $from, float $to): self
    {
        // Joins price index tables
    }

    /**
     * Add category filter — joins catalog_category_product table
     */
    public function addCategoryFilter(\Magento\Catalog\Model\Category $category): self
    {
        // JOIN catalog_category_product
    }
}
```

### Store Scoping in Collections

Product collections handle store scoping automatically:

```php
<?php
// In a given store context, Magento renders:
// LEFT JOIN catalog_product_entity_varchar cv
//     ON p.entity_id = cv.entity_id
//     AND cv.attribute_id = 79
//     AND cv.store_id = CURRENT_STORE_ID
// LEFT JOIN catalog_product_entity_varchar cv_default
//     ON p.entity_id = cv_default.entity_id
//     AND cv.attribute_id = 79
//     AND cv_default.store_id = 0
//     AND cv.store_id IS NULL  -- fallback to default store if no store-specific value
```

When a store-specific value exists, it's used. Otherwise, the default store (`store_id = 0`) value is used. This is all handled transparently by the collection's `_renderFilters()` method.

---

## 9. The Product EAV

Product EAV is the most complex of Magento's EAV implementations. It includes configurable product handling, type-specific logic, and flat table fallback.

### Product Type Model

`Magento\Catalog\Model\Product` is the product entity model. It extends `Magento\Eav\Model\Entity\AbstractEntity` and configures the EAV type:

```php
<?php
// Magento\Catalog\Model\Product
namespace Magento\Catalog\Model;

class Product extends \Magento\Framework\Model\AbstractExtensibleModel
    implements \Magento\Catalog\Api\Data\ProductInterface
{
    const ENTITY = 'catalog_product';

    protected function _construct(): void
    {
        $this->_init(
            \Magento\Catalog\Model\ResourceModel\Product::class
        );
    }

    /**
     * Get product type ID — used to determine type-specific behavior
     */
    public function getTypeId(): string
    {
        return $this->getData('type_id');
    }

    /**
     * Check if product is a composite (configurable, grouped, bundle)
     */
    public function isComposite(): bool
    {
        return in_array($this->getTypeId(), [
            'configurable', 'grouped', 'bundle'
        ], true);
    }
}
```

### Type ID Handling

Products have a `type_id` column in `catalog_product_entity`:

```sql
SELECT entity_id, sku, type_id FROM catalog_product_entity LIMIT 10;
```

```
entity_id: 1   sku: 'WPH01'   type_id: 'simple'
entity_id: 2   sku: 'WPH02'   type_id: 'simple'
entity_id: 3   sku: 'CONFIG1' type_id: 'configurable'
entity_id: 4   sku: 'GROUP1'  type_id: 'grouped'
```

Magento handles each type differently:

| Type ID | Handler Class | Behavior |
|---------|---------------|----------|
| `simple` | `Simple` | Standard product, fully self-contained |
| `virtual` | `Virtual` | Like simple but no shipping weight |
| `downloadable` | `Downloadable` | Has downloadable links |
| `configurable` | `Configurable` | References simple products as variants |
| `grouped` | `Grouped` | Groups simple products with quantities |
| `bundle` | `Bundle` | Dynamically priced bundle of options |

### Configurable Product Special Handling

Configurable products are the most important EAV case — they use an attribute set that marks certain attributes as **configurable** (selectable by the customer):

```php
<?php
// A configurable product has attributes marked as 'is_configurable'
// In eav_attribute.is_configurable = 1 for attributes like 'color', 'size'

// When a configurable product is loaded, Magento:
// 1. Loads the standard EAV attributes
// 2. Queries child products (simple products with same super-attribute values)
// 3. Attaches the child products as $product->getTypeInstance()->getUsedProducts()
```

### Product Flat Tables (Indexer-Generated)

When flat tables are enabled (see Section 11), `catalog_product_flat_1` (store ID 1) is a pre-joined, pre-indexed version of the product EAV data. The product collection switches to querying this flat table instead of the EAV tables:

```php
<?php
// Magento\Catalog\Model\ResourceModel\Product\Collection::_construct()
protected function _construct(): void
{
    // ...
    if ($this->flatEnabled) {
        $this->_setProductFlatTable($this->getStoreId());
    }
}

// When flat is enabled, all addAttributeToSelect/filters work against
// catalog_product_flat_1 instead of EAV tables — transparent to caller
```

---

## 10. Customer EAV

Customer EAV differs from product EAV in three important ways: **no attribute sets**, **website scoping instead of store scoping**, and **separate address EAV**.

### Customer Entity Resource Model

```php
<?php
// Magento\Customer\Model\ResourceModel\Customer
namespace Magento\Customer\Model\ResourceModel;

class Customer extends \Magento\Eav\Model\Entity\AbstractExtensibleEntity
{
    protected function _construct(): void
    {
        $this->_init('customer_entity', 'entity_id');
    }

    /**
     * Load customer by email — handles website scope
     */
    public function loadByEmail(\Magento\Customer\Model\Customer $customer, string $email): self
    {
        $select = $this->_getReadAdapter()->select()
            ->from($this->getEntityTable())
            ->where('email = ?', $email)
            ->where('website_id = ?', $customer->getWebsiteId());

        // ...
    }
}
```

### Customer Address EAV

Customer addresses are stored in `customer_address_entity` (type ID `2`). Address attributes are stored in the same `customer_entity_*` value tables as customer attributes (they share the tables by having the same `backend_type` structure):

```sql
-- customer_entity_varchar stores BOTH customer and customer_address varchar values
-- distinguished by entity_id which references either customer_entity or customer_address_entity

SELECT
    ce.entity_id,
    ce.email,
    ae.entity_id AS address_id,
    ae.value AS street_value
FROM customer_entity ce
LEFT JOIN customer_entity_varchar ae_v
    ON ce.entity_id = ae_v.entity_id
LEFT JOIN customer_address_entity ae
    ON ae_v.entity_id = ae.entity_id
WHERE ...
```

The `customer_address_entity` has its own core attributes:

```sql
CREATE TABLE customer_address_entity (
    entity_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    entity_type_id INT UNSIGNED NOT NULL DEFAULT 2,
    increment_id VARCHAR(50) NULL,
    parent_id INT UNSIGNED NULL COMMENT 'Customer entity ID',
    created_at DATETIME NOT NULL DEFAULT '0000-00-00 00:00:00',
    updated_at DATETIME NOT NULL DEFAULT '0000-00-00 00:00:00',
    PRIMARY KEY (entity_id)
) ENGINE=InnoDB;
```

### Customer Attribute Sets

Customer EAV does **not** use attribute sets. In the Magento admin, there is no "attribute set" dropdown for customers. All customer attributes are global.

However, customer **addresses** technically share the attribute set mechanism through `eav_entity_attribute` — but Magento core doesn't populate it by default. Custom modules that add address attributes must manually insert into `eav_entity_attribute` if they want the attributes to appear in the admin address form.

### Customer Website Scoping

Customer attributes are scoped to **website** (not store view). The `website_id` column in `customer_entity` enables multi-website customer accounts:

```php
<?php
// Customer is visible only in its assigned website
public function beforeSave(): self
{
    // Validate website_id is set
    if ($this->getWebsiteId() === null) {
        throw new \Magento\Framework\Exception\LocalizedException(
            __('Website ID is required')
        );
    }
}
```

When loading a customer collection, website filtering is applied automatically based on the current store's website context.

---

## 11. EAV Indexer and Flat Tables

Flat tables exist because EAV query patterns — especially filtered catalog listing pages — are fundamentally slow without pre-joining all attributes into a single table.

### Why Flat Tables Exist

Consider a category listing page showing 20 products with filters on 5 attributes (price range, color, size, material, status). With pure EAV:

```sql
-- For each of 20 products × 5 attributes = 100 JOINs
SELECT p.entity_id, p.sku
FROM catalog_product_entity p
LEFT JOIN catalog_product_entity_decimal pd_price ON ...
LEFT JOIN catalog_product_entity_varchar pv_color ON ...
LEFT JOIN catalog_product_entity_varchar pv_size ON ...
LEFT JOIN catalog_product_entity_varchar pv_material ON ...
LEFT JOIN catalog_product_entity_int pi_status ON ...
WHERE p.entity_id IN (1,2,3,...,20)
AND pv_color.value = 'Blue'
AND pd_price.value BETWEEN 50 AND 100
AND pi_status.value = 1;
```

With flat table `catalog_product_flat_1`:

```sql
-- Single table, 0 JOINs, all values pre-joined
SELECT entity_id, sku, color, price, size, material, status
FROM catalog_product_flat_1
WHERE entity_id IN (1,2,3,...,20)
AND color = 'Blue'
AND price BETWEEN 50 AND 100
AND status = 1;
```

The flat table stores one row per product with all relevant attribute values as direct columns. This eliminates JOINs entirely.

### catalog_product_flat_N Tables

Flat tables are named `catalog_product_flat_N` where `N` is the **store ID**:

| Table | Store ID |
|-------|----------|
| `catalog_product_flat_1` | Store ID 1 |
| `catalog_product_flat_2` | Store ID 2 |
| `catalog_product_flat_3` | Store ID 3 |

Each flat table has columns for all attributes marked `used_in_product_listing = true` in `eav_attribute`:

```sql
-- Example catalog_product_flat_1 structure
CREATE TABLE catalog_product_flat_1 (
    entity_id INT UNSIGNED PRIMARY KEY,
    attribute_set_id INT UNSIGNED,
    type_id VARCHAR(32),
    sku VARCHAR(64),
    has_options INT DEFAULT 0,
    required_options INT DEFAULT 0,
    created_at DATETIME,
    updated_at DATETIME,

    -- EAV attributes as flat columns:
    name VARCHAR(255),
    description TEXT,
    short_description TEXT,
    price DECIMAL(12,4),
    special_price DECIMAL(12,4),
    status INT,
    visibility INT,
    url_key VARCHAR(255),
    thumbnail VARCHAR(255),
    -- ... all attributes with used_in_product_listing = true
    PRIMARY KEY (entity_id),
    KEY IDX_FLAT_CATALOG_PRODUCT_STATUS (status),
    KEY IDX_FLAT_CATALOG_PRODUCT_VISIBILITY (visibility),
    KEY IDX_FLAT_CATALOG_PRODUCT_PRICE (price),
    KEY IDX_FLAT_CATALOG_PRODUCT_SKU (sku)
) ENGINE=InnoDB;
```

### When Flat Tables Are Updated

Flat tables are **not** updated in real-time. They're regenerated by the `catalog_product_flat` indexer during reindex operations:

| Event | What Happens |
|-------|-------------|
| Product saved | Flat table entry updated **immediately** for that product |
| Category product reassignment | Flat tables updated |
| Full reindex | `bin/magento indexer:reindex catalog_product_flat` rebuilds all flat tables |
| Mass product attribute update | Updates via staging/indexer queue |

The flat table update on product save is done via an observer:

```php
<?php
// Magento\Catalog\Observer\CatalogProductSaveAfter
// Updates the flat table row immediately when a product is saved
// This happens BEFORE the full reindex runs
```

### use_flat_catalog_product Configuration

The `use_flat_catalog_product` configuration controls flat table usage at the catalog level:

```
Stores → Configuration → Catalog → Catalog → Storefront → Use Flat Catalog Product
```

| Setting | Effect |
|---------|--------|
| `Yes (Recommended)` | Product collection queries `catalog_product_flat_N` instead of EAV tables |
| `No` | Product collection queries EAV tables directly |

In production with millions of SKUs, flat tables are essential for catalog listing performance. The indexer runs on a schedule to keep flat tables in sync.

### Category Flat Tables

`catalog_category_flat_1`, `catalog_category_flat_2` etc. follow the same pattern but are simpler since categories have fewer attributes and no website scoping.

---

## 12. Custom EAV Entities

There are legitimate cases where a custom module needs EAV-style storage. Understanding when EAV is appropriate — and when a flat table is better — is critical.

### When NOT to Use EAV

EAV has significant overhead. Use a flat table (`db_schema.xml`) when:

| Scenario | Recommendation |
|----------|---------------|
| Fixed, known columns at development time | Flat table |
| Entity has < 10 attributes, all known upfront | Flat table |
| Millions of records, high write volume | Flat table — EAV write amplification is costly |
| Attributes won't be admin-managed | Flat table |
| Simple key-value lookups | Flat table |
| Admin needs to create new attributes at runtime | EAV |
| Attribute sets needed (different products have different attributes) | EAV |

### When to Use EAV

| Scenario | Recommendation |
|----------|---------------|
| Entity needs dynamic attribute creation | EAV |
| Admin users will add attributes in admin panel | EAV |
| Different instances have different attribute sets | EAV |
| Attributes have multiple input types (varchar, int, decimal) | EAV |

### Registering a Custom EAV Entity

To create a fully managed EAV entity (not using existing catalog/customer/category EAV), you must register it in `eav_entity_type`:

**Step 1: Create entity type registration patch**

```php
<?php
// Setup/Patch/Data/RegisterCustomEntityEav.php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Eav\Model\Config;
use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class RegisterCustomEntityEav implements DataPatchInterface
{
    private ModuleDataSetupInterface $moduleDataSetup;
    private Config $eavConfig;

    public function __construct(
        ModuleDataSetupInterface $moduleDataSetup,
        Config $eavConfig
    ) {
        $this->moduleDataSetup = $moduleDataSetup;
        $this->eavConfig = $eavConfig;
    }

    public function apply(): void
    {
        $this->eavConfig->configureEntityAttribute(
            'custom_entity',                    // entity type code
            \Vendor\Module\Model\ResourceModel\CustomEntity::class,  // entity model
            \Vendor\Module\Model\Entity\Attribute\Backend\CustomEntity::class, // attribute model
            [
                'entity_type_code' => 'custom_entity',
                'entity_model' => \Vendor\Module\Model\ResourceModel\CustomEntity::class,
                'attribute_model' => \Magento\Eav\Model\Entity\Attribute::class,
                'table' => 'vendor_custom_entity',
                'entity_attribute_collection' =>
                    \Vendor\Module\Model\ResourceModel\Attribute\Collection::class,
            ]
        );
    }

    public function getAliases(): array { return []; }
    public static function getDependencies(): array { return []; }
}
```

**Step 2: Create the entity tables**

```xml
<!-- etc/db_schema.xml -->
<table name="vendor_custom_entity" resource="default" engine="innodb"
       comment="Custom Entity Table">
    <column xsi:type="int" name="entity_id" padding="10" unsigned="true"
            identity="true" nullable="false" primary="true"/>
    <column xsi:type="varchar" name="code" length="64" nullable="false"/>
    <column xsi:type="datetime" name="created_at" nullable="false"
            default="CURRENT_TIMESTAMP"/>
    <column xsi:type="datetime" name="updated_at" nullable="false"
            default="CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP"/>
    <column xsi:type="varchar" name="name" length="255" nullable="true"/>
    <column xsi:type="int" name="status" padding="1" nullable="false" default="1"/>
</table>

<!-- Value tables for EAV attributes -->
<table name="vendor_custom_entity_varchar" ...>
<table name="vendor_custom_entity_int" ...>
<table name="vendor_custom_entity_decimal" ...>
<table name="vendor_custom_entity_text" ...>
<table name="vendor_custom_entity_datetime" ...>
```

**Step 3: Create the resource model**

```php
<?php
// Model/ResourceModel/CustomEntity.php
declare(strict_types=1);

namespace Vendor\Module\Model\ResourceModel;

use Magento\Eav\Model\Entity\AbstractEntity;
use Magento\Eav\Model\Entity\Context;

class CustomEntity extends AbstractEntity
{
    protected function _construct(): void
    {
        $this->_init('vendor_custom_entity', 'entity_id');
    }

    public function getDefaultAttributeSetId(): int
    {
        return 0; // No attribute set for custom entity
    }

    public function getEntityType(): \Magento\Eav\Model\Entity\Type
    {
        return $this->_eavEntityType;
    }
}
```

### Magento\Eav\Model\Config Registration

`Magento\Eav\Model\Config` is the central EAV configuration registry. When you call `configureEntityAttribute()`, it:

1. Inserts a row into `eav_entity_type` (if not exists)
2. Associates the entity model, attribute model, and collection class
3. Makes the entity type available for attribute operations via `EavSetup`

The result: your entity can use `EavSetup::addAttribute()` just like products and customers, and collections will generate the correct EAV joins.

---

## 13. EAV and Plugins

The plugin system in Magento 2 intercepts method calls on classes throughout the EAV stack. Understanding where to plugin is key to extending EAV behavior without modifying core classes.

### Key Plugin Points in the EAV Stack

**`AttributeRepository`** — central attribute CRUD:

```php
<?php
// Magento\Eav\Model\AttributeRepository
// Plugins on this class modify attribute retrieval and saving

class AttributeRepositoryPlugin
{
    public function beforeGet(
        \Magento\Eav\Model\AttributeRepository $subject,
        string $entityCode,
        string $attributeCode
    ): array {
        // Before getting an attribute — can modify arguments
        return [$entityCode, $attributeCode];
    }

    public function afterGet(
        \Magento\Eav\Model\AttributeRepository $subject,
        \Magento\Eav\Model\Entity\Attribute $result,
        string $entityCode,
        string $attributeCode
    ): \Magento\Eav\Model\Entity\Attribute {
        // After getting — can modify the returned attribute object
        return $result;
    }

    public function aroundSave(
        \Magento\Eav\Model\AttributeRepository $subject,
        callable $proceed,
        \Magento\Eav\Api\Data\AttributeInterface $attribute
    ): \Magento\Eav\Api\Data\AttributeInterface {
        // Around save — can add before/after logic around attribute save
        // E.g., dispatch custom event, validate attribute code
        return $proceed($attribute);
    }
}
```

**`EavAttributeManagement`** — manages attribute-group associations:

```php
<?php
// Magento\Eav\Model\EavAttributeManagement
// Plugins can intercept attribute assignment to groups

class EavAttributeManagementPlugin
{
    public function beforeAssign(
        \Magento\Eav\Model\EavAttributeManagement $subject,
        string $entityTypeCode,
        int $attributeSetId,
        int $attributeGroupId,
        int $attributeId
    ): array {
        // Before assigning attribute to a group
        // Can throw exception to prevent assignment
        return [$entityTypeCode, $attributeSetId, $attributeGroupId, $attributeId];
    }
}
```

**`Magento\Eav\Model\Entity\Collection\AbstractCollection`** — collection-level plugins:

```php
<?php
// Plugin on addAttributeToSelect — add default attributes automatically
public function aroundAddAttributeToSelect(
    \Magento\Eav\Model\Entity\Collection\AbstractCollection $subject,
    callable $proceed,
    string|array $attribute,
    string $joinType = null
): \Magento\Eav\Model\Entity\Collection\AbstractCollection {
    // Automatically add 'status' to every product collection
    if ($subject instanceof \Magento\Catalog\Model\ResourceModel\Product\Collection) {
        if (is_string($attribute) && $attribute !== 'status') {
            $attribute = [$attribute, 'status'];
        }
    }
    return $proceed($attribute, $joinType);
}
```

### Plugin Anti-Patterns with EAV

**1. Never plugin `AbstractCollection::load()`**

`load()` triggers the entire query pipeline. Around-plugging it without calling `$proceed()` breaks the collection entirely:

```php
// WRONG — breaks collection
public function aroundLoad(
    \Magento\Eav\Model\Entity\Collection\AbstractCollection $subject,
    callable $proceed
) {
    return $subject; // Missing $proceed() — collection never loads
}

// CORRECT — call $proceed()
public function aroundLoad(
    \Magento\Eav\Model\Entity\Collection\AbstractCollection $subject,
    callable $proceed,
    bool $printQuery = false,
    bool $logQuery = false
): \Magento\Eav\Model\Entity\Collection\AbstractCollection {
    $this->_logger->debug('Loading collection', ['class' => get_class($subject)]);
    return $proceed($printQuery, $logQuery);
}
```

**2. Plugin `addAttributeToFilter()` with care**

The collection uses a pending-filter queue. Around-plugging `addAttributeToFilter()` and not calling `$proceed()` loses all queued filters:

```php
// WRONG — discards pending filters
public function aroundAddAttributeToFilter(
    $subject, callable $proceed, $attribute, $condition = null
) {
    // Did not call $proceed() — all previously queued filters are lost!
    return $subject;
}
```

**3. `afterSave()` on attribute repository — infinite loop risk**

If your `afterSave()` plugin calls `$attributeRepository->save()`, and the core `save()` also dispatches a plugin, you can create infinite recursion. Use `beforeSave()` to validate instead.

---

## 14. Performance Pitfalls

EAV is powerful but has well-documented performance traps. Here are the most critical patterns to avoid.

### N+1 Query Problem in EAV Collections

The canonical EAV N+1 occurs when iterating a collection and calling `load()` on related objects inside the loop:

```php
<?php
// WRONG — N+1: runs a query for EACH product
$collection = $this->productCollectionFactory->create();
$collection->addAttributeToFilter('status', 1);

foreach ($collection as $product) {
    // For each product, load its categories (extra query)
    $categories = $product->getCategoryCollection(); // N+1!

    // Or load tier prices (extra query per product)
    $tierPrices = $product->getTierPrice(); // N+1!
}
```

**Fix:** Use `joinTable()` or `getProductsMerger` to pre-load related data in the original collection query.

### Using `->load()` Inside Loops

The most expensive anti-pattern in Magento:

```php
<?php
// WRONG — loads product individually within a loop
foreach ($productIds as $productId) {
    $product = $this->productFactory->create();
    $product->load($productId); // Query for each product!
    // Process product...
}

// CORRECT — load the collection once with all needed attributes
$collection = $this->productCollectionFactory->create()
    ->addAttributeToSelect(['name', 'price', 'status', 'sku'])
    ->addAttributeToFilter('entity_id', ['in' => $productIds]);

foreach ($collection as $product) {
    // All data already loaded — no per-item queries
}
```

### Missing Select Attributes

When you call `getData('attribute_code')` on an entity loaded via a collection that didn't include that attribute in `addAttributeToSelect()`, Magento attempts a lazy load — a separate query per attribute per entity:

```php
<?php
// WRONG — 'special_price' not in select, causes lazy load
$collection = $this->productCollectionFactory->create()
    ->addAttributeToSelect(['name', 'price']); // missing special_price

foreach ($collection as $product) {
    echo $product->getData('special_price'); // Individual query per product!
}
```

This is the "select attributes" problem. For every attribute access beyond what's selected, if the attribute isn't already loaded, Magento generates an `INSERT ... ON DUPLICATE KEY UPDATE` or a separate `SELECT` to fetch it.

**Fix:** Always declare every attribute you need in `addAttributeToSelect()`:

```php
<?php
// CORRECT — all needed attributes pre-selected
$collection = $this->productCollectionFactory->create()
    ->addAttributeToSelect(['name', 'price', 'special_price', 'special_from_date', 'special_to_date']);

foreach ($collection as $product) {
    echo $product->getData('special_price'); // No extra query
}
```

### The `clear()->addData()` Pattern

A common mistake when updating entities in a loop:

```php
<?php
// WRONG — clear() discards loaded data, addData() reloads from DB
$product = $this->productFactory->create();
foreach ($productIds as $productId) {
    $product->load($productId);
    $product->setPrice($newPrice);
    $product->save();       // Full save with all EAV indexers
    $product->clear();     // Discards loaded data
}
```

Each iteration in this pattern causes a `clear()` that dumps all loaded attribute values, followed by `addData()` re-fetching them on the next iteration. It's functionally equivalent to calling `load()` on every iteration.

**Fix:** Work with the collection directly for bulk operations:

```php
<?php
// CORRECT — bulk update without per-item overhead
$this->productResourceModel->getConnection()->update(
    'catalog_product_entity_decimal',
    ['value' => $newPrice],
    ['entity_id IN (?)' => $productIds, 'attribute_id = ?' => $priceAttributeId]
);
```

Or if you need model-level operations:

```php
<?php
// Better — use the collection's mass-update capability
$collection = $this->productCollectionFactory->create()
    ->addAttributeToFilter('entity_id', ['in' => $productIds]);

foreach ($collection as $product) {
    $product->setPrice($newPrice);
    // Only save once per iteration, no clear()
    $product->save();
}
```

### Missing Indexes on Filter Columns

EAV value tables are only as fast as their indexes. If you filter on an attribute that doesn't have an index entry in the value table, MySQL does a full table scan:

```sql
-- After adding a custom attribute, ensure the index exists:
CREATE INDEX IDX_CUSTOM_ATTRIBUTE_VALUE
ON catalog_product_entity_varchar (attribute_id, value);

-- Check existing indexes:
SHOW INDEX FROM catalog_product_entity_varchar;
```

When you add a custom attribute via `EavSetup::addAttribute()`, indexes are NOT automatically created. You must add them manually in a schema patch:

```php
<?php
// Setup/Patch/Schema/AddCustomAttributeIndex.php
public function apply(): void
{
    $this->moduleDataSetup->getConnection()->addIndex(
        'catalog_product_entity_varchar',
        $this->moduleDataSetup->getIdxName(
            'catalog_product_entity_varchar',
            ['attribute_id', 'value'],
            \Magento\Framework\DB\Adapter\AdapterInterface::INDEX_TYPE_INDEX
        ),
        ['attribute_id', 'value'],
        \Magento\Framework\DB\Adapter\AdapterInterface::INDEX_TYPE_INDEX
    );
}
```

### Collection Pagination Without setPageSize()

EAV collections hold all entity IDs in memory until `load()` is called. Without pagination:

```php
<?php
// WRONG — loads all matching products into memory
$collection = $this->productCollectionFactory->create()
    ->addAttributeToFilter('status', 1);
// No setPageSize() — on 50,000 products this exhausts memory

// CORRECT — always paginate
$collection = $this->productCollectionFactory->create()
    ->addAttributeToFilter('status', 1)
    ->setPageSize(100)     // Load 100 at a time
    ->setCurPage(1);       // First page
```

In production catalog listing pages, pagination is critical. The flat table indexer makes this less problematic for products, but custom EAV collections always need explicit pagination.

---

## See Also

- **`04-data-layer/03-di-plugins.md`** — Plugin system for intercepting EAV classes. `di.xml` plugin declarations on `AttributeRepository`, `EavAttributeManagement`, and collection classes are the primary extension points for EAV attribute operations.
- **`04-data-layer/15-event-observer.md`** — Event/Observer patterns. EAV save operations (`catalog_product_save_after`, `customer_save_after`) dispatch events that observers react to. Understanding the event/observer system is prerequisite to tracing data flow through EAV entities.
- **`04-data-layer/README.md`** — Data layer overview covering declarative schema, patches, model/resource/collection patterns, service contracts, and EAV architecture. This chapter expands on the EAV sections (Topics 6 and 7) with deeper technical treatment.
