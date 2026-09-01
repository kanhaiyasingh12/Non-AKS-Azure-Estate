# Non-AKS Azure Estate — SRE Discovery Report (QA Environment)

**Document Owner:** Dipak Singh 
**Environment:** QA  
**Scope:** Non-AKS Compute & Managed Services (16 Non-AKS Resources in QA; 3 Resource Groups; Reconciled with 22 8/3 Baseline where 6 Function Apps are accounted for separately)  
**Collection Date:** September 1, 2026  
**Status:** Discovery complete — no remediation or implementation changes have been performed  

---

[[_TOC_]]

---

## 1. Background and Context

### 1.1 Why This Investigation Exists

The SRE team maintains a **Release Orchestration Capability Matrix** that tracks the operational maturity of every platform component across six core capability columns:

| Column | Description |
|:---|:---|
| **Deploy on merge** | Does the component auto-deploy when code is merged to the main trunk? |
| **Promotion gates (dev > QA > prod)** | Is there a controlled, auditable promotion gate between environments? |
| **Scheduled verification** | Are there recurring synthetic checks or health probes validating the running system? |
| **Alerting** | Are actionable metric/log alert rules and verified action groups configured? |
| **Observability + cost** | Is diagnostic telemetry actively streaming and are cost metrics attributable? |
| **Reporting / audit** | Are deployments, configuration drift, and identity bindings auditable? |

The platform matrix covers: k8s + ArgoCD apps (including Temporal), Frontend (React, IBM Carbon components), Foundry models, Azure Functions (~87 across 3 envs), Flyway DB migrations, **Non-AKS Azure estate (~107 compute & platform resources across DEV/QA/PROD)**, and Fabric / Knowledge Graphs.

When SRE initially reviewed the matrix, **Non-AKS Azure Estate was marked `unknown` across multiple capability dimensions**. Following our investigations into Azure Functions (10 apps in QA) and Non-AKS Compute in DEV (37 resources), this deep-dive targets the remaining compute, container, web hosting, and messaging estate in the **QA** environment.

### 1.2 Investigation Principle

```
DISCOVER --> VERIFY --> SAVE EVIDENCE --> MAP ARCHITECTURE --> EXPLAIN CURRENT STATE --> IDENTIFY OPERATIONAL GAPS --> GET ENGINEERING DECISION --> IMPLEMENT LATER
```

*   **Rule 1: Cross-Verification:** Do not conclude from a single API call. Cross-check related configuration across ARM, Azure CLI, Diagnostic Settings, RBAC assignments, and Service Bus metrics. Preserve all evidence locally.
*   **Rule 2: Read-Only Mandate:** No remediation or configuration mutation is performed during discovery.
*   **Rule 3: Evidence-Backed Facts:** All findings must cite concrete configuration attributes, resource IDs, subscription IDs, and metric data.

---

## 2. QA Environment Scope

| Metric | Value |
|:---|:---|
| **Total Non-AKS Resources in QA Scope** | **16** |
| **Container Apps (`Microsoft.App/containerApps`)** | **7** |
| **Container App Environments (`Microsoft.App/managedEnvironments`)** | **3** (2 Succeeded, 1 Failed) |
| **App Services / Web Apps (`Microsoft.Web/sites`)** | **2** |
| **Static Web Apps (`Microsoft.Web/staticSites`)** | **2** |
| **Logic Apps / Workflow Apps (`Microsoft.Logic/workflows`)** | **2** |
| **Azure Subscription** | `663afada-2155-4c4d-b908-ac771ef2d133` (Helios — QA) |
| **Resource Groups in Scope** | **3 Resource Groups** |
| **Primary Resource Group** | `helios-qa-us-west3-rg` (12 non-AKS resources) |
| **UI Resource Group** | `helios-qa-uswest3-ui` (2 non-AKS resources) |
| **UUDRI Resource Group** | `uudri-qa-rg` (2 non-AKS resources) |

```
                                  QA RESOURCE GROUP DISTRIBUTION (16 Resources)
     +---------------------------------------------------------------------------------------------------+
     |                                                                                                   |
     |   +------------------------------------+   +------------------------+   +---------------------+   |
     |   |       helios-qa-us-west3-rg        |   |  helios-qa-uswest3-ui  |   |     uudri-qa-rg     |   |
     |   |           (12 Resources)           |   |     (2 Resources)      |   |    (2 Resources)    |   |
     |   +------------------------------------+   +------------------------+   +---------------------+   |
     |   | • 7 Container Apps                 |   | • 1 App Service        |   | • 1 App Service     |   |
     |   | • 3 Container App Environments     |   |   (helios-qa-ui-app)   |   |   (UUDRI-App-01)    |   |
     |   | • 2 Logic Workflows                |   | • 1 Static Web App     |   | • 1 Static Web App  |   |
     |   |                                    |   |   (helios-qa-ui-web)   |   |   (UUDRI-SWA-01)    |   |
     |   +------------------------------------+   +------------------------+   +---------------------+   |
     +---------------------------------------------------------------------------------------------------+
```

### 2.1 Reconciliation with 8/3 Audit Baseline (22 Total QA Baseline Resources)

The platform baseline inventory identified **22 total compute/platform resources in QA**. SRE reconciled the scope as follows:
*   **6 Azure Function Apps** were evaluated in the platform baseline (expanded to 10 Function Apps in the dedicated *Azure Functions QA SRE Discovery Report* `Azure_Functions_QA_SRE_Discovery_Report.md`).
*   **16 Non-AKS Resources** (comprising 7 Container Apps, 3 Managed Environments, 2 Web Apps, 2 Static Web Apps, and 2 Logic Workflows) form the exact scope of this report.
*   **Reconciliation Equation:** $6\text{ (Baseline Function Apps)} + 16\text{ (Non-AKS Compute \& Managed Services)} = \mathbf{22\text{ Resources}}$.

---

## 3. Master Resource Inventory

The table below catalogs all 16 Non-AKS QA resources verified via direct Azure Resource Graph and Azure Management API queries (`Non-AKS-Resource-Inventory-QA.csv`):

