# Azure Landing Zone - Essential Design

## Overview

Standardized Azure Landing Zone template implementing CAF/WAF principles using Azure Verified Modules. Designed for consistent deployment across customer environments with minimal customization.

**Core Principles:**
- Security-first design with zero-trust networking
- Complete automation using AVM modules
- CAF-aligned management group structure
- Standardized hub-spoke networking with firewall options
- Built-in compliance and monitoring

---

## Management Group Structure

```
<customer-prefix> (Tenant Root)
├── Platform
│   ├── Platform-Management      # Monitoring, backup, automation
│   ├── Platform-Connectivity   # Hub networking, firewalls
│   └── Platform-Identity       # Domain controllers, ADFS
├── Landing-Zones
│   ├── Landing-Zones-Corp      # Internal workloads
│   └── Landing-Zones-Online    # External-facing workloads
├── Sandbox                     # Dev/test environments
└── Decommissioned             # Cleanup staging
```

**Policy Assignment Strategy:**
- **Root Level:** Global governance (tagging, naming, compliance)
- **Platform:** Platform-specific security and operational policies
- **Landing Zones:** Workload security policies (Corp vs Online have different requirements)
- **Sandbox:** Relaxed policies with cost controls

---

## Subscription Design

### Platform Subscriptions (4 Total)

**Connectivity (`<customer>-connectivity-<region>`):**
- Hub VNet with Azure Firewall
- ExpressRoute/VPN gateways
- Azure Bastion, Private DNS zones
- Third-party NVA (optional)

**Management (`<customer>-management-<region>`):**
- Log Analytics workspace
- Azure Monitor, Automation account
- Backup (Recovery Services Vault)
- Azure Sentinel (optional)

**Identity (`<customer>-identity-<region>`):**
- Domain Controllers
- Certificate Authority
- Azure AD Connect VM

**Security (`<customer>-security-<region>`):**
- Microsoft Defender for Cloud
- Platform Key Vaults
- Security automation tools

### Landing Zone Subscriptions (Templates)

**Corp Workloads:** `<customer>-corp-<workload>-<env>`
**Online Workloads:** `<customer>-online-<workload>-<env>`

---

## Network Architecture

### IP Address Allocation
```
Hub Network:           10.0.0.0/16
Corp Spoke Networks:   10.1.0.0/12  (10.1.0.0 - 10.15.255.255)
Online Spoke Networks: 10.16.0.0/12 (10.16.0.0 - 10.31.255.255)
Sandbox Networks:      10.32.0.0/12 (10.32.0.0 - 10.47.255.255)
```

### Hub Virtual Network (10.0.0.0/16)
```
GatewaySubnet:         10.0.0.0/24  (ExpressRoute/VPN)
AzureFirewallSubnet:   10.0.1.0/24  
AzureBastionSubnet:    10.0.2.0/24
SharedServicesSubnet:  10.0.3.0/24
ManagementSubnet:      10.0.4.0/24
NVASubnet:             10.0.5.0/24  (Third-party NVA)
```

### Spoke Network Templates
```
Corp Spoke (10.1.x.0/16):
├── WebTier:    10.1.1.0/24
├── AppTier:    10.1.2.0/24
├── DataTier:   10.1.3.0/24
└── Management: 10.1.4.0/24

Online Spoke (10.2.x.0/16):
├── DMZ:        10.2.1.0/24
├── WebTier:    10.2.2.0/24
├── AppTier:    10.2.3.0/24
└── DataTier:   10.2.4.0/24
```

### Firewall Options

**Option A: Azure Firewall Only**
- L4 filtering and threat intelligence
- Application rules for FQDN filtering
- Network rules for hub-spoke communication

**Option B: Azure Firewall + Third-Party NVA**
- Azure Firewall: Basic filtering and threat intel
- NVA: Advanced inspection (Fortinet, Palo Alto, Cisco)
- Layered security approach

**Option C: Third-Party NVA Only**
- NVA handles all filtering and inspection
- Azure Firewall bypassed
- Customer preference for specific vendor

---

## Identity and Security

### Entra ID Configuration
- **Tenant:** `<customer>.onmicrosoft.com`
- **Custom Domain:** Customer's verified domain
- **Licensing:** Azure AD Premium P2 (for PIM)
- **Hybrid Identity:** Azure AD Connect with password hash sync

### Privileged Identity Management (PIM)
```yaml
Azure AD Roles (PIM Enabled):
  - Global Administrator
  - Security Administrator
  - Privileged Role Administrator

Azure Resource Roles (PIM Enabled):
  - Owner (Subscription level)
  - Contributor (Resource Group level)
  - User Access Administrator

PIM Settings:
  - MFA: Required
  - Justification: Required
  - Approval: Required for high-privilege roles
  - Max activation: 8 hours
```

