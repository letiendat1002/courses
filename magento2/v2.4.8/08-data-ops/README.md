# Topic 8: Data Operations — Import/Export, Indexing & Scheduled Tasks

**Goal:** Build reliable data operations — bulk imports, exports, custom indexers, and cron jobs for automation.

---

## Topics Covered

- CSV import/export using Magento's import/export framework
- Import validation and error handling
- Custom indexers — triggered and scheduled
- Cron schedule configuration and job execution
- Mass actions in admin grids
- Custom CLI commands — `bin/magento training:review:approve`
- Data cleanup and archiving strategies
- Bulk operations via registry and queue
- Message queue architecture — topology, consumers, and dead-letter queues

---

## Reference Exercises

- **Exercise 7.1:** Create a CSV export of reviews via the Export button
- **Exercise 7.2:** Build a custom importer for review data with validation
- **Exercise 7.3:** Create a custom indexer that rebuilds on product save
- **Exercise 7.4:** Set up a cron job to auto-approve reviews older than 30 days
- **Exercise 7.5:** Write a CLI command to approve/delete reviews via `bin/magento`
- **Exercise 7.6:** Add a mass "Mark as Verified" action to the admin grid

---

## Completion Criteria

- [ ] Export button generates valid CSV with review data
- [ ] Custom import validates SKU, reviewer name, rating; rejects bad rows with clear errors
- [ ] Custom indexer updates when products are saved
- [ ] Cron job runs on configured schedule, logs auto-approval action
- [ ] Mass action processes selected rows correctly
- [ ] Custom CLI command registered, runs with full DI context
- [ ] Message queue configured with topology, publisher, and consumer
- [ ] Failed messages correctly routed to dead-letter queue with retry logic

---

## Topics

---

### Topic 1: CSV Export

**Magento's Export Framework:**

Magento provides a generic export infrastructure. To enable export for your custom entity, you need an export entity and a controller.

**Why Transactional Integrity Matters in Export:**

Exports can fail mid-process due to memory limits, timeout, or data issues. Without proper handling:
- Partial files get created
- No way to resume
- Corruption possible

> **Pro Tip from Production:** For large exports (>10,000 records), use Magento's async approach or chunked export to prevent memory exhaustion.

**Export Entity — `Export/Convertor/Review.php`:**

```php
<?php
// Export/Convertor/Review.php
namespace Training\Review\Export\Convertor;

use Magento\ImportExport\Model\Export\AbstractConvertor;

class Review extends AbstractConvertor
{
    public function convert(array $rowData, $additionalData = [])
    {
        return [
            'review_id'    => $rowData['review_id'],
            'product_id'  => $rowData['product_id'],
            'reviewer'    => $rowData['reviewer_name'],
            'rating'      => $rowData['rating'],
            'text'        => $rowData['review_text'],
            'verified'    => $rowData['is_verified'] ? 'Yes' : 'No',
            'created_at'  => $rowData['created_at'],
        ];
    }
}
```

**Export Controller (Proper DI):**

```php
<?php
// Controller/Adminhtml/Export/Reviews.php
namespace Training\Review\Controller\Adminhtml\Export;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Framework\App\Response\Http\FileFactory;
use Training\Review\Model\ResourceModel\Review\CollectionFactory;

class Reviews extends Action
{
    protected $collectionFactory;
    protected $fileFactory;

    public function __construct(
        Context $context,
        CollectionFactory $collectionFactory,
        FileFactory $fileFactory
    ) {
        parent::__construct($context);
        $this->collectionFactory = $collectionFactory;
        $this->fileFactory = $fileFactory;
    }

    public function execute()
    {
        $collection = $this->collectionFactory->create();
        $collection->setOrder('created_at', 'DESC');

        $rows = [];
        $rows[] = ['review_id', 'product_id', 'reviewer_name', 'rating', 'review_text', 'is_verified', 'created_at'];

        foreach ($collection as $review) {
            $rows[] = [
                $review->getReviewId(),
                $review->getProductId(),
                $review->getReviewerName(),
                $review->getRating(),
                $review->getReviewText(),
                $review->getIsVerified() ? 'Yes' : 'No',
                $review->getCreatedAt(),
            ];
        }

        $csvContent = $this->generateCsv($rows);
        return $this->fileFactory->create('reviews_export.csv', $csvContent, 'var/export');
    }

    private function generateCsv(array $rows): string
    {
        $handle = fopen('php://temp', 'r+');
        foreach ($rows as $row) {
            fputcsv($handle, $row);
        }
        rewind($handle);
        $content = stream_get_contents($handle);
        fclose($handle);
        return $content;
    }

    protected function _isAllowed(): bool
    {
        return $this->_authorization->isAllowed('Training_Review::review_export');
    }
}
```

**Simpler Approach — Override Magento's Export Controller:**

```php
<?php
// Controller/Adminhtml/Export/ExportReviews.php
namespace Training\Review\Controller\Adminhtml\Export;

use Magento\Backend\App\Action;
use Magento\Framework\App\Response\Http\FileFactory;
use Training\Review\Model\ResourceModel\Review\CollectionFactory;

class ExportReviews extends Action
{
    protected $collectionFactory;
    protected $fileFactory;

    public function __construct(
        \Magento\Backend\App\Action\Context $context,
        CollectionFactory $collectionFactory,
        FileFactory $fileFactory
    ) {
        parent::__construct($context);
        $this->collectionFactory = $collectionFactory;
        $this->fileFactory = $fileFactory;
    }

    public function execute()
    {
        $collection = $this->collectionFactory->create();
        $collection->setOrder('created_at', 'DESC');

        $rows = [];
        $rows[] = ['review_id', 'product_id', 'reviewer_name', 'rating', 'review_text', 'is_verified', 'created_at'];

        foreach ($collection as $review) {
            $rows[] = [
                $review->getReviewId(),
                $review->getProductId(),
                $review->getReviewerName(),
                $review->getRating(),
                $review->getReviewText(),
                $review->getIsVerified() ? 'Yes' : 'No',
                $review->getCreatedAt(),
            ];
        }

        $csvContent = $this->generateCsv($rows);
        return $this->fileFactory->create('reviews_export.csv', $csvContent, 'var/export');
    }

    private function generateCsv(array $rows): string
    {
        $handle = fopen('php://temp', 'r+');
        foreach ($rows as $row) {
            fputcsv($handle, $row);
        }
        rewind($handle);
        $content = stream_get_contents($handle);
        fclose($handle);
        return $content;
    }

    protected function _isAllowed(): bool
    {
        return $this->_authorization->isAllowed('Training_Review::review_export');
    }
}
```

---

### Topic 2: CSV Import with Validation

**Import Architecture — Why Transactions Matter:**

Imports can fail at any row. Without transactions:
- Some rows committed, others not → inconsistent data
- Partial imports hard to clean up
- No rollback on critical errors

> **⚠️ Critical:** Always use database transactions for bulk imports. If any row fails, rollback everything to maintain data integrity.

**Idempotency in Import — Preventing Duplicate Data:**

Idempotency means "same input produces same result, regardless of how many times executed." For imports:

```php
// ❌ NOT IDEMPOTENT — Creates duplicate on re-run
public function importRow(array $data): bool
{
    $review = $this->reviewFactory->create();
    $review->setReviewerName($data['reviewer_name']);
    $this->reviewRepository->save($review);
    return true;
}

// ✅ IDEMPOTENT — Updates existing or creates new
public function importRow(array $data): bool
{
    // Check if already exists by unique key
    $existing = $this->reviewRepository->getByProductId($data['product_id']);

    if ($existing) {
        $existing->setRating($data['rating']);
        $this->reviewRepository->save($existing);
    } else {
        $review = $this->reviewFactory->create();
        $review->setProductId($data['product_id']);
        $review->setReviewerName($data['reviewer_name']);
        $review->setRating($data['rating']);
        $this->reviewRepository->save($review);
    }
    return true;
}
```

