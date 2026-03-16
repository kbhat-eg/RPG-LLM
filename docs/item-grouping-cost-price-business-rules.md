# Item Grouping and Cost Price Business Rules

## Introduction

This document describes the business rules enforced by the ASOFAK Item Grouping and Cost Price module (VG prefix programs). The module manages the three-level item group hierarchy (overgroup → main group → subgroup), cost price maintenance with history tracking, cost price updates at goods receipt, and related reporting. All rules described below represent conditions that **block or prevent** an operation from completing. Fields use DB2 TABLE.COLUMN notation where tables are referenced by their physical-file name.

---

## Prerequisites and Master Data Requirements

| Master Data | Table | Key Fields | Used By |
|---|---|---|---|
| Overgroup | VOGRPF (VGOGRPF) | VGFIRM + VGOOGR | VG100R, VG101R, VG102R |
| Main group | VHGRPF | VGFIRM + VGHHGR + VGHOGR | VG101R |
| Subgroup | VUGRPF | VGFIRM + VGUUGR + VGUOGR + VGUHGR | VG102R |
| Account reference | VKHVPF | VKFIRM + VKKONT | VG100R, VG101R |
| Product manager | RA26PF | RAFIRM + RASELG | VG100R, VG101R |
| Sales unit | VENHPF | VEFIRM + VEENH | VG101R |
| Logistics unit | VENHPF | VEFIRM + VEENH | VG101R |
| Price comparison unit | VENHPF | VEFIRM + VEENH | VG101R |
| Item master | VVARPF | VVFIRM + VVVARE | VG200R |
| Cost price register | VGKPST | VKFIRM + VKLAG + VKHKOD + VKVARE | VG200R |
| Order header | LOHEPF | LOFIRM + LONORD | VG800R |
| Order type config | VOTYPF | VOFIRM + VOTYP | VG800R |

---

## Validation Rules

### VG100R — Overgroup Maintenance (VOGRPF)

Save is blocked when any of the following conditions apply:

| Field | Condition | Indicator | Message |
|---|---|---|---|
| Description (VOGRPF.VGOTXT) | Blank | *in31 | Description is mandatory |
| Account reference (VOGRPF.VGOKHV) | Not found in VKHVPF | *in32 | Account reference must exist |
| Product manager (VOGRPF.VGOPRA) | Not found in RA26PF | *in33 | Product manager must exist |

**Delete cascade:** Deleting an overgroup also deletes all associated main groups (VHGRPF) and subgroups (VUGRPF) that reference it. The delete is not blocked by the existence of child records; it cascades. Callers should be aware that all descendant group records are removed.

**Copy option:** Copying an overgroup copies all associated main groups and subgroups to the new overgroup code. If the target overgroup code already exists, the copy is blocked to prevent duplicate key violations.

### VG101R — Main Group Maintenance (VHGRPF)

Save is blocked when any of the following conditions apply:

| Field | Condition | Indicator | Message |
|---|---|---|---|
| Description (VHGRPF.VGHTXT) | Blank | *in31 | Description is mandatory |
| Account reference (VHGRPF.VGHKHV) | Not found in VKHVPF | *in32 | Account reference must exist |
| Sales unit (VHGRPF.VGHENS) | Not found in VENHPF | *in33 | Sales unit must exist |
| Logistics unit (VHGRPF.VGHENL) | Not found in VENHPF | *in34 | Logistics unit must exist |
| Price comparison unit (VHGRPF.VGHENP) | Not found in VENHPF | *in35 | Price comparison unit must exist |
| Product manager (VHGRPF.VGHPRA) | Not found in RA26PF | *in36 | Product manager must exist |

**Delete cascade:** Deleting a main group deletes all associated subgroups (VUGRPF). This cascade is unconditional.

### VG102R — Subgroup Maintenance (VUGRPF)

Subgroup maintenance (VUGRPF keyed by firm + overgroup + main group + subgroup) contains additional fields for dekningsgrad (DG / margin %), average discount, and warehouse allocation. Based on patterns consistent with VG100R and VG101R, save is blocked when:

- Description is blank.
- Any referenced unit code is not found in VENHPF.
- Any referenced account code is not found in VKHVPF.
- Any referenced product manager is not found in RA26PF.

The subgroup is the lowest level of the three-tier hierarchy and has no cascade-delete children within the group structure itself.

### VG200R — Cost Price Maintenance (VGKPST)

Save is blocked under the following conditions:

| Condition | Message | Effect |
|---|---|---|
| Item not found in VVARPF | C1MSG | Cannot maintain cost price for non-existent item |
| Effective date (VGKPST.VGGDAT) equals zero | C2MSG | An effective date is mandatory |
| Cost price (VGKPST.VGCOST) equals zero | C3MSG | A zero cost price is not permitted |

**Update lifecycle:** When an existing cost price record is saved, the following sequence occurs:

1. The current record (VGKPST.VGHKOD ≠ `'H'`) is **deleted**.
2. A history record is **written** with VGKPST.VGHKOD = `'H'`.
3. A new current record is **written** with the updated cost price and effective date.

