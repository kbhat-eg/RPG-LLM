# Documentation: FO910R – Printing of Offers (Tilbud)

## Overview

**Program Name:** FO910R  
**Purpose:**  
This program generates offer printouts (tilbud) for customers. It supports both traditional spool file output and modern XML/PDF (JasperReports) generation for archiving, emailing, and advanced printing. It handles offer headers, item lines, text lines, totals, and integrates with several master and transaction files to gather all necessary data.

**Business Domain Context:**  
This program is part of the order management process, specifically for the creation and distribution of customer offers. It is highly configurable, supporting multiple output formats and business rules regarding discounts, taxes (MVA), department/firma logic, and multi-channel delivery (print, email, archive).

## Structure and Key Components

### File Definitions

The program uses a number of files, each renamed locally for clarity:

- **FOHEL1R** (Order header)
- **FODTL1R** (Order details/lines)
- **FOTXL1R** (Text lines)
- **VVARL1R** (Item master)
- **RKUNL1R** (Customer master)
- **FSTSL1R** (Status/offer keys)
- **AFORL1R** (Routine/master text)
- **AFIRL1R** (Company info)
- **RA09L1R, RA07PFR** (Salesperson, department info)
- **VLTYL1R, VOTYL1R** (Line type, order type)
- **VPKOL1R** (Price code)
- **FVLSL1R** (Length specification lines)
- **VLAGL1R** (Warehouse/item type)
- **FO910P** (Printer file)

### Arrays and Data Structures

- Arrays `mko`, `msa`, `mgr`, `mbe`, `mog`, `mor` are used for handling multiple VAT rates and their calculations.
- A large number of working variables are defined for temporary storage, calculations, and passing parameters to subroutines and external programs.

### LDA (Local Data Area)

The program reads various control values and flags from the LDA, such as print mode, batch info, output queue, department, etc. This enables dynamic behavior per job or user session.

### Parameters

The entry parameter list supports email delivery, customer/project IDs, order numbers, and seller IDs. These are used for routing and personalization of output.

### Main Processing Logic

The program is structured around a main cycle that processes each record based on its type (header, item line, text line), using tags (`TAG01`, `TAG02`, `TAG03`) to branch to the relevant logic.

#### 1. **Header Processing (`TAG01`)**

- Loads order header, customer, and company information.
- Determines if department or company info is to be printed, based on routine register flags.
- Fetches payment terms, delivery address, customer references, etc.
- Handles phone/fax/mobile overrides and formatting.
- Prepares output for both print and XML/PDF generation.
- Calls `hnt_avde` subroutine to fetch department info if needed.

#### 2. **Item Line Processing (`TAG02`)**

- Loads item line details, including product master data.
- Applies discount logic, including special "bruksrettsrabatt" (usage rights discount), and calculates VAT per line.
- Handles variable decimals for quantities based on item configuration.
- Writes line details to print or XML output.
- If applicable, writes additional lines for text2, special discounts, and length specifications.

#### 3. **Text Line Processing (`TAG03`)**

- Loads and outputs text lines associated with the offer.
- Handles special logic for text lines above a certain line number (e.g., for footer placement).
- Converts text to XML-safe format when generating XML/PDF.

#### 4. **Totals and Summary (`T1TOT`)**

- Calculates and outputs offer totals, including net, gross, VAT breakdowns, and discounts.
- Handles overflow logic for page breaks.
- Outputs totals in both print and XML formats.

#### 5. **Overflow Handling**

- The `XOVRFL` subroutine manages page breaks, ensuring headers and footers are written as needed.

#### 6. **VAT Array Handling (`xarray`)**

- Manages VAT calculations for up to 9 different rates per offer, supporting complex scenarios with multiple VAT types per offer.

#### 7. **Department Info (`hnt_avde`)**

- Fetches department information from the department master file for inclusion in printout/XML.

#### 8. **Length Specification Handling (`skr_leng`, `xml_spec`)**

- Special handling for products with length/quantity specifications, both for print and XML.

