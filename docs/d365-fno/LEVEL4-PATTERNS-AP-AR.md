# Level 4 Patterns — AP & AR (Source to Pay / Order to Cash)

Maps workshop answers → field-level D365 F&O configuration for AP and AR.

---

## 05.2 Manage Supplier Relationships

### Pattern: Standard Vendor Setup (Trade + Service Groups)

**Trigger:** "We have trade vendors (goods) and service vendors (services) with different payment terms"

**Questions → Field-Level Output:**

| Question | Answer Example | Configuration |
|---|---|---|
| What vendor groups? | Trade, Service, Intercompany | AP > Setup > Vendor groups > New × 3 |
| Default payment terms per group? | Trade: Net 30, Service: Net 45, Intercompany: Net 0 | Payment terms setup, assign as default per vendor |
| GL accounts per group? | Trade AP: 21000, Service AP: 21100, Interco: 21200 | Vendor posting profiles per Group level |
| Do trade vendors get discounts? | Yes, 2% 10 Net 30 for select vendors | Cash discounts: "2/10N30" → assign to those vendors |
| Default delivery terms for trade? | FOB Shipping Point | Terms of delivery: "FOB-SHIP" → default on vendor |
| Default delivery mode? | Ground for domestic, Air for international | Modes of delivery: "GROUND", "AIR" |

**Output Configuration — Vendor Groups:**
```
AP > Setup > Vendor groups > New
  Group: TRADE    | Description: Trade Vendors (Goods)
  Group: SERVICE  | Description: Service Vendors
  Group: INTERCO  | Description: Intercompany Vendors
```

**Output — Posting Profiles:**
```
AP > Setup > Vendor posting profiles > New
  Profile: VND-TRADE
    Account code: Group | Group: TRADE
    Summary account: 21000 (Trade AP Control)
    
  Profile: VND-SVC
    Account code: Group | Group: SERVICE
    Summary account: 21100 (Service AP Control)
    
  Profile: VND-IC
    Account code: Group | Group: INTERCO
    Summary account: 21200 (Interco AP Control)

  Profile: VND-ALL
    Account code: All
    Summary account: 21000 (Fallback)
```

**Output — Payment Terms:**
```
AP > Payment setup > Terms of payment > New
  Name: NET30  | Payment method: Net | Days: 30
  Name: NET45  | Payment method: Net | Days: 45
  Name: NET0   | Payment method: Net | Days: 0
```

**Output — Cash Discounts:**
```
AP > Payment setup > Cash discounts > New
  Code: 2/10N30
  Discount percentage: 2%
  Number of days (discount): 10
  Main account: 52500 (Purchase Discounts Taken)
```

---

### Pattern: Vendor with Electronic Payment (ACH/NACHA)

**Trigger:** "We want to pay this vendor group electronically"

**Pre-requisite:** Bank account configured with NACHA ER format (see GL banking pattern)

**Questions → Config:**

| Question | Answer | Config |
|---|---|---|
| Which vendors get electronic payment? | All trade vendors | Assign method of payment "ACH" as default on TRADE group |
| Bank account to pay from? | Chase Business Checking | Payment method > Payment control > Bank account |
| Do you need remittance advice emails? | Yes, email vendor when payment sent | Configure vendor email + payment remittance report |

**Output:**
```
1. AP > Payment setup > Methods of payment > New
   Name: ACH
   Period: Current
   Account type: Bank
   Payment account: CHASE-BUS
   File format:
     Generic electronic export format: YES
     Export format configuration: [NACHA ER config]
   Payment control:
     Bank account: CHASE-BUS

2. On each trade vendor record:
   AP > Vendors > [vendor] > Payment tab
   Method of payment: ACH
   Bank account: [vendor's bank info for ACH]
   
3. For remittance:
   AP > Payment setup > Methods of payment > ACH > Remittance
   Print remittance: Yes
   Report format: [standard or custom]
```

---

### Pattern: Three-Way Matching for Inventory Items

**Trigger:** "For physical goods, we want to match PO → Receipt → Invoice before payment"

