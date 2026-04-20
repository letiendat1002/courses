# Topic 4: Data Layer — Models, Database, Service Contracts

**Goal:** Build a complete data layer for a custom entity — from database table creation to a clean, testable repository API.

---

## Topics Covered

- Declarative schema (`db_schema.xml`) — creating tables without raw SQL
- Schema whitelist generation
- Data patches and schema patches for data and structure changes
- Model / ResourceModel / Collection triad
- EAV architecture and custom product attributes
- EAV beyond products — customer attributes, category attributes
- Transaction patterns and SaveHandler for complex entities
- Service Contracts — interface-first design
- Repository pattern with proper error handling
- SearchCriteria for filtering and pagination

---

## Reference Exercises

- **Exercise 3.1:** Create `training_review` table via `db_schema.xml`
- **Exercise 3.2:** Write a Data Patch and a Schema Patch
- **Exercise 3.3:** Build the Model / ResourceModel / Collection layer
- **Exercise 3.4:** Define Service Contract interfaces (`Api/Data/` and `Api/`)
- **Exercise 3.5:** Implement repository and use it in a controller
- **Exercise 3.6:** Add a custom customer attribute (loyalty tier) via setup patch
- **Exercise 3.7:** Add a custom category attribute (featured flag) via setup patch
- **Exercise 3.8:** Implement transaction-wrapped save in repository
- **Exercise 3.9 (optional):** Build a SaveHandler for complex entity logic

---

## Completion Criteria

- [ ] Custom table created via `db_schema.xml` without errors (verified in MySQL)
- [ ] Schema whitelist generated (`etc/db_schema_whitelist.json`) and committed
- [ ] Data Patch inserts records and Schema Patch modifies a column (verified via db:status)
- [ ] Model/ResourceModel/Collection implemented and wired via `_construct()` and `di.xml`
- [ ] Service Contract interfaces defined in `Api/Data/` and `Api/` with proper return types
- [ ] Repository implementation wired via `di.xml` preference
- [ ] Controller uses repository (not direct model access) — verified by code review
- [ ] Customer EAV attribute created and readable via customer object
- [ ] Transaction pattern used in repository for multi-entity operations
- [ ] PHPCS reports zero errors on all PHP files

---

## Topics

---

### Topic 1: Declarative Schema (db_schema.xml)

**Why Declarative Schema?**

| Approach | Problem |
|---------|---------|
| Raw SQL in install scripts | No rollback, version conflicts |
| Setup scripts (old way) | Complex, hard to maintain |
| `db_schema.xml` | Single source of truth, auto-generates whitelist |

> **Why declarative over install scripts?** Install/upgrade scripts (the old approach) run only once and are hard to trace. If something goes wrong, rollback is manual. Declarative schema defines the *desired state* — Magento figures out how to get there. It also generates a whitelist that tracks what's been applied.

**db_schema.xml Structure:**

```xml
<!-- app/code/Training/Review/etc/db_schema.xml -->
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup:Declaration:etc/db_schema.xsd">
    <table name="training_review"
           resource="default"
           comment="Product Review Table"
           engine="innodb">
        <column xsi:type="int" name="review_id" padding="10" unsigned="true"
                identity="true" nullable="false" primary="true"/>
        <column xsi:type="int" name="product_id" padding="10" unsigned="true"
                nullable="false"/>
        <column xsi:type="varchar" name="reviewer_name" length="100"
                nullable="false"/>
        <column xsi:type="int" name="rating" padding="1" unsigned="true"
                nullable="false" default="5"/>
        <column xsi:type="text" name="review_text" nullable="true"/>
        <column xsi:type="timestamp" name="created_at" nullable="false"
                on_update="false" default="CURRENT_TIMESTAMP"/>
        <constraint xsi:type="foreign"
                     referenceId="TRAINING_REVIEW_PRODUCT_ID"
                     table="training_review"
                     column="product_id"
                     referenceTable="catalog_product_entity"
                     referenceColumn="entity_id"
                     onDelete="CASCADE"/>
    </table>
</schema>
```

> **Why `engine="innodb"`?** Magento requires InnoDB for its row-level locking and transaction support. MyISAM doesn't support foreign keys or transactions properly.

> **Why `onDelete="CASCADE"`?** When a product is deleted, all its reviews should be deleted too. Without CASCADE, you'd have orphan records or require manual cleanup.

**Column Types:**

| Type | SQL Equivalent | Use For |
|------|---------------|---------|
| `int` | INT | IDs, counts |
| `varchar` | VARCHAR | Short strings |
| `text` | TEXT | Long text, HTML |
| `decimal` | DECIMAL(12,4) | Prices, weights |
| `timestamp` | TIMESTAMP | Dates |
| `boolean` | TINYINT(1) | Flags |
| `blob` | BLOB | Binary data |

> **Common Pitfall:** Using `datetime` instead of `timestamp`. `datetime` is not timezone-aware. Magento stores everything in UTC. Use `timestamp` for dates, or `datetime` only when you explicitly need non-UTC storage.

**Generate Whitelist:**

```bash
bin/magento setup:db-declaration:generate-whitelist training_review
```

This creates `etc/db_schema_whitelist.json` — **commit this file**.

> **Why whitelist?** The whitelist tracks which schema entries have been applied. Without it, Magento can't determine what changed between module versions. It's the "source of truth" for what the database should look like.

> **Best Practice:** Always generate the whitelist AFTER creating or modifying `db_schema.xml`. Commit the whitelist alongside your schema changes.

**Apply Changes:**

```bash
bin/magento setup:upgrade
bin/magento setup:db:status  # Shows current schema state
```

> **Pro Tip:** `setup:db:status` tells you if there are pending schema or data patches. Run it after `setup:upgrade` to confirm everything applied.

---

### Topic 2: Data Patches & Schema Patches

**Why Patches Over Install/Upgrade Scripts?**

| Old Approach (Setup Scripts) | New Approach (Patches) |
|-----------------------------|------------------------|
| `InstallSchema` runs once | Re-runnable |
| Version-based sequencing | Explicit dependencies |
| No rollback mechanism | Manual but cleaner |
| Hard to trace execution state | `setup:db:status` shows state |

> **Why `DataPatchInterface` vs `SchemaPatchInterface`?** `DataPatchInterface` handles *data* (inserts, updates). `SchemaPatchInterface` handles *structure* (add column, modify table). Use the correct interface for the job.

**Data Patch — Insert Initial Data:**

