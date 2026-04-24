# Magento 2 Backend Developer Course — Topic Mapping

## Prerequisite Skills (Before Course Starts)

| Skill | Level Required | Verification |
|-------|---------------|--------------|
| PHP OOP | Classes, interfaces, inheritance | Can write a simple class with constructor injection |
| JavaScript | Variables, functions, callbacks | Can write a function; AJAX calls conceptually familiar |
| Git | add, commit, push, pull, branch, merge | Comfortable with branching and PRs |
| SQL | SELECT, INSERT, basic JOINs | Write simple queries against a single table |

**Note:** No prior Magento experience required. This course starts from zero.

---

## 21-Topic Curriculum (Backend Track)

---

### Topic 01: Development Environment & Magento Foundations
**Days:** 5
**Philosophy:** Mental model first, tool setup second

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | What is Magento? | Ecosystem, editions, request flow | — |
| 2 | Directory Structure | app/code, module anatomy, config files | — |
| 3 | Module Fundamentals | registration.php, module.xml, first route | `module:enable`, `setup:upgrade` |
| 4 | Docker Setup | docker-compose, Magento install, volume mounts | `docker compose up -d` |
| 5 | bin/magento CLI + Logging | Essential commands, log reading, module workflow | `cache:flush`, `dev:log` |

**New Concepts:** Magento architecture, module skeleton, development environment
**Admin Addition:** Log reading — `var/log/system.log`

**Definition of Done (DoD):**
- [ ] Docker environment running Magento
- [ ] `Training_HelloWorld` module enabled
- [ ] Route `helloworld/index/index` responds with page
- [ ] Custom log written to `var/log/`

**Assessment:** Docker setup (30m) + Module creation (20m) + Route test (15m) + Logging (10m)

---

### Topic 02: Controllers, Blocks, Templates & Layout XML
**Days:** 5
**Philosophy:** Build the request-response chain

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | Request Flow | How Magento routes URL to controller | — |
| 2 | Controllers + Routing | routes.xml, HttpGetActionInterface, HttpPostActionInterface | `router:list` |
| 3 | Blocks + Templates | Block logic, PHTML, $block->escapeHtml() | — |
| 4 | Layout XML | Handles, containers, references, arguments | `cache:flush` |
| 5 | PHPCS + Code Quality | PSR-12, Magento coding standard, pre-commit hook | `vendor/bin/phpcs` |

**New Concepts:** MVC in Magento, layout XML processing, code quality gates
**Admin Addition:** Admin routes (etc/adminhtml/routes.xml) → covered in Topic 6

**Definition of Done (DoD):**
- [ ] Working route with custom controller
- [ ] Custom block passes data to template
- [ ] Layout XML positions elements without modifying core
- [ ] PHPCS reports zero errors

**Assessment:** Routing test (20m) + Block/Template (20m) + PHPCS (15m)

---

### Topic 03: Data Layer — Models, Database, Service Contracts & Repositories
**Days:** 5
**Philosophy:** Understand data persistence end-to-end

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | Declarative Schema | db_schema.xml, generating whitelist | `setup:db-declaration:generate-whitelist` |
| 2 | Data & Schema Patches | Data Patch for initial data, Schema Patch for modifications | `setup:upgrade` |
| 3 | Model/ResourceModel/Collection | Triad pattern, CRUD operations | — |
| 4 | Service Contracts | Api/Data interfaces, interface-first design | — |
| 5 | Repositories + SearchCriteria | Repository pattern, filtering, controller integration | `setup:di:compile` |

**New Concepts:** Declarative schema (db_schema.xml), patch system, repository pattern
**Admin Addition:** EAV attributes — extending product/customer attributes

**Definition of Done (DoD):**
- [ ] Custom db_schema.xml creates table without errors
- [ ] Schema whitelist generated
- [ ] Data Patch inserts sample records, verified in DB
- [ ] Schema Patch modifies existing column
- [ ] ReviewRepositoryInterface + ReviewInterface defined in Api/
- [ ] Controller uses repository (not direct model)
- [ ] `bin/magento setup:db:status` shows module schema as current

