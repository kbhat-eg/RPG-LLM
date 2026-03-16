# Business Logic for Customer and Accounts Receivable

This module covers customer master data maintenance, AR transaction browsing, open-item management, and environment configuration in the ASOKON and ASPGPL systems. The programs covered are RK100R (customer list/overview), RK101R (customer detail maintenance), RK110R (AR posting browse), RK120R (customer open items — factoring and interest), AR100R (AR environment/AMILPF maintenance), and AR700R (version number lookup utility). NXKORR overrides exist for RK100R and RK101R.

---

## Prerequisites / Master Data Requirements

1. **AR Module License Required for Posting Inquiry (RK100R)**
   - Logic: F8 (open the AR posting inquiry via RK110R) is blocked unless `FSTSPF.FAAK03 = 5`, indicating the ASOKON AR module is licensed.
   - File: FSTSPF
   - Field: FAAK03
   - Condition: `FAAK03 ≠ 5` → F8 key blocked (switch 7.07).

2. **Fixed/System Customers Cannot Be Deleted (RK100R)**
   - Logic: Customer numbers that are classified as system customers (e.g., internal or distribution customers) cannot be deleted. The classification is checked via BSTSPF (system constants file).
   - File: BSTSPF
   - Condition: Customer number matches a system-reserved entry → delete blocked (switch 7.08).

3. **Organisation Number (RKFTNR) — Mod11 Validation (RK101R)**
   - Logic: If RKFTNR (organisasjonsnummer) is entered, the Norwegian Mod11 checksum algorithm is applied. If the checksum fails, the save is blocked.
   - File: RKUNPF
   - Field: RKFTNR
   - Condition: Mod11 check fails → error indicator set, save blocked.

4. **Bank Account Number (RKBGIR) — Mod11 Validation (RK101R)**
   - Logic: If RKBGIR (bankgiro / bank account) is entered, the Mod11 checksum is applied. Failure blocks the save.
   - File: RKUNPF
   - Field: RKBGIR
   - Condition: Mod11 check fails → error indicator set, save blocked.

5. **Personal Number (RKPENR) — Mod11 Validation (RK101R)**
   - Logic: If RKPENR (personnummer) is entered, Mod11 validation is applied. Failure blocks the save.
   - File: RKUNPF
   - Field: RKPENR
   - Condition: Mod11 check fails → error indicator set, save blocked.

---

## Validation Rules

6. **Fakturatype 22 or 25 Requires Email Address (RK101R)**
   - Logic: If the invoice delivery type (RKFATY) is 22 (email invoice) or 25 (email + EHF combined), an email address must exist either on the customer record (RKEMAL) or on the primary contact. If neither is present, the save is blocked.
   - File: RKUNPF
   - Fields: RKFATY, RKEMAL
   - Condition: `RKFATY IN (22, 25) and RKEMAL = *blank and no_contact_email` → save blocked.

7. **Non-Paper Invoice Types Require Bank Account Number (RK101R)**
   - Logic: Invoice types other than 20 (paper/post) generally require RKBGIR. Exceptions apply for specific electronic types (21 = EHF, 22 = email, 24 = e-invoice, 25 = email+EHF, 30 = special) that have their own payment routing. For remaining types that don't include these exceptions, missing RKBGIR blocks the save.
   - File: RKUNPF
   - Fields: RKFATY, RKBGIR
   - Condition: `RKFATY ≠ 20 and not_in_exceptions and RKBGIR = 0` → save blocked.

8. **EAN Number Must Be Unique (RK101R)**
   - Logic: If RKEANK (EAN/GLN number) is entered, it is checked against all other customers in RKUNPF for uniqueness. A duplicate EAN number blocks the save.
   - File: RKUNPF
   - Field: RKEANK
   - Condition: Another customer already has the same RKEANK → save blocked.

