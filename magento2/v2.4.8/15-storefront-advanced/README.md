---
title: "15 - Advanced Storefront Customization"
description: "Magento 2.4.8 advanced frontend: checkout JS customization, payment method integration, jQuery widget patterns, third-party theme integration"
rank: 15
pathways: [magento2-deep-dive]
see_also:
  - "Checkout deep-dive: ../10-checkout/README.md"
  - "JavaScript foundations: ../12-storefront-js/README.md"
---

# 15 — Advanced Storefront Customization

> **Prerequisites**: Checkout flow ([10-checkout](../10-checkout/README.md)), JavaScript foundations ([12-storefront-js](./README.md)), Theming ([14-storefront-theming](./README.md))

---

## Learning Objectives

By the end of this chapter you will be able to:
- Customize checkout JavaScript components via plugins and mixins
- Integrate new payment methods into the Magento checkout flow
- Apply jQuery widget extension patterns
- Install and configure third-party themes (Luma child, Hyvä)
- Optimize frontend performance with deferred loading and critical CSS

---

## 1. Checkout JS Architecture

The Magento 2 checkout is a KnockoutJS application with UI Components.

### Directory Structure

```
Magento_Checkout/
├── view/frontend/
│   ├── web/
│   │   ├── js/
│   │   │   ├── model/         # KnockoutJS view models (non-UI)
│   │   │   ├── view/          # KnockoutJS UI Components
│   │   │   ├── template/      # HTML templates
│   │   │   └── action/
│   │   └── template/          # KnockoutJS templates (legacy)
│   └── layout/
│       └── checkout_index_index.xml  # Checkout layout config
```

### Key Checkout JS Models

| Model | Path | Purpose |
|-------|------|---------|
| `quote` | `Magento_Checkout/js/model/quote` | Active quote data |
| `shipping-address/formatter` | `Magento_Checkout/js/model/shipping-address/formatter` | Address formatting |
| `billing-address/formatter` | `Magento_Checkout/js/model/billing-address/formatter` | Billing format |
| `customer-addresses` | `Magento_Checkout/js/model/customer-addresses` | Saved addresses |
| `totals` | `Magento_Checkout/js/model/totals` | Cart totals |
| `checkout-data-resolver` | `Magento_Checkout/js/model/checkout-data-resolver` | Data sync |

### KnockoutJS Template Bindings in Checkout

```html
<!-- Checkout sidebar: Magento_Checkout/view/frontend/web/template/summary/sidebar.html -->
<div class="opc-sidebar" data-bind="visible: isVisible()">
    <each args="items: getItems()">
        <div class="item">
            <span data-bind="text: name"></span>
            <span data-bind="text: price"></span>
        </div>
    </each>
    <div class="grand totals" data-bind="visible: $parent.isFull()">
        <span data-bind="text: $parent.grandTotal"></span>
    </div>
</div>
```

---

## 2. Customizing Checkout Components

### Using Plugins (Interceptors)

Checkout JS models support standard Magento DI plugins:

```javascript
// app/code/MyVendor/CheckoutPlugin/etc/di.xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento_Checkout/js/model/quote">
        <plugin name="MyVendor_CheckoutPlugin_quote"
                type="MyVendor\CheckoutPlugin\Plugin\QuotePlugin"/>
    </type>
</config>
```

```php
// app/code/MyVendor/CheckoutPlugin/Plugin/QuotePlugin.php
namespace MyVendor\CheckoutPlugin\Plugin;

class QuotePlugin
{
    public function beforeSetShippingAddress($subject, $address)
    {
        // Add validation or modification
        return [$address];
    }

    public function afterGetShippingAddress($subject, $result)
    {
        // Modify or enrich the returned address
        return $result;
    }
}
```

### Using Mixins on Checkout JS

```javascript
// app/code/MyVendor/Checkout/view/frontend/web/js/mixin/shipping-address-mixin.js
define(['Magento_Checkout/js/model/quote'], function (quote) {
    'use strict';

    return function (target) {
        var originalSave = target.saveAddress;

        target.saveAddress = function () {
            // Custom pre-save logic
            console.log('Custom shipping save triggered');
            return originalSave.call(this);
        };

        return target;
    };
});
```

### Adding a Custom Checkout Step

Layout XML approach:

```xml
<!-- app/code/MyVendor/Checkout/view/frontend/layout/checkout_index_index.xml -->
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceBlock name="checkout">
            <arguments>
                <argument name="jsLayout" xsi:type="array">
                    <item name="components" xsi:type="array">
                        <item name="checkout" xsi:type="array">
                            <item name="children" xsi:type="array">
                                <item name="my-custom-step" xsi:type="array">
                                    <item name="component"
                                          xsi:type="string">MyVendor_Checkout/js/view/my-custom-step</item>
                                    <item name="sortOrder"
                                          xsi:type="string">20</item>
                                    <item name="displayArea"
                                          xsi:type="string">steps</item>
                                </item>
                            </item>
                        </item>
                    </item>
                </argument>
            </arguments>
        </referenceBlock>
    </body>
</page>
```

