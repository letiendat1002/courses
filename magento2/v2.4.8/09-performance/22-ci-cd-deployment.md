---
title: "33 - CI-CD & Deployment"
description: "Deep dive into Magento 2.4.8 deployment: CI-CD pipelines, environment configuration, zero-downtime deployment, build scripts, and production deployment strategies."
tags: magento2, ci-cd, deployment, pipeline, zero-downtime, build-scripts, environment-config, production
rank: 22
pathways: [magento2-deep-dive]
---

# CI-CD & Deployment

## Introduction

Deploying Magento 2.4.8 to production is one of the most complex challenges in e-commerce infrastructure. Unlike simple web applications, Magento is a monolithic PHP application with multiple deployment phases, a sophisticated dependency injection system, a theme/assets compilation pipeline, and database schema management. A misconfigured deployment can take down a store generating thousands of dollars per hour.

This article covers the complete deployment lifecycle: from repository configuration and CI pipeline design, through build-time compilation steps, to zero-downtime production deployment strategies including blue-green, maintenance mode handling, and Adobe Commerce Cloud specifics. Every concept is illustrated with real `app/etc/config.php`, `composer.json`, GitHub Actions YAML, and CLI commands from or compatible with Magento 2.4.8.

---

## 1. Deployment Overview

### 1.1 GitFlow vs Trunk-Based Development

Two branching strategies dominate Magento deployments. **GitFlow** uses long-lived `develop` and `main` (or `master`) branches with feature branches merged via pull requests. This provides strict quality gates but introduces latency between development and production. **Trunk-based development** uses a single main branch (`main` or `trunk`) with short-lived feature branches that merge quickly (often via fast-forward or squash). Trunk-based development aligns better with continuous delivery because every merged commit is a potential release candidate.

For a typical Magento 2.4.8 store:

```
GitFlow:
  feature/ES-1234-new-checkout → develop → release/2.4.8 → main → TAG

Trunk-Based:
  feature/ES-1234-new-checkout → main (every merged commit = potential release)
```

Trunk-based development is generally preferred for CI-CD because it:
- Reduces merge conflicts by merging frequently
- Enables smaller, more frequent deployments
- Aligns with feature flags to toggle incomplete features off in production
- Simplifies the deployment pipeline (fewer branch-specific environments)

### 1.2 Magento Deploy Phases

Magento 2's deployment process runs in five distinct phases. Understanding them is critical for designing a proper CI-CD pipeline, because each phase has different environmental requirements and failure modes.

**Phase 1 — Build**

The build phase installs or updates Composer dependencies and generates the autoloader. No Magento runtime (database, cache, session) is required. The build phase produces a `vendor/` directory and a bootstrapped Magento application class tree. In CI, this phase runs in a clean container with only PHP and Composer installed.

```bash
# Build phase - no runtime dependencies needed
composer install --no-dev --prefer-dist --optimize-autoloader
```

**Phase 2 — Static Content Deployment**

Magento's theme system requires CSS, JavaScript, and image assets to be pre-processed and published to `pub/static/`. This is the slowest phase in a deployment because it processes all configured languages, themes, and required modules. For a store with three languages, two themes (Luma + custom), and 80 modules, this can take 5-15 minutes.

**Phase 3 — DI Compilation**

Magento's dependency injection container reads `di.xml` files across all modules and generates PHP factories, proxies, and interceptors into `generated/`. This phase converts the declarative XML configuration into optimized PHP code. Without this step, Magento falls back to runtime code generation, which is significantly slower.

**Phase 4 — Database Schema Upgrade**

The `setup:upgrade` command applies declarative schema and data patches. This phase requires a writable database connection and is the primary source of deployment-related downtime if not handled carefully.

**Phase 5 — Cache Flush**

After all compilation steps, Magento's cache storage (Redis, file, or database) is flushed. A full cache flush causes a temporary performance cliff as Magento rebuilds all caches on the first request.

### 1.3 Environment Types

| Environment | Purpose | Config Source | Data |
|---|---|---|---|
| **Local** | Developer workstation | `env.php` (local override) | Sample data or subset of production |
| **Development** | Shared integration testing | `config.php` + `env.php` | Sanitized production export |
| **Staging** | Pre-production validation | Mirror of production infra | Sanitized production export |
| **Production** | Live store traffic | `config.php` + `env.php` (secrets via env vars) | Production data |

Environment configuration flows from `app/etc/config.php` (shared, committed) to `app/etc/env.php` (environment-specific, never committed with real secrets).

---

## 2. Build Configuration

### 2.1 `app/etc/config.php` — The Shared Configuration

`config.php` is the system-wide configuration that gets committed to the repository. It contains module enable/disable states, scuttle URL settings, and deployed admin URLs. In Magento 2.4.8, `config.php` is generated by `bin/magento app:config:dump` and should be version-controlled.

```php
<?php
/**
 * Shared configuration — commit to repository
 * Generated by: bin/magento app:config:dump
 */
return [
    'modules' => [
        'Magento_Store' => 1,
        'Magento_Config' => 1,
        'Magento_Catalog' => 1,
        'Magento_Customer' => 1,
        'Magento_Sales' => 1,
        // ... all enabled modules
        'Vendor_CustomModule' => 1,
    ],
    'scopes' => [
        'websites' => [
            'base' => [
                'website_id' => '1',
                'code' => 'base',
                'name' => 'Main Website',
                'sort_order' => '0',
                'default_group_id' => '1',
                'is_default' => '1',
            ],
        ],
        'groups' => [
            '1' => [
                'group_id' => '1',
                'website_id' => '1',
                'name' => 'Main Website Store',
                'default_store_id' => '1',
                'root_category_id' => '2',
            ],
        ],
        'stores' => [
            '1' => [
                'store_id' => '1',
                'code' => 'default',
                'website_id' => '1',
                'group_id' => '1',
                'name' => 'Default Store View',
                'sort_order' => '0',
                'is_active' => '1',
            ],
        ],
    ],
    'themes' => [
        'frontend/Magento/luma' => [
            'parent_id' => 'Magento/blank',
            'theme_title' => 'Magento Luma',
            'is_featured' => false,
            'area' => 'frontend',
            'type' => '0',
            'path' => 'Magento/luma',
        ],
        'frontend/Vendor/custom' => [
            'parent_id' => 'Magento/luma',
            'theme_title' => 'Custom Theme',
            'is_featured' => false,
            'area' => 'frontend',
            'type' => '0',
            'path' => 'Vendor/custom',
        ],
    ],
];
```

### 2.2 `app/etc/env.php` — Environment-Specific Secrets

`env.php` contains sensitive configuration and environment-specific overrides: database credentials, Redis connections, session storage, administrative credentials, and API keys. **This file must never be committed to version control.** Instead, secrets are injected via environment variables during the CI-CD pipeline.

