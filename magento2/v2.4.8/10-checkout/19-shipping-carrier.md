---
title: "19 - Shipping Architecture & Carrier Integration"
description: "Magento 2.4.8 shipping system: CarrierInterface, rate calculation engine, shipping method architecture, destination validation, and shipping management."
tags: magento2, shipping, carrier, rate-calculation, shipping-method, CarrierInterface, freeshipping, flatrate, ups-fedex-dhl
rank: 19
pathways: [magento2-deep-dive]
see_also:
  - path: "10-checkout/README.md"
    description: "Checkout overview — shipping step integration"
  - path: "10-checkout/03-session-cookie-management.md"
    description: "Session management — quote shipping information"
---

# Shipping Architecture & Carrier Integration

Magento's shipping system is a pluggable architecture where carriers (FedEx, UPS, DHL, USPS, or custom) register themselves via `CarrierInterface` and provide rates on demand. Understanding this architecture is prerequisite to adding a new shipping method, customizing rate display, or debugging shipping calculation issues.

---

## 1. Shipping Architecture Overview

### The Shipping Flow

```
Cart Page (Shipping Step)
    ↓
Magento collects: destination address, cart weight, available shipping methods
    ↓
For each carrier (enabled):
    CarrierInterface::collectRates(RateRequest)
        ↓
    Returns: ShippingRate[] (each rate = one shipping method option)
    ↓
All rates merged and displayed in checkout shipping step
    ↓
Customer selects shipping method
    ↓
Selection stored in Quote (shipping_address.shipping_method)
    ↓
Order placed → shipping method copied to Order
```

### Key Interfaces

```php
<?php
// The primary interface all carriers must implement
// vendor/magento/module-shipping/Model/Carrier/CarrierInterface.php

interface CarrierInterface
{
    // Collect rates for a shipping request
    public function collectRates(RateRequest $request): ?Result;

    // Check if carrier is available for the given address
    public function checkAvailableShipCountries(
        \Magento\Quote\Api\Data\AddressInterface $shippingAddress
    ): bool;

    // Return allowed shipping methods (for admin display)
    public function getAllowedMethods(): array;
}
```

### Core Classes

| Class | Role |
|-------|------|
| `\Magento\Shipping\Model\CarrierFactory` | Creates carrier instance by code |
| `\Magento\Shipping\Model\Rate\Result` | Container for shipping rates |
| `\Magento\Shipping\Model\Rate\ResultFactory` | Creates Result objects |
| `\Magento\Quote\Model\Quote\Address\RateRequest` | Input to collectRates() |
| `\Magento\Shipping\Model\Rate\ShippingRate` | Individual rate (one method option) |
| `\Magento\Checkout\Model\Session` | Holds quote with shipping address |

---

## 2. CarrierInterface Deep Dive

### The collectRates Method

```php
<?php
// \Magento\Shipping\Model\Carrier\AbstractCarrierInterface
// All built-in carriers extend AbstractCarrier and implement CarrierInterface

class Fedex implements \Magento\Shipping\Model\Carrier\CarrierInterface
{
    /**
     * @param RateRequest $request
     * @return Result|null
     */
    public function collectRates(RateRequest $request): ?Result
    {
        if (!$this->canShip()) {
            return null;  // Carrier not available
        }

        /** @var Result $result */
        $result = $this->_rateFactory->create();

        // Create rate for each shipping option
        foreach ($this->getShippingMethods() as $methodCode => $methodData) {
            /** @var \Magento\Shipping\Model\Rate\Result $rate */
            $rate = $this->_rateFactory->create();
            $rate->setCarrier($this->_code);
            $rate->setCarrierTitle($this->getConfigData('title'));
            $rate->setMethod($methodCode);
            $rate->setMethodTitle($methodData['label']);
            $rate->setPrice($methodData['price']);
            $rate->setCost($methodData['cost']);

            $result->addRate($rate);
        }

        return $result;
    }
}
```

### RateRequest Object

```php
<?php
// vendor/magento/module-quote/Model/Quote/Address/RateRequest.php

class RateRequest
{
    public $allItems;          // \Magento\Framework\DataObject[] — all cart items
    public $destCountryId;     // Shipping destination country code
    public $destRegionId;      // Shipping destination region ID
    public $destRegionCode;    // Shipping destination region code
    public $destStreet;         // Destination street address
    public $destCity;          // Destination city
    public $destPostcode;      // Destination postal code
    public $packageWeight;      // Total weight of package
    public $packageValue;       // Total value of package (for insurance)
    public $packageCurrency;    // Currency code
    public $marketplaceId;     // For multi-vendor/marketplace
    public $freeMethodWeight;   // Weight threshold for free shipping
}
```

