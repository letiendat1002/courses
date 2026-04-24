---
title: "02 - PHP 8 Internals"
description: "Deep dive into PHP 8.x internals and advanced features for Magento 2.4.8 development: named arguments, attributes, union types, readonly, enums, match expressions, constructor promotion, fibers, and performance"
tags:
  - magento2
  - php8
  - php83
  - type-system
  - attributes
  - internals
  - performance
rank: 2
pathways:
  - magento2-deep-dive
see_also:
  - /home/letiendat1002/workspace/personal/courses/magento2/v2.4.8/01-dev-environment/01-php8-compatibility.md
---

# PHP 8 Internals

This chapter dives deeper into PHP 8.x internals, exploring advanced features and their practical applications within the Magento 2.4.8 codebase. While the previous chapter covered PHP 8 compatibility at a surface level, here we examine the *why* behind each feature, internal mechanisms, and idiomatic patterns used throughout Magento's core code.

---

## 1. Named Arguments — Beyond the Basics

Named arguments, introduced in PHP 8.0, allow you to pass arguments by name rather than position. This is particularly valuable in Magento's service contracts where method signatures can include multiple optional parameters.

### How Named Arguments Work Internally

PHP 8.0 introduced named arguments as a compile-time transformation. When you write:

```php
$objectManager->create(Product::class, [
    'sku' => 'SKU123',
    'name' => 'Product Name',
]);
```

The PHP engine internally resolves these to positional arguments before passing them to the function. This means named arguments don't change the runtime performance characteristics — they're purely a developer experience improvement.

### Magento Service Contract Patterns

Magento's service contracts often follow this pattern in `Vendor\Module\Api\Data\SomeInterface`:

```php
// vendor/magento/module-catalog/Api/Data/ProductInterface.php
public function setData($key, $value = null);
```

When calling such methods with configuration arrays, named arguments provide clarity:

```php
// Without named arguments - unclear what 'true' means
$product->setData('price', 29.99);
$product->setData('status', 1, true, 'Product updated');

// With named arguments - self-documenting
$product->setData(
    key: 'price',
    value: 29.99
);
$product->setData(
    key: 'status',
    value: 1,
    forceImport: true,
    reason: 'Product updated'
);
```

### Combining with Default Values

Named arguments shine when combining with default values. Consider Magento's `SearchCriteriaBuilder`:

```php
// vendor/magento/module-framework/Search/SearchCriteriaBuilder.php
public function setRequestName(string $requestName): self;
public function setCurrentPage(int $currentPage = 1): self;
public function setPageSize(int $pageSize = 0): self;
```

You can override only what you need:

```php
$criteriaBuilder
    ->setRequestName('advanced_search')
    ->setCurrentPage(pageSize: 20);  // Only override pageSize, currentPage uses default (1)
```

### The `...` Variadic with Named Arguments

The variadic operator (`...`) combines elegantly with named arguments:

```php
// In Magento's Batch Data Translator
public function translate(
    array $entities,
    string $destinationField,
    callable $callback = null,
    array $additionalData = []
): array;

// Call with named arguments and spread
$translateData = [...$entityData];
$result = $translator->translate(
    entities: $translateData,
    destinationField: 'sku',
    callback: fn($e) => strtoupper($e),
    additionalData: ['strict' => true]
);
```

### Common Pitfall: Named Arguments and Trailing Comma

PHP 8.0 allows trailing commas in parameter lists, which pairs well with named arguments:

```php
// This is valid PHP 8.0+
$objectManager->create(
    Product::class,
    [
        'sku' => 'SKU123',
        'name' => 'Product Name',  // trailing comma - valid!
    ],
);
```

However, mixing positional and named arguments has restrictions:

```php
// Valid - named after positional
someFunction($positional, named: 'value');

// INVALID - positional after named
someFunction(named: 'value', $positional);  // Parse error!
```

---

## 2. Attributes (PHP 8.0+) — Internal Architecture

PHP attributes provide a standardized way to add metadata to classes, methods, properties, and functions. Understanding how Magento uses attributes internally helps you read the codebase and create extensible custom modules.

### Attribute Internal Representation

When PHP parses an attribute like `#[Deprecated]`, it creates an `Attribute` instance accessible via the Reflection API:

```php
#[Deprecated(message: 'Use newMethod() instead', since: '2.4.0')]
public function oldMethod(): void {}

// Accessing via reflection
$reflection = new ReflectionMethod(MyClass::class, 'oldMethod');
$attributes = $reflection->getAttributes(Deprecated::class);
$deprecatedAttr = $attributes[0]->newInstance();

echo $deprecatedAttr->getMessage();  // "Use newMethod() instead"
echo $deprecatedAttr->getSince();      // "2.4.0"
```

### Magento's `[Magento\Framework\DataObject\Mapping\Column]`

This attribute maps data object properties to database columns:

```php
// vendor/magento/module-eav/Model/Entity/AbstractEntity.php
use Magento\Framework\DataObject\Mapping\Column;

#[Column(name: 'created_at', type: 'datetime')]
protected $createdAt;
```

The attribute is processed by `Magento\Framework\DataObject\Mapping\Mapper`:

```php
public function getColumnMapping(DataObject $entity): array
{
    $mapping = [];
    $reflection = new ReflectionClass($entity);
    
    foreach ($reflection->getProperties() as $property) {
        $columnAttr = $property->getAttributes(Column::class)[0] ?? null;
        if ($columnAttr) {
            $config = $columnAttr->getArguments();
            $mapping[$property->getName()] = $config;
        }
    }
    
    return $mapping;
}
```

### Magento's `[Magento\Framework\Validator\EntityAttributes]`

This attribute marks a class as providing validation attributes:

```php
// vendor/magento/module-framework/Validator/EntityAttributes.php
#[\Attribute(\Attribute::TARGET_CLASS)]
class EntityAttributes
{
    public function __construct(
        public readonly string $entityType,
        public readonly array $attributes = []
    ) {}
}
```

Usage in Magento's validation system:

```php
#[EntityAttributes(entityType: 'customer', attributes: ['email', 'firstname'])]
class CustomerValidator
{
    // Validation logic for customer entity
}
```

### Magento's `[Magento\Framework\Data\Link]`

Used for linked data in Magento's data architecture:

```php
// vendor/magento/module-framework/Data/Link.php
#[\Attribute(\Attribute::TARGET_PROPERTY)]
class Link
{
    public function __construct(
        public readonly string $rel,
        public readonly string $type,
        public readonly ?string $href = null
    ) {}
}
```

