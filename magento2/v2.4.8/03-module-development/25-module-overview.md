---
title: "25 - Module Overview & Areas"
description: "Magento 2.4.8 module structure, module areas (frontend/adminhtml/webapi/crontab), module registration, module directory layout, component relationships, and dependency management"
tags: [magento2, module, areas, registration, module-structure, dependency-injection, frontend, adminhtml, webapi, crontab]
rank: 25
pathways: [magento2-deep-dive]
---

# 25 - Module Overview & Areas

Magento 2's architecture is built on a modular system. Understanding modules—their structure, registration, areas, and dependencies—is fundamental to building stable, maintainable extensions. This article covers every aspect of module anatomy in Magento 2.4.8, from the low-level XSD schema validation of `module.xml` to the high-level design patterns of service contracts and repository interfaces.

---

## 1. Module Anatomy

### 1.1 The `etc/module.xml` File

Every Magento module must declare itself in `etc/module.xml`. This file tells Magento three critical things: the module's name, its version, and which other modules must be loaded before it (sequence).

```xml
<?xml version="1.0"?>
<!--
/**
 * Copyright © 2025 CompanyName. All rights reserved.
 */
-->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="CompanyName_ModuleName"
            setup_version="1.0.0">
        <sequence>
            <module name="Magento_Store"/>
            <module name="Magento_Eav"/>
            <module name="Magento_Directory"/>
        </sequence>
    </module>
</config>
```

The schema validation target is the official Magento 2 framework module XSD:

```xml
xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd"
```

This XSD is provided by `Magento_FrameworkModule` and defines the valid structure, cardinality, and data type constraints for every `module.xml` in the application. Magento's `ModuleSetupInfra` class uses this XSD at runtime when parsing configuration files.

### 1.2 The `<sequence>` Tag

`<sequence>` declares **load-order dependencies**. Magento does not guarantee any loading order between modules by default—it only guarantees that modules listed inside `<sequence>` will be fully initialized (including their `di.xml`, events, and configuration) *before* the containing module runs.

If `CompanyName_ModuleName` depends on `Magento_Eav`, you must include `Magento_Eav` in the `<sequence>`. Failure to declare a proper sequence is one of the most common causes of "area not loaded" errors, "class not found" errors during plugin application, and "krecently that was upgraded but its InstallSchema scripts were not re-executed" issues.

The sequence is transitive: if `ModuleA` declares `<sequence><module name="ModuleB"/></sequence>` and `ModuleB` declares `<sequence><module name="ModuleC"/></sequence>`, then `ModuleA` effectively depends on `ModuleC` through the chain. Magento resolves the full dependency graph and uses topological sorting to determine the correct load order.

### 1.3 The `setup_version` Attribute

```xml
<module name="CompanyName_ModuleName" setup_version="1.0.0">
```

The `setup_version` attribute records the module's current schema version in the `setup` table of the database (`setup_module` table). This value is compared against what Magento has stored after every `bin/magento setup:upgrade` run. When the stored version is lower than the declared `setup_version`, Magento runs the corresponding `UpgradeSchema` or `UpgradeData` scripts.

### 1.4 Module State: Enabled vs Disable

A module can exist in two states: **enabled** (Magento processes it) or **disabled** (Magento ignores it). The module state is stored in `app/etc/config.php` (for enabled-by-default, shared across environments) or `app/etc/env.php` (for environment-specific overrides).

```php
// app/etc/config.php — shared enablement
return [
    'modules' => [
        'CompanyName_ModuleName' => 1,   // 1 = enabled
        'Magento_Catalog'        => 1,
        // ...
    ],
];
```

```php
// app/etc/env.php — per-environment override
return [
    'modules' => [
        'CompanyName_ModuleName' => 0,   // 0 = disabled, overrides config.php
    ],
];
```

When a module is disabled, Magento:
- Does NOT autoload its classes
- Does NOT process its `di.xml` files
- Does NOT load its configuration (`routes.xml`, `events.xml`, `acl.xml`, etc.)
- Does NOT fire its observers or run its plugins
- Does NOT register its layout XML files

The distinction between `config.php` and `env.php` is critical: `config.php` is committed to version control and represents the "canonical" module list for the project, while `env.php` contains environment-specific overrides (often containing API keys, database credentials, and local module toggles that should never be committed).

### 1.5 The `registration.php` File

Every module must register itself with Magento's `ComponentRegistrar` system. This happens in the module's root `registration.php` file:

```php
<?php

/**
 * Copyright © 2025 CompanyName. All rights reserved.
 */

declare(strict_types=1);

use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'CompanyName_ModuleName',
    __DIR__
);
```

`ComponentRegistrar::register()` accepts three arguments:

1. **Type constant** — `ComponentRegistrar::MODULE`, `ComponentRegistrar::THEME`, or `ComponentRegistrar::LANGUAGE`
2. **Component name** — the fully-qualified module name using underscores (e.g., `CompanyName_ModuleName`, not `CompanyName-ModuleName`)
3. **Path to the component directory** — `__DIR__` resolves to the directory containing `registration.php`

The registration call is picked up by Magento's autoloader system. During the bootstrap phase, `ComponentRegistrar` builds an in-memory map of all registered modules. This map is then used by `Magento\Framework\App\Bootstrap` to discover which modules are available and their filesystem paths.

---

## 2. Module Directory Structure

### 2.1 Complete Directory Tree

A production-quality Magento 2 module following Adobe Commerce standards has the following structure:

```
Vendor/ModuleName/
├── etc/
│   ├── module.xml                    # Module declaration and sequence
│   ├── di.xml                        # Global dependency injection config
│   ├── adminhtml/
│   │   ├── di.xml                    # Admin-specific DI (adminhtml scope)
│   │   ├── events.xml                # Admin-specific event observers
│   │   ├── menu.xml                  # Admin menu items
│   │   └── routes.xml                # Admin URL routes
│   ├── frontend/
│   │   ├── di.xml                    # Frontend-specific DI
│   │   ├── events.xml                # Frontend-specific event observers
│   │   └── routes.xml                # Frontend URL routes
│   ├── webapi_rest/
│   │   └── di.xml                    # REST API-specific DI
│   ├── webapi_graphql/
│   │   └── di.xml                    # GraphQL API-specific DI
│   ├── crontab/
│   │   └── di.xml                    # Cron-specific DI
│   ├── acl.xml                       # Access Control List
│   ├── events.xml                   # Global event observers
│   ├── system.xml                    # System configuration fields
│   ├── indexer.xml                   # Custom indexer declarations
│   ├── webapi.xml                    # REST API endpoint definitions
│   ├── communications.xml            # Async message queue topology
│   ├── queue_topology.xml            # Queue exchange/binding config
│   └── queue_consumer.xml           # Queue consumer definitions
├── Setup/
│   ├── InstallSchema.php             # Runs once on module install
│   ├── InstallData.php               # Runs once on module install
│   ├── UpgradeSchema.php             # Runs when version increments
│   ├── UpgradeData.php              # Runs when version increments
│   └── Recurring.php                 # Runs after every setup:upgrade
├── Plugin/
│   ├── AroundPlugin.php              # Around plugin example
│   ├── BeforePlugin.php              # Before plugin example
│   └── AfterPlugin.php               # After plugin example
├── Observer/
│   ├── CatalogProductSaveAfter.php   # Event observer example
│   └── CustomerLoginSuccess.php      # Event observer example
├── Controller/
│   ├── Adminhtml/
│   │   └── Vendor/
│   │       └── MyController/
│   │           ├── Index.php
│   │           └── Save.php
│   └── Frontend/
│       └── Vendor/
│           └── MyController/
│               ├── Index.php
│               └── View.php
├── Model/
│   ├── MyEntity.php                  # Entity model (extends AbstractModel)
│   ├── MyEntityFactory.php            # Factory for non-injectable models
│   ├── ResourceModel/
│   │   ├── MyEntity.php               # Resource model (extends AbstractDb)
│   │   └── Collection/
│   │       └── MyEntity.php          # Collection (extends AbstractCollection)
│   └── Api/
│       ├── Data/
│       │   └── MyEntityInterface.php  # Service contract data interface
│       └── MyEntityRepositoryInterface.php  # Service contract repository interface
├── Api/
│   └── MyEntityRepositoryInterface.php  # Repository interface (Api/ = service contract)
├── Block/
│   ├── Adminhtml/
│   │   └── MyEntity/
│   │       ├── Edit.php
│   │       └── Grid.php
│   └── Frontend/
│       └── MyEntity/
│           └── View.php
├── view/
│   ├── adminhtml/
│   │   ├── layout/
│   │   │   ├── vendor_myentity_index.xml
│   │   │   └── vendor_myentity_edit.xml
│   │   ├── templates/
│   │   │   └── myentity/
│   │   │       └── edit.phtml
│   │   └── web/
│   │       └── js/
│   │           └── component.js
│   ├── frontend/
│   │   ├── layout/
│   │   │   └── vendor_myentity_view.xml
│   │   ├── templates/
│   │   │   └── myentity/
│   │   │       └── view.phtml
│   │   └── web/
│   │       ├── css/
│   │       │   └── source/
│   │       │       └── _module.less
│   │       └── js/
│   │           └── component.js
│   └── [area]/                       # Additional area-specific views
├── i18n/
│   ├── en_US.csv                     # English translation (base)
│   └── de_DE.csv                     # German translation
├── Cronjob/
│   └── MyScheduledTask.php           # Cron job class
├── Console/
│   └── Command/
│       └── MyCustomCommand.php      # bin/magento custom command
├── etc/
│   └── email_templates.xml          # Email template declarations
├── Observer/                        # Event observers
├── Plugin/                          # Plugin classes
├── Ui/
│   ├── DataProvider/
│   │   └── MyEntityDataProvider.php
│   └── Component/
│       └── MyComponent.php
├── view/
│   └── base/                         # Shared templates (fallback)
└── registration.php                  # ComponentRegistrar entry point
```

### 2.2 etc/ Directory Config Files

Each configuration file in `etc/` has a specific purpose:

| File | Purpose |
|------|---------|
| `module.xml` | Module declaration (name, version, sequence) |
| `di.xml` | Global dependency injection configuration |
| `adminhtml/di.xml` | Admin-specific DI preferences and virtual types |
| `frontend/di.xml` | Frontend-specific DI preferences and virtual types |
| `webapi_rest/di.xml` | REST API-specific DI |
| `webapi_graphql/di.xml` | GraphQL-specific DI |
| `crontab/di.xml` | Cron-specific DI |
| `events.xml` | Global event-to-observer mappings |
| `adminhtml/events.xml` | Admin-specific event mappings |
| `frontend/events.xml` | Frontend-specific event mappings |
| `routes.xml` | Route-to-controller mappings |
| `adminhtml/routes.xml` | Admin URL routing |
| `frontend/routes.xml` | Frontend URL routing |
| `acl.xml` | ACL resources and roles |
| `menu.xml` | Admin menu structure |
| `system.xml` | System configuration sections/fields/tabs |
| `indexer.xml` | Custom indexer declarations |
| `webapi.xml` | REST API method declarations |
| `webapi_rest/events.xml` | REST-specific event mappings |
| `communications.xml` | Async message bus topology |
| `queue_topology.xml` | Message queue exchange and bindings |
| `queue_consumer.xml` | Queue consumer configurations |
| `email_templates.xml` | Email template declarations |

