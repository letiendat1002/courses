---
title: "05 - CMS Page Layout & Page Builder"
description: "Magento 2.4.8 CMS: page layout XML, Page Builder, CMS block rendering, layout handles for CMS pages, and custom CMS content rendering"
tags: [magento2, cms, page-builder, layout-xml, cms-block, widgets]
rank: 5
pathways: [magento2-deep-dive]
see_also:
  - "Full layout XML reference: _supplemental/23-theme-development.md"
  - "Extension attributes: _supplemental/07-security-acl.md"
---

# CMS Page Layout, Page Builder, and CMS Rendering

Magento's CMS system powers all non-catalog content: landing pages, informational pages, the homepage, and any promotional content that lives outside the catalog. Understanding how CMS pages are rendered, how layout XML applies to them, and how Page Builder fits into the pipeline is essential for any non-trivial Magento build.

This chapter covers the full request lifecycle for a CMS page, the layout handle hierarchy, CMS block and widget directives, Page Builder's internal architecture, and the extension patterns you need when building custom CMS functionality.

---

## 1. How CMS Pages Are Rendered

### 1.1 Request Lifecycle

When a customer visits a CMS page (e.g., `/about-us`), the front controller dispatches the `cms` route, which resolves to `Magento\Cms\Controller\Index\Index`:

```php
// Magento\Cms\Controller\Index\Index
// vendor/magento/module-cms/Controller/Index/Index.php

public function execute()
{
    // 1. Load page by identifier (URL key)
    $page = $this->_pageFactory->create()->load($this->getRequest()->getParam('page_id'));

    // 2. If page not found, throw 404
    if (!$page || !$page->getId()) {
        throw new NotFoundException(__('Page not found'));
    }

    // 3. Apply layout handles
    $resultPage = $this->resultPageFactory->create();
    $resultPage->addHandle('cms_page_view');
    $resultPage->addHandle('cms_page_view_id_' . $page->getId());

    // 4. Register page in registry for use in templates
    $this->_coreRegistry->register('cms_page', $page);

    return $resultPage;
}
```

The `resultPageFactory` is a `Magento\Framework\View\Result\PageFactory`. Calling `create()` on it triggers Magento's layout generation pipeline:

1. **Handle collection** — all matched handles are collected (`default`, `cms_page_view`, `cms_page_view_id_{id}`, etc.)
2. **Layout merge** — XML from all matching handles is merged (theme + modules)
3. **Build** — the merged XML is converted into a block/container tree
4. **Render** — the page is rendered via the configured page layout (1-column, 2-columns-left, etc.)

### 1.2 The `cms_page_view.xml` Layout Handle

Every generic CMS page loads with the `cms_page_view` handle. This handle is defined in the `Magento_Cms` module:

```xml
<!-- vendor/magento/module-cms/view/frontend/layout/cms_page_view.xml -->
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceBlock name="page.top">
            <block class="Magento\Cms\Block\Page" name="cms_page"
                   template="Magento_Cms::page/view.phtml"/>
        </referenceBlock>
    </body>
</page>
```

The `cms_page` block (`Magento\Cms\Block\Page`) renders the CMS page's main content via `Magento_Cms::page/view.phtml`:

```php
// vendor/magento/module-cms/view/frontend/templates/page/view.phtml
$page = $block->getPage();
?>
<div class="cms-page-content">
    <?= /* @noEscape */ $block->getContent() ?>
</div>
```

### 1.3 Homepage vs Generic CMS Page

| URL | Controller | Layout Handle |
|-----|------------|---------------|
| `/` (homepage) | `Magento\Cms\Controller\Index\Index` with no `page_id` param | `cms_index_index` |
| `/cms-page-url-key` | `Magento\Cms\Controller\Index\Index` with `page_id` param | `cms_page_view` + `cms_page_view_id_{id}` |

The homepage handle `cms_index_index` extends `cms_page_view` and sets its own structure:

```xml
<!-- vendor/magento/module-cms/view/frontend/layout/cms_index_index.xml -->
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceContainer name="content">
            <block class="Magento\Cms\Block\Page" name="cms_page"/>
        </referenceContainer>
    </body>
</page>
```

