# Statistics Maintenance — Business Rules

## Introduction

The Statistics Maintenance module (module prefix **SX**) provides batch and utility programs for maintaining the sales statistics file (`SSTAPF`), purchase statistics file (`SISTPF`), and related shadow/export tables (`SSTXPF` for Cobuilder, `SCOBPF`). Programs in this module handle:

- Deletion of aged statistics records
- Backfilling of missing product manager codes, cost prices, delivery dates, and discount amounts
- Shadow table population for the Cobuilder integration
- Character-encoding corrections in order text fields

All programs in this module are batch or semi-batch utility programs. Authorization to perform deletions is controlled by the `SX501C` access-check sub-program.

---

## Prerequisites and Master Data Requirements

| Requirement | Table | Key Fields | Used By |
|---|---|---|---|
| Year to delete must be ≤ current system year | — | `*year` | SX500R, SX520R |
| User must pass SX501C authorization check | — | `l_user`, `l_firm` from LDA | SX500R, SX520R |
| Item must have a POS receipt bong reference | `SSTAPF.SFBONG` | — | SX820R, SX840R |
| POS line register record must exist for bong | `BODTPF` | firm + year + bong + item | SX840R |
| Historical purchase invoice header must exist | `SIHEPF` | firm + invoice key | SX831R |
| Cost-price lookup requires VP700R service program | — | — | SX820R |
| Shadow table (SSTXPF) must be initialized before incremental updates | `SSTXPF` | — | SX850R |

---

## Validation Rules

### VR-01 — Year Cannot Exceed Current Year for Deletion (SX500R, SX520R)

When entering deletion parameters:

```
if d_paar > *year → *in31 = *on
```

The year entered for deletion must not exceed the current system year. Future-year deletions are blocked.

*Effect*: Statistics for the current year and future years cannot be deleted through the deletion screen. This prevents accidental removal of active statistics.

### VR-02 — Authorization Check Blocks Deletion (SX500R, SX520R)

After parameter validation, the sub-program `SX501C` is called with the session user and firm:

```
call 'SX501C'
if w_stop = *on → goto avslutt
```

If `SX501C` sets `w_stop = *on`, the deletion is blocked and the program exits without performing any deletions. The authorization check is mandatory and cannot be bypassed by the user.

*Effect*: Only users authorized in `SX501C` may execute statistics deletions.

### VR-03 — Deletion Scope: Only Records Before Specified Year (SX510R, SX530R)

Deletion programs apply the year as a strict less-than filter:

- **SX510R** (sales statistics): deletes `SSTAPF` records where `sfpaar < d_paar`
- **SX530R** (purchase statistics): deletes `SISTPF` records where `sipaar < d_paar`

Records for the specified year itself are retained. Only records from prior years are removed.

### VR-04 — Cost Price Update Blocked by Multiple Conditions (SX820R)

The cost-price backfill skips a statistics record if any of the following conditions are true:

1. `sfbong = *blank` — no POS bong reference
2. `sfvare = *blank` — no item number
3. `sfkpri = 'F'` — cost price is flagged as fixed; not to be overwritten
4. `sfsumk <> 0` — cost price sum is already populated
5. `%scan('-':sfbong) = 0` — bong number does not contain a hyphen (not a scanner receipt)

If any of these conditions are true, the record is skipped. The `leavesr` opcode is used to exit the processing subroutine.

*Effect*: Cost prices are only backfilled for scanner-originated POS receipts where the cost price has not yet been calculated and is not locked.

### VR-05 — Discount Update Blocked by Missing Data (SX840R)

The discount-amount backfill skips a statistics record if:

1. `sfbong = *blank` — no bong reference (skips, unlocks the record)
2. Item not found in `BODTPF` (POS line register) — record unlocked and skipped
3. `bdbrr1 = 0 and bdbrr2 = 0` — both discount rates are zero — record unlocked and skipped

*Effect*: Discount amounts (sfbrk1, sfbrk2) are only updated when the POS line has at least one non-zero discount rate configured.

### VR-06 — Delivery Date Only Backfilled When Missing (SX831R)

```
if sildat = *loval → process; else skip
```

The purchase statistics delivery date (`SILDAT`) is only updated when the existing value is the low value (i.e., not yet set). Records that already have a delivery date are left unchanged.

### VR-07 — Cobuilder Shadow Table — Order Type Filter (SX850R)

When building the Cobuilder shadow table (`SSTXPF`), orders are filtered:

```
if fokode = 0 and fofsys <> 'INT' → process
```

Only order headers where `fokode = 0` (normal orders) and the originating system is not `'INT'` (internal) are included. Internal or special-type orders are excluded.

Additionally (v6.3a):
```
if vaoakk = 3 and vaosta = 0 → goto neste (skip)
```

Order type records where the accumulation type (`vaoakk`) is 3 and the status (`vaosta`) is 0 are skipped.

---

## Configuration and Authorization Rules

### CA-01 — Hardcoded Start Years

Several backfill programs use hardcoded start years rather than reading a parameter:

- **SX810R**: starts from year 2004 (`w_aarr = 2004`)
- **SX820R**: starts from year 2005 (`sstal9_paar = 2005`)
- **SX840R**: uses current year (`w_aarr = *year`)

These hardcoded values mean the programs will always process from a fixed historical starting point. Adjusting the window requires a source code change.

### CA-02 — Hardcoded Firm in SX840R

```
c                   eval      w_firm = 200
```

SX840R uses a hardcoded firm code of 200 instead of reading `l_firm` from the LDA. The comment line above (`c***`) shows that reading from LDA was previously available but has been disabled. Only firm 200 is processed by this program.

### CA-03 — Cobuilder Shadow Table: First-Run vs. Incremental (SX850R)

