---
title: "31 - Testing (Unit/Integration/Functional)"
description: "Deep dive into Magento 2.4.8 testing: PHPUnit unit tests, integration tests, fixture management, API testing, and functional testing patterns."
tags: magento2, testing, phpunit, unit-tests, integration-tests, functional-tests, tdd, test-automation
rank: 18
pathways: [magento2-deep-dive]
---

# Testing (Unit/Integration/Functional)

Magento 2.4.8 ships with a comprehensive testing infrastructure built on PHPUnit, enabling developers to validate customizations at multiple levels. This article covers the testing philosophy, PHPUnit configuration, unit test patterns using ObjectManager, integration test infrastructure with fixtures, API testing approaches, functional testing with the Functional Testing Framework, CLI testing, and code coverage integration.

---

## 1. Testing Philosophy

### Test-Driven Development in Magento

TDD in Magento follows the red-green-refactor cycle:

1. **Red** - Write a failing test that defines expected behavior
2. **Green** - Write minimal code to pass the test
3. **Refactor** - Clean up code while keeping tests green

```php
// app/code/{Vendor}/{Module}/Test/Unit/Model/ProductValidatorTest.php
namespace Vendor\Module\Test\Unit\Model;

use PHPUnit\Framework\TestCase;

class ProductValidatorTest extends TestCase
{
    public function testValidateReturnsTrueForValidProduct()
    {
        // Red: Write failing test
        $validator = new ProductValidator();
        $productMock = $this->createMock(Product::class);
        $productMock->method('getPrice')->willReturn(100);
        
        $result = $validator->validate($productMock);
        $this->assertTrue($result);
    }
}
```

### Test Isolation in Magento

Magento's dependency injection and event-driven architecture make test isolation challenging. Key isolation strategies:

- **Mock external dependencies** - Database, HTTP clients, filesystem
- **Use ObjectManager** - Instantiate classes without full DI container
- **Isolate by scope** - Unit tests mock everything, integration tests use real DI

```php
// Isolation levels in Magento
class ProductRepositoryTest extends TestCase
{
    private $productRepository;
    private $searchCriteriaBuilder;
    
    protected function setUp(): void
    {
        // Unit level: mock everything
        $this->searchCriteriaBuilder = $this->createMock(
            \Magento\Framework\Api\SearchCriteriaBuilder::class
        );
        
        // Integration level: use real ObjectManager
        $objectManager = new \Magento\Framework\TestFramework\Unit\Helper\ObjectManager($this);
        $this->productRepository = $objectManager->getObject(
            \Magento\Catalog\Model\ProductRepository::class,
            [
                'searchCriteriaBuilder' => $this->searchCriteriaBuilder
            ]
        );
    }
}
```

### Why Magento Testing is Unique

Magento presents unique testing challenges:

| Challenge | Description | Mitigation |
|-----------|-------------|------------|
| **Heavy DI container** | Classes have many dependencies | ObjectManager helper |
| **Event/observer system** | Side effects from observer execution | Disable observers in unit tests |
| **Database dependency** | Models require DB connection | Integration tests with fixture rollback |
| **Complex configuration** | XML-based DI and schema | Test stubs for etc/di.xml |
| **Stateful singletons** | Service layer often stateful | Reset singletons between tests |

---

## 2. PHPUnit Configuration

### Module phpunit.xml.dist

Each module declares its test configuration in `Test/phpunit.xml.dist`:

```xml
<?xml version="1.0"?>
<!--
/**
 * Copyright © Magento, Inc. All rights reserved.
 * See COPYING.txt for license details.
 */
-->
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.6/phpunit.xsd"
         bootstrap="./Bootstrap.php"
         colors="true"
         executionOrder="random"
         failOnRisky="true"
         failOnWarning="true"
         stopOnFailure="false"
         cacheResult="false">
    <testsuites>
        <testsuite name="Unit">
            <directory>Unit</directory>
        </testsuite>
        <testsuite name="Integration">
            <directory>Integration</directory>
        </testsuite>
    </testsuites>
    <coverage processUncoveredFiles="true">
        <include>
            <directory suffix=".php">./Unit</directory>
        </include>
        <report>
            <html outputDirectory="var/coverage"/>
            <text outputFile="php://stdout"/>
        </report>
    </coverage>
    <php>
        <ini name="display_errors" value="1"/>
        <ini name="error_reporting" value="-1"/>
        <server name="APP_ENV" value="test" force="true"/>
    </php>
</phpunit>
```

