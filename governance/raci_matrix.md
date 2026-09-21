# Governance & Operating Model RACI Matrix
## Customer vs. SAP ECS vs. System Integrator (SI)

This document establishes the operational boundaries, roles, and responsibilities for managing and maintaining the SAP landscape before, during, and after migration to **RISE with SAP S/4HANA Cloud Private Edition on Microsoft Azure**.

---

## 1. The Tripartite Operating Model

Migrating from customer-managed Azure IaaS to RISE with SAP transitions operations from a single-tier infrastructure team to a structured tripartite model:

```
                      ┌────────────────────────────────────────┐
                      │              THE CUSTOMER              │
                      │ • Business Process Ownership           │
                      │ • Solution Architecture & Approvals    │
                      │ • Functional Master Data & Testing     │
                      └───────────────────┬────────────────────┘
                                          │
                  ┌───────────────────────┴───────────────────────┐
                  ▼                                               ▼
┌───────────────────────────────────┐           ┌───────────────────────────────────┐
│     SAP ENTERPRISE CLOUD (ECS)    │           │     SYSTEM INTEGRATOR (SI)        │
│ • Azure Hyperscaler Subscription  │           │ • Technical Migration (SUM DMO)   │
│ • OS Patching & Hardening         │           │ • Custom Code & CVI Remediation   │
│ • HANA DB Administration          │           │ • S/4HANA Functional Configuration│
│ • Core SAP Basis Operations       │           │ • Testing & Cutover Coordination  │
│ • System SLA & 24/7 Availability  │           │ • Hypercare Support               │
└───────────────────────────────────┘           └───────────────────────────────────┘
```

### RACI Definitions
* **R - Responsible:** The party that conducts the actual technical or business task.
* **A - Accountable:** The party with ultimate ownership, sign-off authority, and veto power.
* **C - Consulted:** The party providing subject-matter expertise, inputs, or prerequisites.
* **I - Informed:** The party updated on progress, milestones, or service disruptions.

---

## 2. Comprehensive Operational RACI Matrix

### 2.1 Hyperscaler Infrastructure & Cloud Foundation (Microsoft Azure)

| Operational Activity | Customer | SAP ECS | System Integrator | Notes |
| :--- | :---: | :---: | :---: | :--- |
| Azure Subscription provisioning (RISE) | I | **A / R** | - | Resides in SAP-owned Azure tenant. |
| Customer Hub VNet, ExpressRoute, & Firewall | **A / R** | C | C | Customer-owned subscription. |
| Cross-Tenant VNet Peering establishment | **A** | **R** | C | Joint configuration across both tenants. |
| IP Range / CIDR block reservation | **A** | R | C | Customer provides non-overlapping CIDR. |
| Azure VM sizing & resource allocation | C | **A / R** | C | Based on Quick Sizer / Readiness Check. |
| Hyperscaler hardware maintenance & Azure SLA | I | **A / R** | - | SAP delivers up to 99.9% availability SLA. |

### 2.2 Operating System (Linux SLES / RHEL) & File Systems

| Operational Activity | Customer | SAP ECS | System Integrator | Notes |
| :--- | :---: | :---: | :---: | :--- |
| OS installation, license, & hardening | I | **A / R** | - | Customers do NOT receive root access. |
| OS security patching & kernel updates | I | **A / R** | I | Scheduled during agreed maintenance windows. |
| File system provisioning (`/sapmnt`, `/usr/sap`) | - | **A / R** | C | Sized according to SAP standards. |
| Shared file mounts for interfaces (SMB/NFS) | C | **A / R** | C | Requires SAP ECS Service Request. |

### 2.3 Database Administration (SAP HANA)

| Operational Activity | Customer | SAP ECS | System Integrator | Notes |
| :--- | :---: | :---: | :---: | :--- |
| HANA DB installation & initial configuration | I | **A / R** | - | Deployed via SAP automation tools. |
| HANA DB parameter tuning & memory optimization | I | **A / R** | C | SAP ECS applies standard SAP best practices. |
| HANA revision upgrades & Support Packages | I | **A / R** | C | Executed by SAP ECS upon customer approval. |
| Target HANA preparation for SUM DMO | I | **A / R** | **R** | SI triggers import; ECS ensures DB readiness. |
| DB health monitoring & expensive statement trace | I | **A / R** | C | Monitored 24/7 by SAP Enterprise Cloud team. |

### 2.4 Core SAP Basis & Technical Operations

| Operational Activity | Customer | SAP ECS | System Integrator | Notes |
| :--- | :---: | :---: | :---: | :--- |
| SAP Kernel updates | I | **A / R** | I | Routine maintenance by SAP ECS. |
| SAP Support Package Stacks (SPS) execution | **A** | **R** | C | Customer requests via SAP for Me ticket. |
| Client copies (Local / Remote) | **A** | **R** | C | Standard service request within contract scope. |
| Background job monitoring (technical jobs) | I | **A / R** | - | E.g., `SAP_COLLECTOR_FOR_PERFMONITOR`. |
| Background job monitoring (business jobs) | **A / R** | - | C | Customer IT monitors business jobs (e.g., MRP). |
| Spool & printer definitions (`SPAD`) | **A** | - | **R** | Application-level Basis handled by Customer/SI. |
| Transport Management System (`STMS`) setup | I | **A / R** | C | Domain Controller configured by SAP ECS. |
| Daily transport imports (`STMS`) | **A** | - | **R** | Managed by Customer/SI unless CAS contracted. |

