# On-Premises Integration Strategy

## Connecting SharePoint On-Prem to the Cloud-Based Integration Hub

### Version: 2.0
### Date: January 10, 2026
### Author: Principal Engineer / Solution Architect

---

## Table of Contents

1. [The Challenge](#1-the-challenge)
2. [Architecture Patterns](#2-architecture-patterns)
3. [Knowledge Bridge Design](#3-knowledge-bridge-design)
4. [Security Model](#4-security-model)
5. [Data Synchronization](#5-data-synchronization)
6. [Implementation Approach](#6-implementation-approach)
7. [Operational Considerations](#7-operational-considerations)
8. [Migration Path to Cloud](#8-migration-path-to-cloud)

---

## 1. The Challenge

### 1.1 SharePoint On-Premises Reality

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ON-PREM INTEGRATION CHALLENGE                         │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    CORPORATE NETWORK                               │ │
│   │                                                                     │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │                 SHAREPOINT ON-PREM                           │ │ │
│   │   │                                                               │ │ │
│   │   │   • Server 2019 / SharePoint 2019                            │ │ │
│   │   │   • Active Directory authentication                          │ │ │
│   │   │   • No external API access                                   │ │ │
│   │   │   • Firewall blocks inbound connections                      │ │ │
│   │   │                                                               │ │ │
│   │   │   Contains:                                                  │ │ │
│   │   │   ├── Marketing playbooks                                    │ │ │
│   │   │   ├── Campaign templates                                     │ │ │
│   │   │   ├── Brand guidelines                                       │ │ │
│   │   │   ├── Process documentation                                  │ │ │
│   │   │   └── Historical campaign data                               │ │ │
│   │   │                                                               │ │ │
│   │   └─────────────────────────────────────────────────────────────┘ │ │
│   │                                                                     │ │
│   │   ═══════════════════════════════════════════════════════════════ │ │
│   │   ║              CORPORATE FIREWALL                              ║ │ │
│   │   ║              (No inbound connections)                        ║ │ │
│   │   ═══════════════════════════════════════════════════════════════ │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                    ╳                                     │
│                           Cannot connect                                │
│                                    ╳                                     │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    AWS CLOUD                                       │ │
│   │                                                                     │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │              MARKETING COMMAND CENTER                        │ │ │
│   │   │                                                               │ │ │
│   │   │   Needs access to playbooks and templates for:               │ │ │
│   │   │   • Knowledge base for agent RAG                             │ │ │
│   │   │   • Campaign strategy generation                             │ │ │
│   │   │   • Best practice recommendations                            │ │ │
│   │   │                                                               │ │ │
│   │   └─────────────────────────────────────────────────────────────┘ │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   CONSTRAINTS:                                                          │
│   • No VPN tunnel (corp security policy)                                │
│   • No Direct Connect (cost, complexity)                                │
│   • No SharePoint Online (data residency concerns)                      │
│   • Cannot expose SharePoint to internet                                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Why Not Just Use SharePoint Online?

| Option | Status | Reason |
|--------|--------|--------|
| **Migrate to SharePoint Online** | Not feasible (2026) | Data residency policies, migration complexity |
| **Hybrid SharePoint** | Partial | Search federation possible, not full integration |
| **VPN Tunnel** | Not approved | Corporate security policy restrictions |
| **AWS Direct Connect** | Too expensive | Cost not justified for knowledge access alone |
| **Graph API** | N/A | Only for SharePoint Online, not On-Prem |

---

## 2. Architecture Patterns

### 2.1 Pattern Comparison

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ON-PREM INTEGRATION PATTERNS                          │
│                                                                          │
│   PATTERN 1: OUTBOUND BRIDGE AGENT (Recommended)                        │
│   ════════════════════════════════════════════════                      │
│                                                                          │
│   ┌─────────────────┐          ┌─────────────────┐                     │
│   │   SharePoint    │          │    AWS Cloud    │                     │
│   │   On-Prem       │          │                 │                     │
│   │                 │          │  ┌───────────┐  │                     │
│   │  ┌───────────┐  │  HTTPS   │  │  API GW   │  │                     │
│   │  │  Bridge   │──┼────────▶─┼──│ (receive) │  │                     │
│   │  │  Agent    │  │ outbound │  └───────────┘  │                     │
│   │  └───────────┘  │          │                 │                     │
│   │       │         │          │                 │                     │
│   │       ▼         │          │                 │                     │
│   │  ┌───────────┐  │          │                 │                     │
│   │  │ SharePoint│  │          │                 │                     │
│   │  │   CSOM    │  │          │                 │                     │
│   │  └───────────┘  │          │                 │                     │
│   └─────────────────┘          └─────────────────┘                     │
│                                                                          │
│   ✓ No inbound firewall rules                                           │
│   ✓ Agent controls what data leaves network                            │
│   ✓ mTLS for secure communication                                       │
│   ✓ Can be deployed on-prem with minimal footprint                      │
│                                                                          │
│   ───────────────────────────────────────────────────────────────────── │
│                                                                          │
│   PATTERN 2: SCHEDULED EXPORT (Alternative)                             │
│   ═════════════════════════════════════════                              │
│                                                                          │
│   ┌─────────────────┐          ┌─────────────────┐                     │
│   │   SharePoint    │          │    AWS Cloud    │                     │
│   │   On-Prem       │          │                 │                     │
│   │                 │          │  ┌───────────┐  │                     │
│   │  ┌───────────┐  │  S3 PUT  │  │    S3     │  │                     │
│   │  │  Export   │──┼────────▶─┼──│  Bucket   │  │                     │
│   │  │  Job      │  │  (HTTPS) │  └───────────┘  │                     │
│   │  └───────────┘  │          │       │        │                     │
│   │       │         │          │       ▼        │                     │
│   │       ▼         │          │  ┌───────────┐  │                     │
│   │  ┌───────────┐  │          │  │  Lambda   │  │                     │
│   │  │ SharePoint│  │          │  │ (process) │  │                     │
│   │  │   CSOM    │  │          │  └───────────┘  │                     │
│   │  └───────────┘  │          │                 │                     │
│   └─────────────────┘          └─────────────────┘                     │
│                                                                          │
│   ✓ Simpler implementation                                              │
│   ✓ Batch processing                                                    │
│   ⚠ Higher latency (not real-time)                                      │
│   ⚠ Full document export (more data movement)                           │
│                                                                          │
│   ───────────────────────────────────────────────────────────────────── │
│                                                                          │
│   PATTERN 3: MANUAL UPLOAD (Fallback)                                   │
│   ═══════════════════════════════════                                    │
│                                                                          │
│   ┌─────────────────┐          ┌─────────────────┐                     │
│   │   SharePoint    │          │    AWS Cloud    │                     │
│   │   On-Prem       │          │                 │                     │
│   │                 │          │  ┌───────────┐  │                     │
│   │  Human exports  │ Manual   │  │  MCC UI   │  │                     │
│   │  documents      │─────────▶│  │  Upload   │  │                     │
│   │                 │          │  └───────────┘  │                     │
│   └─────────────────┘          └─────────────────┘                     │
│                                                                          │
│   ✓ No technical integration needed                                     │
│   ⚠ Manual effort                                                       │
│   ⚠ Content drift (versions get out of sync)                           │
│   ⚠ Not scalable                                                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Recommended: Outbound Bridge Agent

The **Knowledge Bridge Agent** runs on-premises and:
1. Crawls SharePoint for marketing content
2. Pushes content/metadata to AWS via outbound HTTPS
3. No inbound connections required
4. IT-approved deployment on existing infrastructure

---

## 3. Knowledge Bridge Design

### 3.1 Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    KNOWLEDGE BRIDGE ARCHITECTURE                         │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    CORPORATE NETWORK                               │ │
│   │                                                                     │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │                 KNOWLEDGE BRIDGE AGENT                       │ │ │
│   │   │                 (Windows Service / Container)                │ │ │
│   │   │                                                               │ │ │
│   │   │   ┌─────────────────────────────────────────────────────┐   │ │ │
│   │   │   │                    COMPONENTS                        │   │ │ │
│   │   │   │                                                       │   │ │ │
│   │   │   │   ┌───────────────┐      ┌───────────────┐          │   │ │ │
│   │   │   │   │   Crawler     │      │  Change       │          │   │ │ │
│   │   │   │   │   Service     │      │  Detector     │          │   │ │ │
│   │   │   │   │               │      │               │          │   │ │ │
│   │   │   │   │ • Full crawl  │      │ • Monitors    │          │   │ │ │
│   │   │   │   │   (weekly)    │      │   SP events   │          │   │ │ │
│   │   │   │   │ • Incremental │      │ • Triggers    │          │   │ │ │
│   │   │   │   │   (hourly)    │      │   sync        │          │   │ │ │
│   │   │   │   └───────┬───────┘      └───────┬───────┘          │   │ │ │
│   │   │   │           │                      │                   │   │ │ │
│   │   │   │           └──────────┬───────────┘                   │   │ │ │
│   │   │   │                      ▼                               │   │ │ │
│   │   │   │            ┌───────────────┐                         │   │ │ │
│   │   │   │            │   Content     │                         │   │ │ │
│   │   │   │            │   Processor   │                         │   │ │ │
│   │   │   │            │               │                         │   │ │ │
│   │   │   │            │ • Extract text│                         │   │ │ │
│   │   │   │            │ • Parse metadata                        │   │ │ │
│   │   │   │            │ • Classify    │                         │   │ │ │
│   │   │   │            │ • Chunk       │                         │   │ │ │
│   │   │   │            └───────┬───────┘                         │   │ │ │
│   │   │   │                    │                                 │   │ │ │
│   │   │   │                    ▼                                 │   │ │ │
│   │   │   │            ┌───────────────┐                         │   │ │ │
│   │   │   │            │   Sync        │                         │   │ │ │
│   │   │   │            │   Client      │                         │   │ │ │
│   │   │   │            │               │                         │   │ │ │
│   │   │   │            │ • mTLS auth   │                         │   │ │ │
│   │   │   │            │ • Push to AWS │                         │   │ │ │
│   │   │   │            │ • Handle retry│                         │   │ │ │
│   │   │   │            └───────┬───────┘                         │   │ │ │
│   │   │   │                    │                                 │   │ │ │
│   │   │   └────────────────────┼─────────────────────────────────┘   │ │ │
│   │   │                        │                                     │ │ │
│   │   │                        │ OUTBOUND HTTPS (mTLS)               │ │ │
│   │   │                        │                                     │ │ │
│   │   └────────────────────────┼─────────────────────────────────────┘ │ │
│   │                            │                                       │ │
│   │   ═════════════════════════╪═══════════════════════════════════   │ │
│   │   ║       FIREWALL         ║                                  ║   │ │
│   │   ║  (outbound 443 only)   ║                                  ║   │ │
│   │   ═════════════════════════╪═══════════════════════════════════   │ │
│   │                            │                                       │ │
│   └────────────────────────────┼───────────────────────────────────────┘ │
│                                │                                         │
│                                ▼                                         │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    AWS CLOUD                                       │ │
│   │                                                                     │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │              KNOWLEDGE INGESTION ENDPOINT                    │ │ │
│   │   │                                                               │ │ │
│   │   │   ┌───────────────┐      ┌───────────────┐                  │ │ │
│   │   │   │  API Gateway  │─────▶│    Lambda     │                  │ │ │
│   │   │   │  (mTLS)       │      │  (Ingest)     │                  │ │ │
│   │   │   └───────────────┘      └───────┬───────┘                  │ │ │
│   │   │                                  │                           │ │ │
│   │   │                    ┌─────────────┼─────────────┐            │ │ │
│   │   │                    ▼             ▼             ▼            │ │ │
│   │   │            ┌───────────┐ ┌───────────┐ ┌───────────┐       │ │ │
│   │   │            │    S3     │ │  Bedrock  │ │OpenSearch │       │ │ │
│   │   │            │ (Content) │ │ (Embed)   │ │ (Vector)  │       │ │ │
│   │   │            └───────────┘ └───────────┘ └───────────┘       │ │ │
│   │   │                                                               │ │ │
│   │   └─────────────────────────────────────────────────────────────┘ │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Bridge Agent Components

```csharp
// knowledge-bridge/src/BridgeAgent.cs (C# for Windows deployment)

/// <summary>
/// Main Knowledge Bridge Agent service.
/// Runs as a Windows Service on the corporate network.
/// </summary>
public class KnowledgeBridgeAgent : BackgroundService
{
    private readonly ISharePointCrawler _crawler;
    private readonly IChangeDetector _changeDetector;
    private readonly IContentProcessor _contentProcessor;
    private readonly ISyncClient _syncClient;
    private readonly ILogger _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Knowledge Bridge Agent starting");

        // Start change detection (real-time)
        _ = Task.Run(() => _changeDetector.StartMonitoring(stoppingToken));

        // Schedule full crawl (weekly)
        var fullCrawlTimer = new PeriodicTimer(TimeSpan.FromDays(7));
        _ = Task.Run(async () =>
        {
            while (await fullCrawlTimer.WaitForNextTickAsync(stoppingToken))
            {
                await PerformFullCrawl(stoppingToken);
            }
        });

        // Schedule incremental sync (hourly)
        var incrementalTimer = new PeriodicTimer(TimeSpan.FromHours(1));
        while (await incrementalTimer.WaitForNextTickAsync(stoppingToken))
        {
            await PerformIncrementalSync(stoppingToken);
        }
    }

    private async Task PerformIncrementalSync(CancellationToken ct)
    {
        _logger.LogInformation("Starting incremental sync");

        // Get changed documents since last sync
        var changes = await _changeDetector.GetChangesSinceLastSync();

        foreach (var change in changes)
        {
            try
            {
                if (change.Type == ChangeType.Deleted)
                {
                    await _syncClient.DeleteDocument(change.DocumentId);
                }
                else
                {
                    // Process and sync the document
                    var document = await _crawler.GetDocument(change.DocumentId);
                    var processed = await _contentProcessor.Process(document);
                    await _syncClient.SyncDocument(processed);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to sync document {DocumentId}", change.DocumentId);
            }
        }

        _logger.LogInformation("Incremental sync completed. Synced {Count} documents", changes.Count);
    }
}
```

### 3.3 Content Scope Configuration

```yaml
# knowledge-bridge/config/scope.yaml

# Defines what content to crawl and sync
scope:
  # SharePoint site collections to include
  site_collections:
    - url: "https://sharepoint.corp.local/sites/marketing"
      include_subsites: true
      libraries:
        - "Playbooks"
        - "Templates"
        - "Brand Guidelines"
        - "Process Documentation"

    - url: "https://sharepoint.corp.local/sites/campaigns"
      include_subsites: false
      libraries:
        - "Historical Campaigns"
        - "Case Studies"

  # File types to process
  file_types:
    - ".docx"
    - ".pptx"
    - ".pdf"
    - ".xlsx"
    - ".md"

  # Exclusions
  exclude:
    paths:
      - "/Archive/"
      - "/Draft/"
      - "/Personal/"
    patterns:
      - "*_DRAFT*"
      - "*_OLD*"
    older_than_days: 730  # Exclude files older than 2 years

  # Metadata to extract
  metadata_fields:
    - "Title"
    - "Author"
    - "ModifiedBy"
    - "Modified"
    - "ContentType"
    - "Campaign_Type"       # Custom field
    - "Target_Audience"     # Custom field

  # Content classification
  classification:
    rules:
      - match: "*/Playbooks/*"
        category: "playbook"
        tags: ["process", "guide"]

      - match: "*/Templates/*"
        category: "template"
        tags: ["template", "reusable"]

      - match: "*/Brand*/*"
        category: "brand_guideline"
        tags: ["brand", "creative"]
```

---

## 4. Security Model

### 4.1 Authentication and Authorization

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SECURITY MODEL                                        │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    CORPORATE SIDE                                  │ │
│   │                                                                     │ │
│   │   Bridge Agent Authentication:                                     │ │
│   │   • Runs as domain service account                                 │ │
│   │   • Read-only access to specified SharePoint sites                 │ │
│   │   • No write permissions                                           │ │
│   │   • Audited by corporate AD                                        │ │
│   │                                                                     │ │
│   │   Service Account: SVC_MCC_BRIDGE                                  │ │
│   │   Permissions:                                                     │ │
│   │   ├── Site Collection: /sites/marketing → Read                     │ │
│   │   ├── Site Collection: /sites/campaigns → Read                     │ │
│   │   └── Site Collection: /sites/brand → Read                         │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    CLOUD SIDE                                      │ │
│   │                                                                     │ │
│   │   mTLS Authentication:                                             │ │
│   │   • Bridge agent has client certificate                            │ │
│   │   • Certificate issued by private CA                               │ │
│   │   • API Gateway validates certificate                              │ │
│   │   • Certificate tied to specific tenant                            │ │
│   │                                                                     │ │
│   │   Certificate Subject:                                             │ │
│   │   CN=knowledge-bridge.mktg-001.mcc.internal                        │ │
│   │   O=Marketing Tenant                                               │ │
│   │   OU=Knowledge Bridge                                              │ │
│   │                                                                     │ │
│   │   Certificate Attributes:                                          │ │
│   │   └── x-mcc-tenant-id: mktg-001                                    │ │
│   │                                                                     │ │
│   │   API Gateway mTLS Configuration:                                  │ │
│   │   • Requires client certificate                                    │ │
│   │   • Validates against private CA truststore                        │ │
│   │   • Extracts tenant ID from certificate                            │ │
│   │   • Passes tenant context to Lambda                                │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    DATA PROTECTION                                 │ │
│   │                                                                     │ │
│   │   In Transit:                                                      │ │
│   │   • TLS 1.3 for all connections                                    │ │
│   │   • mTLS for bridge-to-cloud communication                         │ │
│   │   • Certificate pinning in bridge agent                            │ │
│   │                                                                     │ │
│   │   At Rest (Cloud):                                                 │ │
│   │   • S3: SSE-KMS with tenant-specific key                          │ │
│   │   • OpenSearch: Encryption at rest                                 │ │
│   │   • DynamoDB: AWS managed encryption                               │ │
│   │                                                                     │ │
│   │   Content Filtering:                                               │ │
│   │   • PII detection before upload                                    │ │
│   │   • Sensitive file detection (salary, HR docs)                     │ │
│   │   • Configurable blocklist                                         │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Certificate Management

```typescript
// knowledge-bridge/aws/certificate-management.ts

/**
 * Certificate management for Knowledge Bridge mTLS authentication.
 */
interface BridgeCertificate {
  certificate_arn: string;
  tenant_id: string;
  common_name: string;
  issued_at: string;
  expires_at: string;
  status: 'active' | 'expiring_soon' | 'expired' | 'revoked';
}

/**
 * Issue a new client certificate for a Knowledge Bridge agent.
 */
async function issueBridgeCertificate(
  tenantId: string
): Promise<{ certificate: string; privateKey: string; ca: string }> {
  // 1. Generate CSR
  const commonName = `knowledge-bridge.${tenantId}.mcc.internal`;

  // 2. Issue certificate from private CA
  const certificate = await acmPca.issueCertificate({
    CertificateAuthorityArn: BRIDGE_CA_ARN,
    Csr: generateCsr(commonName),
    SigningAlgorithm: 'SHA256WITHRSA',
    Validity: {
      Type: 'DAYS',
      Value: 365,
    },
    TemplateArn: 'arn:aws:acm-pca:::template/EndEntityClientAuthCertificate/V1',
  });

  // 3. Store certificate metadata
  await certificateStore.create({
    certificate_arn: certificate.CertificateArn,
    tenant_id: tenantId,
    common_name: commonName,
    issued_at: new Date().toISOString(),
    expires_at: addDays(new Date(), 365).toISOString(),
    status: 'active',
  });

  // 4. Return certificate bundle for bridge installation
  return {
    certificate: certificate.Certificate,
    privateKey: certificate.PrivateKey,  // Only returned once!
    ca: await getCaCertificate(),
  };
}

/**
 * Validate incoming request from Knowledge Bridge.
 */
async function validateBridgeRequest(
  clientCertificate: string
): Promise<TenantContext> {
  // 1. Parse certificate
  const cert = parseCertificate(clientCertificate);

  // 2. Verify against CA
  await verifyCertificateChain(cert, BRIDGE_CA_ARN);

  // 3. Check revocation
  const isRevoked = await checkCertificateRevocation(cert.serialNumber);
  if (isRevoked) {
    throw new UnauthorizedError('Certificate has been revoked');
  }

  // 4. Extract tenant ID from certificate subject
  const tenantId = extractTenantIdFromCert(cert);

  // 5. Load tenant context
  return await loadTenantContext(tenantId);
}
```

---

## 5. Data Synchronization

### 5.1 Sync Protocol

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SYNC PROTOCOL                                         │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    DOCUMENT SYNC MESSAGE                           │ │
│   │                                                                     │ │
│   │   POST /api/v1/knowledge/documents                                 │ │
│   │   Content-Type: application/json                                   │ │
│   │   X-MCC-Sync-Token: {incremental_sync_token}                       │ │
│   │                                                                     │ │
│   │   {                                                                │ │
│   │     "operation": "upsert",                                         │ │
│   │     "document": {                                                  │ │
│   │       "document_id": "sp://sites/marketing/Playbooks/EventPlaybook.docx",│ │
│   │       "title": "Tier 1 Event Marketing Playbook",                  │ │
│   │       "source_url": "https://sharepoint.corp/sites/marketing/...", │ │
│   │       "content_type": "application/vnd.openxmlformats...",        │ │
│   │       "category": "playbook",                                      │ │
│   │       "tags": ["event", "tier-1", "process"],                      │ │
│   │       "metadata": {                                                │ │
│   │         "author": "Jane Smith",                                    │ │
│   │         "modified_at": "2026-01-05T10:30:00Z",                     │ │
│   │         "version": 5,                                              │ │
│   │         "campaign_type": "event"                                   │ │
│   │       },                                                           │ │
│   │       "chunks": [                                                  │ │
│   │         {                                                          │ │
│   │           "chunk_id": "chunk-001",                                 │ │
│   │           "content": "Introduction to Tier 1 Events...",           │ │
│   │           "section": "Introduction",                               │ │
│   │           "page": 1                                                │ │
│   │         },                                                         │ │
│   │         {                                                          │ │
│   │           "chunk_id": "chunk-002",                                 │ │
│   │           "content": "Planning Phase: 90 days before event...",    │ │
│   │           "section": "Planning",                                   │ │
│   │           "page": 3                                                │ │
│   │         }                                                          │ │
│   │       ]                                                            │ │
│   │     }                                                              │ │
│   │   }                                                                │ │
│   │                                                                     │ │
│   │   Response:                                                        │ │
│   │   {                                                                │ │
│   │     "status": "accepted",                                          │ │
│   │     "document_id": "sp://sites/marketing/Playbooks/EventPlaybook.docx",│ │
│   │     "sync_token": "eyJ0..." // For next incremental sync           │ │
│   │   }                                                                │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │                    BATCH SYNC (Full Crawl)                         │ │
│   │                                                                     │ │
│   │   POST /api/v1/knowledge/documents/batch                           │ │
│   │                                                                     │ │
│   │   {                                                                │ │
│   │     "batch_id": "batch-2026-01-10-001",                            │ │
│   │     "documents": [ /* array of documents */ ],                     │ │
│   │     "is_final": false,                                             │ │
│   │     "total_documents": 500,                                        │ │
│   │     "batch_number": 1,                                             │ │
│   │     "total_batches": 5                                             │ │
│   │   }                                                                │ │
│   │                                                                     │ │
│   │   // After all batches:                                            │ │
│   │   POST /api/v1/knowledge/sync/complete                             │ │
│   │   {                                                                │ │
│   │     "batch_id": "batch-2026-01-10-001",                            │ │
│   │     "total_documents_synced": 500,                                 │ │
│   │     "delete_missing": true  // Remove docs not in this sync        │ │
│   │   }                                                                │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Content Processing Pipeline

```typescript
// knowledge-bridge/src/processing/pipeline.ts

/**
 * Content processing pipeline that runs on the bridge agent.
 * Prepares documents for cloud ingestion.
 */
class ContentProcessingPipeline {
  private readonly textExtractor: ITextExtractor;
  private readonly chunker: IDocumentChunker;
  private readonly classifier: IContentClassifier;
  private readonly piiDetector: IPiiDetector;

  async process(document: SharePointDocument): Promise<ProcessedDocument> {
    // 1. Extract text content
    const textContent = await this.textExtractor.extract(document);

    // 2. Check for PII
    const piiResults = await this.piiDetector.scan(textContent);
    if (piiResults.hasPii) {
      // Either redact or skip document
      if (this.config.piiHandling === 'redact') {
        textContent = piiResults.redactedContent;
      } else {
        throw new PiiDetectedError(document.id, piiResults.types);
      }
    }

    // 3. Classify content
    const classification = await this.classifier.classify(document, textContent);

    // 4. Chunk for vector embedding
    const chunks = await this.chunker.chunk(textContent, {
      maxChunkSize: 1000,
      overlapSize: 100,
      preserveSections: true,
    });

    // 5. Build processed document
    return {
      document_id: this.buildDocumentId(document),
      title: document.title,
      source_url: document.webUrl,
      content_type: document.contentType,
      category: classification.category,
      tags: classification.tags,
      metadata: {
        author: document.author,
        modified_at: document.modifiedTime,
        version: document.versionNumber,
        ...document.customFields,
      },
      chunks: chunks.map((chunk, index) => ({
        chunk_id: `${this.buildDocumentId(document)}-chunk-${index}`,
        content: chunk.content,
        section: chunk.sectionTitle,
        page: chunk.pageNumber,
      })),
    };
  }

  private buildDocumentId(document: SharePointDocument): string {
    // Stable ID that survives moves/renames
    return `sp://${document.siteUrl}/${document.libraryName}/${document.uniqueId}`;
  }
}
```

---

## 6. Implementation Approach

### 6.1 Deployment Options

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DEPLOYMENT OPTIONS                                    │
│                                                                          │
│   OPTION 1: Windows Service (Recommended for most orgs)                 │
│   ═════════════════════════════════════════════════════                 │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │   Deployment:                                                      │ │
│   │   • Install on existing Windows Server                             │ │
│   │   • Runs as Windows Service under service account                  │ │
│   │   • MSI installer for easy deployment                              │ │
│   │                                                                     │ │
│   │   Requirements:                                                    │ │
│   │   • Windows Server 2016+                                           │ │
│   │   • .NET 6.0 Runtime                                               │ │
│   │   • 2 GB RAM, 10 GB disk                                           │ │
│   │   • Outbound HTTPS access                                          │ │
│   │                                                                     │ │
│   │   Pros:                                                            │ │
│   │   • Familiar to Windows admins                                     │ │
│   │   • Easy certificate management                                    │ │
│   │   • Uses native SharePoint CSOM                                    │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│   OPTION 2: Container (For Kubernetes environments)                     │
│   ════════════════════════════════════════════════                      │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │   Deployment:                                                      │ │
│   │   • Docker container on internal Kubernetes                        │ │
│   │   • Helm chart for configuration                                   │ │
│   │                                                                     │ │
│   │   Requirements:                                                    │ │
│   │   • Kubernetes cluster with Windows node pool                      │ │
│   │   • Or Linux container with REST-based SP access                   │ │
│   │                                                                     │ │
│   │   Pros:                                                            │ │
│   │   • Easier scaling                                                 │ │
│   │   • Infrastructure as code                                         │ │
│   │   • Better for multi-site deployments                              │ │
│   │                                                                     │ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Implementation Phases

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    IMPLEMENTATION PHASES                                 │
│                                                                          │
│   PHASE 1: POC (Weeks 1-2)                                              │
│   ─────────────────────────                                             │
│   • Manual export of 50 key documents to S3                             │
│   • Test knowledge base ingestion and RAG                               │
│   • Validate agent responses using SharePoint content                   │
│                                                                          │
│   Deliverable: Proof that SharePoint content improves agent responses   │
│                                                                          │
│   PHASE 2: Bridge Development (Weeks 3-6)                               │
│   ────────────────────────────────────────                              │
│   • Develop bridge agent core                                           │ │
│   • SharePoint CSOM integration                                         │
│   • Content processing pipeline                                         │
│   • Sync client with mTLS                                               │
│                                                                          │
│   Deliverable: Working bridge agent in dev environment                  │
│                                                                          │
│   PHASE 3: Cloud Endpoint (Weeks 5-6, parallel)                         │
│   ──────────────────────────────────────────────                        │
│   • API Gateway with mTLS                                               │
│   • Certificate management system                                       │
│   • Ingestion Lambda                                                    │
│   • Integration with knowledge base                                     │
│                                                                          │
│   Deliverable: Cloud endpoint ready for bridge connection               │
│                                                                          │
│   PHASE 4: Security Review (Week 7)                                     │
│   ──────────────────────────────────                                    │
│   • Corporate security team review                                      │
│   • Penetration testing                                                 │
│   • Compliance sign-off                                                 │
│                                                                          │
│   Deliverable: Security approval for production deployment              │
│                                                                          │
│   PHASE 5: Production Deployment (Week 8)                               │
│   ─────────────────────────────────────────                             │
│   • Install bridge agent on production server                           │
│   • Configure site collection scope                                     │
│   • Initial full sync                                                   │
│   • Monitoring and alerting setup                                       │
│                                                                          │
│   Deliverable: Knowledge Bridge operational in production               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Operational Considerations

### 7.1 Monitoring

```yaml
# knowledge-bridge/monitoring/alerts.yaml

alerts:
  - name: bridge_sync_failure
    condition: sync_error_count > 5 in 1 hour
    severity: high
    notification: pagerduty

  - name: bridge_offline
    condition: heartbeat_missing > 15 minutes
    severity: critical
    notification: pagerduty

  - name: sync_latency_high
    condition: sync_latency_p95 > 30 seconds
    severity: medium
    notification: slack

  - name: certificate_expiring
    condition: certificate_days_until_expiry < 30
    severity: high
    notification: email

  - name: content_drift
    condition: documents_not_synced > 100
    severity: medium
    notification: slack

metrics:
  - name: documents_synced_total
    type: counter
    description: Total documents synced

  - name: sync_latency_seconds
    type: histogram
    description: Time to sync each document

  - name: sync_errors_total
    type: counter
    labels: [error_type]
    description: Sync errors by type

  - name: heartbeat_timestamp
    type: gauge
    description: Last heartbeat from bridge agent
```

### 7.2 Disaster Recovery

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DISASTER RECOVERY SCENARIOS                           │
│                                                                          │
│   SCENARIO 1: Bridge Agent Failure                                      │
│   ────────────────────────────────                                      │
│   Impact: New/updated content not synced                                │
│   RTO: 4 hours                                                          │
│                                                                          │
│   Recovery Steps:                                                       │
│   1. Alert triggers when heartbeat missing > 15 min                     │
│   2. On-call engineer checks agent server                               │
│   3. Restart service or restore from backup                             │
│   4. Trigger full sync to catch up                                      │
│                                                                          │
│   Prevention:                                                           │
│   • Run redundant bridge agents (active-passive)                        │
│   • Health check endpoint for load balancer                             │
│                                                                          │
│   ───────────────────────────────────────────────────────────────────── │
│                                                                          │
│   SCENARIO 2: Certificate Expiry                                        │
│   ───────────────────────────────                                       │
│   Impact: Bridge cannot authenticate to cloud                           │
│   RTO: 2 hours (certificate renewal process)                            │
│                                                                          │
│   Recovery Steps:                                                       │
│   1. Alert at 30 days before expiry                                     │
│   2. Generate new certificate                                           │
│   3. Install on bridge agent                                            │
│   4. Verify connectivity                                                │
│                                                                          │
│   Prevention:                                                           │
│   • 30-day advance warning                                              │
│   • Documented renewal runbook                                          │
│   • Automated renewal (future enhancement)                              │
│                                                                          │
│   ───────────────────────────────────────────────────────────────────── │
│                                                                          │
│   SCENARIO 3: SharePoint Unavailable                                    │
│   ──────────────────────────────────                                    │
│   Impact: Cannot crawl new content                                      │
│   RTO: Depends on SharePoint restoration                                │
│                                                                          │
│   Recovery Steps:                                                       │
│   1. Bridge agent retries with exponential backoff                      │
│   2. Alert after sustained failures                                     │
│   3. Cloud-side knowledge base continues to serve existing content      │
│   4. Full sync when SharePoint restored                                 │
│                                                                          │
│   Prevention:                                                           │
│   • Graceful degradation (stale content better than no content)         │
│   • Cloud-side content has TTL but doesn't auto-delete                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Migration Path to Cloud

### 8.1 Future State: SharePoint Online

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MIGRATION PATH TO SHAREPOINT ONLINE                   │
│                                                                          │
│   CURRENT STATE (2026)                                                  │
│   ───────────────────                                                   │
│                                                                          │
│   SharePoint On-Prem ──[Bridge Agent]──▶ MCC Knowledge Base            │
│                                                                          │
│   ───────────────────────────────────────────────────────────────────── │
│                                                                          │
│   HYBRID STATE (2027, if applicable)                                    │
│   ──────────────────────────────────                                    │
│                                                                          │
│   SharePoint On-Prem ──[Bridge Agent]──▶┐                               │
│                                          │                               │
│                                          ├──▶ MCC Knowledge Base        │
│                                          │                               │
│   SharePoint Online ───[Graph API]─────▶┘                               │
│                                                                          │
│   Both sources feed into the same knowledge base                        │
│   Migration happens document-by-document                                │
│                                                                          │
│   ───────────────────────────────────────────────────────────────────── │
│                                                                          │
│   TARGET STATE (2028+)                                                  │
│   ────────────────────                                                  │
│                                                                          │
│   SharePoint Online ───[Graph API / Native]───▶ MCC Knowledge Base     │
│                                                                          │
│   Benefits:                                                             │
│   • No bridge agent to maintain                                         │
│   • Real-time sync via webhooks                                         │
│   • Native Microsoft 365 integration                                    │
│   • Reduced operational overhead                                        │
│                                                                          │
│   ───────────────────────────────────────────────────────────────────── │
│                                                                          │
│   MIGRATION CONSIDERATIONS                                              │
│                                                                          │
│   1. Document ID mapping                                                │
│      • On-prem IDs differ from Online IDs                               │
│      • Maintain mapping table during transition                         │
│      • Update references in knowledge base                              │
│                                                                          │
│   2. Metadata migration                                                 │
│      • Custom columns may need remapping                                │
│      • Content types differ between versions                            │
│      • Plan metadata normalization                                      │
│                                                                          │
│   3. Permissions model                                                  │
│      • On-prem uses AD groups                                           │
│      • Online uses Azure AD groups                                      │
│      • Bridge currently bypasses permissions (service account)          │
│      • Online integration could respect user permissions                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

### Key Architectural Decisions

1. **Outbound Bridge Pattern**
   - No inbound firewall rules required
   - Agent controls what data leaves the network
   - mTLS for secure cloud communication

2. **Content Processing On-Prem**
   - Text extraction before upload
   - PII detection at source
   - Chunking for efficient vector search

3. **Incremental Sync**
   - Real-time change detection where possible
   - Hourly incremental sync as fallback
   - Weekly full sync for consistency

4. **Certificate-Based Identity**
   - mTLS for bridge authentication
   - Certificate tied to tenant
   - 365-day validity with renewal alerts

### Implementation Recommendations

1. Start with **manual export POC** to validate value
2. Deploy **Windows Service** for simplest initial deployment
3. Plan for **migration to SharePoint Online** in roadmap
4. Implement **redundant bridge agents** for high availability

### Next Steps

1. Review [Platform Feasibility Analysis](10-platform-feasibility.md) for build vs buy decision
2. Coordinate with IT security for certificate issuance process
3. Identify pilot SharePoint site collections for POC