**Assessment:** Schema creation (30m) + Patches (20m) + Service Contracts (45m) + SearchCriteria (30m)

---

### Topic 04: Plugins, Observers & Dependency Injection
**Days:** 5
**Philosophy:** Customize Magento without touching core code

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | After Plugins | Modify return values after method runs | `di:compile` |
| 2 | Before Plugins | Validate/change input before method runs | — |
| 3 | Around Plugins | Wrap method logic, conditionally skip | — |
| 4 | Observers & Events | Event dispatch, observer registration, custom events | — |
| 5 | di.xml Advanced | Preferences, virtual types, sort order | `setup:di:compile` |

**New Concepts:** Plugin interception (before/after/around), event-driven architecture
**Admin Addition:** Admin routing (di.xml area configuration)

**Definition of Done (DoD):**
- [ ] After plugin modifies ProductRepositoryInterface::save return value
- [ ] Before plugin validates SKU (not empty) and price (not negative)
- [ ] Around plugin wraps getById with timing/logic
- [ ] Custom event dispatched from controller/service
- [ ] Observer responds to dispatched event
- [ ] Plugin sort order configured for multiple plugins
- [ ] Virtual type or preference configured in di.xml

**Assessment:** After Plugin (20m) + Before Plugin (20m) + Around Plugin (25m) + Observer (20m) + Sort Order (10m)

---

### Topic 05: Admin UI — Routes, Menus, Configuration, ACL & Grids
**Days:** 5
**Philosophy:** Build the Admin Interface — every module needs a home in admin

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | Admin Routes | routes.xml (admin area), admin controller, _isAllowed() | — |
| 2 | Admin Menu | menu.xml, parent references, menu hierarchy | `admin:menu:rebuild` |
| 3 | System Configuration | system.xml, sections/groups/fields, source models | — |
| 4 | ACL & Roles | acl.xml, resource hierarchy, role assignment | — |
| 5 | UI Component Grids | ui_component listing, DataProvider, action columns | — |

**New Concepts:** Admin routing, sidebar navigation, configuration pages, permissions, admin grids
**This is the Admin week** — no storefront frontend content

**Definition of Done (DoD):**
- [ ] Admin route accessible at `/admin/review/index/index`
- [ ] Menu item visible in admin sidebar
- [ ] Menu item hidden when user lacks ACL permission
- [ ] Configuration page at Stores → Config with ≥3 fields
- [ ] Admin grid renders with data from DB, edit/delete work
- [ ] All controllers protected with `_isAllowed()` ACL checks

**Assessment:** Admin route + menu (20m) + System config (20m) + ACL (20m) + Admin grid (45m)

---

### Topic 06: Customization — Plugins, Observers, Events & Preferences
**Days:** 5
**Philosophy:** Master Magento's interception system

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | Plugin Recap + Sorting | Multi-plugin sort order, around vs before/after | `di:compile` |
| 2 | Advanced Plugins | Changing arguments, conditional execution | — |
| 3 | Observer Deep Dive | Event creation, observer priority, multiple observers | — |
| 4 | Preferences & Type Replacement | Concrete class replacement, interface implementation | — |
| 5 | Virtual Types | Injection via configuration, real-world use cases | `setup:di:compile` |

**New Concepts:** Advanced plugin patterns, preference chains, virtual type inheritance
**Admin Addition:** Admin configuration via di.xml (virtual types for API clients)

**Definition of Done (DoD):**
- [ ] Multiple plugins with correct sort order
- [ ] Around plugin conditionally skipping logic
- [ ] Custom event with multiple observers
- [ ] Preference replacing a core model
- [ ] Virtual type used for environment-specific config

**Assessment:** Plugin sorting (15m) + Advanced plugins (25m) + Observer priority (20m) + Preferences (20m) + Virtual types (20m)

---

