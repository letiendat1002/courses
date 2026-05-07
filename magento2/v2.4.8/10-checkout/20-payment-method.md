---
title: "20 - Payment Architecture & Payment Method Integration"
description: "Magento 2.4.8 payment architecture: PaymentMethodInterface, payment gateway integration, payment execution flow, vault, and payment troubleshooting."
tags: magento2, payment, payment-method, PaymentMethodInterface, payment-gateway, payment-exception, vault, payment-processing
rank: 20
pathways: [magento2-deep-dive]
see_also:
  - path: "10-checkout/README.md"
    description: "Checkout — where payment integration happens"
  - path: "15-storefront-advanced/README.md"
    description: "Advanced Storefront — payment frontend integration"
---

# Payment Architecture & Payment Method Integration

Payment integration is one of the most critical and complex parts of Magento. Every order requires a payment method, and understanding the payment architecture is essential for adding new payment gateways, handling payment failures, and debugging checkout issues.

---

## 1. Payment Architecture Overview

### The Payment Flow

```
Checkout Shipping Step (complete)
        ↓
Checkout Payment Step
    ├── Display available payment methods (active payment methods)
    ├── Customer selects payment method
    ├── For credit card: collects card details
    └── For other methods: collects method-specific info
        ↓
Place Order
    ├── Validate payment information
    ├── Execute payment (authorize / authorize+capture)
    ├── On success: create order, save payment
    ├── On failure: show error, stay on payment step
    └── On redirect (PayPal, etc.): redirect to external
        ↓
Order confirmation page (or redirect return)
```

### Core Interfaces

| Interface | Location | Purpose |
|-----------|----------|---------|
| `PaymentMethodInterface` | `Magento\Payment\Model\MethodInterface` | Main interface all payment methods implement |
| `CommandPool` | `Magento\Payment\Model\CommandPool` | Holds payment commands (authorize, capture, etc.) |
| `InfoInterface` | `Magento\Payment\Model\InfoInterface` | Stores payment-specific information |
| `ValidatorPool` | `Magento\Payment\Model\ValidatorPool` | Pre/post execution validators |

### Payment Method Base Class

```php
<?php
// vendor/magento/module-payment/Model/MethodInterface.php

interface PaymentMethodInterface
{
    public function authorize(
        \Magento\Payment\Model\InfoInterface $payment,
        float $amount
    ): \Magento\Payment\Model\InfoInterface;

    public function capture(
        \Magento\Payment\Model\InfoInterface $payment,
        float $amount
    ): \Magento\Payment\Model\InfoInterface;

    public function refund(
        \Magento\Payment\Model\InfoInterface $payment,
        float $amount
    ): \Magento\Payment\Model\InfoInterface;

    public function void(
        \Magento\Payment\Model\InfoInterface $payment
    ): \Magento\Payment\Model\InfoInterface;

    public function canAuthorize(): bool;
    public function canCapture(): bool;
    public function canRefund(): bool;
    public function canVoid(): bool;
}
```

---

## 2. PaymentMethodInterface

### Standard Payment Method Class

