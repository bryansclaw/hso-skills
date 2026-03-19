# Level 4 Patterns — General Ledger (Record to Report)

Maps BPC Level 3 processes → Level 4 scenarios/patterns → specific configuration fields.
Each pattern = a specific customer scenario that drives field-level decisions.

---

## 13.1.3 Develop Chart of Accounts Strategy

### Pattern: Shared Chart of Accounts Across Legal Entities

**Trigger:** Customer says "All our entities should use the same chart of accounts"

**Questions specific to this pattern:**

| # | Question | Field It Configures |
|---|---|---|
| 1 | What is the account number format? (5 digits? 6 digits? Alphanumeric?) | Main account > MainAccountId format |
| 2 | What numbering ranges per account type? | Main account ranges (e.g., 10000-19999 = Assets) |
| 3 | Do you need to suspend any accounts for specific entities? | Main account > Legal entity overrides > Suspended |
| 4 | What is the segment delimiter? | GL parameters > Chart of accounts and dimensions > Change delimiter (default "-") |
| 5 | Do any entities need unique accounts that others don't use? | Main account > Legal entity overrides (add/suspend per entity) |

**Output Configuration:**

```
1. GL > Chart of accounts > Accounts > Chart of accounts > New
   - Chart of accounts: [customer's name, e.g., "Global CoA"]
   
2. GL > Chart of accounts > Accounts > Main accounts > New (for each account)
   - MainAccountId: [per numbering scheme]
   - Name: [account name]
   - Main account type: [Asset/Liability/Equity/Revenue/Expense/P&L/Total]
   - Main account category: [mapped to reporting category]
   - Do not allow manual entry: [Yes for control accounts - AP, AR, Inventory]
   - Currency: [blank = all currencies allowed, or restrict to specific]
   
3. GL > Ledger setup > Ledger (PER ENTITY - switch company context)
   - Chart of accounts: "Global CoA" [same selection for all entities]
```

**Validation Test:** Navigate to GL > Ledger setup > Ledger in each entity. Verify all show the same Chart of accounts name.

---

### Pattern: Separate Chart of Accounts Per Country

**Trigger:** Customer says "Our US entity has different reporting requirements than our German entity"

**Additional Questions:**

| # | Question | Drives |
|---|---|---|
| 1 | Which entities share which CoA? | CoA groupings (e.g., US CoA, EU CoA) |
| 2 | Are there shared accounts across CoAs? (intercompany, cash) | Account numbering coordination |
| 3 | How do you consolidate for group reporting? | Consolidation CoA design |

**Output:** Create multiple CoAs. Each entity assigned its own on the Ledger page. Create a consolidation company with a reporting CoA that maps accounts from both.

---

### Pattern: Financial Dimensions — Department and Cost Center

**Trigger:** Customer says "We need to track by department and cost center"

**Deep questions:**

| # | Question | Drives | Field |
|---|---|---|---|
| 1 | Are departments already defined as operating units in D365? | Entity-backed vs custom | Financial dimensions > Use values from: "Operating Unit — Department" |
| 2 | Are cost centers already operating units? | Same | Financial dimensions > Use values from: "Operating Unit — Cost Center" |
| 3 | How many departments? List them. | Operating unit creation | Org admin > Operating units (type = Department) |
| 4 | How many cost centers? List them. | Operating unit creation | Org admin > Operating units (type = Cost Center) |
| 5 | Should blank be allowed? (Can a transaction post without a department?) | Account structure allowed values | Account structure > Dept segment > Include blank: Yes/No |
| 6 | Are there department-cost center combinations that are INVALID? | Account structure constraints | Account structure > Allowed value details > Criteria |

**Output Configuration:**

```
1. Org admin > Organizations > Operating units > New
   - Type: Department
   - Name: [each department name]
   (Repeat for each department)

2. Org admin > Organizations > Operating units > New
   - Type: Cost Center  
   - Name: [each cost center name]
   (Repeat for each cost center)

3. GL > Chart of accounts > Dimensions > Financial dimensions > New
   - Dimension name: "Department"
   - Use values from: "Operating units" (type filter = Department)
   
4. GL > Chart of accounts > Dimensions > Financial dimensions > New
   - Dimension name: "CostCenter"
   - Use values from: "Operating units" (type filter = Cost Center)

5. GL > Chart of accounts > Structures > Configure account structures
   - Structure name: [e.g., "P&L Structure"]
   - Segments: MainAccount | Department | CostCenter
   - MainAccount allowed values: [P&L range, e.g., 400000..999999]
   - Department allowed values: * (all) or specific list
   - CostCenter allowed values: * (all) or specific list
   - Include blank: [per question 5 answer]
   → ACTIVATE the structure

6. GL > Chart of accounts > Dimensions > Financial dimension configuration for integrating applications
   - Ledger dimension format: MainAccount-Department-CostCenter
   - Default dimension format: Department-CostCenter
   → ACTIVATE both formats
```

