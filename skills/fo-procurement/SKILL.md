---
name: fo-procurement
description: >
  Configure D365 F&O Procurement and sourcing including procurement parameters,
  procurement categories, purchasing policies, purchase requisition workflows,
  purchase agreement classifications, vendor evaluation, and purchase order
  processing. Use when: setting up procurement workflows, purchasing policies,
  or automating the procure-to-pay cycle.
  DEPENDS ON: fo-accounts-payable (vendor setup), fo-inventory-management
  (products/items). Level 320 in Microsoft's config template sequence.
metadata:
  platform:
    category: "fo-procurement"
    riskLevel: "medium"
    configSequence: 9
    configLevel: "320"
    requires:
      products: ["F&O"]
      skills: ["fo-accounts-payable", "fo-inventory-management"]
    bpcProcess: "05 - Source to Pay"
    msLearnPath: "https://learn.microsoft.com/en-us/training/paths/configure-manage-procurement-dyn365-supply-chain-mgmt/"
    mbExam: "MB-330 (Supply Chain Management)"
---

# Procurement and Sourcing Configuration

## Configuration Sequence

### Step 1: Procurement Parameters
- **Menu:** Procurement and sourcing > Setup > Procurement and sourcing parameters
- **MCP:** Form tool
- **Key tabs:** General, Vendor, Prices, Number sequences
- **Important settings:** Default purchase type, PO confirmation, change management, intercompany

### Step 2: Procurement Categories
- **Menu:** Procurement and sourcing > Procurement categories
- **MCP:** Form tool (hierarchical tree structure)
- **Purpose:** Classify what is being purchased (services, raw materials, MRO, IT, etc.)
- **Use:** Purchase requisitions, non-catalog purchasing, spend analysis

### Step 3: Purchasing Policies
- **Menu:** Procurement and sourcing > Setup > Policies > Purchasing policies
- **MCP:** Form tool
- **Policy rules:**
  - Purchase requisition control (approval requirements)
  - Purchase order creation and consolidation
  - Category access policy (who can buy what)
  - Catalog policy

### Step 4: Purchase Requisition Workflows
- **Menu:** Procurement and sourcing > Setup > Procurement and sourcing workflows
- **MCP:** Form tool
- **Common patterns:**
  - Amount-based approval tiers
  - Category-based routing (IT purchases → IT manager)
  - Auto-approval for low-value requisitions

### Step 5: Purchase Agreement Classifications
- **Menu:** Procurement and sourcing > Setup > Purchase agreement classifications
- **MCP:** Form tool
- **Purpose:** Classify blanket/framework agreements with vendors
- **Types:** Volume commitment, value commitment

### Step 6: Vendor Evaluation Criteria (Optional)
- **Menu:** Procurement and sourcing > Setup > Vendor evaluation criteria
- **MCP:** Form tool
- **Purpose:** Rate vendor performance (delivery, quality, price, service)

### Step 7: Charges Setup
- **Menu:** Procurement and sourcing > Setup > Charges > Charges setup
- **MCP:** Form tool
- **Purpose:** Auto-apply freight, handling charges to POs based on rules

## Purchase Order Processing Flow
```
Purchase Requisition → Approval Workflow → Purchase Order Creation
    → PO Confirmation → Product Receipt → Vendor Invoice → 
    Invoice Matching (2-way/3-way) → Payment
```

## MCP Agent Scenario: Automated PO Processing
Using the D365 MCP server, an agent can:
1. `form_find_menu_item("PurchTable")` → find Purchase orders form
2. `form_open_menu_item` → open the form
3. `form_click_control` → click New to create PO
4. `form_set_control_values` → set vendor, delivery date, site, warehouse
5. Add lines with items, quantities, prices
6. `form_click_control` → click Confirm to send to vendor
7. `form_save_form` → save

**Note:** For bulk PO creation, data tools (`data_create_entities` with `PurchaseOrderHeadersV2` and `PurchaseOrderLinesV2`) are more efficient.

## Dual-Write Awareness
- Purchase orders sync to Dataverse: `PurchaseOrderHeadersV2` → `msdyn_purchaseorder`
- If CE/Field Service is deployed, PO creation may flow from CE to F&O via dual-write

## Validation Checks
- [ ] Procurement parameters configured
- [ ] Procurement categories defined (at least top-level hierarchy)
- [ ] Purchasing policies active
- [ ] Purchase requisition workflow configured and active
- [ ] PO number sequence active
- [ ] Charges auto-application rules set (if applicable)
- [ ] Test: Create requisition → approve → generate PO → confirm → receive → invoice → pay

## Related Skills
- **Prerequisite:** `fo-accounts-payable`, `fo-inventory-management`
- **Related:** `fo-warehouse-management` (receiving), `fo-production-control` (production POs)
