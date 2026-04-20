# Topic 15: Sales & Orders — Backend Lifecycle Mastery

**Philosophy:** Master the end-to-end order lifecycle in Magento 2 by understanding the state machine, the transactional entities behind it, and every programmatic intervention point.

---

## Overview

Every transaction in Magento 2 flows through the Sales module. An **Order** is the central record that tracks a customer's purchase from placement to closure. Between those two poles lives a complex choreography of invoices, shipments, and credit memos—each a first-class entity backed by its own table, state machine, and extension API.

This module dissects that choreography from the inside out. You will learn how Magento models orders and their child entities, how the state machine enforces valid transitions, how each step in the fulfillment pipeline gets triggered programmatically, and how to manipulate orders safely via plugins and observers. By the end, you will be able to intercept, create, and cancel any order-related entity without touching core code.

---

## Prerequisites (from core course)

| Topic | Why It Matters Here |
|------|---------------------|
| Topic 04 — Data Layer (models, resource models, collections) | Orders are EAV models backed by `sales_order` and related tables. You must understand how Magento reads and writes to these tables. |
| Topic 05 — Plugins & Observers | Order manipulation is 90% plugins and observers. You will intercept `OrderRepositoryInterface::save()`, `sales_order_place_after`, etc. |
| Topic 06 — Admin UI (optional but helpful) | Admin integration tests for order management; mass actions are built in the admin context. |
| Topic 07 — REST API (optional but helpful) | Order API endpoints mirror the service layer you will use programmatically. |

---

## Learning Objectives

1. Explain the hierarchy of Sales Order entities (`Order`, `OrderItem`, `OrderAddress`) and the role each plays.
2. Diagram Magento's order state machine, list every valid state transition, and identify which events fire at each transition.
3. Intercept order placement using an observer on `sales_order_place_after` and log structured data.
4. Create invoices programmatically—full and partial—using `InvoiceRepositoryInterface`.
5. Generate invoice PDFs and trigger email notifications from code.
6. Create shipments with tracking information using `ShipmentRepositoryInterface`.
7. Generate credit memos (full, partial, online, and offline) using `CreditmemoRepositoryInterface`.
8. Write plugins on `OrderRepositoryInterface` that enforce business rules at save time.
9. Implement order hold, unhold, and cancel operations, and understand their side effects on inventory.
10. Query and manage orders via the `OrderRepositoryInterface` search criteria API.

---

## By End of Module You Must Prove

Before you consider this module complete, you must demonstrate all six of the following:

- [ ] An observer on `sales_order_place_after` that writes `{increment_id}:{customer_email}` to `var/log/order_placed.log` on every order placed.
- [ ] A controller that accepts an order ID, loads the order, generates a full invoice, and returns the invoice entity ID as JSON.
- [ ] A CLI command `bin/magento training:order:ship <order_id> --tracking="<tracking_number>"` that creates a shipment with tracking and emits shipment email.
- [ ] A CLI command `bin/magento training:order:refund <invoice_id>` that creates a full credit memo and returns the credit memo entity ID.
- [ ] A plugin on `OrderRepositoryInterface::save()` that throws `LocalizedException` if the order grand total exceeds $10,000 and the `approved` extension attribute is absent.
- [ ] An understanding (documented via code comments) of how to build a mass action in adminhtml to cancel multiple orders and restore inventory.

---

## Assessment Criteria

| # | Criterion | Evidence |
|---|-----------|----------|
| 1 | Order entity hierarchy is correctly identified in comments | Code comments on `Order`, `OrderItem`, `OrderAddress` in submitted files |
| 2 | State machine transitions are respected | No code attempts invalid state transitions (e.g., `cancel()` on a `complete` order) |
| 3 | Observers use correct event name and registered in `events.xml` | `events.xml` uses `sales_order_place_after`, not a variation |
| 4 | Invoice, shipment, and credit memo are created via repository interfaces | No direct `new Invoice()` model instantiation; use repositories |
| 5 | Plugin intercepts `beforeSave()` or `afterSave()` on `OrderRepositoryInterface` | Correct class and method targeted |
| 6 | All CLI commands are registered in `di.xml` / `etc/di.xml` and extend `Command` | `configure()` sets `Name` and `Description`; `execute()` is implemented |
| 7 | No core files are modified | All code lives under `app/code/Training/Sales/` |
| 8 | File paths match required structure | Files placed relative to `app/code/Training/Sales/` |

---

## Topics

---

### Topic 1: Order Architecture & State Machine

#### The Sales Entity Hierarchy

Magento's Sales module models the order as a composition of three main entities:

```
Magento\Sales\Model\Order
├── address   → Magento\Sales\Model\Order\Address
├── items     → Magento\Sales\Model\Order\Item[]   (1:N, recursive for bundles)
├── payments  → Magento\Sales\Model\Order\Payment
├── invoices  → Magento\Sales\Model\Order\Invoice[] (1:N)
├── shipments → Magento\Sales\Model\Order\Shipment[] (1:N)
├── creditmemos → Magento\Sales\Model\Order\Creditmemo[] (1:N)
└── statusHistory → Magento\Sales\Model\Order\Status\History[] (1:N)
```

**`Magento\Sales\Model\Order`** is the aggregate root. It is an EAV model backed by the `sales_order` table. Key columns:

| Column | PHP Constant | Description |
|--------|-------------|-------------|
| `entity_id` | `$order->getId()` | Primary key (internal use only) |
| `increment_id` | `$order->getIncrementId()` | Public-facing order number (e.g., `000000145`) |
| `quote_id` | `$order->getQuoteId()` | Reference to the originating `quote_id` |
| `store_id` | `$order->getStoreId()` | Store scope |
| `website_id` | `$order->getWebsiteId()` | Website scope |
| `customer_id` | `$order->getCustomerId()` | Customer entity ID (null for guests) |
| `customer_email` | `$order->getCustomerEmail()` | Customer email address |
| `state` | `$order->getState()` | Internal state machine state |
| `status` | `$order->getStatus()` | Visible status label |
| `grand_total` | `$order->getGrandTotal()` | Final order total |
| `subtotal` | `$order->getSubtotal()` | Item subtotal before tax/shipping |
| `base_currency_code` | `$order->getBaseCurrencyCode()` | Base currency |
| `order_currency_code` | `$order->getOrderCurrencyCode()` | Display currency |
| `created_at` | `$order->getCreatedAt()` | Creation timestamp |
| `updated_at` | `$order->getUpdatedAt()` | Last update timestamp |

**`Magento\Sales\Model\Order\Item`** backs each line item. A configurable product creates one parent item with child items; a bundle creates multiple children. Important fields: `product_id`, `sku`, `name`, `qty_ordered`, `qty_invoiced`, `qty_shipped`, `qty_refunded`, `price`, `row_total`.

**`Magento\Sales\Model\Order\Address`** models both billing and shipping address. Type is distinguished by `$address->getAddressType()` returning `Billing` or `Shipping`. The same class is reused for quote addresses.

#### State vs. Status: The Two-Layer Model

Magento uses two fields to track order lifecycle:

- **`state`** — the machine state. Controls which transitions are valid. Not exposed directly to the customer. Examples: `new`, `pending`, `processing`, `holded`, `payment_review`, `complete`, `closed`, `canceled`.
- **`status`** — a human-readable label tied to a state via `sales_order_status` configuration. Multiple statuses can map to the same state. Example: state `processing` can have statuses `processing`, `payment_expected`, ` Preparing`.

This two-layer design lets merchants customize status labels in the admin panel (Admin → Stores → Settings → Order Status) without affecting the state machine logic.

#### The State Machine

```
      ┌──────────────────────────────────────────────────────┐
      │                    new                                 │
      └──────┬───────────────────────┬───────────────────────┘
             │                       │
             ▼                       ▼
      pending                  payment_review
      (manual pay             (PayPal/3DS
       after order             hold for
       placed)                  review)
             │                       │
             │                       ▼
             │                  ┌─────────┐
             │                  │ holded  │◄──────────────┐
             │                  └────┬────┘               │
             │                       │                    │
             ▼                       ▼                    ▼
      ┌──────────┐            ┌──────────────┐      ┌───────────┐
      │processing│            │   closed    │      │ processing│
      └────┬─────┘            └──────────────┘      └─────┬─────┘
           │                                                 │
           │ (all invoices + all shipments complete)        │
           ▼                                                 ▼
      ┌──────────┐                                    ┌──────────┐
      │ complete │                                    │ canceled │
      └────┬─────┘                                    └──────────┘
           │
           │ (full credit memo issued)
           ▼
      ┌──────────┐
      │  closed  │
      └──────────┘
```

**State transition table:**

| From State | To State | Typical Trigger |
|-----------|----------|-----------------|
| `new` | `pending` | Payment not received at checkout |
| `new` | `processing` | Payment confirmed (any payment method) |
| `new` | `payment_review` | PayPal express, 3DS, etc. |
| `new` | `holded` | Admin manually puts order on hold |
| `new` | `canceled` | Admin or system cancels |
| `pending` | `processing` | Payment confirmed |
| `pending` | `canceled` | Timeout or admin cancel |
| `payment_review` | `holded` | Review triggered; awaiting decision |
| `payment_review` | `processing` | Review passed |
| `payment_review` | `canceled` | Review failed |
| `holded` | `processing` | Admin unholds |
| `holded` | `canceled` | Admin cancels while on hold |
| `processing` | `complete` | All invoices and shipments created |
| `processing` | `closed` | Full credit memo issued |
| `processing` | `canceled` | Partial cancellation (rare, use credit memo) |
| `complete` | `closed` | Refund issued |
| `closed` | `canceled` | Credit memo already issued, final cancel |

