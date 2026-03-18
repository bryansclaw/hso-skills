---
name: fo-inventory-management
description: >
  Configure D365 F&O Inventory management including inventory parameters, item
  model groups (costing methods), item groups, storage/tracking dimension groups,
  inventory posting profiles, sites, warehouses, locations, and product setup.
  Also covers inventory master data migration and opening inventory balance
  migration via movement journals. Use when: setting up inventory structure,
  costing methods (FIFO, LIFO, Standard Cost, Moving Average, Weighted Average),
  warehouse structure, or product configuration.
  DEPENDS ON: fo-gl-chart-of-accounts (inventory posting profiles link to GL).
  Level 300-310 in Microsoft's config template sequence.
  NOT for: advanced warehouse management (WMS waves/work templates), production
  control (use fo-production-control), procurement (use fo-procurement).
metadata:
  platform:
    category: "fo-inventory"
    riskLevel: "medium"
    configSequence: 8
    configLevel: "300-310"
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts"]
    bpcProcess: "12 - Inventory to Deliver"
    msLearnPath: "https://learn.microsoft.com/en-us/training/paths/configure-manage-inventory-dyn365-supply-chain-mgmt/"
    mbExam: "MB-330 (Supply Chain Management)"
    sbdPhase: "Implement"
---

# Inventory Management Configuration

## Microsoft Learn References
See `references/ms-learn-sources.md` for verified URLs covering inventory parameters, item model groups (costing methods), posting profiles, dimension groups, product information, inventory close, GL reconciliation, and entity documentation.

## Prerequisites
- GL chart of accounts with inventory-related main accounts (inventory, COGS, purchase price variance, production variance, etc.)
- Legal entities configured with sites/warehouse address structure

## ⚠️ Critical Design Decisions (Must Be Made Before Configuration)

### Costing Method Selection
| Method | Best For | Key Characteristic |
|---|---|---|
| **FIFO** | Standard retail/distribution | First items in = first items out |
| **LIFO** | Tax optimization (US) | Last items in = first items out |
| **Weighted Average** | Commodity/bulk items | Average cost recalculated per receipt |
| **Moving Average** | Real-time cost tracking | Cost updates with each receipt |
| **Standard Cost** | Manufacturing | Predetermined cost, variances tracked |
| **LIFO Date / FIFO Date** | Date-specific tracking | FIFO/LIFO within date groups |

**This decision is set in Item Model Groups and is very difficult to change after go-live.**

### Dimension Group Design
- **Storage dimensions:** Site (mandatory), Warehouse, Location, Pallet, Inventory status
- **Tracking dimensions:** Batch number, Serial number, Owner
- **These determine how granularly inventory is tracked — set wrong = major rework**

## Configuration Sequence

### Step 1: Inventory Parameters
- **Menu:** Inventory management > Setup > Inventory and warehouse management parameters
- **MCP:** Form tool (multi-tab parameter form)
- **Key tabs:** General, Financial update, Counting, Quality management, Number sequences

### Step 2: Item Model Groups
- **Menu:** Inventory management > Setup > Inventory > Item model groups
- **Data entity:** `ItemModelGroups`
- **MCP:** Data tool
- **Critical setting:** Inventory model (FIFO/LIFO/Std Cost/etc.)
- **Also controls:** Physical/financial posting, negative inventory, stocked/non-stocked

### Step 3: Item Groups
- **Menu:** Inventory management > Setup > Inventory > Item groups
- **Data entity:** `ItemGroups`
- **MCP:** Data tool
- **Purpose:** Classify items for reporting and GL posting profiles

### Step 4: Storage Dimension Groups
- **Menu:** Inventory management > Setup > Dimension > Storage dimension groups
- **Data entity:** `StorageDimensionGroups`
- **MCP:** Data tool
- **Defines:** Which storage dimensions (Site, Warehouse, Location) are active and financial vs physical

### Step 5: Tracking Dimension Groups
- **Menu:** Inventory management > Setup > Dimension > Tracking dimension groups
- **Data entity:** `TrackingDimensionGroups`
- **MCP:** Data tool
- **Defines:** Batch/serial number tracking requirements

### Step 6: Inventory Posting Profiles
- **Menu:** Inventory management > Setup > Posting > Posting
- **MCP:** Form tool (complex matrix of accounts by item group, category, and transaction type)
- **Critical accounts:** Physical inventory, financial inventory, COGS, purchase expenditure, production issue, production receipt, profit/loss
- **Note:** This is where inventory flows connect to GL — misaligned posting profiles are the #1 cause of inventory-to-GL reconciliation failures

