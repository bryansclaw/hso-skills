# OData Reference Data Endpoints — Complete Catalog

Use these to pull valid reference values from a D365 F&O environment for pre-migration validation. No X++, no DMF — just authenticated HTTP GET requests.

## Authentication

```
POST https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id={appId}
&client_secret={secret}
&scope=https://{environment}.operations.dynamics.com/.default
```

All queries below use:
```
GET https://{environment}.operations.dynamics.com/data/{EntityName}?{query}
Authorization: Bearer {token}
Accept: application/json
```

## Query Patterns

### Get all values (small reference tables)
```
/data/VendorGroups?$select=VendorGroupId,VendorGroupName
```

### Count only (check if entity has data)
```
/data/VendorsV2?$count=true&$top=0
```

### Check if specific value exists
```
/data/VendorGroups?$filter=VendorGroupId eq 'TRADE-20'&$select=VendorGroupId
```

### Get values for a specific company
```
/data/VendorGroups?$filter=dataAreaId eq 'USMF'&$select=VendorGroupId
```

### Batch multiple checks (JSON batch)
```
POST https://{environment}.operations.dynamics.com/data/$batch
Content-Type: multipart/mixed; boundary=batch_boundary

--batch_boundary
Content-Type: application/http
GET VendorGroups?$select=VendorGroupId HTTP/1.1

--batch_boundary
Content-Type: application/http
GET PaymentTerms?$select=PaymentTermsName HTTP/1.1

--batch_boundary--
```

## Complete Endpoint Catalog

### System & Foundation

| What to Validate | Endpoint | Key Field(s) | $select | Notes |
|---|---|---|---|---|
| Legal entities | `/data/LegalEntities` | `DataArea` | `DataArea,Name` | Company codes |
| Currencies | `/data/Currencies` | `CurrencyCode` | `CurrencyCode,Name` | ISO codes |
| Exchange rate types | `/data/ExchangeRateTypes` | `ExchangeRateTypeName` | `ExchangeRateTypeName` | |
| Countries/regions | `/data/AddressCountryRegions` | `CountryRegionId` | `CountryRegionId,ShortName` | ISO country codes |
| States/provinces | `/data/AddressStates` | `StateId` | `StateId,CountryRegionId,Name` | Filter by country |
| Cities | `/data/AddressCities` | `Name` | `Name,StateId,CountryRegionId` | Filter by state |
| Postal codes | `/data/AddressZipCodes` | `ZipCode` | `ZipCode,City,StateId,CountryRegionId` | |
| Units of measure | `/data/Units` | `UnitSymbol` | `UnitSymbol,UnitClass` | ea, kg, lb, etc. |
| Number sequences | `/data/NumberSequences` | `NumberSequenceCode` | `NumberSequenceCode,NumberSequenceScope` | Verify active |

### General Ledger

| What to Validate | Endpoint | Key Field(s) | $select | Notes |
|---|---|---|---|---|
| Chart of accounts | `/data/ChartOfAccounts` | `ChartOfAccounts` | `ChartOfAccounts,Description` | |
| Main accounts | `/data/MainAccounts` | `MainAccountId` | `MainAccountId,Name,MainAccountType` | Can be large (1K-5K) |
| Main account categories | `/data/MainAccountCategories` | `MainAccountCategoryId` | `MainAccountCategoryId,Description` | |
| Financial dimensions | `/data/FinancialDimensions` | `DimensionName` | `DimensionName,DimensionType` | Definition only |
| Financial dimension values | `/data/FinancialDimensionValues` | `DimensionValue` | `DimensionValue,DimensionName,Description` | Filter by DimensionName |
| Journal names | `/data/LedgerJournalNames` | `JournalName` | `JournalName,JournalType,Description` | Filter by company |
| Fiscal calendar periods | `/data/FiscalCalendarPeriods` | `CalendarId,StartDate` | `CalendarId,StartDate,EndDate,Status` | Check period is Open |

