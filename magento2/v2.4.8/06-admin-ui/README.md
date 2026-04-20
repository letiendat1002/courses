# Topic 6: Admin UI — Routes, Menus, Configuration, ACL & Grids

**Goal:** Build a complete admin interface for your module — navigation entry, configuration page, permission system, and data grid with edit form.

---

## Topics Covered

- Admin routes (`etc/adminhtml/routes.xml`) — backend URL routing
- Admin menu (`etc/adminhtml/menu.xml`) — sidebar navigation
- System configuration pages (`etc/adminhtml/system.xml`) — settings in Stores → Configuration
- ACL permissions (`etc/acl.xml`) — role-based access control
- UI Component listing grids (`view/adminhtml/ui_component/*.xml`) — data tables in admin
- UI Component form (`view/adminhtml/ui_component/*.xml`) — edit/create records
- Data providers for grids and forms
- File upload in admin — FileUploader, validation, storage
- Edit and Save controllers for admin data management

---

## Reference Exercises

- **Exercise 5.1:** Create an admin route and controller with `_isAllowed()` ACL check
- **Exercise 5.2:** Build admin menu with hierarchy and parent references
- **Exercise 5.3:** Create a system configuration section with 3+ fields
- **Exercise 5.4:** Implement ACL with different roles and test access restriction
- **Exercise 5.5 (optional):** Build an admin grid with DataProvider and action columns
- **Exercise 5.6 (optional):** Build an admin edit form with save functionality
- **Exercise 5.7:** Add a file upload controller, store relative path in database

---

## Completion Criteria

- [ ] Admin route accessible at `/admin/review/index/index`
- [ ] Menu item visible in admin sidebar (parent: `Magento_Backend::content`)
- [ ] Menu item hidden when user lacks ACL permission
- [ ] Configuration page at Stores → Configuration with ≥3 fields
- [ ] Admin grid renders with data from repository (not hardcoded)
- [ ] Edit and Delete action buttons work from grid
- [ ] Admin edit form loads with data, Save persists via repository
- [ ] All controllers protected with `_isAllowed()` ACL checks

---

## Topics

---

### Topic 1: Admin Routes

**Admin URLs look different from frontend URLs:**

```
http://localhost/admin/review/index/index/key/xxx
         └─ frontName ┌─ controllerPath └─ action
```

**Why Admin Routes Are Different:**

Admin routes use a separate routing system from frontend:
- Separate `admin` router with higher priority in admin area
- `before="Magento_Backend"` ensures your route is matched before default routes
- All admin URLs include a security key (`key=xxx`) to prevent CSRF attacks
- Sessions are validated against the admin user

> **Security Note:** Never create admin routes without proper ACL checks. An exposed admin route without `_isAllowed()` is a critical security vulnerability.

** **`etc/adminhtml/routes.xml`:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:App/etc/routes.xsd">
    <router id="admin">
        <route id="review" frontName="review">
            <module name="Training_Review" before="Magento_Backend"/>
        </route>
    </router>
</config>
```

Key differences from frontend routes:
- `router id="admin"` (not `standard`)
- `before="Magento_Backend"` — ensures your route takes priority in admin

**Admin Controller:**

```php
<?php
// Controller/Adminhtml/Index/Index.php
namespace Training\Review\Controller\Adminhtml\Index;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Framework\View\Result\PageFactory;

class Index extends Action
{
    protected $resultPageFactory;

    public function __construct(Context $context, PageFactory $resultPageFactory)
    {
        parent::__construct($context);
        $this->resultPageFactory = $resultPageFactory;
    }

    public function execute()
    {
        $resultPage = $this->resultPageFactory->create();
        $resultPage->setActiveMenu('Training_Review::review');
        $resultPage->getConfig()->getTitle()->prepend(__('Reviews'));
        return $resultPage;
    }

    protected function _isAllowed(): bool
    {
        return $this->_authorization->isAllowed('Training_Review::review');
    }
}
```

**Key rule:** Every admin controller must implement `_isAllowed()` returning `true` only if the current user has the required permission.

**PSR-12 / Security Notes for Admin Controllers:**

```php
// ✅ CORRECT — Proper ACL check, form key validation
class Index extends Action
{
    public function execute(): Page
    {
        if (!$this->_isAllowed()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('You do not have permission')
            );
        }
        // ...
    }
}

// ❌ WRONG — Missing ACL check entirely
class Index extends Action
{
    public function execute(): Page
    {
        return $this->resultPageFactory->create(); // Anyone can access!
    }
}
```

**CSRF Protection in Admin Forms:**

Magento automatically validates the form key (`form_key`) on all POST requests. If you're creating AJAX endpoints that modify data, you must:

1. Include form key in your JavaScript requests
2. Validate it server-side

```php
// In your controller — Magento automatically validates form_key for POST
public function execute()
{
    // Magento checks _isAllowed() automatically via parent::execute()
    // Also validates form_key for POST requests
}
```

**Security Best Practices for Admin Controllers:**

| Practice | Why It Matters |
|----------|----------------|
| Always implement `_isAllowed()` | Prevents unauthorized access |
| Return 403 Page for denied access | Proper HTTP response, not silent failure |
| Validate all input with type hints | Prevent injection attacks |
| Use service classes for data operations | Don't put business logic in controllers |
| Log access denials for audit | Track potential security issues |

> **Pro Tip from Production:** Always call `parent::__construct($context)` before your constructor code — the parent constructor sets up authorization that `_isAllowed()` depends on.

---

### Topic 2: Admin Menu

**`etc/adminhtml/menu.xml`:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Backend/etc/menu.xsd">
    <menu>
        <add id="Training_Review::reviews"
             title="Reviews"
             module="Training_Review"
             sortOrder="30"
             parent="Magento_Backend::content"
             action="review/index/index"
             resource="Training_Review::review"/>
    </menu>
</config>
```

