# Discover Entity Schema & Foreign Keys Dynamically

## The Problem
You have a CustomersV3 data file. How do you KNOW which fields are foreign keys that need reference data validation — without hardcoding a list?

## Solution: OData $metadata

Every D365 F&O environment exposes its complete entity schema at:

```
GET https://{environment}.operations.dynamics.com/data/$metadata
```

This returns an XML document (EDMX/CSDL) defining every entity, every field, every navigation property (foreign key relationship), and every enum type.

**Warning:** The full $metadata is HUGE (50MB+). Don't fetch the whole thing. Use targeted queries.

## Method 1: Get Specific Entity Metadata via OData

### Get the entity schema with field types
```
GET https://{env}.operations.dynamics.com/data/CustomersV3?$top=0
```
Response headers include the entity structure. But the real detail is in:

```
GET https://{env}.operations.dynamics.com/data/$metadata#CustomersV3
```

### What the metadata tells you per field

```xml
<EntityType Name="CustomerV3">
  <Key>
    <PropertyRef Name="CustomerAccount"/>
    <PropertyRef Name="dataAreaId"/>
  </Key>
  
  <!-- Simple fields (no FK) -->
  <Property Name="CustomerAccount" Type="Edm.String" Nullable="false"/>
  <Property Name="CustomerName" Type="Edm.String"/>
  
  <!-- Enum fields (validate against enum values) -->
  <Property Name="CreditRating" Type="Microsoft.Dynamics.DataEntities.CreditRating"/>
  
  <!-- FK indicator: NavigationProperty = relationship to another entity -->
  <NavigationProperty Name="CustomerGroup" Type="CustomerGroupEntity" 
    Partner="Customers">
    <ReferentialConstraint Property="CustomerGroupId" 
      ReferencedProperty="CustomerGroupId"/>
  </NavigationProperty>
  
  <NavigationProperty Name="PaymentTerm" Type="PaymentTermEntity">
    <ReferentialConstraint Property="PaymentTermsName" 
      ReferencedProperty="PaymentTermsName"/>
  </NavigationProperty>
  
  <NavigationProperty Name="Currency" Type="CurrencyEntity">
    <ReferentialConstraint Property="CurrencyCode" 
      ReferencedProperty="CurrencyCode"/>
  </NavigationProperty>
  
  <NavigationProperty Name="DeliveryMode" Type="DeliveryModeEntity">
    <ReferentialConstraint Property="DeliveryModeId" 
      ReferencedProperty="ModeOfDeliveryId"/>
  </NavigationProperty>
  
  <!-- ... more navigation properties = more FKs to validate -->
</EntityType>
```

**The rule:** Every `<NavigationProperty>` with a `<ReferentialConstraint>` is a foreign key. The `Property` attribute tells you which field on YOUR entity, and the `Type` tells you which REFERENCE entity to validate against.

## Method 2: MCP Server Metadata Discovery

If your agent uses the D365 MCP server, it can discover schema programmatically:

```
Step 1: data_find_entity_type("CustomersV3")
  → Returns entity type info

Step 2: data_get_entity_metadata("CustomersV3") 
  → Returns full schema including:
     - All fields with types (String, Int, DateTime, Enum)
     - Required vs optional
     - Navigation properties (= foreign keys)
     - Referenced entity for each FK
```

The agent can then automatically build the validation rule set.

## Method 3: Query Entity Field Metadata via OData

You can also programmatically pull entity metadata from the Data Management workspace:

```
GET /data/DataEntities?$filter=EntityName eq 'CustomersV3'
  → Returns entity registration info

GET /data/DataEntityFields?$filter=EntityName eq 'CustomersV3'
  → Returns all fields with:
     - FieldName
     - FieldType  
     - IsMandatory
     - FieldLabel
     - RelatedEntity (if FK)
```

**Note:** `DataEntityFields` may not be available as an OData entity in all versions. The $metadata approach is always available.

## Practical Implementation: Auto-Discovery Workflow

