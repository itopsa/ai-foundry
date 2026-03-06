# AI Foundry Platform – Hub-Based Architecture

> A centralised Azure AI Foundry Hub hosting multiple agency projects with Hub-and-Spoke networking, RBAC isolation, and three onboarding patterns.

**Status:** DRAFT &nbsp;|&nbsp; **Platform:** Azure AI Foundry &nbsp;|&nbsp; **Model:** Hub-Based &nbsp;|&nbsp; **Version:** 1.0 &nbsp;|&nbsp; **Date:** March 2026

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture Model](#2-architecture-model)
3. [Network Topology](#3-network-topology)
4. [Use Cases](#4-use-cases)
5. [Azure Resource Inventory](./README-PAGE2.md#5-azure-resource-inventory)
6. [Security Controls](./README-PAGE2.md#6-security-controls)
7. [RBAC Model](./README-PAGE2.md#7-rbac-model)
8. [Naming Conventions](./README-PAGE2.md#8-naming-conventions)
9. [Onboarding Guide](./README-PAGE2.md#9-onboarding-guide-quick-start)
10. [IaC & Deployment](./README-PAGE2.md#10-iac--deployment)
11. [Prerequisites](./README-PAGE2.md#11-prerequisites)
12. [FAQ](./README-PAGE2.md#12-frequently-asked-questions)
13. [Contact](./README-PAGE2.md#13-contributing--contact)

---

## 1. Overview

The **AI Foundry Platform** is a **Hub-Based Azure AI Foundry deployment** that enables multiple agency teams to onboard AI projects quickly, securely, and consistently. Rather than each team independently provisioning AI infrastructure, all projects share a single AI Foundry Hub — isolating teams through Azure RBAC and dedicated VNet spoke networks.

This architecture was designed to address the three most common onboarding patterns encountered across agency teams, ranging from fully private agent deployments requiring zero public internet exposure, to lightweight LLM access, to projects that consume shared enterprise data resources.

**What this repo/document contains:** Architecture diagrams (Pages 1–2), PRD & Technical Design (Pages 3–4), and this README (Pages 5–6). The draw.io file is the single source of truth for all design artifacts.

---

## 2. Architecture Model

This platform uses the **Hub + Projects model** — one AI Foundry Hub owns the shared infrastructure and acts as the governance boundary. Each team onboards as a **Project** under this Hub. The Hub is *not* a separate deployment per team (that would be the Foundry Projects-only model, which is higher cost and loses shared governance).

**Hub responsibilities:** Owns the AI Foundry resource, shared connected resources (Azure OpenAI, AI Search, Storage, Key Vault), VNet peering configs, and Private DNS Zones.

**Project responsibilities:** Team RBAC, per-project connected resources (where UC requires), private endpoints scoped to the project spoke, and workload compute.

### Model Comparison

| Attribute | Hub-Based ✅ (THIS architecture) | Foundry Projects-Only ❌ (alternative) |
|---|---|---|
| Cost | Lower – shared OpenAI, Search, Storage | Higher – duplicated resources per team |
| Governance | Centralised at Hub (policies, DNS, monitoring) | Per-project, inconsistent across teams |
| Team isolation | RBAC + subnet-level isolation per project | Full resource-level isolation (overkill for most) |
| Onboarding speed | Fast – new Project under existing Hub (<5 days) | Slow – new Hub + infrastructure per team |

---

## 3. Network Topology

The platform uses an **Azure Hub-and-Spoke VNet topology** within a single subscription. All egress traffic is force-tunnelled through Azure Firewall Premium in the Hub VNet. Each use case gets a dedicated Spoke VNet peered to the Hub.

| Component | Address Space | Key Resources |
|---|---|---|
| **Hub VNet** | 10.0.0.0/16 | Azure Firewall Premium, VPN/ExpressRoute GW, Bastion, Private DNS Zones, UDR, NSG, DDoS Standard |
| **Spoke VNet – UC1 (Private Agent)** | 10.1.0.0/16 | Agent Subnet (10.1.1.0/24) + Private Endpoint Subnet (10.1.2.0/24) \| pe-llm, pe-data, pe-srch \| NSG: deny-all inbound |
| **Spoke VNet – UC2 (LLM + Search)** | 10.2.0.0/16 | App Subnet (10.2.1.0/24) \| Service Endpoints: Microsoft.CognitiveServices, Microsoft.Storage, Microsoft.KeyVault \| NSG: deny inbound |
| **Spoke VNet – UC3 (Shared Resources)** | 10.3.0.0/16 | Shared Services Subnet (10.3.1.0/24) \| Cross-sub Private Endpoints to Cosmos DB, Data Lake, AI Search \| Managed Identity access |

---

## 4. Use Cases

### Use Case 1 – Private Agent Capability

**Target teams:** LIRR, OMB (high-security, regulated data environments)

**Pattern:** Fully private. Zero public internet path. All PaaS resources accessed through Private Endpoints only.

**Network:** Dedicated Agent Subnet + Private Endpoint Subnet (Subnet1). NSG deny-all inbound. UDR to Firewall.

**Resources:** Azure OpenAI (pe-llm), AI Search (pe-srch), Cosmos DB (pe-data), Azure Storage, Key Vault — all via PE.

**DNS:** Private DNS A-records in Hub DNS Zones (`privatelink.openai.azure.com`, `privatelink.search.windows.net`, `privatelink.documents.azure.com`).

**When to use:** PII data, regulated workloads, zero-trust requirement, agency mandates no public endpoint.

---

### Use Case 2 – LLM + AI Search Access Only

**Target teams:** Standard agency teams needing AI inference and search, no sensitive data

**Pattern:** Service Endpoints + VNet restrictions. No private endpoints required. Simpler, lower cost.

**Network:** App Subnet. Service Endpoints for CognitiveServices, Storage, KeyVault. NSG deny inbound, allow HTTPS to Azure service tags.

**Resources:** Azure OpenAI (VNet SR), AI Search (IP allowlist + AAD), Cosmos DB (Service Endpoint), Storage, Key Vault.

**Monitoring:** NSG flow logs + Log Analytics. All denied traffic captured.

**When to use:** Internal data, non-regulated, team primarily needs LLM calls and search indexing.

---

### Use Case 3 – Shared Agent Resources

**Target teams:** Projects that consume shared enterprise data platforms

**Pattern:** Cross-subscription Private Endpoints + Managed Identity. Resources live outside AI Foundry in a shared services subscription.

**Resources:** Cosmos DB, Data Lake Storage Container, AI Search (all shared, PE from Spoke). Key Vault in shared sub holds connection strings.

**Access control:** Each project's Managed Identity is granted Reader/Data Reader role on shared resources. No wildcard grants. Read-only access enforced.

**DNS:** Private endpoint DNS A-records registered in Hub DNS Zones for cross-sub resolution.

**When to use:** Data already exists in enterprise data lake; duplication not acceptable; AI Foundry project reads from shared platform.

---

*Continued in [README-PAGE2.md](./README-PAGE2.md) — Sections 5–13*

---

> CONFIDENTIAL – AI Foundry Platform Team | README v1.0 | March 2026 | Internal Use Only
