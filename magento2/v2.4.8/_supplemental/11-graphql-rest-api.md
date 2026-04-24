---
title: "36 - GraphQL & REST API Deep Dive"
description: "Deep dive into Magento 2.4.8 APIs: GraphQL schema, resolvers, mutations, REST endpoints, authentication, and web API performance."
tags: magento2, graphql, rest-api, webapi, api-authentication, resolvers, mutations, rest-endpoints
rank: 11
pathways: [magento2-deep-dive]
---

# GraphQL & REST API Deep Dive

Magento 2.4.8 provides two primary web API approaches: **GraphQL** for flexible, client-optimized queries, and **REST** for traditional resource-based operations. This article explores the architecture, implementation patterns, authentication, and performance optimization strategies for both.

## 1. Web API Architecture

### Service Contracts

Magento's web API is built on **service contracts**—PHP interfaces that define the public API of each module. These contracts ensure loose coupling and allow the API framework to expose any properly configured service.

```php
// Magento/Catalog/Api/ProductRepositoryInterface.php
declare(strict_types=1);

namespace Magento\Catalog\Api;

use Magento\Catalog\Api\Data\ProductInterface;
use Magento\Catalog\Api\Data\ProductSearchResultsInterface;
use Magento\Framework\Api\SearchCriteriaInterface;

interface ProductRepositoryInterface
{
    /**
     * @param string $sku
     * @param bool $editMode
     * @param int|null $storeId
     * @param bool $forceReload
     * @return ProductInterface
     */
    public function get(string $sku, bool $editMode = false, ?int $storeId = null, bool $forceReload = false): ProductInterface;

    /**
     * @param SearchCriteriaInterface $searchCriteria
     * @return ProductSearchResultsInterface
     */
    public function getList(SearchCriteriaInterface $searchCriteria): ProductSearchResultsInterface;

    /**
     * @param ProductInterface $product
     * @return ProductInterface
     */
    public function save(ProductInterface $product): ProductInterface;

    /**
     * @param ProductInterface $product
     * @return bool
     */
    public function delete(ProductInterface $product): bool;

    /**
     * @param string $sku
     * @return bool
     */
    public function deleteById(string $sku): bool;
}
```

### Why GraphQL vs REST?

| Aspect | GraphQL | REST |
|--------|---------|------|
| **Data Fetching** | Single request, exact fields needed | Multiple endpoints, over/under-fetching |
| **Schema** | Type-safe, introspectable | No schema enforcement |
| **Caching** | Complex (POST-only by default) | Simple HTTP caching |
| **Real-time** | Subscriptions supported | Webhooks needed |
| **Mobile Performance** | Excellent (batched queries) | Multiple round trips |
| **Learning Curve** | Steeper | Familiar patterns |

**When to use GraphQL:**
- Headless/PWA storefronts
- Mobile applications
- Complex, nested data requirements
- Clients needing precise data shape

**When to use REST:**
- Simple CRUD operations
- When HTTP caching is critical
- Third-party integrations
- SOAP requirements

### API Framework Request Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        Request Flow                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  HTTP Request                                                    │
│       │                                                          │
│       ▼                                                          │
│  ┌─────────────┐    ┌──────────────┐    ┌────────────────────┐  │
│  │ URL Routing │───▶│ Auth Manager │───▶│ ACL Permission Check│  │
│  └─────────────┘    └──────────────┘    └────────────────────┘  │
│                                                 │                │
│                                                 ▼                │
│  Response                            ┌────────────────────┐     │
│       ▲                              │   Service Method   │     │
│       │                              │   (Repository)    │     │
│  ┌─────────────┐                     └────────────────────┘     │
│  │ Serializer │                                                   │
│  └─────────────┘                                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 2. GraphQL Fundamentals

### Schema Structure

The GraphQL schema in Magento 2.4.8 is defined using `.graphqls` files (GraphQL SDL). The base schema is defined in `Magento/GraphQl/etc/schema.graphqls`, and each module extends it.

```graphql
# app/code/Magento/CatalogGraphQl/etc/schema.graphqls

type Query {
    products(
        search: String @doc(description: "Full text search query")
        filter: ProductAttributeFilterInput @doc(description: "Filter criteria")
        pageSize: Int = 20 @doc(description: "Number of items per page")
        currentPage: Int = 1 @doc(description: "Current page number")
        sort: ProductAttributeSortInput @doc(description: "Sort criteria")
    ): Products @resolver(class: "Magento\\CatalogGraphQl\\Model\\Resolver\\Products") @cache(cacheIdentity: "Magento\\CatalogGraphQl\\Model\\Resolver\\Products\\Identity")
}

type Mutation {
    createProduct(input: ProductCreateInput!): ProductOutput @resolver(class: "Magento\\CatalogGraphQl\\Model\\Resolver\\ProductCreate") @doc(description: "Create a new product")
    updateProduct(input: ProductUpdateInput!): ProductOutput @resolver(class: "Magento\\CatalogGraphQl\\Model\\Resolver\\ProductUpdate") @doc(description: "Update existing product")
    deleteProduct(input: DeleteProductInput!): Boolean @resolver(class: "Magento\\CatalogGraphQl\\Model\\Resolver\\ProductDelete") @doc(description: "Delete a product")
}

type Products @doc(description: "Product search results") {
    items: [ProductInterface] @doc(description: "Array of products")
    page_info: SearchResultPageInfo @doc(description: "Pagination information")
    total_count: Int @doc(description: "Total number of items found")
    aggregations: [Aggregation] @doc(description: "Facet aggregations")
}

interface ProductInterface @typeResolver(class: "Magento\\CatalogGraphQl\\Model\\Resolver\\ProductTypeResolver") {
    id: Int! @doc(description: "The product ID")
    sku: String! @doc(description: "The product SKU")
    name: String @doc(description: "The product name")
    description: String @doc(description: "The product description")
    short_description: String @doc(description: "Short description")
    price_range: PriceRange @resolver(class: "Magento\\CatalogGraphQl\\Model\\Resolver\\PriceRange")
    meta_title: String @doc(description: "Meta title")
    meta_description: String @doc(description: "Meta description")
    image: String @doc(description: "Product image")
    small_image: String @doc(description: "Small product image")
    thumbnail: String @doc(description: "Product thumbnail")
    stock_status: ProductStockStatus @enum(self)
    categories: [CategoryInterface]
    price_range: PriceRange
    ... on ConfigurableProduct {
        configurable_options: [ConfigurableOption] @resolver(class: "Magento\\ConfigurableProductGraphQl\\Model\\Resolver\\ConfigurableOptions")
        variants: [ConfigurableVariant]
    }
}
```

### Input Types (Filters)

