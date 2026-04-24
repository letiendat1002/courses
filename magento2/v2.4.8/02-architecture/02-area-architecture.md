---
title: "02 - Area Architecture"
description: "Deep dive into Magento 2.4.8 Area architecture — the five independent DI + layout + template scopes (frontend, adminhtml, graphql, webapi_rest, webapi_soap), how area loading affects ObjectManager configuration, route resolution by area, and the stateful nature of areas across the request lifecycle."
tags: [magento2, architecture, area, di-xml, routing, layout, state, front-controller, area-list, area-resolver]
rank: 4
pathways: [magento2-deep-dive]
see_also:
  - path: "02-architecture/02-request-flow.md"
    description: "Request lifecycle — covers AreaList, front controller, router matching, and how areas integrate with the overall request pipeline"
---

# Area Architecture

Every incoming HTTP request to a Magento 2.4.8 application passes through one of five configured Areas: `frontend`, `adminhtml`, `graphql`, `webapi_rest`, or `webapi_soap`. The Area system is one of Magento's most critical — and least explained — architectural concepts. It determines which dependency injection configuration loads, which route table resolves, which layout files compile, and which templates render. Misunderstanding Areas leads to bugs that are difficult to diagnose: plugins not firing, routes not matching, layouts missing blocks.

This chapter explains the what, why, and how of Areas at the level of detail needed to work confidently with Magento's internals.

---

## 1. What Is an Area

An **Area** in Magento 2 is an independently configured execution context that scopes three things:

- **Dependency Injection (DI)** — which `di.xml` files are loaded into the `ObjectManager`
- **Routing** — which `routes.xml` defines the frontName-to-module mapping
- **Layout** — which layout XML files compile into the page structure and which templates are used

Each Area is a horizontal slice across all modules. A module can and often does contribute to multiple Areas. For example, `Magento_Catalog` defines:
- `etc/frontend/routes.xml` for frontend product pages
- `etc/adminhtml/routes.xml` for admin catalog management
- `etc/webapi_rest/di.xml` for catalog API endpoints
- `etc/graphql/di.xml` for GraphQL schema contributions

### The Five Areas

| Area Code | Purpose | Front Name | Entry Point | Key Config Files |
|-----------|---------|------------|-------------|------------------|
| `frontend` | Customer-facing storefront | Dynamic (from routes.xml) | `pub/index.php` | `etc/frontend/{di.xml, routes.xml, layout/*.xml}` |
| `adminhtml` | Admin panel | Fixed: `admin` | `pub/index.php` | `etc/adminhtml/{di.xml, routes.xml, menu.xml, acl.xml}` |
| `graphql` | GraphQL API endpoint | `/graphql` | `pub/index.php` | `etc/graphql/di.xml`, `etc/graphql/schema.graphqls` |
| `webapi_rest` | REST API | `/rest` | `pub/index.php` | `etc/webapi_rest/di.xml`, `etc/webapi.xml` |
| `webapi_soap` | SOAP API | `/soap` | `pub/index.php` | `etc/webapi_soap/di.xml`, `etc/webapi.xml` |
| `crontab` | Scheduled CLI tasks | N/A | `bin/magento` | `etc/crontab/di.xml`, `etc/crontab/events.xml` |

> **Note:** The `crontab` Area handles scheduled tasks executed via `bin/magento cron:run`. It has no HTTP front name but still loads its own `di.xml` and responds to events registered in `etc/crontab/events.xml`.

### Area vs Module

The most important mental shift when understanding Areas is this:

```
Module = VERTICAL slice (one feature across all concerns)
Area   = HORIZONTAL slice (one concern across all modules)
```

Consider `Magento_Customer`:
- In `frontend`, it contributes customer account pages, login, registration
- In `adminhtml`, it contributes customer management in the admin panel
- In `graphql`, it contributes the `Customer` type and queries
- In `webapi_rest`, it exposes `/V1/customers/` endpoints

Every module you install contributes code to every Area it has configuration for. When you run `bin/magento setup:di:compile`, Magento iterates over all modules and merges their per-Area DI configurations into a unified object graph for each Area.

---

## 2. Why Areas Exist

The Area system was designed to solve three distinct problems.

### 2.1 Performance — Load Only What Is Needed

When a request comes in for a frontend product page, Magento has no business loading admin-specific DI configuration, admin routes, admin layout files, or admin template files. The `ObjectManager` is configured per-area: the frontend `ObjectManager` only knows about classes and plugins declared in modules' `etc/frontend/di.xml` files. Admin requests never trigger frontend DI configuration.

This separation means:
- Frontend requests initialize faster (less configuration to parse)
- Admin requests initialize faster (no frontend configuration polluting the class graph)
- Memory footprint per request is lower (Area-specific singletons are not loaded for irrelevant contexts)