**Validation Test:**
- Create a test journal entry with account 600100-SALES-CC001
- Verify the entry validates successfully
- Try an invalid combination (if restricted) — verify it's blocked

---

### Pattern: Project Dimension on Expense Accounts Only

**Trigger:** Customer says "We need project tracking but only on expense accounts, not balance sheet"

**This uses advanced rules.**

**Output Configuration:**

```
1. Create Project financial dimension:
   GL > Dimensions > New
   - Name: "Project"
   - Use values from: "Projects" (entity-backed) OR "Custom dimension"

2. Create base account structure WITHOUT project:
   GL > Structures > Configure account structures
   - Name: "Main Structure"
   - Segments: MainAccount | Department | CostCenter
   - MainAccount: * (all accounts)
   → ACTIVATE

3. Create advanced rule structure:
   GL > Structures > Advanced rule structures > New
   - Name: "Project Rule"
   - Add segment: Project

4. Attach advanced rule to main structure:
   GL > Structures > [Main Structure] > Advanced rules > New
   - Criteria: MainAccount between 600000 and 699999 (expense accounts)
   - Advanced rule structure: "Project Rule"
   → ACTIVATE structure with rule

Result: Only expense accounts 600000-699999 prompt for Project dimension.
All other accounts use MainAccount + Department + CostCenter only.
```

**⚠️ Warning:** Budgeting does NOT use advanced rules. If customer wants to budget by Project, Project MUST be in the base structure instead. This forces Project onto ALL accounts, even BS. Trade-off discussion needed.

---

### Pattern: Financial Tags Instead of Dimensions

**Trigger:** Customer says "We want to track some attributes on transactions but don't need them in account structures or budget control"

**When to suggest tags over dimensions:**
- Attribute is for reporting/analysis only, not for controlling postings
- Don't want account structure validation overhead
- Up to 20 tags allowed
- Simpler to set up and maintain

**Output Configuration:**

```
1. GL > Chart of accounts > Financial tags > Financial tag configuration
   - Define tag names (up to 20)
   - Define allowed values per tag (or free text)
   - Tags appear on journal lines and transaction forms
   
2. No account structure changes needed
3. No financial dimension configuration for integration needed
```

---

## 13.1.5 Define Posting Policies

### Pattern: Standard Posting Profiles for AP/AR/Inventory

**Trigger:** All vendors use same GL accounts, all customers use same GL accounts

**Questions → Field-Level Output:**

| Question | Customer Answer | Configuration Output |
|---|---|---|
| "What GL account for AP control (vendor balances)?" | "21000 - Accounts Payable" | AP > Vendor posting profiles > Summary account = 21000 |
| "What GL account for purchase expenditure not yet invoiced?" | "21500 - Accrued Purchases" | AP > Vendor posting profiles > Purchase expenditure for product = 21500 |
| "What GL account for AR control (customer balances)?" | "12000 - Accounts Receivable" | AR > Customer posting profiles > Summary account = 12000 |
| "What GL account for sales revenue?" | "40000 - Sales Revenue" | AR > Customer posting profiles > Revenue = 40000 (or configure per sales posting) |
| "What GL account for inventory?" | "14000 - Inventory" | Inventory management > Posting > Inventory > Financial inventory = 14000 |
| "What GL account for COGS?" | "50000 - Cost of Goods Sold" | Inventory management > Posting > Sales > Cost of goods sold = 50000 |
| "What account for year-end close (retained earnings)?" | "30000 - Retained Earnings" | GL > Posting setup > Accounts for automatic transactions > Year-end close = 30000 |
| "What account for rounding differences?" | "69990 - Rounding" | GL > Posting setup > Accounts for automatic transactions > Penny difference = 69990 |