```graphql
# Filter input type for products
input ProductAttributeFilterInput {
    entity_id: FilterTypeInput @doc(description: "Entity ID filter")
    sku: FilterTypeInput @doc(description: "SKU filter")
    name: FilterTypeInput @doc(description: "Product name filter")
    price: FilterTypeInput @doc(description: "Price filter")
    description: FilterTypeInput @doc(description: "Description filter")
    short_description: FilterTypeInput @doc(description: "Short description filter")
    meta_title: FilterTypeInput @doc(description: "Meta title filter")
    meta_description: FilterTypeInput @doc(description: "Meta description filter")
    category_id: FilterTypeInput @doc(description: "Category ID filter")
    visibility: FilterTypeInput @doc(description: "Visibility filter")
    status: FilterTypeInput @doc(description: "Status filter")
    store_id: FilterTypeInput @doc(description: "Store ID filter")
    created_at: FilterTypeInput @doc(description: "Creation date filter")
    updated_at: FilterTypeInput @doc(description: "Update date filter")
    _search: FilterTypeInput @doc(description: "Full text search")
    category_uid: FilterTypeInput @doc(description: "Category UID filter")
    or: ProductAttributeFilterInput @doc(description: "OR condition")
}

input FilterTypeInput {
    in: [String] @doc(description: "Filter by value in array")
    nin: [String] @doc(description: "Filter by value NOT in array")
    eq: String @doc(description: "Filter by exact match")
    neq: String @doc(description: "Filter by not equal")
    like: String @doc(description: "Filter by LIKE pattern")
    rlike: String @doc(description: "Filter by REGEXP pattern")
    llike: String @doc(description: "Filter by LEFT LIKE pattern")
    from: String @doc(description: "Filter from value")
    to: String @doc(description: "Filter to value")
    is: String @doc(description: "Filter by is operator")
    is_not: String @doc(description: "Filter by is not operator")
}

input ProductAttributeSortInput {
    name: SortEnum @doc(description: "Sort by name")
    price: SortEnum @doc(description: "Sort by price")
    relevance: SortEnum @doc(description: "Sort by relevance")
    position: SortEnum @doc(description: "Sort by position")
    entity_id: SortEnum @doc(description: "Sort by entity ID")
    created_at: SortEnum @doc(description: "Sort by creation date")
    updated_at: SortEnum @doc(description: "Sort by update date")
}
```

### Interfaces and Unions

```graphql
# Interface definition
interface CategoryInterface @typeResolver(class: "Magento\\CatalogGraphQl\\Model\\Resolver\\CategoryTypeResolver") {
    id: Int! @doc(description: "Category ID")
    parent_id: Int @doc(description: "Parent category ID")
    name: String @doc(description: "Category name")
    path: String @doc(description: "Category path")
    position: Int @doc(description: "Category position")
    level: Int @doc(description: "Category level")
    children: [CategoryInterface] @doc(description: "Child categories")
    products(count: Int @doc(description: "Number of products")): CategoryProducts
    breadcrumbs: [Breadcrumb] @doc(description: "Category breadcrumbs")
}

# Union for cart addresses (can be one of multiple types)
union CartAddressInterface = CartAddressShipping | CartAddressBilling

# Implementing types
type CartAddressShipping @doc(description: "Shipping cart address") {
    firstname: String
    lastname: String
    company: String
    street: [String]
    city: String
    region: String
    postcode: String
    country: String
    telephone: String
}

type CartAddressBilling @doc(description: "Billing cart address") {
    firstname: String
    lastname: String
    company: String
    street: [String]
    city: String
    region: String
    postcode: String
    country: String
    telephone: String
}
```

### Directives

Magento extends GraphQL with custom directives:

```graphql
@resolver(class: "Magento\\Module\\Model\\Resolver\\ClassName")
    # Specifies the PHP resolver class

@doc(description: "Description text")
    # Adds documentation to schema elements

@cache(cacheIdentity: "Magento\\Module\\Model\\Resolver\\Identity")
    # Enables caching for the field

@deprecated(reason: "Reason for deprecation")
    # Marks a field as deprecated

@typeResolver(class: "Magento\\Module\\Model\\TypeResolver")
    # Specifies type resolver for interfaces/unions

@query-resolver
    # Indicates this query should use query resolver pattern

@mutation-resolver
    # Indicates this mutation should use mutation resolver pattern
```

### Example GraphQL Queries

```graphql
# Get products with filtering and pagination
query GetProducts {
    products(
        filter: {
            price: { from: "10", to: "100" }
            category_id: { eq: "5" }
            status: { eq: "1" }
        }
        pageSize: 20
        currentPage: 1
        sort: { price: ASC }
    ) {
        items {
            id
            sku
            name
            price_range {
                minimum_price {
                    regular_price { value currency }
                    final_price { value currency }
                    discount { percent_off amount_off }
                }
            }
            thumbnail { url label }
            stock_status
            ... on ConfigurableProduct {
                configurable_options {
                    attribute_code
                    label
                    values { uid label }
                }
            }
        }
        total_count
        page_info {
            current_page
            page_size
            total_pages
        }
    }
}

# Create a customer mutation
mutation CreateCustomer {
    createCustomer(
        input: {
            email: "customer@example.com"
            firstname: "John"
            lastname: "Doe"
            password: "SecurePassword123!"
            password_hash: ""
        }
    ) {
        customer {
            id
            email
            firstname
            lastname
        }
        status
    }
}

# Get customer cart
query GetCustomerCart {
    customer {
        id
        email
        cart {
            id
            items {
                id
                quantity
                product {
                    sku
                    name
                    price_range {
                        minimum_price {
                            final_price { value }
                        }
                    }
                }
            }
            prices {
                grand_total { value currency }
                subtotal_including_tax { value currency }
            }
        }
    }
}
```

## 3. GraphQL Resolvers

### ResolverInterface

Every GraphQL resolver in Magento must implement `ResolverInterface`:

```php
// lib/internal/Magento/Framework/GraphQl/Query/ResolverInterface.php
declare(strict_types=1);

namespace Magento\Framework\GraphQl\Query;

use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;

/**
 * Resolver fetches the data and formats it according to the GraphQL schema.
 *
 * @api
 */
interface ResolverInterface
{
    /**
     * Fetches the data from persistence models and format it according to the GraphQL schema.
     *
     * @param Field $field
     * @param mixed $context Context can contain appState, caches, etc. Passed by GraphQL framework
     * @param ResolveInfo $resolveInfo Contains relevant data for resolving (parents, args, etc)
     * @param array|null $value Original value passed to resolver chain
     * @param array|null $args Original arguments passed to field
     * @return mixed Data to be returned for the resolved field
     */
    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    );
}
```

### ProductResolver Example

