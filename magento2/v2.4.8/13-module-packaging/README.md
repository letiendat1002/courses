# Topic 13: Module Packaging & Release

**Philosophy:** Take your module from working code to production-ready release.

---

## Overview

You've built a module — now learn how to package it properly for distribution via Composer, prepare it for the Magento Marketplace, and manage versions professionally.

---

## Prerequisites

- [ ] Topic 05 completed (module structure, di.xml, service contracts)
- [ ] Working module with Service Contracts
- [ ] Git repo set up

---

## Learning Objectives

By end of this module, you will be able to:

- [ ] Write a proper `composer.json` for a Magento module
- [ ] Configure autoload for PSR-4 and file mappings
- [ ] Apply semantic versioning to module releases
- [ ] Prepare a module for Magento Marketplace submission
- [ ] Write a README with installation instructions
- [ ] Set up basic CI/CD for module testing

---

## Day 1: Composer Packaging

### Content

#### 1.1 Module composer.json Structure

```json
{
    "name": "vendor/module-name",
    "description": "Short description of what the module does",
    "type": "magento2-module",
    "license": [
        "OSL-3.0",
        "AFL-3.0"
    ],
    "autoload": {
        "files": [
            "registration.php"
        ],
        "psr-4": {
            "Vendor\\ModuleName\\": ""
        }
    },
    "require": {
        "php": ">=8.1",
        "magento/framework": "103.0.*"
    },
    "require-dev": {
        "magento/magento-coding-standard": "*"
    },
    "scripts": {
        "post-install-cmd": [
            "Magento\\CodingStandard\\Hook\\Handler\\PostInstall::run"
        ],
        "test": "phpcs --standard=Magento2 ./src"
    }
}
```

#### 1.2 Key Fields Explained

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | Yes | `vendor/module-name` — must be unique on Packagist |
| `type` | Yes | Must be `magento2-module` for Magento to recognize |
| `description` | Yes | Short clear description |
| `license` | Yes | OSL-3.0 or AFL-3.0 for Magento modules |
| `autoload.psr-4` | Yes | PSR-4 namespace mapping |
| `require` | Yes | PHP version and Magento dependencies |

#### 1.2.1 Why `type: "magento2-module"` Matters

The `type` field is not just metadata — it is a **Composer plugin trigger**. When you run `composer install` or `composer update`, Composer's plugin system scans installed packages for type `magento2-module`. When found, Magento's `Magento\Framework\Component\ComponentRegistrar` reads the module's `registration.php` and registers the module with Magento's module list.

Without `type: "magento2-module"`, the `registration.php` file is never executed, and Magento will not see the module — even if all files are in the correct location. This is a common mistake when engineers copy a module's structure but forget the type field, leaving them confused why the module doesn't appear in `bin/magento module:list`.

#### 1.2.2 `autoload` Deep Dive

The `autoload` section tells Composer how to resolve classes in your module:

```json
"autoload": {
    "files": ["registration.php"],
    "psr-4": {
        "Vendor\\ModuleName\\": ""
    }
}
```

