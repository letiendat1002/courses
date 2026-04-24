---
title: "23 - Dependency Injection & Plugin System"
description: "Deep dive into Magento 2.4.8 dependency injection: di.xml, preferences, virtual types, and the plugin system (around/before/after interceptors)."
tags: magento2, dependency-injection, di-xml, plugins, preferences, virtual-types, interceptors
rank: 3
pathways: [magento2-deep-dive]
---

# Dependency Injection & Plugin System

Magento 2.4.8 implements a sophisticated Dependency Injection (DI) container that serves as the architectural foundation for the entire system. Understanding DI and the Plugin system is essential for building maintainable, testable extensions that integrate seamlessly with Magento's architecture.

---

## 1. Dependency Injection Overview

### What is Dependency Injection?

Dependency Injection is a design pattern where an object's dependencies are passed to it rather than being created internally. Instead of this:

```php
// Anti-pattern: tight coupling
class ProductRepository
{
    public function __construct()
    {
        $this->database = new Magento\Framework\Model\ResourceModel\Db();
        $this->logger = new Psr\Log\NullLogger();
    }
}
```

Magento uses constructor injection:

```php
// Best practice: DI via constructor
class ProductRepository
{
    private Db $database;
    private LoggerInterface $logger;

    public function __construct(
        Db $database,
        LoggerInterface $logger
    ) {
        $this->database = $database;
        $this->logger = $logger;
    }
}
```

### Why Magento Uses DI

| Benefit | Description |
|---------|-------------|
| **Testability** | Mock dependencies easily in unit tests |
| **Loose Coupling** | Classes depend on abstractions, not implementations |
| **Flexibility** | Swap implementations via `di.xml` without code changes |
| **Single Responsibility** | Classes focus on business logic, not dependency creation |
| **Reusability** | Share implementations across multiple consumers |

### The ObjectManager Role

`Magento\Framework\ObjectManager\ObjectManager` is the central DI container in Magento 2. It handles:

- **Instance creation**: Constructs objects with injected dependencies
- **Argument resolution**: Converts di.xml arguments to PHP values
- **Plugin invocation**: Orchestrates around/before/after interceptors
- **Singleton management**: Manages shared instances across the application

The ObjectManager is never instantiated directly by application code. Magento bootstraps it during application initialization and injects it where needed via `Magento\Framework\App\ObjectManager::getInstance()`.

---

## 2. di.xml Configuration

### Configuration Hierarchy

Magento 2.4.8 merges DI configuration from multiple sources in this order (later files override earlier):

```
1. app/etc/di.xml                          (global, loaded first)
2. vendor/<vendor>/<module>/etc/di.xml      (module global)
3. app/etc/area/di.xml                     (area-specific global)
4. vendor/<vendor>/<module>/etc/<area>/di.xml (module area-specific)
```

### Schema Definition

All `di.xml` files must declare the Magento ObjectManager schema:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <!-- configuration -->
</config>
```

### Core Configuration Elements

| Element | Purpose |
|---------|---------|
| `<type>` | Configure a class: constructor args, plugins, shared state |
| `<preference>` | Replace a class implementation with another |
| `<plugin>` | Declare a plugin on a type |
| `<virtualType>` | Create a configured instance without PHP class |
| `<argument>` | Inject a value into constructor |
| `<constructor>` | Define constructor parameter mapping |

### Complete di.xml Structure Example

```xml
<?xml version="1.0" encoding="UTF-8"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Virtual Type: customized logger -->
    <virtualType name="WebapiLogger" type="Psr\Log\LoggerInterface">
        <arguments>
            <argument name="name" xsi:type="string">webapi</argument>
        </arguments>
    </virtualType>

    <!-- Preference: replace interface implementation -->
    <preference for="Magento\Catalog\Api\ProductRepositoryInterface"
                type="Vendor\Module\Model\ProductRepository"/>

    <!-- Type configuration with constructor arguments -->
    <type name="Magento\Catalog\Model\Product">
        <arguments>
            <argument name="collectionFactory" xsi:type="object">
                Magento\Framework\Data\Collection\Factory
            </argument>
            <argument name="logger" xsi:type="object">WebapiLogger</argument>
            <argument name="shared" xsi:type="boolean">false</argument>
        </arguments>

        <!-- Plugin declaration -->
        <plugin name="catalogProductPlugin"
                type="Vendor\Module\Plugin\ProductPlugin"
                sortOrder="10"
                disabled="false"/>
    </type>

