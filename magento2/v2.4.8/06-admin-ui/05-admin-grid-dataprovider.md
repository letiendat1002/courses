---
title: "05 - Admin UI Grids & DataProviders"
description: "Magento 2.4.8 admin UI component grids: ui_component listings, DataProvider pattern, SearchResult, actionsColumn, mass actions, and admin grid XML configuration"
tags: [magento2, admin, ui-components, dataprovider, grid, adminhtml, ui-component]
rank: 5
pathways: [magento2-deep-dive]
see_also:
  - "Admin routing, menus, ACL: ../README.md"
  - "Full admin UI reference: _supplemental/07-security-acl.md"
---

# Admin UI Component Grids and DataProviders

Magento 2's admin panel builds all listing grids (products, orders, customers, and custom entities) through a **declarative XML + JavaScript** approach. No PHP templates are rendered for the grid structure itself — the XML configuration in `view/adminhtml/ui_component/*.xml` drives the entire UI component tree, and JavaScript components (from `Magento_Ui` module) handle interactivity. Understanding this pattern is essential for building custom admin grids.

---

## 1. UI Component Architecture Overview

### 1.1 File Location

UI component listing XML files live at:

```
app/code/<Vendor>/<Module>/view/adminhtml/ui_component/<entity>_listing.xml
```

The component name is derived from the filename — `customer_listing.xml` creates a UI component registered as `customer_listing`. This name is used when referencing the grid in layouts and controllers.

### 1.2 Component Tree

A UI component grid is a **tree of nested components**. Each node in the tree is a declarative UI component whose configuration flows from XML into JavaScript. The typical structure of a listing grid:

```
dataSource (Magento\Ui\Component\DataProvider)
  └─ columns (uiCollection)
       ├─ text column "name"
       ├─ text column "status"
       ├─ actionsColumn "actions"
       ├─ checkbox column "_select"
       └─ date column "created_at"

paging
filters
bookmark
```

The **dataSource** is the root data provider for the grid. It sits at the top of the component configuration tree and feeds data down to child components (columns, filters, etc.) via the UI component data exchange mechanism.

### 1.3 How XML Becomes a Component

Magento's `Magento\Ui\Model\Manager` (called by the layout XML processing) reads each `<uiComponent>` node in layout XML. For each component declared there, it instantiates the class, passes the `<argument name="data">` block as configuration, and builds the component tree. The XML arguments become the component's `getData('config')` values.

The schema for all UI component XML files is:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<listing xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Ui:etc/ui_configuration.xsd">
```

This schema is validated during module compilation (`bin/magento setup:di:compile`).

---

## 2. The DataProvider Pattern

### 2.1 Base Interface: `Magento\Framework\Api\SearchResultsInterface`

Every grid in Magento is backed by a **search result** — a collection-like object that implements `SearchResultsInterface`. This is the standard return type for list API operations in Magento's API architecture.

```php
<?php
namespace Magento\Framework\Api;

interface SearchResultsInterface extends ExtensibleDataInterface
{
    public function getItems(): array;
    public function setItems(array $items): SearchResultsInterface;
    public function getSearchCriteria();
    public function setSearchCriteria(\Magento\Framework\Api\SearchCriteriaInterface $searchCriteria);
    public function getTotalCount(): int;
    public function setTotalCount(int $totalCount): SearchResultsInterface;
}
```

### 2.2 `Magento\Ui\Component\DataProvider`

`Magento\Ui\Component\DataProvider` is the base class that glues a UI component grid to a collection. It:

1. Holds a **registry** of named data providers
2. Accepts a **collection** (E.g., `Magento\Cms\Model\ResourceModel\Page\Collection`)
3. Applies **SearchCriteria** from the UI component (sorting, filtering, pagination)
4. Returns a `SearchResultsInterface` as the data source for the listing

```php
<?php
namespace Magento\Ui\Component;

class DataProvider extends AbstractDataProvider
{
    // $collection must implement \Magento\Framework\Data\Collection\DataSourceInterface
    // or be a regular collection that SearchResult can wrap
}
```

**Key detail:** `DataProvider` does NOT require the collection to return `SearchResultsInterface` directly. It wraps the collection's items into a `SearchResults` object internally, applying the search criteria.

### 2.3 Custom DataProvider

For a custom entity, you typically create a dedicated DataProvider:

```php
<?php
namespace Vendor\Module\Ui\DataProvider;

