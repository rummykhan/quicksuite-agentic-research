# Production-Grade Architecture Design

## Marketing Command Center - Connected Systems Platform

### Version: 1.0
### Date: December 29, 2025
### Author: Senior Principal Engineer / Solution Architect

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [High-Level Architecture Diagram](#2-high-level-architecture-diagram)
3. [Component Architecture](#3-component-architecture)
4. [Data Architecture](#4-data-architecture)
5. [Integration Architecture](#5-integration-architecture)
6. [Security Architecture](#6-security-architecture)
7. [Deployment Architecture](#7-deployment-architecture)
8. [Monitoring & Observability](#8-monitoring--observability)
9. [Disaster Recovery](#9-disaster-recovery)
10. [Technology Stack Summary](#10-technology-stack-summary)

---

## 1. Architecture Overview

### 1.1 Design Principles

| Principle | Implementation |
|-----------|----------------|
| **Event-Driven** | All system interactions via EventBridge; async by default |
| **Serverless-First** | Lambda, Step Functions, managed services |
| **API-First** | GraphQL (AppSync) for real-time, REST for integrations |
| **Multi-Agent** | Bedrock AgentCore supervisor pattern |
| **Zero Trust** | IAM policies, Cognito auth, encryption everywhere |
| **Observable** | Centralized logging, tracing, metrics |
| **Extensible** | MCP protocol for tool additions |

### 1.2 System Context

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          EXTERNAL SYSTEMS                               │
├──────────────┬──────────────┬──────────────┬──────────────┬────────────┤
│   Workfront  │    Asana     │  LaunchPad   │  Salesforce  │   Adobe    │
│   (Primary)  │  (Events)    │  (Launches)  │    (CRM)     │ Analytics  │
└──────┬───────┴──────┬───────┴──────┬───────┴──────┬───────┴─────┬──────┘
       │              │              │              │             │
       ▼              ▼              ▼              ▼             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│              MARKETING COMMAND CENTER (AWS Cloud)                       │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                     AI AGENT LAYER                                │  │
│  │   (Bedrock AgentCore + Supervisor Pattern)                       │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                   INTEGRATION LAYER                               │  │
│  │   (EventBridge + API Gateway + Lambda)                           │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                     DATA LAYER                                    │  │
│  │   (DynamoDB + Redshift + S3 + OpenSearch)                        │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                  PRESENTATION LAYER                               │  │
│  │   (AppSync + QuickSight + React UI)                              │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                           ┌───────────────┐
                           │   End Users   │
                           │  (Marketers)  │
                           └───────────────┘
```

---

## 2. High-Level Architecture Diagram

### 2.1 Full System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                        AWS CLOUD                                                     │
│ ┌─────────────────────────────────────────────────────────────────────────────────────────────────┐ │
│ │                                   VPC (10.0.0.0/16)                                              │ │
│ │ ┌─────────────────────────────────────────────────────────────────────────────────────────────┐ │ │
│ │ │                              PRIVATE SUBNET (10.0.1.0/24)                                    │ │ │
│ │ │                                                                                               │ │ │
│ │ │   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │ │ │
│ │ │   │                           AI AGENT LAYER                                             │   │ │ │
│ │ │   │                                                                                       │   │ │ │
│ │ │   │   ┌─────────────────────────────────────────────────────────────────────────────┐   │   │ │ │
│ │ │   │   │                    BEDROCK AGENTCORE                                         │   │   │ │ │
│ │ │   │   │                                                                               │   │   │ │ │
│ │ │   │   │   ┌───────────────────────────────────────────────────────────────────────┐ │   │   │ │ │
│ │ │   │   │   │                  SUPERVISOR AGENT                                      │ │   │   │ │ │
│ │ │   │   │   │   - Query routing    - Agent orchestration    - Response synthesis    │ │   │   │ │ │
│ │ │   │   │   └───────────────────────────────────────────────────────────────────────┘ │   │   │ │ │
│ │ │   │   │                        │                │                │                   │   │   │ │ │
│ │ │   │   │         ┌──────────────┼────────────────┼────────────────┼─────────────┐    │   │   │ │ │
│ │ │   │   │         ▼              ▼                ▼                ▼             │    │   │   │ │ │
│ │ │   │   │   ┌──────────┐  ┌───────────┐  ┌────────────┐  ┌────────────────┐     │    │   │   │ │ │
│ │ │   │   │   │ Strategy │  │Production │  │  Analytics │  │   Knowledge    │     │    │   │   │ │ │
│ │ │   │   │   │  Agent   │  │   Agent   │  │   Agent    │  │     Agent      │     │    │   │   │ │ │
│ │ │   │   │   └────┬─────┘  └─────┬─────┘  └──────┬─────┘  └───────┬────────┘     │    │   │   │ │ │
│ │ │   │   │        │              │               │                │              │    │   │   │ │ │
│ │ │   │   │        └──────────────┴───────────────┴────────────────┘              │    │   │   │ │ │
│ │ │   │   │                               │                                        │    │   │   │ │ │
│ │ │   │   │                               ▼                                        │    │   │   │ │ │
│ │ │   │   │   ┌───────────────────────────────────────────────────────────────┐   │    │   │   │ │ │
│ │ │   │   │   │                 AGENTCORE GATEWAY (MCP)                        │   │    │   │   │ │ │
│ │ │   │   │   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │   │    │   │   │ │ │
│ │ │   │   │   │  │Workfront │ │  Asana   │ │Salesforce│ │ Calendar │ ...      │   │    │   │   │ │ │
│ │ │   │   │   │  │   MCP    │ │   MCP    │ │   MCP    │ │   MCP    │          │   │    │   │   │ │ │
│ │ │   │   │   │  └──────────┘ └──────────┘ └──────────┘ └──────────┘          │   │    │   │   │ │ │
│ │ │   │   │   └───────────────────────────────────────────────────────────────┘   │    │   │   │ │ │
│ │ │   │   └───────────────────────────────────────────────────────────────────────┘    │   │ │ │
│ │ │   │                                                                                 │   │ │ │
│ │ │   │   ┌─────────────────────────────────────────────────────────────────────────┐  │   │ │ │
│ │ │   │   │                    KNOWLEDGE BASE LAYER                                  │  │   │ │ │
│ │ │   │   │                                                                           │  │   │ │ │
│ │ │   │   │   ┌─────────────────────────────────────────────────────────────────┐   │  │   │ │ │
│ │ │   │   │   │              AMAZON Q BUSINESS                                   │   │  │   │ │ │
│ │ │   │   │   │  - Campaign Playbooks    - Brand Guidelines                      │   │  │   │ │ │
│ │ │   │   │   │  - Event Standards       - Historical Performance                │   │  │   │ │ │
│ │ │   │   │   │  - Best Practices        - Process Documentation                 │   │  │   │ │ │
│ │ │   │   │   └─────────────────────────────────────────────────────────────────┘   │  │   │ │ │
│ │ │   │   │                                                                           │  │   │ │ │
│ │ │   │   │   ┌─────────────────────────────────────────────────────────────────┐   │  │   │ │ │
│ │ │   │   │   │              BEDROCK KNOWLEDGE BASE                              │   │  │   │ │ │
│ │ │   │   │   │  - Vector Store (OpenSearch Serverless)                          │   │  │   │ │ │
│ │ │   │   │   │  - Embeddings (Titan)                                            │   │  │   │ │ │
│ │ │   │   │   └─────────────────────────────────────────────────────────────────┘   │  │   │ │ │
│ │ │   │   └─────────────────────────────────────────────────────────────────────────┘  │   │ │ │
│ │ │   └─────────────────────────────────────────────────────────────────────────────────┘   │ │ │
│ │ │                                                                                          │ │ │
│ │ │   ┌──────────────────────────────────────────────────────────────────────────────────┐  │ │ │
│ │ │   │                         INTEGRATION LAYER                                         │  │ │ │
│ │ │   │                                                                                    │  │ │ │
│ │ │   │   ┌────────────────────────────────────────────────────────────────────────────┐ │  │ │ │
│ │ │   │   │                      AMAZON EVENTBRIDGE                                     │ │  │ │ │
│ │ │   │   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │ │  │ │ │
│ │ │   │   │   │campaign-events│  │workflow-events│ │ sync-events │  │notification  │  │ │  │ │ │
│ │ │   │   │   │     Bus      │  │     Bus      │  │     Bus      │  │events Bus    │  │ │  │ │ │
│ │ │   │   │   └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │ │  │ │ │
│ │ │   │   └────────────────────────────────────────────────────────────────────────────┘ │  │ │ │
│ │ │   │                                        │                                          │  │ │ │
│ │ │   │                           ┌────────────┼────────────┐                            │  │ │ │
│ │ │   │                           ▼            ▼            ▼                            │  │ │ │
│ │ │   │   ┌────────────────────────────────────────────────────────────────────────────┐ │  │ │ │
│ │ │   │   │                    AWS STEP FUNCTIONS                                       │ │  │ │ │
│ │ │   │   │   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐   │ │  │ │ │
│ │ │   │   │   │ CampaignCreation│  │ ApprovalWorkflow│  │ ProductionOrchestration │   │ │  │ │ │
│ │ │   │   │   │    Workflow     │  │    Workflow     │  │       Workflow          │   │ │  │ │ │
│ │ │   │   │   └─────────────────┘  └─────────────────┘  └─────────────────────────┘   │ │  │ │ │
│ │ │   │   │   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐   │ │  │ │ │
│ │ │   │   │   │  EventPlanning  │  │   SyncWorkflow  │  │   AnalyticsCollection   │   │ │  │ │ │
│ │ │   │   │   │    Workflow     │  │    Workflow     │  │       Workflow          │   │ │  │ │ │
│ │ │   │   │   └─────────────────┘  └─────────────────┘  └─────────────────────────┘   │ │  │ │ │
│ │ │   │   └────────────────────────────────────────────────────────────────────────────┘ │  │ │ │
│ │ │   │                                                                                    │  │ │ │
│ │ │   │   ┌────────────────────────────────────────────────────────────────────────────┐ │  │ │ │
│ │ │   │   │                         AWS LAMBDA                                          │ │  │ │ │
│ │ │   │   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │ │  │ │ │
│ │ │   │   │   │  Workfront   │  │    Asana     │  │  Salesforce  │  │   Analytics  │  │ │  │ │ │
│ │ │   │   │   │   Handler    │  │   Handler    │  │   Handler    │  │   Handler    │  │ │  │ │ │
│ │ │   │   │   └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │ │  │ │ │
│ │ │   │   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │ │  │ │ │
│ │ │   │   │   │  Webhook     │  │   Event      │  │   Transform  │  │  Notification│  │ │  │ │ │
│ │ │   │   │   │  Processor   │  │  Processor   │  │   Handler    │  │   Handler    │  │ │  │ │ │
│ │ │   │   │   └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │ │  │ │ │
│ │ │   │   └────────────────────────────────────────────────────────────────────────────┘ │  │ │ │
│ │ │   └──────────────────────────────────────────────────────────────────────────────────┘  │ │ │
│ │ │                                                                                          │ │ │
│ │ │   ┌──────────────────────────────────────────────────────────────────────────────────┐  │ │ │
│ │ │   │                           DATA LAYER                                              │  │ │ │
│ │ │   │                                                                                    │  │ │ │
│ │ │   │   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │  │ │ │
│ │ │   │   │   DynamoDB     │  │    Redshift    │  │ OpenSearch    │  │      S3        │ │  │ │ │
│ │ │   │   │                │  │   Serverless   │  │  Serverless   │  │                │ │  │ │ │
│ │ │   │   │ - Campaigns    │  │                │  │               │  │ - Documents    │ │  │ │ │
│ │ │   │   │ - Workflows    │  │ - Analytics    │  │ - Vector KB   │  │ - Assets       │ │  │ │ │
│ │ │   │   │ - State        │  │ - Aggregations │  │ - Search      │  │ - Archives     │ │  │ │ │
│ │ │   │   │ - Events       │  │ - Historical   │  │ - Logs        │  │ - Backups      │ │  │ │ │
│ │ │   │   └────────────────┘  └────────────────┘  └────────────────┘  └────────────────┘ │  │ │ │
│ │ │   │                                                                                    │  │ │ │
│ │ │   │   ┌────────────────────────────────────────────────────────────────────────────┐ │  │ │ │
│ │ │   │   │                    AMAZON TIMESTREAM                                        │ │  │ │ │
│ │ │   │   │   - Real-time metrics    - Campaign performance    - System metrics        │ │  │ │ │
│ │ │   │   └────────────────────────────────────────────────────────────────────────────┘ │  │ │ │
│ │ │   └──────────────────────────────────────────────────────────────────────────────────┘  │ │ │
│ │ └─────────────────────────────────────────────────────────────────────────────────────────┘ │ │
│ │ ┌─────────────────────────────────────────────────────────────────────────────────────────┐ │ │
│ │ │                              PUBLIC SUBNET (10.0.2.0/24)                                 │ │ │
│ │ │                                                                                          │ │ │
│ │ │   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌────────────────────┐   │ │ │
│ │ │   │  API Gateway   │  │   AppSync      │  │   CloudFront   │  │   ALB (WebSocket)  │   │ │ │
│ │ │   │  (REST APIs)   │  │  (GraphQL)     │  │   (Static)     │  │                    │   │ │ │
│ │ │   └────────────────┘  └────────────────┘  └────────────────┘  └────────────────────┘   │ │ │
│ │ └─────────────────────────────────────────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                  │
│ ┌──────────────────────────────────────────────────────────────────────────────────────────────┐│
│ │                              PRESENTATION LAYER                                               ││
│ │   ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────────────────────┐    ││
│ │   │   QuickSight       │  │   React Web App    │  │         Slack Integration         │    ││
│ │   │   Dashboards       │  │   (Chat + Dashboard)│  │       (Notifications)            │    ││
│ │   └────────────────────┘  └────────────────────┘  └────────────────────────────────────┘    ││
│ └──────────────────────────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Architecture

### 3.1 AI Agent Layer

#### 3.1.1 Supervisor Agent Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      SUPERVISOR AGENT                                │
│                                                                       │
│   Model: Claude 3.5 Sonnet (via Bedrock)                            │
│   Memory: AgentCore Memory (conversation context)                    │
│                                                                       │
│   ┌───────────────────────────────────────────────────────────────┐ │
│   │                     ROUTING LOGIC                              │ │
│   │                                                                 │ │
│   │   Input Classification:                                        │ │
│   │   ├── Strategy queries      → Strategy Agent                  │ │
│   │   ├── Production queries    → Production Agent                │ │
│   │   ├── Analytics queries     → Analytics Agent                 │ │
│   │   ├── Knowledge queries     → Knowledge Agent                 │ │
│   │   └── Multi-domain queries  → Orchestrated response           │ │
│   └───────────────────────────────────────────────────────────────┘ │
│                                                                       │
│   ┌───────────────────────────────────────────────────────────────┐ │
│   │                     RESPONSE SYNTHESIS                         │ │
│   │   - Combine multi-agent responses                              │ │
│   │   - Apply business rules                                       │ │
│   │   - Format for user context (role-aware)                       │ │
│   └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

#### 3.1.2 Specialized Agents

| Agent | Responsibilities | Tools | Model |
|-------|------------------|-------|-------|
| **Strategy Agent** | Campaign planning, audience targeting, channel recommendations | Knowledge Base, Campaign History, Audience Data API | Claude 3.5 Sonnet |
| **Production Agent** | Workflow management, task creation, status updates | Workfront MCP, Asana MCP, Calendar API | Claude 3.5 Sonnet |
| **Analytics Agent** | Performance reporting, trend analysis, recommendations | QuickSight API, Adobe Analytics connector, Redshift | Claude 3.5 Sonnet |
| **Knowledge Agent** | Documentation retrieval, playbook guidance, best practices | Q Business, Bedrock KB, SharePoint connector | Claude 3.5 Haiku (fast) |

#### 3.1.3 AgentCore Gateway Configuration

```yaml
# AgentCore Gateway - MCP Server Configuration
gateway:
  name: marketing-command-center-gateway
  description: "MCP Gateway for Marketing Tools"

  servers:
    - name: workfront-mcp
      type: rest-api
      spec: openapi
      endpoint: "${WORKFRONT_API_URL}"
      authentication:
        type: oauth2
        client_credentials:
          client_id: "${WORKFRONT_CLIENT_ID}"
          client_secret: "${WORKFRONT_CLIENT_SECRET}"
      tools:
        - name: create_project
          description: "Create a new project in Workfront"
        - name: create_task
          description: "Create a task within a project"
        - name: update_task_status
          description: "Update the status of a task"
        - name: get_project_status
          description: "Get current project status and metrics"
        - name: assign_resource
          description: "Assign a team member to a task"

    - name: asana-mcp
      type: hosted
      provider: aws-partner
      tools:
        - name: create_project
        - name: create_task
        - name: update_task
        - name: get_project
        - name: search_tasks

    - name: calendar-mcp
      type: rest-api
      spec: openapi
      endpoint: "${M365_GRAPH_API}"
      tools:
        - name: check_availability
        - name: create_event
        - name: get_upcoming_events

  policies:
    - name: pii-protection
      description: "Never include PII in external system fields"
      enforcement: deterministic

    - name: approval-required
      description: "Require human approval for budget commitments over $10,000"
      enforcement: deterministic
```

### 3.2 Knowledge Base Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      KNOWLEDGE BASE SYSTEM                               │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                    DATA SOURCES                                     │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │ │
│  │  │SharePoint│ │   Quip   │ │Confluence│ │    S3    │ │ Crawled  │ │ │
│  │  │  (M365)  │ │          │ │          │ │(Uploaded)│ │ Internal │ │ │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ │ │
│  └───────┼────────────┼────────────┼────────────┼────────────┼────────┘ │
│          │            │            │            │            │          │
│          ▼            ▼            ▼            ▼            ▼          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                    INGESTION PIPELINE                               │ │
│  │                                                                      │ │
│  │   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐           │ │
│  │   │  Connectors  │ → │  Processors  │ → │   Chunking   │           │ │
│  │   │  (40+ types) │   │  (Extract,   │   │  (Semantic)  │           │ │
│  │   │              │   │   Transform) │   │              │           │ │
│  │   └──────────────┘   └──────────────┘   └──────────────┘           │ │
│  │                                              │                       │ │
│  └──────────────────────────────────────────────┼───────────────────────┘ │
│                                                 │                          │
│                                                 ▼                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                    VECTOR STORAGE                                   │  │
│  │                                                                      │  │
│  │   ┌────────────────────┐      ┌────────────────────────────────┐   │  │
│  │   │  Titan Embeddings  │  →   │  OpenSearch Serverless         │   │  │
│  │   │  (Text + Image)    │      │  (Vector Store)                 │   │  │
│  │   │                    │      │  - 1536 dimensions              │   │  │
│  │   │                    │      │  - HNSW index                   │   │  │
│  │   └────────────────────┘      └────────────────────────────────┘   │  │
│  │                                                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │                    RETRIEVAL LAYER                                    │ │
│  │                                                                        │ │
│  │   ┌─────────────────────────────────────────────────────────────┐    │ │
│  │   │              AGENTIC RAG ENGINE                              │    │ │
│  │   │                                                               │    │ │
│  │   │   1. Query Analysis        → Understand intent, context      │    │ │
│  │   │   2. Tool Selection        → Choose retrieval strategy       │    │ │
│  │   │   3. Multi-Retrieval       → Parallel vector + keyword       │    │ │
│  │   │   4. Reranking             → Score and filter results        │    │ │
│  │   │   5. Synthesis             → Generate grounded response      │    │ │
│  │   │   6. Hallucination Check   → Verify against sources          │    │ │
│  │   └─────────────────────────────────────────────────────────────┘    │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘

KNOWLEDGE DOMAINS:
┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│ Campaign        │ Brand           │ Event           │ Production      │
│ Playbooks       │ Guidelines      │ Standards       │ Processes       │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ - Tier 1/2/3    │ - Visual ID     │ - Tier 1 events │ - Asset specs   │
│ - Channel specs │ - Tone/voice    │ - Tier 2 events │ - Review flows  │
│ - Audience      │ - Messaging     │ - Playbooks     │ - Approval      │
│ - Metrics       │ - Templates     │ - Checklists    │ - Handoffs      │
└─────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

### 3.3 Integration Layer

#### 3.3.1 EventBridge Event Buses

```yaml
# EventBridge Configuration
event_buses:
  - name: campaign-events
    description: "Campaign lifecycle events"
    event_types:
      - campaign.created
      - campaign.updated
      - campaign.status_changed
      - campaign.approved
      - campaign.launched
      - campaign.completed

  - name: workflow-events
    description: "Workflow and task events"
    event_types:
      - task.created
      - task.assigned
      - task.status_changed
      - task.completed
      - workflow.started
      - workflow.step_completed
      - workflow.completed
      - workflow.failed

  - name: sync-events
    description: "Cross-system synchronization events"
    event_types:
      - sync.workfront.inbound
      - sync.workfront.outbound
      - sync.asana.inbound
      - sync.asana.outbound
      - sync.conflict_detected
      - sync.resolved

  - name: notification-events
    description: "User notification events"
    event_types:
      - notification.slack
      - notification.email
      - notification.in_app
      - notification.digest
```

#### 3.3.2 Step Functions Workflow Definitions

**Campaign Creation Workflow**:

```json
{
  "Comment": "Campaign Creation and Setup Workflow",
  "StartAt": "ValidateInput",
  "States": {
    "ValidateInput": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:${region}:${account}:function:validate-campaign-input",
      "Next": "EnrichWithKnowledgeBase",
      "Catch": [{
        "ErrorEquals": ["ValidationError"],
        "Next": "HandleValidationError"
      }]
    },
    "EnrichWithKnowledgeBase": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:${region}:${account}:function:enrich-campaign-data",
      "Next": "DetermineWorkManagementSystem"
    },
    "DetermineWorkManagementSystem": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.campaignType",
          "StringEquals": "event",
          "Next": "CreateAsanaProject"
        },
        {
          "Variable": "$.campaignType",
          "StringEquals": "standard",
          "Next": "CreateWorkfrontProject"
        }
      ],
      "Default": "CreateWorkfrontProject"
    },
    "CreateWorkfrontProject": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:${region}:${account}:function:workfront-create-project",
      "Next": "CreateTaskStructure",
      "Retry": [{
        "ErrorEquals": ["ServiceException", "RateLimitError"],
        "IntervalSeconds": 2,
        "MaxAttempts": 3,
        "BackoffRate": 2.0
      }]
    },
    "CreateAsanaProject": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:${region}:${account}:function:asana-create-project",
      "Next": "CreateTaskStructure",
      "Retry": [{
        "ErrorEquals": ["ServiceException", "RateLimitError"],
        "IntervalSeconds": 2,
        "MaxAttempts": 3,
        "BackoffRate": 2.0
      }]
    },
    "CreateTaskStructure": {
      "Type": "Map",
      "ItemsPath": "$.tasks",
      "Iterator": {
        "StartAt": "CreateTask",
        "States": {
          "CreateTask": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:${region}:${account}:function:create-task",
            "End": true
          }
        }
      },
      "Next": "AssignResources"
    },
    "AssignResources": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:${region}:${account}:function:assign-resources",
      "Next": "SetupNotifications"
    },
    "SetupNotifications": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:${region}:${account}:function:setup-notifications",
      "Next": "StoreCampaignState"
    },
    "StoreCampaignState": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "campaigns",
        "Item": {
          "campaignId": {"S.$": "$.campaignId"},
          "status": {"S": "ACTIVE"},
          "createdAt": {"S.$": "$$.State.EnteredTime"},
          "workflowExecutionId": {"S.$": "$$.Execution.Id"}
        }
      },
      "Next": "PublishCampaignCreatedEvent"
    },
    "PublishCampaignCreatedEvent": {
      "Type": "Task",
      "Resource": "arn:aws:states:::events:putEvents",
      "Parameters": {
        "Entries": [{
          "Source": "marketing.campaigns",
          "EventBusName": "campaign-events",
          "DetailType": "campaign.created",
          "Detail.$": "States.JsonToString($)"
        }]
      },
      "Next": "NotifyStakeholders"
    },
    "NotifyStakeholders": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:${region}:${account}:function:notify-stakeholders",
      "End": true
    },
    "HandleValidationError": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:${region}:${account}:function:handle-validation-error",
      "End": true
    }
  }
}
```

### 3.4 Data Layer

#### 3.4.1 DynamoDB Tables

```yaml
tables:
  - name: campaigns
    description: "Campaign state and metadata"
    partition_key: campaignId (S)
    sort_key: null
    gsi:
      - name: status-index
        partition_key: status (S)
        sort_key: createdAt (S)
      - name: team-index
        partition_key: teamId (S)
        sort_key: createdAt (S)
    attributes:
      - campaignId: string (UUID)
      - name: string
      - status: string (DRAFT|ACTIVE|PAUSED|COMPLETED)
      - type: string (standard|event|partner)
      - tier: number (1|2|3)
      - teamId: string
      - workfrontProjectId: string
      - asanaProjectId: string
      - createdAt: string (ISO 8601)
      - updatedAt: string (ISO 8601)
      - launchDate: string (ISO 8601)
      - metadata: map

  - name: workflow-state
    description: "Workflow execution state"
    partition_key: workflowId (S)
    sort_key: stepId (S)
    ttl: expiresAt
    attributes:
      - workflowId: string
      - stepId: string
      - status: string
      - input: map
      - output: map
      - startedAt: string
      - completedAt: string
      - expiresAt: number (epoch)

  - name: sync-state
    description: "Cross-system sync state"
    partition_key: sourceSystem (S)
    sort_key: sourceId (S)
    gsi:
      - name: target-index
        partition_key: targetSystem (S)
        sort_key: targetId (S)
    attributes:
      - sourceSystem: string
      - sourceId: string
      - targetSystem: string
      - targetId: string
      - lastSyncedAt: string
      - syncVersion: number
      - status: string (SYNCED|PENDING|CONFLICT)

  - name: events-log
    description: "Event audit log"
    partition_key: eventDate (S)
    sort_key: eventId (S)
    ttl: expiresAt
    stream: NEW_AND_OLD_IMAGES
    attributes:
      - eventDate: string (YYYY-MM-DD)
      - eventId: string (ULID)
      - eventType: string
      - source: string
      - detail: map
      - expiresAt: number (epoch)
