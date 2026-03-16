# Item Assortment – Business Rules

**Module prefix:** VD
**System:** ASOFAK
**Focus:** What blocks or prevents creation, update, import, or export of assortment records

---

## Prerequisites / Master Data Requirements

- **Creating an assortment (VD750R/VD751R):** Assortment code (`b1vsor`) must be non-blank and description (`b1besk`) must be non-blank; either blank field blocks creation.
- **Adding an item to an assortment (VD110R):** The item must exist in `VVARPF` (active item register), verified via logical view `VVARL5` (keyed by supplier). Items not in `VVARPF` cannot be added.
- **VD110R duplicate prevention:** Before inserting a new assortment line, `VSDTL1` is chained; if the record already exists the routine jumps to `end_opprett` — duplicate assortment lines are blocked.
- **VD761R CSV import:** Each imported row's item number must exist in `VVARPF`. Rows with a blank item number or a non-digit item number are silently skipped. Items not found in `VVARPF` are skipped.
- **VD771R type-C assortment check:** Returns `p_stat = '1'` when the item is found in any type `'C'` (central goods) assortment with a matching or zero warehouse; used by callers to block operations requiring the item to be absent from a type-C assortment.
- **VD600R control list:** Warehouse code (`b1lage`) must exist in `RA10PF` (warehouse register) — *IN32 if not. Assortment type (`b1type`) must be blank, `'V'`, `'P'`, or `'C'` — *IN33 if not.
- **VD760R special price import:** In addition to warehouse and type rules above, assortment code must be non-blank (*IN31), filename must be non-blank (*IN34), and folder must be non-blank (*IN35).

---

## Validation Rules

| Program | Condition | Indicator | Effect |
|---------|-----------|-----------|--------|
| VD750R | `b1vsor = *blank` | *IN31 | Blocks assortment creation |
| VD750R | `b1besk = *blank` | *IN32 | Blocks assortment creation |
| VD600R | Warehouse not in `RA10PF` | *IN32 | Blocks control-list run |
| VD600R | Type not in blank/V/P/C | *IN33 | Blocks control-list run |
| VD760R | `b1vsor = *blank` | *IN31 | Blocks special price import |
| VD760R | Warehouse not in `RA10PF` | *IN32 | Blocks special price import |
| VD760R | Type not in V/P/C | *IN33 | Blocks special price import |
| VD760R | Filename blank | *IN34 | Blocks special price import |
| VD760R | Folder blank | *IN35 | Blocks special price import |
| VD761R | Item number contains non-digits | *IN70 | Row skipped |
| VD761R | Item number blank | — | Row skipped |
| VD761R | Item not found in `VVARPF` | `%found` false | Row skipped |
| VD110R | Assortment line already in `VSDTPF` | `CHAIN vsdtl1` found | `GOTO end_opprett`; duplicate blocked |
| VD100R (type C) | Item not in `VVARPF` | — | Item cannot be added (change log 6.21) |
| VD100R (type P) | Supplier not in `RLEVPF` | — | Supplier association cannot be added (change log 6.21) |

---

## Configuration and Authorization Rules

- The firm number is read from the Local Data Area (LDA positions 944–946). All reads and writes are scoped to that firm.
- Functional group (`l_fgrp`, LDA positions 931–933) is used as a default when the screen field `b1fgrp` is blank on VD750R. This ties newly created assortments to the operator's functional group.
- VD700R (item selection file builder) supports 8 different key combinations to locate items within an assortment, applied in order:
  1. Direct item key (firm + assortment + warehouse + item)
  2. Module/model key
  3. Supplier + product groups key
  4. Product groups only
  5. Supplier + groups with zero warehouse
  6. Groups with zero warehouse
  7. Zero warehouse + supplier + groups
  8. Zero warehouse + groups only
- From version 6.23: when `vdldor <> 0` (supplier is set on the assortment line) AND `vclage <> 0` (warehouse is set on the assortment header), VD700R additionally checks `VLAGPF` (warehouse-specific supplier register). This allows a warehouse-local supplier override for the assortment.

---

## Financial / Transactional Rules

- VD751R (create assortment from full item register) is a destructive rebuild operation:
  1. Creates or updates the assortment header in `VSHEPF`.
  2. **Deletes all existing detail lines** in `VSDTPF` for the assortment (`SETLL` + `READE` + `DELETE` loop).
  3. Reads all records from `VVARPF` (logical view `VVARL9`, keyed by firm) and writes a new `VSDTPF` detail record for each item.
  This means any manually added or modified assortment lines are lost when VD751R runs for that assortment code.