### Bootstrap Files

The bootstrap file initializes the Magento test environment:

```php
// app/code/{Vendor}/{Module}/Test/Bootstrap.php
<?php
/**
 * Copyright © Magento, Inc. All rights reserved.
 * See COPYING.txt for license details.
 */

use Magento\Framework\Component\ComponentRegistrar;
use Magento\Framework\Autoload\AutoloaderRegistry;

define('TEST_Module_DIR', __DIR__);
define('UPLOAD_DIR', __DIR__ . '/../../../../../var/uploads');

require_once __DIR__ . '/../../../../app/autoload.php';

// Initialize autoloader for module classes
ComponentRegistrar::register(
    ComponentRegistrar::MODULE, '{Vendor}_{Module}',
    __DIR__ . '/../'
);
```

### Test Suite Organization

Magento organizes tests hierarchically:

```
app/code/{Vendor}/{Module}/
├── Test/
│   ├── Bootstrap.php              # Unit bootstrap
│   ├── phpunit.xml.dist           # PHPUnit configuration
│   ├── Unit/
│   │   ├── Model/
│   │   │   └── ProductValidatorTest.php
│   │   └── Helper/
│   │       └── DataHelperTest.php
│   └── Integration/
│       ├── Plugin/
│       │   └── ProductPluginTest.php
│       └── Model/
│           └── ProductRepositoryTest.php
```

---

## 3. Unit Tests

### ObjectManager Helper

The `Magento\Framework\TestFramework\Unit\Helper\ObjectManager` instantiates classes with mocked dependencies:

```php
// app/code/{Vendor}/{Module}/Test/Unit/Model/ProductServiceTest.php
namespace Vendor\Module\Test\Unit\Model;

use Magento\Framework\TestFramework\Unit\Helper\ObjectManager;
use PHPUnit\Framework\TestCase;

class ProductServiceTest extends TestCase
{
    private $objectManager;
    private $productService;

    protected function setUp(): void
    {
        $this->objectManager = new ObjectManager($this);
    }

    public function testGetPriceWithDiscount()
    {
        // Create mock with chained method expectations
        $productMock = $this->objectManager->getMockBuilder(
            \Magento\Catalog\Model\Product::class
        )
            ->disableOriginalConstructor()
            ->getMock();

        $productMock->method('getPrice')
            ->willReturn(100.00);
        $productMock->method('getDiscountPercent')
            ->willReturn(20.00);

        $this->productService = $this->objectManager->getObject(
            \Vendor\Module\Model\ProductService::class,
            [
                'product' => $productMock
            ]
        );

        $finalPrice = $this->productService->getFinalPrice($productMock);
        $this->assertEquals(80.00, $finalPrice);
    }
}
```

### Mocking Strategies

**Mocking with method chains:**

```php
$sessionMock = $this->objectManager->getMockBuilder(
    \Magento\Customer\Model\Session::class
)
    ->disableOriginalConstructor()
    ->getMock();

$sessionMock->method('isLoggedIn')->willReturn(true);
$sessionMock->method('getCustomerId')->willReturn(42);
$sessionMock->method('getCustomerData')
    ->willReturn($customerDataMock);
```

**Mocking with callback expectations:**

```php
$priceMock->method('getPrice')
    ->willReturnCallback(function ($arg) {
        return $arg > 0 ? $arg : 0;
    });
```

**Mocking with map of calls:**

```php
$productMock->method('getName')
    ->willReturnOnConsecutiveCalls('Product A', 'Product B', 'Product C');
```

### Testing Protected Methods

Use reflection to test private/protected methods:

```php
public function testValidatePriceWithReflection()
{
    $objectManager = new ObjectManager($this);
    
    $validator = $objectManager->getObject(
        \Vendor\Module\Model\PriceValidator::class
    );
    
    // Use reflection to access protected method
    $reflectionClass = new \ReflectionClass($validator);
    $method = $reflectionClass->getMethod('validatePriceRange');
    $method->setAccessible(true);
    
    $result = $method->invoke($validator, 50.00, 100.00);
    $this->assertTrue($result);
}
```

### Data Provider Pattern

```php
/**
 * @dataProvider priceCalculationDataProvider
 */
public function testPriceCalculation($basePrice, $discount, $expected)
{
    $calculator = new PriceCalculator();
    $result = $calculator->calculate($basePrice, $discount);
    $this->assertEquals($expected, $result);
}

public static function priceCalculationDataProvider(): array
{
    return [
        'standard discount' => [100.00, 10.00, 90.00],
        'no discount' => [100.00, 0.00, 100.00],
        'full discount' => [100.00, 100.00, 0.00],
    ];
}
```

---

## 4. Integration Tests

### Integration Test Bootstrap

Integration tests require Magento's full bootstrap. The integration test configuration for full integration tests is located at `dev/tests/integration/phpunit.xml` (not `framework/bootstrap.php` which is the path referenced internally by the bootstrap attribute):

```xml
// dev/tests/integration/phpunit.xml
<?xml version="1.0"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.6/phpunit.xsd"
         bootstrap="framework/bootstrap.php"
         colors="true"
         executionOrder="depends,random"
         failOnRisky="true"
         failOnWarning="true">
    <testsuites>
        <testsuite name="Magento Integration Tests">
            <directory>testsuite</directory>
        </testsuite>
    </testsuites>
    <php>
        <ini name="display_errors" value="1"/>
        <server name="APP_ENV" value="test" force="true"/>
    </php>
</phpunit>
```

The `bootstrap="framework/bootstrap.php"` attribute refers to a path relative to the phpunit.xml location, resolved internally by the test framework. For module-level integration tests, the bootstrap file at `dev/tests/integration/framework/bootstrap.php` initializes the full Magento application context including the database connection and DI container.

### Test Case Base Classes

Magento provides specialized test base classes:

```php
// Using Magento's integration test base
namespace Vendor\Module\Test\Integration;

use Magento\TestFramework\TestCase\IntegrationTest;

class ProductRepositoryTest extends IntegrationTest
{
    protected function setUp(): void
    {
        // Integration tests have access to real DI container
        $this->objectManager = \Magento\TestFramework\AppObjectManager::getInstance();
    }

    public function testSaveProduct()
    {
        $repository = $this->objectManager->create(
            \Magento\Catalog\Api\ProductRepositoryInterface::class
        );
        
        $product = $this->objectManager->create(\Magento\Catalog\Model\Product::class);
        $product->setSku('test-sku-' . uniqid());
        $product->setName('Test Product');
        $product->setPrice(99.99);
        $product->setAttributeSetId(4);
        
        $saved = $repository->save($product);
        $this->assertNotNull($saved->getId());
    }
}
```

### Test Annotation Patterns

```php
/**
 * Integration tests for catalog product repository
 *
 * @magentoDbIsolation enabled
 * @magentoDataFixture Magento/Catalog/_files/products.php
 */
class ProductRepositoryTest extends IntegrationTest
{
    /**
     * @magentoAppArea frontend
     */
    public function testFrontendProductRetrieval()
    {
        // Tests run in frontend area context
    }

    /**
     * @magentoAppArea adminhtml
     */
    public function testAdminProductCreation()
    {
        // Tests run in adminhtml area context
    }
}
```

---

## 5. Database Fixtures

### SQL Fixtures

SQL fixtures load test data directly into the database:

```php
// dev/tests/integration/etc/fixtures/dump.sql
-- Products fixture
INSERT INTO `catalog_product_entity` (entity_id, sku, attribute_set_id, type_id, created_at, updated_at)
VALUES (1, 'simple_product_1', 4, 'simple', NOW(), NOW());

INSERT INTO `catalog_product_entity_varchar` (value_id, attribute_id, store_id, entity_id, value)
VALUES (1, 73, 0, 1, 'Simple Product One');

INSERT INTO `catalog_product_entity_decimal` (value_id, attribute_id, store_id, entity_id, value)
VALUES (1, 75, 0, 1, 99.99);
```

