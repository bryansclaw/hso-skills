# Procurement and Sourcing Overview

Source: Microsoft Learn (procurement-sourcing-overview) — verified 2026-03-18

## What Procurement Covers

The full procure-to-pay cycle: identifying need → sourcing → ordering → receiving → invoicing → payment

## Process Flow

```
Need Identification → Sourcing → Purchase Order → Receipt → Invoice → Payment
       ↓                ↓             ↓              ↓          ↓         ↓
  Requisition      RFQ/Vendor    PO Creation    Product    Invoice    Vendor
  or demand plan   Selection     & Approval     Receipt    Matching   Payment
```

## Need Identification

### Purchase Requisitions
- Employees request products/services
- **Product catalogs** guide selection of available items
- **Spending limits** constrain requisition amounts
- **Purchasing workflow** requires approval before ordering
- Budget fund allocation can be specified

### Demand from Planning
- Master planning (MRP) identifies demand requiring procurement
- Generates **planned purchase orders** → released to actual POs
- Automatic based on safety stock, reorder points, demand forecasts

## Sourcing

### Request for Quotation (RFQ)
- Send specifications to multiple potential vendors
- Vendors submit bids
- Procurement reviews and selects supplier
- Supports sealed/unsealed bidding

### Purchase Inquiry
- Simpler alternative to RFQ
- Establish terms (prices, discounts, delivery dates) directly with vendor
- Disabled if vendor uses Vendor Portal (order shared directly instead)

### Vendor Catalogs
- Vendors publish their own product catalogs
- Can attach **approved vendor list** to products
- Guides vendor selection and prevents using unintended vendors

## Purchase Order Creation Methods

POs can be created via:
1. **From master planning** — planned orders released to POs
2. **From purchase requisitions** — approved requisitions converted to POs
3. **From purchase agreements** — released orders from blanket agreements
4. **Manually** — direct PO creation not based on another document

## PO Lifecycle

```
Draft → Approved (via workflow) → Confirmed (sent to vendor) → 
    Received (product receipt) → Invoiced → Paid
```

### PO Confirmation
- Represents agreement established with vendor
- Triggers the vendor's delivery process
- PO status progresses through states until invoiced or canceled

### Default Values
When creating a PO, many fields pre-populate from the **Vendors** page:
- Payment terms, delivery terms, delivery mode
- Currency, tax group
- Procurement category defaults
- Minimal manual entry needed for standard orders

## Prices and Discounts

### Trade Agreements
- Represent vendor price lists with prices OR discounts
- Have specific date ranges (effective from/to)
- **Priority resolution (from most to least specific):**
  1. Vendor + Item specific
  2. Vendor + Item price group
  3. All vendors + Item
  4. etc. (same hierarchy as sales pricing)

### Purchase Agreements (Blanket Orders)
- **Product quantity commitment** — commit to buying X quantity over period
- **Product value commitment** — commit to spending $X over period
- Track fulfillment against commitment
- Released orders draw down the commitment balance

## Configuration Checklist

| Component | Navigation | Purpose |
|---|---|---|
| Procurement parameters | Procurement and sourcing > Setup > Procurement and sourcing parameters | Module-wide defaults |
| Procurement categories | Procurement and sourcing > Procurement categories | What can be purchased (hierarchical tree) |
| Purchasing policies | Procurement and sourcing > Setup > Policies > Purchasing policies | Rules for requisition control, PO creation, category access |
| Requisition workflows | Procurement and sourcing > Setup > Procurement and sourcing workflows | Approval routing for requisitions and POs |
| Purchase agreement classifications | Procurement and sourcing > Setup > Purchase agreement classifications | Types of blanket agreements |
| Auto charges | Procurement and sourcing > Setup > Charges > Charges setup | Auto-apply freight, handling to POs |

## Integration Points
- **AP:** PO receipt → vendor invoice → invoice matching → payment
- **Inventory:** PO receipt → inventory on-hand update
- **Budget control:** PO creation checks budget availability (encumbrance)
- **Dual-write:** POs sync to Dataverse (`PurchaseOrderHeadersV2` → `msdyn_purchaseorder`)
- **Master planning:** Planned POs generated from MRP runs