</config>
```

---

## 3. Preferences

### Concept

A preference declaration tells Magento to use a specific class implementation whenever a interface or class is requested. This enables:

- **Interface-based coding**: Program to interfaces, swap implementations via config
- **Extension overrides**: Replace core classes without modifying original code
- **Platform customization**: Swap Magento core components with custom implementations

### Syntax

```xml
<preference for="Original\Class\Or\Interface" type="New\Implementation\Class"/>
```

### Real-World Examples from M2.4.8

**Example 1: Interface Implementation Replacement**

From `vendor/magento/module-catalog/etc/di.xml`:

```xml
<preference for="Magento\Catalog\Api\ProductRepositoryInterface"
            type="Magento\Catalog\Model\ProductRepository"/>
```

**Example 2: Abstract Class Replacement**

```xml
<preference for="Magento\Framework\MessageQueue\ConsumerInterface"
            type="Magento\Framework\MessageQueue\Consumer"/>
```

**Example 3: Model Replacement**

From `vendor/magento/module-customer/etc/di.xml`:

```xml
<preference for="Magento\Customer\Model\Customer"
            type="Magento\Customer\Model\ResourceModel\Customer"/>
```

### Plugin vs Preference

| Aspect | Plugin | Preference |
|--------|--------|------------|
| **Mechanism** | Wraps method calls | Replaces entire class |
| **Scope** | Per-method interception | Complete class substitution |
| **Execution** | Before/after/around method | Replaces instantiation |
| **Use Case** | Modify behavior without replacing | Swap implementation entirely |

---

## 4. Virtual Types

### Concept

A virtual type is a **configured instance** of an existing class, defined entirely in `di.xml`. No PHP class is created - Magento generates one at runtime. This allows different constructor arguments for the same base class.

### When to Use Virtual Types

- You need a configured instance of a class with specific constructor args
- You want to give the configuration a semantic name
- You need multiple "variants" of the same class with different configs

### Basic Virtual Type Example

```xml
<!-- Define a virtual type with custom logger name -->
<virtualType name="WebapiLogger" type="Psr\Log\LoggerInterface">
    <arguments>
        <argument name="name" xsi:type="string">webapi</argument>
    </arguments>
</virtualType>
```

Then use it as an argument:

```xml
<type name="Magento\Webapi\Controller\Rest">
    <arguments>
        <argument name="logger" xsi:type="object">WebapiLogger</argument>
    </arguments>
</type>
```

### Advanced Virtual Type: Different Collections

A common use case is configuring different collection classes:

```xml
<!-- Virtual type for customer grid collection -->
<virtualType name="CustomerGridCollection" type="Magento\Framework\View\Element\UiComponent\DataProvider\CollectionFactory">
    <arguments>
        <argument name="collectionName" xsi:type="string">customer_listing</argument>
        <argument name="moduleName" xsi:type="string">Magento_Customer</argument>
    </arguments>
</virtualType>

<!-- Use the virtual type in a data provider -->
<type name="Magento\Customer\Ui\Component\DataProvider">
    <arguments>
        <argument name="collectionFactory" xsi:type="object">CustomerGridCollection</argument>
    </arguments>
</type>
```

### Virtual Type Inheritance Gotcha

Virtual types **cannot** inherit from other virtual types. Each virtualType must reference a concrete PHP type as its base:

```xml
<!-- VALID -->
<virtualType name="LoggerWithPrefix" type="Psr\Log\LoggerInterface">
    <arguments>
        <argument name="name" xsi:type="string">custom.prefix</argument>
    </arguments>
</virtualType>

<!-- INVALID - virtual types cannot extend virtual types -->
<virtualType name="ExtendedLogger" type="LoggerWithPrefix">
    <!-- This will NOT work -->
