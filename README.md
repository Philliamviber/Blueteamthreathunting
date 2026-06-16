# Blue Team Threat Hunting — Identity Tracker

> A Microsoft Sentinel **blue-team identity threat-hunting** pack that uses **Zero Trust** concepts to
> correlate anomalous behavior of **privileged accounts** and **VIP-flagged accounts**.

![Platform](https://img.shields.io/badge/platform-Microsoft%20Sentinel-blue)
![Identity](https://img.shields.io/badge/identity-Entra%20ID-purple)
![Model](https://img.shields.io/badge/model-Zero%20Trust-0a7e07)
![Framework](https://img.shields.io/badge/mapped%20to-MITRE%20ATT%26CK-red)

## Idea

Privileged and VIP accounts are the highest-value identity targets. This pack tags those accounts with
**Sentinel watchlists**, then hunts for behavior that violates Zero Trust expectations — organised by the
three Zero Trust pillars. Accounts on a watchlist get a **risk boost** so they sort to the top; accounts
on **both** lists score highest.

| Zero Trust pillar | What we watch | Primary tables |
|-------------------|---------------|----------------|
| **Verify explicitly** | MFA outcomes, device/location trust, Conditional Access verdicts | `SigninLogs`, `AADNonInteractiveUserSignInLogs`, `AADRiskyUsers` |
| **Use least privilege** | Role/group additions, PIM activations, off-hours elevation | `AuditLogs`, `IdentityInfo` |
| **Assume breach** | Risk events, UEBA anomalies, dormant-admin reactivation | `AADUserRiskEvents`, `BehaviorAnalytics`, `SigninLogs` |

## What's inside

| Area | Folder | Count |
|------|--------|-------|
| 🆔 VIP + privileged-account watchlists (CSV + schema) | [`watchlists/`](watchlists/) | 2 |
| 🔎 Identity hunting queries (YAML) | [`hunting-queries/`](hunting-queries/) | 7 |
| 🧭 Zero-Trust correlation queries (by pillar) | [`zero-trust/`](zero-trust/) | 6 |
| 🚨 Deployable scheduled analytics rules (ARM) | [`analytics-rules/`](analytics-rules/) | 4 |
| 📊 Zero-Trust identity correlation workbook | [`workbooks/`](workbooks/) | 1 |
| 📚 Docs (ZT model, MITRE map, deployment, tuning) | [`docs/`](docs/) | 5 |

## Detections

| # | Detection | MITRE ATT&CK | ZT pillar | Notes |
|---|-----------|--------------|-----------|-------|
| 1 | Impossible travel (privileged/VIP) | T1078.004 | Assume Breach | Haversine geovelocity |
| 2 | MFA fatigue / repeated denials | T1110 | Verify Explicitly | push-bombing (T1621 in v0.2.0) |
| 3 | Dormant-admin reactivation | T1078 | Assume Breach | 30-day dormancy |
| 4 | New / elevated admin role | T1098.003 | Least Privilege | role add outside PIM |
| 5 | Off-hours privileged role activation | T1078 | Least Privilege | business-hours window |
| 6 | VIP risky sign-ins | T1078 | Verify Explicitly | **requires UEBA** |
| 7 | MFA / auth-method tampering | T1556.006 | Verify Explicitly | backdoor auth method |

Four of these are also promoted to deployable analytics rules. Full mapping + attack paths in
[`docs/mitre-attack-coverage.md`](docs/mitre-attack-coverage.md).

## Quick start

```bash
# 1. Import the watchlists (portal) with aliases VIPUsers and PrivilegedAccounts (key UserPrincipalName)
#    Sentinel > Configuration > Watchlists > + Add new > upload watchlists/*.csv

# 2. Deploy an analytics rule
az deployment group create \
  --resource-group <your-sentinel-rg> \
  --template-file analytics-rules/impossible-travel.json \
  --parameters workspace=<your-workspace-name>

# 3. Import the workbook (portal)
#    Sentinel > Workbooks > Add workbook > Edit > Advanced Editor
#    paste workbooks/zt-identity-correlation.workbook.json > Apply > Save
```

Full steps in [`docs/deployment.md`](docs/deployment.md).

## Documentation

- [`docs/zero-trust-model.md`](docs/zero-trust-model.md) — the ZT identity model, watchlist data design, boost logic
- [`docs/mitre-attack-coverage.md`](docs/mitre-attack-coverage.md) — detection → ATT&CK/pillar map + attack paths
- [`docs/deployment.md`](docs/deployment.md) — deploy/import steps
- [`docs/tuning-notes.md`](docs/tuning-notes.md) — every tunable threshold and how to tune it

## Important notes

- **These are starting templates, not pre-tuned detections.** Travel speed, MFA-denial counts, dormancy
  window and off-hours must be tuned to *your* telemetry — see [`docs/tuning-notes.md`](docs/tuning-notes.md).
- **Some queries require Sentinel UEBA** (`IdentityInfo`, `BehaviorAnalytics`); they are marked and degrade
  gracefully if UEBA is off.
- **Identity-focused and complementary** to the broader
  [`Sentinelmonitoring`](https://github.com/Philliamviber/Sentinelmonitoring) pack — it does not duplicate it.
- This repo was built via an agent delegation plan ([`PLAN.md`](PLAN.md)); the contributor subagent
  definitions live in [`.claude/agents/`](.claude/agents/).

## License

Provided as-is for defensive security use. Review before deploying to production.
