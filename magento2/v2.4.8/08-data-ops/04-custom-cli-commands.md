---
title: "04 - Custom CLI Commands in Magento 2"
description: "Comprehensive guide to creating custom CLI commands in Magento 2.4.8 using Symfony Console, dependency injection, and the CommandPool pattern."
tags: magento2, cli, bin/magento, symfony-console, command-pool, dependency-injection
rank: 4
pathways: [magento2-deep-dive]
see_also:
  - /home/letiendat1002/workspace/personal/courses/magento2/v2.4.8/08-data-ops/README.md
  - /home/letiendat1002/workspace/personal/courses/magento2/v2.4.8/08-data-ops/16-message-queue.md
---

# Custom CLI Commands in Magento 2

Magento 2's command-line interface is built on Symfony Console, providing a powerful framework for creating custom CLI commands that integrate seamlessly with Magento's dependency injection system, service layer, and database. While the README mentions `bin/magento training:review:approve` as an example, this chapter provides an exhaustive reference for building production-grade CLI commands.

---

## 1. CommandInterface — The Foundation

### 1.1 Understanding CommandInterface

Magento 2 implements the Symfony Console Command pattern through `Magento\Framework\Console\CommandInterface`. This interface is the contract that all Magento CLI commands must fulfill, bridging Magento's dependency injection with Symfony Console's command architecture.

```php
// Magento\Framework\Console\CommandInterface
namespace Magento\Framework\Console;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

/**
 * Interface for Magento CLI commands
 */
interface CommandInterface
{
    /**
     * Executes the current command
     *
     * @param InputInterface $input
     * @param OutputInterface $output
     * @return int Exit code
     */
    public function execute(InputInterface $input, OutputInterface $output): int;
}
```

### 1.2 The ListCommand as Reference Implementation

Magento's `ListCommand` (`Magento\Framework\Console\Command\ListCommand`) is the canonical reference implementation for how Magento expects commands to be structured. This command is what runs when you execute `bin/magento list` without any arguments.

```php
// vendor/magento/framework/Console/Command/ListCommand.php (simplified)
namespace Magento\Framework\Console\Command;

use Magento\Framework\Console\CommandInterface;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Input\InputOption;

class ListCommand extends Command implements CommandInterface
{
    // Commands are named using vendor:area:action pattern
    protected $commandName = 'list';

    protected function configure(): void
    {
        $this->setName($this->commandName);
        $this->setDescription('Lists all available commands');

        $this->addOption(
            'format',      // Long name
            'f',           // Short name (can be null)
            InputOption::VALUE_OPTIONAL,  // Value mode
            'Output format (txt, xml, JSON, or list)' // Description
        );

        $this->addOption(
            'raw',
            null,
            InputOption::VALUE_NONE,
            'To output raw command list'
        );

        parent::configure();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        // Implementation...
        return \Magento\Framework\Console\Cli::RETURN_SUCCESS;
    }
}
```

### 1.3 The Two Required Methods

Every Magento CLI command must implement two methods:

**`configure()` — Command Definition**

The `configure()` method is called during command registration and defines:
- The command name (using the `vendor:area:action` convention)
- Command description shown in `bin/magento list`
- Arguments and options the command accepts
- Help text

**`execute(InputInterface, OutputInterface)` — Business Logic**

The `execute()` method contains the actual command logic. It receives:
- `InputInterface` — access to passed arguments and options
- `OutputInterface` — methods for writing to console

Returns an integer exit code: `Cli::RETURN_SUCCESS` (0) or `Cli::RETURN_FAILURE` (1).

---

## 2. di.xml Registration — How Magento Finds Your Commands

### 2.1 CommandPool Pattern

Magento registers CLI commands through `Magento\Framework\Console\CommandPool`. This pool is populated via `di.xml` configuration, where you specify which commands belong to your module.

```xml
<!-- etc/di.xml in your module -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/di.xsd">

    <!-- CommandPool registration — all Magento CLI commands flow through here -->
    <type name="Magento\Framework\Console\CommandPool">
        <arguments>
            <argument name="commands" xsi:type="array">
                <!-- Each item key must match module name for proper sorting -->
                <item name="Training_Review" xsi:type="object">Training\Review\Console\Command\ApproveReview</item>
            </argument>
        </arguments>
    </type>
</config>
```

### 2.2 Alternative: Individual Type Preferences

For more granular control, you can register individual commands via type preferences:

```xml
<!-- etc/di.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/di.xsd">

    <!-- Register individual command classes with DI preferences -->
    <type name="Training\Review\Console\Command\ApproveReview">
        <arguments>
            <argument name="reviewRepository" xsi:type="object">Training\Review\Api\ReviewRepositoryInterface</argument>
        </arguments>
    </type>

    <type name="Training\Review\Console\Command\ListReviews">
        <arguments>
            <argument name="searchCriteriaBuilder" xsi:type="object">Magento\Framework\Api\SearchCriteriaBuilder</argument>
        </arguments>
    </type>
</config>
```

### 2.3 Registration Priority

Commands are discovered and registered in this order:

1. `Magento\Framework\Console\CommandPool` reads `di.xml`
2. All modules' `CommandPool` contributions are merged
3. Commands are sorted by module name
4. `ListCommand` collects all registered commands
5. `bin/magento list` displays the sorted list

> **Pro Tip:** The item key in the `commands` array (e.g., `Training_Review`) determines sorting order in `bin/magento list`. Use your full module name as the key for predictable ordering.

---

## 3. Command Class Anatomy

### 3.1 Full Namespace and Use Statements

A properly structured Magento CLI command uses these Symfony Console imports:

