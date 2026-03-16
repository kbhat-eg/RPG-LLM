# Special Pricing Business Rules

## Introduction

This document describes the business rules enforced by the ASOFAK Special Pricing module (FP prefix programs). The module manages special price agreements (FPRIPF), customer notes (FPRNPF), label printing configuration, electronic shelf-edge label parameters, and special-price reporting. All rules described below represent conditions that **block or prevent** an operation from completing. Fields use DB2 TABLE.COLUMN notation where tables are referenced by their physical-file name.

---

## Prerequisites and Master Data Requirements

The following master data records must exist before special pricing records can be created or maintained:

| Master Data | Table | Key Fields | Used By |
|---|---|---|---|
| Price group | RA30PF.RAPRGR | RAFIRM + RAPRGR | FP100R, FP600R |
| Discount category | RA06PF.RARABL | RAFIRM + RARABL | FP600R |
| Customer category | (internal code) | — | FP600R |
| Customer | RKUNPF.RKKUND | RKFIRM + RKKUND | FP150R, FP600R |
| Customer project | FKPRPF.FKPRNR | FKFIRM + FKPRNR | FP600R |
| Item | VVARPF.VVVARE | VVFIRM + VVVARE | FP600R, FP200R |
| Warehouse | (validated via RS710R) | FIRM + WAREHOUSE | FP200R, FP260R |
| Electronic label config | AELHPF | FIRM + WAREHOUSE | FP260R |
| Printer setup | VFASPF.LCSKTY | LCFIRM + LCLOC | FP205R |

---

## Validation Rules

### FP150R — Special Price Notes

| Condition | Indicator | Effect |
|---|---|---|
| Customer (`p_kund`) equals zero | — | Save is blocked; record cannot be written to FPRNPF |

The note file FPRNPF is keyed by firm + customer + project + item. A save attempt with no customer assigned is rejected unconditionally.

### FP200R — Label Print Parameters

| Field | Condition That Blocks | Indicator |
|---|---|---|
| Warehouse | Warehouse not found via service program RS710R (returns `p_stat='N'`) | *in30 (field protect, display-only mode triggered) |
| Item type (`VVVTYP`) | Not one of: `'L'` (length goods), `'S'` (special), `' '` (standard) | *in31 |
| Campaign flag | Not one of: `'J'` (yes) or `' '` (blank) | *in32 |
| Price override flag | Not one of: `' '` (none) or `'J'` (yes) | *in33 |

When *in30 is set the entire screen enters display-only mode and no updates are permitted.

### FP250R — Radio Terminal Label Parameters

| Field | Condition That Blocks | Description |
|---|---|---|
| Label type (`b1etyp`) | Value not in the set {0, 1, 2, 3, 4, 5, 6, 7, 8} | Invalid label type; save blocked |
| Price code (`b1etpr`) | Value not in the set {1, 2, 3} | Invalid price code; save blocked |

### FP260R — Electronic Shelf-Edge Label Parameters

| Condition | Indicator | Effect |
|---|---|---|
| Warehouse not found via RS710R (`p_stat='N'`) | *in30 | Screen enters display-only mode; no updates permitted |
| No configuration record exists in AELHPF for the firm + warehouse combination | *in34 | Save blocked; electronic label parameters cannot be saved without a base config |

AELHPF stores the supplier code, file ID, label type, and overwrite flag used by the electronic shelf-edge label system. If this record is absent the program cannot proceed.

### FP600R — Special Price Report Parameters

| Condition | Indicator | Effect |
|---|---|---|
| Price group not found in RA30PF | *in38 + `b_feil=*on` | Report generation blocked |
| Discount category FROM > TO | *in31 | Range invalid; blocked |
| Customer category FROM > TO | *in32 | Range invalid; blocked |
| Customer specified together with discount category or customer category | *in33 | Mutually exclusive; blocked |
| Customer not found in RKUNPF | *in34 | Unknown customer; blocked |
| Project specified without a customer | *in35 | Project requires a customer; blocked |
| Customer project not found in FKPRPF | *in36 | Unknown project; blocked |
| Item FROM > TO | *in37 | Range invalid; blocked |