```php
// app/code/Magento/ConfigurableProductGraphQl/Model/Resolver/ProductResolver.php
declare(strict_types=1);

namespace Magento\ConfigurableProductGraphQl\Model\Resolver;

use Magento\ConfigurableProductGraphQl\Model\Resolver\Variants\Options\AttributeEnum;
use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Framework\Exception\LocalizedException;

/**
 * Fetches the Product data according to the GraphQL schema
 */
class ProductResolver implements ResolverInterface
{
    /**
     * @var ItemResolverInterface
     */
    private $configurableItemResolver;

    /**
     * @param ItemResolverInterface $configurableItemResolver
     */
    public function __construct(
        ItemResolverInterface $configurableItemResolver
    ) {
        $this->configurableItemResolver = $configurableItemResolver;
    }

    /**
     * @inheritdoc
     */
    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        if (!isset($value['sku'])) {
            return null;
        }

        $productId = $this->configurableItemResolver->resolveItem($value['sku']);

        if ($productId === null) {
            return null;
        }

        return [
            'model' => $productId,
            'sku' => $value['sku'],
            'parent_sku' => $value['sku'],
        ];
    }
}
```

### BatchResolverInterface for Performance

Magento recommends `BatchResolverInterface` for better performance through query batching:

```php
// lib/internal/Magento/Framework/GraphQl/Query/Resolver/BatchResolverInterface.php
declare(strict_types=1);

namespace Magento\Framework\GraphQl\Query\Resolver;

use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;

/**
 * Batch resolver for improving performance by resolving multiple values at once
 */
interface BatchResolverInterface
{
    /**
     * Resolve multiple values in a single batch operation
     *
     * @param array<array> $values Array of values to resolve, indexed by position in query
     * @param Field $field
     * @param mixed $context
     * @param ResolveInfo $info
     * @param array|null $args
     * @return array Resolved values in same order as input values
     */
    public function resolveBatch(
        array $values,
        Field $field,
        $context,
        ResolveInfo $info,
        array $args = null
    ): array;
}
```

### DataLoader Pattern

The DataLoader pattern prevents N+1 queries by batching and caching:

```php
// app/code/Magento/Deferred/Model/Product.php
declare(strict_types=1);

namespace Magento\Deferred\Model;

use Magento\Catalog\Api\Data\ProductInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Framework\App\ObjectManager;
use Magento\Framework\ObjectManager\ResetAfterRequestInterface;

/**
 * Deferred product resolver using DataLoader pattern for batch loading
 *
 * Implements request-scoped caching to prevent N+1 queries
 */
class Product implements ResetAfterRequestInterface
{
    /**
     * @var ProductRepositoryInterface
     */
    private $productRepository;

    /**
     * @var SearchCriteriaBuilder
     */
    private $searchCriteriaBuilder;

    /**
     * @var array
     */
    private $productList = [];

    /**
     * @var array
     */
    private $productSkus = [];

    /**
     * @var array
     */
    private $attributeCodes = [];

    /**
     * @param ProductRepositoryInterface $productRepository
     * @param SearchCriteriaBuilder $searchCriteriaBuilder
     */
    public function __construct(
        ProductRepositoryInterface $productRepository,
        SearchCriteriaBuilder $searchCriteriaBuilder
    ) {
        $this->productRepository = $productRepository;
        $this->searchCriteriaBuilder = $searchCriteriaBuilder;
    }

    /**
     * Request-scoped state reset for Swoole/long-running processes
     */
    public function _resetState(): void
    {
        $this->productList = [];
        $this->productSkus = [];
        $this->attributeCodes = [];
    }

    /**
     * Load products by SKUs in batch
     *
     * @param array $skus
     * @param array $attributeCodes
     * @return array
     */
    public function load(array $skus, array $attributeCodes = []): array
    {
        $skusToLoad = array_diff($skus, array_keys($this->productList));

        if (!empty($skusToLoad)) {
            $this->searchCriteriaBuilder->addFilter('sku', $skusToLoad, 'in');
            $searchCriteria = $this->searchCriteriaBuilder->create();
            $searchCriteria->setPageSize(count($skusToLoad));

            $products = $this->productRepository->getList($searchCriteria);

            foreach ($products->getItems() as $product) {
                $this->productList[$product->getSku()] = $product;
            }

            if (!empty($attributeCodes)) {
                $this->attributeCodes = array_unique(array_merge($this->attributeCodes, $attributeCodes));
            }
        }

        $result = [];
        foreach ($skus as $sku) {
            $result[$sku] = $this->productList[$sku] ?? null;
        }

        return $result;
    }

    /**
     * Get single product (calls batch internally)
     *
     * @param string $sku
     * @return ProductInterface|null
     */
    public function get(string $sku): ?ProductInterface
    {
        $result = $this->load([$sku]);
        return $result[$sku] ?? null;
    }
}
```

### Processor Pattern

The processor pattern handles complex resolver orchestration:

```php
// app/code/Magento/Framework/GraphQl/Query/Resolver/Processor/Processor.php
declare(strict_types=1);

namespace Magento\Framework\GraphQl\Query\Resolver\Processor;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Framework\GraphQl\Config\Element\Field;

/**
 * Processes resolver execution with caching and error handling
 */
class Processor
{
    /**
     * @var ResolverInterface
     */
    private $resolver;

    /**
     * @var ValueFactory
     */
    private $valueFactory;

    /**
     * @param ResolverInterface $resolver
     * @param ValueFactory $valueFactory
     */
    public function __construct(
        ResolverInterface $resolver,
        ValueFactory $valueFactory
    ) {
        $this->resolver = $resolver;
        $this->valueFactory = $valueFactory;
    }

    /**
     * Process resolver with caching support
     *
     * @param Field $field
     * @param mixed $context
     * @param ResolveInfo $info
     * @param array|null $value
     * @param array|null $args
     * @return mixed
     */
    public function process(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        $cacheKey = $this->getCacheKey($field, $value, $args);

        if ($this->shouldUseCache($field, $context)) {
            $cachedValue = $this->getFromCache($cacheKey);
            if ($cachedValue !== null) {
                return $cachedValue;
            }
        }

        try {
            $result = $this->resolver->resolve($field, $context, $info, $value, $args);

            if ($this->shouldUseCache($field, $context)) {
                $this->setToCache($cacheKey, $result);
            }

            return $result;
        } catch (\Exception $e) {
            throw $e;
        }
    }

    /**
     * Generate cache key for resolver result
     */
    private function getCacheKey(Field $field, array $value = null, array $args = null): string
    {
        return md5($field->getName() . serialize($value) . serialize($args));
    }

    /**
     * Check if field should use caching
     */
    private function shouldUseCache(Field $field, $context): bool
    {
        // Implementation checks @cache directive
        return false;
    }
}
```

## 4. Custom GraphQL Implementation

### Step 1: Define Schema

