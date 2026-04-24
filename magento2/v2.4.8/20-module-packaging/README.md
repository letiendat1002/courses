# Topic 20: Module Packaging — composer.json, SemVer & Marketplace

**Philosophy:** Email is your store's voice in the customer's inbox. Unlike your website (which customers may visit once), transactional emails arrive at moments of high intent — order confirmation, shipment tracking, password reset. This topic teaches you to create, customize, and debug Magento's email system with the same rigor you apply to checkout.

---

## Overview

Magento 2's email system is a fully-featured transactional email platform built on `Magento\Framework\Mail`. It supports:
- HTML email rendering via PHTML templates
- Configurable email templates per event (order, shipment, invoice, etc.)
- Variable interpolation via `{{var}}` syntax
- CSS inlining for email client compatibility
- Admin-defined templates with preview
- Attachments via `MimePartInterface`
- Multi-store email configuration

Unlike the frontend (which renders HTML server-side), email templates in Magento are rendered **synchronously during the request** when the email is sent. This means email template errors cause immediate failures, not just display glitches.

---

## Prerequisites (from core course)

| Module | Why It Matters |
|--------|----------------|
| Topic 03 — Data Layer | Email templates often display entity data (order, customer, product). Understanding repositories helps you load additional data for template use. |
| Topic 05 — Plugins & Observers | Email sending is triggered by events (`sales_order_place_after`, etc.). Plugins on senders let you modify emails without replacing templates. |
| Topic 08 — Configuration | Email settings live in `Stores > Configuration > Sales > Emails`. Admin configuration patterns apply here. |

---

## Learning Objectives

By end of this module, you will:
1. **Understand the email template architecture** — how Magento renders emails, locates templates, and sends via transport
2. **Create custom email templates** — register templates in `email_templates.xml`, write HTML emails compatible with all clients
3. **Use template variables and macros** — inject dynamic data into emails via `TemplateVars` and `setTemplateVars()`
4. **Configure CSS inlining** — understand why it's required and how Magento handles it
5. **Admin-configure email templates** — create editable templates in the admin, preview them before activation
6. **Add custom email senders** — implement `EmailSenderInterface` for your own transactional emails
7. **Handle email failures gracefully** — implement proper error handling for failed sends

---

## By End of Module You Must Prove

- [ ] You can create a custom transactional email template registered in `email_templates.xml`
- [ ] You can write an HTML email template that renders correctly in Gmail, Outlook, and Apple Mail
- [ ] You can add custom variables to an email template via `TemplateVars`
- [ ] You can configure and preview an email template in the Magento admin
- [ ] You can create a custom email sender class implementing `EmailSenderInterface`
- [ ] You understand when to use `setForcedFor` ( BCC) vs direct send

---

## Assessment Criteria

| Criterion | Weight |
|-----------|--------|
| Email template compiles and sends without errors | 30% |
| HTML email renders correctly in at least 3 major email clients (Gmail, Outlook, Apple Mail) | 25% |
| Template uses proper table-based layout (not div-based) | 15% |
| CSS inlining is handled correctly for all styles | 15% |
| Code follows Magento email template conventions | 15% |

---

## Topics

---

### Topic 1: Email Template Architecture

#### How Magento Renders Emails

Magento's email rendering pipeline:

```
Trigger Event (e.g., order placed)
    ↓
Observer catches event
    ↓
Loads email template via `EmailTemplateIdentity` (stores/emails)
    ↓
Renders PHTML template with variables via `Magento\Framework\Mail\Template\TransportBuilder`
    ↓
Applies CSS inlining (via `CssInliner` or inline="1" in template)
    ↓
Sends via `Magento\Framework\Mail\TransportInterface`
```

**The `EmailTemplate` Model**

The core class handling email template rendering is `Magento\Framework\Mail\Template\Template`:

```php
<?php
namespace Magento\Framework\Mail\Template;

class Template implements \Magento\Framework\Mail\TemplateInterface
{
    // Key methods:
    // - setTemplateType(Template::TYPE_HTML | TYPE_TEXT)
    // - setTemplateText($text)
    // - setTemplateVars(array $vars)
    // - setTemplateOptions(['area' => 'frontend', 'store' => $storeId])
    // - getProcessedTemplate() — returns rendered HTML
}
```

**The Transport Builder Pattern**

The `TransportBuilder` is the fluent interface for building emails:

```php
<?php
use Magento\Framework\Mail\Template\TransportBuilder;

public function sendOrderEmail(
    TransportBuilder $transportBuilder,
    Order $order
): void {
    $transport = $transportBuilder
        ->setTemplateIdentifier('sales_email_order_template')  // from email_templates.xml
        ->setTemplateOptions([
            'area' => \Magento\Framework\App\Area::AREA_FRONTEND,
            'store' => $order->getStoreId(),
        ])
        ->setTemplateVars([
            'order' => $order,
            'customer_name' => $order->getCustomerName(),
            'order_id' => $order->getIncrementId(),
        ])
        ->setFrom([
            'name' => 'Store Name',
            'email' => 'orders@example.com',
        ])
        ->addTo($order->getCustomerEmail(), $order->getCustomerName())
        ->getTransport();

    $transport->sendMessage();
}
```

**Why `setTemplateOptions` Matters**

The `area` parameter tells Magento which directory to look in for the template:
- `AREA_FRONTEND` → `<module>/view/frontend/templates/email/`
- `AREA_ADMINHTML` → `<module>/view/adminhtml/templates/email/`

If you forget to set the area, Magento will look in the wrong directory and fail silently or throw a template file not found error.

#### Email Templates XML Registration

Email templates are registered in `etc/email_templates.xml`:

```xml
<?xml version="1.0"?>
<!-- Training/Email/etc/email_templates.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Email:etc/email_templates.xsd">
    <template id="training_email_custom_template"
              label="Custom Training Email"
              file="custom_email.html"
              module="Training_Email"
              type="html"/>
</config>
```

**The `id` attribute** is the identifier used in `setTemplateIdentifier()`.

**The `type` attribute** can be `html` (default) or `text`.

#### Template File Locations

| Template Type | Location |
|--------------|----------|
| Frontend emails | `<module>/view/frontend/templates/email/` |
| Admin emails | `<module>/view/adminhtml/templates/email/` |
| System emails (from config) | `<module>/view/frontend/templates/email/` or admin configured path |

**Why Separate Areas?**

Frontend templates are accessible by customers and should match the storefront design. Admin templates are internal and may include operational details not appropriate for customer-facing emails.

---

### Topic 2: Writing HTML Email Templates

#### The Fundamental Rule: Table-Based Layouts

Unlike modern web development (which uses `div` + `flexbox`/`grid`), email clients **still require table-based layouts** in 2024. This is not a suggestion — it's a requirement for compatibility.

**Why Tables?**

Email clients (especially Outlook) have notoriously broken CSS support. Tables are the only reliable way to ensure:
- Consistent column widths across clients
- Vertical alignment works
- Background colors span the full area
- Content doesn't break in unexpected ways

**The Email Layout Structure:**

```html
<!-- Outer wrapper table — controls max-width -->
<table width="100%" cellpadding="0" cellspacing="0" border="0"
       style="max-width: 600px; margin: 0 auto; background-color: #ffffff;">
    <tr>
        <td>
            <!-- Header table -->
            <table width="100%" cellpadding="0" cellspacing="0" border="0">
                <tr>
                    <td style="padding: 20px; background-color: #333333;">
                        <!-- Logo and store name -->
                    </td>
                </tr>
            </table>

            <!-- Main content table -->
            <table width="100%" cellpadding="0" cellspacing="0" border="0">
                <tr>
                    <td style="padding: 30px 20px;">
                        <!-- Dynamic content goes here -->
                    </td>
                </tr>
            </table>

            <!-- Footer table -->
            <table width="100%" cellpadding="0" cellspacing="0" border="0">
                <tr>
                    <td style="padding: 20px; background-color: #f4f4f4; font-size: 12px; color: #666666;">
                        <!-- Footer content -->
                    </td>
                </tr>
            </table>
        </td>
    </tr>
</table>
```

