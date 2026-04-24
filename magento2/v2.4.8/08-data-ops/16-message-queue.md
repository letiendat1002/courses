---
title: "30 - Message Queue & Async Operations"
description: "Deep dive into Magento 2.4.8 message queue: synchronous vs asynchronous operations, queue configuration, consumers, and bulk operations."
tags: magento2, message-queue, async, consumers, bulk-operations, amqp, rabbitmq
rank: 16
supersedes_topic: 08
pathways: [magento2-deep-dive]
---

# Message Queue & Async Operations

Magento 2.4.8 implements a robust message queue architecture that enables asynchronous processing of operations, decoupling system components and improving overall performance and scalability. This article explores the queue framework, configuration files, publisher/consumer implementation, bulk operations, and dead letter queue handling.

---

## 1. Queue Architecture Overview

### Synchronous vs Asynchronous Operations

**Synchronous operations** execute sequentially - the client waits for each operation to complete before sending the next request. In Magento, this means the frontend user waits while catalog updates, inventory checks, or order processing completes in real-time.

**Asynchronous operations** allow the client to continue immediately while work happens in the background. The message queue acts as a buffer, decoupling the request producer from the consumer processing.

### When to Use Async

Use asynchronous operations when:

- **Long-running tasks** - Operations that take > 100ms to complete
- **Non-critical path work** - Email notifications, logging, audit trail updates
- **High-volume batch processing** - Bulk inventory updates, price changes, export jobs
- **External system integration** - Third-party API calls that may have variable latency
- **Scale-critical operations** - Operations that benefit from horizontal consumer scaling

**Avoid async** when:
- User must see immediate result (add to cart, checkout)
- Operation is part of a transactional workflow requiring rollback
- Results are needed before proceeding to next step

### Magento Queue Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Publisher  │────▶│    Topic     │────▶│   Binding   │────▶│    Queue    │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                                                                   │
                                                                   ▼
                                                            ┌─────────────┐
                                                            │  Consumer   │
                                                            │  (Handler) │
                                                            └─────────────┘
```

Magento 2.4.8 supports multiple queue adapters:
- **MySQL** (default) - Database-backed queue for development/low-volume
- **AMQP** - Advanced Message Queuing Protocol for production (RabbitMQ, ActiveMQ)

---

## 2. Message Queue Components

### Core Components

| Component | Purpose | Magento Class |
|-----------|---------|---------------|
| **Publisher** | Sends messages to topics | `\Magento\Framework\MessageQueue\PublisherInterface` |
| **Topic** | Named message destination routing messages by routing key | `communication.xml` |
| **Exchange** | Routes messages from topics to queues based on bindings | `queue_topology.xml` |
| **Binding** | Defines relationship between exchange and queue | `queue_topology.xml` |
| **Queue** | Message buffer holding messages until consumer processes them | `queue_topology.xml` |
| **Consumer** | Retrieves and processes messages from queue | Implement `ConsumerInterface` |
| **Schema** | Database schema for MySQL adapter | `queue_schema.xml` |

### Publisher Interface

```php
// Magento\Framework\MessageQueue\PublisherInterface
namespace Magento\Framework\MessageQueue;

/**
 * Publisher interface for sending messages to queue
 */
interface PublisherInterface
{
    /**
     * Publish message to specified topic
     *
     * @param string $topic
     * @param array|string $data
     * @return void
     */
    public function publish($topic, $data);
}
```

### Default Publishers

Magento provides several predefined publishers for common operations:

- `inventory.indexer.update.in.inventory` - Inventory index updates
- `inventory.indexer.update.multi.inventory` - Multi-source inventory
- `checkout.submission.reverse` - Checkout reversal operations
- `asycoperations\BundleDetailRepository的感情` - Bundle product operations
- `quote.save.shipping.assignment` - Shipping assignment in quotes

---

## 3. etc/communication.xml

The `communication.xml` file declares all topics the system uses, defining the message schema and whether operations are synchronous or asynchronous.

### File Location
```
app/code/{Vendor}/{Module}/etc/communication.xml
```

### Synchronous vs Asynchronous Topics

**Synchronous topics** use `method` to define a direct callback:

```xml
<?xml version="1.0"?>
<!--
/**
 * Copyright © Magento, Inc. All rights reserved.
 * See COPYING.txt for license details.
 */