use Magento\Ui\Component\DataProvider as UiDataProvider;
use Vendor\Module\Model\ResourceModel\MyEntity\CollectionFactory;

class MyEntityDataProvider extends UiDataProvider
{
    public function __construct(
        $name,
        $primaryFieldName,
        $requestFieldName,
        CollectionFactory $collectionFactory,
        array $meta = [],
        array $data = []
    ) {
        parent::__construct($name, $primaryFieldName, $requestFieldName, $meta, $data);
        $this->collection = $collectionFactory->create();
    }
}
```

Register the DataProvider in `di.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <type name="Magento\Ui\Model\Manager">
        <arguments>
            <argument name="providers" xsi:type="array">
                <argument name="vendor_module_myentity" xsi:type="object">
                    Vendor\Module\Ui\DataProvider\MyEntityDataProvider
                </argument>
            </argument>
        </arguments>
    </type>

</config>
```

The `dataProvider` name (`vendor_module_myentity`) must match the `dataSource` reference in the XML file.

### 2.4 How DataProvider Connects to the Collection

When the grid loads, the UI component JS sends an AJAX request to `mui/index/render` with the component name. This dispatches to `Magento\Ui\Controller\Adminhtml\Index\Render` which:

1. Reads the component name from the request
2. Looks up the registered `DataProvider` by name
3. Calls `DataProvider::getData()` which:
   - Builds a `SearchCriteria` from URL params (`filters`, `sorting`, `pageSize`, `currentPage`)
   - Calls `$this->collection->setSearchCriteria($searchCriteria)`
   - Returns `SearchResultsInterface` with the filtered, sorted, paginated items

---

## 3. `ui_component` Listing XML — Complete Anatomy

### 3.1 `<listing>` Root Element

```xml
<?xml version="1.0" encoding="UTF-8"?>
<listing xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Ui:etc/ui_configuration.xsd">
    <argument name="data" xsi:type="array">
        <item name="js_config" xsi:type="array">
            <item name="provider" xsi:type="string">vendor_module_myentity.vendor_module_myentity_data_source</item>
            <item name="deps" xsi:type="array">
                <item name="vendor_module_myentity.vendor_module_myentity_data_source"
                      xsi:type="string">vendor_module_myentity.vendor_module_myentity_data_source</item>
            </item>
        </item>
    </argument>
</listing>
```

| Attribute | Description |
|-----------|-------------|
| `class` | Optional PHP class override for the listing container (rarely needed) |
| `component` | Path to a JS component to render the listing (defaults to `Magento_Ui/js/form/components/collection`) |
| `cacheifetime` | Cache lifetime in seconds for the rendered grid |
| `resource` | ACL resource required to view the grid |

### 3.2 `<dataSource>`

The `dataSource` links the listing to a registered DataProvider:

```xml
<dataSource name="vendor_module_myentity_data_source" component="Magento_Ui/js/form/components/collection">
    <argument name="data" xsi:type="array">
        <item name="config" xsi:type="array">
            <item name="dataProvider" xsi:type="string">vendor_module_myentity</item>
            <item name="update_url" xsi:type="url" path="mui/index/render"/>
            <item name="filter_url_params" xsi:type="array">
                <!-- optional: bind filter params from URL -->
            </item>
        </item>
    </argument>
</dataSource>
```

**Critical:** The `dataProvider` string (`vendor_module_myentity`) must match the key registered in `di.xml`. The `name` attribute (`vendor_module_myentity_data_source`) is what child components reference as their `dataProvider`.

### 3.3 `<columns>`

The `<columns>` container holds column definitions and controls general listing appearance:

```xml
<columns name="vendor_module_myentity_columns">
    <argument name="data" xsi:type="array">
        <item name="config" xsi:type="array">
            <item name="displayMode" xsi:type="string">grid</item>
            <item name="storageConfig" xsi:type="array">
                <item name="provider" xsi:type="string">vendor_module_myentity.vendor_module_myentity_columns</item>
                <item name="ns" xsi:type="string">vendor_module_myentity_listing</item>
            </item>
        </item>
    </argument>
</columns>
```

---

## 4. Column Components

### 4.1 `text` Column

The simplest column type — renders plain text:

```xml
<column name="entity_name" sortOrder="10">
    <argument name="data" xsi:type="array">
        <item name="config" xsi:type="array">
            <item name="filter" xsi:type="string">text</item>
            <item name="sortOrder" xsi:type="number">10</item>
            <item name="sortable" xsi:type="boolean">true</item>
            <item name="visible" xsi:type="boolean">true</item>
            <item name="label" xsi:type="string" translate="true">Entity Name</item>
        </item>
    </argument>
