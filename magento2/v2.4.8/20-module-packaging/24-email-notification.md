---
title: "24 - Email & Notification Architecture"
description: "Magento 2.4.8 email system: email templates, sender configuration, transport, transactional email processing, and notification patterns."
tags: magento2, email, email-template, transport, mail, notification, sender, template-pattern, transactional-email
rank: 24
pathways: [magento2-deep-dive]
see_also:
  - path: "20-module-packaging/05-email-templates.md"
    description: "Email Templates chapter — template creation and customization"
  - path: "10-checkout/README.md"
    description: "Checkout — order confirmation email trigger"
---

# Email & Notification Architecture

Magento's email system handles transactional emails (order confirmations, shipment notifications, password resets) and administrative alerts. The architecture supports store-specific sender identities, variable interpolation in templates, and HTML/plain-text multipart emails.

---

## 1. Email Architecture Overview

### Core Components

| Component | Class | Purpose |
|-----------|-------|---------|
| Template | `\Magento\Email\Model\Template` | Email content with variable interpolation |
| Transport | `\Magento\Email\Model\Transport` | Sends email via SMTP or PHP mail |
| Sender | `\Magento\Framework\Mail\TransportInterface` | Wraps template + transport |
| Sender Builder | `\Magento\Framework\Mail\EmailTemplateInterface` | Creates sender from template |
| Identity | `\Magento\Framework\Mail\EmailIdentityInterface` | Sender name/email resolution |

### Email Flow

```
Code triggers email send
    ↓
Template loaded (HTML + plain text)
    ↓
Variables interpolated (customer name, order data, etc.)
    ↓
Sender identity resolved (from store config)
    ↓
Email transport (SMTP/Mail)
    ↓
MIME message constructed
    ↓
Delivery to MTA
```

---

## 2. Email Transport

### Configure Email Transport

```php
<?php
// In etc/di.xml

// Default PHP mail
<type name="Magento\Framework\Mail\Transport">
    <arguments>
        <argument name="message">Magento\Framework\Mail\Message</argument>
    </arguments>
</type>

// SMTP transport
<type name="Magento\Framework\Mail\Transport">
    <arguments>
        <argument name="message">Magento\Framework\Mail\Message</argument>
        <argument name="pipelineConfig">smtp-config</argument>
    </arguments>
</type>
```

### SMTP Configuration

```php
<?php
// In Stores → Configuration → Catalog → Email (or General → Email)

// Available transport types:
// - PHP Mail (default)
// - SMTP (requires host/port configuration)

// SMTP Configuration:
'system' => [
    'default' => [
        'general' => [
            'email' => [
                'transport' => [
                    'type' => 'smtp',
                    'host' => 'smtp.example.com',
                    'port' => 587,
                    'auth' => 'login',
                    'username' => 'user@example.com',
                    'password' => 'secret',
                    'secure' => 'tls',  // tls or ssl
                ]
            ]
        ]
    ]
]
```

### Sending Email Programmatically

```php
<?php
public function __construct(
    \Magento\Framework\Mail\TransportInterface $transport
) {
    $this->transport = $transport;
}

public function sendEmail(): void
{
    $message = $this->messageFactory->create();
    $message->setSubject('Order Confirmation')
        ->setFromEmail('noreply@example.com')
        ->setFromName('Magento Store')
        ->setToEmail('customer@example.com')
        ->setBodyHtml('<h1>Thank you!</h1><p>Your order is confirmed.</p>')
        ->setBodyText('Thank you! Your order is confirmed.');

    $this->transport->send($message);
}
```

---

## 3. Email Templates

### Standard Template Files

```
app/code/Magento/Sales/etc/email_templates.xml   ← Declares templates
app/code/Magento/Sales/Model/Template/          ← Template models
```

### Template Declaration (email_templates.xml)

```xml
<!-- etc/email_templates.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Email:etc/email_templates.xsd">
    <template id="sales_email_order_template"
              label="Order Confirmation"
              file="order_new.html"
              type="html"
              module="Magento_Sales"
              area="frontend"/>
    <template id="sales_email_order_guest_template"
              label="Order Confirmation (Guest)"
              file="order_new_guest.html"
              type="html"
              module="Magento_Sales"
              area="frontend"/>
</config>
```

