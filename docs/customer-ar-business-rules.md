# NexStep ERP — Customer and Accounts Receivable Business Rules

**Document scope:** Customer master (Kunderegister), accounts receivable ledger, payment processing, AR aging, and all supporting workflows in the NexStep/ASOKON system built on IBM i (AS/400) using RPG IV and DDS source.

**Source programs analysed:** RK100R, RK108R, RK110R, RK400R, RK500R, RK620R, RE110R, RE120R, RE600R, RE601R, RE602R, RE603R, RE609R, RE613R, RE615R, RE621R, RE623R, RE630R, RE632R, RE633R, AR100R, AR700R, RK750R, RN012R.

---

## 1. Introduction

### Purpose

This document records every business rule that governs how NexStep creates, validates, maintains, and settles customer accounts and the receivables that flow from them. It is written for consultants, developers, and finance staff who need to understand the system without reading source code.

### How to Use This Document

Read sections 3 and 4 before creating a customer. Read sections 5 and 6 before posting a transaction. Read sections 7 and 8 before processing payments. Read sections 9 and 10 for collections and period-end work. Section 11 covers interfaces; section 12 is the field-level reference.

### Field Prefix Conventions

| Prefix | Table | Meaning |
|--------|-------|---------|
| `RK` | RKUNPF | Customer master record fields |
| `RKM` | RKMEPF | Customer memo balance fields (internal: RKMFIR, RKMKUN) |
| `RN` | RKTRPF | Customer posting/transaction fields |
| `RT` | REMSPF | GL/AR batch registration fields |
| `RE` | REKUPF | AR ledger summary / factoring fields |
| `RA` | RAxxPF | Configuration and code table fields |
| `AK` | AKIDPF | KID (payment reference) link fields |

### Key Norwegian Terminology

| Term | Literal Translation | System Meaning |
|------|--------------------|----|
| **Kunderegister** | Customer register | The RKUNPF customer master table and the programs that maintain it (RK100R, RK108R) |
| **Memosaldo** | Memo balance | Real-time credit exposure tracked in RKMEPF.RKMEMA, updated every time an invoice is posted or a payment is received |
| **Postering** | Posting | A single AR transaction record in RKTRPF — one row per invoice, payment, or credit note |
| **Aldersfordeling** | Age distribution | AR aging: open RKTRPF records grouped into overdue buckets by number of days past RNFDAT (due date) |
| **KID** | Customer Identification Number | Norwegian payment reference number printed on invoices; the payer writes it on the bank giro so receipts can be matched automatically; stored in RNKIDR/RNKIDI on the posting |
| **Remisse** | Remittance | OCR/EFT payment file received from the bank containing KID numbers; matched against open RKTRPF records to settle invoices |
| **Spesialavtale** | Special agreement | Customer-specific pricing or terms stored in RKSAPF and flagged by RKSPAV on the master record |
| **Buntnummer** | Bundle number | RTBUNT in REMSPF — a batch identifier grouping related GL/AR registrations entered in one session |

---

## 2. Database Schema Overview

### 2.1 RKUNPF — Customer Master

Primary key: `RKFIRM` + `RKKUND`. One row per customer per company. Reference file: RFRF.

| Field | Type | Len | Description |
|-------|------|-----|-------------|
| RKFIRM | Numeric | 3,0 | Company number |
| RKKUND | Numeric | 6,0 | Customer number (system key) |
| RKNAVN | Alpha | 30 | Customer name — mandatory, cannot be blank |
| RKALFA | Alpha | 18 | Alpha search key — auto-populated from RKNAVN if blank |
| RKGATE | Alpha | 30 | Street address |
| RKADR2 | Alpha | 30 | Secondary address line |
| RKPONR | Numeric | 4,0 | Postal code |
| RKSTED | Alpha | 30 | City |
| RKBGAT | Alpha | 30 | Visit street (separate delivery address) |
| RKBADR | Alpha | 30 | Visit address line 2 |
| RKBPNR | Numeric | 4,0 | Visit postal code |
| RKKOMN | Numeric | 4,0 | Municipality number |
| RKLAND | Alpha | 2 | Country code |
| RKUPNR | Alpha | 9 | Foreign postal number |
| RKFTNR | Numeric | 9,0 | Organisation number (Norwegian foretaksnummer) |
| RKPENR | Numeric | 11,0 | Personal identity number |
| RKPERS | Alpha | 30 | Contact person |
| RKEMAL | Alpha | 50 | Email address — used for electronic invoice/statement delivery |
| RKWEBA | Alpha | 40 | Web address |
| RKTLFN | Alpha | 11 | Phone number |
| RKTLFX | Alpha | 11 | Fax number |
| RKMOBN | Alpha | 11 | Mobile number |
| RKMOBO | Alpha | 2 | Mobile operator code |
| RKBGIR | Numeric | 11,0 | Bank giro number |
| RKFAKU | Numeric | 8,0 | Invoice customer number — consolidated billing; invoices go to this customer instead |
| RKFACN | Numeric | 8,0 | Factoring reference number |
| RKKOKU | Numeric | 6,0 | Consortium/group customer number |
| RKEDIK | Alpha | 1 | EDI code: ' '=none, 'A'=EDIFACT, 'E'=EAN |
| RKEANK | Numeric | 13,0 | EAN customer number |
| RKEANF | Numeric | 13,0 | EAN invoice customer number |
| RKAVDE | Numeric | 4,0 | Department |
| RKKJØR | Numeric | 2,0 | Delivery route |
| RKKJNR | Numeric | 5,0 | Sequence within route |
| RKKATG | Numeric | 4,0 | Customer category |
| RKRABK | Numeric | 3,0 | Discount category |
| RKLAGR | Numeric | 2,0 | Default warehouse |
| RKSLGR | Numeric | 4,0 | Salesperson — must exist in RA09PF |
| RKDIST | Numeric | 4,0 | District |
| RKBETB | Numeric | 2,0 | Payment terms code (days) |
| RKBETM | Numeric | 1,0 | Payment method code |
| RKSALD | Numeric | 11,2 | Sales year-to-date — cumulative invoiced amount; reset annually |
| RKSAIF | Numeric | 11,2 | Sales in prior year — copy of RKSALD from previous year-end |
| RKKJØP | Numeric | 11,2 | Purchases this year |
| RKKJØF | Numeric | 11,2 | Purchases last year |
| RKKJAL | Numeric | 11,2 | Almenning (cooperative) purchases this year |
| RKGRAL | Numeric | 9,0 | Almenning purchase limit |
| RKKTID | Numeric | 3,0 | Credit time in days |
| RKTDAT | Date | — | Last transaction date |
| RKVALK | Alpha | 3 | Default currency code |
| RKSPRK | Alpha | 1 | Block code: ' '=active, 'J'=order blocked |
| RKMKOD | Numeric | 1,0 | VAT code: 0=standard, 1=exempt, 2=export |
| RKLADR | Numeric | 1,0 | Delivery address flag: 0=postal, 1=visit |
| RKLBET | Numeric | 2,0 | Delivery terms code |
| RKFORM | Numeric | 2,0 | Delivery method code |
| RKSPED | Numeric | 3,0 | Freight forwarder number |
| RKORAB | Numeric | 5,2 | Order discount % |
| RKGRNS | Numeric | 5,0 | Credit limit (integer, unit defined per installation) |
| RKKGRK | Alpha | 1 | Credit limit type code |
| RKFRIE | Alpha | 5 | Free classification codes |
| RKRENK | Numeric | 1,0 | Interest code: 0=normal, 1=no interest, 2=level 2, 3=level 3 |
| RKRNEG | Alpha | 1 | Interest note for negative balance flag |
| RKRDAT | Date | — | Date of last interest calculation |
| RKPURR | Numeric | 1,0 | Dunning code: 0=inactive, 1=active |
| RKPDAT | Date | — | Date of last dunning notice sent |
| RKPNEG | Alpha | 1 | Send dunning for negative balance flag |
| RKINKK | Alpha | 1 | Collections flag: ' '=normal, 'J'=in collections |
| RKIDAT | Date | — | Date customer was referred to collections |
| RKFACK | Numeric | 1,0 | Factoring code: 0=no, 1=yes |
| RKSAMM | Alpha | 1 | Summary invoice marker |
| RKORDB | Numeric | 1,0 | Order confirmation: 0=no, 1=yes |
| RKKRES | Numeric | 1,0 | Credit block level: 0=none, 1=warning, 2=order block, 3=full lock |
| RKSPAV | Numeric | 1,0 | Special agreement flag |
| RKPRPA | Numeric | 1,0 | Price/packing slip option |
| RKNETO | Numeric | 1,0 | Show price on invoice: 0=no, 1=yes |
| RKSAMF | Numeric | 1,0 | Consolidated invoice option (0–4) |
| RKREKV | Numeric | 1,0 | Requisition required: 0=no, 1=yes |
| RKFAGB | Numeric | 1,0 | Invoice fee: 0=no, 1=yes |
| RKEKGB | Numeric | 1,0 | Expedition fee: 0=no, 1=yes |
| RKFRGB | Numeric | 1,0 | Freight fee: 0=no, 1=yes |
| RKRASP | Numeric | 1,0 | Discount block: 0=no, 1=yes |
| RKFRSK | Alpha | 10 | Free statistics code |
| RKPBOK | Numeric | 1,0 | Invoice by price book: 0=no, 1=yes |
| RKPTAB | Numeric | 3,0 | Markup table 1 |
| RKPTA2 | Numeric | 3,0 | Markup table 2 |
| RKSPFA | Numeric | 1,0 | Specify invoicing detail: 0=standard, 1=detail, 2=summary |
| RKSPL1–5 | Alpha | 1 | Payroll specification flags (5 fields) |
| RKSEKV | Numeric | 2,0 | Invoice sequence interval (values: 0, 1, 7, 14, 30 days) |
| RKFAGR | Numeric | 4,0 | Minimum invoice amount threshold |
| RKAFDA | Date | — | Valid from date for AG (direct debit) |
| RKATDA | Date | — | Valid to date for AG (direct debit) |
| RKSENT | Numeric | 1,0 | AG mandate sent: 0=no, 1=yes |
| RKTILB | Numeric | 8,0 | Offer number |
| RKOCOP | Numeric | 1,0 | Order confirmation copies |
| RKPCOP | Numeric | 1,0 | Packing slip copies |
| RKFCOP | Numeric | 1,0 | Invoice copies |
| RKPASS | Alpha | 1 | Passive customer: ' '=active, 'J'=passive (excluded from normal searches) |
| RKEBIZ | Numeric | 1,0 | E-commerce customer: 0=no, 1=yes |
| RKLETY | Numeric | 1,0 | Delivery type |
| RKFATY | Numeric | 2,0 | Invoice type |
| RKKOTI | Numeric | 1,0 | Cash customer offer flag |
| RKNDAT | Date | — | Record creation date |
| RKNTIM | Time | — | Record creation time |
| RKNSIG | Alpha | 10 | Created by (user ID) |
| RKEDAT | Date | — | Last modification date |
| RKETIM | Time | — | Last modification time |
| RKESIG | Alpha | 10 | Last modified by (user ID) |
| RKEPGM | Alpha | 10 | Last modified by program |

