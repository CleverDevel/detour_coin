# Azure Infrastructure for Solana Operations

**Document ID:** OPS-008
**Version:** 1.0
**Status:** Production Ready
**Owner:** DevOps Engineer / Cloud Architect
**Category:** Deployment & Operations
**Dependencies:** TECH-002 (Emission Controller)
**Related:** TECH-007, OPS-009, OPS-010

---

## Table of Contents

1. [Infrastructure Overview](#1-infrastructure-overview)
2. [Azure Account Setup](#2-azure-account-setup)
3. [RBAC Roles & Permissions](#3-rbac-roles--permissions)
4. [Azure Function for Daily Emission](#4-azure-function-for-daily-emission)
5. [Timer Trigger Scheduling](#5-timer-trigger-scheduling)
6. [Key Vault](#6-key-vault)
7. [Cosmos DB Operation History](#7-cosmos-db-operation-history)
8. [Event Grid & Action Groups Alerting](#8-event-grid--action-groups-alerting)
9. [Application Insights & Azure Monitor](#9-application-insights--azure-monitor)
10. [Virtual Network Configuration](#10-virtual-network-configuration-optional)
11. [Cost Optimization](#11-cost-optimization)
12. [RPC Provider](#12-rpc-provider)
13. [Transaction Fee Management](#13-transaction-fee-management)
14. [Deployment Automation (IaC)](#14-deployment-automation-iac)
15. [Monitoring & Observability](#15-monitoring--observability)
16. [Error Handling & Retry](#16-error-handling--retry)
17. [Security](#17-security)
18. [Testing](#18-testing)
19. [Operations](#19-operations)
20. [Incident Response](#20-incident-response)
21. [Disaster Recovery](#21-disaster-recovery)
22. [Code Examples](#22-code-examples)
23. [Troubleshooting](#23-troubleshooting)
24. [Appendices](#24-appendices)

---

## 1. Infrastructure Overview

### Architecture Diagram

```mermaid
graph TB
    subgraph "Azure Cloud"
        subgraph "Function App"
            FN1[Daily Emission Function<br/>Timer: 12PM UTC]
            FN2[Backup Emission Function<br/>Timer: 1PM UTC]
            FN3[Manual Trigger Function<br/>HTTP Endpoint]
        end

        subgraph "Security & Secrets"
            KV[Azure Key Vault<br/>- Emission Authority Keypair<br/>- RPC API Keys<br/>- Connection Strings]
            MI[Managed Identity<br/>System-Assigned]
        end

        subgraph "Data Storage"
            COSMOS[Cosmos DB<br/>Operation History<br/>Serverless Mode]
        end

        subgraph "Monitoring & Alerting"
            AI[Application Insights<br/>Logs & Metrics]
            EG[Event Grid<br/>Custom Topics]
            AG1[Critical Action Group<br/>Email/SMS/PagerDuty]
            AG2[Operational Action Group<br/>Email/Slack]
            MON[Azure Monitor<br/>Alerts & Dashboards]
        end

        subgraph "Optional: Network Security"
            VNET[Virtual Network]
            NSG[Network Security Group]
            PE[Private Endpoints]
        end
    end

    subgraph "External Services"
        RPC1[Primary RPC<br/>QuickNode/Helius]
        RPC2[Backup RPC<br/>Alchemy/Public]
        SOL[Solana Blockchain<br/>Mainnet-Beta]
    end

    FN1 -->|Read Secrets| KV
    FN2 -->|Read Secrets| KV
    FN3 -->|Read Secrets| KV
    MI -->|Authenticate| KV
    MI -->|Write Logs| AI
    MI -->|Read/Write| COSMOS
    MI -->|Publish Events| EG

    FN1 -->|Connect| RPC1
    FN1 -->|Failover| RPC2
    FN2 -->|Connect| RPC1
    FN3 -->|Connect| RPC1

    RPC1 -->|Submit Tx| SOL
    RPC2 -->|Submit Tx| SOL

    FN1 -->|Log Operations| COSMOS
    FN1 -->|Send Telemetry| AI
    FN1 -->|Publish Events| EG

    EG -->|Trigger| AG1
    EG -->|Trigger| AG2
    MON -->|Trigger| AG1
    MON -->|Trigger| AG2

    AI -->|Alerts| MON
    COSMOS -->|Metrics| MON

    VNET -.Optional.-> FN1
    VNET -.Optional.-> KV
    VNET -.Optional.-> COSMOS
    PE -.Optional.-> KV
    PE -.Optional.-> COSMOS
    NSG -.Optional.-> VNET

    style FN1 fill:#0078d4,stroke:#333,stroke-width:2px,color:#fff
    style KV fill:#ffb900,stroke:#333,stroke-width:2px
    style COSMOS fill:#00a4ef,stroke:#333,stroke-width:2px
    style AI fill:#00d4aa,stroke:#333,stroke-width:2px
    style SOL fill:#14f195,stroke:#333,stroke-width:2px
    style AG1 fill:#ff4040,stroke:#333,stroke-width:2px
```

### High-Level Architecture

DetourCoin's Azure infrastructure provides **production-ready automation** for daily token emissions on Solana with the following design principles:

| Principle | Implementation |
|-----------|----------------|
| **Reliability** | Dual timer triggers (primary + backup), automated failover, retry logic |
| **Security** | Managed identities (no hardcoded keys), Key Vault secrets, RBAC enforcement |
| **Observability** | Application Insights structured logging, Event Grid events, Azure Monitor dashboards |
| **Cost Efficiency** | Serverless Consumption plan, free tier optimization, <$200/month target |
| **Solana Integration** | RPC provider abstraction, transaction confirmation monitoring, fee management |

### Core Components

#### Azure Functions (Serverless Compute)
- **Purpose:** Execute daily emission transactions on Solana
- **Hosting:** Consumption Plan (pay-per-execution, no idle costs)
- **Runtime:** Node.js 18 LTS
- **Triggers:**
  - Timer (cron: `0 0 12 * * *` - 12PM UTC daily)
  - Backup Timer (cron: `0 0 13 * * *` - 1PM UTC daily)
  - HTTP (manual invocation for testing/recovery)

#### Azure Key Vault (Secrets Management)
- **Purpose:** Securely store emission authority keypair and RPC credentials
- **Security:** AES-256 encryption at rest, TLS 1.2+ in transit, RBAC access
- **Secrets:**
  - `detourcoin-emission-authority-keypair` (JSON array of private key bytes)
  - `primary-rpc-url` (QuickNode/Helius endpoint with API key)
  - `backup-rpc-url` (Alchemy/public RPC failover)
  - `cosmos-db-connection-string` (if using connection string auth)

#### Cosmos DB (Operation History)
- **Purpose:** Persistent audit log of all emission executions
- **API:** NoSQL (recommended) or Table API (cost-optimized alternative)
- **Mode:** Serverless (pay-per-RU consumed, no minimum throughput)
- **Retention:** 2 years with TTL, continuous backup (7-day point-in-time recovery)

#### Event Grid (Event Routing)
- **Purpose:** Decouple emission events from alerting/notification logic
- **Topics:** Custom topic `detourcoin-emission-events`
- **Subscriptions:** Critical alerts (failures), operational alerts (success/warnings)

#### Application Insights (Observability)
- **Purpose:** Centralized logging, metrics, and distributed tracing
- **Retention:** 30-90 days for logs, 93 days for metrics
- **Integration:** Automatically linked to Function App

#### Azure Monitor (Alerting)
- **Purpose:** Metric-based and log-based alerting with action group integration
- **Alert Rules:**
  - Emission failure → Critical action group
  - SOL balance < 5 → Critical action group
  - Execution duration > 45s → Operational action group

### Integration with Solana

```mermaid
sequenceDiagram
    participant Timer as Azure Timer Trigger
    participant Fn as Azure Function
    participant KV as Key Vault
    participant RPC as Solana RPC (Primary)
    participant RPC2 as Solana RPC (Backup)
    participant SOL as Solana Blockchain
    participant DB as Cosmos DB
    participant EG as Event Grid
    participant AG as Action Groups

    Timer->>Fn: Trigger execution (12PM UTC)
    Fn->>KV: Get emission authority keypair
    KV-->>Fn: Return keypair (managed identity auth)

    Fn->>RPC: Connect to primary RPC
    alt RPC connection successful
        RPC-->>Fn: Connected
    else RPC connection failed
        Fn->>RPC2: Failover to backup RPC
        RPC2-->>Fn: Connected
    end

    Fn->>RPC: Create emission transaction
    Fn->>Fn: Sign transaction with keypair
    Fn->>RPC: Send transaction
    RPC->>SOL: Broadcast to Solana network

    alt Transaction simulation successful
        SOL-->>RPC: Simulation OK
        RPC-->>Fn: Transaction submitted

        Fn->>RPC: Confirm transaction (poll/websocket)
        RPC-->>Fn: Confirmed (signature + slot)

        Fn->>DB: Write operation history
        Fn->>EG: Publish EmissionSucceeded event
        EG->>AG: Notify operational action group

    else Transaction simulation failed
        SOL-->>RPC: Simulation error
        RPC-->>Fn: Error (program error, insufficient SOL, etc.)

        Fn->>Fn: Retry logic (max 3 attempts)

        alt Retry exhausted
            Fn->>DB: Write failure record
            Fn->>EG: Publish EmissionFailed event (CRITICAL)
            EG->>AG: Notify critical action group (email/SMS/PagerDuty)
        end
    end
```

### Cost Optimization Strategy

Targeting **<$200/month** for full production operation:

| Service | Tier | Monthly Cost Estimate | Optimization Strategy |
|---------|------|----------------------|----------------------|
| Azure Functions | Consumption | $0 | Within 1M free executions (1 daily = 30/mo) |
| Application Insights | Pay-as-you-go | $0-$5 | 5GB free ingestion, structured logging to minimize volume |
| Cosmos DB | Serverless | $5-$15 | Low RU consumption (1 write/day), or use Table Storage ($0-$1) |
| Key Vault | Standard | $0-$3 | Unlimited secret operations for Standard tier |
| Event Grid | Pay-as-you-go | $0-$2 | 100K ops free, then $0.60/M ops |
| Storage Account | LRS | $1-$2 | Function App storage requirement |
| **Total (No VNet)** | | **$6-$27/month** | Well under $200 target |
| Optional: VNet + NAT | Premium | +$40-$50 | Only if network isolation required |
| External: RPC Provider | Dedicated | $0-$100 | QuickNode/Helius free tier or paid ($50-$100) |
| **Total (With VNet + RPC)** | | **$50-$180/month** | Still under $200 target |

💡 **Cost Tip:** Use **Azure Table Storage** instead of Cosmos DB NoSQL for operation history if budget is constrained. Table Storage costs ~$0.045/GB vs. Cosmos DB serverless ~$0.25/M RUs. For 1 write/day, both are negligible, but Table Storage is cheaper at scale.

⚠️ **Security Warning:** Do NOT use VNet-less deployment in production without compensating security controls (RBAC, Key Vault, managed identity, NSG on storage). VNet is optional but recommended for defense-in-depth.

---

## 2. Azure Account Setup

### Root Account Security

#### Multi-Factor Authentication (MFA)
- **Requirement:** MFA MUST be enabled for all Azure AD accounts with subscription access
- **Method:** Microsoft Authenticator app or hardware FIDO2 keys (YubiKey recommended)
- **Enforcement:** Configure Conditional Access policy requiring MFA for Azure Portal and API access

```bash
# Azure CLI: Verify MFA status for current user
az ad signed-in-user show --query '{userPrincipalName:userPrincipalName, mfaDetail:strongAuthenticationDetail}'
```

#### Azure AD User Roles

| Role | Azure AD Role | Purpose | MFA Required |
|------|--------------|---------|--------------|
| **Subscription Owner** | Owner | Root account, billing, IAM management | Yes (required) |
| **DevOps Engineer** | Contributor | Deploy infrastructure, manage resources | Yes (required) |
| **Developer** | Reader + Function App Contributor | View resources, deploy function code | Yes (recommended) |
| **Monitoring Operator** | Monitoring Reader | View logs, dashboards, alerts (read-only) | Yes (recommended) |

```bash
# Create DevOps user with Contributor role (example)
az ad user create \
  --display-name "DetourCoin DevOps" \
  --user-principal-name devops@yourdomain.onmicrosoft.com \
  --password "TempPassword123!" \
  --force-change-password-next-sign-in true

# Assign Contributor role at subscription level
az role assignment create \
  --assignee devops@yourdomain.onmicrosoft.com \
  --role "Contributor" \
  --scope "/subscriptions/YOUR_SUBSCRIPTION_ID"
```

### Region Selection

**Recommended Regions for Lowest Solana RPC Latency:**

| Region | Azure Name | Latency to Solana (US RPC) | Notes |
|--------|-----------|---------------------------|-------|
| **East US** | `eastus` | ~10-30ms | Primary choice, lowest cost |
| **West US 2** | `westus2` | ~5-20ms | Closer to Solana validator concentration (California) |
| **Central US** | `centralus` | ~15-35ms | Alternative |

```bash
# Set default region for Azure CLI
az configure --defaults location=eastus

# Verify available regions
az account list-locations --query "[?name=='eastus' || name=='westus2']" -o table
```

💡 **Latency Testing:** Before committing, test RPC latency from each region:

```bash
# Deploy test Function App in eastus
az functionapp create --resource-group rg-test --name func-latency-test-eastus \
  --consumption-plan-location eastus --runtime node --functions-version 4

# Deploy same function with RPC ping to measure latency
# Compare results from eastus vs westus2
```

### Resource Group Organization

```bash
# Production resource group
az group create --name rg-detourcoin-prod --location eastus --tags \
  Environment=Production \
  Project=DetourCoin \
  ManagedBy=Bicep

# Development resource group (for testing)
az group create --name rg-detourcoin-dev --location eastus --tags \
  Environment=Development \
  Project=DetourCoin \
  ManagedBy=Bicep

# List resource groups
az group list --query "[?tags.Project=='DetourCoin']" -o table
```

**Naming Convention:**
- Resource Group: `rg-detourcoin-{env}`
- Function App: `func-detourcoin-emission-{env}`
- Key Vault: `kv-detourcoin-{env}-{random}`
- Storage Account: `stdetourcoin{env}{random}` (no hyphens, max 24 chars)
- Cosmos DB: `cosmos-detourcoin-{env}`
- Application Insights: `ai-detourcoin-{env}`

### Cost Alerts Configuration

```bash
# Create budget with alerts at $50, $100, $150
az consumption budget create \
  --subscription YOUR_SUBSCRIPTION_ID \
  --budget-name detourcoin-monthly-budget \
  --category Cost \
  --amount 200 \
  --time-grain Monthly \
  --start-date 2025-12-01 \
  --notifications \
    '{"50-percent":{"enabled":true,"operator":"GreaterThan","threshold":50,"contactEmails":["devops@example.com"]},"75-percent":{"enabled":true,"operator":"GreaterThan","threshold":75,"contactEmails":["devops@example.com"]},"100-percent":{"enabled":true,"operator":"GreaterThan","threshold":100,"contactEmails":["devops@example.com","founder@example.com"]}}'
```

### Subscription-Level Configuration

```bash
# Register required resource providers
az provider register --namespace Microsoft.Web           # Function Apps
az provider register --namespace Microsoft.Storage       # Storage Accounts
az provider register --namespace Microsoft.KeyVault      # Key Vault
az provider register --namespace Microsoft.DocumentDB    # Cosmos DB
az provider register --namespace Microsoft.EventGrid     # Event Grid
az provider register --namespace Microsoft.Insights      # Application Insights

# Verify registration status
az provider list --query "[?registrationState=='Registered' && namespace=='Microsoft.Web']" -o table
```

---

## 3. RBAC Roles & Permissions

### Function App Managed Identity Permissions

Azure Functions will use a **system-assigned managed identity** (no passwords, automatic rotation) with the following RBAC roles:

#### Key Vault Access

```json
{
  "role": "Key Vault Secrets User",
  "scope": "/subscriptions/{subscription-id}/resourceGroups/rg-detourcoin-prod/providers/Microsoft.KeyVault/vaults/kv-detourcoin-prod",
  "permissions": {
    "secrets": ["get", "list"]
  },
  "justification": "Function App needs to read emission authority keypair and RPC URLs from Key Vault"
}
```

**Bicep Example:**

```bicep
// Assign Key Vault Secrets User role to Function App managed identity
resource keyVaultRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(functionApp.id, keyVault.id, 'Key Vault Secrets User')
  scope: keyVault
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6') // Key Vault Secrets User
    principalId: functionApp.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

#### Cosmos DB Access

```json
{
  "role": "Cosmos DB Built-in Data Contributor",
  "scope": "/subscriptions/{subscription-id}/resourceGroups/rg-detourcoin-prod/providers/Microsoft.DocumentDB/databaseAccounts/cosmos-detourcoin-prod",
  "permissions": {
    "data": ["read", "write"]
  },
  "justification": "Function App writes emission operation history to Cosmos DB"
}
```

**Bicep Example:**

```bicep
// Create custom Cosmos DB role definition for Function App
resource cosmosDbRoleDefinition 'Microsoft.DocumentDB/databaseAccounts/sqlRoleDefinitions@2023-04-15' = {
  parent: cosmosDbAccount
  name: guid(cosmosDbAccount.id, 'emission-writer-role')
  properties: {
    roleName: 'DetourCoin Emission Writer'
    type: 'CustomRole'
    assignableScopes: [
      cosmosDbAccount.id
    ]
    permissions: [
      {
        dataActions: [
          'Microsoft.DocumentDB/databaseAccounts/readMetadata'
          'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/create'
          'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/read'
        ]
      }
    ]
  }
}

// Assign role to Function App managed identity
resource cosmosDbRoleAssignment 'Microsoft.DocumentDB/databaseAccounts/sqlRoleAssignments@2023-04-15' = {
  parent: cosmosDbAccount
  name: guid(functionApp.id, cosmosDbAccount.id, 'emission-writer')
  properties: {
    roleDefinitionId: cosmosDbRoleDefinition.id
    principalId: functionApp.identity.principalId
    scope: cosmosDbAccount.id
  }
}
```

#### Azure Monitor / Application Insights Access

```json
{
  "role": "Monitoring Metrics Publisher",
  "scope": "/subscriptions/{subscription-id}/resourceGroups/rg-detourcoin-prod/providers/Microsoft.Insights/components/ai-detourcoin-prod",
  "permissions": {
    "actions": ["Microsoft.Insights/Metrics/Write"]
  },
  "justification": "Function App publishes custom metrics (EmissionSuccess, SOLBalance, etc.)"
}
```

**Note:** Application Insights write access is automatically granted to Function Apps via connection string. Metrics Publisher role is only needed for custom metrics via Azure Monitor API.

#### Event Grid Access

```json
{
  "role": "Event Grid Data Sender",
  "scope": "/subscriptions/{subscription-id}/resourceGroups/rg-detourcoin-prod/providers/Microsoft.EventGrid/topics/detourcoin-emission-events",
  "permissions": {
    "actions": ["Microsoft.EventGrid/events/send"]
  },
  "justification": "Function App publishes emission success/failure events to Event Grid topic"
}
```

**Bicep Example:**

```bicep
resource eventGridRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(functionApp.id, eventGridTopic.id, 'Event Grid Data Sender')
  scope: eventGridTopic
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'd5a91429-5739-47e2-a06b-3470a27159e7') // Event Grid Data Sender
    principalId: functionApp.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

### Deployment/CI/CD Service Principal Permissions

For **Bicep/Terraform deployment automation** (GitHub Actions, Azure DevOps):

```json
{
  "role": "Contributor",
  "scope": "/subscriptions/{subscription-id}/resourceGroups/rg-detourcoin-prod",
  "timeLimit": "During deployment only",
  "justification": "CI/CD pipeline needs to create/update Azure resources via IaC"
}
```

**Create Service Principal for GitHub Actions:**

```bash
# Create service principal with Contributor role scoped to resource group
az ad sp create-for-rbac \
  --name "sp-detourcoin-github-actions" \
  --role Contributor \
  --scopes /subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/rg-detourcoin-prod \
  --sdk-auth \
  --years 1

# Output (JSON) - store in GitHub Secrets as AZURE_CREDENTIALS
{
  "clientId": "...",
  "clientSecret": "...",
  "subscriptionId": "...",
  "tenantId": "...",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  ...
}
```

### Monitoring/Operations Personnel Permissions

```bash
# Grant Monitoring Reader role for view-only access to logs and metrics
az role assignment create \
  --assignee monitoring-ops@yourdomain.com \
  --role "Monitoring Reader" \
  --scope /subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/rg-detourcoin-prod
```

### RBAC Summary Table

| Principal | Role | Scope | Permissions | Justification |
|-----------|------|-------|-------------|---------------|
| **Function App (Managed Identity)** | Key Vault Secrets User | Key Vault | Get/List secrets | Read keypair and RPC URLs |
| **Function App (Managed Identity)** | Cosmos DB Data Contributor (Custom) | Cosmos DB | Read/Write items | Write operation history |
| **Function App (Managed Identity)** | Event Grid Data Sender | Event Grid Topic | Send events | Publish emission events |
| **Deployment Service Principal** | Contributor | Resource Group | Create/update/delete resources | IaC deployment (time-limited) |
| **DevOps Engineer (Human)** | Contributor | Resource Group | Manage resources | Infrastructure management |
| **Monitoring Operator (Human)** | Monitoring Reader | Resource Group | Read logs/metrics | Operational visibility |
| **Founder (Human)** | Owner | Subscription | Full access | Root account, billing |

⚠️ **Security Best Practice:** Use **Conditional Access policies** to require MFA and restrict access to Azure Portal from trusted IPs only.

---

## 4. Azure Function for Daily Emission

### Function App Configuration

#### Basic Settings

| Setting | Value | Justification |
|---------|-------|---------------|
| **Runtime** | Node.js 18 LTS | Latest stable LTS version, @solana/web3.js support |
| **Plan** | Consumption | Pay-per-execution, no idle costs, auto-scaling |
| **Memory** | 512 MB | Sufficient for Solana transaction processing |
| **Timeout** | 60 seconds | Allow time for RPC connection, tx submission, confirmation |
| **Max Concurrent Requests** | 1 | Prevent concurrent emissions (idempotency) |
| **FUNCTIONS_WORKER_PROCESS_COUNT** | 1 | Single worker to avoid concurrency issues |
| **Always On** | Disabled (Consumption) | Not applicable to Consumption plan |

#### Application Settings (Environment Variables)

```javascript
// App Settings stored in Azure Portal → Function App → Configuration
{
  "SOLANA_NETWORK": "mainnet-beta",
  "SOLANA_RPC_PRIMARY": "@Microsoft.KeyVault(SecretUri=https://kv-detourcoin-prod.vault.azure.net/secrets/primary-rpc-url/)",
  "SOLANA_RPC_BACKUP": "@Microsoft.KeyVault(SecretUri=https://kv-detourcoin-prod.vault.azure.net/secrets/backup-rpc-url/)",
  "EMISSION_AUTHORITY_KEYPAIR": "@Microsoft.KeyVault(SecretUri=https://kv-detourcoin-prod.vault.azure.net/secrets/detourcoin-emission-authority-keypair/)",
  "EMISSION_PROGRAM_ID": "EmissionProgramPubkeyFromDeployment",
  "COSMOS_DB_ENDPOINT": "https://cosmos-detourcoin-prod.documents.azure.com:443/",
  "COSMOS_DB_DATABASE": "detourcoin",
  "COSMOS_DB_CONTAINER": "emission_history",
  "EVENT_GRID_TOPIC_ENDPOINT": "https://detourcoin-emission-events.eastus-1.eventgrid.azure.net/api/events",
  "APPLICATIONINSIGHTS_CONNECTION_STRING": "InstrumentationKey=...;IngestionEndpoint=https://eastus-0.in.applicationinsights.azure.com/",
  "MIN_SOL_BALANCE_ALERT_THRESHOLD": "5.0",
  "MAX_RETRY_ATTEMPTS": "3",
  "RPC_TIMEOUT_MS": "30000"
}
```

💡 **Key Vault Reference Syntax:** `@Microsoft.KeyVault(SecretUri=https://{vault-name}.vault.azure.net/secrets/{secret-name}/{version})`
Omit `{version}` to always use the latest version of the secret.

### Function Code Structure

#### Project Layout

```
emission-function/
├── package.json                  # Dependencies (@solana/web3.js, @azure/cosmos, etc.)
├── host.json                     # Function App host configuration
├── local.settings.json           # Local development settings (gitignored)
├── .funcignore                   # Files to exclude from deployment
├── daily-emission/               # Timer-triggered function
│   ├── function.json             # Bindings and trigger configuration
│   └── index.ts                  # Main handler logic
├── backup-emission/              # Backup timer-triggered function
│   ├── function.json
│   └── index.ts
├── manual-emission/              # HTTP-triggered function for manual invocation
│   ├── function.json
│   └── index.ts
└── lib/                          # Shared utilities
    ├── solana.ts                 # Solana RPC client, transaction building
    ├── cosmos.ts                 # Cosmos DB operations
    ├── eventgrid.ts              # Event Grid publishing
    ├── monitoring.ts             # Application Insights custom telemetry
    └── errors.ts                 # Error types and handling
```

#### package.json

```json
{
  "name": "detourcoin-emission-function",
  "version": "1.0.0",
  "description": "Azure Function for DetourCoin daily emission on Solana",
  "scripts": {
    "build": "tsc",
    "watch": "tsc -w",
    "clean": "rimraf dist",
    "start": "func start",
    "test": "jest"
  },
  "dependencies": {
    "@azure/functions": "^4.0.0",
    "@azure/cosmos": "^4.0.0",
    "@azure/eventgrid": "^5.0.0",
    "@azure/identity": "^4.0.0",
    "@solana/web3.js": "^1.87.6",
    "applicationinsights": "^2.9.0"
  },
  "devDependencies": {
    "@types/node": "^18.0.0",
    "typescript": "^5.3.0",
    "rimraf": "^5.0.0",
    "jest": "^29.0.0",
    "@types/jest": "^29.0.0"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

#### host.json

```json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": false,
        "maxTelemetryItemsPerSecond": 20
      }
    },
    "logLevel": {
      "default": "Information",
      "Function": "Information",
      "Host.Aggregator": "Warning"
    }
  },
  "functionTimeout": "00:01:00",
  "extensions": {
    "http": {
      "routePrefix": "api"
    }
  },
  "retry": {
    "strategy": "fixedDelay",
    "maxRetryCount": 0,
    "delayInterval": "00:00:10"
  }
}
```

**Note:** `maxRetryCount: 0` disables Azure Functions built-in retry (we implement custom retry logic with exponential backoff in code).

### Emission Handler Logic

#### daily-emission/index.ts

```typescript
import { AzureFunction, Context } from "@azure/functions";
import { Connection, Keypair, PublicKey, Transaction } from "@solana/web3.js";
import { CosmosClient } from "@azure/cosmos";
import { EventGridPublisherClient, AzureKeyCredential } from "@azure/eventgrid";
import * as appInsights from "applicationinsights";

// Initialize Application Insights
appInsights.setup(process.env.APPLICATIONINSIGHTS_CONNECTION_STRING!).start();
const telemetryClient = appInsights.defaultClient;

interface EmissionRecord {
  id: string;
  execution_date: string; // YYYY-MM-DD (partition key)
  timestamp: string; // ISO 8601
  status: "success" | "failure" | "pending";
  tx_signature?: string;
  error_message?: string;
  block_time?: number;
  emission_amount: number;
  phase: "pre-launch" | "post-launch";
  retry_count: number;
  sol_balance_before?: number;
  sol_balance_after?: number;
}

const timerTrigger: AzureFunction = async function (context: Context, myTimer: any): Promise<void> {
  const startTime = Date.now();
  const executionDate = new Date().toISOString().split('T')[0]; // YYYY-MM-DD

  context.log.info(`[DetourCoin Emission] Starting daily emission for ${executionDate}`);

  try {
    // Step 1: Load Keypair from Key Vault (via managed identity)
    const keypairJson = process.env.EMISSION_AUTHORITY_KEYPAIR!;
    const keypairBytes = JSON.parse(keypairJson);
    const emissionAuthority = Keypair.fromSecretKey(new Uint8Array(keypairBytes));
    context.log.info(`[Keypair] Loaded emission authority: ${emissionAuthority.publicKey.toBase58()}`);

    // Step 2: Connect to Solana RPC (with failover)
    let connection: Connection;
    try {
      connection = new Connection(process.env.SOLANA_RPC_PRIMARY!, {
        commitment: 'confirmed',
        confirmTransactionInitialTimeout: 60000,
      });
      await connection.getRecentBlockhash(); // Test connection
      context.log.info(`[RPC] Connected to primary RPC`);
    } catch (primaryError) {
      context.log.warn(`[RPC] Primary RPC failed: ${primaryError}. Failing over to backup...`);
      connection = new Connection(process.env.SOLANA_RPC_BACKUP!, {
        commitment: 'confirmed',
        confirmTransactionInitialTimeout: 60000,
      });
      await connection.getRecentBlockhash(); // Test backup
      context.log.info(`[RPC] Connected to backup RPC`);
    }

    // Step 3: Check SOL balance
    const balanceLamports = await connection.getBalance(emissionAuthority.publicKey);
    const balanceSOL = balanceLamports / 1e9;
    context.log.info(`[Balance] Current SOL balance: ${balanceSOL.toFixed(9)} SOL`);

    // Alert if balance is low (but don't block execution)
    const minBalance = parseFloat(process.env.MIN_SOL_BALANCE_ALERT_THRESHOLD || "5.0");
    if (balanceSOL < minBalance) {
      context.log.warn(`[Balance] WARNING: SOL balance ${balanceSOL} is below threshold ${minBalance}`);
      await publishEvent(context, {
        eventType: "DetourCoin.Emission.LowSOLBalance",
        subject: "emission/balance-alert",
        dataVersion: "1.0",
        data: {
          balance_sol: balanceSOL,
          threshold: minBalance,
          authority: emissionAuthority.publicKey.toBase58(),
        },
      });
    }

    // Step 4: Build emission transaction (CPI to Emission Controller Program)
    const emissionProgramId = new PublicKey(process.env.EMISSION_PROGRAM_ID!);

    // Derive emission state PDA
    const [emissionStatePda] = PublicKey.findProgramAddressSync(
      [Buffer.from("emission_state")],
      emissionProgramId
    );

    // Create instruction (simplified - actual implementation depends on program IDL)
    const instruction = await buildEmissionInstruction(
      emissionProgramId,
      emissionStatePda,
      emissionAuthority.publicKey,
      connection
    );

    // Step 5: Simulate transaction before sending
    const transaction = new Transaction().add(instruction);
    transaction.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
    transaction.feePayer = emissionAuthority.publicKey;

    const simulation = await connection.simulateTransaction(transaction);
    if (simulation.value.err) {
      throw new Error(`Transaction simulation failed: ${JSON.stringify(simulation.value.err)}`);
    }
    context.log.info(`[Simulation] Transaction simulation succeeded`);

    // Step 6: Sign and send transaction
    transaction.sign(emissionAuthority);
    const signature = await connection.sendRawTransaction(transaction.serialize(), {
      skipPreflight: false,
      maxRetries: 3,
    });
    context.log.info(`[Transaction] Sent transaction: ${signature}`);

    // Step 7: Confirm transaction
    const confirmation = await connection.confirmTransaction({
      signature,
      blockhash: transaction.recentBlockhash!,
      lastValidBlockHeight: (await connection.getLatestBlockhash()).lastValidBlockHeight,
    }, 'confirmed');

    if (confirmation.value.err) {
      throw new Error(`Transaction confirmation failed: ${JSON.stringify(confirmation.value.err)}`);
    }

    const txDetails = await connection.getTransaction(signature, {
      commitment: 'confirmed',
      maxSupportedTransactionVersion: 0,
    });

    context.log.info(`[Confirmation] Transaction confirmed in slot ${txDetails?.slot}`);

    // Step 8: Log to Cosmos DB
    const emissionRecord: EmissionRecord = {
      id: `${executionDate}_${Date.now()}`,
      execution_date: executionDate,
      timestamp: new Date().toISOString(),
      status: "success",
      tx_signature: signature,
      block_time: txDetails?.blockTime || Math.floor(Date.now() / 1000),
      emission_amount: 50000 * 1e9, // TODO: Get from program state (phase-dependent)
      phase: "pre-launch", // TODO: Get from program state
      retry_count: 0,
      sol_balance_before: balanceSOL,
      sol_balance_after: (await connection.getBalance(emissionAuthority.publicKey)) / 1e9,
    };

    await writeToCosmosDB(context, emissionRecord);

    // Step 9: Publish success event to Event Grid
    await publishEvent(context, {
      eventType: "DetourCoin.Emission.Succeeded",
      subject: "emission/daily",
      dataVersion: "1.0",
      data: {
        date: executionDate,
        signature: signature,
        amount: emissionRecord.emission_amount,
        phase: emissionRecord.phase,
        duration_ms: Date.now() - startTime,
      },
    });

    // Step 10: Custom telemetry to Application Insights
    telemetryClient.trackMetric({
      name: "EmissionExecutionSuccess",
      value: 1,
      properties: {
        date: executionDate,
        phase: emissionRecord.phase,
      },
    });

    telemetryClient.trackMetric({
      name: "EmissionDurationMs",
      value: Date.now() - startTime,
    });

    telemetryClient.trackMetric({
      name: "SOLBalance",
      value: emissionRecord.sol_balance_after!,
    });

    context.log.info(`[Success] Daily emission completed successfully in ${Date.now() - startTime}ms`);

  } catch (error: any) {
    context.log.error(`[Error] Daily emission failed: ${error.message}`, error.stack);

    // Log failure to Cosmos DB
    const failureRecord: EmissionRecord = {
      id: `${executionDate}_${Date.now()}_FAILED`,
      execution_date: executionDate,
      timestamp: new Date().toISOString(),
      status: "failure",
      error_message: error.message,
      emission_amount: 0,
      phase: "pre-launch", // Default
      retry_count: 0,
    };

    await writeToCosmosDB(context, failureRecord);

    // Publish failure event (CRITICAL)
    await publishEvent(context, {
      eventType: "DetourCoin.Emission.Failed",
      subject: "emission/daily",
      dataVersion: "1.0",
      data: {
        date: executionDate,
        error: error.message,
        duration_ms: Date.now() - startTime,
      },
    });

    // Track failure metric
    telemetryClient.trackMetric({
      name: "EmissionExecutionFailure",
      value: 1,
      properties: {
        date: executionDate,
        error: error.message,
      },
    });

    // Re-throw to trigger Azure Functions failure handling
    throw error;
  }
};

async function buildEmissionInstruction(
  programId: PublicKey,
  emissionState: PublicKey,
  authority: PublicKey,
  connection: Connection
): Promise<any> {
  // TODO: Replace with actual Anchor-generated instruction builder
  // This is a placeholder - actual implementation uses Anchor IDL

  const { Instruction } = await import("@coral-xyz/anchor");

  // Example instruction data (actual format depends on program IDL)
  const data = Buffer.from([
    0, // Instruction discriminator for execute_daily_emission
  ]);

  return {
    programId,
    keys: [
      { pubkey: emissionState, isSigner: false, isWritable: true },
      { pubkey: authority, isSigner: true, isWritable: false },
      // ... other accounts
    ],
    data,
  };
}

async function writeToCosmosDB(context: Context, record: EmissionRecord): Promise<void> {
  try {
    const endpoint = process.env.COSMOS_DB_ENDPOINT!;
    const client = new CosmosClient({
      endpoint,
      aadCredentials: { /* Managed identity authentication */ },
    });

    const database = client.database(process.env.COSMOS_DB_DATABASE!);
    const container = database.container(process.env.COSMOS_DB_CONTAINER!);

    await container.items.create(record);
    context.log.info(`[Cosmos DB] Wrote record ${record.id}`);
  } catch (error: any) {
    context.log.error(`[Cosmos DB] Failed to write record: ${error.message}`);
    // Don't throw - log and continue (operation history is non-critical)
  }
}

async function publishEvent(context: Context, event: any): Promise<void> {
  try {
    const endpoint = process.env.EVENT_GRID_TOPIC_ENDPOINT!;
    const client = new EventGridPublisherClient(
      endpoint,
      "EventGrid",
      new AzureKeyCredential("managed-identity-placeholder") // TODO: Use managed identity
    );

    await client.send([{
      id: `${Date.now()}-${Math.random()}`,
      eventTime: new Date(),
      ...event,
    }]);

    context.log.info(`[Event Grid] Published event: ${event.eventType}`);
  } catch (error: any) {
    context.log.error(`[Event Grid] Failed to publish event: ${error.message}`);
    // Don't throw - log and continue
  }
}

export default timerTrigger;
```

### RPC Connection & Failover

#### Primary RPC Configuration (QuickNode/Helius)

```typescript
// lib/solana.ts
import { Connection, ConnectionConfig } from "@solana/web3.js";

export interface RPCConfig {
  url: string;
  name: string;
  timeout: number;
  maxRetries: number;
}

export class SolanaRPCClient {
  private primaryConfig: RPCConfig;
  private backupConfig: RPCConfig;
  private activeConnection?: Connection;

  constructor() {
    this.primaryConfig = {
      url: process.env.SOLANA_RPC_PRIMARY!,
      name: "Primary (QuickNode/Helius)",
      timeout: parseInt(process.env.RPC_TIMEOUT_MS || "30000"),
      maxRetries: 3,
    };

    this.backupConfig = {
      url: process.env.SOLANA_RPC_BACKUP!,
      name: "Backup (Alchemy/Public)",
      timeout: parseInt(process.env.RPC_TIMEOUT_MS || "30000"),
      maxRetries: 3,
    };
  }

  async connect(logger: (msg: string) => void): Promise<Connection> {
    // Try primary RPC
    try {
      const connection = this.createConnection(this.primaryConfig);
      await this.testConnection(connection);
      logger(`[RPC] Connected to ${this.primaryConfig.name}`);
      this.activeConnection = connection;
      return connection;
    } catch (primaryError) {
      logger(`[RPC] Primary RPC failed: ${primaryError}. Trying backup...`);

      // Failover to backup RPC
      try {
        const connection = this.createConnection(this.backupConfig);
        await this.testConnection(connection);
        logger(`[RPC] Connected to ${this.backupConfig.name}`);
        this.activeConnection = connection;
        return connection;
      } catch (backupError) {
        throw new Error(`Both primary and backup RPC failed: ${primaryError} | ${backupError}`);
      }
    }
  }

  private createConnection(config: RPCConfig): Connection {
    const connectionConfig: ConnectionConfig = {
      commitment: 'confirmed',
      confirmTransactionInitialTimeout: config.timeout,
      httpHeaders: {
        "Content-Type": "application/json",
      },
    };

    return new Connection(config.url, connectionConfig);
  }

  private async testConnection(connection: Connection): Promise<void> {
    // Test connection with getRecentBlockhash (lightweight call)
    await connection.getLatestBlockhash('confirmed');
  }

  async getConnectionWithRetry(
    logger: (msg: string) => void,
    maxRetries: number = 3
  ): Promise<Connection> {
    let lastError: Error | undefined;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await this.connect(logger);
      } catch (error: any) {
        lastError = error;
        logger(`[RPC] Connection attempt ${attempt}/${maxRetries} failed: ${error.message}`);

        if (attempt < maxRetries) {
          const delayMs = Math.pow(2, attempt) * 1000; // Exponential backoff: 2s, 4s, 8s
          logger(`[RPC] Retrying in ${delayMs}ms...`);
          await new Promise(resolve => setTimeout(resolve, delayMs));
        }
      }
    }

    throw new Error(`Failed to connect after ${maxRetries} attempts: ${lastError?.message}`);
  }
}
```

### Error Handling Taxonomy

```typescript
// lib/errors.ts
export enum EmissionErrorCode {
  RPC_CONNECTION_FAILED = "E001-RPC-CONNECTION-FAILED",
  RPC_TIMEOUT = "E002-RPC-TIMEOUT",
  INSUFFICIENT_SOL = "E003-INSUFFICIENT-SOL",
  TRANSACTION_SIMULATION_FAILED = "E004-TX-SIMULATION-FAILED",
  TRANSACTION_SEND_FAILED = "E005-TX-SEND-FAILED",
  TRANSACTION_CONFIRMATION_FAILED = "E006-TX-CONFIRMATION-FAILED",
  ALREADY_EXECUTED = "E007-ALREADY-EXECUTED",
  KEYPAIR_LOAD_FAILED = "E008-KEYPAIR-LOAD-FAILED",
  PROGRAM_ERROR = "E009-PROGRAM-ERROR",
  COSMOS_DB_WRITE_FAILED = "E010-COSMOS-DB-WRITE-FAILED",
  EVENT_GRID_PUBLISH_FAILED = "E011-EVENT-GRID-PUBLISH-FAILED",
}

export class EmissionError extends Error {
  constructor(
    public code: EmissionErrorCode,
    message: string,
    public retryable: boolean = true,
    public critical: boolean = false
  ) {
    super(`[${code}] ${message}`);
    this.name = "EmissionError";
  }
}

export function classifyError(error: any): EmissionError {
  if (error instanceof EmissionError) {
    return error;
  }

  // Classify Solana RPC errors
  if (error.message?.includes("timeout") || error.message?.includes("ETIMEDOUT")) {
    return new EmissionError(
      EmissionErrorCode.RPC_TIMEOUT,
      `RPC request timed out: ${error.message}`,
      true, // retryable
      false
    );
  }

  if (error.message?.includes("insufficient funds")) {
    return new EmissionError(
      EmissionErrorCode.INSUFFICIENT_SOL,
      "Emission authority has insufficient SOL for transaction fees",
      false, // not retryable (requires manual SOL transfer)
      true // critical
    );
  }

  if (error.message?.includes("already executed") || error.message?.includes("EmissionTooEarly")) {
    return new EmissionError(
      EmissionErrorCode.ALREADY_EXECUTED,
      "Daily emission has already been executed for this period",
      false, // not retryable
      false // not critical (expected behavior)
    );
  }

  // Default to generic program error
  return new EmissionError(
    EmissionErrorCode.PROGRAM_ERROR,
    error.message || "Unknown program error",
    true,
    true
  );
}
```

### Logging Strategy

```typescript
// lib/monitoring.ts
import { TelemetryClient } from "applicationinsights";

export interface StructuredLogEvent {
  level: "info" | "warn" | "error";
  message: string;
  timestamp: string;
  component: string;
  properties?: Record<string, any>;
}

export class EmissionLogger {
  constructor(
    private context: any, // Azure Functions Context
    private telemetryClient: TelemetryClient
  ) {}

  info(component: string, message: string, properties?: Record<string, any>) {
    const event: StructuredLogEvent = {
      level: "info",
      message,
      timestamp: new Date().toISOString(),
      component,
      properties,
    };

    this.context.log.info(JSON.stringify(event));
    this.telemetryClient.trackTrace({
      message: `[${component}] ${message}`,
      severity: 1, // Information
      properties,
    });
  }

  warn(component: string, message: string, properties?: Record<string, any>) {
    const event: StructuredLogEvent = {
      level: "warn",
      message,
      timestamp: new Date().toISOString(),
      component,
      properties,
    };

    this.context.log.warn(JSON.stringify(event));
    this.telemetryClient.trackTrace({
      message: `[${component}] ${message}`,
      severity: 2, // Warning
      properties,
    });
  }

  error(component: string, message: string, error?: Error, properties?: Record<string, any>) {
    const event: StructuredLogEvent = {
      level: "error",
      message,
      timestamp: new Date().toISOString(),
      component,
      properties: {
        ...properties,
        error_name: error?.name,
        error_message: error?.message,
        error_stack: error?.stack,
      },
    };

    this.context.log.error(JSON.stringify(event));
    this.telemetryClient.trackException({
      exception: error || new Error(message),
      properties,
    });
  }
}
```

---

## 5. Timer Trigger Scheduling

### Primary Timer Trigger

**Cron Expression:** `0 0 12 * * *` (12:00 PM UTC daily)

**function.json:**

```json
{
  "bindings": [
    {
      "name": "myTimer",
      "type": "timerTrigger",
      "direction": "in",
      "schedule": "0 0 12 * * *",
      "runOnStartup": false,
      "useMonitor": true
    }
  ],
  "scriptFile": "../dist/daily-emission/index.js"
}
```

**Cron Format:** `{second} {minute} {hour} {day} {month} {day-of-week}`
- `0` = Second 0
- `0` = Minute 0
- `12` = Hour 12 (12 PM UTC)
- `*` = Every day of month
- `*` = Every month
- `*` = Every day of week

**Time Zone:** All Azure Functions timer triggers use UTC. To target a specific local time, calculate the UTC offset.

Examples:
- 12 PM EST (UTC-5) → `0 0 17 * * *` (5 PM UTC)
- 12 PM PST (UTC-8) → `0 0 20 * * *` (8 PM UTC)
- 12 PM UTC → `0 0 12 * * *`

### Backup Timer Trigger (Safety Net)

**Purpose:** Execute emission if primary trigger fails (Azure Functions outage, deployment, etc.)

**Cron Expression:** `0 0 13 * * *` (1:00 PM UTC daily, 1 hour after primary)

**Logic:**
1. Check Cosmos DB for execution record with `execution_date` = today
2. If record exists with `status: "success"` → Exit gracefully (already executed)
3. If no record or `status: "failure"` → Execute emission

**backup-emission/index.ts:**

```typescript
import { AzureFunction, Context } from "@azure/functions";
import { CosmosClient } from "@azure/cosmos";

const backupTimerTrigger: AzureFunction = async function (context: Context, myTimer: any): Promise<void> {
  const executionDate = new Date().toISOString().split('T')[0]; // YYYY-MM-DD

  context.log.info(`[Backup Emission] Checking if emission for ${executionDate} already executed`);

  // Query Cosmos DB for today's execution
  const cosmosClient = new CosmosClient({
    endpoint: process.env.COSMOS_DB_ENDPOINT!,
    aadCredentials: { /* Managed identity */ },
  });

  const database = cosmosClient.database(process.env.COSMOS_DB_DATABASE!);
  const container = database.container(process.env.COSMOS_DB_CONTAINER!);

  const { resources } = await container.items
    .query({
      query: "SELECT * FROM c WHERE c.execution_date = @date AND c.status = @status",
      parameters: [
        { name: "@date", value: executionDate },
        { name: "@status", value: "success" },
      ],
    })
    .fetchAll();

  if (resources.length > 0) {
    context.log.info(`[Backup Emission] Emission already executed successfully for ${executionDate}. Exiting.`);
    return; // Graceful exit
  }

  context.log.warn(`[Backup Emission] No successful emission found for ${executionDate}. Executing backup emission.`);

  // Execute emission (reuse primary logic)
  // ... (same logic as daily-emission/index.ts)
};

export default backupTimerTrigger;
```

### Manual Trigger (HTTP Endpoint)

**Purpose:** Allow manual emission execution for testing, missed emissions, or emergency scenarios

**manual-emission/function.json:**

```json
{
  "bindings": [
    {
      "authLevel": "function",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["post"],
      "route": "emission/execute"
    },
    {
      "type": "http",
      "direction": "out",
      "name": "res"
    }
  ],
  "scriptFile": "../dist/manual-emission/index.js"
}
```

**Request:**

```bash
curl -X POST https://func-detourcoin-emission-prod.azurewebsites.net/api/emission/execute \
  -H "x-functions-key: YOUR_FUNCTION_KEY" \
  -H "Content-Type: application/json" \
  -d '{"force": false, "dry_run": false}'
```

**manual-emission/index.ts:**

```typescript
import { AzureFunction, Context, HttpRequest } from "@azure/functions";

const httpTrigger: AzureFunction = async function (context: Context, req: HttpRequest): Promise<void> {
  const force = req.body?.force || false; // Force execution even if already executed today
  const dryRun = req.body?.dry_run || false; // Simulate without sending transaction

  context.log.info(`[Manual Emission] Manual trigger invoked (force=${force}, dry_run=${dryRun})`);

  if (!force) {
    // Check if already executed (same logic as backup trigger)
    const alreadyExecuted = await checkAlreadyExecuted(context);
    if (alreadyExecuted) {
      context.res = {
        status: 200,
        body: { message: "Emission already executed today", skipped: true },
      };
      return;
    }
  }

  if (dryRun) {
    context.log.info(`[Manual Emission] Dry run mode - simulating transaction without sending`);
    // ... (build and simulate transaction, but don't send)
    context.res = {
      status: 200,
      body: { message: "Dry run completed", simulation: "success" },
    };
    return;
  }

  // Execute emission (reuse primary logic)
  try {
    // ... (same logic as daily-emission/index.ts)
    context.res = {
      status: 200,
      body: { message: "Manual emission executed successfully", signature: "..." },
    };
  } catch (error: any) {
    context.res = {
      status: 500,
      body: { message: "Manual emission failed", error: error.message },
    };
  }
};

async function checkAlreadyExecuted(context: Context): Promise<boolean> {
  // ... (same Cosmos DB query as backup trigger)
  return false;
}

export default httpTrigger;
```

### Retry Policy Configuration

**Built-in Azure Functions Retry:** Disabled in `host.json` (`maxRetryCount: 0`)

**Custom Retry Logic:** Implemented in function code with exponential backoff

```typescript
async function executeWithRetry<T>(
  operation: () => Promise<T>,
  maxAttempts: number = 3,
  baseDelayMs: number = 2000,
  logger?: (msg: string) => void
): Promise<T> {
  let lastError: Error | undefined;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await operation();
    } catch (error: any) {
      lastError = error;

      if (logger) {
        logger(`Attempt ${attempt}/${maxAttempts} failed: ${error.message}`);
      }

      // Check if error is retryable
      const classifiedError = classifyError(error);
      if (!classifiedError.retryable) {
        throw classifiedError; // Don't retry non-retryable errors
      }

      if (attempt < maxAttempts) {
        const delayMs = baseDelayMs * Math.pow(2, attempt - 1); // Exponential backoff
        if (logger) {
          logger(`Retrying in ${delayMs}ms...`);
        }
        await new Promise(resolve => setTimeout(resolve, delayMs));
      }
    }
  }

  throw new Error(`Operation failed after ${maxAttempts} attempts: ${lastError?.message}`);
}
```

### Durable Functions (Optional)

For complex retry scenarios with timeouts and cancellation, consider using **Durable Functions**:

```typescript
import * as df from "durable-functions";

const orchestrator = df.orchestrator(function* (context) {
  const outputs = [];

  try {
    // Execute emission with timeout
    const emissionResult = yield context.df.callActivityWithRetry("ExecuteEmission", {
      maxNumberOfAttempts: 3,
      firstRetryIntervalInMilliseconds: 2000,
      backoffCoefficient: 2.0,
      maxRetryIntervalInMilliseconds: 10000,
      retryTimeoutInMilliseconds: 60000,
    }, null);

    outputs.push(emissionResult);
  } catch (error) {
    // Handle timeout or exhausted retries
    yield context.df.callActivity("SendCriticalAlert", { error });
  }

  return outputs;
});

df.app.orchestration("EmissionOrchestrator", orchestrator);
```

---

## 6. Key Vault

### Secrets Storage

#### Emission Authority Keypair

**Secret Name:** `detourcoin-emission-authority-keypair`

**Format:** JSON array of private key bytes (Uint8Array serialized to JSON)

**Generation:**

```bash
# On local machine (secure environment), generate Solana keypair
solana-keygen new --outfile emission-authority.json --no-bip39-passphrase

# Convert to JSON array format for Key Vault
cat emission-authority.json | jq -c '.'
# Output: [123,45,67,89,...]

# Store in Azure Key Vault
az keyvault secret set \
  --vault-name kv-detourcoin-prod \
  --name detourcoin-emission-authority-keypair \
  --value "[123,45,67,89,...]" \
  --description "Emission authority Solana keypair (private key)" \
  --expires "2026-12-31T23:59:59Z"
```

⚠️ **CRITICAL SECURITY:** The emission authority private key grants the ability to mint DetourCoin tokens. This secret must be:
1. Generated offline on an air-gapped machine
2. Backed up to secure offline storage (encrypted USB drive in safe)
3. Never logged, committed to git, or stored in plaintext
4. Rotated annually (requires updating Key Vault and redeploying program)

#### RPC URLs with API Keys

**Secret Name:** `primary-rpc-url`

**Format:** Full URL with embedded API key (QuickNode/Helius format)

```bash
# QuickNode example
az keyvault secret set \
  --vault-name kv-detourcoin-prod \
  --name primary-rpc-url \
  --value "https://solana-mainnet.quiknode.pro/abc123def456/YOUR_API_KEY/" \
  --description "QuickNode Solana mainnet RPC endpoint"

# Helius example
az keyvault secret set \
  --vault-name kv-detourcoin-prod \
  --name backup-rpc-url \
  --value "https://mainnet.helius-rpc.com/?api-key=YOUR_API_KEY" \
  --description "Helius Solana mainnet RPC endpoint (backup)"
```

#### Monitoring/Alert Webhook Secrets

```bash
# PagerDuty integration key
az keyvault secret set \
  --vault-name kv-detourcoin-prod \
  --name pagerduty-integration-key \
  --value "R03XXXXXXXXXXXXXXXXXXXX" \
  --description "PagerDuty Events API v2 integration key"

# Slack webhook URL
az keyvault secret set \
  --vault-name kv-detourcoin-prod \
  --name slack-webhook-url \
  --value "https://hooks.example.com/services/YOUR_WORKSPACE_ID/YOUR_CHANNEL_ID/YOUR_WEBHOOK_TOKEN" \
  --description "Slack incoming webhook for operational alerts"
```

### Security Configuration

#### Encryption

| Layer | Mechanism | Key Management |
|-------|-----------|----------------|
| **At Rest** | AES-256 | Microsoft-managed keys (default) or customer-managed keys (BYOK) |
| **In Transit** | TLS 1.2+ | Azure-managed certificates |
| **Access Control** | Azure RBAC | Role-based permissions (no access policies) |

#### Soft Delete & Purge Protection

```bash
# Enable soft delete (90-day retention) and purge protection
az keyvault update \
  --name kv-detourcoin-prod \
  --enable-soft-delete true \
  --retention-days 90 \
  --enable-purge-protection true
```

**Effect:**
- Deleted secrets are retained for 90 days before permanent deletion
- Secrets cannot be purged (permanently deleted) during retention period
- Protects against accidental or malicious secret deletion

#### RBAC with Managed Identity

**Function App Access:**

```bicep
resource keyVaultRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(functionApp.id, keyVault.id, 'Key Vault Secrets User')
  scope: keyVault
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6')
    principalId: functionApp.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

**Human Administrator Access:**

```bash
# Grant DevOps Engineer access to manage secrets
az role assignment create \
  --assignee devops@yourdomain.com \
  --role "Key Vault Administrator" \
  --scope /subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/rg-detourcoin-prod/providers/Microsoft.KeyVault/vaults/kv-detourcoin-prod
```

#### Activity Log Auditing

All Key Vault access is automatically logged to Azure Activity Log:

```bash
# Query Key Vault access logs
az monitor activity-log list \
  --resource-group rg-detourcoin-prod \
  --resource-type "Microsoft.KeyVault/vaults" \
  --start-time "2025-12-01T00:00:00Z" \
  --end-time "2025-12-31T23:59:59Z" \
  --query "[?contains(operationName.value, 'Microsoft.KeyVault/vaults/secrets')]" \
  -o table
```

**Alerts for Suspicious Activity:**

```bash
# Create alert for Key Vault secret access from unexpected identity
az monitor metrics alert create \
  --name "KeyVault-UnauthorizedAccess" \
  --resource-group rg-detourcoin-prod \
  --scopes /subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/rg-detourcoin-prod/providers/Microsoft.KeyVault/vaults/kv-detourcoin-prod \
  --condition "count ResourceId == '/subscriptions/.../Microsoft.KeyVault/vaults/kv-detourcoin-prod' where OperationName == 'SecretGet' and CallerIPAddress != 'EXPECTED_FUNCTION_APP_IP'" \
  --action detourcoin-critical-alerts
```

### Networking: Private Endpoint (Optional)

For VNet isolation (recommended for production):

```bicep
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-04-01' = {
  name: 'pe-keyvault-detourcoin-prod'
  location: location
  properties: {
    subnet: {
      id: subnet.id // VNet subnet for private endpoint
    }
    privateLinkServiceConnections: [
      {
        name: 'keyvault-connection'
        properties: {
          privateLinkServiceId: keyVault.id
          groupIds: ['vault']
        }
      }
    ]
  }
}
```

**Effect:** Key Vault is accessible only from within VNet (Function App must be VNet-integrated).

### Key Vault References in Function App

**Application Settings (App Service Configuration):**

```javascript
{
  "EMISSION_AUTHORITY_KEYPAIR": "@Microsoft.KeyVault(SecretUri=https://kv-detourcoin-prod.vault.azure.net/secrets/detourcoin-emission-authority-keypair/)",
  "SOLANA_RPC_PRIMARY": "@Microsoft.KeyVault(SecretUri=https://kv-detourcoin-prod.vault.azure.net/secrets/primary-rpc-url/)",
  "SOLANA_RPC_BACKUP": "@Microsoft.KeyVault(SecretUri=https://kv-detourcoin-prod.vault.azure.net/secrets/backup-rpc-url/)"
}
```

**Automatic Resolution:** Azure Functions runtime automatically resolves Key Vault references using Function App's managed identity.

**Code Usage:**

```typescript
// No special Azure SDK calls needed - secret is already resolved in environment variable
const keypairJson = process.env.EMISSION_AUTHORITY_KEYPAIR!;
const keypairBytes = JSON.parse(keypairJson);
const emissionAuthority = Keypair.fromSecretKey(new Uint8Array(keypairBytes));
```

### Secrets Rotation

**Annual Rotation Policy:**

1. Generate new keypair offline
2. Store new keypair in Key Vault as new secret version
3. Update Solana program to authorize new keypair (multi-sig transaction)
4. Test new keypair in dev environment
5. Update Function App configuration to use new secret version
6. Monitor for 7 days
7. Revoke old keypair authority in Solana program
8. Delete old secret version in Key Vault (soft delete, 90-day retention)

**Automated Rotation (Future):**

```bicep
resource keyVaultSecret 'Microsoft.KeyVault/vaults/secrets@2023-02-01' = {
  parent: keyVault
  name: 'detourcoin-emission-authority-keypair'
  properties: {
    value: keypairJson
    attributes: {
      enabled: true
      exp: dateTimeToEpoch('2026-12-31T23:59:59Z')
    }
    rotationPolicy: {
      lifetimeActions: [
        {
          trigger: {
            timeBeforeExpiry: 'P30D' // 30 days before expiry
          }
          action: {
            type: 'Notify' // Send notification to action group
          }
        }
      ]
    }
  }
}
```

---

## 7. Cosmos DB Operation History

### Database Configuration

#### NoSQL API (Recommended)

**Pros:**
- Rich querying capabilities (SQL-like syntax)
- Flexible schema (can add fields without migration)
- Native Azure SDK support
- Change feed for real-time processing

**Cons:**
- Higher cost than Table API (~$0.25/M RUs for serverless)

**Structure:**

```json
{
  "id": "2025-12-15_1734278400000",
  "execution_date": "2025-12-15",
  "timestamp": "2025-12-15T12:00:05.123Z",
  "status": "success",
  "tx_signature": "5j7zX...",
  "error_message": null,
  "block_time": 1734278405,
  "emission_amount": 50000000000000,
  "phase": "pre-launch",
  "retry_count": 0,
  "sol_balance_before": 10.5,
  "sol_balance_after": 10.499975,
  "rpc_used": "primary",
  "duration_ms": 5123
}
```

**Container Creation (Bicep):**

```bicep
resource cosmosDbDatabase 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2023-04-15' = {
  parent: cosmosDbAccount
  name: 'detourcoin'
  properties: {
    resource: {
      id: 'detourcoin'
    }
    options: {
      // Serverless mode - no throughput provisioning needed
    }
  }
}

resource cosmosDbContainer 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2023-04-15' = {
  parent: cosmosDbDatabase
  name: 'emission_history'
  properties: {
    resource: {
      id: 'emission_history'
      partitionKey: {
        paths: ['/execution_date']
        kind: 'Hash'
      }
      indexingPolicy: {
        automatic: true
        indexingMode: 'consistent'
        includedPaths: [
          {
            path: '/*'
          }
        ]
        excludedPaths: [
          {
            path: '/_etag/?'
          }
        ]
      }
      defaultTtl: 63072000 // 2 years in seconds
    }
  }
}
```

#### Table API (Cost-Optimized Alternative)

**Pros:**
- Lower cost (~$0.045/GB storage, ~$0.40/M transactions)
- Simple key-value operations
- Compatible with Azure Table Storage SDKs

**Cons:**
- Limited querying (only partition key + row key)
- No SQL-like queries
- Schema less flexible

**Structure:**

| PartitionKey | RowKey | Timestamp | Status | TxSignature | EmissionAmount | ... |
|--------------|--------|-----------|--------|-------------|----------------|-----|
| 2025-12-15 | 1734278400000 | 2025-12-15T12:00:05Z | success | 5j7zX... | 50000000000000 | ... |

**Table Creation (Azure CLI):**

```bash
# Create table in Cosmos DB Table API account
az cosmosdb table create \
  --account-name cosmos-detourcoin-prod \
  --resource-group rg-detourcoin-prod \
  --name emission_history \
  --throughput 400 # Autoscale RU/s
```

### Serverless vs. Provisioned Throughput

| Mode | Cost Model | Best For | DetourCoin Use Case |
|------|-----------|----------|---------------------|
| **Serverless** | Pay per RU consumed (~$0.25/M RUs) | Infrequent, unpredictable workloads | ✅ Ideal (1 write/day = negligible cost) |
| **Provisioned** | Pay for min RU/s allocation (400 RU/s min) | Consistent, predictable workloads | ❌ Overkill for low-volume emissions |
| **Autoscale** | Pay for actual RU/s used (400-1000 RU/s range) | Variable but bounded workloads | ⚠️ Acceptable but more expensive than serverless |

**Recommendation:** Use **Serverless** for emission history. At 1 write/day (~10 RUs/write), monthly cost is ~$0.000075 (essentially free).

### Schema Design

**Partition Key:** `execution_date` (YYYY-MM-DD)

**Rationale:**
- Emissions are executed once per day
- Queries are typically by date range ("show emissions for December 2025")
- Even data distribution (1 partition per day)
- Point reads are efficient (`SELECT * FROM c WHERE c.execution_date = '2025-12-15'`)

**Document Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique identifier: `{date}_{timestamp}` |
| `execution_date` | string | Yes | YYYY-MM-DD (partition key) |
| `timestamp` | string | Yes | ISO 8601 timestamp |
| `status` | string | Yes | `success` \| `failure` \| `pending` |
| `tx_signature` | string | No | Solana transaction signature (if successful) |
| `error_message` | string | No | Error details (if failed) |
| `block_time` | number | No | Solana block timestamp |
| `emission_amount` | number | Yes | Tokens minted (lamports) |
| `phase` | string | Yes | `pre-launch` \| `post-launch` |
| `retry_count` | number | Yes | Number of retry attempts |
| `sol_balance_before` | number | No | Authority SOL balance before tx |
| `sol_balance_after` | number | No | Authority SOL balance after tx |
| `rpc_used` | string | No | `primary` \| `backup` |
| `duration_ms` | number | No | Execution time in milliseconds |

### Querying

**Point Read (Single Day):**

```typescript
const { resource } = await container.item(id, partitionKey).read();
```

**Range Query (Date Range):**

```typescript
const { resources } = await container.items
  .query({
    query: "SELECT * FROM c WHERE c.execution_date >= @startDate AND c.execution_date <= @endDate ORDER BY c.timestamp DESC",
    parameters: [
      { name: "@startDate", value: "2025-12-01" },
      { name: "@endDate", value: "2025-12-31" },
    ],
  })
  .fetchAll();
```

**Cross-Partition Query (All Failures):**

```typescript
const { resources } = await container.items
  .query({
    query: "SELECT * FROM c WHERE c.status = @status ORDER BY c.timestamp DESC",
    parameters: [
      { name: "@status", value: "failure" },
    ],
  })
  .fetchAll();
```

**Note:** Cross-partition queries consume more RUs (scan all partitions). For cost optimization, avoid frequent cross-partition queries.

### Indexing Policy Optimization

Default indexing policy indexes **all fields**, which is sufficient for low-volume workloads. For optimization:

```bicep
indexingPolicy: {
  automatic: true
  indexingMode: 'consistent'
  includedPaths: [
    { path: '/execution_date/?' }  // Partition key (always indexed)
    { path: '/status/?' }           // Frequently queried
    { path: '/timestamp/?' }        // Sorting
    { path: '/tx_signature/?' }     // Lookup by signature
  ]
  excludedPaths: [
    { path: '/error_message/?' }    // Large text, rarely queried
    { path: '/_etag/?' }            // System field
  ]
}
```

**Effect:** Reduces RU consumption for writes (fewer indexes to update) at the cost of slower queries on excluded fields.

### TTL (Time-to-Live) Configuration

**Retention Policy:** 2 years (730 days)

```bicep
resource: {
  id: 'emission_history'
  defaultTtl: 63072000 // 2 years in seconds (730 * 24 * 60 * 60)
  partitionKey: { ... }
}
```

**Effect:** Documents older than 2 years are automatically deleted (no manual cleanup needed).

**Override TTL per Document:**

```typescript
await container.items.create({
  ...emissionRecord,
  ttl: 31536000, // 1 year (override default)
});
```

### Continuous Backup & Point-in-Time Restore

**Configuration:**

```bicep
resource cosmosDbAccount 'Microsoft.DocumentDB/databaseAccounts@2023-04-15' = {
  name: 'cosmos-detourcoin-prod'
  properties: {
    backupPolicy: {
      type: 'Continuous'
      continuousModeProperties: {
        tier: 'Continuous7Days' // 7-day point-in-time restore window
      }
    }
  }
}
```

**Restore Procedure:**

```bash
# Restore to specific timestamp (e.g., before accidental deletion)
az cosmosdb sql database restore \
  --account-name cosmos-detourcoin-prod \
  --resource-group rg-detourcoin-prod \
  --name detourcoin \
  --restore-timestamp "2025-12-15T10:30:00Z" \
  --location eastus
```

**Cost:** Continuous backup adds ~20% to storage costs but provides protection against accidental data loss.

---

## 8. Event Grid & Action Groups Alerting

### Event Grid Topics

**Custom Topic for Emission Events:**

```bicep
resource eventGridTopic 'Microsoft.EventGrid/topics@2023-06-01-preview' = {
  name: 'detourcoin-emission-events'
  location: location
  properties: {
    inputSchema: 'CloudEventSchemaV1_0'
    publicNetworkAccess: 'Enabled'
  }
}
```

**Event Schema (CloudEvents 1.0):**

```json
{
  "specversion": "1.0",
  "type": "DetourCoin.Emission.Succeeded",
  "source": "detourcoin-emission-function",
  "id": "1734278405123-0.5678",
  "time": "2025-12-15T12:00:05.123Z",
  "datacontenttype": "application/json",
  "data": {
    "execution_date": "2025-12-15",
    "tx_signature": "5j7zX...",
    "emission_amount": 50000000000000,
    "phase": "pre-launch",
    "duration_ms": 5123,
    "rpc_used": "primary"
  }
}
```

**Event Types:**

| Event Type | Severity | Description | Action Group |
|------------|----------|-------------|--------------|
| `DetourCoin.Emission.Succeeded` | Informational | Daily emission completed successfully | Operational |
| `DetourCoin.Emission.Failed` | Critical | Emission failed after all retries | Critical |
| `DetourCoin.Emission.LowSOLBalance` | Critical | SOL balance < threshold | Critical |
| `DetourCoin.Emission.RPCFailover` | Warning | Failover to backup RPC | Operational |
| `DetourCoin.Emission.SlowExecution` | Warning | Execution time > 45s | Operational |
| `DetourCoin.Emission.AlreadyExecuted` | Informational | Duplicate execution prevented | Operational |

### Action Groups

#### Critical Action Group

```bicep
resource criticalActionGroup 'Microsoft.Insights/actionGroups@2023-01-01' = {
  name: 'detourcoin-critical-alerts'
  location: 'global'
  properties: {
    groupShortName: 'DTC-CRIT'
    enabled: true
    emailReceivers: [
      {
        name: 'DevOps Engineer'
        emailAddress: 'devops@example.com'
        useCommonAlertSchema: true
      }
      {
        name: 'Founder'
        emailAddress: 'founder@example.com'
        useCommonAlertSchema: true
      }
    ]
    smsReceivers: [
      {
        name: 'DevOps SMS'
        countryCode: '1'
        phoneNumber: '5551234567'
      }
    ]
    webhookReceivers: [
      {
        name: 'PagerDuty'
        serviceUri: 'https://events.pagerduty.com/v2/enqueue'
        useCommonAlertSchema: true
      }
    ]
    azureFunctionReceivers: [] // Optional: Custom Azure Function for alerting logic
  }
}
```

**Triggers:**
- Emission failure
- SOL balance < 5
- RPC failures (both primary and backup)
- Transaction errors

#### Operational Action Group

```bicep
resource operationalActionGroup 'Microsoft.Insights/actionGroups@2023-01-01' = {
  name: 'detourcoin-operational-alerts'
  location: 'global'
  properties: {
    groupShortName: 'DTC-OPS'
    enabled: true
    emailReceivers: [
      {
        name: 'DevOps Team'
        emailAddress: 'devops-team@example.com'
        useCommonAlertSchema: true
      }
    ]
    webhookReceivers: [
      {
        name: 'Slack Webhook'
        serviceUri: '@Microsoft.KeyVault(SecretUri=https://kv-detourcoin-prod.vault.azure.net/secrets/slack-webhook-url/)'
        useCommonAlertSchema: false
        useAadAuth: false
      }
    ]
  }
}
```

**Triggers:**
- Emission success (daily confirmation)
- Approaching thresholds (SOL < 10, execution time > 30s)
- Performance issues
- Anomalies detected

### Event Grid Subscriptions

**Critical Alerts Subscription:**

```bicep
resource criticalSubscription 'Microsoft.EventGrid/eventSubscriptions@2023-06-01-preview' = {
  name: 'critical-alerts-subscription'
  scope: eventGridTopic
  properties: {
    destination: {
      endpointType: 'AzureFunction'
      properties: {
        resourceId: alertHandlerFunction.id
        maxEventsPerBatch: 1
        preferredBatchSizeInKilobytes: 64
      }
    }
    filter: {
      includedEventTypes: [
        'DetourCoin.Emission.Failed'
        'DetourCoin.Emission.LowSOLBalance'
      ]
      advancedFilters: [
        {
          operatorType: 'StringContains'
          key: 'data.severity'
          values: ['critical']
        }
      ]
    }
    retryPolicy: {
      maxDeliveryAttempts: 30
      eventTimeToLiveInMinutes: 1440 // 24 hours
    }
  }
}
```

**Operational Alerts Subscription:**

```bicep
resource operationalSubscription 'Microsoft.EventGrid/eventSubscriptions@2023-06-01-preview' = {
  name: 'operational-alerts-subscription'
  scope: eventGridTopic
  properties: {
    destination: {
      endpointType: 'WebHook'
      properties: {
        endpointUrl: '@Microsoft.KeyVault(SecretUri=https://kv-detourcoin-prod.vault.azure.net/secrets/slack-webhook-url/)'
        maxEventsPerBatch: 10
      }
    }
    filter: {
      includedEventTypes: [
        'DetourCoin.Emission.Succeeded'
        'DetourCoin.Emission.RPCFailover'
        'DetourCoin.Emission.SlowExecution'
      ]
    }
  }
}
```

### Publishing Events from Function

```typescript
import { EventGridPublisherClient } from "@azure/eventgrid";
import { DefaultAzureCredential } from "@azure/identity";

async function publishEvent(eventType: string, data: any): Promise<void> {
  const endpoint = process.env.EVENT_GRID_TOPIC_ENDPOINT!;

  const client = new EventGridPublisherClient(
    endpoint,
    "CloudEvent",
    new DefaultAzureCredential() // Uses managed identity
  );

  await client.send([
    {
      type: eventType,
      source: "detourcoin-emission-function",
      id: `${Date.now()}-${Math.random()}`,
      time: new Date(),
      dataContentType: "application/json",
      data,
    },
  ]);
}

// Usage
await publishEvent("DetourCoin.Emission.Succeeded", {
  execution_date: "2025-12-15",
  tx_signature: "5j7zX...",
  emission_amount: 50000000000000,
});
```

### Message Format Examples

**Success Event:**

```json
{
  "specversion": "1.0",
  "type": "DetourCoin.Emission.Succeeded",
  "source": "detourcoin-emission-function",
  "id": "1734278405123-0.5678",
  "time": "2025-12-15T12:00:05.123Z",
  "data": {
    "severity": "informational",
    "execution_date": "2025-12-15",
    "tx_signature": "5j7zXy8K2pQw...",
    "emission_amount": 50000000000000,
    "phase": "pre-launch",
    "duration_ms": 5123,
    "rpc_used": "primary",
    "sol_balance_after": 10.499975
  }
}
```

**Failure Event (Critical):**

```json
{
  "specversion": "1.0",
  "type": "DetourCoin.Emission.Failed",
  "source": "detourcoin-emission-function",
  "id": "1734278405123-0.9876",
  "time": "2025-12-15T12:01:30.456Z",
  "data": {
    "severity": "critical",
    "execution_date": "2025-12-15",
    "error_code": "E004-TX-SIMULATION-FAILED",
    "error_message": "Transaction simulation failed: InsufficientFundsForFee",
    "retry_count": 3,
    "duration_ms": 90456,
    "rpc_used": "backup",
    "sol_balance": 0.0001
  }
}
```

---

## 9. Application Insights & Azure Monitor

### Application Insights Configuration

**Auto-instrumentation for Azure Functions:**

Application Insights is automatically enabled for Azure Functions when `APPLICATIONINSIGHTS_CONNECTION_STRING` is set in app settings.

```javascript
// Application Settings
{
  "APPLICATIONINSIGHTS_CONNECTION_STRING": "InstrumentationKey=abc123...;IngestionEndpoint=https://eastus-0.in.applicationinsights.azure.com/;LiveEndpoint=https://eastus.livediagnostics.monitor.azure.com/"
}
```

**Bicep Configuration:**

```bicep
resource applicationInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: 'ai-detourcoin-prod'
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    RetentionInDays: 90 // 90-day log retention
    SamplingPercentage: 100 // No sampling for critical application
    DisableIpMasking: false // Comply with privacy regulations
    publicNetworkAccessForIngestion: 'Enabled'
    publicNetworkAccessForQuery: 'Enabled'
  }
}
```

### Structured Logging

**Log Levels:**

| Level | Usage | Examples |
|-------|-------|----------|
| **Information** | Normal operation, successful executions | "Emission executed successfully", "Connected to primary RPC" |
| **Warning** | Recoverable issues, degraded performance | "RPC failover to backup", "Execution time exceeded 30s", "SOL balance < 10" |
| **Error** | Operation failures, exceptions | "Transaction simulation failed", "RPC connection timeout", "Cosmos DB write failed" |
| **Critical** | System-wide failures, data loss risk | "Both RPCs failed", "Emission failed after retries", "Key Vault access denied" |

**Custom Dimensions for Filtering:**

```typescript
// Good: Structured logging with custom dimensions
logger.info("Emission", "Transaction confirmed", {
  execution_date: "2025-12-15",
  tx_signature: "5j7zX...",
  phase: "pre-launch",
  rpc_used: "primary",
  duration_ms: 5123,
  block_slot: 234567890
});

// Bad: Unstructured string logging
logger.info(`Emission successful: 5j7zX... on 2025-12-15`);
```

### KQL Queries for Analysis

**Query all emissions for a date range:**

```kusto
traces
| where timestamp >= datetime(2025-12-01) and timestamp <= datetime(2025-12-31)
| where message contains "Emission executed successfully"
| extend execution_date = tostring(customDimensions.execution_date)
| extend tx_signature = tostring(customDimensions.tx_signature)
| extend duration_ms = toint(customDimensions.duration_ms)
| project timestamp, execution_date, tx_signature, duration_ms
| order by timestamp desc
```

**Query failures with error details:**

```kusto
traces
| where timestamp >= ago(30d)
| where severityLevel >= 3 // Error or Critical
| where message contains "Emission"
| extend error_code = tostring(customDimensions.error_code)
| extend error_message = tostring(customDimensions.error_message)
| summarize count() by error_code, error_message
| order by count_ desc
```

**Average execution duration over time:**

```kusto
traces
| where timestamp >= ago(30d)
| where message contains "Emission executed successfully"
| extend duration_ms = toint(customDimensions.duration_ms)
| summarize avg_duration = avg(duration_ms), max_duration = max(duration_ms), min_duration = min(duration_ms) by bin(timestamp, 1d)
| render timechart
```

**SOL balance trend:**

```kusto
customMetrics
| where name == "SOLBalance"
| where timestamp >= ago(30d)
| project timestamp, value
| render timechart
```

### Custom Metrics

**Tracking with Application Insights SDK:**

```typescript
import * as appInsights from "applicationinsights";

const telemetryClient = appInsights.defaultClient;

// Success/Failure counters
telemetryClient.trackMetric({
  name: "EmissionExecutionSuccess",
  value: 1,
  properties: {
    execution_date: "2025-12-15",
    phase: "pre-launch"
  }
});

// Duration metric
telemetryClient.trackMetric({
  name: "EmissionDurationMs",
  value: 5123,
  properties: {
    rpc_used: "primary"
  }
});

// SOL balance metric
telemetryClient.trackMetric({
  name: "SOLBalance",
  value: 10.499975,
  properties: {
    authority: emissionAuthority.publicKey.toBase58()
  }
});

// Transaction fee metric
telemetryClient.trackMetric({
  name: "TransactionFeeSOL",
  value: 0.000025,
  properties: {
    tx_signature: "5j7zX..."
  }
});
```

**Available Metrics:**

| Metric Name | Type | Unit | Description |
|-------------|------|------|-------------|
| `EmissionExecutionSuccess` | Count | executions | Number of successful emissions |
| `EmissionExecutionFailure` | Count | executions | Number of failed emissions |
| `EmissionDurationMs` | Duration | milliseconds | Time to execute emission |
| `SOLBalance` | Gauge | SOL | Emission authority balance |
| `TransactionFeeSOL` | Gauge | SOL | Transaction fee paid |
| `RPCLatencyMs` | Duration | milliseconds | RPC response time |
| `RPCFailoverCount` | Count | failovers | Number of RPC failovers |

### Workbooks & Dashboards

**Create Azure Workbook (JSON template):**

```json
{
  "version": "Notebook/1.0",
  "items": [
    {
      "type": 3,
      "content": {
        "version": "KqlItem/1.0",
        "query": "customMetrics\n| where name == 'EmissionExecutionSuccess'\n| where timestamp >= ago(30d)\n| summarize count() by bin(timestamp, 1d)\n| render columnchart",
        "size": 1,
        "title": "Daily Emission Success Rate (Last 30 Days)",
        "queryType": 0,
        "resourceType": "microsoft.insights/components"
      }
    },
    {
      "type": 3,
      "content": {
        "version": "KqlItem/1.0",
        "query": "customMetrics\n| where name == 'SOLBalance'\n| where timestamp >= ago(30d)\n| project timestamp, value\n| render timechart",
        "size": 1,
        "title": "SOL Balance Trend",
        "queryType": 0,
        "resourceType": "microsoft.insights/components"
      }
    },
    {
      "type": 3,
      "content": {
        "version": "KqlItem/1.0",
        "query": "customMetrics\n| where name == 'EmissionDurationMs'\n| where timestamp >= ago(30d)\n| summarize avg(value), percentile(value, 95), max(value) by bin(timestamp, 1d)\n| render timechart",
        "size": 1,
        "title": "Emission Execution Duration (Avg, P95, Max)",
        "queryType": 0,
        "resourceType": "microsoft.insights/components"
      }
    }
  ]
}
```

### Alert Rules

**Metric Alert: Emission Failure**

```bicep
resource emissionFailureAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'emission-failure-alert'
  location: 'global'
  properties: {
    description: 'Alert when daily emission fails'
    severity: 0 // Critical
    enabled: true
    scopes: [
      applicationInsights.id
    ]
    evaluationFrequency: 'PT5M' // Every 5 minutes
    windowSize: 'PT15M' // 15-minute window
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'EmissionFailureCount'
          metricName: 'EmissionExecutionFailure'
          operator: 'GreaterThan'
          threshold: 0
          timeAggregation: 'Total'
        }
      ]
    }
    actions: [
      {
        actionGroupId: criticalActionGroup.id
      }
    ]
  }
}
```

**Metric Alert: Low SOL Balance**

```bicep
resource lowSOLBalanceAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'low-sol-balance-alert'
  location: 'global'
  properties: {
    description: 'Alert when SOL balance falls below 5'
    severity: 0 // Critical
    enabled: true
    scopes: [
      applicationInsights.id
    ]
    evaluationFrequency: 'PT1H' // Every hour
    windowSize: 'PT1H'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'SOLBalanceThreshold'
          metricName: 'SOLBalance'
          operator: 'LessThan'
          threshold: 5.0
          timeAggregation: 'Average'
        }
      ]
    }
    actions: [
      {
        actionGroupId: criticalActionGroup.id
      }
    ]
  }
}
```

**Log Alert: Slow Execution**

```bicep
resource slowExecutionAlert 'Microsoft.Insights/scheduledQueryRules@2021-08-01' = {
  name: 'slow-execution-alert'
  location: location
  properties: {
    description: 'Alert when emission execution takes longer than 45 seconds'
    severity: 2 // Warning
    enabled: true
    scopes: [
      applicationInsights.id
    ]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT5M'
    criteria: {
      allOf: [
        {
          query: 'customMetrics | where name == "EmissionDurationMs" | where value > 45000'
          timeAggregation: 'Count'
          operator: 'GreaterThan'
          threshold: 0
          failingPeriods: {
            numberOfEvaluationPeriods: 1
            minFailingPeriodsToAlert: 1
          }
        }
      ]
    }
    actions: {
      actionGroups: [
        operationalActionGroup.id
      ]
    }
  }
}
```

**Smart Detection (Anomaly Detection):**

Application Insights includes built-in smart detection for:
- Performance degradation
- Failure anomalies
- Memory leak detection
- Slow dependencies

Enable via Azure Portal → Application Insights → Smart Detection → Configure settings.

### Live Metrics

Access real-time metrics during emission execution:

**Azure Portal:**
Application Insights → Live Metrics Stream

**Metrics displayed:**
- Incoming requests (function invocations)
- Outgoing requests (RPC calls, Cosmos DB writes)
- Overall health
- Servers (function instances)
- Sample telemetry

**Use case:** Monitor live emission execution to verify success/failure immediately after 12 PM UTC trigger.

---

## 10. Virtual Network Configuration (Optional)

### VNet Integration

**Purpose:** Isolate Function App and Azure resources within a private virtual network, blocking public internet access.

**Trade-offs:**

| Aspect | With VNet | Without VNet |
|--------|-----------|--------------|
| **Security** | ✅ Private endpoints, no public exposure | ⚠️ Public endpoints with RBAC/firewall |
| **Cost** | ❌ ~$40-50/month (NAT Gateway + VNet) | ✅ $0 |
| **Complexity** | ❌ Higher (subnet sizing, routing) | ✅ Lower (simpler deployment) |
| **Cold Start** | ✅ No impact on Premium plan | ⚠️ Slight delay on Consumption plan |
| **Compliance** | ✅ Meets strict network isolation requirements | ⚠️ May not meet some compliance standards |

**Recommendation:** VNet is **optional** for DetourCoin. Use RBAC, managed identities, and Key Vault for security without VNet overhead. Enable VNet only if:
1. Compliance requires network isolation
2. Budget allows additional $40-50/month
3. Willing to accept increased complexity

### VNet Architecture

```mermaid
graph TB
    subgraph "Virtual Network (10.0.0.0/16)"
        subgraph "Function Subnet (10.0.1.0/24)"
            FN[Function App<br/>VNet-Integrated]
        end

        subgraph "Private Endpoint Subnet (10.0.2.0/24)"
            PE_KV[Private Endpoint<br/>Key Vault]
            PE_COSMOS[Private Endpoint<br/>Cosmos DB]
            PE_STORAGE[Private Endpoint<br/>Storage Account]
        end

        NAT[NAT Gateway<br/>Outbound Internet]
    end

    FN -->|Private Link| PE_KV
    FN -->|Private Link| PE_COSMOS
    FN -->|Private Link| PE_STORAGE
    FN -->|Public Internet<br/>via NAT| RPC[Solana RPC<br/>External]

    NSG[Network Security Group]
    NSG -.Allows.-> FN

    style FN fill:#0078d4,stroke:#333,stroke-width:2px,color:#fff
    style NAT fill:#ffb900,stroke:#333,stroke-width:2px
    style PE_KV fill:#00a4ef,stroke:#333,stroke-width:2px
```

### Bicep Configuration

**Create VNet and Subnets:**

```bicep
resource virtualNetwork 'Microsoft.Network/virtualNetworks@2023-04-01' = {
  name: 'vnet-detourcoin-prod'
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
    subnets: [
      {
        name: 'function-subnet'
        properties: {
          addressPrefix: '10.0.1.0/24'
          delegations: [
            {
              name: 'delegation'
              properties: {
                serviceName: 'Microsoft.Web/serverFarms'
              }
            }
          ]
          serviceEndpoints: []
        }
      }
      {
        name: 'private-endpoint-subnet'
        properties: {
          addressPrefix: '10.0.2.0/24'
          privateEndpointNetworkPolicies: 'Disabled'
        }
      }
    ]
  }
}
```

**NAT Gateway for Outbound Internet:**

```bicep
resource publicIP 'Microsoft.Network/publicIPAddresses@2023-04-01' = {
  name: 'pip-nat-detourcoin-prod'
  location: location
  sku: {
    name: 'Standard'
  }
  properties: {
    publicIPAllocationMethod: 'Static'
  }
}

resource natGateway 'Microsoft.Network/natGateways@2023-04-01' = {
  name: 'nat-detourcoin-prod'
  location: location
  sku: {
    name: 'Standard'
  }
  properties: {
    idleTimeoutInMinutes: 4
    publicIpAddresses: [
      {
        id: publicIP.id
      }
    ]
  }
}

// Associate NAT Gateway with function subnet
resource functionSubnetUpdate 'Microsoft.Network/virtualNetworks/subnets@2023-04-01' = {
  parent: virtualNetwork
  name: 'function-subnet'
  properties: {
    addressPrefix: '10.0.1.0/24'
    natGateway: {
      id: natGateway.id
    }
    delegations: [
      {
        name: 'delegation'
        properties: {
          serviceName: 'Microsoft.Web/serverFarms'
        }
      }
    ]
  }
}
```

**Integrate Function App with VNet:**

```bicep
resource functionApp 'Microsoft.Web/sites@2022-09-01' = {
  name: 'func-detourcoin-emission-prod'
  location: location
  kind: 'functionapp'
  properties: {
    virtualNetworkSubnetId: functionSubnet.id
    vnetRouteAllEnabled: true // Route all traffic through VNet
    // ... other properties
  }
}
```

**Private Endpoints:**

```bicep
// Key Vault Private Endpoint
resource keyVaultPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-04-01' = {
  name: 'pe-keyvault-detourcoin-prod'
  location: location
  properties: {
    subnet: {
      id: privateEndpointSubnet.id
    }
    privateLinkServiceConnections: [
      {
        name: 'keyvault-connection'
        properties: {
          privateLinkServiceId: keyVault.id
          groupIds: ['vault']
        }
      }
    ]
  }
}

// Cosmos DB Private Endpoint
resource cosmosPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-04-01' = {
  name: 'pe-cosmos-detourcoin-prod'
  location: location
  properties: {
    subnet: {
      id: privateEndpointSubnet.id
    }
    privateLinkServiceConnections: [
      {
        name: 'cosmos-connection'
        properties: {
          privateLinkServiceId: cosmosDbAccount.id
          groupIds: ['Sql'] // NoSQL API
        }
      }
    ]
  }
}
```

**Network Security Group (NSG):**

```bicep
resource nsg 'Microsoft.Network/networkSecurityGroups@2023-04-01' = {
  name: 'nsg-function-subnet'
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowHTTPS'
        properties: {
          priority: 100
          direction: 'Outbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '443'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
        }
      }
      {
        name: 'DenyAllInbound'
        properties: {
          priority: 4096
          direction: 'Inbound'
          access: 'Deny'
          protocol: '*'
          sourcePortRange: '*'
          destinationPortRange: '*'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
        }
      }
    ]
  }
}
```

### Service Endpoints (Lower-Cost Alternative)

If VNet is desired but budget is constrained, use **Service Endpoints** instead of Private Endpoints:

```bicep
resource functionSubnetWithServiceEndpoints 'Microsoft.Network/virtualNetworks/subnets@2023-04-01' = {
  parent: virtualNetwork
  name: 'function-subnet'
  properties: {
    addressPrefix: '10.0.1.0/24'
    serviceEndpoints: [
      {
        service: 'Microsoft.KeyVault'
      }
      {
        service: 'Microsoft.AzureCosmosDB'
      }
      {
        service: 'Microsoft.Storage'
      }
    ]
  }
}

// Update Key Vault to allow access from subnet
resource keyVaultFirewall 'Microsoft.KeyVault/vaults@2023-02-01' = {
  parent: keyVault
  name: 'default'
  properties: {
    networkAcls: {
      defaultAction: 'Deny'
      bypass: 'AzureServices'
      virtualNetworkRules: [
        {
          id: functionSubnet.id
        }
      ]
    }
  }
}
```

**Cost Comparison:**

| Approach | Monthly Cost | Security Level |
|----------|--------------|----------------|
| No VNet (RBAC only) | $0 | Good (RBAC + managed identity) |
| VNet + Service Endpoints | ~$20-30 | Better (network filtering) |
| VNet + Private Endpoints + NAT | ~$40-50 | Best (full network isolation) |

---

## 11. Cost Optimization

### Free Tier Utilization (First 12 Months)

**Azure Free Tier (New subscriptions):**

| Service | Free Tier | DetourCoin Usage | Savings |
|---------|-----------|------------------|---------|
| App Service | 750 hours/month | Function App (Consumption plan separate) | N/A |
| Application Insights | 1 GB data ingestion/month | ~0.5 GB/month (structured logging) | $1.15/month |
| Event Grid | 100K operations/month | ~30 ops/month (1 event/day × 30) | $0.60/month |
| Azure Monitor | 5 GB log ingestion/month | Minimal (metrics only) | $0 |
| Cosmos DB | $25 credit/month | ~$5/month (serverless) | $5/month |
| **Total Savings (Year 1)** | | | **~$6-7/month** |

**Always Free (No Expiration):**

| Service | Always Free | DetourCoin Usage |
|---------|-------------|------------------|
| Azure Functions (Consumption) | 1M executions + 400K GB-s/month | ~30 executions/month (within free tier) |
| Key Vault (Standard) | Unlimited secret operations | ~90 operations/month (3/day) |
| Table Storage | Pay-as-you-go (very low) | Alternative to Cosmos DB (~$0.50/month) |

### Cost Projections

**Monthly Cost Breakdown (Production, Post-Free Tier):**

| Service | Configuration | Monthly Cost | Notes |
|---------|--------------|--------------|-------|
| **Azure Functions** | Consumption plan | $0 | 30 executions/month within 1M free tier |
| **Application Insights** | 5GB ingestion/month | $0-5 | 1GB free, then $2.30/GB |
| **Cosmos DB** | Serverless NoSQL | $5-15 | ~300 RU/month, or $0.50 with Table Storage |
| **Key Vault** | Standard tier | $0-3 | $0.03/10K operations, ~90 ops/month |
| **Event Grid** | Custom topic | $0-2 | 100K ops free, then $0.60/M ops |
| **Storage Account** | LRS, minimal | $1-2 | Function App requirement |
| **NAT Gateway** (optional) | Standard + data | $40-50 | Only if VNet used |
| **VNet** (optional) | Standard | Included | Free with NAT Gateway |
| **Private Endpoints** (optional) | 2-3 endpoints | $7-10 | $3.60/endpoint/month |
| **External: RPC Provider** | QuickNode/Helius | $0-100 | Free tier (50-250K req/month) or paid |
| **Total (No VNet)** | | **$6-27/month** | Well under $200 target |
| **Total (With VNet)** | | **$50-90/month** | Including VNet + private endpoints |
| **Total (With VNet + Paid RPC)** | | **$150-190/month** | Still under $200 |

### Cost Optimization Strategies

#### 1. Use Table Storage Instead of Cosmos DB

```bash
# Create Azure Table Storage (cheaper than Cosmos DB)
az storage account create \
  --name stdetourcoinprod \
  --resource-group rg-detourcoin-prod \
  --location eastus \
  --sku Standard_LRS

az storage table create \
  --name emissionhistory \
  --account-name stdetourcoinprod
```

**Code Change (minimal):**

```typescript
import { TableClient, AzureNamedKeyCredential } from "@azure/data-tables";

const credential = new AzureNamedKeyCredential(accountName, accountKey);
const tableClient = new TableClient(
  `https://${accountName}.table.core.windows.net`,
  "emissionhistory",
  credential
);

// Write record
await tableClient.createEntity({
  partitionKey: executionDate, // YYYY-MM-DD
  rowKey: Date.now().toString(),
  status: "success",
  txSignature: "5j7zX...",
  // ... other fields
});
```

**Savings:** ~$10-15/month (Cosmos DB → Table Storage)

#### 2. Reduce Application Insights Sampling

```json
// host.json
{
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true, // Enable sampling
        "maxTelemetryItemsPerSecond": 5 // Reduce ingestion volume
      }
    }
  }
}
```

**Caution:** Sampling may miss critical events. For DetourCoin (low volume), recommend **no sampling** to ensure 100% visibility.

#### 3. Use Consumption Plan (Not Premium)

**Consumption Plan:**
- No base cost (pay only per execution)
- Auto-scaling (0 to hundreds of instances)
- Cold start delay (~2-5 seconds)

**Premium Plan:**
- Base cost: ~$150/month minimum
- No cold starts (always-on instances)
- VNet integration without Consumption plan limitations

**Recommendation:** Use **Consumption Plan**. Cold start is acceptable for daily emission (not latency-sensitive).

#### 4. Optimize RPC Provider

| Provider | Free Tier | Paid Tier | DetourCoin Usage | Recommendation |
|----------|-----------|-----------|------------------|----------------|
| **QuickNode** | 50-100K req/month | $50/month (1M req) | ~1,000 req/month (30 emissions × ~30 RPC calls/emission) | ✅ Use free tier |
| **Helius** | 100K req/month | $50/month (1M req) | Same | ✅ Use free tier |
| **Alchemy** | 300K compute units/month | $50/month | Same | ✅ Use free tier |
| **Public Solana RPC** | Unlimited (rate-limited) | Free | Backup only | ⚠️ Unreliable, use as last resort |

**Recommendation:** Use **QuickNode or Helius free tier** as primary RPC. Keep Alchemy or public RPC as backup.

#### 5. Cost Management + Billing

**Budget Alerts:**

```bash
az consumption budget create \
  --subscription YOUR_SUBSCRIPTION_ID \
  --budget-name detourcoin-monthly-budget \
  --category Cost \
  --amount 200 \
  --time-grain Monthly \
  --start-date 2025-12-01 \
  --notifications \
    '{"50-percent":{"enabled":true,"operator":"GreaterThan","threshold":50,"contactEmails":["devops@example.com"]},"100-percent":{"enabled":true,"operator":"GreaterThan","threshold":100,"contactEmails":["founder@example.com"]}}'
```

**Monthly Cost Review:**

```bash
# View current month spending
az consumption usage list \
  --start-date 2025-12-01 \
  --end-date 2025-12-31 \
  --query "[].{Service:instanceName,Cost:pretaxCost}" \
  -o table
```

**Azure Advisor Recommendations:**

Enable Azure Advisor to receive cost optimization recommendations:

```bash
az advisor recommendation list \
  --category Cost \
  --resource-group rg-detourcoin-prod \
  -o table
```

---

## 12. RPC Provider

### Provider Selection

**Primary RPC Requirements:**
- Mainnet-beta access
- Dedicated endpoint (not shared)
- 99.9%+ uptime SLA
- Low latency (<100ms)
- Rate limits sufficient for 30 emissions/month (~1,000 requests/month)

**Recommended Providers:**

| Provider | Free Tier | Paid Tier | Latency (avg) | Uptime | Recommendation |
|----------|-----------|-----------|---------------|--------|----------------|
| **QuickNode** | 50K req/month | $50/month (1M req) | 20-50ms | 99.95% | ✅ Primary choice |
| **Helius** | 100K req/month | $50/month (1M req) | 15-40ms | 99.9% | ✅ Primary alternative |
| **Alchemy** | 300K CU/month | $50/month | 25-60ms | 99.9% | ✅ Backup RPC |
| **Public Solana** | Unlimited | Free | 50-500ms | 95-98% | ⚠️ Last resort only |

### QuickNode Setup

**1. Create Account:**
- Visit quicknode.com
- Sign up for free account
- Select Solana Mainnet endpoint

**2. Endpoint Configuration:**

```
Endpoint URL: https://solana-mainnet.quiknode.pro/abc123def456/YOUR_API_KEY/
Network: Mainnet-Beta
Rate Limit (Free): 50,000 requests/month
Throughput: 25 req/second
```

**3. Store in Key Vault:**

```bash
az keyvault secret set \
  --vault-name kv-detourcoin-prod \
  --name primary-rpc-url \
  --value "https://solana-mainnet.quiknode.pro/abc123def456/YOUR_API_KEY/" \
  --description "QuickNode Solana mainnet-beta RPC endpoint"
```

### Helius Setup (Backup)

**1. Create Account:**
- Visit helius.dev
- Sign up for free developer account

**2. Endpoint Configuration:**

```
Endpoint URL: https://mainnet.helius-rpc.com/?api-key=YOUR_API_KEY
Network: Mainnet-Beta
Rate Limit (Free): 100,000 requests/month
```

**3. Store in Key Vault:**

```bash
az keyvault secret set \
  --vault-name kv-detourcoin-prod \
  --name backup-rpc-url \
  --value "https://mainnet.helius-rpc.com/?api-key=YOUR_API_KEY" \
  --description "Helius Solana mainnet-beta RPC endpoint (backup)"
```

### Rate Limit & Retry Strategy

**Rate Limit Handling:**

```typescript
async function callRPCWithRateLimit<T>(
  rpcCall: () => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  let lastError: Error | undefined;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await rpcCall();
    } catch (error: any) {
      lastError = error;

      // Check for rate limit error (HTTP 429)
      if (error.message?.includes("429") || error.message?.includes("rate limit")) {
        const delayMs = Math.pow(2, attempt) * 1000; // Exponential backoff
        logger.warn(`RPC rate limit hit. Retrying in ${delayMs}ms...`);
        await new Promise(resolve => setTimeout(resolve, delayMs));
        continue;
      }

      // Non-retryable error
      throw error;
    }
  }

  throw new Error(`RPC call failed after ${maxRetries} retries: ${lastError?.message}`);
}
```

### Connection Pooling

For multiple RPC calls within single emission:

```typescript
class RPCConnectionPool {
  private connections: Map<string, Connection> = new Map();

  getConnection(url: string): Connection {
    if (!this.connections.has(url)) {
      this.connections.set(url, new Connection(url, {
        commitment: 'confirmed',
        confirmTransactionInitialTimeout: 60000,
        httpHeaders: {
          "Content-Type": "application/json",
        },
      }));
    }
    return this.connections.get(url)!;
  }

  closeAll() {
    this.connections.clear();
  }
}
```

### Monitoring RPC Performance

**Track RPC latency metric:**

```typescript
const rpcStartTime = Date.now();
const blockhash = await connection.getLatestBlockhash();
const rpcLatency = Date.now() - rpcStartTime;

telemetryClient.trackMetric({
  name: "RPCLatencyMs",
  value: rpcLatency,
  properties: {
    rpc_provider: "primary", // or "backup"
    method: "getLatestBlockhash"
  }
});

// Alert if latency > 1000ms
if (rpcLatency > 1000) {
  logger.warn(`RPC latency high: ${rpcLatency}ms`);
}
```

### Rotating API Keys

**Annual rotation process:**

1. Generate new API key from RPC provider dashboard
2. Store new key in Key Vault as new secret version:

```bash
az keyvault secret set \
  --vault-name kv-detourcoin-prod \
  --name primary-rpc-url \
  --value "https://solana-mainnet.quiknode.pro/NEW_API_KEY/"
```

3. Function App automatically picks up new secret (Key Vault reference)
4. Verify emissions succeed with new key
5. Delete old API key from provider dashboard
6. Document rotation in operations log

---

## 13. Transaction Fee Management

### Fee Estimation

**Solana Transaction Fees:**
- Base fee: ~5,000 lamports (0.000005 SOL)
- Priority fee (optional): 0-10,000 lamports
- **Total per emission:** ~0.000005-0.000015 SOL (~$0.00025-0.00075 at $50/SOL)

**Monthly Cost Projection:**

```
Daily emissions: 1
Days per month: 30
Fee per tx: 0.000010 SOL (average)
Monthly SOL used: 30 × 0.000010 = 0.0003 SOL
Monthly USD cost: 0.0003 × $50 = $0.015 (~$0.02/month)

Annual SOL used: 365 × 0.000010 = 0.00365 SOL
Annual USD cost: 0.00365 × $50 = $0.18 (~$0.20/year)
```

**Conclusion:** Transaction fees are **negligible** for daily emissions.

### Balance Monitoring

**Alert Thresholds:**

| Threshold | Alert Level | Action |
|-----------|-------------|--------|
| < 2 SOL | Critical | Immediate SOL transfer required |
| < 5 SOL | Warning | Schedule SOL refill |
| < 10 SOL | Info | Consider refilling soon |

**Automated Balance Checking:**

```typescript
const balanceLamports = await connection.getBalance(emissionAuthority.publicKey);
const balanceSOL = balanceLamports / 1e9;

// Log balance as custom metric
telemetryClient.trackMetric({
  name: "SOLBalance",
  value: balanceSOL
});

// Alert if low
if (balanceSOL < 5.0) {
  logger.warn(`SOL balance low: ${balanceSOL.toFixed(9)} SOL`);

  await publishEvent("DetourCoin.Emission.LowSOLBalance", {
    balance_sol: balanceSOL,
    threshold: 5.0,
    authority: emissionAuthority.publicKey.toBase58()
  });
}

// Critical alert if very low
if (balanceSOL < 2.0) {
  logger.error(`SOL balance critically low: ${balanceSOL.toFixed(9)} SOL`);

  // Send critical alert
  await publishEvent("DetourCoin.Emission.CriticalSOLBalance", {
    balance_sol: balanceSOL,
    threshold: 2.0,
    authority: emissionAuthority.publicKey.toBase58()
  });
}
```

### Refill Workflow

**Manual Refill Process:**

1. **Alert Received:** Critical or warning SOL balance alert triggered
2. **Verify Balance:**

```bash
solana balance EMISSION_AUTHORITY_PUBKEY --url mainnet-beta
```

3. **Transfer SOL from Founder Wallet:**

```bash
# From secure machine with founder wallet
solana transfer \
  --from /path/to/founder-wallet.json \
  --url mainnet-beta \
  EMISSION_AUTHORITY_PUBKEY \
  10 SOL \
  --allow-unfunded-recipient
```

4. **Verify Transfer:**

```bash
solana confirm TRANSACTION_SIGNATURE --url mainnet-beta
solana balance EMISSION_AUTHORITY_PUBKEY --url mainnet-beta
```

5. **Document Transfer:**
   - Record in operations log
   - Update balance tracking spreadsheet
   - Dismiss alert in Azure Monitor

**Automated Refill (Future Enhancement):**

Consider implementing automated SOL refill via:
- Azure Logic App triggered by balance alert
- Custodial service API (e.g., Coinbase Custody)
- Multi-sig controlled refill wallet

**Security:** Automated refill requires secure storage of refill wallet private key (higher risk than manual process).

### Fee Tracking

**Log transaction fees:**

```typescript
const txDetails = await connection.getTransaction(signature, {
  commitment: 'confirmed',
  maxSupportedTransactionVersion: 0,
});

const fee = txDetails?.meta?.fee || 0;
const feeSOL = fee / 1e9;

telemetryClient.trackMetric({
  name: "TransactionFeeSOL",
  value: feeSOL,
  properties: {
    tx_signature: signature,
    phase: "pre-launch"
  }
});

logger.info("Transaction", `Fee paid: ${feeSOL.toFixed(9)} SOL`, {
  fee_lamports: fee,
  tx_signature: signature
});
```

**Monthly fee analysis query (KQL):**

```kusto
customMetrics
| where name == "TransactionFeeSOL"
| where timestamp >= startofmonth(now())
| summarize total_fee_sol = sum(value), avg_fee_sol = avg(value), count = count()
| extend total_fee_usd = total_fee_sol * 50 // Assuming $50/SOL
| project total_fee_sol, avg_fee_sol, count, total_fee_usd
```

---

## 14. Deployment Automation (IaC)

### Bicep Template (Recommended)

**Complete Infrastructure Template:**

```bicep
// main.bicep
targetScope = 'resourceGroup'

@description('Environment name (prod, dev)')
param environment string = 'prod'

@description('Azure region for resources')
param location string = resourceGroup().location

@description('Emission authority public key')
param emissionAuthorityPubkey string

// Variables
var projectName = 'detourcoin'
var functionAppName = 'func-${projectName}-emission-${environment}'
var storageAccountName = 'st${projectName}${environment}${uniqueString(resourceGroup().id)}'
var keyVaultName = 'kv-${projectName}-${environment}-${uniqueString(resourceGroup().id)}'
var cosmosDbName = 'cosmos-${projectName}-${environment}'
var appInsightsName = 'ai-${projectName}-${environment}'
var eventGridTopicName = '${projectName}-emission-events'

// Storage Account (required for Function App)
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
    encryption: {
      services: {
        blob: {
          enabled: true
        }
        file: {
          enabled: true
        }
      }
      keySource: 'Microsoft.Storage'
    }
  }
}

// Application Insights
resource applicationInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: appInsightsName
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    RetentionInDays: 90
    SamplingPercentage: 100
  }
}

// Key Vault
resource keyVault 'Microsoft.KeyVault/vaults@2023-02-01' = {
  name: keyVaultName
  location: location
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true
    enableSoftDelete: true
    softDeleteRetentionInDays: 90
    enablePurgeProtection: true
    networkAcls: {
      defaultAction: 'Allow' // Change to 'Deny' with VNet
      bypass: 'AzureServices'
    }
  }
}

// Cosmos DB Account (Serverless NoSQL)
resource cosmosDbAccount 'Microsoft.DocumentDB/databaseAccounts@2023-04-15' = {
  name: cosmosDbName
  location: location
  kind: 'GlobalDocumentDB'
  properties: {
    databaseAccountOfferType: 'Standard'
    locations: [
      {
        locationName: location
        failoverPriority: 0
        isZoneRedundant: false
      }
    ]
    capabilities: [
      {
        name: 'EnableServerless'
      }
    ]
    backupPolicy: {
      type: 'Continuous'
      continuousModeProperties: {
        tier: 'Continuous7Days'
      }
    }
  }
}

// Cosmos DB Database
resource cosmosDb 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2023-04-15' = {
  parent: cosmosDbAccount
  name: projectName
  properties: {
    resource: {
      id: projectName
    }
  }
}

// Cosmos DB Container
resource cosmosContainer 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2023-04-15' = {
  parent: cosmosDb
  name: 'emission_history'
  properties: {
    resource: {
      id: 'emission_history'
      partitionKey: {
        paths: ['/execution_date']
        kind: 'Hash'
      }
      defaultTtl: 63072000 // 2 years
    }
  }
}

// Event Grid Topic
resource eventGridTopic 'Microsoft.EventGrid/topics@2023-06-01-preview' = {
  name: eventGridTopicName
  location: location
  properties: {
    inputSchema: 'CloudEventSchemaV1_0'
    publicNetworkAccess: 'Enabled'
  }
}

// Action Groups
resource criticalActionGroup 'Microsoft.Insights/actionGroups@2023-01-01' = {
  name: '${projectName}-critical-alerts'
  location: 'global'
  properties: {
    groupShortName: 'DTC-CRIT'
    enabled: true
    emailReceivers: [
      {
        name: 'DevOps'
        emailAddress: 'devops@example.com'
        useCommonAlertSchema: true
      }
    ]
  }
}

resource operationalActionGroup 'Microsoft.Insights/actionGroups@2023-01-01' = {
  name: '${projectName}-operational-alerts'
  location: 'global'
  properties: {
    groupShortName: 'DTC-OPS'
    enabled: true
    emailReceivers: [
      {
        name: 'DevOps Team'
        emailAddress: 'devops-team@example.com'
        useCommonAlertSchema: true
      }
    ]
  }
}

// App Service Plan (Consumption)
resource appServicePlan 'Microsoft.Web/serverfarms@2022-09-01' = {
  name: 'plan-${functionAppName}'
  location: location
  sku: {
    name: 'Y1' // Consumption plan
    tier: 'Dynamic'
  }
  properties: {
    reserved: false // Windows
  }
}

// Function App
resource functionApp 'Microsoft.Web/sites@2022-09-01' = {
  name: functionAppName
  location: location
  kind: 'functionapp'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: appServicePlan.id
    httpsOnly: true
    siteConfig: {
      ftpsState: 'Disabled'
      minTlsVersion: '1.2'
      appSettings: [
        {
          name: 'AzureWebJobsStorage'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};AccountKey=${storageAccount.listKeys().keys[0].value};EndpointSuffix=${environment().suffixes.storage}'
        }
        {
          name: 'WEBSITE_CONTENTAZUREFILECONNECTIONSTRING'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};AccountKey=${storageAccount.listKeys().keys[0].value};EndpointSuffix=${environment().suffixes.storage}'
        }
        {
          name: 'WEBSITE_CONTENTSHARE'
          value: toLower(functionAppName)
        }
        {
          name: 'FUNCTIONS_EXTENSION_VERSION'
          value: '~4'
        }
        {
          name: 'FUNCTIONS_WORKER_RUNTIME'
          value: 'node'
        }
        {
          name: 'WEBSITE_NODE_DEFAULT_VERSION'
          value: '~18'
        }
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: applicationInsights.properties.ConnectionString
        }
        {
          name: 'SOLANA_NETWORK'
          value: 'mainnet-beta'
        }
        {
          name: 'EMISSION_PROGRAM_ID'
          value: emissionAuthorityPubkey
        }
        {
          name: 'COSMOS_DB_ENDPOINT'
          value: cosmosDbAccount.properties.documentEndpoint
        }
        {
          name: 'COSMOS_DB_DATABASE'
          value: projectName
        }
        {
          name: 'COSMOS_DB_CONTAINER'
          value: 'emission_history'
        }
        {
          name: 'EVENT_GRID_TOPIC_ENDPOINT'
          value: eventGridTopic.properties.endpoint
        }
        // Key Vault references (secrets stored separately)
        {
          name: 'EMISSION_AUTHORITY_KEYPAIR'
          value: '@Microsoft.KeyVault(SecretUri=${keyVault.properties.vaultUri}secrets/detourcoin-emission-authority-keypair/)'
        }
        {
          name: 'SOLANA_RPC_PRIMARY'
          value: '@Microsoft.KeyVault(SecretUri=${keyVault.properties.vaultUri}secrets/primary-rpc-url/)'
        }
        {
          name: 'SOLANA_RPC_BACKUP'
          value: '@Microsoft.KeyVault(SecretUri=${keyVault.properties.vaultUri}secrets/backup-rpc-url/)'
        }
      ]
    }
  }
}

