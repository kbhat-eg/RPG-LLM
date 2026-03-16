# Purchase Proposal – Business Rules

**Module prefix:** LH
**System:** ASLAGR
**Focus:** What blocks or prevents generation, transfer, or processing of purchase proposals

---

## Prerequisites / Master Data Requirements

- **LH700R – Proposal generation parameters:** Warehouse must be resolvable via `FÅ720R` (warehouse resolver); failure sets *IN31 (blocks generation). Selection option (`b1valg`) must be 1–4; value > 4 sets *IN32 (blocks generation). A valid base date must be entered; invalid date sets *IN33. Calculation method (`r1valg`) must be 1 or 2; any other value sets *IN33 (blocks generation).
- **LH705R – Hand-terminal transfer parameters:**
  - Order type must exist in `VOTYPF` (order type register); not found → *IN31.
  - Supplier must exist in `RLEVPF` (supplier master); not found → *IN32.
  - Warehouse must be resolvable via `FÅ720R`; failure → *IN33.
  - From-date must be a valid date; invalid → *IN34.
  - To-date must be >= from-date; violation → *IN35.
  - To-date must itself be a valid date; invalid → *IN36.
- **LH500R – Historical stock ledger:** Item must exist in `VVARPF`; not found → *IN31. Warehouse must be valid via `FÅ720R`; failure → *IN32. Date must be valid; invalid → *IN33.
- **LH708R – Order balance calculation:** Purchase order header for each order line must exist in `LOHEPF`; lines with no header are skipped entirely.

---

## Validation Rules

| Program | Condition | Indicator | Effect |
|---------|-----------|-----------|--------|
| LH700R | `b1valg > 4` | *IN32 | Blocks proposal generation |
| LH700R | Warehouse not resolved by FÅ720R | *IN31 | Blocks proposal generation |
| LH700R | Invalid base date | *IN33 | Blocks proposal generation |
| LH700R | `r1valg` not 1 or 2 | *IN33 | Blocks proposal generation |
| LH705R | Order type not in `VOTYPF` | *IN31 | Blocks hand-terminal transfer |
| LH705R | Supplier not in `RLEVPF` | *IN32 | Blocks hand-terminal transfer |
| LH705R | Warehouse not resolved by FÅ720R | *IN33 | Blocks hand-terminal transfer |
| LH705R | Invalid from-date | *IN34 | Blocks hand-terminal transfer |
| LH705R | To-date < from-date | *IN35 | Blocks hand-terminal transfer |
| LH705R | Invalid to-date | *IN36 | Blocks hand-terminal transfer |
| LH500R | Item not in `VVARPF` | *IN31 | Blocks stock history inquiry |
| LH500R | Warehouse not resolved by FÅ720R | *IN32 | Blocks stock history inquiry |
| LH500R | Invalid date | *IN33 | Blocks stock history inquiry |

---

## Configuration and Authorization Rules

- The firm number is read from the Local Data Area (LDA positions 944–946). All reads are scoped to that firm.
- LH700R calls `LH701C` (compiled batch proposal generator) after all parameter validation passes. The batch job runs with the validated parameters; no further interactive validation occurs within LH701C.
- LH708R applies multiple filter conditions to purchase order lines before including them in the order-balance calculation. Lines are skipped when:
  - `VLTYPF.vallag <> 1` — line type does not affect warehouse stock (non-inventory lines are excluded)
  - `VOTYPF.vaosys <> 1` OR `VOTYPF.vaoakk <> 1` OR `VOTYPF.vaolag <> 1` — order type flags indicate the order does not trigger stock movement
  - Warehouse on the order header (`lolage`) does not match the target warehouse (`d_lage`)
  - No delivery date is set on any date field of the order line
  - Delivery date > target date (order will not arrive in time)
  - `ldlety = 1` — delivery type 1 (direct delivery, not warehouse receipt) is excluded

---

## Financial / Transactional Rules

