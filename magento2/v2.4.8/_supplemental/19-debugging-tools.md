---
title: "21 - Debugging Tools"
description: "Comprehensive guide to Magento 2.4.8 debugging tools: query logging, Xdebug configuration, profiling, Elasticsearch debugging, and performance diagnostics"
tags:
  - magento2
  - debugging
  - xdebug
  - profiling
  - elasticsearch
  - query-logging
rank: 19
pathways:
  - magento2-deep-dive
---

# Debugging Tools

Magento 2.4.8 provides comprehensive debugging tools for diagnosing performance issues, query problems, and application errors. This article covers real debugging configurations and techniques for open-source Magento 2.4.8.

---

## 1. Query Logging

### Enable Database Query Logging

To log all database queries, configure `env.php`:

```php
<?php
// app/etc/env.php
return [
    'system' => [
        'default' => [
            'system' => [
                'dev' => [
                    'query_log' => 1,
                    'query_log_depth' => 3
                ]
            ]
        ]
    ]
];
```

Query logs are written to `var/debug/db.log` (not `var/log/debug/db_repo.log`).

### Query Log Analysis

```bash
# View recent query log entries
tail -f var/debug/db.log

# Find slow queries (look for execution time markers)
grep -E "slow|EXECUTE" var/debug/db.log

# Analyze query patterns
awk '/EXECUTE/ {print $NF}' var/debug/db.log | sort | uniq -c | sort -rn | head -20
```

### Enable DB Query Logging with Profiling

```php
<?php
// app/etc/env.php - Full profiling configuration
return [
    'db' => [
        'connection' => [
            'default' => [
                'host' => '127.0.0.1',
                'port' => 3306,
                'dbname' => 'magento_db',
                'username' => 'magento_user',
                'password' => 'password',
                'model' => 'mysql4',
                'engine' => 'innodb',
                'initStatements' => 'SET NAMES utf8mb4'
            ]
        ]
    ],
    'system' => [
        'default' => [
            'system' => [
                'dev' => [
                    'query_log' => 1,
                    'query_log_depth' => 10
                ]
            ]
        ]
    ]
];
```

---

## 2. Xdebug Configuration

### PHPStorm Xdebug Setup

```php
# php.ini development settings
[xdebug]
zend_extension = /usr/lib/php/extensions/debug.so
xdebug.mode = debug,develop
xdebug.start_with_request = trigger
xdebug.client_host = 127.0.0.1
xdebug.client_port = 9003
xdebug.idekey = PHPSTORM
xdebug.max_nesting_level = 512
xdebug.log_level = 0

# For CLI debugging
xdebug.cli_color = 1
```

### Launch Configuration for PHPStorm

```xml
<!-- .idea/workspace.xml or .vscode/launch.json -->
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Magento 2 Debug",
            "type": "php",
            "request": "launch",
            "serverSourceRoot": "/var/www/magento2",
            "pathMappings": {
                "/var/www/magento2": "${workspaceFolder}"
            },
            "port": 9003,
            "idekey": "PHPSTORM"
        }
    ]
}
```

### CLI Xdebug for bin/magento Commands

```bash
# Enable xdebug for CLI commands
PHP_IDE_CONFIG="serverName=Magento2" php -d xdebug.mode=debug -d xdebug.client_host=127.0.0.1 bin/magento cache:flush

# Or use environment variable
export XDEBUG_CONFIG="idekey=PHPSTORM"
bin/magento indexer:reindex
```

---

## 3. Magento Profiling

### Enable Profiling

```php
<?php
// app/etc/env.php
return [
    'system' => [
        'default' => [
            'system' => [
                'dev' => [
                    'debug' => 1,
                    'query_log' => 1
                ]
            ]
        ]
    ]
];
```

### Using Alan Storm's Commerce Bug

```php
// In your module's composer.json require section:
"alan-qc/magento2-profiling": "~1.0"

// Or enable via env.php
return [
    'system' => [
        'default' => [
            'system' => [
                'dev' => [
                    'debug' => 1
                ]
            ]
        ]
    ]
];
```

### Custom Timing Profiler