</virtualType>
```

---

## 5. Plugin System (Interceptors)

### How Plugins Work

Plugins in Magento 2 use the **Interceptor Pattern** implemented via the `__call` magic method. When you call any method on a class that has plugins, Magento's generated interceptor:

1. Looks up all plugins for that class
2. Builds an interceptor chain based on `sortOrder`
3. Executes plugins in order: `before` → `original` → `after`
4. For `around` plugins, wraps the original method

### Request Flow Diagram

```
Method Call on Proxied Class
        │
        ▼
Interceptor::__call()
        │
        ▼
Plugin Chain Execution
        │
   ┌────┴────┐
   ▼         ▼
before    around (first)
plugins   │
           │
           ▼
     Original Method
           │
           ▼
     around (last)
           │
   ┌────┴────┐
   ▼         ▼
after     around (middle)
plugins   │
           │
           ▼
      Return Value
```

### Plugin Declaration in di.xml

```xml
<type name="Magento\Catalog\Model\Product">
    <plugin name="inventoryPlugin"
            type="Vendor\Module\Plugin\ProductInventoryPlugin"
            sortOrder="10"
            disabled="false"/>
</type>
```

### Generated Interceptor Classes

Magento automatically generates interceptor classes in `var/generation/<namespace>/<class>/Interceptor.php`. You should never modify these files - they're regenerated on every setup:upgrade.

---

## 6. Plugin Types

### Before Plugins

`before` plugins modify method arguments **before** the original method executes. The plugin method must return an **array** of arguments (even if empty) or `null`.

**Use Cases:**
- Validate or sanitize input arguments
- Add additional arguments derived from existing ones
- Prevent method execution by throwing exception

**Signature:**
```php
public function before<MethodName>(<OriginalClass> $subject, <methodArgs>...): array
```

**Example:**
```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin;

use Magento\Catalog\Model\Product;

class ProductPlugin
{
    /**
     * Before setName - validate and sanitize product name
     */
    public function beforeSetName(Product $subject, string $name): array
    {
        // Validate minimum length
        if (strlen($name) < 3) {
            throw new \InvalidArgumentException(
                'Product name must be at least 3 characters'
            );
        }

        // Trim whitespace and return modified arguments
        $cleanName = trim($name);
        return [$cleanName];
    }

    /**
     * Before setPrice - convert price to integer cents
     */
    public function beforeSetPrice(Product $subject, float $price): array
    {
        // Convert to integer cents to avoid floating point issues
        $priceInCents = (int) round($price * 100);
        return [$priceInCents];
    }
}
```

### After Plugins

`after` plugins modify the **return value** after the original method executes. The plugin receives the original return value and can return a modified value.

**Use Cases:**
- Transform return value (format, convert, wrap)
- Add logging or analytics
- Post-processing operations

**Signature:**
```php
public function after<MethodName>(<OriginalClass> $subject, <returnType> $result, <methodArgs>...): <returnType>
```

**Example:**
```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin;

use Magento\Catalog\Model\Product;

class ProductPlugin
{
    /**
     * After getName - add suffix to product name
     */
    public function afterGetName(Product $subject, string $result): string
    {
        return $result . ' [Verified]';
    }

    /**
     * After getPrice - convert from cents back to float
     */
    public function afterGetPrice(Product $subject, int $result): int
    {
        // Price was stored in cents, return as-is
        return $result;
    }

    /**
     * After getProductUrl - add tracking parameter
     */
    public function afterGetProductUrl(
        Product $subject,
        string $result,
        string $anchor = ''
    ): string {
        $separator = strpos($result, '?') !== false ? '&' : '?';
        return $result . $separator . 'utm_source=plugin&utm_medium=product';
    }
}
```

### Around Plugins

`around` plugins **wrap** the entire original method. They receive a `$proceed` callback that must be called to execute the original method. This is the most powerful but also most dangerous plugin type.

**Use Cases:**
- Completely replace method behavior
- Conditional method execution
- Add before/after logic with shared state
- Skip original method entirely

**Signature:**
```php
public function around<MethodName>(
    <OriginalClass> $subject,
    callable $proceed,
    <methodArgs>...
): <returnType>
```

**Example:**
```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin;

