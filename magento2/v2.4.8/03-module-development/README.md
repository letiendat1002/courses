# Topic 3: Module Development — Controllers, Blocks, Templates & Layout XML

**Goal:** Build a complete page in Magento — from URL routing to rendered HTML — using controllers, blocks, templates, and layout XML.

---

## Topics Covered

- How Magento routes a URL to a controller action
- Controller implementation patterns (HttpGetActionInterface, HttpPostActionInterface)
- Block classes — PHP logic attached to templates
- PHTML templates — PHP-driven HTML rendering
- Layout XML — Magento's declarative layout system (handles, containers, references)
- PHPCS code quality — PSR-12 Magento coding standard
- Admin routes (preparation for Topic 6)

---

## Reference Exercises

- **Exercise 2.1:** Create a controller that returns a custom message
- **Exercise 2.2:** Build a block that passes dynamic data to a template
- **Exercise 2.3:** Use layout XML to position elements on a page
- **Exercise 2.4:** Run PHPCS and fix any reported errors

---

## Completion Criteria

- [ ] Working route with custom controller returning content (verified with curl)
- [ ] Custom block passes data to a PHTML template (verified by rendering)
- [ ] Layout XML positions elements without modifying core (no core file edits)
- [ ] PHPCS reports zero errors on your module code
- [ ] `declare(strict_types=1)` present in all PHP files
- [ ] All outputs escaped with `$escaper->escapeHtml()` in templates
- [ ] Controller implements `HttpGetActionInterface` or `HttpPostActionInterface`
- [ ] ViewModel used instead of block extension for complex logic
- [ ] Git commit follows Conventional Commits format

---

## Topics

---

### Topic 1: Request Flow

**How Magento Routes a URL:**

```
http://example.com/helloworld/index/index
              │         │      │      │
              │         │      │      └─── Action (Index)
              │         │      └─────────── Controller (Index)
              │         └─────────────────── Route ID (helloworld)
              └───────────────────────────── Front name (from routes.xml)
```

> **Why does this routing structure exist?** Magento's routing is designed so that `frontName` maps to a module, and the next two segments map to Controller folder and Action method. This allows Magento to lazily load only the controller that handles the request — not all controllers for every request.

**The flow:**

1. URL hits `index.php` entry point
2. `Router` matches URL to a route ID and frontName
3. `Controller` is matched (controller path + action)
4. Controller's `execute()` runs
5. Controller returns a `Result` (page, JSON, redirect)
6. Result renders

> **Deep Dive:** See **Topic 02** for the complete request lifecycle including AreaResolvers, router chain, and HttpEntryPoint. This topic focuses on the Controller layer.

**Router Matching in `routes.xml`:**

```xml
<!-- etc/frontend/routes.xml -->
<router id="standard">
    <route id="helloworld" frontName="helloworld">
        <module name="Training_HelloWorld"/>
    </route>
</router>
```

The `frontName` becomes the first URL segment. `route id` is the internal identifier.

> **Why both `frontName` and `route id`?** `frontName` is the URL-visible part (user-facing). `route id` is the internal identifier used in layout handles and code references. Keeping them separate allows URL rewrites to change the visible URL without breaking internal references.

---

### Topic 2: Controllers

**Controller Location Pattern:**

```
Controller/[Area]/[ControllerPath]/[Action].php
```

For `http://localhost/helloworld/index/index`:
- Area: empty (frontend) → `Controller/`
- ControllerPath: `Index` → `Index/`
- Action: `Index` → `Index.php`

```php
<?php
declare(strict_types=1);

// Controller/Index/Index.php
namespace Training\HelloWorld\Controller\Index;

use Magento\Framework\App\Action\HttpGetActionInterface;
use Magento\Framework\Controller\ResultFactory;
use Magento\Framework\View\Result\PageFactory;

class Index implements HttpGetActionInterface
{
    private ResultFactory $resultFactory;

    public function __construct(ResultFactory $resultFactory)
    {
        $this->resultFactory = $resultFactory;
    }

    public function execute()
    {
        /** @var \Magento\Framework\View\Result\Page $resultPage */
        $resultPage = $this->resultFactory->create(ResultFactory::TYPE_PAGE);
        $resultPage->getConfig()->getTitle()->set('Hello World');
        return $resultPage;
    }
}
```

