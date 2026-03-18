# Pre-Validation Service Design

## Problem
DMF imports are slow and give errors only after full staging/processing. For a 50K vendor file, you might wait 30 minutes only to discover 200 rows have invalid vendor groups. Need a fast, lightweight pre-check.

## Solution: OData-Based Pre-Validation

Query the target D365 environment's reference data via OData BEFORE running DMF. Compare source file values against valid reference values. Report all mismatches in seconds.

### Architecture

```
Source Data File (CSV/Excel)
    │
    ▼
┌─────────────────────────────┐
│  Pre-Validation Service     │
│                             │
│  1. Parse source file       │
│  2. Identify foreign keys   │
│  3. Query target via OData  │
│  4. Compare & report        │
└─────────────┬───────────────┘
              │ OData REST calls
              ▼
┌─────────────────────────────┐
│  D365 F&O Environment       │
│  (OData endpoint)           │
│                             │
│  /data/VendorGroups         │
│  /data/PaymentTerms         │
│  /data/SalesTaxGroups       │
│  /data/Currencies           │
│  /data/MainAccounts         │
│  etc.                       │
└─────────────────────────────┘
```

### OData Endpoints for Reference Data

No X++ required. These are built-in OData entities:

#### Foundation Reference Data
| Reference | OData Endpoint | Key Field | Typical Size |
|---|---|---|---|
| Legal entities | `/data/LegalEntities` | `DataArea` | <100 |
| Currencies | `/data/Currencies` | `CurrencyCode` | <50 |
| Countries | `/data/AddressCountryRegions` | `CountryRegionId` | <300 |
| States | `/data/AddressStates` | `StateId` (+ CountryRegionId) | <5000 |
| Units of measure | `/data/Units` | `UnitSymbol` | <200 |

#### Financial Reference Data
| Reference | OData Endpoint | Key Field | Typical Size |
|---|---|---|---|
| Main accounts | `/data/MainAccounts` | `MainAccountId` | 100-5000 |
| Financial dimension values | `/data/FinancialDimensionValues` | `DimensionValue` (+ DimensionName) | 100-10000 |
| Fiscal periods | `/data/FiscalCalendarPeriods` | `Name` + `StartDate` | <200 |

#### AP Reference Data
| Reference | OData Endpoint | Key Field | Typical Size |
|---|---|---|---|
| Vendor groups | `/data/VendorGroups` | `VendorGroupId` | <50 |
| Payment terms | `/data/PaymentTerms` | `PaymentTermsName` | <50 |
| Cash discounts | `/data/CashDiscounts` | `CashDiscountCode` | <30 |
| Vendor payment methods | `/data/VendorPaymentMethods` | `Name` (+ company) | <20 |

#### AR Reference Data
| Reference | OData Endpoint | Key Field | Typical Size |
|---|---|---|---|
| Customer groups | `/data/CustomerGroups` | `CustomerGroupId` | <50 |
| Customer payment methods | `/data/CustomerPaymentMethods` | `Name` (+ company) | <20 |

#### Tax Reference Data
| Reference | OData Endpoint | Key Field | Typical Size |
|---|---|---|---|
| Sales tax groups | `/data/SalesTaxGroups` | `TaxGroup` | <100 |
| Item sales tax groups | `/data/ItemSalesTaxGroups` | `TaxItemGroup` | <50 |
| Tax exempt numbers | `/data/TaxExemptNumbers` | `TaxExemptNumber` | variable |

#### Inventory Reference Data
| Reference | OData Endpoint | Key Field | Typical Size |
|---|---|---|---|
| Item model groups | `/data/ItemModelGroups` | `ModelGroupId` | <20 |
| Item groups | `/data/ItemGroups` | `ItemGroupId` | <50 |
| Storage dimension groups | `/data/StorageDimensionGroups` | `StorageDimensionGroupName` | <10 |
| Tracking dimension groups | `/data/TrackingDimensionGroups` | `TrackingDimensionGroupName` | <10 |
| Sites | `/data/Sites` | `SiteId` | <50 |
| Warehouses | `/data/Warehouses` | `WarehouseId` (+ SiteId) | <200 |
| Released products | `/data/ReleasedProductsV2` | `ItemNumber` | 1K-500K |