### 2.2 Security — Admin Routes Isolated from Public Frontend

The `adminhtml` Area is locked down by design. Routes declared in `etc/adminhtml/routes.xml` have a frontName of `admin` by convention, and the `Magento_Backend` module's front controller enforces admin session validation before any admin controller executes. A request to `/admin/catalog/product/index` will fail with a 302 redirect to the login page if no admin session exists.

Without Area isolation, you'd have to manually ensure every admin route had ACL checks. The Area system bakes authentication enforcement into the routing infrastructure for adminhtml.

### 2.3 Separation of Concerns — Different Rendering Contexts

Frontend pages need full page layout with header, footer, sidebar, and a theme. GraphQL requests bypass layout entirely — there is no HTML rendering, no template files, no block structure. REST API requests return JSON and also bypass the layout system. Each Area's rendering pipeline is appropriate for its use case.

| Area | Layout System Used | Template Context |
|------|-------------------|------------------|
| `frontend` | Full layout.xml with handles | Theme templates |
| `adminhtml` | Admin layout handles | Admin theme templates |
| `graphql` | No layout (API only) | N/A |
| `webapi_rest` | No layout (API only) | N/A |
| `webapi_soap` | No layout (API only) | N/A |

---

## 3. Area Configuration Files

Each Area has a directory inside a module's `etc/` folder. The files inside that directory apply only to that Area.

```
app/code/Vendor/Module/
├── etc/
│   ├── module.xml                    # Global — module declaration
│   ├── di.xml                        # Global — applies to ALL areas
│   ├── events.xml                    # Global — applies to ALL areas
│   ├── frontend/
│   │   ├── di.xml                    # Frontend-only DI
│   │   ├── routes.xml                # Frontend-only routes
│   │   ├── layout/*.xml             # Frontend layout files
│   │   └── events.xml               # Frontend event observers
│   ├── adminhtml/
│   │   ├── di.xml                    # Adminhtml-only DI
│   │   ├── routes.xml                # Adminhtml routes
│   │   ├── menu.xml                 # Admin menu items
│   │   ├── acl.xml                  # Access control lists
│   │   └── events.xml              # Admin event observers
│   ├── graphql/
│   │   ├── di.xml                    # GraphQL-only DI
│   │   └── schema.graphqls           # GraphQL schema
│   ├── webapi_rest/
│   │   └── di.xml                    # REST API DI
│   ├── webapi_soap/
│   │   └── di.xml                    # SOAP API DI
│   └── crontab/
│       ├── di.xml                    # Cron-only DI
│       └── events.xml               # Cron event observers
```

### 3.1 `etc/{area}/di.xml` — Per-Area Dependency Injection

**File**: `vendor/magento/module-catalog/etc/frontend/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Plugin only active in frontend area -->
    <type name="Magento\Catalog\Model\Product">
        <plugin name="CatalogProductPlugin"
                type="Magento\Catalog\Plugin\ProductPlugin"
                sortOrder="10"/>
    </type>

    <!-- Preference for frontend-specific implementation -->
    <preference for="Magento\Catalog\Api\ProductRepositoryInterface"
                type="Magento\Catalog\Model\ProductRepository"/>

</config>
```

**File**: `vendor/magento/module-catalog/etc/adminhtml/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Plugin only active in adminhtml area -->
    <type name="Magento\Catalog\Model\Product">
        <plugin name="CatalogAdminProductPlugin"
                type="Magento\Catalog\Plugin\AdminProductPlugin"
                sortOrder="10"/>
    </type>

    <!-- Admin-specific preference -->
    <preference for="Magento\Catalog\Api\ProductRepositoryInterface"
                type="Magento\Catalog\Model\AdminProductRepository"/>

</config>
```

The same class (`Magento\Catalog\Model\Product`) has **different plugins** depending on which Area is active. The `CatalogProductPlugin` runs for frontend requests; `CatalogAdminProductPlugin` runs for admin requests.

### 3.2 `etc/{area}/routes.xml` — Route Configuration Per Area

Routes define the mapping from URL segments (frontName) to module controllers. Routes are scoped to Areas.

**Frontend routes** (`etc/frontend/routes.xml`):

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:App/etc/routes.xsd">
    <router id="standard">
        <route id="catalog" frontName="catalog">
            <module name="Magento_Catalog" />
        </route>
        <route id="checkout" frontName="checkout">
            <module name="Magento_Checkout" />
        </route>
        <route id="customer" frontName="customer">
            <module name="Magento_Customer" />
        </route>
    </router>
</config>
```

**Adminhtml routes** (`etc/adminhtml/routes.xml`):

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:App/etc/routes.xsd">
    <router id="admin">
        <route id="admin" frontName="admin">
            <module name="Magento_Backend" />
            <!-- Modules register with `before="Magento_Backend"` to insert themselves -->
        </route>
    </router>
</config>
```

