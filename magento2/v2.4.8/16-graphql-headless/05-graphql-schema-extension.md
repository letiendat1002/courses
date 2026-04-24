---
title: "05 - GraphQL Schema Extension"
rank: 5
---

# 05 - GraphQL Schema Extension

## Introduction

Magento's GraphQL API is built on a schema-first architecture. The entire API surface—including queries, mutations, types, inputs, and enum values—is defined declaratively in `.gqls` files that are merged into a single introspection-capable schema. Extending this schema with custom types, queries, mutations, and resolvers is the foundation of headless Magento development.

This chapter covers how Magento's GraphQL schema is constructed, how to extend it with custom fields and types, how to implement resolvers with proper performance patterns (DataLoader), and how to write custom queries and mutations. We also cover the M2.4.6+ plugin-based schema extension mechanism and endpoint security.

---

## 1. How Magento's GraphQL Schema Works

### 1.1 Schema Definition Files

Magento 2 uses two mechanisms to define GraphQL schema:

**`etc/schema.graphqls`** — Module-level schema definitions. Each module can declare types, queries, mutations, inputs, interfaces, unions, enums, and scalar types in its `etc/schema.graphqls` file. This is the primary and most common approach.

**`*.gqls` fragment files** — Additional schema fragment files registered via `di.xml` using `Magento\GraphQl\Schema\Type\SchemaConfigurableFilesProvider`. These are merged into the schema at runtime. This approach is used for extensibility and schema stitching scenarios.

All `.gqls` files across all installed modules are discovered and merged into a single schema by `Magento\Framework\GraphQl\SchemaGenerator`.

### 1.2 Schema Introspection

Magento exposes GraphQL introspection at the `/graphql` endpoint. The full schema can be explored via the **`__schema`** query:

```graphql
query {
  __schema {
    queryType { name }
    mutationType { name }
    types {
      name
      kind
      fields {
        name
        type { name kind }
        args { name type { name } }
      }
    }
  }
}
```

The introspection result is a complete picture of every type, field, argument, and directive available in the current installation—including all third-party extensions.

### 1.3 GraphQL Playground

Magento's built-in GraphQL Playground is available at:

```
http://your-magento-base-url/graphql
```

The Playground provides:
- Interactive schema explorer (clickable docs)
- Query editor with syntax highlighting
- Variables panel
- Response viewer
- HTTP headers configuration (for authentication)

For production environments, the Playground can be disabled via:

```php
# app/etc/env.php
return [
    'system' => [
        'default' => [
            'graphql' => [
                'playground' => '0'
            ]
        ]
    ]
];
```

---

## 2. Extending the Schema with `.gqls` Files

### 2.1 The `extend type` Pattern

Schema extension in GraphQL follows the **additive only** principle. You cannot modify existing type definitions; you can only extend them by adding new fields. In Magento, this is done using the `extend` keyword in `.gqls` files.

### 2.2 Extending an Existing Type

To add a field to an existing Magento type (e.g., `Product`), create an `etc/schema.graphqls` file in your module:

```graphql
# Vendor/Module/etc/schema.graphqls

extend type Product {
  warrantyPeriod: String @doc(description: "Product warranty period in months")
  isEcoFriendly: Boolean @doc(description: "Whether product qualifies as eco-friendly")
}
```

This extends the `Product` type (already defined in `Magento_CatalogGraphQl`) with two new fields. The `@doc` directive adds documentation to the schema introspection.

To expose these fields, you must implement resolvers (see Section 3).

### 2.3 Adding New Types

Define entirely new types, queries, mutations, input types, and enums:

```graphql
# Vendor/Module/etc/schema.graphqls

type EcoImpactScore {
  carbonFootprint: Float!
  recyclingScore: Int!
  sustainabilityGrade: EcoGrade!
}

enum EcoGrade {
  A_PLUS
  A
  B
  C
  D
  F
}

input EcoScoreFilterInput {
  minCarbonFootprint: Float
  maxCarbonFootprint: Float
  grade: EcoGrade
}

type EcoScoreOutput {
  score: EcoImpactScore!
  productSku: String!
}
```

### 2.4 Registering Custom Schema Files via `di.xml`