### Topic 07: API Development — REST, GraphQL & Webhooks
**Days:** 5
**Philosophy:** Connect Magento to the outside world

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | REST API Architecture | webapi.xml, route to service layer | — |
| 2 | Custom REST Endpoints | GET + POST endpoints, request/response | `api:rest:routes` |
| 3 | Authentication | Token auth, OAuth, integration auth | — |
| 4 | GraphQL | Queries, mutations, resolver, GraphiQL IDE | — |
| 5 | Webhooks + System Config | Outgoing webhooks, queue-based handlers, API + config integration | — |

**New Concepts:** REST, GraphQL, webhook patterns, API authentication
**Admin Addition:** System configuration integration with API keys

**Definition of Done (DoD):**
- [ ] Custom REST endpoint GET /V1/review/{id} returns correct JSON
- [ ] Custom REST endpoint POST /V1/review validates and returns 201/400
- [ ] Token authentication works end-to-end
- [ ] Custom GraphQL query returns data in GraphiQL
- [ ] Custom GraphQL mutation accepts input and confirms
- [ ] API tested successfully via Postman or cURL

**Assessment:** REST GET (20m) + REST POST (25m) + GraphQL query/mutation (40m) + Webhook (30m)

---

### Topic 08: Data Operations — Import/Export, Cron & Message Queues
**Days:** 5
**Philosophy:** Automate data flow — push data in, pull data out, schedule tasks

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | REST API — webapi.xml | Expose repository via webapi.xml, ACL resources | `setup:upgrade` |
| 2 | REST API — POST + Webhooks | POST endpoint, token auth, webhook pattern | — |
| 3 | Import Framework | Built-in import, CSV format, custom ImportInterface | `import:export:run` |
| 4 | Export Framework + Cron | Custom ExportInterface, crontab.xml, cron groups | `cron:run` |
| 5 | Message Queues + Integration Test | Queue publisher/consumer, full end-to-end test | `queue:consumers:run` |

**New Concepts:** Import/export framework, cron scheduling, message queues, webhooks
**Admin Addition:** ACL for API access control

**Definition of Done (DoD):**
- [ ] REST GET /V1/reviews returns JSON via cURL
- [ ] REST POST /V1/reviews creates review, returns 201
- [ ] Token authentication working end-to-end
- [ ] Custom Import model processes CSV and saves to DB (5+ rows)
- [ ] Custom Export model generates valid CSV
- [ ] Cron job executes, verified in cron_schedule table
- [ ] All APIs tested via Postman/cURL

**Assessment:** REST GET (15m) + REST POST (20m) + Import (30m) + Export (20m) + Cron (15m)

---

### Topic 09: Performance, Indexing & Capstone
**Days:** 5
**Philosophy:** Production readiness — make it fast, own it

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | Caching | Block cache (getCacheKeyInfo, cacheLifetime), cache tags | `cache:clean`, `cache:flush` |
| 2 | Indexing | Indexer modes (on-save/schedule), mview, reindex strategies | `indexer:reindex` |
| 3 | Profiling | Query logging, Xdebug profiler, built-in profiler | — |
| 4 | Advanced Performance | Full-page cache, Redis, Varnish basics | — |
| 5 | Capstone | Complete module presentation — all 9 foundational topics demonstrated | — |

**New Concepts:** Cache hierarchy, indexer modes, profiling, full-page cache
**Admin Addition:** Admin grid — UI Components for data tables

**Definition of Done (DoD):**
- [ ] Block cache implemented with cache tags
- [ ] At least one indexer in schedule mode
- [ ] Profiler output analyzed, slow query identified
- [ ] Full-page cache configured
- [ ] **Capstone module** complete with all required components
- [ ] Module presented (5 min walkthrough)

**Assessment:** Block cache (20m) + Indexer (15m) + Profiler (20m) + Advanced cache (25m) + Capstone (all day 5)

---