**Route frontnames:**

| Area | Route ID | Front Name | Full URL Example |
|------|----------|------------|------------------|
| `frontend` | `catalog` | `catalog` | `/catalog/product/view/id/123` |
| `frontend` | `checkout` | `checkout` | `/checkout/index/index` |
| `adminhtml` | `admin` | `admin` | `/admin/catalog/product/index` |

### 3.3 `etc/{area}/layout/` — Area-Specific Layout Files

Layout XML files are also area-scoped. When Magento builds the page layout, it only loads layout files from the active Area's `layout/` directories.

**Frontend layout** (`view/frontend/layout/catalog_product_view.xml`):

```xml
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceContainer name="content">
            <block name="product.info"
                   class="Magento\Catalog\Block\Product\View"
                   template="Magento_Catalog::product/view.phtml"
                   cacheable="true"/>
        </referenceContainer>
    </body>
</config>
```

**Adminhtml layout** (`view/adminhtml/layout/catalog_product_edit.xml`):

```xml
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceContainer name="main">
            <block name="product_edit_form"
                   class="Magento\Catalog\Block\Adminhtml\Product\Edit"
                   template="Magento_Catalog::product/edit.phtml"/>
        </referenceContainer>
    </body>
</config>
```

The same logical page (product edit) has a completely different block structure, template, and HTML output in adminhtml versus frontend.

### 3.4 `etc/webapi.xml` and `etc/{area}/webapi.xml` — Web API Endpoint Registration

The web API system uses `webapi.xml` to register service methods as REST or SOAP endpoints. For REST, it's `etc/webapi_rest/di.xml` (for DI) and `etc/webapi.xml` (for the endpoint mapping). For SOAP, it is `etc/webapi_soap/di.xml` and `etc/webapi.xml`.

**File**: `vendor/magento/module-catalog/etc/webapi.xml`

```xml
<?xml version="1.0"?>
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Api/etc/webapi.xsd">
    <route method="GET" url="/V1/products/:id">
        <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="getById"/>
        <resources>
            <resource ref="Magento_Catalog::products"/>
        </resources>
    </route>
</routes>
```

This endpoint is registered in the global `webapi.xml`. The actual service class resolution happens in `di.xml` files scoped to `webapi_rest` or `webapi_soap`.

---

## 4. AreaResolver and Area Detection

### 4.1 `Magento\Framework\App\AreaList`

**File**: `vendor/magento/framework/App/AreaList.php`

```php
<?php
declare(strict_types=1);

namespace Magento\Framework\App;

class AreaList implements AreaListInterface
{
    /**
     * @var array
     */
    protected $_areas;

    /**
     * @var AreaInterface[]
     */
    protected $_areaInstances = [];

    /**
     * @var string
     */
    protected $_defaultAreaCode = 'frontend';

    /**
     * @param array $areas
     * @param FrontNameResolverFactory $resolverFactory
     * @param ObjectManagerInterface $objectManager
     */
    public function __construct(
        array $areas,
        FrontNameResolverFactory $resolverFactory,
        ObjectManagerInterface $objectManager
    ) {
        $this->_areas = $areas;
        $this->_resolverFactory = $resolverFactory;
        $this->_objectManager = $objectManager;
    }

    /**
     * Get area code by front name
     *
     * @param string $frontName
     * @return string|null
     */
    public function getCodeByFrontName(string $frontName): ?string
    {
        foreach ($this->_areas as $areaCode => $areaInfo) {
            if (isset($areaInfo['frontName']) && $areaInfo['frontName'] === $frontName) {
                return $areaCode;
            }
        }
        return null;
    }

    /**
     * Get area object by code
     *
     * @param string $code
     * @return Area
     */
    public function getArea(string $code): Area
    {
        if (!isset($this->_areaInstances[$code])) {
            $area = $this->_objectManager->get(Area::class);
            $area->load($code);
            $this->_areaInstances[$code] = $area;
        }
        return $this->_areaInstances[$code];
    }
}
```

The `AreaList` is configured in `app/etc/di.xml`:

```xml
<type name="Magento\Framework\App\AreaList">
    <arguments>
        <argument name="areas" xsi:type="array">
            <item name="frontend" xsi:type="array">
                <item name="frontName" xsi:type="string">cms</item>
            </item>
            <item name="adminhtml" xsi:type="array">
                <item name="frontName" xsi:type="string">admin</item>
            </item>
            <item name="graphql" xsi:type="array">
                <item name="frontName" xsi:type="string">graphql</item>
            </item>
            <item name="crontab" xsi:type="array">
                <item name="frontName" xsi:type="string">cron</item>
            </item>
        </argument>
    </arguments>
</type>
```

