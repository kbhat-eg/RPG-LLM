# Explanation of RPG Program `AP620RK`: XML Generation for Printing

## Overview

This RPG IV/ILE program, `AP620RK`, is used to generate XML files for regular print jobs ("utskrifter") on IBM i (AS/400) systems. It integrates with the CGIDEV2 library for dynamic HTML/XML generation, and fetches configuration and print metadata from files and the LDA (Local Data Area).

The main purpose is to create an XML representation of print spool files and related metadata, intended for downstream processing or archiving—typically for a Java-based client or archival solution.

---

## Header and Environment

- **H-specs (`H` lines):**
  - DECEDIT('0.'), DATEDIT(*DMY.): Sets decimal and date editing for the program.
  - DftActgrp(*No): Runs in its own activation group (ILE).
  - BndDir('CGIDEV2/CGIDEV2'): Binds to the CGIDEV2 service program, enabling HTML/XML utilities.
  - /copy hspecsbnd: Includes standard binding source.

- **Documentation block**: Describes the program, change log, and special requirements (CGIDEV2 must be in the library list).

---

## File and Copybook Definitions

- **File Declaration:**
  - `fafpslr if e k disk rename(afpspfr:afpslrr)`: Declares a file for lookups; renames record format.

- **Copybooks included:**
  - `/copy prototypeb`
  - `/copy usec`

---

## Parameter and Working Variables

### Parameters

The program expects a large parameter list (defined in *inzsr, using *entry & plist):

- Three sets of spool file info (`p_spfnam`, `p_jobnam`, `p_usrnam`, ... up to three).
- Metadata variables, e.g. customer (`p_kund`, etc), status, email, subject, free text lines, etc.
- Ten possible pairs for user-defined (free-form) meta tags: `p_taggN` and `p_taggNval`.
- Miscellaneous: job keys, batch type, reference number, display tag.

### Working Variables

- Various fields for current date, time, job key, file paths, status codes, logging text, etc.
- XML file paths and template/section variables for building the output.

---

## Program Flow

### Initialization (*inzsr)

- **Parameter List**: Reads in parameters from the program call.
- **Setup Key Fields**: Sets up variables for DB file lookups (e.g., `afpslr_ufil`, `afpslr_uide`).
- **Check Activation (PDF On/Off)**:
  - Looks up `STATUS` in the configuration file (`afpslrr`).
  - Sets `l_poff` flag (on/off).
- **Read XML template and output paths, logo path from config file** (if PDF is active):
  - Template: `DFTXMLSPLK`
  - Output Path: `XMLPATH`
  - Logo: `LOGO`
- **Current Date**: Puts the current date/time in `w_date`.

---

### Mainline Logic

- **Logging**: Logs the start (or skip) of XML generation, using job key and status, by calling external program `AB705R`.
- **If PDF generation is active (`l_poff = '1'`)**
  - Calls subroutine `xml_print` to generate the XML file.

- **Cleanup and End**: Sets LR (Last Record) indicator and terminates.

---

### Subroutine: xml_print

#### Purpose

Handles all XML composition and storage for the print job.

#### Steps

1. **Open XML Document**
   - Calls `UpdHTMLVar` and `WrtSection` procedures (CGIDEV2) to prepare XML sections, insert variables.
   - Inserts batch info, user credentials, schema details.

2. **Spool File Information**
   - Inserts details for up to three spool files (name, number, job, user).
   - Controls release and remove flags for spools.
   - There is code for further print instructions (printer, copies, duplex, etc.)—but it's behind `if 1 = 2` (never executed; likely legacy code).

3. **Display Tag for Archiving**
   - Sets a display tag: either built from routine/user or passed in as `p_dspl`.

4. **Meta Tags**
   - Writes structured meta tags (`Metakey`/`Metavalue`) for screen, company, user, routine, date.
   - Loops through all free meta tags (`p_tagg0` to `p_tagg9`), inserting them if not blank.

5. **Finalization**
   - Writes archive command end and footer sections.
   - Constructs XML output filepath.
   - Calls `WrtHtmlToStmf` to write the constructed XML to the IFS (Integrated File System).
   - Logs XML generation (again, via `AB705R`).

---

## Integration

- **CGIDEV2 functions (`UpdHTMLVar`, `WrtSection`, `WrtHtmlToStmf`, etc.):**
  - Core to the program: Used for setting variables and writing sections to build the XML file dynamically.
- **External program `AB705R`**: Used for logging/reporting job status.

---

## Configuration

- Dynamic properties (template file, output directory, logo, etc.) are fetched by key from the `afpslrr` file, allowing changes without recompiling.
- The approach supports up to three print files and up to ten meta tags supplied from the calling program, enabling flexible extensibility.

---

## Exception/Legacy Handling

- Several code blocks or variables are marked with version comments and seem to be legacy or version-tracked.
- There are commented-out or dummy areas (e.g., `if 1 = 2`, alternate display tag construction, etc.)—indicating possible room for extension or past refactoring.

---

## Summary Table

| Section         | Purpose/Key Actions                                                                            |
|-----------------|-----------------------------------------------------------------------------------------------|
| *Header*        | Sets up environment, binding, and includes documentation                                       |
| *File/Copybooks*| Declares config file, brings in procedure and utility prototypes                              |
| *Parameters*    | Defines all input variables, including print jobs and meta tags                               |
| *Variables*     | Working storage for job, file paths, meta info                                                |
| *Init*          | Sets up config values, checks whether PDF/XML generation is enabled, reads in templates, etc. |
| *Main Program*  | Logs start, generates XML if enabled, logs completion, then ends                              |
| *xml_print*     | Builds XML via CGIDEV2 routines, writes file to IFS, attaches meta tags                       |

---

## Onboarding Advice

- **Understand CGIDEV2**: Familiarity with CGIDEV2 APIs (`UpdHTMLVar`, `WrtSection`, etc.) is key.
- **Config file (afpslrr)**: Holds all environment-specific configuration; know how to add/change keys for new environments.
- **Meta tags**: Adding more flexible metadata is straightforward via the parameter list and corresponding logic blocks.
- **Logging**: The external `AB705R` program is used for job status update/logging.
- **Template changes**: XML format/layout is driven by the sections and template loaded from the IFS—not hardcoded in the RPG.

---

## Practical Use

- **Called by other programs** (with all parameters) to generate XML for print spools, for archiving or further processing.
- **Easily adapted** for new meta tags, new output locations, or changes in the XML schema by updating the config or template files, not the code.

---

### Key Customization Points

- Parameter list: For new data to be included, extend parameter definitions and the relevant XML generation logic.
- Meta tags: For more flexible data storage, utilize the meta tag infrastructure (already scalable up to at least 10 custom tags).
- XML template: For layout or structure changes, modify the template in the IFS as referenced by the program.

---