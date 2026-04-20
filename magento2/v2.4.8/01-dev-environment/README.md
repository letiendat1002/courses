# Topic 1: Development Environment & Magento Foundations

**Goal:** Set up your development environment and understand Magento's architecture well enough to create a working module.

---

## Topics Covered

- PHP 8 modern syntax for reading Magento codebase
- What Magento is, where it fits in e-commerce, request flow overview
- Magento directory structure — where your code lives
- Module anatomy — minimum files needed to create a module
- Docker-based development environment setup
- `bin/magento` CLI — essential commands you'll use daily
- Reading and writing Magento log files
- Module registration, enable/disable cycle

---

### Topic 0: PHP 8 Modern Syntax — Reading Modern Magento Code

**Why This Topic Exists:** Magento 2.4.8 is written in PHP 8.1/8.2. You'll encounter modern syntax throughout the codebase. Understanding these patterns before you read core code saves hours of confusion.

**Topics:**

#### 1. Constructor Property Promotion
```
// BEFORE PHP 8 — boilerplate in every constructor
class ProductRepository {
    private ProductFactory $productFactory;
    private SearchCriteriaBuilder $criteriaBuilder;
    
    public function __construct(
        ProductFactory $productFactory,
        SearchCriteriaBuilder $criteriaBuilder
    ) {
        $this->productFactory = $productFactory;
        $this->criteriaBuilder = $criteriaBuilder;
    }
}

// AFTER PHP 8 — single line
class ProductRepository {
    public function __construct(
        private ProductFactory $productFactory,
        private SearchCriteriaBuilder $criteriaBuilder
    ) {}
}
```

#### 2. Named Arguments
```
// BEFORE — must pass all arguments in order
$order->setStatus('processing', true, false, null);

// AFTER — only required args, self-documenting
$order->setStatus(
    status: 'processing',
    notify: true,
    extend: false,
    comment: null
);
```

#### 3. readonly Properties (PHP 8.1+)
```
// Readonly — cannot be changed after construction
class Address {
    public function __construct(
        public readonly string $street,
        public readonly string $city,
        public readonly string $postcode
    ) {}
}
```

#### 4. First-Class Enums (PHP 8.1+)
```
// BEFORE — constant class with mixed types
class OrderStatus {
    const STATE_PENDING = 'pending';
    const STATE_PROCESSING = 'processing';
}

// AFTER — type-safe enum
enum OrderStatus: string {
    case Pending = 'pending';
    case Processing = 'processing';
    case Complete = 'complete';
    
    public function label(): string {
        return match($this) {
            self::Pending => 'Pending Payment',
            self::Processing => 'Processing',
            self::Complete => 'Complete',
        };
    }
}

// Usage
$status = OrderStatus::Pending;
if ($order->getStatus() === OrderStatus::Processing) {
    // ...
}
```

#### 5. match Expression (PHP 8.0+)
```
// BEFORE — switch statement
switch ($order->getStatus()) {
    case 'pending': $label = 'Pending Payment'; break;
    case 'processing': $label = 'Processing'; break;
    default: $label = 'Unknown';
}

// AFTER — match expression (returns value, no fallthrough)
$label = match($order->getStatus()) {
    'pending' => 'Pending Payment',
    'processing' => 'Processing',
    default => 'Unknown',
};
```

#### 6. Nullsafe Operator (?->)
```
// BEFORE — nested null checks
if ($customer !== null && $customer->getAddress() !== null && $customer->getAddress()->getCity() !== null) {
    $city = $customer->getAddress()->getCity();
}

// AFTER — ?-> short-circuits on null
$city = $customer?->getAddress()?->getCity();
```

#### 7. Union Types and false
```
// PHP 8.0+ — union types
function getOrder(int $id): Order|false {
    // returns Order or false on not found
}

// Usage with match
$result = getOrder(123);
if ($result === false) {
    // handle not found
}
```

