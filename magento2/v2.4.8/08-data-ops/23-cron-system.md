---
title: "23 - Cron System & Scheduled Tasks"
description: "Magento 2.4.8 cron architecture: crontab.xml, cron groups, schedule management, concurrency control, and programmatic cron job creation."
tags: magento2, cron, cron-group, crontab, scheduled-task, cron-schedule, cron-job, indexer-cron, cleaner
rank: 23
pathways: [magento2-deep-dive]
see_also:
  - path: "08-data-ops/16-message-queue.md"
    description: "Message Queue — async alternative to cron"
  - path: "09-performance/07-performance-profiling.md"
    description: "Performance — cron profiling and slow job detection"
---

# Cron System & Scheduled Tasks

Magento's cron system drives scheduled tasks: indexer updates, email sending, cache cleaning, report generation, and external integrations. Understanding the cron architecture is essential for debugging why scheduled tasks don't run, optimizing job performance, and creating reliable custom cron jobs.

---

## 1. Cron Architecture Overview

### How Magento Cron Works

```
System Cron (crontab)
    ↓ (every minute)
bin/magento cron:run
    ↓
Magento Cron Scheduler (reads crontab.xml)
    ↓
For each job due to run:
    - Check cron_schedule table
    - If not running and scheduled time passed → execute
    ↓
Job execution (PHP process)
    ↓
Update cron_schedule with result (success/error)
```

### The crontab.xml

```xml
<!-- app/code/Vendor/Module/etc/crontab.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Cron:etc/crontab.xsd">
    <group id="default">
        <job name="vendor_custom_cron" instance="Vendor\Module\Cron\CustomCron" method="execute">
            <schedule>*/5 * * * *</schedule>
        </job>
    </group>
</config>
```

### System Crontab Entry

The system-level crontab must be configured to run Magento's cron:

```cron
# /var/spool/cron/root (or www-data)
* * * * * /usr/bin/php /var/www/html/bin/magento cron:run -- Groups=default 2>&1 | grep -v "Ran job by scheduler" >> /var/log/magento.cron.log
```

---

## 2. Cron Groups

### What Are Cron Groups

Cron groups allow different sets of jobs to run on different schedules or with different settings. Default Magento groups:

| Group | Jobs | Schedule |
|-------|------|----------|
| `default` | General maintenance | Every minute |
| `index` | Indexer jobs | Per indexer config |
| `sales` | Order processing, invoice, shipment | Every minute |
| `custom` | (For extension use) | Configurable |

### Assigning Jobs to Groups

```xml
<!-- Group by group id -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Cron:etc/crontab.xsd">
    <group id="sales">
        <job name="process_order" instance="Vendor\Sales\Cron\ProcessOrder" method="execute">
            <schedule>0 * * * *</schedule>  <!-- Every hour -->
        </job>
    </group>
</config>
```

### Running Specific Groups

```bash
# Run only the sales group cron
bin/magento cron:run --Groups=sales

# Run only default group
bin/magento cron:run --Groups=default

# Run indexer cron only
bin/magento cron:run --Groups=index

# Run without groups (runs all)
bin/magento cron:run
```

---

## 3. Cron Schedule Management

### cron_schedule Table

```sql
CREATE TABLE cron_schedule (
    schedule_id      INT PRIMARY KEY AUTO_INCREMENT,
    job_code         VARCHAR(255) NOT NULL,     -- 'catalog_product_index_reindex'
    status           VARCHAR(50) NOT NULL,      -- 'pending', 'running', 'success', 'missed', 'error'
    messages         TEXT NULL,                  -- Error message if failed
    created_at      DATETIME NOT NULL,         -- When job was scheduled
    scheduled_at    DATETIME NOT NULL,         -- When to run
    executed_at     DATETIME NULL,             -- When actually started
    finished_at     DATETIME NULL,             -- When completed
    priority        INT DEFAULT 0
);
```

### Status Values

| Status | Meaning |
|--------|---------|
| `pending` | Scheduled, waiting to run |
| `running` | Currently executing |
| `success` | Completed successfully |
| `error` | Failed with error |
| `missed` | Scheduled time passed without execution |

### Cron Schedule Generation

```php
<?php
// \Magento\Cron\Model\Schedule::schedule()
// Called by \Magento\Cron\Observer\GenerateSchedules

public function schedule(string $jobCode, string $schedule, int $priority = 0): self
{
    $this->setJobCode($jobCode);
    $this->setStatus('pending');

    // Parse cron expression to next run time
    $nextRun = $this->cronExpressionResolver->getCronExpression($schedule)->getNextRunDate();

    $this->setScheduledAt($nextRun);
    $this->setPriority($priority);

    return $this;
}
```

---

## 4. Creating Custom Cron Jobs

### Step 1: Create Cron Class