```python
# Pseudo-code for automatic FK discovery and validation

async def discover_and_validate(entity_name, source_file, environment):
    
    # 1. Fetch entity metadata
    metadata = await fetch_metadata(environment, entity_name)
    
    # 2. Extract navigation properties (FKs)
    foreign_keys = []
    for nav_prop in metadata.navigation_properties:
        if nav_prop.referential_constraint:
            foreign_keys.append({
                'source_field': nav_prop.constraint.property,      # e.g., "CustomerGroupId"
                'target_entity': nav_prop.type,                     # e.g., "CustomerGroupEntity"
                'target_field': nav_prop.constraint.referenced,     # e.g., "CustomerGroupId"
                'target_endpoint': pluralize(nav_prop.type),        # e.g., "CustomerGroups"
            })
    
    # 3. For each FK, pull valid reference values
    reference_cache = {}
    for fk in foreign_keys:
        endpoint = f"/data/{fk['target_endpoint']}?$select={fk['target_field']}"
        values = await odata_get(environment, endpoint)
        reference_cache[fk['source_field']] = {
            v[fk['target_field']] for v in values
        }
    
    # 4. Validate source file
    errors = []
    for row_num, row in enumerate(parse_file(source_file)):
        for fk in foreign_keys:
            value = row.get(fk['source_field'])
            if value and value not in reference_cache[fk['source_field']]:
                errors.append({
                    'row': row_num,
                    'field': fk['source_field'],
                    'value': value,
                    'expected_in': fk['target_endpoint'],
                })
    
    # 5. Report
    return ValidationReport(
        entity=entity_name,
        total_rows=len(source_data),
        foreign_keys_checked=len(foreign_keys),
        errors=errors
    )
```

## What This Discovers for CustomersV3

Running the auto-discovery against CustomersV3 $metadata reveals these navigation properties (FK relationships):

| Source Field (on Customer) | Target Entity | Target Endpoint | Target Key Field |
|---|---|---|---|
| `CustomerGroupId` | CustomerGroupEntity | `/data/CustomerGroups` | `CustomerGroupId` |
| `PaymentTermsName` | PaymentTermEntity | `/data/PaymentTerms` | `PaymentTermsName` |
| `CurrencyCode` | CurrencyEntity | `/data/Currencies` | `CurrencyCode` |
| `TaxGroup` | TaxGroupEntity | `/data/SalesTaxGroups` | `TaxGroup` |
| `DeliveryModeId` | DlvModeEntity | `/data/ModesOfDelivery` | `ModeOfDeliveryId` |
| `DeliveryTermsId` | DlvTermEntity | `/data/TermsOfDelivery` | `TermsOfDeliveryId` |
| `PaymentMethod` | CustPaymentMethodEntity | `/data/CustomerPaymentMethods` | `Name` |
| `CashDiscountCode` | CashDiscountEntity | `/data/CashDiscounts` | `CashDiscountCode` |
| `SiteId` | InventSiteEntity | `/data/Sites` | `SiteId` |
| `WarehouseId` | InventLocationEntity | `/data/Warehouses` | `WarehouseId` |
| `LanguageId` | LanguageEntity | `/data/Languages` | `LanguageId` |
| `CustomerPostingProfileId` | CustPostingProfileEntity | `/data/CustomerPostingProfiles` | `PostingProfile` |

**Plus these which need special handling (not simple navigation properties):**
- `DefaultDimensionDisplayValue` — requires parsing per financial dimension format, then validating each segment separately
- `AddressCountryRegionId` / `AddressState` / `AddressCity` — part of the address composite, validated in hierarchy
- `CustomerAccount` — primary key, checked for duplicates not FK

## Handling Non-Standard FKs

Some fields that NEED validation aren't exposed as navigation properties:

### Financial Dimensions
`DefaultDimensionDisplayValue` is a STRING like "001-02-WEST" that combines multiple dimension values. Not a simple FK.

**Validation approach:**
1. Query the active default dimension format from Financial dimension configuration
2. Parse the string by delimiter (typically "-")
3. Map each segment to its dimension name (1st = Department, 2nd = CostCenter, 3rd = Region)
4. Validate each segment: `GET /data/FinancialDimensionValues?$filter=DimensionName eq 'Department' and DimensionValue eq '001'`

### Address Hierarchy
Address fields form a hierarchy, not independent FKs:
1. Validate country first
2. Validate state WHERE country = the validated country
3. Validate city WHERE state = the validated state
4. Validate zip WHERE city = the validated city

### Enum Fields
Fields with `Type="Microsoft.Dynamics.DataEntities.SomeEnum"` in metadata have a fixed set of valid values. Pull enum values from:
```
GET /data/$metadata
```
And search for `<EnumType Name="SomeEnum">` — lists all valid `<Member>` values.

## The Complete Auto-Validation Flow

```
1. DISCOVER:  $metadata → extract NavigationProperties → build FK map
2. ENRICH:    Add special handlers (dimensions, addresses, enums)
3. PULL:      GET /data/{each reference entity}?$select={key} → cache
4. VALIDATE:  For each source row, check each FK value against cache
5. REPORT:    Group errors by field, count, sample values
```

This means your validation agent can handle ANY entity — standard or custom — without pre-mapped dependency tables. The metadata IS the map.
