# MarbleERP Application Reference

## Table of Contents
1. [Overview](#overview)
2. [Architecture & Dependencies](#architecture--dependencies)
3. [Application Structure](#application-structure)
4. [Module Reference](#module-reference)
5. [Screen Hierarchy & Navigation](#screen-hierarchy--navigation)
6. [Extending MarbleERP](#extending-marbleerp)
7. [SimpleScreens Reuse Patterns](#simplescreens-reuse-patterns)
8. [Configuration & Setup](#configuration--setup)
9. [Quasar UI (qapps)](#quasar-ui-qapps)
10. [Common Customization Recipes](#common-customization-recipes)

---

## Overview

Moqui Marble ERP is a comprehensive planning and management application for businesses that produce goods **and** services, selling to both individuals (B2C) and organizations (B2B). It supersedes and combines functionality from:

- **POP Commerce ERP** (POPC) — Retail/wholesale goods-focused ERP
- **HiveMind Admin** — Services/project management administration
- **HiveMind PM** — Project management and time tracking

MarbleERP is organized around **3 primary role categories**:
1. **Internal** — Company employees, management, accounting, configuration
2. **Supplier** — Procurement, vendor management, purchase orders, receiving
3. **Customer** — Sales orders, quotes, fulfillment, customer service

These map to the two high-level business processes:
- **Procure to Pay** (Supplier interactions: PO → Receive → Vendor Invoice → Pay)
- **Order to Cash** (Customer interactions: Quote → SO → Ship → Invoice → Collect)

Plus cross-cutting: Configuration, Accounting, HR/Employee management, Equipment, Supplies, Light Manufacturing.

---

## Architecture & Dependencies

### Component Dependency Chain

```
MarbleERP
└── depends-on: SimpleScreens        ← Admin screen library
    └── depends-on: mantle-usl       ← Business logic services
        └── depends-on: mantle-udm   ← Data model entities
```

### Component File (component.xml)

```xml
<component name="MarbleERP" version="1.0.0">
    <depends-on name="SimpleScreens"/>
</component>
```

### Key Technical Facts
- **No custom entities** — MarbleERP uses Mantle UDM exclusively
- **No custom services** — MarbleERP uses Mantle USL exclusively
- **Screens-only component** — All value-add is in the screen layer
- **Heavy SimpleScreens reuse** — Most screens include/reference SimpleScreens components
- **Quasar UI** — Modern Vue.js based UI at `/qapps/marble`
- **Data files** — Seed data for MarbleERP-specific configuration (menus, settings)

---

## Application Structure

### Directory Layout

```
MarbleERP/
├── component.xml                    # Component metadata & dependencies
├── MoquiConf.xml                    # Screen mounting configuration
├── data/
│   └── MarbleERPSetupData.xml       # Seed data (menus, default settings)
└── screen/
    └── marble.xml                   # Root application screen
        └── marble/                  # Application modules (subscreens)
            ├── Customer/            # Customer-facing operations
            │   ├── Customer.xml
            │   └── Customer/
            │       ├── CustomerDashboard.xml
            │       ├── FindOrder.xml
            │       ├── EditOrder.xml
            │       ├── FindReturn.xml
            │       ├── EditReturn.xml
            │       ├── FindRequest.xml
            │       ├── EditRequest.xml
            │       └── ... (more customer screens)
            ├── Supplier/            # Supplier/procurement operations
            │   ├── Supplier.xml
            │   └── Supplier/
            │       ├── SupplierDashboard.xml
            │       ├── FindOrder.xml      (Purchase Orders)
            │       ├── EditOrder.xml
            │       ├── FindReturn.xml
            │       ├── EditReturn.xml
            │       └── ... (more supplier screens)
            ├── Internal/            # Internal operations
            │   ├── Internal.xml
            │   └── Internal/
            │       ├── InternalDashboard.xml
            │       ├── FindShipment.xml
            │       ├── EditShipment.xml
            │       ├── FindRun.xml
            │       ├── FindAsset.xml
            │       ├── EditAsset.xml
            │       └── ... (more internal screens)
            ├── Accounting/          # GL, AR, AP, reports
            │   ├── Accounting.xml
            │   └── Accounting/
            │       ├── AccountingDashboard.xml
            │       ├── FindInvoice.xml
            │       ├── EditInvoice.xml
            │       ├── FindPayment.xml
            │       ├── EditPayment.xml
            │       ├── FindTransaction.xml
            │       ├── GlAccountReport.xml
            │       └── ... (more accounting screens)
            ├── Project/             # Project & task management (from HiveMind)
            │   ├── Project.xml
            │   └── Project/
            │       ├── ProjectDashboard.xml
            │       ├── FindProject.xml
            │       ├── EditProject.xml
            │       ├── FindTask.xml
            │       ├── EditTask.xml
            │       ├── MyTasks.xml
            │       ├── MyTime.xml
            │       └── ... (more project screens)
            ├── Configure/           # System configuration
            │   ├── Configure.xml
            │   └── Configure/
            │       ├── ProductStore.xml
            │       ├── Facility.xml
            │       ├── Party.xml
            │       ├── Product.xml
            │       ├── Category.xml
            │       └── ... (more config screens)
            ├── QuickSearch.xml      # Global search (QuickLookup + Search combined)
            └── ... (dashboard, common screens)
```

### MoquiConf.xml (Screen Mounting)

MarbleERP mounts under `apps.xml` to appear as a standalone app on the home App Listing screen:

```xml
<moqui-conf xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/moqui-conf-3.xsd">
    <screen-facade>
        <!-- Mount marble as standalone app on home screen -->
        <screen location="component://webroot/screen/webroot/apps.xml">
            <subscreens-item name="marble"
                location="component://MarbleERP/screen/marble.xml"
                menu-title="Marble ERP" menu-index="6"/>
        </screen>
    </screen-facade>
</moqui-conf>
```

> **Key distinction**: Mounting under `apps.xml` makes the app appear on the home screen. Mounting under `marble.xml` adds a tab inside MarbleERP. When creating a custom component, choose based on whether it's an independent app or a MarbleERP extension module.

---

## Module Reference

### Customer Module (`/marble/Customer`)

Handles all customer-facing (B2C/B2B) operations — the **Order to Cash** process:

| Screen | Purpose | Key Mantle Services |
|--------|---------|-------------------|
| CustomerDashboard | Overview, KPIs, recent activity | Various info services |
| FindOrder | Sales order search/list | `entity-find OrderHeader` |
| EditOrder | Sales order detail, items, payments, shipments | `get#OrderDisplayInfo`, `create#OrderItem` |
| FindReturn | Customer return search | `entity-find ReturnHeader` |
| EditReturn | Return detail & processing | Return services |
| FindRequest | Customer request/ticket search | `entity-find RequestItem` |
| EditRequest | Request detail & response | Request services |
| FindParty | Customer search | `find#Party` with Customer role |
| EditParty | Customer detail, contacts, orders | `get#PartyContactInfo` |
| FindQuote | Quote/estimate search | `entity-find OrderHeader` (quote type) |

### Supplier Module (`/marble/Supplier`)

Handles procurement — the **Procure to Pay** process:

| Screen | Purpose | Key Mantle Services |
|--------|---------|-------------------|
| SupplierDashboard | Procurement overview | Various info services |
| FindOrder | Purchase order search | `entity-find OrderHeader` (PO type) |
| EditOrder | PO detail, items, receiving | `get#OrderDisplayInfo` |
| FindReturn | Purchase returns (returns to vendor) | `entity-find ReturnHeader` |
| FindParty | Vendor/supplier search | `find#Party` with Vendor/Supplier role |
| EditParty | Supplier detail, POs, invoices | `get#PartyContactInfo` |

### Internal Module (`/marble/Internal`)

Internal operations — warehouse, inventory, fulfillment:

| Screen | Purpose | Key Mantle Services |
|--------|---------|-------------------|
| InternalDashboard | Operations overview | Various info services |
| FindShipment | Shipment search (all types) | `entity-find Shipment` |
| EditShipment | Shipment detail, pick, pack, ship | `pack#ShipmentProduct`, `ship#Shipment` |
| FindAsset | Inventory/asset search | `entity-find Asset` |
| EditAsset | Asset detail, movements | Asset services |
| FindRun | Production run search | WorkEffort-based |
| EditRun | Production run detail | WorkEffort services |
| FindFacility | Warehouse/facility search | `entity-find Facility` |
| EditFacility | Facility detail, locations | Facility services |

### Accounting Module (`/marble/Accounting`)

Financial management — GL, AR, AP:

| Screen | Purpose | Key Mantle Services |
|--------|---------|-------------------|
| AccountingDashboard | Financial overview, aging | Various info services |
| FindInvoice | Invoice search (AR & AP) | `entity-find Invoice` |
| EditInvoice | Invoice detail, items, payments | Invoice services |
| FindPayment | Payment search | `entity-find Payment` |
| EditPayment | Payment detail, application | Payment services |
| FindTransaction | GL transaction search | `entity-find AcctgTrans` |
| EditTransaction | GL entry detail | Transaction services |
| GlAccountReport | Trial balance, P&L, balance sheet | Report services |
| FindTimePeriod | Fiscal periods | TimePeriod entities |
| EditTimePeriod | Period close/open | TimePeriod services |

### Project Module (`/marble/Project`)

Project & task management (originated from HiveMind):

| Screen | Purpose | Key Mantle Services |
|--------|---------|-------------------|
| ProjectDashboard | Project overview | Various info services |
| FindProject | Project search | `entity-find WorkEffort` (project type) |
| EditProject | Project detail, milestones, team | WorkEffort services |
| FindTask | Task search | `entity-find WorkEffort` (task type) |
| EditTask | Task detail, time entries, notes | WorkEffort services |
| MyTasks | Current user's assigned tasks | Filter by current user |
| MyTime | Current user's time entries | TimeEntry entity |
| FindTimeEntry | Time entry search/report | `entity-find TimeEntry` |

### Configure Module (`/marble/Configure`)

System and business configuration:

| Screen | Purpose |
|--------|---------|
| ProductStore | E-commerce store settings, payment gateways, shipping |
| Facility | Warehouses, locations, contact info |
| Party | Internal org, employees, roles |
| Product | Product catalog, categories, features, prices |
| Category | Product category trees |
| Accounting | Chart of accounts, GL settings, fiscal periods |

---

## Screen Hierarchy & Navigation

### URL Path Structure

```
/qapps/marble                          → marble.xml (app root with module tabs)
/qapps/marble/Customer                 → Customer.xml (customer dashboard / submenu)
/qapps/marble/Customer/FindOrder       → FindOrder.xml (order list)
/qapps/marble/Customer/EditOrder       → EditOrder.xml (order detail)
/qapps/marble/Supplier/FindOrder       → FindOrder.xml (PO list)
/qapps/marble/Internal/FindShipment    → FindShipment.xml (shipment list)
/qapps/marble/Accounting/FindInvoice   → FindInvoice.xml (invoice list)
/qapps/marble/Project/FindTask         → FindTask.xml (task list)
```

### Navigation Architecture

The `marble.xml` root screen defines module tabs via subscreens:

```xml
<screen default-menu-title="Marble ERP" default-menu-index="1">
    <subscreens default-item="Customer"/>
    <!-- Modules appear as top-level tabs -->
    <!-- Customer, Supplier, Internal, Accounting, Project, Configure -->
    <widgets>
        <subscreens-panel id="marble-content" type="tab"/>
    </widgets>
</screen>
```

Each module screen (e.g., `Customer.xml`) then has its own subscreens:

```xml
<screen default-menu-title="Customer">
    <subscreens default-item="CustomerDashboard"/>
    <!-- FindOrder, EditOrder, FindReturn, etc. -->
</screen>
```

---

## Extending MarbleERP

### Pattern 1: Add a Custom Module Tab

Create your component with a screen mounted under marble:

```xml
<!-- YourComponent/MoquiConf.xml -->
<moqui-conf>
    <screen-facade>
        <screen location="component://MarbleERP/screen/marble.xml">
            <subscreens-item name="MyModule"
                location="component://YourComponent/screen/MyModule.xml"
                menu-title="My Module" menu-index="15"/>
        </screen>
    </screen-facade>
</moqui-conf>
```

### Pattern 2: Add a Screen Under Existing Module

Mount a screen under Customer, Supplier, etc.:

```xml
<!-- YourComponent/MoquiConf.xml -->
<moqui-conf>
    <screen-facade>
        <screen location="component://MarbleERP/screen/marble/Customer.xml">
            <subscreens-item name="MyCustomerScreen"
                location="component://YourComponent/screen/marble/Customer/MyCustomerScreen.xml"
                menu-title="My Screen" menu-index="20"/>
        </screen>
    </screen-facade>
</moqui-conf>
```

### Pattern 3: Override an Existing MarbleERP Screen

Use a higher-priority subscreens-item to replace a screen:

```xml
<moqui-conf>
    <screen-facade>
        <screen location="component://MarbleERP/screen/marble/Customer.xml">
            <!-- Override FindOrder with your custom version -->
            <subscreens-item name="FindOrder"
                location="component://YourComponent/screen/marble/Customer/FindOrder.xml"
                menu-title="Orders" menu-index="2"/>
        </screen>
    </screen-facade>
</moqui-conf>
```

### Pattern 4: Add Custom Dashboard Widgets

Dashboards often use `section` and `container-row` patterns. Add your own by extending with include-screen or adding in your MoquiConf.

### Pattern 5: Create a Standalone App (Separate from MarbleERP)

If your component is an independent application (not a MarbleERP module), mount it under `apps.xml` instead of `marble.xml`. This makes it appear as its own tile on the home App Listing screen:

```xml
<!-- YourComponent/MoquiConf.xml -->
<moqui-conf>
    <screen-facade>
        <!-- Mount as standalone app on home screen (NOT inside MarbleERP) -->
        <screen location="component://webroot/screen/webroot/apps.xml">
            <subscreens-item name="myapp" menu-title="My Application" menu-index="20"
                location="component://YourComponent/screen/MyApp.xml"/>
        </screen>
    </screen-facade>
</moqui-conf>
```

**You also need:**
1. A root app screen with `server-static="vuet,qvt"` and `subscreens-panel type="popup"` (see SCREENS.md → App Registration)
2. Artifact authorization seed data (see [Artifact Authorization](#artifact-authorization) section)

> Use this pattern when your app is a separate business domain (e.g., Employee Management, Teacher Management) rather than an extension of MarbleERP's existing modules.

### Pattern 6: Hook Business Logic (Without Touching MarbleERP)

Since MarbleERP has no custom services, all business logic hooks go through Mantle USL via SECA/EECA in your own component:

```xml
<!-- YourComponent/service/mycomp/OrderSeca.secas.xml -->
<secas>
    <seca service="mantle.order.OrderServices.place#Order" when="tx-commit">
        <actions>
            <service-call name="mycomp.MyOrderServices.after#OrderPlaced"
                in-map="context" async="true"/>
        </actions>
    </seca>
</secas>
```

---

## SimpleScreens Reuse Patterns

MarbleERP heavily reuses screens from SimpleScreens. Understanding this is critical for development:

### How Reuse Works

MarbleERP screens typically do one of:

**1. Direct Include** — Include a SimpleScreens screen:
```xml
<screen>
    <widgets>
        <include-screen location="component://SimpleScreens/screen/SimpleScreens/Order/FindOrder.xml"/>
    </widgets>
</screen>
```

**2. Wrapper with Additions** — Wrap SimpleScreens content with MarbleERP-specific additions:
```xml
<screen>
    <actions>
        <!-- Additional data prep -->
    </actions>
    <widgets>
        <!-- MarbleERP-specific header/nav -->
        <include-screen location="component://SimpleScreens/screen/SimpleScreens/Party/FindParty.xml"/>
        <!-- MarbleERP-specific footer/actions -->
    </widgets>
</screen>
```

**3. Transition Includes** — Reuse transitions from SimpleScreens:
```xml
<screen>
    <transition-include name="getPartyList"
        location="component://SimpleScreens/template/party/PartyForms.xml"/>
    <transition-include name="getProductList"
        location="component://SimpleScreens/template/product/ProductTransitions.xml"/>
</screen>
```

### Key SimpleScreens Locations

| Area | SimpleScreens Path | Used By |
|------|-------------------|---------|
| Order Find/Edit | `SimpleScreens/screen/SimpleScreens/Order/` | Customer & Supplier |
| Party Find/Edit | `SimpleScreens/screen/SimpleScreens/Party/` | All modules |
| Product Find/Edit | `SimpleScreens/screen/SimpleScreens/Catalog/` | Configure |
| Asset/Inventory | `SimpleScreens/screen/SimpleScreens/Asset/` | Internal |
| Shipment | `SimpleScreens/screen/SimpleScreens/Shipment/` | Internal |
| Invoice | `SimpleScreens/screen/SimpleScreens/Accounting/Invoice/` | Accounting |
| Payment | `SimpleScreens/screen/SimpleScreens/Accounting/Payment/` | Accounting |
| GL/Transaction | `SimpleScreens/screen/SimpleScreens/Accounting/Transaction/` | Accounting |
| Reports | `SimpleScreens/screen/SimpleScreens/Accounting/Reports/` | Accounting |
| Facility | `SimpleScreens/screen/SimpleScreens/Asset/Facility/` | Internal/Configure |
| Party templates | `SimpleScreens/template/party/` | Reusable party widgets |
| Product templates | `SimpleScreens/template/product/` | Reusable product widgets |

---

## Configuration & Setup

### Initial Setup Steps

1. **Organization Setup** — Create your internal organization (OrgInternal role)
2. **ProductStore** — Configure store with org, currency, locale, inventory settings
3. **Facility** — Set up warehouses with locations
4. **Chart of Accounts** — Configure GL accounts
5. **Fiscal Periods** — Set up time periods for accounting
6. **Users** — Create user accounts with appropriate roles and permissions

### Demo Data

MarbleERP loads demo data via `./gradlew load` which includes:
- Sample organization (Marble Demo Org)
- Demo users (John Doe, etc.)
- Sample products, categories, prices
- Demo orders, invoices, payments
- Sample projects and tasks

### Setup Data (MarbleErpSetupData.xml)

Contains seed data specific to MarbleERP's app configuration:
- Subscreen items for the marble app
- Default subscreens ordering
- App-specific UserPreference defaults
- Menu configuration
- **Artifact authorization** for screen and API access

### Artifact Authorization

Every app that appears on the home App Listing screen requires artifact authorization seed data. Without it, the app is invisible — `isPermitted()` returns false in `AppList.xml` and the tile is hidden.

MarbleERP's authorization pattern (`MarbleErpSetupData.xml`):

```xml
<entity-facade-xml type="seed-initial">
    <!-- Screen access with entity filter for org-level security -->
    <artifactGroups artifactGroupId="MarbleErpAll" description="Marble ERP (via root screen)">
        <artifacts artifactName="component://MarbleERP/screen/marble.xml"
            artifactTypeEnumId="AT_XML_SCREEN" inheritAuthz="Y"/>
        <authz artifactAuthzId="MARBLE_ERP_ADMIN" userGroupId="ADMIN"
            authzTypeEnumId="AUTHZT_ALWAYS" authzActionEnumId="AUTHZA_ALL">
            <filters entityFilterSetId="MANTLE_ACTIVE_ORG"/></authz>
    </artifactGroups>
</entity-facade-xml>
```

For a simpler custom app (without org-level entity filters):

```xml
<entity-facade-xml type="seed">
    <artifactGroups artifactGroupId="MY_APP" description="My App (via root screen)">
        <artifacts artifactName="component://MyComponent/screen/MyApp.xml"
            artifactTypeEnumId="AT_XML_SCREEN" inheritAuthz="Y"/>
        <authz artifactAuthzId="MY_APP_ADMIN" userGroupId="ADMIN"
            authzTypeEnumId="AUTHZT_ALWAYS" authzActionEnumId="AUTHZA_ALL"/>
    </artifactGroups>

    <!-- REST API access (if applicable) -->
    <artifactGroups artifactGroupId="MY_API" description="My App REST API">
        <artifacts artifactTypeEnumId="AT_REST_PATH" inheritAuthz="Y" artifactName="/myapp"/>
        <authz artifactAuthzId="MY_API_ADMIN" userGroupId="ADMIN"
            authzTypeEnumId="AUTHZT_ALWAYS" authzActionEnumId="AUTHZA_ALL"/>
    </artifactGroups>
</entity-facade-xml>
```

**Key rules:**
- `artifactGroupId` and `artifactAuthzId` must be globally unique
- `inheritAuthz="Y"` means all subscreens/sub-endpoints inherit the authorization
- Add `<filters entityFilterSetId="MANTLE_ACTIVE_ORG"/>` if the app needs org-level data filtering
- Add multiple `<authz>` elements to grant access to different user groups
- Data type should be `seed` (reloaded every time) for authorization records
- After creating/modifying, reload data: `java -jar moqui.war load types=seed`

---

## Quasar UI (qapps)

MarbleERP uses the Quasar-based Vue.js UI (the `qvt` render mode), accessible at `/qapps/marble`.

### Render Modes in Moqui

| Path | Mode | Technology | Status |
|------|------|------------|--------|
| `/qapps/` | qvt | Quasar + Vue.js (Material Design) | **Current default** |
| `/vapps/` | vuet | Vue.js (Bootstrap) | Maintained |
| `/apps/` | html | Server-rendered HTML (Bootstrap) | Legacy |

### Quasar UI Features
- **Material Design** look and feel via Quasar Framework
- **Single Page Application** shell with client-side routing
- **Hybrid rendering** — XML Screens rendered server-side, delivered to Vue SPA
- **Responsive** — Works on desktop, tablet, mobile
- **Search box** in header goes to QuickSearch (combined QuickLookup + Search)
- **Theme** customizable via theme data entities

### Custom Vue Components

For advanced screens, you can create Vue component-based screens:
```
MyScreen.xml       ← Screen definition (widgets, actions)
MyScreen.js        ← Vue component JavaScript
MyScreen.vuet      ← Optional Vue template (merged into component)
```

---

## Common Customization Recipes

### Recipe 1: Add Custom Field to Order Screen

```xml
<!-- 1. Extend entity (in your component's entity/) -->
<extend-entity entity-name="OrderHeader" package="mantle.order">
    <field name="myCustomField" type="text-medium"/>
</extend-entity>

<!-- 2. Add to EditOrder screen (your component's screen/) -->
<!-- Override or include-screen with additions -->
```

### Recipe 2: Add Custom Report to Accounting

```xml
<!-- YourComponent/MoquiConf.xml -->
<screen-facade>
    <screen location="component://MarbleERP/screen/marble/Accounting.xml">
        <subscreens-item name="MyReport"
            location="component://YourComponent/screen/marble/Accounting/MyReport.xml"
            menu-title="My Report" menu-index="20"/>
    </screen>
</screen-facade>
```

### Recipe 3: Custom Order Processing Workflow

```xml
<!-- 1. Define custom status flow (data file) -->
<moqui.basic.StatusFlowTransition statusFlowId="Default"
    statusId="OrderApproved" toStatusId="MyCustomStatus" transitionName="Review"/>

<!-- 2. SECA rule for custom logic -->
<seca service="mantle.order.OrderServices.update#OrderStatus" when="post-service">
    <condition><expression>statusId == 'MyCustomStatus'</expression></condition>
    <actions>
        <service-call name="mycomp.MyOrderWorkflow.handle#CustomStatus" in-map="context"/>
    </actions>
</seca>
```

### Recipe 4: Integrate External System on Shipment

```xml
<!-- SECA on ship to push to external WMS/3PL -->
<seca service="mantle.shipment.ShipmentServices.ship#Shipment" when="tx-commit">
    <actions>
        <service-call name="mycomp.integration.WmsServices.send#ShipmentToWms"
            in-map="[shipmentId: shipmentId]" async="true"/>
    </actions>
</seca>
```

### Recipe 5: Custom Dashboard for a Module

Create a dashboard with KPIs:
```xml
<screen default-menu-title="My Dashboard" default-menu-index="1">
    <actions>
        <!-- Order KPIs -->
        <entity-find-count entity-name="mantle.order.OrderHeader" count-field="pendingOrders">
            <econdition field-name="statusId" value="OrderApproved"/>
        </entity-find-count>
        <entity-find-count entity-name="mantle.shipment.Shipment" count-field="readyToShip">
            <econdition field-name="statusId" value="ShipPacked"/>
        </entity-find-count>
        <!-- Revenue this month -->
        <entity-find entity-name="mantle.order.OrderHeaderAndPart" list="recentOrders">
            <econdition field-name="statusId" operator="in" value="OrderApproved,OrderSent,OrderCompleted"/>
            <econdition field-name="placedDate" operator="greater-equals"
                from="ec.l10n.parseTimestamp(ec.l10n.format(ec.user.nowTimestamp, 'yyyy-MM') + '-01 00:00:00', null)"/>
        </entity-find>
    </actions>
    <widgets>
        <container-row>
            <row-col lg="3" md="6">
                <container style="card text-center p-3">
                    <label text="${pendingOrders}" type="h2" style="text-primary"/>
                    <label text="Pending Orders"/>
                </container>
            </row-col>
            <row-col lg="3" md="6">
                <container style="card text-center p-3">
                    <label text="${readyToShip}" type="h2" style="text-success"/>
                    <label text="Ready to Ship"/>
                </container>
            </row-col>
        </container-row>
    </widgets>
</screen>
```

---

## Best Practices for MarbleERP Development

1. **Never modify MarbleERP or SimpleScreens source** — Always use your own component
2. **Use MoquiConf.xml for screen mounting** — Add/override screens declaratively
3. **Use SECA/EECA for business logic hooks** — Don't change Mantle USL
4. **Reuse SimpleScreens templates** — Don't rebuild what already exists
5. **Follow the module pattern** — Keep Customer/Supplier/Internal separation
6. **Test with demo data** — MarbleERP's demo data covers most business scenarios
7. **Use `component://` URLs** — Always reference screens by component path, not filesystem
8. **Match existing UI patterns** — Use the same form-list/form-single patterns for consistency
9. **Add to Configure module** — Put admin/setup screens under Configure, not mixed with operations
10. **Think process-first** — Organize screens around business processes (Order-to-Cash, Procure-to-Pay), not entities

## Verify

After extending or customizing a MarbleERP screen:
1. Navigate to `qapps/marble` → the affected module (Customer / Supplier / Internal / Accounting / Project / Configure).
2. Confirm the screen appears in the module's menu and renders without errors.
3. For form/list extensions: confirm new fields/columns appear and existing data isn't broken.
4. Check artifact authorization if this is a new screen/subscreen root.

Common failures: directly editing `MarbleERP/` (blocked by the protected-zone hook — use `MoquiConf.xml` overrides or new subscreens in your custom component); missing artifact-auth (screen invisible).

See [`../SKILL.md`](../SKILL.md) §"Verify After Each Pattern" for the cross-cutting table.

<!-- begin: related-lessons (auto-generated by scripts/regen-skill-lesson-links.sh — do not edit by hand) -->

## Related Lessons

Cross-references from [`docs/ai/lessons.md`](../../docs/ai/lessons.md) that apply to this area. Filter tags: `Moqui-Services,Process`. Regenerate with `scripts/regen-skill-lesson-links.sh`.

- 2026-04-15 — HSTO backend directory name differs from component name  `[Process,Moqui-Entities]`

<!-- end: related-lessons -->
