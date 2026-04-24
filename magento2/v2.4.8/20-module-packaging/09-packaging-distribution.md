---
title: "13 - Packaging Distribution"
description: "Deep dive into Magento 2.4.8 module packaging and distribution: composer.json, marketplace preparation, semantic versioning, and release management"
tags:
  - magento2
  - packaging
  - composer
  - marketplace
  - release
rank: 9
pathways:
  - magento2-deep-dive
---

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

## Day N: Extension Quality — Beyond the Basics

The patterns in this section go beyond packaging syntax. They cover the
conflict-detection, schema-design, and CI-hardening decisions that separate
a professional extension from a hobby-grade one. Each subsection includes
real configuration snippets you can paste into your module.

> **Audience.** If you are already running PHPCS and PHPUnit in CI, this
> section adds the next layer: static analysis with PHPStan, sequence-aware
> dependency design, preference hygiene, schema strategy, and composer
> conflict resolution.

---

### 1. PHPStan in GitHub Actions — Levels, Baseline, and No-Progress

[PHPStan](https://phpstan.org/) catches type errors, undefined methods, and
missing return types before runtime. The key to a sustainable workflow is
choosing the right level and maintaining a `phpstan-baseline.neon` file that
flags known issues without silencing new ones.

#### Choosing a Level

| Level | What it catches                          | Best for              |
|-------|----------------------------------------|-----------------------|
| 0     | Basic type errors, unknown classes     | New modules / messy code |
| 1     | ± method signatures, property types    | Modules in active dev |
| 2     | ± nullable types, strict void returns  | Recommended default   |
| 3     | ± dead code, incorrect instanceof      | Mature modules        |
| 4     | ± argument type variance              | Library-quality code  |
| 5     | ± fully strict types                   | Target eventually     |

Start at level 1, move to level 2 after the baseline is clean. Avoid jumping
to level 5 on day one — the noise discourages the team.

#### GitHub Actions Pipeline

```yaml
# .github/workflows/phpstan.yml
name: PHPStan Static Analysis

on:
  push:
    paths:
      - '**.php'
      - '.phpstan.neon'
      - 'phpstan-baseline.neon'
  pull_request:
    paths:
      - '**.php'

jobs:
  phpstan:
    name: PHPStan (level ${{ matrix.phpstan-level }})
    runs-on: ubuntu-latest

    strategy:
      matrix:
        phpstan-level: [1, 2]

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          coverage: none
          tools: phpstan

      - name: Cache vendor
        uses: actions/cache@v3
        with:
          path: vendor
          key: phpstan-vendor-${{ runner.os }}-${{ hashFiles('composer.lock') }}
          restore-keys: |
            phpstan-vendor-${{ runner.os }}-

      - name: Install dependencies
        run: composer install --no-interaction --prefer-dist

      - name: Run PHPStan
        run: phpstan analyse \
          --configuration=.phpstan.neon \
          --level=${{ matrix.phpstan-level }} \
          --no-progress \
          --error-format=github
        env:
          GITHUB_OUTPUT: /dev/stderr
```

> The `--no-progress` flag suppresses the progress bar which otherwise
> corrupts the GitHub Actions log output. The `--error-format=github`
> flag emits structured annotations that GitHub can render as inline
> file comments on pull requests.

#### phpstan.neon Configuration

```neon
# .phpstan.neon
include:
  - vendor/phpstan/phpstan/conf/bleedingEdge.neon
  - phpstan-baseline.neon   # generated once; commit it

parameters:
  phpVersion: 80200
  paths:
    - Model/
    - Controller/
    - etc/
  excludePaths:
    - vendor/
  ignoreErrors:
    - '#Call to an undefined method.*#'
    - '#Access to undefined property#'
  reportUnmatchedIgnoredErrors: false
  inferPrivatePropertyTypeFromConstructor: true
```

> **Bleeding edge.** The `bleedingEdge.neon` include activates the latest
> PHPStan rules even before a stable release. If a rule break hits CI, move
> the include to a separate profile and gate it behind a separate job until
> you have fixed the issues.

#### Generating and Maintaining the Baseline

Run PHPStan once at the highest level you target, then generate a baseline
of all current failures so that you can enforce a clean level going forward:

```bash
# Generate baseline for level 2 from scratch
vendor/bin/phpstan analyse --level=2 --no-progress \
  --generate-baseline=phpstan-baseline.neon
```

After generation, commit `phpstan-baseline.neon`. Each time you fix a real
issue, remove the corresponding entry from the baseline rather than adding
a new `ignoreErrors` glob. This keeps the baseline honest.

> **Tip.** Run PHPStan against PHP 8.2 in CI even if your module advertises
> `php: ^7.4`. PHPStan's type inference is sharper on 8.2, and Composer will
> still enforce your declared `php` constraint at install time.

---

### 2. Sequence Conflicts — `module.xml` Loading Order

Magento resolves module loading order via the `<sequence>` block in each
module's `etc/module.xml`. If two modules both extend the same class or
override the same layout handle, the sequence determines which runs first.
Getting it wrong causes cryptic `Class already exists` or layout handle
not found errors.

#### Anatomy of a Sequence Declaration

```xml
<!-- app/code/Vendor/ModuleA/etc/module.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Vendor_ModuleA" setup_version="1.0.0">
        <sequence>
            <!-- ModuleA depends on Magento_Catalog and Magento_Checkout -->
            <module name="Magento_Catalog"/>
            <module name="Magento_Checkout"/>
        </sequence>
    </module>
</config>
```

When `Vendor_ModuleA` is active, Magento loads `Magento_Catalog` and
`Magento_Checkout` first, then `Vendor_ModuleA`. This guarantees that any
class `Vendor_ModuleA` tries to extend or plugin has already been registered.

#### Circular Sequence Detection

Magento does **not** warn you about circular dependencies at registration
time. A cycle like `A → B → C → A` silently breaks the store — the
component registrar accepts all three modules, but the DI container fails
at `bin/magento setup:di:compile` with an "area code not loaded" or
"circular dependency" error.

Detect circles locally with this one-liner before pushing:

```bash
# Detect circular <sequence> references
php -r '
$dir = __DIR__ . "/app/code";
$modules = glob("$dir/*/*/etc/module.xml");
$graph = [];
foreach ($modules as $m) {
    $xml = simplexml_load_file($m);
    $name = (string) $xml->module["name"];
    $seq  = [];
    foreach ($xml->module->sequence->module as $dep) {
        $seq[] = (string) $dep["name"];
    }
    $graph[$name] = $seq;
}
foreach ($graph as $mod => $deps) {
    foreach ($deps as $dep) {
        if (isset($graph[$dep]) && in_array($mod, $graph[$dep])) {
            echo "CIRCULAR: $mod -> $dep -> $mod\n";
        }
    }
}
'
```

Add this as a `bin/circular-check` script or a Composer script:

```json
{
    "scripts": {
        "circular-check": "php bin/circular-check"
    }
}
```

#### Common Sequence Mistakes

| Mistake                           | Symptom                                      | Fix                                           |
|-----------------------------------|----------------------------------------------|-----------------------------------------------|
| Missing sequence for a core module | Plugin not firing, class not found            | Add missing `<module name="Magento_XXX"/>`  |
| Ordering two third-party modules   | One plugin override never fires              | Use `di.xml` `<type>` preference or sequence |
| Sequence only on frontend          | Admin functionality broken after upgrade     | Declare sequences in all `webapi_rest` scopes |
| Forgotten module in sequence       | UpgradeData script runs before its schema    | Alphabetically sort all sequence entries      |

> **Rule of thumb.** If your module extends, plugins, or observers any
> class from another module, that module must appear in `<sequence>`.
> If the other module is a third-party extension, verify its `module.xml`
> to confirm the exact namespace string.

---

### 3. Preference Conflicts in `di.xml`

`<preference>` in `di.xml` tells Magento which implementation to inject
when a class or interface is requested. The most common mistake is
declaring two `<preference>` entries for the same class — the second one
silently wins at runtime, making debugging painful.

#### Detecting Duplicate Preferences with `di:compile`

Magento's DI compiler catches duplicate preferences at compile time:

```bash
bin/magento setup:di:compile
```

If a conflict exists, you will see:

```
  [Magento\Framework\Exception\LocalizedException]
  Preference forVendor_ModuleA\DataSource\RealImplementation
  is already defined in Vendor_ModuleB\RealImplementation
```

If you do not have CLI access, grep the codebase for duplicate `<preference>`

```bash
# Find all <preference> declarations for the same class
grep -rh '<preference for=' app/code/ | sort | uniq -d
```

#### Plugin-over-Preference Strategy

When two modules need to modify the same class, prefer a **plugin** over
a `<preference>` whenever possible. Plugins are composable; preferences
are mutually exclusive. The only legitimate use of `<preference>` is
swapping a concrete class with a custom implementation.

```xml
<!-- WRONG: Two modules preference the same class -->
<!-- Vendor_ModuleA/etc/di.xml -->
<preference for="Magento\Catalog\Api\ProductRepositoryInterface"
           type="Vendor_ModuleA\ProductRepository"/>

<!-- Vendor_ModuleB/etc/di.xml -->
<preference for="Magento\Catalog\Api\ProductRepositoryInterface"
           type="Vendor_ModuleB\ProductRepository"/>
<!-- B wins silently at runtime. A never sees its changes. -->
```

```xml
<!-- CORRECT: Use plugins to intercept, preference to replace -->
<!-- Vendor_ModuleA/etc/di.xml -->
<preference for="Magento\Catalog\Api\ProductRepositoryInterface"
           type="Vendor_ModuleA\ProductRepository"/>

<!-- Vendor_ModuleA/etc/di.xml (plugin that composes around the replacement) -->
<type name="Vendor_ModuleA\ProductRepository">
    <plugin name="decorateWithAuditLog"
            type="Vendor_ModuleA\Plugin\ProductRepositoryAuditPlugin"/>
</type>
```

#### Preference Resolution Table

| Situation                                    | Recommended approach               |
|----------------------------------------------|------------------------------------|
| Replace core class with custom impl          | `<preference>`                     |
| Add logging/hooks to existing class          | Plugin (before/after)              |
| Two modules replace same interface           | Extract interface, each preference  |
| Third-party module replaces same class       | Use `<plugin>` on the third-party  |
| Need to swap at specific scope only          | `<type>` scoped di.xml in area dir  |

---

### 4. Declarative Schema vs Data Patches

Magento 2.3 introduced *Declarative Schema* (`db_schema.xml`) as a
replacement for PHP-based upgrade scripts. Magento 2.3.5+ added *Data
Patches* as a bridge between the two worlds. Choosing the right tool
avoids rollback nightmares and keeps your module upgrade-safe.

#### Decision Table: When to Use db_schema.xml vs Data Patch

| Criterion                            | db_schema.xml                              | Data Patch / UpgradeData Script              |
|--------------------------------------|--------------------------------------------|---------------------------------------------|
| New table                            | ✅ Yes                                      | ⚠️ Possible but verbose                     |
| Drop table                           | ✅ Yes (with `on="remove"`)               | ⚠️ Not recommended — use schema patch       |
| Add column (non-destructive)          | ✅ Yes                                      | ✅ Yes                                       |
| Rename column                        | ✅ Yes (with `rename` attribute)          | ⚠️ Data must be migrated manually            |
| Change column type (destructive)     | ⚠️ Warning — causes data loss             | ✅ Safer — add transform in patch            |
| Insert default data                  | ⚠️ Limited — use `default` attribute       | ✅ Full PHP control over initial data         |
| Conditional data changes             | ❌ No                                       | ✅ Yes — use `if` logic                     |
| DDL trigger needed                   | ❌ No                                       | ✅ Yes — raw SQL allowed                     |
| Uninstall rollback                   | ✅ Covered by `Uninstall` class            | ❌ Manual `UpgradeData` rollback not reliable |

#### db_schema.xml — Full Example

```xml
<!-- app/code/Vendor/ModuleName/etc/db_schema.xml -->
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/DeclarationSchema/etc/schema.xsd">

    <!-- New table -->
    <table name="vendor_module_audit_log"
           resource="default"
           engine="innodb"
           comment="Audit log for Vendor_ModuleName">
        <column xsi:type="int"
                name="entity_id"
                padding="10"
                unsigned="true"
                nullable="false"
                identity="true"
                comment="Entity ID"/>
        <column xsi:type="varchar"
                name="action"
                length="50"
                nullable="false"
                comment="Action performed"/>
        <column xsi:type="timestamp"
                name="created_at"
                nullable="false"
                default="CURRENT_TIMESTAMP"
                on_update="false"
                comment="Created timestamp"/>
        <constraint xsi:type="primary"
                   referenceId="PRIMARY">
            <column name="entity_id"/>
        </constraint>
        <index referenceId="IDX_ACTION"
               indexType="btree">
            <column name="action"/>
        </index>
    </table>

    <!-- Column add to existing table (non-destructive) -->
    <table name="quote">
        <column xsi:type="decimal"
                name="custom_discount"
                scale="4"
                precision="12"
                nullable="true"
                default="null"/>
    </table>

</schema>
```

After creating or editing `db_schema.xml`, generate the declarative schema
diff and whitelist:

```bash
bin/magento setup:db-declaration:generate-whitelist
bin/magento setup:upgrade --keep-generated
```

> **Keep-generated flag.** The `--keep-generated` flag prevents Magento
> from deleting auto-generated classes in `generated/` during `setup:upgrade`.
> It is safe to use in development; remove it in production deployment scripts.

#### Data Patch — Full Example

```php
<?php
// app/code/Vendor/ModuleName/Setup/Patch/Data/BackfillCustomerAttr.php

declare(strict_types=1);

namespace Vendor\ModuleName\Setup\Patch\Data;

use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\Patch\PatchVersionInterface;
use Magento\Customer\Setup\CustomerSetup;
use Magento\Customer\Setup\CustomerSetupFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class BackfillCustomerAttr implements
    DataPatchInterface,
    PatchVersionInterface
{
    /** @var ModuleDataSetupInterface */
    private $moduleDataSetup;

    /** @var CustomerSetupFactory */
    private $customerSetupFactory;

    /**
     * @param ModuleDataSetupInterface $moduleDataSetup
     * @param CustomerSetupFactory $customerSetupFactory
     */
    public function __construct(
        ModuleDataSetupInterface $moduleDataSetup,
        CustomerSetupFactory $customerSetupFactory
    ) {
        $this->moduleDataSetup = $moduleDataSetup;
        $this->customerSetupFactory = $customerSetupFactory;
    }

    public static function getVersion(): string
    {
        return '1.2.0';
    }

    public function apply(): void
    {
        /** @var CustomerSetup $customerSetup */
        $customerSetup = $this->customerSetupFactory->create(
            ['resourceSetup' => $this->moduleDataSetup]
        );

        $customerSetup->addAttribute(
            'customer',
            'preferred_delivery_date',
            [
                'type' => 'datetime',
                'label' => 'Preferred Delivery Date',
                'input' => 'date',
                'required' => false,
                'system' => false,
                'position' => 200,
            ]
        );
    }

    public function getAliases(): array
    {
        return [];
    }

    public static function getDependencies(): array
    {
        return [];
    }
}
```

#### Uninstall Class — Clean Removal

For modules that need to fully remove their schema and data on uninstall
(when the module is removed via Composer), implement `UninstallInterface`:

```php
<?php
// app/code/Vendor/ModuleName/Setup/Uninstall.php

declare(strict_types=1);

namespace Vendor\ModuleName\Setup;

use Magento\Framework\Setup\UninstallInterface;
use Magento\Framework\Setup\SchemaSetupInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class Uninstall implements UninstallInterface
{
    /** @var SchemaSetupInterface */
    private $schemaSetup;

    /** @var ModuleDataSetupInterface */
    private $moduleDataSetup;

    public function __construct(
        SchemaSetupInterface $schemaSetup,
        ModuleDataSetupInterface $moduleDataSetup
    ) {
        $this->schemaSetup = $schemaSetup;
        $this->moduleDataSetup = $moduleDataSetup;
    }

    public function uninstall(
        SchemaSetupInterface $schemaSetup,
        ModuleDataSetupInterface $moduleDataSetup
    ): void {
        $schemaSetup->startSetup();

        // Drop custom table
        $moduleDataSetup->getConnection()->dropTable(
            $moduleDataSetup->getTable('vendor_module_audit_log')
        );

        // Remove attribute from eav_attribute
        $moduleDataSetup->getConnection()->delete(
            $moduleDataSetup->getTable('eav_attribute'),
            ['attribute_code = ?' => 'preferred_delivery_date']
        );

        $schemaSetup->endSetup();
    }
}
```

> **Warning.** `Uninstall` only runs when the module is removed via
> Composer. If a merchant manually deletes the directory, no Uninstall
> class is invoked. Always document manual removal steps in the README.

---

### 5. Module Version Conflicts — Composer `replace` vs `conflict`

Marketplace validation fails if your module declares incompatible
constraints, or if it silently breaks another module a merchant already
has installed. Composer gives you two tools to prevent this:
`replace` and `conflict`.

#### When to Use `replace`

Use `replace` when your module **is a drop-in replacement** for another
module. This happens when you fork an open-source module and maintain your
own version under a different vendor name.

```json
{
    "name": "Vendor_MyModule",
    "description": "Drop-in replacement for Acme_FreeWidget",
    "replace": {
        "acme/free-widget": "*"
    }
}
```

The consumer will see `acme/free-widget` as already satisfied and will
not be forced to install both. This avoids the "two modules implement the
same class" error.

#### When to Use `conflict`

Use `conflict` when your module **cannot be installed alongside** another
module. Common cases:

- Your module patches a core bug in a way that breaks if a specific
  third-party module is also installed
- You bundle a patched copy of a library and the third-party module
  bundles the same library at a different version

```json
{
    "name": "Vendor_SecurePayments",
    "description": "Payment module with hardened token handling",
    "conflict": {
        "insecure-payments/insecure-module": "*"
    }
}
```

At `composer require`, Composer will refuse to install both together:

```
Package insecure-payments/insecure-module is in conflict with
Vendor_SecurePayments.
```

#### Version Constraint Ranges — Pitfalls

Avoid overly broad constraints that include unsupported Magento versions:

```json
{
    "require": {
        "magento/framework": "103.0.*|104.0.*",
        "php": "~8.1|~8.2"
    }
}
```

| Constraint          | Meaning                                           | Risk                                                    |
|--------------------|---------------------------------------------------|---------------------------------------------------------|
| `103.0.*`          | Any patch of 103.0                                | Safe                                                    |
| `^103.0`           | `>=103.0.0 <104.0.0`                             | Safe                                                    |
| `103.0.0 - 103.9.9`| Exact range syntax                                | Confusing; prefer caret or tilde                       |
| `>=103.0@stable`   | Stability constraint                             | Unstable; use `@stable` in `require` not constraint    |
| `*`                | Any version                                        | Dangerous; never use                                   |

> **Marketplace validation.** Magento Marketplace runs automated checks
> against your `composer.json`. If your module declares `php: *`, the
> submission will be rejected. Always specify a concrete PHP version
> constraint (e.g., `php: ~8.1`) and a concrete Magento framework
> constraint (e.g., `magento/framework: ^103.0`).

#### Complete composer.json Conflict Example

```json
{
    "name": "vendor/module-name",
    "description": "Advanced product feed generator with custom attribute support",
    "type": "magento2-module",
    "license": "MIT",
    "version": "1.3.0",
    "require": {
        "php": "~8.1|~8.2",
        "magento/framework": "^103.0",
        "magento/module-catalog": "^103.0",
        "magento/module-eav": "^103.0",
        "magento/module-store": "^103.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^9.5",
        "magento/magento-coding-standard": "*"
    },
    "suggest": {
        "vendor/optional-plugin": "^1.0"
    },
    "conflict": {
        "acme/product-feed": "*",
        "oldvendor/deprecated-module": "<2.0.0"
    },
    "autoload": {
        "psr-4": {
            "Vendor\\ModuleName\\": ""
        },
        "files": [
            "registration.php"
        ]
    },
    "minimum-stability": "stable",
    "prefer-stable": true
}
```

---

## Definition of Done — Extension Quality Checklist

Use this checklist before every release tag and before submitting to
Magento Marketplace.

### Composer Hygiene

- [ ] `php` version constraint is specific (`~8.1` not `*`)
- [ ] All `require` entries use vendor-prefixed packages (not metapackage)
- [ ] No `require-dev` packages in `require` (testing tools are dev-only)
- [ ] `composer.lock` is not committed
- [ ] `replace` or `conflict` entries are documented and intentional

### Module Declaration

- [ ] `etc/module.xml` has `<sequence>` for every module whose class is
      extended, plugin-ed, or observed
- [ ] Circular sequence check passes locally (see Section 2)
- [ ] `registration.php` uses `ComponentRegistrar::MODULE` and correct namespace

### DI Configuration

- [ ] No duplicate `<preference>` for the same interface/class
- [ ] Plugin used instead of `<preference>` wherever possible
- [ ] `di:compile` runs without errors in a fresh Magento environment

### Schema and Data Patches

- [ ] New tables declared in `db_schema.xml` with proper engine and charset
- [ ] Whitelist regenerated via `bin/magento setup:db-declaration:generate-whitelist`
- [ ] Data patches implement `DataPatchInterface` and `PatchVersionInterface`
- [ ] `Uninstall` class drops all custom tables and EAV attributes
- [ ] Decision table in Section 4 was used to choose schema vs patch

### Static Analysis

- [ ] PHPStan runs at level 1 or higher in GitHub Actions on every PHP file change
- [ ] `--no-progress` and `--error-format=github` flags present in the CI step
- [ ] `phpstan-baseline.neon` is committed; no suppression via inline ignore comments
- [ ] PHPStan level is increased by at least one step per quarter

### Documentation

- [ ] README includes installation steps, compatibility matrix, and upgrade notes
- [ ] `CHANGELOG.md` follows Keep a Changelog format
- [ ] All public interfaces are documented in `Api/` PHPDoc blocks
- [ ] `Uninstall` procedure is documented if the module creates EAV attributes

### Marketplace

- [ ] Submission tested against the **oldest** and **newest** patch in the
      supported `magento/framework` range
- [ ] No `*` or unconstrained version in `require`/`require-dev`
- [ ] License field is present and matches the repository license
- [ ] `description` field is 60–200 characters and free of marketing fluff

---

*Magento 2 Backend Developer Course — Topic 13 | Module Packaging & Release*