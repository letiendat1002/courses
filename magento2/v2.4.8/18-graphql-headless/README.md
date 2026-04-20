# Topic 18: GraphQL for Headless Commerce

**Goal:** Build custom GraphQL resolvers for headless storefronts — implementing queries, mutations, and custom schema extensions that integrate cleanly with Magento's GraphQL layer.

---

## Topics Covered

1. GraphQL in Magento — Architecture Overview
2. Custom GraphQL Schema — schema.graphqls
3. Resolver Architecture — Resolver Class Patterns
4. Mutations — Input, Payload, and Response
5. Advanced Patterns — Fragments, Aliases, Query Batching
6. GraphQL Caching — CacheIdentity and Cache Tags
7. Security — Query Complexity, Depth Limiting, Field Authorization
8. Real-World Patterns — Search, Aggregation, Custom Entities

---

## Reference Exercises

- **Exercise 1 — Custom Product Query**: Define a `schema.graphqls` with a custom query that returns enriched product data from a third-party PIM. Wire the resolver via `etc/graphql/di.xml`, implement `ResolverInterface`, and verify via GraphQL Playground.
- **Exercise 2 — Custom Mutation with Payload**: Implement a mutation (e.g., submit a product question) with a structured `input` type and `payload` type returning success/error. Add field-level authorization to restrict logged-in customers only.
- **Exercise 3 — Cache-Aware Resolver**: Extend the Exercise 1 resolver to implement `CacheIdentityInterface`, generate cache tags from entity IDs, and write an observer that invalidates the cache on entity save.
- **Exercise 4 — Extending Core Type**: Use `extend type Product` in `schema.graphqls` to add a custom field sourced from an extension attribute. Verify the extended field appears in introspection and doesn't break any existing product queries.

---

## Completion Criteria

- [ ] Custom GraphQL query accessible via GraphQL Playground returning custom data
- [ ] Custom mutation accepts input and returns structured payload
- [ ] Resolver wired via `etc/graphql/di.xml`
- [ ] Cache tags generated and cache invalidated on entity changes
- [ ] Field-level authorization restricts access appropriately
- [ ] Query complexity within acceptable limits
- [ ] Schema extends existing Magento type without breaking core queries

---

## Topics

---

### Topic 1: GraphQL in Magento — Architecture Overview

**Why GraphQL Over REST for Headless?**

REST's multiple-endpoint model creates inefficiency for frontend clients — a product detail page often requires 5–10 separate API calls (product, price, inventory, media, reviews). GraphQL's single `/graphql` endpoint with a typed schema lets the client request exactly the fields it needs in one round-trip. For headless storefronts (React/Vue/Flutter mobile apps, SPAs), this dramatically reduces latency and over-fetching.

Magento's GraphQL layer was introduced in 2.4.0 and has become the primary API surface for Magento's own PWA Studio ("Venia") and any third-party headless frontends.

**The Single Endpoint Pattern:**

```
REST:  GET  /V1/products/:id
       GET  /V1/products/:id/media
       GET  /V1/products/:id/inventory
       GET  /V1/products/:id/reviews

GraphQL: POST /graphql
         { products(filter: { sku: { eq: "SKU123" } }) { items { name price media_gallery { url } ... } } }
```

Every GraphQL request — whether a query (read) or mutation (write) — goes to the single `/graphql` endpoint. The operation type (`query` or `mutation`) and the operation name tell Magento what to execute.

**Magento's GraphQL Plumbing:**

Unlike REST which uses `webapi.xml` to declare routes, GraphQL uses two separate files:

- `etc/graphql/schema.graphqls` — the **Schema Provider**: defines the SDL (Schema Definition Language) types, queries, and mutations. This is the public contract.
- `etc/graphql/di.xml` — the **Resolver DI config**: wires resolver classes to schema fields. This is the implementation.

The SDL schema is introspectable — hitting `/graphql` with an introspection query returns the entire type system. Frontend clients use this to generate typed client code.

**Authentication in GraphQL:**

Magento supports three authentication contexts for GraphQL, set via the `Authorization` header:

| Context | Header Value | Access Level |
|---------|-------------|--------------|
| Guest | None | Public fields only (defined with `@doc` or default) |
| Customer | `Bearer <customer_token>` | Customer-specific fields + guest access |
| Admin | `Bearer <admin_token>` | Full read/write including admin-only fields |

**Obtaining tokens:**

```bash
# Customer token — returns a 32-char token string
curl -X POST http://localhost/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"mutation { customerToken(email: \"alice@example.com\", password: \"password123\") { token } }"}'

# Admin token — same endpoint as REST: /rest/V1/integration/admin/token
curl -X POST http://localhost/rest/V1/integration/admin/token \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'
```

> **Note:** In GraphQL, if a field requires authentication and the token is missing or invalid, the response typically returns `null` for that field (rather than an HTTP 401). The resolver must check `$info->context->getUserId()` and handle unauthorized access gracefully.

**The GraphQL Playground:**

Starting in 2.4.6, the old GraphiQL IDE at `/graphiql` was removed. The built-in playground is available at `/graphql` when Magento is in developer mode. It provides:
- Syntax-highlighted query editor
- Schema introspection browser
- Response viewer with timing information
- Documentation explorer

```bash
# Access playground in developer mode
open http://localhost/graphql
```

> **Pro Tip:** Always develop and test your custom GraphQL queries in the playground before integrating with frontend code. The playground's "Docs" explorer (top-right) lets you browse the entire schema including your custom types.

**Auto-Generated Schema:**

Magento's core modules (`CatalogGraphQl`, `CheckoutGraphQl`, etc.) register their schemas via `etc/graphql/schema.graphqls` files in each module. These are merged into a single composite schema at runtime. When you install a new module, its schema types appear in the playground automatically.

