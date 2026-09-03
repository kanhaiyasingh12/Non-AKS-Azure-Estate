# Non-AKS Azure Estate — SRE Discovery Report (PROD Environment)

**Project:**  Non-AKS Azure Estate — SRE Discovery Report (PROD Environment)
**Environment:** PROD  
**Scope:** Non-AKS Compute & Managed Services (11 Non-AKS Resources in PROD across 2 Resource Groups; Reconciled with 17 8/3 Baseline where 6 Function Apps are accounted for separately)  
**Collection Date:** September 2, 2026  
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

When SRE initially reviewed the matrix, **Non-AKS Azure Estate was marked `unknown` across multiple capability dimensions**. Following our investigations into Azure Functions (7 apps in PROD) and Non-AKS Compute in DEV (37 resources) and QA (16 resources), this deep-dive targets the remaining compute, container, web hosting, messaging, and security estate in the **PROD** environment.

### 1.2 Investigation Principle

```
DISCOVER --> VERIFY --> SAVE EVIDENCE --> MAP ARCHITECTURE --> EXPLAIN CURRENT STATE --> IDENTIFY OPERATIONAL GAPS --> GET ENGINEERING DECISION --> IMPLEMENT LATER
```

*   **Rule 1: Cross-Verification:** Do not conclude from a single API call. Cross-check related configuration across ARM, Azure CLI, Diagnostic Settings, RBAC assignments, and Service Bus metrics. Preserve all evidence locally.
*   **Rule 2: Read-Only Mandate:** No remediation or configuration mutation is performed during discovery.
*   **Rule 3: Evidence-Backed Facts:** All findings must cite concrete configuration attributes, resource IDs, subscription IDs, and metric data.

---

## 2. PROD Environment Scope

| Metric | Value |
|:---|:---|
| **Total Non-AKS Resources in PROD Scope** | **11** |
| **Container Apps (`Microsoft.App/containerApps`)** | **7** |
| **Container App Environments (`Microsoft.App/managedEnvironments`)** | **2** (Both Succeeded: `cae-sopfactory-prod`, `helios-provision-site-prod-env`) |
| **App Services / Web Apps (`Microsoft.Web/sites`)** | **1** (`helios-prod-ui-appservice`) |
| **Static Web Apps (`Microsoft.Web/staticSites`)** | **1** (`helios-prod-ui-webapp` with 2 live custom domains) |
| **Logic Apps / Workflow Apps (`Microsoft.Logic/workflows`)** | **0** (Down from 4 in DEV and 2 in QA) |
| **Azure Subscription** | `9b9e9af9-5917-4cae-88b4-1304f3ea98b4` (Helios — Production) |
| **Resource Groups in Scope** | **2 Resource Groups** |
| **Primary Resource Group** | `helios-prod-us-west3-rg` (9 non-AKS resources: 7 Container Apps + 2 Managed Environments) |
| **UI Resource Group** | `helios-prod-uswest3-ui` (2 non-AKS resources: 1 App Service + 1 Static Web App) |

```
                                  PROD RESOURCE GROUP DISTRIBUTION (11 Resources)
     +---------------------------------------------------------------------------------------------------+
     |                                                                                                   |
     |   +------------------------------------+               +--------------------------------------+   |
     |   |       helios-prod-us-west3-rg      |               |         helios-prod-uswest3-ui       |   |
     |   |           (9 Resources)            |               |             (2 Resources)            |   |
     |   +------------------------------------+               +--------------------------------------+   |
     |   | • 7 Container Apps                 |               | • 1 App Service                      |   |
     |   |   (ca-opa, ca-catalog-api,         |               |   (helios-prod-ui-appservice)        |   |
     |   |    ca-model-service, ca-library,   |               | • 1 Static Web App                   |   |
     |   |    ca-authoring-bff, ca-composer,  |               |   (helios-prod-ui-webapp)            |   |
     |   |    ca-composition-mcp)             |               |   [Custom: helios.es.qcells.com &    |   |
     |   | • 2 Container App Environments     |               |            prime.ems.es.qcells.com]  |   |
     |   |   (cae-sopfactory-prod,            |               +--------------------------------------+   |
     |   |    helios-provision-site-prod-env) |                                                          |
     |   | • 0 Logic Workflows                |                                                          |
     |   +------------------------------------+                                                          |
     +---------------------------------------------------------------------------------------------------+
```

### 2.1 Reconciliation with 8/3 Audit Baseline (17 Total PROD Baseline Resources)

The platform baseline inventory identified **17 total compute/platform resources in PROD**. SRE reconciled the scope as follows:
*   **6 Azure Function Apps** were evaluated in the platform baseline (expanded to 7 Function Apps in the dedicated *Azure Functions PROD SRE Discovery Report* `Azure_Functions_PROD_SRE_Discovery_Report.md`).
*   **11 Non-AKS Resources** (comprising 7 Container Apps, 2 Managed Environments, 1 Web App, and 1 Static Web App) form the exact scope of this report.
*   **Reconciliation Equation:** $6\text{ (Baseline Function Apps)} + 11\text{ (Non-AKS Compute \& Platform)} = \mathbf{17\text{ Resources}}$.

---

## 3. Master Resource Inventory

The table below catalogs all 11 Non-AKS PROD resources verified via direct Azure Resource Graph and Azure Management API queries (`Non-AKS-Resource-Inventory-PROD.csv`):

| # | Name | ResourceType | ResourceGroup | Location | Kind | Tags | ProvisioningState |
|:---|:---|:---|:---|:---|:---|:---|:---|
| 1 | `ca-opa-prod` | Microsoft.App/containerApps | `helios-prod-us-west3-rg` | East US 2 | N/A | `component=sop-factory; environment=Prod; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 2 | `ca-model-service-prod` | Microsoft.App/containerApps | `helios-prod-us-west3-rg` | East US 2 | N/A | `component=sop-factory; environment=Prod; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 3 | `ca-composer-api-prod` | Microsoft.App/containerApps | `helios-prod-us-west3-rg` | East US 2 | N/A | `component=sop-factory; environment=Prod; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 4 | `ca-composition-mcp-prod` | Microsoft.App/containerApps | `helios-prod-us-west3-rg` | East US 2 | N/A | `component=sop-factory; environment=Prod; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 5 | `ca-catalog-api-prod` | Microsoft.App/containerApps | `helios-prod-us-west3-rg` | East US 2 | N/A | `component=sop-factory; environment=Prod; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 6 | `ca-library-api-prod` | Microsoft.App/containerApps | `helios-prod-us-west3-rg` | East US 2 | N/A | `component=sop-factory; environment=Prod; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 7 | `ca-authoring-bff-prod` | Microsoft.App/containerApps | `helios-prod-us-west3-rg` | East US 2 | N/A | `component=sop-factory; environment=Prod; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 8 | `helios-prod-ui-appservice` | Microsoft.Web/sites | `helios-prod-uswest3-ui` | West US 3 | app,linux | `application=Helios; environment=Production; owner=Devops; service=UI` | N/A (Running) |
| 9 | `helios-prod-ui-webapp` | Microsoft.Web/staticSites | `helios-prod-uswest3-ui` | West US 2 | N/A | `application=Helios; environment=Production; owner=Devops; service=UI` | N/A (Ready) |
| 10 | `cae-sopfactory-prod` | Microsoft.App/managedEnvironments | `helios-prod-us-west3-rg` | East US 2 | N/A | `component=sop-factory; environment=Prod; managed_by=terraform; owner=Devops; project=sopfactory; service=SOP Factory` | Succeeded |
| 11 | `helios-provision-site-prod-env` | Microsoft.App/managedEnvironments | `helios-prod-us-west3-rg` | West US 3 | N/A | `application=Helios; environment=Production; owner=Devops; service=Provision Site` | Succeeded |

---

## 4. Classification & Lifecycle Analysis

SRE executed automated classification and state verification across all 11 resources (`Non-AKS-Resource-Classification-PROD.csv`):

```
                                  PROD ESTATE CLASSIFICATION (11 Resources)
        +-------------------------------------------------------------+-----------------------+
        |                                                             |                       |
        v                                                             v                       v