```php
<?php
// Model/Payment/MyPaymentMethod.php

namespace Vendor\Payment\Model\Payment;

use Magento\Payment\Model\Method\Adapter;
use Magento\Payment\Model\InfoInterface;
use Magento\Framework\DataObject;

class MyPaymentMethod extends Adapter
{
    protected $_code = 'mypayment';

    public function __construct(
        \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig,
        \Magento\Payment\Model\Method\Logger $logger,
        \Magento\Payment\Helper\Data $helper,
        \Magento\Payment\Model\BalanceValidatorFactory $balanceValidatorFactory,
        \Magento\Framework\Event\ManagerInterface $eventManager,
        \Magento\Payment\Model\Method\InstanceFactory $instanceFactory,
        \Magento\Payment\Api\PaymentMethodInterface $paymentData,
        string $code = 'mypayment',
        array $formBlockType = [],
        array $infoBlockType = []
    ) {
        parent::__construct(
            $scopeConfig,
            $logger,
            $helper,
            $balanceValidatorFactory,
            $eventManager,
            $instanceFactory,
            $paymentData,
            $code,
            $formBlockType,
            $infoBlockType
        );
    }

    public function authorize(
        InfoInterface $payment,
        float $amount
    ): InfoInterface {
        // Validate payment is possible
        if (!$this->canAuthorize()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('The authorize action is not available.')
            );
        }

        // Call payment gateway to authorize
        $result = $this->callGatewayAuthorize($payment, $amount);

        // Store transaction ID
        $payment->setTransactionId($result['transaction_id']);

        // Set additional information
        $payment->setAdditionalInformation([
            'authorization_code' => $result['auth_code'],
            'card_type' => $result['card_type']
        ]);

        return $this;
    }

    public function capture(
        InfoInterface $payment,
        float $amount
    ): InfoInterface {
        if (!$this->canCapture()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('The capture action is not available.')
            );
        }

        // If previous authorization exists, capture against it
        if ($payment->getParentTransactionId()) {
            $result = $this->callGatewayCapture(
                $payment,
                $amount,
                $payment->getParentTransactionId()
            );
        } else {
            // Auth and capture in one step
            $result = $this->callGatewayAuthorizeAndCapture($payment, $amount);
        }

        $payment->setTransactionId($result['transaction_id']);

        return $this;
    }
}
```

### Adapter Pattern

Magento's `Adapter` class allows you to use `PaymentMethodInterface` without implementing every method:

```php
<?php
// \Magento\Payment\Model\Method\Adapter extends AbstractMethod

// To implement:
// 1. Set $_code (the payment method code)
// 2. Override specific methods (authorize, capture, etc.)
// 3. Map to gateway commands via commands config
```

---

## 3. Payment Gateway Integration

### Command Pattern

Payment operations (authorize, capture, refund) are executed via "commands":

```php
<?php
// etc/di.xml — wire payment commands
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/di.xsd">

    <type name="Vendor\Payment\Model\Payment\MyPaymentMethod">
        <arguments>
            <argument name="commands" xsi:type="array">
                <item name="authorize" xsi:type="object">Vendor\Payment\Gateway\Command\AuthorizeCommand</item>
                <item name="capture" xsi:type="object">Vendor\Payment\Gateway\Command\CaptureCommand</item>
                <item name="refund" xsi:type="object">Vendor\Payment\Gateway\Command\RefundCommand</item>
            </argument>
        </arguments>
    </type>

</config>
```

### Command Class Implementation

```php
<?php
// Gateway/Command/AuthorizeCommand.php

namespace Vendor\Payment\Gateway\Command;

use Magento\Payment\Gateway\CommandInterface;
use Magento\Payment\Gateway\Data\PaymentDataObjectInterface;
use Magento\Payment\Gateway\Request\BuilderInterface;

class AuthorizeCommand implements CommandInterface
{
    private BuilderInterface $requestBuilder;
    private HandlerInterface $handler;
    private ValidatorInterface $validator;

    public function __construct(
        BuilderInterface $requestBuilder,
        HandlerInterface $handler,
        ValidatorInterface $validator
    ) {
        $this->requestBuilder = $requestBuilder;
        $this->handler = $handler;
        $this->validator = $validator;
    }

    public function execute(array $commandSubject): ResultInterface
    {
        /** @var PaymentDataObjectInterface $paymentDO */
        $paymentDO = $commandSubject['payment'];

        // Build request to payment gateway
        $request = $this->requestBuilder->build($commandSubject);

        // Call payment gateway
        $result = $this->gatewayClient->request($request);

        // Validate response
        $validationResult = $this->validator->validate($result);
        if (!$validationResult->isValid()) {
            throw new \Magento\Payment\Gateway\CommandException(
                $validationResult->getErrorMessage()
            );
        }

        // Handle response (store transaction info)
        $this->handler->handle($commandSubject, $result);

        return $result;
    }
}
```

### Request Builder