```php
<?php
/**
 * Environment-specific configuration — DO NOT COMMIT
 * Populated from environment variables at deploy time
 */
return [
    'backend' => [
        'frontName' => 'admin_k3r8x',
    ],
    'db' => [
        'connection' => [
            'indexer' => [
                'host' => 'db.internal',
                'dbname' => 'magento_production',
                'username' => getenv('DB_USER') ?: 'magento',
                'password' => getenv('DB_PASSWORD') ?: '',
                'model' => 'mysql4',
                'engine' => 'innodb',
                'initStatements' => 'SET NAMES utf8mb4',
                'active' => '1',
            ],
            'default' => [
                // Same structure as indexer connection
                'host' => 'db.internal',
                'dbname' => 'magento_production',
                'username' => getenv('DB_USER') ?: 'magento',
                'password' => getenv('DB_PASSWORD') ?: '',
                'model' => 'mysql4',
                'engine' => 'innodb',
                'initStatements' => 'SET NAMES utf8mb4',
                'active' => '1',
            ],
        ],
        'table_prefix' => '',
    ],
    'crypt' => [
        'key' => getenv('APP_CRYPT_KEY') ?: '',
    ],
    'session' => [
        'save' => 'redis',
        'redis' => [
            'host' => 'redis.internal',
            'port' => '6379',
            'password' => getenv('REDIS_SESSION_PASSWORD') ?: '',
            'timeout' => '2.5',
            'persistent_identifier' => 'magento-session',
            'database' => '0',
            'skip_clustering' => 'true',
            'log_level' => '1',
        ],
    ],
    'cache' => [
        'frontend' => [
            'default' => [
                'backend' => 'Cm_Cache_Backend_Redis',
                'backend_options' => [
                    'server' => 'redis.internal',
                    'port' => '6379',
                    'password' => getenv('REDIS_CACHE_PASSWORD') ?: '',
                    'database' => '1',
                    'compression' => '1',
                    'compression_lib' => 'gzip',
                ],
            ],
            'page_cache' => [
                'backend' => 'Cm_Cache_Backend_Redis',
                'backend_options' => [
                    'server' => 'redis.internal',
                    'port' => '6379',
                    'password' => getenv('REDIS_CACHE_PASSWORD') ?: '',
                    'database' => '2',
                    'compression' => '1',
                    'compression_lib' => 'gzip',
                ],
            ],
        ],
    ],
    'resource' => [
        'default_setup' => [
            'connection' => 'default',
        ],
    ],
    'x-frame-options' => 'SAMEORIGIN',
    'install' => [
        'date' => 'Tue, 15 Oct 2024 08:30:00 +0000',
    ],
];
```

### 2.3 `composer.json` — Dependency Management

A Magento 2.4.8 `composer.json` must pin exact PHP version requirements and declare all production dependencies. The `magento/product-community-edition` metapackage locks the Magento version; individual module packages are listed separately for enterprise-grade control.

```json
{
    "name": "vendor/magento-store",
    "description": "Magento 2.4.8 production deployment",
    "type": "project",
    "license": "proprietary",
    "require": {
        "php": "~8.1.0||~8.2.0",
        "ext-bcmath": "*",
        "ext-ctype": "*",
        "ext-curl": "*",
        "ext-dom": "*",
        "ext-gd": "*",
        "ext-hash": "*",
        "ext-iconv": "*",
        "ext-intl": "*",
        "ext-mbstring": "*",
        "ext-openssl": "*",
        "ext-pdo": "*",
        "ext-pdo_mysql": "*",
        "ext-simplexml": "*",
        "ext-soap": "*",
        "ext-sodium": "*",
        "ext-xsl": "*",
        "ext-zip": "*",
        "magento/product-community-edition": "2.4.8",
        "Vendor_PremiumExtensions": "^1.0.0",
        "stripe/stripe-payments": "^4.0.0",
        "snowdog/module-frontools": "^18.0.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^9.5",
        "phpstan/phpstan": "^1.10",
        "magento/magento-coding-standard": "^8.0",
        "friendsofphp/php-cs-fixer": "^3.4"
    },
    "autoload": {
        "psr-4": {
            "Magento\\Framework\\": "lib/internal/Magento/Framework/",
            "Magento\\Setup\\": "setup/src/Magento/Setup/"
        },
        "psr-0": {
            "": [
                "app/code/",
                "generated/"
            ]
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Magento\\Framework\\": "lib/internal/Magento/Framework/",
            "Magento\\Setup\\": "setup/src/Magento/Setup/"
        }
    },
    "scripts": {
        "post-install-cmd": [
            "Magento\\Composer\\Scripts::postInstall",
            "Magento\\Framework\\Composer\\Cleanup::cleanup"
        ],
        "post-update-cmd": [
            "Magento\\Composer\\Scripts::postUpdate",
            "Magento\\Framework\\Composer\\Cleanup::cleanup"
        ],
        "test:unit": "vendor/bin/phpunit -c dev/tests/unit/phpunit.xml",
        "test:integration": "vendor/bin/phpunit -c dev/tests/integration/phpunit.xml",
        "cs:check": "vendor/bin/phpcs --standard=Magento2 --extensions=php,phtml app/code"
    },
    "config": {
        "preferred-install": "dist",
        "sort-packages": true,
        "allow-plugins": {
            "magento/magento-composer-installer": true,
            "php-http/discovery": true
        }
    },
    "minimum-stability": "stable",
    "prefer-stable": true
}
```

> **CI Tip:** Always run `composer install --no-dev` in production builds. Dev dependencies (PHPUnit, PHPStan, coding standards) are unnecessary and increase attack surface in production containers.

---

## 3. CI Pipeline

### 3.1 GitHub Actions — Full Production Pipeline

The following GitHub Actions workflow implements a multi-stage pipeline: build verification, static analysis, unit tests, integration tests, and deployment to staging/production environments.