**Validation Test (critical — from Microsoft):**
1. Create a PO → receive → invoice → pay → verify GL entries at each step
2. Create a SO → ship → invoice → collect payment → verify GL entries
3. Run trial balance → verify AP control = AP aging total, AR control = AR aging total
4. Do this as a mock period close BEFORE go-live

---

### Pattern: Different Posting Profiles Per Vendor/Customer Group

**Trigger:** "Domestic vendors post to different GL accounts than foreign vendors"

**Output Configuration:**

```
1. AP > Vendor posting profiles > New
   - Posting profile: "DOM"
   - Account code: Group
   - Group: "DOMESTIC"
   - Summary account: 21000 (Domestic AP)
   
2. AP > Vendor posting profiles > New
   - Posting profile: "FOR"
   - Account code: Group
   - Group: "FOREIGN"
   - Summary account: 21100 (Foreign AP)

3. AP > Vendor posting profiles > New  
   - Posting profile: "ALL"
   - Account code: All
   - Summary account: 21000 (fallback)

Priority resolution: D365 checks Table first, then Group, then All.
A vendor in group FOREIGN gets profile FOR (account 21100).
A vendor in group DOMESTIC gets profile DOM (account 21000).
Any vendor not in either group gets ALL (account 21000).
```

---

## 13.1.6 Develop Tax Strategy

### Pattern: US Multi-Jurisdiction (State + County + City)

**Trigger:** "We sell to customers across multiple US states with different tax rates"

**Questions → Configuration:**

| Question | Example Answer | Config |
|---|---|---|
| Which states? | CA, TX, NY, FL | Create sales tax codes: CA-STATE, TX-STATE, NY-STATE, FL-STATE |
| County taxes? | Yes, some counties | Create codes: CA-LA-COUNTY, NY-NYC-COUNTY, etc. |
| City taxes? | NYC has city tax | Create code: NY-NYC-CITY |
| Tax rates? | CA: 7.25%, TX: 6.25%, NY: 4%, NYC county: 4.5%, NYC city: 4.875% | Set values on each code with effective dates |
| How grouped? | By ship-to state | Create sales tax groups per state: TX-GROUP (contains TX-STATE), NY-GROUP (contains NY-STATE + NY-NYC-COUNTY + NY-NYC-CITY) |
| Product taxability? | Most products taxable, food exempt in some states | Create item tax groups: TAXABLE (all codes), FOOD (only codes where food is taxable) |

**Output Configuration:**

```
1. Tax > Sales tax authorities > New (one per state tax authority)
   - Name: "California FTB", "Texas Comptroller", etc.
   - Each creates an auto-vendor for tax payments

2. Tax > Settlement periods > New (per authority)
   - Period: Monthly or Quarterly per state requirement
   - Link to authority

3. Tax > Ledger posting groups > New
   - Sales tax payable: 22100
   - Sales tax receivable: 16100
   - Use tax expense: 63000
   - Settlement: 22100

4. Tax > Sales tax codes > New (one per jurisdiction × rate)
   - Code: CA-STATE | Settlement period: CA-Monthly | Posting group: STD-TAX
   - Values: 7.25% effective 01/01/2026
   
   - Code: TX-STATE | Settlement period: TX-Quarterly | Posting group: STD-TAX
   - Values: 6.25% effective 01/01/2026

   - Code: NY-STATE | Values: 4.00%
   - Code: NY-NYC-COUNTY | Values: 4.50%  
   - Code: NY-NYC-CITY | Values: 4.875%

5. Tax > Sales tax groups > New
   - TX-GROUP: contains TX-STATE
   - NY-GROUP: contains NY-STATE + NY-NYC-COUNTY + NY-NYC-CITY
   - CA-GROUP: contains CA-STATE
   - FL-GROUP: contains FL-STATE

6. Tax > Item sales tax groups > New
   - TAXABLE: contains ALL tax codes
   - FOOD: contains only codes for states where food IS taxable
   - EXEMPT: contains no codes (intersection = nothing = no tax)
```

**Validation:** Create a sales order to a NY customer with a taxable item → tax should be 4% + 4.5% + 4.875% = 13.375%.

---

### Pattern: ISV Tax Engine (Avalara/Vertex)

**Trigger:** "We use Avalara for tax calculation"

