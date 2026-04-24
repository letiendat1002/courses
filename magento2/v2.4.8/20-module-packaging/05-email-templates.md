---
title: "05 - Email Templates"
description: "Magento 2.4.8 transactional email: template architecture, HTML email development, CSS inlining, template variables, admin configuration, custom email rendering"
tags: [magento2, email, transactional-email, template, css-inlining, mjml]
rank: 5
pathways: [magento2-deep-dive]
see_also:
  - "Module packaging: ./09-packaging-distribution.md"
---

# 05 — Email Templates

Transactional email in Magento 2 is powered by the `Magento\Email` module. Every order confirmation, shipment notification, and password reset flows through a unified template pipeline: DB-stored or file-based template → variable resolution → CSS inlining → MIME encoding → queue → delivery.

---

## 1. Email Template Architecture

### `email_template` DB Table

Templates stored in the database land in `email_template`:

```sql
SELECT template_id, template_code, template_subject, template_text,
       template_html, template_styles, is_active, added_at, modified_at
FROM email_template;
```

Key columns:
- `template_text` — plain-text version
- `template_html` — HTML version (used when available)
- `template_styles` — raw CSS stored separately for inlining via Emogrifier
- `is_active` — controls whether the template is selectable in admin

### Core Classes

| Class | Role |
|---|---|
| `Magento\Email\Model\Template` | Main template entity — loads from DB or file, applies filters |
| `Magento\Email\Model\Template\Filter` | Processes `{{var}}`, `{{trans}}`, `{{depend}}`, `{{if}}` directives |
| `Magento\Email\Model\Template\FilterFactory` | Factory to obtain a configured `TemplateFilter` instance |
| `Magento\Email\Model\Template\Css\Processor` | Pre-processes styles before Emogrifier inlines them |
| `Magento\Email\Model\Mail\Transport` | Wraps the email MIME message, handles queue vs. direct send |
| `Magento\Email\Model\Mail\TransportBuilder` | Fluent builder for MIME messages with attachments |

### TemplateFilter Pattern

```php
use Magento\Email\Model\Template\FilterFactory;

class MySender
{
    public function __construct(
        private readonly FilterFactory $filterFactory,
    ) {}

    public function send(): void
    {
        $filter = $this->filterFactory->create();
        $filter->setTemplateParams([
            'order'   => $order,
            'customer' => $customer,
        ]);

        $subject  = $filter->processTemplate($template->getTemplateSubject());
        $bodyHtml = $filter->processTemplate($template->getTemplateHtml());
    }
}
```

The `TemplateFilter` resolves variables via `Variable processor` → `Escaper` → directives (`trans`, `depend`, `if`).

---

## 2. Template File Resolution

### File-based Template Path

File-based templates are resolved through the `view_preprocessed` area:

```
<Vendor>/<Module>/view/<area>/email/...
  └── html/
      └── customer_create_account_email.html
  └── template/
      └── html/
          └── customer_create_account_email.html
```

Area is either `frontend` or `adminhtml`. Resolution uses `Module\View\Url\PathResolver`.

### `email_templates.xml`

The canonical registration for **file-based** templates that can be overridden in the DB lives in `etc/<area>/email_templates.xml`:

```xml
<?xml version="1.0"?>
<email_templates xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                 xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Email:etc/email_templates.xsd">
    <template id="customer_create_account_email"
              label="New Account Confirmation"
              file="customer_create_account_email.html"
              type="html"
              module="Vendor_Module"
              area="frontend"/>
</email_templates>
```

Attributes:
- `id` — unique identifier used in admin and `getTemplateModel()` lookups
- `file` — filename relative to `email/` subdirectory of the module's view area
- `type` — `html` or `text`
- `module` — owning module (used for ACL and translation purposes)
- `area` — `frontend` or `adminhtml`

### Complete Example: `Vendor_Module/etc/frontend/email_templates.xml`

