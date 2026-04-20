# Topic 17: Adobe Commerce vs Open Source — Edition Decision Guide

**Goal:** Understand the differences between Magento Open Source (CE) and Adobe Commerce (EE/Enterprise) so you can make informed architectural decisions and advise clients on edition selection.

---

## Topics Covered

- The shared codebase foundation and EE's enterprise layering
- B2B features (Company Accounts, Shared Catalogs, RFQ, Quick Order, Negotiable Quotes)
- Content Staging & Preview (campaign scheduling, preview URLs, staging dashboards)
- Advanced Search (Live Search powered by Adobe Sensei, OpenSearch, Elasticsearch)
- Performance infrastructure (FPC, indexing, Cloud PaaS vs on-premise)
- Admin and customer experience enhancements (Page Builder, Reporting, Customer Segments)
- Order Management and inventory (MSI in both editions, MCOM as the EE layer)
- Loyalty programs (Reward Points, Store Credits, Gift Wrapping — native EE vs extensions for CE)
- Security differences (Security Scan Tool, WAF/DDoS, SLA-backed response)
- API differences (B2B GraphQL, API Mesh, App Builder — EE-exclusive capabilities)
- Cloud vs on-premise infrastructure (Fastly CDN, auto-scaling, PCI compliance)
- Practical edition decision framework with decision tables and price references

---

## Topics

---

### Topic 1: The Two Editions — Shared Foundation, Divergent Capabilities

**The Codebase Relationship:**

Magento Open Source (CE) and Adobe Commerce (EE) share the same core codebase. Adobe Commerce is not a fork — it is the same platform with a proprietary enterprise layer added on top. The base Magento application framework, all core modules (Catalog, Checkout, Customer, Sales, CMS, etc.), the API architecture, plugin system, and event system are identical in both editions.

The EE layer adds approximately 40+ enterprise modules that are not present in CE. These modules are located in `vendor/magento/module-*-enterprise` (for on-premise EE) or enabled via Adobe Commerce Cloud's `composer.json` when deployed to Cloud.

**What This Means for Developers:**

```php
<?php
// CE: module-customer-import
// EE: module-customer-import + module-customer-segment + module-customer-permissions

// In practice, if you're building a custom module that depends on an EE-only module,
// your code will fail to load in CE. Always check module availability at runtime:

public function __construct(
    \Magento\CustomerSegment\Model\Segment $segment = null // Optional = safe for CE
) {
    $this->segment = $segment;
}

// Or use feature flag detection:
if ($this->moduleManager->isOutputEnabled('Magento_CustomerSegment')) {
    // Safe to use segment features
}
```

**Module Availability Check:**

```php
<?php
use Magento\Framework\Component\ComponentRegistrar;
use Magento\Framework\Module\ModuleList;

class FeatureDetector
{
    public function __construct(
        private readonly ModuleList $moduleList,
    ) {}

    public function isEnterpriseModuleAvailable(string $moduleName): bool
    {
        return $this->moduleList->getConfig($moduleName) !== null;
    }

    public function hasB2bFeatures(): bool
    {
        return $this->isEnterpriseModuleAvailable('Magento_Company')
            && $this->isEnterpriseModuleAvailable('Magento_Rfq');
    }
}
```

**Edition Comparison at a Glance:**

| Dimension | Magento Open Source (CE) | Adobe Commerce (EE) On-Premise | Adobe Commerce Cloud |
|-----------|--------------------------|-------------------------------|---------------------|
| Core platform | ✅ Same | ✅ Same | ✅ Same |
| Source code | Public (GitHub) | Private (composer repo) | Private (composer repo) |
| License | Open Source (OSL 3.0) | Proprietary | Proprietary + Cloud license |
| Price model | Free (self-hosted) | Annual license fee | Annual license + infrastructure |
| Support | Community forums | Adobe support (with SLA) | Adobe TAM + Cloud support |
| Infrastructure | Self-managed | Self-managed | Managed PaaS by Adobe |
| Update cadence | Community-driven | Adobe-driven | Adobe-driven (patch testing) |

> **Why this matters:** When you architect a solution, the choice of edition constrains your dependencies. Code targeting EE-only modules will not run on CE. Module Marketplace submissions must declare CE compatibility or EE exclusivity clearly.

**The "Feature Gap" Misconception:**

Many engineers assume EE is "just CE with more modules." The reality is more nuanced:

1. **Same core, different integration depth:** EE modules are deeply integrated with core modules — not just layered on top. The B2B modules modify core quote, order, and customer flows in ways that third-party extensions cannot replicate.

2. **Infrastructure is part of the product in Cloud:** Adobe Commerce Cloud is not just hosted Magento. It includes Fastly CDN, a pre-configured Varnish layer, Blackfire for profiling, New Relic APM, and a managed database tier. Running EE on-premise does NOT give you these automatically.

3. **B2B is a full platform, not a feature set:** The B2B features (Company, SharedCatalog, RFQ, QuickOrder, NegotiableQuote) form an interconnected system. You cannot extract one piece (e.g., only RFQ) without the underlying data structures and permission models that depend on the others.

---

### Topic 2: B2B Features — Company Accounts, Shared Catalogs, RFQ, Quick Order, Negotiable Quotes

