The document has been written to `/home/runner/work/RPG-LLM/RPG-LLM/NexStep_JV_Item_Product_Maintenance_Business_Rules.md`.

It covers all 12 required sections, drawn directly from JFRF.MBR field definitions and 24 RPG programs (JV100R through JV190R). Key content per section:

1. **Introduction** — program inventory table, all table/prefix mappings, Norwegian terminology glossary
2. **Database Schema** — full field tables for JVARPF, JVDTPF, JLEVPF, JW packaging, JX pricing, JVL supplier links with types and Norwegian names from JFRF.MBR
3. **Prerequisites** — product groups (VUGRPF), units (VENHPF), categories (JKATPF), mandatory supplier (JLEVPF), NOBB uniqueness/digit rules, AVALPF field-level permissions, item number auto-generation from JSTSPF counter
4. **Validation Rules** — from JV101R: uniqueness, description mandatory, NOBB type codes (ST/SS/SP/DS), status values (0/1/2/4), date format checks, under-construction, JV113R alternative number rules
5. **Classification** — three-level hierarchy enforcement logic (indicators *in31/*in32/*in34/*in36), own-group overrides, assortment fields, delivery type, category responsible, JVDOKI
6. **NOBB Integration** — NOBB date fields, replacement NOBB, NRF via JV115R/JVN table, TUN/FINFO via JV113R/JVB table, EAN, producer, module
7. **Pricing** — JX table fields and validity logic, price unit, price/procurement/hold codes per company (JVDTPF), JV150R/JV151R bulk price code change rules, JV125R dual-date logic and all bulk update validations
8. **Packaging** — JW table: orderable flag, gross/net dimensions, GTIN, quantity per package, pallet fields, PSE, packaging class, JZ outer packaging defaults, conversion factors
9. **Supplier** — JVL link table, product owner flag, supplier item number, discontinuation date logic, markings, JVM media, primary supplier validation, passive/suspended supplier blocking
10. **Environmental** — dual environmental marks with country codes, UN/hazard numbers, packaging marking, PSE, frost tolerance, expiry date tracking, tariff codes, approval status, brand
11. **Lifecycle** — status transitions, active date visibility gate, discontinuation blocking, under-construction lock, stockholding flag, JVFFPF file group release, JVK/JVQ logistics links, JVO cross/up-sell, JV180R deleted items, JV190R properties
12. **Field Reference** — complete field-by-field tables for JVARPF (57 fields) and JVDTPF (8 fields) with Norwegian names, types, sizes, and descriptions