Applied to data objects:

```php
class OrderItemData extends \Magento\Framework\DataObject
{
    #[Link(rel: 'product', type: 'application/json', href: '/api/products/{sku}')]
    private string $productLink;
}
```

### Custom Attribute Classes in Magento

Creating a custom attribute for Magento module development:

```php
#[\Attribute(\Attribute::TARGET_CLASS | \Attribute::TARGET_METHOD)]
class TrackingField
{
    public function __construct(
        public readonly string $fieldName,
        public readonly string $gridColumn,
        public readonly bool $searchable = false,
        public readonly bool $filterable = true
    ) {}
}
```

Applying and using it:

```php
#[TrackingField(fieldName: 'order_id', gridColumn: 'ID', searchable: true)]
class OrderGridFilter
{
    public function getTrackingFieldConfig(): array
    {
        $reflection = new ReflectionClass(self::class);
        $attr = $reflection->getAttributes(TrackingField::class)[0] ?? null;
        
        if (!$attr) {
            return [];
        }
        
        return $attr->newInstance();
    }
}
```

### The `#[RestrictedMapping]` Attribute (Magento Specific)

Magento 2.4.x uses this internal attribute in the ORM:

```php
#[\Attribute(\Attribute::TARGET_CLASS)]
class RestrictedMapping
{
    public function __construct(
        public readonly array $exclude = [],
        public readonly array $emap = []
    ) {}
}
```

### Attribute Targets and Combinations

Understanding attribute targets is crucial for proper usage:

| Target Constant | Applies To | Magento Example |
|-----------------|------------|-----------------|
| `TARGET_CLASS` | Classes | `#[EntityAttributes]` |
| `TARGET_METHOD` | Methods | `#[Deprecated]` |
| `TARGET_PROPERTY` | Properties | `#[Column]`, `#[Link]` |
| `TARGET_PARAMETER` | Parameters | Internal PHP use |
| `TARGET_FUNCTION` | Functions | `#[Attribute]` itself |
| `TARGET_CLASS_CONSTANT` | Class constants | PHP 8.3+ |

```php
// Combining targets for broader application
#[\Attribute(\Attribute::TARGET_CLASS | \Attribute::TARGET_METHOD)]
class MagentoApiEndpoint
{
    public function __construct(
        public readonly string $route,
        public readonly string $method = 'GET'
    ) {}
}

// Can apply to class
#[MagentoApiEndpoint(route: '/v1/products')]
class ProductApiService
{
    // or to method
    #[MagentoApiEndpoint(route: '/search', method: 'POST')]
    public function search(): array {}
}
```

### Reading Attributes at Runtime

Magento's `Magento\Framework\Reflection\AttributesReader` processes attributes:

```php
use Magento\Framework\Reflection\AttributesReader;
use Magento\Framework\Api\CustomAttributesDataInterface;

class MyService
{
    public function __construct(
        private AttributesReader $attributesReader
    ) {}

    public function processObject(object $object): array
    {
        $attributes = $this->attributesReader->getAttributesData($object);
        
        return array_map(
            fn($attr) => [
                'name' => $attr->getName(),
                'value' => $attr->getValue(),
            ],
            $attributes
        );
    }
}
```

---

## 3. Union Types — Method Signatures in Depth

Union types allow specifying multiple possible types for a parameter or return value. PHP 8.0 formalized what Magento's internal code was doing with `@param` docblocks for years.

### Internal Representation

When PHP compiles a union type like `string|int|null`, it stores this as an array of type entries:

```php
public function process(int|string $id): ProductInterface|null {}

// Internally represented as:
// Parameter #0: int|string
// Return: ProductInterface|null
```

### Common Magento Union Patterns

#### `string|int|null` — Identifiers

Magento frequently uses this for identifiers that might be passed as either:

```php
// vendor/magento/module-catalog/Api/ProductRepositoryInterface.php
public function getById(int|string $id, bool $editMode = false): ProductInterface;
```

This signature allows:

```php
$product = $productRepo->getById(123);           // int
$product = $productRepo->getById('123');         // string (from request)
$product = $productRepo->getById('SKU-123');     // string SKU
```

#### `array|string` — Flexible Input

```php
// vendor/magento/module-eav/Model/Entity/AbstractEntity.php
public function setData(string|array $key, mixed $value = null): self;
```

The implementation handles both:

```php
public function setData(string|array $key, mixed $value = null): self
{
    if (is_array($key)) {
        foreach ($key as $k => $v) {
            $this->setData($k, $v);  // Recursive with array
        }
        return $this;
    }
    // ... single key handling
    return $this;
}
```

#### `bool|int` — Flag Values

```php
// vendor/magento/module-store/Model/Store.php
public function setId(int $id): bool|int;
```

#### `false` Pseudo-Type

PHP 8.0 introduced the `false` pseudo-type, often used for failure conditions:

```php
// vendor/magento/module-directory/Model/Currency.php
public function getRate(string $from, string $to): false|float
{
    if (!isset($this->getCurrencyCodes()[$from])) {
        return false;
    }
    return $this->rates[$from][$to] ?? 1.0;
}
```

### Nullable Union with `?`

The `?` prefix makes only the last type nullable:

```php
?string|null   // string OR null (same as ?string)
string|null?   // INVALID - ? can only prefix the union
```

### Intersection Types vs Union Types

PHP 8.1 introduced intersection types. Know the difference:

```php
// Union - must be one of these types
public function process(int|string $input): ProductInterface|CategoryInterface {}

// Intersection - must be all of these types (rare in Magento)
public function process(\Countable&\Iterator $input): void {}
```

### Writing Compatible Signatures

When writing Magento modules compatible with PHP 8.2/8.3:

```php
// APPROPRIATE - union types for genuinely flexible inputs
public function findEntities(int|array $ids): array;

// INAPPROPRIATE - don't use union when a single type suffices
public function process(string|int $id) {}  // Pick one if possible
```

### Handling Union Types in Plugins

When writing `before` plugins for methods with union types:

```php
// Original method
public function getById(int|string $id): ProductInterface;

// Before plugin - must handle both types
public function beforeGetById(ProductRepositoryInterface $subject, int|string $id): array
{
    if (is_string($id) && str_starts_with($id, 'SKU-')) {
        $id = $this->skuResolver->resolve($id);
    }
    
    return [$id];  // Return as array for original method's parameters
}
```

---

