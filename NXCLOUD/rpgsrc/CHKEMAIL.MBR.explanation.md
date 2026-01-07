# CHKEMAIL Program Documentation

## Overview

**Program Name:** CHKEMAIL  
**System:** NXCLOUD  
**Purpose:**  
The `CHKEMAIL` program validates that an email address is correctly formatted according to a set of business rules. It is used to ensure that user input for email addresses meets minimum validity requirements before further processing.

## Structure and Flow

This is a classic OPM (Original Program Model) RPG program, using fixed-format RPG IV (RPGLE) conventions. The program is designed to be called with two parameters:

- `p_emal`: The email address to be validated (input).
- `p_stat`: The status flag indicating the result of the validation (output, single character).

The main logic is executed upon call, and the validation result is returned immediately.

### Data Definitions

- **Local Data Area (LDA) Fields:**  
  - `l_firm` and `l_user` are mapped from the Local Data Area but are not used in this program. These fields are likely included by convention for all programs in the system, possibly for future expansion or standardization.

- **Working Variables:**  
  - `w_sk`, `w_sp`, `w_sb`: Integer variables used to hold positions of special characters in the email string (`@`, `.`, and space respectively).
  - `p_emal`: Input parameter, the email address (up to 50 characters).
  - `p_stat`: Output parameter, the status flag (`'F'` for invalid, blank for valid).

### Parameter Handling

Parameters are defined and initialized in the `*inzsr` subroutine, which is standard in this codebase for parameter mapping and initial setup.

### Email Validation Logic

The validation is performed using a series of checks:

1. **Presence of '@':**  
   Checks if the email contains at least one `@`. If not found, sets `p_stat` to `'F'` (invalid) and exits.

2. **Presence of '.':**  
   After the `@`, checks for a `.`. The position is stored in `w_sp`.

3. **Presence of Space:**  
   Checks for a space character, which is not allowed in valid email addresses.

4. **Positional Checks:**  
   A series of business rules are applied:
   - `@` must not appear before position 3 (i.e., at least two characters before it).
   - `.` after `@` must appear at position 6 or later.
   - There must be at least two characters between `@` and `.`.
   - There must be at least two characters after the `.` (i.e., for the domain suffix).

   If any of these checks fail, `p_stat` is set to `'F'` and the program exits.

5. **Multiple '@' Check:**  
   The program ensures that there is only one `@` symbol in the email address. If a second `@` is found, `p_stat` is set to `'F'` and the program exits.

6. **If all checks pass:**  
   `p_stat` remains blank, indicating a valid email address.

### Exit and Cleanup

The program sets the LR (Last Record) indicator on and returns, following standard RPG program termination conventions.

## Design Decisions and Conventions

- **Hardcoded Validation Rules:**  
  The checks are explicitly coded with positional logic, reflecting specific business requirements for email formatting. This approach provides clear, maintainable validation but is less flexible for changes to email standards.

- **Goto Usage:**  
  The use of `GOTO end` for early exit is a legacy pattern in this codebase for error handling in validation routines.

- **Parameter Passing:**  
  The program expects to be called with parameters and returns its result via the output parameter, a common pattern in this system for validation and utility programs.

- **Localization:**  
  Some comments and variable names are in Norwegian (e.g., "Sjekker at email er riktig utfylt" = "Checks that email is correctly filled in"), reflecting the business context.

- **No External Dependencies:**  
  This program is self-contained and does not interact with databases, APIs, or other modules. It is likely called by other programs or user interfaces to validate email input.

- **LDA Fields:**  
  The inclusion of `l_firm` and `l_user` is likely a standard header for all programs in the system, even if not used in this particular program.

## Interaction with Other Modules

- **Calling Context:**  
  `CHKEMAIL` is expected to be called from other application modules (possibly user maintenance, customer registration, etc.) wherever email validation is required. It does not directly interact with other programs but serves as a utility.

## Summary Table

| Variable   | Type   | Description                                  |
|------------|--------|----------------------------------------------|
| p_emal     | Input  | Email address to validate                    |
| p_stat     | Output | Result flag: blank (valid), 'F' (invalid)    |
| w_sk       | Local  | Position of `@` in email                     |
| w_sp       | Local  | Position of `.` after `@`                    |
| w_sb       | Local  | Position of space in email                   |

## Business Rules Enforced

- Email must contain exactly one `@`.
- `@` must not be in the first two positions.
- There must be a `.` after the `@`, at position 6 or later.
- At least two characters must exist between `@` and `.`.
- At least two characters must exist after the `.`.
- No spaces allowed in the email address.

---

This program is a reusable component for email validation, designed for integration into broader business processes within the NXCLOUD system. Its logic and structure reflect both the technical and business standards of the organization.