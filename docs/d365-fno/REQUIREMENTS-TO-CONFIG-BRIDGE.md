# Requirements-to-Configuration Bridge

**Purpose:** Close the loop between requirements workshops and D365 F&O configuration. Every business requirement maps to a BPC process, every process maps to configuration steps, and every configuration step has workshop questions that drive the decisions.

**Version:** 0.1 (Draft)
**Date:** 2026-03-19

---

## How This Document Works

```
WORKSHOP QUESTION → CUSTOMER ANSWER → DECISION → CONFIGURATION → VALIDATION
       ↓                  ↓               ↓            ↓              ↓
  BPC Level 3/4     Drives design    If/then rules  Menu path +    Test case
  business process  choices          per answer     data entity    to verify
```

The BPC has 6 levels. This bridge operates primarily at **Levels 3 and 4** — the business process and scenario levels — because that's where requirements become specific enough to drive configuration decisions.

### BPC Configuration Stage Terminology (from Microsoft)

| Stage | Mandatory? | Can Change Later? | Can Add Later? | When |
|---|---|---|---|---|
| **Base** | YES | NO | NO | Initialize-Design |
| **Fundamental** | NO | YES | YES | Design-Operate |
| **Optional** | NO | NO | YES | Design-Operate |

**Gold** = recommended for production environment configuration
**Early** = don't defer to later iterations
**Continuous** = will be maintained/updated regularly post go-live

---

## 1. Record to Report (End-to-End Process 13)

### 1.1 Define Accounting Policies (Level 2)

#### 1.1.1 Develop Company Structure (Level 3)

**Workshop Questions:**

| # | Question | Why It Matters | Answer Drives |
|---|---|---|---|
| 1 | How many legal entities (companies) will you operate in D365? | Determines legal entity setup, intercompany config | Legal entity creation |
| 2 | List each legal entity with: name, country, currency, tax registration | Country determines localization features; currency sets accounting currency | Legal entity config per country |
| 3 | Do any entities transact with each other (intercompany)? | Requires intercompany accounting setup with due-to/due-from accounts | Intercompany journal config |
| 4 | Do you need centralized payments (pay from one entity on behalf of others)? | Requires org hierarchy with centralized payment purpose | Organization hierarchy setup |
| 5 | What is your organizational reporting structure? (Divisions, departments, regions?) | Determines operating units and hierarchy design | Operating units, hierarchies |
| 6 | Do you need to consolidate financials across entities for reporting? | Requires consolidation company and elimination rules | Consolidation setup |

**Decision Matrix:**

| If Answer | Then Configure |
|---|---|
| Single legal entity | One company, simpler setup, no intercompany |
| Multiple entities, same country | Multiple companies, shared CoA possible, single localization |
| Multiple entities, different countries | Multiple companies, may need unique CoA per country, multiple localizations |
| Intercompany transactions exist | Intercompany accounting pairs (GL > Intercompany accounting), due-to/due-from accounts |
| Centralized payments | Organization hierarchy with centralized payment purpose |
| Consolidation needed | Create consolidation company + elimination rules |

**Configuration Steps:**

| Step | Stage | Menu Path | Entity | MCP |
|---|---|---|---|---|
| Create legal entities | Base; Gold; Early | Org admin > Organizations > Legal entities | `LegalEntities` | Data tool |
| Set primary address per entity | Base; Gold | Legal entity > Addresses | Part of `LegalEntities` | Data tool |
| Create operating units | Fundamental; Gold | Org admin > Organizations > Operating units | `OperatingUnits` | Data tool |
| Create org hierarchies | Fundamental; Gold | Org admin > Organizations > Organization hierarchies | N/A | Form tool |
| Assign hierarchy purposes | Base; Gold | Org hierarchy > Assign purpose | N/A | Form tool |
| Set up intercompany accounting | Fundamental; Gold | GL > Intercompany accounting | N/A | Form tool |
| Create consolidation company | Optional | GL > Period close > Consolidation companies | `LegalEntities` | Data tool |

---

#### 1.1.2 Develop Financial Period Strategy (Level 3)

**Workshop Questions:**

