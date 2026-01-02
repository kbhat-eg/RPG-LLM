# Documentation for Program JV171R – Item/Price Calculation and Inquiry

## Business Purpose

**JV171R** is a core transactional program in the ASVADM system, responsible for querying and calculating item and price details at the item line level. It is typically invoked in sales, purchasing, or logistics scenarios where users need to view, calculate, or adjust item prices, quantities in various units, and associated price conditions.

The program provides a user interface for both inquiry and calculation, supporting complex pricing logic, multi-unit conversions, and integration with related master data and pricing modules.

---

## High-Level Structure and Flow

### 1. **Initialization and Parameter Handling**

- **Parameters**: The program receives parameters such as company, customer, project, delivery type, and item number.
- **Initialization**: Sets up working variables, keys for file access, and retrieves context-specific data (e.g., checks if the item is a "logistics item", fetches price group).
- **Database and File Access**: Opens and prepares access to master data files (items, suppliers, price lists, etc.), and configures embedded SQL for special lookups.

### 2. **Main Logic**

- **Item Retrieval**: Fetches item master data based on the input item number.
- **Supplier/Price Giver Logic**: Determines the price-giving supplier; if the item has multiple price givers, the user can select.
- **Logistics Item Handling**: If the item is a logistics item, additional NOBB (Norwegian Building Product Database) information and supplier item numbers are retrieved using SQL joins.
- **Screen Handling**: Presents a screen for user interaction (quantities, prices, discounts, etc.).
- **User Actions**: Supports multiple function keys for related information (product info, customer/project info, price conditions, NOBB info, etc.).
- **Validation and Calculations**: Performs thorough validation and recalculates prices, discounts, and margins as the user edits fields or selects new options.

### 3. **Subroutines and Logic Blocks**

- **xc1bld**: Main display and calculation loop for the item line.
- **kalkulator**: Converts quantities between up to three different units.
- **enhet**: Handles changes in the selected unit (recalculates prices and factors).
- **betingelse**: Fetches new price conditions via a separate program call.
- **kalk_priser**: Core calculation logic for sales price, cost price, discounts, and margin (dekningsgrad).
- **vis_nobb**: Displays NOBB information for the item.
- **sjekk_pgiv**: Checks if the item has multiple price givers.
- **Initialization (*inzsr)**: Sets up keys, parameters, and context at program start.

---

## Key Business and Domain Logic

### Multi-Unit Handling

- Items may be sold or purchased in up to three different units.
- The program supports conversion between these units, recalculating quantities and prices accordingly.
- The subroutine `kalkulator` ensures that changes in one unit reflect correctly in the other units.

### Price Calculation and Validation

- **Cost Price (Kostpris)** and **Sales Price (Salgspris)** are recalculated based on:
  - Item base data
  - Price group and conditions
  - Discounts (up to two levels)
  - Freight factors and other cost additions
  - User overrides
- **Margin (Dekningsgrad)** is always shown net of discounts, reflecting the real margin after all deductions.
- If any key price, discount, or factor fields are changed, the program recalculates dependent fields to maintain consistency.

### Price Giver (Prisgivende Leverandør)

- The program can handle items with multiple price-giving suppliers.
- The user can select the relevant supplier using function keys, which triggers a refresh of all dependent data and recalculations.

### NOBB Integration

- For logistics items, the program retrieves and displays NOBB numbers and related supplier item numbers.
- NOBB data can be previewed in a dedicated screen via a function key (`F18`).

### External Program Calls

- **JV751R**: Retrieves detailed price and item information.
- **JP505R**: Fetches price conditions.
- **JX500R**: Used for price-giver selection.
- **VL711R**: Retrieves price group information at initialization.
- **FO704C, FL510R, FR510R, FP510R, AP633R**: Provide related information screens for product, customer/project, special prices, and NOBB data.

### Embedded SQL

- Used for complex lookups, such as determining if an item is a logistics item and retrieving associated NOBB and supplier data.

---

## Design Decisions and Conventions

### Versioning and Change Management

- The source is heavily commented with change logs, showing a long history of incremental enhancements, especially around pricing, discount handling, and integration with NOBB and logistics.

