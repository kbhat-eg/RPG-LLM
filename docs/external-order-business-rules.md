# Business Rules: External Order (NO Module)

**System:** NXCLOUD
**Module Prefix:** NO
**Programs Analyzed:** NO100R, NO101R, NO110R, NO120R, NO750R, NO751R, NO752R
**NXKORR Overrides:** None found
**Date:** 2026-03-16

---

## 1. Prerequisites / Master Data Requirements

- External order headers are stored in `NOHEPF`. NO100R requires at least one open external order to display a meaningful subfile. The program reads via multiple logical views: `NOHEL2` (by reference), `NOHEL3` (by customer name), `NOHEL4` (by project), `NOHEL5` (by user), `NOHEL6` (by system).
- External order lines are stored in `NODTPF`. NO120R iterates `NODTPF` line by line when transferring to a sales order. All lines must be transferable for the operation to complete.
- Customer credit information is held in `RKMEPF` (credit memo balance). NO120R checks `RKMEPF` for the customer's credit limit. A missing record in `RKMEPF` means no credit memo balance is tracked — the credit check may pass by default.
- Order type definitions are in `VOTYPPF`. NO120R reads `VOTYPPF` to obtain the order type and number series for the new sales order. If the order type is not found in `VOTYPPF`, the transfer cannot proceed.
- Warehouse and system settings are in `FSTSPF`. NO120R reads `FSTSPF` to resolve the source warehouse. If the warehouse is not configured in `FSTSPF`, transfer is blocked.
- Items must be defined in `VVARPF`. The CO402R switch `ukjent_vare_ok` (version 7.27) controls whether unknown items block the transfer or are allowed to pass.
- EDI staging data is in `EDHEPF` (header) and `EDDTPF` (lines). These are cleaned up by NO120R after successful transfer (version 6.35).
- For Norgros WEB order download (NO750R), an external URL parameter and customer number are required. The download is performed by calling `NO750C` with these parameters.

---

## 2. Validation Rules

### NO100R — External Order Header List

**EDI Import Lock:**
- F14 (import EDI orders): triggers the `hent_meld` subroutine, which checks the EDI import status. If `w_hent <> *blanks` (an EDI import is already in progress) → the operation is blocked and the message "*** EDI innhenting pågår ***" is displayed. The user is redirected back to the list (`goto b2taga`).

**Concurrent Access Lock (version 6.25):**
- NO100R locks a selected external order post using `*hival` in `NONUMM` (external order number lock field). This prevents NO120R or another session from simultaneously transferring the same order. If the record is already locked, the read will wait or fail with a lock contention error.

**F16 — Deleted SMK Orders:**
- Pressing F16 changes the filter to `p_kode = '*SS'`, showing only deleted SMK (EDI system) orders. This is a display-only filter, not a blocking rule.

**Unknown Item Switch (version 7.27):**
- CO402R switch `ukjent_vare_ok`: if this switch is active, items not found in `VVARPF` are allowed through. If the switch is not active, unknown items block the transfer.

### NO110R — Status Codes Display
- Read-only program. No validation rules — displays `NOHEPF.NOMELD` status field (256 chars as 8×32 lines).

### NO101R — External Order Message Display
- Read-only program. No validation rules.

### NO120R — External Order Transfer to Sales Order

**Credit Limit Check:**
- `RKMEPF` (credit memo balance) is read for the customer. If the customer's outstanding memo balance exceeds the credit limit configured in the customer record → `b_sper = *on` is set.
- If `b_sper = *on` (credit block) → the transfer is prevented. The user sees a credit block message and cannot proceed without manual override or credit limit adjustment.

**Extended Memo Days (CO402R switch, version 7.09):**
- CO402R switch `UTVIDET_MEMODAGER`: if active, an extended number of days is used when evaluating outstanding memo items for credit checking. This can loosen the credit block in some configurations.

**Order Type / Number Series:**
- `VOTYPPF` must contain a valid record for the order type being used. If not found → transfer blocked (number series cannot be determined).

**Warehouse / System Lookup:**
- `FSTSPF` must contain the warehouse/system configuration. If not found → transfer blocked.

**Discount Limit Check:**
- `NN750C` is called to check the almenning (customer discount) limit. If the discount on an order line exceeds the limit → the line transfer is blocked.

**Concurrent Lock (version 6.25):**
- NO120R writes `*hival` into `NONUMM` on `NOHEPF` when beginning transfer. This is the same lock field that NO100R checks. If NO100R detects this lock, it shows the order as "in transfer" and prevents a second transfer attempt.

**Existing Quote Replacement (version 8.01):**
- If an existing quote in `NOH2PF` has the same external reference number as the order being transferred → the old quote is deleted before the new sales order is created.

**EDI Cleanup (version 6.35):**
- After successful transfer, corresponding `EDHEPF` and `EDDTPF` records are deleted. If cleanup fails, it is treated as a non-critical post-transfer step.

**Text Copy (version 7.01):**
- Order texts from `XOTXPF` (external order text) are copied to `FOTXPF` (sales order text) during transfer. Missing `XOTXPF` texts are not blocking — the sales order is created without those text lines.