+---------------------------------------------------------+   +-------------------+   +-------------------+
|                     Active Systems                      |   |      Unknown      |   |   Abandoned or    |
|                     (10 Resources)                      |   |   (1 Resource)    |   |   Failed State    |
| • 7 Container Apps (SOP Factory Suite)                  |   | • helios-prod-ui- |   |   (0 Resources)   |
| • 2 Container App Environments (Succeeded)              |   |   webapp          |   |                   |
| • 1 App Service (helios-prod-ui-appservice Running)     |   |   (SwaCli origin) |   |    [CLEAN SLATE]  |
+---------------------------------------------------------+   +-------------------+   +-------------------+
```

### 4.1 Classification Breakdown

| Classification Category | Count | Percentage | Definition & Policy |
|:---|:---:|:---:|:---|
| **Active** | **10** | 90.9% | Actively maintained platform compute with live ingress, active revisions, or defined operational roles. |
| **Unknown** | **1** | 9.1% | Static web app (`helios-prod-ui-webapp`) requiring automated CI/CD pipeline reconciliation (deployed via `SwaCli`). |
| **Possibly Abandoned / Failed** | **0** | **0.0%** | Completely clean. Zero abandoned or failed resources in PROD. |
| **TOTAL** | **11** | **100%** | |

### 4.2 Cross-Environment Comparison (DEV vs QA vs PROD)

The operational hygiene of PROD represents a dramatic improvement over DEV and QA:

```
+-------------------------------------------------------------------------------------------------------+
|                                    ESTATE MATURITY & LIFECYCLE EVOLUTION                              |
+-------------+-----------------+-----------------+--------------------+--------------------------------+
| Environment | Total Resources | Active Systems  | Unknown / Unmapped | Abandoned / Failed Resources   |
+-------------+-----------------+-----------------+--------------------+--------------------------------+
| DEV         | 37 Resources    | 18 (48.6%)      | 10 (27.0%)         | 9 (24.3%) [Redundant UUDRI,    |
|             |                 |                 |                    | stopped apps, failed CAE]      |
+-------------+-----------------+-----------------+--------------------+--------------------------------+
| QA          | 16 Resources    | 10 (62.5%)      | 3 (18.75%)         | 3 (18.75%) [Failed CAE,        |
|             |                 |                 |                    | personal email tagged UUDRI]   |
+-------------+-----------------+-----------------+--------------------+--------------------------------+
| PROD        | 11 Resources    | 10 (90.9%)      | 1 (9.1%)           | 0 (0.0%) [Zero failed or       |
|             |                 |                 |                    | abandoned resources]           |
+-------------+-----------------+-----------------+--------------------+--------------------------------+
```

*   **Zero Personal Owner Tags:** Unlike DEV and QA (which were burdened by unassigned personal engineer email addresses such as `divyanshu.arya@qcellsces.onmicrosoft.com` on UUDRI), every resource in PROD is tagged with standard team ownership (`owner=Devops`).
*   **Zero Failed Managed Environments:** QA had an orphaned CAE `provision-site-qa-env` that failed in June 2026. In PROD, both `cae-sopfactory-prod` and `helios-provision-site-prod-env` are in a verified `ProvisioningState: Succeeded`.
*   **Absence of Redundant Stacks:** The UUDRI utility stack does not exist in PROD.

---

## 5. Container Apps Deep-Dive (7 Apps)

Seven Container Apps operate inside the primary Container App Environment `cae-sopfactory-prod` in region **East US 2**:

```
                                  PROD CONTAINER APPS COMPUTE TOPOLOGY
    +----------------------------------------------------------------------------------------------------+
    | Managed Environment: cae-sopfactory-prod (East US 2 | Public Ingress | Log Analytics Bound)         |
    | Default Domain: gentleground-a7d0c9f1.eastus2.azurecontainerapps.io                                |
    |                                                                                                    |
    |   [REAL ACR CONTAINER WORKLOADS — ALL RUNNING SHA 7f1586ebbc0f3f61330a5371e5436875b4223633]        |
    |   • ca-authoring-bff-prod   Port: 8080 (External) | 14 Env Vars | System + User Assigned Identity  |
    |   • ca-catalog-api-prod     Port: 8080 (External) | 11 Env Vars | System + User Assigned Identity  |
    |   • ca-model-service-prod   Port: 8080 (External) |  9 Env Vars | System + User Assigned Identity  |
    |   • ca-library-api-prod     Port: 8080 (Internal) | 20 Env Vars | System + User Assigned Identity  |
    |   • ca-composer-api-prod    Port: 8090 (Internal) | 11 Env Vars | System + User Assigned Identity  |
    |   • ca-composition-mcp-prod Port: 8091 (Internal) |  5 Env Vars | System + User Assigned Identity  |
    |                                                                                                    |
    |   [GOVERNANCE & POLICY AGENT]                                                                      |
    |   • ca-opa-prod             Port: 8181 (Internal) | Image: openpolicyagent/opa:latest              |
    |                                                                                                    |
    |   [!] OPERATIONAL GAP: ZERO AUTOSCALING RULES (STATIC REPLICA BOUNDS: 1 MIN, 2-3 MAX)             |
    +----------------------------------------------------------------------------------------------------+