> **Why `HttpGetActionInterface`?** This marker interface tells Magento the action handles GET requests. For POST, use `HttpPostActionInterface`. The framework checks this before calling `execute()`. Omitting it works but loses semantic clarity.

> **Common Pitfall:** Returning nothing from `execute()`. Every action must return a `ResultInterface`. For redirects: `return $this->resultRedirectFactory->create()->setPath('path')`. For JSON: `return $this->resultFactory->create(ResultFactory::TYPE_JSON)`.

**Result Types:**

| Result Type | Use For | Return Statement |
|-------------|---------|-----------------|
| `ResultFactory::TYPE_PAGE` | Full HTML page | `return $resultPage` |
| `ResultFactory::TYPE_JSON` | API responses | `return $resultJson` |
| `ResultFactory::TYPE_REDIRECT` | HTTP redirects | `return $redirect` |
| `ResultFactory::TYPE_FORWARD` | Internal forward (no URL change) | `return $forward` |
| `ResultFactory::TYPE_RAW` | Direct output (images, files) | `return $resultRaw` |

**Layout Handle Convention:**

| URL | Layout Handle |
|-----|--------------|
| `helloworld/index/index` | `helloworld_index_index.xml` |
| `helloworld/index/edit` | `helloworld_index_edit.xml` |
| `helloworld/review/save` | `helloworld_review_save.xml` |

> **Why layout handles follow URL convention?** Magento's layout system automatically loads `helloworld_index_index.xml` for `helloworld/index/index`. This connects URL → Controller → Layout XML without explicit registration. Change the URL, layout XML follows automatically.

> **Best Practice:** Keep controllers thin. A controller should: (1) receive input, (2) delegate to a service/repository, (3) return a result. If your controller has business logic beyond this, extract it to a Service class.

---

### Topic 3: Blocks

**Block Role:** Block classes hold PHP logic that prepares data for templates. They are instantiated by the layout system and available in templates via `$block`.

> **Why Blocks Exist?** Blocks are the "ViewModel" layer in Magento's MV architecture. They separate business logic from presentation. Templates (PHTML) should only contain HTML structure and simple conditionals/loops. All complex logic belongs in blocks.

**Minimal Block:**

```php
<?php
declare(strict_types=1);

// Block/Message.php
namespace Training\HelloWorld\Block;

use Magento\Framework\View\Element\Template;
use Magento\Framework\View\Element\Template\Context;

class Message extends Template
{
    private string $message;

    public function __construct(Context $context, array $data = [])
    {
        parent::__construct($context, $data);
        $this->message = 'Hello from the block!';
    }

    public function getMessage(): string
    {
        return $this->message;
    }

    public function getUppercaseMessage(): string
    {
        return strtoupper($this->message);
    }
}
```

> **Why `Template\Context`?** The Context provides the block with access to `ViewRegistry`, `Logger`, `Scaler`, and other view-layer services. Never remove it from the constructor signature.

**Key rule:** Blocks don't output HTML. Templates do. Blocks prepare data.

> **Common Pitfall:** Doing HTML rendering in blocks. `$this->getHtml('something')` in a block is a code smell. Keep HTML in templates.

**Available in templates via `$block`:**

```php
<?php
/** @var \Training\HelloWorld\Block\Message $block */
/** @var \Magento\Framework\Escaper $escaper */
?>
<h1><?= $escaper->escapeHtml($block->getMessage()) ?></h1>
<p><?= $escaper->escapeHtml($block->getUppercaseMessage()) ?></p>
```