```php
<?php
// Console/Command/ApproveReview.php
namespace Training\Review\Console\Command;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Helper\Table;
use Magento\Framework\Console\Cli;
use Psr\Log\LoggerInterface;

/**
 * CLI command to approve reviews from the command line
 *
 * Usage: bin/magento training:review:approve <review_id> [--force]
 */
class ApproveReview extends Command
{
    /** @var LoggerInterface */
    private $logger;

    /** @var ReviewRepositoryInterface */
    private $reviewRepository;

    /**
     * Dependency injection via constructor
     *
     * @param LoggerInterface $logger
     * @param ReviewRepositoryInterface $reviewRepository
     */
    public function __construct(
        LoggerInterface $logger,
        \Training\Review\Api\ReviewRepositoryInterface $reviewRepository
    ) {
        parent::__construct();
        $this->logger = $logger;
        $this->reviewRepository = $reviewRepository;
    }

    protected function configure(): void
    {
        $this->setName('training:review:approve')
            ->setDescription('Mark a review as verified/approved')
            ->setHelp(<<<HELP
The <info>%command.name%</info> command marks a review as verified.

  <info>php %command.full_name% 123</info>

To skip confirmation:

  <info>php %command.full_name% 123 --force</info>
HELP
            )
            ->addArgument(
                'review_id',
                InputArgument::REQUIRED,
                'The review ID to approve'
            )
            ->addOption(
                'force',
                'f',
                InputOption::VALUE_NONE,
                'Skip confirmation prompt'
            );

        parent::configure();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $reviewId = (int) $input->getArgument('review_id');

        try {
            $review = $this->reviewRepository->getById($reviewId);

            if (!$input->getOption('force')) {
                $output->writeln("<comment>About to approve review #{$reviewId}</comment>");
                $output->writeln("Use --force to skip this confirmation");
                return Cli::RETURN_FAILURE;
            }

            $review->setIsVerified(true);
            $this->reviewRepository->save($review);

            $output->writeln("<info>Review #{$reviewId} approved successfully</info>");
            $this->logger->info("Review {$reviewId} approved via CLI command");

            return Cli::RETURN_SUCCESS;

        } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
            $output->writeln("<error>Review #{$reviewId} not found</error>");
            return Cli::RETURN_FAILURE;
        } catch (\Exception $e) {
            $output->writeln("<error>Error: {$e->getMessage()}</error>");
            $this->logger->error("Failed to approve review {$reviewId}", ['exception' => $e]);
            return Cli::RETURN_FAILURE;
        }
    }
}
```

### 3.2 Class Structure Summary

| Component | Purpose |
|-----------|---------|
| `extends Command` | Inherits from Symfony Console's base Command |
| `implements CommandInterface` | Fulfills Magento's CLI contract |
| `private $logger` | PSR-3 logger for audit trail |
| `private $reviewRepository` | Injected service for business logic |
| `configure()` | Defines name, description, arguments, options |
| `execute()` | Business logic, returns exit code |

---

## 4. Arguments vs Options — When to Use Each

### 4.1 Conceptual Differences

| Aspect | Arguments | Options |
|--------|-----------|---------|
| **Syntax** | Positional: `command <value>` | Flag-based: `command --option value` or `command -o value` |
| **Required** | Can be REQUIRED or OPTIONAL | Always optional (VALUE_NONE for boolean flags) |
| **Order** | Position matters | Order doesn't matter |
| **Examples** | `<review_id>` | `--force`, `--format=json` |

### 4.2 InputArgument Modes

```php
// No argument (only options)
// InputArgument::NONE — No value expected

// Optional argument — can be omitted
InputArgument::OPTIONAL

// Required argument — must be provided
InputArgument::REQUIRED

// Array argument — accepts multiple values
InputArgument::IS_ARRAY

// Required + Array — at least one value required
InputArgument::REQUIRED | InputArgument::IS_ARRAY
```

### 4.3 InputOption Modes

```php
// No value — boolean flag
// Usage: bin/magento command --force
InputOption::VALUE_NONE

// Single value
// Usage: bin/magento command --format json
InputOption::VALUE_OPTIONAL

// Must provide a value
// Usage: bin/magento command --format json
InputOption::VALUE_REQUIRED

// Multiple values as array
// Usage: bin/magento command --ids 1 --ids 2 --ids 3
InputOption::VALUE_IS_ARRAY

// Negatable option (2.4.7+)
// Usage: bin/magento command --no-force
InputOption::VALUE_NEGATABLE
```

### 4.4 Decision Matrix

Use this matrix to decide:

| Use Case | Type | Example |
|----------|------|---------|
| Primary identifier (review ID, product SKU) | Required Argument | `<review_id>` |
| Action modifier (format, output type) | Option | `--format=json` |
| Boolean toggle (force, verbose, dry-run) | VALUE_NONE | `--force` |
| Multiple values (IDs, emails) | IS_ARRAY Option | `--ids=1 --ids=2 --ids=3` |
| Optional primary input | Optional Argument | `[entity_id]` |

### 4.5 Examples in Practice

```php
protected function configure(): void
{
    // Required argument for primary input
    $this->addArgument(
        'review_id',
        InputArgument::REQUIRED,
        'The review ID to process'
    );

    // Optional argument with default value handled in execute()
    $this->addArgument(
        'status',
        InputArgument::OPTIONAL,
        'New status (approved/rejected/pending)'
    );

    // Option with required value
    $this->addOption(
        'format',
        'f',
        InputOption::VALUE_REQUIRED,
        'Output format (table, json, csv)',
        'table'  // default value
    );

    // Boolean flag (VALUE_NONE)
    $this->addOption(
        'force',
        null,
        InputOption::VALUE_NONE,
        'Skip confirmation prompt'
    );

    // Short option with VALUE_NONE
    $this->addOption(
        'verbose',
        'v',
        InputOption::VALUE_NONE,
        'Increase verbosity'
    );

    // Array option for multiple values
    $this->addOption(
        'product-ids',
        'p',
        InputOption::VALUE_IS_ARRAY,
        'Product IDs to process',
        []
    );
}
```

### 4.6 Long vs Short Option Names

```php
// Long name only (--verbose)
$this->addOption('verbose', null, InputOption::VALUE_NONE, 'Increase verbosity');

// Short only (-f) — not recommended, harder to discover
$this->addOption(null, 'f', InputOption::VALUE_NONE, 'Force operation');

// Both long and short (--force / -f) — RECOMMENDED
$this->addOption('force', 'f', InputOption::VALUE_NONE, 'Skip confirmation');
```

---

## 5. The configure() Method — Command Definition

### 5.1 Complete configure() Pattern