#### 8. Enumerator Interfaces (BackedEnum in PHP 8.1+)
```
// Magento uses backed enums extensively in 2.4.8
use Magento\Sales\Model\Order;

// Before — string comparison
if ($order->getState() === 'processing') { ... }

// Modern — enum comparison
use Magento\Sales\Model\OrderInterface;
if ($order->getState() === OrderInterface::STATE_PROCESSING) { ... }

// OR if it's a real PHP enum (not all Magento constants are native enums yet)
```

**Pro Tips:**
- In Magento 2.4.8, look for `enum` keyword in core code — that's a native PHP enum
- `declare(strict_types=1)` at the top of every PHP file enables strict typing
- Mixed types (`mixed $value`) in Magento code mean "could be anything" — handle accordingly
- Readonly classes prevent accidental mutation of value objects — Magento uses this for Address, etc.

**Common Pitfalls:**
- Not knowing about constructor promotion → misreading Magento DI
- Confusing `?->` (nullsafe) with `->` (will crash on null)
- Not handling `false` returns from functions that declare `returnType|false`

---

## Topics

---

### Topic 1: What is Magento?

**What is Magento?**

Magento is a PHP-based e-commerce platform providing catalog management, checkout, pricing, promotions, and a highly customizable module architecture. It powers a significant share of global enterprise e-commerce.

> **Why does this matter?** Magento's module architecture means you're not fighting a monolithic application — you're extending a framework. Understanding this philosophy is key to writing code that doesn't break when Magento updates.

**Editions:**

| Edition | Description | Cost |
|---------|-------------|------|
| Magento Open Source | Core platform | Free |
| Magento Commerce | Enterprise features (ACL, staging, advanced reporting) | Paid |
| Magento Commerce Cloud | Cloud-hosted Commerce | Paid |

This course uses **Magento Open Source 2.4.6+**.

> **Pro Tip:** Magento Commerce (paid) includes B2B features like company accounts, shared catalogs, and quote management. For headless architectures, Open Source is sufficient — you'll expose functionality via GraphQL/REST APIs regardless of edition.

**Request Flow — The Big Picture**

```
HTTP Request → Router → Controller → Result (Page, JSON, Redirect)
```

For page requests, the flow adds Block, Template, and Layout XML:

```
Controller → Block (PHP logic) → Template (PHTML → HTML) → Layout XML → HTTP Response
```

> **Why this matters:** Understanding this chain is critical for debugging. If your page is blank, check: (1) Controller `execute()` returning correctly, (2) Block providing data, (3) Template rendering without errors, (4) Layout XML referencing correct template path.

**Magento Architecture Layers (Preview):**

Magento follows a three-layer architecture:

| Layer | Responsibility | Examples |
|-------|---------------|----------|
| **Presentation** | HTTP handling, layouts, templates, blocks | Controllers, Blocks, PHTML, Layout XML |
| **Domain** | Business logic, service contracts | Models, Interceptors, Value Objects |
| **Infrastructure** | External integrations, persistence | ResourceModels, Repositories, SearchResults |

> **Best Practice:** Keep business logic in the Domain layer (Models/Service classes), not in Controllers. Controllers should be thin — they delegate to services and return Results.

**Key Concepts:**

| Concept | What It Is | Example |
|---------|-----------|---------|
| Module | Package of code adding functionality | `Training_HelloWorld` |
| Theme | Controls visual appearance | Luma, Blank |
| Area | Part of Magento with specific config | `frontend`, `adminhtml`, `graphql` |
| Service Contract | Public API via interfaces | `CustomerRepositoryInterface` |
| Dependency Injection | Magento's way of providing dependencies | Constructor injection |

> **Common Pitfall:** Don't confuse "themes" with "modules". Themes control *appearance* (CSS, templates). Modules control *functionality* (PHP logic). A module can work without a theme, but a theme without a module provides only HTML.

> **PSR-12 Note:** Magento follows PSR-12 coding standard. This means 4-space indentation, opening braces on same line for classes/methods, and strict type declarations. Always run `phpcs --standard=Magento2` before committing.

---

### Topic 2: Directory Structure

