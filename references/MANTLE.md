# Mantle Business Model & Process Reference

## Table of Contents
1. [Data Model Overview](#data-model-overview)
2. [Party & Contact Management](#party--contact-management)
3. [Product & Catalog](#product--catalog)
4. [Inventory & Assets](#inventory--assets)
5. [Order Management](#order-management)
6. [Shipment & Fulfillment](#shipment--fulfillment)
7. [Invoicing & Billing](#invoicing--billing)
8. [Payments](#payments)
9. [Returns](#returns)
10. [ProductStore Configuration](#productstore-configuration)
11. [Status Flow Reference](#status-flow-reference)
12. [Complete Order-to-Cash Flow](#complete-order-to-cash-flow)

---

## Data Model Overview

Mantle UDM is based on Len Silverston's "The Data Model Resource Book" with modern extensions. Key principles: Party pattern (generic people/orgs), ContactMech (reusable contact info), Status via StatusItem, Types via Enumeration, temporal validity (fromDate/thruDate), audit stamps.

Recommended learning order: Party → Product → Facility/Asset → Order → Shipment → Invoice → Payment → GL

---

## Party & Contact Management

Party is the supertype for Person and Organization. Each party has roles (PartyRole), relationships (PartyRelationship), and contact info (PartyContactMech → ContactMech → PostalAddress/TelecomNumber/email).

Common roles: `Customer`, `Vendor`, `OrgInternal`, `Supplier`, `Employee`, `Carrier`, `BillToCustomer`, `ShipToCustomer`

Contact purposes: `PostalPrimary`, `PostalShippingDest`, `PostalBilling`, `PhonePrimary`, `EmailPrimary`

Key services: `create#PersonCustomer` (full setup), `find#Party`, `get#PartyContactInfo`, `store#PartyContactMech`

---

## Product & Catalog

Product types: `PtAsset` (physical), `PtService`, `PtDigital`, `PtVirtual` (parent of variants). Virtual products use ProductAssoc (PatVariant) to link to variant products. Features (color/size) attached via ProductFeatureAppl with PfatDistinguishing on variants.

Pricing uses ProductPrice with priceTypeEnumId (PptList=MSRP, PptCurrent=sale), date ranges, min quantities, and customer-specific prices.

Key services: `get#ProductPrice`, `find#ProductByIdValue`, `create#VariantProducts`, `clone#Product`

---

## Inventory & Assets

Asset entity tracks inventory with quantityOnHandTotal and availableToPromiseTotal. AssetReservation holds stock for orders (reducing ATP). AssetIssuance records actual picks/ships (reducing QOH). AssetReceipt tracks receiving.

Key services: `receive#Asset`, `get#AvailableInventory`, `move#AssetReservation`

Flow: Receive → Available → Reserve (on order place) → Issue (on ship) → Adjust (returns/counts)

---

## Order Management

OrderHeader → OrderPart(s) → OrderItem(s). Each OrderPart represents a ship group with its own customer, vendor, shipping address, shipping method, and facility. OrderItems have productId, quantity, unitAmount, and itemTypeEnumId.

Item types: `ItemProduct`, `ItemDiscount`, `ItemShipping`, `ItemSalesTax`, `ItemService`

**Order Lifecycle:**
1. `create#Order` → OrderOpen
2. Add items via `create#OrderItem`
3. Set payment via `add#OrderPartPayment`
4. `place#Order` → OrderPlaced (reserves inventory)
5. `approve#Order` → OrderApproved (may auto-approve)
6. Create shipment, pack, ship → triggers invoicing
7. Payment captured/applied → OrderCompleted

Key services: `create#Order`, `create#OrderItem`, `place#Order`, `approve#Order`, `cancel#Order`, `get#OrderDisplayInfo`

---

## Shipment & Fulfillment

Shipment types: Sales Shipment (ShpTpSales), Purchase Shipment (ShpTpPurchase), Transfer, Sales Return, Purchase Return.

Structure: Shipment → ShipmentItem(s) → ShipmentItemSource (links to order/return items) → ShipmentPackage → ShipmentPackageContent → ShipmentRouteSegment (carrier, tracking).

**Outgoing Shipment Flow:**
1. Create Shipment from approved order
2. Add ShipmentItems (products and quantities)
3. Create ShipmentPackage(s)
4. `pack#ShipmentProduct` — Pack items into packages (creates AssetIssuance)
5. `pack#Shipment` → ShipPacked status (triggers Invoice creation via SECA)
6. `ship#Shipment` → ShipShipped (record actual ship date, tracking)
7. Delivered → ShipDelivered

**Incoming Shipment Flow (Purchase/Return):**
1. Create Shipment (ShpTpPurchase)
2. Add items
3. `receive#EntireShipment` — Receives all items (creates AssetReceipt, updates inventory)

Key services: `create#Shipment`, `create#ShipmentItem`, `pack#ShipmentProduct`, `pack#Shipment`, `ship#Shipment`, `receive#EntireShipment`

---

## Invoicing & Billing

Invoice: fromPartyId (seller) → toPartyId (buyer) with InvoiceItems. Types: Sales/Receivable (InvSalesInvoice), Purchase/Payable (InvPurchaseInvoice), Return (InvCustReturnCredit/InvRtnShipReplace).

InvoiceItem types: `ItemProduct`, `ItemShipping`, `ItemSalesTax`, `ItemDiscount`, `ItemExpense`

Each InvoiceItem can link back to order (orderId, orderItemSeqId), shipment (shipmentId), and asset.

**Invoice Status Flow:**
- Receivable: InvoiceInProcess → InvoiceFinalized → InvoiceApproved → InvoiceSent → InvoicePmtRecvd
- Payable: InvoiceIncoming → InvoiceReceived → InvoiceApproved → InvoicePaid

Invoices are typically auto-created when a shipment is packed (via SECA on `pack#Shipment`).

---

## Payments

Payment: fromPartyId → toPartyId, amount, paymentTypeEnumId, paymentMethodId, paymentInstrumentEnumId.

Payment types: `PtInvoicePayment`, `PtPrePayment`, `PtRefund`, `PtDisbursement`
Instruments: `PiCreditCard`, `PiDebitCard`, `PiEftAccount`, `PiFinancialAccount`, `PiCash`, `PiCheck`, `PiCompanyCheck`

PaymentApplication links payments to invoices (paymentId, invoiceId, amountApplied). A payment may apply to multiple invoices and an invoice may receive multiple payments.

**Payment Status Flow:**
PmntProposed → PmntPromised → PmntAuthorized → PmntConfirmed → PmntDelivered
Also: PmntDeclined, PmntCancelled, PmntRefunded, PmntVoid

---

## Returns

ReturnHeader + ReturnItem. Types: Sales Return (RtSales — customer returning), Purchase Return (RtPurchase — returning to vendor).

ReturnItem links to orderId/orderItemSeqId (original order) and tracks returnQuantity, returnPrice, returnReasonEnumId, responseEnumId (refund, credit, replacement).

Status flow: ReturnCreated → ReturnRequested → ReturnApproved → ReturnShipped → ReturnReceived → ReturnCompleted

---

## ProductStore Configuration

ProductStore is the central configuration for an e-commerce storefront:

```
ProductStore (productStoreId)
├── storeName, organizationPartyId (the selling org)
├── defaultCurrencyUomId, defaultLocale
├── inventoryFacilityId (default warehouse)
├── requireInventory (Y/N)
├── requireCustomerRole (Y/N)
├── reservationAutoEnumId (auto-reserve on place)
├── ProductStoreCategory — Store category tree
├── ProductStorePaymentGateway — Payment gateway config
├── ProductStoreShipOption — Available shipping methods
├── ProductStoreEmail — Email templates per event type
├── ProductStoreSetting — Key-value settings
└── ProductStoreFacility — Multiple warehouses
```

### Important Store Settings
```
<mantle.product.store.ProductStoreSetting productStoreId="MYSTORE"
    settingTypeEnumId="PsstOrderAutoApprove" settingValue="true"/>
<mantle.product.store.ProductStoreSetting productStoreId="MYSTORE"
    settingTypeEnumId="PsstShipAutoCreate" settingValue="true"/>
```

---

## Status Flow Reference

### Order Status
```
OrderOpen ──→ OrderProposed ──→ OrderPlaced ──→ OrderApproved ──→ OrderSent ──→ OrderCompleted
   │              │                  │               │               │
   └──→ OrderCancelled ←─────────────┴───────────────┘               │
                                     │                               │
                              OrderHeld ←────────────────────────────┘
```

### Shipment Status
```
ShipInput ──→ ShipScheduled ──→ ShipPicked ──→ ShipPacked ──→ ShipShipped ──→ ShipDelivered
   │              │                  │              │               │
   └──→ ShipCancelled ←─────────────┴──────────────┘               │
```

### Invoice Status (Receivable)
```
InvoiceInProcess ──→ InvoiceFinalized ──→ InvoiceApproved ──→ InvoiceSent ──→ InvoicePmtRecvd
       │                    │                    │                │
       └──→ InvoiceCancelled ←───────────────────┘                │
                                                    InvoiceWriteOff
```

### Payment Status
```
PmntProposed ──→ PmntPromised ──→ PmntAuthorized ──→ PmntConfirmed ──→ PmntDelivered
     │                │                 │                    │
     └──→ PmntCancelled ←──────────────┘                    │
                              PmntDeclined    PmntRefunded ←─┘
```

---

## Complete Order-to-Cash Flow

### Sales Order → Shipment → Invoice → Payment

```
1. CREATE ORDER
   OrderServices.create#Order → orderId, orderPartSeqId
   OrderServices.create#OrderItem → orderItemSeqId (repeat per product)
   OrderServices.add#OrderPartPayment → paymentId

2. PLACE ORDER
   OrderServices.place#Order
   ├── Status: OrderPlaced
   ├── Auto-reserves inventory (AssetReservation)
   └── May auto-approve (OrderApproved) via ProductStore setting

3. APPROVE ORDER (if not auto)
   OrderServices.approve#Order
   └── Status: OrderApproved

4. CREATE SHIPMENT
   ShipmentServices.create#Shipment (from order or auto via store setting)
   ShipmentServices.create#ShipmentItem (for each product)

5. PACK SHIPMENT
   ShipmentServices.pack#ShipmentProduct (per item)
   ├── Creates AssetIssuance (reduces inventory QOH)
   └── Verifies reservations
   ShipmentServices.pack#Shipment
   ├── Status: ShipPacked
   ├── TRIGGERS: Invoice creation (SECA)
   │   └── Creates Invoice + InvoiceItems
   └── TRIGGERS: Payment processing (if applicable)

6. SHIP
   ShipmentServices.ship#Shipment
   ├── Status: ShipShipped
   └── Records actual ship date, tracking

7. PAYMENT CAPTURE
   Payment authorized/captured → PmntDelivered
   PaymentApplication created linking payment to invoice

8. ORDER COMPLETION
   When all items shipped + paid:
   OrderServices.complete#OrderPart
   └── Status: OrderCompleted
```

### Purchase Order → Receiving → Invoice → Payment

```
1. CREATE PO
   OrderServices.create#Order (orderTypeEnumId="OtPurchase")
   └── vendorPartyId = supplier, customerPartyId = your org

2. PLACE & APPROVE PO

3. CREATE INCOMING SHIPMENT
   ShipmentServices.create#Shipment (shipmentTypeEnumId="ShpTpPurchase")

4. RECEIVE SHIPMENT
   ShipmentServices.receive#EntireShipment
   ├── Creates AssetReceipt records
   ├── Creates/updates Asset records (inventory in)
   └── Status: ShipDelivered

5. VENDOR INVOICE
   Create Invoice (invoiceTypeEnumId="InvPurchaseInvoice")
   └── fromPartyId = vendor, toPartyId = your org

6. PAY VENDOR
   Create Payment (fromPartyId = your org, toPartyId = vendor)
   Apply to invoice via PaymentApplication
```

---

## Common Integration Patterns

### E-Commerce API (PopRestStore pattern)
```
GET  /store/products/{productId}         → Product info
GET  /store/categories/{categoryId}      → Category + products
POST /store/cart/add                     → Add to cart (create OrderItem)
PUT  /store/cart/updateItem              → Update quantity
POST /store/orders/place                 → Place order
GET  /store/orders/{orderId}             → Order status
POST /store/customers/register           → Create customer
```

### Webhook/Event Pattern (SECA)
```xml
<!-- Notify external system when order is placed -->
<seca service="mantle.order.OrderServices.place#Order" when="tx-commit">
    <actions>
        <service-call name="mycomp.integration.WebhookServices.send#OrderWebhook"
            in-map="[orderId: orderId]" async="true"/>
    </actions>
</seca>
```

### Scheduled Jobs
```xml
<moqui.service.job.ServiceJob jobName="DailyInventorySync"
    serviceName="mycomp.integration.InventoryServices.sync#Inventory"
    cronExpression="0 0 2 * * ?"   <!-- 2 AM daily -->
    transactionTimeout="3600"/>
```

### SystemMessage Pattern (EDI/B2B)
```
SystemMessage → SystemMessageRemote (connection config)
├── sendServiceName — Service to send outbound
├── receiveServiceName — Service to process inbound
└── Used for: EDI, API integrations, file-based imports
```

## Verify

When reusing or extending a Mantle entity/service:
1. Read the entity definition in `mantle-udm/entity/*.xml` — confirm the FK shape and any existing extension fields.
2. Cross-check status flow: the Mantle status diagrams in this file are authoritative for `OrderHeader`, `WorkEffort`, `Payment`, etc.
3. If wiring a new SECA on a Mantle service, run the service once via Service Run and tail the log for `Calling SECA {name}`.
4. Confirm any extension field you added shows in the Mantle entity row via `qapps/tools` → Entity Data.

Common failures: assuming a direct FK when a bridge entity exists (see lessons tagged `[Mantle-Reuse]`); duplicating Mantle service logic instead of calling Mantle then extending it.

See [`../SKILL.md`](../SKILL.md) §"Verify After Each Pattern" for the cross-cutting table.

<!-- begin: related-lessons (auto-generated by scripts/regen-skill-lesson-links.sh — do not edit by hand) -->

## Related Lessons

Cross-references from [`docs/ai/lessons.md`](../../docs/ai/lessons.md) that apply to this area. Filter tags: `Mantle-Reuse,Moqui-Entities,Catalog,Registration,Communication`. Regenerate with `scripts/regen-skill-lesson-links.sh`.

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