```graphql
# app/code/Sample/CustomGraphQl/etc/schema.graphqls

type Query {
    customProducts(
        filter: CustomProductFilterInput
        pageSize: Int = 20
        currentPage: Int = 1
    ): CustomProducts @resolver(class: "Sample\\CustomGraphQl\\Model\\Resolver\\CustomProducts")
    customEntity(id: Int!): CustomEntity @resolver(class: "Sample\\CustomGraphQl\\Model\\Resolver\\CustomEntity")
}

type Mutation {
    createCustomEntity(
        input: CustomEntityInput!
    ): CustomEntityOutput @resolver(class: "Sample\\CustomGraphQl\\Model\\Resolver\\CreateCustomEntity")
    updateCustomEntity(
        id: Int!
        input: CustomEntityInput!
    ): CustomEntityOutput @resolver(class: "Sample\\CustomGraphQl\\Model\\Resolver\\UpdateCustomEntity")
}

input CustomProductFilterInput {
    entity_id: FilterTypeInput
    sku: FilterTypeInput
    name: FilterTypeInput
    price: FilterTypeInput
    created_at: FilterTypeInput
}

input CustomEntityInput {
    name: String!
    description: String
    price: Float!
    status: Int!
    custom_attributes: [CustomAttributeInput]
}

type CustomProducts {
    items: [CustomProduct]
    total_count: Int!
    page_info: SearchResultPageInfo
}

type CustomProduct {
    id: Int!
    sku: String!
    name: String!
    price: Float!
    created_at: String
    updated_at: String
}

type CustomEntity {
    id: Int!
    name: String!
    description: String
    price: Float
    status: Int
}

type CustomEntityOutput {
    entity: CustomEntity
    status: Boolean!
}
```

### Step 2: Create Resolver Class

```php
// app/code/Sample/CustomGraphQl/Model/Resolver/CustomProducts.php
declare(strict_types=1);

namespace Sample\CustomGraphQl\Model\Resolver;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Framework\Api\FilterBuilder;
use Sample\CustomGraphQl\Api\CustomProductRepositoryInterface;

class CustomProducts implements ResolverInterface
{
    /**
     * @var CustomProductRepositoryInterface
     */
    private $customProductRepository;

    /**
     * @var SearchCriteriaBuilder
     */
    private $searchCriteriaBuilder;

    /**
     * @var FilterBuilder
     */
    private $filterBuilder;

    /**
     * @param CustomProductRepositoryInterface $customProductRepository
     * @param SearchCriteriaBuilder $searchCriteriaBuilder
     * @param FilterBuilder $filterBuilder
     */
    public function __construct(
        CustomProductRepositoryInterface $customProductRepository,
        SearchCriteriaBuilder $searchCriteriaBuilder,
        FilterBuilder $filterBuilder
    ) {
        $this->customProductRepository = $customProductRepository;
        $this->searchCriteriaBuilder = $searchCriteriaBuilder;
        $this->filterBuilder = $filterBuilder;
    }

    /**
     * @inheritdoc
     */
    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        $this->validateSchema();

        if (isset($args['filter'])) {
            $this->applyFilters($args['filter']);
        }

        $searchCriteria = $this->searchCriteriaBuilder->create();
        $searchCriteria->setPageSize($args['pageSize'] ?? 20);
        $searchCriteria->setCurrentPage($args['currentPage'] ?? 1);

        $searchResult = $this->customProductRepository->getList($searchCriteria);

        return [
            'items' => $this->mapItems($searchResult->getItems()),
            'total_count' => $searchResult->getTotalCount(),
            'page_info' => [
                'current_page' => $searchCriteria->getCurrentPage(),
                'page_size' => $searchCriteria->getPageSize(),
                'total_pages' => ceil($searchResult->getTotalCount() / $searchCriteria->getPageSize()),
            ],
        ];
    }

    /**
     * Apply filters to search criteria
     */
    private function applyFilters(array $filters): void
    {
        foreach ($filters as $field => $condition) {
            if (is_array($condition)) {
                foreach ($condition as $operator => $value) {
                    $filter = $this->filterBuilder
                        ->setField($field)
                        ->setValue($value)
                        ->setConditionType($this->mapOperator($operator))
                        ->create();
                    $this->searchCriteriaBuilder->addFilter($filter);
                }
            }
        }
    }

    /**
     * Map GraphQL operator to Magento condition type
     */
    private function mapOperator(string $operator): string
    {
        $operatorMap = [
            'eq' => 'eq',
            'neq' => 'neq',
            'like' => 'like',
            'in' => 'in',
            'from' => 'gteq',
            'to' => 'lteq',
        ];

        return $operatorMap[$operator] ?? 'eq';
    }

    /**
     * Map repository items to GraphQL response
     */
    private function mapItems(array $items): array
    {
        return array_map(function ($item) {
            return [
                'id' => $item->getId(),
                'sku' => $item->getSku(),
                'name' => $item->getName(),
                'price' => $item->getPrice(),
                'created_at' => $item->getCreatedAt(),
                'updated_at' => $item->getUpdatedAt(),
            ];
        }, $items);
    }

    /**
     * Validate schema access
     */
    private function validateSchema(): void
    {
        // Schema validation logic
    }
}
```

### Step 3: Register in di.xml

```xml
<!-- app/code/Sample/CustomGraphQl/etc/di.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Query Resolvers -->
    <type name="Magento\Framework\GraphQl\Query\ResolverPool">
        <arguments>
            <argument name="resolvers" xsi:type="array">
                <item name="Sample\CustomGraphQl\Model\Resolver\CustomProducts"
                      xsi:type="object">Sample\CustomGraphQl\Model\Resolver\CustomProducts</item>
                <item name="Sample\CustomGraphQl\Model\Resolver\CustomEntity"
                      xsi:type="object">Sample\CustomGraphQl\Model\Resolver\CustomEntity</item>
            </argument>
        </arguments>
    </type>

    <!-- Mutation Resolvers -->
    <type name="Magento\Framework\GraphQl\Mutation\ResolverPool">
        <arguments>
            <argument name="resolvers" xsi:type="array">
                <item name="Sample\CustomGraphQl\Model\Resolver\CreateCustomEntity"
                      xsi:type="object">Sample\CustomGraphQl\Model\Resolver\CreateCustomEntity</item>
                <item name="Sample\CustomGraphQl\Model\Resolver\UpdateCustomEntity"
                      xsi:type="object">Sample\CustomGraphQl\Model\Resolver\UpdateCustomEntity</item>
            </argument>
        </arguments>
    </type>

    <!-- Type Resolvers -->
    <type name="Magento\Framework\GraphQl\Type\DataMapper\ProcessorPool">
        <arguments>
            <argument name="processors" xsi:type="array">
                <item name="CustomProduct"
                      xsi:type="object">Sample\CustomGraphQl\Model\Type\Processor\CustomProductProcessor</item>
            </argument>
        </arguments>
    </type>
</config>
```

