---
title: "21 - Error Handling Patterns"
description: "Comprehensive guide to Magento 2.4.8 error handling patterns: exception hierarchy, LocalizedException, NoSuchEntityException, InputException, CouldNotSave/CouldNotDelete, controller noSuchAction, ExceptionHandler, WebapiExceptionHandler, developer mode, error logging, custom exception classes, try-catch patterns, plugin exception handling, HTTP error responses, Phrase vs strings, and silent failure anti-patterns"
tags:
  - magento2
  - error-handling
  - exceptions
  - debugging
  - localizer
rank: 3
pathways:
  - magento2-deep-dive
see_also:
  - "../19-production-debugging/19-debugging-tools.md"
---

# Error Handling Patterns

Magento 2.4.8 implements a sophisticated exception hierarchy that serves as the backbone of all error handling across the platform. Understanding this hierarchy — when to throw each exception type, how exceptions propagate through controllers and plugins, and how the framework converts exceptions into HTTP responses — is essential for writing robust modules and diagnosing production issues. This chapter provides a thorough treatment of every significant error handling pattern in Magento 2.4.8, from the base `LocalizedException` class to the internals of the `WebapiExceptionHandler` that masks error details in production API responses.

---

## 1. Magento's Exception Hierarchy

Magento 2.4.8 organizes its exceptions in a deliberate hierarchy rooted at `Magento\Framework\Exception\LocalizedException`. Every custom exception your module throws should extend this root, either directly or through one of the framework's intermediate classes. The hierarchy exists so that calling code can catch exceptions at whatever level of granularity is appropriate — a repository might catch `NoSuchEntityException` specifically while a controller might catch the broader `LocalizedException`.

### The Root: `Magento\Framework\Exception\LocalizedException`

```php
// lib/internal/Magento/Framework/Exception/LocalizedException.php
declare(strict_types=1);

namespace Magento\Framework\Exception;

use Magento\Framework\Phrase;

/**
 * Base exception class for all Magento exceptions.
 *
 * All exceptions in Magento should extend this class or its subclasses.
 * This ensures a consistent interface across the entire exception hierarchy.
 */
class LocalizedException extends \Exception
{
    /**
     * @var Phrase
     */
    private $phrase;

    /**
     * @param Phrase|string $message
     * @param \Throwable|null $previous
     */
    public function __construct($message = '', ?\Throwable $previous = null)
    {
        if ($message instanceof Phrase) {
            $this->phrase = $message;
            $rawMessage = $message->render();
        } else {
            $this->phrase = new Phrase($message);
            $rawMessage = (string) $message;
        }
        parent::__construct($rawMessage, 0, $previous);
    }

    /**
     * Get the phrase representation of the message (preserves translation context).
     *
     * @return Phrase
     */
    public function getPhrase(): Phrase
    {
        return $this->phrase;
    }
}
```

The key design decision here is that `LocalizedException` stores a `Phrase` object alongside the raw string message. This `Phrase` is what enables `__('message')` style translation to work in exceptions — the translate call is captured at the point of exception creation and can be re-rendered in any locale context.

### Exception Class Map

Magento 2.4.8 defines the following primary exception classes in `lib/internal/Magento/Framework/Exception/`:

| Exception Class | File | When to Throw |
|---|---|---|
| `LocalizedException` | `LocalizedException.php` | Base class — rarely thrown directly |
| `NoSuchEntityException` | `NoSuchEntityException.php` | Entity not found by ID or unique field |
| `InputException` | `InputException.php` | Invalid input data (API/admin form validation) |
| `CouldNotSaveException` | `CouldNotSaveException.php` | Save operation failed within a transaction |
| `CouldNotDeleteException` | `CouldNotDeleteException.php` | Delete operation failed within a transaction |
| `ValidatorException` | `ValidatorException.php` | One or more validation rules failed |
| `FileSystemException` | `FileSystemException.php` | File system operation failed |
| `DirectoryNotFoundException` | `FileSystemException.php` | Expected directory does not exist |
| `FileNotFoundException` | `FileSystemException.php` | Expected file does not exist |
| `AlreadyExistsException` | `AlreadyExistsException.php` | Duplicate entry (unique constraint violation) |
| `NotFoundException` | `NotFoundException.php` | Resource not found (HTTP 404 equivalent) |
| `StateException` | `StateException.php` | Object is in wrong state for operation |
| `AuthorizationException` | `AuthorizationException.php` | Unauthorized access attempt |

### Class Diagram (Partial Hierarchy)

```
\Exception
└── \Magento\Framework\Exception\LocalizedException
    ├── \Magento\Framework\Exception\NoSuchEntityException
    │       └── NoSuchEntityException::singleField()
    │       └── NoSuchEntityException::doubleField()
    ├── \Magento\Framework\Exception\InputException
    │       └── InputException::invalidFieldValue()
    │       └── InputException::newEmpty()
    ├── \Magento\Framework\Exception\CouldNotSaveException
    ├── \Magento\Framework\Exception\CouldNotDeleteException
    ├── \Magento\Framework\Exception\ValidatorException
    ├── \Magento\Framework\Exception\AlreadyExistsException
    ├── \Magento\Framework\Exception\NotFoundException
    ├── \Magento\Framework\Exception\StateException
    └── \Magento\Framework\Exception\AuthorizationException
```

### Throwing Exceptions Correctly

The single most common mistake in Magento module development is throwing a raw `\Exception` instead of `LocalizedException` or one of its subclasses. This breaks the exception hierarchy in three ways:

1. **Logging**: Magento's `ExceptionHandler` only processes `LocalizedException` subclasses. Raw `\Exception` instances bypass the framework's error handling entirely and produce unpredictable behavior.
2. **Translation**: Only `LocalizedException` properly handles `Phrase` objects, so users see translated error messages.
3. **Plugin chain integrity**: Magento's plugin system relies on exception types to determine how exceptions propagate through `before`, `after`, and `around` plugin methods.

```php
// WRONG — breaks the exception hierarchy
throw new \Exception('Product not found');

// CORRECT — extends LocalizedException
throw new \Magento\Framework\Exception\NoSuchEntityException__(
    __('Product with ID %1 does not exist.', $productId)
);

// CORRECT — using the factory method
throw \Magento\Framework\Exception\NoSuchEntityException::singleField(
    'entity_id',
    $productId
);
```

---

## 2. LocalizedException — The Base Exception Class

`LocalizedException` is the cornerstone of Magento's internationalization-aware error handling. Understanding its design is critical because it governs how error messages appear to users in different locales and how exceptions are logged in production.

### The `__(string $message, $params = [])` Translation Pattern

Magento's translation system uses the double-underscore function `__()` to mark strings for translation. When you use `__()` inside an exception constructor, the string is captured as a `Phrase` object rather than a plain string:

```php
// Simple message with no parameters
throw new \Magento\Framework\Exception\LocalizedException(
    __('An error occurred while processing your request.')
);

// Message with positional parameters
throw new \Magento\Framework\Exception\LocalizedException(
    __('Customer with ID %1 does not exist.', $customerId)
);

// Message with named parameters (more readable for complex messages)
throw new \Magento\Framework\Exception\LocalizedException(
    __('Order %1 cannot be shipped to address in region %2.', [$orderId, $regionCode])
);

// Message with multiple parameters
throw new \Magento\Framework\Exception\LocalizedException(
    __('Product "%1" is out of stock. Requested quantity: %2, available: %3.', 
        [$productName, $requestedQty, $availableQty])
);
```