```yaml
# .github/workflows/magento-ci.yml
name: Magento 2.4.8 CI-CD Pipeline

on:
  push:
    branches: [main, 'release/**']
  pull_request:
    branches: [main]

env:
  MAGENTO_VERSION: "2.4.8"
  PHP_VERSION: "8.2"

jobs:
  # ─────────────────────────────────────────────────────────────────────
  # JOB 1: Quality Gate — Code Style, Static Analysis, Unit Tests
  # ─────────────────────────────────────────────────────────────────────
  quality-gate:
    name: Quality Gate
    runs-on: ubuntu-22.04

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}
          extensions: bcmath, curl, gd, intl, mbstring, pdo_mysql, sodium, xsl, zip
          coverage: xdebug

      - name: Cache Composer vendor
        uses: actions/cache@v4
        with:
          path: |
            vendor/
            ~/.composer/cache/
          key: composer-${{ runner.os }}-${{ hashFiles('**/composer.lock') }}
          restore-keys: |
            composer-${{ runner.os }}-

      - name: Install dependencies
        run: composer install --no-interaction --prefer-dist

      - name: Code style check
        run: vendor/bin/phpcs --standard=Magento2 --extensions=php,phtml --warning-severity=0 app/code/

      - name: Run PHPStan
        run: vendor/bin/phpstan analyse app/code --level=5 --no-progress

      - name: Unit tests
        run: vendor/bin/phpunit -c dev/tests/unit/phpunit.xml --no-coverage

  # ─────────────────────────────────────────────────────────────────────
  # JOB 2: Integration Test Suite
  # ─────────────────────────────────────────────────────────────────────
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-22.04
    needs: quality-gate

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: magento_test
          MYSQL_USER: magento
          MYSQL_PASSWORD: magento
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=5

      redis:
        image: redis:7.0-alpine
        ports:
          - 6379:6379

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}
          extensions: bcmath, curl, gd, intl, mbstring, pdo_mysql, sodium, xsl, zip

      - name: Configure Magento for integration testing
        run: |
          cp dev/tests/integration/phpunit.xml.dist dev/tests/integration/phpunit.xml
          cat > dev/tests/integration/etc/install-config-mysql.php << 'EOF'
          <?php
          return [
              'db-host' => '127.0.0.1',
              'db-user' => 'magento',
              'db-password' => 'magento',
              'db-name' => 'magento_test',
              'db-prefix' => '',
              'backend-frontname' => 'admin',
              'base-url' => 'http://localhost/',
              'base-url-secure' => 'https://localhost/',
              'admin-user' => 'admin',
              'admin-password' => 'Admin12345!',
              'admin-email' => 'admin@example.com',
              'admin-firstname' => 'Admin',
              'admin-lastname' => 'User',
          ];
          EOF

      - name: Cache Composer vendor
        uses: actions/cache@v4
        with:
          path: |
            vendor/
            ~/.composer/cache/
          key: composer-${{ runner.os }}-${{ hashFiles('**/composer.lock') }}

      - name: Install dependencies
        run: composer install --no-interaction --prefer-dist

      - name: Run integration tests
        run: vendor/bin/phpunit -c dev/tests/integration/phpunit.xml --no-coverage

  # ─────────────────────────────────────────────────────────────────────
  # JOB 3: Build — Static Content & DI Compilation
  # ─────────────────────────────────────────────────────────────────────
  build:
    name: Build Artifact
    runs-on: ubuntu-22.04
    needs: integration-tests
    if: github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/heads/release/')

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}
          extensions: bcmath, curl, gd, intl, mbstring, pdo_mysql, sodium, xsl, zip

      - name: Cache Composer vendor
        uses: actions/cache@v4
        with:
          path: |
            vendor/
            ~/.composer/cache/
          key: composer-${{ runner.os }}-${{ hashFiles('**/composer.lock') }}

      - name: Install production dependencies
        run: composer install --no-dev --no-interaction --prefer-dist --optimize-autoloader

      - name: Build static content (English US + 2 locales)
        run: |
          php bin/magento setup:static-content:deploy \
            --language=en_US \
            --language=de_DE \
            --language=fr_FR \
            --theme=Magento/luma \
            --theme=Vendor/custom \
            --jobs=4 \
            --no-ansi

      - name: Compile dependency injection
        run: php bin/magento setup:di:compile

      - name: Set production mode
        run: php bin/magento deploy:mode:set production --no-compilation

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: magento-build-${{ github.sha }}
          path: |
            vendor/
            generated/
            pub/static/
            app/etc/config.php
          retention-days: 7

  # ─────────────────────────────────────────────────────────────────────
  # JOB 4: Deploy to Staging
  # ─────────────────────────────────────────────────────────────────────
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-22.04
    needs: build
    environment: staging

    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: magento-build-${{ github.sha }}

      - name: Deploy to staging via SCP/RSYNC
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /var/www/magento-staging
            php bin/magento maintenance:enable
            rsync -az --delete vendor/ generated/ pub/static/ app/etc/ $SSH_ORIGIN/
            php bin/magento setup:upgrade --keep-generated
            php bin/magento cache:flush
            php bin/magento maintenance:disable

      - name: Smoke test staging
        run: curl -f -s https://staging.example.com | grep -q "Magento" || exit 1

  # ─────────────────────────────────────────────────────────────────────
  # JOB 5: Deploy to Production (Manual Approval Required)
  # ─────────────────────────────────────────────────────────────────────
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-22.04
    needs: deploy-staging
    environment:
      name: production
      url: https://www.example.com
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: magento-build-${{ github.sha }}

      - name: Deploy to production with zero-downtime strategy
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            set -e
            export APP_CRYPT_KEY="${{ secrets.PROD_CRYPT_KEY }}"
            export DB_USER="${{ secrets.PROD_DB_USER }}"
            export DB_PASSWORD="${{ secrets.PROD_DB_PASSWORD }}"

            # Enable maintenance mode
            php bin/magento maintenance:enable

            # Sync code artifacts
            rsync -az --delete vendor/ generated/ pub/static/ $SSH_ORIGIN/

            # Run database upgrade (declarative schema + data patches)
            php bin/magento setup:upgrade \
              --keep-generated \
              --safe-installer=1

            # Compile DI if not already done
            php bin/magento setup:di:compile

            # Flush cache
            php bin/magento cache:flush

            # Disable maintenance mode
            php bin/magento maintenance:disable

            # Verify deployment
            curl -f -s https://www.example.com | grep -q "Magento" || { \
              echo "HEALTH_CHECK_FAILED"; \
              php bin/magento maintenance:enable; \
              exit 1; \
            }
```

### 3.2 GitLab CI Equivalent

For GitLab CI, the pipeline configuration uses a similar structure with GitLab-specific syntax:

```yaml
# .gitlab-ci.yml
stages:
  - quality
  - test
  - build
  - deploy

variables:
  MAGENTO_VERSION: "2.4.8"
  PHP_VERSION: "8.2"

cache:
  key: composer-$CI_COMMIT_REF_SLUG
  paths:
    - vendor/
    - ~/.composer/cache/

# ── Quality Gate ────────────────────────────────────────────────────────
phpcs:
  stage: quality
  image: php:${PHP_VERSION}-cli
  before_script:
    - apt-get update -qq && apt-get install -y -qq git unzip libcurl4-gnutls-dev
    - docker-php-ext-install bcmath curl gd intl mbstring pdo_mysql sodium xsl zip
    - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
  script:
    - composer install --no-interaction
    - vendor/bin/phpcs --standard=Magento2 --extensions=php,phtml app/code/
  allow_failure: false

phpstan:
  stage: quality
  image: php:${PHP_VERSION}-cli
  script:
    - composer install --no-interaction
    - vendor/bin/phpstan analyse app/code --level=5 --no-progress
  allow_failure: true  # Non-blocking

# ── Test Stage ─────────────────────────────────────────────────────────
unit-tests:
  stage: test
  image: php:${PHP_VERSION}-cli
  script:
    - composer install --no-interaction
    - vendor/bin/phpunit -c dev/tests/unit/phpunit.xml --no-coverage

integration-tests:
  stage: test
  image: php:${PHP_VERSION}-cli
  services:
    - mysql:8.0
    - redis:7.0-alpine
  variables:
    MYSQL_ROOT_PASSWORD: root
    MYSQL_DATABASE: magento_test
    MYSQL_USER: magento
    MYSQL_PASSWORD: magento
  script:
    - composer install --no-interaction
    - cp dev/tests/integration/phpunit.xml.dist dev/tests/integration/phpunit.xml
    - vendor/bin/phpunit -c dev/tests/integration/phpunit.xml --no-coverage
  allow_failure: false

# ── Build Stage ────────────────────────────────────────────────────────
build:
  stage: build
  image: php:${PHP_VERSION}-cli
  artifacts:
    paths:
      - vendor/
      - generated/
      - pub/static/
      - app/etc/config.php
    expire_in: 1 day
  script:
    - composer install --no-dev --no-interaction --prefer-dist --optimize-autoloader
    - php bin/magento setup:static-content:deploy --language=en_US --language=de_DE --jobs=4 --no-ansi
    - php bin/magento setup:di:compile
    - php bin/magento deploy:mode:set production --no-compilation

# ── Deploy Stages ──────────────────────────────────────────────────────
deploy-staging:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client rsync
  script:
    - eval $(ssh-agent -s)
    - echo "$STAGING_SSH_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh && chmod 700 ~/.ssh
    - ssh-keyscan $STAGING_HOST >> ~/.ssh/known_hosts
    - ssh $STAGING_USER@$STAGING_HOST "cd /var/www/magento-staging && php bin/magento maintenance:enable"
    - rsync -az --delete vendor/ generated/ pub/static/ $STAGING_USER@$STAGING_HOST:/var/www/magento-staging/
    - ssh $STAGING_USER@$STAGING_HOST "cd /var/www/magento-staging && php bin/magento setup:upgrade --keep-generated && php bin/magento cache:flush && php bin/magento maintenance:disable"
  environment:
    name: staging
  only:
    - main
    - release

deploy-production:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client rsync
  script:
    - eval $(ssh-agent -s)
    - echo "$PROD_SSH_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh && chmod 700 ~/.ssh
    - ssh-keyscan $PROD_HOST >> ~/.ssh/known_hosts
    - ssh $PROD_USER@$PROD_HOST "cd /var/www/magento && php bin/magento maintenance:enable"
    - rsync -az --delete vendor/ generated/ pub/static/ $PROD_USER@$PROD_HOST:/var/www/magento/
    - ssh $PROD_USER@$PROD_HOST "cd /var/www/magento && php bin/magento setup:upgrade --keep-generated && php bin/magento setup:di:compile && php bin/magento cache:flush && php bin/magento maintenance:disable"
  environment:
    name: production
    url: https://www.example.com
  when: manual
  only:
    - main
```