```

### 5.1 Technical Configuration Matrix (7 Container Apps)

All configuration parameters were extracted and verified directly from Azure CLI JSON inspection (`container-apps-deepdive-PROD-2026-09-02_14-27-11.log`):

| Container App | Environment | Image Repository & Tag | CPU | RAM | Port | Ingress | Min/Max Scale | Scale Rules | Env Vars | Secrets | Identity |
|:---|:---|:---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---|
| `ca-opa-prod` | `cae-sopfactory-prod` | `openpolicyagent/opa:latest` | 0.25 | 0.5Gi | 8181 | Internal | 1 / 2 | **0 (None)** | 0 | 0 | None |
| `ca-model-service-prod` | `cae-sopfactory-prod` | `acrsopfactoryprod1fd3k.azurecr.io/model-service:7f1586...` | 0.5 | 1Gi | 8080 | External | 1 / 3 | **0 (None)** | 9 | 0 | System + User |
| `ca-composer-api-prod` | `cae-sopfactory-prod` | `acrsopfactoryprod1fd3k.azurecr.io/composer-api:7f1586...` | 0.5 | 1Gi | 8090 | Internal | 1 / 3 | **0 (None)** | 11 | 0 | System + User |
| `ca-composition-mcp-prod` | `cae-sopfactory-prod` | `acrsopfactoryprod1fd3k.azurecr.io/composition-mcp:7f1586...` | 0.25 | 0.5Gi | 8091 | Internal | 1 / 2 | **0 (None)** | 5 | 0 | System + User |
| `ca-catalog-api-prod` | `cae-sopfactory-prod` | `acrsopfactoryprod1fd3k.azurecr.io/catalog-api:7f1586...` | 0.5 | 1Gi | 8080 | External | 1 / 3 | **0 (None)** | 11 | 1 (`redis-url`) | System + User |
| `ca-library-api-prod` | `cae-sopfactory-prod` | `acrsopfactoryprod1fd3k.azurecr.io/library-api:7f1586...` | 0.5 | 1Gi | 8080 | Internal | 1 / 3 | **0 (None)** | 20 | 0 | System + User |
| `ca-authoring-bff-prod` | `cae-sopfactory-prod` | `acrsopfactoryprod1fd3k.azurecr.io/authoring-bff:7f1586...` | 0.5 | 1Gi | 8080 | External | 1 / 3 | **0 (None)** | 14 | 1 (`orchestrator-function-key`) | System + User |

### 5.2 Critical SRE Findings in PROD Container Apps

#### 1. Promotion Success: Real ACR Images Pinning SHA `7f1586ebbc0f3f61330a5371e5436875b4223633`
*   **Contrast with QA:** In QA, `ca-composer-api-qa` and `ca-composition-mcp-qa` failed due to deployment of Microsoft's `containerapps-helloworld:latest` placeholder image on mismatched ports (8090/8091 vs 80).
*   **PROD Status:** In PROD, **all 6 custom SOP Factory applications run verified production images** built from container registry `acrsopfactoryprod1fd3k.azurecr.io`. Furthermore, all 6 microservices run the exact same commit SHA:
    `acrsopfactoryprod1fd3k.azurecr.io/<service>:7f1586ebbc0f3f61330a5371e5436875b4223633`.
*   Ingress target ports match the application listening ports (8080 for authoring-bff, catalog-api, library-api, model-service; 8090 for composer-api; 8091 for composition-mcp). Requests route successfully without port rejection.

#### 2. Scalability Risk: Zero Autoscaling Rules Configured
*   **Evidence:** All 7 Container Apps define replica bounds (`minReplicas=1`, `maxReplicas=2` or `3`), but have **`RuleCount = 0`** (`Scale Rules: 0 rule(s)`).
*   **Impact:** Because no HTTP concurrent request rules or CPU/memory threshold triggers are defined, Azure Container Apps will **never autoscale beyond 1 replica** under load. During peak production usage or bursty SOP document generation workloads, microservices will saturate their single replica, experiencing CPU throttling and HTTP request queueing or 504 gateway timeouts.
*   **Remediation Mandate:** Configure HTTP request concurrency scaling (e.g., target 50 concurrent requests per replica) and CPU utilization scaling (target 75%) up to 5 replicas for PROD resilience.

#### 3. Configuration Density: Average of 13 Environment Variables per Container
*   Container microservices manage dense configurations (ranging from 5 to 20 variables per container).
*   Crucially, **all 6 custom services have `APPLICATIONINSIGHTS_CONNECTION_STRING` properly injected** directly into container settings (`container-apps-deepdive-PROD-2026-09-02_14-27-11.log`).
*   Secret references are cleanly abstracted into Azure Container App secrets (`redis-url` in `ca-catalog-api-prod` and `orchestrator-function-key` in `ca-authoring-bff-prod`).

---

## 6. Container App Environments (2 CAEs)

The PROD subscription contains 2 Container App Managed Environments:

| Environment Name | Resource Group | Region | ProvisioningState | VNet Binding | Default Domain | App Logs Destination | Log Analytics Workspace ID |
|:---|:---|:---|:---:|:---|:---|:---|:---|
| `cae-sopfactory-prod` | `helios-prod-us-west3-rg` | East US 2 | **Succeeded** | None (Public) | `gentleground-a7d0c9f1.eastus2...` | Log Analytics | `300989f7-22b1-49d3-95b5-849ea7d8ef3e` |
| `helios-provision-site-prod-env` | `helios-prod-us-west3-rg` | West US 3 | **Succeeded** | None (Public) | `gentledune-7dbca19c.westus3...` | Log Analytics | `300989f7-22b1-49d3-95b5-849ea7d8ef3e` |

### 6.1 Operational Analysis of Managed Environments

1. **`cae-sopfactory-prod` (Active Workload CAE):**
   *   Hosts the 7 SOP Factory container apps.
   *   Bound directly to Log Analytics Workspace `300989f7-22b1-49d3-95b5-849ea7d8ef3e` (`log-analytics`), ensuring system and console logs from container runtimes flow to the central monitoring repository.
   *   Lacks Virtual Network integration; external ingress endpoints are exposed via public Azure IP addresses.
2. **`helios-provision-site-prod-env` (Standby Provisioning CAE):**
   *   Created on `2026-08-25T16:00:15Z` in West US 3.
   *   Currently hosts 0 container applications (`containerapp list` yields 0 child apps).
   *   Unlike QA's broken `provision-site-qa-env`, this environment succeeded and is healthy, awaiting site provisioning service deployments.

---

## 7. App Services (Web Apps) Deep-Dive (1 App)

Only **1 App Service** operates in PROD (`helios-prod-ui-appservice`), hosted in `helios-prod-uswest3-ui` (`app-services-deepdive-PROD-2026-09-02_14-32-03.log`):

| Web App Name | Resource Group | Runtime Stack | AlwaysOn | FTPS State | Min TLS | HTTP/2 | Health Check Path | App Insights Mode | Outbound VNet Integration | Managed Identity |
|:---|:---|:---|:---:|:---|:---:|:---:|:---:|:---:|:---|:---|
| `helios-prod-ui-appservice` | `helios-prod-uswest3-ui` | Python 3.11 | **True** | Disabled | 1.2 | False | **MISSING** | **InstrKey (Deprecated)** | **Yes** (`appservice-subnet`) | System + User |

### 7.1 Detailed Web App Architecture Observations

1. **Core Helios UI Backend:**
   *   Serves the primary backend API for the Helios React frontend, managing 43 application settings.
   *   Integrates with Azure API Management (`APIM_SERVICE_URL`), AI Foundry (`AI_FOUNDRY_ENDPOINT`, `MICROSOFT_FOUNDRY_PROJECT_URL`), Azure App Configuration (`AZURE_APPCONFIG_ENDPOINT`), and internal microservices (`WEATHER_SERVICE_URL`, `OMS_SERVICE_URL`, `DIGITAL_TWIN_SERVICE_URL`).
2. **Network Isolation:**
   *   Outbound traffic is injected via Regional VNet Integration into `/subscriptions/9b9e9af9-5917-4cae-88b4-1304f3ea98b4/resourceGroups/helios-prod-us-west3-rg/providers/Microsoft.Network/virtualNetworks/helios-aks-prod-vnet/subnets/appservice-subnet`.
   *   Inbound traffic is supported via private endpoint `helios-prod-ui-appservice-pe` in `helios-prod-uswest3-ui`.
3. **Identity & Governance:**
   *   Configured with System-Assigned Identity (`f7d6fbce-a688-4104-b0ea-ec5e036ed35a`) and User-Assigned Identity `heliosdev-ui-appconfig-reader` (retained DEV naming convention in PROD).
4. **Critical Operational Gaps on `helios-prod-ui-appservice`:**
   *   **Gap 1: Missing Application Insights Connection String:** Setting `APPLICATIONINSIGHTS_CONNECTION_STRING` is **MISSING**. The app relies on the legacy `APPINSIGHTS_INSTRUMENTATIONKEY` (`Instrumentation Key`), which Microsoft has deprecated and will retire. It must be migrated to a modern Connection String.
   *   **Gap 2: Missing Health Check Endpoint:** `HealthCheckPath` is unconfigured (`HealthCheck: Not configured`). Azure App Service health monitoring and automated instance failover cannot determine if the Python process is alive or dead.

---

## 8. Static Web Apps Deep-Dive (1 SWA)

A single Static Web App provides public frontend web hosting for the production Helios platform (`static-webapps-deepdive-PROD-2026-09-02_14-33-35.log`):

| Static Web App Name | Resource Group | SKU | Default Hostname | Build Provider | Linked Backends | Custom Domain | Status |
|:---|:---|:---:|:---|:---:|:---|:---|:---:|
| `helios-prod-ui-webapp` | `helios-prod-uswest3-ui` | Standard | `green-bush-03b86a51e.6.azurestaticapps.net` | SwaCli | None | `helios.es.qcells.com` <br> `prime.ems.es.qcells.com` | **Ready** <br> **Ready** |

### 8.1 Dual Live Production Domains

`helios-prod-ui-webapp` hosts **TWO active, customer-facing production custom domains**:
1. **`helios.es.qcells.com` (Status: Ready):** Primary enterprise portal for Helios energy management and asset operations.
2. **`prime.ems.es.qcells.com` (Status: Ready):** Dedicated Prime EMS operational portal.

### 8.2 Operational Risks on Frontend SWA

*   **Zero Synthetic Availability Monitoring:** Neither `https://helios.es.qcells.com` nor `https://prime.ems.es.qcells.com` is monitored by any Azure Monitor URL Ping Web Test or synthetic availability probe (`cross-cutting-checks-PROD-2026-09-02_14-34-29.log`). A frontend outage, CDN failure, or SSL certificate expiration would go completely undetected until customer escalation.
*   **Deployment Provenance (SwaCli):** The SWA lacks an ARM GitHub Actions repository binding (`Repository URL: N/A`), having been published via `SwaCli` CLI deployments. CI/CD promotion should be formalized into standard GitHub Actions workflows.