**Questions → Config:**

| Question | Answer | Config |
|---|---|---|
| All items or only some? | All stocked items get 3-way; services get 2-way | Item-level matching policy |
| Price tolerance? | 2% or $5, whichever is greater | Price tolerance per item/vendor |
| Quantity tolerance? | Exact match to receipt | No over-billing tolerance |
| Who approves mismatches? | AP Manager | Workflow for invoice holds |

**Output:**
```
1. AP > Setup > AP parameters > Invoice validation tab
   Matching policy: Three-way matching (legal entity default)
   
2. Override at item level for services:
   Product info mgmt > Released products > [service item] > Purchase tab
   Matching policy: Two-way matching

3. AP > Setup > Invoice validation > Price tolerances > New
   Tolerance: 2% AND $5.00 (whichever is greater)
   Apply to: All items (or per item group)

4. AP > Setup > Invoice validation > Invoice totals tolerances
   Tolerance percentage: 1% (for total invoice amount validation)

5. AP > Setup > AP workflows > New
   Type: Vendor invoice workflow
   Condition: If matching status = Failed → route to AP Manager for approval
```

**Validation:** Create PO for 100 units at $10. Receive 100 units. Create invoice for 100 units at $10.25 (2.5% over) → matching fails → routes to AP Manager via workflow.

---

## 02.3 Manage Accounts Receivable

### Pattern: Credit Management with Automatic Hold

**Trigger:** "We want to block sales orders when a customer exceeds their credit limit"

**Questions → Config:**

| Question | Answer | Config |
|---|---|---|
| Default credit limit for new customers? | $50,000 | Credit management parameters |
| What happens when exceeded? | Automatically place order on hold | Blocking rule: Order hold |
| Who releases the hold? | Credit Manager | Hold assignment/release permissions |
| Do you score credit risk? | Yes, by customer group | Credit scoring groups |
| Do you use external credit scores? | No, internal assessment only | Skip external integration |
| Grace period before hold? | None — immediate | Days past credit limit: 0 |

**Output:**
```
1. Credit and collections > Setup > Credit management parameters
   Default credit limit: 50,000
   Enable credit management: Yes
   
2. Credit and collections > Setup > Credit management > Blocking rules > New
   Rule: "Credit Limit Exceeded"
   Blocking type: Sales order hold
   Criteria: Credit limit exceeded by any amount
   
3. Credit and collections > Setup > Credit management > Credit management groups
   Assign users who can release holds (Credit Manager role)

4. On customer records:
   AR > Customers > [customer] > Credit and collections tab
   Credit limit: [per-customer override if different from default]
   Credit management group: [assignment]
```

**Validation:** Create customer with $50K limit. Create SO for $45K → OK. Create second SO for $10K (total $55K exceeds $50K) → order goes on hold. Credit Manager releases → order proceeds.

---

### Pattern: Collection Letters with Escalation

**Trigger:** "We send reminder letters at 30, 60, and 90 days overdue"

**Questions → Config:**

| Question | Answer | Config |
|---|---|---|
| How many levels? | 3 (1st notice, 2nd notice, final notice) | Collection letter sequence |
| Days after due date per level? | 30, 60, 90 | Grace period per letter |
| Fees charged? | None on 1st, $25 on 2nd, $50 on final | Collection letter fee per level |
| Interest on overdue? | Yes, 1% monthly | Interest code setup |
| Who reviews before sending? | AR Clerk reviews, AR Manager approves final notice | Manual process (or workflow) |

**Output:**
```
1. AR > Setup > Collections > Collection letter sequence > New
   Sequence: "STD-COLLECTION"
   
   Line 1: Collection letter code: "FIRST"
     Days: 30 | Fee: 0.00 | Note: "Friendly reminder"
   
   Line 2: Collection letter code: "SECOND"  
     Days: 60 | Fee: 25.00 | Note: "Second notice - balance overdue"
   
   Line 3: Collection letter code: "FINAL"
     Days: 90 | Fee: 50.00 | Note: "Final notice - immediate payment required"

2. AR > Setup > Interest > Interest codes > New
   Code: "1PCT-MONTHLY"
   Calculate interest every: 1 month
   Interest by range: 
     From amount: 0 | Percentage: 1.0% (monthly)
   
3. AR > Setup > Customer posting profiles > [profile]
   Interest code: 1PCT-MONTHLY
   Collection letter sequence: STD-COLLECTION

4. Assign to customer groups or all customers via posting profile
```

