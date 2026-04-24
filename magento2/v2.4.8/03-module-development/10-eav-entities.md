---
title: "27 - EAV & Custom EAV Entities"
description: "Deep dive into Magento 2.4.8 EAV architecture: entity types, attribute sets, EAV attribute management, and building custom EAV entities."
tags: magento2, eav, entities, attributes, attribute-sets, custom-entities, eav-attributes
rank: 10
pathways: [magento2-deep-dive]
---

# EAV & Custom EAV Entities

## 1. EAV Architecture Overview

### What is EAV?

Entity-Attribute-Value (EAV) is a data model that stores entity attributes as rows rather than columns. Instead of a fixed schema per entity, EAV stores:

- **Entity**: A business object (product, customer, order)
- **Attribute**: A named property (color, name, email)
- **Value**: The actual data for that attribute

Magento chose EAV to support:
- **Dynamic attributes**: Add/remove attributes without schema changes
- **Multi-store support**: Same entity can have different values per store
- **Attribute inheritance**: Customer address attributes inherit from customer
- **Flexible typing**: Each attribute knows its own data type

### Core EAV Tables

| Table | Purpose |
|-------|---------|
| `eav_entity_type` | Defines all entity types (product, customer, order, etc.) |
| `eav_attribute` | All attribute definitions with metadata |
| `eav_attribute_set` | Groups of attributes (Default, Default 2, etc.) |
| `eav_attribute_group` | Logical groupings within an attribute set |
| `eav_entity` | Base entity records |
| `eav_entity_datetime` | datetime attribute values |
| `eav_entity_decimal` | decimal/float attribute values |
| `eav_entity_int` | integer attribute values |
| `eav_entity_text` | text/blob attribute values |
| `eav_entity_varchar` | varchar/string attribute values |

### Entity-Specific EAV Tables

Product EAV tables (catalog_product_entity_*):
```sql
catalog_product_entity_varchar   --varchar(255) attributes: name, sku
catalog_product_entity_int       --integer attributes: status, visibility
catalog_product_entity_text      --text attributes: description, short_description
catalog_product_entity_decimal   --decimal attributes: price, special_price
catalog_product_entity_datetime  --datetime attributes: news_from_date
```

Customer EAV tables (customer_entity_*):
```sql
customer_entity_varchar   --email, first_name, last_name
customer_entity_int       --website_id, store_id, group_id
customer_entity_text      --备注/notes
```

### EAV Attribute Value Storage

Each attribute has a specific backend type determining which table stores its values:

```php
// Magento\Eav\Model\Entity\Attribute\Backend\AbstractBackend
// Determines which EAV value table an attribute uses

const TYPE_TEXT       = 'text';
const TYPE_VARCHAR    = 'varchar';
const TYPE_INT        = 'int';
const TYPE_DECIMAL    = 'decimal';
const TYPE_DATETIME   = 'datetime';
```

Value table structure (e.g., catalog_product_entity_varchar):
```sql
value_id    -- Primary key
attribute_id -- FK to eav_attribute
store_id     -- FK to store (0 = default)
entity_id    -- FK to catalog_product_entity
value        -- The actual string value
```

## 2. Entity Types

### Magento\Eav\Model\Entity\Type

The entity type model defines each EAV entity type:

```php
// Magento\Eav\Model\Entity\Type
declare(strict_types=1);

namespace Magento\Eav\Model\Entity;

class Type extends \Magento\Framework\Model\AbstractExtensibleModel
{
    /**
     * Entity type cache key prefix
     */
    public const ENTITY_TYPE_CACHE_TAG = 'EAV_ENTITY_TYPE';

    /**
     * @return string
     */
    public function getEntityTypeCode(): string;

    /**
     * @return int
     */
    public function getId(): ?int;

    /**
     * @return string
     */
    public function getEntityModel(): string;

    /**
     * @return string
     */
    public function getAttributeModel(): string;

    /**
     * @return string|null
     */
    public function getEntityTypeCode(): ?string;
}
```

### Predefined Entity Types

Register in `eav_entity_type` table via `EntitySetup` or declarative schema:

```php
// In Setup/InstallData.php or UpgradeData scripts
use Magento\Eav\Setup\EavSetup;
use Magento\Eav\Setup\EavSetupFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\ModuleContextInterface;

public function install(
    ModuleDataSetupInterface $setup,
    ModuleContextInterface $context
): void {
    $eavSetup = $this->eavSetupFactory->create(['setup' => $setup]);

    // Predefined entity types available:
    // - catalog_product
    // - customer
    // - customer_address
    // - order
    // - invoice
    // - shipment
    // - creditmemo
}
```

### Entity Type Table Structure

```sql
eav_entity_type:
  entity_type_id      -- INT, PK
  entity_type_code    -- VARCHAR(50), UNIQUE (e.g., 'catalog_product')
  entity_model        -- VARCHAR(255), fully qualified model class
  attribute_model     -- VARCHAR(255), default attribute model class
  entity_table        -- VARCHAR(255), main entity table (e.g., catalog_product_entity)
  value_table_prefix  -- VARCHAR(255), prefix for value tables
  entity_id_field     -- VARCHAR(255), field name for entity PK
  increment_model     -- VARCHAR(255), increment ID generator class
  increment_per_store -- TINYINT(1), increment per store flag
  additional_attribute_table    -- VARCHAR(255), additional attributes table
  entity_attribute_collection   -- VARCHAR(255), attribute collection class
  created_at          -- DATETIME
  updated_at          -- DATETIME
```

## 3. Attribute Sets

### Purpose of Attribute Sets

Attribute sets define which attributes are available for an entity of a given type. Products use attribute sets to group attributes like "Default" or "Clothing" or "Electronics".

### eav_attribute_set Table

```sql
eav_attribute_set:
  attribute_set_id    -- INT, PK
  entity_type_id      -- INT, FK to eav_entity_type
  attribute_set_name  -- VARCHAR(255)
  sort_order          -- INT
```

### Attribute Groups (eav_attribute_group)

```sql
eav_attribute_group:
  attribute_group_id  -- INT, PK
  attribute_set_id    -- INT, FK to eav_attribute_set
  attribute_group_name -- VARCHAR(255)
  sort_order          -- INT
  default_id          -- TINYINT(1), is default group
```

### Product Attribute Set Example

```php
// In Setup/InstallData.php
public function install(
    ModuleDataSetupInterface $setup,
    ModuleDataSetupInterface $output
): void {
    $eavSetup = $this->eavSetupFactory->create(['setup' => $setup]);

    // Create attribute set "Clothing" based on Default
    $entityTypeId = $eavSetup->getEntityTypeId(\Magento\Catalog\Model\Product::ENTITY);

    $attributeSet = $this->attributeSetFactory->create();
    $attributeSet->setEntityTypeId($entityTypeId);
    $attributeSet->setName('Clothing');
    $attributeSet->save();

    // Copy attributes from Default attribute set
    $eavSetup->cloneAttributesSet(
        $entityTypeId,
        'Default',
        'Clothing'
    );

    // Add attribute groups
    $eavSetup->addAttributeGroup(
        \Magento\Catalog\Model\Product::ENTITY,
        'Clothing',
        'Clothing Details',
        100
    );
}
```

### Default Product Attribute Groups

When you create a product attribute set based on "Default", these groups are copied:

| Group | Attributes |
|-------|------------|
| General | name, sku, status, url_key, visibility |
| Prices | price, special_price, cost, tier_price |
| Images | image, small_image, thumbnail |
| Search Engine Optimization | meta_title, meta_description, meta_keyword |
| Design | custom_design, custom_layout |
| Autosettings | set_id, type_id, created_at |

## 4. EAV Attributes

### eav_attribute Table Structure

```sql
eav_attribute:
  attribute_id                 -- INT, PK
  entity_type_id               -- INT, FK to eav_entity_type(entity_type_id) -- REQUIRED on every row
  attribute_code               -- VARCHAR(255), UNIQUE per entity_type
  attribute_model              -- VARCHAR(255), custom model class
  backend_model                -- VARCHAR(255), backend type handler
  backend_type                 -- ENUM('static','varchar','int','text','decimal','datetime')
  backend_table                -- VARCHAR(255), custom value table
  frontend_model              -- VARCHAR(255), frontend display class
  frontend_input              -- VARCHAR(50), input type (text, select, multiselect)
  frontend_label              -- VARCHAR(255), admin label
  frontend_class              -- VARCHAR(255), CSS classes
  source_model                 -- VARCHAR(255), options provider (for select/multiselect)
  is_required                 -- TINYINT(1)
  options_container           -- VARCHAR(255), container for frontend display
  help_text                    -- TEXT
  default_value               -- TEXT
  is_unique                    -- TINYINT(1)
  note                         -- VARCHAR(255)
  created_at                  -- DATETIME
  updated_at                  -- DATETIME
```