use Magento\Catalog\Model\Product;
use Magento\Framework\Exception\LocalizedException;

class ProductPlugin
{
    /**
     * Around save - add validation and custom logic
     */
    public function aroundSave(
        Product $subject,
        callable $proceed
    ): Product {
        // Before: Validate product data
        if (!$subject->getName()) {
            throw new LocalizedException(
                __('Product name is required')
            );
        }

        // Call the original method
        $result = $proceed();

        // After: Post-save operations
        $this->indexService->reindex($subject->getId());

        return $result;
    }

    /**
     * Around delete - conditional execution
     */
    public function aroundDelete(
        Product $subject,
        callable $proceed
    ): bool {
        // Check if product can be deleted
        if ($subject->isComposite()) {
            throw new LocalizedException(
                __('Composite products cannot be deleted')
            );
        }

        // Execute original delete
        return $proceed();
    }

    /**
     * Around load - implement caching
     */
    public function aroundLoad(
        \Magento\Catalog\Model\Product $subject,
        callable $proceed,
        int $productId,
        ?int $storeId = null
    ): Product {
        $cacheKey = "product_{$productId}_{$storeId}";

        // Check cache first
        if ($this->cache->has($cacheKey)) {
            return $this->cache->get($cacheKey);
        }

        // Load via original method
        $product = $proceed($productId, $storeId);

        // Cache the result
        if ($product !== null) {
            $this->cache->set($cacheKey, $product);
        }

        return $product;
    }
}
```

### Plugin Method Naming Convention

| Original Method | Before Method | After Method | Around Method |
|----------------|---------------|--------------|---------------|
| `setName(string $name)` | `beforeSetName` | `afterSetName` | `aroundSetName` |
| `getPrice()` | `beforeGetPrice` | `afterGetPrice` | `aroundGetPrice` |
| `save()` | `beforeSave` | `afterSave` | `aroundSave` |
| `delete()` | `beforeDelete` | `afterDelete` | `aroundDelete` |

---

## 7. Plugin Configuration

### sortOrder Attribute

When multiple plugins are declared on the same class, `sortOrder` determines execution priority. **Lower values execute first.**

```xml
<type name="Magento\Catalog\Model\Product">
    <!-- Executes first (sortOrder=10) -->
    <plugin name="loggingPlugin"
            type="Vendor\Module\Plugin\ProductLoggingPlugin"
            sortOrder="10"/>

    <!-- Executes second (sortOrder=20) -->
    <plugin name="validationPlugin"
            type="Vendor\Module\Plugin\ProductValidationPlugin"
            sortOrder="20"/>

    <!-- Executes last (sortOrder=30) -->
    <plugin name="analyticsPlugin"
            type="Vendor\Module\Plugin\ProductAnalyticsPlugin"
            sortOrder="30"/>
</type>
```

**Execution Chain for `getName()`:**

```
beforeLoggingPlugin → beforeValidationPlugin → beforeAnalyticsPlugin
    → getName() (original)
afterAnalyticsPlugin → afterValidationPlugin → afterLoggingPlugin
```

### disabled Attribute

You can disable a plugin without removing its declaration:

```xml
<plugin name="debugPlugin"
        type="Vendor\Module\Plugin\ProductDebugPlugin"
        sortOrder="10"
        disabled="true"/>
```

This is useful for:
- Temporarily disabling plugins during debugging
- Environment-specific plugin activation (via environment variables)
- Conditional plugin execution based on configuration

### Nested Plugins (Plugin on Plugin)

Plugins can be applied to classes that are themselves plugins. This creates nested interceptor chains:

```xml
<!-- Plugin A on original class -->
<type name="Magento\Catalog\Model\Product">
    <plugin name="pluginA" type="Vendor\Module\Plugin\PluginA" sortOrder="10"/>
</type>

<!-- But PluginB is applied to PluginA -->
<type name="Vendor\Module\Plugin\PluginA">
    <plugin name="pluginB" type="Vendor\Module\Plugin\PluginB" sortOrder="5"/>