### NO750R — Norgros WEB Order Download
- Calls `NO750C` with firm, customer number, project, delivery type, and URL. If `NO750C` fails (network error, invalid URL, or authentication failure), an error indicator is returned to the calling menu — but NO750R itself does not validate the parameters before the call.

---

## 3. Configuration and Authorization Rules

- CO402R switch `ukjent_vare_ok` (NO100R/NO120R, version 7.27): controls whether items not in `VVARPF` are allowed. Default (switch not found) = items must exist in `VVARPF`.
- CO402R switch `UTVIDET_MEMODAGER` (NO120R, version 7.09): extends the memo-day window for credit evaluation. When active, more recent credit events are included in the balance, potentially relaxing the credit block.
- LDA position 944–946 (`l_firm`): all NO-module programs scope to `l_firm`. Cross-firm records are excluded.
- LDA position 1001–1002 (`l_lage`): active warehouse, used in `FSTSPF` lookup during transfer.
- LDA position 911–920 (`l_user`): the initiating user is recorded on created sales orders and on the `NOHEPF` lock field during transfer.

---

## 4. Financial / Transactional Rules

- Credit limit enforcement: `RKMEPF.RKSALD` (memo balance) is compared against the customer's configured credit limit. If `RKMEPF.RKSALD > credit_limit` → `b_sper = *on` → transfer blocked.
- Discount limit enforcement: `NN750C` validates that line-level discounts do not exceed the almenning discount maximum. Exceeding the limit blocks that line's transfer.
- Sales order creation (NO120R) writes to `FOHEPF` (sales order header) and `FODTPF` (sales order lines) using the number series from `VOTYPPF`. These records constitute financial commitments once created.
- The transfer is not reversible from NO100R once completed — the external order `NOHEPF` is either deleted or marked as transferred, and the sales order exists in `FOHEPF`/`FODTPF`.

---

## 5. Status and Lifecycle Rules

- `NOHEPF.NOSTUS` (external order status): NO100R uses this to filter the display. F16 shows status '*SS' (deleted) orders; default shows only open orders.
- Lock field `NOHEPF.NONUMM = *hival`: used to indicate an order is being transferred. NO100R checks this field and shows a visual lock indicator. When transfer completes, the lock is released (the record is deleted or the lock value is cleared).
- After transfer via NO120R: the external order may be deleted from `NOHEPF` or set to a transferred status, depending on system configuration. EDI source records in `EDHEPF`/`EDDTPF` are deleted.
- NO100R uses `NONUMM` lock with `*hival` — if the program abnormally terminates during transfer, the lock remains on the record and must be manually cleared by an administrator.

---

## 6. Special Conditions

- EDI import concurrency: the F14 import in NO100R checks a shared EDI status indicator (`w_hent`). This is a single-threaded guard — only one EDI import can run at a time per firm. Concurrent EDI imports from different sessions will see the blocking message.
- Multiple logical views on `NOHEPF`: the sort sequence changes based on the active F-key in NO100R. Each logical view (`NOHEL2`–`NOHEL6`) has a different key. Switching views rebuilds the entire subfile.
- Norgros WEB orders (NO750R) are downloaded in batch mode; the resulting orders appear in `NOHEPF` after `NO750C` completes. The download is not transactional — a partial download (network interruption) may leave incomplete orders in `NOHEPF`.
- Text transfer (version 7.01): `XOTXPF` → `FOTXPF` copy is done for all text types attached to the external order. The text type codes must be compatible between the external order and sales order text systems.

---

## 7. Subprogram Calls Affecting Logic

| Called Program | Called From | Purpose / Effect on Logic |
|---|---|---|
| `CO402R` | NO100R, NO120R | Reads `ukjent_vare_ok` and `UTVIDET_MEMODAGER` switches |
| `NN750C` | NO120R | Almenning discount limit check; blocks line transfer if discount exceeds limit |
| `NO750C` | NO750R | CL driver for Norgros WEB order download |

---

## 8. Reference Tables

| Table | Role in Module |
|---|---|
| `NOHEPF` | External order header — main data file; lock field `NONUMM` used for transfer concurrency control |
| `NODTPF` | External order lines — iterated line by line during transfer |
| `NOH2PF` | External order quote register — existing quotes checked and deleted on transfer (v8.01) |
| `RKMEPF` | Credit memo balance — customer credit limit evaluation |
| `VOTYPPF` | Order type register — number series and order type validation for new sales order |
| `FSTSPF` | Warehouse/system configuration — required for warehouse resolution during transfer |
| `FOHEPF` | Sales order header — target of transfer from `NOHEPF` |
| `FODTPF` | Sales order lines — target of transfer from `NODTPF` |
| `EDHEPF` | EDI order header — cleaned up after transfer (v6.35) |
| `EDDTPF` | EDI order lines — cleaned up after transfer (v6.35) |
| `XOTXPF` | External order texts — copied to `FOTXPF` during transfer (v7.01) |
| `FOTXPF` | Sales order texts — receives copied texts from external order |
| `VVARPF` | Item register — item validation (unknown item switch controls behaviour) |
