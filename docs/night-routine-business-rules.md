# Business Rules: Night Routine (AN Module)

**System:** ASOFAK / NXCLOUD
**Module Prefix:** AN
**Programs Analyzed:** AN010R, AN020R, AN030R, AN040R, AN050R
**NXKORR Overrides:** None found
**Date:** 2026-03-16

---

## 1. Prerequisites / Master Data Requirements

- Night job parameters are stored in `AVERPF` (night job parameter file). AN010R reads and writes this file. If `AVERPF` does not contain a record for the active firm, the screen displays with default/blank values.
- Night job history is stored in `AHSTPF`. AN050R writes history records to `AHSTPF`. If `AHSTPF` is not accessible, history recording fails silently (no explicit error handling shown for file write failures).
- AN020R (reorganise CL source generator) reads `APGMPF` (program register) to obtain the list of files eligible for `RGZPFM`. If `APGMPF` is empty or inaccessible, no reorganise commands are generated.
- AN040R checks whether `AN044C` (generated CL program) already exists in `QCLLESRC`. It reads the source physical file directly. If `QCLLESRC` is not present in the library list, AN040R will fail with a file-not-found error at runtime.
- AN030R (IPL schedule) requires `AN032C` (CL program) to be compiled and present. If `AN032C` is not found at call time, the IPL schedule update fails.

---

## 2. Validation Rules

### AN010R — Night Job Parameter Entry

**Backup / Savefile Conflicts:**
- If `b1back = 2` (full backup selected) AND (`b1savf = 1 or b1savf = 2`) → `*in31` is set, blocking the save. Full backup and savefile cannot be combined.
- If `b1back <> 1` (standard backup not selected) AND (`b1savf = 1 or b1savf = 2`) → `*in34` is set, blocking the save. Savefile requires standard backup (`b1back = 1`).

**Extra Jobs Consistency:**
- If `b1prc1 = 0` (extra jobs flag OFF) AND any of `c1pro1 <> *blank or c1pro2 <> *blank` (job programs defined) → `*in32` blocks. Cannot define extra jobs when the flag is off.
- If `b1prc1 = 1` (extra jobs flag ON) AND both `c1pro1 = *blank and c1pro2 = *blank` (no job programs defined) → `*in33` blocks. Must define at least one job program when the flag is on.

**Generation Trigger:**
- If `b1genp = 1` (generate flag set), AN010R calls `AN012C` (CL driver) after saving. No validation error is raised by `AN012C` call failure at the RPG level — CL execution errors are not surfaced back to the screen.

**Subsystem Display (version 6.20+):**
- AN010R calls `AA073R` to populate the list of subsystems on the parameter screen. If `AA073R` fails or returns no subsystems, the subsystem field remains blank — not a blocking condition.

### AN020R — RGZPFM Source Generator

- Files with a library name starting with 'Q' or 'Æ' are skipped (system libraries excluded).
- Files with a file name starting with 'Q' or 'Æ' are skipped (system files excluded).
- Files derived from QS36F (System/36 environment) are skipped (`QS36F`-origin check).
- Only files of type PF (physical file) are eligible for `RGZPFM` processing. All other file types are silently skipped.
- The output is generated CL source written to `QCLLESRC`. If the source file member already exists, it is overwritten.

### AN030R — IPL Schedule Parameters

- Time field `b1ttmm` (HH:MM format): no explicit date/time format validation shown in available source. The value is passed directly to `AN032C`.
- Day field `b1udag` (weekday number): passed directly to `AN032C`. No range check visible in RPG source.
- User cancels (F3/F12): program exits without calling `AN032C`. IPL schedule is not updated.

### AN040R — AN044C Existence Check

- Reads `QCLLESRC` to check for member `AN044C`. Returns `p_valg = '1'` if found, blank if not found. This is used by the calling program to decide whether to regenerate or reuse the existing source.

### AN050R — Night Job History Update