### Template Variables

Templates use `{{var variable_name}}` syntax:

```html
<!-- order_new.html -->
<h1>Thank you for your order, {{var customer_name}}!</h1>

<p>Order Number: {{var order_id}}</p>
<p>Order Date: {{var order_date}}</p>

<table>
    {{var order_details}}
</table>

<p>Subtotal: {{var order_subtotal}}</p>
<p>Shipping: {{var order_shipping}}</p>
<p>Total: {{var order_total}}</p>

<p>Shipping Address:</p>
{{var shipping_address}}

<p>Billing Address:</p>
{{var billing_address}}
```

### Custom Variables in Templates

```php
<?php
// Add custom variables to email template

public function sendCustomEmail(string $customerName, string $promoCode): void
{
    $templateVars = [
        'customer_name' => $customerName,
        'promo_code' => $promoCode,
        'expiry_date' => $this->getPromoExpiry($promoCode)
    ];

    /** @var \Magento\Email\Model\Template $template */
    $template = $this->templateFactory->get('custom_email_template');

    $template->setTemplateVars($templateVars)
        ->setSenderName($this->getStore()->getName())
        ->setSenderEmail($this->getStore()->getEmail());

    $template->send('customer@example.com', $customerName);
}
```

---

## 4. Sending Emails via SenderBuilder

### Modern Approach (SenderBuilder Pattern)

```php
<?php
// \Magento\Sales\Model\Order\Email\SenderBuilder

class OrderSenderBuilder extends \Magento\Framework\Email\SenderBuilder
{
    public function __construct(
        \Magento\Email\Model\Template $emailTemplate,
        \Magento\Framework\Mail\TransportInterfaceFactory $transportFactory,
        \Magento\Framework\ObjectManager\ConfigInterface $objectManagerConfig,
        \Magento\Framework\Mail\AddressConverter $addressConverter
    ) {
        parent::__construct($emailTemplate, $transportFactory, $objectManagerConfig, $addressConverter);
    }
}
```

### Complete Example

```php
<?php
// Custom module: Vendor/Email/Model/Sender/CustomNotificationSender.php

namespace Vendor\Email\Model\Sender;

use Magento\Framework\Mail\Template\TransportBuilder;
use Magento\Store\Model\StoreManagerInterface;

class CustomNotificationSender
{
    private TransportBuilder $transportBuilder;
    private StoreManagerInterface $storeManager;

    public function __construct(
        TransportBuilder $transportBuilder,
        StoreManagerInterface $storeManager
    ) {
        $this->transportBuilder = $transportBuilder;
        $this->storeManager = $storeManager;
    }

    public function sendNotification(
        string $toEmail,
        string $toName,
        array $templateVars
    ): void {
        $store = $this->storeManager->getStore();

        $this->transportBuilder
            ->setTemplateIdentifier('custom_notification_template')
            ->setTemplateModel(\Magento\Email\Model\Template::class)
            ->setTemplateVars($templateVars)
            ->setFrom([
                'email' => $store->getSenderEmail(),
                'name' => $store->getSenderName()
            ])
            ->addTo($toEmail, $toName)
            ->getTransport()
            ->sendMessage();
    }
}
```

---

## 5. Order Email Sending

### Order Confirmation Email

```php
<?php
// \Magento\Sales\Model\Order\Email\Sender\OrderSender

class OrderSender extends \Magento\Sales\Model\Order\Email\Sender
{
    protected function prepareTemplate(Order $order): void
    {
        $template = $this->templateContainer->getTemplate();

        // Set template ID based on guest vs customer
        if ($order->getCustomerIsGuest()) {
            $templateId = $this->senderConfig->getGuestTemplateId();
            $template->setTemplateId($templateId);
        } else {
            $templateId = $this->senderConfig->getTemplateId();
            $template->setTemplateId($templateId);
        }

        // Populate template vars
        $templateContainer->setTemplateVars([
            'order' => $order,
            'order_id' => $order->getIncrementId(),
            'customer_name' => $order->getCustomerName(),
            'billing' => $order->getBillingAddress(),
            'payment_html' => $this->getPaymentHtml($order),
            'shipping_html' => $this->getShippingHtml($order),
        ]);
    }
}
```

