---
title: "08 - Async Message Queue Corrections"
description: "Magento 2.4.8 async message queue corrections: schema errors found in online tutorials, correct queue_topology.xml/consumer.xsd paths, and DLQ configuration in env.php"
tags:
  - magento2
  - message-queue
  - amqp
  - async
  - queue-topology
  - consumers
  - dead-letter-queue
rank: 21
see_also:
  - comprehensive: "Full MQ architecture (publishers, consumers, bulk ops, exchanges) is covered in [16-message-queue.md](./16-message-queue.md)"
duplicates_topic: 16
pathways:
  - magento2-deep-dive
---

# Async Message Queue

Magento 2.4.8 uses an asynchronous message queue architecture built on RabbitMQ (or the database queue as a fallback). This article covers the correct configuration patterns for `queue_topology.xml`, `consumers.xml`, and `env.php`.

> **Important**: Many online articles and blog posts show incorrect schema elements for `queue_topology.xml`. The `<max-retries>` and `<retry-delay-ms>` elements do NOT exist in the Magento `async_topology.xsd` schema. DLQ configuration belongs in `env.php`, not in XML.

---

## 1. queue_topology.xml — The Correct Schema

### The Wrong Approach (DLQ in XML)

Many articles incorrectly show DLQ configuration directly in `queue_topology.xml`:

```xml
<!-- WRONG - These elements do NOT exist in async_topology.xsd -->
<broker xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Async/MessageQueue/etc/async_topology.xsd">
    <DLQ>
        <queue>inventory.DLQ</queue>
        <max-retries>3</max-retries>
        <retry-delay-ms>5000</retry-delay-ms>
    </DLQ>
</broker>
```

The elements `<max-retries>` and `<retry-delay-ms>` are **not valid** in the async topology schema.

### The Correct Schema Elements

The valid elements in `queue_topology.xsd` are:

| Element | Attributes/Children | Purpose |
|---------|---------------------|---------|
| `<exchange>` | `name`, `type`, `connection` | Declares a message exchange |
| `<binding>` | `queue`, `exchange`, `binding_format` | Binds a queue to an exchange |
| `<queue>` | `name`, `connection`, `consumer` | Declares a queue with consumer |
| `<schema>` | `type`, `binding_format` | Schema for type exchange bindings |

#### Example: Declaring an Exchange and Queue

```xml
<!-- app/code/Vendor/Module/etc/queue_topology.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Async/MessageQueue/etc/async_topology.xsd">
    <broker exchanges="magento_custom">
        <!-- Declare an exchange -->
        <exchange name="magento_custom.exchange"
                  type="topic"
                  connection="amqp"/>

        <!-- Bind a queue to the exchange -->
        <binding queue="magento_custom.inventory.bulk"
                 exchange="magento_custom.exchange"
                 binding_format="topic">
            <topic name="inventory.bulk.*"/>
        </binding>

        <!-- Declare a queue with consumer reference -->
        <queue name="magento_custom.inventory.bulk"
               connection="amqp"
               consumer="inventory_bulk_sql"/>
    </broker>
</config>
```

#### Example: Type Exchange with Schema

For type exchanges that use routing keys:

```xml
<!-- app/code/Vendor/Module/etc/queue_topology.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Async/MessageQueue/etc/async_topology.xsd">
    <broker exchanges="magento_custom">
        <exchange name="magento_custom.type.exchange"
                  type="type"
                  connection="amqp"/>

        <binding queue="magento_custom.product.update"
                 exchange="magento_custom.type.exchange"
                 binding_format="type">
            <topic name="product.update"/>
            <schema type="Magento\Catalog\Api\Data\ProductInterface"/>
        </binding>

        <queue name="magento_custom.product.update"
               connection="amqp"
               consumer="product_update_consumer"/>
    </broker>
</config>
```

---

## 2. consumers.xml — The Correct M2.4.8 Format

### The Wrong Approach (Old XSD Location)

Some articles reference the wrong XSD schema location:

```xml
<!-- WRONG - Wrong schema location -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:MessageQueue/etc/consumer.xsd">
```

### The Correct Format for M2.4.8

The async consumer schema is in the `Async` namespace:

```xml
<!-- app/code/Vendor/Module/etc/queue/consumers.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Async/Consumer/etc/consumer.xsd">
    <consumer name="inventory_bulk_sql"
              queue="inventory.bulk.sql"
              consumerInstance="Magento\Framework\MessageQueue\Consumer"
              handler="Vendor\Module\Model\BulkSqlHandler">
        <observation>
            <topic name="inventory.bulk.sql.executed"/>
        </observation>
    </consumer>

    <consumer name="product_update_consumer"
              queue="magento_custom.product.update"
              consumerInstance="Magento\Framework\MessageQueue\Consumer"
              handler="Vendor\Module\Model\ProductUpdateHandler">
        <observation>
            <topic name="catalog_product_save_after"/>
        </observation>
    </consumer>
</config>
```

#### Consumer Element Attributes

| Attribute | Required | Description |
|-----------|----------|-------------|
| `name` | Yes | Unique consumer identifier |
| `queue` | Yes | Queue name to consume from |
| `consumerInstance` | Yes | Consumer class instance |
| `handler` | Yes | Message handler class |
| `max-messages` | No | Max messages per batch (default: 1000) |
| `visible-to` | No | Visibility setting |

#### Observation Element

The `<observation>` block allows consumers to subscribe to multiple topics:

```xml
<observation>
    <topic name="catalog_product_save_after"/>
    <topic name="catalog_product_delete_after"/>
</observation>
```

---

## 3. env.php — DLQ and Queue Configuration

### The Correct DLQ Configuration

Dead Letter Queue (DLQ) settings belong in `env.php` under the `queue` section, **not** in `queue_topology.xml`.

```php
<!-- app/code/Vendor/Module/etc/env.php -->
<?php
return [
    // ... other config ...

    'queue' => [
        'dead_letter_queue' => [
            'max retries' => 3,
            'retry_delay' => 5000,
        ],
    ],

    // Consumer configurations
    'queue_consumers' => [
        'max_messages' => 1000,
        ' consumers' => [
            'inventory_bulk_sql' => [
                'max-messages' => 500,
            ],
            'product_update_consumer' => [
                'max-messages' => 200,
            ],
        ],
    ],
];
```

### Complete env.php Queue Section

```php
<?php
return [
    // ... other config sections (db, cache, etc.) ...

    'queue' => [
        // Dead Letter Queue settings
        'dead_letter_queue' => [
            'max retries' => 3,
            'retry_delay' => 5000,
        ],

        // RabbitMQ connection (if using AMQP)
        'amqp' => [
            'host' => '127.0.0.1',
            'port' => 5672,
            'user' => 'guest',
            'password' => 'guest',
            'vhost' => '/',
            'ssl' => false,
        ],
    ],

    // Consumer configuration
    'queue_consumers' => [
        // Global max messages setting
        'max_messages' => 1000,

        // Per-consumer overrides
        'consumers' => [
            'inventory_bulk_sql' => [
                'max-messages' => 500,
                'single-queue' => true,
            ],
            'async_operations_api' => [
                'max-messages' => 100,
            ],
        ],
    ],
];
```

#### env.php Queue Configuration Options

| Key | Description |
|-----|-------------|
| `queue.dead_letter_queue.max retries` | Number of retry attempts before message goes to DLQ |
| `queue.dead_letter_queue.retry_delay` | Delay in milliseconds between retries |
| `queue.amqp.*` | RabbitMQ connection settings |
| `queue_consumers.max_messages` | Global max messages per consumer |
| `queue_consumers.consumers.*.max-messages` | Per-consumer message limit |

---

## 4. Complete Module Configuration Example

### File Structure

```
app/code/Vendor/Module/
├── etc/
│   ├── env.php                    # DLQ and queue config
│   ├── queue_topology.xml         # Exchange and binding declarations
│   └── queue/
│       └── consumers.xml          # Consumer declarations
├── Model/
│   ├── BulkSqlHandler.php         # Message handler
│   └── ProductUpdateHandler.php   # Message handler
└── registration.php
```

### queue_topology.xml

