# Invoice Matching — Two-Way and Three-Way

Source: Microsoft Learn (three-way-matching-policies) — verified 2026-03-18

## Matching Types

### Two-Way Matching
Compares: **Purchase order** ↔ **Vendor invoice**
- Validates: unit price on invoice matches PO price within tolerance
- Does NOT check receipt quantities
- Use when: services, items where receipt verification isn't critical

### Three-Way Matching
Compares: **Purchase order** ↔ **Product receipt** ↔ **Vendor invoice**
- Validates: unit price on invoice matches PO price within tolerance
- Also validates: quantity on invoice matches quantity RECEIVED (from product receipt)
- Use when: physical goods, fixed assets, items where quantity verification matters

## Where to Set Matching Policy

Matching policy can be set at multiple levels (most specific wins):
1. **Legal entity level** — AP parameters (default for all)
2. **Vendor level** — per vendor record
3. **Item level** — per released product
4. **Purchase order header level** — per individual PO
5. **Purchase order line level** — per individual line

## Configuration Steps

### Set Legal Entity Default
- **Menu:** Accounts payable > Setup > Accounts payable parameters
- **Field:** Matching policy (under Line matching policy section)
- Options: Not required, Two-way matching, Three-way matching

### Set Tolerances
- **Price tolerance:** Accounts payable > Setup > Invoice validation > Price tolerances
  - Percentage or amount tolerance for unit price differences
- **Invoice totals tolerance:** AP > Setup > Invoice validation > Invoice totals tolerances
  - Percentage tolerance for total invoice amount vs expected
- **Charges tolerance:** AP > Setup > Invoice validation > Charges tolerances

### Enable Header Matching
- **Field:** "Automatically update header matching status" = Yes
- This auto-updates the header match status as lines are matched

## Microsoft's Example (from documentation)

**Scenario:** Fabrikam's controller Ken requires:
- All vendor invoices: two-way matching (invoice vs PO)
- Fixed asset items (CNC machines): three-way matching (invoice vs PO vs product receipt)

**Configuration:**
- Legal entity level: matching policy = Three-way matching (default)
- Item 1500 (CNC Machine): matching policy = Three-way matching
- Match price totals = Percentage, tolerance = 15%

**Process:**
1. PO created for 5 machines at $8,000 each = $40,000 + $3,000 S&H
2. Product receipt recorded: quantity 5 received
3. Invoice submitted by vendor: $40,000 + $3,000 S&H
4. Three-way match validates:
   - Invoice quantity (5) = Product receipt quantity (5) ✓
   - Invoice unit price ($8,000) = PO unit price ($8,000) ✓
   - Invoice total ($43,000) within 15% of PO total ($43,000) ✓

## Common Issues
- **Match fails on quantity:** Product receipt not posted, or partial receipt. Check: was the PO fully received?
- **Match fails on price:** Vendor charged a different price than PO. Check tolerances.
- **Match not enforced:** Matching policy set to "Not required" — no validation occurs.
- **Performance:** Three-way matching is slower — requires product receipt lookup per line.
