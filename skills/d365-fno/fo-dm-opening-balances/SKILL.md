---
name: fo-dm-opening-balances
description: >
  Migrate D365 F&O opening balances (Stage 3) — GL trial balance, open AP
  invoices, open AR invoices, inventory on-hand, fixed asset balances, and
  bank balances. Covers the critical subledger-first migration principle,
  control account reconciliation, journal-based import methods, and post-
  migration validation including automated account reconciliation.
  Use when: importing financial opening balances at cutover, performing mock
  cutover migrations, or reconciling migrated balances.
  DEPENDS ON: fo-dm-master-data (Stage 2 must be complete — vendors, customers,
  products, assets must exist before posting balances against them).
metadata:
  platform:
    category: "fo-data-migration"
    riskLevel: "critical"
    configSequence: 0
    configLevel: "cross-cutting"
    requires:
      products: ["F&O"]
      skills: ["fo-dm-master-data", "fo-gl-journals"]
    bpcProcess: "13 - Record to Report"
    msLearnPath: "https://learn.microsoft.com/en-us/dynamics365/finance/general-ledger/pstg-prfles-ovrvw"
---

# Opening Balance Migration (Stage 3)

## ⚠️ FUNDAMENTAL RULE

**Control account balances CANNOT be posted directly to the general ledger.**

Control accounts (AP, AR, Inventory, FA, Bank) are populated automatically when you post through their respective subledgers. If you post directly to a control account via GL journal, the subledger will NOT have matching entries, causing permanent reconciliation mismatches.

**Correct approach:** Post subledger journals FIRST → control accounts populated automatically → then post remaining GL balances EXCLUDING control accounts.

## Migration Sequence

```
STEP 1: Subledger Opening Balances (posted through module journals)
│
├── 1A: AP Open Invoices
│   └── Posts to: AP control account (GL) + Vendor subledger
│
├── 1B: AR Open Invoices
│   └── Posts to: AR control account (GL) + Customer subledger
│
├── 1C: Inventory On-Hand
│   └── Posts to: Inventory control accounts (GL) + Item ledger
│
├── 1D: Fixed Asset Balances
│   └── Posts to: FA control accounts (GL) + Asset books
│
└── 1E: Bank Balances
    └── Posts to: Bank GL accounts + Bank subledger

STEP 2: General Ledger Opening Balance
│
├── Import remaining GL account balances
├── EXCLUDE: AP control, AR control, Inventory control, FA control, Bank
└── These are already populated by Step 1

STEP 3: Reconciliation
│
├── AP subledger total = AP control account in GL ✓
├── AR subledger total = AR control account in GL ✓
├── Inventory valuation = Inventory control in GL ✓
├── FA net book value = FA control accounts in GL ✓
├── Bank subledger = Bank GL accounts ✓
└── GL trial balance = 0 (debits = credits) ✓
```

## Step 1A: AP Open Invoices

### Method: Vendor Invoice Journal
- **Journal type:** Vendor invoice register or General journal with AccountType=Vendor
- **Menu:** Accounts payable > Invoices > Invoice journal
- **Import entity:** `LedgerJournalEntity` with `ACCOUNTTYPE=Vendor`

### Required Fields per Line
| Field | Value | Notes |
|---|---|---|
| JOURNALNAME | AP opening balance journal name | Must be pre-configured |
| ACCOUNTTYPE | Vendor | |
| ACCOUNTDISPLAYVALUE | Vendor account number | Must exist in vendor master |
| TRANSDATE | Cutover date (or original invoice date) | |
| VOUCHER | Unique voucher per invoice | |
| INVOICE | Original invoice number | For matching/identification |
| DEBIT or CREDIT | Invoice amount | Credit = vendor owes us (unusual); Debit = we owe vendor (normal for AP) |
| CURRENCYCODE | Transaction currency | |
| OFFSETACCOUNTTYPE | Ledger | |
| OFFSETACCOUNTDISPLAYVALUE | Opening balance equity/clearing account | NOT the AP control account |
| DUEDATE | Payment due date | For aging accuracy |

### Offset Account Strategy
Use a **temporary clearing/opening balance account** in equity as the offset — NOT the expense accounts from the original transactions. The opening balance is a point-in-time snapshot, not a re-creation of original postings.

**Example:**
```
Debit:  Opening Balance Clearing (equity)    $50,000
Credit: Vendor A (AP control via subledger)  $20,000
Credit: Vendor B (AP control via subledger)  $15,000
Credit: Vendor C (AP control via subledger)  $15,000
```