**Invalid transitions throw `Magento\Framework\Exception\LocalizedException`.**

**Why does Magento enforce state transitions?**

The state machine exists to ensure data consistency. An order in `complete` state has had all invoices and shipments created. If you could arbitrarily transition it back to `processing`, you would create an inconsistent state where Magento believes all fulfillment has occurred but the order says otherwise. The state machine prevents this class of data corruption.

**What events fire at each transition?**

| Transition | Event(s) Fired |
|-----------|---------------|
| `new` → `processing` | `sales_order_place_after`, `sales_order_save_commit_after` |
| `processing` → `complete` | `sales_order_save_commit_after`, inventory deducible updated |
| `new` → `canceled` | `sales_order_cancel_after`, inventory restored (if configured) |
| `holded` → `processing` | `sales_order_unhold_after` |
| `complete` → `closed` | `sales_order_creditmemo_save_after` |

> **Pro Tip:** The `sales_order_place_after` event only fires on the `new` → `processing` transition during checkout. If an order is created programmatically in `pending` state (e.g., via an API where payment is not yet confirmed), `sales_order_place_after` does NOT fire. For programmatically created orders, use `sales_order_save_commit_after` to react to all saves.

You do not call `$order->setState()` directly. Instead, use the `OrderManagementInterface`:

```php
<?php
// app/code/Training/Sales/Controller/Adminhtml/Order/Cancel.php
<?php
declare(strict_types=1);

namespace Training\Sales\Controller\Adminhtml\Order;

use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Sales\Api\OrderManagementInterface;
use Magento\Sales\Api\Data\OrderInterface;
use Magento\Framework\Controller\ResultFactory;

class Cancel extends HttpPostActionInterface
{
    public function __construct(
        \Magento\Framework\App\Action\Context $context,
        private readonly OrderManagementInterface $orderManagement,
    ) {
        parent::__construct($context);
    }

    public function execute(): \Magento\Framework\Controller\ResultInterface
    {
        $orderId = (int) $this->getRequest()->getParam('order_id');
        try {
            $this->orderManagement->cancel($orderId);
            $this->messageManager->addSuccessMessage(__('Order #%1 has been canceled.', $orderId));
        } catch (\Magento\Framework\Exception\LocalizedException $e) {
            $this->messageManager->addErrorMessage($e->getMessage());
        }

        /** @var \Magento\Framework\Controller\Result\Redirect $result */
        $result = $this->resultFactory->create(ResultFactory::TYPE_REDIRECT);
        $result->setPath('sales/order/index');
        return $result;
    }
}
```

#### The `sales_order_place_after` Event and Order Creation Flow

When a customer completes checkout, the following chain executes:

```
Quote submit (Magento\Quote\Model\QuoteManagement::submit())
  └─► Creates Order via OrderRepositoryInterface::save()
  └─► Fires "sales_order_place_after" event (with order object in event data)
  └─► Fires "sales_order_save_commit_after" (post-commit, DB transaction committed)
```

The `sales_order_place_after` event carries the `order` key in its data array:

```php
<?php
// app/code/Training/Sales/etc/events.xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="sales_order_place_after">
        <observer name="training_sales_order_place_after"
                  instance="Training\Sales\Observer\OrderPlaceAfter" />
    </event>
</config>
```

```php
<?php
// app/code/Training/Sales/Observer/OrderPlaceAfter.php
<?php
declare(strict_types=1);

namespace Training\Sales\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Api\Data\OrderInterface;
use Psr\Log\LoggerInterface;

class OrderPlaceAfter implements ObserverInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {
    }

    public function execute(Observer $observer): void
    {
        /** @var OrderInterface $order */
        $order = $observer->getData('order');
        $this->logger->info('ORDER_PLACED', [
            'increment_id' => $order->getIncrementId(),
            'customer_email' => $order->getCustomerEmail(),
            'grand_total' => $order->getGrandTotal(),
            'state' => $order->getState(),
        ]);
    }
}
```

#### Reading Order Data in a Controller or Observer

Always use the repository interface, never the model directly for loading:

```php
<?php
// app/code/Training/Sales/Controller/Index/Index.php
<?php
declare(strict_types=1);

namespace Training\Sales\Controller\Index;

use Magento\Framework\App\Action\HttpGetActionInterface;
use Magento\Sales\Api\OrderRepositoryInterface;
use Magento\Sales\Api\Data\OrderInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Framework\Controller\ResultFactory;

class Index extends HttpGetActionInterface
{
    public function __construct(
        \Magento\Framework\App\Action\Context $context,
        private readonly OrderRepositoryInterface $orderRepository,
        private readonly SearchCriteriaBuilder $searchCriteriaBuilder,
        private readonly ResultFactory $resultFactory,
    ) {
        parent::__construct($context);
    }

    public function execute(): \Magento\Framework\Controller\ResultInterface
    {
        // Method A: Load by entity_id
        $order = $this->orderRepository->get(123);

        // Method B: Load by increment_id using search criteria
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('increment_id', '000000145')
            ->create();
        $orders = $this->orderRepository->getList($searchCriteria);

        /** @var \Magento\Framework\Controller\Result\Raw $result */
        $result = $this->resultFactory->create(\Magento\Framework\Controller\ResultFactory::TYPE_RAW);
        $result->setContents('Order ID: ' . $order->getId() . PHP_EOL);
        $result->setContents($result->getContents() . 'Increment ID: ' . $order->getIncrementId() . PHP_EOL);
        $result->setContents($result->getContents() . 'Total: ' . $order->getGrandTotal());
        return $result;
    }
}
```

---

### Topic 2: Invoice Generation

#### Invoice vs Order: When Does Magento Generate an Invoice?

Magento does **not** automatically generate an invoice when an order is placed. The invoice is a request for payment—the merchant (or an automated process) explicitly creates it after goods are prepared or services confirmed. The exception is orders placed with the **"capture on shipment"** setting, where Magento generates an invoice automatically at the point of shipment creation.

In the standard flow:
1. Order placed → state `processing` or `new`
2. Merchant creates invoice manually in Admin or programmatically
3. Invoice transitions order toward `complete`

#### Programmatic Invoice Creation

Use `Magento\Sales\Api\InvoiceRepositoryInterface`. Never instantiate `Magento\Sales\Model\Order\Invoice` directly and save it without the repository.

**Full Invoice:**

```php
<?php
// app/code/Training/Sales/Service/InvoiceCreator.php
<?php
declare(strict_types=1);

namespace Training\Sales\Service;

use Magento\Sales\Api\InvoiceRepositoryInterface;
use Magento\Sales\Api\Data\InvoiceInterface;
use Magento\Sales\Api\Data\InvoiceInterfaceFactory;
use Magento\Sales\Api\Data\OrderInterface;
use Magento\Sales\Api\Data\OrderItemInterface;
use Magento\Framework\Exception\LocalizedException;

class InvoiceCreator
{
    public function __construct(
        private readonly InvoiceRepositoryInterface $invoiceRepository,
        private readonly InvoiceInterfaceFactory $invoiceFactory,
    ) {
    }

    /**
     * Creates a full invoice for an order (all items, all quantities).
     *
     * @throws LocalizedException
     */
    public function createFullInvoice(OrderInterface $order): InvoiceInterface
    {
        if ($order->canInvoice()) {
            /** @var InvoiceInterface $invoice */
            $invoice = $this->invoiceFactory->create([
                'data' => [
                    'order_id' => $order->getId(),
                    'invoice_id' => null,
                    'entity_id' => null,
                ],
            ]);

            $invoice->setOrder($order);

            // Add all order items to the invoice
            /** @var OrderItemInterface $orderItem */
            foreach ($order->getItems() as $orderItem) {
                // Skip non-product items (bundle children, etc.) — they are
                // invoiced when their parent is invoiced
                if ($orderItem->getParentItemId()) {
                    continue;
                }
                $qty = (float) $orderItem->getQtyOrdered();
                if ($qty > 0) {
                    $invoice->addItem($orderItem);
                    $invoice->setItemQty($orderItem, $qty);
                }
            }

            $invoice->collectTotals();
            return $this->invoiceRepository->save($invoice);
        }

        throw new LocalizedException(
            __('Order %1 cannot be invoiced (current state: %2)', $order->getIncrementId(), $order->getState())
        );
    }
}
```

**Partial Invoice (specific items and quantities):**

```php
<?php
// Creates a partial invoice for a single item only
public function createPartialInvoiceForItem(OrderInterface $order, int $itemId, float $qty): InvoiceInterface
{
    if (!$order->canInvoice()) {
        throw new LocalizedException(__('Order %1 cannot be invoiced', $order->getIncrementId()));
    }

    $invoice = $this->invoiceFactory->create(['order' => $order]);

    /** @var OrderItemInterface $orderItem */
    foreach ($order->getItems() as $orderItem) {
        if ((int) $orderItem->getId() === $itemId) {
            if ($qty > 0 && $qty <= $orderItem->getQtyOrdered()) {
                $invoice->addItem($orderItem);
                $invoice->setItemQty($orderItem, $qty);
            } else {
                throw new LocalizedException(
                    __('Invalid qty %1 for item %2. Max available: %3', $qty, $itemId, $orderItem->getQtyOrdered())
                );
            }
            break;
        }
    }

    $invoice->collectTotals();
    return $this->invoiceRepository->save($invoice);
}
```

