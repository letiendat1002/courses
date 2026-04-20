# Topic 10: Checkout — From Quote to Order

**Philosophy:** Checkout is where commerce becomes real — every click converts a wishlist into a revenue event. This topic teaches you to master the entire payment and fulfillment pipeline from a backend perspective.

---

## Overview

The checkout process in Magento 2 is the most stateful, high-stakes flow in the system. Unlike catalog browsing or product search — which are largely read-only — checkout is a write-heavy, multi-step transaction that spans cart management, address resolution, shipping selection, payment processing, and order conversion. Every failure mode must be anticipated. Every success path must be atomic.

This module treats checkout as a **backend systems problem**: how data moves, where validation occurs, how totals are aggregated, how the quote-to-order conversion is orchestrated, and how payment integrations plug into the framework. You will build real shipping carriers, real payment methods, real total collectors, and real checkout step modifications — not tutorials, but production-grade code patterns you will use on every Magento project.

---

## Prerequisites (from core course)

| Module | Why It Matters |
|--------|----------------|
| Topic 04 — Data Layer | Quote is an entity managed via `CartRepositoryInterface`. Understanding the repository pattern is required to load, save, and mutate quote state. |
| Topic 05 — Plugins & Observers | Checkout emits dozens of events and supports extensive plugin hooks. You must know both interception patterns to safely modify checkout behavior. |
| Topic 06 — Admin UI (optional but recommended) | Admin order creation reuses quote conversion logic. Knowing this helps you debug order discrepancies. |
| Topic 07 — REST API (optional but recommended) | Guest checkout and headless integrations expose the same quote/order APIs via REST. Understanding the data model helps you build API contracts. |

---

## Learning Objectives

1. **Model the quote as a transaction object** — understand it as a mutable, session-scoped entity that holds items, addresses, shipping method, and payment method as first-class sub-entities.
2. **Manipulate quote addresses** — load, assign, validate, and persist billing and shipping addresses on a quote programmatically.
3. **Integrate shipping carriers** — implement `Carrier\AbstractCarrier` to create shipping methods with live rate calculation and free-shipping thresholds.
4. **Integrate payment methods** — implement `PaymentMethodInterface`/`AbstractMethod` with the command pattern to support authorize/capture/refund flows.
5. **Control totals calculation** — implement custom total collectors, modify existing collectors via observer, and understand the full collector stack execution order.
6. **Customize checkout steps** — add, remove, and modify steps in the Magento checkout flow using layout XML and AJAX data persistence.
7. **Orchestrate order placement** — understand the full quote-to-order conversion pipeline, handle placement failures, and implement rollback logic.

---

## By End of Module You Must Prove

- [ ] You can load a customer or guest quote, add/remove items, set quantities, and persist changes using the repository layer.
- [ ] You can programmatically set billing and shipping addresses on a quote and validate the address data before saving.
- [ ] You can create a custom shipping carrier that returns calculated rates based on order conditions.
- [ ] You can create a custom payment method that stores transaction state in `additional_information` and responds to `authorize()` and `capture()` calls.
- [ ] You can create a custom total collector and register it in the Magento totals pipeline, or modify an existing total via observer.
- [ ] You can add a custom checkout step that persists data to the quote using extension attributes.
- [ ] You can intercept `PlaceOrderInterface::execute()` with a plugin to log or modify order placement behavior.
- [ ] You understand the full failure spectrum — payment decline, inventory shortage, address validation failure — and know how Magento rolls back quote state on each failure type.
- [ ] You understand the difference between `qty` (physical stock) and `salable_quantity` (available to sell minus reservations) and can explain why checking `cataloginventory_stock_item` directly gives stale data.
- [ ] You can create a shipment programmatically and explain why `register()` is required for inventory to be deducted from sources.
- [ ] You can create a credit memo and explain how it triggers inventory returns via the reservation system.

---

## Assessment Criteria

| Criterion | Weight |
|-----------|--------|
| All 7 reference exercises compile and run without errors | 40% |
| Code follows Magento 2 naming conventions and directory structure | 20% |
| Explanatory prose demonstrates understanding of the data flow, not just the code | 20% |
| Edge cases and failure modes are addressed in writing and code | 20% |

---

## Topics

---

### Topic 1: Quote — The Cart Model

#### Quote vs. Order: Two States of a Transaction

Every commercial transaction in Magento 2 passes through two distinct entity stages:

| Property | Quote | Order |
|----------|-------|-------|
| Mutability | Mutable — items, addresses, methods can be freely changed | Immutable — once placed, the record is permanent |
| Scope | Per-session (guest) or per-customer (logged in) | Per-placement (permanent) |
| Lifecycle | Lives in `sales_quote` / `sales_quote_address` / `sales_quote_item` | Lives in `sales_order` / `sales_order_address` / `sales_order_item` |
| Totals | Calculated on-demand, recalculated on every modification | Calculated at placement time, frozen in the record |
| Expiration | Configurable TTL (default 30 days for guests) | Permanent |
| Customer association | `customer_id` can be null (guest) | Always has `customer_id` if placed by a logged-in customer |

The quote is your workspace. The order is your receipt. You build and modify the workspace; the system takes a snapshot when you place the order.

#### Quote Architecture: Sub-Entities

A quote is not a flat database row. It is a composite aggregate root that owns several child entities:

```
Quote
├── Quote\Address  (shipping address — one per quote)
├── Quote\Address  (billing address — one per quote, same table, different address_type)
├── Collection<Quote\Item>  (line items)
├── Collection<Quote\Payment>  (payment method — one per quote)
└── ExtensionAttributes  (custom data: gift wrap flags, PO numbers, loyalty points)
```

Each sub-entity is stored in its own table (`sales_quote_address`, `sales_quote_item`, `sales_quote_payment`) and is loaded/saved through the quote aggregate. You should never directly save a `Quote\Item` in isolation — changes go through the quote's `collectTotals()` pipeline.

#### Quote Sessions: SessionManagerInterface

Magento decouples the session layer from the quote layer. `SessionManagerInterface` (`Magento\Framework\Session\SessionManagerInterface`) is the container that holds the current quote ID for the active session. The relationship:

```
SessionManager (session storage)
    └──getQuoteId() → loads Quote via CartRepositoryInterface
```

For **guest customers**, the quote is stored in `core_session` with an anonymous quote ID. For **logged-in customers**, the quote ID is stored in the customer session and persisted across logins.

Key session classes:

| Class | Use |
|-------|-----|
| `Magento\Checkout\Model\Session` | Gets current checkout quote (used most commonly in frontend) |
| `Magento\Customer\Model\Session` | Gets customer-specific data |
| `Magento\Backend\Model\Session` | Admin-specific session |
| `Magento\Framework\Session\SessionManagerInterface` | Base interface for all sessions |

#### Loading the Current Quote

```php
<?php
declare(strict_types=1);

namespace Training\Checkout\Controller\Index;

use Magento\Checkout\Model\Session as CheckoutSession;
use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Framework\Controller\ResultFactory;
use Magento\Framework\App\Action\HttpGetActionInterface;

class Index implements HttpGetActionInterface
{
    private CheckoutSession $checkoutSession;
    private CartRepositoryInterface $cartRepository;

    public function __construct(
        CheckoutSession $checkoutSession,
        CartRepositoryInterface $cartRepository
    ) {
        $this->checkoutSession = $checkoutSession;
        $this->cartRepository = $cartRepository;
    }

    public function execute()
    {
        /** @var \Magento\Quote\Model\Quote $quote */
        $quote = $this->checkoutSession->getQuote();

        if (!$quote || !$quote->getId()) {
            // No active quote — cart is empty
            return $this->resultFactory->create(ResultFactory::TYPE_FORWARD)
                ->forward('noroute');
        }

        // Alternatively, load by customer ID:
        // $quote = $this->cartRepository->getForCustomer((int)$customerId);

        // Alternatively, load by quote ID:
        // $quote = $this->cartRepository->get((int)$quoteId);

        foreach ($quote->getItems() as $item) {
            printf(
                "SKU: %s | Qty: %d | Price: %.2f\n",
                $item->getSku(),
                $item->getQty(),
                (float)$item->getPrice()
            );
        }
    }
}
```

**Three ways to load a quote:**

| Method | Signature | Returns |
|--------|-----------|---------|
| `CartRepositoryInterface::get($cartId)` | `int $cartId` | `Quote\|null` by ID |
| `CartRepositoryInterface::getForCustomer($customerId)` | `int $customerId` | `Quote` active cart for customer |
| `Checkout\Session::getQuote()` | no args | `Quote` current session quote |

`getForCustomer()` is preferred when you know the customer ID — it respects the shared customer cart setting (when one customer can have one active cart). `getQuote()` from session is used in the frontend context where the session is already resolved.

#### Adding Products to the Quote

The canonical way to add a product to a quote is through `CartItemRepositoryInterface::save()`. This method accepts a `CartItemInterface` Data Transfer Object (DTO):

```php
<?php
use Magento\Quote\Api\CartItemRepositoryInterface;
use Magento\Quote\Api\Data\CartItemInterface;
use Magento\Quote\Api\Data\CartItemInterfaceFactory;
use Magento\Catalog\Api\ProductRepositoryInterface;

public function addProductToCart(
    CartItemRepositoryInterface $cartItemRepository,
    CartItemInterfaceFactory $cartItemFactory,
    ProductRepositoryInterface $productRepository,
    int $cartId,
    string $sku,
    float $qty
): void {
    $product = $productRepository->get($sku);

    /** @var CartItemInterface $cartItem */
    $cartItem = $cartItemFactory->create();
    $cartItem->setQuoteId($cartId);
    $cartItem->setProduct($product);
    $cartItem->setSku($sku);
    $cartItem->setQty($qty);

    $cartItemRepository->save($cartItem);
}
```

**What happens on `save()`:**

1. The repository checks if an item with the same product_id + product_type + options hash already exists in the quote.
2. If it exists (same SKU, same custom options), it increments the quantity instead of creating a new row.
3. If it's a new item, it inserts into `sales_quote_item`.
4. The quote's `collectTotals()` is triggered automatically, recalculating the subtotal.

**Setting quantities** on an existing item:

```php
<?php
/** @var \Magento\Quote\Model\Quote $quote */
/** @var \Magento\Quote\Model\Quote\Item $item */
$item = $quote->getItemById($itemId);
$item->setQty(5);
$quote->collectTotals();
$cartItemRepository->save($item);
```

**Important:** Never call `$quote->save()` directly on a loaded quote. Use the repository. Direct save bypasses the observer/event layer and can corrupt quote state.

#### Quote Item Options

Quote items carry product configuration through the `options` array. These options are serialized into the `additional_data` column or stored as `product_option` JSON.

**Custom options** (from `catalog_product_option`):

```php
<?php
use Magento\Quote\Api\Data\CartItemInterface;

$cartItem->setOption([
    'option_id' => 42,
    'option_value' => 'custom-engraving-text'
]);
```

**Configurable product selections** (specifying which child SKU was chosen):

```php
<?php
$cartItem->setOption([
    'configurable_item_options' => [
        [
            'option_id' => '93',   // attribute_id from configurable attribute
            'option_value' => '56' // value_id of the selected variant
        ]
    ]
]);
```