When the phrase is rendered, Magento's translator looks up the string in the active locale's CSV or JSON translation file. If no translation exists, the original English string is returned. This means your exception messages always appear in the user's language if a translation is available.

### Wrapping Other Exceptions

One of `LocalizedException`'s most important features is its ability to wrap a previous exception using the `$previous` parameter. This preserves the full stack trace of the original error while presenting a user-friendly message:

```php
try {
    $this->productRepository->save($product);
} catch (\PDOException $e) {
    // Wrap the database error in a LocalizedException
    // The original PDOException is preserved in $previous
    throw new \Magento\Framework\Exception\CouldNotSaveException(
        __('Could not save the product. Please try again later.'),
        $e
    );
}
```

In the example above, the `CouldNotSaveException` (which extends `LocalizedException`) wraps the original `PDOException`. When this exception is logged, both the user-facing message ("Could not save the product...") and the full original stack trace are written to `exception.log`. This is invaluable for debugging because you see exactly what the database error was without exposing that detail to the end user.

### Exception Chaining with `$previous`

The `$previous` parameter follows PHP's native exception chaining. When you catch and re-throw:

```php
try {
    $this->inventoryService->checkStock($sku);
} catch (\Magento\Framework\Exception\StockException $e) {
    // Add context before re-throwing
    throw new \Magento\Framework\Exception\InputException(
        __('Cannot check stock for SKU %1: %2', [$sku, $e->getMessage()]),
        $e
    );
}
```

The `getPrevious()` method returns the original exception, and `getMessage()` returns the combined message chain. You can iterate through the entire chain:

```php
catch (\Magento\Framework\Exception\LocalizedException $e) {
    $current = $e;
    while ($current !== null) {
        $this->logger->error($current->getMessage());
        $current = $current->getPrevious();
    }
}
```

### Why Phrase Matters for Translation

The critical difference between `LocalizedException` and a plain `Exception` is the `Phrase` object stored in `$this->phrase`. When you call `getMessage()`, you get the rendered (possibly translated) string. When you call `getPhrase()`, you get the raw `Phrase` object that can be re-rendered with a different locale context.

This distinction matters for API responses. If your exception travels through a REST endpoint, the framework can decide whether to include the translated message (for admin users) or a generic message (for external API consumers) by calling either `getPhrase()->render()` or `getMessage()`.

---

## 3. NoSuchEntityException

`NoSuchEntityException` is Magento's standard way of signaling "the entity you asked for does not exist." It has two factory methods that produce slightly different error messages and are appropriate for different lookup patterns.

### `NoSuchEntityException::singleField()`

Use this when looking up an entity by a single unique identifier:

```php
// Look up by entity ID
public function get(int $productId): \Magento\Catalog\Api\Data\ProductInterface
{
    $product = $this->productResource->load($productId);
    if (!$product || !$product->getId()) {
        throw \Magento\Framework\Exception\NoSuchEntityException::singleField(
            'entity_id',
            $productId
        );
    }
    return $product;
}

// Look up by a different unique field (e.g., SKU)
public function getBySku(string $sku): \Magento\Catalog\Api\Data\ProductInterface
{
    $product = $this->productResource->loadBySku($sku);
    if (!$product || !$product->getId()) {
        throw \Magento\Framework\Exception\NoSuchEntityException::singleField(
            'sku',
            $sku
        );
    }
    return $product;
}
```

The resulting exception message will be: `"No such entity with %1 = %2"` — for example: `"No such entity with sku = ABC123"`.

### `NoSuchEntityException::doubleField()`

Use this when looking up requires two fields together (composite key):

```php
// Look up by a combination of fields
public function getByCustomerAndOrder(int $customerId, int $orderId): OrderInterface
{
    $order = $this->orderResource->loadByCustomerAndOrder($customerId, $orderId);
    if (!$order || !$order->getId()) {
        throw \Magento\Framework\Exception\NoSuchEntityException::doubleField(
            'customer_id',
            $customerId,
            'order_id',
            $orderId
        );
    }
    return $order;
}
```

The resulting exception message will be: `"No such entity with %1 = %2, %3 = %4"` — for example: `"No such entity with customer_id = 5, order_id = 1024"`.

### When to Throw NoSuchEntityException vs Return Null

This is a common design question. Magento's consistent pattern is to **always throw `NoSuchEntityException`** rather than returning `null` from repository `get*` methods. The rationale:

1. **Calling code clarity**: A `try/catch` block is explicit about what can go wrong. Returning `null` requires the caller to always check for `null` — a step that is easily forgotten and causes `null pointer` errors downstream.
2. **Distinguishable errors**: A `NoSuchEntityException` is distinguishable from other errors (network failure, validation error, etc.). Returning `null` conflates "not found" with other failure modes.
3. **API consistency**: The repository interface contracts (in `Api/` folders) specify that `get*` methods throw `NoSuchEntityException`. Adhering to this contract ensures your module is consistent with Magento Core and third-party extensions.

```php
// CORRECT — throws exception for "not found"
public function get(int $customerId): CustomerInterface
{
    $customer = $this->customerResource->load($customerId);
    if (!$customer || !$customer->getId()) {
        throw new \Magento\Framework\Exception\NoSuchEntityException(
            __('Customer with ID %1 does not exist.', $customerId)
        );
    }
    return $customer;
}

// ACCEPTABLE — returns null only for list/search methods where "not found" is a valid empty result
public function findByEmail(string $email): ?CustomerInterface
{
    $collection = $this->customerCollectionFactory->create();
    $collection->addFieldToFilter('email', $email);
    $customer = $collection->getFirstItem();
    return $customer->getId() ? $customer : null;
}
```

In the second example, `findByEmail` returns `?CustomerInterface` because it is a search operation (returning the first match or null is a natural outcome). The `get(int $id)` method, by contrast, is an explicit fetch-by-ID and should throw if the ID does not exist.

---

## 4. InputException

`InputException` is used for validation errors — situations where the caller has passed data that fails validation. It is most appropriate in the API layer (REST, GraphQL) and admin form processors where user input must be validated before processing.

### `InputException::invalidFieldValue()`

Use this when a specific field has an invalid value:

```php
use Magento\Framework\Exception\InputException;
use Magento\Framework\Phrase;

public function createCustomer(array $customerData): int
{
    // Validate email format
    if (!filter_var($customerData['email'], FILTER_VALIDATE_EMAIL)) {
        throw InputException::invalidFieldValue(
            new Phrase('email'),
            $customerData['email']
        );
    }

    // Validate telephone has only digits and allowed characters
    if (!preg_match('/^\+?[0-9]{10,15}$/', $customerData['telephone'])) {
        throw InputException::invalidFieldValue(
            new Phrase('telephone'),
            $customerData['telephone']
        );
    }

    // Validate website_id is numeric and exists
    if (!is_numeric($customerData['website_id'])) {
        throw InputException::invalidFieldValue(
            new Phrase('website_id'),
            $customerData['website_id']
        );
    }
}
```

The resulting message: `"Invalid value of "john@example.com" provided for the email field."`

### `InputException::newEmpty()`

Use this when a required input is missing or empty:

```php
public function register(array $data): int
{
    $requiredFields = ['firstname', 'lastname', 'email', 'password'];

    foreach ($requiredFields as $field) {
        if (empty($data[$field])) {
            throw InputException::newEmpty(
                __($field)
            );
        }
    }

    // Alternative: collect all missing fields and report them together
    $missingFields = [];
    foreach ($requiredFields as $field) {
        if (empty($data[$field])) {
            $missingFields[] = $field;
        }
    }

    if (!empty($missingFields)) {
        throw new InputException(
            __('Required fields are missing: %1', implode(', ', $missingFields))
        );
    }
}
```

### InputException vs ValidatorException

`InputException` is for single-field validation errors (one specific field has an invalid value or is missing). `ValidatorException` is for complex validation scenarios where multiple fields may have errors, or where validation involves cross-field rules:

```php
use Magento\Framework\Exception\ValidatorException;

// InputException — single field
if (!is_numeric($orderId)) {
    throw InputException::invalidFieldValue(
        new Phrase('order_id'),
        $orderId
    );
}

// ValidatorException — multiple fields or complex rules
$errors = [];
if (!is_numeric($orderId)) {
    $errors[] = 'order_id must be numeric';
}
if ($quantity < 1 || $quantity > 1000) {
    $errors[] = 'quantity must be between 1 and 1000';
}
if ($quantity > $availableStock) {
    $errors[] = 'quantity exceeds available stock';
}

if (!empty($errors)) {
    throw new ValidatorException(
        __('Validation failed: %1', implode('; ', $errors))
    );
}
```

---

## 5. CouldNotSaveException and CouldNotDeleteException

These two exceptions exist specifically to wrap errors that occur within database transactions. Their purpose is to distinguish between "your input was bad" (which would be `InputException`) and "the database operation failed even though your input was valid" (which is `CouldNotSaveException` or `CouldNotDeleteException`).

### The Transactional Exception Pattern

The canonical pattern is to wrap a database error within a save or delete operation:

```php
use Magento\Framework\Exception\CouldNotSaveException;
use Magento\Framework\Exception\CouldNotDeleteException;

public function save(\Magento\Catalog\Api\Data\ProductInterface $product): int
{
    try {
        $this->productResource->beginTransaction();
        try {
            $this->productResource->save($product);
            $this->productResource->commit();
            return (int) $product->getId();
        } catch (\Exception $e) {
            $this->productResource->rollBack();
            throw $e; // Re-throw so the outer catch can wrap it
        }
    } catch (\Magento\Framework\DB\Adapter\PDOException $e) {
        throw new CouldNotSaveException(
            __('Could not save the product. Database error occurred.'),
            $e
        );
    } catch (\Exception $e) {
        throw new CouldNotSaveException(
            __('Could not save the product. Please try again later.'),
            $e
        );
    }
}

public function delete(int $productId): bool
{
    try {
        $product = $this->get($productId);
        $this->productResource->beginTransaction();
        try {
            $this->productResource->delete($product);
            $this->productResource->commit();
            return true;
        } catch (\Exception $e) {
            $this->productResource->rollBack();
            throw $e;
        }
    } catch (NoSuchEntityException $e) {
        throw $e; // Not found — let it propagate as-is
    } catch (\Exception $e) {
        throw new CouldNotDeleteException(
            __('Could not delete product %1. Please try again later.', $productId),
            $e
        );
    }
}
```

### When to Use CouldNotSave vs LocalizedException Directly

If your save operation is not part of a broader transaction context, it is acceptable to throw `LocalizedException` directly. Reserve `CouldNotSaveException` and `CouldNotDeleteException` for:

1. **Operations within explicit transactions** (where you call `beginTransaction()` / `commit()` / `rollBack()`)
2. **Repository layer saves/deletes** where the operation might be the first in a chain of operations (e.g., saving a product might also trigger price indexer updates, stock updates, etc.)
3. **Operations that reach the database** where you need to wrap a low-level database error (PDOException, Z派出 exception)

If you are just doing a simple save in a service class with no transaction context, throwing `LocalizedException` directly is simpler and equally correct.

---

## 6. Controller `noSuchAction`

When a controller receives a request for an action method that does not exist, Magento does not automatically return a 404. Instead, it calls a special `noSuchAction` method if it exists, or falls back to `forward('noroute')`. Understanding this mechanism is important for building robust controllers and REST endpoints.

### How Magento Dispatches Controller Actions

In `Magento\Framework\App\Action\Action::dispatch()`, the framework checks if the requested action method exists on the controller:

```php
// lib/internal/Magento/Framework/App/Action/Action.php (simplified)
public function dispatch(\Magento\Framework\App\RequestInterface $request)
{
    // ...
    $action = $request->getActionName();
    $method = $this->actionMap[$action] ?? null;

    if (!$method || !method_exists($this, $method)) {
        // Action not found — forward to 'noroute'
        return $this->resultForwardFactory->create()->forward('noroute');
    }

    return $this->$method($request);
}
```

### Implementing `noSuchAction` in Your Controller

If you want custom behavior when an invalid action is requested, implement a `noSuchAction` method:

```php
// app/code/YourVendor/YourModule/Controller/Adminhtml/Product/NoSuchAction.php
declare(strict_types=1);

namespace YourVendor\YourModule\Controller\Adminhtml\Product;

use Magento\Framework\Controller\Result\ForwardFactory;
use Magento\Framework\Controller\Result\Forward;

class NoSuchAction extends \Magento\Backend\App\Action
{
    protected $resultForwardFactory;

    public function __construct(
        \Magento\Backend\App\Action\Context $context,
        ForwardFactory $resultForwardFactory
    ) {
        parent::__construct($context);
        $this->resultForwardFactory = $resultForwardFactory;
    }

    /**
     * Handle requests for non-existent action methods.
     */
    public function execute()
    {
        /** @var Forward $resultForward */
        $resultForward = $this->resultForwardFactory->create();
        return $resultForward->forward('noroute');
    }
}
```

### ResultForward Factory Pattern

`Magento\Framework\Controller\Result\ForwardFactory` creates a `ResultForward` object that, when told to `forward('noroute')`, creates a new request for the `noroute` action of the current controller. This isMagento's internal routing mechanism for reusing existing action logic.

The `noroute` action typically renders a 404 page. For frontend controllers, this maps to the CMS "404 Page Not Found" CMS page. For admin controllers, it typically renders an error block.

### When to Throw an Exception vs Forward to noroute

The distinction matters:

| Scenario | Approach |
|---|---|
| Entity ID in URL does not exist | Throw `NoSuchEntityException` — more specific, produces better error messages |
| Action method does not exist | `forward('noroute')` — indicates URL routing problem |
| User lacks permission for action | Throw `AuthorizationException` or `forward('denied')` |
| Input validation fails | Throw `InputException` or `ValidatorException` |
| Operation fails due to system error | Throw `LocalizedException` or `CouldNotSaveException` |

The reason you might prefer throwing `NoSuchEntityException` over `forward('noroute')` for missing entities is that the exception carries semantic meaning ("entity 12345 not found") that is useful for logging, debugging, and API responses. A forward to noroute is more generic and loses that context.

### REST API — NoSuchEntityException Becomes 404

In the REST API layer, `NoSuchEntityException` is automatically converted to an HTTP 404 response:

```json
{
    "message": "No such entity with customer_id = 99999",
    "parameters": {
        "field": "customer_id",
        "fieldValue": 99999
    }
}
```

