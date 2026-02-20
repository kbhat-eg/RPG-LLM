I now have enough information from FO616R.MBR to produce the full analysis. Let me compile the documentation.

---

# Invoice Processing Business Rules
## Programs: FO616R (Invoice Engine) and FO615R (Invoice Line Processing)
## Primary Tables: SOHEPF (Invoice Header) and SODTPF (Invoice Lines)

> **Note:** The `FO615R.MBR` source stored in the repository contains the project final-invoicing helper (FO614RMG). All invoice line processing logic documented below was read directly from `FO616R.MBR` (NXCLOUD/rpgsrc/FO616R.MBR), which contains both the engine and all line-level subroutines.

---

## Key Architecture Notes

Before the rules, the in-code documentation at `FO616R.MBR:852` states three design principles that underpin everything:

1. **All amounts in SOHEPF and SODTPF are stored excluding VAT.** VAT is calculated and posted to the GL but is not stored in the invoice line amounts.
2. **One invoice can span multiple orders.** The L2 control break marks the end of one order; L6 marks the end of the invoice. Fees and surcharges are always posted against the last order on the invoice.
3. **All amounts in the order register carry positive signs.** Sign reversal for credit notes is applied inside this program, based on the order type's `VAOKRE` flag and the line type's `VALKRE` flag.

---

## Product and Pricing Rules

### Rule: Item Lookup on Each Invoice Line

**What is checked:** For every product line (`fkskod='V'`), the program looks up the item number (`FODTPF.FDVARE`) in the item master `VVARPF`. This is done with a direct key read using firm + item number.

**Why this is required:** Account codes, VAT codes, product group hierarchy, and statistical unit all come from `VVARPF`. Without this data the line cannot be correctly posted to the GL or statistics.

**What happens when the item is not found:** Subroutine `x_bvare` (`FO616R.MBR:3446`) is called. It blanks all item-master fields and, critically, sets `vvkmva = 0` (no VAT). This prevents a crash but means the line posts without product-group coding. A missing item does **not** block the invoice.

**Fields involved:**
| Field | Table | Meaning |
|---|---|---|
| `VVARPF.VVVARE` | Item master | Item number key |
| `VVARPF.VVKMVA` | Item master | Item VAT flag (0=standard, 1=exempt) |
| `VVARPF.VVOGRP / VVHGRP / VVUGRP` | Item master | Product group hierarchy |
| `VVARPF.VVLVAR` | Item master | Supplier item number |
| `VVARPF.VVENHS` | Item master | Statistics unit |

---

### Rule: Gross Price Calculation

**What is checked:** The gross invoice amount for a line is calculated from the unit selling price (`FODTPF.FDSAPR`) multiplied by quantity (`FODTPF.FDANTA`). Half-adjust rounding (`eval(h)`) is applied.

**Formula:**
```
w_belb  = FODTPF.FDSAPR × FODTPF.FDANTA   (gross invoice amount)
w_kost  = FODTPF.FDKOPR × FODTPF.FDANTA   (cost amount)
w_belo  = FODTPF.FDOPPR × FODTPF.FDANTA   (original list price amount)
```

**Real-world example:** Item at NOK 1,200.00/unit, ordered qty 3 → `w_belb = 3,600.00`. If the original list price was NOK 1,400.00, then `w_belo = 4,200.00`.

**Code location:** `FO616R.MBR:2448`

---

### Rule: LVR (Special Catalogue) Pricing

**What is checked:** When `FODTPF.FDLVRE = 1` (line sourced from LVR/supplier catalogue), pricing follows a different path. The program starts from the NOBB net price (`FODTPF.FDNOPR`) and applies purchase factor (`FODTPF.FDBIFA`) and sales factor (`FODTPF.FDPSFA`).