### Step 7: Sites
- **Menu:** Inventory management > Setup > Inventory breakdown > Sites
- **Data entity:** `Sites`
- **MCP:** Data tool
- **Dependency:** Legal entity (each site belongs to one legal entity)
- **Note:** Site is the primary financial dimension for inventory — most inventory transactions require a site

### Step 8: Warehouses
- **Menu:** Inventory management > Setup > Inventory breakdown > Warehouses
- **Data entity:** `Warehouses`
- **MCP:** Data tool
- **Dependency:** Site (each warehouse belongs to one site)

### Step 9: Locations
- **Menu:** Inventory management > Setup > Inventory breakdown > Locations
- **Data entity:** `WMSLocations`
- **MCP:** Data tool
- **Dependency:** Warehouse

### Step 10: Units of Measure
- **Menu:** Organization administration > Setup > Units > Units
- **Data entity:** `Units`, `UnitConversions`
- **MCP:** Data tool
- **Note:** Define all units (ea, kg, lb, box, pallet) + conversion factors

### Step 11: Product Configuration (Level 310)

#### Released Products
- **Menu:** Product information management > Products > Released products
- **Data entity:** `ReleasedProductsV2`
- **MCP:** Data tool (can be very high volume — thousands of SKUs)
- **Critical fields:** Item number, product name, item model group, item group, storage dimension group, tracking dimension group, default unit of measure
- **Dependencies:** All groups from Steps 2-5 must exist before product import

#### Product Variants (if applicable)
- **Data entity:** `ReleasedProductVariants`
- **Dependency:** Released products + product dimension groups (size, color, style, configuration)

#### BOMs (if manufacturing)
- **Data entities:** `BillOfMaterialsHeaders`, `BillOfMaterialsLines`
- **Dependency:** Released products

## Master Data Migration Sequence

| Seq | Entity | Data Entity Name | Dependencies |
|---|---|---|---|
| 1 | Units of measure | `Units` | None |
| 2 | Unit conversions | `UnitConversions` | Units |
| 3 | Sites | `Sites` | Legal entities |
| 4 | Warehouses | `Warehouses` | Sites |
| 5 | Locations | `WMSLocations` | Warehouses |
| 6 | Item model groups | `ItemModelGroups` | None |
| 7 | Item groups | `ItemGroups` | GL posting profiles |
| 8 | Storage dimension groups | `StorageDimensionGroups` | None |
| 9 | Tracking dimension groups | `TrackingDimensionGroups` | None |
| 10 | Product categories | `ProductCategories` | None |
| 11 | Released products | `ReleasedProductsV2` | Groups (6-9), units, categories |
| 12 | Product variants | `ReleasedProductVariants` | Released products |
| 13 | BOM headers | `BillOfMaterialsHeaders` | Released products |
| 14 | BOM lines | `BillOfMaterialsLines` | BOM headers, released products |
| 15 | On-hand inventory | Via movement journal | Products, sites, warehouses |

## Opening Inventory Balances
- Import via **Inventory Movement Journal** (not direct GL entry)
- Movement journal posts to both inventory subledger AND inventory GL accounts
- After import: Inventory valuation report total MUST equal inventory control accounts in GL
- For standard cost items: set standard cost BEFORE importing on-hand (otherwise variance calculated)

## GL-to-Inventory Reconciliation
The accounts defined for receipts, issues, and consumption must stay consistent across purchasing, sales, and production transactions. Misaligned posting profiles are the most common cause of unexplained balances.

**Reconciliation points:**
- Inventory posting profile accounts = GL inventory control accounts
- Physical vs financial posting timing differences
- Pending invoices (received not invoiced)
- Inventory close/recalculation adjustments

## Validation Checks
- [ ] Inventory parameters configured
- [ ] Item model groups exist for each costing method needed
- [ ] Item groups exist and have inventory posting profiles assigned
- [ ] Dimension groups correctly define financial vs physical tracking
- [ ] Inventory posting profiles have valid GL accounts for all transaction types
- [ ] Sites and warehouses created for all legal entities
- [ ] At least one released product can be created and transacted
- [ ] Test: Create item → receive PO → sell SO → verify GL postings match posting profile

## Common Errors
- "Posting profile not found for transaction type" → Missing account in inventory posting setup
- "Item not found in warehouse" → Product not released to correct legal entity
- "Dimension group mismatch" → Product assigned wrong dimension group
- Inventory-GL reconciliation mismatch → Posting profiles changed after transactions posted
- Standard cost variance unexpected → Standard cost not set before on-hand import

## Related Skills
- **Prerequisite:** `fo-gl-chart-of-accounts`
- **Next:** `fo-procurement` (Level 320), `fo-sales-marketing` (Level 330), `fo-production-control` (Level 410)
- **Dual-write:** Products sync to Dataverse via `ReleasedProductsV2` → `msdyn_sharedproductdetails`
