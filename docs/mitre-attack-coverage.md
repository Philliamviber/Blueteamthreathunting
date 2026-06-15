# MITRE ATT&CK Coverage — blueteamthreathunting v0.1.0

Static, reviewable map of what this identity threat-hunting pack detects. It binds each of the
seven v0.1.0 detections to a **MITRE ATT&CK** tactic + technique, the **Zero Trust (ZT) pillar**
it supports, and the primary **Microsoft Sentinel** table(s) it queries. The workbook renders a
live view; this file is the canonical, version-controlled source for the README detection list
and the `analytics-rules/` technique mappings.

> Scope: privileged and VIP-flagged identities in Microsoft Entra ID. No live tenant —
> templates + docs only. Thresholds named here are starting points and are tunable
> (see `docs/tuning-notes.md`).

> Acronyms on first use: MITRE ATT&CK = MITRE Adversarial Tactics, Techniques, and Common
> Knowledge. ZT = Zero Trust. MFA = multi-factor authentication. PIM = Privileged Identity
> Management. UEBA = User and Entity Behavior Analytics. CA = Conditional Access.

---

## 1. Coverage table

ZT pillars: **VE** = Verify Explicitly · **LP** = Use Least Privilege · **AB** = Assume Breach.

| ID | Detection | Tactic | Technique (ID) | More-precise sub-technique | ZT pillar | Primary Sentinel table(s) | Severity | Requires UEBA |
|----|-----------|--------|----------------|----------------------------|-----------|---------------------------|----------|---------------|
| D1 | Impossible travel | Initial Access | Valid Accounts (T1078) | T1078.004 Cloud Accounts | Assume Breach | `SigninLogs` | High | N |
| D2 | MFA fatigue / repeated denials | Credential Access | Brute Force (T1110) | **T1621** MFA Request Generation *(see note)* | Verify Explicitly | `SigninLogs` | High | N |
| D3 | Dormant-admin reactivation | Persistence | Valid Accounts (T1078) | T1078.004 Cloud Accounts | Assume Breach | `SigninLogs` + `AuditLogs` | Medium | N |
| D4 | New / elevated admin-role assignment | Privilege Escalation | Account Manipulation (T1098) | T1098.003 Additional Cloud Roles | Use Least Privilege | `AuditLogs` | High | N |
| D5 | Off-hours privileged role activation | Defense Evasion | Valid Accounts (T1078) | T1078.004 Cloud Accounts | Use Least Privilege | `AuditLogs` (PIM) / `SigninLogs` | Medium | N |
| D6 | VIP risky sign-ins | Initial Access | Valid Accounts (T1078) | T1078.004 Cloud Accounts | Verify Explicitly | `SigninLogs` (`RiskLevelDuringSignIn`) | High | Y |
| D7 | MFA / auth-method tampering | Credential Access | Modify Authentication Process (T1556) | T1556.006 MFA · T1098.005 Device Registration | Verify Explicitly | `AuditLogs` | High | N |

### Mapping notes (be honest about these)

- **D2 technique ID.** The acceptance criteria and the deployable analytics rule map MFA fatigue
  to **T1110 (Brute Force, Credential Access)** so the README, hunting YAML, and ARM rule stay
  consistent. The *more accurate* current ATT&CK technique for push-bombing / prompt-flooding is
  **T1621 — Multi-Factor Authentication Request Generation** (Credential Access). Treat T1110 as
  the locked v0.1.0 mapping and T1621 as the upgrade target for v0.2.0.
- **T1078 tactic spread.** T1078 Valid Accounts is one technique that legitimately spans four
  tactics (Initial Access, Persistence, Privilege Escalation, Defense Evasion). That is why
  D1/D3/D5/D6 share a technique ID but carry different tactics — the tactic, not the ID,
  distinguishes them. This is intentional, not a copy/paste error.
- **D7 dual sub-technique.** Adding an attacker-controlled authenticator is best modeled as
  **T1556.006 (MFA)**; if the tampering takes the form of registering a new device/authenticator
  for persistence, **T1098.005 (Device Registration)** also applies. The rule's primary technique
  stays T1556.
- **T1550 / T1021 in scope?** PLAN.md lists **T1550 (Use Alternate Authentication Material)** and
  **T1021 (Remote Services)** in the technique palette, but **no v0.1.0 detection maps to them.**
  Token-theft / pass-the-PRT (T1550) and lateral movement (T1021) are out of scope here — listed
  as blind spots in section 3 so the gap is explicit rather than implied-covered.

### ZT pillar rollup

| ZT pillar | Detections | What it asserts |
|-----------|-----------|-----------------|
| Verify Explicitly | D2, D6, D7 | Authenticate/authorize on every signal — risk level, MFA integrity, auth-method changes. |
| Use Least Privilege | D4, D5 | Just-enough / just-in-time access — role grants and off-hours PIM activations are scrutinized. |
| Assume Breach | D1, D3 | Presume compromise — geovelocity and dormant-account reuse imply an active intruder. |

---

## 2. Identity attack-path narrative (privileged / VIP accounts)

How a single phished VIP or admin credential walks the kill chain, and where each detection
(D1–D7) is positioned to catch it. Read top-to-bottom as the adversary's path; the dashed lines
show which detection fires at each stage.