**Dimension values by specific dimension:**
```
/data/FinancialDimensionValues?$filter=DimensionName eq 'Department'&$select=DimensionValue,Description
/data/FinancialDimensionValues?$filter=DimensionName eq 'CostCenter'&$select=DimensionValue,Description
```

### Accounts Payable

| What to Validate | Endpoint | Key Field(s) | $select | Notes |
|---|---|---|---|---|
| Vendor groups | `/data/VendorGroups` | `VendorGroupId` | `VendorGroupId,VendorGroupName` | |
| Vendor posting profiles | `/data/VendorPostingProfiles` | `PostingProfile` | `PostingProfile,Description` | |
| Payment terms | `/data/PaymentTerms` | `PaymentTermsName` | `PaymentTermsName,Description` | Shared with AR |
| Cash discounts | `/data/CashDiscounts` | `CashDiscountCode` | `CashDiscountCode,DiscountPercentage` | |
| Vendor payment methods | `/data/VendorPaymentMethods` | `Name` | `Name,Description,dataAreaId` | Per company |
| Existing vendors | `/data/VendorsV2` | `VendorAccountNumber` | `VendorAccountNumber,VendorName` | Duplicate check |

### Accounts Receivable

| What to Validate | Endpoint | Key Field(s) | $select | Notes |
|---|---|---|---|---|
| Customer groups | `/data/CustomerGroups` | `CustomerGroupId` | `CustomerGroupId,Description` | |
| Customer posting profiles | `/data/CustomerPostingProfiles` | `PostingProfile` | `PostingProfile,Description` | |
| Customer payment methods | `/data/CustomerPaymentMethods` | `Name` | `Name,Description,dataAreaId` | Per company |
| Interest codes | `/data/InterestCodes` | `InterestCode` | `InterestCode,Description` | |
| Existing customers | `/data/CustomersV3` | `CustomerAccount` | `CustomerAccount,CustomerName` | Duplicate check |

### Tax

| What to Validate | Endpoint | Key Field(s) | $select | Notes |
|---|---|---|---|---|
| Sales tax codes | `/data/SalesTaxCodes` | `TaxCode` | `TaxCode,TaxName` | |
| Sales tax groups | `/data/SalesTaxGroups` | `TaxGroup` | `TaxGroup,TaxGroupName` | |
| Item sales tax groups | `/data/ItemSalesTaxGroups` | `TaxItemGroup` | `TaxItemGroup,Name` | |
| Tax authorities | `/data/SalesTaxAuthorities` | `TaxAuthority` | `TaxAuthority,Name` | |
| Tax exempt numbers | `/data/TaxExemptNumbers` | `TaxExemptNumber` | `TaxExemptNumber,CountryRegionId` | |
| Withholding tax codes | `/data/WithholdingTaxCodes` | `WithholdingTaxCode` | `WithholdingTaxCode,WithholdingTaxName` | |
| Withholding tax groups | `/data/WithholdingTaxGroups` | `WithholdingTaxGroup` | `WithholdingTaxGroup,Description` | |

### Bank

| What to Validate | Endpoint | Key Field(s) | $select | Notes |
|---|---|---|---|---|
| Bank groups | `/data/BankGroups` | `BankGroupId` | `BankGroupId,Name` | |
| Bank accounts | `/data/BankAccountsV2` | `BankAccountId` | `BankAccountId,Name,CurrencyCode,dataAreaId` | Per company |
| Bank transaction types | `/data/BankTransactionTypes` | `BankTransactionType` | `BankTransactionType,Name` | |

### Inventory & Products