**Validation:** Create an invoice dated 35 days ago (past 1st notice threshold). Run: Credit and collections > Periodic > Create collection letters. Verify 1st notice generated for that customer.

---

### Pattern: Customer Payment via Direct Debit (SEPA — Europe)

**Trigger:** "We collect from EU customers via SEPA Direct Debit"

**Pre-requisite:** Bank account configured, SEPA DD ER format imported

**Questions → Config:**

| Question | Answer | Config |
|---|---|---|
| Which customers? | All EU customers | Customer payment method default |
| Bank account? | EUR bank account | SEPA DD mandate references this bank |
| Mandate management? | Yes, track DD mandates per customer | Enable mandates in AR |
| Pre-notification required? | Yes, 14 days before collection | Pre-notification parameter |

**Output:**
```
1. Org admin > Electronic reporting > Configurations > Load from repository
   Import: ISO20022 Direct Debit (pain.008)

2. AR > Payment setup > Methods of payment > New
   Name: SEPA-DD
   Period: Current
   Account type: Bank
   Payment account: EUR-BANK
   File format:
     Generic electronic export format: YES
     Export format configuration: ISO20022 Direct Debit
     
3. AR > Setup > Payment > Direct debit mandates
   Enable mandate management: Yes
   Pre-notification days: 14

4. Per customer:
   AR > Customers > [EU customer] > Payment defaults
   Method of payment: SEPA-DD
   Bank account: [customer's bank account for DD]
   
   AR > Customers > [customer] > Direct debit mandates > New
   Mandate ID: [generated]
   Signature date: [date customer authorized]
   Bank account: [customer's bank]
```

---

## Agent Workflow: Workshop to Configuration

The complete agent-driven workflow for closing the requirements-to-config loop:

```
STEP 1: DISCOVERY AGENT
├── Processes meeting recording from requirements workshop
├── Extracts requirements and maps to BPC Level 2/3 processes
├── Identifies which Level 4 patterns apply
├── Output: Structured requirements mapped to BPC

STEP 2: WORKSHOP AGENT  
├── For each identified BPC process:
│   ├── Loads the question set from this document
│   ├── Presents questions to consultant (via Teams Adaptive Card)
│   ├── Records structured answers
│   └── Feeds answers into decision matrix
├── Output: Answered question sets per process

STEP 3: CONFIG PLANNING AGENT
├── Takes answered questions
├── Runs decision matrices (if answer A → config X)
├── Generates sequenced Configuration Plan (respecting Microsoft's dependency levels)
├── Includes: menu path, data entity, MCP approach for each step
├── Output: Configuration Plan document

STEP 4: CONFIG EXECUTION AGENT
├── Takes Configuration Plan
├── For each step:
│   ├── Uses MCP data tools for entity creation (preferred)
│   ├── Uses MCP form tools for parameter/structure setup
│   ├── Logs every action to audit trail
│   └── Requires human approval for write operations on UAT/Prod
├── Output: Configured environment

STEP 5: VALIDATION AGENT
├── For each configured item:
│   ├── Runs validation test (from this document's test cases)
│   ├── Verifies configuration matches workshop answers
│   ├── Flags any mismatches
│   └── Reports to consultant
├── Output: Validation report (pass/fail per item)

STEP 6: REQUIREMENT TRACEABILITY
├── Links: BPC process → Workshop answer → Config item → Validation result
├── Every configuration item traces back to a specific business requirement
├── Gap analysis: any BPC processes in scope that don't have config?
├── Output: Traceability matrix for audit
```

This is the closed loop. No requirement falls through the cracks. No configuration exists without a business justification.
