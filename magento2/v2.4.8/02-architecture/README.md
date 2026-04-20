# Topic 2: Magento Architecture & Design Principles

**Goal:** Understand Magento's layered architecture, request lifecycle, and extension points so you can design modules that integrate cleanly without fighting the framework.

---

## Topics Covered

- Magento architecture layers: Presentation, Domain, Infrastructure
- Request lifecycle deep dive: EntryPoint → AreaResolver → Router → Controller → Response
- Module structure philosophy: What makes a "good" Magento module
- Dependency Injection container: How Magento wires dependencies
- Plugin system: Around/before/after interceptors
- Event/Observer pattern: Decoupled event-driven architecture
- Service contracts: Interface-first design philosophy

---

## Reference Exercises

- **Exercise 2.1:** Trace a frontend request through the Magento stack (add logging to each layer)
- **Exercise 2.2:** Create a plugin on `Magento\Catalog\Model\Product` to modify getPrice()
- **Exercise 2.3:** Create an observer on `catalog_product_save_after` to log product creation
- **Exercise 2.4:** Verify your plugin runs in correct order using sortOrder in `di.xml`
- **Exercise 2.5:** Inspect generated interceptor classes in `var/generation/`

---

## Completion Criteria

- [ ] Can trace a request from `index.php` through Router → Controller → Response
- [ ] Understands Area concept (`frontend`, `adminhtml`, `graphql`) and when each loads
- [ ] Can explain difference between Plugin and Observer, and when to use each
- [ ] Can create a plugin that modifies behavior without touching original class
- [ ] Can create an observer that responds to dispatched events
- [ ] Understands why service contracts matter for testability and API stability
- [ ] Can explain what happens during `bin/magento setup:di:compile`
- [ ] Module created follows single-responsibility principle (not a "god module")

---

## Topics

---

### Topic 1: Architecture Layers — The Big Picture

Magento follows a **three-layer architecture** (also called "onion architecture" or "hexagonal architecture"):

```
┌─────────────────────────────────────────┐
│           PRESENTATION LAYER            │
│   Controllers, Blocks, Templates,       │
│   Layout XML, ViewModels               │
├─────────────────────────────────────────┤
│             DOMAIN LAYER                │
│   Models, Service Contracts (Interfaces)│
│   Business Logic, Domain Events         │
├─────────────────────────────────────────┤
│          INFRASTRUCTURE LAYER          │
│   ResourceModels, Repositories,         │
│   External Services, Search, Cache      │
└─────────────────────────────────────────┘
```

**Why This Matters:**

| Layer | Should Contain | Should NOT Contain |
|-------|---------------|-------------------|
| Presentation | HTTP handling, layout, block rendering, view data preparation | Business rules, SQL, external API logic |
| Domain | Business logic, validation, service orchestration, domain events | HTTP concepts, SQL, external service URLs |
| Infrastructure | Database CRUD, external API clients, caching implementations, logging | Business logic, presentation concerns |

> **Common Pitfall:** Putting business logic in Controllers (presentation layer). Controllers should be thin — they receive HTTP requests and return responses. All business logic belongs in the Domain layer (Service classes, Models).

> **Pro Tip:** A well-structured Magento module keeps each class in its proper layer. If you see `use Magento\Framework\Http\ZendClient` in a Model, that's a red flag — the Model shouldn't know about HTTP transport.

**Magento Core Examples by Layer:**

| Layer | Magento Core Class |
|-------|-------------------|
| Presentation | `Magento\Catalog\Block\Product\View` |
| Domain | `Magento\Catalog\Api\ProductRepositoryInterface` (interface), `Magento\Catalog\Model\Product` |
| Infrastructure | `Magento\Catalog\Model\ResourceModel\Product` (ResourceModel), `Magento\Elasticsearch\Model\Adapter` |

---

### Topic 2: Request Lifecycle Deep Dive

Understanding the request lifecycle is critical for debugging routing issues, understanding when your code runs, and knowing where to intercept requests.

**Full Request Lifecycle (Magento 2.4+):**

