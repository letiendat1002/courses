---
title: "05 - MSI Architecture"
description: "Magento 2.4.8 MSI: sources, stocks, reservations, source selection algorithm, inventory indexers, and the inventory API"
tags: [magento2, inventory, msi, inventory-api, source-selection, reservations]
rank: 6
pathways: [magento2-deep-dive]
---

# 05 — MSI Architecture

## Chapter Overview

Multi-Source Inventory (MSI) is Magento's answer to a deceptively hard problem: **how do you track inventory accurately across multiple physical locations without overselling a single unit?** Before MSI existed, Magento's inventory system treated stock as a single global pool. That works fine for a single warehouse. It breaks down the moment you have two warehouses, a retail store with hands-on fulfillment, or a dropshipper halfway across the country.

This chapter covers the architectural decisions behind MSI, the three core entities (Source, Stock, Reservation), the Source Selection Algorithm, the inventory indexer, and every API you need to work with inventory programmatically. By the end, you'll understand the full inventory pipeline from a customer placing an order to the moment inventory is decremented across your warehouses.

> **Magento 2.4.6+ note:** The inventory reservation cleanup behavior changed subtly. Exired reservations are now cleaned up via the `inventory.reservation.cleanup` cron job rather than being resolved immediately during checkout. This is important for checkout flow timing and is covered in the Reservations section.

---

## 1. MSI Overview

### 1.1 What is MSI?

MSI (Multi-Source Inventory) is the inventory management system introduced in **Magento 2.3.0** as a replacement for the legacy `CatalogInventory` module. It ships as the `Magento_Inventory` module (and a family of `Magento_Inventory*` modules) and is a core part of Magento's Commerce edition.

The key shift MSI introduces is the **separation of "where things are" from "what can be sold where"**:

| Legacy System (pre-2.3) | MSI (2.3+) |
|------------------------|------------|
| Single global stock pool | Multiple Sources (physical locations) |
| Simple quantity per product | Quantity tracked per Source per Product |
| No concept of fulfillment priority | Stocks define which Sources can fulfill for a Sales Channel |
| No reservation mechanism | Explicit Reservation records prevent overselling |
| `cataloginventory_stock_item` only | `inventory_source`, `inventory_stock`, `inventory_reservation` tables |

MSI is not optional in Magento 2.4.x — it is the only inventory system. The legacy `CatalogInventory` module still exists for backward compatibility (it wraps MSI) but all new inventory logic flows through the Inventory module family.

### 1.2 Why MSI Was Introduced

The legacy system had one fatal flaw: **it could not represent physical reality**. Consider this scenario:

```
Warehouse A: 5 units of SKU-123
Warehouse B: 3 units of SKU-123
Customer orders 6 units
```

Under the legacy system, Magento would see "5 + 3 = 8 units available" and let the order through. But those 6 units span two warehouses. Legacy Magento had no concept of "fulfill this order from Warehouse A first, then Warehouse B." There was no reservation — just a static quantity that got decremented whenever an order was placed, with no concept of *pending* reservation state.

MSI solves this by:

1. **Tracking quantity at the source level** — each warehouse/store is a `Source` with its own quantity
2. **Grouping sources into stocks** — a `Stock` is a logical collection of sources tied to a Sales Channel (website)
3. **Creating reservation records** — when an order is placed, a `Reservation` is created that *promises* inventory is held, without immediately decrementing source quantity
4. **Running a Source Selection Algorithm** — when you ship, Magento decides *which source* to pull from based on priority, distance, or custom logic

### 1.3 The Module Family

MSI is not a single module — it is a suite:

```
Magento_Inventory                      — Core interfaces and shared logic
Magento_InventoryApi                   — API contracts (all service contracts live here)
Magento_InventoryCatalog               — Bridges MSI to the catalog (product saves, etc.)
Magento_InventoryConfigurableProduct   — Configurable product support
Magento_InventoryBundleProduct         — Bundle product support
Magento_InventoryGroupedProduct        — Grouped product support
Magento_InventoryDownloadableProduct   — Downloadable product support
Magento_InventorySales                 — Sales order integration, reservation creation
Magento_InventoryShipping               — Shipping orchestration
Magento_InventoryReservations          — Reservation CRUD and calculation
Magento_InventorySourceSelection        — Source Selection Algorithm framework
Magento_InventorySourceSelectionApi     — SSA API layer
Magento_InventoryIndexer               — Inventory indexer
Magento_InventoryLowQuantityNotification — Low stock alerts
Magento_InventoryDistanceBasedSourceSelection — GeoIP-based SSA
```

---

## 2. Core Concepts: Source, Stock, Reservation

Understanding MSI starts with three entities that form a chain: **Source → Stock → Reservation**.

### 2.1 `InventorySource` — Physical Inventory Locations

An `InventorySource` represents a **physical location** that holds inventory. This could be a warehouse, a retail store, a distribution center, or a vendor's location.

**Database table:** `inventory_source`

**Key fields:**

