# Business Logic for Budget Level Register Operations

The budget level register module (ASSTAT system, SB prefix) maintains the SKNIPF dimension-definition register, which defines budget analysis dimensions (sales dimension codes 11–34), accumulation flags, and version/period metadata. From the SB100R interactive maintenance screen, users create and update budget level definitions, enter budget amounts per period in SKBUPF, delete entire budget versions, copy budgets between years/versions, generate budgets from other budgets, load budgets from flat files, transfer budgets to the GL balance matrix (RHSAPF), and accumulate statistics into the accumulation register (SKAKPF). Every blocking rule is derived from key existence checks in SKNIPF and SKBUPF, and from the accumulation flag `snakkk` in SKNIPF.

---

## Prerequisites / Master Data Requirements

- **SKNIPF** (budget level/dimension register): A `sknipf` record keyed by `firm + kode` must exist before any budget entry (option 8), print (option 6), generation (option 60), file load (option 62), GL transfer (option 78), or accumulation (option 99) can run. All sub-programs receive the code as a parameter and chain or setll on SKNIPF at startup.
- **Accumulation flag** (`smakkk` / `snakkk` in SKNIPF): SBADER (and SCADER) skips any code where `smakkk = *blank`. If the accumulation flag is blank, SBANYR will not accumulate statistics for that code — not an error, but budget accumulation silently does nothing.
- **SKBUPF** (budget register): Must have records for the given firm/year/version/code/aabb combination before SBDELR or SBOVFR can find anything to delete or transfer. Empty SKBUPF for the specified combination causes zero deletes and zero GL transfers without error.
- **VVARPF** (item register): SBFILR and SBGENR attempt to resolve item dimension keys (kode position 34 = vare) via VVARL1. If `*in14 = *on` (item not found in VVARPF) when loading from flat file, the item dimension key is left blank; the budget line is still written but without item classification.
- **RKUNPF** (customer register): SBGENR and SB110R use RKUNPF to map customer dimension keys (kode position 25 = kund). If not found, the customer dimension remains at its default zero/blank.
- **RHSAPF** (GL balance matrix): Required by SBOVFR for the GL transfer phase. SBOVFR chains RHSAPF using `firm + year + type=3 + kont + spes + avde`. If the record does not exist (`*in12 = *on`), SBOVFR creates a new one. If it does exist, it updates (subtracts budget amounts). RHSAPF must be accessible in the library list.
- **SSTAPF** (statistics register): SBANYR reads SSTAPF as primary input for accumulation. If SSTAPF has no records for the firm/year, the accumulation phase produces no output.
- **VA720R** (GL account mapping): Called by SBOVFR in both xslett (delete) and xgener (write) phases. Must be present. If VA720R cannot resolve the GL account (`d_kko1 = 0`), RHSAPF is chained with account zero, which may produce incorrect GL records.

---

## Validation Rules

- **Delete budget block** (SB100R D1WIN): The delete dialog (option 4 in subfile) will only call SBDELR if `d1hhaa <> 0 OR d1paar <> 0`. If both year fields are zero, SBDELR is not called. The level register record itself (SKNIPF) is deleted from D1WIN only if `d1valg = 'J'` (user confirms). An `N` answer preserves the SKNIPF record but still runs SBDELR if year parameters are non-zero.
- **Budget level code existence** (SB100R C2BLD): Chains SLNIL1R on `firm + kode`. If found (`*in60 = *off`), the screen shows "Existing record" (`mld(2)`). If not found, shows "New record" (`mld(1)`). For an update, if SLNIL1R is not found at write time, the record is created (write); if found, it is updated. No block on new entry beyond the firm/code key uniqueness enforced by SKNIPF itself.
- **Accumulation prerequisite** (SBADER / SBANYR): SBADER checks `smakkk cabne *blank xneste` in the full-code loop — codes without an accumulation flag are skipped. A single-code call checks `smakkk cabne *blank xslutt`. This means calling option 99 on a code with no accumulation flag completes with zero records processed.
- **Copy budget — duplicate prevention** (SBCOLR): Chains SLBULU (target year/version key) before writing. If the target record already exists (`*in92 = *off`), the write is skipped. Only new records (`*in92 = *on`) are written. Existing target records are never overwritten during copy.
- **Generate budget — delete then recreate** (SBGENR): Unlike SBCOLR, SBGENR first deletes all target year/version records (xslett phase), then recreates from source. If the source year/version has no records matching `wskode + w_aabb`, the target is left empty after the delete. This is destructive: the old budget is gone with no recovery if the source is also empty.
- **File load record type** (SBFILR): Only records with input indicator `01` (field position 1, value `'c'`/digit) are processed. Record type `99` triggers `goto xslutt`. Any other record type is skipped silently.
- **Budget amount computation** (SBFILR): `wwbu01 = inbu01 * 1000 + inbd01`. Integer parts from positions 49–185, decimal parts from positions 55–189 of the 512-byte SINNPF record. If the decimal part is omitted (zeros), the budget is stored as thousands. No validation on the magnitude of the amounts.

