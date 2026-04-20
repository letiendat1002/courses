# Topic 5: Customization — Plugins, Observers & Dependency Injection

**Goal:** Customize Magento behavior without touching core code — using plugins to intercept method calls and observers to react to events.

---

## Topics Covered

- After plugins — modifying return values after a method runs
- Before plugins — validating or modifying input before a method runs
- Around plugins — wrapping method logic, conditionally calling the original
- Event dispatch and observer registration
- Custom event dispatch
- Plugin sort order and interaction between multiple plugins
- di.xml advanced: preferences, virtual types, constructor injection configuration

---

## Reference Exercises

- **Exercise 4.1:** Create an after plugin on `ProductRepositoryInterface::save` that logs the SKU
- **Exercise 4.2:** Create a before plugin on `save` that rejects negative price or empty SKU
- **Exercise 4.3:** Create an around plugin on `getById` that logs execution time
- **Exercise 4.4:** Dispatch a custom event and create an observer to handle it
- **Exercise 4.5:** Configure a virtual type and non-shared factory in di.xml

---

## Completion Criteria

- [ ] After plugin modifies `ProductRepositoryInterface::save` return value
- [ ] Before plugin validates SKU (not empty) and price (not negative)
- [ ] Around plugin wraps `getById` with timing logic
- [ ] Custom event dispatched from controller/service
- [ ] Observer responds to dispatched event
- [ ] Plugin sort order configured for multiple plugins
- [ ] Virtual type or preference configured in di.xml

---

## Topics

---

### Topic 1: After Plugins

**What after plugins do:** Run after the original method. Can modify the return value before it's returned to the caller. The most commonly used plugin type.

**Why After Plugins Exist:**

After plugins solve the "I need to react to what just happened" problem without modifying the original code. They allow you to:
- Enrich return data (add computed fields, modify status)
- Log or audit operations
- Trigger side effects (notifications, events)
- Validate or transform output

The key insight: **the original method already completed successfully** — you just get to see the result before anyone else does.

> **Pro Tip from Production:** After plugins are the safest plugin type because if your plugin throws an exception, the original method has already completed its work. For example, logging plugins should almost always be `after` plugins — you never want a failed logging attempt to prevent a save operation.

**Why Not Modify the Original?**

Instead of editing `ProductRepositoryInterface::save()` directly (which would break on upgrades), you intercept it:
- Original code stays untouched — upgrade-safe
- Multiple modules can intercept the same method (via sortOrder)
- Plugins can be disabled via configuration without code changes
- Follows Open/Closed Principle: open for extension, closed for modification

**Registration in `di.xml`:**

```xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/di.xsd">
    <type name="Magento\Catalog\Api\ProductRepositoryInterface">
        <plugin name="training_review_plugin"
                type="Training\Review\Plugin\ProductRepositoryPlugin"
                sortOrder="10"/>
    </type>
</config>
```

**After Plugin Pattern:**

```php
<?php
// Plugin/ProductRepositoryPlugin.php
namespace Training\Review\Plugin;

use Magento\Catalog\Api\Data\ProductInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;

class ProductRepositoryPlugin
{
    private $logger;

    public function __construct(\Psr\Log\LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function afterSave(
        ProductRepositoryInterface $subject,
        ProductInterface $result
    ): ProductInterface {
        $sku = $result->getSku();
        $this->logger->info("Product saved via plugin: {$sku}");
        return $result;
    }

    public function afterGetById(
        ProductRepositoryInterface $subject,
        ProductInterface $result,
        int $productId
    ): ProductInterface {
        if ($result && !$result->getDescription()) {
            $result->setDescription('Default description set by plugin');
        }
        return $result;
    }
}
```

**Key rules:**
- Return the `$result` (or compatible type) — NOT returning = null = broken
- Method name must be exactly the intercepted method name
- The method receives the original method's arguments plus the result

**PSR-12 / Magento Coding Standards Notes:**
```php
// ✅ CORRECT — After plugin with proper return type hint
public function afterSave(
    ProductRepositoryInterface $subject,
    ProductInterface $result
): ProductInterface {
    return $result;
}

// ❌ WRONG — Missing return type, returns null implicitly
public function afterSave($subject, $result) {
    // modification...
    // forgot to return!
}
```

**Real-World Magento Core Pattern:**

Magento's `Magento\Catalog\Model\Product\Link\Plugin\ProductRepository` uses after plugins to automatically set parent product relationships after a product is saved. This runs silently in the background — your module never knows it happened.