> **Why `$escaper` vs `$block->escapeHtml()`?** Both escape HTML. `$escaper` is preferred in modern Magento because it's explicit and works in any context (templates, controllers, blocks). `$block->escapeHtml()` only works in templates where `$block` is available.

**Always escape output** with `$escaper->escapeHtml()` to prevent XSS.

> **Security Note:** XSS in Magento is a critical vulnerability. ANY user-submitted data rendered without escaping causes XSS. Product names, review text, customer names — escape ALL output unless you have explicit trust.

**ViewModels (Alternative to Blocks):**

For complex logic, use a ViewModel instead of extending the full `Template` block:

```php
<?php
// In layout XML:
<block class="Magento\Framework\View\Element\Template"
       name="training.message"
       template="Training_HelloWorld::message.phtml">
    <arguments>
        <argument name="view_model" xsi:type="object">Training\HelloWorld\ViewModel\Message</argument>
    </arguments>
</arguments>
```

```php
<?php
declare(strict_types=1);

namespace Training\HelloWorld\ViewModel;

use Magento\Framework\View\Element\Block\ArgumentInterface;

class Message implements ArgumentInterface
{
    public function getGreeting(): string
    {
        return 'Hello from ViewModel!';
    }
}
```

> **Why ViewModels?** ViewModels implement `ArgumentInterface`, allowing you to inject them via layout XML arguments. They keep block classes lean and allow multiple ViewModels per template. Use ViewModels when you need the same data on multiple blocks.

---

### Topic 4: Templates (PHTML)

**Template Location:**

```
view/[area]/templates/[Vendor]/[Module]/[file].phtml
```

For frontend templates: `view/frontend/templates/training/helloworld/message.phtml`

> **Why this path structure?** The path maps to module: `Training_HelloWorld::message.phtml` resolves to `view/frontend/templates/training/helloworld/message.phtml`. The double colon (`::`) is the module-specific template path separator in Magento's template notation.

**Connecting Block to Template in Layout XML:**

```xml
<referenceContainer name="content">
    <block class="Training\HelloWorld\Block\Message"
           name="training.message"
           template="Training_HelloWorld::message.phtml"/>
</referenceContainer>
```

> **Why `referenceContainer`?** `referenceContainer` references an existing container (defined in the page layout or parent theme). `container` creates a new wrapper. You rarely create new containers — you add blocks to existing ones.

**Common Template Patterns:**

```php
<?php
/** @var \Training\HelloWorld\Block\Message $block */
/** @var \Magento\Framework\Escaper $escaper */
?>

<!-- Output escaping — ALWAYS escape user data -->
<h1><?= $escaper->escapeHtml($block->getMessage()) ?></h1>

<!-- Conditional -->
<?php if ($block->getMessage()): ?>
    <p>Message: <?= $escaper->escapeHtml($block->getMessage()) ?></p>
<?php endif; ?>

<!-- Loop -->
<?php foreach ($block->getItems() as $item): ?>
    <li><?= $escaper->escapeHtml($item) ?></li>
<?php endforeach; ?>

<!-- URL generation -->
<a href="<?= $escaper->escapeUrl($block->getUrl('helloworld/index/index')) ?>">Go to HelloWorld</a>

<!-- Translation -->
<span><?= $escaper->escapeHtml(__('Welcome, %1!', $block->getCustomerName())) ?></span>
```

> **Pro Tip:** Use `__()` for translatable strings. Magento's i18n system extracts these for translation files. Always wrap user-facing strings in `__()`.

> **Common Pitfall:** Using `$block->getUrl()` in blocks/templates, but the URL is wrong. `getUrl()` by default generates frontend URLs. For adminhtml, use `$block->getUrl('adminhtml/path')` or inject `UrlInterface` and call `setScope()`.

**Template hint for debugging:**

```bash
bin/magento dev:template-hints:enable
bin/magento cache:flush
```

Then hover over elements in the browser to see which block/template renders each piece.

> **Pro Tip:** Use `bin/magento dev:template-hints:show` and `bin/magento dev:template-hints:hide` to toggle. Don't forget to disable after debugging — template hints slow down rendering.