**Registering the invoice so it affects inventory:**

The invoice's `setItemQty()` call registers the quantities. After `$invoiceRepository->save()`, Magento's event system (`sales_order_invoice_save_after`) triggers inventory decrement via the `Magento\CatalogInventory\Observer\ProductToStockItem.php` observer. Do not manually adjust inventory unless you have a specific reason.

#### Invoice PDF Generation

Magento ships with `\Magento\Sales\Model\Order\Pdf\Invoice` which renders a PDF using Zend PDF. The core PDF module is `Magento\Sales\Model\Order\Pdf\Invoice` → `\Magento\Framework\Printify\Renderer\Pdf`.

To generate a PDF and return it as a downloadable file:

```php
<?php
// app/code/Training/Sales/Controller/Index/DownloadInvoicePdf.php
<?php
declare(strict_types=1);

namespace Training\Sales\Controller\Index;

use Magento\Framework\App\Action\HttpGetActionInterface;
use Magento\Sales\Api\InvoiceRepositoryInterface;
use Magento\Sales\Model\Order\Pdf\Invoice as PdfInvoice;
use Magento\Framework\App\Response\Http\FileFactory;

class DownloadInvoicePdf extends HttpGetActionInterface
{
    public function __construct(
        \Magento\Framework\App\Action\Context $context,
        private readonly InvoiceRepositoryInterface $invoiceRepository,
        private readonly PdfInvoice $pdfInvoice,
        private readonly FileFactory $fileFactory,
    ) {
        parent::__construct($context);
    }

    public function execute(): \Magento\Framework\App\ResponseInterface
    {
        $invoiceId = (int) $this->getRequest()->getParam('invoice_id');
        $invoice = $this->invoiceRepository->get($invoiceId);

        $pdf = $this->pdfInvoice->getPdf([$invoice]);
        $pdfContent = $pdf->render();

        return $this->fileFactory->create(
            sprintf('invoice_%s.pdf', $invoice->getIncrementId()),
            $pdfContent,
            'application/pdf'
        );
    }
}
```

#### Email Invoices Automatically on Invoice Save

Invoice emails are sent via the `sales_invoice_save_after` observer wired to `\Magento\Sales\Observer\InvoiceEmailSender`. To trigger email on invoice save:

```php
<?php
// In your custom invoice creation service, after saving:
use Magento\Sales\Observer\InvoiceEmailSender;

public function createAndEmail(OrderInterface $order): InvoiceInterface
{
    $invoice = $this->createFullInvoice($order);

    // The observerInvoiceEmailSender handles email automatically if the
    // "Invoice Email" notification is enabled in Admin → Stores → Config
    // To force-send from code:
    if ($invoice->getEmailSent()) {
        // Already sent
    } else {
        /** @var InvoiceEmailSender $emailSender */
        $emailSender = $this->_objectManager->get(InvoiceEmailSender::class);
        $emailSender->send($invoice);
    }

    return $invoice;
}
```

#### Invoice States

> **Pro Tip:** Understanding the difference between **authorize** and **capture** is critical for payment integration:

| Payment Flow | Description | When Invoice is Created |
|-------------|-------------|----------------------|
| **Authorize Only** | Payment is reserved but not captured (funds held on card) | Created manually after goods are shipped |
| **Capture on Order** | Full amount is authorized and captured immediately at checkout | Created automatically at checkout |
| **Capture on Shipment** | Authorized at checkout, captured when shipment is created | Created at shipment time |

The payment method in Magento controls which flow is used. Check the payment method's `capture()` method to understand exactly when funds are moved. Common payment methods:
- **Braintree**: Supports all three flows via configuration
- **PayPal Express**: Authorizes at checkout, captures on invoice creation
- **Check/Money Order**: No automatic capture (offline)
- **Cash on Delivery**: No automatic capture (offline)

For authorize-only flows, an invoice in `STATE_OPEN` represents the authorization hold. Capturing converts it to `STATE_PAID`. If the order is canceled before capture, the authorization must be released via `void()`.

#### Tax Considerations in Orders

Tax calculation in Magento is complex because it depends on the shipping address, billing address, and product tax class at the time of order placement. Key points:

**1. Tax is calculated at order placement, not at invoice time**

Once an order is placed with a tax amount, that amount is fixed in `sales_order.tax_amount`. Subsequent tax rate changes do not affect existing orders.

**2. Tax fields on the order entity:**

| Field | Description |
|-------|-------------|
| `tax_amount` | Total tax for the order |
| `base_tax_amount` | Tax in base currency |
| `shipping_tax_amount` | Tax on shipping |
| `tax_refunded` | Total tax refunded via credit memos |
| `order_id` | Links to the order |

**3. Tax on partial invoices:**

When creating a partial invoice, tax must be distributed proportionally across invoice items:

```php
<?php
// WRONG — manually setting tax
$invoice->setTaxAmount($invoice->getOrder()->getTaxAmount());
$invoice->setBaseTaxAmount($invoice->getOrder()->getBaseTaxAmount());

// CORRECT — calculate proportional tax per item
$order = $invoice->getOrder();
$orderTaxRate = $order->getTaxAmount() / $order->getSubtotal();

foreach ($invoice->getItems() as $item) {
    $itemTax = $item->getRowTotal() * $orderTaxRate;
    $item->setTaxAmount($itemTax);
    $item->setBaseTaxAmount($itemTax);
}

$invoice->collectTotals();
```

Setting tax manually on a partial invoice will cause rounding discrepancies and audit failures.

**4. Shipping tax:**

Shipping tax is calculated separately. If you refund shipping, the shipping tax must be included:

```php
<?php
if ($order->getShippingAmount() > 0) {
    $creditmemo->setShippingAmount($order->getShippingAmount());
    $creditmemo->setBaseShippingAmount($order->getBaseShippingAmount());
    // Tax on shipping is stored separately — include it in refund
    $creditmemo->setShippingTaxAmount($order->getShippingTaxAmount());
    $creditmemo->setBaseShippingTaxAmount($order->getBaseShippingTaxAmount());
}
```

> **Common Pitfall:** Failing to include shipping tax in a refund. Customers who paid tax on shipping are entitled to that tax being refunded when shipping is refunded.

---

### Topic 2: Invoice Generation

| State Constant | Meaning | Transitions To |
|---------------|---------|----------------|
| `Magento\Sales\Model\Order\Invoice::STATE_OPEN` (= 2) | Invoice created, awaiting payment | `STATE_PAID`, `STATE_CANCELED` |
| `Magento\Sales\Model\Order\Invoice::STATE_PAID` (= 3) | Payment captured | — (terminal) |
| `Magento\Sales\Model\Order\Invoice::STATE_CANCELED` (= 1) | Invoice canceled | — (terminal) |

In practice, most payment methods immediately move invoices to `STATE_PAID`. Check the payment method's `capture()` implementation to understand the exact flow.

---

### Topic 3: Shipment Creation

#### Shipment Creation Flow

A shipment represents the physical (or digital) goods leaving the warehouse. Like invoices, Magento does not auto-create shipments on order placement—merchant or integration must create them explicitly (or they are created automatically by shipping integration modules like MSI).

Use `Magento\Sales\Api\ShipmentRepositoryInterface`:

```php
<?php
// app/code/Training/Sales/Service/ShipmentCreator.php
<?php
declare(strict_types=1);

namespace Training\Sales\Service;

use Magento\Sales\Api\ShipmentRepositoryInterface;
use Magento\Sales\Api\Data\ShipmentInterface;
use Magento\Sales\Api\Data\ShipmentInterfaceFactory;
use Magento\Sales\Api\Data\OrderInterface;
use Magento\Sales\Api\Data\OrderItemInterface;
use Magento\Framework\Exception\LocalizedException;

class ShipmentCreator
{
    public function __construct(
        private readonly ShipmentRepositoryInterface $shipmentRepository,
        private readonly ShipmentInterfaceFactory $shipmentFactory,
    ) {
    }

    /**
     * Creates a full shipment for an order (all items, all quantities).
     */
    public function createFullShipment(OrderInterface $order): ShipmentInterface
    {
        if (!$order->canShip()) {
            throw new LocalizedException(
                __('Order %1 cannot be shipped (state: %2)', $order->getIncrementId(), $order->getState())
            );
        }

        /** @var ShipmentInterface $shipment */
        $shipment = $this->shipmentFactory->create(['order' => $order]);

        /** @var OrderItemInterface $item */
        foreach ($order->getItems() as $item) {
            if ($item->getParentItemId()) {
                continue;
            }
            $qty = (float) $item->getQtyOrdered();
            if ($qty > 0) {
                $shipment->addItem($item);
                $shipment->setItemQty($item, $qty);
            }
        }

        $shipment->collectTotals();
        return $this->shipmentRepository->save($shipment);
    }
}
```

**Partial Shipment (specific item):**

```php
public function createPartialShipmentForItem(OrderInterface $order, int $itemId, float $qty): ShipmentInterface
{
    if (!$order->canShip()) {
        throw new LocalizedException(__('Order %1 cannot be shipped', $order->getIncrementId()));
    }

    $shipment = $this->shipmentFactory->create(['order' => $order]);

    foreach ($order->getItems() as $item) {
        if ((int) $item->getId() === $itemId) {
            $shipment->addItem($item);
            $shipment->setItemQty($item, $qty);
            break;
        }
    }

    $shipment->collectTotals();
    return $this->shipmentRepository->save($shipment);
}
```