```php
// Simplified from Magento_Catalog
public function afterSave(
    \Magento\Catalog\Api\ProductRepositoryInterface $subject,
    \Magento\Catalog\Api\Data\ProductInterface $result
): \Magento\Catalog\Api\Data\ProductInterface {
    // Parent already saved — now handle product links
    $this->productLinkRepository->save($result->getProductLinks());
    return $result;
}
```

**Common Pitfall — After Plugin Modifying the Result:**

```php
// ❌ DANGEROUS — Modifying entity in after plugin
public function afterSave($subject, $result): ProductInterface {
    $result->setPrice(0); // Never do this!
    return $result;
}

// ✅ CORRECT — Create a new value object or use events for side effects
public function afterSave($subject, $result): ProductInterface {
    $this->eventManager->dispatch('training_review_product_saved', [
        'product' => $result
    ]);
    return $result;
}
```

---

### Topic 2: Before Plugins

**What before plugins do:** Run before the original method. Can validate or modify input arguments before the original method runs. Commonly used for input validation.

**Why Before Plugins Exist:**

Before plugins solve the "I need to validate or transform input before it reaches the core logic" problem. They act as guards or translators:

- **Validation:** Reject invalid input early (fail fast)
- **Normalization:** Convert input to the format the original expects
- **Enrichment:** Add default values or computed fields
- **Early Exit:** Skip expensive operations for certain inputs

> **Pro Tip from Production:** Place validation plugins with low `sortOrder` (e.g., 10) so they run first. If validation fails, higher-order plugins and the original method never run — saving expensive operations.

**Why Not Just Validate in the Original Method?**

You can't modify core code! Before plugins let you add validation without touching Magento core. If your validation is business-critical, consider whether an event/observer pattern (Topic 4) might be more appropriate for truly decoupled reactions.

**Before Plugin Pattern:**

```php
<?php
// Plugin/ProductValidationPlugin.php
namespace Training\Review\Plugin;

use Magento\Catalog\Api\Data\ProductInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Framework\Exception\LocalizedException;

class ProductValidationPlugin
{
    public function beforeSave(
        ProductRepositoryInterface $subject,
        ProductInterface $product,
        $saveOptions = []
    ): array {
        $sku = $product->getSku();

        if (empty($sku)) {
            throw new LocalizedException(__('SKU cannot be empty'));
        }

        if (strlen($sku) > 64) {
            throw new LocalizedException(__('SKU cannot exceed 64 characters'));
        }

        $price = $product->getPrice();
        if ($price !== null && $price < 0) {
            throw new LocalizedException(__('Price cannot be negative'));
        }

        return [$product, $saveOptions];
    }
}
```

**Key rules:**
- Method name is `before` + intercepted method name (e.g., `beforeSave`)
- Must return an array of modified arguments matching the original method signature
- Returning nothing = original arguments unchanged
- Used for validation, normalization, early-exit checks

**PSR-12 / Magento Coding Standards Notes:**

```php
// ✅ CORRECT — Proper type hints and return array
public function beforeSave(
    ProductRepositoryInterface $subject,
    ProductInterface $product,
    $saveOptions = []
): array {
    // Validation logic
    return [$product, $saveOptions];
}

// ❌ WRONG — Returning null instead of array breaks the method call
public function beforeSave($subject, $product, $saveOptions = []) {
    if (!$product->getSku()) {
        throw new LocalizedException(__('SKU required'));
    }
    // Forgot to return!
}

// ✅ CORRECT — When not modifying args, still return empty array
public function beforeSave($subject, $product, $saveOptions = []): array
{
    $this->logger->debug('Save called'); // Just log, don't modify
    return [$product, $saveOptions];
}
```

**Real-World Magento Core Pattern:**

`Magento\Catalog\Model\Product\Option\Plugin\SaveOptions` uses a before plugin to normalize save options across different product contexts:

```php
// Simplified from Magento_Catalog
public function beforeSave(
    \Magento\Catalog\Api\ProductRepositoryInterface $subject,
    \Magento\Catalog\Api\Data\ProductInterface $product,
    $saveOptions = []
): array {
    if (!isset($saveOptions['操作'])) {
        $saveOptions['操作'] = true;
    }
    return [$product, $saveOptions];
}
```

**Common Pitfall — Modifying Arguments Incorrectly:**

```php
// ❌ WRONG — Modifying the passed argument object directly
public function beforeSave($subject, $product): array {
    $product->setPrice(100); // Don't modify input!
    return [];
}

// ✅ CORRECT — If modifying, return modified arguments in array
public function beforeSave($subject, $product): array {
    if ($product->getPrice() === null) {
        $product->setPrice(0); // This IS okay — you're setting a default
    }
    return [$product];
}
```