When your module provides additional `.gqls` files beyond the standard `etc/schema.graphqls`, register them explicitly in `di.xml` so the schema generator includes them:

```xml
<!-- Vendor/Module/etc/di.xml -->

<type name="Magento\Framework\GraphQl\Schema\Type\SchemaConfigurableFilesProvider">
    <arguments>
        <argument name="files" xsi:type="array">
            <item name="Vendor_Module::custom_schema.graphqls</item>
            <item name="Vendor_Module::extensions.graphqls</item>
        </argument>
    </arguments>
</type>

<type name="Magento\Framework\GraphQl\Schema\Type\SchemaGenerator">
    <plugin name="Vendor_Module::SchemaExtensionPlugin" type="Vendor\Module\Plugin\SchemaExtensionPlugin"/>
</type>
```

The `SchemaConfigurableFilesProvider` aggregates all registered `.gqls` files. Without this registration, additional `.gqls` files may not be included in the merged schema unless they follow Magento's auto-discovery convention.

### 2.5 Complete Example: Extending the Product Type

**Step 1 — Declare the schema extension:**

```graphql
# app/code/Vendor/EcoProduct/etc/schema.graphqls

extend type Product {
  ecoScore: EcoScoreOutput @doc(description: "Environmental impact score")
}
```

**Step 2 — Implement the resolver:**

```php
// app/code/Vendor/EcoProduct/Plugin/Resolver/EcoScoreResolver.php

<?php
declare(strict_types=1);

namespace Vendor\EcoProduct\Plugin\Resolver;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Vendor\EcoProduct\Model\EcoScore\DataLoader as EcoScoreDataLoader;

class EcoScoreResolver implements ResolverInterface
{
    public function __construct(
        private readonly EcoScoreDataLoader $ecoScoreDataLoader
    ) {}

    public function resolve(
        $fieldValue,
        array $context = [],
        ResolveInfo $info = [],
        array $values = []
    ): ?array {
        if (!isset($values['sku'])) {
            return null;
        }

        return $this->ecoScoreDataLoader->load($values['sku']);
    }
}
```

**Step 3 — Register in `di.xml`:**

```xml
<!-- app/code/Vendor/EcoProduct/etc/di.xml -->

<type name="Magento\CatalogGraphQl\Model\Resolver\Product">
    <plugin name="EcoProduct_ecoScore" type="Vendor\EcoProduct\Plugin\Resolver\EcoScoreResolver"/>
</type>
```

The plugin pattern on the existing `Product` resolver chains our custom resolver into Magento's resolver pipeline.

---

## 3. Custom Resolvers

### 3.1 ResolverInterface

The fundamental resolver contract in Magento is:

```php
interface ResolverInterface
{
    /**
     * @param mixed $fieldValue
     * @param array $context
     * @param ResolveInfo $info
     * @param array $values The resolved values of the parent type fields
     * @return mixed
     */
    public function resolve(
        $fieldValue,
        array $context = [],
        ResolveInfo $info = [],
        array $values = []
    );
}
```

- **`$fieldValue`** — The resolved value of the parent field (e.g., the Product array).
- **`$context`** — The GraphQL context object containing request information, store, customer session, etc.
- **`$info`** — The `ResolveInfo` object containing the current field AST, schema, variable values, and more.
- **`$values`** — The resolved values of the parent object's fields. This is the primary source of data from the parent entity (e.g., product SKU, ID).

### 3.2 DataResolverInterface

For mutations and write operations, Magento provides:

```php
interface DataResolverInterface extends ResolverInterface
{
}
```

`DataResolverInterface` extends `ResolverInterface` with no additional methods—it serves as a marker interface indicating the resolver performs write operations (creates, updates, deletes). All mutation resolvers should implement `DataResolverInterface`.

### 3.3 The Resolver Chain

Magento's GraphQL resolution pipeline follows this sequence:

1. **SchemaReader** — Reads the `.gqls` files and builds the schema AST.
2. **Resolver** — Matches fields to resolver instances and invokes `resolve()`.
3. **DataLoader** — Batches and caches entity loads within a single query execution to avoid N+1 problems.