### XML/Jasper Integration

When XML/PDF output is enabled (`b_pdfp` flag), the program uses a set of subroutines to build the XML structure required by JasperReports. This includes:

- **xml_envm:** Writes environment, credential, and print command sections.
- **xml_head:** Writes header (company, customer, project, delivery, etc.) info.
- **xml_deta:** Writes item line info, including discounts, VAT, campaign, etc.
- **xml_rabatt, xml_netto, xml_vat, xml_totaler:** Write totals, discounts, and VAT breakdowns.
- **xml_topt_str, xml_topptxt, xml_topt_end, xml_bott_str, xml_bunntxt, xml_bott_end:** Handle top and bottom text sections.
- **xml_end:** Finalizes the XML, writes meta tags for archiving, and triggers external processes as needed.
- **pdf_preview:** Launches PDF preview in a browser if required.

The program is tightly integrated with the **CGIDEV2** toolkit for XML generation and IFS file handling.

### Email Integration

If the output queue is set for email, the program prepares all necessary parameters and calls external programs (`FO798R`, `FM798R`, `AM705C`) to handle email delivery. This includes scanning and cleaning subject/body fields for invalid characters.

### External Program Calls

- **F�703R, F�705R, F�707R, F�710R, RS205R, AB700R, AB705R, AP609R, AP611R, AP614R, AP629C, AM705C, FO798R, FM798R, CO402R, AP613R**  
  These programs are called for a variety of purposes: fetching master data, calculating VAT, logging, scanning text for XML compatibility, and managing email and print output.

### Key Business/Domain Logic

- **Discount Calculation:**  
  Handles multiple discount types, including line discounts, order discounts, and special "bruksrettsrabatt" (usage rights discount), with complex rules about which discounts affect VAT basis and totals.
- **VAT Handling:**  
  Supports multiple VAT rates per offer, with logic for both calculation and output.
- **Company/Department Output:**  
  Can print either company or department info based on configuration.
- **Flexible Output:**  
  Can produce both traditional spool files and modern XML/PDF files for archiving, preview, and emailing.
- **Meta Tagging:**  
  When archiving, writes meta tags to the XML for later retrieval/searching in archive clients.

### Design Conventions and Patterns

- **Versioning and Change Log:**  
  Extensive comments document changes by version, with references to business requirements.
- **Conditional Compilation/Execution:**  
  Many features are enabled or disabled based on flags from the LDA or parameters, allowing for flexible deployment.
- **Modularization:**  
  Use of subroutines for discrete logic (header, detail, text, totals, XML sections, overflow, etc.).
- **Integration with External Services:**  
  Email, archiving, and PDF preview are handled by calling specialized external programs, keeping the main logic focused on data assembly and output.

### Interactions with Other Modules

- **Master Data:**  
  Customer, item, department, and company data are all fetched from master files.
- **External Services:**  
  Email and PDF handling are offloaded to dedicated programs.
- **Archiving:**  
  XML files are written to the IFS and meta information is supplied for external archive/search systems.
- **Logging:**  
  Status and errors are logged via external programs for traceability.

## Special Considerations

- **Language:**  
  The code and comments are in Norwegian, but the structure is clear and variable names are consistent with the business domain.
- **Legacy and Modern Output:**  
  The program supports both legacy (SCS spool file) and modern (XML/PDF) output, switching logic based on configuration.
- **Complex Discount and VAT Rules:**  
  The business logic for discounts and VAT is non-trivial and reflects local tax/business requirements.
- **Safety and Validation:**  
  The program validates parameters, output queues, and cleans text for XML compatibility.

---

## Summary

FO910R is a central program for generating, printing, emailing, and archiving customer offers. It is highly configurable, supports complex business rules around discounts and VAT, and integrates with both legacy and modern output technologies. The codebase reflects years of enhancements and business-driven changes, making it robust but also complex. Understanding the relationships between the various files, flags, and external programs is key to effective maintenance and extension.