You can verify the schema is loaded correctly:

```graphql
query {
  __schema {
    types {
      name
      kind
    }
  }
}
```

> **Key Architecture Insight:** Magento's GraphQL layer is read-optimized by design. Writes (mutations) follow a more explicit pattern with Input/Payload types. Direct entity updates through GraphQL resolvers bypass Magento's event system unless the resolver explicitly dispatches events — keep this in mind when integrating with external systems.

---

### Topic 2: Custom GraphQL Schema — schema.graphqls

**Schema Definition Language (SDL) Basics:**

Magento's GraphQL schema uses the standard GraphQL SDL format. A schema file lives at `Vendor/Module/etc/graphql/schema.graphqls` and is automatically merged into the global schema when the module is loaded.

**Scalar and Enum Types:**

```graphql
# Built-in scalars: String, Int, Float, Boolean, ID

# Custom enum for review rating
enum ReviewRating {
  ONE_STAR
  TWO_STARS
  THREE_STARS
  FOUR_STARS
  FIVE_STARS
}

# Custom scalar for ISO8601 dates
scalar DateTime @doc(description: "ISO8601 timestamp")
```

**Object Types and Fields:**

```graphql
type ProductQuestion @doc(description: "A product question submitted by a customer") {
  question_id: Int! @doc(description: "Unique question ID")
  product_sku: String! @doc(description: "SKU of the product being asked about")
  customer_name: String! @doc(description: "Asker's name")
  question_text: String! @doc(description: "The question content")
  created_at: DateTime @doc(description: "When the question was submitted")
  answer_text: String @doc(description: "Reply from the support team")
}
```

> **Note on nullability:** `String!` means non-null (required field). `String` means nullable. `[Product!]!` means a non-null list of non-null Products — the list itself cannot be null but items inside it can be null.

**Input Types (for mutation arguments):**

```graphql
input ProductQuestionInput @doc(description: "Input for submitting a product question") {
  product_sku: String! @doc(description: "SKU of the product to ask about")
  customer_name: String! @doc(description: "Your name")
  question_text: String! @doc(description: "Your question, max 1000 characters")
  customer_email: String! @doc(description: "Your email for notifications")
}
```

**Extending Existing Magento Types:**

You rarely need to create entirely new types for custom functionality. More often you extend existing Magento types to add fields.

```graphql
# In Vendor/Module/etc/graphql/schema.graphqls
extend type Product @doc(description: "Extended product fields for PIM data") {
  pim_last_synced_at: DateTime @doc(description: "Last sync timestamp from external PIM")
  pim_origin_id: String @doc(description: "Product ID in the external PIM system")
  energy_rating: String @doc(description: "EU energy classification label")
}

extend enum ProductAttributeType {
  PIM_ORIGIN @doc(description: "PIM-sourced attribute type")
}
```

> **Why `extend type`?** Magento's product type is defined in `CatalogGraphQl`. Rather than modifying that core file (which would break on upgrade), your module contributes an extension. The schema merger combines all extensions at runtime.

**Query Root Type:**

```graphql
type Query {
  training_product_questions(
    filter: ProductQuestionFilterInput
    pageSize: Int = 20
    currentPage: Int = 1
  ): ProductQuestionOutput @doc(description: "Retrieve product questions")
  training_product_question(id: Int!): ProductQuestion
    @doc(description: "Retrieve a single product question by ID")
    @resolver(class: "Training\\ProductQuestion\\Model\\Resolver\\Question")
}
```

> **The `@resolver` Directive:** You can inline a resolver reference directly in the schema using `@resolver(class: "Vendor\\Module\\Model\\Resolver\\ClassName")`. This is a shortcut that avoids needing an explicit `di.xml` entry for simple cases.

**Mutation Root Type:**

```graphql
type Mutation {
  submitProductQuestion(
    input: ProductQuestionInput!
  ): ProductQuestionPayload! @doc(description: "Submit a new product question")
}
```

**Output Types:**

```graphql
type ProductQuestionOutput @doc(description: "Paginated list of product questions") {
  total_count: Int! @doc(description: "Total number of questions matching the filter")
  items: [ProductQuestion!]! @doc(description: "The questions for this page")
  page_info: SearchResultPageInfo! @doc(description: "Pagination metadata")
}

type ProductQuestionPayload @doc(description: "Response from submitting a product question") {
  success: Boolean! @doc(description: "Whether the submission succeeded")
  message: String @doc(description: "Human-readable status message")
  question_id: Int @doc(description: "ID of created question if successful")
}
```

> **Common Mistake — Forgetting the `!` on required fields:** A mutation input field without `!` is nullable. If your resolver expects the field to always be present, mark it required (`String!`) and validate it in the resolver. Otherwise a null value will silently pass through.

---

### Topic 3: Resolver Architecture — Resolver Class Patterns

**The ResolverInterface:**

Every GraphQL field that needs custom logic (rather than direct property access) requires a resolver class implementing `Magento\Framework\GraphQl\Query\ResolverInterface`.