```

#### 3.4.2 Redshift Serverless Schema

```sql
-- Analytics Schema
CREATE SCHEMA analytics;

-- Campaign Performance Fact Table
CREATE TABLE analytics.fact_campaign_performance (
    campaign_id VARCHAR(36) NOT NULL,
    date_key INTEGER NOT NULL,
    channel VARCHAR(50) NOT NULL,
    impressions BIGINT DEFAULT 0,
    clicks BIGINT DEFAULT 0,
    conversions BIGINT DEFAULT 0,
    spend DECIMAL(15,2) DEFAULT 0,
    revenue DECIMAL(15,2) DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (date_key) REFERENCES analytics.dim_date(date_key)
) DISTKEY(campaign_id) SORTKEY(date_key);

-- Campaign Dimension
CREATE TABLE analytics.dim_campaign (
    campaign_id VARCHAR(36) PRIMARY KEY,
    campaign_name VARCHAR(500),
    campaign_type VARCHAR(50),
    tier INTEGER,
    team_id VARCHAR(36),
    team_name VARCHAR(200),
    launch_date DATE,
    end_date DATE,
    status VARCHAR(20),
    workfront_id VARCHAR(50),
    asana_id VARCHAR(50),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
) DISTSTYLE ALL;

-- Date Dimension
CREATE TABLE analytics.dim_date (
    date_key INTEGER PRIMARY KEY,
    full_date DATE,
    year INTEGER,
    quarter INTEGER,
    month INTEGER,
    week INTEGER,
    day_of_week INTEGER,
    is_weekend BOOLEAN,
    is_holiday BOOLEAN,
    fiscal_year INTEGER,
    fiscal_quarter INTEGER
) DISTSTYLE ALL;

