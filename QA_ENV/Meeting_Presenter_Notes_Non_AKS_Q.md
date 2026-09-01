# Non-AKS Azure Estate – QA Discovery

---

# 1. Executive Summary

### Opening Statement

> "I have completed the detailed SRE discovery for the Helios QA Non-AKS Estate. The environment is more organized than DEV (16 resources vs. 37).

> I found two critical issues that break QA workflows:

>  two SOP Factory microservices are running placeholder Microsoft "Hello World" images on the mismatch ports (8090/8091 instead of 80), causing all requests to fail. 
> 
> Additionally, a failed Container App Environment has been orphaned since June.

>Service Bus has accumulated over **31,000 dead-lettered messages**, and **15 out of 16 resources are running completely dark** with zero diagnostic settings.

---

# 2. Scope & Inventory Baseline

## 2.1 What I Did

> I validated the current QA inventory against the 8 resources across 3 env DEV, QA, PROD platform baseline.
>
> The 8 resources across 3 env reported 22 Azure compute resources. After validating the QA subscription (`663afada-2155-4c4d-b908-ac771ef2d133`), I confirmed the inventory as **6 Azure Function Apps** and **16 Non-AKS compute resources**, distributed across **3 Resource Groups**.

## 2.2 Resource Breakdown

| Resource Type | Count |
|---|---:|
| Container Apps | 7 |
| Container App Environments | 3 |
| App Services (Web Apps) | 2 |
| Static Web Apps | 2 |
| Logic Apps | 2 |
| **Total Non-AKS Resources** | **16** |

## 2.3 Key Points

> Out of these 16 resources, **10 are active, 3 are unknown, and 3 are specifically flagged** (1 failed Container App Environment and 2 UUDRI personal resources).

---

# 3. Finding #1 – Placeholder Images & Port Mismatch (Root Cause of Slack Alerts)

## 3.1 Resource

`ca-composer-api-qa` & `ca-composition-mcp-qa`

## 3.2 What I Found

> I found when i investigated the root cause behind the continuous P3 failures reported daily in the `#daily-qre-report` Slack alerts.
> 
> Terraform initialized these two SOP Factory microservices with Microsoft's default "Hello World" placeholder image, which listens exclusively on Port 80. However, Azure ingress is routing traffic to ports 8090 and 8091.

### Current State

- Running image: `mcr.microsoft.com/azuredocs/containerapps-helloworld:latest` (Port 80).
- Target ingress ports: 8090 and 8091.
- Synthetic health probes hitting 8090/8091 receive Connection Refused.

## 3.3 Why It Matters

> The port mismatch results in a 100% request failure rate.
>
> Every probe and request results in a 502 Bad Gateway, completely breaking QA workflows for these microservices and continuously triggering the daily Slack alerts.

## 3.4 Recommendation

> We must deploy the correct images to match the ingress configuration.

---

# 4. Finding #2 – Massive Service Bus DLQ Accumulation

## 4.1 What I Reviewed

> Similar to the DEV environment, I reviewed the messaging layer in QA to identify operational bottlenecks.

## 4.2 What I Found

>  I found that the Service Bus has accumulated over **31,510 dead-lettered messages**.

### Current Backlog

- Topic `helios-knowledgegraph-events` / Sub `kg-event-processor`: **31,102 dead-lettered messages**
- Topic `helios-knowledgegraph-events` / Sub `weather-service`: **401 dead-lettered messages**
- Topic `helios-oms-events` / Sub `iam-service`: **7 dead-lettered messages** (Session lock risk)

## 4.3 Why Is This Important?

> This massive accumulation indicates that thousands of business events have failed to process and are stuck indefinitely, hiding new failures and consuming storage quota.

---

# 5. Finding #3 – Orphaned Failed Container App Environment

## 5.1 Resource

`provision-site-qa-env`

## 5.2 What I Found

> The Container App Environment `provision-site-qa-env` has been in a 'Failed' state (due to a Capacity Error) since June 2026.
>
> It was replaced by `helios-provision-site-qa-env` in August, but the dead resource was never deleted.

## 5.3 Why It Matters

> Leaving orphaned, failed infrastructure creates architectural clutter, complicates monitoring, and can incur unnecessary costs or quota consumption.

## 5.4 Recommendation

> The resource should be decommissioned immediately.

---

# 6. Finding #4 – Architectural Drift & Missing Services

## 6.1 What I Found

> When comparing the QA architecture to DEV, there is significant architectural drift.
> 
> DEV contains 8 SOP Factory apps (including the React UI) and 2 Site Artifact Repository (SAR) demo apps. QA is completely missing `ca-sopfactory-ui` and all SAR services.

## 6.2 Why It Matters

> Environments should mirror each other. The absence of the UI and SAR backend microservices in QA means these components cannot be validated in the staging environment before production.

---

# 7. Finding #5 – Observability & Custom Domain Risks

## 7.1 What I Found

> The Static Web App `helios-qa-ui-webapp` serves the live domain `https://helios.qa.es.qcells.com` and is marked as Ready.
>
> However, there are **0 URL Ping Web Tests** configured for this domain, and **15 out of 16 Non-AKS resources lack Diagnostic Settings**.

## 7.2 Why It Matters

> We are running a live custom domain completely blind. Without synthetic monitoring and diagnostic logs, we will not know if the UI goes down or if infrastructure fails unless a user reports it.

---

# 8. Questions & Decisions for Leadership

## Q1. Service Bus DLQ Purge Approval
> Can SRE safely purge the 31,102 dead-lettered messages on `kg-event-processor` once sample schemas are extracted, or does the Knowledge Graph team require offline replay into a dead-letter archive?

## Q2. SOP Factory Promotion Cadence
> What is the target release window to promote the SOP Factory UI container and SAR backend microservices from DEV into QA?

## Q3. UUDRI Ownership & Deprecation
> Can we formally transfer the `uudri-qa-rg` stack from `divyanshu.arya@qcellsces.onmicrosoft.com` to the shared Platform Engineering team and standardize its deployment under Terraform?

## Q4. Custom Domain SLI/SLO Target
> What is the formal availability SLO for `helios.qa.es.qcells.com` (recommended: 99.5% during business hours for QA validation)?

---