- **`files`**: `registration.php` is loaded on every Composer operation. It must be present and correctly register the module via `ComponentRegistrar::register()`.
- **`psr-4`**: Maps the `Vendor\ModuleName\` namespace prefix to the module root directory (empty string means "root of the package"). This enables autoloading without any `include` statements.

> **Pro Tip:** You can also add `"classmap"` autoloading for legacy patterns, but PSR-4 is preferred and required for new modules per Magento coding standards.

#### 1.2.3 `require` — Version Constraints That Matter

Magento's version constraints follow specific patterns:

```json
"require": {
    "php": ">=8.1",
    "magento/framework": "103.0.*",
    "magento/module-customer": "104.0.*"
}
```

| Constraint | Meaning | Use Case |
|------------|---------|----------|
| `103.0.*` | Any 103.0.x version | Allows patch updates only |
| `^103.0` | >=103.0.0 and <104.0.0 | Allows minor and patch updates (recommended) |
| `~103.0` | >=103.0.0 and <104.0.0 | Same as `^` for semantic versioning |
| `>=103.0,<104.0` | Explicit range | Use when you need fine control |

For Magento modules, `103.0.*` (exact minor version) is standard because Magento's minor versions carry breaking changes in the framework. Matching the minor version ensures compatibility without unexpected breakage.

> **Common Pitfall:** Using `*` or `>=1.0.0` as a version constraint for `magento/framework` will pull the latest version which may have API changes. Always pin to the Magento version you tested against.

#### 1.2.4 `require-dev` — Development Dependencies

```json
"require-dev": {
    "magento/magento-coding-standard": "*",
    "phpunit/phpunit": "^9.5"
}
```

- `magento/magento-coding-standard` — Enforces Magento-specific code style rules (PSR-12 plus Magento conventions). Run with: `vendor/bin/phpcs --standard=Magento2 ./app/code/Vendor/Module`
- `phpunit/phpunit` — Unit testing framework. Magento 2.4+ uses PHPUnit 9.x.

> **Pro Tip:** Pin the PHPUnit version to what your Magento version ships with. Using PHPUnit 10 with Magento 2.4.6 will cause compatibility issues with the built-in test harness.

#### 1.3 Marketplace Requirements

- [ ] README with: description, installation steps, screenshots, changelog
- [ ] `composer.json` must declare correct version constraints
- [ ] Module must not override core files (use plugins/observers)
- [ ] Must pass Marketplace Technical Review (PHPCS, security scan)

#### 1.3.1 Magento Marketplace Technical Review — What Gets Checked

Before your module is listed on the Magento Marketplace, it undergoes automated and manual review. The automated checks include:

| Check | Tool | What It Scans |
|-------|------|---------------|
| Code style | `phpcs --standard=Magento2` | PSR-12 violations, forbidden functions, naming conventions |
| Security scan | Static analysis | SQL injection patterns, XSS vectors, exposed credentials |
| Architecture | Manual review | No core class overrides, no direct DB writes outside module schema |
| Composer validation | `composer validate --strict` | schema compliance, version constraints |

**Common reasons modules fail technical review:**

1. Using `ObjectManager` directly instead of constructor injection (caught by PHPCS rule `MagentoFrameworkObjectManagerUsage`)
2. Including development dependencies in the `require` section instead of `require-dev`
3. Having a `composer.lock` file in the package (must not be committed)
4. Missing `registration.php` or incorrect registration path
5. Overriding a core class instead of using a plugin (violates the "no overrides" rule)

#### 1.3.2 Preparing for Submission — The Submission Checklist

Before clicking "Submit for Review":

```bash
# 1. Validate composer.json schema
composer validate --strict

# 2. Run code style — ZERO errors allowed
vendor/bin/phpcs --standard=Magento2 app/code/Vendor/ModuleName --severity=10

# 3. Check for forbidden functions
vendor/bin/phpcs --standard=Magento2 --ruleset=Magento2/ruleset.xml \
  app/code/Vendor/ModuleName --exclude=MagentoFrameworkObjectManagerUsage

# 4. Verify registration.php is correct
grep -n "ComponentRegistrar::register" app/code/Vendor/ModuleName/registration.php

# 5. Check no hardcoded passwords or API keys in code
grep -rn "password\s*=" app/code/Vendor/ModuleName --include="*.php"
grep -rn "api[_-]key" app/code/Vendor/ModuleName --include="*.php"