### 2.3 View Layer (view/[area]/)

The `view/` directory contains all frontend presentation assets organized by area. Each area subdirectory holds:

- **`layout/`** — `.xml` layout files mapping handle names to block structures
- **`templates/`** — `.phtml` template files with PHP markup
- **`web/`** — Static assets (CSS, JavaScript, images, fonts)

```text
view/[area]/web/
├── css/
│   └── source/
│       ├── _module.less              # Base styles (imported by all themes)
│       └── _extend.less              # Theme extension styles
├── js/
│   ├── component.js                  # RequireJS module
│   └── model/
│       └── mymodel.js                # UI component model
└── images/
    └── logo.svg                      # Area-specific images
```

The base theme (`Magento/luma` or `Magento/blank`) provides default styles. Your module's `_module.less` is imported into all themes that inherit from base. The `_extend.less` file provides a safe override mechanism that merges with theme styles rather than replacing them entirely.

---

## 3. Module Areas

### 3.1 What Is an Area?

Magento 2 organizes application behavior into **areas**. Each area represents a logical domain of the application with its own configuration scope, routing, layout, and dependency injection rules. The major areas in Magento 2.4.8 are:

| Area Constant | Value | Purpose |
|---------------|-------|---------|
| `Magento\Framework\App\Area::AREA_GLOBAL` | `global` | Configuration shared across all areas |
| `Magento\Framework\App\Area::AREA_FRONTEND` | `frontend` | Customer-facing storefront |
| `Magento\Framework\App\Area::AREA_ADMINHTML` | `adminhtml` | Admin panel |
| `Magento\Framework\App\Area::AREA_WEBAPI_REST` | `webapi_rest` | RESTful web API |
| `Magento\Framework\App\Area::AREA_WEBAPI_GRAPHQL` | `webapi_graphql` | GraphQL API |
| `Magento\Framework\App\Area::AREA_CRONTAB` | `crontab` | Scheduled cron jobs |

### 3.2 Area Loading Mechanism

When a request enters Magento, the application determines which area it belongs to based on the URL path (for frontend/API) or the request context (for adminhtml). The area loading is orchestrated by `Magento\Framework\App\AreaList` and `Magento\Framework\App\State`:

```php
// Magento\Framework\App\AreaList — determines area code from request
class AreaList
{
    public function getAreaCodes(): array
    {
        // Returns ['frontend', 'adminhtml', 'webapi_rest', ...]
    }

    public function getFrontNameByAreaCode(string $areaCode): string
    {
        // Returns the URL frontName (e.g., 'catalog' for adminhtml/catalog)
    }
}
```

```php
// Magento\Framework\App\State — manages the currently active area
class State
{
    private $areaCode;

    public function setAreaCode(string $areaCode): void
    {
        $this->areaCode = $areaCode;
    }

    public function getAreaCode(): string
    {
        return $this->areaCode;
    }
}
```

The `State` object is a request-scoped singleton that holds the current area code. Once an area is set, `Magento\Framework\App\ObjectManager\ConfigLoader` loads that area's specific `di.xml` and `Magento\Framework\Config\Scope` registers which scope is active (e.g., `frontend`, `adminhtml`, `webapi_rest`).

The full area loading sequence for a frontend request:

1. **Bootstrap** → `Magento\Framework\App\Bootstrap` initializes the `ObjectManager`
2. **Area detection** → `Magento\Framework\App\AreaList::getAreaCode()` inspects the URL and determines `frontend`
3. **Area initialization** → `Magento\Framework\App\Area::loadCode()` is called, which:
   - Loads `app/etc/di.xml` (global DI)
   - Loads `app/etc/events.xml` (global events)
   - Loads area-specific `vendor/Magento/ModuleName/etc/frontend/di.xml`
   - Loads area-specific `vendor/Magento/ModuleName/etc/frontend/events.xml`
   - Loads area-specific `vendor/Magento/ModuleName/etc/frontend/routes.xml`
4. **Request routing** → `Magento\Framework\App\Router` maps the URL to a controller

### 3.3 Area-Specific Configuration Files

Each area can have its own `di.xml`, `events.xml`, and `routes.xml` in an area-specific subdirectory of `etc/`. This allows a module to register different dependency injection configurations depending on which area is active.

```xml
<!-- etc/frontend/di.xml — Frontend-specific DI -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Vendor\Module\Model\ProductLocator">
        <arguments>
            <argument name="cache" xsi:type="string">cache_short</argument>
        </arguments>
    </type>
</config>
```

```xml
<!-- etc/adminhtml/di.xml — Admin-specific DI -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Vendor\Module\Model\ProductLocator">
        <arguments>
            <argument name="cache" xsi:type="string">cache_long</argument>
        </arguments>
    </type>
</type>
</config>
```

In this example, the same `ProductLocator` class receives different cache lifetimes depending on whether it's used in the frontend or adminhtml context. The `ObjectManager` merges area-specific configurations on top of global ones, so adminhtml settings override frontend settings when running in the admin area.

### 3.4 When to Use Area-Specific vs Global Configuration

Use the following decision framework:

- **Global (`etc/`):** Use when the configuration applies to all areas equally. Examples:
  - Plugin declarations
  - Class preferences in `di.xml`
  - Event observers that should fire in every area

- **Area-specific (`etc/adminhtml/`, `etc/frontend/`, etc.):** Use when the behavior must differ by area. Examples:
  - Different cache lifetimes per area
  - Different router classes (adminhtml uses `Magento\Backend\App\Router` while frontend uses `Magento\Framework\App\Router`)
  - Area-specific event observers

- **Events (`events.xml` vs `adminhtml/events.xml`):** Events in `etc/events.xml` fire globally. Events in `etc/adminhtml/events.xml` fire only when the adminhtml area is active. You can define the same event in both files, and observers from both files will be called.

### 3.5 Cross-Area Dependencies

Area-specific configuration files can reference classes from any other area, but the `ObjectManager` respects the current area scope when instantiating dependencies. If you inject a class into a constructor, that class's area-specific DI is applied.

The `global` catch-all scope (`urn:magento:framework:ObjectManager/etc/di.xsd`) applies when no area-specific preference is found. Global configurations in `etc/di.xml` are always loaded regardless of area.

```xml
<!-- etc/di.xml — Global fallback -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Vendor\Module\Model\Service">
        <arguments>
            <argument name="timeout" xsi:type="number">30</argument>
        </arguments>
    </type>
</config>
```

If `etc/frontend/di.xml` does not define a `timeout` argument for `Vendor\Module\Model\Service`, the global value of `30` is used in the frontend area. If `etc/adminhtml/di.xml` defines `timeout` as `60`, the adminhtml area uses `60`.

---

## 4. Module Registration

### 4.1 ComponentRegistrar System

Magento uses `Magento\Framework\Component\ComponentRegistrar` as the central registry for all components (modules, themes, and language packs). This registry is populated at autoload time by all `registration.php` files found in the codebase.

The `ComponentRegistrar` class provides three registration constants:

```php
// In Magento\Framework\Component\ComponentRegistrar
class ComponentRegistrar
{
    public const MODULE   = 'Magento_Framework_ModuleSetup';  // internal value: 'module'
    public const THEME   = 'theme';
    public const LANGUAGE = 'language';

    private static $paths = [];

    public static function register(string $type, string $name, string $path): void
    {
        self::$paths[$type][$name] = $path;
    }

    public static function getPaths(string $type): array
    {
        return self::$paths[$type] ?? [];
    }

    public static function getPath(string $type, string $name): string
    {
        return self::$paths[$type][$name] ?? '';
    }
}
```

The registry is a static, in-memory map. When `ComponentRegistrar::register(ComponentRegistrar::MODULE, 'CompanyName_ModuleName', __DIR__)` is called, the module's filesystem path is stored. This path is later used by the `ObjectManager` to locate the module's configuration files, classes, and view assets.

### 4.2 Registration Patterns for Different Component Types

```php
<?php
/**
 * Copyright © 2025 CompanyName. All rights reserved.
 */

declare(strict_types=1);

use Magento\Framework\Component\ComponentRegistrar;

// Module registration
ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'CompanyName_ModuleName',
    __DIR__
);
```

```php
<?php
/**
 * Copyright © 2025 CompanyName. All rights reserved.
 */

declare(strict_types=1);

use Magento\Framework\Component\ComponentRegistrar;

// Theme registration (frontend theme)
ComponentRegistrar::register(
    ComponentRegistrar::THEME,
    'CompanyName/my-theme',
    __DIR__
);
```

```php
<?php
/**
 * Copyright © 2025 CompanyName. All rights reserved.
 */

declare(strict_types=1);

use Magento\Framework\Component\ComponentRegistrar;

// Language pack registration
ComponentRegistrar::register(
    ComponentRegistrar::LANGUAGE,
    'CompanyName_de_de',
    __DIR__
);
```

### 4.3 Composer Autoload vs ComponentRegistrar

There are two separate mechanisms for discovering Magento components:

**Composer autoload (PSR-4):**
```json
{
    "autoload": {
        "psr-4": {
            "CompanyName\\ModuleName\\": ""
        }
    }
}
```

Composer autoload tells PHP's autoloading system where to find classes by namespace prefix. When any code references `CompanyName\ModuleName\Model\MyModel`, the Composer autoloader knows to look in `Vendor/ModuleName/Model/MyModel.php`.

**ComponentRegistrar registration:**
This tells Magento where the module root directory is located. Without `registration.php` calling `ComponentRegistrar::register()`, Magento will not know about the module as a component—it won't process its `module.xml`, won't merge its `di.xml` files, and won't include it in the module list visible to `bin/magento module:status`.

In practice, for a module published via Composer:
1. The `composer.json` autoload PSR-4 entry tells PHP how to load the module's PHP classes
2. The `registration.php` tells Magento how to discover the module as a Magento component

Both are required. The Composer autoloader handles class loading. The `ComponentRegistrar` handles component discovery for Magento-specific infrastructure (configuration merging, area loading, CLI commands, etc.).

### 4.4 Discovery Order

Magento discovers modules in this sequence:

1. **`ComponentRegistrar` scan** — PHP autoloader invokes all `registration.php` files found in `vendor/` and `app/code/`
2. **`module.xml` parsing** — Each registered module's `etc/module.xml` is parsed to extract the module name and version
3. **Module list building** — `Magento\Framework\App\Bootstrap` builds a list of available modules from the `ComponentRegistrar` registry
4. **Enablement check** — The list is filtered through `config.php` and `env.php` to determine which modules are currently enabled
5. **Sequence resolution** — `<sequence>` tags are used to topologically sort the enabled modules
6. **ObjectManager initialization** — DI configuration is merged in sequence order