### Attribute Backend Types

| Type | Storage Table | Used For |
|------|--------------|----------|
| `varchar` | eav_entity_varchar | Short text: names, emails, SKUs |
| `text` | eav_entity_text | Long text: descriptions, comments |
| `int` | eav_entity_int | Integers: IDs, booleans, counts |
| `decimal` | eav_entity_decimal | Decimal: prices, dimensions |
| `datetime` | eav_entity_datetime | Dates: created_at, valid_until |
| `static` | Entity main table | Direct columns on entity table |

### Backend Models

Backend models handle attribute CRUD operations on EAV value tables:

```php
/// Base class for all backend models (Magento\Eav\Model\Entity\Attribute\Backend\AbstractBackend)

// Common backend models:

// For select/multiselect
Magento\Eav\Model\Entity\Attribute\Backend\ArrayBackend

// For datetime
Magento\Eav\Model\Entity\Attribute\Backend\Time

// For price decimal
Magento\Catalog\Model\Entity\Attribute\Backend\Price

// For images/files
Magento\Catalog\Model\Product\Attribute\Backend\Media

// For category references
Magento\Catalog\Model\Product\Attribute\Backend\Category
```

### Source Models

Source models provide option lists for `select` and `multiselect` input types:

```php
// Magento\Eav\Model\Entity\Attribute\Source\OptionInterface
interface OptionInterface
{
    /**
     * Get all options as array value => label
     *
     * @return array
     */
    public function getAllOptions();

    /**
     * Get option by value
     *
     * @param string|int $value
     * @return array|null
     */
    public function getOptionText($value);
}
```

Common source models:
```php
// Boolean yes/no
Magento\Eav\Model\Entity\Attribute\Source\Boolean

// Status (enabled/disabled)
Magento\Catalog\Model\Product\Attribute\Source\Status

// Visibility
Magento\Catalog\Model\Product\Attribute\Source\Visibility

// Custom options via Magento_Catalog
Magento\Catalog\Model\Product\Attribute\Source\Boolean
```

### Frontend Models

Frontend models control how attributes display in admin forms:

```php
// Magento\Eav\Model\Entity\Attribute\Frontend\AbstractFrontend
abstract class AbstractFrontend
{
    /**
     * @return string
     */
    public function getInputType(): string;

    /**
     * @param \Magento\Framework\DataObject $object
     * @return string
     */
    public function getValue(\Magento\Framework\DataObject $object): string;
}

// Common frontend models:

// Datepicker
Magento\Eav\Model\Entity\Attribute\Frontend\Datetime

// Image preview
Magento\Catalog\Model\Product\Attribute\Frontend\Image

// Currency display
Magento\Directory\Model\Attribute\Frontend\Currency
```

## 5. Adding Custom Product Attributes

### Using EavSetup::addAttribute()

```php
<?php
// app/code/Vendor/Module/Setup/UpgradeData.php
declare(strict_types=1);

namespace Vendor\Module\Setup;

use Magento\Eav\Setup\EavSetup;
use Magento\Eav\Setup\EavSetupFactory;
use Magento\Framework\Setup\UpgradeDataInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\ModuleContextInterface;

class UpgradeData implements UpgradeDataInterface
{
    private EavSetupFactory $eavSetupFactory;

    public function __construct(EavSetupFactory $eavSetupFactory)
    {
        $this->eavSetupFactory = $eavSetupFactory;
    }

    public function upgrade(
        ModuleDataSetupInterface $setup,
        ModuleContextInterface $context
    ): void {
        $eavSetup = $this->eavSetupFactory->create(['setup' => $setup]);

        $eavSetup->addAttribute(
            \Magento\Catalog\Model\Product::ENTITY,
            'vendor_color',  // attribute code
            [
                'type' => 'varchar',           // backend_type
                'label' => 'Vendor Color',
                'input' => 'select',           // frontend_input
                'required' => false,
                'sort_order' => 100,
                'position' => 100,
                'global' => \Magento\Catalog\Model\ResourceModel\Eav\Attribute::SCOPE_WEBSITE,
                'group' => 'General',          // assign to group
                'source' => \Vendor\Module\Model\Product\Attribute\Source\Color::class,
                'backend' => \Magento\Eav\Model\Entity\Attribute\Backend\ArrayBackend::class,
                'used_in_product_listing' => true,
                'is_used_in_grid' => true,
                'is_filterable_in_grid' => true,
                'visible_on_front' => true,
                'used_for_sort_by' => true,
            ]
        );
    }
}
```

