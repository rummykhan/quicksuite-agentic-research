# Integration Patterns: OAuth vs Service-to-Service

## A Principal Engineer's Deep Analysis

### Version: 2.0
### Date: January 10, 2026
### Author: Principal Engineer / Solution Architect

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Integration Pattern Taxonomy](#2-integration-pattern-taxonomy)
3. [OAuth (User-Delegated) Integration](#3-oauth-user-delegated-integration)
4. [Service-to-Service Integration](#4-service-to-service-integration)
5. [Pattern Selection Matrix](#5-pattern-selection-matrix)
6. [System Classification](#6-system-classification)
7. [Trust Boundaries](#7-trust-boundaries)
8. [Implementation Architecture](#8-implementation-architecture)
9. [Security Considerations](#9-security-considerations)
10. [Operational Concerns](#10-operational-concerns)

---

## 1. Executive Summary

### The Core Question

When building an enterprise marketing platform that integrates with multiple external systems, the fundamental architectural question is: **Who is making the request, and on whose behalf?**

This question leads to two distinct integration patterns:

| Pattern | Identity | Use Case | Example |
|---------|----------|----------|---------|
| **OAuth (User-Delegated)** | Human user, scoped permissions | Interactive actions in SaaS tools | "Create a project in Workfront as Sarah" |
| **Service-to-Service (S2S)** | Machine identity, system-level | Batch operations, metrics collection | "Sync all campaign metrics from Analytics" |

### Key Insight

> **The choice isn't binary.** A well-architected enterprise platform uses BOTH patterns strategically, choosing the right pattern for each integration based on data sensitivity, action type, and audit requirements.

### Marketing Command Center Recommendation

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    RECOMMENDED INTEGRATION STRATEGY                      │
│                                                                          │
│   EXECUTION SYSTEMS (Workfront, Asana)                                  │
│   └── Primary: OAuth with Integration Owner Pattern                     │
│   └── Fallback: Service Account for batch sync                          │
│                                                                          │
│   KNOWLEDGE SYSTEMS (SharePoint On-Prem, Quip, S3)                      │
│   └── Primary: Service-to-Service                                       │
│   └── Reason: Read-heavy, no user context needed                        │
│                                                                          │
│   MEASUREMENT SYSTEMS (Analytics, Metrics APIs)                         │
│   └── Primary: Service-to-Service (API keys / STS)                      │
│   └── Reason: Aggregated data, no PII writes                            │
│                                                                          │
│   ASSET SYSTEMS (Adobe DAM)                                             │
│   └── Primary: Service-to-Service for reads                             │
│   └── OAuth for uploads (requires user attribution)                     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Integration Pattern Taxonomy

### 2.1 Visual Taxonomy

```
                        INTEGRATION PATTERNS
                               │
           ┌───────────────────┴───────────────────┐
           │                                       │
    USER-DELEGATED                          MACHINE-TO-MACHINE
    (OAuth 2.0)                             (Service-to-Service)
           │                                       │
    ┌──────┴──────┐                    ┌──────────┼──────────┐
    │             │                    │          │          │
 3-Legged     Integration          API Keys    STS/IAM    mTLS
 OAuth        Owner                            Roles
    │             │                    │          │          │
 Per-user     Org-level            Static    Dynamic    Cert-based
 tokens       OAuth app            secrets   credentials  auth
```

### 2.2 Pattern Definitions

#### OAuth 2.0 (User-Delegated)

**Definition**: Authentication flow where a user explicitly grants an application permission to act on their behalf within defined scopes.

**Characteristics**:
- User initiates authorization
- Tokens scoped to specific permissions
- Refresh token lifecycle management
- Actions attributed to the user
- Respects user's access level in target system

**Variants**:
1. **3-Legged OAuth**: Traditional per-user authorization
2. **Integration Owner Pattern**: Single org-level connection, platform enforces user context

#### Service-to-Service (S2S)

**Definition**: Machine identity authentication where one system authenticates directly to another without user involvement.

**Characteristics**:
- No user context in authentication
- System-level permissions
- API keys, certificates, or temporary credentials
- Actions attributed to the service/application
- Typically broader access scope

**Variants**:
1. **Static API Keys**: Simple, long-lived credentials
2. **STS (Security Token Service)**: Short-lived, rotated credentials
3. **IAM Roles**: Cloud-native identity (AWS AssumeRole)
4. **mTLS**: Mutual TLS certificate authentication

---

## 3. OAuth (User-Delegated) Integration

### 3.1 Standard 3-Legged OAuth Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STANDARD 3-LEGGED OAUTH FLOW                          │
│                                                                          │
│   ┌──────────┐         ┌──────────────┐         ┌──────────────────┐   │
│   │   User   │         │   Platform   │         │  Target System   │   │
│   │ (Sarah)  │         │  (MCC Hub)   │         │   (Workfront)    │   │
│   └────┬─────┘         └──────┬───────┘         └────────┬─────────┘   │
│        │                      │                          │              │
│        │  1. "Connect Workfront"                         │              │
│        │─────────────────────▶│                          │              │
│        │                      │                          │              │
│        │  2. Redirect to Workfront OAuth                 │              │
│        │◀─────────────────────│                          │              │
│        │                      │                          │              │
│        │  3. Login + Consent (scopes: read, write)       │              │
│        │────────────────────────────────────────────────▶│              │
│        │                      │                          │              │
│        │  4. Authorization Code                          │              │
│        │◀────────────────────────────────────────────────│              │
│        │                      │                          │              │
│        │  5. Code handoff     │                          │              │
│        │─────────────────────▶│                          │              │
│        │                      │                          │              │
│        │                      │  6. Exchange code for tokens            │
│        │                      │─────────────────────────▶│              │
│        │                      │                          │              │
│        │                      │  7. Access + Refresh tokens             │
│        │                      │◀─────────────────────────│              │
│        │                      │                          │              │
│        │  8. "Connected!"     │                          │              │
│        │◀─────────────────────│                          │              │
│        │                      │                          │              │
│   ┌────┴─────┐         ┌──────┴───────┐         ┌────────┴─────────┐   │
│   │          │         │              │         │                  │   │
│   │ Sarah's  │         │ Encrypted    │         │ Sarah's scoped   │   │
│   │ browser  │         │ token store  │         │ permissions      │   │
│   └──────────┘         └──────────────┘         └──────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 The Per-User Token Problem

**Challenge**: In a marketing organization with 200+ users across 6 personas, requiring each user to individually connect to Workfront, Asana, and other systems creates:

| Problem | Impact |
|---------|--------|
| **Onboarding friction** | Each user must complete OAuth flow for each system |
| **Token sprawl** | 200 users × 5 systems = 1,000 token pairs to manage |
| **Refresh failures** | Individual tokens expire, causing per-user outages |
| **Permission drift** | Users leave, tokens remain active |
| **Audit complexity** | Correlating actions across systems per-user |

### 3.3 Integration Owner Pattern (Recommended)

**Solution**: A single organizational OAuth connection where the platform acts as the integration owner, enforcing user context through application-level RBAC.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    INTEGRATION OWNER PATTERN                             │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    ONE-TIME ADMIN SETUP                            │ │
│   │                                                                     │ │
│   │   Org Admin (IT/Ops) performs OAuth consent once:                  │ │
│   │                                                                     │ │
│   │   1. Register MCC as OAuth app in Workfront/Asana                  │ │
│   │   2. Grant scopes: read, write, admin (for the app, not users)     │ │
│   │   3. Store tokens in platform's secure vault                       │ │
│   │   4. Enable API access for all workspace users                     │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    RUNTIME AUTHORIZATION                           │ │
│   │                                                                     │ │
│   │   ┌─────────┐      ┌───────────────┐      ┌──────────────────┐    │ │
│   │   │  Sarah  │─────▶│   MCC Hub     │─────▶│    Workfront     │    │ │
│   │   │(Marketer│      │               │      │                  │    │ │
│   │   └─────────┘      │ 1. Authenticate│     │                  │    │ │
│   │                    │    via Cognito │      │                  │    │ │
│   │                    │               │      │                  │    │ │
│   │                    │ 2. Check RBAC │      │                  │    │ │
│   │                    │    "Can Sarah │      │                  │    │ │
│   │                    │     create    │      │                  │    │ │
│   │                    │     projects?"│      │                  │    │ │
│   │                    │               │      │                  │    │ │
│   │                    │ 3. If allowed,│      │                  │    │ │
│   │                    │    use org's  │      │                  │    │ │
│   │                    │    OAuth token│      │                  │    │ │
│   │                    │               │      │                  │    │ │
│   │                    │ 4. Call API   │      │                  │    │ │
│   │                    │    with       │─────▶│  Create project  │    │ │
│   │                    │    attribution│      │  (attributed to  │    │ │
│   │                    │               │      │   Integration    │    │ │
│   │                    │               │      │   Owner or       │    │ │
│   │                    │               │      │   mapped user)   │    │ │
│   │                    └───────────────┘      └──────────────────┘    │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   KEY BENEFITS:                                                         │
│   ✓ Single token pair per target system (1 instead of 200)             │
│   ✓ Platform controls authorization (RBAC/ABAC)                        │
│   ✓ Centralized audit log                                              │
│   ✓ No user onboarding friction                                        │
│   ✓ Simpler token lifecycle management                                 │
│                                                                          │
│   TRADE-OFFS:                                                           │
│   ⚠ Actions attributed to integration app, not individual user         │
│   ⚠ Platform must implement its own RBAC layer                         │
│   ⚠ Requires admin-level OAuth consent                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.4 Token Lifecycle Management

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TOKEN LIFECYCLE STATE MACHINE                         │
│                                                                          │
│   ┌─────────────┐                                                       │
│   │   INITIAL   │                                                       │
│   │   (No token)│                                                       │
│   └──────┬──────┘                                                       │
│          │ Admin completes OAuth                                        │
│          ▼                                                              │
│   ┌─────────────┐                                                       │
│   │   ACTIVE    │◀─────────────────────────────────────┐               │
│   │             │                                       │               │
│   │ access_token│      Refresh successful              │               │
│   │ (valid)     │                                       │               │
│   └──────┬──────┘                                       │               │
│          │                                              │               │
│          │ Access token expires (typically 1hr)        │               │
│          ▼                                              │               │
│   ┌─────────────┐                                       │               │
│   │  REFRESHING │───────────────────────────────────────┘               │
│   │             │                                                       │
│   │ Using       │                                                       │
│   │ refresh_token                                                       │
│   └──────┬──────┘                                                       │
│          │                                                              │
│          │ Refresh token expired or revoked                            │
│          ▼                                                              │
│   ┌─────────────┐      ┌─────────────┐                                 │
│   │   EXPIRED   │─────▶│   ALERT     │                                 │
│   │             │      │             │                                 │
│   │ Requires    │      │ Notify ops  │                                 │
│   │ re-auth     │      │ team        │                                 │
│   └─────────────┘      └─────────────┘                                 │
│                                                                          │
│   IMPLEMENTATION:                                                       │
│                                                                          │
│   DynamoDB Schema for Token Storage:                                    │
│   {                                                                     │
│     "pk": "INTEGRATION#workfront",                                      │
│     "sk": "TENANT#acme-marketing",                                      │
│     "access_token": "encrypted:...",                                    │
│     "refresh_token": "encrypted:...",                                   │
│     "expires_at": "2026-01-10T15:00:00Z",                               │
│     "refresh_expires_at": "2026-04-10T12:00:00Z",                       │
│     "scopes": ["read", "write", "admin"],                               │
│     "last_used": "2026-01-10T14:30:00Z",                                │
│     "status": "ACTIVE"                                                  │
│   }                                                                     │
│                                                                          │
│   Lambda: token-refresh-handler                                         │
│   - Triggered by EventBridge schedule (every 45 min)                   │
│   - Checks tokens expiring within 15 min                                │
│   - Proactively refreshes to avoid interruption                         │
│   - Publishes CloudWatch metric on failure                              │
│   - SNS alert to ops team on refresh failure                            │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Service-to-Service Integration

### 4.1 When to Use S2S

Service-to-Service integration is appropriate when:

| Criterion | Example |
|-----------|---------|
| **No user context needed** | Syncing campaign metrics nightly |
| **Batch/bulk operations** | Indexing 10,000 documents from SharePoint |
| **System-level data** | Reading DAM asset metadata |
| **Read-only aggregation** | Pulling analytics dashboards |
| **Background processing** | Scheduled workflow triggers |

### 4.2 S2S Authentication Methods

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    S2S AUTHENTICATION METHODS                            │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │ METHOD 1: API KEYS                                                 │ │
│   │                                                                     │ │
│   │ Characteristics:                                                   │ │
│   │ - Static, long-lived credentials                                   │ │
│   │ - Simple to implement                                              │ │
│   │ - Risk: If leaked, requires manual rotation                        │ │
│   │                                                                     │ │
│   │ Use For:                                                           │ │
│   │ - Low-sensitivity read operations                                  │ │
│   │ - Third-party services without better options                      │ │
│   │                                                                     │ │
│   │ Storage: AWS Secrets Manager with automatic rotation               │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │ METHOD 2: STS / TEMPORARY CREDENTIALS                              │ │
│   │                                                                     │ │
│   │ ┌─────────┐      ┌─────────┐      ┌─────────┐      ┌─────────┐   │ │
│   │ │ Lambda  │─────▶│   STS   │─────▶│ Assume  │─────▶│  Target │   │ │
│   │ │Function │      │AssumeRole│     │  Role   │      │  API    │   │ │
│   │ └─────────┘      └─────────┘      └─────────┘      └─────────┘   │ │
│   │                                                                     │ │
│   │ Characteristics:                                                   │ │
│   │ - Short-lived (15 min - 12 hours)                                  │ │
│   │ - Automatically rotated                                            │ │
│   │ - Scoped to specific permissions                                   │ │
│   │ - Cloud-native (AWS, Azure, GCP)                                   │ │
│   │                                                                     │ │
│   │ Use For:                                                           │ │
│   │ - AWS service-to-service (S3, DynamoDB, Bedrock)                   │ │
│   │ - Cross-account access                                             │ │
│   │ - Federated identity scenarios                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │ METHOD 3: CLIENT CREDENTIALS (OAuth 2.0 Machine-to-Machine)        │ │
│   │                                                                     │ │
│   │ ┌─────────┐      ┌─────────┐      ┌─────────┐                     │ │
│   │ │  MCC    │─────▶│  Auth   │─────▶│  Target │                     │ │
│   │ │ Service │      │ Server  │      │   API   │                     │ │
│   │ └─────────┘      └─────────┘      └─────────┘                     │ │
│   │      │                │                │                           │ │
│   │      │ client_id +    │                │                           │ │
│   │      │ client_secret  │                │                           │ │
│   │      └───────────────▶│                │                           │ │
│   │                       │                │                           │ │
│   │                       │ access_token   │                           │ │
│   │      ◀────────────────┘                │                           │ │
│   │      │                                 │                           │ │
│   │      │ Bearer token                    │                           │ │
│   │      └────────────────────────────────▶│                           │ │
│   │                                                                     │ │
│   │ Use For:                                                           │ │
│   │ - Adobe APIs (Workfront, Analytics, DAM)                           │ │
│   │ - Salesforce (JWT Bearer flow)                                     │ │
│   │ - Modern SaaS APIs                                                 │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │ METHOD 4: MUTUAL TLS (mTLS)                                        │ │
│   │                                                                     │ │
│   │ Characteristics:                                                   │ │
│   │ - Certificate-based authentication                                 │ │
│   │ - Both client and server verify each other                         │ │
│   │ - Highest security for sensitive integrations                      │ │
│   │                                                                     │ │
│   │ Use For:                                                           │ │
│   │ - On-premises system connections                                   │ │
│   │ - Financial/healthcare data                                        │ │
│   │ - Zero-trust network architectures                                 │ │
│   │                                                                     │ │
│   │ Implementation:                                                    │ │
│   │ - AWS Private CA for certificate management                        │ │
│   │ - API Gateway mutual TLS support                                   │ │
│   │ - Certificate rotation via ACM                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.3 S2S for Internal AWS Services

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AWS IAM ROLE-BASED ACCESS                             │
│                                                                          │
│   Lambda Function → S3 (Knowledge Documents)                            │
│                                                                          │
│   IAM Role: mcc-knowledge-indexer-role                                  │
│                                                                          │
│   {                                                                     │
│     "Version": "2012-10-17",                                            │
│     "Statement": [                                                      │
│       {                                                                 │
│         "Effect": "Allow",                                              │
│         "Action": [                                                     │
│           "s3:GetObject",                                               │
│           "s3:ListBucket"                                               │
│         ],                                                              │
│         "Resource": [                                                   │
│           "arn:aws:s3:::mcc-knowledge-base/*",                          │
│           "arn:aws:s3:::mcc-knowledge-base"                             │
│         ]                                                               │
│       },                                                                │
│       {                                                                 │
│         "Effect": "Allow",                                              │
│         "Action": [                                                     │
│           "bedrock:InvokeModel"                                         │
│         ],                                                              │
│         "Resource": [                                                   │
│           "arn:aws:bedrock:*::foundation-model/amazon.titan-embed-*"    │
│         ]                                                               │
│       },                                                                │
│       {                                                                 │
│         "Effect": "Allow",                                              │
│         "Action": [                                                     │
│           "aoss:APIAccessAll"                                           │
│         ],                                                              │
│         "Resource": [                                                   │
│           "arn:aws:aoss:us-west-2:*:collection/mcc-vectors"             │
│         ]                                                               │
│       }                                                                 │
│     ]                                                                   │
│   }                                                                     │
│                                                                          │
│   PRINCIPLE: Least Privilege                                            │
│   - Read-only where possible                                            │
│   - Specific resource ARNs (no wildcards)                               │
│   - Separate roles for separate functions                               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Pattern Selection Matrix

### 5.1 Decision Framework

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    INTEGRATION PATTERN DECISION TREE                     │
│                                                                          │
│                         Is the action...                                │
│                              │                                          │
│              ┌───────────────┴───────────────┐                          │
│              │                               │                          │
│         WRITE/MODIFY                     READ-ONLY                      │
│              │                               │                          │
│              │                               │                          │
│         Does action                     Is data...                      │
│         need user                            │                          │
│         attribution?                         │                          │
│              │                    ┌──────────┴──────────┐               │
│       ┌──────┴──────┐             │                     │               │
│       │             │        AGGREGATED              USER-SPECIFIC      │
│      YES           NO        (no PII)                 (PII)             │
│       │             │             │                     │               │
│       ▼             ▼             ▼                     ▼               │
│   ┌───────┐   ┌───────┐     ┌───────┐             ┌───────┐            │
│   │ OAuth │   │  S2S  │     │  S2S  │             │ OAuth │            │
│   │(Integ │   │       │     │       │             │       │            │
│   │Owner) │   │       │     │       │             │       │            │
│   └───────┘   └───────┘     └───────┘             └───────┘            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 System-by-System Recommendations

| System | Primary Pattern | Rationale | Implementation |
|--------|-----------------|-----------|----------------|
| **Workfront** | OAuth (Integration Owner) | Write operations need attribution for audit | Single org OAuth app, RBAC in MCC |
| **Asana** | OAuth (Integration Owner) | Write operations, event project creation | Same as Workfront |
| **Adobe Analytics** | S2S (Client Credentials) | Read-only aggregated metrics | JWT Bearer token |
| **Adobe DAM** | Hybrid | Read: S2S; Write: OAuth (asset attribution) | Dual auth paths |
| **SharePoint On-Prem** | S2S (Service Account + mTLS) | Read-only indexing, no user context | Bridge agent with cert |
| **Quip** | S2S (API Token) | Read-only template retrieval | API key with rotation |
| **S3** | S2S (IAM Role) | Internal AWS, no external auth | Lambda execution role |
| **Slack** | OAuth (Bot Token) | Read/write messages | Bot-level OAuth |
| **Salesforce** | OAuth (Integration Owner) | CRM data needs careful access control | JWT Bearer or Integration User |

### 5.3 Workfront Deep Dive

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    WORKFRONT INTEGRATION ARCHITECTURE                    │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │ AUTHENTICATION: OAuth 2.0 (Integration Owner)                      │ │
│   │                                                                     │ │
│   │ Setup:                                                             │ │
│   │ 1. Create OAuth2 App in Workfront Setup                            │ │
│   │ 2. Grant scopes: user:read, project:write, task:write, issue:read  │ │
│   │ 3. Admin authorizes app for organization                           │ │
│   │ 4. Store tokens in Secrets Manager                                 │ │
│   │                                                                     │ │
│   │ API Base: https://{company}.my.workfront.com/attask/api/v20.0     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │ WRITE OPERATIONS (OAuth Required)                                  │ │
│   │                                                                     │ │
│   │ Project Creation:                                                  │ │
│   │ POST /proj                                                         │ │
│   │ Authorization: Bearer {access_token}                               │ │
│   │ X-MCC-User-Context: {cognito_user_id}  (for audit)                 │ │
│   │ {                                                                  │ │
│   │   "name": "Q1 Product Launch",                                     │ │
│   │   "templateID": "template-12345",                                  │ │
│   │   "plannedStartDate": "2026-02-01",                                │ │
│   │   "ownerID": "{workfront_integration_user}",                       │ │
│   │   "DE:MCC_Campaign_ID": "camp-uuid-here"  // Custom field link     │ │
│   │ }                                                                  │ │
│   │                                                                     │ │
│   │ Note: ownerID is the integration user, but MCC logs the actual    │ │
│   │       requesting user (Sarah) in its own audit trail.              │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │ READ OPERATIONS (OAuth or API Key)                                 │ │
│   │                                                                     │ │
│   │ For batch sync (nightly), S2S API key is acceptable:              │ │
│   │                                                                     │ │
│   │ GET /proj?fields=*,tasks:*                                        │ │
│   │ apiKey: {workfront_api_key}                                        │ │
│   │                                                                     │ │
│   │ This retrieves project data without user context, which is fine    │ │
│   │ for background sync operations.                                    │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │ WEBHOOK HANDLING (Event Subscriptions)                             │ │
│   │                                                                     │ │
│   │ Workfront pushes events to MCC:                                    │ │
│   │                                                                     │ │
│   │ POST https://api.mcc.example.com/webhooks/workfront               │ │
│   │ X-Workfront-Signature: {hmac_sha256}                               │ │
│   │ {                                                                  │ │
│   │   "newState": { ... },                                             │ │
│   │   "oldState": { ... },                                             │ │
│   │   "subscriptionId": "sub-12345",                                   │ │
│   │   "eventType": "UPDATE",                                           │ │
│   │   "eventTime": "2026-01-10T14:30:00Z"                              │ │
│   │ }                                                                  │ │
│   │                                                                     │ │
│   │ Verification: HMAC signature validation (shared secret)           │ │
│   └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. System Classification

### 6.1 The Four Data Planes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MARKETING SYSTEM CLASSIFICATION                       │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    EXECUTION SYSTEMS                             │   │
│   │                    (Where work happens)                          │   │
│   │                                                                   │   │
│   │   ┌────────────┐  ┌────────────┐  ┌────────────┐               │   │
│   │   │ Workfront  │  │   Asana    │  │   Slack    │               │   │
│   │   │            │  │            │  │            │               │   │
│   │   │ Tasks      │  │ Events     │  │ Comms      │               │   │
│   │   │ Projects   │  │ Partner    │  │ Approvals  │               │   │
│   │   │ Approvals  │  │ Programs   │  │            │               │   │
│   │   └────────────┘  └────────────┘  └────────────┘               │   │
│   │                                                                   │   │
│   │   Integration: OAuth (Integration Owner) - writes need attribution│   │
│   │   Sync Pattern: Bi-directional, event-driven                      │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    KNOWLEDGE SYSTEMS                             │   │
│   │                    (Where playbooks & templates live)            │   │
│   │                                                                   │   │
│   │   ┌────────────┐  ┌────────────┐  ┌────────────┐               │   │
│   │   │ SharePoint │  │    Quip    │  │     S3     │               │   │
│   │   │ (On-Prem)  │  │            │  │            │               │   │
│   │   │            │  │            │  │            │               │   │
│   │   │ Playbooks  │  │ Templates  │  │ Assets     │               │   │
│   │   │ Standards  │  │ Guidelines │  │ Docs       │               │   │
│   │   └────────────┘  └────────────┘  └────────────┘               │   │
│   │                                                                   │   │
│   │   Integration: S2S - read-only indexing, no user context          │   │
│   │   Sync Pattern: Periodic crawl + change detection                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    MEASUREMENT SYSTEMS                           │   │
│   │                    (Where performance data lives)                │   │
│   │                                                                   │   │
│   │   ┌────────────┐  ┌────────────┐  ┌────────────┐               │   │
│   │   │  Adobe     │  │ Salesforce │  │  Internal  │               │   │
│   │   │ Analytics  │  │ Reporting  │  │  Metrics   │               │   │
│   │   │            │  │            │  │            │               │   │
│   │   │ Web/Mobile │  │ Attribution│  │ Custom KPIs│               │   │
│   │   │ Metrics    │  │ Pipeline   │  │            │               │   │
│   │   └────────────┘  └────────────┘  └────────────┘               │   │
│   │                                                                   │   │
│   │   Integration: S2S - aggregated data, no PII writes               │   │
│   │   Sync Pattern: Scheduled batch (hourly/daily)                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    ASSET SYSTEMS                                 │   │
│   │                    (Where creative content lives)                │   │
│   │                                                                   │   │
│   │   ┌────────────┐  ┌────────────┐                                │   │
│   │   │ Adobe DAM  │  │   Brand    │                                │   │
│   │   │            │  │   Portal   │                                │   │
│   │   │            │  │            │                                │   │
│   │   │ Images     │  │ Guidelines │                                │   │
│   │   │ Videos     │  │ Logos      │                                │   │
│   │   │ Templates  │  │ Fonts      │                                │   │
│   │   └────────────┘  └────────────┘                                │   │
│   │                                                                   │   │
│   │   Integration: Hybrid                                             │   │
│   │   - Read: S2S (metadata search)                                   │   │
│   │   - Write: OAuth (uploads need user attribution)                  │   │
│   │   Sync Pattern: Event-driven (asset published triggers)          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Integration Priority Matrix

| Priority | System | Pattern | Complexity | 2026 Roadmap |
|----------|--------|---------|------------|--------------|
| **P0** | Workfront | OAuth (IO) | High | Q1 |
| **P1** | Asana | OAuth (IO) | Medium | Q1 |
| **P1** | Adobe Analytics | S2S | Medium | Q2 |
| **P1** | Slack | OAuth (Bot) | Low | Q1 |
| **P2** | SharePoint On-Prem | S2S + Bridge | Very High | Q2-Q3 |
| **P2** | Salesforce | OAuth | High | Q3 |
| **P2** | Adobe DAM | Hybrid | High | Q3-Q4 |
| **P3** | Quip | S2S | Low | Q2 |
| **P3** | LaunchPad | Custom | Medium | Q3 |

---

## 7. Trust Boundaries

### 7.1 Trust Boundary Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              TRUST BOUNDARY ARCHITECTURE                              │
│                                                                                       │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │                         CORPORATE NETWORK (Internal)                          │   │
│   │                                                                               │   │
│   │   ┌─────────────────────────────────────────────────────────────────────┐   │   │
│   │   │                    ON-PREMISES SYSTEMS                               │   │   │
│   │   │                                                                       │   │   │
│   │   │   ┌────────────────┐      ┌────────────────┐                        │   │   │
│   │   │   │   SharePoint   │      │   Active       │                        │   │   │
│   │   │   │   On-Prem      │      │   Directory    │                        │   │   │
│   │   │   │   Servers      │      │   (Identity)   │                        │   │   │
│   │   │   └───────┬────────┘      └────────────────┘                        │   │   │
│   │   │           │                                                          │   │   │
│   │   │           │ (Internal only)                                          │   │   │
│   │   │           ▼                                                          │   │   │
│   │   │   ┌────────────────┐                                                │   │   │
│   │   │   │  On-Prem       │                                                │   │   │
│   │   │   │  Knowledge     │◀─── mTLS ───────────────────┐                  │   │   │
│   │   │   │  Bridge Agent  │                              │                  │   │   │
│   │   │   └────────────────┘                              │                  │   │   │
│   │   │                                                    │                  │   │   │
│   │   └────────────────────────────────────────────────────┼─────────────────┘   │   │
│   │                                                        │                      │   │
│   │   ════════════════════════════════════════════════════╪══════════════════   │   │
│   │   ║                    DMZ / FIREWALL                  ║                     │   │
│   │   ════════════════════════════════════════════════════╪══════════════════   │   │
│   │                                                        │                      │   │
│   └────────────────────────────────────────────────────────┼──────────────────────┘   │
│                                                            │                          │
│   ┌────────────────────────────────────────────────────────┼──────────────────────┐   │
│   │                         AWS CLOUD (Semi-Trusted)       │                       │   │
│   │                                                        │                       │   │
│   │   ┌───────────────────────────────────────────────────┼───────────────────┐  │   │
│   │   │                    VPC (Private)                   │                    │  │   │
│   │   │                                                    │                    │  │   │
│   │   │   ┌───────────────┐      ┌───────────────┐        │                    │  │   │
│   │   │   │    Lambda     │      │   DynamoDB    │        │                    │  │   │
│   │   │   │   Functions   │      │   (State)     │        │                    │  │   │
│   │   │   └───────────────┘      └───────────────┘        │                    │  │   │
│   │   │           │                      │                 │                    │  │   │
│   │   │           ▼                      ▼                 │                    │  │   │
│   │   │   ┌─────────────────────────────────────────────┐ │                    │  │   │
│   │   │   │                 MCC HUB                      │◀┘                    │  │   │
│   │   │   │           (Integration Layer)               │                      │  │   │
│   │   │   │                                             │                      │  │   │
│   │   │   │  ┌─────────────┐  ┌─────────────────────┐  │                      │  │   │
│   │   │   │  │ Tool Gateway│  │ Token Vault         │  │                      │  │   │
│   │   │   │  │ (PEP)       │  │ (Secrets Manager)   │  │                      │  │   │
│   │   │   │  └──────┬──────┘  └─────────────────────┘  │                      │  │   │
│   │   │   │         │                                    │                      │  │   │
│   │   │   └─────────┼────────────────────────────────────┘                      │  │   │
│   │   │             │                                                            │  │   │
│   │   └─────────────┼────────────────────────────────────────────────────────────┘  │   │
│   │                 │                                                               │   │
│   │   ══════════════╪═══════════════════════════════════════════════════════════   │   │
│   │   ║         PUBLIC INTERNET BOUNDARY                                       ║   │   │
│   │   ══════════════╪═══════════════════════════════════════════════════════════   │   │
│   │                 │                                                               │   │
│   └─────────────────┼───────────────────────────────────────────────────────────────┘   │
│                     │                                                                   │
│   ┌─────────────────┼───────────────────────────────────────────────────────────────┐   │
│   │                 │            EXTERNAL SAAS (Untrusted)                           │   │
│   │                 │                                                                │   │
│   │   ┌─────────────┴──────────────┐                                                │   │
│   │   │                            │                                                │   │
│   │   ▼                            ▼                                                │   │
│   │   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                   │   │
│   │   │   Workfront    │  │     Asana      │  │  Salesforce    │                   │   │
│   │   │                │  │                │  │                │                   │   │
│   │   │   OAuth API    │  │   OAuth API    │  │   OAuth API    │                   │   │
│   │   └────────────────┘  └────────────────┘  └────────────────┘                   │   │
│   │                                                                                │   │
│   │   TRUST LEVEL: External - Always validate responses, sanitize inputs          │   │
│   └────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                       │
│   LEGEND:                                                                            │
│   ════ = Security boundary (firewall, WAF, etc.)                                    │
│   ──── = Encrypted connection (TLS/mTLS)                                            │
│   ◀─── = Direction of authentication/authorization                                  │
│                                                                                       │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Trust Principles

| Boundary | Trust Level | Principle | Implementation |
|----------|-------------|-----------|----------------|
| **On-Prem → Cloud** | Established via mTLS | Zero Trust | Certificate-based auth, outbound only |
| **Cloud → SaaS** | OAuth with scopes | Least Privilege | Minimal scopes, token rotation |
| **User → Platform** | Authenticated | Defense in Depth | Cognito + MFA + RBAC |
| **Agent → Tools** | Policy-enforced | Guardrails | PDP checks every action |

---

## 8. Implementation Architecture

### 8.1 Integration Hub Design

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    INTEGRATION HUB ARCHITECTURE                          │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    INTEGRATION HUB SERVICE                         │ │
│   │                                                                     │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │                  CONNECTOR REGISTRY                          │ │ │
│   │   │                                                               │ │ │
│   │   │   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │ │ │
│   │   │   │  Workfront   │ │    Asana     │ │  SharePoint  │  ...   │ │ │
│   │   │   │  Connector   │ │  Connector   │ │  Connector   │        │ │ │
│   │   │   │              │ │              │ │              │        │ │ │
│   │   │   │ - OAuth flow │ │ - OAuth flow │ │ - mTLS setup │        │ │ │
│   │   │   │ - API client │ │ - API client │ │ - CSOM client│        │ │ │
│   │   │   │ - Retry logic│ │ - Retry logic│ │ - Retry logic│        │ │ │
│   │   │   │ - Rate limit │ │ - Rate limit │ │ - Rate limit │        │ │ │
│   │   │   └──────────────┘ └──────────────┘ └──────────────┘        │ │ │
│   │   └─────────────────────────────────────────────────────────────┘ │ │
│   │                                                                     │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │                  CREDENTIAL MANAGER                          │ │ │
│   │   │                                                               │ │ │
│   │   │   ┌──────────────────────────────────────────────────────┐  │ │ │
│   │   │   │              AWS SECRETS MANAGER                      │  │ │ │
│   │   │   │                                                        │  │ │ │
│   │   │   │   /mcc/integrations/workfront/oauth                   │  │ │ │
│   │   │   │   /mcc/integrations/asana/oauth                       │  │ │ │
│   │   │   │   /mcc/integrations/sharepoint/cert                   │  │ │ │
│   │   │   │   /mcc/integrations/analytics/apikey                  │  │ │ │
│   │   │   │                                                        │  │ │ │
│   │   │   │   Rotation: Automatic (Lambda trigger)                │  │ │ │
│   │   │   │   Encryption: AWS KMS (CMK)                           │  │ │ │
│   │   │   └──────────────────────────────────────────────────────┘  │ │ │
│   │   │                                                               │ │ │
│   │   │   Token Refresh Service:                                     │ │ │
│   │   │   - EventBridge cron: rate(45 minutes)                      │ │ │
│   │   │   - Proactively refresh tokens expiring in 15 min           │ │ │
│   │   │   - Exponential backoff on failure                          │ │ │
│   │   │   - Alert to #ops-alerts on persistent failure              │ │ │
│   │   └─────────────────────────────────────────────────────────────┘ │ │
│   │                                                                     │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │                  REQUEST DISPATCHER                          │ │ │
│   │   │                                                               │ │ │
│   │   │   interface IntegrationRequest {                             │ │ │
│   │   │     targetSystem: 'workfront' | 'asana' | 'sharepoint';     │ │ │
│   │   │     operation: string;                                       │ │ │
│   │   │     payload: Record<string, unknown>;                        │ │ │
│   │   │     userContext: {                                           │ │ │
│   │   │       userId: string;      // MCC user ID                    │ │ │
│   │   │       tenantId: string;    // Org/BU                         │ │ │
│   │   │       roles: string[];     // RBAC roles                     │ │ │
│   │   │     };                                                       │ │ │
│   │   │     traceId: string;       // X-Ray trace                    │ │ │
│   │   │   }                                                          │ │ │
│   │   │                                                               │ │ │
│   │   │   Flow:                                                      │ │ │
│   │   │   1. Validate request schema                                 │ │ │
│   │   │   2. Check policy (PDP)                                      │ │ │
│   │   │   3. Get credentials for target system                       │ │ │
│   │   │   4. Invoke connector                                        │ │ │
│   │   │   5. Transform response                                      │ │ │
│   │   │   6. Audit log                                               │ │ │
│   │   │   7. Return result                                           │ │ │
│   │   └─────────────────────────────────────────────────────────────┘ │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Connector Interface

```typescript
// integration-hub/src/connectors/types.ts

/**
 * Base interface for all integration connectors.
 * Implements the Adapter pattern for uniform access to external systems.
 */
interface IIntegrationConnector {
  /** Unique identifier for this connector */
  readonly id: string;

  /** Human-readable name */
  readonly name: string;

  /** Authentication method used */
  readonly authMethod: 'oauth' | 's2s_apikey' | 's2s_mtls' | 's2s_sts';

  /** Health check endpoint */
  healthCheck(): Promise<HealthStatus>;

  /** Execute an operation on the target system */
  execute<T>(operation: Operation, context: ExecutionContext): Promise<Result<T>>;

  /** List available operations */
  listOperations(): Operation[];
}

interface ExecutionContext {
  /** User making the request (for audit) */
  user: UserContext;

  /** Tenant/org context */
  tenant: TenantContext;

  /** Distributed tracing ID */
  traceId: string;

  /** Request timeout in ms */
  timeoutMs: number;

  /** Retry configuration */
  retry: RetryConfig;
}

interface RetryConfig {
  maxAttempts: number;
  baseDelayMs: number;
  maxDelayMs: number;
  retryableErrors: string[]; // e.g., ['RATE_LIMITED', 'TIMEOUT', 'SERVICE_UNAVAILABLE']
}

/**
 * Example: Workfront Connector Implementation
 */
class WorkfrontConnector implements IIntegrationConnector {
  readonly id = 'workfront';
  readonly name = 'Adobe Workfront';
  readonly authMethod = 'oauth';

  private client: WorkfrontApiClient;
  private credentialManager: CredentialManager;

  async execute<T>(operation: Operation, context: ExecutionContext): Promise<Result<T>> {
    // 1. Get current OAuth token
    const token = await this.credentialManager.getToken(this.id, context.tenant.id);

    // 2. Check token validity, refresh if needed
    if (token.expiresAt < Date.now() + 60000) {
      await this.credentialManager.refreshToken(this.id, context.tenant.id);
    }

    // 3. Execute with retry
    return withRetry(
      () => this.client.execute(operation, token.accessToken),
      context.retry
    );
  }
}
```

---

## 9. Security Considerations

### 9.1 Credential Security Matrix

| Credential Type | Storage | Rotation | Access Control |
|-----------------|---------|----------|----------------|
| OAuth Tokens | Secrets Manager (encrypted) | On refresh (auto) | Lambda execution role |
| API Keys | Secrets Manager (encrypted) | 90 days (scheduled) | Lambda execution role |
| mTLS Certs | ACM + Secrets Manager | 365 days (scheduled) | Limited to bridge agent |
| Client Secrets | Secrets Manager (encrypted) | On demand | Admin only |

### 9.2 Token Security Best Practices

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TOKEN SECURITY BEST PRACTICES                         │
│                                                                          │
│   1. NEVER log tokens (even partially)                                  │
│      ❌ log.info(`Token: ${token.substring(0, 10)}...`)                 │
│      ✓  log.info('Token retrieved successfully', { tenantId })          │
│                                                                          │
│   2. ALWAYS use encrypted storage                                       │
│      - Secrets Manager with KMS CMK                                     │
│      - Enable automatic rotation                                        │
│      - Audit access via CloudTrail                                      │
│                                                                          │
│   3. ALWAYS use short-lived tokens where possible                       │
│      - Prefer STS AssumeRole for AWS services                           │
│      - Refresh OAuth tokens proactively                                 │
│      - Set minimum viable expiration                                    │
│                                                                          │
│   4. ALWAYS scope tokens to minimum required permissions                │
│      - Request only needed OAuth scopes                                 │
│      - Use resource-specific IAM policies                               │
│      - Avoid admin/wildcard permissions                                 │
│                                                                          │
│   5. ALWAYS validate token provenance                                   │
│      - Verify webhook signatures                                        │
│      - Validate JWT issuers                                             │
│      - Check token audience claims                                      │
│                                                                          │
│   6. IMPLEMENT token use monitoring                                     │
│      - Alert on unusual patterns                                        │
│      - Track token age distribution                                     │
│      - Monitor refresh failure rates                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.3 Audit Requirements

Every integration action MUST be logged with:

```json
{
  "timestamp": "2026-01-10T14:30:00.000Z",
  "traceId": "1-abc123-def456",
  "eventType": "integration.action",
  "targetSystem": "workfront",
  "operation": "create_project",
  "user": {
    "id": "user-uuid",
    "email": "sarah@company.com",
    "roles": ["campaign-manager"]
  },
  "tenant": {
    "id": "tenant-uuid",
    "name": "ACME Marketing"
  },
  "request": {
    "projectName": "Q1 Product Launch",
    "templateId": "template-123"
  },
  "response": {
    "success": true,
    "projectId": "WF-456",
    "latencyMs": 234
  },
  "authMethod": "oauth_integration_owner"
}
```

---

## 10. Operational Concerns

### 10.1 Monitoring Dashboard

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    INTEGRATION HEALTH DASHBOARD                          │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  SYSTEM STATUS                                                   │   │
│   │                                                                   │   │
│   │  Workfront    [████████████████████] 100% UP    Latency: 234ms  │   │
│   │  Asana        [████████████████████] 100% UP    Latency: 156ms  │   │
│   │  SharePoint   [████████████████░░░░]  85% UP    Latency: 1.2s   │   │
│   │  Analytics    [████████████████████] 100% UP    Latency: 89ms   │   │
│   │                                                                   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  TOKEN STATUS                                                    │   │
│   │                                                                   │   │
│   │  Workfront OAuth    Expires: 2026-01-10 15:30    [REFRESH OK]   │   │
│   │  Asana OAuth        Expires: 2026-01-10 16:00    [ACTIVE]       │   │
│   │  Analytics API Key  Rotated: 2025-12-15          [OK]           │   │
│   │  SharePoint Cert    Expires: 2026-06-01          [OK]           │   │
│   │                                                                   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  ERROR RATES (Last 24h)                                          │   │
│   │                                                                   │   │
│   │  Workfront     0.02% [▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁]                │   │
│   │  Asana         0.01% [▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁]                │   │
│   │  SharePoint    2.30% [▁▁▁▃▆█▆▃▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁]  ⚠️ Alert      │   │
│   │  Analytics     0.00% [▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁]                │   │
│   │                                                                   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  API CALL VOLUME (Last 24h)                                      │   │
│   │                                                                   │   │
│   │  Total Calls: 45,892                                             │   │
│   │  ├── Workfront: 28,456 (62%)                                    │   │
│   │  ├── Asana: 12,340 (27%)                                        │   │
│   │  ├── Analytics: 4,100 (9%)                                      │   │
│   │  └── SharePoint: 996 (2%)                                       │   │
│   │                                                                   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10.2 Incident Response Runbook

| Scenario | Detection | Response |
|----------|-----------|----------|
| **OAuth token refresh failure** | CloudWatch alarm on refresh Lambda errors | 1. Check Secrets Manager for token state 2. Verify target system status 3. Manual re-auth if needed |
| **Rate limiting from external API** | 429 response codes in logs | 1. Check current call volume 2. Implement exponential backoff 3. Contact vendor if persistent |
| **Webhook delivery failure** | Dead letter queue depth > 0 | 1. Check target system health 2. Replay failed webhooks 3. Investigate payload issues |
| **Credential exposure suspicion** | Security Hub finding | 1. Immediately rotate affected credentials 2. Audit recent usage 3. Notify security team |

---

## Summary

### Key Architectural Decisions

1. **Integration Owner Pattern** for OAuth-based systems (Workfront, Asana)
   - Reduces token management complexity from O(users × systems) to O(systems)
   - Platform-enforced RBAC provides fine-grained access control
   - Centralized audit trail

2. **S2S with STS/IAM** for AWS-native services
   - Leverage cloud-native identity
   - Short-lived credentials by default
   - No secret management overhead

3. **Hybrid approach for DAM**
   - S2S for read operations (bulk metadata retrieval)
   - OAuth for writes (asset attribution required)

4. **mTLS Bridge for On-Premises**
   - Certificate-based trust establishment
   - Outbound-only connections from corporate network
   - No inbound firewall rules needed

### Next Steps

1. Implement [Canonical Data Model](06-canonical-data-model.md) for cross-system data mapping
2. Design [Contract-First Agent Architecture](07-contract-first-agents.md) for safe AI operations
3. Build [Multi-Tenant Integration Hub](08-multi-tenancy-hub.md) for shared infrastructure