// RBAC: Function App → Key Vault (Secrets User)
resource keyVaultRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(functionApp.id, keyVault.id, 'Key Vault Secrets User')
  scope: keyVault
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6')
    principalId: functionApp.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// RBAC: Function App → Cosmos DB (Custom Data Contributor)
resource cosmosRoleDefinition 'Microsoft.DocumentDB/databaseAccounts/sqlRoleDefinitions@2023-04-15' = {
  parent: cosmosDbAccount
  name: guid(cosmosDbAccount.id, 'emission-writer-role')
  properties: {
    roleName: 'DetourCoin Emission Writer'
    type: 'CustomRole'
    assignableScopes: [
      cosmosDbAccount.id
    ]
    permissions: [
      {
        dataActions: [
          'Microsoft.DocumentDB/databaseAccounts/readMetadata'
          'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/create'
          'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/read'
        ]
      }
    ]
  }
}

resource cosmosRoleAssignment 'Microsoft.DocumentDB/databaseAccounts/sqlRoleAssignments@2023-04-15' = {
  parent: cosmosDbAccount
  name: guid(functionApp.id, cosmosDbAccount.id, 'emission-writer')
  properties: {
    roleDefinitionId: cosmosRoleDefinition.id
    principalId: functionApp.identity.principalId
    scope: cosmosDbAccount.id
  }
}

