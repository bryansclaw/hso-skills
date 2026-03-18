# Inventory Close and Costing

Source: Microsoft Learn (inventory-close) — verified 2026-03-18
Navigation: **Cost management > Periodic > Closing and adjustment**

## What Inventory Close Does

The inventory close process settles issue transactions (sales, consumption) to receipt transactions (purchases, production), based on the inventory valuation method selected in the item's item model group.

**Key behaviors:**
- Adjusts the general ledger to reflect cost settlements
- After close, you CANNOT post in periods before the closing date (unless you reverse the close)
- Running average cost price is used until close; close calculates final cost
- Applies to both item and service inventory types

## When to Run

- Most companies run inventory close as part of **month-end close and reconciliation**
- Frequency depends on transaction volume
- **Required step** for all inventory models EXCEPT moving average
- If you try to close a financial period without running inventory close, D365 warns you
- **Run during off-peak hours** to distribute computing resources

## Inventory Close vs Recalculation

| Feature | Inventory Close | Inventory Recalculation |
|---|---|---|
| Creates settlements | Yes | No |
| Adjusts inventory values | Yes | Yes |
| Adjusts GL | Yes (optional) | Yes |
| Locks prior periods | Yes (can't post before close date) | No |
| When to use | Month-end close | Mid-period adjustments needed |

## What Happens During Close

1. System identifies all issue transactions for the period
2. Matches issues to receipts based on the costing method:
   - **FIFO:** First receipts matched to first issues
   - **LIFO:** Last receipts matched to first issues
   - **Weighted average:** Average of all receipts in period
   - **Standard cost:** Issues valued at standard; variance calculated
3. If actual receipt cost ≠ estimated cost on issue → adjustment posted
4. GL updated with adjustment amounts (if "Post to GL" selected)
5. Settlement created linking specific issue to specific receipt

## Critical Rule About GL Account Changes

**From Microsoft:** Adjustments from inventory close are posted to the ORIGINAL ledger accounts used when the transaction was posted — NOT to the current posting profile accounts. If you changed posting profiles since the original transaction, the adjustment still goes to the old accounts.

## Inventory Close Log

After close completes:
- Check message center for warnings
- Warning: "unit cost price might be incorrect because a transaction could not be fully settled" — means there are unmatched issues (no receipt to settle against)
- Review on **Closing and adjustment** page > Overview tab

## Reversing Inventory Close

If close needs to be undone:
- Go to Closing and adjustment page
- Select the close run
- Click **Reversal**
- This reopens the period for posting

## Period-End Checklist (Inventory)

1. Ensure all purchase receipts are financially updated (invoiced)
2. Ensure all sales shipments are financially updated (invoiced)
3. Ensure all production orders are ended (cost calculated)
4. Run inventory recalculation if mid-period adjustments needed
5. Run inventory close for the period
6. Verify: Inventory valuation report = GL inventory control accounts
7. If mismatch → check posting profiles and adjustment postings
