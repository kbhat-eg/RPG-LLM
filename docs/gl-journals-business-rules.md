# Business Rules: GL Journals (RF Module)

**System:** ASOFAK
**Module Prefix:** RF
**Programs Analyzed:** RF010R, RF020R, RF102R, RF105R, RF110R, RF210R, RF220R, RF400R, RF500R, RF501R (NXKORR overrides where present)
**Date:** 2026-03-16

---

## 1. Prerequisites / Master Data Requirements

- Journal batch headers must exist in `RBUNPF` before any posting or distribution can occur. RF102R reads the batch by firm + batch number; if not found, the batch is skipped entirely.
- Batch status in `RBUNPF.RSSTAT` must not equal `'U'` (cancelled/reversed). A cancelled batch causes RF102R to skip posting without error.
- The GL account master `RHOVPF` must contain all accounts referenced in journal line postings. Accounts not present in `RHOVPF` are handled by RF105R by writing the original posting unchanged (no distribution applied).
- Account range boundaries from `RA01PF`/`RASTPF` — specifically `RA1HKT` (top of GL range) and `RA1HRS` (top of customer ledger range) — must be configured. These are used by RF105R to route postings to GL, customer ledger, or supplier ledger.
- Customer master `RKUNPF` must exist for customer-ledger postings. Due-date calculation via RÅ703R requires valid payment terms in `RA03PF`.
- Supplier master `RLEVPF` must exist for supplier-ledger postings. Due-date calculation via RÅ703R requires valid payment terms in `RA03PF`.
- Currency exchange rates must be present in `RA16PF` for any foreign-currency transaction. If the rate is missing, currency conversion produces zero-value postings.
- For GL note maintenance (RF020R), the switch `Sperr_Notat_Kunde` must be configured in `CO402R`. If active, note writes for specific customers are blocked.

---

## 2. Validation Rules

### RF102R – Distribution from Parameters
- Batch not found in `RBUNPF` → skip; no error is raised, the batch number is simply not processed.
- `RBUNPF.RSSTAT = 'U'` (cancelled) → skip posting for this batch.
- If the source data member differs from the expected member (firm-specific member routing) → terminate processing for that batch.
- If the firm number in the batch differs from the expected firm → terminate processing for that batch.
- Billing code `85` (balance posting) → distribution logic is bypassed; posting is written as-is to the ledger.
- Billing code `98` (year-end posting) → distribution logic is bypassed; posting is written as-is.
- Account not found in `RHOVPF` → the original posting is written unchanged to output; no distribution is attempted.

### RF105R – Transaction Splitting/Distribution
- Account ≤ `RA1HKT` → routed to GL main ledger processing (`beh_hovbok` subroutine).
- `RA1HKT` < Account ≤ `RA1HRS` → routed to customer sub-ledger processing (`beh_kunde` subroutine).
- Account > `RA1HRS` → routed to supplier sub-ledger processing (`beh_lever` subroutine).
- `RHOVPF.RHPROJ = 'N'` → project code and activity code are cleared (set to blank) on the GL posting. The posting is still written, but without project dimension.
- `RHOVPF.RHAVDK = 'N'` → department code is cleared on the GL posting. The posting is still written, but without department dimension.
- Currency conversion: if transaction currency differs from base currency, the base-currency amount is calculated using the rate from `RA16PF`. A missing rate produces a zero base-currency amount.
- Due date for customer postings: calculated via RÅ703R using the customer's payment terms. If payment terms are not found, due date defaults to the posting date.
- Due date for supplier postings: calculated via RÅ703R using the supplier's payment terms. If payment terms are not found, due date defaults to the posting date.

### RF400R – Company Change (Firm Number Update)
- No validation is performed. RF400R performs bulk updates of firm number across all RA* configuration tables. It is a maintenance utility invoked only by system administrators; no blocking logic exists in the program itself.

### RF110R – Updates Accounting/Sub-ledger (partial analysis)
- Validates billing codes before posting. Invalid billing codes block individual line postings.
- Due date validation: date fields are tested for validity; invalid dates block posting of that line.

