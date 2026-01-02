# RPG Program Overview

This program, named **AP620RK**, is designed to generate XML documents for printing operations on IBM i systems, usually to integrate with a client or web application for output job (spool) handling, including to PDF. It utilizes the **CGIDEV2** toolkit for managing HTML/XML templates and web output, and relies on data in various spool files as well as configuration settings stored in a physical file.

## Program Structure

### 1. Header and Metadata

- **/TITLE** and `H` specs: Title and compile-time options (decimal and date edit, no debug IO, program binding, activation group).
- **/copy**: Pulls in external copybooks for binding specs and prototypes.
- **Comments**: Rich documentation with change logs, authorship, and adaptation history (in Norwegian).

### 2. File and Data Definitions

#### Files

- **fafpslr**: Input file, renamed for internal use (`afpspfr` to `afpslrr`). Contains system properties/settings.

#### Data Structures (Variables)

- Parameter variables for several spool (output) files: `p_spfnam`, `p_jobnam`, etc.
- Additional variables for customer, user, job keys, status, email, text fields, etc.
- Variables loaded from the **Local Data Area (LDA)**, using overlays for specific offsets (common in RPG for quick session data).
- Other working variables for date, text, status, keys, output paths, etc.

#### XML and Path Variables

- **XML_Output**, **XML_path**, **XML_dftspl**: For handling output XML file name, working path, and default template.
- **jasper_logo**: Path/filename for logo image in output.

### 3. Main Line Logic

- **Logging**: Sets a status (`w_jstat = 10`) and logs a text indicating whether XML is being generated (using `AB705R` call).
- **Conditional Generation**: If PDF generation is enabled (`l_poff = '1'` from LDA), runs the `xml_print` subroutine to generate XML.
- **Program End**: Signals last record (`*inlr = *on`) and returns.

### 4. Subroutines

#### xml_print

- **Purpose**: Build the XML for the print job.
- **CGIDEV2 API**: Uses `UpdHTMLVar` to set variables in a template, then `WrtSection` to write XML sections.
- **Job, Credential, Spool Info**: Inserts job and spool identifiers, credentials (including user/schema/company info), and details for one or more spools.
- **Print & Archive Metadata**: Adds meta tags for display and archiving in the client, like user, routine, and date.
- **Output File**: Determines XML file path and calls `WrtHtmlToStmf` to write the XML to the IFS.
- **Logs Completion**: Updates job text with info that the file was generated, again via `AB705R`.

#### Initialization (*inzsr)

- **Parameters**: Associates input parameters with variables.
- **System Properties Setup**: Fetches settings from the configuration file for:
  - Whether PDF generation is enabled (`STATUS`)
  - Which XML template to use (`DFTXMLSPLK`)
  - Where to save the XML (`XMLPATH`)
  - The logo to use in the output (`LOGO`)
- **Date/Time Capture**: Captures current date for later use.

### 5. Key Techniques & Patterns

- **Template-Driven Output**: Uses CGIDEV2's variable/section system for flexible XML generation.
- **LDA Usage**: Overlays LDA slots for fast intra-session context.
- **Keyed I/O and Chain**: Loads configuration values by key/pair.
- **Multiple Spool Support**: Handles up to three spool files per run (for multi-document jobs).
- **Conditional Metadata**: Versions and comments indicate evolution to include more client-driven tags for display and release handling.

---

## High-Level Flow

1. **Startup**: Loads parameters and system settings.
2. **Log/Status**: Announces whether XML (PDF) generation is active.
3. **If enabled**: Builds and writes XML for one or more spool files, including metadata and print/archive instructions.
4. **Finalization**: Writes a completion message and exits.

---

## Onboarding Tips

- **CGIDEV2 Familiarity**: Understanding `UpdHTMLVar` and `WrtSection` is key—they map program variables to output template sections for dynamic XML creation.
- **LDA Overlays**: Know how LDA data is carved up into individual session variables.
- **File afpslrr**: This is a properties/configuration file, used like a lookup table for system settings.
- **Parameter Passing**: Watch for strict order in input parameters—they align with the PLIST in *inzsr.
- **Error Handling**: Fairly minimal—mainly logs to a batch job log and checks for found/not found in file lookups.

---

## Example Use Cases

- When a user (via client or web interface) requests a print job, this program creates a matching XML output with all required metadata and instructions for later print/PDF handling or archival.
- Multi-document jobs: The program can generate print instructions for up to three related spool files as part of one logical batch.

---

## Key Code Snippets

- **Write variables to template:**
    ```
    UpdHTMLVar('Spoolfilename': p_spfnam);
    WrtSection('SpoolReportCommand');
    ```

- **Output XML to IFS file:**
    ```
    callp WrtHtmlToStmf(XML_Output: 819)
    ```

- **Fetch configuration value:**
    ```
    afpslr_uide = 'STATUS    '
    afpslr_key chain afpslrr
    if not %found
        l_poff = '0'
    else
        movel afudss l_poff
    endif
    ```

---

## Norwegian Terms (for context)

- *utlegg*: output, disposition
- *vanlige utskrifter*: ordinary/standard prints
- *klienten*: the client (user interface or app)
- *frislipp*: release (of print job)
- *arkiv*: archive

---

## Summary

This program is a dynamic XML generator for print job orchestration, with strong configuration, template-based output, and meta-tag integration, ready for use in print-to-file, print-to-PDF, or distributed client printing on IBM i. The main responsibility is to accurately reflect spool job details and control parameters in the output XML for downstream processing.