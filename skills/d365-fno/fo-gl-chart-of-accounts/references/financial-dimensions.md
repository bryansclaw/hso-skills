# Financial Dimensions Setup

Source: Microsoft Learn — verified 2026-03-18
Navigation: **General ledger > Chart of accounts > Dimensions > Financial dimensions**

## What Financial Dimensions Are

Financial dimensions categorize financial transactions beyond the main account. They become segments within the ledger account for analysis (e.g., P&L by Department, trial balance by Cost Center).

**Alternative:** Financial tags (up to 20 user-defined tags per transaction) — lighter weight than dimensions, no account structure enforcement. See Financial tags documentation for comparison.

## Two Types of Financial Dimensions

### Custom Dimensions
- Created by selecting **Custom dimension** in the "Use values from" field
- Values are shared across all legal entities
- Users manually enter and maintain values
- Can set a **dimension value mask** to control format:
  - `#` = placeholder for a number
  - `&` = placeholder for a letter
  - Fixed characters stay the same for every value
  - Example: `CC-###` limits values to "CC-001", "CC-002", etc.
- **Do NOT use the chart of accounts delimiter in the mask**

### Entity-Backed Dimensions
- Created by selecting a system entity in "Use values from" (e.g., Projects, Customers, Stores)
- Values are pulled from the existing entity table — no manual value entry
- A dimension value is automatically created for each record in the backing entity
- Some entity-backed dimensions are **shared** across legal entities; others are **company-specific** (company-striped)
- Company-specific dimensions are only visible from the associated company context

**Common entity-backed dimensions:**
- **Department** — backed by Operating Units (type = Department)
- **Cost Center** — backed by Operating Units (type = Cost Center)
- **Business Unit** — backed by Operating Units (type = Business Unit)
- **Project** — backed by Projects table
- **Customer** — backed by Customers table
- **Vendor** — backed by Vendors table
- **Worker** — backed by Workers table
- **Item** — backed by Released Products table

## Naming Requirements
- Must start with a letter or underscore
- Followed by any combination of letters, numbers, or underscores
- Cannot use reserved system field names (e.g., RecId)
- Invalid: `123Dept` (starts with number), `Cost-Center` (contains hyphen), `RecId` (reserved)
- Valid: `Department`, `CostCenter`, `_CustomDim`, `Project_1`

## Financial Dimension Values
Navigation: **General ledger > Chart of accounts > Dimensions > Financial dimension values**

- For custom dimensions: manually create values here
- For entity-backed dimensions: values auto-populate from the backing table
- Values page shows company if the dimension is company-specific
- **Export limitation:** Only custom dimension values and entity-backed values explicitly marked for export will be included in data entity exports

## Data Entities
- **Dimension definitions:** `FinancialDimensions` — creates the dimension itself
- **Dimension values (custom):** `FinancialDimensionValues`
- **Entity-backed values:** Created by importing records to the backing entity (e.g., import Operating Units to create Department values)
- **MCP approach:** Data tool for definitions and custom values; backing entity tools for entity-backed values

## Key Decisions for Implementation
1. **How many dimensions?** — fewer = simpler but less analytical power. Most implementations use 3-5.
2. **Custom vs entity-backed?** — entity-backed ensures dimension values stay in sync with master data. Use custom only when no backing entity exists.
3. **Which dimensions in which account structure?** — Balance sheet may need fewer dimensions than P&L. Use advanced rules to add dimensions only where needed.
4. **Budgeting impact:** Any dimension you want to budget against MUST be in an account structure (not in advanced rules — budgeting doesn't use advanced rules).