```php
<?php
declare(strict_types=1);

namespace Training\ProductQuestion\Model\Resolver;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Framework\GraphQl\Config\Element\Field;

/**
 * Resolver for training_product_question query — single question by ID
 */
class Question implements ResolverInterface
{
    private QuestionRepositoryInterface $questionRepository;

    public function __construct(
        QuestionRepositoryInterface $questionRepository
    ) {
        $this->questionRepository = $questionRepository;
    }

    /**
     * @param Field $field
     * @param mixed $context  — Magento context (contains userId, userType, websiteId, etc.)
     * @param array $resolvedArgs — already-resolved parent value (for nested fields)
     * @param ResolveInfo $resolveInfo — GraphQL operation metadata
     * @return array|object|null
     */
    public function resolve(
        Field $field,
        $context,
        ResolveInfo $resolveInfo,
        array $resolvedArgs = []
    ) {
        // Only admins can view questions by ID
        if ($context->getUserType() !== \Magento\Authorization\Model\UserContextInterface::USER_TYPE_ADMIN) {
            return null;
        }

        $id = (int) $resolvedArgs['id'];
        $question = $this->questionRepository->getById($id);

        if (!$question) {
            return null;
        }

        // Return array — Magento's GraphQL layer converts to the GraphQL type
        return [
            'question_id' => $question->getId(),
            'product_sku' => $question->getProductSku(),
            'customer_name' => $question->getCustomerName(),
            'question_text' => $question->getQuestionText(),
            'created_at' => $question->getCreatedAt(),
            'answer_text' => $question->getAnswerText(),
        ];
    }
}
```

**Wiring via `etc/graphql/di.xml`:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:GraphQl/etc/graphql.xsd">

    <!-- Query field resolvers -->
    <type name="Magento\Framework\GraphQl\Query\ResolverBatch">
        <arguments>
            <argument name="resolverMap" xsi:type="array">
                <item name="training_product_questions"
                      xsi:type="string">Training\ProductQuestion\Model\Resolver\QuestionList</item>
                <item name="training_product_question"
                      xsi:type="string">Training\ProductQuestion\Model\Resolver\Question</item>
            </argument>
        </arguments>
    </type>
</config>
```

> **Alternative — Inline Resolver in Schema:** You can also use `@resolver(class: "Vendor\\Module\\Model\\Resolver\\Class")` directly in `schema.graphqls` instead of wiring in `di.xml`. Both approaches work; `di.xml` is preferred when you need constructor injection of multiple dependencies or when the same resolver is used by multiple fields.

**ResolverBatchInterface for Batch Resolution:**

For fields that resolve multiple items efficiently (e.g., a list of products), implement `Magento\Framework\GraphQl\Query\ResolverBatchInterface`:

```php
<?php
declare(strict_types=1);

namespace Training\ProductQuestion\Model\Resolver;

use Magento\Framework\GraphQl\Query\ResolverBatchInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;

class QuestionList implements ResolverBatchInterface
{
    private QuestionRepositoryInterface $questionRepository;

    public function __construct(QuestionRepositoryInterface $questionRepository)
    {
        $this->questionRepository = $questionRepository;
    }

    /**
     * @param array $resolvedArgsPerField — map of fieldName => resolvedArgs array
     * @param ResolveInfo $resolveInfo
     * @return array — map of fieldName => result array
     */
    public function resolveBatch(array $resolvedArgsPerField, ResolveInfo $resolveInfo): array
    {
        $results = [];

        foreach ($resolvedArgsPerField as $fieldName => $resolvedArgs) {
            $filter = $this->buildFilterFromArgs($resolvedArgs);
            $questions = $this->questionRepository->getList($filter);

            $results[$fieldName] = [
                'total_count' => $questions->getTotalCount(),
                'items' => array_map([$this, 'transformQuestion'], $questions->getItems()),
                'page_info' => [
                    'current_page' => $resolvedArgs['currentPage'] ?? 1,
                    'page_size' => $resolvedArgs['pageSize'] ?? 20,
                    'total_pages' => ceil($questions->getTotalCount() / ($resolvedArgs['pageSize'] ?? 20)),
                ],
            ];
        }

        return $results;
    }

    private function buildFilterFromArgs(array $args): array
    {
        $filter = [];
        if (!empty($args['filter']['product_sku'])) {
            $filter['product_sku'] = ['eq' => $args['filter']['product_sku']['eq']];
        }
        return $filter;
    }

    private function transformQuestion(QuestionInterface $question): array
    {
        return [
            'question_id' => $question->getId(),
            'product_sku' => $question->getProductSku(),
            'customer_name' => $question->getCustomerName(),
            'question_text' => $question->getQuestionText(),
            'created_at' => $question->getCreatedAt(),
            'answer_text' => $question->getAnswerText(),
        ];
    }
}
```

**Handling Null and Errors:**

Resolvers should return `null` for "not found" and let the GraphQL response omit the field gracefully. For authorization failures, return `null` (do not throw — throwing would propagate as an error response for the whole query):

```php
// Check authorization first
if ($context->getUserId() === null && $this->requiresAuth) {
    return null; // Field will be null in response — no error
}

// Check for not found
$entity = $this->repository->getById($id);
if (!$entity) {
    return null; // Returns null for this specific field, not the whole query
}
```

> **Important:** Unlike REST controllers, GraphQL resolvers do not throw HTTP exceptions. A resolver throwing an exception will result in the entire query failing. Use `null` returns for missing data or access-denied situations where the rest of the query should still succeed.

**The ResolveInfo Object:**

`ResolveInfo $resolveInfo` is the most powerful parameter — it contains the full query AST and field metadata:

```php
// Get the current field being resolved
$currentField = $resolveInfo->fieldName; // e.g., "training_product_questions"

// Get the parent value (for nested fields — when resolver is called on a nested field)
$parentValue = $resolveInfo->parentValue; // e.g., the Product object when resolving a product question

// Get the selection set (fields requested on this type) — useful for partial loading optimization
$selectionSet = array_map(fn($field) => $field->name, $resolveInfo->fieldNodes[0]->selectionSet->selections);
```

---

### Topic 4: Mutations — Input, Payload, and Response

**Mutation Flow:**

A GraphQL mutation follows a three-phase pattern:

```
1. Client sends mutation with input type → Input deserialized by Magento's GraphQL layer
2. Mutation resolver receives input → validates, calls repository/service
3. Resolver returns payload type → GraphQL layer serializes to JSON
```

**Input Type (what the client sends):**

```graphql
input ProductQuestionInput {
  product_sku: String!
  customer_name: String!
  question_text: String!
  customer_email: String!
}
```

**Payload Type (what the resolver returns):**

```graphql
type ProductQuestionPayload {
  success: Boolean!
  message: String
  question_id: Int
}
```

**Mutation Resolver Implementation:**

```php
<?php
declare(strict_types=1);

