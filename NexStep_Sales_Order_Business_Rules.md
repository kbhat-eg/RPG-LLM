# NexStep ERP — Sales Order Business Rules
**Source programs analyzed:** FO100R (order entry/maintenance), FO500R (order inquiry), FO614R (picking/invoicing/backorder), FO730R (price recalculation)
**Schema source:** FFRF.MBR, RFRF.MBR (reference field definitions), DDS source files
**Date of analysis:** 2026-02-20

---

## 1. Introduction

### Purpose of This Document

This document describes every business rule enforced when creating, modifying, and processing sales orders and quotes in the NexStep ERP system. It is drawn directly from the RPG IV source programs that run on IBM i (AS/400). Every rule described here corresponds to actual code logic verified in the source.

The document is intended for:
- New developers learning how orders work
- Business analysts mapping ERP behaviour to process flows
- Support staff diagnosing why an order was blocked or rejected
- System administrators configuring order types, warehouses, and user permissions

### How to Use This Document

Fields in the database follow a **two-letter prefix convention** that tells you which table they belong to:

| Prefix | Table | Meaning |
|--------|-------|---------|
| `SO` | SOHEPF | Sales order / invoice **header** field |
| `SD` | SODTPF | Sales order / invoice **line** field |
| `RK` | RKUNPF | **Customer master** field |
| `VA` | VOTYPF, VLTPPF, VLTYPF, etc. | Various **configuration** register fields |
| `FL` | FKPRPF | **Customer project** field |

Throughout this document, field names are written as `TABLE.FIELD` where useful for precision (for example `RKUNPF.RKKRES` = the credit-blocked flag in the customer master).

### Key Terminology

**Order vs. Quote (Tilbud)**
An *order* is a binding sales document that accumulates against the customer's credit balance and triggers inventory movement. A *quote* (Norwegian: *tilbud*) is a non-binding price offer that does not affect the credit limit. The distinction is made entirely by the order type: if `VOTYPF.VAOAKK = 0`, the document is a quote; any value of 1 or higher makes it a real order.

**Memo Balance (Memosaldo)**
The *memo balance* is the running total of all uninvoiced orders placed by a customer. It is tracked in `RKUNPF.RKMEMS`. When checking whether a customer has exceeded their credit limit, the system adds the memo balance (multiplied by a VAT factor) to the customer's existing open-invoice balance `RKUNPF.RKSALD`. This gives a full picture of total customer exposure before granting more credit.

**Resting (Backorder)**
When an order is partially fulfilled and the remaining items are deferred to a later shipment, the unfulfilled portion is called a *rest* (Norwegian: *resting*). The system creates a new order record with an incremented suffix (`SOSUFF`) to hold the backordered quantities. The field `SOHEPF.SOREST` is set on the new backorder header to mark it as a resting order.

**Customer Project (Kundeprosjekt)**
A *customer project* (Norwegian: *kundeprosjekt*) stored in FKPRPF lets a delivery address, contact, payment terms, invoice type, and discount category be associated with a customer's specific building project or job site. When an order references a customer project via `SOHEPF.SOKPRO`, many header fields are automatically overridden from the project record rather than from the plain customer master.

---

## 2. Database Schema Overview

### SOHEPF — Sales Order / Invoice Header

SOHEPF holds one record per order (or invoice). An open order has `SOBINR = 0`; once invoiced, the invoice number is written into `SOBINR`. The same physical record represents both the open order and the archived invoice.

| Field | Type | Norwegian Label | English Description |
|-------|------|-----------------|---------------------|
| SOFIRM | 3,0 packed | Firmanr | Company number |
| SOAARR | 4,0 packed | År | Fiscal year (YYYY) |
| SOBINR | 8,0 packed | Fakturanummer | Invoice number; 0 = open order, > 0 = invoiced |
| SONUMM | 8,0 packed | Ordrenr | Order number (system-assigned from order type series) |
| SOSUFF | 2,0 packed | Suffix | Backorder suffix; 0 = original, increments per backorder cycle |
| SOOTYP | 2 char | Ordretype | Order type code — looked up in VOTYPF |
| SOKUND | 6,0 packed | Kundenummer | Invoice customer number — looked up in RKUNPF |
| SOLKUN | 6,0 packed | Vare-adresse | Goods delivery customer number (separate ship-to customer) |
| SOKPRO | 6,0 packed | Kundeprosjekt | Customer project number — looked up in FKPRPF |
| SOBDAT | Date (ISO) | Ordredato | Order date |
| SOLDAT | Date (ISO) | Leveringsdato | Requested delivery date |
| SOFKDA | Date (ISO) | Fakturadato | Invoice date (written by FO614R during invoicing) |
| SOSELG | 4,0 packed | Selgernr | Salesperson number — looked up in RA09PF |
| SOAVDE | 4,0 packed | Avdeling | Sales department (override from header) |
| SOLAGE | 2,0 packed | Lager | Warehouse number — looked up in RA07PF |
| SOLETY | 1,0 packed | Leveringstype | Delivery type code — looked up in VLTPPF |
| SOLMAT | 2,0 packed | Leveringsmåte | Shipping method code — looked up in RA17PF |
| SOBETB | 2,0 packed | Bet.betingelser | Payment terms code — looked up in payment terms register |
| SORABP | 5,2 packed | Rabatt i % | Order-level discount percentage applied to all lines |
| SOFSUM | 11,2 packed | Fakturasum | Total invoice amount including VAT |
| SORKAT | 3,0 packed | Rabatt-kategori | Discount category used for price lookup |
| SOVALK | 3 char | Valuta | Currency code (default from customer) |
| SOMKOD | 1,0 packed | Mva-kode | VAT code override |
| SOVREF | 30 char | Vår ref. | Our reference (salesperson or internal) |
| SODREF | 30 char | Deres ref. | Customer's reference |
| SOREKV | 20 char | Kund.rekv.nr. | Customer requisition / purchase order number |
| SONAVN | 30 char | Navn | Delivery name (may differ from invoice customer name) |
| SOADR1 | 30 char | Adresse 1 | Delivery address line 1 |
| SOADR2 | 30 char | Adresse 2 | Delivery address line 2 |
| SOSTED | 30 char | Poststed | Delivery postal city |
| SOKREF | 8,0 packed | Kreditt-ref. | Credit reference number (linked credit note) |
| SOFFDA | Date (ISO) | Forfallsdato | Due date (set during invoicing from payment terms) |
| SODEPO | 11,2 packed | Krav til Depositum | Required deposit amount |
| SODEPB | 11,2 packed | Betalt Depositum | Deposit amount actually paid |
| SODEPT | 11,2 packed | Trukket Depositum | Deposit applied against the invoice |
| SOFOSA | 11,2 packed | Fordelt red.salg | Settlement discount amount applied (excl. VAT) |
| SOFOSI | 11,2 packed | F.red.salgImva | Settlement discount amount applied (incl. VAT) |
| SOINF1 | 12 char | Informasjon-1 | Hold/info code (references VINFL1 register) |
| SOINF2 | 40 char | Informasjon-2 | Hold/info description text |
| SOEDIH | 8,0 packed | Edi-hode | EDI header reference number |
| SONREF | 15 char | Eksternt referansenr | External system reference number |
| SOFATY | 2,0 packed | Fakturatype | Invoice type code |
| SOREST | 1,0 packed | Restordre | Backorder flag: 0 = original order, 1 = backorder |
| SOLAOK | 1,0 packed | Leveringsadresse OK | Delivery address confirmed: 0 = not confirmed, 1 = confirmed |
| SOWSID | 10 char | Arbeidstasjon | Workstation ID of creating terminal |
| SOUSER | 10 char | Bruker | User ID who created the order |
| SOOUSR | 10 char | Opprettet av | User who created (audit field) |
| SOEDAT | Date (ISO) | Endringsdato | Date of last modification |
| SOETIM | Time (ISO) | Endringstidspunkt | Time of last modification |
| SOEUSR | 10 char | Endret av | User who last modified the record |

