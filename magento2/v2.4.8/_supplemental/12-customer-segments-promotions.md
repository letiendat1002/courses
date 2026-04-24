---
title: "35 - Customer Segments & Promotions"
description: "Deep dive into Magento 2.4.8 customer segmentation, cart price rules, catalog price rules, and targeted promotions."
tags: magento2, customer-segments, promotions, cart-price-rules, catalog-price-rules, targeted-offers, personalisation
rank: 12
pathways: [magento2-deep-dive]
---

# 35 - Customer Segments & Promotions

Magento 2's promotion engine is a layered system where customer segmentation provides the targeting intelligence, while cart and catalog price rules provide the discount mechanics. In 2.4.8, these systems are deeply integrated: segments can be assigned to rules, rules can apply to specific customer groups, and scheduled price changes enable time-bound campaigns. This article dissects each layer with real 2.4.8 class references.

## 1. Customer Segments

### The Segment Entity

Customer segments live in the `customer_segment` table. The primary model is `Magento\CustomerSegment\Model\Segment`.

```php
// app/code/Magento/CustomerSegment/Model/Segment.php
namespace Magento\CustomerSegment\Model;

use Magento\Framework\Model\AbstractModel;

class Segment extends AbstractModel
{
    const STATUS_ENABLED = 1;
    const STATUS_DISABLED = 0;

    protected function _construct(): void
    {
        $this->_init(\Magento\CustomerSegment\Model\ResourceModel\Segment::class);
    }

    public function getName(): string
    {
        return (string) $this->getData('name');
    }

    public function getWebsiteIds(): array
    {
        return array_filter(explode(',', (string) $this->getData('website_ids')));
    }

    public function applyToCustomer(\Magento\Customer\Model\Customer $customer): bool
    {
        /** @var \Magento\CustomerSegment\Model\AppliedSegment $applied */
        $applied = $this->_registry->registry('_applied_segment');
        if ($applied !== null) {
            return $applied->checkApply($customer->getId(), $this->getId());
        }
        return $this->_segmentCustomerMatch($customer);
    }
}
```

The `customer_segment` table schema:

```sql
CREATE TABLE customer_segment (
    segment_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    website_ids VARCHAR(255) NOT NULL DEFAULT '0',
    is_active TINYINT(1) DEFAULT 1,
    condition_serialized LONGTEXT,
    condition_config LONGTEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (segment_id)
) ENGINE=InnoDB;
```

### Segment Conditions and Condition Combiner

The segment condition system lives under `Magento\CustomerSegment\Model\Segment\Condition`. The root combiner implements `Magento\Framework\Api\ConditionInterface` (the standard Magento rule condition contract).

```php
// app/code/Magento/CustomerSegment/Model/Segment/Condition/Combine.php
namespace Magento\CustomerSegment\Model\Segment\Condition;

use Magento\Rule\Model\Condition\Combine as RuleCombine;

class Combine extends RuleCombine
{
    protected $name = 'segment_condition';

    public function getDefaultOperatorInputByType(): array
    {
        if (parent::getDefaultOperatorInputByType() === []) {
            return [];
        }
        return parent::getDefaultOperatorInputByType();
    }

    public function getValueSelectRenderer(): ?string
    {
        return null;
    }

    public function getNewChildSelectOptions(): array
    {
        $conditions = parent::getNewChildSelectOptions();
        return array_merge($conditions, [
            // Segment-specific conditions
            ['value' => \Magento\CustomerSegment\Model\Segment\Condition\Customer\Address::class, 'label' => __('Customer Address')],
            ['value' => \Magento\CustomerSegment\Model\Segment\Condition\Customer\Orders::class, 'label' => __('Order History')],
            ['value' => \Magento\CustomerSegment\Model\Segment\Condition\Customer\Visited::class, 'label' => __('Visitation')],
            ['value' => \Magento\CustomerSegment\Model\Segment\Condition\Product::class, 'label' => __('Product in Cart')],
        ]);
    }
}
```

Key condition types in 2.4.8:

| Condition Class | Purpose |
|-----------------|---------|
| `Customer\Address` | Match by billing/shipping address fields (country, region, city, postcode) |
| `Customer\Orders` | Match by order count, total spent, last order date |
| `Customer\Visited` | Match by visit frequency, last visit time |
| `Customer\Orders\Products` | Match by products previously purchased |
| `Product` | Match by products currently in cart |

Example condition tree serialized in DB:

```json
{
  "type": "Magento\\CustomerSegment\\Model\\Segment\\Condition\\Combine",
  "aggregator": "all",
  "value": "1",
  "conditions": [
    {
      "type": "Magento\\CustomerSegment\\Model\\Segment\\Condition\\Customer\\Address",
      "attribute": "country_id",
      "operator": "==",
      "value": "US"
    },
    {
      "type": "Magento\\CustomerSegment\\Model\\Segment\\Condition\\Customer\\Orders",
      "attribute": "total_profit",
      "operator": ">",
      "value": "1000"
    }
  ]
}
```