</column>
```

### 4.2 `actionsColumn` — Row Action Buttons

The `actionsColumn` renders per-row action buttons (Edit, Delete) rendered in the last column:

```xml
<actionsColumn name="actions" class="Vendor\Module\Ui\Column\MyEntityActions" sortOrder="200">
    <argument name="data" xsi:type="array">
        <item name="config" xsi:type="array">
            <item name="urlPath" xsi:type="string">module/entity/edit</item>
            <item name="viewPath" xsi:type="string">module/entity/view</item>
            <item name="redirectName" xsi:type="string">module/entity/edit</item>
            <item name="indexField" xsi:type="string">entity_id</item>
        </item>
    </argument>
</actionsColumn>
```

#### The Action Class

```php
<?php
namespace Vendor\Module\Ui\Column;

use Magento\Ui\Component\Listing\Column\AbstractColumn;
use Magento\Framework\UrlInterface;

class MyEntityActions extends AbstractColumn
{
    protected $urlBuilder;

    public function __construct(
        \Magento\Framework\View\Element\UiComponent\ContextInterface $context,
        \Magento\Framework\View\Element\UiComponentFactory $uiComponentFactory,
        array $components = [],
        array $data = []
    ) {
        parent::__construct($context, $uiComponentFactory, $components, $data);
        $this->urlBuilder = $context->getUrlBuilder();
    }

    public function prepareDataSource(array $dataSource)
    {
        if (isset($dataSource['data']['items'])) {
            foreach ($dataSource['data']['items'] as &$item) {
                $item[$this->getData('name')] = [
                    'edit' => [
                        'href' => $this->urlBuilder->getUrl(
                            'module/entity/edit',
                            ['entity_id' => $item['entity_id']]
                        ),
                        'label' => __('Edit'),
                        '__disableTmpl' => true,
                    ],
                    'delete' => [
                        'href' => $this->urlBuilder->getUrl(
                            'module/entity/delete',
                            ['entity_id' => $item['entity_id']]
                        ),
                        'label' => __('Delete'),
                        '__disableTmpl' => true,
                        'confirm' => [
                            'title' => __('Delete %1', $item['entity_name']),
                            'message' => __('Are you sure you want to delete this entity?'),
                        ],
                    ],
                ];
            }
        }
        return $dataSource;
    }
}
```

Key points:
- Extend `Magento\Ui\Component\Listing\Column\AbstractColumn`
- Override `prepareDataSource()` to inject action URLs into each row
- The `indexField` config tells the column which field holds the entity ID
- `__disableTmpl` prevents the label from being HTML-escaped

### 4.3 `checkbox` Column — For Mass Actions

```xml
<selectionColumn name="ids" sortOrder="1">
    <argument name="data" xsi:type="array">
        <item name="config" xsi:type="array">
            <item name="resizeEnabled" xsi:type="boolean">false</item>
            <item name="resizeDefaultWidth" xsi:type="number">25</item>
            <item name="indexField" xsi:type="string">entity_id</item>
        </item>
    </argument>
</selectionColumn>
```

The `selectionColumn` component renders the checkbox. Combined with `massaction`, it allows selecting multiple rows.

### 4.4 `date` Column

```xml
<column name="created_at" sortOrder="50">
    <argument name="data" xsi:type="array">
        <item name="config" xsi:type="array">
            <item name="filter" xsi:type="string">dateRange</item>
            <item name="sortable" xsi:type="boolean">true</item>
            <item name="visible" xsi:type="boolean">true</item>
            <item name="label" xsi:type="string" translate="true">Created At</item>
            <item name="dateFormat" xsi:type="string">MMM d, yyyy h:mm:ss a</item>
            <item name="timezone" xsi:type="boolean">false</item>
        </item>
    </argument>
</column>
```

### 4.5 `select` and `multiselect` Columns

```xml
<column name="status" sortOrder="30">
    <argument name="data" xsi:type="array">
        <item name="config" xsi:type="array">
            <item name="filter" xsi:type="string">select</item>
            <item name="options" xsi:type="array">
                <item name="option" xsi:type="array">
                    <item name="value" xsi:type="string">enabled</item>
                    <item name="label" xsi:type="string" translate="true">Enabled</item>
                </item>
                <item name="option" xsi:type="array">
                    <item name="value" xsi:type="string">disabled</item>
                    <item name="label" xsi:type="string" translate="true">Disabled</item>
                </item>
            </item>
            <item name="dataType" xsi:type="string">select</item>
            <item name="label" xsi:type="string" translate="true">Status</item>
        </item>
    </argument>
