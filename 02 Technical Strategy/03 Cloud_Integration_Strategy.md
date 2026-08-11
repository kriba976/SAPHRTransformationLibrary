# Cloud Integration – Strategy
## SAP Integration Suite | iFlow Landscape Framework

---

## Overview

This document covers the Cloud Integration Strategy and Governance framework for SAP Integration Suite deployments in a global SuccessFactors implementation context. It addresses integration principles and methodology, licensing editions, instance architecture decisions, security (authentication, networking, data protection), naming conventions, access controls, release management, and operational monitoring.

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

## 1. Integration Methodology

### 1.1 SAP Integration Solution Advisory Methodology (ISA-M)

The **SAP Integration Solution Advisory Methodology (ISA-M)** is the recommended SAP framework for defining an enterprise integration strategy. It provides a structured, technology-agnostic approach to analyse integration requirements, select appropriate patterns, and map them to the right integration technology.

ISA-M is the backbone of integration decision-making and is available as a capability within **SAP Integration Suite → Integration Assessment**.

#### ISA-M Structure — Three Layers

```
Layer 1: Integration Domains
  (Where integration is needed — business context)
        │
        ▼
Layer 2: Integration Styles & Use Case Patterns
  (How integration is realised — architecture patterns)
        │
        ▼
Layer 3: Technology Mapping
  (Which SAP tool addresses the pattern)
```

#### Integration Domains

Integration domains describe the business areas where integration is required, independent of technology:

| Domain | Description | H2R Example |
|---|---|---|
| **Cloud-to-Cloud** | Integration between two cloud applications | SuccessFactors EC → SAP S/4HANA Cloud |
| **Cloud-to-On-Premise** | Integration between a cloud app and an on-premise system | SuccessFactors EC → SAP ERP HCM (via Cloud Connector) |
| **On-Premise-to-On-Premise** | Integration between two on-premise systems | SAP HR → SAP Payroll (internal RFC) |
| **B2B / EDI** | Integration with external business partners | Payroll outsourcing file exchange |
| **IoT / Thing** | Integration of physical devices | Time-tracking hardware → SF Time |

#### Integration Styles & Use Case Patterns

Integration styles classify *how* integration is realised. The five core styles in ISA-M are:

| Style | Description | Typical Pattern |
|---|---|---|
| **Process Integration** | Connects business processes across applications | Hire event in SF EC triggers creation of ERP HR record |
| **Data Integration** | Synchronises or accesses data across applications | Full replication of Cost Centres from ERP to SF EC |
| **User Integration** | Integrates user-facing interfaces | HR self-service portal (Work Zone) aggregating SF and ERP data |
| **Analytics Integration** | Connects analytical tools with operational data | Workforce analytics pulling from SF EC and S/4HANA |
| **Thing Integration** | Connects physical devices with business processes | Biometric time terminal → SF Time & Attendance |

Cross-cutting **use case patterns** complement these styles:

- **API-Managed Integration** — all integration goes through an API layer for governance and security
- **Event-Based Integration** — triggers and subscriptions using Intelligent Services Center (ISC) in SuccessFactors

#### Technology Mapping — ISA-M to SAP Integration Suite

| ISA-M Pattern | Recommended SAP Tool |
|---|---|
| Process Integration (A2A) | SAP Cloud Integration (iFlow) |
| Data Replication | SAP Cloud Integration (Standard Packages) |
| API-Managed Integration | SAP API Management (Integration Suite) |
| EDI/B2B Exchange | SAP Integration Suite EDI capability |
| Event-Based Integration | SuccessFactors Intelligent Services Center + Cloud Integration |

