# RPG Program BOENOR - Detailed Explanation

This RPG/400 program (BOENOR) is a part of the ASSHOP system and is responsible for updating and recalculating transactional lines ("BONG"). The code is written in classic (fixed-format) RPG IV, with comments and some variable names in Norwegian.

Let’s break down the structure and key logic:

---

## 1. **Header & Change History**
- The program title, system/purpose, and modification history are documented at the top in comments. 
- Notable changes include adjustments to price group handling and calls to other programs (`VP900R`).

---

## 2. **File Definitions**
- The program interfaces with several database physical files (DISK). Each file is referenced and sometimes renamed for clarity:
    - `VLTYL1` – (Type file, renamed to VLTYL1R)
    - `BMHEL1` – (Header register, renamed to BMHEL1R)
    - `BMDTLU` – (Line register, renamed to BMDTLUR)
    - `BMDTL1` – (Line register, renamed to BMDTL1R)

---

## 3. **Stand-Alone/Data Structures and Work Variables**
- `w_prgr` (2A): Holds the price group (prisgruppe), widened from 1 to 2 characters in version 8.01.

### a. **Price Transfer Fields (Sender/Receiver)**
- Two data structures, `hirec` (120A) and `horec` (80A), overlay fields for transferring price and discount information between modules or programs.
- Fields are mapped using the `overlay` keyword to reference particular parts of these structures, e.g.:
    - `hifirm` overlays the first 3 digits of `hirec`, company number.
    - `hienhe` overlays positions 35-37, unit of measure.

---

## 4. **Main Calculation and Update Logic**

### a. **Initialization**
- Zeroes out the total sum accumulator (`WWTOTS`).

### b. **Fetch Bong Header**
- Reads the header record (`BMHEL1R`) using a composite key (`HEDKEY`) based on company, year, and other header-level info.

### c. **Line Register Loop**
- Begins iterating over transactional lines associated with the given 'bong' (receipt/order/document).

#### Steps per line:
1. **Fetch Line Record** – Reads each line from `BMDTL1R` file.
2. **Check for Continuation** – Compares current line's keys to the header's to ensure processing only relevant lines.
3. **Fetch Line Type** – Reads from type file (`VLTYL1R`), using key fields.

#### Price/Discount Calculation (for amount lines):
- If the line is a value line (`VALKTX = 0`), overlays variables for price/discount calculation, sets up from line data.
- If the 'status code' (`WPSTAT`) is blank and the price type isn’t fixed, retrieves new prices and discounts:
    - If a fixed discount is present, uses it.
    - Otherwise, calls subroutine `HNTPRI` to fetch prices (calls external program `VP900R`).

#### Sales and Discount Calculation:
- Calculates total amounts, line subtotals, and applies multiple discounts in sequence (from various sources).
- Each resultant subtotal is accumulated into the total for the whole 'bong' (`WWTOTS`).

#### Update Line:
- Updates the `BMDTLUR` file with recalculated fields.

---

## 5. **Subroutines**

### a. **HNTPRI (Fetch Price Info)**
- Populates the sender data structure (`hirec`) with current transaction details.
- Calls external price determination program `VP900R` passing both sender (`hirec`) and receiver (`horec`) DS for results.
- Extracts computed prices and discounts from `horec` into work fields for further processing.

### b. ***INZSR (Initialization)**
- Initializes work variables and overlays for field mapping, including set up of all key fields and "LIKE DEFINE" for cross-variable compatibility.
- Handles reading system-initialized values for company, year, and workstation ID.
- Calls another program (`VL711R`) to fetch the price group into `w_prgr`.

---

## 6. **Parameter Handling**
- The program is driven by incoming parameters:
    - Status, company, year, WSID, line number, fixed discount, and total sum.
- These are mapped to internal variables for use throughout the code.

---

## 7. **Exiting the Program**
- After processing, the program sets the LR indicator (last record) and returns.

---

## 8. **Miscellaneous**
- Includes various key lists (`KLIST`) for file access.
- Makes extensive use of overlays and LIKE-DEFINEs for flexible field mapping, a common RPG technique to keep field usage DRY and consistent.

---

# **Summary of Functionality**

- **Purpose:** Updates the line items for a transactional document ("bong"), recalculating discounts and prices as needed, and writing changes back to the database.
- **Structure:** Processes a header, loops through matching detail lines, fetches supplementary info, performs new calculations, and updates files.
- **Integration:** Interfaces with external programs for price calculations and price group determination.
- **Parameters:** Receives control, selection, and defaulting information via program parameters.

---

## **Key Norwegian Terminology**
- **Bong**: A receipt, ticket, or transactional document.
- **Varelinje**: Item line (transactional line on the bong).
- **Pris**: Price.
- **Rabatt**: Discount.
- **Overføringsfelt**: Transfer field.
- **Oppdatering**: Update.
- **Endring**: Change.

---

## **Onboarding Notes**
- **Modular Design:** Understand the data flow via the data structures (`hirec`/`horec`), which is central to how prices and discounts are handled.
- **Database Layout:** Familiarize yourself with the physical files, especially the “BONG” line and header registers, and type definitions.
- **External Program Calls:** The logic for price fetching is offloaded to `VP900R`.
- **Extensibility:** New price groups or logic can be added by expanding DS fields or adjusting the price routine integration.

---

This explanation should give you a strong understanding of the program’s flow, organization, and integration points. For further onboarding, focus on the price calculation subroutine (`HNTPRI`), file formats, and the parameter protocol between programs.