# Watchlist — `VIPUsers`

Identity context for accounts that carry elevated business risk regardless of their
Entra directory role. A sign-in event, role change, or auth-method modification
involving a VIP account receives a **+10 RiskScore boost** in all hunting queries.
An account that is simultaneously on `PrivilegedAccounts` receives **+25** (see
`docs/zero-trust-model.md` Section 2.4 — Additive VIP + Privileged Risk Boost).

---

## Sentinel watchlist metadata

| Field | Value |
|-------|-------|
| **Alias** | `VIPUsers` |
| **SearchKey (primary join key)** | `UserPrincipalName` |
| **Secondary join key** | `ObjectId` |
| **Source file** | `watchlists/VIPUsers.csv` |

---

## Column schema

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `UserPrincipalName` | string | Yes — primary key | UPN in `user@domain.com` form. Must match `SigninLogs.UserPrincipalName` exactly (case-insensitive). |
| `ObjectId` | string (GUID) | Yes — secondary key | Entra Object ID. Fallback join when UPN changes (rename or domain migration). |
| `DisplayName` | string | Yes | Human-readable name used in workbook tiles and alert enrichment. |
| `Department` | string | Recommended | Business unit / department; used for peer-group context in workbook tiles. |
| `VIPReason` | string | Yes | Why this account is VIP. Accepted values: `Executive`, `BoardMember`, `LegalCounsel`, `ITSecurityLead`, `KeyVendor`, or a custom label. |
| `Notes` | string | No | Free text — change history, ticket reference, next review date. |

---

## Create the watchlist in Sentinel

**Portal:** Microsoft Sentinel → Configuration → Watchlists → New
- Alias: **`VIPUsers`**
- SearchKey: **`UserPrincipalName`**
- Upload: `watchlists/VIPUsers.csv`

**ARM / Bicep:** Use resource type
`Microsoft.OperationalInsights/workspaces/providers/watchlists` with alias `VIPUsers`
and searchKey `UserPrincipalName`.

---

## KQL usage pattern

Every hunting query in `hunting-queries/` uses the following pattern verbatim.
The variable names `VIPList` and `IsVIP` are standardized — do not rename them.

```kql
// Retrieve all VIP accounts from the watchlist
let VIPList = _GetWatchlist("VIPUsers")
    | project UPN = UserPrincipalName,
              ObjId = ObjectId,
              VIPReason;
```

Full join + boost pattern (mirrors `docs/zero-trust-model.md` Section 2.4):

```kql
// ── Watchlist lookups ──────────────────────────────────────────────────────
let VIPList = _GetWatchlist("VIPUsers")
    | project UPN = UserPrincipalName, ObjId = ObjectId, VIPReason;

let PrivList = _GetWatchlist("PrivilegedAccounts")
    | project UPN = UserPrincipalName, ObjId = ObjectId, AssignedRole, RoleSource;

// ── Join + boost ────────────────────────────────────────────────────────────
BaseEvents
| join kind=leftouter (VIPList)  on $left.UserPrincipalName == $right.UPN
| join kind=leftouter (PrivList) on $left.UserPrincipalName == $right.UPN
| extend IsVIP  = isnotempty(VIPReason),
         IsPriv = isnotempty(AssignedRole)
// TUNABLE: +10 VIP-only, +10 Priv-only, +25 both (not cumulative)
| extend RiskBoost = case(IsVIP and IsPriv, 25, IsVIP or IsPriv, 10, 0)
| extend RiskScore = RiskBoost
| extend RiskLabel = case(
    IsVIP and IsPriv, strcat(UserPrincipalName, " [VIP+PRIV — HIGHEST PRIORITY]"),
    IsVIP,            strcat(UserPrincipalName, " [VIP]"),
    IsPriv,           strcat(UserPrincipalName, " [PRIV]"),
    UserPrincipalName)
| sort by RiskScore desc, TimeGenerated desc
```

`leftouter` joins are mandatory: events for accounts **not** on the watchlist still
appear in results — the watchlist controls prioritization, not filtering.

---

## Maintenance

- Review and update this CSV at least quarterly or when executive/VIP roster changes.
- Every row must carry a ticket reference or review date in the `Notes` column.
- After updating the CSV, re-upload to the Sentinel watchlist and verify the alias
  name and SearchKey remain unchanged.

---

## Future enhancement (out of scope for v0.1.0)

For v0.1.0 this watchlist is a hand-maintained static CSV template. A planned
v0.2.0 enhancement will auto-hydrate it by joining `IdentityInfo` against
job-title and department patterns (e.g. `| where JobTitle has_any ("CEO","CFO","General Counsel")`).
Track this work in a GitHub Issue tagged `enhancement / v0.2.0`.

---

*Schema defined by iam-engineer — 2026-06-15. EXAMPLE DATA ONLY — replace all rows
before deployment. All thresholds are tunable; see `docs/tuning-notes.md`.*