9. **Postcode Validation (RK101R — switch 8.14)**
   - Logic: From version 8.14, postcode validation method was updated. The entered postcode is validated against the postcode register. Invalid postcodes block the save.
   - Condition: Postcode not found in register → save blocked.

---

## Configuration and Authorization Rules

10. **Show All vs. Active Customers Toggle (RK100R)**
    - Logic: Indicator `*in55` controls whether the list shows only active customers or all customers including expired ones. `*in56` toggles showing only "quick-created" (hurtigopprettet) customers. Both are user-controlled at runtime.
    - File: RKUNPF
    - Condition: Toggle indicators applied as RKUNPF filter.

11. **Fakturatype / Payment Terms Propagation (RK101R — switch 8.04)**
    - Logic: When RKFATY (invoice type), RKBETB (payment terms), or RKFGEB (invoice fee) is changed on the customer master, the change is propagated to:
      - FKPRPF (customer price/profile records)
      - FOHEPF (open orders — all open/unprocessed orders for this customer)
    - Files: FKPRPF, FOHEPF
    - Condition: Any change to these three fields triggers cascade update.

12. **Factoring Exclusion Flag (RK101R — switch 8.12)**
    - Logic: RKIIZY flag controls whether the customer is excluded from factoring. When set, this customer's invoices are not transmitted to the factoring company.
    - File: RKUNPF
    - Field: RKIIZY
    - Condition: `RKIIZY = 1` → customer excluded from factoring processing.

13. **Consolidated Invoicing Exclusion Flag (RK101R — switch 8.13)**
    - Logic: RKSAMF flag controls whether the customer participates in consolidated invoicing (samlefaktura). When cleared, the customer receives individual invoices even if the system default would consolidate.
    - File: RKUNPF
    - Field: RKSAMF
    - Condition: `RKSAMF = 0` → customer excluded from consolidated invoicing.

---

## Financial / Transactional Rules

14. **Credit Limit Check Referenced from RKMEMS (RK101R, referenced by FO100R)**
    - Logic: RKMEMS stores the customer's current outstanding balance including VAT (from the memo register RKMEPF). This is compared against RKGRAL (credit limit) in order programs. RK101R allows maintenance of both RKMEMS and RKGRAL on the customer master.
    - File: RKUNPF, RKMEPF
    - Fields: RKMEMS, RKGRAL
    - Condition: Maintained by RK101R; enforced by FO100R and other order programs.

15. **AR Transaction Browsing by Bill Year/Number or by Customer (RK110R)**
    - Logic: RK110R positions on RKTRPF either by bill year + bill number (direct lookup) or by customer number (sequential browse). Valg 2 on a transaction opens the update screen (xc2win). The program is primarily informational — no hard blocking logic beyond record-not-found handling.
    - File: RKTRPF (AR transaction register)
    - Fields: RKTAAR (year), RKTBILN (bill number), RKTKUND (customer)

16. **Factoring Extraction (RK120R — V5.40)**
    - Logic: RK120R displays open items from RKWBPF and RKTRPF. The factoring extraction function (V5.40) processes eligible open items for transmission to the factoring company. Items flagged with RKIIZY or belonging to excluded customers are not included.
    - Files: RKWBPF (open items), RKTRPF (AR transactions), RKUNPF (customer master)
    - Condition: Customer must not have RKIIZY set; item must be eligible.

17. **Interest Correction (RK120R — V5.60)**
    - Logic: The interest correction function (V5.60) allows adjustment of interest charges on open items. This is an interactive correction process.
    - File: RKWBPF, RKTRPF

18. **Project Links from RKLAPF (RK120R)**
    - Logic: Open items can be linked to project accounts. RKLAPF is read to display project linkage information alongside the open item.
    - File: RKLAPF (project links)

19. **Notes/Annotations via RF020R (RK120R)**
    - Logic: Pressing F2 from the open items screen launches RF020R to display or edit the customer's annotation sheet (notatark). This is an informational function with no blocking logic.
    - Program: RF020R

---

## Status and Lifecycle Rules

