# Entity Dependency Maps — Foreign Key Validation per Master Data Entity

For each master data entity, this maps EVERY foreign key field to the OData endpoint needed to validate it. Use this to build the pre-validation pull list — before importing an entity, pull all reference data listed in its dependency map.

## VendorsV2

| Source Field | Validates Against | OData Endpoint | Filter |
|---|---|---|---|
| `VendorGroupId` | Vendor groups | `/data/VendorGroups` | `$select=VendorGroupId` |
| `PaymentTermsName` | Payment terms | `/data/PaymentTerms` | `$select=PaymentTermsName` |
| `PaymentMethod` | Vendor payment methods | `/data/VendorPaymentMethods` | `$filter=dataAreaId eq '{company}'` |
| `CashDiscountCode` | Cash discounts | `/data/CashDiscounts` | `$select=CashDiscountCode` |
| `TaxGroup` | Sales tax groups | `/data/SalesTaxGroups` | `$select=TaxGroup` |
| `WithholdingTaxGroupId` | Withholding tax groups | `/data/WithholdingTaxGroups` | `$select=WithholdingTaxGroup` |
| `CurrencyCode` | Currencies | `/data/Currencies` | `$select=CurrencyCode` |
| `DefaultDimensionDisplayValue` | Financial dimension values | `/data/FinancialDimensionValues` | Parse per active format, query per dimension |
| `AddressCountryRegionId` | Countries | `/data/AddressCountryRegions` | `$select=CountryRegionId` |
| `AddressState` | States | `/data/AddressStates` | `$filter=CountryRegionId eq '{country}'` |
| `AddressCity` | Cities | `/data/AddressCities` | `$filter=StateId eq '{state}'` |
| `AddressZipCode` | Postal codes | `/data/AddressZipCodes` | `$filter=City eq '{city}'` |
| `LanguageId` | Languages | `/data/Languages` | `$select=LanguageId` |
| `DeliveryTermsId` | Terms of delivery | `/data/TermsOfDelivery` | `$select=TermsOfDeliveryId` |
| `DeliveryModeId` | Modes of delivery | `/data/ModesOfDelivery` | `$select=ModeOfDeliveryId` |
| `VendorAccountNumber` | *(self — duplicate check)* | `/data/VendorsV2` | `$filter=VendorAccountNumber eq '{value}'` |
| `InvoiceVendorAccountNumber` | Existing vendors (invoice account) | `/data/VendorsV2` | If set, must reference existing vendor |
| `LineOfBusinessId` | Line of business | `/data/LinesOfBusiness` | `$select=LineOfBusinessId` |

**Total reference pulls needed: 15 endpoints**

---

## CustomersV3

