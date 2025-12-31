# POC Implementation Plan

## Marketing Command Center - Proof of Concept

### Version: 1.0
### Date: December 29, 2025
### Duration: 12 weeks

---

## Executive Summary

This POC will validate the core value proposition: **AI-powered campaign creation with bi-directional Workfront synchronization**. The focus is on demonstrating feasibility, not building production-ready features.

### POC Scope

| In Scope | Out of Scope |
|----------|--------------|
| Single AI agent (Strategy + Production) | Multi-agent orchestration |
| Workfront integration (create/update) | Asana integration |
| Basic knowledge base (campaign playbooks) | Full document corpus |
| Simple dashboard (campaign list) | Real-time leadership dashboard |
| Single team pilot | Multi-team deployment |
| Manual workflow triggers | Event-driven automation |

### Success Criteria

| Metric | Target | Measurement |
|--------|--------|-------------|
| Campaign creation time | < 10 minutes | End-to-end timing |
| Workfront sync latency | < 2 minutes | Event timestamp comparison |
| Agent response quality | 80% user satisfaction | Survey feedback |
| System availability | 99% during pilot | CloudWatch metrics |
| Knowledge retrieval accuracy | 75%+ relevant results | Manual evaluation |

---

## Phase 1: Foundation (Weeks 1-3)

### Week 1: Environment Setup

#### Day 1-2: AWS Account & Network

```bash
# Step 1: Create dedicated AWS account (or use existing dev account)
# Request account via AWS Organizations or standalone

# Step 2: Enable required services
aws service-quotas request-service-quota-increase \
  --service-code bedrock \
  --quota-code L-XXXXXXXX \
  --desired-value 100

# Step 3: Create base infrastructure using CDK
mkdir marketing-command-center-poc
cd marketing-command-center-poc
npx cdk init app --language typescript
```

**CDK Stack: Network Foundation**

```typescript
// lib/network-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';

export class NetworkStack extends cdk.Stack {
  public readonly vpc: ec2.Vpc;

  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // VPC with public and private subnets
    this.vpc = new ec2.Vpc(this, 'MccVpc', {
      maxAzs: 2,
      natGateways: 1,
      subnetConfiguration: [
        {
          name: 'Public',
          subnetType: ec2.SubnetType.PUBLIC,
          cidrMask: 24,
        },
        {
          name: 'Private',
          subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
          cidrMask: 24,
        },
      ],
    });

    // VPC Endpoints for AWS services
    this.vpc.addGatewayEndpoint('DynamoDBEndpoint', {
      service: ec2.GatewayVpcEndpointAwsService.DYNAMODB,
    });

    this.vpc.addGatewayEndpoint('S3Endpoint', {
      service: ec2.GatewayVpcEndpointAwsService.S3,
    });

    // Interface endpoint for Bedrock
    this.vpc.addInterfaceEndpoint('BedrockEndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.BEDROCK_RUNTIME,
    });
  }
}
```

#### Day 3-4: Database Setup

**CDK Stack: Data Layer**

```typescript
// lib/data-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as s3 from 'aws-cdk-lib/aws-s3';

export class DataStack extends cdk.Stack {
  public readonly campaignsTable: dynamodb.Table;
  public readonly conversationsTable: dynamodb.Table;
  public readonly documentsBucket: s3.Bucket;

  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Campaigns table
    this.campaignsTable = new dynamodb.Table(this, 'CampaignsTable', {
      tableName: 'mcc-campaigns',
      partitionKey: { name: 'campaignId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      removalPolicy: cdk.RemovalPolicy.DESTROY, // POC only
      pointInTimeRecovery: true,
    });

    // GSI for status queries
    this.campaignsTable.addGlobalSecondaryIndex({
      indexName: 'status-index',
      partitionKey: { name: 'status', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'createdAt', type: dynamodb.AttributeType.STRING },
    });

    // Conversations table (agent memory)
    this.conversationsTable = new dynamodb.Table(this, 'ConversationsTable', {
      tableName: 'mcc-conversations',
      partitionKey: { name: 'conversationId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'messageId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      timeToLiveAttribute: 'expiresAt',
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    // S3 bucket for knowledge base documents
    this.documentsBucket = new s3.Bucket(this, 'DocumentsBucket', {
      bucketName: `mcc-documents-${this.account}`,
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true, // POC only
    });
  }
}
```

#### Day 5: Secrets & Configuration

```bash
# Create Workfront API credentials in Secrets Manager
aws secretsmanager create-secret \
  --name mcc/workfront/api-credentials \
  --secret-string '{
    "clientId": "YOUR_WORKFRONT_CLIENT_ID",
    "clientSecret": "YOUR_WORKFRONT_CLIENT_SECRET",
    "instanceUrl": "https://your-instance.my.workfront.com"
  }'

# Create environment configuration
aws ssm put-parameter \
  --name /mcc/config/environment \
  --type String \
  --value "poc"

aws ssm put-parameter \
  --name /mcc/config/workfront-api-version \
  --type String \
  --value "v20.0"
```

---

### Week 2: Workfront Integration

#### Day 1-2: Workfront API Client

**Create Lambda Layer for Workfront SDK**

```python
# layers/workfront/python/workfront_client.py
import os
import json
import requests
from typing import Dict, Any, Optional
import boto3

class WorkfrontClient:
    """Workfront API client with OAuth2 authentication."""

    def __init__(self):
        self.secrets_client = boto3.client('secretsmanager')
        self._load_credentials()

    def _load_credentials(self):
        """Load credentials from Secrets Manager."""
        secret = self.secrets_client.get_secret_value(
            SecretId='mcc/workfront/api-credentials'
        )
        creds = json.loads(secret['SecretString'])
        self.client_id = creds['clientId']
        self.client_secret = creds['clientSecret']
        self.base_url = creds['instanceUrl']
        self._access_token = None

    def _get_access_token(self) -> str:
        """Get OAuth2 access token."""
        if self._access_token:
            return self._access_token

        response = requests.post(
            f"{self.base_url}/oauth2/token",
            data={
                'grant_type': 'client_credentials',
                'client_id': self.client_id,
                'client_secret': self.client_secret,
            }
        )
        response.raise_for_status()
        self._access_token = response.json()['access_token']
        return self._access_token

    def _request(self, method: str, endpoint: str, **kwargs) -> Dict[str, Any]:
        """Make authenticated API request."""
        headers = {
            'Authorization': f'Bearer {self._get_access_token()}',
            'Content-Type': 'application/json',
        }
        url = f"{self.base_url}/attask/api/v20.0{endpoint}"

        response = requests.request(method, url, headers=headers, **kwargs)
        response.raise_for_status()
        return response.json()

    def create_project(
        self,
        name: str,
        description: str = None,
        template_id: str = None,
        planned_start_date: str = None,
        custom_fields: Dict[str, Any] = None
    ) -> Dict[str, Any]:
        """Create a new project in Workfront."""
        data = {
            'name': name,
            'description': description,
            'status': 'PLN',  # Planning status
        }

        if template_id:
            data['templateID'] = template_id
        if planned_start_date:
            data['plannedStartDate'] = planned_start_date
        if custom_fields:
            data.update(custom_fields)

        return self._request('POST', '/proj', json=data)

    def create_task(
        self,
        project_id: str,
        name: str,
        description: str = None,
        assignee_id: str = None,
        planned_completion_date: str = None
    ) -> Dict[str, Any]:
        """Create a task within a project."""
        data = {
            'projectID': project_id,
            'name': name,
        }

        if description:
            data['description'] = description
        if assignee_id:
            data['assignedToID'] = assignee_id
        if planned_completion_date:
            data['plannedCompletionDate'] = planned_completion_date

        return self._request('POST', '/task', json=data)

    def update_task_status(
        self,
        task_id: str,
        status: str
    ) -> Dict[str, Any]:
        """Update task status."""
        return self._request('PUT', f'/task/{task_id}', json={'status': status})

    def get_project(self, project_id: str) -> Dict[str, Any]:
        """Get project details."""
        return self._request('GET', f'/proj/{project_id}')

    def search_projects(
        self,
        filters: Dict[str, Any] = None,
        limit: int = 100
    ) -> Dict[str, Any]:
        """Search for projects."""
        params = {'$$LIMIT': limit}
        if filters:
            params.update(filters)
        return self._request('GET', '/proj/search', params=params)
```

