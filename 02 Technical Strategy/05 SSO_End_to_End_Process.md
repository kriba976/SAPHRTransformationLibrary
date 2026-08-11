# SSO End-to-End Process – Hire to Access
## SAP SuccessFactors EC | IAS | Active Directory | Global Assignment & Concurrent Employment

---


```table-of-contents
title: 
style: nestedList
minLevel: 0
maxLevel: 0
includeLinks: true
hideWhenEmpty: false
debugInConsole: false
```

---

## Key Concepts Before You Start

### USER_ACCOUNT – The SSO Anchor

| Concept               | Detail                                                                                                                                                                      |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| What it is            | A database table in SF that stores a **unique username** used for SSO                                                                                                       |
| How SSO uses it       | For Single Sign-On, the **Subject Name Identifier** coming back from AAD (Azure Active Directory) is compared with the `USER_ACCOUNT` username to open the employee profile |
| How many per employee | An employee will have **only one** `USER_ACCOUNT` username, regardless of the number of employments in SF or the number of AD accounts they hold                            |
| Login flexibility     | The employee should be able to log in with **any of their AD accounts / email domains**                                                                                     |

### Identity Count Discrepancy – IAS vs AAD

> ⚠️ There is a known discrepancy between the number of identities in **AAD** and those in **SAP IAS**. SAP has designed its identity replication mechanism to create IAS profiles based on `USER_ACCOUNT` — not on individual employments. So however many employments an employee has, they will always have **one IAS profile**.
>
> Multiple IAS profiles cannot be created for an employee because the IAS profile is based on **primary email**, and every employee has only one primary email in SF.

---

## Phase 1 — Hire in SF Employee Central

**Trigger:** HR creates a new hire record in SuccessFactors Employee Central.

### Data Captured at Hire

| System | Field | Value (Example) | Notes |
|---|---|---|---|
| **EC – Person** | `personIdExternal` | `31000765` | Retained from legacy core HR / PERNR |
| **EC – Person** | First Name | `Lucas` | |
| **EC – Person** | Last Name | `Smith` | |
| **EC – Person** | Primary Email | `lucas@personal.com` | Personal email at hire; will be overwritten with business email later |
| **EC – USER_ACCOUNT** | Username | *(empty)* | Not populated yet — USER_ACCOUNT is empty until HRIS Sync runs |
| **EC – Primary Employment** | `userId` | `31000765` | Equals `personIdExternal` for the first / primary employment |
| **EC – Primary Employment** | `username` | *(empty)* | Not populated yet |
| **EC – Primary Employment** | `givenname` | `31000765` | Set equal to `personIdExternal` |
| **EC – Primary Employment** | Country of Hire | `NL` | |
| **Additional IDs Profile** | All fields | *(empty)* | Azure Object ID, Alias etc. not yet populated |
| **SAP IAS** | All fields | *(empty)* | No IAS record yet |

> 💡 **Key rule:** The `username` of the **first / primary employment** is always equal to the `userId`, which is always equal to `personIdExternal`.

---

## Phase 2 — HRIS Sync

**Trigger:** HRIS Sync job runs (scheduled or on-demand).

### What Happens

- The `username` of the **first employment** is read and assigned as the `USER_ACCOUNT` username
- This is the critical moment when the SSO anchor is established

### State After HRIS Sync

| System | Field | Value | Notes |
|---|---|---|---|
| **EC – USER_ACCOUNT** | Username | `31000765` | ✅ Now populated — username of first employment |
| **EC – Primary Employment** | `userId` | `31000765` | |
| **EC – Primary Employment** | `username` | `31000765` | |
| **SAP IAS** | All fields | *(still empty)* | IAS replication not yet triggered |

> 💡 `USER_ACCOUNT.username` = `userId` of the primary employment = `personIdExternal`

---

## Phase 3 — EC to AD Outbound Integration (Custom iFlow)

**Trigger:** EC outbound integration iFlow sends employee data to the corporate Identity Provider / Active Directory.

### What Is Sent to AD

| EC Field | Value |
|---|---|
| `personIdExternal` | `31000765` |
| First Name | `Lucas` |
| Last Name | `Smith` |
| Primary Email | `lucas@personal.com` (at this stage) |
| `USER_ACCOUNT` username | `31000765` |
| `userId` (Primary Employment) | `31000765` |
| `username` (Primary Employment) | `31000765` |
| Country of Hire | `NL` |

### What AD Creates

| AAD Attribute | Value | Notes |
|---|---|---|
| **UPN** (User Principal Name) | `lucas@org.co.uk` | Email address — what the employee uses to sign in |
| **Primary Key / Profile Identifier** | `31000765NL` | `personIdExternal` + 2-character country code of the employment |
| Azure Object ID | `a215c145-b754-4a58-aef…` | GUID generated by AAD |
| Alias | `lsmith` | |

> 💡 The **UPN remains the email address** — the employee always logs in with their email. The `personIdExternal + country code` (e.g., `31000765NL`) is stored as a separate AAD attribute (e.g., `employeeId`) and is used specifically as the **source for the claims transformation** during SSO. It is NOT the login credential.