### Admin UI: Marketing → Segments

The admin UI for segments lives in `Magento\CustomerSegment\Block\Adminhtml\Segment`. The edit form extends `Magento\Backend\Block\Widget\Form\Generic`:

```php
// app/code/Magento/CustomerSegment/Block/Adminhtml/Segment/Edit/Tab/Conditions.php
namespace Magento\CustomerSegment\Block\Adminhtml\Segment\Edit\Tab;

use Magento\Backend\Block\Widget\Form\Generic;
use Magento\Backend\Block\Widget\Tab\TabInterface;

class Conditions extends Generic implements TabInterface
{
    protected $_rendererFieldset;
    protected $_rule;

    public function __construct(
        \Magento\Backend\Block\Context $context,
        \Magento\Framework\Registry $registry,
        \Magento\Framework\Data\FormFactory $formFactory,
        \Magento\Rule\Model\ConditionFactory $conditionFactory,
        array $data = []
    ) {
        parent::__construct($context, $registry, $formFactory, $data);
        $this->_rule = $conditionFactory->create();
    }

    protected function _prepareForm(): void
    {
        $form = $this->_formFactory->create();
        $fieldset = $form->addFieldset('conditions_fieldset', []);

        $fieldset->addField(
            'conditions',
            'text',
            ['name' => 'conditions', 'data-form-part' => $this->getTargetForm()]
        );

        $this->setForm($form);
    }
}
```

Segments apply to customers in two modes:
- **Realtime**: Customer matches conditions on-the-fly during session
- **Scheduled**: `customer_segment` cron job evaluates conditions and stores results in `customer_segment_segment_customer`

```php
// app/code/Magento/CustomerSegment/Model/ResourceModel/Segment.php
class Segment extends \Magento\Framework\Model\ResourceModel\Db\AbstractDb
{
    public function _construct()
    {
        $this->_init('customer_segment', 'segment_id');
    }

    public function getCustomerIds(int $segmentId, ?int $websiteId = null): array
    {
        $connection = $this->getConnection();
        $select = $connection->select()->from(
            ['main_table' => $this->getTable('customer_segment_segment_customer')],
            ['customer_id']
        )->where('segment_id = ?', $segmentId);

        if ($websiteId !== null) {
            $select->where('website_id = ?', $websiteId);
        }

        return $connection->fetchCol($select);
    }
}
```

## 2. Cart Price Rules

### etc/salesrule.xml

Cart price rules are defined via `etc/salesrule.xml` in your module, plus `etc/di.xml` for type preferences.

```xml
<!-- app/code/Magento/SalesRule/etc/salesrule.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_SalesRule:etc/salesrule.xsd">
    <rule name="Magento_SalesRule::salesRule">
        <model>Magento\SalesRule\Model\Rule</model>
        <label>Cart Rule</label>
        <actions_model>Magento\SalesRule\Model\Rule\Action\Collection</actions_model>
        <conditions_model>Magento\SalesRule\Model\Rule\Condition\Combine</conditions_model>
        <valid_children>
            <item name="Magento\SalesRule\Model\Rule\Condition\Address" />
            <item name="Magento\SalesRule\Model\Rule\Condition\Product\Combine" />
        </valid_children>
    </rule>
</config>
```

### The SalesRule Model

`Magento\SalesRule\Model\Rule` extends `Magento\Rule\Model\AbstractRule`:

```php
// app/code/Magento/SalesRule/Model/Rule.php
namespace Magento\SalesRule\Model;

use Magento\Framework\Api\AttributeValueFactory;
use Magento\Framework\Api\ExtensionAttributesFactory;
use Magento\Framework\Data\FormFactory;
use Magento\Framework\Stdlib\DateTime\TimezoneInterface;
use Magento\Rule\Model\AbstractModel;

class Rule extends AbstractModel
{
    const APPLY_TO_CART = 'cart';
    const APPLY_TO_PRODUCT = 'product';
    const DISCOUNT_ACTION_BY_PERCENT = 'by_percent';
    const DISCOUNT_ACTION_BY_FIXED = 'by_fixed';
    const DISCOUNT_ACTION_CART_FIXED = 'cart_fixed';
    const DISCOUNT_ACTION_BUY_X_GET_Y = 'buy_x_get_y';
    const DISCOUNT_ACTION_BUY_X_GET_Y_FIXED = 'buy_x_get_y_fixed';

    protected $_ruleFactory;
    protected $_condCombineFactory;
    protected $_condProdCombineF;
    protected $_couponFactory;
    protected $_couponCodegenFactory;

    protected function _construct(): void
    {
        $this->_init(\Magento\SalesRule\Model\ResourceModel\Rule::class);
    }

    public function getConditions(): \Magento\SalesRule\Model\Rule\Condition\Combine
    {
        return $this->_condCombineFactory->create()->setRule($this);
    }

    public function getActions(): \Magento\SalesRule\Model\Rule\Action\Collection
    {
        return $this->_ruleFactory->create()->setRule($this);
    }

    public function toArray(array $keys = []): array
    {
        $data = parent::toArray($keys);
        $data['conditions'] = $this->getConditions()->serialize();
        $data['actions'] = $this->getActions()->serialize();
        return $data;
    }
}
```