### Topic 10: Checkout — Extensibility & Customization
**Days:** 5
**Philosophy:** Master the checkout — Magento's most critical and complex flow

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | Checkout Architecture | Layout, components, section configuration | — |
| 2 | Adding Custom Fields | checkout/index/add, UI component integration | `cache:flush` |
| 3 | Totals Modification | Totals collectors, processors, custom total models | — |
| 4 | Shipping & Payment Integration | Shipping method, payment method extension | — |
| 5 | Place Order Flow | Order placement, authorization, quote management | — |

**New Concepts:** Checkout layout XML, JavaScript components, totals collectors
**Admin Addition:** Order management in admin (Status, Invoice, Ship)

**Definition of Done (DoD):**
- [ ] Custom checkout field saved and displayed
- [ ] Custom total collector registered and applied
- [ ] Custom shipping method visible at checkout
- [ ] Custom payment method available
- [ ] Order placed with custom data persisted

**Assessment:** Checkout field (25m) + Totals (25m) + Shipping (20m) + Payment (25m) + Order flow (15m)

---

### Topic 11: Search & Inventory Management
**Days:** 5
**Philosophy:** Search and inventory — two pillars of commerce

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | Elasticsearch Architecture | Elasticsearch integration, index structure | — |
| 2 | Search Customization | Search adapters, synonym management | — |
| 3 | Inventory Architecture | Stock management, sources, reservations | — |
| 4 | MSI (Multi-Source Inventory) | Source assignment, quantity management | — |
| 5 | Custom Search & Inventory | Custom search resolver, inventory reservation observer | — |

**New Concepts:** Elasticsearch, search adapters, MSI, inventory reservations
**Admin Addition:** Admin order creation respecting inventory

**Definition of Done (DoD):**
- [ ] Elasticsearch index verified and searchable
- [ ] Custom search synonym configured
- [ ] Multiple sources configured in MSI
- [ ] Inventory reservation observed on order
- [ ] Custom search resolver integrated

**Assessment:** ES architecture (20m) + Search customization (25m) + Inventory (25m) + MSI (20m) + Custom resolver (20m)

---

### Topic 12: Storefront JavaScript — UI Components & Knockout.js
**Days:** 5
**Philosophy:** Understand Magento's client-side architecture

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | RequireJS Architecture | Module loading, dependency injection, var/config | — |
| 2 | Knockout.js Fundamentals | Observables, bindings, templates | — |
| 3 | Magento UI Components | Magento's Knockout binding handlers, koVirtualChildren | — |
| 4 | Custom UI Component | Creating a custom ko component, integrating with layout | — |
| 5 | jQuery Widgets | jQuery widget lifecycle, _trigger methods, $.widget factory | — |

**New Concepts:** RequireJS, Knockout.js, UI Components, jQuery widgets
**Prerequisite:** JavaScript basics

**Definition of Done (DoD):**
- [ ] RequireJS module loads with dependencies
- [ ] Knockout observable updates bound element
- [ ] Custom Magento UI Component registered and renders
- [ ] jQuery widget initialized on custom element
- [ ] AJAX call made from Knockout component

**Assessment:** RequireJS (20m) + Knockout (25m) + UI Components (25m) + Custom component (25m) + jQuery (15m)

---

### Topic 13: Layout XML — Advanced Frontend Customization
**Days:** 5
**Philosophy:** Master layout XML to bend Magento's layout to your will

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | Layout Override vs Update | theme vs module layout, handle ordering | `cache:flush` |
| 2 | Containers & Blocks in Layout | Structural links, insert modifiers, move elements | — |
| 3 | Arguments & Data Attributes | Argument types (string, object, action, etc.) | — |
| 4 | Custom Handle & Reference | Creating custom layout handles, referencing core blocks | — |
| 5 | Layout XML + JavaScript | Linking Knockout templates via layout, component layout | — |

**New Concepts:** Layout override chain, argument processing, container manipulation
**Prerequisite:** Topics 02, 12

**Definition of Done (DoD):**
- [ ] Custom module theme layout overrides Luma
- [ ] Block moved to different container via XML
- [ ] Custom argument passed to block
- [ ] Custom layout handle registered
- [ ] Knockout template loaded via layout XML

