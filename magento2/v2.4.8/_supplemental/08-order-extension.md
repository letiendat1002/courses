---
title: "06 - Order Extension"
description: "Deep dive into Magento 2.4.8 order grid extension patterns: grid collection modification, UI components, custom buttons, and ACL configuration"
tags:
  - magento2
  - order-grid
  - ui-components
  - di-xml
  - acl
rank: 8
pathways:
  - magento2-deep-dive
---

# Order Extension

Extending the order management UI in Magento 2.4.8 requires understanding the modern UI Component architecture. This article covers four extension points: adding columns to the order grid, modifying order view data, adding custom buttons, and configuring ACL resources.

> **Deprecation Note**: `fieldset.xml` was removed in Magento 2.3. The `Magento\Framework\Data\Argument\Interpreter\ModuleDepends` class does not exist. All form modifications now use UI Components.

---

## 1. Extending the Order Grid Column Set

### The Wrong Approach (ColumnPool)

The original article referenced `Magento\Ui\Component\Wrapper\UiCompiler` and `ColumnPool` — **these classes do not exist in Magento 2.4.8**. Do not use them.

### The Correct Approach: Grid Collection Modification

The order grid is rendered by `Magento\Sales\Model\ResourceModel\Order\Grid\Collection`. To add custom columns, you modify this collection via `di.xml` preference.

#### Step 1: Create Custom Grid Collection

```php
// app/code/Vendor/Module/Model/ResourceModel/Order/Grid/CustomCollection.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\ResourceModel\Order\Grid;

use Magento\Framework\Data\Collection\Db\FetchStrategyInterface;
use Magento\Framework\Data\Collection\EntityFactoryInterface;
use Magento\Framework\Event\ManagerInterface;
use Magento\Framework\Model\ResourceModel\Db\VersionControl\Snapshot;
use Magento\Sales\Model\ResourceModel\Order\Grid\Collection as SalesGridCollection;
use Psr\Log\LoggerInterface;

class CustomCollection extends SalesGridCollection
{
    /**
     * @param EntityFactoryInterface $entityFactory
     * @param LoggerInterface $logger
     * @param FetchStrategyInterface $fetchStrategy
     * @param ManagerInterface $eventManager
     * @param Snapshot $entitySnapshot
     * @param \Magento\Framework\Stdlib\DateTime\TimezoneInterface $timezone
     * @param \Magento\Sales\Model\ResourceModel\Order $orderResource
     */
    public function __construct(
        EntityFactoryInterface $entityFactory,
        LoggerInterface $logger,
        FetchStrategyInterface $fetchStrategy,
        ManagerInterface $eventManager,
        Snapshot $entitySnapshot,
        \Magento\Framework\Stdlib\DateTime\TimezoneInterface $timezone,
        \Magento\Sales\Model\ResourceModel\Order $orderResource
    ) {
        parent::__construct(
            $entityFactory,
            $logger,
            $fetchStrategy,
            $eventManager,
            $entitySnapshot,
            $timezone,
            $orderResource
        );
    }

    /**
     * Join custom table to add column to grid
     *
     * @return $this
     */
    protected function _initSelect(): void
    {
        parent::_initSelect();

        // Example: Join a custom 'sales_order_extension' table
        $this->getSelect()->joinLeft(
            ['ext' => $this->getTable('sales_order_extension')],
            'main_table.entity_id = ext.order_id',
            ['custom_column', 'shipping_assignment_id']
        );

        return $this;
    }
}
```

#### Step 2: Declare the Preference in di.xml

```xml
<!-- app/code/Vendor/Module/etc/di.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <preference for="Magento\Sales\Model\ResourceModel\Order\Grid\Collection"
                type="Vendor\Module\Model\ResourceModel\Order\Grid\CustomCollection"/>
</config>
```

#### Step 3: Add Column to UI Component XML

The grid UI Component XML is `sales_order_grid.xml`:

```xml
<!-- app/code/Vendor/Module/view/adminhtml/ui_component/sales_order_grid.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<listing xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="urn:magento:module:Ui/etc/ui_configuration.xsd">
    <columns name="columns">
        <column name="custom_column" sortOrder="100">
            <settings>
                <filter>text</filter>
                <label translate="true">Custom Column</label>
                <visible>true</visible>
            </settings>
        </column>
        <column name="shipping_assignment_id" sortOrder="110">
            <settings>
                <filter>text</filter>
                <label translate="true">Shipping Assignment</label>
                <visible>true</visible>
            </settings>
        </column>
    </columns>
</listing>
```

#### Step 4: Register the UI Component

```xml
<!-- app/code/Vendor/Module/etc/adminhtml/di.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Framework\View\Element\UiComponentFactory">
        <arguments>
            <argument name="js_config" xsi:type="array">
                <argument name="paths" xsi:type="array">
                    <argument name="Vendor_Module::sales_order_grid"
                               xsi:type="string">Vendor_Module/js/sales_order_grid</argument>
                </argument>
            </argument>
        </arguments>
    </type>
</config>
```

---

## 2. Modifying Order View Data (Deprecated fieldset.xml)

### The Wrong Approach (fieldset.xml)

The `fieldset.xml` was **deprecated in Magento 2.3 and removed entirely**. The class `Magento\Framework\Data\Argument\Interpreter\ModuleDepends` does not exist.

### The Correct Approach: UI Component Form Modification

For order view page modifications, use the `sales_order_view.xml` UI Component.

#### Create Order View Form Extension

```xml
<!-- app/code/Vendor/Module/view/adminhtml/layout/sales_order_view.xml -->
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceBlock name="order_info">
            <block class="Vendor\Module\Block\Order\CustomInfo" template="Vendor_Module::order/custom_info.phtml"/>
        </referenceBlock>

        <!-- Add custom fieldset to order view -->
        <referenceBlock name="order_tab_info">
            <block class="Vendor\Module\Block\Order\CustomFieldset" template="Vendor_Module::order/custom_fieldset.phtml"
                   name="custom_order_fieldset"/>
        </referenceBlock>
    </body>
</page>
```

#### Create Block for Custom Data Display

```php
// app/code/Vendor/Module/Block/Order/CustomInfo.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Block\Order;

use Magento\Framework\View\Element\Template;
use Magento\Framework\View\Element\Template\Context;
use Magento\Sales\Model\Order;

class CustomInfo extends Template
{
    /**
     * @param Context $context
     * @param array $data
     */
    public function __construct(
        Context $context,
        array $data = []
    ) {
        parent::__construct($context, $data);
    }

    /**
     * Get order from registry
     *
     * @return Order|null
     */
    public function getOrder(): ?Order
    {
        return $this->_coreRegistry->registry('current_order');
    }

    /**
     * Get custom data from extension attribute
     *
     * @return string|null
     */
    public function getCustomData(): ?string
    {
        $order = $this->getOrder();
        if ($order && $order->getExtensionAttributes()) {
            $extensionAttributes = $order->getExtensionAttributes();
            return $extensionAttributes->getCustomData();
        }
        return null;
    }
}
```

#### Preference for Order Data Modification

```xml
<!-- app/code/Vendor/Module/etc/di.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <!-- Preference for order data modification -->
    <preference for="Magento\Sales\Api\OrderRepositoryInterface"
                type="Vendor\Module\Model\OrderRepositoryDecorator"/>

    <!-- Plugin for order loading -->
    <type name="Magento\Sales\Model\Order">
        <plugin name="vendor_module_order_plugin" type="Vendor\Module\Plugin\OrderPlugin"/>
    </type>
</config>
```

---

## 3. Adding Custom Buttons to Order View

### The Wrong Approach (removeButton/BlockProvider)

`removeButton()` from Layout is not the correct pattern for M2.4.8. The `BlockProvider` class mentioned does not exist in the way described.

### The Correct Approach: Plugin on Order Details Block

For order detail page buttons, create a plugin on `Magento\Sales\Block\Order\Details`.

