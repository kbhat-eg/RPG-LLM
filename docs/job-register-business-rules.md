# Job Register Business Rules

**Module**: Job Register / Price Rounding / Customer Creation (FA prefix)
**System**: ASOFAK
**Source files analyzed**: FA909R, FA910R, FA911R, FA912R, FA913R, FA920R, FA922R, FA924R, FA930R, FA931R, FA932R, FA933R

---

## 1. Prerequisites / Master Data Requirements

The FA module handles price rounding calculations (FA909R/FA910R/FA911R) and customer/job register maintenance (FA920R+, FA930R+). For rounding to execute and for customer creation to succeed, the following must exist:

| Requirement | Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|---|
| Firm status record must exist | Firm Config Required | FA910R chains FSTSPF with key firm; if not found (*in90): p_bel1 returned unchanged, no rounding | FSTSPF | FSFIRM | Not found → rounding bypassed |
| Price code must be 1–4 | Valid Price Code | FA910R: if p_kode not in 1,2,3,4: goto avslutt | Input | p_kode | Invalid code → no rounding |
| Price amount must be non-zero | Non-Zero Amount | FA910R: if p_bel1 = 0: goto avslutt immediately | Input | p_bel1 | Zero amount → no rounding applied |
| Rounding code must be non-zero | Rounding Code Set | FA910R: after reading FSTSPF, the rounding code for the given p_kode is checked; if =0: goto avslutt | FSTSPF | FAAINK/FAAKOS/FAAEMV/FAAIMV | Code=0 → rounding disabled for that price type |
| Sub-group must not have no-rounding flag | Sub-Group Override | FA910R: if item's sub-group (VUGRPF.VGUAVR=1): goto avslutt — item excluded from rounding | VUGRPF | VGUAVR | =1 → rounding bypassed for this item |
| NOBB supplier must be in JLEVPF | Supplier Lookup (FA911R) | FA911R chains JLEVPF; if not found: skip processing, no rounding | JLEVPF | JLFIRM/JLKKAL | Not found → FA911R returns without rounding |
| Rounding rule must exist in JAV1PF | Rule Existence (FA911R) | FA911R searches 11-level hierarchy in JAV1PF; if no rule found: no rounding | JAV1PF | JA1RGL | Not found at any level → skip |
| Rounding rule must be non-zero | Rule Active | FA911R: ja1rgl = 0 means no rounding for matched rule | JAV1PF | JA1RGL | =0 → skip rounding |

---

## 2. Validation Rules

### FA910R — Core Price Rounding Engine

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Overflow Guard | v6.31: if rounded result >= 10,000,000: do not update p_bel1 (called via FA909R which has 9:2 field) | Derived | p_bel1 | Result >= 10M blocked from write-back |
| Price Threshold Assignment | FSTSPF fields faagr0–faagr4 define tier thresholds (e.g., 0, 100, 500, 2000, 10000); the tier is selected by comparing p_bel1 against thresholds | FSTSPF | FAAGR0–FAAGR4 | Amount falls in lowest applicable tier |
| Add-On Amount Per Tier | After tier selection, faabl1–faabl5 is the add-on amount added before rounding | FSTSPF | FAABL1–FAABL5 | Add-on applied before rounding |
| Rounding Method 1 = Round to Nearest | Rounding code 1: standard arithmetic rounding to the unit specified per tier | FSTSPF | FAAINK/FAAKOS/FAAEMV/FAAIMV | Code 1 applies nearest-rounding |
| Rounding Method 2 = Round Up | Rounding code 2: always round up (ceiling) to the next unit | FSTSPF | FAAINK/FAAKOS/FAAEMV/FAAIMV | Code 2 applies ceiling rounding |

### FA911R — VAT-Inclusive Sale Price Rounding (Byggmakker)

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Supplier Must Be NOBB-Type | FA911R checks JLEVPF.JLKKAL=1 and ja1nob=0; if both conditions met, supplier is a NOBB supplier that uses this rounding | JLEVPF | JLKKAL | Non-NOBB supplier → FA911R not applicable |
| 11-Level Hierarchy Lookup | JAV1PF is searched in order: lev+item, item, lev+module, module, ogr+hgr+ugr+lev, ogr+hgr+lev, ogr+lev, lev, ogr+hgr+ugr, ogr+hgr, ogr; first match wins | JAV1PF | JA1NOB/JA1ENH/JA1MOD/JA1OGR/JA1HGR/JA1UGR | Hierarchy determines which rule governs |
| Price Band Lookup in JAV3PF | After finding a rule in JAV1PF, the price is matched against bands in JAV3PF; each band has a rounding unit | JAV3PF | JA3FRA/JA3TIL/JA3RGL | Price outside all bands → no rounding |
| Price Computed Flag | v6.33/6.34: b_pris flag must be *on at end of rounding to confirm a price was actually calculated; if *off, result is not returned | Internal | b_pris | False → output price not set |

---