**Never Do in Templates:**

| ❌ Never | Why | ✅ Instead |
|---------|-----|------------|
| Database queries | Templates have no DB access | Block/ViewModel methods |
| Complex PHP logic | Violates separation of concerns | Block methods |
| `echo $variable` without escape | XSS vulnerability | `$escaper->escapeHtml()` |
| `include` other templates | Layout XML should control includes | Use `$block->getChildHtml()` |
| `$_POST`/`$_GET` directly | Security risk, breaks request abstraction | Controller receives input |

---

### Topic 5: Layout XML

**Layout XML Role:** Layout XML declares which blocks belong on a page, their order, and their configuration. It does not write HTML — it orchestrates blocks.

> **Why Layout XML?** Layout XML is Magento's declarative way to compose pages. Instead of one giant PHP template that outputs everything, Magento splits each piece into separate blocks. This allows different modules to add/remove/modify blocks without editing each other's code — a form of composition over inheritance.

**Anatomy of a Layout Handle:**

```
helloworld_index_index.xml
└── helloworld        ← frontName (from routes.xml)
    └── index        ← controller path
        └── index   ← action name
```

> **Why does the layout handle map to URL structure?** This convention means URL → Layout XML is automatic. Add a new route, and Magento looks for the matching layout file. No manual registration needed.

**Layout File — `view/frontend/layout/helloworld_index_index.xml`:**

```xml
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/layout.xsd">
    <body>
        <referenceContainer name="page.wrapper">
            <container name="custom.container" htmlTag="div" htmlClass="custom-wrapper">
                <block class="Training\HelloWorld\Block\Message"
                       name="training.message"
                       template="Training_HelloWorld::message.phtml"/>
            </container>
        </referenceContainer>
    </body>
</page>
```

**Key Elements:**

| Element | Purpose |
|---------|---------|
| `<page>` | Root element, defines page type (also defines namespace for layout) |
| `<body>` | All visible content lives here |
| `<referenceContainer>` | Reference existing containers (page.wrapper, content, etc.) |
| `<container>` | Group blocks with optional wrapper tag (div/ul/etc.) |
| `<block>` | Define a block and its template |
| `<move>` | Move a block to a different container |
| `<remove>` | Remove a block |

> **Common Pitfall:** Creating `<container>` when you should use `<referenceContainer>`. Containers add wrapper HTML. If you just want to place a block inside an existing container, use `<referenceContainer name="content">` with a `<block>` inside — don't wrap in a new container unless you need a wrapper.

**Common Containers:**

| Container | Location |
|----------|---------|
| `page.wrapper` | Outer wrapper — everything inside `<body>` |
| `header` | Top header |
| `main.content` | Main content area |
| `content` | Content area (inside main.content) |
| `footer` | Bottom footer |

> **Pro Tip:** Check `vendor/magento/module-theme/view/frontend/page_layout/` for layout definitions. These define the containers available for your module.

**Arguments (passing data to blocks):**

```xml
<block class="Training\HelloWorld\Block\Message"
       name="training.message"
       template="Training_HelloWorld::message.phtml">
    <arguments>
        <argument name="title" xsi:type="string">Hello!</argument>
    </arguments>
</block>
```

Access in block: `$this->getData('title')` or in template: `$block->getData('title')`.

> **Best Practice:** Use arguments for configuration that changes per-instance (different page has different title). Use block class properties for configuration that's the same everywhere (shared settings).

**`<move>` Element — Reorder Blocks:**

```xml
<!-- Move a block from content to header -->
<move element="training.message" destination="header" after="-"/>
```

| Attribute | Meaning |
|-----------|---------|
| `destination` | Container to move TO |
| `before` | Insert before specified element |
| `after` | Insert after specified element |
| `after="-"` | Insert at beginning |
| `after="-"` | Insert at end |

**`<remove>` Element — Remove Blocks:**

```xml
<!-- Remove a block from parent module -->
<referenceBlock name="catalog.product.related" remove="true"/>
```