```
app/code/               ← Custom modules (your work goes here)
app/design/             ← Themes (frontend/adminhtml)
app/etc/                ← Global config (env.php, config.php)
pub/static/             ← Generated static assets (CSS, JS, images)
pub/index.php           ← Application entry point
var/generation/         ← Generated factory classes and proxies
var/log/                ← Log files (system.log, exception.log)
var/cache/              ← Cache files
vendor/                 ← Composer dependencies
```

> **Why this structure?** Magento was designed for separation of concerns. `app/code/` holds your customizations, `vendor/` holds dependencies managed by Composer. This means `composer install` won't overwrite your custom code — a common mistake in early Magento development.

**Module Structure:**

```
app/code/[Vendor]/[Module]/
├── registration.php        ← Registers module with Magento
├── etc/
│   ├── module.xml          ← Declares module name + version
│   ├── di.xml             ← Dependency injection config
│   ├── frontend/routes.xml ← Frontend routes
│   └── adminhtml/routes.xml ← Admin routes
├── Controller/            ← HTTP request handlers
├── Block/                 ← PHP view logic
├── Model/                 ← Business logic + data
└── View/                  ← Templates and layout XML
```

> **Pro Tip:** The `[Vendor]` naming convention exists because Composer requires vendor names to be unique globally. `Training_HelloWorld` ensures no collision with Magento's `Magento_*` modules or any third-party modules.

> **Best Practice:** Keep your module focused — one module per feature area. Don't create a `Training_Everything` module. Instead: `Training_Review`, `Training_Shipping`, `Training_Api`.

**Key Configuration Files:**

| File | Purpose | Git Status |
|------|---------|------------|
| `env.php` | Database + cache config | **NEVER COMMIT** — contains secrets |
| `config.php` | Module list + configuration | Commit — deployment config |
| `di.xml` | Plugin preferences, virtual types, factories | Commit |
| `events.xml` | Event observers | Commit |
| `webapi.xml` | REST API route definitions | Commit |

> **Critical Security Note:** `env.php` contains database credentials, encryption keys, and service tokens. Adding it to git is a security incident. Ensure your `.gitignore` includes `app/etc/env.php`.

> **Common Pitfall:** Editing files directly in `vendor/` (e.g., to fix a bug in a third-party module). Your changes will be lost on `composer update`. Instead, use a custom module with a `plugin` to override behavior — see Topic 03.

---

### Topic 3: Your First Module

**Minimum Files Required:**

`registration.php` — registers the module with Magento's component system:

```php
<?php
use Magento\Framework\Component\ComponentRegistrar;
ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'Training_HelloWorld',
    __DIR__
);
```

> **Why does registration exist?** Magento's component system needs to discover all modules at runtime. `ComponentRegistrar` registers your module in a global registry that Magento scans on every request. Without this, Magento cannot autoload your classes.

`etc/module.xml` — declares the module and its version:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Training_HelloWorld" setup_version="1.0.0"/>
</config>
```

> **Why version matter?** The `setup_version` tracks your module's schema version. When this number increases, Magento runs upgrade scripts. If you forget to increment it after adding a `db_schema.xml` change, your table won't be created.

**Naming Convention:** Always use `Vendor_ModuleName` format:
- `Training_HelloWorld` ✅
- `TrainingHelloWorld` ❌ (underscore required)
- `training_helloworld` ❌ (case-sensitive — must match `registration.php`)

**First Route — `etc/frontend/routes.xml`:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:App/etc/routes.xsd">
    <router id="standard">
        <route id="helloworld" frontName="helloworld">
            <module name="Training_HelloWorld"/>
        </route>
    </router>
</config>
```

> **Why `frontName` matters:** The `frontName` becomes the first URL segment. Choose carefully — once in production, changing it breaks SEO URLs and external links. Use the module name in snake_case: `helloworld` not `hello_world`.

> **Common Pitfall:** Using the same `frontName` as an existing module or Magento core. This causes routing conflicts. Check Magento's `router List` in `vendor/magento/framework/App/Router/`.

**First Controller — `Controller/Index/Index.php`:**

