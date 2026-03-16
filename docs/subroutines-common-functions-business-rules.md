# Subroutines / Common Functions — Business Rules

## Introduction

The RS (subroutines) module is a library of shared validation and lookup subprograms called by other ERP modules. Each program receives one or more key values as parameters, performs a keyed READ (CHAIN) against the relevant master-data file, and returns a status code and/or descriptive text to the caller. All programs are bound to the calling job's company number via the Local Data Area (LDA position 944–946). The primary purpose of every program in this module is to block or warn when referenced master data does not exist, is blocked for use, or when a counter has been exhausted. Callers must test the returned status code and halt their own processing accordingly.

---

## Prerequisites / Master Data Requirements

- LDA position 944–946 (packed 3,0) must contain the active company number. All programs in this module use this value as the first key field; without it, lookups fail silently against the wrong (or zero) company.
- The RA4PF counter register must be populated with a valid number range for customer, supplier, and contact-person number allocation (RS001R, RS002R, RS003R).
- Code registers RA01PF, RA02PF, RA03PF, RA05PF, RA07PF, RA10PF, RA20PF must be populated for the active company before their respective validation programs can return a positive result.
- RHOVPF (chart of accounts) must exist for any account that is to be validated by RS740R.
- RKUNPF (customer register) must exist for any customer validated by RS750R.
- RLEVPF (supplier register) must exist for any supplier validated by RS002R.

---

## Validation Rules

### RS001R — Next Available Customer Number

- Chains to RA4PF by company and counter type. If not found: returns p_retk = `'N'` (blocking — counter not configured).
- If the candidate number w_4knr <= RA4PF.RA1HKT (lower bound) OR >= RA4PF.RA1HRS (upper bound): returns p_retk = `'N'` (blocking — number range exhausted).
- After finding a candidate, verifies the number is not already in RKUNPF. Increments until a free number is found or the range is exhausted.

### RS002R — Next Available Supplier Number

- Chains to RA4PF for the supplier counter. If not found: returns p_retk = `'N'`.
- If w_4lnr < RA4PF.RA1HRS: returns p_retk = `'N'` (range exhausted).
- Checks RLEVPF for existing suppliers at each candidate number.

### RS003R — Next Available Contact Person Number

- Chains to RA4PF for the contact-person counter. If not found: returns p_retk = `'N'`.
- Checks RUKPPF for existing contact persons.

### RS701R — Voucher Code Validation (RA01PF)

- CHAIN to RA01PF by company + code. If not found (*in60 = ON): p_stat = `'N'`, p_atxt = `'Koden er ugyldig'` (blocking — code is invalid).

### RS702R — Payment Method Validation (RA02PF)

- CHAIN to RA02PF by company + code. If not found: p_stat = `'N'` (blocking — payment method invalid).

### RS703R — Payment Terms Validation (RA03PF)

- CHAIN to RA03PF by company + code. If not found: p_stat = `'N'` (blocking — payment terms invalid).

### RS705R — VAT Code Validation (RA05PF)

- CHAIN to RA05PF by company + code. If not found: p_stat = `'N'` (blocking — VAT code invalid).

### RS710R — Warehouse Code Validation (RA10PF)

- CHAIN to RA10PF by company + code. If not found: p_stat = `'N'` (blocking — warehouse code invalid).

### RS720R — Account Group Validation (RA20PF)

- CHAIN to RA20PF by company + code. If not found: p_stat = `'N'` (blocking — account group invalid).

### RS740R — GL Account Validation (RHOVPF)

Returns a single-character status code in p_stat. All non-blank returns are blocking:

| Condition | p_stat | Meaning |
|---|---|---|
| p_avde (department) populated AND not found in RA07PF | `'A'` | Department not in code register |
| p_avde populated AND p_kont = 0 (no account) | `'M'` | Department filled without account number |
| p_spes populated AND p_kont = 0 (no account) | `'K'` | Specification filled without account number |
| RHOVPF not found for key firm+kont+spes+avde (all three attempts) | `'N'` | Account number is invalid |
| RHOVPF found AND RHOVPF.RHSPRK = `'J'` | `'S'` | Account is blocked for use |
| (all checks pass) | blank | Account is valid |

The lookup logic is hierarchical: first tries exact match (kont+spes+avde), then kont+spes with avde=0, then kont alone (spes=0, avde=0). If all three CHAIN operations fail, p_stat = `'N'` is returned.

### RS750R — Customer Number Validation (RKUNPF)

| Condition | p_retk | Meaning |
|---|---|---|
| RKUNPF not found for company + customer number (*in60 = ON) | `'N'` | Customer does not exist; p_navn = `'KUNDENUMMER ER UGYLDIG'` |
| RKUNPF.RKSPRK = `'J'` | `'S'` | Customer is blocked for use; p_navn = `'KUNDEN ER SPERRET'` |
| (all checks pass) | blank | Customer is valid; p_navn = RKUNPF.RKNAVN |

---

## Configuration and Authorization Rules

- Company scope: all RS programs derive the company number from LDA position 944–946 at initialization. The company number is the first key field in every CHAIN operation. Programs with the V5.00 notation in their change log previously accepted the company number as a parameter (parameter `p_firm`); from V5.00 onward, this parameter is commented out (marked `S500 c***`) and the LDA value is used instead.
- Counter registers (RA4PF) define valid number ranges. These must be pre-configured by system administration before number-allocation programs (RS001R, RS002R, RS003R) can succeed.
- The department code register (RA07PF) is required by RS740R for department validation. A missing RA07PF entry for a given department causes p_stat = `'A'`, which the caller must treat as a blocking condition.
- The error texts returned by RS750R are stored as compile-time data array entries (`ctdata`) embedded in the source member, not in a database table. Changing these texts requires recompilation of RS750R.

