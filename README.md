# SAP skills by DoZimple

Agent skills for SAP ABAP work, from the projects of [DoZimple](https://dozimple.cl), an SAP consulting
company in Chile.

| Skill | What it does |
|---|---|
| [s4hana-atc-remediation](skills/s4hana-atc-remediation/SKILL.md) | Fix custom ABAP flagged by the ATC S/4HANA readiness checks so the finding is really solved: decision order (SAP Note, precedent, minimal change, suppression), field length extension, VBTYP, KONV, BSEG/ACDOCA, KNKK/FSCM, `_LONG` amounts and definition of done |
| [abap-750-compatible](skills/abap-750-compatible/SKILL.md) | Write ABAP that activates on NetWeaver 7.50 (ECC 6.0 EHP8): what modern syntax exists there, what to write instead (ENUM, CTEs, SELECT FROM @itab, SQL string and date functions, LOOP STEP, utclong), ABAP SQL limits, dictionary objects ADT cannot create on 7.50 and traps found on real systems |

These skills are knowledge only: they do not call any SAP system or API.

## Install

With the [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add dozimple-labs/sap-skills
```

Or copy `skills/<skill>/` into your agent's skills folder (for Claude Code, `.claude/skills/`).

## Need help with this?

DoZimple helps SAP teams with S/4HANA custom code remediation and ABAP development. Our focus is the method:
spec-driven development (SDD), adapted to SAP. Working with your key users, we document your custom code
from its object inventory and real usage, building a knowledge base of your own system.
Contact roviedo@dozimple.cl or asoto@dozimple.cl, or visit [dozimple.cl](https://dozimple.cl).

## Author and license

Maintained by DoZimple (https://dozimple.cl). Licensed under the [Apache License 2.0](LICENSE).

Not affiliated with or endorsed by SAP SE. SAP, ABAP and S/4HANA are trademarks of SAP SE.