```xml
<?xml version="1.0"?>
<email_templates xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                 xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Email:etc/email_templates.xsd">
    <!-- Customer notification: order shipped -->
    <template id="vendor_module_order_shipped_email"
              label="Order Shipped Notification"
              file="order_shipped.html"
              type="html"
              module="Vendor_Module"
              area="frontend"/>

    <!-- Admin notification: low stock alert -->
    <template id="vendor_module_low_stock_alert"
              label="Low Stock Alert"
              file="low_stock_alert.html"
              type="html"
              module="Vendor_Module"
              area="adminhtml"/>

    <!-- Plain-text fallback -->
    <template id="vendor_module_order_shipped_email_text"
              label="Order Shipped Notification (Text)"
              file="order_shipped.txt"
              type="text"
              module="Vendor_Module"
              area="frontend"/>
</email_templates>
```

Corresponding template files:
```
<Vendor>/<Module>/view/frontend/email/order_shipped.html
<Vendor>/<Module>/view/frontend/email/order_shipped.txt
<Vendor>/<Module>/view/adminhtml/email/low_stock_alert.html
```

---

## 3. Email Variables & Directives

All directives are processed by `Magento\Email\Model\Template\Filter`.

### Variable Resolution

```html
<!-- Base variable -->
{{var order.getIncrementId()}}

<!-- Nested property chain -->
{{var order.getShippingAddress().format('html')}}

<!-- Inline scriptlet (no output escaping) -->
{{var helper->formatPrice($item.getPrice())}}

<!-- Template area code for translation -->
{{var store.getFrontendName()}}
```

The `{{var}}` directive applies HTML escaping by default. Use `{{var raw variable_name}}` to suppress escaping.

### Directives

| Directive | Example | Description |
|---|---|---|
| `{{trans}}` | `{{trans "Welcome, %name" name=$customer.getName()}}` | Marks string for translation |
| `{{depend}}` | `{{depend order.getIsVirtual()}}...{{/depend}}` | Conditionally render block if value is truthy |
| `{{if}}` | `{{if order.getStatus() === 'complete'}}...{{/if}}` | Full if/elseif/else logic |
| `{{var}}` | `{{var order.getIncrementId()}}` | Output escaped variable |
| `{{var raw}}` | `{{var raw order.getHtmlNote()}}` | Output unescaped (safe HTML) |
| `{{layout}}` | `{{layout handle="vendor_module_email_footer"}}` | Render a layout handle into the email |

### `{{if}}` / `{{depend}}` / `{{/if}}` / `{{/depend}}`

```html
{{if order.getIsVirtual()}}
  This order has no shipping address.
{{else}}
  Shipping to: {{var order.getShippingAddress().format('html')}}
{{/if}}

{{depend customer.getAttribute('newsletter_subscribed')}}
  You are subscribed to our newsletter.
{{/depend}}
```

---

## 4. HTML Email Development

### Constraints

- **Tables for layout** — no Flexbox, CSS Grid, or floats; use `<table role="presentation">` for structure
- **Inline all CSS** — Gmail and many corporate clients strip `<style>` tags from `<head>`
- **Max width ~600px** — for email client viewport constraints
- **Images require explicit `width`/`height`** — many clients block images by default
- **System fonts only** — Arial, Georgia, Times New Roman; no web fonts

### Basic Responsive Template

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{var order.getIncrementId()}}</title>
</head>
<body style="margin: 0; padding: 0; background-color: #f4f4f4; font-family: Arial, Helvetica, sans-serif;">
  <table role="presentation" width="100%" cellspacing="0" cellpadding="0" border="0">
    <tr>
      <td align="center" style="padding: 40px 0;">
        <!-- Email container -->
        <table role="presentation" width="600" cellspacing="0" cellpadding="0" border="0"
               style="max-width: 600px; background-color: #ffffff;">
          <!-- Header -->
          <tr>
            <td style="background-color: #333333; padding: 20px; text-align: center;">
              <img src="{{var store.getLogoUrl()}}" alt="{{var store.getFrontendName()}}"
                   width="150" height="auto" style="display: block;">
            </td>
          </tr>
          <!-- Body -->
          <tr>
            <td style="padding: 30px 40px;">
              <h1 style="margin: 0 0 20px; font-size: 22px; color: #333333;">
                {{trans "Hello, %name" name=$customer.getName()}}
              </h1>
              <p style="margin: 0 0 15px; font-size: 14px; line-height: 1.6; color: #555555;">
                {{trans "Your order %increment_id has been shipped." increment_id=$order.getIncrementId()}}
              </p>
              <!-- Order items table -->
              <table role="presentation" width="100%" cellspacing="0" cellpadding="0" border="0"
                     style="margin-top: 20px;">
                {{layout handle="vendor_module_email_order_items" order=$order}}
              </table>
            </td>
          </tr>
          <!-- Footer -->
          <tr>
            <td style="background-color: #f4f4f4; padding: 20px 40px; text-align: center;">
              <p style="margin: 0; font-size: 12px; color: #888888;">
                {{trans "Thank you for shopping with %store_name" store_name=$store.getFrontendName()}}
              </p>
            </td>
          </tr>
        </table>
      </td>
    </tr>
  </table>
