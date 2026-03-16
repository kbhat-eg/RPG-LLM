# Data Fixes / Matching / LVR — Business Rules

## Introduction

The RY (data fixes / LVR matching) module contains a collection of one-time and periodic maintenance programs for the ASVADM subsystem, which manages the Leverandørregisteret (LVR — central supplier item catalogue). These programs operate on the LVR database files (JVARPF, JVDTPF, JVPRPF, JMATPF, JFORPF, JVPKPF, and related tables). Their primary functions are: purging obsolete price and item data for specific hard-coded suppliers, importing MAXBO purchasing data to release item holds and update prices, executing a full LVR upgrade pipeline (RY810R), deleting item-related records for items removed from the LVR, and performing bulk company-number rewrites. Because these programs are designed as one-time data fixes and batch maintenance utilities, they contain hard-coded supplier numbers, firm numbers, and search-text literals rather than configurable parameters. They should only be executed by trained administrators under controlled conditions.

---

## Prerequisites / Master Data Requirements

- The LVR database files (JVARPF, JVDTPF, JVPRPF, JMATPF, JFORPF, JVPKPF, JPFAPF, JWPRPF, JSOKPF, JVSOST) must exist and be accessible before any RY program can run. These files are assumed to reside in the library list.
- For RY810R (LVR upgrade pipeline): the physical files for the existing LVR must first be copied to `JxxxPFX` shadow files, and the new LVR data library must be renamed to `ADBLVR`. The called sub-programs (RY810C, RY812R–RY818R, RY820R, JV719R) require both the old and new data to be simultaneously accessible.
- For RY750R and RY751R (MAXBO integration): the MAXBO input file (JRVAPF) must be pre-loaded with current MAXBO item data before the program is run. These programs perform a sequential read of JRVAPF; an empty file results in no changes.
- JVDTPF (item detail / hold register) must exist for RY800R to write hold flags. If a JVDTPF record does not exist for an item, RY800R creates one (add mode).
- For RY852R (company change): the firm number is hard-coded as `100`. The target firm number must be confirmed as correct before execution.

---

## Validation Rules

### RY130R — Remove Prices for Hard-Coded Supplier -1

- No interactive validation. Hard-coded supplier = `900001`, hard-coded search text = `'FARVEMILJØ NORD AS'`.
- Deletes all JVPRPF records (price register) for supplier 900001 via sequential READE.
- For each deleted price record: chains to JSOKPF (search text register) by item + search text; if found, deletes the JSOKPF record.
- No conditions block execution; the program runs to completion regardless of how many records are found.

### RY131R — Remove Prices for Hard-Coded Supplier -2

- No interactive validation. Hard-coded source supplier = `54901`; hard-coded target supplier = `59097`.
- Reads JVARPF for all items belonging to supplier 54901 (via jvarl8 logical view).
- For each item: chains to JVPRPF for item + supplier 59097. If found: deletes the JVPRPF record.

### RY132R — Remove Locally Registered Prices for Hard-Coded Suppliers

- No interactive validation. Hard-coded supplier list: `201698, 202633, 201889, 50955, 50997, 91515, 201124, 201469, 201940`.
- For each JVARPF item: if JVARPF.JVLDOR matches one of the hard-coded supplier numbers, reads JVPRPF records for that item.
- Deletes JVPRPF records where JVPRPF.JXRSTS = `1` (locally registered status). Records with JXRSTS ≠ 1 (LVR-sourced) are preserved.

### RY750R — Release Item Holds from MAXBO File

- Reads JRVAPF (MAXBO input file) sequentially. For each record: extracts the supplier item number from positions 1–6 of the raw input field (JRVAPF.JRVRAD).
- Chains to JVARPF (jvarl8 logical) by supplier 009999 + supplier item number. If not found (*in91=ON): skips this record.
- If found: chains to JVDTPF by firm + item number. If found (*in92=OFF): sets JVDTPF.JVDHKO = `0` (releases hold) and updates. If JVDTPF not found: no action (hold record does not exist; item was not held).

### RY751R — Update Purchase Price and Create Markup from MAXBO

