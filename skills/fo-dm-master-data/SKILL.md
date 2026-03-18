---
name: fo-dm-master-data
description: >
  Migrate D365 F&O master data (Stage 2) — vendors, customers, products, BOMs,
  routes, fixed assets, bank accounts, and all reference/master entities. Covers
  entity selection, dependency sequencing, pre-migration validation, duplicate
  detection, address/dimension handling, and DMF performance optimization for
  high-volume master data loads.
  Use when: importing vendors, customers, products, or any master data into D365
  after configuration is complete.
  DEPENDS ON: fo-dm-configuration (Stage 1 must be complete).
metadata:
  platform:
    category: "fo-data-migration"
    riskLevel: "high"
    configSequence: 0
    configLevel: "cross-cutting"
    requires:
      products: ["F&O"]
      skills: ["fo-dm-configuration"]
    bpcProcess: "99 - Administer to Operate"
    msLearnPath: "https://learn.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/data-entities/data-entities-data-packages"
---

# Master Data Migration (Stage 2)

## Prerequisites
- ALL Stage 1 configuration complete and validated
- Financial dimension configuration for integrating applications is ACTIVE
- All posting profiles have valid GL accounts
- All reference data (groups, terms, tax codes, currencies) exists
- Target environment is Tier 2+ (performance representative)

## Master Data Entity Catalog

### Global Address Book (import first — shared across modules)

| Seq | Entity | Data Entity | Volume | Dependencies | Notes |
|---|---|---|---|---|---|
| 1 | Parties | `DirPartyTable` / `GlobalAddressBookParties` | Medium-High | Address data | Foundation for vendors/customers |
| 2 | Party addresses | `DirPartyPostalAddresses` | High | Parties, countries, states | Multiple per party |
| 3 | Party contacts | `DirPartyContacts` | Medium | Parties, contact types | Phone, email, fax |

**Note:** If importing vendors and customers directly (using VendorsV2/CustomersV3), party records are auto-created. Only import GAB separately if you have parties that aren't vendors or customers.

### Vendor Master Data

| Seq | Entity | Data Entity | Volume | Dependencies | Key Fields |
|---|---|---|---|---|---|
| 1 | Vendors | `VendorsV2` | High | Vendor groups, payment terms, tax groups, currencies, addresses | Account, name, group, terms, tax group, currency, address |
| 2 | Vendor addresses | (included in VendorsV2 or separate `DirPartyPostalAddresses`) | High | Vendors | Address roles (business, remit-to, delivery) |
| 3 | Vendor bank accounts | `VendorBankAccounts` | Medium | Vendors | Account#, routing#, SWIFT, IBAN |
| 4 | Vendor contacts | `VendorContacts` | Medium | Vendors | Name, role, phone, email |
| 5 | Vendor certifications | `VendorCertifications` | Low | Vendors | Insurance, diversity, quality certs |

**Pre-import validation queries (via MCP data tools):**
```
1. data_find_entities("VendorGroups") → verify all referenced groups exist
2. data_find_entities("PaymentTerms") → verify all payment terms exist
3. data_find_entities("SalesTaxGroups") → verify all tax groups exist
4. data_find_entities("Currencies") → verify all currencies exist
5. data_find_entities("AddressCountryRegions") → verify all countries exist
6. Duplicate check: vendor name + tax registration number uniqueness
```

### Customer Master Data

| Seq | Entity | Data Entity | Volume | Dependencies | Key Fields |
|---|---|---|---|---|---|
| 1 | Customers | `CustomersV3` | High | Customer groups, payment terms, tax groups, currencies | Account, name, group, terms, credit limit |
| 2 | Customer addresses | (included or `DirPartyPostalAddresses`) | High | Customers | Invoice, delivery, business addresses |
| 3 | Customer bank accounts | `CustomerBankAccounts` | Medium | Customers | For direct debit |
| 4 | Customer contacts | `CustomerContacts` | Medium | Customers | |

**⚠️ CustomersV3 limitation:** Does NOT support multi-threading. Set Import task count = 1. Error: "Custom sequence is defined, more than one task is not supported."

