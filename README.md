# Blue Team Threat Hunting — Identity Tracker

> A Microsoft Sentinel **blue-team identity threat-hunting** pack that uses **Zero Trust** concepts to correlate anomalous behavior of **privileged accounts** and **VIP-flagged accounts**.

![Platform](https://img.shields.io/badge/platform-Microsoft%20Sentinel-blue)
![Identity](https://img.shields.io/badge/identity-Entra%20ID-purple)
![Model](https://img.shields.io/badge/model-Zero%20Trust-0a7e07)
![Framework](https://img.shields.io/badge/mapped%20to-MITRE%20ATT%26CK-red)

## Idea

Privileged and VIP accounts are the highest-value identity targets. This pack tags those
accounts with **Sentinel watchlists**, then hunts for behavior that violates Zero Trust
expectations — organised by the three Zero Trust pillars:

| Zero Trust pillar | What we watch | Primary tables |
|-------------------|---------------|----------------|
| **Verify explicitly** | MFA outcomes, device/location trust, Conditional Access verdicts | `SigninLogs`, `AADNonInteractiveUserSignInLogs` |
| **Use least privilege** | Role/group additions, PIM activations, off-hours elevation | `AuditLogs`, `IdentityInfo` |
| **Assume breach** | Risk events, UEBA anomalies, dormant-admin reactivation | `AADUserRiskEvents`, `BehaviorAnalytics`, `OfficeActivity` |

## What's inside (v0.1.0 — building)

| Area | Folder |
|------|--------|
| 🆔 VIP + privileged-account watchlists (CSV + schema) | [`watchlists/`](watchlists/) |
| 🔎 Identity hunting queries (KQL/YAML) | [`hunting-queries/`](hunting-queries/) |
| 🚨 Deployable scheduled analytics rules (ARM/JSON) | [`analytics-rules/`](analytics-rules/) |
| 📊 Zero-Trust identity correlation workbook | [`workbooks/`](workbooks/) |
| 🧭 Zero-Trust correlation queries (by pillar) | [`zero-trust/`](zero-trust/) |
| 📚 Architecture, deployment, MITRE map, tuning notes | [`docs/`](docs/) |

> **Status:** this is the scaffold. The full content pack is produced by an automated
> build (see [`PLAN.md`](PLAN.md) for the tech-lead delegation plan driving it).

## Detections (planned)

Impossible travel · MFA fatigue / repeated MFA denials · dormant-admin reactivation ·
new or elevated admin-role assignment · off-hours privileged role activation · VIP risky
sign-ins · authentication-method / MFA tampering — each with a privilege/VIP **boost** via
`_GetWatchlist()` joins and mapped to MITRE ATT&CK (T1078, T1098, T1556, T1110, T1550, T1021).

## Important notes

- **These are starting templates, not pre-tuned detections.** Thresholds (travel speed,
  MFA-denial counts, dormant window, off-hours definition) must be tuned to *your* telemetry.
- **Some queries require Sentinel UEBA** (`IdentityInfo`, `BehaviorAnalytics`) — they are
  marked so they degrade gracefully if UEBA is not enabled.
- Complements (does not duplicate) the broader [`Sentinelmonitoring`](https://github.com/Philliamviber/Sentinelmonitoring) pack — this repo is identity-focused.

## License

Provided as-is for defensive security use. Review before deploying to production.