| # | Name | ResourceType | ResourceGroup | Location | Kind | Tags | ProvisioningState |
|:---|:---|:---|:---|:---|:---|:---|:---|
| 1 | `ca-opa-qa` | Microsoft.App/containerApps | `helios-qa-us-west3-rg` | East US 2 | N/A | `component=sop-factory; environment=QA; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 2 | `ca-catalog-api-qa` | Microsoft.App/containerApps | `helios-qa-us-west3-rg` | East US 2 | N/A | `component=sop-factory; environment=QA; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 3 | `ca-model-service-qa` | Microsoft.App/containerApps | `helios-qa-us-west3-rg` | East US 2 | N/A | `component=sop-factory; environment=QA; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 4 | `ca-library-api-qa` | Microsoft.App/containerApps | `helios-qa-us-west3-rg` | East US 2 | N/A | `component=sop-factory; environment=QA; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 5 | `ca-authoring-bff-qa` | Microsoft.App/containerApps | `helios-qa-us-west3-rg` | East US 2 | N/A | `component=sop-factory; environment=QA; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 6 | `ca-composer-api-qa` | Microsoft.App/containerApps | `helios-qa-us-west3-rg` | East US 2 | N/A | `component=sop-factory; environment=QA; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 7 | `ca-composition-mcp-qa` | Microsoft.App/containerApps | `helios-qa-us-west3-rg` | East US 2 | N/A | `component=sop-factory; environment=QA; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 8 | `UUDRI-App-Service-qa-01` | Microsoft.Web/sites | `uudri-qa-rg` | West US 2 | app | `application=UUDRI; businessunit=other; environment=QA; hidden-link: /app-insights-resource-id=...; owner=divyanshu.arya@qcellsces.onmicrosoft.com` | N/A |
| 9 | `helios-qa-ui-appservice` | Microsoft.Web/sites | `helios-qa-uswest3-ui` | West US 3 | app,linux | `application=Helios; environment=QA; hidden-link: /app-insights-resource-id=...; owner=Devops; service=UI` | N/A |
| 10 | `UUDRI-Static-Web-App-qa-01` | Microsoft.Web/staticSites | `uudri-qa-rg` | West US 2 | N/A | `application=UUDRI; businessunit=other; environment=QA; owner=divyanshu.arya@qcellsces.onmicrosoft.com` | N/A |
| 11 | `helios-qa-ui-webapp` | Microsoft.Web/staticSites | `helios-qa-uswest3-ui` | West US 2 | N/A | `application=Helios; environment=QA; owner=Devops; service=UI` | N/A |
| 12 | `helios-qa-alert-to-slack` | Microsoft.Logic/workflows | `helios-qa-us-west3-rg` | westus3 | N/A | `application=Platform Services; environment=QA; owner=Devops; service=Monitoring` | Succeeded |
| 13 | `helios-qa-ems-plan-narration-alert-to-slack` | Microsoft.Logic/workflows | `helios-qa-us-west3-rg` | westus3 | N/A | `application=Platform Services; environment=QA; owner=Devops; service=Monitoring` | Succeeded |
| 14 | `cae-sopfactory-qa` | Microsoft.App/managedEnvironments | `helios-qa-us-west3-rg` | East US 2 | N/A | `component=sop-factory; environment=QA; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 15 | `helios-provision-site-qa-env` | Microsoft.App/managedEnvironments | `helios-qa-us-west3-rg` | West US 3 | N/A | `application=Helios; environment=QA; owner=Devops; service=Provision Site` | Succeeded |
| 16 | `provision-site-qa-env` | Microsoft.App/managedEnvironments | `helios-qa-us-west3-rg` | West US 3 | N/A | `application=Helios; environment=QA; owner=Devops; service=Provision Site` | **Failed** |

---

## 4. Classification & Lifecycle Analysis

SRE executed automated classification and state verification across all 16 resources (`Non-AKS-Resource-Classification-QA.csv`):

```
                                  QA ESTATE CLASSIFICATION (16 Resources)
       +------------------------------------+------------------------------------+
       |                                    |                                    |
       v                                    v                                    v
+------------------------+        +------------------------+        +------------------------+
|     Active Systems     |        |        Unknown         |        |   Possibly Abandoned   |
|     (10 Resources)     |        |     (3 Resources)      |        |     (3 Resources)      |
|                        |        |                        |        |     & Failed State     |
+------------------------+        +------------------------+        +------------------------+
```

### 4.1 Classification Breakdown

| Classification Category | Count | Percentage | Definition & Policy |
|:---|:---:|:---:|:---|
| **Active** | **10** | 62.5% | Actively maintained platform components with live ingress, active revisions, or defined operational roles. |
| **Unknown** | **3** | 18.75% | Static web apps and workflow logic apps requiring validation of upstream invocation or CI/CD provenance. |
| **Possibly Abandoned / Failed** | **3** | 18.75% | Resources in broken `Failed` state or tagged with unassigned personal individual email addresses. |
| **TOTAL** | **16** | **100%** | |

### 4.2 Detailed Breakdown of Flagged Resources (3 Flagged Items)

```
+---------------------------------------------------------------------------------------------------------------------+
|                                         SRE QA FLAGGED ITEMS RECONCILIATION                                         |
+----+--------------------------------+--------------------+----------------------------------------------------------+
| #  | Resource Name                  | Classification     | Primary Flag Reason                                      |
+----+--------------------------------+--------------------+----------------------------------------------------------+
| 1  | provision-site-qa-env          | Possibly Abandoned | ProvisioningState is 'Failed' (Capacity Heavy Usage)    |
| 2  | UUDRI-App-Service-qa-01        | Possibly Abandoned | UUDRI resource — Individual personal email owner tag     |
| 3  | UUDRI-Static-Web-App-qa-01     | Possibly Abandoned | UUDRI resource — Individual personal email owner tag     |
+----+--------------------------------+--------------------+----------------------------------------------------------+
```