namespace Training\ProductQuestion\Model\Resolver;

use Magento\Framework\GraphQl\Mutation\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Framework\GraphQl\Config\Element\Field;
use Training\ProductQuestion\Api\QuestionRepositoryInterface;

class SubmitQuestion implements ResolverInterface
{
    private QuestionRepositoryInterface $questionRepository;

    public function __construct(QuestionRepositoryInterface $questionRepository)
    {
        $this->questionRepository = $questionRepository;
    }

    public function resolve(Field $field, $context, ResolveInfo $resolveInfo, array $resolvedArgs = []): array
    {
        // Step 1: Authorize — only customers or guests can submit
        // Admins should not submit questions (business rule)
        $userType = $context->getUserType();
        $customerId = $context->getUserId();

        // Step 2: Validate input
        $input = $resolvedArgs['input'];
        $errors = $this->validateInput($input);
        if (!empty($errors)) {
            return [
                'success' => false,
                'message' => implode(', ', $errors),
                'question_id' => null,
            ];
        }

        // Step 3: Create and save the question
        try {
            $question = $this->questionRepository->create();
            $question->setProductSku($input['product_sku']);
            $question->setCustomerName($input['customer_name']);
            $question->setQuestionText($input['question_text']);
            $question->setCustomerEmail($input['customer_email']);
            $question->setCustomerId($customerId ?? 0);

            $savedId = $this->questionRepository->save($question);

            return [
                'success' => true,
                'message' => __('Your question has been submitted successfully.')->render(),
                'question_id' => (int) $savedId,
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'message' => __('Unable to submit your question. Please try again.')->render(),
                'question_id' => null,
            ];
        }
    }

    private function validateInput(array $input): array
    {
        $errors = [];

        if (empty($input['product_sku'])) {
            $errors[] = 'Product SKU is required';
        }
        if (empty($input['question_text'])) {
            $errors[] = 'Question text is required';
        }
        if (strlen($input['question_text'] ?? '') > 1000) {
            $errors[] = 'Question text cannot exceed 1000 characters';
        }
        if (!filter_var($input['customer_email'] ?? '', FILTER_VALIDATE_EMAIL)) {
            $errors[] = 'Valid customer email is required';
        }

        return $errors;
    }
}
```

**Wiring the Mutation in di.xml:**

```xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:GraphQl/etc/graphql.xsd">
    <type name="Magento\Framework\GraphQl\Mutation\ResolverAttacher">
        <arguments>
            <argument name="resolverMap" xsi:type="array">
                <item name="submitProductQuestion"
                      xsi:type="string">Training\ProductQuestion\Model\Resolver\SubmitQuestion</item>
            </argument>
        </arguments>
    </type>
</config>
```

**Chaining Mutations:**

GraphQL allows a single request to include multiple mutations. These execute sequentially (not in parallel) and each can return data used by subsequent operations:

```graphql
mutation {
  # First mutation
  submitProductQuestion(input: { product_sku: "SKU123", customer_name: "Alice", question_text: "Is this eco-friendly?", customer_email: "alice@example.com" }) {
    success
    question_id
  }
}
```

> **Note on mutation order:** Within a single request, mutations execute in the order they appear in the query string. This matters when later mutations depend on the results of earlier ones (e.g., creating an order then applying a payment to it). This is a key difference from REST where each endpoint is independent.

**Using `@doc` for Schema Documentation:**

Every custom field and type should have a `@doc` directive. This documentation appears in the GraphQL playground's schema explorer:

```graphql
type ProductQuestionPayload @doc(description: "Response returned after submitting a product question. Check the success field to determine if submission succeeded.") {
  success: Boolean! @doc(description: "True if the question was saved successfully")
  message: String @doc(description: "Status message explaining what happened")
  question_id: Int @doc(description: "ID of the newly created question. Null if submission failed.")
}
```

> **Common Mistake — Inconsistent nullability in payload:** If `success` is `Boolean!` (non-null), always return a boolean value for `success` — never return `null`. If the mutation fails and you return `null` for `question_id`, make sure it's nullable (`Int` not `Int!`) or the GraphQL response will contain an error. Audit your payload types to ensure nullability matches the actual data contract.

---

### Topic 5: Advanced Patterns — Fragments, Aliases, Query Batching

**Fragments — Reusable Field Sets:**

Fragments let you define a set of fields once and reuse them across multiple queries. This is especially useful when querying the same type through different paths:

```graphql
# Define fragment
fragment ProductCoreFields on ProductInterface {
  sku
  name
  price {
    regularPrice {
      amount {
        value
        currency
      }
    }
  }
  thumbnail {
    url
    label
  }
}

# Use in multiple queries
query FeaturedProducts {
  products(filter: { category_id: { eq: "10" } }, pageSize: 5) {
    items {
      ...ProductCoreFields
      # Extended fields only on configurable products
      ... on ConfigurableProduct {
        variants {
          attributes {
            code
            value_index
          }
        }
      }
    }
  }
}