This automatic conversion is handled by `Magento\Webapi\Model\Soap\Fault` and `Magento\Framework\Webapi\ExceptionHandler`, which we will cover in detail later in this chapter.

---

## 7. ExceptionHandler

Magento's `ExceptionHandler` is the framework's internal mechanism for converting exceptions into responses. Understanding how it works is essential for both debugging production issues (why does this error appear the way it does?) and for building custom error handling.

### The ExceptionHandler Interface

```php
// lib/internal/Magento/Framework/ExceptionHandler/ExceptionHandler.php
declare(strict_types=1);

namespace Magento\Framework\ExceptionHandler;

use Psr\Log\LoggerInterface;

/**
 * Interface for exception handlers that process exceptions and produce error responses.
 */
interface ExceptionHandlerInterface
{
    /**
     * Handle an exception and produce an HTTP response or logged output.
     *
     * @param \Throwable $exception
     * @param LoggerInterface $logger
     * @return void
     */
    public function handle(\Throwable $exception, LoggerInterface $logger): void;
}
```

The framework registers multiple exception handlers in `di.xml` that form a chain. When an exception propagates out of a controller, each handler in the chain gets a chance to process it.

### The Default ExceptionHandler

`Magento\Framework\ExceptionHandler\ExceptionHandler` is the default handler that writes exceptions to `var/log/exception.log`:

```php
// lib/internal/Magento/Framework/ExceptionHandler/ExceptionHandler.php
declare(strict_types=1);

namespace Magento\Framework\ExceptionHandler;

use Psr\Log\LoggerInterface;

class ExceptionHandler implements ExceptionHandlerInterface
{
    /**
     * @var LoggerInterface
     */
    private $logger;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    /**
     * Log the exception to var/log/exception.log.
     */
    public function handle(\Throwable $exception, LoggerInterface $logger): void
    {
        $logger->error($exception->getMessage(), ['exception' => $exception]);
    }
}
```

In developer mode, this handler also displays the full stack trace to the browser. In production mode, it logs only — the user sees a generic error page.

### The WebapiExceptionHandler

`Magento\Framework\Webapi\ExceptionHandler` is the handler responsible for API exceptions. It is registered in `Magento\Webapi\Model\Soap\Fault` and is responsible for converting Magento exceptions into proper HTTP status codes and JSON error responses.

```php
// lib/internal/Magento/Framework/Webapi/ExceptionHandler.php
declare(strict_types=1);

namespace Magento\Framework\Webapi;

use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Phrase;
use Psr\Log\LoggerInterface;

class ExceptionHandler
{
    /**
     * Map Magento exception classes to HTTP status codes.
     *
     * @var array
     */
    private $exceptionHttpCodes = [
        \Magento\Framework\Exception\NoSuchEntityException::class => 404,
        \Magento\Framework\Exception\InputException::class => 400,
        \Magento\Framework\Exception\AuthorizationException::class => 403,
        \Magento\Framework\Exception\StateException::class => 409,
        \Magento\Framework\Exception\CouldNotSaveException::class => 500,
        \Magento\Framework\Exception\CouldNotDeleteException::class => 500,
        \Magento\Framework\Exception\ValidatorException::class => 400,
    ];

    /**
     * Handle an API exception and produce a JSON error response.
     */
    public function handle(
        \Throwable $exception,
        LoggerInterface $logger,
        ?string $areaCode = null
    ): \Magento\Framework\Webapi\Response {
        $httpCode = $this->getHttpCode($exception);
        $isDevMode = $this->isDeveloperMode($areaCode);

        // Build the error response
        $response = new \Magento\Framework\Webapi\Response();
        $response->setHttpResponseCode($httpCode);
        $response->setHeader('Content-Type', 'application/json', true);

        $errorData = $this->prepareErrorData($exception, $isDevMode);
        $response->setBody(json_encode($errorData, JSON_PRETTY_PRINT));

        return $response;
    }

    /**
     * Determine HTTP status code from exception type.
     */
    private function getHttpCode(\Throwable $exception): int
    {
        foreach ($this->exceptionHttpCodes as $exceptionClass => $httpCode) {
            if ($exception instanceof $exceptionClass) {
                return $httpCode;
            }
        }
        return 500;
    }

    /**
     * Prepare error data for JSON response.
     */
    private function prepareErrorData(\Throwable $exception, bool $isDevMode): array
    {
        $errorData = [
            'message' => $exception instanceof LocalizedException
                ? $exception->getPhrase()->render()
                : $exception->getMessage(),
        ];

        if ($exception instanceof LocalizedException && $exception->getPhrase()) {
            $errorData['parameters'] = $this->extractParameters($exception->getPhrase());
        }

        if ($isDevMode) {
            $errorData['trace'] = $exception->getTraceAsString();
        }

        return $errorData;
    }

    private function isDeveloperMode(?string $areaCode): bool
    {
        return $areaCode === 'frontend'
            || $areaCode === 'adminhtml'
            || getenv('MAGE_MODE') === 'developer';
    }
}
```

### JSON Error Format in API Responses

A typical REST API error response looks like this:

```json
// GET /rest/V1/products/99999 → 404 Not Found
{
    "message": "No such entity with entity_id = 99999",
    "parameters": {
        "field": "entity_id",
        "fieldValue": 99999
    }
}
```

```json
// POST /rest/V1/products → 400 Bad Request
{
    "message": "Invalid value of \"not-an-email\" provided for the email field.",
    "parameters": {
        "field": "email",
        "fieldValue": "not-an-email"
    }
}
```

```json
// POST /rest/V1/products → 500 Internal Server Error (production mode)
{
    "message": "An error has occurred. Please try again later."
}
```

Notice that in production mode (non-developer), the internal error details are stripped. This is the `maskError()` behavior that we examine next.

---

## 8. WebapiExceptionHandler — maskError() and Debug Tokens

The `WebapiExceptionHandler` has a critical method called `maskError()` that strips sensitive internal information from API error responses when running in production mode. This is a key security feature.

### The maskError() Method

```php
// lib/internal/Magento/Framework/Webapi/ExceptionHandler.php (relevant portion)
private function maskError(\Throwable $exception, bool $isDebugMode): array
{
    if ($isDebugMode) {
        // Developer mode: return full details
        return [
            'message' => $exception->getMessage(),
            'trace' => $exception->getTraceAsString(),
            'previous' => $this->formatPrevious($exception),
        ];
    }

    // Production mode: mask the error
    $maskedMessage = $this->getMaskedMessage($exception);
    return [
        'message' => $maskedMessage,
    ];
}

private function getMaskedMessage(\Throwable $exception): string
{
    // For known Magento exceptions, provide a generic but useful message
    if ($exception instanceof NoSuchEntityException) {
        return __('An error occurred. Please refer to the error code for details.')->render();
    }

    if ($exception instanceof InputException) {
        return __('Invalid input data. Please check your request parameters.')->render();
    }

    if ($exception instanceof AuthorizationException) {
        return __('Access denied. Please check your credentials.')->render();
    }

    // Generic message for unknown exceptions
    return __('An error has occurred. Please try again later.')->render();
}
```

### The X-Debug-Token Header System

In production, when an error occurs, the full details are logged to `var/log/exception.log` with a unique error report ID. If you need to retrieve those details, you include the `X-Debug-Token` header in your request — the same token is returned in the error response's `X-Debug-Token` header:

```bash
# Request with debug token (production)
curl -X GET https://yourstore.example.com/rest/V1/products/99999 \
     -H "Authorization: Bearer $TOKEN" \
     -H "X-Debug-Token: abc123"

# Response in production (masked)
# Headers:
# X-Debug-Token: xyz789
# HTTP/1.1 404 Not Found
{
    "message": "An error occurred. Please refer to the error code for details."
}

# To get the full error details:
# 1. SSH to the server
# 2. Look in var/report/xyz789 (the error report file)
# 3. Or use the debug token to retrieve via admin:
#    bin/magento dev:report:show xyz789
```

### How Debug Tokens Work

When a report is created, it is stored with a unique identifier. The `X-Debug-Token` header in the response tells the API consumer which report file to look up. In a typical production setup, only internal admins have access to read these report files via `bin/magento dev:report:show`.

```bash
# On the server, view the error report
cat var/report/xyz789

# Or use the CLI to show the report
bin/magento dev:report:show xyz789
```

This two-tier system (masked message in the response + full details in a secure report file) is deliberate: external API consumers get a safe generic message, while internal developers can retrieve full stack traces when needed.

---

## 9. Developer Mode

Magento's `MAGE_MODE` setting fundamentally changes how exceptions are handled. Understanding what gets shown vs hidden in each mode is critical for both development productivity and production security.

### How Developer Mode Affects Exception Handling

In `lib/internal/Magento/Framework/App/Bootstrap::createExceptionHandler()`, Magento registers an error handler that behaves differently based on the mode:

```php
// lib/internal/Magento/Framework/App/Bootstrap.php (conceptual)
public static function createExceptionHandler(string $mageMode): callable
{
    if ($mageMode === 'developer') {
        // In developer mode: display full errors in browser
        return function (\Throwable $exception) {
            echo '<h1>Exception Handler</h1>';
            echo '<pre>' . $exception->getMessage() . '</pre>';
            echo '<pre>' . $exception->getTraceAsString() . '</pre>';
            // Also log to exception.log
            $logger = \Magento\Framework\App\ObjectManager::getInstance()
                ->get(\Psr\Log\LoggerInterface::class);
            $logger->error($exception->getMessage(), ['exception' => $exception]);
        };
    }

    // In production/default mode: log only
    return function (\Throwable $exception) {
        $logger = \Magento\Framework\App\ObjectManager::getInstance()
            ->get(\Psr\Log\LoggerInterface::class);
        $logger->critical($exception->getMessage(), ['exception' => $exception]);
    };
}
```

### What Developer Mode Shows vs Hides

| Information | Developer Mode | Production Mode |
|---|---|---|
| Exception message | Full | Masked generic message |
| Stack trace | Full, in browser | Logged only to `exception.log` |
| File paths | Full paths visible | Paths hidden (stripped in logs) |
| Database queries | Visible in stack trace for DB errors | Logged only |
| `$_POST` / `$_GET` data | Visible in some error contexts | Never shown |
| Internal class names | Visible in stack trace | Masked in API responses |
| Line numbers | Visible | Logged only |

### Setting the Mode

```bash
# Check current mode
bin/magento deploy:mode:show

# Switch to production mode
bin/magento deploy:mode:set production

# Switch to developer mode (NEVER do this on production)
bin/magento deploy:mode:set developer
```

Or via environment variable in your web server configuration:

```apache
# Apache .htaccess or vhost
SetEnv MAGE_MODE developer
```

```nginx
# Nginx configuration
fastcgi_param MAGE_MODE developer;
```

### env.php Mode Configuration

```php
// app/etc/env.php
return [
    // ...
    'MAGE_MODE' => 'production',
    // ...
];
```

If `MAGE_MODE` is not set in `env.php`, Magento falls back to auto-detecting from the environment variable `MAGE_MODE`.

### Exception Display in Developer vs Production

**Developer mode** — an unhandled `NoSuchEntityException` in a frontend controller:

```
Magento\Framework\Exception\NoSuchEntityException: No such entity with entity_id = 99999
in /var/www/html/app/code/Magento/Catalog/Model/ProductRepository.php:123
Stack trace:
#0 /var/www/html/app/code/Magento/Catalog/Controller/Product/View.php(45): Magento\Catalog\Model\ProductRepository->get(99999)
#1 ...
```

**Production mode** — the same exception becomes:

```
An error occurred. Please try again later.
```

And in `var/log/exception.log`:

```
[2025-04-21 14:32:15] main.CRITICAL: No such entity with entity_id = 99999
{"exception":"Magento\\Framework\\Exception\\NoSuchEntityException ..."}
```

---

## 10. Error Logging

Magento 2.4.8 writes exceptions and errors to multiple log files and a dedicated `var/report/` directory. Understanding what goes where and when is essential for production debugging.

### Log Files

| File | What Goes Here | When Written |
|---|---|---|
| `var/log/exception.log` | All unhandled exceptions (exceptions that propagate to the top level) | Always (if logging is enabled) |
| `var/log/system.log` | General system messages, warnings, notices from `PSLogger` | Always |
| `var/log/debug.log` | Debug-level messages (only in developer/default mode, or when explicitly enabled) | Only when `dev/debug_log` is on |
| `var/log/先祖.log` | Deprecation warnings | Always |
| `var/report/` | Full HTML error reports with complete stack traces | On any error reaching the `report` system |

### The Report Directory

The `var/report/` directory is unique. Rather than a log file, each error generates a separate file named with a numeric ID:

```bash
# List recent error reports
ls -la var/report/ | tail -10

# View a specific report
cat var/report/1234567890

# Clean old reports
find var/report/ -type f -mtime +30 -delete
```

A report file contains the full HTML page that would have been shown in developer mode — complete stack trace, request parameters, POST data, registered shutdown functions, and memory usage at the time of the error. This is the most detailed form of error information available in Magento.

### What Triggers a Report

A report is created whenever an exception reaches `Magento\Framework\App\Report` or when `pub/errors/processor.php` handles an error. The threshold configuration controls whether a report is created:

```bash
# Default threshold is 0 (always create report for errors)
# You can configure in dev/etc/di.xml or env.php:

# To see the current threshold:
bin/magento config:show dev/reporting/threshold

# To change the threshold (size in bytes):
bin/magento config:set dev/reporting/threshold 1024
```

### dev/report/threshold Configuration

```php
// In env.php or etc/env.php
'system' => [
    'default' => [
        'dev' => [
            'reporting' => [
                'threshold' => 0  // 0 = always create report, > 0 = only for errors above this size
            ]
        ]
    ]
],
```

In practice, the threshold is rarely changed from the default of 0. Every error should produce a report in production so that there is a complete record of what went wrong.

### Exception Logging Pattern

When you throw an exception that is meant to be handled by the framework, you generally do NOT need to log it yourself — the framework handles that. However, when you catch an exception and re-throw a different one, you might want to log the original:

```php
try {
    $this->externalService->call($params);
} catch (\GuzzleHttp\Exception\ConnectException $e) {
    // Log the connectivity issue before wrapping it
    $this->logger->warning(
        'External service connectivity issue',
        [
            'service' => 'PaymentGateway',
            'exception' => $e,
            'params' => $params
        ]
    );
    throw new \Magento\Framework\Exception\LocalizedException(
        __('Payment service is temporarily unavailable. Please try again.'),
        $e
    );
}
```

---

## 11. Custom Exception Classes