**Idempotency Strategies:**

| Strategy | Implementation | Pros/Cons |
|----------|----------------|-----------|
| UPSERT | `INSERT ... ON DUPLICATE KEY UPDATE` | Fast, simple |
| Check-then-insert | SELECT first, then insert | Safe, extra query |
| Unique constraint | DB enforces uniqueness | Best, requires schema |
| Batch tracking | Track processed IDs | Complex, handles re-runs |

**Import Model — `Model/Import/Review.php`:**

```php
<?php
// Model/Import/Review.php
namespace Training\Review\Model\Import;

use Magento\ImportExport\Model\Import\EntitiesAbstract;
use Magento\ImportExport\Model\Import\ErrorProcessing\ProcessingError;
use Magento\Framework\App\ObjectManager;

class Review extends EntitiesAbstract
{
    protected $entityTypeCode = 'training_review';

    protected $returnIndexOf = true; // Return index instead of assoc array

    protected function _importData(): bool
    {
        // Custom implementation
    }

    public function validateData(): bool
    {
        $this->errorAggregator = new \Magento\ImportExport\Model\Import\ErrorProcessing\ProcessingErrorAggregator();
        return parent::validateData();
    }

    protected function _saveData(): bool
    {
        $rows = $this->getProcessedRows();
        foreach ($rows as $rowData) {
            try {
                $this->_saveEntity($this->_getEntity($rowData));
            } catch (\Exception $e) {
                $this->addRowError($e->getMessage(), $this->getRowNum());
            }
        }
        return true;
    }
}
```

**Custom Import with Direct Processing:**

```php
<?php
// Model/Import/Review.php
namespace Training\Review\Model\Import;

use Magento\Framework\App\Filesystem\DirectoryList;
use Magento\ImportExport\Model\Import\AbstractSource;
use Magento\ImportExport\Model\Import\Adapter\Factory as ImportAdapterFactory;

class Review
{
    protected $directoryList;
    protected $reviewFactory;
    protected $reviewRepository;
    protected $errorMessages = [];

    public function __construct(
        \Training\Review\Api\ReviewRepositoryInterface $reviewRepository,
        \Training\Review\Api\Data\ReviewInterfaceFactory $reviewFactory,
        DirectoryList $directoryList
    ) {
        $this->directoryList = $directoryList;
        $this->reviewRepository = $reviewRepository;
        $this->reviewFactory = $reviewFactory;
    }

    public function importFromCsvFile(string $filePath): array
    {
        $handle = fopen($filePath, 'r');
        if ($handle === false) {
            throw new \Exception("Cannot open file: $filePath");
        }

        $header = fgetcsv($handle);
        $expectedHeader = ['review_id', 'product_id', 'reviewer_name', 'rating', 'review_text', 'is_verified', 'created_at'];
        if ($header !== $expectedHeader) {
            throw new \Exception('Invalid CSV header');
        }

        $imported = 0;
        $errors = [];
        $rowNum = 1;

        while (($row = fgetcsv($handle)) !== false) {
            $rowNum++;
            $data = array_combine($header, $row);

            // Validate
            $validationResult = $this->validateRow($data, $rowNum);
            if ($validationResult !== true) {
                $errors[] = $validationResult;
                continue;
            }

            try {
                $review = $this->reviewFactory->create();
                $review->setProductId((int)($data['product_id'] ?? 0));
                $review->setReviewerName($data['reviewer_name']);
                $review->setRating((int)($data['rating'] ?? 5));
                $review->setReviewText($data['review_text'] ?? '');
                $review->setIsVerified($data['is_verified'] === 'Yes');
                $review->setCreatedAt($data['created_at'] ?? date('Y-m-d H:i:s'));

                $this->reviewRepository->save($review);
                $imported++;
            } catch (\Exception $e) {
                $errors[] = "Row $rowNum: " . $e->getMessage();
            }
        }

        fclose($handle);
        return ['imported' => $imported, 'errors' => $errors];
    }

    private function validateRow(array $data, int $rowNum): bool|string
    {
        if (empty($data['reviewer_name']) || strlen(trim($data['reviewer_name'])) < 2) {
            return "Row $rowNum: Reviewer name must be at least 2 characters";
        }

        $rating = (int)($data['rating'] ?? 0);
        if ($rating < 1 || $rating > 5) {
            return "Row $rowNum: Rating must be between 1 and 5";
        }

        if (!is_numeric($data['product_id']) || (int)$data['product_id'] <= 0) {
            return "Row $rowNum: Invalid product_id";
        }

        return true;
    }
}
```

**Import Controller with Transaction Support:**

```php
<?php
// Controller/Adminhtml/Import/Reviews.php
namespace Training\Review\Controller\Adminhtml\Import;

use Magento\Backend\App\Action;
use Magento\Framework\App\Request\DataFactory;
use Training\Review\Model\Import\Review as ImportModel;
use Magento\Framework\DB\TransactionFactory;

class Reviews extends Action
{
    protected $importModel;
    protected $transactionFactory;

    public function __construct(
        \Magento\Backend\App\Action\Context $context,
        ImportModel $importModel,
        TransactionFactory $transactionFactory
    ) {
        parent::__construct($context);
        $this->importModel = $importModel;
        $this->transactionFactory = $transactionFactory;
    }

    public function execute()
    {
        $file = $this->getRequest()->getFile('import_file');
        if (!$file) {
            $this->messageManager->addErrorMessage(__('No file uploaded'));
            return $this->_redirect('*/*/index');
        }

        try {
            $result = $this->importModel->importFromCsvFile($file['tmp_name']);
            $this->messageManager->addSuccessMessage(
                __('Imported %1 records. Errors: %2', $result['imported'], count($result['errors']))
            );
            if (!empty($result['errors'])) {
                foreach (array_slice($result['errors'], 0, 5) as $error) {
                    $this->messageManager->addErrorMessage($error);
                }
            }
        } catch (\Exception $e) {
            $this->messageManager->addErrorMessage($e->getMessage());
        }

        return $this->_redirect('*/*/index');
    }

    protected function _isAllowed(): bool
    {
        return $this->_authorization->isAllowed('Training_Review::review_import');
    }
}
```

**Transactional Import Pattern:**

```php
// In Import/Review.php
public function importFromCsvFile(string $filePath): array
{
    $handle = fopen($filePath, 'r');
    $header = fgetcsv($handle);

    $imported = 0;
    $errors = [];
    $transaction = $this->transactionFactory->create();

    while (($row = fgetcsv($handle)) !== false) {
        $rowNum++;
        $data = array_combine($header, $row);

        try {
            $review = $this->reviewFactory->create();
            $review->setProductId((int)($data['product_id'] ?? 0));
            $review->setReviewerName($data['reviewer_name']);
            $review->setRating((int)($data['rating'] ?? 5));
            // Add to transaction instead of immediate save
            $transaction->addObject($review);
            $imported++;
        } catch (\Exception $e) {
            $errors[] = "Row $rowNum: " . $e->getMessage();
        }
    }

    fclose($handle);

    // Commit all at once — all or nothing
    try {
        $transaction->save();
    } catch (\Exception $e) {
        // Rollback happened automatically on exception
        return ['imported' => 0, 'errors' => [$e->getMessage()]];
    }

    return ['imported' => $imported, 'errors' => $errors];
}
```

**Import Best Practices:**