1. **`provision-site-qa-env` (Container App Environment — Failed State):**
   *   **Resource ID:** `/subscriptions/663afada-2155-4c4d-b908-ac771ef2d133/resourceGroups/helios-qa-us-west3-rg/providers/Microsoft.App/managedEnvironments/provision-site-qa-env`
   *   **Failure Cause:** Created on `2026-06-24` by `Sachin.MN@qcellsces.onmicrosoft.com`. Failed with `ErrorCode: ManagedEnvironmentCapacityHeavyUsageError` ("Creating a new managed environment is unavailable at this time due to high demand in current region").
   *   **Orphan Status:** On `2026-08-24`, a replacement environment named `helios-provision-site-qa-env` was successfully provisioned in the same resource group. The original failed resource was never deleted and remains orphaned in ARM state.
2. **`UUDRI-App-Service-qa-01` (App Service — Personal Owner Tag):**
   *   **Resource ID:** `/subscriptions/663afada-2155-4c4d-b908-ac771ef2d133/resourceGroups/uudri-qa-rg/providers/Microsoft.Web/sites/UUDRI-App-Service-qa-01`
   *   **Flag Reason:** Tagged with `owner=divyanshu.arya@qcellsces.onmicrosoft.com` and `businessunit=other`. Has no shared platform team assignment, no VNet integration, and no Terraform tracking.
3. **`UUDRI-Static-Web-App-qa-01` (Static Web App — Personal Owner Tag):**
   *   **Resource ID:** `/subscriptions/663afada-2155-4c4d-b908-ac771ef2d133/resourceGroups/uudri-qa-rg/providers/Microsoft.Web/staticSites/UUDRI-Static-Web-App-qa-01`
   *   **Flag Reason:** Static frontend linked to branch `v2.3.2` of repository `https://github.com/qcells-hqct/UUDRI-React`, tagged to individual email `divyanshu.arya@qcellsces.onmicrosoft.com`.

---

## 5. Container Apps Deep-Dive (7 Apps)

Seven Container Apps operate inside the single active Container App Environment `cae-sopfactory-qa` in region **East US 2**:

```
                                  QA CONTAINER APPS COMPUTE TOPOLOGY
    +------------------------------------------------------------------------------------------------+
    | Managed Environment: cae-sopfactory-qa (East US 2 | Static IP: 57.166.255.18 | Public Ingress)  |
    |                                                                                                |
    |   [ACTIVE REAL WORKLOADS]                                                                      |
    |   • ca-authoring-bff-qa   Port: 8080 (External) | Image: acrsopfactoryqao80ns.azurecr.io/...   |
    |   • ca-catalog-api-qa     Port: 8080 (External) | Image: acrsopfactoryqao80ns.azurecr.io/...   |
    |   • ca-model-service-qa   Port: 8080 (External) | Image: acrsopfactoryqao80ns.azurecr.io/...   |
    |   • ca-library-api-qa     Port: 8080 (Internal) | Image: acrsopfactoryqao80ns.azurecr.io/...   |
    |   • ca-opa-qa             Port: 8181 (Internal) | Image: openpolicyagent/opa:latest            |
    |                                                                                                |
    |   [CRITICAL BLOCKERS — PLACEHOLDER IMAGES & PORT MISMATCH]                                    |
    |   • ca-composer-api-qa    Port: 8090 (Internal) | Image: .../containerapps-helloworld:latest    |
    |     --> [!] Envoy proxies to targetPort 8090, but Helloworld image listens ONLY on Port 80!   |
    |   • ca-composition-mcp-qa Port: 8091 (Internal) | Image: .../containerapps-helloworld:latest    |
    |     --> [!] Envoy proxies to targetPort 8091, but Helloworld image listens ONLY on Port 80!   |
    +------------------------------------------------------------------------------------------------+
```

### 5.1 Technical Configuration Matrix (7 Container Apps)

| Container App | Environment | Image Repository & Tag | CPU | RAM | Port | Ingress | Min/Max Scale | Probes Configured | Identity |
|:---|:---|:---|:---:|:---:|:---:|:---:|:---:|:---:|:---|
| `ca-opa-qa` | `cae-sopfactory-qa` | `openpolicyagent/opa:latest` | 0.25 | 0.5Gi | 8181 | Internal | 1 / 2 | None | None |
| `ca-catalog-api-qa` | `cae-sopfactory-qa` | `acrsopfactoryqao80ns.azurecr.io/catalog-api:03b20389...` | 0.5 | 1Gi | 8080 | External | 1 / 3 | None | System + User |
| `ca-model-service-qa` | `cae-sopfactory-qa` | `acrsopfactoryqao80ns.azurecr.io/model-service:03b20389...` | 0.5 | 1Gi | 8080 | External | 1 / 3 | None | System + User |
| `ca-library-api-qa` | `cae-sopfactory-qa` | `acrsopfactoryqao80ns.azurecr.io/library-api:03b20389...` | 0.5 | 1Gi | 8080 | Internal | 1 / 3 | None | System + User |
| `ca-authoring-bff-qa` | `cae-sopfactory-qa` | `acrsopfactoryqao80ns.azurecr.io/authoring-bff:03b20389...` | 0.5 | 1Gi | 8080 | External | 1 / 3 | None | System + User |
| `ca-composer-api-qa` | `cae-sopfactory-qa` | `mcr.microsoft.com/azuredocs/containerapps-helloworld:latest` | 0.5 | 1Gi | **8090** | Internal | 1 / 3 | None | System + User |
| `ca-composition-mcp-qa` | `cae-sopfactory-qa` | `mcr.microsoft.com/azuredocs/containerapps-helloworld:latest` | 0.25 | 0.5Gi | **8091** | Internal | 1 / 2 | None | System + User |

### 5.2 Critical SRE Findings in QA Container Apps