</column>
```

---

## 5. Mass Actions

### 5.1 Declaring MassAction

```xml
<massaction name="listing_massaction">
    <argument name="data" xsi:type="array">
        <item name="config" xsi:type="array">
            <item name="actions" xsi:type="array">
                <item name="delete" xsi:type="array">
                    <item name="type" xsi:type="string">delete</item>
                    <item name="label" xsi:type="string" translate="true">Delete</item>
                    <item name="url" xsi:type="url" path="module/entity/massDelete"/>
                    <item name="confirm" xsi:type="array">
                        <item name="title" xsi:type="string" translate="true">Delete Entities</item>
                        <item name="message" xsi:type="string" translate="true">
                            Are you sure you want to delete the selected entities?
                        </item>
                    </item>
                </item>
            </item>
        </item>
    </argument>
</massaction>
```

### 5.2 Registering the MassAction on the Listing

MassAction must be registered on the `<listing>` root via the `massaction` argument:

```xml
<argument name="data" xsi:type="array">
    <item name="js_config" xsi:type="array">
        <item name="provider" xsi:type="string">vendor_module_myentity.vendor_module_myentity_data_source</item>
        <item name="deps" xsi:type="array">
            <item name="vendor_module_myentity.vendor_module_myentity_data_source"
                  xsi:type="string">vendor_module_myentity.vendor_module_myentity_data_source</item>
        </item>
    </item>
    <item name="config" xsi:type="array">
        <item name="massaction" xsi:type="array">
            <item name="field" xsi:type="string">ids</item>
        </item>
    </item>
</argument>
```

The `field` (`ids`) must match the `name` of the `selectionColumn` checkbox component. When a mass action fires, it collects all checked `ids` values and sends them as the `ids` parameter to the mass action URL.

### 5.3 Custom Mass Action Controller

```php
<?php
namespace Vendor\Module\Controller\Adminhtml\Entity;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Framework\Controller\ResultFactory;
use Magento\Ui\Model\ResourceModel\Bookmark\CollectionFactory as BookmarkCollectionFactory;

class MassDelete extends Action
{
    protected $collectionFactory;

    public function __construct(
        Context $context,
        BookmarkCollectionFactory $collectionFactory
    ) {
        parent::__construct($context);
        $this->collectionFactory = $collectionFactory;
    }

    public function execute()
    {
        $ids = $this->getRequest()->getParam('ids', []);
        if (!is_array($ids)) {
            $ids = explode(',', $ids);
        }

        $deleted = 0;
        foreach ($ids as $id) {
            try {
                // Load and delete entity
                $model = $this->_objectManager->create(\Vendor\Module\Model\MyEntity::class)->load($id);
                if ($model->getId()) {
                    $model->delete();
                    $deleted++;
                }
            } catch (\Exception $e) {
                $this->messageManager->addErrorMessage(__('Error deleting entity %1: %2', $id, $e->getMessage()));
            }
        }

        if ($deleted) {
            $this->messageManager->addSuccessMessage(__('Deleted %1 entity(entities).', $deleted));
        }

        /** @var \Magento\Framework\Controller\Result\Redirect $resultRedirect */
        $resultRedirect = $this->resultFactory->create(ResultFactory::TYPE_REDIRECT);
        return $resultRedirect->setPath('*/*/index');
    }
}
```

---

## 6. Filters, Sorting, and Pagination

### 6.1 `<filters>` — Toolbar Filters

```xml
<filters name="listing_filters">
    <argument name="data" xsi:type="array">
        <item name="config" xsi:type="array">
            <item name="columnsSet" xsi:type="array">
                <item name="fulltext" xsi:type="array">
                    <item name="type" xsi:type="string">fulltext</item>
                    <item name="placeholder" xsi:type="string" translate="true">Fulltext Search...</item>
                </item>
                <item name="entity_id" xsi:type="array">
                    <item name="type" xsi:type="string">text</item>
                    <item name="placeholder" xsi:type="string" translate="true">ID</item>
                </item>
                <item name="entity_name" xsi:type="array">
                    <item name="type" xsi:type="string">text</item>
                    <item name="placeholder" xsi:type="string" translate="true">Entity Name</item>
                </item>
                <item name="status" xsi:type="array">
                    <item name="type" xsi:type="string">select</item>
                    <item name="options" xsi:type="array">
                        <!-- options passed here -->
                    </item>
                </item>
                <item name="created_at" xsi:type="array">
                    <item name="type" xsi:type="string">dateRange</item>
                    <item name="placeholder" xsi:type="string" translate="true">Created At</item>
                </item>
            </item>
        </item>
    </argument>