# 6. Ensure README is complete with:
#    - Module description
#    - Installation steps
#    - Screenshots (for admin configuration pages)
#    - Changelog
```

#### 1.3.3 Metapackage Patterns — When to Use `metapackage` Type

If your module is actually a bundle of multiple modules (common for complex extensions), use the `metapackage` type:

```json
{
    "name": "vendor/module-bundle",
    "description": "Bundle containing modules A, B, and C",
    "type": "metapackage",
    "require": {
        "vendor/module-a": "1.0.0",
        "vendor/module-b": "1.0.0",
        "vendor/module-c": "1.0.0"
    }
}
```

A `metapackage` contains no code itself — it only declares dependencies. Installing it pulls in all child modules as a single unit. This is useful when you want to distribute a suite of modules that must be installed together.

> **Pro Tip:** If your modules can work independently, do NOT bundle them. Each module should be installable on its own. Only use `metapackage` when there is a hard dependency between modules that cannot be broken.

### Exercise 1.1: Write composer.json

Convert your training module into a properly packaged Composer module.

---

## Day 2: Versioning, Release & CI/CD

### Content

#### 2.1 Semantic Versioning (SemVer)

Given a version number `MAJOR.MINOR.PATCH`:

| Part | Increment when | Example |
|------|---------------|---------|
| **MAJOR** | Breaking changes to API | 1.0.0 → 2.0.0 |
| **MINOR** | New functionality (backward compatible) | 1.0.0 → 1.1.0 |
| **PATCH** | Bug fixes (backward compatible) | 1.0.0 → 1.0.1 |

**Magento-specific rules:**
- Always match `major.minor` with Magento framework version (e.g., `103.0.*` for Magento 2.4.6)
- Never break Service Contracts in MINOR updates

#### 2.1.1 Why Service Contracts Must Be Sacred in MINOR Updates

Service Contracts (interfaces in `Api/`) are the public API of your module. Any code that depends on your module uses these interfaces. When you change an interface in a MINOR update (adding a required parameter, changing a return type, removing a method), you break every consumer.

**Breaking changes to Service Contracts include:**
- Removing or renaming an interface method
- Changing a method signature (adding required parameters, changing return type)
- Changing a DTO property (removing fields, changing types in `Data/` interfaces)
- Changing an interface constant value

**Non-breaking changes (safe for MINOR):**
- Adding new methods with default implementations
- Adding new optional parameters to existing methods
- Adding new DTO properties (consumers ignore unknown properties)
- Adding new interfaces

> **Common Pitfall:** Renaming a repository method from `getById()` to `findById()` is a **breaking change**. Even though it "improves naming," it will break every consumer. Use deprecation with a `@deprecated` annotation and keep the old method with a forwarding implementation.

#### 2.1.2 Magento Version Alignment

Magento's version numbers and your module's `composer.json` must be aligned:

| Magento Version | Framework Constraint | Module Version Pattern |
|-----------------|---------------------|----------------------|
| 2.4.6 | `103.0.*` | `1.0.0` (independent) |
| 2.4.5 | `102.0.*` | `1.0.0` (independent) |
| 2.4.4 | `101.0.*` | `1.0.0` (independent) |

Your module version (e.g., `1.0.0`) is independent of Magento's version. The `require` constraint maps what Magento versions your module supports.

```json
{
    "name": "vendor/module-name",
    "require": {
        "magento/framework": "^103.0"
    }
}
```

The `^103.0` constraint means your module works with any Magento 2.4.6+ installation (which uses framework 103.x).

#### 2.1.3 Deprecation Strategy — How to Remove Things Gracefully

When you need to remove a method in a future version, use a deprecation ladder:

```php
<?php
/**
 * @deprecated 1.2.0 — Use getById() instead. Will be removed in 2.0.0.
 */
public function find(int $id): ?OrderInterface
{
    return $this->getById($id);
}
```

This allows consumers to update at their own pace while you signal the eventual removal.

#### 2.2 Release Workflow

```bash
# 1. Create release branch
git checkout -b release/1.1.0

# 2. Update composer.json version
# 3. Update CHANGELOG.md
git add CHANGELOG.md composer.json
git commit -m "Release 1.1.0"

# 4. Tag
git tag -a 1.1.0 -m "Release 1.1.0"
git push origin 1.1.0

# 5. Create GitHub Release
gh release create 1.1.0 --title "Release 1.1.0" --notes "See CHANGELOG"
```

#### 2.3 Simple CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  phpcs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install Composer
        run: composer install
      - name: Run PHPCS
        run: vendor/bin/phpcs --standard=Magento2 app/code/Vendor/Module
```

#### 2.3.1 Production-Grade CI Pipeline for Magento Modules

A real CI pipeline should cover more than just code style:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
          tools: composer, phpcs, phpstan

      - name: Install dependencies
        run: composer install --no-interaction

      - name: PHPStan Static Analysis
        run: vendor/bin/phpstan analyse app/code/Vendor/Module \
               --level=5 --no-progress

      - name: PHPCS Code Style
        run: vendor/bin/phpcs --standard=Magento2 \
               --severity=10 app/code/Vendor/ModuleName

      - name: Check for forbidden functions
        run: vendor/bin/phpcs --standard=Magento2 \
               --exclude=Generic.Files.LineLength app/code/Vendor/ModuleName

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Security scan
        run: |
          # Check for common security issues
          grep -rn "eval(" app/code/Vendor/ModuleName --include="*.php"
          grep -rn "base64_decode\|base64_encode" \
            app/code/Vendor/ModuleName --include="*.php"
          grep -rn "\$_GET\|\$_POST\|\$_REQUEST" \
            app/code/Vendor/ModuleName --include="*.php" \
            | grep -v "RequestInterface"

  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
          coverage: pcov
      - name: Install
        run: composer install && composer require --dev \
               phpunit/phpunit:^9.5 --no-interaction
      - name: Run tests
        run: vendor/bin/phpunit -c app/code/Vendor/ModuleName/Test/phpunit.xml