#### Step 1: Create Plugin Class

```php
// app/code/Vendor/Module/Plugin/Sales/Block/Order/DetailsPlugin.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Sales\Block\Order;

use Magento\Backend\Block\Widget\Button\ButtonList;
use Magento\Backend\Block\Widget\Button\Toolbar;
use Magento\Framework\View\Element\AbstractBlock;
use Magento\Sales\Block\Order\Details as OrderDetails;

class DetailsPlugin
{
    /**
     * Before set toolbar: add custom button
     *
     * @param OrderDetails $subject
     * @param AbstractBlock $toolbar
     * @return void
     */
    public function beforeSetToolbar(
        OrderDetails $subject,
        AbstractBlock $toolbar
    ): void {
        // Add custom button after existing buttons
        if ($this->canShowButton($subject)) {
            $buttonList = $subject->getButtonList();
            $buttonList->add(
                'custom_cancel',
                [
                    'class' => 'cancel',
                    'label' => __('Custom Cancel Order'),
                    'onclick' => 'customCancelAction()',
                    'sort_order' => 90
                ],
                0, // position relative to other buttons
                'after' // placement
            );
        }
    }

    /**
     * Check if button should be displayed
     *
     * @param OrderDetails $subject
     * @return bool
     */
    private function canShowButton(OrderDetails $subject): bool
    {
        $order = $subject->getOrder();
        if ($order) {
            // Only show for pending orders
            return $order->getStatus() === 'pending';
        }
        return false;
    }
}
```

#### Step 2: Declare the Plugin in di.xml

```xml
<!-- app/code/Vendor/Module/etc/di.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Sales\Block\Order\Details">
        <plugin name="Vendor_Module_DetailsPlugin"
                type="Vendor\Module\Plugin\Sales\Block\Order\DetailsPlugin"/>
    </type>
</config>
```

#### Alternative: Direct Block Preference

For more control, create a preference for the block itself:

```xml
<!-- app/code/Vendor/Module/etc/di.xml -->
<preference for="Magento\Sales\Block\Order\Details"
            type="Vendor\Module\Block\Order\CustomDetails"/>
```

```php
// app/code/Vendor/Module/Block/Order/CustomDetails.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Block\Order;

use Magento\Sales\Block\Order\Details as DetailsBlock;

class CustomDetails extends DetailsBlock
{
    /**
     * Constructor adds buttons via _construct
     *
     * @return void
     */
    protected function _construct(): void
    {
        parent::_construct();
        $this->addButton('custom_action', [
            'class' => 'primary',
            'label' => __('Custom Action'),
            'on_click' => 'customAction()',
            'sort_order' => 100
        ]);
    }

    /**
     * Add button using layout method
     *
     * @param string $buttonId
     * @param array $buttonData
     * @return $this
     */
    public function addButton(string $buttonId, array $buttonData): self
    {
        $this->buttonList->add($buttonId, $buttonData);
        return $this;
    }
}
```

#### Button Namespace Reference

Use the correct namespace for buttons:
- **Class**: `Magento\Backend\Block\Widget\Button`
- **Button List**: `Magento\Backend\Block\Widget\Button\ButtonList`
- **Toolbar**: `Magento\Backend\Block\Widget\Button\Toolbar`

---

## 4. ACL (Access Control List) Configuration

### The Wrong Approach (sales_operation resource)

`sales_operation` is **not a valid Magento ACL resource**. The real resources are `Magento_Sales::sales`, `Magento_Sales::sales_order`, and `Magento_Sales::actions`.

### The Correct Approach: Proper acl.xml and menu.xml

#### Step 1: Define ACL Resources