```php
<?php
// Gateway/Request/AuthorizeRequestBuilder.php

class AuthorizeRequestBuilder implements BuilderInterface
{
    public function build(array $commandSubject): array
    {
        /** @var PaymentDataObjectInterface $paymentDO */
        $paymentDO = $commandSubject['payment'];
        $payment = $paymentDO->getPayment();

        return [
            'merchantId' => $this->config->getMerchantId(),
            'transactionType' => 'AUTH_ONLY',
            'amount' => $commandSubject['amount'],
            'cardNumber' => $payment->getCcNumber(),
            'cardExpiry' => $payment->getCcExpMonth() . '/' . $payment->getCcExpYear(),
            'cvv' => $payment->getCcCid(),
            'customerId' => $payment->getOrder()->getCustomerId(),
            'orderId' => $payment->getOrder()->getIncrementId(),
        ];
    }
}
```

### Response Handler

```php
<?php
// Gateway/Response/TransactionHandler.php

class TransactionHandler implements HandlerInterface
{
    public function handle(array $handlingSubject, array $response): void
    {
        /** @var PaymentDataObjectInterface $paymentDO */
        $paymentDO = $handlingSubject['payment'];
        $payment = $paymentDO->getPayment();

        $payment->setTransactionId($response['transaction_id']);
        $payment->setIsTransactionClosed(0);  // Open for capture

        // Store additional response data
        $payment->setAdditionalInformation([
            'auth_code' => $response['auth_code'],
            'card_type' => $response['card_type'],
            'gateway_response_code' => $response['response_code']
        ]);
    }
}
```

---

## 4. Payment Vault (Saved Cards)

### Vault Architecture

```php
<?php
// \Magento\Vault\Model\PaymentToken

// A payment token represents a saved card
// Stored in vault tables, linked to customer

class PaymentToken extends \Magento\Framework\Model\AbstractExtensibleModel
{
    public function getPaymentMethodCode(): string;
    public function getCustomerId(): int;
    public function getTokenValue(): string;        // Encrypted card details
    public function getTokenExpirationDate(): ?string;
    public function getIsActive(): bool;
    public function getIsVisible(): bool;

    // Public hash (for display in UI)
    public function getPublicHash(): string;
}
```

### Vault Provider

```php
<?php
// \Magento\Vault\Model\VaultAvailabilityChecker
// Determines if a payment token can be used

class VaultAvailabilityChecker implements VaultAvailabilityCheckerInterface
{
    public function isAvailable(
        PaymentTokenInterface $token,
        int $storeId
    ): bool {
        // Check if token's payment method is still active
        // Check if token hasn't expired
        // Check if customer still has access
    }
}
```

### Saving a Card During Checkout

```php
<?php
// In your payment method's authorize/capture:

public function authorize(InfoInterface $payment, float $amount): InfoInterface
{
    $result = $this->gatewayClient->authorize($payment, $amount);

    // If customer wants to save card
    if ($payment->getAdditionalInformation('save_card')) {
        $this->saveCardToVault($payment, $result);
    }

    return $this;
}

private function saveCardToVault(InfoInterface $payment, array $gatewayResult): void
{
    /** @var \Magento\Vault\Api\PaymentTokenRepositoryInterface $tokenRepository */
    $tokenRepository = $this->paymentTokenRepository;

    /** @var \Magento\Vault\Model\PaymentToken $token */
    $token = $this->paymentTokenFactory->create();

    $token->setPaymentMethodCode($this->_code);
    $token->setCustomerId($payment->getOrder()->getCustomerId());
    $token->setTokenValue($gatewayResult['token']);
    $token->setTokenExpirationDate($gatewayResult['expiry']);
    $token->setIsActive(1);
    $token->setIsVisible(1);

    // Generate public hash for display
    $token->setPublicHash($this->hashGenerator->generate($token));

    $tokenRepository->save($token);
}
```

---

## 5. Payment Troubleshooting

### Common Payment Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Authorization failed` | Declined by gateway | Check gateway response, card details |
| `Invalid transaction` | Wrong transaction ID | Verify capture against correct auth |
| `Gateway timeout` | Gateway unreachable | Implement retry with circuit breaker |
| `CVV declined` | Card CVV wrong | Ask customer to verify |
| `Insufficient funds` | Card balance exceeded | Suggest alternative payment |

