# System Administration / Setup — Business Rules

## Introduction

The AA (system administration) module manages the foundational reference data that all other ERP modules depend on: companies (AFIRPF), data libraries / file groups (AFILPF), user accounts (AUSRPF), output queues and printer devices (AENHPF), environments (AMILPF), email routing addresses (AFEMPF), and PDF printer definitions (AFPRPF). Programs in this module are the first line of defence for data integrity: they validate mandatory descriptive fields, prevent duplicate master records, and control which company and data library context a user operates in. The NXKORR override AA022R adds user-maintenance restrictions that depend on role and access code.

---

## Prerequisites / Master Data Requirements

- AFILPF (file group / data library register) must contain at least one entry before users can select a working data library (AA001R) or before any module that reads the data-library context from AUSRPF can operate.
- AFIRPF (company register) must contain at least one entry before users can select a working company (AA002R).
- AMILPF (environment register) must contain at least one entry before AA003R can present an environment selection.
- AUSRPF (user register) must contain an entry for the current IBM i user profile before AA100R can populate the LDA with company, file group, department, and level context. If the user is not in AUSRPF, no LDA defaults are loaded.
- AFPRPF (PDF printer register) entries must be associated with the correct file group (AFPRPF.AFSFIL) for AA500R to present them.
- For AA022R (NXKORR override): AUSRPF must contain user records before user maintenance can operate. MGR-level users require a specific access code to modify users within their own scope (version 8.02 rule).

---

## Validation Rules

### AA008R — Company Register Maintenance (AFIRPF)

- AFIRPF.AAFNAV (company name) blank: indicator *in31 = ON (blocking — company name is required before save).
- Copy operation: if the target firm number already exists in AFIRPF: indicator *in32 = ON (blocking — duplicate company code not permitted).

### AA010R — Output Queue / Device Maintenance (AENHPF)

- AENHPF.AABESK (description) blank: indicator *in31 = ON (blocking — description is required).
- Copy operation: if the target printer/output-queue code already exists in AENHPF: indicator *in32 = ON (blocking — duplicate device code not permitted).

### AA012R — File Group / Data Library Maintenance (AFILPF)

- AFILPF.AABESK (description) blank: indicator *in31 = ON (blocking — description is required).
- Duplicate check: if the file group code already exists when creating a new entry, the duplicate is surfaced in the c1msg field (blocking — duplicate file group code not permitted).
- Option 8 (build data library): calls AA032C to physically create the data library on the IBM i system. If AA032C fails, the library is not created but no rollback of the AFILPF record is performed; manual cleanup may be required.

### AA013R — Email Address Maintenance (AFEMPF)

- No hard blocking conditions; this program writes TO and FROM email addresses for each routine code (AFEMPF.AEMRUT). Missing entries cause email routines to fail silently at runtime in other modules.

### AA100R — User Register Lookup

- No blocking logic. Chains to AUSRPF by username; if found, populates LDA positions for file group, data library, company, department, level, agreement group, and environment. If AUSRPF record is not found, LDA fields retain their default (typically zero/blank) values, which may cause other modules to reject the user.

### AA022R (NXKORR override) — User Maintenance

- Version 8.01: ASP users (identified by a specific flag in AUSRPF) cannot have a startup menu assigned. Attempting to set a startup menu for an ASP user is blocked at save.
- Version 8.02: MGR-level users with a specific access code may change users within their own scope. MGR users without the required access code cannot modify user records outside their scope.

### AA001R / AA002R / AA003R — Selection Subfiles

These programs are purely navigational (select a file group, company, or environment). They update AUSRPF with the user's last selection. No blocking conditions beyond standard file-not-found indicators on CHAIN operations.

---

## Configuration and Authorization Rules

- **Company number** (AFIRPF.AAFIRM): the primary scope key for all business data. Once a company record is created, its number cannot be changed through normal maintenance; the AA400R company-change utility must be used, which performs a destructive bulk update across AFIRPF, ANOTPF, ANUMPF, AKIDPF.
- **File group** (AFILPF): determines which data library (AFILPF.AABIB) the user's session operates against. The file group code and library name are stored in AUSRPF (AUSRPF.ABSFIL, AUSRPF.ABSBIB) after selection via AA001R.
- **User level** (AUSRPF.DSLEVL): controls access rights within modules. AA022R enforces that MGR-level users can only administer users within their own scope unless they hold the required access code.
- **Output queues** (AENHPF): each device/output-queue record is associated with a company. Reports and print jobs route through the queue defined in AENHPF. A missing or misconfigured output queue causes print jobs to fail.
- **PDF printers** (AFPRPF): filtered by file group (AFPRPF.AFSFIL = current file group). Only PDF printers associated with the active file group are presented to users in AA500R.
- **Email routing** (AFEMPF): keyed by routine code (AFEMPF.AEMRUT). If the TO or FROM address is missing for a routine, email functions in that routine will fail at runtime.
- **Environment** (AMILPF): controls whether the user's session operates in production, test, or another environment. Selected via AA003R and returned in p_milj parameter.

---

## Financial / Transactional Rules