**Bundle product selections:**

```php
<?php
$cartItem->setOption([
    'bundle_selection_attributes' => json_encode([
        'product_id' => 104,
        'selection_qty' => 2,
        'option_id' => 15
    ])
]);
```

#### Price Calculation

Magento recalculates quote totals on every modification. This is the critical performance characteristic of the quote lifecycle: **every add, remove, or quantity change triggers a full `collectTotals()` pass**.

`collectTotals()` iterates through all registered total collectors in priority order:

```
SubtotalCollector → ShippingCollector → TaxCollector → DiscountCollector → GrandTotalCollector
```

Each collector reads the quote's items and addresses, computes its portion of the total, and writes back to the quote address totals. The result is stored in `quote_address.shipping_amount`, `quote_address.grand_total`, etc.

You can trigger totals recalculation manually:

```php
<?php
/** @var \Magento\Quote\Model\Quote $quote */
$quote->collectTotals();
```

Or disable auto-calculation during batch operations:

```php
<?php
$quote->setTriggerRecollect(0); // suppresses automatic recalculation
// ... batch operations
$quote->collectTotals(); // manual trigger
```

---

### Topic 2: Address Management

#### Billing and Shipping Address on Quote vs. Order

Magento stores both billing and shipping address on a quote and then copies them to the order at placement time. The addresses live on `sales_quote_address` and `sales_order_address` respectively.

| Address Type | Quote Table | Order Table | Notes |
|--------------|-------------|-------------|-------|
| Shipping | `sales_quote_address` with `address_type = 'shipping'` | `sales_order_address` with `address_type = 'shipping'` | Used for shipping method calculation and delivery |
| Billing | `sales_quote_address` with `address_type = 'billing'` | `sales_order_address` with `address_type = 'billing'` | Used for payment method and invoice address |

A quote always has exactly **one shipping address** and **one billing address** (though they may point to the same row).

#### AddressRepositoryInterface

The address repository pattern mirrors the entity repository pattern:

```php
<?php
use Magento\Quote\Api\AddressRepositoryInterface;
use Magento\Quote\Api\Data\AddressInterface;
use Magento\Quote\Model\Quote\Address as QuoteAddress;

public function updateShippingAddress(
    AddressRepositoryInterface $addressRepository,
    QuoteAddress $quote,
    array $addressData
): void {
    /** @var QuoteAddress $shippingAddress */
    $shippingAddress = $quote->getShippingAddress();

    $shippingAddress->setFirstName($addressData['firstname'] ?? '');
    $shippingAddress->setLastName($addressData['lastname'] ?? '');
    $shippingAddress->setStreet(implode("\n", $addressData['street'] ?? []));
    $shippingAddress->setCity($addressData['city'] ?? '');
    $shippingAddress->setRegion($addressData['region'] ?? '');
    $shippingAddress->setPostcode($addressData['postcode'] ?? '');
    $shippingAddress->setCountryId($addressData['country_id'] ?? '');
    $shippingAddress->setTelephone($addressData['telephone'] ?? '');

    $addressRepository->save($shippingAddress);
}
```

**Important:** Address changes on the quote trigger totals recalculation automatically only if the address changes affect shipping address fields. Billing address changes do not trigger shipping recalculation but do get saved.

#### Address Format and Fields

The standard `AddressInterface` fields:

| Field | Description | Example |
|-------|-------------|---------|
| `firstname` | First name | `Jane` |
| `lastname` | Last name | `Smith` |
| `company` | Business name (optional) | `Acme Corp` |
| `street` | Street address lines (array or newline-joined) | `123 Main St\nSuite 400` |
| `city` | City name | `Chicago` |
| `region` | State/region code or string | `IL` or `Illinois` |
| `region_id` | Numeric region ID (preferred for region dropdown) | `12` |
| `postcode` | Postal/ZIP code | `60601` |
| `country_id` | ISO 3166-1 alpha-2 country code | `US` |
| `telephone` | Phone number | `+1-312-555-0100` |
| `fax` | Fax number (optional) | `+1-312-555-0101` |
| `middlename` | Middle name / middle initial | `Marie` |
| `prefix` | Name prefix | `Dr.` |
| `suffix` | Name suffix | `Jr.` |
| `vat_id` | Tax ID for VAT-enabled countries | `IE1234567X` |
| `customer_id` | Reference to customer entity | `42` |

#### Region/State Lookup

For countries with administrative regions (US states, EU provinces, CA provinces), Magento maintains a region lookup table. Use `Directory/Data` to retrieve valid regions for a country:

```php
<?php
use Magento\Directory\Model\RegionFactory;
use Magento\Directory\Api\RegionInformationInterfaceFactory;

public function getUsRegions(RegionFactory $regionFactory): array
{
    /** @var \Magento\Directory\Model\Region $regionModel */
    $regionModel = $regionFactory->create();
    $regions = $regionModel->getResource()->getLoadedRegionCollection(
        $regionModel->getResource()->getRegionId('US')
    );

    $result = [];
    foreach ($regions as $region) {
        $result[] = [
            'id' => $region->getId(),
            'code' => $region->getCode(),
            'name' => $region->getName(),
        ];
    }
    return $result;
}
```

Or via the API layer:

```php
<?php
use Magento\Directory\Api\RegionInformationInterface;
use Magento\Directory\Api\Data\RegionInformationInterfaceFactory;

public function __construct(
    RegionInformationInterfaceFactory $regionInfoFactory
) {
    $this->regionInfoFactory = $regionInfoFactory;
}

public function getRegionsForCountry(string $countryId): array
{
    return $this->regionInfoFactory->create()
        ->getRegionsForCountry($countryId);
}
```

**The `region_id` vs. `region` field:**  
- `region_id` is the numeric FK to `directory_country_region`. Use this when you want the dropdown selector to work with the admin or checkout UI.
- `region` is the free-text string. Use this when the country has no defined regions or when you want manual entry.

#### Validating Addresses Before Checkout

Address validation in Magento 2 is pluggable. The default address validator is `Magento\Customer\Model\Address\Validator`. To add your own validation:

```php
<?php
// etc/di.xml
<preference for="Magento\Customer\Model\Address\ValidatorInterface"
           type="Training\Checkout\Model\Address\Validator"/>
```

```php
<?php
declare(strict_types=1);

namespace Training\Checkout\Model\Address;

use Magento\Customer\Model\Address\ValidatorInterface;
use Magento\Customer\Api\Data\AddressInterface;

class Validator implements ValidatorInterface
{
    public function validate(AddressInterface $address): array
    {
        $errors = [];

        if (!$address->getTelephone() && !$address->getFax()) {
            $errors[] = __('At least one contact method (phone or fax) is required.');
        }

        if (!$address->getStreet()) {
            $errors[] = __('Street address is required.');
        }

        if (!$address->getPostcode()) {
            $errors[] = __('Postcode is required.');
        }

        // Disallow PO Box shipping to specific carriers
        $carrierRestrictions = ['ups', 'fedex'];
        $street = implode(' ', (array)$address->getStreet());
        if (preg_match('/\bP\.?O\.?\s*Box\b/i', $street)) {
            $errors[] = __('PO Box addresses are not supported for this shipment.');
        }

        return $errors;
    }
}
```

#### Setting Addresses on Quote Programmatically

```php
<?php
use Magento\Quote\Model\Quote\Address as QuoteAddress;
use Magento\Quote\Api\AddressRepositoryInterface;

public function setQuoteAddresses(
    QuoteAddress $shippingAddress,
    QuoteAddress $billingAddress,
    AddressRepositoryInterface $addressRepository
): void {
    // Configure shipping address
    $shippingAddress->setFirstName('Jane')
        ->setLastName('Smith')
        ->setStreet(['123 Main Street', 'Floor 3'])
        ->setCity('Austin')
        ->setRegionId(57) // Texas
        ->setPostcode('78701')
        ->setCountryId('US')
        ->setTelephone('+1-512-555-0100')
        ->setAddressType(QuoteAddress::TYPE_SHIPPING);

    // Configure billing address
    $billingAddress->setFirstName('Jane')
        ->setLastName('Smith')
        ->setStreet(['123 Main Street'])
        ->setCity('Austin')
        ->setRegionId(57)
        ->setPostcode('78701')
        ->setCountryId('US')
        ->setTelephone('+1-512-555-0100')
        ->setAddressType(QuoteAddress::TYPE_BILLING);

    $addressRepository->save($shippingAddress);
    $addressRepository->save($billingAddress);
}
```

---

### Topic 3: Shipping Method Integration

#### ShippingMethodInterface vs. CarrierInterface

Magento 2's shipping architecture has two layers:

| Layer | Interface/Class | Role |
|-------|----------------|------|
| Method | `ShippingMethodInterface` | Represents a specific delivery option presented to the customer (e.g., "Flat Rate - $5.99") |
| Carrier | `CarrierInterface` + `Carrier\AbstractCarrier` | Represents the shipping provider (e.g., "FedEx") and provides one or more shipping methods |

A carrier can produce multiple methods. For example, FedEx might produce: `FEDEX_GROUND`, `FEDEX_EXPRESS`, `FEDEX_OVERNIGHT`. Each method is a rate result from the carrier.

**Built-in carriers:**

| Carrier Code | Class | Type |
|--------------|-------|------|
| `flatrate` | `Magento\Shipping\Model\Carrier\FlatRate` | In-house flat rate |
| `tablerate` | `Magento\Shipping\Model\Carrier\Tablerate` | Table-based rates (weight/destination) |
| `freeshipping` | `Magento\Shipping\Model\Carrier\Freeshipping` | Free shipping threshold |
| `ups` | `Magento\Ups\Model\Carrier` | UPS integration |
| `fedex` | `Magento\Fedex\Model\Carrier` | FedEx integration |
| `dhl` | `Magento\Dhl\Model\Carrier` | DHL integration |

#### Creating a Custom Carrier

A carrier extends `Magento\Shipping\Model\Carrier\AbstractCarrier` and implements `Magento\Shipping\Model\Carrier\CarrierInterface`:

```
Training/Shipping/
├── etc/
│   ├── config.xml           # carrier enabled/disable + default config
│   └── di.xml               # CarrierFactory mapping
├── Model/
│   └── Carrier/
│       └── Training.php
└── registration.php
```

**Step 1: Define the carrier in `etc/di.xml`:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Shipping\Model\CarrierFactory">
        <arguments>
            <argument name="carriers" xsi:type="array">
                <item name="training_shipping" xsi:type="string">Training\Shipping\Model\Carrier\Training</item>
            </argument>
        </arguments>
    </type>
</config>
```

**Step 2: Configure default settings in `etc/config.xml`:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Store:etc/config.xsd">
    <default>
        <carriers>
            <training_shipping>
                <active>true</active>
                <sallowspecific>0</sallowspecific>
                <model>Training\Shipping\Model\Carrier\Training</model>
                <title>Training Shipping</title>
                <name>Training Flat Rate</name>
                <price>5.99</price>
                <free_shipping_threshold>100</free_shipping_threshold>
            </training_shipping>
        </carriers>
    </default>
</config>
```