### SODTPF — Sales Order / Invoice Lines

SODTPF holds one record per order line. Key fields:

| Field | Type | Norwegian Label | English Description |
|-------|------|-----------------|---------------------|
| SDFIRM | 3,0 packed | Firmanr | Company number |
| SDAARR | 4,0 packed | År | Fiscal year |
| SDBINR | 8,0 packed | Fakturanummer | Invoice number (mirrors SOBINR) |
| SDNUMM | 8,0 packed | Ordrenr | Order number |
| SDSUFF | 2,0 packed | Suffix | Backorder suffix |
| SDLINE | 4,0 packed | Linjenr | Line number within the order |
| SDVARE | 15 char | Varenr | Item / product number — looked up in VVARPF |
| SDANTA | 11,3 packed | Antall bestilt | Quantity (3 decimal places; positive = sale, negative = return) |
| SDSAPR | 9,2 packed | Salgspris | Sales price per unit |
| SDKOPR | 9,2 packed | Kost-pris | Cost price per unit |
| SDRAB1 | 5,2 packed | Linje-rabatt 1 | Line discount 1 (%) |
| SDRAB2 | 5,2 packed | Linje-rabatt 2 | Line discount 2 (%) |
| SDSALG | 11,2 packed | Salg | Line net sales amount after all discounts |
| SDLTYP | 2 char | Linje-type | Line type code — looked up in VLTYPF |
| SDLAGE | 2,0 packed | Lager | Line-level warehouse (can override header) |
| SDAVDE | 4,0 packed | Avdeling | Line-level department (can override header) |
| SDBRR1 | 5,2 packed | Bruksrettrabatt 1 | Bracket / usage-right discount 1 (%) |
| SDBRR2 | 5,2 packed | Bruksrettrabatt 2 | Bracket / usage-right discount 2 (%) |
| SDPASL | 5,2 packed | Påslag | Markup percentage |
| SDKORD | 1,0 packed | Kode ordrerab. | Order discount override: 0 = use header SORABP, other = suppress |
| SDKPRI | 1 char | Kode, nettopris | Net price code (F = fixed, do not recalculate) |
| SDLVRE | 1,0 packed | Lokalt vreg | Special/procured item flag |
| SDEDIH | 8,0 packed | Edi-hode | EDI header reference |
| SDEDIL | 6,0 packed | Edi-linje | EDI line reference |
| SDWSID | 10 char | Arbeidstasjon | Workstation ID |
| SDUSER | 10 char | Bruker-ID | User who created the line |
| SDEUSR | 10 char | Endret av | User who last updated the line |

### Related Lookup Tables

| Table | Key | Purpose |
|-------|-----|---------|
| RKUNPF | RKFIRM + RKKUND | Customer master — name, address, credit limit, payment terms, salesperson |
| VOTYPF | VAOFIR + VAOTYP | Order type definitions — accumulator type, system flags, number series |
| VLTPPF | VALFIR + VALLETY | Delivery types — controls purchase order integration |
| VLTYPF | VALFIR + VALTYP | Line types — distinguishes normal, text, and special lines |
| FKPRPF | FLFIRM + FLKUND + FLKPRO | Customer projects — delivery address, payment terms, discount, invoice type overrides |
| VVARPF | VVFIRM + VVVARE | Item master — description, unit of measure, price type, VAT |
| RKMEPF | — | Customer memo balance accumulation table |
| RA07PF | — | Warehouse register |
| RA09PF | — | Salesperson register |
| RA17PF | — | Shipping method register |
| FUSRPF | FBFIRM + FBUSER | User permission register — warehouse/department restrictions, packing slip authority |
| FAAOPF | FAAFIR | System configuration parameters — default order types, quote days, delivery requirements |

---

## 3. Order Creation Prerequisites

### Required Master Data

Before any order can be created, the following master data must be set up in the system:

**Customer (RKUNPF):** The invoice customer number entered in `SOKUND` must exist in RKUNPF, keyed by `RKFIRM + RKKUND`. Without a valid customer, the order screen will reject the entry.

**Order Type (VOTYPF):** The order type entered in `SOOTYP` must exist in VOTYPF and must have `VOTYPF.VAOSYS = 0`. The VAOSYS flag of 1 is reserved for system-generated order types (for example, internal picking orders, credit notes). Users can only work with order types where VAOSYS = 0.

**Warehouse (RA07PF):** The warehouse code in `SOLAGE` must exist in RA07PF. Without a valid warehouse, the system cannot route the order to a fulfilment location.

**Salesperson (RA09PF):** A non-zero salesperson code is required in `SOSELG`. The code must exist in RA09PF.

**Payment Terms:** The payment terms code in `SOBETB` must exist in the payment terms register. It defaults from `RKUNPF.RKBETB` when the customer is selected.

### User Permission Requirements

FO100R and FO614R check the user permission register FUSRPF at several points:

**Warehouse restriction:** Users may be restricted to specific warehouses via FUSRPF. If a user's permission record limits them to certain warehouses, they cannot create or modify orders for other warehouses.

**Department restriction:** Similarly, users can be restricted to specific departments. The order's `SOAVDE` must fall within the user's allowed departments.

**Packing slip authority:** FO614R checks FUSRPF before allowing a user to print a packing slip or re-print one that has already been issued. Without this authority flag, the user sees an error.

**Invoicing authority:** FO614R has a separate check (`R3.34`) limiting who can run the invoicing process. Only users with the invoicing permission flag set in FUSRPF may trigger final invoice creation.

**Credit override (F18):** Overriding a credit block requires a specific permission. When approved, the user's ID is written to `SOHEPF.SOUSKR` as an audit trail.

---

## 4. Customer Validation Rules

### 4.1 Customer Existence

**What is checked:** When a customer number is entered in `SOKUND` (invoice customer) or `SOLKUN` (goods-delivery customer), the system performs a direct key lookup on RKUNPF using company number + customer number.

**Why this is required:** All downstream validations — credit limit, payment terms, salesperson defaults, delivery address — depend on finding a matching RKUNPF record. An unrecognised customer number cannot receive credit or be properly invoiced.

**What happens:** If the customer is not found, the screen displays an error message and the cursor returns to the customer number field. The order cannot be saved until a valid customer is entered.

### 4.2 Customer Status (Blocked Indicator)

