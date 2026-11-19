---
name: sap-transport-risk-checklist
description: >
  Decide whether an ABAP transport request (or a set of requests released together) is safe to import into
  quality or production, and write the verdict so project managers, consultants and Basis each know what to do.
  Covers unreleased tasks, objects that do not travel (local packages), dependencies missing in the target,
  version drift between development and target, positional access to structures (offset/length) broken by
  a table change, dictionary activation failures (return code 8), CTS locks held by another request and
  transports of copies, collisions between requests of the same release, import sequence and mutual
  dependencies, tables replicated with SLT triggers, and import return codes 0/4/8/12.
  WHEN: the user asks "can I move this to QAS / PRD?", "is it safe to release?", "what risk does this transport
  have?", "why did the import fail with RC 8?", gives a request number like SIDK900123, plans a go-live with
  several requests, or reviews a transport before production.
  DO NOT USE FOR: running the import itself (that is Basis in STMS), or code review of the ABAP logic.
---

# SAP transport risk checklist

Most failed imports and post go-live dumps are not bugs in the new code. They come from what the request
does **not** carry: a data element that stayed in development, a program that reads a changed table by
byte position, another consultant's half-finished change pulled in by a transport of copies.
This checklist looks for those gaps before the import, and says how to report them.

## First, fix the question

- **Against which system?** The answer changes completely. Checked against development, it only says the
  request is complete. Checked against the target (QAS or PRD), it says whether it will import cleanly there.
  If the user does not say, ask.
- **One request or a release?** When several requests go together ("Friday's release", "these four orders"),
  check them **as a set**. What one request carries covers the others; checked one by one they report
  dependencies as missing that will in fact arrive.
- **Could you read the target?** If the target could not be queried (missing RFC authorization, no
  connection), say "could not be checked against PRD". A check that did not run is never "no risk".

## The checklist

Rate each finding HIGH or MEDIUM. Any HIGH means **not ready to import**.

| # | Check | Where to look | Rating |
|---|---|---|---|
| 1 | All tasks released, and whose they are | SE10 / SE09 | HIGH: the request cannot be released |
| 2 | Every object belongs to a package with a transport layer | SE80 / package attributes; `$TMP` and local packages never travel | HIGH |
| 3 | Every dependency travels in the release **or** already exists active in the target | Where-used and the object's own uses (tables, function modules, classes, data elements, domains) compared with the target | HIGH |
| 4 | Objects of the request exist in the target with an active version, or travel complete | Version management in the target | HIGH if missing and not carried |
| 5 | No version drift: the target has the same version the change was built on | Version management (SE38/SE24 › Utilities › Versions), remote comparison | HIGH: a parameter or field differs and that difference does not travel |
| 6 | Positional access (`struc+off(len)`) to a table or structure the request changes | Source search for the structure name followed by `+` | HIGH when the reading code does **not** travel |
| 7 | Dictionary: new fields typed with data elements or domains that travel or exist in the target | SE11 of each new field | HIGH: activation fails on import, return code 8 |
| 8 | The CTS lock on each object belongs to this request | Transport information of the object (SE80), or table TLOCK | HIGH for a transport of copies (see below) |
| 9 | No other **open** request contains the same objects | SE03 › Search for objects in requests | MEDIUM: coordinate who goes first |
| 10 | Two requests of the same release do not change the same object | Object lists of the release | MEDIUM: the last one imported wins |
| 11 | Import order respects dependencies; mutual dependencies go in the same window | Import queue in STMS | HIGH if the queue order breaks a dependency |
| 12 | Tables the request changes are not replicated with SLT triggers in the target | Ask Basis; SLT registers replicated tables | HIGH: pause replication or drop the trigger before import |
| 13 | Customizing keys the request carries are the intended ones | Object list, table keys | MEDIUM |
| 14 | Previous imports of the same objects into the target ended without errors | Import history (STMS › Import history, return codes) | HIGH for RC ≥ 8 still open |

Return codes: `0` clean, `4` warnings (MEDIUM, read the log), `8` errors (HIGH, something did not activate),
`12` or higher aborted (HIGH, Basis).

A request that is **not yet imported** is not a risk: it is the one about to be imported. Only a **failed**
previous import blocks.

## The three gaps that cause the worst incidents

**Positional access outside the request.** Code like `ls_line+57(10)` or `CONCATENATE ... ls_rec(120)` reads a
structure by byte position. Change the structure (a field added in the middle, a longer field) and that code
keeps running **without any error**, reading the wrong bytes. Classify by scope:
- the reading code travels in the same request: contained, but check the offsets still match;
- the table changes and the reading code stays outside: **the most serious case**. It reaches production
  untested. Either add it to the release or prove the offsets are unaffected.

**Dictionary dependency left in development.** The request adds a field to a table and types it with a custom
data element or domain that is not in the release and not in the target. The table does not activate on
import: return code 8, and everything that uses the table is affected.

**Transport of copies and somebody else's lock.** A transport of copies does not lock objects. It can contain
an object whose CTS lock belongs to another open request. When it is released it takes **whatever is in
development at that moment**, including the other request's unreleased changes. Say it plainly: "this request
carries the form, but it is locked in request X by another consultant; releasing now carries their changes too."

## Missing dependency: two different actions

When a dependency travels in another request, check the target's import queue:
- **already in the queue** at position N: it will arrive. Make sure it is imported before the request that
  needs it;
- **not in the queue**: it will not arrive by itself. Someone has to release it and add it to the queue.

These need different people to act, so keep them apart in the report.

## How to write the verdict

Three layers, in this order. Leave out a layer that is empty; never fill it with filler.

1. **Verdict, for project managers.** Start with the state and the system: "Ready to import into PRD" or
   "Not ready to import into PRD". One or two sentences, no ABAP terms. Say the business effect: "the
   invoice print will fail", "the search help will be empty", not "positional access at offset 57".
   If there are non-blocking warnings, say it is ready and list them.
2. **What to review, for the consultant.** Short list: object, check number, what to look at. HIGH before
   MEDIUM; within positional access, outside-the-request cases first.
3. **Release actions, for Basis.** Which request goes first, which tasks are still open and whose, the
   import order and the current import status.

Never claim an object is in QAS or PRD without having seen it there. Never invent a verdict when a check
could not run; repeat what could not be checked.

## Definition of done

- Every row of the checklist is either checked against the **target**, or explicitly reported as not checked.
- Releases of several requests were checked as a set, with import order and collisions stated.
- The verdict starts with "Ready" or "Not ready", names the target system and states the business effect.
- Releasing and importing stay with whoever owns them (SE10, STMS); this checklist only advises.

---
Maintained by [DoZimple](https://dozimple.cl), SAP consulting. Questions or project support:
roviedo@dozimple.cl or asoto@dozimple.cl. Apache-2.0.
