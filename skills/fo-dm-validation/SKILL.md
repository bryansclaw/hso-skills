---
name: fo-dm-validation
description: >
  Validate D365 F&O data migration at every stage — pre-migration checks,
  staging table validation, post-import verification, cross-module reconciliation,
  duplicate detection, referential integrity checks, and automated account
  reconciliation. Use when: validating any DMF import, running pre-migration
  quality checks, performing post-import reconciliation, or preparing for
  go/no-go decisions. This skill is used continuously throughout migration.
metadata:
  platform:
    category: "fo-data-migration"
    riskLevel: "low"
    configSequence: 0
    configLevel: "cross-cutting"
    requires:
      products: ["F&O"]
    bpcProcess: "99 - Administer to Operate"
---

# Data Migration Validation Framework

## Validation Stages

```
PRE-MIGRATION ──→ STAGING ──→ POST-IMPORT ──→ RECONCILIATION ──→ GO/NO-GO
```

## Stage 0: Auto-Discover Entity Dependencies

**Before validating anything, discover WHAT to validate.** Use OData $metadata to automatically extract foreign key relationships for any entity — standard or custom.

### How It Works
Every FK on a D365 entity is declared as a `NavigationProperty` with a `ReferentialConstraint` in OData $metadata. The constraint tells you: this field on MY entity references that field on THAT entity.

### Agent Workflow
```
1. GET /data/$metadata#CustomersV3 → parse XML
2. Find all <NavigationProperty> with <ReferentialConstraint>
3. For each:
   source_field = constraint.Property       (e.g., "CustomerGroupId")
   target_entity = nav_prop.Type            (e.g., "CustomerGroupEntity") 
   target_endpoint = pluralize(Type)        (e.g., "/data/CustomerGroups")
   target_key = constraint.ReferencedProperty
4. Build validation rule set automatically
```

### MCP Alternative
```
data_get_entity_metadata("CustomersV3") → returns schema with navigation 
properties → agent extracts FK list without parsing XML
```

### Special Cases (not discovered via NavigationProperty)
- **DefaultDimensionDisplayValue** — compound string, parse per active financial dimension format, validate each segment against `/data/FinancialDimensionValues?$filter=DimensionName eq '{dim}'`
- **Address fields** — hierarchy chain: Country → State → City → Zip (each validated WHERE parent = validated parent value)
- **Enum fields** — validate against `<EnumType>` definitions in $metadata

### Reference Files
- See `references/discover-entity-schema.md` for full implementation details
- See `references/entity-dependency-maps.md` for pre-built maps (9 major entities)
- See `references/odata-reference-endpoints.md` for 70+ endpoint catalog
- See `references/pre-validation-service.md` for architecture and caching strategy

---

## Stage 1: Pre-Migration Validation (Before Import)

### Source Data Quality Checks
Run against source data files BEFORE importing:

| Check | Method | Failure Action |
|---|---|---|
| **Mandatory fields populated** | Scan all rows for null/empty in required columns | Reject file, report specific rows |
| **Data types correct** | Dates are dates, numbers are numbers, no formulas in Excel | Convert to values, fix formats |
| **Primary key uniqueness** | No duplicate keys within the file | Deduplicate, merge, or flag for review |
| **Foreign key existence** | All referenced codes (groups, terms, currencies) exist in target | Report missing references, create first |
| **Length validation** | Text fields within D365 max length | Truncate or flag for review |
| **Character encoding** | UTF-8 throughout, no special characters in key fields | Convert encoding |
| **Address validation** | Country→State→City hierarchy valid | Fix address data against GAB |
| **Dimension format match** | Dimension strings match active format in target | Reformat to match |
| **Excel formula check** | No cells contain formulas (copy-paste as values) | Convert to values |
| **Confidential label check** | Excel files not labeled "Confidential" | Remove label |

### Target Readiness Checks
Run via MCP data tools against the TARGET environment:

```python
# Pseudo-code for pre-migration validation agent
async def validate_target_readiness(entity_type, source_data):
    
    # 1. Verify entity exists and is accessible
    entity = await mcp.data_find_entity_type(entity_type)
    if not entity:
        return Error(f"Entity {entity_type} not found in target")
    
    # 2. Get entity metadata (required fields)
    metadata = await mcp.data_get_entity_metadata(entity)
    required_fields = [f for f in metadata.fields if f.required]
    
    # 3. For each foreign key, verify reference data exists
    for fk in source_data.foreign_keys:
        existing = await mcp.data_find_entities(fk.entity, filter=fk.values)
        missing = fk.values - existing.values
        if missing:
            report.add_blocker(f"Missing {fk.entity}: {missing}")
    
    # 4. Verify financial dimension config is active
    # (Cannot query via data tools — must check via form tool or known state)
    
    # 5. Check for existing records (duplicates)
    for key in source_data.primary_keys:
        existing = await mcp.data_find_entities(entity_type, filter=key)
        if existing:
            report.add_warning(f"Record already exists: {key}")
    
    return report
```