**Key CSS Rules for Email:**

```html
<!-- Use inline styles — external CSS is unreliable in email -->
<td style="padding: 20px; font-family: Arial, Helvetica, sans-serif; font-size: 14px; line-height: 1.5; color: #333333;">

<!-- Max-width for responsive emails (Outlook doesn't support max-width in media queries) -->
<table width="100%" cellpadding="0" cellspacing="0" border="0"
       style="max-width: 600px; margin: 0 auto;">

<!-- Avoid shorthand padding — use explicit directions -->
<!-- WRONG: padding: 20px -->
<!-- RIGHT: padding: 20px 20px 20px 20px or padding: 20px -->
```

#### Using Magento's Built-in Header and Footer

Magento provides standard header and footer templates:

```html
<!-- Header template: Magento_Sales/templates/email/header.html -->
{{template config_path="design/email/header_template"}}

<!-- Footer template: Magento_Sales/templates/email/footer.html -->
{{template config_path="design/email/footer_template"}}
```

**Why Use These?**

Using `{{template config_path="..."}}` makes header/footer configurable in the admin:
- `Stores > Configuration > General > Design > Email Templates > Header Template`
- `Stores > Configuration > General > Design > Email Templates > Footer Template`

This allows marketers to change logos and colors without touching code.

#### CSS Inlining: The Critical Step

**Why CSS Inlining is Required**

Most email clients (Gmail, Outlook) strip `<style>` blocks from the `<head>` when rendering. Inline styles are the only reliable way to apply styling.

**Magento's Inline CSS Processing:**

Magento automatically inlines CSS when you use the template system. You can force inlining by adding `inline="1"` to your template variable:

```php
$transport = $this->transportBuilder
    ->setTemplateIdentifier('my_template')
    ->setTemplateOptions(['area' => 'frontend', 'store' => $storeId, 'inline' => '1'])
    // ...
```

**In the template file itself:**

```html
<!-- Add this comment to opt-in to inline CSS processing -->
<!-- template她的手 -->
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>
    <style type="text/css">
        .button { background-color: #0066cc; color: #ffffff; padding: 10px 20px; text-decoration: none; }
        .text { font-family: Arial, sans-serif; font-size: 14px; color: #333333; }
    </style>
</head>
<body>
<!-- The inline CSS processor will inline these styles into the HTML elements -->
<table class="button" cellpadding="0" cellspacing="0" border="0">
    <tr>
        <td style="padding: 10px 20px;">
            <a href="#" style="color: #ffffff; text-decoration: none;">Click Here</a>
        </td>
    </tr>
</table>
</body>
```

**The "Roadmark" CSS Issue**

When CSS classes reference each other (e.g., `.container .button`), the inliner can't resolve cross-class relationships. Always use **flat selectors** in email CSS:

```html
<!-- WRONG: nested selectors don't work when inlined -->
<style>
.container .button { background-color: #0066cc; }
</style>
<table class="container"><tr><td><a class="button">CLICK</a></td></tr></table>

<!-- RIGHT: flat class names -->
<style>
.button { background-color: #0066cc; }
</style>
<table class="button"><tr><td>CLICK</td></tr></table>
```

---

### Topic 3: Template Variables and Macros

#### Variable Syntax: `{{var}}` and `{{depend}}`

Magento's email template variable system uses double-brace syntax:

| Syntax | Purpose | Example |
|--------|---------|---------|
| `{{var variable_name}}` | Output variable | `{{var order.getIncrementId()}}` |
| `{{var var.subproperty}}` | Object property | `{{var order.customer.name}}` |
| `{{depend variable}}...{{/depend}}` | Conditional block | `{{depend order.isVirtual()}}Virtual{{/depend}}` |
| `{{if condition}}...{{else}}...{{/if}}` | If/else block | `{{if show_vat}}VAT: {{var vat}}{{/if}}` |
| `{{htmlescape var="var_name"}}` | Escape HTML | `{{htmlescape var=$customer.name}}` |
| `{{template config_path="..."}}` | Include subtemplate | `{{template config_path="design/email/header"}}` |

**Why `htmlescape` Matters**

Email is a common vector for XSS attacks. Always escape user-generated content:

```html
<!-- WRONG: unsanitized output -->
<p>Hello {{var customer.getName()}},</p>

<!-- RIGHT: escaped output -->
<p>Hello {{htmlescape var=$customer.getName()}},</p>
```

#### Custom Variables via `TemplateVars`

The most common way to pass data to templates:

```php
<?php
use Magento\Framework\Mail\Template\TransportBuilder;

public function sendCustomEmail(
    TransportBuilder $transportBuilder,
    string $recipientEmail,
    string $recipientName,
    array $customData
): void {
    $transport = $transportBuilder
        ->setTemplateIdentifier('training_email_custom')
        ->setTemplateOptions([
            'area' => \Magento\Framework\App\Area::AREA_FRONTEND,
            'store' => $this->storeManager->getStore()->getId(),
        ])
        ->setTemplateVars([
            'customer_name' => $recipientName,
            'order_number' => $customData['order_number'] ?? 'N/A',
            'items' => $customData['items'] ?? [],
            'total' => $customData['total'] ?? 0.00,
            'order_date' => $customData['order_date'] ?? '',
        ])
        ->setFrom([
            'name' => 'Store Support',
            'email' => 'support@example.com',
        ])
        ->addTo($recipientEmail, $recipientName)
        ->getTransport();

    $transport->sendMessage();
}
```

**In the template:**

```html
<!-- training_email_custom.html -->
<!-- template她的手 -->
<head>
    <meta charset="utf-8"/>
    <style>
        .header { background-color: #333333; color: #ffffff; padding: 20px; text-align: center; }
        .content { padding: 30px 20px; font-family: Arial, sans-serif; }
        .item-row { border-bottom: 1px solid #eeeeee; padding: 10px 0; }
        .total-row { font-weight: bold; padding: 10px 0; }
        .footer { background-color: #f4f4f4; padding: 20px; font-size: 12px; color: #666666; text-align: center; }
    </style>
</head>
<body>
{{template config_path="design/email/header_template"}}

<table class="content" width="100%" cellpadding="0" cellspacing="0" border="0">
    <tr>
        <td>
            <h1 style="margin: 0 0 20px 0; font-size: 24px;">Thank You, {{var customer_name}}!</h1>
            <p style="margin: 0 0 20px 0;">Your order #{{var order_number}} has been confirmed.</p>

            {{depend items}}
            <table class="items" width="100%" cellpadding="5" cellspacing="0" border="0">
                <tr style="font-weight: bold; background-color: #f9f9f9;">
                    <td>Item</td>
                    <td align="right">Qty</td>
                    <td align="right">Price</td>
                </tr>
                {{var items}}
            </table>
            {{/depend}}

            <table class="total-row" width="100%" cellpadding="0" cellspacing="0" border="0">
                <tr>
                    <td align="right">Total: ${{var total}}</td>
                </tr>
            </table>
        </td>
    </tr>
</table>

{{template config_path="design/email/footer_template"}}
</body>
```

**Pro Tip: Pre-render Complex Data**

Don't do complex calculations in the template. Pre-render them in the sender class:

```php
// WRONG: logic in template
{{var order.getSubtotal() + order.getTaxAmount() + order.getShippingAmount()}}

// RIGHT: pre-calculate and pass as variable
$emailData = [
    'grand_total' => $order->getGrandTotal(),
    'items_html' => $this->renderItemsHtml($order->getItems()),
];
```

#### Using Macros for Reusable Components

For repeatable elements (order items, address blocks), use Magento's macro system:

```html
<!-- Define a macro at the top of your template -->
{{macro addressRow(label, value)}}
<tr>
    <td style="padding: 5px 0; font-weight: bold; width: 120px;">{{var label}}:</td>
    <td style="padding: 5px 0;">{{var value}}</td>
</tr>
{{/macro}}

<!-- Use the macro multiple times -->
<table class="address" cellpadding="0" cellspacing="0" border="0">
    {{require type="macro" template="Magento_Sales::email/macros"}}
    {{.addressRow label="Name" value=$shippingAddress.name}}
    {{.addressRow label="Street" value=$shippingAddress.street}}
    {{.addressRow label="City" value=$shippingAddress.city}}
</table>
```

---

### Topic 4: Custom Email Senders

#### Implementing `EmailSenderInterface`

Magento provides a clean interface for transactional email senders:

```php
<?php
namespace Magento\Sales\Model\Order\Email;

class OrderSender extends \Magento\Sales\Model\Order\Email\Sender
{
    public function send(Order $order, bool $forceSyncMode = false): void
    {
        $this->identityContainer->setStoreId($order->getStoreId());

        if (!$this->identityContainer->isEnabled() && !$forceSyncMode) {
            return;
        }

        $this->checkAndSend($order);
    }
}
```

**The Sender Pattern:**

```php
<?php
namespace Magento\Framework\Mail;

class TransportBuilder
{
    // Key methods:
    public function setTemplateIdentifier(string $templateId): self;
    public function setTemplateOptions(array $options): self;
    public function setTemplateVars(array $vars): self;
    public function setFrom(mixed $from): self;
    public function addTo(string $address, string $name = null): self;
    public function addCc(string $address, string $name = null): self;
    public function addBcc(string $address): self;
    public function setReplyTo(string $address, string $name = null): self;
    public function getTransport(): TransportInterface;
}
```

#### Creating a Custom Order Sender

**Step 1: Define template in `etc/email_templates.xml`:**

```xml
<!-- Training/Email/etc/email_templates.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Email:etc/email_templates.xsd">
    <template id="training_email_order_confirmation"
              label="Training Order Confirmation"
              file="order_confirmation.html"
              module="Training_Email"
              type="html"/>
</config>
```

**Step 2: Implement the sender class:**

```php
<?php
declare(strict_types=1);

namespace Training\Email\Model\Order;

use Magento\Sales\Model\Order;
use Magento\Sales\Model\Order\Email\Sender;
use Magento\Sales\Model\Order\Email\SenderBuilder;

class ConfirmationSender extends Sender
{
    protected $senderBuilder;

    public function __construct(
        \Magento\Sales\Model\Order\Email\SenderBuilderFactory $senderBuilderFactory,
        \Magento\Sales\Model\Order\Email\Container\Template $templateContainer,
        \Magento\Sales\Model\Order\Email\Container\IdentityInterface $identityContainer,
        \Magento\Framework\Mail\Template\TransportBuilder $transportBuilder,
        \Magento\Framework\App\Area\FrontNameResolverInterface $areaResolver
    ) {
        parent::__construct(
            $templateContainer,
            $identityContainer,
            $transportBuilder,
            $areaResolver
        );
        $this->senderBuilder = $senderBuilderFactory->create();
    }

    /**
     * Send order confirmation email.
     */
    public function send(Order $order, bool $forceSyncMode = false): void
    {
        $this->templateContainer->setTemplateId('training_email_order_confirmation');
        $this->identityContainer->setStoreId($order->getStoreId());

        if (!$this->identityContainer->isEnabled() && !$forceSyncMode) {
            return;
        }

        $this->senderBuilder->send();
    }
}
```