query NewArrivals {
  products(filter: { created_at: { gt: "2024-01-01" } }, pageSize: 10) {
    items {
      ...ProductCoreFields
    }
  }
}
```

> **Why fragments matter in production:** If your frontend uses TypeScript or Flow generated from the GraphQL schema, fragments ensure type consistency across components. A component requesting `...ProductCoreFields` gets the same fields regardless of which parent query fetched the product.

**Aliases — Rename Fields in Response:**

Aliases let you request the same field with different arguments in a single query:

```graphql
query ProductPricing {
  # Same product queried twice with different filters
  standard_price: products(filter: { sku: { eq: "SKU123" } }) {
    items {
      price {
        minimalPrice {
          amount { value }
        }
      }
    }
  }
  tier_price: products(filter: { sku: { eq: "SKU123" }, tier_price: { gt: 0 } }) {
    items {
      price {
        minimalPrice {
          amount { value }
        }
      }
    }
  }
}
```

Response:
```json
{
  "data": {
    "standard_price": { "items": [{ "price": { "minimalPrice": { "amount": { "value": 99.00 } } } }] },
    "tier_price": { "items": [{ "price": { "minimalPrice": { "amount": { "value": 79.00 } } } }] }
  }
}
```

**Query Batching — Multiple Operations Per Request:**

The GraphQL spec allows batching multiple top-level operations in a single HTTP request. This reduces HTTP overhead for related operations:

```graphql
# Single request with two operations
{
  "query": "
    query GetCart($cartId: String!) {
      cart(cart_id: $cartId) { total_quantity }
    }
    query GetCustomer {
      customer { firstname lastname email }
    }
  "
}
```

> **Magento limitation:** Magento's GraphQL endpoint processes operations sequentially and returns an array of results. However, Magento does not support the full GraphQL batch specification where one operation's result can influence another in the same batch. For complex interdependent data, a single combined query with nested fields is more reliable.

**Introspection — Discovering the Schema:**

Frontend tooling (Apollo, Relay, GraphQL Code Generator) uses introspection to build typed clients. You can query the schema directly:

```graphql
# Find all types starting with "Product"
query {
  __schema {
    types {
      name
      kind
      description
      fields(includeDeprecated: true) {
        name
        type {
          name
          kind
        }
        args {
          name
          type { name kind }
        }
      }
    }
  }
}
```

**Meta Fields:**

```graphql
query {
  products(pageSize: 1) {
    items {
      __typename  # Returns: "SimpleProduct" or "ConfigurableProduct" etc.
      name
      ... on ConfigurableProduct {
        __typename  # Returns "ConfigurableProduct" even inside fragment
        variants { sku }
      }
    }
  }
}
```

`__typename` is essential for polymorphic queries — it tells the client which concrete type was returned, enabling correct type narrowing in typed frontend code.

---

### Topic 6: GraphQL Caching — CacheIdentity and Cache Tags

**Why Cache GraphQL?**

GraphQL queries can be expensive — a single product query might trigger multiple repository calls, Elasticsearch queries, and price calculations. Unlike REST where caching is URL-based, GraphQL caching is field-level and identity-based.

Magento's GraphQL caching (introduced in 2.4) works at the resolver level through the `CacheIdentityInterface`.

**CacheIdentityInterface:**

Implement `Magento\Framework\GraphQl\Query\ResolverInterface` AND `Magento\Framework\GraphQl\Query\Resolver\CacheIdentityInterface` on the same resolver class:

```php
<?php
declare(strict_types=1);

namespace Training\ProductQuestion\Model\Resolver;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Query\Resolver\CacheIdentityInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Framework\GraphQl\Config\Element\Field;

class QuestionList implements ResolverInterface, CacheIdentityInterface
{
    private QuestionRepositoryInterface $questionRepository;

    public function __construct(QuestionRepositoryInterface $questionRepository)
    {
        $this->questionRepository = $questionRepository;
    }

    public function resolve(Field $field, $context, ResolveInfo $resolveInfo, array $resolvedArgs = []): array
    {
        $filter = $this->buildFilterFromArgs($resolvedArgs);
        $questions = $this->questionRepository->getList($filter);

        return [
            'total_count' => $questions->getTotalCount(),
            'items' => array_map([$this, 'transformQuestion'], $questions->getItems()),
        ];
    }

    // --- CacheIdentityInterface implementation ---

    public function getCacheIdentity(array $resolvedArgs): array
    {
        // Return unique cache key components based on query args
        $filter = $resolvedArgs['filter'] ?? [];
        $pageSize = (int) ($resolvedArgs['pageSize'] ?? 20);
        $currentPage = (int) ($resolvedArgs['currentPage'] ?? 1);

        // Build a unique tag from filter parameters
        $filterKey = md5(json_encode($filter));

        return [
            'training_question_list',
            'ps' . $pageSize,
            'cp' . $currentPage,
            'f_' . $filterKey,
        ];
    }

    public function getCacheTags(array $resolvedArgs): array
    {
        // Return tags for cache invalidation — tagged invalidation is precise
        $tags = ['training_question_list_global'];

        // If filtering by product, tag with that product's cache tag
        $filter = $resolvedArgs['filter'] ?? [];
        if (!empty($filter['product_sku']['eq'])) {
            $tags[] = 'product_' . $filter['product_sku']['eq'];
        }

        return $tags;
    }

    public function getCacheLifetime(array $resolvedArgs): ?int
    {
        // Return null to disable caching for this query
        // Return integer seconds to enable caching
        // 300 = 5 minutes — reasonable for question lists
        return 300;
    }

    private function transformQuestion(QuestionInterface $question): array
    {
        return [
            'question_id' => $question->getId(),
            'product_sku' => $question->getProductSku(),
            'customer_name' => $question->getCustomerName(),
            'question_text' => $question->getQuestionText(),
        ];
    }