| # | Question | Why It Matters | Answer Drives |
|---|---|---|---|
| 1 | What is your fiscal year? (Jan-Dec? Apr-Mar? Other?) | Determines fiscal calendar setup | Fiscal calendar dates |
| 2 | How many periods per year? (12 monthly? 13-period? 4-4-5?) | D365 supports all structures | Fiscal period generation |
| 3 | Do all legal entities use the same fiscal calendar? | Shared vs separate calendars | Calendar assignment per entity |
| 4 | Do you close periods monthly? Quarterly? | Determines close workspace task lists | Period close setup |
| 5 | How quickly after period end do you close? | Affects period hold timing | Period status management |

**Decision Matrix:**

| If Answer | Then Configure |
|---|---|
| Standard Jan-Dec, 12 periods | Create calendar, generate 12 monthly periods + opening/closing |
| Non-standard fiscal year (Apr-Mar) | Create calendar with correct start month |
| 13-period or 4-4-5 | Create calendar with custom period structure |
| Multiple calendars needed | Create one per entity, assign separately on Ledger page |
| Monthly close | Configure period close workspace with monthly task list |

**Configuration Steps:**

| Step | Stage | Menu Path | Entity |
|---|---|---|---|
| Create fiscal calendar | Base; Gold; Early | GL > Calendars > Fiscal calendars | `FiscalCalendars` |
| Generate fiscal years | Base; Gold | Within calendar > Generate | `FiscalCalendarYears` |
| Generate periods | Base; Gold | Within year > Generate | `FiscalCalendarPeriods` |
| Set period status | Operational; Continuous | Calendar > Period status | Form tool |

---

#### 1.1.3 Develop Chart of Accounts Strategy (Level 3)

**Workshop Questions:**

| # | Question | Why It Matters | Answer Drives |
|---|---|---|---|
| 1 | Will all legal entities share one chart of accounts, or do some need separate ones? | Shared = simpler maintenance; separate = more flexibility per country | CoA creation and assignment |
| 2 | What account numbering scheme? (5-digit? 6-digit? Ranges for asset/liability/equity/revenue/expense?) | Determines main account structure | Account number design |
| 3 | What are your main account categories? (Align to GAAP? IFRS? Both?) | Categories drive default financial reports | Main account category setup |
| 4 | How many financial dimensions do you need? What are they? | Dimensions provide analytical granularity beyond main account | Dimension creation |
| 5 | For each dimension: is it a custom list or backed by an existing entity? (e.g., Department from operating units) | Custom = manual values; entity-backed = auto-synced | Dimension type selection |
| 6 | Which dimensions apply to balance sheet accounts? Which apply to P&L? | Determines account structure design | Account structure segments |
| 7 | Are there combinations of account + dimensions that should NOT be allowed? | Drives account structure constraints and advanced rules | Allowed values, advanced rules |
| 8 | Do any accounts need to be restricted to specific legal entities? | Legal entity overrides | Main account overrides |
| 9 | Do you plan to budget by financial dimension? Which dimensions? | Budgeted dimensions MUST be in account structures, not advanced rules | Account structure design |
| 10 | What is the GL segment delimiter? (Default is "-") | Delimiter separates main account from dimensions in display | GL parameters |

**Decision Matrix:**

| If Answer | Then Configure |
|---|---|
| Shared CoA across all entities | Create one CoA, assign to all entities on Ledger page |
| Separate CoA per entity/country | Create multiple CoAs, assign per entity |
| 3 financial dimensions (Dept, CC, Project) | Create 3 dimensions — Dept and CC as entity-backed (Operating Units), Project as entity-backed or custom |
| Dept+CC on all accounts, Project only on expense | Two account structures: (1) All accounts + Dept + CC, (2) Expense accounts + Dept + CC + Project. OR one structure with advanced rule adding Project to expense range |
| Budget by Dept and CC | These dimensions MUST be in the base account structure (not advanced rules) |
| Microsoft's best practice | Structure 1: BS (100000-399999) + BU; Structure 2: P&L (400000-999999) + BU + Dept + CC; Advanced rule: Sales accounts → also require Customer |

**Configuration Steps:**