</type>
```

**Execution chain for `Product::getName()`:**

```
beforePluginB → beforePluginA → getName() → afterPluginA → afterPluginB
```

---

## 8. Plugin Limitations

### Cannot Plugin These Methods

| Limitation | Reason |
|------------|--------|
| **Final methods** | Cannot be overridden |
| **Final classes** | Cannot be extended |
| **Constructor** | Called during instantiation before plugins apply |
| **Non-public methods** | Plugin system only intercepts public methods |
| **`__call` / `__callStatic`** | Not intercepted by plugin system |
| **Static methods** | Class instance not available for interception |

### Classes with @noinspection

If a class is annotated with `@noinspection` for plugin inspection, plugins won't work:

```php
/**
 * @noinspection PluginInspection
 */
class SomeClass
{
    // Plugins will NOT work on this class
}
```

### Around Plugin Gotchas

**1. Must call $proceed():**

```php
// WRONG - will never execute original method
public function aroundSave(Product $subject, callable $proceed): Product
{
    // Missing $proceed() call - original method never runs!
    return $subject;
}

// CORRECT
public function aroundSave(Product $subject, callable $proceed): Product
{
    // Custom before logic
    $this->validate($subject);

    // Call original
    $result = $proceed();

    // Custom after logic
    $this->postProcess($result);

    return $result;
}
```

**2. Argument passing in around plugins:**

```php
// CORRECT - use variadic for flexible argument handling
public function aroundSetName(
    Product $subject,
    callable $proceed,
    string $name
): Product {
    $modifiedName = $this->transform($name);
    return $proceed($modifiedName);
}

// For methods with variable arguments:
public function aroundExecute(
    Product $subject,
    callable $proceed,
    ...$args
) {
    // $args contains all method arguments
    return $proceed(...$args);
}
```

**3. SortOrder complexity with around plugins:**

When mixing before/after/around plugins, the sortOrder for around plugins determines when they're called **relative to before plugins** but **before the original method**. The exact execution order can be subtle.

---

## 9. Service Contracts & API Interfaces

### Why Service Contracts

Service Contracts define **public APIs** that are versioned and stable. By coding against interfaces, modules remain compatible even when underlying implementations change.

### Repository Pattern in Magento

Magento uses the Repository pattern to decouple business logic from data access:

```
┌─────────────────────────────────┐
│     Service Contract (API)     │
│   Magento\Catalog\Api\ProductRepositoryInterface │
└────────────────┬────────────────┘
                  │ (implemented by)
                  ▼
┌─────────────────────────────────┐
│        Implementation          │
│   Magento\Catalog\Model\ProductRepository │
└────────────────┬────────────────┘
                  │
                  ▼
┌─────────────────────────────────┐
│        Data Source              │
│   Magento\Catalog\Model\ResourceModel\Product │
└─────────────────────────────────┘
```

### Interface Structure

**Repository Interface** (`Magento\Catalog\Api\ProductRepositoryInterface`):

```php
<?php
declare(strict_types=1);

namespace Magento\Catalog\Api;

use Magento\Catalog\Api\Data\ProductInterface;
use Magento\Framework\Api\SearchCriteriaInterface;

interface ProductRepositoryInterface
{
    /**
     * Get product by ID
     */
    public function get(int $id): ProductInterface;

    /**
     * Get product by SKU
     */
    public function getBySku(string $sku): ProductInterface;

    /**
     * Save product (create or update)
     */
    public function save(ProductInterface $product): ProductInterface;

    /**
     * Delete product
     */
    public function delete(ProductInterface $product): bool;

    /**
     * Find products by search criteria
     */
    public function getList(SearchCriteriaInterface $searchCriteria): \Magento\Catalog\Api\Data\ProductSearchResultsInterface;
}
```

**Implementation** (`Magento\Catalog\Model\ProductRepository`):

```php
<?php
declare(strict_types=1);

namespace Magento\Catalog\Model;

use Magento\Catalog\Api\Data\ProductInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Catalog\Model\ResourceModel\Product as ProductResource;
use Magento\Framework\Exception\NoSuchEntityException;

