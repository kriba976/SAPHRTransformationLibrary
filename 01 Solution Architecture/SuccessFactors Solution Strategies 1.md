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

- Instance Management 

-  Determine the most suitable Roll-Out Scenario based upon complexity of the current landscape, size of organization, acceptable risk, and change management needs.

- Determine if a Big Bang approach is technically the preferred approach or identify challenges in going live with all countries and employees at once.

- Determine if a staggered approach is easier to manage and provides a quicker realization on investments being made.

- Determine the impact of the deployment Option (i.e. Phased Roll-Out by Region with EC and legacy On-Premise will result in a EC Read-Only and Phased Roll-Out by module may result in Talent Hybrid or CoreHybrid deployment option).

- Design a HR Global Template - minimum set of data to serv all the Global HR processess and reporting 

---

# 1. SAP Activate Methodology

Prepare 
Explore
Realize
Deploy


---
# 2. Deployment approach 

**Deployment Approach** - all the processes involved in getting new software or hardware up and running properly in its environment. It includes installation, configuration, testing and making changes to optimize the performance of the software

Talent hybrid, EC Read only, EC Core hybrid, **<font color="#ff0000">Full Cloud</font>**

- Start from the Customer’s strategy: How does the Roll-Out / deployment supports the business strategy? - Do you have a executive mandate to fully replace your existing legacy on-prem solutions, or do you have a lot of pain points and want to move to a more robust and integration talent solution, etc
- What are some of the pain points and what should be solved first?

![697](../../01%20SAP%20HR%20Transformation%20Library/01%20Solution%20Architecture/screenshots/20260712114935.png)

---
# 3. **Roll out strategy**

**Employee Central: Implementation Considerations for Phased Rollout**
https://d.dam.sap.com/a/WSqDZAu/IDP%20Employee%20Central%20Implementation%20Considerations%20for%20a%20Phased%20Rollout%20V1.2.pdf?rc=67&inline=true


**Roll out Strategy** - Big Bang or <font color="#ff0000">Phased roll out</font>
It is recommended for Employee Central included implementations to start with defining a Global Template first. This includes the minimum set of data to serve all global HR processes and reporting

![697](20260712112558.png)

Phased Roll-Out - by Module
	![661](20260712112712.png)


---

# 4. SF Instance Strategy 

**SAP SuccessFactors Suite: Instance Management To Support Project Implementation Lifecycle**
https://d.dam.sap.com/a/bj2Gm4w/IDP-%20SAP%20SuccessFactors%20Suite%20Instance%20Management%20To%20Support%20Project%20Implementation%20Life%20Cycle%20V1.31.pdf?rc=67&inline=true


## 4.1 Terms
Environment - Preview or productive 
Instance - Dev, Test, Prod Suite of SF modules like - EC + Comp + Onb + Rec 
Tenant - individual database schema (SF modules) with unique tenant ID

For example, a Prod Instance may be comprised of HCM Core, LMS, and Recruiting Marketing tenants.

![](20260714110422.png)



## 4.2 Default SF Instance (Tenants)

**Employee Central** is provisioned with three (3) instances:
– Non-Production instance in Productive Environment recommended using for QA/UAT
– Non-Production instance in Preview Environment recommended using for Development
– Production instance in Productive Environment

**ECP** comes with 2 non-production instances, and allows multiple clients to be created:
– Development Instance can have 2 clients
– Test instance can have 3 clients
– Production Instance

## 4.3 Instance strategy

- **Dev instance preview stack:** 
	- New enhancements should always be tested in an instance which is not linked to integrations or which does not contain real payroll data. 
	
	- Instance on preview stack should not contain production data

- **QA Instance Production stack:**  
	- Most of the Integration testing and Payroll testing happens in the QA. This will be a production stack
	- Live Payroll data can’t be stored in Preview environment as per the DPP protocols. 
		- Preview environments are designed exclusively for testing and validation, not for safeguarding sensitive corporate and personal records.
		- Preview environments run on experimental, pre-release code that is inherently less stable and more prone to bugs, increasing the risk of data corruption. 
	- Should be able to reproduce production issues on an instance on production stack closest to actual config
	- Customers with QA instance on preview could face twice a year a blackout period

**Example** 
![697](20260714111338.png)

