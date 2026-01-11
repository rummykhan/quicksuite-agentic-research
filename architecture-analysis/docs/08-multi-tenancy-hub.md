# Multi-Tenant Integration Hub

## Building a Shared Integration Platform for Multiple Teams

### Version: 2.0
### Date: January 10, 2026
### Author: Principal Engineer / Solution Architect

---

## Table of Contents

1. [Business Context](#1-business-context)
2. [Multi-Tenancy Requirements](#2-multi-tenancy-requirements)
3. [Tenant Model](#3-tenant-model)
4. [Data Isolation Architecture](#4-data-isolation-architecture)
5. [Integration Sharing Model](#5-integration-sharing-model)
6. [Security Boundaries](#6-security-boundaries)
7. [Resource Management](#7-resource-management)
8. [Onboarding Workflow](#8-onboarding-workflow)
9. [Governance Model](#9-governance-model)
10. [Implementation Roadmap](#10-implementation-roadmap)

---

## 1. Business Context

### 1.1 The Opportunity

The sister team also uses Asana and Workfront. Rather than building duplicate integrations, we can create a **shared Integration Hub** that serves multiple teams while maintaining data isolation.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CURRENT STATE: SILOED INTEGRATIONS                    │
│                                                                          │
│   ┌─────────────────────────┐     ┌─────────────────────────┐          │
│   │   MARKETING TEAM        │     │   SISTER TEAM           │          │
│   │                         │     │                         │          │
│   │   Workfront ────┐       │     │   Workfront ────┐       │          │
│   │   Asana    ────┼──?     │     │   Asana    ────┼──?     │          │
│   │   Analytics────┘       │     │   Analytics────┘       │          │
│   │                         │     │                         │          │
│   │   Building own          │     │   Building own          │          │
│   │   integrations          │     │   integrations          │          │
│   │   (2026 roadmap)        │     │   (2026 roadmap)        │          │
│   │                         │     │                         │          │
│   └─────────────────────────┘     └─────────────────────────┘          │
│                                                                          │
│   PROBLEMS:                                                             │
│   • Duplicate engineering effort                                        │
│   • Inconsistent implementations                                        │
│   • No shared learnings                                                 │
│   • Higher total cost                                                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    TARGET STATE: SHARED INTEGRATION HUB                  │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    INTEGRATION HUB (Multi-Tenant)                  │ │
│   │                                                                     │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │                    SHARED CONNECTORS                         │ │ │
│   │   │                                                               │ │ │
│   │   │   Workfront     Asana      Analytics    SharePoint           │ │ │
│   │   │   Connector     Connector  Connector    Connector            │ │ │
│   │   │                                                               │ │ │
│   │   └─────────────────────────────────────────────────────────────┘ │ │
│   │                           │                                        │ │
│   │              ┌────────────┴────────────┐                          │ │
│   │              │                         │                          │ │
│   │              ▼                         ▼                          │ │
│   │   ┌──────────────────┐     ┌──────────────────┐                  │ │
│   │   │  MARKETING       │     │  SISTER TEAM     │                  │ │
│   │   │  TENANT          │     │  TENANT          │                  │ │
│   │   │                  │     │                  │                  │ │
│   │   │  • Own data      │     │  • Own data      │                  │ │
│   │   │  • Own tokens    │     │  • Own tokens    │                  │ │
│   │   │  • Own policies  │     │  • Own policies  │                  │ │
│   │   │  • Own users     │     │  • Own users     │                  │ │
│   │   └──────────────────┘     └──────────────────┘                  │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   BENEFITS:                                                             │
│   • Single integration investment                                       │
│   • Shared maintenance and improvements                                 │
│   • Consistent experience across teams                                  │
│   • Lower total cost of ownership                                       │
│   • Cross-team best practice sharing                                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Key Constraints

| Constraint | Implication |
|------------|-------------|
| **Data isolation** | Tenant A cannot see Tenant B's campaigns, data, or activity |
| **Credential isolation** | Each tenant manages their own OAuth connections |
| **Policy independence** | Each tenant can have custom policies and workflows |
| **Fair resource sharing** | No tenant can monopolize shared infrastructure |
| **Independent scaling** | Tenant growth doesn't impact other tenants |
| **Audit separation** | Compliance audits are tenant-specific |

---

## 2. Multi-Tenancy Requirements

### 2.1 Isolation Requirements

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ISOLATION REQUIREMENTS MATRIX                         │
│                                                                          │
│   RESOURCE               │ ISOLATION LEVEL │ IMPLEMENTATION              │
│   ───────────────────────┼─────────────────┼─────────────────────────────│
│   Campaign Data          │ STRICT          │ Tenant ID in every record   │
│   Work Items             │ STRICT          │ Tenant ID in every record   │
│   Strategies             │ STRICT          │ Tenant ID in every record   │
│   Measurements           │ STRICT          │ Tenant ID in every record   │
│   ───────────────────────┼─────────────────┼─────────────────────────────│
│   OAuth Tokens           │ STRICT          │ Per-tenant Secrets Manager  │
│   API Keys               │ STRICT          │ Per-tenant key namespaces   │
│   User Identity          │ STRICT          │ Tenant claim in JWT         │
│   ───────────────────────┼─────────────────┼─────────────────────────────│
│   Agent Conversations    │ STRICT          │ Tenant-scoped sessions      │
│   Audit Logs             │ STRICT          │ Tenant partition in logs    │
│   Approvals              │ STRICT          │ Tenant ID on all requests   │
│   ───────────────────────┼─────────────────┼─────────────────────────────│
│   Knowledge Base Content │ CONFIGURABLE    │ Per-tenant + shared pools   │
│   Playbook Templates     │ CONFIGURABLE    │ Per-tenant + platform       │
│   Policy Rules           │ CONFIGURABLE    │ Base + tenant overrides     │
│   ───────────────────────┼─────────────────┼─────────────────────────────│
│   Connector Code         │ SHARED          │ Single deployment           │
│   Platform Infrastructure│ SHARED          │ Multi-tenant design         │
│   Observability          │ SHARED          │ Tenant labels on metrics    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Non-Functional Requirements

| Requirement | Target | Measurement |
|-------------|--------|-------------|
| **Tenant onboarding** | < 1 day | Time from request to first API call |
| **Data isolation breach** | 0 incidents | Security audit findings |
| **Cross-tenant performance impact** | < 5% | Latency variance during load |
| **Tenant-specific downtime** | Independent | No cascading failures |
| **Per-tenant cost visibility** | 100% attributable | Cost allocation accuracy |

---

## 3. Tenant Model

### 3.1 Tenant Entity Schema

```typescript
// multi-tenancy/src/entities/tenant.ts

/**
 * A Tenant represents an organizational unit that uses the Integration Hub.
 * Tenants are isolated from each other in data, credentials, and policies.
 */
interface Tenant {
  tenant_id: string;            // UUID, globally unique

  // Identity
  name: string;                 // "Marketing", "Partner Marketing"
  slug: string;                 // "marketing", "partner-mktg"
  description: string;

  // Hierarchy
  parent_tenant_id?: string;    // For sub-tenants (BUs within org)
  tenant_type: 'organization' | 'business_unit' | 'team';

  // Status
  status: 'provisioning' | 'active' | 'suspended' | 'decommissioned';
  created_at: string;
  activated_at?: string;
  suspended_at?: string;
  suspension_reason?: string;

  // Configuration
  settings: TenantSettings;

  // Limits
  limits: TenantLimits;

  // Contacts
  owners: TenantContact[];      // Admin contacts
  billing_contact?: TenantContact;
}

interface TenantSettings {
  // Integration settings
  default_execution_system: 'workfront' | 'asana';
  enabled_integrations: string[];

  // Agent settings
  allowed_agents: string[];
  custom_agents_enabled: boolean;

  // Policy settings
  custom_policies_enabled: boolean;
  require_approval_for_tier1: boolean;
  max_budget_without_approval: number;

  // Knowledge settings
  knowledge_base_collections: string[];
  shared_kb_access: boolean;

  // Environment
  environment: 'production' | 'sandbox';
}

interface TenantLimits {
  max_users: number;
  max_campaigns_per_month: number;
  max_api_calls_per_day: number;
  max_storage_gb: number;
  max_agent_sessions_concurrent: number;
}

interface TenantContact {
  user_id: string;
  name: string;
  email: string;
  role: 'owner' | 'admin' | 'billing';
}
```

### 3.2 Tenant Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TENANT HIERARCHY MODEL                                │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    ORGANIZATION LEVEL                              │ │
│   │                                                                     │ │
│   │   Tenant: "ACME Corp" (type: organization)                         │ │
│   │   ├── Shared policies (base)                                       │ │
│   │   ├── Global knowledge base                                        │ │
│   │   └── Organization-wide settings                                   │ │
│   │                                                                     │ │
│   │                           │                                        │ │
│   │              ┌────────────┴────────────┐                          │ │
│   │              │                         │                          │ │
│   │              ▼                         ▼                          │ │
│   │   ┌──────────────────┐     ┌──────────────────┐                  │ │
│   │   │ MARKETING BU     │     │ PARTNER MKTG BU  │                  │ │
│   │   │ (type: BU)       │     │ (type: BU)       │                  │ │
│   │   │                  │     │                  │                  │ │
│   │   │ Inherits from:   │     │ Inherits from:   │                  │ │
│   │   │ - Org policies   │     │ - Org policies   │                  │ │
│   │   │ - Org KB         │     │ - Org KB         │                  │ │
│   │   │                  │     │                  │                  │ │
│   │   │ Adds:            │     │ Adds:            │                  │ │
│   │   │ - BU policies    │     │ - BU policies    │                  │ │
│   │   │ - BU KB content  │     │ - BU KB content  │                  │ │
│   │   │ - BU integrations│     │ - BU integrations│                  │ │
│   │   │                  │     │                  │                  │ │
│   │   │ Uses:            │     │ Uses:            │                  │ │
│   │   │ - Workfront      │     │ - Asana          │                  │ │
│   │   │ - Adobe Analytics│     │ - Adobe Analytics│                  │ │
│   │   └──────────────────┘     └──────────────────┘                  │ │
│   │              │                                                     │ │
│   │              ▼                                                     │ │
│   │   ┌──────────────────┐                                            │ │
│   │   │ EVENTS TEAM      │                                            │ │
│   │   │ (type: team)     │                                            │ │
│   │   │                  │                                            │ │
│   │   │ Inherits from:   │                                            │ │
│   │   │ - Marketing BU   │                                            │ │
│   │   │                  │                                            │ │
│   │   │ Adds:            │                                            │ │
│   │   │ - Event playbooks│                                            │ │
│   │   │ - Asana (events) │                                            │ │
│   │   └──────────────────┘                                            │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   INHERITANCE RULES:                                                    │
│   • Child tenants inherit parent policies (can add, not remove)         │
│   • Child tenants inherit parent KB (can add collections)               │
│   • Child tenants have independent data (no visibility to siblings)     │
│   • Limits cascade down (child cannot exceed parent)                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Data Isolation Architecture

### 4.1 Database Design for Multi-Tenancy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DATA ISOLATION STRATEGY                               │
│                                                                          │
│   APPROACH: Logical Isolation with Tenant ID                            │
│                                                                          │
│   Why not physical isolation (separate tables/DBs per tenant)?          │
│   • Initial tenant count: 2-10 (not hundreds)                           │
│   • Operational complexity of many databases                            │
│   • Cross-tenant analytics for platform team                            │
│   • Cost efficiency at small scale                                      │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    DYNAMODB TABLE DESIGN                           │ │
│   │                                                                     │ │
│   │   Table: campaigns                                                 │ │
│   │                                                                     │ │
│   │   Partition Key: TENANT#{tenant_id}#CAMPAIGN#{campaign_id}         │ │
│   │   Sort Key: METADATA                                               │ │
│   │                                                                     │ │
│   │   GSI1: tenant-status-index                                        │ │
│   │   GSI1PK: TENANT#{tenant_id}                                       │ │
│   │   GSI1SK: STATUS#{status}#CREATED#{created_at}                     │ │
│   │                                                                     │ │
│   │   Example Records:                                                 │ │
│   │   ┌──────────────────────────────────────────────────────────────┐│ │
│   │   │ pk: TENANT#mktg-001#CAMPAIGN#camp-123                        ││ │
│   │   │ sk: METADATA                                                  ││ │
│   │   │ tenant_id: mktg-001                                          ││ │
│   │   │ campaign_id: camp-123                                        ││ │
│   │   │ name: "Q1 Product Launch"                                    ││ │
│   │   │ status: "in_progress"                                        ││ │
│   │   │ ...                                                          ││ │
│   │   └──────────────────────────────────────────────────────────────┘│ │
│   │                                                                     │ │
│   │   EVERY query MUST include tenant_id in the key condition          │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    ACCESS PATTERNS                                 │ │
│   │                                                                     │ │
│   │   ✓ ALLOWED:                                                       │ │
│   │     Query(tenant_id='mktg-001', campaign_id='camp-123')           │ │
│   │     Query(tenant_id='mktg-001', status='in_progress')             │ │
│   │                                                                     │ │
│   │   ✗ FORBIDDEN (will fail):                                         │ │
│   │     Scan(filter=status='in_progress')  // No tenant!              │ │
│   │     Query(campaign_id='camp-123')      // No tenant!              │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Tenant Context Propagation

```typescript
// multi-tenancy/src/context.ts

/**
 * TenantContext is propagated through every layer of the application.
 * It's extracted from the authenticated user and validated at every boundary.
 */
interface TenantContext {
  tenant_id: string;
  tenant_name: string;
  tenant_type: 'organization' | 'business_unit' | 'team';
  parent_tenant_ids: string[];  // Hierarchy chain
  environment: 'production' | 'sandbox';

  // Effective limits (considering hierarchy)
  effective_limits: TenantLimits;

  // Effective settings (merged from hierarchy)
  effective_settings: TenantSettings;
}

/**
 * Every Lambda handler starts by extracting and validating tenant context.
 */
async function extractTenantContext(event: APIGatewayEvent): Promise<TenantContext> {
  // 1. Extract tenant claim from JWT
  const claims = decodeJwt(event.headers.Authorization);
  const tenantId = claims['custom:tenant_id'];

  if (!tenantId) {
    throw new UnauthorizedError('Missing tenant context');
  }

  // 2. Load tenant configuration
  const tenant = await tenantService.getTenant(tenantId);

  if (!tenant || tenant.status !== 'active') {
    throw new UnauthorizedError('Invalid or inactive tenant');
  }

  // 3. Build effective context (with inheritance)
  const parentChain = await tenantService.getParentChain(tenantId);
  const effectiveSettings = mergeSettings(parentChain);
  const effectiveLimits = computeEffectiveLimits(parentChain);

  return {
    tenant_id: tenantId,
    tenant_name: tenant.name,
    tenant_type: tenant.tenant_type,
    parent_tenant_ids: parentChain.map(t => t.tenant_id),
    environment: tenant.settings.environment,
    effective_limits: effectiveLimits,
    effective_settings: effectiveSettings,
  };
}

/**
 * Middleware that enforces tenant context on all database operations.
 */
class TenantAwareDynamoDB {
  constructor(
    private dynamodb: DynamoDB,
    private tenantContext: TenantContext
  ) {}

  async query(params: QueryInput): Promise<QueryOutput> {
    // Ensure tenant_id is in key condition
    if (!this.hasValidTenantKey(params)) {
      throw new SecurityError('Query must include tenant_id in key condition');
    }

    return this.dynamodb.query(params);
  }

  async put(params: PutItemInput): Promise<void> {
    // Ensure tenant_id matches context
    const itemTenantId = params.Item.tenant_id?.S;
    if (itemTenantId !== this.tenantContext.tenant_id) {
      throw new SecurityError('Cannot write to different tenant');
    }

    return this.dynamodb.put(params);
  }

  // Scan is disabled for tenant-scoped tables
  async scan(): Promise<never> {
    throw new SecurityError('Scan is not allowed on tenant-scoped tables');
  }

  private hasValidTenantKey(params: QueryInput): boolean {
    const keyCondition = params.KeyConditionExpression || '';
    return keyCondition.includes('tenant_id') ||
           keyCondition.includes(`TENANT#${this.tenantContext.tenant_id}`);
  }
}
```

### 4.3 Cross-Tenant Data Access (Admin Only)

```typescript
// multi-tenancy/src/admin/cross-tenant.ts

/**
 * Platform administrators may need cross-tenant visibility for:
 * - Usage reporting
 * - Compliance audits
 * - Support escalations
 *
 * This access is strictly controlled and logged.
 */
interface CrossTenantAccessRequest {
  requester_id: string;
  requester_role: 'platform_admin' | 'compliance_auditor';
  target_tenant_ids: string[];
  purpose: string;
  access_type: 'read_only' | 'read_write';
  duration_hours: number;
}

async function requestCrossTenantAccess(
  request: CrossTenantAccessRequest
): Promise<CrossTenantAccessGrant> {
  // 1. Validate requester has platform admin role
  if (!isPlatformAdmin(request.requester_id)) {
    throw new UnauthorizedError('Only platform admins can request cross-tenant access');
  }

  // 2. Log access request (pre-audit)
  await auditLogger.logCrossTenantAccessRequest(request);

  // 3. Create time-limited access grant
  const grant = {
    grant_id: generateGrantId(),
    ...request,
    granted_at: new Date().toISOString(),
    expires_at: addHours(new Date(), request.duration_hours).toISOString(),
    status: 'active',
  };

  await accessGrantStore.create(grant);

  // 4. Notify tenant owners
  for (const tenantId of request.target_tenant_ids) {
    await notifyTenantOwners(tenantId, {
      type: 'cross_tenant_access_granted',
      grant_id: grant.grant_id,
      requester: request.requester_id,
      purpose: request.purpose,
    });
  }

  return grant;
}
```

---

## 5. Integration Sharing Model

### 5.1 Shared Connectors, Isolated Credentials

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    INTEGRATION SHARING MODEL                             │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    SHARED CONNECTOR LAYER                          │ │
│   │                                                                     │ │
│   │   Single Deployment, Multi-Tenant Design                           │ │
│   │                                                                     │ │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │ │
│   │   │  Workfront  │  │   Asana     │  │  Analytics  │  ...          │ │
│   │   │  Connector  │  │  Connector  │  │  Connector  │               │ │
│   │   │             │  │             │  │             │               │ │
│   │   │  Shared     │  │  Shared     │  │  Shared     │               │ │
│   │   │  Code       │  │  Code       │  │  Code       │               │ │
│   │   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘               │ │
│   │          │                │                │                       │ │
│   └──────────┼────────────────┼────────────────┼───────────────────────┘ │
│              │                │                │                         │
│              ▼                ▼                ▼                         │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    CREDENTIAL ISOLATION LAYER                      │ │
│   │                                                                     │ │
│   │   AWS Secrets Manager Namespaces:                                  │ │
│   │                                                                     │ │
│   │   /mcc/tenants/mktg-001/integrations/workfront/oauth               │ │
│   │   /mcc/tenants/mktg-001/integrations/asana/oauth                   │ │
│   │   /mcc/tenants/mktg-001/integrations/analytics/apikey              │ │
│   │                                                                     │ │
│   │   /mcc/tenants/partner-001/integrations/workfront/oauth            │ │
│   │   /mcc/tenants/partner-001/integrations/asana/oauth                │ │
│   │   /mcc/tenants/partner-001/integrations/analytics/apikey           │ │
│   │                                                                     │ │
│   │   Each tenant's credentials are completely isolated:               │ │
│   │   • Different OAuth app registrations                              │ │
│   │   • Different API keys                                             │ │
│   │   • Independent token refresh                                      │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    REQUEST ROUTING                                 │ │
│   │                                                                     │ │
│   │   1. Request arrives with tenant context                           │ │
│   │   2. Connector looks up credentials for that tenant                │ │
│   │   3. Request made with tenant's credentials                        │ │
│   │   4. Response returned to tenant                                   │ │
│   │                                                                     │ │
│   │   async function executeWorkfrontOperation(                        │ │
│   │     tenantContext: TenantContext,                                  │ │
│   │     operation: Operation                                           │ │
│   │   ) {                                                              │ │
│   │     // Get tenant-specific credentials                             │ │
│   │     const creds = await secretsManager.getSecret(                  │ │
│   │       `/mcc/tenants/${tenantContext.tenant_id}/integrations/workfront/oauth`│ │
│   │     );                                                             │ │
│   │                                                                     │ │
│   │     // Execute with tenant's token                                 │ │
│   │     return workfrontClient.execute(operation, creds.access_token); │ │
│   │   }                                                                │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Integration Enablement Per Tenant

```typescript
// multi-tenancy/src/integrations/tenant-integrations.ts

interface TenantIntegration {
  tenant_id: string;
  integration_id: string;  // 'workfront', 'asana', etc.

  // Status
  status: 'not_configured' | 'configuring' | 'active' | 'error' | 'disabled';

  // Configuration
  config: {
    // Tenant-specific configuration
    instance_url?: string;       // e.g., their Workfront instance
    workspace_id?: string;       // e.g., their Asana workspace
    custom_fields_mapping?: Record<string, string>;
  };

  // Credentials reference (not the actual credentials)
  credentials_secret_arn: string;
  credentials_status: 'valid' | 'expiring_soon' | 'expired' | 'error';
  credentials_last_validated: string;

  // Usage
  last_used_at: string;
  total_api_calls_today: number;

  // Metadata
  configured_by: string;
  configured_at: string;
}

/**
 * Enable an integration for a tenant.
 * This creates the credential storage but doesn't populate credentials yet.
 */
async function enableIntegration(
  tenantContext: TenantContext,
  integrationId: string
): Promise<TenantIntegration> {
  // 1. Check if integration is allowed for this tenant
  if (!tenantContext.effective_settings.enabled_integrations.includes(integrationId)) {
    throw new ForbiddenError(`Integration ${integrationId} is not enabled for this tenant`);
  }

  // 2. Create secrets placeholder
  const secretArn = await secretsManager.createSecret({
    Name: `/mcc/tenants/${tenantContext.tenant_id}/integrations/${integrationId}/oauth`,
    Description: `OAuth credentials for ${integrationId} - Tenant: ${tenantContext.tenant_name}`,
    Tags: [
      { Key: 'tenant_id', Value: tenantContext.tenant_id },
      { Key: 'integration', Value: integrationId },
    ],
  });

  // 3. Create integration record
  const integration: TenantIntegration = {
    tenant_id: tenantContext.tenant_id,
    integration_id: integrationId,
    status: 'not_configured',
    config: {},
    credentials_secret_arn: secretArn,
    credentials_status: 'expired',
    credentials_last_validated: '',
    last_used_at: '',
    total_api_calls_today: 0,
    configured_by: tenantContext.user_id,
    configured_at: new Date().toISOString(),
  };

  await integrationStore.create(integration);

  return integration;
}
```

---

## 6. Security Boundaries

### 6.1 Authentication and Tenant Resolution

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION FLOW                                   │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    USER AUTHENTICATION                             │ │
│   │                                                                     │ │
│   │   1. User authenticates via Cognito                                │ │
│   │   2. JWT contains custom claims:                                   │ │
│   │      {                                                             │ │
│   │        "sub": "user-uuid",                                         │ │
│   │        "email": "sarah@company.com",                               │ │
│   │        "custom:tenant_id": "mktg-001",                             │ │
│   │        "custom:roles": ["campaign_manager"],                       │ │
│   │        "custom:permissions": ["campaign:create", ...]              │ │
│   │      }                                                             │ │
│   │                                                                     │ │
│   │   3. Tenant assignment happens during user provisioning            │ │
│   │   4. Users cannot change their tenant claim                        │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    REQUEST AUTHORIZATION                           │ │
│   │                                                                     │ │
│   │   API Gateway Authorizer:                                          │ │
│   │                                                                     │ │
│   │   1. Validate JWT signature                                        │ │
│   │   2. Check token expiration                                        │ │
│   │   3. Verify issuer (our Cognito pool)                              │ │
│   │   4. Extract tenant_id claim                                       │ │
│   │   5. Validate tenant is active                                     │ │
│   │   6. Return policy with tenant context                             │ │
│   │                                                                     │ │
│   │   Returned context passed to Lambda:                               │ │
│   │   {                                                                │ │
│   │     "principalId": "user-uuid",                                    │ │
│   │     "context": {                                                   │ │
│   │       "tenant_id": "mktg-001",                                     │ │
│   │       "user_email": "sarah@company.com",                           │ │
│   │       "roles": "campaign_manager"                                  │ │
│   │     }                                                              │ │
│   │   }                                                                │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Cross-Tenant Attack Prevention

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CROSS-TENANT ATTACK PREVENTION                        │
│                                                                          │
│   THREAT: Tenant A tries to access Tenant B's data                      │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    ATTACK VECTOR 1: Direct ID Manipulation         │ │
│   │                                                                     │ │
│   │   Attack: User from Tenant A sends:                                │ │
│   │   GET /campaigns/camp-123-from-tenant-b                            │ │
│   │                                                                     │ │
│   │   Prevention:                                                      │ │
│   │   • Campaign lookup ALWAYS includes tenant_id from JWT             │ │
│   │   • Query: tenant_id = {from_jwt} AND campaign_id = {from_request} │ │
│   │   • Returns 404 (not 403) to avoid enumeration                     │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    ATTACK VECTOR 2: JWT Tampering                  │ │
│   │                                                                     │ │
│   │   Attack: Modify tenant_id claim in JWT                            │ │
│   │                                                                     │ │
│   │   Prevention:                                                      │ │
│   │   • JWT signature verification (RS256)                             │ │
│   │   • Cannot modify claims without private key                       │ │
│   │   • Cognito private key never exposed                              │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    ATTACK VECTOR 3: Credential Access              │ │
│   │                                                                     │ │
│   │   Attack: Access another tenant's OAuth tokens                     │ │
│   │                                                                     │ │
│   │   Prevention:                                                      │ │
│   │   • Secrets Manager resource policy per tenant                     │ │
│   │   • Lambda can only access: /mcc/tenants/{jwt_tenant_id}/*        │ │
│   │   • IAM policy denies access to other tenant paths                │ │
│   │                                                                     │ │
│   │   IAM Policy:                                                      │ │
│   │   {                                                                │ │
│   │     "Effect": "Allow",                                             │ │
│   │     "Action": "secretsmanager:GetSecretValue",                    │ │
│   │     "Resource": "arn:aws:secretsmanager:*:*:secret:/mcc/tenants/${aws:PrincipalTag/tenant_id}/*"│ │
│   │   }                                                                │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    ATTACK VECTOR 4: Webhook Spoofing               │ │
│   │                                                                     │ │
│   │   Attack: Send fake webhook claiming to be from Workfront          │ │
│   │                                                                     │ │
│   │   Prevention:                                                      │ │
│   │   • Webhook signature verification (HMAC)                          │ │
│   │   • Secret key per tenant per integration                          │ │
│   │   • Tenant ID in webhook URL path: /webhooks/{tenant_id}/workfront │ │
│   │   • Verify signature matches tenant's secret                       │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Resource Management

### 7.1 Tenant Limits and Quotas

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    RESOURCE LIMITS BY TIER                               │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    TENANT TIERS                                    │ │
│   │                                                                     │ │
│   │   TIER        │ STANDARD    │ PROFESSIONAL │ ENTERPRISE           │ │
│   │   ────────────┼─────────────┼──────────────┼──────────────────────│ │
│   │   Users       │ 25          │ 100          │ Unlimited            │ │
│   │   Campaigns/mo│ 50          │ 200          │ Unlimited            │ │
│   │   API calls/hr│ 10,000      │ 50,000       │ 200,000              │ │
│   │   Storage     │ 10 GB       │ 50 GB        │ 500 GB               │ │
│   │   Agent sess. │ 5 concurrent│ 20 concurrent│ 100 concurrent       │ │
│   │   Custom agents│ 5          │ 25           │ Unlimited            │ │
│   │   KB size     │ 1 GB        │ 10 GB        │ 100 GB               │ │
│   │   ────────────┼─────────────┼──────────────┼──────────────────────│ │
│   │   Price       │ $500/mo     │ $2,000/mo    │ Custom               │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    LIMIT ENFORCEMENT                               │ │
│   │                                                                     │ │
│   │   Rate Limits (API Gateway + Lambda):                              │ │
│   │   • Token bucket algorithm per tenant                              │ │
│   │   • Redis for distributed counting                                 │ │
│   │   • 429 response when exceeded                                     │ │
│   │                                                                     │ │
│   │   Quota Limits (Database + Background Check):                      │ │
│   │   • Campaigns: Count on create, reject if over                     │ │
│   │   • Storage: Track per-tenant, alert at 80%                        │ │
│   │   • Users: Cognito group membership count                          │ │
│   │                                                                     │ │
│   │   async function checkQuota(                                       │ │
│   │     tenantContext: TenantContext,                                  │ │
│   │     resource: string,                                              │ │
│   │     increment: number = 1                                          │ │
│   │   ): Promise<QuotaCheckResult> {                                   │ │
│   │     const current = await getUsage(tenantContext.tenant_id, resource);│ │
│   │     const limit = tenantContext.effective_limits[resource];        │ │
│   │                                                                     │ │
│   │     if (current + increment > limit) {                             │ │
│   │       return {                                                     │ │
│   │         allowed: false,                                            │ │
│   │         current,                                                   │ │
│   │         limit,                                                     │ │
│   │         message: `Quota exceeded: ${resource}`                     │ │
│   │       };                                                           │ │
│   │     }                                                              │ │
│   │     return { allowed: true, current, limit };                      │ │
│   │   }                                                                │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Cost Attribution

```typescript
// multi-tenancy/src/billing/cost-attribution.ts

interface TenantUsageMetrics {
  tenant_id: string;
  period: string;  // "2026-01" (monthly)

  // Compute
  lambda_invocations: number;
  lambda_duration_ms: number;
  step_function_transitions: number;

  // AI
  bedrock_input_tokens: number;
  bedrock_output_tokens: number;

  // Storage
  dynamodb_read_units: number;
  dynamodb_write_units: number;
  s3_storage_gb: number;

  // Integrations
  external_api_calls: {
    workfront: number;
    asana: number;
    analytics: number;
  };

  // Calculated cost (USD)
  estimated_cost: number;
}

/**
 * Collect usage metrics tagged by tenant for cost attribution.
 */
async function collectTenantUsage(tenantId: string, period: string): Promise<TenantUsageMetrics> {
  const [
    lambdaMetrics,
    bedrockMetrics,
    dynamoMetrics,
    s3Metrics,
    apiCallMetrics,
  ] = await Promise.all([
    cloudwatch.getMetrics('AWS/Lambda', { tenant_id: tenantId }, period),
    cloudwatch.getMetrics('AWS/Bedrock', { tenant_id: tenantId }, period),
    cloudwatch.getMetrics('AWS/DynamoDB', { tenant_id: tenantId }, period),
    s3.getStorageMetrics(tenantId, period),
    getApiCallMetrics(tenantId, period),
  ]);

  return {
    tenant_id: tenantId,
    period,
    lambda_invocations: lambdaMetrics.invocations,
    lambda_duration_ms: lambdaMetrics.duration,
    step_function_transitions: lambdaMetrics.stepFunctionTransitions,
    bedrock_input_tokens: bedrockMetrics.inputTokens,
    bedrock_output_tokens: bedrockMetrics.outputTokens,
    dynamodb_read_units: dynamoMetrics.readUnits,
    dynamodb_write_units: dynamoMetrics.writeUnits,
    s3_storage_gb: s3Metrics.storageGb,
    external_api_calls: apiCallMetrics,
    estimated_cost: calculateCost({
      // ... metrics
    }),
  };
}
```

---

## 8. Onboarding Workflow

### 8.1 Tenant Provisioning

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TENANT ONBOARDING WORKFLOW                            │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │  STEP 1: Request (Day 0)                                           │ │
│   │                                                                     │ │
│   │  • Team lead submits onboarding request form                       │ │
│   │  • Provides: Team name, primary contact, expected usage            │ │
│   │  • Selects: Tier (Standard/Professional/Enterprise)               │ │
│   │  • Specifies: Required integrations                                │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                    │                                     │
│                                    ▼                                     │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │  STEP 2: Approval (Day 0-1)                                        │ │
│   │                                                                     │ │
│   │  • Platform admin reviews request                                  │ │
│   │  • Validates business justification                                │ │
│   │  • Approves or requests more info                                  │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                    │                                     │
│                                    ▼                                     │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │  STEP 3: Provisioning (Automated, ~5 min)                          │ │
│   │                                                                     │ │
│   │  Step Functions Workflow:                                          │ │
│   │  1. Create Tenant record in DynamoDB                               │ │
│   │  2. Create Cognito User Pool Group                                 │ │
│   │  3. Create Secrets Manager namespaces                              │ │
│   │  4. Create S3 prefix for tenant data                               │ │
│   │  5. Initialize tenant settings                                     │ │
│   │  6. Create admin user account                                      │ │
│   │  7. Send welcome email with login instructions                     │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                    │                                     │
│                                    ▼                                     │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │  STEP 4: Integration Setup (Self-Service, Day 1)                   │ │
│   │                                                                     │ │
│   │  Tenant admin logs in and:                                         │ │
│   │  1. Connects Workfront (OAuth flow)                                │ │
│   │  2. Connects Asana (OAuth flow)                                    │ │
│   │  3. Configures knowledge base sources                              │ │
│   │  4. Invites initial users                                          │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                    │                                     │
│                                    ▼                                     │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │  STEP 5: Verification (Day 1)                                      │ │
│   │                                                                     │ │
│   │  Platform team:                                                    │ │
│   │  1. Verifies integrations working                                  │ │
│   │  2. Runs test agent conversation                                   │ │
│   │  3. Confirms data isolation                                        │ │
│   │  4. Marks tenant as 'active'                                       │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                    │                                     │
│                                    ▼                                     │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │  STEP 6: Go Live                                                   │ │
│   │                                                                     │ │
│   │  • Tenant admin notified of activation                             │ │
│   │  • Users can start creating campaigns                              │ │
│   │  • Usage monitoring begins                                         │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Integration Connection Flow

```typescript
// multi-tenancy/src/onboarding/integration-setup.ts

/**
 * OAuth connection flow for tenant integrations.
 * Each tenant has their own OAuth app registration in the target system.
 */
async function initiateIntegrationConnection(
  tenantContext: TenantContext,
  integrationId: string
): Promise<{ authorizationUrl: string; state: string }> {
  // 1. Generate state parameter (CSRF protection)
  const state = generateSecureState();
  await stateStore.save(state, {
    tenant_id: tenantContext.tenant_id,
    integration_id: integrationId,
    initiated_by: tenantContext.user_id,
    expires_at: addMinutes(new Date(), 10).toISOString(),
  });

  // 2. Get integration config
  const integration = getIntegrationConfig(integrationId);

  // 3. Build authorization URL
  // Note: Each tenant should have their own OAuth app in Workfront/Asana
  // For initial setup, we might use a shared app with workspace selection
  const authUrl = new URL(integration.auth_endpoint);
  authUrl.searchParams.set('client_id', integration.client_id);
  authUrl.searchParams.set('redirect_uri', `${MCC_BASE_URL}/oauth/callback`);
  authUrl.searchParams.set('scope', integration.scopes.join(' '));
  authUrl.searchParams.set('state', state);
  authUrl.searchParams.set('response_type', 'code');

  return {
    authorizationUrl: authUrl.toString(),
    state,
  };
}

/**
 * Handle OAuth callback and store credentials.
 */
async function handleOAuthCallback(
  code: string,
  state: string
): Promise<{ success: boolean; integration_id: string }> {
  // 1. Validate and retrieve state
  const stateData = await stateStore.get(state);
  if (!stateData || new Date(stateData.expires_at) < new Date()) {
    throw new InvalidStateError('Invalid or expired state');
  }

  // 2. Exchange code for tokens
  const integration = getIntegrationConfig(stateData.integration_id);
  const tokens = await exchangeCodeForTokens(code, integration);

  // 3. Store tokens in tenant-specific secret
  await secretsManager.putSecretValue({
    SecretId: `/mcc/tenants/${stateData.tenant_id}/integrations/${stateData.integration_id}/oauth`,
    SecretString: JSON.stringify({
      access_token: tokens.access_token,
      refresh_token: tokens.refresh_token,
      expires_at: tokens.expires_at,
      token_type: tokens.token_type,
      scope: tokens.scope,
    }),
  });

  // 4. Update integration status
  await updateTenantIntegration(stateData.tenant_id, stateData.integration_id, {
    status: 'active',
    credentials_status: 'valid',
    credentials_last_validated: new Date().toISOString(),
  });

  // 5. Clean up state
  await stateStore.delete(state);

  return {
    success: true,
    integration_id: stateData.integration_id,
  };
}
```

---

## 9. Governance Model

### 9.1 Platform vs Tenant Responsibilities

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    GOVERNANCE MODEL                                      │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    PLATFORM TEAM RESPONSIBILITIES                  │ │
│   │                                                                     │ │
│   │   Infrastructure:                                                  │ │
│   │   • AWS account management                                         │ │
│   │   • Platform deployment and upgrades                               │ │
│   │   • Security patching                                              │ │
│   │   • Disaster recovery                                              │ │
│   │                                                                     │ │
│   │   Development:                                                     │ │
│   │   • Connector development and maintenance                          │ │
│   │   • Core agent capabilities                                        │ │
│   │   • Platform features and improvements                             │ │
│   │                                                                     │ │
│   │   Operations:                                                      │ │
│   │   • Monitoring and alerting                                        │ │
│   │   • Incident response                                              │ │
│   │   • Performance optimization                                       │ │
│   │                                                                     │ │
│   │   Governance:                                                      │ │
│   │   • Tenant onboarding approval                                     │ │
│   │   • Base policy definition                                         │ │
│   │   • Compliance and audit support                                   │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    TENANT ADMIN RESPONSIBILITIES                   │ │
│   │                                                                     │ │
│   │   Integration Management:                                          │ │
│   │   • OAuth app setup in target systems                              │ │
│   │   • Connection establishment and refresh                           │ │
│   │   • Custom field mapping                                           │ │
│   │                                                                     │ │
│   │   User Management:                                                 │ │
│   │   • User provisioning and deprovisioning                           │ │
│   │   • Role assignment                                                │ │
│   │   • Access reviews                                                 │ │
│   │                                                                     │ │
│   │   Configuration:                                                   │ │
│   │   • Tenant-specific policies                                       │ │
│   │   • Knowledge base content                                         │ │
│   │   • Custom agents (if enabled)                                     │ │
│   │   • Workflow templates                                             │ │
│   │                                                                     │ │
│   │   Compliance:                                                      │ │
│   │   • Data classification                                            │ │
│   │   • Audit log review                                               │ │
│   │   • Incident reporting                                             │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Change Management

```typescript
// multi-tenancy/src/governance/change-management.ts

/**
 * Platform changes that affect all tenants require careful rollout.
 */
interface PlatformChange {
  change_id: string;
  type: 'feature' | 'bugfix' | 'security' | 'breaking';
  description: string;
  affected_tenants: 'all' | string[];  // Specific tenant IDs

  rollout_strategy: {
    type: 'immediate' | 'gradual' | 'opt_in';
    percentage_per_stage?: number[];  // For gradual
    stage_duration_hours?: number;
  };

  // Communication
  notification_required: boolean;
  notification_sent_at?: string;

  // Rollback
  rollback_plan: string;
  rollback_triggered: boolean;
}

/**
 * Tenant-specific changes are managed by tenant admins.
 */
interface TenantChange {
  change_id: string;
  tenant_id: string;
  type: 'policy' | 'integration' | 'user' | 'config';
  description: string;
  requested_by: string;

  // Approval
  requires_approval: boolean;
  approver_id?: string;
  approved_at?: string;

  // Audit
  before_state: Record<string, unknown>;
  after_state: Record<string, unknown>;
  applied_at: string;
}
```

---

## 10. Implementation Roadmap

### 10.1 Phased Rollout

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MULTI-TENANCY IMPLEMENTATION PHASES                   │
│                                                                          │
│   PHASE 1: Foundation (Weeks 1-4)                                       │
│   ───────────────────────────────                                       │
│   • Tenant entity and database schema                                   │
│   • Tenant context propagation                                          │
│   • Data isolation layer                                                │
│   • Single-tenant deployment (Marketing team)                           │
│                                                                          │
│   Deliverable: Marketing team fully operational                         │
│                                                                          │
│   PHASE 2: Multi-Tenant Infrastructure (Weeks 5-8)                      │
│   ────────────────────────────────────────────────                      │
│   • Tenant provisioning workflow                                        │
│   • Per-tenant credential isolation                                     │
│   • Rate limiting and quotas                                            │
│   • Tenant admin UI                                                     │
│                                                                          │
│   Deliverable: Ready for second tenant                                  │
│                                                                          │
│   PHASE 3: Sister Team Onboarding (Weeks 9-10)                          │
│   ────────────────────────────────────────────                          │
│   • Onboard sister team as second tenant                                │
│   • Integration connections (Workfront, Asana)                          │
│   • Knowledge base separation                                           │
│   • User provisioning                                                   │
│                                                                          │
│   Deliverable: Two tenants operational                                  │
│                                                                          │
│   PHASE 4: Refinement (Weeks 11-12)                                     │
│   ─────────────────────────────────                                     │
│   • Cost attribution reporting                                          │
│   • Cross-tenant analytics (platform team)                              │
│   • Self-service onboarding portal                                      │
│   • Documentation and training                                          │
│                                                                          │
│   Deliverable: Production-ready multi-tenant platform                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10.2 Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Tenant onboarding time** | < 1 business day | Request to first API call |
| **Data isolation** | 0 breaches | Security audit + automated tests |
| **Cross-tenant impact** | < 5% latency variance | P99 latency comparison |
| **Platform availability** | 99.9% | Per-tenant uptime |
| **Cost visibility** | 100% attributable | Monthly cost reports |

---

## Summary

### Key Architectural Decisions

1. **Logical isolation, not physical**
   - Tenant ID in every record
   - Shared infrastructure, isolated data
   - Simpler operations at current scale

2. **Shared connectors, isolated credentials**
   - Single codebase for integrations
   - Per-tenant OAuth tokens
   - Independent token lifecycle

3. **Hierarchical tenant model**
   - Organization → BU → Team
   - Policy and settings inheritance
   - Flexible for organizational structures

4. **Self-service with guardrails**
   - Tenant admins manage their integration connections
   - Platform team manages infrastructure
   - Clear responsibility boundaries

### Next Steps

1. Implement [On-Prem Integration](09-on-prem-integration.md) for SharePoint
2. Build [Platform Feasibility Analysis](10-platform-feasibility.md) comparing QuickSuite, Salesforce, and custom
