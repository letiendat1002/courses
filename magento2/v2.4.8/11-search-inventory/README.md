# Topic 11: Inventory — Multi-Source Inventory (MSI)

**Goal:** Master Magento's Multi-Source Inventory system — configure sources and stocks, understand the reservation model, implement source selection algorithms, and manage inventory programmatically.

---

## Table of Contents

1. [MSI Architecture Overview](#1-msi-architecture-overview)
2. [Sources & Stocks Configuration](#2-sources--stocks-configuration)
3. [Reservations Model](#3-reservations-model)
4. [Source Selection Algorithm](#4-source-selection-algorithm)
5. [Inventory Indexers](#5-inventory-indexers)
6. [Programmatic Inventory Management](#6-programmatic-inventory-management)
7. [MSI for Headless/API-First](#7-msi-for-headlessapi-first)

---

## Topics Covered

- MSI architecture and the problem it solves
- Source and stock configuration via admin and API
- Reservation model for deducting/reserving inventory
- Source selection algorithm (SSA) and priority-based allocation
- Inventory indexers: `inventory`, `inventory_reservations`
- Managing inventory across multiple warehouses programmatically
- MSI limitations and inventory aggregation edge cases

---

## Reference Exercises

- **Exercise 11.1:** Create a custom source via `inventory_api` and assign it to a stock
- **Exercise 11.2:** Write a script that gets available quantity across all sources for a SKU
- **Exercise 11.3:** Implement a custom source selection algorithm for prioritising local warehouse
- **Exercise 11.4:** Create reservations programmatically and verify deduct/compensate flow
- **Exercise 11.5:** Build an admin grid extension to display source-level inventory in product edit

---

## Completion Criteria

- [ ] Custom source created and assigned to a stock via API
- [ ] Available quantity query returns aggregated stock across all sources
- [ ] Source selection algorithm selects correct source for an order
- [ ] Reservations created on order placement and removed on shipment/invoice
- [ ] Inventory indexer marked as valid after source/stock changes
- [ ] Custom SSA plugin registered via `di.xml` and operational
- [ ] Product inventory correctly managed across multiple sources

---

## Topics

---

### Topic 1: MSI Architecture Overview

Before MSI (introduced in Magento 2.3), inventory was a single `qty` field on `cataloginventory_stock_item`. MSI introduces a full inventory management system with multiple sources.

**Why MSI Matters:**

```
Traditional (pre-MSI):
┌─────────────────────┐
│  cataloginventory   │
│  _stock_item.qty   │  ← Single number, single source
└─────────────────────┘

MSI (Magento 2.3+):
┌─────────────────────┐     ┌─────────────────────┐
│     Sources         │     │      Stocks          │
│  (Warehouses/SDKs)  │────▶│  (Sales Channels)   │
└─────────────────────┘     └─────────────────────┘
         │                             │
         ▼                             ▼
┌─────────────────────┐     ┌─────────────────────┐
│ inventory_source    │     │ inventory_stock     │
│ inventory_source    │     │ inventory_stock_sour│
│ _item               │     │ _link               │
└─────────────────────┘     └─────────────────────┘
```

**MSI Tables:**

| Table | Purpose |
|-------|---------|
| `inventory_source` | Source definitions (name, address, enabled) |
| `inventory_source_stock_link` | Many-to-many source-stock relationships |
| `inventory_stock` | Stock definitions (name, website linkage) |
| `inventory_reservation` | Active reservations (orders not fulfilled) |
| `inventory_source_item` | Per-SKU quantity at each source |

---

### Topic 2: Sources & Stocks Configuration

**Source** = A physical location (warehouse, store, distribution center)

**Stock** = A virtual grouping of sources tied to a sales channel (website)

**Creating a Source programmatically:**

```php
// Magento\InventoryApi\Api\SourceRepositoryInterface
$source = $this->sourceFactory->create();
$source->setData([
    'source_code' => 'warehouse_east',
    'name' => 'East Coast Warehouse',
    'enabled' => 1,
    'description' => 'Primary East Coast fulfillment center',
    'latitude' => 40.7128,
    'longitude' => -74.0060,
    'country_id' => 'US',
    'region_id' => 43,
    'city' => 'New York',
    'street' => '123 Warehouse Ave',
    'postcode' => '10001',
    'contact_name' => 'John Doe',
    'phone' => '+1-555-0100',
    'email' => 'warehouse@example.com',
]);
$this->sourceRepository->save($source);
```

**Creating a Stock programmatically:**

```php
// Magento\InventoryApi\Api\StockRepositoryInterface
$stock = $this->stockFactory->create();
$stock->setData([
    'name' => 'East Coast Stock',
    'website_ids' => [1], // Link to website
]);
$stockId = $this->stockRepository->save($stock)->getId();
```

**Link Source to Stock:**

```php
// Magento\InventoryApi\Api\StockSourceLinksRepositoryInterface
$this->stockSourceLinksRepository->save([
    'stock_id' => $stockId,
    'source_code' => 'warehouse_east',
    'priority' => 1,
]);
```

---

### Topic 3: Reservations Model

Reservations are the core of MSI's concurrency-safe inventory management. When an order is placed, Magento creates a reservation that deducts from available quantity without immediately modifying `inventory_source_item`.

**Why Reservations over Direct Deduction:**

- **Concurrent orders** — Two orders for the same SKU don't both succeed if stock is insufficient
- **Eventually consistent** — Reservation → shipment creates a compensation entry
- **No deadlocks** — Row-level locking on `inventory_reservation` instead of table locks

**Reservation Structure:**

| Field | Purpose |
|-------|---------|
| `reservation_id` | Primary key |
| `stock_id` | Which stock this reservation belongs to |
| `sku` | Product SKU |
| `quantity` | Positive = reservation, negative = compensation |
| `metadata` | JSON with `order_id`, `order_item_id` |
| `created_at` | Timestamp |

**Reading Reserved Quantity:**

```php
// Magento\InventoryReservationsApi\Api\GetReservationsQuantityInterface
public function execute(string $sku, int $stockId): float
{
    // Returns sum of all reservation quantities for SKU at stock
}
```

**Creating a Manual Reservation:**

```php
// Magento\InventoryReservationsApi\Api\ReservationInterface
$reservation = $this->reservationBuilder
    ->setStockId($stockId)
    ->setSku($sku)
    ->setQuantity($quantity) // Positive = reserve
    ->setMetadata([
        'order_id' => $orderId,
        'object_type' => 'order',
    ])
    ->build();
$this->appendReservations->execute([$reservation]);
```

**Compensation (reverting a reservation):**

```php
// When an order is cancelled or item removed, a negative reservation compensates:
$reservation = $this->reservationBuilder
    ->setStockId($stockId)
    ->setSku($sku)
    ->setQuantity(-$quantity) // Negative = compensation
    ->setMetadata([...])
    ->build();
```

---

### Topic 4: Source Selection Algorithm

The Source Selection Algorithm (SSA) determines which source(s) to fulfil an order from when multiple sources have stock.

**Default SSA Priority Algorithm:**

1. Collect all sources linked to the order's stock
2. Filter sources with sufficient available quantity
3. Sort by **priority** (lower = higher priority from `inventory_source_stock_link`)
4. Select source(s) — can split across multiple if one doesn't have full quantity

**Source Selection Interface:**

```php
// Magento\InventorySourceSelectionApi\Api\SourceSelectionServiceInterface
public function execute(
    SourceSelectionRequestInterface $request
): SourceSelectionResultInterface;
```

**SourceSelectionRequest:**

```php
$request = $this->sourceSelectionRequestBuilder
    ->setSku('WTSHIRT-BL-M')
    ->setQty(3)
    ->setStockId($stockId)
    ->setItemCriteria($criteria)
    ->build();
```

**SourceSelectionResult:**

```php
$result = $this->sourceSelectionService->execute($request);
foreach ($result->getSourceRates() as $rate) {
    $sourceCode = $rate->getSourceCode();
    $qtyToDeduct = $rate->getQty();
    // $rate->getPriority() — from inventory_source_stock_link
}
```

**Custom Source Selection Algorithm:**

Implement `SourceSelectionAlgorithmInterface` and register in `di.xml`:

```php
class CustomSSA implements SourceSelectionAlgorithmInterface
{
    public function execute(
        SourceSelectionRequestInterface $request
    ): SourceSelectionResultInterface {
        // Custom logic: nearest warehouse, least cost, etc.
    }
}
```

```xml
<!-- etc/di.xml -->
<type name="Magento\InventorySourceSelectionApi\Api\SourceSelectionServiceInterface">
    <arguments>
        <argument name="sourceSelectionAlgorithm" xsi:type="object">
            CustomSSA
        </argument>
    </arguments>
</type>
```

---

### Topic 5: Inventory Indexers

MSI has two critical indexers that must stay in sync:

**`inventory` indexer:**

- Indexes `inventory_source_item` quantities
- Triggered on source item save, stock source link change
- `bin/magento indexer:reindex inventory`

**`inventory_reservations` indexer:**

- Indexes reservations for fast available-quantity calculation
- Triggered on reservation create/compensate
- Maintains `quantity_in_stock` for each SKU/stock combination

**Indexer Configuration:**

```xml
<!-- Magento_Inventory/etc/indexer.xml -->
<indexer id="inventory" view="inventory" class="Magento\InventoryIndexer\Indexer">
    <schedule backend_entity="Magento\InventoryIndexer\IndexerPlugin"/>
</indexer>
```

**Make sure to run after source/stock changes:**

```bash
bin/magento indexer:reindex inventory
bin/magento indexer:reindex inventory_reservations
bin/magento c:c fullpage  # Also clean FPC as it caches cataloginventory
```

---

### Topic 6: Programmatic Inventory Management

**Get Available Quantity (all sources):**

```php
// Magento\InventorySalesApi\Api\IsProductSalableInterface
public function execute(string $sku, int $stockId): bool;
// Returns true if any source has available stock
```

```php
// Magento\InventorySalesApi\Api\GetProductSalableQuantityInterface
public function execute(string $sku, int $stockId): float;
// Returns sum of (source_item.qty - reservations) across all linked sources
```

**Get Quantity Per Source:**

```php
// Magento\InventoryApi\Api\GetSourceItemsBySkuInterface
public function execute(string $sku): array;
// Returns array of SourceItemInterface per source
```

```php
$sourceItems = $this->getSourceItemsBySku->execute('WTSHIRT-BL-M');
foreach ($sourceItems as $item) {
    echo $item->getSourceCode() . ': ' . $item->getQuantity();
}
```

**Update Source Item Quantity:**

```php
// Magento\InventoryApi\Api\SourceItemRepositoryInterface
$sourceItem = $this->sourceItemFactory->create();
$sourceItem->setSourceCode('warehouse_east');
$sourceItem->setSku('WTSHIRT-BL-M');
$sourceItem->setQuantity(100);
$sourceItem->setStatus(1); // 1 = enabled, 0 = disabled
$this->sourceItemRepository->save($sourceItem);
```

**Decrement Source Item Quantity (after shipment):**

```php
// Magento\InventorySales\Model\DecrementSourceQuantity.php
public function execute(SourceItemInterface $sourceItem, float $qty): void;
```

---

### Topic 7: MSI for Headless/API-First

In a headless architecture, order creation happens via API. MSI integrates with the Order Management API.

**Order Creation triggers reservation automatically via `SalesOrderInventoryPlaceAfter` observer:**

```xml
<!-- etc/events.xml -->
<event name="sales_order_place_after">
    <observer name="inventory_reservation"
              instance="Magento\InventorySales\Observer\SalesOrderPlaceAfterObserver"
              shared="false"/>
</event>
```

**For custom order creation (e.g., via custom REST endpoint), manually create reservation:**

```php
use Magento\InventorySales\Observer\SalesOrderPlaceAfterObserver;

class CustomOrderCreateObserver implements ObserverInterface
{
    public function __construct(
        private readonly \Magento\InventorySalesApi\Api\PlaceReservationsInterface $placeReservations,
        private readonly \Magento\InventorySales\Model\GetItemsOrderedFromOrder $getItemsOrderedFromOrder,
    ) {}

    public function execute(Observer $observer): void
    {
        $order = $observer->getEvent()->getOrder();

        $items = $this->getItemsOrderedFromOrder->execute($order);
        $placeReservations->execute($items, $order->getId(), $order->getStoreId());
    }
}
```

**Check if product is salable in GraphQL:**

```graphql
type Product {
  sku: String!
  productreference: String
  stock_status: ProductStockStatus
}

type ProductStockStatus {
  is_in_stock: Boolean!
  online_max_qty: Float
  reserved_quantity: Float
}
```

---

## Common Pitfalls

| Pitfall | Cause | Solution |
|---------|-------|----------|
| Overselling | SSA not called before order placement | Always use SSA for multi-source inventory |
| Phantom stock | Race condition in concurrent checkout | Reservations prevent this |
| Indexer lag | `inventory_reservations` out of sync | `bin/magento indexer:reindex inventory_reservations` |
| Source not linked | Source exists but not in stock | Link via `inventory_stock_source_link` |
| Zero qty returned | StockId mismatch | Verify `$stockId` matches order's stock |

---

## Cross-References

- **Topic 10 (Checkout):** Checkout flow triggers inventory reservations
- **Topic 15 (Sales & Orders):** Order placement triggers MSA reservation
- **Topic 18 (GraphQL Headless):** GraphQL returns salability status
- **Topic 11 Search (Supplemental):** Elasticsearch integration, search query types, Elasticsearch 7.x configuration — `_supplemental/17-elasticsearch-search.md`

---

*Magento 2 Backend Developer Course — Topic 11*