#### Tracking Information

Tracking is modeled as `\Magento\Sales\Model\Order\Shipment\Track`. Each track record belongs to a shipment and carries carrier code + tracking number. Magento ships with built-in carriers: `ups`, `fedex`, `dhl`, `usps`, `flatrate`, `freeshipping`.

```php
<?php
// Adding tracking to a shipment after creation
use Magento\Sales\Api\Data\ShipmentTrackInterface;
use Magento\Sales\Api\Data\ShipmentTrackInterfaceFactory;
use Magento\Sales\Api\ShipmentRepositoryInterface;

public function addTracking(ShipmentInterface $shipment, string $carrierCode, string $trackingNumber): void
{
    /** @var ShipmentTrackInterface $track */
    $track = $this->trackFactory->create();
    $track->setShipment($shipment);
    $track->setCarrierCode($carrierCode);        // e.g., 'ups'
    $track->setTitle($this->getCarrierTitle($carrierCode)); // e.g., 'United Parcel Service'
    $track->setTrackNumber($trackingNumber);     // e.g., '1Z999AA10123456784'

    $shipment->getTracks()->addItem($track);
    // Persist via repository to trigger observer events
    $this->shipmentRepository->save($shipment);
}

private function getCarrierTitle(string $code): string
{
    return match ($code) {
        'ups' => 'United Parcel Service',
        'fedex' => 'FedEx',
        'dhl' => 'DHL Express',
        'usps' => 'United States Postal Service',
        default => ucfirst($code),
    };
}
```

**Built-in carrier codes (must be enabled in Admin → Stores → Shipping Methods):**

| Code | Carrier |
|------|---------|
| `ups` | UPS |
| `fedex` | FedEx |
| `dhl` | DHL |
| `usps` | USPS |
| `flatrate` | Flat Rate |
| `freeshipping` | Free Shipping |
| `custom` | Custom Carrier (use `custom` for arbitrary carrier names) |

**Custom carrier title:** If using `custom` as the carrier code, set the title field to your carrier name. You can also create a custom carrier by implementing `\Magento\Shipping\Model\Carrier\CarrierInterface`.

#### The `sales_order_shipment_track_save_after` Observer Pattern

The `sales_order_shipment_track_save_after` event fires after each track record is persisted. Use it to notify external systems (3PL, WMS) or update custom state:

```xml
<!-- app/code/Training/Sales/etc/events.xml -->
<event name="sales_order_shipment_track_save_after">
    <observer name="training_sales_tracking_added"
              instance="Training\Sales\Observer\ShipmentTrackSaved" />
</event>
```

```php
<?php
// app/code/Training/Sales/Observer/ShipmentTrackSaved.php
<?php
declare(strict_types=1);

namespace Training\Sales\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Api\Data\ShipmentTrackInterface;
use Psr\Log\LoggerInterface;

class ShipmentTrackSaved implements ObserverInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {
    }

    public function execute(Observer $observer): void
    {
        /** @var ShipmentTrackInterface $track */
        $track = $observer->getData('track');
        $this->logger->info('TRACKING_ADDED', [
            'track_id' => $track->getId(),
            'carrier_code' => $track->getCarrierCode(),
            'track_number' => $track->getTrackNumber(),
            'shipment_id' => $track->getShipmentId(),
        ]);
    }
}
```

#### Shipment Management — Beyond Basic Creation

**Shipment state tracking:**

Like orders, shipments have their own state tracking. However, shipments are simpler — they are typically in one state:

| State | Constant | Meaning |
|-------|----------|---------|
| `STATE_NEW` | 1 | Shipment created, label may not be generated |
| `STATE_PENDING` | 2 | Waiting for carrier pickup |
| `STATE_LABEL_PRINTED` | 3 | Shipping label generated |
| `STATE_SHIPPED` | 4 | In transit (carrier confirmed) |
| `STATE_DELIVERED` | 5 | Delivered to customer |
| `STATE_CANCELED` | 6 | Shipment canceled |

**Partially shipped orders — splitting shipments:**

When an order has multiple items or multiple addresses (multi-address checkout), you may need to create multiple shipments:

```php
<?php
// Create separate shipments per warehouse or per item group
public function createPartialShipmentByItems(
    OrderInterface $order,
    array $itemsByWarehouse // e.g., ['warehouse_a' => [item_id => qty], ...]
): array {
    $shipments = [];

    foreach ($itemsByWarehouse as $warehouseCode => $items) {
        $shipment = $this->shipmentFactory->create(['order' => $order]);

        foreach ($items as $itemId => $qty) {
            $orderItem = $order->getItemById($itemId);
            $shipment->addItem($orderItem);
            $shipment->setItemQty($orderItem, $qty);
        }

        $shipment->setShippingLabel($this->generateLabel($warehouseCode, $shipment));
        $shipments[] = $this->shipmentRepository->save($shipment);
    }

    return $shipments;
}
```

**Deleting a shipment (before invoice):**

If you created a shipment in error before an invoice was created, you can delete it:

```php
<?php
// Only possible if no invoice exists for these items
$shipment = $this->shipmentRepository->get($shipmentId);
if (!$shipment->getOrder()->hasInvoices()) {
    $this->shipmentRepository->delete($shipment);
}
```

> **Common Pitfall:** Once a shipment has been invoiced, it cannot be deleted. The invoice locks the order's financial state. Cancel the invoice first (if offline) or issue a credit memo for the shipped items.

**Tracking information from external APIs:**

If you use a shipping carrier's API to generate tracking, the observer pattern is critical:

```php
<?php
// etc/events.xml
<event name="sales_order_shipment_save_after">
    <observer name="training_sales_fetch_tracking"
              instance="Training\Sales\Observer\FetchTrackingFromCarrier"/>
</event>
```

```php
<?php
class FetchTrackingFromCarrier implements ObserverInterface
{
    public function execute(Observer $observer): void
    {
        /** @var ShipmentInterface $shipment */
        $shipment = $observer->getData('shipment');

        // Call carrier API (FedEx, UPS, etc.) with the shipment's destination address
        $trackingData = $this->carrierApi->getTracking($shipment->getShippingAddress());

        // Add tracking to the shipment
        foreach ($trackingData['tracks'] as $trackData) {
            $this->addTrackingToShipment($shipment, $trackData);
        }
    }
}
```

#### Shipment Email Notifications

Shipment emails are sent via `\Magento\Sales\Observer\ShipmentEmailSender`. It is triggered by the `sales_order_shipment_save_after` observer if the "Shipment Email" notification is enabled. To send manually:

```php
<?php
// After creating the shipment:
use Magento\Sales\Observer\ShipmentEmailSender;

$shipment = $this->shipmentCreator->createFullShipment($order);
// ...
$emailSender = $this->_objectManager->get(ShipmentEmailSender::class);
$emailSender->send($shipment);
```

---

### Topic 4: Credit Memo / Refund

#### The Credit Memo Concept

A **credit memo** ("credit memo" = memorandum = "note to self") is Magento's way of recording the intent to refund before actually processing the refund. Think of it like writing a check—you create the record, then the payment processor cashes it. This two-phase approach (memo → refund) gives merchants an audit trail and the ability to adjust amounts before committing.

**Why two-phase?** Because in many jurisdictions, a merchant must be able to prove the refund was calculated correctly before executing it. The credit memo is that proof. If a dispute arises, the credit memo is the document that shows exactly what was refunded and why.

#### Online Refund vs Offline Refund

| Type | Mechanism | Use When |
|------|-----------|---------|
| **Online refund** | Calls the original payment method's refund API (PayPal, Stripe, Braintree, etc.) | Payment has been captured and should be returned to the original payment instrument |
| **Offline refund** | Magento records the credit memo but does NOT call the payment gateway | Cash on delivery, check, or store credit; or when the merchant handles refund manually |
| **Store credit** | Magento credits the customer's reward/balance balance | Customer wants store credit instead of reversing the original payment |

```php
<?php
// app/code/Training/Sales/Service/CreditmemoCreator.php
<?php
declare(strict_types=1);

namespace Training\Sales\Service;

use Magento\Sales\Api\CreditmemoRepositoryInterface;
use Magento\Sales\Api\Data\CreditmemoInterface;
use Magento\Sales\Api\Data\CreditmemoInterfaceFactory;
use Magento\Sales\Api\Data\CreditmemoItemInterface;
use Magento\Sales\Api\Data\InvoiceInterface;
use Magento\Sales\Api\Data\OrderInterface;
use Magento\Sales\Api\Data\OrderItemInterface;
use Magento\Sales\Model\Order\Creditmemo;
use Magento\Framework\Exception\LocalizedException;

class CreditmemoCreator
{
    public function __construct(
        private readonly CreditmemoRepositoryInterface $creditmemoRepository,
        private readonly CreditmemoInterfaceFactory $creditmemoFactory,
    ) {
    }

    /**
     * Creates a full offline credit memo for an invoice.
     * "Offline" here means the payment gateway is NOT called.
     */
    public function createFullOfflineRefund(InvoiceInterface $invoice): CreditmemoInterface
    {
        $order = $invoice->getOrder();

        if (!$order->canCreditmemo()) {
            throw new LocalizedException(
                __('Order %1 cannot receive a credit memo (state: %2)', $order->getIncrementId(), $order->getState())
            );
        }

        /** @var CreditmemoInterface $creditmemo */
        $creditmemo = $this->creditmemoFactory->create(['order' => $order]);
        $creditmemo->setInvoice($invoice);

        // Add all items (full refund = all items, all qty)
        /** @var OrderItemInterface $item */
        foreach ($order->getItems() as $item) {
            if ($item->getParentItemId()) {
                continue;
            }

            $qty = (float) $item->getQtyOrdered();

            /** @var CreditmemoItemInterface $memoItem */
            $memoItem = $creditmemo->addItem($item);
            $memoItem->setQty($qty);
            $memoItem->setRowTotal($item->getRowTotal());
        }

        // Shipping (if shipping was charged and should be refunded)
        if ($order->getShippingAmount() > 0) {
            $creditmemo->setShippingAmount($order->getShippingAmount());
            $creditmemo->setBaseShippingAmount($order->getBaseShippingAmount());
        }

        // Tax
        if ($order->getTaxAmount() > 0) {
            $creditmemo->setTaxAmount($order->getTaxAmount());
            $creditmemo->setBaseTaxAmount($order->getBaseTaxAmount());
        }

        // Adjustment (positive = extra refund; negative = deduction)
        $creditmemo->setAdjustment(0.00);
        $creditmemo->setBaseAdjustment(0.00);

        $creditmemo->collectTotals();
        return $this->creditmemoRepository->save($creditmemo);
    }
}
```