```

#### 2.3.2 Repository Configuration — Private Module Distribution

If your module is private (not on Packagist), configure a private repository:

```json
{
    "name": "mycompany/module-internal",
    "repositories": [
        {
            "type": "composer",
            "url": "https://repo.mycompany.com/packages/",
            "canonical": false
        }
    ],
    "require": {
        "php": ">=8.1",
        "magento/framework": "^103.0"
    }
}
```

On the server, authenticate with:

```bash
composer config --global --auth repo.mycompany.com Bearer TOKEN_HERE
```

Or use `auth.json`:

```json
{
    "http-basic": {
        "repo.mycompany.com": {
            "username": "token",
            "password": "Bearer TOKEN_HERE"
        }
    }
}
```

> **Pro Tip:** Never commit `auth.json` to version control. Add it to `.gitignore` and provide a template (`auth.json.example`) with placeholder values.

#### 2.3.3 Git Tagging Strategy

For module repositories, use annotated tags with semantic versions:

```bash
# Create an annotated tag (preferred)
git tag -a 1.0.0 -m "Release 1.0.0: Initial GA release"

# Push the tag
git push origin 1.0.0

# Or push all tags
git push --tags
```

When you create a GitHub Release from a tag, Composer can automatically package and distribute the module via a distribution URL or by pointing to a ZIP archive.

### Exercise 2.1: Release Your Module

1. Write proper composer.json
2. Tag version 1.0.0
3. Push to GitHub
4. Create a GitHub Release

---

### Topic X: CI/CD Pipeline — From Code to Production

**Why This Matters:** Writing code is 20% of the job. Getting it to production reliably is the other 80%. A proper CI/CD pipeline prevents bad code from reaching production, automates testing, and makes deployments boring.

**The Pipeline Stages:**

```
Commit → Lint (PHPCS) → Unit Tests → Build → Deploy to Staging → Integration Tests → Deploy to Production
```

**GitHub Actions CI Pipeline — Full Setup:**

**.github/workflows/ci.yml:**

```yaml
name: Magento Module CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint:
    name: PHPCS Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: dom, curl, gd, intl, mbstring, mysql, zip
          coverage: xdebug
      
      - name: Install Composer dependencies
        run: composer install --no-interaction --prefer-dist
      
      - name: PHPCS Check
        run: vendor/bin/phpcs --standard=Magento2 --extensions=php --ignore=vendor,node_modules,var app/

  unit-test:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: dom, curl, gd, intl, mbstring, mysql, zip
          coverage: xdebug
      
      - name: Install dependencies
        run: composer install --no-interaction --prefer-dist
      
      - name: Run Unit Tests
        run: vendor/bin/phpunit --testsuite=Unit --testdox
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./var/coverage/html/index.html

  integration-test:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: unit-test
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: magento_test
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s
      
      redis:
        image: redis:7
        options: --health-cmd="redis-cli ping" --health-interval=10s --health-timeout=5s
      
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: dom, curl, gd, intl, mbstring, mysql, zip
          coverage: xdebug
      
      - name: Install dependencies
        run: composer install --no-interaction --prefer-dist
      
      - name: Generate TEST_PHP_EXECUTABLE
        run: ./vendor/bin/phpunit --generate-config-test
      
      - name: Run Integration Tests
        run: vendor/bin/phpunit --testsuite=Integration --testdox

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: integration-test
    if: github.ref == 'refs/heads/main'
    environment: staging
    steps:
      - name: Deploy via rsync
        uses: burnett01/rsync-deploy@master
        with:
          switches: -avzr --delete
          path: ./
          target: ${{ secrets.STAGING_USER }}@${{ secrets.STAGING_HOST }}:/var/www/magento/
          key: ${{ secrets.STAGING_SSH_KEY }}
      
      - name: Run Magento commands
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /var/www/magento
            bin/magento maintenance:enable
            bin/magento setup:upgrade --no-interaction
            bin/magento setup:di:compile
            bin/magento setup:static-content:deploy
            bin/magento cache:flush
            bin/magento maintenance:disable