### 1.4 CMS Content Storage

| Table | Purpose |
|-------|---------|
| `cms_page` | Page metadata: title, URL key (`identifier`), content, page_layout, meta tags, layout XML updates |
| `cms_page_store` | Many-to-many linking pages to stores |
| `cms_block` | Reusable block content: title, identifier, content |
| `cms_block_store` | Many-to-many linking blocks to stores |

The `cms_page.content` field stores raw HTML/text with block and widget directives. These directives are processed by `Magento\Cms\Model\Template\FilterProvider` during rendering.

```sql
SELECT page_id, title, identifier, content FROM cms_page LIMIT 5;
-- content column holds raw content with {{block}} and {{widget}} directives
```

---

## 2. CMS Page Layout Handles

### 2.1 Handle Hierarchy

```
default (all pages)
  └── cms_page_view (all CMS pages)
        └── cms_page_view_id_{page_id} (specific page by ID)
```

The `CMS Page Controller` applies handles in this order. More specific handles merge later and override earlier ones.

### 2.2 Adding Custom Containers to All CMS Pages

Override `cms_page_view.xml` in your theme to add site-wide CMS containers:

```xml
<!-- app/design/frontend/Vendor/theme/Magento_Cms/layout/cms_page_view.xml -->
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <container name="cms.banners.after" label="CMS Banners After"
                   as="cmsBannersAfter" after="main.content" htmlTag="div" htmlClass="cms-banners">
            <block class="Vendor\Module\Block\Cms\Banners" name="cms.banners.block"
                   template="Vendor_Module::cms/banners.phtml"/>
        </container>
    </body>
</page>
```

### 2.3 Per-Page Specific Overrides

For a specific CMS page, create a handle named after the page ID:

```xml
<!-- app/design/frontend/Vendor/theme/Magento_Cms/layout/cms_page_view_id_5.xml -->
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <container name="cms.page.sidebar" label="CMS Page Sidebar"
                   before="-" htmlTag="aside" htmlClass="cms-page-sidebar">
            <block class="Magento\Cms\Block\Block" name="cms.about.sidebar">
                <arguments>
                    <argument name="block_id" xsi:type="string">about-sidebar</argument>
                </arguments>
            </block>
        </container>
    </body>
</page>
```

### 2.4 Database-Stored Layout Updates

CMS pages support layout updates stored in the database via the `layout_update_xml` column. These are applied through `Magento\Framework\View\Model\Layout\Merge` and stored per-page in the admin panel under **Content → Pages → [Page] → Design → Layout Update XML**.

Database updates merge **after** all XML files, giving admin users layout override capability without code changes. Database updates have lower precedence than theme layout XML — theme XML always wins over database XML on conflicts.

### 2.5 Page Layout (1-Column, 2-Columns, etc.)

The `page_layout` property on a CMS page controls which root template is used:

| Page Layout | Root Template |
|------------|---------------|
| `1column` | `1column.phtml` |
| `2columns-left` | `2columns-left.phtml` |
| `2columns-right` | `2columns-right.phtml` |
| `3columns` | `3columns.phtml` |

Set in the admin panel under **Content → Pages → [Page] → Design → Layout**, or programmatically:

```php
$page->setPageLayout('2columns-left');
```

---

## 3. CMS Blocks

### 3.1 Rendering Blocks in Templates

CMS blocks are reusable pieces of content referenced by identifier. To render a block in any `.phtml` template:

```php
// In a .phtml template
$block = $block->getLayout()->createBlock('Magento\Cms\Block\Block')->setBlockId('footer-links');
echo $block->toHtml();

// Shorthand via AbstractBlock
echo $block->getLayout()->createBlock('Magento\Cms\Block\Block')
    ->setBlockId('footer-links')
    ->toHtml();
```

`setBlockId()` loads block content from `cms_block` by identifier.

### 3.2 Block Directive in CMS Content

Within CMS page or block content, use the `{{block}}` directive:

```
{{block class="Magento\Cms\Block\Block" name="footer-links" block_id="footer-links"}}
```