### Step 4: Module Registration

```xml
<!-- app/code/Sample/CustomGraphQl/etc/module.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Sample_CustomGraphQl" setup_version="1.0.0">
        <sequence>
            <module name="Magento_GraphQl"/>
        </sequence>
    </module>
</config>
```

```php
// app/code/Sample/CustomGraphQl/registration.php
<?php

use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'Sample_CustomGraphQl',
    __DIR__
);
```

## 5. REST API

### webapi.xml Configuration

The `webapi.xml` file defines REST API routes:

```xml
<!-- app/code/Magento/Customer/etc/webapi.xml -->
<?xml version="1.0"?>
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Webapi:etc/webapi.xsd">

    <!-- Customer Routes -->
    <route url="/V1/customers" method="POST">
        <service class="Magento\Customer\Api\AccountManagementInterface" method="createAccount"/>
        <resources>
            <resource ref="anonymous"/>
        </resources>
    </route>

    <route url="/V1/customers/:customerId" method="GET">
        <service class="Magento\Customer\Api\CustomerRepositoryInterface" method="getById"/>
        <resources>
            <resource ref="Magento_Customer::read"/>
        </resources>
    </route>

    <route url="/V1/customers/me" method="GET">
        <service class="Magento\Customer\Api\CustomerRepositoryInterface" method="getById"/>
        <resources>
            <resource ref="self"/>
        </resources>
        <data>
            <parameter name="customerId" path="customer.id"/>
        </data>
    </route>

    <route url="/V1/customers/me" method="PUT">
        <service class="Magento\Customer\Api\CustomerAccountManagementInterface" method="update"/>
        <resources>
            <resource ref="self"/>
        </resources>
        <data>
            <parameter name="customer" path="customer"/>
        </data>
    </route>

    <!-- Product Routes -->
    <route url="/V1/products" method="GET">
        <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="getList"/>
        <resources>
            <resource ref="Magento_Catalog::products"/>
        </resources>
    </route>

    <route url="/V1/products/:sku" method="GET">
        <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="get"/>
        <resources>
            <resource ref="Magento_Catalog::products"/>
        </resources>
    </route>

    <route url="/V1/products" method="POST">
        <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="save"/>
        <resources>
            <resource ref="Magento_Catalog::products"/>
        </resources>
    </route>

    <route url="/V1/products/:sku" method="PUT">
        <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="save"/>
        <resources>
            <resource ref="Magento_Catalog::products"/>
        </resources>
    </route>

    <route url="/V1/products/:sku" method="DELETE">
        <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="deleteById"/>
        <resources>
            <resource ref="Magento_Catalog::products"/>
        </resources>
    </route>

    <!-- Order Routes -->
    <route url="/V1/orders" method="GET">
        <service class="Magento\Sales\Api\OrderRepositoryInterface" method="getList"/>
        <resources>
            <resource ref="Magento_Sales::orders"/>
        </resources>
    </route>

    <route url="/V1/orders/:orderId" method="GET">
        <service class="Magento\Sales\Api\OrderRepositoryInterface" method="get"/>
        <resources>
            <resource ref="Magento_Sales::orders"/>
        </resources>
    </route>
</routes>
```

### Request/Response Format

#### GET Request Example

```bash
# Get product by SKU
curl -X GET "https://example.com/rest/V1/products/test-product-001" \
  -H "Authorization: Bearer {admin_token}" \
  -H "Content-Type: application/json"
```

#### Response:
```json
{
    "id": 1,
    "sku": "test-product-001",
    "name": "Test Product",
    "attribute_set_id": 4,
    "status": 1,
    "visibility": 4,
    "price": 99.99,
    "type_id": "simple",
    "weight": 1.5,
    "extension_attributes": {
        "category_links": [
            {"category_id": "2", "position": 0}
        ],
        "stock_item": {
            "item_id": 1,
            "product_id": 1,
            "stock_id": 1,
            "qty": 100,
            "is_in_stock": true,
            "is_qty_decimal": false
        }
    },
    "custom_attributes": [
        {"attribute_code": "description", "value": "Product description"},
        {"attribute_code": "meta_title", "value": "Test Product"}
    ]
}
```

#### POST Request Example

```bash
# Create a simple product
curl -X POST "https://example.com/rest/V1/products" \
  -H "Authorization: Bearer {admin_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "product": {
      "sku": "new-product-001",
      "name": "New Product",
      "attribute_set_id": 4,
      "price": 49.99,
      "status": 1,
      "visibility": 4,
      "type_id": "simple",
      "weight": 1.0,
      "extension_attributes": {
        "stock_item": {
          "qty": 50,
          "is_in_stock": true
        }
      }
    }
  }'
```

#### PUT Request Example

```bash
# Update product price
curl -X PUT "https://example.com/rest/V1/products/new-product-001" \
  -H "Authorization: Bearer {admin_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "product": {
      "price": 39.99
    }
  }'
```

#### Search Criteria

```bash
# Search products with filters
curl -X GET "http://example.com/rest/V1/products?searchCriteria[filter_groups][0][filters][0][field]=price&searchCriteria[filter_groups][0][filters][0][value]=50&searchCriteria[filter_groups][0][filters][0][condition_type]=gt&searchCriteria[sort_orders][0][field]=price&searchCriteria[sort_orders][0][direction]=ASC&searchCriteria[pageSize]=10&searchCriteria[currentPage]=1" \
  -H "Authorization: Bearer {admin_token}"
```

### AuthorizationInterface

```php
// lib/internal/Magento/Framework/AuthorizationInterface.php
declare(strict_types=1);

namespace Magento\Framework;

/**
 * Interface for checking if context has access to a resource
 *
 * @api
 */
interface AuthorizationInterface
{
    /**
     * Check if context is allowed to access resource
     *
     * @param string $resource
     * @param int|null $privilege
     * @return bool
     */
    public function isAllowed(string $resource, ?int $privilege = null): bool;
}
```

## 6. API Authentication

### Token-Based Authentication (Admin/Customer)

```php
// Generate Admin Token
# POST /V1/integration/admin/token
# Request: {"username": "admin", "password": "admin123"}
# Response: "token_string"

declare(strict_types=1);

namespace Magento\Integration\Api;

/**
 * Admin token service interface
 */
interface AdminTokenServiceInterface
{
    /**
     * Create access token for admin user
     *
     * @param string $username
     * @param string $password
     * @return string Token
     * @throws AuthenticationException
     */
    public function createAdminAccessToken(string $username, string $password): string;

    /**
     * Revoke admin access token
     *
     * @param int $adminId
     * @return bool
     */
    public function revokeAdminAccessToken(int $adminId): bool;
}

/**
 * Customer token service interface
 */
interface CustomerTokenServiceInterface
{
    /**
     * Create access token for customer
     *
     * @param string $username
     * @param string $password
     * @return string Token
     * @throws AuthenticationException
     */
    public function createCustomerAccessToken(string $username, string $password): string;

    /**
     * Revoke customer access token
     *
     * @param int $customerId
     * @return bool
     */
    public function revokeCustomerAccessToken(int $customerId): bool;
}
```