```php
protected function configure(): void
{
    // Set the command name using vendor:area:action pattern
    $this->setName('training:review:process')
        // Single-line description for --help summary
        ->setDescription('Process reviews in bulk with specified action')
        // Multi-line help text shown with --help
        ->setHelp(<<<'HELP'
The <info>%command.name%</info> command processes reviews in bulk.

Supports approve, reject, and delete actions:

  <info>php %command.full_name% --action=approve --ids=1,2,3</info>
  <info>php %command.full_name% --action=reject --ids=1,2,3 --reason="spam"</info>
  <info>php %command.full_name% --action=delete --force</info>

Use <info>bin/magento training:review:list</info> to see available reviews first.
HELP
        );

    // Add arguments
    $this->addArgument(
        'action',
        InputArgument::REQUIRED,
        'Action to perform: approve, reject, or delete'
    );

    // Add options with VALUE_REQUIRED
    $this->addOption(
        'ids',
        'i',
        InputOption::VALUE_REQUIRED,
        'Comma-separated review IDs'
    );

    // Add boolean flag
    $this->addOption(
        'force',
        'f',
        InputOption::VALUE_NONE,
        'Skip confirmation'
    );

    // Add option with allowed values
    $this->addOption(
        'format',
        null,
        InputOption::VALUE_REQUIRED,
        'Output format',
        'table'
    );

    // REQUIRED: Call parent configure()
    parent::configure();
}
```

### 5.2 Method Chaining

The `configure()` method uses a fluent interface — each setter returns `$this`. This allows chaining:

```php
$this->setName('training:review:approve')
    ->setDescription('Mark review as approved')
    ->setHelp('Usage: php bin/magento training:review:approve <id>')
    ->addArgument('review_id', InputArgument::REQUIRED, 'Review ID')
    ->addOption('force', 'f', InputOption::VALUE_NONE, 'Skip confirmation')
    ->setAliases(['training:review:verify']);  // Alternative names
```

### 5.3 Command Aliases

Add alternative names for backwards compatibility:

```php
$this->setName('training:review:approve')
    ->setAliases(['training:review:verify', 'training:approve-review'])
```

Now users can run any of:
- `bin/magento training:review:approve 123`
- `bin/magento training:review:verify 123`
- `bin/magento training:approve-review 123`

---

## 6. The execute() Method — Command Logic

### 6.1 Return Codes

Magento CLI commands must return specific integer constants:

```php
use Magento\Framework\Console\Cli;

// Success — command completed without errors
return Cli::RETURN_SUCCESS;  // = 0

// Failure — command failed or user aborted
return Cli::RETURN_FAILURE;   // = 1
```

### 6.2 Output Methods

**`writeln()` — With Newline**

```php
// Standard output with newline
$output->writeln('Processing complete');

// With output style tags for coloring
$output->writeln('<info>Success message</info>');     // Green
$output->writeln('<comment>Warning message</comment>'); // Yellow
$output->writeln('<error>Error message</error>');       // Red
$output->writeln('<question>Question?</question>');     // Blue
$output->writeln('<info>Info message</info>');          // Cyan

// Verbosity-aware output
$output->writeln('<info>Always shown</info>');
$output->writeln('<comment>Only in verbose mode</comment>', OutputInterface::VERBOSITY_VERBOSE);
$output->writeln('<comment>Only in very verbose mode</comment>', OutputInterface::VERBOSITY_VERY_VERBOSE);
$output->writeln('<comment>Only in debug mode</comment>', OutputInterface::VERBOSITY_DEBUG);
```

**`write()` — Without Newline**

```php
// Write without newline — stays on same line
$output->write('Processing');
$output->write('.');
$output->write('.');

// Progress indicator pattern
$output->write('Starting... ');
$output->write('Done!');  // Output: "Starting... Done!"
```

### 6.3 Complete execute() Template

```php
protected function execute(InputInterface $input, OutputInterface $output): int
{
    // Phase 1: Parse and validate input
    $reviewId = $input->getArgument('review_id');
    $force = $input->getOption('force');

    if (!$reviewId) {
        $output->writeln('<error>Review ID is required</error>');
        $output->writeln('Usage: bin/magento training:review:approve <review_id>');
        return Cli::RETURN_FAILURE;
    }

    // Phase 2: Business logic
    try {
        $review = $this->reviewRepository->getById((int) $reviewId);

        if (!$force) {
            $output->writeln("<comment>About to approve review #{$reviewId}</comment>");
            $output->writeln("Re-run with --force to confirm");
            return Cli::RETURN_FAILURE;
        }

        $review->setIsVerified(true);
        $this->reviewRepository->save($review);

        // Phase 3: Success output
        $output->writeln("<info>Review #{$reviewId} approved successfully</info>");
        $this->logger->info("Review {$reviewId} approved", [
            'review_id' => $reviewId,
            'by' => 'cli_command'
        ]);

        return Cli::RETURN_SUCCESS;

    } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
        $output->writeln("<error>Review #{$reviewId} not found</error>");
        return Cli::RETURN_FAILURE;

    } catch (\Exception $e) {
        $output->writeln("<error>Error: {$e->getMessage()}</error>");
        $this->logger->error("Failed to approve review", [
            'review_id' => $reviewId,
            'exception' => $e
        ]);
        return Cli::RETURN_FAILURE;
    }
}
```

### 6.4 Interactive Input in execute()

For commands requiring user interaction, use Symfony's question helper:

```php
use Symfony\Component\Console\Question\ConfirmationQuestion;
use Symfony\Component\Console\Question\Question;

protected function execute(InputInterface $input, OutputInterface $output): int
{
    $reviewId = (int) $input->getArgument('review_id');

    if (!$input->getOption('force')) {
        // Create interactive confirmation question
        $helper = $this->getHelper('question');
        $question = new ConfirmationQuestion(
            "<comment>Are you sure you want to approve review #{$reviewId}? [y/N]</comment> ",
            false  // default answer
        );

        if (!$helper->ask($input, $output, $question)) {
            $output->writeln('<comment>Operation cancelled</comment>');
            return Cli::RETURN_FAILURE;
        }
    }

    // Continue with business logic...
    return Cli::RETURN_SUCCESS;
}
```

---

## 7. Dependency Injection in Commands

### 7.1 Constructor Injection Pattern

Commands receive dependencies through constructor injection, just like any other Magento service class:

```php
class ApproveReview extends Command
{
    private $reviewRepository;
    private $logger;
    private $state;

    public function __construct(
        LoggerInterface $logger,
        \Training\Review\Api\ReviewRepositoryInterface $reviewRepository,
        \Magento\Framework\App\State $state
    ) {
        parent::__construct();
        $this->logger = $logger;
        $this->reviewRepository = $reviewRepository;
        $this->state = $state;
    }
}
```

### 7.2 Common Services to Inject