| Attribute | Purpose |
|-----------|---------|
| `id` | Unique identifier (`Vendor_Module::resource_id`) |
| `title` | Display text in sidebar |
| `parent` | Parent menu item — use `Magento_Backend::content` for top-level |
| `action` | Controller path (`review/index/index`) |
| `resource` | ACL resource — user must have this permission to see the item |

**Submenu Item:**

```xml
<add id="Training_Review::pending"
     title="Pending Reviews"
     module="Training_Review"
     sortOrder="10"
     parent="Training_Review::reviews"
     action="review/index/pending"
     resource="Training_Review::review_pending"/>
```

**Rebuilding the Admin Menu Cache:**

```bash
bin/magento admin:menu:rebuild
```

**Menu ACL — Security Deep Dive:**

The menu system is tightly integrated with ACL:

1. When a user lacks the `resource` permission, the menu item is **hidden** (not just disabled)
2. Even if someone guesses the URL, `_isAllowed()` in the controller returns `false`
3. Menu cache is per-user role — cached correctly for permission changes

> **⚠️ Common Security Mistake:** The menu `resource` MUST match an ACL resource in `etc/acl.xml`. If you skip ACL and rely only on controller checks, the menu still shows — confusing users who can't access the page.

**Menu Item Visibility Flow:**

```
User logs in → Role determines ACL permissions →
Menu rendered → Only show items where user has resource →
User clicks menu → Controller calls _isAllowed() →
Access granted or 403 returned
```

---

### Topic 3: System Configuration

**Configuration hierarchy:**

```
Stores → Configuration → [Section] → [Group] → [Field]
```

**`etc/adminhtml/system.xml`:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Config:etc/system_file.xsd">
    <section id="review" translate="label" type="text" sortOrder="100"
             showInDefault="1" showInWebsite="1" showInStore="1">
        <label>Reviews Settings</label>
        <tab>catalog</tab>
        <resource>Training_Review::config</resource>

        <group id="general" translate="label" type="text" sortOrder="10"
               showInDefault="1" showInWebsite="1" showInStore="1">
            <label>General Settings</label>

            <field id="enabled" translate="label" type="select" sortOrder="10"
                   showInDefault="1" showInWebsite="1" showInStore="1">
                <label>Enable Reviews</label>
                <source_model>Magento\Config\Model\Config\Source\Yesno</source_model>
            </field>

            <field id="allow_guest" translate="label" type="select" sortOrder="20"
                   showInDefault="1" showInWebsite="1" showInStore="1">
                <label>Allow Guest Reviews</label>
                <source_model>Magento\Config\Model\Config\Source\Yesno</source_model>
            </field>

            <field id="api_key" translate="label" type="text" sortOrder="30"
                   showInDefault="1" showInWebsite="1" showInStore="1">
                <label>API Key</label>
                <comment>Used for external integration</comment>
            </field>
        </group>
    </section>
</config>
```

**Reading Config Values:**

```php
<?php
public function isEnabled(): bool
{
    return (bool) $this->scopeConfig->getValue(
        'review/general/enabled',
        \Magento\Store\Model\ScopeInterface::SCOPE_STORE
    );
}
```

**Available source models:**
- `Magento\Config\Model\Config\Source\Yesno` — Yes/No select
- `Magento\Config\Model\Config\Source\Enabledisable` — Enabled/Disabled
- `Magento\Config\Model\Config\Source\Locale\Currency` — Currency list
- Custom source model: implement `\Magento\Framework\Data\OptionSourceInterface`

**System Configuration Best Practices:**

| Practice | Why It Matters |
|----------|----------------|
| Always set `showInDefault`, `showInWebsite`, `showInStore` correctly | Controls scope of configuration |
| Use appropriate `sortOrder` | Groups related settings together |
| Set `translate="label"` | Enables inline translation |
| Use validation rules | Prevent invalid configuration |
| Store sensitive data encrypted | API keys, passwords shouldn't be plain text |

> **Pro Tip from Production:** Configuration values are cached. After changing system configuration, flush the configuration cache: `bin/magento c:c config`

---

### Topic 4: ACL — Roles & Permissions

**ACL Hierarchy in `etc/acl.xml`:**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Acl/etc/acl.xsd">
    <acl>
        <resources>
            <resource id="Magento_Backend::admin">
                <resource id="Magento_Backend::content">
                    <resource id="Training_Review::review" title="Reviews" sortOrder="10">
                        <resource id="Training_Review::review_view"  title="View Reviews" sortOrder="10"/>
                        <resource id="Training_Review::review_edit"  title="Edit Reviews" sortOrder="20"/>
                        <resource id="Training_Review::review_delete" title="Delete Reviews" sortOrder="30"/>
                    </resource>
                </resource>
                <resource id="Magento_Backend::system">
                    <resource id="Training_Review::config" title="Reviews Configuration" sortOrder="40"/>
                </resource>
            </resource>
        </resources>
    </acl>
</config>
```

