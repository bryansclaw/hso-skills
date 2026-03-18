---
name: fo-production-control
description: >
  Configure D365 F&O Production control including production parameters, resource
  types, resources, resource capabilities, route groups, operations, production
  groups, BOM/Route configuration, and production order lifecycle. Also covers
  master planning parameters for MRP.
  Use when: setting up discrete or process manufacturing, configuring BOMs and
  routes, or managing production orders.
  DEPENDS ON: fo-inventory-management (products, sites, warehouses must exist).
  Level 410-412 in Microsoft's config template sequence.
metadata:
  platform:
    category: "fo-production"
    riskLevel: "medium"
    configSequence: 10
    configLevel: "410-412"
    requires:
      products: ["F&O"]
      skills: ["fo-inventory-management"]
    bpcProcess: "04 - Plan to Produce"
    msLearnPath: "https://learn.microsoft.com/en-us/training/paths/configure-manage-production-dyn365-supply-chain-mgmt/"
    mbExam: "MB-330, MB-320 (Manufacturing)"
---

# Production Control Configuration

## Configuration Sequence

### Step 1: Production Parameters
- **Menu:** Production control > Setup > Production control parameters
- **MCP:** Form tool
- **Key tabs:** General, Standard update, Operations scheduling, Job scheduling, Number sequences
- **Important:** Default order type, auto-BOM consumption, auto-route consumption, reporting

### Step 2: Resource Types
- **Menu:** Organization administration > Resources > Resource types
- **Data entity:** `ResourceTypes`
- **MCP:** Data tool
- **Common types:** Machine, Human, Tool, Vendor (subcontracting), Facility

### Step 3: Resources (Work Centers/Machines)
- **Menu:** Organization administration > Resources > Resources
- **Data entity:** `Resources`
- **MCP:** Data tool
- **Key fields:** Resource ID, type, calendar (working hours), capabilities, costs
- **Calendar:** Each resource needs a working time calendar for scheduling

### Step 4: Resource Capabilities
- **Menu:** Organization administration > Resources > Resource capabilities
- **Data entity:** `ResourceCapabilities`
- **MCP:** Data tool
- **Purpose:** Define what each resource can do (welding, painting, CNC, etc.)
- **Used by:** Route operation requirements — scheduler matches capabilities to operations

### Step 5: Resource Groups
- **Menu:** Organization administration > Resources > Resource groups
- **MCP:** Form tool
- **Purpose:** Group resources for scheduling and costing (Assembly Line 1, Paint Shop, etc.)
- **Key:** Assign site + warehouse + input/output locations

### Step 6: Route Groups
- **Menu:** Production control > Setup > Routes > Route groups
- **Data entity:** `RouteGroups`
- **MCP:** Data tool
- **Purpose:** Control how route operations are estimated and posted
- **Settings:** Setup time, process time, overlap, auto-report, cost categories

### Step 7: Operations
- **Menu:** Production control > Setup > Routes > Operations
- **Data entity:** `Operations`
- **MCP:** Data tool
- **Examples:** Cut, Weld, Assemble, Paint, Test, Package
- **Linked to:** Route groups, resource requirements

### Step 8: Production Groups
- **Menu:** Production control > Setup > Production > Production groups
- **Data entity:** `ProductionGroups`
- **MCP:** Data tool
- **Purpose:** GL posting control for production orders (link WIP, issue, receipt accounts)

### Step 9: BOM Versions (Bill of Materials)
- **Menu:** Product information management > Bills of materials and formulas > Bills of materials
- **Data entities:** `BillOfMaterialsHeaders`, `BillOfMaterialsLines`
- **MCP:** Data tool for bulk import
- **Structure:** BOM header (for parent item) → BOM lines (component items + quantities)
- **Versioning:** Multiple BOM versions per product (date-effective, site-specific)

### Step 10: Route Versions
- **Menu:** Production control > Routes > All routes
- **Data entities:** `RouteHeaders`, `RouteOperations`
- **MCP:** Data tool
- **Structure:** Route header → Route operations (sequence of operations with resources/times)
- **Versioning:** Multiple route versions per product

### Step 11: Costing Setup (Level 420)
- **Menu:** Cost management > Costing versions > Costing version
- **MCP:** Form tool
- **For standard cost items:** Must set up cost prices BEFORE production or inventory import
- **Costing version types:** Standard cost, Planned cost
- **Cost categories:** Setup, Process, Quantity (linked to route group operations)

## Production Order Lifecycle
```
Created → Estimated → Scheduled → Released → Started → 
    Reported as finished → Ended (cost calculated)
```

Each stage has specific MCP form tool operations:
- Estimation: `form_click_control` → Estimate button
- Scheduling: Operations scheduling or Job scheduling
- Release: Triggers warehouse work (if WMS enabled)
- Reporting: Report quantities finished
- Ending: Calculates actual vs estimated cost, posts variances

## Process Manufacturing (Level 412)
If batch/process manufacturing (food, chemical, pharma):
- **Formulas** instead of BOMs (with co-products, by-products)
- **Batch attributes** — quality characteristics per batch
- **Potency** — variable active ingredient concentration
- **Batch orders** instead of production orders
- **Menu:** Process manufacturing > Setup

## Master Data Migration

| Seq | Entity | Data Entity | Dependencies |
|---|---|---|---|
| 1 | Resource types | `ResourceTypes` | None |
| 2 | Resources | `Resources` | Resource types, calendars |
| 3 | Capabilities | `ResourceCapabilities` | None |
| 4 | Resource groups | (Form tool) | Resources, sites |
| 5 | Route groups | `RouteGroups` | None |
| 6 | Operations | `Operations` | Route groups |
| 7 | Production groups | `ProductionGroups` | GL accounts |
| 8 | BOM headers | `BillOfMaterialsHeaders` | Released products |
| 9 | BOM lines | `BillOfMaterialsLines` | BOM headers, products, units |
| 10 | Route headers | `RouteHeaders` | Released products |
| 11 | Route operations | `RouteOperations` | Route headers, operations, resources |
| 12 | Cost categories | `CostCategories` | None |
| 13 | Costing versions | (Form tool) | Cost categories |
| 14 | Item prices | `InventoryItemPrices` | Costing version, products |

## Validation Checks
- [ ] Production parameters configured
- [ ] Resources created with working time calendars
- [ ] Resource capabilities defined and assigned
- [ ] Route groups configured with cost categories
- [ ] Operations defined
- [ ] Production groups linked to GL posting accounts
- [ ] At least one BOM + Route exists for a manufactured item
- [ ] Costing version created with item cost prices (for standard cost)
- [ ] Test: Create production order → estimate → schedule → start → report as finished → end → verify cost calculation and GL postings

## Common Errors
- "No BOM found" → BOM version not active for date/site or not approved
- "No route found" → Route version not active or not approved
- "Resource not available" → Calendar not assigned or all capacity booked
- "Cost price not found" → Standard cost not set in costing version
- Production variance unexpected → Cost categories or production group posting incorrect
- Scheduling failure → Resource capabilities don't match operation requirements

## Related Skills
- **Prerequisite:** `fo-inventory-management`
- **Related:** `fo-procurement` (raw material POs), `fo-warehouse-management` (material picking)
- **Costing:** Heavily integrated with GL via production groups and cost categories