```php
// app/code/Vendor/Module/Helper/Profiler.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Helper;

use Magento\Framework\Profiler\DriverInterface;

class Profiler
{
    /**
     * @var DriverInterface
     */
    private $profiler;

    /**
     * @param DriverInterface $profiler
     */
    public function __construct(DriverInterface $profiler)
    {
        $this->profiler = $profiler;
    }

    /**
     * Start timing
     *
     * @param string $identifier
     * @return void
     */
    public function start(string $identifier): void
    {
        $this->profiler->start($identifier);
    }

    /**
     * Stop timing
     *
     * @param string $identifier
     * @return void
     */
    public function stop(string $identifier): void
    {
        $this->profiler->stop($identifier);
    }

    /**
     * Get timer stats
     *
     * @param string $identifier
     * @return array
     */
    public function getStats(string $identifier): array
    {
        return $this->profiler->getStats($identifier);
    }
}
```

---

## 4. Elasticsearch Debugging

### Correct Elasticsearch Client Configuration

```php
<?php
// app/etc/env.php
return [
    'system' => [
        'default' => [
            'system' => [
                'dev' => [
                    'debug' => 1
                ]
            ]
        ]
    ]
];
```

### Elasticsearch hosts configuration in env.php:

```php
<?php
// app/etc/env.php - ES client configuration
return [
    'system' => [
        'default' => [
            'system' => [
                'dev' => [
                    'debug' => 1
                ]
            ]
        ]
    ]
];

# Additional Elasticsearch configuration in etc/env.php:
return [
    'elasticsearch' => [
        'hosts' => ['localhost:9200']
    ]
];
```

### Correct Elasticsearch Indexer Batch Size

The Magento-recommended Elasticsearch bulk batch size is **50** (not 1000, which is dangerously high).

```php
<?php
// app/etc/env.php - Indexer configuration
return [
    'indexer' => [
        'batch_size' => 50,
        'batch_size_bytes' => '10mb'
    ]
];
```

### Indexer Configuration for catalog_category_flat and catalog_product_flat

Indexer configuration for flat tables in `etc/env.php`:

```php
<?php
// app/etc/env.php - Flat catalog indexer configuration
return [
    'indexer' => [
        'catalog_category_flat' => [
            'batch_size' => 50
        ],
        'catalog_product_flat' => [
            'batch_size' => 50
        ]
    ]
];
```

### Test Elasticsearch Connection

```bash
# Test ES connection
curl -X GET 'localhost:9200/_cluster/health?pretty'

# Test specific index
curl -X GET 'localhost:9200/magento2_product_1/_search?pretty'

# Check index mappings
curl -X GET 'localhost:9200/magento2_product_1/_mapping?pretty'
```

### Elasticsearch Query Debugging

```bash
# Enable slow log in ES
curl -X PUT 'localhost:9200/_cluster/settings' -H 'Content-Type: application/json' -d '
{
    "transient": {
        "logger.index.slowsearch": "INFO"
    }
}
'

# View slow logs
curl -X GET 'localhost:9200/_nodes/stats/indices/search?pretty'
```

---

## 5. Firebug/Logging

### Magento Logger Configuration

```php
<?php
// app/code/Vendor/Module/etc/di.xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Framework\Logger\Handler\Debug">
        <arguments>
            <argument name="fileName" xsi:type="string">var/debug/debug.log</argument>
        </arguments>
    </type>
</config>
```

### Custom Logger Implementation

```php
// app/code/Vendor/Module/Logger/Handler/CustomHandler.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Logger\Handler;

use Magento\Framework\Logger\Handler\Base;
use Monolog\Logger;

class CustomHandler extends Base
{
    /**
     * @var string
     */
    protected $fileName = '/var/debug/custom.log';

    /**
     * @var int
     */
    protected $loggerType = Logger::DEBUG;
}
```

### Logging to var/debug/debug.log

```php
<?php
// In any class using Psr\Logger\LoggerInterface
public function __construct(
    \Psr\Log\LoggerInterface $logger
) {
    $this->logger = $logger;
}

public function debugMethod(): void
{
    $this->logger->debug('Method started', [
        'param1' => $value1,
        'param2' => $value2
    ]);
}
```

---

## 6. Development Mode Diagnostics

### Enable Developer Mode

```bash
# Set developer mode
export MODE=developer
bin/magento deploy:mode:set developer

# Or in env.php
return [
    'MAGE_MODE' => 'developer'
];
```

### Display Errors in Developer Mode

```php
<?php
// pub/error_processor.php - Enable detailed errors
ini_set('display_errors', 1);
error_reporting(E_ALL);
```

### Check Current Mode

```bash
bin/magento deploy:mode:show
```

---

## 7. db_schema.xml Declaration