> **Reference:** [SAP ISA-M on SAP Help Portal](https://help.sap.com/docs/integration-suite/sap-integration-suite/sap-integration-solution-advisory-methodology) | [ISA-M User Guide (PDF)](https://help.sap.com/doc/ac5a3b73452548cb887f3963877eb9ab/Cloud/en-US/sap.integration.solutions.advisory.pdf) | [SAP Community — ISA-M Define Integration Guidelines](https://community.sap.com/t5/technology-blog-posts-by-sap/integration-solution-advisory-methodology-isa-m-define-integration/ba-p/13397214)

---

### 1.2 Architecture & Design Standards

All integration solutions must be documented with **standardised solution diagrams** before development begins. This enforces consistency across global teams and makes architecture reviews possible.

#### Tooling Standards

| Tool                              | Usage                                                             | Notes                                                                                                      |
| --------------------------------- | ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| **Draw.io (diagrams.net)**        | Primary diagramming tool                                          | SAP-specific shape libraries available; vendor-neutral format; integrates with Confluence, SharePoint, Git |
| ~~**Lucidchart with SAP Extension**~~ | ~~Alternative for organisations with Lucidchart enterprise licences~~ | ~~SAP Lucidchart Extension provides pre-built SAP BTP, IS, and SF component shapes~~                           |
| **SAP LeanIX**                    | Enterprise Architecture Management (where licensed)               | Connects solution diagrams to the enterprise architecture repository                                       |

#### Draw.io Standards for SAP BTP Diagrams

- Use the [SAP BTP Solution Diagrams & Icons](https://github.com/SAP/btp-solution-diagrams) shape library (open-source, maintained by SAP)
- Colour-coding:
  - **Blue border** — SAP BTP managed services
  - **Green border** — SAP on-premise systems
  - **Grey/White** — third-party or non-SAP systems
- Always indicate **protocol** (HTTPS, RFC, SFTP, OData, SOAP) on connector arrows

---

## 2. Strategy — Procurement & Instance Strategy

### 2.1 Procurement Strategy — Cloud Integration Editions

Three editions are available for Cloud Integration, each suited to different implementation scales:

#### 🟢 Starter Edition
- Capabilities for A2A / B2G / Data Replication and other integration styles
- Access to a growing library of 3,200+ prebuilt SAP integration packs
- **50k free messages per month** (each message is a block of 250 KB)
- Cap of **10 custom integrations** that can be built per Global account
- Meant for small SuccessFactors implementations with minimal custom integrations

#### 🔵 Standard Edition
- Capabilities for A2A / B2G / Data Replication and other integration styles
- Access to a growing library of 3,200+ prebuilt SAP integration packs
- **Unlimited connections**, custom integrations and access to all prebuilt adaptors
- **10k free messages per month** — SAP-to-SAP standard messages are not charged; additional messages can be subscribed separately
- API Management capability is included
- Best suited for **mid to large** SuccessFactors implementations

#### 🟣 Premium Edition
- Capabilities for A2A / B2G / Data Replication and other integration styles
- Access to a growing library of 3,200+ prebuilt SAP integration packs
- **10 million messages/month** — SAP-to-SAP standard messages are not charged; additional messages can be subscribed separately
- API Management capability is included
- SAP Alert Notification and SAP Cloud Transport Management services are included with the subscription
- Best suited for organisations with **very high volume** of data traffic

---

### 2.2 Instance Strategy — Cloud Integration Instances

**Guiding principle:** It is recommended to maintain a **one-to-one mapping** between SuccessFactors Employee Central (EC) tenants and Cloud Integration (CI) instances, for each instance that will have integrations to other systems.

- CI instances are **not needed** for EC tenants that do not have integrations (e.g., training or change management tenants)

**Example — 4 EC instances mapped to 3 CI instances:**

| EC Instance | CI Instance |
|---|---|
| DEV | DEV |
| QA | DEV |
| TEST | QA |
| PROD | PROD |

- In this example, the Dev EC instance and QA EC instance both connect to the **Dev CI instance** — 2 copies of the same integration exist, each connecting to a separate EC instance
- In case of defects, the fix must be performed in **both copies** of the integration running in the same CI instance

---

### 2.3 Instance Strategy — Instance Sharing

- Based on the number of SuccessFactors instances used for integrations, a set of Cloud Integration instances is procured for the **entire organisation**
- All regional entities (Geos) within the organisation will **share the same Cloud Integration tenant**

**Example:** A region with 9 country entities, some with multiple Geos — one set of Cloud Integration tenants is procured and shared across the entire region.

> ⚠️ If Geos within an organisation are set up with **multiple SuccessFactors instances**, additional considerations must be taken into account when deciding the Integration Infrastructure Strategy.

---

## 3. Security

### 3.1 Authentication

**Principle:** All communication between SAP Integration Suite and SuccessFactors must use **modern, token-based authentication**. Basic Authentication (username/password) is prohibited in production landscapes.

#### Supported OAuth 2.0 Authentication Types for SuccessFactors

| Auth Type | Grant Flow | When to Use |
|---|---|---|
| **OAuth 2.0 SAML Bearer Assertion** | SAML Assertion → OAuth token | Recommended for user-context API calls where the calling identity (employee/admin) must be asserted; most common for SF OData API |
| **OAuth 2.0 Client Credentials** | Client ID + Secret → OAuth token | For system-to-system (machine-to-machine) calls with no user context; typically used for standard integration flows (e.g., EC replication) |
| **OIDC (OpenID Connect)** | Via IAS as central IdP | Used when IAS is integrated with SuccessFactors and the consumer authenticates against the IAS OIDC endpoint; API Management handles token exchange transparently |

#### OAuth 2.0 SAML Bearer Assertion Flow (Recommended for SF OData)

```
Integration Suite iFlow
        │
        ▼
1. Generate SAML Assertion
   (Signed with the X.509 certificate registered in SF as a trusted OAuth provider)
        │
        ▼
2. POST SAML Assertion to SF Token Endpoint:
   POST /oauth/token
   grant_type=urn:ietf:params:oauth:grant-type:saml2-bearer
   client_id=<API_KEY>
   assertion=<base64_SAML_assertion>
        │
        ▼
3. SF Returns Access Token
   { "access_token": "...", "expires_in": 86400 }
        │
        ▼
4. Call SF OData API with Bearer Token
   Authorization: Bearer <access_token>
```

#### Configuration in SAP Integration Suite

- **Security Material type:** `OAuth2 SAML Bearer Assertion` — stored as a BTP credential in the Integration Suite keystore
- **Certificate:** The X.509 certificate (used to sign the SAML assertion) must be registered in SuccessFactors under **Admin Center → OAuth 2.0 Client Applications**
- **Token caching:** Integration Suite caches the access token for the duration of `expires_in`; subsequent calls within the same iFlow instance reuse the cached token

> **Reference:** [Authentication between SAP Cloud Integration and SAP SuccessFactors using OAuth](https://community.sap.com/t5/human-capital-management-blog-posts-by-sap/authentication-between-sap-cloud-integration-and-sap-successfactors-using/ba-p/13544226) | [OAuth 2.0 with SAML Flow — SuccessFactors Help](https://help.sap.com/docs/successfactors-platform/using-security-center/oauth-2-0-with-saml-flow)

---

### 3.2 Networking — Mandatory API Management Layer

**Principle:** All inbound traffic to SAP Integration Suite from external systems (including SuccessFactors Intelligent Services, third-party HR vendors, and on-premise ERP callbacks) must route through **SAP API Management**. Direct access to iFlow runtime endpoints from outside the BTP network is not permitted in production.

#### Why API Management is Mandatory for Inbound Traffic

| Risk Without API Management | How API Management Addresses It |
|---|---|
| No central authentication enforcement | Authentication Policy enforces OAuth 2.0 / OIDC at the gateway level before the request reaches the iFlow |
| No rate limiting — single caller can flood the integration | Quota and Rate Limit policies throttle calls per application or subscription |
| No audit trail for API consumption | API Management provides a consumption log and analytics dashboard |
| iFlow endpoint URLs are implementation details | API proxies provide stable, versioned URIs decoupled from iFlow deployment details |
| No DDoS or spike protection | Spike Arrest policy limits burst traffic to protect the Integration Suite runtime |

#### API Management Architecture for Inbound Traffic

```
External Caller (SF Intelligent Services / Third Party)
        │
        │  HTTPS (API Proxy URL)
        ▼
SAP API Management (Integration Suite)
  ├── Authentication Policy (OAuth 2.0 / API Key)
  ├── Rate Limit / Quota Policy
  ├── Spike Arrest Policy
  ├── Input Validation / Mediation
  └── Route to iFlow (internal endpoint)
        │
        ▼
Cloud Integration (iFlow)
  (Runtime endpoint not directly exposed externally)
        │
        ▼
Target System (SuccessFactors / ERP / SFTP)
```

> **Reference:** [SAP API Management Capability](https://help.sap.com/docs/integration-suite/sap-integration-suite/api-management-capability) | [Secure SAP SuccessFactors APIs with SAP API Management](https://community.sap.com/t5/technology-blog-posts-by-sap/secure-sap-successfactors-apis-with-sap-api-management-and-microsoft-entra/ba-p/14373038)

---

### 3.3 Data Security — Message-Level Encryption & PII Masking

#### Message-Level Encryption (PGP)

For integrations exchanging sensitive HR payloads — particularly file-based payroll, benefits, or third-party HR vendor exchanges — **OpenPGP encryption** is applied at the iFlow level using SAP Cloud Integration's built-in PGP steps.

| Scenario | PGP Usage |
|---|---|
| Outbound file to payroll vendor | iFlow encrypts the file payload with the vendor's **public PGP key** before placing it on SFTP |
| Inbound file from payroll vendor | iFlow decrypts the received file using the **organisation's private PGP key** stored in the PGP Keyring |
| Signed payloads | iFlow signs outbound files with the organisation's **private key** for non-repudiation |

**PGP Key Management in Integration Suite:**
- PGP keys are stored in the **PGP Keyring** (separate from the Certificate Keystore)
- Public keys of trading partners are imported under **Monitor → Manage Security → PGP Keys**
- The organisation's own private key is imported as a **Secret Key** — access restricted to `PI_Administrator` only
- Key naming convention: `<<Geo>>_<<Partner>>_<<KeyType>>_pgp` (e.g., `NL_ADP_Public_pgp`)

#### PII Data Masking

Sensitive personal data (national ID numbers, bank account numbers, salary data) must not appear in clear text in integration monitoring logs.

**Implementation guidelines:**

| Control | How to Implement |
|---|---|
| Set MPL log level to `Info` or `Error` (not `Trace`) in production | Trace log captures full message payloads and must never be active in production without explicit incident approval |
| Use Groovy script to mask PII fields before logging | Replace sensitive field values with `****` in any custom logging step before writing to the message log |
| Never write sensitive fields to Data Store or Variables without encryption | If temporary storage is required, encrypt the value using `CryptoUtil` in Groovy before storing |
| Limit payload logging in the MPL | Use the `Skip` setting for message log steps to suppress payload capture for sensitive iFlows in production |

> **Reference:** [Security Aspects of Data — Cloud Integration](https://help.sap.com/docs/integration-suite/sap-integration-suite/cloud-integration-security-aspects-of-data-data-flow) | [PGP Encryption and Decryption — SAP Community](https://community.sap.com/t5/technology-blog-posts-by-members/pgp-encryption-and-decryption-in-sap-cloud-integration-a-step-by-step-guide/ba-p/14370923) | [Data Protection and Privacy — Cloud Integration](https://help.sap.com/docs/cloud-integration/sap-cloud-integration/data-protection-and-privacy)

---

## 4. Quick Reference — Naming Conventions

| Artefact | Convention | Example |
|---|---|---|
| Package Name | `<<Org ID>> <<Geo Name>>` | `ORG Netherlands` |
| Package ID | `<<Org ID>>_<<Geo Name>>.INTGR.PACK` | `ORG_Netherlands.INTGR.PACK` |
| Global Integration | `G<<seq>>_<<Geo>>_<<Src>>_to_<<Dest>>_<<Process>>` | `G010_NL_EC_to_S4_Payroll` |
| Local Integration | `L<<seq>>_<<Geo>>_<<Src>>_to_<<Dest>>_<<Process>>` | `L020_NL_EC_to_ADP_Payroll` |
| Credential | `<<Geo>>_<<System>>_<<Instance>>_<<Details>>` | `NL_SFEC_T1_OAuth` |
| Certificate | `<<Geo>>_<<System>>_<<Instance>>_<<Details>>_certificate` | `KR_SFEC_T1_SAMLBearer_certificate` |
| PGP Key | `<<Geo>>_<<Partner>>_<<KeyType>>_pgp` | `NL_ADP_Public_pgp` |
| Custom Role | `<<Org>>_<<Geo>>_CI_Role` | `ORG_KR_CI_Role` |
| Role Collection | `<<Org>>_<<Geo>>_CI_Role_Collection` | `ORG_KR_CI_Role_Collection` |
| Access Policy | `<<Org>>_<<Geo>>_CI_Access_policy` | `ORG_KR_CI_Access_Policy` |

---

*Document Version: 2.0 | Scope: Cloud Integration Strategy & Governance | Last Updated: June 2026*
