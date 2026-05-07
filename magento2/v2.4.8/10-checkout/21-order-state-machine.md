---
title: "21 - Order State Machine & Order Management"
description: "Magento 2.4.8 order state machine: order states, status transitions, hold/cancel/close operations, order creation flow, and programmatic order management."
tags: magento2, order, order-state, state-machine, order-status, order-management, order-creation, order-cancel, order-hold
rank: 21
pathways: [magento2-deep-dive]
see_also:
  - path: "10-checkout/README.md"
    description: "Checkout — where orders are placed"
  - path: "10-checkout/20-payment-method.md"
    description: "Payment — how payment affects order state"
---

# Order State Machine & Order Management

Understanding the order state machine is critical for any backend developer working with Magento. Every order transitions through states — from creation through payment to fulfillment and closure. Misunderstanding this machine leads to bugs in order processing, refunds, and fulfillment automation.

---

## 1. Order State Machine Overview

### Order Entity Core Fields

```sql
sales_order
├── entity_id           -- Internal primary key
├── increment_id        -- Customer-facing order number (e.g., '000000001')
├── state               -- Magento internal state
├── status              -- Custom status (visible to customer)
├── store_id            -- Store order was placed from
├── customer_id         -- NULL for guest
├── customer_email
├── quote_id            -- Link to cart
├── base_grand_total    -- Total in base currency
├── grand_total         -- Total in order currency
├── order_currency_code
├── subtotal            -- Items subtotal
├── shipping_amount     -- Shipping cost
├── tax_amount          -- Tax
├── discount_amount     -- Applied discounts
├── payment             -- Payment method details (JSON)
├── created_at
├── updated_at
└── status_histories   -- Order status history (comments)
```

### The States

| State | Meaning | Can Transition To |
|-------|---------|-------------------|
| `pending` | Order created, no payment | `pending_payment`, `processing`, `holded`, `canceled` |
| `pending_payment` | Awaiting payment (PayPal, etc.) | `processing`, `canceled` |
| `processing` | Payment received, preparing shipment | `complete`, `holded`, `closed` |
| `holded` | Manually put on hold | `processing`, `canceled` |
| `complete` | Fully fulfilled | `pay_pending`, `closed` |
| `closed` | Order closed (refund/cancel) | (terminal) |
| `canceled` | Order canceled | (terminal) |
| `pay_pending` | Payment hold pending | various |

---

## 2. Order Creation Flow

### From Cart to Order

```php
<?php
// \Magento\Checkout\Model\ShippingInformationManagement::saveShippingInformation()
// Called when customer proceeds from shipping to payment step

public function saveShippingInformation(
    $cartId,
    \Magento\Checkout\Api\Data\ShippingInformationInterface $shippingInformation
): int {
    $quote = $this->quoteRepository->get($cartId);

    // 1. Save shipping address
    $quote->setShippingAddress($shippingInformation->getShippingAddress());

    // 2. Save shipping method
    $shippingMethod = $shippingInformation->getShippingMethodCode();
    $quote->setShippingMethod($shippingMethod);

    // 3. Apply shipping description
    $shippingDescription = $this->shippingMethodManagement
        ->getShippingMethod($shippingMethod)
        ->getDescription();
    $quote->setShippingDescription($shippingDescription);

    // 4. Collect totals
    $quote->collectTotals();

    // 5. Save quote
    $this->quoteRepository->save($quote);

    return $quote->getId();
}
```

### Place Order Process

```php
<?php
// \Magento\Checkout\Model\Session::submitOrder()

public function submitOrder()
{
    $quote = $this->getQuote();

    // 1. Create order from quote
    /** @var \Magento\Sales\Api\Data\OrderInterface $order */
    $order = $this->quoteManagement->submit($quote);

    // 2. Handle payment
    $payment = $quote->getPayment();
    if ($payment) {
        $payment->setOrder($order);
        // Payment::place() triggers authorize/capture
        $payment->place();
    }

    // 3. Clear cart
    $quote->setIsActive(false);
    $this->quoteRepository->save($quote);

    // 4. Return order
    return $order;
}
```

### Quote Management Submit

```php
<?php
// \Magento\Quote\Model\QuoteManagement::submit()

public function submit(\Magento\Quote\Model\Quote $quote, array $orderData = []): OrderInterface
{
    // 1. Validate cart (not empty, valid items)
    $this->validateQuote($quote);

    // 2. Convert quote to order
    $order = $this->quoteToOrderConverter->convert($quote, $orderData);

    // 3. Place order
    $order = $this->orderManagement->place($order);

    // 4. Save order
    $this->orderRepository->save($order);

    return $order;
}
```

---

## 3. State Transitions

### Place Order Transitions