- For each JRVAPF record: if JVPRPF.d_inpr (purchase price from MAXBO) = 0: `leavesr` — skips the record (zero price is invalid; no price update performed).
- Chains to JVARPF (jvarl8) by supplier 009999 + supplier item number. If not found (*in91=ON): `leavesr` — item not in LVR; skip.
- Updates JVPRPF (price register) for the item: sets JXPRIS = d_inpr, JXGDAT (price valid date), JXRSTS = `1` (locally registered). Creates record if not existing.
- Creates JPFAPF (markup table) entry only if d_saim (sales price incl. VAT from MAXBO) ≠ 0 AND JVDTPF.JVDHKO ≠ `1` (item is not held). Markup factor is calculated as: `jcsafa = (d_saim / 1.25) / d_inpr`.

### RY752R — Convert Item Package Units

- Sequential read of JVPKPF (item packaging register). For each record: maps old unit codes to new unit codes via SELECT/WHEN:

| Old Unit | New Unit |
|---|---|
| `'BKS'` | `'BOX'` |
| `'GL '` | `'GLA'` |
| `'MTR'` | `'M  '` |
| `'LPM'` | `'LM '` |
| `'PKN'` | `'PAK'` |
| `'SK '` | `'SEK'` |
| `'SP '` | `'SPA'` |
| `'SPR'` | `'STK'` |

- All records are updated regardless of whether the unit code matches any mapping (records with unmapped units are also updated — the SELECT has no WHEN that matches them, so they pass through unchanged but are still UPDATED with their current values, which is a no-op).

### RY800R — Set Hold on All Items

- Sequential read of JVARPF. For each item: chains to JVDTPF by firm + item number.
  - If JVDTPF found (*in61=OFF): sets JVDTPF.JVDHKO = `1` (hold flag) and updates.
  - If JVDTPF not found (*in61=ON): creates a new JVDTPF record with JVDFIR = firm, JVDVAR = item number, JVDHKO = `1`.
- No conditions block execution. This program places every item in the LVR on hold, preventing price and markup updates until holds are selectively released (e.g., by RY750R).

### RY810R — LVR Upgrade Pipeline (Controller)

- Calls the following programs in sequence; if any called program fails abnormally (MSGF error), the pipeline aborts at that point:
  1. RY810C — CL: empty physical files and copy from ADBLVR
  2. RY812R — Update supplier register
  3. RY813R — Update producer register
  4. RY815R — Update item register
  5. RY816R — Update item packaging
  6. RY817R — Update item price register
  7. RY818R — Create item details for new items
  8. JV719R — Update product group / module number on conditions linked to items
  9. RY820R — Update match register with new item information

- No interactive validation. This is a batch pipeline called from a CL job entry. The prerequisite that the shadow files (`JxxxPFX`) and the `ADBLVR` library are in place must be verified externally before execution.

### RY820R — Update Match Register

- Reads JMATPF (match register) sequentially for the current firm.
- For each match record: if JMATPF.JOVAR2 (matched item number) is blank: skips record (`goto neste_vare`).
- Chains to JVARPF by JOVAR2. If found AND JMATPF.JOLDO2 (current supplier in match) ≠ JVARPF.JVLDOR (supplier in LVR): updates JMATPF.JOLDO2 to the LVR supplier number. This synchronizes the match register with supplier reassignments in the LVR.

### RY830R — Delete Records for Items Removed from LVR

- Processes seven related tables in sequence, deleting records for items no longer in JVARPF:
  1. JVDTPF (item details)
  2. JPFAPF (markup table) — only where JPFAPF.JCVARE ≠ blank
  3. JFORPF (packaging) — only where JFORPF.JZVARE ≠ blank
  4. JWPRPF (price work register)
  5. JVPKPF (item packaging) — only where JVPKPF.JWRSTS = `1` (locally registered)
  6. JVPRPF (price register) — only where JVPRPF.JXRSTS = `1` (locally registered)
  7. JSOKPF (search text register)
  8. JMATPF (match register)
  9. JVSOST (item search status)

- The condition for deletion is: chain to JVARPF by item number; if NOT FOUND → delete the related record. This is a referential integrity repair: orphaned records in the nine related tables are removed.

### RY850R — Company Number Change for LVR Tables

