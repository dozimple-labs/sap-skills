---
name: s4hana-atc-remediation
description: >
  Fix custom ABAP code flagged by the ATC S/4HANA readiness checks (simplification items, field length
  extension, simplified objects) so the finding is really solved, not just hidden. Covers the decision order
  (SAP Note first, then project precedent, minimal change, suppression last), amount field length extension
  (DMBTR/AFLE, TYPE p LENGTH bytes, WRITE TO conflicts), SD document category VBTYP, pricing KONV to
  PRCD_ELEMENTS/V_KONV, BSEG vs ACDOCA ledger filter, KNKK to FSCM credit management, *_LONG amounts in BAPIs,
  FIP_*_RANGE types, pseudo-comments CI_USAGE_OK / CI_FLDEXT_OK and the definition of done.
  WHEN: the user is remediating ATC findings for an ECC to S/4HANA conversion, asks how to close an S/4HANA
  readiness finding, mentions a simplification item note, S4HANA_READINESS check variant, custom code
  migration, field length extension, "move length conflict", "Search for usages of simplified objects",
  or asks whether to suppress or fix an ATC finding.
  DO NOT USE FOR: generic clean-code ATC findings (naming, performance) or for running ATC itself.
---

# S/4HANA ATC remediation

Guidance for turning S/4HANA readiness findings in custom ABAP into real fixes. It is tool-agnostic: run ATC
from ADT in Eclipse or SE80/SCI, read notes in SNOTE or SAP for Me, and edit with your usual editor.

## Decision order

For every finding, decide in this order and write down which step decided it:

1. **The SAP Note.** The ATC finding documentation names the note and the simplification category. The note
   says whether the object disappears, only changes its interface, or has a successor. A fix that contradicts
   the note is wrong even if ATC turns green. The note title often explains more than the ATC message
   (for example, the title of note 2368747 reveals why `FIP_*_RANGE` types have no planned successor).
   - Category **B**: not available, alternative exists → migrate.
   - Category **I**: incompatible change, object still exists → adapt.
   - *No alternative planned* → retired without successor → replace with an equivalent or justify suppression.
2. **Project precedent.** Before deciding, search how the same note was solved in the same system: where-used
   of the simplified object, the sibling object (customer vs vendor variant), `"#EC CI_USAGE_OK[<note>]` in the
   module's packages, and version history. One consistent pattern beats a slightly better new one. If you
   deviate, say why in the ticket.
3. **Minimal change** that removes the cause.
4. **Suppression** with the note's pseudo-comment, only with evidence (measured data, a shared file format,
   an interface you must not change) recorded in the ticket.

## Definition of done

- **0 errors and 0 warnings (priority 1 and 2), not 0 findings.** Priority 3 information findings can stay.
- Measure on the **transport request**, not on a single program: the ATC of a program does not enter the
  function groups it calls, so a program can show 0 while its function module still has priority 1 findings.
- Use the **same check variant** you are reporting against. Readiness and default variants do not contain
  each other; a tool that silently runs the default variant will show different numbers.
- A fix must not create a new finding downstream. Re-run ATC after every change, not only on the edited line.
- Findings of an include are often reported with the **main program name and the include's line number**.
  If the reported line of the main program has nothing related, look in its includes.

## Patterns that close the most common findings

See [references/patterns.md](references/patterns.md) for details and examples. Summary:

| Finding | What really fixes it |
|---|---|
| Amount field length extension (note 2610650) | Widen the narrow variable and every link of its assignment chain. Use an anonymous `TYPE p LENGTH 12 DECIMALS 2`, not `TYPE dmbtr`, to avoid propagating the type to consumers. `LENGTH` counts bytes: digits = 2 × LENGTH − 1 |
| `WRITE TO` type conflict (2610650) | Local text variable → widen to 30. Structure that is a shared file format → suppress with `"#EC CI_FLDEXT_OK[2610650]` |
| `move length conflict` into a Z table field | Measure the real maximum against the field's ceiling before extending: extending forces a table conversion in production |
| Z amount data element | Change its domain to the standard `AFLE13D2O16N_TO_23D2O30N`. Check where-used first: one data element can close many findings at once |
| VBTYP (note 2198647) | Retype constants to `VBTYPL` / `VBTYPL_N` / `VBTYPL_V` (CHAR 4). Values are the same, selection does not change |
| KONV (note 2220005) | Read `V_KONV` (classic formats) or name fields from `PRCD_ELEMENTS`. `KONV` is empty after conversion: unmigrated reads silently return nothing |
| BSEG → ACDOCA | Migrate only when the field exists in ACDOCA, and filter the leading ledger (`RLDNR = '0L'`) for sets and amounts. BSEG stays supported (note 2431747): leaving it is safe, migrating badly is not |
| KNKK → FSCM credit (note 2227014) | Read `UKMBP_CMS_SGM` (`PARTNER`, `CREDIT_SGMNT`, `CREDIT_LIMIT`), resolving customer → business partner through CVI, not `partner = kunnr` |
| `*_LONG` amounts in BAPIs (note 2628704) | Follow the value to its final consumer first: a `_LONG` value passed through untyped parameters into a classic-width parameter dumps at runtime, and neither syntax check nor ATC sees it |
| `FIP_*_RANGE` types | Replace with an equivalent range table type that is identical field by field (check DD03L), or `TYPE RANGE OF` in local data |

## Traps

- **Latent is not active.** Many length findings are conformity issues, not live defects. Measure
  (`SELECT MAX( field )`) and say so in the report; do not dramatize.
- **Several findings on adjacent lines are usually one root cause**: one narrow declaration receiving the
  value in several branches. Fix the declaration, not each line.
- **Widening one text variable moves the truncation** to the next `CONCATENATE` or `MOVE` where ATC no longer
  sees it. Widen the whole chain.
- **Printed reports have a line size.** Widening columns can push a list beyond `LINE-SIZE`. Measure the real
  maximum and decide with the business owner.
- **Views move findings, they do not remove them.** Wrapping a simplified table in a Z view moves the finding
  to the view, where no pseudo-comment can be placed.
- **Do not migrate only to close a conformity finding.** A migration that changes results without an error
  (missing ledger filter, wrong partner mapping, different decimal shift) is worse than the finding.
- **Validate with data, not only with code reading.** For any change of selection or join semantics, compare
  results before and after on real data.

## Suppression, when justified

- Use the pseudo-comment the finding documentation proposes, with the note number:
  `"#EC CI_USAGE_OK[<note>]` for simplified objects, `"#EC CI_FLDEXT_OK[<note>]` for field length extension.
- One exemption per element: in a chained `WRITE` with several amounts, split the statement.
- If the line already ends with a comment, split the statement and put the pseudo-comment on the second line.
- Record the evidence (measurement, interface contract) in the ticket, not in long code comments.

## Reporting a fix

For each object: finding and note, decision step that decided it (note / precedent / minimal change /
suppression), what changed, ATC before and after with the same variant, and any data measurement used.

---
Maintained by [DoZimple](https://dozimple.cl). Apache-2.0.