> **Common Pitfall:** `<remove>` only works inside `<referenceBlock>`, not `<block>`. The syntax is `<referenceBlock name="block.to.remove" remove="true"/>`.

**Layout XML Processing Order:**

1. `default.xml` loads first (all pages)
2. Page-specific handle loads (`helloworld_index_index.xml`)
3. Sequential — later XML can move/remove blocks from earlier XML
4. Blocks render in the order they appear in the final merged layout

---

### Topic 6: PHPCS — Code Quality

**PHPCS (PHP CodeSniffer)** enforces coding standards. Magento uses PSR-12 + Magento-specific rules.

> **Why PHPCS matters:** Magento's codebase has hundreds of developers. Without coding standards, code is inconsistent, hard to read, and hard to maintain. PHPCS catches style issues automatically, so reviewers can focus on logic, not formatting.

**Install PHPCS:**

```bash
composer require --dev magento/magento-coding-standard
vendor/bin/phpcs --version
```

> **Note:** The `magento/magento-coding-standard` package includes both PSR-12 rules and Magento-specific rules. Run it on your module before every commit.

**Run PHPCS:**

```bash
vendor/bin/phpcs --standard=Magento2 app/code/Training/HelloWorld
```

**Fixing Errors:**

```bash
# Auto-fix what PHPCS can fix (most issues)
vendor/bin/phpcbf --standard=Magento2 app/code/Training/HelloWorld
```

> **Pro Tip:** `phpcbf` fixes ~80% of issues automatically. Run it first, then manually fix the remaining issues. Never run `phpcbf` on vendor code — only your module code.

**Common Issues to Fix:**

| Issue | PHPCS Code | Fix |
|-------|------------|-----|
| Missing namespace | `Magento2.PHP.Namespace` | Add `namespace Vendor\Module;` |
| Incorrect brace placement | `Generic.ControlStructures.ControlSignature` | Put `{` on same line |
| Missing docblock | `Magento2.CodeAnalysis.Markdown` | Add docblock above class |
| Line too long | `Generic.Files.LineLength` | Break into multiple lines |
| Missing use statement | `Magento2.PHP.Literal` | Add `use` for class references |
| `declare(strict_types)` missing | `Magento2.PHP.Syntax` | Add `<?php declare(strict_types=1);` at top |
| Protected without var tag | `Magento2.PHP.Var` | Use `@var` in docblock or `private` property |

**PSR-12 Key Rules:**

| Rule | Correct | Incorrect |
|------|---------|-----------|
| Opening brace | `class Foo {` | `class Foo\n{` |
| Indentation | 4 spaces | Tabs |
| Namespace | `Vendor\Module\Controller` | `vendor\module\controller` |
| Use statements | After namespace, alphabetically sorted | Random order |
| Line length | Max 120 chars | Lines over 120 chars |

**Pre-commit Hook (optional but recommended):**

Add to `composer.json` scripts:

```json
"scripts": {
    "pre-commit": "phpcs --standard=Magento2 app/code/Training/"
}
```

Or configure in `.git/hooks/pre-commit`.

> **Best Practice:** Use a pre-commit hook so code can't be committed with violations. This is especially important in teams — one developer's unformatted code affects everyone.

---

### Topic 7: Git Workflow for Team Development

**Why Git?** Real Magento projects involve multiple developers. Git enables collaboration, code review, and safe deployments.

> **Magento-Specific Git Note:** Magento's codebase is large. `.gitignore` should exclude `vendor/`, `var/`, `generated/`, `pub/static/`, and `app/etc/env.php`. Only commit your module code and configuration.

**Basic Git Commands:**

```bash
git status                    # Check changes
git add .                     # Stage all changes
git commit -m "Add review module"  # Commit with message

# Create feature branch
git checkout -b feature/add-review-grid

# Switch branches
git checkout main
git checkout feature/add-review-grid
```