#### Authentication Flow

```bash
# 1. Get admin token
curl -X POST "https://example.com/rest/V1/integration/admin/token" \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "admin123"}'

# Response: "ylxuhpthzxc5g3y69h1hbv71h6ybkj5q1v4wbgzb5h7x5y3"

# 2. Use token in requests
curl -X GET "https://example.com/rest/V1/products" \
  -H "Authorization: Bearer ylxuhpthzxc5g3y69h1hbv71h6ybkj5q1v4wbgzb5h7x5y3"

# 3. Get customer token
curl -X POST "https://example.com/rest/V1/integration/customer/token" \
  -H "Content-Type: application/json" \
  -d '{"username": "customer@example.com", "password": "password123"}'

# 4. Customer can access their own resources
curl -X GET "https://example.com/rest/V1/customers/me" \
  -H "Authorization: Bearer {customer_token}"
```

### OAuth 1.0a Authentication

OAuth 1.0a is used for third-party integrations:

```bash
# OAuth 1.0a flow requires:
# 1. Consumer key and secret from Integration registration
# 2. Request token endpoint
# 3. User authorization
# 4. Access token exchange

# Step 1: Get Request Token
curl -X POST "https://example.com/oauth/token/request" \
  -H "Content-Type: application/json" \
  -d '{
    "consumer_key": "your_consumer_key",
    "consumer_secret": "your_consumer_secret"
  }'

# Step 2: Authorize token (redirects user to admin)
# Step 3: Exchange for Access Token
curl -X POST "https://example.com/oauth/token/access" \
  -H "Content-Type: application/json" \
  -d '{
    "oauth_token": "request_token",
    "oauth_verifier": "verifier_code"
  }'

# Step 4: Use Access Token
curl -X GET "https://example.com/rest/V1/products" \
  -H "Authorization: OAuth oauth_consumer_key=xxx, oauth_token=xxx"
```

### Integration Tokens

```php
// Integration authentication for third-party apps
# POST /V1/integration/admin/token
# Request: {"username": "integration_user", "password": "integration_password"}

# Or OAuth for integrations:
# POST /V1/integration/access/token/request
# GET /V1/integration/access/token/authorize
# POST /V1/integration/access/token/access
```

### Authentication Class Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                   Authentication Chain                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │     Magento\Framework\Webapi\Authentication         │   │
│  │         (Main entry point)                           │   │
│  └──────────────────────┬───────────────────────────────┘   │
│                         │                                     │
│         ┌───────────────┼─────────────────┐                  │
│         ▼               ▼                 ▼                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐      │
│  │   Bearer    │  │   OAuth     │  │  Session/IP     │      │
│  │   Token     │  │   1.0a      │  │  Permission     │      │
│  └─────────────┘  └─────────────┘  └─────────────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 7. ACL & Permissions

### webapi.xml Resource Mapping

```xml
<!-- app/code/Magento/Catalog/etc/webapi.xml -->
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Webapi:etc/webapi.xsd">

    <route url="/V1/products" method="GET">
        <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="getList"/>
        <resources>
            <resource ref="Magento_Catalog::products"/>
        </resources>
    </route>

    <route url="/V1/products" method="POST">
        <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="save"/>
        <resources>
            <resource ref="Magento_Catalog::products"/>
        </resources>
    </route>
</routes>
```

### ACL Resources Hierarchy

```xml
<!-- etc/acl.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:etc/acl.xsd">
    <acl>
        <resources>
            <!-- Catalog Resources -->
            <resource id="Magento_Catalog::catalog" title="Catalog" sortOrder="10">
                <resource id="Magento_Catalog::products" title="Products" sortOrder="10">
                    <resource id="Magento_Catalog::products_read" title="Read Products" sortOrder="10"/>
                    <resource id="Magento_Catalog::products_write" title="Write Products" sortOrder="20"/>
                </resource>
                <resource id="Magento_Catalog::categories" title="Categories" sortOrder="20"/>
            </resource>

            <!-- Customer Resources -->
            <resource id="Magento_Customer::customer" title="Customers" sortOrder="20">
                <resource id="Magento_Customer::customer_read" title="Read Customers" sortOrder="10"/>
                <resource id="Magento_Customer::customer_write" title="Write Customers" sortOrder="20"/>
                <resource id="Magento_Customer::magento_customersegments" title="Segments" sortOrder="30"/>
            </resource>

            <!-- Sales Resources -->
            <resource id="Magento_Sales::sales" title="Sales" sortOrder="30">
                <resource id="Magento_Sales::orders" title="Orders" sortOrder="10">
                    <resource id="Magento_Sales::actions" title="Actions" sortOrder="10">
                        <resource id="Magento_Sales::create" title="Create" sortOrder="10"/>
                        <resource id="Magento_Sales::view" title="View" sortOrder="20"/>
                        <resource id="Magento_Sales::update" title="Update" sortOrder="30"/>
                        <resource id="Magento_Sales::cancel" title="Cancel" sortOrder="40"/>
                    </resource>
                </resource>
                <resource id="Magento_Sales::invoices" title="Invoices" sortOrder="20"/>
                <resource id="Magento_Sales::shipments" title="Shipments" sortOrder="30"/>
                <resource id="Magento_Sales::creditmemos" title="Credit Memos" sortOrder="40"/>
            </resource>

            <!-- Special Resources -->
            <resource id="self" title="Self" sortOrder="100"/>
            <resource id="anonymous" title="Anonymous" sortOrder="999"/>
        </resources>
    </acl>
</config>
```

### ACL Reader Implementation

```php
// app/code/Magento/Webapi/Model/Acl/Reader.php
declare(strict_types=1);

namespace Magento\Webapi\Model\Acl;

use Magento\Framework\Config\ReaderInterface;

/**
 * Reads ACL configuration for webapi routes
 */
class Reader implements ReaderInterface
{
    /**
     * @var \Magento\Framework\Acl\AclResource\Loader
     */
    private $aclResourceLoader;

    /**
     * @var \Magento\Framework\Acl\AclCache
     */
    private $aclCache;

    /**
     * @param \Magento\Framework\Acl\AclResource\Loader $aclResourceLoader
     * @param \Magento\Framework\Acl\AclCache $aclCache
     */
    public function __construct(
        \Magento\Framework\Acl\AclResource\Loader $aclResourceLoader,
        \Magento\Framework\Acl\AclCache $aclCache
    ) {
        $this->aclResourceLoader = $aclResourceLoader;
        $this->aclCache = $aclCache;
    }

    /**
     * Read ACL configuration for a route
     *
     * @param string $routePath
     * @return array
     */
    public function read(string $routePath): array
    {
        $cacheKey = 'webapi_acl_' . md5($routePath);
        $cachedData = $this->aclCache->load($cacheKey);

        if ($cachedData !== false) {
            return unserialize($cachedData);
        }

        $resources = $this->aclResourceLoader->load();
        $routeAcl = $this->buildRouteAcl($routePath, $resources);

        $this->aclCache->save(serialize($routeAcl), $cacheKey);

        return $routeAcl;
    }

    /**
     * Build ACL for specific route
     *
     * @param string $routePath
     * @param array $resources
     * @return array
     */
    private function buildRouteAcl(string $routePath, array $resources): array
    {
        // Implementation builds ACL rules for the route
        return [];
    }
}
```