-- Workflow Metrics Fact
CREATE TABLE analytics.fact_workflow_metrics (
    workflow_id VARCHAR(36) NOT NULL,
    campaign_id VARCHAR(36),
    workflow_type VARCHAR(50),
    date_key INTEGER NOT NULL,
    duration_seconds INTEGER,
    steps_completed INTEGER,
    steps_failed INTEGER,
    human_interventions INTEGER,
    status VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) DISTKEY(campaign_id) SORTKEY(date_key);

-- Materialized View: Campaign Summary
CREATE MATERIALIZED VIEW analytics.mv_campaign_summary AS
SELECT
    c.campaign_id,
    c.campaign_name,
    c.campaign_type,
    c.tier,
    c.team_name,
    c.status,
    SUM(p.impressions) as total_impressions,
    SUM(p.clicks) as total_clicks,
    SUM(p.conversions) as total_conversions,
    SUM(p.spend) as total_spend,
    SUM(p.revenue) as total_revenue,
    CASE WHEN SUM(p.impressions) > 0
         THEN (SUM(p.clicks)::FLOAT / SUM(p.impressions)) * 100
         ELSE 0 END as ctr,
    CASE WHEN SUM(p.spend) > 0
         THEN SUM(p.revenue) / SUM(p.spend)
         ELSE 0 END as roas
