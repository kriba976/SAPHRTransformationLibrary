# SAP Integration Governance Technical Reference
## Development, BTP Infrastructure Management, Role based Access Control, Release Management, Operations and maintenance

---

## Overview

In large-scale SAP SuccessFactors implementations, the SAP BTP landscape is not just a runtime — it is a managed platform that requires disciplined infrastructure operations across four pillars: **BTP Structuring**, **DevOps & Release Management**, **Operations & Monitoring**, and **Access Governance**. This document defines the strategy, architecture, setup steps, and governance model for each pillar, with real-world patterns and common pitfalls.

> **Relationship to other documents:**
> - BTP Account Model, Subaccounts, Directories, and Trust Configuration → see `SAP_BTP_Hire_to_Retire`
> - Integration Strategy, Naming Conventions, RBAC → see `Cloud_Integration_Strategy_and_Governance`
> - Audit Log routing to SIEM → see `SAP_BTP_Audit_Management_SIEM`

```table-of-contents
title: 
style: nestedList # TOC style (nestedList|nestedOrderedList|inlineFirstLevel)
minLevel: 0 # Include headings from the specified level
maxLevel: 0 # Include headings up to the specified level
include: 
exclude: 
includeLinks: true # Make headings clickable
hideWhenEmpty: false # Hide TOC if no headings are found
debugInConsole: false # Print debug info in Obsidian console
```



---

## 1. Integration Governance Overview

### 1.1 The Five Pillars of Integration Governance

```
┌──────────────────────────────────────────────────────────────────────────┐
│                SAP BTP Infrastructure Management Overview                 │
│                                                                            │
│  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐    │
│  │  1. BTP           │  │  2. DevOps &       │  │  3. Operations &  │    │
│  │  Structuring      │  │  Release Mgmt      │  │  Monitoring       │    │
│  │                   │  │                    │  │                   │    │
│  │  Terraform IaC    │  │  cTMS              │  │  SAP Cloud ALM    │    │
│  │  Subaccount       │  │  GitHub / FlashPipe│  │  Alert Mgmt       │    │
│  │  Topology         │  │  CI/CD Pipeline    │  │  Anomaly Detect.  │    │
│  └───────────────────┘  └───────────────────┘  └───────────────────┘    │
│                                                                            │
│  ┌───────────────────────────────────────────────────────────────────┐   │
│  │  4. Access Governance                                              │   │
│  │  IAS as centralised IdP · ServiceNow ticketing · BTP RBAC        │   │
│  └───────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.2 End-to-End Release & Operations Flow

```
Developer commits iFlow
        │
        ▼
GitHub Pull Request
  (peer review)
        │
        ▼
Post Development → Merge to main → Deploy to github
        │
        ▼
Developer tests in DEV → raises cTMS transport request
        │
        ▼
cTMS: DEV Node → QA Node (manual approval by team lead)
        │
        ▼
QA Testing (Cloud ALM monitors for errors)
        │
        ▼
cTMS: QA Node → PROD Node (approval by integration manager)
        │
        ▼
PROD Deployment
        │
        ▼
Cloud ALM detects errors / anomalies → SAP Alert Notification → Email / Teams
        │
        ▼
