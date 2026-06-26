# Azure Services Resiliency Inventory

> Generated from Bicep sources — last updated: 2026-04-24
> Scope: Hub Landing Zone + Shared Services Landing Zone (Canada Central)

---

## Resiliency Levels — Legend

| Level | Definition |
|---|---|
| 🟢 **Zone Redundant** | Deployed explicitly across Availability Zones 1, 2, 3 — survives a single-AZ failure |
| 🔵 **Platform HA** | Microsoft manages redundancy internally (zone-redundant by default in Canada Central) |
| 🟡 **Regional** | Deployed in a single region without AZ configuration — survives node/rack failures but not AZ-level outages |
| ⚪ **Global** | Globally distributed by Microsoft — inherently resilient |
| ⬜ **N/A** | Non-stateful policy/config resource |

---

## Architecture Overview

```mermaid
graph TD
    subgraph INT["🌐 Internet / On-Premises"]
        ONP["On-Premises — Riopel\n(IPSec IKEv2)"]
    end

    subgraph HUB["Hub Subscription (canadacentral)"]
        direction TB

        subgraph HUB_NET["Hub Network RG"]
            HUBVNET["Virtual Network\nvnet-cdp-alz-hub-*"]
            AFW["Azure Firewall\nPremium / Basic"]
            VPNG["VPN Gateway\nVpnGw2AZ · Active-Active"]
            LGW["Local Network Gateway"]
            RT["Route Tables ×4"]
        end

        subgraph HUB_DNS["Hub DNS RG"]
            DNSPR["Private DNS Resolver\n2 inbound · 2 outbound endpoints"]
            DNSFR["DNS Forwarding Ruleset\n20+ domain rules"]
            PDZ["Private DNS Zones ×12\nprivatelink.* + custom domains"]
        end

        subgraph HUB_GW["Hub Gateway RG"]
            AGW["Application Gateway\nWAF_v2 · AZ 1,2,3"]
            WAFP["WAF Policy OWASP 3.2"]
            MIAGW["Managed Identity (AGW)"]
            NSGGW["NSG (AGW)"]
            PIPAGW["Public IP (AGW)"]
        end

        subgraph HUB_SH["Hub Shared RG"]
            KVHUB["Key Vault\nkv-alz-hub-*"]
            LAWHUB["Log Analytics\nlog-alz-hub-*"]
        end

        subgraph HUB_ICPL["Hub ICPL RG"]
            PEPSQL["Private Endpoint\nSQL MI (IC)"]
        end
    end

    subgraph SHSVC["Shared Services Subscription (canadacentral)"]
        direction TB

        subgraph SH_BASE["Shared RG"]
            KVSH["Key Vault\nkv-alz-shsvc-*"]
            LAWSH["Log Analytics\nlog-alz-shsvc-*"]
            MISH["Managed Identity\nmi-cdp-shsvc-*"]
        end

        subgraph SH_EXT["Shared Ext RG"]
            SHVNET["Virtual Network\nvnet-cdp-alz-shsvc-*"]
            APIM["API Management\nPremium / Developer · Internal VNet"]
            ACR["Container Registry\nBasic SKU"]
            DC["Dev Center\n3 environment types"]
            APPI["Application Insights"]
            AG["Action Group"]
            NSGAPIM["NSG (APIM)"]
            RTSH["Route Table"]
            PIPAPIM["Public IP (APIM)"]
        end
    end

    ONP -- "IPSec / IKEv2" --> VPNG
    VPNG --> HUBVNET
    LGW --> VPNG
    AFW --> HUBVNET
    AGW -- "HTTPS Reverse Proxy" --> APIM
    DNSPR --> HUBVNET
    PDZ -.-> HUBVNET
    HUBVNET <-- "VNet Peering" --> SHVNET
    APIM --> LAWHUB
    APIM --> APPI
    APIM -- "cert from KV" --> KVHUB
    KVHUB --> LAWHUB
```

---

## Hub Infrastructure Services

### Network & Security (Hub RG)