---

## 4. Static Content Deployment

### 4.1 How Static Content Deployment Works

Magento's frontend uses RequireJS and LESS CSS, both of which require preprocessing. The `setup:static-content:deploy` command reads theme configurations, processes LESS files into CSS (via MAGENTO_ROOT/lib/web/less), merges JavaScript files, copies image assets, and outputs minified, versioned files to `pub/static/{area}/{Vendor}/{theme}/xx_xx/`.

The `--content-version` flag is critical for CDN integration. It creates a unique version folder (e.g., `pub/static/version1732501200/frontend/Magento/luma/en_US/`) allowing CDN caches to be busted simply by changing the version number without regenerating all content.

### 4.2 Deploy Commands

```bash
# Deploy for single locale/theme (fastest, use in CI for specific targets)
php bin/magento setup:static-content:deploy \
  --language=en_US \
  --theme=Magento/luma \
  --theme=Vendor/custom \
  --jobs=4 \
  --no-ansi

# Deploy for multiple languages in parallel (typical production)
php bin/magento setup:static-content:deploy \
  --language=en_US \
  --language=de_DE \
  --language=fr_FR \
  --language=es_ES \
  --theme=Magento/luma \
  --theme=Vendor/custom \
  --jobs=$(nproc) \
  --no-ansi

# Force deploy (redeploy even if detect unchanged)
php bin/magento setup:static-content:deploy \
  --force \
  --language=en_US
```

### 4.3 Theme-Specific Deployment Strategy

In a multi-store setup where different websites use different themes, deploy only the themes in use:

```bash
# List all configured themes first
php bin/magento theme:list

# Deploy for specific areas (frontend vs adminhtml)
php bin/magento setup:static-content:deploy \
  --area=frontend \
  --language=en_US \
  --theme=Magento/luma \
  --theme=Vendor/custom

php bin/magento setup:static-content:deploy \
  --area=adminhtml \
  --language=en_US \
  --theme=Magento/backend
```

### 4.4 Language Pack Integration

For multi-language stores, install language packages via Composer and deploy them:

```bash
# Install language packages
composer require mageplaza/language-de_de
composer require mageplaza/language-fr_fr
composer require mageplaza/language-es_es

# Deploy with language packs
php bin/magento setup:static-content:deploy \
  --language=de_DE \
  --language=fr_FR \
  --language=es_ES
```

Language pack packages register themselves in `config.php` under the `i18n` key and deploy to `pub/static/{area}/{Vendor}/{theme}/{locale}/`.

### 4.5 Build Script for CI

```bash
#!/bin/bash
# scripts/deploy-static-content.sh

set -e

LANGUAGES=${LANGUAGES:-"en_US"}
THEMES=${THEMES:-"Magento/luma Vendor/custom"}
JOBS=${JOBS:-$(nproc)}
AREA=${AREA:-"frontend adminhtml"}

echo "==> Deploying static content for languages: $LANGUAGES"
echo "==> Themes: $THEMES"
echo "==> Areas: $AREA"

for area in $AREA; do
  for lang in $LANGUAGES; do
    for theme in $THEMES; do
      echo "==> Processing: $area / $theme / $lang"
      php bin/magento setup:static-content:deploy \
        --area="$area" \
        --language="$lang" \
        --theme="$theme" \
        --jobs="$JOBS" \
        --no-ansi \
        --no-progress
    done
  done
done

# Create version marker for CDN busting
CONTENT_VERSION=$(date +%s)
echo "$CONTENT_VERSION" > pub/static/version.txt
echo "==> Static content version: $CONTENT_VERSION"
```

---

## 5. DI Compilation

### 5.1 What Gets Generated

Running `bin/magento setup:di:compile` scans every module's `etc/di.xml` and generates PHP classes into `generated/code/`. The generated artifacts include:

- **Factories** (`*Factory.php`) — for classes that need to be instantiated via `ObjectManager::create()`. Without generated factories, Magento falls back to runtime reflection which is slow.
- **Proxies** (`*Proxy.php`) — lazy-loading placeholders for classes marked with `proxy` in `di.xml`. Reduces initialization time for heavy services.
- **Interceptors** (`*Interceptor.php`) — generated around methods decorated with `plugin` definitions. Enables the plugin system without runtime code generation.
- **Plugin List** (`Magento\FrameworkInterception\PluginListGenerator`) — compiled plugin chain for each class.

### 5.2 Compile Command and Flags

```bash
# Full DI compilation (produces generated/code/)
php bin/magento setup:di:compile

# Compile specific area only
php bin/magento setup:di:compile --area=frontend
php bin/magento setup:di:compile --area=adminhtml
php bin/magento setup:di:compile --area=crontab

# Multi-tenant compilation (faster on multi-core)
php bin/magento setup:di:compile-multi-tenant
```

### 5.3 Generated Code Structure

```
generated/
├── code/
│   ├── Magento/
│   │   ├── Catalog/
│   │   │   ├── Model/
│   │   │   │   ├── Product/
│   │   │   │   │   ├── ProductFactory.php          # Generated factory
│   │   │   │   ├── ProductFactory.php.cache        # APCu cache
│   │   │   │   └── ProductInterceptor.php          # Generated interceptor
│   │   │   └── ...
│   │   └── ...
│   └── Vendor/
│       └── CustomModule/
│           └── ...
├── etc/
│   └── di.xml                                      # Compiled di.xml
└── metadata/
    ├── globalandatory.php
    └── primary.php
```

### 5.4 Performance Impact

| Scenario | Without Compile | With Compile | Improvement |
|---|---|---|---|
| First request (product page) | 3.2s | 1.1s | ~65% faster |
| Admin panel load | 2.8s | 0.9s | ~68% faster |
| CLI command execution | 1.5s | 0.4s | ~73% faster |

The performance gain comes from eliminating runtime reflection and XML parsing for every request. In production mode (`app/etc/env.php` `MAGE_MODE=production`), Magento strictly requires `generated/code/` to exist and be up-to-date — it will not fall back to runtime generation.

### 5.5 DI Compile in CI

```bash
# In CI build job, after composer install
composer install --no-dev --optimize-autoloader

# Static content deployment
php bin/magento setup:static-content:deploy \
  --language=en_US \
  --jobs=4

# DI compilation
php bin/magento setup:di:compile

# Verify generated code exists
test -d generated/code && echo "DI compilation successful" || { echo "DI compilation FAILED"; exit 1; }
```

---

## 6. Zero-Downtime Deployment

### 6.1 Understanding the Downtime Window

The traditional Magento deployment causes downtime during the window when:

1. New code is deployed
2. `setup:upgrade` runs (modifies database schema)
3. Cache is flushed
4. Maintenance mode is disabled

For a large store with extensive declarative schema changes, `setup:upgrade` can take 5-30 minutes. This is unacceptable for 24/7 storefronts.

### 6.2 Maintenance Mode

```bash
# Enable maintenance mode (creates var/.maintenance.flag)
php bin/magento maintenance:enable

# Enable for specific IP addresses (whitelist)
php bin/magento maintenance:enable --ip=203.0.113.42 --ip=198.51.100.0/24

# Check if maintenance mode is active
php bin/magento maintenance:status

# Disable maintenance mode
php bin/magento maintenance:disable

# Enable with custom HTML message
php bin/magento maintenance:enable \
  --message="Site under maintenance. Back at 02:00 UTC."
```