| Service | Interface/Class | Use Case |
|---------|-----------------|----------|
| Logger | `Psr\Log\LoggerInterface` | Audit logging |
| Repository | `*[Entity]RepositoryInterface` | Data access |
| SearchCriteria | `Magento\Framework\Api\SearchCriteriaBuilder` | Filtering collections |
| Collection | `*[Entity]CollectionFactory` | Bulk data access |
| State | `Magento\Framework\App\State` | Area switching |
| ManagerInterface | `Magento\Framework\ManagerInterface` | EAV attribute operations |
| ObjectManager | `Magento\Framework\ObjectManagerInterface` | When DI isn't possible |
| Cache | `Magento\Framework\App\CacheInterface` | Cache operations |

### 7.3 Area Switching in Commands

Many commands need Magento's full bootstrap to access models, repositories, or EAV attributes. Use `State` to switch areas:

```php
use Magento\Framework\App\State;

class BulkProcessReviews extends Command
{
    private $state;

    public function __construct(State $state)
    {
        parent::__construct();
        $this->state = $state;
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        try {
            // Ensure we have Magento context
            $this->state->getAreaCode();  // Will throw if not set

        } catch (\Magento\Framework\Exception\LocalizedException $e) {
            // Area not set — bootstrap adminhtml
            $this->state->setAreaCode('adminhtml');
        }

        // Now we can use models, repositories, EAV, etc.
        $collection = $this->collectionFactory->create();
        // ...

        return Cli::RETURN_SUCCESS;
    }
}
```

### 7.4 When to Use ObjectManager Directly

The `ObjectManager` should be a last resort. Use it only when:

1. **Circular dependency** — DI creates a cycle
2. **Late instantiation** — Object needed only on certain code paths
3. **Prototype scope** — Need fresh instance per call

```php
use Magento\Framework\ObjectManagerInterface;

class ComplexCommand extends Command
{
    private $objectManager;

    public function __construct(ObjectManagerInterface $objectManager)
    {
        parent::__construct();
        $this->objectManager = $objectManager;
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        // Use DI for all normal dependencies

        // ObjectManager only for late instantiation
        if ($input->getOption('dry-run')) {
            // Don't instantiate heavy class in dry-run mode
            $importer = $this->objectManager->create(
                \Training\Review\Model\Import\Review::class,
                ['isDryRun' => true]
            );
        }

        return Cli::RETURN_SUCCESS;
    }
}
```

> **⚠️ Warning:** Direct ObjectManager use bypasses Magento's DI, making testing harder and violating the service contract. Always prefer constructor injection.

---

## 8. Interacting with Magento from CLI

### 8.1 Full Bootstrap Context

By default, CLI commands run in the `global` area with limited access. For full Magento functionality, ensure the adminhtml area is loaded:

```php
protected function execute(InputInterface $input, OutputInterface $output): int
{
    /** @var \Magento\Framework\App\State $state */
    $state = $this->state;

    // Handle case where area code is not set
    if (!$state->getAreaCode()) {
        $output->writeln('<comment>Bootstrapping Magento adminhtml area...</comment>');
        $state->setAreaCode('adminhtml');
    }

    /** @var \Magento\Framework\App\AreaList $areaList */
    $areaList = $this->areaList;

    /** @var \Magento\Framework\App\Area $area */
    $area = $areaList->getArea('adminhtml');
    $area->load();

    // Now full Magento is available — repositories, models, EAV, etc.

    return Cli::RETURN_SUCCESS;
}
```

### 8.2 Complete Example with Area Switching

```php
<?php
// Console/Command/GenerateReviewSummary.php
namespace Training\Review\Console\Command;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;
use Magento\Framework\Console\Cli;
use Magento\Framework\App\State;
use Magento\Framework\App\AreaList;
use Magento\Framework\ObjectManagerInterface;
use Psr\Log\LoggerInterface;
use Training\Review\Api\ReviewRepositoryInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;

class GenerateReviewSummary extends Command
{
    private $state;
    private $areaList;
    private $objectManager;
    private $logger;
    private $reviewRepository;
    private $productRepository;

    public function __construct(
        State $state,
        AreaList $areaList,
        ObjectManagerInterface $objectManager,
        LoggerInterface $logger,
        ReviewRepositoryInterface $reviewRepository,
        ProductRepositoryInterface $productRepository
    ) {
        parent::__construct();
        $this->state = $state;
        $this->areaList = $areaList;
        $this->objectManager = $objectManager;
        $this->logger = $logger;
        $this->reviewRepository = $reviewRepository;
        $this->productRepository = $productRepository;
    }

    protected function configure(): void
    {
        $this->setName('training:review:summary')
            ->setDescription('Generate review summary report for products')
            ->addOption(
                'product-id',
                'p',
                InputOption::VALUE_REQUIRED,
                'Specific product ID (optional, generates for all if not set)'
            )
            ->addOption(
                'format',
                'f',
                InputOption::VALUE_REQUIRED,
                'Output format',
                'table'
            );

        parent::configure();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        // Bootstrap Magento adminhtml area for full access
        try {
            if (!$this->state->getAreaCode()) {
                $this->state->setAreaCode('adminhtml');
                $area = $this->areaList->getArea('adminhtml');
                $area->load();
            }
        } catch (\Exception $e) {
            $output->writeln('<error>Failed to bootstrap Magento</error>');
            $output->writeln("<error>{$e->getMessage()}</error>");
            return Cli::RETURN_FAILURE;
        }

        $productId = $input->getOption('product-id');
        $format = $input->getOption('format');

        try {
            if ($productId) {
                $result = $this->generateProductSummary((int) $productId);
            } else {
                $result = $this->generateAllProductsSummary();
            }

            $this->renderOutput($output, $result, $format);

            $this->logger->info('Review summary generated', [
                'product_id' => $productId ?? 'all',
                'format' => $format
            ]);

            return Cli::RETURN_SUCCESS;

        } catch (\Exception $e) {
            $output->writeln("<error>Error: {$e->getMessage()}</error>");
            $this->logger->error('Review summary generation failed', [
                'exception' => $e
            ]);
            return Cli::RETURN_FAILURE;
        }
    }

    private function generateProductSummary(int $productId): array
    {
        $product = $this->productRepository->getById($productId);

        // Use search criteria to get product reviews
        $reviews = $this->reviewRepository->getList(
            $this->searchCriteriaBuilder->addFilter('product_id', $productId)->create()
        );

        $ratings = [];
        foreach ($reviews->getItems() as $review) {
            $ratings[] = $review->getRating();
        }

        return [
            'product_id' => $productId,
            'product_name' => $product->getName(),
            'review_count' => count($ratings),
            'average_rating' => count($ratings) > 0 ? array_sum($ratings) / count($ratings) : 0,
        ];
    }

    private function generateAllProductsSummary(): array
    {
        // Implementation for all products
        return [];
    }

    private function renderOutput(OutputInterface $output, array $data, string $format): void
    {
        if ($format === 'json') {
            $output->writeln(json_encode($data, JSON_PRETTY_PRINT));
        } else {
            $output->writeln("Product ID: {$data['product_id']}");
            $output->writeln("Product Name: {$data['product_name']}");
            $output->writeln("Review Count: {$data['review_count']}");
            $output->writeln("Average Rating: " . round($data['average_rating'], 2));
        }
    }
}
```

