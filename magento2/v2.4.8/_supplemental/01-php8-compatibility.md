---
title: "01 - PHP 8 Compatibility"
description: "Deep dive into Magento 2.4.8 PHP 8.3 compatibility: named arguments, attributes, enums, never return type, union types, match expressions, and readonly properties"
tags:
  - magento2
  - php8
  - php83
  - attributes
  - type-system
rank: 1
pathways:
  - magento2-deep-dive
---

# PHP 8 Compatibility

Magento 2.4.8 requires PHP 8.2 or 8.3. This article covers the PHP 8 features that matter most for Magento development, with correct usage patterns and common pitfalls.

---

## 1. Named Arguments

Named arguments allow you to pass arguments by name rather than position, improving readability and allowing partial argument updates.

### Basic Usage

```php
// Instead of positional
$objectManager->create(Product::class, ['sku' => 'SKU123', 'name' => 'Product Name']);

// Use named arguments for clarity
$objectManager->create(Product::class, [
    'sku' => 'SKU123',
    'name' => 'Product Name',
]);
```

### With Magento Options Array

```php
// Legacy positional - unclear what each value means
$layout->addBlock(Block::class, 'block_name', 'parent', 'template', 'alias');

// Named arguments - self-documenting
$layout->addBlock(
    blockClass: Block::class,
    name: 'block_name',
    parent: 'parent',
    template: 'template',
    alias: 'alias'
);
```

### Benefit for Configuration Arrays

```php
// Without named arguments - what does 'true' mean?
$cache->save($data, 'cache_key', ['tags'], true);

// With named arguments - clear intent
$cache->save(
    data: $data,
    key: 'cache_key',
    tags: ['tags'],
    isPersistent: true
);
```

---

## 2. Attributes (PHP 8.0+)

Attributes replace PHPDoc annotations with native PHP syntax. Magento 2.4.8 supports most PHP 8 attributes.

### `#[Deprecated]` (PHP 8.1+)

Mark methods as deprecated with the `#[Deprecated]` attribute:

```php
#[Deprecated(message: 'Use newMethod() instead', since: '2.0.0')]
public function oldMethod(): string
{
    return 'legacy behavior';
}
```

When the method is called, PHP will emit a deprecation warning with the message.

### `#[ReturnTypeWillChange]` (Internal PHP Attribute)

This attribute is added automatically by PHP when overriding methods with different return types. You don't typically use it directly—it's an internal marker.

```php
// PHP automatically adds this when you change return types in inheritance
class Child extends Parent
{
    #[ReturnTypeWillChange]
    public function getValue() {
        return 'value';
    }
}
```

### `#[Attribute]` (PHP 8.0+)

Create custom attributes for annotations:

```php
// Define a custom attribute
#[Attribute(Attribute::TARGET_CLASS | Attribute::TARGET_METHOD)]
class MagentoAction
{
    public function __construct(
        public readonly string $route,
        public readonly string $aclResource
    ) {}
}

// Use the attribute
#[MagentoAction(route: 'admin_system_config', aclResource: 'config')]
class SystemConfigurationAction
{
}
```

### `#[Override]` (PHP 8.3+)

Mark methods that override parent methods:

```php
class ProductRepository
{
    public function getById(int $id): ProductInterface
    {
        // implementation
    }
}

class CustomProductRepository extends ProductRepository
{
    #[Override]
    public function getById(int $id): ProductInterface
    {
        // This explicitly overrides the parent method
        // Compile error if parent doesn't have this method
    }
}
```

---

## 3. Enums (PHP 8.1+)

Enums provide type-safe enumerations, replacing integer/string constants.

### Basic Enum

```php
enum OrderStatus: string
{
    case Pending = 'pending';
    case Processing = 'processing';
    case Complete = 'complete';
    case Cancelled = 'cancelled';

    public function label(): string
    {
        return match($this) {
            self::Pending => 'Pending Payment',
            self::Processing => 'Processing',
            self::Complete => 'Complete',
            self::Cancelled => 'Cancelled',
        };
    }
}
```

### Backed Enum in Magento

```php
// Using backed enums for status fields
enum CustomerGroup: int
{
    case NOT_LOGGED_IN = 0;
    case GENERAL = 1;
    case WHOLESALE = 2;
    case RETAILER = 3;

    public function label(): string
    {
        return match($this) {
            self::NOT_LOGGED_IN => 'NOT LOGGED IN',
            self::GENERAL => 'General',
            self::WHOLESALE => 'Wholesale',
            self::RETAILER => 'Retailer',
        };
    }
}

// In a class
public function getCustomerGroupLabel(CustomerGroup $group): string
{
    return $group->label();
}

// Comparing enum values
public function isWholesale(CustomerGroup $group): bool
{
    return $group === CustomerGroup::WHOLESALE;
}
```