Magento's maintenance mode checks for `var/.maintenance.flag` (or `var/.maintenance.ip` for IP-specific access). The default maintenance page is served from `errors/default/503.phtml`.

### 6.3 Readonly Mode (Production Warm-Up)

A better approach for planned maintenance windows uses Redis/DB-backed cache to serve a "readonly" but fully-functional storefront:

```php
// pub/index.php — Modified for readonly mode
$bootstrap = \Magento\Framework\App\Bootstrap::create(BP, $_SERVER);

/** @var \Magento\Framework\App\State\Overlay\DeploymentConfig\OverlayDeploymentConfig */
$state = $bootstrap->getObjectManager()->get(\Magento\Framework\App\State::class);

// Check if readonly/rollout mode is active
if (getenv('DEPLOY_READONLY_MODE') === '1') {
    // Serve cached full-page responses from Redis
    // while blocking write operations (cart, checkout, account)
    $state->setAreaCode('frontend');
    $app = $bootstrap->createApplication(\Magento\Framework\App\Http\Placeholder::class);
} else {
    $app = $bootstrap->createApplication(\Magento\Framework\App\Http::class);
}

$response = $bootstrap->run($app);
```

### 6.4 Blue-Green Deployment

Blue-green deployment maintains two near-identical production environments ("blue" and "green") and switches traffic between them. For Magento, this requires shared storage (NFS or EFS) for `media/` and `var/` directories so both environments see the same data.

```
                    ┌─────────────┐
        Traffic ───►│ Load Balancer │
                    └──────┬──────┘
                           │ switch
               ┌───────────┴───────────┐
               │                       │
          [Blue]                   [Green]
       (current)                (new deploy)
       2.4.7                     2.4.8

Steps:
1. Deploy to Green (Maintenance ON for Green only)
2. Smoke test Green
3. Switch LB to Green
4. Traffic now on Green (2.4.8)
5. Keep Blue running (instant rollback if needed)
6. Decomission Blue after N hours
```

**Implementation:**

```bash
#!/bin/bash
# scripts/blue-green-deploy.sh

set -e

BLUE_DIR="/var/www/magento-blue"
GREEN_DIR="/var/www/magento-green"
SHARED_MEDIA="/var/www/shared/media"
SHARED_VAR="/var/www/shared/var"
ACTIVE_COLOR=$(cat /var/www/active_color.txt)

# Determine target environment
if [ "$ACTIVE_COLOR" == "blue" ]; then
  TARGET="green"
else
  TARGET="blue"
fi

TARGET_DIR="/var/www/magento-$TARGET"

echo "==> Deploying to $TARGET (currently $ACTIVE_COLOR is active)"

# Enable maintenance on target
ssh $TARGET_HOST "cd $TARGET_DIR && php bin/magento maintenance:enable"

# Sync code
rsync -az --delete \
  vendor/ \
  generated/ \
  pub/static/ \
  app/etc/ \
  $TARGET_HOST:$TARGET_DIR/

# Run database upgrade on target
ssh $TARGET_HOST "cd $TARGET_DIR \
  && php bin/magento setup:upgrade \
       --keep-generated \
       --safe-installer=1"

# Warm up target environment
ssh $TARGET_HOST "curl -s http://localhost/ | grep -q Magento" || true

# Switch traffic (update load balancer target group)
aws elbv2 modify-listener \
  --listener-arn $LISTENER_ARN \
  --default-actions Type=forward,TargetGroupArn=$TARGET_TG_ARN

# Update active color marker
echo "$TARGET" > /var/www/active_color.txt

# Disable maintenance on new active
ssh $TARGET_HOST "cd $TARGET_DIR && php bin/magento maintenance:disable"

echo "==> Deployment complete. $TARGET is now active."
```

### 6.5 Traffic Shifting with AWS ALB

For AWS-hosted Magento, traffic shifting can be automated with CodeDeploy and ALB weighted routing:

```yaml
# appspec.yml for AWS CodeDeploy
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/magento-green
hooks:
  AfterInstall:
    - location: scripts/deploy-magento.sh
      timeout: 3600
      runas: www-data
  AllowTraffic:
    - location: scripts/enable-traffic.sh
      timeout: 60
      runas: www-data
```

```bash
#!/bin/bash
# scripts/enable-traffic.sh — Traffic shifting via ALB

TARGET_GROUP_ARN="${TARGET_GROUP_ARN:-}"
WEIGHT="${WEIGHT:-100}"

# Shift 10% of traffic initially, verify, then shift remaining 90%
aws elbv2 modify-target-group \
  --target-group-arn "$TARGET_GROUP_ARN" \
  --health-check-path /healthz

# Full traffic switch
aws elbv2 modify-target-group \
  --target-group-arn "$TARGET_GROUP_ARN"
```

### 6.6 Atomic Deployment with Symbolic Links

For single-server or shared-nothing deployments, use symlink swapping for atomic code rollouts:

```bash
#!/bin/bash
# scripts/atomic-deploy.sh

set -e

RELEASE_DIR="/var/www/releases"
RELEASE_NAME="release_$(date +%Y%m%d_%H%M%S)"
SHARED_DIR="/var/www/shared"
CURRENT_LINK="/var/www/magento"
DEPLOY_USER="www-data"

# Create new release directory
mkdir -p $RELEASE_DIR/$RELEASE_NAME
cd $RELEASE_DIR/$RELEASE_NAME

# Build in release directory
composer install --no-dev --prefer-dist
php bin/magento setup:static-content:deploy --language=en_US --jobs=4
php bin/magento setup:di:compile

# Symlink shared directories
ln -s $SHARED_DIR/media $RELEASE_DIR/$RELEASE_NAME/pub/media
ln -s $SHARED_DIR/var $RELEASE_DIR/$RELEASE_NAME/var

# Atomic symlink swap
ln -nfn $RELEASE_DIR/$RELEASE_NAME $CURRENT_LINK
chown -h $DEPLOY_USER:$DEPLOY_USER $CURRENT_LINK

# Run upgrade
cd $CURRENT_LINK && php bin/magento setup:upgrade --keep-generated
cd $CURRENT_LINK && php bin/magento cache:flush

# Enable maintenance briefly for final sync
php bin/magento maintenance:enable
# Any final sync operations
php bin/magento maintenance:disable

# Cleanup old releases (keep last 3)
cd $RELEASE_DIR && ls -t | tail -n +4 | xargs rm -rf

echo "==> Deployed $RELEASE_NAME"
```

---

## 7. Database Migration

### 7.1 The `setup:upgrade` Command

The `bin/magento setup:upgrade` command is the central hub for all database-related deployment operations. It:

1. Reads `app/etc/config.php` to determine which modules are installed
2. Runs `SetupPatch` and `SetupDataPatch` classes that have not yet been applied
3. Applies declarative schema XML changes (`db_schema.xml`)
4. Upgrades the database schema version for each module
5. Cleans up old setup versions

```bash
# Standard upgrade (safe for production with --keep-generated)
php bin/magento setup:upgrade \
  --keep-generated \
  --safe-installer=1

# Verbose output for debugging
php bin/magento setup:upgrade -vvv

# Dry run (show what would be executed)
php bin/magento setup:upgrade --dry-run
```

The `--safe-installer=1` flag (available in 2.4.6+) prevents data corruption during concurrent operations by using `INSERT ... ON CONFLICT` patterns.

### 7.2 Declarative Schema

Magento 2.3+ introduced declarative schema (`db_schema.xml`) which allows module developers to declare the desired database state rather than writing upgrade scripts. Magento automatically determines the delta and applies it.