## 4. Readonly Properties (PHP 8.1+) — Immutability Patterns

Readonly properties, introduced in PHP 8.1, can only be assigned once — during construction or via an initializer. This makes them ideal for value objects and DTOs in Magento.

### How Readonly Works Internally

When you mark a property readonly, PHP enforces single assignment by:

1. Setting the property's `IS_READONLY` flag in its structure
2. Allowing assignment only in constructors and initializers
3. Preventing modification via reflection after initialization

```php
class OrderId
{
    public function __construct(
        private readonly int $orderId
    ) {}
}
```

The `readonly` flag is checked at runtime, not compile time.

### `readonly` Class (PHP 8.2+)

PHP 8.2 introduced `readonly` classes, which make all properties readonly implicitly:

```php
readonly class AddressValue
{
    public function __construct(
        public string $street,
        public string $city,
        public string $postcode
    ) {}
}

// All properties are implicitly readonly
$address = new AddressValue('123 Main St', 'NYC', '10001');
$address->street = '456 Oak Ave';  // Error!
```

### Magento's Immutable Value Objects

Magento uses readonly for value objects that should not change after construction:

```php
// Pattern used in Magento for immutable data transfer
readonly class Price
{
    public function __construct(
        public readonly float $amount,
        public readonly string $currency
    ) {}

    public function toArray(): array
    {
        return [
            'amount' => $this->amount,
            'currency' => $this->currency,
        ];
    }
}
```

### The `#[Magento\Framework\DataObject\Immutable]` Attribute

Magento 2.4.x may use this internal attribute:

```php
#[\Attribute(\Attribute::TARGET_CLASS)]
class Immutable
{
    public function __construct() {}
}
```

Applied to mark classes as immutable:

```php
#[Immutable]
class ConfigurationValue
{
    public function __construct(
        public readonly string $key,
        public readonly mixed $value
    ) {}
}
```

### `__debuginfo()` with Readonly

When a class has readonly properties and you need custom debug output:

```php
readonly class SearchCriteria
{
    public function __construct(
        public readonly array $filters,
        public readonly int $pageSize
    ) {}

    public function __debugInfo(): ?array
    {
        // Return array representation for var_dump()
        return [
            'filters' => $this->filters,
            'pageSize' => $this->pageSize,
            'count' => count($this->filters),
        ];
    }
}
```

### Readonly in Magento's Service Contracts

Magento's generated service contracts may use readonly:

```php
// Generated proxy class example
class ProductProxy implements \Magento\Catalog\Api\ProductInterface
{
    private ProductInterface $subject;
    private bool $isInitialized = false;

    public function __construct(
        private \Magento\Framework\ObjectManagerInterface $objectManager,
        private string $instanceName = \Magento\Catalog\Model\Product::class
    ) {}

    private function getSubject(): ProductInterface
    {
        if (!$this->isInitialized) {
            $this->subject = $this->objectManager->create($this->instanceName);
            $this->isInitialized = true;
        }
        return $this->subject;
    }
}
```

### Initializer Promotion with Readonly

PHP 8.1+ allows combining constructor promotion with readonly:

```php
// Modern Magento pattern for value objects
readonly class CustomerGroup
{
    public function __construct(
        public readonly int $id,
        public readonly string $code,
        public readonly ?string $name = null,
        public readonly bool $isActive = true
    ) {}
}
```

### When Not to Use Readonly

Don't use readonly for entities that need updating:

```php
// WRONG - Order needs to be mutable
readonly class Order  // ERROR - won't work for entities!
{
    public function __construct(
        public readonly int $entityId,  // OK - set once
        public readonly string $status  // PROBLEM - orders change status!
    ) {}
}

// CORRECT - entities should not be readonly
class Order
{
    private int $entityId;
    private string $status;
    
    public function __construct(int $entityId)
    {
        $this->entityId = $entityId;
        $this->status = 'new';
    }
    
    public function getEntityId(): int { return $this->entityId; }
    public function getStatus(): string { return $this->status; }
}
```

---

## 5. Backed Enums — Type-Safe Constants

PHP 8.1 introduced native enums, replacing Magento's constant classes with type-safe alternatives. Magento 2.4.8 uses backed enums extensively for order states, payment statuses, and stock statuses.

### Backed Enum Internal Structure

When you declare:

```php
enum OrderStatus: string
{
    case Pending = 'pending';
    case Processing = 'processing';
    case Complete = 'complete';
}
```

PHP creates:
1. A class named `OrderStatus`
2. Instances: `OrderStatus::Pending`, `OrderStatus::Processing`, `OrderStatus::Complete`
3. Methods: `values()`, `valueOf(string $name)`

### The `BackedEnum` Interface

All backed enums implement `BackedEnum`:

```php
interface BackedEnum
{
    public function value(): int|string;
    public static function cases(): array;
    public static function from(int|string $value): static;
    public static function tryFrom(int|string $value): ?static;
}
```

### Magento's Use of Enums

#### Order States

```php
// vendor/magento/module-sales/Model/Order.php
// Note: Magento uses constants, not native enums in 2.4.8 core
// But the pattern shows what backed enums would look like:

enum OrderState: string
{
    case Pending = 'pending';
    case Processing = 'processing';
    case Complete = 'complete';
    case Closed = 'closed';
    case Canceled = 'canceled';
    case Holded = 'holded';
    case PaymentReview = 'payment_review';

    public function label(): string
    {
        return match($this) {
            self::Pending => __('Pending'),
            self::Processing => __('Processing'),
            // ...
        };
    }

    public function isVisible(): bool
    {
        return match($this) {
            self::Pending, self::PaymentReview => false,
            default => true,
        };
    }
}
```

#### Payment Statuses

```php
enum PaymentStatus: string
{
    case Pending = 'pending';
    case Authorized = 'authorized';
    case Captured = 'captured';
    case Declined = 'declined';
    case Refunded = 'refunded';
    case Voided = 'voided';

    public function canCapture(): bool
    {
        return $this === self::Authorized;
    }

    public function canRefund(): bool
    {
        return $this === self::Captured;
    }
}
```

#### Stock Statuses

```php
enum StockStatus: int
{
    case OutOfStock = 0;
    case InStock = 1;
    case OutOfStockQuality = 2;  // Quality check

    public function label(): string
    {
        return match($this) {
            self::OutOfStock => __('Out of Stock'),
            self::InStock => __('In Stock'),
            self::OutOfStockQuality => __('Out of Stock - Quality Check'),
        };
    }
}
```