### Result
- AP subledger: 3 open invoices with due dates and aging
- GL: AP control account credited $50,000, clearing account debited $50,000
- AP aging report will show correct aged balances

## Step 1B: AR Open Invoices

### Method: Customer Payment Journal or Free Text Invoice Import
- **Option A:** General journal with AccountType=Customer
- **Option B:** Import free text invoices via `FreeTextInvoiceHeaderEntity` + `FreeTextInvoiceLineEntity` (creates proper invoice documents)
- **Option A is simpler** for opening balances; Option B gives full invoice documents

### Required Fields (General Journal method)
Same structure as AP but:
| Field | Value |
|---|---|
| ACCOUNTTYPE | Customer |
| ACCOUNTDISPLAYVALUE | Customer account number |
| DEBIT or CREDIT | Debit = customer owes us (normal for AR) |
| OFFSETACCOUNTTYPE | Ledger |
| OFFSETACCOUNTDISPLAYVALUE | Opening balance clearing account |
| DUEDATE | For aging accuracy |

### Result
- AR subledger: open invoices per customer with aging
- GL: AR control account debited, clearing account credited
- AR aging report accurate

## Step 1C: Inventory On-Hand

### Method: Inventory Movement Journal
- **Menu:** Inventory management > Journal entries > Item counting > Counting
- **Or:** Inventory management > Journal entries > Item movement > Movement
- **Data entity:** `InventoryMovementJournalHeaders` + `InventoryMovementJournalLines`

### Required Fields
| Field | Value | Notes |
|---|---|---|
| Item number | Product ID | Must be released product |
| Site | Site ID | Required |
| Warehouse | Warehouse ID | Required |
| Location | Location ID | If WMS enabled |
| Batch/Serial | If tracked | Per dimension group requirements |
| Quantity | On-hand quantity | |
| Cost price | Unit cost | **Critical for standard cost items — must match costing version** |
| Offset account | Opening balance clearing | |

### Standard Cost Items
- **Must set standard cost BEFORE importing on-hand**
- If cost not set → variance calculated against $0 → wrong cost in system
- Set cost via: Cost management > Costing versions > [version] > Item prices
- Or import via `InventoryItemPrices` entity

### Result
- Item ledger: on-hand quantities per site/warehouse/location/batch
- GL: Inventory control account debited, clearing account credited
- Inventory valuation report matches GL

## Step 1D: Fixed Asset Balances

### Method: FA Acquisition Journal
- **Menu:** Fixed assets > Journal entries > Fixed assets journal
- **Journal type:** Fixed assets (Acquisition transaction type)

### Two Approaches

**Approach A: Net Book Value Only (simpler)**
- Import each asset with: Asset ID, Acquisition amount = current NBV
- One acquisition entry per asset
- Pro: Simple, fast
- Con: No historical depreciation detail

**Approach B: Original Cost + Accumulated Depreciation (more accurate)**
- Import acquisition at original cost
- Then import accumulated depreciation via depreciation journal
- Pro: Full history, accurate reporting
- Con: More complex, two imports per asset

### Required Fields (Acquisition)
| Field | Value |
|---|---|
| Fixed asset number | Asset ID (from master data) |
| Value model | Book (or tax) |
| Transaction type | Acquisition |
| Date | Cutover date (or original acquisition date) |
| Amount | Acquisition cost (or NBV for Approach A) |
| Offset account | Opening balance clearing |

### Result
- FA books: assets with acquisition cost and accumulated depreciation
- GL: FA control accounts debited, clearing account credited

## Step 1E: Bank Balances

### Method: Bank Journal or General Journal
- General journal with AccountType=Bank
- Simple: one line per bank account

| Field | Value |
|---|---|
| ACCOUNTTYPE | Bank |
| ACCOUNTDISPLAYVALUE | Bank account ID |
| DEBIT | Bank balance amount |
| OFFSETACCOUNTTYPE | Ledger |
| OFFSETACCOUNTDISPLAYVALUE | Opening balance clearing |

## Step 2: GL Opening Balance (Remaining Accounts)

### What to Include
- Equity accounts (retained earnings, paid-in capital)
- Prepaid expense accounts
- Accrued liability accounts
- Intercompany accounts
- Any other balance sheet accounts NOT controlled by a subledger
- P&L accounts: usually NOT migrated (start fresh) unless mid-year cutover

### What to EXCLUDE
- AP control account(s) — already populated by Step 1A
- AR control account(s) — already populated by Step 1B
- Inventory control account(s) — already populated by Step 1C
- FA control accounts (acquisition, accumulated depreciation) — already populated by Step 1D
- Bank accounts — already populated by Step 1E
- Tax payable/receivable — usually migrated via clearing or separate tax journal