| Step | Stage | Menu Path | Entity |
|---|---|---|---|
| Create chart of accounts | Base; Gold; Early | GL > Chart of accounts > Accounts > Chart of accounts | `ChartOfAccounts` |
| Create main account categories | Base; Gold | GL > Chart of accounts > Accounts > Main account categories | `MainAccountCategories` |
| Create financial dimensions | Base; Gold; Early | GL > Chart of accounts > Dimensions > Financial dimensions | `FinancialDimensions` |
| Create dimension values | Fundamental; Gold | GL > Dimensions > Financial dimension values (or backing entity) | `FinancialDimensionValues` |
| Create main accounts | Base; Gold | GL > Chart of accounts > Accounts > Main accounts | `MainAccounts` |
| Configure account structures | Base; Gold; Early | GL > Chart of accounts > Structures > Configure account structures | Form tool (complex) |
| Activate account structures | Base; Gold | Account structure > Activate | Form tool |
| Configure advanced rules | Optional | GL > Chart of accounts > Structures > Advanced rule structures | Form tool |
| Assign CoA to ledger | Base; Gold; Early | GL > Ledger setup > Ledger | Form tool |
| Set up financial dimension config for integration | Base; Gold; Early | GL > Chart of accounts > Dimensions > Financial dimension configuration for integrating applications | Form tool |

**⚠️ Critical decisions that CANNOT change later:**
- Chart of accounts assignment CANNOT change after transactions are posted
- Account structure main account ranges CANNOT overlap
- Budgeted dimensions MUST be in base account structures

---

#### 1.1.4 Develop Currency Policies (Level 3)

**Workshop Questions:**

| # | Question | Answer Drives |
|---|---|---|
| 1 | What is the accounting (home) currency for each legal entity? | Ledger setup per entity |
| 2 | Do you need a secondary reporting currency? | Dual-currency reporting on Ledger page |
| 3 | What currencies do you transact in? (List all) | Currency code creation |
| 4 | Where do exchange rates come from? (Manual entry? OANDA? Central bank?) | Exchange rate provider setup |
| 5 | How frequently are exchange rates updated? (Daily? Weekly?) | Rate import automation |
| 6 | Do you revalue open foreign currency balances at period end? | Foreign currency revaluation setup |

**Configuration Steps:**

| Step | Menu Path | Entity |
|---|---|---|
| Create currency codes | GL > Currencies > Currencies | `Currencies` |
| Create exchange rate types | GL > Currencies > Exchange rate types | `ExchangeRateTypes` |
| Configure exchange rate provider | GL > Currencies > Configure exchange rate providers | Form tool |
| Import exchange rates | GL > Currencies > Import currency exchange rates | Form tool / scheduled batch |
| Set accounting + reporting currency on Ledger | GL > Ledger setup > Ledger | Form tool |

---

#### 1.1.5 Define Posting Policies (Level 3)

**Workshop Questions:**

| # | Question | Answer Drives |
|---|---|---|
| 1 | For each module (AP, AR, Inventory, FA), what GL accounts should transactions post to? | Posting profile account assignments |
| 2 | Do you need different posting accounts per vendor/customer group? | Table/Group/All level on posting profiles |
| 3 | What account should year-end P&L close to? (Retained earnings?) | Accounts for automatic transactions |
| 4 | How do you handle penny rounding differences? | Rounding/penny difference accounts |
| 5 | Do you use posting definitions (advanced) or standard posting profiles? | Posting definitions vs profiles choice |

**Decision Matrix:**

| If Answer | Then Configure |
|---|---|
| Same GL accounts for all vendors | One posting profile with Account code = "All" |
| Different accounts per vendor group (domestic vs foreign) | Multiple posting profiles at Group level |
| Standard posting profiles (most implementations) | Configure posting profiles per module |
| Advanced posting definitions | Enable "Use posting definitions" in GL parameters |

---

#### 1.1.6 Develop Tax Strategy (Level 3)

**Workshop Questions:**

| # | Question | Answer Drives |
|---|---|---|
| 1 | What tax jurisdictions do you operate in? | Tax authority creation |
| 2 | What are your tax filing frequencies per jurisdiction? (Monthly? Quarterly?) | Settlement period setup |
| 3 | What tax rates apply? (List tax types with percentages and effective dates) | Tax code creation with values |
| 4 | How do you group tax codes for application to transactions? | Sales tax group design |
| 5 | Do you have different tax treatment for different product types? (Taxable, exempt, zero-rated) | Item sales tax group design |
| 6 | Do you use an ISV tax engine (Avalara, Vertex)? | If yes: ISV handles rates, D365 handles posting only |
| 7 | Do you have withholding tax requirements? | Withholding tax code/group setup |
| 8 | Do you need tax-exempt tracking for certain customers? | Tax exempt number configuration |
| 9 | US: Do you need 1099 reporting? | 1099 field setup in AP |