class ProductRepository implements ProductRepositoryInterface
{
    private ProductResource $resource;
    private \Magento\Catalog\Api\Data\ProductInterfaceFactory $factory;

    public function __construct(
        ProductResource $resource,
        \Magento\Catalog\Api\Data\ProductInterfaceFactory $factory
    ) {
        $this->resource = $resource;
        $this->factory = $factory;
    }

    public function get(int $id): ProductInterface
    {
        $product = $this->factory->create();
        $this->resource->load($product, $id);

        if (!$product->getId()) {
            throw new NoSuchEntityException(
                __('Product with ID %1 does not exist', $id)
            );
        }

        return $product;
    }

    public function save(ProductInterface $product): ProductInterface
    {
        $this->resource->save($product);
        return $product;
    }
}
```

### Data Interfaces

Data interfaces (in `Api/Data`) define the structure of data objects:

```php
<?php
declare(strict_types=1);

namespace Magento\Catalog\Api\Data;

use Magento\Framework\Api\ExtensibleDataInterface;

interface ProductInterface extends ExtensibleDataInterface
{
    /**
     * @return int|null
     */
    public function getId(): ?int;

    /**
     * @return string
     */
    public function getName(): string;

    /**
     * @param string $name
     * @return ProductInterface
     */
    public function setName(string $name): ProductInterface;

    /**
     * @return float
     */
    public function getPrice(): float;

    /**
     * @param float $price
     * @return ProductInterface
     */
    public function setPrice(float $price): ProductInterface;

    // ... more methods
}
```

---

## 10. Argument Types in di.xml

### Available Argument Types

| Type | XML Value | PHP Equivalent |
|------|-----------|---------------|
| String | `xsi:type="string"` | `string` |
| Boolean | `xsi:type="boolean"` | `bool` |
| Number | `xsi:type="number"` | `int` or `float` |
| Integer | `xsi:type="int"` | `int` |
| Array | `xsi:type="array"` | `array` |
| Object | `xsi:type="object"` | Instance of class |
| Null | `xsi:type="null"` | `null` |

### Primitive Arguments

```xml
<type name="Vendor\Module\Model\Example">
    <arguments>
        <!-- String -->
        <argument name="apiKey" xsi:type="string">abc123xyz</argument>

        <!-- Boolean -->
        <argument name="isEnabled" xsi:type="boolean">true</argument>

        <!-- Integer -->
        <argument name="maxItems" xsi:type="int">100</argument>

        <!-- Float -->
        <argument name="taxRate" xsi:type="number">0.085</argument>
    </arguments>
</type>
```

### Object Arguments

```xml
<type name="Vendor\Module\Controller\Index\Index">
    <arguments>
        <!-- Reference to another configured type -->
        <argument name="logger" xsi:type="object">WebapiLogger</argument>

        <!-- Reference to a class directly -->
        <argument name="productRepository" xsi:type="object">
            Magento\Catalog\Api\ProductRepositoryInterface
        </argument>
    </arguments>
</type>
```

### Array Arguments

```xml
<type name="Vendor\Module\Model\Config">
    <arguments>
        <!-- Inline array -->
        <argument name="settings" xsi:type="array">
            <item name="timeout" xsi:type="int">30</item>
            <item name="retries" xsi:type="int">3</item>
            <item name="endpoints" xsi:type="array">
                <item name="primary">https://api.example.com</item>
                <item name="backup">https://backup.example.com</item>
            </item>
        </argument>
    </arguments>
</type>
```

### Shared Instances

Control whether the ObjectManager returns the same instance (`shared="true"`) or creates a new instance (`shared="false"`) each time:

```xml
<type name="Vendor\Module\Model\SharedService">
    <!-- Same instance every time -->
    <argument name="config" xsi:type="shared">true</argument>
</type>

<type name="Vendor\Module\Model\NonSharedService">
    <!-- New instance every time -->
    <argument name="builder" xsi:type="shared">false</argument>
