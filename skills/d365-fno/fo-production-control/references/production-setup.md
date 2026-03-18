# Production Control Setup

Source: Microsoft Learn + verified research — 2026-03-18

## Resource Configuration

### Resource Types
Navigation: **Organization administration > Resources > Resource types**
- Machine, Human, Tool, Vendor (subcontracting), Facility
- Define capabilities per type

### Resources (Work Centers)
Navigation: **Organization administration > Resources > Resources**
- Each resource needs: ID, type, working time calendar, cost per hour
- **Calendar is critical** — scheduling uses calendar to determine available capacity
- Without calendar → scheduling fails with "Resource not available"

### Resource Groups
Navigation: **Organization administration > Resources > Resource groups**
- Group resources for scheduling and costing
- Assign: site + warehouse + input/output warehouse locations
- Resource groups determine where materials are issued from and finished goods received to

## Route Configuration

### Route Groups
Navigation: **Production control > Setup > Routes > Route groups**
Controls how route operations are estimated and posted:
- Setup time estimation method
- Process time estimation method
- Whether to auto-report operations as finished
- Cost categories for each time type (setup, process, quantity)

### Operations
Navigation: **Production control > Setup > Routes > Operations**
- Define the discrete manufacturing steps (Cut, Weld, Assemble, Paint, Test, Package)
- Each operation references a route group
- Operation relations link operations to specific resources or resource groups

### Creating Routes
Navigation: **Production control > Routes > All routes**
- Route header: parent item, site, date effectivity
- Route operations: sequence of steps with:
  - Operation reference
  - Resource requirement (specific resource, resource group, or capability-based)
  - Setup time, process time, queue times
  - Overlap allowed (for parallel processing)
- Routes must be **Approved** and **Activated** to be used

## BOM Configuration

### Creating BOMs
Navigation: **Product information management > Bills of materials and formulas > Bills of materials**
- BOM header: parent item, site, from/to dates (version effectivity)
- BOM lines: component items with:
  - Item number (must be released product)
  - Quantity (per parent unit)
  - Unit of measure
  - Position (for visual BOM)
  - Warehouse (where to pick from)
- BOMs must be **Approved** and **Activated**
- Can have multiple versions (date-effective, site-specific)

## Production Order Lifecycle

```
Created → Estimated → Scheduled → Released → Started → Reported → Ended
```

| Stage | What Happens | GL Impact |
|---|---|---|
| **Created** | Order exists, references BOM + Route | None |
| **Estimated** | Calculates expected cost based on BOM + Route | None |
| **Scheduled** | Operations scheduled on resources (capacity reserved) | None |
| **Released** | Materials available for picking, work orders created | None |
| **Started** | Production begins, materials issued (pick list) | WIP accounts debited, inventory credited |
| **Reported as finished** | Quantity of finished goods reported | Finished goods received to inventory |
| **Ended** | Actual cost calculated, variances posted | Variance accounts (price, quantity, cost category) |

## Production Groups
Navigation: **Production control > Setup > Production > Production groups**
- Link production orders to GL posting accounts
- Define: WIP account, WIP cost value account, cost of goods manufactured account
- Different production groups for different product lines = different GL account sets

## Costing (Level 420)

### Standard Cost Items
- **Must set standard cost BEFORE production or inventory import**
- Navigation: Cost management > Costing versions > [version] > Item prices
- Enter cost per unit for each item
- If cost not set → variance calculated against $0

### Cost Categories
- Setup, Process, Quantity categories
- Linked to route group operations
- Determine which GL accounts different cost types post to

## Common Errors
- "No BOM found" → BOM version not active for the date/site, or not approved
- "No route found" → Route version not active/approved
- "Resource not available" → Calendar not assigned or capacity fully booked
- "Cost price not found" → Standard cost not set in costing version
- Unexpected production variance → Cost categories or production group posting incorrect