> **Pro Tip:** Never commit directly to `main` or `develop`. Every change goes through a feature branch → PR → review → merge.

**Git Flow (Production Standard):**

```
main (production-ready) ────────────────────────────
  ↑                                               │
develop (integration branch) ←────────────────── merge
  ↑                                               │
feature/add-review-grid (your work) ←── PR & review
```

> **Why this structure?** `main` is always deployable. `develop` is where features integrate. Feature branches keep work isolated until it's reviewed and tested. This prevents broken code from reaching production.

**Creating a Pull Request (PR):**

1. Push your branch: `git push -u origin feature/add-review-grid`
2. Go to GitHub/GitLab → Create Pull Request
3. Description: What you changed, why, testing done
4. Request code review from teammate
5. Address feedback, get approval, merge

> **Best Practice:** Keep PRs small (< 400 lines changed). Large PRs are hard to review thoroughly. Break big features into multiple small PRs.

**Commit Message Best Practices:**

```bash
# Good — Conventional Commits format
git commit -m "feat(review): add admin grid with listing UI component"
git commit -m "fix(review): resolve rating validation edge case"
git commit -m "docs(review): update README with installation steps"

# Bad — Too vague!
git commit -m "updates"          # What updates?
git commit -m "fixed stuff"     # What stuff?
git commit -m "WIP"             # Never commit WIP to shared branches
```

**Rule:** Write commit messages that explain "why", not just "what".

**Conventional Commits Format:**

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

| Type | When to Use |
|------|-------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, no code change |
| `refactor` | Code change that neither fixes nor adds |
| `test` | Adding tests |
| `chore` | Maintenance tasks |

> **Common Pitfall:** Committing generated files (factories, proxies in `var/generation/`). These are regenerated by Magento and should NOT be in git. They cause merge conflicts and bloat the repo.

---

## Reading List

- [Magento 2 Request Flow](https://developer.adobe.com/commerce/php/development/build/request-flow/) — How routing works
- [Layouts Overview](https://developer.adobe.com/commerce/frontend-core/guide/layouts/) — Layout XML structure
- [PSR-12 Coding Standard](https://www.php-fig.org/psr/psr-12/) — Code style rules

---

## Edge Cases & Troubleshooting

| Issue | Symptom | Solution |
|-------|---------|----------|
| Route 404 | Controller not found | Check `routes.xml` frontName matches; run `setup:upgrade` |
| Template not found | Blank area on page | Check template path matches layout XML reference |
| Block method undefined | Fatal error in template | Check method exists in Block class |
| Layout changes not showing | Old layout persists | `bin/magento cache:flush` |
| PHPCS errors on generated code | Vendor files flagged | Run PHPCS only on `app/code/` |
| Layout XML not loading | Block doesn't appear | Check layout handle filename matches URL exactly |
| XSS vulnerability | Script injection in output | Always use `$escaper->escapeHtml()` on user data |
| Double rendering | Block appears twice | Check for duplicate `<move>` or `<referenceBlock>` |
| PHPCS `Var` warning | Property not documented | Use `private` property with type hint instead of docblock |

---

## Common Mistakes to Avoid

1. ❌ Writing HTML directly in a controller → Use block + template pattern
2. ❌ Not escaping `$escaper->escapeHtml()` in templates → XSS vulnerability
3. ❌ Putting templates in wrong area folder → `frontend/` vs `adminhtml/`
4. ❌ Hardcoding URLs → Use `$block->getUrl()` for URLs
5. ❌ Editing layout XML in vendor folders → All custom layout goes in `app/code/`
6. ❌ Business logic in templates → Move to Block or ViewModel
7. ❌ Missing `declare(strict_types=1)` → PSR-12 violation
8. ❌ Tabs instead of spaces → Use 4 spaces (Magento standard)
9. ❌ Forgetting to run `cache:flush` → Layout changes don't appear
10. ❌ Committing generated files from `var/generation/` → These are regenerated by Magento

---

*Magento 2 Backend Developer Course — Topic 03*
