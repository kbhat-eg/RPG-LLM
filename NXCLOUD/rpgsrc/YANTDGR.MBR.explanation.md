## Program Overview

**Program Name:** YANTDGR  
**Purpose:**  
This is a utility program designed to calculate the number of days between two dates. It is intended as a shared (fellesprogram) component, reusable across the system wherever such date difference calculations are required.

**Key Functionality:**  
- Accepts two dates as input parameters.
- Returns the number of days between those two dates via an output parameter.
- Internally, also calculates the difference in years and months (though only the day count is used).

---

## Structure and Flow

### Parameter Definitions

- **pDato1** (`s 8 0`): First date, expected in numeric format (YYYYMMDD).
- **pDato2** (`s 8 0`): Second date, expected in numeric format (YYYYMMDD).
- **pDager** (`s 5 0`): Output parameter, will contain the calculated number of days between the two dates.

### Working Variables

- **wdato1** (`s d datfmt(*iso)`): Date variable in ISO format, used for date arithmetic.
- **wdato2** (`s d datfmt(*iso)`): As above, for the second date.
- **wantaar** (`s 3 0`): Holds the difference in years (not used in output).
- **wantmnd** (`s 4 0`): Holds the difference in months (not used in output).
- **wantdgr** (`s 5 0`): Holds the difference in days (used for output).

---

## Main Logic

1. **Initialization and Parameter Handling:**
    - The `*inzsr` subroutine is used to define the program's interface (parameters).
    - The parameters are defined in the order: `pDato1`, `pDato2`, `pDager`.

2. **Date Conversion:**
    - The numeric input dates (`pDato1` and `pDato2`) are moved into ISO date variables (`wdato1` and `wdato2`).
    - This enables use of RPG's date arithmetic operations.

3. **Date Difference Calculation:**
    - The program computes the difference between the two dates in three ways:
        - Years (`wantaar`)
        - Months (`wantmnd`)
        - Days (`wantdgr`)
    - Only the day difference (`wantdgr`) is actually used for the output.

4. **Result Assignment:**
    - The number of days (`wantdgr`) is moved into the output parameter (`pDager`).

5. **Program Termination:**
    - The program ends with a `return` statement, after a labeled section (`avslutt tag`) for clarity and possible future extension.

---

## Business and Domain-Specific Notes

- **Date Format Assumptions:**  
  Input dates are expected as 8-digit numbers in the format YYYYMMDD. The program converts these to ISO date format for calculation.
- **Shared Utility:**  
  This program is meant as a common utility and may be invoked by various business modules needing to compute date differences (e.g., aging, deadlines, durations).
- **Extensibility:**  
  While only the day count is currently output, the code also calculates years and months, making it easy to extend with additional output requirements in the future.

---

## Design Decisions and Conventions

- **Parameter Interface:**  
  Uses a parameter list (`plist`) for compatibility with other RPG programs in the system.
- **Date Arithmetic:**  
  Relies on RPG's built-in date arithmetic (`subdur`) for robust and locale-independent calculations.
- **Variable Naming:**  
  Norwegian naming conventions are used, reflecting the system's origin and business language.
- **Documentation:**  
  The code is well-commented in Norwegian, with a detailed header documenting purpose, parameters, and revision history.

---

## Interactions

- **No External Dependencies:**  
  This program is self-contained and does not interact with files, APIs, or other modules directly.
- **Integration:**  
  Designed to be called by other programs, passing in dates and receiving the day difference.

---

## Summary Table

| Variable   | Type          | Purpose                                   |
|------------|---------------|-------------------------------------------|
| pDato1     | S 8 0         | Input date 1 (YYYYMMDD)                   |
| pDato2     | S 8 0         | Input date 2 (YYYYMMDD)                   |
| pDager     | S 5 0         | Output: number of days between the dates  |
| wdato1     | S D *ISO      | Working date 1 (for date arithmetic)      |
| wdato2     | S D *ISO      | Working date 2 (for date arithmetic)      |
| wantaar    | S 3 0         | Years difference (not used in output)     |
| wantmnd    | S 4 0         | Months difference (not used in output)    |
| wantdgr    | S 5 0         | Days difference (used in output)          |

---

## Key Takeaways

- Use this program whenever you need to compute the number of days between two dates in the system.
- Input and output are all numeric (packed decimal), with internal conversion to ISO date for calculation.
- The program is easily extensible for additional date-related calculations if needed.