#### Day 3-4: Webhook Handler

```python
# functions/workfront_webhook/handler.py
import json
import os
import boto3
import hashlib
import hmac
from datetime import datetime

dynamodb = boto3.resource('dynamodb')
eventbridge = boto3.client('events')
campaigns_table = dynamodb.Table(os.environ['CAMPAIGNS_TABLE'])

def verify_webhook_signature(body: str, signature: str, secret: str) -> bool:
    """Verify Workfront webhook signature."""
    expected = hmac.new(
        secret.encode(),
        body.encode(),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)

def handler(event, context):
    """Handle incoming Workfront webhooks."""

    # Parse webhook payload
    body = event.get('body', '{}')
    payload = json.loads(body)

    # Log for debugging
    print(f"Received webhook: {json.dumps(payload)}")

    # Extract event details
    event_type = payload.get('eventType')
    obj_code = payload.get('objCode')
    obj_id = payload.get('newState', {}).get('ID')

    # Map Workfront events to internal events
    event_mapping = {
        ('UPDATE', 'PROJ'): 'workfront.project.updated',
        ('CREATE', 'TASK'): 'workfront.task.created',
        ('UPDATE', 'TASK'): 'workfront.task.updated',
        ('DELETE', 'TASK'): 'workfront.task.deleted',
    }

    internal_event = event_mapping.get((event_type, obj_code))

    if internal_event:
        # Publish to EventBridge
        eventbridge.put_events(
            Entries=[{
                'Source': 'mcc.workfront',
                'DetailType': internal_event,
                'Detail': json.dumps({
                    'workfrontId': obj_id,
                    'eventType': event_type,
                    'objectCode': obj_code,
                    'newState': payload.get('newState'),
                    'oldState': payload.get('oldState'),
                    'timestamp': datetime.utcnow().isoformat(),
                }),
                'EventBusName': os.environ['EVENT_BUS_NAME'],
            }]
        )

        # Update sync state in DynamoDB if campaign exists
        try:
            campaigns_table.update_item(
                Key={'campaignId': obj_id},
                UpdateExpression='SET lastSyncedAt = :ts, workfrontStatus = :status',
                ExpressionAttributeValues={
                    ':ts': datetime.utcnow().isoformat(),
                    ':status': payload.get('newState', {}).get('status'),
                },
                ConditionExpression='attribute_exists(campaignId)',
            )
        except dynamodb.meta.client.exceptions.ConditionalCheckFailedException:
            pass  # Campaign not tracked in our system

    return {
        'statusCode': 200,
        'body': json.dumps({'message': 'Webhook processed'}),
    }
```

#### Day 5: Integration Testing

```python
# tests/integration/test_workfront_integration.py
import pytest
import os
from workfront_client import WorkfrontClient

@pytest.fixture
def workfront():
    return WorkfrontClient()

class TestWorkfrontIntegration:

    def test_create_project(self, workfront):
        """Test creating a project in Workfront."""
        result = workfront.create_project(
            name="POC Test Campaign - Delete Me",
            description="Created by MCC POC integration test",
        )

        assert 'data' in result
        assert 'ID' in result['data']

        # Clean up
        project_id = result['data']['ID']
        # Note: Delete requires separate permission

    def test_create_task(self, workfront):
        """Test creating a task in Workfront."""
        # First create a project
        project = workfront.create_project(name="Task Test Project")
        project_id = project['data']['ID']

        # Create task
        result = workfront.create_task(
            project_id=project_id,
            name="Test Task",
            description="Created by integration test",
        )

        assert 'data' in result
        assert result['data']['projectID'] == project_id

    def test_search_projects(self, workfront):
        """Test searching for projects."""
        result = workfront.search_projects(
            filters={'status': 'PLN'},
            limit=10
        )

        assert 'data' in result
        assert isinstance(result['data'], list)
```

---

### Week 3: Knowledge Base Setup

#### Day 1-2: Bedrock Knowledge Base

```typescript
// lib/knowledge-base-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as bedrock from 'aws-cdk-lib/aws-bedrock';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as opensearchserverless from 'aws-cdk-lib/aws-opensearchserverless';

export class KnowledgeBaseStack extends cdk.Stack {
  public readonly knowledgeBase: bedrock.CfnKnowledgeBase;

  constructor(scope: cdk.App, id: string, props: {
    documentsBucket: s3.IBucket;
  }) {
    super(scope, id);

    // OpenSearch Serverless collection for vector store
    const collection = new opensearchserverless.CfnCollection(this, 'VectorCollection', {
      name: 'mcc-knowledge-vectors',
      type: 'VECTORSEARCH',
    });

    // IAM role for Bedrock to access S3 and OpenSearch
    const kbRole = new iam.Role(this, 'KnowledgeBaseRole', {
      assumedBy: new iam.ServicePrincipal('bedrock.amazonaws.com'),
    });

    props.documentsBucket.grantRead(kbRole);

    kbRole.addToPolicy(new iam.PolicyStatement({
      actions: ['aoss:APIAccessAll'],
      resources: [collection.attrArn],
    }));

    // Knowledge Base
    this.knowledgeBase = new bedrock.CfnKnowledgeBase(this, 'CampaignKnowledgeBase', {
      name: 'mcc-campaign-knowledge',
      roleArn: kbRole.roleArn,
      knowledgeBaseConfiguration: {
        type: 'VECTOR',
        vectorKnowledgeBaseConfiguration: {
          embeddingModelArn: `arn:aws:bedrock:${this.region}::foundation-model/amazon.titan-embed-text-v2:0`,
        },
      },
      storageConfiguration: {
        type: 'OPENSEARCH_SERVERLESS',
        opensearchServerlessConfiguration: {
          collectionArn: collection.attrArn,
          vectorIndexName: 'campaign-vectors',
          fieldMapping: {
            vectorField: 'embedding',
            textField: 'text',
            metadataField: 'metadata',
          },
        },
      },
    });

    // Data source - S3 bucket
    new bedrock.CfnDataSource(this, 'DocumentsDataSource', {
      knowledgeBaseId: this.knowledgeBase.attrKnowledgeBaseId,
      name: 'campaign-documents',
      dataSourceConfiguration: {
        type: 'S3',
        s3Configuration: {
          bucketArn: props.documentsBucket.bucketArn,
          inclusionPrefixes: ['playbooks/', 'guidelines/', 'templates/'],
        },
      },
    });
  }
}
```

#### Day 3-4: Upload Initial Documents