### Custom Source Model for Select

```php
<?php
// app/code/Vendor/Module/Model/Product/Attribute/Source/Color.php
declare(strict_types=1);

namespace Vendor\Module\Model\Product\Attribute\Source;

use Magento\Eav\Model\Entity\Attribute\Source\AbstractSource;
use Magento\Framework\Data\OptionSourceInterface;

class Color extends AbstractSource implements OptionSourceInterface
{
    /**
     * @return array
     */
    public function getAllOptions(): array
    {
        if ($this->_options === null) {
            $this->_options = [
                ['label' => __('Red'), 'value' => 'red'],
                ['label' => __('Blue'), 'value' => 'blue'],
                ['label' => __('Green'), 'value' => 'green'],
                ['label' => __('Black'), 'value' => 'black'],
                ['label' => __('White'), 'value' => 'white'],
            ];
        }
        return $this->_options;
    }
}
```

### Adding Attribute to Specific Attribute Set/Group

```php
// After adding the attribute, assign to attribute set and group
$eavSetup->addAttributeToSet(
    \Magento\Catalog\Model\Product::ENTITY,
    'Clothing',           // attribute set name
    'Clothing Details',   // attribute group name
    'vendor_color'        // attribute code
);

// Or use the installer class directly
$attribute = $eavSetup->getAttribute(
    \Magento\Catalog\Model\Product::ENTITY,
    'vendor_color'
);
$eavSetup->addAttributeToSet(
    \Magento\Catalog\Model\Product::ENTITY,
    $attribute['attribute_set_id'],
    $attribute['attribute_group_id'],
    $attribute['attribute_id']
);
```

### Product Attribute Scope Constants

| Constant | Scope | Description |
|----------|-------|-------------|
| `SCOPE_GLOBAL` | Global | Same value across all websites/stores |
| `SCOPE_WEBSITE` | Website | Per website, inherited by stores |
| `SCOPE_STORE` | Store | Per store view, inherited by store groups |
| `SCOPE_NOT_SPECIFIED` | Inherited | Value cascades based on hierarchy |

## 6. Adding Custom Customer Attributes

### CustomerSetup Class

Customer attributes use a specialized setup class:

```php
<?php
// app/code/Vendor/Module/Setup/UpgradeData.php
declare(strict_types=1);

namespace Vendor\Module\Setup;

use Magento\Customer\Setup\CustomerSetup;
use Magento\Customer\Setup\CustomerSetupFactory;
use Magento\Framework\Setup\UpgradeDataInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\ModuleContextInterface;

class UpgradeData implements UpgradeDataInterface
{
    private CustomerSetupFactory $customerSetupFactory;

    public function __construct(CustomerSetupFactory $customerSetupFactory)
    {
        $this->customerSetupFactory = $customerSetupFactory;
    }

    public function upgrade(
        ModuleDataSetupInterface $setup,
        ModuleContextInterface $context
    ): void {
        $customerSetup = $this->customerSetupFactory->create(['setup' => $setup]);

        $customerSetup->addAttribute(
            \Magento\Customer\Model\Customer::ENTITY,
            'loyalty_tier',
            [
                'type' => 'int',
                'label' => 'Loyalty Tier',
                'input' => 'select',
                'required' => false,
                'visible' => true,
                'system' => false,
                'position' => 200,
                'source' => \Vendor\Module\Model\Customer\Attribute\Source\LoyaltyTier::class,
            ]
        );

        // Add to form (adminhtml_customer_account, adminhtml_checkout)
        $customerSetup->addAttributeToSet(
            \Magento\Customer\Model\Customer::ENTITY,
            'Default',
            'General',
            'loyalty_tier'
        );

        // Add to forms
        $attribute = $customerSetup->getEavConfig()
            ->getAttribute(\Magento\Customer\Model\Customer::ENTITY, 'loyalty_tier');

        $forms = [
            'adminhtml_customer',
            'customer_account_create',
            'customer_account_edit',
        ];

        foreach ($forms as $formCode) {
            $attribute->setData('used_in_forms', [$formCode]);
            $attribute->save();
        }
    }
}
```