**`Vendor/Module/etc/db_schema.xml`:**

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">
    <table name="vendor_custom_entity"
           resource="default"
           engine="innodb"
           comment="Custom Entity Table">
        <column xsi:type="int"
                name="entity_id"
                padding="10"
                unsigned="true"
                nullable="false"
                identity="true"
                comment="Entity ID"/>
        <column xsi:type="varchar"
                name="name"
                nullable="false"
                length="255"
                comment="Entity Name"/>
        <column xsi:type="varchar"
                name="sku"
                nullable="false"
                length="64"
                comment="SKU"/>
        <column xsi:type="decimal"
                name="price"
                nullable="false"
                precision="10"
                scale="2"
                comment="Price"/>
        <column xsi:type="timestamp"
                name="created_at"
                nullable="false"
                default="CURRENT_TIMESTAMP"
                comment="Created At"/>
        <column xsi:type="timestamp"
                name="updated_at"
                nullable="false"
                default="CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP"
                comment="Updated At"/>
        <column xsi:type="smallint"
                name="status"
                nullable="false"
                default="1"
                comment="Status: 1=Active, 0=Inactive"/>

        <constraint xsi:type="primary"
                     referenceId="PRIMARY">
            <column name="entity_id"/>
        </constraint>
        <constraint xsi:type="unique"
                     referenceId="VENDOR_CUSTOM_ENTITY_SKU">
            <column name="sku"/>
        </constraint>
        <index referenceId="IDX_VENDOR_CUSTOM_ENTITY_STATUS"
               indexType="btree">
            <column name="status"/>
        </index>
        <index referenceId="IDX_VENDOR_CUSTOM_ENTITY_CREATED_AT"
               indexType="btree">
            <column name="created_at"/>
        </index>
    </table>
</schema>
```

After modifying `db_schema.xml`, generate the `.xsd` whitelist to preserve custom column modifications made by merchants:

```bash
# Generate whitelist (required after every schema change)
php bin/magento setup:db-declaration:generate-whitelist \
  --module=Vendor_CustomModule

# This creates etc/whitelist.json
```

### 7.3 Data Patches

Data patches (`SetupDataPatch`) are used for one-time data migrations or updates that need to run once during deployment. They are idempotent via the `$version` decorator.

**`Vendor/Module/Setup/Patch/Data/AddDefaultCategories.php`:**

```php
<?php
declare(strict_types=1);

namespace Vendor\CustomModule\Setup\Patch\Data;

use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\Patch\PatchVersionInterface;

class AddDefaultCategories implements
    DataPatchInterface,
    PatchVersionInterface
{
    private const MODULE_VERSION = '1.0.0';

    /** @var ModuleDataSetupInterface */
    private ModuleDataSetupInterface $moduleDataSetup;

    public function __construct(
        ModuleDataSetupInterface $moduleDataSetup
    ) {
        $this->moduleDataSetup = $moduleDataSetup;
    }

    public static function getVersion(): string
    {
        return self::MODULE_VERSION;
    }

    public function apply(): void
    {
        $setup = $this->moduleDataSetup->getConnection();

        // Check if data already exists (idempotency)
        $select = $setup->select()
            ->from($this->moduleDataSetup->getTable('catalog_category_entity'))
            ->where('entity_id = ?', 2);

        if ($setup->fetchRow($select)) {
            return; // Already applied, skip
        }

        $setup->startSetup();

        // Insert root category
        $this->moduleDataSetup->getConnection()->insert(
            $this->moduleDataSetup->getTable('catalog_category_entity'),
            [
                'entity_id' => 2,
                'attribute_set_id' => 3,
                'parent_id' => 1,
                'name' => 'Default Category',
                'level' => 1,
                'path' => '1/2',
                'position' => 0,
                'children_count' => 0,
                'created_at' => date('Y-m-d H:i:s'),
                'updated_at' => date('Y-m-d H:i:s'),
            ]
        );

        $setup->endSetup();
    }

    public static function getDependencies(): array
    {
        return []; // No dependencies
    }

    public function getAliases(): array
    {
        return []; // No aliases
    }
}
```

### 7.4 Schema Patch for Complex Changes

For complex schema changes that cannot be expressed declaratively, use `SchemaPatchInterface`:

```php
<?php
declare(strict_types=1);

namespace Vendor\CustomModule\Setup\Patch\Schema;

use Magento\Framework\DB\Adapter\AdapterInterface;
use Magento\Framework\Setup\SchemaSetupInterface;
use Magento\Framework\Setup\Patch\SchemaPatchInterface;

class AddLegacyOrderIndex implements SchemaPatchInterface
{
    private SchemaSetupInterface $schemaSetup;

    public function __construct(SchemaSetupInterface $schemaSetup)
    {
        $this->schemaSetup = $schemaSetup;
    }

    public function apply(): void
    {
        $connection = $this->schemaSetup->getConnection();
        $tableName = $this->schemaSetup->getTable('sales_order');

        // Create index that existed in M2.3 but was removed
        if (!$connection->indexExists($tableName, 'SALES_ORDER_GRAND_TOTAL')) {
            $connection->addIndex(
                $tableName,
                'SALES_ORDER_GRAND_TOTAL',
                ['grand_total', 'base_grand_total'],
                AdapterInterface::INDEX_TYPE_INDEX
            );
        }
    }

    public static function getAliases(): array
    {
        return [];
    }

    public static function getDependencies(): array
    {
        return [];
    }
}
```

---

## 8. Production Deployment Checklist

### 8.1 Pre-Deployment Verification

Before triggering a production deployment, verify the following:

```bash
# 1. Verify all modules are registered and versioned correctly
php bin/magento module:status

# 2. Check for pending database patches
php bin/magento setup:db:status

# 3. Verify static content version consistency
php bin/magento static-content:deploy --dry-run

# 4. Validate generated code is present from CI build
test -f generated/code/.metadatalock && echo "Build lock exists" || echo "WARNING: No build lock"

# 5. Verify indexer status
php bin/magento indexer:status

# 6. Check for long-running operations
php bin/magento queue:consumers:list
```

### 8.2 Indexer Configuration

Indexer configuration is one of the most impactful production settings. The `indexer.set` command controls whether indexers run in real-time (`realtime`) or on a schedule (`schedule`).

```bash
# Check current indexer modes
php bin/magento indexer:status

# Set all catalog indexers to schedule mode (preferred for production)
php bin/magento indexer:set-mode schedule
php bin/magento indexer:set-mode schedule --indexer=catalog_category_product
php bin/magento indexer:set-mode schedule --indexer=catalog_product_category

# Configure schedule interval (default is every 1 minute)
# Edit var/indexer.xml or use the admin panel:
# Stores > Settings > Configuration > Catalog > Catalog > Catalog Search >Indexer Optimization

# Rebuild indexers before deployment (reduces post-deploy load)
php bin/magento indexer:reindex
```

**Indexer modes:**

| Mode | Use Case | Pros | Cons |
|---|---|---|---|
| **realtime** | Small catalogs, dev | Data immediately searchable | Heavy on save events |
| **schedule** | Production, large catalogs | Batched processing, predictable load | Search delay until cron runs |

### 8.3 Cache Flush Strategy

```bash
# Flush all caches (full clear)
php bin/magento cache:flush

# Clean specific cache types (preserves other caches)
php bin/magento cache:clean   # Empties cache storage
php bin/magento cache:flush   # Flushes storage AND flushes all cache types

# Per-type cache management
php bin/magento cache:enable  # Enable specific types
php bin/magento cache:disable # Disable specific types
```

# In production deploy script (recommended):
# 1. Flush config cache AFTER env.php/config.php changes
php bin/magento cache:flush config
# 2. Flush layout cache AFTER theme/template changes
php bin/magento cache:flush layout
# 3. Flush full page cache LAST
php bin/magento cache:flush full_page
```

### 8.4 Cron Schedule