When a field is resolved, Magento:
1. Looks up the resolver for the type
2. Calls `resolve()` with the parent field's resolved values
3. Returns the result to the GraphQL executor

### 3.4 Resolving Fields with Parent Values

A resolver's primary job is to take the already-resolved parent entity data and return a value for the requested field. The parent data is passed via `$values`:

```php
// Example: resolving a field on the Product type
// $values contains the resolved Product array, e.g.:
// ['sku' => 'WH01-XS-Red', 'name' => 'Hero Watch', 'price' => 54.00, ...]

public function resolve(
    $fieldValue,
    array $context = [],
    ResolveInfo $info = [],
    array $values = []
): ?array {
    $sku = $values['sku'] ?? null;
    if (!$sku) {
        return null;
    }

    return $this->ecoScoreDataLoader->load($sku);
}
```

---

## 4. The DataLoader Pattern (Critical for Performance)

### 4.1 The N+1 Query Problem

Without DataLoader, a GraphQL query fetching multiple products each requesting the same related data would trigger separate database calls per product:

```graphql
query {
  products(filter: { page_size: 10 }) {
    items {
      ecoScore { carbonFootprint }
      warrantyPeriod
    }
  }
}
```

Without batching: 10 products × 1 eco score lookup = 10 separate database queries. With 100 products in a list, this becomes 100 queries.

### 4.2 DataLoader Class

Magento provides `Magento\Framework\DataObject\DataLoader` as the base DataLoader class implementing `Magento\Framework\DataObject\DataLoaderInterface`. The DataLoader pattern solves this through:

1. **Batching** — Collect all load requests during resolver execution
2. **Caching** — Return cached results for duplicate keys within the same query
3. **Deferred Execution** — Defer the actual database call until the data is actually needed (after all batched requests are collected)

```php
// Magento\Framework\DataObject\DataLoaderInterface
interface DataLoaderInterface
{
    public function load($identifier);
    public function flush(): void;
}
```

### 4.3 DeferredValue

The `Magento\Framework\DataObject\DeferredValue` class is the mechanism that enables deferred loading:

```php
class DeferredValue
{
    public function __construct(callable $loader, mixed $identifier)
    // The loader callable is not invoked until getValue() is called
}
```

When a resolver calls `$dataLoader->load($sku)`, it doesn't execute the query immediately. Instead, it returns a `DeferredValue` wrapper. The actual load is deferred until the GraphQL executor accesses the resolved value.

### 4.4 Magento's Built-in DataLoaders

Magento Core provides several DataLoaders you can inject and use:

| DataLoader | Purpose |
|---|---|
| `ProductDataLoader` | Loads full product data by SKU/ID |
| `CategoryDataLoader` | Loads category data by ID |
| `CustomerDataLoader` | Loads customer data by ID |
| `OrderDataLoader` | Loads order data by number |
| `PriceDataLoader` | Loads price data for products |

Usage example:

```php
use Magento\CatalogGraphQl\DataProvider\Product\Search\DataLoader as ProductSearchDataLoader;

class MyResolver implements ResolverInterface
{
    public function __construct(
        private readonly ProductSearchDataLoader $productSearchDataLoader
    ) {}

    public function resolve($fieldValue, array $context, ResolveInfo $info, array $values)
    {
        // Use the DataLoader — requests are batched automatically
        return $this->productSearchDataLoader->load($values['entity_id']);
    }
}
```

### 4.5 Implementing a Custom DataLoader

Here's a complete custom DataLoader implementation for the eco-score example:

**Step 1 — Define the DataLoader class:**

```php
// app/code/Vendor/EcoProduct/Model/EcoScore/DataLoader.php

<?php
declare(strict_types=1);

namespace Vendor\EcoProduct\Model\EcoScore;

use Magento\Framework\DataObject\DataLoaderInterface;
use Magento\Framework\DataObject\DeferredValue;
use Vendor\EcoProduct\Model\EcoScoreService;

class DataLoader implements DataLoaderInterface
{
    /**
     * @param EcoScoreService $ecoScoreService
     */
    public function __construct(
        private readonly EcoScoreService $ecoScoreService
    ) {}

    /**
     * @inheritdoc
     */
    public function load($identifier): DeferredValue
    {
        return new DeferredValue(
            function () use ($identifier) {
                return $this->ecoScoreService->getScoreBySku($identifier);
            },
            $identifier
        );
    }

    /**
     * Clear any cached data. Called after batch resolution completes.
     */
    public function flush(): void
    {
        // Reset internal caches if any were used
    }
}
```

