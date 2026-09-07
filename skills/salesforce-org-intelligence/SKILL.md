---
name: salesforce-org-intelligence
description: Use when working out how an unfamiliar Salesforce org actually works — inheriting an org or handover, scoping a migration or refactor, finding where the real business processes live, tracing which objects are coupled by flows and Apex before changing one, identifying what an org integrates with and who uses it, or sizing technical debt in an org nobody can explain.
license: Apache-2.0
---

# Salesforce Org Intelligence

Read-only analysis of how a Salesforce org is actually used, via the `sf` CLI. Ranks
where business processes live, maps which objects are coupled by which automation, and
inventories products, personas, channels and integrations.

Where a security audit answers *"how secure is this org"*, this answers *"how does this
org work"*. For the security question use the `salesforce-org-audit` skill instead.

**Local-first and deterministic.** No metadata leaves the machine — the only network
calls are to the authenticated org's APIs. No LLM calls, no telemetry. Same org in, same
findings out, which is what makes the output quotable in a handover document.

## Setup

```bash
sf plugins install @cclabsnz/sf-orgintel
sf org login web --alias myOrg      # if not already authenticated
```

## If you are an agent, run it like this

Pass `--json` to every command. The default output is formatted for a person reading a
terminal; `--json` gives you the result on stdout in a `{status, result, warnings}`
envelope with the progress logging suppressed.

```bash
# 1. Can this org even answer the question? Start here, always.
sf intel probe --target-org myOrg --json

# 2. Where do the business processes live?
sf intel discover --target-org myOrg --json

# 3. What is coupled to what, and by which automation?
sf intel map --target-org myOrg --json
```

**Run `probe` first.** It returns an evidence tier of `A`–`D` describing how much the org
can tell you about itself — Event Monitoring level, field-history coverage, and which
behavioural tables carry twelve months of rows. A `D` org will not support the same
conclusions as an `A` org, and knowing that before you run `discover` or `map` stops you
over-reading a thin result.

**Repeat runs are cheap.** Analysis is memoised under
`~/.orgintel/cache/<orgId>/v<toolVersion>/`, keyed by a hash of the content analysed.
Cold, warm and `--refresh` runs produce identical output, so re-running to read a
different part of the result costs nothing. Pass `--refresh` only when the org itself has
changed — it exists on `probe`, `discover` and `map`. The cache is namespaced by tool
version, so an upgrade never serves a stale result.

## Quick reference

Every row an agent would run carries `--json`. The `--html` variants are for people.

| Goal | Command |
|---|---|
| What can this org tell us about itself | `sf intel probe --target-org myOrg --json` |
| Where the business processes live | `sf intel discover --target-org myOrg --json` |
| Top N candidate anchor objects | `sf intel discover --target-org myOrg --json --top 15` |
| Object couplings and the automation behind them | `sf intel map --target-org myOrg --json` |
| Include inactive flows in the coupling graph | `sf intel map --target-org myOrg --json --include-inactive` |
| Products, personas, channels, integrations | `sf intel anatomy --target-org myOrg --json` |
| Recompute after the org changed | *(add)* `--refresh` — `probe`, `discover`, `map` only |
| Branded report *(for people)* | `sf intel probe --target-org myOrg --html --output ./reports` — `probe`, `map`, `anatomy` only |

Run any command with `--help` for its full flag set.

## What each command returns

| Command | Result |
|---|---|
| `probe` | `org`, `eventMonitoring`, `fieldHistory`, `behavioralTables`, `evidenceTier` (`A`–`D`), `coverage[]`, `recommendations[]` |
| `discover` | `anchors[]` ranked by six evidence-backed signals, `fingerprint`, `weights`, `totalObjectsAnalyzed`, `droppedObjects`, `notes[]` |
| `map` | `couplingGraph`, `manifest`, plus counts of flows, Apex classes and triggers analysed |
| `anatomy` | `products[]`, `personas[]`, and integration edges, each recording how it was detected and, separately, how it was attributed |

With `--output`, `map` also writes `coupling-graph.json` and `landscape-manifest.json`,
and `discover` writes the domain fingerprint to a file unless you pass
`--no-fingerprint-file`.

## Reading the result

**`notes[]` is where the gaps are.** Anything the authenticated user cannot read is
reported as a note rather than failing the run, so a low-privilege user still gets a
partial, clearly-labelled result. Read `notes[]` before treating a result as complete —
an empty `anchors[]` may mean *nothing qualified* or *nothing was readable*, and the
notes are what separate the two.

**Ranking is evidence, not truth.** `discover` ranks anchor candidates with six
deterministic signals — automation density, a status/lifecycle-shaped field, record
volume and velocity, relationship centrality, activity attach rate, and existing history
tracking. The weights are configurable and reported in `weights`. Treat the ranking as a
place to look first, not a conclusion about what the business does.

**Attribution is recorded separately from detection** in `anatomy`. A confirmed
integration call with an unknown owner is reported as exactly that, rather than being
guessed into a product. Do not collapse the two fields when summarising.

## Related

- Security posture of the same org: the `salesforce-org-audit` skill
- Architecture: [`docs/ARCHITECTURE.md`](https://github.com/cclabsnz/sf-orgintel/blob/main/docs/ARCHITECTURE.md)
- Minimum permissions per command: [`PERMISSIONS.md`](https://github.com/cclabsnz/sf-orgintel/blob/main/PERMISSIONS.md)