| Source Field | Validates Against | OData Endpoint | Filter |
|---|---|---|---|
| `CustomerGroupId` | Customer groups | `/data/CustomerGroups` | `$select=CustomerGroupId` |
| `PaymentTermsName` | Payment terms | `/data/PaymentTerms` | `$select=PaymentTermsName` |
| `PaymentMethod` | Customer payment methods | `/data/CustomerPaymentMethods` | `$filter=dataAreaId eq '{company}'` |
| `CashDiscountCode` | Cash discounts | `/data/CashDiscounts` | `$select=CashDiscountCode` |
| `TaxGroup` | Sales tax groups | `/data/SalesTaxGroups` | `$select=TaxGroup` |
| `CurrencyCode` | Currencies | `/data/Currencies` | `$select=CurrencyCode` |
| `DefaultDimensionDisplayValue` | Financial dimension values | `/data/FinancialDimensionValues` | Parse per active format |
| `AddressCountryRegionId` | Countries | `/data/AddressCountryRegions` | `$select=CountryRegionId` |
| `AddressState` | States | `/data/AddressStates` | `$filter=CountryRegionId eq '{country}'` |
| `AddressCity` | Cities | `/data/AddressCities` | `$filter=StateId eq '{state}'` |
| `AddressZipCode` | Postal codes | `/data/AddressZipCodes` | `$filter=City eq '{city}'` |
| `LanguageId` | Languages | `/data/Languages` | `$select=LanguageId` |
| `DeliveryTermsId` | Terms of delivery | `/data/TermsOfDelivery` | `$select=TermsOfDeliveryId` |
| `DeliveryModeId` | Modes of delivery | `/data/ModesOfDelivery` | `$select=ModeOfDeliveryId` |
| `SiteId` | Sites | `/data/Sites` | `$select=SiteId` |
| `WarehouseId` | Warehouses | `/data/Warehouses` | `$filter=SiteId eq '{site}'` |
| `CustomerAccount` | *(self — duplicate check)* | `/data/CustomersV3` | `$filter=CustomerAccount eq '{value}'` |
| `InvoiceCustomerAccountNumber` | Existing customers (invoice account) | `/data/CustomersV3` | If set, must reference existing customer |
| `CollectionLetterSequenceId` | Collection letter sequences | `/data/CollectionLetterSequences` | `$select=SequenceId` |
| `StatisticsGroupId` | Statistics group | `/data/CustomerStatisticsGroups` | `$select=CustomerStatisticsGroupId` |
| `SalesTaxExemptNumber` | Tax exempt numbers | `/data/TaxExemptNumbers` | `$filter=CountryRegionId eq '{country}'` |

**Total reference pulls needed: 18 endpoints**

---

## ReleasedProductsV2

| Source Field | Validates Against | OData Endpoint | Filter |
|---|---|---|---|
| `ItemModelGroupId` | Item model groups | `/data/ItemModelGroups` | `$select=ModelGroupId` |
| `ItemGroupId` | Item groups | `/data/ItemGroups` | `$select=ItemGroupId` |
| `StorageDimensionGroupName` | Storage dim groups | `/data/StorageDimensionGroups` | `$select=StorageDimensionGroupName` |
| `TrackingDimensionGroupName` | Tracking dim groups | `/data/TrackingDimensionGroups` | `$select=TrackingDimensionGroupName` |
| `ProductDimensionGroupName` | Product dim groups | `/data/ProductDimensionGroups` | `$select=ProductDimensionGroupName` |
| `BOMUnitSymbol` | Units | `/data/Units` | `$select=UnitSymbol` |
| `PurchaseUnitSymbol` | Units | `/data/Units` | *(same pull)* |
| `SalesUnitSymbol` | Units | `/data/Units` | *(same pull)* |
| `InventoryUnitSymbol` | Units | `/data/Units` | *(same pull)* |
| `DefaultDimensionDisplayValue` | Financial dimension values | `/data/FinancialDimensionValues` | Parse per format |
| `SalesTaxItemGroupId` | Item sales tax groups | `/data/ItemSalesTaxGroups` | `$select=TaxItemGroup` |
| `PurchaseTaxItemGroupId` | Item sales tax groups | `/data/ItemSalesTaxGroups` | *(same pull)* |
| `BuyerGroupId` | Buyer groups | `/data/BuyerGroups` | `$select=BuyerGroupId` |
| `ProductCategoryName` | Product categories | `/data/ProductCategories` | `$select=CategoryName` |
| `DefaultOrderSettingsSiteId` | Sites | `/data/Sites` | `$select=SiteId` |
| `DefaultOrderSettingsWarehouseId` | Warehouses | `/data/Warehouses` | `$filter=SiteId eq '{site}'` |
| `ItemNumber` | *(self — duplicate check)* | `/data/ReleasedProductsV2` | `$filter=ItemNumber eq '{value}'` |
| `VendorAccountNumber` | Vendors (default vendor) | `/data/VendorsV2` | If set, must exist |
| `CostGroupId` | Cost groups | `/data/CostGroups` | `$select=CostGroupId` |

**Total reference pulls needed: 16 endpoints (deduplicated)**

