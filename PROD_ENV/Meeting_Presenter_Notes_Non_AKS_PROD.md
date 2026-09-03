# Non-AKS Azure Estate – PROD Discovery
## Meeting Presenter Notes

# If senior told 2 minute breif summary

> Hi Team. I have completed our SRE discovery. There are 11 Non-AKS resources in PROD.

> our deployments are clean and our message queues are perfectly healthy.

> However, We are currently blind spot in production due to three critical gaps:

> 1 **The Silent Failure:** The GitHub Activity Logger has been crashed for with in a month, but our alert dashboard shows it as green because the metric alert is flawed. It is completely down, but no alarm is going off.

> 2 **Zero Synthetic Monitoring:** We have no automated ping tests on our live customer-facing domains. If a portal goes down, we won’t know until a customer complains.

> 3 **No Autoscaling:** Our containers are locked to static capacity. If traffic spikes, they will throttle and fail instead of scaling up.


---

# 1. Executive Summary

### Opening Statement

> "I have completed the SRE discovery of the Helios PROD Non-AKS Estate. 
> 
> The production environment is significantly cleaner and more disciplined than DEV and QA.

> For instance, all Container Apps run verified production images pinned to immutable commit SHAs (eliminating the placeholder image failures that broke QA), and our Service Bus DLQ is completely clean with 0 dead-lettered messages. 
> 
> However, I identified **three critical operational vulnerabilities**. 

> First, our primary customer-facing portals have zero synthetic availability monitoring. Second, 10 of 11 resources lack diagnostic streaming.

> Third, the one failing PROD resource in our daily QRE report represents a 'silent alert' blind spot where a permanent host crash stays green.


---

# 2. Scope & Inventory Baseline

## 2.1 What I Did

> I validated the current PROD inventory against the 8/3 platform baseline audit.
>
> The 8/3 audit reported 17 Azure compute resources. After validating the PROD subscription (`9b9e9af9-5917-4cae-88b4-1304f3ea98b4`), I confirmed the inventory as **6 Azure Function Apps** and **11 Non-AKS compute resources**, distributed across **2 Resource Groups**.

## 2.2 Resource Breakdown

| Resource Type | Count |
|---|---:|
| Container Apps | 7 |
| Container App Environments | 2 |
| App Services (Web Apps) | 1 |
| Static Web Apps | 1 |
| **Total Non-AKS Resources** | **11** |

## 2.3 Key Takeaway

> Out of these 11 resources, **10 are active, 1 is unknown** (deployed via SwaCli), and importantly, there are **0 abandoned or failed resources**. Furthermore, the Service Bus Dead-Letter Queue is **completely clean (0 messages)** compared to massive backlogs in DEV (18.1k) and QA (31.5k).

---

# 3. Finding #1 – Critical Alert Blind Spot (The 503 Silent Failure)

## 3.1 Resource

`helios-github-activity-logger-prod-func`

## 3.2 What I Found

> This is the sole failing PROD resource that appears continuously in the `#daily-qre-report` (≥27 days).
> 
> The Python host runtime crashed during container boot (likely a broken dependency, missing module, or Key Vault timeout). Because the host never successfully started, the Azure front-end gateway is intercepting all incoming probes and returning HTTP 503 directly.

## 3.3 Why It Matters

> We have a "Silent Failure Paradox." 
> 
> We have a metric alert (`Http5xx > 0`) set up for this resource, but it is currently **GREEN**. Because the Azure gateway returns the 503 before the request ever reaches the host process, the `Http5xx` metric is not incremented. The service is 100% down, but the alarm does not ring.

## 3.4 Recommendation

> We must eliminate this metric blind spot.


---

# 4. Finding #2 – Zero Synthetic Availability Monitoring

## 4.1 Resource

`helios-prod-ui-webapp`

## 4.2 What I Found

> This resource serves two live, customer-facing production custom domains:
> - `https://helios.es.qcells.com`
> - `https://prime.ems.es.qcells.com`
> 
> However, there are **0 URL Ping Web Tests** configured for either domain.