### Enum with Methods

```php
enum ShipmentState: string
{
    case LabelGenerated = 'label_generated';
    case Shipped = 'shipped';
    case Delivered = 'delivered';
    case Returned = 'returned';

    public function canCancel(): bool
    {
        return match($this) {
            self::LabelGenerated => true,
            self::Shipped, self::Delivered, self::Returned => false,
        };
    }

    public function canTrack(): bool
    {
        return match($this) {
            self::Shipped, self::Delivered, self::Returned => true,
            self::LabelGenerated => false,
        };
    }
}
```

---

## 4. Never Return Type (PHP 8.1+)

The `never` return type indicates a function that **never returns normally**. It must either throw an exception or terminate execution with `exit`/`die`.

### Correct Usage - Throwing Exceptions

```php
// CORRECT - never returns, always throws
public function handleError(string $message): never
{
    throw new \RuntimeException($message);
}

// CORRECT - never returns, throws after logging
public function assertValidState(bool $condition): never
{
    if (!$condition) {
        throw new \LogicException('Invalid state');
    }
    throw new \RuntimeException('Assertion failed');
}
```

### Correct Usage - Exit/Die

```php
// CORRECT - never returns, calls exit
public function bootstrap(): never
{
    $this->init();
    exit(1);
}

// CORRECT - fatal error handler never returns
public function handleFatalError(): never
{
    error_log('Fatal error occurred');
    flush();
    exit(128);
}
```

### Why `never` with `return` is a COMPILE ERROR

```php
// WRONG - This will NOT compile!
public function handleError(): never
{
    return "something";  // ERROR: cannot return value from never-returning function
}
```

PHP will refuse to compile this because declaring `never` means "this function will never return"—so returning a value is logically impossible.

### When to Use `never`

Use `never` for:
- Error handlers that always throw
- Bootstrap functions that exit
- Redirect functions that never return
- Infinite loops that manage their own termination via exit

---

## 5. Union Types (PHP 8.0+)

Union types allow specifying multiple types for a parameter or return value.

### Syntax

```php
// Union of types
public function process(int|float $value): int|string;

// Nullable union
public function findCustomer(int|string $id): ?CustomerInterface;

// Mixed union (any type)
public function logValue(mixed $value): void
```

### In Magento Context

```php
// Multiple return types from a service
public function getIdentifier(): int|string
{
    if ($this->useId) {
        return $this->id;
    }
    return $this->sku;
}

// Nullable entity
public function findBySku(string $sku): ?ProductInterface
{
    // return null if not found
}

// Void with union (rare but valid)
public function updateStatus(int|string $identifier, int $status): void
{
    // implementation
}
```

### `false` Pseudo-Type (PHP 8.0+)

Often used for operations that can fail:

```php
// Returns either the object or false on failure
public function getConnection(): \PDO|false
{
    if (!$this->connected) {
        return false;
    }
    return $this->pdo;
}

// Combining with nullable
public function findRecord(int|string $id): ?Record|false
{
    // Returns null if not found, false on error
}
```

---

## 6. Match Expression (PHP 8.0+)

Match is a more powerful alternative to switch with expression support and return values.

### Basic Match

```php
// Switch version
switch ($status) {
    case OrderStatus::PENDING:
        $label = 'Pending';
        break;
    case OrderStatus::PROCESSING:
        $label = 'Processing';
        break;
    default:
        $label = 'Unknown';
}

// Match version - cleaner and returns a value
$label = match($status) {
    OrderStatus::PENDING => 'Pending',
    OrderStatus::PROCESSING => 'Processing',
    default => 'Unknown',
};
```

### Complex Conditions

```php
$result = match(true) {
    $score >= 90 => 'Excellent',
    $score >= 80 => 'Good',
    $score >= 70 => 'Fair',
    $score >= 60 => 'Pass',
    default => 'Fail',
};
```

### Multiple Conditions

```php
$type = match($code) {
    200, 201, 202 => 'Success',
    400, 422 => 'Client Error',
    500, 503 => 'Server Error',
    default => 'Unknown',
};
```

---

## 7. Readonly Properties (PHP 8.1+)

Readonly properties can only be initialized once and cannot be modified afterward.

### Basic Readonly

```php
class Config
{
    public function __construct(
        public readonly string $name,
        public readonly int $timeout = 30
    ) {}
}

// Usage
$config = new Config('api', timeout: 60);
echo $config->name;      // OK: 'api'
$config->name = 'new';   // ERROR: Cannot modify readonly property
```