#### Bank Reference Data
| Reference | OData Endpoint | Key Field | Typical Size |
|---|---|---|---|
| Bank accounts | `/data/BankAccountsV2` | `BankAccountId` | <50 |
| Bank groups | `/data/BankGroups` | `BankGroupId` | <10 |

#### FA Reference Data
| Reference | OData Endpoint | Key Field | Typical Size |
|---|---|---|---|
| Fixed asset groups | `/data/FixedAssetGroups` | `GroupId` | <30 |

### Validation Rules Per Entity Import

#### Vendor Import (VendorsV2) — Pre-Validation Checks
```
For each row in source file:
  1. VendorGroupId → query /data/VendorGroups → exists?
  2. PaymentTermsName → query /data/PaymentTerms → exists?
  3. TaxGroup → query /data/SalesTaxGroups → exists?
  4. CurrencyCode → query /data/Currencies → exists?
  5. CountryRegionId → query /data/AddressCountryRegions → exists?
  6. StateId → query /data/AddressStates?$filter=CountryRegionId eq '{country}' → exists?
  7. DefaultDimensionDisplayValue → parse per active dimension format → each segment value → query /data/FinancialDimensionValues → exists?
  8. Duplicate check: VendorAccountNumber uniqueness within file
  9. Duplicate check: VendorAccountNumber not already in target (query /data/VendorsV2?$filter=VendorAccountNumber eq '{value}')
```

#### Customer Import (CustomersV3) — Pre-Validation Checks
```
Same pattern as vendor but with:
  1. CustomerGroupId → /data/CustomerGroups
  2. PaymentTermsName → /data/PaymentTerms  
  3. TaxGroup → /data/SalesTaxGroups
  4. CurrencyCode → /data/Currencies
  5-7. Address + dimension validation (same as vendor)
  8-9. Duplicate checks on CustomerAccount
```

#### Product Import (ReleasedProductsV2) — Pre-Validation Checks
```
  1. ItemModelGroupId → /data/ItemModelGroups
  2. ItemGroupId → /data/ItemGroups
  3. StorageDimensionGroupName → /data/StorageDimensionGroups
  4. TrackingDimensionGroupName → /data/TrackingDimensionGroups
  5. BOMUnitSymbol → /data/Units
  6. PurchaseUnitSymbol → /data/Units
  7. SalesUnitSymbol → /data/Units
  8. Duplicate check: ItemNumber uniqueness
```

#### General Journal (LedgerJournalEntity) — Pre-Validation Checks
```
  1. JOURNALNAME → /data/LedgerJournalNames → exists?
  2. CURRENCYCODE → /data/Currencies → exists?
  3. ACCOUNTDISPLAYVALUE → parse per ledger dimension format:
     - MainAccount segment → /data/MainAccounts → exists?
     - Each dimension segment → /data/FinancialDimensionValues → exists?
  4. TRANSDATE → /data/FiscalCalendarPeriods → period is Open?
  5. VOUCHER uniqueness within file
  6. Debit/Credit balance check per voucher (must net to 0)
```

### Caching Strategy

Reference data changes infrequently. Cache OData results:

```
Cache TTL:
  Currencies, Countries, Units     → 24 hours (rarely changes)
  Vendor/Customer groups           → 4 hours (changes during config phase)
  Tax codes/groups                 → 4 hours
  Main accounts, dimensions        → 1 hour (changes during config phase)
  Sites, warehouses                → 4 hours
  Released products                → 30 minutes (may change during product setup)
  
Invalidate cache: when config agent makes changes to reference data
```

### Performance Comparison