| Attribute | Description |
|-----------|-------------|
| `class` | Block class to instantiate |
| `name` | Unique block name in layout |
| `block_id` | Identifier from `cms_block` table (for `Magento\Cms\Block\Block`) |
| `template` | Optional template override |

Any additional attributes become block arguments:

```
{{block class="Vendor\Module\Block\Widget\Promo" name="promo-banner"
       template="Vendor_Module::widget/promo.phtml" discount="20"}}
```

### 3.3 Widget Directive in CMS Content

Widgets are more structured than block directives and support admin-configurable parameters:

```
{{widget type="Vendor\Module\Block\Widget\SaleSlider" template="widget/sale-slider.phtml" title="Flash Sales"}}
```

The `{{widget}}` directive is processed by `Magento\Widget\Model\Template\Filter`.

### 3.4 Creating a Custom Block Processor Plugin

`Magento\Cms\Model\BlockProcessor` processes `{{block}}` directives. You can extend it via a plugin to support custom directive syntax:

```php
// app/code/Vendor/Module/Plugin/Cms/BlockProcessorPlugin.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Cms;

class BlockProcessorPlugin
{
    public function beforeParse(\Magento\Cms\Model\BlockProcessor $subject, string $content): string
    {
        return str_replace(
            '{{custom_banner id="123"}}',
            '{{block class="Vendor\Module\Block\CustomBanner" name="custom-banner" banner_id="123"}}',
            $content
        );
    }
}
```

```xml
<!-- app/code/Vendor/Module/etc/di.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <type name="Magento\Cms\Model\BlockProcessor">
        <plugin name="Vendor_Module_BlockProcessor"
                type="Vendor\Module\Plugin\Cms\BlockProcessorPlugin"/>
    </type>
</config>
```

### 3.5 CMS Block Table Reference

```sql
-- Core table: cms_block
DESCRIBE cms_block;
-- Columns: block_id, title, identifier, content, creation_time, update_time

-- Linking table: cms_block_store
DESCRIBE cms_block_store;
-- Columns: block_id, store_id
```

---

## 4. CMS Page Builder

### 4.1 Overview

Page Builder is Magento's drag-and-drop CMS content editor. In Magento 2.3–2.4 it was an Enterprise Edition feature; starting with 2.4 it is bundled with Open Source as well. It replaces the raw WYSIWYG editor for CMS pages and blocks. Internally, Page Builder content is stored as JSON in the `pagebuilder_content` column.

### 4.2 Page Builder Content Structure

Page Builder content is a nested JSON tree:

```json
{
  "type": "container",
  "name": "Main Content Area",
  "children": [
    {
      "type": "row",
      "children": [
        {
          "type": "column",
          "children": [
            { "type": "heading", "data": { "heading": "Welcome", "tag": "h2" } },
            { "type": "text", "data": { "content": "<p>Our company was founded in 2024...</p>" } }
          ]
        }
      ]
    }
  ]
}
```

The `pagebuilder_content` column in `cms_page` and `cms_block` tables stores this JSON.

### 4.3 Page Builder Stage Architecture

Page Builder renders in two modes:

- **Admin Stage** — The drag-and-drop editing UI in the admin panel, powered by a KnockoutJS-based application that runs inside a `<div data-pagebuilder-stage="admin">`
- **Storefront Renderer** — When visiting the storefront, the JSON is parsed and rendered using Magento's layout XML block system

The stage-to-storefront conversion happens in `Magento\PageBuilder\Model\Stage\Adaptor`:

```php
// vendor/magento/module-pagebuilder/Model/Stage/Adaptor.php
$config = $this->configReader->read($pageId);
$layout = $this->layoutGenerator->generate($config['pagebuilder_content']);
```

### 4.4 Content Types

Page Builder ships with a hierarchy of built-in content types:

```
container (root)
├── row
│   └── column
│       ├── heading, text, banner, slider, video, button, divider
│       ├── products (with products carousel)
│       ├── map
│       └── html (raw HTML wrapper)
```

Each content type is declared in `etc/pagebuilder_content_types.xml`:

```xml
<!-- vendor/magento/module-pagebuilder/etc/pagebuilder_content_types.xml -->
<pagebuilder_content_types xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                            xsi:noNamespaceSchemaLocation="urn:magento:module:PageBuilder:etc/content_types.xsd">
    <type name="banner" label="Banner" icon="icon-banner">
        <move src="banner" dest="main"/>
        <remove src="banner-actions"/>
    </type>
</pagebuilder_content_types>
```

### 4.5 Custom Page Builder Content Type

Creating a custom content type requires four steps:

**Step 1 — Declare the content type:**

```xml
<!-- app/code/Vendor/Module/etc/pagebuilder_content_types.xml -->
<pagebuilder_content_types xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                            xsi:noNamespaceSchemaLocation="urn:magento:module:PageBuilder:etc/content_types.xsd">
    <type name="promo_ticker" label="Promo Ticker"
          icon="icon-promo" masterFormat="block">
        <element name="promo_message" label="Message"/>
        <element name="link" label="Link URL"/>
    </type>
</pagebuilder_content_types>
```

**Step 2 — Create the admin template (KnockoutJS):**

```html
<!-- app/code/Vendor/Module/view/adminhtml/web/template/pagebuilder/promo-ticker.html -->
<div class="promo-ticker" data-bind="attr: { id: generateId() }">
    <div class="promo-message" data-bind="text: displayData().promo_message || 'Enter message...'"/>
    <a class="promo-link" data-bind="attr: { href: displayData().link || '#' }">Shop Now</a>
</div>
```

**Step 3 — Master format / rendering config:**

```xml
<!-- app/code/Vendor/Module/etc/di.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <type name="Magento\PageBuilder\Model\Source\HtmlDocument">
        <arguments>
            <argument name="contentTypes" xsi:type="array">
                <item name="promo_ticker" xsi:type="string">Vendor_Module::content-type/promo-ticker</item>
            </argument>
        </arguments>
    </type>
</config>
```

**Step 4 — Create the storefront block:**

```php
// app/code/Vendor/Module/Block/ContentType/PromoTicker.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Block\ContentType;

class PromoTicker extends \Magento\Framework\View\Element\Template
{
    public function getPromoMessage(): string
    {
        $data = $this->getData('promo_ticker');
        return $data['promo_message'] ?? '';
    }

    public function getLink(): string
    {
        $data = $this->getData('promo_ticker');
        return $data['link'] ?? '#';
    }
}
```

---

## 5. Layout XML for CMS Pages

### 5.1 Adding JavaScript and CSS to CMS Pages

Use `cms_page_view.xml` (or a per-page handle) to add assets to CMS pages:

```xml
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <head>
        <css src="css/cms-special.css"/>
        <script src="https://cdn.example.com/analytics.js" srcType="url"/>
        <script src="Magento_Cms::js/cms-page.js"/>
    </head>
</page>
```

To add assets only on a specific CMS page, use the per-page handle:

```xml
<!-- app/design/frontend/Vendor/theme/Magento_Cms/layout/cms_page_view_id_10.xml -->
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <head>
        <css src="css/campaign-landing.css"/>
    </head>
</page>
```

### 5.2 Removing Blocks from CMS Pages

```xml
<referenceBlock name="breadcrumbs" remove="true"/>
<referenceBlock name="custom.theme.block" remove="true"/>
```

### 5.3 Conditional Block Rendering on CMS Pages

Custom conditional rendering is implemented by overriding `_toHtml()` in a block that checks the current CMS page from the registry:

```xml
<!-- In cms_page_view.xml -->
<referenceContainer name="cms.extra">
    <block class="Vendor\Module\Block\Cms\ConditionalContent" name="cms.conditional">
        <arguments>
            <argument name="page_ids" xsi:type="array">
                <item name="0" xsi:type="string">5</item>
                <item name="1" xsi:type="string">12</item>
            </argument>
        </arguments>
    </block>
</referenceContainer>
```