---

## Configuration and Authorization Rules

- **Firm number from LDA** (positions 944–946): SB100R, SC100R, SB110R, and all sub-programs read `l_firm` (or `dsfirm` in sub-programs via LDA) as the firm key. No cross-firm operation is possible from a single session.
- **Version and aabb** (budget type indicator):
  - `aabb = 'B'` designates a budget (Budsjett)
  - `aabb = 'P'` designates a prognosis (Prognose)
  - Version code (`scvers`, `skvers`) is a 2-character alphanumeric defined by the user (e.g., `'O '` = original). The SBDELR, SBCOLR, SBGENR, and SBOVFR programs accept `aabb` and version as parameters and apply them as key components; incorrect values produce no error but also no effect.
- **Key structure of SKBUPF** (budkey KLIST): The full budget key includes `firm + hhaa + vers + kode + aabb + avde + lage + otyp + land + dist + selg + lkat + ldor + ogrp + hgrp + ugrp + vare`. All 17 key components must be set before a chain or setll on SKBUPF. SB110R, SBFILR, SBGENR, and SBOVFR all initialize unused dimension fields to zero/blank before the KLIST.
- **GL transfer type** (SBOVFR): RHSAPF records written by SBOVFR have `ritype = 3` (budget type) and `riraar = wshhaa` (budget year). If a GL transfer is run twice for the same year/version/code, the xslett phase deletes existing type-3 records before xgener recreates them. The delete uses the same account mapping; if VA720R returns a different account on re-run, orphaned records may remain.

---

## Financial / Transactional Rules

- **Budget amounts stored**: SKBUPF holds 12 monthly period amounts (`sbbu01`–`sbbu12`). SB110R displays these alongside accumulation register actuals (`skakpf` saldo `shsa01`–`shsa12`). The amounts are signed integers; no automatic rounding.
- **GL transfer amount sign** (SBOVFR): Budget amounts are transferred to RHSAPF as negated values: `sub sbbu01 risa01` through `sub sbbu12 risa12`. This means a positive budget amount in SKBUPF produces a negative amount in RHSAPF. This convention must match the GL reporting system's expectation.
- **Statistics accumulation** (SBANYR → VS001R): SBANYR reads SSTAPF statistics records and calls VS001R for each record. VS001R updates SKAKPF with quantities (`sianta`), amounts (`sisumb`, `sisumk`), and discounts (`sirab1 + sirab2 + sirabo`). Period is taken from `sipmnd` (statistics period). If SSTAPF records span multiple years but SBANYR is called with a specific year, only records where `sipaar = w_hhaa` are processed.
- **Accumulation register cleared before accumulation** (SBANYR → SCADER): SBANYR first calls SCADER (or SBADER for SB-prefix variant) to delete all existing SKAKPF records for the firm/year/code. This is always destructive before re-accumulation. The delete loop in SCADER reads all SLAKL1 records for the key and deletes each one found in SLAKLUR.

---

## Status and Lifecycle Rules

- **SKNIPF record lifecycle**: Created via SB100R option F6 (new record screen C1BLD). Updated via option 2 (C2BLD). Deleted via option 4 with `d1valg = 'J'` confirmation. There is no soft-delete or status flag; deletion is immediate and physical.
- **Budget version lifecycle**: A budget version exists as long as SKBUPF records with matching keys exist. Deleting all SKBUPF records for a version (SBDELR) effectively removes the version. The SKNIPF record defining the code remains unless separately deleted.
- **Accumulation register lifecycle**: SKAKPF is populated by SBANYR and deleted by SBADER/SCADER. It is a derived register; all data can be regenerated by re-running SBANYR as long as SSTAPF data is intact.
- **GL balance matrix lifecycle**: RHSAPF type-3 records are managed exclusively by SBOVFR (and SCOVFR for purchase budget). Running SBOVFR again for the same parameters will delete and recreate them. Manual changes to RHSAPF type-3 records will be overwritten on the next transfer.

---

## Special Conditions (Program-Specific)

