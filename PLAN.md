# Build Plan — `blueteamthreathunting` v0.1.0

This file drives the **automated build**. It is self-contained: a fresh Claude Code
session can read this file and execute the build end-to-end.

## How to execute (read this first)

1. Refresh PATH so `git` resolves:
   `$env:Path = [Environment]::GetEnvironmentVariable('Path','Machine')+';'+[Environment]::GetEnvironmentVariable('Path','User')`
2. Work in `C:\Users\<username>\Projects\blueteamthreathunting` (git already initialized, `origin` set).
3. Dispatch the **local user-scope subagents** named in the delegation table below, in order,
   using the grammar `Use the <agent> subagent to <task>`. (Copies of these agents also live in
   `.claude/agents/` for portability.)
4. Write artifacts into the repo folders. **Mirror the conventions** of the sibling repo
   `C:\Users\<username>\Projects\Sentinelmonitoring` (see Reference files).
5. **Commit at milestones** (after scaffold review, after content, after docs, final) as
   `Philliamviber <Philliamviber@users.noreply.github.com>` and **push to GitHub via Git
   Credential Manager — do NOT use the Copilot-hosted GitHub MCP** (its credits are blocked).
6. These are **starting templates** — document every threshold as tunable.

## Goal & definition of done

A Microsoft Sentinel blue-team **identity** threat-hunting tracker that uses **Zero Trust**
to correlate anomalous behavior of **privileged** and **VIP-flagged** accounts. Done when
every folder below is populated with valid, deployable templates that mirror Sentinelmonitoring
conventions, the CI workflow passes (JSON parses + hunting-query YAML schema), and the repo is
committed and pushed. No live Azure tenant — everything is templates + docs.

Target artifacts:
- `watchlists/` — VIP + privileged-account watchlists (CSV + companion `.md` schema), `_GetWatchlist()` ready.
- `hunting-queries/` — KQL-in-YAML set: impossible travel, MFA fatigue/repeated denials, dormant-admin
  reactivation, new/elevated admin-role assignment, off-hours privileged role activation, VIP risky
  sign-ins, auth-method/MFA tampering. Privilege/VIP **boost** via watchlist joins.
- `zero-trust/` — correlation queries grouped by **Verify Explicitly / Use Least Privilege / Assume Breach**, each mapped to tables.
- `analytics-rules/` — highest-fidelity detections promoted to deployable ARM/JSON scheduled rules.
- `workbooks/` — a Zero-Trust identity correlation workbook (JSON).
- `docs/` — Zero-Trust model doc, deployment guide (`az deployment` examples), MITRE ATT&CK coverage map, tuning notes.
- Root README.md (polished) + CHANGELOG.md updated to v0.1.0.

## Resolved defaults (tech-lead open questions — use these, document as tunable)

- **Watchlist keys:** two watchlists, alias `VIPUsers` and `PrivilegedAccounts`, primary key
  `UserPrincipalName`, secondary `ObjectId`.
- **Privileged source of truth:** static CSV template for v0.1.0; note dynamic hydration from
  `IdentityInfo` / PIM-eligible roles as a future enhancement.
- **Thresholds (starting points):** impossible travel ~800 km/h; MFA fatigue 5 denials / 10 min;
  dormant-admin 30 days inactivity; off-hours = outside 07:00–19:00 local + weekends.
- **Scoring:** additive tag-based boost; an account that is both VIP **and** privileged scores highest.
- **Repo:** name `blueteamthreathunting`, owner `Philliamviber`, **private** (flip to public later if desired).
- **UEBA:** mark queries that need `BehaviorAnalytics`/`IdentityInfo` as "requires UEBA" so they degrade gracefully.

## Delegation table (from tech-lead)

| Step | Specialist | Task | Depends on | Parallel? |
|------|-----------|------|-----------|-----------|
| 0 | `requirements-analyst` | Lock v0.1.0 scope: confirm watchlist schema/keys, threshold defaults, VIP/privileged scoring, detection list; output an acceptance-criteria checklist. | — | No (gate) |
| 1 | `iam-engineer` | Zero Trust identity model + watchlist data design (CSV schemas, join keys, boost logic, pillar→table mapping). | 0 | Yes (with 2) |
| 2 | `threat-modeler` | Map each detection to MITRE ATT&CK (T1078, T1098, T1556, T1110, T1550, T1021) and to ZT pillars; drives the coverage map. | 0 | Yes (with 1) |
| 3 | `detection-engineer` | Author `watchlists/` (VIP + privileged CSV + schema docs) and full `hunting-queries/` KQL-in-YAML with `_GetWatchlist()` boosts. | 1, 2 | No |
| 4 | `detection-engineer` | Author `zero-trust/` correlation queries by pillar; promote top detections to deployable `analytics-rules/` ARM JSON (GUID ids, tactics/techniques, entity mappings). | 3 | After 3 |
| 5 | `detection-engineer` | Author `workbooks/` Zero-Trust identity correlation workbook (JSON). | 3 | After 4 |
| 6 | `devops-engineer` | Repo scaffold + CI (already in place: `.github/workflows/validate.yml`, `.gitignore`, `.claude/agents/`); verify and extend if needed. | 0 | Done (verify) |
| 7 | `tech-writer` | `docs/`: Zero-Trust model doc, deployment guide, MITRE ATT&CK coverage map, tuning notes. | 1,2,3,4,5 | Yes (with 8) |
| 8 | `readme-specialist` | Polish root README.md (badges, what's-inside, detections, quick start, ZT framing, tuning caveat); update CHANGELOG to v0.1.0. | 3,4,5 | Yes (with 7) |
| 9 | `code-reviewer` | Read-only review: KQL/JSON validity, watchlist join correctness, MITRE/ZT mapping consistency, thresholds documented as tunable, CI completeness; produce a punch list. | 6,7,8 | No (gate) |
| 10 | `devops-engineer` | Fix punch-list items, run the validate workflow, commit at milestones, push to GitHub. | 9 | No |

> A subagent cannot call another subagent, so dispatch each line one at a time from the main
> thread. Commit milestones: after Step 6 verify, after Step 5 content, after Step 8 docs, final at Step 10.

## Reference files (mirror these conventions)

- `C:\Users\<username>\Projects\Sentinelmonitoring\analytics-rules\entra-privileged-role-added.json` — ARM analytics-rule shape (GUID name, tactics/techniques, entityMappings).
- `C:\Users\<username>\Projects\Sentinelmonitoring\hunting-queries\new-legacy-auth-user.yaml` — hunting-query YAML schema (`id`, `name`, `query`, …).
- `C:\Users\<username>\Projects\Sentinelmonitoring\watchlists\host-inventory.csv` + `host-inventory.md` — watchlist CSV + schema-doc pattern.
- `C:\Users\<username>\Projects\Sentinelmonitoring\workbooks\05-entra-identity-compromise.workbook.json` — workbook JSON shape.
- `C:\Users\<username>\Projects\Sentinelmonitoring\README.md` and `docs\` — README and docs style.

## Sourcing

Canonical reusable reference: **Azure/Azure-Sentinel** (Hunting Queries, Solutions/"Microsoft Entra ID",
Workbooks, Watchlists) and Microsoft Learn (Watchlists, UEBA/BehaviorAnalytics, Identity Protection).
Treat community repos as verify-before-use.
