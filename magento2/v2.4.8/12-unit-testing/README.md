# Topic 12: Unit Testing in Magento 2

**Philosophy:** Code Quality Assurance — Verify your code works correctly, now and after every change. Tests are not a luxury — they are the documentation that never goes stale, the safety net that catches regressions before they reach production, and the enabler of fearless refactoring.

---

## Overview

Unit testing is crucial for professional Magento development. This guide covers how to write and run tests for your custom modules with production-grade patterns.

**Why This Matters for Magento:**

Magento 2 has over 30,000 core classes. When you modify a plugin, observer, or repository, you can inadvertently break functionality that seems unrelated. Without tests, you won't know until a customer reports it. With tests, you catch it in seconds.

---

## Prerequisites

- [ ] Module with working repository
- [ ] Understanding of PHPUnit
- [ ] Test environment configured (`bin/magento dev:tests:run`)

---

## Learning Objectives

By end of this section, you will be able to:

- [ ] Understand unit testing fundamentals and why each test type exists
- [ ] Create unit tests for models and services
- [ ] Create integration tests for repositories and database operations
- [ ] Run tests and interpret results
- [ ] Mock Magento classes properly (ObjectManager dependencies)
- [ ] Use fixture patterns for consistent, isolated tests
- [ ] Understand test isolation issues and how to avoid them

---

## Topics

---

### Topic 1: Unit Testing Fundamentals

#### Why Test? The Real-World Impact

| Without Tests | With Tests |
|--------------|------------|
| A change to OrderRepository breaks customer email sending | Repository tests catch save failures |
| A plugin modification causes a fatal error in checkout | Plugin tests catch invocation errors |
| Refactoring a helper breaks a module nobody knew depended on it | Tests catch unexpected breakage |
| Bug reported by customer = emergency hotfix | Bug caught by test = controlled fix |

**The Maintenance Cost Reality:**

Studies consistently show that fixing a bug in production costs 10-100x more than catching it during development. Tests are a financial decision, not a technical one.

#### Unit vs Integration vs Functional: When to Use Each

| Type | What It Tests | Speed | Scope | Best For |
|------|---------------|-------|-------|----------|
| **Unit** | Single method/class in isolation | <1s | One class | Business logic, validators, calculators |
| **Integration** | Multiple components together | 1-10s | Several classes + DB | Repository CRUD, services, observers |
| **Functional** | Full user workflow | 10s-60s | Entire app | Critical flows (checkout, login) |

**Industry Best Practice: The Testing Pyramid**

```
        /\
       /  \
      / F  \     Functional (few, slow, high-fidelity)
     /unctional\
    /-----------\
   /Integration  \   Integration (some, medium speed)
  /   Tests       \
 /-----------------\
/   Unit Tests      \   Unit (many, fast, isolated)
---------------------
```

Magento's core has ~30,000 unit tests. Your custom module should aim for ~80% unit tests, ~20% integration tests.

#### Unit Test Structure

```php
<?php
declare(strict_types=1);

namespace Training\HelloWorld\Test\Unit\Model;

use PHPUnit\Framework\TestCase;
use Training\HelloWorld\Model\Message as Model;

class MessageTest extends TestCase
{
    public function testGetMessageReturnsString()
    {
        $model = new Model();
        $result = $model->getMessage();

        $this->assertIsString($result);
    }

    public function testSetMessageStoresValue()
    {
        $model = new Model();
        $testMessage = 'Test Message';

        $model->setMessage($testMessage);

        $this->assertEquals($testMessage, $model->getMessage());
    }

    public function testSetMessageWithEmptyString()
    {
        $this->expectException(\Magento\Framework\Exception\InputException::class);

        $model = new Model();
        $model->setMessage('');
    }
}
```

#### Why `declare(strict_types=1)` in Tests?

Just like production code, tests should use strict typing. Without it, PHP's type coercion can mask bugs:

```php
// Without strict_types: "10 eggs" becomes int 10 silently
// With strict_types: throws TypeError
$this->assertEquals(10, $this->model->getEggCount());
```

---

### Topic 2: Mocking Magento Classes

#### The ObjectManager Dependency Problem

Magento classes are not always easy to instantiate because they often depend on `ObjectManager`:

```php
<?php
// This might fail in a unit test because:
// 1. $this->collectionFactory needs ObjectManager
// 2. Collection needs DB connection
// 3. The constructor might run real queries
class OrderRepository
{
    public function __construct(
        \Magento\Sales\Model\Order\CollectionFactory $collectionFactory,
        \Magento\Framework\Api\SearchCriteriaBuilder $searchCriteriaBuilder
    ) {
        $this->collectionFactory = $collectionFactory;
        $this->searchCriteriaBuilder = $searchCriteriaBuilder;
    }
}
```

**The Solution: Mock the Factory**

When a class depends on a Factory (`*Factory`), you can mock it:

```php
<?php
use PHPUnit\Framework\TestCase;
use Training\Review\Model\Review;
use Training\Review\Model\ResourceModel\Review as ReviewResource;
use Training\Review\Model\ReviewFactory;
use Training\Review\Model\ReviewRepository;
use Magento\Framework\Api\SearchResultsInterface;
use Magento\Framework\Api\SearchResultsInterfaceFactory;

class ReviewRepositoryTest extends TestCase
{
    public function testSave()
    {
        // Create mock review
        $review = $this->createMock(Review::class);

        // Create mock resource
        $resource = $this->createMock(ReviewResource::class);
        $resource->expects($this->once())
            ->method('save')
            ->with($review);

        // Create mock factory for review
        $reviewFactory = $this->createMock(ReviewFactory::class);
        $reviewFactory->method('create')->willReturn($review);

        // Create mock search results
        $searchResults = $this->createMock(SearchResultsInterface::class);

        // Create mock search results factory
        $searchResultsFactory = $this->createMock(SearchResultsInterfaceFactory::class);
        $searchResultsFactory->method('create')->willReturn($searchResults);

        // Instantiate repository with mocks
        $repository = new ReviewRepository(
            $resource,
            $reviewFactory,
            $searchResultsFactory
        );

        // Execute
        $repository->save($review);

        // Assertion is implicit via $resource->expects(...)->method('save')
        // If save wasn't called once, this test fails
    }
}
```

#### Mocking ObjectManager-Dependent Classes

When a class requires ObjectManager but doesn't use Factories cleanly:

```php
<?php
// The problem: Some Magento classes need ObjectManager for constructor injection
// during tests

// Solution 1: Use Magento's Test Framework
use Magento\TestFramework\ObjectManager;

$objectManager = ObjectManager::getInstance();
$service = $objectManager->create(MyClass::class);

// Solution 2: Mock the ObjectManager itself (more controlled)
public function testWithObjectManagerMock()
{
    $objectManager = $this->createMock(\Magento\Framework\ObjectManagerInterface::class);

    $objectManager->method('create')
        ->with(MyClass::class, $this->anything())
        ->willReturn(new MyClass());

    // Now inject into test subject
    $subject = new MyService($objectManager);
}
```

#### The `expects()` Matcher Reference

| Matcher | Meaning |
|---------|---------|
| `$this->any()` | Method can be called 0 or more times |
| `$this->once()` | Method must be called exactly once |
| `$this->atLeastOnce()` | Method must be called 1+ times |
| `$this->exactly($n)` | Method must be called exactly $n times |
| `$this->never()` | Method must not be called |
| `$this->returnValue($value)` | Return this value |
| `$this->returnArgument($index)` | Return argument at index |
| `$this->returnCallback($callback)` | Return result of callback function |
| `$this->throwException($e)` | Throw this exception |
| `$this->onConsecutiveCalls(...$values)` | Return different values on consecutive calls |

---

### Topic 3: Integration Testing

#### Why Integration Tests?

Unit tests test classes in isolation. Integration tests verify that classes work together. In Magento:
- Repository tests need the real database
- Observer tests need the real event manager
- Email tests need the real mail transport

**When Unit Tests Are Not Enough:**

```php
<?php
// Unit test passes — class works in isolation
public function testGetOrderTotal()
{
    $calculator = new ShippingCalculator();
    $calculator->setBaseRate(5.99);
    $calculator->setFreeShippingThreshold(100);

    // But this fails in production because:
    // collectTotals() depends on $quote->getItems() which returns []
    // In real execution, items exist on the quote
    $result = $calculator->calculateShipping($quote);

    $this->assertEquals(5.99, $result);
}
```

**The fix: Integration test with a real or mocked quote.**

#### Integration Test Setup