### 4.2 `Magento\Framework\App\Area`

**File**: `vendor/magento/framework/App/Area.php`

```php
<?php
declare(strict_types=1);

namespace Magento\Framework\App;

class Area implements AreaInterface
{
    const PART_CONFIG = 'config';
    const PART_DESIGN = 'design';
    const PART_TRANSLATE = 'translate';
    const PART_CONTROLLER = 'controller';

    /**
     * @var string
     */
    protected $_code;

    /**
     * @var array
     */
    protected $_loadedParts = [];

    /**
     * @var ConfigInterface
     */
    protected $_config;

    /**
     * @var ObjectManagerInterface
     */
    protected $_objectManager;

    /**
     * @param ConfigInterface $config
     * @param ObjectManagerInterface $objectManager
     * @param AreaList $areaList
     */
    public function __construct(
        ConfigInterface $config,
        ObjectManagerInterface $objectManager,
        AreaList $areaList
    ) {
        $this->_config = $config;
        $this->_objectManager = $objectManager;
        $this->_areaList = $areaList;
    }

    /**
     * Load area configuration
     *
     * @param string $areaCode
     * @return void
     */
    public function load(string $areaCode): void
    {
        $this->_code = $areaCode;

        // This is where area-specific di.xml gets loaded into ObjectManager
        $this->_objectManager->configure(
            $this->_config->get(self::PART_CONFIG, $areaCode)
        );
    }

    /**
     * Detect and set area code from request
     *
     * @param RequestInterface $request
     * @return string
     */
    public function detect(RequestInterface $request): string
    {
        $frontName = $request->getFrontName();
        $areaCode = $this->_areaList->getCodeByFrontName($frontName)
            ?? $this->_areaList->getDefaultAreaCode();
        $this->load($areaCode);
        return $areaCode;
    }
}
```

The critical line in `Area::load()` is:

```php
$this->_objectManager->configure(
    $this->_config->get(self::PART_CONFIG, $areaCode)
);
```

This call merges all `etc/{area}/di.xml` files from all modules into the `ObjectManager`. Once configured, the `ObjectManager` has the complete DI graph for that Area — including plugins, preferences, virtual types, and argument overrides specific to that Area.

### 4.3 Area Constants

**File**: `vendor/magento/framework/App/Area.php`

```php
<?php

namespace Magento\Framework\App;

class Area
{
    const AREA_FRONTEND = 'frontend';
    const AREA_ADMINHTML = 'adminhtml';
    const AREA_ADMIN = 'adminhtml';  // Alias
    const AREA_GRAPHQL = 'graphql';
    const AREA_WEBAPI_REST = 'webapi_rest';
    const AREA_WEBAPI_SOAP = 'webapi_soap';
    const AREA_CRONTAB = 'crontab';
    const AREA_GLOBAL = 'global';
}
```

These constants are used throughout the codebase to reference areas programmatically:

```php
use Magento\Framework\App\Area;

$area = $areaList->getArea(Area::AREA_FRONTEND);
$area->load(Area::AREA_FRONTEND);
```

---

## 5. Request Flow by Area

### 5.1 `Http::launch()` — The Application Bootstrap

**File**: `vendor/magento/framework/App/Http.php`

```php
<?php
declare(strict_types=1);

namespace Magento\Framework\App;

class Http implements \Magento\Framework\AppInterface
{
    /**
     * @var ObjectManagerInterface
     */
    protected $_objectManager;

    /**
     * @var AreaList
     */
    protected $_areaList;

    /**
     * @var FrontControllerInterface
     */
    protected $_frontController;

    /**
     * @var State
     */
    protected $_appState;

    public function __construct(
        ObjectManagerInterface $objectManager,
        AreaList $areaList,
        FrontControllerInterface $frontController,
        State $appState
    ) {
        $this->_objectManager = $objectManager;
        $this->_areaList = $areaList;
        $this->_frontController = $frontController;
        $this->_appState = $appState;
    }

    public function launch(): void
    {
        // Step 1: Detect area from the request's front name
        $areaCode = $this->_areaList->getArea(
            $this->_areaList->getCodeByFrontName(
                $this->_request->getFrontName()
            ) ?: Area::AREA_FRONTEND
        );

        // Step 2: Set the area code in app state (this is STATEFUL)
        $this->_appState->setAreaCode($areaCode);

        // Step 3: Load area configuration (this configures ObjectManager)
        $area = $this->_areaList->getArea($areaCode);
        $area->load($areaCode);

        // Step 4: Dispatch through front controller (routing + controller)
        $this->_frontController->dispatch($this->_request);

        return $this->_response;
    }
}
```

### 5.2 How FrontController Routing Differs by Area

The `FrontController` dispatches to routers. Routers are registered in `di.xml` and differ per Area.

