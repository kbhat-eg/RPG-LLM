# GEMINI.md

This file provides guidance to Gemini when working with code in this repository.

## System Overview

This is a comprehensive IBM i (AS/400) ERP business management system built with RPG IV, CL (Control Language), DDS (Data Description Specifications), and SQL. The system provides modules for finance/accounting, inventory/warehouse management, procurement, general ledger, customer management, job management, and reporting.

## Directory Structure

The codebase is organized into two main library structures:

### NXCLOUD/ (Main production library)
- `clsrc/` - Control Language source programs for compilation, job control, system setup
- `ddssrc/` - Data Description Specifications for screen/report formats and database structures  
- `rpgsrc/` - RPG source programs containing business logic
- `sqlsrc/` - SQL DDL and stored procedures

### NXKORR/ (Corrections/patches library)
- Contains modified versions of programs from NXCLOUD
- Includes `.explanation.md` files documenting program functionality
- `sqlsrc/` - Database purge and maintenance scripts

## Key Development Commands

### Compilation Commands
```cl
# Compile RPG program
CRTBNDRPG PGM(library/program) SRCFILE(library/QRPGLESRC)

# Compile CL program  
CRTBNDCL PGM(library/program) SRCFILE(library/QCLLESRC)

# Create display file
CRTDSPF FILE(library/displayfile) SRCFILE(library/QDDSSRC)

# Create SQL RPG program
CRTSQLRPGI PGM(library/program) SRCFILE(library/QRPGLESRC)
```

### System Setup Commands
```cl
NX          # Main system launcher
NXDIST      # Distribution/deployment menu system
FSETUP      # Finance module setup
LSETUP      # Logistics setup
AINSTALL    # System installation script
```

### File Operations
```cl
CRTLIB LIB(libname)                    # Create library
CRTSRCPF FILE(lib/srcfile) RCDLEN(112) # Create source file
CRTPF FILE(lib/datafile)               # Create data file
```

## Architecture and Code Organization

### Naming Conventions
- **Program naming**: `[Module][Number][Type]` (e.g., `FO614C` = Finance Order 614 RPG program)
- **Module prefixes**:
  - `AA*` - System administration
  - `FO*` - Financial/Order processing  
  - `LI*` - Logistics/Inventory
  - `GL*` - General Ledger
  - `AG*` - General utilities
  - `JV*` - Journal/Voucher processing
  - `L**` - Logistics functions
  - `R**` - Reports

### Key System Files
- `AINSTALL.MBR` - Main system installation script
- `NX.MBR` - Main system launcher  
- `NXDIST.MBR` - Distribution/deployment system
- `*SETUP.MBR` - Module-specific setup programs (FSETUP, LSETUP, etc.)

### Traditional IBM i Development Model
- **Green-screen applications** using 5250 display files
- **Subfile-based user interfaces** for list processing
- **Fixed-format RPG** with extensive use of indicators
- **File-centric architecture** with physical files as primary data stores

## Development Workflow

1. **Source Management**: Code stored in source physical files (QRPGLESRC, QCLLESRC, QDDFSRC)
2. **Libraries**: NXCLOUD (production), NXKORR (corrections/patches)
3. **Modifications**: Place corrections in NXKORR library following existing naming conventions
4. **Version Control**: Traditional AS/400 source management with dates and programmer initials
5. **Deployment**: Use `NXDIST` command for distribution management

## Integration Capabilities

The system integrates with:
- EDI processing (Electronic Data Interchange)
- Email systems and PDF generation
- FTP file transfers
- External databases via ODBC/SQL
- Norwegian standards (NOBB building products database)

## Development Best Practices

### File Relationships
- Check DDS source for file layouts and field definitions
- Look for REFFLD keywords to understand field relationships
- Check logical files for key structures and access paths

### Program Modification
- Place corrections in NXKORR library
- Follow existing naming conventions strictly
- Update change logs with version, date, and initials
- Test in development library before promotion

### Compilation
- Use appropriate compile commands for each source type
- Ensure library list includes all required libraries
- Check for dependencies in called programs and service programs