```mermaid
graph LR
    classDef zr fill:#2e7d32,color:#fff,stroke:#1b5e20
    classDef reg fill:#f57f17,color:#fff,stroke:#e65100
    classDef pha fill:#1565c0,color:#fff,stroke:#0d47a1
    classDef na fill:#9e9e9e,color:#fff,stroke:#757575

    HUBVNET["Virtual Network\nvnet-cdp-alz-hub-*\n🟡 Regional"]:::reg
    AFW["Azure Firewall\nafw-alz-hubfw-*\n🟡 Regional"]:::reg
    FWPOL["Firewall Policy\nafwp-afw-alz-hubfw-*\n⬜ N/A"]:::na
    VPNG["VPN Gateway\nvpng-alz-hub-* — VpnGw2AZ\nActive-Active BGP\n🟢 Zone Redundant"]:::zr
    PIPFW["Public IP — Firewall\npip-afw-*\nAZ 1,2,3\n🟢 Zone Redundant"]:::zr
    PIPVPN1["Public IP — VPN #1\npip-vpng-*-01\nAZ 1,2,3\n🟢 Zone Redundant"]:::zr
    PIPVPN2["Public IP — VPN #2\npip-vpng-*-02\nAZ 1,2,3\n🟢 Zone Redundant"]:::zr
    LGW["Local Network Gateway\nlgw-alz-riopel-*\n🟡 Regional"]:::reg
    CONN["VPN Connection\nvcn-azrhub-to-riop-*\nIPSec IKEv2\n🟡 Regional"]:::reg
    RT1["Route Table — FW Subnet\nrt-hubfwsubnet-*\n⬜ N/A"]:::na
    RT2["Route Table — Default Subnet\nrt-hubdsubnet-*\n⬜ N/A"]:::na
    RT3["Route Table — GW Subnet\nrt-hubgwsubnet-*\n⬜ N/A"]:::na
    RT4["Route Table — AGW\nrt-hubagw-*\n⬜ N/A"]:::na

    PIPFW --> AFW
    PIPVPN1 --> VPNG
    PIPVPN2 --> VPNG
    LGW --> CONN
    VPNG --> CONN
    FWPOL --> AFW
```

### DNS (Hub DNS RG)

```mermaid
graph LR
    classDef zr fill:#2e7d32,color:#fff,stroke:#1b5e20
    classDef reg fill:#f57f17,color:#fff,stroke:#e65100
    classDef glob fill:#6a1b9a,color:#fff,stroke:#4a148c

    DNSPR["Private DNS Resolver\ndnspr-alz-hubdns-*\n2 inbound + 2 outbound\n🟡 Regional"]:::reg
    DNSFR["DNS Forwarding Ruleset\ndnsprrl-alz-hub-dns-*\n20+ rules\n🟡 Regional"]:::reg

    PDZ1["privatelink.azure-api.net\n⚪ Global"]:::glob
    PDZ2["privatelink.openai.azure.com\n⚪ Global"]:::glob
    PDZ3["privatelink.cognitiveservices.azure.com\n⚪ Global"]:::glob
    PDZ4["privatelink.search.windows.net\n⚪ Global"]:::glob
    PDZ5["privatelink.vaultcore.azure.net\n⚪ Global"]:::glob
    PDZ6["privatelink.vault.azure.net\n⚪ Global"]:::glob
    PDZ7["azure-api.net\n⚪ Global"]:::glob
    PDZ8["scm.azure-api.net\n⚪ Global"]:::glob
    PDZ9["portal.azure-api.net\n⚪ Global"]:::glob
    PDZ10["developer.azure-api.net\n⚪ Global"]:::glob
    PDZ11["management.azure-api.net\n⚪ Global"]:::glob
    PDZ12["ivanhoecambridge.com\n⚪ Global"]:::glob

    DNSPR --> DNSFR
    DNSFR --> PDZ1 & PDZ2 & PDZ3 & PDZ4
    DNSFR --> PDZ5 & PDZ6 & PDZ7 & PDZ8
    DNSFR --> PDZ9 & PDZ10 & PDZ11 & PDZ12
```

### Application Gateway (Hub Gateway RG)

