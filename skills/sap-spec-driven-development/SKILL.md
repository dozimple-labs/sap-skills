---
name: sap-spec-driven-development
description: >
  Apply spec-driven development (SDD) to custom SAP ABAP development: write requirements, design and tasks
  before code, adapted to how SAP projects really work. Covers starting from an inventory of the existing
  custom objects and their real usage, validating the specification with key users, standard-first decisions
  (customizing, BAPIs, enhancement spots before modifications), release and clean core constraints (ECC 7.50
  vs S/4HANA), dictionary, authorizations and transport plan in the design, small tasks that each end with
  syntax check and activation in the system, traceability from requirement to object and transport, and the
  closing documentation.
  WHEN: the user starts a new custom development, enhancement or interface in SAP, asks to write a functional
  or technical specification, wants an AI assistant to build ABAP from a spec, reviews an estimate, or wants
  to document existing custom code with the business.
  DO NOT USE FOR: small bug fixes with a known cause, or S/4HANA readiness remediation of existing code.
---

# Spec-driven development for SAP

Spec-driven development (SDD) is a method widely used with AI coding assistants: the specification is written
and agreed first, and the code is produced from it, task by task. DoZimple uses it and adapts it to SAP,
where most of the risk is not in writing new code but in what already exists around it: custom objects
nobody remembers, customizing, other projects changing the same objects, and the transport path to production.

The method has three documents, in this order. None is skipped; each one is short.

| Document | Answers | Agreed with |
|---|---|---|
| `requirements.md` | What the business needs, and how we will know it works | Key users and the process owner |
| `design.md` | How it will be built in this SAP system | Technical lead; architect or Basis when needed |
| `tasks.md` | In which small steps, each one verifiable | The developer (or the assistant) who builds it |

## 0. Start from the inventory, not from a blank page

Before writing the requirements, find out what is already there:

- **Existing custom objects** in the area (programs, classes, enhancements, tables, interfaces), their
  packages and the transport requests that changed them recently.
- **Real usage in production**: which of those objects run, how often and by whom (workload statistics,
  usage logging such as ST03N or SCMON). Unused code is a candidate to retire, not to extend.
- **Standard first**: is there customizing, a BAPI, a released API or an enhancement spot that already does
  it? A modification of SAP code is the last option and needs its own justification.
- **Release limits**: ECC on NetWeaver 7.50 and S/4HANA allow different syntax, objects and APIs. In
  S/4HANA, check whether the objects you plan to use are released (clean core).

Documenting existing custom code is done **with the key users**, starting from this inventory: they explain
what each object is for; the code shows how. Neither source alone is enough.

**Never carry code from one client to another.** Reuse the method, the patterns and the lessons learned;
each client's code and data stay in that client's systems.

## 1. Requirements

- Write each requirement as a user story with **acceptance criteria** that a key user can check.
  A structured form helps: "WHEN <event> THE SYSTEM SHALL <response>", "IF <condition> THEN ...".
- Use the business terms of the process (sales order, goods receipt, invoice, warranty claim), with the SAP
  document or transaction only as a reference.
- State what is **out of scope** explicitly.
- Include the non-functional needs SAP projects usually forget: volumes, run times and batch windows,
  authorizations (who may do it), countries or company codes, languages, and the systems it must reach.
- Each requirement has a number. The design and the tasks refer to it.

Review it with the key users before going on. A requirement they cannot check is not finished.

## 2. Design

Write only what this system needs; reference the inventory instead of repeating it.

- **Objects**: new and changed, with type, package and why each is needed. Mark the ones another open
  request is already changing.
- **Data model**: tables, structures, data elements and domains. On 7.50, plan what ADT cannot create
  (packages, tables and domains in SE11/SE21, the remote-enabled flag in SE37).
- **Process flow**: entry points (transaction, job, RFC, OData, enhancement), what reads and writes what.
  A diagram is worth it when there are more than three steps.
- **Interfaces**: direction, format, error handling and restart.
- **Authorizations**: authority checks in the code and the roles the client must assign.
- **Transport plan**: which requests, in which order, what travels with what; customizing apart from code.
- **Tests**: test data, ABAP Unit where the logic allows it, and the key-user tests that close each
  requirement.
- **Risks and decisions**: what was chosen and why, with the rejected options in one line each.

## 3. Tasks

- Small tasks, each one mapped to requirement numbers, in an order that can be built and checked step by step.
- Each task ends with **syntax check and activation in the SAP system**, not only a local linter, and with
  ATC on the changed objects when the project uses it.
- One task, one purpose. A task that touches many objects for different reasons should be split.
- Mark tasks that need a manual step in SAP (SE11, SE21, SE37, SEGW, customizing) and who does it.

When an AI assistant builds from the tasks, give it the three documents and one task at a time. Review each
result against the task's acceptance criteria before the next one.

## Traceability

Keep one line per requirement through the whole project:

| Requirement | Design section | Tasks | Objects | Transport request | Tested by |
|---|---|---|---|---|---|

It answers the questions that always come late: why does this object exist, what breaks if we change it,
and which request has to go to production for requirement 4 to work.

## Closing documentation

At the end, the three documents become the technical specification of what was built. Update them to what
was really built, not what was planned, and add the production transport requests and the key-user
acceptance. Write a summary, not a log of every test run.

## Definition of done

- Every requirement has acceptance criteria and was accepted by a key user.
- Every object in the transport requests appears in the design, and every design object traces to a requirement.
- Every task passed syntax check and activation in the target release; ATC is clean or each exception is justified.
- The transport plan was checked before release (dependencies, order, objects that do not travel).
- The documents describe what was built and are stored with the project, without data from other clients.

---
Maintained by [DoZimple](https://dozimple.cl), SAP consulting. Questions or project support:
roviedo@dozimple.cl or asoto@dozimple.cl. Apache-2.0.