    private function buildFilterFromArgs(array $args): array
    {
        $filter = [];
        if (!empty($args['filter']['product_sku'])) {
            $filter['product_sku'] = ['eq' => $args['filter']['product_sku']['eq']];
        }
        return $filter;
    }
}
```

**When NOT to Cache:**

Not every resolver should be cached. General rules:

| Cache | Don't Cache |
|-------|-------------|
| Queries with stable results (product lists, categories) | Queries with user-specific data (cart, orders, customer profile) |
| Public data any guest can access | Protected data requiring authentication |
| Data with known TTL (prices change but infrequently) | Real-time data (inventory, stock status) |
| Queries without side effects | Mutations (never cache) |

> **Important:** When you implement `CacheIdentityInterface`, you MUST return a non-null cache identity array from `getCacheIdentity()`. If the array is empty, Magento will not cache the result even if `getCacheLifetime()` returns a positive value.

**Cache Invalidation Observer:**

When an entity is modified, the associated GraphQL cache must be invalidated:

```php
<?php
declare(strict_types=1);

namespace Training\ProductQuestion\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Framework\GraphQl\Query\Resolver\CacheInvalidatorInterface;

/**
 * Invalidate question list cache when a question is saved
 */
class QuestionSaveAfter implements ObserverInterface
{
    private CacheInvalidatorInterface $cacheInvalidator;

    public function __construct(CacheInvalidatorInterface $cacheInvalidator)
    {
        $this->cacheInvalidator = $cacheInvalidator;
    }

    public function execute(Observer $observer): void
    {
        /** @var QuestionInterface $question */
        $question = $observer->getEvent()->getData('question');

        // Invalidate the global list cache
        $this->cacheInvalidator->invalidate([
            'training_question_list_global',
        ]);

        // Invalidate product-specific list cache
        $this->cacheInvalidator->invalidate([
            'product_' . $question->getProductSku(),
        ]);
    }
}
```

**events.xml wiring:**

```xml
<event name="training_question_save_after">
    <observer name="invalidate_question_list_cache"
              instance="Training\ProductQuestion\Observer\QuestionSaveAfter"/>
</event>
```

> **Pro Tip:** Use a wildcard-based tag like `training_question_list_global` for the full list cache, then specific product tags like `product_SKU123` for filtered lists. This way, saving a question for product SKU123 only invalidates the cache entries that actually contain that product — not the entire question list.

---

### Topic 7: Security — Query Complexity, Depth Limiting, Field Authorization

**Why GraphQL Security Differs from REST:**

REST's security model is endpoint-based — you protect a URL. GraphQL's single endpoint with a flexible query language creates new attack surfaces: a client could craft a deeply nested query that strains your database, or a query that requests thousands of fields simultaneously.

**Query Complexity:**

Query complexity measures how many fields are requested, weighted by field cost. Expensive fields (full-text search, aggregated data) cost more than simple scalar fields.

Magento's complexity limiter is configured via `di.xml`:

```xml
<type name="Magento\Framework\GraphQlQlQuery\QueryComplexityLimiter">
    <arguments>
        <argument name="maxComplexity" xsi:type="number">300</argument>
        <argument name="maxDepth" xsi:type="number">15</argument>
    </arguments>
</type>
```

> **What "depth" means:** Each nesting level adds 1 to the depth count. `products { items { price { amount { value } } } }` has depth 4. Circular fragment spreads can cause depth to grow exponentially.

**Query Depth Limiting:**

Depth limiting prevents circular queries that would otherwise cause infinite recursion:

```graphql
# This would be depth 3 without a depth limit
query {
  products {
    items {
      categories {
        products {   # Same fields requested again — circular
          categories { products { categories { products } } }
        }
      }
    }
  }
}
```

> **Note:** Magento's depth limiter works in combination with complexity limiting. A query can be rejected for being too complex even if its depth is within limits, or vice versa. Test your worst-case queries with the playground.

**Field-Level Authorization in Resolvers:**

The resolver's `$context` object contains user information:

```php
<?php
declare(strict_types=1);

namespace Training\ProductQuestion\Model\Resolver;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Framework\GraphQl\Config\Element\Field;

class Question implements ResolverInterface
{
    public function resolve(Field $field, $context, ResolveInfo $resolveInfo, array $resolvedArgs = [])
    {
        $userType = $context->getUserType();
        $userId = $context->getUserId();

        switch ($userType) {
            case \Magento\Authorization\Model\UserContextInterface::USER_TYPE_ADMIN:
                // Admins can see everything
                return $this->getFullQuestion($resolvedArgs['id']);

            case \Magento\Authorization\Model\UserContextInterface::USER_TYPE_CUSTOMER:
                // Customers can only see their own questions
                return $this->getCustomerQuestion((int) $resolvedArgs['id'], (int) $userId);

            default:
                // Guest — no access to question details (return null, not an error)
                return null;
        }
    }

    private function getFullQuestion(int $id): array { /* ... */ }
    private function getCustomerQuestion(int $id, int $customerId): array { /* ... */ }
}
```

**Admin vs Customer vs Guest Access Patterns:**

```php
// Pattern 1: Public field — accessible to everyone
// Return data without any user context checks

// Pattern 2: Customer-only field — check userId
if ($context->getUserId() === null) {
    return null; // Guest gets null, not an error
}
$customerId = (int) $context->getUserId();

// Pattern 3: Admin-only field — check userType
if ($context->getUserType() !== \Magento\Authorization\Model\UserContextInterface::USER_TYPE_ADMIN) {
    return null;
}

