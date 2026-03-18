---
name: fo-sales-marketing
description: >
  Configure D365 F&O Sales and marketing including sales parameters, sales
  order processing, pricing (trade agreements, price groups), commission setup,
  sales agreement classifications, delivery modes/terms, and order-to-cash flow.
  Use when: setting up sales order processing, pricing structures, or customer
  delivery preferences.
  DEPENDS ON: fo-accounts-receivable (customer setup), fo-inventory-management
  (products). Level 330 in Microsoft's config template sequence.
metadata:
  platform:
    category: "fo-sales-marketing"
    riskLevel: "medium"
    configSequence: 9
    configLevel: "330"
    requires:
      products: ["F&O"]
      skills: ["fo-accounts-receivable", "fo-inventory-management"]
    bpcProcess: "02 - Order to Cash"
    msLearnPath: "https://learn.microsoft.com/en-us/training/paths/configure-manage-sales-customers-dyn365-supply-chain-mgmt/"
    mbExam: "MB-330"
---

# Sales and Marketing Configuration

## Configuration Sequence

### Step 1: Sales Parameters
- **Menu:** Sales and marketing > Setup > Sales and marketing parameters
- **MCP:** Form tool
- **Key tabs:** General, Prices, Shipment, Number sequences, Quality, Rebates
- **Important:** Default delivery mode, order type defaults, SO confirmation settings

### Step 2: Terms of Delivery
- **Menu:** Procurement and sourcing > Setup > Distribution > Terms of delivery
- **Data entity:** `TermsOfDelivery`
- **MCP:** Data tool
- **Purpose:** Incoterms (FOB, CIF, DDP, etc.)
- **Note:** Shared between procurement and sales

### Step 3: Modes of Delivery
- **Menu:** Sales and marketing > Setup > Distribution > Modes of delivery
- **Data entity:** `ModesOfDelivery`
- **MCP:** Data tool
- **Examples:** Ground, Express, Air, Pickup, LTL

### Step 4: Price Groups & Trade Agreements
- **Menu:** Sales and marketing > Prices and discounts > Price groups
- **Menu:** Sales and marketing > Prices and discounts > Trade agreements
- **Data entities:** `PriceGroups`, `TradeAgreementJournalHeaders`, `TradeAgreementJournalLines`
- **MCP:** Data tool for import, Form tool for journal posting
- **Types:** Sales price, Line discount, Total discount, Multiline discount
- **Hierarchy:** Customer-specific → Customer price group → All customers

### Step 5: Commission Setup (if applicable)
- **Menu:** Sales and marketing > Commissions > Sales commission groups
- **MCP:** Form tool
- **Components:** Commission groups, commission calculation rules, commission rates

### Step 6: Sales Agreement Classifications
- **Menu:** Sales and marketing > Setup > Sales agreement classifications
- **MCP:** Form tool
- **Purpose:** Blanket sales agreements with commitment types

### Step 7: SO Workflows
- **Menu:** Sales and marketing > Setup > Sales and marketing workflows
- **MCP:** Form tool

## Sales Order Processing Flow
```
Sales Quotation → Sales Order → Pick → Pack → Ship → Invoice → Payment
    ↓                                              ↓
  (optional)                              Posts to AR + Revenue GL
```

## Pricing Configuration Deep Dive

### Trade Agreement Priority (evaluated in order)
1. **Customer + Item specific** — highest priority
2. **Customer price group + Item** 
3. **All customers + Item**
4. **Customer + Item price group**
5. **Customer price group + Item price group**
6. **All customers + Item price group**
7. **Customer + All items**
8. **Customer price group + All items**
9. **All customers + All items** — lowest priority (base price)

### Price Groups on Master Data
- Price groups can be assigned to: Customers, Customer groups, Campaign IDs
- Used to segment pricing strategies

## Master Data Migration

| Seq | Entity | Data Entity | Dependencies |
|---|---|---|---|
| 1 | Delivery terms | `TermsOfDelivery` | None |
| 2 | Delivery modes | `ModesOfDelivery` | None |
| 3 | Price groups | `PriceGroups` | None |
| 4 | Trade agreement headers | `TradeAgreementJournalHeaders` | Price groups |
| 5 | Trade agreement lines | `TradeAgreementJournalLines` | Headers, customers, products |
| 6 | Commission setup | (Form tool) | Sales reps, commission groups |

## Dual-Write Awareness
- Sales orders sync to Dataverse: `SalesOrderHeadersV2` → `salesorder`
- If CE Sales is deployed, SO creation can flow from CE to F&O
- Pricing may be calculated in F&O and synced back to CE

## Validation Checks
- [ ] Sales parameters configured
- [ ] Delivery terms and modes defined
- [ ] Pricing structure configured (trade agreements or price lists)
- [ ] SO number sequence active
- [ ] Sales posting profiles correct (revenue, COGS, discount accounts)
- [ ] Test: Create SO → confirm → pick → pack slip → invoice → verify GL postings

## Related Skills
- **Prerequisite:** `fo-accounts-receivable`, `fo-inventory-management`
- **Related:** `fo-warehouse-management` (picking/packing), `fo-procurement` (intercompany orders)