---

## BillOfMaterialsLines

| Source Field | Validates Against | OData Endpoint | Filter |
|---|---|---|---|
| `BOMId` | BOM headers | `/data/BillOfMaterialsHeaders` | `$filter=BOMId eq '{value}'` |
| `ItemNumber` | Released products | `/data/ReleasedProductsV2` | `$filter=ItemNumber eq '{value}'` |
| `UnitSymbol` | Units | `/data/Units` | `$select=UnitSymbol` |
| `WarehouseId` | Warehouses | `/data/Warehouses` | `$select=WarehouseId` |
| `SiteId` | Sites | `/data/Sites` | `$select=SiteId` |
| `CostGroupId` | Cost groups | `/data/CostGroups` | `$select=CostGroupId` |

---

## LedgerJournalEntity (General Journal)

| Source Field | Validates Against | OData Endpoint | Filter |
|---|---|---|---|
| `JOURNALNAME` | Journal names | `/data/LedgerJournalNames` | `$filter=JournalName eq '{value}'` |
| `CURRENCYCODE` | Currencies | `/data/Currencies` | `$select=CurrencyCode` |
| `ACCOUNTDISPLAYVALUE` (when ACCOUNTTYPE=Ledger) | Main accounts + dimensions | Parse per ledger dimension format: |
| — Main account segment | Main accounts | `/data/MainAccounts` | `$select=MainAccountId` |
| — Each dimension segment | Dimension values | `/data/FinancialDimensionValues` | `$filter=DimensionName eq '{dim}'` |
| `ACCOUNTDISPLAYVALUE` (when ACCOUNTTYPE=Vendor) | Vendors | `/data/VendorsV2` | `$filter=VendorAccountNumber eq '{value}'` |
| `ACCOUNTDISPLAYVALUE` (when ACCOUNTTYPE=Customer) | Customers | `/data/CustomersV3` | `$filter=CustomerAccount eq '{value}'` |
| `ACCOUNTDISPLAYVALUE` (when ACCOUNTTYPE=Bank) | Bank accounts | `/data/BankAccountsV2` | `$filter=BankAccountId eq '{value}'` |
| `OFFSETACCOUNTDISPLAYVALUE` | *(same rules as ACCOUNTDISPLAYVALUE based on OFFSETACCOUNTTYPE)* | | |
| `TRANSDATE` | Fiscal periods | `/data/FiscalCalendarPeriods` | Check period status = Open |

---

## FixedAssets

| Source Field | Validates Against | OData Endpoint | Filter |
|---|---|---|---|
| `FixedAssetGroupId` | FA groups | `/data/FixedAssetGroups` | `$select=GroupId` |
| `Location` | Locations | `/data/FixedAssetLocations` | `$select=Location` |
| `WorkerResponsible` | Workers | `/data/Workers` | `$filter=PersonnelNumber eq '{value}'` |
| `DepartmentDimension` | Operating units | `/data/OperatingUnits` | `$filter=OperatingUnitType eq 'Department'` |
| `FixedAssetNumber` | *(self — duplicate check)* | `/data/FixedAssets` | `$filter=FixedAssetNumber eq '{value}'` |

---

## VendorBankAccounts

| Source Field | Validates Against | OData Endpoint | Filter |
|---|---|---|---|
| `VendorAccountNumber` | Vendors | `/data/VendorsV2` | Must exist (parent) |
| `CurrencyCode` | Currencies | `/data/Currencies` | `$select=CurrencyCode` |
| `BankGroupId` | Bank groups | `/data/BankGroups` | `$select=BankGroupId` |
| `CountryRegionId` | Countries | `/data/AddressCountryRegions` | For bank address |

---

## CustomerBankAccounts

