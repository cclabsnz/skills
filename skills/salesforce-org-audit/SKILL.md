---
name: salesforce-org-audit
description: Use when assessing the security posture of a Salesforce org you did not build — inheriting an org, pre-acquisition or vendor due diligence, answering a client security questionnaire, hunting guest-user or Experience Cloud exposure, reviewing permissions and sharing before go-live, gating a release pipeline on org risk, or reconstructing who did what from event logs after an incident.
license: Apache-2.0
---

# Salesforce Org Audit

Read-only security assessment of a live Salesforce org via the `sf` CLI. 90 checks
across identity, permissions, sharing, guest access, integrations, Apex, and
Agentforce/GenAI — correlated into attack chains and mapped to OWASP, SOC 2,
ISO 27001, NZISM, HIPAA and GDPR controls.

Everything here is **GET-only**: SOQL, Tooling and REST reads. No DML, no metadata
deploys, nothing written to the target org.

## Setup

```bash
sf plugins install @cclabsnz/sf-audit
sf org login web --alias myOrg      # if not already authenticated
```

The plugin is JIT-installed — the first `sf audit` invocation may prompt for
confirmation. Install explicitly, as above, before running unattended.

## Quick reference

| Goal | Command |
|---|---|
| Full audit, human-readable | `sf audit security --target-org myOrg` |
| Full audit, machine-readable | `sf audit security --target-org myOrg --format json --output ./reports` |
| Client-ready branded report | `sf audit security --target-org myOrg --format executive --prepared-for "Acme Ltd"` |
| Gate a pipeline | `sf audit security --target-org myOrg --fail-on HIGH` |
| Subset of checks | `sf audit security --target-org myOrg --checks guest-user-access,apex-sharing` |
| See available check IDs | `sf audit list` |
| Posture drift over time | `sf audit history --target-org myOrg` |
| Compare two runs | `sf audit diff baseline.json current.json` |
| Over-privileged connected apps | `sf audit apps --target-org myOrg --since 7` |
| Preserve free event logs | `sf audit events pull --target-org myOrg` |
| Reconstruct an actor's activity | `sf audit timeline --window yesterday --seed ip:203.0.113.50` |

Run any command with `--help` for its full flag set.

## Reading the JSON report

Use `--format json` whenever you intend to reason over results. The report lands at
`<output>/sf-audit-<orgId>-<timestamp>.json` — **the timestamp means you cannot
predict the filename.** Glob for the newest file rather than constructing the path:

```bash
sf audit security --target-org myOrg --format json --output ./reports
ls -t ./reports/sf-audit-*.json | head -1
```

Top level: `healthScore` (0–100), `grade` (A–F), `findings[]`, `attackChains[]`,
`orgId`, `isSandbox`. Each finding carries `checkId`, `riskLevel`, `title`,
`detail`, `remediation`, `complianceTags[]`, and two booleans that decide how you
report it: `passed` and `inconclusive`.

## Interpreting the report

Two flags on each finding decide how you report it: `passed` and `inconclusive`.
`inconclusive: true` means the audit could not see the answer — the authenticated
user lacked the permission to gather evidence. It is scored INFO so it does not
distort the health score. The finding text states which permission was missing.

`attackChains[]` carries the analytical weight: a chain is emitted only when every
ingredient is present, and remediating any one step breaks it.

## Stating results honestly

The grade and health score are prioritisation aids. They are not an attestation,
and this tool is not a penetration test, a runtime test, or an audit of managed-package
internals.

- A control showing "no findings detected" means *this audit surfaced nothing mapped
  to it* — not that the org complies with that framework.
- Compliance tags indicate relevance to a control area, nothing more.
- Never write "SOC 2 compliant", "ISO 27001 certified" or equivalent on the strength
  of a report. Write "no findings mapped to CC6.1 in this scan".

## Common mistakes

| Mistake | Do this instead |
|---|---|
| Constructing the report path by hand | Glob `sf-audit-*.json` and take the newest |
| Running against production first | Start on a sandbox to size the output and confirm permissions |
| `--format html` then asking the model to read it | Use `--format json` for reasoning, `html`/`executive` for people |
| Requesting `View All Data` to clear a blind spot | Only one probe needs it; it reports inconclusive by design |
| Treating a sandbox grade as the production grade | Check `isSandbox` before quoting a score |

## Related

- Full check inventory: [`docs/CHECKS.md`](https://github.com/cclabsnz/sf-audit-plugin/blob/main/docs/CHECKS.md)
- Attack chain catalogue: [`docs/ATTACK-CHAINS.md`](https://github.com/cclabsnz/sf-audit-plugin/blob/main/docs/ATTACK-CHAINS.md)
- Minimum permission set: [`PERMISSIONS.md`](https://github.com/cclabsnz/sf-audit-plugin/blob/main/PERMISSIONS.md)