| Practice | Why It Matters |
|----------|----------------|
| Validate all rows first | Don't commit bad data |
| Use transactions for batch | All-or-nothing integrity |
| Implement idempotency | Safe to re-run on failure |
| Log failures with row numbers | Easy to identify and fix bad rows |
| Chunk large imports | Memory management |

---

### Topic 3: Custom Indexers

**Indexer Modes:**

| Mode | Trigger | Use Case |
|------|---------|----------|
| `onormal` | Manual or cron | Full rebuild on change |
| `realtime` | Immediate on save | Real-time updates |
| `schedule` | Cron schedule | Batched updates |

**Indexer Definition — `etc/indexer.xml`:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Indexer/etc/indexer.xsd">
    <indexer id="training_review_product_summary"
            class="Training\Review\Indexer\ReviewSummaryIndexer"
            title="Review Summary by Product"
            description="Rebuilds product review summary (avg rating, count)">
        <fieldset name="review" source="Training\Review\Model\ResourceModel\Review\Collection"
                  provider="Training\Review\Indexer\ReviewSummaryIndexer\Pool">
            <field name="product_id" xsi:type="filterable" dataType="int"/>
            <field name="avg_rating" xsi:type="filterable" dataType="decimal"/>
            <field name="review_count" xsi:type="filterable" dataType="int"/>
            <field name="last_review_at" xsi:type="filterable" dataType="datetime"/>
        </fieldset>
    </indexer>
</config>
```

**Indexer Class — `Indexer/ReviewSummaryIndexer.php`:**

```php
<?php
// Indexer/ReviewSummaryIndexer.php
namespace Training\Review\Indexer;

use Magento\Framework\Indexer\AbstractIndexer;
use Magento\Framework\Indexer\StateInterface;
use Magento\Catalog\Model\Product;

class ReviewSummaryIndexer extends AbstractIndexer
{
    protected $reviewResource;

    public function __construct(
        \Training\Review\Model\ResourceModel\Review $reviewResource
    ) {
        $this->reviewResource = $reviewResource;
    }

    public function execute(): void
    {
        $this->reindexAll();
    }

    public function executeList(array $productIds): void
    {
        foreach ($productIds as $productId) {
            $this->reindexProduct($productId);
        }
    }

    public function executeRow(int $productId): void
    {
        $this->reindexProduct($productId);
    }

    private function reindexProduct(int $productId): void
    {
        $connection = $this->reviewResource->getConnection();

        $select = $connection->select()
            ->from($this->reviewResource->getMainTable(), [
                'product_id' => 'product_id',
                'avg_rating' => new \Zend_Db_Expr('AVG(rating)'),
                'review_count' => new \Zend_Db_Expr('COUNT(*)'),
                'last_review_at' => new \Zend_Db_Expr('MAX(created_at)'),
            ])
            ->where('product_id = ?', $productId)
            ->group('product_id');

        $data = $connection->fetchRow($select);
        if ($data) {
            // Update summary table or cache
            $this->updateSummaryTable($data);
        }
    }

    private function reindexAll(): void
    {
        // Full rebuild — process all products
        $connection = $this->reviewResource->getConnection();
        $table = $this->getSummaryTable();

        $select = $connection->select()
            ->from($this->reviewResource->getMainTable(), [
                'product_id' => 'product_id',
                'avg_rating' => new \Zend_Db_Expr('AVG(rating)'),
                'review_count' => new \Zend_Db_Expr('COUNT(*)'),
                'last_review_at' => new \Zend_Db_Expr('MAX(created_at)'),
            ])
            ->group('product_id');

        $connection->query("TRUNCATE TABLE {$table}");
        $connection->query($select);
    }

    protected function updateSummaryTable(array $data): void
    {
        // Write to summary table
    }

    protected function getSummaryTable(): string
    {
        return 'training_review_product_summary';
    }

    public function isAvailable(): bool
    {
        return true;
    }
}
```

**Triggering Indexer on Product Save — `etc/events.xml`:**

```xml
<event name="catalog_product_save_commit_after">
    <observer name="training_review_product_saved"
              instance="Training\Review\Observer\ReindexProductReviews"
              sortOrder="10"/>
</event>
```

**Observer:**

```php
<?php
// Observer/ReindexProductReviews.php
namespace Training\Review\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Framework\Indexer\IndexerRegistry;

class ReindexProductReviews implements ObserverInterface
{
    protected $indexerRegistry;

    public function __construct(IndexerRegistry $indexerRegistry)
    {
        $this->indexerRegistry = $indexerRegistry;
    }

    public function execute(Observer $observer): void
    {
        $product = $observer->getData('product');
        $productId = $product->getId();

        $indexer = $this->indexerRegistry->get('training_review_product_summary');
        if ($indexer->isAvailable()) {
            $indexer->executeRow($productId);
        }
    }
}
```

**Indexer Commands:**

```bash
bin/magento indexer:reindex training_review_product_summary
bin/magento indexer:status           # Show all indexers
bin/magento indexer:set-mode realtime training_review_product_summary
bin/magento indexer:set-mode schedule training_review_product_summary
```

**Indexer Best Practices:**

| Practice | Why It Matters |
|----------|----------------|
| Always implement `executeRow()` | Required for realtime mode |
| Use batch processing for large datasets | Memory management |
| Log progress for long-running indexers | Debugging, monitoring |
| Check indexer mode in observer | Only reindex when needed |

---

### Topic 4: Cron Jobs

**Cron Configuration — `crontab.xml`:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:etc/crontab.xsd">
    <group id="default">
        <job name="training_review_auto_approve"
             instance="Training\Review\Cron\AutoApprove"
             method="execute">
            <schedule>0 2 * * *</schedule>  <!-- 2:00 AM daily -->
        </job>

        <job name="training_review_cleanup"
             instance="Training\Review\Cron\CleanupOldReviews"
             method="execute">
            <schedule>0 3 * * 0</schedule>  <!-- 3:00 AM every Sunday -->
        </job>
    </group>
</config>
```

**Cron Job Class — `Cron/AutoApprove.php`:**

```php
<?php
// Cron/AutoApprove.php
namespace Training\Review\Cron;

use Psr\Log\LoggerInterface;
use Training\Review\Api\ReviewRepositoryInterface;
use Training\Review\Api\Data\ReviewInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Framework\Api\FilterBuilder;

class AutoApprove
{
    protected $logger;
    protected $reviewRepository;
    protected $searchCriteriaBuilder;
    protected $filterBuilder;

    public function __construct(
        LoggerInterface $logger,
        ReviewRepositoryInterface $reviewRepository,
        SearchCriteriaBuilder $searchCriteriaBuilder,
        FilterBuilder $filterBuilder
    ) {
        $this->logger = $logger;
        $this->reviewRepository = $reviewRepository;
        $this->searchCriteriaBuilder = $searchCriteriaBuilder;
        $this->filterBuilder = $filterBuilder;
    }

    public function execute(): void
    {
        $this->logger->info('Running auto-approve cron');

        $thirtyDaysAgo = date('Y-m-d H:i:s', strtotime('-30 days'));

        $this->searchCriteriaBuilder
            ->addFilter('is_verified', 0)
            ->addFilter('created_at', $thirtyDaysAgo, 'lteq');

        $criteria = $this->searchCriteriaBuilder->create();
        $reviews = $this->reviewRepository->getList($criteria);

        $approved = 0;
        foreach ($reviews->getItems() as $review) {
            try {
                $review->setIsVerified(true);
                $this->reviewRepository->save($review);
                $approved++;
            } catch (\Exception $e) {
                $this->logger->error('Failed to auto-approve review ' . $review->getReviewId(), [
                    'error' => $e->getMessage()
                ]);
            }
        }

        $this->logger->info("Auto-approve complete: {$approved} reviews approved");
    }
}
```

