---
title: "13 - Frontend Layout XML & Templates"
description: "Magento 2.4.8 frontend layout: layout handles, templates, frontend ACL, page types, customizing via theme"
rank: 13
pathways: [magento2-deep-dive]
see_also:
  - "Full theme reference: _supplemental/23-theme-development.md"
---

# 13 — Frontend Layout XML & Templates

> **Prerequisites**: Layout XML basics ([06-admin-ui](../06-admin-ui/README.md)), block/template layer ([03-module-development](../03-module-development/README.md))

---

## Learning Objectives

By the end of this chapter you will be able to:
- Explain Magento's frontend layout handle system and page hierarchy
- Customize PHTML templates via theme override
- Use `page_container` containers for structural layout control
- Configure frontend ACL to restrict content by customer group
- Enable template hints for frontend debugging

---

## 1. Frontend Layout Handle Hierarchy

Magento's frontend uses a **layout handles** system — XML files in `view/frontend/layout/` that target specific page contexts.

### Handle File Naming Convention

```
<module>_<controller>_<action>.xml
```

| Layout Handle | Triggered On |
|--------------|-------------|
| `default.xml` | Every frontend page |
| `customer_account_index.xml` | Customer account dashboard |
| `catalog_product_view.xml` | Product detail page |
| `checkout_cart_index.xml` | Shopping cart |
| `checkout_index_index.xml` | Checkout page |
| `cms_index_index.xml` | CMS home page |

### Layout Merge Order

Handles are merged in this priority (later overrides earlier):