### RH190R – Balance Distribution Parameters
- From department > to department (`a2pavf > a2pavt`) → `*in31` blocks parameter save.
- From account > to account (`a2pktf > a2pktt`) → `*in32` blocks parameter save.
- Invalid date entry → `*in33` blocks parameter save.
- Accounting year > current year (`*year`) → `*in34` blocks parameter save.
- From period > to period (`a2ppef > a2ppet`) → `*in35` blocks parameter save.

---

## 3. Configuration and Authorization Rules

- The switch `Sperr_Notat_Kunde` (block notes for specific customers), read from `CO402R`, controls whether RF020R can write notes to journal postings associated with certain customer accounts. When active, note creation is blocked for those accounts.
- Account range markers `RA1HKT` and `RA1HRS` from the system status record (`RASTPF`) define the split between GL accounts, customer sub-ledger control accounts, and supplier sub-ledger control accounts. These markers must be correctly configured; any misconfiguration causes postings to be routed to the wrong sub-ledger.
- Billing codes 85 and 98 are system-reserved codes that bypass the distribution engine in RF102R. They are used for balance carry-forward and year-end closing entries respectively. No configuration is needed to activate this bypass — it is hard-coded in RF102R.
- Dimension control flags `RHOVPF.RHPROJ` and `RHOVPF.RHAVDK` are set per account in the chart of accounts. These per-account flags determine whether project and department dimensions are retained on postings to that account.
- RF400R (company change) is a destructive bulk update program. Its use should be restricted to initial system setup or firm restructuring. No access control is implemented within the program itself; access restriction must be enforced at the menu/command level.

---

## 4. Financial / Transactional Rules

### Batch Processing (RF102R)
- Distribution processes one batch at a time. A batch corresponds to a group of journal entries created together (e.g., a payment run, a manual journal).
- The distribution engine reads each line of the batch and applies account-specific distribution rules. Lines are passed to RF105R for the actual routing decision.
- Billing codes 85 and 98 do not pass through RF105R. They are written directly to the target table, bypassing all dimension-clearing and sub-ledger routing logic.

### GL Posting (RF105R)
- A posting is split into as many output records as needed: one for the GL main ledger and optionally one for the customer or supplier sub-ledger.
- Project dimension (`RHPROJ`) and department dimension (`RHAVDK`) are account-specific attributes. If an account is marked as not requiring a project (`RHPROJ = 'N'`), any project reference on the input posting is silently cleared — no error is raised.
- Foreign currency transactions produce two amounts: the original currency amount and the base-currency equivalent. The base-currency amount uses the exchange rate from `RA16PF` for the transaction date. Both amounts are stored on the posting record.
- Due date calculation for sub-ledger entries uses RÅ703R, which applies the payment terms' day-count and calendar rules. The result is stored as the due date on the sub-ledger posting.

### Account Range Routing (RF105R)
- The three ranges are mutually exclusive and cover all valid account numbers:
  - 0 to `RA1HKT` (inclusive): GL main ledger accounts
  - `RA1HKT+1` to `RA1HRS` (inclusive): customer control accounts → customer sub-ledger
  - `RA1HRS+1` and above: supplier control accounts → supplier sub-ledger
- An account above `RA1HRS` with no matching supplier in `RLEVPF` still creates a sub-ledger posting (with blank supplier reference). This may cause orphaned sub-ledger entries.

### Transaction Reversal / Re-run (RF500R, RF501R)
- RF500R provides a log overview with reversal and re-run capabilities. A posted transaction can be reversed (creating an offsetting entry) if the accounting period is still open.
- RF501R provides the detail view of individual transactions in the log. Both programs are read-only inquiry tools combined with action capabilities (reverse, re-run); the actual reversal logic is handled by downstream posting programs.

---

## 5. Status and Lifecycle Rules