**What happens when:**
- `FDBNTP <> 0` and `FDBIFA = 1` → use `FDBNTP` directly as the base (it's already a net price)
- Otherwise → `base = FDNOPR × FDBIFA`, then `selling = base × FDPSFA`

**Code location:** `FO616R.MBR:2400`

---

### Rule: Quantity Sign Handling

**What is checked:** All quantities in the order register are stored as positive. Sign reversal happens here based on:
- Order type flag `VOTYPF.VAOKRE = '-'` → the entire order is a return/credit
- Line type flag `VLTYPF.VALKRE = '-'` → this specific line is a return

**What happens:** When `w_dbkr = 'K'` (credit), all amounts are multiplied by -1 before accumulation: `fdanta`, `fdsums`, `fdsumk`, `w_beln`, `w_belb`, etc.

**Code location:** `FO616R.MBR:2530`

---

## Tax/VAT (MVA) Rules

### Rule: VAT Code Mapping via VMVAPF

**What is checked:** Each line carries an invoice-specific VAT code in `FODTPF.FDOMVA`. For fee lines (gebyrlinjer), the program looks up the item's internal VAT flag (`VVARPF.VVKMVA`) in `VMVAPF` and maps it to the correct output VAT code.

**Why this is required:** The item master stores a simplified VAT flag (0=standard, 1=exempt, 2=reduced rate). The invoice must carry the formal Norwegian VAT code (e.g., '3'=25%, '4'=exempt, 'H'=15%).

**How the lookup works (`FO616R.MBR:3313`):**
- Key into `VMVAPF` using `vvkmva` as `VMVAPF.VAMKOD`
- Found: use `VMVAPF.VAMKOF` as the standard MVA code. If `fomkod=2` (special) and `VMVAPF.VAMKUO` is not blank, use `VAMKUO` instead
- Not found: fall back to: `vvkmva=1 → '4'` (exempt), `vvkmva=2 → 'H'` (15% food rate)

**Fields:**
| Field | Table | Meaning |
|---|---|---|
| `VMVAPF.VAMKOD` | VAT table | Item VAT code (key) |
| `VMVAPF.VAMKOF` | VAT table | Standard invoice VAT code |
| `VMVAPF.VAMKUO` | VAT table | Override VAT code for special order types |

---

### Rule: VAT Rate Retrieval

**What is checked:** For every line where `FODTPF.FDOMVA` is not blank, the program calls subprogram `RS205R` to retrieve the current VAT rate as of the **invoice date** (`l_fkda`, not the order date).

**Why the invoice date matters:** As noted in the change log at `FO616R.MBR:28`, this was corrected in R3.31. Using the invoice date ensures that when VAT rates change on 1 January, invoices created before year-end use the old rate even if they were not yet printed.

**Parameters passed to RS205R:**
- VAT code (`w_mvak = fdomva`)
- Invoice date (`w_dato = l_fkda`)
- Returns: `w_mvap` (percentage, e.g., 25.00), `w_mvat` (decimal rate, e.g., 0.2500), `w_mvaf` (factor, e.g., 1.25000), `w_bpro` (basis percentage)

**Code location:** `FO616R.MBR:2519`

---

### Rule: VAT Base Amount Calculation

**What is checked:** The VAT base (`t_mvag`) is calculated differently depending on two system configuration flags (`faasa1` and `faasao`) stored in the invoice status register (FSTSPF).

**The four combinations:**

| `faasa1` | `faasao` | VAT base is... | Example (gross 1000, line disc 50, order disc 30) |
|---|---|---|---|
| 0 | 0 | `w_belx` = gross − line discounts − order discount | 1000 − 50 − 30 = **920** |
| 1 | 0 | `w_belb − w_rkro` = gross − order discount only | 1000 − 30 = **970** (line discounts go to separate GL account) |
| 0 | 1 | `w_beln` = gross − line discounts only | 1000 − 50 = **950** (order discount on separate GL account) |
| 1 | 1 | `w_belb` = gross only | 1000 (all discounts on separate GL accounts) |

**Important exceptions — VAT base is forced to 0 when:**
- `FODTPF.FOMKOD = 1` (order marked as VAT-exempt)
- `VVARPF.VVKMVA = 1` (item is VAT-exempt)
- Computed VAT percentage `w_mvap = 0`
- VAT code is `'4'`, `'0'`, or `' '`

**Code location:** `FO616R.MBR:2486`

---

### Rule: VAT Amount Calculation and Rounding

**What is checked:** VAT amount is calculated using the MVA base excluding all public discounts (bruksrett discounts are never in the VAT base):

```
w_mgrl = w_belb - w_rkr1 - w_rkr2 - w_rkro - w_fosw
w_moms = ROUND_HALF(w_mgrl × w_mvap / 100)
```

The `eval(h)` operator applies half-adjust rounding (standard Norwegian accounting rounding).

**Why bruksrett discounts are excluded:** Usage-rights discounts (`fdbrr1`, `fdbrr2`) are a legal arrangement between the customer and a forest owner's cooperative (almenning). By Norwegian tax law, these reduce the price before VAT is computed but are not part of the commercial discount structure. VAT code is set to `'0'` for the bruksrett posting. (`FO616R.MBR:2806`)

**Accumulated VAT per invoice:** `w6moms` accumulates across all lines within the invoice (L6 break) and is added to `w6fsum` when computing the invoice total at `FO616R.MBR:1346`.

---

## Discount Rules

### Rule: Line Discount Cascade

**What is checked:** Line discounts are applied in a fixed sequence. All percentages come from the order line (`FODTPF`).

**Calculation sequence (`FO616R.MBR:2453`):**

| Step | Variable | Formula |
|---|---|---|
| 1. Gross amount | `w_belb` | `FDSAPR × FDANTA` |
| 2. Line discount 1 | `w_rkr1` | `w_belb × FDRAB1 / 100` |
| 3. Net after disc 1 | `w_belw` | `w_belb − w_rkr1` |
| 4. Line discount 2 | `w_rkr2` | `w_belw × FDRAB2 / 100` |
| 5. Net line total | `w_beln` | `FDSUMS − w_rkr1 − w_rkr2` |
| 6. Order discount base | `w2rbgp` | accumulates `w_beln` (when `vvkord=0`) |

**Real-world example:** Item NOK 500, qty 2 = NOK 1,000 gross. Disc1 = 5% → w_rkr1 = 50. Net = 950. Disc2 = 2% → w_rkr2 = 19. Net line = 931.

---

### Rule: Order Discount (Ordrerabatt)

**What is checked:** The order-level discount (`FOHEPF.FORABP`) is applied to each line's net amount after line discounts. Lines with `VVARPF.VVKORD = 1` are excluded from the order discount base.

**Formula per line (`FO616R.MBR:2473`):**
```
if vvkord = 0:
   w_rkro = ROUND_HALF(w_beln × FORABP / 100)
```

**At order break (L2), the total order discount is recalculated (`FO616R.MBR:1234`):**
```
sorkop = ROUND_HALF(w2rbgp × forabp / 100)
```
This recalculation prevents accumulated rounding errors from per-line calculations.

---

### Rule: Public/Bracket Discounts (Almenningsrabatt / Bruksrett)

**What is checked:** `FODTPF.FDBRR1` and `FDBRR2` carry percentage rates for public discounts (almenning = forest cooperative discount, bruksrett = usage rights).

**Calculation (`FO616R.MBR:2478`):**
```
w_belx = w_belw - w_rkr2 - w_rkro
w_brk1 = ROUND_HALF(w_belx × FDBRR1 / 100)
w_bely = w_belx - w_brk1
w_brk2 = ROUND_HALF(w_bely × FDBRR2 / 100)
```

**Account lookup for public discounts:** When `faasa2 = 1`, the program looks up the almenning code (`FODTPF.FDALME`) and bracket type (`FODTPF.FDBRTY`) in `RMBTPF` to find the specific GL account to post the discount to. If not found, falls back to the standard discount account `d_kko6`.

**Special handling for bruksrett:** Change log entry R5.77 (`FO616R.MBR:119`) confirms bruksrett discounts must not carry a VAT code (`t_mkod = '0'`) and must not enter the VAT base.

---

### Rule: Customer Discount Update (Bruksrett Limit Tracking)

**What is checked:** When an order contains bruksrett discounts, the customer record in `RKUNPF` (via RKMEPF, the customer memo table) is updated with the accumulated discount amount (`RKUNPF.RKKJAL`).

**Why this is required:** There is a legal annual limit on how much bruksrett discount a customer may receive. The field `RKUNPF.RKGRAL` holds the limit; `RKKJAL` holds the year-to-date total. The invoicing program accumulates this for each invoiced order (`FO616R.MBR:2003`).

---

## Deposit and Prepayment Rules

### Rule: Deposit Deduction from Invoice

**What is checked:** Before finalizing the invoice total, the program looks up the order number in `SDEPPFR` (deposit register) using the key: firm + order number + customer + counter `00`.

**The three deposit fields in SDEPPFR:**
| Field | Meaning |
|---|---|
| `SDEPPFR.SKDEPO` | Original deposit amount invoiced |
| `SDEPPFR.SKDEPB` | Deposit already billed (invoiced to the customer) |
| `SDEPPFR.SKDEPT` | Deposit already deducted from final invoices |

**Calculation (`FO616R.MBR:1455`):**
```
w_trek = skdepb - skdept - w_sumim
```
where `w_sumim` is the VAT amount on the current invoice's lines (for 100% prepayment orders).

**Cap rule:** `w_trek` cannot exceed `w6fsum` (the invoice total). If the deposit exceeds the invoice, only the invoice amount is deducted — the remainder stays in the deposit register for future invoices.

**What happens after deduction:** `SDEPPFR.SKDEPT` is incremented by `w_trek`. The order header `SOHEPF` is updated with `SODEPO`, `SODEPB`, and `SODEPT`.

**Important:** If a deposit record is found, rounding (`øreavrunding`) is **skipped entirely** for this invoice (`FO616R.MBR:1357`).

---

### Rule: Prepayment (Forhåndsbetalt) Settlement

**What is checked:** For fully prepaid orders (`sk100p = 1`), the program calls `FO704R` to compute the current order total (including and excluding VAT). If the order total has changed since the deposit was billed (`w_sumuf <> sktotu`):

```
w_fosa = w_sumuf - sktotu   (change excl. VAT)
w_fosi = w_sumif - skdepb   (change incl. VAT)
```

**Distribution across lines:** The settlement amount (`w_fosa`) is distributed proportionally across invoice lines in proportion to each line's share of total sales (`fdsums / w_sums`). This shows as `SODTPF.SDFOSA` per line.

**Code location:** `FO616R.MBR:4716` (subroutine `finn_forh`)

---

### Rule: Deposit Invoice vs. Final Invoice Type

**What is checked:** The invoice type field `FOHEPF.FOFATY` (invoice type, e.g., 10=standard, 20=EHF/electronic, 21=EHF credit note) is read from the order header. Change log entry R5.67 (`FO616R.MBR:115`) documents that when `fofaty = 20` and the order is a credit note, the type is automatically changed to `21` (EHF credit note format).

**Condition:** `vaokre = '-' and fofaty = 20 → fofaty = 21`

---

## Total Calculation Rules

### Rule: Order-Level Accumulation (L2 Break)

**What is checked:** At the end of each order (L2 break), the program writes the order header to `SOHEPF` with accumulated totals. The critical totals come from the w2 working variables rather than from the order header directly.

**Why:** The order header totals (`fotots`) include VAT; the invoice register must store amounts **excluding** VAT. The program builds clean totals in the w2 variables during line processing.

**Fields written to SOHEPF at L2 (`FO616R.MBR:1216`):**
| SOHEPF Field | Source | Meaning |
|---|---|---|
| `SOSALP` | `w2salp` | Sum of sales amounts (excl. VAT) |
| `SORK1P` | `w2rk1p` | Sum of line discount 1 |
| `SORK2P` | `w2rk2p` | Sum of line discount 2 |
| `SORKOP` | recalculated | Order discount (recalculated to avoid rounding drift) |
| `SOKSTP` | `w2kstp` | Sum of cost |
| `SORB1P` | `w2rb1p` | Public discount 1 (almenning) |
| `SORB2P` | `w2rb2p` | Public discount 2 |
| `SOMVGP` | `w2mvgp` | VAT base amount |

---

### Rule: Invoice Total Calculation (SOFSUM)

**What is checked:** At the L6 (invoice) break, the gross invoice total is computed from the accumulated L6 variables:

```
w6fsum = w6salp                 (total sales)
       - w6rk1p                 (less line discount 1)
       - w6rk2p                 (less line discount 2)
       - w6rkop                 (less order discount)
       - w6rb1p                 (less public discount 1)
       - w6rb2p                 (less public discount 2)
       + w6geby                 (plus fees)
       + w6pasl                 (plus surcharges)
       + w6moms                 (plus VAT)
```

This final value is stored in `SOHEPF.SOFSUM`. (`FO616R.MBR:1333`)

---

### Rule: Rounding to Nearest Whole Krone (Øresavrunding)

**What is checked:** The decimal portion of `w6fsum` (`whlp20`) is examined to determine rounding.

**Current behavior (from V8.26/V8.28):** Rounding is controlled by a switch `b_round`. When `b_round = *off` (the configured default for most customers), the logic rounds to the nearest whole krone:

- Positive invoice, decimal 0.01–0.49: round down (strip decimal)
- Positive invoice, decimal 0.50–0.99: round up (+1.00, strip decimal)
- Negative invoice: mirror logic

**When rounding is skipped entirely:**
- `b_round = *on` (switch-controlled per customer/site)
- A deposit record (`SDEPPFR`) exists for this invoice

**Rounding difference:** The difference `w6fsum - w6hsum` is posted as an `øreavrunding` (rounding difference) to the GL account configured in `FSTSPF.FAAVNR` with billing code `FSTSPF.FAABIL`. (`FO616R.MBR:1825`)

**Zero-value invoice:** There is no explicit block for zero-value invoices. They process normally. The rounding section (`xtag83` tag) is reached directly at zero, so no rounding is applied.

---

## Payment and KID Rules

### Rule: Payment Terms and Due Date

**What is checked:** If the due date (`FOHEPF.FOFFDA`) is not already set, subrogram `FÅ703R` is called with: firm, payment terms code (`FOHEPF.FOBETB`), and invoice date (`FOHEPF.FOFKDA`). The subprogram returns the calculated due date.

**Credit note rule:** For credit notes (`vaokre = '-'`), the due date is always set equal to the invoice date — meaning the credit is immediately available for settlement. (`FO616R.MBR:1978`)

**Collective invoice rule (V8.24):** When collective invoicing batches both invoices and credits, the credit-note due-date rule only fires if the invoice total (`faksum`) is negative.

---

### Rule: KID Number Generation

**What is checked:** After the invoice total is written to `SOHEPF`, the KID (customer payment ID) is assembled and a checksum digit is calculated.

**Standard KID structure (21 digits):**
```
[firm 3] [code 1] [type 1] [lead 1] [customer 6] [invoice# 8] [checksum 1]
```
Example: firm=001, customer=123456, invoice=00001234 → assembled string, then AK720R adds MOD10 digit.

**Short KID (invoice number only, 23 digits):**
- `d_kfak_2 = invoice number`
- Assembles invoice number with prefix from configuration
- MOD10 checksum via AK720R

**SAP KID (13 digits):**
- Only used when `w_sap_kid = 'J'`
- Format: firm text (4 chars) + invoice number (8 digits) + checksum (1 digit)

**Factoring bank KIDs (when `u_803 = *on` and factoring company configured in RA34ST):**
| Bank | Format |
|---|---|
| DNB | fixed (1) + client (5) + invoice# (8) + checksum |
| SB1 | fixed1 (1) + line (4) + invoice# (8) + fixed2 (2) + checksum |
| Nordea | customer (7) + line (4) + invoice# (7) + fixed (1) + checksum |
| NordeaNY | line (5) + fixed (1) + invoice# (8) + checksum |
| Svea | line (5) + fixed (2) + invoice# (8) + checksum |

**All variants call AK720R** for checksum calculation, then call **AK700R** to store the KID-to-invoice cross-reference in the KID reference register (AKIDPF). (`FO616R.MBR:1606`)

**Result stored:** `SOHEPF.SOKIDD` (KID string), `SOHEPF.SOKIDR` (reference number).

---

## Configuration and System Rules

### Rule: Accounting Period Validation

**What is checked:** The accounting period is read from the **Local Data Area (LDA)** at positions 561–566 (`l_peri`, format YYYYMM). This is set before FO616R is called and is not validated inside this program.

**Why the LDA is used:** The invoicing run is a batch job. The period is set once per run in the LDA by the calling job control program, ensuring all invoices in the same batch are posted to the same period.

**Period conversion:** The 6-digit LDA period (YYYYMM) is split into a 2-digit year prefix (`w_eaar`) and 2-digit month (`w_emnd`), then reassembled as `t_peri` (YYMM format) for GL posting to `FOVFPF`. (`FO616R.MBR:4287`)

---

### Rule: Invoice Date Handling

**What is checked:** The invoice date (`l_fkda`) is also in the LDA (positions 561–570, ISO date format). If the order header does not already have an invoice date (`fofkda = *loval`), it is set from the LDA value. (`FO616R.MBR:1955`)

**Why:** Some orders can have a manually overridden invoice date (e.g., for backdating). If no override is present, the batch run's date is used.

---

### Rule: Invoice Number Generation

**What is checked:** For each new invoice (L6 break), a new invoice number is fetched from the number register.

**Process (`FO616R.MBR:4315`, subroutine `x_hntnum`):**
1. Calls `VA752R` with firm and order type to check for a number series override (`w_onum`). This allows certain order types to draw from a different number series than the default.
2. Calls `AS100R` with:
   - `w_kode = '0'` (request a new number — increment and return)
   - `w_firm` (company)
   - `w_fell = faafak` (field name in number register, from FSTSPF)
   - `w_type` (order type, or overridden type)
3. Returns `w_numm` — the new 8-digit invoice number.

This number is stored as `SOHEPF.SOBINR` (invoice number) and `SODTPF.SDBINR`.

---

### Rule: Order Type Configuration Read

**What is checked:** At the start of each new invoice (x_fcycle, `FO616R.MBR:4240`), the order type register (`VOTYPF`) is read for the current order type. Key flags used throughout processing:

| VOTYPF Field | Meaning | Used for |
|---|---|---|
| `VAOKRE` | Credit indicator (`-` = credit) | Sign reversal |
| `VAOBIL` | Document type code | GL posting code |
| `VAOOKO` | Post to GL (1=yes) | Guards all GL writes |
| `VAOSTA` | Post to statistics (1=yes) | Guards stat writes |
| `VAOAKK` | Accumulator type | Controls number allocation |

---

### Rule: Fee (Gebyr) Generation

**What is checked:** After all order lines are processed (at L6 break, `FO616R.MBR:1289`), the program checks the fee register (`FGEBPF`) for applicable fees. The lookup uses an 8-step cascade:

```
1. Order type + customer category + invoice type
2. Order type + 0000 category + invoice type
3. Order type + customer category + 00 type
4. Order type + 0000 + 00
5. blank type + customer category + invoice type
6. blank type + 0000 + invoice type
7. blank type + customer category + 00
8. blank type + 0000 + 00
```

**Fee is applied when:** The fee's threshold amount (`FGEBPF.FGGREN`) is greater than or equal to the invoice total for the applicable base type (`fgtype` 0–3 = different base combinations).

**Exception:** Customers with card type 4 (`rukrl5` lookup) are exempt from invoice fees entirely.

**Fee amount:** If `FGEBPF.FGSATS <> 0`, the fee is a percentage (`fgbelp = w_geby × fgsats / 100`). Otherwise it is a fixed amount.

---

## Subprogram Calls Summary

| Called Program | Purpose | Called From |
|---|---|---|
| `RS205R` | Retrieve VAT rate/factor for a VAT code on a given date | Line processing, prepayment calc |
| `AK720R` | Calculate MOD10 checksum digit for KID | KID generation |
| `AK700R` | Store KID-to-invoice cross reference in AKIDPF | KID generation |
| `VA720R` | Resolve GL account/department/specification from product groups | Every invoice line and fee line |
| `VA752R` | Check for order-type number series override | Invoice number generation |
| `AS100R` | Get next number from number register (invoice sequencing) | Invoice number generation |
| `FÅ703R` | Calculate due date from payment terms and invoice date | Order header processing |
| `FÅ720R` | Resolve department from warehouse code | Line-level warehouse override |
| `RS701R` | Retrieve billing document text from billing code | Every GL posting |
| `VL710R` | Determine price group for statistics | Statistics record |
| `VP905R` | Extended statistics / price category | BM/Maxbo statistics |
| `VG710R` | Find product manager from product group hierarchy | Statistics record |
| `VG701R` | Average sliding cost price at a specific delivery date | Statistics cost price |
| `FO704R` | Compute order total (incl./excl. VAT) | Prepayment settlement |
| `CO402R` | Read configuration parameter (cost allocation switch) | Cost redistribution |
| `AX711R` | Calculate ISO week number from a date | Statistics week field |

---

## Critical Blocking Conditions

The following conditions cause the invoice processing to stop or produce incorrect output. They are **not** soft warnings — they result in either abrupt termination or silently wrong financial data:

| Condition | Effect | Where |
|---|---|---|
| `VOTYPF` record not found for order type | Order type flags are blank/zero; all GL writes are suppressed (`vaooko=0`), no accounting entries are generated | `FO616R.MBR:4281` |
| `FSTSPF` (invoice status register) not found | Configuration flags (`faasa1`, `faasa2`, `faasao`, `faavnr`, etc.) default to zero; rounding and discount GL accounts may be wrong | `FO616R.MBR:4270` |
| `AS100R` fails to return a valid number | `w_numm = 0`; invoice number is zero; the invoice header is written with `SOBINR=0`, making it unrecognizable | `x_hntnum` |
| `RS205R` returns `w_mvap=0` for a taxable VAT code | VAT is not computed; invoiced amounts exclude VAT incorrectly | `FO616R.MBR:2509` |
| `FODTPF` line not found (chain miss) | The line is skipped silently; it will not appear on the invoice or in statistics | `FO616R.MBR:932` |
| Deposit `SDEPPFR.SKDEPT` already equals `SKDEPB` | `w_trek = 0`; no deduction happens even if customer already paid. Invoice is sent for full amount | `FO616R.MBR:1458` |
