# Business Logic for Purchase Budget Accumulation Operations

The purchase budget accumulation module (ASSTAT system, SC prefix) maintains the SLNIPF dimension-definition register, which defines purchase budget analysis dimensions (supplier, order type, item group hierarchies, item, warehouse, and related codes), accumulation flags, and version/period metadata. From the SC100R interactive maintenance screen, users create and update purchase budget level definitions, enter budget amounts per period in SLBUPF, delete entire budget versions, copy budgets between years/versions, generate budgets from other budgets, load budgets from flat files, transfer budgets to the GL balance matrix (RHSAPF), and accumulate supplier statistics into the accumulation register (SLAKPF). Every blocking rule is derived from key existence checks in SLNIPF and SLBUPF, and from the accumulation flag `smakkk` in SLNIPF. This module is the purchase-side parallel of the SB prefix sales budget module; field prefixes and table names differ but the control flow is structurally identical.

---

## Prerequisites / Master Data Requirements

- **SLNIPF** (purchase budget level/dimension register): A `slnipf` record keyed by `firm + kode` must exist before any budget entry (option 8), print (option 6), generation (option 60), file load (option 62), GL transfer (option 78), or accumulation (option 99) can run. All sub-programs receive the code as a parameter and chain or setll on SLNIPF at startup.
- **Accumulation flag** (`smakkk` in SLNIPF): SCADER skips any code where `smakkk = *blank`. If the accumulation flag is blank, SCANYR will not accumulate statistics for that code — not an error, but purchase budget accumulation silently does nothing for that code.
- **SLBUPF** (purchase budget register): Must have records for the given firm/year/version/code/aabb combination before SCDELR or SCOVFR can find anything to delete or transfer. An empty SLBUPF for the specified combination causes zero deletes and zero GL transfers without error.
- **VVARPF** (item register): SCFILR and SCGENR attempt to resolve item dimension keys (kode position 34 = vare) via VVARL1. If `*in14 = *on` (item not found in VVARPF) when loading from a flat file, the item dimension key is left blank; the budget line is still written but without item classification.
- **RLEVPF** (supplier register): SCGENR and SC110R use RLEVL1 to map supplier dimension keys (kode position = levr). If not found, the supplier dimension remains at its default zero/blank.
- **RHSAPF** (GL balance matrix): Required by SCOVFR for the GL transfer phase. SCOVFR chains RHSAPF using `firm + year + type=3 + kont + spes + avde`. If the record does not exist (`*in12 = *on`), SCOVFR creates a new one. If it does exist, it updates (subtracts budget amounts). RHSAPF must be accessible in the library list.
- **SISTPF** (supplier statistics register): SCANYR reads SISTPF as primary input for accumulation. If SISTPF has no records for the firm/year, the accumulation phase produces no output.
- **VA720R** (GL account mapping): Called by SCOVFR in both xslett (delete) and xgener (write) phases via subroutine hntkon. Must be present. If VA720R cannot resolve the GL account (`d_kko1 = 0`), RHSAPF is chained with account zero, which may produce incorrect GL records.
- **VOTYPPF / VHGRPF / VOGRPF / VUGRPF**: Used by SC110R and SCGENR to resolve order type, item group hierarchy dimension keys. If any of these lookups fail, the corresponding dimension field is left blank/zero; no error is raised but the budget line may be mis-keyed.

---

## Validation Rules

- **Delete budget block** (SC100R D1WIN): The delete dialog (option 4 in subfile) will only call SCDELR if `d1hhaa <> 0 OR d1paar <> 0`. If both year fields are zero, SCDELR is not called. The level register record itself (SLNIPF) is deleted from D1WIN only if `d1valg = 'J'` (user confirms). An `N` answer preserves the SLNIPF record but still runs SCDELR if year parameters are non-zero.
- **Budget level code existence** (SC100R C2BLD): Chains SLNIL1R on `firm + kode`. If found (`*in60 = *off`), the screen shows "Existing record" (`mld(2)`). If not found, shows "New record" (`mld(1)`). For an update, if SLNIL1R is not found at write time, the record is created (write); if found, it is updated. No block on new entry beyond the firm/code key uniqueness enforced by SLNIPF itself.
- **Accumulation prerequisite** (SCADER / SCANYR): SCADER checks `smakkk cabne *blank xneste` in the full-code loop — codes without an accumulation flag are skipped. A single-code call checks `smakkk cabne *blank xslutt`. Calling option 99 on a code with no accumulation flag completes with zero records processed.
- **Copy budget — duplicate prevention** (SCCOLR): Chains SLBULU (target year/version key) via tilkey before writing. If the target record already exists (`*in92 = *off`), the write is skipped. Only new records (`*in92 = *on`) are written. Existing target records are never overwritten during copy.
- **Generate budget — delete then recreate** (SCGENR): Unlike SCCOLR, SCGENR first deletes all target year/version records (xslett phase), then recreates from source. If the source year/version has no records matching `wskode + w_aabb`, the target is left empty after the delete. This is destructive: the old budget is gone with no recovery if the source is also empty.
- **File load record type** (SCFILR): Only records with input indicator `01` (field position 1, value `'c'`/digit) are processed. Record type `99` triggers `goto xslutt`. Any other record type is skipped silently.
- **Budget amount computation** (SCFILR): `wwbu01 = inbu01 * 1000 + inbd01`. Integer parts from positions 49–185, decimal parts from positions 55–189 of the 512-byte SINNPF record. If the decimal part is omitted (zeros), the budget is stored as thousands. No validation on the magnitude of the amounts.