```sql
inventory_source
├── source_code      VARCHAR(64)   -- Primary key, e.g. "warehouse_1"
├── name             VARCHAR(255)  -- Human-readable: "Chicago Warehouse"
├── enabled          TINYINT(1)    -- 1 = active, 0 = disabled
├── latitude         DECIMAL(10,8) -- For geo-based source selection
├── longitude        DECIMAL(11,8) -- For geo-based source selection
├── country          VARCHAR(3)     -- ISO country code
├── region           VARCHAR(64)
├── city             VARCHAR(255)
├── street           VARCHAR(255)
├── postcode         VARCHAR(21)
├── phone            VARCHAR(255)
├── fax              VARCHAR(255)
├── email            VARCHAR(255)
└── priority         INT(11)        -- Default priority (used when no SSA override)
```

**API Interface:** `Magento\InventoryApi\Api\Data\SourceInterface`

```php
interface SourceInterface extends \Magento\Framework\Api\ExtensibleDataInterface
{
    public function getSourceCode(): string;
    public function setSourceCode(string $sourceCode): SourceInterface;

    public function getName(): string;
    public function setName(string $name): SourceInterface;

    public function getEnabled(): bool;
    public function setEnabled(bool $enabled): SourceInterface;

    public function getLatitude(): ?float;
    public function getLongitude(): ?float;
    public function getCountry(): ?string;
    public function getPriority(): int;
}
```

**Creating a Source programmatically:**

```php
/** @var \Magento\InventoryApi\Api\SourceRepositoryInterface $sourceRepository */
/** @var \Magento\InventoryApi\Api\Data\SourceInterfaceFactory $sourceFactory */

/** @var \Magento\InventoryApi\Api\Data\SourceInterface $source */
$source = $sourceFactory->create();
$source->setSourceCode('warehouse_chicago');
$source->setName('Chicago Warehouse');
$source->setEnabled(true);
$source->setCountry('US');
$source->setRegion('IL');
$source->setCity('Chicago');
$source->setPostcode('60601');
$source->setPriority(1);

$sourceRepository->save($source);
```

**Important behavior:** A `Source` only *holds* inventory — it does not determine *who can see it*. That is the job of the `Stock`.

### 2.2 `InventoryStock` — Logical Grouping of Sources

A `Stock` is a **logical grouping of Sources** that maps to a Sales Channel (typically a Website). A product assigned to a Stock can be sold from any Source within that Stock.

**Database table:** `inventory_stock`

```sql
inventory_stock
├── stock_id      INT(11)      -- Primary key, e.g. 1 = Default Stock
├── name          VARCHAR(255) -- Human-readable name
├── extension_attributes  -- linkage to sales channel (website)
```

**API Interface:** `Magento\InventoryApi\Api\Data\StockInterface`

The `Default Stock` (stock_id = 1) is created during installation and cannot be deleted. It is linked to the Default Website.

**The Source-Link table:** `inventory_source_link`

```sql
inventory_source_link
├── link_id       INT(11)      -- Primary key
├── stock_id      INT(11)      -- FK to inventory_stock.stock_id
├── source_code   VARCHAR(64)  -- FK to inventory_source.source_code
└── priority      INT(11)      -- Priority of this source within this stock
```

The `priority` field here is **per-stock**, meaning a Source can have different priorities in different Stocks. This is key — a Source that is "primary" for one website might be "secondary" for another.

**Linking a Source to a Stock:**

```php
/** @var \Magento\InventoryApi\Api\StockSourceLinkRepositoryInterface $stockSourceLinkRepository */
/** @var \Magento\InventoryApi\Api\Data\StockSourceLinkInterfaceFactory $stockSourceLinkFactory */

$link = $stockSourceLinkFactory->create();
$link->setSourceCode('warehouse_chicago');
$link->setStockId(1); // Default Stock
$link->setPriority(1); // Highest priority within this stock

$stockSourceLinkRepository->save($link);
```

**Sales Channel linkage** is handled via the `inventory_stock_sales_channel` table (linked in the `stock_id` → website relationship):

```sql
inventory_stock_sales_channel
├── stock_id          INT(11)
├── type              VARCHAR(64)  -- "website" (only type currently)
└── code              VARCHAR(64)  -- website code
```

### 2.3 `InventoryStockItem` — Per-Source Quantity

The `cataloginventory_stock_item` table still exists but now acts as an **aggregate view**. The per-source quantity is tracked in `inventory_source_item`:

```sql
inventory_source_item
├── source_code   VARCHAR(64)  -- PK part 1: source
├── sku           VARCHAR(64)  -- PK part 2: product SKU
├── quantity      DECIMAL(12,4)
└── status        TINYINT(1)    -- 1 = in_stock, 0 = out_of_stock for this source
```

The `cataloginventory_stock_item` table's `qty` column represents the **summed quantity across all sources for a given stock**, and `is_in_stock` reflects whether *any* source has available inventory.

### 2.4 `InventoryReservation` — The Reservation/Correction Mechanism

This is the heart of MSI's overselling prevention. A `Reservation` is a **record stating that inventory is claimed but not yet shipped**.