Operator raises ServiceNow incident → BTP Basis resolves
```

---

## 2. Design Decisions

### 2.1 BTP Structuring

| Decision                    | Recommendation                                                                                | Rationale                                                                                                                                             |
| --------------------------- | --------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| Account topology management | Manual Config or Infrastructure as Code using Terraform                                       | Eliminates manual BTP Cockpit clicks; enables version-controlled, repeatable provisioning; critical for multi-Geo rollouts with dozens of subaccounts |
| Subaccount model            | Minimum 3 per workstream (DEV / QA / PROD); add dedicated monitoring and security subaccounts | Isolation of lifecycle; independent entitlement consumption; clean separation of concerns                                                             |
| Directory structure         | Organise by functional workstream (HR Integration, HR Extensions, Platform Operations)        | Enables directory-level entitlement management and cost tracking per business unit                                                                    |
| Terraform state storage     | Remote state in Azure Blob Storage or AWS S3 (not local)                                      | Enables team collaboration; prevents state conflicts; required for CI/CD pipeline integration                                                         |

### 2.2 DevOps & Release Management

| Decision             | Recommendation                                | Rationale                                                                                                        |
| -------------------- | --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Transport tool       | SAP cTMS (mandatory for production promotion) | Only governed tool that provides audit trail, approval gates, and rollback capability for iFlow promotions       |
| Version control      | GitHub or Azure DevOps                        | Industry standard; pull request model enforces peer code review before any change reaches DEV                    |
| ~~CI/CD pipeline~~       | ~~GitHub Actions or Azure DevOps Pipelines~~      | ~~Automates deploy-to-DEV on merge; removes manual designer interaction for routine deployments~~                    |
| cTMS forwarding mode | Manual (not Auto) for QA → PROD               | Auto-forward would bypass approval gates; manual forwarding ensures an explicit human approval before production |

### 2.3 Operations & Monitoring

| Decision                   | Recommendation                                     | Rationale                                                                                                                               |
| -------------------------- | -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| Monitoring tool            | SAP Cloud ALM (not only Integration Suite monitor) | Cloud ALM provides cross-landscape visibility and actionable alerts; Integration Suite's built-in monitor is reactive and single-tenant |
| Alert notification channel | Teams / email via SAP Alert Notification Service   | Real-time notification to the on-call integration operator; avoids manual log-polling                                                   |
| Collection interval        | 5 minutes for PROD; 15 minutes for QA              | PROD failures impact live employees — fast detection is critical for data integrity                                                     |
| Ticket creation            | Automatic ServiceNow ticket from Cloud ALM alert   | Ensures every production failure is tracked and assigned without manual intervention                                                    |

### 2.4 Access Governance

| Decision | Recommendation | Rationale |
|---|---|---|
| Identity Provider | IAS as centralised proxy IdP for all BTP subaccounts | Single place to enforce MFA, conditional access, and lifecycle management; removing an employee from IAS immediately revokes BTP access |
| Access request process | ServiceNow ITSM — no ad-hoc BTP Cockpit grants | Full audit trail of who requested, who approved, when access was granted and revoked |
| Access review | Quarterly review of all BTP role assignments | Prevents accumulation of stale access; required for SOX and ISO 27001 compliance |

---

## 3. BTP Structuring — Terraform IaC

### 3.1 Why Terraform for SAP BTP

SAP provides an official **Terraform Provider for SAP BTP** (available on the Terraform Registry as `SAP/btp`). It enables the full BTP account hierarchy — Global Account, Directories, Subaccounts, Entitlements, Role Collections, and Trust Configuration — to be managed as code, reviewed in Git, and applied automatically through CI/CD.

**Without Terraform:** Each new country roll-out requires a BTP admin to manually click through the cockpit to create a subaccount, assign entitlements, configure trust, and add members — typically 1–2 hours per subaccount and highly error-prone.

**With Terraform:** The same result is achieved in under 5 minutes by running `terraform apply` against a parameterised template, with full auditability in Git.

### 3.2 Core Terraform Resources for BTP

| Terraform Resource | What It Manages |
|---|---|
| `btp_subaccount` | Create and configure a subaccount (name, subdomain, region, description) |
| `btp_directory` | Create and manage directories within the global account |
| `btp_subaccount_entitlement` | Assign a service plan entitlement to a subaccount (e.g., Integration Suite Standard) |
| `btp_subaccount_subscription` | Subscribe to a SaaS application within a subaccount (e.g., Cloud ALM, Audit Log Viewer) |
| `btp_subaccount_service_instance` | Create a service instance (e.g., Destination Service, Connectivity Service) |
| `btp_subaccount_trust_configuration` | Configure custom IdP trust (IAS tenant) for a subaccount |
| `btp_subaccount_role_collection_assignment` | Assign a BTP role collection to a user |

### 3.3 Terraform Configuration — H2R Integration Subaccount

A typical Terraform configuration for an H2R Integration Suite subaccount:

```hcl
# variables.tf
variable "globalaccount" {
  description = "BTP Global Account subdomain"
  type        = string
}

variable "region" {
  description = "BTP region (e.g., eu10, us10)"
  type        = string
  default     = "eu10"
}

variable "org_id" {
  description = "Organisation identifier for naming conventions"
  type        = string
}

variable "ias_tenant_url" {
  description = "IAS tenant URL for trust configuration"
  type        = string
}
```

```hcl
# main.tf — Create HR Integration Directory and Subaccounts
terraform {
  required_providers {
    btp = {
      source  = "SAP/btp"
      version = "~> 1.0"
    }
  }
}

provider "btp" {
  globalaccount = var.globalaccount
}

# Directory: Human Resources
resource "btp_directory" "hr_integration" {
  name        = "Human Resources Integration"
  description = "Contains all HR Integration subaccounts"
  features    = ["ENTITLEMENTS", "AUTHORIZATIONS"]
}

# Subaccount: HR Integration PROD
resource "btp_subaccount" "hr_integration_prd" {
  name        = "HR-Integration-PRD"
  subdomain   = "${var.org_id}-hr-integration-prd"
  region      = var.region
  parent_id   = btp_directory.hr_integration.id
  description = "Production Integration Suite for H2R"
}

# Entitlement: Integration Suite Standard
resource "btp_subaccount_entitlement" "integration_suite_prd" {
  subaccount_id = btp_subaccount.hr_integration_prd.id
  service_name  = "integrationsuite"
  plan_name     = "standard_edition"
}

# Entitlement: Connectivity Service
resource "btp_subaccount_entitlement" "connectivity_prd" {
  subaccount_id = btp_subaccount.hr_integration_prd.id
  service_name  = "connectivity"
  plan_name     = "lite"
}

# Entitlement: Destination Service
resource "btp_subaccount_entitlement" "destination_prd" {
  subaccount_id = btp_subaccount.hr_integration_prd.id
  service_name  = "destination"
  plan_name     = "lite"
}