```php
<?php
declare(strict_types=1);

// Setup/Patch/Data/AddSampleReviews.php
namespace Training\Review\Setup\Patch\Data;

use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class AddSampleReviews implements DataPatchInterface
{
    private ModuleDataSetupInterface $moduleDataSetup;

    public function __construct(ModuleDataSetupInterface $moduleDataSetup)
    {
        $this->moduleDataSetup = $moduleDataSetup;
    }

    public function apply(): void
    {
        $this->moduleDataSetup->getConnection()->insertMultiple('training_review', [
            ['product_id' => 1, 'reviewer_name' => 'Alice', 'rating' => 5, 'review_text' => 'Great product!'],
            ['product_id' => 1, 'reviewer_name' => 'Bob',   'rating' => 4, 'review_text' => 'Good value'],
            ['product_id' => 2, 'reviewer_name' => 'Carol', 'rating' => 3, 'review_text' => 'Average'],
        ]);
    }

    public function getAliases(): array { return []; }
    public static function getDependencies(): array { return []; }
}
```

> **Pro Tip:** Use `insertMultiple()` for bulk inserts — it's more efficient than multiple `insert()` calls. For single row, use `insert()`.

**Schema Patch — Modify Existing Table:**

```php
<?php
declare(strict_types=1);

// Setup/Patch/Schema/AddVerifiedColumn.php
namespace Training\Review\Setup\Patch\Schema;

use Magento\Framework\Setup\SchemaPatchInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class AddVerifiedColumn implements SchemaPatchInterface
{
    private ModuleDataSetupInterface $moduleDataSetup;

    public function __construct(ModuleDataSetupInterface $moduleDataSetup)
    {
        $this->moduleDataSetup = $moduleDataSetup;
    }

    public function apply(): void
    {
        $this->moduleDataSetup->getConnection()->addColumn(
            'training_review',
            'is_verified',
            [
                'type' => \Magento\Framework\DB\Ddl\Table::TYPE_SMALLINT,
                'nullable' => false,
                'default' => 0,
                'comment' => 'Verified Purchase Flag'
            ]
        );
    }

    public function getAliases(): array { return []; }
    public static function getDependencies(): array { return []; }
}
```

> **Why Schema Patches?** If you later add the same column via `db_schema.xml` modification, Magento's declarative schema will detect the column already exists and skip creation — no conflict.

**Patch Dependencies:**

```php
public static function getDependencies(): array
{
    return [
        \Training\Review\Setup\Patch\Data\AddSampleReviews::class
    ];
}
```

> **Why dependencies?** If your patch requires another patch to run first (e.g., you need data from another table), declare it as a dependency.

**Patch Aliases (for Renaming):**

```php
public function getAliases(): array
{
    return ['old_patch_name'];
}
```

> **Why aliases?** If you rename a patch file, Magento loses track of whether it ran. Adding the old name as an alias tells Magento "this is the same patch, skip it."

**Running Patches:**

```bash
bin/magento setup:upgrade --keep-generated
bin/magento setup:db:status
```

> **Pro Tip:** Use `--keep-generated` to preserve generated classes during development. Without it, `setup:upgrade` regenerates everything (slower).

**Patch Ordering — `getDependencies()` and Version-based Ordering:**