### 2.2 RKMEPF — Customer Memo Balance

Primary key: `RKFIRM` + `RKKUND` (via internal alias fields RKMFIR, RKMKUN). One row per customer per company. Tracks real-time credit exposure — not a ledger; updated immediately on every invoice/payment.

| Field | Type | Len | Description |
|-------|------|-----|-------------|
| RKMFIR | Numeric | 3,0 | Company number (key) |
| RKMKUN | Numeric | 6,0 | Customer number (key) |
| RKMEMS | Numeric | 11,2 | Standard memo balance — sum of all open AR for this customer |
| RKMEMA | Numeric | 11,2 | Almenning memo balance — credit exposure for cooperative purchases |
| RKMEDA | Date | — | Last modification date (standard) |
| RKMETI | Time | — | Last modification time (standard) |
| RKMEUS | Alpha | 10 | Last modified by (standard) |
| RKMODA | Date | — | Last modification date (Almenning) |
| RKMOTI | Time | — | Last modification time (Almenning) |

### 2.3 RKTRPF — Customer Postings

Primary key: `RNFIRM` + `RNKUND` + `RNRAAR` + `RNRPER` + `RNBDAT` + `RNBILN`. One row per AR transaction. Field prefix is `RN` — not `RT`.

| Field | Type | Len | Description |
|-------|------|-----|-------------|
| RNFIRM | Numeric | 3,0 | Company number |
| RNKUND | Numeric | 6,0 | Customer number |
| RNRAAR | Numeric | 4,0 | Accounting year (four digits) |
| RNRPER | Numeric | 2,0 | Accounting period (1–13) |
| RNBDAT | Date | — | Document date |
| RNBILN | Numeric | 8,0 | Voucher/document number |
| RNBILK | Numeric | 2,0 | Document type code (see section 6.1) |
| RNTEXT | Alpha | 20 | Document description text |
| RNBELØ | Numeric | 11,2 | Original document amount (positive=debit, negative=credit) |
| RNREST | Numeric | 11,2 | Remaining/outstanding amount; starts at RNBELØ, reduced as payments applied |
| RNVALK | Alpha | 3 | Currency code |
| RNVALB | Numeric | 11,2 | Original amount in foreign currency |
| RNVALR | Numeric | 11,2 | Remaining amount in foreign currency |
| RNFDAT | Date | — | Due date — basis for aging bucket calculation |
| RNJOUR | Numeric | 5,0 | Journal list number |
| RNSAMF | Numeric | 8,0 | Consolidated invoice number |
| RNSTAT | Alpha | 1 | Status: ' ' or 'O'=open, 'L'=settled, 'D'=deleted |
| RNREFN | Numeric | 8,0 | Cross-reference document number |
| RNVIRK | Alpha | 2 | Business/activity code |
| RNAVDE | Numeric | 4,0 | Department |
| RNKONT | Numeric | 4,0 | GL account |
| RNSPES | Numeric | 4,0 | Cost carrier |
| RNSLGR | Numeric | 4,0 | Salesperson |
| RNREKL | Alpha | 1 | Dispute/complaint flag: ' '=normal, 'J'=disputed |
| RNRENK | Numeric | 1,0 | Interest status: 0=pending, 1=charged, 3=excluded, 5=waived |
| RNRDAT | Date | — | Last interest calculation date |
| RNANTP | Numeric | 1,0 | Count of dunning notices sent |
| RNPDAT | Date | — | Date of last dunning notice |
| RNINKO | Alpha | 1 | Collections status: ' '=none, 'B'=sent to collections, 'J'=confirmed |
| RNIDAT | Date | — | Collections referral date |
| RNODAT | Date | — | Settlement date |
| RNAUTO | Alpha | 1 | AutoGiro code |
| RNADAT | Date | — | AutoGiro date |
| RNFACK | Numeric | 1,0 | Exclude from factoring: 0=include, 1=exclude |
| RNFADA | Date | — | Date transferred to factoring |
| RNTIST | Timestamp | — | Posting timestamp |
| RNKIDI | Alpha | 25 | KID customer identity string (full KID number) |
| RNKIDR | Numeric | 8,0 | KID reference number (key for AKIDPF lookup) |
| RNKSIF | Numeric | 1,0 | KID check digit (MOD10 or MOD11) |
| RNKID2 | Alpha | 25 | Second KID number |
| RNSPOR | Numeric | 10,0 | Trace/source reference |
| RNAVKR | Numeric | 10,0 | Confirmation reference |
| RNDATE | Date | — | Last modification date |
| RNOTIM | Time | — | Last modification time |
| RNOUSR | Alpha | 10 | Last modified by |
| RNEDAT | Date | — | Record creation date |
| RNETIM | Time | — | Record creation time |
| RNEUSR | Alpha | 10 | Created by |

### 2.4 REKUPF — AR Ledger Summary / Factoring Customer Postings

Primary key: `RKFIRM` + `RKKUND`. Contains 116 fields mirroring RKUNPF customer data plus memo balance (RKMEMS). Used for factoring integration and consolidated AR ledger summaries. Fields use RE-prefix aliases (REKUND references RKKUND, RENAVN references RKNAVN, REBELØ references RNBELØ, etc.) defined via REFFLD in RFRF.

### 2.5 REMSPF — GL/AR Posting Registration

Primary key: `RTFIRM` + `RTBUNT` + `RTLINJ`. One row per line in a GL/AR posting batch. Field prefix is `RT`.

| Field | Type | Len | Description |
|-------|------|-----|-------------|
| RTFIRM | Numeric | 3,0 | Company number |
| RTBUNT | Numeric | 5,0 | Bundle/batch number |
| RTLINJ | Numeric | 5,0 | Line number within bundle |
| RTREID | Alpha | 1 | Record ID (H=header, K=customer, L=supplier, P=project) |
| RTSTAT | Alpha | 1 | Bundle status |
| RTBDAT | Date | — | Document date |
| RTBELØ | Numeric | 11,2 | Transaction amount |
| RTRAAR | Numeric | 4,0 | Accounting year |
| RTRPER | Numeric | 2,0 | Accounting period (1–13) |
| RTBILN | Numeric | 8,0 | Voucher number |
| RTBILL | Numeric | 4,0 | Voucher line number |
| RTDAVD | Numeric | 4,0 | Debit department |
| RTDKTO | Numeric | 6,0 | Debit account (GL account number) |
| RTDSPS | Numeric | 4,0 | Debit cost carrier |
| RTDMVA | Alpha | 1 | Debit VAT code |
| RTCAVD | Numeric | 4,0 | Credit department |
| RTCKTO | Numeric | 6,0 | Credit account (GL account number) |
| RTCSPS | Numeric | 4,0 | Credit cost carrier |
| RTCMVA | Alpha | 1 | Credit VAT code |
| RTREFN | Numeric | 8,0 | Reference document number |
| RTFDAT | Date | — | Due date |
| RTKRAB | Numeric | 5,2 | Cash discount amount |
| RTVALK | Alpha | 3 | Currency code |
| RTVALB | Numeric | 11,2 | Foreign currency amount |
| RTRESN | Numeric | 6,0 | AR/AP sub-ledger account |
| RTRESS | Numeric | 4,0 | AR/AP cost carrier |
| RTRESA | Numeric | 4,0 | AR/AP department |
| RTVIRK | Alpha | 2 | Business/activity code |
| RTKIDI | Alpha | 25 | KID customer identity string |
| RTKIDR | Numeric | 8,0 | KID reference number |
| RTKSIF | Numeric | 1,0 | KID check digit |
| RTKID2 | Alpha | 25 | Second KID number |
| RTBILK | Numeric | 2,0 | Document type code |
| RTTEXT | Alpha | 20 | Document description |
| RTBETB | Numeric | 2,0 | Payment terms code |
| RTBREF | Alpha | 12 | Bank reference |
| RTDECK | Alpha | 6 | Declaration/customs code |
| RTAUTX | Alpha | 1 | Settled/crossed flag ('K'=settled) |
| RTBLDE | Alpha | 1 | Registration screen identifier |
| RTEDAT | Date | — | Last modification date |
| RTETIM | Time | — | Last modification time |
| RTEUSR | Alpha | 10 | Last modified by |

