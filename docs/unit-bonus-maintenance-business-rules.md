# Unit and Bonus Maintenance Business Rules

## Introduction

This document describes the business rules enforced by the ASOFAK Unit and Bonus Maintenance module (VA prefix programs). The module manages units of measure (VENHPF), account references/cost-centre assignments (VKHVPF), line types (VLTYPF), order types (VOTYPF), price codes (VPKOPF), bonus templates (EMBHST via VA500R), and related reference tables. All rules described below represent conditions that **block or prevent** an operation from completing. Fields use DB2 TABLE.COLUMN notation where tables are referenced by their physical-file name.

---

## Prerequisites and Master Data Requirements

| Master Data | Table | Key Fields | Used By |
|---|---|---|---|
| Unit register | VENHPF | VEFIRM + VEENH | VA110R |
| Account reference | VKHVPF | VAKFIR + VAKKOD + VAKLTY + VAKOTY | VA120R |
| Line type | VLTYPF | VALFIRM + VALTYP | VA140R |
| Order type | VOTYPF | VAOFIRM + VAOTYP | VA150R |
| Internal order type mapping | VOT1PF | VA1FIRM + VA1TYP | VA150R (v6.20+) |
| Price code | VPKOPF | VAPFIRM + VAPKOD | VA160R |
| Bonus templates | EMBHST | EMHFIRM + EMHBOT + EMHMNR | VA500R |
| Bonus type | VBOTPF | VATFIRM + VATBOT | VA500R |

---

## Validation Rules

### VA110R — Unit Maintenance (VENHPF)

VA110R maintains unit-of-measure records in VENHPF. The subfile is keyed by firm + unit code (VEENH) with secondary access by description text. Standard operations: create (option from F key), change (option 2), copy (option 3), delete (option 4), display (option 5).

**Blocking conditions for save:**

| Field | Condition | Effect |
|---|---|---|
| Unit code (VENHPF.VAENH) | Blank at creation | Save blocked; unit code is mandatory |
| Unit description (VENHPF.VAETXT) | Blank | Save blocked; description is mandatory |
| Duplicate unit code | Code already exists in VENHPF for the firm | Creation blocked (key collision) |

The copy operation duplicates the unit record to a new unit code. If the target code already exists, the copy is blocked.

Tracking fields VAEOUS (last update user) and VAEOEUS are updated on every save.

### VA120R — Account Reference Maintenance (VKHVPF)

VA120R maintains account reference records in VKHVPF keyed by firm + account code (VAKKOD) + delivery type (VAKLTY, from v5.20) + order type (VAKOTY).

**Blocking conditions for save:**

| Field | Condition | Effect |
|---|---|---|
| Account code (VKHVPF.VAKKOD) | Blank at creation | Save blocked |
| Description (VKHVPF.VAKTXT) | Blank | Save blocked |
| Order type (VKHVPF.VAKOTY) | Required field if VAKLTY is set | Must be specified when delivery type is set |

From version 5.20, the key was extended to include delivery type (VAKLTY), allowing different account references for the same account code when the delivery type differs. This means account references are now uniquely identified by firm + code + delivery type + order type.

The copy operation (option 3) duplicates to a new code + delivery type + order type combination. Copy is blocked if the target key already exists.

### VA140R — Line Type Maintenance (VLTYPF)

VA140R maintains line type records keyed by firm + line type code (VALTYP).

**Blocking conditions for save:**

| Field | Condition | Effect |
|---|---|---|
| Line type code (VLTYPF.VALTYP) | Blank at creation | Save blocked |
| Description (VLTYPF.VALTXT) | Blank | Save blocked |
| Duplicate line type code | Already exists in VLTYPF for the firm | Creation blocked |

Standard operations: create, change (option 2), copy (option 3), delete (option 4), display (option 5). No external master data lookups are required for line type records beyond the firm code validation.

### VA150R — Order Type Maintenance (VOTYPF)

VA150R maintains order type records in VOTYPF keyed by firm + order type code (VAOTYP), with secondary access by description text.

**Blocking conditions for save:**

| Field | Condition | Effect |
|---|---|---|
| Order type code (VOTYPF.VAOTYP) | Blank at creation | Save blocked |
| Description (VOTYPF.VAOTXT) | Blank | Save blocked |
| Duplicate order type | Already exists in VOTYPF for the firm | Creation blocked |

**Order type configuration fields** (not blocking but functionally significant):

| Field | Purpose |
|---|---|
| VOTYPF.VAOBIL | Screen format alternative (v5.01) |
| VOTYPF.VAODIS | Processing mode: common / direct / job queue (v5.02) |
| VOTYPF.VAONRS | Order type for number series (v5.32) |
| VOTYPF.VAOMOT | Corresponding order type in other system (v5.40) |
| VOTYPF.VAOINT | Internal order type code (v6.20) |
| VOTYPF.VAOKOT | Internal accounting code (v7.01) |

**Invoice/credit type switching (v8.01):** An additional mapping table (VOT1PF keyed by firm + internal type code VA1TYP) allows the system to switch order types when a transaction changes between invoice and credit note during invoicing. This mapping is maintained in the lower portion of the VA150R screen. A missing VOT1PF record for a given VA1TYP code does not block order processing but may cause the wrong type to be applied during invoicing.