**Default dimension handling:**
- Requires `DefaultDimensionDisplayValue` field populated
- Format must match the ACTIVE "Default dimension format" in Financial dimension configuration for integrating applications
- Example: if format = Department-CostCenter, then value = "001-02"
- Dimensions not used for this customer = leave blank segment: "001-"
- This is a GLOBAL format — same format across all legal entities

### Product Master Data

| Seq | Entity | Data Entity | Volume | Dependencies | Key Fields |
|---|---|---|---|---|---|
| 1 | Product masters | `ProductsV2` | High | None (definition only) | Product number, name, type (item/service) |
| 2 | Released products | `ReleasedProductsV2` | Very High | Item model groups, item groups, dimension groups, units, sites | All product setup |
| 3 | Product dimensions | `ProductDimensionCombinations` | High | Product masters | Size, color, style, configuration |
| 4 | Released product variants | `ReleasedProductVariants` | Very High | Released products, dimensions | Per-variant release |
| 5 | Product attributes | `ProductAttributes` | Medium | Products | Custom attributes |
| 6 | Item default order settings | `ItemDefaultOrderSettings` | High | Released products | Default site, warehouse, order quantities |
| 7 | Item coverage groups | `ItemCoverageGroups` | Low | None | MRP planning parameters |
| 8 | Trade agreement journal headers | `TradeAgreementJournalHeaders` | Medium | None | Price journal headers |
| 9 | Trade agreement journal lines | `TradeAgreementJournalLines` | Very High | Headers, products, customers/vendors | Pricing rules |

**Product import tips:**
- `ReleasedProductsV2` is the primary entity for products that can be transacted
- Each product must be released to each legal entity where it will be used
- Item model group (costing method) is assigned at release — very hard to change later
- Default order settings control default site/warehouse — critical for MRP
- Trade agreements must be in POSTED journals to be active (import header → import lines → post via form tools)

### Fixed Asset Master Data

| Seq | Entity | Data Entity | Volume | Dependencies |
|---|---|---|---|---|
| 1 | Fixed assets | `FixedAssets` | Medium | FA groups |
| 2 | Fixed asset books | `FixedAssetBooks` | Medium | Fixed assets, value models |

### BOM and Route Data (Manufacturing)

| Seq | Entity | Data Entity | Volume | Dependencies |
|---|---|---|---|---|
| 1 | BOM headers | `BillOfMaterialsHeaders` | Medium | Released products |
| 2 | BOM lines | `BillOfMaterialsLines` | Very High | BOM headers, products, units |
| 3 | Route headers | `RouteHeaders` | Medium | Released products |
| 4 | Route operations | `RouteOperations` | High | Route headers, operations, resources |

**BOM/Route import notes:**
- BOM versions must be APPROVED and ACTIVATED to be used in production
- Routes must be APPROVED and ACTIVATED
- Import creates versions in Draft → must approve/activate via form tools or workflow
- BOM line item numbers must be released products with correct dimension groups

### Other Master Data

| Entity | Data Entity | Volume | Notes |
|---|---|---|---|
| Employees/workers | `HcmWorkerEntity` | Medium | If HR module in scope |
| Sales representatives | Linked to workers | Low | Commission setup |
| Warehouse workers | `WarehouseWorkers` | Medium | If WMS in scope |
| Cost prices (standard) | `InventoryItemPrices` | High | Must load BEFORE inventory on-hand for std cost items |

## Complete Migration Sequence (Stage 2)

```
Phase 2A: Foundation Master Data
├── 2A.1: Units of measure + conversions
├── 2A.2: Sites (if not in Stage 1)
├── 2A.3: Warehouses (if not in Stage 1)
└── 2A.4: Locations (if WMS)

Phase 2B: Financial Master Data
├── 2B.1: Main accounts (if not in Stage 1)
├── 2B.2: Financial dimension values
├── 2B.3: Bank accounts (if not in Stage 1)
├── 2B.4: Vendors + vendor bank accounts + contacts
├── 2B.5: Customers + customer bank accounts + contacts
└── 2B.6: Fixed assets + asset books

Phase 2C: Product Master Data
├── 2C.1: Product categories
├── 2C.2: Product masters (if using variants)
├── 2C.3: Released products
├── 2C.4: Product variants
├── 2C.5: Default order settings
├── 2C.6: Standard cost prices (for std cost items)
├── 2C.7: Trade agreements / price lists
├── 2C.8: BOM headers + lines
└── 2C.9: Route headers + operations

Phase 2D: Other Master Data
├── 2D.1: Workers/employees (if HR in scope)
├── 2D.2: Warehouse workers (if WMS)
└── 2D.3: Commission setup (if sales commission)
```