### Using Enums in Magento Code

```php
public function processOrder(int $orderId, OrderState $state): void
{
    $order = $this->orderRepository->getById($orderId);
    
    // Type-safe comparison
    if ($order->getState() !== $state->value) {
        throw new LocalizedException(
            __('Order state must be %1', $state->label())
        );
    }
    
    match($state) {
        OrderState::Processing => $this->processHandler->handle($order),
        OrderState::Complete => $this->completeHandler->handle($order),
        OrderState::Canceled => $this->cancelHandler->handle($order),
        default => throw new LocalizedException(__('Unhandled state')),
    };
}
```

### UnitEnum Methods

Pure enums (without backed values) also have built-in methods:

```php
enum ShipmentType: string
{
    case Standard = 'standard';
    case Express = 'express';
    case Overnight = 'overnight';

    // Built-in method - returns all cases
    public static function cases(): array { /* ... */ }
}

enum Priority
{
    case Low;
    case Medium;
    case High;
    case Critical;

    // Pure enum without backing value
    public function weight(): int
    {
        return match($this) {
            self::Low => 1,
            self::Medium => 5,
            self::High => 10,
            self::Critical => 100,
        };
    }
}
```

### Converting Between Enums and Values

```php
// From value to enum
$status = OrderStatus::from('pending');        // Throws ValueError if not found
$status = OrderStatus::tryFrom('pending');    // Returns null if not found

// From enum to value
$value = $status->value;                      // 'pending'

// Getting all values
$allStatuses = OrderStatus::cases();           // [OrderStatus::Pending, ...]
```

### Backed Enum in Magento Repository Pattern

```php
class OrderRepository implements OrderRepositoryInterface
{
    public function save(OrderInterface $order, ?OrderState $state = null): OrderInterface
    {
        if ($state !== null) {
            // Type-safe - only valid states accepted
            $order->setState($state->value);
        }
        
        return $this->orderResource->save($order);
    }

    public function getByState(OrderState $state): array
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('state', $state->value)
            ->create();
            
        return $this->getList($searchCriteria)->getItems();
    }
}
```

---

## 6. Match Expressions — Advanced Patterns

The `match` expression, introduced in PHP 8.0, is more powerful than `switch` because it returns values, supports expressions, and uses strict comparison.

### Match vs Switch

```php
// Switch - statement, no return value
switch ($paymentStatus) {
    case 'pending':
        $label = 'Pending Payment';
        break;
    case 'paid':
        $label = 'Paid';
        break;
}
$message = "Order is $label";

// Match - expression, returns value
$label = match($paymentStatus) {
    'pending' => 'Pending Payment',
    'paid' => 'Paid',
};
$message = "Order is $label";
```

### Match with Multiple Conditions

```php
$httpStatus = match($statusCode) {
    200, 201, 202, 204 => 'Success',
    301, 302, 307, 308 => 'Redirect',
    400, 401, 403, 404, 422 => 'Client Error',
    500, 502, 503, 504 => 'Server Error',
    default => 'Unknown',
};
```

### Match with Ranges (PHP 8.3+)

```php
// In PHP 8.3+, you can use ranges
$grade = match(true) {
    $score >= 90 => 'A',
    $score >= 80 => 'B',
    $score >= 70 => 'C',
    $score >= 60 => 'D',
    default => 'F',
};
```

### Match in Magento Plugins

Using match inside a around plugin:

```php
class ProductPlugin
{
    public function aroundSave(
        ProductRepositoryInterface $subject,
        callable $proceed,
        ProductInterface $product
    ): ProductInterface {
        $originalState = $product->getStatus();
        
        $result = $proceed($product);
        
        // Use match to determine what to log
        $action = match($product->getStatus()) {
            Status::STATUS_ENABLED => 'activated',
            Status::STATUS_DISABLED => 'deactivated',
            default => "updated (was $originalState)",
        };
        
        $this->logger->info("Product {$product->getSku()} was $action");
        
        return $result;
    }
}
```

### Match with Non-Scalar Subjects (PHP 8.0)

```php
// Match against enum cases
enum Status { case Active, case Inactive, case Pending }

$state = Status::Active;

$label = match($state) {
    Status::Active => 'Currently Active',
    Status::Inactive => 'Not Active',
    Status::Pending => 'Awaiting Activation',
};
```

### Match and Exhaustiveness

PHP 8.0's match doesn't require a default, but it's good practice to include one:

```php
$label = match($status) {
    'new' => 'New Order',
    'processing' => 'Being Processed',
    'complete' => 'Completed',
    // No default - PHP won't warn if you miss a case!
};
```

For exhaustive matching with enums, use a default that throws:

```php
$label = match($status) {
    OrderStatus::Pending->value => 'Pending',
    OrderStatus::Processing->value => 'Processing',
    OrderStatus::Complete->value => 'Complete',
    default => throw new \LogicException("Unknown status: $status"),
};
```

### Using Match for Validation

```php
public function validatePaymentMethod(string $code): bool
{
    return match(true) {
        str_starts_with($code, 'cc_') => $this->validateCC($code),
        str_starts_with($code, 'paypal_') => $this->validatePayPal($code),
        str_starts_with($code, 'bank_') => $this->validateBank($code),
        default => false,
    };
}
```

---

## 7. Nullsafe Operator — Chain Patterns

The nullsafe operator (`?->`) short-circuits evaluation and returns `null` if any part of the chain is `null` — without throwing an error.

### Basic Nullsafe Behavior

```php
// Without nullsafe - notice the explicit checks
if ($session !== null) {
    $customer = $session->getCustomer();
    if ($customer !== null) {
        $address = $customer->getAddress();
        if ($address !== null) {
            $city = $address->getCity();
        }
    }
}

// With nullsafe - short-circuits automatically
$city = $session?->getCustomer()?->getAddress()?->getCity();  // null if any is null
```

### When Nullsafe Helps

```php
// Reading configuration with nested optional structures
$config = $this->config?->getSection('payment')?->getGroup('paypal')?->get('merchant_id');

// Chaining through lazy-loaded relationships
$customerName = $order?->getCustomer()?->getName();

// Optional callback results
$result = $callback?->execute($data);
```

### Nullsafe vs isset()

Use nullsafe when you want `null` propagation:

```php
// Nullsafe - returns null if any part is null
$value = $obj?->property?->subProperty;

// isset() - checks if variable exists and is not null
$value = isset($arr['key']) ? $arr['key'] : null;
$value = $arr['key'] ?? null;  // Equivalent to above
```