```bash
# Create folder structure for knowledge base
mkdir -p knowledge-content/{playbooks,guidelines,templates}

# Create initial playbook content
cat > knowledge-content/playbooks/campaign-tiers.md << 'EOF'
# Campaign Tier Definitions

## Tier 1 Campaigns
- Major product launches
- Flagship events (unBoxed, Cannes)
- Cross-channel, high visibility
- Budget: $100K+
- Lead time: 8-12 weeks
- Approval: VP required

## Tier 2 Campaigns
- Product updates
- Regional events
- Multi-channel execution
- Budget: $25K-$100K
- Lead time: 4-8 weeks
- Approval: Director required

## Tier 3 Campaigns
- BAU communications
- Single channel
- Template-based
- Budget: <$25K
- Lead time: 2-4 weeks
- Approval: Manager
EOF

cat > knowledge-content/playbooks/email-campaign-checklist.md << 'EOF'
# Email Campaign Checklist

## Strategy Phase
- [ ] Define target audience segment
- [ ] Set campaign objectives and KPIs
- [ ] Draft messaging strategy
- [ ] Select email template
- [ ] Plan send cadence

## Production Phase
- [ ] Write subject lines (3 variants)
- [ ] Create email body copy
- [ ] Design email layout
- [ ] Set up tracking links
- [ ] Configure personalization

## QA Phase
- [ ] Test on multiple email clients
- [ ] Verify links work
- [ ] Check mobile rendering
- [ ] Review with stakeholders
- [ ] Get legal approval if required

## Launch Phase
- [ ] Schedule send time
- [ ] Set up A/B test (if applicable)
- [ ] Monitor deliverability
- [ ] Track opens/clicks
- [ ] Report results
EOF

# Upload to S3
aws s3 sync knowledge-content/ s3://mcc-documents-${AWS_ACCOUNT_ID}/

# Trigger knowledge base sync
aws bedrock-agent start-ingestion-job \
  --knowledge-base-id ${KNOWLEDGE_BASE_ID} \
  --data-source-id ${DATA_SOURCE_ID}
```

#### Day 5: Test Knowledge Retrieval

```python
# tests/integration/test_knowledge_base.py
import boto3
import pytest

@pytest.fixture
def bedrock_agent():
    return boto3.client('bedrock-agent-runtime')

class TestKnowledgeBase:

    def test_retrieve_campaign_tiers(self, bedrock_agent):
        """Test retrieving campaign tier information."""
        response = bedrock_agent.retrieve(
            knowledgeBaseId='YOUR_KB_ID',
            retrievalQuery={
                'text': 'What is a Tier 1 campaign?'
            },
            retrievalConfiguration={
                'vectorSearchConfiguration': {
                    'numberOfResults': 3
                }
            }
        )

        assert len(response['retrievalResults']) > 0
        assert 'Tier 1' in response['retrievalResults'][0]['content']['text']

    def test_retrieve_email_checklist(self, bedrock_agent):
        """Test retrieving email campaign checklist."""
        response = bedrock_agent.retrieve(
            knowledgeBaseId='YOUR_KB_ID',
            retrievalQuery={
                'text': 'What are the steps for launching an email campaign?'
            }
        )

        assert len(response['retrievalResults']) > 0
        # Should find the checklist document
```

---

## Phase 2: AI Agent Development (Weeks 4-6)

### Week 4: Basic Agent Setup

#### Day 1-2: Bedrock Agent Definition

```python
# infrastructure/agent_definition.py
import boto3
import json

bedrock_agent = boto3.client('bedrock-agent')

def create_campaign_agent():
    """Create the campaign assistant agent."""

    # Agent instructions
    instructions = """
You are a Marketing Campaign Assistant for the Marketing Command Center.
Your role is to help marketers create and manage campaigns efficiently.

CAPABILITIES:
1. Create new campaign strategies based on user requirements
2. Generate campaign plans following organizational standards
3. Create projects and tasks in Workfront
4. Answer questions about campaign best practices
5. Provide status updates on existing campaigns

GUIDELINES:
- Always reference the knowledge base for campaign standards
- Validate all inputs before creating Workfront items
- Ask clarifying questions when information is incomplete
- Format responses clearly with actionable next steps
- Never make assumptions about budget without explicit user input

WORKFLOW FOR NEW CAMPAIGNS:
1. Ask for campaign basics: name, type, target audience, objective
2. Determine campaign tier based on scope and budget
3. Recommend strategy based on knowledge base playbooks
4. Confirm details with user
5. Create Workfront project with appropriate structure
6. Provide summary and next steps

RESPONSE FORMAT:
- Use bullet points for lists
- Include specific dates when discussing timelines
- Always confirm critical actions before executing
"""

    # Create agent
    response = bedrock_agent.create_agent(
        agentName='mcc-campaign-assistant',
        foundationModel='anthropic.claude-3-5-sonnet-20241022-v2:0',
        instruction=instructions,
        idleSessionTTLInSeconds=1800,  # 30 minutes
        agentResourceRoleArn='arn:aws:iam::ACCOUNT:role/MccAgentRole',
    )

    agent_id = response['agent']['agentId']
    print(f"Created agent: {agent_id}")

    return agent_id
```

#### Day 3-4: Agent Action Groups

```python
# infrastructure/action_groups.py
import boto3
import json

bedrock_agent = boto3.client('bedrock-agent')

def create_workfront_action_group(agent_id: str, lambda_arn: str):
    """Create action group for Workfront operations."""

    # OpenAPI schema for Workfront actions
    api_schema = {
        "openapi": "3.0.0",
        "info": {
            "title": "Workfront Actions",
            "version": "1.0.0"
        },
        "paths": {
            "/createProject": {
                "post": {
                    "operationId": "createProject",
                    "summary": "Create a new project in Workfront",
                    "requestBody": {
                        "required": True,
                        "content": {
                            "application/json": {
                                "schema": {
                                    "type": "object",
                                    "required": ["name", "campaignType"],
                                    "properties": {
                                        "name": {
                                            "type": "string",
                                            "description": "Campaign/project name"
                                        },
                                        "campaignType": {
                                            "type": "string",
                                            "enum": ["standard", "event", "partner"],
                                            "description": "Type of campaign"
                                        },
                                        "tier": {
                                            "type": "integer",
                                            "enum": [1, 2, 3],
                                            "description": "Campaign tier (1=major, 2=medium, 3=BAU)"
                                        },
                                        "description": {
                                            "type": "string",
                                            "description": "Campaign description"
                                        },
                                        "plannedStartDate": {
                                            "type": "string",
                                            "format": "date",
                                            "description": "Planned start date (YYYY-MM-DD)"
                                        },
                                        "targetAudience": {
                                            "type": "string",
                                            "description": "Target audience description"
                                        },
                                        "objective": {
                                            "type": "string",
                                            "description": "Campaign objective"
                                        }
                                    }
                                }
                            }
                        }
                    },
                    "responses": {
                        "200": {
                            "description": "Project created successfully",
                            "content": {
                                "application/json": {
                                    "schema": {
                                        "type": "object",
                                        "properties": {
                                            "projectId": {"type": "string"},
                                            "projectUrl": {"type": "string"},
                                            "status": {"type": "string"}
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            },
            "/createTask": {
                "post": {
                    "operationId": "createTask",
                    "summary": "Create a task within a project",
                    "requestBody": {
                        "required": True,
                        "content": {
                            "application/json": {
                                "schema": {
                                    "type": "object",
                                    "required": ["projectId", "name"],
                                    "properties": {
                                        "projectId": {
                                            "type": "string",
                                            "description": "Workfront project ID"
                                        },
                                        "name": {
                                            "type": "string",
                                            "description": "Task name"
                                        },
                                        "description": {
                                            "type": "string",
                                            "description": "Task description"
                                        },
                                        "dueDate": {
                                            "type": "string",
                                            "format": "date",
                                            "description": "Task due date"
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            },
            "/getProjectStatus": {
                "get": {
                    "operationId": "getProjectStatus",
                    "summary": "Get status of a Workfront project",
                    "parameters": [
                        {
                            "name": "projectId",
                            "in": "query",
                            "required": True,
                            "schema": {"type": "string"}
                        }
                    ]
                }
            }
        }
    }

    response = bedrock_agent.create_agent_action_group(
        agentId=agent_id,
        agentVersion='DRAFT',
        actionGroupName='WorkfrontActions',
        actionGroupExecutor={
            'lambda': lambda_arn
        },
        apiSchema={
            'payload': json.dumps(api_schema)
        },
        description='Actions to interact with Adobe Workfront'
    )

    return response['agentActionGroup']['actionGroupId']

def create_knowledge_action_group(agent_id: str, knowledge_base_id: str):
    """Associate knowledge base with agent."""

    response = bedrock_agent.associate_agent_knowledge_base(
        agentId=agent_id,
        agentVersion='DRAFT',
        knowledgeBaseId=knowledge_base_id,
        description='Campaign playbooks, guidelines, and best practices',
        knowledgeBaseState='ENABLED'
    )

    return response
```

