---
title: "24 - Coding Standards"
description: "Magento 2.4.8 coding standards: PSR-12 PHP, ECMAScript/JavaScript ES6+, LESS, DocBlock XML, and technical guidelines for writing production-quality Magento code"
tags: [magento2, coding-standards, psr-12, php, javascript, less, docblock, eslint, code-style]
rank: 24
pathways: [magento2-deep-dive]
---

# 24 - Coding Standards

> "Programs must be written for people to read, and only incidentally for machines to execute." — Harold Abelson

This article covers the coding standards enforced in Magento 2.4.8, including PSR-12 PHP conventions, JavaScript/ECMAScript standards, LESS styling guidelines, and DocBlock documentation requirements. These standards ensure that code written across teams remains consistent, maintainable, and compatible with Magento's tooling ecosystem.

---

## 1. Why Coding Standards Matter

### The Case for Consistency

Coding standards are the foundation of sustainable software development. Without them:

- **Knowledge transfer suffers**: When every developer writes differently, understanding someone else's code requires decoding their personal style rather than focusing on business logic
- **Review cycles lengthen**: PRs become debates about formatting rather than architecture and correctness
- **Bug risk increases**: Inconsistent patterns hide bugs—tabs vs spaces, camelCase vs snake_case, strict vs loose comparisons
- **Onboarding friction grows**: New team members spend weeks learning "how we do things" instead of learning Magento itself

### Magento-Specific Requirements

Magento 2 differs fundamentally from Magento 1 in its approach to code structure:

| Aspect | Magento 1 | Magento 2 |
|--------|-----------|----------|
| Core Override Method | Direct class editing | Plugins/Interceptors |
| PHP Standard | PEAR coding standard | PSR-12 |
| Autoloading | Custom | Composer PSR-4 |
| Namespace Scope | Limited | Full PHP namespace support |

Magento 2 enforces its standards through:

- **PHP CodeSniffer (`phpcs`)**: Validates PHP and JavaScript code style
- **PHPStan**: Static analysis for type safety and error detection
- **ESLint**: JavaScript linting with Magento-specific rules
- **PHP Mess Detector**: Code complexity analysis

### The PSR-12 Baseline