-->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Communication/etc/communication.xsd">
    <topic name="customer.authentication.login"
           schema="Magento\Customer\Api\Data\CustomerInterface">
        <handler name="loginRequestHandler"
                 type="Magento\Customer\Model\AuthenticationHandler"
                 method="processLoginRequest"/>
    </topic>
</config>
```

**Asynchronous topics** omit the handler, allowing queued execution:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Communication/etc/communication.xsd">
    <topic name="inventory.indexer.update.single"
           schema="Magento\InventoryIndexers\Model\Indexer\Inventory\StockItemData">
        <!-- No handler = asynchronous processing -->
    </topic>

    <topic name="inventory.indexer.update.multi"
           schema="Magento\InventoryIndexers\Api\Data\StockItemData[]">
        <!-- No handler = asynchronous processing -->
    </topic>
</config>
```

### Topic Schema

The `schema` attribute specifies the fully-qualified class name for message serialization:

```xml
<topic name="sales.rule.save"
       schema="Magento\SalesRule\Api\DataRuleInterface">
```

For array types in async operations:

```xml
<topic name="catalog.product.save"
       schema="Magento\Catalog\Api\Data\ProductInterface[]">
```

### Complete Example

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Communication/etc/communication.xsd">
    <!-- Synchronous topic with handler -->
    <topic name="customer.authentication.login"
           schema="Magento\Customer\Api\Data\CustomerInterface">
        <handler name="loginRequestHandler"
                 type="Magento\Customer\Model\AuthenticationHandler"
                 method="processLoginRequest"/>
    </topic>

    <!-- Asynchronous topic - bulk operations -->
    <topic name="inventory.indexer.update.multi"
           schema="Magento\InventoryIndexers\Api\Data\StockItemData[]">
    </topic>

    <!-- Asynchronous topic - single entity -->
    <topic name="inventory.indexer.update.single"
           schema="Magento\InventoryIndexers\Model\Indexer\Inventory\StockItemData">
    </topic>

    <!-- Asynchronous topic - async operations API -->
    <topic name="async_operations_all"
           schema="Magento\Framework\MessageQueue\DataTransferObject">
    </topic>
</config>
```

---

## 4. etc/queue_topology.xml

The `queue_topology.xml` declares the exchange-queue bindings and connection configuration.

### File Location
```
app/code/{Vendor}/{Module}/etc/queue_topology.xml
```

### Connection Configuration

```xml
<?xml version="1.0"?>
<!--
/**
 * Copyright © Magento, Inc. All rights reserved.
 * See COPYING.txt for license details.
 */
-->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework-messagequeue:etc/queue_topology.xsd">
    <broker xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:noNamespaceSchemaLocation="urn:magento:framework-messagequeue:etc/queue_topology.xsd">
        <connection name="amqp"
                    host="rabbitmq.local"
                    port="5672"
                    username="magento"
                    password="password"
                    vhost="/"
                    exchange="magento-db"
                    protocol="amqp"
                    max_messages="10000"/>
    </broker>
</config>
```

### Exchange Declaration

```xml
<exchange name="sales.order.created"
         type="topic"
         connection="amqp"
         durable="true">
    <bindings>
        <binding queue="sales.order.created" routing_key="sales.order.created"/>
    </bindings>
</exchange>
```

### Queue Declaration with Binding

```xml
<exchange name="magento-db"
         type="topic"
         connection="mysql"
         durable="true">
    <bindings>
        <binding queue="inventory.indexer.update.single"
                 routing_key="inventory.indexer.update.single"/>
        <binding queue="inventory.indexer.update.multi"
                 routing_key="inventory.indexer.update.multi"/>
    </bindings>
</exchange>
```

### Full Topology Example

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework-messagequeue:etc/queue_topology.xsd">

    <!-- MySQL Adapter (Default) -->
    <broker>
        <connection name="mysql"
                    type="magento"
                    dbTablePrefix="queue_"
                    username="magento_user"
                    password="magento_password"
                    host="localhost"
                    dbname="magento_db"
                    port="3306"
                    maxMessages="10000"/>
    </broker>

    <!-- AMQP Adapter (Production) -->
    <broker>
        <connection name="amqp"
                    host="rabbitmq.example.com"
                    port="5672"
                    username="admin"
                    password="secret"
                    vhost="/magento"
                    exchange="magento"
                    protocol="amqp"/>
    </broker>

    <!-- Exchange for Inventory Operations -->
    <exchange name="inventory.indexer"
              type="topic"
              connection="amqp"
              durable="true">
        <bindings>
            <binding queue="inventory.update.single"
                     routing_key="inventory.update.single"/>
            <binding queue="inventory.update.bulk"
                     routing_key="inventory.update.bulk"/>
        </bindings>
    </exchange>

    <!-- Exchange for Asynchronous Operations -->
    <exchange name="async.operations"
              type="topic"
              connection="mysql"
              durable="true">
        <bindings>
            <binding queue="async.operations.queue"
                     routing_key="async.operations.*"/>
        </bindings>
    </exchange>
</config>
```