### Data Fixture Attribute

The `@magentoDataFixture` annotation loads PHP fixtures:

```php
/**
 * @magentoDataFixture Magento/Catalog/_files/products.php
 */
public function testProductSearch()
{
    // The products.php fixture creates test products
    // Fixture is automatically rolled back after test
}
```

### Data Fixture File Structure

```php
// magento/module-catalog/Test/Integration/_files/products.php
<?php
/**
 * Copyright © Magento, Inc. All rights reserved.
 * See COPYING.txt for license details.
 */

use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Catalog\Model\Product;
use Magento\TestFramework\Helper\Bootstrap;

$objectManager = Bootstrap::getObjectManager();
$productRepository = $objectManager->get(ProductRepositoryInterface::class);

// Create test products
$product = $objectManager->create(Product::class);
$product->setSku('test-product-sku');
$product->setName('Test Product');
$product->setPrice(49.99);
$product->setAttributeSetId(4);
$product->setTypeId('simple');
$product->setStatus(1);
$productRepository->save($product);

// Store product ID for later retrieval
$_productId = $product->getId();
```

### Rollback Mechanisms

Fixtures are rolled back via transaction rollback or explicit cleanup:

```php
// Automatic rollback via transaction
public function testWithAutomaticRollback()
{
    $this->getDatabaseConnection()->beginTransaction();
    
    try {
        // Create test data
        $this->createTestCustomer();
        
        // Test operations...
        
        // No explicit rollback needed - transaction rolled back on teardown
    } catch (Exception $e) {
        $this->getDatabaseConnection()->rollBack();
        throw $e;
    }
}

// Explicit fixture cleanup
public static function tearDownAfterClass(): void
{
    // Clean up products created during tests
    $productRepository = Bootstrap::getObjectManager()
        ->get(ProductRepositoryInterface::class);
    
    foreach (self::$_createdProducts as $productId) {
        try {
            $productRepository->deleteById($productId);
        } catch (\Exception $e) {
            // Log but continue cleanup
        }
    }
}
```

### DataFixture Attribute Options

```php
/**
 * @magentoDataFixture Magento/Customer/_files/customer.php
 * @magentoDataFixture Magento/Catalog/_files/categories.php
 * @magentoDbIsolation enabled
 * @magentoAppArea frontend
 */
class CheckoutTest extends IntegrationTest
{
    // Multiple fixtures are executed in order
    // DbIsolation ensures each test runs in isolated transaction
}
```

---

## 6. API Testing

### REST API Testing Pattern

```php
// dev/tests/integration/testsuite/Magento/Customer/Api/AccountRepositoryTest.php
namespace Magento\Customer\Test\Api;

use Magento\TestFramework\TestCase\WebapiFunctional;
use Magento\TestFramework\ObjectManager;

class AccountRepositoryTest extends WebapiFunctional
{
    protected function setUp(): void
    {
        $this->serviceBroker = ObjectManager::getInstance()
            ->get(\Magento\TestFramework\Api\ServiceBroker::class);
    }

    /**
     * @magentoApiDataFixture Magento/Customer/_files/customer.php
     */
    public function testCreateCustomer()
    {
        $customerData = [
            'email' => 'newcustomer@example.com',
            'firstname' => 'John',
            'lastname' => 'Doe',
            'website_id' => 1,
        ];

        $serviceInfo = [
            'rest' => [
                'resourcePath' => '/V1/customers',
                'httpMethod' => \Magento\Framework\Webapi\Rest\Request::HTTP_METHOD_POST,
            ]
        ];

        $response = $this->_webapiCall($serviceInfo, ['customer' => $customerData]);
        
        $this->assertArrayHasKey('id', $response);
        $this->assertEquals($customerData['email'], $response['email']);
    }
}
```

### GraphQL Testing Pattern