```php
// app/code/Vendor/Module/Block/Cms/ConditionalContent.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Block\Cms;

class ConditionalContent extends \Magento\Framework\View\Element\Template
{
    public function _toHtml()
    {
        $page = $this->_coreRegistry->registry('cms_page');
        $allowedPages = $this->getData('page_ids') ?? [];

        if ($page && in_array((string)$page->getId(), $allowedPages, true)) {
            return parent::_toHtml();
        }
        return '';
    }
}
```

### 5.4 Full-Page CMS with Custom Page Layout

To build a purely CMS-driven landing page:

```xml
<!-- app/design/frontend/Vendor/theme/Magento_Cms/layout/cms_page_view_id_15.xml -->
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd"
      layout="1column">
    <body>
        <container name="cms.landing.hero" label="Landing Hero" htmlTag="header" htmlClass="landing-hero">
            <block class="Magento\Cms\Block\Block" name="landing.hero.block">
                <arguments>
                    <argument name="block_id" xsi:type="string">landing-hero</argument>
                </arguments>
            </block>
        </container>
        <container name="cms.landing.features" label="Features Grid" htmlTag="section">
            <block class="Magento\Cms\Block\Block" name="landing.features.block">
                <argument name="block_id" xsi:type="string">landing-features</argument>
            </block>
        </container>
        <container name="cms.landing.cta" label="Call to Action" htmlTag="section">
            <block class="Vendor\Module\Block\Widget\CtaWidget" name="landing.cta.widget"
                   template="Vendor_Module::widget/cta.phtml"/>
        </container>
    </body>
</page>
```

---

## 6. CMS Widgets

### 6.1 How Widgets Work in CMS Content

Widgets are structured, admin-configurable content units inserted via the `{{widget}}` directive. Unlike a raw `{{block}}` directive, a widget supports:
- Admin-configurable parameters (the UI shown in the WYSIWYG editor)
- Parameter validation
- Automatic template selection based on widget type
- Built-in caching

The `{{widget}}` directive is processed by `Magento\Widget\Model\Template\Filter`.

### 6.2 Declaring a Widget (etc/widget.xml)

```xml
<!-- app/code/Vendor/Module/etc/widget.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<widgets xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="urn:magento:module:Vendor_Module:etc/widget.xsd">
    <widget id="sale_slider" class="Vendor\Module\Block\Widget\SaleSlider">
        <label translate="true">Sale Products Slider</label>
        <description translate="true">Displays products on sale as a horizontal slider</description>
        <parameters>
            <parameter name="title" xsi:type="text" required="true" visible="true">
                <label translate="true">Title</label>
            </parameter>
            <parameter name="display_mode" xsi:type="select" required="true" visible="true"
                       source_model="Vendor\Module\Model\Config\Source\DisplayMode">
                <label translate="true">Display Mode</label>
            </parameter>
            <parameter name="product_count" xsi:type="text" required="false" visible="true">
                <label translate="true">Number of Products</label>
                <value>8</value>
            </parameter>
            <parameter name="cache_lifetime" xsi:type="text" visible="false">
                <value>86400</value>
            </parameter>
        </parameters>
    </widget>
</widgets>
```

### 6.3 Implementing the Widget Block

```php
// app/code/Vendor/Module/Block/Widget/SaleSlider.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Block\Widget;

class SaleSlider extends \Magento\MagentoWork\View\Element\Template
    implements \Magento\Widget\Block\BlockInterface
{
    protected $_template = 'Vendor_Module::widget/sale-slider.phtml';

    public function getTitle(): string
    {
        return (string) $this->getData('title');
    }

    public function getDisplayMode(): string
    {
        return (string) $this->getData('display_mode');
    }

    public function getProductCount(): int
    {
        return (int) $this->getData('product_count') ?: 8;
    }

    public function getProducts(): array
    {
        if (!$this->hasData('products')) {
            $this->setData('products', $this->productRepository->getList(...)->getItems());
        }
        return $this->getData('products');
    }
}
```

### 6.4 Using a Widget in CMS Content

In the CMS content editor (admin WYSIWYG), use the **Insert Widget** button, or type directly:

```
{{widget type="Vendor\Module\Block\Widget\SaleSlider"
         title="Today's Deals"
         display_mode="sale_first"
         product_count="12"}}
```