### Exchange Types

| Type | Description | Use Case |
|------|-------------|----------|
| `topic` | Routing key pattern matching | Most Magento operations |
| `direct` | Exact routing key matching | Single consumer queues |
| `fanout` | Route to all bound queues | Broadcast operations |

---

## 5. etc/queue_consumer.xml

The `queue_consumer.xml` declares consumers, their connection, queue name, and handler class.

### File Location
```
app/code/{Vendor}/{Module}/etc/queue_consumer.xml
```

### Basic Consumer Declaration

```xml
<?xml version="1.0"?>
<!--
/**
 * Copyright © Magento, Inc. All rights reserved.
 * See COPYING.txt for license details.
 */
-->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Async:Consumer/etc/consumer.xsd">
    <consumer name="inventory.indexer.update.single"
              queue="inventory.update.single"
              connection="amqp"
              handler="Magento\InventoryIndexer\Queue\Handler\InventoryUpdateHandler"
              maxMessages="1000"/>
</config>
```

### Consumer with Custom Handler

```xml
<consumer name="custom.operation.processor"
          queue="custom.operation.queue"
          connection="mysql"
          handler="Vendor\Module\Model\Queue\CustomHandler"
          maxMessages="5000"
          consumerInstance="Vendor\Module\Model\Queue\CustomConsumer"/>
```

### Multiple Consumers Example

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Async:Consumer/etc/consumer.xsd">

    <!-- Inventory Indexer Consumers -->
    <consumer name="inventory.indexer.update.single"
              queue="inventory.update.single"
              connection="amqp"
              handler="Magento\InventoryIndexer\Queue\Handler\InventoryUpdateHandler"/>

    <consumer name="inventory.indexer.update.multi"
              queue="inventory.update.multi"
              connection="amqp"
              handler="Magento\InventoryIndexer\Queue\Handler\MultiInventoryUpdateHandler"/>

    <!-- Asynchronous Operations Consumer -->
    <consumer name="async.operations"
              queue="async.operations.queue"
              connection="mysql"
              handler="Magento\AsynchronousOperations\Api\OperationHandlerInterface"
              consumerInstance="Magento\AsynchronousOperations\Model\Consumer"/>

    <!-- Codegen Pattern Consumer (Generated) -->
    <consumer name="codegenAll"
              queue="codegen.queue.all"
              connection="mysql"
              handler="Magento\Magento\Setup\Handler\CodegenHandler"
              consumerInstance="Magento\Framework\MessageQueue\Consumer"/>
</config>
```

### Consumer Attributes

| Attribute | Required | Description |
|-----------|----------|-------------|
| `name` | Yes | Unique consumer identifier |
| `queue` | Yes | Queue name to consume from |
| `connection` | Yes | Connection name from `queue_topology.xml` |
| `handler` | No | Handler class `execute(topic, message)` |
| `maxMessages` | No | Max messages before restart (default: 1000) |
| `consumerInstance` | No | Custom consumer class override |

---

## 6. Custom Publisher

### Injecting the Publisher

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Framework\MessageQueue\PublisherInterface;
use Magento\Framework\Serialize\SerializerInterface;

/**
 * Publisher service for custom module operations
 */
class CustomPublisher
{
    /**
     * @var PublisherInterface
     */
    private PublisherInterface $publisher;

    /**
     * @var SerializerInterface
     */
    private SerializerInterface $serializer;

    /**
     * @param PublisherInterface $publisher
     * @param SerializerInterface $serializer
     */
    public function __construct(
        PublisherInterface $publisher,
        SerializerInterface $serializer
    ) {
        $this->publisher = $publisher;
        $this->serializer = $serializer;
    }

    /**
     * Publish single item update
     *
     * @param int $itemId
     * @param array $data
     * @return void
     */
    public function publishItemUpdate(int $itemId, array $data): void
    {
        $message = $this->serializer->serialize([
            'item_id' => $itemId,
            'update_data' => $data,
            'timestamp' => time()
        ]);

        $this->publisher->publish('vendor.item.update', $message);
    }

    /**
     * Publish bulk item updates
     *
     * @param array $items Array of ['id' => int, 'data' => array]
     * @return void
     */
    public function publishBulkUpdate(array $items): void
    {
        $messages = [];
        foreach ($items as $item) {
            $messages[] = [
                'id' => $item['id'],
                'operation' => $item['operation'] ?? 'update',
                'data' => $item['data'] ?? [],
                'timestamp' => time()
            ];
        }

        $this->publisher->publish('vendor.item.bulk', $this->serializer->serialize($messages));
    }
}
```