### Result Object

```php
<?php
// vendor/magento/module-shipping/Model/Rate/Result.php

class Result
{
    protected $_rates = [];  // array of ShippingRate objects

    public function addRate(\Magento\Shipping\Model\Rate\Result $rate): self
    {
        $this->_rates[] = $rate;
        return $this;
    }

    public function getRates(): array
    {
        return $this->_rates;
    }
}
```

### ShippingRate Object

```php
<?php
// vendor/magento/module-shipping/Model/Rate/Result.php

class ShippingRate
{
    public $carrier;        // 'fedex', 'ups', 'flatrate'
    public $carrierTitle;   // Display name: 'FedEx', 'United Parcel Service'
    public $method;         // 'FEDEX_GROUND', 'PRIORITY'
    public $methodTitle;    // 'FedEx Ground', 'FedEx Priority'
    public $price;          // Cost to customer
    public $cost;           // Actual cost to merchant
    public $waybill;        // Tracking number (if already assigned)
    public $maxDeliveryDate; // Estimated delivery date
    public $minDeliveryDate;
}
```

---

## 3. Built-in Carriers

### Flat Rate

Flat rate is the simplest carrier — fixed price regardless of destination or weight:

```php
<?php
// \Magento\Shipping\Model\Carrier\Flatrate
class Flatrate implements CarrierInterface
{
    public function collectRates(RateRequest $request): ?Result
    {
        // Get flat rate price from config
        $price = $this->getConfigData('price');

        // Handle handling fee (extra charge on top of shipping)
        $handling = (float) $this->getConfigData('handling_fee');
        $finalPrice = $price + $handling;

        // If free shipping threshold met
        if ($this->getConfigData('free_shipping_enable') &&
            $request->getPackageValueWithDiscount() >= $this->getConfigData('free_shipping_threshold')
        ) {
            $finalPrice = 0;
        }

        $rate = $this->_rateFactory->create();
        $rate->setCarrier($this->_code);
        $rate->setMethod('flatrate');
        $rate->setMethodTitle('Flat Rate');
        $rate->setPrice($finalPrice);
        $rate->setCost(0);  // Free shipping cost to merchant

        return $result;
    }
}
```

### Table Rate

Table rate uses a CSV-based matrix (destination vs weight/price) to determine shipping cost:

```sql
-- Import CSV format for table rate:
Website,Country,Region/State,Zip/Postal Code,"Subtotal (and above)",Weight (and below),Shipping Price
*       ,*      ,*              ,*                   ,0                       ,5               ,5.95
*       ,*      ,*              ,*                   ,0                      ,10              ,8.95
*       ,US     ,*              ,*                   ,0                      ,100             ,12.95
```

Table rate implementation:
```php
<?php
// vendor/magento/module-shipping/Model/Carrier/Tablerate.php
class Tablerate extends \Magento\Shipping\Model\Carrier\AbstractCarrier
    implements \Magento\Shipping\Model\Carrier\CarrierInterface
{
    // Reads CSV from 'tablerate.csv' in module
    // Matches against: country, region, postcode, subtotal, weight
    // Returns matching price
}
```

### Free Shipping

```php
<?php
// vendor/magento/module-shipping/Model/Carrier/Freeshipping.php

class Freeshipping implements CarrierInterface
{
    public function collectRates(RateRequest $request): ?Result
    {
        // Only available when subtotal meets threshold
        $subtotal = $request->getPackageValueWithDiscount();

        if ($subtotal >= $this->getConfigData('free_shipping_subtotal')) {
            // Show free shipping
            $rate = $this->_rateFactory->create();
            $rate->setCarrier($this->_code);
            $rate->setMethod('freeshipping');
            $rate->setMethodTitle('Free Shipping');
            $rate->setPrice(0);
            $rate->setCost(0);

            return $result;
        }

        return false;  // Don't show free shipping method
    }
}
```

---

## 4. Adding a Custom Carrier

### Step 1: Create Carrier Class

