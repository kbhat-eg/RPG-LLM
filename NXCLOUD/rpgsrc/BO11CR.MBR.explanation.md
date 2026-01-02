# RPG Program Explanation: ASSHOP - Customer Information Management (`BO11CR`)

## Overview

This RPG (Report Program Generator) program, `BO11CR`, is part of the `ASSHOP` system and is responsible for working with customer information, including entry, validation, lookup, maintenance, and order registration activities. The code is classical ILE RPG (mostly fixed format, with some free format in newer additions), interacting heavily with database files (physical and logical; DISK) and calling various subprograms for customer and order processing.

The program includes extensive change history and documentation in the header, showing iterative improvements, especially in credit/memo calculations, customer project handling, and integration with newer registers/systems.

---

## Major Sections

### 1. File Definitions

- **Input/Output Files (F-Specs):**
  - Multiple customer-related files are defined, usually with `RENAME` to allow for easier field access (`RKUNPFR`, `RKMEL1R`, etc.).
  - The `WORKSTN` file is a workstation device for user interaction (screen IO).
- **Array Definitions:**  
  - `a_mld1` is a one-element array, typically holding a notification/message.
- **Key Lists:**  
  - Several `KLIST` definitions for handling composite keys when accessing files.

---

### 2. Data Definitions (D-Specs)

- **Work Fields:**  
  - Numerous variables are defined as *like other fields* in files for consistent typing.
  - Special fields for dates, phone numbers, addresses, etc.
- **Data Structure (DS, UDS):**
  - Used for field overlays, holding record images or parameter lists.

- **Constants:**  
  - Lowercase and uppercase letter strings for text conversion.

---

### 3. Mainline Logic (C-Specs)

#### *Flow Overview:*
1. **Initialization & Entry Parameter Handling**
    - Receives program call parameters (customer, order, return fields).
    - Sets up file keys and initializes working fields.
2. **Order Header Lookup (`HNTORD` Subroutine)**
    - Tries to find an existing "bong"/order header and populates display if found.
3. **Screen Initialization**
    - Sets up and clears screen display fields for data entry.
4. **Main Screen Loop**
    - Handles main user interactions, such as:
      - Customer/project lookup
      - Order/customer maintenance functions (F keys)
      - Exit/return handling
5. **Validation**
    - Checks mandatory fields, date validity, special project or requisition requirements.
6. **Customer Data Fetch (`HNTKUN`)**
    - Looks up customer records in the main register, including both delivery and invoice customers, and handles fallback/linked customer numbers.
    - Loads credit/memo info, order discounts, extended agreement texts, etc.
7. **Credit Checking**
    - Verifies if the customer has exceeded their credit limit (with extended/overridden logic for certain customers via "utvidelse av memosaldo").
8. **Data Saving**
    - Updates or writes order headers with the latest customer/order info (`OPPORD`).
    - Converts fields to uppercase if needed.
9. **Subroutines:**
    - `SPKUN`: Customer register inquiry via subprogram.
    - `OPKUN`: New customer creation via subprogram.
    - `*INZSR`: Program initialization.

---

### 4. Notable Logic Details

#### a) **Extended Credit/Memo Handling (`u_kmem` and `w_memodagr`):**
- The program supports "extended memo balances" for certain customer groups (notably "Mestergruppen").
- If enabled (`u_kmem` = *on), it fetches special settings (number of days to include in memosaldo, percentage for credit extension) from a properties file or via service program call (`CO402R`).
- Memo balances are then calculated including VAT (mva).

#### b) **Flexible Customer Lookup and Linking**
- Supports both normal customers and delivery address customers (potentially linked to other customer numbers).
- Handles situations where a delivery customer or a customer project overrides core fields.

#### c) **Requisition Enforcement**
- If a customer requires a requisition number (and the field isn’t filled in), the system will warn and prevent progression.

#### d) **Screen Modes and Error Handling**
- Clear use of indicator fields (`*INxx`) to control error messages, field highlighting, etc.
- Main logic loops back to prior tags (`TAG`, `TAGA`, `TAGB`, etc.) depending on validation outcome.

#### e) **Subprogram Integration**
- Calls out to subprograms (`FL501R`, `FA930R`, `RK500R`, etc.) for modular logic like fetching/updating customer data, registering new customers, etc.

---

## Key Data Files and Their Use

| File        | Purpose                                   |
|-------------|-------------------------------------------|
| `RKUNPFR`   | Customer Master Register                  |
| `RKMEL1R`   | Customer Memo/Balance Register            |
| `FUSRPFR`   | OFAK User Register                        |
| `BSTSPFR`   | Store Status Codes                       |
| `BMHEPFR`   | Order Header / Bong-Header                |
| `FKPRPFR`   | Customer Delivery/Project Register        |

---

## Special Logic by Release (as per header comments)

- **R3.10+**: Extra fields such as mobile phone number (`C1MOBN`).
- **R3.11+**: Warn if customer project exists.
- **R3.31+**: Enforce requisition number if customer settings require it.
- **R3.32+**: Option to display discounts as percent or net.
- **R3.33+**: Extended agreement texts, from separate register.
- **5.xx+**: Integration with new fast customer creation program.
- **5.41+**: Expansion of district code.
- **6.20+**: Memo balance handling (memosaldo) enhancements.
- **6.30+**: Project overrides for customer info.
- **7.01/8.00**: Further extensions for memo/credit bounds, memo days, VAT-included memosaldo, use of new UTVIDET_MEMOSALDO property.

---

## Onboarding Recommendations

- **Understand the interplay** between customers, projects, orders, and memo balances.
- **Familiarize with external subprograms** (`FL501R`, `FA930R`, etc.) as business logic is distributed.
- **Pay attention to how indicators and tags/tags are used** for main process flow and error handling.
- **Changes in customer or order fields** propagate widely; test thoroughly when modifying field logic.
- **Properties/configuration logic** (reading from files or using CO402R) is crucial for advanced clients.

---

## In Summary

This program is a comprehensive, multifaceted entry and validation utility for customer and order header information, designed for high integration into a modularized, file-based business system. It represents classical RPG development with iterative enhancements and robust data validation, credit control, and integration with external modules—all heavily documented and managed with a focus on reliability and business rules enforcement.