---

## 3. Payment Method Integration

### Payment Method Component Structure

```
Magento_Payment/
├── view/frontend/
│   ├── web/
│   │   └── js/
│   │       └── view/
│   │           └── payment/
│   │               └── method-renderer/
│   │                   └── my-payment-method.js
│   └── template/
│       └── payment/
│           └── my-payment-method.html
```

### Payment View Model

```javascript
// app/code/MyVendor/Payment/view/frontend/web/js/view/payment/method-renderer/my-payment-method.js
define([
    'Magento_Payment/js/view/payment/payment-method',
    'jquery'
], function (Component, $) {
    'use strict';

    return Component.extend({
        defaults: {
            template: 'MyVendor_Payment/payment/my-payment-method',
            code: 'my_payment_method'
        },

        getCode: function () {
            return this.code;
        },

        isActive: function () {
            return true;
        },

        getData: function () {
            return {
                method: this.getCode(),
                'additional_data': {
                    'token': this.getToken()
                }
            };
        },

        getToken: function () {
            return $('#my-payment-token').val();
        }
    });
});
```

### Payment Template

```html
<!-- app/code/MyVendor/Payment/view/frontend/web/template/payment/my-payment-method.html -->
<div class="payment-method" data-bind="css: {'_active': isActive()}">
    <div class="payment-method-title field choice">
        <input type="radio"
               name="payment[method]"
               class="radio"
               data-bind="attr: {'id': getCode()}, value: getCode(), checked: isChecked"/>
        <label data-bind="attr: {'for': getCode()}" class="label">
            <span data-bind="text: getTitle()"></span>
        </label>
    </div>
    <div class="payment-method-content">
        <div class="field number">
            <label class="label">Card Number</label>
            <input type="text" id="my-payment-token" class="input-text"/>
        </div>
        <div class="actions-toolbar">
            <button class="action primary">Place Order</button>
        </div>
    </div>
</div>
```

### Adapter Pattern for Payment Gateway

```javascript
// app/code/MyVendor/Payment/view/frontend/web/js/model/adapter.js
define(['jquery', 'Magento_Payment/js/model/credit-card-validation/credit-card-data-validator'], function ($, validator) {
    'use strict';

    return {
        authorize: function (paymentData, successCallback, errorCallback) {
            $.ajax({
                url: '/my-payment/authorize',
                type: 'POST',
                data: {
                    payment: paymentData,
                    token: paymentData.token
                },
                success: function (response) {
                    if (response.success) {
                        successCallback(response);
                    } else {
                        errorCallback(response.message);
                    }
                },
                error: function () {
                    errorCallback('Network error');
                }
            });
        }
    };
});
```

---

## 4. Third-Party Theme Integration

### Installing a Theme via Composer

```bash
# Find the theme package
composer require myvendor/my-theme

# After install, register the theme
bin/magento setup:upgrade

# Deploy static content
bin/magento setup:static-content:deploy -f
```

### Creating a Luma Child Theme

```xml
<!-- app/design/frontend/MyVendor/ChildTheme/theme.xml -->
<theme xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:noNamespaceSchemaLocation="urn:magento:framework:Config/etc/theme.xsd">
    <title>My Child Theme</title>
    <parent>Magento/luma</parent>
    <media>
        <preview_image>media/preview.jpg</preview_image>
    </media>
</theme>
```

### Hyvä Compatibility

Hyvä is a modern Magento theme using Tailwind CSS and AlpineJS instead of LESS/jQuery.

**Key differences:**
- No RequireJS — uses vanilla ES modules
- No KnockoutJS — uses AlpineJS for reactivity
- No LESS — uses Tailwind CSS
- `checkout/` layout uses Hyvä's `checkout/` instead of Magento's

**Installation:**
```bash
composer require hyvä/theme
bin/magento setup:upgrade
bin/magento setup:static-content:deploy -f
```

---

## 5. Performance — Frontend

### Deferring JavaScript

Magento's default approach loads scripts in the `<head>` and before `</body>`. For performance:

```xml
<!-- In your module's layout XML, move JS to deferred -->
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceScript name="Magento_Catalog/js/product-gallery"/>
        <script src="MyVendor_MyModule/js/heavy-component.js" defer="defer"/>
    </body>
</page>
```

### Image Lazy Loading

Magento supports native lazy loading:

```html
<img data-bind="attr: {src: image.url, alt: image.label}"
     loading="lazy"
     class="product-image-photo"/>
```

### Critical CSS

For production, inline critical CSS to avoid render-blocking:

```bash
# Deploy critical CSS inline
bin/magento setup:static-content:deploy -f \
    --area=frontend \
    --theme=MyVendor/MyTheme \
    --no-html-minify
```

### Bundling and Minification

In `app/etc/env.php`:

```php
'css' => [
    'minifier' => 'css',
    'minify' => '1'
],
'system' => [
    'default' => [
        'dev' => [
            'js' => [
                'minify' => '1',
                'enable_sign' => '0'
            ]
        ]
    ]
]
```

---

## 6. jQuery Widget Patterns

### Extending Existing Widgets

```javascript
// Extend Magento's collapsible widget
define(['jquery', 'jquery/ui', 'Magento_Catalog/js/product/collapsible'], function ($) {
    'use strict';

    // Target the existing widget
    $.widget('myVendorProductCollapsible', $.Magento_CatalogProductCollapsible, {
        options: {
            expanded: true,
            showIcon: true
        },
        _create: function () {
            this._super();
            this.element.on('beforeOpen', this._onBeforeOpen.bind(this));
        },
        _onBeforeOpen: function () {
            console.log('Custom before-open handler');
        }
    });

    return $.myVendorProductCollapsible;
});
```

### Creating a Custom Widget Used in Storefront

```javascript
// app/code/MyVendor/MyModule/view/frontend/web/js/widgets/product-compare.js
define(['jquery', 'jquery/ui'], function ($) {
    'use strict';

    $.widget('myVendorProductCompare', {
        options: {
            maxItems: 4,
            baseUrl: '/catalog/product/compare',
            addUrl: '/catalog/product/compare/add'
        },
        _create: function () {
            this.element.on('click', '[data-compare-add]', this._onAddClick.bind(this));
            this._updateCounter();
        },
        _onAddClick: function (e) {
            var $btn = $(e.currentTarget);
            var productId = $btn.data('compare-product-id');

            $.ajax({
                url: this.options.addUrl,
                data: { product_id: productId },
                success: function () {
                    this._trigger('compareAdded', null, { productId: productId });
                    this._updateCounter();
                }.bind(this)
            });
        },
        _updateCounter: function () {
            var count = this.element.find('.compare-item').length;
            this.element.find('.compare-count').text(count);
        },
        getCompareUrl: function () {
            return this.options.baseUrl;
        }
    });

    return $.myVendorProductCompare;
});
```

---

## 7. Cross-Origin Resource Loading

### CDN Configuration

```xml
<!-- In your theme's default.xml -->
<referenceBlock name="head.globals">
    <block name="cdn.config" as="cdn_config" after="-">
        <arguments>
            <argument name="cdn_base_url" xsi:type="string">https://cdn.example.com</argument>
        </arguments>
    </block>
</referenceBlock>
```

### External Script Handling

```javascript
// In your theme's requirejs-config.js
var config = {
    paths: {
        // Map external libraries
        'stripe-js': 'https://js.stripe.com/v3/'
    },
    config: {
        mixins: {
            'Magento_Payment/js/view/payment/method-renderer': {
                'MyVendor_Payment/js/mixin/stripe-mixin': true
            }
        }
    }
};
```

### Data-attribute Based External Loading

```html
<!-- In your .phtml template -->
<script type="text/x-magento-init">
{
    "*": {
        "MyVendor_MyModule/js/external-init": {
            "scriptUrl": "https://cdn.example.com/tracker.js",
            "async": true,
            "attributes": {
                "data-api-key": "pk_live_xxxx"
            }
        }
    }
}
</script>
```

---

## CLI Reference

```bash
# Upgrade after installing theme
bin/magento setup:upgrade

# Deploy static assets
bin/magento setup:static-content:deploy -f --area=frontend

# Clear cache after JS/layout/theme changes
bin/magento cache:clean full_page layout

# Check deployed theme
bin/magento info:theme:installed
```

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `Magento_Checkout/js/model/*.js` | Checkout JS view models |
| `Magento_Checkout/js/view/*.js` | Checkout UI Components |
| `view/frontend/template/payment/*.html` | Payment method templates |
| `web/js/model/adapter.js` | Payment gateway adapter |
| `etc/di.xml` (plugin) | Checkout model plugins |
| `composer.json` (theme) | Third-party theme dependency |

---

## What's Next

- [Full theme reference: _supplemental/23-theme-development.md](../_supplemental/23-theme-development.md)
- [API reference: _supplemental/11-graphql-rest-api.md](../_supplemental/11-graphql-rest-api.md)

---

*[Content coming soon — in-depth chapter prose, examples, and exercises]*