B2B capabilities are the most significant differentiator between CE and EE. These features are built into the enterprise layer and are not available in CE, even via third-party extensions (without significant custom development).

**Company Accounts (`Magento_Company`):**

The Company module introduces a hierarchical organizational structure for B2B customers. Where a CE store treats each customer as an individual (or a guest), the Company module allows multiple buyer users to belong to a single company account.

**Why it matters:** Without Company accounts, implementing B2B workflows requires building your own organizational hierarchy from scratch — something that touches nearly every module in Magento.

```
Company Account Structure:
├── Company (top-level entity)
│   ├── Company Users (individual logins)
│   │   ├── Admin (can approve others, place orders)
│   │   ├── Buyer (can place orders)
│   │   └── Viewer (read-only access)
│   ├── Company Addresses (shared shipping/billing)
│   ├── Company Payment Methods (credit limit)
│   └── Company Shipping Methods (approved carriers)
```

**Key Capabilities:**

| Feature | Description |
|---------|-------------|
| Hierarchical users | Users belong to a company with role-based permissions |
| Team-based approval | Orders above threshold require manager approval |
| Company credit | Credit limits per company (for Net-30 terms) |
| Shared addresses | Company-level address book, no per-user duplication |
| Company admin | One user is the "super-admin" of the company account |

**Shared Catalogs (`Magento_SharedCatalog`):**

Shared Catalogs allows different companies (or customer groups) to see different product collections and pricing. It's essentially a segmented, company-specific catalog.

**CE Substitution vs EE Requirement:**

| Approach | CE Limitation | EE Solution |
|----------|--------------|-------------|
| Group pricing | Per-group tiered pricing exists in CE | Shared catalogs give per-company catalogs with custom pricing |
| Category visibility | Root category per store is the limit | Shared catalog can be applied to specific companies, overriding group-level |
| Custom pricing per customer | Tier prices are group-level | Company-specific unit prices override tier/group pricing |

**Request for Quote (`Magento_Rfq`):**

RFQ allows buyers to request a quote for large quantities or custom configurations. The seller can respond with pricing, and the buyer can accept or negotiate.

```
RFQ Workflow:
1. Buyer adds products to cart → clicks "Request Quote"
2. Seller receives notification in admin
3. Seller creates Quote with custom pricing (negotiated price)
4. Buyer receives quote notification → can Accept / Counter / Decline
5. On acceptance, the Quote converts to an Order with negotiated price
```

**Quick Order (`Magento_QuickOrder`):**

Enables B2B buyers to rapidly order products by SKU or CSV upload — essential for reordering workflows common in procurement.

**Negotiable Quotes (`Magento_NegotiableQuote`):**

Extends RFQ with a negotiation interface — buyers can propose counter-prices, and the seller can accept or reject.

**CE Substitution Reality:**

| EE Feature | CE Alternative | Gap |
|------------|---------------|-----|
| Company accounts | Build custom company entity + user hierarchy | ~3-6 months custom dev |
| Shared catalogs | Category + group pricing + customer groups | Custom pricing per company not possible |
| RFQ | Third-party extension (Magemojo RFQ, ... ) | ~$500-2000/year subscription |
| Quick Order | Manual cart + SKU entry | Feasible with custom dev (~1 month) |
| Negotiable quotes | Not available in CE ecosystem | N/A — EE-exclusive |

> **Pro Tip:** When advising clients on the B2B decision, ask: "Do your customers negotiate pricing, or do they order from a fixed price catalog?" If negotiation is required, EE is the only option. If they simply need tiered pricing and large orders, CE with custom development may suffice at 30-40% of EE cost.

---

### Topic 3: Content Staging & Preview — Campaign Scheduling

Content Staging is an EE-exclusive feature that allows merchants to schedule content changes (price rules, promotions, CMS pages, catalog changes) in advance and preview them before they go live.

**The Staging Dashboard:**

```php
<?php
// EE: Content Staging creates "campaigns" that bundle scheduled changes
// Each campaign has a start/end time and contains change records

// In the admin: Content → Content Staging → Dashboard
// Shows all scheduled changes across price rules, promotions, CMS, and catalog updates
```

**What Gets Staged:**

| Entity | Staging Capability |
|--------|-------------------|
| Catalog Price Rules | Schedule active periods |
| Cart Price Rules | Schedule promotions |
| CMS Pages/Blocks | Schedule content updates |
| Products | Schedule price changes, inventory changes |
| Categories | Schedule structural changes |
| CMS Hierarchy | Schedule navigation changes |

**Preview URLs:**

EE generates preview URLs for staging campaigns — shareable links that show the storefront as it will appear at a future date.

```bash
# Preview URL format (EE only):
https://store.com/catalog/product/123?
   ___store=en_us&
    ___from_store=default&
    stg=campaign_id&
    stg_from=1614556800&      # Campaign start timestamp
    stg_to=1615142400&        # Campaign end timestamp
    stg_preview=1
```

**How Staging Works Under the Hood:**

```php
<?php
// EE: The staging system intercepts entity save operations
// and creates "snapshot" records that override live data at scheduled times

// When a product is saved with a scheduled change:
// 1. Original entity is saved as-is
// 2. A "staging_record" is created in catalog_staging_* tables
// 3. At the scheduled time, the staging value takes precedence

// On the storefront, Magento merges live values with staging values
// using the StagingApplierInterface implementation
```