#### Day 5: Action Group Lambda Handler

```python
# functions/agent_actions/handler.py
import json
import os
from workfront_client import WorkfrontClient
import boto3
from datetime import datetime
import uuid

dynamodb = boto3.resource('dynamodb')
campaigns_table = dynamodb.Table(os.environ['CAMPAIGNS_TABLE'])
workfront = WorkfrontClient()

def handler(event, context):
    """Handle Bedrock Agent action group invocations."""

    print(f"Agent action event: {json.dumps(event)}")

    action = event.get('actionGroup')
    api_path = event.get('apiPath')
    parameters = event.get('requestBody', {}).get('content', {}).get('application/json', {}).get('properties', {})

    # Parse parameters
    params = {p['name']: p['value'] for p in parameters} if parameters else {}

    try:
        if api_path == '/createProject':
            result = create_project(params)
        elif api_path == '/createTask':
            result = create_task(params)
        elif api_path == '/getProjectStatus':
            result = get_project_status(params)
        else:
            result = {'error': f'Unknown action: {api_path}'}

        return format_response(result)

    except Exception as e:
        print(f"Error executing action: {str(e)}")
        return format_response({'error': str(e)}, status_code=500)

def create_project(params: dict) -> dict:
    """Create a Workfront project and track in DynamoDB."""

    # Create in Workfront
    wf_result = workfront.create_project(
        name=params['name'],
        description=params.get('description', ''),
        planned_start_date=params.get('plannedStartDate'),
    )

    workfront_id = wf_result['data']['ID']
    campaign_id = str(uuid.uuid4())

    # Store in DynamoDB
    campaigns_table.put_item(Item={
        'campaignId': campaign_id,
        'workfrontProjectId': workfront_id,
        'name': params['name'],
        'type': params.get('campaignType', 'standard'),
        'tier': params.get('tier', 3),
        'status': 'PLANNING',
        'description': params.get('description', ''),
        'targetAudience': params.get('targetAudience', ''),
        'objective': params.get('objective', ''),
        'createdAt': datetime.utcnow().isoformat(),
        'updatedAt': datetime.utcnow().isoformat(),
    })

    instance_url = os.environ.get('WORKFRONT_INSTANCE_URL', '')

    return {
        'projectId': workfront_id,
        'campaignId': campaign_id,
        'projectUrl': f"{instance_url}/project/{workfront_id}",
        'status': 'Created successfully',
        'message': f"Campaign '{params['name']}' has been created in Workfront."
    }

def create_task(params: dict) -> dict:
    """Create a task in Workfront."""

    wf_result = workfront.create_task(
        project_id=params['projectId'],
        name=params['name'],
        description=params.get('description'),
        planned_completion_date=params.get('dueDate'),
    )

    return {
        'taskId': wf_result['data']['ID'],
        'projectId': params['projectId'],
        'status': 'Created successfully',
        'message': f"Task '{params['name']}' has been created."
    }

def get_project_status(params: dict) -> dict:
    """Get project status from Workfront."""

    project_id = params.get('projectId')
    wf_result = workfront.get_project(project_id)

    data = wf_result.get('data', {})

    return {
        'projectId': project_id,
        'name': data.get('name'),
        'status': data.get('status'),
        'percentComplete': data.get('percentComplete'),
        'plannedCompletionDate': data.get('plannedCompletionDate'),
    }

def format_response(body: dict, status_code: int = 200) -> dict:
    """Format response for Bedrock Agent."""
    return {
        'messageVersion': '1.0',
        'response': {
            'actionGroup': 'WorkfrontActions',
            'apiPath': '/response',
            'httpMethod': 'POST',
            'httpStatusCode': status_code,
            'responseBody': {
                'application/json': {
                    'body': json.dumps(body)
                }
            }
        }
    }
```

---

### Week 5: Agent Testing & Refinement

#### Day 1-2: Agent Preparation & Deployment

```bash
# Prepare agent for testing
aws bedrock-agent prepare-agent --agent-id ${AGENT_ID}

# Create agent alias for testing
aws bedrock-agent create-agent-alias \
  --agent-id ${AGENT_ID} \
  --agent-alias-name poc-test
```

#### Day 3-4: Agent Test Suite

```python
# tests/agent/test_campaign_agent.py
import boto3
import pytest
import uuid

@pytest.fixture
def bedrock_runtime():
    return boto3.client('bedrock-agent-runtime')

@pytest.fixture
def session_id():
    return str(uuid.uuid4())

class TestCampaignAgent:

    AGENT_ID = 'YOUR_AGENT_ID'
    AGENT_ALIAS_ID = 'YOUR_ALIAS_ID'

    def invoke_agent(self, client, session_id: str, prompt: str) -> str:
        """Helper to invoke agent and collect response."""
        response = client.invoke_agent(
            agentId=self.AGENT_ID,
            agentAliasId=self.AGENT_ALIAS_ID,
            sessionId=session_id,
            inputText=prompt,
        )

        # Collect streaming response
        completion = ""
        for event in response['completion']:
            if 'chunk' in event:
                completion += event['chunk']['bytes'].decode()

        return completion

    def test_greeting_response(self, bedrock_runtime, session_id):
        """Test basic agent greeting."""
        response = self.invoke_agent(
            bedrock_runtime,
            session_id,
            "Hello, I need help creating a new campaign."
        )

        assert len(response) > 0
        # Agent should ask clarifying questions
        assert any(word in response.lower() for word in ['type', 'audience', 'objective', 'help'])

    def test_tier_classification(self, bedrock_runtime, session_id):
        """Test that agent correctly classifies campaign tiers."""
        response = self.invoke_agent(
            bedrock_runtime,
            session_id,
            "I want to create a major product launch campaign for unBoxed with a $150,000 budget across all channels."
        )

        assert 'tier 1' in response.lower() or 'tier-1' in response.lower()

    def test_knowledge_retrieval(self, bedrock_runtime, session_id):
        """Test that agent retrieves from knowledge base."""
        response = self.invoke_agent(
            bedrock_runtime,
            session_id,
            "What are the steps for launching an email campaign?"
        )

        # Should include checklist items from knowledge base
        assert any(word in response.lower() for word in ['checklist', 'subject line', 'test', 'qa'])

    def test_campaign_creation_flow(self, bedrock_runtime, session_id):
        """Test full campaign creation conversation."""
        # Step 1: Start conversation
        response1 = self.invoke_agent(
            bedrock_runtime,
            session_id,
            "Create a new email campaign for SMB customers to promote Q1 offers."
        )

        # Agent should ask for more details
        assert len(response1) > 0

        # Step 2: Provide details
        response2 = self.invoke_agent(
            bedrock_runtime,
            session_id,
            "The campaign is called 'Q1 SMB Promotion'. Target audience is small business owners. Objective is to increase ASIN penetration by 15%. Budget is $30,000. Launch date is February 15, 2026."
        )

        # Agent should summarize and offer to create
        assert 'q1 smb' in response2.lower() or 'campaign' in response2.lower()

        # Step 3: Confirm creation
        response3 = self.invoke_agent(
            bedrock_runtime,
            session_id,
            "Yes, please create this campaign in Workfront."
        )

        # Should confirm project was created
        assert 'created' in response3.lower() or 'workfront' in response3.lower()
```