**Decision Matrix:**

| If Answer | Then Configure |
|---|---|
| Single tax jurisdiction, simple rates | One authority, one settlement period, few tax codes |
| Multi-jurisdiction (state + city + county) | Multiple tax codes, compound rates, group hierarchy |
| ISV tax engine (Avalara/Vertex) | Install ISV, configure posting groups only, defer rate config to ISV |
| Withholding tax required | Enable withholding tax, create codes and groups |

**Tax Calculation Logic (for agent to explain to customer):**
```
Tax on transaction = Intersection of:
  Sales Tax Group (from customer/vendor) ∩ Item Sales Tax Group (from product)
  = Only tax codes that appear in BOTH groups are applied
```

---

#### 1.1.7 Define Banking Policies (Level 3)

**Workshop Questions:**

| # | Question | Answer Drives |
|---|---|---|
| 1 | How many bank accounts do you have? List: bank name, account #, currency, purpose | Bank account creation |
| 2 | How do you pay vendors? (Check? ACH/NACHA? Wire? SEPA?) | Payment format selection, ER config import |
| 3 | How do you receive customer payments? (Check? Direct debit? Wire?) | Customer payment method setup |
| 4 | How do you reconcile bank statements? (Manual? Import statements?) | Reconciliation setup (standard vs advanced) |
| 5 | What bank statement format? (MT940? BAI2? camt.053? CSV?) | ER import format configuration |
| 6 | Do you use positive pay (check fraud prevention)? | Positive pay format setup |
| 7 | Do you need cash flow forecasting? | Cash flow forecast configuration |

**Decision Matrix:**

| If Answer | Then Configure |
|---|---|
| ACH/NACHA payments (US) | Import NACHA ER config from Dataverse, assign to vendor payment method, set Generic electronic export = Yes |
| SEPA payments (Europe) | Import ISO20022 Credit Transfer ER config |
| Advanced bank reconciliation | Enable advanced recon, configure matching rules, import statement format |
| Standard reconciliation | Basic manual matching |
| Cash flow forecasting | Enable in Cash & Bank parameters, configure liquidity accounts on posting profiles |

---

### 1.2 Manage Budgets (Level 2)

#### 1.2.1 Create and Manage Budgets (Level 3)

**Workshop Questions:**

| # | Question | Answer Drives |
|---|---|---|
| 1 | Do you do budgeting in D365 or an external system? | If external: budget import via DMF. If D365: full budget setup |
| 2 | How many budget models? (Original? Revised? Forecast?) | Budget model creation |
| 3 | Which dimensions do you budget by? | Budget dimension selection (must be in account structures) |
| 4 | How is budget distributed across periods? (Equal? Seasonal? Custom?) | Budget allocation terms |
| 5 | Do you need budget control (prevent overspending)? | Budget control configuration |
| 6 | If budget control: hard stop or soft warning? | Over-budget permissions |
| 7 | Who can override budget limits? | User group permissions |
| 8 | What documents should check budget? (POs? Requisitions? Invoices?) | Document/journal selection in budget control |

---

## 2. Source to Pay (End-to-End Process 05)

### 2.1 Develop Procurement and Sourcing Strategies (Level 2)

#### 2.1.1 Develop Procurement Policies (Level 3)

**Workshop Questions:**

| # | Question | Answer Drives |
|---|---|---|
| 1 | Do employees create purchase requisitions? | Requisition workflow setup |
| 2 | What approval levels for purchases? (Amount-based? Category-based?) | Workflow routing rules |
| 3 | Is there a threshold below which no approval is needed? | Auto-approval rules |
| 4 | Do you use procurement categories? How are they structured? | Procurement category hierarchy |
| 5 | Do you restrict which categories employees can purchase from? | Category access policies |

### 2.2 Manage Supplier Relationships (Level 2)

#### 2.2.1 Create and Manage Vendor Records (Level 3)

**Workshop Questions:**

