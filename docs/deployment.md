# Deployment guide

How to deploy the `blueteamthreathunting` identity pack into a Microsoft Sentinel workspace.
Everything here is a **starting template** — review thresholds (see [`tuning-notes.md`](tuning-notes.md))
before enabling rules in production.

## Prerequisites

- A **Log Analytics workspace** with **Microsoft Sentinel** enabled.
- The **Microsoft Entra ID data connector** enabled, sending at least:
  - `SigninLogs`, `AADNonInteractiveUserSignInLogs`, `AuditLogs`
  - (recommended) `AADUserRiskEvents` / `AADRiskyUsers` for the Assume-Breach corroboration.
- **Optional — Sentinel UEBA** (`IdentityInfo`, `BehaviorAnalytics`). The queries that use it are
  flagged `requiresUEBA` / carry an inline UEBA note and **degrade gracefully** if it is off
  (remove the marked join block).
- Permissions: `Microsoft Sentinel Contributor` on the workspace; `az` CLI logged in (`az login`).

## 1. Import the watchlists

These tag the accounts the detections boost on. Sentinel portal →
**Configuration → Watchlists → + Add new**:

| Watchlist alias | File | Key (SearchKey) | Columns |
|-----------------|------|-----------------|---------|
| `VIPUsers` | [`watchlists/VIPUsers.csv`](../watchlists/VIPUsers.csv) | `UserPrincipalName` | UserPrincipalName, ObjectId, DisplayName, Department, VIPReason, Notes |
| `PrivilegedAccounts` | [`watchlists/PrivilegedAccounts.csv`](../watchlists/PrivilegedAccounts.csv) | `UserPrincipalName` | UserPrincipalName, ObjectId, DisplayName, AssignedRole, RoleSource, Notes |

> Set the **alias** exactly to `VIPUsers` / `PrivilegedAccounts` — every query calls
> `_GetWatchlist("VIPUsers")` / `_GetWatchlist("PrivilegedAccounts")` by that name.
> The CSV rows are **fictitious examples** (`*.example.com`) — replace with your real accounts.
> Schema details and the join pattern are in each watchlist's companion `.md`.

## 2. Deploy the analytics rules

Each rule in [`analytics-rules/`](../analytics-rules/) is an ARM template. Deploy at resource-group scope:

```bash
az deployment group create \
  --resource-group <your-sentinel-rg> \
  --template-file analytics-rules/impossible-travel.json \
  --parameters workspace=<your-workspace-name>
```

Repeat for `mfa-fatigue.json`, `new-elevated-admin-role.json`, and `mfa-authmethod-tampering.json`.
Rules deploy `enabled: true` — review their thresholds first.

## 3. Run the hunting & zero-trust queries

- **Hunting queries** ([`hunting-queries/`](../hunting-queries/), YAML): paste the `query:` body into
  Sentinel **Logs**, or onboard them via **Hunting** (or the repo CI / a content pipeline). These are
  single-signal, recall-oriented — good for proactive hunts.
- **Zero-Trust correlation queries** ([`zero-trust/`](../zero-trust/), `.kql`, grouped by pillar): paste
  into **Logs**. These are multi-signal, precision-oriented — alert-quality.

## 4. Import the workbook

Sentinel → **Workbooks → + Add workbook → Edit → Advanced Editor**, then paste the contents of
[`workbooks/zt-identity-correlation.workbook.json`](../workbooks/zt-identity-correlation.workbook.json)
→ **Apply → Save**. Set the `TimeRange` and `AccountFilter` parameters at the top.

## Notes

- If UEBA is **not** enabled, the `VIP risky sign-ins` hunting query and the `ve-01` / `lp-01`
  zero-trust queries / workbook tiles still run — remove the join blocks marked `// UEBA-DISABLED`.
- All geovelocity logic uses straight-line (Haversine) distance from `SigninLogs` location data.
- This pack is **identity-focused** and complements the broader
  [`Sentinelmonitoring`](https://github.com/Philliamviber/Sentinelmonitoring) pack — it does not duplicate it.
