# CloudCounsel Agent Skills

Agent skills for assessing Salesforce orgs you did not build.

Salesforce's own skill libraries are excellent at *building* — Apex, LWC, Flow,
metadata, Agentforce. These skills cover the other half: walking into an existing
org and working out what is actually in it, how exposed it is, and what to fix first.

## Install

```bash
npx skills add cclabsnz/skills
```

Works with Claude Code, Cursor, Codex, OpenCode, and anything else that reads
`SKILL.md`.

## Skills

| Skill | Use when |
|---|---|
| [`salesforce-org-audit`](skills/salesforce-org-audit) | Assessing the security posture of an org you inherited — due diligence, client security questionnaires, guest-user exposure, pre-go-live review, or reconstructing activity after an incident |

## The tools behind them

These skills drive real, published CLIs. They are read-only: SOQL, Tooling and
REST **GET** queries. Nothing is written to the target org.

- [`@cclabsnz/sf-audit`](https://www.npmjs.com/package/@cclabsnz/sf-audit) — 90-check
  security audit with attack-chain correlation and compliance mapping

## Licence

Apache-2.0