**Menu → ACL Connection:**

The `resource` attribute in `menu.xml` links the menu item to the ACL resource:

```xml
<add id="Training_Review::reviews"
     ...
     resource="Training_Review::review"/>
```

If user lacks `Training_Review::review`, the menu item is hidden.

**Checking ACL in Controllers:**

```php
protected function _isAllowed(): bool
{
    return $this->_authorization->isAllowed('Training_Review::review_edit');
}
```

**Assigning Roles:**

Admin → System → Permissions → User Roles → Create role → Assign resources.

**ACL Best Practices:**

| Practice | Why It Matters |
|----------|----------------|
| Define granular ACL resources | Principle of least privilege |
| Match menu `resource` to ACL exactly | Menu visibility depends on this |
| Test with different roles | Ensure restrictions work correctly |
| Don't hardcode ACL checks | Use `isAllowed()` consistently |

> **Security Note:** ACL resources follow a hierarchy. If you deny a parent resource, all children are also denied. Always define your ACL resources following Magento's resource tree structure.

**Real-World ACL Pattern from Magento Core:**

From `Magento_Catalog/etc/acl.xml`:

```xml
<resource id="Magento_Catalog::catalog" title="Catalog" sortOrder="10">
    <resource id="Magento_Catalog::products" title="Products" sortOrder="10">
        <resource id="Magento_Catalog::edit_product_desc" title="Edit Product Description"/>
    </resource>
</resource>
```

---

### Topic 5: Admin UI Component Grids