## Stage 2: Staging Table Validation (During Import)

### DMF Staging Validation
After import to staging, before target push:

| Check | How | Notes |
|---|---|---|
| **Record count** | Staging count = source file count | Any mismatch = parsing error |
| **Mapping accuracy** | View staging data, spot-check field mapping | Data management > View staging data |
| **Enum values** | All enum/option set values resolve | Common failure point |
| **Date parsing** | Dates parsed correctly (not shifted by timezone) | Compare sample dates |
| **Amount precision** | Decimal places preserved | Currency amounts need correct precision |
| **Dimension resolution** | Dimension strings parse per active format | Check ACCOUNTDISPLAYVALUE format |

### Enable "Preview" Before Push
- In DMF project, use "Preview rows" to inspect staging data before pushing to target
- This catches mapping issues before they become data errors

## Stage 3: Post-Import Verification (After Import)

### Record Count Validation
```
For each entity imported:
1. Source file rows:        ____
2. Staging table rows:      ____ (should = source)
3. Target table rows:       ____ (should = staging - errors)
4. Error count:             ____ (should be 0)
5. Skipped (duplicates):    ____ (should be expected or 0)
```

### Spot-Check Validation
For a random sample of records (e.g., 10%):
- Verify all fields match source data
- Check addresses render correctly
- Verify financial dimensions display properly
- Confirm related records created (e.g., vendor → party record)

### DMF Execution Log Review
- **Menu:** Data management > Job history > [job] > Execution details
- Check per-entity status: Succeeded / Warning / Error
- Download error files for any failures
- Review error messages against known error catalog (see `fo-data-migration` skill)

## Stage 4: Cross-Module Reconciliation

### Configuration Reconciliation (Stage 1 imports)
| Check | Query | Expected |
|---|---|---|
| Main accounts count | `data_find_entities("MainAccounts", $count)` | = source CoA count |
| Tax codes active | `data_find_entities("SalesTaxCodes")` | All expected codes present |
| Posting profiles complete | Manual review or form tool inspection | No blank account assignments |
| Number sequences active | Check status per module | All = Active/Continuous |

### Master Data Reconciliation (Stage 2 imports)
| Check | Query | Expected |
|---|---|---|
| Vendor count | `data_find_entities("VendorsV2", $count)` | = source vendor count |
| Customer count | `data_find_entities("CustomersV3", $count)` | = source customer count |
| Product count | `data_find_entities("ReleasedProductsV2", $count)` | = source product count |
| FA count | `data_find_entities("FixedAssets", $count)` | = source asset count |
| BOM count | `data_find_entities("BillOfMaterialsHeaders", $count)` | = source BOM count |

### Referential Integrity Checks
```
# Every vendor → valid vendor group
SELECT vendor WHERE vendorGroup NOT IN (SELECT vendorGroupId FROM VendorGroups)

# Every customer → valid customer group
SELECT customer WHERE customerGroup NOT IN (SELECT customerGroupId FROM CustomerGroups)

# Every product → valid item model group + item group
SELECT product WHERE itemModelGroupId NOT IN (SELECT id FROM ItemModelGroups)

# Every posting profile → valid GL main account
SELECT postingProfile WHERE mainAccount NOT IN (SELECT accountNum FROM MainAccounts)
```

Via MCP: Use `data_find_entities` with OData `$filter` to find orphaned references.

### Financial Reconciliation (Stage 3 imports)
| Check | Report A | Report B | Must Equal |
|---|---|---|---|
| AP balance | Vendor aging report (total) | GL trial balance: AP control | Exactly |
| AR balance | Customer aging report (total) | GL trial balance: AR control | Exactly |
| Inventory value | Inventory valuation report | GL trial balance: Inventory control | Exactly |
| FA NBV | FA net book value report | GL trial balance: FA accounts | Exactly |
| Bank balance | Bank account balance | GL trial balance: Bank accounts | Exactly |
| Trial balance | GL trial balance total | Debits - Credits | = $0 |
| Clearing account | GL trial balance: clearing | Balance | = $0 |

### Automated Account Reconciliation (v10.0.44+)
- **Menu:** General ledger > Periodic tasks > Account reconciliation
- Run after EVERY Stage 3 import
- Reconciles GL with: AP, AR, Tax, Bank subledgers automatically
- Reports mismatches with drill-down to specific transactions