```php
<?php
// Vendor/Shipping/Model/Carrier/CustomCarrier.php
declare(strict_types=1);

namespace Vendor\Shipping\Model\Carrier;

use Magento\Shipping\Model\Carrier\AbstractCarrier;
use Magento\Shipping\Model\Carrier\CarrierInterface;
use Magento\Shipping\Model\Rate\ResultFactory;
use Magento\Quote\Model\Quote\Address\RateRequest;
use Magento\Quote\Model\Quote\Address\RateResult\ErrorFactory;
use Psr\Log\LoggerInterface;

class CustomCarrier extends AbstractCarrier implements CarrierInterface
{
    protected $_code = 'customcarrier';

    protected ResultFactory $rateFactory;
    protected LoggerInterface $logger;

    public function __construct(
        \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig,
        \Magento\Shipping\Model\Rate\ErrorFactory $rateErrorFactory,
        LoggerInterface $logger,
        ResultFactory $rateFactory,
        array $data = []
    ) {
        parent::__construct($scopeConfig, $rateErrorFactory, $logger, $data);
        $this->rateFactory = $rateFactory;
        $this->logger = $logger;
    }

    public function collectRates(RateRequest $request): ?\Magento\Shipping\Model\Rate\Result
    {
        if (!$this->canShip()) {
            return null;
        }

        /** @var \Magento\Shipping\Model\Rate\Result $result */
        $result = $this->_rateFactory->create();

        // Standard shipping option
        $rate = $this->createRate();
        $rate->setMethod('standard');
        $rate->setMethodTitle('Standard Shipping');
        $rate->setPrice($this->calculateStandardRate($request));
        $rate->setCost($this->calculateStandardCost($request));
        $result->addRate($rate);

        // Express shipping option
        $rate = $this->createRate();
        $rate->setMethod('express');
        $rate->setMethodTitle('Express Shipping');
        $rate->setPrice($this->calculateExpressRate($request));
        $rate->setCost($this->calculateExpressCost($request));
        $result->addRate($rate);

        return $result;
    }

    public function getAllowedMethods(): array
    {
        return [
            'standard' => 'Standard Shipping',
            'express' => 'Express Shipping',
        ];
    }

    private function createRate(): \Magento\Shipping\Model\Rate\Result
    {
        /** @var \Magento\Shipping\Model\Rate\Result $rate */
        $rate = $this->_rateFactory->create();
        $rate->setCarrier($this->_code);
        $rate->setCarrierTitle($this->getConfigData('title') ?: 'Custom Carrier');
        return $rate;
    }

    private function calculateStandardRate(RateRequest $request): float
    {
        $baseRate = 5.95;
        $perPound = 0.50;
        return $baseRate + ($request->getPackageWeight() * $perPound);
    }

    private function calculateExpressRate(RateRequest $request): float
    {
        $baseRate = 15.95;
        $perPound = 1.25;
        return $baseRate + ($request->getPackageWeight() * $perPound);
    }

    private function calculateStandardCost(RateRequest $request): float
    {
        return $this->calculateStandardRate($request) * 0.7;  // 30% margin
    }

    private function calculateExpressCost(RateRequest $request): float
    {
        return $this->calculateExpressRate($request) * 0.65;  // 35% margin
    }
}
```

### Step 2: Declare in di.xml

```xml
<!-- etc/di.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/di.xsd">

    <type name="Magento\Shipping\Model\CarrierFactory">
        <arguments>
            <argument name="carriers" xsi:type="array">
                <item name="customcarrier" xsi:type="string">Vendor\Shipping\Model\Carrier\CustomCarrier</item>
            </argument>
        </arguments>
    </type>

</config>
```

### Step 3: Add System Configuration

```xml
<!-- etc/adminhtml/system.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Config:etc/system.xsd">
    <section id="carriers" translate="label" type="text" sortOrder="320" showInDefault="1">
        <group id="customcarrier" translate="label" type="text" sortOrder="10">
            <label>Custom Carrier</label>
            <field id="active" translate="label" type="select" sortOrder="1">
                <label>Enabled</label>
                <source_model>Magento\Config\Model\Config\Source\Yesno</source_model>
            </field>
            <field id="title" translate="label" type="text" sortOrder="2">
                <label>Title</label>
            </field>
            <field id="base_price" translate="label" type="text" sortOrder="3">
                <label>Base Price</label>
                <validate>validate-number</validate>
            </field>
        </group>
    </section>
</config>
```

### Step 4: Enable in Admin

In admin: **Stores → Settings → Configuration → Sales → Shipping Methods → Custom Carrier**

---

## 5. Rate Calculation Logic

### Weight-Based Calculation

```php
<?php
// Common pattern: weight tiers
private function calculateByWeight(RateRequest $request): float
{
    $weight = $request->getPackageWeight();

    if ($weight <= 1) {
        return 5.95;
    } elseif ($weight <= 5) {
        return 9.95;
    } elseif ($weight <= 10) {
        return 14.95;
    } else {
        return 19.95 + ($weight - 10) * 1.50;
    }
}
```

