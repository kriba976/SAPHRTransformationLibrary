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
## 1. Roles and Responsibilities

### 1.1 Non Client/COE
- Advisory capacity for Procurement 
- Integration Strategy Development (What and Why)
	- Integration Principles & Methodology - with focus on reusability and API first approach, Integration Solution methodology - SAP ISA-M, Integration tooling
	- Architecture & Design standards - Enforcing Solution diagram standards like Draw.io templates or Lucid chat SAP extensions
	- Security
		- Permissions - RBAC for Integration suite, 
		- Authentication - Modern and secure authentication types like OAuth 2.0 for all SuccessFactors communication,  
		- Networking - Mandatory usage of API management layer for incoming traffic management, 
		- Data Security - Usage of message level encryption and data masking for PII data
	- Governance
		- Naming conventions for iflows, artifacts, security credentials, 
		- Rate Limiting - Implementing Rate limiting for incoming connections, 
		- Identity & Access Management
 - Infrastructure Management
	 - BTP Structuring - Subaccount topology, entitlements & quotas management and use of IaaS tools like Terraform
	 - DevOps and Release Management - enforcing the use of tools like cTMS for release / transport management, GitHub for Backup and version management, etc
	 - Operations - Real time monitoring using Cloud ALM for integration errors and notifications
	 - <font color="#ff0000">Audit Logging - Routing all audit logs from SAP BTP Audit logging service to the client's security information and event management systems like Splunk or Sentinel</font>
	 - BTP team management - use ticketing tools like ServiceNow to manage role based access and permissions to BTP entitlements 
	 - Security and Access Control - IAS to be used as the primary and centralized proxy identity provider 
### 1.2 Client 
- Building the Integration Inventory document
- Designing the Integration Architecture blueprint 
- Conducting Integration requirement gathering workshops 
- Technical project management
- Providing regular workstream updates to the project stakeholders 
- Overseeing the Integration workstream 
- Code reviews
- Supervising the cutover activities
- Managing the go-live

## 2. Aspirations
To become a HR Transformation leader with deep subject matter expertise. To further this aspiration of mine, I have completed certifications in all the 3 domains of a HR Transformation project

- **Cloud**: SAP SuccessFactors Employee Central Core – Certified Application Associate (2023)
- **Middleware:** SAP Certified Integration Associate – SAP SuccessFactors Full Cloud/Core Hybrid (2020)
- **On-Prem**: SAP Certified Development Associate – ABAP with SAP NetWeaver 7.0

### 3. Impacts
➤ Co-created a Teams channel containing SF Replication Technical Workbook, Requirements gathering Workshop templates, SAP Help Documents, CPI Training videos, Groovy Scripts snippets – A one-stop-shop for SF Integration architects!  
➤ Fashioned various ready-to-use CPI templates used as a kick-starter to reduce development time by 20%.  
➤ Tailored a Skill Set Assessment worksheet which is used to assess Technical Strengths and Weaknesses of potential candidates and compared to current projects and other projects in the pipeline.  
➤ Created a resource allocation worksheet used by the PMO office to forecast allocations at the level of - Project Phase/Integration/Resource/Day

## 4. Skill Set
### Integration Suite skills  
- SF Integration: Master Data Replication and Org. Assignment, Org. Data replication, CC replication  
- Integration Suite: Full-lifecycle development of Standard Pre-Packaged and Custom Integrations  
- Technical Tooling: Advanced data transformation using Groovy Scripting and XSL Transformation (XSLT)  
   
### SuccessFactors skills  
<font color="#ff0000">- <font color="#ff0000">SF Employee Central (Business): Foundation Objects, Employee Data, Event Reasons, etc  </font></font>
- SuccessFactors (Technical): Employee Central Odata, SFAPI, Intelligent Services, Integration Center  
   
### SAP On-Premise Technical skills  
- BADI, Enhancements, Enterprise Web Services, Consumer & Server Proxy config, SOAMANAGER   
   
### SAP On-Premise Business skills  
<font color="#ff0000">- Expertise with HCM Personnel Administration, Organizational Management  </font>
<font color="#ff0000">- Basic knowledge on HR Time & Attendance and Payroll</font>



**Focus** 

| Topic                | Priority | Next Steps                                                                                                                      |
| -------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Migration            | 1        | 1. Try running the migration program with existing employees<br>2. Continue with the Transformation work and then run migration |
| Replication          | 2        |                                                                                                                                 |
| ALM                  | 3        |                                                                                                                                 |
| Transport Management | 4        | Set up Transport Management in Jamie's BTP system and run a few transport                                                       |
| Audit Management     | 5        | Build a CAP program to view the logs and monitor for any security changes                                                       |