**Database table:** `inventory_reservation`

```sql
inventory_reservation
├── reservation_id   INT(11)       -- Primary key, auto-increment
├── stock_id         INT(11)       -- Which stock this reservation is against
├── sku             VARCHAR(64)   -- Product SKU
├── quantity        DECIMAL(12,4) -- Negative = reserved (claim), Positive = compensation (release)
├── reservation_type VARCHAR(32)  -- "source" or "invalidated"
└── metadata         JSON         -- Extra info: order_item_id, order_id, etc.
```

**The critical insight about `quantity`:** It is **signed**. A reservation created during order placement has a **negative** quantity (inventory is "claimed"). A compensation (e.g., order cancellation, shipment rollback) has a **positive** quantity (inventory is "released").

```php
// Reservation during order placement (pseudo-code):
// quantity = -3 means "3 units are being held for this order"

// Compensation during order cancellation:
// quantity = +3 means "3 units are being returned to available pool"
```

**API Interface:** `Magento\InventoryReservationsApi\Api\Data\ReservationInterface`

```php
interface ReservationInterface extends \Magento\Framework\Api\ExtensibleDataInterface
{
    public function getReservationId(): int;
    public function getStockId(): int;
    public function getSku(): string;
    public function getQuantity(): float;
    public function getMetadata(): array;
}
```

### 2.5 The `reserved_quantity` Field

The `cataloginventory_stock_item` table has a `reserved_quantity` column. This is a **denormalized counter** that is updated by the indexer — it is the sum of all active reservation quantities for this product-stock combination.

```sql
cataloginventory_stock_item
├── item_id              INT(11)
├── product_id           INT(10)
├── stock_id             INT(10)
├── qty                 DECIMAL(12,4)  -- Quantity available
├── reserved_quantity   DECIMAL(12,4) -- Sum of active reservations
├── is_in_stock          TINYINT(1)
└── low_stock_date      DATETIME
```

**The "available quantity" calculation:**

```
available_quantity = quantity - reserved_quantity
```

When Magento checks if a product can be ordered, it uses this formula. If `available_quantity < requested_quantity`, the order line fails or goes on backorder depending on configuration.

The `reserved_quantity` is updated by the `inventory` indexer — it is **not updated synchronously** when a reservation is created. This is intentional: it would be a massive performance bottleneck to recalculate every reservation-affected item on every order. Instead, the indexer runs on schedule and keeps this column in sync with the actual reservation state.

**Race condition prevention:** Because the indexer is scheduled (not real-time), the `available_quantity` in `cataloginventory_stock_item` can briefly lag behind reality. The `async` operations mode (covered in Section 6) is designed to handle this lag gracefully.

---

## 3. Source Selection Algorithm (SSA)

When you ship an order (or a partial order), Magento needs to answer the question: **"Which Source(s) should fulfill which quantity of which product?"** The Source Selection Algorithm (SSA) answers this.

### 3.1 `SourceSelectionInterface`

```php
// Magento\InventorySourceSelectionApi\Api\SourceSelectionInterface

interface SourceSelectionInterface
{
    /**
     * @param \Magento\InventorySourceSelectionApi\Api\Data\SourceSelectionRequestInterface $request
     * @return \Magento\InventorySourceSelectionApi\Api\Data\SourceSelectionResultInterface
     */
    public function execute(
        \Magento\InventorySourceSelectionApi\Api\Data\SourceSelectionRequestInterface $request
    ): \Magento\InventorySourceSelectionApi\Api\Data\SourceSelectionResultInterface;
}
```

The input (`SourceSelectionRequestInterface`) contains:
- The items to fulfill (sku, quantity requested)
- The stock ID
- Optional parameters (shipment date, destination address for distance-based selection)

The output (`SourceSelectionResultInterface`) is a list of **SourceItemSelection** objects — each representing how much of a product should be taken from which source.

### 3.2 `SourceSelectionResultInterface`

```php
interface SourceSelectionResultInterface extends \Magento\Framework\Api\ExtensibleDataInterface
{
    public function getSourceSelectionItems(): array; // SourceItemSelection[]
    public function getTotalQty(): float;
}

interface SourceItemSelectionInterface
{
    public function getSourceCode(): string;
    public function getSku(): string;
    public function getQtyToDeduct(): float;
    public function getQtyAvailable(): float;
}
```

### 3.3 Priority-Based Selection (Default)

The default SSA implementation is `Magento\InventorySourceSelection\Model\Source\Priority\PrioritySourceSelection`.

This algorithm sorts Sources by their `priority` in `inventory_source_link` and fills from the highest priority source first.

**How it works step by step:**

1. Get all Sources linked to the Stock, sorted by `priority` ASC (lower number = higher priority)
2. For each requested item:
   - Try to fill `requested_qty` from Source 1
   - If Source 1 has insufficient quantity, fill from Source 1 + Source 2
   - Continue until requested quantity is fulfilled or all sources exhausted
3. Return the list of SourceItemSelections with quantities