---

## 6. Shipment Email

### Shipment Notification

```php
<?php
// \Magento\Sales\Model\Order\Email\Sender\ShipmentSender

class ShipmentSender extends \Magento\Sales\Model\Order\Email\Sender\Sender
{
    protected function prepareTemplate(Order $order): void
    {
        $templateId = $this->identityContainer->getTemplateId();
        $template->setTemplateId($templateId);

        // Template vars for shipment
        $templateContainer->setTemplateVars([
            'order' => $order,
            'shipment' => $this->shipment,
            'comment' => $this->identityContainer->getComment(),
            'billing' => $order->getBillingAddress(),
            'payment_html' => $this->getPaymentHtml(),
            'shipping_html' => $this->getShippingHtml(),
            'order_id' => $order->getIncrementId(),
        ]);
    }
}
```

---

## 7. Admin Email Notifications

### New Order Admin Alert

```php
<?php
// \Magento\Sales\Model\Order\Email\NotifySender

// Configured in Admin: Stores → Configuration → Sales → Sales Emails
// Can send to: Order Confirmation Email Recipient

$identity = $this->identityContainer->getConfigValue('order_email_receiver_email');
$identityName = $this->identityContainer->getConfigValue('order_email_receiver_name');
```

### Custom Admin Notification

```php
<?php
public function sendAdminAlert(
    string $alertMessage,
    string $severity = 'critical'
): void {
    $adminEmails = $this->getAdminEmailsFromConfig();

    foreach ($adminEmails as $email) {
        $this->transportBuilder
            ->setTemplateIdentifier('admin_alert_template')
            ->setTemplateVars([
                'alert_message' => $alertMessage,
                'severity' => $severity,
                'store_name' => $this->storeManager->getStore()->getName()
            ])
            ->setFrom('general')
            ->addTo($email)
            ->getTransport()
            ->sendMessage();
    }
}
```

---

## 8. Email Template Customization

### Override Default Templates

```xml
<!-- app/design/frontend/Vendor/Theme/Magento_Sales/email/template.html -->

<!-- Copy Magento_Sales/templates/email/order_new.html -->
<!-- Modify as needed -->
```

### Programmatically Modify Template

```php
<?php
// Plugin on template loading
public function beforeLoadTemplate(
    \Magento\Email\Model\Template $subject,
    string $templateId
): array {
    if ($templateId === 'sales_email_order_template') {
        // Swap to custom template
        $templateId = 'custom_order_template';
    }
    return [$templateId];
}
```

---

## 9. Common Email Issues

### Debug Email Sending

```bash
# Turn off email sending in development
bin/magento config:set mail/smtp/disable 1

# Or set catch-all email in development
bin/magento config:set mail/smtp/catch_all_email developer@example.com
```

### Email Not Sending

```php
<?php
// Check if emails are disabled
$isDisabled = $this->scopeConfig->getValue(
    'system/smtp/disable',
    \Magento\Store\Model\ScopeInterface::SCOPE_STORE
);

if ($isDisabled) {
    // Email sending is disabled
}
```

### Template Variables Not Replaced

```php
<?php
// Ensure variables are set on template before sending
$template->setTemplateVars($vars);
$template->setForcedScope($storeId);

// The template model must have:
// 1. Template ID set
// 2. Template vars set
// 3. Sender identity set
// 4. Recipient set
```

---

## Reading List

- [Email configuration](https://experienceleague.adobe.com/docs/commerce-admin/systems/config-email.html)
- [Email templates](https://experienceleague.adobe.com/docs/commerce-admin/systems/templates/emails.html)
- [Transactional emails](https://developer.adobe.com/commerce/php/development/components/email/)

---

## Common Mistakes to Avoid

1. ❌ Hardcoding sender email → Use store config so it's configurable per store
2. ❌ Not setting template vars before sending → Blank emails or errors
3. ❌ Forgetting HTML vs plain text → Always provide both versions
4. ❌ Not escaping user data in templates → XSS vulnerability
5. ❌ Testing in production → Use development SMTP or disable sending

---

*Magento 2 Backend Developer Course — Topic 20 — Module Packaging*