FROM analytics.dim_campaign c
LEFT JOIN analytics.fact_campaign_performance p ON c.campaign_id = p.campaign_id
GROUP BY 1,2,3,4,5,6;
```

---

## 4. Data Architecture

### 4.1 Data Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              DATA FLOW ARCHITECTURE                               │
│                                                                                   │
│  INGESTION                 PROCESSING              STORAGE           CONSUMPTION │
│                                                                                   │
│  ┌──────────┐             ┌───────────┐          ┌───────────┐    ┌───────────┐ │
│  │Workfront │──webhook───▶│  Lambda   │─────────▶│ DynamoDB  │───▶│  AppSync  │ │
│  │  Events  │             │ (Process) │          │(Real-time)│    │(GraphQL)  │ │
│  └──────────┘             └───────────┘          └───────────┘    └───────────┘ │
│                                │                                                  │
│  ┌──────────┐                  │                                                  │
│  │  Asana   │──webhook────────▶│                                                 │
│  │  Events  │                  │                                                  │
│  └──────────┘                  │                                                  │
│                                ▼                                                  │
│  ┌──────────┐             ┌───────────┐          ┌───────────┐    ┌───────────┐ │
│  │  Adobe   │──scheduler──│  Kinesis  │─────────▶│ Redshift  │───▶│QuickSight │ │
│  │Analytics │──(hourly)──▶│  Firehose │          │(Analytics)│    │(Dashboard)│ │
│  └──────────┘             └───────────┘          └───────────┘    └───────────┘ │
│                                                                                   │
│  ┌──────────┐             ┌───────────┐          ┌───────────┐    ┌───────────┐ │
│  │Documents │──S3 event──▶│  Lambda   │─────────▶│ OpenSearch│───▶│  Bedrock  │ │
│  │(S3/M365) │             │(Ingest KB)│          │  (Vector) │    │(Knowledge)│ │
│  └──────────┘             └───────────┘          └───────────┘    └───────────┘ │
│                                                                                   │
│  ┌──────────┐             ┌───────────┐          ┌───────────┐    ┌───────────┐ │
│  │  Agent   │──EventBrdg─▶│   Step    │─────────▶│Timestream │───▶│CloudWatch │ │
│  │ Actions  │             │ Functions │          │ (Metrics) │    │(Monitoring)│ │
│  └──────────┘             └───────────┘          └───────────┘    └───────────┘ │
│                                                                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Data Synchronization Strategy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BI-DIRECTIONAL SYNC ARCHITECTURE                      │
│                                                                          │
│   COMMAND CENTER (Source of Truth: Intent)                              │
│   WORKFRONT/ASANA (Source of Truth: Execution)                          │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    SYNC STATE MACHINE                            │   │
│   │                                                                   │   │
│   │   ┌─────────┐      ┌─────────┐      ┌─────────┐      ┌───────┐  │   │
│   │   │ SYNCED  │─────▶│ PENDING │─────▶│ SYNCING │─────▶│SYNCED │  │   │
│   │   └─────────┘      └─────────┘      └─────────┘      └───────┘  │   │
│   │       │                                   │                      │   │
│   │       │           ┌──────────┐            │                      │   │
│   │       └──────────▶│ CONFLICT │◀───────────┘                      │   │
│   │                   └──────────┘                                   │   │
│   │                        │                                         │   │
│   │                        ▼                                         │   │
│   │                   ┌──────────┐                                   │   │
│   │                   │ RESOLVED │                                   │   │
│   │                   └──────────┘                                   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   CONFLICT RESOLUTION RULES:                                            │
│   1. Last-write-wins for non-critical fields (description, notes)       │
│   2. Source-of-truth-wins for critical fields:                          │
│      - Status changes: Workfront/Asana wins                             │
│      - Strategy changes: Command Center wins                            │
│   3. Human escalation for structural conflicts (task deletion)          │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    SYNC VERSION TRACKING                         │   │
│   │                                                                   │   │
│   │   DynamoDB Record:                                               │   │
│   │   {                                                              │   │
│   │     "sourceSystem": "command-center",                            │   │
│   │     "sourceId": "campaign-123",                                  │   │
│   │     "targetSystem": "workfront",                                 │   │
│   │     "targetId": "WF-456",                                        │   │
│   │     "syncVersion": 15,                                           │   │
│   │     "lastSyncedAt": "2025-12-29T10:30:00Z",                      │   │
│   │     "status": "SYNCED",                                          │   │
│   │     "fieldVersions": {                                           │   │
│   │       "name": { "version": 15, "source": "command-center" },     │   │
│   │       "status": { "version": 14, "source": "workfront" },        │   │
│   │       "assignee": { "version": 12, "source": "workfront" }       │   │
│   │     }                                                            │   │
│   │   }                                                              │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Integration Architecture

### 5.1 Workfront Integration

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      WORKFRONT INTEGRATION LAYER                         │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    INBOUND (Workfront → AWS)                     │   │
│   │                                                                   │   │
│   │   Workfront                                                       │   │
│   │      │                                                            │   │
│   │      │ Event Subscription (webhooks)                              │   │
│   │      ▼                                                            │   │
│   │   ┌──────────────┐                                                │   │
│   │   │ API Gateway  │◀──── WAF (rate limiting, IP filtering)        │   │
│   │   │ (webhook     │                                                │   │
│   │   │  endpoint)   │                                                │   │
│   │   └──────┬───────┘                                                │   │
│   │          │                                                        │   │
│   │          ▼                                                        │   │
│   │   ┌──────────────┐      ┌──────────────┐                         │   │
│   │   │   Lambda     │─────▶│ EventBridge  │                         │   │
│   │   │ (validate,   │      │ (sync-events │                         │   │
│   │   │  transform)  │      │    bus)      │                         │   │
│   │   └──────────────┘      └──────────────┘                         │   │
│   │                                                                   │   │
│   │   Event Types Subscribed:                                        │   │
│   │   - PROJECT (CREATE, UPDATE, DELETE)                             │   │
│   │   - TASK (CREATE, UPDATE, DELETE, STATUS_CHANGE)                 │   │
│   │   - ISSUE (CREATE, UPDATE)                                       │   │
│   │   - DOCUMENT (UPLOAD, VERSION)                                   │   │
│   │   - NOTE (CREATE)                                                │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    OUTBOUND (AWS → Workfront)                    │   │
│   │                                                                   │   │
│   │   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │   │
│   │   │  Agent /     │─────▶│    Lambda    │─────▶│  Workfront   │  │   │
│   │   │ Step Func    │      │  (API calls) │      │   REST API   │  │   │
│   │   └──────────────┘      └──────────────┘      └──────────────┘  │   │
│   │                              │                                    │   │
│   │                              ▼                                    │   │
│   │                         ┌──────────────┐                         │   │
│   │                         │   Secrets    │ (OAuth tokens)          │   │
│   │                         │   Manager    │                         │   │
│   │                         └──────────────┘                         │   │
│   │                                                                   │   │
│   │   API Operations:                                                │   │
│   │   - POST /attask/api/v20.0/proj (Create Project)                │   │
│   │   - POST /attask/api/v20.0/task (Create Task)                   │   │
│   │   - PUT /attask/api/v20.0/task/{id} (Update Task)               │   │
│   │   - GET /attask/api/v20.0/proj/{id} (Get Project)               │   │
│   │   - POST /attask/api/v20.0/optask (Create Issue)                │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    MCP TOOL DEFINITIONS                          │   │
│   │                                                                   │   │
│   │   tools:                                                         │   │
│   │     - name: workfront_create_project                             │   │
│   │       description: Create a new marketing campaign project       │   │
│   │       parameters:                                                │   │
│   │         - name: required (string)                                │   │
│   │         - templateId: optional (string)                          │   │
│   │         - plannedStartDate: optional (ISO 8601)                  │   │
│   │         - portfolioId: optional (string)                         │   │
│   │         - customFields: optional (object)                        │   │
│   │                                                                   │   │
│   │     - name: workfront_create_task                                │   │
│   │       description: Create a task within a project                │   │
│   │       parameters:                                                │   │
│   │         - projectId: required (string)                           │   │
│   │         - name: required (string)                                │   │
│   │         - assignedToId: optional (string)                        │   │
│   │         - plannedCompletionDate: optional (ISO 8601)             │   │
│   │         - description: optional (string)                         │   │
│   │                                                                   │   │
│   │     - name: workfront_update_status                              │   │
│   │       description: Update task or project status                 │   │
│   │       parameters:                                                │   │
│   │         - objectType: required (task|project)                    │   │
│   │         - objectId: required (string)                            │   │
│   │         - status: required (string)                              │   │
│   │                                                                   │   │
│   │     - name: workfront_get_capacity                               │   │
│   │       description: Get team capacity and availability            │   │
│   │       parameters:                                                │   │
│   │         - teamId: required (string)                              │   │
│   │         - startDate: required (ISO 8601)                         │   │
│   │         - endDate: required (ISO 8601)                           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Asana Integration

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        ASANA INTEGRATION LAYER                           │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │              USING AGENTCORE GATEWAY MCP                         │   │
│   │                                                                   │   │
│   │   AgentCore Gateway already provides hosted Asana MCP server     │   │
│   │                                                                   │   │
│   │   Available Tools (Pre-built):                                   │   │
│   │   - asana_create_project                                         │   │
│   │   - asana_create_task                                            │   │
│   │   - asana_update_task                                            │   │
│   │   - asana_get_project                                            │   │
│   │   - asana_search_tasks                                           │   │
│   │   - asana_add_comment                                            │   │
│   │   - asana_get_user_tasks                                         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │              WEBHOOK SUBSCRIPTION (Inbound)                      │   │
│   │                                                                   │   │
│   │   Asana Webhooks → API Gateway → Lambda → EventBridge            │   │
│   │                                                                   │   │
│   │   Subscribed Resources:                                          │   │
│   │   - Projects (created, changed, deleted)                         │   │
│   │   - Tasks (added, changed, removed, undeleted)                   │   │
│   │   - Stories (added) - for comments                               │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Security Architecture

### 6.1 Identity and Access Management

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SECURITY ARCHITECTURE                             │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    IDENTITY LAYER                                │   │
│   │                                                                   │   │
│   │   ┌───────────────────────────────────────────────────────────┐ │   │
│   │   │                   AMAZON COGNITO                           │ │   │
│   │   │                                                             │ │   │
│   │   │   User Pool: marketing-command-center-users                │ │   │
│   │   │                                                             │ │   │
│   │   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │ │   │
│   │   │   │   SAML      │  │   OIDC      │  │  Username/  │       │ │   │
│   │   │   │   (Corp     │  │   (SSO)     │  │  Password   │       │ │   │
│   │   │   │    IdP)     │  │             │  │  (Backup)   │       │ │   │
│   │   │   └─────────────┘  └─────────────┘  └─────────────┘       │ │   │
│   │   │                                                             │ │   │
│   │   │   Groups:                                                  │ │   │
│   │   │   - marketing-leadership (full dashboard access)           │ │   │
│   │   │   - campaign-managers (campaign CRUD)                      │ │   │
│   │   │   - production-team (production workflows)                 │ │   │
│   │   │   - event-marketers (Asana + events)                       │ │   │
│   │   │   - readonly-users (dashboard view only)                   │ │   │
│   │   │                                                             │ │   │
│   │   │   Custom Attributes:                                       │ │   │
│   │   │   - custom:teamId (string)                                 │ │   │
│   │   │   - custom:department (string)                             │ │   │
│   │   │   - custom:workfrontUserId (string)                        │ │   │
│   │   │   - custom:asanaUserId (string)                            │ │   │
│   │   └───────────────────────────────────────────────────────────┘ │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    AUTHORIZATION LAYER                           │   │
│   │                                                                   │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │              API GATEWAY AUTHORIZER                      │   │   │
│   │   │                                                           │   │   │
│   │   │   Type: Lambda Authorizer (Request-based)                │   │   │
│   │   │                                                           │   │   │
│   │   │   Logic:                                                  │   │   │
│   │   │   1. Validate JWT signature (aws-jwt-verify)             │   │   │
│   │   │   2. Extract user groups from token                      │   │   │
│   │   │   3. Extract teamId custom attribute                     │   │   │
│   │   │   4. Generate IAM policy based on groups + team          │   │   │
│   │   │   5. Return context (userId, teamId, permissions)        │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │                                                                   │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │              APPSYNC AUTHORIZATION                       │   │   │
│   │   │                                                           │   │   │
│   │   │   Primary: Cognito User Pools                            │   │   │
│   │   │   Field-level: @auth directives                          │   │   │
│   │   │                                                           │   │   │
│   │   │   type Campaign @auth(rules: [                           │   │   │
│   │   │     { allow: groups, groups: ["marketing-leadership"] }  │   │   │
│   │   │     { allow: owner, ownerField: "teamId",                │   │   │
│   │   │       identityClaim: "custom:teamId" }                   │   │   │
│   │   │   ])                                                      │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    DATA PROTECTION                               │   │
│   │                                                                   │   │
│   │   Encryption at Rest:                                            │   │
│   │   - DynamoDB: AWS managed CMK (aws/dynamodb)                    │   │
│   │   - S3: SSE-S3 (AES-256)                                        │   │
│   │   - Redshift: AWS managed key                                   │   │
│   │   - OpenSearch: AWS managed key                                 │   │
│   │                                                                   │   │
│   │   Encryption in Transit:                                         │   │
│   │   - TLS 1.2+ for all API endpoints                              │   │
│   │   - HTTPS enforced via CloudFront                               │   │
│   │   - VPC endpoints for AWS service communication                  │   │
│   │                                                                   │   │
│   │   Secrets Management:                                            │   │
│   │   - AWS Secrets Manager for API keys/tokens                     │   │
│   │   - Automatic rotation enabled (30-day cycle)                   │   │
│   │   - Lambda extensions for secret caching                        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    NETWORK SECURITY                              │   │
│   │                                                                   │   │
│   │   ┌───────────────────────────────────────────────────────────┐ │   │
│   │   │                    VPC DESIGN                              │ │   │
│   │   │                                                             │ │   │
│   │   │   VPC: 10.0.0.0/16                                         │ │   │
│   │   │   ├── Public Subnet: 10.0.1.0/24 (NAT Gateway, ALB)       │ │   │
│   │   │   ├── Private Subnet A: 10.0.2.0/24 (Lambda, compute)     │ │   │
│   │   │   ├── Private Subnet B: 10.0.3.0/24 (Lambda, compute)     │ │   │
│   │   │   └── Isolated Subnet: 10.0.4.0/24 (RDS, if needed)       │ │   │
│   │   │                                                             │ │   │
│   │   │   VPC Endpoints:                                           │ │   │
│   │   │   - com.amazonaws.{region}.dynamodb (Gateway)              │ │   │
│   │   │   - com.amazonaws.{region}.s3 (Gateway)                    │ │   │
│   │   │   - com.amazonaws.{region}.secretsmanager (Interface)      │ │   │
│   │   │   - com.amazonaws.{region}.bedrock-runtime (Interface)     │ │   │
│   │   │   - com.amazonaws.{region}.events (Interface)              │ │   │
│   │   └───────────────────────────────────────────────────────────┘ │   │
│   │                                                                   │   │
│   │   ┌───────────────────────────────────────────────────────────┐ │   │
│   │   │                    WAF RULES                               │ │   │
│   │   │                                                             │ │   │
│   │   │   - AWS Managed Rules (Common, SQLi, XSS)                  │ │   │
│   │   │   - Rate limiting: 2000 req/5min per IP                    │ │   │
│   │   │   - Geo-blocking: Allow only corporate regions             │ │   │
│   │   │   - IP allowlist for Workfront webhook IPs                 │ │   │
│   │   └───────────────────────────────────────────────────────────┘ │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Agent Governance (AgentCore Policy)

```yaml
# AgentCore Governance Policies
policies:
  - name: data-protection
    rules:
      - description: "Never expose PII in responses"
        action: BLOCK
        condition: "response contains SSN, credit_card, or personal_address"

      - description: "Mask email addresses in logs"
        action: TRANSFORM
        condition: "log entry contains email pattern"
        transformation: "mask_email"

  - name: budget-approval
    rules:
      - description: "Require approval for budget > $10,000"
        action: REQUIRE_APPROVAL
        condition: "action == 'create_campaign' AND budget > 10000"
        approvers: ["marketing-leadership"]

      - description: "Block budget > $100,000 without VP approval"
        action: BLOCK
        condition: "action == 'create_campaign' AND budget > 100000"
        message: "VP approval required for campaigns over $100,000"

  - name: system-boundaries
    rules:
      - description: "Read-only access to production systems"
        action: ALLOW_READ_ONLY
        condition: "target_system IN ['salesforce', 'adobe_analytics']"

      - description: "No deletion of Workfront projects"
        action: BLOCK
        condition: "action == 'delete' AND target_system == 'workfront' AND object_type == 'project'"
        message: "Project deletion must be done manually in Workfront"

  - name: rate-limiting
    rules:
      - description: "Limit API calls per minute"
        action: RATE_LIMIT
        limits:
          workfront: 100/minute
          asana: 150/minute
          salesforce: 50/minute