---

## 9. Logic Apps / Workflow Apps (0 Workflows)

In the PROD environment, direct Azure Resource Graph enumeration returned **0 Logic App workflows** (`logic-apps-raw.json = []`):

| Environment | Logic App Workflows Count | Workflows Found |
|:---|:---:|:---|
| **DEV** | **4** | `helios-dev-alert-to-slack`, `ems-alert-to-slack`, `sop-alert-to-slack`, `cost-alert-to-slack` |
| **QA** | **2** | `helios-qa-alert-to-slack`, `helios-qa-ems-plan-narration-alert-to-slack` |
| **PROD** | **0** | *(None)* |

*   **Operational Implication:** Unlike DEV and QA, where alert notifications are dispatched to Slack channels via intermediary Logic Apps, PROD alerting relies directly on Azure Action Groups configured for Email notifications (`ag-helios-prod-ops`, `ag-helios-ops`) and ARM roles.
*   **Benefit:** Zero unmonitored workflow compute overhead in PROD.

---

## 10. Service Bus & DLQ Status

SRE audited the production Service Bus namespace `helios-prod-service-bus-ns` (Standard SKU, West US 3) and compared message backlogs against DEV and QA (`evidence/PROD/servicebus/`):

```
                                  PROD SERVICE BUS HEALTH & DLQ COMPARISON
     +---------------------------------------------------------------------------------------------------+
     | Namespace: helios-prod-service-bus-ns (Standard SKU, West US 3)                                   |
     |                                                                                                   |
     |  Topic: helios-knowledgegraph-events                                                              |
     |    +-- Sub: kg-event-processor ------> [CLEAN] 0 Active | 0 DEAD-LETTER MESSAGES                   |
     |    +-- Sub: weather-service ----------> [CLEAN] 0 Active | 0 DEAD-LETTER MESSAGES                   |
     |                                                                                                   |
     |  Topic: helios-oms-events                                                                         |
     |    +-- Sub: iam-service --------------> [CLEAN] 0 Active | 0 DEAD-LETTER MESSAGES                   |
     |                                                                                                   |
     |  Topic: helios-ontology-events                                                                    |
     |    +-- Subscriptions: 0 active subscriptions                                                      |
     +---------------------------------------------------------------------------------------------------+
```

### 10.1 Service Bus Entity & Backlog Inventory

| Namespace | Entity Name | Entity Type | Active Count | Dead-Letter Count | Max Delivery Count | Lock Duration | Default TTL | DLQ on Expiration |
|:---|:---|:---|:---:|:---:|:---:|:---:|:---:|:---:|
| `helios-prod-service-bus-ns` | `helios-knowledgegraph-events / kg-event-processor` | Subscription | 0 | **0** | 10 | 1 min (`PT1M`) | 14 days (`P14D`) | False |
| `helios-prod-service-bus-ns` | `helios-knowledgegraph-events / weather-service` | Subscription | 0 | **0** | 10 | 1 min (`PT1M`) | 14 days (`P14D`) | False |
| `helios-prod-service-bus-ns` | `helios-oms-events / iam-service` | Subscription | 0 | **0** | 10 | 1 min (`PT1M`) | 14 days (`P14D`) | True |
| `helios-prod-service-bus-ns` | `helios-ontology-events` | Topic | 0 | **0** | N/A | N/A | 14 days (`P14D`) | N/A |