**Step 3: Register the sender in events:**

```xml
<!-- etc/events.xml -->
<event name="sales_order_place_after">
    <observer name="training_email_send_confirmation"
              instance="Training\Email\Observer\SendOrderConfirmation"
              shared="false"/>
</event>
```

**Step 4: Implement the observer:**

```php
<?php
declare(strict_types=1);

namespace Training\Email\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Model\Order;

class SendOrderConfirmation implements ObserverInterface
{
    private ConfirmationSender $confirmationSender;

    public function __construct(ConfirmationSender $confirmationSender)
    {
        $this->confirmationSender = $confirmationSender;
    }

    public function execute(Observer $observer): void
    {
        /** @var Order $order */
        $order = $observer->getOrder();

        if (!$order->getId()) {
            return;
        }

        try {
            $this->confirmationSender->send($order);
        } catch (\Exception $e) {
            // Log but don't throw — email failure shouldn't fail the order
            $this->logger->error('Failed to send order confirmation email', [
                'order_id' => $order->getId(),
                'error' => $e->getMessage(),
            ]);
        }
    }
}
```

**Why Catch Exceptions in Email Senders?**

Email delivery failures should **never** fail the order placement. An SMTP outage should not cause a checkout failure. Always wrap email sending in try/catch with logging.

#### BCC: Blind Carbon Copy for Email Tracking

BCC is used to send a copy of every email to a monitoring address without the recipient knowing:

```php
<?php
$transport = $this->transportBuilder
    ->setTemplateIdentifier('my_template')
    ->setTemplateOptions([...])
    ->setTemplateVars([...])
    ->setFrom($this->identityContainer->getEmailIdentity())
    ->addTo($order->getCustomerEmail())
    ->addBcc('orders-monitor@example.com')  // Blind CC — recipient won't see this
    ->getTransport();

$transport->sendMessage();
```

**When to Use BCC:**
- Email delivery monitoring (archive all outgoing emails)
- Customer support CC (support sees all order emails)
- Audit/compliance (retain copies of all communications)

---

### Topic 5: Admin Configuration and Preview

#### Configuring Email Templates in Admin

**Path:** `Stores > Configuration > Sales > Emails`

Each email type has settings:
- Enabled/Disabled toggle
- Email template selector (dropdown of registered templates)
- Sender email identity (Store name, Custom email)

**Adding Template Options to Admin:**

To make a custom email template selectable in the admin:

```xml
<!-- etc/adminhtml/system.xml -->
<system>
    <section id="sales">
        <group id="email">
            <field id="training_order_template" translate="label" type="select" sortOrder="100">
                <label>Training Order Confirmation Template</label>
                <source_model>Magento\Config\Model\Config\Source\Email\Template</source_model>
            </field>
        </group>
    </section>
</system>
```

#### Previewing Email Templates

**In Admin:**
1. Navigate to `Marketing > Email Templates`
2. Click "Add New Template"
3. Select your template from "Load default template" dropdown
4. Click "Preview" to see the rendered HTML

**Programmatically:**

```php
<?php
use Magento\Email\Model\ResourceModel\Template as TemplateResource;
use Magento\Email\Model\TemplateFactory;

public function previewTemplate(
    TemplateFactory $templateFactory,
    TemplateResource $templateResource,
    string $templateId,
    int $storeId
): string {
    /** @var \Magento\Email\Model\Template $template */
    $template = $templateFactory->create();
    $template->setTemplateId($templateId);
    $template->setTemplateOptions([
        'area' => \Magento\Framework\App\Area::AREA_FRONTEND,
        'store' => $storeId,
    ]);

    // Load preview vars (typically from a sample order/customer)
    $template->setTemplateVars([
        'order' => $this->getSampleOrder(),  // Get a real or mock order
        'store' => $this->storeManager->getStore($storeId),
    ]);

    return $template->getProcessedTemplate();
}
```

