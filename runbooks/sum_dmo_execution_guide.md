# SUM DMO with System Move - Technical Execution Guide
## Hands-on Basis Procedure for Azure IaaS to RISE S/4HANA Migration

This guide provides the technical, screen-by-screen, and command-line execution procedure for running **SAP Software Update Manager (SUM) with Database Migration Option (DMO) - System Move** from an SAP ECC 6.0 instance on Azure IaaS to an SAP S/4HANA Cloud Private Edition instance in RISE with SAP.

---

## 1. Prerequisites & Media Staging

### 1.1 Maintenance Planner & Stack XML
1. Log in to **SAP Maintenance Planner** (`https://apps.support.sap.com/sap/support/mp`).
2. Select your source ECC 6.0 system.
3. Choose **Plan an S/4HANA Conversion**.
4. Select the target release (e.g., *SAP S/4HANA 2023 FPS02*).
5. Verify add-on compatibility and select required business functions.
6. Push target software packages to the SAP Download Basket.
7. Download the generated **Stack Configuration XML** file (`MP_Stack_XXXXXXXX_XXXX_.xml`) and copy it to the source host:
   ```bash
   /sapmnt/trans/EPS/in/MP_Stack_XXXXXXXX_XXXX_.xml
   ```

### 1.2 Download & Extract SUM on Source ECC Host
1. Download the latest **Software Update Manager 2.0 (SUM)** SP from the SAP Support Portal.
2. Log in as `root` on the source Azure ECC application server.
3. Create the SUM directory and extract the archive:
   ```bash
   mkdir -p /usr/sap/<SID>/SUM
   cd /usr/sap/<SID>/SUM
   SAPCAR -xvf /sapmnt/trans/EPS/in/SUM20SPXX_X-XXXXXXXX.SAR
   chown -R <sid>adm:sapsys /usr/sap/<SID>/SUM
   ```

### 1.3 Target RISE HANA Preparation
Coordinate with SAP ECS (via Service Request) to confirm:
* Target S/4HANA Private Edition HANA database is running.
* Schema `SAP<SID>` and database users (`SYSTEM`, `DDIC`, `<sid>adm`) are provisioned.
* Cross-tenant Azure VNet Peering is active and port `4239` (SUM System Move) or `3<Instance#>15` (HANA SQL) is reachable.
* Test network reachability from source Azure VM to target RISE VM:
   ```bash
   nc -zvw3 10.200.30.15 4239
   nc -zvw3 10.200.30.10 30015
   ```

---

## 2. Source System Table Splitting & Benchmark

To prevent large application tables (e.g., `BKPF`, `BSIS`, `BSEG`, `ACDOCA` predecessors) from choking a single R3load process, configure table splitting during the preparation phase.

1. Generate table split input file `table_split.txt`:
   ```text
   # Table%SplitCount
   BKPF%8
   BSIS%8
   BSAS%8
   BSAD%8
   BSID%8
   COEP%12
   MSEG%8
   VBRP%6
   VBRK%4
   ```
2. Feed the split file to R3ldctl / R3ta during the SUM configuration phase, or place it in:
   ```bash
   /usr/sap/<SID>/SUM/abap/bin/table_split.txt
   ```

---

## 3. Launching and Configuring SUM DMO

### 3.1 Start SAPup Process
As `root` on the source host:
```bash
cd /usr/sap/<SID>/SUM/abap
./SAPup startconf
```
Access the SUM Web GUI from your browser:
```text
https://<Source_ECC_Hostname_or_IP>:1129/lmsl/sumabap/<SID>/doc/sluigui
```
Log in using `<sid>adm` credentials.

### 3.2 Screen-by-Screen Configuration (Uptime Phase)

| SUM Step / Screen | Required Input / Action | Technical Notes |
| :--- | :--- | :--- |
| **Select Target** | Select **Database Migration Option (DMO)** with **System Move**. | Enables cross-host / cross-network migration. |
| **Stack XML** | Browse to `/sapmnt/trans/EPS/in/MP_Stack_XXXXXXXX_XXXX_.xml`. | SUM validates software components against the stack. |
| **Passwords** | Enter master passwords for `<sid>adm`, `DDIC`, and database administrator. | Stored securely in SUM secure storage. |
| **System Move Mode** | Choose **Memory Pipe Mode (Socket Transfer)** or **Staged File Mode (NFS/ANF Mount)**. | **Recommended:** Socket transfer over Azure VNet Peering for maximum speed. |
| **Target Database Details** | Hostname: `10.200.30.10`<br/>DB Port: `30015` (or `3<inst>15`)<br/>HANA Schema: `SAPHANADB`<br/>User: `SYSTEM` | Connects directly to the RISE HANA DB over VNet Peering. |
| **Process Parameters** | • Uptime R3load Processes: **8 to 16**<br/>• Downtime R3load Processes: **32 to 64**<br/>• Target DB parallel load: **Max available CPU / 2** | Tuned to saturate the Azure VM bandwidth and CPU. |

---

## 4. Execution Phases: Step-by-Step

### 4.1 Phase: Uptime Pre-Processing (`PREP_GENCHECKS` & `SHADOW`)
1. **Consistency Checks:** SUM scans the source dictionary, checks for duplicate indexes, and verifies Unicode compliance.
2. **Shadow Repository Build:** SUM creates shadow tables in the source database and imports target S/4HANA software components without impacting current business users.
3. **Table Comparison:** Validates custom dictionary structures against S/4HANA simplification data.
4. **Data Export Preparation:** SUM initializes export jobs and calculates data block segments.