Any field marked with an error indicator causes FP600C (the report generation batch program) to not be called.

---

## Configuration and Authorization Rules

### Label Program Routing (FP205R)

The label print program is determined by reading VFASPF for the printer type code LCSKTY (values 0–6). The label type VNETYP (values 1–9) is mapped to print programs FP213R through FP221R and stored in LCRUTI. If no matching VFASPF record exists, no print program name is resolved and label printing cannot proceed.

### Electronic Label Provider Configuration (FP260R)

AELHPF provides the following mandatory parameters for electronic shelf-edge label output:
- Supplier code
- File ID for label transfer
- Label type identifier
- Overwrite flag (controls whether existing labels are replaced)

If AELHPF is missing the entire electronic label function is unavailable for that warehouse.

---

## Financial and Transactional Rules

### Special Price Hierarchy in FPRIPF

Special prices in FPRIPF are sorted through nine logical views (fpril1–fpril9) covering, in priority order:

1. Price group (FPPRGR, 2 characters from v8.01)
2. Discount category
3. Customer category
4. Customer
5. Customer project
6. Item
7. Unit
8. Delivery type
9. Sequence number

The price engine (VP900R) applies these in the order listed. A more specific record overrides a more general one.

### Price Group Extension (v8.01)

From version 8.01, price group FPPRGR was extended from 1 to 2 characters. Any special price record with a price group code longer than the original single-character format requires the updated system to be applied. Existing single-character codes remain valid without change.

---

## Status and Lifecycle Rules

### Special Price Inquiry (FP510R)

FP510R is a read-only inquiry program. It reads FPRIPF and FPRNPF (notes) but performs no updates. Filtering is supported by price group, discount category, customer category, customer, customer project, and item. No blocking conditions apply during inquiry; all records are displayed regardless of date or status.

---

## Special Conditions

### Customer Requirement for Notes (FP150R)

The note file FPRNPF requires a valid customer number (`p_kund <> 0`). Notes that are item-level or project-level with no associated customer cannot be saved. This prevents orphan notes that cannot be associated with a billing relationship.

### Warehouse Validation via Service Program

Both FP200R (label parameters) and FP260R (electronic label parameters) call service program RS710R to validate the warehouse. RS710R returns `p_stat='N'` if the warehouse does not exist or is not active. This indirect validation means the warehouse check is centralised and not duplicated in the calling programs.

### Report Parameter Mutual Exclusivity (FP600R)

A customer number cannot be combined with discount-category or customer-category range parameters in the same report request. This restriction (indicator *in33) prevents logically contradictory report selections that would produce empty or misleading output.

---

## Subprogram Calls

| Calling Program | Called Program | Purpose |
|---|---|---|
| FP200R | RS710R | Validate warehouse; returns `p_stat='N'` if invalid |
| FP205R | FP213R–FP221R | Label print execution (selected by label type) |
| FP260R | RS710R | Validate warehouse |
| FP600R | FP600C | Batch report generation (called only if all validations pass) |
| FP510R | (none) | Read-only inquiry; no subprogram calls |

---

## Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| FPRIPF | Special price agreements | FPFIRM + FPPRGR + FPRABL + FPKKAT + FPKUND + FPKPRO + FPVARE + FPENHE + FPLETY + FPSEQN |
| FPRNPF | Special price notes | FPFIRM + FPKUND + FPKPRO + FPVARE |
| RA30PF | Price groups | RAFIRM + RAPRGR |
| RA06PF | Discount categories | RAFIRM + RARABL |
| RKUNPF | Customer master | RKFIRM + RKKUND |
| FKPRPF | Customer projects | FKFIRM + FKPRNR |
| VVARPF | Item master | VVFIRM + VVVARE |
| VFASPF | Printer/location setup | LCFIRM + LCLOC |
| AELHPF | Electronic label parameters | Firm + warehouse |
| RS710R | Warehouse validation service program | — |