**What is checked:** `RKUNPF.RKSPAV` holds the general customer block indicator. A non-zero value means the customer has been administratively blocked.

**Why this is required:** A blocked customer should not be able to place new orders. This might occur after a customer has been set up incorrectly, is under review, or has been deactivated.

**What happens:** If `RKSPAV <> 0`, the system sets an error indicator and the order cannot proceed. This is a hard block with no F18-style override — the customer record itself must be corrected first.

### 4.3 Cash Customer Restrictions

**What is checked:** Certain customers are designated as "cash customers" using a specific customer type setup. Cash customers are linked to a default order type in FAAOPF (`FAAOPF.FAAKUN` = cash customer number, `FAAOPF.FAANRD` = cash order type).

**Why this is required:** A cash customer has no credit arrangement and should only receive quotes — not orders that accumulate credit exposure or trigger inventory commits.

**What happens:** If a cash customer tries to create an order (as opposed to a quote), the system routes them to the quote order type. They cannot directly create an order with `VAOAKK > 0`.

### 4.4 Invoice Customer vs. Goods Customer

**What is checked:** `RKUNPF.RKFAKU` stores the "invoice customer number" that should be used for billing when the customer is a goods-delivery-only customer. The system checks that `SOKUND` matches `RKUNPF.RKFAKU` when applicable.

**Why this is required:** A warehouse, building site, or branch may have its own customer number in RKUNPF for delivery purposes, but the invoice must go to the parent account. If someone tries to invoice the delivery-address customer directly, the billing would go to the wrong account.

**What happens:** If a mismatch is detected, the system warns the user and may redirect `SOKUND` to the correct invoice customer. Both the invoice customer (`SOKUND`) and the goods-delivery customer (`SOLKUN`) are stored on the order header.

### 4.5 Customer Project Requirements [CONFIGURABLE]

**What is checked:** `RKUNPF.RKKATG` (customer category group) drives whether a customer project is required. When the category group is configured to require a project, the order cannot be saved with `SOKPRO = 0`. When a project number is entered, it is validated against FKPRPF using the key `FLFIRM + FLKUND + FLKPRO`.

**Why this is required:** Some customers — typically large contractors or key accounts — require every order to be tagged to a specific project so that costs, discounts, and invoices can be tracked at project level.

**What happens:** If no project is found in FKPRPF for the entered `SOKPRO`, the system displays an error. If the category forces a project but none is entered, the order is blocked until a project number is supplied.

### 4.6 Internal Customer Validation

**What is checked:** `RKUNPF.RKSPFA` identifies internal customers (for example, another branch of the same company). An internal customer should use an internal order type (one flagged with `VA1INT = 1` in the order type additional register VOT1PF). The program checks that internal order types are only used with internal customers, and regular order types are only used with regular customers.

**Why this is required:** Using a regular customer-facing order type for an internal stock transfer would create incorrect revenue postings and expose internal pricing to standard reports.

**What happens:** If an internal customer (`RKSPFA <> 0`) tries to use a non-internal order type, or vice versa, the system sets error code `'C'` on `w_feil_kund` and returns to the header screen. The user must select the correct order type before proceeding.

---

## 5. Credit Limit Rules

### 5.1 Credit Limit Check

**What is checked:** The available credit is calculated as:

```
Available Credit = (RKUNPF.RKGRNS × 1000) minus (RKUNPF.RKSALD + (RKUNPF.RKMEMS × VAT factor))
```

`RKUNPF.RKGRNS` stores the credit limit in thousands (so a value of 500 means NOK 500,000). `RKUNPF.RKSALD` is the outstanding invoice balance. `RKUNPF.RKMEMS` is the memo balance — the sum of all open uninvoiced orders.

**Example:** Customer 12345 has `RKGRNS = 200` (credit limit NOK 200,000), `RKSALD = 85,000` open invoices, `RKMEMS = 40,000` in open orders, and VAT factor = 1.25. Exposure = 85,000 + (40,000 × 1.25) = 135,000. Available credit = 200,000 − 135,000 = 65,000.

**Why this is required:** The credit check prevents the company from shipping more goods to a customer than they can reasonably be expected to pay for.

### 5.2 Credit Block Status

**What is checked:** `RKUNPF.RKKRES` is the credit block flag: 0 = not blocked, 1 = credit blocked.

**Why this is required:** When the credit department decides to stop a customer's account, setting `RKKRES = 1` immediately prevents all new orders — regardless of whether there is technically available credit — until the block is manually removed.

**What happens:** With `RKKRES = 1`, the system sets `s_sper = 1`. A blocked order shows a visual warning on screen. Quotes (order types with `VAOAKK = 0`) are still allowed because they do not commit credit. Only someone with F18 authority can override the credit block for a real order.

### 5.3 Quote Exception

**What is checked:** When the order type has `VOTYPF.VAOAKK = 0` (accumulator type = 0), the system treats the document as a quote.

**Why this is required:** Quotes are tentative. A customer who is credit-blocked can still receive a price offer — blocking quotes would prevent the sales team from doing any business with the customer while a dispute is resolved.

**What happens:** For order types with `VAOAKK = 0`, the credit block check is bypassed entirely. The memo balance is not accumulated, so the quote has no effect on `RKUNPF.RKMEMS`.

### 5.4 Credit Override Permissions

**What is checked:** Pressing F18 on the order header screen when a credit block warning is displayed triggers an authorisation check in FUSRPF.

**Why this is required:** Some senior users (such as sales managers) should be able to approve an order for a credit-blocked customer without waiting for the credit department to remove the block.

**What happens:** If the user has F18 authority, the system records the approving user's ID in `SOHEPF.SOUSKR` and allows the order to proceed. This creates an audit trail showing who overrode the credit block and when.

### 5.5 Extended Memo Balance [CONFIGURABLE]

**What is checked:** The memo balance calculation can be extended to include future uninvoiced orders with VAT. This is controlled by a configuration option (`u_716` flag). When active, the system multiplies `RKUNPF.RKMEMS` by a VAT factor (`w_mvat`, typically 1.25 for 25% VAT) before adding it to the balance.

**Why this is required:** Without VAT, the memo balance understates the true credit exposure because the customer will owe VAT on top of the net order value. Including VAT gives a more conservative (and accurate) view of exposure.

**What happens:** If the extended calculation is enabled and the resulting balance exceeds the credit limit, the credit warning fires earlier than it would without the VAT factor.

### 5.6 Public Discount Limit (Almenning)

**What is checked:** `RKUNPF.RKGRAL` holds a customer-specific public/cooperative discount limit. If this is zero, the program calls FO106R to retrieve a system-wide default. The limit is stored in the running variable `s_gral` for use during price recalculation.

**Why this is required:** Norwegian building products co-operatives ("almenning") have contractually capped discount levels. Orders that would breach the limit must be flagged during invoicing (checked in FO614R, logged with code 5.60).

**What happens:** During FO614R invoicing, if the accumulated almenning discount exceeds `s_gral`, a warning or error is raised depending on configuration. This is tracked through the bracket discount fields `SDBRR1`/`SDBRR2` on each line.

---

## 6. Order Header Validation

### 6.1 Order Type (SOOTYP)