## Stage 5: Go/No-Go Validation

### Go/No-Go Checklist
| Criteria | Status | Threshold |
|---|---|---|
| All Stage 1 config imported and validated | Pass/Fail | 100% pass required |
| All Stage 2 master data imported | Pass/Fail | 100% critical entities |
| All Stage 3 balances imported and reconciled | Pass/Fail | 100% reconciliation |
| AP aging = GL AP control | Pass/Fail | Exact match |
| AR aging = GL AR control | Pass/Fail | Exact match |
| Inventory = GL inventory | Pass/Fail | Exact match |
| FA NBV = GL FA | Pass/Fail | Exact match |
| Bank = GL bank | Pass/Fail | Exact match |
| GL trial balance balanced | Pass/Fail | Debits = Credits |
| Clearing account = $0 | Pass/Fail | Exact zero |
| Key user spot-checks approved | Pass/Fail | Sign-off obtained |
| Cutover completed within time window | Pass/Fail | ≤ planned duration |
| No critical DMF errors outstanding | Pass/Fail | 0 unresolved errors |

### Decision
- **All Pass → GO** — proceed to go-live
- **Any Fail → NO-GO** — diagnose, fix, re-run mock cutover

## Error Tracking Template

| # | Entity | Error Message | Rows Affected | Root Cause | Resolution | Status |
|---|---|---|---|---|---|---|
| 1 | VendorsV2 | "Vendor group not found: TRADE-20" | 47 | Group missing from Stage 1 | Create TRADE-20 group, re-import | Resolved |
| 2 | CustomersV3 | "Task count > 1 not supported" | All | Multi-thread attempted | Set task count = 1 | Resolved |
| 3 | LedgerJournal | "Dimension format error" | All | No ledger dimension format | Configure in Financial dim config | Resolved |

## MCP-Powered Validation Workflow

### Pre-Import (Auto-Discovered FK Validation)
```
1. Agent receives: "Pre-validate vendor migration file"

2. Discover dependencies:
   data_get_entity_metadata("VendorsV2") → extract NavigationProperties
   → Builds FK map: VendorGroupId→VendorGroups, PaymentTermsName→PaymentTerms,
     TaxGroup→SalesTaxGroups, DeliveryModeId→ModesOfDelivery, 
     CurrencyCode→Currencies, DeliveryTermsId→TermsOfDelivery, etc.

3. Pull reference data (cached):
   For each FK target entity:
     data_find_entities("{TargetEntity}", $select={keyField}) → cache as HashSet

4. Parse source file:
   Extract unique values per FK column
   Compare against cached reference sets
   Flag: missing references, duplicates, null mandatory fields

5. Report to consultant:
   "Pre-validation of 4,230 vendors:
    ✅ VendorGroups: 12 values, all found
    ✅ PaymentTerms: 6 values, all found
    ✅ Currencies: 3 values (USD, EUR, GBP), all found
    ⚠️  DeliveryModes: 4 values, 1 missing → 'AIR-EXPRESS' (47 rows affected)
    ❌ SalesTaxGroups: 5 values, 1 missing → 'EXEMPT-EXPORT' (203 rows affected)
    ✅ No duplicate VendorAccountNumbers in file
    ⚠️  8 VendorAccountNumbers already exist in target
    
    Blockers: 1 (EXEMPT-EXPORT tax group — create it or remap 203 vendors)
    Warnings: 2 (AIR-EXPRESS delivery mode, 8 existing vendors)
    
    Create missing reference data? [Yes / Show affected rows / Skip]"
```

### Post-Import Verification
```
1. Agent receives: "Verify vendor migration"

2. Count check:
   data_find_entities("VendorsV2", $count) → 4,230 (expected 4,230) ✅

3. Referential integrity spot-check:
   data_find_entities("VendorsV2", $filter=VendorGroupId eq 'TRADE-20') → 847 vendors ✅
   data_find_entities("VendorsV2", $filter=DefaultDimension eq null) → 23 missing ⚠️
   data_find_entities("VendorsV2", $filter=TaxGroup eq null) → 0 ✅

4. Report:
   "Post-import verification: 4,230 vendors
    ✅ Count matches source
    ✅ All vendor groups assigned correctly
    ✅ All tax groups assigned
    ⚠️  23 vendors missing default dimensions — review?
    ⚠️  5 vendors missing bank accounts — expected?"
```

## Related Skills
- **Used with:** `fo-dm-configuration`, `fo-dm-master-data`, `fo-dm-opening-balances`
- **Reconciliation:** `fo-gl-posting-profiles`, `fo-period-close`
- **MCP tools:** `fo-integration-patterns`
