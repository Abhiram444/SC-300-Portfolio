# Project 1 — Zero Trust Conditional Access Framework

**SC-300 domain:** Access management — plan and implement Conditional Access
**Environment:** Microsoft Entra ID, cloud-only lab tenant (IAM-LAB)
**Status:** Complete — 6 of 7 evidence items captured, 1 documented gap

---

## Business scenario

The lab tenant started with no Conditional Access design — no break-glass accounts, no
documented policy set, and no safe way to test a change before it hits every user. This
project replaced that with a structured, persona-based Zero Trust framework: ten policies,
numbered and scoped by persona, rolled out safely through report-only validation and
What-If testing before any enforcement.

---

## Design

### Personas and policy coverage

| Persona | Definition | Baseline expectation | Policies |
|---|---|---|---|
| Global / all users | Every identity | MFA required, legacy auth blocked, high-risk countries blocked | CA001, CA002, CA004 |
| Admins | Directory role holders | Phishing-resistant MFA, no unmanaged platforms, session limits, PIM activation bound to auth context | CA101, CA102, CA150, CA501 |
| Internals | Employees | Compliant or hybrid-joined device required for sensitive apps | CA201 |
| Externals / guests | B2B identities | MFA required, restricted app scope | CA301 |
| Service accounts / workload identities | Non-human identities | Location-locked, excluded from user MFA policies | **Not yet implemented — known gap** |
| Break-glass | 2 emergency accounts | Excluded from all CA policies | Excluded from all 10 policies below |

### Policy numbering scheme

```
CA000-099  Global baseline
CA100-199  Admin protection
CA200-299  Internal users
CA300-399  Guest / external
CA400-499  Workload identities (not yet built)
CA500-599  Session controls
```

### Policy register

| ID | Purpose | Targets | Excluded | Blast radius if misconfigured |
|---|---|---|---|---|
| CA001 | Block legacy authentication | All users | Break-glass | Locks out apps/scripts still using basic auth |
| CA002 | Require MFA — all users, all apps | All users | Break-glass | Locks out users without MFA registered |
| CA004 | Block sign-in from high-risk countries | All users | Break-glass | Can block legitimate travel/VPN sign-ins |
| CA101 | Admins require phishing-resistant MFA | Directory role holders | Break-glass | Locks admins out of their own roles if unenrolled |
| CA102 | Admins blocked from unmanaged platforms | Directory role holders | Break-glass | Blocks legitimate admin access from personal devices |
| CA150 | Privileged role activation requires phishing-resistant MFA | PIM-eligible admins, via auth context | Break-glass | Misconfigured context can block all role activation |
| CA201 | Require compliant device for sensitive apps | Internal employees | Break-glass | Zero compliant devices = all internal access blocked |
| CA301 | Guests require MFA | External / B2B | Break-glass | Blocks guest collaboration if MFA isn't guided |
| CA501 | Admin session limits | Directory role holders | Break-glass | Overly aggressive frequency hurts admin productivity |
| CA502 | App-enforced restrictions (Office 365) | All users, unmanaged devices | Break-glass | Can block legitimate unmanaged-device Office access |

All 10 policies are currently deployed in **Report-only** mode, pending ringed enablement.

---

## Safe rollout methodology

1. **Break-glass first, always** — two cloud-only emergency accounts created and excluded
   from every policy before any policy work began.
2. **Report-only mode** — every policy launches in report-only; impact is measured before
   enforcement.
3. **Ringed enablement** — pilot group first, then all users (planned; not yet executed in
   this lab).
4. **What-If validation** — each persona tested against the policy set using the
   Conditional Access What-If tool.

---

## Evidence

| # | Item | File |
|---|---|---|
| 1 | Persona → policy design table | `CA-Framework-Documentation.docx` |
| 2 | Report-only impact (CA002, 7-day window, 100% report-only) | `Screenshot 2026-08-11 at 8.02.08 AM.png` |
| 3 | What-If — Admin persona (auth context, PIM activation) | `Screenshot 2026-08-11 at 12.46.01 AM.png` |
| 3 | What-If — Standard user persona (Windows, 2 policies apply) | `Screenshot 2026-08-11 at 12.45.38 AM.png` |
| 3 | What-If — Device registration persona (Linux, 3 policies apply) | `Screenshot 2026-08-11 at 12.46.53 AM.png` |
| 4 | Real Conditional Access block in sign-in logs (error 50097) | `Screenshot 2026-08-11 at 8.04.30 AM.png` |
| 5 | Break-glass sign-in alert firing | **Not implemented — see Known Gaps** |
| 6 | Exported policy set as code | `ca-policies.json` |
| 7 | Policy register | `CA-Framework-Documentation.docx` |

Policy list overview: `Screenshot 2026-08-11 at 12.39.11 AM.png`

### Config as code

```powershell
Connect-MgGraph -Scopes "Policy.Read.All"
Get-MgIdentityConditionalAccessPolicy -All | ConvertTo-Json -Depth 20 | Out-File config/ca-policies.json
```

---

## Runbook — real failures hit during this build

- **SoD-adjacent lesson carried from later projects, applies here too:** admin-direct
  actions in Entra do not automatically bypass safety rails — always verify a control's
  actual enforcement boundary rather than assuming it.
- **Conditional Access attribution gap:** sign-ins with `Conditional Access = Failure` in
  the summary log column did not always map to a specific CA policy in the detail pane —
  Microsoft's own UI notes that interruptions can also come from sign-in risk or user risk
  policies, which aren't listed on the Conditional Access tab. Diagnosed via the per-event
  detail pane rather than assumed from the summary column.
- **Low lab traffic volume:** only 2 sign-in events in a 1-month window carried any
  Conditional Access result at all, both from a `pim test` service-style account. Real
  policy-attributed blocks were harder to capture than in a live tenant — the Report-only
  Policy Impact chart (evidence #2) was the more reliable signal in this environment.

---

## Known gaps

- **Break-glass sign-in alerting** was not built in this iteration. The break-glass
  accounts exist and are correctly excluded from every policy, but no automated
  notification (Sentinel rule, Logic App, or scheduled script) fires on their use. This is
  the top priority for hardening this framework further.
- **Workload identity policies (CA400 range)** were scoped out of this phase. Non-human
  identities need their own containment model — IP/location lock, no interactive sign-in —
  which is a natural next iteration.

---

## Resume line

> Designed and deployed a persona-based Zero Trust Conditional Access framework in
> Microsoft Entra ID (break-glass accounts, report-only rollout, What-If validation across
> three personas), replacing an undocumented policy set with a numbered, auditable
> 10-policy register.

Backed by: `ca-policies.json`, the policy register, and report-only impact evidence.

---

## Interview questions this unlocks

1. How do you roll out a Conditional Access policy without causing an outage?
2. What is a break-glass account and how do you protect and monitor it?
3. Why report-only mode, and what do you look at before enforcing?
4. How does CA differ from sign-in risk / user risk policies, and how do you tell which one
   interrupted a sign-in?
5. Design CA for admins vs. standard users — what differs?
6. What is authentication context and when do you use it with PIM?
7. What would you build next to close the gaps in this framework?
