# Topic 19: CMS, Widgets, Page Builder & Extension Attributes

**Goal:** Integrate custom content into Magento's CMS layer, create reusable widget instances, and expose custom entity fields via extension attributes for API access.

---

## Topics Covered

1. CMS Blocks and Pages — Architecture and Storage
2. The Widget System — widget.xml and Widget Instances
3. Page Builder — Drag-and-Drop Content Creation
4. Extension Attributes — Adding Fields to API Entities
5. Custom Attributes vs Extension Attributes vs EAV
6. CMS in Headless Architectures
7. CMS Block Permissions and Preview
8. Widget Instance Assignment and Rendering
9. Extension Attributes via GraphQL
10. Content Staging Integration (EE Perspective)

---

## Reference Exercises

1. **Exercise 1:** Create a CMS block programmatically using a Data Patch, then render it in a custom PHTML template.
2. **Exercise 2:** Build a custom widget with configurable parameters (text input and select), declare it in `etc/widget.xml`, and assign an instance to a CMS page.
3. **Exercise 3:** Declare an extension attribute for the Product entity in `extension_attributes.xml`, add the database field via `db_schema.xml`, and expose it via GraphQL resolver.
4. **Exercise 4:** Create a custom product attribute via setup script, then compare it against an extension attribute — explain when each is appropriate.

---

## Completion Criteria

- [ ] CMS block created and rendered in a template
- [ ] Custom widget declared in widget.xml with parameters
- [ ] Widget instance assigned to a CMS page
- [ ] Extension attribute declared in extension_attributes.xml for a Product
- [ ] Extension attribute field accessible via GraphQL
- [ ] Custom attribute on Product/Customer via setup script
- [ ] Can explain when to use extension attribute vs EAV custom attribute
- [ ] Can describe the difference between CE and EE Page Builder features

---

## Topics

---

### Topic 1: CMS Blocks and Pages — Architecture and Storage

CMS Blocks and Pages are the fundamental content units in Magento. Blocks are reusable content pieces that can be embedded anywhere in the layout. Pages are full content documents with their own URLs.

**Database Schema:**

CMS content lives in three primary tables:

```sql
-- cms_block: stores individual content blocks
cms_block
├── block_id    (PK, identity)
├── title       (admin-facing title)
├── identifier   (unique string used in code/templates, e.g. "footer-links")
├── content      (HTML content, often contains widgets)
├── creation_time
├── update_time
└── is_active    (1=enabled, 0=disabled)

-- cms_page: stores full CMS pages
cms_page
├── page_id
├── title
├── identifier  (URL key, e.g. "about-us")
├── content      (HTML content with widgets)
├── content_heading
├── url_key      (redirects to identifier in newer versions)
├── meta_title
├── meta_keywords
├── meta_description
├── layout_update_xml   (custom layout instructions, rarely used)
├── custom_theme
├── custom_root_template
├── layout_handle        (associated layout handle)
├── is_active
├── sort_order
└── publish_date

-- cms_page_store: many-to-many between pages and stores
cms_page_store
├── page_id  (FK → cms_page.page_id)
└── store_id (FK → store/store_id)
```

> **Key architectural point:** CMS pages use a many-to-many relationship with stores via `cms_page_store`. A single page can be assigned to multiple stores. CMS blocks, by contrast, are global by default but can be filtered by `store_id` in the block repository queries.

**Creating CMS Blocks via Data Patch:**

The cleanest way to create CMS blocks programmatically is through a `DataPatchInterface` implementation. This ensures idempotent data creation — the patch checks whether the block exists before inserting.

```php
<?php
// Setup/Patch/Data/CreateFooterLinksBlock.php
declare(strict_types=1);

namespace Training\Cms\Setup\Patch\Data;

use Magento\Cms\Model\BlockFactory;
use Magento\Cms\Model\BlockRepository;
use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class CreateFooterLinksBlock implements DataPatchInterface
{
    private ModuleDataSetupInterface $moduleDataSetup;
    private BlockFactory $blockFactory;
    private BlockRepository $blockRepository;

    public function __construct(
        ModuleDataSetupInterface $moduleDataSetup,
        BlockFactory $blockFactory,
        BlockRepository $blockRepository
    ) {
        $this->moduleDataSetup = $moduleDataSetup;
        $this->blockFactory = $blockFactory;
        $this->blockRepository = $blockRepository;
    }

    public function apply(): void
    {
        // Check if block already exists
        try {
            $this->blockRepository->getById('footer-links');
            // Block exists — patch is already applied, skip
            return;
        } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
            // Block doesn't exist — proceed with creation
        }

        $block = $this->blockFactory->create([
            'data' => [
                'title' => 'Footer Links Block',
                'identifier' => 'footer-links',
                'content' => '<div class="footer-links"><ul>'
                    . '<li><a href="{{store url="privacy-policy"}}">Privacy Policy</a></li>'
                    . '<li><a href="{{store url="terms"}}">Terms of Service</a></li>'
                    . '<li><a href="{{store url="contact"}}">Contact Us</a></li>'
                    . '</ul></div>',
                'is_active' => 1,
            ]
        ]);

        $this->blockRepository->save($block);
    }

    public function getAliases(): array { return []; }
    public static function getDependencies(): array { return []; }
}
```

> **Why use `BlockFactory` + `BlockRepository` instead of direct model save?** The repository applies all registered plugins (e.g., URL rewrites, cache cleaning) and enforces the service contract. Direct model saves bypass these hooks and can cause cache/integration issues.

**Creating CMS Pages via Data Patch:**

```php
<?php
// Setup/Patch/Data/CreateAboutUsPage.php
declare(strict_types=1);

namespace Training\Cms\Setup\Patch\Data;

use Magento\Cms\Model\PageFactory;
use Magento\Cms\Model\PageRepository;
use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class CreateAboutUsPage implements DataPatchInterface
{
    private ModuleDataSetupInterface $moduleDataSetup;
    private PageFactory $pageFactory;
    private PageRepository $pageRepository;

    public function __construct(
        ModuleDataSetupInterface $moduleDataSetup,
        PageFactory $pageFactory,
        PageRepository $pageRepository
    ) {
        $this->moduleDataSetup = $moduleDataSetup;
        $this->pageFactory = $pageFactory;
        $this->blockRepository = $pageRepository;
    }

    public function apply(): void
    {
        $page = $this->pageFactory->create([
            'data' => [
                'title' => 'About Us',
                'page_layout' => '1column',
                'identifier' => 'about-us',
                'content_heading' => 'About Our Company',
                'content' => '<h2>Welcome</h2><p>We are a customer-first company...</p>',
                'meta_keywords' => 'about, company, story',
                'meta_description' => 'Learn about our company story and values',
                'is_active' => 1,
                'sort_order' => 0,
                'layout_update_xml' => '',
            ]
        ]);

        $this->pageRepository->save($page);

        // Assign to specific stores (store IDs 0 = all stores, or array of IDs)
        $page->setStores([0]);
    }

    public function getAliases(): array { return []; }
    public static function getDependencies(): array { return []; }
}
```

**Rendering CMS Blocks in Templates:**

To render a CMS block inside any PHTML template, use the layout's `createBlock` method:

```php
<?php
// In any .phtml template file
$blockHtml = $block->getLayout()
    ->createBlock(\Magento\Cms\Block\Block::class)
    ->setBlockId('footer-links')
    ->toHtml();
?>
```

This approach respects the block's `is_active` flag — if the block is disabled, it renders nothing. The block is looked up by its `identifier` field.