## 4.3 Why It Matters

> We are flying blind on our live production portals. If there is an edge/CDN outage, a DNS routing failure, or a certificate expiration, it will be 100% invisible to us until a customer reports it.

## 4.4 Recommendation

> Deploy Azure Monitor URL Ping Tests immediately.


---

# 5. Finding #3 – Promotion Success on Container Apps (A Positive Finding)

## 5.1 What I Found

> In QA, we discovered that microservices were running broken "Hello World" placeholder images on mismatched ports, causing 100% request failures.
>
> In PROD, the deployment pipeline is working exactly as intended. All 7 custom Container Apps run verified production ACR images that are properly pinned to an immutable commit SHA (`7f1586ebbc0f3f61330a5371e5436875b4223633`) on matching ports. 

## 5.2 Why It Matters

> This proves our production deployment hygiene is much stronger than in lower environments and provides a blueprint for standardizing QA deployments.

---

# 6. Finding #4 – Zero Container Autoscaling Rules

## 6.1 What I Found

> While the PROD Container Apps are correctly deployed, they lack dynamic scaling elasticity.
> All 7 Container Apps define replica bounds (e.g., 1-3 replicas), but their `RuleCount` is **0**.

## 6.2 Why It Matters

> With zero scaling rules defined (like CPU % or concurrent requests), these containers will **never** autoscale under load. If there is a sudden spike in traffic, the apps risk severe CPU throttling and 504 timeouts instead of scaling up to 3 replicas.

## 6.3 Recommendation

> I must establish dynamic elasticity for production bursts.


---

# 7. Finding #5 – Observability & Security Governance Gaps

## 7.1 What I Found

> Production governance still has several legacy holdovers.

### Gaps Identified:
- **Diagnostic Settings:** 10 of 11 resources lack Diagnostic Settings forwarding to Log Analytics.
- **Application Insights:** `helios-prod-ui-appservice` relies on a deprecated Instrumentation Key instead of a Connection String.
- **Key Vaults:** All 8 production Key Vaults are running legacy Access Policies instead of modern Azure RBAC.

## 7.2 Recommendation

> Harden the production baseline.

---

# 9. Key Numbers

| Metric | Value |
|---|---:|
| Total Non-AKS Resources | **11** |
| Container Apps | **7** |
| Container App Environments | **2** |
| Production Key Vaults on Legacy Policies | **8** |
| Unmonitored Live Custom Domains | **2** |
| `kg-event-processor` DLQ | **0 (Clean!)** |
| Resources Missing Diagnostic Settings | **10 / 11** |
| Failing PROD Resources | **1** |

---

# 10. Expected Questions

## Q1. Custom Domain Availability SLOs
> **What is the formal availability target for the custom domains?**
> We recommend a 99.9% uptime target with 5-minute multi-region synthetic ping alerting.

## Q2. Key Vault RBAC Migration Window
> **Can we schedule the migration of the 8 production Key Vaults during the next maintenance window?**
> Yes. We will pre-assign the `Key Vault Secrets User` roles to the Managed Identities first to ensure a zero-downtime cutover.

## Q3. Container App Dynamic Autoscaling
> **Do application teams approve setting dynamic autoscaling bounds?**
> We recommend a minimum of 1 and maximum of 5 replicas, scaling on 50 concurrent requests or 75% CPU to gracefully handle bursty document generation loads.

## Q4. GitHub Activity Logger Ownership
> **Who is the designated point of contact to fix the Python host crash?**
> We need to identify the application engineering owner so SRE can hand over the logs and assist in resolving the crash.

---

# 11. Closing Statement

> The PROD discovery validates that our core deployment mechanics are strong—our Container Apps run immutable SHAs, and our message queues are clean. 
> 
> However, we are operating production workloads with significant observability blind spots. The silent failure of the GitHub Activity Logger proves that improper metric alerts leave us completely vulnerable. 
> 
> By executing this 3-phase remediation plan, I will establish true synthetic monitoring, enable dynamic elasticity for traffic spikes, and secure our Key Vaults with modern RBAC.