```php
// Simplified pseudo-code of PrioritySourceSelection::execute()

$sourceItems = $this->sourceItemRepository->getAssignedSourcesByStockId($stockId);
$sourceItemsSorted = sortByPriority($sourceItems); // lowest priority number first

$requestedQty = $request->getItem()->getQty();

// Walk through sources in priority order, claiming quantity
foreach ($sourceItemsSorted as $sourceItem) {
    $available = $sourceItem->getQuantity();
    $qtyToTake = min($available, $requestedQty);

    if ($qtyToTake > 0) {
        $selections[] = new SourceItemSelection(
            $sourceItem->getSourceCode(),
            $sourceItem->getSku(),
            $qtyToTake,
            $available
        );
        $requestedQty -= $qtyToTake;
    }

    if ($requestedQty <= 0) break;
}
```

### 3.4 Distance-Based Selection

Magento ships a second SSA: `Magento\InventoryDistanceBasedSourceSelection\Model\DistanceBasedSourceSelection`.

This algorithm uses the **destination address** from the `SourceSelectionRequest` and calculates the distance to each source using either:
- **GeoIP** — looks up the customer's IP address location
- **Distance attribute** — uses the `latitude`/`longitude` of each source and a destination address

Sources are then sorted by **distance** (shortest first).

To enable distance-based selection, you need to configure it:

```bash
# In Stores > Configuration > Catalog > Inventory > Source Selection Algorithm
# Set "Distance Priority" to Enabled
```

Or via CLI:

```bash
bin/magento inventory:source:update:lat-lng --source-code="warehouse_chicago" --latitude=41.8781 --longitude=-87.6298
```

### 3.5 Custom SSA Implementation

You can register your own Source Selection Algorithm by implementing `SourceSelectionInterface` and declaring it in `di.xml`.

**Step 1: Create the class**

```php
// app/code/MyCompany/Inventory/Model/Source/Selection/CheapShippingSourceSelection.php

namespace MyCompany\Inventory\Model\Source\Selection;

use Magento\InventorySourceSelectionApi\Api\Data\SourceSelectionRequestInterface;
use Magento\InventorySourceSelectionApi\Api\Data\SourceSelectionResultInterface;
use Magento\InventorySourceSelectionApi\Api\SourceSelectionInterface;
use Magento\InventorySourceSelectionApi\Api\Data\SourceItemSelectionInterfaceFactory;
use Magento\InventorySourceSelectionApi\Api\Data\SourceSelectionResultInterfaceFactory;

class CheapShippingSourceSelection implements SourceSelectionInterface
{
    public function __construct(
        private readonly SourceItemSelectionInterfaceFactory $sourceItemSelectionFactory,
        private readonly SourceSelectionResultInterfaceFactory $sourceSelectionResultFactory
    ) {}

    public function execute(SourceSelectionRequestInterface $request): SourceSelectionResultInterface
    {
        $requestedItem = $request->getItem();
        $requestedQty = $requestedItem->getQty();
        $remainingQty = $requestedQty;

        $selections = [];
        $sourceItems = $this->getSourceItemsSortedByShippingCost($request->getStockId());

        foreach ($sourceItems as $sourceItem) {
            if ($remainingQty <= 0) {
                break;
            }

            $available = $sourceItem->getQuantity();
            $qtyToTake = min($available, $remainingQty);

            if ($qtyToTake > 0) {
                $selections[] = $this->sourceItemSelectionFactory->create([
                    'sourceCode' => $sourceItem->getSourceCode(),
                    'sku' => $sourceItem->getSku(),
                    'qtyToDeduct' => $qtyToTake,
                    'qtyAvailable' => $available,
                ]);

                $remainingQty -= $qtyToTake;
            }
        }

        return $this->sourceSelectionResultFactory->create([
            'sourceSelectionItems' => $selections,
            'totalQty' => $requestedQty - $remainingQty,
        ]);
    }
}
```

**Step 2: Declare it in `di.xml`**

```xml
<!-- app/code/MyCompany/Inventory/etc/di.xml -->

<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <type name="Magento\InventorySourceSelectionApi\Model\SourceSelectionPool">
        <arguments>
            <argument name="sourceSelections">
                <item name="cheap_shipping" xsi:type="string">MyCompany\Inventory\Model\Source\Selection\CheapShippingSourceSelection</item>
            </argument>
        </arguments>
    </type>

</config>
```

**Step 3: Configure it as active**

Set your custom SSA as the active algorithm in `Stores > Configuration > Catalog > Inventory > Source Selection Algorithm Options`.

---

## 4. Inventory Indexer

### 4.1 The `inventory` Indexer

Magento 2.4.x uses the **indexer** as the mechanism to keep `cataloginventory_stock_item` data in sync with MSI state.

```bash
bin/magento indexer:reindex inventory
```

**Indexer name:** `inventory`
**Indexer class:** `Magento\InventoryIndexer\Model\Indexer`