**Block Cache Behavior:**

CMS blocks have nuanced caching behavior:

| Scenario | Cache Behavior |
|----------|----------------|
| Block `is_active=1`, no cache lifetime | Cached in full-page cache (FPC) |
| Block rendered via `createBlock()->toHtml()` | Not cached by default — rendered fresh each time |
| Block inside `cacheable="false"` container | Not cached |
| Block with explicit `cache_lifetime` set | Cached for that duration |

> **Performance implication:** If you render a CMS block dynamically in a template (via `createBlock()->toHtml()`), it is NOT cached in the full-page cache by default. Each page request will re-render the block. For frequently used blocks, consider embedding them via layout XML instead, which respects the FPC.

**CMS Page Versions (EE Feature):**

Adobe Commerce (EE) adds a versioning layer to CMS pages via the `magento_staging` module. Each page becomes a staging entity with versioned updates. CE does not have this capability — a CMS page has one set of content at any given time.

In EE, the content staging system creates `staging_update` records that link to CMS entities, allowing scheduled content changes. We cover this in detail in Topic 10.

**Service Contract Interfaces:**

Magento exposes CMS entities through well-defined service contract interfaces:

```php
<?php
// Magento\Cms\Api\BlockRepositoryInterface
interface BlockRepositoryInterface
{
    public function save(\Magento\Cms\Api\Data\BlockInterface $block): \Magento\Cms\Api\Data\BlockInterface;
    public function getById(int $blockId): \Magento\Cms\Api\Data\BlockInterface;
    public function delete(\Magento\Cms\Api\Data\BlockInterface $block): void;
    public function deleteById(int $blockId): void;
    public function getList(\Magento\Framework\Api\SearchCriteriaInterface $searchCriteria): \Magento\Cms\Api\Data\BlockSearchResultsInterface;
}

// Magento\Cms\Api\PageRepositoryInterface — similar pattern
interface PageRepositoryInterface
{
    // save, getById, delete, deleteById, getList
}
```

Always inject and use these interfaces rather than the concrete models — this keeps your code decoupled and testable.

---

### Topic 2: The Widget System — widget.xml and Widget Instances

Widgets are reusable UI components that merchants can embed inside CMS blocks, CMS pages, or any WYSIWYG content area. Unlike hardcoded blocks, widgets are configurable at runtime by content editors through the admin.

**What Makes a Widget:**

A widget consists of:
1. A Block class that implements the widget logic
2. A `.phtml` template for frontend rendering
3. A declaration in `etc/widget.xml` that defines available parameters
4. Optionally: an admin form for the widget instance configuration UI

**Declaring a Widget in widget.xml:**

```xml
<?xml version="1.0"?>
<!-- app/code/Training/HelloWorld/etc/widget.xml -->
<widgets xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Widget:etc/widget.xsd">

    <widget id="training_hello_world"
            class="Training\HelloWorld\Block\Widget\HelloWorld"
            translate="label"
            template="Training_HelloWorld::widget/hello.phtml">

        <!-- Label shown in the widget insertion dropdown -->
        <label translate="true">Hello World Widget</label>
        <description translate="true">Displays a configurable greeting message</description>

        <!-- Parameters that content editors can configure -->
        <parameters>

            <!-- Text parameter -->
            <parameter name="greeting" xsi:type="text" required="true" sort_order="10">
                <label translate="true">Greeting Text</label>
                <description translate="true">The greeting message to display</description>
            </parameter>

            <!-- Select/dropdown parameter -->
            <parameter name="style" xsi:type="select" required="false" sort_order="20">
                <label translate="true">Style</label>
                <options>
                    <option name="friendly" value="friendly" label="Friendly"/>
                    <option name="formal" value="formal" label="Formal"/>
                    <option name="casual" value="casual" label="Casual"/>
                </options>
            </parameter>

            <!-- Textarea parameter for longer content -->
            <parameter name="custom_message" xsi:type="textarea" required="false" sort_order="30">
                <label translate="true">Custom Message</label>
                <description translate="true">Optional longer message</description>
            </parameter>

            <!-- Block reference parameter — lets editor pick a CMS block -->
            <parameter name="block_id" xsi:type="block" required="false" sort_order="40">
                <label translate="true">CMS Block</label>
                <description translate="true">Select a CMS block to display</description>
            </parameter>

        </parameters>
    </widget>
</widgets>
```

**Widget Parameter Types:**

| Type | Use Case | Admin UI |
|------|----------|----------|
| `text` | Short strings, titles | Text input |
| `select` | Dropdown options | Select menu |
| `textarea` | Long text, HTML | Text area |
| `block` | Reference to a CMS block | Block picker |
| `checkbox` | Boolean toggle | Checkbox |
| `page_link` | Link to a specific page | Page selector |

**Widget Block Class:**

```php
<?php
// Block/Widget/HelloWorld.php
declare(strict_types=1);

namespace Training\HelloWorld\Block\Widget;

use Magento\Framework\View\Element\Template;
use Magento\Widget\Block\BlockInterface;

class HelloWorld extends Template implements BlockInterface
{
    protected $_template = 'Training_HelloWorld::widget/hello.phtml';

    /**
     * Return the greeting text with configured style applied.
     */
    public function getGreeting(): string
    {
        $greeting = (string) $this->getData('greeting');
        $style = (string) $this->getData('style');

        if (empty($greeting)) {
            return '';
        }

        $prefix = match ($style) {
            'formal' => 'Dear Customer,',
            'casual' => 'Hey there,',
            default => 'Hello,',
        };

        return $prefix . ' ' . $greeting;
    }

    public function getStyle(): string
    {
        return (string) $this->getData('style') ?: 'friendly';
    }
}
```

**Widget Template:**

```php
<?php
// view/frontend/templates/widget/hello.phtml
/** @var \Training\HelloWorld\Block\Widget\HelloWorld $block */

$styleClass = match ($block->getStyle()) {
    'formal' => 'greeting-formal',
    'casual' => 'greeting-casual',
    default => 'greeting-friendly',
};
?>
<div class="widget-hello-world <?= $block->escapeHtmlAttr($styleClass) ?>">
    <p class="greeting-text"><?= $block->escapeHtml($block->getGreeting()) ?></p>
    <?php if ($customMessage = $block->getData('custom_message')): ?>
        <div class="custom-message"><?= $block->escapeHtml($customMessage) ?></div>
    <?php endif; ?>
</div>
```

> **BlockInterface is required:** Any class used as a widget block must implement `Magento\Widget\Block\BlockInterface`. This interface doesn't require any specific methods — it's a marker interface that tells Magento's widget system this block is valid for widget instantiation.

**When to Use Widgets vs Static Blocks vs Template Inclusion:**

| Approach | Use When |
|----------|----------|
| **Widget** | Content editors need to embed it dynamically in CMS content. Requires admin-configurable parameters. Reusable across many pages. |
| **CMS Block** | A fixed piece of content reused in multiple templates/layouts. Content editors manage the content. No dynamic parameters at placement time. |
| **Template inclusion** (`getChildHtml`) | The content is developer-managed, not editor-managed. Used in layout XML for structural elements (e.g., header, footer). |

**Widget Instance Storage:**

When a merchant creates a widget instance via the admin (Content → Widgets), the configuration is stored in the `widget_instance` table:

```sql
widget_instance
├── id
├── instance_type        (e.g., "Training\HelloWorld\Block\Widget\HelloWorld")
├── package_theme        (e.g., "Magento/luma")
├── widget_parameters    (JSON — stores parameter values)
├── sort_order
├── display_on          (specific_page, all_pages)
└── entity_ids          (page IDs or block IDs where assigned)
```

