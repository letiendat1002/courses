---
title: "28 - Elasticsearch & Search"
description: "Deep dive into Magento 2.4.8 Elasticsearch integration: search architecture, Elasticsearch configuration, custom search providers, and catalog search optimization."
tags: magento2, elasticsearch, search, catalog-search, es-configuration, search-adapters
rank: 17
pathways: [magento2-deep-dive]
---

# Elasticsearch & Search

## 1. Search Architecture Overview

Magento 2.4.8 implements a **layered search architecture** that decouples the search interface from the underlying search engine implementation. This abstraction allows different search providers (Elasticsearch 7, OpenSearch, MySQL LIKE) to be swapped without changing application code.

### Search Architecture Components

```
┌─────────────────────────────────────────────────────────────┐
│                     Search UI Layer                         │
│  (Query Input → SearchController → SearchModel)             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  Search Engine Interface                     │
│          Magento\Search\Model\SearchEngine                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Search Adapter Pool (DI)                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │Elasticsearch7│  │  OpenSearch   │  │  MySQL Like  │      │
│  │   Adapter    │  │   Adapter    │  │   Adapter    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  Search Engine                               │
│        (Elasticsearch 7.x / OpenSearch 1.x)                 │
└─────────────────────────────────────────────────────────────┘
```

### SearchProviders Interface

The core interface is `Magento\Search\Api\SearchProviderInterface`:

```php
// Magento\Search\Api\SearchProviderInterface
declare(strict_types=1);

namespace Magento\Search\Api;

use Magento\Framework\Api\Search\DocumentInterface;

interface SearchProviderInterface
{
    /**
     * Execute search and return documents
     *
     * @param \Magento\Framework\Search\Request $request
     * @return \Magento\Framework\Api\Search\SearchResultInterface
     */
    public function search(\Magento\Framework\Search\Request $request): \Magento\Framework\Api\Search\SearchResultInterface;
}
```

### SearchEngine Model

The `Magento\Search\Model\SearchEngine` acts as a façade, delegating to the configured adapter:

```php
// Magento\Search\Model\SearchEngine
declare(strict_types=1);

namespace Magento\Search\Model;

use Magento\Framework\Search\SearchEngineInterface;

class SearchEngine implements SearchEngineInterface
{
    public function __construct(
        \Magento\Framework\Search\SearchEngineProvider $searchEngineProvider,
        \Magento\Framework\Search\AdapterPool $adapterPool
    ) {}

    public function search(\Magento\Framework\Search\Request $request): \Magento\Framework\Api\Search\SearchResultInterface
    {
        $adapter = $this->adapterPool->getAdapter();
        return $adapter->query($request);
    }
}
```

### Search Response Aggregation

Search results are aggregated through `\Magento\Framework\Api\Search\SearchResult`:

```php
// Magento\Framework\Api\Search\SearchResult
class SearchResult implements \Magento\Framework\Api\Search\SearchResultInterface
{
    public function getItems(): array
    {
        // Returns array of \Magento\Framework\Api\Search\DocumentInterface
    }

    public function getAggregations(): \Magento\Framework\Api\Search\AggregationInterface
    {
        // Returns bucket aggregations (categories, price ranges, etc.)
    }

    public function getTotal(): int
    {
        // Total matching documents
    }

    public function getSearchRequest(): \Magento\Framework\Search\Request
    {
        // Original search request
    }
}
```

---

## 2. Elasticsearch 7 Integration

Magento 2.4.8 requires **Elasticsearch 7.x** as the default search engine. Elasticsearch provides full-text search, faceted filtering, and real-time indexing capabilities essential for catalog search performance.

### Version Requirements

| Component | Version | Notes |
|-----------|---------|-------|
| Elasticsearch | 7.9+ | 7.9, 7.11, 7.17 supported |
| OpenSearch | 1.x | Alternative engine (Adobe Commerce only) |
| PHP Extension | elasticsearch-php 7.0+ | Client library |

### Elasticsearch7 Adapter

The `Elasticsearch7` adapter is located at `Magento\Elasticsearch7\SearchAdapter\Adapter`:

```php
// Magento\Elasticsearch7\SearchAdapter\Adapter
declare(strict_types=1);

namespace Magento\Elasticsearch7\SearchAdapter;

use Magento\Framework\Search\SearchAdapterInterface;
use Magento\Elasticsearch7\SearchAdapter\Mapper;
use Magento\Elasticsearch7\SearchAdapter\ResponseFactory;

class Adapter implements \Magento\Framework\Search\SearchAdapterInterface
{
    public function __construct(
        Mapper $mapper,
        ConnectionManager $connectionManager,
        ResponseFactory $responseFactory,
        \Magento\Elasticsearch7\SearchAdapter\Proxy $proxy
    ) {}

    public function query(\Magento\Framework\Search\Request $request): \Magento\Framework\Api\Search\SearchResultInterface
    {
        // 1. Map request to Elasticsearch query
        $esQuery = $this->mapper->buildQuery($request);

        // 2. Execute against Elasticsearch
        $response = $this->connectionManager->connect()->search($esQuery);

        // 3. Transform response to Magento format
        return $this->responseFactory->create($response, $request);
    }
}
```

### Connection Configuration

Elasticsearch connection is configured in `app/etc/env.php`:

```php
// app/etc/env.php
return [
    // ...
    'elasticsearch' => [
        'hosts' => [
            'localhost:9200',
            // Or cluster nodes:
            // 'es-node-1.example.com:9200',
            // 'es-node-2.example.com:9200',
        ],
        'username' => 'elastic',
        'password' => 'your_secure_password',
        // Optional: SSL configuration
        // 'ssl' => true,
        // 'certificate' => '/path/to/certificate.crt',
    ],
    // Override default index settings
    'catalog' => [
        'search' => [
            'elasticsearch7' => [
                'enable_auth' => 1,
                'index' => [
                    'name' => 'magento2_product',
                    'shards' => 1,
                    'replicas' => 0,
                ],
            ],
        ],
    ],
];
```

### Elasticsearch Client Configuration

The client factory creates the connection:

```php
// Magento\Elasticsearch7\Model\Client\ElasticsearchFactory
declare(strict_types=1);

namespace Magento\Elasticsearch7\Model\Client;

class ElasticsearchFactory
{
    public function create(array $options): \Elasticsearch\Client
    {
        $builder = \Elasticsearch\ClientBuilder::create()
            ->setHosts($options['hosts']);

        if (!empty($options['username']) && !empty($options['password'])) {
            $builder->setBasicAuthentication(
                $options['username'],
                $options['password']
            );
        }

        if (!empty($options['ssl'])) {
            $builder->setSSL(false); // Configure SSL properly in production
        }

        return $builder->build();
    }
}
```

---

## 3. Indexer Configuration

The `catalogsearch_fulltext` indexer maintains the product search index in Elasticsearch.

### Indexer Registration

The indexer is defined in `Magento\CatalogSearch/etc/indexer.xml`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Indexer/etc/indexer.xsd">
    <indexer id="catalogsearch_fulltext"
             class="Magento\CatalogSearch\Model\Indexer\Fulltext"
             action="Magento\CatalogSearch\Model\Indexer\Action\Full"
             view="catalogsearch_fulltext">
        <schedule backend_entity="Magento\CatalogSearch\Model\Indexer\ScheduleIndexerPlugin"/>
    </indexer>