---

## Configuration and Authorization Rules

- **Firm number from LDA** (positions 944–946): SC100R and all sub-programs read `l_firm` (or `dsfirm` in sub-programs via LDA) as the firm key. No cross-firm operation is possible from a single session.
- **Version and aabb** (budget type indicator):
  - `aabb = 'B'` designates a budget (Budsjett)
  - `aabb = 'P'` designates a prognosis (Prognose)
  - Version code (`scvers`, `slvers`) is a 2-character alphanumeric defined by the user. The SCDELR, SCCOLR, SCGENR, and SCOVFR programs accept `aabb` and version as parameters and apply them as key components; incorrect values produce no error but also no effect.
- **Key structure of SLBUPF** (budkey KLIST): The full purchase budget key includes `firm + hhaa + vers + kode + aabb + avde + lage + levr + land + dist + otyp + lkat + ogrp + hgrp + ugrp + vare`. All 16 key components must be set before a chain or setll on SLBUPF. SC110R, SCFILR, SCGENR, and SCOVFR all initialize unused dimension fields to zero/blank before the KLIST.
- **GL transfer type** (SCOVFR): RHSAPF records written by SCOVFR have `ritype = 3` (budget type) and `riraar = wshhaa` (budget year). The account is resolved by subroutine hntkon which calls VA720R. If a GL transfer is run twice for the same year/version/code, the xslett phase deletes existing type-3 records before xgener recreates them. If VA720R returns a different account on re-run, orphaned records may remain.
- **Supplier dimension** (SC110R / SCGENR): Unlike the SB sales budget, SCGENR and SC110R use RLEVL1 (supplier logical) rather than RKUNPF (customer logical) to resolve the dimension key for the levr field. An absent supplier in RLEVPF leaves the dimension blank without blocking.

---

## Financial / Transactional Rules

- **Budget amounts stored**: SLBUPF holds 12 monthly period amounts (`scbu01`–`scbu12`). SC110R displays these alongside accumulation register actuals (`slakpf` saldo `shsa01`–`shsa12`). The amounts are signed integers; no automatic rounding.
- **GL transfer amount sign** (SCOVFR): Budget amounts are transferred to RHSAPF as negated values via `sub scbu01 risa01` through `sub scbu12 risa12`. This means a positive budget amount in SLBUPF produces a negative amount in RHSAPF. This convention must match the GL reporting system's expectation; it mirrors the SB sales budget sign convention.
- **Statistics accumulation** (SCANYR → VS001R): SCANYR reads SISTPF supplier statistics records and calls VS001R for each record. VS001R updates SLAKPF with quantities (`sianta`), amounts (`sisumb`, `sisumk`), and discounts (`sirab1 + sirab2 + sirabo`). Period is taken from `sipmnd` (statistics period). If SISTPF records span multiple years but SCANYR is called with a specific year, only records where `sipaar = w_hhaa` are processed.
- **Accumulation register cleared before accumulation** (SCANYR → SCADER): SCANYR first calls SCADER to delete all existing SLAKPF records for the firm/year/code. This is always destructive before re-accumulation. The delete loop in SCADER reads all SLAKL1 records for the key and deletes each one found in SLAKLUR.

---

## Status and Lifecycle Rules

- **SLNIPF record lifecycle**: Created via SC100R option F6 (new record screen C1BLD). Updated via option 2 (C2BLD). Deleted via option 4 with `d1valg = 'J'` confirmation. There is no soft-delete or status flag; deletion is immediate and physical.
- **Budget version lifecycle**: A budget version exists as long as SLBUPF records with matching keys exist. Deleting all SLBUPF records for a version (SCDELR) effectively removes the version. The SLNIPF record defining the code remains unless separately deleted.
- **Accumulation register lifecycle**: SLAKPF is populated by SCANYR and deleted by SCADER. It is a derived register; all data can be regenerated by re-running SCANYR as long as SISTPF data is intact.
- **GL balance matrix lifecycle**: RHSAPF type-3 records written by SCOVFR are managed exclusively by SCOVFR (and SBOVFR for sales budget). Running SCOVFR again for the same parameters will delete and recreate them. Manual changes to RHSAPF type-3 records will be overwritten on the next transfer.

---

## Special Conditions (Program-Specific)