The discovery order is NOT alphabetical—it is determined entirely by the dependency graph defined in `<sequence>` tags. If two modules have no declared dependency relationship between them, their relative load order is undefined.

---

## 5. Module Dependencies

### 5.1 The `<sequence>` Tag Explained

As described in Section 1, `<sequence>` declares that the containing module must be loaded *after* the listed modules. The Magento autoloader/ObjectManager system respects this declaration and ensures the sequence is fully respected.

```xml
<!-- app/code/Vendor/ModuleA/etc/module.xml -->
<module name="Vendor_ModuleA" setup_version="1.0.0">
    <sequence>
        <module name="Magento_Catalog"/>
        <module name="Magento_Eav"/>
    </sequence>
</module>
```

```xml
<!-- app/code/Vendor/ModuleB/etc/module.xml -->
<module name="Vendor_ModuleB" setup_version="1.0.0">
    <sequence>
        <module name="Vendor_ModuleA"/>
    </sequence>
</module>
```

In this example, the effective load order is:
1. `Magento_Catalog` and `Magento_Eav` load first (no sequences)
2. `Vendor_ModuleA` loads second (depends on `Magento_Catalog` and `Magento_Eav`)
3. `Vendor_ModuleB` loads third (depends on `Vendor_ModuleA`)

If `Vendor_ModuleB` tried to list only `Vendor_ModuleA` in its sequence but not the transitive dependencies (`Magento_Catalog`, `Magento_Eav`), the system would still work because those transitive dependencies are already loaded (Magento resolves the full graph). However, explicitly listing all direct dependencies is the correct pattern.

### 5.2 Circular Dependency Detection

Circular dependencies cause fatal startup errors in Magento. If `ModuleA` requires `ModuleB`, and `ModuleB` requires `ModuleA`, Magento cannot resolve a topological ordering and throws an exception during `setup:upgrade`.

```php
// Magento\Framework\Module\DependencyChecker — verifies no circular dependencies
class DependencyChecker
{
    public function checkCircularDependencies(array $modules): void
    {
        // Uses Kahn's algorithm for topological sort
        // If the graph has cycles, the sort fails and an exception is thrown
    }
}
```

The error message in a circular dependency scenario typically reads:

```
Circular dependency in modules: Vendor_ModuleA -> Vendor_ModuleB -> Vendor_ModuleA
```

To resolve this, refactor the shared logic into a third module (`Vendor_Shared`) that both `ModuleA` and `ModuleB` can depend on without creating a cycle.

### 5.3 Hard vs Soft Dependencies

**Hard dependencies** (declared via `<sequence>`) mean the dependent module cannot function at all without the dependency. If `Vendor_ModuleB` has `<sequence><module name="Vendor_ModuleA"/></sequence>`, `ModuleB` will fail to load if `ModuleA` is disabled.

**Soft dependencies** (declared via `composer.json` `require`) mean the dependent module *could* function without the dependency, but benefits from it. If `Vendor_ModuleB` requires `Vendor_ModuleA` via Composer but not via `<sequence>`, `ModuleB` will still load and run (perhaps with degraded functionality) if `ModuleA` is disabled.

```json
// composer.json — soft dependency via Composer
{
    "name": "vendor/module-b",
    "require": {
        "vendor/module-a": "^1.0.0"
    }
}
```

```xml
<!-- etc/module.xml — hard dependency via <sequence> -->
<module name="Vendor_ModuleB" setup_version="1.0.0">
    <sequence>
        <module name="Vendor_ModuleA"/>
    </sequence>
</module>
```

**Rule of thumb:** Use `<sequence>` (hard dependency) when the module will throw errors or produce incorrect behavior without the dependency. Use Composer `require` (soft dependency) when the module can gracefully degrade without it.

### 5.4 Dependency Graph and Topological Sort

Magento uses **Kahn's algorithm** (a topological sort) to determine the correct load order from the directed acyclic graph (DAG) formed by `<sequence>` declarations. The algorithm:

1. Build a directed graph where each module is a node and each `<sequence>` edge is a directed edge from the dependent module to the dependency
2. Calculate in-degrees (number of incoming edges) for each node
3. Initialize a queue with all nodes that have in-degree 0
4. Repeatedly dequeue a node, add it to the sorted list, and decrement in-degrees of all neighbors
5. If the result list contains fewer nodes than the total, a cycle exists (circular dependency)

The resulting sorted list is the order in which modules are loaded, their `di.xml` files are merged, and their configuration is registered.

### 5.5 Real-World Dependency Example: Catalog → Eav → Directory

```xml
<!-- Magento_Catalog/etc/module.xml -->
<module name="Magento_Catalog" setup_version="102.0.0">
    <sequence>
        <module name="Magento_Eav"/>
        <module name="Magento_Cms"/>
        <module name="Magento_Store"/>
    </sequence>
</module>
```

```xml
<!-- Magento_Eav/etc/module.xml -->
<module name="Magento_Eav" setup_version="102.0.0">
    <sequence>
        <module name="Magento_Directory"/>
        <module name="Magento_Eav"/>
        <module name="Magento_Store"/>
    </sequence>
</module>
```

This chain—`Magento_Catalog` → `Magento_Eav` → `Magento_Directory`—exists because the catalog's entity attribute value (EAV) system depends on the directory region/country data. Any module that extends catalog entities effectively depends on the entire chain.

---

## 6. Component Relationships

