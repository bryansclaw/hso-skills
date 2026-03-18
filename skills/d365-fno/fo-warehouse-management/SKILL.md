---
name: fo-warehouse-management
description: >
  Configure D365 F&O advanced Warehouse management (WMS) including WMS
  parameters, location types, location profiles, zone groups, wave templates,
  work templates, location directives, mobile device setup, and warehouse
  processing flows. Use when: setting up advanced warehousing with directed
  put-away and picking, wave processing, or mobile device workflows.
  DEPENDS ON: fo-inventory-management (sites, warehouses, products must exist).
  Level 400 in Microsoft's config template sequence.
  NOT for: basic inventory management (use fo-inventory-management).
metadata:
  platform:
    category: "fo-warehouse"
    riskLevel: "medium"
    configSequence: 10
    configLevel: "400"
    requires:
      products: ["F&O"]
      skills: ["fo-inventory-management"]
    bpcProcess: "12 - Inventory to Deliver"
    msLearnPath: "https://learn.microsoft.com/en-us/training/paths/configure-manage-warehouse-management-dyn365-supply-chain-mgmt/"
    mbExam: "MB-330"
---

# Warehouse Management Configuration

## When to Use WMS vs Basic Inventory
| Feature | Basic Inventory | Advanced WMS |
|---|---|---|
| Locations | Optional | Required |
| License plates | No | Yes |
| Directed pick/put | No | Yes |
| Wave processing | No | Yes |
| Mobile device workflows | No | Yes |
| Work-based operations | No | Yes |

**Decision:** Enable WMS per warehouse (Warehouse > Setup > Warehouse > Use warehouse management processes = Yes). This is a one-way toggle — cannot be reversed.

## Configuration Sequence

### Step 1: WMS Parameters
- **Menu:** Warehouse management > Setup > Warehouse management parameters
- **MCP:** Form tool
- **Key settings:** Work processing, wave processing, load posting, mobile device

### Step 2: Location Types
- **Menu:** Warehouse management > Setup > Warehouse > Location types
- **MCP:** Form tool
- **Common types:** Bulk, Pick, Stage, Bay door, Quality, Overflow

### Step 3: Location Formats
- **Menu:** Warehouse management > Setup > Warehouse > Location formats
- **MCP:** Form tool
- **Purpose:** Define naming convention for locations (Aisle-Rack-Shelf-Bin)

### Step 4: Location Profiles
- **Menu:** Warehouse management > Setup > Warehouse > Location profiles
- **MCP:** Form tool
- **Controls:** Allow mixed items, allow mixed inventory status, allow cycle counting, dimensions

### Step 5: Zone Groups and Zones
- **Menu:** Warehouse management > Setup > Warehouse > Zone groups / Zones
- **MCP:** Form tool
- **Purpose:** Logical grouping of locations (Receiving zone, Picking zone, Staging zone)

### Step 6: Locations (WMS-enabled)
- **Menu:** Warehouse management > Setup > Warehouse > Locations
- **MCP:** Data tool or Form tool (use wizard for bulk generation)
- **Note:** WMS locations have more attributes than basic locations (profile, zone, dimensions)

### Step 7: Location Directives
- **Menu:** Warehouse management > Setup > Location directives
- **MCP:** Form tool (complex configuration with header/lines/actions)
- **Purpose:** Rules for WHERE to put/pick inventory
- **Types:** Purchase receive put, Sales pick, Transfer pick, Production pick, etc.
- **Logic:** Evaluated top-to-bottom, first match wins
- **⚠️ This is one of the most complex WMS configurations — order matters**

### Step 8: Work Templates
- **Menu:** Warehouse management > Setup > Work > Work templates
- **MCP:** Form tool
- **Purpose:** Define WHAT work is created (pick + put sequence)
- **Types:** Sales order pick, Purchase order put, Transfer pick, Production pick
- **Lines:** Pick from location type → Put to location type
- **Note:** Work templates and location directives work together — template says "do a pick", directive says "from where"

### Step 9: Wave Templates
- **Menu:** Warehouse management > Setup > Waves > Wave templates
- **MCP:** Form tool
- **Purpose:** Define HOW work is released to the warehouse floor
- **Steps:** Wave step methods (create work, allocate inventory, print labels, containerize)
- **Processing:** Can be manual or auto-process on release

### Step 10: Mobile Device Menu Items
- **Menu:** Warehouse management > Setup > Mobile device > Mobile device menu items
- **MCP:** Form tool
- **Purpose:** Define operations available on warehouse mobile devices
- **Common items:** Receive PO, Put away, Sales pick, Cycle count, Transfer
- **Activity types:** Work creation, Inquiry, Indirect activity

### Step 11: Mobile Device Menu
- **Menu:** Warehouse management > Setup > Mobile device > Mobile device menu
- **MCP:** Form tool
- **Purpose:** Organize menu items into the menu structure workers see on their devices

## Key WMS Processing Flows

### Inbound (Receiving)
```
PO Confirmation → Arrival → Receive (mobile device) → Put-away work created 
    → Location directive determines put location → Put-away (mobile device)
```

### Outbound (Shipping)
```
SO Released to warehouse → Wave processing → Work created → 
    Pick (mobile device, location directive determines pick location) → 
    Stage → Load → Ship → Packing slip/Invoice
```

### Cycle Counting
```
Cycle count plan → Work created → Count (mobile device) → 
    Variance → Adjustment journal (auto or approval)
```

## Validation Checks
- [ ] WMS enabled on warehouse (one-way toggle!)
- [ ] Location types defined
- [ ] Location profiles created
- [ ] Zones and zone groups configured
- [ ] Locations generated/imported
- [ ] Location directives configured for all work types needed
- [ ] Work templates configured for all process flows
- [ ] Wave templates created
- [ ] Mobile device menu items and menus configured
- [ ] Test: Complete receive → put-away → pick → ship cycle on mobile device

## Common Errors
- "No location directive found" → No matching directive for work type/site/warehouse
- "Work cannot be created" → Work template missing or doesn't match transaction type
- Location capacity exceeded → Location profile max dimensions/weight/volume reached
- "License plate not found" → LP tracking not enabled or LP not created
- Wave processing failure → Wave step methods missing or in wrong order

## Related Skills
- **Prerequisite:** `fo-inventory-management`
- **Related:** `fo-procurement` (inbound receiving), `fo-sales-marketing` (outbound shipping), `fo-production-control` (production material picking)