**Preview URL:**

You can also preview via URL in developer mode:
```
https://yourstore.com/admin/email/template/preview/?template_id=sales_email_order_template
```

**Why Preview Matters**

HTML email rendering varies wildly between clients. Always preview before activating:
- **Gmail:** Most lenient, but clips long emails
- **Outlook:** Has the worst CSS support (uses Word rendering engine)
- **Apple Mail:** Generally good, but watch for font fallbacks

---

### Topic 6: Common Pitfalls and Best Practices

#### Email Client Compatibility Checklist

| Issue | Affected Clients | Solution |
|-------|------------------|----------|
| `max-width` ignored | Outlook | Use `width` on `<table>` elements, not CSS |
| Media queries ignored | Outlook, Gmail | Don't rely on responsive email CSS |
| `background-image` stripped | Most clients | Use `background-color` instead |
| Fonts fallback | All | Use web-safe fonts: Arial, Helvetica, Georgia, Times |
| Images blocked by default | Most clients | Always include `alt` text; 300px width minimum |
| HTML entities decoded | Gmail | Use ASCII when possible in links |

**Pro Tip: Test with Litmus or Email on Acid**

These services render your email in 90+ email clients and show screenshots. This is the industry standard for email QA.

#### Security: XSS in Email Templates

Email is a high-value XSS target because it bypasses browser CSP:

```html
<!-- WRONG: Unsanitized user data in email -->
<p>Welcome, {{var customer.getName()}}</p>

<!-- RIGHT: Always use htmlescape for user data -->
<p>Welcome, {{htmlescape var=$customer.getName()}}</p>
```

**Why Not Just `{{var}}`?**

The `{{var}}` syntax does NOT auto-escape. It outputs raw data. Only `{{htmlescape var=...}}` provides XSS protection. Treat all user data as potentially malicious.

#### Handling Failed Sends

```php
<?php
public function sendEmailWithErrorHandling(
    TransportBuilder $transportBuilder,
    Order $order
): bool {
    try {
        $transport = $transportBuilder
            ->setTemplateIdentifier('my_template')
            ->setTemplateOptions([...])
            ->setTemplateVars([...])
            ->setFrom($this->identityContainer->getEmailIdentity())
            ->addTo($order->getCustomerEmail())
            ->getTransport();

        $transport->sendMessage();
        return true;
    } catch (\Magento\Framework\Exception\MailException $e) {
        // SMTP failure — log and continue
        $this->logger->error('SMTP failure sending order email', [
            'order_id' => $order->getId(),
            'error' => $e->getMessage(),
        ]);
        return false;
    } catch (\Exception $e) {
        // Template rendering failure — this is a code bug
        $this->logger->critical('Email template error', [
            'order_id' => $order->getId(),
            'error' => $e->getMessage(),
            'trace' => $e->getTraceAsString(),
        ]);
        return false;
    }
}
```

#### Attachment Support

```php
<?php
use Magento\Framework\Mail\EmailMessageInterfaceFactory;
use Magento\Framework\Mail\MimePartInterfaceFactory;
use Zend\Mime\Part;
use Zend\Mime\Message;

public function sendWithAttachment(
    TransportBuilder $transportBuilder,
    string $recipientEmail,
    string $pdfPath
): void {
    $pdfContent = file_get_contents($pdfPath);

    $attachment = new Part($pdfContent);
    $attachment->type = 'application/pdf';
    $attachment->filename = 'order-confirmation.pdf';
    $attachment->encoding = Part::ENCODING_BASE64;

    $mimeMessage = new Message();
    $mimeMessage->addPart($attachment);

    $this->transportBuilder
        ->setTemplateIdentifier('my_template')
        ->setTemplateOptions([...])
        ->setTemplateVars([...])
        ->setFrom([...])
        ->addTo($recipientEmail)
        ->setBody($mimeMessage);

    $this->transportBuilder->getTransport()->sendMessage();
}
```

