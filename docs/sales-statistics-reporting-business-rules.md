# Sales Statistics / Reporting — Business Rules

## Introduction

The SF (sales statistics) module accumulates, queries, and reports on transactional sales and purchasing statistics within the ASSTAT subsystem. Statistical records are written to SSTAPF (sales statistics) when orders are invoiced or store receipts are processed. Query programs (SF500R, SF160R/SF561R) let users filter by year, customer, customer-project, item, and date. Report programs (SF600R, SF611R) extract from work files populated by parameter programs and produce printed or exported output. Utility programs maintain data integrity: SF750R recalculates unit quantities when the statistical reporting unit is changed, and SF400R rewrites the firm number across all statistics tables when a company is cloned. All blocking conditions in this module are input-validation guards on parameter screens; there are no financial posting controls.

---

## Prerequisites / Master Data Requirements

- SSTAPF (sales statistics register) must contain records for the selected firm, year, and customer combination before any query program returns results. Empty statistics produce an empty subfile — not an error.
- RKUNPF (customer register) must contain the selected customer number. SF160R (*in33=ON) and SF500R (*in34=ON) block submission if the customer is not found.
- FKPRPF (customer-project register) must contain the selected customer-project combination if a project filter is applied. SF160R (*in34=ON) and SF500R (*in36=ON) block submission if the project is not found.
- VVARPF (item register) and VVASPF (deleted-item register) are checked by SF500R when an item number filter is applied. A missing item record produces a warning text in the item description field but does not block submission (since version R3.31).
- SKPMPF, SKBUPF, SKNIPF, SKAKPF (customer statistics tables) and SLPMPF, SLBUPF, SLNIPF, SLAKPF, SISTPF (supplier statistics tables) must exist for SF400R's company-change operation to be meaningful.
- VVENPF (item unit conversion register) must contain conversion factors for all units involved in SF750R unit-recalculation. If a conversion factor is missing or zero, recalculation is skipped for that item.

---

## Validation Rules

### SF160R (SF560R) — Sales Statistics Registration Parameter Screen

All conditions set b_feil = *on and prevent the call to SF161R:

| Condition | Indicator | Blocking Condition |
|---|---|---|
| b1paar = 0 (year not entered) | *in31 = ON | Year is required |
| b1bida entered AND invalid date format | *in32 = ON | Date entered is not a valid DMY date |
| b1vkun = 0 (no customer) | *in33 = ON | Customer is required (program defaults to p_vkun if blank) |
| b1vkun not found in RKUNPF | *in33 = ON | Customer does not exist |
| b1kpro entered AND not found in FKPRPF for this customer | *in34 = ON | Customer-project does not exist |

### SF500R — Sales Statistics Query Parameter Screen

All conditions set b_feil = *on and prevent the call to SF501R or SF502R:

| Condition | Indicator | Blocking Condition |
|---|---|---|
| b1paar = 0 (year not entered) | *in31 = ON | Year is required |
| b1bida entered AND invalid date format | *in32 = ON | Date entered is not a valid DMY date |
| b1vkun = 0 AND b1vare = 0 (neither customer nor item) | *in33 = ON | Customer or item number is required |
| b1vkun entered AND not found in RKUNPF | *in34 = ON | Customer does not exist |
| b1vkun = 0 AND b1kpro ≠ 0 (project without customer) | *in35 = ON | Project requires a customer |
| b1kpro entered AND not found in FKPRPF | *in36 = ON | Customer-project does not exist |
| b1vare entered AND not found in VVARPF OR VVASPF | Warning text only (non-blocking since R3.31) | Item not in item register (informational) |

### SF510R — Display Sold Item Detail

- Chains to SSTAPF by key (firm, year, receipt-nr, line, date). If not found (*in60 = ON): `goto avslutt` — program terminates without displaying anything.

### SF600R — Statistical Transaction Extraction

- Reads SKPWPF (parameter work file), SSTAPF (statistics), RKUNPF, VVARPF, VVENPF.
- No interactive blocking; this program is called from a CL driver after parameters are validated.
- Parameters are read from SKPWPF which is populated by SF610R from SKPMPF. An empty parameter work file produces no output lines.

### SF610R — Copy Parameters to Work File

- Reads SKPMPF (parameter register) for the given firm and type (p_type), and writes copies to SKPWPF (work file).
- No blocking validation. If SKPMPF is empty for the given type, SKPWPF is left empty and downstream report programs produce no output.

