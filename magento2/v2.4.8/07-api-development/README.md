# Topic 7: REST APIs, Webhooks & Integration

**Goal:** Build and consume REST APIs — both Magento's Web API framework and external services. Understand webhook patterns for event-driven external system integration.

---

## Topics Covered

- Magento REST API architecture (webapi.xml, service contracts → API binding)
- Creating custom REST endpoints via `webapi.xml`
- Token-based authentication (admin tokens, customer tokens)
- OAuth 2.0 and Integration authentication
- Webhook dispatcher pattern for external system notifications
- Bulk and Async API for large-scale operations
- Rate limiting and error handling for public APIs
- Custom GraphQL resolvers and schema design

---

## Reference Exercises

- **Exercise 7.1:** Create a REST endpoint via `webapi.xml` and test with cURL
- **Exercise 7.2:** Implement token-based authentication for admin access
- **Exercise 7.3:** Build a webhook observer to dispatch on order creation
- **Exercise 7.4:** Implement bulk import using async queue processing
- **Exercise 7.5:** Create a custom GraphQL query and resolver

---

## Completion Criteria

- [ ] Custom REST endpoint accessible at `/rest/V1/training/reviews/:id`
- [ ] Admin token obtained and used to call a protected endpoint
- [ ] Webhook dispatched when order is placed (observer on `sales_order_place_after`)
- [ ] Bulk import processes via async queue without blocking HTTP
- [ ] API returns proper error responses (404, 401, 400) for invalid input
- [ ] Custom GraphQL query returns data via resolver
- [ ] GraphQL mutation creates/updates entities through repository

---

## Topics

---

### Topic 1: Magento REST API Architecture

**How Magento's Web API Works:*

```
HTTP Request → Authentication (token/OAuth) → Route matched via webapi.xml
    → Service Class (Repository) → DB
         ↑ Auto-generated if using Service Contracts
```

**Why the Web API Framework Exists:**

Magento's Web API provides a standardized way to expose business logic to external systems:

- **Security:** Every endpoint requires authentication and ACL authorization
- **Standardization:** Consistent response formats, error handling, authentication
- **Service Contracts:** Uses same interfaces as internal code (Repository pattern)
- **Discoverability:** Auto-generated documentation, integration tools

> **Pro Tip from Production:** Always use Service Contracts (interfaces in `Api/` directory) for your custom REST endpoints. Direct class exposure couples your API to implementation details.

**The Full Request/Response Flow:**

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. HTTP Request arrives                                             │
│    Authorization: Bearer <token>                                    │
│    Content-Type: application/json                                    │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ 2. Magento's WebApiModule intercepts                                 │
│    - Validates route exists in webapi.xml                           │
│    - Extracts ACL resources from route definition                    │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ 3. Authentication                                                   │
│    - Token extraction from header                                    │
│    - Token validation against Magento's token table                  │
│    - User/Integration identification                                 │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ 4. ACL Authorization                                                 │
│    - Checks if authenticated user has required resource            │
│    - Returns 403 if denied                                           │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ 5. Request deserialization                                           │
│    - JSON body → PHP object (based on service class hints)          │
│    - URL parameters mapped to method arguments                      │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ 6. Service class invocation                                          │
│    - Repository/Service method called                               │
│    - Business logic executes                                         │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ 7. Response serialization                                            │
│    - PHP object → JSON response                                     │
│    - Error → standardized error format                              │
└─────────────────────────────────────────────────────────────────────┘
```

**Route Definition in `etc/webapi.xml`:*

```xml
<?xml version="1.0"?>
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Webapi/etc/webapi.xsd">

    <!-- GET /V1/training/reviews — list all -->
    <route method="GET" url="/V1/training/reviews">
        <service class="Training\Review\Api\ReviewRepositoryInterface" method="getList"/>
        <resources>
            <resource ref="Training_Review::review_view"/>
        </resources>
    </route>

    <!-- GET /V1/training/reviews/:id — get one -->
    <route method="GET" url="/V1/training/reviews/:id">
        <service class="Training\Review\Api\ReviewRepositoryInterface" method="getById"/>
        <resources>
            <resource ref="Training_Review::review_view"/>
        </resources>
    </route>

    <!-- POST /V1/training/reviews — create -->
    <route method="POST" url="/V1/training/reviews">
        <service class="Training\Review\Api\ReviewRepositoryInterface" method="save"/>
        <resources>
            <resource ref="Training_Review::review_edit"/>
        </resources>
    </route>

    <!-- DELETE /V1/training/reviews/:id -->
    <route method="DELETE" url="/V1/training/reviews/:id">
        <service class="Training\Review\Api\ReviewRepositoryInterface" method="deleteById"/>
        <resources>
            <resource ref="Training_Review::review_delete"/>
        </resources>
    </route>