Magento processes patches in two stages:
1. **Version-based ordering** — patches in `Setup/Patch/Data/` and `Setup/Patch/Schema/` are sorted by module version (from `module.xml`'s `<sequence>` and module version number)
2. **Explicit dependencies** — `getDependencies()` can force ordering across modules

```php
// Module A patch
public static function getDependencies(): array
{
    return [
        \Training\Other\Setup\Patch\Data\SomePatch::class  // Runs before this patch
    ];
}
```

> **北极星 (Polaris) Patch Ordering Principle:** If Patch B depends on Patch A, declare `PatchB::getDependencies()` to return `[PatchA::class]`. Never rely on filename alphabetically sorting correctly — explicit dependencies are the only reliable ordering mechanism.

**Patch Ordering Example — Multi-module Dependencies:**

```
Module: Training_Review (v1.0.0)
  └── Setup/Patch/Data/CreateReviewTable.php       ← No dependencies
  └── Setup/Patch/Data/AddSampleReviews.php        ← Depends on CreateReviewTable

Module: Training_Promotion (v1.0.0)
  └── Setup/Patch/Data/AddPromotionReviews.php    ← Depends on Training_Review data
```

```php
// AddPromotionReviews.php
public static function getDependencies(): array
{
    return [
        \Training\Review\Setup\Patch\Data\AddSampleReviews::class
    ];
}
```

> **When to use setup vs upgrade scripts (legacy):** Only use `InstallSchema`/`UpgradeSchema` when your module has NO upgrade path from a pre-2.3 version. For all new modules targeting 2.3+, always use declarative schema (`db_schema.xml`) + patches. The legacy install/upgrade scripts are harder to maintain, harder to test, and provide no rollback mechanism.

**Patch Idempotency — Making Patches Re-Runnable:**

A good patch is idempotent — running it twice produces the same result as running it once:

```php
// BAD — fails if column already exists
public function apply(): void
{
    $this->moduleDataSetup->getConnection()->addColumn(...);
}

// GOOD — check before modifying
public function apply(): void
{
    $connection = $this->moduleDataSetup->getConnection();
    if (!$connection->tableColumnExists('training_review', 'is_verified')) {
        $connection->addColumn(...);
    }
}
```

> **Pro Tip:** Declarative schema (`db_schema.xml`) handles idempotency automatically for structural changes. Schema patches that modify columns/tables should still check existence to avoid errors on re-run. Data patches that insert records should check for existing data before re-inserting.

**Whitelist Generation After Schema Changes:**

Whenever you change `db_schema.xml` (add/modify/remove a column or table), regenerate the whitelist:

```bash
# Generate for a specific module
bin/magento setup:db-declaration:generate-whitelist --module-name=Training_Review

# Check current status
bin/magento setup:db:status
```

**When NOT to use Declarative Schema (Upgrades Requiring Data Migration):**

Declarative schema handles structure changes automatically. But when you need to **migrate data** during an upgrade (not just structure), a schema patch is not enough:

| Scenario | Approach |
|----------|----------|
| Rename column and migrate data | Schema patch (structure) + Data patch (migrate values) |
| Split one column into two | Schema patch (add new columns) + Data patch (copy data) |
| Backfill new column from old column | Data patch with `UPDATE` query |
| Delete column and archive data | Data patch (archive first) + Schema patch (drop column) |

Declarative schema will drop columns marked for removal — if you need the data, migrate it in a data patch BEFORE the schema patch runs (use `getDependencies()` to order them).

**Whitelist Generation Importance:**

The whitelist (`etc/db_schema_whitelist.json`) is not optional. Without it:
- `setup:db:status` cannot determine whether a column from `db_schema.xml` was already applied
- In upgrade scenarios, Magento may try to re-apply a column that already exists → errors
- Uninstall/reinstall of the module cannot cleanly remove the schema

> **Best Practice:** Always generate the whitelist immediately after any `db_schema.xml` change, and commit it alongside the schema change. The whitelist is the source of truth for what's been applied to the database.

---

### Topic 3: Model, ResourceModel & Collection

**The Triad:**

| Class | Responsibility |
|-------|---------------|
| Model | Entity logic, holds data, knows nothing about DB |
| ResourceModel | Database operations (CRUD for a specific table) |
| Collection | Set of Model instances, supports filtering/sorting/pagination |

> **Why this separation?** Single Responsibility. The Model doesn't know about SQL. The ResourceModel doesn't know about business rules. The Collection doesn't know about individual entities. This makes each piece testable in isolation.

**Model:**

```php
<?php
declare(strict_types=1);

// Model/Review.php
namespace Training\Review\Model;

use Magento\Framework\Model\AbstractModel;

class Review extends AbstractModel
{
    protected function _construct(): void
    {
        $this->_init(\Training\Review\Model\ResourceModel\Review::class);
    }

    // Custom business logic
    public function isVerified(): bool
    {
        return (bool) $this->getData('is_verified');
    }
}
```

> **Why extend `AbstractModel`?** It provides `getData()`, `setData()`, `getId()`, `save()`, `delete()` — the standard model interface. Your concrete Model adds business logic only.

**ResourceModel:**

```php
<?php
declare(strict_types=1);

// Model/ResourceModel/Review.php
namespace Training\Review\Model\ResourceModel;

use Magento\Framework\Model\ResourceModel\Db\AbstractDb;

class Review extends AbstractDb
{
    protected function _construct(): void
    {
        $this->_init('training_review', 'review_id');
    }
}
```

> **Why `_init('table_name', 'primary_key')`?** This tells the ResourceModel which table and which column is the primary key. The ResourceModel uses this for `load()`, `save()`, `delete()` operations.

**Collection:**

```php
<?php
declare(strict_types=1);

// Model/ResourceModel/Review/Collection.php
namespace Training\Review\Model\ResourceModel\Review;

use Magento\Framework\Model\ResourceModel\Db\Collection\AbstractCollection;

class Collection extends AbstractCollection
{
    protected function _construct(): void
    {
        $this->_init(
            \Training\Review\Model\Review::class,
            \Training\Review\Model\ResourceModel\Review::class
        );
    }
}
```

> **Why does Collection take BOTH Model and ResourceModel?** The Collection uses the Model class for individual items and the ResourceModel for the `load()` logic that retrieves from the database.

**Using the Collection:**

```php
$collection = $this->reviewCollectionFactory->create();
$collection->addFieldToFilter('product_id', 1);
$collection->setOrder('created_at', 'DESC');
$collection->setPageSize(10)->setCurPage(1);

foreach ($collection as $review) {
    echo $review->getReviewerName();
}
```

> **Common Pitfall:** Iterating over a collection without `setPageSize()`. On large tables, this loads ALL records into memory. Always paginate: `setPageSize(100)->setCurPage(1)`.

> **Pro Tip:** Collections are lazy-loaded — they don't hit the database until you iterate or call `load()`. You can build filters incrementally without performance cost.

---

### Topic 4: Service Contracts (Interface-First Design)

**Why Service Contracts?**

| Without Interfaces | With Interfaces |
|-------------------|-----------------|
| Direct model dependency | Depend on interface |
| Hard to unit test | Easy to mock |
| Internal structure leaked | Internal structure hidden |
| Breaks when you refactor | Refactor safely |

> **Why interface-first?** When you depend on an interface, the implementation can change without breaking consumers. You can swap `ReviewRepository` with `CachedReviewRepository` by changing `di.xml` — no code changes in consuming classes.

> **Pro Tip:** Any class used by other modules should be exposed via a service contract. If another module uses your Model directly, you're locked into that structure. Service contracts provide a stable API.

**Data Interface — `Api/Data/ReviewInterface.php`:**

```php
<?php
declare(strict_types=1);

namespace Training\Review\Api\Data;

interface ReviewInterface
{
    public function getReviewId(): int;
    public function setReviewId(int $id): self;
    public function getProductId(): int;
    public function setProductId(int $id): self;
    public function getReviewerName(): string;
    public function setReviewerName(string $name): self;
    public function getRating(): int;
    public function setRating(int $rating): self;
    public function getReviewText(): string;
    public function setReviewText(string $text): self;
    public function getCreatedAt(): string;
    public function setCreatedAt(string $date): self;
    public function getIsVerified(): bool;
    public function setIsVerified(bool $verified): self;
}
```

> **Why all getters AND setters?** The data interface is a Plain Old PHP Object (POPO) — a pure data container. No business logic, no dependencies. This makes it serializable and cacheable.

**Repository Interface — `Api/ReviewRepositoryInterface.php`:**

```php
<?php
declare(strict_types=1);

namespace Training\Review\Api;

use Training\Review\Api\Data\ReviewInterface;
use Magento\Framework\Api\SearchCriteriaInterface;
use Magento\Framework\Api\SearchResultsInterface;

interface ReviewRepositoryInterface
{
    public function save(ReviewInterface $review): ReviewInterface;
    public function getById(int $reviewId): ReviewInterface;
    public function delete(ReviewInterface $review): bool;
    public function deleteById(int $reviewId): bool;
    public function getList(SearchCriteriaInterface $searchCriteria): SearchResultsInterface;
}
```

> **Why return `ReviewInterface` not `Review`?** Consumers should only know about the interface. If you return the concrete class, consumers can access internal methods that might change.

**Wiring in `di.xml`:**

```xml
<preference for="Training\Review\Api\ReviewRepositoryInterface"
            type="Training\Review\Model\ReviewRepository"/>
<preference for="Training\Review\Api\Data\ReviewInterface"
            type="Training\Review\Model\Review"/>
```

> **Common Pitfall:** Forgetting to declare the preference. Without it, Magento doesn't know which implementation to use when `ReviewRepositoryInterface` is injected.

**Why `Api/` Interfaces Actually Matter (Beyond Decoupling):**

The interfaces in `Api/` are not just about testability — they're the foundation for Magento's integration layer:

| Integration Point | Why It Needs the Interface |
|-------------------|--------------------------|
| **GraphQL** | GraphQL resolvers bind to `webapi.xml` which references `Api/` interfaces. Without the interface, you cannot expose the entity via GraphQL. |
| **REST/JSON API** | `webapi.xml` `<routes>` specify a service method. The route maps to an interface method. Consumers get a stable contract. |
| **SOAP** | Similarly bound via `webapi.xml`. Interface defines the WSDL contract. |
| **Dependency Injection** | When another module injects `ReviewRepositoryInterface`, it gets the configured implementation via `di.xml` preference — no concrete class reference. |
| **Service Layer** | Business logic in other modules depends on the interface, never the concrete repository class. |

```xml
<!-- etc/webapi.xml -->
<routes>
    <route url="/V1/reviews/:id" method="GET">
        <service class="Training\Review\Api\ReviewRepositoryInterface" method="getById"/>
        <resources>
            <resource ref="anonymous"/>
        </resources>
    </route>
</routes>
```

Without `ReviewRepositoryInterface` in `Api/`, there is no way to expose the entity via Magento's web API. The interface is the binding contract.

**Service Contract Granularity:**

Service contracts should be granular — one interface per entity, with methods that map to repository operations. Avoid "god interfaces" with dozens of methods. If the interface grows beyond 8-10 methods, consider splitting into separate interfaces (e.g., `ReadOnlyReviewInterface`, `WriteReviewInterface`).

**Interface Stability Rule:**

Once an interface method is public, never change its signature in a way that breaks consumers. If you need a new parameter, add a new method (e.g., `saveAndSendEmail()`) rather than modifying `save()`. This is the Interface Segregation Principle applied to Magento's service contracts.

---

### Topic 5: Repository Implementation & SearchCriteria

**Full Repository with Error Handling:**

```php
<?php
declare(strict_types=1);

// Model/ReviewRepository.php
namespace Training\Review\Model;

use Training\Review\Api\ReviewRepositoryInterface;
use Training\Review\Api\Data\ReviewInterface;
use Training\Review\Api\Data\ReviewInterfaceFactory;
use Training\Review\Model\ResourceModel\Review as ResourceModel;
use Magento\Framework\Api\SearchCriteriaInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Framework\Api\SearchResultsInterfaceFactory;
use Magento\Framework\Exception\CouldNotSaveException;
use Magento\Framework\Exception\NoSuchEntityException;
use Training\Review\Model\ResourceModel\Review\CollectionFactory;

class ReviewRepository implements ReviewRepositoryInterface
{
    private ResourceModel $resourceModel;
    private ReviewInterfaceFactory $reviewFactory;
    private SearchResultsInterfaceFactory $searchResultsFactory;
    private SearchCriteriaBuilder $searchCriteriaBuilder;
    private CollectionFactory $reviewCollectionFactory;

    public function __construct(
        ResourceModel $resourceModel,
        ReviewInterfaceFactory $reviewFactory,
        SearchResultsInterfaceFactory $searchResultsFactory,
        SearchCriteriaBuilder $searchCriteriaBuilder,
        CollectionFactory $reviewCollectionFactory
    ) {
        $this->resourceModel = $resourceModel;
        $this->reviewFactory = $reviewFactory;
        $this->searchResultsFactory = $searchResultsFactory;
        $this->searchCriteriaBuilder = $searchCriteriaBuilder;
        $this->reviewCollectionFactory = $reviewCollectionFactory;
    }

    public function save(ReviewInterface $review): ReviewInterface
    {
        try {
            $this->resourceModel->save($review);
        } catch (\Exception $e) {
            throw new CouldNotSaveException(
                __('Cannot save review: %1', $e->getMessage())
            );
        }
        return $review;
    }

    public function getById(int $reviewId): ReviewInterface
    {
        $review = $this->reviewFactory->create();
        $this->resourceModel->load($review, $reviewId);
        if (!$review->getReviewId()) {
            throw new NoSuchEntityException(
                __('Review with ID %1 not found', $reviewId)
            );
        }
        return $review;
    }

    public function getList(SearchCriteriaInterface $searchCriteria): \Magento\Framework\Api\SearchResultsInterface
    {
        $collection = $this->reviewCollectionFactory->create();
        $this->applySearchCriteria($collection, $searchCriteria);

        $searchResult = $this->searchResultsFactory->create();
        $searchResult->setItems($collection->getItems());
        $searchResult->setTotalCount($collection->getSize());
        $searchResult->setSearchCriteria($searchCriteria);
        return $searchResult;
    }

    private function applySearchCriteria($collection, $searchCriteria)
    {
        foreach ($searchCriteria->getFilterGroups() as $filterGroup) {
            foreach ($filterGroup->getFilters() as $filter) {
                $collection->addFieldToFilter(
                    $filter->getField(),
                    [$filter->getConditionType() => $filter->getValue()]
                );
            }
        }
    }

    public function delete(ReviewInterface $review): bool
    {
        try {
            $this->resourceModel->delete($review);
        } catch (\Exception $e) {
            return false;
        }
        return true;
    }

    public function deleteById(int $reviewId): bool
    {
        $review = $this->getById($reviewId);
        return $this->delete($review);
    }
}
```

> **Why `ReviewInterfaceFactory` instead of `ReviewFactory`?** Factories generated by Magento's DI system. The `InterfaceFactory` pattern returns an instance of the concrete class (Review) but typed as the interface (ReviewInterface). This keeps consumers decoupled from the implementation.

> **Best Practice:** Always wrap `save()` in try-catch and throw `CouldNotSaveException`. Don't expose raw `\Exception` — consumers shouldn't need to know your storage mechanism.

**SearchCriteria Usage:**

```php
$criteria = $this->searchCriteriaBuilder
    ->addFilter('product_id', 1)
    ->addFilter('rating', 3, 'gteq')   // >= 3
    ->addFilter('is_verified', 1)
    ->addSortOrder('created_at', 'DESC')
    ->setPageSize(10)
    ->setCurrentPage(1)
    ->create();

$result = $this->reviewRepository->getList($criteria);

// Access results
$reviews = $result->getItems();
$totalCount = $result->getTotalCount();
```

**Filter Operators:**

| Operator | Meaning |
|----------|---------|
| `eq` | Equals |
| `neq` | Not equals |
| `like` | SQL LIKE (use `%` for wildcards) |
| `in` | IN array (pass array) |
| `nin` | NOT IN array |
| `gt`, `gteq` | Greater than / Greater than or equal |
| `lt`, `lteq` | Less than / Less than or equal |
| `null` | IS NULL |
| `notnull` | IS NOT NULL |

> **Common Pitfall:** `like` without escaping `%`. If your user input contains `%` or `_` (which are LIKE wildcards), wrap them: `addFilter('name', '%' . $escaped . '%', 'like')`.

**Why Repositories Over Direct Model Access:**

| Aspect | Direct Model Access | Repository Pattern |
|--------|--------------------|--------------------|
| Testability | Hard to unit test — model has DB dependencies | Easy to mock — depend on interface |
| Coupling | Couples consumer to model internals | Consumer depends only on interface |
| Query encapsulation | Query logic scattered across consumers | Single place for all query logic |
| Caching | No standard caching | Can add caching via `di.xml` preference swap |
| Transaction support | Manual per-save | Can wrap multi-operation transactions |
| API stability | Model can change, breaking consumers | Interface is stable |

**SearchCriteriaBuilder vs Hardcoded Filters:**

Avoid embedding filter logic directly in consumers. Use `SearchCriteriaBuilder` to let callers construct queries dynamically:

```php
// BAD — hardcoded filter in consumer
$collection = $this->reviewCollectionFactory->create();
$collection->addFieldToFilter('product_id', $productId);
$collection->addFieldToFilter('rating', 5, 'gteq');

// GOOD — consumer passes SearchCriteria, repository interprets it
$criteria = $this->searchCriteriaBuilder
    ->addFilter('product_id', $productId)
    ->addFilter('rating', 5, 'gteq')
    ->create();
$result = $this->reviewRepository->getList($criteria);
```

The repository exposes one method (`getList(SearchCriteriaInterface)`) that handles all filtering via the standard `SearchCriteria` contract. Callers that need different filters don't modify the repository — they modify the `SearchCriteria` they pass.

**Using `getList()` Properly:**

`getList()` returns a `SearchResultsInterface` — not a raw collection. The `SearchResults` object contains:
- `getItems()` — array of entity objects (e.g., `ReviewInterface[]`)
- `getTotalCount()` — total matching records (ignoring pagination)
- `getSearchCriteria()` — the original `SearchCriteria` passed in (useful for debugging)

```php
/** @var \Magento\Framework\Api\SearchResultsInterface $result */
$result = $this->reviewRepository->getList($criteria);

$reviews = $result->getItems();        // Array of ReviewInterface
$total = $result->getTotalCount();     // For pagination UI

// Never do this with SearchResults:
foreach ($result as $review) { } // ❌ SearchResults is NOT iterable
```

**Pagination with SearchCriteria:**

```php
// Page 2, 10 items per page
$criteria = $this->searchCriteriaBuilder
    ->addFilter('product_id', 1)
    ->setPageSize(10)
    ->setCurrentPage(2)
    ->create();

$result = $this->reviewRepository->getList($criteria);

// In the UI:
echo "Page 2 of " . ceil($result->getTotalCount() / 10);
```

> **Best Practice:** Always check `getTotalCount()` before rendering pagination controls. If `getTotalCount()` is 0, show "No results" instead of an empty grid.

**Filter Groups (AND vs OR Logic):**

`SearchCriteria` has `FilterGroups`. By default, filters within a `FilterGroup` are AND'd together. Multiple `FilterGroup`s are OR'd together:

```php
// Filter: (product_id = 1 AND rating >= 3) OR (product_id = 2 AND rating >= 4)

$group1 = $this->filterBuilder
    ->addFilter('product_id', 1)
    ->addFilter('rating', 3, 'gteq')
    ->create();

$group2 = $this->filterBuilder
    ->addFilter('product_id', 2)
    ->addFilter('rating', 4, 'gteq')
    ->create();

$criteria = $this->searchCriteriaBuilder
    ->setFilterGroups([$group1, $group2])
    ->create();
```

> **Note:** Complex OR logic across many filter groups can generate slow SQL. If you find yourself with 5+ filter groups, consider whether the query can be restructured or if a custom indexer is needed.

**Sorting:**

```php
$criteria = $this->searchCriteriaBuilder
    ->addSortOrder('created_at', 'DESC')
    ->addSortOrder('review_id', 'ASC')  // Tiebreaker
    ->create();
```

> **Common Pitfall:** Forgetting to set a deterministic sort order. Without it, pagination across large result sets can return inconsistent rows (especially on non-indexed columns), causing duplicate or missing items in paginated grids.

**Sorting and filtering should always be on indexed columns.** Check `EXPLAIN` output if a query feels slow — sorting on a non-indexed `varchar` column with `LIKE` can be catastrophic for performance at scale.

---

### Topic 6: EAV Architecture

**Why EAV?**

Flat tables have fixed columns. EAV stores attribute definitions separately so different products/customers can have different attributes without altering the table schema.

**EAV Table Structure:**

```
eav_attribute              ← Attribute definitions (name, type, validation)
catalog_product_entity      ← Core entity rows (product_id, created_at)
catalog_product_entity_varchar   ← String values
catalog_product_entity_int      ← Integer values
catalog_product_entity_decimal  ← Decimal values (prices)
catalog_product_entity_text     ← Text/HTML values
catalog_product_entity_datetime ← Date/time values
```

Each value table: `(entity_id, attribute_id, value)`

**EAV vs Flat Table:**

| Scenario | Use |
|----------|-----|
| Custom entity with fixed columns | Flat table (`db_schema.xml`) — simpler |
| Product, customer, category | EAV — dynamic attributes |
| High-volume data (millions rows) | Flat table — EAV has overhead |
| Admin-managed attributes | EAV — attributes created in admin |

**Adding a Custom Product Attribute:**

```php
<?php
// Setup/Patch/Data/AddReviewRatingAttribute.php
namespace Training\Review\Setup\Patch\Data;

use Magento\Catalog\Model\Product;
use Magento\Eav\Setup\EavSetup;
use Magento\Eav\Setup\EavSetupFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class AddReviewRatingAttribute implements DataPatchInterface
{
    protected $moduleDataSetup;
    protected $eavSetupFactory;

    public function __construct(
        ModuleDataSetupInterface $moduleDataSetup,
        EavSetupFactory $eavSetupFactory
    ) {
        $this->moduleDataSetup = $moduleDataSetup;
        $this->eavSetupFactory = $eavSetupFactory;
    }

    public function apply(): void
    {
        $eavSetup = $this->eavSetupFactory->create(['setup' => $this->moduleDataSetup]);
        $eavSetup->addAttribute(Product::ENTITY, 'allow_review', [
            'type' => 'int',
            'label' => 'Allow Customer Reviews',
            'input' => 'select',
            'source' => \Magento\Eav\Model\Entity\Attribute\Source\Boolean::class,
            'global' => \Magento\Catalog\Model\ResourceModel\Eav\Attribute::SCOPE_GLOBAL,
            'visible' => true,
            'required' => false,
            'user_defined' => true,
            'default' => 1,
        ]);
    }

    public function getAliases(): array { return []; }
    public static function getDependencies(): array { return []; }
}
```

**Reading Custom Attributes:**

```php
$product = $this->product->getProduct();
$allowReview = $product->getData('allow_review');
// or
$allowReview = $product->getAllowReview();
```

**EAV Drawbacks — When NOT to Use EAV:**

EAV is powerful but comes with significant costs:

| Drawback | Impact |
|----------|--------|
| **Join complexity** | Every attribute access requires a JOIN across 1-3 EAV tables. A single product with 20 custom attributes generates 20+ JOINs per load. |
| **Index performance** | Filtering on EAV attributes requires JOINs before applying WHERE clauses. Without proper indexing, queries can be 10-100x slower than flat table equivalents. |
| **Storage overhead** | EAV tables store `entity_id`, `attribute_id`, `value` per attribute per entity. A product with 50 attributes stores 50 rows (not 1 row). |
| **Attribute scoping** | EAV scope resolution (`SCOPE_GLOBAL`, `SCOPE_WEBSITE`) adds complexity to value retrieval. |
| **Backup & migration** | Backing up EAV tables is larger and slower due to row multiplication. |

**When to Use Flat Tables vs EAV:**

| Scenario | Recommended Approach |
|----------|---------------------|
| Custom entity with fixed, known columns | Flat table via `db_schema.xml` — simpler, faster |
| Entity attributes known at module build time | Flat table — no runtime flexibility needed |
| Millions of records with simple attributes | Flat table — EAV overhead is unsustainable |
| Admin users need to create attributes at runtime | EAV — attribute_set flexibility required |
| Attributes vary per product/customer/category | EAV — no fixed schema possible |
| High-volume filtering on custom attributes | Flat table + custom indexer — EAV JOINs won't scale |

> **Rule of Thumb:** If you know all attributes at development time and your entity is not `product`, `customer`, or `category`, prefer a flat table. Reserve EAV for entities that genuinely need runtime attribute creation and admin-managed attribute sets.

---

### Topic 7: EAV Beyond Products — Customer & Category Attributes

**Customer Attributes:**

Product EAV was covered in Topic 6. But Magento's EAV system applies equally to customers and categories. Customer attributes are stored across these tables:

| Table | Stores |
|-------|--------|
| `customer_entity` | Core customer data |
| `customer_entity_varchar` | String values (e.g., `nickname`, `twitter`) |
| `customer_entity_int` | Integer values (e.g., `points`, `loyalty_tier`) |
| `customer_entity_decimal` | Decimal values |
| `customer_entity_text` | Text values (e.g., `bio`) |
| `customer_entity_datetime` | Date/time values |

**Adding a Customer Attribute:**

```php
<?php
// Setup/Patch/Data/AddCustomerLoyaltyTier.php
namespace Training\Review\Setup\Patch\Data;

use Magento\Customer\Model\Customer;
use Magento\Eav\Setup\EavSetup;
use Magento\Eav\Setup\EavSetupFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class AddCustomerLoyaltyTier implements DataPatchInterface
{
    protected $moduleDataSetup;
    protected $eavSetupFactory;

    public function __construct(
        ModuleDataSetupInterface $moduleDataSetup,
        EavSetupFactory $eavSetupFactory
    ) {
        $this->moduleDataSetup = $moduleDataSetup;
        $this->eavSetupFactory = $eavSetupFactory;
    }

    public function apply(): void
    {
        $eavSetup = $this->eavSetupFactory->create(['setup' => $this->moduleDataSetup]);
        $eavSetup->addAttribute(
            Customer::ENTITY,
            'loyalty_tier',
            [
                'type' => 'int',
                'label' => 'Loyalty Tier',
                'input' => 'select',
                'source' => \Training\Review\Model\Customer\Source\LoyaltyTier::class,
                'global' => \Magento\Catalog\Model\ResourceModel\Eav\Attribute::SCOPE_WEBSITE,
                'default' => 0,
                'visible' => true,
                'required' => false,
                'user_defined' => true,
            ]
        );
    }

    public function getAliases(): array { return []; }
    public static function getDependencies(): array { return []; }
}
```

**Custom Source Model for Customer Attribute:**

```php
<?php
// Model/Customer/Source/LoyaltyTier.php
namespace Training\Review\Model\Customer\Source;

use Magento\Eav\Model\Entity\Attribute\Source\AbstractSource;

class LoyaltyTier extends AbstractSource
{
    public function getAllOptions(): array
    {
        if ($this->_options === null) {
            $this->_options = [
                ['value' => 0, 'label' => __('Standard')],
                ['value' => 1, 'label' => __('Silver')],
                ['value' => 2, 'label' => __('Gold')],
                ['value' => 3, 'label' => __('Platinum')],
            ];
        }
        return $this->_options;
    }
}
```

**Category Attributes:**

```php
<?php
// Setup/Patch/Data/AddCategoryFeaturedAttribute.php
namespace Training\Review\Setup\Patch\Data;

use Magento\Catalog\Model\Category;
use Magento\Eav\Setup\EavSetup;
use Magento\Eav\Setup\EavSetupFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class AddCategoryFeaturedAttribute implements DataPatchInterface
{
    protected $moduleDataSetup;
    protected $eavSetupFactory;

    public function __construct(
        ModuleDataSetupInterface $moduleDataSetup,
        EavSetupFactory $eavSetupFactory
    ) {
        $this->moduleDataSetup = $moduleDataSetup;
        $this->eavSetupFactory = $eavSetupFactory;
    }

    public function apply(): void
    {
        $eavSetup = $this->eavSetupFactory->create(['setup' => $this->moduleDataSetup]);
        $eavSetup->addAttribute(Category::ENTITY, 'is_featured', [
            'type' => 'int',
            'label' => 'Featured Category',
            'input' => 'select',
            'source' => \Magento\Eav\Model\Entity\Attribute\Source\Boolean::class,
            'global' => \Magento\Catalog\Model\ResourceModel\Eav\Attribute::SCOPE_GLOBAL,
            'visible' => true,
            'required' => false,
            'default' => 0,
        ]);
    }

    public function getAliases(): array { return []; }
    public static function getDependencies(): array { return []; }
}
```

**Reading Custom Customer Attributes:**

```php
<?php
$customer = $this->customerSession->getCustomer();
$loyaltyTier = $customer->getData('loyalty_tier');
// or
$loyaltyTier = $customer->getCustomAttribute('loyalty_tier')?->getValue();
```

**Key Difference from Product EAV:**

| Aspect | Product EAV | Customer/Category EAV |
|--------|-------------|----------------------|
| Attribute sets | Yes (grouping) | No |
| Default entity | `Catalog\Product` | `Customer::ENTITY` or `Category::ENTITY` |
| Admin visibility | Product edit page | Customer account page |
| Searchable | Yes (if indexed) | Yes (if indexed) |
| Scope | `SCOPE_GLOBAL` / `SCOPE_WEBSITE` | `SCOPE_WEBSITE` (customer) |

---

### Topic X: Extension Attributes — Adding API Fields to Entities

**What Extension Attributes Are:**
- A mechanism for adding custom fields to existing API entities (Product, Customer, Order, Quote, etc.)
- Different from custom attributes (which are EAV-based) — extension attributes use dedicated extension attribute tables
- Declared in `extension_attributes.xml` files
- Automatically included in GraphQL/REST API responses when the entity is loaded
- The "right way" to expose custom data on standard Magento entities without modifying core code

**Why Extension Attributes Matter for API Development:**
- If you want to add a custom field to a Product and expose it via GraphQL, you MUST use extension attributes
- Custom attributes (EAV) appear in the admin panel but are NOT automatically in APIs
- Extension attributes are the API bridge — they make custom data accessible to headless frontends

**How to Implement Extension Attributes:**

1. **Declare in extension_attributes.xml:**
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ExtensionAttribute/etc/extension_attributes.xsd">
    <extension attribute for="Magento\Catalog\Api\Data\ProductInterface">
        <attribute name="warranty_period" data-type="string"/>
    </extension attribute>
</config>
```

2. **Add to db_schema.xml:**
```xml
<table name="catalog_product_entity" resource="default">
    <column xsi:type="varchar" name="warranty_period" nullable="true" length="50"/>
</table>
```

3. **Implement in your Product model:**
```php
use Magento\Catalog\Api\Data\ProductExtensionInterface;

class Product extends \Magento\Catalog\Model\Product
    implements \Magento\Catalog\Api\Data\ProductInterface {

    public function getWarrantyPeriod(): ?string {
        return $this->getData('warranty_period');
    }

    public function setWarrantyPeriod(?string $period): static {
        return $this->setData('warranty_period', $period);
    }
}
```

4. **For GraphQL: Wire a resolver:**
In `etc/graphql/di.xml`:
```xml
<type name="Magento\Catalog\Model\Product">
    <plugin name="Training_ProductWarranty_WarrantyPlugin"/>
</type>
```

**Extension Attributes vs Custom Attributes Decision Matrix:**

| Aspect | Custom Attribute (EAV) | Extension Attribute |
|--------|----------------------|---------------------|
| Storage | EAV tables (catalog_product_entity_varchar, etc.) | Dedicated column in entity table |
| Admin UI | Automatic in product/customer form | Requires custom admin field |
| API Exposure | Not automatic | Automatic via extension_attributes |
| GraphQL | Requires custom resolver | Automatic with extension_attributes.xml |
| Performance | Multiple joins | Single column access |
| Use Case | Admin-managed attributes with UI | Programmatic/code-only fields exposed via API |

**Pro Tips:**
- Extension attributes are the correct pattern for headless — always use them when the field needs to be in API responses
- If you need admin UI for a field, add a custom attribute AND an extension attribute (the EAV for storage, extension for API exposure)
- Magento's Product model already has extension interface support — implement `getExtensionAttributes()` and `setExtensionAttributes()` properly

**Common Mistakes:**
- ❌ Adding fields via direct column in EAV tables without declaring extension_attributes.xml → field not in API
- ❌ Confusing extension attributes with custom attributes (EAV) → wrong storage pattern selected
- ❌ Not implementing `getExtensionAttributes()` on the model → API returns null for custom fields

---

### Topic 8: Transaction Patterns & SaveHandler

**The Problem Without Transactions:**

```php
<?php
// BAD — if savePoint() fails, partial data is committed
public function saveReview(ReviewInterface $review, array $additionalData): void
{
    $this->resourceModel->save($review);      // Commits immediately
    $this->saveAdditionalData($review, $additionalData); // If this fails, $review already saved
    $this->sendNotification($review);        // If this fails, first two already committed
}
```

**Using Database Transactions:**

```php
<?php
// GOOD — all or nothing
public function saveReview(ReviewInterface $review, array $additionalData): void
{
    $connection = $this->resourceModel->getConnection();
    $connection->beginTransaction();

    try {
        $this->resourceModel->save($review);
        $this->saveAdditionalData($review, $additionalData);
        $this->sendNotification($review);
        $connection->commit();
    } catch (\Exception $e) {
        $connection->rollBack();
        throw $e;
    }
}
```

**SaveHandler Pattern — Magento's Structured Save:**

For complex entities, implement `SaveHandlerInterface` to encapsulate all save logic including validation and side-effects:

```php
<?php
// Model/ReviewSaveHandler.php
namespace Training\Review\Model;

use Magento\Framework\Model\ResourceModel\Db\SaveHandlerInterface;
use Magento\Framework\Model\ResourceModel\Db\Context;
use Training\Review\Api\Data\ReviewInterface;
use Psr\Log\LoggerInterface;

class ReviewSaveHandler implements SaveHandlerInterface
{
    protected $resourceModel;
    protected $logger;

    public function __construct(
        \Training\Review\Model\ResourceModel\Review $resourceModel,
        LoggerInterface $logger
    ) {
        $this->resourceModel = $resourceModel;
        $this->logger = $logger;
    }

    public function save(\Magento\Framework\Model\AbstractModel $review): \Magento\Framework\Model\AbstractModel
    {
        $connection = $this->resourceModel->getConnection();
        $connection->beginTransaction();

        try {
            // Validate before any write
            $this->validate($review);

            // Trigger business logic before save
            $this->beforeSave($review);

            // Perform the save
            $this->resourceModel->save($review);

            // Trigger business logic after save
            $this->afterSave($review);

            $connection->commit();
        } catch (\Exception $e) {
            $connection->rollBack();
            $this->logger->error('Review save failed', [
                'review_id' => $review->getId(),
                'error' => $e->getMessage()
            ]);
            throw $e;
        }

        return $review;
    }

    protected function validate(ReviewInterface $review): void
    {
        if (strlen($review->getReviewerName()) < 2) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Reviewer name must be at least 2 characters')
            );
        }
        if ($review->getRating() < 1 || $review->getRating() > 5) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Rating must be between 1 and 5')
            );
        }
    }

    protected function beforeSave(ReviewInterface $review): void
    {
        if (!$review->getCreatedAt()) {
            $review->setCreatedAt(date('Y-m-d H:i:s'));
        }
    }

    protected function afterSave(ReviewInterface $review): void
    {
        // Reindex product review summary after save
        $this->indexerRegistry->get('training_review_product_summary')
            ->executeRow($review->getProductId());
    }
}
```

**Wiring SaveHandler in `di.xml`:**

```xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/di.xsd">
    <type name="Training\Review\Model\Review">
        <arguments>
            <argument name="saveHandler" xsi:type="object">Training\Review\Model\ReviewSaveHandler</argument>
        </arguments>
    </type>