#### 1. Critical Blocker: Placeholder Images & Port Mismatch on Composer & MCP
*   **Evidence:** `ca-composer-api-qa` and `ca-composition-mcp-qa` both run the Microsoft quickstart image `mcr.microsoft.com/azuredocs/containerapps-helloworld:latest`.
*   **Failure Mechanism:**
    *   The ingress definitions specify `targetPort = 8090` (for composer) and `targetPort = 8091` (for MCP).
    *   The `containerapps-helloworld` image is a simple NGINX/Node webserver hardcoded to listen on **port 80**.
    *   Consequently, Azure Container Apps Envoy internal ingress proxy sends traffic to ports 8090/8091 inside the container, where no daemon is listening. All requests fail with immediate `HTTP 502 Bad Gateway` or connection timeouts.
    *   This completely breaks SOP Factory document composition and MCP tool calling workflows in the QA environment.

#### 2. Architectural Drift vs DEV Environment (Missing SOP Factory UI & SAR Apps)
*   **DEV Baseline:** In DEV, SOP Factory contains 8 container apps including the dedicated React frontend `ca-sopfactory-ui-dev` (Port 8090) and Site Artifact Repository demo apps (`ca-sar-api-demo-dev`, `ca-sar-graph-demo-dev` in `cae-sar-demo-dev`).
*   **QA Status:** In QA, `ca-sopfactory-ui-qa` and the SAR demo apps **do not exist**. SOP Factory UI in QA is hosted via alternative static distribution, and SAR backend services have not been promoted to QA.

---

## 6. Container App Environments (3 CAEs)

The QA subscription contains 3 Container App Managed Environments:

| Environment Name | Resource Group | Region | ProvisioningState | VNet Binding | Default Domain | App Logs Destination | Log Analytics Workspace ID |
|:---|:---|:---|:---:|:---|:---|:---|:---|
| `cae-sopfactory-qa` | `helios-qa-us-west3-rg` | East US 2 | **Succeeded** | None (Public) | `gentlebeach-ed727b85.eastus2...` | Log Analytics | `a3046cd6-1875-4d27-9d3c-68a4939ca665` |
| `helios-provision-site-qa-env` | `helios-qa-us-west3-rg` | West US 3 | **Succeeded** | None (Public) | `graysea-2e1b72a4.westus3...` | Log Analytics | `a3915040-71be-4d74-8a0b-cbd9729ebfb4` |
| `provision-site-qa-env` | `helios-qa-us-west3-rg` | West US 3 | **Failed** | None (Public) | `wonderfulgrass-35df3ee5.westus3...` | Log Analytics | `a3915040-71be-4d74-8a0b-cbd9729ebfb4` |

> [!WARNING]
> **Failed Resource Alert:** `provision-site-qa-env` is in a permanent `Failed` state due to `ManagedEnvironmentCapacityHeavyUsageError`. It contains 0 container apps and should be deleted immediately to prevent confusion with `helios-provision-site-qa-env`.

---

## 7. App Services (Web Apps) Deep-Dive (2 Apps)

Two Web Apps operate in QA across two resource groups (`app-services-deepdive-QA-2026-09-01_13-00-23.log`):

| Web App Name | Resource Group | Runtime Stack | AlwaysOn | FTPS State | Min TLS | Health Check Path | App Insights | Outbound VNet Integration | Managed Identity |
|:---|:---|:---|:---:|:---:|:---:|:---:|:---:|:---:|:---|
| `helios-qa-ui-appservice` | `helios-qa-uswest3-ui` | Python 3.11 | **True** | Disabled | 1.2 | *(none)* | **Configured** | **Yes** (`appservice-subnet`) | System + User |
| `UUDRI-App-Service-qa-01` | `uudri-qa-rg` | .NET 10.0 | **True** | Disabled | 1.2 | *(none)* | **Configured** | **No** | None |

### 7.1 Detailed Web App Architecture Observations

1. **`helios-qa-ui-appservice` (Core Helios UI Backend):**
   *   Runs Python 3.11 with 48 application settings configuring connections to APIM (`APIM_SERVICE_URL`), AI Foundry (`MICROSOFT_FOUNDRY_PROJECT_URL`), Azure AI Search (`AZURE_SEARCH_SERVICE_URL`), and backend microservices (`WEATHER_SERVICE_URL`, `MONETIZATION_SERVICE_URL`, `OMS_SERVICE_URL`).
   *   Integrates with VNet subnet `helios-aks-qa-vnet/subnets/appservice-subnet`.
   *   Configured with System-Assigned Identity and User-Assigned Identity `heliosdev-ui-appconfig-reader` (reused naming convention in QA).
   *   Has private endpoint `helios-qa-ui-appservice-pe` in `helios-qa-uswest3-ui`.
2. **`UUDRI-App-Service-qa-01` (UUDRI .NET API):**
   *   Runs .NET 10.0 isolated worker process with 34 application settings.
   *   Connects to Azure Communication Services (`ACS_BASE_ENDPOINT`), Blob storage (`BLOB_STOTAGE_URL`), and PostgreSQL database (`POSTGRES_CONNECTION_STRING`).
   *   Lacks VNet integration and has no IP access restriction rules configured.

---

## 8. Static Web Apps Deep-Dive (2 SWAs)

Two Static Web Apps provide frontend web hosting in QA:

| # | Static Web App Name | Resource Group | SKU | Repository URL | Branch | Build Provider | Default Hostname | Custom Domain | Custom Domain Status |
|:---:|:---|:---|:---:|:---|:---|:---:|:---|:---|:---:|
| 1 | `helios-qa-ui-webapp` | `helios-qa-uswest3-ui` | Standard | *(none)* | *(none)* | SwaCli | `icy-pond-0b8bfa31e.2.azurestaticapps.net` | `helios.qa.es.qcells.com` | **Ready** |
| 2 | `UUDRI-Static-Web-App-qa-01` | `uudri-qa-rg` | Free | `https://github.com/qcells-hqct/UUDRI-React` | `v2.3.2` | Custom | `green-bush-0e087ff1e.6.azurestaticapps.net` | *(none)* | N/A |

### 8.1 SWA Configuration & Domain Audit

1. **Custom Domain on `helios-qa-ui-webapp`:**
   *   Configured with verified custom domain `helios.qa.es.qcells.com` (Status: `Ready`, Created: `2026-07-10T17:46:53Z`).
   *   Provides the primary QA web interface for the Helios platform.
   *   Deployed via `SwaCli` directly without a linked GitHub Actions repository binding in ARM metadata.