**Step 2 — Define the service that performs the actual data fetching:**

```php
// app/code/Vendor/EcoProduct/Model/EcoScore/EcoScoreService.php

<?php
declare(strict_types=1);

namespace Vendor\EcoProduct\Model\EcoScore;

use Vendor\EcoProduct\Api\EcoScoreRepositoryInterface;

class EcoScoreService
{
    public function __construct(
        private readonly EcoScoreRepositoryInterface $ecoScoreRepository
    ) {}

    public function getScoreBySku(string $sku): array
    {
        $score = $this->ecoScoreRepository->getBySku($sku);

        return [
            'carbonFootprint' => $score->getCarbonFootprint(),
            'recyclingScore' => $score->getRecyclingScore(),
            'sustainabilityGrade' => $score->getGrade(),
        ];
    }
}
```

**Step 3 — Register the DataLoader in `di.xml`:**

```xml
<!-- app/code/Vendor/EcoProduct/etc/di.xml -->

<type name="Vendor\EcoProduct\Model\EcoScore\DataLoader">
    <arguments>
        <argument name="ecoScoreService" xsi:type="object">Vendor\EcoProduct\Model\EcoScore\EcoScoreService</argument>
    </arguments>
</type>
```

### 4.6 When to Use DataLoader vs. Direct Service

| Scenario | Approach |
|---|---|
| Field on list entity (product, category, customer) | Use DataLoader to batch |
| Single-entity query (one specific product by ID) | Direct service is acceptable |
| Mutation return types | Direct service (no batch needed) |
| Nested resolved values in list queries | Always DataLoader |

The DataLoader is not needed for single-entity lookups or mutations, but any field on a list-type query that requires a database call **must** use DataLoader to prevent N+1 performance degradation.

---

## 5. Custom Queries

### 5.1 Schema Definition

Define your query in `etc/schema.graphqls`:

```graphql
type Query {
  ecoScore(productSku: String!): EcoScoreOutput
    @doc(description: "Get environmental impact score for a product by SKU")
    @resolver(class: "Vendor\\EcoProduct\\Model\\Resolver\\EcoScoreQuery")

  ecoScores(filter: EcoScoreFilterInput, pageSize: Int = 10, currentPage: Int = 1): EcoScoreOutputCollection
    @doc(description: "List eco scores with optional filtering")
    @resolver(class: "Vendor\\EcoProduct\\Model\\Resolver\\EcoScoreListQuery")
}
```

The `@resolver` directive tells Magento which resolver class handles this field.

### 5.2 Implementing the Resolver

```php
// app/code/Vendor/EcoProduct/Model/Resolver/EcoScoreQuery.php

<?php
declare(strict_types=1);

namespace Vendor\EcoProduct\Model\Resolver;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Vendor\EcoProduct\Model\EcoScore\DataLoader;

class EcoScoreQuery implements ResolverInterface
{
    public function __construct(
        private readonly DataLoader $ecoScoreDataLoader
    ) {}

    public function resolve(
        $fieldValue,
        array $context = [],
        ResolveInfo $info = [],
        array $values = []
    ): array {
        $sku = $info->args['productSku'] ?? null;

        if (!$sku) {
            throw new \Magento\Framework\GraphQl\Exception\GraphQlInputException(
                __('Product SKU is required')
            );
        }

        // Use DataLoader for consistent batching across query contexts
        $deferredValue = $this->ecoScoreDataLoader->load($sku);

        return [
            'sku' => $sku,
            'score' => $deferredValue,
        ];
    }
}
```

### 5.3 List Query with Pagination

```graphql
# etc/schema.graphqls
type EcoScoreOutputCollection {
  total_count: Int! @doc(description: "Total number of results")
  items: [EcoScoreOutput!]! @doc(description: "List of eco score items")
}
```