```php
<?php
declare(strict_types=1);

namespace Training\HelloWorld\Controller\Index;

use Magento\Framework\App\Action\HttpGetActionInterface;
use Magento\Framework\Controller\ResultFactory;

class Index implements HttpGetActionInterface
{
    private ResultFactory $resultFactory;

    public function __construct(ResultFactory $resultFactory)
    {
        $this->resultFactory = $resultFactory;
    }

    public function execute()
    {
        $resultPage = $this->resultFactory->create(ResultFactory::TYPE_PAGE);
        return $resultPage;
    }
}
```

> **PSR-12 Compliance:** `declare(strict_types=1)` is required at the top of every PHP file. Note the `private` property with type hint — this is the modern Magento pattern. The `HttpGetActionInterface` marker tells Magento this controller handles GET requests.

**Result:** Visit `http://localhost/helloworld/index/index`

> **Pro Tip:** The URL follows pattern: `http://[host]/[frontName]/[ControllerPath]/[Action]`. `helloworld/index/index` resolves to `Controller/Index/Index.php`. If your action is `execute()`, why is the URL `.../index/index`? The first `index` is the Controller folder, second `index` is the Action method name.

**Enable Module:**

```bash
bin/magento module:enable Training_HelloWorld
bin/magento setup:upgrade
bin/magento module:status  # verify listed as enabled
```

> **Why `setup:upgrade`?** This command triggers Magento's component registrar to re-scan all modules, run schema/data patches, and regenerate the dependency injection container. After ANY module change (new module, new file in `etc/`, new plugin), you must run this.

> **Best Practice:** In developer mode, Magento auto-generates classes (factories, proxies) on demand. In production mode, you must run `bin/magento setup:di:compile` explicitly. Always use `setup:upgrade` during development — it runs `di:compile` as a subset.

---

### Topic 4: Docker Setup

**Why Docker?**

| Without Docker | With Docker |
|---------------|-------------|
| "It works on my machine" | Consistent across team |
| Manual PHP/MySQL/ES install | One command starts everything |
| Version conflicts | Same versions for all |

> **Why Elasticsearch?** Magento 2.4+ requires Elasticsearch for catalog search. MySQL full-text search was deprecated in 2.4 and removed in 2.4.6. If `elasticsearch` isn't running, product search breaks.

> **Pro Tip:** Use named volumes (`db-data:`, `magento-vendor:`) instead of bind mounts for database and vendor directories. Named volumes persist when you recreate containers, so `composer install` doesn't re-download all dependencies every time.

**Docker Compose File:**

> ⚠️ **IMPORTANT:** The Docker configuration is in the `docker/` directory. Make sure you are in that directory when running Docker commands.

```yaml
version: '3.8'
services:
  app:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    container_name: magento2-app
    volumes:
      - ./src/app/code:/var/www/html/app/code:cached
      - ./src/app/etc:/var/www/html/app/etc:cached
      - ./src/composer.json:/var/www/html/composer.json:cached
      - ./src/generated:/var/www/html/generated:cached
      - ./src/var:/var/www/html/var:cached
      - magento-vendor:/var/www/html/vendor
    ports:
      - "80:80"
    environment:
      - PHP_MEMORY_LIMIT=2G
      - PHP_MAX_EXECUTION_TIME=1800
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
      elasticsearch:
        condition: service_healthy

  db:
    image: mysql:8.0
    container_name: magento2-db
    environment:
      MYSQL_DATABASE: magento
      MYSQL_USER: magento
      MYSQL_PASSWORD: magento
      MYSQL_ROOT_PASSWORD: rootpassword
    volumes:
      - db-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-prootpassword"]
      interval: 10s
      timeout: 5s
      retries: 10

  redis:
    image: redis:7-alpine

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.16
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m

  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    environment:
      - PMA_HOST=db
      - PMA_PORT=3306
      - PMA_USER=magento
      - PMA_PASSWORD=magento
    ports:
      - "8080:80"

volumes:
  db-data:
  magento-vendor:
```

**Setup Commands (Step-by-Step):**

> ⚠️ **IMPORTANT:** Run each command one at a time. Wait for each to complete before running the next.