```

**Deployment Best Practices:**

1. **Zero-downtime deploy pattern:**
   ```
   1. Enable maintenance mode
   2. Backup current deployment
   3. rsync new code
   4. setup:upgrade + di:compile
   5. static-content:deploy
   6. cache:flush
   7. Disable maintenance mode
   ```
   
   For truly zero-downtime (blue-green):
   ```
   1. Deploy to new directory (magento_new)
   2. Point nginx to magento_new
   3. Old version continues serving
   4. Switch is instant
   5. Rollback: flip nginx back to magento_old
   ```

2. **Composer autoload optimization before deploy:**
   ```bash
   composer dump-autoload -o --no-dev
   ```
   Production should use `--no-dev` (no dev dependencies in production)

3. **Environment variables in GitHub Secrets:**
   ```
   STAGING_HOST, STAGING_USER, STAGING_SSH_KEY
   PRODUCTION_HOST, PRODUCTION_USER, PRODUCTION_SSH_KEY
   ```

**Pro Tips:**
- Always run `setup:di:compile` before static-content:deploy — the compiled DI is needed for generation
- Static content deploy should NOT run on every deploy — only when JS/layout changes, not on every code change
- Use semantic versioning tags to mark releases: `git tag v1.2.3 && git push --tags`
- Composer lock should be committed but vendor/ should be in .gitignore
- Use a dedicated deploy user with minimal permissions (not root)

---

## Module Checklist

Before releasing your module, verify:

- [ ] `composer.json` is valid (`composer validate`)
- [ ] `registration.php` present and correct
- [ ] `etc/module.xml` has correct version
- [ ] CHANGELOG.md updated
- [ ] README has installation instructions
- [ ] PHPCS passes zero errors
- [ ] No hardcoded values (use di.xml / config.yaml)
- [ ] Service Contracts defined in `Api/` folder
- [ ] Tests written (if applicable)

---

## Common Mistakes to Avoid

### 1. Forgetting `type: "magento2-module"` in composer.json

This causes the module to be invisible to Magento. Always verify the type field is present.

### 2. Using `require` instead of `require-dev` for testing packages

Development dependencies (phpcs, phpunit) must be in `require-dev`. Putting them in `require` means they get installed in production, bloating the deployment and potentially conflicting with the main application's dependencies.

### 3. Hardcoding passwords or API keys

Never hardcode credentials in PHP files, XML configs, or anywhere in version control. Use environment variables via `$_ENV` or Magento's configuration system (`Magento\Framework\Config\Filesystem::getAppConfigPath()`).

### 4. Using `ObjectManager` directly in constructors

```php
// WRONG
public function __construct()
{
    $this->resource = \Magento\Framework\App\ObjectManager::getInstance()
        ->get(\Magento\Framework\App\ResourceConnection::class);
}

// CORRECT (constructor injection)
public function __construct(
    \Magento\Framework\App\ResourceConnection $resource
) {
    $this->resource = $resource;
}
```

PHPCS rule `MagentoFrameworkObjectManagerUsage` will flag direct `ObjectManager` usage.

### 5. Committing `composer.lock`

Never commit `composer.lock` for a module. The `composer.lock` pins exact versions of dependencies but is specific to the environment where `composer install` was last run. For a distributable module, let the consumer's environment resolve dependencies.

### 6. Breaking Service Contracts in MINOR updates

As noted above, interfaces in `Api/` are public API. Adding a new method is fine; changing or removing an existing one is a MAJOR version bump.

### 7. Not specifying PHP version constraint

Without `"php": ">=8.1"` in `require`, Composer may try to install the module on PHP 7.x where it will fail at runtime. Always specify the minimum PHP version.

### 8. Missing or incorrect `registration.php`

The `registration.php` file is what tells Magento's component registrar about your module. A missing or misnamed namespace here is the #1 reason modules fail to appear in `bin/magento module:list`.

Correct structure:

```php
<?php

use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'Vendor_ModuleName',
    __DIR__
);
```

### 9. Releasing without testing against the target Magento version

Your module's `composer.json` may say it supports Magento `^103.0`, but if you only tested on `103.0.0` and the consumer installs `103.0.6` (a patch with possible behavioral changes), problems may arise. Always test against the oldest and newest patch version in your supported range.

---

## Reference Links

- [Composer.json schema](https://getcomposer.org/doc/04-schema.md)
- [Magento Marketplace](https://marketplace.magento.com/)
- [Semantic Versioning](https://semver.org/)
- [Magento coding standard](https://github.com/magento/magento-coding-standard)

---

## Topics Covered

| Topic | Description |
|-------|-------------|
| Day 1 | Composer Packaging — `composer.json` structure, autoload, Marketplace requirements |
| Day 2 | Versioning, Release & CI/CD — SemVer, release workflow, basic CI pipeline |
| Topic X | CI/CD Pipeline — Full GitHub Actions pipeline with staging/production deployment |

---

## Completion Criteria

By completing this topic, you have demonstrated ability to:

- [x] Write a proper `composer.json` for a Magento module
- [x] Configure autoload for PSR-4 and file mappings
- [x] Apply semantic versioning to module releases
- [x] Prepare a module for Magento Marketplace submission
- [x] Write a README with installation instructions
- [x] Set up CI/CD pipeline with GitHub Actions
- [x] Implement zero-downtime deployment patterns
- [x] Configure staging and production deployment workflows

---

*Magento 2 Backend Developer Course — Topic 13 | Module Packaging & Release*