---

### Topic 3: Around Plugins

**What around plugins do:** Completely wrap the original method. You control when (and if) the original method runs. Use sparingly — prefer before/after when possible.

**Why Around Plugins Exist — and Why They're Dangerous:**

Around plugins give you total control over the method invocation. You decide:
- Whether to run the original at all
- How many times to run it
- What to do before and after

> **⚠️ CRITICAL WARNING:** Around plugins are the most powerful but also the most dangerous plugin type. A single bug can bring down your entire application. Always prefer `before` and `after` plugins when possible.

**The `$proceed()` Callback — Your Most Important Responsibility:**

The `$proceed` callable is a reference to the next plugin in the chain, ultimately calling the original method. **You MUST call it exactly once** (unless intentionally skipping):

```php
// ✅ CORRECT — Around plugin that properly calls proceed()
public function aroundGetById(
    ProductRepositoryInterface $subject,
    callable $proceed,
    int $productId
): ProductInterface {
    $this->logger->info("Before getById($productId)");
    $result = $proceed($productId);  // ✅ Calls the original
    $this->logger->info("After getById($productId)");
    return $result;
}
```

**Common Fatal Mistakes with `$proceed()`:**

```php
// ❌ FATAL — Forgot to call $proceed() — original method NEVER runs
public function aroundGetById($subject, callable $proceed, int $productId) {
    // Cache hit?
    if ($this->cache->get($productId)) {
        return $this->cache->get($productId);
    }
    // ❌ BUG: Never called $proceed()! The actual product is never loaded.
    return $this->cache->get($productId);
}

// ✅ CORRECT — Cache pattern with proper proceed() call
public function aroundGetById($subject, callable $proceed, int $productId) {
    if ($this->cache->get($productId)) {
        return $this->cache->get($productId);
    }
    $result = $proceed($productId);  // ✅ Now call original
    $this->cache->set($productId, $result);
    return $result;
}
```

**Around Plugin Execution Order — The Full Picture:**

When multiple plugins exist on the same method, the execution follows this pattern:

```
Plugin A (sortOrder=10), Plugin B (sortOrder=20), Plugin C (sortOrder=30)

Execution flow:
1. All beforePlugins() run in ASCENDING sortOrder
   → A::beforeGetById() → B::beforeGetById() → C::beforeGetById()

2. Original method runs once

3. All afterPlugins() run in DESCENDING sortOrder
   → C::afterGetById() → B::afterGetById() → A::afterGetById()
```

**Around Plugins and Sort Order:**

Around plugins combine before + after in a single method. The sortOrder still applies:
- Plugin with `sortOrder=10` wraps the one with `sortOrder=20`
- The `sortOrder=10`'s `around` runs first (before), then `sortOrder=20`'s `around` runs
- After plugins run in reverse (sortOrder=30's after runs before sortOrder=10's after)

```xml
<type name="Magento\Catalog\Api\ProductRepositoryInterface">
    <!-- Runs FIRST in before phase, LAST in after phase -->
    <plugin name="logging"       type="LoggingPlugin"       sortOrder="10"/>
    <!-- Runs SECOND in before phase, SECOND-TO-LAST in after phase -->
    <plugin name="validation"    type="ValidationPlugin"    sortOrder="20"/>
    <!-- Runs LAST in before phase, FIRST in after phase -->
    <plugin name="cache"         type="CachePlugin"         sortOrder="30"/>
</type>

Execution:
before:  logging → validation → cache → ORIGINAL
after:   cache → validation → logging
```

> **Pro Tip:** Around plugins with lower sortOrder wrap those with higher sortOrder. If you need to completely replace behavior, use `sortOrder=1` and don't call `$proceed()`.

**When to Use Around vs Before/After:**

| Situation | Plugin Type | Reason |
|-----------|-------------|--------|
| Modify return value | `after` | Safe — original already ran |
| Validate/change input | `before` | Fail fast, original never runs if invalid |
| Conditionally skip method | `around` | Don't call `$proceed()` |
| Add logic before AND modify return | `around` | Combine both in one method |
| Implement caching layer | `around` | Check cache before/after |
| Implement retry logic | `around` | Call `$proceed()` multiple times |
| Wrap long operations with timing | `around` | Time before and after |

**Real-World Magento Core Caching Pattern:**

Magento's `Magento\Framework\App\Action\Plugin\RemoveMessage` uses an around plugin to prevent duplicate form key messages:

```php
// Simplified from Magento_Framework
public function aroundExecute(
    Action $subject,
    callable $proceed
) {
    try {
        return $proceed();
    } finally {
        $this->messageManager->getMessages(true); // Clear messages
    }
}
```

