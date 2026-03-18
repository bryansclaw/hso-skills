# AP Setup Sequence and Configuration Details

Source: Microsoft Learn (accounts-payable-overview) — verified 2026-03-18

## Prerequisites (Must Complete Before AP Setup)

From Microsoft: Before you can set up Accounts payable, complete the following:

1. **In General ledger:**
   - Set up payment journals (if you plan to use them)
   - Set up currency codes (General ledger > Currencies > Currencies)
   - Set up exchange rate types and rates
2. **In Cash and bank management:**
   - Set up bank accounts to use with methods of payment

## Required Setup Pages (Microsoft's Recommended Order)

1. **Terms of payment** — define payment terms assigned to POs, vendors, and determine invoice due dates
2. **Methods of payment - vendors** — how the organization pays vendors (check, electronic, wire)
3. **Vendor groups** — groups that share posting, settlement, payment, reporting, and forecasting parameters
4. **Vendor posting profiles** — define how vendor transactions post to GL
5. **Accounts payable parameters** — default settings, functionality parameters, number sequences
6. **Form setup** — format of vendor-related documents
7. **Vendors** — create vendor accounts + tax authorities

## Optional Setup Pages (by Category)

### Policies
- **Vendor invoice policy** — invoice validation rules

### Invoice Matching
- **Invoice totals tolerances** — acceptable variance for invoice totals
- **Matching policy** — two-way or three-way matching
- **Price tolerances** — acceptable unit price variance
- **Item price tolerance groups** — per-item tolerance
- **Vendor price tolerance groups** — per-vendor tolerance
- **Charges tolerances** — acceptable charges variance

### Workflow
- **Accounts payable workflows** — journal approvals, purchase requisitions

### Charges
- **Charges code** — codes for charges used in purchase orders
- **Vendor charges group** — charges groups for vendors
- **Item charge groups** — charges groups for items
- **Auto charges** — charges automatically assigned to orders

### Distribution
- **Terms of delivery** — Incoterms (FOB, CIF, DDP, etc.)
- **Modes of delivery** — transport methods (ground, express, air)
- **Destination codes** — delivery destination identifiers

### Payments
- **Cash discounts** — terms for obtaining cash discounts (linked to vendors, applied to POs)
- **Payment schedules** — installment payment schedules
- **Payment days** — specific days of week/month for payments
- **Payment fee** — fees associated with payment processing
- **Payment instruction** — payment instructions for vendors

### Statistics
- **Aging period definitions** — user-defined intervals for vendor maturity analysis
- **Line of business** — LOB codes assigned to vendors

### Tax 1099 (US only)
- **1099 fields** — verify and update minimum reporting amounts per IRS requirements