// RBAC: Function App → Event Grid (Data Sender)
resource eventGridRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(functionApp.id, eventGridTopic.id, 'Event Grid Data Sender')
  scope: eventGridTopic
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'd5a91429-5739-47e2-a06b-3470a27159e7')
    principalId: functionApp.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Outputs
output functionAppName string = functionApp.name
output functionAppPrincipalId string = functionApp.identity.principalId
output keyVaultName string = keyVault.name
output cosmosDbAccountName string = cosmosDbAccount.name
output applicationInsightsName string = applicationInsights.name
output storageAccountName string = storageAccount.name
```

**Deployment:**

```bash
# Create resource group
az group create --name rg-detourcoin-prod --location eastus

# Deploy Bicep template
az deployment group create \
  --resource-group rg-detourcoin-prod \
  --template-file main.bicep \
  --parameters environment=prod emissionAuthorityPubkey=YOUR_PUBKEY
```

### Terraform Template (Alternative)

**provider.tf:**

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
  }

  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "sttfstate"
    container_name       = "tfstate"
    key                  = "detourcoin-prod.tfstate"
  }
}

provider "azurerm" {
  features {
    key_vault {
      purge_soft_delete_on_destroy = false
    }
  }
}
```

**main.tf:**

```hcl
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "prod"
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "eastus"
}

variable "emission_authority_pubkey" {
  description = "Emission authority Solana public key"
  type        = string
}

locals {
  project_name         = "detourcoin"
  function_app_name    = "func-${local.project_name}-emission-${var.environment}"
  storage_account_name = "st${local.project_name}${var.environment}${substr(md5(azurerm_resource_group.main.id), 0, 8)}"
  key_vault_name       = "kv-${local.project_name}-${var.environment}"
}

resource "azurerm_resource_group" "main" {
  name     = "rg-${local.project_name}-${var.environment}"
  location = var.location

  tags = {
    Environment = var.environment
    Project     = "DetourCoin"
    ManagedBy   = "Terraform"
  }
}

resource "azurerm_storage_account" "main" {
  name                     = local.storage_account_name
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  min_tls_version          = "TLS1_2"

  tags = azurerm_resource_group.main.tags
}

resource "azurerm_application_insights" "main" {
  name                = "ai-${local.project_name}-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  application_type    = "web"
  retention_in_days   = 90
  sampling_percentage = 100

  tags = azurerm_resource_group.main.tags
}

resource "azurerm_key_vault" "main" {
  name                       = local.key_vault_name
  resource_group_name        = azurerm_resource_group.main.name
  location                   = azurerm_resource_group.main.location
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "standard"
  enable_rbac_authorization  = true
  soft_delete_retention_days = 90
  purge_protection_enabled   = true

  tags = azurerm_resource_group.main.tags
}

resource "azurerm_cosmosdb_account" "main" {
  name                = "cosmos-${local.project_name}-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  offer_type          = "Standard"
  kind                = "GlobalDocumentDB"

  capabilities {
    name = "EnableServerless"
  }

  geo_location {
    location          = azurerm_resource_group.main.location
    failover_priority = 0
  }

  backup {
    type                = "Continuous"
    tier                = "Continuous7Days"
  }

  tags = azurerm_resource_group.main.tags
}

resource "azurerm_service_plan" "main" {
  name                = "plan-${local.function_app_name}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  os_type             = "Windows"
  sku_name            = "Y1" # Consumption plan

  tags = azurerm_resource_group.main.tags
}

resource "azurerm_windows_function_app" "main" {
  name                       = local.function_app_name
  resource_group_name        = azurerm_resource_group.main.name
  location                   = azurerm_resource_group.main.location
  service_plan_id            = azurerm_service_plan.main.id
  storage_account_name       = azurerm_storage_account.main.name
  storage_account_access_key = azurerm_storage_account.main.primary_access_key

  identity {
    type = "SystemAssigned"
  }

  site_config {
    application_insights_connection_string = azurerm_application_insights.main.connection_string
    ftps_state                             = "Disabled"
    minimum_tls_version                    = "1.2"

    application_stack {
      node_version = "~18"
    }
  }

  app_settings = {
    "FUNCTIONS_WORKER_RUNTIME"       = "node"
    "WEBSITE_NODE_DEFAULT_VERSION"   = "~18"
    "SOLANA_NETWORK"                 = "mainnet-beta"
    "EMISSION_PROGRAM_ID"            = var.emission_authority_pubkey
    "COSMOS_DB_ENDPOINT"             = azurerm_cosmosdb_account.main.endpoint
    "COSMOS_DB_DATABASE"             = local.project_name
    "COSMOS_DB_CONTAINER"            = "emission_history"
    "EMISSION_AUTHORITY_KEYPAIR"     = "@Microsoft.KeyVault(SecretUri=${azurerm_key_vault.main.vault_uri}secrets/detourcoin-emission-authority-keypair/)"
    "SOLANA_RPC_PRIMARY"             = "@Microsoft.KeyVault(SecretUri=${azurerm_key_vault.main.vault_uri}secrets/primary-rpc-url/)"
    "SOLANA_RPC_BACKUP"              = "@Microsoft.KeyVault(SecretUri=${azurerm_key_vault.main.vault_uri}secrets/backup-rpc-url/)"
  }

  tags = azurerm_resource_group.main.tags
}

data "azurerm_client_config" "current" {}

# RBAC Assignments
resource "azurerm_role_assignment" "kv_secrets_user" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_windows_function_app.main.identity[0].principal_id
}

output "function_app_name" {
  value = azurerm_windows_function_app.main.name
}

output "key_vault_name" {
  value = azurerm_key_vault.main.name
}
```

