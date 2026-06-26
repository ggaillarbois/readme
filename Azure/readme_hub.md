# Azure Landing Zone Infrastructure

This repository contains the infrastructure-as-code (IaC) for deploying an Azure Landing Zone using Bicep templates. The project implements a hub-spoke network topology with shared services, API Management, and Application Gateway infrastructure.

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Project Structure](#project-structure)
- [Created Resources](#created-resources)
- [Deployment Flow](#deployment-flow)
- [Getting Started](#getting-started)
- [Module Usage](#module-usage)

## Architecture Overview

This solution implements an Azure Landing Zone with the following components (shown for production environment in Canada Central):

```mermaid
graph TB
    subgraph "Hub Subscription: cdp-alz-hub-pr"
        subgraph "RG: rg-alz-hub-pr-01-c-cnc"
            VNET[vnet-cdp-alz-hub-pr-01<br/>Hub Virtual Network]
            FW[afw-alz-hubfw-pr-01-c-cnc<br/>Azure Firewall]
            FWPOL[afwp-afw-alz-hubfw-pr-01-c-cnc<br/>Firewall Policy]
            VPN[vpng-alz-hub-pr-01-c-cnc<br/>VPN Gateway]
            LNG[lgw-alz-riopel-pr-01-c-cnc<br/>Local Network Gateway]

            VNET --> FW
            FW --> FWPOL
            VNET --> VPN
            VPN --> LNG
        end

        subgraph "RG: rg-alz-hubshsvc-pr-01-c-cnc"
            LAW[log-alz-hub-pr-01-c-cnc<br/>Log Analytics]
            KVHUB[kv-alz-hub-pr-01-c-cnc<br/>Key Vault]
        end

        subgraph "RG: rg-alz-hubdns-pr-01-c-cnc"
            DNSRES[dnspr-alz-hubdns-pr-01-c-cnc<br/>DNS Resolver]
            DNSFW[dnsprrl-alz-hub-dns-01-pr-c-cnc<br/>DNS Forwarding Ruleset]
            PDZ1[privatelink.openai.azure.com<br/>Private DNS Zone]
            PDZ2[privatelink.vaultcore.azure.net<br/>Private DNS Zone]
            PDZ3[cdpq.cloud<br/>Private DNS Zone]

            DNSRES --> DNSFW
            VNET -.link.- PDZ1
            VNET -.link.- PDZ2
            VNET -.link.- PDZ3
        end

        subgraph "RG: rg-alz-hubagw-pr-01-c-cnc"
            AGW[agw-hub-pr-01-c-cnc<br/>Application Gateway]
            WAFPOL[waf-agw-alz-pr-01-c-cnc<br/>WAF Policy]
            PIPAGW[pip-agw-alz-pr-01-c-cnc<br/>Public IP]
            MIAGW[mi-alz-hubagw-pr-01-c-cnc<br/>Managed Identity]

            AGW --> WAFPOL
            AGW --> PIPAGW
            AGW -.uses.- MIAGW
        end

        subgraph "RG: rg-alz-hubicpl-pr-01-c-cnc"
            ICPL[Interconnect/<br/>Private Link Resources]
        end
    end

    subgraph "Shared Subscription: cdp-alz-shsvc-pr"
        subgraph "RG: rg-alz-shsvc-pr-01-c-cnc"
            SHSVC[Shared Services<br/>Resources]
        end

        subgraph "RG: rg-alz-shsvc-shsvc-pr-01-c-cnc"
            APIM[apim-alz-shsvc-pr-01-c-cnc<br/>API Management<br/>apim-alz-shsvc-pr-01-c-cnc.cdpq.cloud]
            KVAPIM[kv-alz-shsvc-pr-01-c-cnc<br/>Key Vault]
            MIAPIM[mi-cdp-shsvc-apim-pr-01-c-cnc<br/>Managed Identity]
            PIPIPM[pip-apim-alz-shsvc-pr-01-c-cnc<br/>Public IP]

            APIM --> KVAPIM
            APIM -.uses.- MIAPIM
            APIM --> PIPIPM
        end

        subgraph "RG: rg-shsvc-amacat-pr-01-c-cnc"
            AMACAT[Azure Managed<br/>Application Catalog]
        end
    end

    subgraph "Backend Services"
        OPENAI[OpenAI Service]
        DOCINT[Document Intelligence]
    end

    AGW -->|Backend Pool| APIM
    APIM -->|API Proxy| OPENAI
    APIM -->|API Proxy| DOCINT
    FW -->|Route Traffic| AGW

    style VNET fill:#0078D4,color:#fff
    style FW fill:#FF6C00,color:#fff
    style VPN fill:#7FBA00,color:#fff
    style AGW fill:#008272,color:#fff
    style APIM fill:#0089D6,color:#fff
    style DNSRES fill:#50E6FF,color:#000
    style LAW fill:#FCD116,color:#000
```

### Key Components by Subscription

#### Hub Subscription (`cdp-alz-hub-pr`)

**Networking (rg-alz-hub-pr-01-c-cnc)**
- `vnet-cdp-alz-hub-pr-01` - Hub Virtual Network
- `afw-alz-hubfw-pr-01-c-cnc` - Azure Firewall with policy `afwp-afw-alz-hubfw-pr-01-c-cnc`
- `vpng-alz-hub-pr-01-c-cnc` - VPN Gateway for on-premises connectivity
- `lgw-alz-riopel-pr-01-c-cnc` - Local Network Gateway
- Route Tables: `rt-hubfwsubnet-pr-01`, `rt-hubdsubnet-pr-01`, `rt-hubgwsubnet-pr-01`

**Subnets**
- `AzureFirewallSubnet` - Azure Firewall subnet (10.225.2.0/26 for CDPQ)
- `GatewaySubnet` - VPN Gateway subnet (10.225.2.64/27 for CDPQ)
- `snet-alz-dnspr-ie-pr-01-cnc` - DNS Resolver Inbound Endpoint 01 (10.225.2.128/28)
- `snet-alz-dnspr-oe-pr-01-cnc` - DNS Resolver Outbound Endpoint 01 (10.225.2.144/28)
- `snet-alz-dnspr-ie-pr-02-cnc` - DNS Resolver Inbound Endpoint 02 (10.225.2.160/28)
- `snet-alz-dnspr-oe-pr-02-cnc` - DNS Resolver Outbound Endpoint 02 (10.225.2.176/28)
- `snet-alz-agw-oe-pr-01-cnc` - Application Gateway subnet (10.225.3.0/24)

**Monitoring & Security (rg-alz-hubshsvc-pr-01-c-cnc)**
- `log-alz-hub-pr-01-c-cnc` - Log Analytics Workspace
- `kv-alz-hub-pr-01-c-cnc` - Key Vault for certificates and secrets

**DNS Services (rg-alz-hubdns-pr-01-c-cnc)**
- `dnspr-alz-hubdns-pr-01-c-cnc` - Azure DNS Private Resolver
- `dnsprrl-alz-hub-dns-01-pr-c-cnc` - DNS Forwarding Ruleset
- Private DNS Zones:
  - `privatelink.openai.azure.com`
  - `privatelink.cognitiveservices.azure.com`
  - `privatelink.vaultcore.azure.net`
  - `privatelink.azure-api.net`
  - `cdpq.cloud` / `cdpqdev.com` (based on tenant)

**Application Gateway (rg-alz-hubagw-pr-01-c-cnc)**
- `agw-hub-pr-01-c-cnc` - Application Gateway (routes to APIM)
- `waf-agw-alz-pr-01-c-cnc` - Web Application Firewall Policy
- `pip-agw-alz-pr-01-c-cnc` - Public IP Address
- `mi-alz-hubagw-pr-01-c-cnc` - Managed Identity (for Key Vault access)
- `nsg-agw-pr-01` - Network Security Group

**Interconnect (rg-alz-hubicpl-pr-01-c-cnc)**
- Private Link and interconnect resources

#### Shared Services Subscription (`cdp-alz-shsvc-pr`)

**Main Services (rg-alz-shsvc-pr-01-c-cnc)**
- Shared infrastructure resources

**APIM Services (rg-alz-shsvc-shsvc-pr-01-c-cnc)**
- `apim-alz-shsvc-pr-01-c-cnc` - API Management instance
  - Custom domain: `apim-alz-shsvc-pr-01-c-cnc.cdpq.cloud`
  - SKU: Premium (production) / Developer (non-prod)
  - VNet Mode: Internal
- `kv-alz-shsvc-pr-01-c-cnc` - Key Vault for APIM certificates
- `mi-cdp-shsvc-apim-pr-01-c-cnc` - Managed Identity for APIM
- `pip-apim-alz-shsvc-pr-01-c-cnc` - Public IP Address

**API Backends**
- OpenAI Service endpoints
- Document Intelligence endpoints

**Managed App Catalog (rg-shsvc-amacat-pr-01-c-cnc)**
- Azure Managed Application catalog resources

### Network Flow

```mermaid
sequenceDiagram
    participant User as External User
    participant PIP as pip-agw-alz-pr-01-c-cnc
    participant AGW as agw-hub-pr-01-c-cnc
    participant FW as afw-alz-hubfw-pr-01-c-cnc
    participant APIM as apim-alz-shsvc-pr-01-c-cnc
    participant DNS as dnspr-alz-hubdns-pr-01-c-cnc
    participant AI as OpenAI/DocInt

    User->>PIP: HTTPS Request (443)
    PIP->>AGW: Forward to App Gateway
    AGW->>AGW: WAF Policy Check
    AGW->>DNS: Resolve APIM internal address
    DNS-->>AGW: Private IP
    AGW->>FW: Route through Firewall
    FW->>APIM: Forward to APIM (Internal VNet)
    APIM->>AI: API Call to Backend
    AI-->>APIM: Response
    APIM-->>AGW: Response
    AGW-->>User: HTTPS Response
```

### Resource Dependencies

```mermaid
graph LR
    subgraph "Deployment Order"
        S1[1. Subscriptions<br/>sub-hub.bicep<br/>sub-shsvc.bicep]
        S2[2. Resource Groups<br/>sub-hub-rg.bicep<br/>sub-shsvc-rg.bicep]
        S3[3. Networking<br/>vnet-cdp-alz-hub-pr-01]
        S4[4. DNS Resolver<br/>dnspr-alz-hubdns-pr-01-c-cnc]
        S5[5. Key Vaults<br/>kv-alz-hub-pr-01-c-cnc<br/>kv-alz-shsvc-pr-01-c-cnc]
        S6[6. Azure Firewall<br/>afw-alz-hubfw-pr-01-c-cnc]
        S7[7. APIM<br/>apim-alz-shsvc-pr-01-c-cnc]
        S8[8. App Gateway<br/>agw-hub-pr-01-c-cnc]

        S1 --> S2
        S2 --> S3
        S3 --> S4
        S3 --> S5
        S3 --> S6
        S5 --> S7
        S7 --> S8
    end

    style S1 fill:#e1f5ff
    style S2 fill:#ffe1e1
    style S3 fill:#e1ffe1
    style S4 fill:#fff5e1
    style S5 fill:#f5e1ff
    style S6 fill:#ffe1cc
    style S7 fill:#e6ccff
    style S8 fill:#ccffeb
```

## Project Structure

```
alz-landingzone/
├── bicepconfig.json                 # Bicep configuration
├── README.md                        # This file
├── fichiers-bicep/                  # Bicep templates
│   ├── apim-infra/                 # API Management infrastructure
│   │   ├── apim.bicep
│   │   └── apim.azcli
│   ├── app-gateway-infra/          # Application Gateway infrastructure
│   │   ├── app-gateway-infra.bicep
│   │   └── app-gateway-infra.bicepparam
│   ├── landing-hub-infra/          # Hub infrastructure
│   │   ├── sub-hub.bicep           # Hub subscription creation
│   │   ├── sub-hub-rg.bicep        # Hub resource groups
│   │   ├── rg-hub-main-components.bicep
│   │   ├── rg-hub-dns-components.bicep
│   │   ├── rg-hub-gateway-components.bicep
│   │   ├── rg-hub-shsvc-components.bicep
│   │   ├── rg-hub-icpl-components.bicep
│   │   ├── add-forwarding-rule.bicep
│   │   ├── dep-hub-gateway-subnet.bicep
│   │   ├── dep-hub-icpl-arecords.bicep
│   │   ├── dep-hub-shsvc-keyvault.bicep
│   │   ├── dep-hub-sqlmi-icpl-arecords.bicep
│   │   └── prerequis/              # Prerequisites (resource provider registration)
│   └── landing-shared-infra/       # Shared services infrastructure
│       ├── sub-shsvc.bicep         # Shared subscription creation
│       ├── sub-shsvc-rg.bicep      # Shared resource groups
│       ├── rg-shsvc-components.bicep
│       ├── rg-shsvc-shsvc-components.bicep
│       ├── dep-shsvc-shsvc-devCenter.bicep
│       └── dep-shsvc-shsvc-keyvault.bicep
└── pipelines/                       # Azure DevOps pipelines
    ├── main.yaml
    ├── deploy-env-job.yaml
    ├── get-secure-file.yaml
    └── variables/
```

## Created Resources

### Resource Naming Convention

Resources follow the naming pattern: `{prefix}-{workload}-{component}-{env}-{instance}-{org}-{location}`

- **{prefix}**: Resource type (rg, vnet, kv, agw, apim, etc.)
- **{workload}**: Workload identifier (alz, shsvc, hub, etc.)
- **{component}**: Component name (hub, dns, gateway, etc.)
- **{env}**: Environment (pr=production, dev=development)
- **{instance}**: Instance number (01, 02, etc.)
- **{org}**: Organization code (c=CDPQ, d=dev)
- **{location}**: Location short code (cnc=Canada Central, cne=Canada East)

**Examples:**
- `rg-alz-hub-pr-01-c-cnc` - Hub resource group for production in Canada Central
- `vnet-cdp-alz-hub-pr-01` - Hub virtual network for production
- `agw-hub-pr-01-c-cnc` - Application Gateway for production
- `apim-alz-shsvc-pr-01-c-cnc` - API Management for shared services in production

### Resource Summary by Type

#### Subscriptions
- **Hub**: `cdp-alz-hub-{env}` - Contains networking, security, and gateway resources
- **Shared Services**: `cdp-alz-shsvc-{env}` - Contains shared application services

#### Resource Groups (Total: 8)

**Hub Subscription (5 groups):**
1. `rg-alz-hub-{env}-01-{org}-{location}` - Core hub networking (VNet, Firewall, VPN)
2. `rg-alz-hubshsvc-{env}-01-{org}-{location}` - Hub monitoring & security (Log Analytics, Key Vault)
3. `rg-alz-hubdns-{env}-01-{org}-{location}` - DNS services (DNS Resolver, Private DNS Zones)
4. `rg-alz-hubagw-{env}-01-{org}-{location}` - Application Gateway & WAF
5. `rg-alz-hubicpl-{env}-01-{org}-{location}` - Interconnect & Private Link

**Shared Services Subscription (3 groups):**
1. `rg-alz-shsvc-{env}-01-{org}-{location}` - Main shared services
2. `rg-alz-shsvc-shsvc-{env}-01-{org}-{location}` - APIM & API services
3. `rg-shsvc-amacat-{env}-01-{org}-{location}` - Azure Managed Application catalog

For detailed component lists, see [Key Components by Subscription](#key-components-by-subscription) section above.

## Deployment Flow

The deployment follows this orchestrated sequence:

```mermaid
sequenceDiagram
    participant Pipeline as Azure DevOps Pipeline
    participant MG as Management Group
    participant HubSub as Hub Subscription<br/>(cdp-alz-hub-pr)
    participant SharedSub as Shared Subscription<br/>(cdp-alz-shsvc-pr)

    Pipeline->>MG: 1. Create Subscriptions
    MG->>HubSub: Create cdp-alz-hub-pr
    MG->>SharedSub: Create cdp-alz-shsvc-pr

    Pipeline->>HubSub: 2. Create Hub Resource Groups
    HubSub->>HubSub: rg-alz-hub-pr-01-c-cnc
    HubSub->>HubSub: rg-alz-hubshsvc-pr-01-c-cnc
    HubSub->>HubSub: rg-alz-hubdns-pr-01-c-cnc
    HubSub->>HubSub: rg-alz-hubagw-pr-01-c-cnc
    HubSub->>HubSub: rg-alz-hubicpl-pr-01-c-cnc

    Pipeline->>SharedSub: 3. Create Shared Resource Groups
    SharedSub->>SharedSub: rg-alz-shsvc-pr-01-c-cnc
    SharedSub->>SharedSub: rg-alz-shsvc-shsvc-pr-01-c-cnc
    SharedSub->>SharedSub: rg-shsvc-amacat-pr-01-c-cnc

    Pipeline->>HubSub: 4. Deploy Hub Networking
    HubSub->>HubSub: vnet-cdp-alz-hub-pr-01
    HubSub->>HubSub: afw-alz-hubfw-pr-01-c-cnc
    HubSub->>HubSub: vpng-alz-hub-pr-01-c-cnc

    Pipeline->>HubSub: 5. Deploy DNS Infrastructure
    HubSub->>HubSub: dnspr-alz-hubdns-pr-01-c-cnc
    HubSub->>HubSub: dnsprrl-alz-hub-dns-01-pr-c-cnc
    HubSub->>HubSub: Private DNS Zones

    Pipeline->>HubSub: 6. Deploy Key Vaults
    HubSub->>HubSub: kv-alz-hub-pr-01-c-cnc

    Pipeline->>SharedSub: 7. Deploy Shared Key Vault
    SharedSub->>SharedSub: kv-alz-shsvc-pr-01-c-cnc

    Pipeline->>SharedSub: 8. Deploy APIM
    SharedSub->>SharedSub: apim-alz-shsvc-pr-01-c-cnc
    SharedSub->>SharedSub: Configure OpenAI Backend
    SharedSub->>SharedSub: Configure DocInt Backend

    Pipeline->>HubSub: 9. Deploy Application Gateway
    HubSub->>HubSub: agw-hub-pr-01-c-cnc
    HubSub->>SharedSub: Link to APIM Backend
```

## Getting Started

### Prerequisites

1. **Azure CLI** - [Install Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli)
2. **Bicep CLI** - Installed with Azure CLI
3. **Azure Subscription** - With appropriate permissions
4. **Azure DevOps** - For pipeline execution

### Installation

1. Clone the repository:
   ```bash
   git clone <repository-url>
   cd alz-landingzone
   ```

2. Configure Bicep (if needed):
   ```bash
   az bicep install
   az bicep upgrade
   ```

3. Login to Azure:
   ```bash
   az login
   az account set --subscription <subscription-id>
   ```

### Manual Deployment

To deploy individual components manually:

#### Deploy Hub Infrastructure
```bash
cd fichiers-bicep/landing-hub-infra
az deployment mg create \
  --management-group-id <mg-id> \
  --location canadacentral \
  --template-file sub-hub.bicep \
  --parameters sub-hub.bicepparam
```

#### Deploy Shared Infrastructure
```bash
cd fichiers-bicep/landing-shared-infra
az deployment mg create \
  --management-group-id <mg-id> \
  --location canadacentral \
  --template-file sub-shsvc.bicep \
  --parameters sub-shsvc.bicepparam
```

#### Deploy Application Gateway
```bash
cd fichiers-bicep/app-gateway-infra
az deployment sub create \
  --location canadacentral \
  --template-file app-gateway-infra.bicep \
  --parameters app-gateway-infra.bicepparam
```

### Pipeline Deployment

The repository includes Azure DevOps pipelines for automated deployment:

1. Configure pipeline variables in `pipelines/variables/`
2. Run the main pipeline: `pipelines/main.yaml`
3. Monitor deployment progress in Azure DevOps

## Module Usage

This project leverages **Azure Verified Modules (AVM)** from the public Bicep registry for standardized, best-practice resource deployments:

```mermaid
graph LR
    A[Local Bicep Files] --> B[AVM Module Registry<br/>br/public:avm/]
    B --> C[Resource Groups<br/>res/resources/resource-group:0.4.2]
    B --> D[Role Assignments<br/>ptn/authorization/role-assignment:0.2.2]
    B --> E[Subscription Vending<br/>ptn/lz/sub-vending:0.3.7]

    style B fill:#4CAF50
    style C fill:#2196F3
    style D fill:#2196F3
    style E fill:#2196F3
```

### Key AVM Modules Used

- `br/public:avm/res/resources/resource-group:0.4.2` - Resource group creation
- `br/public:avm/ptn/authorization/role-assignment:0.2.2` - RBAC assignments
- `br/public:avm/ptn/lz/sub-vending:0.3.7` - Subscription creation and management

## Configuration

### Parameter Files

Parameter files (`.bicepparam`) contain environment-specific configuration:

- `sub-hub.bicepparam` - Hub subscription parameters
- `sub-shsvc.bicepparam` - Shared services parameters
- `app-gateway-infra.bicepparam` - Application Gateway parameters

### Bicep Configuration

The `bicepconfig.json` file configures:
- Module aliases
- Linter rules
- Formatting preferences

## Key Features

- ✅ Hub-spoke network topology
- ✅ Centralized DNS management
- ✅ API Management with OpenAI and Document Intelligence backends
- ✅ Application Gateway with WAF
- ✅ Infrastructure as Code using Bicep
- ✅ Azure Verified Modules for standardization
- ✅ CI/CD pipeline automation
- ✅ Multi-environment support (dev, prod)

## Troubleshooting

### Common Issues

1. **Module not found**: Ensure Bicep CLI is up to date
   ```bash
   az bicep upgrade
   ```

2. **Insufficient permissions**: Verify RBAC assignments at management group level

3. **Resource provider not registered**: Run prerequisite scripts in `landing-hub-infra/prerequis/`

## Contributing

1. Create a feature branch
2. Make changes following the existing naming conventions
3. Test deployments in a non-production environment
4. Submit a pull request with detailed description

## License

[Specify license information]

## Support

For issues or questions, contact the infrastructure team.