**Assessment:** Override (20m) + Containers (20m) + Arguments (20m) + Custom handle (20m) + JS integration (20m)

---

### Topic 14: Theming — HTML, CSS & Luma
**Days:** 5
**Philosophy:** Build beautiful storefronts on Magento's theming foundation

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | Theme Architecture | Parent/child themes, theme.xml, registration.php | — |
| 2 | LESS/CSS Customization | LESS variables, mixins, module-level CSS | `cache:flush` |
| 3 | Luma Deep Dive | Luma components, navigation, product listing | — |
| 4 | Custom Theme Creation | New theme extending Luma, full-storefront customization | — |
| 5 | Responsive Design | Mobile-first, breakpoint configuration | — |

**New Concepts:** Theme inheritance, LESS preprocessing, Luma anatomy
**Prerequisite:** Topics 12, 13

**Definition of Done (DoD):**
- [ ] Custom theme registered and active
- [ ] LESS override changes Luma styles
- [ ] Product listing page custom-styled
- [ ] Theme mobile responsive
- [ ] JavaScript widget themed correctly

**Assessment:** Theme setup (20m) + LESS (25m) + Luma anatomy (20m) + Custom theme (25m) + Responsive (15m)

---

### Topic 15: Advanced Storefront — Performance & Personalization
**Days:** 5
**Philosophy:** Take the storefront further — faster, smarter, more personal

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | Customer Segmentation | Customer groups, segment rules | — |
| 2 | Private Sales / Events | Event-based pricing, category access | — |
| 3 | Wishlist & Gift Options | Wishlist customization, gift wrap, gift cards | — |
| 4 | Advanced Cache | FPC holes, ESI, per-customer cache | `cache:flush` |
| 5 | Personalization Integration | Persistent cart, recently viewed, comparison | — |

**New Concepts:** Customer segmentation, private sales, FPC holes, personalization
**Prerequisite:** Topics 09, 12

**Definition of Done (DoD):**
- [ ] Customer segment with dynamic content
- [ ] Private sale event created and restricted
- [ ] Wishlist extended with custom attribute
- [ ] FPC hole punched for personalized content
- [ ] Recently viewed configured

**Assessment:** Segmentation (20m) + Private sales (20m) + Wishlist (20m) + FPC holes (25m) + Personalization (20m)

---

### Topic 16: GraphQL & Headless Architecture
**Days:** 5
**Philosophy:** Decouple — power PWA, mobile apps, and third-party integrations

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | GraphQL Schema Design | Schema types, queries defined in di.xml | — |
| 2 | Custom Resolvers | Resolver class, Context, parent resolver | — |
| 3 | Mutations | Input types, mutations, error handling | — |
| 4 | Caching & Performance | Query caching, schema caching, DataLoader | — |
| 5 | PWA Integration | Magento PWA Studio overview, Apollo Client | — |

**New Concepts:** GraphQL schema, resolvers, mutations, PWA architecture
**Prerequisite:** Topic 07 (API basics)

**Definition of Done (DoD):**
- [ ] Custom GraphQL query added to schema
- [ ] Resolver returns correct data
- [ ] Custom mutation accepting complex input
- [ ] Query caching enabled
- [ ] PWA storefront consumes custom endpoint

**Assessment:** Schema design (20m) + Resolvers (25m) + Mutations (25m) + Caching (20m) + PWA (15m)

---

### Topic 17: CMS & Extensions
**Days:** 5
**Philosophy:** Extend content management and build reusable extensions

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | CMS Blocks & Pages | WYSIWYG, static blocks, hierarchy | — |
| 2 | CMS Widgets | Widget creation, template assignment | — |
| 3 | Page Builder | Page Builder layouts, custom content types | — |
| 4 | Extension Attributes | Adding fields to entity without EAV | — |
| 5 | Custom CMS Functionality | Custom CMS page controller, routing | — |