// Pattern 4: ACL resource check (for admin panel actions)
$aclContext = $context->getExtensionAttributes()->getAclResource();
if (!$this->aclChecker->isAllowed($aclContext, 'Training_ProductQuestion::questions_view')) {
    return null;
}
```

**Input Validation in Resolvers:**

Always validate and sanitize resolver input before database access — never trust the GraphQL arguments:

```php
public function resolve(Field $field, $context, ResolveInfo $resolveInfo, array $resolvedArgs = [])
{
    $input = $resolvedArgs['input'] ?? [];

    // Whitelist + type cast to prevent injection
    $productSku = $this->sanitize->stripTags($input['product_sku'] ?? '');
    if (!$this->validator->isValidSku($productSku)) {
        throw new \Magento\Framework\Exception\InputException(
            __('Invalid product SKU format')
        );
    }

    // Cast to int to prevent SQL-like injection
    $questionId = (int) ($resolvedArgs['id'] ?? 0);
    if ($questionId <= 0) {
        throw new \Magento\Framework\Exception\InputException(
            __('Question ID must be a positive integer')
        );
    }

    // Proceed with validated data
}
```

> **Common Mistake — Trusting GraphQL nullability:** Just because a field is marked `String!` in the schema doesn't mean it passed validation — it only means the client sent a value. A client could send `""` (empty string) where you expect a non-empty SKU. Always validate the content, not just the presence.

**Rate Limiting GraphQL:**

GraphQL itself doesn't have built-in rate limiting — rate limiting is typically handled at the HTTP layer (Varnish, nginx, or a API gateway). However, you can detect expensive queries and respond accordingly:

```php
// In a before plugin on Magento\Framework\GraphQl\Query\Resolver::resolve
public function beforeResolve(ResolverInterface $resolver, ...): array
{
    $queryString = $this->request->getParam('query');
    $depth = $this->queryAnalyzer->estimateDepth($queryString);

    if ($depth > $this->maxAllowedDepth) {
        throw new \Magento\Framework\GraphQl\Exception\GraphQlInputException(
            __('Query exceeds maximum allowed depth of %1', $this->maxAllowedDepth)
        );
    }

    return [];
}
```

> **Pro Tip:** For headless deployments, implement rate limiting at the CDN/edge layer (Fastly, Cloudflare) rather than in PHP. By the time a query reaches PHP, it has already consumed CPU and memory. Rate limiting at the edge drops the request before it hits your application servers.

---

### Topic 8: Real-World Patterns — Search, Aggregation, Custom Entities

**Full-Text Search via GraphQL:**

Magento's product search is powered by Elasticsearch. To implement a custom search resolver:

```php
<?php
declare(strict_types=1);

namespace Training\ProductQuestion\Model\Resolver;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Elasticsearch\Model\ResourceModel\Search\ClientFactory;
use Magento\Framework\Search\Request\Builder as RequestBuilder;

class SearchQuestions implements ResolverInterface
{
    private ClientFactory $searchClientFactory;
    private RequestBuilder $requestBuilder;

    public function __construct(
        ClientFactory $searchClientFactory,
        RequestBuilder $requestBuilder
    ) {
        $this->searchClientFactory = $searchClientFactory;
        $this->requestBuilder = $requestBuilder;
    }

    public function resolve(Field $field, $context, ResolveInfo $resolveInfo, array $resolvedArgs = []): array
    {
        $query = $resolvedArgs['query'] ?? '';
        $pageSize = (int) ($resolvedArgs['pageSize'] ?? 20);
        $currentPage = (int) ($resolvedArgs['currentPage'] ?? 1);

        $searchQuery = [
            'index' => 'training_questions',
            'body' => [
                'from' => ($currentPage - 1) * $pageSize,
                'size' => $pageSize,
                'query' => [
                    'multi_match' => [
                        'query' => $query,
                        'fields' => ['question_text^2', 'customer_name', 'product_sku'],
                        'type' => 'best_fields',
                        'fuzziness' => 'AUTO',
                    ],
                ],
                'highlight' => [
                    'fields' => [
                        'question_text' => new \stdClass(),
                    ],
                ],
            ],
        ];

        /** @var \Elasticsearch\Client $client */
        $client = $this->searchClientFactory->create();
        $response = $client->search($searchQuery);

        $items = [];
        foreach ($response['hits']['hits'] as $hit) {
            $items[] = [
                'question_id' => $hit['_source']['question_id'],
                'product_sku' => $hit['_source']['product_sku'],
                'question_text' => $hit['_source']['question_text'],
                'score' => $hit['_score'],
                'highlight' => $hit['highlight']['question_text'][0] ?? null,
            ];
        }

        return [
            'total_count' => $response['hits']['total']['value'] ?? 0,
            'items' => $items,
        ];
    }
}
```

**Aggregation and Facets:**

Aggregations return the "buckets" for filter UI — counts per category, price ranges, attribute values. In Magento's GraphQL, the `aggregations` field is built into the product query:

```graphql
query {
  products(
    filter: { price: { from: "50", to: "200" } }
    pageSize: 10
  ) {
    aggregations {
      attribute_code
      label
      count
      options {
        label
        value
        count
      }
    }
    items { name sku }
  }
}
```

For custom aggregations in your own resolver:

```php
public function resolve(...): array
{
    $aggregations = [
        [
            'attribute_code' => 'product_sku',
            'label' => 'Product',
            'count' => $questions->getSize(),
            'options' => $this->buildSkuBuckets($questions),
        ],
    ];

    return [
        'items' => $questions,
        'aggregations' => $aggregations,
    ];
}
```

**Exposing Extension Attributes via GraphQL:**

Extension attributes are the cleanest way to add fields to existing Magento entities (products, customers, orders) without modifying the core EAV tables. The extension attribute is backed by a `db_schema.xml` table, and a plugin enriches the entity at load time:

**Step 1 — Declare extension attribute in `etc/extension_attributes.xml`:**

```xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Extension/etc/extension_attribute.xsd">
    <extension_attributes for="Magento\Catalog\Api\Data\ProductInterface">
        <attribute code="pim_last_synced_at" type="string"/>
        <attribute code="pim_origin_id" type="string"/>
    </extension_attributes>
</config>
```

**Step 2 — Plugin on ProductRepository to populate from PIM:**

```php
<?php
declare(strict_types=1);

