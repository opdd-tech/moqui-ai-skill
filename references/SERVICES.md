# Moqui Service Development Reference

> **Schema**: Service XML must conform to `framework/xsd/service-definition-3.xsd`. Service ECA rules must conform to `framework/xsd/service-eca-3.xsd`. REST API definitions must conform to `framework/xsd/rest-api-3.xsd`. XML Actions blocks must conform to `framework/xsd/xml-actions-3.xsd`.

## Table of Contents
1. [Service Definition](#service-definition)
2. [Service Types](#service-types)
3. [Parameters](#parameters)
4. [XML Actions Complete Reference](#xml-actions-complete-reference)
5. [Service ECA Rules](#service-eca-rules)
6. [Transaction Control](#transaction-control)
7. [Mantle USL Key Services](#mantle-usl-key-services)
8. [REST API Configuration](#rest-api-configuration)
9. [Service Facade Groovy API](#service-facade-groovy-api)
10. [Error Handling](#error-handling)

---

## Service Definition

### Complete Service Template

```xml
<services xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/service-definition-3.xsd">

    <service verb="create" noun="SalesOrder" type="inline"
             authenticate="true"
             allow-remote="true"
             transaction="use-or-begin"
             transaction-timeout="600"
             no-remember-parameters="false"
             semaphore="wait" semaphore-timeout="120">
        <description>Create a new Sales Order with items</description>

        <in-parameters>
            <parameter name="customerPartyId" required="true" type="String"/>
            <parameter name="items" type="List" required="true">
                <parameter name="productId" required="true"/>
                <parameter name="quantity" type="BigDecimal" required="true"/>
                <parameter name="unitAmount" type="BigDecimal"/>
            </parameter>
            <parameter name="orderName" type="String"/>
            <auto-parameters entity-name="mantle.order.OrderHeader" include="nonpk">
                <exclude field-name="statusId"/>
                <exclude field-name="placedDate"/>
            </auto-parameters>
        </in-parameters>

        <out-parameters>
            <parameter name="orderId" required="true"/>
            <parameter name="orderPartSeqId"/>
        </out-parameters>

        <actions>
            <!-- Service logic here -->
        </actions>
    </service>
</services>
```

### Service Naming Convention

File location determines the service name prefix:
```
File: service/mycomp/myapp/OrderServices.xml
Service: mycomp.myapp.OrderServices.create#SalesOrder
         ^package^         ^file^       ^verb^ ^noun^
```

### Verb Conventions (Mantle patterns)
- `create` — Create new record
- `update` — Update existing record
- `delete` — Delete record
- `store` — Create or update (upsert)
- `get` — Retrieve data (read-only)
- `find` — Search/query data
- `place` — Place/submit (orders)
- `approve` — Approve workflow
- `cancel` — Cancel
- `complete` — Complete/close
- `send` — Send notification/email
- `receive` — Receive (shipments, data)
- `calculate` — Compute derived values
- `validate` — Validate data
- `process` — Process (payments, etc.)
- `apply` — Apply (payments to invoices)
- `handle` — Event handler
- `check` — Check condition
- `setup` — Initialize/configure

---

## Service Types

```xml
<!-- inline: XML Actions (most common) -->
<service verb="get" noun="Data" type="inline">
    <actions><!-- XML Actions --></actions>
</service>

<!-- script: Groovy or other script -->
<service verb="process" noun="Data" type="script"
    location="component://MyComp/script/ProcessData.groovy"/>

<!-- java: Java/Groovy class method -->
<service verb="process" noun="Data" type="java"
    location="mycomp.MyServiceClass" method="processData"/>

<!-- entity-auto: Auto CRUD for entities -->
<!-- These exist automatically for ALL entities, no definition needed:
     create#package.EntityName
     update#package.EntityName
     delete#package.EntityName
     store#package.EntityName
-->

<!-- interface: Define contract, no implementation -->
<service verb="receive" noun="DataFeed" type="interface">
    <in-parameters>
        <parameter name="dataFeedId" required="true"/>
        <parameter name="feedStamp" type="Timestamp"/>
        <parameter name="documentList" type="List" required="true"/>
    </in-parameters>
</service>

<!-- remote-xml-rpc / remote-json-rpc -->
<service verb="get" noun="ExternalData" type="remote-json-rpc"
    location="http://remote-host/rpc" method="getExternalData"/>
```

---

## Parameters

### Parameter Attributes

```xml
<parameter name="orderId"
    type="String"           <!-- String, Integer, Long, BigDecimal, Timestamp, List, Map, etc. -->
    required="true"         <!-- true/false -->
    default-value="literal" <!-- Literal default -->
    default="expression"    <!-- Groovy expression default -->
    format=""               <!-- Output format pattern -->
    allow-html="safe"       <!-- none (default), safe (sanitized), any -->
    entity-name=""          <!-- For validation against entity field -->
    field-name=""           <!-- Field name in entity (if different from param name) -->
/>
```

### Auto Parameters

```xml
<!-- Include all non-PK fields from entity -->
<auto-parameters entity-name="mantle.order.OrderHeader" include="nonpk"/>

<!-- Include PK fields -->
<auto-parameters entity-name="mantle.order.OrderHeader" include="pk" required="true"/>

<!-- Include all fields -->
<auto-parameters entity-name="mantle.order.OrderHeader" include="all"/>

<!-- With exclusions -->
<auto-parameters entity-name="mantle.order.OrderHeader" include="nonpk">
    <exclude field-name="statusId"/>
    <exclude field-name="lastUpdatedStamp"/>
</auto-parameters>
```

### Nested Parameters (for List/Map types)

```xml
<parameter name="items" type="List">
    <parameter name="productId" required="true"/>
    <parameter name="quantity" type="BigDecimal"/>
    <parameter name="unitAmount" type="BigDecimal"/>
</parameter>

<parameter name="address" type="Map">
    <parameter name="address1" required="true"/>
    <parameter name="city" required="true"/>
    <parameter name="stateProvinceGeoId"/>
    <parameter name="postalCode"/>
</parameter>
```

---

## XML Actions Complete Reference

> **CRITICAL — XSD Validation**: All XML Actions elements MUST conform to the Moqui XSD schema at `framework/xsd/xml-actions-3.xsd`. **Never invent element names.** Common mistakes:
> - `<make-value>` ✗ → `<entity-make-value>` ✓
> - `<sequenced-id-primary>` ✗ → `<entity-sequenced-id-primary>` ✓
> - `<find-one>` ✗ → `<entity-find-one>` ✓
> - `<find>` ✗ → `<entity-find>` ✓
>
> All entity operation elements are prefixed with `entity-`. When in doubt, check the XSD file or the element list below.

### Valid XML Actions Element Names (from XSD)

| Category | Elements |
|----------|----------|
| **Variables** | `set`, `script` |
| **Control flow** | `if`, `else-if`, `else`, `iterate`, `while`, `return`, `message` |
| **Entity read** | `entity-find-one`, `entity-find`, `entity-find-count`, `entity-find-related` |
| **Entity write** | `entity-make-value`, `entity-create`, `entity-update`, `entity-delete`, `entity-delete-by-condition`, `entity-delete-related`, `entity-set` |
| **Entity ID** | `entity-sequenced-id-primary`, `entity-sequenced-id-secondary` |
| **Service** | `service-call` |
| **Transaction** | `transaction` |
| **Logging** | `log` |
| **XML actions** | `actions` (nested block) |
| **Order by** | `order-by` (inside entity-find) |

### Variables & Expressions

```xml
<!-- Set from literal -->
<set field="status" value="active"/>

<!-- Set from expression -->
<set field="total" from="quantity * unitAmount"/>
<set field="nowTimestamp" from="ec.user.nowTimestamp"/>
<set field="formattedDate" from="ec.l10n.format(myDate, 'yyyy-MM-dd')"/>

<!-- Set with type conversion -->
<set field="amount" from="new BigDecimal(amountStr)" type="BigDecimal"/>

<!-- Default (only set if null) -->
<set field="statusId" from="statusId ?: 'DefaultStatus'"/>
```

### Entity Operations

```xml
<!-- FIND ONE -->
<entity-find-one entity-name="mantle.order.OrderHeader" value-field="order"/>
<entity-find-one entity-name="mantle.order.OrderHeader" value-field="order" for-update="true"/>
<entity-find-one entity-name="mantle.order.OrderHeader" value-field="order" cache="true"/>
<entity-find-one entity-name="mantle.order.OrderHeader" value-field="order">
    <field-map field-name="orderId" from="myOrderId"/>
</entity-find-one>

<!-- FIND LIST -->
<entity-find entity-name="mantle.order.OrderItem" list="items" cache="false"
        for-update="false" distinct="false" offset="0" limit="100">
    <econdition field-name="orderId"/>
    <econdition field-name="statusId" value="ItemApproved"/>
    <econdition field-name="statusId" operator="in" from="['ItemApproved','ItemCompleted']"/>
    <econdition field-name="quantity" operator="greater" from="0"/>
    <econdition field-name="description" operator="like" value="%search%"/>
    <econdition field-name="thruDate" operator="is-null"/>
    <econditions combine="or">
        <econdition field-name="typeA" value="X"/>
        <econdition field-name="typeB" value="Y"/>
    </econditions>
    <date-filter/>  <!-- fromDate <= now <= thruDate -->
    <date-filter date="someTimestamp" from-field-name="validFrom" thru-field-name="validThru"/>
    <select-field field-name="orderId,productId,quantity"/>
    <order-by field-name="orderItemSeqId"/>
    <order-by field-name="-unitAmount"/>  <!-- DESC -->
</entity-find>

<!-- OPTIONAL FILTER CONDITIONS -->
<!-- Use ignore-if-empty for simple equals conditions -->
<econdition field-name="statusId" ignore-if-empty="true"/>

<!-- PITFALL: For "like" operator with Groovy string interpolation, use ignore="!paramName" instead.
     ignore-if-empty checks the INTERPOLATED value. When paramName is null:
       value="%${paramName}%" evaluates to "%null%" which is NOT empty,
       so ignore-if-empty="true" does NOT skip the condition.
     This causes queries to fail with: WHERE field LIKE '%null%' -->

<!-- WRONG — %${null}% = "%null%" is non-empty, condition is NOT skipped -->
<econdition field-name="firstName" operator="like" value="%${firstName}%" ignore-if-empty="true"/>

<!-- CORRECT — ignore="!firstName" checks the RAW parameter before interpolation -->
<econdition field-name="firstName" operator="like" value="%${firstName}%" ignore="!firstName"/>
<econdition field-name="lastName" operator="like" value="%${lastName}%" ignore="!lastName"/>
<econdition field-name="email" operator="like" value="%${email}%" ignore="!email"/>

<!-- COUNT -->
<entity-find-count entity-name="mantle.order.OrderItem" count-field="itemCount">
    <econdition field-name="orderId"/>
</entity-find-count>

<!-- CREATE (via auto service) -->
<service-call name="create#mantle.order.OrderHeader" in-map="context" out-map="context"/>

<!-- UPDATE -->
<entity-find-one entity-name="mycomp.MyEntity" value-field="myEntity" for-update="true"/>
<set field="myEntity.statusId" value="NewStatus"/>
<entity-update value-field="myEntity"/>

<!-- DELETE -->
<entity-delete value-field="myEntity"/>
<entity-delete-by-condition entity-name="mycomp.MyEntityItem">
    <econdition field-name="myEntityId"/>
</entity-delete-by-condition>

<!-- MAKE VALUE + CREATE -->
<entity-make-value entity-name="mycomp.MyEntity" value-field="newEntity" map="context"/>
<entity-sequenced-id-primary value-field="newEntity"/>
<entity-create value-field="newEntity"/>

<!-- SET FIELDS from map -->
<entity-set value-field="existingEntity" map="updateMap" set-if-empty="false"/>
```

### Service Calls

```xml
<!-- Basic call -->
<service-call name="mantle.order.OrderServices.place#Order"
    in-map="[orderId: orderId]" out-map="placeOut"/>

<!-- Call with context (auto-maps matching field names) -->
<service-call name="create#mantle.order.OrderHeader" in-map="context" out-map="context"/>

<!-- Call with explicit map + additions -->
<service-call name="mycomp.MyServices.process#Order"
    in-map="context + [extraParam: 'value']" out-map="result"/>

<!-- Async call -->
<service-call name="mycomp.MyServices.send#Notification"
    in-map="[orderId: orderId]" async="true"/>

<!-- Distributed async (persisted in ServiceJob) -->
<service-call name="mycomp.MyServices.send#Notification"
    in-map="[orderId: orderId]" async="distribute"/>

<!-- Force new transaction -->
<service-call name="mycomp.MyServices.log#Activity"
    in-map="context" transaction="force-new"/>

<!-- Ignore errors -->
<service-call name="mycomp.MyServices.optional#Step"
    in-map="context" ignore-error="true"/>

<!-- Multi-call (iterate) -->
<service-call name="mycomp.MyServices.process#Item"
    in-map="[orderId: orderId, orderItemSeqId: item.orderItemSeqId]"
    multi="true"/>
```

### Control Flow

```xml
<!-- If/else -->
<if condition="order.statusId == 'OrderPlaced'">
    <then>
        <service-call name="..." in-map="..."/>
    </then>
    <else-if condition="order.statusId == 'OrderApproved'">
        <service-call name="..." in-map="..."/>
    </else-if>
    <else>
        <return error="true" message="Invalid status: ${order.statusId}"/>
    </else>
</if>

<!-- Iterate -->
<iterate list="orderItems" entry="item">
    <set field="item.statusId" value="ItemProcessed"/>
    <entity-update value-field="item"/>
</iterate>

<!-- While -->
<!-- Not directly available; use script block with while loop -->

<!-- Script block (Groovy) -->
<script><![CDATA[
    import java.math.RoundingMode
    def subtotal = BigDecimal.ZERO
    for (item in orderItems) {
        subtotal += (item.quantity ?: 0) * (item.unitAmount ?: 0)
    }
    context.subtotal = subtotal.setScale(2, RoundingMode.HALF_UP)
]]></script>
```

### Messages & Return

```xml
<!-- Info message (shown to user) -->
<message>Order ${orderId} has been placed successfully.</message>
<message type="info">Processing complete.</message>
<message type="warning">Low inventory for product ${productId}.</message>
<message type="success">Payment applied.</message>

<!-- Error and return -->
<return error="true" message="Order ${orderId} cannot be cancelled in status ${order.statusId}"/>

<!-- Return with type -->
<return type="warning" message="Order partially processed"/>
<return type="danger" message="Critical error occurred"/>

<!-- Check for errors -->
<if condition="ec.message.hasError()"><return/></if>

<!-- Simple return (stop execution) -->
<return/>

<!-- Return with public message (for REST/API) -->
<message public="true">Order created: ${orderId}</message>
```

### Logging

```xml
<log level="info" message="Processing order ${orderId} with ${items?.size() ?: 0} items"/>
<log level="warn" message="Inventory low for ${productId}: ${available}"/>
<log level="error" message="Failed to process payment for order ${orderId}"/>
<log level="debug" message="Item details: ${item}"/>
```

---

## Service ECA Rules

### SECA File (*.secas.xml)

```xml
<secas xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/service-eca-definition-3.xsd">

    <!-- Run after service completes successfully -->
    <seca service="mantle.order.OrderServices.place#Order"
          when="post-service" run-on-error="false" priority="5">
        <condition><expression>orderId</expression></condition>
        <actions>
            <service-call name="mycomp.MyOrderServices.after#OrderPlaced"
                in-map="[orderId: orderId]"/>
        </actions>
    </seca>

    <!-- Run before service -->
    <seca service="mantle.order.OrderServices.cancel#Order" when="pre-service">
        <condition><expression>orderId</expression></condition>
        <actions>
            <service-call name="mycomp.MyOrderServices.validate#OrderCancel"
                in-map="[orderId: orderId]"/>
        </actions>
    </seca>

    <!-- Run on commit (after transaction commits) -->
    <seca service="mantle.shipment.ShipmentServices.pack#Shipment"
          when="tx-commit" run-on-error="false">
        <actions>
            <service-call name="mycomp.MyShipmentServices.after#ShipmentPacked"
                in-map="context" async="true"/>
        </actions>
    </seca>
</secas>
```

### SECA `when` Values
- `pre-validate` — Before input validation
- `pre-service` — After validation, before execution
- `post-service` — After execution, before commit
- `tx-commit` — After transaction commit (async recommended)
- `pre-auth` — Before authorization check

---

## Transaction Control

### Service-Level Transaction

```xml
<!-- Default: use existing or begin new -->
<service verb="..." noun="..." transaction="use-or-begin"/>

<!-- Force new transaction (suspends existing) -->
<service verb="..." noun="..." transaction="force-new"/>

<!-- Force cache clear after completion -->
<service verb="..." noun="..." transaction="cache-clear"/>

<!-- No transaction -->
<service verb="..." noun="..." transaction="ignore"/>

<!-- Timeout in seconds -->
<service verb="..." noun="..." transaction="use-or-begin" transaction-timeout="300"/>
```

### Groovy Transaction API
```groovy
boolean beganTx = ec.transaction.begin(600)  // timeout in seconds
try {
    // operations
    ec.transaction.commit()
} catch (Exception e) {
    ec.transaction.rollback(beganTx, "Error", e)
    throw e
}
```

---

## Mantle USL Key Services

### Order Services (mantle.order.OrderServices)
```
create#Order          — Create OrderHeader + first OrderPart
create#OrderPart      — Add order part (ship group)
create#OrderItem      — Add item to order
update#OrderItem      — Update item qty/price
delete#OrderItem      — Remove item
add#OrderPartPayment  — Add payment method/info to order
place#Order           — Place order (triggers reservation)
approve#Order         — Approve order
cancel#Order          — Cancel order
cancel#OrderItem      — Cancel specific item
complete#OrderPart    — Complete order part
clone#Order           — Clone an existing order
```

### Order Info Services (mantle.order.OrderInfoServices)
```
get#OrderDisplayInfo  — Complete order info for display
```

### Shipment Services (mantle.shipment.ShipmentServices)
```
create#Shipment          — Create shipment
create#ShipmentItem      — Add item to shipment
create#ShipmentPackage   — Create package
pack#ShipmentProduct     — Pack product into package
pack#Shipment            — Mark shipment as packed (triggers invoicing)
ship#Shipment            — Mark as shipped
receive#EntireShipment   — Receive all items
```

### Product Services (mantle.product.ProductServices)
```
clone#Product             — Clone product
create#VariantProducts    — Create variants from features
find#ProductByIdValue     — Find by UPC, SKU, etc.
find#VariantProduct       — Find variant by feature combo
```

### Party Services (mantle.party.PartyServices)
```
create#Person              — Create person
create#PersonCustomer      — Create person + customer role + contact info
find#Party                 — Search parties
search#Party               — ElasticSearch party search
```

### Contact Services (mantle.party.ContactServices)
```
get#PartyContactInfo       — Get party's contact details
get#PartyContactInfoList   — Get all contact info for party
get#PrimaryEmailAddress    — Get primary email
store#PartyContactMech     — Add/update contact mech
delete#PartyContactMech    — Expire contact mech
```

### Payment Services (mantle.account.PaymentServices)
```
update#Payment             — Update payment details
```

### Asset/Inventory Services (mantle.product.AssetServices)
```
receive#Asset              — Receive inventory
get#AvailableInventory     — Check availability
move#AssetReservation      — Move reservation to different asset
```

### Price Services (mantle.product.PriceServices)
```
get#ProductPrice           — Calculate product price
```

---

## REST API Configuration

### Service REST API (.rest.xml in component)

REST API resource files use the naming convention `{component-name}.rest.xml` and **must be placed directly in the `service/` directory** (NOT in a subdirectory). The framework only scans direct children of `service/` for `*.rest.xml` files — files in subdirectories will NOT be discovered.

**File location**: `service/{component}.rest.xml` *(directly in service/, not in a subdirectory)*
**Schema**: `framework/xsd/rest-api-3.xsd`

```xml
<!-- File: service/mycomp.rest.xml  ← MUST be directly in service/ folder -->
<!-- General Guideline Verbs: GET=find, POST=create/do, PUT=store, PATCH=update, DELETE=delete -->
<resource xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="http://moqui.org/xsd/rest-api-3.xsd"
        name="mycomp" displayName="My Component REST API" version="1.0.0"
        description="REST API for my component.">

    <resource name="orders">
        <method type="get">
            <service name="mycomp.myapp.OrderServices.find#Orders"/>
        </method>
        <method type="post">
            <service name="mycomp.myapp.OrderServices.create#Order"/>
        </method>
        <id name="orderId">
            <method type="get">
                <service name="mycomp.myapp.OrderServices.get#OrderDetail"/>
            </method>
            <method type="patch">
                <service name="mycomp.myapp.OrderServices.update#Order"/>
            </method>
            <method type="delete">
                <service name="mycomp.myapp.OrderServices.cancel#Order"/>
            </method>
            <resource name="items">
                <method type="get">
                    <service name="mycomp.myapp.OrderServices.get#OrderItems"/>
                </method>
                <method type="post">
                    <service name="mycomp.myapp.OrderServices.add#OrderItem"/>
                </method>
            </resource>
        </id>
    </resource>
</resource>
```

The root `<resource name="...">` determines the base URL path: `/rest/s1/{name}/...`

Endpoint: `POST /rest/s1/mycomp/orders` → calls `create#Order`
Endpoint: `GET /rest/s1/mycomp/orders/100123` → calls `get#OrderDetail` with orderId=100123
Endpoint: `GET /rest/s1/mycomp/orders/100123/items` → calls `get#OrderItems` with orderId=100123

### REST Authentication Options

The `require-authentication` attribute can be set on `<resource>` elements to control access:

| Value | Meaning |
|-------|---------|
| `true` *(default)* | Requires authenticated user |
| `anonymous-view` | Allows anonymous access for read (GET) operations |
| `anonymous-all` | Allows anonymous access for all operations |

Authentication can be inherited — set it on the root `<resource>` and override on specific nested resources as needed.

### Authentication for REST
- API key in header: `Authorization: Bearer <api_key>`
- Basic auth: `Authorization: Basic <base64>`
- Login service: `POST /rest/s1/moqui/login` with `username` and `password`

---

## Service Facade Groovy API

```groovy
// Synchronous call
Map result = ec.service.sync().name("mantle.order.OrderServices.place#Order")
    .parameter("orderId", orderId)
    .call()

// With map of parameters
Map result = ec.service.sync().name("mycomp.MyServices.process#Data")
    .parameters([param1: "value1", param2: "value2"])
    .call()

// Async call
ec.service.async().name("mycomp.MyServices.send#Notification")
    .parameter("orderId", orderId)
    .call()

// Async distributed (persisted)
ec.service.async().name("mycomp.MyServices.send#Notification")
    .parameter("orderId", orderId)
    .distribute(true)
    .call()

// Special calls
ec.service.special().name("mycomp.MyServices.on#Commit")
    .parameter("orderId", orderId)
    .registerOnCommit()  // Run after current transaction commits
```

---

## Error Handling

### In XML Actions
```xml
<!-- Check for errors from sub-service -->
<service-call name="mycomp.MyServices.validate#Data" in-map="context" out-map="validateOut"/>
<if condition="ec.message.hasError()"><return/></if>

<!-- Add error message -->
<message error="true">Validation failed for ${fieldName}</message>
<return error="true" message="Cannot proceed"/>

<!-- Try-catch in script -->
<script><![CDATA[
try {
    ec.service.sync().name("...").parameters(context).call()
} catch (Exception e) {
    ec.logger.error("Service failed", e)
    ec.message.addError("Processing failed: ${e.message}")
}
]]></script>
```

### In Groovy Services
```groovy
// Add messages
ec.message.addMessage("Success!", "success")
ec.message.addError("Something failed")
ec.message.addValidationError(null, "fieldName", null, "Field is required")

// Check errors
if (ec.message.hasError()) return

// Public messages (exposed in REST API response)
ec.message.addPublicMessage("Order created", "success")
```

## Verify

After adding or changing a service:
1. Call it from `qapps/tools` → Service Run with sample input. Confirm response shape and that `ec.message.errors` is empty.
2. If the service has a SECA, trigger the parent service and tail `runtime/log/moqui.log` for `Calling SECA {name}`.
3. If exposed via REST, `curl -u admin:moqui http://localhost:8080/rest/s1/landmark/{resource}` — expect 200 and the documented shape; 401 without auth.

Common failures: `ServiceException: parameter X required` (auto-parameters not aligned with the entity), `EntityNotFoundException` (seed data missing), silent SECA (missing `if` predicate or wrong `when`).

See [`../SKILL.md`](../SKILL.md) §"Verify After Each Pattern" for the cross-cutting table.

<!-- begin: related-lessons (auto-generated by scripts/regen-skill-lesson-links.sh — do not edit by hand) -->

## Related Lessons

Cross-references from [`docs/ai/lessons.md`](../../docs/ai/lessons.md) that apply to this area. Filter tags: `Moqui-Services,Moqui-REST,Mantle-Reuse`. Regenerate with `scripts/regen-skill-lesson-links.sh`.

- 2026-05-28 — `OrderItem` has no direct `workEffortId` FK  `[Moqui-Entities,Mantle-Reuse,Registration]`
- 2026-05-28 — Polymorphic comms use stock bridges, not target-specific FKs  `[Communication,Mantle-Reuse]`

<!-- end: related-lessons -->
