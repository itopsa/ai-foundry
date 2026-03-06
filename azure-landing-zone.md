# Azure Landing Zone – AI Foundry Best Practices (3 Use Cases)

```mermaid
flowchart TD
    subgraph MG["Azure Management Group (AI Foundry Organization)"]
        direction LR
        POL["Azure Policy\n(Enforce Private Endpoints,\nNo Public Access, TLS 1.2+)"]
        RBAC_MG["Azure RBAC\n(Reader/Contributor scoped\nper project subscription)"]
        MON["Azure Monitor + Defender for Cloud\n(Security posture, alerts, logs)"]
        KV_MG["Azure Key Vault\n(Centralised secrets,\nCMK encryption)"]
        LAW["Log Analytics Workspace\n(Centralised diagnostics,\naudit logs)"]
        POL ~~~ RBAC_MG ~~~ MON ~~~ KV_MG ~~~ LAW
    end

    subgraph HUB_SUB["Hub Subscription (Platform Landing Zone)"]
        subgraph HUB_VNET["Hub VNet (10.0.0.0/16)"]
            direction LR
            FW["Azure Firewall\n(Premium)\nSubnet: 10.0.1.0/26"]
            VPN["VPN / ExpressRoute\nGateway\nSubnet: 10.0.0.0/27"]
            BASTION["Azure Bastion\n(Secure admin access)\nSubnet: 10.0.2.0/27"]
            DNS["Azure Private DNS Zones\n(privatelink.openai.azure.com\nprivatelink.search.windows.net\nprivatelink.documents.azure.com)"]
            UDR["UDR / Route Table\n(Force tunnel to Firewall)"]
            NSG_HUB["NSG\n(Hub default deny)"]
            DDOS["DDoS Protection\nStandard"]
            FW ~~~ VPN ~~~ BASTION ~~~ DNS ~~~ UDR ~~~ NSG_HUB ~~~ DDOS
        end
    end

    subgraph AIF_SUB["AI Foundry Subscription (Application Landing Zone)"]
        subgraph AIF_HUB["AI Foundry Hub Resource\n(Single Hub hosting all Projects)"]
            direction LR

            subgraph UC1["Spoke VNet UC1 (10.1.0.0/16) – VNet Peering to Hub"]
                direction TB
                UC1_HDR["Use Case 1: Private Agent Capability\n(LIRR, OMB – highest security)"]
                UC1_PROJ["Project 1 (Team 1)\nRBAC: Team1 Contributor"]
                UC1_AGT["Agent Subnet (10.1.1.0/24)\nPrivate Agent Capability\nNSG: deny all inbound, allow HTTPS outbound"]
                UC1_PE_SN["Private Endpoint Subnet – Subnet1 (10.1.2.0/24)"]
                PE_LLM["pe-llm\n(OpenAI)"]
                PE_DATA["pe-data\n(Storage/Cosmos)"]
                PE_SRCH["pe-srch\n(AI Search)"]
                UC1_RES["Connected Resources:\n• Azure OpenAI (private endpoint)\n• AI Search (private endpoint)\n• Cosmos DB (private endpoint)\n• Azure Storage (private endpoint)\n• Key Vault (private endpoint)"]
                UC1_HDR --> UC1_PROJ --> UC1_AGT --> UC1_PE_SN
                UC1_PE_SN --> PE_LLM & PE_DATA & PE_SRCH
                PE_LLM & PE_DATA & PE_SRCH --> UC1_RES
            end

            subgraph UC2["Spoke VNet UC2 (10.2.0.0/16) – VNet Peering to Hub"]
                direction TB
                UC2_HDR["Use Case 2: LLM + AI Search Access Only\n(Standard, no private endpoints)"]
                UC2_PROJ["Project 2 (Team 2)\nRBAC: Team2 Contributor"]
                UC2_SN["App Subnet (10.2.1.0/24)\nNSG: allow HTTPS outbound to service endpoints\nService Endpoints: OpenAI, Storage, KeyVault"]
                UC2_CONN["Connected via Service Endpoints (no PE):\n• Azure OpenAI (public w/ VNET SR)\n• AI Search (public w/ IP restrict)\n• Azure Storage (service endpoint)\n• Cosmos DB (service endpoint)\n• Key Vault (service endpoint)"]
                UC2_NSG["NSG Best Practices:\n• Deny all public inbound\n• Allow HTTPS to Azure PaaS tags\n• Log all denied traffic to Log Analytics"]
                UC2_HDR --> UC2_PROJ --> UC2_SN --> UC2_CONN --> UC2_NSG
            end

            subgraph UC3["Spoke VNet UC3 (10.3.0.0/16) – VNet Peering to Hub"]
                direction TB
                UC3_HDR["Use Case 3: Shared Agent Resources\n(External resources, shared across projects)"]
                UC3_PROJ["Project 3 (Team 3)\nRBAC: Team3 Contributor"]
                UC3_SN["Shared Services Subnet (10.3.1.0/24)\nNSG: restricted to AI Foundry projects only"]
                UC3_COSMOS["Cosmos DB\n(Shared, private endpoint)"]
                UC3_LAKE["Data Lake\nStorage Container\n(Shared, private endpoint)"]
                UC3_SRCH["AI Search\n(Shared index)"]
                UC3_RBAC["Shared Resource Access:\n• RBAC: Reader role on shared resources\n• Managed Identity per project\n• Private endpoints from each spoke\n• Key Vault for shared secrets"]
                UC3_NOTE["NOTE: Cosmos DB + Data Lake are EXTERNAL\nto AI Foundry (in shared services subscription)\nConnected via Private Endpoint from each Spoke"]
                UC3_HDR --> UC3_PROJ --> UC3_SN
                UC3_SN --> UC3_COSMOS & UC3_LAKE & UC3_SRCH
                UC3_COSMOS & UC3_LAKE & UC3_SRCH --> UC3_RBAC --> UC3_NOTE
            end

        end
    end

    subgraph SHARED_SUB["Shared Services Subscription"]
        direction LR
        SH_COSMOS["Cosmos DB\n(Private Endpoint)"]
        SH_LAKE["Data Lake\nStorage Container\n(Private Endpoint)"]
        SH_KV["Key Vault\n(CMK + Secrets)\n(Private Endpoint)"]
        SH_ACR["Container Registry\n(AI Model images)\n(Private Endpoint)"]
        SH_OPENAI["Azure OpenAI Service\n(Private Endpoint)\nShared across projects"]
        SH_SEARCH["Azure AI Search\n(Private Endpoint)\nShared index"]
        SH_COSMOS ~~~ SH_LAKE ~~~ SH_KV ~~~ SH_ACR ~~~ SH_OPENAI ~~~ SH_SEARCH
    end

    %% ── Vertical section ordering (invisible) ─────────────────────────
    MG ~~~ HUB_SUB
    HUB_SUB ~~~ AIF_SUB

    %% ── VNet Peering: Hub → each Spoke ───────────────────────────────
    HUB_VNET -.->|VNet Peering| UC1
    HUB_VNET -.->|VNet Peering| UC2
    HUB_VNET -.->|VNet Peering| UC3

    %% ── UC3 → Shared Services (cross-subscription Private Endpoint) ───
    UC3_NOTE -->|Private Endpoint| SHARED_SUB

    %% ── Styles ────────────────────────────────────────────────────────
    classDef mgBox    fill:#ffe6cc,stroke:#d79b00,color:#000
    classDef hubBox   fill:#0050ef,stroke:#001dbc,color:#fff
    classDef uc1Hdr  fill:#9673a6,stroke:#9673a6,color:#fff
    classDef uc1Box  fill:#e1d5e7,stroke:#9673a6,color:#000
    classDef uc2Hdr  fill:#82b366,stroke:#82b366,color:#fff
    classDef uc2Box  fill:#d5e8d4,stroke:#82b366,color:#000
    classDef uc3Hdr  fill:#d79b00,stroke:#d79b00,color:#fff
    classDef uc3Box  fill:#ffe6cc,stroke:#d79b00,color:#000
    classDef uc3Note fill:#f0a30a,stroke:#bd7000,color:#fff
    classDef shared  fill:#fff2cc,stroke:#d6b656,color:#000

    class POL,RBAC_MG,MON,KV_MG,LAW mgBox
    class FW,VPN,BASTION,DNS,UDR,NSG_HUB,DDOS hubBox
    class UC1_HDR uc1Hdr
    class UC1_PROJ,UC1_AGT,UC1_PE_SN,PE_LLM,PE_DATA,PE_SRCH,UC1_RES uc1Box
    class UC2_HDR uc2Hdr
    class UC2_PROJ,UC2_SN,UC2_CONN,UC2_NSG uc2Box
    class UC3_HDR uc3Hdr
    class UC3_PROJ,UC3_SN,UC3_COSMOS,UC3_LAKE,UC3_SRCH,UC3_RBAC uc3Box
    class UC3_NOTE uc3Note
    class SH_COSMOS,SH_LAKE,SH_KV,SH_ACR,SH_OPENAI,SH_SEARCH shared
```

---

## Legend

| Colour | Meaning |
|---|---|
| 🔵 Blue | Hub Networking (Firewall, Bastion, DNS, VPN, UDR, NSG, DDoS) |
| 🔴 Pink/Purple | UC1 – Private Agent (highest security, private endpoints only) |
| 🟢 Green | UC2 – LLM + AI Search (service endpoints, no private endpoints) |
| 🟠 Orange | UC3 – Shared Agent Resources (cross-sub, read-only Managed Identity) |
| 🟡 Yellow | Shared Services (Cosmos DB, Data Lake, Key Vault, ACR, OpenAI, AI Search) |
| 🔶 Amber | Governance / Policy (Management Group level controls) |