**Partial Refund per Item:**

```php
public function createPartialItemRefund(InvoiceInterface $invoice, int $orderItemId, float $qty): CreditmemoInterface
{
    $order = $invoice->getOrder();
    $creditmemo = $this->creditmemoFactory->create(['order' => $order]);
    $creditmemo->setInvoice($invoice);

    foreach ($order->getItems() as $item) {
        if ((int) $item->getId() === $orderItemId) {
            $creditmemoItem = $creditmemo->addItem($item);
            $creditmemoItem->setQty($qty);
            break;
        }
    }

    $creditmemo->collectTotals();
    return $this->creditmemoRepository->save($creditmemo);
}
```

**Refund with Adjustment:**

Adjustments let you add a miscellaneous amount to the refund:

```php
// Add $5.00 goodwill credit on top of item refund
$creditmemo->setAdjustment(5.00);        // Positive: increase refund
$creditmemo->setBaseAdjustment(5.00);
$creditmemo->collectTotals();

// Or deduct $2.00 for a damaged item
$creditmemo->setAdjustment(-2.00);
$creditmemo->setBaseAdjustment(-2.00);
$creditmemo->collectTotals();
```

The `adjustment` is a separate line item in the credit memo and does not affect individual item rows.

#### Refund Amount Distribution

When a full credit memo is created, the totals are distributed as follows:

| Component | Source | Refundable |
|-----------|--------|-----------|
| Items | `OrderItem::getRowTotal()` for each item | Yes, per item or in full |
| Shipping | `Order::getShippingAmount()` | Yes, if shipping was charged |
| Tax | `Order::getTaxAmount()` | Yes, proportional to item refund |
| Adjustment (+) | Manually set | Yes, miscellaneous addition |
| Adjustment (-) | Manually set | Yes, miscellaneous deduction |
| Discount | `Order::getDiscountAmount()` | Yes, proportionally distributed |

#### The `sales_order_creditmemo_save_after` Observer Pattern

```xml
<!-- app/code/Training/Sales/etc/events.xml -->
<event name="sales_order_creditmemo_save_after">
    <observer name="training_sales_creditmemo_saved"
              instance="Training\Sales\Observer\CreditmemoSaved" />
</event>
```

```php
<?php
// app/code/Training/Sales/Observer/CreditmemoSaved.php
<?php
declare(strict_types=1);

namespace Training\Sales\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Api\Data\CreditmemoInterface;
use Psr\Log\LoggerInterface;

class CreditmemoSaved implements ObserverInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {
    }

    public function execute(Observer $observer): void
    {
        /** @var CreditmemoInterface $creditmemo */
        $creditmemo = $observer->getData('creditmemo');
        $this->logger->info('CREDITMEMO_ISSUED', [
            'creditmemo_id' => $creditmemo->getId(),
            'increment_id' => $creditmemo->getIncrementId(),
            'order_id' => $creditmemo->getOrderId(),
            'grand_total' => $creditmemo->getGrandTotal(),
            'base_grand_total' => $creditmemo->getBaseGrandTotal(),
        ]);
    }
}
```

#### Restocking Inventory on Refund

By default, when a credit memo is created, Magento **does not automatically restock inventory**. This behavior is configured in Admin → Stores → Configuration → Catalog → Inventory → Product Stock Options → "Credit Memo Stock" (default: **No**).

To restock programmatically, you must explicitly update the stock items:

```php
<?php
// app/code/Training/Sales/Observer/CreditmemoSaved.php
<?php
declare(strict_types=1);

namespace Training\Sales\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Api\Data\CreditmemoInterface;
use Magento\InventorySalesAdminUi\Model\GetSkuFromCreditmemoItem;
use Magento\CatalogInventory\Api\StockRegistryInterface;

class CreditmemoSaved implements ObserverInterface
{
    public function __construct(
        private readonly StockRegistryInterface $stockRegistry,
    ) {
    }

    public function execute(Observer $observer): void
    {
        /** @var CreditmemoInterface $creditmemo */
        $creditmemo = $observer->getData('creditmemo');

        foreach ($creditmemo->getItems() as $item) {
            $productSku = $item->getSku();
            $qty = (float) $item->getQty();

            $stockItem = $this->stockRegistry->getStockItemBySku($productSku);
            $stockItem->setQty($stockItem->getQty() + $qty);
            $stockItem->setIsInStock(true);
            $this->stockRegistry->updateStockItemBySku($productSku, $stockItem);
        }
    }
}
```

> **Note:** This uses the older `CatalogInventory` API. If you have MSI (Inventory Management) installed, use `Magento\InventorySales\Model\ReturnProcessor` or the `SalesOrderCreditmemoSaveAfter` observer with MSI-aware stock management. The MSI path is more complex and typically handled by `Magento\Inventory\Model\ResourceModel\ReturnToStockQuantity` — check the official MSI documentation for exact implementation.

#### Refund Workflow Deep Dive — The Full Refund Lifecycle

Understanding the complete refund lifecycle is critical for implementing correct business logic. Here is the state machine for refunds:

```
Order (complete/closed)
       │
       ▼
  Invoice (paid)
       │
       ▼
 Credit Memo Request  ──► Offline Refund (cash/check/store credit)
       │                      │
       │                      ▼
       │                Credit Memo (STATE_REFUNDED)
       │                      │
       │                      ▼
       │                Order state → "closed" (if full refund)
       │
       ▼
 Online Refund (payment gateway API)
       │
       ▼
 Payment gateway processes refund
       │
       ├─► Refund succeeds → Credit Memo (STATE_REFUNDED) + order updated
       └─► Refund fails → Exception, credit memo not created
```

**The refund amount must never exceed the invoiced amount.** This is enforced by Magento but you should understand the boundary conditions:

| Scenario | Max Refundable | Notes |
|----------|---------------|-------|
| Full refund | `invoice.grand_total` | All items + shipping + tax |
| Partial item refund | `item.qty_invoiced * item price` | Cannot refund more than invoiced |
| Shipping refund | `invoice.shipping_amount` | Cannot exceed invoiced shipping |
| Multiple partial refunds | Sum cannot exceed invoice total | Track `qty_refunded` per item |

**Preventing double-refund:**

```php
<?php
public function createPartialRefund(OrderInterface $order, int $itemId, float $qty): CreditmemoInterface
{
    $orderItem = $order->getItemById($itemId);
    $maxRefundQty = $orderItem->getQtyInvoiced() - $orderItem->getQtyRefunded();

    if ($qty > $maxRefundQty) {
        throw new LocalizedException(
            __('Cannot refund %1 items. %2 items already refunded.',
                $qty, $orderItem->getQtyRefunded())
        );
    }
    // ... create credit memo
}
```

> **Common Pitfall:** Refunding an item that was already refunded. Always check `qty_refunded` before creating a new credit memo item.

**The `sales_order_creditmemo_save_commit_after` event** fires AFTER the credit memo is committed to the database (within a transaction). Use this for:
- Sending refund confirmation emails
- Updating external systems (ERP, accounting)
- Recording the refund in a custom audit table

Use `sales_order_creditmemo_save_after` (before commit) for operations that should not block the refund creation.

---

### Topic 5: Order Manipulation Plugins & Observers

#### Plugin on `OrderRepositoryInterface::save()` to Enforce Business Rules

Magento's service layer is the correct place to enforce business rules. A plugin on `OrderRepositoryInterface::save()` intercepts every order save operation, including those triggered from Admin, API, and the frontend.

**Example: Require approval for high-value orders ($10,000+) via extension attributes:**

First, declare the extension attribute in `etc/extension_attributes.xml`:

```xml
<!-- app/code/Training/Sales/etc/extension_attributes.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Config/etc/extension_attributes.xsd">
    <extension_attributes for="Magento\Sales\Api\Data\OrderInterface"
                         attribute="approved"
                         type="boolean" />
</config>
```

Then write the plugin:

```php
<?php
// app/code/Training/Sales/Plugin/OrderRepositoryPlugin.php
<?php
declare(strict_types=1);

namespace Training\Sales\Plugin;

use Magento\Sales\Api\Data\OrderInterface;
use Magento\Sales\Api\OrderRepositoryInterface;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Api\ExtensionAttributes\ExtensionAttributesInterface;

class OrderRepositoryPlugin
{
    /**
     * beforeSave plugin — runs before every order save.
     * Use beforeSave to validate data before it is written to the DB.
     */
    public function beforeSave(
        OrderRepositoryInterface $subject,
        OrderInterface $order,
    ): void {
        $total = (float) $order->getGrandTotal();
        $threshold = 10000.00;

        if ($total > $threshold) {
            $approved = $this->getApprovedFlag($order);
            if (!$approved) {
                throw new LocalizedException(
                    __('Orders over $%1 require an "approved" flag in extension attributes. ' .
                        'Please contact your manager to approve order %2.',
                        $threshold,
                        $order->getIncrementId())
                );
            }
        }
    }

    /**
     * afterSave plugin — runs after the order is saved to the DB.
     * Use for logging, notifications, or triggering downstream actions.
     */
    public function afterSave(
        OrderRepositoryInterface $subject,
        OrderInterface $order,
    ): OrderInterface {
        // Log the save event for audit purposes
        // (in production, inject LoggerInterface and avoid logging sensitive order data)
        return $order;
    }

    /**
     * Retrieves the 'approved' boolean from the order's extension attributes.
     */
    private function getApprovedFlag(OrderInterface $order): bool
    {
        $extensionAttributes = $order->getExtensionAttributes();
        if (!$extensionAttributes instanceof ExtensionAttributesInterface) {
            return false;
        }

        $attribute = $extensionAttributes->getApproved();
        return (bool) $attribute;
    }
}
```

Register the plugin in `etc/di.xml`:

```xml
<!-- app/code/Training/Sales/etc/di.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Sales\Api\OrderRepositoryInterface">
        <plugin name="Training\Sales\OrderRepositoryPlugin"
                type="Training\Sales\Plugin\OrderRepositoryPlugin"
                sortOrder="10" />
    </type>
</config>
```

#### `sales_order_place_before` Observer for Fraud Detection (Stub)

The `sales_order_place_before` event fires before the order is committed to the database. This is the right place for fraud checks because you can still block order creation by throwing an exception:

```xml
<!-- app/code/Training/Sales/etc/events.xml -->
<event name="sales_order_place_before">
    <observer name="training_sales_fraud_check"
              instance="Training\Sales\Observer\FraudCheck" />
</event>
```

```php
<?php
// app/code/Training/Sales/Observer/FraudCheck.php
<?php
declare(strict_types=1);

namespace Training\Sales\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Framework\Exception\LocalizedException;
use Magento\Sales\Api\Data\OrderInterface;
use Psr\Log\LoggerInterface;

class FraudCheck implements ObserverInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {
    }

    public function execute(Observer $observer): void
    {
        /** @var OrderInterface $order */
        $order = $observer->getData('order');

        $customerEmail = $order->getCustomerEmail();

        // STUB: Example fraud signals — replace with real integration
        $blockedDomains = ['tempmail.com', 'throwaway.net'];

        foreach ($blockedDomains as $domain) {
            if (str_ends_with($customerEmail, '@' . $domain)) {
                $this->logger->warning('FRAUD_SUSPECTED', [
                    'increment_id' => $order->getIncrementId(),
                    'email' => $customerEmail,
                    'reason' => 'Blocked email domain',
                ]);

                // Put the order on hold for manual review
                // Note: We cannot cancel here because the order doesn't exist yet.
                // Putting it on hold after save (via plugin/afterSave) is the right approach.
                // For now, just log it — the plugin on OrderRepository can catch this.
                break;
            }
        }

        // Example: Flag high-value guest orders for review
        if (!$order->getCustomerId() && $order->getGrandTotal() > 500) {
            $this->logger->info('HIGH_VALUE_GUEST_ORDER', [
                'increment_id' => $order->getIncrementId(),
                'total' => $order->getGrandTotal(),
            ]);
        }
    }
}
```

#### Cancelled Orders: `orderCancel` Service, Inventory Restoration

Use `Magento\Sales\Api\OrderManagementInterface::cancel($orderId)`:

```php
<?php
// app/code/Training/Sales/Service/OrderCanceller.php
<?php
declare(strict_types=1);

namespace Training\Sales\Service;

use Magento\Sales\Api\OrderManagementInterface;
use Magento\Sales\Api\Data\OrderInterface;
use Magento\Sales\Api\OrderRepositoryInterface;
use Magento\Framework\Exception\LocalizedException;

class OrderCanceller
{
    public function __construct(
        private readonly OrderManagementInterface $orderManagement,
        private readonly OrderRepositoryInterface $orderRepository,
    ) {
    }

    /**
     * Cancels an order and restores its inventory.
     *
     * Cancellation side effects:
     *  1. Order state → 'canceled'
     *  2. Status → the status mapped to 'canceled' state
     *  3. Inventory is NOT automatically restored unless
     *     "Products Inventory" → "Display Out of Stock" is off AND
     *     the order's items have 'qty_canceled' set via the inventory
     *     deducer. For programmatic cancellation, call it explicitly.
     */
    public function cancelAndRestoreInventory(int $orderId): OrderInterface
    {
        $order = $this->orderRepository->get($orderId);

        if (!$order->canCancel()) {
            throw new LocalizedException(
                __('Order %1 cannot be canceled in state %2', $order->getIncrementId(), $order->getState())
            );
        }

        // Cancel the order (updates state/status)
        $this->orderManagement->cancel($orderId);

        // Reload to get post-cancel state
        $order = $this->orderRepository->get($orderId);

        // Manually restore inventory for each item
        $this->restoreInventory($order);

        // Add history comment
        $order->addStatusHistoryComment(
            __('Order canceled by training system. Inventory restored.')
        );
        $this->orderRepository->save($order);

        return $order;
    }

    private function restoreInventory(OrderInterface $order): void
    {
        /** @var \Magento\CatalogInventory\Api\StockRegistryInterface $stockRegistry */
        $stockRegistry = $this->_objectManager->get(\Magento\CatalogInventory\Api\StockRegistryInterface::class);

        /** @var \Magento\Sales\Api\Data\OrderItemInterface $item */
        foreach ($order->getItems() as $item) {
            $productId = (int) $item->getProductId();
            if ($productId === 0) {
                continue;
            }

            $sku = $item->getSku();
            $qty = (float) $item->getQtyOrdered();

            try {
                $stockItem = $stockRegistry->getStockItemBySku($sku);
                $stockItem->setQty($stockItem->getQty() + $qty);
                $stockItem->setIsInStock(true);
                $stockRegistry->updateStockItemBySku($sku, $stockItem);
            } catch (\Magento\Framework\Exception\LocalizedException $e) {
                // Log but don't throw — partial failure should not block cancellation
                $this->logger->warning('INVENTORY_RESTORE_FAILED', [
                    'sku' => $sku,
                    'qty' => $qty,
                    'error' => $e->getMessage(),
                ]);
            }
        }
    }
}
```

#### Hold/Unhold Orders

```php
<?php
// Hold an order
$orderManagement->hold($orderId);

// Unhold (resume) an order
$orderManagement->unhold($orderId);
```

An order must be in `holded` state to call `unhold()`. An order must be in `new`, `pending`, or `holded` to be put on hold.

```php
<?php
// Check if hold is possible
if ($order->canHold()) {
    $this->orderManagement->hold($orderId);
}

// Check if unhold is possible
if ($order->canUnhold()) {
    $this->orderManagement->unhold($orderId);
}
```

#### Comment History on Orders

Every state change should be recorded in the order's comment history for auditability. Use `addStatusHistoryComment()`:

> **Why this matters (beyond compliance):** The order comment history is the primary audit trail for any order modification. In dispute resolution (chargebacks, customer complaints), the comment history is the first document auditors request. It is also the only structured record of *who* did *what* and *when* — critical for internal accountability.

**Key points about status history:**

```php
<?php
// $order is a loaded OrderInterface
$order->addStatusHistoryComment(
    __('Customer requested cancellation via phone call. Reason: changed mind.')
        ->setIsVisibleOnFront(true)   // Show on customer account page
        ->setIsCustomerNotification(true) // Optionally email the customer
);

$this->orderRepository->save($order);
```

**What NOT to put in comments:**

| Content | Why Not |
|---------|--------|
| Passwords, payment card numbers | Security risk — comments are stored in DB and may appear in logs |
| Internal-only notes about fraud | These should be in a separate `sales_order_fraud_log` table |
| Debug information | Use `var/log/system.log` or a dedicated debug table |

**Retrieving comment history:**

```php
<?php
// Get all history records
$history = $order->getStatusHistory();

foreach ($history as $comment) {
    echo $comment->getComment();        // The comment text
    echo $comment->getStatus();         // Status at time of comment
    echo $comment->getCreatedAt();      // Timestamp
    echo $comment->getIsVisibleOnFront(); // Whether customer can see it
    echo $comment->getIsCustomerNotified(); // Whether customer was notified
}
```

**Programmatically adding comments without triggering notifications:**

If you need to add internal notes that should NOT send emails:

```php
<?php
$order->addStatusHistoryComment(
    __('Internal: Verified payment authorization.')  // No customer notification
);

$this->orderRepository->save($order);
```

The `setIsCustomerNotification(false)` (the default) ensures no email is sent to the customer when this comment is added.

```php
<?php
// $order is a loaded OrderInterface
$order->addStatusHistoryComment(
    __('Customer requested cancellation via phone call. Reason: changed mind.')
        ->setIsVisibleOnFront(true)   // Show on customer account page
        ->setIsCustomerNotification(true) // Optionally email the customer
);

$this->orderRepository->save($order);
```

