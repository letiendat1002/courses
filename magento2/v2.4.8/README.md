# Magento 2 Backend Developer Course

**Focus:** Backend Headless Development | **Version:** 2.4.8 (LTS)

---

## Course Overview

A comprehensive, self-paced course covering Magento 2 backend development at a middle-engineer level. No time limits — learn at your own pace, master each topic deeply.

**Target:** Developers with PHP OOP basics who want to become proficient Magento 2 backend engineers.

**Out of Scope:** Storefront frontend (themes, LESS, Knockout.js, PWA). This course focuses exclusively on Magento backend for headless/API-first architectures.

---

## Learning Path

The course is organized into **19 topics**, progressing from fundamentals to advanced patterns. Complete them in order for structured learning, or jump to specific topics as needed.

| # | Topic | Description |
|---|-------|-------------|
| 01 | [Development Environment](./01-dev-environment/README.md) | Docker, Composer, Magento CLI, module registration, PSR-12 standards, development tools |
| 02 | [Architecture](./02-architecture/README.md) | Request flow, area routing, service contracts, DI container, plugin architecture, event system |
| 03 | [Module Development](./03-module-development/README.md) | Controller lifecycle, block/view models, layout XML, routing, frontend ACL |
| 04 | [Data Layer](./04-data-layer/README.md) | Models/ResourceModel/Collection, repositories, SearchCriteria, EAV, declarative schema, data patches |
| 05 | [Customization](./05-customization/README.md) | Plugins (before/after/around), observers, di.xml preferences, virtual types, sort order calculation |
| 06 | [Admin UI](./06-admin-ui/README.md) | Admin routing, menus, ACL/CSRF security, UI Components, DataProviders, form validation |
| 07 | [API Development](./07-api-development/README.md) | REST/GraphQL endpoints, webapi.xml, authentication, rate limiting, batch operations |
| 08 | [Data Operations](./08-data-ops/README.md) | Import/export, indexer management, cron schedules, message queues, bulk operations |
| 09 | [Performance](./09-performance/README.md) | Full-page cache, block cache, Redis caching, query optimization, profiling tools |
| 10 | [Checkout](./10-checkout/README.md) | Quote management, inventory reservation, shipping carriers, payment methods, order state machine |
| 11 | [Email Templates](./11-email-templates/README.md) | Email template architecture, HTML emails, CSS inlining, template variables, admin configuration |
| 12 | [Unit Testing](./12-unit-testing/README.md) | PHPUnit, unit/integration tests, fixtures, mocking Magento classes, test isolation |
| 13 | [Module Packaging](./13-module-packaging/README.md) | composer.json, semantic versioning, metapackage, marketplace submission |
| 14 | [Advanced Performance](./14-advanced-performance/README.md) | Varnish, Elasticsearch, production profiling, New Relic/Blackfire, latency optimization |
| 15 | [Sales & Orders](./15-sales-orders/README.md) | Order creation lifecycle, invoices, shipments, credit memos, order status management |
| 16 | [Customer Account](./16-customer-account/README.md) | Customer EAV, registration flow, account management, loyalty programs |
| 17 | [Adobe Commerce](./17-adobe-commerce/README.md) | CE vs EE vs Cloud, B2B features, edition decision framework |
| 18 | [GraphQL Headless](./18-graphql-headless/README.md) | Custom GraphQL resolvers, schema design, mutations, caching, security |
| 19 | [CMS & Extension Attributes](./19-cms-widgets-ext/README.md) | CMS blocks/pages, widgets, Page Builder, extension attributes |

---

## Topic Dependencies

```
01-dev-environment
└── 02-architecture
    └── 03-module-development ─────────────────────┐
         └── 04-data-layer                         │
              └── 05-customization ───────────────┤
                   └── 06-admin-ui                │
                        └── 07-api-development    │
                             └── 08-data-ops     │
                                  └── 09-performance

10-checkout (parallel track)
├── requires: 04-data-layer, 05-customization
└── leads to: 15-sales-orders

11-email-templates, 12-unit-testing, 13-module-packaging
├── can be studied at any point after 05-customization
└── recommended after completing core topics

14-advanced-performance
└── requires: 09-performance

15-sales-orders
├── requires: 04-data-layer, 05-customization
└── optional: 10-checkout

16-customer-account
└── requires: 04-data-layer
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
for i in {01..19}; do
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

*Last Updated: 2026-04-20*