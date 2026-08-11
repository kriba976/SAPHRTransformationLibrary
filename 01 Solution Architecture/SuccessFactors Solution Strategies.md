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
# 1. Deployment approach 

**Deployment Approach** - all the processes involved in getting new software or hardware up and running properly in its environment. It includes installation, configuration, testing and making changes to optimize the performance of the software

Talent hybrid, EC Read only, EC Core hybrid, **<font color="#ff0000">Full Cloud</font>**

- Start from the Customer’s strategy: How does the Roll-Out / deployment supports the business strategy? - Do you have a executive mandate to fully replace your existing legacy on-prem solutions, or do you have a lot of pain points and want to move to a more robust and integration talent solution, etc
- What are some of the pain points and what should be solved first?

![[Pasted image 20260712114935.png|697]]

---
# 2. **Roll out strategy**

**Roll out Strategy** - Big Bang or <font color="#ff0000">Phased roll out</font>
It is recommended for Employee Central included implementations to start with defining a Global Template first. This includes the minimum set of data to serve all global HR processes and reporting

![[Pasted image 20260712112558.png|697]]

Phased Roll-Out - by Module
	![[Pasted image 20260712112712.png|661]]



---
# 3. SF Instance Strategy 

## 3.1 Terms
Environment - Preview or productive 
Instance - Dev, Test, Prod Suite of SF modules like - EC + Comp + Onb + Rec 
Tenant - individual database schema (SF modules) with unique tenant ID

For example, a Prod Instance may be comprised of HCM Core, LMS, and Recruiting Marketing tenants.

## 3.2 Default SF Instance (Tenants)

**Employee Central** is provisioned with three (3) instances:
– Non-Production instance in Productive Environment recommended using for QA/UAT
– Non-Production instance in Preview Environment recommended using for Development
– Production instance in Productive Environment

**ECP** comes with 2 non-production instances, and allows multiple clients to be created:
– Development Instance can have 2 clients
– Test instance can have 3 clients
– Production Instance

## 3.3 Instance strategy

- **Dev instance preview stack:** 
	- New enhancements should always be tested in an instance which is not linked to integrations or which does not contain real payroll data. 
	
	- Instance on preview stack should not contain production data

- **QA Instance Production stack:**  
	- Most of the Integration testing and Payroll testing happens in the QA. This will be a production stack
	- Live Payroll data can’t be stored in Preview environment as per the DPP protocols
	- Should be able to reproduce production issues on an instance on production stack closest to actual config
	- Customers with QA instance on preview could face twice a year a blackout period

### 3.3.1 Employee Central Instance strategy
<font color="#ff0000">(possibly a sandbox instance)</font>
Dev instance
Test instance
Payroll Parallel instance
Training Instance
Production Instance


### 3.3.2 EC Payroll Instance strategy

1 Dev tenant
	1 Client for Dev
1 Test Tenant 
	1 client for SIT/UAT
	1 client for Payroll Paralell
1 Production tenant 
	1 client for Prod


![[Pasted image 20260713125747.png]]


### 3.3.3 Data Scrambling 
**TBD**




---
# 4. Identity and Authentication Strategy

**TBD**





---
# 5. Refresh Strategy
- Refresh the instances before the preview and production software releases
- Determine of an offcycle refresh is needed
- Once a refresh is don't to Dev and Test - data anonymizing is needed



---
# 6. Testing Strategy

**TBD**



---
# 7. Payroll Process Strategy

**TBD**



---
# 8. Data Migration strategy**TBD**