```php
<?php
declare(strict_types=1);

namespace Training\Review\Test\Integration;

use Magento\Test\Framework\IntegrationTestCase;
use Magento\TestFramework\ObjectManager;
use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Quote\Api\Data\CartItemInterface;
use Magento\Quote\Api\Data\CartItemInterfaceFactory;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Catalog\Api\Data\ProductInterface;

class CartManagementTest extends IntegrationTestCase
{
    private CartRepositoryInterface $cartRepository;
    private CartItemInterfaceFactory $cartItemFactory;
    private ProductRepositoryInterface $productRepository;

    protected function setUp(): void
    {
        parent::setUp();
        $objectManager = ObjectManager::getInstance();
        $this->cartRepository = $objectManager->get(CartRepositoryInterface::class);
        $this->cartItemFactory = $objectManager->get(CartItemInterfaceFactory::class);
        $this->productRepository = $objectManager->get(ProductRepositoryInterface::class);
    }

    public function testAddProductToCart()
    {
        // Create a guest cart
        $cart = $this->cartRepository->create();
        $cart->setStoreId(1);

        // Get a real product (from test fixture or DB)
        $product = $this->productRepository->get('simple_product');

        // Create cart item
        /** @var CartItemInterface $cartItem */
        $cartItem = $this->cartItemFactory->create();
        $cartItem->setQuoteId($cart->getId());
        $cartItem->setProduct($product);
        $cartItem->setSku($product->getSku());
        $cartItem->setQty(2);

        // Save via repository
        $this->cartRepository->save($cartItem);

        // Reload cart
        $cart = $this->cartRepository->get($cart->getId());

        // Assert
        $this->assertEquals(1, $cart->getItemsCount());
        $this->assertEquals(2, $cart->getItemByProduct($product)->getQty());

        // Cleanup
        $this->cartRepository->delete($cart);
    }
}
```

#### Fixture Patterns

Fixtures provide known, consistent test data. Magento's integration test framework uses YAML fixtures:

**Fixture File: `Test/Integration/_files/review.yml`**

```yaml
# Creates a product with specific attributes for testing
- entity_id: 1
  attribute_set_id: 15
  type_id: simple
  sku: test-product-1
  name: Test Product 1
  price: 99.99
  stock_data:
    qty: 100
    is_in_stock: 1
```

**Loading Fixtures:**

```php
<?php
use Magento\Framework\Module\Dir\Reader;
use Magento\TestFramework\Helper\Bootstrap;

class ReviewRepositoryTest extends IntegrationTestCase
{
    protected function setUp(): void
    {
        parent::setUp();

        // Load fixture before each test
        $this->loadFixture('_files/review.yaml');
    }

    protected function tearDown(): void
    {
        // Cleanup after each test
        $this->cleanupFixtures();
        parent::tearDown();
    }
}
```

**Why Fixtures Matter:**

1. **Isolation:** Each test starts with known data
2. **Repeatability:** Same data every test run
3. **Speed:** No manual data entry
4. **CI/CD:** Tests run without human interaction

#### Data Fixture vs. Stub Data

| Approach | Pros | Cons |
|----------|------|------|
| **Real fixture (YAML)** | Tests real code paths, most realistic | Slower, needs DB setup |
| **Mock object** | Fast, no DB needed | May miss integration issues |
| **Factory-created stub** | Middle ground | Still needs some mocking |

**Pro Tip: Always Test With Real Data in Integration Tests**

Mocking the database in integration tests defeats the purpose. Use `IntegrationTestCase` with real fixtures for true integration testing.

---

### Topic 4: Test Isolation Issues

#### Why Tests Must Not Affect Each Other

In a perfect test suite:
- Test A passes whether Test B runs or not
- Test A passes whether Test B passes or fails
- Running tests in any order produces the same results

**In Magento, this is challenging because:**

1. Static properties persist across tests
2. Singleton instances are reused
3. The ObjectManager is a global registry
4. Database state can leak between tests

#### Common Isolation Problems and Fixes

**Problem 1: Singleton State Leaking**

```php
<?php
// A class that stores state in a static property
class PriceCalculator
{
    private static array $discountCache = [];

    public static function applyDiscount(float $price): float
    {
        if (!isset(self::$discountCache['rate'])) {
            self::$discountCache['rate'] = Config::getDiscountRate();
        }
        return $price * self::$discountCache['rate'];
    }
}

// Test 1 sets the cache
public function testDiscountApplied()
{
    PriceCalculator::applyDiscount(100.00); // Caches rate
}

// Test 2 fails because it inherits cached rate
public function testZeroDiscount()
{
    PriceCalculator::applyDiscount(0.00); // Uses cached rate from Test 1
}
```