</filters>
```

### 6.2 `<paging>` — Pagination

```xml
<paging name="listing_paging">
    <argument name="data" xsi:type="array">
        <item name="config" xsi:type="array">
            <item name="storageConfig" xsi:type="array">
                <item name="provider" xsi:type="string">vendor_module_myentity.vendor_module_myentity_columns</item>
                <item name="ns" xsi:type="string">vendor_module_myentity_listing</item>
            </item>
            <item name="config" xsi:type="array">
                <item name="pageSize" xsi:type="number">20</item>
                <item name="current" xsi:type="number">1</item>
            </item>
        </item>
    </argument>
</paging>
```

The default pageSize is configurable. Users can change the page size via the UI.

### 6.3 Bookmarking

Bookmarks allow users to save their column configurations, filter states, and display settings:

```xml
<bookmark name="listing_bookmark">
    <argument name="data" xsi:type="array">
        <item name="config" xsi:type="array">
            <item name="storageConfig" xsi:type="array">
                <item name="provider" xsi:type="string">vendor_module_myentity.vendor_module_myentity_columns</item>
                <item name="ns" xsi:type="string">vendor_module_myentity_listing</item>
            </item>
        </item>
    </argument>
</bookmark>
```

Bookmarks are stored in the `ui_bookmark` database table.

---

## 7. Full Example — Complete Entity Listing

Below is a complete, working `vendor_module_entity_listing.xml` for a custom entity (`MyEntity`) with all standard features: columns, actionsColumn, massAction with delete, filters, and paging.

### File: `app/code/Vendor/Module/view/adminhtml/ui_component/vendor_module_myentity_listing.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<listing xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Ui:etc/ui_configuration.xsd">

    <!-- Grid-level configuration: JS deps, massaction field binding -->
    <argument name="data" xsi:type="array">
        <item name="js_config" xsi:type="array">
            <item name="provider" xsi:type="string">
                vendor_module_myentity.vendor_module_myentity_data_source
            </item>
            <item name="deps" xsi:type="array">
                <item name="vendor_module_myentity.vendor_module_myentity_data_source"
                      xsi:type="string">
                    vendor_module_myentity.vendor_module_myentity_data_source
                </item>
            </item>
        </item>
        <item name="config" xsi:type="array">
            <item name="massaction" xsi:type="array">
                <item name="field" xsi:type="string">ids</item>
            </item>
        </item>
    </argument>

    <!-- DataSource: ties to DataProvider registered in di.xml -->
    <dataSource name="vendor_module_myentity_data_source"
                component="Magento_Ui/js/form/components/collection">
        <argument name="data" xsi:type="array">
            <item name="config" xsi:type="array">
                <item name="dataProvider" xsi:type="string">vendor_module_myentity</item>
                <item name="update_url" xsi:type="url" path="mui/index/render"/>
                <item name="filter_url_params" xsi:type="array">
                    <item name="store" xsi:type="string">*</item>
                </item>
            </item>
        </argument>
    </dataSource>

    <!-- Column set container -->
    <columns name="vendor_module_myentity_columns">

        <!-- Checkbox column for mass action selection -->
        <selectionColumn name="ids" sortOrder="1">
            <argument name="data" xsi:type="array">
                <item name="config" xsi:type="array">
                    <item name="resizeEnabled" xsi:type="boolean">false</item>
                    <item name="resizeDefaultWidth" xsi:type="number">25</item>
                    <item name="indexField" xsi:type="string">entity_id</item>
                </item>
            </argument>
        </selectionColumn>

        <!-- Entity ID column -->
        <column name="entity_id" sortOrder="10">
            <argument name="data" xsi:type="array">
                <item name="config" xsi:type="array">
                    <item name="filter" xsi:type="string">text</item>
                    <item name="sortable" xsi:type="boolean">true</item>
                    <item name="visible" xsi:type="boolean">true</item>
                    <item name="label" xsi:type="string" translate="true">ID</item>
                </item>
            </argument>
        </column>

        <!-- Entity Name column (text) -->
        <column name="entity_name" sortOrder="20">
            <argument name="data" xsi:type="array">
                <item name="config" xsi:type="array">
                    <item name="filter" xsi:type="string">text</item>
                    <item name="sortable" xsi:type="boolean">true</item>
                    <item name="visible" xsi:type="boolean">true</item>
                    <item name="label" xsi:type="string" translate="true">Entity Name</item>
                </item>
            </argument>
        </column>

        <!-- Status column (select) -->
        <column name="status" sortOrder="30">
            <argument name="data" xsi:type="array">
                <item name="config" xsi:type="array">
                    <item name="filter" xsi:type="string">select</item>
                    <item name="options" xsi:type="array">
                        <item name="option" xsi:type="array">
                            <item name="value" xsi:type="string">enabled</item>
                            <item name="label" xsi:type="string" translate="true">Enabled</item>
                        </item>
                        <item name="option" xsi:type="array">
                            <item name="value" xsi:type="string">disabled</item>
                            <item name="label" xsi:type="string" translate="true">Disabled</item>
                        </item>
                    </item>
                    <item name="dataType" xsi:type="string">select</item>
                    <item name="label" xsi:type="string" translate="true">Status</item>
                </item>
            </argument>
        </column>

        <!-- Created At column (date) -->
        <column name="created_at" sortOrder="40">
            <argument name="data" xsi:type="array">
                <item name="config" xsi:type="array">
                    <item name="filter" xsi:type="string">dateRange</item>
                    <item name="sortable" xsi:type="boolean">true</item>
                    <item name="visible" xsi:type="boolean">true</item>
                    <item name="label" xsi:type="string" translate="true">Created At</item>
                    <item name="dateFormat" xsi:type="string">MMM d, yyyy h:mm:ss a</item>
                    <item name="timezone" xsi:type="boolean">false</item>
                </item>
            </argument>
        </column>

        <!-- Updated At column (date) -->
        <column name="updated_at" sortOrder="50">
            <argument name="data" xsi:type="array">
                <item name="config" xsi:type="array">
                    <item name="filter" xsi:type="string">dateRange</item>
                    <item name="sortable" xsi:type="boolean">true</item>
                    <item name="visible" xsi:type="boolean">false</item>
                    <item name="label" xsi:type="string" translate="true">Updated At</item>
                    <item name="dateFormat" xsi:type="string">MMM d, yyyy h:mm:ss a</item>
                    <item name="timezone" xsi:type="boolean">false</item>
                </item>
            </argument>
        </column>

        <!-- Actions Column (Edit, Delete) -->
        <actionsColumn name="actions" class="Vendor\Module\Ui\Column\MyEntityActions" sortOrder="200">
            <argument name="data" xsi:type="array">
                <item name="config" xsi:type="array">
                    <item name="urlPath" xsi:type="string">module/entity/edit</item>
                    <item name="viewPath" xsi:type="string">module/entity/view</item>
                    <item name="redirectName" xsi:type="string">module/entity/edit</item>
                    <item name="indexField" xsi:type="string">entity_id</item>
                </item>
            </argument>
        </actionsColumn>

    </columns>

    <!-- Mass Actions (Delete selected) -->
    <massaction name="listing_massaction">
        <argument name="data" xsi:type="array">
            <item name="config" xsi:type="array">
                <item name="actions" xsi:type="array">
                    <item name="delete" xsi:type="array">
                        <item name="type" xsi:type="string">delete</item>
                        <item name="label" xsi:type="string" translate="true">Delete</item>
                        <item name="url" xsi:type="url" path="module/entity/massDelete"/>
                        <item name="confirm" xsi:type="array">
                            <item name="title" xsi:type="string" translate="true">Delete Entities</item>
                            <item name="message" xsi:type="string" translate="true">
                                Are you sure you want to delete the selected entities?
                            </item>
                        </item>
                    </item>
                </item>
            </item>
        </argument>
    </massaction>

    <!-- Filters toolbar -->
    <filters name="vendor_module_myentity_filters">
        <argument name="data" xsi:type="array">
            <item name="config" xsi:type="array">
                <item name="columnsSet" xsi:type="array">
                    <item name="fulltext" xsi:type="array">
                        <item name="type" xsi:type="string">fulltext</item>
                        <item name="placeholder" xsi:type="string" translate="true">
                            Fulltext Search...
                        </item>
                    </item>
                    <item name="entity_id" xsi:type="array">
                        <item name="type" xsi:type="string">text</item>
                        <item name="placeholder" xsi:type="string">ID</item>
                    </item>
                    <item name="entity_name" xsi:type="array">
                        <item name="type" xsi:type="string">text</item>
                        <item name="placeholder" xsi:type="string" translate="true">
                            Entity Name
                        </item>
                    </item>
                    <item name="status" xsi:type="array">
                        <item name="type" xsi:type="string">select</item>
                        <item name="options" xsi:type="array">
                            <item name="option" xsi:type="array">
                                <item name="value" xsi:type="string">enabled</item>
                                <item name="label" xsi:type="string" translate="true">Enabled</item>
                            </item>
                            <item name="option" xsi:type="array">
                                <item name="value" xsi:type="string">disabled</item>
                                <item name="label" xsi:type="string" translate="true">Disabled</item>
                            </item>
                        </item>
                    </item>
                    <item name="created_at" xsi:type="array">
                        <item name="type" xsi:type="string">dateRange</item>
                        <item name="placeholder" xsi:type="string" translate="true">Created At</item>
                    </item>
                </item>
            </item>
        </argument>
    </filters>

    <!-- Pagination -->
    <paging name="vendor_module_myentity_paging">
        <argument name="data" xsi:type="array">
            <item name="config" xsi:type="array">
                <item name="storageConfig" xsi:type="array">
                    <item name="provider" xsi:type="string">
                        vendor_module_myentity.vendor_module_myentity_columns
                    </item>
                    <item name="ns" xsi:type="string">vendor_module_myentity_listing</item>
                </item>
                <item name="config" xsi:type="array">
                    <item name="pageSize" xsi:type="number">20</item>
                </item>
            </item>
        </argument>
    </paging>