- Batch status `'U'` in `RBUNPF.RSSTAT` is a terminal state — cancelled batches cannot be re-activated or re-posted through the normal RF102R path. A cancelled batch is permanently excluded from distribution.
- Once a batch passes through RF102R distribution, its status is updated. Attempting to re-run distribution on an already-distributed batch is controlled by the batch status field, preventing double-posting.
- GL postings in `RFORPF` are immutable once written. Corrections require a reversal posting (new offsetting entry) followed by a corrected posting. The RF500R reversal function creates the offsetting entry.
- Year-end postings (billing code 98) carry the accounting year boundary. Once year-end entries are posted, the prior year is effectively closed for normal journal entry.

---

## 6. Special Conditions

- **Billing code bypass (85 and 98):** These two codes bypass the entire RF105R distribution engine. Postings with these codes are written directly without dimension clearing, account validation, or sub-ledger routing. This is intentional — balance carry-forwards and year-end entries must preserve their exact form.
- **Account not in chart of accounts:** Rather than blocking, RF102R writes the original posting unchanged when the account is not in `RHOVPF`. This ensures that orphaned account postings are not silently lost but are recorded, allowing manual cleanup. This is a deliberate "fail-open" design.
- **Project dimension on GL accounts:** The per-account `RHPROJ` flag in `RHOVPF` is the only mechanism preventing project references from appearing on accounts that should not have project dimension. There is no system-level prevention — only account-level configuration.
- **Firm change (RF400R):** This program bulk-updates firm numbers across all RA* tables. It is not reversible without a backup. The program contains no confirmation dialog or transaction rollback.
- **Note blocking (RF020R):** The `Sperr_Notat_Kunde` switch in `CO402R` is a customer-category-level block, not a per-customer block. All customers matching the blocked category cannot have notes written via RF020R while the switch is active.
- **Currency missing rate:** When a foreign-currency transaction has no matching rate in `RA16PF`, the base-currency amount is computed as zero. The posting is still written with zero base-currency value. This does not raise an error at posting time but will cause reporting discrepancies.

---

## 7. Subprogram Calls Affecting Logic

| Caller | Subprogram | Purpose | Blocking Effect |
|--------|-----------|---------|-----------------|
| RF102R | RF105R | Route each journal line to GL/customer/supplier | Account range determines routing; account not in RHOVPF → write unchanged |
| RF105R | RÅ703R | Calculate due date for sub-ledger postings | No block; defaults to posting date if terms not found |
| RF020R | CO402R | Check Sperr_Notat_Kunde switch | Blocks note write if switch active for customer |
| RF102R | (internal) | Batch status check in RBUNPF | Status 'U' skips the entire batch |
| RF110R | RS701R | Billing code validation | Invalid billing code blocks line posting |

---

## 8. Reference Tables

| Table | Description | Key Fields Used |
|-------|------------|-----------------|
| `RBUNPF` | Journal batch header | `RSFIRM`, `RSBILAG`, `RSSTAT` |
| `RFORPF` | GL journal postings (main ledger) | `RKFIRM`, `RKBILAG`, `RKKONT` |
| `RHOVPF` | Chart of accounts | `RHFIRM`, `RHKONT`, `RHPROJ`, `RHAVDK` |
| `RASTPF` | System status / accounting parameters | `RA1HKT`, `RA1HRS`, `RA1AAR` |
| `RKUNPF` | Customer master | `RKKUND`, `RKBETA` |
| `RLEVPF` | Supplier master | `RLFIRM`, `RLLEV` |
| `RA03PF` | Payment terms | `RAFIRM`, `RABETA` |
| `RA16PF` | Currency exchange rates | `RAFIRM`, `RAVALUTA`, `RAKURS` |
| `RA01PF` | Document codes | `RAFIRM`, `RACOD1` |
| `CO402R` | System switches / configuration | Switch name `Sperr_Notat_Kunde`, `OKO_TILGANGSKONTROLL` |
| `RLOSPF` | Supplier ledger postings | `RLFIRM`, `RLLEV` |
| `RLOSOPF` | Customer ledger postings | `RLFIRM`, `RLLEV` |