</routes>
```

**ACL Resources (required in every route):*

```xml
<resource ref="anonymous"/>                   <!-- Anyone -->
<resource ref="self"/>                        <!-- Current customer -->
<resource ref="Training_Review::review_edit"/> <!-- ACL resource -->
```

---

### Topic 2: REST API Calls

**Getting an Admin Token:*

```bash
curl -X POST http://localhost:8080/rest/V1/integration/admin/token \
  -H "Content-Type: application/json" \
  -d '{"username":"admin", "password":"admin123"}'
```

Returns: `"a1b2c3d4e5f6g7h8i9j0..."`

**Calling a Protected Endpoint:*

```bash
# Get all reviews
curl -X GET http://localhost:8080/rest/V1/training/reviews \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json"

# Get review by ID
curl -X GET http://localhost:8080/rest/V1/training/reviews/1 \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

**Creating a Review via POST:*

```bash
curl -X POST http://localhost:8080/rest/V1/training/reviews \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "review": {
      "product_id": 1,
      "reviewer_name": "Alice",
      "rating": 5,
      "review_text": "Excellent product quality!"
    }
  }'
```

**Updating a Review:*

```bash
curl -X PUT http://localhost:8080/rest/V1/training/reviews/1 \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "review": {
      "rating": 4,
      "review_text": "Updated: Good but delivery was slow"
    }
  }'
```

**Deleting a Review:*

```bash
curl -X DELETE http://localhost:8080/rest/V1/training/reviews/1 \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

**Error Responses — Complete Format:*

| HTTP Code | Meaning | Response Body |
|-----------|---------|---------------|
| 200 | Success | `{"...": "..."}` |
| 400 | Bad Request | `{"message": "Invalid parameter", "parameters": {...}}` |
| 401 | Unauthorized | `{"message": "Consumer is not authorized"}` |
| 403 | Forbidden | `{"message": "ACL resource denied"}` |
| 404 | Not Found | `{"message": "Requested entity not found"}` |
| 500 | Server Error | `{"message": "Internal error"}` |

**Magento Error Response Format — Detailed Structure:**

```json
{
    "message": "Invalid parameter",
    "parameters": {
        "field": "rating",
        "value": "invalid",
        "code": "VALIDATION_ERROR"
    },
    "trace": "..."  // Only in developer mode
}
```

**Creating Custom Error Responses:**

```php
// In your repository or service
public function getById(int $id): ReviewInterface
{
    $review = $this->reviewRepository->getById($id);
    if (!$review) {
        throw new \Magento\Framework\Webapi\Exception(
            __('Review with ID %1 not found', $id),
            0,
            \Magento\Framework\Webapi\Exception::HTTP_NOT_FOUND
        );
    }
    return $review;
}
```

**Error Handling Best Practices:**

| Practice | Why It Matters |
|----------|----------------|
| Always return JSON error format | API consumers expect consistent format |
| Include error codes | Helps consumers handle specific errors |
| Log errors server-side | Don't expose internal details to clients |
| Use appropriate HTTP status codes | 400 for validation, 404 for not found, etc. |
| Don't expose stack traces | Only show in development mode |

> **Security Note:** Never expose internal paths, database queries, or sensitive data in error messages. Always log the details server-side.

---

### Topic 3: Authentication Methods

**Token-Based Auth (simplest):*

```php
// Get admin token
$token = $this->httpClient->post(
    BASE_URL . '/rest/V1/integration/admin/token',
    ['json' => ['username' => 'admin', 'password' => 'admin123']]
)->getBody();

