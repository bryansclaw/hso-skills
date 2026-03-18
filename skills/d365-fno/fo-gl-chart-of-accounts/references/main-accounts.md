# Create and Configure Main Accounts

Source: Microsoft Learn — verified 2026-03-18
Navigation: **General ledger > Chart of accounts > Accounts > Main accounts**

## Create a Main Account

1. Go to **General ledger > Chart of accounts > Accounts > Main accounts**
2. Click **New**
3. Enter **Main account** number (the GL account number)
4. Enter **Name** (description)
5. Select **Main account type** — this determines how the account behaves:

### Main Account Types

| Type | Behavior | Financial Statement |
|---|---|---|
| **Profit and loss** | P&L posting account — records revenue and expenses | Income Statement |
| **Revenue** | Same function as P&L — tracks revenue specifically | Income Statement |
| **Expense** | Same function as P&L — tracks expenses specifically | Income Statement |
| **Balance sheet** | Transaction account — records what entity owns/owes | Balance Sheet |
| **Asset** | Same as Balance sheet — tracks assets | Balance Sheet |
| **Liability** | Same as Balance sheet — tracks liabilities | Balance Sheet |
| **Equity** | Same as Balance sheet — tracks equity | Balance Sheet |
| **Total** | Display-only — shows sum of an account interval range. Configure via Account interval page. Cannot be suspended (simulate with Active from/to dates). | Reporting |
| **Reporting** | Financial statement reporting for Brazil only | Brazil localization |

**Key:** Revenue + Expense accounts have the same function as Profit and loss accounts. The distinction is for reporting/categorization.

6. Select **Account category** — links account to default financial reports and Power BI dashboards
7. Set **Default currency** (optional — restricts which currencies can post to this account)
8. Set **Debit/credit default** — determines which side is the normal balance

## Legal Entity Overrides Section
- Click **Add** to select a legal entity
- Check **Suspended** to prevent posting to this account in that entity
- Set **Active from** / **Active to** dates to time-limit the account

## Financial Reporting Section
- Set **Exchange rate type** for currency translation
- Set **Currency translation type** — method for calculating exchange rates for this account in financial reports

## Important Rules
- **Do NOT use the delimiter character in account numbers.** If your delimiter is "-", don't create account "1000-01" — this causes parsing issues when the system splits account + dimension strings.
- Main account number can only exist in ONE account structure assigned to the ledger (no overlapping ranges between structures).
- After transactions post to an account, the account type cannot be changed.
- Main accounts link to posting profiles — changing or deleting an account that's referenced in a posting profile blocks related configuration changes.

## Data Entity
- **Entity name:** `MainAccounts`
- **MCP approach:** Data tool (`data_create_entities`) for bulk creation
- **Key fields in entity:** MainAccountId, Name, MainAccountType, MainAccountCategory, DebitCreditDefault, CurrencyCode