The `widget_parameters` JSON field stores the configuration set by the content editor in the admin:

```json
{
    "greeting": "Welcome to our store!",
    "style": "friendly",
    "custom_message": "Special offer for new customers"
}
```

**Common Mistakes — Widget Development:**

> **Mistake 1: Forgetting `translate="label"` on widget and parameter elements.** Without `translate="true""`, the `<label>` child of `<parameter>` will not be picked up by Magento's internationalization system. Always add `translate="true"` to any element containing user-facing text.

> **Mistake 2: Using a non-existent template path.** The template path in `template="Training_HelloWorld::widget/hello.phtml"` must resolve to `view/frontend/templates/widget/hello.phtml`. If the template file doesn't exist or the path is mis-typed, the widget renders nothing.

> **Mistake 3: Widget block caching.** Widget blocks are cached by default in the full-page cache. If your widget shows dynamic content (e.g., time-sensitive), you must either set `cacheable="false"` on the widget block OR implement proper cache lifetime handling via `_cacheTags` and `_cacheLifetime`.

---

### Topic 3: Page Builder — Drag-and-Drop Content Creation

Page Builder replaced the legacy WYSIWYG editor (TinyMCE) starting in Magento 2.3 as the default content editing experience. It provides a drag-and-drop interface for building rich content layouts without requiring HTML knowledge.

**CE vs EE Feature Set:**

| Content Type | CE (Open Source) | EE (Adobe Commerce) |
|-------------|-----------------|---------------------|
| Row | ✅ | ✅ |
| Column | ✅ | ✅ |
| Heading | ✅ | ✅ |
| Text | ✅ | ✅ |
| Image | ✅ | ✅ |
| Button | ✅ | ✅ |
| Divider | ✅ | ✅ |
| Tabs | ❌ | ✅ |
| Sliders | ❌ | ✅ |
| Banners | ❌ | ✅ |
| Video | ❌ | ✅ |
| Google Maps | ❌ | ✅ |
| Products Carousel | ❌ | ✅ |
| HTML Code | ❌ | ✅ |
| Schedule (Staging) | ❌ | ✅ |

CE has had basic Page Builder since Magento 2.4.3. Before that, CE used the old TinyMCE editor.

**Page Builder Architecture:**

Page Builder content types are defined declaratively via XML configuration. Each content type consists of:

1. A master template (`.html` file defining the wrapper structure)
2. An element template (`.html` file defining the actual rendered output)
3. An icon (for the Page Builder stage panel)
4. A `<content_type>.xml` file defining the type metadata

The content types are defined in `etc/content_types.xml` files inside a module:

```xml
<?xml version="1.0"?>
<!-- In a custom module's etc/content_types.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_PageBuilder:etc/content_types.xsd">

    <content_type name="training_banner"
                  master_template="Training_Banner::master.html"
                  preview_template="Training_Banner::preview.html"
                  form="Training_Banner::form.html"
                  icon="Training_Banner::images/icon-banner.svg">

        <children>
            <element name="wrapper">
                <style name="text_align" source="textAlign"/>
                <attribute name="data-banner-id" storage="preview"/>
            </element>
        </children>

        <template method="appearance">
            <template name="default" value="default"/>
            <template name="hover" value="hover"/>
        </template>

    </content_type>
</config>
```

> **CE Limitation:** Developing custom content types requires either EE or a paid third-party extension. CE does not ship with the content type development toolkit. You can consume (place on a page) existing content types, but you cannot create new ones in CE without significant custom development that isn't officially supported.

**Migrating from Old WYSIWYG Content:**

When upgrading or migrating from TinyMCE-based content to Page Builder, widget tags get converted to Page Builder XML format:

```html
<!-- Old TinyMCE / WYSIWYG content with a widget -->
<div class="some-class">
    {{widget type="Training\HelloWorld\Block\Widget\HelloWorld" greeting="Hello"}}
    <p>Some text content</p>
</div>

<!-- After Page Builder migration, stored as Page Builder XML -->
<row>
    <column>
        <heading>Some Heading</heading>
    </column>
</row>
```

Magento provides a migration tool for converting legacy widget tags to Page Builder format, but complex widget structures may require manual conversion.

**CSS Classes System in Page Builder:**

Page Builder elements generate specific CSS classes for styling:

```html
<!-- Page Builder Row generates this structure -->
<div class="page-builder-row" data-content-type="row" ...>
    <div class="row-container" ...>
```

Merchants can apply pre-defined CSS classes via the "Advanced" section of each element:

```
// Available CSS class categories in Page Builder:
- Alignment classes
- Background classes (color, image)
- Margin/Padding classes
- Border classes
- Responsive visibility classes
```

**Headless/PWA Integration with Page Builder:**

In headless architectures, Page Builder content is rendered server-side and served as HTML through the CMS API endpoints. The frontend (e.g., a PWA storefront) receives pre-rendered HTML that it can display directly or hydrate.

```php
<?php
// Rendering Page Builder content in a headless context
// The Page Builder content is stored as XML and rendered to HTML:

use Magento\Cms\Model\Template\FilterProvider;

class CmsContentRenderer
{
    public function __construct(
        private readonly FilterProvider $filterProvider,
    ) {}

    public function renderPageContent(string $content): string
    {
        // Page Builder stores content as encoded XML
        // FilterProvider processes it and returns rendered HTML
        return $this->filterProvider->getPageFilter()->filter($content);
    }
}
```

---

### Topic 4: Extension Attributes — Adding Fields to API Entities

Extension attributes are the correct mechanism for adding custom fields to Magento entities that need to be exposed via API (REST or GraphQL). They are NOT stored in the entity's EAV tables — they live in separate tables or are computed at runtime.

**The Core Problem Extension Attributes Solve:**

Magento's core entities (Product, Order, Customer, etc.) have fixed database schemas. Adding columns directly to the core tables is bad practice because:

1. Core tables change in every Magento release
2. API responses are generated from service contracts and extension attributes
3. Custom columns in core tables may be overwritten during upgrades

**Declaring Extension Attributes:**

Extension attributes are declared in `extension_attributes.xml`:

```xml
<?xml version="1.0"?>
<!-- app/code/Training/Product/etc/extension_attributes.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Extension/etc/extension_attributes.xsd">

    <!-- Extension attribute for Product entity -->
    <extension_attributes for="Magento\Catalog\Api\Data\ProductInterface">
        <attribute code="warranty_period" type="string">
            <join reference_table="training_product_warranty"
                  join_field="product_id"
                  filter="deleted=0">
                <column name="warranty_months"/>
            </join>
        </attribute>
    </extension_attributes>
</config>
```

> **How the `<join>` works:** The `join` clause tells Magento's extension attribute system to JOIN to a separate table (`training_product_warranty`) when loading the product. The `join_field` specifies the column in the join table that maps to the product. This is the "separate table" approach — the warranty data lives in `training_product_warranty`, not in `catalog_product_entity_*`.

**Database Schema for Extension Attribute Storage:**

When using a `<join>`, you must create the reference table via `db_schema.xml`:

```xml
<?xml version="1.0"?>
<!-- app/code/Training/Product/etc/db_schema.xml -->
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup:Declaration:etc/db_schema.xsd">
    <table name="training_product_warranty"
           resource="default"
           comment="Product Warranty Extension Attribute Table"
           engine="innodb">
        <column xsi:type="int" name="entity_id" padding="10" unsigned="true"
                identity="true" nullable="false" primary="true"/>
        <column xsi:type="int" name="product_id" padding="10" unsigned="true"
                nullable="false"/>
        <column xsi:type="int" name="warranty_months" padding="3" unsigned="true"
                nullable="false" default="12"/>
        <column xsi:type="smallint" name="deleted" padding="1" unsigned="true"
                nullable="false" default="0"/>
        <constraint xsi:type="foreign" referenceId="TRAINING_WARRANTY_PRODUCT_ID"
                    table="training_product_warranty" column="product_id"
                    referenceTable="catalog_product_entity" referenceColumn="entity_id"
                    onDelete="CASCADE"/>
        <index referenceId="TRAINING_WARRANTY_PRODUCT_IDX" table="training_product_warranty">
            <column name="product_id"/>
        </index>
    </table>
</schema>
```

**Entity Interface Implementation:**

The entity interface must declare `getExtensionAttributes()` and `setExtensionAttributes()`:

```php
<?php
// Magento\Catalog\Api\Data\ProductInterface (already has these in core)
interface ProductInterface extends \Magento\Framework\Api\ExtensibleDataInterface
{
    // ... existing methods ...

    /**
     * Return the extension attributes for the product.
     */
    public function getExtensionAttributes(): ?\Magento\Catalog\Api\Data\ProductExtensionInterface;

    /**
     * Set the extension attributes for the product.
     */
    public function setExtensionAttributes(\Magento\Catalog\Api\Data\ProductExtensionInterface $extensionAttributes): ProductInterface;
}
```

> **Core already implements these methods.** You do NOT need to modify the core `ProductInterface` or `Product` model. You only need to create your `ProductExtensionInterface` with your custom fields, and the extension attribute system handles the rest.

**Creating the Extension Interface:**

```php
<?php
// Api/Data/ProductWarrantyExtensionInterface.php
declare(strict_types=1);

namespace Training\Product\Api\Data;

use Magento\Framework\Api\ExtensionAttributesInterface;

interface ProductWarrantyExtensionInterface extends ExtensionAttributesInterface
{
    public function getWarrantyMonths(): ?int;
    public function setWarrantyMonths(?int $warrantyMonths): self;
}
```

**Populating Extension Attributes via Plugin:**

If the extension attribute is not a simple JOIN (e.g., it requires external system lookup or complex computation), use a plugin on the product repository:

```php
<?php
// Plugin/ProductRepository/WarrantyLoaderPlugin.php
declare(strict_types=1);

namespace Training\Product\Plugin\ProductRepository;

use Magento\Catalog\Api\Data\ProductInterface;
use Magento\Catalog\Api\Data\ProductSearchResultsInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;

class WarrantyLoaderPlugin
{
    private WarrantyService $warrantyService;

    public function __construct(WarrantyService $warrantyService)
    {
        $this->warrantyService = $warrantyService;
    }

    /**
     * After getting a single product, populate its warranty extension attribute.
     */
    public function afterGet(
        ProductRepositoryInterface $subject,
        ProductInterface $product
    ): ProductInterface {
        $warrantyData = $this->warrantyService->getWarrantyForProduct($product->getId());

        $extensionAttributes = $product->getExtensionAttributes() ?? $this->extensionFactory->create(ProductInterface::class);
        $extensionAttributes->setWarrantyMonths($warrantyData['months']);

        $product->setExtensionAttributes($extensionAttributes);
        return $product;
    }

    /**
     * After getting a product list, populate warranty for each product.
     */
    public function afterGetList(
        ProductRepositoryInterface $subject,
        ProductSearchResultsInterface $searchResult
    ): ProductSearchResultsInterface {
        $products = $searchResult->getItems();
        $productIds = array_map(fn(ProductInterface $p) => $p->getId(), $products);
        $warrantyData = $this->warrantyService->getWarrantyForProducts($productIds);

        foreach ($products as $product) {
            $warrantyMonths = $warrantyData[$product->getId()] ?? null;
            $extensionAttributes = $product->getExtensionAttributes()
                ?? $this->extensionFactory->create(ProductInterface::class);
            $extensionAttributes->setWarrantyMonths($warrantyMonths);
            $product->setExtensionAttributes($extensionAttributes);
        }

        return $searchResult;
    }
}
```

**Common Use Cases for Extension Attributes:**

| Use Case | Why Extension Attribute | Alternative Considered |
|----------|----------------------|----------------------|
| Product warranty period (API only) | External table, not EAV, not in layered nav | Could be EAV but not filterable |
| Order approval metadata (B2B) | Computed from workflow engine, not stored | Plugin resolves at runtime |
| Customer loyalty tier | From external loyalty system, API-only | EAV if also used for display |
| Gift message extended | Added by gift wrapping module, complex relationship | Join table |
| Shipping method metadata | Requires carrier API lookup, computed | Plugin pattern |

**When Extension Attributes Are the Right Choice:**

- The field should be exposed in the REST/GraphQL API
- The field is NOT used for filtering, sorting, or layered navigation on the storefront
- The field may come from an external system or require complex computation
- You don't want the field in the main EAV tables

---

### Topic 5: Custom Attributes vs Extension Attributes vs EAV

This topic clarifies the decision boundary between three distinct mechanisms for adding custom data to Magento entities. Using the wrong approach leads to performance problems, API inconsistency, or inability to filter on the storefront.

**EAV (Entity-Attribute-Value) — The Foundation:**

EAV is Magento's dynamic attribute system, primarily used for Products, Customers, and Categories. Rather than having fixed columns for every possible attribute, EAV stores attribute values in separate tables keyed by entity type.

```sql
-- Product EAV structure (simplified)
catalog_product_entity          -- entity (product) table
├── entity_id
├── attribute_set_id
├── type_id
└── created_at

catalog_product_entity_varchar  -- values for varchar attributes
├── value_id
├── attribute_id  (FK → eav_attribute.attribute_id)
├── store_id      (FK → store/store_id — supports per-store values)
├── entity_id     (FK → catalog_product_entity.entity_id)
└── value

catalog_product_entity_int      -- values for integer attributes
catalog_product_entity_decimal   -- values for decimal attributes
catalog_product_entity_datetime  -- values for datetime attributes
catalog_product_entity_text     -- values for text/HTML attributes
```

The `eav_attribute` table defines the attribute metadata:

```sql
eav_attribute
├── attribute_id
├── entity_type_id         (maps to customer, product, category)
├── attribute_code         (e.g., "color", "meta_description")
├── backend_model          (e.g., how to load/save/validate)
├── frontend_model         (how to render in admin)
├── source_model           (for dropdown/select attributes)
└── is_filterable          (can appear in layered nav)
```

**Custom Attributes (Setup Script Approach):**

Custom attributes are EAV attributes added via `Setup\Patch\Data` or `UpgradeData` scripts:

```php
<?php
// Setup/Patch/Data/AddProductCustomAttributes.php
declare(strict_types=1);

namespace Training\Product\Setup\Patch\Data;

use Magento\Catalog\Model\Product;
use Magento\Eav\Setup\EavSetup;
use Magento\Eav\Setup\EavSetupFactory;
use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class AddProductCustomAttributes implements DataPatchInterface
{
    private ModuleDataSetupInterface $moduleDataSetup;
    private EavSetupFactory $eavSetupFactory;

    public function __construct(
        ModuleDataSetupInterface $moduleDataSetup,
        EavSetupFactory $eavSetupFactory
    ) {
        $this->moduleDataSetup = $moduleDataSetup;
        $this->eavSetupFactory = $eavSetupFactory;
    }

    public function apply(): void
    {
        /** @var EavSetup $eavSetup */
        $eavSetup = $this->eavSetupFactory->create(['setup' => $this->moduleDataSetup]);

        // Add a text attribute
        $eavSetup->addAttribute(
            Product::ENTITY,
            'featured_product',   // attribute code
            [
                'type' => 'int',               // int, varchar, decimal, text, datetime
                'group' => 'General',          // Attribute group in admin
                'label' => 'Featured Product',
                'input' => 'boolean',          // text, select, boolean, date, textarea
                'source' => \Magento\Eav\Model\Entity\Attribute\Source\Boolean::class,
                'default' => 0,
                'global' => \Magento\Catalog\Model\ResourceModel\Eav\Attribute::SCOPE_WEBSITE,
                'filterable' => true,          // Appears in layered navigation
                'searchable' => true,           // Searchable in catalog search
                'used_in_product_listing' => true,  // Appears on category pages
                'required' => false,
                'user_defined' => true,
            ]
        );

        // Add a select/dropdown attribute with options
        $eavSetup->addAttribute(
            Product::ENTITY,
            'product_condition',
            [
                'type' => 'varchar',
                'group' => 'General',
                'label' => 'Condition',
                'input' => 'select',
                'source' => \Magento\Eav\Model\Entity\Attribute\Source\Table::class,
                'options' => [
                    'New' => 'New',
                    'Refurbished' => 'Refurbished',
                    'Used' => 'Used',
                ],
                'global' => \Magento\Catalog\Model\ResourceModel\Eav\Attribute::SCOPE_GLOBAL,
                'filterable' => true,
                'required' => false,
            ]
        );
    }

    public function getAliases(): array { return []; }
    public static function getDependencies(): array { return []; }
}
```

> **Important:** When using `source` with a Table source (`\Magento\Eav\Model\Entity\Attribute\Source\Table`), Magento auto-creates `eav_attribute_option_*` tables to store the options. If using a custom source model, you must implement option loading yourself.

**Decision Matrix — When to Use Each Approach:**

| Requirement | Approach | Notes |
|-------------|---------|-------|
| Display on storefront product page | Custom attribute (EAV) | Automatically loaded with product |
| Filterable in layered navigation | Custom attribute (EAV) | Set `filterable => true` |
| Searchable in catalog search | Custom attribute (EAV) | Set `searchable => true` |
| Admin-only field (not in API) | Custom attribute (EAV) | Don't set `used_in_product_listing` |
| API-only field (not in storefront) | Extension attribute | Keeps EAV tables clean |
| Field from external system | Extension attribute | Populated via plugin at read time |
| Complex computation to derive value | Extension attribute | Computed in plugin, not stored |
| Field needs to be sortable | Custom attribute (EAV) | Extension attrs can be sorted but require custom query |
| Part of an attribute set | Custom attribute (EAV) | Extension attributes are not part of attribute sets |
| Customer segment-specific data | Extension attribute | Resolved by segment plugin |

**Performance Implications:**

| Approach | Read Performance | Write Performance | Storage |
|----------|----------------|------------------|---------|
| EAV Custom attribute | Each attribute type adds a JOIN. 10 custom attrs = up to 10 JOINs on product load. Use `product-reads-batch` and `collectTierPrices` profiling to monitor. | Each attribute write may update multiple tables (global + store scope). | Spreads across `*_entity_*` tables |
| Extension attribute (JOIN) | One extra JOIN per extension attribute. Typically better than EAV for sparse data. | Single table write. No multi-table updates. | Separate table |
| Extension attribute (Plugin) | No extra JOIN — computed on demand. Adds latency to `getList` calls. | No extra writes if computed. | No storage if purely computed |

> **Pro Tip:** Use EAV custom attributes when the field is dense (most products have a value) and needs to be filtered. Use extension attributes when the field is sparse (few products have a value) or is read from an external system.

**Combining Both Approaches:**

A common pattern is using both — EAV for storefront filtering/search and extension attributes for API enrichment:

```php
<?php
// A product has:
// 1. "featured" as EAV int attribute (filterable in layered nav, shown on category page)
// 2. "warranty_months" as extension attribute (API-only, joined from warranty table)

// In GraphQL resolver, you can access both:
$product->getData('featured');                    // EAV custom attribute
$product->getExtensionAttributes()->getWarrantyMonths();  // Extension attribute
```

---

### Topic 6: CMS in Headless Architectures

Headless Magento separates the frontend (presentation layer) from the backend (Magento). In headless setups, the frontend is typically a PWA (Progressive Web App), a mobile app, or a separate frontend framework. CMS content must be accessible via API.

**CMS Content via REST API:**

Magento exposes CMS blocks and pages through the REST API:

```bash
# Get a single CMS block
GET /rest/V1/cmsBlock/{blockId}

# Get CMS blocks with search criteria
GET /rest/V1/cmsBlocks?searchCriteria[filter_groups][0][filters][0][field]=identifier&searchCriteria[filter_groups][0][filters][0][value]=footer-links

# Get a CMS page
GET /rest/V1/cmsPage/{pageId}

# Get CMS page by URL key
GET /rest/V1/cmsPage祈祷byIdentifier?identifier=about-us
```

The corresponding PHP implementation uses the repository interfaces:

```php
<?php
// Model/CmsApi/BlockFetcher.php
declare(strict_types=1);

namespace Training\Cms\Model\CmsApi;

use Magento\Cms\Api\BlockRepositoryInterface;
use Magento\Cms\Api\Data\BlockInterface;

class BlockFetcher
{
    public function __construct(
        private readonly BlockRepositoryInterface $blockRepository,
    ) {}

    /**
     * Fetch a CMS block by its identifier for API response.
     */
    public function getBlockByIdentifier(string $identifier): ?BlockInterface
    {
        // Note: BlockRepository::getById takes an ID (integer), not identifier
        // You need to search via getList with a filter
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('identifier', $identifier, 'eq')
            ->create();

        $result = $this->blockRepository->getList($searchCriteria);
        $items = $result->getItems();

        return $items ? reset($items) : null;
    }

    /**
     * Fetch block and render its content through the CMS filter.
     */
    public function getRenderedBlockContent(string $identifier): ?string
    {
        $block = $this->getBlockByIdentifier($identifier);

        if (!$block || !$block->getIsActive()) {
            return null;
        }

        // Process {{store url="..."}} and {{widget ...}} directives
        return $this->filterProvider->getBlockFilter()->filter($block->getContent());
    }
}
```

**CMS Content via GraphQL:**

GraphQL provides a more flexible query mechanism — you can request exactly the fields you need:

```graphql
# Query a CMS block via GraphQL
query {
    cmsBlock(id: 5) {
        id
        title
        identifier
        content
        content_html
    }
}

# Query a CMS page via GraphQL
query {
    cmsPage(id: 3) {
        id
        title
        identifier
        content
        meta_title
        meta_keywords
    }
}
```

**Processing CMS Content with Variables:**

CMS content often contains variable directives like `{{store url="page"}}` and widget directives. In headless contexts, these must be processed server-side before sending to the frontend:

```php
<?php
// In a custom GraphQL resolver or API controller
use Magento\Cms\Model\Template\FilterProvider;
use Magento\Framework\Filter\Template;

class CmsContentProcessor
{
    public function __construct(
        private readonly FilterProvider $filterProvider,
    ) {}

    /**
     * Process CMS content directives (store_url, widget, directive variables).
     * Must be called server-side — PWA cannot process these directives.
     */
    public function processContent(string $content, int $storeId): string
    {
        // Set the correct store context for {{store url="..."}} directives
        $this->filterProvider->setStoreId($storeId);

        // Process block filter (handles widgets and directives)
        return $this->filterProvider->getBlockFilter()->filter($content);
    }
}
```

