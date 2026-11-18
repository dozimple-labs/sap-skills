---
name: abap-750-compatible
description: >
  Write ABAP that compiles and activates on SAP NetWeaver 7.50 (SAP ECC 6.0 EHP8 and other 7.50 systems):
  which modern syntax exists there and which does not (ENUM, common table expressions, SELECT FROM @itab,
  ABAP SQL string and date functions, LOOP STEP, PCRE, utclong), what to write instead, ABAP SQL limits,
  dictionary objects ADT cannot create on 7.50, and traps found on real 7.50 systems.
  WHEN: the user writes, reviews or fixes ABAP for an ECC 6.0 EHP8 / NW 7.50 system, mentions SAP_BASIS 750,
  a syntax error such as "unknown", "not supported" or "not allowed here" after pasting code written for
  S/4HANA or the cloud, or asks whether a statement, function or class exists on 7.50.
  DO NOT USE FOR: ABAP Cloud / RAP / BTP development, or S/4HANA readiness remediation.
---

# ABAP compatible with NetWeaver 7.50

Most ABAP examples on the web, and most code an assistant writes by default, target recent releases.
On a 7.50 system part of that syntax does not exist, and some of it fails in ways that are hard to see.
This skill says what to use, what to avoid and what to write instead.

## First, confirm the release

- **System › Status › Component information**: the `SAP_BASIS` row gives release and support package.
  ECC 6.0 EHP8 runs on 7.50. EHP7 runs on 7.40, where availability also depends on the SP (SP02, SP05, SP08).
- **Database**: AMDP and CDS table functions need SAP HANA as the database. Many ECC systems still run on
  another database; there, CDS views work but AMDP does not.
- Never trust a linter alone. The real check is the **syntax check and activation in the target system**.

## Safe to use on 7.50

| Area | Available |
|---|---|
| Declarations | Inline `DATA( )` and `FIELD-SYMBOL( )`, also in `SELECT ... INTO @DATA( )` |
| Constructor expressions | `VALUE` (with `BASE`, `DEFAULT`/`OPTIONAL` on table expressions), `NEW`, `CONV`, `EXACT`, `REF`, `CAST`, `COND`, `SWITCH`, `CORRESPONDING`, `REDUCE`, `FILTER`, `LET`, `FOR` (also `FOR GROUPS OF`) |
| Internal tables | Table expressions `itab[ ... ]`, `line_exists( )`, `line_index( )`, `LOOP AT ... GROUP BY`, meshes |
| Logic | `xsdbool( )`, predicative method calls, `IS INSTANCE OF`, `CASE TYPE OF` (both new in 7.50) |
| Exceptions | `RAISE EXCEPTION ... MESSAGE` (new in 7.50), `COND ... THROW` |
| Types | `int8` (new in 7.50), `decfloat16/34`, string templates `|...|` |
| ABAP SQL | Comma-separated lists, host variables with `@`, host expressions `@( )` (new in 7.50), `CASE`, `COALESCE`, arithmetic expressions, `UNION` (new in 7.50), CDS views with parameters and path expressions |
| CDS | Views, parameters, associations, `CONCAT`, `REPLACE`, `ROUND`, `LENGTH`, `COALESCE`, `CONCAT_WITH_SPACE`, table functions (HANA only) |

## Not on 7.50: what to write instead

See [references/replacements.md](references/replacements.md) for code examples.