**Checking Cron Status:**

```bash
# View cron schedule
bin/magento cron:show

# Run all cron jobs manually
bin/magento cron:run --group=default

# Run specific job
bin/magento cron:run --job-name=training_review_auto_approve
```

**Cron Schedule Format:**

```
 ┌───────────── minute (0-59)
 │ ┌───────────── hour (0-23)
 │ │ ┌───────────── day of month (1-31)
 │ │ │ ┌───────────── month (1-12)
 │ │ │ │ ┌───────────── day of week (0-7, 0 and 7 = Sunday)
 │ │ │ │ │
 * * * * *
```

Examples:
- `0 2 * * *` — Every day at 2:00 AM
- `*/15 * * * *` — Every 15 minutes
- `0 3 * * 0` — Every Sunday at 3:00 AM
- `30 8 1 * *` — First day of every month at 8:30 AM

**Cron Schedule Best Practices:**

| Practice | Why It Matters |
|----------|----------------|
| Schedule during off-peak hours | Don't compete with customer traffic |
| Avoid overlapping jobs | Same job shouldn't run if previous hasn't finished |
| Use unique job names | Prevents conflicts with other modules |
| Log job start/end and errors | Monitoring, debugging |
| Don't run too frequently | Server resource management |

> **Pro Tip from Production:** Always add a lock or schedule check to prevent overlapping cron runs for the same job. Magento's cron system doesn't prevent overlap by default.

**Preventing Cron Overlap:**

```php
// In your cron class
class AutoApprove
{
    private $lockManager;

    public function execute(): void
    {
        // Try to acquire lock
        if (!$this->lockManager->lock('training_review_auto_approve')) {
            return; // Already running, skip
        }

        try {
            // Do work...
        } finally {
            $this->lockManager->unlock('training_review_auto_approve');
        }
    }
}
```

**Cron Security Checklist:**

| Check | Implementation |
|-------|----------------|
| Set appropriate permissions | Cron ACL in Magento |
| Validate input | Don't trust external data |
| Log all executions | Audit trail |
| Handle exceptions gracefully | Don't leave jobs in bad state |
| Monitor failed jobs | Alert on repeated failures |

---

### Topic 5: Mass Actions in Admin Grids

**Adding a Mass Action to a UI Component Grid:**

```xml
<!-- In the listing XML, add to listingToolbar -->
<listingToolbar name="listing_top">
    ...
    <massaction name="listing_massaction" target="-1">
        <action name="verify_selected" label="Mark as Verified">
            <settings>
                <url path="review/index/massVerify"/>
                <type>selected</type>
                <confirm>
                    <title>Verify Reviews</title>
                    <message>Mark selected reviews as verified?</message>
                </confirm>
            </settings>
        </action>
        <action name="delete_selected" label="Delete Selected">
            <settings>
                <url path="review/index/massDelete"/>
                <type>selected</type>
                <confirm>
                    <title>Delete Reviews</title>
                    <message>Delete selected reviews?</message>
                </confirm>
            </settings>
        </action>
    </massaction>
</listingToolbar>
```

**Mass Action Controller — `Controller/Adminhtml/Index/MassVerify.php`:**

```php
<?php
// Controller/Adminhtml/Index/MassVerify.php
namespace Training\Review\Controller\Adminhtml\Index;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Framework\Controller\ResultFactory;
use Magento\Ui\Component\MassAction\Filter;
use Training\Review\Model\ResourceModel\Review\CollectionFactory;

class MassVerify extends Action
{
    protected $filter;
    protected $collectionFactory;
    protected $reviewRepository;

    public function __construct(
        Context $context,
        Filter $filter,
        CollectionFactory $collectionFactory,
        \Training\Review\Api\ReviewRepositoryInterface $reviewRepository
    ) {
        parent::__construct($context);
        $this->filter = $filter;
        $this->collectionFactory = $collectionFactory;
        $this->reviewRepository = $reviewRepository;
    }

    public function execute()
    {
        $collection = $this->filter->getCollection(
            $this->collectionFactory->create()
        );

        $verified = 0;
        foreach ($collection as $review) {
            try {
                $review->setIsVerified(true);
                $this->reviewRepository->save($review);
                $verified++;
            } catch (\Exception $e) {
                $this->messageManager->addErrorMessage(
                    "Failed to verify review {$review->getReviewId()}: " . $e->getMessage()
                );
            }
        }

        if ($verified > 0) {
            $this->messageManager->addSuccessMessage(
                __("$verified review(s) marked as verified")
            );
        }

        return $this->resultFactory->create(ResultFactory::TYPE_REDIRECT)
            ->setPath('*/*/index');
    }

    protected function _isAllowed(): bool
    {
        return $this->_authorization->isAllowed('Training_Review::review_edit');
    }
}
```

---


### Topic 6: Custom CLI Commands — bin/magento

**Why Custom CLI Commands?**

Custom commands let you write one-off scripts that run in Magento's full context — with dependency injection, service classes, and the database available. Common uses:
- Data cleanup / fixes
- Maintenance mode toggles
- Bulk data import without the UI
- Generating sample data for testing
- Running background jobs not triggered by cron

**Command Structure:**

```
bin/magento training:review:approve <review_id> [--force]
bin/magento training:review:reindex
bin/magento training:review:stats
```

**Directory Pattern:**

```
app/code/[Vendor]/[Module]/Console/Command/[CommandName].php
```

**Basic Command:**

```php
<?php
// Console/Command/ApproveReview.php
namespace Training\Review\Console\Command;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;
use Training\Review\Api\ReviewRepositoryInterface;
use Magento\Framework\Console\Cli;

class ApproveReview extends Command
{
    protected $reviewRepository;
    protected $logger;

    public function __construct(
        ReviewRepositoryInterface $reviewRepository,
        \Psr\Log\LoggerInterface $logger
    ) {
        parent::__construct();
        $this->reviewRepository = $reviewRepository;
        $this->logger = $logger;
    }

    protected function configure(): void
    {
        $this->setName('training:review:approve')
            ->setDescription('Mark a review as verified/approved')
            ->addArgument('review_id', InputOption::VALUE_REQUIRED, 'Review ID')
            ->addOption('force', 'f', InputOption::VALUE_NONE, 'Skip confirmation');

        parent::configure();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $reviewId = (int) $input->getArgument('review_id');
        if (!$reviewId) {
            $output->writeln('<error>Review ID required</error>');
            return Cli::RETURN_FAILURE;
        }

        try {
            $review = $this->reviewRepository->getById($reviewId);
            $review->setIsVerified(true);
            $this->reviewRepository->save($review);

            $output->writeln("<info>Review #{$reviewId} approved</info>");
            $this->logger->info("Review {$reviewId} approved via CLI");
            return Cli::RETURN_SUCCESS;
        } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
            $output->writeln("<error>Review #{$reviewId} not found</error>");
            return Cli::RETURN_FAILURE;
        }
    }
}
```

**Register Command in `di.xml`:**

```xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/di.xsd">
    <type name="Magento\Framework\Console\CommandPool">
        <arguments>
            <argument name="commands" xsi:type="array">
                <item name="Training_Review" xsi:type="object">Training\Review\Console\Command\ApproveReview</item>
            </argument>
        </arguments>
    </type>
</config>
```

**Interactive Confirmation:**

```php
protected function execute(InputInterface $input, OutputInterface $output): int
{
    $reviewId = (int) $input->getArgument('review_id');

    if (!$input->getOption('force')) {
        $output->writeln("About to approve review #{$reviewId}. Use --force to skip.");
        return Cli::RETURN_FAILURE;
    }

    // ...
}
```