</type>
```

---

## 11. Factory Classes

### What are Factory Classes?

Factory classes are **generated classes** that create object instances via the ObjectManager. They're needed for classes that:
- Are not injectible (no interface, or non-service class)
- Need to be created at runtime with varying parameters
- Cannot be dependency-injected due to their nature (e.g., `Product`, `Order`)

### Generated Factory Naming

Magento generates factories with the naming pattern: `<OriginalClass>Factory`

| Original Class | Factory Class |
|---------------|---------------|
| `Magento\Catalog\Model\Product` | `Magento\Catalog\Model\ProductFactory` |
| `Magento\Sales\Model\Order` | `Magento\Sales\Model\OrderFactory` |
| `Vendor\Module\Model\CustomEntity` | `Vendor\Module\Model\CustomEntityFactory` |

### Using Factories in Constructors

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Controller\Index;

use Magento\Framework\View\Result\PageFactory;
use Magento\Catalog\Model\ProductFactory;

class Index extends \Magento\Framework\App\Action\Action
{
    private ProductFactory $productFactory;
    private PageFactory $resultPageFactory;

    public function __construct(
        \Magento\Framework\App\Action\Context $context,
        ProductFactory $productFactory,
        PageFactory $resultPageFactory
    ) {
        parent::__construct($context);
        $this->productFactory = $productFactory;
        $this->resultPageFactory = $resultPageFactory;
    }

    public function execute(): \Magento\Framework\Controller\ResultInterface
    {
        // Create a NEW Product instance (not singleton)
        $product = $this->productFactory->create();

        // Load product data
        $product->load($this->getRequest()->getParam('id'));

        // Work with the product...

        return $this->resultPageFactory->create();
    }
}
```

### Factory vs Dependency Injection

| Aspect | Constructor Injection | Factory Injection |
|--------|----------------------|-------------------|
| **Creation time** | At object construction | At runtime when needed |
| **Use case** | Services, dependencies | Entities, models |
| **Testability** | Easy to mock | Can mock factory |
| **Lifecycle** | Managed by ObjectManager | Controlled by developer |

### Creating Objects with Constructor Arguments

```php
<?php
// Using factory to create with specific arguments
$product = $this->productFactory->create([
    'data' => [
        'name' => 'Custom Product',
        'price' => 99.99,
        'sku' => 'CUSTOM-SKU'
    ]
]);

// Or set after creation
$product = $this->productFactory->create();
$product->setName('Custom Product');
$product->setPrice(99.99);
```

### Why Factories are Needed

**Problem**: Magento's DI system injects dependencies via constructor. But some classes (like `Product`) are not services - they're entities with identity (ID), and you may need multiple instances.

**Solution**: Factories create new instances on-demand:

```php
// WRONG - Product is not a service, shouldn't be injected directly
public function __construct(Product $product)  // Always same instance!

// CORRECT - Use factory for entity classes
public function __construct(ProductFactory $productFactory)
{
    $this->productFactory = $productFactory;
}

// Then create instances as needed
$product1 = $this->productFactory->create();
$product2 = $this->productFactory->create();  // Different instance
```

---

## 12. Proxy Classes

### Concept

Proxy classes are **generated subclasses** that implement the same interface as the original class but defer expensive initialization until a method is actually called. This is "lazy loading" at the object level.

### When to Use Proxies

| Use Case | Example |
|----------|---------|
| **Expensive constructor** | Heavy dependency not always used |
| **Database connection** | Load only when needed |
| **External API client** | Initialize only when called |
| **Large data structure** | Defer loading until accessed |

### Proxy Class Naming

Proxy classes are named with a `Proxy` suffix: `<OriginalClass>Proxy`

| Original Class | Proxy Class |
|---------------|-------------|
| `Magento\Catalog\Model\Product` | `Magento\Catalog\Model\ProductProxy` |
| `Vendor\Module\Model\HeavyService` | `Vendor\Module\Model\HeavyServiceProxy` |

### Configuration in di.xml

```xml
<type name="Vendor\Module\Controller\Index\Index">
    <arguments>
        <!-- Inject the proxy instead of the real class -->
        <argument name="heavyService" xsi:type="object">
            Vendor\Module\Model\HeavyService\Proxy
        </argument>
    </arguments>
</type>
```