---

## 9. Input/Output Streams — Verbosity and Progress

### 9.1 OutputInterface Verbosity Levels

Symfony Console defines verbosity levels:

| Constant | Level | When Triggered |
|----------|-------|----------------|
| `VERBOSITY_QUIET` | -1 | `bin/magento command -q` |
| `VERBOSITY_NORMAL` | 0 | Default |
| `VERBOSITY_VERBOSE` | 1 | `bin/magento command -v` |
| `VERBOSITY_VERY_VERBOSE` | 2 | `bin/magento command -vv` |
| `VERBOSITY_DEBUG` | 3 | `bin/magento command -vvv` |

### 9.2 Using Verbosity in Output

```php
protected function execute(InputInterface $input, OutputInterface $output): int
{
    // Always shown
    $output->writeln('<info>Starting process</info>');

    // Only shown with -v or higher
    $output->writeln(
        '<comment>Connecting to database...</comment>',
        OutputInterface::VERBOSITY_VERBOSE
    );

    // Only shown with -vv or higher
    $output->writeln(
        "<comment>SQL: {$query}</comment>",
        OutputInterface::VERBOSITY_VERY_VERBOSE
    );

    // Only shown with -vvv (debug)
    $output->writeln(
        "<comment>Memory usage: " . memory_get_usage(true) . " bytes</comment>",
        OutputInterface::VERBOSITY_DEBUG
    );

    // Conditional output based on verbosity
    if ($output->getVerbosity() >= OutputInterface::VERBOSITY_VERBOSE) {
        $output->writeln("Processing item {$i} of {$total}");
    }

    return Cli::RETURN_SUCCESS;
}
```

### 9.3 Setting Verbosity Programmatically

```php
// Increase verbosity
$output->setVerbosity(OutputInterface::VERBOSITY_VERBOSE);

// Decrease verbosity
$output->setVerbosity(OutputInterface::VERBOSITY_QUIET);

// Combine with output
$output->writeln(
    '<info>Detailed output</info>',
    $output->getVerbosity() | OutputInterface::VERBOSITY_VERBOSE
);
```

### 9.4 Progress Bar with SymfonyStyle

For long-running operations, use `SymfonyStyle` for automatic progress bars:

```php
use Symfony\Component\Console\Style\SymfonyStyle;
use Symfony\Component\Console\Helper\ProgressBar;

protected function execute(InputInterface $input, OutputInterface $output): int
{
    $io = new SymfonyStyle($input, $output);

    $total = 1000;
    $io->info("Processing {$total} items...");

    // Create progress bar (auto-adapts to terminal width)
    $progressBar = $io->createProgressBar($total);
    $progressBar->setFormat(' %current%/%max% [%bar%] %percent:3s%% %elapsed:6s%/%estimated:-6s%');
    $progressBar->setRedrawFrequency(100);  // Update every 100 items

    $processed = 0;
    foreach ($this->getItemsToProcess() as $item) {
        $this->processItem($item);
        $progressBar->advance();
        $processed++;

        // Advanced: Add custom messages at milestones
        if ($processed % 100 === 0) {
            $progressBar->setMessage("Processed {$processed} items");
        }
    }

    $progressBar->finish();
    $io->newLine(2);  // Add spacing after progress bar
    $io->success("Processed {$processed} items successfully");

    return Cli::RETURN_SUCCESS;
}
```

---

## 10. Table Output — Formatted Data Display

### 10.1 Basic Table with Table Helper

```php
use Symfony\Component\Console\Helper\Table;

protected function execute(InputInterface $input, OutputInterface $output): int
{
    $reviews = $this->reviewRepository->getList(
        $this->searchCriteriaBuilder->create()
    );

    $table = new Table($output);
    $table->setHeaders(['ID', 'Product', 'Reviewer', 'Rating', 'Status', 'Created']);

    foreach ($reviews->getItems() as $review) {
        $table->addRow([
            $review->getReviewId(),
            $review->getProductId(),
            $review->getReviewerName(),
            $this->renderStars($review->getRating()),
            $review->getIsVerified() ? '<info>Verified</info>' : '<comment>Pending</comment>',
            $review->getCreatedAt(),
        ]);
    }

    $table->render();

    return Cli::RETURN_SUCCESS;
}

private function renderStars(int $rating): string
{
    return str_repeat('★', $rating) . str_repeat('☆', 5 - $rating);
}
```

### 10.2 Table with Different Styles

```php
// Use compact style for many rows
$table->setStyle('compact');

// Use borderless style for cleaner output
$table->setStyle('borderless');

// Use Box style for emphasis
$table->setStyle('box');

// Modify existing style
$style = $table->getStyle();
$style->setCellHeaderFormat('<info>%s</info>');
$style->setBorderFormat('<comment>%s</comment>');
```

### 10.3 Vertical Table for Single Record

```php
protected function execute(InputInterface $input, OutputInterface $output): int
{
    $reviewId = (int) $input->getArgument('review_id');
    $review = $this->reviewRepository->getById($reviewId);

    $table = new Table($output);
    $table->setHeaders(['Field', 'Value']);
    $table->setRows([
        ['ID', $review->getReviewId()],
        ['Product ID', $review->getProductId()],
        ['Reviewer', $review->getReviewerName()],
        ['Rating', $this->renderStars($review->getRating())],
        ['Verified', $review->getIsVerified() ? 'Yes' : 'No'],
        ['Created', $review->getCreatedAt()],
        ['Text', $review->getReviewText()],
    ]);

    $table->render();

    return Cli::RETURN_SUCCESS;
}
```