### 6.1 Service Contracts

Service contracts are the recommended way for modules to communicate with each other. They define formal interfaces in the `Api/` directory that guarantee a stable, versioned API. Other modules should depend on these interfaces, not on concrete implementations.

```php
<?php
// Magento\Catalog\Api\Data\ProductInterface.php
namespace Magento\Catalog\Api\Data;

use Magento\Framework\Api\ExtensibleDataInterface;

interface ProductInterface extends ExtensibleDataInterface
{
    public function getId(): int;
    public function getName(): string;
    public function setName(string $name): void;
    public function getPrice(): float;
    public function setPrice(float $price): void;
    // ... extensive list of product data accessors
}
```

```php
<?php
// Magento\Catalog\Api\ProductRepositoryInterface.php
namespace Magento\Catalog\Api;

interface ProductRepositoryInterface
{
    public function get(string $sku, bool $editMode = false, ?int $storeId = null): \Magento\Catalog\Api\Data\ProductInterface;
    public function save(\Magento\Catalog\Api\Data\ProductInterface $product): \Magento\Catalog\Api\Data\ProductInterface;
    public function delete(\Magento\Catalog\Api\Data\ProductInterface $product): bool;
    public function deleteById(string $sku): bool;
    public function getList(\Magento\Framework\Api\SearchCriteriaInterface $searchCriteria): \Magento\Catalog\Api\Data\ProductSearchResultsInterface;
}
```

### 6.2 Repository Pattern

The repository pattern separates the data access layer from the business logic layer. Concrete repository classes implement their corresponding `*RepositoryInterface` and are configured via `di.xml` as the preferred implementation:

```xml
<!-- Magento_Catalog/etc/di.xml -->
<preference for="Magento\Catalog\Api\ProductRepositoryInterface"
            type="Magento\Catalog\Model\ProductRepository"/>
```

```php
<?php
// Magento\Catalog\Model\ProductRepository
namespace Magento\Catalog\Model;

class ProductRepository implements \Magento\Catalog\Api\ProductRepositoryInterface
{
    public function __construct(
        private readonly \Magento\Catalog\Model\ProductFactory $productFactory,
        private readonly \Magento\Catalog\Model\ResourceModel\Product $resourceModel,
        private readonly \Magento\Catalog\Api\Data\ProductSearchResultsInterfaceFactory $searchResultsFactory
    ) {}

    public function get(string $sku, bool $editMode = false, ?int $storeId = null): \Magento\Catalog\Api\Data\ProductInterface
    {
        // Load and return product by SKU
    }
}
```

### 6.3 di.xml Preferences

`di.xml` preferences allow a module to substitute a different implementation for an interface or class without changing any consumer code:

```xml
<!-- etc/di.xml — Substitute a custom implementation -->
<preference for="Magento\Catalog\Api\ProductRepositoryInterface"
            type="CompanyName\ModuleName\Model\CustomProductRepository"/>
```

This means every `ProductRepositoryInterface` injection throughout the system receives `CompanyName\ModuleName\Model\CustomProductRepository` instead of `Magento\Catalog\Model\ProductRepository`. This is a powerful extensibility mechanism that enables:
- Third-party module replacements for core functionality
- Stubs and mocks for testing
- Feature flag-based implementation switching

### 6.4 Plugin Cross-Module Discovery

Plugins (interceptors) declared in `etc/di.xml` are applied across module boundaries:

```xml
<!-- CompanyName/ModuleName/etc/di.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Catalog\Model\ProductRepository">
        <plugin name="companyModuleProductRepositoryPlugin"
                type="CompanyName\ModuleName\Plugin\ProductRepositoryPlugin"/>
    </type>
</config>
```

```php
<?php
// CompanyName/ModuleName/Plugin/ProductRepositoryPlugin.php
namespace CompanyName\ModuleName\Plugin;

class ProductRepositoryPlugin
{
    public function beforeSave(
        \Magento\Catalog\Model\ProductRepository $subject,
        \Magento\Catalog\Api\Data\ProductInterface $product
    ): array {
        // Modify the product data before it is saved
        // Can call $subject->save() and return its result
        return [$product];
    }

    public function afterGetList(
        \Magento\Catalog\Model\ProductRepository $subject,
        \Magento\Catalog\Api\Data\ProductSearchResultsInterface $result,
        \Magento\Framework\Api\SearchCriteriaInterface $searchCriteria
    ): \Magento\Catalog\Api\Data\ProductSearchResultsInterface {
        // Modify the search results after retrieval
        return $result;
    }
}
```

### 6.5 Shared Modules vs Non-Shared

Some modules are designated as "shared" (also called "sequence-free" or "free") because they do not enforce a specific load order and can be loaded in parallel. The `Magento_Framework` module and `Magento_PatternLibrary` are examples.

Non-shared modules have strict ordering requirements. The `<sequence>` tags ensure they load after their dependencies. Shared modules are loaded first in the initialization phase, followed by non-shared modules in their dependency order.

---

## 7. Module Discovery

### 7.1 The Discovery Flow

The complete module discovery flow in Magento 2.4.8:

```
1. Composer Autoloader Initialization
   └─> PSR-4 autoload registers all namespace-to-path mappings
   └─> registration.php files are NOT yet executed

2. Magento\Framework\Component\ComponentRegistrar initialization
   └─> Magento\Framework\App\Bootstrap::bootstrapComponentRegistrar()
   └─> ComponentRegistrar::register() called for each found registration.php
   └─> Result: static $paths[ComponentRegistrar::MODULE] populated

3. Module list construction
   └─> Magento\Framework\App\Bootstrap::createObjectManager()
   └─> ModuleList::getNames() returns list of registered module names
   └─> Enablement filter applied (config.php + env.php)

4. Dependency ordering
   └─> Magento\Framework\Module\DependencyChecker::sortDependencies()
   └─> Topological sort resolves <sequence> into load order

5. ObjectManager configuration loading
   └─> For each module in load order:
       ├─> Load etc/module.xml (register module with ModuleManager)
       ├─> Merge etc/di.xml into global ObjectManager config
       ├─> Merge area-specific etc/[area]/di.xml (when area is active)
       └─> Register plugin declarations

6. Area-specific loading (request-scoped)
   └─> Magento\Framework\App\AreaList::getAreaCode() determines area
   └─> Magento\Framework\App\Area::loadArea() loads area-specific config
   └─> Route maps built from etc/routes.xml
```

### 7.2 Composer Autoload vs ComponentRegistrar

The critical distinction:

- **Composer autoload PSR-4** tells PHP's SPL autoloader how to find a class file given its namespace. It handles PHP class loading only. Composer discovers `registration.php` because it is listed in `composer.json` `autoload` > `files` (not PSR-4).

- **ComponentRegistrar** tells Magento's infrastructure (ObjectManager, configuration loaders, area bootloaders) where to find module resources. It handles non-PHP assets like XML configuration, view templates, and JavaScript.

```json
{
    "autoload": {
        "psr-4": {
            "CompanyName\\ModuleName\\": ""
        },
        "files": [
            "registration.php"
        ]
    }
}
```

The `files` key in Composer autoload ensures `registration.php` is included on every request (not just when a class from the module is referenced).

### 7.3 module.xml Discovery Order

The order in which `module.xml` files are processed is determined by:
1. The module's position in the sorted dependency graph
2. Whether the module is in `app/code/` (community modules) or `vendor/` (Composer-installed modules)

There is no guaranteed order between two modules that have no declared relationship. However, in practice, modules in `app/code/` are typically loaded before modules in `vendor/` due to how the autoloader resolves paths, though this should not be relied upon—always use `<sequence>` to express ordering requirements.

---

## 8. Enabling and Disabling Modules

### 8.1 CLI Commands

```bash
# Enable a module
bin/magento module:enable CompanyName_ModuleName

# Disable a module
bin/magento module:disable CompanyName_ModuleName

# List all modules and their status
bin/magento module:status

# Uninstall a module (removes from Composer and database)
bin/magento module:uninstall CompanyName_ModuleName
```

### 8.2 What `module:enable` Does

When `bin/magento module:enable Vendor_Module` is executed:

1. Reads `app/etc/config.php` and `app/etc/env.php`
2. Sets `Vendor_Module => 1` in the appropriate file (`config.php` by default, `env.php` if `--env` flag is used)
3. Runs `setup:upgrade` to register the module's version in the `setup_module` table
4. Regenerates the Magento cache (`cache:sclean`)

The enable flag in `config.php` means the module is **enabled by default for all environments**. If the same module is set to `0` in `env.php`, the environment-specific setting takes precedence.

### 8.3 What `module:disable` Does

When `bin/magento module:disable Vendor_Module` is executed:

1. Reads `app/etc/env.php`
2. Sets `Vendor_Module => 0` in `env.php` (environment-specific, not committed to version control)
3. Flushes all caches

Disabling a module via `module:disable` does NOT remove the module from `config.php`. It adds an override in `env.php`. This ensures the module remains disabled even if `config.php` is re-committed with the module enabled.

### 8.4 Safe Disable Order

Before disabling any module in production:

1. **Put the store in maintenance mode** (optional but recommended):
   ```bash
   bin/magento maintenance:enable
   ```

2. **Flush the cache**:
   ```bash
   bin/magento cache:flush
   ```

3. **Set the application mode to production** (to ensure generated factories and proxies are valid):
   ```bash
   bin/magento deploy:mode:set production
   ```

4. **Disable the module**:
   ```bash
   bin/magento module:disable Vendor_Module
   ```

5. **Re-run setup:upgrade** to update schema versions:
   ```bash
   bin/magento setup:upgrade
   ```

6. **Regenerate static content** if the module has view files:
   ```bash
   bin/magento setup:static-content:deploy
   ```

7. **Disable maintenance mode**:
   ```bash
   bin/magento maintenance:disable
   ```

Failure to flush cache after a disable can cause "class not found" errors because the ObjectManager's cached configuration still references the disabled module's classes.

### 8.5 What `module:uninstall` Does

```bash
bin/magento module:uninstall CompanyName_ModuleName
```

The uninstall process:
1. **Verifies no other enabled module depends on it** via `<sequence>` checking
2. **Calls the component uninstaller** registered in `composer.json`:
   ```json
   {
       "scripts": {
           "Component uninstall.php": [
               "Magento\\Framework\\Composer\\ComponentUninstall::uninstall"
           ]
       }
   }
   ```
3. **Removes the module from Composer** (`composer remove vendor/module-name`)
4. **Removes the module's files** from the filesystem
5. **Updates `app/etc/config.php`** to remove the module from the list

```bash
# If the module has database tables created by InstallSchema,
# those tables must be manually dropped or the uninstall will fail
bin/magento module:uninstall CompanyName_ModuleName --remove-data
```

---

## 9. Module Versioning and Schema

### 9.1 `setup_version` vs `composer.json` Version

Magento uses two separate version numbers for each module:

| Version | Location | Purpose | When to Increment |
|---------|----------|---------|-------------------|
| `setup_version` | `etc/module.xml` | Database schema version. Triggers `UpgradeSchema` and `UpgradeData` scripts | When database schema or data changes |
| `composer.json` version | `composer.json` | Package version for Composer. Used for `composer require` dependency resolution | Following semantic versioning for the package release |

```xml
<!-- etc/module.xml -->
<module name="CompanyName_ModuleName" setup_version="1.0.1">
```

```json
<!-- composer.json -->
{
    "name": "companyname/module-name",
    "version": "1.0.1",
    "description": "Company Module for Magento 2",
    "require": {
        "magento/framework": "103.0.*"
    }
}
```

### 9.2 Composer Version Requirements

`composer.json` version requirements must follow semantic versioning and use Composer version constraints:

| Constraint | Meaning | Example |
|------------|---------|---------|
| `1.0.0` | Exact version | Only `1.0.0` |
| `^1.0.0` | Min version, less than next major | `>=1.0.0 <2.0.0` |
| `~1.0.0` | Minor version flexibility | `>=1.0.0 <1.1.0` |
| `1.0.*` | Wildcard | `>=1.0.0 <1.1.0` |
| `2.0.0-beta` | Pre-release | Beta release |

```json
{
    "require": {
        "magento/framework": "103.0.*",
        "php": "^8.1"
    }
}
```

### 9.3 When Upgrade Scripts Run

| Stored Version | Declared `setup_version` | Scripts Run |
|---------------|-------------------------|-------------|
| `null` (not installed) | `1.0.0` | `InstallSchema`, `InstallData` |
| `1.0.0` | `1.0.1` | `UpgradeSchema`, `UpgradeData` |
| `1.0.0` | `1.0.0` | Nothing (already at current version) |
| `1.0.0` | `1.1.0` | `UpgradeSchema`, `UpgradeData` (minor bump also triggers upgrade) |
| Any | `2.0.0` | `UpgradeSchema`, `UpgradeData` (major version change is still an upgrade) |

Magento compares the **stored version** in `setup_module` table against the **declared `setup_version`** in `module.xml`. If the declared version is higher, the upgrade scripts run.

The `Recurring.php` script runs after **every** `setup:upgrade`, regardless of whether any upgrade was needed. This is useful for database operations that need to run on every deployment (e.g., updating search index structures that are not managed by specific schema versions).

### 9.4 module.xml Schema Validation

The official XSD for `module.xml` is:

```
urn:magento:framework:Module/etc/module.xsd
```

This schema is provided by `Magento_FrameworkModule` and defines:

- The `module` element as the root
- The `name` attribute (required, string, the module name)
- The `setup_version` attribute (required, string, semantic version)
- The `sequence` element as an optional container
- The `module` child element inside `sequence` (zero or more, each naming a dependency)

```xml
<?xml version="1.0"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           targetNamespace="urn:magento:framework:Module/etc/module.xsd">
    <xs:element name="config">
        <xs:complexType>
            <xs:sequence>
                <xs:element name="module" minOccurs="1" maxOccurs="unbounded">
                    <xs:complexType>
                        <xs:sequence>
                            <xs:element name="sequence" minOccurs="0" maxOccurs="1">
                                <xs:complexType>
                                    <xs:sequence>
                                        <xs:element name="module" minOccurs="0" maxOccurs="unbounded">
                                            <xs:complexType>
                                                <xs:attribute name="name" type="xs:string" use="required"/>
                                            </xs:complexType>
                                        </xs:element>
                                    </xs:sequence>
                                </xs:complexType>
                            </xs:element>
                        </xs:sequence>
                        <xs:attribute name="name" type="xs:string" use="required"/>
                        <xs:attribute name="setup_version" type="xs:string" use="required"/>
                    </xs:complexType>
                </xs:element>
            </xs:sequence>
        </xs:complexType>
    </xs:element>
</xs:schema>
```

Magento validates the `module.xml` against this XSD during `setup:upgrade`. Invalid XML causes a bootstrap failure with an error like:

```
Element 'module': Missing attribute 'name'.
Element 'module': Missing attribute 'setup_version'.
```

### 9.5 Schema Version Compatibility

Magento 2.4.8 requires `setup_version` to be a valid semantic version string. The format is `MAJOR.MINOR.PATCH` (e.g., `1.0.0`, `2.1.3`). Pre-release versions like `1.0.0-alpha` or `2.0.0-beta` are valid in `composer.json` but the `module.xml` `setup_version` typically uses stable versions in production modules.

---

## Further Reading

- [Magento 2.4.8 Official Documentation](https://experienceleague.adobe.com/docs/commerce-operations/release/notes/magento-open-source/2-4-8.html)
- [Module Development Guide](https://developer.adobe.com/commerce/php/development/)
- [Magento Coding Standards](https://developer.adobe.com/commerce/php/coding-standards/)
- [Dependency Injection Configuration](https://developer.adobe.com/commerce/php/development/configuration/)
- [Component Registrar](https://developer.adobe.com/commerce/php/development/components/registrar/)
- [Module File Structure](https://developer.adobe.com/commerce/php/development/configuration/create/)
- [Service Contracts](https://developer.adobe.com/commerce/php/development/configuration/service-contracts/)
- [Area Code Loading](https://developer.adobe.com/commerce/php/development/configuration/areas/)
- [Module Deployment](https://experienceleague.adobe.com/docs/commerce-operations/configuration-guide/deploy/module-deployment.html)
- [Magento Security Best Practices](https://developer.adobe.com/commerce/php/best-practices/security/)