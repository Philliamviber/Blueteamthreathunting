# Watchlist — `PrivilegedAccounts`

Identity context for accounts that hold elevated Entra ID (Azure Active Directory)
directory roles — permanently (`Static`) or via Privileged Identity Management (PIM).
A sign-in, role assignment, or auth-method event involving a privileged account receives
a **+10 RiskScore boost** in all hunting queries. An account simultaneously on `VIPUsers`
receives **+25** (see `docs/zero-trust-model.md` Section 2.4 — Additive VIP + Privileged
Risk Boost).

---

## Sentinel watchlist metadata

| Field | Value |
|-------|-------|
| **Alias** | `PrivilegedAccounts` |
| **SearchKey (primary join key)** | `UserPrincipalName` |
| **Secondary join key** | `ObjectId` |
| **Source file** | `watchlists/PrivilegedAccounts.csv` |

---

## Column schema

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `UserPrincipalName` | string | Yes — primary key | UPN. Must match `AuditLogs` and `SigninLogs` values exactly (case-insensitive). |
| `ObjectId` | string (GUID) | Yes — secondary key | Entra Object ID. Critical for PIM events where UPN may not appear in the log. |
| `DisplayName` | string | Yes | Human-readable name used in workbook tiles and alert enrichment. |
| `AssignedRole` | string | Yes | Entra role display name, e.g. `Global Administrator`, `Privileged Role Administrator`, `Security Administrator`. |
| `RoleSource` | string | Yes | One of three values: `Static` (permanently assigned), `PIM-eligible` (can activate but not currently active), `PIM-active` (currently activated). |
| `Notes` | string | No | Free text — ticket reference, activation expiry, review date, justification. |

### RoleSource semantics (detection-relevant)

`RoleSource` is meaningful for detection logic, not just documentation:

- A `PIM-eligible` account that signs in during off-hours **without** a corresponding PIM
  activation event in `AuditLogs` is a stronger anomaly signal.
- A `PIM-active` account with an upcoming expiry is expected to show role-removal events
  in `AuditLogs`; absence of that event may indicate the activation was extended covertly.
- A `Static` account should never need PIM activation events — flag any such events.

---

## Create the watchlist in Sentinel

**Portal:** Microsoft Sentinel → Configuration → Watchlists → New
- Alias: **`PrivilegedAccounts`**
- SearchKey: **`UserPrincipalName`**
- Upload: `watchlists/PrivilegedAccounts.csv`

**ARM / Bicep:** Use resource type
`Microsoft.OperationalInsights/workspaces/providers/watchlists` with alias
`PrivilegedAccounts` and searchKey `UserPrincipalName`.

---

## KQL usage pattern

Every hunting query in `hunting-queries/` uses the following pattern verbatim.
The variable names `PrivList` and `IsPriv` are standardized — do not rename them.

```kql
// Retrieve all privileged accounts from the watchlist
let PrivList = _GetWatchlist("PrivilegedAccounts")
    | project UPN = UserPrincipalName,
              ObjId = ObjectId,
              AssignedRole,
              RoleSource;
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
// TUNABLE: +10 VIP-only, +10 Priv-only, +25 both (not cumulative — iff semantics)
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
appear in results. The watchlist controls prioritization, not filtering.

---

## Maintenance

- Review and update this CSV at least monthly, or immediately when:
  - A new Global Administrator, Privileged Role Administrator, or Security Administrator
    is assigned.
  - A PIM-eligible role activation expires and the account should be downgraded.
  - A `Static` role assignment is revoked.
- Every row must carry a ticket reference or next-review date in the `Notes` column.
- After updating the CSV, re-upload to the Sentinel watchlist and confirm the alias
  and SearchKey remain unchanged.

---

## Future enhancement (out of scope for v0.1.0)

For v0.1.0 this watchlist is a static CSV template. A planned v0.2.0 enhancement
will auto-refresh `RoleSource` and `AssignedRole` from PIM eligible/active role
assignments using the Microsoft Graph API or by querying `IdentityInfo` in Sentinel
UEBA. Track this work in a GitHub Issue tagged `enhancement / v0.2.0`.

---

*Schema defined by iam-engineer — 2026-06-15. EXAMPLE DATA ONLY — replace all rows
before deployment. All thresholds are tunable; see `docs/tuning-notes.md`.*