```php
// app/code/Vendor/EcoProduct/Model/Resolver/EcoScoreListQuery.php

<?php
declare(strict_types=1);

namespace Vendor\EcoProduct\Model\Resolver;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Vendor\EcoProduct\Api\EcoScoreRepositoryInterface;
use Vendor\EcoProduct\Model\EcoScore\DataLoader;

class EcoScoreListQuery implements ResolverInterface
{
    public function __construct(
        private readonly EcoScoreRepositoryInterface $ecoScoreRepository,
        private readonly DataLoader $ecoScoreDataLoader
    ) {}

    public function resolve(
        $fieldValue,
        array $context = [],
        ResolveInfo $info = [],
        array $values = []
    ): array {
        $filter = $info->args['filter'] ?? [];
        $pageSize = $info->args['pageSize'] ?? 10;
        $currentPage = $info->args['currentPage'] ?? 1;

        $collection = $this->ecoScoreRepository->getList($filter, $pageSize, $currentPage);

        $items = [];
        foreach ($collection->getItems() as $score) {
            $deferredValue = $this->ecoScoreDataLoader->load($score->getSku());
            $items[] = [
                'sku' => $score->getSku(),
                'score' => $deferredValue,
            ];
        }

        return [
            'total_count' => $collection->getTotalCount(),
            'items' => $items,
        ];
    }
}
```

### 5.4 Registering in `di.xml`

```xml
<!-- app/code/Vendor/EcoProduct/etc/di.xml -->

<!-- Queries are automatically resolved via @resolver directive, but you can
     also explicitly register type resolvers for complex scenarios: -->

<type name="Magento\Framework\GraphQl\Query\Resolver\TypeFactory">
    <plugin name="EcoProduct_ecoScore" type="Vendor\EcoProduct\Plugin\TypeResolverPlugin"/>
</type>
```

Queries with `@resolver(class: "...")` directive are auto-resolved by Magento's `SchemaReader` at runtime. No additional `di.xml` entry is needed for simple query resolvers.

---

## 6. Custom Mutations

### 6.1 Schema Definition

Mutations follow the same `.gqls` pattern as queries. Define the input type, output type, and the mutation type:

```graphql
input EcoScoreInput {
  carbonFootprint: Float!
  recyclingScore: Int!
  sustainabilityGrade: EcoGrade!
}

input EcoScoreProductInput {
  productSku: String!
  ecoScore: EcoScoreInput!
}

type EcoScoreMutationOutput {
  success: Boolean!
  message: String
  ecoScore: EcoScoreOutput
}

type Mutation {
  submitEcoScore(input: EcoScoreProductInput!): EcoScoreMutationOutput
    @doc(description: "Submit eco score data for a product")
    @resolver(class: "Vendor\\EcoProduct\\Model\\Resolver\\SubmitEcoScoreMutation")
}
```

### 6.2 Implementing the Mutation Resolver