- **LH400R – Delete purchase proposal:** Unconditionally deletes all `LFDTPF` (proposal detail) lines for the given proposal number, then deletes the `LFHEPF` (proposal header) record. There are no blocking conditions — the deletion proceeds without confirmation or dependency checks on orders.
- **LH708R – Order balance:** Calculates the open order quantity for each item/warehouse combination by reading `LODTPF` (purchase order detail lines) and summing quantities from lines that pass all filter conditions. The result is used as input to the proposal calculation to avoid over-ordering items already on order.
- **LH712R – Transfer to purchase order:** Converts approved proposal lines from `LFDTPF` into actual purchase order lines in `LODTPF` and headers in `LOHEPF`. This is a large program (> 25,000 tokens, only preview available); the key business rule is that proposal lines must be in an approved state before transfer.

---

## Status and Lifecycle Rules

- **Proposal lifecycle:**
  1. Parameters entered in LH700R → batch job LH701C generates proposal lines in `LFDTPF` with header in `LFHEPF`.
  2. Proposal is reviewed (LH601R prints it; LH502R displays it interactively).
  3. Approved lines are transferred to purchase orders via LH712R.
  4. The proposal is deleted via LH400R after transfer or rejection.
- **LH502R** (interactive display, large file) presents the proposal subfile for review and adjustment. Individual lines can be modified before transfer.
- **LH601R** (print, large file) produces the printed purchase proposal report.
- The proposal header in `LFHEPF` carries the proposal number and status. The detail lines in `LFDTPF` are keyed by firm + proposal number + item.

---

## Special Conditions

- **FÅ720R** is a shared warehouse-resolver subroutine called by LH700R, LH705R, and LH500R. It validates the warehouse code and returns the resolved warehouse number. A failure from FÅ720R is the primary gateway for warehouse-related blocking in all three programs.
- **LH708R order balance filtering:** The combination of `VLTYPF.vallag = 1` (stock-affecting line type) AND `VOTYPF.vaosys = vaoakk = vaolag = 1` (full warehouse-affecting order type) is a strict gate. Purchase orders placed for non-warehouse-affecting order types (e.g., direct-to-customer orders) are correctly excluded from the stock balance so they do not reduce the proposal quantity.
- **Delivery type exclusion in LH708R:** Lines with `ldlety = 1` (direct delivery) are excluded. Direct deliveries bypass the warehouse and do not reduce stock levels, so they must not be counted as incoming stock in the proposal calculation.
- **LH705R hand-terminal transfer:** This program's parameter screen prepares the transfer job for mobile/hand-terminal-based receiving. All six blocking validations must pass before the transfer job is submitted. The strict date-range validation (`to-date >= from-date`) prevents the creation of an impossible date window.

---

## Subprogram Calls Affecting Logic

| Caller | Called Program | Purpose | Blocking Effect |
|--------|---------------|---------|-----------------|
| LH700R | FÅ720R | Warehouse validation/resolution | Failure sets *IN31; blocks generation |
| LH700R | LH701C | Batch proposal generation | Called after all validation passes |
| LH705R | FÅ720R | Warehouse validation/resolution | Failure sets *IN33; blocks transfer |
| LH500R | FÅ720R | Warehouse validation/resolution | Failure sets *IN32; blocks inquiry |

---

## Reference Tables

| Table | Logical View | Key Fields | Usage |
|-------|-------------|-----------|-------|
| LFHEPF | — | firm + proposal number | Proposal header; created by LH701C, deleted by LH400R |
| LFDTPF | — | firm + proposal number + item | Proposal detail lines; built by LH701C, deleted by LH400R, transferred by LH712R |
| LOHEPF | — | firm + order number | Purchase order header; existence checked in LH708R |
| LODTPF | — | firm + order number + line | Purchase order detail; read for balance calculation in LH708R |
| VVARPF | — | firm + item | Active item register; existence required in LH500R |
| RLEVPF | — | firm + supplier | Supplier master; existence required in LH705R |
| VOTYPF | — | firm + order type | Order type register; existence required in LH705R; flags checked in LH708R |
| VLTYPF | — | firm + line type | Line type register; `vallag = 1` required in LH708R |
| RA10PF | — | firm + warehouse | Warehouse register; resolved via FÅ720R |