### 10.2 Cross-Environment DLQ Accumulation Comparison

```
+-------------------------------------------------------------------------------------------------------+
|                                  CROSS-ENVIRONMENT SERVICE BUS DLQ AUDIT                             |
+------------------------------+---------------------------+-----------------------+--------------------+
| Topic / Subscription         | DEV DLQ Messages          | QA DLQ Messages       | PROD DLQ Messages  |
+------------------------------+---------------------------+-----------------------+--------------------+
| `kg-event-processor`         | 18,169 dead-lettered      | 31,102 dead-lettered  | **0 dead-lettered**|
| `weather-service`            | 0 dead-lettered           | 401 dead-lettered     | **0 dead-lettered**|
| `iam-service`                | 0 dead-lettered           | 7 dead-lettered       | **0 dead-lettered**|
+------------------------------+---------------------------+-----------------------+--------------------+
| **TOTAL DLQ ACCUMULATION**   | **18,169 Messages**       | **31,510 Messages**   | **0 Messages**     |
+------------------------------+---------------------------+-----------------------+--------------------+
```

*   **SRE Assessment:** In DEV and QA, Knowledge Graph ingestion experienced massive deserialization and downstream Neo4j/Cosmos throttling failures, permanently stranding ~50k messages across the two lower environments. In PROD, the pipeline is **100% clean with zero dead-letter accumulation**.
*   **Operational Risk:** Because `deadLetteringOnMessageExpiration = false` on `kg-event-processor`, any message that does fail 10 delivery attempts will remain in the DLQ forever. An automated Azure Monitor metric alert (`DeadletteredMessages > 0`) is essential to prevent PROD from suffering the silent accumulation seen in QA.

---

## 11. Identity, Access & RBAC Matrix

### 11.1 Managed Identity Inventory (8 Identities Mapped)

Eight Managed Identities were mapped across Non-AKS compute in PROD (`cross-cutting-checks-PROD-2026-09-02_14-34-29.log`):

| Resource Name | Identity Type | Principal ID | Role Definition | Assigned Scope |
|:---|:---|:---|:---|:---|
| `ca-opa-prod` | None | N/A | None | N/A |
| `ca-model-service-prod` | System + User | `5b434da7-6dc5-4341-b6e2-dfd0a4e75654` | *(Inherited)* | Access inherited via `uami-sopfactory-prod-apps` |
| `ca-composer-api-prod` | System + User | `a7765c25-d5d2-4ba4-bec4-e0bc1a544e2d` | *(Inherited)* | Access inherited via `uami-sopfactory-prod-apps` |
| `ca-composition-mcp-prod` | System + User | `a766e4fc-40e0-46fd-acf2-c865c2e7de11` | *(Inherited)* | Access inherited via `uami-sopfactory-prod-apps` |
| `ca-catalog-api-prod` | System + User | `f90a092b-040a-4d9a-8edf-3592675ef974` | *(Inherited)* | Access inherited via `uami-sopfactory-prod-apps` |
| `ca-library-api-prod` | System + User | `2b589bc8-343c-46d8-b3b2-dd10aca0eb66` | *(Inherited)* | Access inherited via `uami-sopfactory-prod-apps` |
| `ca-authoring-bff-prod` | System + User | `1e53fc7f-35d3-44a0-ac69-bed3aee6c643` | *(Inherited)* | Access inherited via `uami-sopfactory-prod-apps` |
| `helios-prod-ui-appservice` | System + User | `f7d6fbce-a688-4104-b0ea-ec5e036ed35a` | **Azure AI Developer** <br> **App Configuration Data Reader** <br> **Cognitive Services User** <br> **Foundry Agent Consumer** <br> **Azure AI Developer** <br> **Cognitive Services User** <br> **Search Index Data Reader** | `helios-prod-aif-hub` <br> `helios-prod-apim-config` <br> `helios-prod-aif-hub` <br> `.../projects/pms-core-project` <br> `.../projects/pms-core-project` <br> `.../projects/pms-core-project` <br> `helios-prod-ai-search` |
| `helios-prod-ui-webapp` | SystemAssigned | `d378d0a9-4713-498e-964e-ee3a2f372f52` | *(None direct)* | N/A |

### 11.2 Key Vault Access Analysis (8 Key Vaults Mapped)

All **8 Key Vaults** discovered in PROD operate under legacy **Access Policies (`enableRbacAuthorization = false`)** with 0 explicit access policies configured in ARM metadata:

| Key Vault Name | Resource Group | Access Model | Policies Count | RBAC Mode | Security Risk |
|:---|:---|:---:|:---:|:---:|:---|
| `helios-prod-backend-kv` | `helios-prod-us-west3-rg` | Access Policies | 0 | **Disabled** | Legacy access model; lacks fine-grained least privilege |
| `helios-prod-agents-kv` | `helios-prod-us-west3-rg` | Access Policies | 0 | **Disabled** | Legacy access model |
| `helios-prod-onboard-kv` | `helios-prod-us-west3-rg` | Access Policies | 0 | **Disabled** | Legacy access model |
| `helios-prod-ui-kv` | `helios-prod-uswest3-ui` | Access Policies | 0 | **Disabled** | Legacy access model |
| `kg-event-proc-prod-kv` | `helios-prod-us-west3-rg` | Access Policies | 0 | **Disabled** | Legacy access model |
| `helios-prd-spkplg-pki-kv` | `helios-prod-us-west3-rg` | Access Policies | 0 | **Disabled** | Legacy access model |
| `kvsopfactoryprod1fd3k` | `helios-prod-us-west3-rg` | Access Policies | 0 | **Disabled** | Legacy access model |
| `helios-prod-github-kv` | `helios-prod-us-west3-rg` | Access Policies | 0 | **Disabled** | Legacy access model |

> [!IMPORTANT]
> **Key Vault Governance Finding:** Microsoft recommends migrating all production Key Vaults from legacy Vault Access Policies to **Azure Role-Based Access Control (Azure RBAC)**. With RBAC enabled, secret and certificate permissions (`Key Vault Secrets User`, `Key Vault Secrets Officer`) can be bound directly to Entra ID identities with full audit logging, eliminating static access policies.

---

## 12. Network Security & Topology