| # | Question | Answer Drives |
|---|---|---|
| 1 | How do you classify vendors? (Trade, Service, Intercompany, One-time?) | Vendor group creation |
| 2 | What standard payment terms do you use? (Net 30? Net 60? 2% 10 Net 30?) | Payment terms setup |
| 3 | Do you offer or receive early payment discounts? | Cash discount code setup |
| 4 | What default delivery terms (Incoterms) do you use? | Terms of delivery setup |
| 5 | What delivery modes? (Ground, Express, Air, Pickup?) | Modes of delivery setup |
| 6 | Do you track vendor certifications? (Insurance, diversity, quality?) | Vendor certification setup |
| 7 | Do you need vendor approval workflows? (Approve new vendors before use?) | Vendor approval workflow |
| 8 | How many vendors are you migrating? What fields? | Vendor migration planning |

**Decision Matrix:**

| If Answer | Then Configure |
|---|---|
| 3 vendor groups: Trade, Service, Intercompany | Create 3 groups, assign posting profiles to each with appropriate GL accounts |
| Early payment discounts (2% 10 Net 30) | Create cash discount code "2%10N30" with percentage and days |
| Vendor approval required | Configure AP workflow for vendor creation/change approval |
| 50K vendors to migrate | Use VendorsV2 entity via DMF, pre-validate all FKs via OData |

**Configuration Steps:**

| Step | Stage | Menu Path | Entity |
|---|---|---|---|
| Create vendor groups | Base; Gold | AP > Setup > Vendor groups | `VendorGroups` |
| Create payment terms | Base; Gold | AP > Payment setup > Terms of payment | `PaymentTerms` |
| Create cash discounts | Fundamental; Gold | AP > Payment setup > Cash discounts | `CashDiscounts` |
| Create delivery terms | Fundamental; Gold | Procurement > Setup > Distribution > Terms of delivery | `TermsOfDelivery` |
| Create delivery modes | Fundamental; Gold | Procurement > Setup > Distribution > Modes of delivery | `ModesOfDelivery` |
| Create vendor posting profiles | Base; Gold | AP > Setup > Vendor posting profiles | `VendorPostingProfiles` |
| Configure AP parameters | Base; Gold; Early | AP > Setup > AP parameters | Form tool |
| Create vendor payment methods | Base; Gold | AP > Payment setup > Methods of payment | `VendorPaymentMethods` |
| Import ER payment format | Base; Gold | Org admin > Electronic reporting > Configurations | Form tool |
| Configure invoice matching | Fundamental; Gold | AP > Setup > Invoice validation > Matching policies | Form tool |
| Create AP workflows | Fundamental; Gold | AP > Setup > AP workflows | Form tool |

### 2.3 Manage Accounts Payable (Level 2)

#### 2.3.1 Invoice Matching Strategy (Level 3)

**Workshop Questions:**

| # | Question | Answer Drives |
|---|---|---|
| 1 | Do you match invoices to POs? (2-way: PO vs invoice? 3-way: PO vs receipt vs invoice?) | Matching policy |
| 2 | What price tolerance do you allow? (Exact match? 1%? 5%?) | Price tolerance setup |
| 3 | What quantity tolerance? (Must match exactly? Allow 5% over-receipt?) | Quantity tolerance setup |
| 4 | Do you validate invoice totals against PO totals? | Invoice totals tolerance |
| 5 | Do you need invoice automation (scan invoices from email/PDF)? | Invoice capture solution consideration |

---

## 3. Order to Cash (End-to-End Process 02)

### 3.1 Develop Sales Policies (Level 2)

#### 3.1.1 Define Pricing Strategy (Level 3)

**Workshop Questions:**

| # | Question | Answer Drives |
|---|---|---|
| 1 | How do you price products? (Fixed price list? Customer-specific? Volume-based?) | Trade agreement type |
| 2 | Do different customer groups get different prices? | Price groups + customer assignment |
| 3 | What types of discounts? (Line? Total? Multi-line? Rebate?) | Discount setup per type |
| 4 | Are prices date-effective? (Change quarterly? Annually?) | Trade agreement date ranges |
| 5 | Do you need commission tracking for sales reps? | Commission group setup |

**Decision Matrix:**

| If Answer | Then Configure |
|---|---|
| Fixed price list, same for all | Trade agreements at "All customers" level |
| Customer-specific pricing | Trade agreements at "Table" (specific customer) level |
| Customer group pricing | Create price groups, assign to customers, trade agreements at Group level |
| Volume-based discounts | Multiline discount trade agreements with quantity breaks |

### 3.2 Manage Accounts Receivable (Level 2)

#### 3.2.1 Customer Setup (Level 3)

**Workshop Questions:**

