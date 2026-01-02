## Overview

This program (BORGVR) is part of the ASSHOP system and handles both inquiries (“spørring”) and registration (“registrering”) of item lines. It uses a subfile-driven screen interface to allow users to:

- Search for items by various criteria (group hierarchy, partial text, item numbers, etc.).  
- Register multiple items in a “butikk-linje-register” (store/order line register) as part of a point-of-sale or order capture process.  
- Calculate and display pricing, discounts, and VAT-inclusive totals.  
- Handle special cases such as discontinued products, items with multiple units of measure, or items that require further detail input.

Although it uses standard RPG subfile techniques, this code also interacts with multiple files, external programs, and domain-specific logic for item groups, NOBB numbers, and custom price retrieval. Below is a high-level breakdown of how the program is structured and the main responsibilities in each section.

---

## Key Responsibilities

1. **Parameter Handling and Initialization**  
   - The program starts by reading five parameters (firm, accounting year, workstation ID, line number, and total sum).  
   - These parameters form the context needed to retrieve and update records in the “butikk-linje-register” and to display or update lines in the subfile.  
   - Initialization routines fetch relevant defaults (e.g., user info, date, VAT rates) by chaining to files like BWSDL1, BSTSL1, FUSRL1, BMHEL1 and by calling specialized utilities (e.g., FØ705R).

2. **Subfile Management**  
   - The screen file (borgvd) defines a subfile (`b1sfl`) and associated control record (`b2ctl`).  
   - Common subfile functions, such as clearing and populating the subfile (subroutines `CLR_SUBFILE` and `CRT_SUBFILE`), are used to refresh the displayed item list under different search or positioning criteria.  
   - The subroutine `DSP_SUBFILE` handles displaying the subfile to the user, while `SUBFILE` processes changed entries (e.g., lines where a quantity was entered).

3. **Searching and Positioning**  
   - The user can search for items by specifying an overgruppe (b2ogrp), hovedgruppe (b2hgrp), undergruppe (b2ugrp), or a “søketekst” (b2stek).  
   - Different subroutines handle the logic for grouping keys (O → overgruppe only, H → overgruppe + hovedgruppe, U → overgruppe + hovedgruppe + undergruppe, and so on).  
   - When searching by partial text, the program calls `AS700R` to parse the input text into up to five partial words (w_ste1, w_ste2, etc.), then uses embedded SQL cursors to filter items.  
   - The user can also position the subfile based on item number, NOBB number, or supplier item number (via subroutines `POSISJONER` and `SØK`).

4. **Item Line Registration**  
   - Once an item is selected, or a quantity is entered, the subroutine `REGISTRER` retrieves detailed item data (vvarlr) and calls various external programs to get price information for up to three units of measure.  
   - Pricing details are stored in the line register fields (bmsapr, bmkopr, bmrab1, etc.) along with item text.  
   - Certain item types (skaffevarer, diverse, or service items) trigger additional windows (`xc1win`) for the user to verify or enter extended details.  
   - There is a separate window (`xc2win`) for registering free-form text lines (non-item lines).

5. **Price and Discount Calculation**  
   - Multiple calls to external programs (particularly `VP715R` and `VP900R`) gather item pricing details. These include base price, cost price, discounts, and whether an item is on special offer or has store-specific pricing.  
   - Subroutines like `HENT_PRIS` encapsulate the logic to populate fields used in discount calculations.  
   - `BEREGN_TOTAL` calculates the running total including or excluding VAT, rounding to the nearest 0.5 or 1.0 as appropriate for this business logic.  
   - The user can toggle between displaying sales price with or without VAT (F22). The subroutines `ADD_MVA` and `MVA_SUB` handle conversions between gross and net amounts.

6. **Handling Discontinued and NOBB Items**  
   - Dates and statuses are checked to determine if the item should be flagged as “** UTGÅTT **” (discontinued) or “**NOBB.UTG**” (discontinued in the NOBB system).  
   - Logic in the subfile creation (`CRT_SUBFILE`) updates the item’s display text accordingly if the item is out of date (`w_utda`) or marked as discontinued (vvnuda).

7. **Unit Conversion (Kalkulator)**  
   - Items can have multiple units of measure. The `KALKULATOR` subroutine calculates how entering a value in one unit impacts the others. If the item has multiple possible units, the program handles converting from the chosen unit to the base or alternative units.

8. **Additional Function Keys and Actions**  
   - The code references many function keys (F2, F9, F13, F14, F15, F21, F22, F23) for actions like toggling item display, re-searching, viewing extended customer or discount info, or switching between including/excluding special “skaffevarer.”  
   - “F2=Registrering av tekstlinjer” calls the window for text-only lines, “F14=Kundeinformasjon” calls an external customer program (FL510R), and so forth.  
   - These calls highlight the program’s integration with other modules in the system to retrieve or edit specialized information.

---

## Notable Design Details

1. **Renames and Prefixes**  
   - The code uses multiple physical files with similar structures (e.g., vvarl1, vvarl2, etc.) and renames them to avoid naming collisions. This is particularly important when chaining or reading from these files for item information.  
   - The code consistently overlays external data structures (hirec, horec) for price retrieval routines to pack input and output parameters when calling `VP900R` and similar programs.

2. **Toggling Behavior**  
   - Several indicator-based toggles exist (e.g., *IN83, *IN84) to switch between including/excluding items with particular attributes (like skaffevarer) or showing prices including/excluding VAT. This pattern is repeated to manage user preferences and system states without reloading the entire program.

3. **Reuse of Common Subroutines**  
   - Functions like `CLR_SUBFILE`, `CRT_SUBFILE`, and `DSP_SUBFILE` are repeatedly called to update the subfile in different flows (search, positioning, toggling indicators). This further clarifies how changes in user input or search criteria always lead to subfile regeneration.

4. **External Program Calls**  
   - Many specialized calls handle domain-specific tasks, such as retrieving the correct price logic (`VP715R`, `VP900R`), item type logic (`VL710R`), group logic (`VG510R`, `VG511R`, `VG512R`), and discount or cost price calculations (`FO524R`, `FR510R`, `FL510R`).  
   - These calls centralize functionality in smaller modules, which is beneficial for maintenance and consistency.

5. **Multiple Lines and Sequencing**  
   - When adding a new line to the “butikk-linje-register,” the code checks if a line number is already in use. If so, it increments the line number by 10 and tries again. This is a domain-specific strategy to keep line numbers in increments of 10, leaving room for inserted lines if needed.

---

## Summary

In essence, this program provides a single entry point to:
- Search items by text, hierarchy, or miscellaneous identifiers.  
- Update or register item lines with detailed pricing and discount calculations.  
- Switch between various display modes (VAT on/off, grouping filters, currency rounding).  
- Integrate with other parts of the system through calls to specialized programs for pricing, item data, group data, and customer data.  

Almost all domain logic revolves around seamlessly managing item data (including discontinued logic, NOBB-specific markers, or multiple units of measure) and ensuring the user can confirm or adjust pricing in real time. The partitioning into dedicated subroutines (search, position, subfile clearing/creation, line registration, windows for extended details) keeps each functional piece relatively contained, yet orchestrated by the main interactive flow controlled through function keys.