```xml
<!-- app/code/Vendor/Module/etc/acl.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Acl/etc/acl.xsd">
    <acl>
        <resources>
            <!-- Top-level Sales resource -->
            <resource id="Magento_Sales::sales" title="Sales" sortOrder="10">
                <!-- Orders resource with actions children -->
                <resource id="Magento_Sales::sales_order" title="Orders" sortOrder="10">
                    <!-- Action-level resources for granular control -->
                    <resource id="Vendor_Module::cancel" title="Cancel Orders" sortOrder="10"/>
                    <resource id="Vendor_Module::custom_action" title="Custom Actions" sortOrder="20"/>
                </resource>

                <!-- Custom operations under sales -->
                <resource id="Vendor_Module::custom_operation" title="Custom Operations" sortOrder="50">
                    <resource id="Vendor_Module::custom_operation_view" title="View" sortOrder="10"/>
                    <resource id="Vendor_Module::custom_operation_edit" title="Edit" sortOrder="20"/>
                </resource>
            </resource>
        </resources>
    </acl>
</config>
```

#### Step 2: Create Admin Menu

```xml
<!-- app/code/Vendor/Module/etc/adminhtml/menu.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:backend:etc/menu.xsd">
    <menu>
        <!-- Add under Sales menu -->
        <add id="Vendor_Module::sales_custom" title="Custom Orders" resource="Vendor_Module::custom_operation"
             module="Vendor_Module" sortOrder="100">
            <!-- Child menu items -->
            <add id="Vendor_Module::sales_custom_index" title="All Custom Orders"
                 action="custom/order/index" resource="Vendor_Module::custom_operation_view"
                 module="Vendor_Module" sortOrder="10"/>
            <add id="Vendor_Module::sales_custom_report" title="Custom Report"
                 action="custom/order/report" resource="Vendor_Module::custom_operation_edit"
                 module="Vendor_Module" sortOrder="20"/>
        </add>
    </menu>
</config>
```

#### Step 3: Reference ACL in Controller

```php
// app/code/Vendor/Module/Controller/Adminhtml/Order/CustomAction.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Controller\Adminhtml\Order;

use Magento\Framework\App\Action\HttpGetActionInterface;
use Magento\Sales\Controller\Adminhtml\Order as OrderController;

class CustomAction extends OrderController implements HttpGetActionInterface
{
    /**
     * Authorization level of a basic admin session
     *
     * @see _isAllowed()
     */
    const ADMIN_RESOURCE = 'Vendor_Module::cancel';

    /**
     * Execute custom action
     *
     * @return \Magento\Framework\View\Result\Page
     */
    public function execute(): \Magento\Framework\View\Result\Page
    {
        $resultPage = $this->resultPageFactory->create();
        $resultPage->getConfig()->getTitle()->prepend(__('Custom Order Action'));
        return $resultPage;
    }

    /**
     * Check if allowed access
     *
     * @return bool
     */
    protected function _isAllowed(): bool
    {
        return $this->_authorization->isAllowed('Vendor_Module::cancel');
    }
}
```

#### Standard Magento Sales ACL Resources

| Resource Identifier | Description |
|---------------------|-------------|
| `Magento_Sales::sales` | Top-level Sales menu access |
| `Magento_Sales::sales_order` | Order management access |
| `Magento_Sales::actions` | General sales actions |
| `Magento_Sales::create` | Create orders |
| `Magento_Sales::view` | View orders |
| `Magento_Sales::edit` | Edit orders |
| `Magento_Sales::cancel` | Cancel orders |

---

## Summary: Correct Patterns for M2.4.8

| Extension Point | Correct Approach | Wrong Approach |
|-----------------|-----------------|---------------|
| **Grid Column** | `di.xml` preference on `Grid\Collection` + UI Component XML | `ColumnPool`, `UiCompiler` |
| **Order View Data** | UI Component `sales_order_view.xml` + Block plugin | `fieldset.xml`, `ModuleDepends` |
| **Custom Buttons** | Plugin on `Magento\Sales\Block\Order\Details` | `removeButton()`, `BlockProvider` |
| **ACL** | Real resources in `etc/acl.xml` + `etc/adminhtml/menu.xml` | `sales_operation` |

Always use dependency injection preferences or plugins to modify core behavior. UI Components handle all frontend rendering in modern Magento 2.4.8.