- Hard-coded target firm number: `w_firm = 100` (set on line 50 of the source).
- Performs a sequential full-table scan and firm-number update on: JBETPF, JRAEPF, JFORPF, JKAHPF, JKAVPF, JMATPF, JPFAPF, JRAPPF, JRVGPF, JRFIPF, JSTSPF, JVDTPF, JWPRPF.
- No blocking conditions. The hard-coded firm number `100` must be manually changed in source before compilation if a different target firm is needed.

---

## Configuration and Authorization Rules

- **Hard-coded supplier numbers**: RY130R (900001), RY131R (54901, 59097), RY132R (nine supplier codes) all operate on fixed supplier numbers compiled into the program. If the LVR supplier number scheme changes, these programs must be recompiled with updated values.
- **Hard-coded firm number**: RY850R uses firm = `100`. All other RY programs derive the firm from LDA (position 944–946), except RY130R and RY131R which do not use the firm key in their primary logic.
- **Hold mechanism** (JVDTPF.JVDHKO): a value of `1` = item on hold (blocked from price/markup updates). A value of `0` = item released. RY800R sets all items to hold; RY750R selectively releases items based on MAXBO purchase data. RY751R respects the hold by checking JVDHKO = `1` before creating markup entries.
- **Locally registered vs. LVR-sourced** (JXRSTS in JVPRPF, JWRSTS in JVPKPF): `1` = locally registered (created by local system); non-1 = sourced from LVR. RY132R, RY830R, and RY751R selectively process or protect records based on this flag.
- **LVR upgrade procedure** (RY810R): requires that all LVR physical files have been copied to `JxxxPFX` shadow files and the new LVR library is at `ADBLVR` before RY810C runs. This is an external operational prerequisite; RY810R itself has no guard against running on an unprepared environment.

---

## Financial / Transactional Rules

- RY751R calculates the markup factor (JPFAPF.JCSAFA) as: `(d_saim / 1.25) / d_inpr`. This applies a 25% VAT reverse (divides by 1.25) to the sales price before computing the margin ratio against the purchase price. This is the standard Norwegian 25% VAT rate hard-coded into the formula.
- RY751R also sets JXPRIS (purchase price) and JXGDAT (price valid date) in JVPRPF. The JXRSTS = `1` flag marks the updated price as locally registered, distinguishing it from LVR-sourced prices.
- Price and markup updates via RY751R apply only to items that are NOT on hold (JVDHKO ≠ 1). Items set to hold by RY800R are protected from price and markup changes.
- RY830R deletes locally registered prices (JXRSTS = 1) and locally registered packaging (JWRSTS = 1) for items no longer in the LVR. LVR-sourced records (JXRSTS ≠ 1) for deleted items are not cleaned up by this program.

---

## Status and Lifecycle Rules

| Status | File | Field | Value | Meaning | Effect |
|---|---|---|---|---|---|
| Item on hold | JVDTPF | JVDHKO | `1` | Item blocked from price/markup update | RY751R skips markup creation |
| Item released | JVDTPF | JVDHKO | `0` | Item eligible for price/markup update | RY751R can create markup |
| Locally registered price | JVPRPF | JXRSTS | `1` | Price entered locally (not from LVR) | Preserved by RY817R; deleted by RY830R if item removed; targeted by RY132R |
| LVR-sourced price | JVPRPF | JXRSTS | ≠ `1` | Price from central LVR | Protected from RY132R deletion; eligible for LVR upgrade |
| Locally registered packaging | JVPKPF | JWRSTS | `1` | Packaging entered locally | Deleted by RY830R if item removed |
| Match register — no LVR item | JMATPF | JOVAR2 | blank | No matching LVR item set | Skipped by RY820R |
| Zero purchase price from MAXBO | JRVAPF | d_inpr | `0` | Invalid price in MAXBO file | RY751R skips this record |

---

## Special Conditions (Program-Specific)

### RY800R + RY750R — Hold/Release Cycle

The canonical workflow for importing MAXBO purchase data is:
1. Run RY800R: sets JVDHKO = `1` on all items (mass hold).
2. Load JRVAPF with current MAXBO data.
3. Run RY750R: sets JVDHKO = `0` for items that appear in the MAXBO file (selective release).
4. Run RY751R: updates purchase prices and creates markup entries, but only for released items.