Magento's cron system schedules indexer updates, customer notification emails, dirty catalog price recalculations, and custom scheduled tasks. Verify the cron is running correctly:

```bash
# Check cron is configured (run as web server user)
crontab -l -u www-data

# Standard Magento crontab entries:
# */5 * * * * php /var/www/magento/bin/magento cron:run --group=index --without-lock >> /var/log/magento/cron.log 2>&1
# */15 * * * * php /var/www/magento/bin/magento cron:run --group=default >> /var/log/magento/cron.log 2>&1
# */30 * * * * php /var/www/magento/bin/magento cron:run --group=ddg --without-lock >> /var/log/magento/cron.log 2>&1

# Run cron manually (for testing)
php bin/magento cron:run --group=index

# Verify cron schedule
php bin/magento cron:schedule:show
```

### 8.5 Logging Configuration

Production logging should be configured to write to a centralized logging system (ELK stack, CloudWatch, etc.) rather than local files:

```php
<?php
// app/etc/env.php — Production logging configuration
return [
    // ... other config ...
    'log' => [
        'level' => \Monolog\Logger::WARNING,  // Only WARNING and above in production
        'handler' => \Magento\Framework\Logger\Handler\Syslog::class,
        'handlerStream' => null,  // Disable file logging
    ],
    'system' => [
        'default' => [
            'dev/debug' => '0',           // Disable debug logging
            'dev/log' => 'warning',       // WARNING level only
            'system/log' => 'exception',  // Log exceptions only
        ],
    ],
];
```

### 8.6 Complete Production Deploy Script

```bash
#!/bin/bash
# scripts/production-deploy.sh

set -e

MAGENTO_ROOT="/var/www/magento"
DEPLOY_USER="www-data"
BACKUP_RETENTION_DAYS=7
LOG_FILE="/var/log/magento/deploy-$(date +%Y%m%d-%H%M%S).log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "==> Starting production deployment"

# ── Pre-deployment checks ─────────────────────────────────────────────
log "Checking maintenance mode status..."
if [ -f "$MAGENTO_ROOT/var/.maintenance.flag" ]; then
    log "ERROR: Maintenance mode is already enabled. Aborting."
    exit 1
fi

log "Checking disk space..."
DISK_USAGE=$(df --output=pcent "$MAGENTO_ROOT" | tail -1 | tr -d ' ')
if [ "$DISK_USAGE" -gt 85 ]; then
    log "ERROR: Disk usage at ${DISK_USAGE}%. Aborting."
    exit 1
fi

# ── Enable maintenance ────────────────────────────────────────────────
log "Enabling maintenance mode..."
php bin/magento maintenance:enable --ip="$(cat /var/www/maintenance_whitelist.txt)"

# ── Backup current state ──────────────────────────────────────────────
log "Creating rollback point..."
cp -r "$MAGENTO_ROOT/var/.maintenance.flag" /tmp/rollback.flag 2>/dev/null || true
tar -czf "/tmp/magento-rollback-$(date +%Y%m%d-%H%M%S).tar.gz" \
  -C "$MAGENTO_ROOT" vendor generated app/etc 2>/dev/null || true

# ── Code sync ─────────────────────────────────────────────────────────
log "Syncing new code..."
rsync -az --delete \
  vendor/ \
  generated/ \
  pub/static/ \
  app/etc/config.php \
  $MAGENTO_ROOT/

# ── Database upgrade ───────────────────────────────────────────────────
log "Running database upgrade..."
php bin/magento setup:upgrade \
  --keep-generated \
  --safe-installer=1 \
  --no-ansi

# ── Re-compile DI if needed ───────────────────────────────────────────
if [ -n "$NEED_DI_COMPILE" ]; then
    log "Running DI compilation..."
    php bin/magento setup:di:compile
fi

# ── Deploy static content (if not done in build) ──────────────────────
if [ -n "$DEPLOY_STATIC_CONTENT" ]; then
    log "Deploying static content..."
    php bin/magento setup:static-content:deploy \
      --language=en_US \
      --language=de_DE \
      --language=fr_FR \
      --jobs=$(nproc) \
      --no-ansi
fi

# ── Cache management ──────────────────────────────────────────────────
log "Flushing caches..."
php bin/magento cache:flush config
php bin/magento cache:flush full_page
php bin/magento cache:clean

# ── Warm up caches ────────────────────────────────────────────────────
log "Warming up application..."
curl -s "https://www.example.com/" > /dev/null &
curl -s "https://www.example.com/category/" > /dev/null &
curl -s "https://www.example.com/catalog/product/view/" > /dev/null &
wait

# ── Verify deployment ─────────────────────────────────────────────────
log "Verifying deployment..."
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://www.example.com/)
if [ "$HTTP_STATUS" -eq 200 ]; then
    log "Health check passed (HTTP $HTTP_STATUS)"
else
    log "ERROR: Health check failed (HTTP $HTTP_STATUS)"
    log "Re-enabling maintenance mode..."
    php bin/magento maintenance:enable
    exit 1
fi

# ── Disable maintenance ──────────────────────────────────────────────
log "Disabling maintenance mode..."
php bin/magento maintenance:disable

# ── Notify ────────────────────────────────────────────────────────────
log "==> Deployment completed successfully at $(date)"

# ── Cleanup old backups ──────────────────────────────────────────────
find /tmp -name "magento-rollback-*.tar.gz" -mtime +$BACKUP_RETENTION_DAYS -delete
```

---

## 9. Cloud Deployment — Adobe Commerce Cloud

### 9.1 Architecture Overview

Adobe Commerce Cloud (formerly Magento Commerce Cloud) is a managed, auto-scaling Platform-as-a-Service built on top of AWS/Azure/GCP. The infrastructure is defined as code via `.magento.app.yaml` and `magento.vars.php`. Key architectural components:

```
┌─────────────────────────────────────────────────────────────────┐
│                      CDN (Fastly)                               │
│              (caches static content, provides WAF)               │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                    Load Balancer                                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│              Auto-Scaling Web Node(s)                            │
│         ┌──────────────────┐  ┌──────────────────┐              │
│         │   PHP-FPM (N)   │  │   PHP-FPM (N)    │              │
│         │   Magento 2.4.8   │  │   Magento 2.4.8   │              │
│         └────────┬─────────┘  └────────┬─────────┘              │
│                  │                       │                       │
│         ┌────────▼────────────────────────▼────────┐              │
│         │         Shared NFS Mount (media/, var/)  │              │
│         └──────────────────────────────────────────┘              │
└──────────────────────────┬──────────────────────────────────────┘
                           │
          ┌─────────────────┼─────────────────┐
          │                 │                  │
┌────────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
│   Database    │  │    Redis    │  │   Elasticsearch │
│  (AWS RDS)    │  │  (ElastiCache)│ │    (OpenSearch)  │
└───────────────┘  └─────────────┘  └─────────────┘
```

### 9.2 `.magento.app.yaml` — Infrastructure Definition