### Publishing Synchronous vs Asynchronous

The same publisher interface handles both modes based on topic configuration in `communication.xml`:

```php
<?php
// Both calls look identical to publisher, behavior differs by topic config
$this->publisher->publish('synchronous.topic', $data); // Blocks until complete
$this->publisher->publish('asynchronous.topic', $data); // Returns immediately
```

### Publishing from Service Classes

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Api;

use Magento\Framework\MessageQueue\PublisherInterface;

/**
 * Service contract for inventory operations
 */
class InventoryUpdateService implements InventoryUpdateServiceInterface
{
    /**
     * @var PublisherInterface
     */
    private PublisherInterface $publisher;

    /**
     * @param PublisherInterface $publisher
     */
    public function __construct(PublisherInterface $publisher)
    {
        $this->publisher = $publisher;
    }

    /**
     * {@inheritdoc}
     */
    public function updateStock(int $productId, float $quantity): bool
    {
        // Synchronous immediate update
        $this->publisher->publish(
            'inventory.stock.update.sync',
            [
                'product_id' => $productId,
                'quantity' => $quantity,
                'source' => 'default'
            ]
        );

        // Trigger async reindex
        $this->publisher->publish(
            'inventory.indexer.update.single',
            [
                'product_id' => $productId
            ]
        );

        return true;
    }
}
```

---

## 7. Custom Consumer

### Consumer Interface

```php
// Magento\Framework\MessageQueue\ConsumerInterface
namespace Magento\Framework\MessageQueue;

/**
 * Queue consumer interface
 */
interface ConsumerInterface
{
    /**
     * Process messages in the queue
     *
     * @return void
     */
    public function process(): void;
}
```

### Simple Consumer Implementation

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Queue;

use Magento\Framework\MessageQueue\ConsumerInterface;
use Magento\Framework\MessageQueue\ConsumerConfig\Data;
use Psr\Log\LoggerInterface;

/**
 * Custom consumer for vendor operations
 */
class CustomConsumer implements ConsumerInterface
{
    /**
     * @var LoggerInterface
     */
    private LoggerInterface $logger;

    /**
     * @var CustomProcessor
     */
    private CustomProcessor $processor;

    /**
     * @param LoggerInterface $logger
     * @param CustomProcessor $processor
     */
    public function __construct(
        LoggerInterface $logger,
        CustomProcessor $processor
    ) {
        $this->logger = $logger;
        $this->processor = $processor;
    }

    /**
     * Process messages from queue
     *
     * @return void
     */
    public function process(): void
    {
        $this->logger->info('CustomConsumer: Processing started');
        // Message processing logic handled by message callback
    }

    /**
     * Execute single message
     *
     * @param string $topic
     * @param string $message
     * @return void
     */
    public function execute(string $topic, string $message): void
    {
        $this->logger->info('CustomConsumer: Processing message', [
            'topic' => $topic,
            'message_size' => strlen($message)
        ]);

        try {
            $data = json_decode($message, true);
            $this->processor->process($data);
            $this->logger->info('CustomConsumer: Message processed successfully');
        } catch (\Throwable $e) {
            $this->logger->error('CustomConsumer: Processing failed', [
                'topic' => $topic,
                'error' => $e->getMessage()
            ]);
            throw $e;
        }
    }
}
```

### Handler Class (Alternative Approach)

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Queue;

use Magento\Framework\MessageQueue\HandlerInterface;
use Magento\Framework\Serialize\SerializerInterface;
use Psr\Log\LoggerInterface;