```mermaid
graph LR
    classDef zr fill:#2e7d32,color:#fff,stroke:#1b5e20
    classDef reg fill:#f57f17,color:#fff,stroke:#e65100
    classDef glob fill:#6a1b9a,color:#fff,stroke:#4a148c
    classDef na fill:#9e9e9e,color:#fff,stroke:#757575

    PIPAGW["Public IP — AGW\npip-agw-*\nStandard, DDoS enabled\n🟡 Regional"]:::reg
    NSG["NSG — AGW\nnsg-agw-*\n⬜ N/A"]:::na
    RT["Route Table — AGW\nrt-hubagw-*\n⬜ N/A"]:::na
    MI["Managed Identity\nmi-alz-hubagw-*\n⚪ Global"]:::glob
    WAFP["WAF Policy\nwaf-agw-alz-*\nOWASP 3.2 Detection\n⬜ N/A"]:::na
    AGW["Application Gateway\nagw-hub-*\nWAF_v2 · AZ 1,2,3\nAutoscale 1–2\n🟢 Zone Redundant"]:::zr

    PIPAGW --> AGW
    MI --> AGW
    WAFP --> AGW
    NSG -.-> AGW
    RT -.-> AGW
```

### Shared Services (Hub Shared RG)

```mermaid
graph LR
    classDef pha fill:#1565c0,color:#fff,stroke:#0d47a1

    KVHUB["Key Vault\nkv-alz-hub-*\nRBAC · Purge Protection\n🔵 Platform HA"]:::pha
    LAWHUB["Log Analytics Workspace\nlog-alz-hub-*\nPerGB2018 · 30d retention\n🔵 Platform HA"]:::pha

    LAWHUB -.-> KVHUB
```

### ICPL (Hub ICPL RG)

```mermaid
graph LR
    classDef reg fill:#f57f17,color:#fff,stroke:#e65100

    PEPSQL["Private Endpoint\npep-sqlmi-ic-dm-dev-*\nSQL Managed Instance\n🟡 Regional"]:::reg
```

---

## Shared Services Infrastructure

### Shared Base (Shared RG)

```mermaid
graph LR
    classDef pha fill:#1565c0,color:#fff,stroke:#0d47a1
    classDef glob fill:#6a1b9a,color:#fff,stroke:#4a148c

    KVSH["Key Vault\nkv-alz-shsvc-*\nRBAC · Private Endpoint\nPublic access Disabled\n🔵 Platform HA"]:::pha
    LAWSH["Log Analytics Workspace\nlog-alz-shsvc-*\nPerGB2018 · 30d retention\n🔵 Platform HA"]:::pha
    MISH["User Assigned Managed Identity\nmi-cdp-shsvc-*\n⚪ Global"]:::glob
```

### Shared Ext (Shared Ext RG)

```mermaid
graph LR
    classDef zr fill:#2e7d32,color:#fff,stroke:#1b5e20
    classDef reg fill:#f57f17,color:#fff,stroke:#e65100
    classDef glob fill:#6a1b9a,color:#fff,stroke:#4a148c
    classDef na fill:#9e9e9e,color:#fff,stroke:#757575

    SHVNET["Virtual Network\nvnet-cdp-alz-shsvc-*\n🟡 Regional"]:::reg
    NSGAPIM["NSG — APIM\nnsg-alz-apim-*\n⬜ N/A"]:::na
    RT["Route Table\nrt-shsvcdvtohub-*\n⬜ N/A"]:::na
    PIPAPIM["Public IP — APIM\npip-apim-alz-shsvc-*\nStandard · no AZ\n🟡 Regional"]:::reg
    APIM["API Management\napim-alz-shsvc-*\nPremium (prod) / Developer (dev)\nInternal VNet · Custom domain\n🟡 Regional"]:::reg
    ACR["Container Registry\ncralzpriv*\nBasic SKU\n🟡 Regional"]:::reg
    DC["Dev Center\ndc-shsvc-ado-*\nADE · 3 env types\n🟡 Regional"]:::reg
    APPI["Application Insights\nappi-alz-shsvc-*\nPrivate ingestion\n🟡 Regional"]:::reg
    AG["Action Group\nag-alz-shsvc-*\n⚪ Global"]:::glob

    SHVNET --> APIM
    NSGAPIM -.-> SHVNET
    RT -.-> SHVNET
    PIPAPIM --> APIM
    APIM --> APPI
    APIM --> AG
```

---

## Full Resiliency Summary

```mermaid
pie title Services by Resiliency Level
    "Zone Redundant 🟢" : 5
    "Platform HA 🔵" : 4
    "Regional 🟡" : 18
    "Global ⚪" : 14
    "N/A ⬜" : 8
```

---

## Complete Service Inventory