**Step 3: Implement the carrier:**

```php
<?php
declare(strict_types=1);

namespace Training\Shipping\Model\Carrier;

use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Quote\Model\Quote\Address\RateRequest;
use Magento\Shipping\Model\Carrier\AbstractCarrier;
use Magento\Shipping\Model\Carrier\CarrierInterface;
use Magento\Shipping\Model\Rate\Result;
use Magento\Shipping\Model\Rate\ResultFactory;
use Magento\Framework\App\State;
use Magento\Shipping\Model\Rate\Error\ConditionInterface;
use Psr\Log\LoggerInterface;

class Training extends AbstractCarrier implements CarrierInterface
{
    protected $_code = 'training_shipping';

    private ResultFactory $rateResultFactory;
    private LoggerInterface $logger;
    private ?string $areaCode;

    public function __construct(
        ScopeConfigInterface $scopeConfig,
        \Magento\Shipping\Model\Rate\Error\ConditionInterface $rateError,
        LoggerInterface $logger,
        ResultFactory $rateResultFactory,
        State $state,
        array $data = []
    ) {
        parent::__construct($scopeConfig, $rateError, $logger, $data);
        $this->rateResultFactory = $rateResultFactory;
        $this->logger = $logger;
        $this->areaCode = $state->getAreaCode();
    }

    /**
     * @return bool
     */
    public function getConfigFlag(string $path): bool
    {
        return (bool)$this->_scopeConfig->getValue(
            'carriers/' . $this->_code . '/' . $path,
            \Magento\Store\Model\ScopeInterface::SCOPE_STORE
        );
    }

    /**
     * @return array
     */
    public function getAllowedMethods(): array
    {
        return ['flatrate' => $this->getConfigData('name') ?? 'Training Flat Rate'];
    }

    /**
     * {@inheritdoc}
     *
     * @param RateRequest $request
     * @return Result|false
     */
    public function collectRates(RateRequest $request)
    {
        if (!$this->getConfigFlag('active')) {
            return false;
        }

        $this->_logger->info('Training carrier: checking rates', [
            'package_value' => $request->getPackageValue(),
            'dest_country_id' => $request->getDestCountryId(),
        ]);

        /** @var Result $result */
        $result = $this->rateResultFactory->create();

        $baseRate = (float)($this->getConfigData('price') ?? 5.99);
        $threshold = (float)($this->getConfigData('free_shipping_threshold') ?? 100);

        // Free shipping if order total exceeds threshold
        if ($request->getPackageValue() >= $threshold) {
            $method = $this->createRateMethod(
                'free',
                'Free Training Shipping',
                0.00
            );
            $result->append($method);

            $this->_logger->info('Training carrier: free shipping applied', [
                'package_value' => $request->getPackageValue(),
            ]);
        } else {
            $method = $this->createRateMethod(
                'flatrate',
                (string)($this->getConfigData('name') ?? 'Flat Rate'),
                $baseRate
            );
            $result->append($method);
        }

        return $result;
    }

    /**
     * Create a single rate method within a Result.
     *
     * @param string $methodCode
     * @param string $methodTitle
     * @param float $price
     * @return \Magento\Shipping\Model\Rate\Method
     */
    private function createRateMethod(string $methodCode, string $methodTitle, float $price)
    {
        /** @var \Magento\Shipping\Model\Rate\Method $method */
        $method = $this->_rateMethodFactory->create();
        $method->setCarrier($this->_code);
        $method->setCarrierTitle($this->getConfigData('title') ?? 'Training Shipping');
        $method->setMethod($methodCode);
        $method->setMethodTitle($methodTitle);
        $method->setMethodDescription('Standard handling fee applies');
        $method->setPrice($price);
        $method->setCost(0.00);

        return $method;
    }

    /**
     * Determines if this carrier is applicable for the given destination.
     *
     * @param string $destCountryId
     * @return bool
     */
    public function isShippingAvailable(string $destCountryId): bool
    {
        return true;
    }
}
```

#### Rate Request / Rate Result Pattern

The `RateRequest` object封装 all context needed to calculate shipping:

| Property on RateRequest | Description |
|-------------------------|-------------|
| `getPackageWeight()` | Total weight of the package in grams |
| `getPackageValue()` | Subtotal of the package (before tax/shipping) |
| `getPackageCurrency()` | Currency code |
| `getDestCountryId()` | Destination country (ISO code) |
| `getDestRegionId()` | Destination region ID |
| `getDestRegionCode()` | Destination region code string |
| `getDestPostcode()` | Destination postal code |
| `getDestCity()` | Destination city |
| `getPackageValueWithDiscount()` | Subtotal after discounts |
| `getPackageQty()` | Total quantity of items |
| `getStoreId()` | Current store ID |
| `getWebsiteId()` | Current website ID |

The `RateResult` holds multiple `Method` objects. You append each method to the result. Returning `false` from `collectRates()` indicates the carrier is unavailable — this causes Magento to suppress it from the shipping method list entirely.

#### Setting Shipping Method on Quote

Once the customer selects a shipping method in the checkout UI, Magento posts the selection to `saveShippingMethod` action. The method is stored on the shipping quote address:

```php
<?php
/** @var \Magento\Quote\Model\Quote\Address $shippingAddress */
$shippingAddress->setShippingMethod('training_shipping_flatrate');
$shippingAddress->setShippingDescription('Flat Rate - $5.99');
$shippingAddress->setShippingAmount(5.99);
$shippingAddress->setBaseShippingAmount(5.99);
$shippingAddress->collectShippingRates();
```

The `collectShippingRates()` call triggers re-evaluation of shipping rates for the current address with the now-set shipping method.

#### Shipping Step Observer

The key event for shipping method processing is:

```
checkout_controller_onepage_save_shipping_method
```

This observer fires when the customer selects a shipping method and proceeds. You can observe it to:

1. Store custom attributes on the quote address (e.g., delivery notes)
2. Apply conditional discounts based on the selected carrier
3. Log shipping method selection for analytics

```xml
<!-- etc/events.xml -->
<event name="checkout_controller_onepage_save_shipping_method">
    <observer name="training_shipping_method_save"
              instance="Training\Checkout\Observer\SaveShippingMethod"
              shared="false"/>
</event>
```

```php
<?php
declare(strict_types=1);

namespace Training\Checkout\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Psr\Log\LoggerInterface;

class SaveShippingMethod implements ObserverInterface
{
    private LoggerInterface $logger;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function execute(Observer $observer): void
    {
        $quote = $observer->getQuote();
        $shippingAddress = $quote->getShippingAddress();
        $shippingMethod = $shippingAddress->getShippingMethod();
        $shippingDescription = $shippingAddress->getShippingDescription();

        $this->logger->info('Shipping method selected', [
            'quote_id' => $quote->getId(),
            'method' => $shippingMethod,
            'description' => $shippingDescription,
        ]);
    }
}
```

#### Handling Fees

Magento's built-in carriers support a per-method handling fee calculated as:

```
handlingFee = (baseHandlingFee + (packageCost * handlingAction)) * packageQty
```

Configure this in `config.xml`:

```xml
<flatrate>
    <handling_type>F</handling_type>  <!-- 'I' = fixed, 'P' = percent -->
    <handling_fee>5.00</handling_fee>
</flatrate>
```

To add handling fee to your custom carrier, override `getHandlingFee()`:

```php
<?php
public function getHandlingFee(float $cost): float
{
    $handlingType = $this->getConfigData('handling_type');
    $handlingFee = (float)$this->getConfigData('handling_fee');

    if ($handlingType === 'P') {
        return $cost * ($handlingFee / 100);
    }

    return $handlingFee;
}
```

---

### Topic 4: Payment Method Integration

#### PaymentMethodInterface

The payment method interface defines the contract every payment method must implement:

```php
<?php
interface PaymentMethodInterface
{
    public function authorize(
        \Magento\Payment\Model\InfoInterface $payment,
        float $amount
    ): void;

    public function capture(
        \Magento\Payment\Model\InfoInterface $payment,
        float $amount
    ): void;

    public function cancel(\Magento\Payment\Model\InfoInterface $payment): void;

    public function refund(
        \Magento\Payment\Model\InfoInterface $payment,
        float $amount
    ): void;

    public function void(\Magento\Payment\Model\InfoInterface $payment): void;

    public function canCapture(): bool;
    public function canAuthorize(): bool;
    public function canRefund(): bool;
    public function canUseForCountry(string $country): bool;
    public function canUseForCurrency(string $currencyCode): bool;
    public function getTitle(): string;
    public function getCode(): string;
}
```

In practice, you extend `AbstractMethod` which provides default implementations for most methods, and you override only the ones relevant to your payment flow.

**The payment flow types:**

| Flow | Description | Method Call |
|------|-------------|-------------|
| Authorize Only | Reserve funds, do not capture | `authorize()` |
| Authorize and Capture | Charge immediately at time of order | `capture()` |
| Capture on invoice | Capture after order is placed | `capture()` on invoice |
| Refund | Return funds to customer | `refund()` |

**AbstractMethod's key properties:**

| Property | Description |
|----------|-------------|
| `$_code` | Unique payment method code |
| `$_infoBlockType` | Block type for payment info rendering |
| `$_formBlockType` | Block type for the payment form |
| `$_canAuthorize` | Whether `authorize()` is supported |
| `$_canCapture` | Whether `capture()` is supported |
| `$_canCapturePartial` | Whether partial captures are supported |
| `$_canRefund` | Whether `refund()` is supported |
| `$_canRefundInvoicePartial` | Whether partial refunds on invoice are supported |
| `$_canVoid` | Whether `void()` is supported |
| `$_canUseForMultishipping` | Whether usable in multishipping checkout |

#### PaymentMethodDataProviderInterface

This interface provides default configuration for the payment method when it has no stored configuration. It is used by the payment method factory when instantiating a method with no existing config:

```php
<?php
declare(strict_types=1);

namespace Training\Checkout\Model\Payment;

use Magento\Payment\Model\MethodInterface;
use Magento\Payment\Model\Method\Adapter;
use Magento\Payment\Helper\Data as PaymentHelper;

class TrainingOffline extends Adapter implements MethodInterface
{
    public function __construct(
        PaymentHelper $paymentDataHelper,
        \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig,
        \Magento\Payment\Model\Carrier\Source\MethodInterface $methodSource,
        \Magento\Payment\Model\InfoFactory $infoFactory,
        \Magento\Payment\Model\Command\PoolFactory $commandPoolFactory,
        \Magento\Payment\Model\Validator\PoolFactory $validatorPool,
        \Magento\Framework\Event\ManagerInterface $eventManager,
        \Magento\Framework\App\CacheInterface $cache,
        \Magento\Framework\App\Config\ScopeConfigInterface $config,
        string $code = 'training_offline'
    ) {
        parent::__construct(
            $code,
            $paymentDataHelper,
            $scopeConfig,
            $methodSource,
            $infoFactory,
            $commandPoolFactory,
            $validatorPool,
            $eventManager,
            $cache,
            $config
        );
    }
}
```

#### Command Pattern for Payment Operations