```php
// app/code/Vendor/EcoProduct/Model/Resolver/SubmitEcoScoreMutation.php

<?php
declare(strict_types=1);

namespace Vendor\EcoProduct\Model\Resolver;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Framework\GraphQl\Exception\GraphQlInputException;
use Magento\Framework\GraphQl\Exception\GraphQlAuthorizationException;
use Vendor\EcoProduct\Api\EcoScoreRepositoryInterface;
use Vendor\EcoProduct\Model\EcoScore\Validation\InputValidator;

class SubmitEcoScoreMutation implements ResolverInterface
{
    public function __construct(
        private readonly EcoScoreRepositoryInterface $ecoScoreRepository,
        private readonly InputValidator $inputValidator
    ) {}

    public function resolve(
        $fieldValue,
        array $context = [],
        ResolveInfo $info = [],
        array $values = []
    ): array {
        // Authorization check — only admin or integration can submit
        if (!$context->getExtensionAttributes()->getIsAuthorized()) {
            throw new GraphQlAuthorizationException(
                __('Unauthorized to submit eco scores')
            );
        }

        $input = $info->args['input'] ?? [];
        $this->validateInput($input);

        try {
            $sku = $input['productSku'];
            $ecoScoreData = $input['ecoScore'];

            $entity = $this->ecoScoreRepository->createEntity([
                'sku' => $sku,
                'carbon_footprint' => $ecoScoreData['carbonFootprint'],
                'recycling_score' => $ecoScoreData['recyclingScore'],
                'grade' => $ecoScoreData['sustainabilityGrade'],
            ]);

            $this->ecoScoreRepository->save($entity);

            return [
                'success' => true,
                'message' => __('Eco score saved successfully')->render(),
                'ecoScore' => [
                    'carbonFootprint' => $entity->getCarbonFootprint(),
                    'recyclingScore' => $entity->getRecyclingScore(),
                    'sustainabilityGrade' => $entity->getGrade(),
                ],
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'message' => $e->getMessage(),
                'ecoScore' => null,
            ];
        }
    }

    private function validateInput(array $input): void
    {
        if (empty($input['productSku'])) {
            throw new GraphQlInputException(__('Product SKU is required'));
        }

        if (!isset($input['ecoScore']['carbonFootprint']) || $input['ecoScore']['carbonFootprint'] < 0) {
            throw new GraphQlInputException(
                __('Carbon footprint must be a non-negative number')
            );
        }

        if (!isset($input['ecoScore']['recyclingScore']) ||
            $input['ecoScore']['recyclingScore'] < 0 ||
            $input['ecoScore']['recyclingScore'] > 100) {
            throw new GraphQlInputException(
                __('Recycling score must be between 0 and 100')
            );
        }
    }
}
```

### 6.3 Input Type Validation with Zod

For complex input validation, use a validation layer:

```php
// app/code/Vendor/EcoProduct/Model/EcoScore/Validation/InputValidator.php

<?php
declare(strict_types=1);

namespace Vendor\EcoProduct\Model\EcoScore\Validation;

use Magento\Framework\GraphQl\Exception\GraphQlInputException;
use Vendor\EcoProduct\Api\Data\EcoScoreInterface;

class InputValidator
{
    private const MIN_RECYCLING_SCORE = 0;
    private const MAX_RECYCLING_SCORE = 100;

    public function validate(array $input): void
    {
        if (empty($input['productSku'])) {
            throw new GraphQlInputException(__('Product SKU is required'));
        }

        if (!isset($input['ecoScore']['carbonFootprint'])) {
            throw new GraphQlInputException(
                __('Carbon footprint is required')
            );
        }

        if (!is_numeric($input['ecoScore']['carbonFootprint']) || $input['ecoScore']['carbonFootprint'] < 0) {
            throw new GraphQlInputException(
                __('Carbon footprint must be a non-negative number')
            );
        }

        $recyclingScore = $input['ecoScore']['recyclingScore'] ?? null;
        if (!is_int($recyclingScore) ||
            $recyclingScore < self::MIN_RECYCLING_SCORE ||
            $recyclingScore > self::MAX_RECYCLING_SCORE) {
            throw new GraphQlInputException(
                __('Recycling score must be an integer between %1 and %2',
                    self::MIN_RECYCLING_SCORE,
                    self::MAX_RECYCLING_SCORE
                )
            );
        }

        $validGrades = [EcoScoreInterface::GRADE_A_PLUS, EcoScoreInterface::GRADE_A,
                        EcoScoreInterface::GRADE_B, EcoScoreInterface::GRADE_C,
                        EcoScoreInterface::GRADE_D, EcoScoreInterface::GRADE_F];
        $grade = $input['ecoScore']['sustainabilityGrade'] ?? null;
        if (!in_array($grade, $validGrades, true)) {
            throw new GraphQlInputException(
                __('Sustainability grade must be one of: %1', implode(', ', $validGrades))
            );
        }
    }
}
```

### 6.4 Mutation Return Type Design

A well-designed mutation return type follows this pattern:

```graphql
type EcoScoreMutationOutput {
  success: Boolean!
  message: String
  ecoScore: EcoScoreOutput  # null on failure, populated on success
}
```

This pattern enables the client to:
- Check `success` boolean for flow control
- Read `message` for user-facing error text
- Access `ecoScore` when success is true
- Handle `null` gracefully when success is false

---

## 7. Schema Extension via Plugins (M2.4.6+)

### 7.1 The Plugin Mechanism