**New Concepts:** CMS blocks, widgets, Page Builder, extension attributes
**Admin Addition:** CMS page creation, widget placement

**Definition of Done (DoD):**
- [ ] CMS block with widget embedded
- [ ] Custom widget registered and placed
- [ ] Page Builder custom content type
- [ ] Extension attribute added to product
- [ ] Custom CMS route responds correctly

**Assessment:** CMS blocks (15m) + Widgets (20m) + Page Builder (25m) + Extension attrs (25m) + Custom CMS (15m)

---

### Topic 18: Security & Adobe Commerce Hardening
**Days:** 5
**Philosophy:** Secure your code, harden your environment

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | Security Fundamentals | XSS, CSRF, SQL injection, LFI | — |
| 2 | ACL & Permissions | Role-based access, resource ownership | — |
| 3 | Encryption & Secrets | Cryptographic modules, env.php secrets | — |
| 4 | Adobe Commerce Security | Two-factor auth, reCAPTCHA, patch management | — |
| 5 | Security Audit | Code scanning, penetration testing basics | — |

**New Concepts:** Security vulnerabilities, Adobe Commerce features, audit tools
**Admin Addition:** Admin user management, session security

**Definition of Done (DoD):**
- [ ] XSS vulnerability identified and fixed in custom module
- [ ] CSRF token verified in form
- [ ] Sensitive data encrypted with Magento's encryption
- [ ] reCAPTCHA integrated on custom form
- [ ] Security audit performed on capstone module

**Assessment:** Security fundamentals (25m) + ACL (20m) + Encryption (20m) + Adobe Commerce features (20m) + Audit (20m)

---

### Topic 19: Production Debugging & Deployment
**Days:** 5
**Philosophy:** Survive production — debug fast, deploy safely

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | Production Logging | Monolog configuration, log rotation | — |
| 2 | Xdebug in Production | Remote debugging, step debugging | — |
| 3 | Deployment Strategies | Zero-downtime, rolling deploys, rollbacks | — |
| 4 | Monitoring & Alerting | New Relic, health checks, alerting | — |
| 5 | Incident Response | Runbooks, post-mortem, on-call | — |

**New Concepts:** Production debugging, deployment pipelines, monitoring
**Admin Addition:** Admin error monitoring, log aggregation

**Definition of Done (DoD):**
- [ ] Custom monolog handler configured
- [ ] Xdebug connected to production environment
- [ ] Deployment script tested and documented
- [ ] Monitoring dashboard created
- [ ] Runbook written for capstone module

**Assessment:** Production logging (20m) + Xdebug (20m) + Deployment (25m) + Monitoring (20m) + Incident response (20m)

---

### Topic 20: Module Packaging & Distribution
**Days:** 5
**Philosophy:** Package your module for the world — composer, versioning, marketplace

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | Composer Fundamentals | composer.json, autoload, versioning | — |
| 2 | Module Metadata | module.xml, registration.php, package naming | — |
| 3 | SemVer & Releases | Semantic versioning, changelog, Git tagging | — |
| 4 | Marketplace Submission | Commerce Marketplace requirements, review process | — |
| 5 | Email Templates | Transactional email CSS, inline styles, preview | — |

**New Concepts:** Composer packaging, SemVer, Marketplace submission, email templates
**Note:** Email Templates merged into this topic for efficient packaging coverage

**Definition of Done (DoD):**
- [ ] composer.json passes validation
- [ ] Module installable via composer
- [ ] SemVer tags applied correctly
- [ ] Marketplace submission drafted
- [ ] Custom email template sent and styled

**Assessment:** Composer (20m) + Metadata (15m) + SemVer (20m) + Marketplace (20m) + Email templates (25m)

---

### Topic 21: Unit Testing & Quality Assurance
**Days:** 5
**Philosophy:** Test with confidence — unit tests that catch bugs before they catch you

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | PHPUnit Fundamentals | Assertions, data providers, annotations | `vendor/bin/phpunit` |
| 2 | Integration Tests | Integration test framework, DB transaction isolation | — |
| 3 | Mocking & Fixtures | Object Manager, mock objects, fixture factories | — |
| 4 | Test-Driven Development | Red-green-refactor, writing tests before code | — |
| 5 | Quality Gates | Code coverage, static tests, CI integration | — |