// Use in subsequent requests
$headers = ['Authorization' => "Bearer $token"];
```

**OAuth 2.0 (for third-party integrations):*
More complex — requires pre-registration in Admin → Extensions → Integrations. Returns `access_token` and `refresh_token`.

**Integration Authentication:*
In Admin → System → Integrations, create an integration. Magento generates consumer key/secret for OAuth flow.

**Rate Limiting — Protecting Your API:**

Magento doesn't have built-in rate limiting for REST API, but you should implement it at the application or infrastructure level:

```php
// Example: Implementing rate limiting via plugin
// Plugin/RateLimitPlugin.php
class RateLimitPlugin
{
    private $rateLimiter;

    public function aroundGetById(
        ReviewRepositoryInterface $subject,
        callable $proceed,
        int $id
    ) {
        $key = 'review_api_' . $id;
        if (!$this->rateLimiter->attempt($key)) {
            throw new \Magento\Framework\Webapi\Exception(
                __('Rate limit exceeded'),
                0,
                \Magento\Framework\Webapi\Exception::HTTP_TOO_MANY_REQUESTS
            );
        }
        return $proceed($id);
    }
}
```

**Rate Limiting Best Practices:**

| Strategy | Implementation | Pros/Cons |
|----------|----------------|-----------|
| Token bucket | Redis/token bucket algorithm | Fair, allows bursts |
| Leaky bucket | Fixed rate, queues excess | Predictable, can delay |
| Fixed window | Per-minute/hour limits | Simple, but edge cases |
| Sliding window | Rolling time window | Most accurate, complex |

> **Pro Tip from Production:** Rate limit based on the integration's needs, not a one-size-fits-all. Partner integrations may need higher limits than customer-facing APIs.

**Securing Specific Endpoints with ACL:**

```xml
<!-- Allow anonymous for public reads -->
<route method="GET" url="/V1/training/reviews">
    <service .../>
    <resources><resource ref="anonymous"/></resources>
</route>

<!-- Require authentication for writes -->
<route method="POST" url="/V1/training/reviews">
    <service .../>
    <resources><resource ref="Training_Review::review_edit"/></resources>
</route>
```

**API Security Checklist:**

| Check | Implementation |
|-------|----------------|
| Use HTTPS only | No plaintext API communication |
| Rotate tokens regularly | Tokens should expire |
| Use minimum required ACL | Principle of least privilege |
| Validate all input | Reject malformed requests early |
| Log API access | Audit trail for compliance |
| Implement rate limiting | Prevent abuse |

---

### Topic 4: Webhooks for External Systems

**Pattern:** When something happens in Magento → dispatch notification → external system reacts.

**Webhook Observer — `Observer/OrderCreatedWebhook.php`:*

```php
<?php
// Observer/OrderCreatedWebhook.php
namespace Training\Webhook\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Model\Order;

class OrderCreatedWebhook implements ObserverInterface
{
    protected $logger;
    protected $webhookService;

    public function __construct(
        \Psr\Log\LoggerInterface $logger,
        \Training\Webhook\Service\WebhookService $webhookService
    ) {
        $this->logger = $logger;
        $this->webhookService = $webhookService;
    }

    public function execute(Observer $observer): void
    {
        /** @var Order $order */
        $order = $observer->getData('order');

        // sales_order_place_after only fires on new order creation (not updates)

        $payload = [
            'event' => 'order.created',
            'timestamp' => time(),
            'data' => [
                'order_id' => $order->getId(),
                'increment_id' => $order->getIncrementId(),
                'customer_email' => $order->getCustomerEmail(),
                'total' => $order->getGrandTotal(),
                'currency' => $order->getOrderCurrencyCode(),
            ]
        ];

        try {
            $this->webhookService->dispatch('order.created', $payload);
            $this->logger->info('Webhook dispatched for order: ' . $order->getId());
        } catch (\Exception $e) {
            $this->logger->error('Webhook dispatch failed: ' . $e->getMessage());
        }
    }
}
```

**Webhook Service — `Service/WebhookService.php`:*

```php
<?php
// Service/WebhookService.php
namespace Training\Webhook\Service;

