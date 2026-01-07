# RPG Program FO920R: Explanation and Onboarding Guide

## Overview

This program, `FO920R`, is used for the printing (and optionally email or archiving) of order confirmations ("Ordrebekreftelse"). It is a mature, business-critical report program with extensive modularity, integration with XML/Jasper (for PDF output), and legacy support for classic spool file printing. It is historically extended and actively maintained (see version comments).

---

## Structure

### 1. **Header & Definitions**

- **Program Title**: Marked with `H/TITLE`.
- **Special Keywords**:
  - Datedit/Decedit: Number/date formatting.
  - `option(*nodebugio)`: Optimizes I/O.
  - `DftActgrp(*No) BndDir('CGIDEV2/CGIDEV2')`: Indicates no default activation group and binds CGI routines (for web integration).
- **Copybooks**: `/copy hspecsbnd`, `/copy prototypeb`, `/copy usec` bring in external type/prototype specs.

---

### 2. **Change Log / Revision History**

Large comment block lists all enhancements, fixes, and changes, with version, date, author, and description. It's crucial for understanding business and technical context.

---

### 3. **File Definitions**

- **Logical/Physical Files**:  e.g., `FFPSOL1`, `FFSTSL1`, ..., `FFODTL1`, etc.
- **Printer File**: `ffO920p`.
- **Various data files**: Customers, orders, departments, deposit/prepayment records.
- **Files are renamed for local record formats** (e.g., `RENAME(FPSOPFR:FPSOL1R)`).

---

### 4. **Variable Definitions**

- **Arrays**: Used for handling multiple VAT rates, amounts, etc.
- **Keys/Fields**: Many "like()" declarations to inherit types.
- **Data Structures (DS)**: For Local Data Area (LDA), customer/mobile info, payment params, etc.
- **Parameters**: Passed in when program is called (for email, customer, etc.).
- **General Working Variables**: For report fields, calculations, temporary storage, etc.

---

### 5. **Main Logic Flow**

#### **A. Input Cycle**
- Standard RPG cycle processes records by key (`CASEQ`) and dispatches:
  - 'H' → Header/Order
  - 'V' → Detail/Item line
  - 'K' → Comment/Text line

---

#### **B. Processing Headers, Details, and Texts**

##### **Header (Order Head)**
1. **Initialize variables**.
2. **Fetch order header** (`FOHEL1R`) by key.
3. **Translate and format dates, order numbers, etc.**
4. **Lookup customer, handle alternate delivery addresses**.
5. **Check for external project/department/customer reference**.
6. **Determine VAT rates** and **payment conditions**.
7. **Fetch phone/fax/mobile info**, possibly override with order-specified numbers.
8. **Fetch company/department info** based on steering routine.
9. **Handle email parameters** if the output is email.
10. **Output header** either as spool file or XML/Jasper structure, depending on `b_pdfp`.

##### **Detail (Item Line)**
1. **Clear and setup variables**.
2. **Fetch order line** (`FODTL1R`), item master info (`VVARL1R`).
3. **Calculate discounts, net/gross prices**, special handling for multiple VAT rates.
4. **Check line types and skip/write as needed**.
5. **If printing, write using print format; if XML, call XML detail routines**.
6. **Handle length specifications if present**.

##### **Textline**
1. **Fetch text from `FOTXL1R`**.
2. **Output text line in print or XML format depending on mode**.

---

#### **C. Totals & Finalization**

- **Calculate report totals** (net, gross, VAT, discounts, etc.).
- **Output totals and possibly bruksrettsrabatt (usage right rebates)** in print and/or XML.
- **Output trailing text (if any)**.
- **If output is XML/Jasper** (PDF or archive), finish XML structure, possibly trigger preview or archiving, and log activity.

---

### 6. **Subroutines / Procedures**

- **sbbetb**: Fetch/payment terms text.
- **skr_leng**: Print length details for item lines (steel, pipes, etc.).
- **xovrfl**: Handle page overflow (print new heading).
- **xarray**: Array handling for VAT rates/amounts.
- **xml_head/xml_deta/xml_totaler**: Routines for assembling XML for Jasper.
- **xml_netto/xml_vat/xml_rabatt**: Output net, VAT, and discount lines into XML.
- **xml_brrab/xml_brrabskr**: Output bruksrettsrabatt in XML.
- **hnt_avde**: Fetch department info for output.
- **dann_kid, xml_forsku, list_forsku, upd_kid**: Prepayment/KID number handling (bank payment identification logic).

---

### 7. **Email, PDF, and Archive Handling**

- **Integrates with CGIDEV2 & Jasper**: For modern output as PDF, XML.
- **Output can go to email, archive, or preview**.
- **Flexible setup**: Determines at runtime (via LDA/parameters) whether to print, email, or archive.

---

### 8. **Initialization and Setup**

- **INZSR subroutine**: Handles all variable initialization, fetches payment info, sets up keys and defaults, and detects if output is email.
- **Entry Parameters** (from *ENTRY PLIST) set up runtime options like email flag, customer, order, seller, etc.

---

## Key Design Aspects for Developers

- **Extensible**: The code is heavily commented with change numbers and is built for business logic evolution.
- **Dual output support**: Classic spool + modern PDF/XML/Jasper.
- **Separation of concerns**: Report formatting, business logic, special calculations, and file handling are modularized.
- **Local Data Area (LDA)**: Used as a steering mechanism for runtime configuration (e.g., what info to print, output type).
- **Integration points**: External programs for email (FO798R, FM798R), KID generation (AK720R), VAT routines, etc.

---

## Common Business Scenarios Handled

- **Multiple VAT rates** per order/item line.
- **Conditional printing of discounts, project, and department info**.
- **Prepayment (deposit) handling** with KID generation for bank.
- **Flexible report output**: print, mail, XML, or archive.
- **Customer-specific price/rabatt rules**.
- **Automatic logging and audit trail of output actions**.

---

## Tips for Onboarding Developers

- **Understand the file structure and record formats**; each logical/physical file may map to a specific piece of the order process.
- **Focus on the subroutine structure**: Most business logic is encapsulated in BEGSR/ENDSR blocks.
- **When in doubt, check the comment history** for rationale on business rules.
- **For output logic, follow the `b_pdfp` flag**: if off, it's classic print; if on, it's XML/Jasper.
- **Localization/Formatting**: Code has many routines for formatting numbers, dates, and converting special characters for XML.
- **Testing**: Changes should be regression tested for both print and Jasper output, to avoid breaking either.

---

## Summary

This is a critical business report program, designed for robust and flexible output, supporting both legacy and modern workflows. Modification requires careful attention to business rules, output format consistency, and integration with external systems (mail, archive, prepayment, etc.). The extensive commenting and modularization should guide you as you start extending or troubleshooting the process.