**What it does:** For every product and every stock, it calculates:
1. The total quantity across all sources assigned to the stock
2. The sum of all active reservations (updates `reserved_quantity`)
3. Whether any source for this product/stock is `is_in_stock = 1`
4. Writes the result to `cataloginventory_stock_item`

### 4.2 `cataloginventory_stock_item` — The Aggregated View

Think of `cataloginventory_stock_item` as a **read cache** of MSI state. It is denormalized for performance — the product page and checkout need to quickly answer "is this in stock and how many can I buy?" without doing complex joins across `inventory_source_item` + `inventory_reservation` + `inventory_source_link` every time.

```sql
-- The indexer populates this table:
cataloginventory_stock_item
├── product_id         INT(10)         -- FK to catalog_product_entity.entity_id
├── stock_id           INT(10)         -- Usually 1 (Default Stock)
├── qty                DECIMAL(12,4)  -- Total available: SUM(source_item.quantity)
├── reserved_quantity  DECIMAL(12,4)   -- SUM(reservation.quantity WHERE quantity < 0)
├── is_in_stock        TINYINT(1)     -- 1 if ANY source is in stock
└── low_stock_date     DATETIME       -- For low stock monitoring
```

### 4.3 Scheduled vs. On-Save Indexing

Like all Magento indexers, `inventory` supports two modes:

**On-save (`cataloginventory_stock_item`):**
```bash
bin/magento indexer:set-mode inventory onSave
```
Inventory is recalculated immediately when source/stock data changes. This provides real-time accuracy but can be slow under heavy write load.

**Scheduled (`realtime` or `schedule`):**
```bash
bin/magento indexer:set-mode inventory schedule
```
The indexer runs on a cron schedule (every minute by default). Changes are batched and applied in a single pass. This is the recommended mode for production.

**Why not real-time everywhere?** Because updating `reserved_quantity` involves reading all reservation records for a product. In high-volume stores, doing this synchronously on every order would create lock contention and massive slowdown. The async/scheduled approach trades brief staleness for throughput.

### 4.4 Indexer Data Flow

```
Order placed
    │
    ▼
InventorySalesApi::registerProductSale()    ← Creates reservation record (negative quantity)
    │
    ▼
inventory.indexer runs (on schedule or on-save)
    │
    ▼
For each product+stock:
  1. SUM(inventory_source_item.quantity)     → qty
  2. SUM(inventory_reservation.quantity)      → reserved_quantity
  3. Check if ANY source_item.status = 1     → is_in_stock
    │
    ▼
Writes to cataloginventory_stock_item
    │
    ▼
Product page / checkout reads qty and reserved_quantity
    │
    ▼
available_qty = qty - reserved_quantity
```

---

## 5. Inventory API

All MSI service contracts live in `Magento_InventoryApi`. Always use the API interfaces — do not directly manipulate the database tables.

### 5.1 `InventoryCatalogApi::getStockItemsDataBySkus`

Retrieves stock item data (quantity, is_in_stock, etc.) for a list of SKUs.

```php
// Magento\InventoryCatalogApi\Api\InventoryCatalogInterface

interface InventoryCatalogInterface
{
    /**
     * @param string $sku
     * @param int $stockId
     * @return \Magento\InventoryCatalogApi\Api\Data\StockItemInterface
     */
    public function getStockItem(string $sku, int $stockId): \Magento\InventoryCatalogApi\Api\Data\StockItemInterface;

    /**
     * @param array $skus
     * @param int $stockId
     * @return array [sku => ['qty' => float, 'is_in_stock' => bool], ...]
     */
    public function getStockItemsDataBySkus(array $skus, int $stockId): array;
}
```

**Usage example:**

```php
/** @var \Magento\InventoryCatalogApi\Api\InventoryCatalogInterface $inventoryCatalog */

$skus = ['SKU-123', 'SKU-456'];
$stockId = 1; // Default Stock

$stockItems = $inventoryCatalog->getStockItemsDataBySkus($skus, $stockId);

foreach ($stockItems as $sku => $data) {
    printf(
        "SKU: %s | Qty: %s | In Stock: %s | Available: %s\n",
        $sku,
        $data['qty'],
        $data['is_in_stock'] ? 'Yes' : 'No',
        $data['qty'] - ($data['reserved_quantity'] ?? 0)
    );
}
```

### 5.2 `InventoryCatalogApi::updateSourceItem`

Updates the quantity for a specific Source + SKU combination.

```php
// Magento\InventoryCatalogApi\Api\UpdateSourceItemInterface

interface UpdateSourceItemInterface
{
    /**
     * @param \Magento\InventoryCatalogApi\Api\Data\SourceItemInterface $sourceItem
     * @return void
     */
    public function execute(\Magento\InventoryCatalogApi\Api\Data\SourceItemInterface $sourceItem): void;
}
```

**Usage example:**

```php
/** @var \Magento\InventoryCatalogApi\Api\Data\SourceItemInterfaceFactory $sourceItemFactory */
/** @var \Magento\InventoryCatalogApi\Api\UpdateSourceItemInterface $updateSourceItem */

$sourceItem = $sourceItemFactory->create();
$sourceItem->setSourceCode('warehouse_chicago');
$sourceItem->setSku('SKU-123');
$sourceItem->setQuantity(50);
$sourceItem->setStatus(1); // In stock

$updateSourceItem->execute($sourceItem);
```