</config>
```

**When to Use Transactions:**

| Situation | Use Transaction? |
|-----------|-----------------|
| Single model save | No (Magento handles it) |
| Multiple saves in sequence | Yes |
| Save + event dispatch (observer side-effects) | Yes |
| External API call after save | Yes (with queue for the external call) |
| Complex validation across multiple entities | Yes |

**The Golden Rule:** If you call multiple `$repository->save()` in sequence, wrap them in a transaction. If any fails, rollback all of them.


## Reading List

- [Magento Model Architecture](https://developer.adobe.com/commerce/php/development/components/entities/) — Model/ResourceModel/Collection
- [Declarative Schema](https://developer.adobe.com/commerce/php/development/components/declarative-schema/) — db_schema.xml
- [Service Contracts](https://developer.adobe.com/commerce/php/development/components/service-contracts/) — Interface-first design
- [EAV](https://developer.adobe.com/commerce/php/development/components/attributes/) — Custom attributes

---

## Edge Cases & Troubleshooting

| Issue | Symptom | Solution |
|-------|---------|----------|
| Table not created | Table missing from DB | Check db_schema.xml syntax, run `setup:upgrade` |
| Whitelist missing | Upgrade script fails | `setup:db-declaration:generate-whitelist` |
| Patch not running | Data not inserted | Check class name matches filename exactly |
| Repository class not found | DI compile error | `setup:di:compile` after adding new classes |
| EAV attribute not showing | Attribute missing in admin | Check attribute set assignment |

---

## Common Mistakes to Avoid

1. ❌ Creating table manually in SQL → Use db_schema.xml only
2. ❌ Forgetting whitelist generation → Schema changes won't apply
3. ❌ Repository returns raw model instead of interface → Return type must be interface
4. ❌ SearchCriteria filter on non-indexed column → Slow on large datasets
5. ❌ Forgetting `di.xml` preferences → Interface not wired to implementation
6. ❌ Using `datetime` instead of `timestamp` → Not timezone-aware, causes date bugs
7. ❌ Flat table for entity needing dynamic attributes → Use EAV for product/customer/category
8. ❌ Forgetting `SCOPE_WEBSITE` for customer attributes → Admin store scope breaks
9. ❌ Loading full collection without `setPageSize()` → Memory exhaustion on large tables
10. ❌ Direct model access in controllers → Always go through repository

**Pro Tips — Production Data Layer:**

1. **Foreign key indexes are not automatic.** Every `foreign` key in `db_schema.xml` should have an index on the referenced column. Without an index, `ON DELETE CASCADE` and JOINs on foreign keys are slow. Add explicit `<index>` elements for any foreign key columns.

2. **Batch process large collections.** When you need to process thousands of records, don't load them all at once:
```php
// BAD — memory exhaustion
$collection = $this->reviewCollectionFactory->create();
foreach ($collection as $review) { ... }