---

## 11. Exception Handling in Commands

### 11.1 Exception Hierarchy

Magento commands should throw and catch specific exception types:

| Exception | When to Use |
|-----------|-------------|
| `LocalizedException` | User-facing errors with translatable messages |
| `NoSuchEntityException` | Entity not found (404) |
| `InputException` | Invalid arguments/options provided |
| `CouldNotSaveException` | Persistence operation failed |
| `StateException` | System in invalid state for operation |

### 11.2 Proper Exception Handling Pattern

```php
protected function execute(InputInterface $input, OutputInterface $output): int
{
    $reviewId = $input->getArgument('review_id');

    // Phase 1: Input validation
    if (!$reviewId) {
        throw new \Magento\Framework\Console\Exception\InputException(
            __('Review ID is required. Usage: bin/magento %1 <review_id>', $this->getName())
        );
    }

    if (!is_numeric($reviewId) || (int) $reviewId <= 0) {
        throw new \Magento\Framework\Console\Exception\InputException(
            __('Invalid review ID: %1. Must be a positive integer.', $reviewId)
        );
    }

    // Phase 2: Business logic with specific catches
    try {
        $review = $this->reviewRepository->getById((int) $reviewId);

    } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
        $output->writeln("<error>Review #{$reviewId} not found</error>");
        return Cli::RETURN_FAILURE;

    } catch (\Magento\Framework\Exception\CouldNotSaveException $e) {
        $output->writeln("<error>Failed to save review: {$e->getMessage()}</error>");
        $this->logger->error('Could not save review', [
            'review_id' => $reviewId,
            'exception' => $e
        ]);
        return Cli::RETURN_FAILURE;

    } catch (\Exception $e) {
        // Generic fallback — log and report
        $output->writeln("<error>Unexpected error: {$e->getMessage()}</error>");
        $this->logger->critical('Unexpected error in review command', [
            'review_id' => $reviewId,
            'exception' => $e
        ]);
        return Cli::RETURN_FAILURE;
    }

    return Cli::RETURN_SUCCESS;
}
```

### 11.3 LocalizedException for User Messages

```php
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Phrase;

// In your repository or service
throw new LocalizedException(
    new Phrase('Review with ID %1 does not exist', [$reviewId])
);

// In command catch block — exception message is already translated
} catch (LocalizedException $e) {
    $output->writeln("<error>{$e->getMessage()}</error>");
    return Cli::RETURN_FAILURE;
}
```

### 11.4 Exit Codes for Different Failure Types

```php
// Success
return Cli::RETURN_SUCCESS;  // 0

// User error (wrong input, not found)
return Cli::RETURN_FAILURE;  // 1

// System error (DB down, permission denied)
return Cli::RETURN_FAILURE;  // 1

// Custom exit codes for scripts that need to distinguish
const RETURN_INVALID_INPUT = 2;
const RETURN_NOT_FOUND = 3;
const RETURN_ALREADY_PROCESSED = 4;

// Use in your command
if ($invalidInput) {
    $output->writeln('<error>Invalid input</error>');
    return 2;  // Custom code
}
```

---

## 12. Reference Commands in Magento Core

### 12.1 Listing All Available Commands

```bash
# List all commands
bin/magento list

# List commands matching a pattern
bin/magento list | grep cache
bin/magento list | grep indexer

# Get help for specific command
bin/magento help cache:flush
bin/magento cache:status --help
```

### 12.2 Commands to Study as References

| Command | Class | What to Learn |
|---------|-------|---------------|
| `bin/magento dev:di:info` | `Magento\Developer\Console\Command\DiInfoCommand` | Complex argument handling, table output |
| `bin/magento cache:status` | `Magento\Cache\Command\Status` | Simple status command |
| `bin/magento cache:flush` | `Magento\Cache\Command\Flush` | Confirmation patterns |
| `bin/magento indexer:reindex` | `Magento\Indexer\Console\Command\Indexer\ReindexCommand` | Progress bars, batch processing |
| `bin/magento setup:upgrade` | `Magento\Setup\Console\Command\UpgradeCommand` | Area loading, long-running tasks |
| `bin/magento module:status` | `Magento\Setup\Console\Command\ModuleStatusCommand` | List output, formatting |

### 12.3 DevDiInfoCommand — Rich Command Example

The `DevDiInfoCommand` is an excellent reference because it demonstrates:
- Multiple required arguments
- Optional options
- Table output with multiple sections
- Complex data rendering

```php
// Key patterns from Magento\Developer\Console\Command\DiInfoCommand
protected function configure(): void
{
    $this->setName('dev:di:info')
        ->setDescription('Provides information about Dependency Injection configuration')
        ->addArgument(
            'class',
            InputArgument::REQUIRED,
            'The name of the class for which to view DI configuration'
        );

    parent::configure();
}

protected function execute(InputInterface $input, OutputInterface $output): int
{
    $className = $input->getArgument('class');

    // Use Table helper for structured output
    $table = new Table($output);
    $table->setHeaders(['Configuration']);

    // Add rows with formatted data
    foreach ($configurations as $item) {
        $table->addRow([$item]);
    }

    $table->render();

    return Cli::RETURN_SUCCESS;
}
```

### 12.4 IndexerReindexCommand — Progress Bar Example

```php
// Key patterns from Magento\Indexer\Console\Command\Indexer\ReindexCommand
protected function execute(InputInterface $input, OutputInterface $output): int
{
    $io = new SymfonyStyle($input, $output);

    // Long operation with progress
    $io->info('Reindexing...');

    $progressBar = $io->createProgressBar($total);
    $progressBar->start();

    foreach ($items as $item) {
        $this->processItem($item);
        $progressBar->advance();
    }

    $progressBar->finish();
    $io->newLine();
    $io->success('Reindexing completed');

    return Cli::RETURN_SUCCESS;
}
```

---

## 13. Testing CLI Commands

### 13.1 Unit Testing with CommandTester

Symfony Console provides `CommandTester` for testing commands:

```php
<?php
// Test/Unit/Console/Command/ApproveReviewTest.php
namespace Training\Review\Test\Unit\Console\Command;

use PHPUnit\Framework\TestCase;
use Symfony\Component\Console\Application;
use Symfony\Component\Console\Tester\CommandTester;
use Symfony\Component\Console\Input\ArrayInput;
use Symfony\Component\Console\Output\NullOutput;
use Magento\Framework\Console\Cli;
use Training\Review\Console\Command\ApproveReview;
use Training\Review\Api\ReviewRepositoryInterface;
use Training\Review\Api\Data\ReviewInterface;
use Psr\Log\LoggerInterface;

class ApproveReviewTest extends TestCase
{
    private $command;
    private $reviewRepository;
    private $logger;

    protected function setUp(): void
    {
        $this->reviewRepository = $this->createMock(ReviewRepositoryInterface::class);
        $this->logger = $this->createMock(LoggerInterface::class);

        $this->command = new ApproveReview(
            $this->logger,
            $this->reviewRepository
        );
    }

    public function testExecuteWithMissingReviewId(): void
    {
        $application = new Application();
        $application->add($this->command);

        $command = $application->find('training:review:approve');
        $commandTester = new CommandTester($command);

        // Execute without required argument
        $commandTester->execute([
            'command' => $command->getName(),
            // Missing 'review_id' argument
        ]);

        $this->assertEquals(Cli::RETURN_FAILURE, $commandTester->getStatusCode());
        $this->assertStringContainsString(
            'Review ID is required',
            $commandTester->getDisplay()
        );
    }

    public function testExecuteWithNonExistentReview(): void
    {
        $this->reviewRepository
            ->method('getById')
            ->with(999)
            ->willThrowException(
                new \Magento\Framework\Exception\NoSuchEntityException(
                    __('Review with ID %1 does not exist', 999)
                )
            );

        $application = new Application();
        $application->add($this->command);

        $command = $application->find('training:review:approve');
        $commandTester = new CommandTester($command);

        $commandTester->execute([
            'command' => $command->getName(),
            'review_id' => 999,
        ]);

        $this->assertEquals(Cli::RETURN_FAILURE, $commandTester->getStatusCode());
        $this->assertStringContainsString('not found', $commandTester->getDisplay());
    }

    public function testExecuteSuccess(): void
    {
        $review = $this->createMock(ReviewInterface::class);
        $review->method('getReviewId')->willReturn(123);
        $review->method('getIsVerified')->willReturn(false);

        $this->reviewRepository
            ->method('getById')
            ->with(123)
            ->willReturn($review);

        $this->reviewRepository
            ->expects($this->once())
            ->method('save')
            ->with($review);

        $this->logger
            ->expects($this->once())
            ->method('info')
            ->with('Review 123 approved', $this->anything());

        $application = new Application();
        $application->add($this->command);

        $command = $application->find('training:review:approve');
        $commandTester = new CommandTester($command);

        $commandTester->execute([
            'command' => $command->getName(),
            'review_id' => 123,
            '--force' => true,
        ]);

        $this->assertEquals(Cli::RETURN_SUCCESS, $commandTester->getStatusCode());
        $this->assertStringContainsString('approved successfully', $commandTester->getDisplay());
    }
}
```

### 13.2 Integration Testing with Full Magento Bootstrap

```php
<?php
// Test/Integration/Console/Command/ApproveReviewIntegrationTest.php
namespace Training\Review\Test\Integration\Console\Command;

use Magento\TestFramework\Console\ExternalBootstrap;
use Symfony\Component\Console\Tester\CommandTester;

class ApproveReviewIntegrationTest extends ExternalBootstrap
{
    protected function setUp(): void
    {
        // Bootstrap Magento application
        $this->bootstrap = static::getInstance();
    }

    public function testFullWorkflow(): void
    {
        // Create a review via repository
        $review = $this->reviewFactory->create();
        $review->setProductId(1);
        $review->setReviewerName('Test Reviewer');
        $review->setRating(5);
        $review->setIsVerified(false);
        $this->reviewRepository->save($review);

        $reviewId = $review->getReviewId();

        // Run the CLI command
        $command = $this->application->find('training:review:approve');
        $commandTester = new CommandTester($command);

        $commandTester->execute([
            'command' => $command->getName(),
            'review_id' => $reviewId,
            '--force' => true,
        ]);

        // Verify output
        $this->assertEquals(0, $commandTester->getStatusCode());

        // Verify database state
        $savedReview = $this->reviewRepository->getById($reviewId);
        $this->assertTrue($savedReview->getIsVerified());
    }
}
```

### 13.3 Testing Interactive Commands

```php
public function testInteractiveConfirmation(): void
{
    $review = $this->createMock(ReviewInterface::class);
    $this->reviewRepository->method('getById')->willReturn($review);

    $application = new Application();
    $application->add($this->command);

    $command = $application->find('training:review:approve');
    $commandTester = new CommandTester($command);

    // Simulate 'n' (no) response to confirmation
    $commandTester->setInputs(['n']);

    $commandTester->execute([
        'command' => $command->getName(),
        'review_id' => 123,
        // Note: --force NOT provided, so confirmation is needed
    ]);

    $this->assertEquals(Cli::RETURN_FAILURE, $commandTester->getStatusCode());
    $this->assertStringContainsString('cancelled', $commandTester->getDisplay());
}
```

---

## 14. Common Mistakes to Avoid

### 14.1 Missing Constructor Dependencies in di.xml

**❌ Wrong:** Command has dependencies but they're not declared

```php
// Command expects injected repository
public function __construct(ReviewRepositoryInterface $reviewRepository)
{
    parent::__construct();
    $this->reviewRepository = $reviewRepository;
}
```

```xml
<!-- di.xml — MISSING the command's dependencies! -->
<type name="Magento\Framework\Console\CommandPool">
    <arguments>
        <argument name="commands" xsi:type="array">
            <item name="Training_Review" xsi:type="object">Training\Review\Console\Command\ApproveReview</item>
        </argument>
    </arguments>
</type>
<!-- ReviewRepositoryInterface is NOT bound anywhere! -->
```

**✅ Correct:** Either inject via di.xml OR let Magento's auto-di resolve it

```xml
<!-- Option 1: Explicit preference for interface -->
<preference for="Training\Review\Api\ReviewRepositoryInterface"
            type="Training\Review\Model\ReviewRepository"/>
```

### 14.2 Using ObjectManager When Not Necessary

**❌ Wrong:** Bypassing DI entirely

```php
public function __construct()
{
        parent::__construct();
        $this->objectManager = \Magento\Framework\App\ObjectManager::getInstance();
}
```