- History code 99: writes the content of LDA positions 701–740 as a text entry to `AHSTPF`.
- History codes 1–30: text is looked up from an internal array (30-element text array). The code is used as an array index.
- History code > 30 and not 99: uses array entry 1 (generic "undefined code" text). This is not a blocking error — the history record is written with the fallback text.
- `AN050C` (CL delay program) is called first. If `AN050C` fails, history recording does not proceed.

---

## 3. Configuration and Authorization Rules

- `b1back` values: 0 = no backup, 1 = standard backup, 2 = full backup. Value 2 (full backup) is incompatible with any savefile option.
- `b1savf` values: 0 = no savefile, 1 = savefile option A, 2 = savefile option B. Savefile requires `b1back = 1`.
- `b1prc1` values: 0 = extra jobs disabled, 1 = extra jobs enabled. When enabled, at least one of `c1pro1` or `c1pro2` must be a non-blank program name.
- `b1genp = 1` triggers immediate CL generation via `AN012C` after saving parameters. The generation creates a new night-job CL program.
- LDA position 944–946 (`l_firm`): night job parameters are firm-scoped in `AVERPF`. All reads and writes use the firm from LDA.
- AN020R uses the library list at runtime to locate `APGMPF`. The active library list determines which programs/files are included in the reorganise source.

---

## 4. Financial / Transactional Rules

- The night routine module is a system administration module — it does not perform financial transactions. No financial validation rules apply.
- `AN050R` records job execution history in `AHSTPF`. History entries are write-only from the RPG layer; no aggregation or balance calculations occur.

---

## 5. Status and Lifecycle Rules

- `AVERPF` stores the current night-job configuration. Writing new parameters via AN010R overwrites the existing configuration for the firm.
- `AHSTPF` history entries are append-only from AN050R. Existing history records are not modified.
- Once a night job CL source is generated (`b1genp = 1` → `AN012C`), the previous generated source in `QCLLESRC` is overwritten. There is no version history of generated CL programs.
- AN040R returns the presence/absence of `AN044C` as a binary flag. The caller uses this to determine whether re-generation is needed.

---

## 6. Special Conditions

- The backup/savefile combination rules are independent of each other: both the "full backup + savefile" check (`*in31`) and the "savefile without standard backup" check (`*in34`) are evaluated, meaning it is possible to have multiple simultaneous error indicators set.
- Extra jobs validation (`*in32` and `*in33`) are also independently evaluated — both can be set simultaneously if the flag is in an inconsistent state.
- AN020R skips QS36F-derived files specifically to avoid reorganising System/36 environment objects, which use different file organisation and may not be safely reorganised with standard `RGZPFM`.
- History code 99 is a special escape code that writes raw LDA text (positions 701–740) rather than looking up a predefined text string. This is used for free-form status messages during the night job execution.
- AN050R calls `AN050C` (a CL delay) before writing history. The purpose is to insert a short wait so that concurrent processes do not write conflicting history entries at the same timestamp.

---

## 7. Subprogram Calls Affecting Logic

| Called Program | Called From | Purpose / Effect on Logic |
|---|---|---|
| `AN012C` | AN010R | CL driver: generates night-job program from current `AVERPF` parameters |
| `AN032C` | AN030R | CL driver: updates IPL schedule with entered time and day |
| `AN050C` | AN050R | CL delay: short wait before writing history to `AHSTPF` |
| `AA073R` | AN010R (v6.20+) | Subsystem list lookup for parameter screen display |

---

## 8. Reference Tables

| Table | Role in Module |
|---|---|
| `AVERPF` | Night job parameter file — holds backup type, savefile option, extra-job flags, and generation trigger |
| `AHSTPF` | Night job history file — append-only execution history records |
| `APGMPF` | Program register — source for files eligible for RGZPFM in AN020R |
| `QCLLESRC` | CL source physical file — target for generated night-job CL; checked for AN044C existence by AN040R |