use GuzzleHttp\Client;
use GuzzleHttp\Exception\GuzzleException;
use Psr\Log\LoggerInterface;
use Magento\Framework\App\Config\ScopeConfigInterface;

class WebhookService
{
    protected $logger;
    protected $scopeConfig;
    protected $client;

    public function __construct(
        LoggerInterface $logger,
        ScopeConfigInterface $scopeConfig,
        Client $client
    ) {
        $this->logger = $logger;
        $this->scopeConfig = $scopeConfig;
        $this->client = $client;
    }

    public function dispatch(string $event, array $payload): bool
    {
        $webhookUrl = $this->scopeConfig->getValue("webhooks/events/{$event}/url");
        if (!$webhookUrl) return false;

        try {
            $response = $this->client->post($webhookUrl, [
                'headers' => [
                    'Content-Type' => 'application/json',
                    'X-Webhook-Signature' => $this->sign($payload),
                    'X-Webhook-Event' => $event,
                ],
                'json' => $payload
            ]);

            return $response->getStatusCode() === 200;
        } catch (GuzzleException $e) {
            $this->logger->error("Webhook failed for {$event}: " . $e->getMessage());
            return false;
        }
    }

    private function sign(array $payload): string
    {
        $secret = $this->scopeConfig->getValue('webhooks/secret');
        return hash_hmac('sha256', json_encode($payload), $secret);
    }
}
```

**Register Event — `etc/events.xml`:*

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="sales_order_place_after">
        <observer name="training_webhook_order"
                  instance="Training\Webhook\Observer\OrderCreatedWebhook"
                  sortOrder="10"/>
    </event>
</config>
```

**Important:** In production, dispatch webhooks asynchronously via queue — never synchronously in the observer, or a slow external call will block the HTTP response.

**Webhook Signature Verification — Security Best Practice:**

Always verify webhook signatures to ensure requests are from your system:

```php
// In WebhookService.php
private function sign(array $payload): string
{
    $secret = $this->scopeConfig->getValue('webhooks/secret');
    return hash_hmac('sha256', json_encode($payload), $secret);
}

// Verify incoming webhook signature
public function verifySignature(string $payload, string $signature): bool
{
    $expected = hash_hmac('sha256', $payload, $this->getSecret());
    return hash_equals($expected, $signature);
}
```

**Webhook Best Practices:**

| Practice | Why It Matters |
|----------|----------------|
| Always verify signatures | Prevents spoofed webhooks |
| Use HTTPS for webhook URLs | Encrypts sensitive payload |
| Implement idempotency | Same webhook can be delivered multiple times |
| Use queue for dispatch | Don't block HTTP response |
| Log all webhook events | Audit trail for debugging |
| Implement retry with backoff | External systems may be temporarily down |

> **⚠️ Critical:** Never process webhooks synchronously in the observer. If the external system is slow or down, your Magento request will timeout or hang.

---

### Topic 5: Bulk & Async API

**Use case:** Large imports/exports that would timeout if run synchronously.

**Queue Configuration — `etc/queue.xml`:*

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:MessageQueue/etc/queue.xsd">
    <queue name="training_review_bulk_export"
           connection="db"
           exchange="magento"
           topic="training.review.bulk_export">
        <consumer name="Training_Review_Consumer_BulkExport"
                  queue="training_review_bulk_export"
                  maxMessages="100"
                  instance="Training\Review\Model\Async\BulkExportConsumer"/>
    </queue>
</config>
```

**Scheduling Bulk Operations:*

```php
<?php
use Magento\AsynchronousOperations\Api\BulkScheduleInterface;
use Magento\Framework\Async\BulkOperationInterface;