## 3. Configuration and Authorization Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Price Code Meanings | Code 1=purchase price, 2=cost price, 3=sale price ex VAT, 4=sale price incl VAT; rounding config per code stored in FSTSPF | FSTSPF | FAAINK/FAAKOS/FAAEMV/FAAIMV | Each code has its own rounding on/off flag |
| Sub-Group No-Rounding Flag | VUGRPF.VGUAVR=1 exempts all items in that sub-group from rounding; checked after item+sub-group lookup | VUGRPF | VGUAVR | =1 bypasses all rounding for items in sub-group |
| Next Number Assignment (FA920R) | FA920R calls AS100R with field code FAALSM to get the next available member number; 8-digit from v6.30 | FSTSPF/NUMREG | FAALSM | Determines new customer/job number |

---

## 4. Financial / Transactional Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Rounding Applied In-Place | FA910R modifies p_bel1 (the passed price) by reference; the caller receives the rounded value back | Input/output | p_bel1 | Zero or overflow → unchanged |
| FA909R Wrapper for 9:2 Format | FA909R provides a 9:2 decimal wrapper that calls FA910R with an 11:2 internal variable to avoid overflow; the 11:2 result is returned in a 9:2 parameter only if < 10,000,000 | FA910R | p_bel1 (11:2 internal) | Overflow guard at 10M |
| Cost Price vs Purchase Price | Code 2 (cost price) uses FAAKOS rounding flag; code 1 (purchase price) uses FAAINK; both go through the same tier logic but may have different on/off settings | FSTSPF | FAAINK/FAAKOS | Separate controls for purchase vs cost |
| Sale Price Including VAT (Code 4) | Uses FAAIMV rounding flag in FA910R; FA911R handles the Byggmakker VAT-inclusive rounding separately via JAV1PF rules | FSTSPF/JAV1PF | FAAIMV/JA1RGL | Two independent systems for code 4 |

---

## 5. Status and Lifecycle Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Rounding Result Returned or Unchanged | FA910R either rounds and returns p_bel1, or leaves it unchanged (on any blocking condition); caller cannot distinguish between "rounded to same value" and "not rounded" | FA910R | p_bel1 | Always returns a value; blocking = no change |
| FA930R Customer Creation | FA930R (73KB) handles customer creation including company number (foretaksnummer), bank giro, phone, email, mobile, and mobile operator validation; calls contact person and card programs on success | RKUNPF | Multiple | Customer record written on successful validation |
| Next Number from Number Register | FA920R/FA912R call AS100R to allocate the next member number before writing a new record; this ensures no duplicate keys | Number register | FAALSM/equivalent | AS100R returns next available number |

---

## 6. Special Conditions (Program-Specific)

### FA910R — Price Rounding

- Reads VVARPF to find the item's sub-group (VVUGRP); then reads VUGRPF to check VGUAVR.
- The 5-tier rounding uses price amounts as boundaries: tier 0 (lowest, < faagr0), then tiers 1–4 ascending.
- Each tier has both a threshold (faagr*) and an add-on amount (faabl*); the add-on is added before rounding to the nearest unit.
- The rounding unit itself is derived from the tier and is implicit in the FSTSPF configuration; it is not a separate stored field.

### FA911R — Byggmakker VAT-Inclusive Rounding

- This program is specifically designed for the Byggmakker customer (Norwegian hardware chain) NOBB-based price rounding.
- The 11-level hierarchy allows very specific rules (e.g., a specific item+supplier combination) to override broader rules (e.g., a whole over-group).
- JAV3PF bands define price ranges; within each band, a rounding unit (JA3RGL) specifies the increment.

### FA920R — Next Member Number

- Utility program; no validation logic. Calls AS100R with field code FAALSM and returns WPNUMM (8-digit from v6.30).
- Used by FA930R and other creation programs to get a unique sequential number.

---

## 7. Subprogram Calls Affecting Logic

| Caller | Callee | Purpose | Effect on Blocking |
|---|---|---|---|
| FA909R | FA910R | Core rounding; FA909R wraps 9:2 conversion | If FA910R returns >= 10M, FA909R does not update |
| FA910R | (none — direct file reads) | FSTSPF, VVARPF, VUGRPF | VGUAVR=1 blocks rounding |
| FA911R | (none — direct file reads + SQL) | JLEVPF, JAV1PF, JAV3PF | No matching rule = no rounding |
| FA920R | AS100R | Get next number from number register | Returns next available sequential number |
| FA930R | AS100R (via FA920R) | Customer number allocation | Determines new RKUNPF key |
| FA930R | Contact person program | Customer contact creation | Called after successful customer save |

---

## 8. Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| FSTSPF | Firm status / configuration | FSFIRM |
| VVARPF | Item master | VVFIRM, VVVARE |
| VUGRPF | Item sub-group register | VGFIRM, VGUGRP |
| JLEVPF | NOBB supplier register | JLFIRM, JLKKAL |
| JAV1PF | NOBB rounding rules (11-level hierarchy) | JA1FIRM, JA1NOB, JA1ENH, JA1MOD, JA1OGR, JA1HGR, JA1UGR |
| JAV3PF | NOBB price band rounding parameters | JA3FIRM, JA3RGL, JA3FRA, JA3TIL |
| RKUNPF | Customer register | RKFIRM, RKKUND |
