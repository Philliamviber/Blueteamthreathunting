# Tuning notes

Every detection in this pack ships with **starting-point thresholds**, not production-tuned values.
Tune them against *your* telemetry and document the change here. All thresholds are exposed as named
`let` statements at the top of each query (look for `// TUNE:` comments) or as `tuneThresholds:` in the
hunting-query YAML.

## Threshold reference

| Detection | Parameter | Default | What it controls | How to tune | FP risk if too loose |
|-----------|-----------|---------|------------------|-------------|----------------------|
| Impossible travel | `MaxSpeedKmh` | `800` | Speed above which travel between two sign-ins is "impossible" | Lower to catch slower VPN hops; raises FPs | VPN/proxy egress, mobile carrier-grade NAT |
| Impossible travel | `MinDistanceKm` | `50` | Minimum distance between sign-ins before computing speed | Raise to suppress same-metro noise | Same-city dual sign-ins |
| Impossible travel | `LookbackWindow` | `1d` | Evaluation window per run | Match to rule frequency | — |
| MFA fatigue | `DenialThreshold` | `5` | MFA denials in the window to flag push-bombing | Raise on kiosks/shared devices | High legitimate MFA failure rate |
| MFA fatigue | window | `10 min` | Burst window for denials | Shorten to require tighter bursts | Spread-out legitimate retries |
| MFA fatigue | rule frequency | `PT15M` | How often the analytics rule runs | Balance latency vs cost | — |
| Dormant-admin reactivation | inactivity window | `30 days` | "Dormant" cutoff before a new sign-in is suspicious | Raise for seasonal admins | Infrequent-but-legitimate admins |
| Off-hours privileged activation | business hours | `07:00–19:00` + weekends off | Defines "off-hours" role activation | Set to your org's hours/timezone | Global teams / shift work |
| New/elevated admin role | PIM lookback | `2h` | Window to confirm a PIM activation preceded an assignment | Widen if PIM logging lags | Slow PIM propagation |
| MFA/auth-method tampering | tamper window | `24h` | Window correlating method changes to a denial burst | Shorten to tighten the chain | Legitimate method rotation |
| All watchlist-boosted | `BoostVIP` | `+10` | RiskScore added for VIP-only accounts | Org preference | — |
| All watchlist-boosted | `BoostPriv` | `+10` | RiskScore added for privileged-only accounts | Org preference | — |
| All watchlist-boosted | `BoostBoth` | `+25` | RiskScore for accounts on BOTH watchlists | **Keep > BoostVIP and BoostPriv** | — |

## Boost logic (consistent across the pack)

The watchlist boost is `case()`-based, not cumulative addition:

```kusto
| extend RiskBoost = case(IsVIP and IsPriv, 25, IsVIP or IsPriv, 10, 0)
```

An account on **both** watchlists scores `25` (highest priority), a single-list account `10`, and an
untagged account `0` — but untagged accounts still appear (the joins are `leftouter`).

## UEBA-dependent items

These require Sentinel UEBA (`IdentityInfo` / `BehaviorAnalytics`) and degrade gracefully if it is off —
remove the join block marked `// UEBA-DISABLED`:

- `hunting-queries/vip-risky-sign-ins.yaml`
- `zero-trust/verify-explicitly/ve-01-risky-signin-ca-gap.kql` (BehaviorAnalytics)
- `zero-trust/least-privilege/lp-01-role-assignment-outside-pim.kql` (IdentityInfo)
- The corresponding workbook tiles (flagged inline).