```php
/**
 * @magentoApiGraphQL
 */
class ProductQueryTest extends WebapiFunctional
{
    public function testProductQuery()
    {
        $query = <<<GRAPHQL
        {
            products(filter: { sku: { eq: "simple_product" } }) {
                items {
                    sku
                    name
                    price
                }
            }
        }
GRAPHQL;

        $serviceInfo = [
            'graphql' => [
                'endpoint' => '/graphql',
                'query' => $query,
            ]
        ];

        $response = $this->graphQlCall($serviceInfo);
        
        $this->assertArrayHasKey('products', $response);
        $this->assertEquals('simple_product', $response['products']['items'][0]['sku']);
    }
}
```

### Service Contract Testing

```php
public function testProductRepositoryGetList()
{
    $searchCriteria = $this->objectManager->create(
        \Magento\Framework\Api\SearchCriteria::class
    );
    
    $filterBuilder = $this->objectManager->create(
        \Magento\Framework\Api\FilterBuilder::class
    );
    
    $filterBuilder->setField('status')
        ->setValue(1)
        ->setConditionType('eq');
    
    $searchCriteria->setFilterGroups([
        $filterBuilder->create()
    ]);
    
    $productRepository = $this->objectManager->create(
        \Magento\Catalog\Api\ProductRepositoryInterface::class
    );
    
    $result = $productRepository->getList($searchCriteria);
    
    $this->assertInstanceOf(
        \Magento\Catalog\Api\Data\ProductSearchResultsInterface::class,
        $result
    );
}
```

---

## 7. Page Object Pattern

### Functional Testing Framework Setup

The Functional Testing Framework (MTF) uses the Page Object pattern:

```xml
<!-- dev/tests/functional/phpunit.xml -->
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.6/phpunit.xsd"
         bootstrap="vendor/magento/mtf/lib/bootstrap.php"
         colors="true">
    <testsuites>
        <testsuite name="Magento Functional Test Suite">
            <directory>testsuite</directory>
        </testsuite>
    </testsuites>
</phpunit>
```

### Page Object Structure

```php
// dev/tests/functional/Magento/ConfigurableProduct/Test/Page/Product/ProductView.php
namespace Magento\ConfigurableProduct\Test\Page\Product;

use Magento\FunctionalTestingFramework\Page\Handlers\PageObjectHandler;

class ProductView
{
    /**
     * Section locator for price display
     */
    public $priceSection = '.price-box';

    /**
     * Block locator for add to cart button
     */
    public $addToCartBlock = '#product-addtocart-button';

    /**
     * Get the displayed price
     *
     * @return string
     */
    public function getPrice()
    {
        $this->priceSection->waitForElementVisible();
        return $this->priceSection->getText();
    }

    /**
     * Click add to cart
     *
     * @return void
     */
    public function addToCart()
    {
        $this->addToCartBlock->click();
        $this->addToCartBlock->waitForElementNotVisible();
    }
}
```

### Cest Format Tests

```php
// dev/tests/functional/Magento/Catalog/Test/Cest/ProductViewCest.php
namespace Magento\Catalog\Test\Cest;

use Magento\FunctionalTestingFramework\Tester\AcceptanceTester;
use Magento\FunctionalTestingFramework\Pageobjects\PageObject;

class ProductViewCest
{
    public function viewSimpleProduct(AcceptanceTester $I)
    {
        // Open product page
        $I->amOnPage('/catalog/product/view/id/1');
        
        // Verify product information displays
        $I->waitForElementVisible('.product-info-main');
        $I->see('Simple Product');
        $I->see('$99.99');
        
        // Add to cart
        $I->click('#product-addtocart-button');
        $I->waitForElementVisible('.message-success');
        $I->see('You added Simple Product to your shopping cart.');
    }
}
```

### Section Objects

```php
// dev/tests/functional/Magento/Catalog/Test/Section/ProductInfoSection.php
namespace Magento\Catalog\Test\Section;

use Magento\FunctionalTestingFramework\Pageobjects\Section;

class ProductInfoSection extends Section
{
    /**
     * Array of CSS selectors mapped to section names
     *
     * @var array
     */
    protected $ selectors = [
        'productTitle' => '.product-name span',
        'productPrice' => '.price-box .price',
        'addToCartButton' => '#product-addtocart-button',
        'qtyInput' => '#qty',
    ];

    /**
     * Get product title
     *
     * @return string
     */
    public function getProductTitle()
    {
        $this->waitForElementVisible('productTitle');
        return $this->findElement('productTitle')->getText();
    }

    /**
     * Set quantity
     *
     * @param string $qty
     * @return void
     */
    public function setQuantity($qty)
    {
        $this->findElement('qtyInput')->fill($qty);
    }
}
```