### Note on RE01PF, RE02PF, RE03PF

These three tables are **GL reporting tables only** — they are not AR transaction tables and play no role in AR posting, payment, or aging:

- **RE01PF** — Saldomatriser Trioplan: 12-column monthly balance matrix used by RE603R to export GL trial balance to the Trioplan financial system.
- **RE02PF** — Resultatregister til PC: Income statement register written by RE621R/RE623R for PC-based reporting tools (Akelius, Finale, Maestro).
- **RE03PF** — Saldobalanse til PC: Trial balance register written by RE601R for PC-based reporting.

---

## 3. Customer Master Prerequisites

### 3.1 Required Master Data Lookups

Before a customer record can be saved, the following codes must exist in their respective tables:

| Field | Lookup Table | What Is Checked |
|-------|-------------|-----------------|
| RKBETB (payment terms) | RA03PF | Code must exist; description loaded to confirm validity |
| RKBETM (payment method) | RA02PF | Code must exist |
| RKSLGR (salesperson) | RA09PF | RAIKOD must match an active salesperson record |
| RKAVDE (department) | Department master | Department must exist |
| RKKATG (category) | RA511 lookup | Category code must be valid |
| RKRABK (discount category) | RA506 lookup | Discount category code must be valid |
| RKLAGR (warehouse) | Logistics master | Warehouse must exist |
| RKDIST (district) | RA510 lookup | District code must be valid |

**Why this matters:** Downstream programs (invoicing, pricing, shipping) rely on these codes being resolvable. A customer with an invalid salesperson code will cause errors in sales commission reporting. An invalid payment terms code will produce incorrect due dates on RKTRPF.RNFDAT.

### 3.2 User Permission Model

RK100R checks the user's authorisation record in AUSRL1 (user file, field FAAK03) before allowing certain operations:

- **FAAK03 level < 5:** Full maintenance access (create, edit, delete, copy).
- **FAAK03 = 5:** Read-only access; F8 (posting inquiry to RK110R) is shown but edit functions are suppressed.
- **ABBAVD = 'J' on the user record:** Restricts the user to their own department; in RK620R (customer register print), the department-from and department-to fields are locked to the user's department and cannot be changed.

### 3.3 Customer Number Range Rules

Customer numbers are controlled by company-level configuration in RAA1PF (the status record, usually key 000001 for each company), which holds:

- **RA1HKT** (Numeric 6,0): Lower bound — customer numbers must be **greater than** this value.
- **RA1HRS** (Numeric 6,0): Upper bound — customer numbers must be **less than or equal to** this value.

**What is checked:** Every program that creates or imports a customer — RK100R (new customer screen), RE110R (external import) — validates:

```
rkkund > ra1hkt  AND  rkkund <= ra1hrs
```

If the number falls outside this range, processing stops with an error. This range separates customer number spaces between divisions or subsidiary companies that share the same physical library.

**New number allocation:** When a user presses F6 (new customer) in RK100R, the program calls RS001R (sequence number program) which reads and increments RA4KNR in RAA4PF (the last-used customer number counter). The next available number within the RA1HKT–RA1HRS range is proposed and may be overridden by the user.

---

## 4. Customer Master Validation Rules

### 4.1 Name Is Mandatory

**What is checked:** RKUNPF.RKNAVN must be non-blank at write time. The DDS definition includes `COMP(NE ' ')` with `CHKMSGID(RMK0370 *LIBL/QMFOKON)`.

**Why this is required:** RKNAVN feeds the alpha search key (RKALFA), the search text register (RKSOPF via RK750R), and every printed document. A blank name breaks customer lookup, dunning notices, and invoice generation.

**What happens when RKNAVN is blank:** The screen displays message RMK0370 and the cursor returns to the name field. The record is not written.

### 4.2 Alpha Key Auto-Population

**What is checked:** Before calling RK750R to update the search register, RK100R checks whether RKALFA is blank. If it is, the program sets RKALFA = RKNAVN (first 18 characters) and converts it to uppercase using `XLATE` with the constant tables `lo`/`up` (Norwegian lowercase and uppercase character sets).

**Why this is required:** Searching by alpha key in RK500R (customer inquiry subprogram) uses RKALFA as the database key for `rkunl2` (customer master logical ordered by alpha key). A blank alpha key makes the customer unreachable by name search.

**What happens when this runs:** RK750R then rebuilds RKSOPF.RKSTEK (150-character search text) by concatenating RKALFA, RKNAVN, RKGATE, RKSTED, RKTLFN, RKMOBN, RKKUND, and RKFTNR — all converted to uppercase. This enables free-text search via SQL on the RKSOPF file in RK500R.

### 4.3 Organisation Number Format

**Field:** RKUNPF.RKFTNR — Numeric 9,0. The Norwegian foretaksnummer (organisation number) is always exactly nine digits. The system stores it as a zero-padded nine-digit integer.

**What is checked:** No algorithmic MOD-11 check is applied within the maintenance programs; the user is responsible for entering the correct number. The value is used for duplicate detection (see section 4.5) and for OCR/KID number generation.

### 4.4 Block Status (RKSPRK)

**Field:** RKUNPF.RKSPRK — Alpha 1, values ' ' or 'J'. This is a manual, unconditional block. Setting RKSPRK = 'J' prevents new sales orders regardless of credit exposure.

**What is checked:** The order processing system reads RKSPRK first. If 'J', the order is blocked before any credit limit check occurs.

**Relationship to RKKRES:** RKSPRK and RKKRES are independent. A customer can have RKSPRK = 'J' (manual block) with RKKRES = 0 (no credit issue), or RKSPRK = ' ' (not manually blocked) with RKKRES = 3 (fully credit-locked).

### 4.5 Invoice Customer (RKFAKU)

**Field:** RKUNPF.RKFAKU — Numeric 8,0. When non-zero, AR invoices for this customer are billed to the RKFAKU customer number instead. This supports consolidated billing: a group of delivery customers can all have invoices sent to one head-office invoice customer.

**What is checked at maintenance:** The RKFAKU customer must exist in RKUNPF for the same company. If RKFAKU is changed while open AR transactions exist, the existing postings in RKTRPF retain the original RNKUND assignment; only future invoices go to the new invoice customer.

**Why this is required:** Without RKFAKU, every delivery point would receive its own invoice. Large retail chains require a single monthly consolidated invoice to their accounts payable department.

### 4.6 Duplicate Detection via RK108R

**What is checked:** RK108R contains a "show duplicates" toggle (F7 key, indicator `*in58`). When active, the subroutine `xsrkont` compares each customer's name, address, postal code, and phone number against adjacent records in the alpha-sorted subfile. Customers that match on the selected comparison field are flagged and displayed together.

**What happens when duplicates are found:** The operator can select the "move" option (option 6) on one customer row, enter the target customer number, and RK108R copies all postings, KID references, memos, special agreements, and notes to the target, then deletes the source customer. The TESTSPERRE check (section 4.7) must pass first.

### 4.7 Deletion Checklist via RK400R

RK400R is a parameter screen that calls RK400C to batch-delete customers meeting specific age criteria. Both RK100R (interactive) and RK400C (batch) enforce the same preconditions:

**A customer can only be deleted when all of the following are true:**

1. `RKUNPF.RKSALD = 0` — no outstanding sales balance.
2. `RKMEPF.RKMEMS = 0` — no standard memo balance.
3. `RKMEPF.RKMEMA = 0` — no Almenning memo balance.
4. No records exist in RKTRPF for this company + customer number (chain to `rktrl1` fails = no postings, open or settled).
5. No records exist in SOHEPF for this company + customer (program RK792C returns blank status = no open sales orders).
6. RKKUND is not equal to system cash-sale customer numbers stored in company configuration (FAAKUN, FAAKNR, BAAKUN).

