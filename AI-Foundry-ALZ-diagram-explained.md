# AI Foundry ALZ Diagram – Explained

The **AI Foundry ALZ** diagram describes an **Azure Landing Zone** for AI workloads: governance, hub-and-spoke networking, three security tiers, and shared platform services.

---

## 1. Management Group (top)

Governance and policy for the AI Foundry org:

- **Azure Policy** – Enforce private endpoints, no public access, TLS 1.2+
- **RBAC** – Reader/Contributor scoped per project subscription
- **Azure Monitor + Defender for Cloud** – Security posture, alerts, logs
- **Key Vault** – Central secrets, CMK encryption
- **Log Analytics Workspace** – Central diagnostics and audit logs

---

## 2. Hub Subscription (platform)

Central networking and security:

- **Hub VNet** (10.0.0.0/16) with:
  - **Azure Firewall (Premium)** – Outbound control
  - **VPN / ExpressRoute** – On-prem connectivity
  - **Azure Bastion** – Secure admin access
  - **Private DNS Zones** – For OpenAI, AI Search, Cosmos DB
  - **UDR** – Force traffic through the firewall
  - **NSG** – Hub default deny
  - **DDoS Protection Standard**

---

## 3. AI Foundry Subscription (application)

Three spoke VNets, each a different use case:

| Use Case | Security Level | Approach |
|----------|----------------|----------|
| **UC1** (10.1.0.0/16) | Highest | **Private Agent** – Private endpoints for OpenAI, AI Search, Cosmos DB, Storage, Key Vault. Agent subnet with strict NSG (deny inbound, allow HTTPS outbound). |
| **UC2** (10.2.0.0/16) | Standard | **LLM + AI Search only** – Service endpoints (no private endpoints). Public endpoints with VNet service rules and IP restrictions. |
| **UC3** (10.3.0.0/16) | Shared | **Shared Agent Resources** – Shared Cosmos DB, Data Lake, AI Search across projects. Private endpoints from each spoke. RBAC and Managed Identity per project. |

---

## 4. Shared Services Subscription

Shared platform services used by multiple projects:

- Cosmos DB, Data Lake, Key Vault, Container Registry
- Azure OpenAI and AI Search (shared indexes)
- All exposed via private endpoints

---

## Flow

- Hub VNet peers to each spoke (UC1, UC2, UC3).
- UC3 connects to Shared Services via cross-subscription private endpoints.
- Traffic is routed through the Hub firewall and DNS for private resolution.

---

## Summary

The diagram shows how to run AI workloads (agents, LLMs, search) in Azure with different security levels and shared platform services.

**Related:**

- Diagram: [AI Foundry ALZ](https://itopsa.github.io/ai-foundry/AI%20Foundry%20ALZ.drawio.html) (drawio)
- Mermaid + details: [azure-landing-zone.md](azure-landing-zone.md)
