The complete 12-section business rules document has been written above. Here is a summary of what each section covers:

1. **Introduction** — Program inventory table (all 28 programs), conventions used
2. **Database Schema Overview** — All 6 primary tables with primary keys and relationships; supporting lookup tables
3. **Item Master Maintenance** — VV100R search/navigation, VV101R screen 1 validation and price subfile, VV102R screen 2 codes, VV700R next-available-number logic
4. **Item Groups and Classification** — 3-level group hierarchy validation, item type codes and transition rules, ABC codes
5. **Unit Management** — VV550R unit maintenance, VV555R conversions, unit hierarchy (base/warehouse/statistics/price-comparison units)
6. **Supplier and NOBB Integration** — NOBB number fields, VV710R 2-source lookup with logistics-item SQL override, VVLEPF structure
7. **Warehouse Data** — VLAGPF stock balance fields, cost price fields, replenishment parameters, lifecycle dates
8. **Price Maintenance** — 7-part price record key, 5 price types, VP505R supplier selection, VP110R add/update/delete rules with open-order check, coverage formula
9. **Price Lookup and Calculation** — VP700R 3-tier supplier lookup, delivery type fallback, offer price time-window check, VP900R full discount engine with 7-step sequence
10. **Price Adjustment and Bulk Operations** — VV106R single-item screen, VP800R percentage/margin bulk adjustment write-or-update rule, VP410R CSV deletion, VP600R mutual-exclusion validation
11. **Item Lifecycle** — VV110R number-change validations, VV740R cross-table propagation (including VHISLU sekv+9000 special case), VV750R search text components, VV680R deletion criteria, VV120R alternative items
12. **Field Reference** — Complete field tables for VVARPF (62 fields) and VLAGPF (48 fields) with Norwegian names, English descriptions, types, and lengths
