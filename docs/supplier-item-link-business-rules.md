# Supplier Item Link — Business Rules

## Introduction

The FW (supplier item / price import) module manages the import, maintenance, and reporting of supplier item catalogues and pricing data. It provides a framework for reading supplier price files in up to 29 different proprietary formats from disk or IFS, populating the FLVAPF (supplier item register) work file, and from there creating or updating items in the main item register. A two-step parameter flow controls the process: FW100R selects the supplier format, FW101R selects the specific supplier and file location, and FW102R orchestrates the file copy and format-specific parsing. FW120R/FW121R handle new-item creation from the imported data. FW600R/FW601R produce printed or on-screen lists of the imported items. FW710R is a representative format-specific parser. All interactive blocking conditions arise from parameter-validation screens before any file-copy or database operation begins.

---

## Prerequisites / Master Data Requirements

- RLEVPF (supplier register) must contain a record for the selected supplier number. FW101R (*in33=ON) and FW120R (*in32=ON) and FW600R (*in33=ON) all block submission if the supplier is not found.
- FLVAPF (supplier item / price work register) must be pre-cleared for the selected supplier by FW102R before new items are written. FW102R performs a keyed delete-all on FLVAPF by firm + supplier before calling the format parser.
- FENKPF (unit conversion register for supplier units) must contain conversion entries for the source units in the supplier file. FW710R looks up FENKPF (fenkl1_key) to translate supplier units to system units; if not found, the original supplier unit is used without conversion.
- VVARPF (item register) is read by FW121R (new-item creation) to check whether the supplier's item number already maps to an existing system item via VVARPF.VVLVAR (supplier item number cross-reference).
- For FW121R — new-item transfer (F20): VVARPF product group fields (w_ogrp, w_hgrp) and item type (w_vtyp) must be pre-set via FW123R (standard values) before the transfer can proceed.

---

## Validation Rules

### FW100R — Supplier Format Selection

- b1valg must be one of the supported format codes (1, 4–16, 20–25, 27–29). If b1valg is any other value: indicator *in31 = ON, b_feil = *on (blocking — invalid format selection).

Supported format codes:
| Code | Supplier |
|---|---|
| 1 | ASP old standard format |
| 4 | Jernia |
| 5 | Luna |
| 6 | Arvid Nilsson |
| 7 | ESAB |
| 8 | Farveringen |
| 9 | Farvemiljø Nord |
| 10 | Christ Engebretsen |
| 11 | ELFO standard format |
| 12 | Ole Moe Engros |
| 13 | Coward |
| 14 | Sandvik |
| 15 | Felleskjøpet Maskin |
| 16 | Smith |
| 20 | Hov+Dokka |
| 21 | Andvord |
| 22 | Sørbø |
| 23 | Norske Røtter |
| 24 | Emco |
| 25 | General SDV format |
| 27 | General Integ.dat (IFS) |
| 28 | General Integ.dat (QDLS) |
| 29 | Control list Integ.dat (IFS) |

### FW101R — Supplier and File Location Parameters

| Condition | Indicator | Blocking Condition |
|---|---|---|
| b1file blank (file name not entered) | *in31 = ON | File name is required |
| b1ldor = 0 (supplier number not entered) | *in32 = ON | Supplier number is required |
| b1ldor not found in RLEVPF (*in91=ON) | *in33 = ON | Supplier does not exist in supplier register |

After FW101R calls FW102R and FW102R returns p_feil = *on (file-copy error): indicator *in35 = ON is set on the FW101R screen (informational — file not found or copy failed; user must correct the file path/name).

### FW102R — File Copy and Format Dispatch (Controller)

- Calls FW103C (IFS copy) if p_valg = 27 or 29, else calls FW102C (QDLS copy). If the called CL returns p_feil = *on: FW102R sets p_feil = *on and `goto avslutt` (blocking — file cannot be copied; no processing occurs).
- After successful file copy: deletes all existing FLVAPF records for the selected supplier (full pre-clear by firm + supplier). This is irreversible; old imported data is always discarded before the new import.
- For format 25 (FW725R) or 27 (FW727R): dispatched specially without firm/supplier parameters passed to the format parser.
- For format 29 (FW729R): dispatched as a control-list run via FW729C.
- All other formats: calls the format-specific program (FW701R–FW720R etc.) with firm and supplier as parameters.

### FW120R — New Item Creation Parameter Screen

