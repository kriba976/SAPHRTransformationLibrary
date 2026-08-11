# SAP Business Technology Platform (BTP) — Hire-to-Retire Technical Reference

> **Scope:** This document covers SAP BTP account model, entitlements, user management, permissions, and Cloud Connector configuration scoped exclusively to the **Hire-to-Retire** business process. All scenarios reference SAP SuccessFactors (Employee Central, Onboarding, Payroll, Time, Recruiting), SAP Integration Suite, and on-premise SAP HXM/ERP (PA, OM, Payroll, Time Management).

---
## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Design Decisions](#2-design-decisions)
3. [Governance Model and Best Practices](#3-governance-model-and-best-practices)
4. [Global Account Structure](#4-global-account-structure)
5. [Directories](#5-directories)
6. [Subaccounts](#6-subaccounts)
7. [Entitlements and Service Plans](#7-entitlements-and-service-plans)
8. [User Management](#8-user-management)
9. [Role Collections and Permissions](#9-role-collections-and-permissions)
10. [Cloud Connector](#10-cloud-connector)
11. [Destinations](#11-destinations)
12. [Trust Configuration with IAS — Step-by-Step](#12-trust-configuration-with-ias--step-by-step)
13. [Security Strategy for H2R](#13-security-strategy-for-h2r)
14. [Reference Links](#14-reference-links)

---

## 1. Architecture Overview

### 1.1 High-Level Landscape

In a Hire-to-Retire landscape, SAP BTP acts as the **integration and extension platform** sitting between:

- **Cloud systems:** SAP SuccessFactors (Employee Central, Onboarding 2.0, Recruiting, Payroll, Time Tracking), SAP Identity Authentication Service (IAS)
- **On-premise systems:** SAP ERP HCM (PA/OM/Payroll/Time), SAP S/4HANA HCM
- **Third-party systems:** Microsoft Azure AD, SFTP servers, Payroll processors, Benefits providers

```
┌─────────────────────────────────────────────────────────────────┐
│                        SAP BTP Global Account                    │
│                                                                   │
│  ┌─────────────────────┐     ┌─────────────────────────────┐    │
│  │  Directory: HR Dev   │     │  Directory: HR Production    │    │
│  │  ┌───────────────┐  │     │  ┌──────────────────────┐   │    │
│  │  │ Subaccount:   │  │     │  │ Subaccount:           │   │    │
│  │  │ Integration   │  │     │  │ Integration Suite PRD │   │    │
│  │  │ Suite DEV     │  │     │  ├──────────────────────┤   │    │
│  │  ├───────────────┤  │     │  │ Subaccount:           │   │    │
│  │  │ Subaccount:   │  │     │  │ SF Extensions PRD     │   │    │
│  │  │ Integration   │  │     │  └──────────────────────┘   │    │
│  │  │ Suite QA      │  │     └─────────────────────────────┘    │
│  │  └───────────────┘  │                                         │
│  └─────────────────────┘                                         │
└─────────────────────────────────────────────────────────────────┘
        │                                        │
        ▼                                        ▼
┌──────────────┐                    ┌──────────────────────┐
│ SAP           │                    │ On-Premise SAP ERP   │
│ SuccessFactors│◄──── Cloud ────────│ HXM (PA/OM/Payroll/  │
│ Employee      │     Connector      │ Time Management)     │
│ Central       │                    └──────────────────────┘
└──────────────┘
```

### 1.2 BTP Services Used in Hire-to-Retire

| BTP Service                               | Purpose in H2R                                   |
| ----------------------------------------- | ------------------------------------------------ |
| SAP Integration Suite (Cloud Integration) | iFlow-based replication: SF EC → SAP ERP HCM     |
| SAP Integration Suite (API Management)    | Expose/manage SuccessFactors OData APIs          |
| Cloud Connector                           | Secure tunnel between BTP and on-premise SAP ERP |
| Destination Service                       | Manage connectivity configs to SF, ERP, SFTP     |
| SAP Identity Authentication Service (IAS) | SSO and user lifecycle for all H2R applications  |
| SAP Identity Provisioning Service (IPS)   | Replicate users from SF EC → IAS → on-prem LDAP  |
| SAP Build Work Zone                       | HR Employee Self-Service portal                  |

---

## 2. Design Decisions

### 2.1 Account Topology

| Decision | Recommendation | Rationale |
|---|---|---|
| Number of subaccounts | Minimum 3: DEV, QA, PRD | Isolation of integration workloads; independent lifecycle |
| Directory usage | Yes — use HR-specific directory | Groups entitlements and billing for HR workloads; enables directory-level role assignment |
| Region selection | EU10 or US10 depending on SuccessFactors data center | Always align BTP subaccount region with SuccessFactors data center to reduce latency and comply with data residency |
| Cloud Foundry vs Kyma | Cloud Foundry for Integration Suite | Integration Suite runs exclusively on Cloud Foundry environment |
| Entitlement model | Assign entitlements at directory level, distribute to subaccounts | Easier management when scaling across DEV/QA/PRD |

### 2.2 User Management Strategy

| Decision | Recommendation | Rationale |
|---|---|---|
| Platform vs Business users | Strictly separate | Platform users = BTP admins/developers. Business users = HR end-users via IAS |
| Default IdP vs IAS | Use IAS as custom IdP | Default SAP IdP (accounts.sap.com) should not be used in production HR landscapes |
| Shadow users | Enable shadow user creation | Required for user federation from IAS to BTP subaccounts |

### 2.3 Cloud Connector Design

| Decision | Recommendation | Rationale |
|---|---|---|
| HA setup | Yes — Master + Shadow instance | H2R data replication is business-critical; no single point of failure |
| On-premise exposure | Expose only RFC/HTTP services required for PA/OM/Payroll/Time | Principle of least privilege |
| System mapping | One virtual host per on-prem system type | Separate virtual hosts for PA system, Payroll system, and Time system |

---

## 3. Governance Model and Best Practices

### 3.1 What is Infrastructure Governance?

Infrastructure governance is a framework for effective infrastructure management ensuring security compliance, audit capabilities, and accountability across the BTP landscape. For H2R, where employee data is processed, governance is not optional — it is a compliance requirement.

**Governance Model consists of:**
- **Landscape Strategy** — defines, sets up, and maintains your cloud landscape
- **Onboarding Strategy** — Services onboarding, team onboarding with clear roles and responsibilities
- **Security Strategy** — authentication mechanisms, SSO, authorization management
- **Cost Management** — managing BTP Global Account costs under consumption-based commercial models

### 3.2 Landscape Strategy — H2R Account Model Best Practice

The correct BTP account model setup sequence for H2R is:

1. As part of the Global Contract with SAP, the Global Account is furnished with Entitlements
2. An Account Model is established defining the hierarchy: Global Account → Directories → Subaccounts
3. Entitlement management is activated at the Directory level
4. Service Plans for services (e.g., Integration Suite) are added to the Directory
5. Service Plans are then assigned to individual Subaccounts (making them available in the Service Marketplace)
6. Subaccount Administrator subscribes to or instantiates the Service
7. Service becomes available in **Instances and Subscriptions**

### 3.3 Naming Conventions for H2R Landscape

| Component | Convention | Example |
|---|---|---|
| Subaccount name | `<WorkStream>_<BusinessUnit>_<SystemIdentifier>` | `H2R_Contoso_IS_PRD` |
| Subdomain | Lowercase only, hyphens, no spaces, max 63 chars | `h2r-contoso-is-prd` |
| Subdomain format | UUID recommended for uniqueness | `a1b2c3d4-is-prd` |
| Directory name | Descriptive, reflecting functional area | `Human-Resources` |
| Cloud Foundry Org | Matches subaccount subdomain pattern | `h2r-contoso-prd` |
| CF Space | Functional name | `integration`, `monitoring` |

**Important:** URLs cannot be longer than 63 characters. Avoid underscores in subdomains — use hyphens only.

### 3.4 Global Account Governance Rules

| Rule | Detail |
|---|---|
| Every Global Account must have at least **2 Administrators** | Prevents single point of failure for admin access |
| Every service subscription/instance must have at least **2 Role Collections** | Minimum: Service Administrator + Service Developer |
| Global Services Catalogue must be maintained | Document listing all services the Global Account is entitled to |
| Local Services Catalogue per Subaccount | Document listing all service plans assigned to each subaccount |
| User Onboarding via documented process | All users added via the User Onboarding document — no ad-hoc access grants |
| All Global Account Administrators should also be Subaccount Administrators | Prevents admin lockout scenarios |

### 3.5 Onboarding Strategy for H2R Integration Team

When onboarding a new H2R integration project, document the following before requesting subaccount creation:

| Field | Example |
|---|---|
| Organization/department | `HR Integration Team` |
| Application name | `H2R Employee Central Integration` |
| Technical application name | `h2r-ec-integration` (no spaces, alphabetic only) |
| Business case | Employee master data replication SF EC → SAP ERP HCM |
| Planned Go-Live date | `2025-03-31` |
| Application owners | `integration-lead@company.com` |
| Data flow description | SF EC OData API → Integration Suite → ERP HCM via Cloud Connector |
| Programming languages | Groovy, XSLT (within Integration Suite iFlows) |
| Git repository URL | `https://github.com/company/h2r-integration` |
| Test strategy | Unit test in DEV, Integration test in QA with SF QA + ERP QA |

### 3.6 Services Catalogue (H2R Subaccount)

Maintain this catalogue per subaccount for the H2R landscape:

| Service | Plan | Status | Subaccount |
|---|---|---|---|
| Integration Suite | standard | Subscribed | HR-Integration-PRD |
| Connectivity Service | lite | Entitled + Instance created | HR-Integration-PRD |
| Destination Service | lite | Entitled + Instance created | HR-Integration-PRD |
| Authorization and Trust Management | application | Instance created | HR-Integration-PRD |
| Cloud Identity Services (IAS) | application | Subscribed | HR-Integration-PRD |
| Alert Notification Service | standard | Instance created | HR-Integration-PRD |

---

## 4. Global Account Structure

### 3.1 What is a Global Account?

A **Global Account** is the contractual and administrative root entity in SAP BTP. It represents your signed contract with SAP (via CPEA, BTPEA, or subscription-based agreements). All resources, subaccounts, entitlements, and billing are anchored to the Global Account.

**Key facts:**
- One Global Account per SAP contract
- Entitlements (service plans) are assigned at the Global Account level and then distributed to subaccounts
- Global Account Administrators have full visibility across all subaccounts
- Managed via: [SAP BTP Cockpit](https://cockpit.btp.cloud.sap/)

### 3.2 Global Account Administration for H2R

The following roles should be assigned at Global Account level for the HR integration team:

| Role Collection | Assigned To | Scope |
|---|---|---|
| Global Account Administrator | Basis/BTP Admin team (max 2-3 people) | Full control: subaccounts, entitlements, members |
| Global Account Viewer | HR Integration Architects | Read-only visibility across all subaccounts |
| Directory Administrator | HR Integration Lead | Manage the HR directory and its subaccounts |

### 3.3 Contract Types Relevant to H2R

| Contract Type | Description | H2R Use Case |
|---|---|---|
| **CPEA** (Cloud Platform Enterprise Agreement) | Consumption-based; credits consumed per service used | Preferred for H2R — predictable SAP Integration Suite + IAS usage |
| **BTPEA** (BTP Enterprise Agreement) | Similar to CPEA with volume commitments | Large enterprises with defined H2R scope |
| **Free Tier / Trial** | Limited capacity, no SLA | Development/POC only; NOT for production H2R |
| **Subscription** | Fixed monthly subscription per service | Suitable if only subscribing to Integration Suite standard plan |

---

## 4. Directories

### 4.1 Purpose of Directories

Directories are optional organizational containers within a Global Account that group subaccounts. They are particularly valuable for HR landscapes to:

- Separate HR workloads from other BTP workloads (Finance, Supply Chain, etc.)
- Assign entitlements at the directory level and auto-distribute to child subaccounts
- Apply directory-level role collections (e.g., all HR subaccount administrators)
- Enable consolidated cost tracking per business unit

### 4.2 Recommended Directory Structure for H2R

```
Global Account
└── Directory: Human Resources (HR)
    ├── Directory: HR Integration
    │   ├── Subaccount: HR-Integration-DEV
    │   ├── Subaccount: HR-Integration-QA
    │   └── Subaccount: HR-Integration-PRD
    └── Directory: HR Extensions
        ├── Subaccount: HR-Extensions-DEV
        └── Subaccount: HR-Extensions-PRD
```

### 4.3 Directory Entitlement Management

1. Navigate to **BTP Cockpit > Global Account > Entitlements > Entity Assignments**
2. Select the **HR Directory**
3. Assign service plans (e.g., Integration Suite — Standard Edition) to the directory with quota
4. From the directory, distribute quota to individual subaccounts
5. Subaccounts can only consume what has been allocated to them

> **Reference:** [Configure Entitlements and Quotas for Subaccounts](https://help.sap.com/docs/btp/sap-business-technology-platform/configure-entitlements-and-quotas-for-subaccounts)

---

## 5. Subaccounts

### 5.1 What is a Subaccount?

A **Subaccount** is the deployable unit in BTP — it is where services are subscribed, instances are created, applications are deployed, and users are managed. Each subaccount:

- Belongs to exactly one Global Account (and optionally one Directory)
- Has its own trust configuration (IdP)
- Has its own role collections, users, and authorizations
- Maps to a specific **region** (e.g., `eu10`, `us10`)
- Has its own Cloud Foundry **Space** hierarchy (if CF environment is enabled)

### 5.2 Subaccount Design for H2R

| Subaccount | Environment | Purpose | Region |
|---|---|---|---|
| `HR-Integration-DEV` | Cloud Foundry | Integration Suite development, iFlow unit testing | Same as SF DEV data center |
| `HR-Integration-QA` | Cloud Foundry | Integration testing with SF QA tenant and ERP QA | Same as SF QA data center |
| `HR-Integration-PRD` | Cloud Foundry | Production H2R data replication flows | Same as SF PRD data center |
| `HR-Extensions-PRD` | Cloud Foundry or Kyma | Custom apps, SF extensions, self-service portals | Same as SF PRD |

### 5.3 Cloud Foundry Environment Setup

For each subaccount running Integration Suite:

1. In BTP Cockpit, navigate to the subaccount
2. Under **Cloud Foundry Environment**, click **Enable**
3. Provide an **Org Name** (e.g., `HR-Integration-PRD`)
4. Once CF is enabled, create at least one **Space** (e.g., `integration`, `monitoring`)
5. Assign Space Developers and Space Managers from the HR integration team

### 5.4 Subaccount Properties Configuration

Configure these properties for each H2R subaccount:

| Property | Setting | Notes |
|---|---|---|
| Custom IdP | IAS tenant URL | Replace the default SAP IdP |
| Shadow User Creation | Enabled | Required for IAS-federated users |
| Usage Analytics | Enabled | Monitor Integration Suite message consumption |

---

## 6. Entitlements and Service Plans

### 6.1 Core H2R Entitlements

The following entitlements must be configured in BTP for a full H2R landscape:

| Service | Plan | Subaccount | Notes |
|---|---|---|---|
| **Integration Suite** | Standard Edition or Enterprise Edition | HR-Integration-PRD | Core middleware for H2R data replication |
| **Cloud Identity Services** (IAS) | application | HR-Integration-PRD | SSO and user authentication |
| **SAP Identity Provisioning** | | HR-Integration-PRD | Bundled with Cloud Identity Services since 2022 |
| **Connectivity Service** | lite | All subaccounts | Required for Cloud Connector tunnels |
| **Destination Service** | lite | All subaccounts | Manages connection configs to SF, ERP, SFTP |
| **Authorization and Trust Management** | application | All subaccounts | Required for OAuth-based integrations |
| **HTML5 Application Repository** | app-host | Extensions subaccount | If building HR self-service UIs |
| **SAP Build Work Zone** | standard | Extensions subaccount | Employee Self-Service for H2R |

### 6.2 Integration Suite — Plan Comparison

| Plan | Message Volume | Features | H2R Recommendation |
|---|---|---|---|
| **Free** | 50 messages/month | Cloud Integration only | Dev/POC only |
| **Standard Edition** | Up to 10K messages/month | Cloud Integration + API Management + basic connectors | Small to mid H2R landscapes |
| **Enterprise Edition** | Unlimited (pay-per-use) | Full suite: CI, API Mgmt, EDI, Open Connectors, Integration Advisor | Large enterprises, complex H2R |

> A "message" in Integration Suite = one processed artifact payload. For H2R, count messages per employee event (hire, change, terminate) × number of integration flows triggered.

### 6.3 Configuring Entitlements Step-by-Step

1. Log in to [BTP Cockpit](https://cockpit.btp.cloud.sap/) as **Global Account Administrator**
2. Navigate to **Global Account > Entitlements > Entity Assignments**
3. Search for and select subaccount(s) (e.g., `HR-Integration-PRD`)
4. Click **Configure Entitlements > Add Service Plans**
5. Search for **Integration Suite** → select **standard** or **enterprise** plan → Add
6. Search for **Connectivity** → select **lite** → Add
7. Search for **Destination** → select **lite** → Add
8. Click **Save**

---

## 7. User Management

### 7.1 Platform Users vs Business Users

SAP BTP distinguishes two user types — understanding this distinction is critical for H2R:

| Attribute | Platform User | Business User |
|---|---|---|
| Who | BTP Administrators, Integration Developers, Basis team | HR End-users, Managers, Employees |
| Stored in | SAP ID Service (default) or IAS | IAS (federated from Azure AD / on-prem AD) |
| Used for | BTP Cockpit access, deploying iFlows, managing subaccounts | Accessing SF Work Zone, HR apps, SSO scenarios |
| Managed via | BTP Cockpit → Security → Users | IAS Admin Console + IPS provisioning |
| Authentication | SAP Universal ID or IAS | IAS (as IdP or proxy) |

### 7.2 Managing Platform Users

#### Adding a Platform User to a Subaccount

1. BTP Cockpit → Subaccount → **Security > Users**
2. Click **Create** → Enter email address (must match SAP Universal ID or IAS user)
3. Select **Identity Provider** (Custom IAS tenant if configured)
4. Assign applicable **Role Collections** (see Section 8)

#### Adding a Platform User to a Directory

1. BTP Cockpit → Directory → **Members**
2. Click **Add Member** → Enter user email
3. Assign Directory Administrator or Directory Viewer role

### 7.3 Business User Lifecycle in H2R

Business users in H2R are managed through the following flow:

```
SF Employee Central
  (New Hire event)
       │
       ▼
IPS (Identity Provisioning)
  Source: SF EC
  Target: IAS
       │
       ▼
IAS (Identity Authentication)
  User profile created/updated
       │
       ▼
BTP Subaccount (Shadow User)
  Auto-created on first login
       │
       ▼
Role Collection Assigned
  (manually or via IPS attribute mapping)
```

### 7.4 Shadow Users

Shadow users are automatically created in a BTP subaccount when a user authenticated via IAS first accesses the subaccount. To enable:

1. BTP Cockpit → Subaccount → **Security > Trust Configuration**
2. Click on the IAS trust entry
3. Ensure **Create Shadow Users During Logon** is set to **Enabled**

---

## 8. Role Collections and Permissions

### 8.1 BTP Permission Model

```
Application Role Template
       ▼
Role (created from template, may have attribute values)
       ▼
Role Collection (groups multiple roles)
       ▼
User / Group (role collection assigned to user or user group)
```

### 8.2 Default Role Collections Relevant to H2R

#### At Global Account Level

| Role Collection | Purpose |
|---|---|
| `Global Account Administrator` | Full GA management — assign to 2-3 BTP Basis admins only |
| `Global Account Viewer` | Read-only — HR architects, auditors |
| `Directory Administrator` | Manage the HR directory and its subaccounts |

#### At Subaccount Level

| Role Collection | Purpose |
|---|---|
| `Subaccount Administrator` | Full subaccount management |
| `Subaccount Viewer` | Read-only — for L2 support |
| `Cloud Connector Administrator` | Manage Cloud Connector connections |
| `Destination Administrator` | Create/modify destinations |

#### Integration Suite Role Collections

| Role Collection | Purpose | Assign To |
|---|---|---|
| `PI_Administrator` | Full Integration Suite administration | Integration Suite Basis admin |
| `PI_Integration_Developer` | Design and deploy iFlows | HR integration developers |
| `PI_Business_Expert` | Monitor message processing | HR functional team leads |
| `PI_Read_Only` | View iFlows, monitoring only | L1/L2 support |

### 8.3 Assigning Role Collections

**Via BTP Cockpit (individual user):**
1. Subaccount → Security → Users → Select user
2. Click **Assign Role Collection**
3. Select the relevant role collections

**Via IAS Groups (bulk assignment — recommended for production):**
1. In IAS Admin Console, create user groups (e.g., `BTP_Integration_Developers`)
2. In BTP Cockpit → Trust Configuration → IAS entry → **Role Collection Mappings**
3. Map IAS group `BTP_Integration_Developers` → Role Collection `PI_Integration_Developer`
4. All users in the IAS group automatically receive the role collection on login

### 8.4 Custom Role Collections for H2R

Create these custom role collections for fine-grained H2R access control:

| Custom Role Collection | Roles Included | Purpose |
|---|---|---|
| `H2R_Integration_Developer_DEV` | `PI_Integration_Developer`, `Subaccount_Viewer` | Developers with deploy rights in DEV only |
| `H2R_Integration_Developer_PRD` | `PI_Business_Expert` | Developers with monitor-only access in PRD |
| `H2R_Support_L1` | `PI_Read_Only`, `Destination_Viewer` | Support team — read-only monitoring |
| `H2R_Basis_Admin` | `PI_Administrator`, `Cloud_Connector_Administrator` | Full admin for H2R subaccount |

---

## 9. Cloud Connector

### 9.1 What is the Cloud Connector?

The **SAP Cloud Connector (SCC)** is a lightweight on-premise agent that establishes a **reverse-invoke tunnel** from the on-premise network to SAP BTP. It does NOT require opening inbound firewall ports — the SCC initiates the outbound connection to BTP.

In H2R, the Cloud Connector is required to:
- Replicate employee master data from SF Employee Central → on-premise SAP ERP HCM (PA/OM)
- Trigger RFC function modules in on-premise SAP Payroll/Time from Integration Suite iFlows
- Retrieve organizational structure from on-premise SAP OM for SF synchronization

### 9.2 Architecture

```
On-Premise Network (DMZ/Application Zone)
┌──────────────────────────────────────────┐
│                                          │
│  SAP ERP HCM (PA/OM/Payroll/Time)        │
│       ▲                                  │
│       │ RFC / HTTP                       │
│       │                                  │
│  ┌────┴──────────────────┐               │
│  │  Cloud Connector       │               │
│  │  (Master Instance)     │               │
│  │  Port 8443 (UI)        │               │
│  │  Port: outbound HTTPS  │               │
│  └────────────┬──────────┘               │
│               │ Outbound HTTPS (443)      │
│               │ No inbound ports needed  │
└───────────────┼──────────────────────────┘
                │
                ▼
        SAP BTP Region Endpoint
        (e.g., connectivity.eu10.hana.ondemand.com)
                │
                ▼
        BTP Subaccount (HR-Integration-PRD)
                │
                ▼
        Integration Suite iFlow
```

### 9.3 Installation and Initial Setup

#### Prerequisites
- Java 11 or higher installed on the host server
- Network access from the Cloud Connector host to `*.hana.ondemand.com` on port 443
- SAP BTP subaccount details (Region, Subaccount ID, Subdomain)

#### Installation Steps

1. Download the Cloud Connector installer from [SAP Tools](https://tools.hana.ondemand.com/#cloud)
   - Select: **Cloud Connector** → version matching your OS (Linux RPM/DEB, Windows MSI)
2. Install on a dedicated on-premise server (not the SAP ERP application server)
3. Start the SCC service:
   - **Linux:** `systemctl start scc_daemon`
   - **Windows:** Start via Services panel or `net start SAP_Cloud_Connector`
4. Access the SCC Admin UI: `https://<hostname>:8443`
5. Default credentials: `Administrator` / `manage` (change immediately)

#### Connecting to BTP Subaccount

1. In SCC Admin UI → **Define Subaccount**
2. Fill in:
   - **Region**: Select the BTP region (e.g., `Europe (Frankfurt) - EU10`)
   - **Subaccount**: Enter the Subaccount ID (from BTP Cockpit → Subaccount Overview)
   - **Subaccount User**: Email of a BTP user with `Cloud Connector Administrator` role
   - **Password**: Password of the above user
3. Click **Save** — the SCC will establish the tunnel
4. Status should show **Connected**

> **Reference:** [Cloud Connector Setup](https://help.sap.com/docs/connectivity/sap-btp-connectivity-cf/cloud-connector)

### 9.4 Configuring On-Premise System Access (Access Control)

This is where you define which on-premise systems and resources are accessible from BTP.

#### Adding a System Mapping (for SAP ERP HCM)

1. SCC Admin UI → Subaccount → **Cloud To On-Premise > Access Control**
2. Click **Add (+)**
3. Configure:
   - **Back-end Type**: `SAP ABAP System` (for RFC) or `Non-SAP System` (for HTTP)
   - **Protocol**: `RFC` or `HTTP/HTTPS`
   - **Internal Host**: `<SAP ERP application server hostname>` (internal DNS)
   - **Internal Port**: `33<instance_no>` for RFC (e.g., `3300` for instance 00), `80<instance_no>` for HTTP
   - **Virtual Host**: `erp-hcm-prd` (this is what BTP destinations reference — never expose real hostname)
   - **Virtual Port**: `33300` (for RFC) or `8000` (for HTTP)
4. Add **Resources** under the system mapping:
   - For RFC: Click **Add** under Resources → Enter function module name or `*` for all (NOT recommended for PRD)
   - For H2R specific RFCs, add individually:
     - `HRMD_A` (HR Master Data)
     - `BAPI_EMPLOYEE_GETDATA`
     - `RFC_READ_TABLE` (use cautiously)
     - Custom BAPIs for Payroll/Time if applicable

#### Access Control Best Practice for H2R

| Resource | Protocol | Allowed | Notes |
|---|---|---|---|
| `/sap/bc/srt/scs/` | HTTPS | Yes | SOAP WebService endpoint for HR BAPIs |
| `/sap/bc/rest/` | HTTPS | Yes (specific paths only) | REST services for OM/Time |
| `RFC` to PA/OM FMs | RFC | Yes (specific FMs only) | Never use wildcard `*` in PRD |
| `/sap/bc/adt/` | HTTPS | No | ABAP Development Tools — not needed for runtime |

### 9.5 High Availability Setup

For production H2R landscapes, configure a Master-Shadow pair:

1. Install a second SCC instance on a separate server
2. In the Shadow SCC Admin UI → **HA Settings**
3. Configure:
   - **Master Host**: Hostname of the Master SCC
   - **Master Port**: `8443`
   - **Own Address**: This server's hostname
4. The Shadow instance takes over automatically if the Master fails

### 9.6 Certificate Management for Cloud Connector

For certificate-based authentication between SCC and BTP:
1. SCC Admin UI → **Configuration > On-Premise to Cloud Security**
2. Generate a **Certificate Signing Request (CSR)**
3. Submit the CSR to your corporate CA or use a self-signed cert
4. Import the signed certificate
5. In BTP Cockpit → Subaccount → **Connectivity > Cloud Connectors**, the SCC will show the certificate fingerprint — verify it matches

---

## 10. Destinations

### 10.1 What are Destinations?

Destinations in SAP BTP store connection parameters (URL, authentication type, credentials) for calling external systems. In H2R, destinations are used by Integration Suite iFlows to connect to:
- SAP SuccessFactors OData APIs
- On-premise SAP ERP HCM via Cloud Connector
- SFTP servers for file-based payroll/time integrations

### 10.2 Destination Configuration for H2R

#### SuccessFactors OData API Destination

| Property | Value |
|---|---|
| Name | `SuccessFactors_PRD` |
| Type | `HTTP` |
| URL | `https://<SF_API_endpoint>.successfactors.com/odata/v2` |
| Proxy Type | `Internet` |
| Authentication | `OAuth2SAMLBearerAssertion` |
| Audience | `www.successfactors.com` |
| AuthTokenEndpoint | `https://<SF_tenant>.successfactors.com/oauth/token` |
| Client Key | `<Client ID from SF OAuth configuration>` |
| Token Service URL | `https://<SF_tenant>.successfactors.com/oauth/token` |

**Additional Properties:**
| Key | Value |
|---|---|
| `nameIdFormat` | `urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified` |
| `companyId` | `<SF Company ID>` |
| `apiKey` | `<SF API Key>` (if using API Key auth) |

#### On-Premise SAP ERP HCM Destination (via Cloud Connector)

| Property | Value |
|---|---|
| Name | `SAP_ERP_HCM_PRD` |
| Type | `RFC` |
| Proxy Type | `OnPremise` |
| Authentication | `BasicAuthentication` |
| User | `<RFC technical user in ERP>` |
| Password | `<password>` |

**Additional Properties:**
| Key | Value |
|---|---|
| `jco.client.client` | `100` (SAP client number) |
| `jco.client.sysnr` | `00` (system number) |
| `jco.client.lang` | `EN` |
| `jco.destination.pool_capacity` | `5` |

#### SFTP Destination for Payroll/Time File Transfer

| Property | Value |
|---|---|
| Name | `SFTP_Payroll_PRD` |
| Type | `HTTP` |
| URL | `sftp://<sftp_server_hostname>:22` |
| Proxy Type | `Internet` or `OnPremise` (if SFTP is internal) |
| Authentication | `BasicAuthentication` or `ClientCertificateAuthentication` |

---

## 12. Trust Configuration with IAS — Step-by-Step

### 12.0 Why IAS Must Replace the Default IdP

The default SAP ID Service (`accounts.sap.com`) is only suitable for initial setup. For H2R production landscapes, it must be replaced with SAP IAS because:
- IAS supports federation with Azure AD and on-premise AD (corporate credentials)
- IAS enables SSO across all SAP cloud applications (SF, BTP, Build Work Zone)
- IAS allows risk-based authentication and MFA enforcement for HR data access
- IAS manages the new hire activation email and password lifecycle

### 12.1 Replacing the Default IdP with IAS

For H2R production subaccounts, the default SAP IdP (`accounts.sap.com`) must be replaced with a custom **SAP Identity Authentication Service (IAS)** tenant.

#### Configuration Steps

1. In IAS Admin Console (`https://<tenant>.accounts.ondemand.com/admin`):
   - Navigate to **Applications > Add Application**
   - Create a new application of type **SAP BTP**
   - Note the **Metadata URL** (e.g., `https://<tenant>.accounts.ondemand.com/saml2/metadata`)

2. In BTP Cockpit → Subaccount → **Security > Trust Configuration**:
   - Click **Establish Trust**
   - Select **SAP Cloud Identity Services** (automatic if IAS is in the same global account)
   - Or click **New Trust Configuration** and enter the IAS metadata URL manually
   - Set **Status** to **Active**
   - Check **Create Shadow Users During Logon**: **Enabled**

3. Disable the default SAP ID Service trust entry (set to **Inactive**) after IAS trust is established and tested

### 12.2 Detailed BTP Trust Configuration Steps

This is the full step-by-step process for establishing BTP ↔ IAS trust for H2R:

**In BTP Cockpit (step 1 — create signing key):**
1. Subaccount → **Security > Trust Configuration**
2. Click **Local Service Provider**
3. Under **Signing Key** → click **Generate Key Pair**
4. Download the BTP SP metadata file (contains the public certificate and ACS URL)
5. Send this metadata to the IAS tenant administrator (or upload it yourself if you have IAS admin access)

**In IAS Admin Console (step 2 — register BTP as application):**
1. IAS Admin Console → **Applications & Resources > Applications**
2. Click **Create > SAP BTP Solution**
3. Set **Display Name**: `BTP-H2R-PRD`
4. Under **Trust > SAML 2.0 Configuration**:
   - Upload the BTP SP metadata from step 1
   - The **Assertion Consumer Service (ACS) URL** and **Entity ID** auto-populate
5. Under **Subject Name Identifier**:
   - Source: `Identity Directory`
   - Value: `Login Name`
6. Under **Attributes** → add:
   - `Groups` → from Identity Directory → Groups attribute (for role collection mapping)
   - `email` → from Identity Directory → E-mail
   - `first_name` → from Identity Directory → First Name
   - `last_name` → from Identity Directory → Last Name

**In BTP Cockpit (step 3 — configure IAS as IdP):**
1. Download the IAS metadata: `https://<tenant>.accounts.ondemand.com/saml2/metadata`
2. Subaccount → **Security > Trust Configuration**
3. Click **New Trust Configuration**
4. Upload the IAS metadata
5. Set **Name**: `IAS Corporate IdP`
6. Enable **Available for User Logon**: ✓
7. Enable **Create Shadow Users During Logon**: ✓
8. Set **Status**: **Active**

**SAML Assertion Example (what IAS sends to BTP):**
```xml
<Assertion xmlns="urn:oasis:names:tc:SAML:2.0:assertion"
           IssueInstant="2025-06-01T08:52:39Z">
  <Issuer>company.accounts.ondemand.com</Issuer>
  <Subject>
    <NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress">
      john.doe@company.com
    </NameID>
  </Subject>
  <Conditions NotBefore="2025-06-01T08:47:39Z" NotOnOrAfter="2025-06-01T09:02:39Z">
    <AudienceRestriction>
      <Audience>h2r-integration-prd</Audience>  <!-- BTP subaccount subdomain -->
    </AudienceRestriction>
  </Conditions>
  <AttributeStatement>
    <Attribute Name="Groups">
      <AttributeValue>PI_Integration_Developer</AttributeValue>
      <AttributeValue>H2R_Support_L1</AttributeValue>
    </Attribute>
    <Attribute Name="email">
      <AttributeValue>john.doe@company.com</AttributeValue>
    </Attribute>
  </AttributeStatement>
</Assertion>
```

**Subject Name Identifier — how BTP identifies the user:**
The `NameID` in the SAML assertion is how BTP creates/matches the shadow user. If `NameID` source is set to `Login Name` in IAS, then BTP creates a shadow user with that login name. Choose the field consistently across all BTP subaccounts.

### 12.3 Role Collection Mapping via IAS Groups

Instead of assigning role collections to individual users, configure group-based mapping:

1. In BTP Cockpit → Subaccount → **Security > Trust Configuration**
2. Click the IAS trust entry → **Role Collection Mappings**
3. Add mappings:

| IAS Group | BTP Role Collection |
|---|---|
| `BTP_Integration_Developers` | `PI_Integration_Developer` |
| `BTP_H2R_Admins` | `PI_Administrator`, `Subaccount_Administrator` |
| `BTP_H2R_Support` | `PI_Read_Only` |
| `BTP_Basis_Admins` | `Subaccount_Administrator`, `Cloud_Connector_Administrator` |

4. These groups must exist in IAS and users must be assigned to them
5. Groups are populated either manually in IAS or automatically via IPS (from SF EC)
6. On user login, BTP reads the `Groups` attribute from the SAML assertion and assigns matching role collections

### 12.4 Trust for Platform Users

Platform users (admins, developers) should also authenticate via IAS:
1. In the Trust Configuration, set the IAS entry as **Available for User Logon**
2. Under **Custom Identity Provider for Platform Users**, enable IAS as the platform user IdP
3. This enables MFA enforcement at IAS level for BTP platform access

---

## 13. Security Strategy for H2R

### 13.1 Security Layers in BTP

| Layer | Controls |
|---|---|
| **Network** | TLS 1.2+ for all connections; Cloud Connector outbound-only tunnel; no inbound firewall ports |
| **Host** | SAP-managed BTP infrastructure; firewalls, DDoS protection, intrusion detection |
| **Application** | Role-based access via role collections; IAS MFA enforcement; OAuth2 token-based API access |
| **Data** | PGP encryption for file transfers; TLS for API calls; GDPR-compliant data handling |

### 13.2 Employee Identity Lifecycle Management in BTP

SAP BTP supports **identity federation** — applications on BTP do not store or manage user credentials. Instead, authentication is delegated to IAS (which in turn delegates to Azure AD/ADFS). This means:

- BTP never stores employee passwords
- Employee account activation/deactivation in SF EC automatically propagates to IAS via IPS, which deactivates BTP access
- HR managers do not need to separately manage BTP user accounts — lifecycle is fully automated

### 13.3 Authorization Management via Microsoft Entra (Azure AD)

For clients using Azure AD, BTP authorization groups can be managed through **Azure AD Access Packages**:

1. In Azure AD → **Identity Governance > Access Packages**
2. Create an access package: `BTP H2R Integration Developer Access`
3. Include the resource group: IAS group `BTP_Integration_Developers`
4. Configure approval workflow (e.g., manager approval required)
5. Users request access via `myapps.microsoft.com` → manager approves → they are added to the IAS group → BTP role collection auto-assigned

This creates a fully auditable access request and approval process for BTP access in H2R projects.

### 13.4 Data Protection and Privacy for H2R

BTP provides the following services for GDPR compliance in H2R:
- **SAP Data Retention Management** — define retention rules for employee data in integration flows
- **SAP Personal Data Manager** — identify and report on personal data processed in BTP
- **SAP Data Privacy Integration** — manage consent and data subject requests

For H2R integration flows, implement:
- Log only non-personal data in iFlow message logs (employee ID, not names or dates of birth)
- Enable message log retention limits (30 days for DEV/QA, 90 days for PRD with legal hold)
- Use `SAP_ApplicationID` header (employee ID only) for correlation — never log full payloads containing personal data in PRD

---

## 14. Common Pitfalls & Real-World Issues

### Account Model & Subaccounts

| Issue | Root Cause | Resolution |
|---|---|---|
| **Subaccount subdomain cannot be changed after creation** | The subdomain is immutable once set; it forms part of the tenant URL for all services | Plan subdomains carefully before go-live; use a UUID-based or consistent naming pattern from day one; never use project codenames that may change |
| **Entitlement quota exhausted mid-project — Integration Suite cannot be subscribed in a new subaccount** | All quota was assigned to existing subaccounts at Global Account level, leaving nothing for new ones | Always hold back a buffer at the Global Account level; use directory-level entitlement management to set soft ceilings per workstream |
| **Subaccount administrator locked out — no second admin was configured** | Only one admin was set up; that person left the project | Enforce the governance rule: every subaccount must have a minimum of 2 administrators from the BTP Basis team |
| **Custom IdP (IAS) trust was configured but the default SAP IdP was never disabled — users can still log in with personal SAP accounts** | Both trust configurations are active; users choose which to authenticate with | After testing the IAS trust thoroughly, set the default SAP ID Service trust to **Inactive** in Trust Configuration |

### Cloud Connector

| Issue | Root Cause | Resolution |
|---|---|---|
| **Cloud Connector connects but iFlow cannot reach the on-premise RFC function module** | The resource (function module name) was not added to the Access Control list in SCC; wildcard `*` was intentionally avoided | Add the specific RFC function module name (e.g., `HRMD_A`) as an allowed resource under the system mapping |
| **SCC HA Shadow instance never takes over when the Master goes down** | Shadow was configured but the **Own Host** setting used an IP address that changed after a VM reboot | Always use a stable DNS hostname (not IP address) for the Shadow's Own Host configuration |
| **SCC connection drops nightly — tunnel disconnects at the same time each night** | Corporate firewall or load balancer has an idle connection timeout that is shorter than the SCC keep-alive interval | Increase the SCC keep-alive interval in Configuration → Advanced → Tunnel settings, or request a firewall rule change to extend the idle timeout |
| **After a certificate rotation in BTP, the SCC shows "untrusted certificate" and stops connecting** | The BTP subaccount system certificate was rotated (auto-renewal), and the SCC still holds the old certificate fingerprint | In BTP Cockpit → Subaccount → Connectivity → Cloud Connectors, verify the fingerprint matches the SCC; in SCC, re-accept the new certificate |

### Destinations

| Issue | Root Cause | Resolution |
|---|---|---|
| **OAuth2SAMLBearerAssertion destination to SuccessFactors returns 401 after working for months** | The OAuth client certificate in SuccessFactors expired; SuccessFactors OAuth applications have a certificate validity period | Monitor the certificate expiry date in SF Admin Center → OAuth 2.0 Client Applications; rotate and re-register the certificate at least 30 days before expiry |
| **Destination test in BTP Cockpit returns "OK" but iFlow fails with connection refused** | The Cockpit test only validates that the destination configuration is syntactically correct and the target is reachable from BTP network — it does not test the full auth flow | Always test destinations end-to-end by running a real iFlow call, not just the Cockpit test; the test button is not a substitute for integration testing |

---

## 15. Reference Links

| Topic | Link |
|---|---|
| BTP Account Model | [help.sap.com — BTP Account Model](https://help.sap.com/docs/btp/sap-business-technology-platform/account-model) |
| Configure Entitlements | [help.sap.com — Entitlements and Quotas](https://help.sap.com/docs/btp/sap-business-technology-platform/configure-entitlements-and-quotas-for-subaccounts) |
| Role Collections | [help.sap.com — Role Collections and Roles](https://help.sap.com/docs/btp/sap-business-technology-platform/role-collections-and-roles-in-global-accounts-directories-and-subaccounts) |
| Cloud Connector | [help.sap.com — Cloud Connector](https://help.sap.com/docs/connectivity/sap-btp-connectivity-cf/cloud-connector) |
| Connecting SCC to EC-S4HCM Integration | [help.sap.com — Connecting Cloud Connector to Cloud Integration](https://help.sap.com/docs/successfactors-ec-s4-hcm-integration/implementing-data-replication-from-employee-central/connecting-cloud-connector-to-your-sap-cloud-integration-account) |
| BTP Onboarding Series | [SAP Community — Account Structure and Decision Making](https://community.sap.com/t5/technology-blog-posts-by-sap/sap-btp-onboarding-series-account-structure-and-decision-making/ba-p/13523012) |
| SuccessFactors BTP Integration | [SAP Community — Integrating SF APIs with BTP](https://community.sap.com/t5/human-capital-management-blog-posts-by-sap/integrating-sap-successfactors-apis-with-sap-btp-destination-setup-and/ba-p/14294201) |
| Hire to Retire Overview | [SAP Learning — Hire to Retire Solutions](https://learning.sap.com/learning-journeys/get-started-with-sap-hcm-payroll/outlining-sap-successfactors-hire-to-retire-solutions_fe35f87a-7d77-413c-b489-ad88a1599009) |
| Default BTP Role Collections | [help.sap.com — Default Role Collections CF](https://help.sap.com/docs/btp/sap-business-technology-platform/default-role-collections-of-sap-btp-cloud-foundry-environment) |

---

*Document Version: 1.0 | Scope: Hire-to-Retire | Last Updated: June 2026*