#### Day 5: Prompt Engineering & Refinement

```python
# scripts/agent_prompt_tuning.py
"""
Script to test different agent instruction variations and measure quality.
"""

import boto3
import json
from typing import List, Dict

bedrock_agent = boto3.client('bedrock-agent')
bedrock_runtime = boto3.client('bedrock-agent-runtime')

# Test scenarios
TEST_SCENARIOS = [
    {
        "name": "Simple campaign creation",
        "prompts": [
            "I need to create a new email campaign",
            "It's for SMB customers, promoting new features",
            "Budget is $25,000, launch in 4 weeks",
        ],
        "expected_behaviors": [
            "asks_clarifying_questions",
            "identifies_tier",
            "references_checklist",
            "offers_to_create_project",
        ]
    },
    {
        "name": "Event campaign",
        "prompts": [
            "Help me plan our presence at Cannes 2026",
            "It's a Tier 1 event with booth, speaking sessions, and sponsored activities",
        ],
        "expected_behaviors": [
            "recognizes_tier_1",
            "mentions_event_playbook",
            "suggests_comprehensive_plan",
        ]
    },
    {
        "name": "Status inquiry",
        "prompts": [
            "What's the status of the Q4 Product Launch campaign?",
        ],
        "expected_behaviors": [
            "asks_for_project_id_or_name",
            "retrieves_status",
        ]
    }
]

def evaluate_response(response: str, expected_behaviors: List[str]) -> Dict[str, bool]:
    """Evaluate if response exhibits expected behaviors."""
    results = {}

    behavior_checks = {
        "asks_clarifying_questions": lambda r: "?" in r,
        "identifies_tier": lambda r: any(f"tier {i}" in r.lower() for i in [1, 2, 3]),
        "references_checklist": lambda r: "checklist" in r.lower() or "steps" in r.lower(),
        "offers_to_create_project": lambda r: "create" in r.lower() and ("workfront" in r.lower() or "project" in r.lower()),
        "recognizes_tier_1": lambda r: "tier 1" in r.lower() or "tier-1" in r.lower(),
        "mentions_event_playbook": lambda r: "playbook" in r.lower() or "event" in r.lower(),
        "suggests_comprehensive_plan": lambda r: len(r) > 500,  # Detailed response
        "asks_for_project_id_or_name": lambda r: "id" in r.lower() or "name" in r.lower() or "which" in r.lower(),
        "retrieves_status": lambda r: "status" in r.lower(),
    }

    for behavior in expected_behaviors:
        if behavior in behavior_checks:
            results[behavior] = behavior_checks[behavior](response)
        else:
            results[behavior] = False

    return results

def run_evaluation():
    """Run all test scenarios and report results."""
    # This would invoke the agent and evaluate responses
    # Implementation depends on deployed agent
    pass

if __name__ == "__main__":
    run_evaluation()
```

---

### Week 6: Basic UI & Dashboard

#### Day 1-2: AppSync GraphQL API

```typescript
// lib/api-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as appsync from 'aws-cdk-lib/aws-appsync';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as cognito from 'aws-cdk-lib/aws-cognito';

export class ApiStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props: {
    campaignsTable: dynamodb.ITable;
    conversationsTable: dynamodb.ITable;
    userPool: cognito.IUserPool;
  }) {
    super(scope, id);

    // GraphQL API
    const api = new appsync.GraphqlApi(this, 'MccApi', {
      name: 'mcc-graphql-api',
      schema: appsync.SchemaFile.fromAsset('schema/schema.graphql'),
      authorizationConfig: {
        defaultAuthorization: {
          authorizationType: appsync.AuthorizationType.USER_POOL,
          userPoolConfig: {
            userPool: props.userPool,
          },
        },
      },
      xrayEnabled: true,
    });

    // DynamoDB data source for campaigns
    const campaignsDS = api.addDynamoDbDataSource('CampaignsDS', props.campaignsTable);

    // Resolver: List Campaigns
    campaignsDS.createResolver('ListCampaignsResolver', {
      typeName: 'Query',
      fieldName: 'listCampaigns',
      requestMappingTemplate: appsync.MappingTemplate.dynamoDbScanTable(),
      responseMappingTemplate: appsync.MappingTemplate.dynamoDbResultList(),
    });

    // Resolver: Get Campaign
    campaignsDS.createResolver('GetCampaignResolver', {
      typeName: 'Query',
      fieldName: 'getCampaign',
      requestMappingTemplate: appsync.MappingTemplate.dynamoDbGetItem('campaignId', 'id'),
      responseMappingTemplate: appsync.MappingTemplate.dynamoDbResultItem(),
    });

    // Lambda for agent interactions
    const agentLambda = new lambda.Function(this, 'AgentInteractionFn', {
      runtime: lambda.Runtime.PYTHON_3_12,
      handler: 'handler.handler',
      code: lambda.Code.fromAsset('functions/agent_interaction'),
      timeout: cdk.Duration.seconds(60),
      environment: {
        AGENT_ID: process.env.AGENT_ID || '',
        AGENT_ALIAS_ID: process.env.AGENT_ALIAS_ID || '',
        CONVERSATIONS_TABLE: props.conversationsTable.tableName,
      },
    });

    props.conversationsTable.grantReadWriteData(agentLambda);

    const agentDS = api.addLambdaDataSource('AgentDS', agentLambda);

    agentDS.createResolver('SendMessageResolver', {
      typeName: 'Mutation',
      fieldName: 'sendAgentMessage',
      requestMappingTemplate: appsync.MappingTemplate.lambdaRequest(),
      responseMappingTemplate: appsync.MappingTemplate.lambdaResult(),
    });

    // Output API URL
    new cdk.CfnOutput(this, 'GraphQLApiUrl', {
      value: api.graphqlUrl,
    });
  }
}
```

**GraphQL Schema**