## DMF Performance Optimization for Master Data

### Volume-Specific Settings

| Entity | Typical Volume | Set-Based | Multi-Thread | Batch Mode | Notes |
|---|---|---|---|---|---|
| Vendors | 1K-50K | Yes | Yes | Yes | |
| Customers | 1K-100K | No (custom sequence) | **No (task=1)** | Yes | Single-threaded only |
| Released products | 5K-500K | Yes | Yes | Yes | Largest entity typically |
| Trade agreements | 10K-1M | Yes | Yes | Yes | Can be very high volume |
| BOM lines | 10K-500K | Yes | Yes | Yes | |
| Dimension values | 1K-50K | Yes | Yes | Yes | |

### Pre-Import Checklist
- [ ] Clean staging tables (Job history cleanup)
- [ ] Update SQL statistics (`sp_updatestats` on sandbox)
- [ ] Configure batch group with dedicated nodes
- [ ] Set Maximum batch threads = 12-16 (per server)
- [ ] Break files into chunks (start with 1K, then 10K, then 100K)
- [ ] Disable change tracking on entities being imported
- [ ] Test in Tier 2+ (not Tier 1 — performance not representative)

### Data Quality Pre-Checks
- [ ] No null values in mandatory fields
- [ ] No duplicate primary keys
- [ ] All foreign keys reference existing records (groups, terms, currencies)
- [ ] Address data valid (country → state → city hierarchy)
- [ ] Dimension values match active dimension format
- [ ] Currency codes are ISO standard
- [ ] Date formats consistent (ISO 8601 recommended)
- [ ] Number formats consistent (decimal separator)
- [ ] Text fields within max length limits
- [ ] Excel cells formatted as TEXT (not formulas — formulas cause DMF errors)

## Post-Import Validation

### Record Count Validation
```
For each entity imported:
1. Source file record count
2. Staging table record count (should match)
3. Target table record count (should match minus known skips)
4. Error file record count (should be 0 or explained)
```

### Referential Integrity Checks
```
1. Every vendor has a valid vendor group → data_find_entities filter
2. Every customer has a valid customer group
3. Every product has valid item model group + item group
4. Every released product has valid dimension groups
5. Every bank account links to valid GL main account
6. Every FA has valid FA group + value model
7. No orphaned party records (party without vendor/customer)
```

### Dimension Validation
```
1. Default dimensions on all masters resolve against dimension values
2. No invalid dimension combinations (account structure validation)
3. Entity-backed dimensions (Department, CostCenter) have matching records
```

## Common Master Data Migration Errors

| Error | Entity | Cause | Fix |
|---|---|---|---|
| "Vendor group not found" | VendorsV2 | Referenced group doesn't exist | Import vendor groups first (Stage 1) |
| "Customer import task count > 1" | CustomersV3 | Multi-threading not supported | Set task count = 1 |
| "Default dimension format error" | Any with dimensions | Format not configured | Set up Financial dimension config for integrating apps |
| "Product not released to company" | ReleasedProductsV2 | Wrong legal entity context | Set correct company on import project |
| "Item model group required" | ReleasedProductsV2 | Missing mandatory field | Populate in source file |
| "Unit not found" | Any with quantities | UoM not imported | Import units first (Phase 2A) |
| "Duplicate record" | Any | Record already exists in target | Enable "Ignore errors" to skip, or clean target |
| "Address validation failed" | VendorsV2/CustomersV3 | Invalid country/state/city | Verify GAB address data complete |
| "Formula in Excel cell" | Any Excel import | Cell contains formula not value | Copy-paste as values, or use CSV |
| "BOM version not approved" | BOMHeaders | Import creates Draft version | Must approve/activate via form tool post-import |

## Related Skills
- **Previous stage:** `fo-dm-configuration` (Stage 1)
- **Next stage:** `fo-dm-opening-balances` (Stage 3)
- **Optimization:** `fo-data-migration` (general DMF reference)