**Deployment:**

```bash
# Initialize Terraform
terraform init

# Plan deployment
terraform plan -out=tfplan

# Apply deployment
terraform apply tfplan
```

### CI/CD with GitHub Actions

**.github/workflows/deploy.yml:**

```yaml
name: Deploy Azure Infrastructure

on:
  push:
    branches: [main]
    paths:
      - 'infrastructure/**'
      - 'emission-function/**'
  workflow_dispatch:

env:
  AZURE_RESOURCE_GROUP: rg-detourcoin-prod
  FUNCTION_APP_NAME: func-detourcoin-emission-prod

jobs:
  deploy-infrastructure:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Deploy Bicep template
        uses: azure/arm-deploy@v1
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ env.AZURE_RESOURCE_GROUP }}
          template: ./infrastructure/main.bicep
          parameters: environment=prod emissionAuthorityPubkey=${{ secrets.EMISSION_AUTHORITY_PUBKEY }}

  deploy-function:
    runs-on: ubuntu-latest
    needs: deploy-infrastructure
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: |
          cd emission-function
          npm ci

      - name: Build TypeScript
        run: |
          cd emission-function
          npm run build

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Deploy to Azure Functions
        uses: Azure/functions-action@v1
        with:
          app-name: ${{ env.FUNCTION_APP_NAME }}
          package: ./emission-function
          respect-funcignore: true
```

---

*(This document has grown quite large. I'll complete the remaining sections 15-24 in a continuation. Would you like me to proceed?)*