### 2.5 Security, Identity & User Management

| Operational Activity | Customer | SAP ECS | System Integrator | Notes |
| :--- | :---: | :---: | :---: | :--- |
| Microsoft Entra ID (Azure AD) SSO configuration | **A / R** | C | C | Configured on Customer Azure tenant. |
| SAP Cloud Identity Services (IAS/IPS) setup | **A** | C | **R** | Federates S/4HANA with Entra ID. |
| ABAP user master maintenance (`SU01`, `PFCG`) | **A** | - | **R** | Application security owned by Customer/SI. |
| S/4HANA Fiori Catalog & Business Role design | **A** | - | **R** | New Fiori authorizations model. |
| SAP standard user hardening (`DDIC`, `SAP*`) | C | **A / R** | - | Managed by SAP ECS security baselines. |

### 2.6 Application Configuration & Custom Code (Clean Core)

| Operational Activity | Customer | SAP ECS | System Integrator | Notes |
| :--- | :---: | :---: | :---: | :--- |
| Customer-Vendor Integration (CVI) execution | **A** | - | **R** | Mandatory ECC pre-conversion work. |
| S/4HANA functional delta configuration | **A** | - | **R** | New Asset Accounting, Material Ledger, etc. |
| ABAP custom code remediation (ATC checks) | **A** | - | **R** | Fixing syntax errors & simplification items. |
| Core Data Services (CDS) & RAP implementation | **A** | - | **R** | Modernizing reports and interfaces. |
| Clean Core compliance monitoring | **A** | I | **R** | Minimizing core modifications. |

### 2.7 Interfaces, Satellite Systems & Network Printing

| Operational Activity | Customer | SAP ECS | System Integrator | Notes |
| :--- | :---: | :---: | :---: | :--- |
| Interface inventory & re-pointing | **A** | I | **R** | RFC, Web Services, IDoc endpoints. |
| Network printer DNS & line-of-sight setup | **A / R** | C | C | Routing between RISE VNet and corporate LAN. |
| Third-party add-on compatibility check | **A** | C | **R** | Vertex, OpenText, CyberSource, etc. |
| SAP Cloud Connector setup & maintenance | **A** | C | **R** | Connects on-prem/RISE to SAP BTP. |

### 2.8 Backup, High Availability & Disaster Recovery (HA/DR)

| Operational Activity | Customer | SAP ECS | System Integrator | Notes |
| :--- | :---: | :---: | :---: | :--- |
| HANA DB backup scheduling (Full, Diff, Log) | I | **A / R** | - | Automated via Azure storage / Backint by ECS. |
| Backup retention policy execution | **A** | **R** | - | Governed by RISE Service Description. |
| Pacemaker cluster setup & maintenance (HA) | I | **A / R** | - | Local zone automated failover. |
| Disaster Recovery (DR) drill execution | **A** | **R** | C | Annual DR failover test managed by SAP ECS. |
| Ad-hoc DB backup request before critical project changes | **A** | **R** | C | Requested via ticket before cutover events. |

### 2.9 Cutover & Migration Project Execution

| Operational Activity | Customer | SAP ECS | System Integrator | Notes |
| :--- | :---: | :---: | :---: | :--- |
| Provisioning of target Sandbox/DEV/QAS/PRD | I | **A / R** | C | SAP ECS builds systems per project schedule. |
| Execution of SUM DMO System Move | C | C | **A / R** | SI Basis team conducts conversion runs. |
| Post-migration financial reconciliation (`FINS_MIG`) | **A** | - | **R** | Financial validation and balance sign-off. |
| User Acceptance Testing (UAT) sign-off | **A / R** | - | C | Business process owners validate system. |
| Go / No-Go final decision | **A / R** | I | C | Customer steering committee executive gate. |

---

## 3. Communication & Service Request Protocol with SAP ECS

All requests to SAP Enterprise Cloud Services must follow standard RISE operational channels:

1. **Service Channel:** **SAP for Me** portal (`https://me.sap.com`).
2. **Service Request Types:**
   * **Incident (Defect/Outage):** High/Very High priority for production stoppage or performance degradation.
   * **Service Request (Standard Changes):** Parameter changes, client copies, certificate updates, firewall port openings. Minimum lead time: **5 business days**.
   * **Project Support:** For Cutover Weekend, a **Hypercare / Critical Event Ticket** must be registered with SAP ECS at least **3 weeks prior** to ensure dedicated 24/7 on-call Basis & DB engineers.
3. **Change Freeze Periods:**
   * Align SAP ECS scheduled maintenance windows (typically weekends) to avoid conflicts with project cutover dress rehearsals.