- **SC100R option 60 — generate budget**: Opens a dialog (T0WIN) to specify source year/version and target year/version. Calls SCGENR. SCGENR deletes the entire target code/year/version from SLBUPF before regenerating. If the source year/version is empty, the result is an empty target. There is no undo.
- **SC100R option 62 — load from file**: Opens dialog (V0WIN) to specify source flat file (SINNPF), library, and parameters. Calls SCFILR. SCFILR reads the fixed 512-byte SINNPF records and creates SLBUPF entries. Existing SLBUPF records for the same key are not deleted first; duplicate key writes are blocked by SLBUPF's key uniqueness. A partial load followed by a re-load may therefore leave the previous records intact for keys not covered by the new file.
- **SC100R option 78 — transfer to GL**: Opens dialog (V1WIN) to confirm year/version/code/aabb. Calls SCOVFR. SCOVFR resolves GL accounts via hntkon subroutine (which calls VA720R) for each budget line. If VA720R fails for any line, that line is skipped in both delete and generate phases. No error is raised; the GL record for that line remains at its previous state.
- **SC100R option 99 — accumulate**: Opens a U0WIN dialog to confirm. Calls SCANYR which calls SCADER then loops SISTPF calling VS001R. Pressing F3/F12 at U0WIN exits without accumulation.
- **SC200R**: Read-only version of SC100R; option 5 opens SC210R in view mode. No modifications possible.
- **SC210R**: Detail budget entry per period. Shows purchase budget (`slbupf`) alongside actuals (`slakpf`). Users can enter or modify `scbu01`–`scbu12` directly. No specific blocking beyond the KLIST/chain requirement that the SLNIPF code and firm combination must already exist.
- **Supplier vs. customer dimension**: The primary structural difference from the SB module is that SC uses RLEVPF/RLEVL1 (supplier) rather than RKUNPF (customer) for the `levr` dimension key. All other dimension lookups (VVARPF, VOTYPPF, VOGRPF, VHGRPF, VUGRPF) are shared between the two modules.

---

## Subprogram Calls Affecting Logic

| Program | Trigger | Logic | Impact |
|---------|---------|-------|--------|
| SCDELR | SC100R D1WIN when d1hhaa <> 0 or d1paar <> 0 | Deletes all SLBUPF records for firm+code+aabb+year+prior-year+version+prior-version | Irreversible bulk delete; no confirmation beyond D1WIN dialog |
| SCCOLR | SC100R K1WIN copy dialog | Reads source SLBUPF; writes target only if target key does not exist | Safe copy — never overwrites existing target records |
| SCGENR | SC100R T0WIN generate dialog | Deletes target SLBUPF, then reads source and writes to target with new dimension keys via VVARL1 and RLEVL1 | Destructive — target is deleted before source is re-read |
| SCFILR | SC100R V0WIN load from file | Reads SINNPF flat file; writes SLBUPF records; uses RLEVL1 for supplier dimension | Does not pre-delete; duplicate keys are silently skipped |
| SCOVFR | SC100R V1WIN GL transfer dialog | Phase 1: deletes RHSAPF type-3; Phase 2: recreates from SLBUPF via hntkon/VA720R | Two-phase destructive transfer; account mapping via VA720R required |
| SCANYR | SC100R U0WIN accumulate | Calls SCADER, then reads SISTPF, calls VS001R for each record | Clears and rebuilds SLAKPF accumulation register |
| SCADER | Called by SCANYR | Deletes all SLAKPF records for firm/year/code; skips if smakkk = *blank | Prerequisite for SCANYR; must run before VS001R calls |
| VA720R | SCOVFR hntkon subroutine in xslett and xgener phases | Returns GL account (d_kko1), specialization (d_ksp1), department (d_kav1) for budget line | Required for GL transfer; blank account produces GL records with account 0 |
| VS001R | SCANYR for each SISTPF record | Updates SLAKPF with supplier statistics period amounts | Core accumulation engine; must be present in library list |

---

## Reference Tables

| Table | Key Fields | Purpose |
|-------|-----------|---------|
| SLNIPF (SLNIL*) | firm, kode | Purchase budget level/dimension definition register — defines code structure and accumulation flag (`smakkk`) |
| SLBUPF (SLBUL*) | firm, hhaa, vers, kode, aabb, avde, lage, levr, land, dist, otyp, lkat, ogrp, hgrp, ugrp, vare | Purchase budget register — 12 monthly period amounts (`scbu01`–`scbu12`) per dimension combination |
| SLAKPF (SLAKL*) | firm, hhaa, kode, avde, lage, levr, land, dist, otyp, lkat, ogrp, hgrp, ugrp, vare | Accumulation register — supplier statistics actuals per dimension |
| SISTPF (SISTL*) | firm, paar, binr, line, bida | Supplier statistics register — source data for SCANYR |
| RHSAPF (RHSAL*) | firm, raar, type, kont, spes, avde | GL balance matrix — receives purchase budget amounts from SCOVFR with ritype=3 |
| SINNPF | sequential | Flat file input for SCFILR (512-byte fixed records) |
| VVARPF | firm, vare | Item register — dimension key lookup for vare (code 34) |
| RLEVPF | firm, levr | Supplier register — dimension key lookup for levr |
| VOTYPPF | firm, otyp | Order type — dimension key for otyp |
| VOGRPF / VHGRPF / VUGRPF | firm, ogrp/hogr/ugrp | Group registers — dimension key lookups for item group hierarchy |