```bash
# Step 1: Navigate to docker directory
cd docker

# Step 2: Start services (this builds the Docker image first time, takes 3-5 minutes)
docker compose up -d --build

# Step 3: Wait for all services to be ready
# Check that all services show "healthy" or "running"
docker compose ps

# If any service shows "restarting" or "unhealthy", wait 30 seconds and check again
# MySQL takes 30-60 seconds on first start

# Step 4: Verify services are listening on expected ports
curl -s http://localhost:9200 | head -5   # Should return JSON from Elasticsearch
docker compose exec db mysqladmin ping -h localhost -u root -prootpassword  # Should say "mysqld is alive"

# Step 5: Install Composer dependencies (takes 5-15 minutes first time)
docker compose exec app composer install --no-interaction --prefer-dist

# If composer install fails with auth error:
# Create auth.json with your Magento marketplace credentials
# docker compose exec app composer config -g http-basic.repo.magento.com <public_key> <private_key>

# Step 6: Install Magento (creates database and admin user)
docker compose exec app bin/magento setup:install \
  --db-host=db --db-name=magento --db-user=magento --db-password=magento \
  --base-url=http://localhost/ \
  --admin-firstname=Admin --admin-lastname=User \
  --admin-email=admin@example.com \
  --admin-user=admin --admin-password=admin123

# Step 7: Enable developer mode
docker compose exec app bin/magento deploy:mode:set developer

# Step 8: Disable cache during development (speeds up development)
docker compose exec app bin/magento cache:disable

# Step 9: Fix permissions (required for file write operations)
docker compose exec app chmod -R 777 var generated pub/static

# Step 10: Verify installation succeeded
docker compose exec app bin/magento --version
# Expected output: Magento CLI version 2.4.6 (or similar)
```

> **Common Pitfall:** If `setup:install` fails with "Elasticsearch not ready", wait for Elasticsearch health check to pass first. Run `curl http://localhost:9200/_cluster/health` to check status.

> **Best Practice:** After initial setup, commit your `src/` directory (excluding `vendor/`, `generated/`, `var/`). When a new developer clones, they run `docker compose up -d --build && docker compose exec app composer install && docker compose exec app bin/magento setup:upgrade` — everything rebuilds automatically.

---

### Troubleshooting Setup Issues

| Error | Cause | Solution |
|-------|-------|----------|
| `Connection refused` to MySQL | MySQL not ready yet | Wait 60 seconds, run `docker compose ps` |
| `Access denied` for magento user | Wrong password in env.php | Check docker-compose.yml has `MYSQL_PASSWORD=magento` |
| `Elasticsearch not ready` | ES taking long to start | Run `curl http://localhost:9200` to check |
| `Composer install timeout` | Slow network | Try again, or use `--no-dev` flag |
| `Permission denied` on var/ | User mismatch | Run `docker compose exec app chown -R www-data:www-data /var/www/html` |
| `Route returns 404` | Module not enabled | Run `bin/magento setup:upgrade` then `bin/magento cache:flush` |

---

**Daily Docker Commands:**

```bash
cd docker
docker compose up -d              # Start
docker compose down                # Stop
docker compose exec app bash       # Shell into container
docker compose logs -f app         # Watch app logs
docker compose restart app         # Restart app only
```

---

### Topic 5: bin/magento CLI & Logging

**Essential Commands:**

```bash
# Module management
bin/magento module:enable Vendor_Module
bin/magento module:disable Vendor_Module
bin/magento module:status

# Cache
bin/magento cache:enable
bin/magento cache:disable
bin/magento cache:flush           # Clear all cache
bin/magento cache:clean           # Clean cache types

# DI compilation
bin/magento setup:di:compile     # Generate factories, proxies
bin/magento setup:upgrade        # Run DB upgrades + regenerate

# Indexer
bin/magento indexer:reindex
bin/magento indexer:status
```