```xml
<!-- app/code/Vendor/Module/etc/queue_topology.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Async/MessageQueue/etc/async_topology.xsd">
    <broker exchanges="magento_custom">
        <!-- Exchange declaration -->
        <exchange name="magento_custom.exchange"
                  type="topic"
                  connection="amqp"/>

        <!-- Queue to exchange binding -->
        <binding queue="magento_custom.inventory.bulk"
                 exchange="magento_custom.exchange"
                 binding_format="topic">
            <topic name="inventory.bulk.*"/>
        </binding>

        <!-- Queue with consumer reference -->
        <queue name="magento_custom.inventory.bulk"
               connection="amqp"
               consumer="inventory_bulk_sql"/>
    </broker>
</config>
```

### consumers.xml

```xml
<!-- app/code/Vendor/Module/etc/queue/consumers.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Async/Consumer/etc/consumer.xsd">
    <consumer name="inventory_bulk_sql"
              queue="magento_custom.inventory.bulk"
              consumerInstance="Magento\Framework\MessageQueue\Consumer"
              handler="Vendor\Module\Model\BulkSqlHandler"
              max-messages="500"/>
</config>
```

### env.php

```php
<?php
return [
    // ... existing config ...

    'queue' => [
        'dead_letter_queue' => [
            'max retries' => 3,
            'retry_delay' => 5000,
        ],
        'amqp' => [
            'host' => '127.0.0.1',
            'port' => 5672,
            'user' => 'guest',
            'password' => 'guest',
            'vhost' => '/',
        ],
    ],

    'queue_consumers' => [
        'max_messages' => 1000,
        'consumers' => [
            'inventory_bulk_sql' => [
                'max-messages' => 500,
            ],
        ],
    ],
];
```

### Message Handler

```php
// app/code/Vendor/Module/Model/BulkSqlHandler.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Framework\MessageQueue\ConsumerInterface;
use Magento\Framework\MessageQueue\DomainInterface;
use Magento\Framework\MessageQueue\MessageController;

class BulkSqlHandler implements DomainInterface
{
    /**
     * @param ConsumerInterface $consumer
     * @param MessageController $messageController
     */
    public function __construct(
        ConsumerInterface $consumer,
        MessageController $messageController
    ) {
        $this->consumer = $consumer;
        $this->messageController = $messageController;
    }

    /**
     * Execute message handler
     *
     * @param \Magento\Framework\MessageQueue\EnvelopeInterface $envelope
     * @return void
     */
    public function execute(\Magento\Framework\MessageQueue\EnvelopeInterface $envelope): void
    {
        $body = $envelope->getBody();
        $topicName = $envelope->getProperties()['topic_name'] ?? '';

        // Process the message
        $data = json_decode($body, true);

        // Handle inventory bulk SQL operations
        if (strpos($topicName, 'inventory.bulk') !== false) {
            $this->processBulkInventory($data);
        }
    }

    /**
     * Process bulk inventory data
     *
     * @param array $data
     * @return void
     */
    private function processBulkInventory(array $data): void
    {
        // Implementation
    }
}
```

---

## 5. Common Mistakes and Corrections

| Mistake | Correction |
|---------|------------|
| DLQ config in `queue_topology.xml` with `<max-retries>` | Move to `env.php` under `queue.dead_letter_queue` |
| Duplicate `queue_consumers` keys | Merge into single `queue_consumers` array |
| Wrong XSD: `urn:magento:framework:MessageQueue/etc/consumer.xsd` | Correct: `urn:magento:framework:Async/Consumer/etc/consumer.xsd` |
| Using `<DLQ>` element in topology XML | DLQ is configured via `env.php` only |
| `connection="rabbitmq"` | Use `connection="amqp"` for RabbitMQ |

---

## Summary: Correct Patterns for M2.4.8

| Configuration File | Correct Approach |
|-------------------|------------------|
| **queue_topology.xml** | Use `<exchange>`, `<binding>`, `<queue>` elements only. No DLQ config. |
| **consumers.xml** | Use `urn:magento:framework:Async/Consumer/etc/consumer.xsd` schema |
| **env.php** | DLQ settings under `queue.dead_letter_queue`. Consumer limits under `queue_consumers` |

Always verify schema locations against the Magento 2.4.8 codebase. The `async_topology.xsd` and `consumer.xsd` schemas are in the `Framework/Async/MessageQueue/` and `Framework/Async/Consumer/` directories respectively.

(End of file)