```

---

## 7. Deployment Architecture

### 7.1 Infrastructure as Code

```yaml
# AWS CDK Stack Structure
stacks:
  - name: NetworkStack
    resources:
      - VPC with public/private subnets
      - NAT Gateways
      - VPC Endpoints
      - Security Groups

  - name: DatabaseStack
    depends_on: [NetworkStack]
    resources:
      - DynamoDB Tables
      - Redshift Serverless Namespace/Workgroup
      - OpenSearch Serverless Collection

  - name: IntegrationStack
    depends_on: [NetworkStack, DatabaseStack]
    resources:
      - EventBridge Event Buses
      - API Gateway (REST)
      - Lambda Functions (integration)
      - Step Functions State Machines
      - Secrets Manager Secrets

  - name: AIStack
    depends_on: [IntegrationStack]
    resources:
      - Bedrock Agent definitions
      - AgentCore configurations
      - Q Business application
      - Knowledge Base configurations

  - name: PresentationStack
    depends_on: [AIStack]
    resources:
      - AppSync GraphQL API
      - CloudFront Distribution
      - Cognito User Pool
      - QuickSight resources (via console/API)

  - name: ObservabilityStack
    depends_on: [All]
    resources:
      - CloudWatch Dashboards
      - CloudWatch Alarms
      - X-Ray configurations
      - Log Groups