### 5.3 `InventorySalesApi::registerProductSale` — Reservation Creation

This is the API call that fires when an order is placed. It **creates a reservation** (negative quantity) to claim inventory.

```php
// Magento\InventorySalesApi\Api\InventorySalesInterface

interface InventorySalesInterface
{
    /**
     * @param string $sku
     * @param float $quantity
     * @param int $stockId
     * @return void  -- creates reservation record
     */
    public function registerProductSale(string $sku, float $quantity, int $stockId): void;

    /**
     * @param string $sku
     * @param float $quantity
     * @param int $stockId
     * @return void  -- creates compensation reservation (positive quantity)
     */
    public function revertProductSale(string $sku, float $quantity, int $stockId): void;
}
```

**Usage example (manual reservation creation):**

```php
/** @var \Magento\InventorySalesApi\Api\InventorySalesInterface $inventorySales */

$inventorySales->registerProductSale(
    'SKU-123',
    3.0,       // 3 units ordered
    1          // stock ID (Default Stock)
);
// Internally: creates inventory_reservation row with quantity = -3
// This prevents overselling by making those 3 units unavailable for other orders
```

### 5.4 `InventoryReservationBuilder` — Building Reservations

The `ReservationBuilder` is a fluent builder for creating complex reservation records (used internally by `InventorySales` and for custom scenarios).

```php
// Magento\InventoryReservationsApi\Model\ReservationBuilder

class ReservationBuilder
{
    public function setSku(string $sku): self;
    public function setStockId(int $stockId): self;
    public function setQuantity(float $quantity): self;  // negative for claim, positive for release
    public function setMetadata(array $metadata): self;
    public function build(): ReservationInterface;
    public function create(): ReservationInterface;  // builds AND saves in one call
}
```

**Usage example:**

```php
/** @var \Magento\InventoryReservationsApi\Model\ReservationBuilder $reservationBuilder */
/** @var \Magento\InventoryReservationsApi\Api\ReservationRepositoryInterface $reservationRepository */

$reservation = $reservationBuilder
    ->setSku('SKU-123')
    ->setStockId(1)
    ->setQuantity(-5.0)  // Claim 5 units
    ->setMetadata([
        'order_id' => '100000123',
        'reason' => 'manual_hold'
    ])
    ->build();

$reservationRepository->add($reservation);
```

**Key behavior:** The `ReservationBuilder` does **not** immediately update `cataloginventory_stock_item.reserved_quantity`. That update happens when the `inventory` indexer runs. This is intentional — you should not assume `reserved_quantity` is accurate immediately after calling the reservation API.

### 5.5 Full Example: Registering a Product Sale

Here is the complete flow for manually registering a product sale and observing the reservation:

```php
<?php
// Example: Manual sale registration flow
// (In reality, Magento handles this via Magento\Sales\Model\Order::place()
//  which calls InventorySales::registerProductSale automatically)

use Magento\Framework\App\ObjectManager;
use Magento\InventorySales\Model\ResourceModel\GetReservationsquantity;

$objectManager = ObjectManager::getInstance();

/** @var \Magento\InventorySalesApi\Api\InventorySalesInterface $inventorySales */
/** @var \Magento\InventorySalesApi\Api\GetStockItemsDataBySkusInterface $getStockItems */
$inventorySales = $objectManager->get(\Magento\InventorySalesApi\Api\InventorySalesInterface::class);
$getStockItems = $objectManager->get(\Magento\InventorySalesApi\Api\GetStockItemsDataBySkusInterface::class);

// Before sale: check available quantity
$before = $getStockItems->execute(['SKU-123'], 1);
echo "Before: qty={$before[0]['qty']}, reserved={$before[0]['reserved_quantity']}\n";
// Output: Before: qty=50, reserved=0

// Register the sale: claim 3 units
$inventorySales->registerProductSale('SKU-123', 3.0, 1);

// Note: cataloginventory_stock_item.reserved_quantity is NOT updated yet
// (it updates when the inventory indexer runs)

// Verify reservation was created
/** @var \Magento\InventoryReservationsApi\Api\GetReservationsBySKUsAndStockInterface $getReservations */
$getReservations = $objectManager->get(
    \Magento\InventoryReservationsApi\Api\GetReservationsBySKUsAndStockInterface::class
);
$reservations = $getReservations->execute(['SKU-123'], 1);
foreach ($reservations as $reservation) {
    echo sprintf(
        "Reservation: sku=%s, qty=%s, stock_id=%s\n",
        $reservation->getSku(),
        $reservation->getQuantity(),  // should be -3.0
        $reservation->getStockId()
    );
}
// Output: Reservation: sku=SKU-123, qty=-3.0, stock_id=1
```

---

## 6. Reservations and Race Condition Prevention

### 6.1 The Available Quantity Calculation