**What RK400R additionally validates:** The operator enters two date filters — "older than date" (a2edat) and "entered after date" (a2ndat). RK400C deletes only customers whose last transaction date (RKTDAT) is older than a2edat AND whose creation date (RKNDAT) is after a2ndat. This prevents accidental deletion of long-standing customers with no recent activity.

**On successful delete:** RKUNPF record is removed; associated records in RKSOPF (search text), RKSAPF (special agreements), RKLAL2 (note blocks), and RKU3PF (PM register) are also deleted.

### 4.8 Merge/Move Rules via RK108R

**TESTSPERRE check:** Before allowing any merge, RK108R calls the function `TESTSPERRE`. If the current company/group has the merge feature locked, a message is shown and the program exits. This prevents unauthorised reorganisation of customer numbers during live operations.

**Move validation rules:**

1. Target customer number must be non-zero.
2. Source and target must not be the same customer number.
3. If the target customer does not exist, RK108R creates it by copying RKUNPF fields from the source.
4. If the target customer already exists, balances are merged: target RKSALD += source RKSALD; target RKMEMS += source RKMEMS; target RKMEMA += source RKMEMA.

**What is migrated:** All RKTRPF postings (RNKUND updated to target), AKIDPF KID references, RKSOPF search text, RKSAPF special agreements, and RKLAL2 note blocks are moved. The original customer is deleted after a successful move.

---

## 5. Credit and Financial Limit Rules

### 5.1 Credit Limit Field (RKGRNS)

**Field:** RKUNPF.RKGRNS — Numeric 5,0. Stores the maximum permitted outstanding AR balance for this customer. The unit of measure (NOK, hundreds, thousands) is set at installation level.

**What is checked:** The sales order processing module reads RKGRNS and compares it against the current memo balance (RKMEPF.RKMEMA). If the new order amount would push RKMEMA above RKGRNS, the response depends on RKKRES (see section 5.3).

**RKGRNS = 0 interpretation:** Zero typically means no credit limit is configured (unlimited), not that the limit is zero. Installations that want a hard zero limit set a small positive value such as 1.

### 5.2 Why RKMEPF Is Used for Real-Time Credit Checks Instead of RKSALD

| Measure | Field | What It Represents | Update Frequency |
|---------|-------|-------------------|-----------------|
| RKSALD | RKUNPF.RKSALD | Cumulative sales year-to-date — total invoiced, regardless of payment | Updated at invoice creation; reset at year-end |
| Memo balance | RKMEPF.RKMEMA | Net outstanding AR — invoiced but not yet paid | Updated immediately at every invoice post and every payment receipt |

**Why the distinction matters:** Consider a customer invoiced 500,000 NOK in January, who paid all invoices in February, then placed a new order in March. RKSALD = 500,000 (year-to-date sales), but RKMEMA = 0 (nothing outstanding). Using RKSALD for a credit check would incorrectly block the March order even though the customer owes nothing.

**Real-world example:** Customer 10500 has RKGRNS = 50 (50,000 NOK credit limit). Three open invoices total 38,000 NOK — so RKMEMA = 38,000. A new order worth 15,000 NOK would push exposure to 53,000 NOK, exceeding the limit. The credit check fires. RKSALD might show 200,000 NOK (year-to-date sales) and would not usefully constrain the check.

### 5.3 Four-Level Credit Block (RKKRES)

**Field:** RKUNPF.RKKRES — Numeric 1,0, values 0–3 (added in system version R6.26).

| Value | Name | What Happens |
|-------|------|-------------|
| 0 | No block | Orders proceed normally; no warning |
| 1 | Warning | Order can proceed; system displays a credit warning to the operator who may override |
| 2 | Order block | New orders are blocked; existing open orders may be held; credit review required before releasing |
| 3 | Full lock | All transactions blocked; no orders, no postings; used for customers in active dispute or collections |

**Who sets RKKRES:** The credit controller updates RKKRES manually on the customer master (RK100R) or the system sets it automatically when an automated credit check detects an overrun and the installation has auto-blocking configured.

### 5.4 Payment Terms Inheritance (RKBETB)

**Field:** RKUNPF.RKBETB — Numeric 2,0. When a customer is copied (option 3 in RK100R, subroutine `xk1win`), if the source customer's RKBETB = 0, the new customer inherits the system default payment terms from RA1BEK (the company status record field). This ensures that every customer has a valid payment terms code from creation.

**What payment terms control:** RKBETB drives the due date calculation on RKTRPF.RNFDAT. The number of days in the payment terms code is added to the invoice date (RNBDAT) to produce RNFDAT. Every aging bucket comparison uses RNFDAT as its reference point.

### 5.5 Almenning Purchase Limit

Almenning is a Norwegian cooperative purchasing scheme. Two fields on the customer master control cooperative purchase limits:

- **RKUNPF.RKKJAL** (Numeric 11,2): Running total of cooperative purchases made by this customer in the current year.
- **RKUNPF.RKGRAL** (Numeric 9,0): Maximum cooperative purchase amount permitted for this year.
- **RKMEPF.RKMEMA** (Numeric 11,2): Real-time memo balance for cooperative credit exposure.

**What is checked:** When a cooperative purchase order is created, RKKJAL + new order amount is compared to RKGRAL. Exceeding RKGRAL triggers a cooperative purchase limit warning or block.

---

## 6. Customer Transaction Rules

### 6.1 Document Type Codes in RKTRPF.RNBILK

RNBILK is a two-digit numeric field. Valid codes are defined in the RA01PF code table (RAAKOD). The standard codes used in Norwegian AR:

| Numeric Code | Short Name | Description | Sign of RNBELØ |
|-------------|-----------|-------------|----------------|
| 10 | FA — Faktura | Invoice — debits the customer | Positive |
| 20 | BE — Betaling | Payment received — credits the customer | Negative |
| 30 | KR — Kreditnota | Credit note — reduces outstanding balance | Negative |
| 40 | RE — Rentenota | Interest note — additional charge on overdue balance | Positive |
| 50 | IN — Inkasso | Collections charge — fee added at collections referral | Positive |
| 60 | SA — Samlefaktura | Consolidated invoice — groups multiple delivery notes | Positive |

The RA01PF record for each code also carries flags: RAAINK (payment indicator), RAARNK (interest indicator), RAAFAC (factoring indicator), RAASUM (consolidated invoice indicator). These flags drive which downstream processing routines handle each document type.

### 6.2 Period and Date Validation

**Period range:** Accounting periods 1 through 13 are valid. Period 13 is reserved for year-end adjustment entries. RE120R and RE600R enforce: `a2pper >= 1 AND a2pper <= 13`.

**Year validation:** The accounting year cannot be in the future. Both RE120R and RE600R enforce: `a2prår <= *year` (current system year).

**Open period check:** Before posting, the system checks whether the target period is open in the company's accounting calendar. A closed period rejects new postings.

**Two-digit year handling in RE120R:** During initialisation, RE120R computes the default posting period by subtracting 1 from the current month. If the result is zero (i.e., the current month is January), the program sets the default period to 12 and decrements the year by 1. This ensures that January postings default to December of the prior year, which is standard in Norwegian accounting where January represents the closing period for the previous fiscal year.

### 6.3 Amount Sign Rules

- **RNBELØ positive:** Customer is debited — invoice, interest note, collections charge.
- **RNBELØ negative:** Customer is credited — payment received, credit note, overpayment refund.
- **RNREST tracks the outstanding portion:** Starts equal to RNBELØ at creation. As payments are applied, RNREST is reduced. When RNREST = 0, the posting is fully settled and RNSTAT is set to 'L'.
- **Partial payments:** RNREST can be any value between 0 and RNBELØ. A posting with RNREST > 0 remains at RNSTAT = 'O' (open).

### 6.4 KID Generation and Structure

KID (Kunde IDentifikasjonsnummer) is the Norwegian standard payment reference number printed on invoices. In RKTRPF:

- **RNKIDI** (Alpha 25): Full KID string as printed on the invoice.
- **RNKIDR** (Numeric 8,0): Numeric KID reference number — key into AKIDPF for bank file matching.
- **RNKSIF** (Numeric 1,0): Check digit calculated using MOD10 or MOD11 algorithm.
- **RNKID2** (Alpha 25): Second KID number (for invoices that require two payment references).

The same KID fields appear in REMSPF (RTKIDI, RTKIDR, RTKSIF, RTKID2) for the GL registration side.

The AKIDPF table (key: AKFIRM + AKKIDR) links a KID reference number back to the invoice, enabling automatic matching when OCR bank files arrive with KID numbers.

### 6.5 Accounting Period Assignment

When a transaction is posted:

1. RNRAAR is set to the four-digit fiscal year.
2. RNRPER is set to the accounting period (1–13).
3. RNBDAT is the document date.
4. RNFDAT is calculated as RNBDAT + payment terms days (from RKBETB via RA03PF).

**Special handling for period 00 and 14:** In legacy RPG source (RE603R), the balance matrix reads period 00 as the year-opening balance and period 14 as year-end totals. These periods do not appear in RKTRPF; they are used only in the GL balance matrix (RHSAPF).

---

## 7. AR Ledger Rules

### 7.1 Document Creation Rules

When an invoice is generated by the sales/invoicing system, a corresponding RKTRPF record is written with:

| Field | Value Set |
|-------|-----------|
| RNFIRM | Company from LDA l_firm |
| RNKUND | Customer number (if RKFAKU is set, the invoice customer number is used) |
| RNRAAR | Current fiscal year |
| RNRPER | Current accounting period |
| RNBDAT | Invoice date |
| RNBILN | Next invoice number from RA4SUF counter in RAA4PF |
| RNBILK | Document type code (e.g., 10 for Faktura) |
| RNBELØ | Invoice amount (positive) |
| RNREST | Same as RNBELØ at creation |
| RNFDAT | Invoice date + payment terms days |
| RNSTAT | 'O' (open) |
| RNKIDR | KID reference number assigned from AKIDPF sequence |

Simultaneously, RKMEPF.RKMEMA is incremented by the invoice amount — this is the real-time credit exposure update.

### 7.2 RNSTAT Status Lifecycle

```
Created → RNSTAT = 'O' (Open)
                    │
         Payment applied, RNREST decremented
                    │
         RNREST = 0 → RNSTAT = 'L' (Settled/Lukket)
                    │
         Reversal or deletion → RNSTAT = 'D' (Deleted)
```

- **'O' (Open):** Invoice is outstanding. RNREST > 0. Posting appears in aging reports. RKMEMA includes this amount.
- **'L' (Settled):** Fully paid. RNREST = 0. Does not appear in open item lists or aging. RKMEMA has been reduced by the payment.
- **'D' (Deleted):** Record has been voided/reversed. A compensating posting exists. Does not affect open item balances.

### 7.3 Payment Allocation

When a payment arrives and is matched to an open invoice:

1. The RKTRPF record for the invoice is read and locked.
2. RNREST is reduced by the payment amount.
3. If RNREST reaches 0: RNSTAT is set to 'L'; RNODAT (settlement date) is set to today.
4. RKMEPF.RKMEMA is reduced by the payment amount (real-time credit exposure update).
5. RKUNPF.RKTDAT (last transaction date) is updated.

### 7.4 Three-Way Balance Reconciliation

At any point in time, three measures of customer balance should be consistent:

| Measure | Calculation | Table |
|---------|------------|-------|
| Open item sum | Sum of RNREST where RNSTAT = 'O' | RKTRPF |
| Memo balance | RKMEMA | RKMEPF |
| Sales YTD | RKSALD | RKUNPF |

**RKMEMA must equal the sum of RNREST for all open (RNSTAT='O') postings for this customer.** If they diverge, there has been a posting without a corresponding memo balance update — this is a data integrity error.

**RKSALD is not equal to the open item sum** — RKSALD accumulates all invoices regardless of payment status, so RKSALD >= sum(RNREST where RNSTAT='O') for a customer with no credits.

### 7.5 Interest Calculation Using RA6 Configuration

Interest on overdue invoices is governed by the RAA6PF configuration record:

| RA6 Field | Meaning |
|-----------|---------|
| RA6PPR | Annual interest rate (e.g., 1200 = 12.00% p.a.) |
| RA6PGR | Minimum total interest amount before an interest note is issued |
| RA6PFG | Minimum interest per invoice line |
| RA6PIB | 'J' = charge interest only on overdue portion (RNREST, not RNBELØ) |
| RA6POR | 'J' = post interest charges to GL accounts automatically |
| RA6PBT | Payment terms code applied to the interest note (RNBILK=40) |
| RA6RPU | 'J' = calculate and attach interest at dunning time |

**Customer-level override:** RKUNPF.RKRENK controls interest eligibility:
- 0 = Normal (follow RA6 settings)
- 1 = No interest charged
- 2 = Level 2 interest rate
- 3 = Level 3 interest rate

**Per-posting override:** RKTRPF.RNRENK tracks interest status per posting: 0=pending, 1=charged, 3=excluded from interest calculation, 5=interest waived.

---

## 8. Payment Processing Rules

### 8.1 Manual Payment Entry via REMSPF

Manual payments (cash, cheque, bank transfer not via OCR) are entered as REMSPF (GL registration) records:

1. Operator opens a new bundle (RTBUNT = next bundle number from RA4 sequence).
2. One REMSPF header record (RTREID = 'H') captures the bundle totals.
3. One REMSPF line record (RTREID = 'K' for customer) captures:
   - RTDKTO = bank/cash account (debit — money received into bank)
   - RTCKTO = AR sub-ledger account (credit — reducing what customer owes)
   - RTBELØ = payment amount
   - RTREFN = invoice number being paid
   - RTKIDR = KID number from the payment if available
4. The journal posting run reads REMSPF records, matches them to open RKTRPF postings, and updates RNREST and RNSTAT.
5. RTAUTX is set to 'K' (crossed/settled) on the REMSPF line after the posting is processed.

### 8.2 Remittance/OCR KID Matching Process

Norwegian banks transmit OCR (Optical Character Recognition) remittance files containing KID numbers paid by customers. The matching process operates in four steps:

**Step 1 — Receive bank file:** The bank OCR file arrives and is staged in an input file. Each record contains: KID number, amount, bank value date.

**Step 2 — KID lookup in AKIDPF:** For each KID in the bank file, the system chains to AKIDPF using AKKIDR (KID reference number). AKIDPF returns the associated invoice details: customer number, company, year.

**Step 3 — Match against open RKTRPF:** Using the customer and invoice details from AKIDPF, the system locates the open RKTRPF posting (RNSTAT = 'O', RNKIDR = AKKIDR). Amount in the bank file is compared to RNREST.

**Step 4 — Apply payment:** If the KID amount matches RNREST exactly, RNSTAT is set to 'L' and RKMEMA is reduced. If the bank amount differs from RNREST, the system creates either a partial settlement (RNREST reduced but remains > 0) or a credit note posting for the overpayment.

The RE630R suite (RE630R parameter collection, RE631R processing, RE632R label output, RE633R user authentication) handles the archive routing of the resulting accounting entries to the Nexstep Archive document management system.

### 8.3 Overpayment Credit Posting

When a payment exceeds the invoice amount:

1. The open invoice RKTRPF record is settled: RNREST = 0, RNSTAT = 'L'.
2. A new RKTRPF record is created for the credit balance: RNBILK = credit note code, RNBELØ = negative overpayment amount, RNSTAT = 'O'.
3. RKMEMA is decremented by the invoice amount, then incremented by the credit amount — net effect is that the credit balance increases RKMEMA.
4. The credit posting remains open until it is applied to a future invoice or refunded.

### 8.4 Write-Off Rules

Small balances below a configured threshold are written off via a specific document type code (typically a write-off variant of the credit note code). The write-off posts to a write-off expense GL account via REMSPF rather than to cash. RNSTAT is set to 'L' after the write-off posting.

### 8.5 Credit Note Lifecycle