/**
 * Handler for vendor operations queue
 */
class VendorOperationHandler implements HandlerInterface
{
    /**
     * @var LoggerInterface
     */
    private LoggerInterface $logger;

    /**
     * @var SerializerInterface
     */
    private SerializerInterface $serializer;

    /**
     * @var OperationService
     */
    private OperationService $operationService;

    /**
     * @param LoggerInterface $logger
     * @param SerializerInterface $serializer
     * @param OperationService $operationService
     */
    public function __construct(
        LoggerInterface $logger,
        SerializerInterface $serializer,
        OperationService $operationService
    ) {
        $this->logger = $logger;
        $this->serializer = $serializer;
        $this->operationService = $operationService;
    }

    /**
     * Execute handler with message
     *
     * @param string $topic
     * @param string $message
     * @return void
     */
    public function execute(string $topic, string $message): void
    {
        $this->logger->debug('VendorOperationHandler: Received message', [
            'topic' => $topic
        ]);

        $data = $this->serializer->unserialize($message);

        switch ($topic) {
            case 'vendor.item.update':
                $this->handleItemUpdate($data);
                break;
            case 'vendor.item.delete':
                $this->handleItemDelete($data);
                break;
            case 'vendor.item.bulk':
                $this->handleBulkOperation($data);
                break;
            default:
                $this->logger->warning('Unknown topic: ' . $topic);
        }
    }

    /**
     * Handle single item update
     *
     * @param array $data
     * @return void
     */
    private function handleItemUpdate(array $data): void
    {
        $this->operationService->updateItem(
            $data['item_id'],
            $data['update_data']
        );
    }

    /**
     * Handle item deletion
     *
     * @param array $data
     * @return void
     */
    private function handleItemDelete(array $data): void
    {
        $this->operationService->deleteItem($data['item_id']);
    }

    /**
     * Handle bulk operation
     *
     * @param array $data
     * @return void
     */
    private function handleBulkOperation(array $data): void
    {
        foreach ($data['items'] ?? [] as $item) {
            $this->operationService->updateItem($item['id'], $item['data']);
        }
    }
}
```

### Registering the Consumer

```xml
<!-- etc/queue_consumer.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Async:Consumer/etc/consumer.xsd">

    <consumer name="vendor.operations"
              queue="vendor.operations.queue"
              connection="mysql"
              handler="Vendor\Module\Model\Queue\VendorOperationHandler"
              consumerInstance="Vendor\Module\Model\Queue\CustomConsumer"
              maxMessages="5000"/>
</config>
```

---

## 8. Bulk Operations

The Bulk Operations API enables scheduling multiple entity operations for asynchronous processing via a single API call.

### AsyncEntities Bulk API

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Api;

use Magento\AsynchronousOperations\Api\BulkManagementInterface;
use Magento\AsynchronousOperations\Api\FloatbulkOperationsInterface;
use Magento\Framework\Bulk\OperationInterface;

/**
 * Service for bulk asynchronous operations
 */
class BulkOperationService
{
    /**
     * @var BulkManagementInterface
     */
    private BulkManagementInterface $bulkManagement;

    /**
     * @var OperationsPoolInterface
     */
    private OperationsPoolInterface $operationsPool;

    /**
     * @param BulkManagementInterface $bulkManagement
     * @param OperationsPoolInterface $operationsPool
     */
    public function __construct(
        BulkManagementInterface $bulkManagement,
        OperationsPoolInterface $operationsPool
    ) {
        $this->bulkManagement = $bulkManagement;
        $this->operationsPool = $operationsPool;
    }

    /**
     * Schedule bulk product updates
     *
     * @param array $productUpdates Array of [sku => [price => float]]
     * @param int $userId
     * @return string Bulk UUID
     */
    public function scheduleBulkProductUpdates(array $productUpdates, int $userId): string
    {
        $operations = [];

        foreach ($productUpdates as $sku => $updateData) {
            $operations[] = [
                'sku' => $sku,
                'price' => $updateData['price'] ?? null,
                'status' => $updateData['status'] ?? 1
            ];
        }

        $topicName = 'async_operations_all';
        $bulkUuid = $this->bulkManagement->scheduleBulk(
            $userId,
            [$topicName],
            [$this->serializeOperations($operations)],
            'Bulk product update - ' . date('Y-m-d H:i:s')
        );

        return $bulkUuid;
    }

    /**
     * Serialize operations for queue
     *
     * @param array $operations
     * @return string
     */
    private function serializeOperations(array $operations): string
    {
        return json_encode($operations);
    }
}
```