**CE Substitution:**

| EE Feature | CE Approach | Limitation |
|-----------|-------------|------------|
| Scheduled price rules | Cron jobs to enable/disable rules | No preview, no coordinated campaigns |
| Preview URLs | Manually test with date manipulation | No shareable preview links |
| Staging dashboard | Manual tracking in spreadsheets | No unified view of all scheduled changes |
| Campaign bundling | Multiple cron jobs with no grouping | All-or-nothing deployment |

> **Common Pitfall:** Engineers sometimes try to build a "staging lite" system for CE using database flags and cron jobs. This works for simple price rule scheduling but breaks down when you need preview URLs, campaign bundling, or content staging for multiple entities simultaneously.

---

### Topic 4: Advanced Search — Live Search & Elasticsearch

**The Search Landscape in Magento 2.4:**

| Search Type | Edition | Engine Used |
|------------|---------|-------------|
| Basic MySQL LIKE search | CE/EE | MySQL FULLTEXT |
| Elasticsearch (3rd party) | CE/EE | Elasticsearch (self-hosted) |
| OpenSearch support | CE 2.4.6+ / EE | OpenSearch (AWS, self-managed) |
| Adobe Commerce Live Search | EE Cloud only | Adobe Sensei + Elasticsearch Cloud |

**Live Search (`Magento_LiveSearch`):**