**✅ Correct:** Use proper constructor injection

```php
public function __construct(
    ReviewRepositoryInterface $reviewRepository,
    LoggerInterface $logger
) {
    parent::__construct();
    $this->reviewRepository = $reviewRepository;
    $this->logger = $logger;
}
```

### 14.3 Forgetting to Return Exit Code

**❌ Wrong:** No return statement — returns null instead of exit code

```php
protected function execute(InputInterface $input, OutputInterface $output): int
{
    $output->writeln('Done');
    // Missing return! PHP returns null implicitly
}
```

**✅ Correct:** Always return an explicit exit code

```php
protected function execute(InputInterface $input, OutputInterface $output): int
{
    try {
        $this->processSomething();
        $output->writeln('<info>Done</info>');
        return Cli::RETURN_SUCCESS;
    } catch (\Exception $e) {
        $output->writeln('<error>Failed</error>');
        return Cli::RETURN_FAILURE;
    }
}
```

### 14.4 Forgetting to Call parent::__construct()

**❌ Wrong:** Command name not set

```php
public function __construct(
    LoggerInterface $logger
) {
    // Forgot: parent::__construct();
    $this->logger = $logger;
}
```

**✅ Correct:** Always call parent constructor

```php
public function __construct(
    LoggerInterface $logger
) {
    parent::__construct();  // Required!
    $this->logger = $logger;
}
```

### 14.5 Swallowing Exceptions Without Logging

**❌ Wrong:** Errors disappear silently

```php
try {
    $this->riskyOperation();
} catch (\Exception $e) {
    // Error swallowed! No logging, no output
    return Cli::RETURN_FAILURE;
}
```

**✅ Correct:** Always log exceptions

```php
try {
    $this->riskyOperation();
} catch (\Exception $e) {
    $this->logger->error('Operation failed', [
        'exception' => $e,
        'context' => 'for debugging'
    ]);
    $output->writeln("<error>Error: {$e->getMessage()}</error>");
    return Cli::RETURN_FAILURE;
}
```

### 14.6 Mixing up write() and writeln()

**❌ Wrong:** Using writeln when you want to stay on the same line

```php
// Creates unwanted newlines in progress output
for ($i = 0; $i < 10; $i++) {
    $output->writeln("Processing item $i");  // Each on new line
}
```

**✅ Correct:** Use write() for progress, writeln() when done

```php
for ($i = 0; $i < 10; $i++) {
    $output->write("\rProcessing item $i");  // Overwrites same line
}
$output->writeln('');  // Newline when done
```

### 14.7 Not Validating Input Types

**❌ Wrong:** Blindly assuming input is correct type

```php
$reviewId = $input->getArgument('review_id');
$review = $this->reviewRepository->getById($reviewId);  // $reviewId could be "abc"
```

**✅ Correct:** Validate and cast input

```php
$reviewId = $input->getArgument('review_id');

if (!is_numeric($reviewId)) {
    $output->writeln('<error>Review ID must be numeric</error>');
    return Cli::RETURN_FAILURE;
}

$review = $this->reviewRepository->getById((int) $reviewId);
```

---

## 15. Complete Module Structure for CLI Commands

### 15.1 File Organization

```
app/code/Training/Review/
├── Console/
│   └── Command/
│       ├── ApproveReview.php
│       ├── RejectReview.php
│       ├── ListReviews.php
│       ├── BulkProcessReviews.php
│       └── GenerateSummaryReport.php
├── etc/
│   ├── di.xml                    # Command registration
│   └── module.xml
├── Model/
│   └── ReviewRepository.php
├── Api/
│   ├── ReviewRepositoryInterface.php
│   └── Data/
│       └── ReviewInterface.php
└── registration.php
```

### 15.2 di.xml for Multiple Commands

```xml
<!-- etc/di.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/di.xsd">

    <!-- CommandPool — register all commands under module key -->
    <type name="Magento\Framework\Console\CommandPool">
        <arguments>
            <argument name="commands" xsi:type="array">
                <item name="Training_Review" xsi:type="object">Training\Review\Console\Command\ApproveReview</item>
            </argument>
        </arguments>
    </type>

    <!-- Individual command dependencies -->
    <type name="Training\Review\Console\Command\ApproveReview">
        <arguments>
            <argument name="reviewRepository" xsi:type="object">Training\Review\Api\ReviewRepositoryInterface</argument>
            <argument name="logger" xsi:type="object">Psr\Log\LoggerInterface</argument>
        </arguments>
    </type>

    <type name="Training\Review\Console\Command\ListReviews">
        <arguments>
            <argument name="reviewRepository" xsi:type="object">Training\Review\Api\ReviewRepositoryInterface</argument>
            <argument name="searchCriteriaBuilder" xsi:type="object">Magento\Framework\Api\SearchCriteriaBuilder</argument>
        </arguments>
    </type>

    <type name="Training\Review\Console\Command\BulkProcessReviews">
        <arguments>
            <argument name="state" xsi:type="object">Magento\Framework\App\State</argument>
            <argument name="areaList" xsi:type="object">Magento\Framework\App\AreaList</argument>
            <argument name="logger" xsi:type="object">Psr\Log\LoggerInterface</argument>
        </arguments>
    </type>
</config>
```

---

## See Also

- [Magento 2.4.8 CLI Overview](https://developer.adobe.com/commerce/php/development/cli-commands/) — Official CLI documentation
- [Symfony Console Documentation](https://symfony.com/doc/current/console.html) — Full Symfony Console reference
- [Magento CommandPool Source](https://github.com/magento/magento2/blob/2.4.8/lib/internal/Magento/Framework/Console/CommandPool.php) — How Magento registers commands
- [ListCommand Source](https://github.com/magento/magento2/blob/2.4.8/lib/internal/Magento/Framework/Console/Command/ListCommand.php) — Reference implementation
- [/home/letiendat1002/workspace/personal/courses/magento2/v2.4.8/08-data-ops/README.md](/home/letiendat1002/workspace/personal/courses/magento2/v2.4.8/08-data-ops/README.md) — Data Operations topic overview
- [/home/letiendat1002/workspace/personal/courses/magento2/v2.4.8/08-data-ops/16-message-queue.md](/home/letiendat1002/workspace/personal/courses/magento2/v2.4.8/08-data-ops/16-message-queue.md) — Related async operations
