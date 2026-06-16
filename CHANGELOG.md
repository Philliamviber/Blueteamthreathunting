# Changelog

All notable changes to this project are documented here. Format loosely follows
[Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

## [0.1.0] - 2026-06-15

### Added
- **Watchlists** (`watchlists/`): `VIPUsers` and `PrivilegedAccounts` (CSV + schema docs), keyed on
  `UserPrincipalName` / `ObjectId`, used for the additive VIP+privileged risk boost.
- **Hunting queries** (`hunting-queries/`): 7 identity detections — impossible travel, MFA fatigue,
  dormant-admin reactivation, new/elevated admin role, off-hours privileged role activation, VIP risky
  sign-ins, MFA/auth-method tampering — each with watchlist boost, tunable thresholds, and ATT&CK mapping.
- **Zero-Trust correlation queries** (`zero-trust/`): 6 multi-signal, precision-tuned queries grouped by
  pillar (verify-explicitly, least-privilege, assume-breach).
- **Analytics rules** (`analytics-rules/`): 4 deployable scheduled-rule ARM templates with GUIDs, ATT&CK
  techniques, and entity mappings.
- **Workbook** (`workbooks/`): Zero-Trust identity correlation workbook with per-pillar tiles, a summary
  grid, and time-range + account-filter parameters.
- **Docs** (`docs/`): Zero-Trust model, MITRE ATT&CK coverage map, deployment guide, tuning notes, and
  the v0.1.0 acceptance criteria.
- Repo scaffold: CI validation workflow, `.gitignore`, contributor subagent definitions (`.claude/agents/`),
  and the build delegation plan (`PLAN.md`).

### Notes
- All detections are **starting templates** requiring threshold tuning against real telemetry.
- Some queries require Sentinel **UEBA** and degrade gracefully if it is disabled.