```
HTTP Request
    │
    ▼
pub/index.php (Entry Point)
    │
    ▼
Magento\Framework\App\Bootstrap::run()
    │
    ▼
HttpEntryPoint::launch()  ──────────────────────────┐
    │                                               │
    ▼                                               │
AreaList::getArea()                                 │
    │                                               │
    ▼                                               │
AreaResolver::resolve()  ───────────────────────────┤
    │                                               │
    ▼                                               │
RouterList::match()                                 │
    │ (iterates through routers)                    │
    ├── UrlRewriteRouter                            │
    ├── StandardRouter (matches frontName)          │
    ├── CMS Router (magento cms page)               │
    └── NoRouteHandler (404)                       │
    │                                               │
    ▼                                               │
ControllerFront::dispatch()
    │
    ▼
Controller\AreafrontName\ControllerPath\Action::execute()
    │
    ├── Load controllers + dependencies             │
    ├── Execute business logic                      │
    └── Return Result object                       │
    │
    ▼
ResultInterface::renderResult()
    │
    ▼
HTTP Response
```

**Why Each Step Matters:**

1. **EntryPoint** (`pub/index.php`): Creates the Bootstrap object with `ObjectManager`. Not much happens here — it's the bootstrap initialization.

2. **Bootstrap::run()**: Starts the application. Creates the `ObjectManager` and `AreaList`.

3. **HttpEntryPoint::launch()**: Determines which area to load (`frontend` vs `adminhtml` vs `graphql`). Delegates to `AreaList`.

4. **AreaList::getArea()**: Returns the configured area. Reads from `app/etc/di.xml` and `app/etc/env.php`.

5. **AreaResolver::resolve()**: Runs **after** the area is identified. Loads area-specific configuration (routes, events, plugins) into the `ObjectManager`.

> **Why this matters:** Plugins and observers defined in `adminhtml/di.xml` only load for admin requests. If you're debugging why a plugin isn't running, check if the correct area configuration is loaded.

6. **RouterList::match()**: Iterates through registered routers. First router to return a non-null match wins.

> **Common Pitfall:** Custom router registration. If your route isn't being matched, check that your module's `routes.xml` is in the correct area (`etc/frontend/routes.xml` for frontend).

7. **Controller::execute()**: Your code runs here. Returns a `ResultInterface` object.

8. **ResultInterface::renderResult()**: Takes the Result object and writes to the HTTP response stream.

**The Role of FrontName:**

| URL Segment | Maps To |
|-------------|---------|
| `helloworld` | frontName from `routes.xml` |
| `index` | Controller folder name |
| `index` | Action method name (`execute()`) |

**How Areas Affect Request Processing:**

| Area | Route Location | Authentication |
|------|---------------|----------------|
| `frontend` | `etc/frontend/routes.xml` | Public by default |
| `adminhtml` | `etc/adminhtml/routes.xml` | Requires admin session |
| `graphql` | `etc/graphql/routes.xml` | Token-based |
| `webapi_rest` | `etc/webapi.xml` | OAuth/token |

> **Pro Tip:** Areas are NOT just for routing. Each area has its own `di.xml`, `events.xml`, and layout XML. A plugin defined in `adminhtml/di.xml` won't run for frontend requests.

---

### Topic 3: Dependency Injection Container

Magento's DI container is the backbone of its architecture. Every class is instantiated via the container, which enables plugins, preferences, and testability.

**How DI Works in Magento:**

```php
// When you write:
public function __construct(ResultFactory $resultFactory)
{
    $this->resultFactory = $resultFactory;
}

// Magento's ObjectManager does:
$objectManager->get(ResultFactory::class);
```

> **Why Constructor Injection?** All dependencies are explicit. You can see exactly what a class needs. This makes testing trivial — you inject mocks. It also enables the DI container to manage object lifecycle (singletons, factories).

**The ObjectManager:**

The `ObjectManager` is Magento's DI container implementation. It reads `di.xml` to determine:
- Which class to instantiate for an interface (preferences)
- Which plugins to apply (interceptors)
- Which arguments to inject (constructor arguments)
- Which instances are shared (singleton vs new instance)

> **Common Pitfall:** Direct instantiation with `new`. Never do `$myClass = new MyClass()`. Always inject via constructor. Direct instantiation bypasses DI, meaning plugins don't run and configuration isn't applied.

**di.xml Scope Hierarchy:**

```
Global:     app/etc/di.xml          ← All areas
Area:       app/code/Vendor/Module/etc/[area]/di.xml  ← Area-specific

Loading order: Global → Area (area overrides global)
```

