# Contract-First Agent Design

## Building Safe, Predictable AI Agents for Enterprise Operations

### Version: 2.0
### Date: January 10, 2026
### Author: Principal Engineer / Solution Architect

---

## Table of Contents

1. [The Problem: Unconstrained AI](#1-the-problem-unconstrained-ai)
2. [Contract-First vs Prompt-First](#2-contract-first-vs-prompt-first)
3. [Agent Architecture Principles](#3-agent-architecture-principles)
4. [Contract Definition Language](#4-contract-definition-language)
5. [Tool Gateway Design](#5-tool-gateway-design)
6. [Policy Decision Point (PDP)](#6-policy-decision-point-pdp)
7. [Agent Playbooks](#7-agent-playbooks)
8. [Human-in-the-Loop Patterns](#8-human-in-the-loop-patterns)
9. [Agent Observability](#9-agent-observability)
10. [Agent Studio: User-Created Agents](#10-agent-studio-user-created-agents)

---

## 1. The Problem: Unconstrained AI

### 1.1 The Risk of "Prompt-First" Agents

When AI agents can take arbitrary actions based on natural language interpretation:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PROMPT-FIRST FAILURE MODES                            │
│                                                                          │
│   USER: "Create a campaign for our Q1 product launch"                   │
│                                                                          │
│   AGENT (Unconstrained):                                                │
│   "I'll help with that! Let me..."                                      │
│                                                                          │
│   ❌ FAILURE MODE 1: Hallucinated Actions                               │
│      Creates project in wrong Workfront instance                        │
│      Uses non-existent template                                         │
│      Assigns to users who don't exist                                   │
│                                                                          │
│   ❌ FAILURE MODE 2: Scope Creep                                        │
│      Automatically approves budget without human review                 │
│      Publishes incomplete strategy as "approved"                        │
│      Sends notifications to entire organization                         │
│                                                                          │
│   ❌ FAILURE MODE 3: Data Leakage                                       │
│      Includes confidential product details in campaign name             │
│      Shares internal metrics with wrong stakeholders                    │
│      Logs PII in conversation history                                   │
│                                                                          │
│   ❌ FAILURE MODE 4: Inconsistent Behavior                              │
│      Same request produces different structures each time               │
│      Field values don't match taxonomy requirements                     │
│      Breaks downstream system integrations                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Enterprise Requirements for AI Agents

| Requirement | Description | Risk if Missing |
|-------------|-------------|-----------------|
| **Predictability** | Same input produces structurally consistent output | Integration failures, audit issues |
| **Auditability** | Every action traceable to user, policy, and decision | Compliance violations |
| **Guardrails** | Actions bounded by explicit rules | Data breaches, unauthorized actions |
| **Reversibility** | Ability to undo agent actions | Operational disruption |
| **Transparency** | Users understand what agent will/won't do | Trust erosion, support burden |

### 1.3 The Core Insight

> **The LLM is brilliant at understanding intent and generating content. It should NOT be trusted to decide what actions are permissible.** That's the job of explicit contracts and policy engines.

---

## 2. Contract-First vs Prompt-First

### 2.1 Paradigm Comparison

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PARADIGM COMPARISON                                   │
│                                                                          │
│   PROMPT-FIRST (Dangerous at Scale)                                     │
│   ═══════════════════════════════════                                   │
│                                                                          │
│   ┌─────────────────┐                                                   │
│   │   User Prompt   │                                                   │
│   │ "Create a       │                                                   │
│   │  campaign..."   │                                                   │
│   └────────┬────────┘                                                   │
│            │                                                            │
│            ▼                                                            │
│   ┌─────────────────┐                                                   │
│   │      LLM        │──────────────────────▶ ANY ACTION                 │
│   │  (Decides what  │                        (Unbounded)                │
│   │   to do)        │                                                   │
│   └─────────────────┘                                                   │
│                                                                          │
│   Problems:                                                             │
│   • LLM decides which tools to call                                     │
│   • LLM decides parameter values                                        │
│   • LLM decides if action is appropriate                                │
│   • Behavior varies with model, prompt, context                         │
│                                                                          │
│   ─────────────────────────────────────────────────────────────────────│
│                                                                          │
│   CONTRACT-FIRST (Enterprise-Ready)                                     │
│   ═══════════════════════════════════                                   │
│                                                                          │
│   ┌─────────────────┐                                                   │
│   │   User Prompt   │                                                   │
│   │ "Create a       │                                                   │
│   │  campaign..."   │                                                   │
│   └────────┬────────┘                                                   │
│            │                                                            │
│            ▼                                                            │
│   ┌─────────────────┐      ┌─────────────────┐                         │
│   │      LLM        │─────▶│  Plan/Action    │                         │
│   │  (Generates     │      │  (Structured)   │                         │
│   │   structured    │      │                 │                         │
│   │   output)       │      │  {              │                         │
│   └─────────────────┘      │   "action":     │                         │
│                            │    "create_     │                         │
│                            │     campaign",  │                         │
│                            │   "params": {}  │                         │
│                            │  }              │                         │
│                            └────────┬────────┘                         │
│                                     │                                   │
│                                     ▼                                   │
│   ┌─────────────────┐      ┌─────────────────┐      ┌───────────────┐ │
│   │    CONTRACT     │─────▶│  POLICY ENGINE  │─────▶│   ALLOWED     │ │
│   │    VALIDATOR    │      │     (PDP)       │      │   ACTIONS     │ │
│   │                 │      │                 │      │   (Bounded)   │ │
│   │  Schema valid?  │      │  Policy allows? │      │               │ │
│   │  Required fields│      │  User permitted?│      │               │ │
│   │  Value ranges?  │      │  Budget ok?     │      │               │ │
│   └─────────────────┘      └─────────────────┘      └───────────────┘ │
│                                     │                                   │
│                                     ▼                                   │
│                            ┌─────────────────┐                         │
│                            │   TOOL GATEWAY  │                         │
│                            │     (PEP)       │                         │
│                            │                 │                         │
│                            │  Execute action │                         │
│                            │  with guardrails│                         │
│                            └─────────────────┘                         │
│                                                                          │
│   Benefits:                                                             │
│   • LLM generates structured data, not decisions                        │
│   • Contracts define what's valid                                       │
│   • Policies define what's permitted                                    │
│   • Gateway enforces execution boundaries                               │
│   • Behavior is deterministic and auditable                             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 The Three Layers of Control

| Layer | Responsibility | Implemented By |
|-------|----------------|----------------|
| **Contract Layer** | "Is this a valid action structure?" | JSON Schema, Zod, type validation |
| **Policy Layer** | "Is this action permitted for this user/context?" | PDP (Policy Decision Point) |
| **Execution Layer** | "How do we safely execute this action?" | PEP (Policy Enforcement Point) / Tool Gateway |

---

## 3. Agent Architecture Principles

### 3.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONTRACT-FIRST AGENT ARCHITECTURE                     │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                         USER INTERFACE                             │ │
│   │                                                                     │ │
│   │   User: "Create a tier 1 campaign for our Q1 product launch"       │ │
│   │         "targeting enterprise customers in North America"          │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                    │                                     │
│                                    ▼                                     │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                      ORCHESTRATION LAYER                           │ │
│   │                                                                     │ │
│   │   ┌─────────────────┐      ┌─────────────────┐                    │ │
│   │   │   Supervisor    │      │   Conversation  │                    │ │
│   │   │     Agent       │◀────▶│     Memory      │                    │ │
│   │   │                 │      │                 │                    │ │
│   │   │ Routes to       │      │ Context window  │                    │ │
│   │   │ specialist      │      │ + long-term     │                    │ │
│   │   │ agents          │      │ storage         │                    │ │
│   │   └────────┬────────┘      └─────────────────┘                    │ │
│   │            │                                                       │ │
│   │            ▼                                                       │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │               SPECIALIST AGENT (Campaign Agent)              │ │ │
│   │   │                                                               │ │ │
│   │   │   ┌─────────────────────────────────────────────────────┐   │ │ │
│   │   │   │                 LLM (Claude Sonnet)                  │   │ │ │
│   │   │   │                                                       │   │ │ │
│   │   │   │  System Prompt:                                      │   │ │ │
│   │   │   │  "You are a campaign strategy agent. Your job is to  │   │ │ │
│   │   │   │   generate structured campaign plans that conform to │   │ │ │
│   │   │   │   the CampaignPlan schema. You MUST NOT suggest      │   │ │ │
│   │   │   │   actions outside your allowed tool set."            │   │ │ │
│   │   │   │                                                       │   │ │ │
│   │   │   │  Available Tools: (Constrained set)                  │   │ │ │
│   │   │   │  - generate_campaign_strategy                        │   │ │ │
│   │   │   │  - search_knowledge_base                             │   │ │ │
│   │   │   │  - get_audience_insights                             │   │ │ │
│   │   │   │  - submit_for_approval                               │   │ │ │
│   │   │   │                                                       │   │ │ │
│   │   │   │  OUTPUT: Structured CampaignPlan JSON                │   │ │ │
│   │   │   └─────────────────────────────────────────────────────┘   │ │ │
│   │   │                          │                                   │ │ │
│   │   └──────────────────────────┼───────────────────────────────────┘ │ │
│   │                              │                                     │ │
│   └──────────────────────────────┼─────────────────────────────────────┘ │
│                                  │                                       │
│                                  ▼                                       │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                      CONTRACT VALIDATION                           │ │
│   │                                                                     │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │                 CONTRACT VALIDATOR                           │ │ │
│   │   │                                                               │ │ │
│   │   │   Input: Agent's structured output                           │ │ │
│   │   │   Schema: CampaignPlan (JSON Schema / Zod)                   │ │ │
│   │   │                                                               │ │ │
│   │   │   Validations:                                               │ │ │
│   │   │   ✓ Required fields present                                  │ │ │
│   │   │   ✓ Field types correct                                      │ │ │
│   │   │   ✓ Enum values valid                                        │ │ │
│   │   │   ✓ References exist (campaign_id, user_id)                  │ │ │
│   │   │   ✓ Business rules (tier 1 requires VP approval)             │ │ │
│   │   │                                                               │ │ │
│   │   │   Output: Validated CampaignPlan or ValidationError          │ │ │
│   │   └─────────────────────────────────────────────────────────────┘ │ │
│   │                              │                                     │ │
│   └──────────────────────────────┼─────────────────────────────────────┘ │
│                                  │                                       │
│                                  ▼                                       │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                      POLICY ENFORCEMENT                            │ │
│   │                                                                     │ │
│   │   ┌───────────────────┐      ┌───────────────────┐               │ │
│   │   │        PDP        │      │        PEP        │               │ │
│   │   │  (Policy Decision │      │  (Policy Enforce- │               │ │
│   │   │   Point)          │      │   ment Point)     │               │ │
│   │   │                   │      │                   │               │ │
│   │   │  Can user create  │─────▶│  Execute with:    │               │ │
│   │   │  tier 1 campaign? │      │  - Rate limiting  │               │ │
│   │   │  Budget within    │      │  - Timeout        │               │ │
│   │   │  limit?           │      │  - Rollback on    │               │ │
│   │   │  Required         │      │    failure        │               │ │
│   │   │  approvals?       │      │  - Audit logging  │               │ │
│   │   └───────────────────┘      └───────────────────┘               │ │
│   │                                      │                             │ │
│   └──────────────────────────────────────┼─────────────────────────────┘ │
│                                          │                               │
│                                          ▼                               │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                      TOOL GATEWAY                                  │ │
│   │                                                                     │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │              EXECUTION (Safe, Bounded)                       │ │ │
│   │   │                                                               │ │ │
│   │   │   create_campaign_in_mcc(validated_plan)                     │ │ │
│   │   │   ├── Store in DynamoDB                                      │ │ │
│   │   │   ├── Publish campaign.created event                         │ │ │
│   │   │   ├── Trigger sync to Workfront                              │ │ │
│   │   │   └── Return campaign_id                                     │ │ │
│   │   │                                                               │ │ │
│   │   │   All actions:                                               │ │ │
│   │   │   - Logged to audit trail                                    │ │ │
│   │   │   - Bounded by timeouts                                      │ │ │
│   │   │   - Retryable on transient failures                          │ │ │
│   │   │   - Reversible where possible                                │ │ │
│   │   └─────────────────────────────────────────────────────────────┘ │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Design Principles

| Principle | Implementation |
|-----------|----------------|
| **Least Privilege** | Agents have access only to tools they need |
| **Explicit Contracts** | All inputs/outputs have defined schemas |
| **Defense in Depth** | Multiple validation layers |
| **Fail Safe** | Invalid actions are blocked, not attempted |
| **Transparency** | Users can see what agent will do before execution |
| **Auditability** | Every decision and action is logged |

---

## 4. Contract Definition Language

### 4.1 Action Contract Schema

```typescript
// contracts/src/schemas/action-contract.ts

import { z } from 'zod';

/**
 * Every agent action is defined by a contract that specifies:
 * 1. What the action does
 * 2. What inputs it requires
 * 3. What outputs it produces
 * 4. What policies apply
 */

// Base action contract
const ActionContractSchema = z.object({
  // Unique identifier for this action type
  action_id: z.string(),

  // Human-readable name and description
  name: z.string(),
  description: z.string(),

  // Input schema (what the agent must provide)
  input_schema: z.record(z.any()), // JSON Schema

  // Output schema (what the action returns)
  output_schema: z.record(z.any()), // JSON Schema

  // Required permissions
  required_permissions: z.array(z.string()),

  // Policy rules that apply
  policy_rules: z.array(z.string()),

  // Whether action requires human approval
  requires_approval: z.boolean(),

  // Side effects (for documentation and rollback planning)
  side_effects: z.array(z.object({
    system: z.string(),
    operation: z.enum(['create', 'update', 'delete', 'notify']),
    description: z.string(),
    reversible: z.boolean(),
  })),

  // Rate limiting configuration
  rate_limit: z.object({
    max_per_minute: z.number(),
    max_per_hour: z.number(),
    max_per_day: z.number(),
  }).optional(),
});

type ActionContract = z.infer<typeof ActionContractSchema>;
```

### 4.2 Campaign Creation Contract

```typescript
// contracts/src/actions/create-campaign.ts

const CreateCampaignContract: ActionContract = {
  action_id: 'campaign.create',
  name: 'Create Campaign',
  description: 'Creates a new marketing campaign with strategy and execution setup',

  input_schema: {
    type: 'object',
    required: ['name', 'type', 'tier', 'launch_date', 'channels'],
    properties: {
      name: {
        type: 'string',
        minLength: 5,
        maxLength: 200,
        description: 'Campaign name (must be unique within tenant)',
      },
      description: {
        type: 'string',
        maxLength: 5000,
        description: 'Detailed campaign description',
      },
      type: {
        type: 'string',
        enum: ['standard', 'event', 'partner', 'product_launch'],
        description: 'Campaign type determines workflow and approvals',
      },
      tier: {
        type: 'integer',
        minimum: 1,
        maximum: 3,
        description: 'Tier 1 = highest visibility, Tier 3 = routine',
      },
      launch_date: {
        type: 'string',
        format: 'date',
        description: 'Planned launch date (ISO 8601)',
      },
      end_date: {
        type: 'string',
        format: 'date',
        description: 'Optional end date',
      },
      budget: {
        type: 'object',
        properties: {
          amount: { type: 'number', minimum: 0 },
          currency: { type: 'string', pattern: '^[A-Z]{3}$' },
        },
        description: 'Planned budget',
      },
      channels: {
        type: 'array',
        items: {
          type: 'string',
          enum: ['email', 'web', 'social', 'events', 'paid_media', 'partner'],
        },
        minItems: 1,
        description: 'Target channels',
      },
      audiences: {
        type: 'array',
        items: { type: 'string' },
        description: 'Target audience segments',
      },
      strategy: {
        type: 'object',
        description: 'Optional initial strategy content',
        properties: {
          messaging: { type: 'string' },
          key_messages: {
            type: 'array',
            items: { type: 'string' },
          },
          success_metrics: {
            type: 'array',
            items: { type: 'string' },
          },
        },
      },
    },
  },

  output_schema: {
    type: 'object',
    required: ['campaign_id', 'status'],
    properties: {
      campaign_id: {
        type: 'string',
        format: 'uuid',
        description: 'Unique identifier for created campaign',
      },
      status: {
        type: 'string',
        enum: ['created', 'pending_approval'],
        description: 'Initial status',
      },
      workfront_project_id: {
        type: 'string',
        description: 'If auto-created in Workfront',
      },
      next_steps: {
        type: 'array',
        items: { type: 'string' },
        description: 'Suggested next actions',
      },
    },
  },

  required_permissions: [
    'campaign:create',
    'campaign:set_tier',
  ],

  policy_rules: [
    'budget_within_limit',
    'tier_approval_required',
    'launch_date_not_past',
    'name_uniqueness',
  ],

  requires_approval: false, // Set dynamically based on tier

  side_effects: [
    {
      system: 'mcc',
      operation: 'create',
      description: 'Creates campaign record in MCC database',
      reversible: true,
    },
    {
      system: 'workfront',
      operation: 'create',
      description: 'Creates corresponding project in Workfront',
      reversible: true,
    },
    {
      system: 'eventbridge',
      operation: 'notify',
      description: 'Publishes campaign.created event',
      reversible: false,
    },
  ],

  rate_limit: {
    max_per_minute: 10,
    max_per_hour: 100,
    max_per_day: 500,
  },
};
```

### 4.3 Workfront Task Update Contract

```typescript
// contracts/src/actions/update-workfront-task.ts

const UpdateWorkfrontTaskContract: ActionContract = {
  action_id: 'workfront.task.update',
  name: 'Update Workfront Task',
  description: 'Updates an existing task in Workfront',

  input_schema: {
    type: 'object',
    required: ['task_id'],
    properties: {
      task_id: {
        type: 'string',
        description: 'Workfront task ID',
      },
      // Only certain fields can be updated by agent
      allowed_updates: {
        type: 'object',
        properties: {
          name: {
            type: 'string',
            maxLength: 200,
          },
          description: {
            type: 'string',
            maxLength: 5000,
          },
          planned_completion_date: {
            type: 'string',
            format: 'date',
          },
          // Status updates have restrictions
          status: {
            type: 'string',
            enum: ['INP', 'CPL'], // Can only set In Progress or Complete
            // Note: Cannot set 'APR' (Approved) - requires human
          },
        },
      },
    },
  },

  output_schema: {
    type: 'object',
    properties: {
      success: { type: 'boolean' },
      updated_fields: {
        type: 'array',
        items: { type: 'string' },
      },
      task_url: {
        type: 'string',
        format: 'uri',
      },
    },
  },

  required_permissions: [
    'workfront:task:update',
  ],

  policy_rules: [
    'task_belongs_to_user_campaign', // Can only update tasks in campaigns user owns
    'no_status_downgrade',           // Cannot move task backwards in workflow
    'no_approval_bypass',            // Cannot mark as approved without approver
  ],

  requires_approval: false,

  side_effects: [
    {
      system: 'workfront',
      operation: 'update',
      description: 'Updates task fields in Workfront',
      reversible: true,
    },
    {
      system: 'mcc',
      operation: 'update',
      description: 'Syncs update to MCC work_item record',
      reversible: true,
    },
  ],

  rate_limit: {
    max_per_minute: 30,
    max_per_hour: 300,
    max_per_day: 2000,
  },
};
```

---

## 5. Tool Gateway Design

### 5.1 Gateway Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TOOL GATEWAY (PEP) ARCHITECTURE                       │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                         TOOL GATEWAY                               │ │
│   │                                                                     │ │
│   │   Entry Point: All agent tool calls flow through here              │ │
│   │                                                                     │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │                    REQUEST PIPELINE                          │ │ │
│   │   │                                                               │ │ │
│   │   │   1. AUTHENTICATION                                          │ │ │
│   │   │      └── Verify agent identity and session                   │ │ │
│   │   │                                                               │ │ │
│   │   │   2. CONTRACT VALIDATION                                     │ │ │
│   │   │      └── Validate input against action contract schema       │ │ │
│   │   │                                                               │ │ │
│   │   │   3. POLICY DECISION (PDP Call)                              │ │ │
│   │   │      └── Evaluate all applicable policies                    │ │ │
│   │   │      └── Return: ALLOW / DENY / REQUIRE_APPROVAL             │ │ │
│   │   │                                                               │ │ │
│   │   │   4. RATE LIMITING                                           │ │ │
│   │   │      └── Check user/tenant/global rate limits                │ │ │
│   │   │                                                               │ │ │
│   │   │   5. EXECUTION                                               │ │ │
│   │   │      └── Call target system with timeout                     │ │ │
│   │   │      └── Handle retries on transient failures                │ │ │
│   │   │                                                               │ │ │
│   │   │   6. RESPONSE VALIDATION                                     │ │ │
│   │   │      └── Validate output against contract schema             │ │ │
│   │   │                                                               │ │ │
│   │   │   7. AUDIT LOGGING                                           │ │ │
│   │   │      └── Log request, response, decision, timing             │ │ │
│   │   │                                                               │ │ │
│   │   └─────────────────────────────────────────────────────────────┘ │ │
│   │                                                                     │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │                    ERROR HANDLING                            │ │ │
│   │   │                                                               │ │ │
│   │   │   Contract Violation → Return structured error to agent      │ │ │
│   │   │   Policy Denied      → Return denial reason to agent         │ │ │
│   │   │   Rate Limited       → Return retry-after to agent           │ │ │
│   │   │   Execution Error    → Log, retry if transient, else error   │ │ │
│   │   │   Timeout            → Log, return timeout error to agent    │ │ │
│   │   │                                                               │ │ │
│   │   └─────────────────────────────────────────────────────────────┘ │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Gateway Implementation

```typescript
// tool-gateway/src/gateway.ts

import { z } from 'zod';

interface ToolRequest {
  action_id: string;
  input: Record<string, unknown>;
  context: {
    user_id: string;
    tenant_id: string;
    agent_session_id: string;
    trace_id: string;
  };
}

interface ToolResponse {
  success: boolean;
  output?: Record<string, unknown>;
  error?: {
    code: string;
    message: string;
    details?: Record<string, unknown>;
  };
  metadata: {
    execution_time_ms: number;
    policy_decision: 'ALLOW' | 'DENY' | 'REQUIRE_APPROVAL';
    audit_id: string;
  };
}

class ToolGateway {
  private contractRegistry: ContractRegistry;
  private policyEngine: PolicyDecisionPoint;
  private rateLimiter: RateLimiter;
  private toolExecutors: Map<string, ToolExecutor>;
  private auditLogger: AuditLogger;

  async execute(request: ToolRequest): Promise<ToolResponse> {
    const startTime = Date.now();
    const auditId = generateAuditId();

    try {
      // 1. Get contract for action
      const contract = this.contractRegistry.get(request.action_id);
      if (!contract) {
        return this.errorResponse('UNKNOWN_ACTION', `Unknown action: ${request.action_id}`, auditId, startTime);
      }

      // 2. Validate input against contract schema
      const validationResult = this.validateInput(request.input, contract.input_schema);
      if (!validationResult.valid) {
        return this.errorResponse('CONTRACT_VIOLATION', validationResult.error, auditId, startTime, {
          schema_errors: validationResult.errors,
        });
      }

      // 3. Check policy
      const policyDecision = await this.policyEngine.evaluate({
        action_id: request.action_id,
        input: request.input,
        user_id: request.context.user_id,
        tenant_id: request.context.tenant_id,
        required_permissions: contract.required_permissions,
        policy_rules: contract.policy_rules,
      });

      if (policyDecision.decision === 'DENY') {
        return this.errorResponse('POLICY_DENIED', policyDecision.reason, auditId, startTime, {
          violated_rules: policyDecision.violated_rules,
        });
      }

      if (policyDecision.decision === 'REQUIRE_APPROVAL') {
        // Create approval request and return pending status
        const approvalId = await this.createApprovalRequest(request, contract, policyDecision);
        return {
          success: true,
          output: {
            status: 'pending_approval',
            approval_id: approvalId,
            required_approvers: policyDecision.required_approvers,
          },
          metadata: {
            execution_time_ms: Date.now() - startTime,
            policy_decision: 'REQUIRE_APPROVAL',
            audit_id: auditId,
          },
        };
      }

      // 4. Check rate limits
      const rateLimitResult = await this.rateLimiter.check(
        request.context.user_id,
        request.action_id,
        contract.rate_limit
      );
      if (!rateLimitResult.allowed) {
        return this.errorResponse('RATE_LIMITED', 'Rate limit exceeded', auditId, startTime, {
          retry_after_ms: rateLimitResult.retry_after_ms,
        });
      }

      // 5. Execute action
      const executor = this.toolExecutors.get(request.action_id);
      const output = await executor.execute(request.input, request.context);

      // 6. Validate output
      const outputValidation = this.validateOutput(output, contract.output_schema);
      if (!outputValidation.valid) {
        // Log warning but don't fail - output schema violations are internal issues
        console.warn('Output schema violation', { action_id: request.action_id, errors: outputValidation.errors });
      }

      // 7. Log audit
      await this.auditLogger.log({
        audit_id: auditId,
        action_id: request.action_id,
        user_id: request.context.user_id,
        tenant_id: request.context.tenant_id,
        input: this.sanitizeForAudit(request.input),
        output: this.sanitizeForAudit(output),
        policy_decision: 'ALLOW',
        execution_time_ms: Date.now() - startTime,
        timestamp: new Date().toISOString(),
      });

      return {
        success: true,
        output,
        metadata: {
          execution_time_ms: Date.now() - startTime,
          policy_decision: 'ALLOW',
          audit_id: auditId,
        },
      };
    } catch (error) {
      await this.auditLogger.logError({
        audit_id: auditId,
        action_id: request.action_id,
        error: error.message,
        stack: error.stack,
      });

      return this.errorResponse('EXECUTION_ERROR', error.message, auditId, startTime);
    }
  }

  private validateInput(input: unknown, schema: Record<string, any>): ValidationResult {
    // Use JSON Schema or Zod validation
    try {
      const validator = new Ajv().compile(schema);
      const valid = validator(input);
      return {
        valid,
        errors: validator.errors || [],
      };
    } catch (e) {
      return { valid: false, error: e.message, errors: [] };
    }
  }

  private sanitizeForAudit(data: Record<string, unknown>): Record<string, unknown> {
    // Remove PII, secrets before logging
    const sanitized = { ...data };
    const sensitiveFields = ['password', 'token', 'secret', 'ssn', 'credit_card'];
    for (const field of sensitiveFields) {
      if (field in sanitized) {
        sanitized[field] = '[REDACTED]';
      }
    }
    return sanitized;
  }

  private errorResponse(
    code: string,
    message: string,
    auditId: string,
    startTime: number,
    details?: Record<string, unknown>
  ): ToolResponse {
    return {
      success: false,
      error: { code, message, details },
      metadata: {
        execution_time_ms: Date.now() - startTime,
        policy_decision: 'DENY',
        audit_id: auditId,
      },
    };
  }
}
```

---

## 6. Policy Decision Point (PDP)

### 6.1 Policy Engine Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    POLICY DECISION POINT (PDP)                           │
│                                                                          │
│   Input: Action request + User context + Required permissions            │
│   Output: ALLOW | DENY | REQUIRE_APPROVAL + Reason                      │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    POLICY EVALUATION PIPELINE                      │ │
│   │                                                                     │ │
│   │   1. PERMISSION CHECK (RBAC)                                       │ │
│   │      ├── Get user's roles                                          │ │
│   │      ├── Get permissions for roles                                 │ │
│   │      └── Check required_permissions ⊆ user_permissions             │ │
│   │                                                                     │ │
│   │   2. ATTRIBUTE-BASED RULES (ABAC)                                  │ │
│   │      ├── Evaluate each policy_rule                                 │ │
│   │      ├── Rules can access: user, tenant, input, time, etc.         │ │
│   │      └── All rules must pass                                       │ │
│   │                                                                     │ │
│   │   3. APPROVAL REQUIREMENTS                                         │ │
│   │      ├── Check if action requires approval                         │ │
│   │      ├── Determine required approvers                              │ │
│   │      └── Return REQUIRE_APPROVAL if needed                         │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    EXAMPLE POLICIES                                │ │
│   │                                                                     │ │
│   │   POLICY: budget_within_limit                                      │ │
│   │   IF action.input.budget.amount > user.budget_limit                │ │
│   │   THEN DENY "Budget exceeds your limit of ${user.budget_limit}"    │ │
│   │                                                                     │ │
│   │   POLICY: tier_approval_required                                   │ │
│   │   IF action.input.tier == 1 AND user.role != 'director'            │ │
│   │   THEN REQUIRE_APPROVAL approvers=['director', 'vp_marketing']     │ │
│   │                                                                     │ │
│   │   POLICY: no_weekend_launches                                      │ │
│   │   IF dayOfWeek(action.input.launch_date) IN ['Saturday', 'Sunday'] │ │
│   │   THEN DENY "Launches cannot be scheduled on weekends"             │ │
│   │                                                                     │ │
│   │   POLICY: workfront_project_limit                                  │ │
│   │   IF count(user.active_campaigns) >= 10                            │ │
│   │   THEN DENY "You have reached max concurrent campaigns (10)"       │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Policy Definition Language

```typescript
// policy-engine/src/policies.ts

interface Policy {
  policy_id: string;
  name: string;
  description: string;
  applies_to: string[];  // Action IDs this policy applies to
  condition: PolicyCondition;
  effect: PolicyEffect;
}

type PolicyCondition =
  | { type: 'expression'; expression: string }  // CEL or similar
  | { type: 'function'; function: (ctx: PolicyContext) => boolean };

type PolicyEffect =
  | { type: 'DENY'; message: string }
  | { type: 'REQUIRE_APPROVAL'; approvers: string[]; message: string }
  | { type: 'ALLOW' };

interface PolicyContext {
  action: {
    action_id: string;
    input: Record<string, unknown>;
  };
  user: {
    user_id: string;
    email: string;
    roles: string[];
    permissions: string[];
    attributes: Record<string, unknown>;  // Custom attributes
  };
  tenant: {
    tenant_id: string;
    name: string;
    tier: string;
    settings: Record<string, unknown>;
  };
  time: {
    timestamp: Date;
    day_of_week: string;
    is_business_hours: boolean;
  };
  // Dynamic lookups
  lookups: {
    getUserCampaignCount: () => Promise<number>;
    getCampaignBudgetSpent: (campaignId: string) => Promise<number>;
    // ... other lookups
  };
}

// Example policies
const POLICIES: Policy[] = [
  {
    policy_id: 'budget_within_limit',
    name: 'Budget Within User Limit',
    description: 'Ensures campaign budget does not exceed user authorization',
    applies_to: ['campaign.create', 'campaign.update'],
    condition: {
      type: 'function',
      function: (ctx) => {
        const budget = ctx.action.input.budget?.amount || 0;
        const userLimit = ctx.user.attributes.budget_limit || 10000;
        return budget <= userLimit;
      },
    },
    effect: {
      type: 'DENY',
      message: 'Budget exceeds your authorization limit',
    },
  },
  {
    policy_id: 'tier1_requires_director_approval',
    name: 'Tier 1 Requires Director Approval',
    description: 'Tier 1 campaigns must be approved by a director',
    applies_to: ['campaign.create'],
    condition: {
      type: 'function',
      function: (ctx) => {
        const tier = ctx.action.input.tier;
        const isDirector = ctx.user.roles.includes('director');
        return tier !== 1 || isDirector;  // True = no approval needed
      },
    },
    effect: {
      type: 'REQUIRE_APPROVAL',
      approvers: ['director'],
      message: 'Tier 1 campaigns require director approval',
    },
  },
  {
    policy_id: 'no_approval_bypass',
    name: 'No Approval Status Bypass',
    description: 'Cannot mark work items as approved without proper authorization',
    applies_to: ['workfront.task.update'],
    condition: {
      type: 'function',
      function: (ctx) => {
        const newStatus = ctx.action.input.allowed_updates?.status;
        const isApprover = ctx.user.permissions.includes('approve:tasks');
        return newStatus !== 'APR' || isApprover;
      },
    },
    effect: {
      type: 'DENY',
      message: 'You do not have permission to approve tasks',
    },
  },
];
```

### 6.3 Role-Based Access Control (RBAC)

```typescript
// policy-engine/src/rbac.ts

interface Role {
  role_id: string;
  name: string;
  permissions: string[];
  inherits_from?: string[];  // Role inheritance
}

const ROLES: Role[] = [
  {
    role_id: 'viewer',
    name: 'Viewer',
    permissions: [
      'campaign:read',
      'work_item:read',
      'dashboard:view',
    ],
  },
  {
    role_id: 'marketer',
    name: 'Marketer',
    permissions: [
      'campaign:create',
      'campaign:read',
      'campaign:update',
      'work_item:create',
      'work_item:update',
      'strategy:create',
      'strategy:submit_for_review',
    ],
    inherits_from: ['viewer'],
  },
  {
    role_id: 'campaign_manager',
    name: 'Campaign Manager',
    permissions: [
      'campaign:set_tier',
      'campaign:set_budget',
      'work_item:assign',
      'strategy:approve',
    ],
    inherits_from: ['marketer'],
  },
  {
    role_id: 'director',
    name: 'Director',
    permissions: [
      'campaign:approve_tier1',
      'budget:approve_over_50k',
      'team:manage',
    ],
    inherits_from: ['campaign_manager'],
  },
  {
    role_id: 'admin',
    name: 'Administrator',
    permissions: [
      'tenant:manage',
      'user:manage',
      'integration:manage',
      'policy:manage',
    ],
    inherits_from: ['director'],
  },
];

class RBACEngine {
  private roleMap: Map<string, Role>;
  private permissionCache: Map<string, Set<string>>;

  getPermissionsForRole(roleId: string): Set<string> {
    if (this.permissionCache.has(roleId)) {
      return this.permissionCache.get(roleId)!;
    }

    const role = this.roleMap.get(roleId);
    if (!role) {
      return new Set();
    }

    const permissions = new Set(role.permissions);

    // Add inherited permissions
    for (const parentRoleId of role.inherits_from || []) {
      const parentPermissions = this.getPermissionsForRole(parentRoleId);
      for (const perm of parentPermissions) {
        permissions.add(perm);
      }
    }

    this.permissionCache.set(roleId, permissions);
    return permissions;
  }

  hasPermission(userRoles: string[], requiredPermission: string): boolean {
    for (const roleId of userRoles) {
      const permissions = this.getPermissionsForRole(roleId);
      if (permissions.has(requiredPermission)) {
        return true;
      }
    }
    return false;
  }
}
```

---

## 7. Agent Playbooks

### 7.1 Playbook Concept

Playbooks are predefined workflows that agents execute. They provide:
- **Structure**: Defined sequence of actions
- **Guardrails**: Each step has contracts and policies
- **Transparency**: Users can see what the playbook will do
- **Consistency**: Same playbook produces consistent results

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AGENT PLAYBOOK STRUCTURE                              │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                PLAYBOOK: Create Standard Campaign                  │ │
│   │                                                                     │ │
│   │   Trigger: User says "create a campaign for..."                    │ │
│   │                                                                     │ │
│   │   STEP 1: Gather Requirements                                      │ │
│   │   ├── Action: ASK_USER                                             │ │
│   │   ├── Required info: name, type, tier, channels, launch_date       │ │
│   │   └── Use knowledge base for suggestions                           │ │
│   │                                                                     │ │
│   │   STEP 2: Generate Strategy                                        │ │
│   │   ├── Action: generate_campaign_strategy                           │ │
│   │   ├── Input: Gathered requirements + playbook template             │ │
│   │   └── Output: Draft strategy document                              │ │
│   │                                                                     │ │
│   │   STEP 3: Review Strategy with User                                │ │
│   │   ├── Action: PRESENT_FOR_REVIEW                                   │ │
│   │   ├── Show: Generated strategy                                     │ │
│   │   └── Options: Approve / Edit / Regenerate                         │ │
│   │                                                                     │ │
│   │   STEP 4: Create Campaign Record                                   │ │
│   │   ├── Action: campaign.create                                      │ │
│   │   ├── Requires: User approval of strategy                          │ │
│   │   └── Policy check: budget, tier approval                          │ │
│   │                                                                     │ │
│   │   STEP 5: Setup Execution (if approved)                            │ │
│   │   ├── Action: workfront.project.create OR asana.project.create     │ │
│   │   ├── Input: Campaign details + task template                      │ │
│   │   └── Output: Project ID, initial tasks                            │ │
│   │                                                                     │ │
│   │   STEP 6: Confirm Completion                                       │ │
│   │   ├── Action: NOTIFY_USER                                          │ │
│   │   └── Show: Campaign ID, project links, next steps                 │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Playbook Definition

```typescript
// playbooks/src/create-standard-campaign.ts

interface PlaybookStep {
  step_id: string;
  name: string;
  type: 'action' | 'user_input' | 'review' | 'branch';
  config: StepConfig;
  on_success?: string;  // Next step ID
  on_failure?: string;  // Failure handling step
  on_approval?: string; // For review steps
  on_rejection?: string;
}

interface Playbook {
  playbook_id: string;
  name: string;
  description: string;
  trigger_patterns: string[];  // NL patterns that trigger this playbook
  required_permissions: string[];
  steps: PlaybookStep[];
  variables: PlaybookVariable[];  // Variables collected during execution
}

const CREATE_STANDARD_CAMPAIGN_PLAYBOOK: Playbook = {
  playbook_id: 'create_standard_campaign',
  name: 'Create Standard Campaign',
  description: 'Guides user through creating a new marketing campaign with strategy and execution setup',

  trigger_patterns: [
    'create a campaign',
    'start a new campaign',
    'I need to launch a campaign',
    'help me plan a campaign',
  ],

  required_permissions: ['campaign:create'],

  variables: [
    { name: 'campaign_name', type: 'string', required: true },
    { name: 'campaign_type', type: 'enum', options: ['standard', 'event', 'partner', 'product_launch'], required: true },
    { name: 'tier', type: 'number', min: 1, max: 3, required: true },
    { name: 'launch_date', type: 'date', required: true },
    { name: 'channels', type: 'array', items: 'channel_enum', required: true },
    { name: 'budget', type: 'object', schema: 'monetary_amount', required: false },
    { name: 'audiences', type: 'array', items: 'string', required: false },
    { name: 'strategy_content', type: 'object', schema: 'strategy', required: false },
  ],

  steps: [
    {
      step_id: 'gather_requirements',
      name: 'Gather Campaign Requirements',
      type: 'user_input',
      config: {
        prompt_template: `
          I'll help you create a new campaign. Let me gather some information:

          1. What's the campaign name?
          2. What type is it? (standard, event, partner, product_launch)
          3. What tier? (1=highest visibility, 3=routine)
          4. When should it launch?
          5. Which channels? (email, web, social, events, paid_media, partner)

          You can provide all at once or I'll ask follow-up questions.
        `,
        required_variables: ['campaign_name', 'campaign_type', 'tier', 'launch_date', 'channels'],
        validation_rules: [
          'launch_date must be in the future',
          'campaign_name must be unique',
        ],
      },
      on_success: 'generate_strategy',
    },
    {
      step_id: 'generate_strategy',
      name: 'Generate Campaign Strategy',
      type: 'action',
      config: {
        action_id: 'strategy.generate',
        input_mapping: {
          campaign_name: '${variables.campaign_name}',
          campaign_type: '${variables.campaign_type}',
          tier: '${variables.tier}',
          channels: '${variables.channels}',
          audiences: '${variables.audiences}',
        },
        output_variable: 'strategy_content',
        knowledge_base_query: 'playbook for ${variables.campaign_type} campaign',
      },
      on_success: 'review_strategy',
      on_failure: 'handle_strategy_error',
    },
    {
      step_id: 'review_strategy',
      name: 'Review Generated Strategy',
      type: 'review',
      config: {
        display_template: `
          ## Draft Strategy for "${variables.campaign_name}"

          ${variables.strategy_content.formatted}

          ---
          Please review and choose:
          - **Approve**: Create the campaign with this strategy
          - **Edit**: Modify specific sections
          - **Regenerate**: Generate a new strategy with different parameters
        `,
        options: ['approve', 'edit', 'regenerate'],
      },
      on_approval: 'create_campaign',
      on_rejection: 'edit_strategy',
    },
    {
      step_id: 'create_campaign',
      name: 'Create Campaign Record',
      type: 'action',
      config: {
        action_id: 'campaign.create',
        input_mapping: {
          name: '${variables.campaign_name}',
          type: '${variables.campaign_type}',
          tier: '${variables.tier}',
          launch_date: '${variables.launch_date}',
          channels: '${variables.channels}',
          budget: '${variables.budget}',
          strategy: '${variables.strategy_content}',
        },
        output_variable: 'campaign_result',
      },
      on_success: 'setup_execution',
      on_failure: 'handle_creation_error',
    },
    {
      step_id: 'setup_execution',
      name: 'Setup Execution System',
      type: 'branch',
      config: {
        condition: '${variables.campaign_type}',
        branches: {
          'event': 'create_asana_project',
          'default': 'create_workfront_project',
        },
      },
    },
    {
      step_id: 'create_workfront_project',
      name: 'Create Workfront Project',
      type: 'action',
      config: {
        action_id: 'workfront.project.create',
        input_mapping: {
          campaign_id: '${campaign_result.campaign_id}',
          name: '${variables.campaign_name}',
          template_id: '${get_template_for_type(variables.campaign_type)}',
        },
        output_variable: 'workfront_result',
      },
      on_success: 'confirm_completion',
    },
    {
      step_id: 'confirm_completion',
      name: 'Confirm Completion',
      type: 'action',
      config: {
        action_id: 'notify.user',
        input_mapping: {
          message_template: `
            ✅ Campaign created successfully!

            **Campaign ID**: ${campaign_result.campaign_id}
            **Workfront Project**: ${workfront_result.project_url}

            **Next Steps**:
            1. Review and finalize the strategy
            2. Assign team members to tasks
            3. Begin asset production

            Would you like me to help with any of these?
          `,
        },
      },
    },
  ],
};
```

---

## 8. Human-in-the-Loop Patterns

### 8.1 When to Require Human Approval

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HUMAN-IN-THE-LOOP DECISION MATRIX                     │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    APPROVAL REQUIREMENTS                           │ │
│   │                                                                     │ │
│   │   ACTION CATEGORY        │ ALWAYS HUMAN │ CONDITIONAL │ AUTO-OK   │ │
│   │   ───────────────────────┼──────────────┼─────────────┼───────────│ │
│   │   Create content         │              │      ✓      │           │ │
│   │   Publish content        │      ✓       │             │           │ │
│   │   Send notifications     │              │      ✓      │           │ │
│   │   Budget commitment      │      ✓       │             │           │ │
│   │   External API writes    │              │      ✓      │           │ │
│   │   Delete data            │      ✓       │             │           │ │
│   │   Read data              │              │             │     ✓     │ │
│   │   Generate suggestions   │              │             │     ✓     │ │
│   │   Search knowledge base  │              │             │     ✓     │ │
│   │                                                                     │ │
│   │   CONDITIONAL RULES:                                               │ │
│   │   • Content: Auto-ok for drafts, human for final approval          │ │
│   │   • Notifications: Auto-ok for <10 recipients, human for broadcast │ │
│   │   • External writes: Auto-ok for routine updates, human for creates│ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    APPROVAL WORKFLOW                               │ │
│   │                                                                     │ │
│   │   1. Agent proposes action                                         │ │
│   │   2. Policy engine flags for approval                              │ │
│   │   3. Create ApprovalRequest record                                 │ │
│   │   4. Notify required approvers (Slack, email)                      │ │
│   │   5. Approver reviews in UI                                        │ │
│   │   6. Approver: Approve / Reject / Modify                           │ │
│   │   7. If approved: Execute action with approval audit               │ │
│   │   8. Notify original user of outcome                               │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Approval Request Schema

```typescript
// approval/src/types.ts

interface ApprovalRequest {
  request_id: string;
  action_id: string;
  action_input: Record<string, unknown>;

  // Who requested
  requester: {
    user_id: string;
    email: string;
    agent_session_id: string;
  };

  // Why approval needed
  trigger: {
    policy_id: string;
    reason: string;
  };

  // Who can approve
  required_approvers: Array<{
    type: 'user' | 'role' | 'group';
    identifier: string;
  }>;
  approval_rule: 'any' | 'all' | 'majority';

  // Status
  status: 'pending' | 'approved' | 'rejected' | 'expired' | 'cancelled';
  decisions: ApprovalDecision[];

  // Timing
  created_at: string;
  expires_at: string;
  resolved_at?: string;
}

interface ApprovalDecision {
  approver_id: string;
  approver_email: string;
  decision: 'approved' | 'rejected';
  comment?: string;
  modifications?: Record<string, unknown>;  // Allowed changes
  decided_at: string;
}
```

---

## 9. Agent Observability

### 9.1 What to Monitor

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AGENT OBSERVABILITY METRICS                           │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    BUSINESS METRICS                                │ │
│   │                                                                     │ │
│   │   • Campaigns created via agent (vs manual)                        │ │
│   │   • Time to create campaign (agent vs manual)                      │ │
│   │   • Strategy approval rate (first attempt vs revision)             │ │
│   │   • User satisfaction (thumbs up/down)                             │ │
│   │   • Task completion rate                                           │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    TECHNICAL METRICS                               │ │
│   │                                                                     │ │
│   │   • Response latency (p50, p95, p99)                               │ │
│   │   • Token usage per conversation                                   │ │
│   │   • Tool call success rate                                         │ │
│   │   • Policy denial rate (and by policy)                             │ │
│   │   • Contract validation failure rate                               │ │
│   │   • External API error rate                                        │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    SAFETY METRICS                                  │ │
│   │                                                                     │ │
│   │   • Actions blocked by policy                                      │ │
│   │   • Approval requests created                                      │ │
│   │   • Approval rejection rate                                        │ │
│   │   • Agent "confusion" rate (fallback to human)                     │ │
│   │   • PII detection events                                           │ │
│   │   • Hallucination detection events                                 │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Distributed Tracing

```typescript
// observability/src/tracing.ts

interface AgentTrace {
  trace_id: string;
  session_id: string;
  user_id: string;
  tenant_id: string;

  spans: AgentSpan[];

  // Aggregated metrics
  total_duration_ms: number;
  total_llm_tokens: number;
  total_tool_calls: number;
  outcome: 'success' | 'failure' | 'abandoned';
}

interface AgentSpan {
  span_id: string;
  parent_span_id?: string;
  operation: string;
  start_time: string;
  end_time: string;
  duration_ms: number;

  // Span-specific attributes
  attributes: {
    // LLM spans
    model?: string;
    input_tokens?: number;
    output_tokens?: number;
    temperature?: number;

    // Tool spans
    action_id?: string;
    policy_decision?: string;
    external_system?: string;

    // Error info
    error?: string;
    error_type?: string;
  };
}

// Example trace for "create campaign" flow
const exampleTrace: AgentTrace = {
  trace_id: 'trace-abc123',
  session_id: 'session-xyz',
  user_id: 'user-123',
  tenant_id: 'tenant-456',
  spans: [
    {
      span_id: 'span-1',
      operation: 'user_message',
      start_time: '2026-01-10T14:00:00Z',
      end_time: '2026-01-10T14:00:00.100Z',
      duration_ms: 100,
      attributes: {},
    },
    {
      span_id: 'span-2',
      parent_span_id: 'span-1',
      operation: 'llm_inference',
      start_time: '2026-01-10T14:00:00.100Z',
      end_time: '2026-01-10T14:00:02.500Z',
      duration_ms: 2400,
      attributes: {
        model: 'claude-3-5-sonnet',
        input_tokens: 1500,
        output_tokens: 800,
      },
    },
    {
      span_id: 'span-3',
      parent_span_id: 'span-1',
      operation: 'tool_call',
      start_time: '2026-01-10T14:00:02.500Z',
      end_time: '2026-01-10T14:00:03.200Z',
      duration_ms: 700,
      attributes: {
        action_id: 'campaign.create',
        policy_decision: 'ALLOW',
        external_system: 'mcc',
      },
    },
    {
      span_id: 'span-4',
      parent_span_id: 'span-3',
      operation: 'external_api',
      start_time: '2026-01-10T14:00:03.200Z',
      end_time: '2026-01-10T14:00:04.000Z',
      duration_ms: 800,
      attributes: {
        external_system: 'workfront',
        action_id: 'workfront.project.create',
      },
    },
  ],
  total_duration_ms: 4000,
  total_llm_tokens: 2300,
  total_tool_calls: 2,
  outcome: 'success',
};
```

---

## 10. Agent Studio: User-Created Agents

### 10.1 Vision

> **Agent Studio** enables power users to create custom agents without code, using configuration and guardrails defined by the platform.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AGENT STUDIO CONCEPT                                  │
│                                                                          │
│   WHO: Power users, team leads, marketing ops                           │
│   WHAT: Create custom agents for specific workflows                     │
│   HOW: Configuration UI, not code                                       │
│   GUARD: Platform enforces contracts and policies                       │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    AGENT STUDIO UI                                 │ │
│   │                                                                     │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │  Create New Agent                                            │ │ │
│   │   │                                                               │ │ │
│   │   │  Name: [Event Campaign Assistant________________]            │ │ │
│   │   │  Description: [Helps event marketers plan Tier 1 events]    │ │ │
│   │   │                                                               │ │ │
│   │   │  Personality:                                                │ │ │
│   │   │  [Professional, detail-oriented, asks clarifying questions]  │ │ │
│   │   │                                                               │ │ │
│   │   │  Available Tools: (Select from allowed list)                 │ │ │
│   │   │  ☑ search_knowledge_base                                     │ │ │
│   │   │  ☑ generate_campaign_strategy                                │ │ │
│   │   │  ☑ create_asana_project                                      │ │ │
│   │   │  ☐ create_workfront_project  (Not for events)                │ │ │
│   │   │  ☑ send_slack_notification                                   │ │ │
│   │   │  ☐ approve_budget  (Requires director role)                  │ │ │
│   │   │                                                               │ │ │
│   │   │  Knowledge Sources:                                          │ │ │
│   │   │  ☑ Event Marketing Playbook                                  │ │ │
│   │   │  ☑ Tier 1 Event Standards                                    │ │ │
│   │   │  ☑ Brand Guidelines                                          │ │ │
│   │   │  ☐ Partner Marketing Guide                                   │ │ │
│   │   │                                                               │ │ │
│   │   │  Default Values:                                             │ │ │
│   │   │  campaign_type: event                                        │ │ │
│   │   │  tier: 1                                                     │ │ │
│   │   │                                                               │ │ │
│   │   │  [ Save Draft ]  [ Test Agent ]  [ Publish ]                │ │ │
│   │   └─────────────────────────────────────────────────────────────┘ │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   PLATFORM ENFORCES:                                                    │
│   • User can only select tools they have permission for                 │
│   • Contracts and policies still apply to all tool calls               │
│   • Agent cannot bypass approval requirements                           │
│   • All actions audited with agent creator attribution                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10.2 Custom Agent Schema

```typescript
// agent-studio/src/types.ts

interface CustomAgent {
  agent_id: string;
  name: string;
  description: string;

  // Creator
  created_by: string;
  tenant_id: string;
  team_id?: string;  // If team-specific

  // Personality
  system_prompt_additions: string;  // Added to base prompt
  personality_traits: string[];

  // Capabilities
  allowed_tools: string[];           // Subset of platform tools
  knowledge_sources: string[];       // KB collections
  default_values: Record<string, unknown>;

  // Restrictions
  restricted_actions: string[];      // Explicitly blocked
  max_actions_per_session: number;
  require_human_review: boolean;     // All actions need review

  // Access
  visibility: 'private' | 'team' | 'tenant';
  allowed_users?: string[];
  allowed_roles?: string[];

  // Status
  status: 'draft' | 'testing' | 'published' | 'archived';
  published_at?: string;
  version: number;
}
```

---

## Summary

### Key Architectural Decisions

1. **Contract-First, Not Prompt-First**
   - LLM generates structured outputs, not decisions
   - Contracts define validity, policies define permissibility

2. **Defense in Depth**
   - Contract validation → Policy evaluation → Safe execution
   - Multiple layers ensure no single point of failure

3. **Explicit Approvals**
   - High-impact actions require human sign-off
   - Approval flows are first-class citizens

4. **Playbooks Over Ad-Hoc**
   - Predefined workflows ensure consistency
   - Users know what agent will do before it acts

5. **Observability Built-In**
   - Every action traced and audited
   - Metrics for both business and safety

### Next Steps

1. Implement [Policy Engine](08-policy-engine.md) with full RBAC/ABAC
2. Build [Multi-Tenant Hub](09-multi-tenancy-hub.md) for shared infrastructure
3. Deploy [On-Prem Integration](10-on-prem-integration.md) for SharePoint