2. **Tagged Release Tracking on `UUDRI-Static-Web-App-qa-01`:**
   *   Tracks Git tag `v2.3.2` on `https://github.com/qcells-hqct/UUDRI-React`.
   *   Operates on the `Free` SKU tier with `stagingEnvironmentPolicy: Enabled`.

---

## 9. Logic Apps / Workflow Apps (2 Workflows)

Two active Azure Logic App workflows handle alert routing and notification distribution in QA (`logic-apps-raw.json`):

| App Name | Resource Group | Location | State | ProvisioningState | Created Time | Tags / Purpose |
|:---|:---|:---|:---:|:---:|:---|:---|
| `helios-qa-alert-to-slack` | `helios-qa-us-west3-rg` | `westus3` | Enabled | **Succeeded** | `2026-08-14T18:10:31Z` | `application=Platform Services; service=Monitoring` |
| `helios-qa-ems-plan-narration-alert-to-slack` | `helios-qa-us-west3-rg` | `westus3` | Enabled | **Succeeded** | `2026-08-14T20:28:12Z` | `application=Platform Services; service=Monitoring` |

*   **Integration Mechanisms:** Both Logic Apps act as webhook receivers triggered by Azure Monitor Action Groups (`ag-helios-qa-ops` and `ag-helios-qa-ems-plan-narration`). They format incoming JSON telemetry and post rich incident cards to dedicated Slack channels.
*   **Operational Gap:** Neither workflow has Diagnostic Settings enabled to stream run histories, trigger failures, or action execution latencies to Log Analytics.

---

## 10. Service Bus & Dead-Letter Queue (DLQ) Deep-Dive

SRE audited all Service Bus entities in the QA namespace `helios-qa-service-bus-ns` (`servicebus/`):

```
                                QA SERVICE BUS & DLQ ACCUMULATION
    +------------------------------------------------------------------------------------------------+
    | Namespace: helios-qa-service-bus-ns (Standard SKU, West US 3)                                  |
    |                                                                                                |
    |  Topic: helios-knowledgegraph-events                                                           |
    |    +-- Sub: kg-event-processor ------> [CRITICAL] 31,102 DEAD-LETTERED MESSAGES (0 Active)      |
    |    |                                    Max Delivery Count: 10 | Lock Duration: 1m             |
    |    +-- Sub: weather-service ----------> [WARNING] 401 DEAD-LETTERED MESSAGES (0 Active)        |
    |                                         Max Delivery Count: 10 | Lock Duration: 1m             |
    |                                                                                                |
    |  Topic: helios-oms-events                                                                      |
    |    +-- Sub: iam-service --------------> [WARNING] 7 DEAD-LETTERED MESSAGES (0 Active)          |
    |                                         Requires Session: True | Max Delivery: 10              |
    |                                                                                                |
    |  Topic: helios-ontology-events                                                                 |
    |    +-- Subscriptions: 0 active subscriptions                                                   |
    +------------------------------------------------------------------------------------------------+
```

### 10.1 Service Bus Backlog Metric Inventory

| Namespace | Entity Name | Entity Type | Active Count | Dead-Letter Count | Max Delivery Count | Lock Duration | Default TTL | DLQ on Expiration |
|:---|:---|:---|:---:|:---:|:---:|:---:|:---:|:---:|
| `helios-qa-service-bus-ns` | `helios-knowledgegraph-events / kg-event-processor` | Subscription | 0 | **31,102** | 10 | 1 min (`PT1M`) | 14 days (`P14D`) | False |
| `helios-qa-service-bus-ns` | `helios-knowledgegraph-events / weather-service` | Subscription | 0 | **401** | 10 | 1 min (`PT1M`) | 14 days (`P14D`) | False |
| `helios-qa-service-bus-ns` | `helios-oms-events / iam-service` | Subscription | 0 | **7** | 10 | 1 min (`PT1M`) | 14 days (`P14D`) | True |
| `helios-qa-service-bus-ns` | `helios-ontology-events` | Topic | 0 | 0 | N/A | N/A | 14 days (`P14D`) | N/A |

### 10.2 Root Cause Analysis of the 31,510 Dead-Lettered Messages in QA

1. **Massive Backlog on `kg-event-processor` (31,102 Messages):**
   *   The `kg-event-processor` subscription in QA accumulated **31,102 dead-lettered events** since its creation on `2026-05-19`.
   *   **Failure Trigger:** Knowledge Graph ingestion events generated during QA bulk test runs encountered deserialization errors or downstream Neo4j/Cosmos graph API rate limiting in `kg-event-processor-qa` Function App. After 10 failed delivery attempts (`maxDeliveryCount = 10`), messages were routed to DLQ.
   *   Because `DeadLetteringOnMessageExpiration = false`, failed messages are preserved forever until explicitly drained or purged.
2. **Backlog on `weather-service` (401 Messages):**
   *   Accumulated 401 failed weather telemetry sync events due to intermittent third-party weather API timeout exceptions.
3. **Backlog on `iam-service` (7 Messages):**
   *   Session-enabled subscription (`requiresSession = true`) holding 7 poison messages causing session lock stalls.

---

## 11. UUDRI QA Estate Discovery

Unlike DEV (which contained a redundant dual-stack configuration across two resource groups), the QA environment operates a single, consolidated UUDRI stack in `uudri-qa-rg`:

```
                                    UUDRI QA STACK ARCHITECTURE
     +------------------------------------------------------------------------------------------+
     | Resource Group: uudri-qa-rg (West US 2)                                                  |
     |                                                                                          |
     |   Frontend:               UUDRI-Static-Web-App-qa-01 (Release v2.3.2)                   |
     |                           Host: green-bush-0e087ff1e.6.azurestaticapps.net               |
     |                                          |                                               |
     |                                          v                                               |
     |   Backend API:            UUDRI-App-Service-qa-01 (.NET 10.0 Isolated)                  |
     |                           Host: uudri-app-service-qa-01.azurewebsites.net                |
     |                                          |                                               |
     |                                          v                                               |
     |   Supporting Services:    • Blob Storage: uudristorageaccountqa01                        |
     |                           • Key Vault: UUDRI-Key-Vault-qa-02                             |
     |                           • App Insights: UUDRI-App-Insights-qa-01                       |
     |                           • Function App: UUDRI-bill-processor-qa-01                     |
     +------------------------------------------------------------------------------------------+
```