```graphql
# schema/schema.graphql
type Query {
  getCampaign(id: ID!): Campaign
  listCampaigns(limit: Int, nextToken: String): CampaignConnection!
  getConversation(conversationId: ID!): Conversation
}

type Mutation {
  sendAgentMessage(conversationId: ID, message: String!): AgentResponse!
  createCampaign(input: CreateCampaignInput!): Campaign!
  updateCampaign(id: ID!, input: UpdateCampaignInput!): Campaign!
}

type Subscription {
  onCampaignUpdated: Campaign
    @aws_subscribe(mutations: ["updateCampaign"])
}

type Campaign {
  campaignId: ID!
  name: String!
  type: String!
  tier: Int!
  status: String!
  description: String
  targetAudience: String
  objective: String
  workfrontProjectId: String
  workfrontUrl: String
  createdAt: AWSDateTime!
  updatedAt: AWSDateTime!
}

type CampaignConnection {
  items: [Campaign!]!
  nextToken: String
}

type AgentResponse {
  conversationId: ID!
  messageId: ID!
  content: String!
  timestamp: AWSDateTime!
}

type Conversation {
  conversationId: ID!
  messages: [Message!]!
}

type Message {
  messageId: ID!
  role: String!
  content: String!
  timestamp: AWSDateTime!
}

input CreateCampaignInput {
  name: String!
  type: String!
  tier: Int
  description: String
  targetAudience: String
  objective: String
}

input UpdateCampaignInput {
  name: String
  status: String
  description: String
}
```

#### Day 3-4: React Frontend (Chat UI)

```typescript
// frontend/src/components/ChatInterface.tsx
import React, { useState, useRef, useEffect } from 'react';
import { useMutation, useQuery } from '@apollo/client';
import { SEND_MESSAGE, GET_CONVERSATION } from '../graphql/queries';

interface Message {
  role: 'user' | 'assistant';
  content: string;
  timestamp: string;
}

export const ChatInterface: React.FC = () => {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');
  const [conversationId, setConversationId] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const messagesEndRef = useRef<HTMLDivElement>(null);

  const [sendMessage] = useMutation(SEND_MESSAGE);

  const scrollToBottom = () => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  };

  useEffect(() => {
    scrollToBottom();
  }, [messages]);

  const handleSend = async () => {
    if (!input.trim() || isLoading) return;

    const userMessage: Message = {
      role: 'user',
      content: input,
      timestamp: new Date().toISOString(),
    };

    setMessages(prev => [...prev, userMessage]);
    setInput('');
    setIsLoading(true);

    try {
      const { data } = await sendMessage({
        variables: {
          conversationId,
          message: input,
        },
      });

      const response = data.sendAgentMessage;

      if (!conversationId) {
        setConversationId(response.conversationId);
      }

      const assistantMessage: Message = {
        role: 'assistant',
        content: response.content,
        timestamp: response.timestamp,
      };

      setMessages(prev => [...prev, assistantMessage]);
    } catch (error) {
      console.error('Error sending message:', error);
      setMessages(prev => [...prev, {
        role: 'assistant',
        content: 'Sorry, I encountered an error. Please try again.',
        timestamp: new Date().toISOString(),
      }]);
    } finally {
      setIsLoading(false);
    }
  };

  const handleKeyPress = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      handleSend();
    }
  };

  return (
    <div className="chat-container">
      <div className="chat-header">
        <h2>Campaign Assistant</h2>
        {conversationId && (
          <span className="conversation-id">Session: {conversationId.slice(0, 8)}...</span>
        )}
      </div>

      <div className="messages-container">
        {messages.length === 0 && (
          <div className="welcome-message">
            <h3>Welcome to the Marketing Command Center</h3>
            <p>I can help you:</p>
            <ul>
              <li>Create new marketing campaigns</li>
              <li>Get campaign best practices and playbooks</li>
              <li>Set up projects in Workfront</li>
              <li>Check campaign status</li>
            </ul>
            <p>How can I assist you today?</p>
          </div>
        )}

        {messages.map((msg, idx) => (
          <div key={idx} className={`message ${msg.role}`}>
            <div className="message-content">
              {msg.content.split('\n').map((line, i) => (
                <p key={i}>{line}</p>
              ))}
            </div>
            <span className="timestamp">
              {new Date(msg.timestamp).toLocaleTimeString()}
            </span>
          </div>
        ))}

        {isLoading && (
          <div className="message assistant loading">
            <div className="typing-indicator">
              <span></span>
              <span></span>
              <span></span>
            </div>
          </div>
        )}

        <div ref={messagesEndRef} />
      </div>

      <div className="input-container">
        <textarea
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyPress={handleKeyPress}
          placeholder="Type your message..."
          disabled={isLoading}
          rows={2}
        />
        <button onClick={handleSend} disabled={isLoading || !input.trim()}>
          Send
        </button>
      </div>
    </div>
  );
};
```

#### Day 5: Campaign List Dashboard

```typescript
// frontend/src/components/CampaignDashboard.tsx
import React from 'react';
import { useQuery, useSubscription } from '@apollo/client';
import { LIST_CAMPAIGNS, ON_CAMPAIGN_UPDATED } from '../graphql/queries';

interface Campaign {
  campaignId: string;
  name: string;
  type: string;
  tier: number;
  status: string;
  workfrontUrl?: string;
  createdAt: string;
}

export const CampaignDashboard: React.FC = () => {
  const { data, loading, error, refetch } = useQuery(LIST_CAMPAIGNS);

  // Subscribe to real-time updates
  useSubscription(ON_CAMPAIGN_UPDATED, {
    onData: () => refetch(),
  });

  if (loading) return <div className="loading">Loading campaigns...</div>;
  if (error) return <div className="error">Error loading campaigns</div>;

  const campaigns: Campaign[] = data?.listCampaigns?.items || [];

  const statusColors: Record<string, string> = {
    PLANNING: '#3b82f6',
    ACTIVE: '#22c55e',
    PAUSED: '#f59e0b',
    COMPLETED: '#6b7280',
  };

  const tierLabels: Record<number, string> = {
    1: 'Tier 1 - Major',
    2: 'Tier 2 - Medium',
    3: 'Tier 3 - BAU',
  };

  return (
    <div className="dashboard-container">
      <div className="dashboard-header">
        <h2>Campaign Overview</h2>
        <div className="stats">
          <div className="stat">
            <span className="stat-value">{campaigns.length}</span>
            <span className="stat-label">Total Campaigns</span>
          </div>
          <div className="stat">
            <span className="stat-value">
              {campaigns.filter(c => c.status === 'ACTIVE').length}
            </span>
            <span className="stat-label">Active</span>
          </div>
          <div className="stat">
            <span className="stat-value">
              {campaigns.filter(c => c.tier === 1).length}
            </span>
            <span className="stat-label">Tier 1</span>
          </div>
        </div>
      </div>

      <div className="campaigns-table">
        <table>
          <thead>
            <tr>
              <th>Campaign Name</th>
              <th>Type</th>
              <th>Tier</th>
              <th>Status</th>
              <th>Created</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {campaigns.map(campaign => (
              <tr key={campaign.campaignId}>
                <td className="campaign-name">{campaign.name}</td>
                <td>
                  <span className="type-badge">{campaign.type}</span>
                </td>
                <td>
                  <span className="tier-badge" data-tier={campaign.tier}>
                    {tierLabels[campaign.tier]}
                  </span>
                </td>
                <td>
                  <span
                    className="status-badge"
                    style={{ backgroundColor: statusColors[campaign.status] }}
                  >
                    {campaign.status}
                  </span>
                </td>
                <td>{new Date(campaign.createdAt).toLocaleDateString()}</td>
                <td>
                  {campaign.workfrontUrl && (
                    <a
                      href={campaign.workfrontUrl}
                      target="_blank"
                      rel="noopener noreferrer"
                      className="action-link"
                    >
                      View in Workfront
                    </a>
                  )}
                </td>
              </tr>
            ))}
          </tbody>
        </table>

        {campaigns.length === 0 && (
          <div className="empty-state">
            <p>No campaigns yet. Start by chatting with the Campaign Assistant!</p>
          </div>
        )}
      </div>
    </div>
  );
};
```