**Fix:** Reset static state in `tearDown()`

```php
protected function tearDown(): void
{
    // Clear static caches
    PriceCalculator::resetDiscountCache();
    Config::resetInstance();

    parent::tearDown();
}
```

**Problem 2: ObjectManager Singletons**

```php
<?php
// ObjectManager::get() returns the same instance by default
$objectManager = ObjectManager::getInstance();
$service1 = $objectManager->get(MyService::class);  // Instance A
$service2 = $objectManager->get(MyService::class);  // Instance A (same!)

// If Test 1 modifies service1's state, Test 2 sees it
```

**Fix:** Use `ObjectManager::resetInstance()` in `tearDown()`

```php
protected function tearDown(): void
{
    ObjectManager::getInstance()->resetInstance();
    parent::tearDown();
}
```

**Problem 3: Database State Leakage**

```php
<?php
// Test 1 creates an order
public function testOrderCreation()
{
    $order = $this->orderRepository->save($orderData);
    $this->orderId = $order->getId(); // Stored for next test
}

// Test 2 tries to create order with same data
public function testDuplicateOrderFails()
{
    // This might fail because order #1 was never cleaned up
    // Or it might pass when it shouldn't (unique constraint not enforced)
}
```

**Fix:** Use database transactions that rollback

```php
protected function setUp(): void
{
    parent::setUp();
    $this->dbConnection->beginTransaction();
}

protected function tearDown(): void
{
    $this->dbConnection->rollBack();  // Undo all changes
    parent::tearDown();
}
```

#### Using `Magento\Framework\Interception\PluginList` Safely

Magento's plugin system caches plugin configurations in `PluginList`. This can cause test interference:

```php
<?php
// In your test's tearDown
ObjectManager::getInstance()->removeShared(\Magento\Framework\Interception\PluginList::class);
ObjectManager::getInstance()->removeShared(\Magento\Framework\Interception\CachedReader::class);
```

---

### Topic 5: PHPUnit Configuration

#### phpunit.xml.dist for Magento Modules

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- app/code/Training/HelloWorld/phpunit.xml.dist -->
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.5/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         colors="true"
         convertErrorsToExceptions="true"
         convertNoticesToExceptions="true"
         convertWarningsToExceptions="true"
         beStrictAboutOutputDuringTests="true"
         beStrictAboutTodoAnnotatedTests="true"
         beStrictAboutTestsThatDoNotTestAnything="true"
         failOnRisky="true"
         failOnWarning="true"
         executionOrder="random"
         group="unit">
    <testsuites>
        <testsuite name="Unit">
            <directory>Test/Unit</directory>
        </testsuite>
        <testsuite name="Integration">
            <directory>Test/Integration</directory>
        </testsuite>
    </testsuites>

    <groups>
        <include>
            <group>unit</group>
        </include>
        <exclude>
            <group>integration</group>
        </exclude>
    </groups>

    <php>
        <ini name="display_errors" value="1"/>
        <ini name="error_reporting" value="-1"/>
        <env name="APPLICATION_ENV" value="testing"/>
        <env name="KERNEL_CLASS" value="Magento\Framework\App\Kernel"/>
    </php>
</phpunit>
```

#### Key PHPUnit Configuration Options

| Setting | Purpose | Why Important |
|---------|---------|---------------|
| `beStrictAboutTestsThatDoNotTestAnything` | Fail if test has no assertions | Prevents accidental empty tests |
| `failOnRisky` | Fail if test modifies global state | Catches tests that "leak" |
| `executionOrder="random"` | Run tests in random order | Exposes order dependencies |
| `beStrictAboutOutputDuringTests` | Fail if test outputs unexpected text | Catches debug statements left in code |

**Why `executionOrder="random"` is Critical**

If Test B always passes because it runs after Test A (which sets up state), you'll never catch the dependency. Random order exposes hidden dependencies.

#### Running Tests: Commands Reference

```bash
# Run all unit tests
vendor/bin/phpunit --testsuite Unit

# Run all integration tests
vendor/bin/phpunit --testsuite Integration

# Run specific test class
vendor/bin/phpunit app/code/Training/HelloWorld/Test/Unit/Model/MessageTest.php

# Run specific test method
vendor/bin/phpunit --filter testGetMessageReturnsString

# Run tests with coverage report
vendor/bin/phpunit --coverage-html coverage/Unit

# Run only tests with @group unit annotation
vendor/bin/phpunit --group unit

