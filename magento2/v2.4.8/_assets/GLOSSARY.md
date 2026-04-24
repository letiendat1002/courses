# Magento 2 Developer Course — Glossary

## Purpose

This glossary ensures consistent terminology throughout the course. Each term is defined in simple English with practical examples.

---

## How to Use

For each topic, add new terms to this master glossary. Use the format:

```
Term: [word]
Definition: [simple explanation]
Example: [code snippet or practical use]
```

---

## Topic 01: Development Environment

| Term | Definition | Example |
|------|------------|---------|
| **Magento** | An e-commerce platform written in PHP | Used by online stores worldwide |
| **Module** | A package of code that extends Magento functionality | `Training_HelloWorld` module |
| **registration.php** | File that registers a module with Magento | `ComponentRegistrar::register()` |
| **module.xml** | Declares module name and version | `<module name="Vendor_Module" setup_version="1.0.0"/>` |
| **Docker** | Containerization tool for consistent development environments | `docker-compose up -d` |
| **Composer** | PHP package manager for dependencies | `composer require vendor/package` |
| **Xdebug** | PHP extension for debugging code | Set breakpoints in VS Code |
| **Logger** | System for writing messages to log files | `$this->logger->info('message')` |
| **bin/magento** | Magento CLI tool for commands | `bin/magento cache:flush` |

---

## Topic 02: Architecture

| Term | Definition | Example |
|------|------------|---------|
| **Request Flow** | How Magento routes a URL to a controller | Front controller → router → controller |
| **Area** | Part of Magento with its own routing/configuration | `frontend`, `adminhtml`, `graphql` |
| **ObjectManager** | Magento's DI container that instantiates classes | Directly in controllers only |
| **di.xml** | Dependency injection configuration file | `<type name="..."><plugin name="..."/></type>` |
| **Virtual Type** | Subclass injected via di.xml configuration | Custom repository extending a base class |
| **Preference** | Class replacement via di.xml | Replace a core model with a custom one |
| **Plugin** | Modifies behavior of public methods | Around/before/after modifier |
| **PHPCS** | PHP Code Sniffer for coding standards | `vendor/bin/phpcs app/code/` |

---

## Topic 03: Module Development

| Term | Definition | Example |
|------|------------|---------|
| **Controller** | Class that handles URL requests | `Controller/Index/Index.php` |
| **Router** | Maps URLs to controllers | Standard router in Magento |
| **Action** | Method in controller that handles request | `execute()` method |
| **Block** | PHP class that provides data to templates | `Block/Index.php` |
| **Template** | PHTML file that renders HTML | `view/frontend/templates/index.phtml` |
| **Layout** | XML file that defines page structure | `default.xml`, `index_index.xml` |
| **Handle** | Unique identifier for a page configuration | `default`, `catalog_product_view` |
| **Container** | Layout element that holds blocks | `<container name="content">` |
| **ACL** | Access Control List — permission system | `acl.xml` defines resources |

---

## Topic 04: Data Layer

| Term | Definition | Example |
|------|------------|---------|
| **Model** | Class that represents data entity | `Model/Review.php` extends `AbstractModel` |
| **Resource Model** | Class that interacts with database | `Model/ResourceModel/Review.php` |
| **Collection** | Class that handles sets of model instances | `Model/ResourceModel/Review/Collection` |
| **db_schema.xml** | File that declares database table structure | Creates `training_product_review` table |
| **Data Patch** | Class that adds data to database | `Setup/Patch/Data/AddReviews.php` |
| **Schema Patch** | Class that modifies database structure | `Setup/Patch/Schema/AddColumn.php` |
| **EAV** | Entity-Attribute-Value - flexible data model | Product attributes system |
| **Service Contract** | Interface defining data access API | Repository interfaces in `Api/` |
| **Repository** | Class that manages data access (CRUD) | `Model/ReviewRepository.php` |
| **Search Criteria** | Object for filtering and sorting queries | `SearchCriteriaBuilder` |
| **Filter Group** | Groups multiple filters together | Multiple conditions in API calls |

---

## Topic 05: Customization

| Term | Definition | Example |
|------|------------|---------|
| **Plugin** | Modifies behavior of public methods | Around/before/after modifier |
| **Around Plugin** | Wraps original method completely | `aroundExecute()` |
| **Before Plugin** | Runs before original method | `beforeExecute()` |
| **After Plugin** | Runs after original method | `afterExecute()` |
| **Observer** | Responds to events dispatched in Magento | Listens for `catalog_product_save_after` |
| **Event** | Signal that something happened in the system | `customer_register_success` |
| **Preference** | Concrete class replacement via di.xml | Replace a core model with custom implementation |
| **Virtual Type** | Subclass injected via di.xml configuration | Custom logger with environment-specific config |