---

## Phase 3: Integration & Testing (Weeks 7-9)

### Week 7: End-to-End Integration

#### Day 1-2: Complete Workflow Testing

```python
# tests/e2e/test_campaign_workflow.py
import pytest
import boto3
import time
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

class TestCampaignWorkflow:
    """End-to-end tests for campaign creation workflow."""

    @pytest.fixture
    def browser(self):
        options = webdriver.ChromeOptions()
        options.add_argument('--headless')
        driver = webdriver.Chrome(options=options)
        yield driver
        driver.quit()

    @pytest.fixture
    def dynamodb(self):
        return boto3.resource('dynamodb')

    def test_full_campaign_creation_via_chat(self, browser, dynamodb):
        """Test creating a campaign through chat interface."""
        # Navigate to app
        browser.get('https://poc.marketing-command-center.example.com')

        # Login (assuming Cognito hosted UI)
        # ... login steps ...

        # Wait for chat interface
        chat_input = WebDriverWait(browser, 10).until(
            EC.presence_of_element_located((By.CSS_SELECTOR, '.input-container textarea'))
        )

        # Send initial message
        chat_input.send_keys("I want to create a new email campaign for SMB customers")
        browser.find_element(By.CSS_SELECTOR, '.input-container button').click()

        # Wait for response
        time.sleep(5)  # Wait for agent response

        # Provide campaign details
        chat_input = browser.find_element(By.CSS_SELECTOR, '.input-container textarea')
        chat_input.send_keys(
            "Campaign name: E2E Test Campaign. "
            "Target: Small business owners. "
            "Objective: Increase sign-ups by 10%. "
            "Budget: $20,000. "
            "Launch: March 1, 2026."
        )
        browser.find_element(By.CSS_SELECTOR, '.input-container button').click()

        time.sleep(5)

        # Confirm creation
        chat_input = browser.find_element(By.CSS_SELECTOR, '.input-container textarea')
        chat_input.send_keys("Yes, please create this campaign in Workfront.")
        browser.find_element(By.CSS_SELECTOR, '.input-container button').click()

        # Wait for creation
        time.sleep(10)

        # Verify campaign appears in dashboard
        browser.find_element(By.CSS_SELECTOR, 'a[href="/dashboard"]').click()

        WebDriverWait(browser, 10).until(
            EC.presence_of_element_located((By.XPATH, "//td[contains(text(), 'E2E Test Campaign')]"))
        )

        # Verify in DynamoDB
        campaigns_table = dynamodb.Table('mcc-campaigns')
        response = campaigns_table.scan(
            FilterExpression='#n = :name',
            ExpressionAttributeNames={'#n': 'name'},
            ExpressionAttributeValues={':name': 'E2E Test Campaign'}
        )

        assert len(response['Items']) == 1
        campaign = response['Items'][0]
        assert campaign['type'] == 'standard'
        assert campaign['workfrontProjectId'] is not None

    def test_workfront_sync_after_update(self, dynamodb):
        """Test that Workfront updates sync back to our system."""
        # This would require:
        # 1. Create campaign via agent
        # 2. Update status in Workfront directly
        # 3. Verify webhook triggers
        # 4. Verify DynamoDB record updated
        pass
```

#### Day 3-5: Performance & Load Testing

```python
# tests/load/locustfile.py
from locust import HttpUser, task, between
import json
import uuid

class ChatUser(HttpUser):
    """Simulated user interacting with chat interface."""

    wait_time = between(1, 5)

    def on_start(self):
        """Authenticate user."""
        # Get Cognito token
        self.token = self._get_auth_token()
        self.conversation_id = None

    def _get_auth_token(self):
        # Implement Cognito authentication
        pass

    @task(3)
    def send_chat_message(self):
        """Send a message to the agent."""
        messages = [
            "What are the steps for creating a Tier 2 campaign?",
            "Help me plan an email campaign",
            "What's the checklist for launching a campaign?",
        ]

        payload = {
            "query": """
                mutation SendMessage($conversationId: ID, $message: String!) {
                    sendAgentMessage(conversationId: $conversationId, message: $message) {
                        conversationId
                        content
                    }
                }
            """,
            "variables": {
                "conversationId": self.conversation_id,
                "message": messages[hash(str(uuid.uuid4())) % len(messages)]
            }
        }

        headers = {
            "Authorization": f"Bearer {self.token}",
            "Content-Type": "application/json"
        }

        with self.client.post(
            "/graphql",
            json=payload,
            headers=headers,
            catch_response=True
        ) as response:
            if response.status_code == 200:
                data = response.json()
                if 'errors' in data:
                    response.failure(f"GraphQL error: {data['errors']}")
                else:
                    self.conversation_id = data['data']['sendAgentMessage']['conversationId']
                    response.success()
            else:
                response.failure(f"HTTP {response.status_code}")

    @task(1)
    def list_campaigns(self):
        """Load campaign dashboard."""
        payload = {
            "query": """
                query ListCampaigns {
                    listCampaigns(limit: 50) {
                        items {
                            campaignId
                            name
                            status
                        }
                    }
                }
            """
        }

        headers = {
            "Authorization": f"Bearer {self.token}",
            "Content-Type": "application/json"
        }

        self.client.post("/graphql", json=payload, headers=headers)
```

---

### Week 8: Pilot User Testing

#### Day 1-2: Pilot Setup

```markdown
# Pilot User Guide

## Overview
You are participating in the POC pilot for the Marketing Command Center.
This system allows you to create and manage campaigns using natural language.

## Getting Started

1. **Access**: Navigate to https://poc.mcc.example.com
2. **Login**: Use your corporate SSO credentials
3. **Interface**: You'll see a chat interface and a campaign dashboard

## What to Test

### Scenario 1: Create a Simple Email Campaign
1. Open the chat interface
2. Tell the assistant you want to create an email campaign
3. Provide details when asked:
   - Campaign name
   - Target audience
   - Objective
   - Budget
   - Timeline
4. Confirm when ready
5. Verify the campaign appears in Workfront

### Scenario 2: Ask About Best Practices
1. Ask the assistant about email campaign best practices
2. Ask about campaign tier definitions
3. Ask for the launch checklist

### Scenario 3: Check Campaign Status
1. After creating a campaign, ask for its status
2. Verify the information matches Workfront

## Feedback Form
After each session, please complete the feedback form at:
https://forms.example.com/mcc-pilot-feedback

Rate the following (1-5):
- Ease of use
- Response quality
- Time saved
- Overall satisfaction

Describe any issues encountered.
```

#### Day 3-5: Collect & Analyze Feedback