| Condition | Indicator | Blocking Condition |
|---|---|---|
| b1ldor = 0 (supplier number not entered) | *in31 = ON | Supplier number is required |
| b1ldor not found in RLEVPF (*in32=ON from CHAIN) | *in32 = ON | Supplier does not exist |
| b1lvaf > b1lvat (from-number after to-number) | *in33 = ON | From supplier item number must be ≤ to supplier item number |
| Date entered AND fails DMY validation (*in34=ON) | *in34 = ON | Date entered is invalid |

### FW121R — New Item Display and Transfer Subfile

- F20 (start transfer to item register): if w_ogrp = 0 OR w_hgrp = 0 OR w_vtyp = *blank: indicator *in31 = ON (blocking — main product group, sub-group, and item type must all be set before transfer can proceed).
- These three values are set via FW123R (standard values screen, F15). The transfer button (F20) is intentionally blocked until the user has defined these mandatory classification values.

### FW600R — Item List Print Parameter Screen

| Condition | Indicator | Blocking Condition |
|---|---|---|
| b1rtyp not in (1, 2, 3) | *in31 = ON | Invalid report type |
| b1ldor = 0 | *in32 = ON | Supplier number is required |
| b1ldor not found in RLEVPF (*in91=ON) | *in33 = ON | Supplier does not exist |
| b1lvaf > b1lvat (from after to) | *in34 = ON | From-number must be ≤ to-number |
| Date entered AND fails DMY validation | *in35 = ON | Invalid date |
| Date entered AND b1oper blank → defaults to `'EQ'` | — | Non-blocking; default applied |
| b1oper not in blank/`'EQ'`/`'GT'`/`'LT'` | *in36 = ON | Invalid price comparison operator |
| b1endr ≠ 0 AND b1rtyp ≠ 1 (change% only valid for report type 1) | *in37 = ON | Change percentage only for report type 1 |
| b1ekod not in blank/`'X'`/`'N'`/`'E'`/`'S'`/`'P'` | *in38 = ON | Invalid change code |

---

## Configuration and Authorization Rules

- **Format code** (b1valg in FW100R): each code maps to a specific format parser program (FW701R, FW704R, ... FW729C) and a default file name and folder path stored in the compile-time array a_ldor. Adding a new supplier format requires both a new array entry and a new parser program.
- **Supplier number** (b1ldor): must exist in RLEVPF for the active company. The supplier record provides the name displayed on confirmation screens and printed reports.
- **FLVAPF pre-clear**: every import run deletes all existing FLVAPF records for the selected supplier before writing new ones. There is no incremental update; the import is always a full replace. This means concurrent imports for the same supplier will corrupt each other's data.
- **Product group / item type for new-item creation** (FW121R): w_ogrp (main group), w_hgrp (sub-group), and w_vtyp (item type) are job-level variables. They are set via FW123R and persist for the duration of the FW121R session. If the user exits FW121R and re-enters, these values are reset to zero/blank, requiring re-entry before F20 (transfer) can succeed.
- **Change code** (b1ekod in FW600R): valid values are blank (all), `'X'` (deleted), `'N'` (new), `'E'` (replaced/changed), `'S'` (discontinued), `'P'` (price changed). Any other value is rejected with *in38=ON.
- **IFS path**: for formats 27 and 29, the file is read from IFS via FW103C. The folder path (b1fldr) and file name (b1file) must accurately describe an existing IFS object accessible to the IBM i job.

---

## Financial / Transactional Rules

- FW121R new-item transfer (F20/FW124R): creates new records in VVARPF (item register) with base prices (VVKPRI), item type, and product group classification. These are real item master records used in order processing and pricing. Incorrect classification values (wrong product group or type) produce incorrectly classified items that may affect pricing, discount calculation, and statistical reporting.
- FW710R (and all other format parsers) writes raw supplier data to FLVAPF including: supplier item number (FWLVAR), EAN number (FWEANN), NOBB number (FWNOBB), unit (FWENHE), purchase price (FWINPR), sales price (FWSAPR), and a change code (FWEKOD). The change code `'S'` (discontinued/expired) is automatically set when the supplier's expiry date (d_udat) is in the past.
- FW601R (report): reads FLVAPF and cross-references with VVARPF to identify which imported supplier items already have matching system items, and which are new. This is an informational comparison; it does not modify any records.

---

## Status and Lifecycle Rules