- **SB100R option 60 — generate budget**: Opens a dialog (T0WIN) to specify source year/version and target year/version. Calls SBGENR. SBGENR deletes the entire target code/year/version from SKBUPF before regenerating. If the source year/version is empty, the result is an empty target. There is no undo.
- **SB100R option 62 — load from file**: Opens dialog (V0WIN) to specify source flat file (SINNPF), library, and parameters. Calls SBFILR. SBFILR reads the fixed 512-byte SINNPF records and creates SKBUPF entries. Existing SKBUPF records for the same key are not deleted first; duplicate key writes are blocked by SKBUPF's key uniqueness. A partial load followed by a re-load may therefore leave the previous records intact for keys not covered by the new file.
- **SB100R option 78 — transfer to GL**: Opens dialog (V1WIN) to confirm year/version/code/aabb. Calls SBOVFR. SBOVFR resolves GL accounts via VA720R for each budget line. If VA720R fails for any line, that line is skipped in both delete and generate phases. No error is raised; the GL record for that line remains at its previous state.
- **SB100R option 99 — accumulate**: Opens a U0WIN dialog to confirm. Calls SBANYR which calls SBADER then loops SSTAPF calling VS001R. Pressing F3/F12 at U0WIN exits without accumulation.
- **SB200R**: Read-only version of SB100R; option 5 opens SB210R in view mode. No modifications possible.
- **SB210R**: Detail budget entry per period. Shows budget (`skbupf`) alongside actuals (`skakpf`). Users can enter or modify `sbbu01`–`sbbu12` directly. No specific blocking beyond the KLIST/chain requirement that the SKNIPF code and firm combination must already exist.

---

## Subprogram Calls Affecting Logic

| Program | Trigger | Logic | Impact |
|---------|---------|-------|--------|
| SBDELR | SB100R D1WIN when d1hhaa <> 0 or d1paar <> 0 | Deletes all SKBUPF records for firm+code+aabb+year+prior-year+version+prior-version | Irreversible bulk delete; no confirmation beyond D1WIN dialog |
| SBCOLR | SB100R K1WIN copy dialog | Reads source SKBUPF; writes target only if target key does not exist | Safe copy — never overwrites existing target records |
| SBGENR | SB100R T0WIN generate dialog | Deletes target, then reads source and writes to target with new dimension keys | Destructive — target is deleted before source is re-read |
| SBFILR | SB100R V0WIN load from file | Reads SINNPF flat file; writes SKBUPF records | Does not pre-delete; duplicate keys are silently skipped |
| SBOVFR | SB100R V1WIN GL transfer dialog | Phase 1: deletes RHSAPF type-3; Phase 2: recreates from SKBUPF via VA720R | Two-phase destructive transfer; account mapping via VA720R required |
| SBANYR | SB100R U0WIN accumulate | Calls SBADER, then reads SSTAPF, calls VS001R for each record | Clears and rebuilds SKAKPF accumulation register |
| SBADER | Called by SBANYR | Deletes all SKAKPF records for firm/year/code | Prerequisite for SBANYR; must run before VS001R calls |
| SB211R / SB611R | Option 6 print in SB100R | Prints budget from SKBUPF filtered by parameters | Print only; calls VA720R for label patterns |
| VA720R | SBOVFR xslett and xgener | Returns GL account (d_kko1), specialization (d_ksp1), department (d_kav1) for budget line | Required for GL transfer; blank account produces GL records with account 0 |
| VS001R | SBANYR for each SSTAPF record | Updates SKAKPF with statistics period amounts | Core accumulation engine; must be present in library list |

---

## Reference Tables

| Table | Key Fields | Purpose |
|-------|-----------|---------|
| SKNIPF (SLNIL*) | firm, kode | Budget level/dimension definition register — defines code structure and accumulation flag |
| SKBUPF (SLBUL*) | firm, hhaa, vers, kode, aabb, avde, lage, otyp, land, dist, selg, lkat, ldor, ogrp, hgrp, ugrp, vare | Budget register — 12 monthly period amounts per dimension combination |
| SKAKPF (SLAKL*) | firm, hhaa, kode, avde, lage, otyp, land, dist, selg, lkat, ldor, ogrp, hgrp, ugrp, vare | Accumulation register — statistics actuals per dimension |
| SSTAPF (SISTL*) | firm, paar, binr, line, bida | Statistics register — source data for SBANYR |
| RHSAPF (RHSAL*) | firm, raar, type, kont, spes, avde | GL balance matrix — receives budget amounts from SBOVFR with ritype=3 |
| SINNPF | sequential | Flat file input for SBFILR (512-byte fixed records) |
| VVARPF | firm, vare | Item register — dimension key lookup for vare (code 34) |
| RKUNPF | firm, kund | Customer register — dimension key lookup for kund (code 25) |
| VOTYPPF | firm, otyp | Order type — dimension key for otyp (code 13) |
| VOGRPF / VHGRPF / VUGRPF | firm, ogrp/hogr/ugrp | Group registers — dimension key lookups for codes 31–33 |
| RA13PF | firm, mkod | Code register — used by SB110R for market code dimension |