> **In headless, all CMS content processing must be server-side.** The PWA/headless frontend cannot process `{{widget}}` or `{{store url}}` directives. Always resolve these on the Magento server before returning content.

**Caching Strategy for Headless CMS:**

| Strategy | Pros | Cons |
|----------|------|------|
| Server-side render + cache | Fast delivery, works with CDN | Must invalidate on CMS change |
| Cache per-store + per-block | Granular invalidation | More cache keys to manage |
| No caching (dynamic each request) | Always fresh | High server load |

```php
<?php
// In your headless CMS API service
use Magento\Framework\App\CacheInterface;
use Magento\Cms\Api\BlockRepositoryInterface;

class CachedCmsContent
{
    private const CACHE_TAG = 'cms_content';
    private const CACHE_LIFETIME = 86400; // 24 hours

    public function __construct(
        private readonly BlockRepositoryInterface $blockRepository,
        private readonly FilterProvider $filterProvider,
        private readonly CacheInterface $cache,
    ) {}

    public function getRenderedBlock(string $identifier, int $storeId): ?string
    {
        $cacheKey = sprintf('block_%s_store_%d', $identifier, $storeId);
        $cached = $this->cache->load($cacheKey);

        if ($cached !== false) {
            return $cached;
        }

        $block = $this->getBlockByIdentifier($identifier, $storeId);
        if (!$block) {
            return null;
        }

        $rendered = $this->filterProvider->getBlockFilter()->filter($block->getContent());

        $this->cache->save($rendered, $cacheKey, [self::CACHE_TAG], self::CACHE_LIFETIME);
        return $rendered;
    }
}
```

**Widgets in Headless / PWA Contexts:**

There are two approaches for widgets in headless:

1. **Server-side render:** The widget is rendered on the Magento server as an HTML string, embedded in the CMS content, and sent to the PWA as plain HTML. The PWA displays it directly.

2. **Client-side hydration:** The widget placeholder (e.g., `<div data-widget="hello-world" data-props='{"greeting":"Hi"}'>`) is sent to the PWA, which then renders the widget using a JavaScript widget library on the client.

The server-side approach is simpler and is what Magento's native CMS API does. The client-side approach requires building a custom widget JavaScript SDK on the PWA side.

---

### Topic 7: CMS Block Permissions and Preview

Understanding CMS content visibility controls and preview mechanisms is essential for content staging and draft workflows.

**Block Active Status:**

The `is_active` flag on CMS blocks controls whether the block is rendered:

```php
<?php
// When rendering a block via getLayout()->createBlock()->toHtml()
// The block class checks is_active internally:

// Magento\Cms\Block\Block::_toHtml()
if (!$this->getBlockId() || !$this->getIsActive()) {
    return '';
}
```

A block with `is_active = 0` renders nothing — it is effectively invisible everywhere.

**Store-Level Visibility for Pages:**

CMS pages are assigned to stores via the `cms_page_store` join table. A page assigned to `store_id = 0` (or not present in `cms_page_store`) is shown on all stores.

```php
<?php
// Assigning a CMS page to specific stores:
$page->setStores([0]);          // All stores
$page->setStores([1, 2]);       // Only stores 1 and 2
```

The PageRepository respects store scope when loading pages:

```php
// When loading a page for store 1:
// 1. Check if page exists in cms_page_store for store_id=1
// 2. If not found, check if page exists with store_id=0 (all stores)
// 3. If neither, page not found for this store
```

**Preview Mode for Draft Content:**

In Adobe Commerce (EE), CMS pages can be in draft status and previewed before publishing. In CE, there is no formal draft/publish workflow — pages are either `is_active=1` (visible) or `is_active=0` (hidden).

For EE preview functionality, the URL parameter `___store=preview_code` switches the storefront to preview mode:

```
# Preview a CMS page before it goes live
https://example.com/about-us?utm_content ___store=preview_code&id=45

# Where:
# - about-us = the URL key of the page
# - preview_code = a store view code configured for preview
# - id = staging update ID (EE content staging feature)
```

> **CE Preview Limitation:** Without EE's content staging, there's no built-in preview workflow for CMS pages. You can simulate preview by setting `is_active=0`, sharing the direct URL with reviewers, then setting `is_active=1` when ready. This is a manual process.

**BlockInterface and PageInterface — Entity Data Structures:**

Magento's CMS entities implement data interfaces that define the contract for the API layer:

```php
<?php
// Magento\Cms\Api\Data\BlockInterface
interface BlockInterface extends \Magento\Framework\Api\ExtensibleDataInterface
{
    public function getId(): ?int;
    public function setId(?int $id): BlockInterface;
    public function getTitle(): ?string;
    public function setTitle(string $title): BlockInterface;
    public function getIdentifier(): ?string;
    public function setIdentifier(string $identifier): BlockInterface;
    public function getContent(): ?string;
    public function setContent(string $content): BlockInterface;
    public function getCreationTime(): ?string;
    public function getUpdateTime(): ?string;
    public function getIsActive(): bool;
    public function setIsActive(bool $isActive): BlockInterface;
    public function getExtensionAttributes(): ?\Magento\Cms\Api\Data\BlockExtensionInterface;
    public function setExtensionAttributes(\Magento\Cms\Api\Data\BlockExtensionInterface $extensionAttributes): BlockInterface;
}

// Magento\Cms\Api\Data\PageInterface — similar structure with additional page-specific fields
interface PageInterface extends \Magento\Framework\Api\ExtensibleDataInterface
{
    public function getId(): ?int;
    public function setId(?int $id): PageInterface;
    public function getTitle(): ?string;
    public function setTitle(string $title): PageInterface;
    public function getPageLayout(): ?string;
    public function setPageLayout(string $layout): PageInterface;
    public function getMetaTitle(): ?string;
    public function getMetaKeywords(): ?string;
    public function getMetaDescription(): ?string;
    // ... many more page-specific fields
}
```

**Preview URL Construction in EE:**

```php
<?php
// In EE, the preview URL is constructed by the staging module:
use Magento\Staging\Model\Preview\UrlBuilder;

class PreviewUrlGenerator
{
    public function __construct(
        private readonly UrlBuilder $urlBuilder,
    ) {}

    public function generatePreviewUrl(int $updateId, int $storeId): string
    {
        return $this->urlBuilder->getPreviewUrl($updateId, $storeId);
        // Returns: https://example.com/cms-page-url?id={updateId}&_store=preview_code
    }
}
```

---

### Topic 8: Widget Instance Assignment and Rendering

Widget instances are specific configured placements of a widget type on a page or block. Understanding how instances are stored and rendered is key to debugging widget issues.

**Widget Instance Storage — Deep Dive:**

```sql
widget_instance
├── instance_id
├── instance_type          -- PHP class (e.g., Magento\Cms\Block\Widget\Block)
├── package_theme          -- e.g., "Magento/luma"
├── widget_parameters      -- JSON: {"block_id": "5", "template": "default"}
├── sort_order
├── store_ids              -- comma-separated store IDs (e.g., "0" = all)
├── page_id                -- CMS page ID (if page-specific)
├── block_id               -- CMS block ID (if block-specific)
├── layout_handle          -- For specific layout handle assignment (e.g., "default")
└── entity_ids             -- JSON: specific page/block IDs for placement
```