| # | Service | Resource Name Pattern | SKU / Tier | Resiliency Level | Notes |
|---|---|---|---|---|---|
| 1 | Virtual Network (Hub) | `vnet-cdp-alz-hub-*` | Standard | 🟡 Regional | Hub VNet, no AZ config |
| 2 | Azure Firewall | `afw-alz-hubfw-*` | Premium / Basic | 🟡 Regional | No `zones` configured |
| 3 | Firewall Policy | `afwp-afw-alz-hubfw-*` | Standard / Basic | ⬜ N/A | Config resource |
| 4 | VPN Gateway | `vpng-alz-hub-*` | **VpnGw2AZ** | 🟢 Zone Redundant | Active-Active BGP, Gen2 |
| 5 | Public IP — Firewall | `pip-afw-*` | Standard | 🟢 Zone Redundant | Zones 1, 2, 3 |
| 6 | Public IP — VPN #1 | `pip-vpng-*-01` | Standard | 🟢 Zone Redundant | Zones 1, 2, 3 |
| 7 | Public IP — VPN #2 | `pip-vpng-*-02` | Standard | 🟢 Zone Redundant | Zones 1, 2, 3 |
| 8 | Local Network Gateway | `lgw-alz-riopel-*` | — | 🟡 Regional | On-prem gateway reference |
| 9 | VPN Connection | `vcn-azrhub-to-riop-*` | IPSec / IKEv2 | 🟡 Regional | Inherits VPN GW HA |
| 10 | Route Table — FW Subnet | `rt-hubfwsubnet-*` | — | ⬜ N/A | Non-stateful config |
| 11 | Route Table — Default Subnet | `rt-hubdsubnet-*` | — | ⬜ N/A | Non-stateful config |
| 12 | Route Table — GW Subnet | `rt-hubgwsubnet-*` | — | ⬜ N/A | Non-stateful config |
| 13 | Route Table — AGW | `rt-hubagw-*` | — | ⬜ N/A | Non-stateful config |
| 14 | Private DNS Resolver | `dnspr-alz-hubdns-*` | — | 🟡 Regional | 2 inbound + 2 outbound endpoints |
| 15 | DNS Forwarding Ruleset | `dnsprrl-alz-hub-dns-*` | — | 🟡 Regional | 20+ forwarding rules |
| 16 | Private DNS Zone — azure-api.net | `azure-api.net` | Global | ⚪ Global | Hub VNet linked |
| 17 | Private DNS Zone — scm.azure-api.net | `scm.azure-api.net` | Global | ⚪ Global | |
| 18 | Private DNS Zone — portal.azure-api.net | `portal.azure-api.net` | Global | ⚪ Global | |
| 19 | Private DNS Zone — developer.azure-api.net | `developer.azure-api.net` | Global | ⚪ Global | |
| 20 | Private DNS Zone — management.azure-api.net | `management.azure-api.net` | Global | ⚪ Global | |
| 21 | Private DNS Zone — privatelink.azure-api.net | `privatelink.azure-api.net` | Global | ⚪ Global | |
| 22 | Private DNS Zone — openai | `privatelink.openai.azure.com` | Global | ⚪ Global | |
| 23 | Private DNS Zone — cognitive services | `privatelink.cognitiveservices.azure.com` | Global | ⚪ Global | |
| 24 | Private DNS Zone — search | `privatelink.search.windows.net` | Global | ⚪ Global | |
| 25 | Private DNS Zone — vault.azure.net | `privatelink.vault.azure.net` | Global | ⚪ Global | |
| 26 | Private DNS Zone — vaultcore | `privatelink.vaultcore.azure.net` | Global | ⚪ Global | |
| 27 | Private DNS Zone — ivanhoecambridge.com | `ivanhoecambridge.com` | Global | ⚪ Global | |
| 28 | Application Gateway | `agw-hub-*` | WAF_v2 | 🟢 Zone Redundant | AZ 1,2,3 · autoscale 1–2 |
| 29 | WAF Policy | `waf-agw-alz-*` | OWASP 3.2 | ⬜ N/A | Config resource |
| 30 | Public IP — AGW | `pip-agw-*` | Standard | 🟡 Regional | DDoS enabled, no zones |
| 31 | Managed Identity — AGW | `mi-alz-hubagw-*` | User Assigned | ⚪ Global | Platform-managed |
| 32 | NSG — AGW | `nsg-agw-*` | — | ⬜ N/A | Non-stateful config |
| 33 | Key Vault (Hub) | `kv-alz-hub-*` | Standard | 🔵 Platform HA | RBAC · Purge Protection |
| 34 | Log Analytics Workspace (Hub) | `log-alz-hub-*` | PerGB2018 | 🔵 Platform HA | Microsoft-managed AZ |
| 35 | Private Endpoint — SQL MI | `pep-sqlmi-ic-dm-dev-*` | — | 🟡 Regional | ICPL cross-subscription |
| 36 | Key Vault (Shared Services) | `kv-alz-shsvc-*` | Standard | 🔵 Platform HA | Private endpoint · public disabled |
| 37 | Log Analytics Workspace (Shared) | `log-alz-shsvc-*` | PerGB2018 | 🔵 Platform HA | Microsoft-managed AZ |
| 38 | Managed Identity (Shared) | `mi-cdp-shsvc-*` | User Assigned | ⚪ Global | Used by APIM + DevCenter |
| 39 | Virtual Network (Shared Services) | `vnet-cdp-alz-shsvc-*` | Standard | 🟡 Regional | Peered to Hub VNet |
| 40 | NSG — APIM | `nsg-alz-apim-*` | — | ⬜ N/A | Non-stateful config |
| 41 | Route Table (Shared) | `rt-shsvcdvtohub-*` | — | ⬜ N/A | Non-stateful config |
| 42 | Public IP — APIM | `apim-alz-shsvc-*-public-ip` | Standard | 🟡 Regional | No zones configured |
| 43 | API Management | `apim-alz-shsvc-*` | Premium / Developer | 🟡 Regional | Internal VNet · custom domain from KV |
| 44 | Container Registry | `cralzpriv*` | Basic | 🟡 Regional | No geo-replication |
| 45 | Dev Center | `dc-shsvc-ado-*` | — | 🟡 Regional | ADE, 3 env types (dev/test/prod) |
| 46 | Application Insights | `appi-alz-shsvc-*` | Web | 🟡 Regional | Private ingestion/query |
| 47 | Action Group | `ag-alz-shsvc-*` | Global | ⚪ Global | APIM alert notifications |