Magento 2 decouples payment operations (authorize, capture, refund) from the payment method using a command pool. Each command implements `Gateway\CommandInterface`:

```php
<?php
namespace Magento\Payment\Model\Command;

interface CommandInterface
{
    /**
     * @param array $subject  The payment subject (contains payment + amount)
     * @param array $arguments  Additional arguments (e.g., invoice for capture)
     * @return ResultInterface
     */
    public function execute(array $subject, array $arguments = []);
}
```

The `CommandPool` maps command names to command instances in `etc/di.xml`:

```xml
<!-- Training/Payment/etc/di.xml -->
<preference for="Magento\Payment\Model\Method\Command\CommandPoolInterface"
           type="Training\Payment\Model\Command\TrainingCommandPool"/>
```

A complete offline payment command implementation (authorize only):

```php
<?php
declare(strict_types=1);

namespace Training\Payment\Model\Command;

use Magento\Payment\Gateway\CommandInterface;
use Magento\Payment\Gateway\Data\DataObjectInterface;
use Magento\Payment\Gateway\Response\HandlerInterface;

class AuthorizeCommand implements CommandInterface
{
    private HandlerInterface $handler;
    private LoggerInterface $logger;

    public function __construct(
        HandlerInterface $handler,
        LoggerInterface $logger
    ) {
        $this->handler = $handler;
        $this->logger = $logger;
    }

    public function execute(array $subject, array $arguments = []): void
    {
        /** @var DataObjectInterface $order */
        $order = $subject['order'];
        $amount = $subject['amount'];

        $this->logger->info('Training offline: authorizing payment', [
            'amount' => $amount,
            'order_id' => $order->getExtOrderId(),
        ]);

        // Simulate successful authorization
        $transactionId = 'AUTH-' . time() . '-' . random_int(1000, 9999);

        $response = [
            HandlerInterface::TXN_ID => $transactionId,
            HandlerInterface::TXN_STATUS => 'pending',
        ];

        $this->handler->handle($subject, $response);
    }
}
```

#### Building a Custom Payment Method: Full Implementation

**File structure:**

```
Training/Payment/
├── etc/
│   ├── config.xml            # payment method config
│   └── di.xml                # command pool, validators
├── etc/adminhtml/
│   └── system.xml           # payment method settings in admin
├── Model/
│   ├── Method/
│   │   └── TrainingOffline.php
│   └── Command/
│       ├── AuthorizeCommand.php
│       ├── CaptureCommand.php
│       └── RefundCommand.php
├── Observer/
│   └── PaymentMethodIsActive.php
└── registration.php
```

**The payment method model:**

```php
<?php
declare(strict_types=1);

namespace Training\Payment\Model\Method;

use Magento\Payment\Model\Method\Adapter;
use Magento\Payment\Model\Method\TransparentInterface;
use Magento\Framework\App\ObjectManager;

class TrainingOffline extends Adapter implements TransparentInterface
{
    protected $_code = 'training_offline';
    protected $_canAuthorize = true;
    protected $_canCapture = false;
    protected $_canRefund = false;
    protected $_canVoid = false;
    protected $_canUseInternal = true;
    protected $_canUseCheckout = true;
    protected $_isInitializeNeeded = false;
    protected $_formBlockType = \Magento\Payment\Block\Form::class;
    protected $_infoBlockType = \Training\Payment\Block\Info\TrainingOffline::class;

    /**
     * Override authorize to always succeed for training purposes.
     * In production, replace with actual gateway call.
     */
    public function authorize(\Magento\Payment\Model\InfoInterface $payment, float $amount): void
    {
        if (!$this->canAuthorize()) {
            throw new \Magento\Framework\Exception\PaymentException(
                __('The payment action "%1" is not available.', 'authorize')
            );
        }

        $this->logger->info('Training offline: processing authorize', [
            'amount' => $amount,
            'payment_id' => $payment->getId(),
        ]);

        // Store authorization details in payment info
        $payment->setTransactionId('AUTH-' . time());
        $payment->setIsTransactionClosed(false);
        $payment->setTransactionPendingStatus();

        // Always mark as successful for training module
        // Real implementation: call the payment gateway API here
    }

    /**
     * Mark order as paid immediately.
     */
    public function capture(\Magento\Payment\Model\InfoInterface $payment, float $amount): void
    {
        if (!$this->canCapture()) {
            parent::capture($payment, $amount);
            return;
        }

        $this->logger->info('Training offline: processing capture', [
            'amount' => $amount,
            'payment_id' => $payment->getId(),
        ]);

        $payment->setTransactionId('CAPTURE-' . time());
        $payment->setIsTransactionClosed(true);
        $payment->setTransactionApprovedStatus();
    }

    /**
     * @inheritDoc
     */
    public function getTitle(): string
    {
        return $this->getConfigData('title') ?? 'Training Offline Payment';
    }
}
```

**Payment info block for admin display:**

```php
<?php
declare(strict_types=1);

namespace Training\Payment\Block\Info;

use Magento\Payment\Block\Info;

class TrainingOffline extends Info
{
    protected $_template = 'Training_Payment::info/training_offline.phtml';

    public function getSpecificInformation(): array
    {
        $data = parent::getSpecificInformation();
        $info = $this->getInfo();

        $additionalData = $info->getAdditionalInformation();

        if (!empty($additionalData['transaction_id'])) {
            $data[] = [
                'label' => __('Transaction ID'),
                'value' => $additionalData['transaction_id'],
            ];
        }

        if (!empty($additionalData['po_number'])) {
            $data[] = [
                'label' => __('PO Number'),
                'value' => $additionalData['po_number'],
            ];
        }

        return $data;
    }
}
```

**Template `view/adminhtml/templates/info/training_offline.phtml`:**

```php
<?php
/** @var \Training\Payment\Block\Info\TrainingOffline $block */
$info = $block->getInfo();
?>
<dl class="payment-method">
    <dt class="title"><?= $block->escapeHtml($block->getMethod()->getTitle()) ?></dt>
    <?php if ($block->getSpecificInformation()): ?>
    <dd class="content">
        <table class="data-table">
            <?php foreach ($block->getSpecificInformation() as $item): ?>
            <tr>
                <td class="label"><?= $block->escapeHtml($item['label']) ?></td>
                <td class="value"><?= $block->escapeHtml($item['value']) ?></td>
            </tr>
            <?php endforeach; ?>
        </table>
    </dd>
    <?php endif; ?>
</dl>
```

**Observer to restrict payment method availability:**

```xml
<!-- etc/events.xml -->
<event name="payment_method_is_active">
    <observer name="training_payment_restrict"
              instance="Training\Payment\Observer\PaymentMethodIsActive"
              shared="false"/>
</event>
```

```php
<?php
declare(strict_types=1);

namespace Training\Payment\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

class PaymentMethodIsActive implements ObserverInterface
{
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getQuote();
        $methodInstance = $observer->getMethodInstance();
        $result = $observer->getResult();

        if ($methodInstance->getCode() !== 'training_offline') {
            return;
        }

        // Restrict to orders over $10 to prevent test orders
        if ($quote && $quote->getBaseGrandTotal() < 10.00) {
            $result->isAvailable = false;
        }
    }
}
```

#### Storing Additional Information

Payment methods store extra data in `additional_information`:

```php
<?php
// Storing
$payment->setAdditionalInformation('po_number', 'PO-12345');
$payment->setAdditionalInformation('transaction_id', $txnId);
$payment->setAdditionalInformation('auth_code', $authCode);

// Retrieving
$poNumber = $payment->getAdditionalInformation('po_number');

// Removing
$payment->unsAdditionalInformation('po_number');
```

This data is serialized into `sales_order_payment.additional_data` and `sales_quote_payment.additional_data`.

#### Payment Failures

When payment fails, throw `\Magento\Framework\Exception\CouldNotSaveException`:

```php
<?php
use Magento\Framework\Exception\CouldNotSaveException;

try {
    $this->processPayment($payment, $amount);
} catch (\RuntimeException $e) {
    throw new CouldNotSaveException(
        __('Payment could not be processed: %1', $e->getMessage()),
        $e,
        422
    );
}
```

The `CouldNotSaveException` is caught by the checkout controller and displayed as an error message on the payment step. The quote is not modified — it stays in the "awaiting payment" state.

---

### Topic 5: Totals Collectors — Quote Price Calculation

#### How Magento Calculates Totals: TotalCollectorInterface

Every time the quote's items, addresses, shipping method, or payment method changes, Magento recalculates the quote's totals by running the full collector stack. The stack is a priority-sorted list of total collectors registered in `sales_totals` di.xml.

Each collector:
1. Receives the entire `Quote` object
2. Reads the relevant data from items and addresses
3. Computes and sets totals on the `Quote\Address` (shipping address) and/or `Quote` root
4. Returns `void` — the results are written directly to the address object

#### The Collector Stack

The default collector execution order:

| Priority | Collector | Sets on Address |
|----------|-----------|-----------------|
| 10 | `subtotal` | `subtotal`, `base_subtotal`, `subtotal_with_discount`, `base_subtotal_with_discount` |
| 20 | `shipping` | `shipping_amount`, `base_shipping_amount`, `shipping_discount_amount`, `base_shipping_discount_amount`, `shipping_incl_tax`, `base_shipping_incl_tax` |
| 30 | `discount` | `discount_amount`, `base_discount_amount`, `隐藏_discount_description` |
| 40 | `tax` | `tax_amount`, `base_tax_amount`, `hidden_tax_amount` |
| 50 | `grand_total` | `grand_total`, `base_grand_total`, `base_to_global_rate`, `global_currency_code` |
| 60 | `customer_balance` | `customer_balance_amount`, `base_customer_balance_amount` |
| 70 | `reward` | `reward_points_balance`, `reward_currency_amount` |

#### The `collect(Quote $quote)` Method

Every total collector implements this interface:

```php
<?php
namespace Magento\Quote\Model\Quote\Address\Total;

use Magento\Quote\Model\Quote;
use Magento\Quote\Model\Quote\Address;

interface TotalCollectorInterface
{
    /**
     * @return float
     */
    public function getPriority(): int;

    /**
     * @param Quote $quote
     * @return $this
     */
    public function collect(Quote $quote): self;

    /**
     * @param Address $address
     * @return $this
     */
    public function fetch(Quote\Address $address): self;
}
```

Example: a handling fee collector (`Training\Checkout\Model\Total\HandlingFee`):

