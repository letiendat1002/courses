---
title: "17 - Configuration System & Scope Resolution"
description: "Magento 2.4.8 configuration architecture: env.php, config.php, core_config_data layering, config scopes, encrypted values, and configuration caching mechanisms."
tags: magento2, configuration, env-php, config-php, core-config-data, config-scope, config-caching, env-config
rank: 17
pathways: [magento2-deep-dive]
see_also:
  - path: "04-data-layer/16-store-hierarchy.md"
    description: "Store hierarchy — config scoping is per store/website, ties to scope resolution"
  - path: "_supplemental/26-store-hierarchy.md"
    description: "Store hierarchy authoritative — ScopeConfigInterface detail"
  - path: "18-security-hardening/README.md"
    description: "Security — encrypted config values and secret management"
---

# Configuration System & Scope Resolution

Magento's configuration system is one of the most powerful and most misunderstood parts of the platform. Configuration values determine how the system behaves, and they can come from three fundamentally different sources with a strict priority order. Understanding this system is prerequisite to debugging why a setting "doesn't take effect," how to safely store secrets, and how to manage configuration across different environments.

---

## 1. The Three Sources of Configuration

### Configuration Priority (Most Specific Wins)

```
1. core_config_data (database)     ← Highest priority
       ↓
2. env.php (environment)          ← Middle priority
       ↓
3. config.php + etc/config.php    ← Lowest priority (defaults)
```

Every configuration path in Magento can be defined at all three levels. The most specific scope wins at runtime.

### config.php — Application Defaults

`app/etc/env.php` and `app/etc/config.php` are the static defaults applied at installation time.

```php
<?php
// app/etc/config.php — generated during installation
return [
    'backend' => [
        'frontName' => 'admin'
    ],
    'crypt' => [
        'key' => 'your_encryption_key_here'
    ],
    'db' => [
        'connection' => [
            'index' => [
                'host' => 'localhost',
                'dbname' => 'magento',
                'username' => 'root',
                'password' => '',
                'model' => 'mysql8',
                'engine' => 'innodb'
            ]
        ]
    ],
    'system' => [
        'default' => 'general' // section to show on install
    ]
];
```

**Key characteristic:** Changes here require file system access and are version-controlled. Used for base infrastructure (DB connection, encryption key, admin front name).

### env.php — Environment Overrides

```php
<?php
// app/etc/env.php — environment-specific overrides
return [
    'system' => [
        'default' => [
            'carriers' => [
                'flatrate' => [
                    'active' => '1'
                ]
            ]
        ]
    ],
    'db' => [
        'connection' => [
            'index' => [
                'password' => 'prod_password_here'
            ]
        ]
    ],
    'cache' => [
        'frontend' => [
            'default' => [
                'backend' => 'redis',
                'backend_options' => [
                    'server' => 'redis-host',
                    'port' => '6379'
                ]
            ]
        ]
    ]
];
```