The `{{widget}}` directive renders using the block's template (`Vendor_Module::widget/sale-slider.phtml`).

---

## 7. Extension Attributes on CMS Pages

### 7.1 Overview

Extension attributes allow you to attach custom data to CMS pages without modifying the core `cms_page` table. This is the Magento-supported approach for adding custom fields that may be used by third-party modules or stored in external systems.

### 7.2 Declaring Extension Attributes

Extension attributes are declared in `etc/extension_attributes.xml`:

```xml
<!-- app/code/Vendor/Module/etc/extension_attributes.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<extension_attributes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                     xsi:noNamespaceSchemaLocation="urn:magento:framework:Extension/etc/extension_attributes.xsd">
    <attribute code="featured_category" type="Vendor\Module\Api\Data\CmsPageFeatureInterface"/>
    <attribute code="campaign_id" type="string"/>
</extension_attributes>
```

### 7.3 Implementing the Extension Interface

```php
// app/code/Vendor/Module/Api/Data/CmsPageFeatureInterface.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Api\Data;

interface CmsPageFeatureInterface
{
    public function getFeaturedCategoryId(): ?int;
    public function setFeaturedCategoryId(?int $id): self;
}
```

### 7.4 Plugin on CMS Page Repository

To handle the custom attribute persistence, create a plugin on `Magento\Cms\Api\PageRepositoryInterface`:

```php
// app/code/Vendor/Module/Plugin/Cms/PageRepositoryPlugin.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Cms;

class PageRepositoryPlugin
{
    public function afterSave(
        \Magento\Cms\Api\PageRepositoryInterface $subject,
        \Magento\Cms\Api\Data\PageInterface $page
    ): \Magento\Cms\Api\Data\PageInterface {
        $extensionAttributes = $page->getExtensionAttributes();
        if ($extensionAttributes && $extensionAttributes->getFeaturedCategoryId()) {
            $this->saveFeaturedCategory($page->getId(), $extensionAttributes->getFeaturedCategoryId());
        }
        return $page;
    }

    public function afterGetById(
        \Magento\Cms\Api\PageRepositoryInterface $subject,
        \Magento\Cms\Api\Data\PageInterface $page
    ): \Magento\Cms\Api\Data\PageInterface {
        $featuredCategoryId = $this->loadFeaturedCategory($page->getId());
        $extensionAttributes = $page->getExtensionAttributes() ?? $page->getExtensionAttributes();
        $extensionAttributes->setFeaturedCategoryId($featuredCategoryId);
        $page->setExtensionAttributes($extensionAttributes);
        return $page;
    }
}
```

```xml
<!-- app/code/Vendor/Module/etc/di.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <type name="Magento\Cms\Api\PageRepositoryInterface">
        <plugin name="Vendor_Module_PageRepository"
                type="Vendor\Module\Plugin\Cms\PageRepositoryPlugin"/>
    </type>
</config>
```

### 7.5 Using Extension Attributes in Templates

```php
// In a .phtml template or Block class
$page = $block->getPage();
$extensionAttributes = $page->getExtensionAttributes();
if ($extensionAttributes && $extensionAttributes->getFeaturedCategoryId()) {
    $categoryId = $extensionAttributes->getFeaturedCategoryId();
    // Use the custom data
}
```

---

## Summary

| Concept | Key File/Class |
|---------|---------------|
| CMS Page Controller | `Magento\Cms\Controller\Index\Index` |
| Generic CMS layout handle | `cms_page_view.xml` |
| Homepage layout handle | `cms_index_index.xml` |
| Per-page layout handle | `cms_page_view_id_{id}.xml` |
| CMS block directive | `{{block class="..." block_id="..."}}` |
| CMS widget directive | `{{widget type="..."}}` |
| Block processor | `Magento\Cms\Model\BlockProcessor` |
| Widget filter | `Magento\Widget\Model\Template\Filter` |
| Page Builder JSON column | `pagebuilder_content` |
| Widget declaration | `etc/widget.xml` |
| Extension attributes | `etc/extension_attributes.xml` |