# Run tests excluding @group integration
vendor/bin/phpunit --exclude-group integration

# Run with testdox output (human-readable)
vendor/bin/phpunit --testdox --testsuite Unit
```

#### Annotations: `@group` and `@depends`

```php
<?php
/**
 * @group unit
 * @group model
 */
class MessageTest extends TestCase
{
    /**
     * Dependencies: this test needs testSetup to pass first
     * @depends testSetup
     */
    public function testGetMessageReturnsString()
    {
        // ...
    }

    /**
     * @group slow
     */
    public function testLargeDatasetProcessing()
    {
        // ...
    }
}
```

**Use `@depends` Sparingly**

Dependencies between tests are a code smell — they indicate the tests are not truly isolated. Only use `@depends` when you genuinely need to share expensive setup between tests.

---

## Reference Exercises

**Exercise 1:** Create unit tests for your model's getters and setters. Include tests for invalid input (out-of-range values, empty strings, null).

**Exercise 2:** Create a unit test for a service class that mocks its dependencies using `createMock()`.

**Exercise 3:** Create an integration test that loads a real product, adds it to a cart, and verifies the cart total.

**Exercise 4:** Create a fixture file for a test product and load it in an integration test.

**Exercise 5:** Run your tests with `--testdox` and verify the output describes the tests in plain English.

**Exercise 6:** Run your tests with random order (`--execution-order=random`) and verify they still pass.

---

## Reading List

| Resource | Why Read It |
|----------|-------------|
| `dev/tests/unit/phpunit.xml.dist` | Magento core PHPUnit config |
| `Magento\TestFramework\ObjectManager` | Integration test helpers |
| `Magento\TestFramework\Annotation\ComponentRegistry` | How Magento handles component state in tests |
| [PHPUnit Documentation](https://phpunit.readthedocs.io/) | Full PHPUnit reference |
| [Mocking Magento Classes](https://developer.adobe.com/commerce/php/development/components/unit-testing/) | Official Magento testing guide |

---

## Edge Cases & Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Test passes locally but fails in CI | Environment difference | Ensure same PHP version, extensions, env vars |
| Tests fail randomly | Test isolation issue | Check for shared state, use `tearDown()` cleanup |
| `createMock()` returns null | Class doesn't exist | Check class name spelling, run `setup:upgrade` |
| Integration test DB locked | Previous test didn't rollback | Use transaction rollback in `tearDown()` |
| ObjectManager error in unit test | Need to mock ObjectManager | Use `Magento\TestFramework\ObjectManager` |
| Test times out | Infinite loop or slow query | Add timeout; check for uncapped loops |

---

## Common Mistakes to Avoid

1. ❌ **Testing implementation details instead of behavior**
   - **Fix:** Test what the code does, not how it does it. `assertEquals(100, $calculator->getTotal())` not `assertEquals(5, count($calculator->getItems()))`.

2. ❌ **No assertions in tests**
   - **Fix:** Every test must have at least one assertion. PHPUnit will warn about this.

3. ❌ **Mocking too much**
   - **Fix:** If you're mocking 10 dependencies, the class is too complex. Consider refactoring.

4. ❌ **Not cleaning up in `tearDown()`**
   - **Fix:** Always rollback DB transactions, clear static caches, reset ObjectManager singletons.

5. ❌ **Using `@depends` to share state**
   - **Fix:** If tests depend on each other, the setup should be in `setUp()`, not in another test.

6. ❌ **Not running tests in random order**
   - **Fix:** Add `<executionOrder>random</executionOrder>` to catch hidden dependencies.

7. ❌ **Testing with production data**
   - **Fix:** Always use fixtures for known, controlled test data.

8. ❌ **Leaving debug statements in tests**
   - **Fix:** Use `beStrictAboutOutputDuringTests="true"` to catch stray `var_dump()` calls.

---

## Completion Criteria

- [ ] Unit tests created for all model getters/setters with invalid input cases
- [ ] Unit tests created for service classes with properly mocked dependencies
- [ ] Integration tests created for repository CRUD operations using real fixtures
- [ ] Tests run successfully with `vendor/bin/phpunit --testdox --testsuite Unit`
- [ ] Tests pass in random order (`--execution-order=random`)
- [ ] No tests left with `@depends` (tests are self-contained)
- [ ] `tearDown()` methods properly clean up DB transactions and static state

---

*Magento 2 Backend Developer Course — Topic 12 | Unit Testing*