**Key characteristic:** Environment-specific (dev/staging/prod). Not in version control (it's in `.gitignore` by default). Holds secrets and environment-specific overrides.

### core_config_data — Database Scope

The `core_config_data` table is where admin-saved configuration lives:

```sql
CREATE TABLE core_config_data (
    config_id     INT PRIMARY KEY AUTO_INCREMENT,
    scope          VARCHAR(32) NOT NULL DEFAULT 'default',
    scope_id       INT NOT NULL DEFAULT 0,
    path           VARCHAR(255) NOT NULL,
    value          TEXT
);

-- Examples:
-- Global default (scope='default', scope_id=0)
INSERT INTO core_config_data (scope, scope_id, path, value)
VALUES ('default', 0, 'carriers/fedex/active', '1');

-- Website-specific override
INSERT INTO core_config_data (scope, scope_id, path, value)
VALUES ('websites', 1, 'carriers/fedex/active', '0');

-- Store-view-specific override
INSERT INTO core_config_data (scope, scope_id, path, value)
VALUES ('stores', 3, 'general/locale/code', 'de_DE');
```

**Key characteristic:** Editable via admin UI (Stores → Settings → Configuration). Scoped to default/website/store. Clear cache required after changes.

---

## 2. Config Path Structure

### Path Syntax

Configuration paths follow a `section/group/field` pattern:

```
carriers/fedex/path          ← section=carriers, group=fedex, field=path
general/locale/code          ← section=general, group=locale, field=code
catalog/product/product_image_customer
```

### Where Paths Are Declared

The system.xml file in a module declares which configuration fields exist:

```xml
<!-- etc/adminhtml/system.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Config:etc/system.xsd">
    <section id="carriers" translate="label" type="text" sortOrder="320" showInDefault="1">
        <group id="fedex" translate="label" type="text" sortOrder="40" showInDefault="1">
            <field id="active" translate="label" type="select" sortOrder="1" showInDefault="1">
                <label>Enabled</label>
                <source_model>Magento\Config\Model\Config\Source\Yesno</source_model>
            </field>
            <field id="path" translate="label" type="text" sortOrder="2">
                <label>Path to Fedex Binary</label>
            </field>
        </group>
    </section>
</config>
```

---

## 3. Config Scopes (core_config_data)

### Scope Constants

```php
<?php
// Magento\Store\Model\ScopeInterface
const SCOPE_STORES   = 'stores';    // Store view level
const SCOPE_GROUPS   = 'groups';    // Store (group) level
const SCOPE_WEBSITES  = 'websites';  // Website level
const SCOPE_DEFAULT  = 'default';   // Global default level
```

### Scope Precedence

When reading `general/locale/code`:

```
1. Check SCOPE_STORES for store_id = 5 → value found: 'de_DE'
   → Return 'de_DE'

   ↓ (not found)
2. Check SCOPE_GROUPS for group_id = 2 → value found: 'fr_FR'
   → Return 'fr_FR'

   ↓ (not found)
3. Check SCOPE_WEBSITES for website_id = 1 → value found: 'en_US'
   → Return 'en_US'

   ↓ (not found)
4. Check SCOPE_DEFAULT (scope='default', scope_id=0)
   → Return default from system.xml or null
```

This means the admin's "Current Scope" selector directly changes which row in `core_config_data` gets created or edited.

### Reading Config in Code

```php
<?php
// Inject \Magento\Framework\App\Config\ScopeConfigInterface

public function __construct(
    \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig
) {
    $this->scopeConfig = $scopeConfig;
}

// Read at default scope (global)
$globalDefault = $this->scopeConfig->getValue(
    'carriers/fedex/path',
    \Magento\Store\Model\ScopeInterface::SCOPE_DEFAULT
);

// Read at website scope
$websiteValue = $this->scopeConfig->getValue(
    'carriers/fedex/path',
    \Magento\Store\Model\ScopeInterface::SCOPE_WEBSITES,
    $websiteId
);

// Read at store view scope
$storeValue = $this->scopeConfig->getValue(
    'general/locale/code',
    \Magento\Store\Model\ScopeInterface::SCOPE_STORES,
    $storeId
);

// Current store scope (most common)
$currentStoreValue = $this->scopeConfig->getValue(
    'general/locale/code',
    'stores',  // or null for current store
    $this->storeManager->getStore()->getId()
);
```

---

## 4. The Priority Conflict Problem

### When All Three Define the Same Path

This is the most common source of "my config change doesn't work" bugs:

```php
// app/etc/config.php — set at install time
'system' => [
    'default' => [
        'general' => ['locale' => ['code' => 'en_US']]
    ]
]

// app/etc/env.php — environment override
'system' => [
    'default' => [
        'general' => ['locale' => ['code' => 'de_DE']]  // overrides config.php
    ]
]

// core_config_data — admin override
-- WHERE scope='websites' AND scope_id=1 AND path='general/locale/code'
-- value = 'fr_FR'  ← This ALWAYS wins if it exists
```

If there's a row in `core_config_data` for `scope='stores'` and `scope_id=3`, the admin UI setting overrides BOTH env.php and config.php.

### Debugging Config Priority Issues

**Step 1: Check core_config_data directly**

```sql
SELECT * FROM core_config_data
WHERE path LIKE '%fedex%';

-- Output might show:
-- scope=default, scope_id=0, path=carriers/fedex/active, value=1
-- scope=websites, scope_id=1, path=carriers/fedex/active, value=0
```

**Step 2: Clear config cache**

```bash
bin/magento cache:clean config
# NOT just cache:flush — that clears ALL caches
# config:clean specifically targets compiled configuration
```

**Step 3: Verify env.php structure**

```php
<?php
// In CLI: php bin/magento config:show carriers/fedex/path
// Output shows the resolved value (not the source)

// Or in code, dump all three levels:
echo "config.php: " . ($configArray['system']['default']['carriers']['fedex']['path'] ?? 'not set') . "\n";
echo "env.php: " . ($envArray['system']['default']['carriers']['fedex']['path'] ?? 'not set') . "\n";
```

### The env.php Override Gotcha

env.php is processed at bootstrap time and merged into a global configuration array. But `core_config_data` is read from the DB every time (with caching). If you edit env.php but don't see changes, the issue is almost always `core_config_data` having a conflicting value that takes precedence.

---

## 5. Configuration Caching

### What Gets Cached

After first request, compiled configuration is cached in:
- Redis (if configured) as `CONFIG`
- Filesystem as `generated/code/` or `var/cache/` (fallback)

### Cache Invalidation

```bash
# Clean only config cache (preserves other caches)
bin/magento cache:clean config

# Flush all caches
bin/magento cache:flush

# Check cache status
bin/magento cache:status
```

**When config cache auto-invalidates:**
- `setup:upgrade` — module install/upgrade triggers config rebuild
- `app:config:import` — imports config from config.php/env.php
- Module enable/disable
- After running `setup:di:compile`

### Manual Cache Invalidation in Code

```php
<?php
// Inject \Magento\Framework\App\Cache\Type\Config and \Magento\Framework\App\Cache\Manager
public function __construct(
    \Magento\Framework\App\Cache\Type\Config $configCacheType,
    \Magento\Framework\App\Cache\Manager $cacheManager
) {
    $this->configCacheType = $configCacheType;
    $this->cacheManager = $cacheManager;
}

// Clean config cache directly
$this->configCacheType->clean();

// Or invalidate via cache manager (triggers proper event dispatch)
$this->cacheManager->clean(['config']);
```

---

## 6. Encrypted Configuration Values

### env.php vs core_config_data for Secrets

The encryption key (from `crypt/key` in config.php/env.php) is used to encrypt values stored in `core_config_data` that are marked as sensitive.

```php
<?php
// In system.xml, mark a field as sensitive:
<field id="api_key" translate="label" type="obscure">
    <label>API Key</label>
    <backend_model>Magento\Config\Model\Config\Backend\Encrypted</backend_model>
</field>
```

Encrypted fields:
- Stored in `core_config_data` with value prefixed by a hash marker
- Decrypted automatically when read via `ScopeConfigInterface`
- Never appear in logs, debug output, or error messages
- The encryption key in `app/etc/env.php` must match across environments

### Changing the Encryption Key

```bash
# Generate a new key
openssl rand -base64 16

# Update app/etc/env.php:
'crypt' => [
    'key' => 'new_key_here'
]

# Then re-encrypt existing values:
bin/magento crypt:upgrade-key

# Warning: this requires the old key to decrypt existing values first
```

### Secret Storage Best Practices

| Secret Type | Storage Location | Why |
|-------------|-----------------|-----|
| Encryption key | `env.php` (crypt/key) | Must be consistent across instances |
| DB password | `env.php` (db/connection/*/password) | Environment-specific, not in Git |
| Payment API keys | `core_config_data` (encrypted) | Admin-managed, per-environment |
| OAuth tokens | `core_config_data` (encrypted) | Admin-managed |

---

## 7. Environment-Specific Configuration Patterns

### Development Environment Setup

```php
<?php
// app/etc/env.php for development
return [
    'system' => [
        'default' => [
            'dev/debug' => ['show_debug' => '1'],
            'dev/template' => ['allow_symlink' => '0'],
            'system' => ['logging' => ['level' => 'DEBUG']]
        ]
    ]
];
```

### Staging Environment

```php
<?php
// app/etc/env.php for staging
return [
    'system' => [
        'default' => [
            'dev/debug' => ['show_debug' => '0'],
            'system' => ['logging' => ['level' => 'WARNING']]
        ]
    ],
    'cache' => [
        'frontend' => [
            'default' => [
                'backend' => 'redis',
                'backend_options' => [
                    'server' => 'staging-redis-host'
                ]
            ]
        ]
    ]
];
```

### Configuration Export/Import

```bash
# Export current config to files (idempotent, good for deployment)
bin/magento app:config:dump

# This writes to app/etc/config.php and app/etc/env.php
# Then commit these files to deploy to other environments

# Import config from files (during deployment)
bin/magento app:config:import

# Import validates changes before applying
# Fails if environment-specific paths would be overwritten
```

---

## 8. Sensitive vs Environment-Specific Configuration

### The Difference

| Aspect | Sensitive | Environment-Specific |
|--------|-----------|---------------------|
| Contains secrets | Yes (API keys, passwords) | No (URLs, feature flags) |
| Stored in | `core_config_data` (encrypted) or `env.php` | `core_config_data` or `env.php` |
| Editable via admin | With backend_model=Encrypted | Yes |
| Should be in Git | No | Partial (non-secret values in config.php) |
| Changes per environment | Key must stay consistent | Yes, varies |

### Proper Separation Example

```php
<?php
// app/etc/config.php — non-secret defaults (version-controlled)
return [
    'system' => [
        'default' => [
            'general' => ['locale' => ['code' => 'en_US']],
            'web' => [
                'unsecure' => ['base_url' => 'https://example.com/'],
                'secure' => ['base_url' => 'https://example.com/']
            ]
        ]
    ]
];

// app/etc/env.php — secrets + environment overrides (NOT in Git)
return [
    'crypt' => ['key' => 'actual_encryption_key_here'],
    'db' => ['connection' => ['default' => ['password' => 'db_password_here']]],
    'system' => [
        'default' => [
            'web' => [
                'secure' => ['base_url' => 'https://prod.example.com/']
            ]
        ]
    ]
];
```

---

## 9. Programmatic Config Reading and Writing

### Reading Configuration

```php
<?php
// Best practice: use ScopeConfigInterface for runtime reads
$value = $this->scopeConfig->getValue('section/group/field', 'stores', $storeId);

// Read multiple values at once (more efficient for same scope)
$configArray = $this->scopeConfig->getValue('carriers', 'websites', $websiteId);

// Check if path exists (returns null if not set)
if ($this->scopeConfig->getValue('carriers/fedex/path') === null) {
    // Not configured
}

// Check if path is explicitly set vs using default
// Use \Magento\Framework\App\Config\ConfigSourceInterface to access sources
```

### Writing Configuration

```php
<?php
// Use \Magento\Framework\App\Config\Storage\WriterInterface for writes
public function __construct(
    \Magento\Framework\App\Config\Storage\WriterInterface $configWriter
) {
    $this->configWriter = $configWriter;
}

// Save to default scope (global)
$this->configWriter->save('section/group/field', 'value');

// Save to website scope
$this->configWriter->save('section/group/field', 'value', 'websites', $websiteId);

// Save to store scope
$this->configWriter->save('section/group/field', 'value', 'stores', $storeId);

// After writing, ALWAYS clean config cache
$this->cacheManager->clean(['config']);
```

### Deleting Configuration

```php
<?php
// Use \Magento\Framework\App\Config\Storage\WriterInterface
// Delete by saving null/empty
$this->configWriter->delete('section/group/field', 'stores', $storeId);
// This removes the row, falling back to lower priority
```

---

## 10. Common Pitfalls and Debugging

### Pitfall 1: Edit env.php but Admin Overrides It

```php
// You edit app/etc/env.php to set:
// 'system' => ['default' => ['general' => ['locale' => ['code' => 'de_DE']]]]

// But admin has 'stores/view 3' with 'de_DE' → admin value wins
// Solution: must delete the row from core_config_data
DELETE FROM core_config_data WHERE scope='stores' AND scope_id=3 AND path='general/locale/code';
```

### Pitfall 2: Change Doesn't Apply Because Cache

```bash
# ALWAYS clear config cache after any config change
bin/magento cache:clean config
# Note: cache:flush also works but is slower (clears ALL caches)
```

### Pitfall 3: env.php Syntax Error Breaks Bootstrap

```php
// app/etc/env.php must be valid PHP that returns an array
// A single syntax error (missing comma, quote) breaks the entire site
// Always validate env.php syntax:
php -l app/etc/env.php
```

### Pitfall 4: Different Encryption Key Per Environment

```bash
# If production has key A and staging has key B,
# encrypted values from production cannot be decrypted on staging
# Always keep encryption keys consistent across environments
# or re-encrypt all values when copying DB between environments
```

### Pitfall 5: config.php vs env.php Confusion

```php
// WRONG: putting secrets in config.php (version-controlled!)
// This exposes secrets if config.php is committed to repo

// CORRECT: secrets go in env.php only
// Non-secrets can go in config.php (install defaults)
```

### Pitfall 6: Import Overwrites Manual Admin Changes

```bash
# If you ran app:config:dump then app:config:import on a system
# where an admin manually changed values in core_config_data
# the import may overwrite those manual changes
# Solution: use deployment scripts that properly sequence config changes
```

---

## Reading List

- [Configuration management in Magento 2](https://developer.adobe.com/commerce/php/development/components/configuration/) — Official docs
- [\Magento\Framework\App\Config\ScopeConfigInterface](https://github.com/magento/magento2/blob/2.4-develop/lib/internal/Magento/Framework/App/Config/ScopeConfigInterface.php)
- [Environment configuration](https://experienceleague.adobe.com/docs/commerce-operations/configuration-guide/) — Adobe docs

---

## Edge Cases & Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| Config change not applying | Stale config cache | `bin/magento cache:clean config` |
| Admin value overrides env.php | `core_config_data` row exists | `DELETE FROM core_config_data WHERE path LIKE '%...%'` |
| Encryption key mismatch | Different keys per environment | Keep encryption key consistent |
| env.php breaks site | Syntax error in PHP | Run `php -l app/etc/env.php` to validate |
| All stores use wrong locale | Default scope value overriding stores | Check `core_config_data` for scope='default' rows |
| Config import fails | DB values conflict with import | Run `bin/magento app:config:import --no-lock` to preview |

---

## Common Mistakes to Avoid

1. ❌ Putting secrets in `config.php` → They go in `env.php` (not in Git)
2. ❌ Editing `core_config_data` manually → Use admin UI or `WriterInterface`
3. ❌ Forgetting to clear config cache → Always `cache:clean config` after changes
4. ❌ Different encryption keys per environment → Causes decryption failures
5. ❌ Thinking env.php overrides `core_config_data` → Database always wins if row exists
6. ❌ Changing env.php without syntax validation → A single typo breaks the whole site
7. ❌ Using `DELETE` on `core_config_data` without WHERE → Removes all config (disaster)

---

*Magento 2 Backend Developer Course — Topic 04 — Data Layer*