**Widget Instance Types:**

| Assignment Type | How It's Set | Example |
|----------------|-------------|---------|
| Specific page | Selected in admin widget form | Homepage only |
| Specific block | Selected in admin widget form | Footer block only |
| All pages | Layout handle "default" | Every page |
| Catalog category | Layout handle "catalog_category_view" | Category pages only |
| Catalog product | Layout handle "catalog_product_view" | Product pages only |

**Widget Placement via Layout XML:**

When you assign a widget to a specific container via the admin, Magento generates layout XML. This XML is stored in the `widget_instance` table and applied during layout generation:

```xml
<!-- Rendered/generated layout XML for a widget instance -->
<reference name="content">
    <block type="Magento\Cms\Block\Widget\Block"
           name="cms.widget.instance.1">
        <action method="setBlockId">
            <argument name="blockId" xsi:type="string">footer-links</argument>
        </action>
        <action method="setTemplate">
            <argument name="template" xsi:type="string">Magento_Cms::widget/block.phtml</argument>
        </action>
    </block>
</reference>
```

**Widget Rendering Flow:**

```
1. HTTP Request → Magento\Framework\App\FrontController
2. Layout generation (Magento\Framework\View\Layout)
3. Widget block is instantiated (type=Magento\Cms\Block\Widget\Block)
4. Block receives widget_parameters from layout XML
5. Block._toHtml() is called
6. If block_id is set: load CMS block from repository, apply filter
7. Return rendered HTML
```

```php
<?php
// Magento\Cms\Block\Widget\Block — the built-in CMS block widget
class Block extends \Magento\Cms\Block\Block implements \Magento\Widget\Block\BlockInterface
{
    protected $_template = 'Magento_Cms::widget/block.phtml';

    protected function _toHtml()
    {
        $blockId = $this->getBlockId();
        if (!$blockId) {
            return '';
        }

        $block = $this->blockRepository->getById($blockId);

        if (!$block || !$block->getIsActive()) {
            return '';
        }

        $html = $this->_filterProvider->getBlockFilter()->filter($block->getContent());
        return $html;
    }
}
```

**Widget Instance Admin Grid:**

The widget instance admin grid (`Content → Widgets`) lists all widget instances and allows editing their configurations. The edit form is built from the widget's declared parameters in `widget.xml`.

```php
<?php
// The widget instance edit form references widget.xml to build the UI:
// Magento\Widget\Block\Adminhtml\Widget\Instance\Edit\Form
// This form iterates over the widget's <parameters> and renders appropriate form fields
```

**Custom Widget Instance Layout Updates:**

You can also programmatically add widget instances via `di.xml` and layout XML without using the admin:

```xml
<!-- In your module's view/frontend/layout/default.xml -->
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceContainer name="sidebar.additional">
            <block type="Training\HelloWorld\Block\Widget\HelloWorld"
                   name="training.hello.world.sidebar"
                   template="Training_HelloWorld::widget/hello.phtml">
                <action method="setData">
                    <argument name="key" xsi:type="string">greeting</argument>
                    <argument name="value" xsi:type="string">Welcome to our site!</argument>
                </action>
                <action method="setData">
                    <argument name="key" xsi:type="string">style</argument>
                    <argument name="value" xsi:type="string">friendly</argument>
                </action>
            </block>
        </referenceContainer>
    </body>
</page>
```

> **This approach bypasses the admin widget instance system entirely.** It's a developer-managed placement. The widget's parameters are hardcoded in layout XML. Content editors won't be able to modify these parameters from the admin.

---

### Topic 9: Extension Attributes via GraphQL

Extension attributes become powerful when exposed through GraphQL — this is the primary API for headless Magento storefronts. The GraphQL layer provides type-safe, flexible access to extension data.

**Declaring Extension Attributes for GraphQL:**

Extension attributes declared in `extension_attributes.xml` are automatically available in REST. For GraphQL, you need to also declare the GraphQL schema extension.

**Step 1: Declare Extension Attributes (already done in Topic 4):**

```xml
<!-- Training/Product/etc/extension_attributes.xml -->
<extension_attributes for="Magento\Catalog\Api\Data\ProductInterface">
    <attribute code="warranty_period" type="string">
        <join reference_table="training_product_warranty"
              join_field="product_id">
            <column name="warranty_months"/>
        </join>
    </attribute>
</extension_attributes>
```

**Step 2: Create a GraphQL Resolver for the Extension Attribute:**

```php
<?php
// Model/Resolver/ProductWarranty.php
declare(strict_types=1);

namespace Training\Product\Model\Resolver;

use Magento\Catalog\Api\Data\ProductInterface;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\GraphQl\Query\ResolverInterface;

class ProductWarranty implements ResolverInterface
{
    private WarrantyService $warrantyService;

    public function __construct(WarrantyService $warrantyService)
    {
        $this->warrantyService = $warrantyService;
    }

    /**
     * Resolve the warranty_period extension attribute for a product.
     *
     * @param \Magento\Framework\GraphQl\Config\Element\Field $field
     * @param array $context  — GraphQL context (contains current user, store, etc.)
     * @param \Magento\Framework\GraphQl\Query\Resolver\Value $value  — resolved parent value
     * @param array $args     — GraphQL query arguments
     */
    public function resolve(
        \Magento\Framework\GraphQl\Config\Element\Field $field,
        $context,
        \Magento\Framework\GraphQl\Query\Resolver\Value $value,
        array $args = null
    ): ?string {
        if (!isset($value['model'])) {
            return null;
        }

        /** @var ProductInterface $product */
        $product = $value['model'];
        $extensionAttributes = $product->getExtensionAttributes();

        if (!$extensionAttributes) {
            return null;
        }

        $warrantyMonths = $extensionAttributes->getWarrantyMonths();

        if ($warrantyMonths === null) {
            return null;
        }

        return sprintf('%d months', $warrantyMonths);
    }
}
```

**Step 3: Register the Resolver in di.xml for GraphQL:**

```xml
<?xml version="1.0"?>
<!-- Training/Product/etc/graphql/di.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Register the resolver for the extension attribute -->
    <type name="Magento\Framework\GraphQl\Query\Resolver\CompositeResolver">
        <arguments>
            <argument name="resolvers">
                <item name="warranty_period">
                    Training\Product\Model\Resolver\ProductWarranty
                </item>
            </argument>
        </arguments>
    </type>
</config>
```

> **Alternative approach:** Instead of `CompositeResolver`, you can also use a plugin on `Magento\Framework\GraphQl\Schema\Type\Input\Source\ExtensionAttributeSource` or a dedicated `extension_attributes` schema generator. The CompositeResolver approach is simpler and works well for custom extension attributes.

**Step 4: Define the GraphQL Schema Extension:**

You need to extend the Product type in the GraphQL schema to declare the new field:

```graphql
# In your module's etc/graphqrschema/product_extension_attributes.graphqls
# (Magento looks for files matching *extension_attributes*.graphqls)

extend type Product {
    warranty_period: String @doc(description: "Extended warranty period in months")
}
```

> **How Magento merges extension schemas:** Files matching `*extension_attributes*.graphqls` in any enabled module are automatically merged into the global schema by `Magento\Framework\GraphQl\Schema\SchemaGenerator`. This is how your extension attribute field becomes part of the Product type.

**Querying Extension Attributes in GraphQL:**

```graphql
# Query products with the warranty_period extension attribute
query {
    products(filter: {name: {match: "Laptop"}}) {
        items {
            id
            name
            sku
            warranty_period  # Your custom extension attribute
        }
    }
}
```