### BulkManagementInterface

```php
// Magento\AsynchronousOperations\Api\BulkManagementInterface
namespace Magento\AsynchronousOperations\Api;

/**
 * Interface BulkManagementInterface
 */
interface BulkManagementInterface
{
    /**
     * Schedule bulk operation
     *
     * @param int $userId
     * @param string[] $topics
     * @param string[] $serializedMessages
     * @param string $description
     * @param int|null $groupStartedAt
     * @return string Bulk UUID
     */
    public function scheduleBulk(
        int $userId,
        array $topics,
        array $serializedMessages,
        string $description,
        ?int $groupStartedAt = null
    ): string;

    /**
     * Schedule bulk async operations from entity IDs
     *
     * @param int $userId
     * @param string $topic
     * @param int[] $entityIds
     * @param string $description
     * @return string Bulk UUID
     */
    public function scheduleBulkAsyncOperations(
        int $userId,
        string $topic,
        array $entityIds,
        string $description
    ): string;
}
```

### Operations Pool Configuration

```xml
<!-- etc/queue_consumer.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Async:Consumer/etc/consumer.xsd">

    <consumer name="async_operations"
              queue="async.operations.queue"
              connection="mysql"
              handler="Magento\AsynchronousOperations\Api\OperationHandlerInterface"
              consumerInstance="Magento\AsynchronousOperations\Model\Consumer"/>

    <!-- Floatbulk Operations -->
    <consumer name="floatbulk_operations"
              queue="floatbulk.operations.queue"
              connection="mysql"
              handler="Magento\AsynchronousOperations\Api\FloatbulkOperationsInterface"
              consumerInstance="Magento\AsynchronousOperations\Model\FloatbulkConsumer"/>
</config>
```

### Operation Handler Implementation

```php
<?php
declare(strict_types=1);

namespace Magento\AsynchronousOperations\Model;

use Magento\Framework\Serialize\SerializerInterface;
use Magento\Framework\MessageQueue\HandlerInterface;
use Psr\Log\LoggerInterface;

/**
 * Handler for async operations queue
 */
class OperationHandler implements HandlerInterface
{
    /**
     * @var LoggerInterface
     */
    private LoggerInterface $logger;

    /**
     * @var SerializerInterface
     */
    private SerializerInterface $serializer;

    /**
     * @var OperationRepositoryInterface
     */
    private OperationRepositoryInterface $operationRepository;

    /**
     * @param LoggerInterface $logger
     * @param SerializerInterface $serializer
     * @param OperationRepositoryInterface $operationRepository
     */
    public function __construct(
        LoggerInterface $logger,
        SerializerInterface $serializer,
        OperationRepositoryInterface $operationRepository
    ) {
        $this->logger = $logger;
        $this->serializer = $serializer;
        $this->operationRepository = $operationRepository;
    }

    /**
     * Execute operation handler
     *
     * @param string $topic
     * @param string $message
     * @return void
     */
    public function execute(string $topic, string $message): void
    {
        $data = $this->serializer->unserialize($message);

        if (isset($data['bulk_uuid'])) {
            $this->processByBulkUuid($data);
        } else {
            $this->processDirect($topic, $data);
        }
    }

    /**
     * Process operation with bulk UUID
     *
     * @param array $data
     * @return void
     */
    private function processByBulkUuid(array $data): void
    {
        $operation = $this->operationRepository->getByBulkUuid($data['bulk_uuid']);
        $operation->setStatus(OperationInterface::STATUS_COMPLETE);
        $operation->setResult(json_encode(['processed' => true]));
        $this->operationRepository->save($operation);
    }
}
```

### Bulk Status Tracking

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Api;

use Magento\AsynchronousOperations\Api\Data\BulkSummaryInterface;
use Magento\AsynchronousOperations\Api\Data\BulkOperationsStatusInterface;

/**
 * Service for checking bulk operation status
 */
class BulkStatusService
{
    /**
     * @var BulkOperationsStatusInterface
     */
    private BulkOperationsStatusInterface $statusService;

    /**
     * @param BulkOperationsStatusInterface $statusService
     */
    public function __construct(
        BulkOperationsStatusInterface $statusService
    ) {
        $this->statusService = $statusService;
    }