### 4.2 Phase: Transition to Downtime
1. Follow the [technical_cutover_runbook.md](file:///Users/priyankapalo/Downloads/sap_migration/runbooks/technical_cutover_runbook.md):
   * Run `BTCTRNS1` (suspend batch).
   * Lock users via `SU10`.
   * Clear queues (`SMQ1`, `SMQ2`, `SM58`).
   * Run baseline financial reconciliation (`RFBILA00`).
2. In SUM Web GUI, confirm the prompt to enter **DOWNTIME**:
   ```
   Phase: MOD_SELROAD/RUN_FDIC_DOWNTIME -> Proceed to Downtime
   ```

### 4.3 Phase: Downtime Data Migration (`EU_CLONE_MIG_DT_RUN`)
1. **Delta Export:** SUM extracts delta data written since uptime shadow generation.
2. **Streaming Transfer:** Data streams across the Azure VNet Peering connection directly into target R3load processes.
3. **Target HANA Import:** Parallel `R3load` threads bulk-insert rows into SAP HANA columnar tables.
4. **Monitoring Throughput:**
   * View live progress in the SUM GUI under **DMO Process Monitor**.
   * On Linux CLI, inspect active pipes:
     ```bash
     tail -f /usr/sap/<SID>/SUM/abap/log/MIGRATE_UT_RUN.LOG
     tail -f /usr/sap/<SID>/SUM/abap/log/MIGRATE_DT_RUN.LOG
     ```
   * On Azure portal: Monitor `Network Out` on source VM and `Network In` on target RISE VM.

### 4.4 Phase: S/4HANA Structure Conversion & `XPRAS`
1. Once data is imported, SUM executes phase `EU_IMPORT`:
   * Creates secondary indexes on SAP HANA.
   * Transforms legacy financial tables (`BSIS`, `BSAS`, `BSEG`, `COEP`) into the Universal Journal foundation view (`ACDOCA`).
2. **Execution of XPRA programs (`RUN_XPRAS`):**
   * Automatically executes S/4HANA application conversion routines across SD, MM, and FI.
3. **Post-Downtime Cleanup (`PUDEL`):**
   * Deletes temporary shadow tables and temporary R3load dump files.
4. SUM notifies: **"Update completed successfully"**.

---

## 5. Post-Conversion Functional & Technical Configuration

Immediately after SUM completes:

### 5.1 Financial Data Migration (`FINS_MIG_STATUS`)
1. Log in to the new S/4HANA Private Edition system in RISE.
2. Execute transaction `FINS_MIG_STATUS`:
   ```text
   Step 1: Check Pre-Requisites
   Step 2: Material Ledger (ML) Migration (CKML_MIG)
   Step 3: Universal Journal Data Migration (FINS_MIG_UJ)
   Step 4: Asset Accounting Migration (FAA_MIG)
   Step 5: Reconcile General Ledger Balances
   ```
3. Verify that all migration steps show **Status Green (Completed)**.

### 5.2 ABAP Mass Compilation (`SGEN`)
To prevent end-user latency on first-time transaction execution:
1. Launch transaction `SGEN`.
2. Select **Generate All Objects of Selected Software Components**.
3. Allocate available parallel background work processes.
4. Monitor until compilation reaches 100%.

### 5.3 Interface & System Re-Pointing
1. **RFC Destinations (`SM59`):** Update target hostnames, IP addresses, and gateway ports to route through Azure VNet Peering.
2. **Web Services (`SOAMANAGER`):** Re-generate endpoints for external consumers.
3. **Spool / Printers (`SPAD`):** Update access methods (Access Method `L` or `U` with target Azure-routed IP addresses).
4. **Fiori Launchpad Activation:**
   * Execute task list `SAP_FIORI_FOUNDATION_S4` via transaction `STC01`.
   * Activate target OData services in `/IWFND/MAINT_SERVICE`.

---

## 6. Troubleshooting Common SUM DMO Errors

| Phase | Common Error | Root Cause | Resolution |
| :--- | :--- | :--- | :--- |
| `PREP_GENCHECKS` | `Open update requests found in VBDT/VBHDR` | Uncommitted updates in source ECC before conversion. | Process or delete orphan update records via transaction `SM13`. |
| `RUN_FDIC_UPTIME` | `Duplicate index error on table XXX` | Custom index on source table conflicts with target S/4HANA standard index. | Drop duplicate `Z*` index in source DB; SUM will re-create target indexes. |
| `EU_CLONE_MIG_DT_RUN` | `R3load: Socket connection broken / timeout` | Azure network security group or idle TCP timeout drop on port 4239. | Increase Azure VNet Peering idle timeout to 30 mins; verify port 4239 in NSG rules. |
| `EU_IMPORT` | `HANA DB out of memory (OOM)` | High number of parallel R3load import jobs exceeding HANA allocation limit. | Temporarily lower `R3load` import processes in SUM; adjust `global_allocation_limit` in HANA Studio. |
| `RUN_XPRAS` | `Program XXX ended with RC 12` | Obsolete custom exit or un-remediated ABAP code called during post-import script. | Review `/usr/sap/<SID>/SUM/abap/log/SAPup.log` and note referenced in the job log; apply SNOTE or skip if permitted by SAP Support. |