The comment history is retrievable via `$order->getStatusHistory()` (returns `DataObject[]`).

---

### Topic 6: Order Search & Management via API

#### Loading Order by `increment_id` vs `entity_id`

| Loader | Method | Use Case |
|--------|--------|----------|
| `entity_id` | `OrderRepositoryInterface::get($id)` | Internal operations, FK relationships |
| `increment_id` | SearchCriteria with `increment_id` filter | External systems, customer-facing display |
| `quote_id` | SearchCriteria with `quote_id` filter | Associate order back to original quote |

```php
<?php
// app/code/Training/Sales/Service/OrderFinder.php
<?php
declare(strict_types=1);

namespace Training\Sales\Service;

use Magento\Sales\Api\OrderRepositoryInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Framework\Api\FilterBuilder;
use Magento\Framework\Api\Search\FilterGroupBuilder;
use Magento\Sales\Api\Data\OrderInterface;

class OrderFinder
{
    public function __construct(
        private readonly OrderRepositoryInterface $orderRepository,
        private readonly SearchCriteriaBuilder $searchCriteriaBuilder,
    ) {
    }

    public function findByIncrementId(string $incrementId): ?OrderInterface
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('increment_id', $incrementId)
            ->create();

        $results = $this->orderRepository->getList($searchCriteria);
        $items = $results->getItems();

        return $items ? reset($items) : null;
    }

    public function findByCustomerId(int $customerId, int $pageSize = 50): array
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('customer_id', $customerId)
            ->setPageSize($pageSize)
            ->create();

        $results = $this->orderRepository->getList($searchCriteria);
        return $results->getItems();
    }

    public function findByDateRange(\DateTime $from, \DateTime $to): array
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('created_at', $from->format('Y-m-d H:i:s'), 'gteq')
            ->addFilter('created_at', $to->format('Y-m-d H:i:s'), 'lteq')
            ->create();

        $results = $this->orderRepository->getList($searchCriteria);
        return $results->getItems();
    }

    public function findByStatus(string $status, string $state): array
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('status', $status)
            ->addFilter('state', $state)
            ->create();

        $results = $this->orderRepository->getList($searchCriteria);
        return $results->getItems();
    }

    public function findByStoreId(int $storeId, int $pageSize = 20): array
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('store_id', $storeId)
            ->setPageSize($pageSize)
            ->create();

        $results = $this->orderRepository->getList($searchCriteria);
        return $results->getItems();
    }
}
```

#### Order Totals Exposed in API Response

When an order is loaded via `OrderRepositoryInterface::get()`, the `OrderInterface` exposes these totals:

| Method | Description |
|--------|-------------|
| `$order->getSubtotal()` | Sum of item row totals |
| `$order->getSubtotalInclTax()` | Subtotal including tax |
| `$order->getTaxAmount()` | Total tax |
| `$order->getDiscountAmount()` | Total discount (negative) |
| `$order->getShippingAmount()` | Shipping charge |
| `$order->getShippingInclTax()` | Shipping including tax |
| `$order->getGrandTotal()` | Final total |
| `$order->getTotalPaid()` | Amount paid so far |
| `$order->getTotalDue()` | Amount outstanding |
| `$order->getTotalRefunded()` | Amount refunded |

#### Updating Shipping Address via API

Order addresses can be updated via `OrderAddressRepositoryInterface::save()` only if the order is in a state that allows modification (typically before any invoice or shipment exists):

```php
<?php
// app/code/Training/Sales/Service/AddressUpdater.php
<?php
declare(strict_types=1);

namespace Training\Sales\Service;

use Magento\Sales\Api\OrderAddressRepositoryInterface;
use Magento\Sales\Api\Data\OrderAddressInterface;
use Magento\Framework\Exception\LocalizedException;

class AddressUpdater
{
    public function __construct(
        private readonly OrderAddressRepositoryInterface $addressRepository,
    ) {
    }

    public function updateShippingAddress(
        int $addressId,
        string $street,
        string $city,
        string $region,
        string $postcode,
        string $countryId,
    ): OrderAddressInterface {
        $address = $this->addressRepository->get($addressId);

        $address->setStreet([$street]);
        $address->setCity($city);
        $address->setRegion($region);
        $address->setPostcode($postcode);
        $address->setCountryId($countryId);

        return $this->addressRepository->save($address);
    }
}
```

> **Limitation:** If an invoice or shipment has already been created, updating the address is not recommended—those entities may have already been shipped or billed to the old address. Check `$order->hasInvoices()` before allowing the update.

#### Adding Order Comments via API

Use `OrderStatusHistoryRepositoryInterface::save()`:

```php
<?php
// app/code/Training/Sales/Service/OrderCommentPoster.php
<?php
declare(strict_types=1);

namespace Training\Sales\Service;

use Magento\Sales\Api\OrderStatusHistoryRepositoryInterface;
use Magento\Sales\Api\Data\OrderStatusHistoryInterface;
use Magento\Sales\Api\Data\OrderStatusHistoryInterfaceFactory;

class OrderCommentPoster
{
    public function __construct(
        private readonly OrderStatusHistoryRepositoryInterface $historyRepository,
        private readonly OrderStatusHistoryInterfaceFactory $historyFactory,
    ) {
    }

    public function addComment(
        int $orderId,
        string $comment,
        bool $isVisibleOnFront = false,
        bool $isCustomerNotified = false,
    ): OrderStatusHistoryInterface {
        /** @var OrderStatusHistoryInterface $history */
        $history = $this->historyFactory->create();
        $history->setParentId($orderId);
        $history->setComment($comment);
        $history->setIsVisibleOnFront($isVisibleOnFront);
        $history->setIsCustomerNotified($isCustomerNotified);
        $history->setEntityName('order'); // Required: 'order' or 'invoice', 'shipment', 'creditmemo'

        return $this->historyRepository->save($history);
    }
}
```

---

## Reference Exercises

### Exercise 1: Observer — Log Order Placement

**Task:** Create an observer on `sales_order_place_after` that logs the order `increment_id` and `customer_email` to `var/log/order_placed.log` every time an order is successfully placed.

**Files to create:**
- `app/code/Training/Sales/etc/events.xml` — registers the observer
- `app/code/Training/Sales/Observer/OrderPlaceAfter.php` — implements `ObserverInterface`

**Acceptance criteria:**
- [ ] `events.xml` uses event name `sales_order_place_after` exactly
- [ ] Observer receives `order` from `$observer->getData('order')`
- [ ] Logger writes a structured entry with `increment_id` and `customer_email`
- [ ] Log file is `var/log/order_placed.log`
- [ ] Observer does not re-dispatch the event (always call parent)

---

### Exercise 2: Controller — Generate Invoice

**Task:** Create a frontend controller at `/sales/order/invoice` (route: `training_sales/order/invoice`) that:
1. Reads the `order_id` GET parameter
2. Loads the order via `OrderRepositoryInterface`
3. Generates a full invoice via `InvoiceRepositoryInterface` (all items, all qty)
4. Returns JSON: `{"invoice_id": <id>, "increment_id": "<invoice_increment>"}`

**Files to create:**
- `app/code/Training/Sales/Controller/Order/Invoice.php`
- `app/code/Training/Sales/etc/frontend/routes.xml` — declares the route
- `app/code/Training/Sales/Service/InvoiceCreator.php` — reusable service class

**Acceptance criteria:**
- [ ] Returns HTTP 400 with error message if order cannot be invoiced
- [ ] Returns HTTP 404 if order not found
- [ ] JSON response has both `invoice_id` and `increment_id`
- [ ] Invoice is created via repository interface (not direct model instantiation and save)

---

### Exercise 3: CLI Command — Create Shipment with Tracking

**Task:** Create a CLI command `bin/magento training:order:ship <order_id> --tracking="<tracking_number>"` that:
1. Loads the order by `$orderId`
2. Creates a full shipment for all items
3. Attaches a tracking record (carrier code: `ups`, title: `United Parcel Service`, number: the `--tracking` value)
4. Sends shipment email
5. Prints the shipment ID on success

**Files to create:**
- `app/code/Training/Sales/Console/Command/ShipOrder.php`
- `app/code/Training/Sales/etc/di.xml` — configures the command in `Magento\Framework\Console\CommandList`
- `app/code/Training/Sales/Service/ShipmentCreator.php` — reusable service class

**Acceptance criteria:**
- [ ] Command is registered and appears in `bin/magento list` under `training`
- [ ] `--tracking` option is required (validated in `configure()`)
- [ ] Order that cannot be shipped (already shipped or canceled) returns an error with the order state
- [ ] Tracking is attached via `ShipmentTrackInterfaceFactory`, not direct table insert

---

### Exercise 4: CLI Command — Create Full Credit Memo

**Task:** Create a CLI command `bin/magento training:order:refund <invoice_id>` that:
1. Loads the invoice via `InvoiceRepositoryInterface`
2. Creates a full offline credit memo (all items, all qty)
3. Prints the credit memo entity ID on success

**Files to create:**
- `app/code/Training/Sales/Console/Command/RefundOrder.php`
- `app/code/Training/Sales/Service/CreditmemoCreator.php`

**Acceptance criteria:**
- [ ] Command accepts invoice ID as positional argument
- [ ] Returns error if order cannot receive a credit memo (state check)
- [ ] Credit memo created via repository interface
- [ ] Prints `Credit memo created with ID: <id>`

---

### Exercise 5: Plugin — Enforce High-Value Order Approval