```mermaid
flowchart TD
    A["Goal: durable control of a privileged / VIP identity"]:::goal

    subgraph CA["Credential Access"]
        B1["Phish / password spray / token theft"]
        B2["Valid credential obtained"]
    end
    subgraph MFA["MFA Defeat"]
        C1["MFA prompt bombing<br/>(push fatigue)"]
        C2["Register attacker authenticator /<br/>alter auth method"]
        C3["Sign-in succeeds from anomalous geo"]
    end
    subgraph PE["Privilege Escalation"]
        D1n["Assign / elevate admin role"]
        D2n["Activate PIM-eligible role<br/>(often off-hours)"]
    end
    subgraph PERS["Persistence"]
        E1["Reactivate dormant admin account"]
        E2["Keep rogue authenticator as backdoor"]
    end

    A --> B1 --> B2
    B2 --> C1
    B2 --> C2
    C1 --> C3
    C2 --> C3
    C3 --> D1n
    C3 --> D2n
    D1n --> E1
    D2n --> E1
    C2 --> E2

    B2 -.detected by.-> T_D6["D6 VIP risky sign-in (T1078)"]:::det
    C1 -.detected by.-> T_D2["D2 MFA fatigue (T1110 / T1621)"]:::det
    C2 -.detected by.-> T_D7["D7 auth-method tampering (T1556.006)"]:::det
    C3 -.detected by.-> T_D1["D1 impossible travel (T1078)"]:::det
    D1n -.detected by.-> T_D4["D4 new/elevated role (T1098.003)"]:::det
    D2n -.detected by.-> T_D5["D5 off-hours activation (T1078)"]:::det
    E1 -.detected by.-> T_D3["D3 dormant-admin reactivation (T1078)"]:::det

    classDef goal fill:#7b1f1f,stroke:#000,color:#fff;
    classDef det fill:#0b3d0b,stroke:#000,color:#fff;
```

**Reading the path.** Credential access (phish/spray) yields a valid credential. The adversary
then defeats MFA either by fatigue (D2) or by registering their own authenticator (D7); the
resulting sign-in often trips geovelocity (D1) or risk scoring (D6). With a session in hand they
escalate by granting themselves a role (D4) or activating a PIM-eligible role off-hours (D5),
then persist by reusing a dormant admin (D3) or leaving the rogue authenticator in place (E2).
**The watchlist boost is what makes this actionable:** an account present in **both** `VIPUsers`
and `PrivilegedAccounts` scores highest, so a single identity walking three or more of these
stages surfaces at the top of the queue.

### Top-risk attack tree (privilege escalation via compromised VIP/admin)

```mermaid
flowchart TD
    R["Attacker controls a privileged Entra role"]:::root
    O1["AND: obtain credential"]
    O2["AND: defeat MFA"]
    O3["AND: escalate or persist"]

    R --> O1
    R --> O2
    R --> O3

    O1 --> L1["Phish VIP"]
    O1 --> L2["Password spray (T1110)"]
    O1 --> L3["Token / PRT theft (T1550) — BLIND SPOT"]:::blind

    O2 --> M1["Push fatigue (D2)"]
    O2 --> M2["Register rogue authenticator (D7)"]
    O2 --> M3["CA-policy weakness (T1556.009) — partial"]:::blind

    O3 --> N1["Self-assign admin role (D4)"]
    O3 --> N2["Off-hours PIM activation (D5)"]
    O3 --> N3["Reuse dormant admin (D3)"]

    classDef root fill:#7b1f1f,stroke:#000,color:#fff;
    classDef blind fill:#5a4a00,stroke:#000,color:#fff;
```

The two amber leaves (token/PRT theft, CA-policy tampering) are paths the attacker can take that
v0.1.0 does **not** reliably catch — see blind spots below.

---

## 3. Known blind spots — what v0.1.0 does NOT cover

Being explicit so reviewers do not assume coverage that is not there.

| Gap | Technique(s) | Why uncovered in v0.1.0 | Backlog target |
|-----|-------------|--------------------------|----------------|
| Token / PRT theft, pass-the-cookie | T1550 / T1550.001 / T1528 | Needs token-binding + sign-in session-ID correlation; no detection authored. | v0.2.0 |
| Lateral movement after compromise | T1021 (Remote Services) | Out of identity scope; needs `SecurityEvent` / device telemetry not in this pack. | Separate pack |
| Conditional Access policy tampering | T1556.009 | D7 watches auth methods, not CA-policy `AuditLogs` change events. | v0.2.0 |
| OAuth / app-consent grant abuse | T1098.001 (Additional Cloud Credentials), illicit consent | No detection for service-principal credential adds or consent grants. | v0.2.0 |
| Federation / hybrid-identity backdoors | T1556.007 (Hybrid Identity), T1484.002 | Needs federation-trust + AD FS telemetry; cloud-only scope here. | Future |
| MFA-fatigue precise mapping | T1621 vs locked T1110 | D2 intentionally locked to T1110 for ARM-rule consistency; T1621 deferred. | v0.2.0 |
| Service-principal / managed-identity abuse | T1078.004 (non-interactive) | Detections center on interactive user sign-ins; SP sign-in logs not modeled. | v0.2.0 |
| UEBA-dependent fidelity | — | D6 needs Entra Identity Protection / `BehaviorAnalytics`; degrades to raw risk columns if UEBA absent. | Tenant-dependent |

**Detection-confidence caveats inside scope:** D1 (impossible travel) yields false positives on
VPN/corporate-proxy egress and cloud-hosted clients; D5 (off-hours) is timezone-sensitive and
noisy for follow-the-sun admins. Both are tuned, not eliminated — see `docs/tuning-notes.md`.

---

*Maintained by the threat-modeler step. Technique IDs verified against attack.mitre.org
(T1078, T1098, T1110, T1556 and sub-techniques) on 2026-06-15. Review with the blue team each
release. Drives: README detection list, `analytics-rules/` `techniques` arrays, hunting-query
`relevantTechniques`.*