> **Why `cache:flush` vs `cache:clean`?** `cache:clean` removes only stale cache entries (based on tags). `cache:flush` wipes everything — including valid cache. Use `cache:flush` when debugging (you're not sure what's cached). Use `cache:clean` in deployment scripts (you want to preserve other processes' cache).

> **Pro Tip:** During heavy development, disable all cache: `bin/magento cache:disable`. But remember — every cache disable means Magento hits the database more. If you see slow page loads, it's often because cache is disabled. Re-enable cache types selectively: `bin/magento cache:enable config full_page`

> **Why `setup:di:compile`?** Magento's DI container generates factories for models, proxies for deferred loading, and interceptor classes for plugins. In developer mode, these generate on-demand (slower first request). In production, you must run `setup:di:compile` explicitly — it's part of your CI/CD pipeline.

**Reading Logs:**

```bash
# Watch system log in real-time
tail -f var/log/system.log

# Watch exceptions
tail -f var/log/exception.log
```

> **Best Practice:** Use `tail -f` during development to watch logs in real-time. When debugging a specific flow, run the flow and watch for entries. This is faster than refreshing and checking.

> **Common Pitfall:** Forgetting to check logs when page is blank. `exception.log` almost always contains the error. Check it before restarting your Docker container or changing code randomly.

**Writing Custom Logs:**

```php
<?php
declare(strict_types=1);

use Psr\Log\LoggerInterface;

class MyClass
{
    private LoggerInterface $logger;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function doSomething(int $id): void
    {
        $this->logger->info('Processing review', ['review_id' => $id]);
        try {
            // ... work
        } catch (\Exception $e) {
            $this->logger->error('Failed to process review', [
                'review_id' => $id,
                'error' => $e->getMessage(),
                'trace' => $e->getTraceAsString()
            ]);
            throw $e;
        }
    }
}
```

Log appears in `var/log/training_helloworld.log` (auto-generated from namespace).

> **Why PSR-3 Logger?** Magento uses PSR-3 (PHP Standard Recommendation) for logging. This means you inject `LoggerInterface` (not a concrete class), making your code testable and decoupled. Magento's logger adapts to Monolog under the hood.

> **Pro Tip:** Always include context arrays in log calls. `['review_id' => $id]` is searchable in log aggregation tools (Datadog, Splunk). Raw strings are not.

### Topic 6: Xdebug Setup for Debugging

**Why Xdebug?** Allows step-by-step debugging, breakpoints, variable inspection.

> **When to use Xdebug vs logging?** Xdebug is for understanding code flow and variable state during complex logic. Logging is for tracking execution across requests/background processes and for production debugging. Don't Xdebug a production issue — use logs.

**Enable Xdebug in Docker:**

```bash
# In docker-compose.yml, add to app service:
environment:
  - XDEBUG_MODE=develop,debug
  - XDEBUG_CLIENT_HOST=host.docker.internal
  - XDEBUG_START_WITH_REQUEST=1
```

> **Why `host.docker.internal`?** This special DNS name resolves to the host machine's IP from inside the container. Xdebug connects back to your IDE via this address. On Linux, you may need to add `--add-host=host.docker.internal:host-gateway` to your docker-compose.

**PHPStorm Configuration:**

1. PHP → Servers: Add new (Name: magento2, Host: localhost, Port: 80, Debugger: Xdebug)
2. Use path mappings: `/var/www/html` → your local `src/` directory
3. Enable "Can accept external connections"

> **Common Pitfall:** Forgetting path mappings causes Xdebug to show "waiting for connection" but break on wrong files. The mapping must point IDE path → container path.

**Debug Commands:**

```bash
# Enable Xdebug
docker compose exec app php -d xdebug.mode=develop,debug -r "..."

# Or use browser extension: Xdebug Helper
# Add ?XDEBUG_SESSION=1 to URL to start debugging
```

> **Pro Tip:** Use Xdebug's "Zero Configuration" debugging with the browser helper extension. It automatically starts debugging when you add a breakpoint. No need for `?XDEBUG_SESSION=1`.

---

### Topic 7: Composer Basics

**What is Composer?** PHP's dependency manager. Magento uses it to manage modules and libraries.

> **Why Composer matters in Magento?** Every Magento module is a Composer package. Magento's metapackage (`magento/product-community-edition`) defines which versions of core modules are compatible. Your custom module should also be a Composer package for proper autoloading and deployment via `composer require`.

**Key Commands:**

```bash
composer install      # Install dependencies from composer.lock
composer update       # Update to latest versions (may break things!)
composer require vendor/package  # Add new package
composer show         # List installed packages
composer dump-autoload  # Regenerate autoload files
```

> **Why `composer.lock`?** This file pins exact versions of all dependencies. `composer install` reads it to install identical versions everywhere. `composer update` regenerates it with new versions. Always commit `composer.lock` for reproducible builds.

> **Common Pitfall:** Running `composer update` locally and committing the new `composer.lock`. This forces the same dependency versions on all developers and CI. Only run `composer update` when you intentionally want to upgrade dependencies.

**composer.json Structure:**

```json
{
    "name": "vendor/module-name",
    "type": "magento2-module",
    "require": {
        "php": ">=8.1",
        "magento/framework": "103.0.*"
    },
    "autoload": {
        "files": ["registration.php"],
        "psr-4": {"Vendor\\Module\\": ""}
    }
}
```

> **Why `type: magento2-module`?** This special type tells Composer that this package is a Magento module. Magento's component registrar scans packages with this type for `registration.php`.

> **Best Practice:** Pin your `magento/framework` requirement to a compatible minor version (`103.0.*` not `^103.0`). This allows security patches within the minor version without requiring you to manually update.

**Important:** Never run `composer update` in production. It may update dependencies and break compatibility. In production, use `composer install` with a committed `composer.lock`.

---

## Reading List

- [Magento 2 Architecture](https://developer.adobe.com/commerce/php/architecture/) — Directory structure, module layout
- [Magento 2 Development](https://developer.adobe.com/commerce/php/development/) — Request flow, dependency injection
- [Docker Get Started](https://docs.docker.com/get-started/) — Containers, networking, volumes
- [PSR-12 Coding Standard](https://www.php-fig.org/psr/psr-12/) — PHP formatting rules Magento enforces
- [Magento Coding Standard](https://developer.adobe.com/commerce/php/coding-standards/) — Magento-specific rules

---

## Edge Cases & Troubleshooting

| Issue | Symptom | Solution |
|-------|---------|----------|
| Module not showing | `module:status` doesn't list it | Check `registration.php` spelling and path |
| Route 404 | Page returns 404 | Check `routes.xml` frontName matches controller |
| Blank page | White screen | Check `var/log/exception.log` |
| Permission denied | Cannot write to var/ | `chmod -R 777 var generated` |
| Docker not starting | Port 8080 already in use | Stop other services using port 8080 |
| MySQL connection fail | Cannot connect to DB | Docker services running? `docker compose ps` |
| Cache stale | Old content showing | `bin/magento cache:flush` |
| Elasticsearch fail | Search returns no results | Check ES health: `curl localhost:9200/_cluster/health` |
| PHP memory limit | Fatal error: memory exhausted | Increase `PHP_MEMORY_LIMIT` in docker-compose.yml |
| Admin login loop | Redirects back to login | Check `env.php` session config, try `bin/magento cache:flush` |

---

## Common Mistakes to Avoid

1. ❌ Forgetting `bin/magento setup:upgrade` after creating a new module → Module doesn't load
2. ❌ Skipping `bin/magento cache:flush` after changes → Old content persists
3. ❌ Editing files outside `app/code/` → Changes lost on `composer install`
4. ❌ Using production mode during development → Slow, can't override templates
5. ❌ Committing `env.php` → Contains secrets, never commit this file
6. ❌ Editing core/vendor files to fix bugs → Use plugins/observers instead (Topic 03/05)
7. ❌ Running `composer update` on a running project → Can update dependencies unexpectedly
8. ❌ Forgetting `chmod -R 777 var generated` after Docker setup → File permission errors on Linux
9. ❌ Using same `frontName` as existing module → Routing conflicts
10. ❌ Missing `declare(strict_types=1)` in PHP files → PSR-12 violation, type safety broken

---

*Magento 2 Backend Developer Course — Topic 01*