---

## Topic 06: Admin UI

| Term | Definition | Example |
|------|------------|---------|
| **Admin Route** | URL path for admin controllers | `/admin/review/index/index` |
| **Menu** | Admin sidebar navigation | `menu.xml` defines item hierarchy |
| **System Config** | Stores → Configuration page | `system.xml` defines fields |
| **ACL** | Access Control List for admin resources | `acl.xml` defines permissions |
| **UI Component** | JavaScript-based admin UI elements | Grid, form, file uploader |
| **DataProvider** | Provides data to UI Component grids | `Magento\Ui\DataProvider\DataProviderInterface` |
| **CSRF** | Cross-Site Request Forgery protection | Form tokens in admin |
| **_isAllowed()** | Controller method that checks ACL | Must call `parent::_isAllowed()` |

---

## Topic 07: Customer Account

| Term | Definition | Example |
|------|------------|---------|
| **Customer EAV** | EAV attributes for customer entities | Custom attributes in `customer_attribute` table |
| **Customer Session** | Persistent customer data across requests | `$customerSession->getCustomer()` |
| **Registration** | Customer account creation flow | `customer_account_create` event |
| **Loyalty Program** | Reward points and customer incentives | Configurable point rules |
| **Customer Segment** | Dynamic customer grouping | Rules-based segment membership |

---

## Topic 07: API Development

| Term | Definition | Example |
|------|------------|---------|
| **REST API** | HTTP-based API for communication | `/rest/V1/products` |
| **GraphQL** | Query language for APIs | `/graphql` endpoint |
| **OAuth** | Authentication protocol for APIs | Token-based auth |
| **Webhooks** | HTTP callbacks for events | Outgoing notifications |
| **API Endpoint** | URL that provides API access | `/rest/default/V1/customers` |
| **Swagger/OpenAPI** | API documentation format | Generated API docs |
| **webapi.xml** | WebAPI route configuration | `<route method="GET" url="/V1/products">` |
| **Rate Limiting** | API request throttling | Per-user or per-IP limits |

---

## Topic 08: Data Operations

| Term | Definition | Example |
|------|------------|---------|
| **Import** | Process of loading data into Magento | Import products from CSV |
| **Export** | Process of extracting data from Magento | Export orders to CSV |
| **CSV** | Comma-Separated Values file format | `product_import.csv` |
| **Cron** | Scheduled task system in Magento | Runs every minute |
| **Job** | Single cron task | `catalog_product_alert` |
| **Message Queue** | System for async processing | RabbitMQ integration |

---

## Topic 15: Performance

| Term | Definition | Example |
|------|------------|---------|
| **Cache** | Stored data for fast access | Full page cache |
| **Indexer** | Process that updates catalog data | Product indexer |
| **Varnish** | HTTP caching proxy | Built-in full page cache |
| **Redis** | In-memory data store | Cache and session storage |
| **Profiler** | Tool for measuring performance | Built-in Magento profiler |
| **Full Page Cache** | Caches entire rendered pages | Built-in FPC or Varnish |
| **Zero-Downtime Deploy** | Deployment without site outage | Database migrations designed for it |
| **CI/CD** | Continuous Integration / Deployment | Automated testing and deployment pipeline |

---

## Common Terms Reference

### Magento-Specific

| Term | Meaning |
|------|---------|
| `app/code` | Directory for custom modules |
| `app/design` | Directory for themes |
| `pub/static` | Generated static files |
| `var/` | Generated files and logs |
| `vendor/` | Composer dependencies |
| `etc/` | Configuration directory |
| `Controller/` | Request handling classes |
| `Block/` | Logic classes for templates |
| `Model/` | Data classes |
| `View/` | Template and layout files |

### Development

| Term | Meaning |
|------|---------|
| **CLI** | Command Line Interface |
| **CRUD** | Create, Read, Update, Delete |
| **OOP** | Object-Oriented Programming |
| **MVC** | Model-View-Controller pattern |
| **DTO** | Data Transfer Object |
| **API** | Application Programming Interface |

---

## Adding New Terms

When adding new content:

1. **Define first** — Use simple English
2. **Provide example** — Code snippet or use case
3. **Add to glossary** — Update this file
4. **Be consistent** — Use same term throughout

---

*Document Version: 2.0*
*Last Updated: 2026-04-24*