</listing>
```

### Supporting DataProvider (di.xml registration)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <type name="Magento\Ui\Model\Manager">
        <arguments>
            <argument name="providers" xsi:type="array">
                <argument name="vendor_module_myentity" xsi:type="object">
                    Vendor\Module\Ui\DataProvider\MyEntityDataProvider
                </argument>
            </argument>
        </arguments>
    </type>

</config>
```

### Supporting Action Class

```php
<?php
namespace Vendor\Module\Ui\Column;

use Magento\Ui\Component\Listing\Column\AbstractColumn;
use Magento\Framework\UrlInterface;

class MyEntityActions extends AbstractColumn
{
    protected $urlBuilder;

    public function __construct(
        \Magento\Framework\View\Element\UiComponent\ContextInterface $context,
        \Magento\Framework\View\Element\UiComponentFactory $uiComponentFactory,
        array $components = [],
        array $data = []
    ) {
        parent::__construct($context, $uiComponentFactory, $components, $data);
        $this->urlBuilder = $context->getUrlBuilder();
    }

    public function prepareDataSource(array $dataSource)
    {
        if (isset($dataSource['data']['items'])) {
            $name = $this->getData('name');
            foreach ($dataSource['data']['items'] as &$item) {
                $item[$name] = [
                    'edit' => [
                        'href' => $this->urlBuilder->getUrl(
                            'module/entity/edit',
                            ['entity_id' => $item['entity_id']]
                        ),
                        'label' => __('Edit'),
                        '__disableTmpl' => true,
                    ],
                    'delete' => [
                        'href' => $this->urlBuilder->getUrl(
                            'module/entity/delete',
                            ['entity_id' => $item['entity_id']]
                        ),
                        'label' => __('Delete'),
                        '__disableTmpl' => true,
                        'confirm' => [
                            'title' => __('Delete %1', $item['entity_name'] ?? $item['entity_id']),
                            'message' => __('Are you sure you want to delete this entity?'),
                        ],
                    ],
                ];
            }
        }
        return $dataSource;
    }
}
```