---

## 8. CLI Testing

### Command Testing Pattern

```php
// dev/tests/integration/framework/Magento/Framework/ShellCommandTest.php
namespace Magento\Framework\Shell\Test\Unit;

use Magento\Framework\TestFramework\Unit\Helper\ObjectManager;
use Magento\Framework\Shell\Command;
use PHPUnit\Framework\TestCase;

class CommandTest extends TestCase
{
    public function testExecuteSimpleCommand()
    {
        $command = new Command('echo "test"');
        $output = $command->execute();
        
        $this->assertEquals("test\n", $output);
    }

    public function testExecuteWithParameters()
    {
        $command = new Command('php -r "echo $argv[1];"', ['hello world']);
        $output = $command->execute();
        
        $this->assertEquals('hello world', $output);
    }
}
```

### Application Command Testing

```php
// dev/tests/integration/testsuite/Magento/Catalog/Console/Command/ProductAttributeTest.php
namespace Magento\Catalog\Console\Command;

use Magento\TestFramework\Console\ExternalCommand;
use PHPUnit\Framework\TestCase;

class ProductAttributeCommandTest extends TestCase
{
    public function testProductAttributeListCommand()
    {
        $output = ExternalCommand::run(['bin/magento', 'catalog:product:attributes:list']);
        
        // Verify command executed
        $this->assertStringContainsString('Starting product attributes list', $output);
        
        // Verify output format
        $this->assertRegExp('/\|.+ \| [a-z_]+ \|/', $output);
    }
}
```

### Testing Admin Console Commands

```php
public function testCacheCleanCommand()
{
    $this->objectManager->get(\Magento\Framework\App\Cache::class)
        ->save('test_data', 'test_cache_id');
    
    $output = ExternalCommand::run(['bin/magento', 'cache:clean', 'config']);
    
    // Verify cache was cleared
    $this->assertStringContainsString('Cleaned cache', $output);
    
    // Verify cache no longer contains data
    $this->assertNull(
        $this->objectManager->get(\Magento\Framework\App\Cache::class)
            ->load('test_cache_id')
    );
}
```

---

## 9. Code Coverage

### XDebug Configuration

```php
// php.ini development settings for coverage
[XDebug]
zend_extension=xdebug.so
xdebug.mode=coverage
xdebug.start_with_request=trigger
xdebug.output_dir=/var/www/html/dev/tests/integration/var/log
xdebug.var_display_max_data=10000
xdebug.var_display_max_depth=10
```

### PHPDBG Configuration

```php
// php.ini production-style coverage
[phpdbg]
phpdbg.executable_path=/usr/bin/phpdbg
phpdbg.section=0
phpdbg.watches=
phpdbg.prompt=phpdbg>
```

### Coverage Configuration in phpunit.xml

```xml
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         bootstrap="bootstrap.php"
         colors="true"
         executionOrder="random"
         beStrictAboutOutputDuringTests="true"
         beStrictAboutTodoAnnotatedTests="true">
    <testsuites>
        <testsuite name="Unit">
            <directory>testsuite/Unit</directory>
        </testsuite>
    </testsuites>
    <coverage processUncoveredFiles="true">
        <include>
            <directory suffix=".php">app/code</directory>
            <directory suffix=".php">lib/internal</directory>
        </include>
        <exclude>
            <directory suffix=".php">app/code/*/Test</directory>
            <file>lib/internal/*.php</file>
        </exclude>
        <report>
            <clover outputFile="var/coverage/clover.xml"/>
            <html outputDirectory="var/coverage/html"/>
            <text outputFile="var/coverage/text.txt" showUncoveredFiles="false"/>
            <php outputFile="var/coverage/coverage.php"/>
        </report>
        <project name="Magento">
            <source>
                <include>
                    <directory>app/code</directory>
                </include>
            </source>
        </project>
    </coverage>
</phpunit>
```