**New Concepts:** PHPUnit, integration test framework, mocking, TDD
**Admin Addition:** Testing admin functionality

**Definition of Done (DoD):**
- [ ] Unit test covers plugin logic
- [ ] Integration test verifies repository behavior
- [ ] Mock object used to isolate unit under test
- [ ] TDD cycle completed on new feature
- [ ] Coverage report generated

**Assessment:** PHPUnit fundamentals (20m) + Integration tests (25m) + Mocking (25m) + TDD (20m) + Quality gates (15m)

---

## Must-Know bin/magento Commands

| Topic | Commands |
|------|----------|
| 01 | `module:enable`, `setup:upgrade`, `dev:log`, `cache:flush` |
| 02 | `cache:flush`, `dev:di:compile`, `router:list` |
| 03 | `setup:db-declaration:generate-whitelist`, `setup:db:status`, `setup:upgrade`, `setup:di:compile` |
| 04 | `di:compile`, `cache:flush` |
| 05 | `admin:menu:rebuild`, `admin:routes:list` |
| 06 | `api:rest:routes`, `cache:flush` |
| 07 | `import:export:run`, `cron:run`, `cron:list`, `indexer:reindex` |
| 08 | `cache:clean`, `cache:flush`, `indexer:reindex`, `deploy:mode:set developer` |
| 09 | `cache:flush`, `indexer:reindex`, `dev:profiler` |
| 10 | `cache:flush` |
| 11 | `indexer:reindex`, `inventory:reservation:list` |
| 12 | `cache:flush` |
| 13 | `cache:flush` |
| 14 | `cache:flush`, `setup:static-content:deploy` |
| 15 | `cache:flush` |
| 16 | `cache:flush` |
| 17 | `cache:flush` |
| 18 | `cache:flush` |
| 19 | `cache:flush`, `deploy:mode:set production` |
| 20 | `cache:flush` |
| 21 | `vendor/bin/phpunit`, `setup:di:compile` |

---

## Topics Covered — Summary

| Category | Topics |
|----------|--------|
| **Environment** | Docker, Composer, Xdebug (intro), Magento install |
| **Module Development** | registration.php, module.xml, di.xml |
| **Routing & Controllers** | routes.xml, Controller execute(), HTTP methods |
| **Views** | Blocks, Templates (PHTML), Layout XML |
| **Data Layer** | db_schema.xml, Model/ResourceModel/Collection, Data Patch, Schema Patch |
| **Service Contracts** | Api/Data interfaces, Repository pattern, SearchCriteria |
| **Customization** | Plugins (around/before/after), Observers, Events, Preferences, Virtual Types |
| **Admin UI** | Admin routes, menu.xml, system.xml, acl.xml, UI Component grids |
| **APIs** | REST (webapi.xml), GraphQL (resolver, mutation), Webhooks, Token/OAuth auth |
| **Data Operations** | Import/Export framework, Cron, Message queues |
| **Performance** | Cache types, cache tags, Indexer modes, Profiling, FPC, Redis |
| **Checkout** | Checkout architecture, custom fields, totals, shipping, payment |
| **Search & Inventory** | Elasticsearch, MSI, inventory reservations |
| **Storefront JS** | RequireJS, Knockout.js, UI Components, jQuery widgets |
| **Layout XML** | Layout overrides, containers, arguments, custom handles |
| **Theming** | Theme inheritance, LESS/CSS, Luma, responsive design |
| **Advanced Storefront** | Customer segmentation, private sales, FPC holes, personalization |
| **Quality** | PHPCS, PSR-12, pre-commit hooks, code coverage |
| **Deployment** | Composer install, CI/CD, zero-downtime deploy |
| **Editions** | Adobe Commerce EE vs Open Source, B2B features, Cloud vs on-prem |
| **Headless GraphQL** | Custom GraphQL resolvers, schema design, mutations, caching, PWA |
| **CMS & Extensions** | CMS blocks/pages, widgets, Page Builder, extension attributes |
| **Security** | XSS, CSRF, SQL injection, ACL, encryption, Adobe Commerce hardening |
| **Production** | Logging, debugging, monitoring, incident response |
| **Packaging** | Composer, SemVer, Marketplace, email templates |
| **Testing** | PHPUnit, integration tests, mocking, TDD |