**Grid Architecture — How UI Components Work:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                        UI Component Layer                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  XML Configuration (view/adminhtml/ui_component/*.xml)       │   │
│  │  - Column definitions, filters, actions                      │   │
│  │  - Data source binding, ACL resource                        │   │
│  │  - Toolbar configuration (bookmarks, filters, export)         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ↓                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  JavaScript Components (Magento_Ui/js/grid/*)               │   │
│  │  - request() / receive() - data fetching                     │   │
│  │  - insert() / render() - DOM manipulation                    │   │
│  │  - Various plugins for filtering, sorting, pagination        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ↓                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  DataProvider (PHP)                                          │   │
│  │  - getData() returns items + totalRecords                    │   │
│  │  - Receives search criteria from JS                          │   │
│  │  - Calls repository for actual data                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ↓                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Repository / Database                                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

**Why This Architecture Matters:**

The UI Component system provides:
- **Decoupling:** XML defines structure, JS handles rendering, PHP handles data
- **Reusability:** Same components work across different grids
- **Extensibility:** Override specific components without touching others
- **Performance:** JS handles sorting/filtering without server round-trips

> **Pro Tip from Production:** The UI Component grid is complex but powerful. Don't try to customize it with jQuery or DOM manipulation — work with the component system or you'll break the architecture.

**Listing XML:**

```xml
<?xml version="1.0"?>
<listing xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Ui:etc/ui_configuration.xsd">
    <argument name="data" xsi:type="array">
        <item name="js_config" xsi:type="array">
            <item name="provider" xsi:type="string">
                training_review_review_listing.training_review_review_listing_data_source
            </item>
        </item>
    </argument>

    <settings>
        <spinner>training_review_columns</spinner>
        <deps>
            <dep>training_review_review_listing.training_review_review_listing_data_source</dep>
        </deps>
    </settings>

    <dataSource name="training_review_review_listing_data_source"
                component="Magento_Ui/js/grid/provider">
        <settings>
            <updateUrl path="mui/index/render"/>
        </settings>
        <aclResource>Training_Review::review_view</aclResource>
        <dataProvider class="Training\Review\Ui\DataProvider\ReviewDataProvider"
                      name="training_review_review_listing_data_source">
            <settings>
                <requestFieldName>id</requestFieldName>
                <primaryFieldName>review_id</primaryFieldName>
            </settings>
        </dataProvider>
    </dataSource>

    <listingToolbar name="listing_top">
        <settings><sticky>true</sticky></settings>
        <bookmark name="bookmarks"/>
        <columnsControls name="columns_controls"/>
        <filterSearch name="fulltext"/>
        <filters name="listing_filters"/>
        <paging name="listing_paging"/>
        <exportButton name="export_button"/>
    </listingToolbar>

    <columns name="training_review_columns">
        <column name="review_id">
            <settings>
                <filter>textRange</filter>
                <sorting>asc</sorting>
                <label translate="true">ID</label>
            </settings>
        </column>
        <column name="reviewer_name">
            <settings>
                <filter>text</filter>
                <label translate="true">Reviewer</label>
            </settings>
        </column>
        <column name="rating">
            <settings>
                <filter>text</filter>
                <label translate="true">Rating</label>
            </settings>
        </column>
        <column name="created_at" class="Magento\Ui\Component\Listing\Columns\Date">
            <settings>
                <filter>dateRange</filter>
                <label translate="true">Created</label>
            </settings>
        </column>
        <actionsColumn name="actions"
                       class="Training\Review\Ui\Component\Listing\Column\ReviewActions">
            <settings>
                <indexField>review_id</indexField>
                <label translate="true">Actions</label>
            </settings>
        </actionsColumn>
    </columns>
</listing>
```

**Data Provider:**

```php
<?php
// Ui/DataProvider/ReviewDataProvider.php
namespace Training\Review\Ui\DataProvider;

use Magento\Framework\View\Element\UiComponent\DataProvider\DataProvider;
use Training\Review\Model\ResourceModel\Review\CollectionFactory;

class ReviewDataProvider extends DataProvider
{
    private $collectionFactory;

    public function __construct(
        $name,
        $primaryFieldName,
        $requestFieldName,
        CollectionFactory $collectionFactory,
        array $meta = [],
        array $data = []
    ) {
        parent::__construct($name, $primaryFieldName, $requestFieldName, $meta, $data);
        $this->collectionFactory = $collectionFactory;
    }

    public function getData(): array
    {
        $collection = $this->collectionFactory->create();
        return [
            'items' => $collection->getItems(),
            'totalRecords' => $collection->getSize()
        ];
    }
}
```

**DataProvider Architecture Deep Dive:**

The DataProvider extends `Magento\Framework\View\Element\UiComponent\DataProvider\DataProvider` which provides:
- `getSearchResult()` — Returns search results based on search criteria
- `getData()` — Returns formatted data for the UI
- `getMeta()` — Returns metadata for the component

```php
// Full DataProvider with search, filtering, and pagination support
class ReviewDataProvider extends DataProvider
{
    public function getData(): array
    {
        // Get search criteria from the UI component request
        $searchCriteria = $this->getSearchCriteria();

        // Build collection with filters
        $collection = $this->collectionFactory->create();

        // Apply filters from search criteria
        foreach ($searchCriteria->getFilterGroups() as $filterGroup) {
            foreach ($filterGroup->getFilters() as $filter) {
                $collection->addFieldToFilter(
                    $filter->getField(),
                    [$filter->getConditionType() => $filter->getValue()]
                );
            }
        }

        // Apply sorting
        foreach ($searchCriteria->getSortOrders() as $sortOrder) {
            $collection->setOrder(
                $sortOrder->getField(),
                $sortOrder->getDirection()
            );
        }

        // Paginate
        $collection->setCurPage($searchCriteria->getCurrentPage());
        $collection->setPageSize($searchCriteria->getPageSize());

        return [
            'items' => $collection->getItems(),
            'totalRecords' => $collection->getSize(),
        ];
    }
}
```

**DataProvider Best Practices:**

| Practice | Why It Matters |
|----------|----------------|
| Use `CollectionFactory` (non-shared) | Each request gets fresh collection |
| Return `totalRecords` accurately | Pagination needs correct totals |
| Apply search criteria filters | UI filters must work |
| Don't hardcode data | Data should come from repository |
| Clear collection between calls | DataProvider may be called multiple times |

> **Pro Tip:** The DataProvider is also responsible for **search** functionality. The grid's search box sends requests to the same `getData()` method with filter criteria. Your collection must apply these filters.

**Real-World Pattern from Magento Core:**

From `Magento_Cms/Ui/DataProvider/PageDataProvider`:

```php
class PageDataProvider extends DataProvider
{
    public function getData(): array
    {
        $collection = $this->getCollection();
        $data = parent::getData(); // Gets items + total

        // Post-processing: add URL to each item
        foreach ($data['items'] as &$item) {
            $item['preview_url'] = $this->urlBuilder->getUrl(
                'cms/page/view',
                ['page_id' => $item['page_id']]
            );
        }

        return $data;
    }
}
```

**Action Column (Edit/Delete):**

```php
<?php
// Ui/Component/Listing/Column/ReviewActions.php
namespace Training\Review\Ui\Component\Listing\Column;

use Magento\Framework\UrlInterface;
use Magento\Framework\View\Element\UiComponent\ContextInterface;
use Magento\Framework\View\Element\UiComponentFactory;
use Magento\Ui\Component\Listing\Columns\Column;

class ReviewActions extends Column
{
    protected $urlBuilder;

    public function __construct(
        ContextInterface $context,
        UiComponentFactory $uiComponentFactory,
        UrlInterface $urlBuilder,
        array $components = [],
        array $data = []
    ) {
        parent::__construct($context, $uiComponentFactory, $components, $data);
        $this->urlBuilder = $urlBuilder;
    }

    public function prepareDataSource(array $dataSource): array
    {
        if (isset($dataSource['data']['items'])) {
            foreach ($dataSource['data']['items'] as &$item) {
                $item[$this->getData('name')] = [
                    'edit' => [
                        'href' => $this->urlBuilder->getUrl(
                            'review/index/edit',
                            ['review_id' => $item['review_id']]
                        ),
                        'label' => __('Edit')
                    ],
                    'delete' => [
                        'href' => $this->urlBuilder->getUrl(
                            'review/index/delete',
                            ['review_id' => $item['review_id']]
                        ),
                        'label' => __('Delete'),
                        'confirm' => [
                            'title' => __('Delete Review'),
                            'message' => __('Delete this review?')
                        ]
                    ]
                ];
            }
        }
        return $dataSource;
    }
}
```

**Layout File — `view/adminhtml/layout/review_index_index.xml`:**

```xml
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Backend/etc/pages.xsd">
    <update handle="styles"/>
    <body>
        <referenceContainer name="content">
            <uiComponent name="training_review_review_listing"/>
        </referenceContainer>
    </body>
</page>
```

---

### Topic 6: Admin Form UI Component

**Form Architecture:**

```
UI Component Form XML → DataProvider → Repository → Database
view/adminhtml/ui_component/training_review_review_form.xml
```

**Form Architecture Deep Dive:**

The admin form system uses the same UI Component architecture as grids:

```
┌─────────────────────────────────────────────────────────────────┐
│  Form UI Component                                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  XML: field definitions, fieldsets, validation rules     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              ↓                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  JS: Magento_Ui/js/form/form — handles submit, validate   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              ↓                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  PHP DataProvider: loads data, handles save              │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Why Separate DataProvider for Forms?**

Unlike grids (which fetch data for display), form DataProviders typically:
1. Load existing entity data for edit
2. Return empty array for new entity creation
3. Handle the distinction between save and create

**Form XML:**

```xml
<?xml version="1.0"?>
<form xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Ui:etc/ui_configuration.xsd">
    <argument name="data" xsi:type="array">
        <item name="js_config" xsi:type="array">
            <item name="provider" xsi:type="string">
                training_review_review_form.training_review_review_form_data_source
            </item>
        </item>
    </argument>

    <settings>
        <spinner>training_review_form_fields</spinner>
        <dataScope>data</dataScope>
        <namespace>training_review_review_form</namespace>
    </settings>

    <dataSource name="training_review_review_form_data_source">
        <settings>
            <submitUrl path="review/index/save"/>
        </settings>
        <aclResource>Training_Review::review_edit</aclResource>
        <dataProvider class="Training\Review\Ui\DataProvider\ReviewFormDataProvider"
                      name="training_review_review_form_data_source">
            <settings>
                <requestFieldName>review_id</requestFieldName>
                <primaryFieldName>review_id</primaryFieldName>
            </settings>
        </dataProvider>
    </dataSource>

    <fieldset name="review_details">
        <settings>
            <label translate="true">Review Information</label>
        </settings>

        <field name="review_id" formElement="input">
            <settings><disabled>true</disabled><label translate="true">ID</label></settings>
        </field>

        <field name="product_id" formElement="input">
            <settings>
                <label translate="true">Product ID</label>
                <validation><rule name="required" xsi:type="boolean">true</rule></validation>
            </settings>
        </field>

        <field name="reviewer_name" formElement="input">
            <settings>
                <label translate="true">Reviewer Name</label>
                <validation><rule name="required" xsi:type="boolean">true</rule></validation>
            </settings>
        </field>

        <field name="rating" formElement="select">
            <settings>
                <label translate="true">Rating</label>
                <options>
                    <option name="1" label="1 Star" value="1"/>
                    <option name="2" label="2 Stars" value="2"/>
                    <option name="3" label="3 Stars" value="3"/>
                    <option name="4" label="4 Stars" value="4"/>
                    <option name="5" label="5 Stars" value="5"/>
                </options>
            </settings>
        </field>

        <field name="review_text" formElement="textarea">
            <settings><label translate="true">Review Text</label><rows>4</rows></settings>
        </field>
    </fieldset>
</form>
```

**Form Validation — Built-in Rules:**

Magento provides validation rules you can configure in XML:

```xml
<field name="reviewer_name">
    <settings>
        <validation>
            <rule name="required" xsi:type="boolean">true</rule>
            <rule name="minLength" xsi:type="number">2</rule>
            <rule name="maxLength" xsi:type="number">255</rule>
        </validation>
    </settings>
</field>

<field name="rating">
    <settings>
        <validation>
            <rule name="required" xsi:type="boolean">true</rule>
            <rule name="validate-digits" xsi:type="boolean">true</rule>
            <rule name="validate-range" xsi:type="string">1-5</rule>
        </validation>
    </settings>
</field>
```

**Available Validation Rules:**

| Rule | XML Value | Purpose |
|------|-----------|---------|
| Required | `required="true"` | Field cannot be empty |
| Min/Max Length | `minLength`, `maxLength` | String length constraints |
| Min/Max Value | `min`, `max` | Numeric constraints |
| Email | `validate-email="true"` | Valid email format |
| Url | `validate-url="true"` | Valid URL format |
| Date | `validate-date="true"` | Valid date format |
| Pattern | `pattern="regex"` | Custom regex validation |
| Equals | `equalTo="#otherField"` | Match another field value |

**PSR-12 / Form Security Best Practices:**

| Practice | Why It Matters |
|----------|----------------|
| Always validate on server AND client | Client validation is UX only, not security |
| Sanitize input before saving | Prevent XSS, SQL injection |
| Use `escapeHtml()` in templates | Prevent XSS in displayed data |
| Set `disabled="true"` on auto-generated fields | Prevent tampering with IDs, timestamps |

> **Security Note:** Client-side validation (in the browser) is for user experience only. Always validate server-side in your controller before saving. Never trust user input.

**Form Data Provider:**

```php
<?php
// Ui/DataProvider/ReviewFormDataProvider.php
namespace Training\Review\Ui\DataProvider;

use Magento\Framework\View\Element\UiComponent\DataProvider\DataProvider;
use Training\Review\Api\ReviewRepositoryInterface;

class ReviewFormDataProvider extends DataProvider
{
    protected $reviewRepository;

    public function __construct(
        $name, $primaryFieldName, $requestFieldName,
        ReviewRepositoryInterface $reviewRepository,
        array $meta = [], array $data = []
    ) {
        parent::__construct($name, $primaryFieldName, $requestFieldName, $meta, $data);
        $this->reviewRepository = $reviewRepository;
    }

    public function getData(): array
    {
        $reviewId = (int) $this->getRequest()->getParam($this->getRequestFieldName());
        if (!$reviewId) return [];

        try {
            $review = $this->reviewRepository->getById($reviewId);
            return ['training_review_review_form' => ['data' => $review->getData()]];
        } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
            return [];
        }
    }
}
```

**Edit Controller:**

```php
<?php
// Controller/Adminhtml/Index/Edit.php
namespace Training\Review\Controller\Adminhtml\Index;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Framework\View\Result\PageFactory;
use Training\Review\Api\ReviewRepositoryInterface;

class Edit extends Action
{
    protected $resultPageFactory;
    protected $reviewRepository;

    public function __construct(Context $context, PageFactory $resultPageFactory, ReviewRepositoryInterface $reviewRepository)
    {
        parent::__construct($context);
        $this->resultPageFactory = $resultPageFactory;
        $this->reviewRepository = $reviewRepository;
    }

    public function execute()
    {
        $reviewId = $this->getRequest()->getParam('review_id');
        $resultPage = $this->resultPageFactory->create();
        $resultPage->setActiveMenu('Training_Review::review');
        $resultPage->getConfig()->getTitle()->prepend(
            $reviewId ? __('Edit Review #%1', $reviewId) : __('New Review')
        );
        return $resultPage;
    }

    protected function _isAllowed(): bool
    {
        return $this->_authorization->isAllowed('Training_Review::review_edit');
    }
}
```

**Save Controller:**

```php
<?php
// Controller/Adminhtml/Index/Save.php
namespace Training\Review\Controller\Adminhtml\Index;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Training\Review\Api\Data\ReviewInterfaceFactory;
use Training\Review\Api\ReviewRepositoryInterface;

class Save extends Action
{
    protected $reviewRepository;
    protected $reviewFactory;

    public function __construct(
        Context $context,
        ReviewRepositoryInterface $reviewRepository,
        ReviewInterfaceFactory $reviewFactory
    ) {
        parent::__construct($context);
        $this->reviewRepository = $reviewRepository;
        $this->reviewFactory = $reviewFactory;
    }

    public function execute()
    {
        $data = $this->getRequest()->getPostValue();
        if (!$data) return $this->_redirect('review/index/index');

        try {
            $reviewId = isset($data['review_id']) ? (int)$data['review_id'] : null;
            $review = $reviewId
                ? $this->reviewRepository->getById($reviewId)
                : $this->reviewFactory->create();

            $review->setProductId((int)$data['product_id']);
            $review->setReviewerName($data['reviewer_name']);
            $review->setRating((int)$data['rating']);
            $review->setReviewText($data['review_text'] ?? '');

            $this->reviewRepository->save($review);
            $this->messageManager->addSuccessMessage(__('Review saved'));
        } catch (\Exception $e) {
            $this->messageManager->addErrorMessage($e->getMessage());
        }

        return $this->_redirect('review/index/index');
    }

    protected function _isAllowed(): bool
    {
        return $this->_authorization->isAllowed('Training_Review::review_edit');
    }
}
```

**Save Controller Security Checklist:**

| Check | Implementation |
|-------|----------------|
| CSRF Protection | Magento validates form_key automatically for POST |
| ACL Check | `_isAllowed()` verifies user has `review_edit` |
| Input Validation | Validate type, range, format before save |
| Mass Assignment | Only set fields that should be modifiable |
| Error Handling | Catch exceptions, show user-friendly message |

**⚠️ Mass Assignment Vulnerability Prevention:**

```php
// ❌ WRONG — Setting any field from POST (mass assignment)
public function execute()
{
    $data = $this->getRequest()->getPostValue();
    $review->setData($data); // Sets ALL POST fields — dangerous!
    $this->reviewRepository->save($review);
}

// ✅ CORRECT — Only set specific allowed fields
public function execute()
{
    $data = $this->getRequest()->getPostValue();
    $allowedFields = ['product_id', 'reviewer_name', 'rating', 'review_text'];
    foreach ($allowedFields as $field) {
        if (isset($data[$field])) {
            $review->setData($field, $data[$field]);
        }
    }
    $this->reviewRepository->save($review);
}
```

> **Pro Tip from Production:** Always validate the data types. `getPostValue()` returns mixed types. Cast to expected types: `(int)$data['rating']`, `(string)$data['reviewer_name']`.

---

---

### Topic 7: File & Image Upload in Admin Forms

**Overview:** Admin modules frequently need to accept file uploads — CSV import files, PDF attachments, images. Magento provides a structured `FileUploader` system that integrates with its media storage and handles validation, renaming, and security.

**The `FileUploaderFactory` Pattern:**

Unlike `move_uploaded_file()`, Magento's `FileUploader` handles:
- File validation (type, size, extensions)
- Moving to media storage with unique filenames
- Generating collision-free names
- Returning the relative path for database storage

**Controller — Handling File Upload:**

```php
<?php
// Controller/Adminhtml/Import/Upload.php
namespace Training\Review\Controller\Adminhtml\Import;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Framework\App\Filesystem\DirectoryList;
use Magento\Framework\Filesystem;
use Magento\Framework\Message\ManagerInterface;
use Magento\MediaStorage\Model\File\UploaderFactory;

class Upload extends Action
{
    protected $uploaderFactory;
    protected $filesystem;
    protected $messageManager;

    public function __construct(
        Context $context,
        UploaderFactory $uploaderFactory,
        Filesystem $filesystem,
        ManagerInterface $messageManager
    ) {
        parent::__construct($context);
        $this->uploaderFactory = $uploaderFactory;
        $this->filesystem = $filesystem;
        $this->messageManager = $messageManager;
    }

    public function execute()
    {
        $fileData = $this->getRequest()->getFiles('import_file');
        if (!$fileData || !$fileData['tmp_name']) {
            $this->messageManager->addErrorMessage(__('No file uploaded'));
            return $this->_redirect('*/*/index');
        }

        try {
            $uploader = $this->uploaderFactory->create(['fileId' => 'import_file']);
            $uploader->setAllowedExtensions(['csv', 'xml']);
            $uploader->setAllowRenameFiles(true);
            $uploader->setFilesDispersion(false);
            $uploader->setAllowCreateFolders(true);

            $mediaDir = $this->filesystem->getDirectoryWrite(DirectoryList::MEDIA);
            $targetPath = $mediaDir->getAbsolutePath('import/training_review');

            $result = $uploader->save($targetPath);
            $fileName = $result['file'];

            $this->messageManager->addSuccessMessage(__('File uploaded: %1', $fileName));
            $this->_forward('import', null, null, ['file' => $fileName]);
        } catch (\Exception $e) {
            $this->messageManager->addErrorMessage($e->getMessage());
            return $this->_redirect('*/*/index');
        }
    }

    protected function _isAllowed(): bool
    {
        return $this->_authorization->isAllowed('Training_Review::review_import');
    }
}
```

**Restrictive Uploader — Size and Extension Validation:**

```php
<?php
$uploader = $this->uploaderFactory->create(['fileId' => 'attachment']);
$uploader->setAllowedExtensions(['jpg', 'jpeg', 'png', 'gif', 'pdf']);
$uploader->setMaxSize('file', 5242880); // 5MB max
$uploader->setAllowCreateFolders(true);
```

**Storing File Path in Database:**

Always store the **relative path** — never the absolute path:

```php
<?php
// GOOD — portable across environments
$review->setAttachmentFile('import/training_review/' . $fileName);