[PSR-12](https://www.php-fig.org/psr/psr-12/) (Extended Coding Style) is the PHP-FIG standard that Magento 2 builds upon. It supersedes PSR-2 and defines:

- 4-space indentation (no tabs)
- Line length soft limit of 120 characters
- Opening braces on their own line for classes and methods
- Alphabetical use statement ordering
- Constant naming in `SCREAMING_SNAKE_CASE`
- Property and method naming in `$camelCase`

### PHP CodeSniffer Integration

Magento 2.4.8 includes the `MagentoCodingStandard` ruleset at:

```
dev/tests/static/framework/Magento/
```

This ruleset extends PSR-12 with Magento-specific sniffs:

```
Magento2.Arrays.ArrayNewline
Magento2.Classes.ClassMeta
Magento2.Namespaces.NamespaceDeclaration
Magento2.Functions.FunctionDeclaration
Magento2.CodeStyle.EmptyLines
Magento2.Commenting.ClassPropertyComment
Magento2.WhiteSpace.EmptyLineAfterWholeDefinition
```

To run code style checks:

```bash
# Check entire module
vendor/bin/phpcs --standard=Magento2 app/code/Vendor/Module/

# Check specific file
vendor/bin/phpcs --standard=Magento2 app/code/Vendor/Module/Model/Example.php

# With verbose output
vendor/bin/phpcs -vv --standard=Magento2 app/code/Vendor/Module/

# Show only errors (ignore warnings)
vendor/bin/phpcs --standard=Magento2 --severity=5 app/code/Vendor/Module/
```

### Git Pre-Commit Hooks Pattern

Integrate `phpcs` into your development workflow via pre-commit hooks:

```bash
#!/bin/bash
# .git/hooks/pre-commit

# Run phpcs on staged files
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.(php|js)$')

if [ -n "$STAGED_FILES" ]; then
    echo "Running PHP CodeSniffer on staged files..."
    vendor/bin/phpcs --standard=Magento2 $STAGED_FILES

    if [ $? -ne 0 ]; then
        echo "Code style violations found. Please fix them before committing."
        exit 1
    fi
fi

exit 0
```

For Husky (npm-based hooks):

```json
// package.json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged"
    }
  },
  "lint-staged": {
    "*.php": "vendor/bin/phpcs --standard=Magento2",
    "*.js": "vendor/bin/eslint"
  }
}
```

---

## 2. PSR-12 PHP Coding Standard

### File Structure

Every PHP file in Magento 2 must follow this structure:

```php
<?php
/**
 * Copyright © 2025 Vendor Name. All rights reserved.
 */

declare(strict_types=1);

namespace Vendor\Module\Controller\Index;

use Magento\Framework\App\Action\HttpGetActionInterface;
use Magento\Framework\Controller\ResultInterface;
use Magento\Framework\View\Result\PageFactory;

/**
 * Example controller demonstrating PSR-12 compliance.
 */
class Index implements HttpGetActionInterface
{
    /**
     * @var PageFactory
     */
    private PageFactory $resultPageFactory;

    /**
     * @param PageFactory $resultPageFactory
     */
    public function __construct(
        PageFactory $resultPageFactory
    ) {
        $this->resultPageFactory = $resultPageFactory;
    }

    /**
     * Execute request based on matched routing configuration.
     *
     * @return ResultInterface
     */
    public function execute(): ResultInterface
    {
        return $this->resultPageFactory->create();
    }
}
```

### Declaration Order

The `declare(strict_types=1)` statement must appear immediately after the PHP opening tag, before any other statements (including namespace). The copyright notice should precede it but remain after the opening tag on the same line or after a single blank line.

### Brace Placement

**Classes and Methods**: Opening brace on its own line.

```php
// PSR-12 Compliant
class ExampleClass
{
    public function doSomething(): void
    {
        // body
    }
}

// Non-compliant (brace on same line)
class BadExampleClass {
    public function badMethod() {
        // body
    }
}
```

**Control Structures**: Brace on the same line for control structures.

```php
// PSR-12 Compliant
if ($condition === true) {
    doSomething();
} elseif ($otherCondition === true) {
    doSomethingElse();
} else {
    doDefault();
}

// Alternative control structure brace style
while ($condition) {
    doSomething();
}

foreach ($items as $item) {
    processItem($item);
}
```

### Namespace and Use Statements

```php
<?php

declare(strict_types=1);

namespace Vendor\Module\Api\Data;

// Group 1: Magento Framework (alphabetical within group)
use Magento\Framework\Api\SearchResultsInterface;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Exception\NoSuchEntityException;

// Group 2: PHPUnit (only if used)
use PHPUnit\Framework\TestCase;

// Group 3: Magento module classes (alphabetical)
use Vendor\Module\Api\ItemRepositoryInterface;
use Vendor\Module\Api\Data\ItemInterface;

// Blank line between groups

// Internal use statements follow...
```

### Property and Method Naming

| Element | Convention | Example |
|---------|------------|---------|
| Classes | PascalCase | `ExampleClass` |
| Interfaces | PascalCase + Interface suffix | `ItemRepositoryInterface` |
| Traits | PascalCase | `LoggerTrait` |
| Constants | SCREAMING_SNAKE_CASE | `DEFAULT_PAGE_SIZE` |
| Member properties | $camelCase | `$this->itemRepository` |
| Methods | camelCase | `getItemById()` |
| Parameters | $camelCase | `itemId`, `searchCriteria` |
| Traits (using) | camelCase | `$this->logger` |

```php
class ItemService
{
    public const DEFAULT_PAGE_SIZE = 20;
    public const MAX_PAGE_SIZE = 100;

    private ItemRepositoryInterface $itemRepository;
    private int $pageSize;

    public function __construct(
        ItemRepositoryInterface $itemRepository,
        int $pageSize = self::DEFAULT_PAGE_SIZE
    ) {
        $this->itemRepository = $itemRepository;
        $this->pageSize = $pageSize;
    }

    public function getItem(int $itemId): ?ItemInterface
    {
        try {
            return $this->itemRepository->getById($itemId);
        } catch (NoSuchEntityException $e) {
            return null;
        }
    }
}
```

### The `final` and `readonly` Modifiers

Magento 2.4.8 requires classes used as services to be declared `final` to prevent direct extension:

```php
// Correct: Final class prevents decoration issues
final class ItemRepository implements ItemRepositoryInterface
{
    private ItemFactory $itemFactory;

    public function __construct(ItemFactory $itemFactory)
    {
        $this->itemFactory = $itemFactory;
    }
}

// Incorrect: Non-final class as service contract implementation
class ItemRepository implements ItemRepositoryInterface // VIOLATION
{
    // ...
}
```

Use `readonly` for constructor-injected dependencies that never change:

```php
// Magento 2.4.6+ : Constructor property promotion with readonly
final class ItemService
{
    public function __construct(
        private readonly ItemRepositoryInterface $itemRepository,
        private readonly SearchCriteriaBuilderFactory $searchCriteriaBuilderFactory
    ) {
    }
}
```

### Line Length and Wrapping

The soft limit is 120 characters. When lines exceed this:

```php
// Acceptable single-line (under 120 chars)
$items = $this->itemRepository->findByCategory($categoryId, $searchCriteria);

// Wrapped parameters (4 spaces indent)
$items = $this->itemRepository->findByCategory(
    $categoryId,
    $searchCriteria,
    $sortOrder
);

// Chained method calls
$collection = $this->itemRepository
    ->getCollection()
    ->addFieldToFilter('status', ['eq' => 'active'])
    ->setPageSize(self::DEFAULT_PAGE_SIZE);
```

### Before/After Comparison

**Before (Non-PSR-12):**

```php
<?php
namespace Vendor\Module\Controller;

class Index extends \Magento\Framework\App\Action\Action {
    private $resultPageFactory;
    public function __construct(
        \Magento\Framework\App\Action\Context $context,
        \Magento\Framework\View\Result\PageFactory $resultPageFactory
    ) {
        parent::__construct($context);
        $this->resultPageFactory = $resultPageFactory;
    }
    public function execute(){
        return $this->resultPageFactory->create();
    }
}
```

**After (PSR-12 Compliant):**

```php
<?php

declare(strict_types=1);

namespace Vendor\Module\Controller\Index;

use Magento\Framework\App\Action\HttpGetActionInterface;
use Magento\Framework\Controller\ResultInterface;
use Magento\Framework\View\Result\PageFactory;

/**
 * Index controller for module frontend.
 */
class Index implements HttpGetActionInterface
{
    /**
     * @param PageFactory $resultPageFactory
     */
    public function __construct(
        private readonly PageFactory $resultPageFactory
    ) {
    }

    /**
     * Execute request based on matched routing configuration.
     *
     * @return ResultInterface
     */
    public function execute(): ResultInterface
    {
        return $this->resultPageFactory->create();
    }
}
```

---

## 3. PHP DocBlock Standard

### Basic DocBlock Structure

Every `public` and `protected` method requires a DocBlock. The structure follows phpDocumentor conventions:

```php
/**
 * Short description of the method (single line preferred).
 *
 * Longer description if needed (optional). Can span multiple lines
 * and provide detailed explanation of the method's behavior.
 *
 * @param Type $paramName Description of parameter
 * @return Type Description of return value
 * @throws FullyQualified\ExceptionClass When condition causes exception
 */
public function methodName(Type $paramName): Type
```

### Type Annotations

Use lowercase for scalar types:

```php
/**
 * @param string $sku Product SKU identifier
 * @param int $quantity Number of items to add
 * @param bool $notify Notify customer of stock status
 * @param array $options Additional configuration options
 * @param float $price Item price in USD
 * @param null|string $couponCode Optional coupon code
 *
 * @return bool True on success
 * @throws LocalizedException If validation fails
 */
public function addItem(
    string $sku,
    int $quantity,
    bool $notify = true,
    array $options = [],
    float $price = 0.0,
    ?string $couponCode = null
): bool {
    // implementation
}
```

### InheritDoc for Interface Implementation

When implementing an interface method, use `@inheritdoc` to avoid duplicating documentation:

```php
/**
 * @inheritdoc
 */
public function getById(int $itemId): ?ItemInterface
{
    // implementation
}
```

For traits or parent classes, let the inheriting method reference the parent:

```php
/**
 * Retrieves item by ID from repository.
 *
 * @inheritdoc
 */
public function get(int $itemId): ?DataInterface
```

### API Annotations for Service Contracts

Magento 2 uses `@api` annotation to mark service contracts—interfaces that external code may depend on:

```php
<?php

declare(strict_types=1);

namespace Vendor\Module\Api;

use Vendor\Module\Api\Data\ItemInterface;

/**
 * Item repository interface for CRUD operations on items.
 *
 * @api
 */
interface ItemRepositoryInterface
{
    /**
     * Retrieve item by ID.
     *
     * @param int $itemId Item identifier
     * @return ItemInterface|null Item if found, null otherwise
     * @throws \Magento\Framework\Exception\NoSuchEntityException If item not found
     */
    public function getById(int $itemId): ?ItemInterface;

    /**
     * Save item to repository.
     *
     * @param ItemInterface $item Item to save
     * @return ItemInterface Saved item
     * @throws \Magento\Framework\Exception\CouldNotSaveException If save fails
     */
    public function save(ItemInterface $item): ItemInterface;

    /**
     * Delete item from repository.
     *
     * @param ItemInterface $item Item to delete
     * @return bool True on success
     */
    public function delete(ItemInterface $item): bool;
}
```

### Deprecation Notices

When deprecating functionality, include `@deprecated` with `@since` and `@see`:

```php
/**
 * Get item by SKU.
 *
 * @deprecated Use getById() instead. This method will be removed in version 3.0.0
 * @since 2.0.0
 * @see getById()
 *
 * @param string $sku Product SKU
 * @return ItemInterface|null
 */
public function getBySku(string $sku): ?ItemInterface
```

### What NOT to Include in DocBlocks

| Tag | Reason for Exclusion |
|-----|---------------------|
| `@author` | Authorship tracked in VCS, not code |
| `@category` | Deprecated phpDocumentor tag |
| `@package` | Creates artificial coupling |
| `@subpackage` | Deprecated concept |
| `@var` on properties | Use `@property` or declare types in code |
| `@method` on classes | Only use on actual methods |

Correct property documentation:

```php
/**
 * Search criteria builder.
 *
 * @var SearchCriteriaBuilder
 */
private SearchCriteriaBuilder $searchBuilder;

// Better: readonly with constructor promotion (M2.4.6+)
private readonly SearchCriteriaBuilder $searchBuilder;
```

---

## 4. JavaScript/ECMAScript Standards

### ES6+ Required Features

Magento 2.4.8 requires ECMAScript 6 (ES2015) or higher for all JavaScript files. The following features are mandatory:

```javascript
// ✓ Required: const/let instead of var
const config = {
    baseUrl: '/api/items',
    timeout: 5000
};

let requestCount = 0;

// ✓ Required: Arrow functions
const fetchItems = async (categoryId) => {
    const response = await fetch(`${config.baseUrl}/${categoryId}`);
    return response.json();
};

// ✓ Required: Template literals
const message = `Fetching ${count} items from ${config.baseUrl}`;

// ✓ Required: Destructuring
const { baseUrl, timeout } = config;
const [first, second, ...rest] = items;

// ✓ Required: Spread operator
const newItems = [...existingItems, ...newItemsArray];

// ✓ Required: Object shorthand
const createItem = (name, price) => ({ name, price });

// ✓ Required: Modules (export/import)
export default class ItemService {
    // class body
}

import ItemService from './ItemService';
```

**Prohibited patterns:**

```javascript
// ✗ Forbidden: var keyword
var oldStyle = 'deprecated';

// ✗ Forbidden: Traditional function expressions (in most contexts)
const oldHandler = function(event) {
    // deprecated
};

// ✗ Forbidden: String concatenation
const message = 'Count: ' + count + ' items';

// ✗ Forbidden: Non-template literal strings for dynamic content
```

### RequireJS Module Format

All Magento JavaScript uses RequireJS for module loading:

```javascript
// Define a module with dependencies
define([
    'jquery',
    'Magento_Ui/js/model/messageList',
    'mage/validation'
], function ($, messageList, validation) {
    'use strict';

    return function (config, element) {
        $(element).validation({
            submitHandler: function (form) {
                $.ajax({
                    url: config.saveUrl,
                    data: $(form).serialize(),
                    success: function (response) {
                        if (response.success) {
                            messageList.addSuccessMessage({
                                message: 'Saved successfully'
                            });
                        }
                    }
                });
            }
        });
    };
});
```

Named callback pattern for cleaner async handling:

```javascript
define([
    'jquery',
    'Magento_Customer/js/customer-data'
], function ($, customerData) {
    'use strict';

    return function (config, element) {
        const sectionData = customerData.get('cart');

        customerData.invalidate(['cart']);

        customerData.reload(['cart'], true).then(function () {
            // Handle reload complete
            updateMiniCart(sectionData);
        });
    };
});
```

### JSDoc for Function Documentation

```javascript
/**
 * Service for managing cart operations.
 *
 * @namespace MyModule/Cart
 */
define([
    'jquery',
    'Magento_Checkout/js/model/quote'
], function ($, quote) {
    'use strict';

    /**
     * Add item to shopping cart.
     *
     * @param {number} productId - Product identifier
     * @param {number} qty - Quantity to add
     * @returns {Promise<Object>} Promise resolving to add item response
     * @throws {Error} When product not found or invalid quantity
     */
    function addToCart(productId, qty) {
        return $.post('/cart/add', {
            product: productId,
            qty: qty
        });
    }

    return addToCart;
});
```

### ESLint Configuration

Magento 2.4.8 includes ESLint configuration at:

```
dev/tests/static/etc/eslintrc.js
```

The default configuration:

```javascript
module.exports = {
    extends: 'eslint:recommended',
    env: {
        browser: true,
        jquery: true,
        es6: true
    },
    parserOptions: {
        ecmaVersion: 2020,
        sourceType: 'module'
    },
    rules: {
        'no-unused-vars': 'warn',
        'no-console': 'off',
        'no-shadow': 'error',
        'eqeqeq': ['error', 'always'],
        'prefer-const': 'error'
    }
};
```

Run ESLint:

```bash
# Check all JavaScript files
vendor/bin/eslint app/code/Vendor/Module/view/frontend/web/js/

# Check specific file
vendor/bin/eslint app/code/Vendor/Module/view/frontend/web/js/model/item-service.js

# With auto-fix
vendor/bin/eslint --fix app/code/Vendor/Module/view/frontend/web/js/
```

### TypeScript Type Declarations

For complex components, use `.d.ts` files for type declarations:

```typescript
// etc/definitions.d.ts

declare namespace Magento_Ui {
    interface Component {
        initModules(): void;
        getData(): object;
    }
}

interface JQuery {
    modal(options?: ModalOptions): JQuery;
    confirmation(options: ConfirmationOptions): JQuery;
}

interface ModalOptions {
    title?: string;
    content?: string;
    buttons?: ButtonConfig[];
}
```

---

## 5. jQuery Widget Coding Standard

### Widget Declaration Pattern

Magento 2 jQuery widgets extend the `$.widget` base:

```javascript
define([
    'jquery',
    'jquery/ui'
], function ($) {
    'use strict';

    $.widget('vendor.itemSlider', {

        // Widget configuration (accessible via options)
        options: {
            itemsPerSlide: 4,
            autoPlay: false,
            autoPlayDelay: 3000,
            navigation: true,
            responsiveBreakpoints: {
                mobile: 1,
                tablet: 2,
                desktop: 4
            }
        },

        /**
         * Widget constructor.
         * @private
         */
        _create: function () {
            this._bind();
            this._initSlider();
        },

        /**
         * Initialize slider on first open.
         * @private
         */
        _init: function () {
            if (!this.options.autoPlay) {
                return;
            }
            this.startAutoPlay();
        },

        /**
         * Bind event handlers.
         * @private
         */
        _bind: function () {
            this.element.on('click' + this.eventNamespace, $.proxy(this._onClick, this));
            $(window).on('resize' + this.eventNamespace, $.debounce(500, $.proxy(this._onResize, this)));
        },

        /**
         * Event handlers stored for later removal.
         * @private
         */
        _events: {
            'click .slide-next': '_nextSlide',
            'click .slide-prev': '_prevSlide',
            'mouseenter': '_pauseAutoPlay',
            'mouseleave': '_resumeAutoPlay'
        },

        /**
         * Handle element click.
         * @private
         */
        _onClick: function (event) {
            // handler implementation
        },

        /**
         * Navigate to next slide.
         * @private
         */
        _nextSlide: function () {
            this.currentIndex = (this.currentIndex + 1) % this.totalSlides;
            this._render();
        }
    });

    return $.vendor.itemSlider;
});
```

### Namespaced Events

Always use namespaced events to prevent conflicts:

```javascript
// Widget-specific namespace
this.eventNamespace = '.vendor_itemSlider';

// Namespaced events
this.element.on('click' + this.eventNamespace, handler);
$(window).on('resize' + this.eventNamespace, handler);

// Trigger namespaced custom events
this.element.trigger('beforeSlideChange' + this.eventNamespace, [slideIndex]);
```

Handle custom events in other widgets:

```javascript
// In another widget
this.element.on('beforeSlideChange.vendor_itemSlider', function (event, slideIndex) {
    // React to slider change
});
```

### Widget Initialization

Two patterns exist for widget initialization:

**Pattern 1: `data-mage-init` attribute (preferred for declarative initialization)**

```html
<div data-mage-init='{"vendor_itemSlider": {"itemsPerSlide": 4, "autoPlay": true}}'
     class="item-slider">
    <!-- slides content -->
</div>
```

**Pattern 2: Programmatic initialization (for dynamic configuration)**

```javascript
define(['jquery', 'vendor/itemSlider'], function ($) {
    'use strict';

    return function (config, element) {
        $(element).itemSlider({
            itemsPerSlide: config.itemsPerSlide || 4,
            autoPlay: config.autoPlay || false,
            responsiveBreakpoints: {
                mobile: 1,
                tablet: 2,
                desktop: 4
            }
        });
    };
});
```

### Option Method for Configuration

The `option()` method allows runtime configuration changes:

```javascript
// Get option value
const itemsPerSlide = $(selector).itemSlider('option', 'itemsPerSlide');

// Set single option
$(selector).itemSlider('option', 'itemsPerSlide', 6);

// Set multiple options
$(selector).itemSlider('option', {
    itemsPerSlide: 6,
    autoPlay: true,
    navigation: false
});

// Retrieve all options
const allOptions = $(selector).itemSlider('option');
```

### Widget Destruction

Proper cleanup when widget is no longer needed:

```javascript
/**
 * Remove widget instance and cleanup.
 */
_destroy: function () {
    this._unbind();
    this.element.removeData(this.widgetFullName);
    this.element.empty();
},

/**
 * Unbind all event handlers.
 * @private
 */
_unbind: function () {
    this.element.off(this.eventNamespace);
    $(window).off(this.eventNamespace);
}
```

---

## 6. LESS Coding Standard

### File Organization

Magento LESS follows a specific structure:

```
app/design/frontend/Vendor/theme/
├── web/
│   ├── css/
│   │   ├── _styles.less          # Main entry point
│   │   ├── _variables.less       # Theme variables
│   │   ├── _extend.less          # Extends and mixins
│   │   └── _icons.less           # Icon definitions
│   ├── css/source/
│   │   ├── _extends.less         # Extension utilities
│   │   ├── _links.less           # Link styles
│   │   └── _layout.less          # Layout components
│   └── less/
│       ├── _variables.less        # All theme variables
│       └── _mixins.less           # Reusable mixins
```

### Nesting Depth Limit

Keep nesting to 3-4 levels maximum:

```less
// ✓ Good: 3 levels of nesting
.product-item {
    &__image {
        img {
            max-width: 100%;
        }
    }

    &__title {
        font-size: @product-title-font-size;
    }
}

// ✗ Bad: 5+ levels of nesting
.product-item {
    .product-item-details {
        .product-item-name {
            .product-item-link {
                .product-item-inner {
                    // too deep
                }
            }
        }
    }
}
```

### Variables and Mixins

**Variable naming**: Use descriptive, lowercase with hyphens:

```less
// Color palette variables
@color-white: #ffffff;
@color-black: #000000;
@color-brand-primary: #006bb4;
@color-brand-secondary: #ff5501;
@color-gray: #999999;
@color-gray-dark: #333333;
@color-gray-light: #e1e1e1;

// Typography variables
@font-family-base: 'Open Sans', sans-serif;
@font-size-base: 14px;
@font-weight-normal: 400;
@font-weight-bold: 700;

// Spacing variables
@spacing-xs: 4px;
@spacing-sm: 8px;
@spacing-md: 16px;
@spacing-lg: 24px;
@spacing-xl: 32px;
```

**Mixins with parentheses**:

```less
// Mixin definition
.border-radius(@radius) {
    border-radius: @radius;
    -webkit-border-radius: @radius;
    -moz-border-radius: @radius;
}

// Mixin usage
.button-primary {
    .border-radius(4px);
    background: @color-brand-primary;
    color: @color-white;
}

.button-secondary {
    .border-radius(0);
    background: @color-white;
    border: 1px solid @color-gray;
}
```

**Mixins without parentheses (pattern library)**:

```less
// Pattern mixin (no parentheses)
.action-primary {
    display: inline-block;
    padding: @spacing-sm @spacing-md;
    background: @color-brand-primary;
    color: @color-white;
    font-weight: @font-weight-bold;
}

// Usage
.primary-button {
    .action-primary;
}
```

### BEM Methodology

Use Block__Element--Modifier pattern:

```less
// Block
.product-list {
    &__items {
        display: flex;
        flex-wrap: wrap;
        margin: 0 -@spacing-sm;
    }

    &__item {
        flex: 0 0 100%;
        padding: @spacing-sm;

        @media (min-width: @screen-tablet) {
            flex: 0 0 50%;
        }

        @media (min-width: @screen-desktop) {
            flex: 0 0 33.333%;
        }
    }

    &__image {
        width: 100%;
    }

    &__title {
        font-size: @font-size-base;
        font-weight: @font-weight-bold;
    }

    // Modifier
    &--grid {
        .product-list__items {
            flex-direction: row;
        }
    }

    &--list {
        .product-list__items {
            flex-direction: column;
        }
    }
}
```

### Import Directives and Compilation Order

Magento LESS compilation follows a specific import order:

```less
// 1. Magento library (required base)
@import 'lib/_lib.less';

// 2. Variables (theming overrides)
@import '_variables.less';

// 3. Extend/utility styles
@import '_extend.less';

// 4. Components
@import 'css/source/_components.less';

// 5. Theme-specific styles
@import '_theme.less';

// 6. Module overrides
@import '_module.less';
```

The `@magento-import` directive handles module-level compilation:

```less
// Within a module's less file
// @magento-import directive compiles module CSS in correct order
@import '@{base_dir}/Magento_Theme/web/css/source/_links.less';
@import '@{base_dir}/Magento_Catalog/css/source/_module.less';
```

### Media Query Handling

Avoid deeply nested `@media` queries:

```less
// ✓ Good: Media queries at root level or shallow nesting
.product-grid {
    display: grid;
    grid-template-columns: 1fr;

    @media (min-width: @screen-tablet) {
        grid-template-columns: repeat(2, 1fr);
    }

    @media (min-width: @screen-desktop) {
        grid-template-columns: repeat(4, 1fr);
    }
}

// ✗ Bad: Deeply nested media queries
.product-grid {
    .product-grid-inner {
        .product-grid-items {
            @media (min-width: @screen-tablet) {
                // too deeply nested
            }
        }
    }
}
```

### Avoiding `!important`

Reserve `!important` for utility classes only:

```less
// ✗ Bad: Overusing !important
.title {
    font-size: 16px !important;
}

// ✓ Good: Specificity over !important
.product-details {
    .product-info {
        .title {
            font-size: 16px;
        }
    }
}

// ✓ Acceptable: Utility class for visibility control
.hidden {
    display: none !important;
}

.visible-hidden {
    visibility: hidden !important;
}
```

### Magento LESS Compiler

The `Magento\StyleMiner\LessCompiler` class processes LESS files during static content deployment:

```php
<?php

declare(strict_types=1);

namespace Magento\StyleMiner;

use Magento\Framework\Less\CompilerInterface;
use Magento\Framework\Less\FileGenerator;

/**
 * Less compiler for theme CSS generation.
 */
class LessCompiler implements CompilerInterface
{
    private FileGenerator $fileGenerator;
    private array $variableCollectors;

    /**
     * @param FileGenerator $fileGenerator
     * @param array $variableCollectors
     */
    public function __construct(
        FileGenerator $fileGenerator,
        array $variableCollectors = []
    ) {
        $this->fileGenerator = $fileGenerator;
        $this->variableCollectors = $variableCollectors;
    }

    /**
     * Compile LESS files to CSS.
     *
     * @param string[] $lessFiles
     * @param string $outputFile
     * @return void
     */
    public function compile(array $lessFiles, string $outputFile): void
    {
        // Compilation logic
    }
}
```

---

## 7. HTML Style Guide

### Script and Style Avoidance

Inline `<script>` and `<style>` tags should be avoided in templates:

```html
<!-- ✗ Bad: Inline JavaScript -->
<div onclick="addToCart(123)" class="product">...</div>

<!-- ✗ Bad: Inline styles -->
<div style="display: flex; justify-content: center;">...</div>

<!-- ✓ Good: External via data-mage-init -->
<div data-mage-init='{"addToCart": {"productId": 123}}' class="product">
    <span data-bind="text: name"></span>
</div>

<!-- ✓ Good: Using CSS classes -->
<div class="product product--centered">...</div>
```

### Knockout Data Bindings

Use `data-bind` for Knockout.js bindings:

```html
<!-- Knockout text binding -->
<span data-bind="text: productName"></span>

<!-- Conditional rendering -->
<div data-bind="visible: isInStock">
    <span data-bind="text: stockQty"></span>
</div>

<!-- Form bindings -->
<input type="text"
       data-bind="value: customerName,
                  valueUpdate: 'input',
                  attr: {placeholder: namePlaceholder}" />

<!-- Event handling -->
<button data-bind="click: addToCart">
    Add to Cart
</button>

<!-- Template rendering -->
<div data-bind="template: {name: 'product-template', foreach: products}">
</div>
```

### Translation Function

Use the `translate` filter for internationalized strings:

```html
<!-- Translate static text -->
<span data-bind="i18n: 'Add to Cart'"></span>

<!-- Translate with variable -->
<span data-bind="i18n: 'Welcome, ' + customerName"></span>

<!-- In templates (using Magento Translate mixin) -->
<span i18n="'Product price: ' + price"></span>

<!-- PHP template translation -->
<?= $escaper->escapeHtml(__('Add to Cart')) ?>
```

### Attribute Handling

**Always quote attributes:**

```html
<!-- ✓ Good: Quoted attributes -->
<input type="text" id="customer-name" name="customer[name]" />
<input type="checkbox" checked="checked" />

<!-- ✗ Bad: Unquoted attributes -->
<input type=text id=customer-name />
```

**Self-closing void elements:**

```html
<!-- ✓ Good: Self-closing void elements -->
<input type="hidden" name="form_key" value="<?= $escaper->escapeHtmlAttr($formKey) ?>" />
<br />
<hr />
<img src="<?= $escaper->escapeUrl($imageUrl) ?>" alt="Product image" />

<!-- ✗ Bad: Unnecessary closing tags on void elements -->
<input type="hidden" name="form_key" value="..." />
<br></br>
<hr></hr>
```

### Magento UI Component Templates

Follow Magento UI component conventions in `.html` templates:

```html
<!-- ko template: {name: 'Magento_Ui::grid/columns/actions.html',
                   data: {rows: $row, actions: actions}} -->
<!-- /ko -->

<div class="product-actions" data-bind="css: { 'product-actions--active': isOpen }">
    <button type="button"
            class="action"
            data-bind="attr: {title: actionLabel, 'aria-label': actionLabel}">
        <span data-bind="text: actionLabel"></span>
    </button>
</div>
```

---

## 8. CLI Static Tests

### PHP CodeSniffer Command

Run code style validation on your module:

```bash
# Basic usage - check entire module
vendor/bin/phpcs --standard=Magento2 app/code/Vendor/Module/

# Check with specific severity (errors only)
vendor/bin/phpcs --standard=Magento2 --severity=5 app/code/Vendor/Module/

# Verbose output showing sniff codes
vendor/bin/phpcs -vv --standard=Magento2 app/code/Vendor/Module/

# Check only PHP files in specific directory
vendor/bin/phpcs --standard=Magento2 --extensions=php app/code/Vendor/Module/Model/

# Generate report to file
vendor/bin/phpcs --standard=Magento2 -o report.txt app/code/Vendor/Module/

# Generate JSON report for CI integration
vendor/bin/phpcs --standard=Magento2 -o report.json --report-format=json app/code/Vendor/Module/
```

### Suppressing Specific Sniffs

Use inline annotations to suppress specific warnings:

```php
<?php

declare(strict_types=1);

namespace Vendor\Module\Model;

/**
 * Item model.
 */
class Item
{
    // phpcs:disable Magento2.Classes.ClassMeta
    // Note: This pattern is intentional for legacy compatibility
    use LegacyTrait {
        // phpcs:enable Magento2.Classes.ClassMeta
        getLegacyData as getData;
    }

    /**
     * @inheritDoc
     * @phpcs:disable Magento2.Functions.FunctionDeclaration
     */
    protected function _prepareData(array $data): array // Line exception here
    {
        return $data;
    }
    // phpcs:enable Magento2.Functions.FunctionDeclaration
}
```

Alternatively, create a `phpcs.xml` or `phpcs.xml.dist` in your module root:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ruleset name="Vendor Module PHP CodeSniffer">
    <description>PHP CodeSniffer configuration for Vendor Module</description>

    <file>app/code/Vendor/Module</file>

    <exclude-pattern>*/test/*</exclude-pattern>
    <exclude-pattern>*/Test/*</exclude-pattern>

    <!-- Suppress specific sniffs globally -->
    <rule ref="Magento2.Functions.FunctionDeclaration">
        <properties>
            <property name="allowComments" value="true"/>
        </properties>
    </rule>

    <!-- Ignore warnings but fail on errors -->
    <arg name="severity" value="5"/>
    <arg name="warningsprecious" value="true"/>

    <arg name="colors"/>
    <arg name="tab-width" value="4"/>
</ruleset>
```

### PHPStan Static Analysis

Magento 2.4.8 supports PHPStan for deeper static analysis:

```bash
# Run with default configuration
vendor/bin/phpstan analyse app/code/Vendor/Module/

# Run with specific level (0-9)
vendor/bin/phpstan analyse --level=5 app/code/Vendor/Module/

# Generate baseline for legacy code
vendor/bin/phpstan analyse --generate-baseline app/code/Vendor/Module/
```

Configuration in `phpstan.neon`:

```neon
parameters:
    level: 5
    paths:
        - app/code/Vendor/Module
    excludePaths:
        - app/code/Vendor/Module/Test
    ignoreErrors:
        - '#Method .* has parameter .* with no type specified#'
    reportUnmatchedIgnoredErrors: false
```

### GitHub Actions CI Integration

Create a workflow file at `.github/workflows/phpcs.yml`:

```yaml
name: Code Style

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  phpcs:
    name: PHP CodeSniffer
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: php-setup-action@v5
        with:
          php-version: '8.2'

      - name: Install dependencies
        run: composer install --no-interaction

      - name: Run PHP CodeSniffer
        run: vendor/bin/phpcs --standard=Magento2 --colors app/code/Vendor/Module/

  phpstan:
    name: PHPStan Analysis
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: php-setup-action@v5
        with:
          php-version: '8.2'

      - name: Install dependencies
        run: composer install --no-interaction

      - name: Run PHPStan
        run: vendor/bin/phpstan analyse --memory-limit=1G

  eslint:
    name: ESLint
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint:js
```

---

## 9. Technical Guidelines (Magento-Specific)

### No `@var` Annotations on Properties

Use proper type declarations in code instead of `@var` annotations:

```php
// ✗ Bad: @var annotation
/**
 * @var ItemRepositoryInterface
 */
private $itemRepository;

// ✓ Good: Constructor promotion with type declaration
private readonly ItemRepositoryInterface $itemRepository;

// ✓ Good: Type hint in property declaration
/** @var ItemRepositoryInterface */
private ItemRepositoryInterface $itemRepository;

// ✓ Good (M2.4.6+): Constructor property promotion
public function __construct(
    private readonly ItemRepositoryInterface $itemRepository
) {
}
```

### Avoid CodeIgnore Comments

Minimize use of `// @codingStandardsIgnoreStart/End`:

```php
// ✗ Bad: Entire blocks ignored
// @codingStandardsIgnoreStart
function legacyFunction() {
    // old incompatible code
}
// @codingStandardsIgnoreEnd

// ✓ Good: Targeted suppressions only
public function processLegacyData(array $data): array
{
    $legacy = $this->legacyProcessor->process(/** @scrutinizer ignore-type */ $data);
    return $legacy;
}
```

### No Global Variables

Magento architecture does not support `global` keyword usage:

```php
// ✗ Bad: Global variable
function processConfig() {
    global $config;
    return $config['setting'];
}

// ✓ Good: Dependency injection
class ConfigProcessor
{
    private ConfigInterface $config;

    public function __construct(ConfigInterface $config)
    {
        $this->config = $config;
    }

    public function process(): array
    {
        return $this->config->getSetting();
    }
}
```

### No `@author` Tags

Authorship belongs in version control, not DocBlocks:

```php
// ✗ Bad: @author tag
/**
 * @author John Developer <john@example.com>
 * @author Jane Developer <jane@example.com>
 */
public function calculateTax(): float

// ✓ Good: No @author, rely on git blame
/**
 * Calculate tax for order based on applicable rules.
 *
 * @param OrderInterface $order
 * @return float Tax amount
 */
public function calculateTax(OrderInterface $order): float
```

### No `@category`, `@package`, or `@subpackage` Tags

These phpDocumentor tags create artificial coupling:

```php
// ✗ Bad: Unnecessary DocBlock tags
/**
 * @category VendorModule
 * @package Vendor_Module
 * @subpackage Model
 */
class Item

// ✓ Good: Minimal DocBlock
/**
 * Item model representing catalog item entity.
 */
class Item
```

### Constructor Parameter Type Matching

Constructor parameters must match their injected types exactly:

```php
// ✗ Bad: Type mismatch
public function __construct(
    ItemRepositoryInterface $itemRepository,
    $someParameter // Missing type hint
) {
    $this->itemRepository = $itemRepository;
    $this->someParameter = $someParameter;
}

// ✓ Good: All parameters typed
public function __construct(
    ItemRepositoryInterface $itemRepository,
    int $pageSize = self::DEFAULT_PAGE_SIZE,
    ?LoggerInterface $logger = null
) {
    $this->itemRepository = $itemRepository;
    $this->pageSize = $pageSize;
    $this->logger = $logger;
}
```

### No `@method` on Actual Methods

The `@method` annotation is for magic methods, not real method implementations:

```php
// ✗ Bad: @method on actual method
/**
 * @method string getName()
 * @method void setStatus(int $status)
 */
class Item
{
    public function getName(): string // Real method, not magic
    {
        return $this->name;
    }
}

// ✓ Good: No @method for real methods, use @return instead
/**
 * Get item display name.
 *
 * @return string
 */
public function getName(): string
{
    return $this->name;
}
```

---

## Further Reading

- [Adobe Commerce Coding Standards](https://developer.adobe.com/commerce/php/coding-standards/)
- [PSR-12: Extended Coding Style](https://www.php-fig.org/psr/psr-12/)
- [Magento PHP Coding Standard (phpcs)](https://github.com/magento/magento-coding-standard)
- [ESLint Configuration for Magento](https://developer.adobe.com/commerce/frontend-core/guide/css/coding-standards/)
- [JavaScript Coding Standards](https://developer.adobe.com/commerce/javascript/coding-standards/)
- [LESS Coding Standards](https://developer.adobe.com/commerce/frontend-core/guide/css/coding-standards/less/)
- [DocBlock Documentation Standard](https://phpdoc.org/docs/)
- [Magento Security Best Practices](https://developer.adobe.com/commerce/php/best-practices/security/)