```php
<?php
declare(strict_types=1);

namespace Training\Checkout\Model\Total;

use Magento\Quote\Model\Quote;
use Magento\Quote\Model\Quote\Address;
use Magento\Quote\Model\Quote\Address\Total as AddressTotal;
use Magento\Quote\Model\Quote\Address\Total\AbstractTotal;
use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Store\Model\ScopeInterface;

class HandlingFee extends AbstractTotal
{
    private const HANDLING_FEE_AMOUNT = 2.99;

    private ScopeConfigInterface $scopeConfig;

    public function __construct(
        ScopeConfigInterface $scopeConfig,
        array $data = []
    ) {
        parent::__construct($data);
        $this->scopeConfig = $scopeConfig;
    }

    /**
     * @param Quote $quote
     * @return $this
     */
    public function collect(Quote $quote): self
    {
        parent::collect($quote);

        $shippingAddress = $quote->getShippingAddress();
        if (!$shippingAddress || !$shippingAddress->getId()) {
            return $this;
        }

        // Only add handling fee if there are items in the cart
        if ($quote->getItemsCount() === 0) {
            return $this;
        }

        $feeAmount = $this->getHandlingFeeAmount();
        $baseFeeAmount = $feeAmount;

        // Add to address totals
        $shippingAddress->setHandlingFee($feeAmount);
        $shippingAddress->setBaseHandlingFee($baseFeeAmount);

        $shippingAddress->setGrandTotal(
            $shippingAddress->getGrandTotal() + $feeAmount
        );
        $shippingAddress->setBaseGrandTotal(
            $shippingAddress->getBaseGrandTotal() + $baseFeeAmount
        );

        $shippingAddress->setTotalAmount('handling_fee', $feeAmount);
        $shippingAddress->setBaseTotalAmount('handling_fee', $baseFeeAmount);

        return $this;
    }

    /**
     * @param Address $address
     * @return $this
     */
    public function fetch(Quote\Address $address): self
    {
        $amount = $address->getHandlingFee();
        if ($amount > 0) {
            $address->addTotal([
                'code' => 'handling_fee',
                'title' => __('Handling Fee'),
                'value' => $amount,
            ]);
        }

        return $this;
    }

    private function getHandlingFeeAmount(): float
    {
        return (float)($this->scopeConfig->getValue(
            'training_checkout/totals/handling_fee_amount',
            ScopeInterface::SCOPE_STORE
        ) ?? self::HANDLING_FEE_AMOUNT);
    }
}
```

#### Registering a Custom Total Collector

In `etc/di.xml` (in the `Magento/Sales/etc/di.xml` extended config or module-specific):

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Quote\Model\Quote\Address\Total\Collector">
        <arguments>
            <argument name="collectors" xsi:type="array">
                <item name="handling_fee" xsi:type="string">Training\Checkout\Model\Total\HandlingFee</item>
            </argument>
        </arguments>
    </type>
</config>
```

**Alternative: Modify an existing total via observer (no di.xml modification):**

```xml
<!-- etc/events.xml -->
<event name="sales_quote_address_total_collect">
    <observer name="training_adjust_subtotal"
              instance="Training\Checkout\Observer\AdjustSubtotal"
              shared="false"/>
</event>
```

```php
<?php
declare(strict_types=1);

namespace Training\Checkout\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Quote\Model\Quote;

class AdjustSubtotal implements ObserverInterface
{
    public function execute(Observer $observer): void
    {
        /** @var Quote $quote */
        $quote = $observer->getQuote();
        $address = $quote->getShippingAddress();

        // Add a $0.50 processing fee per item (not per order)
        $itemCount = $quote->getItemsCount();
        $processingFee = $itemCount * 0.50;

        $address->setTotalAmount('processing_fee', $processingFee);
        $address->setBaseTotalAmount('processing_fee', $processingFee);

        $address->setGrandTotal($address->getGrandTotal() + $processingFee);
        $address->setBaseGrandTotal($address->getBaseGrandTotal() + $processingFee);
    }
}
```

#### Tax Calculation: TaxCalculationInterface

For proper tax calculation integration, use the `TaxCalculationInterface`:

```php
<?php
use Magento\Tax\Api\TaxCalculationInterface;
use Magento\Quote\Model\Quote\Item;
use Magento\Tax\Api\Data\QuoteDetailsItemInterface;
use Magento\Tax\Api\Data\QuoteDetailsItemInterfaceFactory;
use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Customer\Api\GroupRepositoryInterface;

public function calculateItemTax(
    TaxCalculationInterface $taxCalculation,
    QuoteDetailsItemInterfaceFactory $itemFactory,
    Quote $quote,
    Item $item,
    int $customerId
): float {
    $taxRate = $taxCalculation->getRate(
        $item->getProduct()->getTaxClassId(),
        $customerId,
        $quote->getStoreId()
    );

    $itemDetails = $itemFactory->create([
        'code' => $item->getSku(),
        'quantity' => $item->getQty(),
        'unit_price' => $item->getPrice(),
        'tax_class_key' => [
            'type' => QuoteDetailsItemInterface::KEY_TYPE_ID,
            'value' => $item->getProduct()->getTaxClassId(),
        ],
    ]);

    $quoteDetails = new \Magento\Tax\Api\Data\QuoteDetailsData();
    $quoteDetails->setItems([$itemDetails]);
    $quoteDetails->setCustomerId($customerId);
    $quoteDetails->setShippingAddress($quote->getShippingAddress()->getDataModel());

    $taxDetails = $taxCalculation->calculateTax($quoteDetails, $quote->getStoreId());

    return $taxDetails->getRowTax();
}
```

#### Discount Totals: Discount Collector with Coupon Support

The `Discount` collector aggregates all applied discounts (catalog rules, cart rules, shipping discounts) into a single `discount_amount` total:

```php
<?php
// Apply a discount programmatically
$quote->setCouponCode('SAVE20');
$quote->collectTotals();

// Check if discount was applied
$discountAmount = $quote->getShippingAddress()->getDiscountAmount();
$discountDescription = $quote->getShippingAddress()->getDiscountDescription();

// To apply a custom discount without a coupon rule:
// Use a salesrule (cart price rule) programmatically
<?php
use Magento\SalesRule\Model\RuleFactory;
use Magento\SalesRule\Model\Rule;

public function applyPercentDiscount(RuleFactory $ruleFactory, Quote $quote, float $percent): void
{
    /** @var Rule $rule */
    $rule = $ruleFactory->create();
    $rule->setName('Training Custom Discount')
        ->setSimpleAction(Rule::BY_PERCENT_ACTION)
        ->setDiscountAmount($percent)
        ->setIsActive(true)
        ->setStoreLabels([
            ['store_id' => 0, 'label' => 'Custom Discount']
        ]);

    // Apply the rule directly to the quote
    $rule->getConditions()->loadPost([]);
    $rule->apply($quote);
}
```

---

### Topic 6: Checkout Step Customization

#### Checkout Layout Handles

The Magento checkout is a full-page application loaded via the `checkout_index_index` layout handle. Each step is a Knockout.js component registered through layout XML. The layout file is:

```
<module_path>/view/frontend/layout/checkout_index_index.xml
```

**Key layout handles for checkout:**

| Handle | Purpose |
|--------|---------|
| `checkout_index_index` | Main checkout page (all steps) |
| `checkout_onepage_index` | Legacy onepage checkout |
| `checkout_index_index.xml` | Checkout page layout (modern) |

#### Adding a Custom Checkout Step

**Step 1: Register the step in layout XML:**

```xml
<!-- Training/Checkout/view/frontend/layout/checkout_index_index.xml -->
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceBlock name="checkout.root">
            <arguments>
                <argument name="jsLayout" xsi:type="array">
                    <item name="components" xsi:type="array">
                        <item name="checkout" xsi:type="array">
                            <item name="children" xsi:type="array">
                                <item name="steps" xsi:type="array">
                                    <item name="children" xsi:type="array">
                                        <item name="training-po-step" xsi:type="array">
                                            <item name="component" xsi:type="string">
                                                Training_Checkout/js/view/checkout/checkout-po-step
                                            </item>
                                            <item name="sortOrder" xsi:type="string">4</item>
                                            <item name="children" xsi:type="array">
                                                <item name="step-content" xsi:type="array">
                                                    <item name="component"
                                                          xsi:type="string">
                                                        Training_Checkout/js/view/checkout/pop/step-content
                                                    </item>
                                                </item>
                                            </item>
                                        </item>
                                    </item>
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

**Step 2: Create the Knockout.js step view model:**

```javascript
// Training/Checkout/view/frontend/web/js/view/checkout/checkout-po-step.js
define([
    'ko',
    'Magento_Ui/js/core/app',
    'mage/translate',
    'Magento_Checkout/js/view/step-abstract'
], function (ko, _, $t, StepAbstract) {
    'use strict';

    return StepAbstract.extend({
        defaults: {
            template: 'Training_Checkout/checkout/pop/step',
            stepCode: 'trainingPoStep',
            stepTitle: $t('Purchase Order')
        },

        initialize: function () {
            this._super();
            this.stepCode = 'trainingPoStep';
            return this;
        },

        initObservable: function () {
            this._super()
                .observe({
                    poNumber: ''
                });
            return this;
        },

        /**
         * @returns {boolean}
         */
        isVisible: function () {
            return ko.unwrap(this.isShow());
        },

        /**
         * @returns {boolean}
         */
        validate: function () {
            var poNumber = this.poNumber();
            if (!poNumber || poNumber.trim().length === 0) {
                this.error(true);
                return false;
            }
            if (!/^PO-\d{4,}$/.test(poNumber)) {
                this.error('PO number must be in format PO-XXXX');
                return false;
            }
            this.error(false);
            return true;
        },

        /**
         * @returns {void}
         */
        savePoNumber: function () {
            this.source.set('poNumber', this.poNumber());
            this.isProcessed(true);
        }
    });
});
```

**Step 3: Create the step content component:**

```javascript
// Training/Checkout/view/frontend/web/js/view/checkout/pop/step-content.js
define([
    'ko',
    'Magento_Ui/js/form/form',
    'Magento_Checkout/js/model/step-navigator'
], function (ko, Form, stepNavigator) {
    'use strict';

    return Form.extend({
        defaults: {
            template: 'Training_Checkout/checkout/pop/step-content',
           一步_code: 'trainingPoStep'
        },

        initialize: function () {
            this._super();
            stepNavigator.registerStep(
                this.stepCode,
                null,
                this.stepTitle,
                this.isVisible.bind(this),
                this.navigate.bind(this),
                this.sortOrder
            );
            return this;
        },

        navigate: function () {
            // Called when step becomes active
        },

        onSubmit: function () {
            if (this.source.get('poNumber')) {
                stepNavigator.next();
            }
        }
    });
});
```

**Step 4: Save step data to quote via server-side controller:**

```php
<?php
declare(strict_types=1);

namespace Training\Checkout\Controller\SavePo;

use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Framework\Controller\ResultFactory;
use Magento\Framework\App\Action\Context;
use Magento\Checkout\Model\Session as CheckoutSession;
use Magento\Framework\Controller\Result\JsonFactory;

class Index implements HttpPostActionInterface
{
    private CheckoutSession $checkoutSession;
    private ResultFactory $resultFactory;
    private JsonFactory $resultJsonFactory;

    public function __construct(
        Context $context,
        CheckoutSession $checkoutSession,
        JsonFactory $resultJsonFactory
    ) {
        parent::__construct($context);
        $this->checkoutSession = $checkoutSession;
        $this->resultJsonFactory = $resultJsonFactory;
        $this->resultFactory = $context->getResultFactory();
    }

    public function execute()
    {
        $poNumber = $this->getRequest()->getParam('po_number', '');
        $quote = $this->checkoutSession->getQuote();

        // Store in quote extension attribute
        $quote->getExtensionAttributes()
            ?? $quote->extensionAttributesFactory->create();

        $quote->getExtensionAttributes()->setPoNumber($poNumber);
        $quote->getExtensionAttributes()->setData('po_number', $poNumber);

        // Also store directly on quote for simplicity
        $quote->setData('po_number', $poNumber);

        try {
            $quote->collectTotals();
            $quote->save();

            return $this->resultJsonFactory->create()->setData([
                'success' => true,
            ]);
        } catch (\Exception $e) {
            return $this->resultJsonFactory->create()->setData([
                'success' => false,
                'error' => $e->getMessage(),
            ]);
        }
    }
}
```