**Around Plugin Pattern:**

```php
<?php
// Plugin/ProductTimingPlugin.php
namespace Training\Review\Plugin;

use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Catalog\Api\Data\ProductInterface;

class ProductTimingPlugin
{
    private $logger;

    public function __construct(\Psr\Log\LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function aroundGetById(
        ProductRepositoryInterface $subject,
        callable $proceed,
        int $productId
    ): ProductInterface {
        $start = microtime(true);

        $product = $proceed($productId);  // Call original method

        $elapsed = round((microtime(true) - $start) * 1000, 2);
        $this->logger->info("getById took {$elapsed}ms for product {$productId}");

        return $product;
    }

    public function aroundSave(
        ProductRepositoryInterface $subject,
        callable $proceed,
        ProductInterface $product,
        $saveOptions = []
    ): ProductInterface {
        // Conditionally skip — for special flagged products
        if ($product->getData('skip_save')) {
            $this->logger->info('Skipping save for flagged product');
            return $product;
        }

        return $proceed($product, $saveOptions);
    }
}
```

**Key rules:**
- **You MUST call `$proceed()`** — otherwise the original method never runs and Magento breaks
- Only call `$proceed()` when you want the original to run
- Not calling `$proceed()` = you completely replaced the method's behavior

**⚠️ Around Plugin Pitfalls — Production Incidents:**

| Pitfall | Symptom | Prevention |
|---------|---------|------------|
| Forgot `$proceed()` | Request hangs, timeout, 500 error | Always call `$proceed()` unless intentionally skipping |
| Calling `$proceed()` multiple times | Duplicate operations (e.g., double charge) | Only call once unless implementing retry |
| Exception before `$proceed()` | Original method never runs | Wrap in try-catch if you must catch |
| Modifying `$subject` | Unpredictable side effects | Don't modify the subject, only `$proceed()` result |

**PSR-12 Notes for Around Plugins:**

```php
// ✅ CORRECT — Proper type hints, return type, and proceed() call
public function aroundSave(
    ProductRepositoryInterface $subject,
    callable $proceed,
    ProductInterface $product,
    $saveOptions = []
): ProductInterface {
    $start = microtime(true);
    try {
        $result = $proceed($product, $saveOptions);
        $this->logger->info('Save completed', [
            'sku' => $product->getSku(),
            'duration_ms' => (microtime(true) - $start) * 1000
        ]);
        return $result;
    } catch (\Throwable $e) {
        $this->logger->error('Save failed', ['sku' => $product->getSku()]);
        throw $e; // Re-throw to preserve original behavior
    }
}

// ❌ WRONG — Caught exception and didn't re-throw
public function aroundSave($subject, callable $proceed, $product) {
    try {
        return $proceed($product);
    } catch (\Exception $e) {
        return null; // ❌ Swallowed exception — caller doesn't know it failed!
    }
}
```

**Why Not Just Use Events? When to Choose Plugins vs Events:**

| Criteria | Plugins | Events |
|----------|---------|--------|
| Coupling | Tighter — knows exact method | Looser — just event name |
| Modification | Can modify input/output | Cannot modify, only react |
| Performance | Faster — no event dispatch | Slower — dispatches to all observers |
| Ordering | SortOrder controls execution | Event observer order |
| Scope | Class-level interception | Global event bus |

> **Pro Tip:** Use plugins when you need to intercept a specific method's behavior. Use events when you want to react to something happened without caring how it happened. Events are better for truly decoupled modules.

**When to use around vs before/after:**