This three-step process ensures that a full audit trail is maintained in VGKPST. The display subfile (VG200R) shows only current records (VGHKOD ≠ `'H'`). VG201R (cost price history) shows all records including history records.

---

## Configuration and Authorization Rules

### Cost Price Update at Goods Receipt (VG800R)

VG800R is called automatically at goods-receipt time. It reads the order header (LOHEPF) and evaluates the order type configuration (VOTYPF.VGOPPD) to determine whether cost price posting should occur:

| VOTYPF.VGOPPD | Meaning | Required Accumulation Code |
|---|---|---|
| `'M'` | Cost price posting based on goods receipt | LOHEPF.VAOAKK must equal 2 |
| `'F'` | Cost price posting based on invoice | LOHEPF.VAOAKK must equal 3 |

**Blocking conditions in VG800R:**

| Condition | Effect |
|---|---|
| VOTYPF.VGOPPD is not `'M'` or `'F'` | No cost price update is performed |
| LOHEPF.LDLETY = 1 (direct delivery) | Line is skipped; no cost price update for direct-delivery lines |
| Quantity < 0.001 | Line is skipped; zero or near-zero quantities do not trigger cost updates |
| VLTYPF.VALLAG ≠ 1 (line type not a warehouse line) | Line is skipped; only warehouse lines update cost price |
| LOHEPF.VGKRGN ≠ `'J'` | Accounting postings via VG804R are skipped (cost update may still occur) |

---

## Financial and Transactional Rules

### Cost Price Calculation (VG800R → VG702R)

For eligible lines (meeting all VG800R conditions above), VG800R calls VG702R to calculate the new cost price. VG802R then calls VG804R for each line to generate the accounting postings. If the `VGKRGN` flag is not `'J'`, accounting postings are suppressed but the cost price record may still be written.

### History Record Marker (VG200R)

VGKPST.VGHKOD = `'H'` is the sole indicator distinguishing history from current records. The current record always has VGHKOD ≠ `'H'`. Only one current record should exist per firm + warehouse + item combination. The three-step delete/write/write sequence in VG200R enforces this constraint programmatically.

---

## Status and Lifecycle Rules

### Cost Price History Display (VG201R)

VG201R is a read-only display. It shows all VGKPST records for a given item + warehouse, including those with VGHKOD = `'H'`. No updates are permitted. This program provides an audit view of all historical cost price changes.

### Overgroup Copy (VG100R)

When copying an overgroup, all main groups and subgroups are duplicated to the new overgroup code. Any pre-existing records under the target overgroup code will cause a key-collision error during the copy, blocking the copy operation.

---

## Special Conditions

### VG700R — Cost Price Update Orchestrator

Note: The source file VG700R.MBR does not exist in the NXCLOUD/rpgsrc directory. This program is referenced in the module specification but is absent from the source library. Its documented functionality (orchestrating cost price updates) is partially fulfilled by VG800R. Any references to VG700R in runtime job logs or call stacks indicate a missing member that must be investigated before compilation.

### Parameter-Driven Cost Price Setup (VG220R)

Cost price parameter setup is handled by VG220R, which provides configuration for the cost price maintenance process. This program is called from VG200R contexts where global cost price parameters (such as applicable warehouses and calculation methods) must be defined before individual cost prices can be entered.

### Item Group Lookup (VG510R)

VG510R is a read-only lookup/search program that returns the overgroup code (VGOOGR) and description (VGOTXT) to the calling program via return parameters. It does not perform any writes and applies no blocking conditions.

---

## Subprogram Calls

| Calling Program | Called Program | Purpose |
|---|---|---|
| VG800R | VG702R | Calculate new cost price for goods-receipt line |
| VG800R | VG804R | Generate accounting postings for cost price change |
| VG100R | (none) | Direct VOGRPF read/write; cascade delete handled internally |
| VG101R | (none) | Direct VHGRPF read/write |
| VG200R | VG220R | Cost price parameter setup |
| VG600R | VG600C | Batch item group report |

---

## Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| VOGRPF (VGOGRPF) | Item overgroup | VGFIRM + VGOOGR |
| VHGRPF | Item main group | VGFIRM + VGHOGR + VGHHGR |
| VUGRPF | Item subgroup | VGFIRM + VGUOGR + VGUHGR + VGUUGR |
| VGKPST | Cost price register | VKFIRM + VKLAG + VKHKOD + VKVARE |
| VKHVPF | Account references | VKFIRM + VKKONT |
| RA26PF | Product managers / sales persons | RAFIRM + RASELG |
| VENHPF | Unit register | VEFIRM + VEENH |
| VVARPF | Item master | VVFIRM + VVVARE |
| LOHEPF | Purchase order header | LOFIRM + LONORD |
| VOTYPF | Order type configuration | VOFIRM + VOTYP |
| VLTYPF | Line type configuration | VALFIRM + VALTYP |