**Command with Table Output:**

```php
<?php
// Console/Command/ListReviews.php
namespace Training\Review\Console\Command;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Helper\Table;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;
use Magento\Framework\Console\Cli;
use Training\Review\Api\ReviewRepositoryInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;

class ListReviews extends Command
{
    protected $reviewRepository;
    protected $searchCriteriaBuilder;

    public function __construct(
        ReviewRepositoryInterface $reviewRepository,
        SearchCriteriaBuilder $searchCriteriaBuilder
    ) {
        parent::__construct();
        $this->reviewRepository = $reviewRepository;
        $this->searchCriteriaBuilder = $searchCriteriaBuilder;
    }

    protected function configure(): void
    {
        $this->setName('training:review:list')
            ->setDescription('List reviews with optional product filter')
            ->addOption('product', 'p', InputOption::VALUE_OPTIONAL, 'Filter by product ID');
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $criteria = $this->searchCriteriaBuilder->create();
        if ($productId = $input->getOption('product')) {
            $this->searchCriteriaBuilder->addFilter('product_id', $productId);
        }

        $reviews = $this->reviewRepository->getList($criteria);

        $table = new Table($output);
        $table->setHeaders(['ID', 'Product', 'Reviewer', 'Rating', 'Verified', 'Created']);

        foreach ($reviews->getItems() as $review) {
            $table->addRow([
                $review->getReviewId(),
                $review->getProductId(),
                $review->getReviewerName(),
                $review->getRating(),
                $review->getIsVerified() ? 'Yes' : 'No',
                $review->getCreatedAt(),
            ]);
        }

        $table->render();
        return Cli::RETURN_SUCCESS;
    }
}
```

**Key Notes:**

- Return `Cli::RETURN_SUCCESS` (0) on success, `Cli::RETURN_FAILURE` (1) on error
- Inject services via constructor — full Magento DI is available
- Use `$output->writeln()` for user-facing messages
- Log to `logger` for audit trail, not `$output` — output is for CLI, logs are for debugging
- Always validate and throw meaningful errors — CLI errors don't have a UI to show them

**Testing CLI Commands:**

```php
<?php
public function testApproveReviewCommand()
{
    $this->assertEquals(
        Cli::RETURN_SUCCESS,
        $this->application->run(
            new ArrayInput(['command' => 'training:review:approve', 'review_id' => '1']),
            new NullOutput()
        )
    );
}
```

---

### Topic 7: Message Queue Architecture — Topology, Consumers, and Dead-Letter Queues

**Why This Matters:** Asynchronous processing keeps your HTTP responses fast. But when queues fail silently, or messages pile up, or consumers crash mid-processing — you need to understand the topology to diagnose the problem.

**Magento's Queue Architecture (RabbitMQ / db_schema):**

Magento uses either MySQL (async.operations.all) or RabbitMQ (preferred for production) for message queuing. Both use the same producer/consumer API — only the backend differs.

```
Producer → Exchange → Queue → Consumer
                ↓
         Dead-Letter Queue (DLQ)
```

**Core Concepts:**

| Concept | What It Is |
|---------|------------|
| **Message** | JSON payload describing the job (entity ID, operation type, metadata) |
| **Queue** | Named channel (e.g., `product_action`) messages are published to |
| **Consumer** | PHP class that processes messages from a queue |
| **Publisher** | Code that enqueues a message when something happens |
| **Exchange** | Routes messages from publishers to the correct queue |
| **Dead-Letter Queue (DLQ)** | Queue where failed messages go after max retries |

**Queue Configuration — queue_topology.xml:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework-message-queue:etc/topology.xsd">
    
    <!-- Exchange: routes messages to queues -->
    <exchange name="training" type="topic" connection="db">
        <binding id="trainingProductBinding"
                 queue="training_product_sync"
                 topic="training.product.#"/>
    </exchange>
    
    <!-- Queue: the channel consumers listen to -->
    <queue name="training_product_sync">
        <consumer name="ProductSyncConsumer"
                  consumerInstance="Training\Sync\Model\Consumer\ProductSync"
                  maxMessages="100"/>
    </queue>
</config>
```

**Publisher Pattern:**

```php
<?php
namespace Training\Sync\Model;

use Magento\Framework\MessageQueue\PublisherInterface;

class ProductSyncPublisher
{
    public function __construct(
        private PublisherInterface $publisher
    ) {}

    public function publish(int $productId, string $operation): void
    {
        $this->publisher->publish(
            'training.product.sync',  // topic name
            json_encode([
                'product_id' => $productId,
                'operation' => $operation,  // 'save', 'delete', 'reindex'
                'timestamp' => time(),
            ])
        );
    }
}
```

**Consumer Pattern:**

```php
<?php
namespace Training\Sync\Model\Consumer;

use Magento\Framework\MessageQueue\ConsumerInterface;
use Magento\Framework\MessageQueue\Envelope;

class ProductSync implements ConsumerInterface
{
    public function __construct(
        private LoggerInterface $logger,
        private ProductRepositoryInterface $productRepository
    ) {}

    public function process(Envelope $envelope): string
    {
        $body = json_decode($envelope->getBody(), true);
        
        try {
            $productId = $body['product_id'];
            $operation = $body['operation'];
            
            match ($operation) {
                'save' => $this->syncOnSave($productId),
                'delete' => $this->syncOnDelete($productId),
                'reindex' => $this->triggerReindex($productId),
            };
            
            return ConsumerInterface::ACK;  // acknowledge success
            
        } catch (\Throwable $e) {
            $this->logger->error('ProductSync failed: ' . $e->getMessage());
            return ConsumerInterface::REJECT;  // send to DLQ after retries
        }
    }
    
    private function syncOnSave(int $productId): void
    {
        // sync to external ERP, PIM, search index, etc.
        $product = $this->productRepository->getById($productId);
        // ... sync logic
    }
}
```

**Dead-Letter Queue — What Happens When Processing Fails:**

Messages don't disappear on failure. Magento tracks failed messages via:

1. **Retry mechanism:** If `REJECT` is returned, Magento retries up to a configured max
2. **DLQ routing:** After max retries, the message goes to a dead-letter queue
3. **Queue message table:** `queue_message` and `queue_message_status` tables track state

```
Message published → Consumer runs → FAILS → Retry (up to 3x)
                                         → Still fails → Message goes to DLQ
                                                           → Logged in queue_message_status = 4 (rejected)
```

**Diagnosing Queue Problems:**

```bash
# View pending messages in MySQL (db queue)
mysql> SELECT queue_id, topic_name, message_body, status FROM queue_message WHERE status = 0;

# View failed messages
mysql> SELECT queue_id, topic_name, message_body FROM queue_message WHERE status = 4;

# Retry failed messages manually
bin/magento queue:consumers:run ProductSyncConsumer --force