---

## 8. Common Issues

### "No Data" — Grid Renders But Has No Rows

**Causes and fixes:**

1. **DataProvider name mismatch**: The `dataProvider` string in `<dataSource>` must exactly match the key registered in `di.xml`. If you register as `vendor_module_myentity` but put `vendor_module_entity` in the XML, the grid has no data.

2. **Collection returning empty**: Add debug output to the DataProvider's `getData()` method. Check that the collection's SQL query is correct.

3. **Wrong primary field name**: The DataProvider's `$primaryFieldName` must match the actual primary key column name in the database table.

4. **Store filter active**: The `filter_url_params` in `<dataSource>` or the `store` parameter in URL may be filtering results to a specific store view that has no data.

5. **ACL blocking**: If the user lacks the ACL resource referenced in the listing's `resource` attribute, the grid appears with no data (no error shown).

### Column Not Appearing in Grid

1. **Check `visible`**: Set to `true` explicitly. Default is `true`, but JS override can hide it.
2. **Check `sortOrder`**: Columns render in `sortOrder` ascending. If two columns have the same sortOrder, ordering may be unpredictable.
3. **Check `label`**: A column with an empty label still renders; a missing label defaults to the column name.
4. **Check `addField` in DataProvider**: If using a custom DataProvider, ensure the `addFieldsToSelect` or `getSelect` includes the column.
5. **Check the JS component**: Some columns require specific JS components to render. The `filter` attribute determines which filter JS component is used; if the wrong type is specified, the column may be excluded from rendering.