### SF611R — Statistics Report Print

- Input-primary file is SKWWPF (statistics work file). No interactive blocking.
- Looks up RKUNPF, RLEVPF, VVARPF, VVASL1, and various code registers (RA06PF–RA12PF) for descriptive text. Missing register entries result in blank description fields on the report.

### SF650R — License Usage Print Parameter Screen

- If date-from (b1fdat) entered AND fails DMY date validation (*in36 = ON): loops back to screen.
- If date-to (b1tdat) entered AND fails DMY date validation (*in37 = ON): loops back to screen.
- If w_fdat > w_tdat (from-date after to-date): indicator *in38 = ON → loops back to screen (from-date must not be after to-date).
- On valid input: writes LDA fields (l_ruti, l_chgv) and calls AP600R (printer selection) followed by SF652C.

### SF750R — Unit Recalculation on Statistics Records

- Chains to VVENPF for the item and new unit. If not found (*in90 = ON) OR VVENPF.VEOMRE = 0 (conversion factor is zero): `goto avslutt` — no recalculation is performed for this item.
- For each SSTAPF record with a different unit than the new unit: attempts to find the old unit's conversion factor in VVENPF. If found and non-zero, recalculates SSTAPF.SFANTA (quantity) and updates the record.
- If SSTAPF record itself is not found during the update chain (*in91 = ON): skips that specific record.

---

## Configuration and Authorization Rules

- **Year parameter** (b1paar): mandatory for all query and registration programs. Statistics records are keyed by year; a zero year cannot retrieve any records.
- **Customer vs. item constraint** (SF500R): at least one of customer number (b1vkun) or item number (b1vare) must be provided. This prevents unscoped queries that would return the entire statistics table.
- **Customer-project dependency**: a project filter (b1kpro) always requires a customer filter (b1vkun). You cannot filter by project without specifying the owning customer.
- **Parameter register** (SKPMPF): defines the statistical breakdown dimensions (main group, sub-group, item group, customer category, discount category, district, salesperson, country, etc.) for the SF600R/SF611R report programs. These must be configured in SKPMPF before report runs produce meaningful grouped output.
- **Statistics work file** (SKPWPF): a transient work file used to pass parameters from SF610R to SF600R and SF611R. It is rebuilt on each run. Multiple concurrent report runs for the same firm/type combination may overwrite each other's work files.
- **Unit conversion register** (VVENPF): VVENPF.VEOMRE (conversion factor) must be non-zero for SF750R to apply recalculation. A zero factor means the unit conversion is undefined and the statistics records remain with the old unit and quantity.

---

## Financial / Transactional Rules

- SF750R modifies SSTAPF.SFANTA (quantity) in place. This is an irreversible recalculation; once quantities are converted to the new unit, the original unit-specific quantities are lost. The old unit (SFENHO) is overwritten with the new unit (p_enhe).
- SF400R updates firm numbers across ten statistics tables in a sequential full-table scan. This is a destructive bulk operation intended for company-cloning scenarios. No rollback is available.
- The statistics module does not write to GL (general ledger) or AP/AR. It is a read-mostly analytical store. However, SF160R/SF561R can be called from registration workflows to add new statistics records from order data.

---

## Status and Lifecycle Rules

| Status / Condition | File | Field | Meaning | Effect |
|---|---|---|---|---|
| Year blank | Screen | b1paar | Year not entered | *in31=ON, blocked |
| Invalid date | Screen | b1bida | Non-valid DMY date | *in32=ON, blocked |
| Customer required | Screen | b1vkun | Customer zero | *in33=ON, blocked (SF160R) / *in33=ON (SF500R needs customer or item) |
| Customer not found | RKUNPF | — | No matching record | *in33=ON (SF160R) / *in34=ON (SF500R), blocked |
| Project without customer | Screen | b1kpro / b1vkun | Project entered, no customer | *in35=ON (SF500R), blocked |
| Project not found | FKPRPF | — | No matching record | *in34=ON (SF160R) / *in36=ON (SF500R), blocked |
| Item not in register | VVARPF / VVASPF | — | Item not found in either table | Warning text only; non-blocking (R3.31+) |
| Statistics record not found | SSTAPF | — | No record for key | SF510R aborts; report programs produce empty output |
| Unit conversion factor zero | VVENPF | VEOMRE | Undefined conversion | SF750R skips recalculation for this item |
| Date range invalid | Screen | b1fdat / b1tdat | From-date after to-date | *in38=ON (SF650R), loops back |