# Trust Configuration: IAS
resource "btp_subaccount_trust_configuration" "ias_trust_prd" {
  subaccount_id     = btp_subaccount.hr_integration_prd.id
  identity_provider = var.ias_tenant_url
  name              = "IAS Corporate IdP"
  auto_create_shadow_users = true
}
```

### 3.4 Terraform CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/btp-provision.yml
name: Provision SAP BTP Landscape

on:
  push:
    branches: [main]
    paths:
      - 'terraform/**'

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.7.0

      - name: Terraform Init
        run: terraform init
        working-directory: ./terraform
        env:
          ARM_ACCESS_KEY: ${{ secrets.AZURE_STORAGE_ACCESS_KEY }}  # Remote state

      - name: Terraform Plan
        run: terraform plan -out=tfplan
        working-directory: ./terraform
        env:
          BTP_USERNAME: ${{ secrets.BTP_USERNAME }}
          BTP_PASSWORD: ${{ secrets.BTP_PASSWORD }}
          TF_VAR_globalaccount: ${{ secrets.BTP_GLOBALACCOUNT }}
          TF_VAR_ias_tenant_url: ${{ secrets.IAS_TENANT_URL }}

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply tfplan
        working-directory: ./terraform
```

> **Reference:** [IaC for SAP BTP: Automating Environment Management with Terraform and CI/CD](https://community.sap.com/t5/technology-blog-posts-by-sap/iac-for-sap-btp-automating-environment-management-with-terraform-and-ci-cd/ba-p/14330408) | [Automating SAP BTP setup with the new Terraform Provider](https://community.sap.com/t5/technology-blog-posts-by-members/automating-sap-btp-setup-with-the-new-terraform-provider-for-sap-btp/ba-p/13549469) | [Terraform Registry — SAP/btp Provider](https://registry.terraform.io/providers/SAP/btp/latest/docs/resources/subaccount_entitlement)

### 3.5 Real-World Issues — Terraform on SAP BTP

| Issue | Root Cause | Resolution |
|---|---|---|
| **`terraform apply` fails on subaccount creation with "subdomain already exists"** | BTP subaccount subdomains are globally unique across all customers — not just within your Global Account | Use a UUID or organisation prefix in the subdomain (e.g., `a1b2c3d4-hr-integ-prd`); check subdomain availability before provisioning |
| **Entitlement quota not available after `apply` succeeds** | `btp_subaccount_entitlement` creates the entitlement assignment, but if the Global Account has no quota remaining, the resource applies with zero quota | Check Global Account entitlement quota first; add quota at the directory level before assigning to subaccounts |
| **State drift — Terraform state no longer matches BTP reality** | Manual changes made in BTP Cockpit outside of Terraform | Run `terraform refresh` to sync state; enforce a governance rule that all BTP changes go through Terraform — no manual cockpit edits in shared environments |
| **Trust configuration resource fails when IAS tenant is in a different Global Account** | Terraform provider uses the Global Account context; cross-GA trust requires a different setup flow | Use manual trust setup for cross-GA IAS tenants; document this as an exception in the IaC runbook |
| **Circular dependency between entitlement and subscription resources** | `btp_subaccount_subscription` depends on the entitlement being active, but Terraform applies them in parallel | Use `depends_on` to explicitly sequence the subscription after the entitlement resource |

---

## 4. Development

### 4.1 Package / Artifact Structure & Naming Conventions

#### Package Setup
- Every Geo within the organisation will be set up with an **individual Package**

**Package Naming Format:**

| Field | Format |
|---|---|
| Name | `<<Org ID>> <<Geo / Entity Name>>` |
| ID | `<<Org ID>>_<<Geo / Entity Name>>.INTGR.PACK` |

#### Artifact / Integration Naming Format

| Type | Format |
|---|---|
| Global Integrations | `G<<number sequence>>_<<Geo>>_<<Source>>_to_<<Destination>>_<<Process/Data>>` |
| Local Integrations | `L<<number sequence>>_<<Geo>>_<<Source>>_to_<<Destination>>_<<Process/Data>>` |

**Example:** A local CI development team building a custom payroll integration would create it within the relevant Geo package and name it:
`L020_NL_EC_to_ADP_Payroll`

---

### 4.2 Credentials & Certificates Naming Conventions

#### Credentials
- **Format:** `<<Geo>>_<<Resource system>>_<<Resource System instance>>_<<(optional) Additional Details>>`
- **Example:** `NL_SFEC_T1_OAuth`

#### Certificates
- **Format:** `<<Geo>>_<<Resource system>>_<<Resource System instance>>_<<(optional) Additional Details>>_certificate`
- **Example:** `KR_SFEC_T1_SAMLBearer_certificate`

#### PGP Keys
- **Format:** `<<Geo>>_<<Partner>>_<<KeyType>>_pgp`
- **Example:** `NL_ADP_Public_pgp`, `NL_OrgPrivate_Secret_pgp`

---

### 4.3 Rate Limiting

**Principle:** All inbound API connections to Integration Suite must be rate-limited to protect the runtime from unintended or malicious overloading. Rate limiting is enforced at the **API Management** layer using Quota and Spike Arrest policies.

#### Policy Types for Rate Control

| Policy | Purpose | Configuration |
|---|---|---|
| **Quota Policy** | Limits the total number of API calls allowed over a time window (minute / hour / day / month) | Set per application subscription, e.g., 10,000 calls/day for a SuccessFactors Intelligent Services subscription |
| **Spike Arrest Policy** | Prevents burst traffic from overwhelming the runtime; smooths traffic to a maximum rate | Configured as N calls per second/minute regardless of total quota — e.g., `10ps` (10 per second) |
| **ConcurrentRateLimit Policy** | Limits the number of concurrent in-flight requests to the iFlow runtime | Used for iFlows with long-running processing (e.g., mass data load) |

#### Quota Policy Configuration in API Management (Step-by-Step)

1. In SAP Integration Suite → **API Management → APIs** → Open the relevant API proxy
2. Click **Edit** → Navigate to the **PreFlow** (Proxy Endpoint)
3. Click **+ Step** → Select **Quota** policy
4. Configure:
   - **Allow Count:** `1000` (calls allowed per interval)
   - **Interval:** `1`
   - **Time Unit:** `hour`
   - **Identifier:** `request.header.apikey` (rate limit per API key / application)
5. Save and deploy the API proxy

> **Reference:** [SAP API Management — Rate Limiting API Calls](https://community.sap.com/t5/technology-blog-posts-by-sap/sap-api-management-rate-limiting-api-calls-per-application/ba-p/13338781) | [Designing Quota Policy](https://help.sap.com/docs/integration-suite/sap-integration-suite/designing-quota-policy)

---

### 4.4 Role Based Access Control (RBAC)

- The first level of Integration and Data security is provided by **Role Based Access**
- Access to Cloud Integration instances is provided to integration teams based on **BTP Roles**

#### Standard BTP Role Collections

| BTP Role (Cloud Foundry) | Responsibilities |
|---|---|
| **PI_Administrator** | Perform administrative tasks on the tenant; monitor integration flows and artifact status; deploy security content and integration content |
| **PI_Integration_Developer** | Discover, design, and deploy integration artifacts; create, edit, import, export, delete packages and artifacts in the Design workspace; view message processing details |
| **PI_Business_Expert** | Read data that may contain sensitive business content (message payloads / attachments); monitor integration flows and artifact status; read message payload and attachments; manage design-time artifact locks |

---

### 4.5 Identity & Access Management — Access Policies

**Access Policies** provide a second layer of integration and data security on top of RBAC.

- Access policies **restrict access to specific integration packages, artifacts, and the data** processed and stored by them
- You can define access policies to govern an **entire package** or only **certain types of artifacts** within a package

#### Access Policy Dimensions

| Dimension | What It Controls |
|---|---|
| **Design-time artifact** | Operations in the Design section — creating, uploading, editing, saving, deploying, or deleting the artifact |
| **Deployed artifact** | Operations under Monitor → Manage Integration Content — restarting/un-deploying the artifact, changing the log level for monitoring |
| **Data stored and processed at runtime** | Access to data collected during execution; monitoring data (message processing log attachments, business data at Trace log level); data in data stores, variables, and message queues |

#### ⚠️ Important Limitation
Access Policies **cannot** manage Security Material (credentials), KeyStore, or PGP keys. These are managed centrally by the Global Integration team.

#### Supported Artifact Types for Access Policies
Integration Flow · API · OData API · REST API · SOAP API · Script Collection · Value Mapping · Message Mapping · Message Queue · Global Data Store · Global Variable · Data Type · Message Type

---

### 4.6 Access Policy Setup — Step-by-Step

Setting up secure access for multiple Cloud Integration teams on the **same CI instance** involves four steps:

1. **Custom Role Creation**
2. **Custom Role Collection Creation**
3. **Access Policy Creation**
4. **Access Policy Assignment**

#### Step 1 – Custom Role Creation
- The BTP subaccount administrator creates the Custom Role in SAP BTP
- **Naming convention:** `<<Org>>_<<Geo>>_CI_Role`
- **Example:** `ORG_KR_CI_Role`

#### Step 2 – Custom Role Collection Creation
- The BTP subaccount administrator creates the Custom Role Collection in SAP BTP
- **Naming convention:** `<<Org>>_<<Geo>>_CI_Role_Collection`
- **Example:** `ORG_KR_CI_Role_Collection`
- The Role Collection includes the Custom Role, and users are assigned to it

#### Step 3 – Access Policy Creation
- The Integration administrator creates Access Policies in the Cloud Integration tenant
- **Naming convention:** `<<Org>>_<<Geo>>_CI_Access_policy`
- **Example:** `ORG_KR_CI_Access_Policy`
- The Access Policy restricts access to only the artifacts belonging to a specific Integration Package (e.g., `ORG.KR.INTGR.PACK`)

#### Step 4 – Access Policy Assignment
- Access Policies are assigned to Cloud Integration developers in the BTP Subaccount


---

## 4. DevOps & Release Management

### 4.1 SAP Cloud Transport Management (cTMS) — Overview

**SAP Cloud Transport Management (cTMS)** is a BTP service that provides a governed, auditable pipeline for transporting integration artifacts (iFlows, API proxies, Value Mappings, Message Mappings) from DEV through QA to PROD. It replaces manual Export/Import workflows and provides the approval gates needed for SOX and change management compliance.

**What cTMS transports for Integration Suite:**
- iFlow packages (entire package or individual artifacts)
- API Management proxies and products
- Role Collections (via MTA export)
- SAP Build Work Zone content

### 4.2 cTMS Landscape Architecture

```
┌─────────────────────────────────────────────────────────┐
│               cTMS Service Subaccount                    │
│               (BTP-Platform-Operations)                  │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Transport Landscape                 │    │
│  │                                                  │    │
│  │  [DEV Node]──Route A──►[QA Node]──Route B──►[PROD Node] │
│  │  HR-Integ-DEV          HR-Integ-QA          HR-Integ-PRD │
│  │                                                  │    │
│  │  Transport Queue                                 │    │
│  │  DEV Queue ── triggered by developer             │    │
│  │  QA Queue  ── approved by Team Lead              │    │
│  │  PROD Queue ── approved by Integration Manager   │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### 4.3 cTMS Setup — Step-by-Step

**Prerequisites:**
- cTMS service subscribed in the Platform Operations subaccount
- Cloud Integration tenants exist in DEV, QA, and PROD subaccounts
- cTMS service instance created with the `standard` plan; service key created for each environment node

**Step 1: Subscribe to cTMS in the Platform Operations subaccount**
1. BTP Cockpit → `BTP-Platform-Operations` subaccount → **Services > Service Marketplace**
2. Find **Cloud Transport Management** → click **Create** → plan `standard`
3. Navigate to **Instances and Subscriptions > Subscriptions** → subscribe to the **Cloud Transport Management** application

**Step 2: Create transport nodes**
1. Open the cTMS application → **Landscape Wizard** → click **Create Nodes**
2. Create three nodes, one per Integration Suite environment:

| Node Name | Connected Subaccount | Type |
|---|---|---|
| `HR-IS-DEV` | HR-Integration-DEV | Source node |
| `HR-IS-QA` | HR-Integration-QA | Target node |
| `HR-IS-PROD` | HR-Integration-PRD | Target node |

3. For each node, select **SAP Cloud Integration** as the content type and provide the service key from the Integration Suite subaccount

**Step 3: Create transport routes**
1. In Landscape Wizard → **Routes** → click **Add Route**
2. Route A: Source = `HR-IS-DEV`, Target = `HR-IS-QA`, Forwarding Mode = **Manual**
3. Route B: Source = `HR-IS-QA`, Target = `HR-IS-PROD`, Forwarding Mode = **Manual**

**Step 4: Configure Integration Suite to use cTMS**
1. In Integration Suite (DEV) → **Settings → Integrations → Transport**
2. Set Transport Mode to **Transport Management Service**
3. Enter the cTMS service key credentials

**Step 5: Transport an iFlow package**
1. In Integration Suite Design, open the package → click **Transport** (three-dot menu)
2. Select **Add to Transport Request** → a new transport request appears in cTMS DEV queue
3. In cTMS → DEV Queue → select the request → click **Forward to QA**
4. QA Team Lead reviews, tests → in cTMS QA Queue → **Import** to QA
5. After QA sign-off → QA Queue → **Forward to PROD** → Integration Manager approves → **Import** to PROD

> **Reference:** [Step By Step Configuring CTMS Setup in BTP](https://community.sap.com/t5/technology-blog-posts-by-members/step-by-step-configuring-ctms-setup-in-btp/ba-p/14119106) | [SAP Cloud Transport Management FAQ](https://pages.community.sap.com/topics/devops/cloud-transport-management-faq)

### 4.4 Version Control & Backup — GitHub + FlashPipe

FlashPipe runs as a scheduled GitHub Actions job to pull all iFlow artifacts from Integration Suite DEV and commit them to Git. This creates a continuous backup and enables peer code review via Pull Requests before any change is deployed.

**Repository structure:**

```
/integration-repo
  /packages
    /ORG_Netherlands
      /iflows
        G010_NL_EC_to_S4_Payroll/
          iflow.iflw
          parameters.prop
          src/main/resources/...
      /valuemappings
      /scriptcollections
    /ORG_Korea
      ...
  /apim
    /proxies
      SF_EC_OData_Proxy/
  /terraform
    main.tf
    variables.tf
    outputs.tf
```

**GitHub Actions workflow — FlashPipe backup:**

```yaml
# .github/workflows/flashpipe-backup.yml
name: FlashPipe - Backup Integration Artifacts

on:
  schedule:
    - cron: '0 2 * * *'   # Every day at 02:00 UTC

jobs:
  backup:
    runs-on: ubuntu-latest
    container:
      image: engswee/flashpipe:latest

    steps:
      - uses: actions/checkout@v3
        with:
          token: ${{ secrets.GH_TOKEN }}

      - name: Sync artifacts from Integration Suite DEV
        env:
          HOST_TMN: ${{ secrets.CI_DEV_HOST }}
          OAUTH_HOST: ${{ secrets.CI_DEV_OAUTH_HOST }}
          OAUTH_CLIENTID: ${{ secrets.CI_DEV_CLIENT_ID }}
          OAUTH_CLIENTSECRET: ${{ secrets.CI_DEV_CLIENT_SECRET }}
        run: |
          flashpipe sync apim --dir apim/proxies
          flashpipe sync iflow --package-id ORG_Netherlands.INTGR.PACK \
            --dir packages/ORG_Netherlands

      - name: Commit and push changes
        run: |
          git config user.email "ci-bot@company.com"
          git config user.name "FlashPipe CI"
          git add .
          git diff --staged --quiet || git commit -m "chore: daily backup $(date +%Y-%m-%d)"
          git push
```

### 4.5 Real-World Issues — DevOps & Release Management

| Issue | Root Cause | Resolution |
|---|---|---|
| **cTMS import to PROD fails with "Content mismatch" error** | The iFlow was manually edited directly in the QA or PROD Integration Suite designer after the transport was created, causing a version mismatch | Enforce a policy: no direct designer edits in QA/PROD; all changes must come through cTMS transport from DEV |
| **FlashPipe backup job fails silently — no new commits for days** | GitHub Actions scheduled jobs can become inactive if there is no repository activity for 60 days; or the CI DEV OAuth token has rotated | Set up a monitoring alert on the GitHub Actions workflow; rotate secrets proactively; add a "heartbeat" commit to keep the schedule active |
| **cTMS transport queue accumulates unreleased requests** | Developers create transport requests but forget to forward them; no expiry mechanism | Establish a sprint-end cleanup ritual: all transport requests must be either forwarded or deleted before sprint close; Cloud ALM can be configured to alert on stale queues |
| **Two developers transport the same package simultaneously, causing a conflict in QA** | cTMS does not prevent concurrent transports of the same package by different developers | Implement a "transport lock" process: developers announce in the Teams channel before transporting; cTMS import is sequential (last import wins) |
| **FlashPipe creates a massive commit with hundreds of changed files when a Value Mapping table is refreshed** | Value Mapping exports include binary data that changes on every export even with no logical changes | Exclude Value Mapping artifacts from FlashPipe sync and manage them separately; or use `.gitattributes` to mark Value Mapping files as binary and suppress diff noise |

---

## 5. Operations — SAP Cloud ALM Integration Monitoring

### 5.1 What Cloud ALM Provides for Integration Operations

SAP Cloud ALM acts as the **central operations cockpit** for the entire BTP integration landscape. Rather than requiring operators to log into each Integration Suite subaccount separately to check message processing logs, Cloud ALM aggregates health and alert data from all registered systems into a single dashboard.

### 5.2 Monitoring Architecture

```
Integration Suite DEV (HR-Integ-DEV)
Integration Suite QA  (HR-Integ-QA)          All push health
Integration Suite PRD (HR-Integ-PRD)  ──────► metrics and alerts
                                               │
                                               ▼
                                   ┌──────────────────────┐
                                   │   SAP Cloud ALM       │
                                   │   (Operations)        │
                                   │                       │
                                   │  ┌─────────────────┐  │
                                   │  │ Integration &    │  │
                                   │  │ Exception Mon.   │  │
                                   │  └────────┬────────┘  │
                                   │           │            │
                                   │  ┌────────▼────────┐  │
                                   │  │  Alert Mgmt     │  │
                                   │  │  Inbox          │  │
                                   │  └────────┬────────┘  │
                                   └───────────┼────────────┘
                                               │
                       ┌───────────────────────┼───────────────────┐
                       ▼                       ▼                   ▼
               Email Notification       Teams Webhook        ServiceNow Ticket
               (integration-ops         (#btp-alerts)        (auto-created INC)
                @company.com)
```

### 5.3 Cloud ALM Setup for Integration Suite — Step-by-Step

**Prerequisites:**
- SAP Cloud ALM tenant provisioned (available as part of SAP Enterprise Support)
- Integration Suite subaccounts registered in the SAP BTP cockpit

**Step 1: Connect Integration Suite to Cloud ALM (Landscape Management)**

1. In SAP Cloud ALM → **Administration → Landscape Management**
2. Click **Add System** → select **SAP Integration Suite**
3. Provide the Integration Suite service key (from the `it-rt` service instance in the subaccount)
4. Verify the connection status shows **Connected**
5. Repeat for all environments (DEV, QA, PROD)

**Step 2: Activate Integration & Exception Monitoring use case**

1. In Cloud ALM → **Operations → Integration & Exception Monitoring**
2. Click the settings icon (top-right) → **Select Services**
3. Check each connected Integration Suite instance
4. Enable monitoring categories:
   - ✅ Failed Messages
   - ✅ Retry Messages
   - ✅ Anomaly Detection
5. Click **Save**

**Step 3: Configure alert thresholds**

1. In Integration & Exception Monitoring → **Alert Configuration**
2. Set thresholds per environment:

| Environment | Alert Condition | Threshold |
|---|---|---|
| PROD | Failed messages | > 1 in any 5-minute window |
| PROD | Zero messages (anomaly) | Expected message count drops to 0 during business hours |
| QA | Failed messages | > 5 in any 15-minute window |

**Step 4: Configure notification channels**

1. In Cloud ALM → **Administration → Notification Management**
2. Add notification channels:
   - **Email:** `integration-ops@company.com`
   - **Microsoft Teams Webhook:** `https://company.webhook.office.com/...`
3. Map channels to alert severity levels (Critical → Teams + Email; Warning → Email only)

**Step 5: Enable automatic ServiceNow ticket creation**

1. In Cloud ALM → **Administration → Ticketing Integration**
2. Connect to ServiceNow instance (ServiceNow OAuth credentials)
3. Configure auto-ticket rule: if alert severity = **Critical** and environment = **PROD** → auto-create INC with priority P2

> **Reference:** [How to integrate Cloud ALM with Integration Suite — Easy and Updated Steps](https://community.sap.com/t5/technology-blog-posts-by-sap/how-to-integrate-cloud-alm-with-integration-suite-easy-and-updated-steps/ba-p/14262529) | [Integration & Exception Monitoring — Setup & Configuration](https://support.sap.com/en/alm/sap-cloud-alm/operations/expert-portal/integration-monitoring/int-mon-setup-support.html) | [SAP Integration Suite in Cloud ALM](https://support.sap.com/en/alm/sap-cloud-alm/operations/expert-portal/integration-monitoring/calm-cpi.html)

### 5.4 Real-World Issues — Cloud ALM Operations

| Issue | Root Cause | Resolution |
|---|---|---|
| **Cloud ALM shows a system as "Connected" but no messages appear in monitoring** | The data collection agent requires the `MonitoringDataConsumer` role collection to be assigned to the Cloud ALM technical user in the Integration Suite subaccount | Verify the Cloud ALM technical user in the BTP subaccount has `MonitoringDataConsumer` and `MonitoringDataRead` role collections assigned |
| **Anomaly detection triggers false positives every weekend** | The anomaly model is trained on historical message volumes, including business hours; weekend silence looks like an anomaly | Configure anomaly detection schedules to exclude weekend hours; or set a minimum threshold floor of 0 for off-hours periods |
| **Alert emails for PROD failures arrive 20–30 minutes after the failure** | Default Cloud ALM data collection interval is set to 15 minutes in the initial setup | Reduce the collection interval to 5 minutes for PROD systems in the monitoring service configuration |
| **ServiceNow auto-created tickets have no useful description** | Default ticket template uses generic Cloud ALM field mappings; doesn't include iFlow name, error message, or tenant | Customise the ServiceNow integration mapping in Cloud ALM to include `alert.resourceName` (iFlow name), `alert.description`, and `alert.serviceInstance` in the INC short description |
| **Cloud ALM loses connection to Integration Suite after BTP subaccount trust reconfiguration** | The Cloud ALM service key (OAuth client) is bound to the trust configuration; changing or rotating the IAS trust invalidates the existing service key | Re-create the `it-rt` service key after any trust reconfiguration; update the service key in Cloud ALM Landscape Management; document this dependency in the runbook |

---

## 6. Access Governance

### 6.1 IAS as the Centralised Identity Provider

SAP Identity Authentication Service (IAS) must be configured as the **primary proxy Identity Provider** for all BTP subaccounts in the integration landscape. This ensures a single, consistent authentication and access lifecycle across all environments.

**Why IAS must be the proxy (not just a corporate AD directly):**

| Requirement | How IAS Fulfils It |
|---|---|
| MFA enforcement | IAS enforces MFA policies (TOTP, FIDO2) for all BTP access regardless of the corporate AD's capabilities |
| Risk-based authentication | IAS can step up to stronger authentication for sensitive actions (e.g., deploying to PROD) |
| Single place for lifecycle management | Deactivating a user in IAS immediately cuts access to all federated applications (BTP, SuccessFactors, S/4HANA Cloud) |
| Group-based BTP role assignment | IAS groups drive BTP role collection mapping; group membership = BTP access |
| Decouples BTP from corporate AD changes | AD migrations or restructuring don't require changes in each BTP subaccount — only in the IAS corporate IdP configuration |

### 6.2 Access Governance via ServiceNow

All access requests to BTP entitlements flow through a ServiceNow ITSM process. No ad-hoc cockpit-level role assignments are permitted in shared environments.

#### Access Request Workflow

```
1. Integration Developer
   │  Raises ServiceNow RITM (Request Item):
   │  System: SAP BTP / Integration Suite DEV
   │  Role requested: PI_Integration_Developer
   │  Justification: Sprint 12 — NL payroll integration
   │
   ▼
2. Line Manager
   │  Approves or rejects in ServiceNow
   │
   ▼
3. BTP Basis Admin
   │  Assigns role collection in BTP Cockpit
   │  Closes RITM with fulfilment notes
   │
   ▼
4. Quarterly Access Review (Team Lead)
   │  Reviews all active assignments via ServiceNow report
   │  Initiates revocation for stale access
   │
   ▼
5. Off-boarding (triggered by HR termination in SuccessFactors)
   Automatic ServiceNow off-boarding task → BTP Basis removes
   all role collections and subaccount membership within 24 hours
```

#### Access Request — ServiceNow RITM Fields

| Field | Value | Notes |
|---|---|---|
| Request Type | `BTP Access` | Used to route to BTP Basis assignment group |
| System | `SAP BTP — [Subaccount Name]` | Identifies which subaccount/environment |
| Role Collection | `PI_Integration_Developer` | Select from approved catalogue |
| Duration | `Permanent` / `Time-bound (end date)` | Time-bound for project contractors |
| Justification | Free text | Required field — must reference project or ticket |
| Approver | Line Manager (auto-populated from HR data) | |

### 6.3 Real-World Issues — Access Governance

| Issue | Root Cause | Resolution |
|---|---|---|
| **Developer has `PI_Administrator` in PROD from a previous go-live — never revoked** | No quarterly access review was conducted; no automated off-boarding for role downgrades post-go-live | Implement quarterly access certification in ServiceNow; run a `PI_Administrator` audit report after every go-live and immediately revoke elevated access once go-live is stable |
| **IAS group change doesn't propagate to BTP role collection — user still has access** | BTP role collection mapping from IAS groups is evaluated at login time, not in real-time. Removing a user from an IAS group revokes access on their next login, not immediately | For immediate revocation (e.g., disciplinary case), remove the user from the BTP subaccount directly in addition to removing from the IAS group; do not wait for next login |
| **ServiceNow RITM auto-assigns to the wrong team** | The assignment group logic uses the "System" field, but BTP subaccount names don't match the ServiceNow CMDB entries | Maintain a mapping table in ServiceNow CMDB with exact BTP subaccount names mapped to the correct support group; keep it updated after each new subaccount is created via Terraform |
| **New contractor onboarded without BTP access for 3 days — blocking integration development** | ServiceNow approval queue has a 24-hour SLA but manager was on leave; no escalation path | Define an escalation path in the RITM workflow: if no manager approval within 8 business hours, escalate to the next-level manager; add a "delegate approver" field for leave periods |
| **Shadow user creation fails in BTP for a new employee — cannot log in** | The IAS trust configuration in the BTP subaccount has "Create Shadow Users During Logon" disabled | Verify this setting is enabled: BTP Cockpit → Subaccount → Security → Trust Configuration → IAS entry → ensure **Create Shadow Users During Logon** = Enabled |

---

## 7. Reference Links

| Topic | Link |
|---|---|
| Terraform Provider for SAP BTP — Registry | [registry.terraform.io — SAP/btp](https://registry.terraform.io/providers/SAP/btp/latest) |
| IaC for SAP BTP: Terraform and CI/CD | [SAP Community — IaC BTP Terraform CI/CD](https://community.sap.com/t5/technology-blog-posts-by-sap/iac-for-sap-btp-automating-environment-management-with-terraform-and-ci-cd/ba-p/14330408) |
| Automating SAP BTP with Terraform | [SAP Community — Terraform BTP Setup](https://community.sap.com/t5/technology-blog-posts-by-members/automating-sap-btp-setup-with-the-new-terraform-provider-for-sap-btp/ba-p/13549469) |
| Extending IaC for SAP BTP | [SAP Community — Extending IaC for BTP](https://community.sap.com/t5/technology-blog-posts-by-sap/extending-infrastructure-as-code-for-sap-btp-managing-resources-on-sap-btp/ba-p/14062455) |
| SAP Cloud Transport Management FAQ | [SAP Community — cTMS FAQ](https://pages.community.sap.com/topics/devops/cloud-transport-management-faq) |
| Step By Step cTMS Setup in BTP | [SAP Community — cTMS Step-by-Step](https://community.sap.com/t5/technology-blog-posts-by-members/step-by-step-configuring-ctms-setup-in-btp/ba-p/14119106) |
| Cloud ALM Integration Suite Setup | [SAP Community — Cloud ALM + IS Setup](https://community.sap.com/t5/technology-blog-posts-by-sap/how-to-integrate-cloud-alm-with-integration-suite-easy-and-updated-steps/ba-p/14262529) |
| Integration & Exception Monitoring — Setup | [SAP Support — Int. Monitoring Setup](https://support.sap.com/en/alm/sap-cloud-alm/operations/expert-portal/integration-monitoring/int-mon-setup-support.html) |
| Cloud ALM for Operations | [SAP Support — Cloud ALM Operations](https://support.sap.com/en/alm/sap-cloud-alm/operations.html) |
| FlashPipe (open-source) | [FlashPipe GitHub](https://engswee.github.io/flashpipe/) |
| SAP BTP Account Model | [help.sap.com — BTP Account Model](https://help.sap.com/docs/btp/sap-business-technology-platform/account-model) |

---

*Document Version: 1.0 | Scope: SAP BTP Infrastructure Management | Last Updated: June 2026*