### 4.4.1 Employee Central Instance strategy
-> possibly a GOLDEN CLIENT instance
1 Dev instance - Preview Environment
1 Test instance - SIT & UAT - Production environment
1 Payroll Parallel instance - Payroll Parallel - Production environment
-> <font color="#ff0000">1 Training Instance - Preview environment</font>
1 Production Instance - Production environment


### 4.4.2 EC Payroll Instance strategy

1 Dev tenant
	1 Client for Dev
1 Test Tenant 
	1 client for SIT/UAT
	1 client for Payroll Parallel
1 Production tenant 
	1 client for Prod


![](20260713125747.png)

## 4.5 Data Scrambling 
- Consideration for setting up production like permissions within the test environment
- To consider scrambling of employee data in all non-productive instances


## 4.6 Refresh Strategy
- Refresh the instances before the preview and production software releases
	- Review and communicate the benefits and impact of changes
	- Regression test major feature changes
	- Identify periods when Preview and Production Data Centers are on different releases.
- Determine of an offcycle refresh is needed
- Once a refresh is don't to Dev and Test - data anonymizing is needed



---

# 5. Identity and Authentication Strategy






---

# 6. Testing Strategy

Unit Testing
Requirements validation testing - after every Iteration cycle
Solution Testing (Iterative end-to-end testing) - after the third iteration
	Expand the testing group to additional participants
Permission testing
Payroll Parallel Testing
SIT
UAT



---

# 7. Data Migration strategy

**SAP SuccessFactors Employee Central: Employee Data Migration Strategy and Considerations**
https://d.dam.sap.com/a/ec3czwe/IDP_-_Employee_Central_Data_Migration_Strategy-_V5.3.pdf?rc=67&inline=true

**Employee Central Data Migration: Cutover Optimization Strategy Using Infoporter**
https://d.dam.sap.com/a/hufD6Jb/IDP%20-%20Employee%20Central%20Data%20Migration%20Cutover%20Optimization%20Stratergy%20Using%20Infoporter%20V1.8.pdf?rc=67&inline=true

**Employee Central Core Hybrid: Data and Process Distribution Strategy**
https://d.dam.sap.com/a/rLVc7sG/Employee%20Central%20Core%20Hybrid%20-%20Data%20and%20Process%20Distribution%20Strategy.pdf?rc=67&inline=true

https://d.dam.sap.com/a/f7gNkXE/IDP_EC_COREHYBRID_process_functional.pdf?rc=67&inline=true

**SAP Activate methodology**
- Prepare - Team readiness, roles and resp, migration architecture, system readiness
- Explore - Data migration planning, data mapping, analyze data, clean data, POC
- Realize - extract data, translation and transformation rules., mock run, loading and testing in IT1,2,3, support
- Deploy - Free legacy data, run prod, reconciliation and validation, KT, support

Based on scope of data migration, mock data conversions should be planned as part of overall project timeline

 Objective of mock conversion includes:
– Validate the data extraction, transformation and data import process and data accuracy
– Confirm the final cutover plan in terms of data migration process steps and timings
– Determine if any data cleansing is needed in legacy systems

First mock conversion to be performed around iteration 3 by when the data model is expected to be finalized.

Data quality metrics should be defined to measure completeness and accuracy of data migration.

![](20260714101622.png)
**SuccessFactors Recruitment** 
- RCM - Recruitment Management RCM serves as the core Applicant Tracking System (ATS). It manages the internal hiring process, data tracking, and compliance
	- Requisition Mangement
	- Candidate Tracking
	- Resume Parsing
	- Interview Scheduling
	- Offer Management

- RMK - Recruiting Marketing RMK is the candidate-facing attraction engine. It focuses on sourcing, candidate experience, and inbound marketing
	- Career Site Builder
	- Multi Job Posting
	- Talent COmmunity
	- EMail Campaigns
	- SEO Optimizations



---

# 8. Payroll Process Strategy

**SAP SuccessFactors Employee Central Payroll: Migrating from SAP ERP HCM Payroll to SAP SuccessFactors Employee Central Payroll**
https://d.dam.sap.com/a/MxUrzbM/IDP%20-%20Employee%20Central%20Payroll%20Migrating%20from%20ERP%20HCM%20Payroll%20to%20Employee%20Central%20Payroll%20V1.7.1.pdf?rc=67&inline=true