**Output Configuration (what D365 still needs):**

```
D365 handles POSTING only. Avalara handles CALCULATION.

Still needed in D365:
1. Tax > Ledger posting groups > New
   - Sales tax payable: 22100 (Avalara sends the amount, D365 posts it here)
   - Sales tax receivable: 16100

2. Tax > Sales tax codes > New
   - Create a minimal set (e.g., one code per tax type)
   - Rates can be 0% — Avalara overrides at runtime
   
3. Tax > Sales tax groups / Item sales tax groups
   - Minimal setup — Avalara maps products and customers to rates

4. Install Avalara ISV solution from AppSource
5. Configure Avalara connection (API keys, company code)
6. Map D365 tax codes to Avalara tax types

What you DON'T configure:
- Individual state/county/city tax codes (Avalara has its own database)
- Rate values (Avalara calculates real-time)
- Tax jurisdiction hierarchies
```

---

## 13.1.7 Define Banking Policies

### Pattern: US Company with ACH Payments and Bank Statement Import

**Trigger:** "We pay vendors electronically via ACH and import bank statements for reconciliation"

**Questions → Configuration:**

| Question | Answer | Config |
|---|---|---|
| Bank name? | Chase Business Checking | Bank account record |
| Account #? | ****1234 | Bank account > Account number |
| Routing #? | 021000021 | Bank account > Routing number |
| GL account? | 11000 - Cash | Bank account > Main account = 11000 |
| Currency? | USD | Bank account > Currency = USD |
| Statement format? | BAI2 | Import ER config for BAI2 |
| Reconciliation type? | Advanced (auto-matching) | Enable advanced bank reconciliation |
| Matching rules? | Match by amount + date within 1 day tolerance | Reconciliation matching rules |

**Output Configuration:**

```
1. Cash and bank > Bank groups > New
   - Bank group: "DOMESTIC"

2. Cash and bank > Bank accounts > New
   - Bank account: "CHASE-BUS"
   - Name: "Chase Business Checking"
   - Bank group: DOMESTIC
   - Routing number: 021000021
   - Account number: ****1234
   - Main account: 11000
   - Currency: USD
   - Advanced bank reconciliation: Yes (on General tab)

3. Org admin > Electronic reporting > Configurations > Load from repository
   - Search: "NACHA"
   - Import: Generic NACHA format
   
   - Search: "BAI2"
   - Import: BAI2 bank statement format

4. AP > Payment setup > Methods of payment > New (or edit "ELECTRONIC")
   - Name: "ACH"
   - Period: Current
   - File format tab:
     - Generic electronic export format: YES
     - Export format configuration: [select imported NACHA config]
   - Payment control tab:
     - Bank account: CHASE-BUS

5. Cash and bank > Advanced bank reconciliation setup > Reconciliation matching rules > New
   - Rule: "Amount+Date"
   - Match criteria: Amount (exact) + Date (tolerance: 1 day)
   - Auto-match: Yes

6. Cash and bank > Setup > Bank statement format
   - Format: BAI2
   - Configuration: [select imported BAI2 ER config]
```

**Validation:** Create a vendor payment journal → generate ACH file → verify NACHA format. Import a sample BAI2 statement → run matching → verify auto-matched transactions.

---

## 13.2.1 Create and Manage Budgets

### Pattern: Annual Budget with Monthly Allocation and Budget Control

**Trigger:** "We create an annual budget and spread it equally across 12 months. We want to prevent overspending."

**Questions → Configuration:**

| Question | Answer | Config |
|---|---|---|
| Budget model name? | "FY2026 Budget" | Budget model creation |
| Dimensions to budget by? | Main account + Department | Budget dimensions (must be in account structure) |
| Allocation method? | Equal across 12 months | Budget allocation terms = Equal |
| Control type? | Hard stop — prevent posting over budget | Over-budget permissions = No override |
| Which documents check budget? | Purchase orders and vendor invoices | Documents and journals tab |
| Budget funds formula? | Budget - Actuals - Encumbrances = Available | Budget funds available calculation |
| Who manages budget? | Controller (Jane Smith) | Budget manager assignment |
| Threshold for warning? | 90% consumed = warning | Threshold = 90% |

**Output Configuration:**