### 11.1 Governance & Ownership Risk

*   **Individual Ownership Tag:** Both `UUDRI-App-Service-qa-01` and `UUDRI-Static-Web-App-qa-01` are tagged with `owner=divyanshu.arya@qcellsces.onmicrosoft.com`.
*   **Remediation Mandate:** Assign official platform team ownership (`owner=platform-team`, `team=uudri-engineering`), bring the stack under centralized Terraform IaC control, and configure VNet integration.

---

## 12. Identity, Access & RBAC Matrix

### 12.1 Managed Identity Inventory (8 Identities Mapped)

Eight Managed Identities were mapped across Non-AKS compute in QA (`rbac-summary-QA.csv`):

| Resource Name | Identity Type | Principal ID | Role Definition | Assigned Scope |
|:---|:---|:---|:---|:---|
| `ca-catalog-api-qa` | System + User | `3670ff5c-2350-4ecc-a283-ecebe17f9e12` | **AcrPull** | `acrsopfactoryqao80ns` registry |
| `ca-model-service-qa` | System + User | `3b89affd-f5cb-4190-9530-777c29adc463` | *(None direct)* | Inherited via `uami-sopfactory-qa-apps` |
| `ca-library-api-qa` | System + User | `4dd490c5-369a-4dae-a738-a9b093087b5b` | **AcrPull** | `acrsopfactoryqao80ns` registry |
| `ca-authoring-bff-qa` | System + User | `f40c9bf8-d9d0-4751-9e46-4985d658b528` | *(None direct)* | Inherited via `uami-sopfactory-qa-apps` |
| `ca-composer-api-qa` | System + User | `2040ac1d-576d-4cc0-a0b7-6d412f7e3cd3` | *(None direct)* | Inherited via `uami-sopfactory-qa-apps` |
| `ca-composition-mcp-qa` | System + User | `ff61a66e-4366-431a-add8-f93fd7a76d03` | *(None direct)* | Inherited via `uami-sopfactory-qa-apps` |
| `helios-qa-ui-appservice` | System + User | `38026cd0-1acc-4ca6-801d-79936f4d2e92` | **App Configuration Data Reader** <br> **Azure AI Developer** <br> **Azure AI Developer** <br> **Cognitive Services User** <br> **Cognitive Services User** <br> **Foundry Agent Consumer** <br> **Search Index Data Reader** | `helios-qa-apim-config` <br> `helios-qa-aif-hub` <br> `.../projects/pms-core-project` <br> `helios-qa-aif-hub` <br> `.../projects/pms-core-project` <br> `.../projects/pms-core-project` <br> `helios-qa-ai-search` |
| `helios-qa-ui-webapp` | SystemAssigned | `5c3e1def-2a4d-4551-bdfe-a17333649a9a` | *(None direct)* | N/A |

### 12.2 Key Vault Access Analysis (8 Key Vaults Mapped)

All **8 Key Vaults** discovered in QA operate under legacy **Access Policies (`enableRbacAuthorization = false`)** with 0 explicit access policies configured in ARM metadata (`keyvault-access-QA.csv`):

| Key Vault Name | Resource Group | Access Model | Policies Count | RBAC Mode |
|:---|:---|:---:|:---:|:---:|
| `UUDRI-Key-Vault-qa-02` | `uudri-qa-rg` | Access Policies | 0 | Disabled |
| `helios-qa-backend-kv` | `helios-qa-us-west3-rg` | Access Policies | 0 | Disabled |
| `helios-qa-agents-kv` | `helios-qa-us-west3-rg` | Access Policies | 0 | Disabled |
| `helios-qa-onboarding-kv` | `helios-qa-us-west3-rg` | Access Policies | 0 | Disabled |
| `helios-qa-ui-kv` | `helios-qa-uswest3-ui` | Access Policies | 0 | Disabled |
| `kg-event-processor-qa-kv` | `helios-qa-us-west3-rg` | Access Policies | 0 | Disabled |
| `helios-qa-spkplug-pki-kv` | `helios-qa-us-west3-rg` | Access Policies | 0 | Disabled |
| `kvsopfactoryqao80ns` | `helios-qa-us-west3-rg` | Access Policies | 0 | Disabled |

---

## 13. Network Security & Topology

```
                                  QA NETWORK SECURITY & TOPOLOGY
    +------------------------------------------------------------------------------------------+
    | VNet: helios-aks-qa-vnet (West US 3)                                                     |
    |                                                                                          |
    |   Subnet: appservice-subnet                                                              |
    |     • helios-qa-ui-appservice (Outbound Integration)                                     |
    |                                                                                          |
    |   Subnet: private-1 (7 Private Endpoints)                                                |
    |     • Key Vaults: agents-kv, backend-kv, onboarding-kv                                   |
    |     • AI / Storage: helios-qa-aif-hub, heliosqaaifsa, heliosqaobjectstore                |
    |     • Databases: helios-qa-cosmos-askhelios, helios-qa-digital-twins                     |
    |     • Web Apps: helios-qa-ui-appservice-pe                                               |
    |                                                                                          |
    |   Subnet: privateakssubnet (5 Private Endpoints)                                         |
    |     • Messaging: helios-qa-eventhub-ns, platform-audit-log-ns, eventgrid-pe, mqtt-broker  |
    |     • AKS: kube-apiserver                                                                |
    |                                                                                          |
    |   Subnet: private-2 & private-3 (2 Private Endpoints)                                    |
    |     • Monitoring: platform-backend-insights-qa-ampls (private-2)                         |
    |     • UI Key Vault: helios-qa-ui-kv (private-3)                                          |
    +------------------------------------------------------------------------------------------+
```

