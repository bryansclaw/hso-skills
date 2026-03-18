---
name: fo-integration-patterns
description: >
  Guide for D365 F&O integration patterns including OData/data entities, Data
  Management Framework (DMF), dual-write (F&O ↔ Dataverse/CE), virtual entities,
  business events, Power Automate flows, MCP server, and Azure Data Factory.
  Use when: deciding which integration pattern to use, configuring dual-write
  maps, setting up business events, or designing data flows between F&O and
  external systems.
  This is a cross-cutting skill used across all modules.
metadata:
  platform:
    category: "fo-integration"
    riskLevel: "medium"
    configSequence: 0
    configLevel: "cross-cutting"
    requires:
      products: ["F&O"]
    bpcProcess: "99 - Administer to Operate (integrations)"
    msLearnPath: "https://learn.microsoft.com/en-us/dynamics365/guidance/implementation-guide/integrate-other-solutions-guidance-product"
---

# Integration Patterns for D365 F&O

## Pattern Selection Matrix

| Pattern | Best For | Volume | Latency | Direction |
|---|---|---|---|---|
| **OData (Data entities)** | Synchronous API, Excel add-in, lightweight reads/writes | Low-Medium | Real-time | Bidirectional |
| **DMF (Data Management Framework)** | Bulk migration, config import/export, recurring integration | High | Batch | Import/Export |
| **Dual-write** | Near-real-time sync F&O ↔ Dataverse/CE apps | Low-Medium | Near-real-time | Bidirectional |
| **Virtual entities** | Read F&O data from Dataverse without data movement | Low | Real-time (read) | F&O → Dataverse (read-only) |
| **Business events** | Event-driven triggers on business actions | N/A | Event-driven | F&O → external |
| **Power Automate** | No-code workflow automation, approvals, notifications | Low | Event/Schedule | Bidirectional |
| **MCP Server** | AI agent interactions, headless config, autonomous operations | Low-Medium | Request/Response | Bidirectional |
| **Azure Data Factory** | Complex ETL, data warehouse, hybrid cloud | High | Batch/Scheduled | Both |
| **Custom services (X++)** | Custom business logic APIs | Any | Real-time | Bidirectional |

## ⚠️ Critical Warnings

### Dual-Write Is NOT for Data Migration
- Dual-write Initial Sync is designed for LOW VOLUME setup/reference data ONLY
- It uses F&O OData entities underneath — inherently slower than DMF for bulk
- Keep data migration COMPLETELY SEPARATE from dual-write infrastructure
- Use DMF for migration, dual-write for ongoing sync after go-live

### OData Throttling
- F&O has OData API throttling limits (priority-based)
- High-volume OData calls can be throttled or fail
- For bulk operations, always use DMF instead

### MCP Server Limitations (as of v10.0.47)
- Still in preview — "slow to respond and behave inconsistently" per community reports
- Form state returns max 25 rows per tool call
- No deep inserts (create parent first, then child)
- Entity names must be PLURAL in OData paths
- Not supported on Cloud Hosted Environments (CHE)
- Sidecar chat panel integration not yet supported

## Dual-Write Configuration

### Prerequisites
- F&O environment linked to Dataverse (PPAC-managed)
- Dual-write solution installed in Dataverse
- Initial sync completed for foundation maps

### Key Maps for F&O Implementations

| F&O Entity | Dataverse Table | Notes |
|---|---|---|
| `CustomersV3` | `account` | Requires CDS Parties, Postal Addresses, Contacts V3 maps |
| `VendorsV2` | `msdyn_vendor` | Similar party map dependencies |
| `ReleasedProductsV2` | `msdyn_sharedproductdetails` | F&O → Dataverse direction typically |
| `Currencies` | `transactioncurrency` | Foundation map |
| `SalesOrderHeadersV2` | `salesorder` | Bidirectional if CE Sales deployed |
| `PurchaseOrderHeadersV2` | `msdyn_purchaseorder` | F&O → Dataverse typically |

### Dual-Write Best Practices
- Start with foundation maps (currencies, units, companies)
- Add party/address maps before customer/vendor maps
- Test with small data sets before enabling for all records
- Monitor dual-write health dashboard for errors
- Don't enable dual-write on entities being actively migrated via DMF

## Business Events Configuration

### Setup
- **Menu:** System administration > Setup > Business events > Business events catalog
- Browse available events per module
- Activate events for specific legal entities
- Configure endpoint (Power Automate, Azure Service Bus, Azure Event Grid, HTTPS webhook)

### Common Business Events

| Module | Event | Trigger |
|---|---|---|
| GL | Journal posted | GL journal posting completes |
| AP | Invoice approved | Vendor invoice workflow approved |
| AP | Payment posted | Vendor payment journal posted |
| AR | Invoice posted | Customer invoice posted |
| AR | Payment received | Customer payment applied |
| Inventory | Transfer order shipped | Transfer order shipment confirmed |
| Production | Production order ended | Production order cost finalized |
| Workflow | Work item assigned | Any workflow approval assigned |
| Workflow | Work item completed | Any workflow step completed |

### Power Automate Integration
- **Trigger:** "When a Business Event occurs" (Dataverse connector)
- **Or:** "When a record is created, updated, or deleted" (CUD on F&O virtual entities)
- **Common flows:**
  - Invoice approved → Teams notification to payment team
  - PO confirmed → Email to vendor
  - Credit hold applied → Approval card to credit manager
  - Config change detected → Validation agent triggered

## MCP Server Integration (for our platform)

### Tool Categories
1. **Data tools** — CRUD via OData (preferred for standard operations)
2. **Form tools** — headless UI navigation (for business logic, button clicks)
3. **Action tools** — custom X++ logic via ICustomAPI

### Our Platform Integration Architecture
```
Platform Agent (Semantic Kernel)
    │
    ├── D365FOMCPPlugin (wraps MCP tools)
    │   ├── data_find_entity_type → entity discovery
    │   ├── data_get_entity_metadata → schema inspection
    │   ├── data_find_entities → read/query
    │   ├── data_create/update/delete_entities → write
    │   ├── form_find_menu_item → navigation
    │   ├── form_open_menu_item → open forms
    │   ├── form_set_control_values → configure settings
    │   ├── form_click_control → execute actions
    │   └── api_invoke_action → custom X++ logic
    │
    └── Approval Gate (before writes)
        ├── Dev environment → auto-approve reads + writes
        ├── UAT environment → single approval for writes
        └── Prod environment → multi-approval for writes
```

### MCP + DMF Hybrid Pattern
For data migration, combine MCP (for discovery and validation) with DMF (for bulk import):
1. MCP `data_find_entity_type` → discover correct entities
2. MCP `data_get_entity_metadata` → understand field requirements
3. MCP `data_find_entities` → validate target has prerequisite data
4. DMF package import → bulk load data
5. MCP `data_find_entities` → post-import validation queries
6. Account Reconciliation → verify subledger = GL

## Validation Checks
- [ ] Integration pattern selected appropriately per use case
- [ ] Dual-write maps enabled and healthy (if using dual-write)
- [ ] Business events activated for required events
- [ ] Power Automate flows configured and tested
- [ ] MCP server enabled in Feature Management (v10.0.47+)
- [ ] Our platform registered as Allowed MCP Client
- [ ] API throttling limits understood and respected
- [ ] Test: End-to-end integration flow verified per pattern

## Related Skills
- **Used by:** All module skills (for data access patterns)
- **Related:** `fo-data-migration` (DMF-specific operations)