| Source Field | Validates Against | OData Endpoint | Filter |
|---|---|---|---|
| `CustomerAccountNumber` | Customers | `/data/CustomersV3` | Must exist (parent) |
| `CurrencyCode` | Currencies | `/data/Currencies` | `$select=CurrencyCode` |
| `BankGroupId` | Bank groups | `/data/BankGroups` | `$select=BankGroupId` |

---

## TradeAgreementJournalLines (Pricing)

| Source Field | Validates Against | OData Endpoint | Filter |
|---|---|---|---|
| `JournalNumber` | Trade agreement headers | `/data/TradeAgreementJournalHeaders` | Must exist (parent) |
| `ItemNumber` | Released products | `/data/ReleasedProductsV2` | `$filter=ItemNumber eq '{value}'` |
| `AccountCode=Table` → `AccountRelation` | Customer or vendor | `/data/CustomersV3` or `/data/VendorsV2` | Depends on agreement type |
| `CurrencyCode` | Currencies | `/data/Currencies` | |
| `UnitSymbol` | Units | `/data/Units` | |
| `SiteId` | Sites | `/data/Sites` | If site-specific price |
| `WarehouseId` | Warehouses | `/data/Warehouses` | If warehouse-specific price |

---

## Pre-Validation Pull Strategy

### Step 1: Analyze Source File
For a given source file (e.g., customer CSV):
1. Parse column headers → identify which entity is being imported
2. Look up this entity's dependency map (above)
3. Identify which reference endpoints to query

### Step 2: Pull Reference Data (Batch)
```
For each endpoint in the dependency map:
    If data already cached and cache not expired:
        Use cached data
    Else:
        GET /data/{endpoint}?$select={keyField}
        Cache the result set as a HashSet<string>
```

**Optimization:** Many entities share the same references. Pull once:
- Currencies → used by Vendors, Customers, Products, Journals, Bank accounts
- Payment terms → used by Vendors AND Customers
- Tax groups → used by Vendors AND Customers
- Units → used by Products (4 unit fields all validated against same set)
- Sites/Warehouses → used by Products, Customers, BOMs, Pricing

### Step 3: Validate Source File
```
For each row in source:
    For each foreign key field:
        If value is not blank AND value not in reference HashSet:
            Add to error list: (row#, field, value, "not found in {entity}")
    Check duplicate primary key within file
    Check primary key doesn't already exist in target (optional)
```

### Step 4: Generate Report
```
Pre-Validation Report: CustomersV3
═══════════════════════════════════
Source file: customers_migration.csv
Records: 12,450
Target environment: contoso-uat.operations.dynamics.com
Date: 2026-03-18

Reference Data Checks:
  ✅ CustomerGroups: 8 values, all found
  ✅ PaymentTerms: 6 values, all found
  ✅ Currencies: 3 values, all found
  ✅ SalesTaxGroups: 5 values, all found
  ⚠️  DeliveryModes: 4 values, 1 missing → "AIR-EXPRESS"
  ❌ States: 847 values, 3 invalid → "XX", "ZZ", "N/A"
  ✅ Sites: 4 values, all found
  ✅ Warehouses: 12 values, all found

Duplicate Checks:
  ✅ No duplicate CustomerAccount values in file
  ⚠️  23 CustomerAccount values already exist in target

Summary:
  Blockers: 1 (invalid states on 3 rows)
  Warnings: 2 (missing delivery mode on 47 rows, 23 existing customers)
  Ready: NO — fix states and decide on delivery mode before import
```

## Automating Dependency Map Discovery

If you want to dynamically discover foreign keys (instead of hardcoding the maps above), you can use the MCP server:

```
1. data_find_entity_type("{EntityName}") → discover the entity
2. data_get_entity_metadata("{EntityName}") → returns full schema including:
   - Field names, types, required/optional
   - Navigation properties (foreign key relationships)
   - Referenced entity for each FK
3. For each navigation property → build the validation endpoint
```

This way the agent can handle ANY entity, even custom ones, without pre-mapped dependency tables. But for the standard entities, the hardcoded maps above are faster and more reliable.