**Frontend routing** — RouterList in `frontend` includes:
- `Magento\Cms\Controller\Router` (sortOrder 10) — CMS page matching
- `Magento\UrlRewrite\Controller\Router` (sortOrder 20) — URL rewrite table
- `Magento\Framework\App\Router\Base` (sortOrder 30) — Standard frontName/action matching

**Adminhtml routing** — RouterList in `adminhtml` includes:
- `Magento\Backend\Controller\Router` — Admin URL matching with session validation
- `Magento\Framework\App\Router\Base` — Standard routing as fallback

**GraphQL routing** — Uses `AreaManager` rather than `FrontController`:

**File**: `vendor/magento/module-graph-ql/Controller/GraphQl.php`

```php
<?php
declare(strict_types=1);

namespace Magento\GraphQl\Controller;

use Magento\Framework\App\Area\AreaInterface;
use Magento\Framework\App\AreaList;
use Magento\Framework\App\State;

class GraphQl implements \Magento\Framework\AppInterface
{
    public function __construct(
        private readonly AreaList $areaList,
        private readonly State $appState,
        private readonly \Magento\Framework\App\RequestInterface $request
    ) {
    }

    public function launch(): void
    {
        // GraphQL uses AreaManager to load the graphql area
        $area = $this->areaList->getArea(Area::AREA_GRAPHQL);
        $area->load(Area::AREA_GRAPHQL);

        // Then processes the GraphQL request through the schema resolver
        // No front controller, no router list
    }
}
```

### 5.3 Route Resolution Sequence

```
Request arrives: GET /catalog/product/view/id/123

1. FrontName extraction: "catalog" becomes the frontName
2. Area detection: AreaList::getCodeByFrontName("catalog") → "frontend"
3. Area loading: Area::load("frontend") merges all etc/frontend/di.xml
4. Route matching: StandardRouter matches "catalog" → "Magento_Catalog"
5. Controller resolution: "product/view" → Product\View controller
6. Action execution: execute() returns ResultInterface
7. Response rendering: ResultInterface::renderResult()
```

For admin requests (`/admin/catalog/product/index`):

```
1. FrontName extraction: "admin"
2. Area detection: AreaList::getCodeByFrontName("admin") → "adminhtml"
3. Area loading: Area::load("adminhtml") merges all etc/adminhtml/di.xml
4. Route matching: Backend router validates admin session first
5. Controller resolution: "catalog/product" → Catalog\Product\Controller\Index\Index
6. Action execution: execute() returns ResultInterface (adminhtml uses its own layout)
```

---

## 6. Stateful Areas — Areas Are Not Stateless

This is a critical point that trips up many developers: **Areas are stateful**. They affect global system state, not just request-scoped context.

### 6.1 How Area Loading Mutates ObjectManager State

When `Area::load()` is called, it calls `$this->_objectManager->configure(...)`. This modifies the `ObjectManager`'s internal configuration. If two requests from different Areas execute in the same PHP process (possible in long-running CLI processes or certain server configurations), the second request may see stale configuration from the first.

In practice, Magento handles this by:
1. Reconfiguring the `ObjectManager` at the start of each request
2. Running `setup:di:compile` to generate fresh interceptor classes for each Area
3. Using isolated application instances in most deployment configurations

### 6.2 `Magento\Framework\App\State` — The Area Code Registry

**File**: `vendor/magento/framework/App/State.php`

```php
<?php
declare(strict_types=1);

namespace Magento\Framework\App;

class State
{
    /**
     * @var string
     */
    private $areaCode;

    /**
     * @var string
     */
    private $mode;

    /**
     * @var ConfigInterface
     */
    private $config;

    /**
     * @var AreaList
     */
    private $areaList;

    /**
     * @param string $areaCode
     * @return void
     */
    public function setAreaCode(string $areaCode): void
    {
        $this->areaCode = $areaCode;
    }

    /**
     * @return string
     */
    public function getAreaCode(): string
    {
        return $this->areaCode;
    }

    /**
     * Emulate area code
     *
     * @param string $areaCode
     * @return void
     */
    public function emulateAreaCode(string $areaCode): void
    {
        $this->areaCode = $areaCode;
    }

    /**
     * Revert to original area code
     *
     * @return void
     */
    public function revertAreaCode(): void
    {
        // Reverts to the original area code
    }
}
```

The `State` class holds the currently active area code as an instance property. This value is used by:
- `Magento\Framework\View\Design` to determine which theme to load
- `Magento\Framework\Translate` to determine which locale file to load
- Email templates to determine which design package to use for transactional emails
- `\Magento\Framework\App\Config` to scope configuration to the right area

### 6.3 Why This Matters: Theme Loading Depends on Current Area

**File**: `vendor/magento/framework/View/Design.php`