### Customer Address Attributes

```php
// Adding customer address attributes follows the same pattern
// but uses the customer_address entity type

$customerSetup->addAttribute(
    \Magento\Customer\Model\Address::ENTITY,
    'address_verification_status',
    [
        'type' => 'int',
        'label' => 'Address Verification Status',
        'input' => 'select',
        'required' => false,
        'visible' => true,
        'system' => false,
        'position' => 150,
        'source' => \Vendor\Module\Model\Customer\Attribute\Source\AddressVerificationStatus::class,
    ]
);

// Assign to address forms
$attribute = $customerSetup->getEavConfig()
    ->getAttribute(\Magento\Customer\Model\Address::ENTITY, 'address_verification_status');

$addressForms = [
    'adminhtml_customer_address',
    'customer_register_address',
    'customer_address_edit',
];

foreach ($addressForms as $formCode) {
    $attribute->setData('used_in_forms', [$formCode]);
    $attribute->save();
}
```

### Customer Attribute Properties

Customer attributes have additional properties compared to product attributes:

| Property | Description |
|----------|-------------|
| `system` | Boolean - if true, cannot be deleted in admin |
| `visible` | Show in customer account pages |
| `required` | Mandatory at registration/checkout |
| `used_in_forms` | Array of form codes where attribute appears |
| `user_defined` | Can customer update this themselves |

### Customer Form Codes Reference

| Form Code | Purpose |
|-----------|---------|
| `adminhtml_customer` | Admin customer edit page |
| `adminhtml_customer_address` | Admin address edit |
| `customer_account_create` | Registration form |
| `customer_account_edit` | Account information |
| `customer_address_edit` | Address book edit |
| `customer_register_address` | New address during registration |
| `checkout_register` | Register during checkout |

## 7. Custom EAV Entity

### When to Create Custom EAV Entities

Create a custom EAV entity when you need:
- Dynamic attributes that admin can manage
- Per-store attribute values
- Integration with Magento's attribute management UI
- Complex entity with many optional attributes

### AbstractEntity Base Class

```php
<?php
// app/code/Vendor/Module/Model/ResourceModel/Entity.php
declare(strict_types=1);

namespace Vendor\Module\Model\ResourceModel;

use Magento\Eav\Model\Entity\AbstractEntity;

class Entity extends AbstractEntity
{
    protected function _construct(): void
    {
        $this->_init(\Vendor\Module\Model\ResourceModel\Entity::class);
    }

    /**
     * @return string
     */
    public function getEntityType(): string
    {
        return 'vendor_entity';
    }

    /**
     * @return int
     */
    public function getTypeId(): int
    {
        return 42; // from eav_entity_type
    }
}
```

### Entity Model Class

```php
<?php
// app/code/Vendor/Module/Model/Entity.php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Framework\Model\AbstractExtensibleModel;
use Magento\Eav\Model\Entity\Type as EntityType;

class Entity extends AbstractExtensibleModel
{
    protected function _construct(): void
    {
        $this->_init(\Vendor\Module\Model\ResourceModel\Entity::class);
    }

    /**
     * Retrieve entity type
     *
     * @return EntityType|null
     */
    public function getEntityType(): EntityType
    {
        return $this->_getResource()->getEntityType();
    }

    /**
     * @return int|null
     */
    public function getId(): ?int
    {
        return $this->getData('entity_id');
    }
}
```

### Collection Class

```php
<?php
// app/code/Vendor/Module/Model/ResourceModel/Entity/Collection.php
declare(strict_types=1);

namespace Vendor\Module\Model\ResourceModel\Entity;

use Magento\Eav\Model\Entity\Collection\AbstractCollection;

class Collection extends AbstractCollection
{
    protected function _construct(): void
    {
        $this->_init(
            \Vendor\Module\Model\Entity::class,
            \Vendor\Module\Model\ResourceModel\Entity::class
        );
    }
}
```

### Registering the Entity Type

