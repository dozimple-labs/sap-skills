# Remediation patterns in detail

Patterns verified by re-running ATC with the S/4HANA readiness variant after the change. Always confirm the
note text in your own system: notes are updated and your release may differ.

## Amount field length extension (note 2610650)

`TYPE p LENGTH n` counts **bytes**, not digits: digits = 2 × n − 1. `LENGTH 9` holds 17 digits; the extended
amount (`DMBTR` in S/4HANA) needs `LENGTH 12` (23 digits).

```abap
" Before: narrow variable receiving DMBTR in several branches
DATA lv_total TYPE p LENGTH 9 DECIMALS 2.

" After: anonymous packed type wide enough for the extended amount
DATA lv_total TYPE p LENGTH 12 DECIMALS 2.
```

- Prefer the anonymous packed type over `TYPE dmbtr`. Typing with the data element propagates its identity
  into public types and can trigger new findings in consumers.
- `CONV dmbtr( ... )` on a value that is already `DMBTR` raises "redundant conversion": it does not help.
- Widen every variable the value flows through. Otherwise a new warning appears downstream.

### `WRITE TO` type conflict

The extended amount formats into 30 characters.

- Local text target: widen it to 30, and the variables it is concatenated into.
- Field of a DDIC structure that defines a shared file layout: do not widen, suppress with
  `"#EC CI_FLDEXT_OK[2610650]` and record why.

### `move length conflict` into a Z table field

Closing the finding means changing the DDIC and converting the table, and that conversion travels to
production. Measure first:

```sql
SELECT MAX( amount_field ) FROM ztable
SELECT fieldname, rollname, leng, decimals FROM dd03l
  WHERE tabname = 'ZTABLE' AND as4local = 'A' AND datatype = 'CURR'
```

A margin of three orders of magnitude between the real maximum and the ceiling means the truncation is latent,
not active. Extending is still legitimate if your project already does it; if so, see the next section.

### Extending a Z amount data element

Change the data element's domain to the standard `AFLE13D2O16N_TO_23D2O30N` (CURR 13 → 23) rather than
redefining the data element.

- Look for all users first: `SELECT tabname FROM dd03l WHERE rollname = 'ZMY_AMOUNT'`, then `DD02L-TABCLASS`.
  If all users are structures (`INTTAB`), one dictionary change closes every finding with no conversion.
- Tables with data need a conversion. Activating from ADT may stop with "dependent table must be converted";
  activating in SE11 and accepting the adjustment converts while keeping the data. Verify afterwards with
  `DD03L` (active version, new length), an empty `TBATG` for the table, and the row count.
- Test the conversion in quality with a copy of production data.

## SD document category VBTYP (note 2198647)

| Field | Data element in S/4HANA | What old code usually declares |
|---|---|---|
| `VBAK-VBTYP` | `VBTYPL` CHAR 4 | `TYPE vbtyp` CHAR 1 |
| `VBFA-VBTYP_N` | `VBTYPL_N` CHAR 4 | `TYPE vbtyp` CHAR 1 |
| `VBFA-VBTYP_V` | `VBTYPL_V` CHAR 4 | `TYPE vbtyp` CHAR 1 |

Retype the constants and variables. The values (`'C'`, `'M'`, …) exist in both domains (check `DD07L`), so the
selected documents do not change. Prefer retyping over `"#EC CI_USAGE_OK[2198647]`: it repairs instead of
hiding.

## Pricing conditions KONV (note 2220005)

After conversion, `KONV` is empty and the data is in `PRCD_ELEMENTS`. Unmigrated reads return nothing, and an
`INNER JOIN` with `KONV` returns nothing, without any error.

- `PRCD_ELEMENTS-KBETR` is not stored in the classic format: amounts are not shifted by currency decimals and
  percentages are in percent, not per mille. Code that expects classic values must read the compatibility
  view `V_KONV`, which returns them in the classic format.
- Do not `SELECT *` into a table typed `TYPE TABLE OF konv`: names match but some types do not
  (activation fails on `KDATU`). Name the fields the program actually uses, then `MOVE-CORRESPONDING`.
- `V_KONV` requires strict Open SQL syntax (`@` host variables, comma-separated fields, `INTO` at the end).

## BSEG and ACDOCA

`BSEG` is still supported in S/4HANA (note 2431747). Migrating to `ACDOCA` is optional and has a trap:
`ACDOCA` has one line per ledger. A set selection without a ledger filter returns more lines, with no error,
and the program shows inflated figures.

Check how many ledgers each company code uses:

```sql
SELECT rbukrs, rldnr, COUNT(*) FROM acdoca GROUP BY rbukrs, rldnr
```

Decision tree:

1. Does the field exist in `ACDOCA`? Some do not (for example payment term fields `ZFBDT`, `ZBD1T`). Check
   `DD03L` first. If not, there is no migration.
2. `SELECT SINGLE` by full key, ledger-independent field → safe to migrate.
3. `SELECT SINGLE` by key, amount field → add `RLDNR = '0L'` (leading ledger) to be deterministic.
4. `INTO TABLE`, `FOR ALL ENTRIES`, `SUM` → do not migrate, or make the ledger filter mandatory and compare
   amounts before and after.

## Credit management KNKK → FSCM (note 2227014)

`KNKK` may still contain rows after conversion, but it is no longer maintained: new limits go to FSCM. Code that
reads `KNKK` drifts silently from the real limits.

- Read `UKMBP_CMS_SGM`: `KUNNR` → `PARTNER`, `KKBER` → `CREDIT_SGMNT`, `KLIMK` → `CREDIT_LIMIT`.
- Resolve the business partner through the CVI link (`CVI_CUST_LINK` gives the partner GUID, `BUT000` its number), not `partner = kunnr`: the
  numbers often differ.
- Compare before deciding: count limits that differ between `KNKK` and FSCM for the same customer and segment,
  and check the last change dates of both.
- Customer-specific credit fields (appends) must be migrated as data, not only as structure.

## `*_LONG` amounts in BAPIs (note 2628704)

Moving from a classic amount to its `_LONG` counterpart (DEC 23 → DEC 31) can compile and still dump with
`CALL_FUNCTION_CONFLICT_LENG`. This happens when the value reaches a classic-width parameter through untyped
`FORM` parameters (for example a currency conversion BAPI whose amount is `BAPICURR-BAPICURR`, DEC 23).
Neither syntax check nor ATC sees it. Trace the value to its last consumer first: if it ends in a classic-width
typed parameter, the change is a rewrite, not a field swap, and suppression may be the right call.

## Released RFC and BAPI adaptations (note 2438131)

The note title asks for code adaptations, not suppression. Adapt where the long field can be consumed without
rewriting the downstream conversion; suppress only where it cannot, and say why.

## `FIP_*_RANGE` types (note 2368747)

The note concerns Fresh Item Procurement and the types have no planned successor. Equivalent range table types
exist that are identical field by field (`SIGN`, `OPTION`, `LOW`, `HIGH` with the same type and length; verify
in `DD03L`). Replacing them changes no byte in an RFC interface, so no caller notices. In local data,
`TYPE RANGE OF` is enough; in interfaces, use the equivalent DDIC table type.

## Status tables VBUK/VBUP

In S/4HANA the header status fields are in `VBAK` (`GBSTK`, `ABSTK`, `LFSTK`). If `VBUK`/`VBUP` are still
populated in your system, moving reads to `VBAK` is technical debt, not a data emergency: plan it, do not rush it.
