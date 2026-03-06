# AI Foundry Platform – README (cont.) | Sections 5–13

*Continued from [README-PAGE1.md](./README-PAGE1.md)*

---

## 5. Azure Resource Inventory

| Resource | SKU / Tier | UC1 | UC2 | UC3 | Notes |
|---|---|---|---|---|---|
| AI Foundry Hub | Standard S0 | Shared | Shared | Shared | Single Hub hosts all projects |
| Azure OpenAI Service | Standard S0 | Private EP | Svc Endpoint | Shared | GPT-4o deployed per Hub |
| Azure AI Search | Standard S1 | Private EP | IP Restrict | Shared index | Semantic search enabled |
| Cosmos DB | Serverless | Private EP | Svc Endpoint | Shared PE | NoSQL; auto-scale RU |
| Azure Storage (ADLS) | Standard LRS | Private EP | Svc Endpoint | Shared PE | Hierarchical namespace on |
| Azure Key Vault | Standard | Private EP | Svc Endpoint | Shared PE | Soft-delete + purge protect |
| Azure Firewall | Premium | Hub shared | Hub shared | Hub shared | IDPS enabled; TLS inspect |
| Log Analytics WS | Pay-as-you-go | All | All | All | Centralised; 90-day retain |
| VNet (Spoke) | N/A | 10.1.0.0/16 | 10.2.0.0/16 | 10.3.0.0/16 | Each peered to Hub VNet |
| Managed Identity | System-assigned | Per project | Per project | Per project | Used for all resource auth |
| Azure Bastion | Standard | Hub shared | Hub shared | Hub shared | Secure RDP/SSH; no public IP |
| Container Registry | Standard | Optional | Optional | Shared | For AI model container images |

---

## 6. Security Controls

### Identity & Authentication
All authentication via Azure Active Directory (Entra ID). No shared keys, passwords, or connection strings in code. Managed Identity per project for all resource access. Conditional Access enforced for admin operations.

### Network Security
NSG deny-all inbound on all subnets. Outbound restricted to Azure service tags only. Azure Firewall Premium with IDPS enabled. TLS inspection active. Force-tunnel via UDR – no direct internet egress from any spoke.

### Data Protection
All storage encrypted with Customer-Managed Keys (CMK) in centralised Key Vault. TLS 1.2+ enforced via Azure Policy. Key Vault soft-delete (90 days) and purge protection enabled. No plaintext secrets in repos or configs.

### Monitoring & Alerting
All resources emit diagnostic logs to centralised Log Analytics Workspace. NSG flow logs and Azure Firewall logs enabled. Defender for Cloud Standard on subscription. Alerts for: failed auth, unusual egress, policy violations, quota thresholds.

### Policy & Compliance
Azure Policy enforces: no public endpoints (UC1), TLS 1.2+, CMK encryption, approved regions only, required tags (env, team, project, cost-centre). Deny effect on non-compliant resources. Policy compliance dashboard reviewed monthly.

---

## 7. RBAC Model

| Role | Assigned To | Scope | Permissions |
|---|---|---|---|
| Hub Owner | Platform Team only | AI Foundry Hub | Create/delete projects, manage Hub settings, assign project-level roles |
| Project Contributor | Team AAD Group | AI Foundry Project (scoped) | Deploy models, manage project resources, view logs. Cannot access sibling projects. |
| Reader | Stakeholders / Auditors | AI Foundry Project | Read-only view of project resources and outputs. No write access. |
| Managed Identity – Data Reader | Project Managed Identity | Shared Resources (UC3) | Read data from Cosmos DB, Data Lake, AI Search. No write, no delete. |
| Key Vault Secrets User | Project Managed Identity | Key Vault (scoped) | Read secrets. Cannot set, delete, or purge secrets. Scoped per project vault. |
| Network Contributor | Platform Team only | Hub VNet + Spoke VNets | Manage peering, NSGs, UDRs, private endpoints. Not granted to project teams. |
| Log Analytics Reader | All Project Teams | Log Analytics Workspace | Query own project logs only (workspace scoped by resource tags). |

---

## 8. Naming Conventions

| Resource Type | Naming Pattern | Example |
|---|---|---|
| Resource Group | `rg-aif-{env}-{team}-{region}` | `rg-aif-prod-lirr-eus` |
| AI Foundry Hub | `aih-{org}-{env}-{region}` | `aih-nymta-prod-eus` |
| AI Foundry Project | `aip-{team}-{env}` | `aip-lirr-prod` |
| VNet (Hub) | `vnet-hub-{org}-{env}` | `vnet-hub-nymta-prod` |
| VNet (Spoke) | `vnet-spoke-{team}-{env}` | `vnet-spoke-lirr-prod` |
| Subnet (Agent) | `snet-agent-{team}-{env}` | `snet-agent-lirr-prod` |
| Subnet (PE) | `snet-pe-{team}-{env}` | `snet-pe-lirr-prod` |
| Private Endpoint | `pe-{svc}-{team}-{env}` | `pe-llm-lirr-prod` / `pe-srch-lirr-prod` |
| Key Vault | `kv-{team}-{env}-{region}` | `kv-lirr-prod-eus` (max 24 chars) |
| Log Analytics WS | `law-aif-{org}-{env}` | `law-aif-nymta-prod` |
| NSG | `nsg-{subnet}-{team}-{env}` | `nsg-agent-lirr-prod` |
| Managed Identity | `id-{team}-{env}` | `id-lirr-prod` |

---

## 9. Onboarding Guide (Quick Start)

**Step 1: Submit Onboarding Request**
Open a service desk ticket. Provide: team name, project name, use case (UC1/UC2/UC3), data classification level, list of required resources, designated technical lead, and Azure AD group name.

