# D365 ERP MCP Server Guide

Source: Microsoft Learn (copilot-mcp, build-agent-mcp) + Microsoft Blog (Ignite 2025) — verified 2026-03-18

## Overview

The Dynamics 365 ERP MCP server provides a standard interface for AI agents to access F&O data and business logic. Three tool categories enable nearly any operation a user can perform through the application interface.

## Prerequisites
- F&O version: 10.0.47 or later
- Feature enabled: "Dynamics 365 ERP Model Context Protocol server" in Feature Management
- Environment: Tier 2+ or UDE (NOT supported on Cloud Hosted Environments)
- Your agent platform must be registered in **Allowed MCP Clients** page

## Tool Categories

### Data Tools (OData-based)
**Best for:** Standard CRUD operations. Faster and fewer tool calls than form tools.

| Tool | Purpose |
|---|---|
| `data_find_entity_type` | Search for OData entity types. Returns multiple top hits. |
| `data_get_entity_metadata` | Get schema for an entity. Required before CRUD operations. |
| `data_create_entities` | Create records. **No deep inserts** (create parent first, then child). |
| `data_delete_entities` | Delete records. |
| `data_update_entities` | Update records. |
| `data_find_entities` | Query/read records with OData filters. |

**Rules:**
- Use PLURAL entity name in OData path
- For V2+ entities, plural before V: `SalesOrderHeadersV2` not `SalesOrderHeaderV2s`
- Enum filtering: `$filter=Style has Namespace.Pattern'Yellow'`

### Form Tools (Headless UI)
**Best for:** Operations requiring business logic, button clicks, or actions NOT available via simple CRUD. Works through server APIs, not screen scraping.

| Tool | Purpose |
|---|---|
| `form_find_menu_item` | Find a menu item by search |
| `form_open_menu_item` | Open a form |
| `form_click_control` | Click buttons/actions (via control name or actionId) |
| `form_set_control_values` | Set field values. **Don't use for lookup fields** — use `form_open_lookup` instead. |
| `form_filter_form` | Apply filter on form |
| `form_filter_grid` | Filter on a grid |
| `form_find_controls` | Find controls on the form (one search term per call) |
| `form_open_lookup` | Open a lookup control |
| `form_open_or_close_tab` | Navigate tabs |
| `form_save_form` | Save the form |
| `form_close_form` | Close the form |
| `form_select_grid_row` | Select a grid row |
| `form_sort_grid_column` | Sort a grid column |

**Rules:**
- Use menu item NAMES not labels
- Use control NAMES not labels
- Use tab NAMES not labels
- Use grid column NAMES not labels
- Form state returns max 25 rows — warn if truncated
- `(lessThanDate(x:int))` is valid for date column filters

### Action Tools (Custom X++ Logic)
**For:** Operations exposed through custom X++ classes implementing `ICustomAPI`.

| Tool | Purpose |
|---|---|
| `api_find_actions` | Discover available custom actions |
| `api_invoke_action` | Execute a custom action |

## Dynamic Context

The MCP server dynamically updates context with each tool call based on:
- Agent's **security role** — only returns objects the role can access
- **Application configuration, extensions, personalization** — ISV extensions automatically available
- When `form_find_menu_item` is called → returns only menu items accessible to the role
- When form view model is returned → includes only data/fields/actions the role permits

## Tool Selection Priority (from Microsoft's recommended agent instructions)

```
1. For CRUD operations → ALWAYS prefer data tools first
2. When explicitly instructed, or if impossible via data tools → use form tools
3. For custom X++ logic → use action tools
```

**Why:** Data tools are 3-5x faster than form tools for standard operations.

## Recommended Orchestration Model

Microsoft recommends **Claude Sonnet 4.5** as the agent orchestration model in Copilot Studio. GPT-4.1 should NOT be used as the orchestration model (it works in VS Code but not well in Copilot Studio). If Claude isn't available, use **GPT-5 (Chat)** as fallback.

## Known Limitations (as of v10.0.47)

1. **Performance:** Community reports describe preview as "slow to respond and behave inconsistently" depending on interface
2. **Form state truncation:** Max 25 rows per tool call response
3. **No deep inserts:** Can't create nested entities — parent first, then child
4. **Sidecar not supported:** Adding MCP to Copilot for F&O sidecar chat causes errors. Microsoft doesn't guarantee assistance.
5. **CHE not supported:** Only Tier 2+ or UDE environments
6. **Static MCP server retiring:** The 13-tool static server from Build 2025 will be retired in 2026

## MCP for Analytics (Preview — December 2025)

A second MCP server for analytics:
- Connects to Business Performance Analytics (BPA) dimensional models
- Generates DAX queries from natural language
- Enforces row-level security
- Provides governed access to measures, dimensions, reports, semantic models