```
1. Budgeting > Setup > Basic budgeting > Budget models > New
   - Budget model: "FY2026"
   
2. Budgeting > Setup > Basic budgeting > Budget dimensions
   - Select: Main account ☑, Department ☑

3. Budgeting > Setup > Basic budgeting > Budget allocation terms > New
   - Allocation: "EQUAL-12"
   - Method: Equal, 12 periods

4. Budgeting > Setup > Budget control > Budget control configuration
   Tab: Define parameters
   - Financial dimensions: Main account, Department
   - Time span: Fiscal year
   - Budget manager: Jane Smith
   - Threshold: 90%
   
   Tab: Over budget permissions
   - [No user groups added = nobody can override]
   
   Tab: Budget funds available
   - Original budget: ☑
   - Actual expenditures: ☑ (subtract)
   - Encumbrances (POs): ☑ (subtract)
   - Available = Budget - Actuals - Encumbrances
   
   Tab: Documents and journals
   - Purchase orders: ☑
   - Vendor invoices: ☑
   
   → ACTIVATE budget control

5. Import budget via DMF:
   Entity: BudgetAccountEntry
   - Budget model: FY2026
   - Account + Department per line
   - Amount: annual amount ÷ 12 per period
   → Post the budget register entry (status = Posted, not Draft)
```

**Validation:** Create a PO that exceeds the budget for a department → verify the system blocks it with "Budget funds not available" error.

---

## Process → Questions → Config Summary Table

The master lookup an agent uses to find the right questions for any BPC process:

| BPC ID | Level 3 Process | Key Patterns (Level 4) | # Questions | Config Outputs |
|---|---|---|---|---|
| 13.1.1 | Develop company structure | Single entity, Multi-entity, Multi-country, Intercompany | 6 | Legal entities, operating units, hierarchies |
| 13.1.2 | Develop financial period strategy | Standard year, Non-standard, 13-period, 4-4-5 | 5 | Fiscal calendar, periods, period close workspace |
| 13.1.3 | Develop chart of accounts strategy | Shared CoA, Separate CoA, Dimensions, Tags, Advanced rules | 10 | CoA, main accounts, dimensions, structures, integration config |
| 13.1.4 | Develop currency policies | Single currency, Multi-currency, Revaluation | 6 | Currencies, exchange types, rates, providers |
| 13.1.5 | Define posting policies | Standard profiles, Group-level profiles, Posting definitions | 5 | Posting profiles per module, automatic transaction accounts |
| 13.1.6 | Develop tax strategy | Single jurisdiction, Multi-jurisdiction, ISV engine, Withholding | 9 | Tax authorities, codes, groups, item groups, posting groups |
| 13.1.7 | Define banking policies | ACH, SEPA, Check, Statement import, Positive pay, Cash flow | 7 | Bank accounts, ER formats, payment methods, reconciliation |
| 13.1.8 | Define costing policies | Standard cost, FIFO, Moving avg, Inventory close | 4 | Item model groups, costing versions, close parameters |
| 13.1.9 | Develop asset policies | Standard depreciation, Tax book, Capitalization | 7 | FA groups, depreciation profiles, value models, posting profiles |
| 13.1.10 | Develop budgeting strategy | No budget, External budget, D365 budget, Budget control | 8 | Budget models, dimensions, control config, allocation terms |
| 05.2.1 | Create vendor records | Vendor groups, Payment terms, Matching, Electronic payments | 8 | Vendor groups, terms, discounts, posting profiles, methods |
| 05.4.1 | Invoice matching | 2-way, 3-way, Price tolerance, Quantity tolerance | 5 | Matching policy, tolerances per level |
| 02.3.1 | Customer setup | Customer groups, Credit mgmt, Collections, Payment methods | 10 | Customer groups, posting profiles, credit rules, collection sequences |
| 02.1.1 | Pricing strategy | Fixed prices, Customer-specific, Volume, Discounts | 5 | Trade agreements, price groups, discount types |
| 12.1.1 | Inventory costing | FIFO, LIFO, Standard, Moving avg, WMS toggle | 7 | Item model groups, dimension groups, WMS decision |
| 04.1.1 | Production model | Discrete, Process, BOM, Routes, Scheduling | 10 | Resources, route groups, operations, production groups |
| 08.1.1 | Asset classification | FA groups, Depreciation, Value models, Capitalization | 7 | FA groups, profiles, value models, posting profiles |