| What to Validate | Endpoint | Key Field(s) | $select | Notes |
|---|---|---|---|---|
| Item model groups | `/data/ItemModelGroups` | `ModelGroupId` | `ModelGroupId,ModelGroupName` | |
| Item groups | `/data/ItemGroups` | `ItemGroupId` | `ItemGroupId,Name` | |
| Storage dim groups | `/data/StorageDimensionGroups` | `StorageDimensionGroupName` | `StorageDimensionGroupName` | |
| Tracking dim groups | `/data/TrackingDimensionGroups` | `TrackingDimensionGroupName` | `TrackingDimensionGroupName` | |
| Sites | `/data/Sites` | `SiteId` | `SiteId,SiteName` | |
| Warehouses | `/data/Warehouses` | `WarehouseId` | `WarehouseId,WarehouseName,SiteId` | Filter by site |
| Product categories | `/data/ProductCategories` | `CategoryName` | `CategoryName,CategoryHierarchyName` | |
| Released products | `/data/ReleasedProductsV2` | `ItemNumber` | `ItemNumber,ProductName,dataAreaId` | Can be huge — use $filter |
| Unit conversions | `/data/UnitConversions` | `FromUnit,ToUnit` | `FromUnit,ToUnit,Factor` | |

**Check if specific product exists:**
```
/data/ReleasedProductsV2?$filter=ItemNumber eq 'ITEM-001' and dataAreaId eq 'USMF'&$select=ItemNumber
```

### Fixed Assets

| What to Validate | Endpoint | Key Field(s) | $select | Notes |
|---|---|---|---|---|
| FA groups | `/data/FixedAssetGroups` | `GroupId` | `GroupId,GroupName` | |
| Depreciation profiles | `/data/DepreciationProfiles` | `DepreciationProfileId` | `DepreciationProfileId,Name` | |
| Existing assets | `/data/FixedAssets` | `FixedAssetNumber` | `FixedAssetNumber,Name,dataAreaId` | Duplicate check |

### Production

| What to Validate | Endpoint | Key Field(s) | $select | Notes |
|---|---|---|---|---|
| Route groups | `/data/RouteGroups` | `RouteGroupId` | `RouteGroupId,Name` | |
| Operations | `/data/Operations` | `OperationId` | `OperationId,Name` | |
| Production groups | `/data/ProductionGroups` | `ProductionGroupId` | `ProductionGroupId,Name` | |
| Resources | `/data/Resources` | `ResourceId` | `ResourceId,Name,ResourceType` | |
| BOM headers | `/data/BillOfMaterialsHeaders` | `BOMId` | `BOMId,BOMName,ItemNumber` | |

### Procurement & Sales

| What to Validate | Endpoint | Key Field(s) | $select | Notes |
|---|---|---|---|---|
| Procurement categories | `/data/ProcurementCategories` | `CategoryName` | `CategoryName` | |
| Terms of delivery | `/data/TermsOfDelivery` | `TermsOfDeliveryId` | `TermsOfDeliveryId,Description` | Incoterms |
| Modes of delivery | `/data/ModesOfDelivery` | `ModeOfDeliveryId` | `ModeOfDeliveryId,Description` | |
| Price groups | `/data/PriceGroups` | `PriceGroupId` | `PriceGroupId,Name` | |

## Pagination for Large Entities

For entities with >5000 records (like ReleasedProductsV2), OData returns paginated results:

```json
{
  "value": [...],
  "@odata.nextLink": "https://{env}.operations.dynamics.com/data/ReleasedProductsV2?$skiptoken=..."
}
```

Follow `@odata.nextLink` until it's absent. Or use server-side paging:
```
/data/ReleasedProductsV2?$select=ItemNumber&$top=5000&$skip=0
/data/ReleasedProductsV2?$select=ItemNumber&$top=5000&$skip=5000
```

## Throttling Awareness

D365 F&O has priority-based OData throttling:
- Interactive (user-context) requests get higher priority
- Service (app-context) requests can be throttled under load
- **Best practice:** Pull reference data during off-peak hours and cache
- If throttled, you get HTTP 429 — retry with exponential backoff
- Don't hammer the endpoint in tight loops — batch requests or add small delays

## Cross-Company Queries

By default, OData returns data for the authenticated user's default company. To query across companies:
```
/data/VendorGroups?cross-company=true&$select=VendorGroupId,dataAreaId
```

Or filter to a specific company:
```
/data/VendorGroups?$filter=dataAreaId eq 'USMF'&$select=VendorGroupId
```
