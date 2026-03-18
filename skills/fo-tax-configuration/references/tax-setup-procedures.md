# Tax Setup Procedures

Source: Microsoft Learn (set-up-sales-tax-codes, set-up-sales-tax-groups-item-sales-tax-groups) — verified 2026-03-18

## Create Sales Tax Codes

Navigation: **Tax > Indirect taxes > Sales tax > Sales tax codes**

1. Select **New**
2. Enter **Sales tax code** (identifier)
3. Enter **Name** (description)
4. Select **Settlement period** — determines which tax authority and reporting/payment intervals
5. Select **Ledger posting group** — determines which GL accounts receive tax postings
6. Expand **Calculation** FastTab:
   - **Origin** field: if "Amount per unit" → value is multiplied by quantity. Otherwise value is a percentage.
7. On Action Pane, select **Sales tax code** > **Values**
8. Enter the tax rate in the **Value** column
9. Save

**Tax types (v10.0.22+ with Tax service):**
- Standard VAT
- Reduced VAT
- VAT 0%
- Excise
- Other

## Create Sales Tax Groups

Navigation: **Tax > Indirect taxes > Sales tax > Sales tax groups**

Sales tax groups are attached to **customers, vendors, and ledger accounts for transactions**.

1. Click **New**
2. Enter **Sales tax group** code
3. Enter **Description**
4. Expand **Setup** section
5. Click **Add**
6. Select **Sales tax code(s)** to include in the group
7. Save

**Assignment:** Sales tax groups are assigned to customers, vendors, and on transaction headers.

## Create Item Sales Tax Groups

Navigation: **Tax > Indirect taxes > Sales tax > Item sales tax groups**

Item sales tax groups are attached to **products/resources**.

1. Click **New**
2. Enter **Item sales tax group** code
3. Enter **Description**
4. Click **Add**
5. Select **Sales tax code(s)** to include
6. Save

**Assignment:** Item sales tax groups are assigned to released products and on transaction lines.

## How Tax Calculation Works

**From Microsoft:** "Sales tax can be calculated only if a sales tax group AND an item sales tax group are selected for each transaction for which sales tax must be calculated."

```
Tax applied to a transaction = INTERSECTION of:
  Sales Tax Group (on header — from customer/vendor)
  ∩
  Item Sales Tax Group (on line — from product)
  =
  Tax codes that appear in BOTH groups
```

**Example:**
- Sales tax group "DOMESTIC" contains: TAX-STATE (6%), TAX-CITY (2%)
- Item sales tax group "TAXABLE" contains: TAX-STATE (6%), TAX-CITY (2%), TAX-LUXURY (5%)
- Transaction gets: TAX-STATE + TAX-CITY (intersection) = 8% total
- TAX-LUXURY is NOT applied because it's not in the sales tax group

**If intersection is empty:** No tax is calculated. This is a common error — "Tax code not found for this combination."

## Ledger Posting Groups

Navigation: **Tax > Setup > Sales tax > Ledger posting groups**

Defines WHICH GL accounts tax transactions post to:

| Account | Purpose |
|---|---|
| Sales tax payable | Credit when collecting tax (sales) |
| Sales tax receivable | Debit when paying tax (purchases) |
| Use tax expense | Expense for use tax (US-specific) |
| Use tax payable | Liability for use tax |
| Settlement account | Account used during tax settlement process |

## Settlement Periods

Navigation: **Tax > Indirect taxes > Sales tax > Sales tax settlement periods**

Defines the calendar for tax filing:
- Links to a tax authority (which auto-creates a vendor for payment)
- Period intervals: Monthly, Quarterly, Annual, or custom
- Used to run the "Settle and post sales tax" periodic process

## Settlement Process

Navigation: **Tax > Declarations > Sales tax > Settle and post sales tax**

1. Select settlement period
2. Select date range
3. System calculates net tax payable/receivable
4. Posts settlement journal (clears payable/receivable to settlement account)
5. Creates a vendor invoice for the tax authority