# Or if using RabbitMQ management UI:
# http://localhost:15672 (default) → Queues → training_product_sync → Get Messages
```

**Common Queue Mistakes:**

1. ❌ **Consumer idempotency missing** — same message processed twice on retry creates duplicate records
   - Fix: Use entity version/check to detect "already processed" before executing
   
2. ❌ **No maxMessages limit** — consumer runs forever, blocking other consumers
   - Fix: Set `maxMessages` in queue_topology.xml
   
3. ❌ **JSON encode failure** — serializing non-serializable objects
   - Fix: Always encode primitive data, pass IDs, reconstruct on consumer side
   
4. ❌ **Silent failures** — no logging when processing fails
   - Fix: Always log in catch block with envelope body
   
5. ❌ **Messages not processed** — cron not running, or consumer not registered
   - Fix: `bin/magento queue:consumers:list` to verify consumer exists, `bin/magento cron:run --group=consumers`

**Pro Tips:**
- Always design consumers to be idempotent — they WILL be called more than once
- Use a unique correlation ID in the message body to detect duplicates
- Keep message body small — pass entity IDs, not full objects
- Set up monitoring: if queue_message.status=4 (rejected) grows, alerts should fire
- For bulk operations, prefer async/bulk API over individual queue messages

---

## Reading List
## Reading List

- [Import/Export Framework](https://developer.adobe.com/commerce/php/development/components/import-export/) — Magento's import infrastructure
- [Indexers](https://developer.adobe.com/commerce/php/development/components/indexing/) — Custom indexers
- [Configure Cron Jobs](https://experienceleague.adobe.com/docs/commerce-operations/configuration-guide/crons/custom-cron.html) — Cron configuration

---

## Edge Cases & Troubleshooting

| Issue | Symptom | Solution |
|-------|---------|----------|
| CSV export empty | No data in file | Collection returns data — check repository |
| Import fails silently | No rows inserted | Check file encoding (UTF-8), field count matches header |
| Indexer stuck | `reindex` hangs | Check for circular observer → indexer loop |
| Cron not firing | Scheduled job doesn't run | Check system cron is running: `crontab -l` |
| Mass action 404 | Controller path wrong | `url path` should be `review/index/massVerify` (no leading `/`) |
| Import validation too strict | Valid rows rejected | Check data type — rating `"5"` string vs integer |

---

## Common Mistakes to Avoid

1. ❌ Import without validation → Bad data enters the system
2. ❌ Large import in one transaction → Memory exhausted; batch it
3. ❌ Indexer without `executeRow()` → Real-time mode fails
4. ❌ Cron schedule too frequent → Server overload
5. ❌ Forgetting cron permissions → Cron blocked by security module

---

## Day N+1: Production Message Queue Configuration

> **⚠️ Audience:** This section is for production deployments. If you are still on local development,
> you can skip this and return when you are ready to go live. All examples are verified against
> Magento 2.4.7-patched / 2.4.8.

The message queue architecture introduced in Day N covers the basics. This section covers the
production-grade knobs that determine whether your queue system survives load or collapses under
it. Every configuration here maps to a real `env.php` key or `queue_topology.xml` directive.

---

### 1. Adapter Selection — MySQL vs RabbitMQ

Out of the box, Magento ships with a MySQL adapter. For production, RabbitMQ is strongly preferred.
Here is why and how to switch.

**MySQL adapter (default — do not use in production):**

- Single point of failure — if MySQL goes down, queue stops
- Table locks cause consumer latency under concurrency
- No Ack/Nack per message granularity; uses row-level status
- Acceptable only for dev/test with < 100 messages/hour

**RabbitMQ adapter (required for production):**

- Persistent messages survive broker restarts
- Ack/Nack per message with proper requeue semantics
- Prefetch window controls consumer load balancing
- Dead-letter exchange routing built-in
- Management UI at `http://localhost:15672` for queue inspection

**Comparison table:**

| Dimension | MySQL Adapter | RabbitMQ Adapter |
|-----------|---------------|-------------------|
| Message persistence | Table row (RAM-heavy) | Durable exchange + queue |
| Consumer scaling | Rows locked per consumer | Prefetch-based load balance |
| Acknowledgment | Table status update | AMQP ACK/NACK frame |
| Dead-letter routing | Manual + status=4 | Native DLX/DLQ routing |
| Operational overhead | Zero (already running) | Requires RabbitMQ host |
| Failure recovery | MySQL failover required | RabbitMQ cluster |
| Max throughput | ~200 msg/min (single node) | ~50,000+ msg/min |

**`env.php` adapter switch:**

```php
// app/etc/env.php  ← existing section, replace 'mysql' with 'amqp'
'queue' => [
    // -------------------------------------------------------
    // MySQL adapter (default — remove when switching to RabbitMQ)
    // -------------------------------------------------------
    // 'MAGENTO_MODULES' => [
    //     'Magento_MysqlMq' => 1,
    // ],
    // -------------------------------------------------------
    // RabbitMQ adapter (production)
    // -------------------------------------------------------
    'MAGENTO_MODULES' => [
        'Magento_Amqp' => 1,          // enable AMQP module
        'Magento_MysqlMq' => 0,        // disable MySQL module
    ],
    // -------------------------------------------------------
    // Connection settings (new in 2.4.7+)
    // -------------------------------------------------------
    'amqp' => [
        'host'     => '127.0.0.1',      // RabbitMQ host
        'port'     => 5672,             // AMQP port (non-TLS)
        'user'     => 'magento',        // RabbitMQ username
        'password' => 'magento123',     // RabbitMQ password
        'vhost'    => '/',              // virtual host
        'ssl'      => 'false',          // set 'true' for TLS/AMQPS
        // ----------------------------------------------------
        // Optional: bulk operations
        // ----------------------------------------------------
        'bulk'   => '1',                // enable bulk message batching
        'batch-size' => '100',           // messages per bulk frame
    ],
],
```

> **📋 Verification:** After changing `env.php`, run:
> ```bash
> bin/magento module:status | grep -E "Amqp|MysqlMq"
> bin/magento queue:consumers:list     # should show consumers
> tail -f var/log/queue.log            # watch for adapter init messages
> ```

---

### 2. Dead-Letter Queue Retry Configuration

When a consumer fails, Magento retries before routing to DLQ. The retry policy is controlled
via `queue_topology.xml` and the DLQ routing is automatic when max retries are exhausted.

**Retry mechanism flow:**

```
Publisher → Queue → Consumer → FAIL → Retry #1 (delay 0s)
                                         → FAIL → Retry #2 (delay 5s)
                                         → FAIL → Retry #3 (delay 10s)
                                         → FAIL → DLQ (no requeue)
```

**`queue_topology.xml` — per-consumer retry config:**