**What is checked:** `SOOTYP` must exist in VOTYPF (`chain` on VAOFIR + VAOTYP). Additionally, `VOTYPF.VAOSYS` must equal 0 — system-reserved order types cannot be selected by users.

**Cannot change when lines exist:** If the order already has item lines, the order type cannot be changed. Changing the order type could alter the number series, inventory update rules, and invoice layout in ways that would make the existing lines inconsistent.

**What happens when type is missing:** An indicator `*in62` fires and the user sees an error message. The order cannot be saved.

### 6.2 Order Number (SONUMM)

**What is checked:** Order numbers are system-assigned. The program reads the next available number from a status/sequence table specific to the order type. If `VOTYPF.VAONUM` is set, that order type uses its own dedicated number series; otherwise it falls back to the company-wide series.

**Suffix rules for backorders:** When a backorder (resting) is created from order number 12345 suffix 0, the original order becomes 12345/00 and the backorder becomes 12345/01. The system increments the suffix rather than assigning a new order number, keeping all related shipments grouped under the same order number.

**What happens when a conflict occurs:** If the next number in the series is already taken (race condition in a multi-user environment), the program loops and tries the next number until it finds a free slot.

### 6.3 Salesperson (SOSELG)

**What is checked:** `SOSELG` must be non-zero and must exist in RA09PF.

**Default inheritance:** When a customer is selected, if `RKUNPF.RKSLGR <> 0`, the salesperson defaults from the customer master. This can be overridden by a configuration flag (`u_713`). If the user has a salesperson number assigned in their profile, that may also be used as the default.

**What happens when invalid:** Indicator `*in26` fires, the field is highlighted, and the order cannot be saved.

### 6.4 Warehouse (SOLAGE)

**What is checked:** `SOLAGE` must exist in RA07PF. User warehouse restrictions from FUSRPF also apply — a user restricted to warehouse 10 cannot create an order for warehouse 20.

**Changing warehouse with existing lines:** If the warehouse on the header is changed after lines have been entered, FO100R calls FO794R which asks the user whether to update all lines, only lines that match the old warehouse, or none.

**What happens when invalid:** Indicator `*in42` fires and the order cannot be saved.

### 6.5 Department (SOAVDE) [CONFIGURABLE]

**What is checked:** If the department field is enabled (configurable per company), `SOAVDE` must exist in the department register. The department must be consistent with the selected warehouse.

**Why this is required:** Departments are used for GL posting. An order posted to the wrong department would corrupt financial reporting.

**What happens:** If the department/warehouse combination is not in the allowed cross-reference table, an error message appears.

### 6.6 Warehouse–Department Consistency

**What is checked:** A cross-reference table defines which departments are valid for each warehouse. When both `SOLAGE` and `SOAVDE` are entered, the combination is checked against this table.

**Why this is required:** Prevents posting sales to department/warehouse combinations that don't make business sense — for example, posting a hardware store sale to the insulation department of a different branch.

### 6.7 Delivery Type (SOLETY)

**What is checked:** `SOLETY` must exist in VLTPPF. The delivery type controls whether the system creates a purchase order (direct delivery from supplier), uses stock from the warehouse, or does something else.

**Affects purchase order creation:** Certain delivery type values in VLTPPF indicate "direct delivery" — in these cases, FO100R links the order line to a purchase order in FPLOPF. Changing the delivery type on the header after lines have been entered triggers a recalculation of all line delivery types.

### 6.8 Shipping Method (SOLMAT)

**What is checked:** `SOLMAT` must exist in RA17PF. Zero is allowed (meaning "no shipping method specified").

**Transport API integration:** Certain shipping method codes trigger an external transport API call (for example, booking a specific carrier). If the shipping method is changed after such a call has been made, the system attempts to cancel the original booking before creating a new one.

### 6.9 Payment Terms (SOBETB)

**What is checked:** `SOBETB` must exist in the payment terms register. It defaults from `RKUNPF.RKBETB` when the customer is first selected.

**Customer project override:** If a customer project is referenced via `SOKPRO`, the project's payment terms (`FKPRPF.FLBETB`) replace the customer-default terms. This is the mechanism through which specific projects get extended terms or special net arrangements.

### 6.10 Customer References (SOVREF, SODREF, SOREKV) [CONFIGURABLE]

**What is checked:** Three reference fields can each be made mandatory via configuration switches:

- `SOVREF` (Our reference — the salesperson or internal contact): made mandatory by `u_vref = *on`
- `SODREF` (Their reference — the customer's buyer or reference): made mandatory by `u_dref = *on`
- `SOREKV` (Customer requisition number): configurable separately

**Why this is required:** Many customers, especially government and large enterprise accounts, require a valid purchase order reference on the invoice before they will pay. Making these fields mandatory prevents invoices from being issued without the reference.

**What happens when missing:** Indicators `*in56` (SOVREF) and `*in57` (SODREF) fire and the order cannot be saved until the required reference is supplied.

### 6.11 Delivery Address

**Defaults:** When an order is created, the delivery address fields (`SONAVN`, `SOADR1`, `SOADR2`, `SOSTED`) are populated from the goods customer (`SOLKUN`) if different from the invoice customer, or from the customer project if `SOKPRO` is set.

**LAOK confirmation:** `SOHEPF.SOLAOK` is a delivery address confirmation flag. When set to 0, the delivery address has not been confirmed. Certain configurations require this to be confirmed (set to 1) before picking or invoicing can proceed.

---

## 7. Order Line Validation

### 7.1 Line Type (SDLTYP)

**What is checked:** `SDLTYP` is looked up in VLTYPF. The key field `VLTYPF.VALKTX` determines how the line is treated:

| VALKTX | Treatment |
|--------|-----------|
| 0 | Normal item line — full price and inventory validation |
| Non-zero | Text/comment line — no item lookup, no pricing, no inventory |

FO730R skips any line where `VALKTX <> 0` during price recalculation.

**D-type items:** `VVARPF.VVVTYP = 'D'` marks an item as manually priced. Lines referencing a D-type item are skipped during automatic price recalculation in FO730R (`goto no_pris`). The price must be entered manually.

### 7.2 Item Number (SDVARE)

**What is checked:** `SDVARE` must exist in VVARPF, keyed by `VVFIRM + VVVARE`. The item must have a status that permits sales (non-discontinued, available for the current order type).

**What happens when not found:** The line cannot be saved. An error message indicates the item number is unknown.

### 7.3 Quantity (SDANTA)

**Sign rules:** A positive quantity represents a standard sale. A negative quantity represents a return — but only when the order's line type in VLTYPF has `VLTYPF.VALKRE = '-'`, which signals that this line type allows credit/return quantities. FO730R applies the sign: `if valkre = '-' then s_anta = s_anta * -1` before calculating totals.

**Decimal places:** Quantity is stored as 11 digits with 3 decimal places (11S 3). This supports products sold by fractional units such as metres of timber or litres of paint.

### 7.4 Unit of Measure (SDENH1)

**What is checked:** The unit of measure must be valid for the item as defined in VVARPF. If the wrong unit is entered, pricing may be incorrect (price is per metre, but quantity was entered as pieces).

### 7.5 Line-Level Warehouse (SDLAGE)

**Defaults from header:** When a line is created, `SDLAGE` defaults from the order header `SOLAGE`. This can be overridden on individual lines to allow an order to be fulfilled from multiple warehouses.

**Validation:** A line-level warehouse override must also exist in RA07PF and must pass the same user permission check as the header warehouse.

### 7.6 Line-Level Department (SDAVDE)

**Defaults from header:** `SDAVDE` defaults from `SOAVDE`. Individual lines can be overridden to post to a different department, enabling mixed-department orders for customers who buy from multiple product areas.

**Consistency:** The line-level department must be consistent with the line-level warehouse per the same cross-reference table used for the header.

---

## 8. Price and Discount Rules

### 8.1 Price Determination via VP900R

All price lookup is delegated to the external service program `VP900R`. The calling program (FO730R via subroutine `hent_pris`) passes an input record (`hirec`) containing:

- Company, discount category, customer, customer project, item, delivery type, unit of measure, order date, order number, price-priority code, markup percentage, internal customer flag, price group

VP900R applies the following hierarchy to determine the sales price:

1. **Customer-specific price** — a price negotiated specifically for this customer and item
2. **Customer price group price** — a price applying to all customers in the same price group
3. **Standard list price** — the base price from the price master
4. **Manual override** — for items with `VVARPF.VVVTYP = 'D'` (type D items), no automatic lookup is performed; the price must be entered manually

If VP900R returns `hosapr > 0.01` and the line's price code `SDKPRI <> 'F'` (fixed), the returned price replaces the current price on the line.

### 8.2 Discount Hierarchy

Discounts are applied as a cascade — each successive discount applies to the running *net* after the previous discount, not to the original gross. The FO730R calculation is:

```
Gross line value:
  w_sums = SDSAPR × SDANTA

Order-level discount (from SOHEPF.SORABP):
  if VLTYPF.VVKORD = 0:
    w_sumr += w_sums × SORABP / 100

Line Discount 1 (SDRAB1):
  if SDRAB1 <> 0:
    w_sumr += (w_sums - w_sumr) × SDRAB1 / 100

Line Discount 2 (SDRAB2):
  if SDRAB2 <> 0:
    w_sumr += (w_sums - w_sumr) × SDRAB2 / 100

Bracket Discount 1 (SDBRR1):
  if SDBRR1 <> 0:
    w_sumr += (w_sums - w_sumr) × SDBRR1 / 100

Bracket Discount 2 (SDBRR2):
  if SDBRR2 <> 0:
    w_sumr += (w_sums - w_sumr) × SDBRR2 / 100

Return fee (if AKTIVER_RETURGEBYR configured):
  w_retb = (w_sums - w_sumr) × FOD2PF.FD2SAT / 100
  w_sumr += w_retb

Net line sales amount:
  SDSALG = w_sums - w_sumr
```

**Important:** The order discount (`SORABP`) is only applied to a line if `SDKORD = 0`. If `SDKORD <> 0`, the line is exempt from the header-level order discount.

### 8.3 Discount Category Override via FL710R

Before calling VP900R, FO730R calls `FL710R` with the customer number and customer project. FL710R may return a different discount category (`w_rkat`) than the one stored in `SOHEPF.SORKAT`.

If the returned category differs from the stored header category, the customer number and project number are zeroed out (`hikund = 0`, `hikpro = 0`) in the VP900R input record. This forces VP900R to look up prices purely based on the overridden discount category, ignoring any customer-specific pricing that might be attached to the original customer.

### 8.4 Price Group (SDPRGR / FDPRGR)

Before each line is priced, FO730R calls `VL712R` passing `FDLAGE` (warehouse) and `FDAVDE` (department) to obtain a 2-character price group code. This price group (`w_prgr`, stored in `SDPRGR`) is passed into the VP900R price lookup as `hiprgr`. Different warehouses or departments may thus charge different prices for the same item.

### 8.5 Return Fee [CONFIGURABLE]

**What is checked:** FO730R calls `CO402R` with key `'AKTIVER_RETURGEBYR'`. If the configuration value returns `'1'` (enabled), the system reads the return fee percentage (`FOD2PF.FD2SAT`) for each line and includes it in the discount cascade.

**Why this is required:** Some industries apply a restocking or handling fee when goods are returned. This fee is stored as a percentage in a line side-table (FOD2PF) and flows into the net discount total.

**What happens:** The return fee amount is added to `w_sumr` (the total discount/deduction), effectively reducing the net sales amount further, and is tracked separately in `SDBRRA` for reporting.

### 8.6 Manual Pricing (Type D Items)

**What is checked:** `VVARPF.VVVTYP = 'D'` marks an item as requiring manual pricing.

**Why this is required:** Certain products — custom or project-specific items — do not have a standard price list entry. The sales rep must enter the agreed price directly.

**What happens:** In FO730R, if the item is found in VVARPF and `VVVTYP = 'D'`, the program jumps to tag `no_pris` and skips all VP900R price lookups. Whatever price is already on the line (`FDSAPR`) stays unchanged. The totals are still recalculated using the manually entered price.

### 8.7 Public Discount Limit Tracking

During invoicing (FO614R), almenning (public/co-op) discount amounts are accumulated separately (tracked as `w_suma` in FO730R → stored in `SOHEPF.SOBRRA`). When the accumulated almenning discount exceeds the customer's limit (`s_gral`, from `RKUNPF.RKGRAL`), a warning or block is raised depending on configuration.

---

## 9. Date Validation Rules

### 9.1 Order Date (SOBDAT)

**What is checked:** The order date must be non-zero and must be a valid calendar date (the RPG `test(d)` operation validates it). An invalid date — such as 30 February or a corrupted numeric value — sets indicator `*in32` and blocks the order.

**Accounting period:** The order date is compared against the current open accounting period read from the local data area (`l_peri`). An order date that falls within a closed period is rejected because it would post to a locked GL period.

### 9.2 Delivery Date (SOLDAT)

**What is checked:** If a delivery date is entered, it must be a valid calendar date and must be on or after the order date (`SOLDAT >= SOBDAT`). An earlier delivery date than the order date is logically impossible and sets indicator `*in34`.

**Required for orders [CONFIGURABLE]:** The `FAAOPF.FAALDS` flag controls whether delivery date is mandatory:
- `FAALDS = 0`: delivery date is optional (validated only if entered)
- `FAALDS = 1`: delivery date is required for orders (`VAOAKK > 0`), not for quotes

Additionally, if the configuration switch `u_726` is active, delivery date is always required on any document with `VAOAKK > 0`.

### 9.3 Follow-Up Date (FODDAT) for Quotes

**What is checked:** The follow-up date for a quote (`b1ddat`) must be a valid date and must be on or after the order date.

**Auto-calculation:** When a quote is first created, if no follow-up date is entered, the system can auto-calculate it by adding `FAAOPF.FAAOPF` (days to follow-up, a 3-digit field in the FAAPF configuration register) to the order date.

**Required for quotes [CONFIGURABLE]:** With the `u_ddat` flag active, quotes (`VAOAKK = 0`) require a follow-up date. If `b1ddat = 0` and `u_ddat = *on`, indicator `*in58` fires.

### 9.4 Expiry Date (FOTDAT) for Quotes

**What is checked:** The quote expiry date (`b1tdat`) must be a valid date and must be *strictly greater than* (not equal to) the order date.

**Auto-calculation:** The expiry date can be calculated automatically by adding `FAAOPF.FAAUTG` (days until expiry, also a 3-digit field) to the order date when the quote is created.

### 9.5 Invoice Date (SOFKDA)

**Set by invoicing process:** `SOFKDA` is not entered by users during order creation. It is written by FO614R at the moment the invoice is generated, set to the current system date (or a date entered in the invoicing run parameters).

**Not validated on entry:** Since it is system-set, no user validation applies to this field. The invoice date drives the calculation of the due date `SOFFDA` based on the payment terms `SOBETB`.

---

## 10. Status and Workflow Rules

### 10.1 Order States

An order moves through the following states:

| State | Description | Key indicator |
|-------|-------------|---------------|
| New (header only) | Header created, no item lines yet | No FODTPF rows |
| Open | Has item lines; available for picking | FODTPF rows exist, `SOBINR = 0` |
| On Hold | Has an info code blocking processing | `SOINF1 <> blank` |
| In Picking | Packing slip printed; picking in progress | `SOPAKS > 0` |
| Picked | All lines confirmed picked | All lines confirmed |
| Partially fulfilled | Some lines fulfilled; backorder created | New suffix record, `SOREST = 1` |
| Invoiced | Invoice number assigned | `SOBINR > 0` |

### 10.2 Order Lock

**What is checked:** `SOHEPF.FOKODE` (in the FOHEPF working file) is set to a non-zero value when an order is being actively processed (for example, during picking or invoicing). This prevents two users from modifying the same order simultaneously.

**What happens:** If a user tries to open an order that has `FOKODE <> 0`, they see a "being processed" warning screen (`xc2win` subroutine, FU704R called). Limited display-only access is available, but modifications are blocked until the processing is complete and `FOKODE` is reset to 0.

### 10.3 Picking Status

**What is checked:** `SOHEPF.SOPAKS > 0` means a packing slip has been printed and picking has begun. Re-printing is only allowed with the packing slip authority flag in FUSRPF (FO614R R5.01 / R5.41).

**Limited modifications after picking:** Once picking has started, changes to quantities, items, or warehouses are restricted. Changes that would affect already-picked quantities require supervisor intervention.

**What happens:** Attempting to re-print without authority shows an error. Attempting to delete an order with a packing slip in progress also requires authority.

### 10.4 Info Code Hold (SOINF1/SOINF2)

**What is checked:** `SOINF1` is a 12-character code referencing the VINFL1 register (`VINFPF`). VINFL1 contains a flag `VAIFAK` (stop at invoicing) that determines whether the hold code blocks invoicing.

**Why this is required:** Holds are used for order review situations — for example, a price dispute, a quality check, or a compliance requirement. The hold code `SOINF1` together with text `SOINF2` gives the reviewer context.

**What happens:** FO614R reads the VINFL1 register. If `VAIFAK = 'J'` (yes, stop at invoicing), the order is excluded from the invoicing run. The hold must be cleared (SOINF1 blanked) by an authorised user before the order can be invoiced.

### 10.5 Backorder (Resting) Processing

**How it works:** When FO614R processes an order and finds that not all lines can be fulfilled, it:

1. Creates a new order header record with the same order number but an incremented suffix (`SOSUFF + 1`)
2. Copies all unfulfilled lines to the new suffix
3. Sets `SOREST = 1` on the new backorder header
4. Updates the original suffix with the fulfilled quantities and proceeds to invoicing
5. If no lines remain on the original suffix after the backorder split, the original header is deleted
6. Writes log entries (code `'R'` for rested) to the order log file

**What happens to open purchase orders:** If any backordered lines are linked to purchase orders (FPLOPF/FPLLPF), FO614R updates those purchase order links to point to the new backorder suffix. From version 8.09, if the order has a new suffix, linked purchase order lines (`LODTPF`) are also updated to reference the new suffix.

---

## 11. Integration Rules

### 11.1 Customer Project Integration

When `SOKPRO` is set on an order, the program calls `FL501R` to retrieve all project attributes from FKPRPF. The following header fields may be overridden by the project:

| Order Header Field | Source from FKPRPF |
|--------------------|---------------------|
| Delivery name/address (`SONAVN`, `SOADR1`, `SOADR2`, `SOSTED`) | `FLNAVN`, `FLADR1`, `FLADR2` |
| Contact phone (`b1tlfn`) | `FKPRPF.FLTLFA` |
| Contact mobile (`b1mobn`) | `FKPRPF.FLTLFM` |
| Invoice type (`SOFATY`) | `FKPRPF.FLFATY` |
| Payment terms (`SOBETB`) | `FKPRPF.FLBETB` |
| VAT code (`SOMKOD`) | `FKPRPF.FLMKOD` |
| Excl. discount suppression | `FKPRPF.FLEKGB` |
| Incl. discount suppression | `FKPRPF.FLFAGB` |
| Net invoice flag | `FKPRPF.FLNETO` |
| Discount category (`SORKAT`) | Via `FL710R` using project's settings |

This means an order for customer 5000 with project 42 may have completely different payment terms, invoice type, and delivery address than a standard order for customer 5000 with no project — all driven by the FKPRPF record.

If both `SOLKUN` (goods customer) and `SOKPRO` (project) are set, and `SOLKUN <> SOKUND`, the delivery address is taken from the goods customer's RKUNPF record (NXS-10959 fix, version 8.03).

### 11.2 EDI Requirements

**What is checked:** `SOHEPF.SOEDIH > 0` indicates the order originated from an EDI (Electronic Data Interchange) transmission. `SONREF` (external reference number, 15 chars) is required for EDI orders and must match the trading partner's order reference.

At the line level, `SDEDIH` mirrors the header EDI reference and `SDEDIL` holds the EDI line number from the partner's order, enabling exact line-by-line reconciliation with the trading partner's system.

**Why this is required:** EDI trading agreements typically require that every acknowledgement and invoice refers back to the partner's specific document and line numbers. Failure to carry these references causes EDI reconciliation failures at the partner.

### 11.3 Transport Integration

**What is checked:** When certain `SOLMAT` (shipping method) values are selected — those that are mapped to an external transport system — the order creation triggers a booking call to the carrier API.

**Cancellation on change:** If the shipping method is changed after a booking has been made, the system detects the change (comparing the new `SOLMAT` to the previously stored value in the header) and sends a cancellation to the carrier before making the new booking. This happens automatically in FO100R.

**Why this is required:** External carriers allocate capacity at booking time. Failing to cancel a booking when the method changes would create phantom capacity holds at the carrier.

### 11.4 Deposit / Prepayment

**What is checked:** `SOHEPF.SODEPO > 0` indicates the customer must pay a deposit before the order ships. `SODEPB` tracks how much has actually been paid. `SODEPT` tracks how much deposit has been applied to an invoice.

`SOFOSA` / `SOFOSI` are settlement fields that track allocated discount/reduction amounts (excl. VAT and incl. VAT respectively) used for deposit settlement reconciliation.

**Deletion restriction:** FO100R and FO500R prevent deletion of orders with unpaid deposits (`SODEPO > 0` and `SODEPB < SODEPO`). The deposit must be fully settled (`SOFOSI` applied) before the order can be deleted, to prevent the company from losing a deposit that was never returned or applied.

---

## 12. Field Reference

### Complete SOHEPF Header Field Table

| Field | Type | Norwegian | English Description |
|-------|------|-----------|---------------------|
| SOFIRM | 3,0 packed | Firmanr | Company number |
| SOAARR | 4,0 packed | År | Fiscal year (YYYY) |
| SOBINR | 8,0 packed | Fakturanummer | Invoice number; 0 = open order, > 0 = invoiced |
| SONUMM | 8,0 packed | Ordrenr | Order number |
| SOSUFF | 2,0 packed | Suffix | Backorder suffix (0 = original) |
| SOOTYP | 2 char | Ordretype | Order type code |
| SOLETY | 1,0 packed | Leveringstype | Delivery type (numeric lookup in VLTPPF) |
| SOLAGE | 2,0 packed | Lager | Warehouse number |
| SOKREF | 8,0 packed | Kreditt-ref. | Credit reference number |
| SOBDAT | Date (ISO) | Ordredato | Order creation date |
| SOLDAT | Date (ISO) | Leveringsdato | Requested delivery date |
| SOPDAT | Date (ISO) | Planlagt lev. | Planned delivery date (system-set) |
| SOPADA | Date (ISO) | Pakkseddel dato | Packing slip date |
| SOFKDA | Date (ISO) | Fakturadato | Invoice date (set during invoicing) |
| SOFFDA | Date (ISO) | Forfallsdato | Invoice due date |
| SOBETB | 2,0 packed | Bet.betingelser | Payment terms code |
| SOSELG | 4,0 packed | Selgernr | Salesperson number |
| SODIST | 4,0 packed | Distrikt | District code |
| SOLMAT | 2,0 packed | Leveringsmåte | Shipping method code |
| SOLBET | 2,0 packed | Leveringsbet. | Delivery terms code |
| SOVALK | 3 char | Valuta | Currency code |
| SORKAT | 3,0 packed | Rabatt-kategori | Discount category |
| SOSPED | 3,0 packed | Speditør | Freight forwarder code |
| SOKJOR | 2,0 packed | Kjørerute | Route code |
| SOMKOD | 1,0 packed | Mva-kode | VAT code |
| SOLKUN | 6,0 packed | Vare-adresse | Goods delivery customer number |
| SOKUND | 6,0 packed | Kundenummer | Invoice customer number |
| SOKUNA | 30 char | Kunde-alfa | Customer sort name |
| SONAVN | 30 char | Navn | Delivery name |
| SOADR1 | 30 char | Adresse 1 | Delivery address line 1 |
| SOADR2 | 30 char | Adresse 2 | Delivery address line 2 |
| SOSTED | 30 char | Poststed | Delivery postal city |
| SOAVDE | 4,0 packed | Avdeling | Sales department |
| SOKONT | 6,0 packed | Konto | GL account code |
| SOSPES | 4,0 packed | Spes | GL specification |
| SOPROJ | 6 char | Prosjektnr | External project number |
| SOUPRO | 6 char | U.Prosjekt | Sub-project number |
| SOAKTI | 6 char | Aktivitet | Activity code |
| SOVREF | 30 char | Vår ref. | Our reference |
| SODREF | 30 char | Deres ref. | Customer's reference |
| SOKPNR | 6,0 packed | Kontaktpersonnr | Contact person number |
| SOREKV | 20 char | Kund.rekv.nr. | Customer requisition number |
| SOLMTX | 20 char | Lev. måte tekst | Shipping method free text |
| SOLBTX | 75 char | Lev. bet. tekst | Delivery terms free text |
| SORABP | 5,2 packed | Rabatt i % | Order-level discount percentage |
| SOPASL | 5,2 packed | % påslag | Order-level markup percentage |
| SOPASK | 1 char | Påslag/Ønsket DB | Markup mode code |
| SOKOLI | 11,3 packed | Kolli | Number of parcels |
| SOSALP | 11,2 packed | Salg plikt | Gross sales — VAT-liable goods |
| SOSALH | 11,2 packed | Salg 1/2 mva | Gross sales — half-rate VAT goods |
| SOSALF | 11,2 packed | Salg fritt | Gross sales — VAT-exempt goods |
| SORK1P | 11,2 packed | Rab.1 plikt | Line discount 1 on VAT-liable goods |
| SORK1H | 11,2 packed | Rab.1 1/2 mva | Line discount 1 on half-rate VAT goods |
| SORK1F | 11,2 packed | Rab.1 fritt | Line discount 1 on VAT-exempt goods |
| SORK2P | 11,2 packed | Rab.2 plikt | Line discount 2 on VAT-liable goods |
| SORK2H | 11,2 packed | Rab.2 1/2 mva | Line discount 2 on half-rate VAT goods |
| SORK2F | 11,2 packed | Rab.2 fritt | Line discount 2 on VAT-exempt goods |
| SORB1P | 11,2 packed | Alm Rab.1 plikt | Bracket disc 1 — VAT-liable (almenning) |
| SORB1H | 11,2 packed | Alm Rab.1 1/2 | Bracket disc 1 — half-rate (almenning) |
| SORB1F | 11,2 packed | Alm Rab.1 fritt | Bracket disc 1 — exempt (almenning) |
| SORB2P | 11,2 packed | Alm Rab.2 plikt | Bracket disc 2 — VAT-liable (almenning) |
| SORB2H | 11,2 packed | Alm Rab.2 1/2 | Bracket disc 2 — half-rate (almenning) |
| SORB2F | 11,2 packed | Alm Rab.2 fritt | Bracket disc 2 — exempt (almenning) |
| SOGR1P | 11,2 packed | Br.rabgr1 plikt | Usage-right bracket 1 — VAT-liable |
| SOGR1H | 11,2 packed | Br.rabgr1 1/2 | Usage-right bracket 1 — half-rate |
| SOGR1F | 11,2 packed | Br.rabgr1 fritt | Usage-right bracket 1 — exempt |
| SOGR2P | 11,2 packed | Br.rabgr2 plikt | Usage-right bracket 2 — VAT-liable |
| SOGR2H | 11,2 packed | Br.rabgr2 1/2 | Usage-right bracket 2 — half-rate |
| SOGR2F | 11,2 packed | Br.rabgr2 fritt | Usage-right bracket 2 — exempt |
| SORBGP | 11,2 packed | Rabgr plikt | Discount group total — VAT-liable |
| SORBGF | 11,2 packed | Rabgr fritt | Discount group total — VAT-exempt |
| SORKOP | 11,2 packed | Orab plikt | Order discount total — VAT-liable |
| SORKOF | 11,2 packed | Orab fritt | Order discount total — VAT-exempt |
| SOKSTP | 11,2 packed | Kostp plikt | Cost total — VAT-liable |
| SOKSTF | 11,2 packed | Kostp fritt | Cost total — VAT-exempt |
| SOMVGF | 11,2 packed | Mva.grunnl fritt | VAT base — VAT-exempt goods |
| SOMVGP | 11,2 packed | Mva.grunnl plikt | VAT base — VAT-liable goods |
| SOFSUM | 11,2 packed | Fakturasum | Total invoice amount |
| SOKPRO | 6,0 packed | Kundeprosjekt | Customer project number |
| SOBKUN | 6,0 packed | Kunde/bestiller | Ordering customer number |
| SOKKUN | 6,0 packed | Kunde/kjøper | Buyer customer number |
| SONETO | 1,0 packed | Netto u/rab | Net invoice flag (suppress discount on printout) |
| SOUTSK | 1,0 packed | Kode for utskrift | Print code |
| SOUSKR | 10 char | Bruker gir Kreditt | User who authorised credit override (F18) |
| SOJOUR | 1 char | Journal kjørt | Journal posted flag |
| SOEDIH | 8,0 packed | Edi-hode | EDI header reference |
| SONREF | 15 char | Eksternt referansenr | External reference number |
| SOFSYS | 3 char | Eksternt system | External system code |
| SOBENR | 8,0 packed | Bestillings-nr | Purchase order number reference |
| SOBSUF | 2,0 packed | Bestillings-suffix | Purchase order suffix reference |
| SOMOBI | 15 char | Kundens mobilnr | Customer mobile number |
| SOKBNR | 15 char | Kundens best.nr | Customer's own order number |
| SOINF1 | 12 char | Informasjon-1 | Hold / info code |
| SOINF2 | 40 char | Informasjon-2 | Hold / info description |
| SODEPO | 11,2 packed | Krav til Depositum | Required deposit amount |
| SODEPB | 11,2 packed | Betalt Depositum | Paid deposit amount |
| SODEPT | 11,2 packed | Trukket Depositum | Deposit deducted from invoice |
| SOFOSA | 11,2 packed | Fordelt red.salg | Settlement amount applied (excl. VAT) |
| SOFOSI | 11,2 packed | F.red.salgImva | Settlement amount applied (incl. VAT) |
| SOFATY | 2,0 packed | Fakturatype | Invoice type code |
| SOWSID | 10 char | Arbeidstasjon | Workstation ID |
| SOUSER | 10 char | Bruker | User who created the record |
| SODATE | Date (ISO) | Dato opprettet | Date record was created |
| SOTIME | Time (ISO) | Klokkeslett | Time record was created |
| SOOUSR | 10 char | Opprettet av | Audit: user who created the record |
| SOEDAT | Date (ISO) | Endringsdato | Audit: date of last change |
| SOETIM | Time (ISO) | Endringstidspunkt | Audit: time of last change |
| SOEUSR | 10 char | Endret av | Audit: user who made the last change |

### Complete SODTPF Line Field Table

| Field | Type | Norwegian | English Description |
|-------|------|-----------|---------------------|
| SDFIRM | 3,0 packed | Firmanr | Company number |
| SDAARR | 4,0 packed | År | Fiscal year |
| SDBINR | 8,0 packed | Fakturanummer | Invoice number |
| SDNUMM | 8,0 packed | Ordrenr | Order number |
| SDSUFF | 2,0 packed | Suffix | Backorder suffix |
| SDLINE | 4,0 packed | Linjenr | Line number within order |
| SDVARE | 15 char | Varenr | Item / product number |
| SDLTYP | 2 char | Linje-type | Line type code (VLTYPF lookup) |
| SDLETY | 1,0 packed | Leveringstype | Line-level delivery type |
| SDAVDE | 4,0 packed | Avdeling | Line-level department |
| SDKONT | 6,0 packed | Kontonummer | GL account for this line |
| SDSPES | 4,0 packed | Spesifikasjon | GL specification |
| SDLAGE | 2,0 packed | Lager | Line-level warehouse |
| SDANTA | 11,3 packed | Antall bestilt | Ordered quantity (3 decimal places) |
| SDOPPR | 9,2 packed | Opprinnelig pris | Original price (before manual change) |
| SDSAPR | 9,2 packed | Salgspris | Sales price per unit |
| SDKOPR | 9,2 packed | Kost-pris | Cost price per unit |
| SDRAB1 | 5,2 packed | Linje-rabatt 1 | Line discount 1 (%) |
| SDRAB2 | 5,2 packed | Linje-rabatt 2 | Line discount 2 (%) |
| SDBRR1 | 5,2 packed | Bruksrettrabatt 1 | Bracket / usage-right discount 1 (%) |
| SDBRR2 | 5,2 packed | Bruksrettrabatt 2 | Bracket / usage-right discount 2 (%) |
| SDPASL | 5,2 packed | Påslag | Markup percentage |
| SDPSKM | 11,2 packed | Påslag Kr. Mgr | Markup amount (master group) |
| SDSALG | 11,2 packed | Salg | Line net sales amount |
| SDRKR1 | 11,2 packed | Rabatt 1 | Line discount 1 in currency |
| SDRKR2 | 11,2 packed | Rabatt 2 | Line discount 2 in currency |
| SDBRK1 | 11,2 packed | Bruksrett Rab 1 | Bracket discount 1 in currency |
| SDBRK2 | 11,2 packed | Bruksrett Rab 2 | Bracket discount 2 in currency |
| SDRKRO | 11,2 packed | Ordrerab. | Order discount on this line in currency |
| SDKOST | 11,2 packed | Kostpris | Total cost (price × quantity) |
| SDFOSA | 11,2 packed | Fordelt red.salg | Settlement discount on this line |
| SDKPRI | 1 char | Kode, nettopris | Net price code (F = fixed, skip recalc) |
| SDKMVA | 1,0 packed | Mva-kode | VAT code for this line |
| SDOMVA | 1 char | Økonomi MVA-kode | GL VAT category code |
| SDMPRO | 4,2 packed | Margin% | Margin percentage |
| SDKORD | 1,0 packed | Kode ordrerab. | Order discount override (0 = apply header SORABP) |
| SDLVRE | 1,0 packed | Lokalt vreg | Special / procured item (1 = yes) |
| SDEDIH | 8,0 packed | Edi-hode | EDI header reference |
| SDEDIL | 6,0 packed | Edi-linje | EDI line reference |
| SDBENR | 8,0 packed | Bestillings-nr | Linked purchase order number |
| SDBSUF | 2,0 packed | Bestillings-suff | Linked purchase order suffix |
| SDBLIN | 4,0 packed | Bestillings-linje | Linked purchase order line |
| SDKBNR | 15 char | Kundens best.nr | Customer's own order number |
| SDKBLI | 6,0 packed | Kundens best.linje | Customer's own order line |
| SDOPRG | 1,0 packed | Kode, Overst. prisgr | Overridden price group flag |
| SDWSID | 10 char | Arbeidstasjon | Workstation ID |
| SDUSER | 10 char | Bruker-ID | User who created the line |
| SDEUSR | 10 char | Endret av | User who last updated the line |

---

*Document compiled from source analysis of FO100R.MBR (v8.23), FO500R.MBR, FO614R.MBR (v8.15), FO730R.MBR (v8.02), and DDS reference files FFRF.MBR / RFRF.MBR from library NXCLOUD/NXKORR.*