// BAD — breaks when domain or mount point changes
$review->setAttachmentFile('/var/www/html/pub/media/import/training_review/' . $fileName);
```

**Retrieving File URL in Admin:**

```php
<?php
$filePath = $review->getAttachmentFile();
$mediaUrl = $this->storeManager->getStore()
    ->getBaseUrl(\Magento\Framework\Url::URL_TYPE_MEDIA);
$fileUrl = $mediaUrl . $filePath;
```

**Security Checklist:**

| Risk | Mitigation |
|------|------------|
| PHP shell upload | Block `php`, `phtml`, `exe`, `sh` extensions |
| Large file DoS | `setMaxSize()` — 5MB max is reasonable |
| Path traversal | Magento's `FileUploader` sanitizes paths |
| Overwriting files | `setAllowRenameFiles(true)` generates unique names |
| File type spoofing | Use `application_file_delete` for actual MIME check |

**File Upload Security Deep Dive:**

```php
// ✅ CORRECT — Comprehensive file validation
$uploader = $this->uploaderFactory->create(['fileId' => 'attachment']);

// 1. Whitelist allowed extensions only
$uploader->setAllowedExtensions(['jpg', 'jpeg', 'png', 'gif', 'pdf']);

// 2. Set maximum file size (bytes)
$uploader->setMaxSize('file', 5242880); // 5MB