Live Search is an EE-exclusive, cloud-native search service powered by Adobe Sensei (Adobe's AI/ML platform). It is only available on Adobe Commerce Cloud and replaces the self-managed Elasticsearch stack.

**Key Capabilities:**

| Feature | Basic Elasticsearch | Live Search |
|---------|-------------------|-------------|
| AI-powered ranking | Manual synonyms + boosting | Adobe Sensei automatic ranking |
| Faceted search | Manual facet configuration | Auto-generated facets |
| Search synonyms | Manual management | AI-curated synonyms |
| Performance | Depends on your infra | Managed cloud (low latency globally) |
| Reporting | External tools needed | Built-in search analytics |
| Typo tolerance | Basic | Advanced (semantic understanding) |
| "Search as you type" | Requires custom work | Built-in |

**CE Elasticsearch/OpenSearch Setup:**

```php
<?php
// In both CE and EE, search engines are configured via env.php:
// (These are NOT EE-only capabilities — CE supports Elasticsearch too)

return [
    'system' => [
        'default' => [
            'catalog' => [
                'search' => [
                    'engine' => 'elasticsearch7',
                    'elasticsearch7_server_hostname' => 'http://elasticsearch:9200',
                    'elasticsearch7_index_prefix' => 'magento',
                ]
            ]
        ]
    ]
];
```

**When Elasticsearch Alone Is Sufficient (CE):**

For catalogs under ~500,000 SKUs with standard faceted search needs:
- CE + self-managed Elasticsearch provides 90% of search capability at minimal cost
- Elasticsearch 7.x or OpenSearch with proper synonym configuration handles most use cases
- Custom synonym management and boosting rules can compensate for lack of AI

**When Live Search Is Required:**

- Very large catalogs (>1M SKUs) where AI ranking improves relevance
- Businesses where search conversion rate is a primary KPI
- Merchants who want built-in search analytics without additional tooling
- Global stores where low-latency search across regions is critical

**Elasticsearch for EE On-Premise:**

EE on-premise does NOT include Live Search — it still uses self-managed Elasticsearch or OpenSearch. The Live Search cloud service is separate and requires Cloud infrastructure.

---

### Topic 5: Performance — FPC, Indexing, Cloud Infrastructure

**Full-Page Cache (FPC) — Shared Foundation:**

Both CE and EE include full-page caching via Varnish or built-in cache. The FPC architecture is identical. The difference lies in the infrastructure layer.

**Indexing — Shared Foundation, EE Add-ons:**

| Indexer | CE | EE |
|---------|----|----|
| `catalog_product_price` | ✅ | ✅ |
| `catalogsearch_fulltext` | ✅ | ✅ |
| `inventory` (MSI) | ✅ | ✅ |
| `customer_grid` | ✅ | ✅ |
| `b2b_company_category_permissi` | ❌ | ✅ |
| `shared_catalog_category_permissi` | ❌ | ✅ |
| `staging_product_price` | ❌ | ✅ |

**Cloud Infrastructure Advantages:**

Adobe Commerce Cloud provides a managed Platform-as-a-Service (PaaS) environment. The infrastructure differences between EE on-premise and EE Cloud are substantial:

| Component | EE On-Premise | EE Cloud |
|-----------|---------------|----------|
| CDN | Self-configure (CloudFlare, Fastly, etc.) | Fastly CDN (included, pre-configured) |
| Application servers | Your server management | Auto-scaling (up/down based on traffic) |
| Database | Self-managed MySQL | Managed MySQL (Adobe-monitored) |
| Cache | Redis (self-configured) | Redis (managed, HA configuration) |
| Search | Self-managed Elasticsearch | Elasticsearch Cloud (for non-LiveSearch setups) |
| Profiling | Manual New Relic/Blackfire setup | Blackfire (included, pre-configured) |
| APM | External service | New Relic APM (included) |
| Deployment | Manual CI/CD | Git-push deployment (auto-build, deploy hooks) |
| PCI compliance | Self-audit | PCI compliance handled at Cloud level |
| SLA | None | 99.9% uptime SLA |
| Security | Self-managed | Managed WAF, DDoS protection |

**Auto-Scaling:**

Cloud scales horizontally — adding application server nodes when traffic increases. This is not available in on-premise EE.

```bash
# Cloud: Auto-scaling triggers based on metrics in .magento.app.yaml:
name: my-store
type: php:8.1
build:
    flavor: magento
dependencies:
    php:
        composer/composer: '*'
web:
    commands:
        start: |
            php-fpm -D
            nginx -c /app/etc/nginx/nginx.conf
scale:
    min_s_instances: 2
    max_s_instances: 10
    triggers:
        - type: cpuwait
          threshold: 70
        - type: memory
          threshold: 80
```

**Why Infrastructure Matters for Performance:**

Even with identical code, EE on-premise and EE Cloud perform differently due to infrastructure:

1. **Fastly CDN** serves cached pages from edge nodes globally — CE users need to configure this themselves
2. **Managed Redis** with proper session/cache separation doesn't require tuning
3. **Blackfire** profiling is pre-configured in Cloud — on-premise requires manual installation
4. **Database replication** (read replicas) are easier to configure in Cloud

> **Decision point:** If a client is running EE on-premise but without proper CDN, Redis, and APM setup, they are getting the EE code layer but without most of the performance advantages Adobe built into the platform. The infrastructure investment is as important as the license.

---

### Topic 6: Admin & Customer Experience — Page Builder, Reporting, Segmentation

**Page Builder (`Magento_PageBuilder`):**

Page Builder is EE-exclusive for creating rich content pages via a drag-and-drop interface in the admin. In CE, content pages require direct HTML/Knockout.js template editing.

**What Page Builder Enables:**

- Drag-and-drop layout of CMS pages, blocks, and product descriptions
- No coding required for content managers
- Pre-built content types: text, images, video, sliders, tabs, accordions
- Inventory-aware dynamic blocks (shows product availability)

**CE Substitution:**

| Content Need | CE Approach | Gap |
|-------------|-------------|-----|
| Rich CMS pages | Direct HTML/CSS in .phtml or CMS content | No visual editor for non-technical users |
| Dynamic product blocks | Widgets with hardcoded product IDs | No dynamic inventory-aware blocks |
| Banner carousels | Third-party extensions | ~$200-1000 per extension |

**Commerce Intelligence (Business Intelligence):**

Commerce Intelligence is Adobe's reporting and analytics platform — included with EE. It provides:
- Revenue and profit analytics
- Customer lifetime value
- Cohort analysis
- Product performance
- Inventory forecasting
- Custom dashboard builder

**CE Reporting Alternative:**

CE has basic reports (Reports → Statistics). For anything beyond basic sales data, CE merchants typically integrate with:
- Google Analytics 4 (free, basic)
- Metorik (~$200-500/month, Magento-native analytics)
- Tableau/Power BI (self-service BI, requires data warehouse setup)

| Report Type | CE Native | Commerce Intelligence |
|-------------|-----------|----------------------|
| Sales by day/week/month | ✅ Basic | ✅ Advanced + forecasting |
| Customer acquisition | ❌ | ✅ |
| Cohort analysis | ❌ | ✅ |
| Profit margin analysis | ❌ | ✅ |
| Inventory forecasting | ❌ | ✅ |
| Custom dashboards | ❌ | ✅ |

**Customer Segments — EE Enhancements:**

Customer Segments exist in both CE and EE, but EE adds:

| Enhancement | CE | EE |
|-------------|----|----|
| Segment creation (admin UI) | ✅ | ✅ |
| Real-time segment recalculation | ⚠️ Partial | ✅ Full |
| Segment-based content in CMS | ❌ | ✅ |
| Segment-based promotions | ⚠️ Basic | ✅ Advanced (including negotiable quote) |
| API access to segments | ❌ | ✅ |
| Segment-based shared catalogs | ❌ | ✅ |

---

### Topic 7: Order Management — Multi-Source Inventory & MCOM

**Multi-Source Inventory (MSI) — Available in Both CE and EE:**

MSI was introduced in Magento 2.3 and is available in both editions. It enables tracking inventory across multiple warehouses/sources.

```php
<?php
// MSI is identical in both CE and EE:

// Sources = physical locations (warehouse, store, distribution center)
use Magento\InventoryApi\Api\SourceRepositoryInterface;

// Stocks = logical groupings of sources for sales channels
use Magento\InventoryApi\Api\StockRepositoryInterface;

// SourceItems = per-SKU, per-Source quantity tracking
use Magento\InventoryApi\Api\SourceItemRepositoryInterface;

// Reservation = promises of inventory (prevents overselling)
use Magento\InventoryReservationsApi\Api\ReservationBuilderInterface;
```

**What MSI Provides in Both Editions:**

| Feature | CE | EE |
|---------|----|----|
| Multiple source management | ✅ | ✅ |
| Inventory reservations | ✅ | ✅ |
| Shipment source selection | ✅ | ✅ |
| In-store pickup | ✅ | ✅ |
| Default Stock (single source) | ✅ | ✅ |

**Magento Commerce Operations (MCOM) — EE Cloud Only:**

MCOM is Adobe's cloud-native order management system that layers on top of MSI for enterprise merchants with complex fulfillment needs.

**When MSI Alone Is Enough:**

- Single warehouse or store with simple fulfillment
- Up to ~5 shipping origins
- Standard backorder/preorder handling
- Basic inventory tracking across 1-3 sources

**When MCOM Is Required:**

- 5+ distribution centers or stores
- Complex fulfillment orchestration (drop-ship, cross-dock)
- Integration with 3PL systems (warehouse management)
- Order promising based on real-time inventory across sources
- Returns management with automatic refund/replacement routing

| Capability | MSI Only | MCOM |
|-----------|----------|------|
| Simple fulfillment | ✅ | ✅ |
| Multi-warehouse routing | ✅ (manual) | ✅ (automated) |
| 3PL integration | Custom dev | Pre-built connectors |
| Order promising (ATP) | Basic | Real-time, multi-source |
| Returns management | Basic | Advanced (RAMA logic) |
| Inventory optimization | ❌ | ✅ |

---

### Topic 8: Loyalty — Reward Points, Store Credits, Gift Wrapping

**EE-Exclusive Loyalty Features:**

Adobe Commerce includes native loyalty modules that are not available in CE:

| Feature | CE Status | EE Module |
|---------|-----------|-----------|
| Reward Points | ❌ (extension required) | ✅ `Magento_Reward` |
| Store Credit | ❌ (extension required) | ✅ `Magento_CustomerBalance` |
| Gift Wrapping | ❌ (extension required) | ✅ `Magento_GiftWrapping` |
| Gift Cards | ❌ (extension required) | ✅ `Magento_GiftCard` |
| Loyalty points on Order | Extension | Native + advanced configuration |

**Reward Points (`Magento_Reward`):**

```php
<?php
// In EE, reward points are fully integrated:

use Magento\Reward\Model\Reward;

// Earning points:
$reward->setCustomerId($customerId)
    ->setWebsiteId($websiteId)
    ->setPointsBalance(100)
    ->setPointsDelta(50)  // Award 50 points
    ->setPointsHistoryAction(\Magento\Reward\Model\Reward::ACTION_ORDER)
    ->save();

// Redeeming points at checkout:
$quote->getExtensionAttributes()
    ->getRewardCurrencyInfo()
    ->setRewardPoints(100);  // Use 100 points as discount
```

**Store Credit (`Magento_CustomerBalance`):**

```php
<?php
// Store credit is a credit limit per company (B2B) or customer (B2C)

// Checking balance:
use Magento\CustomerBalance\Model\CustomerBalance;
$balance = $customerBalance->loadByCustomer($customerId);
echo $balance->getAmount();  // Available credit

// Applying credit at checkout:
$quote->setUseCustomerBalance(true);
```

**CE Extension Ecosystem:**

For CE merchants wanting loyalty features, the extension ecosystem provides options:

| Extension Type | Options | Price Range |
|---------------|---------|------------|
| Reward Points | MageRewards, Aheadworks Reward Points, Sweet Tooth | ~$300-2000 (one-time) |
| Store Credit | Custom dev or extension | ~$200-1000 |
| Gift Wrapping | Fooman Fusion, Swissuplabs Giftwrap | ~$150-500 |
| Gift Cards | Unifer Gift Cards, Mageworx Gift Cards | ~$200-1500 |

> **Trade-off:** Third-party extensions provide feature parity for individual capabilities but lack the deep integration that EE's native modules have with other EE features (e.g., Reward Points + Company accounts + Negotiable Quotes all share the same points ledger).

---

### Topic 9: Security — What's Shared, What's EE-Exclusive

**Shared Security Features (Available in CE):**

| Security Feature | CE | EE |
|-----------------|----|----|
| Password hashing (bcrypt) | ✅ | ✅ |
| Account locking (failed login) | ✅ | ✅ |
| CAPTCHA/reCAPTCHA | ✅ | ✅ |
| Admin 2FA | ✅ | ✅ |
| Security Scan Tool | ✅ (free) | ✅ (included) |
| SQL injection protection | ✅ | ✅ |
| XSS protection (output escaping) | ✅ | ✅ |
| CSRF tokens | ✅ | ✅ |

**Security Scan Tool:**

The Security Scan Tool (`Magento_Securityscan`) is free for CE as well. It monitors for known vulnerabilities and malware.

```bash
# Running the security scan (CLI):
bin/magento security:scan --scan="[your Magento ID]"
```

**EE-Exclusive Security:**

| Feature | CE | EE On-Prem | EE Cloud |
|---------|----|-----------|----------|
| WAF (Web Application Firewall) | ❌ | ❌ | ✅ (Incapsula/Fastly) |
| DDoS protection | ❌ | ❌ | ✅ |
| SPLA-backed support response | ❌ | ❌ | ✅ |
| Penetration testing (on request) | ❌ | ❌ | ✅ |
| PCI compliance tooling | Basic | Self-audit | Managed PCI |
| Adobe security team SLA | ❌ | ❌ | ✅ |

**The WAF Difference:**

The Web Application Firewall on Cloud filters malicious traffic at the network edge before it reaches Magento. CE and EE on-premise merchants can purchase WAF services separately (e.g., CloudFlare, AWS WAF), but they are not included.

**SLA-Backed Support Response:**

| Support Tier | Response Time | Incident Support |
|-------------|---------------|-----------------|
| CE Community | Forums, volunteer | N/A |
| EE On-Premise | Business hours (8x5) | Best-effort |
| EE Cloud | 24/7 with SLA | Designated TAM, escalation path |

---

### Topic 10: API Differences — B2B GraphQL, API Mesh, App Builder

**REST/GraphQL — Common Foundation:**

Both CE and EE share the same REST and GraphQL foundation. The same endpoints work in both editions for core entities (products, customers, orders, categories).

**B2B GraphQL Endpoints — EE Only:**

| Endpoint | CE | EE |
|----------|----|----|
| `/V1/guest-carts/:id/billing-address` | ✅ | ✅ |
| `/V1/carts/:id/billing-address` | ✅ | ✅ |
| `/V1/requestforquote` | ❌ | ✅ |
| `/V1/sharedCatalog` | ❌ | ✅ |
| `/V1/company` | ❌ | ✅ |
| `/V1/negotiableQuote` | ❌ | ✅ |
| `/V1/quickOrder` | ❌ | ✅ |
| `/V1/reward` | ❌ | ✅ |

**B2B GraphQL Schema Example (EE only):**

```graphql
# Negotiable Quote — EE only
type NegotiableQuote {
  uid: ID!
  company_id: Int!
  company_user_id: Int!
  items: [NegotiableQuoteItem!]!
  status: NegotiableQuoteStatus!
  negotiated_price_value: Float
  shipping_address: NegotiableQuoteAddress
  notes: String
}

enum NegotiableQuoteStatus {
  SUBMITTED
  PROCESSING
  UPDATED
  AGREED
  DECLINED
  CLOSED
}

type Mutation {
  createNegotiableQuote(input: NegotiableQuoteCreateInput!): NegotiableQuote
  submitNegotiableQuote(uid ID!): NegotiableQuote
  updateNegotiableQuotePrice(uid: ID!, price: Float!): NegotiableQuote
}
```

**API Mesh (`Magento_ApiMesh`) — EE Cloud Only:**

API Mesh is an EE Cloud-exclusive feature that allows aggregating multiple APIs (not just Magento) into a single GraphQL gateway. It enables:

- Combining Magento GraphQL with external APIs (ERP, CRM, PIM)
- Schema stitching/federation
- Authentication across multiple API sources
- Unified GraphQL endpoint for headless storefronts

**CE GraphQL Limitations:**

- No B2B-specific mutations/queries
- No API Mesh (must build custom aggregation layer)
- No real-time WebSocket subscriptions (GraphQL subscriptions are CE 2.4.7+ but limited)

**Adobe App Builder — EE Cloud Only:**

App Builder is Adobe's serverless platform for building extensibility apps that connect to Adobe Commerce. It is Cloud-exclusive and provides:

- Serverless functions (React-based, Node.js backend)
- Integration with Adobe I/O events
- Pre-built connectors for Adobe ecosystem (Analytics, Target, Experience Platform)
- Custom admin UI extensions

> **Important for API architects:** If a client's project requires B2B APIs, API Mesh, or App Builder, the only viable edition is EE Cloud. EE on-premise does not include these features.

---

### Topic 11: Cloud vs On-Premise — Infrastructure Differences

**The Two Cloud Products:**

Adobe Commerce Cloud comes in two flavors — both are Platform-as-a-Service (PaaS) but with different infrastructure:

| Aspect | Adobe Commerce Cloud (ACaaS) | Adobe Commerce on Cloud (Paas) |
|--------|------------------------------|--------------------------------|
| Managed infrastructure | ✅ | ✅ |
| Fastly CDN | ✅ | ✅ |
| Auto-scaling | ✅ | ✅ |
| Blackfire profiling | ✅ | ✅ |
| New Relic APM | ✅ | ✅ |
| Dedicated environment | Starter | Pro |
| Multi-environment | Staging + Production | Staging + Production + Integration |
| Git-push deploy | ✅ | ✅ |
| PCI compliance | ✅ | ✅ |
| SLA | 99.9% | 99.9% |

**The "Cloud" Misconception:**

Many engineers think "Adobe Commerce Cloud" is simply Magento running on someone else's servers. The reality is a heavily customized PaaS that includes:

1. **ECE-tools:** A set of scripts that handle build, deploy, and configuration management specifically for the Cloud environment (`.magento.app.yaml`, `deploy.php`, hooks)

2. **Restricted write access:** On Cloud, certain directories are read-only at runtime (`var`, `pub/static`, `pub/media`). You cannot write to these directories from the application. This is a common source of bugs when CE-developed code is moved to Cloud.

3. **Environment-specific configurations:** Database credentials, Redis endpoints, and search hosts are injected via environment variables at deploy time, not stored in `env.php`.

4. **Build and deploy hooks:** The CI/CD pipeline is defined in `.magento.app.yaml`:

```yaml
# .magento.app.yaml
name: my-store
type: php:8.1
build:
    hooks:
        build: |
            php bin/magento maintenance:enable
            php bin/magento setup:di:compile
            php bin/magento setup:static-content:deploy -f
        deploy: |
            php bin/magento cache:flush
            php bin/magento maintenance:disable
web:
    commands:
        start: |
            php-fpm -D
            nginx -c /app/etc/nginx/nginx.conf
```

**On-Premise EE Limitations:**

If you deploy EE on your own infrastructure, you do NOT get:
- Managed CDN
- Auto-scaling
- Pre-configured APM/profiling
- Managed WAF
- PCI compliance tooling
- SLA-backed support

Running EE on-premise is effectively "CE with more modules but the same infrastructure constraints." The infrastructure advantages are only realized in Cloud.

---

### Topic 12: Practical Edition Decision Framework

**Decision Matrix — When to Choose CE vs EE vs Cloud:**

| Requirement | Choose CE | Choose EE On-Prem | Choose EE Cloud |
|-------------|----------|-------------------|-----------------|
| Budget < $5K/year | ✅ | ❌ | ❌ |
| Budget $5K-$25K/year | ⚠️ | ✅ (if B2B needed) | ❌ |
| Budget > $25K/year | ❌ | ⚠️ | ✅ |
| B2B features required | ❌ | ✅ | ✅ |
| Company accounts, RFQ | ❌ | ✅ | ✅ |
| Content staging + preview | ❌ | ✅ | ✅ |
| Live Search (Adobe Sensei) | ❌ | ❌ | ✅ |
| Managed infrastructure | ❌ | ❌ | ✅ |
| PCI compliance required | ❌ | ⚠️ (self-audit) | ✅ |
| 99.9% SLA required | ❌ | ❌ | ✅ |
| Auto-scaling for traffic spikes | ❌ | ❌ | ✅ |
| Large catalog (>500K SKUs) | ⚠️ (tuning needed) | ⚠️ | ✅ |
| Headless architecture | ✅ | ✅ | ✅ |
| Custom Magento development | ✅ | ✅ | ⚠️ (restrictions apply) |

**Price Range Reference:**

| Edition | Approximate Annual Cost |
|---------|------------------------|
| Magento Open Source | Free (hosting costs apply: ~$50-500/month) |
| Adobe Commerce On-Premise | ~$22,000 – $125,000+/year (varies by GMV, countries, modules) |
| Adobe Commerce Cloud | ~$55,000 – $250,000+/year (includes infrastructure + license) |

> **Note:** Adobe Commerce pricing is GMV (Gross Merchandise Value) based. The license fee scales with revenue. The figures above are rough estimates — actual pricing depends on negotiation, contract terms, and Adobe's current pricing policy.

**Team Capability Assessment:**

| Team Capability | Recommended Edition |
|----------------|---------------------|
| Small team, limited DevOps | CE → Cloud migration when budget allows |
| Experienced ops team, tight budget | EE On-Premise |
| Enterprise team, compliance requirements | EE Cloud |
| B2B requirements + limited budget | EE On-Premise (B2B is EE-only) |
| Heavy traffic, no ops team | EE Cloud |

**Feature Availability Quick Reference:**

| Feature | CE | EE On-Prem | EE Cloud |
|---------|----|-----------|----------|
| Core platform | ✅ | ✅ | ✅ |
| B2B Company Accounts | ❌ | ✅ | ✅ |
| B2B Shared Catalogs | ❌ | ✅ | ✅ |
| B2B RFQ + Negotiable Quotes | ❌ | ✅ | ✅ |
| Content Staging | ❌ | ✅ | ✅ |
| Live Search (Sensei) | ❌ | ❌ | ✅ |
| Page Builder | ❌ | ✅ | ✅ |
| Commerce Intelligence | ❌ | ✅ | ✅ |
| Reward Points | ❌ (extension) | ✅ | ✅ |
| Store Credit | ❌ (extension) | ✅ | ✅ |
| Gift Wrapping | ❌ (extension) | ✅ | ✅ |
| MCOM (Order Management) | ❌ | ❌ | ✅ |
| API Mesh | ❌ | ❌ | ✅ |
| App Builder | ❌ | ❌ | ✅ |
| Fastly CDN | ❌ (self-configure) | ❌ (self-configure) | ✅ |
| Managed WAF/DDoS | ❌ | ❌ | ✅ |
| Auto-scaling | ❌ | ❌ | ✅ |
| Blackfire/New Relic (managed) | ❌ | ❌ | ✅ |
| PCI compliance tooling | ❌ | ⚠️ (self-audit) | ✅ |

**Decision Tree:**

```
Is B2B functionality required?
├── NO → Is 99.9% SLA or managed infrastructure required?
│       ├── NO → Is budget < hosting costs?
│       │       ├── YES → CE
│       │       └── NO → Is auto-scaling needed?
│       │               ├── YES → EE Cloud
│       │               └── NO → CE
│       └── YES → EE Cloud
└── YES → Is managed infrastructure needed?
        ├── NO → EE On-Premise
        └── YES → EE Cloud
```

---

## Completion Criteria

- [ ] **Understand** the shared codebase relationship between CE and EE and explain what "enterprise layering" means in Magento's architecture
- [ ] **Compare** the B2B feature set (Company Accounts, Shared Catalogs, RFQ, Quick Order, Negotiable Quotes) and explain which cannot be replicated in CE
- [ ] **Evaluate** content staging capabilities and determine when preview URLs and campaign scheduling justify the EE license for a given client's content workflow
- [ ] **Recommend** an appropriate search solution (basic MySQL, Elasticsearch/OpenSearch, or Live Search) based on catalog size, AI-ranking needs, and budget constraints
- [ ] **Identify** the infrastructure differences between EE on-premise and EE Cloud, and explain which performance advantages require Cloud infrastructure
- [ ] **Describe** the loyalty module landscape (Reward Points, Store Credit, Gift Wrapping) and compare native EE capabilities vs the CE extension ecosystem
- [ ] **Compare** security features across CE vs EE On-Prem vs EE Cloud, and identify which security capabilities require Cloud infrastructure
- [ ] **Evaluate** API capabilities by edition (B2B GraphQL endpoints, API Mesh, App Builder) and determine which require EE Cloud exclusivity
- [ ] **Apply** the edition decision framework to recommend CE vs EE On-Prem vs EE Cloud for a given set of business requirements, budget, and team capabilities
- [ ] **Identify** which EE features are available in EE On-Premise and which are Cloud-only, and explain the implications for architectural decisions

---

## Summary Feature Matrix

| Feature Area | Feature | CE | EE On-Prem | EE Cloud |
|-------------|---------|----|-----------|----------|
| **Foundation** | Core platform | ✅ | ✅ | ✅ |
| | Source code access | Public | Private | Private |
| | Proprietary modules | ❌ | ✅ | ✅ |
| **B2B** | Company Accounts | ❌ | ✅ | ✅ |
| | Shared Catalogs | ❌ | ✅ | ✅ |
| | RFQ (Request for Quote) | ❌ | ✅ | ✅ |
| | Quick Order | ❌ | ✅ | ✅ |
| | Negotiable Quotes | ❌ | ✅ | ✅ |
| **Content** | Content Staging | ❌ | ✅ | ✅ |
| | Preview URLs | ❌ | ✅ | ✅ |
| | Page Builder | ❌ | ✅ | ✅ |
| **Search** | Basic MySQL search | ✅ | ✅ | ✅ |
| | Elasticsearch/OpenSearch | ✅ | ✅ | ✅ |
| | Live Search (Adobe Sensei) | ❌ | ❌ | ✅ |
| **Performance** | FPC (Varnish/cache) | ✅ | ✅ | ✅ |
| | Fastly CDN | ❌ (self-config) | ❌ (self-config) | ✅ |
| | Auto-scaling | ❌ | ❌ | ✅ |
| | Blackfire (managed) | ❌ | ❌ | ✅ |
| | New Relic APM (managed) | ❌ | ❌ | ✅ |
| **Inventory** | Multi-Source Inventory (MSI) | ✅ | ✅ | ✅ |
| | MCOM (Order Management) | ❌ | ❌ | ✅ |
| **Loyalty** | Reward Points | ❌ (ext) | ✅ | ✅ |
| | Store Credit | ❌ (ext) | ✅ | ✅ |
| | Gift Wrapping | ❌ (ext) | ✅ | ✅ |
| **Reporting** | Native reports (basic) | ✅ | ✅ | ✅ |
| | Commerce Intelligence | ❌ | ✅ | ✅ |
| **Security** | Security Scan Tool | ✅ | ✅ | ✅ |
| | WAF / DDoS protection | ❌ | ❌ | ✅ |
| | Managed PCI compliance | ❌ | ❌ | ✅ |
| | SLA-backed support | ❌ | ⚠️ (8x5) | ✅ (24/7) |
| **API** | REST/GraphQL (core) | ✅ | ✅ | ✅ |
| | B2B GraphQL endpoints | ❌ | ✅ | ✅ |
| | API Mesh | ❌ | ❌ | ✅ |
| | App Builder | ❌ | ❌ | ✅ |
| **Infrastructure** | Managed PaaS | ❌ | ❌ | ✅ |
| | Self-managed (on-prem) | ✅ | ✅ | ❌ |
| | Git-push CI/CD | ❌ | ❌ | ✅ |

---

## Reading List

- [Adobe Commerce Editions Comparison](https://business.adobe.com/products/commerce/magento/compare-editions.html) — Official edition comparison
- [Magento Open Source vs Adobe Commerce](https://developer.adobe.com/commerce/php/architecture/magento-vs-adobe-commerce/) — Architecture deep dive
- [B2B Features Documentation](https://experienceleague.adobe.com/docs/commerce-admin/b2b/introduction.html) — B2B capabilities
- [Content Staging](https://experienceleague.adobe.com/docs/commerce-admin/content-tools/staging/staging.html) — EE staging features
- [Adobe Commerce Cloud](https://experienceleague.adobe.com/docs/commerce-cloud-service/overview.html) — Cloud infrastructure guide
- [Live Search](https://experienceleague.adobe.com/docs/commerce-learn/tutorials/getting-started-with-live-search.html) — Live Search overview
- [Multi-Source Inventory](https://experienceleague.adobe.com/docs/commerce-inventory/overview.html) — MSI documentation
- [Adobe Commerce pricing](https://business.adobe.com/products/commerce/magento/pricing.html) — Official pricing page

---

*Informational Module: Edition Decision Guide*
---

*Magento 2 Backend Developer Course — Topic 17*
*Language: English*
