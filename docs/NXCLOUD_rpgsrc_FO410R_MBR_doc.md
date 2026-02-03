# FO410R.MBR

**Type**: Single File Analysis
**Path**: NXCLOUD/rpgsrc/FO410R.MBR
**Generated**: 2026-02-03 11:02:43
**Processing Time**: 10550ms

---

```markdown
# Business Logic for Order Deletion (FO410R)

This document outlines the business rules that govern the deletion of orders within the system, based on an analysis of the RPG program(s) `FO410R`. The primary focus is on the conditions that will **block** an order from being deleted.

The core logic for the deletion process is contained within the `opplag` subroutine in `FO410R`. An order must pass all the following checks to be processed.

## Order Status and Header Rules

1.  **Check Order Existence in Order Header**
    *   **Logic:** The program must verify if the order exists in the `fohelur` file before deletion.
    *   **File:** `fohelu` (Order Header Register)
    *   **Field:** `fohelu_numm`, `fohelu_suff`
    *   **Condition:** The program will **not** delete an order if the order header (`fohelu`) is not found, indicated by `*in92 = *off`.

2.  **Check for Santander Payment**
    *   **Logic:** Orders paid with Santander payment method must be specifically checked and deleted accordingly.
    *   **File:** `rksjlur` (Santander Payment Status)
    *   **Field:** `rksjlu_jnum`, `rksjlu_jsuf`
    *   **Condition:** The program will **not** delete the order if corresponding Santander records exist, indicated by `if %found`.

3.  **Check for NOBB Frakt Deletion**
    *   **Logic:** Verify if the order requires the deletion of associated NOBB freight details.
    *   **File:** `foh2lur` (NOBB Freight Details)
    *   **Field:** `fohelu_numm`, `fohelu_suff`
    *   **Condition:** The order will **not** be deleted if existing NOBB freight entries are found, indicated by `if %found`.

## Configuration and Authorization Rules

4.  **Deletion Criteria Based on Type**
    *   **Logic:** Orders designated with certain types need special treatment before deletion.
    *   **File:** `fdellur` (Deleted Orders)
    *   **Field:** `p_type`
    *   **Condition:** An order will **not** delete lines if `p_type` is equal to *blank.

## Financial and Transactional Rules

5.  **Update Customer Memo Balance**
    *   **Logic:** The customer memo balance must be updated when deleting an order.
    *   **File:** `customer_register` (Customer Register)
    *   **Field:** `footyp`, `fokund`
    *   **Condition:** The deletion will **not** proceed if failing to recalculate balances dictated by transaction lines (`fototr`, `fotots`).

## Special Conditions (Program-Specific)

6.  **Delete Associated Deposit Information**
    *   **Logic:** The program must check and delete deposit information linked to orders.
    *   **File:** `sdeplur` (Deposit Information)
    *   **Field:** `sdeplu_numm`
    *   **Condition:** The deletion process **will not** complete if the deposit record is locked or not found, indicated by previous conditional checks.

7.  **Clear Transport Logging Entries**
    *   **Logic:** Link transport logging entries must be cleared as part of the order deletion.
    *   **File:** `ftrpi2r` (Transport Logging)
    *   **Field:** `ftrpi2_rnum`, `ftrpi2_rsuf`
    *   **Condition:** Deletion **will not** occur if transportation records cannot be verified as found.

## Subprogram Calls Affecting Logic

Beyond direct file checks, several external subprograms are called that play a significant role in the workflow.

1.  **`FS711R` (Log Order Deletion)**
    *   **Trigger:** Called for logging before exiting the order deletion subroutine.
    *   **Logic:** This subroutine handles the registration of deleted orders in the deletion log.
    *   **Impact:** Essential for maintaining an audit trail; bypassing or failure can affect overall compliance and traceability.

2.  **`FA922R` (Update Customer Memo)**
    *   **Trigger:** Called after calculating memo balances during the order deletion.
    *   **Logic:** It updates the customer memo balance based on the order values.
    *   **Impact:** Inability to call this subroutine will impact the accuracy of financial records related to customer accounts.
```