The core `sales_rule` table:

```sql
CREATE TABLE sales_rule (
    rule_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    from_date DATE,
    to_date DATE,
    coupon_type VARCHAR(255) DEFAULT 'NO_COUPON',
    coupon_code VARCHAR(255) NULL,
    use_auto_generation TINYINT(1) DEFAULT 0,
    uses_per_coupon INT DEFAULT 0,
    customer_group_ids VARCHAR(255) NOT NULL DEFAULT '0',
    is_active TINYINT(1) DEFAULT 0,
    stop_rules_processing TINYINT(1) DEFAULT 1,
    is_advanced TINYINT(1) DEFAULT 1,
    product_ids TEXT,
    sort_order INT DEFAULT 0,
    simple_action VARCHAR(255),
    discount_amount DECIMAL(12,4) DEFAULT 0,
    discount_qty INT DEFAULT 0,
    discount_step INT DEFAULT 0,
    apply_to_shipping TINYINT(1) DEFAULT 0,
    times_used INT DEFAULT 0,
    is_system TINYINT(1) DEFAULT 0,
    condition_serialized LONGTEXT,
    action_serialized LONGTEXT,
    website_ids VARCHAR(255) NOT NULL DEFAULT '0',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### Condition SQL Generation

When a cart price rule is validated, conditions are converted to SQL via `\Magento\SalesRule\Model\Rule\Condition\Address::validate()`:

```php
// app/code/Magento/SalesRule/Model/Rule/Condition/Address.php
namespace Magento\SalesRule\Model\Rule\Condition;

use Magento\Rule\Model\Condition\Address as BaseAddress;

class Address extends BaseAddress
{
    public function getAttribute(): string
    {
        return (string) $this->getData('attribute');
    }

    public function getOperator(): ?string
    {
        $op = $this->getData('operator');
        return $op !== null ? (string) $op : null;
    }

    /**
     * Validate address against condition
     * @param \Magento\Framework\DataObject $address
     * @return bool
     */
    public function validate(\Magento\Framework\DataObject $address): bool
    {
        $attr = $this->getAttribute();
        $value = $this->getValueParsed();
        $operator = $this->getOperator();

        $addressValue = $address->getData($attr);
        if ($addressValue === null) {
            return false;
        }

        return $this->validateValue($addressValue, $operator, $value);
    }
}
```

The base condition `Magento\Framework\Model\AbstractModel` provides `validateValue()`:

```php
// Simplified condition validation flow
public function validateValue($value, $operator, $validatorValue): bool
{
    switch ($operator) {
        case '==':
            return $value == $validatorValue;
        case '!=':
            return $value != $validatorValue;
        case '>':
            return $value > $validatorValue;
        case 'gte':
            return $value >= $validatorValue;
        case 'lte':
            return $value <= $validatorValue;
        case 'lt':
            return $value < $validatorValue;
        case 'contains':
            return is_string($value) && strpos($value, (string)$validatorValue) !== false;
        case '!contains':
            return !is_string($value) || strpos($value, (string)$validatorValue) === false;
        case 'finset':
            return is_array($value) && in_array($validatorValue, $value);
        default:
            return $value == $validatorValue;
    }
}
```

The condition tree for cart rules supports attributes like:

| Attribute Code | Description | Example Value |
|---------------|-------------|---------------|
| `base_subtotal` | Cart subtotal | `>100` |
| `total_item_count` | Total items in cart | `>=3` |
| `payment_method` | Selected payment method | `cash_on_delivery` |
| `shipping_method` | Selected shipping method | `flatrate_flatrate` |
| `customer_group_id` | Customer's group | `1` |
| `country_id` | Shipping country | `US` |
| `region_id` | Shipping region | `12` |
| `postcode` | Shipping postcode | `10001` |

### Actions: Buy X Get Y

Buy X get Y is handled by the `actions` model, specifically `\Magento\SalesRule\Model\Rule\Action\Collection`:

```php
// app/code/Magento/SalesRule/Model/Rule/Action/Collection.php
namespace Magento\SalesRule\Model\Rule\Action;

use Magento\Rule\Model\Action\Collection as RuleActionCollection;

