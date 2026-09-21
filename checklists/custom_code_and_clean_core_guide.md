# Custom Code Remediation & Clean Core Guide
## S/4HANA Simplification Database & ABAP Cloud Strategy

Custom code remediation is one of the most critical workstreams when converting SAP ECC 6.0 to **RISE with SAP S/4HANA Cloud Private Edition**. This guide provides the tools, procedures, and rules for analyzing, fixing, and decoupling custom ABAP (`Z*` and `Y*`) code.

---

## 1. Clean Core & Extensibility Paradigm

In RISE with SAP, maintaining a **Clean Core** ensures upgrade readiness, eliminates technical debt, and improves security.

```mermaid
flowchart TD
    subgraph LegacyApproach ["Legacy ECC 6.0 Approach"]
        CoreECC["SAP Core Software"]
        Modifications["User Exits / Core Modifications"]
        DirectSQL["Direct Native SQL / DB Hints"]
        CoreECC --- Modifications
        CoreECC --- DirectSQL
    end

    subgraph ModernCleanCore ["S/4HANA Clean Core Architecture"]
        CoreS4["S/4HANA Cloud Private Edition<br/>(Standard Core)"]
        OnStack["On-Stack Developer Extensibility<br/>(ABAP Cloud / Released APIs)"]
        SideBySide["Side-by-Side Extensibility<br/>(SAP Business Technology Platform - BTP)"]
        CoreS4 <==>|Released Core Data Services (CDS)| OnStack
        CoreS4 <==>|OData / REST APIs / Events| SideBySide
    end
```

### Extensibility Tiers in RISE:
1. **Key-User Extensibility:** Low-code custom fields, custom logic, and UI adaptations built directly in Fiori apps without modifying ABAP code.
2. **On-Stack Developer Extensibility (ABAP Cloud):** Custom ABAP code written using the RESTful Application Programming (RAP) model using only **Released SAP APIs** (`C1` contract).
3. **Side-by-Side Extensibility (SAP BTP):** Decoupled microservices, integrations, and external user portals running on SAP Business Technology Platform.

---

## 2. ABAP Test Cockpit (ATC) Setup & Execution

### 2.1 ATC Tooling Architecture
The **ABAP Test Cockpit (ATC)** checks custom code against the S/4HANA Simplification Database.

* **Recommended Setup:** Run ATC checks via a dedicated Central Check System (e.g., S/4HANA Sandbox or SAP BTP Custom Code Migration app) targeting your ECC 6.0 code base via RFC.

### 2.2 Execution Steps
1. **Import Simplification Database:**
   * Download the latest Simplification Database content file from SAP Support Portal.
   * Upload and activate it in transaction `/SDF/TECHS_CHECK` or `CCMS_S_DATABASE`.
2. **Configure ATC Check Variant:**
   * Use check variant `S4HANA_READINESS_REMOTE` or `S4HANA_READINESS_2023`.
3. **Schedule Remote Inspection:**
   * Launch transaction `ATC` -> **Runs** -> **Schedule**.
   * Select the target RFC destination pointing to ECC PRD/DEV.
   * Target package: `Z*`, `Y*`, and customer-namespace packages.
4. **Export ATC Results:**
   * Export findings to Excel or review directly in **ABAP Development Tools (ADT) in Eclipse**.

---

## 3. Top Custom Code Remediation Categories & Fixes

### 3.1 Removed & Consolidated Data Model Tables

| Obsolete ECC Table | S/4HANA Replacement | Required Remediation Action |
| :--- | :--- | :--- |
| `BSEG`, `BSIS`, `BSAS`, `BSID`, `BSAD` | **`ACDOCA`** (Universal Journal) | Replace direct `SELECT` statements with Core Data Services (CDS) views or S/4HANA compatibility views. |
| `KNC1`, `KNB1` (Customer Balances) | **`ACDOCA`** | Balance tables are now computed on-the-fly; eliminate direct updates to balance totals. |
| `KONV` (Pricing Conditions) | **`PRCD_ELEMENTS`** | Direct access to `KONV` aborts at runtime. Replace with `PRCD_ELEMENTS` or call function `PRICING`. |
| `VBUK`, `VBUP` (Order Status) | **`VBAK`, `VBAP`** | Status fields have been merged directly into header (`VBAK`) and item (`VBAP`) tables. |
| `MKPF`, `MSEG` (Material Documents) | **`MATDOC`** | Read access is supported via compatibility view `NSDM_V_MSEG`; write access must use standard BAPIs (`BAPI_GOODSMVT_CREATE`). |

### 3.2 Database Operations & SAP HANA Optimization
1. **Unsorted `SELECT` Statements:**
   * *Problem:* On AnyDB, queries without `ORDER BY` often returned rows in the primary key sequence. SAP HANA columnar store returns rows in non-deterministic order.
   * *Remediation:* Add explicit `ORDER BY` clauses to all queries where business logic expects sorted results.
2. **Native SQL & Database Hints:**
   * *Problem:* Native SQL (`EXEC SQL ... ENDEXEC`) or DB hints (`%_HINTS ORACLE ...`) will cause short dumps on SAP HANA.
   * *Remediation:* Replace native SQL with Open SQL or CDS views.
3. **Direct Pool & Cluster Table Access:**
   * *Problem:* Table clusters (e.g., `BSEG`, `CDCLS`) are declustered into transparent tables in HANA.
   * *Remediation:* Remove any code relying on cluster-specific access routines.

### 3.3 Field Length Extensions
* **Material Number Extension (MATNR):**
  * Extended from **18 characters to 40 characters**.
  * Verify custom structures, internal tables, and BAPI parameters: replace hardcoded `C(18)` definitions with standard data element `MATNR`.
* **Amount Field Length Extension:**
  * Currencies and amounts now support extended decimals in Universal Journal tables.

---

## 4. Code Remediation Workflow & Workload Estimation

```
                    Total Custom Code Objects Inventory
                                     │
                    ┌────────────────┴────────────────┐
                    ▼                                 ▼
          Used Objects (> 0 executions)       Unused Objects (0 executions)
          (Derived from ST03N / SUSG / CCLM)  (Not executed in past 12-18 months)
                    │                                 │
                    ▼                                 ▼
            Run ATC S/4HANA Checks               Scope for Deletion
                    │                            (Retire before conversion)
        ┌───────────┴───────────┐
        ▼                       ▼
   Syntactic / Hard Errors    Performance / Quality
   (Priority 1 - Mandatory)  (Priority 2 - Post Go-Live)
   • Obsolete tables         • Missing ORDER BY
   • Direct KONV queries     • SELECT * optimization
   • Native SQL hints
```

### Automatic Code Remediation via Quick Fixes:
Using **ABAP Development Tools (ADT) in Eclipse**:
1. Open the **ATC Result Browser**.
2. Select issues categorized under *S/4HANA Readiness*.
3. Right-click and choose **Quick Fix (Ctrl + 1)**:
   * Automatic replacement of `MATNR` character definitions.
   * Automatic translation of `KONV` queries to `PRCD_ELEMENTS`.
   * Automatic insertion of `ORDER BY PRIMARY KEY`.
* **Efficiency:** Automated Quick Fixes resolve **50% to 70%** of all syntactic ATC errors automatically.