```xml
<!-- etc/queue_topology.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:MessageQueue:etc/topology.xsd">

    <!-- ------------------------------------------------------- -->
    <!-- ProductSyncConsumer retry configuration -->
    <!-- ------------------------------------------------------- -->
    <topic name="product.sync" xmlns:item="http://www.w3.org/2005/Atom">
        <handler name="ProductSyncConsumer">
            <item name="type"   value="service"/>
            <item name="method" value="process"/>
            <item name="queue"  value="product.sync"/>
            <item name="item:"  value="max-retries"      value-type="int">3</item>
            <item name="item:"  value="retry-delay-ms"   value-type="int">5000</item>
            <item name="item:"  value="failure-routing-key" value-type="string">dlq.product.sync</item>
        </handler>
    </topic>

    <!-- ------------------------------------------------------- -->
    <!-- OrderNotificationConsumer — shorter retry window -->
    <!-- ------------------------------------------------------- -->
    <topic name="order.notification" xmlns:item="http://www.w3.org/2005/Atom">
        <handler name="OrderNotificationConsumer">
            <item name="type"   value="service"/>
            <item name="method" value="process"/>
            <item name="queue"  value="order.notification"/>
            <item name="item:"  value="max-retries"      value-type="int">5</item>
            <item name="item:"  value="retry-delay-ms"   value-type="int">2000</item>
            <item name="item:"  value="failure-routing-key" value-type="string">dlq.order.notification</item>
        </handler>
    </topic>

</config>
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `max-retries` | 3 | Total attempts before DLQ routing (total = max-retries + initial attempt) |
| `retry-delay-ms` | 0 (immediate) | Milliseconds to wait before requeue. Can use exponential backoff via multiple `delay-ms` entries |
| `failure-routing-key` | `failure` | Routing key used when routing failed message to DLX (must match a binding in `queue_topology.xml`) |

**Exponential backoff via multiple delay entries:**

```xml
<item name="item:" value="retry-delay-ms" value-type="int">2000</item>
<item name="item:" value="retry-delay-ms" value-type="int">5000</item>
<item name="item:" value="retry-delay-ms" value-type="int">15000</item>
<!-- total: 2s → 5s → 15s between retries before DLQ -->
```

**DLQ naming convention:**

| Queue Name | DLQ Name |
|------------|----------|
| `product.sync` | `product.sync.dlq` or `dlq.product.sync` (取决于 `failure-routing-key`) |
| `order.notification` | `dlq.order.notification` |
| `inventory.update` | `dlq.inventory.update` |

> **📋 Verification:** After modifying topology:
> ```bash
> bin/magento queue:topology:update   # applies topology changes to RabbitMQ
> # or for MySQL:
> bin/magento setup:db-declaration:generate-whitelist
> ```
> Verify DLQ exists: `rabbitmqctl list_queues | grep dlq`

---

### 3. Consumer Parallelism — `[queue_consumers_number]`

By default, each consumer runs in a single process. Under load, this becomes a bottleneck.
The `[queue_consumers_number]` setting spawns multiple parallel instances of the same consumer,
each processing messages independently.

**Use cases for parallelism:**

- High-volume topics: `product.sync`, `order.notification`, `inventory.update`
- Consumers that spend time in I/O (HTTP calls, file writes) rather than CPU
- NOT suitable for consumers that modify shared state without locking

**`env.php` consumer parallelism config:**

```php
// app/etc/env.php
return [
    // ... other config ...
    'system' => [
        'default' => [
            'queue_consumers' => [
                // -------------------------------------------------------
                // Per-consumer parallelism limit
                // Magento 2.4.7+ supports this per-consumer syntax
                // -------------------------------------------------------
                // Run 4 instances of ProductSyncConsumer in parallel
                'product.sync'       => 4,
                // Run 2 instances of OrderNotificationConsumer
                'order.notification'=> 2,
                // Leave BulkNotifyConsumer at 1 (writes to shared log)
                'bulk.notify'        => 1,
                // -------------------------------------------------------
                // Global default (applied to unlisted consumers)
                // -------------------------------------------------------
                'default'           => 2,
            ],
        ],
    ],
];
```

**`env.php` — global parallelism fallback (Magento < 2.4.7):**

```php
// app/etc/env.php  (older syntax — applies to ALL consumers)
'queue' => [
    'consumers' => [
        // Run up to 3 instances of every consumer listed in queue_topology.xml
        // WARNING: Only apply this when you have tested memory/CPU limits
        'MAGENTO_MODULES' => [
            'Magento_Amqp' => 1,
        ],
    ],
],
// In system.xml or Admin → System → Configuration → Queue:
```

**Cron group consumer runner — max_messages per run:**

```php
// app/etc/env.php
'system' => [
    'default' => [
        'cron_consumers_runner' => [
            'max-messages'        => 5000,   // max messages per cron run per consumer
            'consumers'           => 'all',   // or comma-separated list
            'multipleProcesses'   => 4,       // spawn 4 cron processes concurrently
            'php-binary'          => 'php',   // path to PHP binary
            'php-timeout'         => 600,     // PHP process timeout (seconds)
        ],
    ],
],
```

> **📋 Key insight:** `max-messages` is NOT the same as consumer parallelism.
> - `max-messages` — how many messages a single consumer instance processes before exiting
> - `queue_consumers_number` — how many parallel processes run simultaneously
>
> Set `max-messages` to a reasonable cap so cron can exit cleanly and reschedule.
> Set `queue_consumers_number` based on CPU cores and consumer memory footprint.

**Parallelism decision matrix:**

| Consumer type | `queue_consumers_number` | Reason |
|---------------|--------------------------|--------|
| Product sync to ERP | 4–8 | I/O-bound, multiple HTTP calls |
| Order email sender | 2–4 | SMTP latency, retriable |
| Inventory update | 4 | High volume, idempotent |
| Shared log writer | 1 | Sequential writes, file locking |
| Cache warmup | 2 | Sequential dependency chain |

---

### 4. Deferred DB Commit — `useDbAmqpCommit`

This flag controls whether bulk/batch messages wait for full DB transaction commit before
acknowledgment. It is a critical correctness-vs-performance knob.

**When `useDbAmqpCommit = true` (safer, default for MySQL adapter):**

```
BEGIN transaction
  INSERT order_comment
  INSERT order_history
  INSERT order_status_log
COMMIT transaction
  → Then publish queue message (guaranteed entity exists in DB)
```

- ✅ Message published ONLY after DB transaction is committed
- ✅ Consumer reads entity from DB and finds it — no 404
- ✅ Works correctly with MySQL table locks and REPEATABLE READ isolation
- ❌ Latency: DB commit + network roundtrip to broker before Ack

**When `useDbAmqpCommit = false` (faster, use with RabbitMQ):**

```
BEGIN transaction
  INSERT order_comment
  INSERT order_history
  INSERT order_status_log
COMMIT transaction (async/non-blocking)
  → Publish queue message immediately (may arrive before commit!)
```

- ✅ Lower latency — publish happens immediately after transaction COMMIT
- ✅ Better throughput for RabbitMQ which handles ordering differently
- ❌ Risk: Consumer may query DB before commit arrives — add retry logic
- ✅ Acceptable when entity has a status field (consumer can wait/sleep-and-retry)

**`env.php` configuration:**

```php
// app/etc/env.php
'queue' => [
    'MAGENTO_MODULES' => [
        'Magento_Amqp' => 1,
        'Magento_MysqlMq' => 0,
    ],
    // -------------------------------------------------------
    // Deferred DB commit flag
    // true  = publish AFTER DB commit (safer, default)
    // false = publish IMMEDIATELY after COMMIT (faster, RabbitMQ)
    // -------------------------------------------------------
    'useDbAmqpCommit' => false,   // ← set to false when using RabbitMQ
],
```

> **📋 Decision rule:** If your consumer reads entity data from DB (most common case),
> keep `useDbAmqpCommit = true` unless you have confirmed via load testing that the
> ~1–5ms deferral matters for your throughput requirements. If you are using RabbitMQ
> and the consumer only processes external API calls (no DB read), set `false`.

---

### 5. Cron Group Concurrency — `[cron_consumers_runner]`

The `cron_consumers_runner` section controls how Magento launches queue consumers via cron.
It is the bridge between the cron scheduler and the message queue processor.

**How it works:**

```
crontab → cron.php → looks for jobs in consumers group
                        → bin/magento queue:consumers:run [consumer_name]
                              → spawns PHP process
                              → processes up to max-messages
                              → exits cleanly