### Running Coverage Reports

```bash
# Generate HTML coverage report
vendor/bin/phpunit --coverage-html var/coverage/html

# Generate Clover XML for CI integration
vendor/bin/phpunit --coverage-clover var/coverage/clover.xml

# Generate text summary
vendor/bin/phpunit --coverage-text=var/coverage/summary.txt

# Run with specific test filter
vendor/bin/phpunit --filter=testSaveProduct --coverage-html var/coverage/product-repo/
```

### Coverage Interpretation

```php
/**
 * Interpretation guidelines for Magento code coverage
 *
 * Target coverage by module type:
 * - Business logic: 80%+ coverage required
 * - Controllers: 70%+ coverage required
 * - Models: 75%+ coverage required
 * - Helpers: 85%+ coverage required
 * - Blocks: 60%+ coverage acceptable
 */
```

---

## 10. Test Category Matrix

### Decision Framework

| Scenario | Recommended Test Type | Why |
|----------|----------------------|-----|
| **Testing utility/helper classes** | Unit | No Magento dependencies |
| **Testing service layer** | Unit with mocking | Isolated, fast execution |
| **Testing repository methods** | Integration | Real DB queries needed |
| **Testing event observers** | Integration | Full event system required |
| **Testing API endpoints** | API (REST/GraphQL) | Validates contract |
| **Testing admin UI flows** | Functional (MTF) | Browser-based simulation |
| **Testing frontend checkout** | Functional (Cest) | Real user interaction |
| **Testing CLI commands** | CLI integration | Process execution |
| **Testing plugin interception** | Integration | DI container required |
| **Testing email notifications** | Integration | Mail capture system |

### Test Pyramid for Magento

```
                    ▲
                   /█\
                  /███\
                 /█████\
                /███████\
               /█████████\
              /███████████\
             ┌─────────────────────┐
             │   Functional (10%)   │  ← E2E scenarios, critical paths
             ├─────────────────────┤
             │ Integration (30%)    │  ← API contracts, DB operations, events
             ├─────────────────────┤
             │   Unit Tests (60%)   │  ← Business logic, helpers, validators
             └─────────────────────┘
```

### Speed vs Coverage Matrix

| Test Type | Execution Time | Coverage Scope | Use Case |
|-----------|----------------|----------------|----------|
| **Unit** | < 50ms/test | Single class | TDD, CI pipeline |
| **Integration** | 100-500ms/test | Module scope | Feature validation |
| **API** | 200-1000ms/test | Service contract | Contract testing |
| **Functional** | 1-10s/test | Full application | Regression testing |

### Recommended Test Distribution

For a typical Magento module:

```
Total tests: ~200
├── Unit Tests: ~120 (60%)
│   ├── Model validators: 30
│   ├── Helper classes: 25
│   ├── Service classes: 40
│   └── Block classes: 25
├── Integration Tests: ~60 (30%)
│   ├── Repository operations: 20
│   ├── Event observers: 15
│   ├── Plugin interception: 15
│   └── CLI commands: 10
└── Functional Tests: ~20 (10%)
    ├── Admin UI flows: 10
    └── Frontend flows: 10
```

---

## Summary

Magento 2.4.8's testing infrastructure provides multiple layers of validation:

- **Unit tests** with `ObjectManager` and mock builders for isolated testing
- **Integration tests** with fixtures for database and full DI validation
- **API tests** for REST and GraphQL service contracts
- **Functional tests** with MTF for end-to-end browser scenarios
- **CLI tests** for command validation

The test category matrix helps select the appropriate test type based on what needs to be validated, balancing execution speed against coverage scope. Coverage reports generated via XDebug or PHPDBG provide visibility into code untested, enabling teams to focus efforts on high-value test creation.

For TDD practitioners, start with unit tests for business logic, expand to integration tests for repository and event behavior, and reserve functional tests for critical user paths. This layered approach maintains fast feedback loops while ensuring comprehensive validation across the stack.

(End of file - total 1061 lines)