# Store and Retail Export (POS) Business Rules

## Introduction

This document describes the business rules enforced by the ASOFAK Store/Retail Export module (BU prefix programs). The module exports master data, prices, and transactional reference data to a POS (PC-kasse / cash register) system via a set of staging tables. All rules described below represent conditions that **block or prevent** a record from being included in the export, or that prevent the export process from executing. Fields use DB2 TABLE.COLUMN notation where tables are referenced by their physical-file name.

---

## Prerequisites and Master Data Requirements

| Master Data | Table | Key Fields | Used By |
|---|---|---|---|
| POS store configuration | BSTSPF | BAFIRM + BATERM | BU900R, BU980R |
| Firm master | (LDA / system) | l_firm | BU715R |
| Delivery types | VLTPPF | VYFIRM + VYLETY | BU710R |
| Item types | VVTYPF | VAFIRM + VAVTYP | BU711R |
| Item groups (overgroup/main/subgroup) | VOGRPF / VHGRPF / VUGRPF | VGFIRM + group keys | BU720R |
| Item master | VVARPF | VVFIRM + VVVARE | BU725R, BU720R |
| Item units | VVENPF | VEFIRM + VEVARE + VEENH | BU725R |
| Item prices | VVPRPF | VPFIRM + VPVARE + VPLDOR + VPPRGR | BU725R |
| Campaign prices | VTILPF | VTFIRM + VTVARE | BU728R |
| EAN codes | VEANPF | VEAFIRM + VEAVARE | BU725R |
| Customer master | RKUNPF | RKFIRM + RKKUND | BU730R |
| Customer project postal codes | APOSPF | APFIRM + APPOSN | BU730R |
| Memo configuration | AFPSPF (via CO402R) | FIRM | BU730R |
| Order number series | ANUMPF (via AS100R) | FIRM + series | BU980R |
| Existing orders | FOHEPF | FOFIRM + FONUMM | BU980R |

---

## Validation Rules

### BU730R — Customer Export Blocking Conditions

Customers are exported to BOKUST and BOKPST. Two categories of customers are **blocked from export**:

| Condition | Field | Effect |
|---|---|---|
| Customer is blocked (active block) | RKUNPF.RKSPRK ≠ `' '` (not blank) | Customer is excluded from POS export |
| Customer is passive (inactive) | RKUNPF.RKPASS ≠ `' '` (not blank) | Customer is excluded from POS export |

**Exception:** The cash customer defined in BSTSPF.BAAKUN is always exported regardless of the RKSPRK or RKPASS flags. This ensures the POS anonymous-sale customer is always available.

### BU720R — Item Group Export Blocking Condition

An item group level (overgroup, main group, or subgroup) is only written to BOVGST if at least one active item exists in VVARPF under that group key combination. Groups with no associated items are silently excluded from the export.

### BU980R — POS Order Number Generation Blocking Condition

BU980R generates POS order numbers by calling AS100R in a loop. After each call, the generated number is checked against FOHEPF to ensure it does not already exist:

- If the number already exists in FOHEPF, AS100R is called again until a unique number is found.
- Order type selection: if the order total sum ≥ 0, BSTSPF.BAAOCD (normal order type) is used; if the sum < 0, BSTSPF.BAAOCK (credit order type) is used.
- VA752R is called to determine whether the order type should be overridden for the number series.

A POS order cannot be created if the number series is exhausted (AS100R fails to return a valid number).

---

## Configuration and Authorization Rules

### Export Parameter Conversion (BU900R)

BU900R is a parameter conversion wrapper. It accepts dates and times in YYYYMMDD and HHMMSS format, converts them to ISO format, and calls BU700C with:
- Library name
- POS terminal code
- Firm code
- ISO date
- ISO time

BU700C will not execute if the date/time conversion fails. This is the entry point for scheduled or batch POS export jobs.

### Price Group Resolution (BU920R, BU725R)

Before calculating item prices for POS, BU725R and BU920R resolve the price group through VL712R using firm + warehouse + department. If the price group cannot be resolved, price calculation falls through to the standard (no price group) price retrieval path.

### Delta Export Logic (BU725R, BU730R)

If a date parameter is provided to the export:
- BU725R also includes items whose unit records (VVENPF), price records (VVPRPF), or EAN records (VEANPF) have been changed since the given date. The change date is checked against the record's last-update date field.
- BU730R includes customers whose RKUNPF record has been updated since the given date (checked via RKUNPF.RKEDAT / RKUNPF.RKETIM).

If no date parameter is provided, a full export is performed.

---

## Financial and Transactional Rules

### Price Retrieval Hierarchy (BU725R)

For each item and unit, BU725R retrieves prices from VVPRPF using the following fallback sequence:

1. With price group: warehouse supplier (via VL710R) → main supplier → VPLDOR = 0
2. Without price group: warehouse supplier → main supplier → VPLDOR = 0

If a price is found at step 1, step 2 is not attempted. If no price is found at either level, the item is exported with a zero price. This does not block the export but the item will appear with no price in the POS system.

### VAT Code Resolution (BU725R)

The VAT code for each item is retrieved from VMVAPF using the item's VAT group. If VMVAPF returns no record, the item is exported with the default VAT code.