#### Disabling Built-in Steps

To disable a built-in checkout step (e.g., remove the shipping step for virtual-only orders):

```xml
<!-- In your module's checkout_index_index.xml -->
<referenceBlock name="checkout.root">
    <arguments>
        <argument name="jsLayout" xsi:type="array">
            <item name="components" xsi:type="array">
                <item name="checkout" xsi:type="array">
                    <item name="children" xsi:type="array">
                        <item name="steps" xsi:type="array">
                            <item name="children" xsi:type="array">
                                <item name="shipping-step" xsi:type="array">
                                    <item name="component" xsi:type="string">
                                        knockoutjs/empty-component
                                    </item>
                                </item>
                            </item>
                        </item>
                    </item>
                </item>
            </item>
        </argument>
    </arguments>
</referenceBlock>
```

Or use `<item name="shipping-step" remove="true"/>` to completely remove it from the layout.

#### Custom Step Data Storage in Quote Extension Attributes

Extension attributes are the Magento 2 recommended way to add custom fields to any entity without modifying the core schema. Define them in `etc/extension_attributes.xml`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Extension/etc/extension_attribute.xsd">
    <extension_attributes for="Magento\Quote\Api\Data\CartInterface">
        <attribute code="po_number" type="string"/>
        <attribute code="gift_wrap" type="boolean"/>
        <attribute code="gift_message" type="string"/>
    </extension_attributes>
    <extension_attributes for="Magento\Sales\Api\Data\OrderInterface">
        <attribute code="po_number" type="string"/>
        <attribute code="gift_wrap" type="boolean"/>
        <attribute code="gift_message" type="string"/>
    </extension_attributes>
</config>
```

Then read/write using the extension attributes API:

```php
<?php
use Magento\Quote\Api\Data\CartInterface;

// Write
$quote->getExtensionAttributes()->setPoNumber('PO-20240101');

// Read
$poNumber = $quote->getExtensionAttributes()->getPoNumber();
```

#### AJAX vs. Form Post for Step Saves

All modern Magento checkout step saves use **AJAX** (XMLHttpRequest), not form posts. The pattern:

```
Frontend (Knockout.js) → AJAX POST → Server Controller → Quote Update → JSON Response
```

The server response must be JSON:

```json
{
    "success": true,
    "goto_section": "payment",
    "allow_sections": ["shipping"]
}
```

On error:

```json
{
    "success": false,
    "error": "Invalid PO number format.",
    "fields": {
        "po_number": "Invalid format"
    }
}
```

---

### Topic 7: Order Placement & Failures

#### PlaceOrderInterface

`PlaceOrderInterface` (`Magento\Checkout\Model\PlaceOrderInterface`) is the orchestration service that converts a validated quote into an order. It is the single entry point for order placement:

```php
<?php
namespace Magento\Checkout\Api;

interface PlaceOrderInterface
{
    /**
     * @param int $cartId
     * @return int  Order ID
     * @throws \Magento\Framework\Exception\CouldNotSaveException
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function execute(int $cartId): int;
}
```

Implementation pattern:

```php
<?php
public function execute(int $cartId): int
{
    // 1. Load quote
    $quote = $this->cartRepository->get($cartId);

    // 2. Validate quote
    $this->validateQuote($quote);

    // 3. Reserve order ID
    $orderId = $this->sequenceManager->getNextOrderId();

    // 4. Convert quote to order
    $order = $this->quoteToOrderConverter->convert($quote, $orderId);

    // 5. Save order
    $this->orderRepository->save($order);

    // 6. Trigger post-placement observers
    $this->eventManager->dispatch('checkout_onepage_place_order', [
        'order' => $order,
        'quote' => $quote,
    ]);

    // 7. Invalidate quote
    $this->invalidateQuote($quote);

    return (int)$order->getId();
}
```

#### Quote → Order Conversion Flow

The conversion happens in `Magento\Quote\Model\QuoteManagement::submit()`:

```
QuoteManagement::submit()
    ├── triggerRecollect()              → recalculate totals one final time
    ├── _validateQuote()               → check quote is active, has items, has addresses
    ├── _prepareQuoteAddresses()        → copy addresses to order addresses
    ├── _prepareQuoteItems()           → copy line items to order items, apply price rules
    ├── _createOrderPayment()          → copy payment method to order payment
    ├── _createOrder()                 → instantiate and populate the Order entity
    ├── _insertOrderItems()            → save order items
    ├── _notifyCustomer()              → trigger order confirmation email
    └── invalidateQuote()              → mark quote as inactive
```

#### Order Placement Failures

**Failure category 1: Payment declined**

```php
<?php
use Magento\Framework\Exception\CouldNotSaveException;

// In your payment command
try {
    $gatewayResponse = $this->paymentGateway->authorize($amount);
    if (!$gatewayResponse->isSuccessful()) {
        throw new CouldNotSaveException(
            __($gatewayResponse->getDeclineMessage() ?? 'Payment authorization failed.')
        );
    }
} catch (CouldNotSaveException $e) {
    // Already a CouldNotSaveException, re-throw
    throw $e;
} catch (\Exception $e) {
    throw new CouldNotSaveException(
        __('Unable to process payment. Please try again.')
    );
}
```

**On payment failure:** The quote remains in its current state. No items are removed, no totals are changed. The customer stays on the payment step and Magento displays the error message.

**Failure category 2: Inventory check failure**

```php
<?php
use Magento\InventorySalesApi\Api\CheckItemsAvailabilityInterface;

public function placeOrder(int $cartId): int
{
    $quote = $this->cartRepository->get($cartId);
    $items = $quote->getItems();

    foreach ($items as $item) {
        $sku = $item->getSku();
        $qty = $item->getQty();

        $result = $this->checkItemsAvailability->execute([$sku], $qty);
        if (!$result->isAvailable($sku)) {
            throw new NoSuchEntityException(
                __('The requested qty is not available for product "%1".', $sku)
            );
        }
    }

    // ... proceed with order
}
```

**On inventory failure:** The `NoSuchEntityException` is caught by the checkout controller and translated to a user-facing message showing which SKU is out of stock.

**Failure category 3: Address validation failure**

```php
<?php
use Magento\Customer\Model\Address\Validator as CustomerAddressValidator;
use Magento\Framework\Exception\LocalizedException;

public function validateAddresses(Quote $quote): void
{
    $shippingAddress = $quote->getShippingAddress();
    $errors = $this->addressValidator->validate($shippingAddress);

    if (!empty($errors)) {
        throw new LocalizedException(
            __('Please verify the shipping address: %1', implode(', ', $errors))
        );
    }
}
```

**On address validation failure:** The exception is caught on the checkout controller and displayed on the shipping step before the customer can proceed to payment.

#### Payment Verification and 3D Secure

For payment methods that require additional authentication (3D Secure, AVS, CVV checks):

```xml
<!-- etc/di.xml -->
<preference for="Magento\Payment\Api\PaymentVerificationInterface"
           type="Training\Payment\Verification\Training3DSecure"/>
```

```php
<?php
declare(strict_types=1);

namespace Training\Payment\Verification;

use Magento\Payment\Api\PaymentVerificationInterface;
use Magento\Payment\Model\InfoInterface;

class Training3DSecure implements PaymentVerificationInterface
{
    public function verify(
        InfoInterface $payment,
        array $gatewayResponse
    ): bool {
        $eci = $gatewayResponse['eci'] ?? null;
        $cavv = $gatewayResponse['cavv'] ?? null;
        $xid = $gatewayResponse['xid'] ?? null;

        // 3D Secure authentication required for Visa/Mastercard
        if (in_array($eci, ['02', '05', '06'])) {
            return true; // Fully authenticated
        }

        if (in_array($eci, ['01', '07'])) {
            // Attempted authentication, not authenticated
            $this->logger->warning('3D Secure authentication failed', [
                'eci' => $eci,
                'payment_id' => $payment->getId(),
            ]);
            return false;
        }

        // No 3D Secure — apply AVS/CVV rules
        return $this->applyAvsRules($gatewayResponse);
    }

    private function applyAvsRules(array $response): bool
    {
        $avsResult = $response['avs_result'] ?? 'unknown';

        if ($avsResult === 'N') {
            return false; // AVS failure — address mismatch
        }

        return true; // Proceed with caution
    }
}
```

#### Inventory Reservation (MSI)

**Why MSI Matters for Commerce**

Before Magento 2.3, inventory was a simple in_stock flag. MSI (Multi-Stock Inventory) introduced the concept of "sources" (warehouses, stores) and "stocks" (logical groupings of sources). This enables:
- Ship-from-store scenarios
- Real-time inventory across multiple warehouses
- Backorder management with source priority

**The Two Quantity Concepts: `qty` vs `salable_quantity`**

This distinction is critical and causes confusion:

| Concept | What It Means | Table | Use |
|---------|--------------|-------|-----|
| `qty` | Physical stock at a specific source | `inventory_quantity` (source item) | Actual warehouse count |
| `salable_quantity` | Available to sell (minus reservations) | Calculated dynamically | What customer can order |

**Salable Quantity Calculation:**

```php
<?php
use Magento\InventorySalesApi\Api\GetSalableQuantityDataBySkuInterface;

public function getAvailableQty(string $sku): float
{
    /** @var \Magento\InventorySalesApi\Api\Data\SalableQuantityInterface $salableQty */
    $salableQty = $this->getSalableQuantity->execute($sku);

    // Returns the TOTAL salable quantity across all stocks
    return $salableQty->getQuantity();
}

// For per-stock analysis:
public function getQtyPerStock(string $sku, int $stockId): float
{
    $sources = $this->getSourcesByStockId->execute($stockId);
    $total = 0.0;

    foreach ($sources as $source) {
        $sourceItem = $this->getSourceItemBySourceCodeAndSku->execute(
            $source->getSourceCode(),
            $sku
        );
        $total += $sourceItem->getQuantity();
    }

    // Subtract reservations
    $reservations = $this->getReservationsBySku->execute($sku);
    return $total - $reservations;
}
```

**The Reservation System: How It Prevents Overselling**

Reservations work as a "soft hold" on inventory:

```
Order placed → Reservation created (qty reserved, not deducted from source)
    ↓
Shipment created → Reservation CONVERTS to actual deduction from source
    ↓