```

### 7.2 CI/CD Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          CI/CD PIPELINE                                  │
│                                                                          │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                │
│   │   GitHub    │───▶│  CodeBuild  │───▶│ CodePipeline│                │
│   │   (Source)  │    │   (Build)   │    │  (Deploy)   │                │
│   └─────────────┘    └─────────────┘    └──────┬──────┘                │
│                                                 │                        │
│         ┌───────────────────┬─────────────────┬┴───────────────┐       │
│         ▼                   ▼                 ▼                ▼       │
│   ┌───────────┐       ┌───────────┐     ┌───────────┐   ┌───────────┐ │
│   │    DEV    │──────▶│  STAGING  │────▶│   PROD    │   │   DR      │ │
│   │  Account  │ auto  │  Account  │manual│  Account  │   │  Account  │ │
│   └───────────┘       └───────────┘     └───────────┘   └───────────┘ │
│                                                                          │
│   Pipeline Stages:                                                      │
│   1. Source: GitHub webhook trigger                                     │
│   2. Build:                                                             │
│      - npm install / pip install                                        │
│      - CDK synth                                                        │
│      - Unit tests                                                       │
│      - Security scan (Checkov, cfn-nag)                                │
│   3. Deploy DEV: Automatic                                              │
│   4. Integration Tests: API tests against DEV                           │
│   5. Deploy STAGING: Automatic                                          │
│   6. UAT: Manual approval gate                                          │
│   7. Deploy PROD: Blue/Green or Canary                                  │
│   8. Smoke Tests: Automated post-deploy verification                    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.3 Environment Configuration

```yaml
# Environment-specific configurations
environments:
  dev:
    account_id: "111111111111"
    region: "us-west-2"
    bedrock_model: "anthropic.claude-3-haiku-20240307-v1:0"  # Cost-optimized
    dynamodb_billing: "PAY_PER_REQUEST"
    log_retention_days: 7
    workfront_env: "sandbox"

  staging:
    account_id: "222222222222"
    region: "us-west-2"
    bedrock_model: "anthropic.claude-3-5-sonnet-20241022-v2:0"
    dynamodb_billing: "PAY_PER_REQUEST"
    log_retention_days: 30
    workfront_env: "preview"

  prod:
    account_id: "333333333333"
    region: "us-west-2"
    bedrock_model: "anthropic.claude-3-5-sonnet-20241022-v2:0"
    dynamodb_billing: "PROVISIONED"  # Cost predictability
    log_retention_days: 365
    workfront_env: "production"

  dr:
    account_id: "333333333333"
    region: "us-east-1"
    # Mirrors prod, activated on failover