### Default ACL Resources

| Resource | Description |
|----------|-------------|
| `anonymous` | Access without authentication |
| `self` | Access to user's own resources |
| `Magento_Catalog::products` | Full product access |
| `Magento_Catalog::products_read` | Read-only product access |
| `Magento_Customer::customer` | Full customer access |
| `Magento_Sales::orders` | Order access |
| `Magento_Sales::actions_view` | View orders |

## 8. API Rate Limiting

### How Rate Limiting Works in M2.4.8

**Important:** Rate limiting in M2.4.8 is handled **programmatically**, NOT via XML/XSD configuration files. There is no `induction.xml` file in Magento. Rate limiting is configured via `di.xml` dependency injection.

The rate limiting mechanism is implemented by `\Magento\Framework\Webapi\RateLimiter` class and its related interfaces.

### RateLimiterInterface

```php
// lib/internal/Magento/Framework/Webapi/RateLimiterInterface.php
declare(strict_types=1);

namespace Magento\Framework\Webapi;

/**
 * Interface for rate limiting web API requests
 */
interface RateLimiterInterface
{
    /**
     * Check if request is allowed
     *
     * @param string $identifier
     * @return bool
     */
    public function isAllowed(string $identifier): bool;

    /**
     * Add request to rate limit counter
     *
     * @param string $identifier
     * @return void
     */
    public function limit(string $identifier): void;

    /**
     * Get remaining requests
     *
     * @param string $identifier
     * @return int
     */
    public function getRemaining(string $identifier): int;

    /**
     * Get reset time
     *
     * @param string $identifier
     * @return int
     */
    public function getResetTime(string $identifier): int;
}
```

### FixedRateLimiter Implementation

```php
// lib/internal/Magento/Framework/Webapi/RateLimiter/FixedRateLimiter.php
declare(strict_types=1);

namespace Magento\Framework\Webapi\RateLimiter;

/**
 * Fixed-rate limiter implementation
 */
class FixedRateLimiter implements \Magento\Framework\Webapi\RateLimiterInterface
{
    /**
     * @var int
     */
    private $limit;

    /**
     * @var int
     */
    private $period;

    /**
     * @param int $limit
     * @param int $period
     */
    public function __construct(int $limit, int $period)
    {
        $this->limit = $limit;
        $this->period = $period;
    }

    /**
     * {@inheritdoc}
     */
    public function isAllowed(string $identifier): bool
    {
        return $this->getRemaining($identifier) > 0;
    }

    /**
     * {@inheritdoc}
     */
    public function limit(string $identifier): void
    {
        // Implementation increments counter
    }

    /**
     * {@inheritdoc}
     */
    public function getRemaining(string $identifier): int
    {
        // Returns remaining requests in current window
        return max(0, $this->limit - $this->getCurrentCount($identifier));
    }

    /**
     * Get current request count
     */
    private function getCurrentCount(string $identifier): int
    {
        // Implementation uses cache to track counts
        return 0;
    }
}
```

### Rate Limit Configuration via di.xml

Rate limiting is configured via `di.xml` (not via any XSD schema file):

```xml
<!-- app/code/{Vendor}/{Module}/etc/di.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    
    <!-- Default rate limiter for all API routes -->
    <type name="Magento\Framework\Webapi\RateLimiter">
        <arguments>
            <argument name="defaultLimiter" xsi:type="object">FixedRateLimiter</argument>
            <argument name="limit" xsi:type="number">100</argument>
            <argument name="period" xsi:type="number">1</argument>
        </arguments>
    </type>

    <!-- Custom rate limiter per route - configured in webapi.xml resources -->
    <type name="Magento\Customer\Model\RateLimiter">
        <arguments>
            <argument name="limit" xsi:type="number">100</argument>
            <argument name="period" xsi:type="number">60</argument>
        </arguments>
    </type>
</config>
```

### Per-Route Rate Limits in webapi.xml

Rate limits are configured per resource in `webapi.xml` using the `<rateLimiter>` element:

```xml
<!-- app/code/Magento/Customer/etc/webapi.xml -->
<?xml version="1.0"?>
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Webapi:etc/webapi.xsd">

    <route url="/V1/customers" method="POST">
        <service class="Magento\Customer\Api\AccountManagementInterface" method="createAccount"/>
        <resources>
            <resource ref="anonymous"/>
        </resources>
        <rateLimiter>
            <param name="limit">100</param>
            <param name="period">60</param>
        </rateLimiter>
    </route>
</routes>
```

### Rate Limiter Classes

| Class | Purpose |
|-------|---------|
| `\Magento\Framework\Webapi\RateLimiter` | Main rate limiter service |
| `\Magento\Framework\Webapi\RateLimiterInterface` | Interface for custom limiters |
| `\Magento\Framework\Webapi\RateLimiter\FixedRateLimiter` | Fixed-window rate limiter |
| `\Magento\Webapi\Model\RateLimiter\CustomerRateLimiter` | Per-customer rate limiting |
| `\Magento\Webapi\Model\RateLimiter\GuestRateLimiter` | Per-guest IP rate limiting |
| `\Magento\Webapi\Model\RateLimiter\AdminRateLimiter` | Per-admin rate limiting |

### Built-in Rate Limiters

Magento provides built-in rate limiters for different user types:

| User Type | Default Limit | Period |
|-----------|---------------|--------|
| Guest | 100 | 60 seconds |
| Customer | 100 | 60 seconds |
| Admin | 500 | 60 seconds |

### Default Rate Limits

| User Type | Default Limit | Period |
|-----------|---------------|--------|
| Guest | 20 requests | 60 seconds |
| Customer | 100 requests | 60 seconds |
| Admin | 500 requests | 60 seconds |
| Integration | 1000 requests | 60 seconds |

### GraphQL Rate Limiting (2.4.7+)