```php
<?php
// Cron/CustomTask.php
declare(strict_types=1);

namespace Vendor\Module\Cron;

class CustomTask
{
    private LoggerInterface $logger;
    private ProductRepositoryInterface $productRepository;
    private State $indexerState;

    public function __construct(
        LoggerInterface $logger,
        ProductRepositoryInterface $productRepository,
        \Magento\Framework\Indexer\State $indexerState
    ) {
        $this->logger = $logger;
        $this->productRepository = $productRepository;
        $this->indexerState = $indexerState;
    }

    public function execute(): void
    {
        $this->logger->info('Running custom cron task');

        try {
            // Do work
            $this->processProducts();

            $this->logger->info('Custom cron task completed successfully');

        } catch (\Exception $e) {
            $this->logger->error('Custom cron task failed', [
                'error' => $e->getMessage(),
                'trace' => $e->getTraceAsString()
            ]);
            // Throw exception so cron marks job as 'error'
            throw $e;
        }
    }

    private function processProducts(): void
    {
        // Processing logic
        $product = $this->productRepository->get('SKU001');
        // ...
    }
}
```

### Step 2: Declare in crontab.xml

```xml
<!-- etc/crontab.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Cron:etc/crontab.xsd">
    <group id="default">
        <job name="vendor_custom_task"
             instance="Vendor\Module\Cron\CustomTask"
             method="execute">
            <schedule>0 2 * * *</schedule>  <!-- Daily at 2 AM -->
        </job>
    </group>
</config>
```

### Step 3: Configure Group (Optional)

```xml
<!-- etc/cron_groups.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Cron:etc/cron_groups.xsd">
    <group id="custom_group">
        <schedule_generate_every>1</schedule_generate_every>
        <schedule_ahead_for>4</schedule_ahead_for>
        <schedule_lifetime>2</schedule_lifetime>
        <history_cleanup_every>10</history_cleanup_every>
        <history_lifetime>1440</history_lifetime>
        <use_separate_process>0</use_separate_process>
    </group>
</config>
```

---

## 5. Cron Expression Format

### Standard Cron Format

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6) (Sunday=0)
│ │ │ │ │
* * * * *
```

### Common Examples

| Expression | Meaning |
|------------|---------|
| `* * * * *` | Every minute |
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour (at minute 0) |
| `0 2 * * *` | Daily at 2 AM |
| `0 3 * * 0` | Weekly on Sunday at 3 AM |
| `30 9 15 * *` | 15th of every month at 9:30 AM |
| `0 */4 * * *` | Every 4 hours |

### Config-Based Schedule

```xml
<!-- Can also pull schedule from system config -->
<job name="custom_task" instance="Vendor\Module\Cron\CustomTask" method="execute">
    <config_path>vendor/custom_cron/schedule</config_path>
</job>
```

In admin: `Stores → Settings → Configuration → Vendor → Custom Cron → Schedule`

---

## 6. Cron Groups Configuration

### Group Parameters

| Parameter | Meaning | Default |
|-----------|---------|---------|
| `schedule_generate_every` | How often (minutes) to generate schedule | 1 |
| `schedule_ahead_for` | How far ahead (minutes) to generate | 4 |
| `schedule_lifetime` | Max minutes a job can run before killed | 2 |
| `history_cleanup_every` | Minutes between cleanup runs | 10 |
| `history_lifetime` | Minutes to keep history records | 1440 (24h) |
| `use_separate_process` | Run jobs in separate PHP process | 0 |

### Using Separate Process

```xml
<!-- For heavy jobs, run in separate process -->
<group id="heavy_jobs">
    <use_separate_process>1</use_separate_process>
    <!-- other settings -->
</group>
```

---

## 7. Programmatic Cron Management

### Schedule a Cron Job Manually

```php
<?php
// Create a schedule entry manually
public function scheduleCustomJob(): void
{
    /** @var \Magento\Cron\Model\Schedule $schedule */
    $schedule = $this->scheduleFactory->create();
    $schedule->setJobCode('custom_email_batch');
    $schedule->setStatus('pending');
    $schedule->setScheduledAt(date('Y-m-d H:i:s', strtotime('+5 minutes')));

    $this->scheduleRepository->save($schedule);
}
```

### Dispatch Custom Event via Cron

```php
<?php
// For event-driven processing
public function execute(): void
{
    // Dispatch custom event for observers to handle
    $this->_eventManager->dispatch('custom_cron_batch_process', [
        'batch_id' => $this->batchId
    ]);
}
```

### Run Cron on Demand

```php
<?php
// Trigger a cron job manually (bypass schedule)
public function forceRunTask(string $jobCode): void
{
    /** @var \Magento\Cron\Model\Schedule $schedule */
    $schedule = $this->scheduleFactory->create();
    $schedule->setJobCode($jobCode);
    $schedule->setStatus('pending');
    $schedule->setScheduledAt(date('Y-m-d H:i:s'));

    $this->scheduleRepository->save($schedule);

    // Or run directly without scheduling
    $this->cronExecutor->execute($jobCode);
}
```

---

## 8. Cron and Indexers

### Indexer Cron Jobs

```xml
<!-- From MagentoIndexer/etc/crontab.xml -->
<config>
    <group id="index">
        <job name="catalog_product_index_reindex" instance="Magento\Indexer\Cron\ReindexAll" method="execute">
            <schedule>0 * * * *</schedule>
        </job>
        <job name="catalog_category_flat_reindex" instance="Magento\Catalog\Cron\ReindexFlatCategories" method="execute">
            <schedule>0 */6 * * *</schedule>
        </job>
    </group>