**Task:** Add a plugin on `OrderRepositoryInterface::save()` that throws `LocalizedException` if:
- The order grand total exceeds **$10,000**
- AND the `approved` extension attribute is not set to `true`

**Files to create/modify:**
- `app/code/Training/Sales/etc/extension_attributes.xml` — declares `approved` attribute on `OrderInterface`
- `app/code/Training/Sales/etc/di.xml` — registers the plugin
- `app/code/Training/Sales/Plugin/OrderRepositoryPlugin.php` — the plugin class

**Acceptance criteria:**
- [ ] `beforeSave` plugin throws `LocalizedException` with clear message for unapproved high-value orders
- [ ] Orders ≤ $10,000 save without error (regardless of approval flag)
- [ ] Orders > $10,000 with `approved = true` save without error
- [ ] Plugin is correctly wired in `di.xml` with correct `sortOrder`

---

### Exercise 6 (Optional): Mass Cancel Action in Adminhtml

**Task:** Design and implement a mass action in the Adminhtml order grid (`sales_order_index` or a custom grid) that:
1. Appears as "Cancel & Restore Inventory" in the mass action dropdown
2. Iterates over selected order IDs
3. For each order, checks `canCancel()` and calls `OrderManagementInterface::cancel()`
4. Explicitly restores inventory for each item (using `StockRegistryInterface`)
5. Adds a status history comment: "Canceled via mass action by admin."

**Files to create:**
- `app/code/Training/Sales/Controller/Adminhtml/Order/MassCancel.php`
- `app/code/Training/Sales/etc/adminhtml/events.xml` — `sales_order_mass_action` (custom event)
- `app/code/Training/Sales/etc/adminhtml/di.xml` — preference for `Magento\Sales\Controller\Adminhtml\Order\MassAction`
- `app/code/Training/Sales/Service/MassCancelService.php`

**Design notes:**
- The mass action URL is registered in the admin layout XML (`sales_order_grid.xml`) via a `<massaction>` block
- Each mass action calls a controller with `batch_size` parameters
- Inventory restoration must use `StockRegistryInterface::updateStockItemBySku()`
- Handle partial failures: if order #3 fails to cancel, log it and continue with the rest. Do not fail the entire batch.

---

## Reading List

| Resource | URL | Why Read |
|---------|-----|---------|
| Magento 2 Developer Documentation — Sales | https://developer.adobe.com/commerce/php/development/components/orders/ | Official overview of the sales architecture |
| `Magento\Sales\Model\Order` source | Examine in `vendor/magento/module-sales/Model/Order.php` | Ground truth for the Order model |
| `Magento\Sales\Api\OrderRepositoryInterface` | `vendor/magento/module-sales/Api/OrderRepositoryInterface.php` | Service contract |
| `Magento\Sales\Model\Order\State` | `vendor/magento/module-sales/Model/Order/State.php` | State constants and transition rules |
| Invoice architecture | https://developer.adobe.com/commerce/php/development/components/invoices/ | Invoice lifecycle |
| Shipment architecture | https://developer.adobe.com/commerce/php/development/components/shipments/ | Shipment lifecycle |
| Credit Memo architecture | https://developer.adobe.com/commerce/php/development/components/credit-memos/ | Credit memo lifecycle |
| MSI (Inventory Management) overview | https://developer.adobe.com/commerce/php/development/components/inventory/ | If using MSI for stock |
| Magento 2 Service Contracts | https://developer.adobe.com/commerce/php/development/services/ | How to correctly use repository interfaces |
| Plugin system | https://developer.adobe.com/commerce/php/development/components/plugin/ | Official plugin documentation |

---

## Edge Cases & Troubleshooting

### Order won't cancel

**Symptom:** `$order->canCancel()` returns `false`.

**Possible causes and fixes:**

| Cause | Check | Fix |
|-------|-------|-----|
| Order already invoiced (fully or partially) | `$order->hasInvoices()` | Invoices must be canceled first. A paid invoice cannot be uncaptured. |
| Order already shipped | `$order->getShipmentsCollection()->getSize() > 0` | Shipments must be deleted first (`ShipmentRepositoryInterface::delete($shipment)`). |
| Order is in `complete` or `closed` state | `$order->getState()` | Only `new`, `pending`, `holded` orders can be canceled. |
| Order payment is in `paypal_review` or similar | Check payment state | Payment must be released first. |

### Invoice won't create (canInvoice() returns false)

Even if the order is in `processing` state, `canInvoice()` is `false` if:
- The order has already been invoiced in full (all items qty_ordered ≤ qty_invoiced)
- The payment method does not support invoicing
- The order is on hold

Use `getInvoiceCollection()` to check existing invoices.

### Shipment won't create (canShip() returns false)

`canShip()` returns `false` if:
- The order has no shippable items (`qty_to_ship = 0` for all items)
- The order is already canceled or closed
- An existing shipment covers all items

### Credit memo won't create (canCreditmemo() returns false)

`canCreditmemo()` returns `false` if:
- The order has not been invoiced at all (no invoice to refund against — use `offline` refund flag)
- The order has been fully refunded already
- The order is canceled

For offline refunds where `canCreditmemo()` returns `false`, you can force-create a refund via the `RefundRegisterInterface` or by using an `order_creditmemo_register` event approach, but this is advanced and typically requires a custom service.

### Plugin on OrderRepositoryInterface doesn't fire

The repository `save()` method internally calls `afterSave()` plugins but `beforeSave()` plugins should fire for all saves. If your `beforeSave` plugin is not being called:
1. Check that `di.xml` has the correct class name: `Magento\Sales\Api\OrderRepositoryInterface`
2. Check the plugin `sortOrder` — lower numbers run first
3. Ensure the plugin class exists and compiles (`bin/magento setup:di:compile`)
4. Verify no other plugin with a lower `sortOrder` is throwing an exception before reaching your plugin

### Inventory not restored on cancel

Cancellation by default does NOT restore inventory unless:
1. The `Credit Memo Stock` setting is `Yes` AND the order is refunded, OR
2. You call `StockRegistryInterface::updateStockItemBySku()` explicitly

Cancellation is different from a credit memo. Cancellation alone frees up reserved inventory but does not return sold inventory to stock unless implemented as shown in Exercise 5.

### Order state transitions throw exceptions in plugin

Never call `$order->setState()` directly. Use `OrderManagementInterface::cancel()`, `hold()`, `unhold()`. Direct state manipulation bypasses business logic and can corrupt the order state machine.

### Plugin recursion

A plugin on `OrderRepositoryInterface::afterSave()` that calls `$orderRepository->save($order)` will cause infinite recursion. If you need to perform a save from within a plugin, use an event observer on `sales_order_save_after` instead, which will not re-trigger the repository save chain.

---

## Common Mistakes to Avoid

### 1. Instantiating models directly and calling save()

**Wrong:**
```php
$invoice = $objectManager->create(\Magento\Sales\Model\Order\Invoice::class);
$invoice->setOrder($order);
$invoice->save(); // Bypasses repository, skips events, can corrupt state
```

**Correct:**
```php
$invoice = $this->invoiceFactory->create(['order' => $order]);
$this->invoiceRepository->save($invoice);
```

### 2. Calling `$order->setState()` directly

**Wrong:**
```php
$order->setState(\Magento\Sales\Model\Order::STATE_CANCELED);
$order->save();
```

**Correct:**
```php
$this->orderManagement->cancel($order->getId());
// Or reload via repository after the service call
```

Direct state manipulation bypasses transition validation, event dispatch, and indexer updates.

### 3. Not checking `can*()` before performing actions

Always call `$order->canInvoice()`, `$order->canShip()`, `$order->canCreditmemo()`, `$order->canCancel()` before attempting creation. Performing these actions on an order that cannot support them throws an exception at best and corrupts data at worst.

### 4. Forgetting to set `entity_id` of extension attributes

When using extension attributes on new (unsaved) entities, the extension attribute must be set on the entity **before** calling the repository's `save()`. Setting it after a load is fine, but before save it must be wired up correctly or the attribute will be silently dropped.

### 5. Not using `SearchCriteriaBuilder` for order searches

Never query `sales_order` table directly with SQL. Use `OrderRepositoryInterface::getList(SearchCriteriaBuilder)` to take advantage of Magento's service contract, caching, and ACL layer. Direct SQL queries bypass everything.

### 6. Mixing Invoice / Shipment / Creditmemo totals

When creating partial invoices or credit memos, make sure the distributed totals add up consistently. For example, if you invoice 3 out of 5 items, the invoice subtotal should reflect exactly those 3 items' row totals. Never hardcode `setSubtotal()` — always call `collectTotals()` and let Magento compute the correct totals from the item rows.

### 7. Not adding order history comments

Every significant action (cancel, hold, shipment, invoice) should be accompanied by a `addStatusHistoryComment()` call. Without it, the admin and customer have no audit trail of what happened to the order and when. This is especially important in legal/regulatory contexts.

### 8. Assuming invoice = payment captured

An invoice being created does not always mean payment has been captured. Forauthorize-then-capture payment flows, the invoice is created at authorization time, but capture happens separately. Check your payment method's flow to understand when funds are actually moved.

### 9. Forgetting to `setup:di:compile` after creating new classes

New repository plugins, observers registered in `events.xml`, and CLI commands registered in `di.xml` are only active after `bin/magento setup:di:compile`. During development, running `bin/magento cache:flush` and then `bin/magento setup:di:compile` is the reliable cycle.

---

*Magento 2 Backend Developer Course — Topic 15 | Sales & Orders*