```php
// Magento\Framework\GraphQl\Query\ComplexityLimiter
// In 2.4.7+, GraphQL has query complexity and alias limits

// Query length validation (2.4.9+)
// Maximum query string length: 15,000 characters

// Alias limit validation (2.4.9+)
// Maximum aliases per query: 10
```

## 9. API Performance

### N+1 Problem in Resolvers

The N+1 problem occurs when resolving a list of items triggers individual queries for each item:

```php
// PROBLEMATIC: N+1 resolver
class BadProductResolver implements ResolverInterface
{
    public function resolve(Field $field, $context, ResolveInfo $info, array $value = null, array $args = null)
    {
        $products = $this->productRepository->getList($searchCriteria);

        foreach ($products as $product) {
            // Each iteration triggers a separate query!
            $categories[] = $this->categoryRepository->getByProductId($product->getId());
        }

        return $products;
    }
}
```

### DataLoader Batching Solution

```php
// SOLUTION: DataLoader pattern for batch loading
class GoodProductResolver implements ResolverInterface
{
    public function __construct(
        \Magento\Deferred\Model\Product $deferredProduct
    ) {
        $this->deferredProduct = $deferredProduct;
    }

    public function resolve(Field $field, $context, ResolveInfo $info, array $value = null, array $args = null)
    {
        $products = $this->productRepository->getList($searchCriteria);

        $skus = array_map(fn($p) => $p->getSku(), $products->getItems());

        // Single batch call loads all products
        $allCategories = $this->deferredCategory->loadBySkus($skus);

        // Assign categories to each product
        foreach ($products as $product) {
            $product->setCategories($allCategories[$product->getSku()] ?? []);
        }

        return $products;
    }
}
```

### BatchResolver Pattern

```php
// Using BatchResolverInterface for maximum performance
class CategoryBatchResolver implements BatchResolverInterface
{
    private $categoryLoader;

    public function resolveBatch(
        array $values,
        Field $field,
        $context,
        ResolveInfo $info,
        array $args = null
    ): array {
        // Collect all product IDs from the batch
        $productIds = [];
        foreach ($values as $value) {
            if (isset($value['product_id'])) {
                $productIds[] = $value['product_id'];
            }
        }

        // Single query for all categories
        $categories = $this->categoryRepository->getByProductIds($productIds);

        // Map results back to input order
        $results = [];
        foreach ($values as $value) {
            $productId = $value['product_id'] ?? null;
            $results[] = [
                'categories' => $categories[$productId] ?? []
            ];
        }

        return $results;
    }
}
```

### Caching Strategies

#### Field-Level Caching

```graphql
# Enable caching with @cache directive
type Query {
    products(filter: ProductAttributeFilterInput): Products @cache(cacheIdentity: "Magento\\CatalogGraphQl\\Model\\Resolver\\Products\\Identity")
}

# Identity class determines cache tags
class Identity implements CacheIdentityInterface
{
    public function getIdentities(array $resolvedData): array
    {
        $tags = [];

        foreach ($resolvedData['items'] ?? [] as $item) {
            $tags[] = 'product_' . $item['id'];
        }

        return array_merge($tags, ['products']);
    }
}
```

#### Cache Configuration

```xml
<!-- etc/cache.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Cache/etc/cache.xsd">
    <type name="graphql_product" translate="label" instance="Magento\CatalogGraphQl\Cache">
        <label>GraphQL Product Cache</label>
        <description>Cache for product GraphQL queries</description>
        <tags>graphql_product_tag</tags>
        <bin_stale>Magento\CatalogGraphQl\Cache\Backend\StaleUpdate</bin_stale>
    </type>
</config>
```

### Performance Best Practices

1. **Use Deferred/DataLoader Pattern**

```php
// Always batch load related data
class ProductResolver
{
    public function __construct(
        \Magento\Deferred\Model\Product $deferredProduct,
        \Magento\Deferred\Model\Category $deferredCategory,
        \Magento\Deferred\Model\Price $deferredPrice
    ) {
        $this->deferredProduct = $deferredProduct;
        $this->deferredCategory = $deferredCategory;
        $this->deferredPrice = $deferredPrice;
    }
}
```

2. **Implement BatchResolverInterface for Complex Fields**

```php
// For fields that require data from multiple sources
interface BatchServiceContractResolverInterface extends ResolverInterface
{
    public function executeBatch(array $resolvedData, array $args): array;
}
```

3. **Use Query Batching in Frontend**

```graphql
# Instead of multiple queries, use fragments to enable batching
query GetProducts($filter: ProductAttributeFilterInput) {
    products(filter: $filter) {
        items { ...ProductFields }
        aggregations { ...AggregationFields }
    }
}

fragment ProductFields on ProductInterface {
    id
    sku
    name
    price_range { ...PriceRangeFields }
}

fragment PriceRangeFields on PriceRange {
    minimum_price { ...PriceFields }
}

fragment PriceFields on Price {
    final_price { value currency }
    regular_price { value currency }
}
```

4. **Implement Proper Cache Invalidation**

```php
// Observer for cache invalidation
class CacheInvalidator
{
    public function afterSave(ProductInterface $product)
    {
        $this->cache->invalidate(['graphql_product_' . $product->getId()]);
        $this->cache->invalidate(['graphql_product_tag']);
    }
}
```

5. **Monitor Query Count**

```bash
# Enable query logging for development
bin/magento dev:query-log:enable

# Check logs
tail -f var/debug/db.log
```

### Performance Checklist

- [ ] Use `BatchResolverInterface` for list queries
- [ ] Implement DataLoader pattern for related entities
- [ ] Configure `@cache` directive for stable queries
- [ ] Use proper cache identity classes
- [ ] Avoid N+1 in resolver chains
- [ ] Implement request-scoped caching with `ResetAfterRequestInterface`
- [ ] Monitor query count with `bin/magento dev:query-log:enable`
- [ ] Use GraphQL persisted queries for production
- [ ] Implement proper cache invalidation on data changes

---

## Summary

Magento 2.4.8's web API framework provides a robust foundation for both GraphQL and REST APIs. Key takeaways:

**GraphQL:**
- Use `schema.graphqls` for type-safe schema definition
- Implement `ResolverInterface` for custom resolvers
- Prefer `BatchResolverInterface` for performance
- Use DataLoader pattern to prevent N+1 queries
- Configure `@cache` directive for frequently accessed data

**REST:**
- Define routes in `webapi.xml` with proper ACL resources
- Use service contracts for loose coupling
- Implement proper search criteria for list endpoints
- Configure rate limiting per user type

**Security:**
- Always use proper ACL resources in `webapi.xml`
- Prefer token-based auth over OAuth for simple integrations
- Implement rate limiting to prevent abuse
- Validate all input in resolvers

**Performance:**
- Batch load related entities using DataLoader pattern
- Use request-scoped caching with proper invalidation
- Monitor query counts during development
- Implement GraphQL persisted queries in production