### Campaign Price Export (BU728R)

BU728R exports campaign prices from VTILPF to the POS staging file. Campaign prices are exported as-is; no date-window filtering is applied during the export. The POS system applies the date window at point-of-sale using the VTFDAT/VTTDAT fields.

### Memo Balance Configuration (BU730R)

BU730R calls CO402R to read the UTVIDET_MEMOSALDO configuration record from AFPSPF. This record contains:
- Number of days for memo balance calculation
- Credit limit extension percentage

These values are written to the BOKUST customer export record. If CO402R cannot find the configuration record, zero values are written for these fields.

---

## Status and Lifecycle Rules

### Deleted Items Export (BU729R)

BU729R exports a list of items that have been deleted since a given date. This allows the POS system to remove obsolete items from its local database. The export is based on a deletion log; items are included if their deletion date is on or after the export date parameter.

### Customer Note Export (BU731R)

Customer notes are exported by BU731R. Notes are included regardless of note type. No filtering on note status or date is applied unless a delta date parameter is provided.

### Sequential Export Orchestration (BU700R)

BU700R calls all sub-exporters in a fixed sequence. If any sub-exporter fails (program call error), the sequence halts at that point. Subsequent sub-exporters are not called. The sequence is:

1. BU710R — delivery types → BOLTST
2. BU711R — item types → BOVTST
3. BU712R — price codes
4. BU715R — firm
5. BU716R — departments
6. BU717R — warehouses
7. BU718R — sellers
8. BU720R — item groups → BOVGST
9. BU721R — module free text
10. BU725R — item info → BOVAST / BOVEST / BOVPST / BOEAST
11. BU726R — item list
12. BU728R — campaign prices
13. BU729R — deleted items
14. BU730R — customer info → BOKUST / BOKPST
15. BU731R — customer notes

---

## Special Conditions

### Standard Item Number from SHOP (BU725R, v6.3d)

From version 6.3d, standard item numbers derived from SHOP (BSTSPF.BASNR) cannot be used as item numbers in the export. If an item's number matches the SHOP standard item number, it is excluded from the item export. This prevents the POS system from registering the internal SHOP placeholder as a saleable item.

### Item Type and Expiry Date (BU725R via VL710R)

For each item, BU725R calls VL710R to retrieve:
- Item type (varetype) for warehouse context
- Expiry date management flag

Items with expiry date management are exported with the relevant flag set. Items where VL710R returns a non-standard item type are exported with that type, which may affect POS discount and pricing logic.

### Deposit (Depositum) Flag (BU725R)

The deposit flag on exported price records informs the POS system whether a deposit amount applies to the item. The deposit amount and associated rules are read from the price record in VVPRPF. Items without a deposit flag are exported with a zero deposit amount.

---

## Subprogram Calls

| Calling Program | Called Program | Purpose |
|---|---|---|
| BU700R | BU710R–BU731R | Sequential export of all POS data sets |
| BU725R | VL710R | Retrieve item type and supplier for warehouse |
| BU725R | VL711R | Retrieve price group for item |
| BU920R | VL712R | Retrieve price group from firm + warehouse + dept |
| BU920R | VL710R | Retrieve item type and supplier |
| BU920R | VP900R | Calculate sale price, cost price, discounts, campaign, VAT |
| BU730R | CO402R | Read memo balance configuration from AFPSPF |
| BU900R | BU700C | Batch export execution after parameter conversion |
| BU980R | VA752R | Determine order type for number series |
| BU980R | AS100R | Generate POS order number |

---

## Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| BSTSPF | POS store configuration | BAFIRM + BATERM |
| BOLTST | POS delivery type staging | — |
| BOVTST | POS item type staging | — |
| BOVGST | POS item group staging | — |
| BOVAST | POS item master staging | — |
| BOVEST | POS item unit staging | — |
| BOVPST | POS item price staging | — |
| BOEAST | POS EAN code staging | — |
| BOKUST | POS customer staging | — |
| BOKPST | POS customer project staging | — |
| VVARPF | Item master | VVFIRM + VVVARE |
| VVENPF | Item unit register | VEFIRM + VEVARE + VEENH |
| VVPRPF | Item price register | VPFIRM + VPVARE + VPLDOR + VPPRGR + VPENHE + VPLETY + VPPDAT |
| VTILPF | Campaign prices | VTFIRM + VTVARE |
| VEANPF | EAN/barcode register | VEAFIRM + VEAVARE |
| RKUNPF | Customer master | RKFIRM + RKKUND |
| VLTPPF | Delivery types | VYFIRM + VYLETY |
| VVTYPF | Item types | VAFIRM + VAVTYP |
| VOGRPF | Item overgroup | VGFIRM + VGOOGR |
| VHGRPF | Item main group | VGFIRM + VGHHGR |
| VUGRPF | Item subgroup | VGFIRM + VGUUGR |
| VMVAPF | VAT code register | VMFIRM + VMVGRP |
| FOHEPF | Order header | FOFIRM + FONUMM |
| ANUMPF | Number series | ANFIRM + series |
| AFPSPF | System parameters | FIRM + key |
| APOSPF | Postal code register | APFIRM + APPOSN |