</body>
</html>
```

### MJML Workflow

For teams that prefer modern CSS-in-JS authoring, MJML → HTML conversion with CSS extraction is a common workflow:

1. **Write** `.mjml` file using MJML syntax
2. **Compile** to `.html` using `mjml` CLI
3. **Extract CSS** from the `<style>` tag into a separate file
4. **Register** the template via `email_templates.xml`
5. **Store CSS** in `template_styles` column (picked up by Emogrifier)

MJML's `mjml-section` and `mjml-column` map naturally to email tables.

---

## 5. CSS Inlining

### Emogrifier

Magento bundles `pelago/emogrifier` (the library formerly maintained by Magento, now community-driven). After `TemplateFilter` processes variables and directives, `Emogrifier` inlines the stored `template_styles` into the rendered HTML.

Pipeline:
```
Template.getTemplateHtml()
  → TemplateFilter.processTemplate()    ← variables resolved
  → Css\Processor::preProcess()        ← rewrites relative selectors
  → Emogrifier.inlinize()              ← inlines CSS into HTML
  → Mail\TransportBuilder              ← builds MIME message
```

### Custom CSS Processor

If you need to inject additional CSS before inlining:

```php
use Magento\Email\Model\Template\Css\Processor;
use Pelago\Emogrifier;

class EmailStyleProcessor
{
    public function __construct(
        private readonly Processor $cssProcessor,
    ) {}

    public function process(string $html, string $css): string
    {
        $processedCss = $this->cssProcessor->process($css);
        $emogrifier   = new Emogrifier($html, $processedCss);
        $emogrifier->setHtml5Format(true);
        $emogrifier->disableInvisibleNodeProcessing();
        return $emogrifier->emogrify();
    }
}
```

### Inlining in Custom Code

```php
use Magento\Email\Model\Template\FilterFactory;
use Pelago\Emogrifier;

class CustomEmailSender
{
    public function __construct(
        private readonly FilterFactory $filterFactory,
    ) {}

    public function send(): void
    {
        $filter = $this->filterFactory->create();
        $filter->setTemplateParams(['order' => $order, 'customer' => $customer]);

        $html     = $filter->processTemplate($template->getTemplateHtml());
        $css      = $template->getTemplateStyles();   // from email_template.template_styles
        $emogrifier = new Emogrifier($html, $css);
        $body    = $emogrifier->emogrify();

        // ... pass to TransportBuilder
    }
}
```

---

## 6. Custom Email Templates

### Declaring via `etc/email_templates.xml`

See [Section 2](#2-template-file-resolution) for the full XML structure.

### Rendering in Code

#### Option A — `TemplateFactory` (recommended for custom templates)

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Controller\Order;

use Magento\Email\Model\TemplateFactory;
use Magento\Framework\App\Response\Http;
use Magento\Framework\Controller\ResultFactory;

class SendShipmentEmail
{
    public function __construct(
        private readonly TemplateFactory $templateFactory,
        private readonly ResultFactory   $resultFactory,
    ) {}

    public function execute(): \Magento\Framework\Controller\ResultInterface
    {
        $template = $this->templateFactory->getTemplateModel('vendor_module_order_shipped_email');

        $template->setTemplateParams([
            'order'    => $order,
            'customer' => $customer,
            'shipment' => $shipment,
        ]);

        $template->send($customer->getEmail(), $customer->getName(), [
            'order'    => $order,
            'customer' => $customer,
            'shipment' => $shipment,
        ]);

        return $this->resultFactory->create(ResultFactory::TYPE_JSON)
            ->setData(['status' => 'sent']);
    }
}
```