**Step 2: Platform Team Review**
Platform team validates use case selection, checks subnet CIDR availability, confirms resource requirements, and assigns VNet address space from the master IPAM table.

**Step 3: Provision via IaC**
Platform team runs the appropriate Bicep/Terraform module:
- UC1: `module/uc1-private-agent`
- UC2: `module/uc2-llm-search`
- UC3: `module/uc3-shared-resources`

All modules are pre-approved and policy-compliant.

**Step 4: Configure RBAC**
Assign team AAD group to Contributor role scoped to their AI Foundry Project. Create system-assigned Managed Identity for the project. For UC3: assign Reader role on shared resources.

**Step 5: Connect Resources**
Register connected resources in the AI Foundry Project. UC1: deploy private endpoints + DNS A-records in Hub zones. UC2: configure service endpoints + IP restrictions. UC3: register cross-sub private endpoints.

**Step 6: Validate & Test**
Platform team runs connectivity validation script. Verifies: DNS resolution, private endpoint reachability, RBAC isolation (no cross-project access), NSG rules, Firewall logs clean.

**Step 7: Handoff & Go Live**
Team receives: project URL, access credentials, runbook, and monitoring dashboard link. Project marked **LIVE** in the registry. Onboarding ticket closed.

---

## 10. IaC & Deployment

**Tooling:** Bicep (primary) + Terraform modules (optional). All deployments via Azure DevOps pipeline or GitHub Actions. No manual portal deployments — enforced by Azure Policy (deny deployments outside approved pipeline service principal).

**Repo structure:**
```
/infra/hub/              # Hub VNet, Firewall, Bastion, DNS
/infra/modules/uc1/      # UC1 private agent module
/infra/modules/uc2/      # UC2 LLM + Search module
/infra/modules/uc3/      # UC3 shared resources module
/infra/shared/           # Cosmos DB, Data Lake, Key Vault
/scripts/validate-onboarding.sh
```

**Pipeline flow:** `lint → plan → security scan (Checkov/tfsec) → approval gate → apply → smoke test → notify team`

**State management:** Terraform state in Azure Blob Storage (private endpoint, versioning enabled, RBAC restricted to pipeline SP only).

---

## 11. Prerequisites

| Requirement | Details |
|---|---|
| **Azure Subscription** | Active subscription with Owner or Contributor rights. Quota pre-increased for OpenAI (GPT-4o), AI Search, Cosmos DB in target region (e.g. East US). |
| **Azure AD / Entra ID** | Global Admin or Privileged Role Administrator to assign RBAC roles. AAD groups created per team before onboarding. |
| **Networking** | IP Address Management (IPAM) plan agreed. Hub VNet deployed (or deploy via `/infra/hub/`). ExpressRoute or VPN if on-prem connectivity is needed for UC1. |
| **Tooling** | Azure CLI 2.55+, Bicep CLI 0.23+, Terraform 1.6+ (optional), Python 3.10+ (for validation scripts), Git access to this repo. |
| **AI Foundry Permissions** | Subscription-level permission to create AI Foundry Hub resource. `Microsoft.MachineLearningServices` provider registered on subscription. |
| **Defender for Cloud** | Defender for Cloud Standard plan enabled on subscription BEFORE first project onboards (required for UC1 compliance). |

---

## 12. Frequently Asked Questions

**Q: Do the 3 use cases require separate subscriptions?**
A: No. All three use cases run within a single subscription. Isolation is achieved through RBAC (scoped per project) and dedicated VNet spokes. Separate subscriptions are only needed if there is a hard compliance boundary (e.g. FedRAMP vs standard) or a billing chargeback requirement that cannot be satisfied with cost tags.

**Q: Can UC1 and UC2 teams share an AI Search index?**
A: No by default. Each project has its own connected AI Search resource. UC3 is the pattern for intentional resource sharing — it provides controlled read-only access via Managed Identity with explicit RBAC grants.

**Q: What happens if a team needs both private endpoints AND shared resources?**
A: A project can be a hybrid of UC1 and UC3. The private endpoint pattern from UC1 is applied for connectivity, and the shared resource access model from UC3 is applied for the external Cosmos DB / Data Lake. Document this as a custom variant in the onboarding ticket.

**Q: Can teams deploy their own resources outside the AI Foundry Project?**
A: Only within their scoped Resource Group. Azure Policy prevents deploying public-facing resources or resources in non-approved regions. All networking changes must go through the Platform Team.

**Q: How are LLM model deployments managed?**
A: Model deployments (e.g. GPT-4o, Ada-002) are managed at the Hub level by the Platform Team. Teams request new deployments via ticket. Quota is allocated per project using Azure OpenAI deployment quotas.

**Q: Is the Hub a single point of failure?**
A: The AI Foundry Hub is a control plane resource. It does not process inference traffic — that goes directly to Azure OpenAI. Hub unavailability affects project management (portal/API), not live AI inference calls already in flight.

---

## 13. Contributing & Contact

**Document Owner:** AI Platform Team — responsible for all architecture decisions, IaC modules, and onboarding approvals.

**To request changes to this architecture:** Raise a PR against this draw.io file in the platform repo, or open an Architecture Review Board (ARB) ticket for major changes (new use case, subscription changes, security control changes).

**To onboard a new project:** Follow [Section 9](#9-onboarding-guide-quick-start) — submit a service desk ticket with the required fields. SLA: 5 business days from approved ticket to live project.

**Security incidents:** Report immediately to the Security Operations Centre (SOC). Do not raise a standard service desk ticket for suspected breaches.

---

> AI Foundry Platform | README v1.0 | March 2026 | CONFIDENTIAL – Internal Use Only | Owner: AI Platform Team