### Method
- General journal import via `LedgerJournalEntity`
- ACCOUNTTYPE = Ledger
- Use ledger dimension format for account + dimensions
- Each line: Main account + dimensions, debit or credit, transaction date
- **Net total of ALL Step 2 entries must balance to exactly offset the clearing account from Steps 1A-1E**

### The Clearing Account Must Net to Zero

After ALL opening balance entries (Steps 1A through 2):
```
Opening Balance Clearing Account:
  Debited by: AP invoices (offset)           $  50,000
  Debited by: AR invoices (offset)          ($ 75,000) ← credit, so net debit reduced
  Debited by: Inventory (offset)             $ 200,000
  Debited by: Fixed assets (offset)          $ 500,000
  Debited by: Bank (offset)                  $ 100,000
  Credited by: GL remaining accounts         $ 775,000
  ──────────────────────────────────────────
  Net balance:                               $       0 ✓
```

If the clearing account doesn't net to zero → the legacy trial balance was wrong or something was missed.

## Step 3: Reconciliation

### Automated Reconciliation (v10.0.44+)
- **Menu:** General ledger > Periodic tasks > Account reconciliation
- Automatically reconciles GL with AP, AR, tax, and bank subledgers
- **Our Validation Agent should run this after every migration attempt**

### Manual Reconciliation Checklist

| Check | Source Report | GL Account | Must Match |
|---|---|---|---|
| AP total | Vendor balance report (aging) | AP control account | Exactly |
| AR total | Customer balance report (aging) | AR control account | Exactly |
| Inventory value | Inventory valuation report | Inventory control account(s) | Exactly |
| FA NBV | FA net book value report | FA asset - accumulated depreciation | Exactly |
| Bank | Bank balance per bank account | Bank GL accounts | Exactly |
| Trial balance | GL trial balance | All accounts | Debits = Credits |
| Clearing account | GL trial balance detail | Opening balance clearing | = $0 |

### Reconciliation Failures — Common Causes
| Issue | Likely Cause | Resolution |
|---|---|---|
| AP subledger ≠ GL control | Invoices posted to wrong account or some posted directly to GL | Reverse and re-post through subledger |
| AR subledger ≠ GL control | Same as above for AR | |
| Inventory ≠ GL | Posting profile changed between import batches | Verify posting profile consistency |
| FA ≠ GL | Depreciation posted without acquisition | Check asset book transaction history |
| TB doesn't balance | Missing entries or wrong debit/credit | Review clearing account for net position |
| Clearing ≠ 0 | Legacy TB didn't balance or entity missed | Identify gap amount and trace to source |

## Mock Cutover

### Microsoft's Recommendation
*"Do a mock period close and reconcile each of your subledgers to the ledger before your go-live. Also do a mock cutover of all open balances and open transactions before your initial go-live."*

### Mock Cutover Process
1. Restore clean golden config to migration environment
2. Import Stage 1 config packages
3. Import Stage 2 master data
4. Import Stage 3 opening balances (all steps 1A-2)
5. Run Step 3 reconciliation
6. Log time taken per stage
7. Log errors encountered and resolution
8. Compare to cutover window — does it fit?
9. Identify improvements for next mock
10. **Run at least 2-3 mock cutovers before go-live**

### Cutover Timing Template
| Activity | Owner | Est. Duration | Actual Duration | Notes |
|---|---|---|---|---|
| Freeze source system | Customer | — | — | No transactions after this point |
| Extract final data | Migration team | 2h | | |
| Transform data | Migration team | 4h | | |
| Import config (Stage 1) | Migration team | 1h | | If not already in prod |
| Import master data (Stage 2) | Migration team | 4h | | Incremental since last load |
| Import opening balances (Stage 3) | Migration team | 4h | | Full re-import |
| Reconciliation (Step 3) | Finance lead | 2h | | Must pass all checks |
| Go/No-go decision | Steering committee | 1h | | Based on reconciliation |
| User validation | Key users | 2h | | Spot checks |
| Go-live confirmation | Project manager | — | — | |
| **Total estimated** | | **20h** | | Plan for weekend cutover |

## Related Skills
- **Previous stage:** `fo-dm-master-data` (Stage 2)
- **General reference:** `fo-data-migration`
- **GL journals:** `fo-gl-journals` (import entity details)
- **Posting profiles:** `fo-gl-posting-profiles` (control account mapping)
- **Period close:** `fo-period-close` (mock period close during mock cutover)
