# Magento 2 Backend Developer Course

**Focus:** Backend Headless Development | **Version:** 2.4.8 (LTS)

---

## Course Overview

A comprehensive, self-paced course covering Magento 2 backend development at a middle-engineer level. No time limits — learn at your own pace, master each topic deeply.

**Target:** Developers with PHP OOP basics who want to become proficient Magento 2 backend engineers.

**Out of Scope:** Storefront frontend (themes, LESS, Knockout.js, PWA). This course focuses exclusively on Magento backend for headless/API-first architectures.

---

## Learning Path

The course is organized into **21 topics**, progressing from fundamentals to advanced patterns. Complete them in order for structured learning, or jump to specific topics as needed.

| # | Topic | Directory | Description |
|---|-------|-----------|-------------|
| 01 | Development Environment | `01-dev-environment/` | Docker, Composer, Magento CLI, module registration, PSR-12 standards, development tools |
| 02 | Architecture | `02-architecture/` | Request flow, area routing, service contracts, DI container, plugin architecture, event system |
| 03 | Module Development | `03-module-development/` | Controller lifecycle, block/view models, layout XML, routing, frontend ACL |
| 04 | Data Layer | `04-data-layer/` | Models/ResourceModel/Collection, repositories, SearchCriteria, EAV, declarative schema, data patches |
| 05 | Customization | `05-customization/` | Plugins (before/after/around), observers, di.xml preferences, virtual types, sort order calculation |
| 06 | Admin UI | `06-admin-ui/` | Admin routing, menus, ACL/CSRF security, UI Components, DataProviders, form validation |
| 07 | API Development | `07-api-development/` | REST/GraphQL endpoints, webapi.xml, authentication, rate limiting, batch operations |
| 08 | Data Operations | `08-data-ops/` | Import/export, indexer management, cron schedules, message queues, bulk operations |
| 09 | Performance | `09-performance/` | Full-page cache, block cache, Redis caching, query optimization, profiling tools, zero-downtime deploy |
| 10 | Checkout | `10-checkout/` | Quote management, inventory reservation, shipping carriers, payment methods, order state machine |
| 11 | Search & Inventory | `11-search-inventory/` | Elasticsearch architecture, query building, catalog search, Multi-Source Inventory (MSI) |
| 12 | Storefront JavaScript | `12-storefront-js/` | KnockoutJS bindings, RequireJS modules/mixins, jQuery widgets, UI Components on frontend |
| 13 | Layout XML & Templates | `13-storefront-layout/` | Frontend layout handles, PHTML templates, frontend ACL, template hints |
| 14 | Theming & LESS | `14-storefront-theming/` | LESS compilation, view.xml, parent-child theme inheritance, CSS output |
| 15 | Advanced Storefront | `15-storefront-advanced/` | Checkout JS customization, payment integration, jQuery widget patterns, third-party themes |
| 16 | GraphQL & Headless | `16-graphql-headless/` | Custom GraphQL resolvers, schema design, mutations, caching, PWA Studio, Hyvä |
| 17 | CMS & Extensions | `17-cms-ext/` | CMS blocks/pages, widgets, Page Builder, extension attributes |
| 18 | Security Hardening | `18-security-hardening/` | SQL injection, CSRF, rate limiting, secret management, 2FA, security headers |
| 19 | Production Debugging | `19-production-debugging/` | Request tracing, debug.log correlation, query logging, slow API workflows, common errors |
| 20 | Module Packaging | `20-module-packaging/` | composer.json, semantic versioning, metapackage, marketplace submission, email templates |
| 21 | Unit Testing | `21-unit-testing/` | PHPUnit, unit/integration tests, fixtures, mocking Magento classes, test isolation |

**Reference (not a topic — CE course focus):**
| | Name | Directory | Description |
|---|---|---|---|
| 🗂 | Adobe Commerce Reference | `_reference/adobe-commerce/` | EE/Cloud/B2B features as reference only |

---

## Topic Dependencies

```
01 → 02 → 03 → 04 → 05
                  ↓
            06-admin-ui → 07-api-dev → 08-data-ops → 09-performance
                                                              ↓
                                  11-search-inventory  18-security  19-production
                                         ↓
                                   16-graphql (depends on 07)

05-customization → 06-admin-ui → 17-cms-ext (depends on 06)

05-customization → 12-storefront-js → 13-layout → 14-theming → 15-advanced
(parallel to backend, same prerequisites)
```

---

## What You'll Build

By completing this course, you will have:

- **Solid Magento 2 fundamentals** — Module development from scratch
- **Data layer mastery** — Declarative schema, repositories, service contracts
- **Customization expertise** — Plugins, observers, dependency injection
- **Admin development skills** — Full admin UI with ACL and data grids
- **API capabilities** — REST endpoints, GraphQL, webhook integration
- **Performance knowledge** — Caching strategies, query optimization, profiling
- **Professional tooling** — Unit testing, CI/CD, module packaging

---

## Prerequisites

- PHP 8.1+ knowledge (OOP, interfaces, traits)
- Basic HTML/CSS understanding
- Git basics (clone, commit, branch, merge)
- SQL fundamentals (SELECT, INSERT, UPDATE, JOIN)
- Composer basics (install, require, autoload)

---

## Quick Start

```bash
# Navigate to course directory
cd magento2/v2.4.8

# Start with Topic 01
cd 01-dev-environment
cat README.md

# Or follow the full learning path sequentially
for i in {01..21}; do
    dir=$(printf "%02d-*" $i)
    echo "=== Studying $(ls -d $dir 2>/dev/null | head -1) ==="
done
```

---

## Reference Materials

- [Magento 2 Architecture](https://developer.adobe.com/commerce/php/architecture/)
- [Magento 2 Development](https://developer.adobe.com/commerce/php/development/)
- [PSR-12 Coding Standard](https://www.php-fig.org/psr/psr-12/)
- [Magento Coding Standard](https://github.com/magento/magento-coding-standard)
- [Magento DevDocs](https://developer.adobe.com/commerce/docs/)

---

## Additional Resources

| Resource | Location |
|----------|----------|
| Glossary | [_assets/GLOSSARY.md](./_assets/GLOSSARY.md) |
| Topic Mapping | [_assets/TOPIC-MAPPING.md](./_assets/TOPIC-MAPPING.md) |

---

*Last Updated: 2026-04-24*