### When NOT to Use Nullsafe

Don't use nullsafe when you need to distinguish between "not set" and "set to null":

```php
// PROBLEM - both return null, can't tell which case
$result1 = $config?->get('timeout');        // null if config or timeout is null
$result2 = $config?->get('optional_setting'); // also null

// BETTER - use explicit null check when distinction matters
if ($config !== null) {
    $result1 = $config->get('timeout');  // timeout might legitimately be null
    if (array_key_exists('optional_setting', $config->toArray())) {
        $result2 = $config->get('optional_setting'); // explicitly set to null
    }
}
```

### Nullsafe in Magento Templates

In Magento PHTML templates:

```php
<?php
// Instead of long if-else chains
<?= $block->getCustomer()?->getName() ?>
<?= $block->getOrder()?->getIncrementId() ?>

// Chaining with escaping
<?= $escaper->escapeHtml($block->getCustomer()?->getName()) ?>
```

### Combining with Null Coalescing

```php
// Nullsafe with coalescing for defaults
$customerName = $block->getCustomer()?->getName() ?? 'Guest';
$street = $address?->getStreet() ?? [];
$city = $address?->getCity() ?? 'Unknown';
```

### Nullsafe Limitations

Nullsafe cannot be used for assignment:

```php
// INVALID
$object?->property = 'value';  // Parse error!

// VALID - but doesn't short-circuit assignment
if ($object !== null) {
    $object->property = 'value';
}

// VALID - set default value then use nullsafe
$object ??= new stdClass();
$object->property = 'value';
```

---

## 8. Constructor Property Promotion — DI Patterns

Constructor property promotion, introduced in PHP 8.0, allows combining property declaration and assignment in the constructor parameter list. Magento uses this extensively for dependency injection.

### Traditional vs Promoted Constructors

```php
// TRADITIONAL - three declarations + three assignments
class ProductRepository
{
    private ProductFactory $productFactory;
    private SearchCriteriaBuilder $criteriaBuilder;
    private LoggerInterface $logger;

    public function __construct(
        ProductFactory $productFactory,
        SearchCriteriaBuilder $criteriaBuilder,
        LoggerInterface $logger
    ) {
        $this->productFactory = $productFactory;
        $this->criteriaBuilder = $criteriaBuilder;
        $this->logger = $logger;
    }
}

// PROMOTED - single declaration per parameter
class ProductRepository
{
    public function __construct(
        private ProductFactory $productFactory,
        private SearchCriteriaBuilder $criteriaBuilder,
        private LoggerInterface $logger
    ) {}
}
```

### Visibility in Promoted Parameters

```php
// public - accessible from outside
public function __construct(public ProductFactory $factory) {}

// protected - accessible in class and subclasses
protected function __construct(protected LoggerInterface $logger) {}

// private - accessible only in this class (most common for DI)
private function __construct(private ConfigInterface $config) {}
```

### Combining with Readonly

```php
class OrderId
{
    public function __construct(
        private readonly int $orderId,
        private readonly int $storeId
    ) {}

    public function getOrderId(): int { return $this->orderId; }
    public function getStoreId(): int { return $this->storeId; }
}
```

### Default Values with Promotion

```php
class CacheService
{
    public function __construct(
        private string $cachePrefix = 'mage_',
        private int $ttl = 3600,
        private ?LoggerInterface $logger = null
    ) {}
}
```

### Promoted Parameters with DocBlocks

When using promoted parameters, you can still add DocBlocks for additional documentation:

```php
class CustomerSession
{
    /**
     * Constructor
     *
     * @param CustomerRepositoryInterface $customerRepository Customer repository
     * @param SessionManager $sessionManager Magento session manager
     * @param int $customerId Customer ID (null for guests)
     */
    public function __construct(
        #[Injection('customer_repository')]
        private CustomerRepositoryInterface $customerRepository,
        private SessionManager $sessionManager,
        private ?int $customerId = null
    ) {}
}
```

### Magento DI with Constructor Promotion

Magento's generated code (factories, proxies) uses promotion:

```php
// Generated ProductFactory
class ProductFactory
{
    /**
     * @var ObjectManagerInterface
     */
    private $objectManager;

    public function __construct(ObjectManagerInterface $objectManager)
    {
        $this->objectManager = $objectManager;
    }

    public function create(array $data = []): ProductInterface
    {
        $instance = $this->objectManager->create(ProductInterface::class, $data);
        // ...
        return $instance;
    }
}
```

### Complex Patterns — Multiple Constructor Calls

When a class needs to call multiple parent constructors:

```php
class CompositeProcessor
{
    public function __construct(
        private ProcessorInterface $processor1,
        private ProcessorInterface $processor2,
        private ?ProcessorInterface $fallbackProcessor = null
    ) {}

    public function process(mixed $data): Result
    {
        try {
            return $this->processor1->process($data);
        } catch (\Exception $e) {
            try {
                return $this->processor2->process($data);
            } catch (\Exception $e) {
                if ($this->fallbackProcessor) {
                    return $this->fallbackProcessor->process($data);
                }
                throw $e;
            }
        }
    }
}
```

### Testing with Promoted Properties

Promoted properties can be accessed in tests via `$this`:

```php
class ProductRepositoryTest extends TestCase
{
    public function testGetById(): void
    {
        $factory = $this->createMock(ProductFactory::class);
        $builder = $this->createMock(SearchCriteriaBuilder::class);
        
        $repository = new class($factory, $builder) extends ProductRepository
        {
            public function __construct(
                public ProductFactory $productFactory,
                public SearchCriteriaBuilder $criteriaBuilder
            ) {}
            
            // Expose parent constructor for testing
            public function __construct(
                ProductFactory $productFactory,
                SearchCriteriaBuilder $criteriaBuilder
            ) {
                parent::__construct($productFactory, $criteriaBuilder);
            }
        };
        
        $this->assertSame($factory, $repository->productFactory);
        $this->assertSame($builder, $repository->criteriaBuilder);
    }
}
```

---

## 9. First-Class Callable Syntax — Modern Patterns

PHP 8.0 introduced first-class callable syntax as a cleaner alternative to `Closure::fromCallable()`. The syntax `$object->method(...)` creates a callable that passes through arguments when invoked.

### Basic First-Class Callable

