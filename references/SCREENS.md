# Moqui Screen & Form Development Reference

> **Schema**: Screen XML must conform to `framework/xsd/xml-screen-3.xsd`. Form definitions within screens must conform to `framework/xsd/xml-form-3.xsd`. XML Actions blocks must conform to `framework/xsd/xml-actions-3.xsd`.

## Table of Contents
1. [Screen Structure](#screen-structure)
2. [Screen Hierarchy & Routing](#screen-hierarchy--routing)
3. [App Registration & Home Screen](#app-registration-on-the-home-screen)
4. [Transitions](#transitions)
5. [Form Single](#form-single)
6. [Form List](#form-list)
7. [Layout Widgets](#layout-widgets)
8. [Input Widgets](#input-widgets)
9. [Display Widgets](#display-widgets)
10. [Sections & Dynamic Content](#sections--dynamic-content)
11. [SimpleScreens Patterns](#simplescreens-patterns)
12. [Theming & Templates](#theming--templates)

---

## Screen Structure

### Complete Screen Template

```xml
<?xml version="1.0" encoding="UTF-8"?>
<screen xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/xml-screen-3.xsd"
    default-menu-title="My Screen"
    default-menu-index="5"
    default-menu-include="true"
    require-authentication="true"
    begin-transaction="false">

    <!-- Subscreens configuration -->
    <subscreens default-item="FindMyEntity" always-use-full-path="false"/>

    <!-- Pre-actions: run before ALL screens in path (including parent screens) -->
    <pre-actions>
        <set field="html_title" value="My Application"/>
        <set field="activeOrgId" from="ec.user.getPreference('ACTIVE_ORGANIZATION')"/>
    </pre-actions>

    <!-- Transitions (URL endpoints that process input) -->
    <transition name="createEntity">
        <service-call name="mycomp.MyServices.create#MyEntity"/>
        <default-response url="."/>  <!-- Reload current screen -->
    </transition>
    <transition name="editEntity">
        <default-response url="../EditMyEntity"/>  <!-- Navigate to sibling -->
    </transition>
    <transition name="downloadReport">
        <actions><!-- prepare data --></actions>
        <default-response url="." url-type="plain"/>  <!-- Return raw content -->
    </transition>
    <transition name="getPartyList">
        <service-call name="mantle.party.PartyServices.search#Party"
            in-map="[anyField: term, pageSize: 20]" out-map="context"/>
        <default-response type="none"/>  <!-- JSON response for AJAX -->
    </transition>

    <!-- Actions: data preparation for rendering -->
    <actions>
        <entity-find entity-name="mycomp.myapp.MyEntity" list="entityList">
            <search-form-inputs default-order-by="-lastUpdatedStamp"/>
        </entity-find>
    </actions>

    <!-- Widgets: visual elements -->
    <widgets>
        <!-- Screen content here -->
    </widgets>
</screen>
```

### Screen Attributes

| Attribute | Description |
|-----------|-------------|
| `default-menu-title` | Menu label text |
| `default-menu-index` | Menu sort order (lower = first) |
| `default-menu-include` | Show in parent menu (true/false) |
| `require-authentication` | `true`, `false`, `anonymous-view`, `anonymous-all` |
| `begin-transaction` | Begin DB transaction for screen render |

---

## Screen Hierarchy & Routing

### Directory-Based Subscreens

```
screen/
├── MyApp.xml                        → /myapp
│   ├── MyApp/                       (subscreens directory)
│   │   ├── FindOrder.xml            → /myapp/FindOrder
│   │   ├── EditOrder.xml            → /myapp/EditOrder
│   │   ├── Settings.xml             → /myapp/Settings
│   │   └── Settings/                (nested subscreens)
│   │       ├── General.xml          → /myapp/Settings/General
│   │       └── Payment.xml          → /myapp/Settings/Payment
```

### XML-Based Subscreens

```xml
<screen>
    <subscreens default-item="dashboard">
        <subscreens-item name="dashboard" location="component://MyComp/screen/Dashboard.xml"
            menu-title="Dashboard" menu-index="1"/>
        <subscreens-item name="orders" location="component://MyComp/screen/Orders.xml"
            menu-title="Orders" menu-index="2"/>
    </subscreens>
</screen>
```

### URL Navigation in Transitions

```xml
<!-- Current screen -->
<default-response url="."/>

<!-- Sibling screen -->
<default-response url="../EditOrder"/>

<!-- Child screen -->
<default-response url="./Details"/>

<!-- Absolute path from root -->
<default-response url="//myapp/EditOrder"/>

<!-- With parameters -->
<default-response url="../EditOrder">
    <parameter name="orderId"/>
</default-response>

<!-- External URL -->
<default-response url="https://example.com" url-type="plain"/>

<!-- Conditional response -->
<conditional-response url="../ViewOrder">
    <condition><expression>orderId</expression></condition>
</conditional-response>
<default-response url="."/>
```

### App Registration on the Home Screen

The Moqui home screen (`/qapps`) displays app tiles for each app registered under `apps.xml`. For an app to be visible, **three conditions** must be met (checked in `AppList.xml`):

1. **Screen is found** — The screen XML file exists and is properly mounted
2. **No required parameters** — The root app screen must NOT have `<parameter required="true"/>`
3. **`isPermitted()` returns true** — Artifact authorization seed data must grant access

#### Mounting a Standalone App

Mount your app under `apps.xml` (NOT `webroot.xml`) in your `MoquiConf.xml`:

```xml
<moqui-conf xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/moqui-conf-3.xsd">
    <screen-facade>
        <!-- Standalone app on home screen (alongside MarbleERP, System, Tools) -->
        <screen location="component://webroot/screen/webroot/apps.xml">
            <subscreens-item name="myapp" menu-title="My Application" menu-index="20"
                location="component://MyComponent/screen/MyApp.xml"/>
        </screen>
    </screen-facade>
</moqui-conf>
```

#### Root App Screen Requirements

A standalone app's root screen needs specific attributes to work with the Quasar SPA:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<screen xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/xml-screen-3.xsd"
    default-menu-title="My Application"
    menu-image="fa fa-cogs" menu-image-type="icon"
    server-static="vuet,qvt">

    <always-actions>
        <service-call name="mantle.party.PartyServices.setup#UserOrganizationInfo" out-map="context"/>
    </always-actions>

    <subscreens default-item="Dashboard" always-use-full-path="true"/>

    <widgets>
        <subscreens-panel id="MyAppPanel" type="popup" title="My Application"/>
    </widgets>
</screen>
```

Key attributes explained:
| Attribute | Purpose |
|-----------|---------|
| `server-static="vuet,qvt"` | Required for Quasar/Vue UI — pre-renders static screen structure |
| `subscreens-panel type="popup"` | Creates the left-drawer navigation menu (standard for standalone apps) |
| `always-use-full-path="true"` | Ensures URL paths always include the subscreen name |
| `menu-image` | Icon class shown on the home screen app tile (FontAwesome) |
| `menu-image-type="icon"` | Specifies the image is an icon class, not a URL |

> **Do NOT add `<parameter>` elements to root app screens** — the home screen skips apps with required parameters.

#### Artifact Authorization (Required!)

Without artifact authorization, the app will NOT appear on the home screen. Create a seed data file:

```xml
<!-- data/MyComponentSetupData.xml -->
<entity-facade-xml type="seed">
    <artifactGroups artifactGroupId="MY_APP" description="My App (via root screen)">
        <artifacts artifactName="component://MyComponent/screen/MyApp.xml"
            artifactTypeEnumId="AT_XML_SCREEN" inheritAuthz="Y"/>
        <authz artifactAuthzId="MY_APP_ADMIN" userGroupId="ADMIN"
            authzTypeEnumId="AUTHZT_ALWAYS" authzActionEnumId="AUTHZA_ALL"/>
    </artifactGroups>
</entity-facade-xml>
```

After creating this, reload seed data: `java -jar moqui.war load types=seed`

#### Mounting Inside MarbleERP vs Standalone

| Goal | Mount Under | URL Pattern |
|------|------------|-------------|
| Standalone app on home screen | `component://webroot/screen/webroot/apps.xml` | `/qapps/myapp` |
| New tab inside MarbleERP | `component://MarbleERP/screen/marble.xml` | `/qapps/marble/MyModule` |
| Screen under existing MarbleERP module | `component://MarbleERP/screen/marble/Customer.xml` | `/qapps/marble/Customer/MyScreen` |

#### How the Quasar SPA Navigation Works

The Quasar UI (`/qapps/`) is a single-page application:
- `basePath` = `/apps` (used for server data fetching — the SPA fetches screen data from `/apps/...`)
- `linkBasePath` = `/qapps` (used for displayed URLs in the browser address bar)
- The `menuData` transition on `webroot.xml` provides the navigation menu structure to the SPA
- `qapps.xml` has `allow-extra-path="true"` and renders screens from the shared `apps.xml` screen tree

---

## Transitions

### Transition with Service Call

```xml
<!-- Simple service call (auto-maps form parameters) -->
<transition name="updateOrder">
    <service-call name="mantle.order.OrderServices.update#OrderHeader"/>
    <default-response url="."/>
</transition>

<!-- Service call with explicit mapping -->
<transition name="placeOrder">
    <service-call name="mantle.order.OrderServices.place#Order"
        in-map="[orderId: orderId]" out-map="context"/>
    <conditional-response url="../ViewOrder">
        <condition><expression>!ec.message.hasError()</expression></condition>
    </conditional-response>
    <default-response url="."/>
</transition>
```

### Transition with Custom Actions

```xml
<transition name="processData">
    <actions>
        <entity-find-one entity-name="mycomp.MyEntity" value-field="myEntity"/>
        <if condition="myEntity.statusId != 'Active'">
            <message error="true">Entity must be Active</message>
            <return/>
        </if>
        <service-call name="mycomp.MyServices.process#Entity" in-map="context"/>
    </actions>
    <default-response url="."/>
</transition>
```

### AJAX/JSON Transition (for dropdowns, search, etc.)

```xml
<transition name="getProductList">
    <service-call name="mantle.product.ProductServices.search#Products"
        in-map="[term: term, pageSize: 20]" out-map="context"/>
    <default-response type="none"/>  <!-- Returns JSON -->
</transition>

<!-- Used in form as: -->
<drop-down allow-empty="true">
    <dynamic-options transition="getProductList" server-search="true" min-length="2">
        <option key="productId" text="${productName} [${pseudoId}]"/>
    </dynamic-options>
</drop-down>
```

---

## Form Single

### Complete Form-Single Example

```xml
<form-single name="EditOrder" transition="updateOrder" map="orderHeader">
    <!-- Hidden PK field -->
    <field name="orderId"><default-field><hidden/></default-field></field>

    <!-- Display field (read-only) -->
    <field name="placedDate"><default-field title="Placed">
        <display format="yyyy-MM-dd HH:mm"/></default-field></field>

    <!-- Text input -->
    <field name="orderName"><default-field title="Order Name">
        <text-line size="40"/></default-field></field>

    <!-- Dropdown from entity -->
    <field name="statusId"><default-field title="Status">
        <drop-down>
            <entity-options text="${description}" key="${statusId}">
                <entity-find entity-name="moqui.basic.StatusItem">
                    <econdition field-name="statusTypeId" value="OrderHeader"/>
                    <order-by field-name="sequenceNum"/>
                </entity-find>
            </entity-options>
        </drop-down></default-field></field>

    <!-- Dropdown with dynamic search -->
    <field name="customerPartyId"><default-field title="Customer">
        <drop-down allow-empty="true">
            <dynamic-options transition="getPartyList" server-search="true" min-length="2">
                <option key="partyId"
                    text="${organizationName ?: ''} ${firstName ?: ''} ${lastName ?: ''} [${pseudoId}]"/>
            </dynamic-options>
        </drop-down></default-field></field>

    <!-- Date picker -->
    <field name="estimatedDeliveryDate"><default-field title="Est. Delivery">
        <date-time type="date"/></default-field></field>

    <!-- Textarea -->
    <field name="shippingInstructions"><default-field title="Instructions">
        <text-area rows="4" cols="60"/></default-field></field>

    <!-- Checkbox -->
    <field name="isGift"><default-field title="Gift?">
        <check><option key="Y" text=" "/></check>
    </default-field></field>

    <!-- Submit button -->
    <field name="submitButton"><default-field title="Update">
        <submit/></default-field></field>
</form-single>
```

---

## Form List

### Complete Form-List Example

```xml
<form-list name="OrderList" list="orderList" transition="editOrder"
        header-dialog="true"       <!-- Filter fields in dialog -->
        select-columns="true"      <!-- User can toggle columns -->
        saved-finds="true"         <!-- Save filter combos -->
        show-csv-button="true"     <!-- Export to CSV -->
        show-xlsx-button="true"    <!-- Export to Excel -->
        show-pdf-button="true"     <!-- Export to PDF -->
        show-all-button="true"     <!-- Show all (no pagination) -->
        paginate="true">           <!-- Enable pagination -->

    <!-- Row-level data preparation -->
    <row-actions>
        <entity-find-one entity-name="mantle.party.PartyDetail" value-field="customerDetail">
            <field-map field-name="partyId" from="customerPartyId"/></entity-find-one>
        <service-call name="mantle.order.OrderInfoServices.get#OrderDisplayInfo"
            in-map="[orderId: orderId]" out-map="orderInfo"/>
    </row-actions>

    <!-- ID with link -->
    <field name="orderId">
        <header-field show-order-by="true"><text-find size="15"/></header-field>
        <default-field><link url="editOrder" text="${orderId}" link-type="anchor"/></default-field>
    </field>

    <!-- Status with dropdown filter -->
    <field name="statusId">
        <header-field title="Status" show-order-by="true">
            <drop-down allow-empty="true" allow-multiple="true">
                <entity-options text="${description}" key="${statusId}">
                    <entity-find entity-name="moqui.basic.StatusItem">
                        <econdition field-name="statusTypeId" value="OrderHeader"/>
                        <order-by field-name="sequenceNum"/>
                    </entity-find>
                </entity-options>
            </drop-down>
        </header-field>
        <default-field><display-entity entity-name="moqui.basic.StatusItem"/></default-field>
    </field>

    <!-- Date with range filter -->
    <field name="placedDate">
        <header-field show-order-by="true"><date-find type="date"/></header-field>
        <default-field><display format="yyyy-MM-dd"/></default-field>
    </field>

    <!-- Computed display -->
    <field name="customer"><default-field>
        <display text="${customerDetail?.organizationName ?: ''} ${customerDetail?.firstName ?: ''} ${customerDetail?.lastName ?: ''}"/>
    </default-field></field>

    <!-- Currency display -->
    <field name="grandTotal">
        <header-field show-order-by="true"/>
        <default-field><display format="#,##0.00" currency-unit-field="currencyUomId"/></default-field>
    </field>

    <!-- Submit button in header (for search) -->
    <field name="findButton"><header-field title="Find"><submit/></header-field></field>
</form-list>
```

### search-form-inputs

In screen actions, `search-form-inputs` auto-wires header-field inputs to entity queries:

```xml
<actions>
    <entity-find entity-name="mantle.order.OrderHeader" list="orderList">
        <search-form-inputs
            default-order-by="-placedDate"
            skip-fields="statusId"      <!-- Don't auto-wire this field -->
            paginate="true"
            input-fields-map="ec.web.parameters"/>
        <!-- Additional manual conditions -->
        <econdition field-name="statusId" operator="in" from="statusId_op == 'in' ? statusId?.split(',')?.toList() : null"
            ignore-if-empty="true"/>
    </entity-find>
</actions>
```

---

## Layout Widgets

```xml
<!-- Container (div) -->
<container style="my-css-class">
    <label text="Hello" type="h3"/>
</container>

<!-- Bootstrap grid row -->
<container-row>
    <row-col lg="4" md="6"><label text="Left"/></row-col>
    <row-col lg="4" md="6"><label text="Center"/></row-col>
    <row-col lg="4" md="12"><label text="Right"/></row-col>
</container-row>

<!-- Panel layout -->
<container-panel id="MainPanel">
    <panel-header><label text="Header" type="h4"/></panel-header>
    <panel-left size="3"><!-- Nav menu --></panel-left>
    <panel-center><!-- Main content --></panel-center>
    <panel-right size="3"><!-- Sidebar --></panel-right>
    <panel-footer><label text="Footer"/></panel-footer>
</container-panel>

<!-- Modal dialog -->
<container-dialog id="CreateDialog" button-text="Create New" width="700">
    <form-single name="CreateForm" transition="createEntity">
        <!-- form fields -->
    </form-single>
</container-dialog>

<!-- Dynamic dialog (loads content via AJAX from transition) -->
<dynamic-dialog id="DetailDialog" button-text="View Details"
    transition="getDetailContent" width="800">
    <parameter name="orderId"/>
</dynamic-dialog>

<!-- Tabs -->
<container type="tabs" id="MainTabs">
    <container type="tab" id="Tab1" title="Overview">
        <!-- content -->
    </container>
    <container type="tab" id="Tab2" title="Items">
        <!-- content -->
    </container>
</container>

<!-- Accordions -->
<container type="accordion" id="MainAccordion">
    <container type="accordion-item" title="Section 1">
        <!-- content -->
    </container>
</container>
```

---

## Input Widgets

```xml
<!-- Text line -->
<text-line size="30" maxlength="100" default-value=""/>

<!-- Text area -->
<text-area rows="5" cols="60" maxlength="4000"/>

<!-- Text find (search with operator) -->
<text-find size="25" default-operator="begins" hide-options="false"/>

<!-- Password -->
<password size="20"/>

<!-- Drop-down (select) -->
<drop-down allow-empty="true" allow-multiple="false" submit-on-select="false"
        current="selected" current-description="">
    <!-- Static options -->
    <option key="Y" text="Yes"/>
    <option key="N" text="No"/>

    <!-- From entity -->
    <entity-options text="${description} [${enumId}]" key="${enumId}">
        <entity-find entity-name="moqui.basic.Enumeration">
            <econdition field-name="enumTypeId" value="OrderType"/>
            <order-by field-name="sequenceNum,description"/>
        </entity-find>
    </entity-options>

    <!-- Dynamic search (AJAX) -->
    <dynamic-options transition="searchParties" server-search="true" min-length="2"
            depends-on="vendorPartyId" depends-optional="true">
        <option key="partyId" text="${name} [${pseudoId}]"/>
    </dynamic-options>
</drop-down>

<!-- Date/time -->
<date-time type="date"/>           <!-- Date only -->
<date-time type="date-time"/>      <!-- Date + time -->
<date-time type="time"/>           <!-- Time only -->

<!-- Date find (range for search) -->
<date-find type="date"/>           <!-- From/thru date range -->

<!-- Range find (numeric range) -->
<range-find size="10"/>

<!-- Checkboxes -->
<check all-checked="false">
    <option key="Y" text="Active"/>
    <entity-options text="${description}" key="${statusId}">
        <entity-find entity-name="moqui.basic.StatusItem">
            <econdition field-name="statusTypeId" value="MyType"/>
        </entity-find>
    </entity-options>
</check>

<!-- Radio buttons -->
<radio no-current-selected-key="N">
    <option key="Y" text="Yes"/>
    <option key="N" text="No"/>
</radio>

<!-- File upload -->
<file size="30" accept=".csv,.xlsx"/>

<!-- Hidden field -->
<hidden default-value="${orderId}"/>

<!-- Submit -->
<submit text="Save" type="primary"/>
<submit text="Cancel" confirmation="Are you sure?"/>

<!-- Link as button -->
<link url="cancelOrder" text="Cancel Order" btn-type="danger"
    confirmation="Cancel this order?" link-type="hidden-form"
    parameter-map="[orderId: orderId]"/>
```

---

## Display Widgets

```xml
<!-- Simple display -->
<display/>
<display text="${firstName} ${lastName}"/>
<display format="yyyy-MM-dd"/>
<display format="#,##0.00" currency-unit-field="currencyUomId"/>

<!-- Display from related entity (auto-lookup) -->
<display-entity entity-name="moqui.basic.StatusItem" text="${description}"/>
<display-entity entity-name="mantle.party.PartyDetail"
    text="${organizationName ?: ''} ${firstName ?: ''} ${lastName ?: ''}"/>
<display-entity entity-name="moqui.basic.Enumeration" key-field-name="enumId"
    text="${description}"/>

<!-- Hyperlink -->
<link url="editOrder" text="${orderId}" link-type="anchor"/>
<link url="editOrder" text="View Order" link-type="anchor"
    parameter-map="[orderId: orderId]"/>
<link url="https://example.com" url-type="plain" text="External" link-type="anchor"/>

<!-- Link as form submission (POST) -->
<link url="deleteItem" text="Delete" link-type="hidden-form"
    confirmation="Delete this item?" btn-type="danger"
    parameter-map="[orderId: orderId, orderItemSeqId: orderItemSeqId]"/>

<!-- Image -->
<image url="//${productImageUrl}" url-type="plain" alt="${productName}" width="100"/>

<!-- Label -->
<label text="Section Title" type="h3"/>
<label text="Note: ${importantInfo}" style="text-warning"/>

<!-- Render sub-template -->
<render-mode>
    <text type="html" location="component://MyComp/template/myTemplate.html.ftl"/>
</render-mode>

<!-- Include subscreen content -->
<subscreens-panel id="center-content" type="tab"/>
```

---

## Sections & Dynamic Content

```xml
<!-- Conditional section -->
<section name="AdminSection">
    <condition><expression>ec.user.isInGroup("ADMIN")</expression></condition>
    <actions><!-- data prep for admin view --></actions>
    <widgets><!-- admin widgets --></widgets>
    <fail-widgets><!-- non-admin widgets --></fail-widgets>
</section>

<!-- Section iterate -->
<section-iterate name="ItemSection" list="orderItems" entry="item">
    <actions>
        <entity-find-one entity-name="mantle.product.Product" value-field="product">
            <field-map field-name="productId" from="item.productId"/>
        </entity-find-one>
    </actions>
    <widgets>
        <container>
            <label text="${product.productName}: ${item.quantity} x ${ec.l10n.formatCurrency(item.unitAmount, currencyUomId)}"/>
        </container>
    </widgets>
</section-iterate>

<!-- Include screen from another location -->
<include-screen location="component://MyComp/screen/SharedWidgets.xml"/>

<!-- Dynamic container (AJAX-loaded content) -->
<dynamic-container id="detailArea" transition="getDetailContent">
    <parameter name="orderId"/>
</dynamic-container>
```

---

## SimpleScreens Patterns

### Typical Find/Edit Screen Pair

**FindMyEntity.xml** (list screen):
```xml
<screen default-menu-title="My Entities">
    <transition name="editMyEntity"><default-response url="../EditMyEntity"/></transition>
    <transition name="createMyEntity">
        <service-call name="create#mycomp.myapp.MyEntity"/>
        <default-response url="../EditMyEntity"/>
    </transition>

    <actions>
        <entity-find entity-name="mycomp.myapp.MyEntity" list="entityList">
            <search-form-inputs default-order-by="-lastUpdatedStamp"/>
        </entity-find>
    </actions>

    <widgets>
        <container-dialog id="NewEntityDialog" button-text="New Entity">
            <form-single name="NewEntity" transition="createMyEntity">
                <field name="description"><default-field><text-line size="60"/></default-field></field>
                <field name="submitButton"><default-field title="Create"><submit/></default-field></field>
            </form-single>
        </container-dialog>

        <form-list name="EntityList" list="entityList" transition="editMyEntity"
                header-dialog="true" select-columns="true" saved-finds="true">
            <field name="myEntityId"><header-field show-order-by="true"/>
                <default-field><link url="editMyEntity" text="${myEntityId}" link-type="anchor"/></default-field></field>
            <field name="description"><header-field show-order-by="true"><text-find/></header-field>
                <default-field><display/></default-field></field>
            <field name="statusId"><header-field title="Status">
                    <drop-down allow-empty="true">
                        <entity-options text="${description}" key="${statusId}">
                            <entity-find entity-name="moqui.basic.StatusItem">
                                <econdition field-name="statusTypeId" value="MyEntity"/>
                                <order-by field-name="sequenceNum"/>
                            </entity-find>
                        </entity-options>
                    </drop-down></header-field>
                <default-field><display-entity entity-name="moqui.basic.StatusItem"/></default-field></field>
            <field name="findButton"><header-field title="Find"><submit/></header-field></field>
        </form-list>
    </widgets>
</screen>
```

**EditMyEntity.xml** (detail screen):
```xml
<screen default-menu-title="Edit Entity" default-menu-include="false">
    <parameter name="myEntityId" required="true"/>

    <transition name="updateMyEntity">
        <service-call name="update#mycomp.myapp.MyEntity"/>
        <default-response url="."/>
    </transition>
    <transition-include name="searchPartyList"
        location="component://SimpleScreens/template/party/PartyForms.xml"/>

    <actions>
        <entity-find-one entity-name="mycomp.myapp.MyEntity" value-field="myEntity"/>
    </actions>

    <widgets>
        <container-row>
            <row-col lg="6">
                <form-single name="EditEntity" transition="updateMyEntity" map="myEntity">
                    <field name="myEntityId"><default-field><display/></default-field></field>
                    <field name="description"><default-field><text-line size="60"/></default-field></field>
                    <field name="statusId"><default-field title="Status">
                        <drop-down current="selected">
                            <entity-options text="${description}" key="${toStatusId}">
                                <entity-find entity-name="moqui.basic.StatusFlowTransition">
                                    <econdition field-name="statusFlowId" value="Default"/>
                                    <econdition field-name="statusId" from="myEntity.statusId"/>
                                </entity-find>
                            </entity-options>
                            <option key="${myEntity.statusId}" text="${myEntity.status?.description ?: myEntity.statusId}"/>
                        </drop-down></default-field></field>
                    <field name="submitButton"><default-field title="Update"><submit/></default-field></field>
                </form-single>
            </row-col>
            <row-col lg="6">
                <!-- Related info panel -->
            </row-col>
        </container-row>
    </widgets>
</screen>
```

---

## Theming & Templates

### FTL Macro Templates

Screen rendering uses FreeMarker templates. Default templates are in `moqui-runtime`:
```
runtime/base-component/webroot/screen/webroot/
├── js/                 # Client-side JavaScript
├── css/                # Stylesheets
└── assets/             # Static assets
```

### Custom CSS/JS

```xml
<!-- In screen actions or pre-actions -->
<set field="html_scripts" from="(html_scripts ?: '') + '&lt;script src=&quot;/myapp/js/custom.js&quot;&gt;&lt;/script&gt;'"/>
<set field="html_stylesheets" from="(html_stylesheets ?: '') + '&lt;link rel=&quot;stylesheet&quot; href=&quot;/myapp/css/custom.css&quot;&gt;'"/>
```

### Render Mode (multi-format output)

```xml
<render-mode>
    <text type="html" location="component://MyComp/template/report.html.ftl"/>
    <text type="csv" location="component://MyComp/template/report.csv.ftl"/>
    <text type="xlsx" location="component://MyComp/template/report.xlsx.ftl"/>
    <text type="pdf" location="component://MyComp/template/report.xsl-fo.ftl"/>
</render-mode>
```

## Verify

After adding or modifying a screen:
1. Navigate to it in `qapps/marble` (or wherever it's mounted). Confirm artifact authorization is set if this is a new app root — otherwise you'll get a 404 or a blank tile.
2. For a form: submit it; confirm the transition lands on the expected next screen with the expected message.
3. For a list: confirm filter / sort / paginate work; confirm the empty state and the error state.
4. Check the browser console for FTL or widget errors.

Common failures: missing transition target, missing parameter on a list filter, business logic in the screen (move it to a service).

See [`../SKILL.md`](../SKILL.md) §"Verify After Each Pattern" for the cross-cutting table.

<!-- begin: related-lessons (auto-generated by scripts/regen-skill-lesson-links.sh — do not edit by hand) -->

## Related Lessons

Cross-references from [`docs/ai/lessons.md`](../../docs/ai/lessons.md) that apply to this area. Filter tags: `Moqui-Services`. Regenerate with `scripts/regen-skill-lesson-links.sh`.

_No matching lessons captured yet for these tags._

<!-- end: related-lessons -->