- VD751R calls `FO711C` for each item with `vvvare < '10000000'` to resolve item-number matching (renaming/replacement). If a replacement item number is returned, the replacement number is written to `VSDTPF` instead of the original.
- VD755R (assortment copy from file) reads the description from any existing `VSHEPF` record for the same assortment code. If no existing record is found it defaults the description to `'Automatisk opprettet'`. It then calls `VD751C` (compiled version of VD751R) to perform the same full rebuild.
- VD761R (CSV import) creates new `VSDTPF` lines or updates existing ones. The `vdlage` (warehouse) field is set from the parameter; the product-group hierarchy fields (`vdogrp`, `vdhgrp`, `vdugrp`, `vdmodn`) are populated from the corresponding fields on the `VVARPF` item record.

---

## Status and Lifecycle Rules

- **VD601R – duplicate assortment check:** Reads every item in `VVARPF`, then for each item checks all assortment headers of the same type (via `VSHEPF`, logical view `VSHEL3`, keyed by firm + type). For each header it calls `sjekk_vsor` which checks `VSDTPF` for the item in that assortment. If `w_ant > 1` (item found in more than one assortment of the same type), the item is written to the control report as a duplicate. This does not block any operation — it is a reporting-only check.
- Assortment types:
  - `'C'` – Central goods (sentralvarer): items must exist in `VVARPF`
  - `'P'` – Period goods (periodevarer): supplier must exist in `RLEVPF`
  - `'V'` – Item list (vareliste)
  - `'H'` – Shopping list (handleliste nettbutikk)
- VD771R is called by other programs (e.g. order entry, webshop export) to gate operations that require an item NOT to be in a central assortment. A return of `p_stat = '1'` means the item IS in a type-C assortment and the calling program may block the operation.

---

## Special Conditions

- The `sjekk_vsor` subroutine in VD601R performs a two-phase check: first it searches `VSDTPF` directly by item (direct-item assortment entries), and second it searches `VSDTPF` by product-group hierarchy (group-based assortment entries). Both phases increment the same counter `w_ant`. This means an item can be counted once for a direct entry and once for a group-based entry within the same assortment — which may produce false-positive "duplicates" in the report if both assignment methods are used together.
- VD700R is shared infrastructure used by both the webshop export (NN720R) and internal assortment processing. Its 8-key-combination logic ensures that items assigned at any level of the product hierarchy (from specific item number down to broad group) are correctly resolved into the selection file.
- Warehouse `= 0` in the assortment header (`VSHEPF`) means "all warehouses"; VD700R and VD771R both treat a zero warehouse as matching any warehouse value, so an all-warehouse assortment entry applies to warehouse-specific queries.

---

## Subprogram Calls Affecting Logic

| Caller | Called Program | Purpose | Blocking Effect |
|--------|---------------|---------|-----------------|
| VD750R | VD751C | Creates/rebuilds assortment | Called after validation passes |
| VD755R | VD751C | Copy-creates assortment | Called unconditionally (no blocking in VD755R itself) |
| VD760R | VD760C, VD761R | Special price import | Called after parameter validation |
| VD751R | FO711C | Item number matching/replacement | Replaces vvvare with successor number if found |
| VD700R | (direct reads) | VSDTPF + VLAGPF lookups | — |
| Callers of VD771R | VD771R | Check item in type-C assortment | `p_stat = '1'` returned if found; caller may block |

---

## Reference Tables

| Table | Logical View | Key Fields | Usage |
|-------|-------------|-----------|-------|
| VSHEPF | VSHEL3 | firm + type | Assortment headers by type; read in VD601R |
| VSHEPF | VSHEL1 | firm + assortment | Assortment header lookup in VD755R |
| VSHEPF | VSHELUR (update) | firm + assortment | Header create/update in VD751R |
| VSDTPF | VSDTL1 | firm + assortment + warehouse + hierarchy + item | Detail line; duplicate check in VD110R |
| VSDTPF | VSDTL6 | firm + assortment + warehouse + item | Direct-item detail; read in VD601R sjekk_vsor |
| VSDTPF | VSDTLUR (update) | firm + assortment | Detail rebuild in VD751R (delete + write) |
| VVARPF | VVARL1 | firm | Active items; sequential read in VD601R |
| VVARPF | VVARL5 | firm + supplier | Item by supplier; used in VD110R |
| VVARPF | VVARL9 | firm | Full item read in VD751R |
| RLEVPF | — | firm + supplier | Supplier existence; required for type-P assortments (VD100R) |
| RA10PF | — | firm + warehouse | Warehouse existence; required in VD600R and VD760R |
| VLAGPF | — | firm + warehouse + supplier | Warehouse-specific supplier override; checked in VD700R from 6.23 |
| VOGRPF | — | firm + overgroup | Product overgroup; used in group-key lookups in VD700R |
| VHGRPF | — | firm + maingroup | Product main group; used in group-key lookups |
| VUGRPF | — | firm + subgroup | Product subgroup; used in group-key lookups |