class BulkExportController
{
    protected $bulkSchedule;

    public function __construct(BulkScheduleInterface $bulkSchedule)
    {
        $this->bulkSchedule = $bulkSchedule;
    }

    public function execute(): array
    {
        $reviewIds = $this->getReviewIdsFromRequest();

        $bulkUuid = $this->bulkSchedule->scheduleBulk(
            'training_export_reviews',
            $reviewIds,
            [],
            'Bulk review export',
            0
        );

        return ['uuid' => $bulkUuid, 'status' => 'scheduled'];
    }
}
```

**Async Consumer:*

```php
<?php
// Model/Async/BulkExportConsumer.php
namespace Training\Review\Model\Async;

use Magento\Framework\MessageQueue\ConsumerInterface;
use Psr\Log\LoggerInterface;

class BulkExportConsumer implements ConsumerInterface
{
    protected $logger;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function processMessage(array $message): void
    {
        $reviewIds = $message['review_ids'] ?? [];
        $this->logger->info('Processing bulk export for ' . count($reviewIds) . ' reviews');

        foreach ($reviewIds as $reviewId) {
            // Generate export row...
        }

        $this->logger->info('Bulk export complete');
    }
}
```

**Running the Consumer:*

```bash
# Continuous worker
bin/magento queue:consumers:start Training_Review_Consumer_BulkExport

# One-shot (cron-based)
bin/magento queue:consumers:run Training_Review_Consumer_BulkExport --max-messages=100
```

**Pagination Patterns for REST API:**

Magento's REST API uses cursor-based pagination for large datasets:

```bash
# First page
GET /V1/training/reviews?searchCriteria[pageSize]=10&searchCriteria[currentPage]=1

# Response includes total count and page info
{
    "items": [...],
    "search_criteria": {...},
    "total_count": 150
}
```

**Implementing Pagination in Your Repository:**

```php
// In your repository's getList method
public function getList(\Magento\Framework\Api\SearchCriteriaInterface $searchCriteria)
{
    $collection = $this->collectionFactory->create();

    // Apply filters from search criteria
    foreach ($searchCriteria->getFilterGroups() as $filterGroup) {
        foreach ($filterGroup->getFilters() as $filter) {
            $collection->addFieldToFilter(
                $filter->getField(),
                [$filter->getConditionType() => $filter->getValue()]
            );
        }
    }

    // Apply sorting
    foreach ($searchCriteria->getSortOrders() as $sortOrder) {
        $collection->setOrder(
            $sortOrder->getField(),
            $sortOrder->getDirection()
        );
    }

    // Apply pagination
    $collection->setCurPage($searchCriteria->getCurrentPage());
    $collection->setPageSize($searchCriteria->getPageSize());

    return $collection;
}
```

**Pagination Best Practices:**

| Practice | Why It Matters |
|----------|----------------|
| Default page size | Prevents fetching too much data |
| Maximum page size | Prevents memory exhaustion |
| Always return total count | Clients need to show pagination UI |
| Use cursor-based for real-time data | Offset-based can miss/missort new items |

---

### Topic 6: OpenAPI/Swagger Integration

**Magento's Built-in Swagger UI:**

Magento includes Swagger UI for API documentation at:
```
/swagger
/rest/V1/schema
```

**Why OpenAPI/Swagger Matters:**

- **Self-documenting API:** Auto-generates documentation from code
- **Interactive testing:** Try API calls directly in browser
- **Client SDK generation:** Generate API clients for any language
- **Contract testing:** Verify API matches specification

**Registering Custom Endpoints in Swagger:**

Magento automatically discovers endpoints from `webapi.xml`. Your routes will appear in Swagger automatically if they follow Magento conventions.

**Custom API Documentation:**

```xml
<!-- In etc/webapi.xml -->
<route method="GET" url="/V1/training/reviews/:id">
    <service class="Training\Review\Api\ReviewRepositoryInterface" method="getById"/>
    <resources>
        <resource ref="Training_Review::review_view"/>
    </resources>
    <data>
        <parameter name="id" required="true" type="integer">Review ID</parameter>
    </data>
