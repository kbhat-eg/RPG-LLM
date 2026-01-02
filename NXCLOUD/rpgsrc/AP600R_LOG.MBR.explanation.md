# RPG Program: AP600R – Print Parameter Retrieval

This program, **AP600R**, is designed to collect and update print parameters from various control files for IBM i systems. It ensures that the appropriate print parameters are retrieved and set up for print jobs, supporting both classic spool/print jobs and extended handling for PDF output with special parameters. Below, you will find an explanation of the key sections and logic:

---

## 1. **Program Purpose** 
- **Main Task**: Retrieve print parameters from form files and possibly user, device, and screen registers.
- **Parameterization**: Modern version uses program parameters (not just LDA – Local Data Area).
- **PDF Support**: Enhanced to support PDF output, with dedicated parameter fields, via conditional code (`pdf` preprocessor lines).
- **Fallback Logic**: If specific parameters are not found for a routine, fall back to defaults.
- **User Interaction**: Presents screen(s) to allow operators to override or confirm values, unless job is running in batch mode.
- **Member Numbering**: Generates and manages unique member/job numbers using a database file as a sequence.

---

## 2. **File Declarations**
Various physical and logical files are declared:
- **ap600d**: Display/input screen handler.
- **aforl1**, **afopl1**, **afapl1**, **afprlr**, **afpslr**: Routine/form/register control files, some tied to PDF.
- **fausrl1**, **fawsdl1**, **faenhl1**: User, workstation, and device registers.
- **fanumlu**: Sequence/member-number counter.

---

## 3. **Data Structures & Variables**

### LDA (Local Data Area) Fields
The LDA is structured with many overlays (e.g., `l_outq`, `l_form`, `l_acpi`, `l_save`, etc.) that store parameter values for printing.

### Parameter Variables
- **p_ruti**: Routine to execute (routine name).
- **p_valg**: Flag for manual override of values.
- **p_tvbt**: Force batch (Y/N).
- **p_in03**: Indicator for F03/F12 (job cancel).

### Work Variables
- For keys, routine values, return codes, work-variables for screen fields, and PDF control.

---

## 4. **Main Logic Overview**

### a. **Overrides and Fallbacks**
1. **Printfile Override**: Default is to override (set `l_ovrk` to '1'), unless not found in the routine register and override flag is blank.
2. **Defaults**: If the routine entry isn't found, the program sets default print parameters using subroutine `xdefault`. If a default routine (`ASPSTD`) exists, those values are used.

### b. **Populate LDA from Registers**
- Parameters are loaded from the routine register (and potentially user/device/screen registers) using subroutine `xupd_lda`.
- If further levels of overrides exist (screen, user, workstation), these are checked and can supersede earlier values.

### c. **PDF Logic**
When PDF output is "activated" (`l_poff = '1'`):
- Looks up PDF-related parameters in special PDF registers (`afapl1r`, `afopl1r`).
- Populates a PDF control data structure, then updates the LDA accordingly (subroutine `xupd_lda_pdf`).
- If the routine disables PDF or isn't found, PDF is deactivated and fields zeroed.

### d. **Screen Handling**
If operator involvement is required (not batch & override is set), the program displays an input screen for the operator to adjust/confirm parameters. There are two versions of the screen: one for normal, one for PDF jobs.

### e. **Validation & Completion**
- After parameter setup, checks (subroutine `xslutt_ktr`) are performed: e.g., valid CPI, line count, output queue, form type, and so on. Missing or invalid values are corrected to sensible defaults.
- A unique member/job number is obtained from the sequence file (`xmember`).
- If the operator cancels, the program skips to end.

---

## 5. **Subroutines**

### `xdefault`
- Sets default print values, either from a default routine or hardcoded if not found.

### `xupd_lda`
- Populates LDA fields from the routine register (`aforl1`), handling manual overrides, and applying most fields.
- Also incorporates overrides from screen/user/workstation/printer as needed, in prioritized order.
- If special settings exist in device registers (e.g., OCR codes), these are also loaded.

### `xupd_lda_pdf`
- Similar to `xupd_lda`, but applies only PDF-specific parameterization to the LDA.

### `xskjerm` & `xskjerm_pdf`
- Handles the display/input screen logic for operator intervention.
- Validates screen entries (e.g., correct output queue, number of lines).
- Handles error conditions and user actions (e.g., cancel, renew, show printers).

### `xslutt_ktr`
- Performs a final check and adjustment of parameters (e.g., validate form, output queue, priority).

### `xmember`
- Manages a sequence/member number, incrementing and rolling over as needed.

### `pdf_batchno`
- Creates a unique batch/job number for PDF jobs, optionally logging details.

---

## 6. **Initialization (`*inzsr`)**
- Initializes LDA, parameter list, and sets up passed-in parameters.
- Sets up initial keys/fields for register lookups.
- For PDF, checks activation status and calls for a batch/job number.

---

## 7. **Parameter List**
At entry, receives (as parameters):
- Routine name
- Manual selection flag
- Force batch flag
- End-job indicator
- Batch indicator

---

## 8. **Error Handling & Logging**
Multiple places in the code call external programs (`AX020C`, `AB705R`, etc.) for validation and logging.

---

## 9. **Notable Comments (in Norwegian)**
- Much of the code is annotated with Norwegian comments, explaining history, version changes, and each section’s purpose (e.g., "Henter utskriftsparametre i fra formularfile" = "Retrieves print parameters from form file").
- Code is structured with detailed change logs linked to version numbers.

---

## 10. **Version & Feature Evolution**
- The program has grown over time with support for batch handling, advanced printer controls (drawers, duplex, quality), archiving, PDF output, and so on, as documented in the change log in the header.

---

# **Summary for New Developers**

- **AP600R** centralizes the logic for collecting and validating all print parameters needed for IBM i print jobs, including complex routines such as PDF document handling.
- It retrieves parameter values from several data sources (routine, user, device, screen), applies priorities and overrides, validates, and finally updates the LDA for use by downstream print programs.
- Special handling for PDF is interleaved but clearly marked by preprocessor tags (`pdf`).
- The code is heavily reliant on external database files and other programs, and integrates closely with legacy IBM i print (SCS/IPDS) and new PDF workflows.
- The flow is: **Initialize → Retrieve (with fallback) → Allow override (screen) → Finalize/validate → Update LDA → Generate member/job number → End/Log**.
- Documentation is in Norwegian, but code logic follows standard RPG idioms, and code is modular with clear, purpose-named subroutines.

---

#### If you're onboarding to maintain or extend this program, focus first on understanding the LDA overlays, external file structures, and how parameters move through the system. Then, study the screen handling, override hierarchies, and the special PDF logic blocks. The change logs in the comments will help you map code fragments to business requirements and system evolution.