</config>
```

### Running Indexers via Cron

```php
<?php
// \Magento\Indexer\Cron\ReindexAll::execute()
public function execute(): void
{
    // Get list of indexers
    $indexers = $this->indexerCollection->getAllIndexes();

    foreach ($indexers as $indexer) {
        try {
            $indexer->reindexAll();
        } catch (\Exception $e) {
            $this->logger->error('Indexer reindex failed: ' . $indexer->getId(), [
                'error' => $e->getMessage()
            ]);
        }
    }
}
```

### Per-Indexer Schedule

Some indexers have individual cron configurations in admin:
- `Stores → Configuration → Catalog → Catalog → Product Indexers`
- `Stores → Configuration → Index Management`

---

## 9. Troubleshooting Cron Issues

### Cron Not Running

```bash
# Check if cron is set up in system crontab
crontab -l

# Or check Magento cron log
tail -f var/log/cron.log

# Run cron manually to see errors
bin/magento cron:run --verbose

# Check cron_schedule table
SELECT * FROM cron_schedule WHERE status IN ('error', 'missed') ORDER BY scheduled_at DESC LIMIT 10;
```

### Stuck Cron Jobs

```sql
-- Find running jobs that started long ago
SELECT * FROM cron_schedule
WHERE status = 'running'
AND executed_at < DATE_SUB(NOW(), INTERVAL 30 MINUTE);

-- Clear stuck jobs (set back to pending)
UPDATE cron_schedule SET status='pending', executed_at=NULL, finished_at=NULL
WHERE status='running' AND executed_at < DATE_SUB(NOW(), INTERVAL 1 HOUR);
```

### Missing Schedule Entries

```sql
-- Check if schedule entries are being generated
SELECT job_code, COUNT(*) as cnt
FROM cron_schedule
WHERE scheduled_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)
GROUP BY job_code;

-- If nothing generated:
-- 1. Check cron_schedule_generate_every config
-- 2. Check \Magento\Cron\Observer\GenerateSchedules observer
-- 3. Run: bin/magento cron:run to generate manually
```

### Cleanup Old Schedules

```sql
-- Manually clean old completed/crashed schedules
DELETE FROM cron_schedule
WHERE status IN ('success', 'error')
AND finished_at < DATE_SUB(NOW(), INTERVAL 7 DAY);

-- Or use Magento's cleanup
bin/magento cron:cleanup
```

---

## 10. Performance Optimization

### Use Separate Process for Heavy Jobs

```xml
<!-- For jobs that take long time -->
<group id="heavy">
    <use_separate_process>1</use_separate_process>
    <schedule_lifetime>60</schedule_lifetime>  <!-- Allow 60 minutes -->
</group>
```

### Batch Processing in Cron

```php
<?php
public function execute(): void
{
    $offset = 0;
    $batchSize = 100;

    while (true) {
        // Process batch
        $collection = $this->getBatch($offset, $batchSize);

        if ($collection->getSize() === 0) {
            break;  // Done
        }

        foreach ($collection as $item) {
            $this->processItem($item);
        }

        $offset += $batchSize;

        // Check time limit before next batch
        if ($this->isTimeLimitApproaching()) {
            break;  // Job will be rescheduled
        }
    }
}
```

### Schedule Optimization

```php
<?php
// Don't schedule too frequently
// BAD: */1 * * * * (every minute)
// GOOD: */15 * * * * (every 15 minutes)

if ($this->shouldRunNow()) {
    $this->doWork();
}

// Avoid overlapping: check if previous job still running
public function execute(): void
{
    $lockName = 'cron_vendor_task.lock';
    if (!$this->lockManager->acquire($lockName, 300)) {
        return;  // Previous job still running
    }

    try {
        $this->doWork();
    } finally {
        $this->lockManager->release($lockName);
    }
}
```

---

## Reading List

- [Configure cron jobs](https://experienceleague.adobe.com/docs/commerce-admin/config/config-cron.html)
- [Custom cron job](https://developer.adobe.com/commerce/php/development/components/cron/)
- [Cron reference](https://experienceleague.adobe.com/docs/commerce-admin/systems/cron.html)

---

## Common Mistakes to Avoid

1. ❌ Not setting up system crontab → Jobs never run
2. ❌ Too frequent schedules → Server overload
3. ❌ Long-running jobs without separate process → Block other jobs
4. ❌ Forgetting to clear cron_schedule → Table grows indefinitely
5. ❌ Not catching exceptions → Failed job status not recorded
6. ❌ Batch jobs without time limit check → Cron timeout kills them

---

*Magento 2 Backend Developer Course — Topic 08 — Data Operations*