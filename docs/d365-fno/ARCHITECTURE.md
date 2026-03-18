# D365 AI Implementation Platform — Architecture Document

**Version:** 0.1 (Draft)
**Date:** 2026-03-18
**Status:** Proposal

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Vision & Goals](#2-vision--goals)
3. [Architecture Overview](#3-architecture-overview)
4. [Core Components](#4-core-components)
   - 4.1 [Gateway Service](#41-gateway-service)
   - 4.2 [Agent Runtime](#42-agent-runtime)
   - 4.3 [Teams Integration Layer](#43-teams-integration-layer)
   - 4.4 [Skills Engine](#44-skills-engine)
   - 4.5 [Meeting Intelligence Pipeline](#45-meeting-intelligence-pipeline)
   - 4.6 [Scheduler & Task Engine](#46-scheduler--task-engine)
   - 4.7 [Blazor Admin UI](#47-blazor-admin-ui)
   - 4.8 [D365 Environment Connectors](#48-d365-environment-connectors)
   - 4.9 [DevOps Integration](#49-devops-integration)
5. [Agent Catalog](#5-agent-catalog)
6. [Data Architecture](#6-data-architecture)
7. [Security & Access Control](#7-security--access-control)
8. [Skills Framework (Detailed)](#8-skills-framework-detailed)
9. [Session & Memory Model](#9-session--memory-model)
10. [Deployment Architecture](#10-deployment-architecture)
11. [Integration Points](#11-integration-points)
12. [Build Phases & Roadmap](#12-build-phases--roadmap)
13. [Risk Register](#13-risk-register)
14. [Appendix: OpenClaw Pattern Mapping](#appendix-openclaw-pattern-mapping)

---

## 1. Executive Summary

This document describes the architecture for an AI-powered implementation platform purpose-built for Microsoft Dynamics 365 practices. The platform orchestrates specialized AI agents that assist consultants through every phase of the D365 implementation lifecycle — from discovery and requirements gathering through configuration, data migration, testing, and go-live.

The architecture is inspired by [OpenClaw](https://github.com/openclaw/openclaw), an open-source personal AI gateway that routes conversations across messaging channels to specialized agents. We adapt its proven patterns — gateway control plane, multi-agent routing, skills-based extensibility, session management, and scheduled automation — to an enterprise context using the Microsoft technology stack.

**Key differences from OpenClaw:**

| Concern | OpenClaw | This Platform |
|---|---|---|
| Target user | Individual developer | Implementation team (5-50 people) |
| Channel | 20+ consumer messengers | Microsoft Teams (primary) |
| Extensibility | Open (anyone installs skills) | Controlled (admin-approved skills) |
| Persistence | Filesystem (JSONL) | Azure Cosmos DB + Blob Storage |
| Runtime | Node.js + Pi agent | .NET 8 + Semantic Kernel |
| UI | Web Control UI (read-only) | Blazor Server (full admin) |
| Auth | DM pairing / allowlists | Azure AD + RBAC |
| Hosting | Self-hosted (local machine) | Azure Container Apps (managed) |

---

## 2. Vision & Goals

### Vision

Every D365 implementation consultant has an AI partner that listens in meetings, extracts requirements, generates configuration, tracks progress, and handles the repetitive work — so humans can focus on judgment, relationships, and the parts of consulting that actually require a brain.

### Goals

1. **Reduce implementation time by 30-40%** through automated artifact generation (BPA, BPC, config packages, data mappings)
2. **Capture institutional knowledge** in skills that any consultant can leverage, not just senior resources
3. **Eliminate context loss** between project phases by maintaining continuous AI memory per project
4. **Standardize delivery quality** through agent-enforced best practices and templates
5. **Surface insights from meetings** that would otherwise require manual note-taking and follow-up

### Non-Goals

- Replacing consultants (agents assist, humans decide)
- Fully autonomous D365 configuration without review
- Supporting non-Microsoft ERP platforms (initially)
- End-user/customer-facing AI (internal team tool only)

---

## 3. Architecture Overview

### High-Level Component Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Microsoft Teams                               │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│   │ Project A │  │ Project B │  │ Admin    │  │ General  │          │
│   │ Channel   │  │ Channel   │  │ Channel  │  │ Channel  │          │
│   └─────┬─────┘  └─────┬─────┘  └─────┬────┘  └─────┬────┘          │
└─────────┼───────────────┼───────────────┼────────────┼───────────────┘
          │               │               │            │
          ▼               ▼               ▼            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    GATEWAY SERVICE (.NET 8)                          │
│                                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐                │
│  │ Teams Bot    │  │ SignalR Hub  │  │ REST API     │                │
│  │ Adapter      │  │ (real-time)  │  │ (admin ops)  │                │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                │
│         │                 │                  │                        │
│  ┌──────▼─────────────────▼──────────────────▼──────┐                │
│  │              MESSAGE ROUTER                       │                │
│  │  (session resolution → agent binding → dispatch)  │                │
│  └──────────────────────┬────────────────────────────┘                │
│                         │                                            │
│  ┌──────────┬───────────┼───────────┬──────────────┐                │
│  ▼          ▼           ▼           ▼              ▼                │
│ ┌────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌──────────┐          │
│ │Cron│  │Session │  │Skills  │  │Agent   │  │Audit     │          │
│ │Svc │  │Store   │  │Loader  │  │Runtime │  │Logger    │          │
│ └────┘  └────────┘  └────────┘  └────────┘  └──────────┘          │
└─────────────────────────────────────────────────────────────────────┘
          │                                    │
          ▼                                    ▼
┌──────────────────┐              ┌──────────────────────────────┐
│  SCHEDULER       │              │  AGENT RUNTIME               │
│  (Durable Funcs) │              │  (Semantic Kernel)           │
│                  │              │                              │
│  • Cron jobs     │              │  ┌─────────┐ ┌───────────┐  │
│  • One-shot      │              │  │Discovery│ │BPC Agent  │  │
│  • Approval      │              │  │Agent    │ │           │  │
│    workflows     │              │  └─────────┘ └───────────┘  │
│  • Retry/DLQ     │              │  ┌─────────┐ ┌───────────┐  │
└──────────────────┘              │  │Config   │ │Migration  │  │
                                  │  │Agent    │ │Agent      │  │
┌──────────────────┐              │  └─────────┘ └───────────┘  │
│  MEETING         │              │  ┌─────────┐ ┌───────────┐  │
│  INTELLIGENCE    │              │  │DevOps   │ │Solution   │  │
│                  │              │  │Agent    │ │Agent      │  │
│  • Graph API     │              │  └─────────┘ └───────────┘  │
│  • Transcription │              └──────────────────────────────┘
│  • LLM Extract   │                         │
│  • Artifact Gen  │              ┌───────────▼────────────────┐
└──────────────────┘              │  D365 ENVIRONMENT          │
                                  │  CONNECTORS                │
┌──────────────────┐              │                            │
│  BLAZOR ADMIN UI │              │  • CE (Dataverse API)      │
│                  │              │  • F&O (Data Entities API)  │
│  • Dashboards    │              │  • DevOps (REST API)        │
│  • Skill mgmt    │              │  • Power Platform (Admin)   │
│  • Task builder  │              └────────────────────────────┘
│  • Meeting view  │
│  • Config review │
└──────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                    DATA LAYER                                 │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │
│  │Cosmos DB │  │Blob      │  │Redis     │  │Azure SQL   │  │
│  │(sessions,│  │Storage   │  │(cache,   │  │(structured │  │
│  │ config,  │  │(files,   │  │ pubsub)  │  │ project    │  │
│  │ audit)   │  │ recordings│ │          │  │ data)      │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### Request Flow (Teams Message → Agent Response)

```
1. User @mentions agent in Teams channel
2. Bot Framework receives activity
3. Gateway resolves: channel → project → agent binding
4. Session loaded from Cosmos DB (or created)
5. Skills loader injects relevant skills for this agent
6. Semantic Kernel agent turn executes:
   a. System prompt (AGENTS.md + SOUL.md + PROJECT.md + skills)
   b. User message + conversation history
   c. Tool calls (D365 connector, DevOps, file ops, web search)
   d. Response generation
7. Response delivered to Teams via Adaptive Card or text
8. Session persisted, audit log written
```

---

## 4. Core Components

### 4.1 Gateway Service

**Technology:** ASP.NET Core 8, hosted on Azure Container Apps

**Responsibilities:**
- Single entry point for all inbound messages (Teams bot webhook)
- SignalR hub for real-time admin UI connections
- REST API for admin operations (CRUD agents, skills, schedules, projects)
- Message routing: resolve inbound activity → project → agent binding → dispatch
- Config management (hot reload, validation)
- Health monitoring and diagnostics

**OpenClaw pattern:** OpenClaw's Gateway is a single Node.js process that owns all channel connections and exposes a typed WebSocket API. Ours mirrors this as a single .NET service with SignalR (WebSocket) for real-time and REST for admin ops.

```csharp
// Conceptual Gateway structure
public class GatewayService : BackgroundService
{
    private readonly IMessageRouter _router;
    private readonly ISessionStore _sessions;
    private readonly ISkillsLoader _skills;
    private readonly IAgentRuntime _agentRuntime;
    private readonly IScheduler _scheduler;
    private readonly IAuditLogger _audit;

    // Inbound from Teams Bot Framework
    public async Task OnMessageActivity(ITurnContext context)
    {
        var route = await _router.Resolve(context);
        var session = await _sessions.GetOrCreate(route.SessionKey);
        var skills = await _skills.LoadForAgent(route.AgentId);
        var response = await _agentRuntime.Execute(route, session, skills);
        await DeliverResponse(context, response);
        await _audit.Log(route, response);
    }
}
```

**Key config (mirrors OpenClaw's `openclaw.json`):**

```json5
{
  "gateway": {
    "bind": "0.0.0.0:8080",
    "auth": { "provider": "AzureAD", "tenantId": "..." }
  },
  "agents": {
    "defaults": {
      "model": { "primary": "azure-openai/gpt-4o" },
      "workspace": "/app/workspaces/{agentId}"
    },
    "list": [
      { "id": "discovery", "displayName": "Discovery Agent", "workspace": "..." },
      { "id": "bpc", "displayName": "BPC Agent", "workspace": "..." }
    ]
  },
  "bindings": [
    {
      "agentId": "discovery",
      "match": { "channel": "teams", "teamId": "...", "channelPattern": "discovery-*" }
    }
  ],
  "projects": {
    "list": [
      {
        "id": "contoso-fno",
        "customer": "Contoso",
        "modules": ["Finance", "Supply Chain"],
        "product": "F&O",
        "environments": { "dev": "...", "uat": "...", "prod": "..." }
      }
    ]
  }
}
```

### 4.2 Agent Runtime

**Technology:** Microsoft Semantic Kernel (.NET)

**Why Semantic Kernel over AutoGen:**
- Native .NET integration (no Python interop)
- Mature plugin/function-calling model that maps well to our Skills concept
- Built-in Azure OpenAI support with streaming
- Process framework for multi-step orchestration
- Active Microsoft investment and enterprise adoption

**Agent Structure:**

Each agent is a Semantic Kernel `ChatCompletionAgent` with:
- A system prompt assembled from workspace files (AGENTS.md, SOUL.md, PROJECT.md, skills)
- A set of registered plugins (tools) specific to its role
- A session-scoped chat history
- Access to the skills loaded for its agent type

```csharp
public class AgentFactory
{
    public async Task<ChatCompletionAgent> CreateAgent(
        AgentConfig config,
        ProjectContext project,
        IEnumerable<Skill> skills)
    {
        var kernel = Kernel.CreateBuilder()
            .AddAzureOpenAIChatCompletion(config.Model, endpoint, apiKey)
            .Build();

        // Register tool plugins based on agent type + skills
        foreach (var skill in skills)
            kernel.Plugins.Add(skill.ToKernelPlugin());

        // Register D365 connector if agent needs environment access
        if (config.RequiresEnvironmentAccess)
            kernel.Plugins.Add(new D365ConnectorPlugin(project.Environments));

        // Register DevOps plugin if needed
        if (config.RequiresDevOps)
            kernel.Plugins.Add(new DevOpsPlugin(project.DevOpsConfig));

        var systemPrompt = await BuildSystemPrompt(config, project, skills);

        return new ChatCompletionAgent
        {
            Name = config.Id,
            Instructions = systemPrompt,
            Kernel = kernel,
            Arguments = new KernelArguments(
                new AzureOpenAIPromptExecutionSettings
                {
                    ToolCallBehavior = ToolCallBehavior.AutoInvokeKernelFunctions,
                    Temperature = 0.3f // Lower for configuration tasks
                })
        };
    }
}
```

**System Prompt Assembly (mirrors OpenClaw's workspace injection):**

```
[AGENTS.md]        → Agent behavior rules and constraints
[SOUL.md]          → Persona and tone (professional consultant)
[PROJECT.md]       → Customer context, modules, timeline, key decisions
[TOOLS.md]         → Environment-specific notes, tenant URLs
[Loaded Skills]    → Skill instructions for available capabilities
[Memory Context]   → Recent decisions, meeting summaries, requirements
```

### 4.3 Teams Integration Layer

**Technology:** Microsoft Bot Framework SDK v4 (.NET)

**Capabilities:**

1. **Agent-as-Bot-Member:** Each agent type can appear as a distinct bot in Teams channels. Consultants @mention the agent they need.

2. **Adaptive Cards:** Rich interactive responses for:
   - Configuration review/approval cards
   - Data validation result summaries
   - Requirement confirmation workflows
   - BPC category selection/editing
   - Migration status dashboards

3. **Proactive Messaging:** Agents can initiate conversations:
   - "Meeting recording processed — 14 requirements extracted. Review?"
   - "Nightly data validation found 3 errors in vendor master"
   - "Scheduled config deployment to UAT completed successfully"

4. **Channel Topology:**

```
Teams Organization
├── Implementation Team (Team)
│   ├── General                     → All agents available
│   ├── #discovery-contoso          → Discovery Agent bound
│   ├── #config-contoso-finance     → Config Agent (F&O) bound
│   ├── #config-contoso-ce          → Config Agent (CE) bound
│   ├── #migration-contoso          → Migration Agent bound
│   ├── #devops-contoso             → DevOps Agent bound
│   └── #meetings-contoso           → Meeting Intelligence output
├── Admin (Team)
│   ├── #agent-management           → Admin commands
│   ├── #skills-approval            → Skill submission/review
│   └── #system-health              → Alerts and diagnostics
```

5. **Activation Modes (from OpenClaw):**
   - `mention` (default): Agent responds only when @mentioned
   - `always`: Agent monitors all messages (useful for dedicated channels)
   - Configurable per channel binding

### 4.4 Skills Engine

**This is the most critical architectural decision for enterprise use.**

**Design Philosophy:** Users extend functionality through curated skills, never source code. Skills are declarative packages — prompt instructions + tool definitions + optional scripts — that go through an approval pipeline before activation.

**Skill Structure (AgentSkills-compatible, same as OpenClaw):**

```
skills/
├── ce-security-roles/
│   ├── SKILL.md                    # Frontmatter + instructions
│   ├── references/
│   │   ├── role-matrix.md          # Standard role templates
│   │   └── privilege-reference.md  # CE privilege documentation
│   ├── scripts/
│   │   └── validate-roles.ps1     # PowerShell validation script
│   └── assets/
│       └── role-template.xlsx      # Template for role design
│
├── fo-chart-of-accounts/
│   ├── SKILL.md
│   ├── references/
│   │   ├── coa-best-practices.md
│   │   └── dimension-patterns.md
│   └── scripts/
│       └── validate-coa.py
│
├── meeting-requirements-extract/
│   ├── SKILL.md
│   ├── references/
│   │   └── requirement-schema.md
│   └── scripts/
│       └── parse-transcript.py
```

**SKILL.md Format:**

```yaml
---
name: ce-security-roles
description: >
  Configure and validate Dynamics 365 CE security roles.
  Use when: user asks about security roles, access levels, team assignments,
  or privilege configuration in CE/Dataverse.
  NOT for: F&O security (use fo-security-roles), Azure AD/Entra configuration.
metadata:
  platform:
    category: "ce-configuration"
    riskLevel: "medium"          # low | medium | high | critical
    approvalStatus: "approved"   # draft | submitted | approved | deprecated
    approvedBy: "admin@company.com"
    approvedAt: "2026-03-15"
    version: "1.2.0"
    requires:
      products: ["CE"]
      permissions: ["d365.securityRole.read", "d365.securityRole.write"]
      environments: ["dev", "uat"]  # Never auto-execute on prod
    tags: ["security", "ce", "configuration"]
---

# CE Security Role Configuration

## When to Use
- Consultant asks to set up or modify security roles
- Requirements mention access control or data visibility
- BPA/BPC references role-based security

## Workflow

1. **Analyze requirements** from PROJECT.md and meeting artifacts
2. **Map to standard roles** using references/role-matrix.md
3. **Generate role definition** in Dataverse-importable format
4. **Validate** against privilege reference
5. **Present for review** via Adaptive Card in Teams
6. **Apply** only after human approval (NEVER auto-apply to UAT/Prod)

## Constraints
- Always show the consultant what will change before applying
- Log every role modification to the audit trail
- Flag any role that grants Organization-level Delete privileges

[Instructions continue...]
```

**Skill Lifecycle:**

```
┌──────────┐    ┌───────────┐    ┌──────────┐    ┌──────────┐
│  Draft   │───▶│ Submitted │───▶│ In Review│───▶│ Approved │
│          │    │           │    │          │    │          │
│ Author   │    │ Notify    │    │ Admin    │    │ Active   │
│ creates  │    │ admins    │    │ reviews  │    │ in prod  │
└──────────┘    └───────────┘    └──────────┘    └──────────┘
                                       │               │
                                       ▼               ▼
                                 ┌──────────┐   ┌───────────┐
                                 │ Rejected │   │Deprecated │
                                 │          │   │           │
                                 │ Feedback │   │ Replaced  │
                                 │ to author│   │ by newer  │
                                 └──────────┘   └───────────┘
```

**Gating (extended from OpenClaw's `metadata.openclaw.requires`):**

```csharp
public class SkillGate
{
    // From OpenClaw: binary/env/config requirements
    public List<string> RequiredBins { get; set; }
    public List<string> RequiredEnvVars { get; set; }
    public List<string> RequiredConfig { get; set; }

    // Enterprise extensions
    public List<string> RequiredProducts { get; set; }     // ["CE", "F&O"]
    public List<string> RequiredPermissions { get; set; }  // RBAC permissions
    public List<string> AllowedEnvironments { get; set; }  // ["dev", "uat"]
    public string RiskLevel { get; set; }                   // Affects approval flow
    public string ApprovalStatus { get; set; }              // Must be "approved"
}
```

### 4.5 Meeting Intelligence Pipeline

**The killer feature for consulting firms.** Transforms meeting recordings into structured implementation artifacts.

**Architecture:**

```
Teams Meeting Recording
        │
        ▼
┌─────────────────────┐
│ Graph API Listener  │  ← Webhook: new recording available
│ (Azure Function)    │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ Transcript Fetcher  │  ← Graph API: /communications/callRecords
│                     │     Pull .vtt transcript + recording URL
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ Pre-Processor       │  ← Chunk transcript, identify speakers,
│                     │     match to project/meeting type
└─────────┬───────────┘
          │
          ▼
┌─────────────────────────────────────────────┐
│ LLM Extraction Pipeline (Semantic Kernel)    │
│                                              │
│  Pass 1: Meeting Classification              │
│  ├── Type: BPA | Requirements | Design |     │
│  │         Sprint Planning | Status | Other  │
│  ├── Project: (match to known projects)      │
│  └── Participants: (match to known contacts) │
│                                              │
│  Pass 2: Content Extraction (type-specific)  │
│  ├── BPA Session →                           │
│  │   ├── Current state processes             │
│  │   ├── Pain points                         │
│  │   ├── Desired future state                │
│  │   └── Fit/gap observations                │
│  ├── Requirements Session →                  │
│  │   ├── Functional requirements             │
│  │   ├── Non-functional requirements         │
│  │   ├── Assumptions                         │
│  │   ├── Open questions                      │
│  │   └── Decisions made                      │
│  └── Design Session →                        │
│      ├── Configuration decisions             │
│      ├── Customization needs                 │
│      └── Integration requirements            │
│                                              │
│  Pass 3: Artifact Generation                 │
│  ├── Structured requirements (JSON)          │
│  ├── BPC entries (if BPA session)            │
│  ├── DevOps work items (draft)               │
│  ├── Meeting summary (markdown)              │
│  └── Action items with owners                │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌──────────────────────────────┐
│ Artifact Storage & Routing   │
│                              │
│  → Cosmos DB (structured)    │
│  → Blob Storage (files)      │
│  → DevOps (work items)       │
│  → Teams (notification)      │
│  → Memory (agent context)    │
└──────────────────────────────┘
```

**Meeting Type → Artifact Mapping:**

| Meeting Type | Primary Artifacts | Secondary Artifacts |
|---|---|---|
| BPA Session | Process maps, current/future state, pain points | BPC entries, fit/gap items |
| Requirements Gathering | Functional requirements, acceptance criteria | DevOps user stories, open questions |
| Design Workshop | Configuration specs, data model changes | Technical design doc, customization backlog |
| Sprint Planning | Sprint backlog, task assignments | DevOps sprint items |
| Status/Steering | Decision log, risk register updates | Action items, meeting minutes |
| Data Review | Data mapping updates, validation rules | Migration backlog items |

### 4.6 Scheduler & Task Engine

**Technology:** Azure Durable Functions (orchestration) + Gateway-embedded scheduler (simple recurring)

**Two-tier approach (mirrors OpenClaw's cron + heartbeat pattern):**

**Tier 1: Gateway Scheduler (lightweight, in-process)**
- Simple recurring checks (heartbeat-equivalent)
- Agent health monitoring
- Session maintenance (pruning, rotation)
- Cache refresh

**Tier 2: Durable Functions (complex, isolated)**
- Scheduled tasks with approval workflows
- Long-running orchestrations (data migration runs)
- Fan-out/fan-in patterns (validate 10K records in parallel)
- Retry with backoff and dead-letter queues

**Admin-Created Scheduled Tasks:**

```json5
{
  "id": "nightly-data-validation",
  "name": "Nightly Data Validation - Contoso",
  "schedule": { "kind": "cron", "expression": "0 2 * * *", "tz": "America/Chicago" },
  "project": "contoso-fno",
  "agent": "migration",
  "task": {
    "kind": "agentTurn",
    "message": "Run data validation on latest vendor master migration batch. Report errors to #migration-contoso channel.",
    "skills": ["fo-data-validation", "fo-vendor-master"],
    "maxDurationMinutes": 30
  },
  "delivery": {
    "channel": "teams",
    "target": "migration-contoso",
    "onSuccess": "summary",      // summary | full | silent
    "onFailure": "alert"          // alert | summary | silent
  },
  "approval": {
    "required": false,            // No approval needed for read-only validation
    "requiredForWrite": true      // But any fix actions need approval
  },
  "createdBy": "consultant@company.com",
  "createdAt": "2026-03-15T10:00:00Z"
}
```

**Task Categories:**

| Category | Examples | Approval Required |
|---|---|---|
| Monitoring | Environment health, data quality checks | No |
| Reporting | Weekly status summary, metrics rollup | No |
| Data Operations | Validation runs, export jobs | Read: No, Write: Yes |
| Configuration | Apply config packages, enable features | Always |
| Deployment | Solution deployment, data migration push | Always + multi-approver |

### 4.7 Blazor Admin UI

**Technology:** Blazor Server (.NET 8) hosted alongside the Gateway

**Why Blazor Server (not WASM):**
- Real-time SignalR connection to Gateway (same server)
- No client-side secrets (Azure AD tokens stay server-side)
- Faster initial load (critical for admin tools)
- Full .NET library access (Semantic Kernel, Graph SDK)

**Page Layout:**

```
┌────────────────────────────────────────────────────────┐
│  D365 AI Platform          [Project ▼]  [Admin ▼]  🔔 │
├────────────┬───────────────────────────────────────────┤
│            │                                           │
│  Dashboard │  Content Area                             │
│  Projects  │                                           │
│  Agents    │  (varies by selected nav item)             │
│  Skills    │                                           │
│  Meetings  │                                           │
│  Tasks     │                                           │
│  DevOps    │                                           │
│  Settings  │                                           │
│            │                                           │
│  ───────── │                                           │
│  System    │                                           │
│  Health    │                                           │
│  Audit Log │                                           │
│            │                                           │
└────────────┴───────────────────────────────────────────┘
```

**Key Views:**

1. **Project Dashboard**
   - Implementation phase progress (Discovery → Config → Migration → Test → Go-Live)
   - Agent activity feed (recent conversations, actions taken)
   - Requirements coverage (extracted vs confirmed vs implemented)
   - Upcoming milestones and scheduled tasks

2. **Meeting Intelligence Viewer**
   - Recording playback with synced transcript
   - Extracted artifacts sidebar (requirements, decisions, action items)
   - Artifact status (draft → reviewed → confirmed → implemented)
   - Link artifacts to DevOps work items

3. **Skills Management**
   - Browse installed skills by category (CE, F&O, Cross-product, Meeting)
   - Skill detail view (instructions, risk level, usage stats)
   - Submission form for new skills
   - Admin approval queue with diff viewer
   - Skill activation/deactivation per project

4. **Scheduled Task Builder**
   - Visual task creator (schedule, agent, message, delivery)
   - Task calendar view
   - Execution history with logs
   - Enable/disable/delete tasks

5. **Configuration Review**
   - Side-by-side: proposed config vs current environment state
   - Approval workflow (request → review → approve → apply)
   - Rollback capability
   - Configuration drift detection

### 4.8 D365 Environment Connectors

**The bridge between agents and actual D365 tenants.**

**CE Connector (Dataverse Web API):**

```csharp
public interface ICEConnector
{
    // Read operations
    Task<EntityMetadata> GetEntityMetadata(string entityName);
    Task<SecurityRole[]> GetSecurityRoles();
    Task<BusinessUnit[]> GetBusinessUnits();
    Task<Workflow[]> GetWorkflows(string entityName);
    Task<SystemForm[]> GetForms(string entityName);
    Task<SavedQuery[]> GetViews(string entityName);

    // Configuration operations (require approval)
    Task<Result> CreateSecurityRole(SecurityRoleDefinition role);
    Task<Result> UpdateBusinessRule(BusinessRuleDefinition rule);
    Task<Result> ImportSolution(byte[] solutionZip);
    Task<Result> ApplyConfigurationPackage(ConfigPackage package);

    // Data operations
    Task<DataValidationResult> ValidateData(string entity, ValidationRules rules);
    Task<ImportResult> ImportData(string entity, DataPayload data);
    Task<ExportResult> ExportData(string entity, QueryExpression query);
}
```

**F&O Connector (Data Entities + OData):**

```csharp
public interface IFOConnector
{
    // Read operations
    Task<DataEntity[]> GetDataEntities();
    Task<NumberSequence[]> GetNumberSequences();
    Task<LegalEntity[]> GetLegalEntities();
    Task<ChartOfAccounts> GetChartOfAccounts(string company);
    Task<BatchJob[]> GetBatchJobs();

    // Configuration operations (require approval)
    Task<Result> ConfigureNumberSequence(NumberSequenceDefinition def);
    Task<Result> SetupLegalEntity(LegalEntityDefinition def);
    Task<Result> ImportChartOfAccounts(CoAPackage package);
    Task<Result> ConfigureBatchJob(BatchJobDefinition job);

    // Data operations
    Task<DataValidationResult> ValidateDataEntity(string entity, ValidationRules rules);
    Task<ImportResult> ImportDataPackage(DMFPackage package);
    Task<ExportResult> ExportDataPackage(string project);
}
```

**Environment Security Model:**

```
┌─────────────────────────────────────────────┐
│ Agent requests D365 operation                │
│                                              │
│   ┌───────────────┐                          │
│   │ Is it read?   │──Yes──▶ Execute directly │
│   └───────┬───────┘                          │
│           │ No (write)                       │
│           ▼                                  │
│   ┌───────────────────┐                      │
│   │ Which environment?│                      │
│   └───────┬───────────┘                      │
│           │                                  │
│     ┌─────┼──────┐                           │
│     ▼     ▼      ▼                           │
│   [Dev] [UAT]  [Prod]                        │
│     │     │      │                           │
│     ▼     ▼      ▼                           │
│   Auto  Single  Multi-                       │
│   apply approve approve                      │
│         gate    gate                          │
└─────────────────────────────────────────────┘
```

### 4.9 DevOps Integration

**Technology:** Azure DevOps REST API

**Sync Points:**

1. **Requirements → Work Items**
   - Meeting-extracted requirements become User Stories (draft)
   - Consultant reviews and confirms in Teams or Blazor UI
   - Confirmed requirements sync as approved User Stories
   - Linked to source meeting recording + transcript

2. **Agent Activity → Work Item Updates**
   - Config agent completes setup → linked task marked done
   - Migration agent finds errors → bug work items created
   - Solution agent generates code → PR created with linked work items

3. **Sprint/Board Awareness**
   - Agents can read current sprint scope
   - Status meetings get auto-generated sprint summaries
   - Blocked items surface proactively in Teams

```csharp
public interface IDevOpsConnector
{
    // Work Items
    Task<WorkItem> CreateWorkItem(WorkItemType type, WorkItemFields fields);
    Task<WorkItem> UpdateWorkItem(int id, WorkItemFields fields);
    Task<WorkItem[]> QueryWorkItems(string wiql);

    // Repos & PRs
    Task<PullRequest> CreatePullRequest(PRDefinition pr);
    Task<BuildResult> QueueBuild(string pipelineId, Dictionary<string, string> params);

    // Boards
    Task<Sprint> GetCurrentSprint(string teamId);
    Task<BoardColumn[]> GetBoardState(string teamId);
}
```

---

## 5. Agent Catalog

### Discovery Agent

**Purpose:** Process meeting recordings and extract implementation artifacts

**Skills:** meeting-requirements-extract, bpa-session-processor, stakeholder-mapping, fit-gap-analysis

**Tools:** Meeting Intelligence Pipeline, Blob Storage, DevOps (create work items)

**Activation:** Automatic (meeting recording webhook) + @mention in discovery channels

**Outputs:** Requirements documents, BPA summaries, fit/gap matrices, stakeholder maps

### BPC Agent

**Purpose:** Build and maintain the Business Process Catalog

**Skills:** bpc-catalog-generator, process-mapping, cross-module-dependency

**Tools:** Cosmos DB (BPC store), DevOps (link to requirements), D365 connector (read current config)

**Activation:** @mention + triggered after Discovery Agent produces requirements

**Outputs:** BPC documents, process flow diagrams (mermaid), module dependency maps

### Config Agent (CE)

**Purpose:** Translate approved requirements into CE/Dataverse configuration

**Skills:** ce-security-roles, ce-business-rules, ce-workflows, ce-forms-views, ce-solution-packaging

**Tools:** CE Connector (read + write with approval), DevOps (update tasks)

**Activation:** @mention in CE config channel + scheduled config review tasks

**Outputs:** Configuration packages, security role definitions, solution XML

### Config Agent (F&O)

**Purpose:** Translate approved requirements into F&O configuration

**Skills:** fo-chart-of-accounts, fo-number-sequences, fo-legal-entities, fo-batch-jobs, fo-dimension-framework, fo-workflow-config

**Tools:** F&O Connector (read + write with approval), DevOps (update tasks)

**Activation:** @mention in F&O config channel + scheduled config tasks

**Outputs:** Data packages, configuration guides, setup validation reports

### Migration Agent

**Purpose:** Orchestrate and validate data migration

**Skills:** data-mapping, data-validation, data-transform, dmf-package-builder, migration-runbook

**Tools:** CE/F&O Connectors (data operations), Blob Storage (staging files), DevOps (track migration tasks)

**Activation:** @mention + scheduled nightly validation runs

**Outputs:** Data mapping documents, validation reports, migration runbooks, error resolution guides

### DevOps Agent

**Purpose:** Sync implementation artifacts with Azure DevOps, manage builds and deployments

**Skills:** devops-work-items, devops-boards, devops-pipelines, sprint-management

**Tools:** DevOps Connector (full API), Teams (status notifications)

**Activation:** @mention + automatic (triggered by other agents creating artifacts)

**Outputs:** Work items, sprint summaries, build status reports, deployment logs

### Solution Agent

**Purpose:** Generate X++ (F&O) or C#/Plugin code (CE) based on approved specifications

**Skills:** ce-plugin-development, ce-webresource-patterns, fo-xpp-patterns, fo-extension-patterns, unit-test-generation

**Tools:** DevOps (create branches, PRs), code generation tools, static analysis

**Activation:** @mention only (never automatic — code generation requires explicit request)

**Outputs:** Source code in PR (never committed directly), unit tests, technical documentation

**Safety constraints:**
- ALL generated code goes through PR — never direct commit
- Requires human code review before merge
- Static analysis must pass before PR is created
- Agent flags any generated code that modifies base objects

---

## 6. Data Architecture

### Storage Design

```
┌─────────────────────────────────────────────────────────────┐
│                     Azure Cosmos DB                          │
│                     (NoSQL, multi-region)                    │
│                                                              │
│  Container: sessions                                         │
│  ├── Partition: /projectId                                   │
│  ├── Documents: session state, chat history, token counts    │
│  └── TTL: 90 days (configurable per project)                 │
│                                                              │
│  Container: projects                                         │
│  ├── Partition: /projectId                                   │
│  ├── Documents: project config, agent bindings, env config   │
│  └── TTL: none                                               │
│                                                              │
│  Container: skills                                           │
│  ├── Partition: /category                                    │
│  ├── Documents: skill metadata, approval records, versions   │
│  └── TTL: none                                               │
│                                                              │
│  Container: audit                                            │
│  ├── Partition: /projectId                                   │
│  ├── Documents: every agent action, approval, config change  │
│  └── TTL: 365 days (compliance)                              │
│                                                              │
│  Container: artifacts                                        │
│  ├── Partition: /projectId                                   │
│  ├── Documents: requirements, BPC entries, config specs      │
│  └── TTL: none                                               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     Azure Blob Storage                        │
│                                                              │
│  Container: meeting-recordings                               │
│  ├── Path: /{projectId}/{meetingId}/recording.mp4            │
│  ├── Path: /{projectId}/{meetingId}/transcript.vtt           │
│  └── Access: private, SAS token for playback                 │
│                                                              │
│  Container: agent-workspaces                                 │
│  ├── Path: /{agentId}/AGENTS.md                              │
│  ├── Path: /{agentId}/SOUL.md                                │
│  ├── Path: /{agentId}/skills/{skillName}/SKILL.md            │
│  └── Path: /{agentId}/memory/{date}.md                       │
│                                                              │
│  Container: data-migration                                   │
│  ├── Path: /{projectId}/staging/{entity}/{batch}.csv         │
│  ├── Path: /{projectId}/validated/{entity}/{batch}.csv       │
│  └── Path: /{projectId}/packages/{package}.zip               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     Azure Redis Cache                        │
│                                                              │
│  • Active session cache (avoid Cosmos reads on every msg)    │
│  • Agent availability / health status                        │
│  • Skill metadata cache (reload on change)                   │
│  • Rate limiting counters                                    │
│  • Pub/sub for real-time UI updates                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     Azure SQL Database                        │
│                     (structured / relational)                 │
│                                                              │
│  Tables:                                                     │
│  • Projects (master data)                                    │
│  • Users (RBAC, project assignments)                         │
│  • ScheduledTasks (cron definitions)                         │
│  • TaskExecutions (run history)                              │
│  • SkillApprovals (workflow state)                            │
│  • EnvironmentConnections (D365 tenant configs)              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Security & Access Control

### Authentication & Authorization

```
┌────────────────────────────────────────────────────┐
│                 Azure AD (Entra ID)                 │
│                                                     │
│  App Registration: "D365 AI Platform"               │
│  ├── API Permissions:                               │
│  │   ├── Microsoft Graph (Calendars, Teams, Users)  │
│  │   ├── Dynamics 365 (per environment)              │
│  │   └── Azure DevOps (full access)                  │
│  │                                                   │
│  └── App Roles:                                      │
│      ├── Platform.Admin                              │
│      ├── Platform.ProjectLead                        │
│      ├── Platform.Consultant                         │
│      └── Platform.Viewer                             │
└────────────────────────────────────────────────────┘
```

### Role-Based Access Control

| Capability | Admin | Project Lead | Consultant | Viewer |
|---|:---:|:---:|:---:|:---:|
| View dashboards | ✅ | ✅ | ✅ (own projects) | ✅ (own projects) |
| Chat with agents | ✅ | ✅ | ✅ | ❌ |
| Create scheduled tasks | ✅ | ✅ | ❌ | ❌ |
| Approve config changes | ✅ | ✅ | ❌ | ❌ |
| Manage skills | ✅ | ❌ | ❌ | ❌ |
| Submit skills | ✅ | ✅ | ✅ | ❌ |
| Manage agents | ✅ | ❌ | ❌ | ❌ |
| Access Prod environments | ✅ | ✅ (read) | ❌ | ❌ |
| View audit logs | ✅ | ✅ (own projects) | ❌ | ❌ |

### Agent Action Classification

```csharp
public enum ActionRisk
{
    Read,          // No approval — read D365 config, query data
    LowWrite,     // Auto-log — update work items, write to memory
    MediumWrite,  // Single approval — apply config to Dev
    HighWrite,    // Single approval — apply config to UAT, run migration
    Critical      // Multi-approval — anything touching Prod, deploy solutions
}
```

### Data Protection

- All Cosmos DB data encrypted at rest (Microsoft-managed keys)
- All transit encrypted (TLS 1.3)
- Meeting recordings stored in customer-isolated blob containers
- PII detection in agent outputs (flag before external delivery)
- Session transcripts auto-purged after configurable retention (default 90 days)
- No customer data leaves Azure tenant boundaries
- Agent-generated code scanned for secrets before PR creation

---

## 8. Skills Framework (Detailed)

### Skill Categories

```
Platform Skills (bundled, immutable)
├── core/
│   ├── web-search              # Search documentation, KB articles
│   ├── file-operations         # Read/write workspace files
│   ├── memory-management       # Agent memory read/write
│   └── teams-messaging         # Send Adaptive Cards, proactive messages
│
├── meeting-intelligence/
│   ├── meeting-classifier      # Classify meeting type
│   ├── requirement-extractor   # Extract requirements from transcripts
│   ├── bpa-processor           # Process BPA session recordings
│   ├── action-item-extractor   # Extract action items + owners
│   └── decision-logger         # Extract and log decisions
│
├── ce/
│   ├── ce-security-roles       # Security role configuration
│   ├── ce-business-rules       # Business rule creation
│   ├── ce-workflows            # Workflow/Power Automate flows
│   ├── ce-forms-views          # Form and view customization
│   ├── ce-solution-packaging   # Solution export/import
│   ├── ce-plugin-patterns      # Plugin code generation patterns
│   └── ce-data-model           # Entity/field configuration
│
├── fo/
│   ├── fo-chart-of-accounts    # Chart of accounts setup
│   ├── fo-number-sequences     # Number sequence configuration
│   ├── fo-legal-entities       # Legal entity setup
│   ├── fo-dimension-framework  # Financial dimensions
│   ├── fo-batch-jobs           # Batch job configuration
│   ├── fo-workflow-config      # Workflow configuration
│   ├── fo-xpp-patterns         # X++ code generation patterns
│   └── fo-data-entities        # Data entity configuration
│
├── migration/
│   ├── data-mapping            # Source → target field mapping
│   ├── data-validation         # Validation rule engine
│   ├── data-transform          # Transformation scripts
│   ├── dmf-package-builder     # Data Management Framework packages
│   └── migration-runbook       # Runbook generation
│
└── devops/
    ├── work-item-management    # Create/update/query work items
    ├── sprint-management       # Sprint planning assistance
    ├── pipeline-management     # Build/release pipeline ops
    └── pr-management           # Pull request creation/review

Approved Skills (admin-curated, per-organization)
├── industry/
│   ├── manufacturing-bom       # Manufacturing BOM patterns
│   ├── retail-pos-config       # Retail POS configuration
│   └── healthcare-compliance   # Healthcare-specific compliance
│
└── isv/
    ├── avalara-tax-setup       # Avalara tax integration
    ├── celigo-integration      # Celigo connector patterns
    └── binary-stream-config    # Binary Stream configuration
```

### Skill Inheritance & Override

```
Resolution order (highest precedence first):
1. Project workspace skills    → /workspaces/{projectId}/skills/
2. Organization skills         → /workspaces/shared/skills/
3. Platform skills (bundled)   → /app/skills/

Same name at higher level overrides lower level.
Example: project customizes "ce-security-roles" with client-specific role templates.
```

---

## 9. Session & Memory Model

### Session Keys (adapted from OpenClaw)

```
Pattern: project:{projectId}:agent:{agentId}:{channelKey}

Examples:
  project:contoso-fno:agent:discovery:teams-discovery-channel
  project:contoso-fno:agent:config-fo:teams-config-channel
  project:contoso-fno:agent:migration:task-nightly-validation
  project:contoso-fno:agent:bpc:direct-consultant-123
```

### Memory Architecture (adapted from OpenClaw's MEMORY.md pattern)

```
Per-Agent Memory:
├── AGENTS.md           → Behavior rules (immutable per agent type)
├── SOUL.md             → Persona definition
├── PROJECT.md          → Project context (auto-updated)
│   ├── Customer name, industry, size
│   ├── Modules in scope
│   ├── Key stakeholders
│   ├── Timeline and milestones
│   ├── Major decisions log
│   └── Known constraints / risks
├── MEMORY.md           → Long-term curated memory
│   ├── Lessons learned this project
│   ├── Customer preferences ("they hate pop-ups")
│   ├── Integration quirks discovered
│   └── What worked / what didn't
└── memory/
    ├── requirements/
    │   ├── finance-requirements.md
    │   ├── supply-chain-requirements.md
    │   └── hr-requirements.md
    ├── decisions/
    │   ├── 2026-03-01.md  → "Decided: single legal entity"
    │   └── 2026-03-10.md  → "Decided: custom invoice format"
    └── meetings/
        ├── 2026-03-05-bpa-finance.md
        └── 2026-03-12-requirements-scm.md
```

### Session Maintenance

Same patterns as OpenClaw:
- `pruneAfter`: 90 days (longer for enterprise — audit trails)
- `maxEntries`: 1000 per project
- `rotateBytes`: 50MB per session transcript
- Archived to cold storage (Blob) before deletion

---

## 10. Deployment Architecture

### Azure Resources

```
Resource Group: rg-d365-ai-platform-{env}
│
├── Azure Container Apps Environment
│   ├── Gateway Service (2-4 replicas, auto-scale)
│   ├── Meeting Intelligence Workers (0-N, event-driven scale)
│   └── Blazor Admin UI (2 replicas)
│
├── Azure Functions (Consumption plan)
│   ├── Meeting Recording Webhook Listener
│   ├── Durable Task Orchestrators
│   └── DevOps Webhook Handlers
│
├── Azure Cosmos DB (serverless or provisioned)
│   └── Database: d365-ai-platform
│
├── Azure Blob Storage (LRS)
│   └── Containers: recordings, workspaces, migration, exports
│
├── Azure Redis Cache (Standard C1)
│
├── Azure SQL Database (S2)
│
├── Azure Key Vault
│   └── D365 connection strings, API keys, bot secrets
│
├── Azure Application Insights
│   └── Telemetry, distributed tracing, availability tests
│
├── Azure Bot Service
│   └── Bot registrations per agent type
│
└── Azure OpenAI Service
    └── GPT-4o deployment (primary model)
```

### Estimated Monthly Cost (Production)

| Resource | SKU | Est. Monthly |
|---|---|---|
| Container Apps | 4 vCPU, 8 GB × 3 apps | $200-400 |
| Azure Functions | Consumption (event-driven) | $50-100 |
| Cosmos DB | Serverless (~100K RU/month) | $100-200 |
| Blob Storage | 500 GB LRS | $10 |
| Redis Cache | Standard C1 | $80 |
| Azure SQL | S2 (50 DTU) | $75 |
| Azure OpenAI | GPT-4o (~2M tokens/day) | $300-600 |
| Bot Service | Standard | $0 (free tier) |
| Key Vault | Standard | $5 |
| App Insights | 5 GB/month ingestion | $15 |
| **Total** | | **$835-1,485/mo** |

---

## 11. Integration Points

```
┌──────────────────────────────────────────────────────────┐
│                    EXTERNAL SYSTEMS                       │
│                                                          │
│  Microsoft Graph API                                     │
│  ├── Teams: channels, messages, adaptive cards           │
│  ├── Calendar: meeting schedules                         │
│  ├── Users: directory lookup, presence                   │
│  ├── OneDrive: shared files                              │
│  └── Communications: call records, transcripts           │
│                                                          │
│  Dynamics 365 CE (Dataverse Web API)                     │
│  ├── Metadata: entities, attributes, relationships       │
│  ├── Configuration: security, workflows, business rules  │
│  ├── Data: CRUD, bulk operations                         │
│  └── Solutions: import/export                            │
│                                                          │
│  Dynamics 365 F&O (OData + Data Entities)                │
│  ├── Configuration: parameters, setup tables             │
│  ├── Data: DMF packages, data entities                   │
│  ├── Batch: job scheduling, monitoring                   │
│  └── Deployment: LCS API (if available)                  │
│                                                          │
│  Azure DevOps REST API                                   │
│  ├── Work Items: create, update, query (WIQL)            │
│  ├── Boards: sprint, backlog, columns                    │
│  ├── Repos: branches, PRs, commits                       │
│  ├── Pipelines: builds, releases                         │
│  └── Webhooks: push events to platform                   │
│                                                          │
│  Azure OpenAI Service                                    │
│  ├── Chat Completions (GPT-4o, GPT-4o-mini)             │
│  ├── Embeddings (for semantic search over artifacts)     │
│  └── Whisper (backup transcription)                      │
│                                                          │
│  Power Platform Admin API (optional)                     │
│  ├── Environment management                              │
│  ├── Solution deployment                                 │
│  └── DLP policy awareness                                │
└──────────────────────────────────────────────────────────┘
```

---

## 12. Build Phases & Roadmap

### Phase 1: Foundation (Weeks 1-8)

**Goal:** Single agent talking in Teams, basic skill loading, admin shell

**Deliverables:**
- [ ] .NET 8 Gateway service scaffolding (ASP.NET Core + SignalR)
- [ ] Bot Framework integration (single bot, Teams channel)
- [ ] Message router (channel → session → agent)
- [ ] Semantic Kernel agent runtime (single agent, chat completions)
- [ ] Session store (Cosmos DB, basic CRUD)
- [ ] Skill loader (parse SKILL.md frontmatter, inject into system prompt)
- [ ] 3-5 platform skills (web-search, file-ops, memory, teams-messaging)
- [ ] Blazor shell (auth, navigation, placeholder pages)
- [ ] Azure AD authentication (admin role only)
- [ ] CI/CD pipeline (Azure DevOps → Container Apps)
- [ ] Basic health endpoint and logging (App Insights)

**Demo:** @mention agent in Teams channel → agent responds with skill-augmented answers

### Phase 2: Meeting Intelligence (Weeks 9-16)

**Goal:** Meeting recordings auto-processed, artifacts extracted and stored

**Deliverables:**
- [ ] Graph API webhook for meeting recordings
- [ ] Transcript fetcher and pre-processor
- [ ] LLM extraction pipeline (3-pass: classify, extract, generate)
- [ ] Meeting type classifiers (BPA, requirements, design, status)
- [ ] Artifact storage (Cosmos DB + Blob)
- [ ] Meeting skills (requirement-extractor, bpa-processor, action-item-extractor)
- [ ] Teams proactive notifications ("14 requirements extracted from today's BPA session")
- [ ] Blazor: Meeting Intelligence Viewer (recording + transcript + artifacts)
- [ ] Artifact → DevOps work item linking (manual trigger)

**Demo:** Record a mock BPA session → platform extracts requirements, creates draft work items, notifies team

### Phase 3: Multi-Agent & Skills Engine (Weeks 17-24)

**Goal:** Multiple specialized agents, approval-gated skills, scheduled tasks

**Deliverables:**
- [ ] Multi-agent routing (bindings config, per-channel agent assignment)
- [ ] Agent workspace isolation (per-agent SOUL.md, PROJECT.md, memory)
- [ ] Discovery Agent + BPC Agent (fully operational)
- [ ] Skill approval pipeline (submit → review → approve → activate)
- [ ] Skill management UI in Blazor
- [ ] Scheduler implementation (Durable Functions for complex, in-process for simple)
- [ ] Scheduled task admin UI in Blazor
- [ ] RBAC expansion (Project Lead, Consultant, Viewer roles)
- [ ] Project setup wizard (Blazor)
- [ ] Audit logging (every agent action → Cosmos audit container)

**Demo:** Create project → assign agents to channels → process meetings → BPC auto-generated → skills managed via UI

### Phase 4: D365 Connectors (Weeks 25-36)

**Goal:** Agents can read/write D365 environments with approval gates

**Deliverables:**
- [ ] CE Connector (Dataverse Web API, read + write)
- [ ] F&O Connector (OData + Data Entities, read + write)
- [ ] Environment pairing (OAuth consent per tenant, stored in Key Vault)
- [ ] Config Agent (CE) — reads requirements, proposes config, applies after approval
- [ ] Config Agent (F&O) — same pattern for F&O
- [ ] Migration Agent — data validation, mapping, DMF package generation
- [ ] Approval workflow for write operations (Adaptive Cards in Teams)
- [ ] Configuration review UI in Blazor (proposed vs current state)
- [ ] Environment health monitoring
- [ ] 15+ CE skills, 15+ F&O skills (configuration categories)

**Demo:** Requirements → Config Agent proposes security roles → consultant approves in Teams → applied to Dev → validated

### Phase 5: DevOps & Code Generation (Weeks 37-48)

**Goal:** Full DevOps integration, solution agent for code generation

**Deliverables:**
- [ ] DevOps Agent (work items, boards, pipelines, PRs)
- [ ] Bi-directional sync (artifacts ↔ work items)
- [ ] Solution Agent (CE plugins, F&O X++, with heavy guardrails)
- [ ] PR creation with auto-generated code + tests
- [ ] Sprint summary auto-generation
- [ ] Pipeline trigger and monitoring
- [ ] Blazor DevOps dashboard
- [ ] Code review assistance (agent reviews PRs for D365 best practices)

**Demo:** Consultant describes customization need → Solution Agent generates code + tests → PR created → DevOps Agent tracks through sprint

---

## 13. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|:---:|:---:|---|
| LLM generates incorrect D365 config | High | Critical | All writes require human approval; read-before-write validation; never auto-apply to Prod |
| Meeting transcription quality varies | Medium | Medium | Multi-pass extraction with confidence scores; flag low-confidence artifacts for human review |
| Consultant trust/adoption resistance | Medium | High | Start with read-only value (meeting intelligence); gradual capability rollout; always show agent reasoning |
| Skills contain errors or bad patterns | Medium | High | Mandatory approval pipeline; version control; risk-level classification; rollback capability |
| Token costs exceed budget | Medium | Medium | Per-project token budgets; model tiering (GPT-4o-mini for classification, GPT-4o for generation); caching |
| D365 API rate limits | Low | Medium | Request queuing; exponential backoff; batch operations where possible |
| Data leakage between projects | Low | Critical | Strict session isolation; per-project Cosmos partitions; RBAC enforcement; audit everything |
| Azure OpenAI service outage | Low | High | Fallback model configuration; queue pending requests; graceful degradation (notify, don't fail) |

---

## Appendix: OpenClaw Pattern Mapping

Detailed mapping of every OpenClaw architectural pattern to its equivalent in this platform.

| OpenClaw Component | OpenClaw Implementation | D365 Platform Equivalent |
|---|---|---|
| **Gateway** | Node.js process, WS on :18789 | ASP.NET Core 8, SignalR + REST |
| **Channel plugins** | Baileys, grammY, discord.js, Bolt | Bot Framework SDK (Teams only initially) |
| **Agent runtime** | Pi agent (RPC mode) | Semantic Kernel ChatCompletionAgent |
| **Multi-agent routing** | `agents.list[]` + `bindings[]` in config | Same pattern, project-scoped bindings |
| **Session model** | `agent:<id>:<key>`, JSONL transcripts | `project:<id>:agent:<id>:<key>`, Cosmos DB |
| **Session maintenance** | File pruning, rotation, disk budgets | Cosmos TTL, cold-archive to Blob |
| **Skills** | SKILL.md + frontmatter + gating | Same format + approval pipeline + risk levels |
| **Skill tiers** | Bundled → managed → workspace | Platform → approved → project workspace |
| **Skill gating** | `metadata.openclaw.requires` (bins, env, config) | Extended: + products, permissions, environments |
| **ClawHub registry** | Public registry at clawhub.com | Internal registry (Azure DevOps Artifacts or Cosmos) |
| **Workspace files** | AGENTS.md, SOUL.md, USER.md, TOOLS.md | AGENTS.md, SOUL.md, PROJECT.md, TOOLS.md |
| **Memory** | MEMORY.md + memory/YYYY-MM-DD.md | Same + memory/requirements/ + memory/decisions/ |
| **Cron scheduler** | `~/.openclaw/cron/jobs.json`, main + isolated | Durable Functions + in-process scheduler |
| **Heartbeat** | HEARTBEAT.md, periodic agent wakeup | Same pattern, health + cache refresh |
| **Nodes** | macOS/iOS/Android device nodes via WS | D365 environment connectors via OAuth |
| **Canvas / UI** | A2UI, agent-driven HTML push | Blazor Server admin UI |
| **Control UI** | Web dashboard served from Gateway | Blazor app (full admin, not just monitoring) |
| **WebChat** | Browser chat UI | Teams is the chat surface |
| **DM pairing** | Pairing codes, allowlists | Azure AD authentication |
| **Sandbox** | Docker isolation for non-main sessions | Agent action classification + approval gates |
| **Config** | `openclaw.json` (JSON5, hot reload) | `appsettings.json` + Cosmos config store |
| **Doctor** | `openclaw doctor` (diagnostics + fix) | Health endpoint + Blazor diagnostics page |
| **Audit** | Session transcripts | Dedicated audit container (Cosmos) + App Insights |
| **Plugin SDK** | `openclaw/plugin-sdk` exports per channel | `IChannelAdapter` interface (extensible for future channels) |
| **Webhook** | Gateway webhook endpoints | Azure Functions webhook handlers |
| **Media pipeline** | Image/audio/video transcription hooks | Meeting recording pipeline (Graph API → LLM) |
| **Browser control** | Managed Chrome/CDP | Not applicable (D365 APIs instead) |

---

## Next Steps

1. **Review this document** with technical leadership and implementation practice leads
2. **Validate agent catalog** with senior consultants (are these the right agents?)
3. **Prototype Phase 1** (4-week spike: Gateway + single Teams bot + Semantic Kernel)
4. **Estimate effort** with development team based on this architecture
5. **Secure Azure OpenAI capacity** (apply for quota if needed)
6. **Define 3 pilot projects** for early testing (Phase 2-3)

---

*This is a living document. Update it as architectural decisions are made and validated.*