| # | Question | Answer Drives |
|---|---|---|
| 1 | How do you classify customers? (Domestic, Export, Government, Retail?) | Customer group creation |
| 2 | Do you manage credit limits? | Credit management setup |
| 3 | What happens when credit limit is exceeded? (Block order? Warning?) | Credit hold rules |
| 4 | Do you send collection letters for overdue invoices? | Collection letter sequence |
| 5 | Do you charge interest on overdue balances? | Interest code setup |
| 6 | What are your aging buckets? (Current, 1-30, 31-60, 61-90, 91+?) | Aging period definition |
| 7 | How do customers pay? (Check? Wire? ACH? Credit card? Direct debit?) | Customer payment methods |
| 8 | Do you need customer approval workflows? | AR workflow setup |
| 9 | What default delivery mode and terms per customer? | Customer master defaults |
| 10 | How many customers migrating? | Migration planning |

**Decision Matrix:**

| If Answer | Then Configure |
|---|---|
| Credit management active | Enable credit management in AR parameters, set default credit limits per group, configure hold rules |
| Collection letters needed | Create collection letter sequence (1st notice → 2nd notice → final notice), set fee amounts |
| Interest charges | Create interest codes with rates and calculation methods |
| Direct debit payments (Europe) | Import SEPA Direct Debit ER config, set up customer bank accounts, create DD mandates |

---

## 4. Inventory to Deliver (End-to-End Process 12)

### 4.1 Define Inventory Policies (Level 2)

#### 4.1.1 Inventory Costing Strategy (Level 3)

**Workshop Questions:**

| # | Question | Answer Drives |
|---|---|---|
| 1 | What costing method for each product category? (FIFO? LIFO? Standard? Moving avg? Weighted avg?) | Item model group selection |
| 2 | Do you use standard costs? If yes, how often do you update them? | Costing version setup |
| 3 | Do you need to track inventory by batch/lot number? | Tracking dimension group |
| 4 | Do you need serial number tracking? (At receipt? At sale? Full lifecycle?) | Tracking dimension group config |
| 5 | How many sites and warehouses? | Site/warehouse creation |
| 6 | Do you need advanced warehouse management (WMS)? For which warehouses? | WMS toggle (ONE-WAY — cannot be reversed!) |
| 7 | Do you allow negative inventory? | Item model group setting |

**⚠️ Critical decisions that CANNOT change later:**
- Item model group (costing method) on a released product — very difficult to change after transactions
- WMS enabled on a warehouse — one-way toggle, CANNOT be reversed
- Dimension group assignment on a product — cannot change after transactions

**Decision Matrix:**

| If Answer | Then Configure |
|---|---|
| Standard cost for manufactured items, FIFO for purchased | Two item model groups: "StdCost" (for MFG) + "FIFO" (for purchased) |
| Batch tracking required | Tracking dimension group with Batch number = Active + Physical/Financial |
| Serial tracking only at sale | Tracking dimension group with Serial = Active, physical only (not financial) |
| Advanced WMS needed | Enable on warehouse BEFORE any transactions — cannot undo |
| Negative inventory allowed | Item model group > Physical negative inventory = Yes |

---

## 5. Plan to Produce (End-to-End Process 04)

### 5.1 Define Manufacturing Strategy (Level 2)

#### 5.1.1 Production Model (Level 3)

**Workshop Questions:**

| # | Question | Answer Drives |
|---|---|---|
| 1 | Discrete manufacturing, process manufacturing, or both? | Module selection (Production control vs Process manufacturing) |
| 2 | Do you use BOMs (bills of materials)? How many levels deep? | BOM structure design |
| 3 | Do you use routes (operation sequences)? | Route configuration |
| 4 | How many resources (machines, people, work centers)? | Resource setup |
| 5 | How do you schedule production? (Operations scheduling? Job scheduling?) | Scheduling method selection |
| 6 | Do you need capacity planning? | Resource capacity calendars |
| 7 | Do you subcontract any operations to external vendors? | Vendor resource type |
| 8 | How do you report production? (Start/end per operation? Per order?) | Route group auto-report settings |
| 9 | Do you use co-products or by-products? | Formula setup (process manufacturing) |
| 10 | Do you track batch attributes (potency, grade, concentration)? | Batch attribute setup |

---

## 6. Acquire to Dispose (End-to-End Process 08)

### 6.1 Define Asset Policies (Level 2)

#### 6.1.1 Asset Classification and Depreciation (Level 3)