```php
<?php
// app/code/Vendor/Module/Setup/InstallEntityType.php
declare(strict_types=1);

namespace Vendor\Module\Setup;

use Magento\Eav\Model\Entity\AttributeSet;
use Magento\Eav\Model\Entity\Type;
use Magento\Eav\Setup\EavSetup;
use Magento\Eav\Setup\EavSetupFactory;
use Magento\Framework\Setup\InstallDataInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\ModuleContextInterface;

class InstallData implements InstallDataInterface
{
    private EavSetupFactory $eavSetupFactory;

    public function __construct(EavSetupFactory $eavSetupFactory)
    {
        $this->eavSetupFactory = $eavSetupFactory;
    }

public function install(
    ModuleDataSetupInterface $setup,
    ModuleContextInterface $context
): void {
        $eavSetup = $this->eavSetupFactory->create(['setup' => $setup]);

        // Add entity type
        $eavSetup->addEntityType(
            \Vendor\Module\Model\Entity::class,
            [
                'entity_model' => \Vendor\Module\Model\ResourceModel\Entity::class,
                'attribute_model' => \Magento\Eav\Model\Entity\Attribute::class,
                'table' => 'vendor_entity',
                'entity_id' => 'entity_id',
                'increment_model' => \Magento\Eav\Model\Entity\Increment\NumericIncrement::class,
                'increment_per_store' => '0',
            ]
        );

        // Create default attribute set
        $entityTypeId = $eavSetup->getEntityTypeId(\Vendor\Module\Model\Entity::class);

        $eavSetup->addAttributeSet(
            $entityTypeId,
            'Default'
        );
    }
}
```

### Value Table Naming Convention

Custom EAV entities use the same value table pattern:

| Table | Attributes |
|-------|------------|
| `{entity}_entity_varchar` | VARCHAR(255) values |
| `{entity}_entity_int` | Integer values |
| `{entity}_entity_text` | Long text values |
| `{entity}_entity_decimal` | Decimal values |
| `{entity}_entity_datetime` | Datetime values |

```sql
-- Value tables for custom entity
vendor_entity_varchar
vendor_entity_int
vendor_entity_text
vendor_entity_decimal
vendor_entity_datetime
```

### Di.xml Configuration

```xml
<?xml version="1.0"?>
<!-- app/code/Vendor/Module/etc/di.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Entity Model -->
    <preference for="Vendor\Module\Api\Data\EntityInterface"
                type="Vendor\Module\Model\Entity"/>

    <!-- Resource Model -->
    <preference for="Vendor\Module\Model\ResourceModel\Entity"
                type="Vendor\Module\Model\ResourceModel\Entity"/>

    <!-- Setup Model -->
    <preference for="Vendor\Module\Api\EntityRepositoryInterface"
                type="Vendor\Module\Model\EntityRepository"/>
</config>
```

## 8. Extension Attributes

### When to Use Extension Attributes vs EAV

| Scenario | Use Extension Attributes | Use EAV |
|----------|-------------------------|---------|
| Simple scalar value (string, int) | ✅ | |
| Object/array data (JSON) | ✅ | |
| One-off custom field on entity | ✅ | |
| Admin-manageable attributes | | ✅ |
| Per-store attribute values | | ✅ |
| Searchable/filterable in product listing | | ✅ |
| Complex entity with many optional attributes | | ✅ |
| API extension data | ✅ | |

### extension_attributes.xml Declaration

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- app/code/Vendor/Module/etc/extension_attributes.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Api/etc/extension_attributes.xsd">

    <!-- Extend Product with custom field -->
    <extension_attributes for="Magento\Catalog\Api\Data\ProductInterface">
        <attribute code="vendor_reference" type="string"/>
        <attribute code="approval_status" type="Vendor\Module\Api\Data\ApprovalStatusInterface"/>
        <attribute code="custom_options" type="Vendor\Module\Api\Data\CustomOptionInterface[]"/>
    </extension_attributes>

    <!-- Extend Customer with custom data -->
    <extension_attributes for="Magento\Customer\Api\Data\CustomerInterface">
        <attribute code="loyalty_points" type="int"/>
        <attribute code="account_manager_id" type="int"/>
    </extension_attributes>

    <!-- Extend Order Item with order-specific data -->
    <extension_attributes for="Magento\Sales\Api\Data\OrderItemInterface">
        <attribute code="applied_discounts" type="Vendor\Module\Api\Data\DiscountDataInterface[]"/>
        <attribute code="vendor_note" type="string"/>
    </extension_attributes>