For most modules, using Magento's built-in exception classes is sufficient. However, creating a custom exception class becomes valuable when you need to communicate module-specific error semantics that are meaningful to other extensions or to API consumers.

### Creating a Module-Specific Exception

```php
// app/code/YourVendor/YourModule/Exception/YourModuleException.php
declare(strict_types=1);

namespace YourVendor\YourModule\Exception;

use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Phrase;

/**
 * Base exception class for YourVendor_YourModule.
 *
 * All exceptions thrown by YourModule should extend this class
 * or one of its subclasses.
 */
class YourModuleException extends LocalizedException
{
    /**
     * Exception codes for programmatic identification.
     */
    public const CODE_VALIDATION_FAILED = 1001;
    public const CODE_INTEGRATION_ERROR = 1002;
    public const CODE_CONFIGURATION_ERROR = 1003;

    /**
     * @param Phrase $phrase
     * @param int $code
     * @param \Throwable|null $previous
     */
    public function __construct(
        Phrase $phrase,
        int $code = 0,
        ?\Throwable $previous = null
    ) {
        parent::__construct($phrase, $previous);
        $this->code = $code;
    }
}
```

### A Concrete Custom Exception

```php
// app/code/YourVendor/YourModule/Exception/PaymentProcessingException.php
declare(strict_types=1);

namespace YourVendor\YourModule\Exception;

use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Phrase;

/**
 * Exception thrown when payment processing fails.
 */
class PaymentProcessingException extends YourModuleException
{
    /**
     * @param Phrase $phrase
     * @param \Throwable|null $previous
     */
    public function __construct(Phrase $phrase, ?\Throwable $previous = null)
    {
        parent::__construct($phrase, self::CODE_VALIDATION_FAILED, $previous);
    }

    /**
     * Factory method for card declined scenario.
     */
    public static function cardDeclined(string $cardType, string $lastFour): self
    {
        return new self(
            __('Payment declined. Card type: %1, last four digits: %2', $cardType, $lastFour)
        );
    }

    /**
     * Factory method for insufficient funds scenario.
     */
    public static function insufficientFunds(string $amount, string $balance): self
    {
        return new self(
            __('Insufficient funds. Requested: %1, available: %2', $amount, $balance)
        );
    }
}
```

### When to Create vs Reuse

| Situation | Approach |
|---|---|
| Simple "not found" | Use `NoSuchEntityException::singleField()` |
| Simple validation error | Use `InputException` or `ValidatorException` |
| Database save failure (transactional) | Use `CouldNotSaveException` |
| Authorization failure | Use `AuthorizationException` |
| Complex domain logic with multiple error states | Create a custom exception with factory methods |
| API needs a specific error format | Create a custom exception and register an ExceptionHandler for it |
| Third-party integration errors | Wrap in a custom integration exception |

### Naming Conventions

Magento's convention for exception class names is `Exception` suffix. The class should be in an `Exception/` subdirectory of your module's `Model/` or root directory:

```
app/code/YourVendor/YourModule/
├── Exception/
│   ├── YourModuleException.php          (base class)
│   └── PaymentProcessingException.php   (specific)
├── Model/
│   └── PaymentService.php
```

---

## 12. Try-Catch Patterns

How you structure try-catch blocks determines whether errors are handled gracefully or hidden from developers and users alike. This section covers best practices and anti-patterns.

### When to Catch LocalizedException

**Catch at the appropriate boundary** — typically at the controller or service layer, not deep inside repositories:

```php
// Controller — catch all expected exceptions and convert to HTTP responses
public function execute()
{
    try {
        $customerId = $this->customerRepository->get($customerId);
        $this->customerService->notify($customerId);
        return $this->jsonResponse->success(['customer_id' => $customerId]);
    } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
        return $this->jsonResponse->error($e->getMessage(), 404);
    } catch (\Magento\Framework\Exception\InputException $e) {
        return $this->jsonResponse->error($e->getMessage(), 400);
    } catch (\Exception $e) {
        $this->logger->critical($e->getMessage(), ['exception' => $e]);
        return $this->jsonResponse->error(
            __('An unexpected error occurred. Please try again later.'),
            500
        );
    }
}
```

### Letting Exceptions Propagate

**Do not catch exceptions unnecessarily.** If a repository method throws `NoSuchEntityException` and your service layer cannot reasonably handle "not found," let the exception propagate:

```php
// Service layer — do NOT catch NoSuchEntityException here
// Let it propagate to the controller or API layer
public function processCustomerOrder(int $customerId, int $orderId): Order
{
    // These should NOT be caught here — caller needs to handle "not found"
    $customer = $this->customerRepository->get($customerId);
    $order = $this->orderRepository->get($orderId);

    // Business logic that might throw CouldNotSaveException
    $this->orderProcessor->process($order, $customer);

    return $order;
}
```

### The Fallback catch (\Exception $e)

Always include a fallback catch for `\Exception` after your specific catches. This prevents truly unexpected errors from leaking out unhandled:

```php
try {
    $result = $this->someOperation();
} catch (\Magento\Framework\Exception\LocalizedException $e) {
    // Handle known Magento exceptions
    $this->messageManager->addErrorMessage($e->getMessage());
} catch (\Exception $e) {
    // CRITICAL: Catch everything else to prevent silent failures
    $this->logger->critical(
        'Unexpected error in ' . __METHOD__,
        [
            'exception' => $e,
            'operation' => 'someOperation',
        ]
    );
    $this->messageManager->addErrorMessage(
        __('An unexpected error occurred. Please contact support.')
    );
}
```

### The Anti-Pattern: Catch and Re-Throw Without Context

The most dangerous anti-pattern is catching an exception and re-throwing without adding context:

```php
// WRONG — loses the original exception's context
try {
    $this->productRepository->save($product);
} catch (\Exception $e) {
    throw new \Magento\Framework\Exception\LocalizedException(
        __('An error occurred.')
    );
}

// CORRECT — preserves and enhances context
try {
    $this->productRepository->save($product);
} catch (\Magento\Framework\Exception\LocalizedException $e) {
    // Re-throw Magento exceptions as-is (they already have good context)
    throw $e;
} catch (\Exception $e) {
    // Wrap other exceptions with context
    throw new \Magento\Framework\Exception\CouldNotSaveException(
        __('Could not save product %1.', $product->getSku()),
        $e
    );
}
```

### Catching Multiple Exception Types

PHP 7.1+ supports catching multiple exception types with the `|` operator:

```php
try {
    $this->paymentService->process($order);
} catch (\Magento\Framework\Exception\CouldNotSaveException |
          \Magento\Framework\Exception\CouldNotDeleteException $e) {
    // Handle transaction-related failures
    $this->logger->critical($e->getMessage(), ['exception' => $e]);
    throw $e;
} catch (\Magento\Framework\Exception\LocalizedException $e) {
    // Handle Magento-specific exceptions
    $this->messageManager->addErrorMessage($e->getMessage());
} catch (\Throwable $e) {
    // Handle anything else
    $this->logger->critical($e);
    throw new \Magento\Framework\Exception\LocalizedException(
        __('An unexpected error occurred.'),
        $e
    );
}
```

---

## 13. Plugin Exception Handling

Plugin exception handling has nuanced behavior depending on whether the exception is thrown in a `before*`, `after*`, or `around*` plugin method. Understanding this is critical because exceptions in plugins can silently break the plugin chain.

### Exceptions in after* Plugins