```python
# scripts/analyze_pilot_feedback.py
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime

def analyze_feedback(feedback_file: str):
    """Analyze pilot user feedback."""

    df = pd.read_csv(feedback_file)

    # Overall satisfaction metrics
    metrics = ['ease_of_use', 'response_quality', 'time_saved', 'overall_satisfaction']

    print("=== Pilot Feedback Analysis ===\n")

    for metric in metrics:
        avg = df[metric].mean()
        median = df[metric].median()
        print(f"{metric.replace('_', ' ').title()}")
        print(f"  Average: {avg:.2f} / 5")
        print(f"  Median: {median}")
        print(f"  Distribution: {df[metric].value_counts().sort_index().to_dict()}")
        print()

    # Success rate
    success_rate = (df['campaign_created_successfully'].sum() / len(df)) * 100
    print(f"Campaign Creation Success Rate: {success_rate:.1f}%\n")

    # Common issues
    print("Common Issues Reported:")
    issues = df['issues_encountered'].dropna()
    issue_counts = issues.str.lower().value_counts()
    for issue, count in issue_counts.head(5).items():
        print(f"  - {issue}: {count} occurrences")

    # Time comparison
    if 'estimated_time_saved_minutes' in df.columns:
        avg_time_saved = df['estimated_time_saved_minutes'].mean()
        print(f"\nAverage Estimated Time Saved: {avg_time_saved:.0f} minutes per campaign")

    # Generate report
    generate_report(df, metrics)

def generate_report(df: pd.DataFrame, metrics: list):
    """Generate visual report."""

    fig, axes = plt.subplots(2, 2, figsize=(12, 10))

    for i, metric in enumerate(metrics):
        ax = axes[i // 2, i % 2]
        df[metric].hist(ax=ax, bins=5, range=(1, 5), edgecolor='black')
        ax.set_title(metric.replace('_', ' ').title())
        ax.set_xlabel('Rating')
        ax.set_ylabel('Count')

    plt.tight_layout()
    plt.savefig(f'pilot_feedback_report_{datetime.now().strftime("%Y%m%d")}.png')
    print("\nReport saved to pilot_feedback_report_*.png")

if __name__ == "__main__":
    analyze_feedback('pilot_feedback.csv')
```

---

### Week 9: Refinement & Documentation

#### Day 1-3: Address Feedback Issues

```markdown
# Issue Tracking - POC Refinements

## High Priority

### ISSUE-001: Agent sometimes hallucinates campaign details
- **Reported by**: 3 pilot users
- **Description**: Agent occasionally makes up details not provided by user
- **Root cause**: Insufficient grounding in instructions
- **Fix**: Add explicit instruction to only use provided information
- **Status**: FIXED

### ISSUE-002: Workfront sync delay > 5 minutes
- **Reported by**: 2 pilot users
- **Description**: Updates in Workfront take too long to appear
- **Root cause**: Webhook processing delay
- **Fix**: Optimize Lambda cold start, add connection pooling
- **Status**: IN PROGRESS

### ISSUE-003: Knowledge base missing event playbook
- **Reported by**: 1 pilot user
- **Description**: Asked about event planning, no relevant info returned
- **Root cause**: Event playbook not uploaded
- **Fix**: Upload event marketing documentation
- **Status**: FIXED

## Medium Priority

### ISSUE-004: Chat history lost on page refresh
- **Description**: Refreshing loses conversation context
- **Fix**: Implement conversation persistence in frontend
- **Status**: BACKLOG

### ISSUE-005: No confirmation before Workfront actions
- **Description**: Agent should always confirm before creating projects
- **Fix**: Add confirmation step in agent instructions
- **Status**: FIXED
```

#### Day 4-5: Final Documentation

```markdown
# POC Summary Report

## Executive Summary

The Marketing Command Center POC successfully demonstrated the feasibility of
AI-powered campaign creation with Workfront integration. Over 3 weeks of pilot
testing with 10 users, we achieved:

- **85% user satisfaction** (target: 80%)
- **8-minute average campaign creation** (target: < 10 minutes)
- **92% campaign creation success rate**
- **78% knowledge retrieval accuracy** (target: 75%)

## Key Findings

### What Worked Well
1. Natural language campaign creation significantly reduces friction
2. Knowledge base integration provides consistent guidance
3. Workfront synchronization is reliable after initial setup
4. Users reported high satisfaction with response quality

### Areas for Improvement
1. Agent response latency (avg 4.2s, target < 3s)
2. More comprehensive knowledge base content needed
3. Better handling of complex, multi-channel campaigns
4. Need for approval workflows before critical actions

### Technical Debt
1. Workfront client needs connection pooling
2. Agent instructions require further tuning
3. Error handling in UI needs improvement
4. Logging and monitoring gaps identified

## Recommendations

### Go / No-Go: GO with conditions

**Conditions for Production**:
1. Complete Asana integration (Phase 2)
2. Implement approval workflows
3. Expand knowledge base content
4. Add comprehensive monitoring
5. Security audit required

### Next Steps
1. Present POC results to stakeholders (Week 10)
2. Begin Phase 2 planning (Week 11)
3. Budget approval for production build
4. Expand pilot to 50 users

## Metrics Summary

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| Campaign creation time | < 10 min | 8 min | PASS |
| User satisfaction | 80% | 85% | PASS |
| Sync latency | < 2 min | 1.8 min | PASS |
| Knowledge accuracy | 75% | 78% | PASS |
| System availability | 99% | 99.7% | PASS |
```

---

## Phase 4: Handoff & Planning (Weeks 10-12)

### Week 10: Stakeholder Presentation

- Present POC results
- Demo live system
- Review metrics against success criteria
- Discuss production roadmap

### Week 11: Production Planning

- Detailed architecture review
- Security assessment
- Cost projections
- Team requirements
- Timeline for production build

### Week 12: Documentation & Handoff

- Complete technical documentation
- Runbooks for operations
- Knowledge transfer sessions
- Production backlog creation

---

## Appendix A: POC Infrastructure Checklist

```markdown
## Pre-POC Setup (Week 0)

- [ ] AWS Account provisioned
- [ ] Service quotas increased (Bedrock, Lambda)
- [ ] VPC and networking configured
- [ ] IAM roles and policies created
- [ ] Workfront sandbox access obtained
- [ ] Cognito user pool configured
- [ ] Development environment set up

## Week 1 Checklist

- [ ] CDK project initialized
- [ ] Network stack deployed
- [ ] Data stack deployed (DynamoDB, S3)
- [ ] Secrets configured

## Week 2 Checklist

- [ ] Workfront client implemented
- [ ] Webhook handler deployed
- [ ] Integration tests passing
- [ ] Event bus configured

## Week 3 Checklist

- [ ] Knowledge base created
- [ ] Initial documents uploaded
- [ ] Retrieval tests passing

## Week 4-5 Checklist

- [ ] Bedrock agent created
- [ ] Action groups configured
- [ ] Agent tests passing
- [ ] Prompt tuning complete

## Week 6 Checklist

- [ ] AppSync API deployed
- [ ] Frontend deployed
- [ ] E2E tests passing

## Week 7-9 Checklist

- [ ] Pilot users onboarded
- [ ] Feedback collected
- [ ] Critical issues resolved
- [ ] Documentation complete
```

---

## Appendix B: Cost Tracking

```markdown
## POC Cost Tracking (12 weeks)

| Week | Service | Cost | Notes |
|------|---------|------|-------|
| 1-3 | DynamoDB | $50 | On-demand, low usage |
| 1-3 | Lambda | $20 | Minimal invocations |
| 1-3 | S3 | $5 | Document storage |
| 4-6 | Bedrock | $500 | Agent development |
| 4-6 | OpenSearch | $400 | Vector store |
| 4-6 | Q Business | $800 | 10 test users |
| 7-9 | Bedrock | $1,500 | Pilot usage |
| 7-9 | All other | $300 | Increased usage |
| 10-12 | All | $800 | Maintenance mode |
| **Total** | | **~$4,375** | Under $5K target |
```