**Workshop Questions:**

| # | Question | Answer Drives |
|---|---|---|
| 1 | What types of fixed assets? (Buildings, vehicles, equipment, furniture, IT, leasehold?) | Fixed asset group creation |
| 2 | What is your capitalization threshold? (Minimum value to be a fixed asset) | FA parameters |
| 3 | What depreciation method per asset type? (Straight line? Reducing balance? MACRS?) | Depreciation profile setup |
| 4 | Do you need separate book and tax depreciation? | Value model (book) + depreciation book (tax) |
| 5 | What useful life per asset category? | Default service life per FA group |
| 6 | How do you acquire assets? (Purchase through PO? Direct acquisition? Transfer from inventory?) | Acquisition method configuration |
| 7 | How many assets are you migrating? | FA migration planning (asset master + opening balances) |

---

## Workshop Framework — Complete Question Hierarchy

### Phase 1: Discovery Questions (Open-Ended)

Asked first to understand current state before making design decisions:

| Area | Discovery Question |
|---|---|
| General | Walk me through your month-end close process. What are the steps? |
| General | What reports do you produce for leadership? What dimensions do they slice by? |
| General | What are your biggest pain points with your current system? |
| GL | How is your current chart of accounts structured? Show me your trial balance. |
| GL | What financial dimensions do you report by today? |
| AP | How do you receive and process vendor invoices today? |
| AP | How do you make vendor payments? What formats? |
| AR | How do you invoice customers? What triggers an invoice? |
| AR | How do you manage collections for overdue accounts? |
| Tax | What tax jurisdictions do you file in? |
| Inventory | How do you value inventory? What costing method? |
| Inventory | Do you track lots/batches/serial numbers? |
| Production | What does your manufacturing process look like? |
| FA | How do you track and depreciate fixed assets today? |

### Phase 2: Design Questions (Specific — Drive Configuration)

These are the questions from the sections above, organized by BPC Level 2 area. Each answer directly maps to a configuration decision.

### Phase 3: Validation Questions (Confirm Before Configuring)

Asked after initial design to confirm understanding:

| Area | Validation Question |
|---|---|
| GL | "So to confirm: you want a shared chart of accounts with 5-digit account numbers, 3 dimensions (Dept, CC, Project), and the structure where BS accounts get Dept+CC and P&L gets Dept+CC+Project. Correct?" |
| AP | "You want 3 vendor groups (Trade, Service, Intercompany), Net 30 as default terms, 3-way matching for inventory items and 2-way for services, and ACH/NACHA for electronic payments. Correct?" |
| AR | "You want credit management with a $50K default limit, warning (not block) when exceeded, collection letters after 30/60/90 days, and 1% interest on overdue balances. Correct?" |
| Tax | "You'll use Avalara for tax calculation, so we configure the D365 posting groups but Avalara handles rates. Correct?" |
| Inventory | "Standard cost for manufactured items, FIFO for purchased items, batch tracking on raw materials, serial tracking on finished goods at sale only. Correct?" |

---

## Output: From Answers to Configuration Plan

After completing workshops, the agent generates a **Configuration Plan** — a sequenced list of every configuration step needed, with:

| Column | Content |
|---|---|
| Sequence # | Order to execute (follows Microsoft's config template levels) |
| BPC Reference | Level 1 > Level 2 > Level 3 process |
| Configuration Item | What to configure |
| Workshop Answer | The customer's answer that drives this config |
| Menu Path | Where in D365 |
| Data Entity | For DMF import |
| Stage | Base/Fundamental/Optional |
| Owner | Who's responsible |
| Status | Not started/In progress/Complete/Validated |
| Dependencies | What must be done first |

### Example Configuration Plan (excerpt for GL):