### Destination-Based Calculation

```php
<?php
// Zone-based pricing: regions/countries grouped into zones
private function calculateZoneRate(RateRequest $request): float
{
    $zone = $this->determineZone(
        $request->getDestCountryId(),
        $request->getDestRegionCode(),
        $request->getDestPostcode()
    );

    $zoneRates = [
        'domestic' => 5.95,
        'europe' => 14.95,
        'asia' => 19.95,
        'international' => 29.95
    ];

    return $zoneRates[$zone] ?? 29.95;
}
```

### Handling Fee

Most carriers support a "handling fee" — an additional charge on top of the carrier rate:

```php
<?php
// In collectRates:
$carrierRate = $this->getRateFromApi($request);
$handlingFee = $this->getConfigData('handling_fee');
$handlingAction = $this->getConfigData('handling_action');  // 'per_order' or 'per_box'

if ($handlingAction === 'per_box') {
    $boxCount = ceil($request->getPackageWeight() / $this->getConfigData('max_package_weight'));
    $finalPrice = $carrierRate + ($handlingFee * $boxCount);
} else {
    $finalPrice = $carrierRate + $handlingFee;
}
```

---

## 6. Shipping Address Validation

### Address Validation Before Rate Calculation

```php
<?php
// Check if carrier ships to destination before returning rates
public function checkAvailableShipCountries(
    \Magento\Quote\Api\Data\AddressInterface $shippingAddress
): bool {
    $allowedCountries = $this->getConfigData('sallowed_countries');
    // 'US,CA,MX' or empty = all countries

    if (empty($allowedCountries)) {
        return true;  // All countries allowed
    }

    $destCountry = $shippingAddress->getCountryId();
    $allowedList = array_map('trim', explode(',', $allowedCountries));

    return in_array($destCountry, $allowedList);
}
```

### Limiting by Region/Postcode

```php
<?php
// Also implement allowedRegionCodes or specific postcode rules
public function getAllowedRegionCodes(): ?string
{
    // Comma-separated region codes: 'CA,NY,TX'
    return $this->getConfigData('region_ids');
}

// Use in collectRates to filter before API call
$destRegionId = $request->getDestRegionId();
if (!$this->isRegionAllowed($destRegionId)) {
    return false;  // Don't show this carrier
}
```

---

## 7. Handling API-Based Carriers (FedEx, UPS, DHL)

### API Integration Pattern

```php
<?php
// For carriers that call external APIs (FedEx, UPS, etc.)
class Fedex extends AbstractCarrier implements CarrierInterface
{
    public function collectRates(RateRequest $request): ?Result
    {
        if (!$this->canShip()) {
            return null;
        }

        try {
            // Build API request from RateRequest
            $apiRequest = $this->buildApiRequest($request);

            // Call FedEx API
            $apiResponse = $this->fedexClient->getRates($apiRequest);

            // Parse response into ShippingRate objects
            $rates = $this->parseApiResponse($apiResponse);

            $result = $this->_rateFactory->create();
            foreach ($rates as $rate) {
                $result->addRate($rate);
            }

            return $result;

        } catch (\Exception $e) {
            // Log error and return error (don't break checkout)
            $this->_logger->error('FedEx API error: ' . $e->getMessage());
            $error = $this->_rateErrorFactory->create();
            $error->setCarrier($this->_code);
            $error->setCarrierTitle($this->getConfigData('title'));
            $error->setErrorMessage('Unable to retrieve FedEx rates. Please try again.');
            return $error;
        }
    }
}
```

### API Error Handling

```php
<?php
// If carrier API fails, don't crash — show graceful error
// In checkout, the shipping method simply won't appear (or shows error message)

public function parseErrorResponse($response): void
{
    if ($response['error']) {
        $error = $this->_rateErrorFactory->create();
        $error->setCarrier($this->_code);
        $error->setErrorMessage(
            $response['error_message'] ?: 'Shipping rate unavailable'
        );
        // Result with error means "don't show this carrier"
        $result->addError($error);
    }
}
```

---

## 8. Free Shipping Calculation

### Common Free Shipping Patterns