Starting with **Magento 2.4.6**, you can extend the GraphQL schema via plugins on `Magento\Framework\GraphQl\Schema\Type\SchemaGeneratorInterface` without modifying the original `.gqls` files. This enables third-party modules to add fields to types defined by other modules.

### 7.2 Plugin Interface

```php
// Magento\Framework\GraphQl\Schema\Type\SchemaGeneratorInterface
interface SchemaGeneratorInterface
{
    public function generate(): Document;
}
```

Plugin on this interface to modify the schema AST before it's finalized.

### 7.3 Implementing a Schema Extension Plugin

```php
// app/code/Vendor/EcoProduct/Plugin/SchemaExtensionPlugin.php

<?php
declare(strict_types=1);

namespace Vendor\EcoProduct\Plugin;

use Magento\Framework\GraphQl\Schema\Type\SchemaGeneratorInterface;
use Magento\Framework\GraphQl\Schema\Type\Extension\Processor\FieldExtensionProcessor;
use Magento\Framework\ObjectManager\InterceptableInterface;

class SchemaExtensionPlugin
{
    /**
     * Add ecoScore field to the Product type via plugin.
     * This approach modifies the schema AST without touching the original .gqls files.
     */
    public function beforeGenerate(
        SchemaGeneratorInterface $subject
    ): void {
        // The plugin can inject additional type extensions into the schema
        // before the final schema document is generated.
        // Use FieldExtensionProcessor to add fields to existing types.
    }
}
```

### 7.4 FieldExtensionProcessor for Adding Fields

```php
// Using FieldExtensionProcessor to add fields to an existing type

<?php
declare(strict_types=1);

namespace Vendor\EcoProduct\Plugin;

use Magento\Framework\GraphQl\Schema\Type\SchemaGeneratorInterface;
use Magento\Framework\GraphQl\Schema\Type\Extension\Processor\FieldExtensionProcessor;
use Magento\Catalog\Api\Data\ProductInterface;

class SchemaExtensionPlugin
{
    public function __construct(
        private readonly FieldExtensionProcessor $fieldExtensionProcessor
    ) {}

    public function beforeGenerate(SchemaGeneratorInterface $subject): void
    {
        $this->fieldExtensionProcessor->addFieldToType(
            'Product',  // target type
            [
                'name' => 'ecoScore',
                'type' => 'EcoScoreOutput',
                'resolver' => 'Vendor\\EcoProduct\\Model\\Resolver\\EcoScoreResolver',
                'description' => 'Environmental impact score',
            ]
        );
    }
}
```

### 7.5 Use Case: Adding Fields from a Third-Party Module

This plugin pattern is particularly valuable for modules that need to extend the `Product` type defined by `Magento_CatalogGraphQl` without requiring changes to the core catalog module:

```
Magento_CatalogGraphQl (core)     → defines "type Product { ... }"
Vendor_EcoProduct (third-party)    → Plugin on SchemaGenerator adds "ecoScore" field
```

The third-party module adds its field to the existing type cleanly, without modifying the original module's code. This is the recommended approach for M2.4.6+.

### 7.6 Registering the Schema Plugin in `di.xml`

```xml
<!-- app/code/Vendor/EcoProduct/etc/di.xml -->

<type name="Magento\Framework\GraphQl\Schema\Type\SchemaGeneratorInterface">
    <plugin name="EcoProduct_SchemaExtension" type="Vendor\EcoProduct\Plugin\SchemaExtensionPlugin"/>
</type>
```

---

## 8. Securing GraphQL Endpoints

### 8.1 Authentication Methods

Magento GraphQL supports three authentication mechanisms:

**Customer Token (B2C):**

```graphql
mutation {
  generateCustomerToken(email: "customer@example.com", password: "password123") {
    token
  }
}
```

```graphql
# Use the token in the Authorization header:
# Authorization: Bearer <token>
```

**Admin Token (B2B / Admin Operations):**

```graphql
mutation {
  generateAdminToken(email: "admin@example.com", password: "adminpassword") {
    token
  }
}
```

**Integration Token (System-to-System):**

Tokens are generated when creating an integration in **System → Integrations**.

### 8.2 Authorization with `webapi.xml`