</config>
```

### Extension Attribute Data Interface

```php
<?php
// app/code/Vendor/Module/Api/Data/ApprovalStatusInterface.php
declare(strict_types=1);

namespace Vendor\Module\Api\Data;

use Magento\Framework\Api\ExtensibleDataInterface;

interface ApprovalStatusInterface extends ExtensibleDataInterface
{
    /**
     * @return string
     */
    public function getStatus(): string;

    /**
     * @param string $status
     * @return $this
     */
    public function setStatus(string $status): self;

    /**
     * @return string|null
     */
    public function getApprovedAt(): ?string;

    /**
     * @param string|null $approvedAt
     * @return $this
     */
    public function setApprovedAt(?string $approvedAt): self;
}
```

### Implementing Extension Attributes

```php
<?php
// app/code/Vendor/Module/Plugin/ProductExtensionAttributesPlugin.php
declare(strict_types=1);

namespace Vendor\Module\Plugin;

use Magento\Catalog\Api\Data\ProductInterface;
use Magento\Catalog\Api\Data\ProductExtensionInterface;

class ProductExtensionAttributesPlugin
{
    /**
     * After load - populate extension attributes from database/storage
     *
     * @param ProductInterface $product
     * @param ProductExtensionInterface|null $extension
     * @return ProductExtensionInterface
     */
    public function afterGetExtensionAttributes(
        ProductInterface $product,
        ?ProductExtensionInterface $extension
    ): ?ProductExtensionInterface {
        if ($extension === null) {
            $extension = $this->extensionFactory->create();
        }

        // Load custom data (from custom table, API, etc.)
        $customData = $this->loadVendorReference($product->getId());

        $extension->setVendorReference($customData['reference']);
        $extension->setApprovalStatus($this->createApprovalStatus($customData));

        return $extension;
    }

    /**
     * Before save - store extension attribute data
     *
     * @param ProductInterface $product
     * @return void
     */
    public function beforeSave(ProductInterface $product): void
    {
        $extension = $product->getExtensionAttributes();
        if ($extension === null) {
            return;
        }

        $vendorReference = $extension->getVendorReference();
        if ($vendorReference !== null) {
            $this->saveVendorReference($product->getId(), $vendorReference);
        }
    }
}
```

### extension_attributes.xsd Schema Reference

```xsd
<!-- Location: vendor/magento/framework/Api/etc/extension_attributes.xsd -->
<xs:schema>
    <xs:element name="extension_attributes">
        <xs:complexType>
            <xs:sequence>
                <xs:element name="attribute" maxOccurs="unbounded">
                    <xs:complexType>
                        <xs:attribute name="code" type="xs:string" use="required"/>
                        <xs:attribute name="type" type="xs:string" use="required"/>
                    </xs:complexType>
                </xs:element>
            </xs:sequence>
            <xs:attribute name="for" type="xs:string" use="required"/>
        </xs:complexType>
    </xs:element>
</xs:schema>
```

## 9. EAV Performance

### The N+1 Query Problem

EAV's flexibility comes with performance costs. The classic problem:

```php
// BAD: N+1 queries when loading products with EAV attributes
$products = $productCollection->getItems(); // 1 query

foreach ($products as $product) {
    // Each attribute access triggers a separate query
    echo $product->getName();           // +1 query
    echo $product->getPrice();          // +1 query
    echo $product->getDescription();    // +1 query
}
// Total: 1 + 3*N queries!
```

### Solutions for N+1

**1. Use getData() with specific attributes:**

```php
// BETTER: Pre-load specific attributes
$productCollection->addAttributeToSelect(['name', 'price', 'description']);

// Still not ideal - each attribute requires separate join
```

**2. Join all EAV tables:**

```php
// GOOD: Use eager loading with joins
$productCollection = $this->productCollectionFactory->create();
$productCollection->addAttributeToSelect('*');

// Or filter at database level
$productCollection->addAttributeToFilter('status', 1);
```

**3. Use flat tables (Catalog Product Flat):**

```php
<?xml version="1.0"?>
<!-- Storefront performance: enable flat tables -->
<!-- In Magento Admin: Stores > Settings > Configuration > Catalog > Catalog > Storefront -->
<!-- Enable "Use Flat Catalog Product" = Yes -->
```

Product flat tables (`catalog_product_flat_1`) contain denormalized attributes as columns:

```sql
-- catalog_product_flat_1 has all attributes as columns
-- Single query: SELECT * FROM catalog_product_flat_1 WHERE store_id = 1