```php
<?php

namespace Magento\Framework\View;

class Design
{
    public function getArea(): string
    {
        return $this->_appState->getAreaCode() ?: 'frontend';
    }

    public function getConfiguration(): array
    {
        $area = $this->getArea();
        // Uses the current area code to determine:
        // - Which theme to load (adminhtml uses admin theme, frontend uses storefront theme)
        // - Which locale to use
        // - Which layout files to parse
    }
}
```

If you call code that depends on the current area (e.g., `Design::getTheme()`) before `State::setAreaCode()` has been called, you'll get incorrect results. This is a common source of bugs in console commands and background jobs.

---

## 7. Layout Building Per Area

### 7.1 Handle Loading Sequence

When Magento builds the layout for a request, it loads handles in a specific order. The handles determine which layout XML files are merged and which blocks are created.

**Handle loading sequence for a frontend product page:**

```
1. default             → vendor/magento/module-theme/view/frontend/layout/default.xml
2. catalog_product_view → module-catalog/view/frontend/layout/catalog_product_view.xml
3. STORE_{store_code}  → Store-specific overrides (if applicable)
4. THEME_{area}        → Theme overrides from app/design/frontend/Vendor/theme/
```

**Handle loading sequence for an admin product edit page:**

```
1. default              → vendor/magento/module-backend/view/adminhtml/layout/default.xml
2. adminhtml_product_edit → module-catalog/view/adminhtml/layout/catalog_product_edit.xml
3. cms_handle           → Additional CMS/admin-specific handles
```

### 7.2 Which Handles Are Loaded by Area

| Area | Default Handle | Module Handles | Theme Handles |
|------|---------------|----------------|---------------|
| `frontend` | `default` | `{module}_{controller}_{action}` | `Magento_Theme::default.xml`, theme layouts |
| `adminhtml` | `adminhtml_default` | `{module}_{controller}_{action}` | Admin theme layouts |
| `graphql` | None (no layout) | N/A | N/A |
| `webapi_rest` | None (no layout) | N/A | N/A |

### 7.3 Layout File Merging Per Area