GraphQL endpoints still use `webapi.xml` for ACL-based authorization. For a mutation:

```xml
<!-- app/code/Vendor/EcoProduct/etc/webapi.xml -->

<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Webapi:etc/webapi.xsd">
    <route url="/V1/eco-product/submit" method="POST">
        <service class="Vendor\EcoProduct\Api\EcoScoreManagementInterface" method="submit"/>
        <resources>
            <resource ref="Vendor_EcoProduct::eco_score_submit"/>
        </resources>
    </route>
</routes>
```

In `acl.xml`:

```xml
<!-- app/code/Vendor/EcoProduct/etc/acl.xml -->

<acl>
    <resources>
        <resource id="Magento_Backend::admin">
            <resource id="Vendor_EcoProduct::eco_score_submit"
                      title="Submit Eco Score"
                      sortOrder="10"/>
        </resource>
    </resources>
</acl>
```

Authorization in GraphQL resolvers:

```php
public function resolve($fieldValue, array $context, ResolveInfo $info, array $values): array
{
    $customerId = $context->getUserId();

    if (!$this->aclChecker->isAllowed('Vendor_EcoProduct::eco_score_submit', $customerId)) {
        throw new GraphQlAuthorizationException(
            __('Authorization required to submit eco scores')
        );
    }

    // proceed with mutation logic
}
```

### 8.3 Rate Limiting

GraphQL rate limiting is configured in `env.php`:

```php
// app/etc/env.php
return [
    'system' => [
        'default' => [
            'graphql' => [
                'rate_limiting' => [
                    'enabled' => true,
                    'max_complexity' => 1000,      # Maximum query complexity score
                    'max_depth' => 20,             # Maximum nesting depth
                    'max_aliases' => 100,           # Maximum field aliases per query
                ]
            ]
        ]
    ]
];
```

Rate limiting can also be applied per customer group:

```php
'graphql' => [
    'rate_limiting' => [
        'enabled' => true,
        'max_complexity' => 1000,
        'customer_group_limits' => [
            'NOT LOGGED IN' => ['max_complexity' => 200],
            'General' => ['max_complexity' => 500],
            'Wholesale' => ['max_complexity' => 2000],
        ]
    ]
]
```

### 8.4 Disabling Introspection in Production

For production environments, disable schema introspection to prevent leaking schema details:

```php
# app/etc/env.php
return [
    'system' => [
        'default' => [
            'graphql' => [
                'introspection' => '0'
            ]
        ]
    ]
];
```

This disables the `__schema` and `__type` introspection queries. Clients can still execute queries if they already know the schema structure, but cannot discover it programmatically.

---

## Summary

Magento's GraphQL schema is a merged artifact of all `*.gqls` files across installed modules. Extending it involves:

1. **`.gqls` files** — Use `extend type` to add fields to existing types or define entirely new types, queries, and mutations.
2. **Resolvers** — Implement `ResolverInterface` for queries and `DataResolverInterface` for mutations. The resolver receives parent field values via `$values` and returns the field's resolved value.
3. **DataLoader pattern** — Critical for any field accessed in list contexts. It batches duplicate entity loads within a single query execution, eliminating N+1 queries. Use `DeferredValue` to defer the actual data fetch until the value is accessed.
4. **Custom queries** — Declare in `.gqls` with `@resolver(class: "...")` directive, implement the resolver with DataLoader support, and register via `di.xml`.
5. **Custom mutations** — Declare input types, output types, and the mutation. Implement with input validation and proper error handling using `GraphQlInputException`.
6. **Schema extension plugins (M2.4.6+)** — Plugin on `SchemaGeneratorInterface` to inject fields into existing types without modifying original schema files.
7. **Security** — Authenticate via customer/admin/integration tokens; authorize via ACL resources in `webapi.xml`; configure rate limiting in `env.php`.

---

## See Also

- **REST/GraphQL API reference:** `../07-api-development/11-graphql-rest-api.md`
- Magento 2.4.8 `Magento/Framework/GraphQl` component documentation
- `Magento/Framework/DataObject/DataLoader` — DataLoader base implementation
- `Magento/Framework/GraphQl/Schema/Type/SchemaConfigurableFilesProvider` — schema file registration