// 3. Allow file renaming to prevent overwrites
$uploader->setAllowRenameFiles(true);

// 4. Create folders if needed
$uploader->setAllowCreateFolders(true);

// 5. No dispersion (flat structure)
$uploader->setFilesDispersion(false);

// 6. Validate file before saving
// (Magento internally checks MIME type against extension)
```

**Dangerous File Extensions — Block These:**

```
php, phtml, phar, phpt, php3, php4, php5, php7, phps
exe, sh, bash, zsh, cmd, bat, com,msi
svg (can contain JS), html, htm, xml (XXE risk)
```

> **⚠️ Never rely on client-side file extension validation.** Always use server-side whitelist.

**Storing File Paths — Security Note:**

```php
// ✅ CORRECT — Store relative path only
$review->setAttachmentFile('import/training_review/' . $fileName);

// ❌ WRONG — Store absolute path
$review->setAttachmentFile('/var/www/html/pub/media/import/' . $fileName);

// ❌ WRONG — Store full URL
$review->setAttachmentFile('http://example.com/media/import/' . $fileName);
```

---

## Reading List
## Reading List

- [Admin Routing](https://developer.adobe.com/commerce/php/development/components/routing/#admin-routes) — Admin route structure
- [UI Components](https://developer.adobe.com/commerce/frontend-core/ui-components/) — Listing and form components
- [System Configuration](https://developer.adobe.com/commerce/php/development/configuration/) — system.xml structure

---

## Edge Cases & Troubleshooting

| Issue | Symptom | Solution |
|-------|---------|----------|
| Route 404 | `/admin/review/...` returns 404 | Check `routes.xml` area=`admin`, `before="Magento_Backend"` |
| Menu not showing | Menu item absent from sidebar | ACL permission denied — verify `resource` in menu.xml |
| Config page blank | Stores → Config shows empty | Missing `system.xml` or wrong `showInDefault` flags |
| Grid empty | UI Component loads but no data | DataProvider not returning items in correct format |
| Grid loading forever | Spinner keeps spinning | Wrong `requestFieldName` in dataProvider |
| ACL blocking access | 403 on all actions | `_isAllowed()` returning false — verify ACL resource |
| Form save fails silently | No error, but data not saved | Check repository exceptions, logging |
| File upload fails | Upload completes but file missing | Check directory permissions, disk space |
| Export button missing | Export button not in toolbar | Add `<exportButton name="export_button"/>` to listing XML |

**Debugging Admin UI Components:**

```bash
# Clear admin cache
bin/magento cache:clean config