</route>
```

**OpenAPI Best Practices:**

| Practice | Why It Matters |
|----------|----------------|
| Use descriptive endpoint names | `/V1/products/{id}` not `/V1/p/{id}` |
| Document response formats | Help consumers understand your API |
| Version your API | `/V1/`, `/V2/` for breaking changes |
| Use standard HTTP methods | GET=read, POST=create, PUT=update, DELETE=delete |
| Return consistent error formats | All errors follow same structure |

---

### Topic 7: GraphQL in Magento — Beyond REST

**Why GraphQL Matters for Headless:** REST requires multiple endpoints for complex data needs. GraphQL lets the frontend request exactly the fields it needs in a single query. Modern Magento headless storefronts (React/Vue/Next.js) are built on GraphQL.

**Magento's GraphQL Architecture:**

```
HTTP POST /graphql
  → GraphQL Schema (auto-generated from webapi.xml + custom .graphqls files)
  → Resolver chain (etc/graphql/di.xml wiring)
  → Service classes (repositories)
  → Database
```

**Key Difference from REST:** REST uses `webapi.xml` to expose service contracts as HTTP endpoints. GraphQL uses `etc/graphql/di.xml` to wire `ResolverInterface` implementations to schema fields.

**Custom GraphQL Query — Step by Step:**

**1. Define schema in etc/graphql/schema.graphqls:**
```graphql
type Query {
    trainingReview(id: Int!): TrainingReviewOutput @doc(description: "Get a single review by ID")
}

type TrainingReviewOutput {
    review_id: Int
    title: String
    detail: String
    rating: Float
    nickname: String
}

extend type Product {
    reviews: [TrainingReviewOutput] @resolver(class: "Training\\Review\\Model\\Resolver\\ProductReviews")
}
```

**2. Create the resolver class:**
```php
<?php
namespace Training\Review\Model\Resolver;

use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Exception\GraphQlInputException;
use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;

class ProductReviews implements ResolverInterface
{
    public function __construct(
        private ReviewRepositoryInterface $reviewRepository
    ) {}

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ): array {
        if (!isset($args['id'])) {
            throw new GraphQlInputException(__('Review ID is required'));
        }

        $review = $this->reviewRepository->getById($args['id']);
        return [
            'review_id' => $review->getId(),
            'title' => $review->getTitle(),
            'detail' => $review->getDetail(),
            'rating' => $review->getRating(),
            'nickname' => $review->getNickname(),
        ];
    }
}
```

**3. Wire resolver in etc/graphql/di.xml:**
```xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:etc/graphql.xsd">
    <type name="Training\Review\Model\Resolver\ProductReviews">
        <arguments>
            <argument name="reviewRepository" xsi:type="object">Training\Review\Api\ReviewRepositoryInterface</argument>
        </arguments>
    </type>
</config>
```

**Mutation Pattern:**
```graphql
mutation CreateReview($input: TrainingReviewInput!) {
    createTrainingReview(input: $input) {
        review_id
        nickname
        rating
    }
}