Order cancelled/refunded → Reservation RELEASED (qty returned to salable)
```

**Why This Design?**

1. **Prevents overselling:** No two orders can reserve the same inventory
2. **Enables partial ship:** Can reserve 3 of 5 ordered items while 2 are backordered
3. **Supports split shipments:** Each shipment only decrements what it actually ships

**Reservation Tables:**

| Table | Purpose |
|-------|---------|
| `inventory_reservation` | Stores pending reservations (order_id, sku, quantity, stock_id) |
| `inventory_source_item` | Physical quantity per source per SKU |
| `inventory_stock` | Logical stock definitions |

**PlaceReservationsInterface — The Core MSI API:**

```php
<?php
use Magento\InventorySalesApi\Api\PlaceReservationsInterface;
use Magento\InventorySalesApi\Api\Data\SalesEventInterface;
use Magento\InventorySalesApi\Api\Data\SalesEventInterfaceFactory;
use Magento\InventorySalesApi\Api\Data\ItemToSellInterface;
use Magento\InventorySalesApi\Api\Data\ItemToSellInterfaceFactory;

/**
 * Places a reservation for items being sold.
 * Called during order placement, BEFORE the order is saved.
 *
 * @param array $items Array of ItemToSellInterface
 * @param SalesEventInterface $salesEvent The type of sale event (order placed, shipment created, etc.)
 */
public function placeReservations(array $items, SalesEventInterface $salesEvent): void;
```

**SalesEvent Types:**

| Event Type | When Fired | Purpose |
|------------|------------|---------|
| `EVENT_ORDER_PLACED` | Order placed | Creates reservation |
| `EVENT_ORDER_CANCELLED` | Order cancelled | Releases reservation |
| `EVENT_SHIPMENT_CREATED` | Shipment created | Converts reservation to actual deduction |
| `EVENT_CREDITMEMO_CREATED` | Credit memo created | Adjusts reservation |

**Checking Availability Before Order:**

```php
<?php
use Magento\InventorySalesApi\Api\CheckItemsAvailabilityInterface;

public function checkStock(array $skus, array $qtys): array
{
    $result = $this->checkItemsAvailability->execute($skus, $qtys);

    $unavailable = [];
    foreach ($skus as $sku) {
        if (!$result->isAvailable($sku)) {
            $unavailable[] = [
                'sku' => $sku,
                'requested' => $qtys[$sku],
                'available' => $result->getQty($sku),
            ];
        }
    }

    return $unavailable;
}
```

**Pro Tip: Why Your Custom Code Might Show Wrong Stock**

If you're checking `cataloginventory_stock_item` directly instead of using `GetSalableQuantityDataBySkuInterface`, you'll get stale data. Always use the salable quantity API:

```php
<?php
// WRONG — may show qty even though it's reserved
$stockItem = $this->stockRegistry->getStockItem($productId);
if ($stockItem->getQty() > $orderedQty) { ... }

// RIGHT — uses MSI reservation system
$salableQty = $this->getSalableQuantity->execute($sku);
if ($salableQty->getQuantity() >= $orderedQty) { ... }
```

#### Order State Machine

Magento's order state machine defines the valid states and transitions:

```
Pending / Pending Payment
    ├── Payment received → Processing
    ├── Payment confirmed → Processing
    └── Payment failed → Canceled / Closed

Processing
    ├── Shipment created → Processing (or Complete if fully shipped)
    ├── Invoice created → Processing (or Complete if fully invoiced)
    └── Hold placed → On Hold

On Hold
    ├── Hold removed → Processing
    └── Canceled → Canceled

Complete (terminal state)
Canceled (terminal state)
Closed (terminal state — for refunds)
```

**State vs Status:**

| Concept | What It Is | Example |
|---------|-----------|---------|
| `state` | Internal state machine state | `processing`, `complete`, `canceled` |
| `status` | Custom label tied to state | `Awaiting Payment`, `Partially Shipped`, `Ready for Pickup` |

You can add custom statuses in `sales_order_status` table, but states are hardcoded in `\Magento\Sales\Model\Order`.

**Order Status Transitions — The `can` Methods:**

```php
<?php
/** @var \Magento\Sales\Model\Order $order */

// Can we invoice?
if ($order->canInvoice()) { ... }

// Can we ship (partially)?
if ($order->canShip()) { ... }

// Can we hold?
if ($order->canHold()) { ... }

// Can we cancel (before shipment/invoice)?
if ($order->canCancel()) { ... }

// Can we creditmemo (refund)?
if ($order->canCreditmemo()) { ... }
```

**The `canCreditmemo()` Logic (Often Misunderstood):**

```php
// canCreditmemo() returns true if ANY of these conditions are met:
// 1. Order has invoices AND has NOT been fully refunded
// 2. Order uses a payment method that doesn't require invoice (e.g., paypal)
// 3. Order is in 'closed' state (full refund already)
// 4. Order has a `creditmemo` flag set

// Returns false if:
// - Order has no invoices and payment doesn't allow creditmemo
// - Order is already fully refunded
```

#### Shipment Creation Patterns

**Why Shipments Matter Beyond Shipping:**

A shipment in Magento is not just a tracking number. It's the trigger for:
1. Inventory deduction (converting reservation to actual source deduction)
2. Order status update (partial or full shipment)
3. Revenue recognition (for partially invoiced orders)
4. Carrier integration (tracking number storage)

**Creating a Shipment Programmatically:**

```php
<?php
use Magento\Sales\Api\ShipmentRepositoryInterface;
use Magento\Sales\Model\Order\ShipmentFactory;
use Magento\Sales\Model\Order\Shipment\ItemFactory;
use Magento\Framework\DB\Transaction;

public function createShipment(
    Order $order,
    array $qtysByItemId = [],
    string $trackingNumber = '',
    string $carrierCode = 'custom',
    string $title = 'Custom Carrier'
): Shipment {
    // Validate order can ship
    if (!$order->canShip()) {
        throw new \Magento\Framework\Exception\LocalizedException(
            __('Order %1 cannot be shipped.', $order->getIncrementId())
        );
    }

    /** @var \Magento\Sales\Model\Order\Shipment $shipment */
    $shipment = $this->shipmentFactory->create($order);

    // Add items
    foreach ($order->getItems() as $orderItem) {
        if ($orderItem->getQtyToShip() > 0 && !$orderItem->getIsVirtual()) {
            $shipmentItem = $this->shipmentItemFactory->create();
            $shipmentItem->setOrderItem($orderItem);
            $shipmentItem->setQty($qtysByItemId[$orderItem->getId()] ?? $orderItem->getQtyToShip());
            $shipment->addItem($shipmentItem);
        }
    }

    // Register shipment BEFORE saving (this triggers inventory reservation conversion)
    $shipment->register();

    // Add tracking
    if ($trackingNumber) {
        $track = $this->trackFactory->create()
            ->setNumber($trackingNumber)
            ->setCarrierCode($carrierCode)
            ->setTitle($title);
        $shipment->addTrack($track);
    }

    // Save with transaction for atomicity
    /** @var Transaction $transaction */
    $transaction = $this->transactionFactory->create()
        ->addObject($shipment)
        ->addObject($order);

    $transaction->save();

    // Send shipment email
    $this->shipmentSender->send($shipment);

    return $shipment;
}
```

**Why `register()` is Critical:**

The `register()` method on shipment does several important things:
1. Sets shipment state to `complete` or `processing` based on whether order is fully shipped
2. **Converts inventory reservations to actual source deductions** — this is the critical inventory step
3. Updates `qty_shipped` on order items
4. Triggers `sales_order_shipment_save_after` event

**Without `register()`, inventory is never deducted from your sources.**

**Partial Shipment Pattern:**

```php
<?php
// Ship only 3 of 5 ordered items
$partialQtys = [
    $orderItem1->getId() => 3,
    $orderItem2->getId() => 0, // Don't ship this item
];

$shipment = $this->createShipment($order, $partialQtys);

// Order still has 2 items pending shipment
// Remaining reservation stays until next shipment or order cancellation
```

#### Credit Memo Flows

**Credit Memo = Refund Record**

A credit memo in Magento is a document that records the refund amount and triggers the actual refund process. It does NOT directly issue the refund — it initiates the flow that ends in money returning to the customer.

**The Credit Memo Flow:**

```
Credit Memo Created
    ├── Refund to original payment method (online)
    ├── Refund to store credit
    ├── Refund to offline (cash/check)
    └── Partial refund (specific items or amount)
```

**Creating a Credit Memo:**

```php
<?php
use Magento\Sales\Api\CreditmemoRepositoryInterface;
use Magento\Sales\Model\Order\CreditmemoFactory;
use Magento\Sales\Model\Order\Creditmemo\ItemFactory;
use Magento\Sales\Model\Order\Creditmemo\CommentFactory;

public function createCreditmemo(
    Order $order,
    array $qtysByItemId = [],
    float $adjustment = 0.0,
    float $adjustmentNegative = 0.0
): Creditmemo {
    // Validate order can creditmemo
    if (!$order->canCreditmemo()) {
        throw new \Magento\Framework\Exception\LocalizedException(
            __('Order %1 is not eligible for creditmemo.', $order->getIncrementId())
        );
    }

    /** @var \Magento\Sales\Model\Order\Creditmemo $creditmemo */
    $creditmemo = $this->creditmemoFactory->createByOrder($order);

    // Set specific item qtys for partial refund
    if (!empty($qtysByItemId)) {
        foreach ($creditmemo->getAllItems() as $creditmemoItem) {
            $orderItemId = $creditmemoItem->getOrderItemId();
            if (isset($qtysByItemId[$orderItemId])) {
                $creditmemoItem->setQty($qtysByItemId[$orderItemId]);
            }
        }
    }

    // Add adjustment (fees) or negative adjustment (refunds on shipping)
    $creditmemo->setAdjustment($adjustment);       // Positive = fee deducted from refund
    $creditmemo->setAdjustmentNegative($adjustmentNegative); // Positive = additional refund

    // Calculate totals
    $creditmemo->collectTotals();

    // Register — this triggers the actual refund
    $creditmemo->register();

    // Save
    $this->creditmemoRepository->save($creditmemo);

    // Add comment
    $comment = $this->commentFactory->create()
        ->setCreditmemo($creditmemo)
        ->setComment(__('Refund processed automatically.'))
        ->setIsVisibleOnFront(true);

    return $creditmemo;
}
```

**Online vs Offline Refund:**

| Type | How It Works | Use Case |
|------|--------------|---------|
| `online` | Calls payment gateway to refund (e.g., Stripe, PayPal) | Credit card, PayPal — money returns to card |
| `offline` | No gateway call — Magento records refund only | Cash, check, store credit |

**Refund to Store Credit:**

```php
<?php
use Magento\CustomerBalance\Model\Balance;
use Magento\CustomerBalance\Model\BalanceFactory;
use Magento\CustomerBalance\Api\CustomerBalanceRepositoryInterface;

public function refundToStoreCredit(Order $order, float $amount): void
{
    $customerId = $order->getCustomerId();

    /** @var Balance $balance */
    $balance = $this->balanceFactory->create();
    $balance->setCustomerId($customerId);
    $balance->setWebsiteId($order->getStore()->getWebsiteId());
    $balance->setAmount($amount);
    $balance->setHistoryAction(Balance::ACTION_REFUNDED);
    $balance->save();

    // Notify customer
    $this->customerBalanceNotification->notify($balance);
}
```

**Why Inventory Returns on Credit Memo:**

When a credit memo is created for an item, Magento automatically adds back to the salable quantity:

```php
<?php
// This happens automatically via the creditmemo registration
// 1. Creates a negative reservation (effectively +qty back to salable)
// 2. If source item tracking is enabled, increases source item qty

