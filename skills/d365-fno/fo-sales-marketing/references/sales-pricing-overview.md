# Sales Order Processing and Pricing

Source: Microsoft Learn + verified research — 2026-03-18

## Sales Order Processing Flow

```
Sales Quotation (optional) → Sales Order → Confirmation → 
    Pick List → Packing Slip → Invoice → Payment Collection
```

| Stage | What Happens | GL Impact |
|---|---|---|
| **Quotation** | Price proposal to customer | None |
| **Order creation** | SO created, prices calculated | None (unless budget control/encumbrance) |
| **Confirmation** | SO confirmed with customer | None |
| **Pick list** | Warehouse picks items | Inventory reserved/picked |
| **Packing slip** | Goods shipped to customer | Inventory physically issued |
| **Invoice** | Revenue recognized | AR debited, Revenue credited, COGS debited, Inventory credited |
| **Payment** | Customer pays | Cash/Bank debited, AR credited |

## Pricing Architecture

### Trade Agreements
Navigation: **Sales and marketing > Prices and discounts > Trade agreements**

Trade agreements are the primary pricing mechanism. They define vendor price lists with prices or discounts, with specific effective date ranges.

### Price Resolution Priority (Most to Least Specific)

| Priority | Sales Tax Group | Item |
|---|---|---|
| 1 | Specific customer | Specific item |
| 2 | Customer price group | Specific item |
| 3 | All customers | Specific item |
| 4 | Specific customer | Item price group |
| 5 | Customer price group | Item price group |
| 6 | All customers | Item price group |
| 7 | Specific customer | All items |
| 8 | Customer price group | All items |
| 9 | All customers | All items |

**First match wins.** System evaluates from priority 1 down and uses the first applicable price.

### Discount Types
| Type | Description |
|---|---|
| **Line discount** | Percentage or amount off per line |
| **Multiline discount** | Discount when buying across multiple lines |
| **Total discount** | Discount on order total |
| **Rebate** | Post-sale credit (typically volume-based) |

### Price Groups
- Assign to: Customers, Customer groups, Campaigns
- Used to segment pricing strategies
- A customer can belong to multiple price groups
- Enable targeted pricing without per-customer trade agreements

## Trade Agreement Journal Import

For migrating pricing data:
| Entity | Purpose |
|---|---|
| `TradeAgreementJournalHeaders` | Price journal headers |
| `TradeAgreementJournalLines` | Price/discount rules |

**Critical:** After import, trade agreement journals must be POSTED via form tools to become active. Import creates the journal; posting activates the prices.

## Sales Agreement (Blanket Orders)

Navigation: **Sales and marketing > Sales agreements**
- Commitment types: Product quantity, Product value
- Track fulfillment against commitment
- Release orders draw down the agreement balance

## Configuration Components

| Component | Navigation |
|---|---|
| Sales parameters | Sales and marketing > Setup > Sales and marketing parameters |
| Terms of delivery | Procurement and sourcing > Setup > Distribution > Terms of delivery (shared) |
| Modes of delivery | Sales and marketing > Setup > Distribution > Modes of delivery |
| Price groups | Sales and marketing > Prices and discounts > Price groups |
| Commission groups | Sales and marketing > Commissions > Sales commission groups |
| Sales agreement classifications | Sales and marketing > Setup > Sales agreement classifications |
| SO workflows | Sales and marketing > Setup > Sales and marketing workflows |

## Dual-Write Awareness
- Sales orders sync to Dataverse: `SalesOrderHeadersV2` → `salesorder`
- If CE Sales is deployed, SOs may originate in CE and flow to F&O via dual-write
- Pricing can be calculated in F&O and synced back to CE
- Customer changes are bidirectional via dual-write customer maps

## Key Settings in Sales Parameters

| Tab | Key Settings |
|---|---|
| General | Default order type, SO confirmation settings |
| Prices | Price calculation preferences, discount search behavior |
| Shipment | Default delivery mode, packing slip settings |
| Number sequences | SO numbers, quotation numbers, packing slip numbers |
| Quality | Quality association triggers |
| Rebates | Rebate processing parameters |