This ensures that only items MAXBO has confirmed as purchased items receive updated pricing. Items not in MAXBO remain held and are not priced from this run.

### RY810R — No Error Recovery in Pipeline

RY810R calls nine sub-programs sequentially. There is no error checking between calls; if one sub-program fails abnormally (e.g., due to a missing file or authority problem), the pipeline aborts at that point and subsequent programs do not run. This leaves the LVR in a partially upgraded state. Manual intervention is required to complete or roll back the upgrade.

### RY850R — Hard-Coded Firm Number

The target firm number in RY850R is set at line 50: `c eval w_firm = 100`. This value must be changed in source and the program recompiled before running against a different target company. Running with the wrong firm number corrupts all scoped records in the thirteen affected tables.

### RY130R / RY131R / RY132R — One-Time Cleanup Programs

These programs appear to be one-time or periodic cleanup programs written for specific historical data problems (Farvemiljø Nord AS, supplier 59097, and nine other specific suppliers). They contain no interactive parameter entry and no safeguards against repeated execution. Running them multiple times produces the same result (idempotent deletions) but may produce zero changes after the first run.

---

## Subprogram Calls Affecting Logic

| Calling Program | Called Program | Purpose | Blocking Effect |
|---|---|---|---|
| RY810R | RY810C | CL: empty and copy LVR files | Must complete; subsequent programs depend on output |
| RY810R | RY812R | Update supplier register | Sequential in pipeline |
| RY810R | RY813R | Update producer register | Sequential in pipeline |
| RY810R | RY815R | Update item register | Sequential in pipeline |
| RY810R | RY816R | Update item packaging | Sequential in pipeline |
| RY810R | RY817R | Update item price register | Sequential in pipeline |
| RY810R | RY818R | Create item details for new items | Sequential in pipeline |
| RY810R | JV719R | Update product group/module nr | Sequential in pipeline |
| RY810R | RY820R | Sync match register with LVR | Sequential in pipeline |

---

## Reference Tables

| Table (Physical File) | Key Fields | Relevant Fields | Role in Module |
|---|---|---|---|
| JVARPF (jvarl1, jvarl8) | Item nr / Supplier+SupplierItemNr | JVVARE, JVLDOR, JVLVAR | LVR item register; master for all RY programs |
| JVDTPF (jvdtl1, jvdtlu) | Firm, Item nr | JVDHKO (hold flag), JVDFIR | Item detail / hold register; JVDHKO=1 blocks pricing |
| JVPRPF (jvprl1, jvprl2, jvprlur) | Item nr / Supplier+Item nr | JXPRIS, JXGDAT, JXRSTS, JXLDOR | Price register; JXRSTS=1 = locally registered |
| JMATPF (jmatl1, jmatlur) | Firm, Item 1 (JOVAR1) | JOVAR2, JOLDO2 | Match register; RY820R syncs JOLDO2 to LVR |
| JSOKPF (jsokl1, jsokl3, jsoklur) | Item nr, Search text | — | Search text register; cleaned by RY130R/RY830R |
| JPFAPF (jpfal6, jpfalur) | Firm, Groups, Supplier, Item | JCVARE, JCSAFA (markup) | Markup table; created by RY751R |
| JFORPF (jforl5, jforlur) | Firm, Groups, Supplier, Item | JZVARE | Packaging (forpakning); cleaned by RY830R |
| JVPKPF (jvpkl1, jvpklur) | Item nr | JWVARE, JWRSTS, JWENHE | Item packaging; JWRSTS=1 = locally registered |
| JWPRPF (jwprl1, jwprlur) | Item nr | JBVARE | Price work register; cleaned by RY830R |
| JVSOST (jvsoi1, jvsoiur) | Various | — | Item search status; cleaned by RY830R |
| JRVAPF | (sequential) | JRVRAD (raw input), d_lvar, d_inpr, d_saim, d_gdat | MAXBO input file; pre-loaded externally |
| JBETPF / JRAEPF / JKAHPF / JKAVPF / JRAPPF / JRVGPF / JRFIPF / JSTSPF | Firm | Firm number | LVR tables updated by RY850R company change |