The `TemplateFactory::getTemplateModel($templateId)` resolves the `id` declared in `email_templates.xml` and loads the file-based template from the appropriate `Vendor_Module::email/...` path.

#### Option B — Direct template entity usage

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Mail;

use Magento\Email\Model\TemplateFactory;
use Magento\Framework\Mail\Template\TransportBuilder;

class ShipmentNotification
{
    public function __construct(
        private readonly TransportBuilder $transportBuilder,
    ) {}

    public function send(
        \Magento\Sales\Model\Order $order,
        \Magento\Sales\Model\Order\Shipment $shipment,
    ): void {
        $this->transportBuilder
            ->setTemplateIdentifier('vendor_module_order_shipped_email')
            ->setTemplateOptions([
                'area'    => \Magento\Framework\App\Area::AREA_FRONTEND,
                'store'   => $order->getStoreId(),
            ])
            ->setTemplateVars([
                'order'    => $order,
                'shipment' => $shipment,
                'customer' => $order->getCustomer(),
            ])
            ->setFromByScope(
                $order->getStore()->getConfig('trans_email/ident_sales/email'),
                $order->getStoreId()
            )
            ->addTo($order->getCustomerEmail(), $order->getCustomerName())
            ->getTransport()
            ->sendMessage();
    }
}
```

---

## 7. Admin Configuration

### Path: System → Configuration → General → Design → Email Templates

| Store scope setting | Config path | Purpose |
|---|---|---|
| Header Template | `design/email/header_template` | Logo + heading block |
| Footer Template | `design/email/footer_template` | Closing text, unsubscribe |
| Sender Email | `trans_email/ident_sales/email` | Reply-to / from address |
| Logo Image | `design/email/logo` | Logo uploaded in admin |

When a template is **stored in the DB**, the admin UI allows editing the subject line, HTML body, plain-text body, and inline styles independently per store scope.

### Adding a custom template to admin

After declaring in `etc/email_templates.xml`, the template appears in the admin dropdown at:
**System → Configuration → General → Design → Email Templates → (Template selector)**

No additional configuration needed — the template is auto-registered via the XML schema.

---

## 8. Preview & Testing

### CLI Commands

```bash
# Preview a specific template with a mock order/customer context
bin/magento dev:email --template Vendor_Module::order_shipped.html

# List all available email templates
bin/magento dev:email --list

# Preview with specific store
bin/magento dev:email --template Vendor_Module::order_shipped.html --store=2
```

The `dev:email` command spins up a minimal CLI context with pre-seeded mock objects (`Order`, `Customer`, `Store`) so you can inspect the fully-rendered HTML.

### Admin Preview

In the admin panel under **Marketing → Email Templates**, clicking "Preview" renders the template with the currently selected store's logo, address, and mock customer data from a test order.

### Inline Styles Verification

After generating the email, inspect the rendered `<body>` for:
- All `style` attributes on every element (confirming Emogrifier ran)
- No `class` or `id` selectors remaining in inline `style` attributes
- `max-width` replaced with explicit `width` (older clients)

---

## Summary

| Concern | Where handled |
|---|---|
| Template storage | `email_template` DB table or `Vendor_Module/view/*/email/` |
| Template registration | `etc/email_templates.xml` (file-based) |
| Variable/directive processing | `Magento\Email\Model\Template\Filter` |
| CSS inlining | `pelago/emogrifier` via `Magento\Email\Model\Template\Css\Processor` |
| Sending | `Magento\Framework\Mail\Template\TransportBuilder` |
| Admin config | System → Configuration → General → Design → Email Templates |
| Preview/test | `bin/magento dev:email` + admin preview |
