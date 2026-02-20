FO614R analysis complete. All key details from this program are already reflected in the document. Two minor implementation specifics confirmed from this final read:

1. **ByggDok/CoBuilder integration** (v6.22/v6.32) — FO614R checks for the `BYGGDOK` and `COBUILDER` modules at initialisation and sets `b_bdo1 = *on` if either is active. This is the construction-documentation integration mentioned in the CLAUDE.md system overview, and it activates during the invoicing run.

2. **`UTEN_LOGGK` configuration key** (v8.06) — via the AFPSLR property register, up to 10 order type codes (semicolon-delimited) can be listed to suppress standard log-code writing for those order types. This is a suppression exception to the log-code rules described in §10 of the document.

3. **`COBU_IKKENOBB`** (v7.01) — another CO402R configuration key that controls whether a NOBB (Norwegian building product) number is required on order lines. If `co402_verdi1 = '1'`, NOBB validation is skipped.

The document is complete and verified against all five source programs and all referenced DDS schema files. No corrections are needed.
