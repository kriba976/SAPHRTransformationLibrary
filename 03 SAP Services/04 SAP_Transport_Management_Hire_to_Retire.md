# SAP Cloud Transport Management System (cTMS) — Hire-to-Retire Technical Reference

> **Scope:** This document covers SAP Cloud Transport Management (cTMS), specifically for managing the lifecycle and promotion of SAP Integration Suite artifacts (iFlows, Value Mappings, etc.) across DEV, QA, and Production environments in the context of the **Hire-to-Retire** business process. It covers procurement, architecture, design decisions, and configuration.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Design Decisions](#2-design-decisions)
3. [Procurement and Licensing](#3-procurement-and-licensing)
4. [How cTMS Fits in an H2R Transformation Project](#4-how-ctms-fits-in-an-h2r-transformation-project)
5. [Provisioning cTMS](#5-provisioning-ctms)
6. [Setting Up Transport Nodes for H2R](#6-setting-up-transport-nodes-for-h2r)
7. [Setting Up Transport Routes](#7-setting-up-transport-routes)
8. [Configuring Integration Suite for cTMS Export](#8-configuring-integration-suite-for-ctms-export)
9. [Transporting H2R iFlows — Step-by-Step Workflow](#9-transporting-h2r-iflows--step-by-step-workflow)
10. [Externalized Parameters and Environment-Specific Configuration](#10-externalized-parameters-and-environment-specific-configuration)
11. [SAP Solution Manager Integration (Optional)](#11-sap-solution-manager-integration-optional)
12. [CI/CD Integration with cTMS](#12-cicd-integration-with-ctms)
13. [Operations and Governance](#13-operations-and-governance)
14. [Reference Links](#14-reference-links)

---

## 1. Architecture Overview

### 1.1 cTMS in the H2R Landscape

SAP Cloud Transport Management (cTMS) is a BTP-native service that provides controlled, auditable promotion of cloud artifacts from lower to higher environments. In H2R, the primary artifact type transported is **SAP Integration Suite content** (iFlows, Value Mappings, XSLT scripts, integration packages).

```
┌────────────────────────────────────────────────────────────────────────┐
│                    H2R Transport Landscape                              │
│                                                                          │
│  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐  │
│  │  H2R-DEV        │     │  H2R-QA         │     │  H2R-PRD        │  │
│  │  Subaccount     │     │  Subaccount     │     │  Subaccount     │  │
│  │                 │     │                 │     │                 │  │
│  │  Integration    │     │  Integration    │     │  Integration    │  │
│  │  Suite DEV      │     │  Suite QA       │     │  Suite PRD      │  │
│  │                 │     │                 │     │                 │  │
│  │  Developer      │     │  Test Team      │     │  Operations     │  │
│  │  Builds iFlows  │     │  Validates      │     │  Runs Production│  │
│  └────────┬────────┘     └────────┬────────┘     └─────────────────┘  │
│           │                       │                         ▲           │
│           │ Export via            │ Promote to             │            │
│           │ Content Agent         │ PRD (cTMS)             │            │
│           ▼                       ▼                         │           │
│  ┌─────────────────────────────────────────────────────────┐           │
│  │                 SAP Cloud Transport Management           │           │
│  │                                                          │           │
│  │  Transport Node: DEV ──Route──► QA ──Route──► PRD       │           │
│  │                                                          │           │
│  │  Transport Request (package zip)                        │           │
│  │  - iFlows                                               │           │
│  │  - Value Mappings                                       │           │
│  │  - XSLT resources                                       │           │
│  │  - Integration Packages                                 │           │
│  └─────────────────────────────────────────────────────────┘           │
└────────────────────────────────────────────────────────────────────────┘
```

### 1.2 What Can Be Transported for H2R

| Artifact Type | Transportable via cTMS | Notes |
|---|---|---|
| Integration Packages | ✓ | Full package with all contained artifacts |
| Individual iFlows | ✓ | Selective artifact transport |
| Value Mappings | ✓ | Country code tables, HR code mappings |
| Message Mapping artifacts | ✓ | Included in package |
| XSLT resources | ✓ | Included in iFlow resources |
| Externalized Parameter values | ✗ | Environment-specific — managed separately |
| Security Material (certs, credentials) | ✗ | Must be configured per environment manually |
| Destinations (BTP) | ✗ | Must be configured per subaccount manually |

---

## 2. Design Decisions

### 2.1 Governance Model

| Artifact State | Who Can Promote | Approval Required |
|---|---|---|
| DEV → QA | Integration Developer | No (auto-forward) |
| QA → PRD | Integration Lead | Yes — HR Functional Lead + Integration Architect |
| Emergency PRD fix | Senior Developer only | Yes — Change Advisory Board (CAB) |
| Rollback in PRD | Integration Lead | Yes — CAB |

---

## 3. Procurement and Licensing

### 3.1 How to Get cTMS

SAP Cloud Transport Management is available as a BTP service:

| Procurement Path | Access |
|---|---|
| **SAP BTP (CPEA/BTPEA)** | Available as a free service plan (`lite`) within BTP credits |
| **BTP Free Tier** | Available under free tier limits |
| **Standalone subscription** | Can be subscribed to directly in BTP marketplace |

The `lite` service plan is sufficient for H2R transport needs. It includes:
- Unlimited transport nodes and routes
- Unlimited transport requests
- Integration with Content Agent Service for Integration Suite

### 3.2 Required Entitlements

In your BTP Global Account, ensure the following entitlements are configured:

| Service | Plan | Required In |
|---|---|---|
| **Cloud Transport Management** | `lite` | The subaccount where cTMS is hosted (typically a dedicated "transport" subaccount or the DEV subaccount) |
| **Content Agent Service** | `application` | Each source subaccount (DEV, and optionally QA for QA → PRD) |
| **Content Agent Service** | `standard` (service plan) | Each source subaccount (for service instance/key creation) |
| **Destination Service** | `lite` | Each subaccount (DEV, QA, PRD) — for cTMS destinations |

---

## 5. Provisioning cTMS

### 5.1 Subscribe to Cloud Transport Management

1. BTP Cockpit → Navigate to the **Transport subaccount** (or use DEV subaccount)
2. **Services > Service Marketplace** → Search for **Cloud Transport Management**
3. Click **Create** → Plan: `lite`
4. After subscription, click **Go to Application** to open the cTMS UI

### 5.2 Assign cTMS Role Collections

In the subaccount hosting cTMS, assign roles to team members:

| Role Collection | Purpose | Assign To |
|---|---|---|
| `TMS_LandscapeOperator` | Manage transport landscape (nodes, routes) | Integration Architect / Lead |
| `TMS_ImportOperator` | Execute transport imports | Integration Lead (for QA→PRD approvals) |
| `TMS_TransportOperator` | Create and forward transport requests | Integration Developers |
| `TMS_Viewer` | Read-only access to transport requests | Project Manager, HR Functional Lead |

### 5.3 Create Destinations in Each Target Subaccount

cTMS needs **Destination** entries in the DEV subaccount that point to the QA and PRD subaccounts' deployment endpoints.

For each target environment (QA, PRD), create a destination in the **source/transport subaccount**:

| Property | Value |
|---|---|
| Name | `ctms-qa-is` (for QA) / `ctms-prd-is` (for PRD) |
| Type | `HTTP` |
| URL | `https://<IS_tenant>.integrationsuite.cfapps.<region>.hana.ondemand.com` |
| Authentication | `OAuth2ClientCredentials` |
| Token Service URL | `https://<subaccount>.authentication.<region>.hana.ondemand.com/oauth/token` |
| Client ID / Secret | From Service Key of **Content Agent** instance in the target subaccount |

---

## 6. Setting Up Transport Nodes for H2R

### 6.1 What is a Transport Node?

A **Transport Node** in cTMS represents one environment endpoint — typically one Integration Suite tenant. For H2R, you create three nodes: DEV, QA, PRD.

### 6.2 Creating Transport Nodes

1. Open the cTMS application
2. Navigate to: **Landscape > Transport Nodes**
3. Click **Create**

#### DEV Node Configuration

| Parameter | Value |
|---|---|
| **Name** | `H2R-IS-DEV` |
| **Description** | `H2R Integration Suite DEV` |
| **Allow Upload** | ✓ **Yes** — content is exported TO this node first |
| **Forward Mode** | `Auto` — auto-forward to QA after export |
| **Controlled by SAP Solution Manager** | ✗ (unless you have ChaRM) |

#### QA Node Configuration

| Parameter | Value |
|---|---|
| **Name** | `H2R-IS-QA` |
| **Description** | `H2R Integration Suite QA` |
| **Allow Upload** | ✗ No — QA receives content via transport route only |
| **Forward Mode** | `Manual` — require approval before forwarding to PRD |
| **Deployment Target** | Select destination: `ctms-qa-is` |
| **Controlled by SAP Solution Manager** | ✓ Yes (if using ChaRM for QA→PRD change control) |

#### PRD Node Configuration

| Parameter | Value |
|---|---|
| **Name** | `H2R-IS-PRD` |
| **Description** | `H2R Integration Suite PRD` |
| **Allow Upload** | ✗ No — PRD only receives content via transport |
| **Forward Mode** | `Manual` |
| **Deployment Target** | Select destination: `ctms-prd-is` |
| **Controlled by SAP Solution Manager** | ✓ Yes (if using ChaRM) |

---

## 7. Setting Up Transport Routes

### 7.1 What is a Transport Route?

A **Transport Route** defines the path that a transport request travels — from one node to the next. For H2R:
- Route 1: `H2R-IS-DEV` → `H2R-IS-QA`
- Route 2: `H2R-IS-QA` → `H2R-IS-PRD`

### 7.2 Creating Transport Routes

1. cTMS → **Landscape > Transport Routes**
2. Click **Create**

#### Route 1: DEV → QA

| Parameter | Value |
|---|---|
| **Name** | `H2R-DEV-to-QA` |
| **Source Node** | `H2R-IS-DEV` |
| **Target Node** | `H2R-IS-QA` |

#### Route 2: QA → PRD

| Parameter | Value |
|---|---|
| **Name** | `H2R-QA-to-PRD` |
| **Source Node** | `H2R-IS-QA` |
| **Target Node** | `H2R-IS-PRD` |

### 7.3 Visualizing the Transport Landscape

After creating nodes and routes, the cTMS Landscape view shows:

```
[H2R-IS-DEV] ──Route 1──► [H2R-IS-QA] ──Route 2──► [H2R-IS-PRD]
  Allow Upload                Manual Fwd              Manual Fwd
  Auto-forward                (QA approval)           (CAB approval)
```

---

## 8. Configuring Integration Suite for cTMS Export

### 8.1 Content Agent Service Setup

The **Content Agent Service** is the bridge between Integration Suite and cTMS. It packages Integration Suite content and uploads it to the cTMS DEV node.

#### Step 1: Subscribe to Content Agent Service in DEV Subaccount

1. BTP Cockpit → DEV Subaccount → **Service Marketplace**
2. Search **Content Agent Service** → Create instance with plan `standard`
3. Create a **Service Key** → download the JSON (needed for integration with cTMS)

#### Step 2: Configure Content Agent in Integration Suite

1. In Integration Suite → **Settings > Transport**
2. Click **Enable**
3. Under **Transport Mode**: Select **Transport Management Service (TMS)**
4. Configure:
   - **TMS Service Key**: Paste the cTMS service key
   - **TMS Node Name**: `H2R-IS-DEV` (the DEV node in cTMS)
5. Save

Now Integration Suite can export packages directly to cTMS.

### 8.2 Role Required for Transport in Integration Suite

The user performing the export needs the role `PI_Integration_Developer` AND the transport action must be enabled. Add the `TransportExport` permission if using custom role collections.

---

## 9. Transporting H2R iFlows — Step-by-Step Workflow

### 9.1 Step 1: Complete Development in DEV

1. Developer builds and unit-tests the H2R iFlow in `HR-Integration-DEV` subaccount
2. All externalized parameters are set for the DEV environment
3. iFlow is deployed and tested end-to-end in DEV

### 9.2 Step 2: Export to cTMS (DEV → cTMS Node)

1. In Integration Suite (DEV) → **Design > Packages**
2. Select the H2R package (e.g., `H2R_Employee_Replication`)
3. Click the **Transport** icon (truck icon) → **Transport to TMS**
4. Provide a description: `New Hire iFlow v1.3 - Added delta timestamp support`
5. Click **OK**

What happens:
- Content Agent packages the iFlow (and all artifacts in the package) into a ZIP
- The ZIP is uploaded to the `H2R-IS-DEV` node in cTMS as a **Transport Request**
- Since DEV node is set to Auto-forward, the request is immediately added to the `H2R-IS-QA` node's import queue

### 9.3 Step 3: Import into QA

Since DEV → QA auto-forward is enabled:
1. The transport request appears automatically in the QA node's import queue
2. cTMS automatically deploys (imports) the package into the QA Integration Suite tenant
3. No manual action needed for DEV → QA

To verify:
- cTMS → **Landscape > Transport Requests** → filter by node `H2R-IS-QA` → Status should be `Succeeded`

### 9.4 Step 4: Configure QA Environment Parameters

After import, the iFlow is deployed to QA IS but needs QA-specific configuration:

1. Open QA Integration Suite → **Design > Packages > H2R_Employee_Replication**
2. Click the imported iFlow → **Configure**
3. Update externalized parameters:
   - `SF_EC_API_URL` → QA SF EC endpoint
   - `ERP_HCM_Virtual_Host` → QA ERP virtual hostname
   - `SF_OAuth_Alias` → QA credential alias
4. **Deploy** the iFlow in QA

> The transport brings the iFlow structure and logic; environment-specific values must always be set after transport.

### 9.5 Step 5: Testing in QA

1. Test team executes the H2R integration test scenarios
2. Test results are documented in the change ticket / test management tool
3. Functional HR lead signs off on test results

### 9.6 Step 6: Approve and Promote to PRD

1. In cTMS → **Landscape > H2R-IS-QA Node > Import Queue**
2. Select the validated transport request
3. Click **Forward to PRD** (this moves it to `H2R-IS-PRD` import queue)

OR, if using ChaRM (SAP Solution Manager):
1. A change request is created in ChaRM
2. The status change in ChaRM triggers the cTMS import into PRD automatically

4. Integration Lead / PRD import operator clicks **Import** in the PRD node import queue
5. cTMS deploys the package into the PRD Integration Suite tenant

### 9.7 Step 7: Configure PRD Environment Parameters

Same as Step 4 — update PRD-specific externalized parameters and deploy the iFlow in PRD IS.

### 9.8 Step 8: Go-Live Verification

1. Trigger a test event in SF EC (or use a test employee in PRD with care)
2. Confirm message processed successfully in Integration Suite PRD monitor
3. Confirm data appears in ERP HCM PRD
4. Cloud ALM shows green status for the H2R scenario

---

## 10. Externalized Parameters and Environment-Specific Configuration

### 10.1 The Challenge

When an iFlow is transported from DEV to QA to PRD, the following must change per environment:
- SF EC API URL (different SF tenant per environment)
- OAuth credential aliases (DEV/QA/PRD have separate credentials)
- ERP HCM virtual hostnames (different Cloud Connector virtual hosts)
- SFTP server credentials and hostnames
- Company IDs (if testing in a different SF company)

cTMS transports the **iFlow definition** — not the parameter values. Parameter values must be managed separately.

### 10.2 Recommended Pattern: Externalized Parameters + Configuration Document

**In the iFlow:** Parameterize all environment-specific values:
```
SF_API_URL        = {{SF_EC_API_URL}}
SF_OAuth_Alias    = {{SF_OAuth_Credential_Alias}}
ERP_VirtHost      = {{ERP_HCM_Virtual_Host}}
SFTP_Alias        = {{SFTP_Payroll_Credential_Alias}}
Company_ID        = {{SF_Company_ID}}
```

**Maintain a configuration document** (Excel or a BTP Configuration Management tool) per environment:

| Parameter | DEV Value | QA Value | PRD Value |
|---|---|---|---|
| `SF_EC_API_URL` | `apisf-dev.sap.com/odata/v2` | `apisf-qa.sap.com/odata/v2` | `apisf.sap.com/odata/v2` |
| `SF_OAuth_Credential_Alias` | `sf_oauth_dev` | `sf_oauth_qa` | `sf_oauth_prd` |
| `ERP_HCM_Virtual_Host` | `erp-hcm-dev:33300` | `erp-hcm-qa:33300` | `erp-hcm-prd:33300` |
| `SF_Company_ID` | `COMPANY_DEV` | `COMPANY_QA` | `COMPANY001` |

After each cTMS import, the **deployment engineer** applies the correct parameter values for that environment.


---

## 11. Operations and Governance

### 11.1 Transport Request Naming Convention

Enforce a naming convention for all H2R transport requests:

```
H2R_<Domain>_<IFlowName>_<Version>_<Description>
Examples:
  H2R_Employee_NewHireToERP_v1.3_AddedDeltaTimestamp
  H2R_Payroll_SFTPExport_v2.1_PGPEncryptionEnabled
  H2R_OrgSync_ERPtoEC_v1.0_InitialRelease
```

### 11.2 Rollback Procedure

If a transport causes issues in PRD:
1. cTMS → **Landscape > H2R-IS-PRD > Import History**
2. Find the previous successful transport request
3. Click **Re-Import** to re-deploy the previous version
4. In PRD Integration Suite: verify the older iFlow version is active
5. Undeploy the broken version
6. Notify HR operations of temporary rollback

### 11.3 Transport Audit Log

cTMS maintains a full audit log of all transport actions:
- Who exported from DEV
- When it was imported into QA
- Who approved the QA → PRD promotion
- When it was deployed to PRD
- Whether deployment succeeded or failed

This audit log is accessible under: **cTMS > Landscape > Transport Requests > History**

For compliance requirements (SOX, GDPR), the audit log demonstrates that H2R integration changes followed an approved change management process before touching production HR data.

---

## 12. Common Pitfalls & Real-World Issues

### Transport Landscape Setup

| Issue | Root Cause | Resolution |
|---|---|---|
| **cTMS transport node shows "Authentication Failed" when connecting to the Integration Suite subaccount** | The service key used for the transport node was created from the wrong service instance (e.g., `process-integration-runtime` instead of `it-rt`) or the OAuth credentials have rotated | Verify the service key is from the `it-rt` instance in the target Integration Suite subaccount; re-create the node with fresh credentials if rotated |
| **Landscape Wizard does not show the Integration Suite subaccount as an available node target** | The cTMS subscription is in a different BTP Global Account from the Integration Suite, or the CF environment is not enabled on the target subaccount | cTMS must be in the same Global Account; verify CF is enabled on each target subaccount; ensure the cTMS service instance has the appropriate BTP entitlements |
| **Transport routes were created but packages fail to forward — "No route found" error** | The route was defined in the wrong direction (Target → Source instead of Source → Target) | Re-check the route configuration in the Landscape Wizard; Source must be the lower environment (DEV) and Target must be the higher environment (QA/PROD) |

### Transport Operations

| Issue | Root Cause | Resolution |
|---|---|---|
| **Import to PROD fails with "Content version mismatch"** | The iFlow was manually edited directly in the QA or PROD Integration Suite designer after the transport was created, creating a divergence between what cTMS holds and what is deployed | Enforce a zero-tolerance policy for direct edits in QA/PROD environments; all changes must originate from DEV and flow through cTMS |
| **Two developers transported the same package to QA simultaneously — the second import silently overwrote the first** | cTMS does not enforce a transport lock; concurrent imports of the same package result in last-write-wins behaviour | Establish a team communication protocol (Teams channel announcement) before transporting; consider using CI/CD pipeline integration to serialise transports |
| **cTMS queue has accumulated 30+ unreleased transport requests over a sprint — impossible to trace what changed** | Developers create transport requests during development but don't always release and forward them; there is no expiry or cleanup mechanism | Establish a sprint-end cleanup ritual: all requests must be forwarded or deleted before sprint close; mandate one transport request per story/defect for traceability |
| **After a rollback in PROD, the previous version cannot be re-transported because it was overwritten in DEV** | cTMS retains transport request history, but if the DEV artifact was overwritten and a new transport was created, the old version is no longer easily accessible | Use Git (FlashPipe backup) as the canonical version history source; roll back in PROD by exporting the previous version from Git and importing manually if cTMS history doesn't cover it |

### Externalized Parameters

| Issue | Root Cause | Resolution |
|---|---|---|
| **After transport to PROD, an iFlow connects to the QA SuccessFactors tenant instead of PROD** | The externalized parameter value for the SF destination was not updated in the PROD Integration Suite Configuration → `iflow-external-config` after the transport | After every transport to a new environment, verify all externalized parameters are correctly set; maintain an Environment Parameters Catalogue document per iFlow |
| **Externalized parameters are correctly set but the iFlow runtime still uses the old value** | The iFlow was not redeployed after the parameter values were updated; parameter changes require a redeploy to take effect | Always redeploy the iFlow after any externalized parameter change; set up a post-transport checklist that includes a redeploy confirmation step |

---

## 13. Reference Links

| Topic | Link |
|---|---|
| SAP Cloud Transport Management (Official Docs) | [help.sap.com — cTMS PDF Guide](https://help.sap.com/doc/64d7df2dac3d41ab83a257e9336d15af/Cloud/en-US/Transport_Management_en.pdf) |
| Create Transport Nodes | [help.sap.com — Create Transport Nodes](https://help.sap.com/docs/cloud-transport-management/sap-cloud-transport-management/create-transport-nodes) |
| cTMS FAQ (Community) | [SAP Community — cTMS FAQ](https://pages.community.sap.com/topics/devops/cloud-transport-management-faq) |
| Step-by-Step cTMS Setup | [SAP Community — CTMS Setup Steps](https://community.sap.com/t5/technology-blog-posts-by-members/step-by-step-configuring-ctms-setup-in-btp/ba-p/14119106) |
| cTMS Setup for CPI/Integration Suite | [SAP Community — cTMS for CPI](https://community.sap.com/t5/technology-blog-posts-by-members/ctms-setup-for-cpi/ba-p/14136125) |
| cTMS + Build Work Zone Transport | [SAP Community — cTMS + BWZ](https://community.sap.com/t5/technology-blog-posts-by-members/sap-cloud-transport-management-system-setup-for-transporting-sap-build-work/ba-p/14049409) |
| Cloud ALM Automated Setup | [help.sap.com — Cloud ALM Standard Setup](https://help.sap.com/docs/cloud-alm/setup-administration/standard-setup-in-sap-cloud-alm) |

---

*Document Version: 1.0 | Scope: Hire-to-Retire | Last Updated: June 2026*