class Collection extends RuleActionCollection
{
    public function getNewChildSelectOptions(): array
    {
        return array_merge(
            parent::getNewChildSelectOptions(),
            [
                ['value' => \Magento\SalesRule\Model\Rule\Action\Product::class, 'label' => __('Catalog Item Attribute')],
                ['value' => \Magento\SalesRule\Model\Rule\Action\Discount\Data::class, 'label' => __('Discount Amount')],
            ]
        );
    }
}
```

The buy X get Y logic is implemented in `\Magento\SalesRule\Model\Rule\Action\Discount\BuyXGetY`:

```php
// app/code/Magento/SalesRule/Model/Rule/Action/Discount/BuyXGetY.php
namespace Magento\SalesRule\Model\Rule\Action\Discount;

class BuyXGetY
{
    /**
     * Calculate buy X get Y discount
     * @param \Magento\SalesRule\Model\Rule $rule
     * @param \Magento\Quote\Model\Quote\Item\AbstractItem $item
     * @param float $qty
     * @return float
     */
    public function calculate(\Magento\SalesRule\Model\Rule $rule, $item, $qty): float
    {
        $discount = 0;
        $ruleQty = (int) $rule->getDiscountStep();
        $buyX = $ruleQty > 0 ? $ruleQty : 1;
        $getY = (int) $rule->getDiscountQty();

        if ($getY > 0 && $buyX > 0) {
            $applicableItems = $this->_getApplicableItems($item, $buyX + $getY);
            $freeItems = floor(count($applicableItems) / $buyX) * $getY;
            $discount = $this->_calculateDiscount($item, $freeItems);
        }

        return $discount;
    }
}
```

## 3. Catalog Price Rules

### CatalogRule Model

Catalog price rules apply at the product level, modifying prices in the catalog before items are added to cart. The main model is `Magento\CatalogRule\Model\Rule`:

```php
// app/code/Magento/CatalogRule/Model/Rule.php
namespace Magento\CatalogRule\Model;

use Magento\CatalogRule\Api\Data\RuleInterface;
use Magento\Framework\Model\AbstractModel;

class Rule extends AbstractModel implements RuleInterface
{
    const STATUS_ENABLED = 1;
    const STATUS_DISABLED = 0;

    protected $_eventPrefix = 'catalog_rule';
    protected $_eventObject = 'rule';

    protected $_combineFactory;
    protected $_actionCollectionFactory;

    protected function _construct(): void
    {
        $this->_init(\Magento\CatalogRule\Model\ResourceModel\Rule::class);
    }

    public function getConditionsInstance(): \Magento\CatalogRule\Model\Rule\Condition\Combine
    {
        return $this->_combineFactory->create()->setRule($this);
    }

    public function getActionsInstance(): \Magento\CatalogRule\Model\Rule\Action\Collection
    {
        return $this->_actionCollectionFactory->create()->setRule($this);
    }

    public function matchingProductIds(): array
    {
        $productIds = [];
        $collection = $this->_productCollectionFactory->create();
        $this->getConditions()->applyToCollection($collection);
        foreach ($collection as $product) {
            $productIds[] = $product->getId();
        }
        return $productIds;
    }
}
```

### The CatalogRule Indexer

The catalog rule indexer (`catalogrule_rule`) is a critical component that materializes price calculations into the `catalogrule_rule_price` table:

```php
// app/code/Magento/CatalogRule/Model/Indexer/Rule/RuleProductProcessor.php
namespace Magento\CatalogRule\Model\Indexer\Rule;

class RuleProductProcessor
{
    public const INDEXER_ID = 'catalogrule_rule';

    public function __construct(
        \Magento\CatalogRule\Model\Indexer\Rule $indexer,
        \Magento\Framework\Mview\ViewInterface $view,
        \Magento\CatalogRule\Model\Indexer\Rule\CreateSlatDataProvider $slatDataProvider
    ) {
        $this->_indexer = $indexer;
        $this->_view = $view;
        $this->_slatDataProvider = $slatDataProvider;
    }

    public function reindex(): void
    {
        $this->_indexer->execute();
    }
}
```

The actual indexer implements `\Magento\Framework\Indexer\ActionInterface`:

```php
// app/code/Magento/CatalogRule/Model/Indexer/Rule.php
namespace Magento\CatalogRule\Model\Indexer;

use Magento\Framework\Indexer\ActionInterface;

class Rule implements ActionInterface
{
    public function executeFull(): void
    {
        $this->_ruleProductSorter->sortPriceRules();
        $this->_rulePriceStorage->refresh();
    }

    public function execute($productIds): void
    {
        foreach ($productIds as $productId) {
            $this->_rulePriceStorage->reindex($productId);
        }
    }

    public function executeList($productIds): void
    {
        $this->execute($productIds);
    }