| File | Scope | Use For |
|------|-------|---------|
| `etc/di.xml` | Global | Preferences, virtual types, plugins, factories |
| `etc/adminhtml/di.xml` | Admin only | Admin-specific preferences |
| `etc/frontend/di.xml` | Frontend only | Frontend-specific config |
| `etc/graphql/di.xml` | GraphQL only | GraphQL-specific resolvers |

**Preference (Interface → Implementation):**

```xml
<!-- Make ReviewRepositoryInterface resolve to ReviewRepository -->
<preference for="Training\Review\Api\ReviewRepositoryInterface"
            type="Training\Review\Model\ReviewRepository"/>
```

> **Why preferences matter:** You can replace ANY Magento class with your own by declaring a preference. This is how third-party modules override core behavior.

**Virtual Types:**

Virtual types create named subclasses of existing classes with different constructor arguments — without creating an actual PHP file:

```xml
<!-- Create a "training.review.search.builder" that differs from standard SearchCriteriaBuilder -->
<virtualType name="TrainingReviewSearchBuilder"
             type="Magento\Framework\Api\SearchCriteriaBuilder">
    <arguments>
        <argument name="filterGroupBuilder" xsi:type="object">TrainingReviewFilterGroupBuilder</argument>
    </arguments>
</virtualType>
```

> **Why Virtual Types?** When two services need slightly different configurations of the same class, you create a virtual type. Example: `Magento\Catalog\Model\Layer\Search\FilterPool` vs `Magento\Catalog\Model\Layer\Category\FilterPool`.

**Constructor Property Promotion (Magento 2.4+/PHP 8):**

```php
// Modern Magento style (PHP 8, Magento 2.4+)
public function __construct(
    private readonly ReviewInterfaceFactory $reviewFactory,
    private readonly ResourceModel $resourceModel,
    private readonly LoggerInterface $logger
) {
}
```

> **PSR-12 Compliance:** Magento 2.4+ enforces PSR-12. Constructor property promotion with `readonly` is the modern pattern. Always use `private readonly` for injected dependencies.

**Singleton vs Prototype:**

By default, DI resolves classes as singletons (same instance reused). To force a new instance:

```xml
<type name="Training\Review\Api\ReviewInterface">
    <arguments>
        <argument name="reviewFactory" xsi:type="object">FactoryClassName</argument>
    </arguments>
</type>
```

Or in code:

```php
// Get a new instance (prototype)
$review = $this->reviewFactory->create();

// Get singleton (same instance)
$review = $this->reviewRepository; // injected as singleton
```

---

### Topic 4: Plugin System (Interceptors)

Plugins allow you to modify behavior of ANY Magento class's public methods without touching the original class. This is Magento's primary extension point.

**Why Plugins Over Class Extension:**

| Approach | Problem |
|----------|---------|
| Class inheritance | Tighter coupling, fragile, conflicts with other extensions |
| `__construct` override | Doesn't work reliably in Magento DI |
| Direct class editing | Lost on upgrade, violates integrity |
| **Plugins** | Decoupled, ordered, works with DI, upgrade-safe |

**Plugin Types:**

| Type | When It Runs | Common Use |
|------|-------------|------------|
| `before` | Before original method | Modify arguments, validate input |
| `after` | After original method | Modify return value, post-processing |
| `around` | Instead of original | Complete replacement, conditional execution |

**Plugin Declaration in `di.xml`:**

```xml
<type name="Magento\Catalog\Model\Product">
    <plugin name="TrainingCatalogProductPlugin"
            type="Training\Catalog\Plugin\ProductPlugin"/>
</type>
```

**Plugin Implementation:**

```php
<?php
declare(strict_types=1);

namespace Training\Catalog\Plugin;

use Magento\Catalog\Model\Product as Subject;
use Magento\Framework\DataObject;

class ProductPlugin
{
    /**
     * Before getPrice — modify input arguments
     */
    public function beforeGetPrice(Subject $subject, float $price): array
    {
        // Add $price adjustment
        return [$price * 1.1]; // Override the price argument
    }

    /**
     * After getName — modify return value
     */
    public function afterGetName(Subject $subject, string $result): string
    {
        // Add suffix to product name
        return $result . ' (Reviewed)';
    }

    /**
     * Around getDescription — wrap original execution
     */
    public function aroundGetDescription(
        Subject $subject,
        callable $proceed,
        ...$args
    ): string {
        // Log before
        $this->logger->info('Getting product description', ['product_id' => $subject->getId()]);

        // Call original (or skip it entirely)
        $description = $proceed(...$args);

        // Log after
        $this->logger->info('Product description retrieved', ['length' => strlen($description)]);

        return $description;
    }
}
```