SX850R operates in two modes controlled by the parameter `p_first`:

- `p_first = '1'`: full rebuild of the shadow table from all historical orders (`dann_total` subroutine).
- `p_first ≠ '1'`: incremental update — only orders created since the last recorded timestamp (`w_tims` from the most recent `SSTXPF` record) are processed.

The last processed timestamp is read from `SSTXPF` using `*hival` read (most recent record first). If no records exist in `SSTXPF`, `w_tims = *loval` (all orders processed).

---

## Financial and Transactional Rules

### FT-01 — Discount Calculation in SX840R

Discount amounts are calculated as follows:

```
w_totn = sfsumb - sfrab1 - sfrab2          (net amount after existing discounts)
w_tot1 = w_totn × bdbrr1 / 100             (discount 1 amount)
w_totn = w_totn - w_tot1                   (remaining after discount 1)
w_tot2 = w_totn × bdbrr2 / 100             (discount 2 amount, applied to residual)
```

The two discount rates from `BODTPF` (`bdbrr1`, `bdbrr2`) are applied sequentially (cascaded), not additively. The second discount is applied to the amount remaining after the first discount.

Results are written to `SSTAPF.SFBRK1` and `SSTAPF.SFBRK2`.

### FT-02 — Cost Price Calculation via VP700R (SX820R)

Cost price is calculated by calling `VP700R` with:
- Firm, item number, unit, price type, order date, current time, and a "last record" flag

The program returns sale price, purchase price, target price, and cost price. Only the cost price (`w_kopr`) is used:

```
sfsumk = w_kopr × sfanta
```

The total cost sum is cost price per unit multiplied by the sold quantity.

### FT-03 — Product Manager Lookup Cascade (SX810R)

Product manager (`SFPRAN`) is backfilled by cascading through product group levels:

1. Look up sub-group (`VUGR`) using the item's sub-group code → get product manager.
2. If not found, look up main group (`VHGR`) → get product manager.
3. If not found, look up top group (`VOGR`) → get product manager.

The first non-blank product manager found in this hierarchy is written to `SSTAPF.SFPRAN`.

---

## Status and Lifecycle Rules

### SL-01 — Line Number Reset (SX802R)

`SX802R` resets `SFLINE = 0` for all `SSTAPF` records where `sfbinr <> 0` (has a receipt number). This is a maintenance program to be run when line numbers need to be re-sequenced. The operation processes the entire statistics file for the firm; it is not selective by year or period.

### SL-02 — Special Character Conversion is Destructive (SX981R)

`SX981R` uses the RPG `XLATE` operation to replace hex character `x'3F'` with `'È'` in text fields:

- `SSTAPF` text fields: `sdtek1`, `sdtek2`, `sdtek3`
- `FODTPF` text fields: `fdtek1`, `fdtek2`

The conversion is applied to the entire file sequentially. If a character `x'3F'` appears in a field where it is intentional (not a corrupted special character), it will be incorrectly replaced. The operation is irreversible without a file backup.

### SL-03 — Cobuilder Export (SX903R)

`SX903R` reads `SSTAPF` via SQL cursor filtered by:
- `sffirm = w_firm`
- `sfbida >= w_dats` (billing date from last export date)
- `sfeann >= w_eann` (EAN number filter)

Distinct EAN codes are selected and each is processed for Cobuilder XML output. The program reads `JVPKPF` (packaging data), `JVARPF` (item data), `JLEVPF` (supplier data), and `SCOBPF` / `SCOVPF` (shadow tables). If `w_ant > 0` after processing, the XML footer is written and (optionally) an email is sent.

---

## Special Conditions

### SC-01 — SX981R Processes Entire File Without Firm Filter

`SX981R` reads `SODTPF` and `FODTPF` using a simple sequential read (`read`) without a key set. It processes all records in the file regardless of firm. This program must be run with care in multi-firm environments.

### SC-02 — SX850R Array Dimension for Order Numbers

The order number arrays `a_ordr` and `a_suff` each have a dimension of 6000 (v6.38). If more than 6000 orders have been processed since the last shadow table update, the array will overflow. Earlier versions used dimension 6000 with a 6-digit order number; v6.38 extended to 8-digit order numbers.

---

## Subprogram Calls Affecting Logic

| Program | Called Sub-Program | Purpose | Failure Effect |
|---|---|---|---|
| SX500R | `SX501C` | Deletion authorization check | Deletion blocked |
| SX500R | `SX510R` | Execute sales statistics deletion | Deletion not performed |
| SX520R | `SX501C` | Deletion authorization check | Deletion blocked |
| SX520R | `SX530R` | Execute purchase statistics deletion | Deletion not performed |
| SX820R | `VP700R` | Cost price lookup | Cost price remains zero |
| SX850R | `VP905R` | Price/condition data for shadow table | Shadow table data incomplete |

---

## Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| `SSTAPF` | Sales statistics | firm + year + bong + item |
| `SISTPF` | Purchase statistics | firm + year + invoice + item |
| `SSTXPF` | Cobuilder shadow statistics | firm + timestamp + order |
| `SCOBPF` | Cobuilder shadow table | firm + NOBB + item |
| `SCOVPF` | Cobuilder variant table | firm + NOBB |
| `BODTPF` | POS (bong) line register | firm + year + bong + item |
| `SIHEPF` | Historical purchase invoice header | firm + invoice key |
| `VUGR` | Sub-group register | firm + sub-group |
| `VHGR` | Main-group register | firm + main-group |
| `VOGR` | Top-group register | firm + top-group |
| `SODTPF` | Sales order detail (text fields) | firm + order + line |
| `FODTPF` | Purchase order detail (text fields) | firm + order + line |