### Readonly in Magento Services

```php
class OrderId
{
    public function __construct(
        private readonly int $orderId,
        private readonly int $storeId
    ) {}

    public function getOrderId(): int
    {
        return $this->orderId;
    }

    public function getStoreId(): int
    {
        return $this->storeId;
    }
}
```

### Readonly Constructor Promotion (PHP 8.1+)

```php
// Combine constructor promotion with readonly
class ApiResponse
{
    public function __construct(
        public readonly string $status,
        public readonly array $data,
        public readonly ?string $error = null
    ) {}
}
```

### Readonly with Initializer (PHP 8.2+)

```php
class Configuration
{
    // Can initialize inline or in constructor
    public readonly string $version;
    public readonly ?string $environment;

    public function __construct(?string $env = null)
    {
        $this->version = '2.4.8';
        $this->environment = $env ?? 'production';
    }
}
```

---

## 8. First-Class Collection Syntax (PHP 8.4+)

PHP 8.4 introduced first-class callable syntax for closures and arrow functions.

### First-Class Callables

```php
// Traditional closure
$callback = Closure::fromCallable([$this, 'process']);

// First-class callable (PHP 8.4+)
$callback = $this->process(...);
```

### In Magento Event Observer Context

```php
// Instead of
$observer->getEvent()->getData('order');

// With first-class callable
$getOrder = $observer->getEvent()->getData(...);
$order = $getOrder('order');
```

---

## 9. PHP 8.3 Specific Features

### Typed Class Constants (PHP 8.3+)

```php
interface PaymentInterface
{
    public const string PENDING = 'pending';
    public const string COMPLETE = 'complete';
    public const string FAILED = 'failed';
}
```

### #[\Deprecated] Attribute Improvements (PHP 8.3+)

```php
// More detailed deprecation with version and remove version
#[Deprecated(message: 'Use NewPaymentMethod instead', since: '2.4.0', removedIn: '3.0.0')]
public function processPayment(): bool
{
    // deprecated implementation
}
```

---

## 10. Common Compatibility Pitfalls

### 1. Mixed Type Comparisons

```php
// PHP 7/8 behavior difference
$mixed = '123';
$int = 123;

// Loose comparison - works same
if ($mixed == $int) { } // true in both

// Strict comparison - PHP 8 is more strict
if ($mixed === $int) { } // false in both PHP versions
```

### 2. String to Number Conversion

```php
// PHP 8 behavior change
$result = '10' + '20';  // 30 in both, but PHP 8 is more explicit
$result = '10.5' + '20.5'; // 31 in both
```

### 3. @ Operator and Warnings

```php
// @ operator no longer silences fatal errors in PHP 8.0+
// Always use proper error handling
try {
    $result = file_get_contents($nonexistent);
} catch (\Error $e) {
    // Handle properly
}
```

### 4. Nullsafe Operator

```php
// Chain nullsafe calls
$country = $session?->getCustomer()?->getAddress()?->getCountry();

// Equivalent to
if ($session !== null) {
    $customer = $session->getCustomer();
    if ($customer !== null) {
        $address = $customer->getAddress();
        if ($address !== null) {
            $country = $address->getCountry();
        }
    }
}
```

---

## 11. Summary of PHP 8 Changes by Version

| Version | Feature | Benefit |
|---------|---------|---------|
| 8.0 | Named Arguments | Cleaner function calls |
| 8.0 | Union Types | Type flexibility |
| 8.0 | Match Expression | Better switch |
| 8.0 | Attributes | Native annotations |
| 8.1 | Enums | Type-safe constants |
| 8.1 | Never Return Type | Compile-time safety |
| 8.1 | Readonly Properties | Immutability |
| 8.2 | Constants in Traits | Code reuse |
| 8.3 | Typed Class Constants | Type safety |
| 8.4 | First-Class Callables | Cleaner closures |

---

## 12. Migration Checklist

- [ ] Replace PHPDoc types with native type declarations
- [ ] Replace `@deprecated` annotations with `#[Deprecated]`
- [ ] Convert string constants to backed enums
- [ ] Add return types to all methods
- [ ] Add parameter types to all methods
- [ ] Replace dynamic properties with typed properties
- [ ] Update `@return` annotations for `never` returning methods
- [ ] Test with `declare(strict_types=1)` enabled
- [ ] Verify `@method` PHPDoc annotations are still needed
- [ ] Check for deprecated `each()` function usage
- [ ] Replace `money_format()` with `NumberFormatter`