    /**
     * Get bulk operation status
     *
     * @param string $bulkUuid
     * @return BulkSummaryInterface
     */
    public function getBulkStatus(string $bulkUuid): BulkSummaryInterface
    {
        return $this->statusService->getBulkOperationsStatus($bulkUuid);
    }

    /**
     * Check if bulk is complete
     *
     * @param string $bulkUuid
     * @return bool
     */
    public function isComplete(string $bulkUuid): bool
    {
        $status = $this->getBulkStatus($bulkUuid);
        return $status->getStatus() === 'complete';
    }
}
```

---

## 9. Dead Letter Queue (DLQ)

Dead Letter Queues handle messages that fail processing after exhausting retries, preventing message loss and enabling manual intervention.

### Failure Configuration

Messages are sent to DLQ after:
1. **Max Retries Exceeded** - Default: 3 retries
2. **Consumer Exception** - Non-recoverable processing error
3. **Message Expiration** - Time-to-live exceeded

### DLQ Configuration in env.php

DLQ is configured ONLY in `env.php` under the `queue.dead_letter_queue` path:

```php
// app/etc/env.php
return [
    // ...
    'queue' => [
        'dead_letter_queue' => [
            'source' => 'database',  // or 'amqp' for RabbitMQ
            'connection' => 'mysql',
            'queue' => 'queue_dead_letter',
        ],
    ],
];
```

**Note:** Do NOT configure DLQ bindings in `queue_topology.xml`. DLQ routing is handled automatically by Magento's queue framework based on the `env.php` configuration above. The DLQ exchange and queue bindings are created implicitly when the dead letter queue is configured.

### Custom DLQ Configuration

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework-messagequeue:etc/queue_topology.xsd">

    <!-- Custom Exchange with DLQ -->
    <exchange name="vendor.custom.exchange"
              type="topic"
              connection="mysql"
              durable="true">
        <bindings>
            <!-- Main queue -->
            <binding queue="vendor.custom.queue"
                     routing_key="vendor.custom.#"/>

            <!-- Dead Letter Queue -->
            <binding queue="vendor.custom.dlq"
                     routing_key="vendor.custom.dlq"/>
        </bindings>
    </exchange>
</config>
```

### MySQL Queue Schema for DLQ

```xml
<!-- etc/queue_schema.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework-messagequeue:etc/queue_schema.xsd">
    <broker connection="mysql" default="true">
        <queue name="vendor.custom.dlq" consumer="vendor.custom.dlq.consumer">
            <schema type="Magento\Framework\MessageQueue\DataTransferObject"/>
        </queue>
    </broker>
</config>
```

### DLQ Consumer Handler

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Queue;

use Magento\Framework\MessageQueue\HandlerInterface;
use Magento\Framework\Serialize\SerializerInterface;
use Psr\Log\LoggerInterface;

/**
 * Handler for Dead Letter Queue processing
 */
class DlqHandler implements HandlerInterface
{
    /**
     * @var LoggerInterface
     */
    private LoggerInterface $logger;

    /**
     * @var SerializerInterface
     */
    private SerializerInterface $serializer;

    /**
     * @var DlqRepository
     */
    private DlqRepository $dlqRepository;

    /**
     * @param LoggerInterface $logger
     * @param SerializerInterface $serializer
     * @param DlqRepository $dlqRepository
     */
    public function __construct(
        LoggerInterface $logger,
        SerializerInterface $serializer,
        DlqRepository $dlqRepository
    ) {
        $this->logger = $logger;
        $this->serializer = $serializer;
        $this->dlqRepository = $dlqRepository;
    }