### 13.1 Private Endpoints Summary (16 Mapped)

SRE verified **16 active Private Endpoints** securing data and backend layers in QA (`private-endpoints-QA.csv`):
*   **Key Vaults (4):** `helios-qa-agents-kv-pe`, `helios-qa-backend-kv-pe`, `helios-qa-onboarding-kv-pe`, `helios-qa-ui-kv-pe`.
*   **AI & Cognitive Services (1):** `helios-qa-aif-pe` (AI Foundry Hub).
*   **Storage Accounts (2):** `helios-qa-aif-storage-pe` (`heliosqaaifsa`), `heliosqaobjectstore-blob-pe` (`heliosqaobjectstore`).
*   **Databases & Digital Twins (2):** `helios-qa-cosmos-askhelios-pe`, `helios-qa-digital-twins-pe`.
*   **Messaging & Eventing (4):** `helios-qa-eventhub-pe`, `helios-qa-platform-audit-log-pe`, `helios-qa-eventgrid-pe`, `helios-qa-mqtt-broker-pe`.
*   **Monitoring & Ingress (3):** `platform-backend-insights-qa-pe`, `helios-qa-ui-appservice-pe`, `kube-apiserver`.

### 13.2 Public Exposure Analysis

*   **7 out of 7 Container Apps** are reachable via public DNS hostnames (`gentlebeach-ed727b85.eastus2.azurecontainerapps.io`) with no VNet injection on `cae-sopfactory-qa`.
*   **1 out of 2 App Services** (`UUDRI-App-Service-qa-01`) is publicly accessible with no VNet binding and 0 IP restrictions.
*   **2 out of 2 Static Web Apps** are exposed to the public internet (`helios.qa.es.qcells.com` and `green-bush-0e087ff1e.6.azurestaticapps.net`).

---

## 14. Observability & Alerting Estate

### 14.1 Diagnostic Settings Coverage Gap (15 of 16 Missing)

```
                                DIAGNOSTIC SETTINGS COVERAGE IN QA
              +---------------------------------------------------------------+
              |  [✓] Configured (1 Resource / 6.25%):                         |
              |      • helios-qa-ui-appservice (Streaming to Log Analytics)   |
              |                                                               |
              |  [!] Missing (15 Resources / 93.75%):                         |
              |      • 7 Container Apps (ca-opa-qa, ca-catalog-api, etc.)     |
              |      • 3 Container App Environments (cae-sopfactory-qa, etc.) |
              |      • 1 Web App (UUDRI-App-Service-qa-01)                    |
              |      • 2 Static Web Apps (helios-qa-ui-webapp, UUDRI-SWA)     |
              |      • 2 Logic Workflows (alert-to-slack, ems-alert-to-slack) |
              +---------------------------------------------------------------+
```

### 14.2 Metric & Log Alert Rules

1. **Metric Alerts (19 Rules):**
   *   7 Container Apps covered by zero-replica alerts (`ca-opa-qa-zero-replicas`, `ca-catalog-api-qa-zero-replicas`, etc.).
   *   1 App Service covered by HTTP 5xx alert (`helios-qa-ui-appservice-http5xx`).
   *   1 Service Bus Dead-Letter alert (`helios-qa-knowledgegraph-events-deadletters`).
   *   10 Function App HTTP 5xx alerts.
2. **Log (Scheduled Query) Alerts (44 Rules):**
   *   44 KQL query alerts monitoring OTEL pipeline alarms, Bronze ratio ingestion errors, market demand forecast drift, and weather data ingest errors (`scheduled-query-alerts-all.json`).
3. **Action Groups (4 Mapped):**
   *   `Application Insights Smart Detection` (ARMRole: 2)
   *   `ag-helios-qa-ops` (Email: 1, LogicApp: 1)
   *   `ag-helios-ops` (Email: 1)
   *   `ag-helios-qa-ems-plan-narration` (LogicApp: 1)
4. **Availability & Web Tests:**
   *   **0 URL Ping Web Tests** are configured across all QA resource groups (`availability-tests-QA`). Neither `helios.qa.es.qcells.com` nor any backend API has active synthetic uptime verification.

---

## 15. IaC & Terraform Governance Status

SRE audited Terraform state coverage and tags (`iac-coverage-QA.csv`):

```
                             QA IAC GOVERNANCE DISTRIBUTION (16 Resources)
    +---------------------------------------+---------------------------------------+
    |           TERRAFORM MANAGED           |          UNMANAGED / UNKNOWN          |
    |             (8 Resources)             |             (8 Resources)             |
    +---------------------------------------+---------------------------------------+
    | • 7 Container Apps (SOP Factory suite)| • 1 App Service (helios-qa-ui-appserv)|
    | • 1 Container App Environment         | • 1 App Service (UUDRI-App-01)        |
    |   (cae-sopfactory-qa)                 | • 2 Static Web Apps (Helios UI, UUDRI)|
    |                                       | • 2 Logic Apps (Slack alert workflows)|
    |                                       | • 2 Container App Environments        |
    |                                       |   (helios-provision-site, failed CAE) |
    +---------------------------------------+---------------------------------------+
```

*   **IaC Managed (8):** The SOP Factory Container Apps and `cae-sopfactory-qa` are deployed via Terraform modules (`component=sop-factory; managed_by=terraform`).
*   **Unmanaged / Unknown (8):** UI web apps, UUDRI stack, Logic Apps, and Provision Site environments were created via portal/CLI or custom scripts without `managed_by=terraform` tags.

---

## 16. Current QA Architecture Diagram

