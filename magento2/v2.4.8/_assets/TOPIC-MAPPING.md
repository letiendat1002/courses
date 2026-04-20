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

## 19-Topic Curriculum (Backend Track)

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

### Topic 7: APIs — REST, GraphQL & Webhooks
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

### Topic 07: Data Operations — REST APIs, Import/Export & Cron
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

### Topic 9: Performance, Deployment & Capstone
**Days:** 5  
**Philosophy:** Production readiness — make it fast, ship it, own it

| Day | Topic | Content | Commands |
|-----|-------|---------|----------|
| 1 | Caching | Block cache (getCacheKeyInfo, cacheLifetime), cache tags | `cache:clean`, `cache:flush` |
| 2 | Indexing | Indexer modes (on-save/schedule), mview, reindex strategies | `indexer:reindex` |
| 3 | Profiling | Query logging, Xdebug profiler, built-in profiler | — |
| 4 | Deployment + CI/CD | composer.json, deployment script, GitHub Actions pipeline | — |
| 5 | Capstone | Complete module presentation — all 8 weeks demonstrated | — |

**New Concepts:** Cache hierarchy, indexer modes, deployment strategies, CI/CD  
**Admin Addition:** Admin grid — UI Components for data tables

**Definition of Done (DoD):**
- [ ] Block cache implemented with cache tags
- [ ] At least one indexer in schedule mode
- [ ] Profiler output analyzed, slow query identified
- [ ] Deployment script created + CI pipeline passing
- [ ] **Capstone module** complete with all required components
- [ ] Module presented (5 min walkthrough)

**Assessment:** Block cache (20m) + Indexer (15m) + Profiler (20m) + Deployment (30m) + Capstone (all day 5)

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

---

## Topics Covered — Summary

| Category | Topics |
|----------|--------|
| **Environment** | Docker, Composer, Xdebug (intro), Magento install |
| **Module Development** | registration.php, module.xml, di.xml |
| **Routing & Controllers** | routes.xml, Controller execute(), HTTP methods |
| **Views (Admin)** | Blocks, Templates (PHTML), Layout XML (adminhtml) |
| **Data Layer** | db_schema.xml, Model/ResourceModel/Collection, Data Patch, Schema Patch |
| **Service Contracts** | Api/Data interfaces, Repository pattern, SearchCriteria |
| **Customization** | Plugins (around/before/after), Observers, Events, Preferences, Virtual Types |
| **Admin UI** | Admin routes, menu.xml, system.xml, acl.xml, UI Component grids |
| **APIs** | REST (webapi.xml), GraphQL (resolver, mutation), Webhooks, Token/OAuth auth |
| **Data Operations** | Import/Export framework, Cron, Message queues |
| **Performance** | Cache types, cache tags, Indexer modes, Profiling |
| **Quality** | PHPCS, PSR-12, pre-commit hooks |
| **Deployment** | Composer install, CI/CD, zero-downtime deploy |
| **Editions** | Adobe Commerce EE vs Open Source, B2B features, Cloud vs on-prem |
| **Headless GraphQL** | Custom GraphQL resolvers, schema design, mutations, caching |
| **CMS & Extensions** | CMS blocks/pages, widgets, Page Builder, extension attributes |

---

## Topics NOT Covered (Out of Scope)

| Category | Why Excluded |
|----------|-------------|
| Theme development (storefront) | Storefront frontend |
| LESS/CSS compilation (storefront) | Storefront frontend |
| Knockout.js (storefront) | Storefront frontend |
| RequireJS modules (storefront) | Storefront frontend |
| jQuery widgets (storefront) | Storefront frontend |
| Checkout customization | Advanced — optional post-course |
| Varnish full configuration | Advanced — optional post-course |
| Elasticsearch | Advanced — optional post-course |

---

## Admin Topics — Where They Appear

| Topic | Admin Topic | File | Purpose |
|------|-------------|------|---------|
| 01 | Debug/Logs | var/log/*.log | Reading system logs |
| 04 | Admin Routes | etc/adminhtml/routes.xml | Backend URL routing |
| 06 | Admin Menu | etc/adminhtml/menu.xml | Navigation sidebar |
| 06 | System Config | etc/adminhtml/system.xml | Settings pages |
| 06 | ACL | etc/acl.xml + roles | Permission system |
| 06 | Admin Grid | view/adminhtml/ui_component/*.xml | Data tables |
| 08 | ACL for API | etc/acl.xml | API access control |
| 09 | Admin Grid (capstone) | ui_component/*.xml | Capstone requirement |

---

## Topic Weight (Balanced)

| Topic | Lines | Content Density |
|------|-------|----------------|
| 01 | ~550 | Medium — environment setup takes time |
| 02 | ~450 | Medium — standard routing/content |
| 03 | ~530 | Medium — data layer but well-structured |
| 04 | ~560 | Medium — 3 plugin types + observers |
| 05 | ~690 | High — Admin UI is dense |
| 06 | ~640 | Medium — APIs well-covered |
| 07 | ~580 | Medium — import/export + APIs + cron |
| 08 | ~480 | Medium — deployment + capstone |
| **Total** | **~4,500** | **Balanced** |

---

## Optional Post-Course Modules

*For interns who finish early or want to go deeper:*

| Module | Description |
|--------|-------------|
| [Email Templates](OPTIONAL-module-packaging/README.md) | Transactional emails, CSS inlining |
| [Module Packaging](OPTIONAL-module-packaging/README.md) | composer.json, SemVer, CI/CD |
| [Unit Testing](OPTIONAL-unit-testing/README.md) | PHPUnit, integration tests |

---

*Document Version: 6.0*  
*Last Updated: 2026-04-20*