---

## Topics NOT Covered (Out of Scope)

| Category | Why Excluded |
|----------|-------------|
| Varnish full configuration | Advanced — optional post-course |
| Full PWA Studio development | Separate specialized track |
| Magento Cloud infrastructure | DevOps specialized track |
| B2B advanced features | Adobe Commerce specialized |

---

## Admin Topics — Where They Appear

| Topic | Admin Topic | File | Purpose |
|------|-------------|------|---------|
| 01 | Debug/Logs | var/log/*.log | Reading system logs |
| 05 | Admin Routes | etc/adminhtml/routes.xml | Backend URL routing |
| 05 | Admin Menu | etc/adminhtml/menu.xml | Navigation sidebar |
| 05 | System Config | etc/adminhtml/system.xml | Settings pages |
| 05 | ACL | etc/acl.xml + roles | Permission system |
| 05 | Admin Grid | view/adminhtml/ui_component/*.xml | Data tables |
| 08 | ACL for API | etc/acl.xml | API access control |
| 09 | Admin Grid (capstone) | ui_component/*.xml | Capstone requirement |
| 10 | Order Management | Sales orders in admin | Order status, invoice, ship |
| 11 | Admin Order Creation | Create order respecting inventory | Stock management |
| 17 | CMS Page Creation | CMS pages in admin | Content editing |
| 17 | Widget Placement | Widget instance placement | Admin widget placement |
| 18 | Admin User Management | System → Permissions → Users | Session security |
| 19 | Admin Error Monitoring | System → Logs | Log aggregation |
| 21 | Testing Admin Functionality | Admin controllers tested via integration | Access control tested |

---

## Topic Weight (Balanced)

| Topic | Lines | Content Density |
|------|-------|----------------|
| 01 | ~550 | Medium — environment setup takes time |
| 02 | ~450 | Medium — standard routing/content |
| 03 | ~530 | Medium — data layer but well-structured |
| 04 | ~560 | Medium — 3 plugin types + observers |
| 05 | ~690 | High — Admin UI is dense |
| 06 | ~500 | Medium — advanced customization |
| 07 | ~640 | Medium — APIs well-covered |
| 08 | ~580 | Medium — import/export + APIs + cron |
| 09 | ~520 | Medium — performance + capstone |
| 10 | ~550 | High — checkout is complex |
| 11 | ~500 | Medium — search + inventory |
| 12 | ~550 | Medium — JS storefront architecture |
| 13 | ~480 | Medium — layout XML |
| 14 | ~520 | Medium — theming + LESS |
| 15 | ~500 | Medium — personalization |
| 16 | ~560 | Medium — GraphQL + headless |
| 17 | ~500 | Medium — CMS + extensions |
| 18 | ~480 | Medium — security |
| 19 | ~460 | Medium — production |
| 20 | ~490 | Medium — packaging |
| 21 | ~520 | Medium — testing |
| **Total** | **~11,150** | **Balanced** |

---

## Optional Post-Course Modules

*For interns who finish early or want to go deeper:*

| Module | Description |
|--------|-------------|
| [Module Packaging](20-module-packaging/README.md) | Already covered in Topic 20 — composer.json, SemVer, CI/CD |
| [Unit Testing](21-unit-testing/README.md) | Already covered in Topic 21 — PHPUnit, integration tests |

---

*Document Version: 7.0*
*Last Updated: 2026-04-24*
*21-Topic Structure*