    public function executeRow($productId): void
    {
        $this->execute([$productId]);
    }
}
```

### catalogrule_rule Table

The `catalogrule_rule` table stores rule metadata for the indexer:

```sql
CREATE TABLE catalogrule_rule (
    rule_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    from_date DATE,
    to_date DATE,
    is_active TINYINT(1) DEFAULT 0,
    sort_order INT DEFAULT 0,
    action_operator VARCHAR(10) DEFAULT 'by_percent',
    action_amount DECIMAL(12,4) DEFAULT 0,
    action_stop TINYINT(1) DEFAULT 0,
    conditions_serialized LONGTEXT,
    actions_serialized LONGTEXT,
    customer_group_ids VARCHAR(255) NOT NULL DEFAULT '0',
    website_ids VARCHAR(255) NOT NULL DEFAULT '0',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

The computed prices land in `catalogrule_rule_price`:

```sql
CREATE TABLE catalogrule_rule_price (
    rule_price_id INT AUTO_INCREMENT PRIMARY KEY,
    rule_id INT NOT NULL,
    customer_group_id INT NOT NULL,
    product_id INT NOT NULL,
    rule_date DATE NOT NULL,
    website_id INT NOT NULL,
    price DECIMAL(12,4) NOT NULL DEFAULT 0,
    rule_price_type VARCHAR(255) DEFAULT 'TO_PERCENT',
    rule_weight DECIMAL(12,4) DEFAULT 0,
    earliest_end_date VARCHAR(255) NULL,
    latest_start_date VARCHAR(255) NULL,
    UNIQUE KEY `UNQ_CATALOGRULE_RULE_PRICE` (rule_id, customer_group_id, product_id, rule_date, website_id),
    KEY `IDX_CATALOGRULE_RULE_PRICE_PRODUCT_ID` (product_id),
    KEY `IDX_CATALOGRULE_RULE_PRICE_WEBSITE_ID` (website_id),
    KEY `IDX_CATALOGRULE_RULE_PRICE_RULE_DATE` (rule_date)
);
```

### Scheduled Price Changes

Catalog rules support scheduled activation via `from_date`/`to_date`. The `Magento_CatalogRule\Model\Indexer\Rule` runs via cron at `catalogrule_apply_all`:

```php
// app/code/Magento/CatalogRule/etc/crontab.xml
<?xml version="1.0"?>
<config>
    <group id="default">
        <job name="catalogrule_apply_all" instance="Magento\CatalogRule\Model\Indexer\Rule" method="executeFull">
            <schedule>0 * * * *</schedule>
        </job>
    </group>
</config>
```

When the indexer runs, it recalculates all active rule prices for the current day:

```php
// app/code/Magento/CatalogRule/Model/Indexer/Rule.php (executeFull)
public function executeFull(): void
{
    /** @var \Magento\CatalogRule\Model\ResourceModel\Rule $resourceModel */
    $resourceModel = $this->_resourceRule;
    $connection = $resourceModel->getConnection();

    $select = $connection->select()->from(
        ['main_table' => $resourceModel->getTable('catalogrule_rule')],
        ['*']
    )->where('is_active = ?', 1)
     ->where('from_date <= ?', $this->_date->date('Y-m-d'))
     ->where('to_date >= ?', $this->_date->date('Y-m-d'));

    $rules = $connection->fetchAll($select);

    foreach ($rules as $ruleData) {
        $this->_calculateRulePrice($ruleData);
    }
}
```

## 4. Targeted Promotions

### Segment-Based Rule Assignment

Cart price rules can be assigned to customer segments via the `Magento\SalesRule\Model\Rule::getCustomerGroupIds()` which is matched against the customer's segment membership:

```php
// app/code/Magento/SalesRule/Model/Rule.php
public function getCustomerGroupIds(): array
{
    $groupIds = $this->getData('customer_group_ids');
    if (empty($groupIds)) {
        return [];
    }
    return array_filter(explode(',', (string) $groupIds));
}

public function validate(\Magento\Quote\Model\Quote\Item\AbstractItem $item): bool
{
    $address = $item->getAddress();
    $customerGroupId = $address->getQuote()->getCustomerGroupId();

    if (!in_array($customerGroupId, $this->getCustomerGroupIds())) {
        return false;
    }

    return $this->getConditions()->validate($address);
}
```

For segment-based targeting (not just customer group), the rule checks segment membership via `\Magento\CustomerSegment\Model\Segment\Condition\Combine`:

```php
// app/code/Magento/CustomerSegment/Model/Segment/Condition/Customer/Orders.php
class Orders extends \Magento\CustomerSegment\Model\Segment\Condition\Combine
{
    protected $fieldName = 'orders';

    public function getAttribute(): string
    {
        return $this->fieldName;
    }

    public function validate($customer): bool
    {
        /** @var \Magento\CustomerSegment\Model\ResourceModel\SegmentCustomer $segmentCustomer */
        $segmentCustomer = $this->_segmentCustomerResource;

        $orderCount = $segmentCustomer->getOrderCount($customer->getId());
        $totalSpent = $segmentCustomer->getTotalSpent($customer->getId());

        $value = $this->getValue();
        $operator = $this->getOperator();

        return $this->validateValue($orderCount, $operator, $value);
    }
}
```

### Guest vs Logged-In Targeting

Guest customers are assigned to the "NOT LOGGED IN" customer group (group ID `0`). Cart rules targeting `customer_group_ids = 0` apply to all guests:

```php
// Magento\SalesRule\Model\Rule::applyToQuote()
public function applyToQuote(\Magento\Quote\Model\Quote $quote): void
{
    $customerGroupId = $quote->getCustomerGroupId();

    // For guests, customer_group_id is 0 (NOT LOGGED IN)
    if ($customerGroupId === 0) {
        // Check if rule applies to guest group
        if (!in_array(0, $this->getCustomerGroupIds())) {
            return false;
        }
    }

    // For logged-in customers
    if (!in_array($customerGroupId, $this->getCustomerGroupIds())) {
        return false;
    }

    return $this->getConditions()->validate($quote->getShippingAddress());
}
```

Catalog rules target guests via the same `customer_group_ids` mechanism. In the `catalogrule_rule_price` table, a `customer_group_id = 0` entry means the rule price applies to guests:

```sql
-- All products get 10% off for guests
INSERT INTO catalogrule_rule_price (rule_id, customer_group_id, product_id, rule_date, website_id, price)
SELECT r.rule_id, 0, p.entity_id, CURDATE(), w.website_id, p.price * 0.9
FROM catalogrule_rule r
CROSS JOIN catalog_product_entity p
CROSS JOIN store_website w
WHERE r.is_active = 1 AND r.from_date <= CURDATE() AND (r.to_date >= CURDATE() OR r.to_date IS NULL);
```

### Matching Logic: Cart Rules vs Catalog Rules

| Aspect | Cart Price Rules | Catalog Price Rules |
|--------|-----------------|---------------------|
| Application | After add to cart | Before add to cart |
| Condition base | Cart/quote address | Product attributes |
| Price modification | Discount applied at checkout | Displayed price on PDP/category |
| Guest support | Via customer_group_id=0 | Via customer_group_id=0 |
| Segment-based | Via condition tree | Via condition tree |
| Indexer | None (runtime evaluation) | `catalogrule_rule` indexer |

Cart rules are evaluated at quote validation time in `Magento\SalesRule\Model\Validator`:

```php
// app/code/Magento/SalesRule/Model/Validator.php
public function process(\Magento\Quote\Model\Quote $quote): void
{
    $rules = $this->_ruleCollectionFactory->create()
        ->setValidationFilter($quote->getWebsiteId(), $quote->getCustomerGroupId());

    foreach ($rules as $rule) {
        if (!$rule->getIsActive()) {
            continue;
        }

        if ($rule->validate($quote)) {
            $rule->process($quote);
        }
    }
}
```

Catalog rules are pre-indexed and applied via price engine:

```php
// app/code/Magento/CatalogRule\Pricing\PriceResolver.php
public function resolvePrice(\Magento\Catalog\Model\Product $product, int $qty): float
{
    $price = $product->getPrice();
    $rulePrice = $this->_rulePriceStorage->getRulePrice(
        $product->getId(),
        $this->_customerGroupId,
        $this->_websiteId
    );

    if ($rulePrice !== null) {
        return min($price, $rulePrice);
    }

    return $price;
}
```

## 5. Promotion CLI Commands

### salesrule:create

Magento provides `bin/magento salesrule:create` to create cart price rules from CLI arguments:

```php
// app/code/Magento/SalesRule/Console/Command/CreateCommand.php
namespace Magento\SalesRule\Console\Command;

use Symfony\Component\Console\Command\Command;
use Magento\Framework\Console\HelperInterface;

class CreateCommand extends Command
{
    protected $_ruleFactory;
    protected $_conditionFactory;

    public function __construct(
        \Magento\SalesRule\Model\RuleFactory $ruleFactory,
        \Magento\SalesRule\Model\Rule\Condition\CombineFactory $conditionFactory
    ) {
        $this->_ruleFactory = $ruleFactory;
        $this->_conditionFactory = $conditionFactory;
        parent::__construct();
    }

    protected function configure(): void
    {
        $this->setName('salesrule:create')
            ->setDescription('Create a sales rule')
            ->addArgument('name', InputArgument::REQUIRED)
            ->addArgument('discount_amount', InputArgument::REQUIRED)
            ->addArgument('customer_group_ids', InputArgument::OPTIONAL);

        parent::configure();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        /** @var \Magento\SalesRule\Model\Rule $rule */
        $rule = $this->_ruleFactory->create();

        $rule->setName($input->getArgument('name'))
            ->setDiscountAmount($input->getArgument('discount_amount'))
            ->setCustomerGroupIds($input->getArgument('customer_group_ids') ?? '1')
            ->setWebsiteIds('1')
            ->setIsActive(1)
            ->setSimpleAction(\Magento\SalesRule\Model\Rule::DISCOUNT_ACTION_BY_PERCENT)
            ->setStopRulesProcessing(0);

        $rule->save();
        $output->writeln(sprintf('Rule created with ID: %d', $rule->getId()));

        return Command::SUCCESS;
    }
}
```

### catalogrule:apply

For catalog rules, `bin/magento catalogrule:apply` triggers immediate rule application:

```php
// app/code/Magento/CatalogRule/Console/Command/ApplyCommand.php
namespace Magento\CatalogRule\Console/Command;

class ApplyCommand extends Command
{
    protected $_ruleFactory;

    protected function configure(): void
    {
        $this->setName('catalogrule:apply')
            ->setDescription('Apply all catalog rules');

        parent::configure();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        /** @var \Magento\CatalogRule\Model\Indexer\Rule $indexer */
        $indexer = $this->_indexerRegistry->get('catalogrule_rule');

        $output->writeln('Applying catalog rules...');
        $indexer->executeFull();

        $output->writeln('Catalog rules applied successfully.');

        return Command::SUCCESS;
    }
}
```

### indexer:reindex catalogrule_rule

The catalog rule indexer can be triggered manually:

```bash
# Reindex single indexer
bin/magento indexer:reindex catalogrule_rule

# Reindex all indexers
bin/magento indexer:reindex

# Check indexer status
bin/magento indexer:status
```

In 2.4.8, the indexer configuration in `etc/indexer.xml`:

```xml
<!-- app/code/Magento/CatalogRule/etc/indexer.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Indexer/etc/indexer.xsd">
    <indexer id="catalogrule_rule"
             view_id="catalogrule_rule"
             class="Magento\CatalogRule\Model\Indexer\Rule"
             model="Magento\CatalogRule\Model\Indexer\Rule">
        <title translate="true">Catalog Rule Product</title>
        <description translate="true">Update catalog rule price for affected products</description>
    </indexer>
</config>
```

### Full Workflow: Creating and Applying a Catalog Rule

```bash
# 1. Create rule via CLI (if custom command exists)
# Or create via admin: Marketing → Catalog Price Rules

# 2. Apply rules immediately
bin/magento catalogrule:apply

# 3. Or let cron handle it (default: every hour)
# Crontab entry: 0 * * * * php bin/magento cron:run --group=default

# 4. Verify indexer status
bin/magento indexer:status catalogrule_rule

# 5. Check prices in database
SELECT * FROM catalogrule_rule_price WHERE rule_date = CURDATE();
```

## 6. Custom Promotion Condition

### ConditionInterface

To create a custom condition class, implement `Magento\Framework\Api\ConditionInterface`:

```php
// app/code/Vendor/CustomPromotion/Model/Condition/PurchaseHistory.php
namespace Vendor\CustomPromotion\Model\Condition;

use Magento\Framework\Api\ConditionInterface;
use Magento\Framework\DataObject;
use Magento\Rule\Model\Condition\AbstractCondition;

class PurchaseHistory extends AbstractCondition implements ConditionInterface
{
    protected $fieldName = 'purchase_history';

    /**
     * @return string
     */
    public function getAttribute(): string
    {
        return $this->fieldName;
    }

    /**
     * Validate customer against purchase history condition
     * @param DataObject $customer
     * @return bool
     */
    public function validate(DataObject $customer): bool
    {
        $value = (float) $this->getValue();
        $operator = $this->getOperator();

        /** @var \Magento\Sales\Model\ResourceModel\Order\Collection $orderCollection */
        $orderCollection = $this->_orderCollectionFactory->create()
            ->addFieldToFilter('customer_id', $customer->getId());

        $totalSpent = 0;
        foreach ($orderCollection as $order) {
            $totalSpent += $order->getGrandTotal();
        }

        return $this->validateValue($totalSpent, $operator, $value);
    }

    /**
     * Get input type for admin condition builder
     * @return string
     */
    public function getInputType(): string
    {
        return 'numeric';
    }

    /**
     * Get value element type for admin form
     * @return string
     */
    public function getValueElementType(): string
    {
        return 'text';
    }

    /**
     * Get condition label for admin
     * @return string
     */
    public function getDefaultLabel(): string
    {
        return __('Total Purchased Greater Than');
    }

    /**
     * Build admin form element for this condition
     * @return \Magento\Framework\Data\Form\Element\AbstractElement
     */
    public function getValueElement(): \Magento\Framework\Data\Form\Element\AbstractElement
    {
        $element = parent::getValueElement();
        $element->setClass('validate-number');
        $element->setPlaceholder(__('Enter amount'));
        return $element;
    }

    /**
     * Get attribute options for dropdown
     * @return array
     */
    public function getAttributeOptions(): array
    {
        return [
            ['value' => 'total_profit', 'label' => __('Total Spent')],
            ['value' => 'order_count', 'label' => __('Order Count')],
            ['value' => 'avg_order_value', 'label' => __('Average Order Value')],
        ];
    }
}
```

### Register the Condition in di.xml

```xml
<!-- app/code/Vendor/CustomPromotion/etc/di.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Register condition for SalesRule -->
    <type name="Magento\SalesRule\Model\Rule\Condition\Combine">
        <plugin name="vendor_custom_promotion_add_conditions" type="Vendor\CustomPromotion\Plugin\AddConditionsPlugin" />
    </type>

    <!-- Register as shared instance -->
    <type name="Vendor\CustomPromotion\Model\Condition\PurchaseHistory">
        <arguments>
            <argument name="orderCollectionFactory" xsi:type="object">Magento\Sales\Model\ResourceModel\Order\CollectionFactory</argument>
        </arguments>
    </type>

</config>
```

### Plugin to Add Condition to Builder

```php
// app/code/Vendor/CustomPromotion/Plugin/AddConditionsPlugin.php
namespace Vendor\CustomPromotion\Plugin;

class AddConditionsPlugin
{
    public function beforeGetNewChildSelectOptions(
        \Magento\SalesRule\Model\Rule\Condition\Combine $subject,
        $result
    ) {
        return array_merge($result ?? [], [
            ['value' => \Vendor\CustomPromotion\Model\Condition\PurchaseHistory::class, 'label' => __('Purchase History')],
        ]);
    }
}
```

### Use in Rule Condition Tree

Once registered, the custom condition appears in the rule builder dropdown. When saved, the condition is serialized into `condition_serialized`:

```json
{
  "type": "Magento\\SalesRule\\Model\\Rule\\Condition\\Combine",
  "aggregator": "all",
  "value": "1",
  "conditions": [
    {
      "type": "Vendor\\CustomPromotion\\Model\\Condition\\PurchaseHistory",
      "attribute": "total_profit",
      "operator": ">=",
      "value": "500"
    }
  ]
}
```

### Validation Flow

When a cart is validated, the condition chain processes:

```
1. Quote created → Validator::process() called
2. Load active rules: SalesRule/Model/ResourceModel/Rule/Collection::setValidationFilter()
3. For each rule:
   a. Rule::validate($quote) → checks customer_group_ids
   b. Rule::getConditions()->validate($quote->getShippingAddress())
   c. Custom conditions receive DataObject with customer data
   d. Condition::validate($customer) → executes custom business logic
4. Matching rules apply discounts
```

### Key Methods in ConditionInterface

The `ConditionInterface` contract requires:

```php
// Magento\Framework\Api\ConditionInterface
interface ConditionInterface
{
    /**
     * Validate that condition matches given entity
     * @param \Magento\Framework\DataObject $entity
     * @return bool
     */
    public function validate(\Magento\Framework\DataObject $entity): bool;
}
```

`Magento\Rule\Model\Condition\AbstractCondition` provides the full implementation base including:
- `getAttribute()`, `getOperator()`, `getValue()` — condition parameters
- `validateValue()` — operator-based validation (==, !=, >, <, >=, <=, contains, etc.)
- `loadArray()`, `asArray()` — serialization support
- `getNewChildSelectOptions()` — admin UI dropdown population

## Summary

Magento 2.4.8's promotion system operates as an integrated pipeline:

1. **Customer Segments** — Dynamic classification via `\Magento\CustomerSegment\Model\Segment`. Conditions evaluate customer data in realtime or via cron-assigned batch (`customer_segment_segment_customer` table).

2. **Cart Price Rules** — `\Magento\SalesRule\Model\Rule` with `conditions_serialized`/`action_serialized` LONGTEXT columns. Validation runs at checkout via `\Magento\SalesRule\Model\Validator`. Buy X get Y implements complex discount math in `\Magento\SalesRule\Model\Rule\Action\Discount\BuyXGetY`.

3. **Catalog Price Rules** — Pre-indexed into `catalogrule_rule_price` via `catalogrule_rule` indexer. `\Magento\CatalogRule\Model\Rule` stores conditions in `conditions_serialized`. Scheduled activation via `from_date`/`to_date` columns.

4. **Targeting** — Both rule types use `customer_group_ids` (VARCHAR, comma-separated) for group-based targeting. Customer segments add behavioral/demographic conditions via the `\Magento\CustomerSegment` condition tree.

5. **CLI** — `catalogrule:apply` triggers immediate indexing. `indexer:reindex catalogrule_rule` runs the indexer manually. Cron runs `catalogrule_apply_all` on schedule.

6. **Custom Conditions** — Implement `ConditionInterface` (or extend `AbstractCondition`) and register via `di.xml` plugin on the appropriate Combine class. The condition appears in the rule builder UI and is serialized into the rule's condition tree.