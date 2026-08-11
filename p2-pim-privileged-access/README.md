# Project 2 — Privileged Access with PIM

**SC-300 domain:** Access management — plan and implement privileged access
**Environment:** Microsoft Entra ID, cloud-only lab tenant (IAM-LAB)
**Status:** Core lifecycle complete — 7 of 7 evidence items captured, 1 documented gap outside the checklist

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
Conditional Access authentication context. That link is proven two ways in this repo: a live
screenshot of the enforcement banner during activation, and the exported policy JSON showing
the actual authentication context claim wired into the role's assignment policy.

---

## Design

### Role tiering (as designed)

| Tier | Roles | Activation | Approval | Max duration |
|---|---|---|---|---|
| 0 — Critical | Global Administrator | Eligible only | Required, single-stage | 2 hours |
| 1 — High | (planned, not built in this lab) | — | — | — |
| 2 — Standard | (planned, not built in this lab) | — | — | — |

Only Tier 0 (Global Administrator) was actually built out for the test population in this
lab tenant. `pim.test` holds Global Administrator as an **eligible** assignment, not a
standing one.

### The link back to Project 1 — proven in the exported policy, not just described

The exported policy JSON (`config/pim-role-policies.json`) contains the real rule set
governing Global Administrator activation. Two rules matter most:

**Approval is required, with a named approver:**
```json
"isApprovalRequired": true,
"isRequestorJustificationRequired": true,
"approvalMode": "SingleStage",
"primaryApprovers": [{ "description": "Abhi Admin" }]
```

**Activation is bound to a Conditional Access authentication context:**
```json
"isEnabled": true,
"claimValue": "c1"
```

That `claimValue: "c1"` is the literal, machine-readable proof that activating Global
Administrator triggers Conditional Access policy CA150 from Project 1
(`c1-Privileged-Admin-Activation`), rather than relying only on PIM's own native MFA prompt.
This is the strongest single artefact in the project — it shows the cross-project design
decision in configuration, not just in a diagram.

The export also confirms the other design values matched what was intended:
`Expiration_EndUser_Assignment` → `"maximumDuration": "PT2H"` (the 2-hour activation window),
and `Expiration_Admin_Assignment` → `"maximumDuration": "P180D"` (the eligible assignment
itself expires after 180 days if not renewed).

---

## Implementation and what actually happened

The activation flow was run end to end as the test identity `pim.test`, with `abhi.admin`
acting as the configured approver. The real sequence, taken directly from the tenant's PIM
audit log, matches the exported policy exactly — approval was required, and it happened:

| Time | Actor | Event |
|---|---|---|
| 10:56:25 | pim test | Role activation requested |
| 10:56:29 | pim test | Approval requested |
| 10:58:59 | pim test | Duplicate activation attempt — **rejected** (see Runbook) |
| 11:00:08 | Abhi Admin | Request **approved** |
| 11:00:18 | pim test | Activation **completed** |
| 11:07:34 | pim test | Deactivation requested |
| 11:07:41 | pim test | Deactivation **completed** |

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
| 7 | Exported role management policy — approval, auth context, expiration rules | `config/pim-role-policies.json` |

### Config as code

```powershell
$gaRole = Get-MgRoleManagementDirectoryRoleDefinition -Filter "displayName eq 'Global Administrator'"
$gaAssignment = Get-MgPolicyRoleManagementPolicyAssignment -Filter "scopeId eq '/' and scopeType eq 'DirectoryRole' and roleDefinitionId eq '$($gaRole.Id)'"
$gaPolicy = Get-MgPolicyRoleManagementPolicy -UnifiedRoleManagementPolicyId $gaAssignment.PolicyId -ExpandProperty "Rules"

$summary = [PSCustomObject]@{
    RoleName = "Global Administrator"
    PolicyId = $gaAssignment.PolicyId
    Rules    = $gaPolicy.Rules | ForEach-Object { $_ | ConvertTo-Json -Depth 10 | ConvertFrom-Json }
}

$summary | ConvertTo-Json -Depth 10 | Out-File config/pim-role-policies.json
```

Scoped deliberately to Global Administrator alone rather than all 145 built-in directory
roles the tenant returns by default — the point of this export is to prove one role's policy
in full detail, not to dump the entire role catalog.

---

## Runbook — real failures hit during this build

- **Duplicate activation request rejected.** A second click on Activate while the first
  request was still pending produced `Activate role failed — There is already an existing
  pending Role assignment request`. Captured in the audit log as the 10:58:59 event with a
  failure status, sitting right alongside the successful attempts — kept as evidence rather
  than cleaned up, since a real operator hits this exact error the first time they use PIM
  under a slow approval flow.
- **The first policy export came back effectively empty.** An initial attempt at exporting
  role management policies returned a JSON file containing only the literal text `null`. The
  cause: the export loop tried to resolve `RoleDefinitionId` through `Get-MgDirectoryRole`,
  which expects an *activated* directory role instance ID — a different ID space from the
  role definition ID PIM actually uses — so every iteration silently failed. Root-caused by
  isolating each step of the pipeline (`$assignments.Count`, a single policy lookup, then
  `.Rules` directly) rather than re-running the whole script blind, which is what actually
  surfaced that `$policy.Rules` was populated all along — the bug was in how the results were
  being reconstructed afterward, not in the Graph query itself. The corrected script round-
  trips each rule through `ConvertTo-Json | ConvertFrom-Json` instead of relying on
  `.AdditionalProperties`, which resolved it.
- **Access review could not be force-started.** A recurring access review was configured on
  the Global Administrator eligible population (owner: `abhi.admin`, scoped to Microsoft
  Entra directory roles), but Entra's backend does not provide a reliable manual override to
  begin a review before its scheduled activation window, even when the start date is the
  current day. The review exists and is correctly configured; it had not yet transitioned to
  "In progress" at the time evidence was captured.

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
> activation with justification, single-stage approval, and time-boxing, with activation
> additionally gated by a phishing-resistant Conditional Access authentication context, and a
> recurring access review configured on the eligible population. Verified end to end via the
> tenant's own audit log and the exported role management policy.

Backed by: the full PIM audit trail (request → approval → activation → deactivation), the
CA150 authentication-context enforcement screenshot, and the exported policy JSON showing the
approval rule and the `claimValue: "c1"` authentication context binding.

---

## Interview questions this unlocks

1. Walk me through activating a privileged role with approval, end to end.
2. How do you bind PIM activation to a Conditional Access authentication context, and why
   would you do that instead of relying on PIM's own MFA setting?
3. Show me how you'd prove that binding exists without just describing it — what would you
   pull from Graph?
4. What happens if a user double-clicks Activate while a request is already pending?
5. Why would you never rely on a review's scheduled start date to guarantee it begins
   exactly on time?
6. How do you tier privileged roles, and why shouldn't every role get the same activation
   policy?
7. What's missing from this build if you were to take it to production, and in what order
   would you add it back?