-- Trade-off: Must rebuild flat tables when attributes change
-- Triggered by: bin/magento indexer:reindex catalog_product_flat
```

### Lazy Loading in EAV

Magento implements lazy loading at the attribute level:

```php
<?php
// vendor/magento/module-eav/Model/Entity/AbstractEntity.php

/**
 * @inheritdoc
 */
public function getAttribute($attributeCode)
{
    $attributes = $this->_getAttributes($this->getType());
    if (!isset($attributes[$attributeCode])) {
        return null;
    }

    $attribute = $attributes[$attributeCode];
    // Attribute object is cached, but value is loaded on demand
    return $attribute;
}

/**
 * Load attribute value only when accessed
 */
public function getData(string $key = null, string $index = null)
{
    // Value is loaded from EAV tables only when getData() is called
    // Not loaded during entity load
}
```

### Performance Best Practices

**1. Minimize EAV attribute count:**
```php
// Bad: Many single-use EAV attributes
$eavSetup->addAttribute('catalog_product', 'custom_attr_1', [...]);
$eavSetup->addAttribute('catalog_product', 'custom_attr_2', [...]);

// Good: Group related data into JSON extension attributes
// Or use custom tables for complex data
```

**2. Index EAV attributes properly:**
```sql
-- Essential indexes on EAV value tables
CREATE INDEX idx_entity_type ON eav_attribute(entity_type_id);
CREATE INDEX idx_attribute_code ON eav_attribute(attribute_code);
CREATE INDEX idx_entity_attribute ON catalog_product_entity_varchar(attribute_id, entity_id, store_id);
```

**3. Use attribute sets strategically:**
```php
// Only load attributes needed for current context
$productCollection
    ->addAttributeToSelect(['name', 'price']) // Minimal select
    ->addAttributeToFilter('status', 1);
```

**4. Cache entity loading:**
```xml
<!-- etc/cache.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Cache/etc/cache.xsd">
    <type name="eav" translated="label">
        <label>EAV Attributes</label>
        <description>Caches EAV attribute metadata</description>
    </type>
</config>
```

### Full Reindex Strategy

EAV attributes must be reindexed when:
- Attributes added/modified
- Products saved in bulk
- Store configuration changes

```bash
# Reindex all EAV-based indexers
bin/magento indexer:reindex

# Target specific indexers
bin/magento indexer:reindex catalog_product_attribute
bin/magento indexer:reindex customer_grid
```

### Comparing Storage Models

| Approach | Pros | Cons |
|----------|------|------|
| **EAV** | Flexible, per-store values, admin manageable | N+1 queries, complex joins |
| **Flat Table** | Single query, fast reads | Denormalized, rebuild required |
| **Extension Attributes** | Simple, API-ready | Limited querying |
| **Custom Tables** | Full control, optimized | No admin UI integration |

---

## Summary

Magento's EAV system provides unparalleled flexibility for managing entity attributes dynamically. The key takeaways:

1. **EAV tables** store entity data in a normalized format (eav_entity_* tables), enabling per-store values and dynamic attribute management

2. **Entity Types** define the schema for each business object, from products to customers to custom entities

3. **Attribute Sets** group attributes for different product types, with attribute groups providing UI organization

4. **EAV Attributes** have rich metadata: backend/source/frontend models control their behavior, type determines storage

5. **Custom Product Attributes** use EavSetup with proper scope and source model configuration

6. **Customer Attributes** follow the same pattern with CustomerSetup and form-specific visibility

7. **Custom EAV Entities** extend AbstractEntity for truly custom business objects with full EAV capabilities

8. **Extension Attributes** are the modern choice for simple scalar values and should replace one-off EAV attributes

9. **Performance** requires careful consideration: flat tables, eager loading, and caching strategies are essential

10. The decision between EAV and extension attributes should be driven by whether you need admin-managed, per-store, or searchable attributes (EAV) versus simple API extension data (extension attributes).

*Article part of the **2025-Q4-magento2-deep-dive** course | Rank 27 | Last updated: 2025*