// GOOD — batch processing with pagination
$batchSize = 100;
$page = 1;
do {
    $collection = $this->reviewCollectionFactory->create();
    $collection->setPageSize($batchSize)->setCurPage($page);
    $collection->load();
    foreach ($collection as $review) { /* process */ }
    $collection->clear();
    $page++;
} while ($collection->getSize() > 0);
```

3. **Use `insertMultiple()` for bulk inserts.** Multiple `insert()` calls create multiple round-trips. `insertMultiple()` batches them into a single query.

4. **Index your EAV filter columns.** When filtering on an EAV attribute (e.g., `rating` in `training_review`), ensure there is an index on that column. EAV attribute filtering without indexes is a common performance killer.

5. **Keep SearchCriteria filter groups small.** Each FilterGroup adds a level of OR logic, which compounds query complexity. If you need complex AND/OR combinations, consider a dedicated repository method with a custom query instead of SearchCriteria.

6. **`db_schema_whitelist.json` is your schema history.** Commit it and treat it as part of your version control. When upgrading across multiple module versions, the whitelist determines what was applied — never manually edit it.

7. **Always wrap multi-entity saves in transactions.** If your repository `save()` method touches multiple tables or dispatches events that trigger side-effects, wrap the operation in a transaction. A partial failure should never leave the database in an inconsistent state.

8. **Use `SearchCriteriaInterface` in controller parameters, not builders.** When exposing APIs, accept `SearchCriteriaInterface` as a controller parameter — Magento automatically populates it from the request query parameters. Using `SearchCriteriaBuilder` inside the controller couples the controller to the filter structure.

9. **Keep EAV attributes lean.** Every `addAttribute()` call adds rows to EAV tables. A product with 100 custom attributes needs 100 rows per store view. Audit attributes periodically — remove unused ones.

10. **`setPageSize()` is your friend.** Always set a `setPageSize()` on collections before loading, even for seemingly small result sets. Code evolves, data grows, and a collection without pagination will eventually cause an OOM in production.

---

*Magento 2 Backend Developer Course — Topic 04*