**Fragment Usage for Extension Attributes:**

```graphql
# Define a fragment to reuse extension attribute selections
fragment ProductWithWarranty on Product {
    id
    name
    sku
    warranty_period
    price {
        regularPrice {
            amount {
                value
            }
        }
    }
}

query {
    products(filter: {category_id: {eq: "10"}}) {
        items {
            ...ProductWithWarranty
        }
    }
}
```

**Extension Attributes on Customer and Order via GraphQL:**

The same pattern applies to Customer and Order entities:

```graphql
# Customer extension attribute (e.g., loyalty tier)
# etc/graphql/customer_extension_attributes.graphqls
extend type Customer {
    loyalty_tier: String @doc(description: "Customer loyalty program tier")
}

# Order extension attribute (e.g., approval metadata)
# etc/graphql/order_extension_attributes.graphqls
extend type Order {
    approval_status: String @doc(description: "B2B order approval status")
}
```

The resolvers follow the same pattern — implement `ResolverInterface` and register in `di.xml`.

> **Performance note for GraphQL extension attributes:** Each extension attribute resolver is called once per item in the result set. If you have 30 products and 5 extension attributes, your resolver chain runs 150 times (30 × 5). Use batch loading in your resolver's service class to avoid N+1 query problems.

---

### Topic 10: Content Staging Integration (EE Perspective)

Content Staging is one of Adobe Commerce's most valuable enterprise features. It allows merchants to schedule content changes (CMS pages, blocks, promotions) in advance and preview them before they go live. This topic is covered from a CE developer's perspective — understanding EE's architecture helps you build CE-compatible code that won't break when EE is layered on top.

**Content Staging Architecture Overview:**

Content Staging in EE works by creating a versioning layer on top of the base CMS entities. The `magento_staging` module introduces:

1. **Staging Updates** — Each scheduled change is a `staging_update` record with a start time and end time
2. **Entity Staging** — CMS pages and blocks are linked to staging updates via `*_staging` pivot tables
3. **Preview** — EE generates preview URLs that show what content will look like at a future date

**Core Staging Tables:**

```sql
-- staging_update: a scheduled moment in time
staging_update
├── id              (unique identifier)
├── start_date      (when this update goes live)
├── end_date        (optional, when this update expires)
├── is_rollback     (boolean — marks if this is a rollback)
└── rollback_id     (FK to previous staging_update)

-- cms_page_staging: links CMS pages to staging updates
cms_page_staging
├── id
├── page_id          (FK → cms_page.page_id)
├── update_id        (FK → staging_update.id)
└── row_id           (staging-specific row identifier)

-- cms_block_staging: links CMS blocks to staging updates
cms_block_staging
├── id
├── block_id         (FK → cms_block.block_id)
├── update_id        (FK → staging_update.id)
└── row_id
```

**Content Staging Workflow:**

```
1. Content Editor creates a "Campaign" (staging_update) for December 25th
2. Editor modifies a CMS page (adds holiday banner) and assigns it to the campaign
3. The change is stored in cms_page_staging with update_id pointing to the campaign
4. On December 25th, Magento's cron job activates the update
5. The CMS page now shows the holiday banner
6. On January 2nd, another cron job rolls back (or the update expires)
```

**CE Developer Implications — Building for Compatibility:**

When you build custom CMS functionality in CE, your code should be aware of how EE's staging layer operates:

```php
<?php
// ❌ WRONG: Directly loading a CMS page without considering staging context
$page = $this->pageRepository->getById($pageId);
$content = $page->getContent();

// ✅ BETTER: In CE this works identically. In EE, the staging-aware model
// automatically returns the content for the current time/context.
$page = $this->pageRepository->getById($pageId);
$content = $page->getContent();

// ✅ BEST: Use the staging-aware repository when available
// (In CE this is the same repository; in EE it's wrapped by staging)
if ($this->moduleManager->isOutputEnabled('Magento_Staging')) {
    // EE path: use Magento/Staging\Model\Entity\StagingManagement
    // to preview what content will look like at a future date
}
```

**Staging-Aware CMS Page Loading in EE:**

```php
<?php
// In EE, the PageRepository is wrapped by a staging interceptor
// that intercepts getById() to return the appropriate versioned content
// based on the current preview context.

// If you're viewing preview mode, you get:
// - The modified content from the staged update
// - Even if the staged update hasn't gone live yet

use Magento\Staging\Model\Entity\StagingManagement;

class PreviewController
{
    public function __construct(
        private readonly StagingManagement $stagingManagement,
    ) {}

    public function previewPage(int $pageId, int $updateId): string
    {
        // Get the page content as it will appear at updateId's start_date
        $content = $this->stagingManagement->getPreviewContent(
            $pageId,
            $updateId,
            \Magento\Cms\Api\PageRepositoryInterface::class
        );
        return $content;
    }
}
```

**Preview URL Structure in EE:**

```
https://store.example.com/about-us
    ?id={staging_update_id}          # The scheduled update to preview
    &_store=preview_store_code       # A store view configured for preview

# Server-side:
# 1. Magento reads the id parameter
# 2. Looks up the staging_update record
# 3. Loads the CMS page content as it will be at update.start_date
# 4. Renders with the preview store's theme/design
```

**B2B Staging — Company-Specific Content:**

In EE with B2B, content staging can be combined with customer segments to show different content to different company accounts:

```
Campaign: "VIP Holiday Sale"
├── Staged for: December 1 - December 31
├── Applied to: All customers
└── BUT: VIP tier customers see an enhanced banner
```

This is achieved by layering B2B's customer segmentation on top of the staging system.

**CE Development Strategy with EE Awareness:**

| CE Approach | EE Layer | Implication |
|------------|---------|-------------|
| CMS pages with `is_active` flag | EE adds scheduled activation via staging | Your CE code works in EE, staging is additive |
| CMS blocks with `is_active` flag | EE adds versioned block content | Your block queries should use repository (not direct model) |
| Custom CMS entities | EE may need explicit staging support | Design your entities to support `created_in` / `updated_in` fields |
| Widgets in CMS content | EE staging preview works with widgets | Widgets must implement cache correctly for preview to work |

> **Practical CE advice:** Always use the repository interfaces (`BlockRepositoryInterface`, `PageRepositoryInterface`) for CMS entity access. These are the contracts that EE's staging layer wraps. Direct model access bypasses the repository and can cause issues when EE staging is layered on top.

**Key Takeaway:**

Content Staging is EE-only, but understanding its architecture helps you:
1. Write CE code that won't break when EE staging is enabled
2. Design custom CMS entities that could support staging in the future
3. Advise clients accurately on what CE can and cannot do regarding scheduled content

---

## Reading List

1. **Magento 2 Developer Documentation — CMS Pages and Blocks**
   <https://developer.adobe.com/commerce/php/development/components/cms-content/>

2. **Magento 2 Developer Documentation — Widgets**
   <https://developer.adobe.com/commerce/php/development/components/widgets/>

3. **Magento 2 Developer Documentation — Page Builder**
   <https://developer.adobe.com/commerce/frontend-core/page-builder/>

4. **Adobe Commerce Developer Documentation — Extension Attributes**
   <https://developer.adobe.com/commerce/webapi/requests/extension-attributes/>

5. **Magento 2 Developer Documentation — GraphQL Extension Attributes**
   <https://developer.adobe.com/commerce/webapi/graphql/development/extensions/>

---

*Magento 2 Backend Developer Course — Topic 19*