---

## Phase 4 — AD Inbound Integration (Write-back to EC)

**Trigger:** AD Inbound iFlow writes the newly created AD account details back into SF EC.

### What Is Written Back to EC

| Target | Field | Value | Notes |
|---|---|---|---|
| **EC – Primary Email** | Primary Email | `lucas@org.co.uk` | ✅ Business email **overwrites** the personal email entered at hire |
| **EC – Additional IDs Profile** | `userId` | `31000765` | |
| **EC – Additional IDs Profile** | Email | `lucas@org.co.uk` | Business email stored here |
| **EC – Additional IDs Profile** | Azure Object ID | `a215c145-b754-4a58-aef…` | GUID assigned by AAD |
| **EC – Additional IDs Profile** | Alias | `lsmith` (example) | |

### State After AD Write-back

| System | Field | Value |
|---|---|---|
| EC – Person | Primary Email | `lucas@org.co.uk` (business email) |
| EC – USER_ACCOUNT | Username | `31000765` |
| EC – Primary Employment | `userId` / `username` | `31000765` |
| EC – Additional IDs Profile | Azure Object ID | `a215c145-b754-4a58-aef…` |
| SAP IAS | All fields | *(still empty)* |

> 💡 The **Azure Object ID** in the Additional IDs Profile will be replicated to **FG** to use as the SSO parameter, and also replicated to **SAP ECC** systems that use it for SSO (e.g., UK entity).

---

## Phase 5 — Replicate to SAP IAS

**Trigger:** IAS outbound replication iFlow runs (event-driven or scheduled).

### What Is Replicated to IAS

| IAS Field | Source in EC | Value |
|---|---|---|
| First Name | EC Person | `Lucas` |
| Last Name | EC Person | `Smith` |
| Email | EC Primary Email | `lucas@org.co.uk` |
| **Login Name** | `USER_ACCOUNT.username` | `31000765` |

> 🔑 **Critical:** The `USER_ACCOUNT` username is set as the **IAS Login Name**. This is the identifier used for SSO.

### State After IAS Replication

| System | Field | Value |
|---|---|---|
| SAP IAS | First Name | `Lucas` |
| SAP IAS | Last Name | `Smith` |
| SAP IAS | Email | `lucas@org.co.uk` |
| SAP IAS | Login Name | `31000765` ← USER_ACCOUNT username |

---

## Phase 6 — Replicate to FG and S/4HANA

**Trigger:** Downstream system replication iFlow runs.

- The **Azure Object ID** from the Additional IDs Profile is sent to **SuccessFactors FG** as the SSO parameter
- The same object is replicated to **SAP ECC / S/4HANA** for systems that rely on it for SSO (e.g., UK legal entity)
- **IAS Login Name** (`31000765`) is used by connected SAP applications via the IAS / IPS federation layer

---

## Phase 7 — Single Sign-On (SSO)

**Trigger:** Employee attempts to log in via AAD (corporate IdP).

### How SSO Works

1. Employee signs in with their **email / UPN** (e.g., `lucas@org.co.uk`)
2. AAD authenticates the user and identifies their AAD profile — the profile holds the attribute `31000765NL` (`personIdExternal` + country code)
3. A **claims transformation rule in AAD** strips the 2-character country code suffix → produces `31000765`
4. AAD sends the **SAML assertion** to IAS with **Subject Name Identifier = `31000765`**
5. IAS matches `31000765` against the **IAS Login Name** (= `USER_ACCOUNT.username` replicated from SF)
6. Match found → IAS forwards the assertion to SF EC
7. SF EC compares Subject Name Identifier with `USER_ACCOUNT.username` → Match → employee profile opened ✅

> 💡 **Why `personIdExternal + country code` as the AAD primary key works for multi-employment:**
> - An employee on global assignment has 2 AAD profiles: `31000765NL` (home) and `31000765BE` (host)
> - The claims transformation strips the country code from **both** profiles, always producing `31000765`
> - SF/IAS sees the same Subject Name Identifier regardless of which AAD account the employee logs in from
> - The UPN/email is just the login credential — SSO resolution is driven by Employee ID, not email

---

## Phase 8 — Global Assignment / Concurrent Employment

> ⚠️ **Key Rules:**
> - When an employee gets a **new employment** (Global Assignment or Concurrent Employment), the `USER_ACCOUNT` username **does NOT change**
> - When an employee gets **rehired** with a new employment, the `USER_ACCOUNT` username **does NOT change**

### Adding the Secondary Employment

| System | Field | Value | Notes |
|---|---|---|---|
| EC – Secondary Employment | `userId` | `31000765-1` | New employment ID (suffix appended) |
| EC – Secondary Employment | `username` | `31000765-1` | |
| EC – Secondary Employment | Country | `IN` (example) | Host country for the global assignment |
| EC – USER_ACCOUNT | Username | `31000765` | ✅ Unchanged |