// For MSI, creates a reservation with EVENT_CREDITMEMO_CREATED
// which adds qty back to the salable qty pool
```

#### Rollback on Failure

When order placement fails after inventory has been reserved:

```php
<?php
use Magento\InventoryReservationsApi\Api\RevertReservationsInterface;
use Magento\InventorySalesApi\Api\PlaceReservationsInterface;

public function rollbackOnFailure(
    Quote $quote,
    int $orderId,
    \Exception $failureReason
): void {
    $this->logger->error('Order placement failed, rolling back', [
        'order_id' => $orderId,
        'quote_id' => $quote->getId(),
        'error' => $failureReason->getMessage(),
    ]);

    // Void the payment authorization
    if ($quote->getPayment()) {
        $payment = $quote->getPayment();
        if ($payment->getAuthorizationTransaction()) {
            $payment->getAuthorizationTransaction()->void();
        }
    }

    // Revert inventory reservations
    $this->revertReservations->execute([$quote->getId()]);

    // Re-activate the quote
    $quote->setIsActive(true);
    $quote->setReservedOrderId(null);
    $quote->save();
}
```

#### Post-Place Workflows

After a successful order placement, Magento triggers several post-placement workflows:

**1. Order confirmation email:**

```php
<?php
// Triggered by event: sales_order_place_after
// Handled by: Magento\Sales\Model\Order\Email\Sender\OrderSender
public function sendOrderConfirmationEmail(Order $order): void
{
    $this->orderEmailSender->send($order);
}
```

**2. Invoice generation automation:**

```php
<?php
// For payment methods that support authorize-and-capture, automatically generate invoice:
// Triggered after order save via observer
<event name="sales_order_save_commit_after">
    <observer name="training_auto_invoice"
              instance="Training\Checkout\Observer\AutoInvoice"
              shared="false"/>
</event>
```

```php
<?php
declare(strict_types=1);

namespace Training\Checkout\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Api\InvoiceRepositoryInterface;
use Magento\Sales\Model\Order;
use Magento\Sales\Model\Service\InvoiceService;

class AutoInvoice implements ObserverInterface
{
    private InvoiceService $invoiceService;
    private InvoiceRepositoryInterface $invoiceRepository;

    public function __construct(
        InvoiceService $invoiceService,
        InvoiceRepositoryInterface $invoiceRepository
    ) {
        $this->invoiceService = $invoiceService;
        $this->invoiceRepository = $invoiceRepository;
    }

    public function execute(Observer $observer): void
    {
        /** @var Order $order */
        $order = $observer->getOrder();

        if (!$order->getId() || $order->hasInvoices()) {
            return;
        }

        if ($order->getPayment()->getMethod() !== 'training_offline') {
            return;
        }

        if (!$order->canInvoice()) {
            return;
        }

        $invoice = $this->invoiceService->prepareInvoice($order);
        $invoice->register();
        $invoice->setState(\Magento\Sales\Model\Order\Invoice::STATE_PAID);

        $this->invoiceRepository->save($invoice);
    }
}
```

---

## Reference Exercises

**Exercise 1:** Create a CLI command `bin/magento training:cart:dump` that loads the current session quote and prints all items (SKU, qty, price) to console.

**Exercise 2:** Build a custom shipping carrier `TrainingShipping` that offers a flat $5.99 flat rate with free shipping for orders over $100. Use `FlatRate` as your template. Configure the threshold in `config.xml`.

**Exercise 3:** Build a custom payment method `training_offline` with a simple "mark as paid" flow using `authorize()` that always succeeds. The method should appear in checkout and store the `transaction_id` in `additional_information`. Add admin configuration for enabling/disabling and setting a title.

**Exercise 4:** Add a custom total `handling_fee` ($2.99 flat) to every quote via `sales_quote_address_total_collect` observer. Register the amount as `handling_fee` in the totals collector output so it appears separately on the order review page.

**Exercise 5:** Add a custom checkbox "gift wrap" to the shipping step that adds $3.99 to the order total when checked. Store the gift wrap selection in a quote extension attribute, and display the $3.99 as a separate line item in the totals.

**Exercise 6:** Write a plugin on `PlaceOrderInterface::execute()` that logs the order ID and customer email if the order total exceeds $500. The plugin should log to `var/log/training-highvalue-orders.log`.

**Exercise 7 (optional):** Build a custom checkout step that collects a purchase order number before payment. The step should appear between "Shipping Method" and "Payment". The PO number must be stored in a quote extension attribute and must appear on the order after placement. Validate the PO number format as `PO-\d{4,}` on the client side.

**Exercise 8 (inventory):** Create a CLI command that checks the salable quantity for a given SKU using `GetSalableQuantityDataBySkuInterface` and compares it to the sum of source quantities from `inventory_source_item`. Explain why they differ.

**Exercise 9 (shipments):** Create a CLI command that ships a specific order by creating a shipment with tracking information. The command should verify `canShip()` before attempting shipment creation.

**Exercise 10 (credit memo):** Create a CLI command that creates a credit memo for a specific order, refunding a specific set of items and an optional adjustment amount. Verify the reservation system is properly updated.

---

## Reading List

| Resource | Why Read It |
|----------|-------------|
| `Magento/Checkout/etc/di.xml` — `PlaceOrderInterface` wiring | Understand how order placement is assembled in the service layer |
| `Magento/Quote/Model/QuoteManagement.php` — `submit()` method | The canonical quote-to-order conversion implementation |
| `Magento/Sales/etc/di.xml` — `TotalCollector` registration | See the full collector stack and their priorities |
| `Magento/Shipping/Model/Carrier/AbstractCarrier.php` — `collectRates()` | Base class for all shipping carriers |
| `Magento/Payment/Model/Method/Adapter.php` — full implementation | The default adapter all payment methods inherit from |
| `Magento/Tax/etc/di.xml` — `TaxCalculation` configuration | How tax calculation is integrated into the totals pipeline |
| `Magento/InventorySales/Model/PlaceReservations.php` | MSI inventory reservation on order placement |
| Alan Storm's "Magento 2 Checkout Architecture" (blog.algolia.com/alan-storm) | Deep-dive into the modern checkout JS/Knockout component architecture |
| Magento DevDocs — "Order Processing" (devdocs.magento.com) | Official documentation for order placement and management |

---

## Edge Cases & Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Quote totals not recalculating after item add | `collectTotals()` not triggered | Use `CartItemRepositoryInterface::save()` which auto-triggers; avoid direct `Quote::addItem()` |
| Shipping methods not appearing in checkout | Carrier disabled or `collectRates()` returning `false` | Check `carriers/<code>/active` config flag; add logging to carrier's `collectRates()` |
| Payment method not appearing in checkout | `payment_method_is_active` observer blocking it | Add logging to the observer; check the `$result->isAvailable` flag |
| Order placed but quote still active | `invalidateQuote()` not called or failed | Check `PlaceOrderInterface` implementation; ensure quote is set `setIsActive(false)` |
| Duplicate orders from double-submit | Race condition in order placement | Use `Quote::reserveOrderId()` before conversion; idempotency key pattern |
| Tax total is zero even with taxable product | Tax settings not configured, or tax rule not matched | Verify `STORES > Config > Sales > Tax` settings; check `tax_class_id` on product |
| Extension attribute returns null after save | `extension_attributes` not properly set | Use `getExtensionAttributes()` with null-check and `??` fallback to factory create |
| Free shipping not applying at threshold | Threshold check comparing wrong field | Ensure `getPackageValueWithDiscount()` is used, not `getPackageValue()` |
| Payment `additional_information` empty after order | Data stored before quote-to-order conversion | Ensure data is set on `$payment` object before `PlaceOrderInterface::execute()` is called |
| Custom checkout step not saving data | Server controller returning non-JSON | The checkout JS expects `{ success: true, ... }` JSON; non-JSON causes silent failure |
| Product shows in-stock but order fails | Checking legacy stock table, not MSI salable qty | Use `GetSalableQuantityDataBySkuInterface` instead of `StockRegistry` |
| Shipment created but inventory not deducted | Shipment not registered, or using legacy inventory | Call `$shipment->register()`; use MSI `PlaceReservationsInterface` |
| Credit memo doesn't return inventory | Legacy `cataloginventory` module not fully migrated | Ensure `inventory_reservation` entries are created on creditmemo |
| Order shows `canCreditmemo=false` but refund needed | Order has no invoice or payment doesn't support offline refund | Check `sales_order_payment.method`; some payments require online refund |
| `qty` vs `salable_quantity` mismatch | Reservations exist but not yet converted | Reservations are temporary holds; only `salable_quantity` is accurate for selling |

---

## Common Mistakes to Avoid

**1. Calling `$quote->save()` directly**  
The quote has a complex aggregate lifecycle. Direct saves bypass observers, total recalculation triggers, and event propagation. Always use `CartRepositoryInterface::save()` or let the repository-managed operations handle persistence.

**2. Modifying totals without triggering `collectTotals()`**  
If you set `$quote->getShippingAddress()->setGrandTotal(100)` without calling `$quote->collectTotals()`, other totals will be stale and the order total won't match the sum of line items.

**3. Not handling the guest quote case**  
Guest quotes have no `customer_id`. When loading a quote by customer ID using `CartRepositoryInterface::getForCustomer()`, it will return null for guests. Use `Checkout\Session::getQuote()` for the session-scoped approach that handles both cases.

**4. Creating a payment method without implementing `canUseForCountry()`**  
Returning `false` for the current country silently hides the payment method. Always check `canUseForCountry()` and return `true` for all countries you intend to support, or it will disappear without any error.

**5. Adding checkout step JS without a corresponding server-side save action**  
Knockout.js validates and submits data, but if there is no server endpoint to persist the step data to the quote, the data is lost when the customer proceeds to the next step. Every custom step needs both a JS view model and a corresponding `HttpPost` controller.

**6. Not registering a total collector in `di.xml`**  
Creating the collector class is not enough. The collector must be registered in the `Magento\Quote\Model\Quote\Address\Total\Collector` type with its code as the array key, otherwise it will never be invoked in the `collectTotals()` pipeline.

**7. Forgetting `areaCode` in console/DI contexts**  
`Magento\Framework\App\State::getAreaCode()` is required by some core models (particularly shipping carriers and payment methods that inherit from `AbstractCarrier`). In console commands, the area is not automatically set. Either inject `State` and call `$state->setAreaCode(Area::AREA_FRONTEND)` or use `AreaList` to resolve the current area before using those models.

**8. Using `sales_order_place_after` for inventory operations that must be atomic**  
This event fires after the order is saved. If you reserve inventory in this observer and a later operation fails, the inventory reservation is orphaned. Use `inventory_reservation` mutations (via `PlaceReservationsInterface`) which are part of the transaction or use the explicit rollback path.

---

*Magento 2 Backend Developer Course — Topic 10 | Checkout & Inventory*
