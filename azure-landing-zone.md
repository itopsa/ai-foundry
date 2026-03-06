# Azure Landing Zone – AI Foundry Best Practices (3 Use Cases)

```mermaid
flowchart TD
    subgraph MG["🏛 Azure Management Group — AI Foundry Organization"]
        direction LR
        POL["📜 Azure Policy\nNo Public Endpoints · No Public Access · TLS 1.2+"]
        RBAC_MG["👥 Azure RBAC\nReader/Contributor scoped per project subscription"]
        MON["📊 Azure Monitor + Defender for Cloud\nSecurity posture · alerts · logs"]
        KV_MG["🔑 Azure Key Vault\nCentralised secrets · CMK encryption"]
        LAW["📋 Log Analytics Workspace\nCentralised diagnostics · audit logs"]
    end

    subgraph HUB_SUB["Hub Subscription — Platform Landing Zone"]
        subgraph HUB_VNET["Hub VNet  10.0.0.0/16"]
            direction LR
            FW["🔥 Azure Firewall Premium\nSubnet: 10.0.1.0/26"]
            VPN_GW["🔗 VPN / ExpressRoute GW\nSubnet: 10.0.0.0/27"]
            BASTION["🛡 Azure Bastion\nSubnet: 10.0.2.0/27"]
            DNS["🌐 Private DNS Zones\nprivatelink.openai.azure.com\nprivatelink.search.windows.net\nprivatelink.documents.azure.com"]
            UDR["↪ UDR / Route Table\nForce tunnel → Firewall"]
            NSG_HUB["🚧 NSG\nHub default deny"]
            DDOS["☁ DDoS Protection Standard"]
        end
    end

    subgraph AIF_SUB["AI Foundry Subscription — Application Landing Zone"]
        subgraph AIF_HUB["AI Foundry Hub  —  Single Hub hosting all Projects"]

            subgraph UC1["🔐  Spoke VNet UC1   10.1.0.0/16   —   VNet Peering to Hub"]
                UC1_PROJ["Project 1  |  Team 1\nRBAC: Team1 Contributor"]
                UC1_AGT["Agent Subnet  10.1.1.0/24\nPrivate Agent Capability\nNSG: deny-all inbound · allow HTTPS outbound"]
                UC1_PE_SN["Private Endpoint Subnet — Subnet1  10.1.2.0/24"]
                PE_LLM["pe-llm\nOpenAI"]
                PE_DATA["pe-data\nStorage / Cosmos"]
                PE_SRCH["pe-srch\nAI Search"]
                UC1_RES["Connected Resources via Private Endpoint\nAzure OpenAI · AI Search\nCosmos DB · Azure Storage · Key Vault"]
            end

            subgraph UC2["🟢  Spoke VNet UC2   10.2.0.0/16   —   VNet Peering to Hub"]
                UC2_PROJ["Project 2  |  Team 2\nRBAC: Team2 Contributor"]
                UC2_SN["App Subnet  10.2.1.0/24\nService Endpoints: OpenAI · Storage · KeyVault\nNSG: allow HTTPS outbound to Azure service tags"]
                UC2_CONN["Connected via Service Endpoints\nAzure OpenAI · AI Search\nCosmos DB · Storage · Key Vault\nNo Private Endpoints required"]
                UC2_NSG["NSG Best Practices\nDeny all public inbound\nAllow HTTPS to Azure PaaS tags\nLog all denied traffic to Log Analytics"]
            end

            subgraph UC3["🟠  Spoke VNet UC3   10.3.0.0/16   —   VNet Peering to Hub"]
                UC3_PROJ["Project 3  |  Team 3\nRBAC: Team3 Contributor"]
                UC3_SN["Shared Services Subnet  10.3.1.0/24\nNSG: restricted to AI Foundry projects only"]
                UC3_COSMOS["Cosmos DB\nShared · PE"]
                UC3_LAKE["Data Lake\nStorage Container\nShared · PE"]
                UC3_SRCH["AI Search\nShared index"]
                UC3_RBAC["RBAC: Reader role on shared resources\nManaged Identity per project\nKey Vault for shared secrets"]
                UC3_NOTE["⚠ Cosmos DB + Data Lake are EXTERNAL\nto AI Foundry — in Shared Services Sub\nConnected via Cross-Sub Private Endpoint"]
            end

        end
    end

    subgraph SHARED_SUB["Shared Services Subscription"]
        direction LR
        SH_COSMOS["Cosmos DB\nPrivate Endpoint"]
        SH_LAKE["Data Lake\nStorage Container\nPrivate Endpoint"]
        SH_KV["Key Vault\nCMK + Secrets · PE"]
        SH_ACR["Container Registry\nAI Model images · PE"]
        SH_OPENAI["Azure OpenAI Service\nPrivate Endpoint\nShared across projects"]
        SH_SEARCH["Azure AI Search\nPrivate Endpoint\nShared index"]
    end

    %% ── UC1 internal flow ──────────────────────────────────────────────
    UC1_PROJ --> UC1_AGT
    UC1_AGT  --> UC1_PE_SN
    UC1_PE_SN --> PE_LLM & PE_DATA & PE_SRCH
    PE_LLM & PE_DATA & PE_SRCH --> UC1_RES

    %% ── UC2 internal flow ──────────────────────────────────────────────
    UC2_PROJ --> UC2_SN
    UC2_SN   --> UC2_CONN
    UC2_SN   --> UC2_NSG

    %% ── UC3 internal flow ──────────────────────────────────────────────
    UC3_PROJ --> UC3_SN
    UC3_SN   --> UC3_COSMOS & UC3_LAKE & UC3_SRCH
    UC3_SN   --> UC3_RBAC
    UC3_COSMOS & UC3_LAKE & UC3_SRCH --> UC3_NOTE

    %% ── Hub → Spoke VNet Peering ───────────────────────────────────────
    HUB_VNET -.->|VNet Peering| UC1
    HUB_VNET -.->|VNet Peering| UC2
    HUB_VNET -.->|VNet Peering| UC3

    %% ── UC3 → Shared Services (cross-subscription) ─────────────────────
    UC3_NOTE -->|Cross-Sub Private Endpoint| SHARED_SUB

    %% ── Styling ────────────────────────────────────────────────────────
    classDef hub    fill:#0050ef,color:#fff,stroke:#001dbc
    classDef uc1    fill:#f8cecc,color:#000,stroke:#b85450
    classDef uc2    fill:#d5e8d4,color:#000,stroke:#82b366
    classDef uc3    fill:#ffe6cc,color:#000,stroke:#d79b00
    classDef shared fill:#fff2cc,color:#000,stroke:#d6b656
    classDef gov    fill:#ffe6cc,color:#000,stroke:#d79b00
    classDef note   fill:#f0a30a,color:#fff,stroke:#bd7000

    class FW,VPN_GW,BASTION,DNS,UDR,NSG_HUB,DDOS hub
    class UC1_PROJ,UC1_AGT,UC1_PE_SN,PE_LLM,PE_DATA,PE_SRCH,UC1_RES uc1
    class UC2_PROJ,UC2_SN,UC2_CONN,UC2_NSG uc2
    class UC3_PROJ,UC3_SN,UC3_COSMOS,UC3_LAKE,UC3_SRCH,UC3_RBAC uc3
    class UC3_NOTE note
    class SH_COSMOS,SH_LAKE,SH_KV,SH_ACR,SH_OPENAI,SH_SEARCH shared
    class POL,RBAC_MG,MON,KV_MG,LAW gov
```

---

## Legend

| Colour | Meaning |
|---|---|
| 🔵 Blue | Hub networking components (Firewall, Bastion, DNS, etc.) |
| 🔴 Red/Pink | UC1 – Private Agent (highest security, private endpoints only) |
| 🟢 Green | UC2 – LLM + AI Search (service endpoints, standard tier) |
| 🟠 Orange | UC3 – Shared Agent Resources (cross-sub access, read-only) |
| 🟡 Yellow | Shared Services (Cosmos DB, Data Lake, Key Vault, ACR) |
| 🔶 Amber | Governance & Policy (Management Group level controls) |