| Situation | Plugin Type |
|-----------|-------------|
| Modify return value | `after` |
| Validate/change input | `before` |
| Conditionally skip method | `around` (don't call `$proceed()`) |
| Add logic before AND modify return | `around` |
| Caching layer | `around` |

---

### Topic 4: Observers & Events

**What observers do:** React to dispatched events — completely decoupled from the code that dispatches the event. Use when you want to react to something that happened, not intercept how it happened.

**Why Events & Observers Exist — The Observer Pattern:**

Events solve the "I want to通知 others when something happens without knowing who cares" problem:

- **Decoupled:** The dispatcher doesn't know or care who's listening
- **Extensible:** Add new observers without touching the dispatcher
- **Multiple Subscribers:** Many observers can react to the same event
- **Centralized:** All reactions to an event in one place

> **Pro Tip from Production:** Events are for reactions (something happened → do something). Plugins are for interception (modify how it happens). Don't use events when you need to modify behavior — use plugins.

**Event Dispatching — What Really Happens:**

When you call `$this->eventManager->dispatch('event_name', $data)`:

1. Magento looks up `event_name` in all `events.xml` files
2. Builds a list of observers, sorted by `sortOrder`
3. For each observer, instantiates the class and calls `execute()`
4. All observers run, even if one throws an exception (unless configured otherwise)

**Why Events Are Async-Unfriendly:**

Events fire **synchronously** during the request. If your observer makes an HTTP call to an external system, it blocks the request. For external integrations, use webhooks (Topic 7) or message queues (Topic 8).

**Event Naming Convention — Magento's Best Practices:**

Magento has specific naming patterns for events. Following them ensures your events integrate properly with Magento's ecosystem:

| Event Pattern | Example | When It Fires |
|--------------|---------|---------------|
| `{entity}_{action}_before` | `catalog_product_save_before` | Before save |
| `{entity}_{action}_after` | `catalog_product_save_after` | After save |
| `{area}_{controller}_{action}` | `controller_action_predispatch` | Before any action |
| `sales_order_{action}` | `sales_order_place_after` | After order action |

**Custom Event Naming:**

```php
// ✅ CORRECT — Use vendor prefix, descriptive action
$this->eventManager->dispatch('training_review_submitted', [...]);
$this->eventManager->dispatch('training_review_approved', [...]);

// ❌ WRONG — Generic names can conflict with other modules
$this->eventManager->dispatch('review_submitted', [...]);
```

**Dispatching a Custom Event:**

```php
<?php
// In any service class
use Magento\Framework\Event\ManagerInterface;

class ReviewManagement
{
    protected $eventManager;

    public function __construct(ManagerInterface $eventManager)
    {
        $this->eventManager = $eventManager;
    }

    public function submitReview(array $reviewData): void
    {
        // ... save review ...

        $this->eventManager->dispatch('training_review_submitted', [
            'review' => $reviewData,
            'result' => $savedReview
        ]);
    }
}
```

**Registering an Observer — `etc/events.xml`:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="training_review_submitted">
        <observer name="training_review_handler"
                  instance="Training\Review\Observer\ProcessReviewObserver"
                  sortOrder="10"/>
    </event>
</config>
```

**Observer Implementation:**

```php
<?php
// Observer/ProcessReviewObserver.php
namespace Training\Review\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;

class ProcessReviewObserver implements ObserverInterface
{
    private $logger;

    public function __construct(\Psr\Log\LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function execute(Observer $observer): void
    {
        $review = $observer->getData('review');
        $this->logger->info('Processing submitted review', $review);
        // Send notification, update stats, trigger workflow...
    }
}
```

**Key rule:** Get event data via `$observer->getData('key')`.

**Observer Best Practices:**

```php
// ✅ CORRECT — Check if data exists before accessing
public function execute(Observer $observer): void
{
    $review = $observer->getData('review');
    if (!$review) {
        return; // Guard against missing data
    }
    // Process review...
}

// ❌ WRONG — Assumes data always exists
public function execute(Observer $observer): void
{
    $review = $observer->getData('review'); // Could be null!
    $this->process($review->getId()); // NullPointerException!
}
```

**PSR-12 Notes for Observers:**

```php
// ✅ CORRECT — Proper interface implementation, type hints
use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;

class ProcessReviewObserver implements ObserverInterface
{
    public function execute(Observer $observer): void
    {
        // ...
    }
}

// ❌ WRONG — No return type, missing interface
class ProcessReviewObserver
{
    public function execute($observer)
    {
        // ...
    }
}
```

**Real-World Magento Core Observer Pattern:**

Magento's `Magento/Catalog/Observer` directory contains many examples. The `CatalogProductSaveAfterObserver` demonstrates proper observer implementation:

```php
// Simplified from Magento_Catalog
class CatalogProductSaveAfterObserver implements ObserverInterface
{
    private $productFlatIndexer;
    private $productPriceIndexer;

    public function execute(Observer $observer): void
    {
        $product = $observer->getData('product');
        // Trigger related indexers asynchronously
        $this->productFlatIndexer->reindexRow($product->getId());
        $this->productPriceIndexer->reindexRow($product->getId());
    }
}
```

**Magento Core Events:**

Many core events exist — key ones to know:
- `controller_action_predispatch` — Before any controller action
- `sales_order_place_after` — After order placed
- `catalog_product_save_after` — After product saved
- `checkout_submit_all_after` — After checkout complete

**Event Dispatching Best Practices:**

| Practice | Why It Matters |
|----------|----------------|
| Always check event name in `events.xml` matches exactly | Typos silently break observers |
| Don't dispatch events in constructors | Object not fully constructed |
| Use meaningful event names with vendor prefix | Avoids conflicts |
| Pass only necessary data in event payload | Reduces memory, maintains encapsulation |
| Don't modify event data — observers shouldn't | Breaks other observers expecting original data |

**Common Observer Pitfalls:**

| Pitfall | Symptom | Solution |
|---------|---------|----------|
| Typo in event name | Observer never fires | Double-check `events.xml` matches dispatch |
| Missing `ObserverInterface` | Observer silently skipped | Must implement `execute(Observer $observer)` |
| Throwing exception in observer | Other observers skipped, request may break | Catch exceptions, log errors, don't re-throw |
| Long-running task in observer | HTTP request timeout | Use queue/async for external calls |

---

### Topic 5: Plugin Sort Order & di.xml Advanced
Many core events exist — key ones to know:
- `controller_action_predispatch` — Before any controller action
- `sales_order_place_after` — After order placed
- `catalog_product_save_after` — After product saved
- `checkout_submit_all_after` — After checkout complete

---

### Topic 5: Plugin Sort Order & di.xml Advanced

**Plugin Sort Order — The Complete Execution Model:**

Lower `sortOrder` = earlier execution. But the full picture is more nuanced:

**Phase 1: BEFORE Phase (ascending sortOrder)**
```
Plugin A (sortOrder=10).before() → Plugin B (sortOrder=20).before() → Plugin C (sortOrder=30).before()
```

**Phase 2: ORIGINAL Method**
```
Only runs once, here
```

**Phase 3: AROUND Phase (wrapping the original)**
```
Around plugins with LOWER sortOrder are OUTERMOST
Plugin A (sortOrder=10) around() {
    // Plugin B's around is INSIDE here
    Plugin B.around() {
        // Plugin C's around is INSIDE here
        Plugin C.around() {
            ORIGINAL METHOD
        }
    }
}
```

**Phase 4: AFTER Phase (descending sortOrder)**
```
Plugin C (sortOrder=30).after() → Plugin B (sortOrder=20).after() → Plugin A (sortOrder=10).after()
```

**Visual Sort Order Diagram:**

```xml
<type name="Magento\Catalog\Api\ProductRepositoryInterface">
    <plugin name="logger"     type="LoggerPlugin"     sortOrder="10"/>
    <plugin name="validator"  type="ValidatorPlugin"  sortOrder="20"/>
    <plugin name="notifier"   type="NotifierPlugin"   sortOrder="30"/>
</type>
```

Execution flow:
```
┌─────────────────────────────────────────────────────────────────┐
│ 1. LoggerPlugin::beforeSave()         (sortOrder 10, earliest) │
│ 2. ValidatorPlugin::beforeSave()       (sortOrder 20)           │
│ 3. NotifierPlugin::beforeSave()        (sortOrder 30, latest)   │
├─────────────────────────────────────────────────────────────────┤
│ 4. ORIGINAL METHOD                                             │
├─────────────────────────────────────────────────────────────────┤
│ 5. NotifierPlugin::afterSave()        (sortOrder 30, earliest) │
│ 6. ValidatorPlugin::afterSave()       (sortOrder 20)           │
│ 7. LoggerPlugin::afterSave()          (sortOrder 10, latest)   │
└─────────────────────────────────────────────────────────────────┘
```

> **Pro Tip:** When using around plugins, the one with LOWEST sortOrder is the outermost wrapper. So `sortOrder=10`'s around() wraps `sortOrder=20`'s around() which wraps the original.

**Plugin Sort Order with Mixed Plugin Types:**

When you have before, after, and around plugins together, around plugins from lower sortOrders wrap around plugins from higher sortOrders:

```
All before plugins (ascending) → All around plugins (ascending, nested) → Original → All around plugins (descending) → All after plugins (descending)
```

**Preferences — Replace Entire Class:**

```xml
<preference for="Magento\Catalog\Api\ProductRepositoryInterface"
            type="Training\Review\Custom\ProductRepository"/>
```

Replaces the entire interface implementation. Use sparingly.

> **⚠️ When to Use Preferences (and When Not To):**

| Use Preference When | Don't Use Preference When |
|---------------------|---------------------------|
| Replacing core class entirely | A plugin would suffice |
| Creating new module that IS the implementation | You just need to add behavior |
| Implementing a complete custom workflow | You need to modify specific methods |

**Why Preferences Are Dangerous:**

Preferences replace the ENTIRE class. Any plugin on the original class won't work on your replacement unless you re-implement the pluginable methods correctly.

```php
// ❌ WRONG — Preference breaks plugins on the original
<preference for="Magento\Catalog\Api\ProductRepositoryInterface"
            type="MyModule\Custom\ProductRepository"/>
// All plugins on ProductRepositoryInterface break!
```

**Virtual Types — Custom Configurations Without New Classes:**

Virtual types let you create custom configurations of existing classes without writing new PHP code. They're particularly useful for:
- Customizing constructor arguments
- Creating type-specific configurations
- Injecting different implementations based on context

```xml
<virtualType name="TrainingReviewSearchResults"
             type="Magento\Framework\Api\SearchResults">
    <arguments>
        <argument name="logger" xsi:type="object">TrainingReviewLogger</argument>
    </arguments>
</virtualType>
```

**Real-World Virtual Type from Magento Core:**

Magento's `Magento/Framework/ObjectManager/etc/di.xml` defines many virtual types for customizing framework behavior:

```xml
<!-- From Magento_Framework -->
<virtualType name="Magento\Framework\Search\Request\Aggregation\Status"
             type="Magento\Framework\Search\Request\Aggregation\Status">
    <arguments>
        <argument name="builders" xsi:type="array">
            <item name="category" xsi:type="object">
                Magento\CatalogSearch\Model\Search\Category\AggregationBuilder
            </item>
        </argument>
    </arguments>
</virtualType>
```

> **Pro Tip:** Virtual types are powerful but can be confusing. Document WHY you're using a virtual type. If you find yourself nesting virtual types deeply, consider creating an actual class instead.

**Constructor Injection via di.xml:**

You can override constructor arguments via di.xml. This is useful for:
- Injecting configuration values
- Providing different implementations for specific classes
- Testing (easy to mock)

```xml
<type name="Training\Review\Controller\Index\Index">
    <arguments>
        <argument name="paramName" xsi:type="string">custom_value</argument>
        <argument name="logger" xsi:type="object">CustomLogger</argument>
    </arguments>
</type>
```

```php
// In your controller
public function __construct(
    \Magento\Framework\App\Action\Context $context,
    private readonly string $paramName,
    private readonly LoggerInterface $logger
) {
    parent::__construct($context);
}
```

**⚠️ Constructor Injection vs Factory Injection:**

| Scenario | Approach | Example |
|----------|----------|---------|
| Get same instance every time | Constructor injection, `shared=true` | Logger, Config |
| Get new instance each time | Constructor with `shared="false"`, or use Factory | ReviewFactory |
| Conditional instance | Virtual type | Different logger per context |

**Shared vs Non-Shared — The Memory Tradeoff:**

```xml
<!-- SHARED (default): Same instance for entire request -->
<type name="Training\Review\Model\ReviewManagement"
      shared="true"/>
<!-- Result: One ReviewManagement instance, all code shares it -->

<!-- NOT SHARED: New instance every time it's injected -->
<type name="Training\Review\Model\ReviewFactory"
      shared="false"/>
<!-- Result: Each injection creates a new ReviewManagement -->
```

**When to Use Non-Shared (`shared="false"`):**

- **Factories** — Always non-shared (need new instance to create objects)
- **Stateful services** — When each use needs fresh state
- **Heavy objects** — When you don't want to keep in memory
- **Data transfer objects** — When you want to prevent accidental state sharing

> **Pro Tip from Production:** Never set repositories as non-shared — they're expensive to instantiate and should be reused. Only use `shared="false"` for factories and prototype-scoped objects.

**PSR-12 / di.xml Formatting:**

```xml
<!-- ✅ CORRECT — Proper formatting, sorted arguments -->
<type name="Training\Review\Controller\Index\Index">
    <arguments>
        <argument name="paramA" xsi:type="string">value_a</argument>
        <argument name="paramB" xsi:type="object">ServiceA</argument>
        <argument name="paramC" xsi:type="object">ServiceB</argument>
    </arguments>
</type>

<!-- ❌ WRONG — Unsorted, inconsistent formatting -->
<type name="Training\Review\Controller\Index\Index">
<arguments>
<argument name="paramB" xsi:type="object">ServiceB</argument>
<argument name="paramA" xsi:type="string">value</argument>
</arguments>
</type>
```

---

## Reading List

- [Magento 2 Plugins](https://developer.adobe.com/commerce/php/development/components/plugins/) — Plugin types, sort order
- [Events and Observers](https://developer.adobe.com/commerce/php/development/components/events/) — Event dispatch, observer registration
- [Dependency Injection](https://developer.adobe.com/commerce/php/development/components/di/) — di.xml, virtual types, preferences

---

## Edge Cases & Troubleshooting

| Issue | Symptom | Solution |
|-------|---------|----------|
| Plugin not firing | Method unchanged | Check method name in di.xml matches exactly |
| Around plugin hangs | Request times out | Missing `$proceed()` call — original never runs |
| Observer not triggered | Event not caught | Check event name in `events.xml` matches dispatch |
| Circular dependency | DI compilation error | Plugin cannot depend on the type it intercepts |
| After returns null | Original result lost | After plugin must return a value |
| Plugin on private method | Nothing happens | Only public methods support plugins |
| Virtual type not resolving | DI error on compile | Virtual type name case-sensitive, check spelling |
| Preference not taking effect | Original class still used | Clear `var/cache`, `var/generation`, run `bin/magento c:c` |
| Multiple plugins same sortOrder | Unpredictable order | Always use unique sortOrder values |
| Event observer not loading | 404 or silent skip | Observer class must implement `ObserverInterface` |

**Di.xml Compilation Issues:**

```bash
# Clear generated code and cache
bin/magento cache:clean
bin/magento cache:flush
rm -rf var/generation/*
bin/magento setup:di:compile

# Or in development mode, let Magento auto-generate
bin/magento deploy:mode:set developer
```

**Debugging Plugin Execution:**

```php
// Add this temporarily to see if plugin runs
public function beforeSave($subject, $product): array
{
    $this->logger->info('BEFORE SAVE PLUGIN CALLED', [
        'sku' => $product->getSku(),
        'trace' => (new \Exception())->getTraceAsString()
    ]);
    return [$product];
}
```

---

## Common Mistakes to Avoid

1. ❌ **Forgetting to call `$proceed()` in around plugin** → Request hangs forever; original method never runs and HTTP request times out
   - **Prevention:** Always run `$proceed()` at least once unless intentionally skipping

2. ❌ **Using preference when plugin would suffice** → Preference replaces the ENTIRE class; all plugins on the original break
   - **Prevention:** Only use preference when you need to completely replace an implementation, not just modify behavior

3. ❌ **Typo in event name in events.xml** → Observer silently never fires; no error, just doesn't work
   - **Prevention:** Copy-paste event names, verify exact match with dispatch call

4. ❌ **Plugin on non-public method** → Plugin silently doesn't work; no error thrown
   - **Prevention:** Only plugin public methods; check Magento docs for which methods are pluginable

5. ❌ **Modifying constructor args without understanding DI** → Creates hard-to-debug issues; unpredictable behavior
   - **Prevention:** Understand Magento's DI auto-wiring before modifying di.xml

6. ❌ **Returning wrong type from after plugin** → NullPointerException or type error downstream
   - **Prevention:** Always return the same type the original method returns

7. ❌ **Throwing exceptions in after plugins** → Original operation already committed; partial state possible
   - **Prevention:** Use before plugins for validation; use after only for side effects that can fail safely

8. ❌ **Around plugin calling `$proceed()` multiple times** → Duplicate operations (e.g., double-charging, duplicate saves)
   - **Prevention:** Only call `$proceed()` once unless implementing specific retry logic

9. ❌ **Not clearing cache after di.xml changes** → Changes not taking effect; confusing debugging
   - **Prevention:** Always `bin/magento c:c` after di.xml modifications in production

10. ❌ **Circular dependency through plugins** → DI compilation fails or runtime crash
    - **Prevention:** Plugin cannot depend on the type it intercepts (directly or indirectly)

---

## Pro Tips from Production

1. **Start with events, graduate to plugins** — Events are safer for decoupling; plugins are more powerful but riskier

2. **Use around plugins only when necessary** — If a before+after combo works, prefer that over around

3. **Name your plugins descriptively** — `training_review_after_save_logger` not just `plugin1`

4. **Document plugin execution order assumptions** — If plugin B depends on plugin A running first, document it in comments

5. **Log plugin execution in development** — Helps track execution order and catch missing `$proceed()` calls early

6. **Test plugins in isolation** — Unit test each plugin separately; integration test the full chain

7. **Be wary of plugin stacking** — Multiple modules with plugins on the same method can be hard to debug; document your plugins' presence

---

## Cross-References

- **Topic 06 (Admin UI):** Controllers use the same ACL pattern (`_isAllowed()`) as plugin authorization
- **Topic 07 (API):** API authentication uses plugins on `Magento\Framework\App\Action\HttpPostActionRouter`
- **Topic 08 (Data Ops):** Cron observers use the same event/observer pattern as regular observers

---

*Magento 2 Backend Developer Course — Topic 05*