### Mass Action Not Working

1. **`fieldName` mismatch**: The `massaction.field` in listing config must match the `name` of the `<selectionColumn>`. Default is `ids`, but if your checkbox column is named `my_ids`, update the massaction field.

2. **No `selectionColumn`**: If there's no `<selectionColumn>` in `<columns>`, the "Select All" checkbox in the toolbar header won't appear and mass actions will have nothing to operate on.

3. **Controller not receiving IDs**: In the mass action controller, `$this->getRequest()->getParam('ids')` returns the comma-separated or array of selected IDs. Verify the controller correctly reads this param.

4. **CSRF mismatch**: If the mass action URL uses `path="module/entity/massDelete"`, ensure the corresponding controller route is registered in `routes.xml` and the controller extends `MassActionInterface` or has proper CSRF token handling via `_csrfUpdate` or `_excludeCsrf` in newer Magento versions.

5. **URL param mismatch**: Mass actions send IDs as a URL parameter named after the `field` (`ids` by default). If your controller expects `selected` or `entity_ids`, check the massaction config.

### SortOrder — Column Ordering Off

When multiple columns share the same `sortOrder` value, they sort alphabetically by name as a tiebreaker. Always assign unique sortOrder values. Also ensure sortOrder is set inside `<item name="config">` not at the top level of `<argument name="data">`.

### Schema Validation Fails

If XML editing produces a schema validation error, the most common causes:

- **Wrong schema URL**: Must be `urn:magento:module:Magento_Ui:etc/ui_configuration.xsd`
- **Missing `xmlns:xsi`**: The `xsi:noNamespaceSchemaLocation` attribute requires the `xmlns:xsi` namespace declaration
- **Wrong `xsi:type`**: Values like `string`, `array`, `number`, `boolean`, `url` must be lowercase and valid XSI types
- **Item nesting**: Each `<item>` inside an array must have both `name` and `xsi:type`

### Performance — Slow Grid Load

- Enable SQL query logging and check for N+1 queries in the collection's `_beforeLoad()`
- Ensure proper indexes exist on the columns used in filters (`filter` attribute in column config)
- The `update_url` in dataSource pointing to `mui/index/render` is correct — avoid custom renderers
- For very large datasets, consider implementing `SearchResult` with a cursor-based approach instead of offset pagination

---

## Quick Reference

| File | Purpose |
|------|---------|
| `view/adminhtml/ui_component/entity_listing.xml` | Main grid XML definition |
| `Ui/DataProvider/CustomDataProvider.php` | Custom DataProvider class |
| `Ui/Column/MyEntityActions.php` | ActionsColumn renderer |
| `etc/di.xml` | DataProvider registration |
| `Controller/Adminhtml/Entity/MassDelete.php` | Mass action controller |
| `etc/adminhtml/routes.xml` | Admin route registration |

| XML Component | Schema type for `filter` |
|---------------|--------------------------|
| `text` column | `text` |
| `date` column | `dateRange` |
| `select` column | `select` |
| `multiselect` column | `multiselect` |
| `checkbox` column | N/A (no filtering) |

All column components extend `Magento\Ui\Component\Listing\Column\AbstractColumn`. The `dataSource` always references the DataProvider registered in `di.xml`, and the component tree flows data downward — dataSource provides items, columns consume and render them, and the listing wrapper coordinates filters, sorting, and pagination through the same data exchange mechanism.