---

## Special Conditions (Program-Specific)

### SF500R vs SF160R — Item-Only Query Path

SF500R supports an item-only query (b1vare ≠ 0, b1vkun = 0). In this case, SF502R is called instead of SF501R. This path was not available in the original SF160R design, which always required a customer. The item-only path bypasses the customer-existence check.

### SF501R — Excel Export and Archiving

SF501R (version 6.32+) supports export to Excel via a CGI/CGIDEV2 binding. LDA flags l_exlp (export to Excel) and l_arch (archive lookup) control this behaviour. Export failures are non-blocking for the screen display but produce no Excel file. Version 7.01 adds lookup against VOTYPF (order type register) for archiving.

### SF611R — Multiple Grouping Dimensions

SF611R reads up to 8 different code registers (RA06PF through RA12PF) to resolve group codes to text. Missing code register entries produce blank text fields on the printed report. The grouping dimensions are controlled by the SKPWPF parameter records; each combination of sptype and spkode activates a different set of grouping lookups.

---

## Subprogram Calls Affecting Logic

| Calling Program | Called Program | Purpose | Blocking Effect |
|---|---|---|---|
| SF160R | SF161R | Execute registration from order data | Called only after validation passes |
| SF160R | RK500R | Customer lookup (F4) | Populates b1knav; non-blocking |
| SF160R | FL500R | Customer-project lookup (F4) | Populates b1pnav; non-blocking |
| SF500R | SF501R | Query sales by customer | Called when b1vare=0 |
| SF500R | SF502R | Query sales by item | Called when b1vare≠0 |
| SF500R | RK500R | Customer lookup (F4) | Populates b1knav; non-blocking |
| SF500R | FL500R | Customer-project lookup (F4) | Populates b1pnav; non-blocking |
| SF500R | VV500R | Item lookup (F4) | Populates b1tek1; non-blocking |
| SF610R | (SKPWPF write) | Populate parameter work file | Empty SKPMPF = empty work file |
| SF650R | AP600R | Printer selection | Non-blocking |
| SF650R | SF652C | Execute license usage print | Called after AP600R |
| SF750R | (SSTAPF update) | Recalculate quantities | Skipped if VVENPF missing or factor=0 |

---

## Reference Tables

| Table (Physical File) | Key Fields | Relevant Fields | Role in Module |
|---|---|---|---|
| SSTAPF (sstal1..sstal8) | Firm, Year, Receipt nr, Line, Date | SFVKUN, SFKPRO, SFVARE, SFANTA, SFENHE, SFENHO, SFFIRM | Sales statistics primary store |
| RKUNPF (rkunl1) | Firm, Customer nr | RKNAVN | Customer register; required for customer validation |
| FKPRPF (fkprl1) | Firm, Customer nr, Project nr | FLNAVN | Customer-project register; required for project validation |
| VVARPF (vvarl1) | Firm, Item nr | VVTEK1 | Item register; checked in SF500R item filter |
| VVASPF (vvasl1) | Firm, Item nr | VVTEK1 | Deleted-item register; fallback if item not in VVARPF |
| VVENPF (vvenl1) | Firm, Item nr, Unit | VEOMRE (conversion factor) | Item unit conversion; VEOMRE=0 blocks SF750R recalculation |
| SKPMPF (skpml1) | Firm, Type | Various parameter fields | Statistics parameter register; source for SF610R |
| SKPWPF | Firm, Type, Code | Various | Parameter work file; populated by SF610R, read by SF600R/SF611R |
| SKWWPF | Firm, Sort fields | Statistics totals | Statistics output work file; primary-input for SF611R |
| SKBUPF | Firm | Customer budget | Customer budget statistics; updated by SF400R |
| SKNIPF | Firm | Accumulation levels | Customer accumulation-level register; updated by SF400R |
| SKAKPF | Firm | Accumulation register | Customer accumulation statistics; updated by SF400R |
| SLPMPF / SLBUPF / SLNIPF / SLAKPF | Firm | Supplier statistics tables | Supplier statistics; updated by SF400R |
| SISTPF | Firm | Purchase statistics | Purchasing statistics; updated by SF400R |
| RA06PF–RA12PF | Firm, Code | Code descriptions | Code registers used for group text resolution in SF611R |
