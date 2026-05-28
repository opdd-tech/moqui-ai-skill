---
name: moqui-framework
description: "Use this skill for ANY development task involving the Moqui Framework ecosystem including moqui-framework, moqui-runtime, mantle-udm (Universal Data Model), mantle-usl (Universal Service Library), SimpleScreens, and MarbleERP. Triggers include: creating or editing XML entity definitions, XML service definitions, XML screen files, XML form definitions, writing Groovy scripts for Moqui, creating data load files, designing REST APIs, configuring moqui-conf XML, building e-commerce or ERP features, working with Mantle business logic (orders, invoices, payments, shipments, products, parties, accounting), creating or modifying screen transitions, working with Entity ECA or Service ECA rules, extending entities, creating view-entities, writing XML Actions, configuring ProductStore, building admin screens with SimpleScreens patterns, extending or customizing MarbleERP modules (Customer, Supplier, Internal, Accounting, Project, Configure), and any mention of 'Moqui', 'Mantle', 'MarbleERP', 'Marble ERP', 'mantle-udm', 'mantle-usl', 'SimpleScreens', 'XML Screen', 'XML Form', 'entity-find', 'service-call', 'ExecutionContext', 'qapps', or 'Quasar UI'. Also trigger when user references ERP, order management, inventory, warehouse, fulfillment, procure-to-pay, order-to-cash, or accounting in a Moqui context. Do NOT use for unrelated Java/Groovy projects or other frameworks."
---

# Moqui Framework Development Skill

> **Before writing code in the HSTO monorepo**, also read [`docs/ai/lessons.md`](../docs/ai/lessons.md) — cross-session lessons that supersede any guidance below where they conflict.

## Overview

Moqui Framework is an all-in-one enterprise application framework based on Java and Groovy. It provides tools for databases, services, screens/forms, security, localization, caching, and integration. The ecosystem consists of:

- **moqui-framework** — Core framework (Entity Facade, Service Facade, Screen Facade, etc.)
- **moqui-runtime** — Default runtime directory with configuration, components, webroot screens
- **mantle-udm** — Universal Data Model (Party, Product, Order, Shipment, Invoice, Payment, Accounting, etc.)
- **mantle-usl** — Universal Service Library (business logic services for all Mantle UDM entities)
- **SimpleScreens** — Admin/back-office UI screens built on Mantle
- **MarbleERP** — Full ERP application (Customer, Supplier, Internal, Accounting, Project modules)

> **Before writing any code**, read the relevant reference file(s) in this skill's `references/` directory:
> - `references/ENTITIES.md` — Entity definitions, view-entities, extend-entity, field types, relationships, ECA rules
> - `references/SERVICES.md` — Service definitions, XML Actions, service types, SECA rules, REST API
> - `references/SCREENS.md` — XML Screens, XML Forms, transitions, widgets, subscreens, menus
> - `references/MANTLE.md` — Mantle UDM/USL entity packages, key entities, service naming, status flows, business processes
> - `references/MARBLE_ERP.md` — MarbleERP application modules, screen hierarchy, extending patterns, customization recipes

---

## Before You Code (Moqui)

Run this five-question check before opening an editor:

1. **Success condition.** State the user-visible outcome in one sentence. If you can't, the requirement is unclear — go back to `work/{TICKET}/requirement.md` §1.
2. **What could break if my model is wrong?** Mantle's data model is opinionated. Wrong assumptions about FKs, status flows, bridge entities, or extension vs new entity will surface as silent miswires. Skim `references/MANTLE.md` for the entity package you're touching.
3. **Does it already exist?** Grep `references/MANTLE.md`, `references/SERVICES.md`, and the `mantle-usl` source for a service with the verb you'd write. The strongest answer to "should I build this?" is "Mantle already has it."
4. **What lessons apply?** Search `docs/ai/lessons.md` for the tag (`[Moqui-Entities]`, `[Mantle-Reuse]`, `[Catalog]`, etc.). Lessons supersede this skill where they conflict.
5. **What's the smallest change?** Prefer `extend-entity` over a new entity. Prefer `SECA` over wrapping a stock service. Prefer a custom service that calls Mantle services over duplicating their logic. (This is the *Demand Elegance* gate from AGENTS.md §10.)

---

## Anti-Patterns — DO NOT

The protected-zone PreToolUse hook (`scripts/claude-hooks/protected-zone-guard.sh`) already blocks the hardest of these at the tool layer (modifying `mantle-udm`, `mantle-usl`, `SimpleScreens`, `MarbleERP`, `framework/`, or vendored components). The soft DO-NOTs the hook can't catch:

- **DO NOT put business logic in screens.** All business logic goes in services. Screens orchestrate, services compute. (Logic in screens is invisible to REST, untestable, and unreachable from SECA.)
- **DO NOT hardcode status IDs.** Use the Mantle enum constants (`OrderApproved`, `WeInProgress`, `PartyActive`, etc.). String literals drift and break when seed data is renamed.
- **DO NOT create a new entity when `extend-entity` would work.** Mantle's tables are designed to be extended. Adding columns is cheap; adding aggregates is expensive and pollutes the data model.
- **DO NOT add a direct FK when a stock bridge entity exists.** E.g. don't add `workEffortId` to `OrderItem` — use `OrderItemWorkEffort`. Don't add per-target FKs to `CommunicationEvent` — use `CommEventOrder` / `CommEventWorkEffort`. (See lessons tagged `[Mantle-Reuse]`.)
- **DO NOT skip artifact authorization for new apps or REST resources.** New `qapps` and REST resources are invisible by default unless authorized in `data/SecurityTypeData.xml`. The symptom is a 404 or a missing tile in the app list.
- **DO NOT call services or read entities directly from XML Screens when a `service-call` or `entity-find` already does it.** XML Screens have first-class elements for both.
- **DO NOT load business data via `seed-initial`.** That's reserved for framework primitives. Use `seed` for type/enum data, `install` for required runtime data, `demo` for sample data.
- **DO NOT write a SECA without an `if` predicate.** Unconditional SECAs fire on every call to the trigger service — including failure paths and rollback paths.

---

## Verify After Each Pattern

The patterns later in this skill are paired with `## Verify` blocks in the `references/*.md` files. The common verification recipes:

| Change | How to verify locally |
|--------|----------------------|
| New / extended entity | `cd moqui-framework && ./gradlew load` → tail `runtime/log/moqui.log` for `Entity {name} loaded`. On failure search for `XSDValidationError` or `ConstraintViolation`. |
| New service | Call from `qapps/tools` Service Run UI with sample input. Confirm response shape; confirm any SECA fired (check log for `Calling SECA ...`). |
| New REST resource | `curl -u admin:moqui http://localhost:8080/rest/s1/landmark/{resource}` — confirm 200 and the expected JSON shape; confirm 401 without auth. |
| New SECA | Trigger the underlying service; confirm log shows `Calling SECA {name}` and the side effect occurred. If silent, check the `if` predicate. |
| New screen / subscreen | Navigate to it in `qapps/marble` (or wherever it's mounted); confirm artifact auth is set if it's a new app root. |
| Seed data change | `./gradlew load -Ptypes=seed` then query the affected entity via Entity Data UI under `qapps/tools`. |

For *all* backend changes: `git diff` for unintended modifications; run `./gradlew :runtime:component:landmark-hsto-core:compileTestGroovy` (and tests if they exist); paste output into `work/{TICKET}/review.md` §5 *Verification Evidence*.

---

## Architecture Quick Reference

### Project Structure

```
moqui/                          # Root (executable WAR)
├── framework/                  # moqui-framework core
│   ├── src/main/               # Java/Groovy source
│   ├── entity/                 # Framework entities (moqui.* packages)
│   ├── service/                # Framework services
│   └── screen/                 # Framework screens (Tools app)
├── runtime/                    # moqui-runtime
│   ├── conf/                   # MoquiProductionConf.xml, etc.
│   ├── component/              # All add-on components live here
│   │   ├── mantle-udm/         # Data model definitions
│   │   │   ├── entity/         # Entity XML files (*.xml)
│   │   │   └── data/           # Seed/demo data files
│   │   ├── mantle-usl/         # Service library
│   │   │   ├── service/        # Service XML files
│   │   │   └── data/           # Service-related data
│   │   ├── SimpleScreens/      # Admin UI screens
│   │   │   ├── screen/         # Screen XML hierarchy
│   │   │   └── template/       # FTL templates
│   │   └── YourComponent/      # Custom component
│   │       ├── entity/         # Entity definitions: {Domain}Entities.xml,
│   │       │                   #   {Domain}ExtendedEntities.xml, {Domain}ViewEntities.xml,
│   │       │                   #   {Domain}Entities.eecas.xml (all flat, no subfolders)
│   │       ├── service/        # Service definitions (.xml, .secas.xml)
│   │       ├── screen/         # Screen hierarchy
│   │       ├── data/           # Data load files
│   │       ├── lib/            # JARs (added to classpath)
│   │       └── script/         # Groovy/other scripts
│   ├── db/                     # Database files (H2 default)
│   └── log/                    # Log files
```

### Component Directory Convention

All files in these directories are auto-discovered:

| Directory | Contents | Auto-loaded | Structure |
|-----------|----------|-------------|-----------|
| `entity/` | Entity definitions (*.xml), Entity ECA rules (*.eecas.xml) | Yes | **Flat** — no subfolders |
| `service/` | Service definitions (*.xml), Service ECA rules (*.secas.xml) | Yes | Hierarchical — subdirs map to package |
| `data/` | Entity XML data files (seed, seed-initial, install, demo) | Via `-load` | Flat |
| `screen/` | XML Screens (referenced by component:// URL) | By reference | Hierarchical (URL path) |
| `script/` | Groovy/XML Action scripts (by component:// URL) | By reference | By reference |
| `lib/` | JAR files added to classpath | Yes | Flat |

> **Important**: Entity files go directly in `entity/` with NO package subfolders. The entity package is defined via the XML `package` attribute, not the directory structure. Service files DO use subdirectories that map to the Java package name (e.g., `service/mycomp/myapp/MyServices.xml` → service name `mycomp.myapp.MyServices.verb#Noun`).

#### Entity File Naming Convention

Organize entity files by **type and domain** — one file per domain per type:

| File Type | Naming Pattern | Example |
|-----------|---------------|---------|
| New entities | `{Domain}Entities.xml` | `StudentEntities.xml`, `ProjectEntities.xml` |
| Extended entities | `{Domain}ExtendedEntities.xml` | `OrderExtendedEntities.xml`, `PartyExtendedEntities.xml` |
| View entities | `{Domain}ViewEntities.xml` | `OrderViewEntities.xml`, `StudentViewEntities.xml` |
| Entity ECA rules | `{Domain}Entities.eecas.xml` | `OrderEntities.eecas.xml` |

**Key rules:**
- **Group related extend-entity definitions in a single file** — e.g., ALL order-related `extend-entity` definitions go in `OrderExtendedEntities.xml`
- **Group related view-entity definitions in a single file** — e.g., ALL order-related `view-entity` definitions go in `OrderViewEntities.xml`
- **Keep extend-entity and view-entity separate from new entity definitions** — Do NOT mix them in the same file (exception: small components with 1-2 tightly coupled view entities for their own custom entities)
- See `references/ENTITIES.md` → [File Naming Conventions](references/ENTITIES.md#file-naming-conventions) for full details and examples

### Execution Context (ec)

The central API object, available in all XML Actions and Groovy scripts:

```groovy
ec.entity      // EntityFacade - CRUD, find, queries
ec.service     // ServiceFacade - call services
ec.user        // UserFacade - current user, login, preferences
ec.message     // MessageFacade - messages and errors
ec.resource    // ResourceFacade - files, scripts, templates
ec.logger      // LoggerFacade - logging
ec.cache       // CacheFacade - caching
ec.transaction // TransactionFacade - JTA transactions
ec.screen      // ScreenFacade - screen rendering
ec.l10n        // L10nFacade - localization, formatting
ec.web         // WebFacade - servlet request/response (web context only)
```

---

## XSD Schema Compliance

**All Moqui XML files MUST conform to the relevant XSD schema** in `framework/xsd/`. Using invalid elements, misspelled attributes, or wrong nesting will cause runtime errors. Always check the correct XSD for the artifact type you are writing.

### XSD-to-Artifact Mapping

| XSD File | Root Element(s) | Used By | Component File Pattern |
|----------|----------------|---------|----------------------|
| `entity-definition-3.xsd` | `entities`, `entity`, `extend-entity`, `view-entity` | Entity definitions | `entity/*.xml` |
| `entity-eca-3.xsd` | `eecas`, `eeca` | Entity Event Condition Actions | `entity/*.eecas.xml` |
| `service-definition-3.xsd` | `services`, `service` | Service definitions | `service/**/*.xml` |
| `service-eca-3.xsd` | `secas`, `seca` | Service Event Condition Actions | `service/**/*.secas.xml` |
| `rest-api-3.xsd` | `resource`, `id`, `method` | REST API resource definitions | `service/*.rest.xml` *(directly in service/, NOT in subdirectories)* |
| `xml-screen-3.xsd` | `screen`, `screen-extend` | Screen definitions | `screen/**/*.xml` |
| `xml-form-3.xsd` | `form-single`, `form-list` | Form widgets (embedded in screens) | `screen/**/*.xml` |
| `xml-actions-3.xsd` | `actions`, `condition` | Action blocks (embedded in services, screens, ECAs) | Embedded within other artifacts |
| `moqui-conf-3.xsd` | `moqui-conf` | Framework/component configuration | `MoquiConf.xml`, `runtime/conf/*.xml` |
| `email-eca-3.xsd` | `emecas`, `emeca` | Email Event Condition Actions | `entity/*.emecas.xml` |
| `common-types-3.xsd` | *(type definitions only)* | Shared types used by all other XSDs | *(Not directly referenced)* |

### Key Rules

1. **Always set `xsi:noNamespaceSchemaLocation`** on root elements to reference the correct XSD:
   ```xml
   <!-- Entity files -->
   <entities xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/entity-definition-3.xsd">

   <!-- Service files -->
   <services xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/service-definition-3.xsd">

   <!-- Screen files -->
   <screen xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/xml-screen-3.xsd">

   <!-- REST API files -->
   <resource xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/rest-api-3.xsd">

   <!-- EECA files -->
   <eecas xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/eeca-definition-3.xsd">

   <!-- SECA files -->
   <secas xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/service-eca-definition-3.xsd">

   <!-- MoquiConf files -->
   <moqui-conf xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/moqui-conf-3.xsd">
   ```

2. **Use only valid elements defined in the XSD** — Never invent element names. For XML Actions (`xml-actions-3.xsd`), all entity operation elements are prefixed with `entity-` (e.g., `entity-make-value`, `entity-find-one`, `entity-sequenced-id-primary`). Never use shortened names like `make-value` — they will cause template errors at runtime. See `references/SERVICES.md` for the complete valid action element list.

3. **Validate element nesting** — Each XSD defines what child elements are allowed. For example, `<econdition>` goes inside `<entity-find>`, not outside it. `<field>` goes inside `<entity>`, not at the `<entities>` root level.

4. **Check attribute names and types** — Use exact attribute names from the XSD (e.g., `entity-name` not `entityName`, `field-name` not `fieldName`, `join-from-alias` not `joinFromAlias`). Moqui XML uses kebab-case for attributes.

---

## Entity Development (mantle-udm patterns)

### Entity Definition XML

```xml
<entities xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/entity-definition-3.xsd">

    <!-- Standard entity -->
    <entity entity-name="MyEntity" package="mycomp.myapp" short-alias="myEntities">
        <field name="myEntityId" type="id" is-pk="true"/>
        <field name="description" type="text-medium"/>
        <field name="statusId" type="id"/>
        <field name="amount" type="currency-amount"/>
        <field name="quantity" type="number-decimal"/>
        <field name="fromDate" type="date-time"/>
        <field name="partyId" type="id"/>
        <relationship type="one" related="mantle.party.Party" short-alias="party"/>
        <relationship type="one" title="MyEntity" related="moqui.basic.StatusItem" short-alias="status"/>
        <relationship type="many" related="mycomp.myapp.MyEntityItem" short-alias="items">
            <key-map field-name="myEntityId"/></relationship>
        <seed-data>
            <moqui.basic.StatusType statusTypeId="MyEntity" description="My Entity"/>
            <moqui.basic.StatusItem statusId="MyActive" statusTypeId="MyEntity" description="Active"/>
            <moqui.basic.StatusItem statusId="MyClosed" statusTypeId="MyEntity" description="Closed"/>
            <moqui.basic.StatusFlowTransition statusFlowId="Default"
                statusId="MyActive" toStatusId="MyClosed" transitionName="Close"/>
        </seed-data>
    </entity>

    <!-- NOTE: extend-entity and view-entity go in SEPARATE files (see below) -->
</entities>

<!-- File: entity/OrderExtendedEntities.xml — ALL order-related extend-entity in one file -->
<entities ...>
    <extend-entity entity-name="OrderHeader" package="mantle.order">
        <field name="myCustomField" type="text-medium"/>
        <relationship type="one" related="mycomp.myapp.MyEntity"/>
    </extend-entity>
</entities>

<!-- File: entity/OrderViewEntities.xml — ALL order-related view-entity in one file -->
<entities ...>
    <view-entity entity-name="OrderItemDetail" package="mycomp.myapp">
        <member-entity entity-alias="OI" entity-name="mantle.order.OrderItem"/>
        <member-entity entity-alias="OH" entity-name="mantle.order.OrderHeader" join-from-alias="OI">
            <key-map field-name="orderId"/></member-entity>
        <member-entity entity-alias="PROD" entity-name="mantle.product.Product" join-from-alias="OI" join-optional="true">
            <key-map field-name="productId"/></member-entity>
        <alias entity-alias="OI" name="orderId"/>
        <alias entity-alias="OI" name="orderItemSeqId"/>
        <alias entity-alias="OI" name="unitAmount"/>
        <alias entity-alias="OI" name="quantity"/>
        <alias entity-alias="OH" name="placedDate"/>
        <alias entity-alias="OH" name="statusId"/>
        <alias entity-alias="PROD" name="productName"/>
    </view-entity>
</entities>
```

### Field Types

| Type | Java Type | SQL Default | Notes |
|------|-----------|-------------|-------|
| `id` | String | VARCHAR(40) | PKs, FKs |
| `id-long` | String | VARCHAR(255) | Longer IDs |
| `text-short` | String | VARCHAR(63) | Short text |
| `text-medium` | String | VARCHAR(255) | Medium text |
| `text-long` | String | VARCHAR(4095) | Long text |
| `text-very-long` | String | TEXT/CLOB | Very long text |
| `number-integer` | Long | NUMERIC(20,0) | Integers |
| `number-decimal` | BigDecimal | NUMERIC(26,6) | Decimals |
| `number-float` | Double | DOUBLE | Floating point |
| `currency-amount` | BigDecimal | NUMERIC(22,2) | Money |
| `currency-precise` | BigDecimal | NUMERIC(23,3) | Precise money |
| `date-time` | Timestamp | TIMESTAMP | Date + time |
| `date` | Date | DATE | Date only |
| `time` | Time | TIME | Time only |
| `binary-very-long` | byte[] | BLOB | Binary data |

### Entity ECA Rules (*.eecas.xml)

```xml
<eecas xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/eeca-definition-3.xsd">
    <eeca entity="mantle.order.OrderHeader" on-create="true" on-update="true" run-on-error="false">
        <condition><expression>statusId == 'OrderPlaced' &amp;&amp; statusId_old != 'OrderPlaced'</expression></condition>
        <actions>
            <service-call name="mycomp.myapp.MyServices.handle#OrderPlaced"
                in-map="[orderId: orderId]"/>
        </actions>
    </eeca>
</eecas>
```

---

## Service Development (mantle-usl patterns)

### Service Definition XML

```xml
<services xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/service-definition-3.xsd">

    <service verb="create" noun="MyEntity" type="inline">
        <description>Create a new MyEntity record</description>
        <in-parameters>
            <parameter name="description" required="true"/>
            <parameter name="statusId" default-value="MyActive"/>
            <parameter name="partyId" required="true"/>
            <auto-parameters entity-name="mycomp.myapp.MyEntity" include="nonpk"/>
        </in-parameters>
        <out-parameters>
            <parameter name="myEntityId"/>
        </out-parameters>
        <actions>
            <service-call name="create#mycomp.myapp.MyEntity" in-map="context" out-map="context"/>
        </actions>
    </service>

    <service verb="update" noun="MyEntity">
        <in-parameters>
            <parameter name="myEntityId" required="true"/>
            <auto-parameters entity-name="mycomp.myapp.MyEntity" include="nonpk"/>
        </in-parameters>
        <actions>
            <service-call name="update#mycomp.myapp.MyEntity" in-map="context"/>
        </actions>
    </service>

    <service verb="get" noun="MyEntityDisplayInfo">
        <in-parameters>
            <parameter name="myEntityId" required="true"/>
        </in-parameters>
        <out-parameters>
            <parameter name="myEntity" type="Map"/>
            <parameter name="itemList" type="List"/>
            <parameter name="partyDetail" type="Map"/>
        </out-parameters>
        <actions>
            <entity-find-one entity-name="mycomp.myapp.MyEntity" value-field="myEntity"/>
            <entity-find entity-name="mycomp.myapp.MyEntityItem" list="itemList">
                <econdition field-name="myEntityId"/>
                <order-by field-name="sequenceNum"/>
            </entity-find>
            <service-call name="mantle.party.PartyServices.get#PartyDetail"
                in-map="[partyId: myEntity.partyId]" out-map="partyDetail"/>
        </actions>
    </service>
</services>
```

### Service Naming Convention (Mantle Pattern)

Services follow verb#Noun naming within a file path that maps to the service name:

```
File: service/mycomp/myapp/MyServices.xml
Service name: mycomp.myapp.MyServices.create#MyEntity
```

Common Mantle service file patterns:
```
mantle.order.OrderServices           # Order CRUD, place, approve, cancel
mantle.order.OrderInfoServices       # Order display/query info
mantle.product.ProductServices       # Product CRUD
mantle.product.AssetServices         # Inventory/asset management
mantle.product.PriceServices         # Price calculation
mantle.shipment.ShipmentServices     # Shipment lifecycle
mantle.account.InvoiceServices       # Invoice creation/processing
mantle.account.PaymentServices       # Payment processing
mantle.party.PartyServices           # Party/person/org management
mantle.party.ContactServices         # Addresses, phone, email
```

### CRUD Shorthand Services

Moqui auto-generates CRUD services for any entity:
```xml
<!-- These don't need XML definitions — they exist automatically -->
<service-call name="create#mantle.order.OrderHeader" in-map="context" out-map="context"/>
<service-call name="update#mantle.order.OrderHeader" in-map="context"/>
<service-call name="delete#mantle.order.OrderHeader" in-map="[orderId: orderId]"/>
<service-call name="store#mantle.order.OrderHeader" in-map="context" out-map="context"/>
```

### XML Actions Reference

> **IMPORTANT**: All XML must conform to the relevant Moqui XSD schema. See [XSD Schema Compliance](#xsd-schema-compliance) below for the full mapping.

```xml
<actions>
    <!-- Variables -->
    <set field="myVar" value="literal"/>
    <set field="myVar" from="someExpression"/>
    <set field="myVar" from="ec.user.nowTimestamp"/>

    <!-- Entity Operations -->
    <entity-find-one entity-name="mantle.order.OrderHeader" value-field="orderHeader"/>
    <entity-find-one entity-name="mantle.order.OrderHeader" value-field="orderHeader" for-update="true"/>

    <entity-find entity-name="mantle.order.OrderItem" list="itemList">
        <econdition field-name="orderId"/>
        <econdition field-name="statusId" value="OrderApproved"/>
        <econdition field-name="quantity" operator="greater" from="0"/>
        <date-filter/>  <!-- Checks fromDate/thruDate against now -->
        <select-field field-name="orderId"/>
        <select-field field-name="unitAmount"/>
        <order-by field-name="-unitAmount"/>  <!-- Prefix - for DESC -->
        <limit count="100" offset="0"/>
    </entity-find>

    <entity-find-count entity-name="mantle.order.OrderItem" count-field="itemCount">
        <econdition field-name="orderId"/>
    </entity-find-count>

    <!-- Service calls -->
    <service-call name="mantle.order.OrderServices.place#Order"
        in-map="[orderId: orderId]" out-map="placeResult"/>
    <service-call name="mantle.order.OrderServices.place#Order"
        in-map="context" out-map="context" transaction="force-new"/>

    <!-- Control flow -->
    <if condition="orderHeader.statusId == 'OrderApproved'">
        <then><!-- actions --></then>
        <else-if condition="orderHeader.statusId == 'OrderPlaced'"><!-- actions --></else-if>
        <else><!-- actions --></else>
    </if>

    <iterate list="itemList" entry="item">
        <set field="item.statusId" value="ItemApproved"/>
        <entity-update value-field="item"/>
    </iterate>

    <!-- Return messages -->
    <return message="Order ${orderId} processed successfully"/>
    <return error="true" message="Cannot process order in status ${orderHeader.statusId}"/>
    <if condition="ec.message.hasError()"><return/></if>

    <!-- Script (Groovy) -->
    <script>
        import java.math.RoundingMode
        def total = items.sum { it.unitAmount * it.quantity }
        total = total.setScale(2, RoundingMode.HALF_UP)
    </script>

    <!-- Logging -->
    <log level="info" message="Processing order ${orderId} with ${itemList.size()} items"/>
</actions>
```

### Service ECA Rules (*.secas.xml)

```xml
<secas xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/service-eca-definition-3.xsd">
    <seca service="mantle.order.OrderServices.place#Order" when="post-service" run-on-error="false">
        <condition><expression>orderId</expression></condition>
        <actions>
            <service-call name="mycomp.myapp.MyServices.after#OrderPlaced"
                in-map="[orderId: orderId]"/>
        </actions>
    </seca>
</secas>
```

### REST API

REST APIs are defined via `*.rest.xml` resource files that map HTTP methods to services.

**CRITICAL**: The `.rest.xml` file **must be placed directly in the `service/` directory** — NOT in a subdirectory. The framework only scans direct children of `service/` for `*.rest.xml` files.

```
✅ Correct:  service/mycomp.rest.xml
❌ Wrong:    service/mycomp/mycomp.rest.xml    ← will NOT be discovered
```

REST endpoints follow: `{method} /rest/s1/{resource-name}/{sub-resources}`

Example:
```
GET    /rest/s1/mycomp/orders              → find#Orders
POST   /rest/s1/mycomp/orders              → create#Order
GET    /rest/s1/mycomp/orders/{orderId}    → get#Order
PATCH  /rest/s1/mycomp/orders/{orderId}    → update#Order
```

See `references/SERVICES.md` → REST API Configuration for full details.

---

## Screen Development (SimpleScreens patterns)

### XML Screen Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<screen xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/xml-screen-3.xsd"
    default-menu-title="My Entities" default-menu-index="5">

    <!-- Pre-actions: run before all screens in path -->
    <pre-actions><set field="html_title" value="My Entities"/></pre-actions>

    <!-- Transition: handles form submissions and navigation -->
    <transition name="createMyEntity">
        <service-call name="mycomp.myapp.MyServices.create#MyEntity"/>
        <default-response url="."/>
    </transition>
    <transition name="editMyEntity">
        <default-response url="../EditMyEntity"/>
    </transition>

    <!-- Data preparation -->
    <actions>
        <entity-find entity-name="mycomp.myapp.MyEntity" list="myEntityList">
            <search-form-inputs default-order-by="-lastUpdatedStamp"/>
        </entity-find>
    </actions>

    <widgets>
        <!-- Search/filter form -->
        <form-list name="MyEntityList" list="myEntityList" transition="editMyEntity"
                header-dialog="true" select-columns="true" saved-finds="true">
            <row-actions>
                <entity-find-one entity-name="mantle.party.PartyDetail" value-field="partyDetail">
                    <field-map field-name="partyId"/></entity-find-one>
            </row-actions>

            <field name="myEntityId"><header-field show-order-by="true"/>
                <default-field><link url="editMyEntity" text="${myEntityId}" link-type="anchor"/></default-field></field>
            <field name="description"><header-field show-order-by="true"/>
                <default-field><display/></default-field></field>
            <field name="statusId"><header-field title="Status" show-order-by="true">
                    <widget-template-include><set field="efqEnumTypeId" value="MyEntityStatus"/></widget-template-include>
                </header-field>
                <default-field><display-entity entity-name="moqui.basic.StatusItem"/></default-field></field>
            <field name="partyId"><header-field title="Party"/>
                <default-field><display text="${partyDetail?.organizationName ?: ''} ${partyDetail?.firstName ?: ''} ${partyDetail?.lastName ?: ''}"/></default-field></field>
            <field name="submitButton"><header-field title="Find"><submit/></header-field></field>
        </form-list>

        <!-- Create dialog -->
        <container-dialog id="CreateMyEntityDialog" button-text="New My Entity">
            <form-single name="CreateMyEntity" transition="createMyEntity">
                <field name="description"><default-field><text-line size="60"/></default-field></field>
                <field name="partyId"><default-field title="Party">
                    <drop-down allow-empty="true">
                        <dynamic-options transition="getPartyList" server-search="true" min-length="2"/>
                    </drop-down></default-field></field>
                <field name="submitButton"><default-field title="Create"><submit/></default-field></field>
            </form-single>
        </container-dialog>
    </widgets>
</screen>
```

### Screen Hierarchy & Subscreens

Screens form a hierarchy. The directory structure mirrors the URL path:

```
screen/
└── MyApp.xml                    → /myapp
    └── MyApp/
        ├── FindMyEntity.xml     → /myapp/FindMyEntity
        ├── EditMyEntity.xml     → /myapp/EditMyEntity
        └── dashboard.xml        → /myapp/dashboard
```

Mount in MoquiConf.xml (two options):

```xml
<!-- Option A: Standalone app on home App Listing screen (like MarbleERP, System, Tools) -->
<screen-facade>
    <screen location="component://webroot/screen/webroot/apps.xml">
        <subscreens-item name="myapp" location="component://MyComponent/screen/MyApp.xml"
            menu-title="My App" menu-index="20"/>
    </screen>
</screen-facade>

<!-- Option B: Tab inside MarbleERP -->
<screen-facade>
    <screen location="component://MarbleERP/screen/marble.xml">
        <subscreens-item name="MyModule" location="component://MyComponent/screen/marble/MyModule.xml"
            menu-title="My Module" menu-index="15"/>
    </screen>
</screen-facade>
```

> **Note**: Option A requires artifact authorization seed data. See [Artifact Authorization](#artifact-authorization-required-for-app-visibility).

### Common Screen Widgets

```xml
<!-- Display widgets -->
<display/>                                    <!-- Read-only text -->
<display-entity entity-name="..." text="..."/> <!-- Display from related entity -->
<link url="..." text="..." link-type="anchor"/> <!-- Hyperlink -->

<!-- Input widgets -->
<text-line size="30"/>                        <!-- Text input -->
<text-area rows="5" cols="60"/>               <!-- Textarea -->
<text-find size="25" default-operator="begins"/> <!-- Search input -->
<drop-down allow-empty="true">                <!-- Select dropdown -->
    <entity-options text="${description}" key="${enumId}">
        <entity-find entity-name="moqui.basic.Enumeration">
            <econdition field-name="enumTypeId" value="MyType"/>
            <order-by field-name="description"/>
        </entity-find>
    </entity-options>
</drop-down>
<date-time type="date-time"/>                 <!-- Date/time picker -->
<date-find type="date"/>                      <!-- Date range filter -->
<check><entity-options.../></check>           <!-- Checkboxes -->
<radio><entity-options.../></radio>           <!-- Radio buttons -->

<!-- Layout widgets -->
<container>...</container>                    <!-- Div wrapper -->
<container-row><row-col lg="6">...</row-col></container-row>  <!-- Bootstrap grid -->
<container-panel id="...">                    <!-- Panel layout -->
    <panel-header>...</panel-header>
    <panel-left size="3">...</panel-left>
    <panel-center>...</panel-center>
    <panel-right size="3">...</panel-right>
    <panel-footer>...</panel-footer>
</container-panel>
<container-dialog id="..." button-text="..."> <!-- Modal dialog -->
    ...
</container-dialog>

<!-- Conditional rendering -->
<section name="MySection">
    <condition><expression>someCondition</expression></condition>
    <widgets><!-- shown when true --></widgets>
    <fail-widgets><!-- shown when false --></fail-widgets>
</section>
<section-iterate name="..." list="myList" entry="item">
    <widgets>...</widgets>
</section-iterate>
```

---

## Mantle UDM Key Entity Packages

### Core Packages

| Package | Key Entities | Purpose |
|---------|-------------|---------|
| `mantle.party` | Party, Person, Organization, PartyRole, PartyRelationship, RoleType | People and organizations |
| `mantle.party.contact` | ContactMech, PostalAddress, TelecomNumber, PartyContactMech | Contact info |
| `mantle.party.agreement` | Agreement, AgreementItem, AgreementTerm | Agreements |
| `mantle.product` | Product, ProductAssoc, ProductCategory, ProductFeature, ProductPrice, ProductStore | Products and catalog |
| `mantle.product.asset` | Asset, AssetDetail, AssetReservation | Inventory |
| `mantle.product.issuance` | AssetIssuance, AssetReservation | Inventory reservations/issuance |
| `mantle.product.receipt` | AssetReceipt | Inventory receiving |
| `mantle.order` | OrderHeader, OrderPart, OrderItem, OrderItemBilling | Orders |
| `mantle.order.return` | ReturnHeader, ReturnItem | Returns |
| `mantle.shipment` | Shipment, ShipmentItem, ShipmentPackage, ShipmentRouteSegment | Shipping |
| `mantle.account.invoice` | Invoice, InvoiceItem, SettlementTerm | Invoicing |
| `mantle.account.payment` | Payment, PaymentApplication | Payments |
| `mantle.account.method` | PaymentMethod, CreditCard, BankAccount | Payment methods |
| `mantle.account.financial` | FinancialAccount, FinancialAccountTrans | Financial accounts |
| `mantle.ledger` | AcctgTrans, AcctgTransEntry, GlAccount | General ledger |
| `mantle.facility` | Facility, FacilityLocation, FacilityContactMech | Warehouses/locations |
| `mantle.work.effort` | WorkEffort, WorkEffortParty, TimeEntry | Projects, tasks, time |

### Key Status Flows

**Order**: OrderOpen → OrderProposed → OrderPlaced → OrderApproved → OrderSent → OrderCompleted / OrderCancelled / OrderHeld

**Invoice**: InvoiceIncoming/InvoiceInProcess → InvoiceFinalized → InvoiceApproved → InvoiceSent → InvoicePmtRecvd (receivable) or InvoicePaid (payable) / InvoiceCancelled / InvoiceWriteOff

**Shipment**: ShipInput → ShipScheduled → ShipPicked → ShipPacked → ShipShipped → ShipDelivered / ShipCancelled

**Payment**: PmntProposed → PmntPromised → PmntAuthorized → PmntConfirmed → PmntDelivered / PmntDeclined / PmntCancelled / PmntRefunded

**Asset**: AstAvailable, AstOnHold, AstIncoming, AstInStorage

### Order Processing Flow

1. **Create Order** — `OrderServices.create#Order` → OrderOpen status
2. **Add Items** — `OrderServices.create#OrderItem` with productId, quantity, unitAmount
3. **Set Payment** — `OrderServices.add#OrderPartPayment`
4. **Set Shipping** — Update OrderPart with shipmentMethodEnumId, postalContactMechId
5. **Place Order** — `OrderServices.place#Order` → OrderPlaced status (triggers inventory reservation)
6. **Approve** — `OrderServices.approve#Order` → OrderApproved (may be automatic)
7. **Ship** — Create Shipment, pack items, ship → triggers Invoice creation
8. **Invoice** — Auto-created on shipment pack, or manually via `InvoiceServices`
9. **Payment** — Process payment, apply to invoice

---

## Data Loading

### Data File Format

```xml
<entity-facade-xml type="seed">
    <!-- type: seed (always loaded), seed-initial (first load only),
         install (initial setup), demo (demo/test data) -->
    <moqui.basic.Enumeration enumId="MyEnumId" enumTypeId="MyEnumType" description="My Value"/>
    <mantle.party.RoleType roleTypeId="MyRole" description="My Custom Role"/>
    <mycomp.myapp.MyEntity myEntityId="TEST1" description="Test Record" statusId="MyActive"/>
</entity-facade-xml>
```

Load command: `java -jar moqui.war load types=seed,seed-initial,install`

---

## Configuration (moqui-conf.xml)

### Key Configuration Sections

```xml
<moqui-conf xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/moqui-conf-3.xsd">

    <!-- Database configuration -->
    <entity-facade>
        <datasource group-name="transactional" database-conf-name="postgres" schema-name="public">
            <inline-jdbc jdbc-uri="jdbc:postgresql://localhost:5432/moqui"
                jdbc-username="moqui" jdbc-password="moqui" pool-maxsize="50"/>
        </datasource>
    </entity-facade>

    <!-- Component loading -->
    <component-list>
        <component name="mantle-udm" location="component/mantle-udm"/>
        <component name="mantle-usl" location="component/mantle-usl"/>
        <component name="SimpleScreens" location="component/SimpleScreens"/>
        <component name="MyComponent" location="component/MyComponent"/>
    </component-list>

    <!-- Screen mounting: Standalone app on home App Listing screen -->
    <screen-facade>
        <screen location="component://webroot/screen/webroot/apps.xml">
            <subscreens-item name="myapp" location="component://MyComponent/screen/MyApp.xml"
                menu-title="My Application" menu-index="10"/>
        </screen>
    </screen-facade>

    <!-- Screen mounting: Tab inside MarbleERP (not on home screen) -->
    <screen-facade>
        <screen location="component://MarbleERP/screen/marble.xml">
            <subscreens-item name="MyModule"
                location="component://MyComponent/screen/marble/MyModule.xml"
                menu-title="My Module" menu-index="15"/>
        </screen>
    </screen-facade>

    <!-- Webapp configuration -->
    <webapp-list>
        <webapp name="webroot">
            <root-screen host=".*" location="component://webroot/screen/webroot.xml"/>
        </webapp>
    </webapp-list>
</moqui-conf>
```

> **Critical**: Mounting under `apps.xml` makes your app appear on the home App Listing screen (alongside MarbleERP, System, Tools). Mounting under `marble.xml` adds it as a tab inside MarbleERP. Choose based on whether your app is an independent application or a MarbleERP module extension.

### Artifact Authorization (Required for App Visibility)

**Every app mounted on the home screen MUST have artifact authorization seed data**, or it will be invisible. The home screen (`AppList.xml`) checks `isPermitted()` for each app — without `ArtifactGroup` and `ArtifactAuthz` records, the framework denies access and hides the app tile.

Create a seed data file in `data/`:

```xml
<!-- YourComponent/data/YourComponentSetupData.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<entity-facade-xml type="seed">
    <!-- Screen access: grants access to root screen and all subscreens (inheritAuthz=Y) -->
    <artifactGroups artifactGroupId="MY_APP" description="My App (via root screen)">
        <artifacts artifactName="component://MyComponent/screen/MyApp.xml"
            artifactTypeEnumId="AT_XML_SCREEN" inheritAuthz="Y"/>
        <!-- Grant access to ADMIN user group (required at minimum) -->
        <authz artifactAuthzId="MY_APP_ADMIN" userGroupId="ADMIN"
            authzTypeEnumId="AUTHZT_ALWAYS" authzActionEnumId="AUTHZA_ALL"/>
    </artifactGroups>

    <!-- REST API access (if your component exposes REST endpoints) -->
    <artifactGroups artifactGroupId="MY_API" description="My App REST API">
        <artifacts artifactTypeEnumId="AT_REST_PATH" inheritAuthz="Y" artifactName="/myapp"/>
        <authz artifactAuthzId="MY_API_ADMIN" userGroupId="ADMIN"
            authzTypeEnumId="AUTHZT_ALWAYS" authzActionEnumId="AUTHZA_ALL"/>
    </artifactGroups>
</entity-facade-xml>
```

After creating this file, reload data: `java -jar moqui.war load types=seed`

**Key rules for artifact authorization:**
- `artifactGroupId` must be globally unique (use a descriptive prefix)
- `artifactAuthzId` must be globally unique
- `inheritAuthz="Y"` on artifacts means all subscreens/sub-paths inherit authorization
- `userGroupId="ADMIN"` grants access to the admin group — add more `<authz>` elements for other user groups
- Data type MUST be `seed` (loaded every time) — not `seed-initial` or `demo`

---

## Common Development Patterns

### 1. Creating a New Component

```bash
mkdir -p runtime/component/MyComponent/{entity,service,screen,data,lib,script}
```

Create `component.xml`:
```xml
<component name="MyComponent" version="1.0.0">
    <depends-on name="mantle-udm"/>
    <depends-on name="mantle-usl"/>
</component>
```

### 2. Extending Mantle Entities

Create a **dedicated file per domain** for extended entities (e.g., `OrderExtendedEntities.xml`). Group ALL related `extend-entity` definitions together:

```xml
<!-- File: entity/OrderExtendedEntities.xml -->
<!-- Contains ALL order-related extend-entity definitions in one file -->
<entities xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/entity-definition-3.xsd">

    <extend-entity entity-name="OrderHeader" package="mantle.order">
        <field name="myCustomField" type="text-medium"/>
        <field name="myEnumId" type="id"/>
        <relationship type="one" title="MyCustomType" related="moqui.basic.Enumeration">
            <key-map field-name="myEnumId"/></relationship>
    </extend-entity>

    <extend-entity entity-name="OrderItem" package="mantle.order">
        <field name="taskId" type="id"/>
        <relationship type="one" related="mycomp.project.ProjectTask">
            <key-map field-name="taskId"/></relationship>
    </extend-entity>
</entities>
```

Similarly, create **view-entity files per domain** (e.g., `OrderViewEntities.xml`):

```xml
<!-- File: entity/OrderViewEntities.xml -->
<!-- Contains ALL order-related view-entity definitions in one file -->
<entities xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/entity-definition-3.xsd">

    <view-entity entity-name="OrderItemDetail" package="mycomp.order">
        <member-entity entity-alias="OI" entity-name="mantle.order.OrderItem"/>
        <member-entity entity-alias="OH" entity-name="mantle.order.OrderHeader" join-from-alias="OI">
            <key-map field-name="orderId"/></member-entity>
        <alias entity-alias="OI" name="orderId"/>
        <alias entity-alias="OI" name="orderItemSeqId"/>
        <alias entity-alias="OH" name="placedDate"/>
    </view-entity>
</entities>
```

### 3. Overriding/Extending Mantle Services (SECA)

Don't modify mantle-usl; use SECA rules to hook into existing services:

```xml
<!-- service/mycomp/MyOrderSeca.secas.xml -->
<secas>
    <seca service="mantle.order.OrderServices.place#Order" when="post-service">
        <actions>
            <service-call name="mycomp.MyOrderServices.after#PlaceOrder" in-map="context"/>
        </actions>
    </seca>
</secas>
```

### 4. ProductStore Configuration

ProductStore is central to e-commerce configuration:
```xml
<mantle.product.store.ProductStore productStoreId="MYSTORE"
    storeName="My Store" organizationPartyId="ORG_MYCOMP"
    defaultCurrencyUomId="USD" defaultLocale="en"
    requireCustomerRole="true" requireInventory="true"/>
```

### 5. Registering a Standalone App on the Home Screen

To create an independent app that appears on the home App Listing screen (alongside MarbleERP, System, Tools), you need **three things**: (1) a root app screen, (2) MoquiConf.xml mounting under `apps.xml`, and (3) artifact authorization seed data.

**Step 1: Root App Screen** (`screen/MyApp.xml`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<screen xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/xml-screen-3.xsd"
    default-menu-title="My Application" menu-image="fa fa-cogs" menu-image-type="icon"
    server-static="vuet,qvt">
    <always-actions>
        <service-call name="mantle.party.PartyServices.setup#UserOrganizationInfo" out-map="context"/>
    </always-actions>
    <subscreens default-item="FindMyEntity" always-use-full-path="true"/>
    <widgets>
        <subscreens-panel id="MyAppPanel" type="popup" title="My Application"/>
    </widgets>
</screen>
```

Key attributes:
- `server-static="vuet,qvt"` — Required for Quasar/Vue UI compatibility
- `subscreens-panel type="popup"` — Creates left-drawer navigation menu (standard for standalone apps)
- `always-use-full-path="true"` — Ensures URL paths include the subscreen name
- `menu-image` — Icon shown on the home screen app tile
- **No `<parameter>` elements** — The root screen must NOT have required parameters (or it won't show on home screen)

**Step 2: MoquiConf.xml** — Mount under `apps.xml`:
```xml
<moqui-conf xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/moqui-conf-3.xsd">
    <screen-facade>
        <screen location="component://webroot/screen/webroot/apps.xml">
            <subscreens-item name="myapp" menu-title="My Application" menu-index="20"
                location="component://MyComponent/screen/MyApp.xml"/>
        </screen>
    </screen-facade>
</moqui-conf>
```

**Step 3: Artifact Authorization** — Create `data/MyComponentSetupData.xml` (see [Artifact Authorization](#artifact-authorization-required-for-app-visibility) section above).

Your app will be accessible at `/qapps/myapp` and appear on the home screen at `/qapps`.

### 6. REST API for External Integration

Expose services as REST endpoints:
```xml
<resource name="myapi" require-authentication="true">
    <method type="get"><service name="mycomp.MyServices.get#Data"/></method>
    <method type="post"><service name="mycomp.MyServices.create#Data"/></method>
    <id name="dataId">
        <method type="get"><service name="mycomp.MyServices.get#DataDetail"/></method>
        <method type="put"><service name="mycomp.MyServices.update#Data"/></method>
    </id>
</resource>
```

---

## MarbleERP Application

MarbleERP is the flagship Moqui ERP application, combining goods and services management for B2C and B2B businesses. It is a **screens-only** component — no custom entities or services — relying entirely on Mantle UDM/USL and SimpleScreens.

### Access URL
- **Quasar UI (default)**: `http://localhost:8080/qapps/marble`
- Dynamic Bootstrap UI: `http://localhost:8080/vapps/marble`
- Legacy Standard UI: `http://localhost:8080/apps/marble`

### Module Structure

| Module | URL Path | Business Process |
|--------|----------|-----------------|
| **Customer** | `/marble/Customer` | Order-to-Cash (quotes, sales orders, returns, customer mgmt) |
| **Supplier** | `/marble/Supplier` | Procure-to-Pay (purchase orders, vendor returns, supplier mgmt) |
| **Internal** | `/marble/Internal` | Operations (shipments, inventory, production, facilities) |
| **Accounting** | `/marble/Accounting` | Finance (invoices, payments, GL, reports, fiscal periods) |
| **Project** | `/marble/Project` | PM (projects, tasks, time tracking — from HiveMind) |
| **Configure** | `/marble/Configure` | Setup (ProductStore, facilities, products, categories, GL) |

### Extending MarbleERP

**Always use your own component** — never modify MarbleERP source:

```xml
<!-- YourComponent/MoquiConf.xml — Add a new module tab -->
<screen-facade>
    <screen location="component://MarbleERP/screen/marble.xml">
        <subscreens-item name="MyModule"
            location="component://YourComponent/screen/marble/MyModule.xml"
            menu-title="My Module" menu-index="15"/>
    </screen>
</screen-facade>

<!-- Add screen under existing module -->
<screen-facade>
    <screen location="component://MarbleERP/screen/marble/Customer.xml">
        <subscreens-item name="MyScreen"
            location="component://YourComponent/screen/marble/Customer/MyScreen.xml"
            menu-title="My Screen" menu-index="20"/>
    </screen>
</screen-facade>
```

> For full MarbleERP module details, screen hierarchy, SimpleScreens reuse patterns, and customization recipes, read `references/MARBLE_ERP.md`.

---

## Critical Rules & Best Practices

1. **Follow the XSD schemas** — All Moqui XML MUST conform to the relevant XSD in `framework/xsd/`. Use only valid elements, attributes, and nesting defined in the schema. See [XSD Schema Compliance](#xsd-schema-compliance) for the full mapping.
2. **Never modify framework or Mantle source** — Use extend-entity, SECA/EECA, and custom components
3. **Use service-call for all business logic** — Don't put logic directly in screens
4. **Follow Mantle naming conventions** — verb#Noun for services, package.Entity for entities
5. **Status changes should go through services** — Never directly update statusId fields
6. **Use entity relationships** — Define relationships for proper FK management and navigation
7. **Seed data for types/statuses** — Use `<seed-data>` in entity files or data/ directory
8. **Transaction boundaries** — Services have automatic transaction handling; use `transaction="force-new"` sparingly
9. **Use `in-map="context"`** — Pass context automatically to sub-services instead of building maps manually
10. **Use `auto-parameters`** — Let service definitions auto-discover parameters from entity definitions
11. **Prefer entity-find with search-form-inputs** — For list screens, this auto-wires filter forms to queries
12. **Use `store#Entity`** — Instead of separate create/update logic, `store#` creates if PK missing, updates if exists
13. **Cache wisely** — Set `cache="true"` on entity-find-one for configuration/type entities, not transactional data
14. **Entity files are flat** — Place all entity XML files directly in `entity/` with NO package subfolders; the package is defined via the XML `package` attribute
15. **Service files use subdirectories** — Service XML files go in subdirectories that map to the package name (e.g., `service/mycomp/myapp/MyServices.xml`)
16. **Separate files for extend-entity** — Put ALL related extended entities in a single domain-specific file (e.g., `OrderExtendedEntities.xml` for order-related extensions, `PartyExtendedEntities.xml` for party-related extensions). Do NOT mix `extend-entity` with new `entity` definitions.
17. **Separate files for view-entity** — Put ALL related view entities in a single domain-specific file (e.g., `OrderViewEntities.xml`, `StudentViewEntities.xml`). Do NOT mix `view-entity` with base `entity` definitions (exception: small components with 1-2 tightly coupled view entities for their own custom entities).
18. **Always create artifact authorization for new apps** — Any app mounted on the home screen needs `ArtifactGroup` + `ArtifactAuthz` seed data or it will be invisible
19. **Root app screens must have no required parameters** — The home screen checks `isPermitted()` and skips screens with required parameters