---

## Resiliency Gaps & Recommendations

```mermaid
graph TD
    subgraph GAPS["⚠️ Resiliency Gaps"]
        G1["Azure Firewall\nNo AZ zones configured\n→ Add zones 1,2,3"]
        G2["Public IP — AGW\nNo AZ zones configured\n→ Add availabilityZones 1,2,3"]
        G3["Public IP — APIM\nNo AZ zones configured\n→ Add availabilityZones 1,2,3"]
        G4["API Management Premium\nNo zones configured\n→ Add zones for AZ deployment"]
        G5["Container Registry — Basic\nNo geo-replication\n→ Upgrade to Standard/Premium for zone redundancy"]
    end

    subgraph STRENGTHS["✅ Resiliency Strengths"]
        S1["VPN Gateway VpnGw2AZ\nActive-Active BGP\nFully zone redundant"]
        S2["Application Gateway WAF_v2\nZones 1,2,3 + autoscale\nFully zone redundant"]
        S3["Key Vaults (Hub & Shared)\nPlatform HA + Purge Protection\nPrivate endpoint on shared KV"]
        S4["Private DNS Zones ×12\nGlobal distribution\nMicrosoft-managed HA"]
        S5["DNS Resolver\nDual inbound + outbound endpoints\nProvides basic redundancy"]
    end
```

| Priority | Service | Gap | Recommended Action |
|---|---|---|---|
| High | Azure Firewall | No AZ zones → single-AZ failure risk | Add `zones: [1, 2, 3]` to `azFirewall` module |
| High | API Management Premium | No AZ zones → single-AZ failure risk | Add `zones: [1, 2, 3]` to `apimAVMService` module |
| Medium | Public IP — AGW | No AZ zones | Add `availabilityZones: [1, 2, 3]` to `publicIpAddress` module |
| Medium | Public IP — APIM | No AZ zones | Add `availabilityZones: [1, 2, 3]` to `apimPublicIP` module |
| Low | Container Registry Basic | No zone redundancy, no geo-replication | Upgrade to Standard or Premium SKU |