```

---

## 8. Monitoring & Observability

### 8.1 Metrics and Dashboards

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY ARCHITECTURE                            │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    CLOUDWATCH METRICS                            │   │
│   │                                                                   │   │
│   │   Business Metrics:                                              │   │
│   │   - Campaigns created per day                                    │   │
│   │   - Agent conversations per hour                                 │   │
│   │   - Workflow completion rate                                     │   │
│   │   - Average campaign creation time                               │   │
│   │   - Knowledge base query accuracy                                │   │
│   │                                                                   │   │
│   │   Technical Metrics:                                             │   │
│   │   - Lambda invocation count, duration, errors                    │   │
│   │   - Step Functions execution success/failure                     │   │
│   │   - API Gateway latency (p50, p95, p99)                         │   │
│   │   - EventBridge event delivery rate                              │   │
│   │   - Bedrock token usage and latency                             │   │
│   │   - DynamoDB consumed capacity                                   │   │
│   │                                                                   │   │
│   │   Integration Metrics:                                           │   │
│   │   - Workfront API call success rate                             │   │
│   │   - Asana sync latency                                          │   │
│   │   - Sync conflict rate                                          │   │
│   │   - Webhook delivery failures                                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    ALARMS                                        │   │
│   │                                                                   │   │
│   │   Critical (PagerDuty):                                          │   │
│   │   - Lambda error rate > 5% for 5 minutes                        │   │
│   │   - API Gateway 5xx > 1% for 5 minutes                          │   │
│   │   - Workflow failure rate > 10%                                  │   │
│   │   - Workfront integration down > 10 minutes                     │   │
│   │                                                                   │   │
│   │   Warning (Slack):                                               │   │
│   │   - Lambda duration p95 > 10 seconds                            │   │
│   │   - DynamoDB throttling events                                   │   │
│   │   - Bedrock latency p95 > 30 seconds                            │   │
│   │   - Sync conflict rate > 5%                                      │   │
│   │                                                                   │   │
│   │   Info (Dashboard):                                              │   │
│   │   - Daily campaign count below average                          │   │
│   │   - Knowledge base content staleness                            │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    DISTRIBUTED TRACING (X-Ray)                   │   │
│   │                                                                   │   │
│   │   Trace Groups:                                                  │   │
│   │   - agent-conversations (full conversation flow)                │   │
│   │   - campaign-creation (end-to-end workflow)                     │   │
│   │   - workfront-sync (integration traces)                         │   │
│   │                                                                   │   │
│   │   Custom Annotations:                                            │   │
│   │   - campaignId                                                   │   │
│   │   - userId                                                       │   │
│   │   - workflowType                                                 │   │
│   │   - agentType                                                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    LOGGING                                       │   │
│   │                                                                   │   │
│   │   Log Groups:                                                    │   │
│   │   - /aws/lambda/mcc-* (Lambda functions)                        │   │
│   │   - /aws/appsync/mcc-api (GraphQL resolvers)                    │   │
│   │   - /aws/apigateway/mcc-rest (REST API)                         │   │
│   │   - /aws/stepfunctions/mcc-workflows (State machines)           │   │
│   │   - /mcc/agents/* (Agent conversations - separate for privacy)  │   │
│   │                                                                   │   │
│   │   Structured Logging Format (JSON):                              │   │
│   │   {                                                              │   │
│   │     "timestamp": "ISO 8601",                                    │   │
│   │     "level": "INFO|WARN|ERROR",                                 │   │
│   │     "service": "campaign-service",                              │   │
│   │     "traceId": "X-Ray trace ID",                                │   │
│   │     "message": "Human readable",                                │   │
│   │     "context": { ... }                                          │   │
│   │   }                                                              │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Disaster Recovery

### 9.1 DR Strategy

| Component | RPO | RTO | Strategy |
|-----------|-----|-----|----------|
| DynamoDB | 0 | 1 hour | Global Tables (multi-region) |
| S3 | 0 | 1 hour | Cross-region replication |
| Redshift | 4 hours | 4 hours | Automated snapshots, restore |
| OpenSearch | 24 hours | 4 hours | Manual snapshot restore |
| Step Functions | N/A | 1 hour | Stateless, redeploy |
| Lambda | N/A | 30 min | Redeploy from CDK |
| Secrets | 0 | 1 hour | Multi-region secrets |

### 9.2 Failover Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MULTI-REGION DR ARCHITECTURE                          │
│                                                                          │
│   ┌────────────────────────────┐     ┌────────────────────────────┐    │
│   │      US-WEST-2 (Primary)   │     │      US-EAST-1 (DR)        │    │
│   │                            │     │                            │    │
│   │   ┌──────────────────┐    │     │   ┌──────────────────┐    │    │
│   │   │   Route 53       │◄───┼──┬──┼──▶│   Route 53       │    │    │
│   │   │   (Active)       │    │  │  │   │   (Standby)      │    │    │
│   │   └──────────────────┘    │  │  │   └──────────────────┘    │    │
│   │                            │  │  │                            │    │
│   │   ┌──────────────────┐    │  │  │   ┌──────────────────┐    │    │
│   │   │   CloudFront     │    │  │  │   │   CloudFront     │    │    │
│   │   │   (Primary)      │    │  │  │   │   (Standby)      │    │    │
│   │   └──────────────────┘    │  │  │   └──────────────────┘    │    │
│   │                            │  │  │                            │    │
│   │   ┌──────────────────┐    │  │  │   ┌──────────────────┐    │    │
│   │   │   DynamoDB       │◄───┼──┴──┼──▶│   DynamoDB       │    │    │
│   │   │   Global Table   │  sync    │   │   Global Table   │    │    │
│   │   └──────────────────┘    │     │   └──────────────────┘    │    │
│   │                            │     │                            │    │
│   │   ┌──────────────────┐    │     │   ┌──────────────────┐    │    │
│   │   │   S3 (Source)    │────┼─────┼──▶│   S3 (Replica)   │    │    │
│   │   │                  │  CRR     │   │                  │    │    │
│   │   └──────────────────┘    │     │   └──────────────────┘    │    │
│   │                            │     │                            │    │
│   └────────────────────────────┘     └────────────────────────────┘    │
│                                                                          │
│   Failover Process:                                                     │
│   1. Health check fails on primary region                               │
│   2. Route 53 automatically routes to DR                                │
│   3. DR region Lambda functions activate                                │
│   4. DynamoDB Global Table serves traffic                               │
│   5. Ops team notified, begins investigation                            │
│   6. Manual failback when primary restored                              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Technology Stack Summary

### 10.1 Core AWS Services

| Layer | Service | Purpose |
|-------|---------|---------|
| **AI/ML** | Amazon Bedrock | Foundation models (Claude) |
| | Bedrock AgentCore | Agent orchestration, MCP gateway |
| | Amazon Q Business | Enterprise RAG, knowledge base |
| **Compute** | AWS Lambda | Event handlers, integrations |
| | AWS Step Functions | Workflow orchestration |
| **Integration** | Amazon EventBridge | Event routing |
| | Amazon API Gateway | REST APIs, webhooks |
| | AWS AppSync | GraphQL, real-time subscriptions |
| **Data** | Amazon DynamoDB | Operational data, state |
| | Amazon Redshift Serverless | Analytics warehouse |
| | Amazon OpenSearch Serverless | Vector search, logs |
| | Amazon S3 | Documents, assets |
| | Amazon Timestream | Time-series metrics |
| **Security** | Amazon Cognito | Authentication |
| | AWS WAF | Web application firewall |
| | AWS Secrets Manager | Credentials management |
| **Observability** | Amazon CloudWatch | Metrics, logs, alarms |
| | AWS X-Ray | Distributed tracing |
| **Presentation** | Amazon QuickSight | BI dashboards |
| | Amazon CloudFront | CDN |

### 10.2 External Integrations

| System | Integration Method | Priority |
|--------|-------------------|----------|
| Adobe Workfront | REST API + Webhooks | P0 |
| Asana | MCP (AgentCore Gateway) | P1 |
| Microsoft 365 | Graph API + MCP | P1 |
| Salesforce | MCP connector | P2 |
| Adobe Analytics | REST API | P1 |
| Slack | Webhooks + Bot | P1 |
| LaunchPad | Custom API | P2 |

### 10.3 Development Stack

| Category | Technology |
|----------|------------|
| IaC | AWS CDK (TypeScript) |
| Backend | Python 3.12 (Lambda), TypeScript |
| Frontend | React 18, TypeScript |
| API | GraphQL (AppSync), REST (API Gateway) |
| Testing | pytest, Jest, Playwright |
| CI/CD | GitHub Actions, AWS CodePipeline |

---

## Appendix A: API Schema (GraphQL)

```graphql
type Query {
  # Campaigns
  getCampaign(id: ID!): Campaign
  listCampaigns(filter: CampaignFilterInput, limit: Int, nextToken: String): CampaignConnection

  # Workflows
  getWorkflowStatus(workflowId: ID!): WorkflowStatus

  # Dashboard
  getDashboardMetrics(teamId: ID, dateRange: DateRangeInput!): DashboardMetrics

  # Knowledge
  searchKnowledgeBase(query: String!, limit: Int): [KnowledgeResult]
}