1. Credit note is created with RNBILK = 30 (KR), RNBELØ = negative amount, RNSTAT = 'O'.
2. RKMEMA is incremented by the credit note amount (reduces the customer's net balance — a credit note reduces what they owe).
3. The credit note is applied against open invoices: RNREST of both the invoice and the credit note are reduced simultaneously.
4. When RNREST = 0 on the credit note: RNSTAT = 'L'.
5. If the credit note is not applied within a defined period, it may trigger a refund cheque process.

---

## 9. AR Aging and Collections Rules

### 9.1 Aging Bucket Boundaries

Aging is calculated by comparing today's date against RKTRPF.RNFDAT (due date) for each open posting. Bucket boundaries are configured in RAA3PF customer aging fields:

| Configuration Field | Meaning |
|--------------------|---------|
| RA3KA1 | Days: boundary between "current" and bucket 1 |
| RA3KA2 | Days: boundary between bucket 1 and bucket 2 |
| RA3KA3 | Days: boundary between bucket 2 and bucket 3 |
| RA3KA4 | Days: boundary between bucket 3 and bucket 4 (oldest) |

**Real-world example** with RA3KA1=30, RA3KA2=60, RA3KA3=90, RA3KA4=120:

| Bucket | Condition | Label |
|--------|-----------|-------|
| Current | RNFDAT >= today | Not yet due |
| Bucket 1 | 1–30 days past RNFDAT | 1–30 days overdue |
| Bucket 2 | 31–60 days past RNFDAT | 31–60 days overdue |
| Bucket 3 | 61–90 days past RNFDAT | 61–90 days overdue |
| Bucket 4 | 91–120 days past RNFDAT | 91–120 days overdue |
| Beyond | > 120 days past RNFDAT | Oldest — automatic collections trigger |

A customer with invoices in bucket 3 and beyond would typically be flagged for dunning. A customer with any amount in "beyond" bucket is a candidate for RKINKK = 'J' (collections referral).

### 9.2 Dunning Configuration

Dunning levels are configured in RAA5PF. Each dunning level (RA5RPU = 01, 02, 03...) has its own record:

| Field | Meaning |
|-------|---------|
| RA5RPU | Dunning level number |
| RA5RGB | 'J' = add a dunning fee to this level |
| RA5GEB | Dunning fee amount in NOK |
| RA5PBT | Payment terms code for the dunning fee invoice |
| RA5BFO | Print form number for the dunning notice |
| RA5RTX | Text paragraph number to include on the notice |
| RA5KPR | Contact person name (30 chars) |
| RA5TFN | Contact person phone |
| RA5EML | Contact person email |
| RA5BFI | 'J' = print company information on the notice |

**Level increment rule:** Each time a dunning run processes a posting, RKTRPF.RNANTP is incremented by 1. The current dunning level applied to a posting equals RNANTP. Level 1 = first reminder (mild tone); level 3+ = formal legal notice tone.

**Dispute exclusion:** If RKTRPF.RNREKL = 'J' (posting is flagged as disputed), it is excluded from dunning runs at all levels. The posting will not receive a dunning notice until RNREKL is cleared.

### 9.3 What Updates When a Dunning Notice Is Sent

| Update | Field | Table |
|--------|-------|-------|
| Increment dunning count | RNANTP += 1 | RKTRPF |
| Record dunning date on posting | RNPDAT = today | RKTRPF |
| Record dunning date on customer | RKPDAT = today | RKUNPF |
| Set customer dunning flag | RKPURR = 1 | RKUNPF |
| If fee added, new posting created | RNBILK = fee code, RNBELØ = RA5GEB | RKTRPF |

### 9.4 Collections Escalation via RKINKK

When a customer is referred to an external collections agency:

1. RKUNPF.RKINKK is set to 'J'.
2. RKUNPF.RKIDAT is set to the referral date.
3. Each open posting being referred has RKTRPF.RNINKO set to 'B' (sent to collections).
4. When the collections agency confirms receipt, RNINKO is updated to 'J'.
5. RKTRPF.RNIDAT is set to the collections referral date on each posting.
6. Collections postings are excluded from normal dunning runs (RNINKO <> ' ' filters them out).

### 9.5 Statement Generation Parameters via RK620R

RK620R collects the following parameters before printing the customer register or account statements:

| Parameter | Screen Field | Validation |
|-----------|-------------|------------|
| Customer range | From/to RKKUND | From must not exceed To |
| Department range | From/to RKAVDE | From must not exceed To |
| Category range | From/to RKKATG | From must not exceed To |
| Seller range | From/to RKSLGR | From must not exceed To |
| District range | From/to RKDIST | From must not exceed To |
| Print type | a2ptyp | Must be 'E' (email) or 'F' (print/fysisk) |
| Sort type | Sort flag | Must be 'J' (yes, sort by name) or blank (sort by customer number) |
| Sales/discount/salesperson | a2psal, a2prab, a2ppsl | Only one of these three flags may be set |
| "Older than" date | a2edat | Must be a valid date; defaults to today if blank |

**User department restriction:** If the logged-in user's ABBAVD = 'J', RK620R locks the department-from and department-to fields to the user's own department number. The fields are protected (display only) and cannot be changed. This ensures finance clerks see only their department's customers.

---

## 10. Customer Account Status and Workflow Rules

### 10.1 Five Lifecycle States

| State | RKSPRK | RKKRES | RKPASS | What the Customer Can Do |
|-------|--------|--------|--------|--------------------------|
| Active | ' ' | 0 | ' ' | Full ordering and AR activity |
| Order Blocked | 'J' | any | ' ' | No new orders; existing AR can be settled |
| Credit Locked | ' ' | 2 | ' ' | Orders blocked; AR activity continues |
| Fully Blocked | 'J' | 3 | ' ' | No orders, no new AR postings |
| Passive | any | any | 'J' | Excluded from normal searches; can still have open AR |

### 10.2 Passive Customer Rules (RKPASS)

**Field:** RKUNPF.RKPASS — Alpha 1, ' '=active, 'J'=passive.

**What is checked in RK500R (customer inquiry):** Indicator `*in55` controls visibility of passive customers. The F5 key toggles *in55:
- *in55 = OFF (default): passive customers (RKPASS = 'J') are skipped during subfile population.
- *in55 = ON: all customers including passive are shown.

**What is checked in RK100R:** Same F5 toggle. By default, passive customers are hidden from the maintenance subfile. An operator who needs to access a passive customer must first toggle *in55.

**Passive customers and AR:** Being passive does not settle or remove outstanding AR. Open RKTRPF postings for a passive customer remain open and continue to appear in aging reports. The passive flag only affects customer lookup visibility.

**Typical use:** A customer who has permanently stopped trading but has a small residual balance awaiting write-off is set to passive. This prevents new orders while the balance is being closed.

### 10.3 Modification Restrictions on Key Fields

| Field | Restriction | Reason |
|-------|------------|--------|
| RKUNPF.RKFAKU | Cannot be changed if the customer has unsettled RKTRPF postings with RNSTAT='O' | Changing the invoice customer mid-stream would split an account's AR between two invoice customers |
| RKUNPF.RKKUND | Customer number cannot be edited in place; use RK108R (move/merge) | The customer number is the primary key; changing it in place would orphan all RKTRPF, AKIDPF, SOHEPF records |
| RKUNPF.RKBETB | Can be changed at any time | Changes affect only future invoices; existing RKTRPF.RNFDAT values are not recalculated |
| RKUNPF.RKGRNS | Can be changed immediately | Takes effect on the next sales order credit check |
| RKEMAL | Explicitly blanked when a customer is copied (RK100R K1WIN) | The copied customer is a new entity; the source email must not receive documents intended for the copy |

### 10.4 Year-End Processing

At year-end, the following sequence is executed for each company:

1. **Copy RKSALD to RKSAIF:** RKUNPF.RKSAIF = RKUNPF.RKSALD. This preserves the prior-year sales figure.
2. **Reset RKSALD:** RKUNPF.RKSALD = 0. The new fiscal year starts with a clean sales accumulator.
3. **Reset sequence counters in RAA4PF:** RA4SUF (outgoing invoice number), RA4SIF (incoming invoice number), RA4KNR (last customer number allocated), and RA4SNO (interest note number) are reset to their starting values for the new year.
4. **Open period 1 of the new year:** The accounting calendar in RAA1PF is updated to open period 1 of the new fiscal year.

**Note:** Open RKTRPF postings (RNSTAT='O') are **not** closed at year-end. They carry forward with their original RNRAAR/RNRPER values intact. AR aging continues to work correctly because it uses RNFDAT (due date), not RNRAAR/RNRPER.

---

## 11. Integration Rules

### 11.1 External Customer Import via RE110R

RE110R reads from RE0KPF (an external customer data staging file) and updates or creates records in RKUNPF and RKMEPF. Seven rules govern every imported row:

| Rule | What Is Checked | What Happens When |
|------|----------------|-------------------|
| 1. Company match | Source RKFIRM must equal LDA l_firm | Row is skipped if RKFIRM ≠ l_firm |
| 2. Number range | Source RKKUND must satisfy: > RA1HKT AND <= RA1HRS | Row is skipped; number outside company's allocated range |
| 3. Non-zero overwrite | Each field in RKUNPF is updated only if the incoming value is non-blank (for alpha) or non-zero (for numeric) | Existing values are preserved when the incoming field is blank/zero; prevents wiping populated fields with empty import data |
| 4. New customer creation | If CHAIN on RKUNPF fails (customer not found) | A new RKUNPF record is written with all incoming fields |
| 5. Balance field initialisation | RKSALD, RKSAIF, RKKJØP, RKKJØF set to zero for new customers | Balance figures from the external system are not imported; they will be rebuilt from actual postings |
| 6. Memo balance initialisation | If RKMEPF record does not exist for this RKFIRM+RKKUND | A new RKMEPF record is written with RKMEMS=0, RKMEMA=0, stamped with current date/time/user |
| 7. Alpha search register | Always executed after update or create | RK750R is called with RKFIRM and RKKUND to rebuild RKSOPF.RKSTEK |

**Why rule 3 (non-zero overwrite) matters:** If the external system sends a customer file monthly but omits email addresses for customers who did not change, rule 3 ensures that RKEMAL populated in a prior import run is not overwritten with blank.

### 11.2 External Posting Parameters via RE120R

RE120R collects and validates fiscal year and accounting period for external postings (integrations that push AR transactions directly into the system):

- Date defaults to system date if blank; validated as a legal date.
- Year cannot exceed the current system year.
- Period must be 1–13.
- If current month is January, default period = 12 and default year = current year − 1 (prior December rule).

### 11.3 Sales Order Credit Check Flow

When a sales order is saved or released:

1. Read RKUNPF for the customer (RKFIRM + RKKUND).
2. **Check RKSPRK:** If 'J' → order is immediately blocked; stop.
3. **Check RKKRES:** If 2 or 3 → order is blocked; stop.
4. **Read RKMEPF:** Get RKMEMA (current real-time credit exposure).
5. **Compare:** RKMEMA + new order value vs. RKGRNS (credit limit).
6. **Apply RKKRES threshold:**
   - RKKRES = 0: no check performed (or check is informational only).
   - RKKRES = 1: if over limit, display warning; operator can override and proceed.
   - RKKRES = 2: if over limit, hard block; order cannot be saved until credit is released.
   - RKKRES = 3: all orders blocked regardless of balance.

### 11.4 GL Posting Split — AR Side vs. Transaction Side

Every AR event produces two posting lines in REMSPF (debit + credit), controlled by RTREID:

| Scenario | RTDKTO (Debit) | RTCKTO (Credit) |
|----------|---------------|----------------|
| Invoice posted | AR sub-ledger account (receivable) | Revenue account |
| Payment received | Bank/cash account | AR sub-ledger account |
| Credit note | Revenue account | AR sub-ledger account |
| Interest note | AR sub-ledger account | Interest income account |

The AR sub-ledger account is derived from RAIKTO in RA09PF (the salesperson's AR account code) combined with the customer's department (RKAVDE). This allows AR to be tracked by division.

### 11.5 KID/Bank File Matching via AKIDPF

When an OCR remittance file arrives from the bank:

1. Each KID number in the file is looked up in AKIDPF by AKKIDR.
2. AKIDPF returns AKKSIF (check digit) for validation; a MOD10 or MOD11 check is performed. If the check digit fails, the payment is routed to an exception queue.
3. On successful KID lookup, the system reads RKTRPF using RNKIDR = AKKIDR to find the open posting.
4. Payment is applied: RNREST decremented, RNSTAT set to 'L' if fully settled.
5. RKMEPF.RKMEMA is decremented by the payment amount.

### 11.6 Email/PDF Delivery via RKEMAL

**Field:** RKUNPF.RKEMAL — Alpha 50.

- If RKEMAL is non-blank, invoices and statements are delivered electronically to that address.
- If RKEMAL is blank, the document is printed and posted physically.
- RKEMAL is explicitly blanked when a customer is copied in RK100R (the `xk1win` copy subroutine includes this reset), ensuring the copy starts without an inherited email address.
- RKEMAL is passed to the invoicing and statement programs via customer master reads at print time.

### 11.7 GL Reporting Programs — Scope Note

The following programs (RE600R, RE601R, RE602R, RE603R, RE609R, RE613R, RE615R, RE621R, RE623R) are **GL reporting programs only** — they read from GL balance matrices (RHSAPF, RE01PF, RE02PF, RE03PF) and have no interaction with AR tables (RKTRPF, RKMEPF, RKUNPF). They export trial balance and income statement data to external systems (Trioplan, Akelius, Finale, Maestro). They are listed in the program inventory for completeness but do not implement AR business rules.

---

## 12. Field Reference

### 12.1 RKUNPF — Complete Field Reference

Key: RKFIRM (3,0) + RKKUND (6,0). Source: RFRF.MBR lines 2643–3097 / RKUNPF.MBR.

| Field | Type | Len | Norwegian Label | English Description |
|-------|------|-----|----------------|---------------------|
| RKFIRM | Num | 3,0 | Firma | Company number |
| RKKUND | Num | 6,0 | Kundenummer | Customer number |
| RKNAVN | Alpha | 30 | Kundenavn | Customer name — mandatory |
| RKALFA | Alpha | 18 | Alfabetisk søkefelt | Alpha search key — auto from RKNAVN |
| RKGATE | Alpha | 30 | Gateadresse | Street address |
| RKADR2 | Alpha | 30 | Mellomadresse | Address line 2 |
| RKPONR | Num | 4,0 | Postnr. | Postal code |
| RKSTED | Alpha | 30 | Poststed | City |
| RKBGAT | Alpha | 30 | Besøksgate | Visit street |
| RKBADR | Alpha | 30 | Besøksadresse | Visit address line 2 |
| RKBPNR | Num | 4,0 | Besøkspostnr. | Visit postal code |
| RKKOMN | Num | 4,0 | Kommunenr. | Municipality code |
| RKLAND | Alpha | 2 | Landkode | Country code (ISO 2-letter) |
| RKUPNR | Alpha | 9 | Utenlandsk Postnr. | Foreign postal number |
| RKFTNR | Num | 9,0 | Foretaksnummer | Norwegian organisation number |
| RKPENR | Num | 11,0 | Personnr. | Personal identity number |
| RKPERS | Alpha | 30 | Kontaktperson | Contact person name |
| RKEMAL | Alpha | 50 | E-mail adresse | Email — electronic document delivery |
| RKWEBA | Alpha | 40 | Web adresse | Web address |
| RKTLFN | Alpha | 11 | Telefonnummer | Phone number |
| RKTLFX | Alpha | 11 | Telefaxsnummer | Fax number |
| RKMOBN | Alpha | 11 | Mobil-nr. | Mobile number |
| RKMOBO | Alpha | 2 | Mobiloperatør | Mobile operator code |
| RKBGIR | Num | 11,0 | Bankgiro | Bank giro number |
| RKFAKU | Num | 8,0 | Faktura kundenr. | Invoice customer — billing consolidation |
| RKFACN | Num | 8,0 | Factoring-nummer | Factoring reference number |
| RKKOKU | Num | 6,0 | Konsern kundenr. | Group/consortium customer number |
| RKEDIK | Alpha | 1 | EDI-Kode | EDI code: ' '=none, 'A'=EDIFACT, 'E'=EAN |
| RKEANK | Num | 13,0 | EAN-Kunde | EAN customer number |
| RKEANF | Num | 13,0 | EAN-fakt.kunde | EAN invoice customer number |
| RKAVDE | Num | 4,0 | Avdeling | Department |
| RKKJØR | Num | 2,0 | Kjørerute | Delivery route |
| RKKJNR | Num | 5,0 | Rekkefølge innen rute | Sequence within route |
| RKKATG | Num | 4,0 | Kundekategori | Customer category |
| RKRABK | Num | 3,0 | Rabatt-kategori | Discount category |
| RKLAGR | Num | 2,0 | Lager | Default warehouse |
| RKSLGR | Num | 4,0 | Selger | Salesperson — must exist in RA09PF |
| RKDIST | Num | 4,0 | Distrikt | District |
| RKBETB | Num | 2,0 | Betalingsbetingelse | Payment terms code |
| RKBETM | Num | 1,0 | Betalingsmåte | Payment method code |
| RKSALD | Num | 11,2 | Saldo | Sales year-to-date; reset at year-end |
| RKSAIF | Num | 11,2 | Saldo i fjor | Prior year sales (copy of last year's RKSALD) |
| RKKJØP | Num | 11,2 | Kjøp i år | Purchases this year |
| RKKJØF | Num | 11,2 | Kjøp i fjor | Purchases last year |
| RKKJAL | Num | 11,2 | Kjøp i år alm. | Almenning (cooperative) purchases this year |
| RKGRAL | Num | 9,0 | Beløpgrense alm. | Almenning purchase limit |
| RKKTID | Num | 3,0 | Kredittid | Credit days |
| RKTDAT | Date | — | Siste trans-dato | Last transaction date |
| RKVALK | Alpha | 3 | Valutakode | Default currency code |
| RKSPRK | Alpha | 1 | Sperrekode | Block code: ' '=active, 'J'=blocked |
| RKMKOD | Num | 1,0 | MVA-beregning | VAT: 0=standard, 1=exempt, 2=export |
| RKLADR | Num | 1,0 | Leveringsadresse | Delivery address: 0=postal, 1=visit |
| RKLBET | Num | 2,0 | Leveringsbetingelse | Delivery terms code |
| RKFORM | Num | 2,0 | Forsendelsesmåte | Delivery method code |
| RKSPED | Num | 3,0 | Speditørnummer | Freight forwarder number |
| RKORAB | Num | 5,2 | Ordrerabatt | Order discount percentage |
| RKGRNS | Num | 5,0 | Kreditgrense | Credit limit |
| RKKGRK | Alpha | 1 | Kreditgr.kode | Credit limit type code |
| RKFRIE | Alpha | 5 | Frie koder | Free classification codes |
| RKRENK | Num | 1,0 | Rentekode | Interest code: 0=normal, 1=none, 2=lvl2, 3=lvl3 |
| RKRNEG | Alpha | 1 | Rentenota negativ | Interest note for negative balance |
| RKRDAT | Date | — | Siste rente-dato | Last interest calculation date |
| RKPURR | Num | 1,0 | Purrekode | Dunning active: 0=no, 1=yes |
| RKPDAT | Date | — | Siste purre-dato | Last dunning notice date |
| RKPNEG | Alpha | 1 | Purring negativ | Dun for negative balance |
| RKINKK | Alpha | 1 | Inkassokode | Collections: ' '=normal, 'J'=referred |
| RKIDAT | Date | — | Siste inkasso-dato | Collections referral date |
| RKFACK | Num | 1,0 | Factoringkode | Factoring: 0=no, 1=yes |
| RKSAMM | Alpha | 1 | Samlefakturamerke | Summary invoice marker |
| RKORDB | Num | 1,0 | Ordrebekreftelse | Order confirmation: 0=no, 1=yes |
| RKKRES | Num | 1,0 | Kredittsperre | Credit block: 0=none,1=warn,2=block,3=lock |
| RKSPAV | Num | 1,0 | Spesialavtale | Special pricing agreement flag |
| RKPRPA | Num | 1,0 | Pris/Pakkseddel | Price on packing slip option |
| RKNETO | Num | 1,0 | Pris på faktura | Show price on invoice |
| RKSAMF | Num | 1,0 | Samlefaktura | Consolidated invoice option (0–4) |
| RKREKV | Num | 1,0 | Krav til rekvisisjon | Requisition required: 0=no, 1=yes |
| RKFAGB | Num | 1,0 | Fakturagebyr | Invoice fee: 0=no, 1=yes |
| RKEKGB | Num | 1,0 | Ekspedisjonsgebyr | Expedition fee: 0=no, 1=yes |
| RKFRGB | Num | 1,0 | Fraktgebyr | Freight fee: 0=no, 1=yes |
| RKRASP | Num | 1,0 | Rabatt-sperre | Discount block: 0=no, 1=yes |
| RKFRSK | Alpha | 10 | Frie statistikk-kode | Free statistics code |
| RKPBOK | Num | 1,0 | Fakt. etter prisbok | Invoice by price book: 0=no, 1=yes |
| RKPTAB | Num | 3,0 | Påslagstabell | Markup table 1 |
| RKPTA2 | Num | 3,0 | Påslagstab 2 | Markup table 2 |
| RKSPFA | Num | 1,0 | Spesifiser fakturering | Invoice detail: 0=standard,1=detail,2=summary |
| RKSPL1 | Alpha | 1 | Spesifiser lønn 1 | Payroll spec flag 1 |
| RKSPL2 | Alpha | 1 | Spesifiser lønn 2 | Payroll spec flag 2 |
| RKSPL3 | Alpha | 1 | Spesifiser lønn 3 | Payroll spec flag 3 |
| RKSPL4 | Alpha | 1 | Spesifiser lønn 4 | Payroll spec flag 4 |
| RKSPL5 | Alpha | 1 | Spesifiser lønn 5 | Payroll spec flag 5 |
| RKSEKV | Num | 2,0 | Fakturasekvens | Invoice batch interval: 0,1,7,14,30 days |
| RKFAGR | Num | 4,0 | Fakturagrense | Minimum invoice amount threshold |
| RKAFDA | Date | — | Gyldig dato-fra AG | AutoGiro valid from date |
| RKATDA | Date | — | Gyldig dato-til AG | AutoGiro valid to date |
| RKSENT | Num | 1,0 | Fullm. sendt AG | AG mandate sent: 0=no, 1=yes |
| RKTILB | Num | 8,0 | Tilbudsnummer | Quote/offer number |
| RKOCOP | Num | 1,0 | Ordre - ant.kopi | Order confirmation copies |
| RKPCOP | Num | 1,0 | Pakk. - ant.kopi | Packing slip copies |
| RKFCOP | Num | 1,0 | Fakt. - ant.kopi | Invoice copies |
| RKPASS | Alpha | 1 | Passiv kunde | Passive: ' '=active, 'J'=passive |
| RKEBIZ | Num | 1,0 | E_handels kunde | E-commerce: 0=no, 1=yes |
| RKLETY | Num | 1,0 | Leveringstype | Delivery type |
| RKFATY | Num | 2,0 | Fakturatype | Invoice type |
| RKKOTI | Num | 1,0 | Kontant kunde tilbud | Cash customer offer flag |
| RKNDAT | Date | — | Ny Dato | Record creation date |
| RKNTIM | Time | — | Ny Kl. | Record creation time |
| RKNSIG | Alpha | 10 | Ny av (User-ID) | Created by user |
| RKEDAT | Date | — | Endr.dato | Last modification date |
| RKETIM | Time | — | Endr.tid | Last modification time |
| RKESIG | Alpha | 10 | Endret av (User-ID) | Last modified by user |
| RKEPGM | Alpha | 10 | Endret av program | Last modified by program |

### 12.2 RKTRPF — Complete Field Reference

Key: RNFIRM + RNKUND + RNRAAR + RNRPER + RNBDAT + RNBILN. Field prefix: RN. Source: RFRF.MBR lines 3192–3339 / RKTRPF.MBR.

| Field | Type | Len | Norwegian Label | English Description |
|-------|------|-----|----------------|---------------------|
| RNFIRM | Num | 3,0 | Firma | Company number |
| RNKUND | Num | 6,0 | Kunde-nr. | Customer number |
| RNRAAR | Num | 4,0 | Regnskapsår | Accounting year (4 digits) |
| RNRPER | Num | 2,0 | Periode | Accounting period (1–13) |
| RNBDAT | Date | — | Bilags-dato | Document/voucher date |
| RNBILN | Num | 8,0 | Bilagsnummer | Voucher/document number |
| RNBILK | Num | 2,0 | Bilags-kode | Document type code (10=invoice etc.) |
| RNTEXT | Alpha | 20 | Bilagstekst | Document description text |
| RNBELØ | Num | 11,2 | Beløp | Original document amount |
| RNREST | Num | 11,2 | Restbeløp | Remaining/outstanding amount |
| RNVALK | Alpha | 3 | Valuta-kode | Currency code |
| RNVALB | Num | 11,2 | Valutabeløp | Original amount in foreign currency |
| RNVALR | Num | 11,2 | Valuta-rest | Remaining foreign currency amount |
| RNFDAT | Date | — | Forfalls-dato | Due date — basis for aging |
| RNJOUR | Num | 5,0 | Journalliste-nr. | Journal list number |
| RNSAMF | Num | 8,0 | Samlefaktura-nr. | Consolidated invoice number |
| RNSTAT | Alpha | 1 | Status | Status: 'O'=open, 'L'=settled, 'D'=deleted |
| RNREFN | Num | 8,0 | Referansenummer | Cross-reference document number |
| RNVIRK | Alpha | 2 | Virksomhetskode | Business/activity code |
| RNAVDE | Num | 4,0 | Avdeling | Department |
| RNKONT | Num | 4,0 | Konto | GL account |
| RNSPES | Num | 4,0 | Bærer | Cost carrier |
| RNSLGR | Num | 4,0 | Selger | Salesperson |
| RNREKL | Alpha | 1 | Reklamasjonskode | Dispute flag: ' '=normal, 'J'=disputed |
| RNRENK | Num | 1,0 | Renteberegnet | Interest status: 0=pending,1=charged,3=excl,5=waived |
| RNRDAT | Date | — | Siste rente-dato | Last interest calculation date |
| RNANTP | Num | 1,0 | Antall purringer | Number of dunning notices sent |
| RNPDAT | Date | — | Siste purre-dato | Last dunning notice date |
| RNINKO | Alpha | 1 | Inkassokode | Collections: ' '=none, 'B'=sent, 'J'=confirmed |
| RNIDAT | Date | — | Inkassodato | Collections referral date |
| RNODAT | Date | — | Oppgjørsdato | Settlement date |
| RNAUTO | Alpha | 1 | Autogiro kode | AutoGiro code |
| RNADAT | Date | — | Autogiro dato | AutoGiro date |
| RNFACK | Num | 1,0 | Ikke factoring | Factoring exclusion: 0=include, 1=exclude |
| RNFADA | Date | — | Overført dato | Factoring transfer date |
| RNTIST | Timestamp | — | Post. timestamp | Posting timestamp |
| RNKIDI | Alpha | 25 | Kundeident | Full KID identity string |
| RNKIDR | Num | 8,0 | Kidd referansenr | KID reference number (key to AKIDPF) |
| RNKSIF | Num | 1,0 | Kontroll-siffer | KID check digit (MOD10/MOD11) |
| RNKID2 | Alpha | 25 | Kiddnummer 2 | Second KID number |
| RNSPOR | Num | 10,0 | Spor.ref. | Trace/source reference |
| RNAVKR | Num | 10,0 | Avkr.ref. | Confirmation reference |
| RNDATE | Date | — | Endr.dato | Last modification date |
| RNOTIM | Time | — | Endr.tid | Last modification time |
| RNOUSR | Alpha | 10 | Endret av | Last modified by user |
| RNEDAT | Date | — | Oppr.dato | Record creation date |
| RNETIM | Time | — | Oppr.tid | Record creation time |
| RNEUSR | Alpha | 10 | Opprettet av | Created by user |

### 12.3 RKMEPF — Complete Field Reference

Key: RKMFIR + RKMKUN. Source: RFRF.MBR lines 3101–3177 / RKMEPF.MBR. Added in system version R6.28.

| Field | Type | Len | Norwegian Label | English Description |
|-------|------|-----|----------------|---------------------|
| RKMFIR | Num | 3,0 | Firma | Company number (key) |
| RKMKUN | Num | 6,0 | Kundenummer | Customer number (key) |
| RKMEMS | Num | 11,2 | Memosaldo | Standard memo balance — total open AR exposure |
| RKMEMA | Num | 11,2 | Memosaldo alm. | Almenning (cooperative) memo balance |
| RKMEDA | Date | — | Endr.dato | Last modification date (standard balance) |
| RKMETI | Time | — | Endr.tid | Last modification time (standard balance) |
| RKMEUS | Alpha | 10 | Endret av | Last modified by (standard balance) |
| RKMODA | Date | — | Endr.dato | Last modification date (Almenning balance) |
| RKMOTI | Time | — | Endr.tid | Last modification time (Almenning balance) |
