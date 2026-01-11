# Canonical Data Model: The Campaign Spine

## Building the System of Record for Marketing Operations

### Version: 2.0
### Date: January 10, 2026
### Author: Principal Engineer / Solution Architect

---

## Table of Contents

1. [The Problem: Data Fragmentation](#1-the-problem-data-fragmentation)
2. [The Solution: Canonical Campaign Spine](#2-the-solution-canonical-campaign-spine)
3. [Core Entity Model](#3-core-entity-model)
4. [Entity Schemas](#4-entity-schemas)
5. [Field Authority Matrix](#5-field-authority-matrix)
6. [Cross-System ID Mapping](#6-cross-system-id-mapping)
7. [Data Flow Architecture](#7-data-flow-architecture)
8. [Measurement Mapping](#8-measurement-mapping)
9. [Schema Evolution Strategy](#9-schema-evolution-strategy)
10. [Implementation Guidelines](#10-implementation-guidelines)

---

## 1. The Problem: Data Fragmentation

### 1.1 Current State: Islands of Data

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CURRENT STATE: FRAGMENTED DATA                        │
│                                                                          │
│   Each system has its own identifier, schema, and truth about a campaign│
│                                                                          │
│   ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐  │
│   │    WORKFRONT    │     │      ASANA      │     │    ANALYTICS    │  │
│   │                 │     │                 │     │                 │  │
│   │ Project: WF-123 │     │ Project: 12345  │     │ Campaign: abc   │  │
│   │ Name: "Q1       │     │ Name: "Q1       │     │ Channel: email  │  │
│   │       Launch"   │     │       Launch    │     │ Impressions:    │  │
│   │ Status: Active  │     │       Event"    │     │   500,000       │  │
│   │ Tasks: 45       │     │ Status: On Track│     │                 │  │
│   │                 │     │ Tasks: 23       │     │                 │  │
│   └────────┬────────┘     └────────┬────────┘     └────────┬────────┘  │
│            │                       │                        │           │
│            │                       │                        │           │
│            ▼                       ▼                        ▼           │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │                         QUESTIONS:                               │  │
│   │                                                                   │  │
│   │  • Is WF-123 the same campaign as Asana 12345?                  │  │
│   │  • Which status is correct: "Active" or "On Track"?             │  │
│   │  • How do 500K impressions relate to the 45 Workfront tasks?    │  │
│   │  • What's the total budget across all systems?                  │  │
│   │  • Who approved what, and when?                                 │  │
│   │                                                                   │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│   CONSEQUENCES:                                                         │
│   • Manual reconciliation (hours per week)                              │
│   • Conflicting reports to leadership                                   │
│   • No unified campaign view                                            │
│   • Attribution gaps                                                    │
│   • Audit nightmares                                                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 The Fundamental Challenge

Marketing operations span multiple systems, each optimized for different purposes:

| System | Purpose | Data Strength | Data Weakness |
|--------|---------|---------------|---------------|
| **Workfront** | Task execution | Granular work items, time tracking | No measurement data |
| **Asana** | Event/partner projects | Flexible structure, collaboration | Limited custom fields |
| **Analytics** | Performance metrics | Rich measurement data | No operational context |
| **DAM** | Asset management | Asset metadata, versions | No campaign attribution |
| **Salesforce** | CRM/Pipeline | Revenue attribution | Delayed sync, sales-centric |

**Key Insight**: No single system can be the system of record for ALL campaign data. Each system is authoritative for its domain, but we need a canonical model that unifies them.

---

## 2. The Solution: Canonical Campaign Spine

### 2.1 Core Philosophy

> The **Campaign Spine** is not a new database of truth. It's a **canonical schema** that defines how entities relate across systems, with clear ownership of each field.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CANONICAL CAMPAIGN SPINE                              │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    MCC CANONICAL MODEL                             │ │
│   │                                                                     │ │
│   │   campaign_id: "mcc-camp-uuid-12345"  (Globally Unique)            │ │
│   │                                                                     │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │                     IDENTITY LAYER                           │ │ │
│   │   │                                                               │ │ │
│   │   │   Canonical ID → maps to → System-specific IDs               │ │ │
│   │   │   ├── workfront_id: "WF-123"                                 │ │ │
│   │   │   ├── asana_id: "12345"                                      │ │ │
│   │   │   ├── analytics_campaign_key: "abc"                          │ │ │
│   │   │   ├── salesforce_campaign_id: "701xx..."                     │ │ │
│   │   │   └── dam_collection_id: "col-789"                           │ │ │
│   │   └─────────────────────────────────────────────────────────────┘ │ │
│   │                                                                     │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │                     FIELD OWNERSHIP                          │ │ │
│   │   │                                                               │ │ │
│   │   │   Each field has ONE authoritative source:                   │ │ │
│   │   │                                                               │ │ │
│   │   │   name              → MCC (user-defined, synced to others)   │ │ │
│   │   │   status            → Execution system (WF or Asana)         │ │ │
│   │   │   tasks[]           → Execution system                       │ │ │
│   │   │   budget            → MCC (strategy input)                   │ │ │
│   │   │   budget_spent      → Analytics/Finance                      │ │ │
│   │   │   impressions       → Analytics                              │ │ │
│   │   │   conversions       → Analytics                              │ │ │
│   │   │   assets[]          → DAM                                    │ │ │
│   │   │   pipeline_impact   → Salesforce                             │ │ │
│   │   └─────────────────────────────────────────────────────────────┘ │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   PRINCIPLE: "One throat to choke" for each data element               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Why Not Just Use Workfront as the System of Record?

| Consideration | Using Workfront as SoR | Using Canonical Spine |
|---------------|------------------------|----------------------|
| **Events team uses Asana** | Forced migration or manual sync | Both systems map to spine |
| **Sister team integration** | They must adopt Workfront | They map their system to spine |
| **Measurement data** | Manual entry into Workfront | Analytics syncs directly |
| **Schema flexibility** | Constrained by Workfront fields | Custom fields in canonical model |
| **Vendor lock-in** | High | Low (spine is platform-agnostic) |

---

## 3. Core Entity Model

### 3.1 Entity Relationship Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CANONICAL ENTITY MODEL                                │
│                                                                          │
│                           ┌──────────────┐                              │
│                           │   TENANT     │                              │
│                           │              │                              │
│                           │ id           │                              │
│                           │ name         │                              │
│                           │ type (org/BU)│                              │
│                           └──────┬───────┘                              │
│                                  │ 1                                    │
│                                  │                                      │
│                                  │ has many                             │
│                                  ▼ *                                    │
│                           ┌──────────────┐                              │
│                           │  CAMPAIGN    │                              │
│                           │              │                              │
│                           │ campaign_id  │◀─────────────────┐          │
│                           │ name         │                   │          │
│                           │ type         │                   │          │
│                           │ tier         │                   │          │
│                           │ status       │                   │          │
│                           │ launch_date  │                   │          │
│                           │ end_date     │                   │          │
│                           │ budget       │                   │          │
│                           └──────┬───────┘                   │          │
│                                  │ 1                         │          │
│                    ┌─────────────┼─────────────┬─────────────┤          │
│                    │             │             │             │          │
│                    │ has many    │ has many    │ has many    │ has many │
│                    ▼ *           ▼ *           ▼ *           ▼ *        │
│            ┌──────────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐ │
│            │   STRATEGY   │ │ WORK_ITEM│ │  ASSET   │ │ MEASUREMENT  │ │
│            │              │ │          │ │          │ │              │ │
│            │ strategy_id  │ │ item_id  │ │ asset_id │ │ measurement_id│ │
│            │ campaign_id  │ │ campaign_│ │ campaign_│ │ campaign_id  │ │
│            │ type         │ │   id     │ │   id     │ │ date         │ │
│            │ content      │ │ type     │ │ type     │ │ channel      │ │
│            │ approved_by  │ │ status   │ │ url      │ │ impressions  │ │
│            │ approved_at  │ │ assignee │ │ metadata │ │ clicks       │ │
│            │ version      │ │ due_date │ │ source   │ │ conversions  │ │
│            └──────────────┘ └──────────┘ └──────────┘ └──────────────┘ │
│                                  │                                      │
│                                  │ 1                                    │
│                                  │                                      │
│                                  │ maps to                              │
│                                  ▼ 1                                    │
│                           ┌──────────────┐                              │
│                           │ SYSTEM_REF   │                              │
│                           │              │                              │
│                           │ ref_id       │                              │
│                           │ entity_type  │                              │
│                           │ entity_id    │                              │
│                           │ system       │                              │
│                           │ external_id  │                              │
│                           │ last_sync    │                              │
│                           └──────────────┘                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Entity Definitions

| Entity | Purpose | Source of Truth | Sync Pattern |
|--------|---------|-----------------|--------------|
| **Tenant** | Organization/BU isolation | MCC | N/A (internal) |
| **Campaign** | Unified campaign record | MCC (identity), multiple (fields) | Bi-directional |
| **Strategy** | Planning artifacts | MCC | One-way (MCC → Execution) |
| **WorkItem** | Tasks, milestones | Workfront/Asana | Bi-directional |
| **Asset** | Creative files | DAM | One-way (DAM → MCC) |
| **Measurement** | Performance metrics | Analytics | One-way (Analytics → MCC) |
| **SystemRef** | Cross-system ID mapping | MCC | On create/sync |

---

## 4. Entity Schemas

### 4.1 Campaign Entity

```typescript
// canonical-model/src/entities/campaign.ts

/**
 * The Campaign entity is the central spine connecting all marketing activities.
 * It aggregates identity, strategy, execution, and measurement data.
 */
interface Campaign {
  // === IDENTITY (Source: MCC) ===
  campaign_id: string;           // UUID, generated by MCC
  tenant_id: string;             // Owning tenant

  // === CORE ATTRIBUTES (Source: MCC, synced to execution systems) ===
  name: string;                  // Campaign name
  description: string | null;    // Rich text description
  type: CampaignType;           // standard | event | partner | product_launch
  tier: 1 | 2 | 3;              // Campaign tier (affects resources, approvals)

  // === TIMING (Source: MCC, validated against execution) ===
  launch_date: ISO8601Date;     // Planned launch
  end_date: ISO8601Date | null; // Planned end (null = evergreen)

  // === STATUS (Source: Execution System - Workfront or Asana) ===
  status: CampaignStatus;       // draft | planning | in_progress | launched | completed | cancelled
  status_updated_at: ISO8601DateTime;
  status_source: 'workfront' | 'asana' | 'mcc';

  // === OWNERSHIP (Source: MCC) ===
  owner_id: string;             // Primary campaign owner (MCC user ID)
  team_id: string;              // Owning team
  stakeholder_ids: string[];    // Additional stakeholders

  // === BUDGET (Source: MCC for planned, Analytics for actuals) ===
  budget_planned: MonetaryAmount | null;
  budget_committed: MonetaryAmount | null;   // From finance/procurement
  budget_spent: MonetaryAmount | null;       // From analytics

  // === CROSS-SYSTEM REFERENCES (Source: MCC mapping layer) ===
  external_refs: {
    workfront_project_id: string | null;
    asana_project_gid: string | null;
    salesforce_campaign_id: string | null;
    analytics_campaign_key: string | null;
    dam_collection_id: string | null;
  };

  // === CATEGORIZATION (Source: MCC) ===
  channels: Channel[];          // email | web | social | events | paid_media
  audiences: string[];          // Target audience segments
  products: string[];           // Associated products
  regions: string[];            // Geographic regions
  tags: string[];               // Freeform tags

  // === METADATA ===
  created_at: ISO8601DateTime;
  created_by: string;
  updated_at: ISO8601DateTime;
  updated_by: string;
  version: number;              // Optimistic locking
}

type CampaignType = 'standard' | 'event' | 'partner' | 'product_launch';

type CampaignStatus =
  | 'draft'           // Initial creation, not yet planned
  | 'planning'        // Strategy being developed
  | 'approved'        // Strategy approved, ready for execution
  | 'in_progress'     // Active work underway
  | 'launched'        // Campaign is live
  | 'completed'       // Campaign finished
  | 'cancelled'       // Campaign cancelled
  | 'on_hold';        // Temporarily paused

type Channel = 'email' | 'web' | 'social' | 'events' | 'paid_media' | 'partner' | 'other';

interface MonetaryAmount {
  amount: number;               // Decimal value
  currency: string;             // ISO 4217 (e.g., 'USD')
}

type ISO8601Date = string;      // "2026-01-15"
type ISO8601DateTime = string;  // "2026-01-15T09:30:00Z"
```

### 4.2 Strategy Entity

```typescript
// canonical-model/src/entities/strategy.ts

/**
 * Strategy captures the planning artifacts generated by agents and approved by humans.
 * Strategies are versioned and immutable after approval.
 */
interface Strategy {
  // === IDENTITY ===
  strategy_id: string;          // UUID
  campaign_id: string;          // Parent campaign

  // === CONTENT ===
  type: StrategyType;           // messaging | audience | channel | creative_brief | full
  title: string;                // Human-readable title
  content: StrategyContent;     // Structured content (see below)

  // === APPROVAL WORKFLOW ===
  status: 'draft' | 'pending_review' | 'approved' | 'rejected' | 'superseded';
  submitted_at: ISO8601DateTime | null;
  submitted_by: string | null;
  approved_at: ISO8601DateTime | null;
  approved_by: string | null;
  rejection_reason: string | null;

  // === VERSIONING ===
  version: number;              // 1, 2, 3...
  previous_version_id: string | null;

  // === GENERATION CONTEXT ===
  generated_by: 'human' | 'agent';
  agent_session_id: string | null;  // If agent-generated, link to conversation
  prompt_used: string | null;       // The prompt that generated this

  // === METADATA ===
  created_at: ISO8601DateTime;
  updated_at: ISO8601DateTime;
}

type StrategyType =
  | 'messaging'       // Key messages, value props
  | 'audience'        // Target audience definition
  | 'channel'         // Channel mix strategy
  | 'creative_brief'  // Brief for production team
  | 'full';           // Comprehensive strategy doc

interface StrategyContent {
  // Flexible structure based on type
  sections: StrategySection[];
  attachments: AttachmentRef[];
}

interface StrategySection {
  heading: string;
  body: string;         // Markdown-formatted
  subsections?: StrategySection[];
}
```

### 4.3 WorkItem Entity

```typescript
// canonical-model/src/entities/work-item.ts

/**
 * WorkItem is a unified view of tasks/issues across execution systems.
 * The canonical model normalizes Workfront tasks, Asana tasks, and other work items.
 */
interface WorkItem {
  // === IDENTITY ===
  item_id: string;              // UUID (MCC-generated)
  campaign_id: string;          // Parent campaign
  parent_item_id: string | null; // For subtasks

  // === CORE ATTRIBUTES ===
  name: string;
  description: string | null;
  item_type: WorkItemType;
  priority: 'low' | 'medium' | 'high' | 'urgent';

  // === STATUS (Source: Execution System) ===
  status: WorkItemStatus;
  status_updated_at: ISO8601DateTime;
  percent_complete: number;     // 0-100

  // === TIMING ===
  planned_start_date: ISO8601Date | null;
  planned_end_date: ISO8601Date | null;
  actual_start_date: ISO8601Date | null;
  actual_end_date: ISO8601Date | null;
  duration_hours: number | null;  // Estimated effort

  // === ASSIGNMENT ===
  assignee_id: string | null;     // MCC user ID
  assignee_email: string | null;  // Denormalized for display
  team_id: string | null;

  // === EXECUTION SYSTEM REFERENCE ===
  source_system: 'workfront' | 'asana' | 'mcc';
  external_id: string | null;     // e.g., Workfront task ID
  external_url: string | null;    // Deep link to source system

  // === SYNC STATE ===
  last_synced_at: ISO8601DateTime;
  sync_status: 'synced' | 'pending' | 'conflict' | 'error';
  sync_error: string | null;

  // === METADATA ===
  created_at: ISO8601DateTime;
  updated_at: ISO8601DateTime;
}

type WorkItemType =
  | 'task'            // Standard work item
  | 'milestone'       // Key delivery point
  | 'approval'        // Requires sign-off
  | 'review'          // Content review task
  | 'deliverable';    // Output artifact

type WorkItemStatus =
  | 'not_started'
  | 'in_progress'
  | 'blocked'
  | 'pending_review'
  | 'approved'
  | 'completed'
  | 'cancelled';
```

### 4.4 Measurement Entity

```typescript
// canonical-model/src/entities/measurement.ts

/**
 * Measurement captures performance metrics from analytics systems.
 * Data is append-only and aggregated by date/channel for trend analysis.
 */
interface Measurement {
  // === IDENTITY ===
  measurement_id: string;       // UUID
  campaign_id: string;          // Parent campaign

  // === DIMENSIONS ===
  date: ISO8601Date;            // Measurement date
  channel: Channel;             // email | web | social | paid_media | etc.
  segment: string | null;       // Optional audience segment

  // === ENGAGEMENT METRICS ===
  impressions: number;
  clicks: number;
  unique_visitors: number | null;
  sessions: number | null;
  page_views: number | null;
  time_on_site_seconds: number | null;
  bounce_rate: number | null;   // 0-1

  // === CONVERSION METRICS ===
  conversions: number;
  conversion_value: MonetaryAmount | null;
  leads: number | null;
  mqls: number | null;          // Marketing Qualified Leads
  sqls: number | null;          // Sales Qualified Leads
  opportunities: number | null;
  pipeline_generated: MonetaryAmount | null;
  revenue_attributed: MonetaryAmount | null;

  // === COST METRICS ===
  spend: MonetaryAmount | null;
  cpc: MonetaryAmount | null;   // Cost per click
  cpm: MonetaryAmount | null;   // Cost per mille
  cpa: MonetaryAmount | null;   // Cost per acquisition

  // === CALCULATED METRICS ===
  ctr: number | null;           // Click-through rate (0-1)
  cvr: number | null;           // Conversion rate (0-1)
  roas: number | null;          // Return on ad spend

  // === SOURCE TRACKING ===
  source_system: 'adobe_analytics' | 'google_analytics' | 'salesforce' | 'custom';
  source_campaign_key: string;  // ID in source system
  data_freshness: 'realtime' | 'hourly' | 'daily' | 'weekly';

  // === METADATA ===
  collected_at: ISO8601DateTime;
  created_at: ISO8601DateTime;
}
```

---

## 5. Field Authority Matrix

### 5.1 Who Owns What?

This matrix defines the authoritative source for each field. This is critical for conflict resolution.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FIELD AUTHORITY MATRIX                                │
│                                                                          │
│   Field                    │ Authority      │ Sync Direction │ Conflicts│
│   ─────────────────────────┼────────────────┼────────────────┼──────────│
│   campaign_id              │ MCC            │ MCC → All      │ N/A      │
│   name                     │ MCC            │ MCC → WF/Asana │ MCC wins │
│   description              │ MCC            │ MCC → WF/Asana │ MCC wins │
│   type                     │ MCC            │ MCC → WF/Asana │ MCC wins │
│   tier                     │ MCC            │ MCC → WF/Asana │ MCC wins │
│   ─────────────────────────┼────────────────┼────────────────┼──────────│
│   status                   │ Exec System    │ WF/Asana → MCC │ Exec wins│
│   tasks                    │ Exec System    │ WF/Asana → MCC │ Exec wins│
│   assignees                │ Exec System    │ WF/Asana → MCC │ Exec wins│
│   due_dates                │ Exec System    │ Bi-directional │ LWW*     │
│   ─────────────────────────┼────────────────┼────────────────┼──────────│
│   budget_planned           │ MCC            │ MCC → WF       │ MCC wins │
│   budget_spent             │ Analytics      │ Analytics → MCC│ N/A      │
│   impressions              │ Analytics      │ Analytics → MCC│ N/A      │
│   conversions              │ Analytics      │ Analytics → MCC│ N/A      │
│   pipeline_generated       │ Salesforce     │ SF → MCC       │ N/A      │
│   ─────────────────────────┼────────────────┼────────────────┼──────────│
│   assets                   │ DAM            │ DAM → MCC      │ N/A      │
│   asset_metadata           │ DAM            │ DAM → MCC      │ N/A      │
│   ─────────────────────────┼────────────────┼────────────────┼──────────│
│   strategy_content         │ MCC            │ MCC only       │ N/A      │
│   approval_status          │ MCC            │ MCC only       │ N/A      │
│                                                                          │
│   * LWW = Last Writer Wins (with timestamp comparison)                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Conflict Resolution Rules

```typescript
// sync-engine/src/conflict-resolution.ts

/**
 * Conflict resolution strategy based on field authority.
 */
enum ConflictResolution {
  AUTHORITY_WINS = 'authority_wins',  // Field owner always wins
  LAST_WRITE_WINS = 'last_write_wins', // Most recent update wins
  MANUAL_RESOLUTION = 'manual',        // Flag for human review
  MERGE = 'merge',                     // Attempt to merge (e.g., arrays)
}

const FIELD_RESOLUTION_RULES: Record<string, ConflictResolution> = {
  // MCC-owned fields: MCC always wins
  'campaign.name': ConflictResolution.AUTHORITY_WINS,
  'campaign.description': ConflictResolution.AUTHORITY_WINS,
  'campaign.type': ConflictResolution.AUTHORITY_WINS,
  'campaign.tier': ConflictResolution.AUTHORITY_WINS,
  'campaign.budget_planned': ConflictResolution.AUTHORITY_WINS,

  // Execution system-owned: Execution system wins
  'campaign.status': ConflictResolution.AUTHORITY_WINS,
  'work_item.status': ConflictResolution.AUTHORITY_WINS,
  'work_item.assignee_id': ConflictResolution.AUTHORITY_WINS,

  // Timing fields: Last write wins (both systems can update)
  'campaign.launch_date': ConflictResolution.LAST_WRITE_WINS,
  'work_item.due_date': ConflictResolution.LAST_WRITE_WINS,

  // Structural changes: Manual resolution
  'campaign.external_refs.workfront_project_id': ConflictResolution.MANUAL_RESOLUTION,
  'work_item.parent_item_id': ConflictResolution.MANUAL_RESOLUTION,

  // Arrays: Merge (union)
  'campaign.tags': ConflictResolution.MERGE,
  'campaign.stakeholder_ids': ConflictResolution.MERGE,
};

function resolveConflict(
  field: string,
  mccValue: unknown,
  externalValue: unknown,
  mccTimestamp: Date,
  externalTimestamp: Date,
  fieldAuthority: 'mcc' | 'external'
): { resolvedValue: unknown; resolution: string } {
  const strategy = FIELD_RESOLUTION_RULES[field] || ConflictResolution.AUTHORITY_WINS;

  switch (strategy) {
    case ConflictResolution.AUTHORITY_WINS:
      return {
        resolvedValue: fieldAuthority === 'mcc' ? mccValue : externalValue,
        resolution: `Authority (${fieldAuthority}) wins`,
      };

    case ConflictResolution.LAST_WRITE_WINS:
      const winner = mccTimestamp > externalTimestamp ? 'mcc' : 'external';
      return {
        resolvedValue: winner === 'mcc' ? mccValue : externalValue,
        resolution: `Last write (${winner} at ${winner === 'mcc' ? mccTimestamp : externalTimestamp}) wins`,
      };

    case ConflictResolution.MERGE:
      if (Array.isArray(mccValue) && Array.isArray(externalValue)) {
        return {
          resolvedValue: [...new Set([...mccValue, ...externalValue])],
          resolution: 'Merged arrays',
        };
      }
      // Fallback to authority wins
      return {
        resolvedValue: fieldAuthority === 'mcc' ? mccValue : externalValue,
        resolution: 'Could not merge, authority wins',
      };

    case ConflictResolution.MANUAL_RESOLUTION:
      throw new ConflictRequiresManualResolution(field, mccValue, externalValue);
  }
}
```

---

## 6. Cross-System ID Mapping

### 6.1 The ID Mapping Challenge

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ID MAPPING ARCHITECTURE                               │
│                                                                          │
│   PROBLEM: Each system generates its own IDs                            │
│                                                                          │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐              │
│   │ Workfront   │     │   Asana     │     │  Analytics  │              │
│   │ WF-123      │     │ 12345678    │     │ camp_abc    │              │
│   └─────────────┘     └─────────────┘     └─────────────┘              │
│                                                                          │
│   SOLUTION: MCC maintains the canonical mapping                          │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    SYSTEM_REF TABLE                                │ │
│   │                                                                     │ │
│   │   ref_id    │ entity_type │ canonical_id │ system    │ external_id │ │
│   │   ──────────┼─────────────┼──────────────┼───────────┼─────────────│ │
│   │   ref-001   │ campaign    │ mcc-camp-123 │ workfront │ WF-123      │ │
│   │   ref-002   │ campaign    │ mcc-camp-123 │ asana     │ 12345678    │ │
│   │   ref-003   │ campaign    │ mcc-camp-123 │ analytics │ camp_abc    │ │
│   │   ref-004   │ campaign    │ mcc-camp-123 │ salesforce│ 701xx...    │ │
│   │   ref-005   │ work_item   │ mcc-item-456 │ workfront │ WF-T-789    │ │
│   │   ref-006   │ work_item   │ mcc-item-456 │ asana     │ 98765432    │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   QUERY PATTERNS:                                                       │
│                                                                          │
│   • "Get MCC ID for Workfront WF-123":                                  │
│     SELECT canonical_id FROM system_ref                                 │
│     WHERE system = 'workfront' AND external_id = 'WF-123'               │
│                                                                          │
│   • "Get all external IDs for campaign mcc-camp-123":                   │
│     SELECT system, external_id FROM system_ref                          │
│     WHERE entity_type = 'campaign' AND canonical_id = 'mcc-camp-123'    │
│                                                                          │
│   • "Resolve webhook from Asana task 98765432":                         │
│     SELECT canonical_id FROM system_ref                                 │
│     WHERE system = 'asana' AND external_id = '98765432'                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 DynamoDB Schema for ID Mapping

```typescript
// infrastructure/dynamodb/system-ref-table.ts

/**
 * DynamoDB table design for cross-system ID mapping.
 * Optimized for two primary access patterns:
 * 1. Lookup canonical ID from external system ID
 * 2. Lookup all external IDs for a canonical entity
 */

// Table: system-refs
// PK: SYSTEM#{system}#ENTITY#{entity_type}
// SK: EXTERNAL#{external_id}
// GSI1PK: CANONICAL#{entity_type}#{canonical_id}
// GSI1SK: SYSTEM#{system}

interface SystemRefRecord {
  pk: string;                // "SYSTEM#workfront#ENTITY#campaign"
  sk: string;                // "EXTERNAL#WF-123"
  gsi1pk: string;            // "CANONICAL#campaign#mcc-camp-uuid"
  gsi1sk: string;            // "SYSTEM#workfront"

  ref_id: string;            // Unique reference ID
  entity_type: string;       // campaign | work_item | asset
  canonical_id: string;      // MCC entity ID
  system: string;            // workfront | asana | analytics | etc.
  external_id: string;       // ID in external system
  external_url: string;      // Deep link to external system

  created_at: string;        // ISO 8601
  last_synced_at: string;    // ISO 8601
  sync_version: number;      // Incremented on each sync
}

// Access pattern 1: External ID → Canonical ID
async function getCanonicalId(
  system: string,
  entityType: string,
  externalId: string
): Promise<string | null> {
  const result = await dynamodb.get({
    TableName: 'system-refs',
    Key: {
      pk: `SYSTEM#${system}#ENTITY#${entityType}`,
      sk: `EXTERNAL#${externalId}`,
    },
  });
  return result.Item?.canonical_id ?? null;
}

// Access pattern 2: Canonical ID → All External IDs
async function getExternalIds(
  entityType: string,
  canonicalId: string
): Promise<Array<{ system: string; external_id: string; external_url: string }>> {
  const result = await dynamodb.query({
    TableName: 'system-refs',
    IndexName: 'gsi1',
    KeyConditionExpression: 'gsi1pk = :pk',
    ExpressionAttributeValues: {
      ':pk': `CANONICAL#${entityType}#${canonicalId}`,
    },
  });
  return result.Items.map(item => ({
    system: item.system,
    external_id: item.external_id,
    external_url: item.external_url,
  }));
}
```

---

## 7. Data Flow Architecture

### 7.1 Sync Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DATA SYNC ARCHITECTURE                                │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    OUTBOUND SYNC (MCC → External)                  │ │
│   │                                                                     │ │
│   │   1. User/Agent creates campaign in MCC                            │ │
│   │   2. MCC generates canonical campaign_id                           │ │
│   │   3. Campaign saved to DynamoDB                                    │ │
│   │   4. EventBridge event: "campaign.created"                         │ │
│   │   5. Step Functions workflow:                                      │ │
│   │      a. Determine target system (Workfront or Asana)               │ │
│   │      b. Create project in target system                            │ │
│   │      c. Store ID mapping in system-refs table                      │ │
│   │      d. Create tasks from strategy template                        │ │
│   │      e. Publish "campaign.synced" event                            │ │
│   │                                                                     │ │
│   │   ┌───────────┐    ┌───────────┐    ┌───────────┐                 │ │
│   │   │    MCC    │───▶│EventBridge│───▶│Step Funcs │                 │ │
│   │   │ DynamoDB  │    │           │    │           │                 │ │
│   │   └───────────┘    └───────────┘    └─────┬─────┘                 │ │
│   │                                           │                        │ │
│   │                              ┌────────────┴────────────┐           │ │
│   │                              ▼                         ▼           │ │
│   │                        ┌───────────┐            ┌───────────┐     │ │
│   │                        │ Workfront │            │   Asana   │     │ │
│   │                        │  Lambda   │            │  Lambda   │     │ │
│   │                        └───────────┘            └───────────┘     │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    INBOUND SYNC (External → MCC)                   │ │
│   │                                                                     │ │
│   │   1. User updates task in Workfront/Asana                          │ │
│   │   2. Webhook fires to API Gateway                                  │ │
│   │   3. Lambda validates webhook signature                            │ │
│   │   4. Lookup canonical ID from external ID                          │ │
│   │   5. Transform external schema to canonical schema                 │ │
│   │   6. Apply field authority rules                                   │ │
│   │   7. Update MCC entity (if authority allows)                       │ │
│   │   8. Publish "work_item.updated" event                             │ │
│   │                                                                     │ │
│   │   ┌───────────┐    ┌───────────┐    ┌───────────┐    ┌─────────┐ │ │
│   │   │ Workfront │───▶│    API    │───▶│  Lambda   │───▶│   MCC   │ │ │
│   │   │  Webhook  │    │  Gateway  │    │ (Process) │    │DynamoDB │ │ │
│   │   └───────────┘    └───────────┘    └───────────┘    └─────────┘ │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    MEASUREMENT SYNC (Analytics → MCC)              │ │
│   │                                                                     │ │
│   │   1. EventBridge scheduled rule (daily at 6 AM)                    │ │
│   │   2. Lambda queries Analytics API for previous day                 │ │
│   │   3. Transform to canonical Measurement schema                     │ │
│   │   4. Match campaign via analytics_campaign_key                     │ │
│   │   5. Append to Measurement table (time-series)                     │ │
│   │   6. Aggregate to Campaign.budget_spent                            │ │
│   │   7. Publish "measurement.collected" event                         │ │
│   │                                                                     │ │
│   │   ┌───────────┐    ┌───────────┐    ┌───────────┐    ┌─────────┐ │ │
│   │   │SchedulRule│───▶│  Lambda   │───▶│ Analytics │    │   MCC   │ │ │
│   │   │ (Daily)   │    │(Collector)│◀───│   API     │───▶│DynamoDB │ │ │
│   │   └───────────┘    └───────────┘    └───────────┘    └─────────┘ │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Event Schema

```typescript
// events/schemas/campaign-events.ts

/**
 * EventBridge event schemas for campaign lifecycle.
 */

interface CampaignCreatedEvent {
  source: 'mcc.campaigns';
  'detail-type': 'campaign.created';
  detail: {
    campaign_id: string;
    tenant_id: string;
    name: string;
    type: CampaignType;
    tier: number;
    created_by: string;
    timestamp: string;
  };
}

interface CampaignSyncedEvent {
  source: 'mcc.sync';
  'detail-type': 'campaign.synced';
  detail: {
    campaign_id: string;
    target_system: 'workfront' | 'asana';
    external_id: string;
    sync_type: 'create' | 'update';
    timestamp: string;
  };
}

interface WorkItemUpdatedEvent {
  source: 'mcc.work_items';
  'detail-type': 'work_item.updated';
  detail: {
    item_id: string;
    campaign_id: string;
    source_system: 'workfront' | 'asana' | 'mcc';
    changed_fields: string[];
    previous_values: Record<string, unknown>;
    new_values: Record<string, unknown>;
    timestamp: string;
  };
}

interface MeasurementCollectedEvent {
  source: 'mcc.measurements';
  'detail-type': 'measurement.collected';
  detail: {
    campaign_id: string;
    date: string;
    channel: string;
    source_system: string;
    impressions: number;
    conversions: number;
    timestamp: string;
  };
}
```

---

## 8. Measurement Mapping

### 8.1 The Measurement Challenge

Different analytics systems use different identifiers for the same campaign:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MEASUREMENT ID MAPPING                                │
│                                                                          │
│   PROBLEM:                                                              │
│                                                                          │
│   Adobe Analytics:    utm_campaign=q1_launch_2026                       │
│   Google Analytics:   campaign_id=CMP_12345                             │
│   Salesforce:         Campaign_ID__c=701xx...                           │
│   Internal Metrics:   tracking_code=MKT-Q1-LAUNCH                       │
│                                                                          │
│   How do we know these all refer to MCC campaign mcc-camp-uuid-123?     │
│                                                                          │
│   SOLUTION: Campaign Registration with Tracking Keys                    │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    CAMPAIGN_TRACKING_KEYS TABLE                    │ │
│   │                                                                     │ │
│   │   campaign_id     │ tracking_system   │ tracking_key               │ │
│   │   ────────────────┼───────────────────┼────────────────────────────│ │
│   │   mcc-camp-123    │ adobe_analytics   │ q1_launch_2026             │ │
│   │   mcc-camp-123    │ google_analytics  │ CMP_12345                  │ │
│   │   mcc-camp-123    │ salesforce        │ 701xx...                   │ │
│   │   mcc-camp-123    │ internal_metrics  │ MKT-Q1-LAUNCH              │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   WORKFLOW:                                                             │
│                                                                          │
│   1. When campaign created in MCC, generate tracking keys               │
│   2. Tracking keys follow naming convention per system                  │
│   3. Keys are included in campaign brief for production team            │
│   4. Production team implements keys in UTM params, tracking pixels     │
│   5. Measurement collector uses keys to attribute data back to campaign │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Tracking Key Generation

```typescript
// campaigns/src/tracking-keys.ts

/**
 * Generate tracking keys for a new campaign.
 * Keys follow system-specific conventions for compatibility.
 */
function generateTrackingKeys(campaign: Campaign): CampaignTrackingKeys {
  const baseName = slugify(campaign.name, { lower: true, strict: true });
  const year = new Date(campaign.launch_date).getFullYear();
  const quarter = getQuarter(campaign.launch_date);

  return {
    campaign_id: campaign.campaign_id,

    // Adobe Analytics: utm_campaign format
    adobe_analytics: `${baseName}_${year}`,

    // Google Analytics: CMP_ prefix
    google_analytics: `CMP_${campaign.campaign_id.slice(-8).toUpperCase()}`,

    // Salesforce: Will be populated when SF campaign created
    salesforce: null, // Set during Salesforce sync

    // Internal: Standard marketing code format
    internal_metrics: `MKT-${quarter}${year}-${baseName.slice(0, 10).toUpperCase()}`,

    // UTM parameters template
    utm_template: {
      utm_source: '{source}', // To be filled per channel
      utm_medium: '{medium}', // To be filled per channel
      utm_campaign: `${baseName}_${year}`,
      utm_content: '{content}', // To be filled per asset
      utm_term: '{term}', // Optional, for paid search
    },
  };
}

function getQuarter(date: string): string {
  const month = new Date(date).getMonth();
  return `Q${Math.floor(month / 3) + 1}`;
}
```

---

## 9. Schema Evolution Strategy

### 9.1 Principles

> **Schemas will change.** The canonical model must support evolution without breaking existing integrations.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SCHEMA EVOLUTION PRINCIPLES                           │
│                                                                          │
│   1. ADDITIVE CHANGES ARE SAFE                                          │
│      ✓ Add new optional fields                                          │
│      ✓ Add new entity types                                             │
│      ✓ Add new enum values                                              │
│                                                                          │
│   2. BREAKING CHANGES REQUIRE MIGRATION                                 │
│      ⚠ Renaming fields                                                  │
│      ⚠ Changing field types                                             │
│      ⚠ Removing fields                                                  │
│      ⚠ Making optional fields required                                  │
│                                                                          │
│   3. VERSION THE SCHEMA                                                 │
│      • Schema version stored with each entity                           │
│      • Transformers upgrade old records on read                         │
│      • Batch migration for large changes                                │
│                                                                          │
│   4. MAINTAIN BACKWARDS COMPATIBILITY FOR 2 VERSIONS                    │
│      • Version N and N-1 must both work                                 │
│      • Deprecation warnings before removal                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Versioned Schema Example

```typescript
// canonical-model/src/versioning.ts

interface VersionedEntity {
  _schema_version: number;
  // ... rest of entity fields
}

// Schema version history
const SCHEMA_VERSIONS = {
  campaign: {
    current: 3,
    migrations: {
      1: (entity: any) => ({
        ...entity,
        // v1 → v2: Added 'tier' field with default
        tier: entity.tier ?? 2,
        _schema_version: 2,
      }),
      2: (entity: any) => ({
        ...entity,
        // v2 → v3: Renamed 'owner' to 'owner_id'
        owner_id: entity.owner_id ?? entity.owner,
        _schema_version: 3,
      }),
    },
  },
};

function migrateToCurrentVersion<T extends VersionedEntity>(
  entity: T,
  entityType: keyof typeof SCHEMA_VERSIONS
): T {
  const config = SCHEMA_VERSIONS[entityType];
  let current = entity;

  while (current._schema_version < config.current) {
    const migration = config.migrations[current._schema_version];
    if (!migration) {
      throw new Error(`No migration path from version ${current._schema_version}`);
    }
    current = migration(current);
  }

  return current;
}
```

---

## 10. Implementation Guidelines

### 10.1 Phased Rollout

| Phase | Scope | Duration | Dependencies |
|-------|-------|----------|--------------|
| **Phase 1** | Campaign + WorkItem entities | 4 weeks | Workfront integration |
| **Phase 2** | Strategy entity + approval flow | 3 weeks | Agent platform |
| **Phase 3** | Measurement entity + analytics sync | 3 weeks | Analytics API access |
| **Phase 4** | Asset entity + DAM integration | 4 weeks | DAM API access |
| **Phase 5** | Sister team onboarding (Asana) | 2 weeks | Multi-tenancy |

### 10.2 Validation Rules

```typescript
// canonical-model/src/validation.ts

import { z } from 'zod';

const CampaignSchema = z.object({
  campaign_id: z.string().uuid(),
  tenant_id: z.string().uuid(),
  name: z.string().min(3).max(200),
  description: z.string().max(5000).nullable(),
  type: z.enum(['standard', 'event', 'partner', 'product_launch']),
  tier: z.number().int().min(1).max(3),
  status: z.enum([
    'draft', 'planning', 'approved', 'in_progress',
    'launched', 'completed', 'cancelled', 'on_hold'
  ]),
  launch_date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  end_date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).nullable(),
  owner_id: z.string().uuid(),
  team_id: z.string().uuid(),
  budget_planned: z.object({
    amount: z.number().positive(),
    currency: z.string().length(3),
  }).nullable(),
  channels: z.array(z.enum([
    'email', 'web', 'social', 'events', 'paid_media', 'partner', 'other'
  ])),
  external_refs: z.object({
    workfront_project_id: z.string().nullable(),
    asana_project_gid: z.string().nullable(),
    salesforce_campaign_id: z.string().nullable(),
    analytics_campaign_key: z.string().nullable(),
    dam_collection_id: z.string().nullable(),
  }),
  created_at: z.string().datetime(),
  updated_at: z.string().datetime(),
  version: z.number().int().positive(),
});

function validateCampaign(data: unknown): Campaign {
  return CampaignSchema.parse(data);
}
```

### 10.3 Testing Strategy

```typescript
// canonical-model/tests/sync-integration.test.ts

describe('Campaign Sync Integration', () => {
  describe('Outbound Sync (MCC → Workfront)', () => {
    it('should create Workfront project when campaign created', async () => {
      const campaign = await createCampaign({
        name: 'Q1 Product Launch',
        type: 'standard',
        tier: 1,
      });

      // Wait for async sync
      await waitForEvent('campaign.synced', { campaign_id: campaign.campaign_id });

      // Verify Workfront project exists
      const refs = await getExternalIds('campaign', campaign.campaign_id);
      const wfRef = refs.find(r => r.system === 'workfront');
      expect(wfRef).toBeDefined();

      // Verify project in Workfront
      const wfProject = await workfrontClient.getProject(wfRef.external_id);
      expect(wfProject.name).toBe('Q1 Product Launch');
    });
  });

  describe('Inbound Sync (Workfront → MCC)', () => {
    it('should update MCC when Workfront task status changes', async () => {
      // Setup: Create campaign with synced work item
      const campaign = await createSyncedCampaign();
      const workItem = campaign.work_items[0];

      // Simulate Workfront webhook
      await simulateWebhook('workfront', 'task.updated', {
        taskID: workItem.external_refs.workfront_task_id,
        status: 'CPL', // Workfront's "Complete" status
      });

      // Verify MCC work item updated
      const updatedItem = await getWorkItem(workItem.item_id);
      expect(updatedItem.status).toBe('completed');
    });

    it('should respect field authority on conflict', async () => {
      const campaign = await createSyncedCampaign();

      // Update name in MCC (MCC is authority)
      await updateCampaign(campaign.campaign_id, { name: 'Updated Name' });

      // Simulate conflicting update from Workfront
      await simulateWebhook('workfront', 'project.updated', {
        projectID: campaign.external_refs.workfront_project_id,
        name: 'Workfront Name', // Should be ignored
      });

      // Verify MCC name preserved
      const updated = await getCampaign(campaign.campaign_id);
      expect(updated.name).toBe('Updated Name');
    });
  });
});
```

---

## Summary

### Key Architectural Decisions

1. **MCC generates canonical IDs** - Not delegated to any external system
2. **Field authority is explicit** - Each field has one owner, documented
3. **Sync is event-driven** - No polling, webhooks + EventBridge
4. **Schema is versioned** - Migrations handle evolution
5. **ID mapping is bidirectional** - Fast lookups both directions

### Next Steps

1. Implement [Contract-First Agent Design](07-contract-first-agents.md) to ensure agents operate within canonical model constraints
2. Build [Policy Engine](08-policy-engine.md) to enforce field authority during agent actions
3. Deploy [Multi-Tenant Hub](09-multi-tenancy-hub.md) to support sister team integration