### Generated Proxy Structure

Proxies inherit from the original class and override all public methods:

```php
<?php
declare(strict_types=1);

// Auto-generated proxy class
namespace Magento\Catalog\Model;

class ProductProxy extends \Magento\Catalog\Model\Product
{
    private ?Product $instance = null;

    public function getName(): string
    {
        // Lazy initialization on first method call
        if ($this->instance === null) {
            $this->instance = $this->_objectFactory->create(Product::class);
        }
        return $this->instance->getName();
    }

    public function save(): Product
    {
        if ($this->instance === null) {
            $this->instance = $this->_objectFactory->create(Product::class);
        }
        return $this->instance->save();
    }

    // ... all public methods delegated lazily
}
```

### Real-World Example: Inventory Management

```xml
<!-- In your module's di.xml -->
<type name="Vendor\Module\Model\OrderProcessor">
    <arguments>
        <!-- Stock checking is expensive - only load when actually checking stock -->
        <argument name="stockService" xsi:type="object">
            Magento\CatalogInventory\Api\StockManagementInterface\Proxy
        </argument>
    </arguments>
</type>
```

### Proxy vs Virtual Type

| Aspect | Proxy | Virtual Type |
|--------|-------|--------------|
| **Class generation** | Extends original class | Configured instance of original |
| **Purpose** | Lazy initialization | Different configuration |
| **Arguments** | Same as original | Can be different |
| **Use case** | Performance optimization | Semantic naming of config |

### Automatic Proxy Generation

Magento automatically generates proxies when:
1. A class is requested via di.xml with ` Proxy` suffix
2. The actual proxy file doesn't exist in `var/generation`

You can also configure proxy behavior in di.xml:

```xml
<type name="Vendor\Module\Model\HeavyService" shared="false">
    <!-- When injected, Magento may create a proxy instead -->
</type>
```

---

## Summary: DI Configuration Decision Tree

```
Need to change constructor arguments of a class?
    │
    ├─► Create VIRTUAL TYPE for semantic naming
    │
Need to replace a class entirely?
    │
    ├─► Use PREFERENCE to swap implementation
    │
Need to modify method behavior without replacing class?
    │
    ├─► Create PLUGIN (before/after/around)
    │
Need to delay expensive initialization?
    │
    ├─► Configure PROXY class injection
    │
Need to create entity/model instances in code?
    │
    ├─► Use FACTORY class
```

---

## Key Takeaways

1. **Constructor Injection**: Always prefer constructor injection over property injection or ObjectManager direct usage.

2. **di.xml Hierarchy**: Global `app/etc/di.xml` loads first, module area-specific files load last and can override.

3. **Preferences Replace Classes**: Use `<preference>` to completely replace a class implementation.

4. **Virtual Types for Configuration**: Use `<virtualType>` to create semantically-named configured instances without PHP classes.

5. **Plugins Wrap Methods**: Plugins intercept method calls via the interceptor pattern - they don't replace the class.

6. **Plugin Types**: `before` modifies arguments, `after` modifies return value, `around` wraps entire method.

7. **sortOrder Matters**: Lower values execute first. Around plugins wrap the original between all befores and afters.

8. **Plugin Limitations**: Cannot plugin final methods/classes, constructors, or non-public methods.

9. **Service Contracts**: Code against interfaces (`*RepositoryInterface`) for stable, versioned APIs.

10. **Factories for Entities**: Use generated factory classes (`*Factory`) to create entity/model instances.

11. **Proxies for Lazy Loading**: Use proxy classes to defer expensive initialization until actually needed.

---

## Additional Resources

- [Magento 2.4.8 Developer Documentation](https://developer.adobe.com/commerce/php/development/)
- [Dependency Injection Configuration](https://developer.adobe.com/commerce/php/development/components/dependency-injection/)
- [Plugin Developer Guide](https://developer.adobe.com/commerce/php/development/components/plugins/)
- [Service Contracts](https://developer.adobe.com/commerce/php/architecture/domain/service-layer/)