**Plugin Sort Order:**

When multiple plugins target the same class, use `sortOrder` in `di.xml`:

```xml
<type name="Magento\Catalog\Model\Product">
    <plugin name="pluginA"
            type="Training\PluginA"
            sortOrder="10"/>
    <plugin name="pluginB"
            type="Training\PluginB"
            sortOrder="20"/>
</type>
```

Plugins execute in ascending `sortOrder`. Plugin A runs before Plugin B.

> **Common Pitfall:** `around` plugins that don't call `$proceed()`. This prevents subsequent plugins AND the original method from running. Only skip `$proceed()` when you intentionally want to replace the original behavior entirely.

> **Pro Tip:** Keep plugins small and focused. A plugin should do one thing. If you find yourself needing to do multiple things, use multiple plugins in sequence.

**Plugin Limitations (What Plugins CAN'T Do):**

| Limitation | Workaround |
|------------|------------|
| Cannot modify `final` methods/classes | Use observer on event |
| Cannot modify `final` classes (e.g., `ObjectManager`) | Cannot extend |
| Cannot modify `protected`/`private` methods | Cannot extend |
| Cannot modify constructor arguments | Use `di.xml` arguments |
| Cannot modify static methods | Cannot extend |

> **Why `final` exists?** Magento marks certain classes as `final` because their internal behavior is tightly coupled. Extending them would break invariants. Use observers for final classes.

---

### Topic 5: Event/Observer System

Events provide a decoupled way to respond to Magento actions. When something happens in Magento (product saved, order placed), it dispatches an event. Any observer listening receives the event payload.

**Why Events Over Direct Coupling:**

| Direct Call | Event-Based |
|-------------|-------------|
| Module A calls Module B directly | Module A dispatches event |
| Module A knows about Module B | Module B observes event |
| If Module B removed, Module A breaks | Module A doesn't care about Module B |
| Hard to test | Easy to test (just don't dispatch in tests) |

**Event Dispatching:**

```php
<?php
// In any class:
use Magento\Framework\Event\ManagerInterface;

public function saveProduct(Product $product): void
{
    // Before save logic
    $this->eventManager->dispatch('catalog_product_save_before', ['product' => $product]);

    // Save logic
    $this->resourceModel->save($product);

    // After save logic
    $this->eventManager->dispatch('catalog_product_save_after', ['product' => $product]);
}
```

> **Pro Tip:** Magento dispatches hundreds of events. Check `vendor/magento/module-catalog/etc/events.xml` for product events. Don't reinvent — if an event exists for your use case, use it.

**Event Declaration in `events.xml`:**

```xml
<!-- etc/events.xml (global) or etc/adminhtml/events.xml (admin only) -->
<event name="catalog_product_save_after">
    <observer name="training_review_observer"
              instance="Training\Review\Observer\ProductSavedObserver"/>
</event>
```

**Observer Implementation:**

```php
<?php
declare(strict_types=1);

namespace Training\Review\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Psr\Log\LoggerInterface;

class ProductSavedObserver implements ObserverInterface
{
    private LoggerInterface $logger;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function execute(Observer $observer): void
    {
        /** @var \Magento\Catalog\Model\Product $product */
        $product = $observer->getEvent()->getData('product');

        $this->logger->info('Product saved', [
            'product_id' => $product->getId(),
            'sku' => $product->getSku()
        ]);
    }
}
```

**Observer vs Plugin — When to Use Which:**

| Scenario | Use Plugin | Use Observer |
|----------|-----------|--------------|
| Modifying method arguments/return | ✅ Yes | ❌ No |
| Responding to an action (side-effects) | ❌ No | ✅ Yes |
| Adding validation | ✅ Yes | ❌ No |
| Triggering external system | ❌ No | ✅ Yes |
| The class is `final` | ❌ No | ✅ Yes |
| Multiple observers same event | ❌ No | ✅ Yes |

> **Common Pitfall:** Using observers for business logic that modifies data. Observers run after the main action, and if one observer throws an exception, the original action has already committed. Use plugins for data modification, observers for reactions (logging, external notifications).

**Event Object (Payload):**

The `Observer::getEvent()` returns an event object with data:

```php
$event = $observer->getEvent();
$product = $event->getData('product');        // Same as getProduct()
$items = $event->getData('items');            // Array of items
```

> **Pro Tip:** Check Magento core `events.xml` files to see what data is passed with each event. Example: `vendor/magento/module-sales/etc/events.xml` for order events.

---

### Topic 6: Service Contracts

Service contracts are PHP interfaces that define public APIs for Magento modules. They provide stable, versioned APIs that third-party code can depend on.

**Why Service Contracts Matter:**

| Without Service Contracts | With Service Contracts |
|--------------------------|----------------------|
| Depend on concrete Model | Depend on Interface |
| Model changes break consumers | Only interface changes break |
| Hard to mock in tests | Easy to mock |
| Internal structure leaked | Internal structure hidden |
| Can't have multiple implementations | Can swap implementations via DI |

**Service Contract Layers:**

```
┌─────────────────────────────────────────┐
│          Api/Data/ (Data Interfaces)    │
│   Define data structure (getters only)  │
├─────────────────────────────────────────┤
│          Api/ (Repository Interfaces)    │
│   Define CRUD operations                 │
└─────────────────────────────────────────┘
```

**Data Interface Example:**

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
    public function getRating(): int;
    public function setRating(int $rating): self;
    public function getReviewText(): string;
    public function setReviewText(string $text): self;
    // ...
}
```

> **Why getters/setters?** The data interface is a Plain Old PHP Object (POPO). No business logic, no dependencies. It's a data container that service contracts return.

**Repository Interface Example:**

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
    public function getList(SearchCriteriaInterface $searchCriteria): SearchResultsInterface;
}
```

> **Why `SearchCriteriaInterface`?** This decouples filtering/pagination from the repository. Consumers build their own SearchCriteria without knowing how the repository implements filtering.

**Wiring Service Contracts:**

```xml
<!-- etc/di.xml -->
<preference for="Training\Review\Api\ReviewRepositoryInterface"
            type="Training\Review\Model\ReviewRepository"/>
<preference for="Training\Review\Api\Data\ReviewInterface"
            type="Training\Review\Model\Review"/>
```

**Using Service Contracts:**

```php
<?php
// In a controller or service class:
public function __construct(
    private readonly ReviewRepositoryInterface $reviewRepository
) {
}

public function getReview(int $reviewId): ReviewInterface
{
    return $this->reviewRepository->getById($reviewId);
}
```

> **Pro Tip:** NEVER inject the concrete Model class in your services. Always inject the Interface. This allows you to swap implementations (e.g., for testing with mocks) without changing consumers.

**The `SearchResultsInterface`:**

```php
<?php
// Getting a list with pagination:
$searchCriteria = $this->searchCriteriaBuilder
    ->addFilter('product_id', 1)
    ->setPageSize(10)
    ->setCurrentPage(1)
    ->create();

$searchResults = $this->reviewRepository->getList($searchCriteria);

$reviews = $searchResults->getItems();        // Array of ReviewInterface
$totalCount = $searchResults->getTotalCount(); // Total matching records
$searchCriteria = $searchResults->getSearchCriteria(); // Original criteria
```

---

### Topic 7: What Makes a "Good" Magento Module

A well-structured Magento module follows principles that make it maintainable, testable, and compatible with other modules.

**Single Responsibility Principle:**

| Bad Module | Good Module |
|------------|-------------|
| `Training_Everything` | `Training_Review`, `Training_Shipping`, `Training_Api` |
| Does product reviews AND shipping AND API | One focused feature per module |
| Hard to disable, conflicts everywhere | Easy to disable, isolated |

> **Why?** When `Training_Everything` breaks, what do you disable? Nothing — it's all or nothing. With separate modules, you can disable `Training_Shipping` without affecting `Training_Review`.

**Module File Structure:**

```
app/code/Training/Review/
├── etc/
│   ├── module.xml                 ← Module declaration
│   ├── di.xml                    ← Global DI config
│   ├── events.xml                ← Global events
│   ├── frontend/
│   │   ├── routes.xml            ← Frontend routes
│   │   ├── di.xml                ← Frontend DI
│   │   └── events.xml            ← Frontend events
│   ├── adminhtml/
│   │   ├── routes.xml            ← Admin routes
│   │   └── menu.xml              ← Admin menu
│   └── webapi.xml               ← REST API routes
├── Api/
│   ├── Data/
│   │   └── ReviewInterface.php  ← Data interface
│   └── ReviewRepositoryInterface.php
├── Model/
│   ├── Review.php                ← Entity model
│   ├── ReviewRepository.php     ← Repository implementation
│   └── ResourceModel/
│       ├── Review.php            ← ResourceModel
│       └── Review/
│           └── Collection.php    ← Collection
├── Observer/
│   └── ProductSavedObserver.php  ← Event observer
├── Plugin/
│   └── ProductPlugin.php         ← Plugin
├── Setup/
│   ├── Patch/
│   │   ├── Data/                ← Data patches
│   │   └── Schema/              ← Schema patches
│   └── db_schema.xml            ← Declarative schema
├── Controller/
│   └── Frontend/
│       └── Review/
│           └── Index/
│               └── Index.php    ← Frontend controller
└── registration.php
```

**What Goes Where:**

| Directory | What It Contains |
|----------|-----------------|
| `Api/Data/` | Data transfer object interfaces (getters/setters) |
| `Api/` | Repository/service interfaces (CRUD operations) |
| `Model/` | Domain logic, service implementations |
| `Model/ResourceModel/` | Database operations, table mapping |
| `Setup/` | Schema/data patches |
| `Observer/` | Event observers |
| `Plugin/` | Interceptors |
| `Controller/` | HTTP request handlers |

**Module Dependencies:**

Declare dependencies in `module.xml`:

```xml
<module name="Training_Review" setup_version="1.0.0">
    <sequence>
        <module name="Magento_Catalog"/>      <!-- We depend on catalog -->
    </sequence>
</module>
```

> **Why sequences matter?** `sequence` tells Magento to load `Training_Review` AFTER `Magento_Catalog`. This ensures our observer on `catalog_product_save_after` has the product model loaded before we try to use it.

**No Cross-Module Business Logic:**

A module should not call another module's Models directly. Instead:

1. Use Service Contracts (interfaces)
2. If no interface exists, dispatch an event
3. If you must call directly, note it as a documented dependency

---

## Reading List

- [Magento Architecture](https://developer.adobe.com/commerce/php/architecture/) — Three-layer architecture overview
- [Dependency Injection](https://developer.adobe.com/commerce/php/development/components/dependency-injection/) — DI container deep dive
- [Plugins](https://developer.adobe.com/commerce/php/development/components/plugins/) — Interceptor system
- [Events and Observers](https://developer.adobe.com/commerce/php/development/components/events-and-observers/) — Event-driven patterns
- [Service Contracts](https://developer.adobe.com/commerce/php/development/components/service-contracts/) — Interface-first design
- [Request Flow](https://developer.adobe.com/commerce/php/development/build/request-flow/) — How HTTP requests are processed

---

## Edge Cases & Troubleshooting

| Issue | Symptom | Solution |
|-------|---------|----------|
| Plugin not running | Modified method has no effect | Check area (frontend vs adminhtml), check di.xml syntax |
| Observer not firing | Event handler not called | Verify events.xml in correct area, check observer class name |
| Circular dependency | DI compile fails | Refactor to break cycle, use `ObjectManager` as last resort |
| Area not loading | 404 on admin route | Verify route in correct area's `routes.xml` |
| Preference not taking | Wrong class being used | Clear `var/generation/`, run `setup:di:compile` |
| Multiple plugins conflict | Unexpected behavior | Use `sortOrder` in di.xml |

---

## Common Mistakes to Avoid

1. ❌ Putting business logic in Controllers → Move to Domain/Service layer
2. ❌ Direct instantiation (`new MyClass()`) → Use constructor injection
3. ❌ Plugin on `final` method → Use observer instead
4. ❌ `around` plugin not calling `$proceed()` → Original method never runs
5. ❌ Observer modifying entity data → Use plugin for data modification
6. ❌ `di.xml` in wrong area → Admin plugin not running on frontend
7. ❌ Forgetting to run `setup:di:compile` → Generated classes stale
8. ❌ Creating "god module" → Split into focused, single-responsibility modules
9. ❌ Cross-module direct Model coupling → Use Service Contracts (interfaces)
10. ❌ Missing `sequence` declaration → Module loads before its dependency

---

*Magento 2 Backend Developer Course — Topic 02*