| Approach | Validate 50K Vendors | Validate 100K Products |
|---|---|---|
| **DMF import (full cycle)** | 30-60 minutes | 60-120 minutes |
| **OData pre-validation** | 30-60 seconds | 2-5 minutes |
| **Speedup** | ~60x faster | ~30x faster |

OData pre-validation is faster because:
- No staging table writes
- No field mapping processing  
- No business logic execution (validateWrite, insert methods)
- No voucher/number sequence allocation
- Only reading small reference tables (not processing the import file through D365)

### Implementation Options

#### Option A: .NET Service (for your Semantic Kernel platform)
```csharp
public class PreValidationService
{
    private readonly HttpClient _odataClient; // authenticated OData client
    private readonly IMemoryCache _cache;
    
    public async Task<ValidationReport> ValidateVendorFile(
        Stream sourceFile, 
        string environment)
    {
        // 1. Load reference data (cached)
        var vendorGroups = await GetCached<HashSet<string>>(
            $"{environment}/data/VendorGroups", 
            r => r.Select(x => x.VendorGroupId));
        var paymentTerms = await GetCached<HashSet<string>>(...);
        var taxGroups = await GetCached<HashSet<string>>(...);
        
        // 2. Parse source file
        var rows = ParseCsv(sourceFile);
        
        // 3. Validate each row
        var errors = new List<ValidationError>();
        foreach (var row in rows)
        {
            if (!vendorGroups.Contains(row.VendorGroupId))
                errors.Add(new("VendorGroupId", row.Line, 
                    $"Group '{row.VendorGroupId}' not found in target"));
            // ... repeat for each foreign key
        }
        
        return new ValidationReport(rows.Count, errors);
    }
}
```

#### Option B: MCP Agent Workflow (for AI-driven validation)
```
Agent receives: "Pre-validate vendor migration file"

1. Parse source file → extract unique foreign key values
2. For each reference type:
   data_find_entities("VendorGroups") → get all valid group IDs
   Compare against source file values → collect missing
3. Report:
   "Pre-validation of 50,230 vendors:
    ✅ 48 vendor groups — all found
    ✅ 12 payment terms — all found  
    ⚠️  3 tax groups missing: EXEMPT-01, EXPORT-CA, SERVICE-NY
    ✅ 8 currencies — all found
    ❌ 147 rows have invalid state/country combinations
    ❌ 23 duplicate vendor account numbers in file
    
    Blockers: 2 (tax groups, duplicates)
    Fix tax groups before proceeding?"
```

#### Option C: Python Script (standalone, uses d365fo-client)
```python
from d365fo_client import FOClient, FOClientConfig

async def pre_validate(source_file, environment_url):
    config = FOClientConfig(base_url=environment_url)
    async with FOClient(config) as client:
        # Pull reference data
        vendor_groups = {r['VendorGroupId'] 
            for r in await client.query("VendorGroups", select="VendorGroupId")}
        payment_terms = {r['PaymentTermsName'] 
            for r in await client.query("PaymentTerms", select="PaymentTermsName")}
        
        # Validate source
        errors = []
        for row in parse_csv(source_file):
            if row['VendorGroupId'] not in vendor_groups:
                errors.append(f"Row {row.line}: VendorGroup '{row['VendorGroupId']}' missing")
        
        return errors
```

### Why NOT X++

| X++ API Approach | OData Approach |
|---|---|
| Requires deployment to F&O environment | No deployment needed |
| Custom code to maintain through upgrades | Uses standard OData endpoints (Microsoft-maintained) |
| Runs inside F&O process (competes with users) | Runs externally (no impact on F&O performance) |
| Needs developer for changes | Configuration-driven (add new entities to validation rules) |
| Version-specific compilation | Version-independent REST calls |
| Only accessible from within F&O or via custom service | Accessible from any HTTP client |

The ONLY reason to write X++ would be if you needed to validate against business logic that ISN'T exposed through data entities (e.g., a custom validation method on a table). For standard reference data validation, OData is better in every dimension.