```php
<?php
// \Magento\Sales\Model\Order::place()

public function place(): Order
{
    $this->_eventManager->dispatch('sales_order_place_before', ['order' => $this]);

    $this->_placePayment();  // Triggers payment->place()

    $this->setState(Order::STATE_PROCESSING);
    $this->setStatus('processing');

    $this->_eventManager->dispatch('sales_order_place_after', ['order' => $this]);

    return $this;
}
```

### Payment Trigger

```php
<?php
// \Magento\Sales\Model\Order\Payment::place()

public function place(): void
{
    $this->_eventManager->dispatch('sales_order_payment_place_start', ['payment' => $this]);

    // For authorize/capture payment methods:
    // The payment method's authorize() / capture() is called
    // On success: order transitions to 'processing'
    // On failure: exception thrown, order stays in 'pending'

    $this->_eventManager->dispatch('sales_order_payment_place_end', ['payment' => $this]);
}
```

### Manual State Transitions

```php
<?php
// HOLD ORDER
public function hold(): Order
{
    if ($this->canHold()) {
        $this->_eventManager->dispatch('sales_order_status_history_before_save', [
            'order' => $this
        ]);
        $this->setState(Order::STATE_HOLDED);
        $this->setStatus('holded');
        $this->save();
    }
    return $this;
}

// UNHOLD ORDER
public function unhold(): Order
{
    if ($this->canUnhold()) {
        $this->setState(Order::STATE_PROCESSING);
        $this->setStatus('processing');
        $this->save();
    }
    return $this;
}

// CANCEL ORDER
public function cancel(): Order
{
    if ($this->canCancel()) {
        // Restore inventory
        $this->_inventoryManagement->registerProductsDrawn($this);

        // Cancel payment (if authorized)
        $payment = $this->getPayment();
        if ($payment) {
            $payment->cancel();
        }

        $this->setState(Order::STATE_CANCELED);
        $this->setStatus('canceled');
        $this->save();
    }
    return $this;
}

// CLOSE ORDER
public function close(): Order
{
    $this->setState(Order::STATE_CLOSED);
    $this->setStatus('closed');
    $this->save();
    return $this;
}
```

### Transition Conditions

```php
<?php
// \Magento\Sales\Model\Order::canHold()
public function canHold(): bool
{
    return $this->getState() === Order::STATE_PENDING
        || $this->getState() === Order::STATE_PROCESSING;
}

// \Magento\Sales\Model\Order::canUnhold()
public function canUnhold(): bool
{
    return $this->getState() === Order::STATE_HOLDED;
}

// \Magento\Sales\Model\Order::canCancel()
public function canCancel(): bool
{
    // Can cancel if: pending, holded, or not yet invoiced (if invoice exists, cannot cancel)
    $state = $this->getState();
    if ($state === Order::STATE_PENDING || $state === Order::STATE_HOLDED) {
        return true;
    }
    if ($state === Order::STATE_PROCESSING && !$this->hasInvoices()) {
        return true;
    }
    return false;
}
```

---

## 4. Order Status Configuration

### Status vs State

- **State**: Internal Magento state (`pending`, `processing`, `complete`)
- **Status**: Custom label shown to customer (stored in `sales_order_status` table)

```sql
sales_order_status
├── status    -- 'pending', 'processing', 'completed'
└── label     -- 'Pending', 'Processing', 'Completed'

sales_order_status_state
├── status       -- links to sales_order_status.status
├── state        -- 'pending', 'processing', etc.
├── is_default   -- 1 if this is default status for state
└── visible_on_front
```

### Custom Status Example

```php
<?php
// etc/di.xml — add custom status
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/di.xsd">
    <type name="Magento\Sales\Model\Order\Status">
        <arguments>
            <argument name="statuses" xsi:type="array">
                <item name="partially_shipped" xsi:type="string">Partially Shipped</item>
            </argument>
            <argument name="stateToStatusMap" xsi:type="array">
                <item name="processing" xsi:type="array">
                    <item name="partially_shipped" xsi:type="string">partially_shipped</item>
                </item>
            </argument>
        </arguments>
    </type>
</config>
```

### Setting Status on Order

```php
<?php
// Set specific status (not just default for state)
$order->setState(Order::STATE_PROCESSING);
$order->setStatus('partially_shipped');
$order->save();

// Check what statuses are available for a state
$statuses = $this->orderStatusFactory->create()
    ->getStatusesAvailableForState(Order::STATE_PROCESSING);
```

---

## 5. Order Invoicing

### Invoice Creation Flow