# Clear view preprocessed files
rm -rf var/view_preprocessed/*

# Clear generated code
rm -rf var/generation/*

# Rebuild DI
bin/magento setup:di:compile

# In developer mode, clear pub/static
rm -rf pub/static/adminhtml/*
```

---

## Common Mistakes to Avoid

1. ❌ **Forgetting `_isAllowed()` in admin controllers** → Security hole; anyone can access
   - **Prevention:** Always implement and test `_isAllowed()`

2. ❌ **Menu `resource` not matching `acl.xml`** → Menu visible to everyone or no one
   - **Prevention:** Use identical resource IDs in both files

3. ❌ **Wrong `requestFieldName` in UI Component** → Grid doesn't load, spinner forever
   - **Prevention:** Match `requestFieldName` to URL parameter name (e.g., `review_id`)

4. ❌ **Forgetting `bin/magento c:f` after ACL changes** → Stale cache; confusing behavior
   - **Prevention:** Always flush cache after configuration changes

5. ❌ **`routes.xml` using `frontend` area for admin routes** → Route never matches
   - **Prevention:** Use `<router id="admin">` not `<router id="standard">`

6. ❌ **Mass assignment in save controller** → Users can modify protected fields
   - **Prevention:** Only set explicitly allowed fields

7. ❌ **Missing form key validation** → CSRF vulnerability
   - **Prevention:** Magento validates automatically for POST; don't bypass

8. ❌ **Not escaping output in templates** → XSS vulnerability
   - **Prevention:** Use `$escaper->escapeHtml()` for all user content

9. ❌ **Using FilePath instead of FileArray in FileUploader** → Inconsistent behavior
   - **Prevention:** Use `['fileId' => 'field_name']` format

10. ❌ **Hardcoding admin URL paths** → Breaks in multi-store/custom admin paths
    - **Prevention:** Always use `$this->urlBuilder->getUrl()` for URLs

---

## Pro Tips from Production

1. **Admin routes always need `before="Magento_Backend"`** — Without it, your route may not be found in the admin area

2. **Test ACL with multiple roles** — Create a role without your module's permissions and verify everything is hidden

3. **DataProviders are instantiated per request** — They're not shared, so you can safely use constructor injection

4. **Grid columns are sortable by default** — Set `sorting="false"` on columns that shouldn't be sortable

5. **Use `addFilterToMap` in DataProvider** — Maps UI field names to actual column names in your collection

6. **Export runs server-side** — No AJAX, it's a regular HTTP request that returns a file

7. **Form validation runs client AND server** — But server validation is the security boundary

8. **When in doubt with UI Components, clear everything** — `rm -rf var/* pub/static/*` and redeploy

---

## Cross-References

- **Topic 05 (Customization):** Plugins can intercept admin controller methods
- **Topic 07 (API):** Admin APIs use the same ACL system for authorization
- **Topic 08 (Data Ops):** Import/export operations often triggered from admin grids

---

*Magento 2 Backend Developer Course — Topic 06*