---

## Financial / Transactional Rules

- RS740R enforces the three-part GL account key: account number (RHKONT) + specification (RHSPES) + department (RHAVDE). Partial keys (specification or department without account) are rejected with distinct status codes (`'K'` and `'M'` respectively). This prevents misposting to partially-defined account combinations.
- RHOVPF.RHSPRK = `'J'` (account blocked) is an absolute block. Even if the account exists and is otherwise valid, no transaction may be posted to a blocked account. The status `'S'` must be trapped by all callers.
- RKUNPF.RKSPRK = `'J'` (customer blocked) prevents any new business transaction referencing that customer. Programs that create orders, invoices, or receipts must call RS750R and refuse to continue if p_retk = `'S'`.

---

## Status and Lifecycle Rules

| Status Code | Program | Return Field | Meaning |
|---|---|---|---|
| `'N'` | RS001R, RS002R, RS003R | p_retk | Counter not configured or number range exhausted |
| `'N'` | RS701R–RS720R | p_stat | Code not found in respective register |
| `'N'` | RS740R | p_stat | GL account combination not found |
| `'A'` | RS740R | p_stat | Department not in RA07PF code register |
| `'M'` | RS740R | p_stat | Department entered without account number |
| `'K'` | RS740R | p_stat | Specification entered without account number |
| `'S'` | RS740R | p_stat | GL account is blocked (RHOVPF.RHSPRK = `'J'`) |
| `'N'` | RS750R | p_retk | Customer not found |
| `'S'` | RS750R | p_retk | Customer blocked (RKUNPF.RKSPRK = `'J'`) |
| blank | All | p_stat / p_retk | Validation passed; record exists and is usable |

---

## Special Conditions (Program-Specific)

### RS740R — Three-Level Account Key Fallback

RS740R attempts up to three CHAIN operations against RHOVPF when the full key (account + specification + department) is not found. It first zeroes the department and retries, then zeroes both specification and department and retries. This means a transaction with a specific department can still be validated if the account exists without a department dimension, and similarly for specifications. Only when all three attempts fail is p_stat set to `'N'`. This three-level fallback is the core business logic that makes the chart-of-accounts flexible across single-dimension and multi-dimension account setups.

### V5.00 Parameter Change (Firm Number)

Programs RS740R and RS750R (and likely others in the RS series) have a commented-out parameter `p_firm` (marked `S500 c***`) following the V5.00 change. This parameter was removed from the external interface and replaced by the LDA-sourced firm number. Any integration or calling program written against the pre-V5.00 interface must be updated to omit the firm parameter.

---

## Subprogram Calls Affecting Logic

| Calling Program | Called Program | Purpose | Blocking Effect |
|---|---|---|---|
| Any order/voucher program | RS701R | Validate voucher code | p_stat=`'N'` blocks transaction |
| Any AP/AR program | RS702R | Validate payment method | p_stat=`'N'` blocks transaction |
| Any AP/AR program | RS703R | Validate payment terms | p_stat=`'N'` blocks transaction |
| Any tax-bearing transaction | RS705R | Validate VAT code | p_stat=`'N'` blocks transaction |
| Any inventory transaction | RS710R | Validate warehouse code | p_stat=`'N'` blocks transaction |
| Any GL posting program | RS740R | Validate GL account | p_stat non-blank blocks posting |
| Any customer-facing program | RS750R | Validate customer number | p_retk non-blank blocks transaction |
| Master-data creation programs | RS001R / RS002R / RS003R | Allocate next number | p_retk=`'N'` blocks record creation |

---

## Reference Tables

| Table (Physical File) | Key Fields | Relevant Fields | Role in Module |
|---|---|---|---|
| RA01PF | Firm, Voucher code | Code description | Voucher code register (RS701R) |
| RA02PF | Firm, Payment method code | Code description | Payment method register (RS702R) |
| RA03PF | Firm, Payment terms code | Terms definition | Payment terms register (RS703R) |
| RA05PF | Firm, VAT code | VAT rate, description | VAT code register (RS705R) |
| RA07PF | Firm, Department code (RAGKOD) | Department description | Department code register (RS740R) |
| RA10PF | Firm, Warehouse code | Warehouse description | Warehouse register (RS710R) |
| RA20PF | Firm, Account group code | Group description | Account group register (RS720R) |
| RA4PF | Firm, Counter type | RA1HKT (lower), RA1HRS (upper) | Number-range counter (RS001R/002R/003R) |
| RHOVPF | Firm, RHKONT, RHSPES, RHAVDE | RHSPRK, RHTEXT | Chart of accounts; RHSPRK=`'J'` blocks account |
| RKUNPF | Firm, Customer nr | RKSPRK, RKNAVN | Customer register; RKSPRK=`'J'` blocks customer |
| RLEVPF | Firm, Supplier nr | Supplier fields | Supplier register (RS002R duplicate check) |
| RUKPPF | Firm, Contact person nr | Contact person fields | Contact person register (RS003R duplicate check) |