```
                                         CLIENT & INGRESS LAYER
       +-----------------------------------------------------------------------------------------+
       |  Public Internet / QA Testers / Slack Webhooks                                          |
       +--------+-----------------------+--------------------+--------------------+--------------+
                |                       |                    |                    |
                v                       v                    v                    v
      +-------------------+   +-------------------+   +-------------+   +------------------------+
      | Static Web Apps   |   | Web Apps          |   | Logic Apps  |   | Container Apps         |
      | (2 SWAs)          |   | (2 App Services)  |   | (2 Apps)    |   | (7 Apps in East US 2)  |
      |                   |   |                   |   |             |   |                        |
      | • helios-qa-ui-   |   | • helios-qa-ui-   |   | • alert-to- |   | • ca-catalog-api       |
      |   webapp (Custom: |   |   appservice      |   |   slack     |   | • ca-model-service     |
      |   helios.qa.es.   |   |   (Python 3.11,   |   | • ems-plan- |   | • ca-library-api       |
      |   qcells.com)     |   |   VNet Connected) |   |   narration-|   | • ca-authoring-bff     |
      | • UUDRI-Static-   |   | • UUDRI-App-01    |   |   alert     |   | • ca-opa-qa            |
      |   Web-App-qa-01   |   |   (.NET 10.0,     |   +-------------+   | ---------------------- |
      |   (v2.3.2)        |   |   Public Only)    |                     | [!] ca-composer-api-qa |
      +---------+---------+   +---------+---------+                     |     (Port 8090 HELLO)  |
                |                       |                               | [!] ca-composition-mcp |
                +-----------------------+-------------------------------+     (Port 8091 HELLO)  |
                                        |                               +-----------+------------+
                                        v                                           |
       +----------------------------------------------------------------------------+------------+
       |                          INTERNAL SERVICE MESH & VNET LAYER                             |
       |                                                                                         |
       |  VNet: helios-aks-qa-vnet (West US 3)                                                   |
       |    • Subnet: appservice-subnet (helios-qa-ui-appservice outbound)                       |
       |                                                                                         |
       |  Private Endpoints (16 PEs in West US 3):                                               |
       |    • Key Vaults: backend-kv, agents-kv, onboarding-kv, ui-kv                            |
       |    • AI Services: helios-qa-aif-hub, helios-qa-ai-search, pms-core-project              |
       |    • Storage: heliosqaobjectstore, heliosqaaifsa                                        |
       |    • Databases: helios-qa-cosmos-askhelios, digital-twins                               |
       |                                                                                         |
       |  [!] CAE Orphan State:                                                                  |
       |    • provision-site-qa-env --> PROVISIONINGSTATE: FAILED (Capacity Error)               |
       +------------------------------------+----------------------------------------------------+
                                            |
                                            v
       +-----------------------------------------------------------------------------------------+
       |                          MESSAGING & EVENT QUEUES LAYER                                 |
       |                                                                                         |
       |  Service Bus: helios-qa-service-bus-ns (West US 3)                                      |
       |    • Topic: helios-knowledgegraph-events                                                |
       |        +-- Sub: kg-event-processor ----> [CRITICAL] 31,102 DEAD-LETTERED MESSAGES       |
       |        +-- Sub: weather-service -------> [WARNING] 401 DEAD-LETTERED MESSAGES           |
       |    • Topic: helios-oms-events                                                           |
       |        +-- Sub: iam-service -----------> [WARNING] 7 DEAD-LETTERED MESSAGES             |
       +------------------------------------+----------------------------------------------------+
                                            |
                                            v
       +-----------------------------------------------------------------------------------------+
       |                          OBSERVABILITY & MONITORING LAYER                               |
       |                                                                                         |
       |  Application Insights:                  Azure Monitor:                                  |
       |    • Configured on both Web Apps          • Diagnostic Settings: 1/16 configured        |
       |    • Connected to UI & UUDRI              • Metric Alerts: 19 rules                     |
       |                                           • Log Alerts: 44 rules                        |
       |                                           • Web Ping Tests: 0 configured                |
       +-----------------------------------------------------------------------------------------+
```

---

## 17. Capability Matrix Update for Non-AKS Estate

Based on QA findings across the 16 Non-AKS resources, the SRE Release Orchestration Capability Matrix QA column is updated with factual evidence citations:

| Capability Column | QA Status | SRE Evidence Citation |
|:---|:---:|:---|
| **Deploy on merge** | **Partial** | **Container Apps:** 5/7 apps deploy via CI/CD; `ca-composer-api-qa` and `ca-composition-mcp-qa` are stalled on quickstart placeholder images (`container-apps-deepdive-QA-2026-09-01_12-52-27.log`). <br>**Web Apps:** `helios-qa-ui-appservice` deploys via GitHub Actions; `UUDRI-App-Service-qa-01` has no recorded deployer metadata. <br>**Static Web Apps:** `helios-qa-ui-webapp` uses manual/CLI `SwaCli` push; `UUDRI-Static-Web-App-qa-01` tracks release tag `v2.3.2`. |
| **Promotion gates** | **Partial** | Container Apps in SOP Factory promote via SHA digests (`acrsopfactoryqao80ns.azurecr.io/catalog-api:03b20389...`). However, Composer and MCP were never promoted, remaining on `containerapps-helloworld:latest`. OPA runs mutable tag `:latest`. |
| **Scheduled verification** | **No** | **0 out of 16 resources** have Azure Monitor URL Ping Web Tests configured (`cross-cutting-checks-QA-2026-09-01_13-02-33.log`). Web App health check paths are unconfigured on both apps. Custom domain `helios.qa.es.qcells.com` has zero synthetic health validation. |
| **Alerting** | **Partial** | 19 Metric Alerts cover zero-replica states on Container Apps and HTTP 5xx on `helios-qa-ui-appservice`. 44 Log Query alerts active in Log Analytics. However, **8 resources have zero alert rules** (including UUDRI and Logic Apps), and no alerts monitor SWA latency. |
| **Observability + cost** | **Partial** | **15 out of 16 resources lack Azure Monitor Diagnostic Settings** (`diagnostic-settings-QA.csv`). 31,102 dead-lettered messages on Service Bus indicate unmonitored processing failures in Knowledge Graph pipelines. |
| **Reporting / audit** | **Partial** | 8 of 16 resources are IaC-tagged and Terraform-managed (`iac-coverage-QA.csv`). 8 resources are unmanaged or lack ownership metadata. All 8 Managed Identities and 16 Private Endpoints are fully mapped in audit logs. |

---