```php
<?php
// \Magento\Sales\Model\Order\Invoice

// Invoice is created when you capture payment
// One order can have multiple invoices (partial shipments)

public function create(int $orderId, array $items = []): InvoiceInterface
{
    $order = $this->orderRepository->get($orderId);

    // Create invoice
    /** @var \Magento\Sales\Model\Order\Invoice $invoice */
    $invoice = $this->invoiceFactory->create($order);

    // Add items (or all items if $items is empty)
    if (!empty($items)) {
        foreach ($items as $itemId => $qty) {
            $invoice->addItem($order->getItemById($itemId)->setQty($qty));
        }
    } else {
        foreach ($order->getAllItems() as $item) {
            $invoice->addItem($item->setQty($item->getQtyOrdered()));
        }
    }

    // Calculate totals
    $invoice->collectTotals();

    // Register invoice
    $invoice->register();

    // Save
    $this->invoiceRepository->save($invoice);

    // Notify customer
    $this->notifier->notify($invoice);

    return $invoice;
}
```

### Invoice and State

```php
<?php
// When invoice is saved, order state may change
public function register(): Invoice
{
    // Sets order state to STATE_PROCESSING if not already
    // Order cannot go back to 'pending' after first invoice

    // If all items invoiced (complete), order may transition to 'complete'
    // (depends on configuration and shipment status)

    return $this;
}
```

---

## 6. Order Shipment

### Shipment Creation Flow

```php
<?php
// \Magento\Sales\Model\Order\Shipment

// Created when order items are shipped

public function create(int $orderId, array $items = []): ShipmentInterface
{
    $order = $this->orderRepository->get($orderId);

    /** @var \Magento\Sales\Model\Order\Shipment $shipment */
    $shipment = $this->shipmentFactory->create($order);

    // Add items
    foreach ($items as $itemId => $qty) {
        $shipment->addItem($order->getItemById($itemId)->setQty($qty));
    }

    // Add tracking number (optional)
    if ($trackingNumber) {
        $shipment->addTrackingNumber($trackingNumber);
    }

    $shipment->register();
    $this->shipmentRepository->save($shipment);

    return $shipment;
}
```

### Shipment and State

```php
<?php
// When all items shipped, order state may transition to 'complete'
public function register(): Shipment
{
    // Check if all items are fully shipped
    // If yes, trigger complete
    return $this;
}
```

---

## 7. Order Comments and History

### Add Order Comment

```php
<?php
public function addComment(string $comment, bool $isVisibleOnFront = false): Order
{
    // Create status history entry
    /** @var \Magento\Sales\Model\Order\Status\History $history */
    $history = $this->statusHistoryFactory->create();
    $history->setComment($comment);
    $history->setIsVisibleOnFront($isVisibleOnFront);
    $history->setEntityName('order');
    $history->setStatus($this->getStatus());

    // Add to order
    $this->addStatusHistory($history);

    return $this;
}
```

### Order Comment Examples

```php
<?php
// After payment capture
$order->addComment('Payment authorized. Transaction ID: ' . $transactionId);

// After invoice
$order->addComment('Invoice #INV-001 created');

// After shipment
$order->addComment('Shipment #SHIP-001 created. Tracking: TRACK123');

// Customer notification
$order->addComment('Order shipped via FedEx Ground', true);  // true = visible to customer

// Internal note
$order->addComment('Customer called about order - handled');  // false = admin only
```

---

## 8. Programmatic Order Management

### Create Order for Customer

```php
<?php
public function createOrderForCustomer(
    int $customerId,
    int $productId,
    float $price,
    int $qty = 1
): OrderInterface {
    // 1. Get customer
    $customer = $this->customerRepository->getById($customerId);

    // 2. Create quote
    $quote = $this->quoteFactory->create();
    $quote->setStore($this->storeManager->getStore());
    $quote->setCurrency();
    $quote->assignCustomer($customer);

    // 3. Add product
    $product = $this->productRepository->getById($productId);
    $quote->addProduct($product, $qty);

    // 4. Set payment method (checkmo for simplicity)
    $quote->getPayment()->setMethod('checkmo');

    // 5. Set shipping
    $quote->getShippingAddress()
        ->setStreet('123 Main St')
        ->setCity('Los Angeles')
        ->setRegion('CA')
        ->setPostcode('90001')
        ->setCountryId('US')
        ->setTelephone('555-555-5555');

    $quote->getShippingAddress()->setShippingMethod('flatrate_flatrate');

    // 6. Collect totals
    $quote->collectTotals();

    // 7. Submit
    $order = $this->quoteManagement->submit($quote);

    return $order;
}
```

### Hold Order

```php
<?php
public function holdOrder(int $orderId): void
{
    $order = $this->orderRepository->get($orderId);

    if ($order->canHold()) {
        $order->hold();
        $order->addComment('Order placed on hold by admin');
    } else {
        throw new \Magento\Framework\Exception\LocalizedException(
            __('Order cannot be placed on hold in current state: %1', $order->getState())
        );
    }
}
```

