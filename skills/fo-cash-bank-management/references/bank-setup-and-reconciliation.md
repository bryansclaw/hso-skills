# Bank Setup and Reconciliation

Source: Microsoft Learn + verified research — 2026-03-18

## Bank Account Setup

Navigation: **Cash and bank management > Bank accounts > Bank accounts**

### Required Fields
| Field | Description |
|---|---|
| Bank account ID | Unique identifier |
| Name | Display name |
| Bank group | Classification (domestic, international, payroll) |
| Routing number | Bank routing/sort code |
| Bank account number | Account number at the bank |
| SWIFT code | International bank identifier |
| IBAN | International bank account number (Europe) |
| Main account | **GL main account this bank maps to** — this is the critical link to GL |
| Currency | Account currency |

### Electronic Payment Format Setup (ER)

To generate payment files (NACHA, SEPA, etc.):

1. **Import ER configuration:**
   - Menu: Organization administration > Electronic reporting > Configurations
   - Click Exchange > Load from Dataverse repository (replaces LCS for new implementations)
   - Search for the required format (e.g., "NACHA", "ISO20022 Credit Transfer")
   - Import the configuration

2. **Assign to payment method:**
   - Menu: Accounts payable > Payment setup > Methods of payment
   - Select the payment method (e.g., "Electronic")
   - File format FastTab: Set "Generic electronic export format" = Yes
   - Select the imported ER configuration in "Export format configuration" field

3. **Assign to bank account (if bank-specific format):**
   - On the bank account record, File format FastTab
   - Assign the ER configuration

### Common Payment Formats

| Region | Format | ER Configuration Name | Use |
|---|---|---|---|
| US | ACH/NACHA | NACHA (multiple variants) | Domestic electronic vendor payments |
| US | Positive Pay | Positive Pay | Check fraud prevention file to bank |
| Europe | SEPA Credit Transfer | ISO20022 Credit Transfer | Euro vendor payments |
| Europe | SEPA Direct Debit | ISO20022 Direct Debit | Customer collection via bank |
| Global | Wire Transfer | Various bank-specific | International payments |

## Bank Reconciliation

### Standard Reconciliation
- Menu: Cash and bank management > Bank accounts > [account] > Reconcile > Bank reconciliation
- Manual matching of bank statement lines to D365 bank transactions
- Mark matched lines, post remaining as adjustments

### Advanced Bank Reconciliation
- Menu: Cash and bank management > Setup > Advanced bank reconciliation setup
- **Matching rules:** Define automatic matching criteria:
  - Match by amount + date
  - Match by reference number
  - Match by check number
  - Tolerance amounts for rounding differences
- **Bank statement format:** Configure import format for bank files (MT940, BAI2, camt.053)
- **Statement import:** Upload bank statements and auto-match against D365 transactions
- **Unmatched items:** Create adjustment journals for bank charges, interest, etc.

### Modern Bank Reconciliation (v10.0.44+)
New features:
- Automated transaction matching (more intelligent matching algorithms)
- Direct subledger postings from reconciliation (post adjustments without separate journal)
- Configurable descriptions for GL postings from bank adjustments
- Improved performance for high-volume reconciliation

## Cash Flow Forecasting

Navigation: **Cash and bank management > Setup > Cash flow forecasting**

- AI-driven cash flow forecasts (included in D365 Finance license)
- Compare forecasts to snapshots and actuals
- Sources: AP open invoices, AR open invoices, budget, project forecasts
- Integrates with BPA for MCP Analytics server queries
- Requires: Settle account configured on posting profiles (for liquidity account mapping)

## Bank Statement Import Formats

| Format | Standard | Region | Configuration |
|---|---|---|---|
| MT940 | SWIFT | Global | ER configuration import |
| BAI2 | Banking Administration Institute | US | ER configuration import |
| camt.053 | ISO 20022 | Europe/Global | ER configuration import |
| CSV | Custom | Any | Custom ER format or direct mapping |

## Validation Checks After Setup

- [ ] Each bank account has a valid GL main account assigned
- [ ] Bank account currency matches or is supported
- [ ] ER payment format imported from Dataverse repository
- [ ] Payment method has ER format assigned (File format FastTab)
- [ ] Reconciliation matching rules defined (if using advanced reconciliation)
- [ ] Bank statement import format configured
- [ ] Cash flow forecasting setup complete (if using)
- [ ] Test: Create payment journal > generate payment file > verify file format is correct
- [ ] Test: Import bank statement > run matching > verify matched transactions