An exception in an `after*` plugin method propagates back to the calling code. If the `after*` method is decorating a return value, any exception will prevent the original return value from being returned to the caller:

```php
// Subject class
class PriceService
{
    public function getPrice(int $productId): float
    {
        return $this->priceRepository->getPrice($productId);
    }
}

// after* plugin — THROWING AN EXCEPTION HERE IS DANGEROUS
class PriceServiceAfterPlugin
{
    public function afterGetPrice(PriceService $subject, float $price, int $productId): float
    {
        // If this throws, $price is never returned to the caller
        if ($price < 0) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Price cannot be negative for product %1.', $productId)
            );
        }
        return $price;
    }
}
```

The exception thrown in `afterGetPrice` propagates through the plugin chain back to the original caller. The caller of `PriceService::getPrice()` will receive the exception, not the price.

### Exceptions in around* Plugins

An exception in an `around*` plugin's **inner callable call** (the `proceed()` call) propagates normally. But if the exception is thrown in the code before or after the `proceed()` call, it behaves differently:

```php
class PriceServiceAroundPlugin
{
    public function aroundGetPrice(
        PriceService $subject,
        callable $proceed,
        int $productId
    ): float {
        // Code before proceed() — exceptions here propagate normally
        $this->logger->debug('Before getPrice');

        try {
            $price = $proceed($productId); // Call the original method
        } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
            // Exceptions from proceed() can be caught and handled
            $this->logger->warning('Product not found: ' . $productId);
            throw $e; // Re-throw after logging
        }

        // Code after proceed() — exceptions here also propagate normally
        $this->logger->debug('After getPrice, price: ' . $price);

        if ($price <= 0) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Invalid price for product %1.', $productId)
            );
        }

        return $price;
    }
}
```

### Plugin Chain Exception Propagation

When multiple plugins exist on the same method, the exception behavior follows this pattern:

1. `before*` plugins execute first. Any exception stops the chain immediately.
2. If all `before*` plugins succeed, the `around*` plugins execute (in sequence, each wrapping the next).
3. The innermost `around*` plugin calls `proceed()`, which invokes the original method.
4. `after*` plugins execute after the original method (and inner around plugins) return successfully.
5. Any exception in `after*` propagates back through the chain.

The practical implication: **do not throw exceptions in `after*` plugins unless you are certain the caller can handle them.** If an `after*` plugin throws, the caller receives an exception instead of a return value, which can cause `TypeError` or `null` reference errors in downstream code.

### ResultInterface Exception Handling in Controllers

Controllers that return `ResultInterface` objects do not directly throw exceptions from plugin chains. Instead, they return error result objects:

```php
// Returning an error via ResultForward (404)
public function execute(): ResultInterface
{
    if (!$this->product->getId()) {
        /** @var Forward $resultForward */
        $resultForward = $this->resultForwardFactory->create();
        return $resultForward->forward('noroute');
    }
    // ...
}

// Returning an error via ResultJson (400)
public function execute(): ResultInterface
{
    if (!validateInput($data)) {
        /** @var Json $resultJson */
        $resultJson = $this->resultJsonFactory->create();
        $resultJson->setHttpResponseCode(400);
        $resultJson->setData([
            'message' => __('Invalid input data.'),
        ]);
        return $resultJson;
    }
    // ...
}
```

The `ResultInterface` approach is preferred in controllers because it keeps the HTTP response handling centralized and makes it easy to test whether a controller returns a success or error response.

---

## 14. HTTP Error Responses

Magento's framework provides standardized ways to return error responses from controllers, both for HTML (frontend/adminhtml) and for APIs (REST/GraphQL).

### ResultForward — Rendering 404 Pages

`Magento\Framework\Controller\Result\Forward` is used to forward to another controller action. The most common use is forwarding to `noroute`:

```php
use Magento\Framework\Controller\Result\ForwardFactory;

public function __construct(
    ForwardFactory $resultForwardFactory
) {
    $this->resultForwardFactory = $resultForwardFactory;
}

public function execute(): ResultInterface
{
    $product = $this->productRepository->get($productId);
    if (!$product->getId()) {
        return $this->resultForwardFactory->create()->forward('noroute');
    }
    // ...
}
```

### ResultRedirect — Redirecting After Error

`Magento\Framework\Controller\Result\Redirect` is used when you want to redirect the user to another page after an error (e.g., redirect back to the form with an error message):

```php
use Magento\Framework\Controller\Result\RedirectFactory;

public function execute(): ResultInterface
{
    if (!$this->formKeyValidator->validate($this->getRequest())) {
        $this->messageManager->addErrorMessage(__('Invalid form key.'));
        /** @var Redirect $resultRedirect */
        $resultRedirect = $this->resultRedirectFactory->create();
        return $resultRedirect->setPath('*/*/index');
    }
    // ...
}
```

### ResultJson — JSON API Responses

`Magento\Framework\Controller\Result\Json` is used for AJAX endpoints and JSON APIs:

```php
use Magento\Framework\Controller\Result\JsonFactory;

public function __construct(
    JsonFactory $resultJsonFactory
) {
    $this->resultJsonFactory = $resultJsonFactory;
}

public function execute(): ResultInterface
{
    try {
        $data = $this->service->getData($id);
        return $this->resultJsonFactory->create()->setData([
            'success' => true,
            'data' => $data,
        ]);
    } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
        $result = $this->resultJsonFactory->create();
        $result->setHttpResponseCode(404);
        $result->setData([
            'success' => false,
            'message' => $e->getMessage(),
        ]);
        return $result;
    }
}
```

### HTTP Bad Request Helper

`Magento\Framework\Controller\Result\ForwardFactory` and `Magento\Framework\Controller\Result\JsonFactory` have no built-in `httpBadRequest()` method. The common pattern is:

```php
$resultJson = $this->resultJsonFactory->create();
$resultJson->setHttpResponseCode(400);
$resultJson->setData([
    'message' => __('Invalid parameter: %1', $paramName),
    'parameters' => ['field' => $paramName, 'value' => $paramValue],
]);
return $resultJson;
```

### REST/GraphQL Error Responses

For REST endpoints, exceptions thrown in service classes are automatically converted by `WebapiExceptionHandler`:

```json
// Exception: NoSuchEntityException → HTTP 404
{
    "message": "No such entity with customer_id = 123",
    "parameters": {
        "field": "customer_id",
        "fieldValue": 123
    }
}
```

```json
// Exception: InputException → HTTP 400
{
    "message": "Invalid value of \"bad-email\" provided for the email field.",
    "parameters": {
        "field": "email",
        "fieldValue": "bad-email"
    }
}
```

```json
// Exception: CouldNotSaveException → HTTP 500
{
    "message": "An error has occurred. Please try again later."
}
```

GraphQL errors follow a similar pattern but wrapped in the GraphQL error format:

```json
{
    "errors": [
        {
            "message": "No such entity with product_id = 99999",
            "extensions": {
                "code": "NO_SUCH_ENTITY",
                "category": "internal"
            },
            "locations": [{"line": 3, "column": 5}],
            "path": ["products"]
        }
    ],
    "data": null
}
```

---

## 15. Phrase vs Strings in Exceptions

A subtle but important distinction in Magento 2.4.8 exception handling is the difference between using a raw string and a `Phrase` object when constructing exceptions. Using `__('string')` (which creates a `Phrase`) is almost always the correct choice.

### What is Phrase?