input TrainingReviewInput {
    product_id: Int!
    title: String!
    detail: String!
    rating: Float!
    nickname: String!
}
```

```php
class CreateReview implements ResolverInterface
{
    public function resolve(Field $field, $context, ResolveInfo $info, ...): array
    {
        $input = $args['input'];
        // Validate, save to repository
        return ['review_id' => $savedId, 'nickname' => $input['nickname'], 'rating' => $input['rating']];
    }
}
```

**GraphQL vs REST — When to Choose Which:**

| Scenario | Recommendation |
|----------|----------------|
| Mobile app, single object per request | REST — simpler |
| PWA/React storefront, complex nested data | GraphQL — eliminates over-fetching |
| Admin panel operations | REST — admin tokens, simpler auth |
| Headless B2B with custom entities | GraphQL — flexible schema |
| Bulk operations | REST Async API — queue processing |

**Pro Tips:**
- GraphQL Playground (GraphiQL) is available at `/graphql` in developer mode — use it to test queries before building frontend
- Always implement `CacheIdentityInterface` on resolvers if the data doesn't change frequently
- Field authorization: check `$context->getUserId()` and `$context->getUserType()` in resolvers
- Query depth limits prevent circular queries from crashing the server

**Common GraphQL Mistakes:**
- ❌ Not declaring `extend type` for existing Magento types → duplicate type definition error
- ❌ Forgetting `di.xml` resolver wiring → resolver never called, returns null
- ❌ Not handling null returns → GraphQL returns errors for null scalar fields without proper nullable declaration
- ❌ Expensive queries without caching → GraphQL calls the database on every request, no built-in caching like REST FPC

---

## Reading List

- [Magento Web API](https://developer.adobe.com/commerce/php/development/components/webapi/) — REST and GraphQL
- [Create Custom REST API](https://developer.adobe.com/commerce/webapi/rest/tutorials/orders/) — webapi.xml configuration
- [Authentication](https://developer.adobe.com/commerce/php/development/components/webapi/authentication/) — Token, OAuth, Integration
- [Message Queue](https://developer.adobe.com/commerce/php/development/components/message-queue/) — Async operations

---

## Edge Cases & Troubleshooting

| Issue | Symptom | Solution |
|-------|---------|----------|
| API returns 401 | Token expired/invalid | Re-authenticate; tokens expire |
| Webhook not firing | External system stale | Verify event name in `events.xml` matches exactly |
| Bulk API timeout | Import stalls | Split into smaller batches or use async queue |
| CORS errors | Browser blocked | Use server-side (PHP) API calls, not AJAX |
| Missing ACL resource | 403 on endpoint | `<resource ref="..."/>` must exist in `etc/acl.xml` |

---

## Common Mistakes to Avoid

1. ❌ **Exposing sensitive endpoints without ACL** → Use `anonymous` only for public reads
   - **Prevention:** Always define ACL resources, test with limited roles

2. ❌ **Not validating API input** → Check required fields in webapi.xml and repository
   - **Prevention:** Validate in service layer, not just webapi.xml

3. ❌ **Synchronous webhooks** → Use queue for external HTTP calls
   - **Prevention:** Dispatch to queue, let consumer handle HTTP

4. ❌ **Hardcoding API secrets** → Use `scopeConfig`, never commit to git
   - **Prevention:** Environment variables or Magento config

5. ❌ **Forgetting error handling** → API errors should return proper HTTP codes
   - **Prevention:** Use `\Magento\Framework\Webapi\Exception`

6. ❌ **Returning internal error details** → Exposes system info to attackers
   - **Prevention:** Log details, return generic message

7. ❌ **Not handling token expiration** → API calls fail silently
   - **Prevention:** Implement token refresh logic

8. ❌ **Using GET for mutations** → Violates REST, caching issues
   - **Prevention:** GET=read only, POST/PUT/DELETE for changes

---

## Pro Tips from Production

1. **Always version your API** — `/V1/`, `/V2/` for breaking changes; consumers depend on stability

2. **Return consistent response formats** — Even errors should follow same structure

3. **Implement idempotency keys for POST operations** — Same request shouldn't create duplicate orders

4. **Use batch operations for bulk changes** — One call with 100 items vs 100 separate calls

5. **Log API access for debugging** — Who called what, when, with what result

6. **Test with various auth contexts** — Anonymous, customer, admin all have different permissions

7. **Monitor API response times** — Slow APIs indicate missing indexes or N+1 queries

8. **Document breaking changes** — Consumers need to know when you change response format

---

## Cross-References

- **Topic 05 (Customization):** Plugins can intercept API controller methods
- **Topic 06 (Admin UI):** Admin APIs use the same ACL system
- **Topic 08 (Data Ops):** Bulk operations often exposed via API

---

*Magento 2 Backend Developer Course — Topic 07*