### HRIS Sync (Additional Employment)

- **No update** to `USER_ACCOUNT` username — HRIS Sync does not change it when an additional employment exists
- Additional employments **do not change** the value of `USER_ACCOUNT` username

### IAS Replication (Additional Employment)

- **No new IAS record created**
- Even though the employee now has 2 employments, only the **User_Account object** is replicated to IAS
- Only **one identity** is maintained for the employee in IAS

---

## Phase 9 — Global Assignment: New AD Account & Email

**Scenario:** Employee on Global Assignment to another country requires a **new AD account** and a **new business email address** for the host country.

### New AD Account Creation (Host Country)

**Trigger:** EC to AD Outbound Integration iFlow detects the secondary employment and creates a new AD account in the host country's directory.

| Action | Detail |
|---|---|
| New AD Account | Created in host country AAD tenant |
| New Business Email | Assigned based on host country domain (e.g., `lucas@org.in`) |
| New Azure Object ID | Generated by host AAD for this account |

### AD Write-back for Secondary Employment

| Target | Field | Value |
|---|---|---|
| EC – Secondary Employment Additional IDs | Email | `lucas@org.in` |
| EC – Secondary Employment Additional IDs | Azure Object ID | `b4115e80-d3f4-… (new GUID)` |

### Primary Email Update

- The **Primary Email in EC** is updated to the **new host-country business email** (`lucas@org.in`)
- This triggers an update to IAS since IAS email = EC Primary Email

### IAS Update After Global Assignment

| IAS Field | Updated Value | Notes |
|---|---|---|
| Email | `lucas@org.in` | Updated to new business email |
| Login Name | `31000765` | ✅ Unchanged — still the USER_ACCOUNT username |

### SSO with Multiple AD Accounts

| Scenario | AAD Profile Key | Claims Transformation | Subject Name ID sent to IAS | Result |
|---|---|---|---|---|
| Logs in with home email (`lucas@org.co.uk`) | `31000765NL` | Strip `NL` → `31000765` | `31000765` | ✅ Matches USER_ACCOUNT |
| Logs in with host email (`lucas@org.be`) | `31000765BE` | Strip `BE` → `31000765` | `31000765` | ✅ Matches USER_ACCOUNT |

> 💡 Both AAD accounts produce the same Subject Name Identifier after claims transformation, so SSO works identically from either account — even though the email addresses and AAD profiles are different.

---

## End-to-End Summary

```
HIRE IN EC
│  personIdExternal = 31000765 | First/Last Name | Primary Email (personal)
│  USER_ACCOUNT.username = EMPTY | userId = 31000765
▼
HRIS SYNC
│  USER_ACCOUNT.username = 31000765 (= username of first employment)
▼
EC → AD OUTBOUND (Custom iFlow)
│  Creates AD user account | Assigns business email | Generates Azure Object ID
▼
AD → EC INBOUND (Write-back)
│  Primary Email updated to business email
│  Additional IDs Profile: Azure Object ID, Alias written back
▼
REPLICATE TO IAS
│  IAS: First Name, Last Name, Email, Login Name = 31000765 (USER_ACCOUNT)
▼
REPLICATE TO FG / S4
│  Azure Object ID → FG (SSO parameter)
│  IAS Login Name → SAP ECC / S/4
▼
SSO
│  Employee logs in with email/UPN (e.g. lucas@org.co.uk)
│  AAD profile key = 31000765NL → claims rule strips country code → 31000765
│  AAD sends Subject Name Identifier = 31000765 to IAS
│  IAS matches Login Name = 31000765 → forwards to SF
│  SF matches USER_ACCOUNT.username = 31000765 → Profile opened ✅
▼
GLOBAL ASSIGNMENT / CONCURRENT EMPLOYMENT
│  New secondary employment added (userId = 31000765-1)
│  USER_ACCOUNT.username stays = 31000765 (UNCHANGED)
│  New AD account + new business email for host country
│  IAS: still one profile, Login Name still = 31000765
│  SSO works with both AD accounts ✅
```

---

## Field Mapping Master Reference

| Field | EC Location | IAS | AAD | Notes |
|---|---|---|---|---|
| `personIdExternal` | Person | — | — | = `userId` of primary employment |
| `USER_ACCOUNT.username` | USER_ACCOUNT table | Login Name | Subject Name Identifier (after claims transformation) | The SSO anchor — never changes |
| AAD Profile Primary Key | — | — | `personIdExternal` + 2-char country code (e.g. `31000765NL`) | Claims rule strips country code before sending to IAS |
| First Name | Person | First Name | Given Name | |
| Last Name | Person | Last Name | Surname | |
| Primary Email | Person | Email | Mail | Updated to business email after AD write-back |
| Azure Object ID | Additional IDs Profile | — | Object ID | Used for FG / ECC SSO |
| `userId` (primary) | Primary Employment | — | — | = `personIdExternal` |
| `userId` (secondary) | Secondary Employment | — | — | `personIdExternal` + suffix |