```php
<?php
// Pattern 1: Subtotal threshold
public function isFreeShippingAvailable(RateRequest $request): bool
{
    $minSubtotal = $this->getConfigData('free_shipping_subtotal');
    $currentSubtotal = $request->getPackageValueWithDiscount();

    return $minSubtotal > 0 && $currentSubtotal >= $minSubtotal;
}

// Pattern 2: Coupon-based free shipping
public function isCouponFreeShipping(string $couponCode): bool
{
    $freeShippingCoupons = ['FREESHIP', 'FREESHIP2024'];
    return in_array($couponCode, $freeShippingCoupons);
}

// Pattern 3: Customer group-based
public function isGroupFreeShipping(\Magento\Customer\Api\Data\CustomerInterface $customer): bool
{
    $freeShippingGroups = [1, 2, 3];  // VIP groups
    return in_array($customer->getGroupId(), $freeShippingGroups);
}
```

---

## 9. Multi-Package Shipping

### Splitting Shipments

```php
<?php
// When cart has items from different warehouses/locations
// Each source gets its own package with separate rates

public function collectRates(RateRequest $request): ?Result
{
    $items = $request->getAllItems();
    $warehouseGroups = $this->groupItemsByWarehouse($items);

    $result = $this->_rateFactory->create();

    foreach ($warehouseGroups as $warehouseId => $warehouseItems) {
        $warehouseRate = $this->calculateWarehouseRate(
            $warehouseId,
            $warehouseItems,
            $request
        );
        $result->addRate($warehouseRate);
    }

    // If splitting isn't possible, return single combined rate
    return $result;
}
```

---

## 10. Common Pitfalls and Debugging

### Pitfall 1: Carrier Not Showing in Checkout

```bash
# Check if carrier is enabled in config
bin/magento config:show carriers/fedex/active
# Should return 1

# Check carrier code matches (case-sensitive!)
# In carrier class: protected $_code = 'fedex';
# In system.xml: <field id="active"> must be under <group id="fedex">

# Check allowed countries
bin/magento config:show carriers/fedex/sallowed_countries
# If set, destination country must be in list
```

### Pitfall 2: Rate Always Returns 0

```php
<?php
// Check: is the rate method returning 0 intentionally?
// Check getPrice() logic — sometimes 0 means "free" not "error"

// If rate always 0, verify:
// 1. Base rate calculation isn't returning 0
// 2. Free shipping threshold met (returns 0 for free)
// 3. Handling fee not adding to 0
```

### Pitfall 3: API Timeout Slowing Down Checkout

```php
<?php
// For external API carriers, implement timeout
$apiResponse = $this->fedexClient->getRates(
    $apiRequest,
    ['timeout' => 5]  // 5 second timeout
);

// If API times out, fall back to flat rate
if ($apiResponse === null) {
    return $this->getFallbackRate();
}
```

### Debugging Shipping Rates

```php
<?php
// Add to checkout shipping step to see raw rate data
// In a custom module's checkout override:

public function afterCollectRates(
    \Magento\Checkout\Model\ShippingInformationManagement $subject,
    $result
) {
    // Inspect all rates
    foreach ($result->getAllRates() as $rate) {
        $this->logger->info('Rate', [
            'carrier' => $rate->getCarrier(),
            'method' => $rate->getMethod(),
            'price' => $rate->getPrice()
        ]);
    }
}
```

### CLI Debugging

```bash
# List all configured carriers
bin/magento shipping:carriers:list

# Test specific carrier rate calculation
bin/magento shipping:rates:fetch --carrier=fedex --country=US --postcode=90210 --weight=2

# View active shipping methods in current cart
# Use XDebug or add temporary logging in collectRates()
```

---

## Reading List

- [Shipping configuration](https://experienceleague.adobe.com/docs/commerce-admin/stores-guides/core-flows/checkout/shipping-config.html)
- [Carrier implementation](https://developer.adobe.com/commerce/php/development/shipping-carriers/)
- [\Magento\Shipping\Model\Carrier\CarrierInterface](https://github.com/magento/magento2/blob/2.4-develop/app/code/Magento/Shipping/Model/Carrier/CarrierInterface.php)

---

## Common Mistakes to Avoid

1. ❌ Not checking `canShip()` in collectRates → Carrier shows when it can't ship to destination
2. ❌ API errors crashing checkout → Always wrap API calls in try/catch, return error gracefully
3. ❌ Hardcoded price → Read from config via `getConfigData()` so merchant can adjust
4. ❌ Wrong carrier code case → `fedex` vs `Fedex` vs `FEDEX` must match exactly
5. ❌ Not clearing cache after enabling carrier → `bin/magento cache:clean config`
6. ❌ API timeout blocking checkout → Always implement timeout, never wait indefinitely

---

*Magento 2 Backend Developer Course — Topic 10 — Checkout*