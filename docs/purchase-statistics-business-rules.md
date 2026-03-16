# Purchase Statistics – Business Rules

**Module prefix:** SI
**System:** ASSTAT
**Focus:** What blocks or prevents viewing, editing, or processing purchase statistics records

---

## Prerequisites / Master Data Requirements

- A valid statistical year (`b1paar`) must be supplied; year = 0 blocks entry (*IN31 set ON in SI500R).
- Either a supplier number **or** an item number must be supplied; having neither blocks entry (*IN33 set ON in SI500R).
- When a supplier number is given it must exist in `RLEVPF` (supplier master) — missing supplier blocks with *IN34.
- When an item number is given it must exist in either `VVARPF` (active items) or `VVASPF` (deleted items) — item missing from both files blocks with *IN37.
- For the unit-conversion routine (SI750R) the item/unit combination must exist in `VVENPF` (logical view `VVENL1`, key firm+item+unit); if the chain fails (*IN90 ON) the conversion is skipped.
- The conversion unit ratio (`veomre`) must be non-zero; a zero ratio causes SI750R to skip updating the statistics record and goto `avslutt`.

---

## Validation Rules

| Rule | Indicator / Condition | Effect |
|------|-----------------------|--------|
| Year field zero | `b1paar = 0` → *IN31 ON | Blocks SI500R entry, cursor returns to year field |
| Invalid date entered | Date validation fails → *IN32 ON | Blocks entry |
| No supplier AND no item | Both blank → *IN33 ON | Blocks entry |
| Supplier not in RLEVPF | `CHAIN rlevpf` not found → *IN34 ON | Blocks entry |
| Item not in VVARPF or VVASPF | Both CHAINs fail → *IN37 ON | Blocks entry |
| Statistics record not found (SI510R) | `CHAIN sistl1` → *IN60 ON | `GOTO avslutt`; record cannot be viewed |
| Statistics record not found (SI521R) | `CHAIN sistl1` → *IN60 ON | `GOTO avslutt`; edit is blocked |
| Unit not found or zero ratio (SI750R) | `CHAIN vvenl1` → *IN90 ON, or `veomre = 0` | Conversion skipped; `GOTO avslutt` |

---

## Configuration and Authorization Rules

- The program reads firm number from the Local Data Area (LDA positions 944–946, field `l_firm`). All file reads are keyed by firm; records from a different firm are never accessible.
- Logical files used for sequential reads (SISTL3, SISTL4, SISTL6, SISTL7, SISTL8 in SI501R; SISTL9, SISTLB, SISTLC, SISTLD in SI502R) are keyed by `sifirm + sipaar + sivaln` (by-supplier view) or `sifirm + sipaar + sivare` (by-item view). When the key changes during the `READE` loop the read ends; only records matching the selected year and value/item are visible.
- No explicit user-level or role-level access gate is coded within the SI programs themselves; access is controlled at the menu/command level upstream.

---

## Financial / Transactional Rules

- SI521R allows updating the following statistical fields directly on `SISTPF`:
  - `SISUMK` – purchase amount in currency
  - `SIKAMP` – quantity in package units
  - `SIKAVD` – quantity in secondary units
  - `SISELG` – selling quantity
- SI750R updates `SIVENH` (unit code) and `SIANTA` (quantity after conversion) in `SISTPF` using the unit-conversion ratio from `VVENPF.VEOMRE`. The update is applied only when the unit record is found and `veomre <> 0`.
- SI610R copies parameter records from `SLPMPF` (parameter master) to `SLPWPF` (work file) as a batch setup step; no blocking conditions are applied during the copy — all records are transferred unconditionally.

---

## Status and Lifecycle Rules

- Statistics records in `SISTPF` are addressed via eight logical file views (SISTL1–SISTLD). The primary key for a detail record is firm + year + invoice number + invoice line + date (SISTL1/SISTL2).
- SI501R and SI502R display records in a subfile loop; the loop ends when the firm, year, or value/item key changes — this prevents cross-year or cross-supplier contamination of the display.
- Option 5 from the by-supplier list calls SI510R (item detail view). Option 6 calls FO681R (order inquiry). Option 7 calls SH410R (delivery/receipt inquiry). Option 9 calls AX700R followed by AP622C (attachment/document handling).
- Option 2 from the statistics search (SI520R) calls SI521R for direct edit of the statistics record.

---

## Special Conditions

- Item-number lookup in SI500R first checks `VVARPF` for active items; only if not found does it check `VVASPF` (deleted items). This allows viewing historical statistics for items that have since been deleted from the active register, without blocking the inquiry.
- The unit-conversion program SI750R is a batch utility. It rewrites the unit and quantity fields on existing statistics records; if the unit is missing from `VVENPF` the record is left unchanged (no error raised, program simply ends).
- SI610R is a one-way parameter-copy routine with no validation; it is typically called from a batch job at the start of a statistics run.

---

## Subprogram Calls Affecting Logic

| Caller | Called Program | Purpose | Blocking Effect |
|--------|---------------|---------|-----------------|
| SI500R | SI501R | By-supplier statistics display | Called after parameter validation passes |
| SI500R | SI502R | By-item statistics display | Called after parameter validation passes |
| SI510R | (direct read) | Item detail view via SISTL1 | Record not found → goto avslutt |
| SI520R → SI521R | SI521R | Direct statistics record edit | Record not found → goto avslutt |
| SI501R (option 5) | SI510R | Item detail drill-down | — |
| SI501R (option 6) | FO681R | Order/invoice inquiry | — |
| SI501R (option 7) | SH410R | Delivery/receipt inquiry | — |
| SI501R (option 9) | AX700R, AP622C | Document/attachment handling | — |
| SI750R | (file update) | Unit conversion on SISTPF | Missing unit or zero ratio → skipped |

---

## Reference Tables

| Table | Logical View | Key Fields | Usage |
|-------|-------------|-----------|-------|
| SISTPF | SISTL1 | firm + year + invoice + line + date | Primary statistics detail; chained in SI510R and SI521R |
| SISTPF | SISTL2 | firm + item | Item search in SI520R |
| SISTPF | SISTL3/L4/L6/L7/L8 | firm + year + supplier value | By-supplier sequential read in SI501R |
| SISTPF | SISTL9/LB/LC/LD | firm + year + item | By-item sequential read in SI502R |
| RLEVPF | — | firm + supplier | Supplier existence check in SI500R |
| VVARPF | — | firm + item | Active item existence check in SI500R |
| VVASPF | — | firm + item | Deleted item existence check in SI500R |
| VVENPF | VVENL1 | firm + item + unit | Unit conversion ratio (veomre) in SI750R |
| SLPMPF | — | — | Source parameter records copied by SI610R |
| SLPWPF | — | — | Work file written by SI610R |
