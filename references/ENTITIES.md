# Moqui Entity Development Reference

> **Schema**: Entity XML must conform to `framework/xsd/entity-definition-3.xsd`. Entity ECA rules must conform to `framework/xsd/entity-eca-3.xsd`. XML Actions embedded within ECA rules must conform to `framework/xsd/xml-actions-3.xsd`.

## Table of Contents
1. [Entity File Placement](#entity-file-placement)
2. [Entity Definition Elements](#entity-definition-elements)
3. [Field Types & Attributes](#field-types--attributes)
4. [Relationships](#relationships)
5. [View Entities](#view-entities)
6. [Extend Entity](#extend-entity)
7. [Entity ECA Rules](#entity-eca-rules)
8. [Entity Find Operations (XML Actions)](#entity-find-operations)
9. [Entity Facade Groovy API](#entity-facade-groovy-api)
10. [Mantle UDM Entity Map](#mantle-udm-entity-map)
11. [Seed & Demo Data](#seed--demo-data)

---

## Entity File Placement

### Directory Convention

Entity XML files go **directly in the `entity/` directory** of your component — **NO package subfolders**.

```
✅ CORRECT — Flat structure (follows mantle-udm convention):
YourComponent/
└── entity/
    ├── StudentEntities.xml              # New entities: package="studentmgmt" defined in XML
    ├── OrderExtendedEntities.xml        # All order-related extend-entity definitions
    ├── PartyExtendedEntities.xml        # All party-related extend-entity definitions
    ├── OrderViewEntities.xml            # All order-related view-entity definitions
    ├── StudentViewEntities.xml          # All student-related view-entity definitions
    └── StudentEntities.eecas.xml        # Entity ECA rules

❌ WRONG — Do NOT create package subfolders:
YourComponent/
└── entity/
    └── mycomp/
        └── myapp/
            └── YourEntities.xml   # This is incorrect!
```

The entity **package** is defined as an XML attribute on the `<entity>` or `<extend-entity>` element, NOT by the directory structure:

```xml
<!-- File: entity/StudentEntities.xml -->
<entities>
    <entity entity-name="Student" package="studentmgmt">  <!-- package defined HERE -->
        <field name="studentId" type="id" is-pk="true"/>
        ...
    </entity>
</entities>
```

### Why Flat?

This follows the **mantle-udm convention**. In mantle-udm, all entity files are directly in `entity/`:
```
mantle-udm/entity/
├── AccountEntities.xml        # package="mantle.account.invoice", "mantle.account.payment", etc.
├── HumanResourcesEntities.xml # package="mantle.humanres...."
├── OrderEntities.xml          # package="mantle.order"
├── PartyEntities.xml          # package="mantle.party"
├── ProductEntities.xml        # package="mantle.product"
├── ShipmentEntities.xml       # package="mantle.shipment"
└── WorkEntities.xml           # package="mantle.work.effort"
```

### File Naming Conventions

Organize entity files by **type and domain**. Each file should group related definitions together:

| File Type | Naming Convention | Contents | Example |
|-----------|-------------------|----------|---------|
| New entities | `{Domain}Entities.xml` | All new entity definitions for a domain | `StudentEntities.xml`, `ProjectEntities.xml` |
| Extended entities | `{Domain}ExtendedEntities.xml` | All `extend-entity` definitions for a domain | `OrderExtendedEntities.xml`, `PartyExtendedEntities.xml` |
| View entities | `{Domain}ViewEntities.xml` | All `view-entity` definitions for a domain | `OrderViewEntities.xml`, `StudentViewEntities.xml` |
| Entity ECA rules | `{Domain}Entities.eecas.xml` | ECA rules for a domain | `OrderEntities.eecas.xml` |

**Rules:**
- **One file per domain per type** — Put ALL order-related extended entities in a single `OrderExtendedEntities.xml`, not scattered across multiple files.
- **Keep extend-entity definitions separate from new entity definitions** — Do NOT mix `extend-entity` and `entity` definitions in the same file. Extended entities modify existing Mantle/framework entities and should be clearly separated.
- **Keep view-entity definitions separate from base entity definitions** — Create a dedicated `{Domain}ViewEntities.xml` file for view entities. This keeps join/aggregate logic separate from table definitions.
- **Small components exception** — If your component has only 1-2 view entities or extend-entities closely tied to a single new entity, they MAY be placed in the same file as the new entity (e.g., a view-entity that only joins your own custom entities). Once you have 3+ or they span multiple domains, split them out.

**Example — component with multiple domains:**
```
MyComponent/
└── entity/
    ├── ProjectEntities.xml               # New entities: Project, ProjectTask, ProjectMember
    ├── ProjectViewEntities.xml           # View entities: ProjectTaskDetail, ProjectMemberSummary
    ├── OrderExtendedEntities.xml         # extend-entity: OrderHeader + OrderItem (added custom fields)
    ├── PartyExtendedEntities.xml         # extend-entity: Party + Person (added custom fields)
    ├── OrderViewEntities.xml             # View entities: OrderItemWithProject, OrderProjectSummary
    └── ProjectEntities.eecas.xml         # ECA rules for project entities
```

**Example — `OrderExtendedEntities.xml` (all order-related extensions in one file):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<entities xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/entity-definition-3.xsd">

    <extend-entity entity-name="OrderHeader" package="mantle.order">
        <field name="projectId" type="id"/>
        <field name="customChannel" type="text-short"/>
        <relationship type="one" related="mycomp.project.Project">
            <key-map field-name="projectId"/></relationship>
    </extend-entity>

    <extend-entity entity-name="OrderItem" package="mantle.order">
        <field name="taskId" type="id"/>
        <relationship type="one" related="mycomp.project.ProjectTask">
            <key-map field-name="taskId"/></relationship>
    </extend-entity>

</entities>
```

**Example — `OrderViewEntities.xml` (all order-related view entities in one file):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<entities xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/entity-definition-3.xsd">

    <view-entity entity-name="OrderItemWithProject" package="mycomp.order">
        <member-entity entity-alias="OI" entity-name="mantle.order.OrderItem"/>
        <member-entity entity-alias="PRJ" entity-name="mycomp.project.Project"
                join-from-alias="OI" join-optional="true">
            <key-map field-name="projectId"/></member-entity>
        <alias entity-alias="OI" name="orderId"/>
        <alias entity-alias="OI" name="orderItemSeqId"/>
        <alias entity-alias="PRJ" name="projectName"/>
    </view-entity>

    <view-entity entity-name="OrderProjectSummary" package="mycomp.order">
        <!-- ... another order-related view entity ... -->
    </view-entity>

</entities>
```

### Service Files ARE Different

Unlike entities, **service files DO use subdirectories** that map to the Java package name:
```
mantle-usl/service/
├── mantle/
│   ├── order/
│   │   ├── OrderServices.xml     → mantle.order.OrderServices.verb#Noun
│   │   └── OrderInfoServices.xml → mantle.order.OrderInfoServices.verb#Noun
│   ├── product/
│   │   └── ProductServices.xml   → mantle.product.ProductServices.verb#Noun
│   └── party/
│       └── PartyServices.xml     → mantle.party.PartyServices.verb#Noun
```

---

## Entity Definition Elements

### Full Entity Template

```xml
<entity entity-name="MyEntity" package="mycomp.myapp"
        short-alias="myEntity"
        use="transactional"
        cache="never"
        authorize-skip="view"
        sequence-bank-size="10"
        optimistic-lock="true"
        no-update-stamp="false">

    <!-- Primary key fields -->
    <field name="myEntityId" type="id" is-pk="true"/>

    <!-- Regular fields -->
    <field name="myField" type="text-medium" not-null="true"
           default="defaultValue" encrypt="false" enable-audit-log="true">
        <description>Field description for documentation</description>
    </field>

    <!-- Relationships -->
    <relationship type="one" related="other.package.OtherEntity" short-alias="other"/>
    <relationship type="many" related="mycomp.myapp.MyEntityItem" short-alias="items">
        <key-map field-name="myEntityId"/></relationship>

    <!-- Indexes -->
    <index name="MY_IDX_STATUS">
        <index-field name="statusId"/>
    </index>
    <index name="MY_IDX_UNIQUE" unique="true">
        <index-field name="externalId"/>
    </index>

    <!-- Master definition (for auto-generated screens and data documents) -->
    <master>
        <detail relationship="items"/>
        <detail relationship="other"/>
    </master>

    <!-- Inline seed data -->
    <seed-data>
        <moqui.basic.StatusType statusTypeId="MyEntity" description="My Entity Status"/>
        <moqui.basic.StatusItem statusId="MeActive" statusTypeId="MyEntity" description="Active" sequenceNum="1"/>
        <moqui.basic.StatusItem statusId="MeClosed" statusTypeId="MyEntity" description="Closed" sequenceNum="99"/>
        <moqui.basic.StatusFlowTransition statusFlowId="Default"
            statusId="MeActive" toStatusId="MeClosed" transitionName="Close"/>
    </seed-data>
</entity>
```

### Entity `use` Attribute Values
- `transactional` — Default, for business transaction data
- `nontransactional` — For logging, analytics (may use separate DB)
- `configuration` — For config/setup data, typically cached
- `tenantcommon` — Shared across tenants in multi-tenant mode

### Entity `cache` Attribute Values
- `true` — Cache find-one and find-list results
- `false` / `never` — No caching (default for transactional)

---

## Field Types & Attributes

### Complete Field Type Reference

| Type | Java | SQL (Postgres) | SQL (MySQL) | SQL (H2) |
|------|------|----------------|-------------|----------|
| `id` | String | VARCHAR(40) | VARCHAR(40) | VARCHAR(40) |
| `id-long` | String | VARCHAR(255) | VARCHAR(255) | VARCHAR(255) |
| `text-indicator` | String | CHAR(1) | CHAR(1) | CHAR(1) |
| `text-short` | String | VARCHAR(63) | VARCHAR(63) | VARCHAR(63) |
| `text-medium` | String | VARCHAR(255) | VARCHAR(255) | VARCHAR(255) |
| `text-intermediate` | String | VARCHAR(511) | VARCHAR(511) | VARCHAR(511) |
| `text-long` | String | VARCHAR(4095) | VARCHAR(4095) | VARCHAR(4095) |
| `text-very-long` | String | TEXT | TEXT | CLOB |
| `binary-very-long` | byte[] | BYTEA | LONGBLOB | BLOB |
| `date-time` | Timestamp | TIMESTAMP | DATETIME(3) | TIMESTAMP |
| `time` | Time | TIME | TIME | TIME |
| `date` | Date | DATE | DATE | DATE |
| `number-integer` | Long | NUMERIC(20,0) | DECIMAL(20,0) | NUMERIC(20,0) |
| `number-decimal` | BigDecimal | NUMERIC(26,6) | DECIMAL(26,6) | NUMERIC(26,6) |
| `number-float` | Double | FLOAT8 | DOUBLE | DOUBLE |
| `currency-amount` | BigDecimal | NUMERIC(22,2) | DECIMAL(22,2) | NUMERIC(22,2) |
| `currency-precise` | BigDecimal | NUMERIC(23,3) | DECIMAL(23,3) | NUMERIC(23,3) |

### Field Attributes
- `is-pk="true"` — Part of primary key
- `not-null="true"` — NOT NULL constraint
- `default="value"` — Default value (literal)
- `default="expression"` — Default from Groovy expression
- `encrypt="true"` — Encrypt at rest
- `enable-audit-log="true"` — Track changes in AuditLog entity
- `enable-audit-log="update"` — Track only updates

---

## Relationships

### Relationship Types
```xml
<!-- one: FK from this entity to related (this entity has the FK field) -->
<relationship type="one" related="mantle.party.Party" short-alias="party">
    <key-map field-name="partyId"/></relationship>

<!-- one: with title prefix (when multiple FKs to same entity) -->
<relationship type="one" title="From" related="mantle.party.Party" short-alias="fromParty">
    <key-map field-name="fromPartyId" related="partyId"/></relationship>
<relationship type="one" title="To" related="mantle.party.Party" short-alias="toParty">
    <key-map field-name="toPartyId" related="partyId"/></relationship>

<!-- one-nofk: logical relationship without FK constraint -->
<relationship type="one-nofk" related="mantle.party.Person" short-alias="person">
    <key-map field-name="partyId"/></relationship>

<!-- many: reverse relationship (related entity has FK to this) -->
<relationship type="many" related="mantle.order.OrderItem" short-alias="items">
    <key-map field-name="orderId"/></relationship>

<!-- many-nofk: many without FK constraint -->
<relationship type="many-nofk" related="..." short-alias="..."/>
```

### Key-Map Rules
- If field names match between entities, `key-map` is optional for `type="one"`
- For different field names: `<key-map field-name="localField" related="relatedField"/>`
- Multiple key-maps for compound keys

### Accessing Relationships in Groovy
```groovy
def order = ec.entity.find("mantle.order.OrderHeader").condition("orderId", orderId).one()
def items = order.items  // Uses short-alias "items"
def customer = order.part?.customerParty  // Navigate through relationships
```

---

## View Entities

> **File convention**: Put ALL related view entities in a single domain-specific file named `{Domain}ViewEntities.xml` (e.g., `OrderViewEntities.xml` for all order-related view entities). See [File Naming Conventions](#file-naming-conventions) above.

### Join Patterns

```xml
<!-- File: entity/OrderViewEntities.xml -->
<view-entity entity-name="OrderItemAndProduct" package="mycomp.myapp">
    <!-- Primary member entity -->
    <member-entity entity-alias="OI" entity-name="mantle.order.OrderItem"/>

    <!-- Inner join (default) -->
    <member-entity entity-alias="OH" entity-name="mantle.order.OrderHeader" join-from-alias="OI">
        <key-map field-name="orderId"/></member-entity>

    <!-- Left outer join -->
    <member-entity entity-alias="PROD" entity-name="mantle.product.Product"
            join-from-alias="OI" join-optional="true">
        <key-map field-name="productId"/></member-entity>

    <!-- Join through another alias -->
    <member-entity entity-alias="OP" entity-name="mantle.order.OrderPart"
            join-from-alias="OI">
        <key-map field-name="orderId"/>
        <key-map field-name="orderPartSeqId"/>
    </member-entity>

    <!-- Aliases (select fields) -->
    <alias entity-alias="OI" name="orderId"/>
    <alias entity-alias="OI" name="orderItemSeqId"/>
    <alias entity-alias="OI" name="productId"/>
    <alias entity-alias="OI" name="quantity"/>
    <alias entity-alias="OI" name="unitAmount"/>
    <alias entity-alias="OH" name="placedDate"/>
    <alias entity-alias="OH" name="statusId" field="statusId"/>
    <alias entity-alias="PROD" name="productName"/>
    <alias entity-alias="OP" name="customerPartyId"/>

    <!-- Calculated/aggregate alias -->
    <alias entity-alias="OI" name="itemTotal" function="sum">
        <complex-alias operator="*">
            <complex-alias-field entity-alias="OI" field="quantity"/>
            <complex-alias-field entity-alias="OI" field="unitAmount"/>
        </complex-alias>
    </alias>

    <!-- Entity condition (permanent WHERE clause) -->
    <entity-condition>
        <econdition entity-alias="OH" field-name="statusId" operator="not-equals" value="OrderCancelled"/>
    </entity-condition>
</view-entity>
```

### Aggregate Functions
Available for alias `function` attribute: `min`, `max`, `sum`, `avg`, `count`, `count-distinct`, `upper`, `lower`

---

## Extend Entity

Add fields, relationships, or seed data to existing entities without modifying the original.

> **File convention**: Put ALL related extended entities in a single domain-specific file named `{Domain}ExtendedEntities.xml` (e.g., `OrderExtendedEntities.xml` for all order-related extensions). See [File Naming Conventions](#file-naming-conventions) above.

```xml
<!-- File: entity/OrderExtendedEntities.xml -->
<!-- All order-related extend-entity definitions in one file -->
<entities xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/entity-definition-3.xsd">

    <!-- Extend Mantle's OrderHeader -->
    <extend-entity entity-name="OrderHeader" package="mantle.order">
        <field name="myCustomField" type="text-medium"/>
        <field name="myTypeEnumId" type="id"/>
        <relationship type="one" title="MyType" related="moqui.basic.Enumeration">
            <key-map field-name="myTypeEnumId"/></relationship>
        <index name="OH_MY_CUSTOM"><index-field name="myCustomField"/></index>
        <seed-data>
            <moqui.basic.EnumerationType enumTypeId="MyOrderType" description="My Order Type"/>
            <moqui.basic.Enumeration enumId="MotStandard" enumTypeId="MyOrderType" description="Standard"/>
        </seed-data>
    </extend-entity>

    <!-- Extend Mantle's OrderItem (same file since it's order domain) -->
    <extend-entity entity-name="OrderItem" package="mantle.order">
        <field name="customNotes" type="text-long"/>
    </extend-entity>

</entities>
```

```xml
<!-- File: entity/SecurityExtendedEntities.xml -->
<!-- Framework entity extensions go in their own domain file -->
<entities xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/entity-definition-3.xsd">

    <!-- Extend framework's UserAccount -->
    <extend-entity entity-name="UserAccount" package="moqui.security">
        <field name="myPreference" type="text-medium"/>
    </extend-entity>

</entities>
```

---

## Entity ECA Rules

### EECA File (*.eecas.xml)

```xml
<eecas xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/eeca-definition-3.xsd">

    <!-- Trigger on status change -->
    <eeca entity="mantle.order.OrderHeader" on-create="false" on-update="true"
            on-delete="false" on-find-one="false" on-find-list="false"
            run-on-error="false" get-entire-entity="true" get-original-value="true">
        <condition>
            <expression>statusId == 'OrderApproved' &amp;&amp; statusId_old == 'OrderPlaced'</expression>
        </condition>
        <actions>
            <service-call name="mycomp.MyServices.on#OrderApproved"
                in-map="[orderId: orderId, oldStatusId: statusId_old]"/>
        </actions>
    </eeca>

    <!-- Trigger on create -->
    <eeca entity="mantle.product.asset.Asset" on-create="true">
        <condition><expression>assetTypeEnumId == 'AstTpInventory'</expression></condition>
        <actions>
            <service-call name="mycomp.MyServices.on#InventoryReceived"
                in-map="[assetId: assetId, productId: productId, quantity: quantityOnHandTotal]"/>
        </actions>
    </eeca>
</eecas>
```

### EECA Available Variables
- All entity fields are in context
- `fieldName_old` — Previous value (on update, when `get-original-value="true"`)
- `entityValue` — The EntityValue object itself (when `get-entire-entity="true"`)

---

## Entity Find Operations

### XML Actions Entity Operations

```xml
<!-- Find one by PK (auto from context) -->
<entity-find-one entity-name="mantle.order.OrderHeader" value-field="orderHeader"/>

<!-- Find one with explicit PK -->
<entity-find-one entity-name="mantle.order.OrderHeader" value-field="orderHeader">
    <field-map field-name="orderId" from="myOrderId"/>
</entity-find-one>

<!-- Find one for update (SELECT FOR UPDATE) -->
<entity-find-one entity-name="mantle.order.OrderHeader" value-field="orderHeader" for-update="true"/>

<!-- Find list with conditions -->
<entity-find entity-name="mantle.order.OrderItem" list="orderItems">
    <econdition field-name="orderId"/>
    <econdition field-name="statusId" value="ItemApproved"/>
    <econdition field-name="quantity" operator="greater" from="0"/>
    <econdition field-name="productId" operator="is-not-null"/>
    <econditions combine="or">
        <econdition field-name="itemTypeEnumId" value="ItemProduct"/>
        <econdition field-name="itemTypeEnumId" value="ItemService"/>
    </econditions>
    <having-econditions>
        <econdition field-name="totalAmount" operator="greater" from="100"/>
    </having-econditions>
    <select-field field-name="orderId"/>
    <select-field field-name="productId"/>
    <order-by field-name="orderItemSeqId"/>
    <limit count="50" offset="0"/>
</entity-find>

<!-- Find with search-form-inputs (for screen list forms) -->
<entity-find entity-name="mantle.order.OrderHeader" list="orderList">
    <search-form-inputs default-order-by="-placedDate"
        skip-fields="statusId" paginate="true"/>
    <econdition field-name="statusId" operator="in" from="filterStatusList"
        ignore-if-empty="true"/>
</entity-find>

<!-- Count -->
<entity-find-count entity-name="mantle.order.OrderItem" count-field="itemCount">
    <econdition field-name="orderId"/>
</entity-find-count>

<!-- Create -->
<service-call name="create#mycomp.myapp.MyEntity" in-map="context" out-map="context"/>
<!-- OR manual: -->
<entity-make-value entity-name="mycomp.myapp.MyEntity" value-field="newEntity" map="context"/>
<entity-sequenced-id-primary value-field="newEntity"/>
<entity-create value-field="newEntity"/>

<!-- Update -->
<entity-find-one entity-name="mycomp.myapp.MyEntity" value-field="myEntity" for-update="true"/>
<set field="myEntity.description" from="newDescription"/>
<entity-update value-field="myEntity"/>

<!-- Store (create or update) -->
<service-call name="store#mycomp.myapp.MyEntity" in-map="context" out-map="context"/>

<!-- Delete -->
<entity-delete value-field="myEntity"/>
<entity-delete-by-condition entity-name="mycomp.myapp.MyEntityItem">
    <econdition field-name="myEntityId"/>
</entity-delete-by-condition>
```

### Econdition Operators
`equals` (default), `not-equals`, `less`, `less-equals`, `greater`, `greater-equals`, `in`, `not-in`, `between`, `not-between`, `like`, `not-like`, `is-null`, `is-not-null`

---

## Entity Facade Groovy API

```groovy
// Find one
def order = ec.entity.find("mantle.order.OrderHeader")
    .condition("orderId", orderId).one()

// Find list
def items = ec.entity.find("mantle.order.OrderItem")
    .condition("orderId", orderId)
    .conditionDate("fromDate", "thruDate", ec.user.nowTimestamp)
    .orderBy("orderItemSeqId")
    .limit(100)
    .list()

// Iterator (for large result sets)
def iter = ec.entity.find("mantle.order.OrderItem")
    .condition("orderId", orderId).iterator()
try {
    while (iter.hasNext()) {
        def item = iter.next()
        // process item
    }
} finally {
    iter.close()
}

// Count
long count = ec.entity.find("mantle.order.OrderItem")
    .condition("orderId", orderId).count()

// Create
def newEntity = ec.entity.makeValue("mycomp.myapp.MyEntity")
newEntity.setAll(context)
newEntity.setSequencedIdPrimary()
newEntity.create()

// Update
order.statusId = "OrderApproved"
order.update()

// Store (create or update)
ec.entity.makeValue("mycomp.myapp.MyEntity").setAll(context).createOrUpdate()

// Delete
order.delete()
```

---

## Mantle UDM Entity Map

### Party Domain
```
Party (partyId) ← Person | Organization
├── PartyRole (partyId, roleTypeId)
├── PartyRelationship (fromPartyId, roleTypeId, toPartyId, toRoleTypeId, fromDate)
├── PartyContactMech (partyId, contactMechId, fromDate) → ContactMech
│   ├── PostalAddress (contactMechId)
│   ├── TelecomNumber (contactMechId)
│   └── ContactMech.infoString (for email)
├── PartyClassificationAppl
├── PartyContent
├── PartyNote
└── PartyIdentification
```

### Order Domain
```
OrderHeader (orderId)
├── OrderPart (orderId, orderPartSeqId) — one per vendor/ship-group
│   ├── customerPartyId, vendorPartyId
│   ├── postalContactMechId, telecomContactMechId
│   ├── shipmentMethodEnumId, carrierPartyId
│   └── facilityId
├── OrderItem (orderId, orderItemSeqId)
│   ├── productId, quantity, unitAmount
│   ├── itemTypeEnumId, statusId
│   └── orderPartSeqId
├── OrderContactMech
├── OrderNote
├── OrderTerm
└── OrderContent
```

### Product & Inventory Domain
```
Product (productId)
├── ProductAssoc (productId, toProductId, productAssocTypeEnumId, fromDate)
├── ProductCategory / ProductCategoryMember
├── ProductFeature / ProductFeatureAppl
├── ProductPrice (productId, priceTypeEnumId, ..., fromDate)
├── ProductIdentification
└── ProductStore (productStoreId) — e-commerce configuration

Asset (assetId) — Inventory
├── productId, facilityId, locationSeqId
├── quantityOnHandTotal, availableToPromiseTotal
├── AssetDetail — change log
├── AssetReservation — reserved for orders
├── AssetReceipt — receiving records
└── AssetIssuance — shipment/picking records
```

### Shipment Domain
```
Shipment (shipmentId)
├── shipmentTypeEnumId (sales, purchase, transfer, return)
├── fromPartyId, toPartyId
├── originFacilityId, destinationFacilityId
├── statusId (ShipInput → ShipScheduled → ShipPicked → ShipPacked → ShipShipped → ShipDelivered)
├── ShipmentItem (shipmentId, productId)
│   └── ShipmentItemSource (orderId/returnId linkage)
├── ShipmentPackage (shipmentId, shipmentPackageSeqId)
│   └── ShipmentPackageContent (productId, quantity in package)
└── ShipmentRouteSegment (carrier, method, tracking, addresses)
```

### Accounting Domain
```
Invoice (invoiceId)
├── invoiceTypeEnumId, fromPartyId, toPartyId
├── statusId, currencyUomId, invoiceDate, dueDate
└── InvoiceItem (invoiceId, invoiceItemSeqId)
    ├── itemTypeEnumId, quantity, amount
    └── productId, assetId

Payment (paymentId)
├── paymentTypeEnumId, fromPartyId, toPartyId
├── paymentMethodId, amount, statusId
└── PaymentApplication — links payment to invoice(s)

AcctgTrans (acctgTransId) — GL transaction
├── acctgTransTypeEnumId, organizationPartyId
└── AcctgTransEntry — debit/credit entries with glAccountId
```

---

## Seed & Demo Data

### Data Types (loaded via `type` attribute)

| Type | When Loaded | Purpose |
|------|-------------|---------|
| `seed` | Every load | Status types, enumerations, system config |
| `seed-initial` | First load only | Default org, admin user |
| `install` | On install | Initial business data |
| `demo` | Demo environments | Sample/test data |

### Data File Example

```xml
<?xml version="1.0" encoding="UTF-8"?>
<entity-facade-xml type="seed">
    <!-- Enumerations -->
    <moqui.basic.EnumerationType enumTypeId="MyCustomType" description="My Custom Type"/>
    <moqui.basic.Enumeration enumId="MctTypeA" enumTypeId="MyCustomType" description="Type A" sequenceNum="1"/>
    <moqui.basic.Enumeration enumId="MctTypeB" enumTypeId="MyCustomType" description="Type B" sequenceNum="2"/>

    <!-- Status flow -->
    <moqui.basic.StatusType statusTypeId="MyWorkflow" description="My Workflow"/>
    <moqui.basic.StatusItem statusId="MwDraft" statusTypeId="MyWorkflow" description="Draft" sequenceNum="1"/>
    <moqui.basic.StatusItem statusId="MwActive" statusTypeId="MyWorkflow" description="Active" sequenceNum="2"/>
    <moqui.basic.StatusItem statusId="MwComplete" statusTypeId="MyWorkflow" description="Complete" sequenceNum="3"/>
    <moqui.basic.StatusFlowTransition statusFlowId="Default"
        statusId="MwDraft" toStatusId="MwActive" transitionName="Activate"/>
    <moqui.basic.StatusFlowTransition statusFlowId="Default"
        statusId="MwActive" toStatusId="MwComplete" transitionName="Complete"/>
</entity-facade-xml>
```

### Foreign Key References in Demo Data

**CRITICAL**: Demo data records that reference other entities via foreign keys (e.g., `partyId` → `mantle.party.Party`) **must** use IDs that already exist in the loaded seed/demo data. Using non-existent IDs causes `JdbcSQLIntegrityConstraintViolationException` errors at load time.

Common FK fields and where to find valid values:
- **`partyId`** → Use IDs from `mantle-usl/data/ZbaOrganizationDemoData.xml` (e.g., `ORG_ZIZI_JD`, `ORG_ZIZI_BD`, `CustJqp`, `EX_JOHN_DOE`). See [TESTING.md — Available Demo Party IDs](TESTING.md#available-demo-party-ids-for-foreign-keys) for the full list.
- **`productId`** → Use IDs from `mantle-usl/data/` product demo data (e.g., `DEMO_1_1`)
- **`statusId`** → Must match a `StatusItem` defined in seed data
- **`enumId`** → Must match an `Enumeration` defined in seed data

**Never invent foreign key values** — always verify the referenced record exists in the codebase's seed/demo data files.

### Loading Data
```bash
# Load all types
java -jar moqui.war load

# Load specific types
java -jar moqui.war load types=seed,seed-initial,install,demo
```

## Verify

After adding or extending an entity:
1. `cd moqui-framework && ./gradlew load` — confirm `runtime/log/moqui.log` shows `Entity {name} loaded`.
2. Open `qapps/tools` → Entity Data — find the entity, confirm fields and seed rows.
3. If you added a relationship, query a sample row and confirm the related record resolves.

Common failures: `XSDValidationError` (entity XML malformed), `Unknown field` (field referenced in view-entity doesn't exist), `Constraint violation` (NOT NULL field has no default).

See [`../SKILL.md`](../SKILL.md) §"Verify After Each Pattern" for the cross-cutting table.

<!-- begin: related-lessons (auto-generated by scripts/regen-skill-lesson-links.sh — do not edit by hand) -->

## Related Lessons

Cross-references from [`docs/ai/lessons.md`](../../docs/ai/lessons.md) that apply to this area. Filter tags: `Moqui-Entities,Mantle-Reuse,Catalog,Registration`. Regenerate with `scripts/regen-skill-lesson-links.sh`.

- 2026-04-15 — HSTO backend directory name differs from component name  `[Process,Moqui-Entities]`
- 2026-05-28 — `OrderItem` has no direct `workEffortId` FK  `[Moqui-Entities,Mantle-Reuse,Registration]`
- 2026-05-28 — Seat capacity lives on `WorkEffortProduct.estimatedQuantity`, not on `Product`  `[Moqui-Entities,Catalog]`
- 2026-05-28 — Sessions are `WorkEffort`, not `CommunicationEvent`  `[Moqui-Entities,Catalog]`
- 2026-05-28 — Stock Mantle Geo IDs are ISO alpha-3, not alpha-2  `[Moqui-Entities]`
- 2026-05-28 — Reviewer pricing uses `OrderItem.priceTierEnumId`, not `priceTypeEnumId`  `[Registration]`
- 2026-05-28 — Guest checkout creates a provisional `Party`, not name/email columns on `OrderHeader`  `[Registration]`
- 2026-05-28 — Consent is a per-artifact row, never a boolean on `OrderHeader`  `[Registration]`
- 2026-05-28 — Polymorphic comms use stock bridges, not target-specific FKs  `[Communication,Mantle-Reuse]`

<!-- end: related-lessons -->