### Debugging Payment Flow

```php
<?php
// Enable payment debug logging
// In etc/di.xml:

<logger type="payment" path="Vendor\Payment\Logger\Handler"/>

// In payment method:

public function authorize(InfoInterface $payment, float $amount): InfoInterface
{
    $this->_logger->info('Payment authorize', [
        'amount' => $amount,
        'customer_id' => $payment->getOrder()->getCustomerId(),
        'transaction_id' => $payment->getParentTransactionId()
    ]);

    try {
        $result = $this->gatewayClient->authorize(...);
    } catch (\Exception $e) {
        $this->_logger->error('Payment authorize failed', [
            'error' => $e->getMessage(),
            'trace' => $e->getTraceAsString()
        ]);
        throw $e;
    }

    return $this;
}
```

### Payment Info Storage

```php
<?php
// Payment additional information is stored in:
// sales_order.payment.additional_information (JSON)

$payment->setAdditionalInformation([
    'card_type' => 'VI',  // Visa
    'card_last4' => '1111',
    'authorization_code' => 'ABC123'
]);

// Retrieve
$cardType = $payment->getAdditionalInformation('card_type');

// Save with order
$payment->setAdditionalInformation('save_card', true);
```

---

## 6. Payment Configuration

### Payment Method System Configuration

```xml
<!-- etc/adminhtml/system.xml -->
<section id="payment" translate="label" type="text" showInDefault="1">
    <group id="mypayment" translate="label" type="text">
        <label>My Payment Method</label>
        <field id="active" translate="label" type="select">
            <label>Enabled</label>
            <source_model>Magento\Config\Model\Config\Source\Yesno</source_model>
        </field>
        <field id="title" translate="label" type="text">
            <label>Title</label>
        </field>
        <field id="gateway_url" translate="label" type="text">
            <label>Gateway URL</label>
        </field>
        <field id="merchant_id" translate="label" type="text">
            <label>Merchant ID</label>
            <backend_model>Magento\Config\Model\Config\Backend\Encrypted</backend_model>
        </field>
    </group>
</section>
```

### Payment Configuration Class

```php
<?php
// Model/Config.php

class Config extends \Magento\Payment\Model\Config
{
    protected $_methodCode = 'mypayment';

    public function getGatewayUrl(): string
    {
        return (string) $this->getConfigData('gateway_url');
    }

    public function getMerchantId(): string
    {
        return (string) $this->getConfigData('merchant_id');
    }

    public function getGatewayTimeout(): int
    {
        return (int) $this->getConfigData('gateway_timeout') ?: 30;
    }
}
```

---

## 7. Order State Machine

### Order States and Statuses

```sql
-- Order entity has state and status
-- state: internal Magento state machine state
-- status: customizable label shown to customer

sales_order
├── state          -- 'pending', 'pending_payment', 'processing', 'complete', 'closed', 'canceled'
├── status         -- 'pending', 'pending_payment', 'processing', 'completed', 'closed'
├── base_grand_total
├── grand_total
├── entity_id
└── increment_id
```

### State Transitions

```php
<?php
// vendor/magento/module-sales/Model/Order.php

// Valid state transitions:
// pending → pending_payment, processing, holded, canceled
// pending_payment → processing, canceled
// processing → complete, holded, refund, closed
// complete → pay_reversed, closed, refund
// holded → processing, cancel
// canceled → (terminal)
// closed → (terminal)
// pay_pending → (transitions to various states from payment decisions)
```

### Order Status Configuration

```xml
<!-- etc/adminhtml/di.xml — map status to state -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/di.xsd">
    <type name="Magento\Sales\Model\Order\Status">
        <arguments>
            <argument name="stateToStatusMap" xsi:type="array">
                <item name="pending" xsi:type="array">
                    <item name="pending" xsi:type="string">pending</item>
                </item>
                <item name="processing" xsi:type="array">
                    <item name="processing" xsi:type="string">processing</item>
                    <item name="custom_processing" xsi:type="string">custom_status</item>
                </item>
            </argument>
        </arguments>
    </type>
</config>
```

