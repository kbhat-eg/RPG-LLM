# CO402R.MBR

**Type**: Single File Analysis
**Path**: NXCLOUD/rpgsrc/CO402R.MBR
**Generated**: 2026-02-03 10:48:21
**Processing Time**: 7868ms

---

```markdown
# Business Logic for CO402R

This document outlines the business rules that govern the retrieval of configuration values for a given order, based on an analysis of the RPG program(s) `CO402R`. The primary focus is on the conditions that will **block** an order from being processed.

The core logic for retrieving values is contained within the `CO402R` subroutine in `CO402R`. An order must pass all the following checks to be processed.

## Order Status and Header Rules

1.  **Valid Filgruppe Check**
    *   **Logic:** The program tries to retrieve values from the `SKTIST` file based on the provided `Filgruppe`, `Firma`, `Lager`, and `Nokkel`.
    *   **File:** `SKTIST` (Configuration Values Lookup)
    *   **Field:** `Filgruppe` 
    *   **Condition:** The process will not select an order if the combination of `Filgruppe`, `Firma`, `Lager`, and `Nokkel` does not yield any results (i.e., SQL code `100` indicates no rows found).

2.  **Fallback to Chain Lookup**
    *   **Logic:** If no value is found for the given parameters, it attempts to call the `GETCHAIN` program to retrieve an alternative `Filgruppe`.
    *   **File:** `SKTIST` (Configuration Values Lookup)
    *   **Field:** `Filgruppe` 
    *   **Condition:** If the SQL code returns `100` after checking the original `Filgruppe`, a chain value is fetched, and another lookup is attempted.

## Configuration and Authorization Rules

3.  **Blank Filgruppe Check**
    *   **Logic:** As a last resort, the program checks if there are any results for an empty `Filgruppe`.
    *   **File:** `SKTIST` (Configuration Values Lookup)
    *   **Field:** `Filgruppe` 
    *   **Condition:** The program will not proceed if no values are found for `Firma`, `Lager`, and `Nokkel` with a blank `Filgruppe`, as indicated by the SQL code `100`.

## Financial and Transactional Rules

[There are no financial or transactional blocking rules present in this program.]

## Special Conditions (Program-Specific)

[There are no special conditions documented that affect the order workflow beyond those listed above.]

## Subprogram Calls Affecting Logic

Beyond direct file checks, several external subprograms are called that play a significant role in the workflow.

1.  **`GETCHAIN` (Retrieve Alternate Filgruppe)**
    *   **Trigger:** The program is called when the initial lookup in `SKTIST` returns no results (SQL code `100`).
    *   **Logic:** It attempts to get an alternative `Filgruppe` that will then be used in a second lookup for the original `Firma`, `Lager`, and `Nokkel`.
    *   **Impact:** This call allows the program to expand its search criteria and continues processing if a valid chain value is found.
```