```php
class EventDispatcher
{
    public function dispatch(string $event, array $data): void
    {
        // dispatch logic
    }
}

$dispatcher = new EventDispatcher();

// ARRAY CALLABLE syntax (legacy)
$callback = [$dispatcher, 'dispatch'];  // callable but not typed

// FIRST-CLASS CALLABLE syntax (PHP 8.0+)
$callback = $dispatcher->dispatch(...);  // callable with correct type hints preserved
```

### Magento Plugin System Usage

```php
// In Magento\Framework\Interception\Interceptor
class ProductRepositoryInterceptor extends ProductRepository
{
    private Closure $___pluginBeforeSave;

    public function __construct(
        SubjectLocatorInterface $subjectLocator,
        MethodLocatorInterface $methodLocator,
        PluginListInterface $pluginList,
        // ...
    ) {
        parent::__construct(/* ... */);
        
        // FIRST-CLASS CALLABLE - cleaner than Closure::fromCallable
        $this->___pluginBeforeSave = $this->beforeSave(...);
    }
}
```

### Callable Type Preservation

First-class callables preserve type hints:

```php
function processCustomer(CustomerInterface $customer): string
{
    return $customer->getName();
}

$customer = $this->customerRepository->getById(1);

// Array callable - loses type info
$callback = [$this, 'processCustomer'];  // callable, but PHP doesn't know signature

// First-class callable - preserves signature
$callback = $this->processCustomer(...);  // callable with exact signature

// This means IDEs and static analyzers can properly type-check
$callback($customer);  // PHP knows customer must be CustomerInterface
```

### Use in Array Functions

```php
class OrderCollection
{
    /** @var OrderInterface[] */
    private array $orders = [];

    public function filterByStatus(string $status): OrderCollection
    {
        $filtered = array_filter(
            $this->orders,
            fn(OrderInterface $order) => $order->getStatus() === $status  // Already a callable
        );
        
        return new self($filtered);
    }

    public function mapToArray(callable $transformer): array
    {
        return array_map($transformer, $this->orders);
    }
}

// Usage
$collection = new OrderCollection($orders);

// Inline closure
$names = $collection->mapToArray(fn($o) => $o->getCustomerName());

// Method reference as first-class callable
$names = $collection->mapToArray($this->extractCustomerName(...));
```

### Callable vs Arrow Functions

Arrow functions (`fn`) are limited — they capture variables by value and can only contain a single expression:

```php
// Arrow function - limited to single expression, auto-captures $this
$getName = fn(CustomerInterface $c) => $c->getName();

// Anonymous function - full PHP, explicit $this
$getName = function (CustomerInterface $c) use ($this) {
    return $this->formatter->format($c->getName());
};

// Method reference (first-class callable)
$getName = $this->formatter->format(...);  // callable when invoked
```

### Static Method References

```php
class StatusMapper
{
    public static function toLabel(string $status): string
    {
        return match($status) {
            'pending' => 'Pending',
            'complete' => 'Complete',
            default => 'Unknown',
        };
    }
}

// Static method as first-class callable
$mapper = new StatusMapper();
$toLabel = $mapper->toLabel(...);  // Works! Callable that calls static method

// Or directly
$toLabel = StatusMapper::toLabel(...);
```

---

## 10. The `mixed` Pseudo-Type — Strict Typing

PHP 8.0 introduced the `mixed` pseudo-type, which represents "any type". Understanding when to use `mixed` vs union types is crucial for writing well-typed Magento code.

### What `mixed` Means

```php
// These are equivalent:
function process(mixed $value): mixed
{
    // Can accept and return any type
}

function process($value)
{
    // Old way - no type information
}
```

### `mixed` Internal Representation

`mixed` is represented internally as a union of all types:

```php
// mixed === int|float|string|bool|array|object|null|resource
```

### When to Use `mixed`

```php
// Appropriate - truly any type possible
public function logValue(mixed $value): void
{
    $type = gettype($value);
    $this->logger->debug("Value of type $type", ['value' => $value]);
}

// Appropriate - processing varied input
public function normalizeInput(mixed $input): array
{
    if (is_array($input)) {
        return $input;
    }
    if ($input instanceof \Traversable) {
        return iterator_to_array($input);
    }
    return [(string) $input];
}
```

### When NOT to Use `mixed`

```php
// INAPPROPRIATE - should be specific union
public function setQuantity(mixed $qty): void  // BAD - what types are valid?
{
    $this->quantity = (int) $qty;  // Hides type error!
}

// APPROPRIATE - specific union
public function setQuantity(int|float $qty): void  // GOOD - clear what types accepted
{
    $this->quantity = $qty;
}

// INAPPROPRIATE - for configuration that expects specific structure
public function setConfig(mixed $config): void  // BAD - no type safety
{
    $this->config = $config;
}

// APPROPRIATE - use array shape or specific type
public function setConfig(array $config): void  // BETTER - at minimum, array
{
    $this->config = $config;
}
```

### `mixed` vs Union with `?`

```php
// ?string - string OR null
?string $name;   // Can be string, can be null

// mixed - absolutely any type (including null)
mixed $value;    // Can be anything

// When to use each:
?string $name = $customer?->getName();    // Name can be absent - use nullable
mixed $value = $this->getDynamicValue(); // Truly unknown type - use mixed
```

### Handling `mixed` in Magento

```php
class DataObject extends \Magento\Framework\DataObject
{
    public function setData(mixed $key, mixed $value = null): self
    {
        // mixed for flexible key/value storage
        if (is_array($key)) {
            foreach ($key as $k => $v) {
                $this->setData($k, $v);
            }
            return $this;
        }
        
        $this->data[$key] = $value;
        return $this;
    }

    public function getData(mixed $key = null): mixed
    {
        // mixed return - could be anything or array of everything
        if ($key === null) {
            return $this->data;
        }
        
        return $this->data[$key] ?? null;
    }
}
```

### Migrating from DocBlocks to `mixed`

```php
// BEFORE - PHPDoc only
/**
 * @param mixed $value
 * @return mixed
 */
public function process($value) { }

// AFTER - native type with nullable union
public function process(mixed $value): mixed { }

// EVEN BETTER - specific union if possible
public function process(int|string|array $value): int|array { }
```

---

## 11. New Functions in PHP 8 — Modern Alternatives

PHP 8 introduced several new string and utility functions that replace common patterns in Magento code.

### `str_contains()` — Replacing `strpos()`

```php
// OLD WAY - confusing strpos return value
if (strpos($sku, 'SKU-') !== false) {  // Note: !== false, not === true!
    // Contains substring
}

// PHP 8.0+ - clear intent
if (str_contains($sku, 'SKU-')) {
    // Contains substring
}
```