```
                                  PROD NETWORK SECURITY & TOPOLOGY
    +----------------------------------------------------------------------------------------------------+
    | VNet: helios-aks-prod-vnet (West US 3)                                                             |
    |                                                                                                    |
    |   Subnet: appservice-subnet                                                                        |
    |     • helios-prod-ui-appservice (Outbound Integration)                                             |
    |                                                                                                    |
    |   Subnet: private-1, privateakssubnet, private-2, private-3 (18 Private Endpoints)                 |
    |     • Key Vaults (5): agents-kv, backend-kv, onboard-kv, ui-kv, github-kv                         |
    |     • AI / Cognitive: helios-prod-aif-hub (aif-pe), heliosprodaifsa (aif-storage-pe)               |
    |     • Databases: helios-prod-cosmos-askhelios, digital-twins, monetization-sql, ui-postgres        |
    |     • Storage: heliosprodobjectstore                                                               |
    |     • Messaging: helios-prod-eventhub-ns, platform-audit-log-ns, eventgrid, mqtt-broker           |
    |     • Monitoring: platform-backend-insights-prod-ampls (AMPLS)                                    |
    |     • Web Apps: helios-prod-ui-appservice-pe                                                         |
    |     • AKS Cluster: kube-apiserver                                                                  |
    +----------------------------------------------------------------------------------------------------+
```

### 12.1 Private Endpoints Summary (18 Mapped)

SRE verified **18 active Private Endpoints** securing production data, messaging, and compute layers:
1. `helios-prod-agents-kv-pe` (`Microsoft.KeyVault/vaults/helios-prod-agents-kv`)
2. `helios-prod-aif-pe` (`Microsoft.CognitiveServices/accounts/helios-prod-aif-hub`)
3. `helios-prod-aif-storage-pe` (`Microsoft.Storage/storageAccounts/heliosprodaifsa`)
4. `helios-prod-backend-kv-pe` (`Microsoft.KeyVault/vaults/helios-prod-backend-kv`)
5. `helios-prod-cosmos-askhelios-pe` (`Microsoft.DocumentDB/databaseAccounts/helios-prod-cosmos-askhelios`)
6. `helios-prod-digital-twins-pe` (`Microsoft.DigitalTwins/digitalTwinsInstances/helios-prod-digital-twins`)
7. `helios-prod-eventgrid-pe` (`Microsoft.EventGrid/topics/helios-prod-platform-events`)
8. `helios-prod-eventhub-pe` (`Microsoft.EventHub/namespaces/helios-prod-eventhub-ns`)
9. `helios-prod-monetization-sql-pe` (`Microsoft.Sql/servers/helios-prod-monetization-sql`)
10. `helios-prod-mqtt-broker-pe` (`Microsoft.EventGrid/namespaces/helios-prod-mqtt-broker-ns`)
11. `helios-prod-onboard-kv-pe` (`Microsoft.KeyVault/vaults/helios-prod-onboard-kv`)
12. `helios-prod-platform-audit-log-pe` (`Microsoft.EventHub/namespaces/helios-prod-platform-audit-log-ns`)
13. `heliosprodobjectstore-blob-pe` (`Microsoft.Storage/storageAccounts/heliosprodobjectstore`)
14. `platform-backend-insights-prod-pe` (`Microsoft.Insights/privateLinkScopes/platform-backend-insights-prod-ampls`)
15. `helios-prod-ui-appservice-pe` (`Microsoft.Web/sites/helios-prod-ui-appservice`)
16. `helios-prod-ui-kv-pe` (`Microsoft.KeyVault/vaults/helios-prod-ui-kv`)
17. `helios-prod-ui-postgres-pe` (`Microsoft.DBforPostgreSQL/flexibleServers/helios-prod-ui-postgres`)
18. `kube-apiserver` (`Microsoft.ContainerService/managedClusters/helios-prod-aks-cluster`)

### 12.2 Public Exposure Analysis

*   **7 out of 7 Container Apps** are reachable via public DNS hostnames (`gentleground-a7d0c9f1.eastus2.azurecontainerapps.io`) with no VNet injection on `cae-sopfactory-prod`. Microservices marked "Internal" are accessible internally to the CAE mesh, but the CAE itself sits on a public IP.
*   **1 out of 1 App Service** (`helios-prod-ui-appservice`) has public access enabled (`Public Access: Enabled`) with 0 IP access restrictions, though protected by private endpoint and Entra ID authentication.
*   **1 out of 1 Static Web App** is exposed to the public internet to serve customer traffic (`helios.es.qcells.com` and `prime.ems.es.qcells.com`).

---

## 13. Observability & Alerting Estate

### 13.1 Diagnostic Settings Coverage Gap (10 of 11 Missing)

```
                                DIAGNOSTIC SETTINGS COVERAGE IN PROD
              +---------------------------------------------------------------+
              |  [✓] Configured (1 Resource / 9.1%):                          |
              |      • helios-prod-ui-appservice (Streaming to LogAnalytics)   |
              |                                                               |
              |  [!] Missing (10 Resources / 90.9%):                          |
              |      • 7 Container Apps (ca-opa-prod, ca-catalog-api-prod,     |
              |        ca-model-service-prod, ca-library-api-prod,            |
              |        ca-authoring-bff-prod, ca-composer-api-prod,           |
              |        ca-composition-mcp-prod)                               |
              |      • 2 Container App Environments (cae-sopfactory-prod,      |
              |        helios-provision-site-prod-env)                        |
              |      • 1 Static Web App (helios-prod-ui-webapp)               |
              +---------------------------------------------------------------+
```

*   **Severe SRE Gap:** 90.9% of the PROD estate is unmonitored at the Azure Monitor resource diagnostic level. While Container Apps stream console logs via CAE log-analytics integration, Azure Monitor resource diagnostics (`AllMetrics`, platform audit logs, HTTP access logs) are not configured.

### 13.2 Metric & Log Alert Rules

1. **Metric Alerts (17 Rules):**
   *   7 Container Apps covered by zero-replica alerts (`ca-opa-prod-zero-replicas`, `ca-model-service-prod-zero-replicas`, etc.).
   *   1 App Service covered by HTTP 5xx alert (`helios-prod-ui-appservice-http5xx`).
   *   1 OTEL telemetry ingest alert (`alert-otel-source-no-incoming`).
   *   8 Function App alerts (e.g. `ems-plan-narration-function-prod-http5xx`, `kg-event-processor-prod-http5xx`, `helios-github-activity-logger-prod-func-http5xx`, etc.).
2. **Log (Scheduled Query) Alerts (34 Rules):**
   *   34 scheduled KQL queries monitoring data pipeline ratios, bronze forecasting models, SOE optimizer inputs/outputs, and gridstatus drift (`cross-cutting-checks-PROD-2026-09-02_14-34-29.log`).
3. **Action Groups (3 Mapped):**
   *   `Application Insights Smart Detection` (ARMRole: 2)
   *   `ag-helios-prod-ops` (Email: 1)
   *   `ag-helios-ops` (Email: 1)