### Conditional Access Policies
```yaml
Baseline Policies:
  - Block legacy authentication
  - Require MFA for Azure management
  - Block high-risk countries
  - Require compliant device for Office 365

Risk-Based Policies:
  - High user risk: Block until password change
  - High sign-in risk: Require MFA
```

### Key Vault Strategy
- **Platform Key Vaults:** One per platform subscription
- **Workload Key Vaults:** One per landing zone subscription
- **Configuration:** Private endpoints, RBAC access, soft delete enabled

---

## Security Baseline

### Microsoft Defender for Cloud
- **All Plans Enabled:** Servers, App Service, Databases, Storage, Key Vault
- **Integration:** Log Analytics workspace, automated responses
- **Compliance:** Azure Security Benchmark enabled

### Azure Policy Framework
```yaml
Security Initiative:
  - Require encryption at rest and in transit
  - Block public storage access
  - Require backup for VMs
  - Block RDP/SSH from internet
  - Require NSGs on subnets

Network Initiative:
  - Require Azure Firewall in hub
  - Enforce UDRs on spoke subnets
  - Require private endpoints for PaaS

Operational Initiative:
  - Enforce resource tagging
  - Require naming conventions
  - Block expensive SKUs in dev/test
```

---

## Monitoring and Operations

### Azure Monitor Setup
```yaml
Log Analytics Workspace:
  - Location: Primary region
  - Retention: 730 days
  - Daily cap: 50GB (adjustable)

Data Collection:
  - Azure Activity Logs (all subscriptions)
  - Resource diagnostic logs
  - Security logs (Defender, Key Vault, Firewall)
  - Performance metrics (VMs, databases)

Alerts:
  - Security: Failed auth, privilege escalation
  - Performance: CPU >90%, Memory >85%
  - Network: ExpressRoute down, Firewall health
  - Cost: Budget thresholds (50%, 80%, 100%)
```

### Automation
```yaml
Azure Automation Account:
  - VM start/stop schedules
  - Update management (monthly patching)
  - Backup verification
  - Cost optimization tasks
  - Security compliance checks
```

---

## Backup and Disaster Recovery

### Backup Strategy
```yaml
Recovery Services Vault:
  - Geo-redundant storage
  - Cross-region restore enabled

Backup Policies:
  - VMs: Daily backup, 30-day retention
  - Databases: Transaction log every 15 min
  - File shares: Daily backup

Scope:
  - Platform VMs: All included
  - Landing zone VMs: All production
  - Application data: Per workload requirements
```

### Disaster Recovery
- **Primary → Secondary region replication**
- **RTO Target:** 4 hours (platform), 8 hours (workloads)
- **RPO Target:** 15 minutes
- **Testing:** Quarterly for platform, bi-annually for workloads

---

## AVM Module Strategy

### Core AVM Modules

**Pattern Modules:**
- **Landing Zone Pattern:** Management groups, policies, RBAC baseline
- **Hub-Spoke Network Pattern:** Complete network topology
- **AKS Production Pattern:** For container workloads

**Resource Modules:**
- **Virtual Network:** Hub and spoke networks
- **Azure Firewall:** Network security
- **Key Vault:** Secrets management
- **Log Analytics:** Monitoring foundation
- **Recovery Services Vault:** Backup services
- **Virtual Machine:** Domain controllers, jumpboxes

### Module Hierarchy
```
Landing Zone Pattern Module
├── Management Group Module
├── Policy Assignment Module
├── Virtual Network Module (Hub)
├── Azure Firewall Module
├── Key Vault Module
├── Log Analytics Module
└── Spoke Network Templates
    ├── Virtual Network Module (Spoke)
    ├── Network Security Group Module
    └── Route Table Module
```

---

## Customer Customization

### Required Parameters
```yaml
Customer Configuration:
  - customer_prefix: Company identifier
  - primary_region: Main Azure region
  - secondary_region: DR region
  - address_space: IP address allocation
  - domain_name: Customer domain

Optional Parameters:
  - nva_vendor: Third-party firewall preference
  - compliance_frameworks: Additional compliance needs
  - backup_retention: Extended retention requirements
  - monitoring_level: Standard/enhanced monitoring
```

### Deployment Variables
```hcl
# terraform.tfvars template
customer_prefix    = "contoso"
primary_region     = "East US 2"
secondary_region   = "Central US"
hub_address_space  = "10.0.0.0/16"
corp_address_space = "10.1.0.0/12"
domain_name        = "contoso.com"
nva_vendor         = "fortinet" # optional
```