### Correct M2.4.8 db_schema.xml Format

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etcSchema.xsd">
    <table name="vendor_module_custom_table" resource="default"
           engine="innodb"
           comment="Custom Table">
        <column xsi:type="int" name="entity_id" padding="10"
                unsigned="true" nullable="false" identity="true"/>
        <column xsi:type="varchar" name="name" length="255" nullable="false"/>
        <column xsi:type="datetime" name="created_at" nullable="false"/>
        <column xsi:type="text" name="description" nullable="true"/>
        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="entity_id"/>
        </constraint>
        <index referenceId="IDX_NAME" indexType="index">
            <column name="name"/>
        </index>
    </table>
</schema>
```

### Add Column with db_schema.xml

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etcSchema.xsd">
    <table name="vendor_module_custom_table" resource="default"
           engine="innodb" comment="Custom Table">
        <column xsi:type="int" name="entity_id" padding="10"
                unsigned="true" nullable="false" identity="true"/>
        <column xsi:type="varchar" name="name" length="255" nullable="false"/>
        <column xsi:type="datetime" name="created_at" nullable="false"/>
        <column xsi:type="text" name="description" nullable="true"/>
        <!-- Add new column -->
        <column xsi:type="decimal" name="price" precision="12" scale="4" nullable="true"/>
        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="entity_id"/>
        </constraint>
    </table>
</schema>
```

---

## 8. Async Bulk Operations Logging

For debugging async bulk operations:

```php
<?php
// app/etc/env.php
return [
    'system' => [
        'default' => [
            'system' => [
                'dev' => [
                    'debug' => 1
                ]
            ]
        ]
    ]
];
```

### Message Queue Consumer Logging

```bash
# Run consumer with logging
bin/magento queue:consumers:run async.operations.all --verbose

# Check message queue status
bin/magento queue:queue:status
```

---

## 9. Performance Debugging

### Enable New Relic Logging

```bash
# In env.php
return [
    'newrelic' => [
        'application' => 'Magento2',
        'monolog' => [
            'level' => 'warning'
        ]
    ]
];
```

### MySQL Slow Query Log

```sql
-- Enable slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
SET GLOBAL long_query_time = 2;
```

### Cache Debugging

```bash
# Enable cache debug logging
bin/magento cache:enable debug

# View cache statistics
bin/magento cache:stats

# Clean cache and measure impact
bin/magento cache:flush
# Note the time it takes to rebuild
```

---

## 10. Remote Debugging

### Configure Remote Debug Server

```php
<?php
// For Xdebug 3.x - use xdebug.ini
zend_extension=xdebug.so
xdebug.mode=debug
xdebug.client_discovery_client=1
xdebug.client_host=<YOUR_IP>
xdebug.client_port=9003
xdebug.idekey=PHPSTORM
```

### VS Code Launch Configuration

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Listen for Xdebug",
            "type": "php",
            "request": "launch",
            "port": 9003,
            "log": true,
            "pathMappings": {
                "/var/www/html": "${workspaceFolder}"
            },
            "ignore": [
                "/var/www/html/vendor/**"
            ]
        }
    ]
}
```

---

## Summary: Magento 2.4.8 Debugging Tools

| Debug Tool | Configuration | Log Location |
|------------|----------------|--------------|
| **Query Logging** | Set `query_log => 1` in env.php | `var/debug/db.log` |
| **Xdebug** | Configure php.ini with xdebug settings | N/A |
| **Profiler** | Enable `debug => 1` in env.php | `var/debug/debug.log` |
| **ES Debugging** | Configure hosts in env.php | ES logs |
| **Flat Catalog** | Configure indexer batch sizes | N/A |
| **db_schema** | Use correct XSD (`etcSchema.xsd`) | N/A |
| **New Relic** | Configure in env.php | New Relic dashboard |

### Key Corrections from Common Mistakes

1. **Query log path**: Use `var/debug/db.log` not `var/log/debug/db_repo.log`
2. **ES bulk size**: Use `50` not `1000` (dangerously high otherwise)
3. **Flat catalog**: Indexers use `batch_size` setting in env.php
4. **db_schema XSD**: Must be `urn:magento:framework:Setup/Declaration/Schema/etcSchema.xsd`

---

## References

- [Logging Configuration](https://developer.adobe.com/commerce/docs/cloud/configure/setup/standard-logging/)
- [Debug Tools](https://developer.adobe.com/commerce/php/development/debugging/)
- [Query Logging](https://developer.adobe.com/commerce/php/development/components/async-bulk-operations/)

(End of file - total 650 lines)