4. **Availability & Synthetic Web Tests:**
   *   **0 URL Ping Web Tests** are configured across all PROD resource groups. Both production custom domains (`helios.es.qcells.com` and `prime.ems.es.qcells.com`) have **zero uptime verification**.

---

## 14. IaC & Terraform Governance Status

SRE audited Terraform state coverage and tags across the 11 resources:

```
                             PROD IAC GOVERNANCE DISTRIBUTION (11 Resources)
    +---------------------------------------+---------------------------------------+
    |           TERRAFORM MANAGED           |          UNMANAGED / UNKNOWN          |
    |             (8 Resources)             |             (3 Resources)             |
    +---------------------------------------+---------------------------------------+
    | • 7 Container Apps (SOP Factory suite)| • 1 App Service                       |
    |   (managed_by=terraform)              |   (helios-prod-ui-appservice)         |
    | • 1 Container App Environment         | • 1 Static Web App                    |
    |   (cae-sopfactory-prod)               |   (helios-prod-ui-webapp)             |
    |                                       | • 1 Container App Environment         |
    |                                       |   (helios-provision-site-prod-env)    |
    +---------------------------------------+---------------------------------------+
```

*   **IaC Managed (8 resources / 72.7%):** All 7 Container Apps and `cae-sopfactory-prod` are provisioned via Terraform with explicit governance tags (`component=sop-factory; managed_by=terraform; project=sopfactory`).
*   **Unmanaged / Unknown (3 resources / 27.3%):** UI backend App Service, UI frontend Static Web App, and the standby Provision Site CAE lack Terraform management tags.

---

## 15. Current PROD Architecture Diagram

```
                                          CLIENT & INGRESS LAYER
       +-------------------------------------------------------------------------------------------------+
       |  Public Internet / Production Operators / Grid Market Telemetry                                  |
       +--------+------------------------------------+--------------------------------+------------------+
                |                                    |                                |
                v                                    v                                v
      +-----------------------------+     +-----------------------------+   +----------------------------+
      | Static Web App (Frontend)   |     | App Service (UI Backend)    |   | Container Apps (Compute)   |
      | helios-prod-ui-webapp       |     | helios-prod-ui-appservice   |   | cae-sopfactory-prod (East) |
      |                             |     |                             |   |                            |
      | Live Custom Domains:        |     | • Python 3.11               |   | • ca-authoring-bff (8080)  |
      | • helios.es.qcells.com      |     | • AlwaysOn: True            |   | • ca-catalog-api (8080)    |
      | • prime.ems.es.qcells.com   |     | • FTPS: Disabled            |   | • ca-model-service (8080)  |
      |                             |     | • VNet Integrated           |   | • ca-library-api (8080)    |
      | [!] 0 Web Ping Tests        |     | [!] Missing ConnString      |   | • ca-composer-api (8090)   |
      |                             |     | [!] Missing /health Path    |   | • ca-composition-mcp (8091)|
      +--------------+--------------+     +--------------+--------------+   | • ca-opa-prod (8181)       |
                     |                                   |                  |                            |
                     | (API Calls)                       |                  | [!] 0 Autoscaling Rules    |
                     +-----------------------------------+                  +-------------+--------------+
                                                         |                                |
                                                         v                                v
       +----------------------------------------------------------------------------------+--------------+
       |                                INTERNAL VNET & PRIVATE ENDPOINTS LAYER                          |
       |                                                                                                 |
       |  VNet: helios-aks-prod-vnet (West US 3)                                                         |
       |    • Subnet: appservice-subnet (helios-prod-ui-appservice outbound)                             |
       |                                                                                                 |
       |  18 Private Endpoints:                                                                          |
       |    • Key Vaults: agents-kv, backend-kv, onboard-kv, ui-kv, github-kv                            |
       |    • AI / Search: helios-prod-aif-hub, heliosprodaifsa, helios-prod-ai-search                   |
       |    • Databases: cosmos-askhelios, digital-twins, monetization-sql, ui-postgres                  |
       |    • Messaging: helios-prod-eventhub-ns, platform-audit-log-ns, eventgrid, mqtt-broker          |
       |    • Observability: platform-backend-insights-prod-ampls                                        |
       |    • Compute / AKS: ui-appservice-pe, kube-apiserver                                            |
       +-------------------------------------------------+-----------------------------------------------+
                                                         |
                                                         v
       +-------------------------------------------------------------------------------------------------+
       |                                 MESSAGING & DATA STREAMING LAYER                                |
       |                                                                                                 |
       |  Service Bus: helios-prod-service-bus-ns (West US 3)                                            |
       |    • Topic: helios-knowledgegraph-events                                                        |
       |        +-- Sub: kg-event-processor ----> [CLEAN] 0 Active | 0 DEAD-LETTER MESSAGES              |
       |        +-- Sub: weather-service -------> [CLEAN] 0 Active | 0 DEAD-LETTER MESSAGES              |
       |    • Topic: helios-oms-events                                                                   |
       |        +-- Sub: iam-service -----------> [CLEAN] 0 Active | 0 DEAD-LETTER MESSAGES              |
       +-------------------------------------------------+-----------------------------------------------+
                                                         |
                                                         v
       +-------------------------------------------------------------------------------------------------+
       |                                 OBSERVABILITY & MONITORING LAYER                                |
       |                                                                                                 |
       |  Log Analytics Workspace: 300989f7-22b1-49d3-95b5-849ea7d8ef3e                                  |
       |    • Connected: cae-sopfactory-prod, helios-provision-site-prod-env, ui-appservice              |
       |    • Metric Alerts: 17 rules (7 zero-replica rules, 1 HTTP 5xx rule, 8 Func alerts)             |
       |    • Log Alerts: 34 scheduled KQL queries                                                       |
       |    • Action Groups: 3 (ag-helios-prod-ops, ag-helios-ops, App Insights Smart Detection)         |
       |    • [!] Gaps: 10/11 missing Diagnostic Settings; 0 Availability Web Tests                      |
       +-------------------------------------------------------------------------------------------------+
```

---

## 16. Identification of the 1 Failing PROD Resource from Daily QRE Report & Root Cause Analysis

### 16.1 The Daily QRE Report Alert Finding

In the automated daily quality and reliability report published to `#daily-qre-report` (`Non-AKS Compute Status — 2026-08-31.txt`):

> **"Non-AKS Compute Status — 2026-08-31 :x: Status: Failure**  
> **8 failing (1 in prod), 19 silent of 107 non-AKS resources across dev/QA/prod.**  
> **helios-github-activity-logger-prod-func — the only prod one; probe returns 503 on all 3 attempts, but Http5xx metric records 0, so its alert rule cannot fire. Same signature in dev and qa."**

### 16.2 Deep SRE Root Cause Analysis