| Status / Condition | File | Field | Value | Meaning | Effect |
|---|---|---|---|---|---|
| File copy failed | (FW102C/103C return) | p_feil | *on | Source file not found or unreadable | FW102R aborts; no FLVAPF records written |
| Supplier not found | RLEVPF | — | *in91=ON | Supplier number invalid | *in31–*in33=ON on screens; blocked |
| Item expired | FLVAPF | FWEKOD | `'S'` | Expiry date in past | Written as discontinued; transfer to item register may be skipped |
| Product group not set | Session var | w_ogrp / w_hgrp / w_vtyp | 0 / blank | Classification not defined | FW121R F20 blocked (*in31=ON) |
| Invalid format | Screen | b1valg | Out of range | Format code not recognized | *in31=ON (FW100R); blocked |
| FLVAPF empty | FLVAPF | — | No records | No import data | FW121R subfile is empty; FW601R produces no output |

---

## Special Conditions (Program-Specific)

### FW102R — FLVAPF Pre-Clear Is Irreversible

The delete-all of FLVAPF records for the selected supplier occurs before the new import is read. If FW102R's file copy succeeds but the format parser subsequently fails or produces no records, the FLVAPF for that supplier will be empty. There is no transaction-based rollback. The user must re-run the full import to restore data.

### FW121R — Three-Key Browsing Modes

FW121R supports three browsing modes for the subfile: by supplier item number (FWLVAR, *in80=ON), by EAN number (FWEANN, *in81=ON), and by NOBB number (FWNOBB, *in82=ON). F23 cycles through these modes. The initial mode is supplier item number. Position-to fields in the subfile control header (b2lvar, b2eann, b2nobb) correspond to the active mode.

### FW710R (and Format Parsers) — Change Code Derivation

Format parsers like FW710R automatically derive the change code FWEKOD = `'S'` when the supplier's expiry date field (d_udat) is non-zero and is earlier than the current date. Items with change code `'S'` are visible in FW121R and FW601R but should not be transferred to the active item register.

### FW190R — Network File Operations

FW190R is a simple pass-through: it presents a screen and immediately calls FW190C (CL program) to manage network file operations (e.g., copying supplier files from a network share). FW190R itself contains no validation logic; all work is in FW190C.

---

## Subprogram Calls Affecting Logic

| Calling Program | Called Program | Purpose | Blocking Effect |
|---|---|---|---|
| FW100R | FW101R | Second parameter screen (format specific) | Called only after format validation |
| FW101R | FW102R | File copy and format dispatch | p_feil=*on → *in35=ON on screen |
| FW101R | RL500R | Supplier lookup (F4) | Non-blocking; populates w_lnav |
| FW102R | FW102C or FW103C | Copy file from QDLS or IFS | p_feil=*on → abort |
| FW102R | FW725R | General SDV format parser | Special dispatch; no firm/supplier params |
| FW102R | FW727C | General Integ.dat parser | Special dispatch |
| FW102R | FW729C | Control list parser | Special dispatch |
| FW102R | FW701R–FW720R | Format-specific parsers | Called with firm + supplier |
| FW120R | FW121R | New item display/transfer subfile | Called after validation |
| FW120R | RL500R | Supplier lookup (F4) | Non-blocking |
| FW121R | FW110R | Status codes reference screen (F13) | Non-blocking |
| FW121R | FW122R | Group processing (F14) | Updates FLVAPF group assignments |
| FW121R | FW123R | Standard values (F15) | Sets w_ogrp, w_hgrp, w_vtyp, w_kpri |
| FW121R | FW124R | Execute new-item transfer (F20) | Blocked if w_ogrp=0 or w_hgrp=0 or w_vtyp blank |
| FW600R | FW600C | Execute item-list print job | Called after validation |
| FW600R | RL500R | Supplier lookup (F4) | Non-blocking |

---

## Reference Tables

| Table (Physical File) | Key Fields | Relevant Fields | Role in Module |
|---|---|---|---|
| FLVAPF (flval2–flval5, flvalu) | Firm, Supplier nr, Line nr | FWLVAR, FWEANN, FWNOBB, FWENHE, FWINPR, FWSAPR, FWEKOD, FWUDAT, FWDATO, FWTEK1 | Supplier item work register; pre-cleared on each import |
| RLEVPF (rlevl1) | Firm, Supplier nr | RLNAVN | Supplier register; required for all supplier validation |
| VVARPF (vvarl5) | Firm, Item nr | VVLVAR (supplier item cross-ref) | Item register; checked for existing matches in FW121R/FW601R |
| FENKPF (fenkl1) | Supplier unit (FEENHF) | System unit (FEENHT) | Unit conversion register; translates supplier units |
| FLVWPF | (sequential) | Raw supplier input records | Temporary work file; written by FW102C/103C, read by parsers |
| JVARPF | (various logical views) | Item master (LVR) | Item register used for cross-reference in FW121R |