Magento example:

```php
// Before
if (strpos($product->getSku(), '_bundle') !== false) {
    $type = 'bundle';
}

// After (PHP 8.0+)
if (str_contains($product->getSku(), '_bundle')) {
    $type = 'bundle';
}
```

### `str_starts_with()` — Head Check

```php
// OLD WAY
if (strpos($code, 'cc_') === 0) {  // Check position 0
    $type = 'credit_card';
}

// PHP 8.0+ - clear intent
if (str_starts_with($code, 'cc_')) {
    $type = 'credit_card';
}
```

Magento usage:

```php
public function getPaymentMethodType(string $code): string
{
    if (str_starts_with($code, 'paypal_')) {
        return 'paypal';
    }
    if (str_starts_with($code, 'cc_')) {
        return 'credit_card';
    }
    if (str_starts_with($code, 'bank_')) {
        return 'bank_transfer';
    }
    return 'unknown';
}
```

### `str_ends_with()` — Tail Check

```php
// OLD WAY
if (substr($filename, -5) === '.html') {
    $type = 'html';
}

// PHP 8.0+ - clear intent
if (str_ends_with($filename, '.html')) {
    $type = 'html';
}
```

Magento usage:

```php
public function isEmailTemplate(string $path): bool
{
    return str_ends_with($path, '.email.twig') 
        || str_ends_with($path, '.email.phtml');
}
```

### `fdiv()` — Safe Float Division

```php
// OLD WAY - division by zero returns inf, may cause issues
$result = $price / $quantity;  // Returns INF if quantity is 0

// PHP 8.0+ - fdiv is faster and returns INF (IEEE 754 compliant)
$result = fdiv($price, $quantity);  // Returns INF, NAN, or finite value
```

Use case in pricing calculations:

```php
public function calculateAveragePrice(array $prices): float
{
    $count = count($prices);
    if ($count === 0) {
        return 0.0;
    }
    
    // fdiv is faster than manual check
    return fdiv(array_sum($prices), $count);
}
```

### `get_resource_id()` — Resource to ID

```php
// OLD WAY - casting resource to int
$id = (int) $connection;  // Magic conversion

// PHP 8.0+ - explicit function
$id = get_resource_id($connection);  // Clear intent
```

### Other PHP 8.0+ Functions

```php
// get_debug_type() - better type reporting
get_debug_type($var);  // Returns 'int', 'string', 'null', 'Magento\Framework\DataObject', etc.

// array_is_list() - check if array is sequential list
array_is_list([1, 2, 3]);    // true
array_is_list(['a' => 1]);   // false

// Key existence with array access
$array = ['a' => 1, 'b' => 2];
isset($array['c']);              // false
array_key_exists('c', $array);  // true (isset returns false for null values)
```

### PHP 8.1+ New Functions

```php
// array_find(), array_find_key() - search arrays
$found = array_find([1, 2, 3], fn($v) => $v > 1);  // 2

// array_any(), array_all() - check conditions
$hasActive = array_any($products, fn($p) => $p->isActive());
$allEnabled = array_all($products, fn($p) => $p->isEnabled());

// fsync() - flush changes to disk
fsync($fileHandle);  // Ensure data is on disk
```

### PHP 8.2+ New Functions

```php
// get_stateless() - get PHP configuration
$config = get_stateless('session');  // Returns session config array

// gc_status() - garbage collector stats
$gcStatus = gc_status();
// ['collected' => 100, 'runs' => 5, 'running' => false]
```

---

## 12. Fibers and Generators — Async Patterns

PHP 8.1 introduced Fibers, providing the foundation for async patterns. Understanding generators and fibers helps when working with Magento's batch processing and iterative operations.

### Generator Basics

Generators use `yield` to produce values lazily:

```php
function generateOrders(int $count): \Generator
{
    for ($i = 0; $i < $count; $i++) {
        yield $this->orderRepository->getById($i);
    }
}

// Usage - lazy iteration
foreach (generateOrders(1000) as $order) {
    $this->process($order);  // Only loads one at a time
}
```

### Generator Delegation with `yield from`

```php
// Child generators
function generateSkus(): \Generator
{
    yield 'SKU-001';
    yield 'SKU-002';
    yield 'SKU-003';
}

function generateNames(): \Generator
{
    yield 'Product A';
    yield 'Product B';
    yield 'Product C';
}

// DELEGATION - yield from passes control to another generator
function generateAll(): \Generator
{
    yield from generateSkus();
    yield from generateNames();
}

// Equivalent to:
function generateAll(): \Generator
{
    foreach (generateSkus() as $sku) {
        yield $sku;
    }
    foreach (generateNames() as $name) {
        yield $name;
    }
}
```

### Return Values in Generators

PHP 7.0+ generators can return values:

```php
function countOrders(): \Generator
{
    $count = 0;
    foreach ($this->orderRepository->getAll() as $order) {
        yield $order;
        $count++;
    }
    return $count;  // Return value accessible via ->getReturn()
}

$generator = countOrders();
foreach ($generator as $order) {
    // process
}
$totalCount = $generator->getReturn();  // Total orders processed
```

### Generators in Magento

Magento uses generators for memory-efficient processing:

```php
// vendor/magento/module-eav/Model/Entity/AttributeLoader.php
public function loadAllAttributes($entityType): \Generator
{
    foreach ($this->getEntityTypeAttributes($entityType) as $attributeCode => $attributeConfig) {
        yield $this->createAttribute($attributeCode, $attributeConfig);
    }
}

// Memory-efficient - doesn't load all attributes at once
foreach ($loader->loadAllAttributes('catalog_product') as $attribute) {
    $this->processAttribute($attribute);
}
```

### Fibers — The Async Foundation

Fibers provide controlled execution interruption:

```php
$fiber = new Fiber(function (): void {
    $value = Fiber::suspend('suspended with value');
    echo "Resumed with: $value\n";
});

$value = $fiber->start();  // Starts fiber, suspends immediately
echo "Suspended with: $value\n";  // $value = 'suspended with value'

$fiber->resume(' resumed value');  // Continues fiber from suspension point
```

### Why Fibers Matter for Magento

Fibers enable async I/O patterns without callbacks:

```php
// Hypothetical async HTTP client using fibers
class AsyncHttpClient
{
    public function get(string $url): Fiber
    {
        return new Fiber(function () use ($url) {
            $socket = $this->createSocket($url);
            $response = Fiber::suspend(['socket' => $socket]);
            return $this->parseResponse($response);
        });
    }
}

// Usage without blocking
$fiber = $this->httpClient->get('https://api.example.com/data');
// Do other work while waiting
$result = $fiber->resume();  // Get result when ready
```

### Generator vs Fiber Comparison

| Feature | Generator | Fiber |
|---------|-----------|-------|
| Introduced | PHP 5.5 | PHP 8.1 |
| Purpose | Lazy iteration | Controlled suspension |
| Can suspend | Yes (implicitly with yield) | Yes (explicit with suspend) |
| Can resume from same point | No | Yes |
| Use case | Large datasets, lazy loading | Async I/O, coroutines |
| In Magento | Batch processing, iterators | Future async patterns |

---

## 13. PHP 8 Performance Improvements — Magento Impact

PHP 8.x brings significant performance improvements that directly affect Magento 2.4.8 runtime behavior.

### JIT Compiler Impact

The Just-In-Time (JIT) compiler, introduced in PHP 8.0, compiles PHP bytecode to machine code at runtime.

#### JIT Configuration for Magento

```php
; php.ini for Magento with JIT
opcache.enable=1
opcache.memory_consumption=512
opcache.max_accelerated_files=100000
opcache.jit_buffer_size=100M
opcache.jit=tracing  ; or 'function' for different strategies
```

JIT effectiveness varies by workload:

| Workload Type | JIT Benefit | Recommended JIT Mode |
|---------------|-------------|---------------------|
| Web requests (short-lived) | Moderate | `function` |
| Long-running CLI scripts | High | `tracing` |
| CPU-intensive calculations | Very High | `tracing` |
| I/O-bound operations | Low | `disable` |

#### Magento CLI Scripts and JIT

```bash
# Run indexer with JIT enabled
php -d opcache.jit=tracing -d opcache.jit_buffer_size=100M bin/magento indexer:reindex
```

### Real-World Benchmark Context

Based on community benchmarks for Magento 2.4.x on PHP 8.2:

| Operation | PHP 7.4 | PHP 8.1 | PHP 8.2 | Improvement |
|-----------|---------|---------|---------|-------------|
| Page Load (cached) | 100ms | 85ms | 80ms | ~20% faster |
| Page Load (uncached) | 800ms | 650ms | 600ms | ~25% faster |
| Checkout (warm) | 1.2s | 950ms | 900ms | ~25% faster |
| Indexer (full) | 180s | 140s | 130s | ~28% faster |

*Numbers are representative of typical enterprise Magento deployments*

### Key Performance Features by Version

#### PHP 8.0 Improvements
- **JIT Compiler**: 2-3x speedup for CPU-bound operations
- **Named Arguments**: No runtime overhead (compile-time only)
- **Match Expressions**: Slightly faster than switch due to hash lookup
- **Array Unpacking with String Keys**: Optimized

```php
// PHP 8.0 optimized array merge
$config = [...$baseConfig, ...$overrideConfig];  // Faster than array_merge()
```

#### PHP 8.1 Improvements
- **Readonly Properties**: Minimal overhead, improved memory efficiency
- **Enumerations**: Faster than constant classes due to internal optimization
- **First-class Callable Syntax**: Slight performance improvement over array callables
- **Fibers**: No overhead when not used

```php
// PHP 8.1 enum is faster than class constants
enum OrderStatus: string {
    case Pending = 'pending';
}
// vs
class OrderStatus {
    public const STATE_PENDING = 'pending';
}
```

#### PHP 8.2 Improvements
- **Readonly Classes**: No additional overhead over readonly properties
- **Disjunctive Normal Form Types**: Better optimization opportunities
- **Constants in Traits**: Same performance as regular constants

```php
// PHP 8.2 readonly class - optimized by engine
readonly class Point {
    public function __construct(
        public float $x,
        public float $y
    ) {}
}
```

#### PHP 8.3 Improvements
- **JIT Optimization**: Better inlining decisions
- **Readonly amendments**: Improved readonly handling
- **Typed class constants**: No runtime overhead

### Memory Usage Improvements

PHP 8.x reduces memory usage through:

1. **Better String Interning**: Repeated strings share memory
2. **Optimized Property Access**: Reduced memory per object
3. **Improved Array Handling**: Less memory for dense arrays

```php
// Magento optimization: object pooling benefits
class ObjectPool
{
    /** @var SplObjectStorage */
    private $pool;

    public function __construct()
    {
        $this->pool = new SplObjectStorage();
    }

    public function get(string $class): object
    {
        if (!isset($this->pool[$class])) {
            $this->pool[$class] = new $class();
        }
        return $this->pool[$class];
    }
}
```

### Opcache Settings for Magento

Recommended production settings:

```ini
; opcache.ini for Magento 2.4.8 on PHP 8.2+
opcache.enable=1
opcache.memory_consumption=1024
opcache.max_accelerated_files=60000
opcache.validate_timestamps=0
opcache.save_comments=1
opcache.enable_file_override=1
opcache.optimization_level=0x7FFFBFFF
opcache.preload=/var/www/html/preload.php
```

### Preloading for Magento (PHP 7.4+/8.0+)

Preloading loads specified files into opcache at startup:

```php
// preload.php - included in opcache.preload
$files = [
    '/var/www/html/vendor/magento/framework/Registration.php',
    '/var/www/html/vendor/magento/framework/ObjectManager/ConfigLoader.php',
    // ... key framework files
];

foreach ($files as $file) {
    if (file_exists($file)) {
        opcache_compile_file($file);
    }
}
```

Preloading benefits:
- Framework classes loaded once at startup
- No per-request compilation
- Reduced memory footprint
- 5-15% improvement on first request after restart

---

## See Also

- [01 - PHP 8 Compatibility](/home/letiendat1002/workspace/personal/courses/magento2/v2.4.8/01-dev-environment/01-php8-compatibility.md) — Surface-level overview of PHP 8 features for Magento compatibility
- [Magento 2 Architecture](https://developer.adobe.com/commerce/php/architecture/) — Directory structure and module layout
- [Magento 2 Development](https://developer.adobe.com/commerce/php/development/) — Dependency injection and service contracts
- [PHP 8.3 Documentation](https://www.php.net/manual/en/migration83.php) — Official migration guide
- [PSR-12 Coding Standard](https://www.php-fig.org/psr/psr-12/) — PHP formatting rules Magento enforces