### Creating Custom Order Status

```xml
<!-- etc/di.xml (module) -->
<config>
    <type name="Magento\Sales\Model\Order\Status">
        <arguments>
            <argument name="statuses" xsi:type="array">
                <item name="custom_processing" xsi:type="string">Custom Processing Status</item>
            </argument>
        </arguments>
    </type>
</config>
```

---

## 8. Payment-Order State Integration

### Payment Result Affects Order State

```php
<?php
// After payment capture, order state transitions automatically

public function capture(InfoInterface $payment, float $amount): InfoInterface
{
    $result = $this->gatewayClient->capture($payment, $amount);

    // If payment successful and order not yet in processing
    if ($result->isSuccessful()) {
        $order = $payment->getOrder();
        if ($order->getState() === Order::STATE_PENDING_PAYMENT) {
            $order->setState(Order::STATE_PROCESSING);
            $order->setStatus('processing');
        }
    }

    return $this;
}
```

### Manual State Transitions

```php
<?php
// In admin or programmatically:
$order = $this->orderFactory->create()->load($orderId);

// Hold order
$order->setState(Order::STATE_HOLDED);
$order->setStatus('holded');
$order->save();

// Unhold order
$order->setState(Order::STATE_PROCESSING);
$order->save();

// Cancel order
$order->cancel();
$order->setState(Order::STATE_CANCELED);

// Close order
$order->setState(Order::STATE_CLOSED);
```

---

## 9. Payment for Different Scenarios

### Zero Subtotal Checkout

```php
<?php
// Payment is typically skipped or marked as 'zero-checkout'
// In PaymentMethod::canAuthorize() / canCapture():

public function canAuthorize(): bool
{
    $total = $this->getInfoInstance()->getOrder()->getGrandTotal();

    if ($total <= 0) {
        return false;  // No payment needed
    }

    return true;
}
```

### Pre-Authorized then Captured Later

```php
<?php
// For payment methods that support deferred capture (e.g., some credit cards)

// Step 1: Authorize only
$payment->authorize($amount);  // Sets transaction_id, is_transaction_closed = 0

// Step 2: Later, capture
$payment->capture($amount);     // Uses parent_transaction_id from auth

// If never captured:
// Order stays in 'holded' or 'pending'
// Cron job may void abandoned pre-authorizations
```

### Refund Flow

```php
<?php
// Refund can be online (via gateway) or offline

public function refund(InfoInterface $payment, float $amount): InfoInterface
{
    if (!$this->canRefund()) {
        throw new \Magento\Framework\Exception\LocalizedException(__('Cannot refund.'));
    }

    // If payment was captured previously
    if ($payment->getParentTransactionId()) {
        $result = $this->gatewayClient->refund(
            $payment,
            $amount,
            $payment->getParentTransactionId()
        );

        $payment->setTransactionId($result['refund_transaction_id']);
    }

    // Set order status back
    $payment->getOrder()->setStatus('closed');

    return $this;
}
```

---

## 10. Common Mistakes to Avoid

1. ❌ Not wrapping gateway calls in try/catch → Unhandled exceptions crash checkout
2. ❌ Storing raw credit card numbers → NEVER store card details; use vault tokens
3. ❌ Not checking `canAuthorize()` before calling authorize → Payment method might not support it
4. ❌ Mixing up transaction IDs → `getTransactionId()` vs `getParentTransactionId()` for capture vs auth
5. ❌ Forgetting to set `is_transaction_closed` → Incomplete order state tracking
6. ❌ Hardcoding gateway URL → Use config so merchant can change environments

---

## Reading List

- [Payment implementation](https://developer.adobe.com/commerce/php/development/payments/)
- [Payment method configuration](https://experienceleague.adobe.com/docs/commerce-admin/stores-guides/core-flows/checkout/configure-payment-methods.html)
- [Payment vault](https://developer.adobe.com/commerce/php/development/payments/vault/)

---

*Magento 2 Backend Developer Course — Topic 10 — Checkout*