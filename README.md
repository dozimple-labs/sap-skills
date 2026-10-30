# SAP skills by DoZimple

Agent skills for SAP ABAP work, from the projects of [DoZimple](https://dozimple.cl), an SAP consulting
company in Chile.

| Skill | What it does |
|---|---|
| [s4hana-atc-remediation](skills/s4hana-atc-remediation/SKILL.md) | Fix custom ABAP flagged by the ATC S/4HANA readiness checks so the finding is really solved: decision order (SAP Note, precedent, minimal change, suppression), field length extension, VBTYP, KONV, BSEG/ACDOCA, KNKK/FSCM, `_LONG` amounts and definition of done |

These skills are knowledge only: they do not call any SAP system or API.

## Install

With the [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add dozimple-labs/sap-skills
```

Or copy `skills/<skill>/` into your agent's skills folder (for Claude Code, `.claude/skills/`).

## Author and license

Maintained by DoZimple (https://dozimple.cl). Licensed under the [Apache License 2.0](LICENSE).

Not affiliated with or endorsed by SAP SE. SAP, ABAP and S/4HANA are trademarks of SAP SE.
