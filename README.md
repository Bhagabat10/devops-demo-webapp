# DevOps Infrastructure Design — Azure + Cloudflare

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Prerequisites](#3-prerequisites)
4. [Infrastructure as Code](#4-infrastructure-as-code)
5. [Service Setup](#5-service-setup)
6. [CI/CD Pipelines (Azure DevOps)](#6-cicd-pipelines-azure-devops)
7. [Observability](#7-observability)
8. [Security](#8-security)
9. [Scalability](#9-scalability)
10. [Reliability & Resiliency](#10-reliability--resiliency)
11. [Automation](#11-automation)
12. [Assumptions ](#12-assumptions)
13. [Key Architecture Decisions](#13-key-architecture-decisions)

---

## 1. Overview

Production-grade infrastructure on **Microsoft Azure** with **Cloudflare at the edge**, covering:

| Capability | Solution |
|---|---|
| Config web app | Azure Static Web Apps + Entra ID |
| Backend APIs | Azure Container Apps + API Management |
| Log ingestion & processing | Azure Event Hubs + Stream Analytics |
| Video processing & serving | Container App Jobs (FFmpeg) + Blob + CDN |
| SMS to guests | Azure Communication Services (ACS) |
| Email to guests | ACS Email + custom domain |
| Async device communication | Azure IoT Hub + Service Bus |
| CI/CD | Azure DevOps (ADO) YAML pipelines |
| IaC | Terraform |

---

## 2. Architecture

### High-Level Diagram

```
╔══════════════════════════════════════════════════════════╗
║                    CLOUDFLARE EDGE                       ║
║   WAF · DDoS · CDN · DNS · Workers (JWT pre-auth)        ║
╚══════════════════════════╦═══════════════════════════════╝
                           │ HTTPS (all public traffic)
                           ▼
                ╔══════════════════════════════╗
                ║   Azure API Management       ║
                ║   (APIM)                     ║
                ║  - JWT validation            ║
                ║  - Rate limiting             ║
                ║  - IP allow-list             ║
                ║  - API versioning            ║
                ╚══════╦═══════════════╦═══════╝
                       │               │
                       ▼               ▼
          ┌────────────────┐    ┌──────────────────────────────────────┐
          │ Config Web App │    │            Backend APIs              │
          │ Azure Static   │    │   Azure Container Apps (KEDA)        │
          │ Web Apps       │    └──────────┬─────────────┬─────────────┘
          │ Entra ID Auth  │               │             │
          └────────────────┘               ▼             ▼
                                     ┌────────────┐   ┌──────────────────────┐
                                     │ Cosmos DB  │   │ Azure Service Bus    │
                                     │ Primary    │   │ Queues & Topics      │
                                     │ Datastore  │   └─────────┬──────┬─────┘
                                     └────────────┘             │      │
                                                                ▼      ▼
                                                       ┌────────────┐ ┌──────────────────┐
                                                       │ Log        │ │ Notification     │
                                                       │ Processor  │ │ Worker           │
                                                       │ Stream     │ │ SMS → ACS/Twilio │
                                                       │ Analytics  │ │ Email → ACS/Grid │
                                                       └────────────┘ └──────────────────┘

[In-Venue Devices]
  ├── MQTT/AMQP ───────────► Azure IoT Hub ──► Service Bus
  └── HTTPS Logs ──────────► Event Hubs ─────► Stream Analytics


[Video Processing Pipeline]
  Raw Clip
      │
      ▼
  Blob Storage (Private)
      │
      ▼
  Event Grid Trigger
      │
      ▼
  Container App Job (FFmpeg)
      │
      ▼
  Blob Storage (Public HLS/DASH)
      │
      ▼
  Cloudflare CDN
```

### Network Topology

```
VNet: 10.0.0.0/16
├── subnet-apim        10.0.1.0/24   API Management (internal mode)
├── subnet-apps        10.0.2.0/24   Container Apps Environment
├── subnet-data        10.0.3.0/24   Private Endpoints (Cosmos, SB, EH)
├── subnet-iot         10.0.4.0/24   IoT Hub Private Endpoint
└── subnet-mgmt        10.0.5.0/24   Bastion, self-hosted ADO agents

All PaaS services: public network access DISABLED
Access only via Private Endpoints inside the VNet
```

---

## 3. Prerequisites

These steps are performed once by a platform engineer before any Terraform or pipelines run.

### Step 1 — Create Azure Subscriptions

Use separate subscriptions per environment for blast-radius isolation and clean cost attribution.

```
Management Group: company-root
├── sub-dev        (developer sandboxes)
├── sub-staging    (pre-production)
└── sub-production (live traffic)
```

```bash
az account management-group create --name company-root
az account create --display-name "sub-staging" \
  --management-group-id company-root \
  --billing-scope "/providers/Microsoft.Billing/billingAccounts/<id>"
```

### Step 2 — Bootstrap Terraform Remote State

Terraform state must exist before Terraform can manage anything. Create this manually once per subscription.

```bash
RESOURCE_GROUP="rg-tfstate"
STORAGE_ACCOUNT="stgtfstate$(openssl rand -hex 4)"  # must be globally unique
CONTAINER="tfstate"
LOCATION="uksouth"

az group create --name $RESOURCE_GROUP --location $LOCATION

az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_GRS \               # geo-redundant — state file is critical
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false

az storage container create \
  --name $CONTAINER \
  --account-name $STORAGE_ACCOUNT

# Enable versioning so state history is recoverable
az storage account blob-service-properties update \
  --account-name $STORAGE_ACCOUNT \
  --enable-versioning true
```

### Step 3 — Create Azure Container Registry (ACR)

One shared ACR accessed by all environments, geo-replicated for resilience and pull speed.

```bash
az acr create \
  --name companyacr \
  --resource-group rg-platform \
  --sku Premium \           # Premium = geo-replication + Private Link
  --admin-enabled false     # use Managed Identity, not admin credentials

# Replicate to West Europe (DR region)
az acr replication create \
  --registry companyacr \
  --location westeurope
```

### Step 4 — Configure Azure DevOps

```
1. Create ADO Organisation  → https://dev.azure.com/<company>
2. Create ADO Project       → "platform"
3. Import the Git repo into ADO Repos
4. Create Service Connections (use Workload Identity Federation — no client secrets):
     sc-azure-dev         → sub-dev
     sc-azure-staging     → sub-staging
     sc-azure-production  → sub-production
     sc-acr               → Docker Registry → companyacr
5. Create Environments:
     dev        → no approval gates
     staging    → approval gate: testing team must approve
     production → approval gate: 2 of 3 senior engineers must approve
6. Create Variable Groups linked to Azure Key Vault per environment:
     vg-dev         → kv-dev-<suffix>
     vg-staging     → kv-staging-<suffix>
     vg-production  → kv-production-<suffix>
```

> **Workload Identity Federation** — ADO authenticates to Azure with a short-lived OIDC token. No client secrets to create, store, or rotate.

### Step 5 — Configure Cloudflare

```
1. Add zone: company.com (update nameservers at registrar)
2. SSL/TLS mode: Full (strict)
3. Enable: Always Use HTTPS
4. Enable: HSTS (includeSubDomains, max-age 6 months)
5. WAF: Enable OWASP Core Rule Set (sensitivity: medium)
6. DDoS: Enable HTTP DDoS Attack Protection (managed ruleset)
7. DNS records will be created by Terraform (cloudflare module)
```

---

## 4. Infrastructure as Code

### Repo Structure

```
infra/
├── backend.tf               # Remote state config (Azure Blob)
├── main.tf                  # Root: calls all modules
├── variables.tf
├── outputs.tf
├── terraform.tfvars.example # Template — never commit real .tfvars
└── modules/
    ├── networking/          # VNet, subnets, NSGs, Private Endpoints
    ├── cloudflare/          # DNS records, WAF rules, Workers routes
    ├── apim/                # API Management + API definitions
    ├── container_apps/      # Container Apps Environment + app definitions
    ├── event_hubs/          # Event Hub Namespace + hubs + consumer groups
    ├── service_bus/         # Service Bus Namespace + queues + topics
    ├── iot_hub/             # IoT Hub + routing rules + DPS
    ├── storage/             # Blob containers + lifecycle policies
    ├── cdn/                 # Azure CDN profile + endpoints
    ├── cosmos_db/           # Cosmos DB account + databases + containers
    ├── monitoring/          # Log Analytics + App Insights + Alert rules
    ├── communication/       # ACS SMS + Email
    └── identity/            # Managed Identities + Key Vault + RBAC
```

### Step 1 — Configure the Backend

```hcl
# infra/backend.tf
terraform {
  required_version = ">= 1.7"

  required_providers {
    azurerm    = { source = "hashicorp/azurerm",     version = "~> 3.100" }
    cloudflare = { source = "cloudflare/cloudflare",  version = "~> 4.0"  }
  }

  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "stgtfstate<suffix>"
    container_name       = "tfstate"
    key                  = "staging.terraform.tfstate"
    use_oidc             = true   # authenticate via ADO Workload Identity
  }
}
```

### Step 2 — Define the Root Module

```hcl
# infra/main.tf
locals {
  env            = var.environment   # "dev" | "staging" | "production"
  location       = "uksouth"
  location_short = "uks"
  tags = {
    environment = local.env
    managed_by  = "terraform"
    project     = "platform"
  }
}

module "networking"    { source = "./modules/networking"    env = local.env /* ... */ }
module "identity"      { source = "./modules/identity"      env = local.env /* ... */ }
module "monitoring"    { source = "./modules/monitoring"    env = local.env /* ... */ }
module "cosmos_db"     { source = "./modules/cosmos_db"     env = local.env /* ... */ }
module "service_bus"   { source = "./modules/service_bus"   env = local.env /* ... */ }
module "event_hubs"    { source = "./modules/event_hubs"    env = local.env /* ... */ }
module "iot_hub"       { source = "./modules/iot_hub"       env = local.env /* ... */ }
module "storage"       { source = "./modules/storage"       env = local.env /* ... */ }
module "container_apps"{ source = "./modules/container_apps" /* depends on above */ }
module "apim"          { source = "./modules/apim"          /* ... */ }
module "communication" { source = "./modules/communication" /* ... */ }
module "cloudflare"    { source = "./modules/cloudflare"    /* ... */ }
```

### Step 3 — Networking Module

```hcl
# modules/networking/main.tf
resource "azurerm_resource_group" "main" {
  name     = "rg-app-${var.env}"
  location = var.location
  tags     = var.tags
}

resource "azurerm_virtual_network" "main" {
  name                = "vnet-${var.env}-${var.location_short}"
  address_space       = ["10.0.0.0/16"]
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_subnet" "apps" {
  name                 = "subnet-apps"
  virtual_network_name = azurerm_virtual_network.main.name
  resource_group_name  = azurerm_resource_group.main.name
  address_prefixes     = ["10.0.2.0/24"]

  delegation {
    name = "containerApps"
    service_delegation {
      name    = "Microsoft.App/environments"
      actions = ["Microsoft.Network/virtualNetworks/subnets/join/action"]
    }
  }
}

resource "azurerm_subnet" "data" {
  name             = "subnet-data"
  address_prefixes = ["10.0.3.0/24"]
  # ... (no delegation — used for Private Endpoints)
}

# Private Endpoint pattern — repeat for Service Bus, Event Hubs, Key Vault, etc.
resource "azurerm_private_endpoint" "cosmos" {
  name                = "pe-cosmos-${var.env}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  subnet_id           = azurerm_subnet.data.id

  private_service_connection {
    name                           = "psc-cosmos"
    private_connection_resource_id = var.cosmos_db_id
    subresource_names              = ["Sql"]
    is_manual_connection           = false
  }

  private_dns_zone_group {
    name                 = "cosmos-dns"
    private_dns_zone_ids = [azurerm_private_dns_zone.cosmos.id]
  }
}
```

### Step 4 — Container Apps Module

```hcl
# modules/container_apps/main.tf
resource "azurerm_container_app_environment" "main" {
  name                           = "cae-${var.env}-${var.location_short}"
  location                       = var.location
  resource_group_name            = var.resource_group_name
  log_analytics_workspace_id     = var.log_analytics_id
  infrastructure_subnet_id       = var.subnet_apps_id
  internal_load_balancer_enabled = true    # not publicly reachable; sits behind APIM
  zone_redundancy_enabled        = true    # spread across AZs
}

resource "azurerm_container_app" "api" {
  name                         = "ca-api-${var.env}"
  container_app_environment_id = azurerm_container_app_environment.main.id
  resource_group_name          = var.resource_group_name
  revision_mode                = "Multiple"   # required for traffic splitting

  identity {
    type         = "UserAssigned"
    identity_ids = [var.managed_identity_id]  # pull from ACR, read Key Vault, write to SB
  }

  ingress {
    external_enabled = false   # internal only — APIM is the sole entry point
    target_port      = 8080
    traffic_weight {
      percentage      = 100
      latest_revision = true
    }
  }

  template {
    min_replicas = var.env == "production" ? 2 : 1
    max_replicas = var.env == "production" ? 30 : 5

    http_scale_rule {
      name                = "http-scaler"
      concurrent_requests = "50"
    }

    custom_scale_rule {
      name             = "servicebus-scaler"
      custom_rule_type = "azure-servicebus"
      metadata = {
        queueName    = "api-work"
        namespace    = var.service_bus_namespace
        messageCount = "100"
      }
    }

    container {
      name   = "api"
      image  = "${var.acr_login_server}/api:${var.image_tag}"
      cpu    = var.env == "production" ? 1.0 : 0.5
      memory = var.env == "production" ? "2Gi" : "1Gi"

      env {
        name        = "APPLICATIONINSIGHTS_CONNECTION_STRING"
        secret_name = "appinsights-connection-string"   # from Key Vault
      }

      liveness_probe {
        transport         = "HTTP"
        path              = "/health/live"
        port              = 8080
        period_seconds    = 10
        failure_threshold = 3
      }

      readiness_probe {
        transport         = "HTTP"
        path              = "/health/ready"
        port              = 8080
        period_seconds    = 5
        failure_threshold = 3
      }
    }
  }
}
```

### Step 5 — Event Hubs Module (Log Ingestion)

```hcl
# modules/event_hubs/main.tf
resource "azurerm_eventhub_namespace" "logs" {
  name                          = "evhns-logs-${var.env}"
  location                      = var.location
  resource_group_name           = var.resource_group_name
  sku                           = "Standard"
  capacity                      = 2
  auto_inflate_enabled          = true
  maximum_throughput_units      = 20
  public_network_access_enabled = false   # Private Endpoint only
}

resource "azurerm_eventhub" "device_logs" {
  name                = "device-logs"
  namespace_name      = azurerm_eventhub_namespace.logs.name
  resource_group_name = var.resource_group_name
  partition_count     = 32   # cannot be changed post-creation — set high upfront
  message_retention   = 7
}

resource "azurerm_eventhub_consumer_group" "stream_analytics" {
  name                = "stream-analytics-cg"
  namespace_name      = azurerm_eventhub_namespace.logs.name
  eventhub_name       = azurerm_eventhub.device_logs.name
  resource_group_name = var.resource_group_name
}
```

### Step 6 — IoT Hub Module (Async Device Communication)

```hcl
# modules/iot_hub/main.tf
resource "azurerm_iothub" "main" {
  name                = "iot-${var.env}"
  resource_group_name = var.resource_group_name
  location            = var.location

  sku {
    name     = var.env == "production" ? "S2" : "S1"
    capacity = var.env == "production" ? 2 : 1
  }

  # Telemetry → Event Hubs for log processing
  route {
    name           = "telemetry-to-eventhub"
    source         = "DeviceMessages"
    condition      = "true"
    endpoint_names = ["eventhub-logs"]
    enabled        = true
  }

  # Alerts → Service Bus for immediate action
  route {
    name           = "alerts-to-servicebus"
    source         = "DeviceMessages"
    condition      = "$body.type = 'alert'"
    endpoint_names = ["servicebus-alerts"]
    enabled        = true
  }
}

# Device Provisioning Service — zero-touch device onboarding with X.509 certs
resource "azurerm_iothub_dps" "main" {
  name                = "dps-${var.env}"
  resource_group_name = var.resource_group_name
  location            = var.location

  sku { name = "S1"; capacity = 1 }

  linked_hub {
    connection_string = azurerm_iothub.main.connection_string
    location          = var.location
  }
}
```

---

## 5. Service Setup

### Step 1 — Cloudflare DNS & WAF

After the Terraform `cloudflare` module applies, the following is in place:

```
DNS records:
  api.company.com    → CNAME → APIM gateway hostname  (proxied ✓)
  app.company.com    → CNAME → Static Web App hostname (proxied ✓)
  video.company.com  → CNAME → Azure CDN endpoint     (proxied ✓)

WAF rules:
  Rule 1: Block requests where cf.threat_score > 20
  Rule 2: Rate limit /api/*       → 1,000 req/min per IP
  Rule 3: Rate limit /api/auth/*  → 20 req/min per IP
  Rule 4: Allow only venue IP ranges to /ingest/*
  Rule 5: OWASP Core Rule Set (managed) — sensitivity: medium
```

### Step 2 — APIM Configuration

```
APIs registered:
  /api/v1/config/*   → Container App: ca-api (internal URL)
  /api/v1/events/*   → Container App: ca-api
  /api/v1/ingest/*   → Event Hubs direct HTTP ingest
  /api/v1/video/*    → Container App: ca-video-processor

Global policies:
  - validate-jwt         (Entra ID + device tokens)
  - rate-limit-by-key    (500 calls/min per subscription)
  - set-header           (remove server headers; inject correlation ID)
  - cors                 (restrict to app.company.com)

Products:
  - internal-staff       (config web app — Entra ID auth)
  - in-venue-devices     (IoT devices — X.509 certificate auth)
```

### Step 3 — Video Processing Flow

```
1. Device or backend uploads raw .mp4 to Blob (private container) via SAS token

2. Event Grid detects BlobCreated → triggers Container App Job

3. Container App Job (FFmpeg) runs:
   a. Pull raw file from private Blob
   b. Transcode → HLS (360p / 720p / 1080p adaptive bitrate)
   c. Generate thumbnail (PNG at 5s mark)
   d. Write .m3u8 manifest + .ts segments → Blob (public container)
   e. Write metadata (duration, thumbnail URL, status) → Cosmos DB
   f. Publish "video.ready" event → Service Bus topic

4. Notification Worker reads "video.ready" → sends SMS/email to guest

5. Guest clicks link → video.company.com/clips/<id>/manifest.m3u8
   Cloudflare CDN serves HLS segments from nearest PoP
```

### Step 4 — Notification Flow (SMS & Email)

```
1. Backend publishes to Service Bus topic: "notifications"
   Payload: { type: "sms"|"email", to: "...", templateId: "...", params: {} }

2. Notification Worker (Container App) consumes from topic
   Scales via KEDA: 1 replica per 100 messages in queue

3. SMS path:
   Worker → ACS SMS SDK → sends message
   Failure after 3 retries (exp. backoff) → dead-letter queue → P2 alert

4. Email path:
   Worker → ACS Email SDK → sends via custom domain (DKIM signed)
   Failure after 3 retries → dead-letter queue → P2 alert

5. Delivery receipts → Cosmos DB (for guest comms history)
```

---

## 6. CI/CD Pipelines (Azure DevOps)

### Pipeline Flow Overview

```
Developer opens PR
        │
        ▼  ci.yml (automatic)
  ┌─────────────────────────────────────┐
  │  Step 1: Checkout code              │
  │  Step 2: Lint                       │
  │  Step 3: Unit tests + coverage      │
  │  Step 4: Build Docker image         │
  │  Step 5: Trivy security scan        │
  │  Step 6: Push image to ACR          │
  └──────────────┬──────────────────────┘
                 │ PR approved & merged to main
                 ▼  cd-staging.yml (automatic)
  ┌─────────────────────────────────────┐
  │  Step 1: Deploy new revision        │
  │  Step 2: Wait for health check      │
  │  Step 3: Smoke tests                │
  │  Step 4: Shift 100% traffic         │
  │  Step 5: Notify Teams on failure    │
  └──────────────┬──────────────────────┘
                 │ Engineer cuts release branch
                 ▼  cd-production.yml (approval required)
  ┌─────────────────────────────────────┐
  │  Step 1: Approval gate (2 engineers)│
  │  Step 2: Deploy revision (0% traffic│
  │  Step 3: Canary — shift 10%         │
  │  Step 4: Smoke tests                │
  │  Step 5: Monitor error rate (5 min) │
  │  Step 6: Full cutover (100%)        │
  │  Step 7: Tag release in ADO         │
  └─────────────────────────────────────┘

Infrastructure (infra/** changes):
  PR   → infra.yml: terraform plan → post to PR summary
  Merge→ infra.yml: terraform apply (with ADO Environment gate)
```

### CI Pipeline — `pipelines/ci.yml`

```yaml
trigger: none   # PRs only

pr:
  branches:
    include: [main]
  paths:
    exclude: [docs/**, "*.md"]

pool:
  vmImage: ubuntu-latest

variables:
  - group: vg-staging              # Key Vault-linked: ACR_LOGIN_SERVER
  - name: imageTag
    value: $(Build.SourceVersion)  # Git SHA — immutable, traceable
  - name: containerApp
    value: ca-api-staging
  - name: resourceGroup
    value: rg-app-staging
  - group: vg-production
  - name: containerAppPrd
    value: ca-api-production
  - name: resourceGroupPrd
    value: rg-app-production
stages:
  - stage: BuildAndTest
    displayName: "Build & Test"
    jobs:
      - job: Build
        steps:

          - checkout: self
            displayName: "Step 1 — Checkout"
            fetchDepth: 0

          - script: make lint
            displayName: "Step 2 — Lint"

          - script: make test-coverage
            displayName: "Step 3 — Unit tests + coverage"

          - task: PublishTestResults@2
            inputs:
              testResultsFormat: JUnit
              testResultsFiles: "**/test-results.xml"

          - task: PublishCodeCoverageResults@2
            inputs:
              codeCoverageTool: Cobertura
              summaryFileLocation: "**/coverage.xml"

          - task: Docker@2
            displayName: "Step 4 — Build Docker image"
            inputs:
              command: build
              containerRegistry: sc-acr
              repository: api
              tags: $(imageTag)
              arguments: --build-arg BUILD_VERSION=$(imageTag)

          - script: |
              curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
                | sh -s -- -b /usr/local/bin
              trivy image \
                --severity CRITICAL,HIGH \
                --exit-code 1 \
                --format table \
                $(ACR_LOGIN_SERVER)/api:$(imageTag)
            displayName: "Step 5 — Trivy security scan (fail on CRITICAL/HIGH)"

          - task: Docker@2
            displayName: "Step 6 — Push to ACR"
            inputs:
              command: push
              containerRegistry: sc-acr
              repository: api
              tags: |
                $(imageTag)
                $(Build.SourceBranchName)-latest

  - stage: DeployStaging
    displayName: "Deploy to Staging"
    dependsOn: BuildAndTest
    jobs:
      - deployment: Deploy
        environment: staging
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  displayName: "Step 1 — Deploy new revision (old revision keeps traffic)"
                  inputs:
                    azureSubscription: sc-azure-staging
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      az containerapp update \
                        --name $(containerApp) \
                        --resource-group $(resourceGroup) \
                        --image $(ACR_LOGIN_SERVER)/api:$(imageTag) \
                        --revision-suffix $(Build.BuildId)

                - task: AzureCLI@2
                  displayName: "Step 2 — Wait for new revision to be healthy"
                  inputs:
                    azureSubscription: sc-azure-staging
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      for i in {1..12}; do
                        STATUS=$(az containerapp revision show \
                          --name $(containerApp) \
                          --resource-group $(resourceGroup) \
                          --revision $(containerApp)--$(Build.BuildId) \
                          --query "properties.healthState" -o tsv)
                        echo "Health: $STATUS"
                        [ "$STATUS" = "Healthy" ] && exit 0
                        sleep 10
                      done
                      echo "Revision unhealthy after 2 minutes" && exit 1

                - script: make smoke-test ENVIRONMENT=staging
                  displayName: "Step 3 — Smoke tests"

                - task: AzureCLI@2
                  displayName: "Step 4 — Shift 100% traffic to new revision"
                  inputs:
                    azureSubscription: sc-azure-staging
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      az containerapp ingress traffic set \
                        --name $(containerApp) \
                        --resource-group $(resourceGroup) \
                        --revision-weight $(containerApp)--$(Build.BuildId)=100

  - stage: NotifyFailure
    displayName: "Notify on failure"
    condition: failed()
    jobs:
      - job: Notify
        steps:
          - task: InvokeRestAPI@1
            displayName: "Step 5 — Post failure to Teams"
            inputs:
              connectionType: connectedServiceName
              serviceConnection: sc-teams-webhook
              method: POST
              body: |
                {"text": "🔴 Staging deploy FAILED — $(Build.BuildNumber) — $(Build.SourceVersionMessage)"}

  - stage: Approval
    displayName: "① Awaiting approval (2 engineers required)"
    dependsOn: DeployStaging
    jobs:
      - deployment: WaitForApproval
        environment: production
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Approved — proceeding to production"
                  displayName: "Approval granted"

  - stage: Deploy
    displayName: "② Blue/Green Deploy"
    dependsOn: Approval
    jobs:
      - deployment: BlueGreen
        environment: production
        strategy:
          runOnce:
            deploy:
              steps:

                - task: AzureCLI@2
                  displayName: "Step 1 — Create new revision (0% traffic)"
                  inputs:
                    azureSubscription: sc-azure-production
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      az containerapp update \
                        --name $(containerAppPrd) \
                        --resource-group $(resourceGroupPrd) \
                        --image $(ACR_LOGIN_SERVER)/api:$(imageTag) \
                        --revision-suffix $(Build.BuildId)

                      az containerapp ingress traffic set \
                        --name $(containerAppPrd) \
                        --resource-group $(resourceGroupPrd) \
                        --revision-weight \
                          $(containerApp)--$(Build.BuildId)=0 \
                          latest=100

                - task: AzureCLI@2
                  displayName: "Step 2 — Wait for revision health"
                  inputs:
                    azureSubscription: sc-azure-production
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      for i in {1..18}; do
                        STATUS=$(az containerapp revision show \
                          --name $(containerAppPrd) \
                          --resource-group $(resourceGroupPrd) \
                          --revision $(containerApp)--$(Build.BuildId) \
                          --query "properties.healthState" -o tsv)
                        [ "$STATUS" = "Healthy" ] && exit 0
                        sleep 10
                      done
                      exit 1

                - task: AzureCLI@2
                  displayName: "Step 3 — Canary: shift 10% traffic to new revision"
                  inputs:
                    azureSubscription: sc-azure-production
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      az containerapp ingress traffic set \
                        --name $(containerAppPrd) \
                        --resource-group $(resourceGroupPrd) \
                        --revision-weight \
                          $(containerApp)--$(Build.BuildId)=10 \
                          latest=90

                - script: make smoke-test ENVIRONMENT=production
                  displayName: "Step 4 — Smoke tests against canary"

                - task: AzureCLI@2
                  displayName: "Step 5 — Monitor error rate for 5 minutes (auto-rollback on breach)"
                  inputs:
                    azureSubscription: sc-azure-production
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      END=$((SECONDS+300))
                      while [ $SECONDS -lt $END ]; do
                        ERROR_RATE=$(az monitor metrics list \
                          --resource $(containerAppId) \
                          --metric "Requests" \
                          --filter "StatusCodeClass eq '5xx'" \
                          --aggregation Total \
                          --query "value[0].timeseries[0].data[-1].total" \
                          -o tsv)
                        echo "5xx count: ${ERROR_RATE:-0}"
                        if [ "${ERROR_RATE:-0}" -gt "50" ]; then
                          echo "Error threshold exceeded — rolling back to previous revision"
                          az containerapp ingress traffic set \
                            --name $(containerAppPrd) \
                            --resource-group $(resourceGroupPrd) \
                            --revision-weight latest=100
                          exit 1
                        fi
                        sleep 30
                      done

                - task: AzureCLI@2
                  displayName: "Step 6 — Full cutover: 100% traffic to new revision"
                  inputs:
                    azureSubscription: sc-azure-production
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      az containerapp ingress traffic set \
                        --name $(containerAppPrd) \
                        --resource-group $(resourceGroupPrd) \
                        --revision-weight $(containerApp)--$(Build.BuildId)=100

                - script: |
                    echo "##vso[build.addbuildtag]production-release"
                    echo "##vso[build.addbuildtag]$(Build.SourceBranchName)"
                  displayName: "Step 7 — Tag release in ADO"
```

### Infrastructure Pipeline — `pipelines/infra.yml`

```yaml
trigger:
  branches:
    include: [main]
  paths:
    include: [infra/**]

pr:
  paths:
    include: [infra/**]

pool:
  vmImage: ubuntu-latest

variables:
  - group: vg-staging
  - name: TF_DIR
    value: infra/
  - name: TF_VERSION
    value: "1.7.5"

stages:

  - stage: Plan
    displayName: "① Terraform Plan (PR only)"
    condition: eq(variables['Build.Reason'], 'PullRequest')
    jobs:
      - job: TerraformPlan
        steps:

          - task: TerraformInstaller@1
            displayName: "Step 1 — Install Terraform $(TF_VERSION)"
            inputs:
              terraformVersion: $(TF_VERSION)

          - task: TerraformTaskV4@4
            displayName: "Step 2 — Terraform Init"
            inputs:
              provider: azurerm
              command: init
              workingDirectory: $(TF_DIR)
              backendServiceArm: sc-azure-staging
              backendAzureRmResourceGroupName: rg-tfstate
              backendAzureRmStorageAccountName: stgtfstate
              backendAzureRmContainerName: tfstate
              backendAzureRmKey: staging.terraform.tfstate

          - task: TerraformTaskV4@4
            displayName: "Step 3 — Terraform Validate"
            inputs:
              provider: azurerm
              command: validate
              workingDirectory: $(TF_DIR)

          - task: TerraformTaskV4@4
            displayName: "Step 4 — Terraform Plan"
            inputs:
              provider: azurerm
              command: plan
              workingDirectory: $(TF_DIR)
              environmentServiceNameAzureRM: sc-azure-staging
              commandOptions: -out=tfplan -detailed-exitcode

          - script: |
              terraform show -no-color tfplan > $(System.DefaultWorkingDirectory)/plan.txt
              echo "##vso[task.uploadsummary]$(System.DefaultWorkingDirectory)/plan.txt"
            displayName: "Step 5 — Post plan to PR summary"
            workingDirectory: $(TF_DIR)

  - stage: Apply
    displayName: "② Terraform Apply (merge only)"
    condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
    jobs:
      - deployment: TerraformApply
        environment: staging
        strategy:
          runOnce:
            deploy:
              steps:

                - task: TerraformInstaller@1
                  displayName: "Step 1 — Install Terraform $(TF_VERSION)"
                  inputs:
                    terraformVersion: $(TF_VERSION)

                - task: TerraformTaskV4@4
                  displayName: "Step 2 — Terraform Init"
                  inputs:
                    provider: azurerm
                    command: init
                    workingDirectory: $(TF_DIR)
                    backendServiceArm: sc-azure-staging
                    backendAzureRmResourceGroupName: rg-tfstate
                    backendAzureRmStorageAccountName: stgtfstate
                    backendAzureRmContainerName: tfstate
                    backendAzureRmKey: staging.terraform.tfstate

                - task: TerraformTaskV4@4
                  displayName: "Step 3 — Terraform Apply"
                  inputs:
                    provider: azurerm
                    command: apply
                    workingDirectory: $(TF_DIR)
                    environmentServiceNameAzureRM: sc-azure-staging
                    commandOptions: -auto-approve
```

---

## 7. Observability

### Step 1 — Central Log Analytics Workspace

All Azure services send diagnostics to a single workspace. No siloed, per-service logs.

```hcl
resource "azurerm_log_analytics_workspace" "main" {
  name                = "law-${var.env}"
  location            = var.location
  resource_group_name = var.resource_group_name
  sku                 = "PerGB2018"
  retention_in_days   = var.env == "production" ? 90 : 30
}
```

### Step 2 — Application Insights

```hcl
resource "azurerm_application_insights" "main" {
  name                = "appi-${var.env}"
  location            = var.location
  resource_group_name = var.resource_group_name
  workspace_id        = azurerm_log_analytics_workspace.main.id
  application_type    = "web"
}
```

Apps connect via `APPLICATIONINSIGHTS_CONNECTION_STRING` (pulled from Key Vault at runtime). All requests, dependencies, exceptions, and custom events are captured with full distributed tracing (W3C TraceContext).

### Step 3 — Three Pillars

| Pillar | Tool | What it covers |
|---|---|---|
| **Logs** | Log Analytics Workspace | Structured JSON from all services; query with KQL |
| **Metrics** | Azure Monitor + App Insights | Request rate, latency p50/p95/p99, error rate, queue depth, replica count |
| **Traces** | Application Insights | End-to-end distributed traces across API → Service Bus → worker |

### Step 4 — Dashboards

```
Azure Workbook: "Platform Operations"
├── API Health       — request rate, latency p50/p95/p99, error rate, APIM throttles
├── Infrastructure   — replica counts, queue depth, Event Hub lag, Cosmos RU usage
├── Video Pipeline   — jobs triggered/completed/failed, avg transcode time, CDN cache hit ratio
└── Notifications    — SMS/Email sent/failed, DLQ depth
```

### Step 5 — Alerts

| Alert | Condition | Severity | Action |
|---|---|---|---|
| API error rate high | 5xx > 1% for 5 min | P2 | PagerDuty |
| API latency degraded | p99 > 2s for 5 min | P3 | Teams |
| Event Hub lag | Consumer lag > 10,000 | P2 | PagerDuty |
| Dead-letter queue | DLQ depth > 0 | P2 | PagerDuty |
| Video job failures | Failure rate > 5% | P3 | Teams |
| Container App at max replicas | replicas = max | P3 | Teams |
| Cosmos RU throttling | HTTP 429 rate > 1% | P2 | PagerDuty |
| Device mass disconnect | >10% devices offline in 5 min | P2 | PagerDuty |

---

## 8. Security

| Layer | Control | Implementation |
|---|---|---|
| **Edge** | WAF, DDoS, rate limiting | Cloudflare managed + custom rules per endpoint |
| **Network** | No public IPs on PaaS | All services behind Private Endpoints; APIM internal mode |
| **Identity** | Zero-trust, no passwords | Managed Identities service-to-service; Entra ID for humans |
| **Secrets** | Central vault | Azure Key Vault — secrets never in code, repos, or ADO |
| **API** | AuthN + AuthZ at gateway | APIM: JWT validation + subscription keys + IP allowlist |
| **Containers** | Minimal attack surface | Non-root user; read-only filesystem; no SSH; Trivy in CI |
| **Data at rest** | Encrypted | Azure platform keys (CMK optional for PII fields) |
| **Data in transit** | TLS everywhere | TLS 1.2+ enforced; HSTS via Cloudflare |
| **IoT Devices** | Per-device identity | X.509 certs via IoT Hub DPS; auto-rotation on enrollment group |
| **Compliance** | Policy as code | Azure Policy: require tags, deny public IPs, enforce diagnostics |
| **Supply chain** | Dependency scanning | Trivy in CI; Renovate Bot for automated dep update PRs |

---

## 9. Scalability

| Component | Scale trigger | Ceiling |
|---|---|---|
| Backend APIs | HTTP concurrent requests > 50 | 30 replicas |
| Log processor | Event Hub consumer lag | 20 replicas |
| Video jobs | Blob creation events | Unlimited parallel jobs |
| Notification worker | Service Bus queue depth > 100 | 10 replicas |
| Event Hubs | Throughput units (auto-inflate) | 20 TUs = 20 MB/s ingest |
| Service Bus | Messaging units (manual scale) | 16 MUs |
| Cosmos DB | Autoscale RU/s | 400 → 40,000 RU/s (configurable higher) |
| IoT Hub | Tier upgrade | S2 = 6M messages/day; S3 = 300M |
| Cloudflare CDN | Automatic, globally distributed | No practical limit |

---

## 10. Reliability & Resiliency

### Availability Targets

| Service | Azure SLA | Composed target |
|---|---|---|
| Container Apps (zone-redundant) | 99.95% | 99.9% |
| API Management | 99.99% | 99.9% |
| Cosmos DB (multi-region) | 99.999% | 99.99% |
| Service Bus Premium | 99.9% | 99.9% |

### Design Patterns

**Retry with exponential backoff** — all Service Bus consumers retry 3 times (1s → 2s → 4s) before dead-lettering.

**Dead-letter queues** — every queue has a DLQ; depth > 0 triggers a P2 alert; messages are inspected and replayed.

**Circuit breaker** — APIM backend circuit breaker opens after 5 consecutive errors and half-opens after 30s. Client SDKs mirror this with Polly / equivalent.

**Health probes** — every Container App exposes `/health/live` and `/health/ready`; traffic is only routed to ready instances.

**Zone redundancy** — Container App Environment, Service Bus Premium, and Event Hubs span all 3 availability zones in UK South.

**Geo-redundancy (day 2)**

```
Primary:   UK South
Secondary: West Europe

Cosmos DB:    multi-region writes, automatic failover
Service Bus:  geo-disaster-recovery namespace pairing
Event Hubs:   geo-disaster-recovery namespace pairing
Blob Storage: GRS (geo-redundant replication)
IoT Hub:      manual failover (~10 min RTO, built-in)
```

**RPO / RTO**

```
RPO: < 1 minute   (Cosmos DB continuous backup)
RTO: < 15 minutes (Container Apps redeploy from ACR)
```

---

## 11. Automation

| What | How |
|---|---|
| All infrastructure | Terraform — Azure Policy denies non-IaC resources |
| Infrastructure changes | PR → plan in ADO → review → merge → auto-apply |
| App deployments | ADO pipelines — zero manual steps from merge to live |
| Secret rotation | Key Vault auto-rotation; ADO Variable Groups pull latest at runtime |
| TLS certificates | Cloudflare ACME (Let's Encrypt) auto-renewal; IoT certs via DPS |
| Dependency updates | Renovate Bot — weekly PRs for images, Terraform providers, packages |
| Dev environment cost | Azure Automation runbook — shuts down dev resources at 20:00 UTC, restarts at 07:00 |
| Cost alerts | Azure Cost Management — budget alerts at 80% and 100% per subscription |

---

## 12. Assumptions

| Assumption | Trade-off |
|---|---|
| In-venue devices support MQTT or HTTPS | MQTT preferred for constrained hardware; raw AMQP is faster but needs heavier client libraries |
| Video moments are short highlights (< 5 min), not live | Blob + FFmpeg + CDN is far cheaper than Azure Live Events; revisit if live is needed |
| Peak concurrent guests per venue: ~50,000 | Container Apps handles this comfortably; AKS gives more control at significantly higher ops cost |
| Multi-region is day-2 | Single primary (UK South) simplifies day-1; Cosmos/SB geo-DR are pre-configured for fast promotion |
| Config web app is staff-only (not public) | Entra ID is sufficient; Azure AD B2C not needed |
| SMS/Email are bursty, not sustained high-volume | ACS managed service avoids running own SMTP/SMPP; Twilio abstraction layer ready as fallback |
| IaC tool: Terraform | Bicep is simpler and Azure-native, but Terraform handles Cloudflare resources natively in the same plan |

---

## 13. Key Architecture Decisions

**1. Container Apps over AKS** — lower ops burden, KEDA built-in, managed control plane. Revisit if advanced Kubernetes features (service mesh, custom schedulers) are needed.

**2. Event Hubs over raw Kafka** — native Azure integration, cheaper at moderate scale. Kafka-compatible protocol means a future migration to Confluent/MSK is straightforward.

**3. IoT Hub over plain Service Bus for device communication** — built-in per-device X.509 registry, Device Provisioning Service, C2D messaging, and routing rules justify the cost at scale.

**4. Cloudflare WAF + CDN over Azure Front Door alone** — better global PoP coverage, superior WAF, Workers for edge logic, unified DNS. Azure CDN sits between Cloudflare and Blob as an origin shield.

**5. Azure Communication Services over Twilio/SendGrid** — single Azure-native vendor, simpler billing. Twilio/SendGrid abstraction layer retained in the notification worker so the provider is swappable via config.

**6. Workload Identity Federation for ADO** — no client secrets to create, store, or rotate. ADO authenticates to Azure with short-lived OIDC tokens, eliminating the biggest source of credential leakage in CI/CD systems.