    /**
     * Execute DLQ message handling
     *
     * @param string $topic
     * @param string $message
     * @return void
     */
    public function execute(string $topic, string $message): void
    {
        $data = $this->serializer->unserialize($message);

        $this->logger->warning('DLQ: Processing failed message', [
            'topic' => $topic,
            'original_topic' => $data['_original_topic'] ?? 'unknown',
            'error' => $data['_error'] ?? 'unknown',
            'retry_count' => $data['_retry_count'] ?? 0
        ]);

        // Store failed message for manual review
        $this->dlqRepository->saveFailedMessage([
            'topic' => $topic,
            'message' => $message,
            'failure_reason' => $data['_error'] ?? 'Unknown',
            'retry_count' => $data['_retry_count'] ?? 0,
            'failed_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

### Retry Configuration in communication.xml

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Communication/etc/communication.xsd">
    <topic name="vendor.custom.operation"
           schema="Vendor\Module\Api\Data\CustomDataInterface">
    </topic>

    <topic name="vendor.critical.operation"
           schema="Vendor\Module\Api\Data\CriticalDataInterface">
    </topic>
</config>
```

**Note:** M2.4.8 does NOT support `retry` or `maxDelay` attributes on topic elements in `communication.xml`. Retry behavior is handled by the consumer configuration or env.php settings.

### Message Tracking Schema

MySQL adapter stores queue state in these tables:

| Table | Purpose |
|-------|---------|
| `queue_message` | Message metadata and status |
| `queue_message_status` | Current message status |
| `queue_bulk` | Bulk operation tracking |
| `queue_operation` | Individual operation status |

---

## 10. CLI Commands

### List All Consumers

```bash
bin/magento queue:consumers:list
```

**Output:**
```
InventoryIndexeratcherUpdateSingle
InventoryIndexeratcherUpdateMulti
async_operations
codegenAll
exportprocessor
inventory.indexer.update.single
inventory.indexer.update.multi
inventory.mass.update
sales.order.save
```

### Start a Consumer

```bash
# Start specific consumer
bin/magento queue:consumers:start inventory.indexer.update.single

# Start with max messages limit
bin/magento queue:consumers:start async_operations --max-messages=5000

# Start with batch size for bulk consumer
bin/magento queue:consumers:start async_operations --batch-size=100
```

**Options:**
| Option | Description |
|--------|-------------|
| `--max-messages` | Max messages before consumer stops (default: 0 = unlimited) |
| `--batch-size` | Number of messages to process per batch |
| `--single-thread` | Prevent parallel consumer instances |
| `--exclude` | Consumer instance IDs to exclude |

### Check Queue Status

```bash
bin/magento queue:status
```

**Output:**
```
+-------------------------+----------------+-------------+----------+
| Queue                   | Messages Total | Ready       | Running  |
+-------------------------+----------------+-------------+----------+
| inventory.indexer      | 152            | 145         | 2        |
| async.operations        | 2341           | 2298        | 5        |
| sales.order.save        | 89             | 89          | 0        |
+-------------------------+----------------+-------------+----------+
```

### Additional Queue Commands

```bash
# Show queue topology information
bin/magento queue:topology:show

# Clean old messages (maintenance)
bin/magento queue:purge <consumer-name>

# Retry failed message from DLQ
bin/magento queue:retry:retry --topic="vendor.custom.operation"
```

### RabbitMQ Management (with AMQP)

```bash
# List queues via RabbitMQ CLI
rabbitmqctl list_queues name messages messages_ready

# View message content
rabbitmqctl get_messages <queue-name>
```

### systemd Service for Production

Create consumer as a systemd service:

```ini
# /etc/systemd/system/magento-queue-consumer.service
[Unit]
Description=Magento Queue Consumer
After=network.target rabbitmq.service

[Service]
Type=simple
User=magento
Group=magento
ExecStart=/usr/bin/php /var/www/magento2/bin/magento queue:consumers:start inventory.indexer.update.single --single-thread
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start
sudo systemctl enable magento-queue-consumer
sudo systemctl start magento-queue-consumer

# View logs
sudo journalctl -u magento-queue-consumer -f
```

---

## Summary

Magento 2.4.8's message queue architecture provides a flexible framework for asynchronous operations:

- **Queue Architecture** - Decouples producers from consumers, enabling scalable processing
- **Configuration Files** - `communication.xml`, `queue_topology.xml`, `queue_consumer.xml` define the queue ecosystem
- **Topics & Exchanges** - Route-based message forwarding with binding configuration
- **Custom Publishers** - Inject `PublisherInterface` to send messages asynchronously
- **Custom Consumers** - Implement `ConsumerInterface` or `HandlerInterface` for processing
- **Bulk Operations** - `BulkManagementInterface` schedules multiple entity operations
- **Dead Letter Queue** - Handles failures with retry limits and DLQ routing
- **CLI Commands** - `queue:consumers:list`, `queue:consumers:start`, `queue:status` for management

For production environments, configure AMQP (RabbitMQ) for reliable, clustered message processing with horizontal consumer scaling.