This is the formula used throughout Magento to determine if a product can be ordered:

```
available_quantity = source_quantity_sum − |sum_of_negative_reservations|
```

For a given SKU + Stock:
1. **Source quantity sum** = SUM of `quantity` from `inventory_source_item` for all sources in the stock
2. **Sum of negative reservations** = SUM of `quantity` (which is negative) from `inventory_reservation` where the stock matches and the SKU matches

In the `cataloginventory_stock_item` table, this becomes:
```
available_quantity = qty - reserved_quantity
```

Where `qty` is the sum of all source quantities and `reserved_quantity` is the sum of all negative reservation quantities (stored as a positive number for display).

### 6.2 Async Operations for Inventory Updates

Magento 2.4.3 introduced **async inventory operations** to handle a specific race condition: under high load, simultaneous orders for the same SKU could both read the same available quantity before either had created a reservation, resulting in overselling.

**The race condition:**
```
T1: Order A reads available = 5  (before reservation)
T2: Order B reads available = 5  (before reservation)
T3: Order A creates reservation for 5 → success
T4: Order B creates reservation for 5 → success  ← oversold by 5!
```

**The async solution:** When async mode is enabled, the `registerProductSale` call does not immediately create the reservation. Instead, it places a message in the `inventory` message queue. A consumer processes the message and creates the reservation. This **serializes** the inventory claims — the consumer processes them one at a time.

To enable async mode:

```bash
bin/magento config:set cataloginventory/options/use_async_notify 1
```

Or in `Stores > Configuration > Catalog > Inventory > Asynchronous Operations`:
- Set "Enable Async Inventory Operations" to `Yes`

**When does this matter?** On high-volume stores (100+ orders/minute), the async queue acts as a buffer. The synchronous path is acceptable for lower-volume stores where the risk of concurrent ordering is low.

### 6.3 Cleanup of Expired Reservations

Reservations are **not automatically removed** when an order is shipped. They are "compensated" (a positive reservation is created) when:
- The order is **cancelled** (full or partial)
- The shipment is **rolled back**
- A **credit memo** is issued (full or partial refund)

When none of these happen and the order reaches a configured **expiration period**, the `inventory.reservation.cleanup` cron job removes stale reservations.

```bash
# Runs every minute via cron (default configuration)
bin/magento cron:run --group=inventory
```

**What qualifies as "expired"?** A reservation is considered orphaned/expired when it has been open for longer than the configured **cleanup period** (default: 0, meaning never, unless explicitly configured). Actually, in Magento 2.4.x, reservations are primarily cleaned up through compensations, not through expiration timeouts.

**The `inventory:reservation:list-inconsistencies` command** (covered in CLI Commands) detects reservations that have no corresponding order and flags them for cleanup.

### 6.4 Reservation Compensation Flow

```
Order placed
    │
    ▼
registerProductSale()  ──► Reservation created (quantity = -N)
    │
    ▼
Order shipped (partial or full)
    │
    ▼
inventory:source: deduction  ──► Compensation Reservation created (quantity = +N)
    │
    ▼
Order cancelled (if applicable)
    │
    ▼
inventory:reservation:create-compensations  ──► Compensation Reservation (quantity = +N)
```

---

## 7. Customizing MSI

### 7.1 Creating a Custom Source

The most common MSI customization is adding a physical inventory location.

**Step 1: Create via Admin Panel**

Navigate to **Stores > Inventory > Sources** and click "Add New Source".

**Step 2: Create programmatically:**

```php
/** @var \Magento\InventoryApi\Api\SourceRepositoryInterface $sourceRepository */
/** @var \Magento\InventoryApi\Api\Data\SourceInterfaceFactory $sourceFactory */

// $source = $sourceFactory->create() ...
// $sourceRepository->save($source) ...
```

**Step 3: Assign Sources to the Stock**

```php
/** @var \Magento\InventoryApi\Api\StockSourceLinkRepositoryInterface $linkRepository */
/** @var \Magento\InventoryApi\Api\Data\StockSourceLinkInterfaceFactory $linkFactory */

$link = $linkFactory->create();
$link->setSourceCode('dropship_vendor_a');
$link->setStockId(1);
$link->setPriority(2); // Secondary to your primary warehouse

$linkRepository->save($link);
```

**Step 4: Set Source Quantities**

```php
/** @var \Magento\InventoryApi\Api\SourceItemRepositoryInterface $sourceItemRepository */
/** @var \Magento\InventoryApi\Api\Data\SourceItemInterfaceFactory $sourceItemFactory */

// $sourceItem = $sourceItemFactory->create() ...
// $sourceItem->setSourceCode('dropship_vendor_a') ...
// $sourceItem->setSku('SKU-123') ...
// $sourceItem->setQuantity(100) ...
// $sourceItem->setStatus(1) ...

$sourceItemRepository->save($sourceItem);
```

### 7.2 Assigning Sources to Stocks

One Source can belong to **multiple Stocks** with different priorities. For example:

```
Stock: US West Website
  └── warehouse_west      (priority: 1)
  └── warehouse_central  (priority: 2)

Stock: US East Website
  └── warehouse_east     (priority: 1)
  └── warehouse_central  (priority: 2)
```

This means `warehouse_central` fulfills for both websites but at lower priority than the regional warehouse.

### 7.3 Disabling MSI (Using Legacy Stock)

In rare cases (typically during migration), you may want to disable MSI and use the legacy single-stock behavior. This is done by:

**1. Setting all products to use only the Default Stock:**

Products in Magento 2.4.x are implicitly assigned to all stocks by default (unless explicitly assigned to a specific source). When a product is only assigned to `Default Stock`, MSI behaves effectively like the legacy system.

**2. Disabling the async queue:**

```bash
bin/magento config:set cataloginventory/options/use_async_notify 0
```

**3. Switching to on-save indexer mode:**

```bash
bin/magento indexer:set-mode inventory onSave
```

**Important:** Truly "disabling MSI" is not straightforward because the Inventory module family is loaded very early in Magento's bootstrap. The supported approach is to **minimize MSI complexity** rather than disable it entirely — use only the Default Stock, a single source, and the default priority SSA.

---

## 8. CLI Commands

### 8.1 `inventory:reservation:create-compensations`

Fixes **orphaned reservations** — reservations that exist but have no corresponding order, or where the order was cancelled but compensation was never created.

```bash
# Create compensation reservations for orphaned entries
bin/magento inventory:reservation:create-compensations [--dry-run] [--sku="SKU-123"] [--stock-id=1]
```

**When to use it:**
- After a failed order data migration where orders were imported but reservations were not
- After a bug caused order cancellations to fail to create compensation reservations
- After manually manipulating the `sales_order` or `inventory_reservation` tables

**How it works:** It scans for reservations with `metadata` referencing an `order_id`, checks if that order exists and if its status warrants a compensation, and if not, creates the appropriate positive reservation to release the inventory.

### 8.2 `inventory:reservation:list-inconsistencies`

Lists **quantity inconsistencies** — situations where the quantity in the reservation table does not match what it should be based on orders and shipments.

```bash
# List all reservation inconsistencies
bin/magento inventory:reservation:list-inconsistencies

# Output example:
# SKU-123 | Expected: 0.0000 | Reserved: -5.0000 | Inconsistency
```

**What it detects:**
- Reservations for orders that no longer exist
- Reservations that were never compensated after shipment
- Mismatches between `inventory_reservation` totals and `cataloginventory_stock_item.qty`

**The resolution** is typically to run `inventory:reservation:create-compensations` to fix the inconsistencies.

### 8.3 `inventory:source:list`

Lists all configured inventory sources.

```bash
bin/magento inventory:source:list

# Output:
# Source Code    | Name                  | Enabled | Priority
# --------------|----------------------|---------|----------
# default       | Default Source        |   1     |    0
# warehouse_1   | Main Warehouse        |   1     |    1
# warehouse_2   | Secondary Warehouse   |   1     |    2
# store_boston  | Boston Retail Store   |   1     |    3
```

### 8.4 `inventory:stock:list`

Lists all configured stocks.

```bash
bin/magento inventory:stock:list

# Output:
# Stock ID | Name            | Website IDs
# ---------|-----------------|------------
#    1     | Default Stock   |     1
#    2     | US West Stock   |     2
#    3     | EU Stock        |     3
```

### 8.5 Other Useful Commands

```bash
# Update source item quantity (quick update without Admin)
bin/magento inventory:source:item:update --sku="SKU-123" --source="warehouse_1" --quantity=50

# Update source latitude/longitude (for distance-based SSA)
bin/magento inventory:source:update:lat-lng --source-code="warehouse_1" --latitude=41.8781 --longitude=-87.6298

# Rebuild the inventory index
bin/magento indexer:reindex inventory

# Show indexer status
bin/magento indexer:status
```

---

## Chapter Summary

MSI is Magento's comprehensive answer to multi-warehouse inventory management. The key takeaways:

| Concept | What It Is | Key Table/Interface |
|---------|-----------|---------------------|
| **Source** | Physical location (warehouse, store) | `inventory_source` / `SourceInterface` |
| **Stock** | Logical group of sources per sales channel | `inventory_stock` / `StockInterface` |
| **Reservation** | Claim on inventory (negative = held, positive = released) | `inventory_reservation` / `ReservationInterface` |
| **Source Selection** | Algorithm to pick which source to fulfill from | `SourceSelectionInterface` |
| **Indexer** | Syncs MSI state to `cataloginventory_stock_item` | `inventory` indexer |

**The reservation formula:** `available_quantity = source_quantity_sum − |sum(negative_reservations)|`

**The race condition solution:** Async operations queue serialize concurrent inventory claims.

**The cleanup mechanism:** `inventory.reservation.cleanup` cron + `inventory:reservation:create-compensations` CLI.

**Next:** The next chapter covers Inventory reservation internals and the asynchronous operations mode in greater depth, including the message queue architecture and the `inventory_indexer` consumer group behavior.