```yaml
# .magento.app.yaml
name: mymagento
type: php:8.2

build:
  flavor: magento
  hooks:
    # Build hooks run during the build phase (no runtime access)
    build: |
      set -e
      composer install --no-interaction --prefer-dist --optimize-autoloader
      php bin/magento setup:static-content:deploy \
        --language=en_US \
        --language=de_DE \
        --language=fr_FR \
        --jobs=4
      php bin/magento setup:di:compile

    post_build: |
      # Execute quality checks after build
      echo "Build completed"

deploy: |
  # Deploy hooks run AFTER the application is deployed to a node
  # but BEFORE the container starts serving traffic
  set -e

  # Clean and flush cache BEFORE traffic
  php bin/magento cache:clean
  php bin/magento cache:flush

  # Run database upgrade
  php bin/magento setup:upgrade \
    --keep-generated \
    --safe-installer=1

  # Reindex in schedule mode
  php bin/magento indexer:reindex

  # Warm up critical pages
  for URL in "/" "/cms-page" "/category"; do
    curl -s "https://${MAGENTO_URL}${URL}" > /dev/null || true
  done

runtime:
  extensions:
    - xsl
    - redis
    - sodium
    - intl
    - bc-math
    - socket
    - ssh2
  ini:
    display_errors: "0"
    max_execution_time: "1800"
    memory_limit: "2G"

relationships:
  database: "mysql:mysql"
  redis: "redis:redis"
  elasticsearch: "elasticsearch:elasticsearch"

web:
  locations:
    "/":
      # Serve static content from pub/static (CDN-backed)
      root: "pub/static"
      expires: "1y"
      passthru: "/index.php"
      scripts: false
      status: 200

    "/media":
      root: "pub/media"
      expires: "1y"
      scripts: false
      status: 200

    "/var":
      root: "var"
      expires: "0"
      scripts: false
      passthru: "/index.php"
      status: 200

    "/setup":
      root: "setup"
      expires: "0"
      scripts: false
      passthru: "/index.php"
      status: 200

mounts:
  # EFS/NFS-mounted directories (shared across all nodes)
  "var": "shared:var"
  "pub/media": "shared:media"
  "generated": "shared:generated"

processes:
  # Background processes (cron)
  cron:
    spec: "* * * * *"
    cmd: "php bin/magento cron:run"
```

### 9.3 Environment Variables

Adobe Commerce Cloud provides environment-specific variables through the Project Web UI, CLI (`magento-cloud variable:create`), or `.magento.env.yaml`:

```yaml
# .magento.env.yaml — committed to repository
stage:
  global:
    MAGE_MODE: production
    LOG_LEVEL: warning
  build:
    composer_options: "--optimize-autoloader --no-dev"
    SKIP_SCD: false
  deploy:
    # Minimum log level (exception = only exceptions)
    SYSTEM_LOGGING: exception
    # Disable debug mode
    DEV_DEBUG: false
    # Elasticsearch settings
    ELASTICSEARCH_PREFIX: ""
    # Admin URL secret (randomized per environment)
    ADMIN_URL_SUFFIX: "${ADMIN_URL_SUFFIX}"
  route:
    # HTTPS enforced on all routes
    "https://{all}":
      type: redirect
      redirect:
        from: "/.well-known/"
        to: "/"
        protocol: HTTPS
```

```bash
# Set environment variables via CLI
magento-cloud variable:create --name=ADMIN_EMAIL --value=admin@example.com --environment=production --json=false
magento-cloud variable:create --name=MAGE_MODE --value=production --environment=production
magento-cloud variable:create --name=LOG_LEVEL --value=warning --environment=production

# List all environment variables
magento-cloud variable:list
```

### 9.4 Build and Deploy Hooks

The `.magento.app.yaml` `hooks` section defines two lifecycle hooks:

**Build Hook** (`build`): Runs on a dedicated build node with PHP and Composer installed but no MySQL/Redis. This is where static content deployment and DI compilation happen.

```yaml
build:
  hooks:
    build: |
      set -e
      composer install --no-interaction
      php bin/magento setup:static-content:deploy \
        --language=en_US \
        --language=de_DE \
        --jobs=4
      php bin/magento setup:di:compile
```

**Deploy Hook** (`deploy`): Runs on each target node after code is copied but before traffic is routed. Each node in an auto-scaling group runs this hook independently. The deploy hook has access to all services (MySQL, Redis, Elasticsearch).

```yaml
deploy: |
  set -e
  php bin/magento maintenance:enable
  php bin/magento setup:upgrade --keep-generated --safe-installer=1
  php bin/magento cache:flush
  php bin/magento indexer:reindex
  php bin/magento maintenance:disable
```

### 9.5 Multi-Environment Configuration

Adobe Commerce Cloud supports three environments: `integration` (shared dev), `staging` (near-production), and `production` (live). Each can have different configurations:

```yaml
# .magento/routes.yaml
"https://www.example.com/":
  type: upstream
  upstream: mymagento:php
  cache:
    enabled: true
    headers: [Accept,Accept-Language,X-Magento-Tags]
    sMaxAge: 86400

"https://staging.example.com/":
  type: upstream
  upstream: mymagento:php
  cache:
    enabled: true

# .magento/services.yaml
mysql:
  type: mysql:3.6
  disk: 5120
  configuration:
    max_connections: 300
    innodb-buffer-size: 512M

redis:
  type: redis:7.0
  relationships:
    cache: "redis:redis"
    session: "redis:redis"

elasticsearch:
  type: elasticsearch:2.4
  disk: 512
  relationships:
    search: "elasticsearch:elasticsearch"
```

### 9.6 Cloud CI Integration

For Adobe Commerce Cloud, the GitHub Actions pipeline connects to the Magento Cloud CLI for deployment:

```yaml
# .github/workflows/cloud-deploy.yml
name: Adobe Commerce Cloud Deploy

on:
  push:
    branches:
      - main

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'

      - name: Install Magento Cloud CLI
        run: |
          curl -sS https://accounts.magento.cloud/cli/installer | php
          export PATH="$HOME/.magento-cloud/bin:$PATH"
          echo "$MAGENTO_CLOUD_CLI_KEY" | magento-cloud auth:login --no-interaction

      - name: Merge and push to staging
        env:
          MAGENTO_CLOUD_CLI_KEY: ${{ secrets.MAGENTO_CLOUD_CLI_KEY }}
        run: |
          export PATH="$HOME/.magento-cloud/bin:$PATH"
          magento-cloud merge --force --no-interaction staging

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'

      - name: Install Magento Cloud CLI
        run: |
          curl -sS https://accounts.magento.cloud/cli/installer | php
          export PATH="$HOME/.magento-cloud/bin:$PATH"
          echo "$PROD_MAGENTO_CLOUD_CLI_KEY" | magento-cloud auth:login --no-interaction

      - name: Push to production
        env:
          PROD_MAGENTO_CLOUD_CLI_KEY: ${{ secrets.PROD_MAGENTO_CLOUD_CLI_KEY }}
        run: |
          export PATH="$HOME/.magento-cloud/bin:$PATH"
          magento-cloud push --force --no-interaction production
```

---

## Summary

A well-designed Magento 2.4.8 CI-CD pipeline is not a single tool or a single script — it is a coordinated system spanning version control strategy, automated testing at multiple layers, build-time compilation, database migration handling, and production deployment strategies tailored to business tolerance for downtime.

**Key principles to remember:**

1. **Build in CI, deploy artifacts.** Run `composer install`, `setup:static-content:deploy`, and `setup:di:compile` once in CI, archive the artifacts, and deploy those artifacts to every environment. This guarantees consistency.
2. **Never run `setup:upgrade` without `--keep-generated`** in production if the DI compilation is already done in the build phase.
3. **Use `--safe-installer=1`** in `setup:upgrade` on production to prevent data corruption from concurrent requests during schema changes.
4. **Indexers in `schedule` mode** are non-negotiable for production stores with catalogs above a few thousand SKUs.
5. **Zero-downtime is architectural.** It requires either shared storage (NFS/EFS) for blue-green deployments, CDN-cached static content, or Redis-backed readonly mode during maintenance windows.
6. **Adobe Commerce Cloud handles the hard parts** (auto-scaling, shared storage, CDN, WAF) but requires discipline in `.magento.app.yaml` hooks and environment variable management.

The investment in a mature deployment pipeline pays dividends in reduced incident count, faster recovery from failures, and the confidence to deploy frequently — which is the foundation of every successful Magento operation.