| Not available on 7.50 | From | Write instead |
|---|---|---|
| `TYPES ... ENUM` | 7.51 | Constants in an interface, or a domain with fixed values |
| `WITH` (common table expressions) | 7.51 | Two `SELECT`s, a subquery in `WHERE`, or a CDS view |
| `SELECT ... FROM @itab` | 7.52 | `FOR ALL ENTRIES IN` (check the table is not empty first), or `READ TABLE` / `FILTER` on the internal table |
| ABAP SQL `UPPER`, `LOWER`, `LEFT`, `RPAD`, `INSTR`, `CONCAT_WITH_SPACE` | 7.51 | Do it in ABAP after the `SELECT`, or in a CDS view (CDS on 7.50 has more string functions than ABAP SQL) |
| ABAP SQL `DATS_*` functions, `DIVISION` | 7.51 | Calculate in ABAP; date arithmetic on `d` fields works directly |
| ABAP SQL `TSTMP_*`, `IS INITIAL`, `UUID`, window functions (`LEAD`, `LAG`, `STRING_AGG`) | 7.53–7.55 | Read the rows and calculate in ABAP (`cl_abap_tstmp`, `CONVERT TIME STAMP`, `LOOP ... GROUP BY`) |
| `utclong`, `UTCL_*` | 7.54–7.55 | `timestampl` and `GET TIME STAMP` |
| `LOOP AT ... STEP` | 7.57 | `LOOP AT ... FROM ... TO` and skip by index, or `DO` with `READ TABLE INDEX` |
| PCRE regular expressions (`PCRE` addition) | 7.55 | POSIX regex (`FIND REGEX`, `cl_abap_regex` without PCRE) |
| `CORRESPONDING ... DEFAULT` | 7.58 | `MAPPING`, then fill defaults explicitly |
| Dynamic component `ref->(name)` | 7.56 | `ASSIGN ref->(name) TO <fs>` |
| `RAISE SHORTDUMP` | 7.53 | `MESSAGE ... TYPE 'X'` with a clear text |
| SQL test doubles (`cl_osql_test_environment`) | 7.52 | Wrap the `SELECT` in a method you can replace in the test |
| RAP, EML, behavior pools, `ABAP for Cloud Development` | 7.53+ | Classic: classes, function modules, BAPIs, Gateway (SEGW) |

## Traps found on real 7.50 systems

Found on real 7.50 systems. The syntax ones were reproduced with the syntax check of a 7.50 system.

- **`boolc( )` returns a string**, and the syntax check warns about it. For `abap_bool` use `xsdbool( )`.
- **Inline `@DATA( )` does not work with a dynamic `FROM (lv_table)`.** Declare the target first.
- **`GROUP BY` together with `FOR ALL ENTRIES` is rejected.** Group in ABAP afterwards.
- **`AVG( field AS type )` does not compile.** Use `SUM` and `COUNT` and divide in ABAP.
- **`cl_abap_tstmp=>subtract` works in whole seconds.** For milliseconds, subtract the integer and fraction
  parts separately.
- **`RAISING cx_root` in a method signature is rejected.** Declare the concrete exception classes.
- **An ABAP Doc comment `"!` before a chained statement (`TYPES:`) raises a warning.** Use separate
  statements.
- **Generic data elements: not every length exists.** `CHAR30`, `CHAR61`, `CHAR128` exist; `CHAR60` does not.
  Check DD04L before using one, or declare `TYPE c LENGTH n`.
- **Unit tests run against the active version.** A test run can stay green while the inactive version does
  not activate. Activate first, then run the tests.

## Dictionary and tooling on 7.50

ADT on 7.50 covers less than on recent releases. Plan these steps in the classic transactions:

| Object | On 7.50 |
|---|---|
| Package | Create in **SE21** (no package editor in ADT before 7.52) |
| Database table, domain | **SE11** (ADT table editor from 7.52, domain editor from 7.51) |
| Data element, structure | ADT works; SE11 too |
| Remote-enabled flag of a function module | **SE37**, attributes tab. It is not part of the source code |
| OData service | **SEGW** project; then edit the generated `_DPC_EXT` / `_MPC_EXT` classes as normal classes |
| Text elements | Supported in ADT; SE38 › Goto › Text elements also works |

When a class is edited as a single source in ADT, save the definition and the implementation together.
A class saved in pieces can fail activation with "implementation missing for method".

## Definition of done

- Syntax check and activation succeed **in the 7.50 system**, not only in a local linter.
- No statement from the "not on 7.50" table remains, including in test classes and local classes.
- ABAP SQL that moved logic to ABAP gives the same result as before. Compare on real data when the
  change touches selection, grouping or joins.
- If something only works from a later release, say so in the ticket and name the 7.50 alternative used.

## Sources

Release availability cross-checked against the
[ABAP Keyword Documentation](https://help.sap.com/doc/abapdocu_latest_index_htm/latest/en-US/index.htm)
(release news) and the [Software-Heroes ABAP Feature Matrix](https://software-heroes.com/en/abap-feature-matrix).
The traps come from DoZimple's own work on 7.50 systems.

---
Maintained by [DoZimple](https://dozimple.cl), SAP consulting. Questions or project support:
roviedo@dozimple.cl or asoto@dozimple.cl. Apache-2.0.