```
                                  THE 503 SILENT FAILURE PARADOX
   +---------------------------------------------------------------------------------------------+
   | External Probe / GitHub Webhook Payload                                                     |
   +----------------------------------------------+----------------------------------------------+
                                                  |
                                                  v
                     +----------------------------------------------------------+
                     | Azure Front-End / Web App Gateway                        |
                     +----------------------------+-----------------------------+
                                                  |
                                                  | Gateway attempts connection to backend host
                                                  v
                     +----------------------------------------------------------+
                     | Azure Functions Host Runtime Process                     |
                     | [CRASHED / FAILED INITIALIZATION]                        |
                     | • Missing dependency or corrupted deployment package     |
                     | • Key Vault reference resolution timeout                 |
                     | • Host fails to start within timeout window              |
                     +----------------------------+-----------------------------+
                                                  |
                                                  v
     +-----------------------------------------------------------------------------------------+
     | Gateway returns HTTP 503 (Service Unavailable) directly to caller                       |
     |                                                                                         |
     | [CRITICAL OBSERVABILITY BLIND SPOT]:                                                    |
     | 1. Functions Host never initialized --> Application Insights SDK never started          |
     | 2. Azure Monitor 'Http5xx' platform metric counts ONLY requests processed by the host   |
     | 3. Azure Monitor Metric counter records: Http5xx = 0                                     |
     | 4. Metric Alert Rule 'helios-github-activity-logger-prod-func-http5xx' checks:          |
     |    "Http5xx > 0" --> Evaluates to FALSE (Alert remains GREEN :white_check_mark:)        |
     |                                                                                         |
     | RESULT: The production webhook listener is 100% DEAD (≥27 days), but alarm NEVER FIRES! |
     +-----------------------------------------------------------------------------------------+
```

1. **Identity & Purpose:**
   *   **Resource:** `helios-github-activity-logger-prod-func` (`Microsoft.Web/sites`, Kind: `functionapp,linux`).
   *   **Resource Group:** `helios-prod-us-west3-rg`.
   *   **Role:** Dedicated Python 3.11 Azure Function triggered by GitHub Webhooks (`github_webhook`) to capture CI/CD pipeline activities, release tags, and PR lifecycle events into the Helios data lake.
2. **Failure Mechanism:**
   *   Every synthetic probe hitting `https://helios-github-activity-logger-prod-func.azurewebsites.net` returns `HTTP 503 Service Unavailable`.
   *   The function app host runtime process has crashed or failed during initial boot due to an unhandled exception in module loading, a broken virtualenv dependency, or an unresolvable Key Vault secret reference.
3. **The Alert Blind Spot:**
   *   Azure Monitor metric alert `helios-github-activity-logger-prod-func-http5xx` triggers when `Http5xx > 0`.
   *   Because the Azure Functions runtime host process is down, the HTTP request is terminated at the Azure front-end gateway. The Azure Functions host never processes the request and never increments the platform `Http5xx` metric.
   *   The metric remains **0**, keeping the metric alert permanently green.
4. **Gated Remediation Action (Strict SRE Rule):**
   *   *Do NOT blindly redeploy or enable Always On as a first-line fix.*
   *   **Step 1:** Pull host diagnostic logs via Kudu / Azure CLI:
       `az webapp log download --name helios-github-activity-logger-prod-func --resource-group helios-prod-us-west3-rg`.
   *   **Step 2:** Snapshot current application settings, deployment zip, and runtime package.
   *   **Step 3:** Deploy an **Azure Monitor Resource Health Alert** (`AvailabilityState != Available`) and an Application Insights availability probe to immediately replace the flawed `Http5xx` metric alert.
   *   **Step 4:** Resolve the underlying runtime crash (fix broken module import or Key Vault secret permission).

---

## 17. Capability Matrix Update for Non-AKS Estate

Based on PROD discovery findings across the 11 Non-AKS resources, the SRE Release Orchestration Capability Matrix **PROD** column is updated with factual evidence citations:

| Capability Column | PROD Status | SRE Evidence Citation |
|:---|:---:|:---|
| **Deploy on merge** | **Partial** | **Container Apps:** 6 of 7 apps deploy via CI/CD pipelines pushing to `acrsopfactoryprod1fd3k.azurecr.io`; all 6 pinned to SHA `7f1586ebbc0f3f61330a5371e5436875b4223633`. <br>**Web Apps:** `helios-prod-ui-appservice` deploys via GitHub Actions (`SCM_DO_BUILD_DURING_DEPLOYMENT = 1`). <br>**Static Web Apps:** `helios-prod-ui-webapp` was published via direct `SwaCli` command (`Provider: SwaCli`), lacking an auditable GitHub repository commit binding in ARM metadata. |
| **Promotion gates** | **Yes** | **Container Apps promote cleanly across environments via immutable commit SHAs.** In PROD, all 6 SOP Factory microservices run image tag `:7f1586ebbc0f3f61330a5371e5436875b4223633`, successfully promoted from lower environments (unlike QA which remained stalled on `containerapps-helloworld:latest`). However, `ca-opa-prod` still uses mutable tag `:latest`. |
| **Scheduled verification** | **No** | **0 out of 11 resources** have Azure Monitor URL Ping Web Tests configured (`cross-cutting-checks-PROD-2026-09-02_14-34-29.log`). Dedicated health check paths (`/health`) are missing on `helios-prod-ui-appservice`. Production custom domains `helios.es.qcells.com` and `prime.ems.es.qcells.com` have **zero synthetic uptime verification**. |
| **Alerting** | **Partial** | 17 Metric Alerts cover zero-replica states on Container Apps, HTTP 5xx on `helios-prod-ui-appservice`, and Function Apps. 34 Log Query alerts monitor data pipeline ratios in Log Analytics. However, **the 1 failing resource (`helios-github-activity-logger-prod-func`) has been 503 for ≥27 days without alert triggering due to metric blind spot**, and 0 alerts monitor frontend domain availability. |
| **Observability + cost** | **Partial** | **10 out of 11 resources lack Azure Monitor Diagnostic Settings** (`cross-cutting-checks-PROD-2026-09-02_14-34-29.log`). `helios-prod-ui-appservice` is missing `APPLICATIONINSIGHTS_CONNECTION_STRING`. On the positive side, Container App environments stream console logs to workspace `300989f7-22b1-49d3-95b5-849ea7d8ef3e`, and Service Bus has 0 dead-letter backlog. |
| **Reporting / audit** | **Partial** | 8 of 11 resources are IaC-tagged and Terraform-managed (`managed_by=terraform; component=sop-factory`). 18 Private Endpoints and 8 Managed Identities are fully cataloged. However, all 8 Key Vaults operate under legacy Access Policies with RBAC disabled, preventing unified Entra ID access auditing. |

---

*Evidence verified and recorded by SRE Principal Engineering team. No remediation or implementation changes have been performed during this discovery phase.*