### Parameter Passing

- Extensive use of parameter lists for program calls, reflecting a pattern of modular, loosely coupled business logic components.

### Data Area and Variable Naming

- Variable and field names are mostly Norwegian abbreviations; understanding these is essential for maintenance (e.g., `sapr` = sales price, `kopr` = cost price, `dekn` = margin, `ant` = quantity, `enhe` = unit, etc.).
- Working variables for each screen field are maintained to track changes and support validation.

### Error and Validation Handling

- Uses indicator variables (e.g., `*in31`, `*in32`, etc.) to flag specific validation errors, which are then reflected in the user interface.

### User Interface

- The program is designed for interactive use, with function keys mapped to common business operations.
- The main screen is repeatedly shown after each action or validation, supporting iterative user input and correction.

---

## Interactions with Other Modules

- **Master Data**: Reads item, supplier, and price list files directly.
- **Pricing and Conditions**: Delegates to other programs for complex price and condition logic.
- **NOBB**: Integrates with the NOBB database for building product information.
- **Customer and Project Info**: Accessed via related programs for context-sensitive pricing and discounts.

---

## Notable Patterns and Non-Obvious Logic

- **Recalculation Loops**: The program is designed to re-enter the main calculation/display loop (`xc1bld`) after any change, ensuring that all dependent fields are always in sync.
- **Unit Conversion Logic**: Handles all possible transitions between three units, including recalculation of quantities and prices, using a consistent pattern in `kalkulator` and `enhet`.
- **Discount Application**: Discounts are always applied multiplicatively and are reflected in both the sales price and margin calculations.
- **Margin Calculation**: The margin (dekningsgrad) is recalculated after all discounts, and is capped within a defined range to avoid unrealistic values.
- **Logistics Item Special Handling**: Items flagged as logistics items trigger additional SQL lookups and NOBB integration.
- **Function Key Mapping**: Business operations are mapped to function keys, with clear separation of concerns for each related data area (product info, price conditions, customer/project info, etc.).

---

## Summary Table of Key Variables

| Variable   | Meaning (Norwegian)      | English Equivalent           |
|------------|-------------------------|------------------------------|
| sapr       | Salgspris               | Sales price                  |
| kopr       | Kostpris                | Cost price                   |
| dekn       | Dekningsgrad            | Margin/coverage              |
| rab1/rab2  | Rabatt                  | Discount (level 1/2)         |
| ntpr       | Nettopris               | Net price                    |
| inpr       | Innkjøpspris            | Purchase price               |
| anta/ant1/ant2/ant3 | Antall         | Quantity (various units)     |
| enh1/enh2/enh3 | Enhet               | Unit (1/2/3)                 |
| grpr       | Grunnpris               | Base price                   |
| ifak/kfak/sfak/ffak | Faktor         | Factor (various types)       |
| kbel       | Beløp                   | Amount                       |
| bvar/pvar  | Varenr                  | Item number (base/price)     |
| prgr       | Prisgruppe              | Price group                  |
| lety       | Leveringstype           | Delivery type                |
| nobb       | NOBB-nr                 | NOBB number                  |
| logi       | Logistikk-vare          | Logistics item flag          |

---

## Tips for New Developers

- **Understand the domain language:** Many variables and comments use Norwegian abbreviations. Reference the summary table above and ask domain experts as needed.
- **Follow the calculation flow:** When troubleshooting, follow the sequence: input → validation → calculation → display, noting that recalculation is triggered by almost any change.
- **Use the function keys:** Each F-key triggers a distinct business operation; be familiar with their mappings and the external programs they invoke.
- **Integration points matter:** Pricing, conditions, and NOBB logic are handled by other programs—be aware of their interfaces and parameter lists.
- **Test multi-unit scenarios:** Always verify that calculations remain correct when switching between units or applying discounts.

---

## Conclusion

JV171R is a comprehensive and central program for item and price inquiry/calculation, supporting advanced pricing scenarios, multi-unit handling, and deep integration with price conditions and product databases. Its design reflects years of incremental business-driven enhancement, and familiarity with its structure will provide a solid foundation for working with the ASVADM system’s pricing and item logic.