---

## Reference Exercises

**Exercise 1:** Create a custom email template for a "Welcome New Customer" email. The template should include:
- Store logo and header
- Personalized greeting with customer name
- Account activation link
- Store contact information

**Exercise 2:** Create a CLI command `bin/magento training:email:preview <template_id>` that renders and displays a template with sample data.

**Exercise 3:** Create a custom email sender class for a "Low Stock Alert" email that is triggered when a product's quantity falls below a threshold. Include product details in the email.

**Exercise 4:** Add BCC to all order-related emails so that a "orders-archive@example.com" address receives copies of all order emails without customers knowing.

**Exercise 5:** Create a custom email template for "Abandoned Cart" recovery. The template should include:
- Cart items with images (via `resize(100)`)
- Subtotal and checkout link

---

## Reading List

| Resource | Why Read It |
|----------|-------------|
| `Magento/Email/etc/email_templates.xsd` | Schema for email template XML registration |
| `Magento/Framework/Mail/Template/TransportBuilder.php` | The transport builder API |
| `Magento/Sales/Model/Order/Email/Sender/OrderSender.php` | How Magento sends order emails |
| `Magento/Email/Model/Template.php` | The template rendering engine |
| [Litmus Email Testing](https://www.litmus.com/) | Email client compatibility testing |
| [Mailgun Email Testing Guide](https://www.mailgun.com/blog/email-testing/) | How to test transactional emails |
| [Google SMTP Settings for Magento](https://support.google.com/accounts/answer/6010255) | Configuring Gmail as SMTP relay |

---

## Edge Cases & Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Email not sending | SMTP not configured or disabled | Check `Stores > Configuration > Sales > Emails > General Settings` |
| Template file not found | Wrong `area` in `setTemplateOptions` | Use `AREA_FRONTEND` for frontend templates |
| CSS styles not applying | Inline CSS not enabled | Add `'inline' => '1'` to `setTemplateOptions` |
| Variables not replaced | Missing `setTemplateVars` or typo in variable name | Verify variable names match exactly (case-sensitive) |
| Email shows raw HTML | Wrong content-type header | Ensure template uses `type="html"` in `email_templates.xml` |
| Image broken in email | External images blocked by default | Use CID (Content-ID) attachments for images, or upload to CDN |
| WYSIWYG editor corrupting HTML | Admin WYSIWYG is aggressive | Use plain text editor for email templates in admin |
| Preview works but live fails | Different store/area context | Check that live email uses correct store ID and area |

---

## Common Mistakes to Avoid

1. ❌ Using `div`-based layout → Email clients (especially Outlook) break div layouts
   - **Fix:** Always use table-based layouts for email-compatible HTML

2. ❌ Not escaping user data → XSS vulnerability in email
   - **Fix:** Always use `{{htmlescape var=$var}}` for customer-provided data

3. ❌ Forgetting `setTemplateOptions` → Template file not found
   - **Fix:** Always set `['area' => '...', 'store' => $storeId]`

4. ❌ Throwing exceptions in email observers → Order placement fails
   - **Fix:** Always wrap email sending in try/catch with logging

5. ❌ Using external CSS `<style>` blocks → Most email clients strip these
   - **Fix:** Use inline styles or rely on Magento's CSS inlining feature

6. ❌ Non-responsive images → Horizontal scroll on mobile
   - **Fix:** Use `width="100%"` with `max-width` on images, test on mobile clients

7. ❌ Sending to invalid email address → Uncaught exception
   - **Fix:** Validate email format before adding to `addTo()`

8. ❌ Not testing email client compatibility → Broken layout in Outlook/Gmail
   - **Fix:** Use Litmus or Email on Acid to test across 90+ clients

---

*Magento 2 Backend Developer Course — Topic 11 | Email Templates*
