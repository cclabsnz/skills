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

## If you are an agent, run it like this

Always pass the flags below. The defaults are tuned for a person reading a report in a
terminal; without these you get 90 lines of progress logging, a timestamped file to hunt
for, and a payload where most of the tokens are passing checks and prose you will not use.

```bash
# 1. Check what this audit user can actually establish — before spending a run
sf audit preflight --target-org myOrg --json

# 2. Audit, compact and machine-readable
sf audit security --target-org myOrg --json --digest
```

| Flag | Why an agent wants it |
|---|---|
| `--json` | Result on stdout in a `{status, result, warnings}` envelope; progress logging suppressed. Errors arrive in the same shape, so you parse rather than scrape |
| `--digest` | Drops passing checks and `detail` prose, caps affected-item lists. Roughly **15x smaller** — 617 KB to 41 KB on a 405-finding report |
| `--fail-on-inconclusive` | Add when gating CI, so a blind audit fails loudly instead of reading as clean |

Run `sf audit preflight` **first**, every time. It costs one query and tells you which
checks will come back inconclusive. The audit itself can only report a permission gap
after the run is already spent.

Requires `@cclabsnz/sf-audit` 1.10.0 or later.

## Quick reference

Every row an agent would run carries `--json`. The two marked *for people* are the
exceptions — they exist to be read by a human, not parsed.

| Goal | Command |
|---|---|
| Check coverage before auditing | `sf audit preflight --target-org myOrg --json` |
| Full audit | `sf audit security --target-org myOrg --json --digest` |
| Full audit, every detail | `sf audit security --target-org myOrg --json` |
| Gate a pipeline | `sf audit security --target-org myOrg --json --fail-on HIGH --fail-on-inconclusive` |
| Subset of checks | `sf audit security --target-org myOrg --json --checks guest-user-access,apex-sharing` |
| See available check IDs | `sf audit list --json` |
| Posture drift over time | `sf audit history --target-org myOrg --json` |
| Compare two runs | `sf audit diff baseline.json current.json --json` |
| Over-privileged connected apps | `sf audit apps --target-org myOrg --since 7 --json` |
| Preserve free event logs | `sf audit events pull --target-org myOrg --json` |
| Reconstruct an actor's activity | `sf audit timeline --window yesterday --seed ip:203.0.113.50 --json` |
| Persist a report for later diffing | `sf audit security --target-org myOrg --format json --output ./reports` |
| Terminal-readable audit *(for people)* | `sf audit security --target-org myOrg` |
| Client-ready branded report *(for people)* | `sf audit security --target-org myOrg --format executive --prepared-for "Acme Ltd"` |

Run any command with `--help` for its full flag set.

## Machine-readable output

Pass `--json` to any command. Human progress logging is suppressed and the result
is emitted on stdout in the standard envelope:

```bash
sf audit security --target-org myOrg --json
```

```json
{ "status": 0, "result": { "healthScore": 61, "grade": "D",
  "findings": [...], "attackChains": [...] }, "warnings": [] }
```

Errors come back the same way — `{"status": 1, "name": ..., "message": ...}` —
so parse the envelope rather than scraping stderr. Setting `SF_CONTENT_TYPE=JSON`
in the environment has the same effect when you cannot control the argv.

A non-zero exit does not suppress the report: `--json --fail-on HIGH` returns the
full result *and* exits 1, so read the envelope rather than inferring from the code.

Add `--digest` for the compact shape: `counts` for every severity plus passed and
inconclusive, active findings with `remediation` and `affectedCount`, inconclusive
reduced to `checkId` + `title`, and `attackChains` intact. The exit code is still
computed from the full result, so gating is unaffected by digesting.

### Exit codes

| Code | Meaning |
|-----:|---------|
| `0` | Audit completed; nothing the caller asked to fail on |
| `1` | Findings at or above `--fail-on` |
| `2` | The audit could not run — auth, connection, or bad flags |
| `3` | Ran, but checks could not gather evidence (`--fail-on-inconclusive` only) |

`3` is how you tell "this org is fine" apart from "the audit user could not see
enough to judge". Requires `@cclabsnz/sf-audit` 1.9.0 or later.

**Prefer `--json` over `--format json`.** The latter writes a *file* named
`sf-audit-<orgId>-<timestamp>.json`; the timestamp means you cannot predict the
path. Use it only when you want the report persisted for a later `sf audit diff`,
and then glob for it rather than constructing the name:

```bash
ls -t ./reports/sf-audit-*.json | head -1
```

Report top level: `healthScore` (0–100), `grade` (A–F), `findings[]`,
`attackChains[]`, `orgId`, `isSandbox`. Each finding carries `checkId`,
`riskLevel`, `title`, `detail`, `remediation`, `complianceTags[]`, `passed`
and `inconclusive`.

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
| Scraping progress logs off stdout | Pass `--json` — logging is suppressed and the result is the only output |
| Using `--format json` to read results | That writes a timestamped file; `--json` puts it on stdout |
| Auditing before checking coverage | Run `sf audit preflight` first — one query, and it names what will come back blind |
| Reading the full result when you only need to act | Add `--digest`; the prose and affected lists are ~93% of the payload |
| Running against production first | Start on a sandbox to size the output and confirm permissions |
| `--format html` then asking the model to read it | Use `--json` for reasoning, `html`/`executive` for people |
| Requesting `View All Data` to clear a blind spot | Only one probe needs it; it reports inconclusive by design |
| Treating a sandbox grade as the production grade | Check `isSandbox` before quoting a score |

## Related

- Full check inventory: [`docs/CHECKS.md`](https://github.com/cclabsnz/sf-audit-plugin/blob/main/docs/CHECKS.md)
- Attack chain catalogue: [`docs/ATTACK-CHAINS.md`](https://github.com/cclabsnz/sf-audit-plugin/blob/main/docs/ATTACK-CHAINS.md)
- Minimum permission set: [`PERMISSIONS.md`](https://github.com/cclabsnz/sf-audit-plugin/blob/main/PERMISSIONS.md)