namespace Training\PimSync\Plugin;

use Magento\Catalog\Api\Data\ProductInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Training\PimSync\Api\PimClientInterface;

class EnrichProductFromPim
{
    private PimClientInterface $pimClient;

    public function __construct(PimClientInterface $pimClient)
    {
        $this->pimClient = $pimClient;
    }

    public function afterGet(
        ProductRepositoryInterface $subject,
        ProductInterface $product,
        string $sku
    ): ProductInterface {
        $pimData = $this->pimClient->getProductBySku($sku);
        if ($pimData) {
            $extension = $product->getExtensionAttributes();
            $extension->setPimLastSyncedAt($pimData['last_synced_at']);
            $extension->setPimOriginId($pimData['origin_id']);
            $product->setExtensionAttributes($extension);
        }
        return $product;
    }
}
```

**Step 3 — Extend the Product type in GraphQL schema:**

```graphql
extend type Product {
  pim_last_synced_at: String @doc(description: "Last sync timestamp from PIM")
  pim_origin_id: String @doc(description: "Product ID in the external PIM")
}
```

**Step 4 — Wire a resolver for the extension attribute (optional — if data isn't loaded via plugin):**

```php
<?php
declare(strict_types=1);

namespace Training\PimSync\Model\Resolver;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Catalog\Api\Data\ProductInterface;

class ProductPimData implements ResolverInterface
{
    public function resolve($field, $context, ResolveInfo $resolveInfo, array $resolvedArgs = [])
    {
        /** @var ProductInterface $product */
        $product = $resolvedArgs['model']; // Parent value contains the product

        $extension = $product->getExtensionAttributes();
        if (!$extension) {
            return null;
        }

        $fieldName = $resolveInfo->fieldName;
        switch ($fieldName) {
            case 'pim_last_synced_at':
                return $extension->getPimLastSyncedAt();
            case 'pim_origin_id':
                return $extension->getPimOriginId();
            default:
                return null;
        }
    }
}
```

**Pagination Patterns:**

Magento's GraphQL uses offset-based pagination (not cursor-based):

```graphql
query {
  products(pageSize: 20, currentPage: 2) {
    total_count
    page_info {
      current_page
      page_size
      total_pages
    }
    items { sku name }
  }
}
```

For cursor-based pagination (useful for infinite scroll), implement it in your custom resolver:

```php
public function resolve(...): array
{
    $searchAfter = $resolvedArgs['searchAfter'] ?? null;
    $pageSize = (int) ($resolvedArgs['pageSize'] ?? 20);

    $query = [
        'body' => [
            'size' => $pageSize + 1, // Fetch one extra to determine if there's a next page
            'search_after' => $searchAfter,
            'sort' => [
                ['question_id' => 'asc'],
                ['_score' => 'desc'],
            ],
            'query' => [ /* ... */ ],
        ],
    ];

    $response = $this->elasticsearchClient->search($query);
    $hits = $response['hits']['hits'];

    $hasNextPage = count($hits) > $pageSize;
    if ($hasNextPage) {
        array_pop($hits); // Remove the extra item
    }

    $nextCursor = $hasNextPage && !empty($hits)
        ? $hits[count($hits) - 1]['sort']
        : null;

    return [
        'items' => $hits,
        'page_info' => [
            'start_cursor' => $hits[0]['sort'] ?? null,
            'end_cursor' => $nextCursor,
            'has_next_page' => $hasNextPage,
        ],
    ];
}
```

> **Pagination choice:** Offset pagination (`currentPage/pageSize`) is simpler and works well for regular page navigation with page numbers. Cursor pagination (`searchAfter`) is better for infinite scroll and handles real-time data better because it doesn't skip items when results are inserted/deleted between page requests.

---

### Common Mistakes

- **Skipping `di.xml` resolver wiring:** Forgetting to register your resolver in `etc/graphql/di.xml` results in the query returning `null` silently with no error. Always verify the wiring.
- **Returning `null` from a non-nullable field:** If your schema defines `field: String!` but the resolver returns `null`, the GraphQL response will contain an error. Use nullable types in the schema or ensure resolvers always return a value.
- **Not implementing `CacheIdentityInterface` fully:** Implementing only `getCacheTags()` without `getCacheIdentity()` means the cache is never actually stored — only invalidated. All three methods must be implemented together.
- **Mutating data in a cached query resolver:** A resolver that implements `CacheIdentityInterface` must be idempotent. If calling the resolver also modifies state (e.g., logging, counter updates), move those side effects to a separate non-cached operation.
- **Missing `@doc` directives on custom schema fields:** Without documentation, frontend developers have no description for the field in the playground's schema explorer.
- **Not validating GraphQL input before database access:** GraphQL arguments bypass Magento's form key and HTML escaping. Always sanitize string inputs and type-cast numeric inputs.
- **Using `@resolver` directive when di.xml is needed:** The `@resolver(class: ...)` directive works for simple resolvers but doesn't support constructor injection of multiple dependencies. Use `di.xml` for anything beyond a simple single-dependency resolver.

---

### Reading List

1. **Adobe Commerce GraphQL Overview** — https://developer.adobe.com/commerce/webapi/graphql/
2. **Magento 2 GraphQL Developer Guide** — https://developer.adobe.com/commerce/services/graphql/ (covers core queries, mutations, and schema extension)
3. **GraphQL Schema Design Best Practices** — https://graphql.org/learn/schema/
4. **Magento CacheIdentityInterface Reference** — https://developer.adobe.com/commerce/php/development/components/graphql/ (search: "cache resolvers")
5. **Elasticsearch PHP Client Documentation** — https://www.elastic.co/guide/en/elasticsearch/client/php-api/current/

---

*Magento 2 Backend Developer Course — Topic 18*