</config>
```

### Index Structure

The index name follows the pattern: `magento2_product_{store_id}_v{version}`:

```
magento2_product_1_v1    # Store 1, version 1
magento2_product_2_v1    # Store 2, version 1
```

### Index Mapping

The fulltext indexer creates the following field mapping:

```json
{
  "mappings": {
    "properties": {
      "entity_id": { "type": "integer" },
      "name": {
        "type": "text",
        "analyzer": "product_name_analyzer",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "description": {
        "type": "text",
        "analyzer": "product_description_analyzer"
      },
      "short_description": { "type": "text" },
      "sku": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "price": {
        "type": "scaled_float",
        "scaling_factor": "100"
      },
      "category_ids": { "type": "integer" },
      "store_id": { "type": "integer" },
      "visibility": { "type": "integer" },
      "status": { "type": "integer" },
      "weight": { "type": "integer" },
      "created_at": { "type": "date" },
      "updated_at": { "type": "date" },
      "thumbnail": { "type": "keyword" },
      "url_key": { "type": "keyword" },
      "in_stock": { "type": "integer" }
    }
  }
}
```

### Indexer Data Source

The fulltext indexer processes data from multiple sources:

```php
// Magento\CatalogSearch\Model\Indexer\Action\Full
declare(strict_types=1);

namespace Magento\CatalogSearch\Model\Indexer\Action;

class Full
{
    public function __construct(
        \Magento\Catalog\Model\Product\Attribute\Repository $attributeRepository,
        \Magento\CatalogSearch\Model\ResourceModel\Fulltext $fulltextResource,
        \Magento\Catalog\Model\Product\Option $productOption,
        \Magento\Eav\Model\Config $eavConfig
    ) {}

    public function rebuildIndex($storeId, $productIds = null): void
    {
        // 1. Fetch products with all searchable attributes
        $products = $this->getProducts($storeId, $productIds);

        // 2. Process each product for index
        foreach ($products as $productData) {
            // Apply search weight, synonyms, stopwords
            $indexData = $this->prepareIndexData($productData);
            $this->saveIndexData($indexData);
        }
    }

    private function prepareIndexData(array $productData): array
    {
        $indexData = [
            'entity_id' => $productData['entity_id'],
            'name' => $this->processTextField($productData['name']),
            'description' => $this->processTextField($productData['description']),
            'sku' => $productData['sku'],
            'price' => $productData['price'] ?? 0,
            'category_ids' => $productData['category_ids'] ?? [],
            'store_id' => $productData['store_id'],
            'visibility' => $productData['visibility'],
            'status' => $productData['status'],
            // Apply search weights
            'search_weight' => $this->calculateSearchWeight($productData),
        ];

        return $indexData;
    }
}
```

### Synonyms Configuration

Magento supports Elasticsearch synonym configuration:

```php
// Magento\CatalogSearch\Model\Search\SynonymReader
declare(strict_types=1);

namespace Magento\CatalogSearch\Model\Search;

class SynonymReader
{
    public function getSynonyms(int $websiteId): array
    {
        // Returns synonyms from catalog_search_synonym table
        // Format: ["tshirt,t-shirt,tee"] -> multi-match query expansion
    }

    public function buildSynonymQuery(string $phrase): array
    {
        return [
            'bool' => [
                'should' => [
                    ['match' => ['name' => $phrase]],
                    // Synonym expansion queries
                ],
                'minimum_should_match' => 1
            ]
        ];
    }
}
```

Synonyms are stored in `catalog_search_synonym` table:

| synonym_id | website_id | store_id | phrase | synonyms |
|------------|------------|----------|--------|----------|
| 1 | 1 | 0 | tshirt | t-shirt, tee, tshirts |

### Stopwords Configuration

Stopwords are filtered out during search:

```php
// Magento\Elasticsearch\Model\ResourceModel\Stopwords
declare(strict_types=1);

namespace Magento\Elasticsearch\Model\ResourceModel;

class Stopwords
{
    public function getStopwords(int $storeId): array
    {
        // Returns stopwords for store from catalog_search_stopwords table
        return ['a', 'an', 'the', 'is', 'are', 'was', 'were'];
    }

    public function buildStopwordFilter(string $field): array
    {
        return [
            'filter' => [
                'common' => [
                    'gram' => false,
                    'stopwords' => $this->getStopwords($storeId)
                ]
            ]
        ];
    }
}
```

---

## 4. Search Query Building

The query builder transforms Magento search requests into Elasticsearch queries.

### QueryBuilder Class

```php
// Magento\Elasticsearch\SearchAdapter\QueryBuilder
declare(strict_types=1);

namespace Magento\Elasticsearch\SearchAdapter;

use Magento\Framework\Search\Dynamic\DataProviderInterface;
use Magento\Framework\Search\Request\Query\BoolQuery;
use Magento\Framework\Search\Request\Query\MatchQuery;
use Magento\Framework\Search\Request\Query\Range;
use Magento\Elasticsearch\Model\Adapter\FieldMapper;

class QueryBuilder
{
    public function __construct(
        FieldMapper $fieldMapper,
        \Magento\Elasticsearch\Model\Adapter\Query\Builder\MatchQuery $matchQueryBuilder,
        \Magento\Elasticsearch\Model\Adapter\Query\Builder\TermQuery $termQueryBuilder,
        \Magento\Elasticsearch\Model\Adapter\Query\Builder\Range $rangeQueryBuilder
    ) {}

    public function build(\Magento\Framework\Search\Request $request): array
    {
        $query = $request->getQuery();
        $size = $request->getSize() ?? 100;
        $from = $request->getFrom() ?? 0;

        $esQuery = [
            'index' => $this->getIndexName(),
            'body' => [
                'from' => $from,
                'size' => $size,
                'query' => $this->buildBoolQuery($query),
                'sort' => $this->buildSort($request),
            ]
        ];

        // Add aggregations for facets
        $esQuery['body']['aggs'] = $this->buildAggregations();

        return $esQuery;
    }

    private function buildBoolQuery($query): array
    {
        $boolQuery = ['bool' => []];

        if ($query instanceof BoolQuery) {
            // Handle must (AND), should (OR), must_not (NOT)
            if (!empty($query->getMust())) {
                $boolQuery['bool']['must'] = array_map(
                    [$this, 'buildQueryClause'],
                    $query->getMust()
                );
            }

            if (!empty($query->getShould())) {
                $boolQuery['bool']['should'] = array_map(
                    [$this, 'buildQueryClause'],
                    $query->getShould()
                );
                $boolQuery['bool']['minimum_should_match'] = 1;
            }

            if (!empty($query->getMustNot())) {
                $boolQuery['bool']['must_not'] = array_map(
                    [$this, 'buildQueryClause'],
                    $query->getMustNot()
                );
            }
        }

        return $boolQuery;
    }
}
```

### Bool Query Types

Elasticsearch bool queries support four clauses:

```php
// bool query structure example
$boolQuery = [
    'bool' => [
        'must' => [
            // MUST match - affects relevance score
            ['match' => ['name' => 'blue shirt']],
            ['term' => ['visibility' => 4]], // Catalog search visibility
        ],
        'should' => [
            // SHOULD match - optional, increases score
            ['match' => ['description' => 'cotton']],
            ['match' => ['short_description' => 'cotton']],
        ],
        'must_not' => [
            // MUST NOT match - excludes documents
            ['term' => ['status' => 2]], // Disabled products
            ['term' => ['in_stock' => 0]],
        ],
        'filter' => [
            // FILTER - no scoring, cached
            ['term' => ['store_id' => 1]],
            ['range' => ['price' => ['gte' => 10, 'lte' => 100]]],
        ]
    ]
];
```

### Match vs Term Queries

**Match Query** - Full-text search with relevance scoring:

```php
// Match query for text search
['match' => [
    'name' => [
        'query' => 'blue shirt',
        'operator' => 'or', // 'and' for stricter matching
        'fuzziness' => 'AUTO', // Fuzzy matching for typos
        'boost' => 2.0, // Name field weighted higher
    ]
]]
```

**Term Query** - Exact value filtering (no scoring):

```php
// Term query for exact matches
['term' => ['sku.keyword' => 'WTSHIRT-BL-M']]
['term' => ['visibility' => 4]]
['term' => ['status' => 1]]
```

### Multi-Match Query

Search across multiple fields with field-specific weights:

```php
// Multi-match query across product fields
['multi_match' => [
    'query' => 'wireless headphone',
    'type' => 'best_fields', // or 'cross_fields', 'phrase', 'phrase_prefix'
    'fields' => [
        'name^3',           // Name weighted 3x
        'description^1',    // Description default weight
        'sku^2',            // SKU weighted 2x
        'short_description^1.5'
    ],
    'operator' => 'or',
    'minimum_should_match' => '75%',
    'fuzziness' => 'AUTO'
]]
```

### Fuzzy Matching

Handles typos and misspellings:

```php
// Fuzzy query configuration
[
    'match' => [
        'name' => [
            'query' => 'headphne', // User typo
            'fuzziness' => 'AUTO', // 0=exact, 1=1 edit, 2=2 edits
            'prefix_length' => 2,  // First 2 chars must match
            'max_expansions' => 50, // Max term variations
        ]
    ]
]

// AUTO fuzziness: edit distance based on term length
// - 0-2 chars: 0 edits
// - 3-5 chars: 1 edit
// - 6+ chars: 2 edits
```

---

## 5. Custom Search Adapter

Creating a custom search adapter allows integration with alternative search engines (Solr, Algolia, Elasticsearch OpenSource, etc.).

### When to Create a Custom Adapter

- Integrate with proprietary search services (Algolia, Elasticsearch Cloud)
- Implement specialized search ranking algorithms
- Add support for search engines not officially supported
- Custom faceting requirements beyond Elasticsearch capabilities

### SearchAdapterInterface Implementation

```php
// MyVendor\Search\Model\Adapter\MySearchAdapter
declare(strict_types=1);

namespace MyVendor\Search\Model\Adapter;

use Magento\Framework\Search\SearchAdapterInterface;
use Magento\Framework\Api\Search\SearchResultInterface;
use Magento\Framework\Search\Request;
use MyVendor\Search\Model\Client\MySearchClient;
use MyVendor\Search\Model\Response\QueryResponseFactory;

class MySearchAdapter implements SearchAdapterInterface
{
    private MySearchClient $client;
    private QueryResponseFactory $responseFactory;

    public function __construct(
        MySearchClient $client,
        QueryResponseFactory $responseFactory
    ) {
        $this->client = $client;
        $this->responseFactory = $responseFactory;
    }

    public function query(Request $request): SearchResultInterface
    {
        // 1. Transform Magento request to search engine query
        $searchQuery = $this->transformRequest($request);

        // 2. Execute against search engine
        $response = $this->client->search($searchQuery);

        // 3. Transform response to Magento SearchResult
        return $this->responseFactory->create($response, $request);
    }

    private function transformRequest(Request $request): array
    {
        $query = $request->getQuery();
        $searchQuery = [
            'index' => $this->client->getIndexName(),
            'body' => [
                'query' => $this->buildQuery($query),
                'from' => $request->getFrom() ?? 0,
                'size' => $request->getSize() ?? 100,
            ]
        ];

        // Add sorting
        foreach ($request->getSort() as $sort) {
            $searchQuery['body']['sort'][] = [
                $sort['field'] => ['order' => $sort['direction']]
            ];
        }

        return $searchQuery;
    }

    private function buildQuery($query): array
    {
        // Build search-engine specific query
        // Transform BoolQuery, MatchQuery, Range to native format
    }
}
```

### QueryResponseFactory

```php
// MyVendor\Search\Model\Response\QueryResponseFactory
declare(strict_types=1);

namespace MyVendor\Search\Model\Response;

class QueryResponseFactory
{
    public function create(array $response, Request $request): SearchResultInterface
    {
        $documents = [];
        $aggregations = [];

        // Transform engine response to Magento documents
        foreach ($response['hits']['hits'] as $hit) {
            $documents[] = new Document([
                'id' => $hit['_id'],
                'score' => $hit['_score'],
                'fields' => $hit['_source'],
            ]);
        }

        // Transform aggregations to facets
        foreach ($response['aggregations'] as $name => $bucket) {
            $aggregations[$name] = $this->buildBucket($bucket);
        }

        return new SearchResult([
            'items' => $documents,
            'aggregations' => $aggregations,
            'total' => $response['hits']['total']['value'],
            'searchRequest' => $request,
        ]);
    }

    private function buildBucket(array $bucket): AggregationInterface
    {
        $items = [];
        foreach ($bucket['buckets'] as $value) {
            $items[] = new AggregationValue([
                'value' => $value['key'],
                'count' => $value['doc_count'],
            ]);
        }

        return new Bucket([...]);
    }
}
```

### Dependency Injection Configuration

Register the custom adapter in `etc/di.xml`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Register custom search client -->
    <type name="MyVendor\Search\Model\Client\MySearchClient">
        <arguments>
            <argument name="host" xsi:type="string">localhost</argument>
            <argument name="port" xsi:type="number">8983</argument>
        </arguments>
    </type>

    <!-- Register custom adapter -->
    <type name="Magento\Framework\Search\AdapterPool">
        <arguments>
            <argument name="adapters" xsi:type="array">
                <item name="mycustom" xsi:type="object">MyVendor\Search\Model\Adapter\MySearchAdapter</item>
            </argument>
        </arguments>
    </type>

    <!-- Configure SearchEngine to use custom adapter -->
    <type name="Magento\Search\Model\SearchEngine">
        <arguments>
            <argument name="pool" xsi:type="object">MyVendor\Search\Model\Adapter\Pool\MySearchPool</argument>
        </arguments>
    </type>

</config>
```

### Adapter Pool Configuration

```xml
<!-- etc/di.xml in MyVendor\Search module -->
<type name="MyVendor\Search\Model\Adapter\Pool\MySearchPool">
    <arguments>
        <argument name="defaultSearchEngine" xsi:type="string">mycustom</argument>
        <argument name="engines" xsi:type="array">
            <item name="mycustom" xsi:type="array">
                <item name="adapter" xsi:type="object">MyVendor\Search\Model\Adapter\MySearchAdapter</item>
                <item name="client" xsi:type="object">MyVendor\Search\Model\Client\MySearchClient</item>
            </item>
        </argument>
    </arguments>
</type>
```

### Document Interface Implementation

```php
// Magento\Framework\Api\Search\Document
declare(strict_types=1);

namespace Magento\Framework\Api\Search;

class Document implements DocumentInterface
{
    private string $id;
    private array $fields;
    private float $score;

    public function __construct(array $fields)
    {
        $this->id = $fields['id'] ?? null;
        $this->fields = $fields;
        $this->score = $fields['_score'] ?? 0;
    }

    public function getId(): string
    {
        return $this->id;
    }

    public function getField(string $name): mixed
    {
        return $this->fields[$name] ?? null;
    }

    public function getFields(): array
    {
        return $this->fields;
    }

    public function getScore(): float
    {
        return $this->score;
    }
}
```

---

## 6. Catalog Search Optimization

Optimize search performance through configuration and index tuning.

### Search Weight Configuration

Product attributes have configurable search weights:

```php
// Magento\Catalog\Model\Product\Attribute\Repository
declare(strict_types=1);

namespace Magento\Catalog\Model\Product\Attribute;

class Repository
{
    public function getSearchableAttributes(): array
    {
        // Returns attributes with search_weight set
    }
}
```

Set searchable attribute weights via admin: **Stores > Attributes > Product > Search**

| Weight | Description | Example |
|--------|-------------|---------|
| 10 | Highest relevance | Product Name |
| 5 | High relevance | SKU, Brand |
| 3 | Medium relevance | Description |
| 1 | Low relevance | Short Description |

### Batch Indexer Size

Control memory usage during full reindex:

> **Note:** The `catalogsearch_fulltext` indexer batch_size defaults to 50 in Magento 2.4.8, not 1000. A batch_size of 1000 is dangerously high and can cause memory issues during reindex. The `batch_size` value in `env.php` or environment variables applies to the async reindex operation, which is different from the search indexer's internal batch processing.

```php
// Environment variable or env.php
return [
    // ...
    'system' => [
        'default' => [
            'catalog_search' => [
                'batch_size' => '50', // Default for M2.4.8 fulltext indexer
            ]
        ]
    ]
];
```

Or via CLI:

```bash
# Set batch size via env variable
export 'catalog_search_batch_size=50'

# Or in app/etc/env.php
putenv('catalog_search_batch_size=50');
```

### Async Reindexing

Enable async indexing for large catalogs:

```php
// app/etc/env.php
return [
    // ...
    'cron' => [
        'index' => [
            'catalog_search_fulltext' => [
                'cron_expr' => '*/5 * * * *', // Every 5 minutes
                'use_different_lock' => true,
            ]
        ]
    ]
];
```

Manual async reindex trigger:

```bash
# Trigger async reindex
bin/magento cron:run --group=index

# Check catalogsearch indexer status
bin/magento indexer:status catalogsearch_fulltext
```

### Search Recommendations

Enable search suggestions for query improvement:

```php
// Store > Configuration > Catalog > Catalog Search > Search Recommendations
return [
    'catalog' => [
        'search' => [
            'recommendations_enabled' => 1,
            'max_count' => 10, // Max suggestions
            'show_suggestion_count' => 1,
        ]
    ]
];
```

### Synonyms Management

Programmatically manage search synonyms:

```php
// Magento\CatalogSearch\Model\Synonym
declare(strict_types=1);

namespace Magento\CatalogSearch\Model;

class SynonymRepository
{
    public function saveSynonyms(array $synonyms): void
    {
        // Batch save synonyms
        foreach ($synonyms as $synonym) {
            $this->resourceModel->save($synonym);
        }
    }

    public function getSynonymsForPhrase(string $phrase, int $storeId): array
    {
        // Return all synonyms for phrase including scope
    }
}

// Example: Import synonyms from CSV
$synonymData = [
    ['phrase' => 'phone', 'synonyms' => 'telephone,mobile,smartphone'],
    ['phrase' => 'laptop', 'synonyms' => 'notebook,computer,pc'],
];
```

### Query Debugging

Enable query debug mode for development:

```php
// app/etc/di.xml
<type name="Magento\Elasticsearch\SearchAdapter\Adapter">
    <arguments>
        <argument name="debugLog" xsi:type="boolean">true</argument>
    </arguments>
</type>

// Or via environment variable
export 'MAGE_DEBUG_MODE=true'

// Check query logs
tail -f var/log/system.log | grep Elasticsearch
```

---

## 7. Autocomplete / Quick Search

The autocomplete feature provides instant search suggestions as users type.

### Ajax Suggest Controller

```php
// Magento\Search\Controller\Ajax\Suggest
declare(strict_types=1);

namespace Magento\Search\Controller\Ajax;

class Suggest extends \Magento\Framework\App\Action\HttpGetAction
{
    public function __construct(
        \Magento\Framework\App\Request\Http $request,
        \Magento\Search\Model\QueryFactory $queryFactory,
        \Magento\Search\Model\Autocomplete\DataProvider $dataProvider
    ) {}

    public function execute(): \Magento\Framework\App\Response\Http
    {
        $queryText = $this->getRequest()->getParam('q', '');
        $size = (int)$this->getRequest()->getParam('size', 10);

        // Build search query for suggestions
        $query = $this->queryFactory->create()->setQueryText($queryText);

        // Get autocomplete items
        $items = $this->dataProvider->getItems($query, $size);

        $response = $this->getResponse();
        $response->representJson(
            json_encode(['items' => $items])
        );

        return $response;
    }
}
```

### Autocomplete Data Provider

```php
// Magento\Search\Model\Autocomplete\DataProvider
declare(strict_types=1);

namespace Magento\Search\Model\Autocomplete;

class DataProvider
{
    public function __construct(
        \Magento\CatalogSearch\Model\ResourceModel\Query\CollectionFactory $collectionFactory,
        \Magento\Search\Model\QueryFactory $queryFactory
    ) {}

    public function getItems(\Magento\Search\Model\Query $query, int $size): array
    {
        $collection = $this->collectionFactory->create();
        $collection->setQueryText($query->getQueryText())
            ->setPageSize($size)
            ->load();

        $items = [];
        foreach ($collection as $suggestion) {
            $items[] = [
                'title' => $suggestion->getTitle(),
                'url' => $suggestion->getUrl(),
                'num_results' => $suggestion->getNumResults(),
            ];
        }

        return $items;
    }
}
```

### JavaScript Autocomplete Integration

The frontend uses a RequireJS-based autocomplete widget:

```javascript
// Magento_Search/autocomplete.js
define([
    'jquery',
    'mage/url',
    'mage/storage',
    'domReady!'
], function ($, url) {
    'use strict';

    $.widget('mage.autocomplete', {
        options: {
            minSearchLength: 2,
            responseFields: ['title', 'url', 'num_results'],
            template: '<li data-role="autocomplete-item"><a href="<%- url %>"><%- title %></a></li>',
            delay: 300,
        },

        _create: function () {
            this.element.attr('autocomplete', 'off');
            this.element.on('input', this._onInput.bind(this));
            this.element.on('focusout', this._onFocusOut.bind(this));
        },

        _onInput: function (e) {
            var value = this.element.val();

            if (value.length >= this.options.minSearchLength) {
                this._debouncedSearch(value);
            }
        },

        _debouncedSearch: function () {
            clearTimeout(this._timeout);
            this._timeout = setTimeout(function () {
                this._search.apply(this, arguments);
            }.bind(this), this.options.delay);
        },

        _search: function (query) {
            $.get(url.build('search/ajax/suggest?q=' + encodeURIComponent(query)))
                .done(function (response) {
                    this._renderSuggestions(response.items);
                }.bind(this));
        },

        _renderSuggestions: function (items) {
            var html = '';
            items.forEach(function (item) {
                html += this._renderItem(item);
            }.bind(this));

            this._getPopup().html(html).show();
        },

        _renderItem: function (item) {
            return this.options.template
                .replace('<%- url %>', item.url)
                .replace('<%- title %>', item.title);
        },

        _getPopup: function () {
            if (!this._popup) {
                this._popup = $('<ul class="search-autocomplete"></ul>')
                    .insertAfter(this.element);
            }
            return this._popup;
        },
    });

    return $.mage.autocomplete;
});
```

### Knockout Autocomplete Template

```html
<!-- Magento_Search/templates/autocomplete.html -->
<ul class="search-autocomplete" data-bind="foreach: suggestions">
    <li class="item" data-bind="click: $parent.selectItem.bind($parent, $data)">
        <a data-bind="attr: { href: url }">
            <span data-bind="text: title"></span>
            <span class="results-count" data-bind="text: '(' + num_results + ' results)'"></span>
        </a>
    </li>
</ul>
```

### Typeahead Configuration

```javascript
// requirejs-config.js
var config = {
    map: {
        '*': {
            'Magento_Search/js/autocomplete': 'MyModule/js/typeahead'
        }
    },
    config: {
        mixins: {
            'Magento_Search/js/autocomplete': {
                'MyModule/js/mixins/autocomplete-mixin': true
            }
        }
    }
};
```

---

## 8. GraphQL Search

GraphQL provides a flexible search interface for headless implementations.

### Products Query with Search

```graphql
type Query {
  products(
    search: String
    filter: ProductAttributeFilterInput
    sort: ProductAttributeSortInput
    pageSize: Int = 20
    currentPage: Int = 1
  ): Products
}
```

### SearchCriteria Input Type

```graphql
# Complex search with multiple criteria
input SearchCriteria {
  filter_groups: [FilterGroup]
  sort: [Sort]
  page_size: Int
  current_page: Int
  search: String
}

input FilterGroup {
  filters: [Filter]
}

input Filter {
  field: String!
  value: String
  condition_type: String  # eq, neq, like, in, between, etc.
}
```

### GraphQL Resolver for Products

```php
// Magento\CatalogGraphQl\Model\Resolver\Products
declare(strict_types=1);

namespace Magento\CatalogGraphQl\Model\Resolver;

use Magento\Framework\GraphQL\Config\GraphQlAttribute;
use Magento\CatalogGraphQl\Model\Search\SearchResult;
use Magento\Framework\GraphQL\Query\FieldHandlerInterface;

class Products implements ResolverInterface
{
    public function __construct(
        \Magento\CatalogGraphQl\Model\Search\SearchResultFactory $searchResultFactory,
        \Magento\Search\Model\SearchEngine $searchEngine
    ) {}

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ): array {
        $searchCriteria = $this->buildSearchCriteria($args);
        $searchResult = $this->searchEngine->search($searchCriteria);

        return $this->formatResult($searchResult);
    }

    private function buildSearchCriteria(array $args): \Magento\Framework\Search\Request
    {
        $requestBuilder = $this->requestBuilderFactory->create();
        $requestBuilder->setRequestName('cataloggraphql');

        // Build bool query from search string
        if (!empty($args['search'])) {
            $requestBuilder->bind('query', $args['search']);
        }

        // Add filters
        foreach ($args['filter'] ?? [] as $filter) {
            $this->addFilter($requestBuilder, $filter);
        }

        return $requestBuilder->create();
    }
}
```

### Aggregations in Results (Facets)

```graphql
type Products {
  items: [ProductInterface]
  aggregations: [Aggregation]!
  total_count: Int!
  page_info: SearchResultPageInfo
}

type Aggregation {
  attribute_code: String!
  label: String!
  options: [AggregationOption]!
  count: Int!
}

type AggregationOption {
  label: String!
  value: String!
  count: Int!
}
```

### PHP Aggregation Builder

```php
// Magento\Elasticsearch\Model\Adapter\Aggregation\CategoryAggregationBuilder
declare(strict_types=1);

namespace Magento\Elasticsearch\Model\Adapter\Aggregation;

class CategoryAggregationBuilder
{
    public function build(array $bucketConfigs, array $documents): array
    {
        $aggregations = [];

        foreach ($bucketConfigs as $bucketName => $config) {
            $aggregations[$bucketName] = [
                'buckets' => $this->buildBuckets($config, $documents)
            ];
        }

        return $aggregations;
    }

    private function buildBuckets(AggregationConfig $config, array $documents): array
    {
        $buckets = [];

        if ($config->getType() === 'terms') {
            // Build term buckets (e.g., category_ids, price ranges)
            foreach ($this->getValues($config->getField()) as $value) {
                $buckets[] = [
                    'key' => $value,
                    'doc_count' => $this->countDocuments($config->getField(), $value, $documents)
                ];
            }
        }

        return $buckets;
    }
}
```

### Complete Search Query Example

```php
// Example: GraphQL products search with filters
public function searchProducts(string $searchTerm, array $filters, int $pageSize, int $currentPage): array
{
    $request = $this->requestBuilder
        ->setRequestName('catalog_product_search')
        ->bind('query', $searchTerm)
        ->setSize($pageSize)
        ->setFrom($currentPage * $pageSize)
        ->setSort('relevance', 'desc')
        ->create();

    $searchResult = $this->searchEngine->search($request);

    return [
        'items' => $this->convertDocuments($searchResult->getItems()),
        'aggregations' => $this->buildAggregations($searchResult->getAggregations()),
        'total_count' => $searchResult->getTotal(),
        'page_info' => [
            'current_page' => $currentPage,
            'page_size' => $pageSize,
            'total_pages' => ceil($searchResult->getTotal() / $pageSize)
        ]
    ];
}
```

---

## Summary

Magento 2.4.8's search architecture provides a flexible, extensible framework for catalog search powered by Elasticsearch 7:

| Component | Purpose | Key Class |
|-----------|---------|-----------|
| SearchEngine | Façade for search adapters | `Magento\Search\Model\SearchEngine` |
| Elasticsearch7 Adapter | ES 7.x query execution | `Magento\Elasticsearch7\SearchAdapter\Adapter` |
| QueryBuilder | Request to ES query transformation | `Magento\Elasticsearch\SearchAdapter\QueryBuilder` |
| Fulltext Indexer | Product data indexing | `Magento\CatalogSearch\Model\Indexer\Fulltext` |
| Autocomplete | Quick search suggestions | `Magento\Search\Controller\Ajax\Suggest` |
| GraphQL Search | Headless product search | `Magento\CatalogGraphQl\Model\Resolver\Products` |

Understanding these components enables customization of search behavior, integration with alternative search engines, and optimization of catalog search performance for large product catalogs.

(End of file - total 1423 lines)