20. **AMILPF Environment Record Management (AR100R)**
    - Logic: AR100R provides simple CRUD operations for AMILPF (AR environment records). Records are identified by ARMILJ (environment code). If a record is not found on chain, create mode is activated. Beyond the standard "record not found" handling, there are no additional blocking conditions in AR100R.
    - File: AMILPF
    - Field: ARMILJ (environment key)

21. **Version Number Lookup (AR700R)**
    - Logic: AR700R is a utility called by other programs to retrieve the version number (ARVERS) for a given AR environment (ARMILJ). If the environment key is not found in AMILPF, ARVERS is returned as 0. This utility has no blocking conditions.
    - File: AMILPF
    - Fields: ARMILJ (input), ARVERS (output)
    - Condition: `not %found(AMILPF)` → returns ARVERS = 0.

22. **Customer Status Display Toggle (RK100R)**
    - Logic: The list defaults to showing active customers only. `*in84` (analogous to VV100R) or `*in55` provides a toggle to include expired/inactive customers. This is purely a display filter with no business blocking effect.

---

## Special Conditions (Program-Specific)

23. **RK101R: Fakturatype 21 (EHF Electronic Invoice)**
    - Logic: Fakturatype 21 (EHF — Electronic Invoice for Norway) has relaxed bank account requirements but requires that the customer has a valid organisation number (RKFTNR) for EHF routing. The organisation number must pass Mod11 validation.
    - Condition: `RKFATY = 21 and RKFTNR fails Mod11` → save blocked.

24. **RK101R: Fakturatype 24 (E-Invoice)**
    - Logic: Fakturatype 24 requires the EAN/GLN number (RKEANK) to be populated for EDI routing. Without it, invoices cannot be transmitted electronically.
    - Condition: `RKFATY = 24 and RKEANK = 0` → save blocked or warning issued.

25. **RK100R: NXKORR Override**
    - Logic: The NXKORR library contains a customer-specific override of RK100R. This override may apply additional filtering or display modifications specific to the installation. The standard base-program logic applies unless superseded by the override.

26. **RK101R: NXKORR Override**
    - Logic: The NXKORR library contains a customer-specific override of RK101R. This may include additional validation rules or field constraints beyond the standard NXCLOUD version. All standard rules above apply as a minimum floor; the NXKORR version may add stricter conditions.

---

## Subprogram Calls Affecting Logic

| Program | Trigger | Logic | Impact |
|---------|---------|-------|--------|
| RK110R | RK100R F8 key | Open AR posting browse | Blocked if FAAK03 ≠ 5 (module not licensed) |
| RF020R | RK120R F2 key | Open customer notes/annotation sheet | Informational only |
| RK120R (factoring) | V5.40 function | Extract open items for factoring | Skips customers with RKIIZY set |
| AR700R | Called by other AR programs | Return version number for AR environment | Returns 0 if environment not configured |

---

## Reference Tables

| Table | Key | Purpose in Customer/AR Processing |
|-------|-----|-----------------------------------|
| RKUNPF | RKFIRM + RKKUND | Customer master — all customer attributes, credit limit, invoice type, flags |
| RKMEPF | Customer key | Credit memo register — current outstanding balance (RKMEMS) |
| RKTRPF | RKTFIRM + RKTAAR + RKTBILN | AR transaction register — all posted AR transactions |
| RKWBPF | Open-item key | Customer open items — outstanding invoices for interest/factoring |
| RKLAPF | Project link key | Open-item to project linkage |
| FKPRPF | FLKUND | Customer price/profile records — propagation target for fakturatype changes |
| FOHEPF | FOFIRM + FONUMM + FOSUFF | Order headers — propagation target for fakturatype/betingelse changes |
| FSTSPF | FAFIRM | System status — FAAK03 module license flag |
| BSTSPF | System key | System constants — protected/system customer number ranges |
| AMILPF | ARMILJ | AR environment configuration — version number, environment parameters |