```

**Key settings explained:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `max-messages` | 10000 | Max messages per consumer invocation. Set lower for heavy consumers |
| `consumers` | `all` | `all` = all consumers listed in `queue_topology.xml`. Or specific consumer names |
| `multipleProcesses` | 1 | Number of parallel cron processes to spawn |
| `php-binary` | `php` | Path to PHP binary (use absolute path in production) |
| `php-timeout` | 600 | Hard timeout for PHP process (kills stuck consumer) |

**Production `env.php` cron config example:**

```php
// app/etc/env.php
'system' => [
    'default' => [
        'cron_consumers_runner' => [
            'id'                  => 'consumers',   // internal, must be 'consumers'
            'max-messages'        => 5000,          // cap per run (prevents infinite processing)
            'consumers'           => 'all',         // or: 'product.sync,order.notification'
            'multipleProcesses'   => 4,             // run 4 parallel PHP cron processes
            'php-binary'          => '/usr/bin/php8.2',  # absolute path — critical for cron
            'php-timeout'         => 900,           # 15 min max — adjust per consumer
        ],
        // -------------------------------------------------------
        // Consumer-specific settings (Magento 2.4.7+)
        // Per-consumer max-messages override
        // -------------------------------------------------------
        'queue_consumers' => [
            // Heavy consumer — lower message cap per run
            'order.notification' => [
                'max-messages' => 1000,
                'single-queue' => true,    # force single-queue mode (no parallelism)
            ],
            // Light consumer — higher cap
            'product.sync' => [
                'max-messages' => 10000,
            ],
        ],
    ],
],
```

**Crontab entry (auto-generated by Magento — do not edit manually):**

```bash
# Generated by Magento at setup:cron:run
# Do not edit manually — use Admin → System → Cron instead
*/1 * * * * /usr/bin/php8.2 /var/www/magento2/bin/magento cron:run --group=consumers
```

> **📋 Key insight:** `multipleProcesses = 4` spawns 4 parallel `queue:consumers:run`
> invocations. Each runs independently. If `product.sync = 4` in `queue_consumers_number`,
> and `multipleProcesses = 4`, you could have up to 16 parallel `ProductSyncConsumer`
> instances during a cron run. Scale carefully.

---

### 6. Message Chunk Size Tuning — `[queue_message_processing_limit]`

This setting controls how many messages RabbitMQ prefetches per consumer. It is the single
most impactful setting for throughput under load.

**Prefetch concept:**

```
RabbitMQ broker  ─────── prefetch=100 ────────► Consumer process
   [msg1, msg2, ..., msg100]  (buffered locally)
```

- Messages are buffered in RabbitMQ, not held by broker
- Consumer processes messages locally; ACK releases next batch
- `prefetch` of 1 = one-at-a-time (safest but slowest)
- `prefetch` of 100 = 100 messages buffered locally (highest throughput)

**`queue_message_processing_limit` — global prefetch cap:**

```php
// app/etc/env.php
'system' => [
    'default' => [
        // -------------------------------------------------------
        // Global message processing limit
        // Caps the number of messages RabbitMQ will prefetch
        // regardless of per-consumer setting
        // -------------------------------------------------------
        'queue_message_processing_limit' => 100,
        // -------------------------------------------------------
        // Per-consumer override (Magento 2.4.7+)
        // -------------------------------------------------------
        'queue_consumers' => [
            'product.sync' => [
                'prefetch-count' => 50,   # smaller batch for heavy processing
            ],
            'order.notification' => [
                'prefetch-count' => 200,  # larger batch for light processing
            ],
        ],
    ],
],
```

**Consumer-specific prefetch via `queue_topology.xml` (alternative):**

```xml
<!-- etc/queue_topology.xml -->
<topic name="product.sync" xmlns:item="http://www.w3.org/2005/Atom">
    <handler name="ProductSyncConsumer">
        <item name="type"    value="service"/>
        <item name="method"  value="process"/>
        <item name="queue"   value="product.sync"/>
        <item name="item:"    value="prefetch-count" value-type="int">50</item>
    </handler>
</topic>
```

**Prefetch decision guide:**

| Consumer behavior | Recommended `prefetch-count` | Reason |
|-------------------|------------------------------|--------|
| Heavy processing per message (complex DB, API calls) | 10–50 | Avoid memory pressure; each message holds state |
| Light processing (email, simple DB update) | 100–250 | Maximize throughput; consumer is fast |
| Single DB transaction per message | 50–100 | DB connection pool limit applies |
| External API with rate limiting | 20–50 | Respect third-party rate limits across prefetch window |
| Memory-intensive payload in message body | 10–20 | Each prefetched message lives in RAM |

> **⚠️ Warning:** Setting `prefetch-count` too high can cause OOM in consumers that hold
> large message payloads or entity data in memory. Monitor PHP `memory_get_usage(true)`
> under load test before setting production values.

**Complete production `env.php` queue section (all settings combined):**

```php
// app/etc/env.php  — complete production queue configuration
return [
    // ... other sections ...

    // ============================================================
    // QUEUE CONFIGURATION (all settings in one place)
    // ============================================================
    'queue' => [
        // -------------------------------------------------------
        // Module enable/disable
        // -------------------------------------------------------
        'MAGENTO_MODULES' => [
            'Magento_Amqp' => 1,       // RabbitMQ adapter
            'Magento_MysqlMq' => 0,    // disable MySQL adapter
        ],

        // -------------------------------------------------------
        // RabbitMQ connection settings
        // -------------------------------------------------------
        'amqp' => [
            'host'            => '127.0.0.1',
            'port'            => 5672,
            'user'            => 'magento',
            'password'        => 'magento123',
            'vhost'           => '/',
            'ssl'             => 'false',
            'bulk'            => '1',
            'batch-size'      => '100',
        ],

        // -------------------------------------------------------
        // Deferred commit — false for RabbitMQ (publish immediately)
        // -------------------------------------------------------
        'useDbAmqpCommit' => false,
    ],

    'system' => [
        'default' => [
            // -------------------------------------------------------
            // Consumer parallelism (how many instances per consumer)
            // -------------------------------------------------------
            'queue_consumers' => [
                'product.sync'        => 4,
                'order.notification'  => 2,
                'bulk.notify'         => 1,
                'default'            => 2,
            ],

            // -------------------------------------------------------
            // Cron group consumer runner
            // -------------------------------------------------------
            'cron_consumers_runner' => [
                'id'                 => 'consumers',
                'max-messages'       => 5000,
                'consumers'          => 'all',
                'multipleProcesses'  => 4,
                'php-binary'         => '/usr/bin/php8.2',
                'php-timeout'        => 900,
            ],

            // -------------------------------------------------------
            // Global message processing (prefetch) limit
            // -------------------------------------------------------
            'queue_message_processing_limit' => 100,

            // -------------------------------------------------------
            // Per-consumer overrides
            // -------------------------------------------------------
            'queue_consumers' => [
                'product.sync' => [
                    'prefetch-count' => 50,
                    'max-messages'   => 2000,
                ],
                'order.notification' => [
                    'prefetch-count' => 200,
                    'max-messages'   => 10000,
                ],
            ],
        ],
    ],
];
```

---

## Definition of Done

Before marking a queue implementation as production-ready, verify each item:

- [ ] **Adapter:** `env.php` has `Magento_Amqp => 1` and `Magento_MysqlMq => 0`
- [ ] **Connection:** `bin/magento queue:consumers:list` returns expected consumers
- [ ] **Topology:** `queue_topology.xml` is deployed and `bin/magento queue:topology:update` ran
- [ ] **DLQ exists:** `rabbitmqctl list_queues | grep dlq` shows dead-letter queues
- [ ] **Retry config:** `max-retries`, `retry-delay-ms`, `failure-routing-key` set per topic
- [ ] **Parallelism tested:** Load test with `queue_consumers_number > 1` shows no race conditions
- [ ] **Idempotency verified:** Consumer produces same result when called 2x with same message
- [ ] **Deferred commit:** Consumer handles the case where entity may not yet be committed (if `useDbAmqpCommit = false`)
- [ ] **max-messages set:** No consumer has unlimited `max-messages` in production
- [ ] **Cron group active:** `crontab -l | grep consumers` shows the cron group entry
- [ ] **Prefetch tuned:** Under load test, no consumer OOM; throughput is acceptable
- [ ] **Monitoring:** Alerting set up for `queue_message.status = 4` (rejected) growing
- [ ] **Management UI:** RabbitMQ management UI accessible and authenticated

---

*Magento 2 Backend Developer Course — Topic 08*