| Seq | BPC | Config Item | Answer Drives It | Menu Path | Entity | Stage |
|---|---|---|---|---|---|---|
| 1 | 13.1.2 | Fiscal calendar: Jan-Dec, 12 periods | "Our fiscal year is calendar year" | GL > Calendars | `FiscalCalendars` | Base |
| 2 | 13.1.3 | Chart of accounts: shared, 5-digit | "All 3 entities share one CoA" | GL > Chart of accounts | `ChartOfAccounts` | Base |
| 3 | 13.1.3 | Main account categories: GAAP aligned | "We report under US GAAP" | GL > Main account categories | `MainAccountCategories` | Base |
| 4 | 13.1.3 | 3 dimensions: Dept (entity-backed), CC (entity-backed), Project (entity-backed) | "We report by department, cost center, and project" | GL > Dimensions | `FinancialDimensions` | Base |
| 5 | 13.1.3 | Account structure 1: BS + Dept + CC | "BS accounts need dept and cost center" | GL > Structures | Form tool | Base |
| 6 | 13.1.3 | Account structure 2: P&L + Dept + CC + Project | "P&L also needs project" | GL > Structures | Form tool | Base |
| 7 | 13.1.3 | Financial dim config for integration: ledger format + default format | "We will import data via DMF" | GL > Dimensions > Config for integration | Form tool | Base |
| 8 | 13.1.4 | Currencies: USD (accounting), EUR, GBP (transactional) | "We transact in USD, EUR, and GBP" | GL > Currencies | `Currencies` | Base |
| 9 | 13.1.4 | Exchange rates from OANDA, daily | "We update rates daily from OANDA" | GL > Exchange rate providers | Form tool | Fundamental |
| 10 | 13.1.3 | Ledger setup: assign CoA, calendar, USD/EUR to each entity | Per-entity decisions | GL > Ledger setup > Ledger | Form tool | Base |

---

## Mapping: BPC Process → Skill → Config Agent

| BPC Level 2 Area | BPC Level 3 Processes | Platform Skill | Config Agent |
|---|---|---|---|
| 13.1 Define accounting policies | Company structure, periods, CoA, currency, posting, tax, banking, costing, assets | `fo-org-admin`, `fo-gl-chart-of-accounts`, `fo-tax-configuration`, `fo-cash-bank-management`, `fo-gl-posting-profiles` | GL Agent |
| 13.2 Manage cash | Bank accounts, reconciliation, cash management | `fo-cash-bank-management` | Bank Agent |
| 13.3 Manage budgets | Budget models, control, plans | `fo-budgeting` | Budget Agent |
| 13.4 Record financial transactions | Journals, allocations, intercompany | `fo-gl-journals` | GL Agent |
| 13.5 Close financial periods | Period close, revaluation, consolidation, year-end | `fo-period-close` | GL Agent |
| 13.6 Analyze financial performance | Financial reporting, cash flow analysis | `fo-financial-reporting` | Reporting Agent |
| 05.1 Develop procurement strategies | Policies, categories, requisition workflows | `fo-procurement` | Procurement Agent |
| 05.2 Manage supplier relationships | Vendor records, evaluation, collaboration | `fo-accounts-payable` | AP Agent |
| 05.3 Procure goods and services | PO creation, confirmation, receipt | `fo-procurement` | Procurement Agent |
| 05.4 Manage accounts payable | Invoicing, matching, payment | `fo-accounts-payable` | AP Agent |
| 02.1 Develop sales policies | Pricing, channels, discounts | `fo-sales-marketing` | Sales Agent |
| 02.2 Manage sales orders | SO processing, delivery, returns | `fo-sales-marketing` | Sales Agent |
| 02.3 Manage accounts receivable | Invoicing, collections, credit | `fo-accounts-receivable` | AR Agent |
| 12.1 Define inventory policies | Costing, tracking, warehousing | `fo-inventory-management` | Inventory Agent |
| 04.1 Define manufacturing strategy | BOM, routes, scheduling, resources | `fo-production-control` | Production Agent |
| 08.1 Define asset policies | Groups, depreciation, value models | `fo-fixed-assets` | FA Agent |

---

## Workshop Templates

Microsoft provides downloadable Word templates for BPC workshops at:
**https://aka.ms/businessprocessworkshops**

These templates include:
- Storyboard diagrams per end-to-end process
- Stakeholder lists
- Agenda templates
- Discussion questions
- Input/output definitions

### Our Platform Enhancement

Our platform's Discovery Agent should:
1. Load the BPC workshop template for the relevant end-to-end process
2. Present the storyboard diagram 
3. Walk through each Level 3 subprocess with the workshop questions from this document
4. Record answers in structured format (not free text)
5. Automatically generate the Configuration Plan from answers
6. Feed the Configuration Plan to the appropriate Config Agent
7. Config Agent executes the plan via MCP server tools
8. Validation Agent verifies configuration matches answers

This closes the loop: **Requirement → Question → Answer → Config → Validation → Done.**