type Mutation {
  # Agent interactions
  sendAgentMessage(conversationId: ID, message: String!): AgentResponse

  # Campaigns
  createCampaign(input: CreateCampaignInput!): Campaign
  updateCampaign(id: ID!, input: UpdateCampaignInput!): Campaign

  # Workflows
  triggerWorkflow(campaignId: ID!, workflowType: WorkflowType!): WorkflowExecution
  approveWorkflowStep(workflowId: ID!, stepId: ID!, approved: Boolean!, notes: String): WorkflowStatus
}

type Subscription {
  # Real-time updates
  onCampaignUpdated(teamId: ID): Campaign
  onWorkflowStepCompleted(workflowId: ID!): WorkflowStep
  onAgentResponse(conversationId: ID!): AgentResponse
  onDashboardMetricsUpdated(teamId: ID): DashboardMetrics
}

type Campaign {
  id: ID!
  name: String!
  type: CampaignType!
  tier: Int!
  status: CampaignStatus!
  teamId: ID!
  teamName: String
  workfrontProjectId: String
  asanaProjectId: String
  launchDate: AWSDateTime
  createdAt: AWSDateTime!
  updatedAt: AWSDateTime!
  metrics: CampaignMetrics
  tasks: [Task]
}

type AgentResponse {
  conversationId: ID!
  messageId: ID!
  content: String!
  actions: [AgentAction]
  sources: [KnowledgeSource]
  timestamp: AWSDateTime!
}

# ... additional types
```

---

## Appendix B: Cost Estimation (Production - Monthly)

| Service | Configuration | Est. Monthly Cost |
|---------|---------------|-------------------|
| Bedrock (Claude 3.5 Sonnet) | 10M input + 5M output tokens | $4,500 |
| Lambda | 5M invocations, 512MB, 2s avg | $150 |
| Step Functions | 500K state transitions | $125 |
| DynamoDB | 50GB storage, 100 WCU, 500 RCU | $200 |
| EventBridge | 10M events | $10 |
| API Gateway | 5M requests | $175 |
| AppSync | 2M requests + subscriptions | $200 |
| Redshift Serverless | 128 RPU, 4 hrs/day | $1,500 |
| OpenSearch Serverless | 2 OCU | $700 |
| Q Business | 100 users | $4,000 |
| CloudFront | 100GB transfer | $100 |
| S3 | 500GB storage | $15 |
| Secrets Manager | 20 secrets | $8 |
| CloudWatch | Logs + metrics | $200 |
| **Total Estimated** | | **~$11,900/month** |

*Note: Actual costs vary based on usage patterns. Consider Reserved Capacity for predictable workloads.*