### VA160R — Price Code Maintenance (VPKOPF)

VA160R maintains price code records in VPKOPF keyed by firm + price code (VAPKOD). Operations: change (option 2) and display (option 5) only — no create or delete from the subfile; price codes must be created via the C2BLD screen.

**Blocking conditions for save:**

| Field | Condition | Effect |
|---|---|---|
| Price code (VPKOPF.VAPKOD) | Blank | Save blocked |
| Description (VPKOPF.VAPBSK) | Blank | Save blocked |

---

## Configuration and Authorization Rules

### Order Type Processing Mode (VA150R, VOTYPF.VAODIS)

The processing mode field (VAODIS) controls how orders of this type are processed:
- `'F'` — common (felles) processing queue
- `'D'` — direct processing
- `'J'` — job queue processing

This field determines the routing of orders to picking, invoicing, and other downstream processes. An invalid or blank VAODIS value is not blocked by VA150R but may cause runtime errors in the order processing programs that read this field.

### Account Reference Delivery Type Extension (VA120R, v5.20)

From version 5.20, account references are keyed by a three-part key: account code + delivery type (VAKLTY) + order type (VAKOTY). This allows different general ledger accounts to be posted depending on the delivery type on the order line. If no account reference exists for the specific delivery type, the calling program (typically VG804R or the accounting module) must fall back to the account reference with a blank delivery type. This fallback is handled in the accounting module, not in VA120R.

---

## Financial and Transactional Rules

### Price Code Usage

Price codes in VPKOPF are referenced in the price hierarchy by VP900R (the core price engine). The price code (VAPKOD) controls which pricing tier is applied to an order line. A price code that exists in VPKOPF but has no associated price records in VVPRPF or FPRIPF results in no price being found; the price engine returns a zero price in that case.

### Bonus Template Inquiry (VA500R)

VA500R is a read-only inquiry program that displays bonus templates from EMBHST and associated bonus types from VBOTPF. It is called from customer and order programs to show the bonus agreements applicable to a customer or transaction. Parameters returned to the caller:

| Parameter | Field | Description |
|---|---|---|
| `p_tbot` | EMBHST.EMHBOT | Bonus type code |
| `p_hmnr` | EMBHST.EMHMNR | Template number |
| `p_mbes` | EMBHST.EMHBES | Bonus description from template |
| `p_tbes` | VBOTPF.VATBES | Bonus type description |

VA500R exits via `b_valg_ok = *on` when a selection is confirmed (the caller uses this return to know a valid bonus template was selected). No blocking conditions prevent reading the template list.

---

## Status and Lifecycle Rules

### Unit Deletion (VA110R)

Before deleting a unit code, the system should verify that no active item master records (VVARPF), price records (VVPRPF), or order lines (FODTPF) reference the unit code being deleted. VA110R itself presents a confirmation window (D1WIN) before deletion; however, it does not perform a cross-reference check against all dependent files. The responsibility for ensuring no active references exist lies with the administrator.

### Line Type Deletion (VA140R)

Similarly, line type deletion (option 4) presents a confirmation window. VA140R does not check for order lines referencing the line type being deleted. Deleting a line type still in use by active order lines will cause those lines to reference an invalid type.

### Order Type Deletion (VA150R)

VA150R does not include a delete option in the main subfile. Order types are considered long-lived configuration records. Removal requires direct database intervention.

---

## Special Conditions

### Internal Order Type for Invoicing (VA150R, v8.01)

The VOT1PF extension table (VA1TYP → target order type) is used exclusively during invoicing when an invoice/credit note changes type. For example, a credit note may switch from order type 10 to order type 11 during invoicing. This mapping is transparent during normal order entry and does not affect order line processing.

### Delivery Type as Key Segment (VA120R, v5.20)

The inclusion of VAKLTY (delivery type) as a key segment in VKHVPF means that queries against VKHVPF must supply the delivery type or use a partial-key access path. Programs written before v5.20 that performed simple code-only lookups must be updated to include the delivery type or they will retrieve incorrect or no account references.

---

## Subprogram Calls

| Calling Program | Called Program | Purpose |
|---|---|---|
| VA110R | (none) | Direct VENHPF read/write |
| VA120R | (none) | Direct VKHVPF read/write |
| VA140R | (none) | Direct VLTYPF read/write |
| VA150R | (none) | Direct VOTYPF + VOT1PF read/write |
| VA160R | (none) | Direct VPKOPF read/write |
| VA500R | (none) | Read-only EMBHST + VBOTPF inquiry |

---

## Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| VENHPF | Unit of measure register | VEFIRM + VAENH |
| VKHVPF | Account references | VAKFIR + VAKKOD + VAKLTY + VAKOTY |
| VLTYPF | Line type register | VALFIRM + VALTYP |
| VOTYPF | Order type register | VAOFIRM + VAOTYP |
| VOT1PF | Order type invoice/credit mapping | VA1FIRM + VA1TYP |
| VPKOPF | Price code register | VAPFIRM + VAPKOD |
| EMBHST | Bonus template header | EMHFIRM + EMHBOT + EMHMNR |
| VBOTPF | Bonus type descriptions | VATFIRM + VATBOT |