1. `default.xml` — applied to all pages
2. `<module>` namespace handles — e.g., `Magento_Catalog::default.xml`
3. `<area>/default.xml` — e.g., `frontend/default.xml`
4. **Theme** overrides (in theme's `Magento_Theme/layout/`)
5. Page-specific handles

### Handle Override via Theme

To customize a page layout, place your override in your theme:

```
app/design/frontend/<Vendor>/<Theme>/Magento_Catalog/layout/catalog_product_view.xml
```

---

## 2. Template Customization

### Block Class Hierarchy on Frontend

Frontend blocks extend from:

```
Magento\Framework\View\Element\Template
    └── Magento\Framework\View\Element\Html\Link
            └── Magento\Framework\View\Element\Link
```

Most frontend content blocks use `Magento\Framework\View\Element\Template` as base.

### PHTML Template Usage

```php
<?php
/** @var $block \Magento\Framework\View\Element\Template */
/** @var \Magento\Framework\Escaper $escaper */

// Always escape output
echo $escaper->escapeHtml($block->getProductName());

// url builder
echo $block->getUrl('catalog/product/view', ['id' => 123]);

// template-specific methods
$product = $block->getProduct();
?>
<div class="product-view">
    <h1><?= $escaper->escapeHtml($product->getName()) ?></h1>
    <price><?= $block->getProductPrice() ?></price>
</div>
```

### Overriding Templates via Theme

1. Copy the original template from:
   `app/code/Magento/Catalog/view/frontend/templates/product/view.phtml`
2. Place it in your theme at:
   `app/design/frontend/<Vendor>/<Theme>/Magento_Catalog/templates/product/view.phtml`

The theme file takes precedence over the module file.

---

## 3. Frontend ACL

Access Control List on the frontend restricts content visibility by customer group or logged-in state.

### Layout XML ACL Reference

```xml
<!-- Restrict a block to logged-in customers only -->
<referenceBlock name="review-list" remove="true">
    <arguments>
        <argument name="acl" xsi:type="string">Magento_Customer::account</argument>
    </arguments>
</referenceBlock>

<!-- Alternative: using conditions -->
<referenceBlock name="price-final" ifconfig="catalog/product/tier_prices">
</referenceBlock>
```

### Condition-Based Display

```xml
<!-- Show only to guest users -->
<referenceBlock name="newsletter-subscription" displayCondition="not logged_in" />

<!-- Show only to specific customer group -->
<referenceBlock name="b2b-pricing" displayCondition="customer_group_4" />
```

---

## 4. Container and Block Reference

### Magento Page Structure

A typical Magento Luma page structure:

```
page_container (root container)
├── header.container
│   ├── header.panel.wrapper
│   │   └── header.panel
│   │       └── logo, navigation, search, cart
│   └── skip to content
├── main.container
│   └── columns wrapper (1 column / 2 columns / 3 columns)
│       └── content (main column)
│           └── page.main.title, catalog.leftnav, etc.
└── footer.container
    └── footer, copyright
```

### Key System Containers

| Container | Purpose |
|-----------|---------|
| `page.container` | Root container for all pages |
| `header.container` | Site header wrapper |
| `content` | Main column content |
| `footer` | Footer content |
| `sidebar.main` | Left navigation (catalog) |
| `sidebar.additional` | Right sidebar |
| `columns.top` | Above main content |
| `columns.bottom` | Below main content |

### Adding Blocks to Containers via Layout XML

```xml
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceContainer name="header.panel">
            <block name="my.custom.link"
                   class="Magento\Framework\View\Element\Html\Link"
                   template="MyVendor_MyModule::custom-link.phtml">
                <arguments>
                    <argument name="label" xsi:type="string">Support</argument>
                    <argument name="path" xsi:type="string">support</argument>
                </arguments>
            </block>
        </referenceContainer>
    </body>
</page>
```

---

## 5. Template Hints

### Enabling Template Path Hints (CLI)

```bash
# Enable for a specific store
bin/magento dev:template-hints:enable

# Disable
bin/magento dev:template-hints:disable

# Enable with block hints (more detail, can impact performance)
bin/magento dev:template-hints:enable --with-block-hints
```

### Alternative: Admin Configuration

**Stores → Configuration → Advanced → Developer → Debug → Template Path Hints**

Set to "Yes" and scope to your store view.

> **Warning**: Never enable template hints in production. Performance impact is significant.

---

## 6. Custom Page Layouts

### Creating a Custom Handle

Create `app/code/<Vendor>/<Module>/view/frontend/layout/<my_handle>.xml` and reference it from a controller or CMS page.

### Declaring Custom Layout from CMS Page

In a CMS page's **Layout XML Update** field:

```xml
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <update handle="my_custom_handle"/>
    <body>
        <referenceBlock name="page.main.title" remove="true"/>
    </body>
</page>
```

### Using page_layout XML

For full custom page layouts, use `page_layout/*.xml` files:

```xml
<!-- app/code/<Vendor>/<Module>/view/frontend/page_layout/my-custom-layout.xml -->
<layout xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_layout.xsd">
    <label>My Custom Layout</label>
    <referenceContainer name="main.content">
        <container htmlClass="custom-layout" htmlTag="div">
            <block name="custom.block" template="MyVendor_MyModule::custom.phtml"/>
        </container>
    </referenceContainer>
</layout>
```

---

## CLI Reference

```bash
# Flush layout cache after changing XML
bin/magento cache:clean layout

# Enable developer mode for layout XML changes to take effect without cache clear
bin/magento deploy:mode:set developer

# Recompile layout (static content deployment)
bin/magento setup:static-content:deploy -f --area frontend
```

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `view/frontend/layout/*.xml` | Frontend layout handles |
| `view/frontend/templates/*.phtml` | Frontend PHTML templates |
| `view/frontend/page_layout/*.xml` | Custom page layout definitions |
| `etc/view.xml` | Theme image config |
| `Magento_Theme/layout/default.xml` | Global theme layout |

---

## What's Next

- [14 — Storefront Theming & LESS](./README.md) — LESS compilation, theme hierarchy
- [12 — Storefront JavaScript Foundations](./README.md) — JS initialization and KnockoutJS
- [Full theme reference: _supplemental/23-theme-development.md](../_supplemental/23-theme-development.md)

---

*[Content coming soon — in-depth chapter prose, examples, and exercises]*