Layout files are merged from modules in sequence order (defined by `module.xml`'s `<sequence>` tags) and then by theme inheritance (child theme overrides parent theme).

**File**: `vendor/magento/framework/Layout/ReaderPool.php`

```php
<?php
declare(strict_types=1);

namespace Magento\Framework\View\Layout;

class ReaderPool
{
    /**
     * @param Layout\Reader\Context $context
     * @return void
     */
    public function read(Layout\Reader\Context $context): void
    {
        $area = $context->getPageConfig()->getArea();

        // Area-specific read:
        // For frontend: reads from module-theme, module-catalog, etc. view/frontend/layout/
        // For adminhtml: reads from module-backend, module-catalog, etc. view/adminhtml/layout/

        // The $area determines which directory is used for each module
    }
}
```

The `ReaderPool` iterates over all modules and reads layout files from the `view/{area}/layout/` directory of each module. This is why layout files are organized by area: `view/frontend/layout/` for frontend, `view/adminhtml/layout/` for adminhtml.

### 7.4 `update` Handles

Layout handles can include other handles via `update` directives:

```xml
<!-- In a layout file -->
<layout xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/layout.xsd">
    <update handle="customer_account_overlay"/>
    <update handle="secure_handler"/>
</layout>
```

Update handles are processed first, allowing common layout structures to be defined once and reused across multiple pages.

---

## 8. Switching Areas Programmatically

There are rare but legitimate cases where you need to switch the current Area programmatically — for example, running a CLI command that should use frontend configuration, or sending an email from a cron job that needs adminhtml area context.

### 8.1 The `AreaList::getArea()->load()` Pattern

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Console\Command;

use Magento\Framework\App\AreaList;
use Magento\Framework\App\State;
use Magento\Framework\App\AreaInterface;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class SyncCatalogCommand extends Command
{
    public function __construct(
        private readonly AreaList $areaList,
        private readonly State $appState
    ) {
        parent::__construct();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        // We need frontend area to resolve product URLs
        $area = $this->areaList->getArea('frontend');
        $area->load('frontend');

        // Now ObjectManager is configured for frontend
        // DI preferences, plugins, etc. are all frontend-scoped

        // Do work...
        $this->syncProducts();

        return Command::SUCCESS;
    }
}
```

### 8.2 `State::emulateAreaCode()` for Temporary Area Switching

**File**: `vendor/magento/framework/App/State.php`

```php
<?php

class State
{
    /**
     * @var string|null
     */
    private $emulatedAreaCode;

    /**
     * @var string
     */
    private $areaCode;

    /**
     * Emulate area code for the duration of a callback
     *
     * @param string $areaCode
     * @param callable $callback
     * @return mixed
     */
    public function emulateAreaCode(string $areaCode, callable $callback): mixed
    {
        $previousArea = $this->areaCode;
        $this->areaCode = $areaCode;

        try {
            return $callback();
        } finally {
            $this->areaCode = $previousArea;
        }
    }
}
```

Usage:

```php
<?php

// Send an email that needs adminhtml context (for admin notifications)
$emailHtml = $this->appState->emulateAreaCode('adminhtml', function () {
    // Inside this closure, State::getAreaCode() returns 'adminhtml'
    // Layout, design, and translation all use adminhtml scope
    return $this->emailBuilder->buildAdminNotificationEmail();
});
```

This pattern is used in:
- `Magento\Framework\Mail\Transport` — to determine which email template to use
- `Magento\Cron\Model\Security` — to determine which area's ACL applies to cron jobs
- `Magento\Backend\Console\AreaPolicy` — to ensure CLI commands have proper area context before running

### 8.3 When to Switch Areas

**Legitimate use cases:**
1. CLI commands that need to render frontend-style output
2. Email template rendering in admin notifications
3. Data export/import scripts that need to use frontend product URLs
4. Unit tests that need to verify area-specific behavior

**When NOT to switch:**
1. In HTTP controllers — the area is already set by the front controller
2. In observers — the area is already active when observers fire
3. In plugins — the area is already determined by the calling context

---

## 9. Common Mistakes

### 9.1 Putting `di.xml` in the Wrong Area

**Wrong:**
```
app/code/Vendor/Module/etc/
├── di.xml                    ← Global — applies to ALL areas
├── frontend/
│   └── routes.xml           ← Correct
└── (di.xml accidentally placed here instead of etc/di.xml)
    └── routes.xml           ← WRONG: this di.xml won't load for frontend
```

If you place a plugin declaration in `etc/adminhtml/di.xml` but expect it to run on frontend requests, it won't. The plugin only loads when the adminhtml Area is active.

### 9.2 Using Frontend Preferences in Adminhtml

```xml
<!-- etc/frontend/di.xml -->
<type name="Vendor\Module\Model\Service">
    <preference for="Vendor\Module\Api\ServiceInterface"
                type="Vendor\Module\Model\Service\FrontendService"/>
</type>

<!-- etc/adminhtml/di.xml -->
<type name="Vendor\Module\Model\Service">
    <preference for="Vendor\Module\Api\ServiceInterface"
                type="Vendor\Module\Model\Service\AdminService"/>  <!-- DIFFERENT -->
</type>
```

If you inject `Vendor\Module\Api\ServiceInterface` into a controller, you'll get `FrontendService` in frontend requests and `AdminService` in adminhtml requests. This is intentional but can be confusing if you're debugging why a different implementation is being used.

### 9.3 ACL Checks That Differ by Area

Access control in adminhtml is enforced through `acl.xml` and the `Authorization` interface:

**File**: `vendor/magento/module-backend/Controller/Adminhtml/Auth.php`

```php
<?php

namespace Magento\Backend\Controller\Adminhtml;

class Auth
{
    public function checkAcl(string $resourceId): bool
    {
        // Adminhtml ACL is enforced here
        // Frontend does NOT enforce ACL through this mechanism
    }
}
```

A common mistake is expecting frontend controllers to respect admin ACL resources — they don't. Frontend ACL is a separate system (customer permissions, category permissions) that must be checked explicitly in your controller logic.

### 9.4 Forgetting Area-Specific Route Files When Creating a Module

When creating a module that adds a new page in the admin panel, you must:
1. Create `etc/adminhtml/routes.xml` (NOT `etc/frontend/routes.xml`)
2. Create layout files in `view/adminhtml/layout/` (NOT `view/frontend/layout/`)
3. Create menu items in `etc/adminhtml/menu.xml`
4. Declare ACL in `etc/adminhtml/acl.xml`

A common bug is creating `etc/routes.xml` (without the area subdirectory) and wondering why the route doesn't work. `etc/routes.xml` is not a valid Magento 2 configuration path — it must be `etc/frontend/routes.xml` or `etc/adminhtml/routes.xml`.

### 9.5 Layout Caching with Area-Sensitive Blocks

Blocks that behave differently per area must not be cached with the same cache key:

```xml
<!-- WRONG: cacheable block with area-dependent output -->
<block name="catalog.menu" class="Magento\Catalog\Block\Navigation"
       template="Magento_Catalog::nav/menu.phtml"
       cacheable="true"/>
<!-- If this template renders differently for frontend vs adminhtml, caching will cause issues -->
```

For area-sensitive content, either:
- Disable caching (`cacheable="false"`)
- Use `cache_tags` and `cache_lifetime` to make the cache key area-specific
- Use ` Magento\Framework\View\Element\Context::getCacheKeyInfo()` to include area in the key

---

## 10. GraphQL and WebAPI Area Behavior

### 10.1 GraphQL Area

The `graphql` Area is special: it has no layout system, no template files, and no traditional routing. Instead:

1. Requests come in at `/graphql` (configured in `app/etc/di.xml` frontName mapping)
2. `Area::load('graphql')` is called, which merges all `etc/graphql/di.xml` files
3. The GraphQL schema is built from all modules' `etc/graphql/schema.graphqls` files
4. The resolver processes the query and returns JSON

**File**: `vendor/magento/module-graph-ql/Model/Resolver/Processor.php`

```php
<?php

namespace Magento\GraphQl\Model\Resolver;

class Processor
{
    public function process(GraphQlQuery $query, Context $context): array
    {
        // The graphql area provides:
        // - Schema type definitions from etc/graphql/schema.graphqls
        // - Resolver classes from etc/graphql/di.xml
        // - Field args and directives

        // No layout system involved
    }
}
```

### 10.2 REST and SOAP Areas

The `webapi_rest` and `webapi_soap` Areas work similarly to GraphQL — they bypass the layout system. They load service class configurations from `etc/webapi_rest/di.xml` and `etc/webapi_soap/di.xml`, then process requests through the API dispatcher:

**File**: `vendor/magento/module-webapi/Controller/Rest.php`

```php
<?php

namespace Magento\Webapi\Controller;

class Rest
{
    public function dispatch(RequestInterface $request): ResponseInterface
    {
        // Loads webapi area
        // Reads etc/webapi.xml for route definitions
        // Maps route to service class + method
        // Calls service method and returns result as JSON/SOAP XML

        // No layout, no templates
    }
}
```

---

## 11. Complete Request-to-Area Flow

```
pub/index.php (Entry Point)
    │
    ▼
Bootstrap::run()
    │
    ▼
Http::launch()
    │
    ├─► AreaList::getCodeByFrontName(request.getFrontName())
    │       │
    │       ├─► "catalog" → "frontend"
    │       ├─► "admin"   → "adminhtml"
    │       └─► "graphql" → "graphql"
    │
    ├─► State::setAreaCode(areaCode)         ← STATEFUL: sets global state
    │
    ├─► AreaList::getArea(areaCode)->load()
    │       │
    │       └─► ObjectManager::configure(merged di.xml for that area)
    │           │
    │           ├─► etc/di.xml               (always loaded)
    │           ├─► etc/{area}/di.xml        (loaded for matching area)
    │           └─► etc/{area}/events.xml    (loaded for matching area)
    │
    ├─► FrontController::dispatch()
    │       │
    │       ├─► foreach RouterList as Router:
    │       │       ├─► Router::match(request)
    │       │       │       ├─► CMS Router
    │       │       │       ├─► UrlRewrite Router
    │       │       │       ├─► StandardRouter
    │       │       │       └─► DefaultRouter
    │       │       │
    │       │       └─► ActionFactory::create()->dispatch()
    │       │
    │       └─► ResultInterface::renderResult()
    │
    ▼
HTTP Response
```

Each Area gets its own independent:
- `ObjectManager` DI configuration
- Route resolution table
- Layout file compilation
- Template directory

---

## 12. Summary: Key Points to Remember

1. **Five Areas exist**: `frontend`, `adminhtml`, `graphql`, `webapi_rest`, `webapi_soap`, plus `crontab`

2. **Area is a horizontal slice**: A module can contribute to all Areas. `etc/frontend/di.xml` only applies to frontend requests; `etc/adminhtml/di.xml` only applies to admin requests.

3. **Area affects global state**: `State::setAreaCode()` sets a process-wide value that affects theme resolution, translation loading, and configuration scoping.

4. **Route resolution differs by Area**: Frontend uses CMS/UrlRewrite/Standard routers. Adminhtml uses a session-validating backend router. GraphQL/WebAPI bypass routing entirely and use area-specific dispatchers.

5. **Layout files are area-scoped**: Layout XML files in `view/frontend/layout/` and `view/adminhtml/layout/` are only loaded when their respective area is active.

6. **Switching areas is possible but deliberate**: Use `AreaList::getArea($code)->load()` or `State::emulateAreaCode()` when you need a different area context (CLI commands, email sending).

7. **The most common bug**: Putting `di.xml` in the wrong area subdirectory, causing plugins or preferences to not be loaded when expected.

---

## See Also

- **[Request Flow & Front Controller](/02-architecture/02-request-flow.md)** — Covers how `AreaList`, `State`, and `FrontController` work together through the request lifecycle, with detailed router matching and controller execution flow
- **[Coding Standards](/02-architecture/24-coding-standards.md)** — PSR-12 compliance, DocBlock requirements, and JavaScript/LESS standards enforced in Magento 2.4.8

---

*Magento 2.4.8 Deep Dive — Architecture Series*