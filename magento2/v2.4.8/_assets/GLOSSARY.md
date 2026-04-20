# Magento 2 Zero To Hero - Glossary Template

## Purpose

This glossary ensures consistent terminology throughout the course. Each term is defined in simple English with practical examples.

---

## How to Use

For each week, add new terms to this master glossary. Use the format:

```
Term: [word]
Definition: [simple explanation]
Example: [code snippet or practical use]
```

---

## Topic 01: Introduction & Setup

| Term | Definition | Example |
|------|------------|---------|
| **Magento** | An e-commerce platform written in PHP | Used by online stores worldwide |
| **Module** | A package of code that extends Magento functionality | `Training_HelloWorld` module |
| **Theme** | Controls visual appearance of the store | Luma, Blank |
| **registration.php** | File that registers a module with Magento | `ComponentRegistrar::register()` |
| **module.xml** | Declares module name and version | `<module name="Vendor_Module" setup_version="1.0.0"/>` |
| **Docker** | Containerization tool for consistent development environments | `docker-compose up -d` |
| **Composer** | PHP package manager for dependencies | `composer require vendor/package` |
| **Xdebug** | PHP extension for debugging code | Set breakpoints in VS Code |
| **Logger** | System for writing messages to log files | `$this->logger->info('message')` |
| **bin/magento** | Magento CLI tool for commands | `bin/magento cache:flush` |

---

## Topic 02: Routing & Controllers

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
| **Block** | Layout element that renders content | `<block name="product.info">` |
| **PHPCS** | PHP Code Sniffer for coding standards | `vendor/bin/phpcs app/code/` |

---

## Topic 4: Data Layer

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

---

## Topic 04: Repositories

| Term | Definition | Example |
|------|------------|---------|
| **Repository** | Class that manages data access (CRUD) | `Model/ReviewRepository.php` |
| **Service Contract** | Interface defining what methods are available | `Api/ReviewRepositoryInterface.php` |
| **Data Interface** | Interface defining data structure | `Api/Data/ReviewInterface.php` |
| **Search Criteria** | Object for filtering and sorting queries | `SearchCriteriaBuilder` |
| **Filter Group** | Groups multiple filters together | Multiple conditions in API calls |
| **DTO** | Data Transfer Object | Simple object for transferring data |

---

## Topic 05: Plugins & Observers

| Term | Definition | Example |
|------|------------|---------|
| **Plugin** | Modifies behavior of public methods | Around/before/after modifier |
| **Around Plugin** | Wraps original method completely | `aroundExecute()` |
| **Before Plugin** | Runs before original method | `beforeExecute()` |
| **After Plugin** | Runs after original method | `afterExecute()` |
| **Observer** | Responds to events dispatched in Magento | Listens for `catalog_product_save_after` |
| **Event** | Signal that something happened in the system | `customer_register_success` |
| **Area** | Part of Magento (frontend, adminhtml, crons) | Different configurations per area |
| **di.xml** | Dependency injection configuration | Defines preferred implementations |

---

## Topic 7: APIs

| Term | Definition | Example |
|------|------------|---------|
| **REST API** | HTTP-based API for communication | `/rest/V1/products` |
| **GraphQL** | Query language for APIs | `/graphql` endpoint |
| **OAuth** | Authentication protocol for APIs | Token-based auth |
| **Webhooks** | HTTP callbacks for events | Outgoing notifications |
| **API Endpoint** | URL that provides API access | `/rest/default/V1/customers` |
| **Swagger/OpenAPI** | API documentation format | Generated API docs |

---

## Topic 8: Data Operations

| Term | Definition | Example |
|------|------------|---------|
| **Import** | Process of loading data into Magento | Import products from CSV |
| **Export** | Process of extracting data from Magento | Export orders to CSV |
| **CSV** | Comma-Separated Values file format | `product_import.csv` |
| **Cron** | Scheduled task system in Magento | Runs every minute |
| **Job** | Single cron task | `catalog_product_alert` |
| **Message Queue** | System for async processing | RabbitMQ integration |

---

## Topic 08: Performance

| Term | Definition | Example |
|------|------------|---------|
| **Cache** | Stored data for fast access | Full page cache |
| **Indexer** | Process that updates catalog data | Product indexer |
| **Varnish** | HTTP caching proxy | Built-in full page cache |
| **Redis** | In-memory data store | Cache and session storage |
| **Profiler** | Tool for measuring performance | Built-in Magento profiler |
| **Compilation** | Generating factory classes | `bin/magento di:compile` |
| **AJAX** | Asynchronous JavaScript for dynamic updates | Product price update |
| **UI Component** | JavaScript-based UI elements | Knockout.js grid |

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

*Document Version: 1.0*  
*Last Updated: 2026-03-23*