Not applicable. The AA module is a pure administration and setup module; it contains no financial transaction logic. However, the master data maintained here is a hard prerequisite for all financial processing:
- A missing or invalid AFIRPF company record prevents financial postings from being scoped to the correct ledger.
- A missing AUSRPF user record means the user's LDA is not populated, causing company-scoped lookups in financial modules to fail.
- Missing AFILPF entries mean no data library is available, making the entire system inoperable for that configuration.

---

## Status and Lifecycle Rules

| Status / Condition | File | Field | Meaning | Effect |
|---|---|---|---|---|
| Company name blank | AFIRPF | AAFNAV | Description not entered | *in31=ON, save blocked |
| Duplicate company | AFIRPF | AAFIRM | Company code already exists | *in32=ON on copy, blocked |
| Device description blank | AENHPF | AABESK | Description not entered | *in31=ON, save blocked |
| Duplicate device | AENHPF | Device code | Code already exists | *in32=ON on copy, blocked |
| File group description blank | AFILPF | AABESK | Description not entered | *in31=ON, save blocked |
| Duplicate file group | AFILPF | File group code | Code already exists | Error in c1msg, blocked |
| ASP user — startup menu | AUSRPF | ASP flag | ASP user cannot have startup menu | Save blocked (AA022R v8.01) |
| MGR without access code | AUSRPF | DSLEVL, access code | Insufficient scope | Cannot modify out-of-scope users (AA022R v8.02) |

---

## Special Conditions (Program-Specific)

### AA012R — Build Data Library (Option 8)

When the user selects option 8 on an AFILPF record, AA012R calls AA032C (a CL program) to issue CRTLIB and related commands on the IBM i system. This is the only program in the AA module that has a direct system-level side effect. If AA032C fails (e.g., library already exists or insufficient authority), no rollback occurs for the AFILPF database record. Manual reconciliation between the AFILPF register and the actual library list is required in case of failure.

### AA100R — LDA Population

AA100R is typically called at job startup or when the user switches company/file group. It reads AUSRPF and writes the following LDA positions:

| LDA Position | Field | Source (AUSRPF) |
|---|---|---|
| 931–933 | File group (DSSFIL) | AUSRPF.ABSFIL |
| (library) | Library name (DSSBIB) | AUSRPF.ABSBIB |
| 944–946 | Company number (DSSFIR) | AUSRPF.ABSFIR |
| 947–950 | Department (DSSAVD) | AUSRPF.ABSAVD |
| 951–980 | Company name (DSFNAV) | AFIRPF.AAFNAV (joined) |
| Various | File group lib, company, dept (DSBFIL, DSBFIR, DSBAVD) | AUSRPF backup values |
| Various | Level (DSLEVL), agreement (DSLAGR), environment (DSMILJ) | AUSRPF |

If the AUSRPF chain fails, all of these remain at default values.

### AA400R — Company Change

AA400R performs a mass firm-number update across four tables: AFIRPF, ANOTPF, ANUMPF, AKIDPF. There are no blocking conditions. This program is intended for use only when duplicating a company's configuration to a new company number as part of a system-cloning or migration procedure. Incorrect execution will corrupt all company-scoped administration records.

---

## Subprogram Calls Affecting Logic

| Calling Program | Called Program | Purpose | Blocking Effect |
|---|---|---|---|
| AA001R | (AUSRPF update) | Save selected file group to user record | None — informational update |
| AA002R | (AUSRPF update) | Save selected company to user record | None — informational update |
| AA003R | (AMILPF read) | Return selected environment | None — returns p_milj |
| AA005R | (display) | Show warning/error window (A1WIN) | None — display only |
| AA007R | (display) | Show confirmation window (M1WIN) | F12 sets p_in12=1; caller decides |
| AA012R | AA032C | Build data library (CL) | Failure leaves AFILPF orphaned |
| AA100R | (AUSRPF chain) | Populate LDA from user record | Not found = blank LDA |
| AA500R | (AFPRPF read) | Select PDF printer filtered by file group | No selection = blank return |

---

## Reference Tables

| Table (Physical File) | Key Fields | Relevant Fields | Role in Module |
|---|---|---|---|
| AFIRPF | Firm | AAFNAV (name), AAFIRM (firm nr) | Company register; AAFNAV required |
| AFILPF | File group code | AABIB (library), AABESK (description) | Data library register; AABESK required |
| AUSRPF | Username | ABSFIL, ABSBIB, ABSFIR, ABSAVD, DSLEVL, DSLAGR, DSMILJ | User register; provides LDA context |
| AENHPF | Firm, Device code | AABESK (description) | Output queue/printer register; AABESK required |
| AMILPF | Environment code | Environment description | Environment register |
| AFEMPF | Routine code (AEMRUT) | TO address, FROM address | Email routing; missing = runtime failure |
| AFPRPF | File group (AFSFIL), Printer code | Printer description | PDF printer register; filtered by active file group |
| ANOTPF | Firm | Notes | Company notes; updated by AA400R |
| ANUMPF | Firm | Number series | Number series for company; updated by AA400R |
| AKIDPF | Firm | KID/reference data | KID reference data; updated by AA400R |