### Cancel Order

```php
<?php
public function cancelOrder(int $orderId): void
{
    $order = $this->orderRepository->get($orderId);

    if ($order->canCancel()) {
        $order->cancel();
        $order->addComment('Order canceled by admin');
        $this->orderRepository->save($order);

        // Restore inventory
        $this->inventoryManagement->registerProductsDrawn($order);
    } else {
        throw new \Magento\Framework\Exception\LocalizedException(
            __('Order cannot be canceled. State: %1, has invoices: %2',
                $order->getState(),
                $order->hasInvoices() ? 'yes' : 'no'
            )
        );
    }
}
```

---

## 9. Order Workflow Events

### Order Lifecycle Events

| Event | When | Common Uses |
|-------|------|-------------|
| `sales_order_place_before` | Before order placed | Modify order data, validate |
| `sales_order_place_after` | After order placed | Send notification, trigger ERP |
| `sales_order_save_before` | Before any order save | Modify data |
| `sales_order_save_after` | After any order save | Logging, custom processing |
| `sales_order_payment_place_start` | Before payment action | Payment preprocessing |
| `sales_order_payment_place_end` | After payment action | Post-payment processing |

### Observer Example

```php
<?php
// etc/events.xml
<event name="sales_order_place_after">
    <observer name="custom_order_observer"
              instance="Vendor\Module\Observer\OrderPlacedAfter"
              method="execute"/>
</event>

// Observer/OrderPlacedAfter.php
class OrderPlacedAfter implements \Magento\Framework\Event\ObserverInterface
{
    public function execute(\Magento\Framework\Event\Observer $observer): void
    {
        /** @var \Magento\Sales\Api\Data\OrderInterface $order */
        $order = $observer->getEvent()->getOrder();

        // Send order data to ERP
        $this->erpClient->sendOrder($order);

        // Create fulfillment task in external system
        $this->fulfillmentService->createTask($order);

        // Add internal note
        $order->addComment('Order synced to ERP system', false);
    }
}
```

---

## 10. Common Pitfalls and Debugging

### Pitfall 1: Trying to Cancel an Invoiced Order

```php
<?php
// Orders with invoices cannot be canceled via cancel()
// Must be closed instead

if ($order->hasInvoices()) {
    // Cannot cancel, must close
    // Or: refund invoices first, then close
}
```

### Pitfall 2: Wrong State Transition

```php
<?php
// Attempting to hold an order in wrong state throws exception
try {
    $order->hold();
} catch (\Magento\Framework\Exception\LocalizedException $e) {
    // "Order cannot be put on hold"
    // Check $order->getState() first
}

// Safe check:
if ($order->canHold()) {
    $order->hold();
}
```

### Pitfall 3: Order in Pending State Forever

```php
<?php
// If payment was never processed (e.g., PayPal redirect abandoned),
// order stays in 'pending_payment' state
// Cron job should eventually cancel these

// Check for stale pending_payment orders:
SELECT * FROM sales_order
WHERE state = 'pending_payment'
AND created_at < DATE_SUB(NOW(), INTERVAL 1 HOUR);

// Cancel via CLI or cron
```

### Debugging State

```php
<?php
// Dump order state information
echo "State: " . $order->getState() . "\n";
echo "Status: " . $order->getStatus() . "\n";
echo "Can hold: " . ($order->canHold() ? 'yes' : 'no') . "\n";
echo "Can cancel: " . ($order->canCancel() ? 'yes' : 'no') . "\n";
echo "Can unhold: " . ($order->canUnhold() ? 'yes' : 'no') . "\n";

// Dump history
foreach ($order->getStatusHistoryCollection() as $history) {
    echo $history->getCreatedAt() . " - " . $history->getStatus() . " - " . $history->getComment() . "\n";
}
```

---

## Reading List

- [Order management](https://experienceleague.adobe.com/docs/commerce-admin/stores-guides/core-flows/order-management/orders.html)
- [Order state machine](https://developer.adobe.com/commerce/php/development/components/order-management/orders/)
- [Order processing workflow](https://experienceleague.adobe.com/docs/commerce-admin/stores-guides/core-flows/order-management/order-workflow.html)

---

## Common Mistakes to Avoid

1. ❌ Trying to cancel an order with invoices → Close it or refund first
2. ❌ Assuming order can always be held → Check `canHold()` first
3. ❌ Forgetting to call `collectTotals()` on quote before submitting → Order totals wrong
4. ❌ Not restoring inventory on cancel → Inventory stays reserved
5. ❌ Hardcoding order state checks → Use `canHold()`, `canCancel()`, etc.
6. ❌ Forgetting `save()` after state change → Change not persisted

---

*Magento 2 Backend Developer Course — Topic 10 — Checkout*