```php
// lib/internal/Magento/Framework/Phrase.php
declare(strict_types=1);

namespace Magento\Framework;

class Phrase
{
    private $text;
    private $arguments;

    public function __construct(string $text, array $arguments = [])
    {
        $this->text = $text;
        $this->arguments = $arguments;
    }

    /**
     * Render the phrase with its arguments.
     */
    public function render(): string
    {
        if (empty($this->arguments)) {
            return $this->text;
        }
        return sprintf(str_replace('%', '%%', $this->text), ...array_values($this->arguments));
    }
}
```

`Phrase` is a simple text rendering class that stores a template string and optional arguments. Its key feature is that the rendering is lazy — the string is only rendered when `render()` is called.

### Why `__('message')` is Correct for Exceptions

```php
// WRONG — raw string, cannot be translated
throw new \Magento\Framework\Exception\LocalizedException(
    'Customer with ID ' . $customerId . ' does not exist.'
);

// CORRECT — Phrase, can be translated
throw new \Magento\Framework\Exception\LocalizedException(
    __('Customer with ID %1 does not exist.', $customerId)
);
```

The second form creates a `Phrase` object. When `getMessage()` is called on the exception, the phrase is rendered in the current locale. If no translation exists for the current locale, the original English string is returned.

### When to Use Raw Strings in Exceptions

There are rare cases where a raw string is appropriate:

1. **Logging-only exceptions**: If you are throwing an exception purely for logging purposes and the message will never be shown to a user:
    ```php
    // Only used for catching and re-throwing with a localized message
    catch (\Exception $e) {
        throw new \Magento\Framework\Exception\LocalizedException(
            'Original error: ' . $e->getMessage() // OK — internal context
        );
    }
    ```

2. **Dynamic content that should not be translatable**: If the message contains dynamically generated content that should not be in translation files (e.g., actual SQL queries, XML snippets):
    ```php
    catch (\Exception $e) {
        throw new \Magento\Framework\Exception\LocalizedException(
            new \Magento\Framework\Phrase('Raw error: ' . $rawData)
        );
    }
    ```

3. **Performance-critical paths**: In extremely hot code paths where the overhead of creating a `Phrase` object is measurable. In practice, this is almost never a real concern.

### Phrase and Exception Parameters

When using `Phrase` with parameters, use `%N` placeholders (not `{N}` or `{N}s`):

```php
// CORRECT — sprintf-style placeholders
__('Product %1 is out of stock. Requested: %2, Available: %3', [$sku, $requested, $available])

// WRONG — printf-style placeholders (%s) can cause issues
__('Product %s is out of stock', $sku)

// WRONG — brace-style placeholders are not supported
__('Product {sku} is out of stock')
```

---

## 16. Silent Failures — Empty Catch Blocks

Empty catch blocks are among the most dangerous anti-patterns in software development. They silently swallow exceptions, making debugging nearly impossible and allowing corrupted or incomplete state to propagate through the system.

### The Anti-Pattern

```php
// DANGEROUS — silently swallows ALL exceptions
try {
    $this->paymentService->charge($order);
} catch (\Exception $e) {
    // Nothing — exception is silently swallowed
}

// DANGEROUS — logs but continues as if nothing happened
try {
    $this->paymentService->charge($order);
} catch (\Exception $e) {
    $this->logger->warning('Payment charge failed: ' . $e->getMessage());
    // Execution continues even though payment failed
}

// STILL DANGEROUS — only catches specific type, everything else is swallowed
try {
    $this->paymentService->charge($order);
} catch (\Magento\Framework\Exception\LocalizedException $e) {
    // Catches only Magento exceptions — PDOException, Guzzle exceptions, etc. are still swallowed
}
```

### Why Silent Failures are Dangerous

1. **No user feedback**: The user has no idea the operation failed. They may believe their payment went through when it did not.
2. **Corrupted state**: If you catch an exception but continue executing as if nothing happened, your system enters an inconsistent state.
3. **Debugging impossible**: When something goes wrong later (e.g., the order shows as paid but the payment never went through), there is no exception to trace back to the root cause.
4. **Security risk**: Silently catching `AuthorizationException` or `AuthenticationException` could allow unauthorized operations to proceed.

### Graceful Failure Patterns

If you need to handle a failure gracefully without propagating an exception, do so explicitly:

```php
// GOOD — catch, log, set state, return gracefully
try {
    $this->paymentService->charge($order);
    $order->setPaymentStatus(Order::PAYMENT_STATUS_PAID);
} catch (\Magento\Framework\Exception\CouldNotSaveException $e) {
    $this->logger->critical(
        'Payment charge failed for order %1',
        ['order_id' => $order->getId(), 'exception' => $e]
    );
    $order->setPaymentStatus(Order::PAYMENT_STATUS_FAILED);
    $order->addCommentToStatusHistory(
        __('Payment failed: %1', $e->getMessage())
    );
    // Re-throw only if the caller needs to know
    throw $e;
} catch (\Exception $e) {
    $this->logger->critical('Unexpected error during payment charge', [
        'order_id' => $order->getId(),
        'exception' => $e,
    ]);
    $order->setPaymentStatus(Order::PAYMENT_STATUS_FAILED);
    throw new \Magento\Framework\Exception\LocalizedException(
        __('An unexpected error occurred during payment processing.'),
        $e
    );
}
```

### Try-Catch-Log-Throw Pattern

A useful pattern when you need to add context before propagating:

```php
catch (\Exception $e) {
    $this->logger->error(sprintf(
        'Failed to process customer %d in %s: %s',
        $customerId,
        __METHOD__,
        $e->getMessage()
    ), ['exception' => $e]);

    throw new \Magento\Framework\Exception\LocalizedException(
        __('Could not process customer. Please try again or contact support.'),
        $e
    );
}
```

### Always Log Before Swallowing

If you truly must swallow an exception (e.g., in a non-critical background job where failure should not affect the main flow), log it at minimum:

```php
try {
    $this->nonCriticalCacheWarmup->preloadCategory($categoryId);
} catch (\Exception $e) {
    // Non-critical operation — log but don't fail the page
    $this->logger->warning(
        'Non-critical cache warmup failed for category %1: %2',
        [$categoryId, $e->getMessage()]
    );
}
```

Even in this case, ask yourself whether silently continuing is the right behavior. Perhaps the cache warmup failure should be tracked in a metric or monitored separately.

---

## See Also

- **[19 - Debugging Tools](/home/letiendat1002/workspace/personal/courses/magento2/v2.4.8/19-production-debugging/19-debugging-tools.md)** — Companion chapter covering query logging, Xdebug configuration, profiler setup, and Elasticsearch diagnostics
- **Magento DevDocs: Exception Handling** — [Adobe Commerce Developer Documentation](https://developer.adobe.com/commerce/php/development/components/exception-handling/) for official exception handling guidelines
- **Magento DevDocs: Webapi Exception Handler** — [Webapi ExceptionHandler](https://developer.adobe.com/commerce/php/development/components/webapi/exception-handler/) for API error response formats
- **Alan Storm: Magento 2 Exception Handling** — [In-depth analysis](https://alanstorm.com/category/magento-2/#magento-2-exceptions) of Magento's exception hierarchy and handling patterns
- **Magento 2 Exception Class Reference** — `lib/internal/Magento/Framework/Exception/` directory in the Magento 2.4.8 codebase for all exception class definitions

---

(End of file — 1448 lines)
