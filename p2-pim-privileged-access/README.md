# Project 2 — Privileged Access with PIM

**SC-300 domain:** Access management — plan and implement privileged access
**Environment:** Microsoft Entra ID, cloud-only lab tenant (IAM-LAB)
**Status:** Core lifecycle complete — 6 of 7 evidence items captured, 1 documented gap

---

## Business scenario

Standing privileged access is the finding every auditor flags and every attacker targets.
This project eliminates it for the test population in the lab tenant: instead of an account
holding Global Administrator permanently, the role is **eligible-only**, activated
just-in-time with justification and MFA, time-boxed, and logged end to end.

This project is also where **Project 1's Conditional Access framework stops being a separate
control and becomes part of the privileged-access story.** Global Administrator activation in
this build doesn't just rely on PIM's own MFA setting — it's additionally gated by CA150
(*Privileged role activation requires phishing-resistant MFA*), enforced through a
Conditional Access authentication context. That link is proven with a live screenshot, not
just described — see the evidence table below.

---

## Design

### Role tiering (as designed)

| Tier | Roles | Activation | Approval | Max duration |
|---|---|---|---|---|
| 0 — Critical | Global Administrator | Eligible only | Configured | 2 hours |
| 1 — High | (planned, not built in this lab) | — | — | — |
| 2 — Standard | (planned, not built in this lab) | — | — | — |

Only Tier 0 (Global Administrator) was actually built out for the test population in this
lab tenant. `pim.test` holds Global Administrator as an **eligible** assignment, not a
standing one.

### The link back to Project 1

Global Administrator activation is bound to **CA150 — Privileged role activation requires
phishing-resistant MFA**, via a Conditional Access authentication context
(`c1-Privileged-Admin-Activation`). This means activating the tenant's most sensitive role
triggers a Conditional Access evaluation, not just a PIM-native MFA prompt — a genuinely
senior design pattern, and one that's proven live in this build rather than only described.

---

## Implementation and what actually happened

The activation flow was run end to end as the test identity `pim.test`, with `abhi.admin`
acting as approver where required. The real sequence, taken directly from the tenant's PIM
audit log:

| Time | Actor | Event |
|---|---|---|
| 10:56:25 | pim test | Role activation requested |
| 10:56:29 | pim test | Approval requested |
| 10:58:59 | pim test | Duplicate activation attempt — **rejected** (see Runbook) |
| 11:00:08 | Abhi Admin | Request **approved** |
| 11:00:18 | pim test | Activation **completed** |
| 11:07:34 | pim test | Deactivation requested |
| 11:07:41 | pim test | Deactivation **completed** |

The full cycle — eligible → requested → approved → activated → deactivated → back to
eligible — is captured in the audit log evidence below, with real timestamps, not staged
ones.

---

## Evidence

| # | Item | File |
|---|---|---|
| 1 | Eligible assignment view (requester's own perspective, before activation) | `Screenshot 2026-08-11 at 11.00.16 AM.png` |
| 2 | Activation request panel with CA150 Conditional Access banner | `Screenshot 2026-08-11 at 11.03.16 AM.png` |
| 3 | Active assignment — role activated, end time shown | `Screenshot 2026-08-11 at 11.03.40 AM.png` |
| 4 | PIM resource audit — full request → approval → activation trail | `Screenshot 2026-08-11 at 11.04.57 AM.png` |
| 5 | PIM resource audit — deactivation trail | `Screenshot 2026-08-11 at 11.09.06 AM.png` |
| 6 | Access review configured on the privileged role population | `Screenshot 2026-08-11 at 11.30.46 AM.png` |
| 7 | Exported role management policy as code | `config/pim-role-policies.json` |

### Config as code

```powershell
Connect-MgGraph -Scopes "RoleManagement.Read.Directory"
Get-MgPolicyRoleManagementPolicyAssignment -Filter "scopeId eq '/' and scopeType eq 'DirectoryRole'" -All |
    ConvertTo-Json -Depth 20 | Out-File config/pim-role-policies.json
```

---

## Runbook — real failures hit during this build

- **Duplicate activation request rejected.** A second click on Activate while the first
  request was still pending produced `Activate role failed — There is already an existing
  pending Role assignment request`. This is captured in the audit log as the 10:58:59 event
  with a failure status, sitting right alongside the successful attempts — kept as evidence
  rather than cleaned up, since a real operator hits this exact error the first time they use
  PIM under a slow approval flow.
- **Access review could not be force-started.** A recurring access review was configured on
  the Global Administrator eligible population (owner: `abhi.admin`, scoped to Microsoft
  Entra directory roles), but Entra's backend does not provide a reliable manual override to
  begin a review before its scheduled activation window, even when the start date is the
  current day. The review exists and is correctly configured; it had not yet transitioned to
  "In progress" at the time evidence was captured. Documented honestly rather than forced.

---

## Known gaps

- **"Before" evidence — standing admin assignments prior to PIM conversion — was not
  captured.** By the time evidence-gathering began, `pim.test` was already configured as
  eligible-only. The headline "12 standing admins → 0" before/after artefact described in the
  original project design could not be produced retroactively in this lab. In a real
  engagement this would be captured at the very start of the project, before any conversion
  work begins — noted here as a sequencing lesson for next time.
- **Only Tier 0 (Global Administrator) was built.** Tier 1 and Tier 2 role tiering, with
  their different approval and duration rules, were scoped out of this lab iteration.
- **The access review has not yet produced a completed decision cycle** — see Runbook above.

---

## Resume line

> Implemented Microsoft Entra Privileged Identity Management to eliminate standing
> administrative access for Global Administrator — converted to just-in-time eligible
> activation with justification, approval, and time-boxing, with activation additionally
> gated by a phishing-resistant Conditional Access authentication context, and a recurring
> access review configured on the eligible population.

Backed by: the full PIM audit trail (request → approval → activation → deactivation), the
CA150 authentication-context enforcement screenshot, and the exported role management policy.

---

## Interview questions this unlocks

1. Walk me through activating a privileged role with approval, end to end.
2. How do you bind PIM activation to a Conditional Access authentication context, and why
   would you do that instead of relying on PIM's own MFA setting?
3. What happens if a user double-clicks Activate while a request is already pending?
4. Why would you never rely on a review's scheduled start date to guarantee it begins
   exactly on time?
5. How do you tier privileged roles, and why shouldn't every role get the same activation
   policy?
6. How do you prove to an auditor that a specific privileged action was approved by a named
   individual at a specific time?
7. What's missing from this build if you were to take it to production, and in what order
   would you add it back?
