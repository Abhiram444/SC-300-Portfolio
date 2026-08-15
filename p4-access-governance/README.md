# Project 4 — Access Governance: Entitlement Management, Access Reviews & SoD

**SC-300 domain:** Identity governance — plan and implement entitlement management and access reviews
**Environment:** Microsoft Entra ID, cloud-only lab tenant (IAM-LAB)
**Status:** Complete — all major components built and evidenced, with one significant documented finding

---

## Business scenario

Access granted by ticket and free text. No catalog, no owner, no expiry, no recertification, no
conflict checking. When an auditor asks *"who approved this access, and when was it last
reviewed?"*, there is no answer.

This project replaces that with a governed access catalog where access is a defined product —
it has an owner, an approval path, an expiry, and a conflict policy — plus recurring
certification campaigns that recheck it over time.

This is the governance heart of the portfolio and the closest thing Microsoft Entra has to what
SailPoint does. For anyone targeting IGA or governance-engineering roles, this is the project
that matters most.

---

## Design

### Catalogs — governance boundaries

| Catalog | Purpose | External users enabled |
|---|---|---|
| Finance Applications | AP Analyst / AP Approver groups | No |
| Engineering Tooling | Prod read/write groups | No |
| Elevated Access | Privileged groups and admin app roles | No |
| Contractor Access | External/guest identities, time-boxed | **Yes** |

A catalog is a trust boundary, not access itself. Resources must be explicitly adopted into a
catalog before any package can reference them — a detail that caused a real early failure in
this build (see Runbook).

### Access packages

| Package | Approval | Expiry | Notes |
|---|---|---|---|
| Finance - AP Analyst | Manager, 1 stage | 180 days | SoD conflict with AP Approver |
| Finance - AP Approver | Manager + Finance owner, 2 stage | 90 days | SoD conflict with AP Analyst |
| Eng - Prod Read | Manager (request path) | 365 days | **Two policies** — see below |
| Eng - Prod Write | Manager + Security, 2 stage | 30 days | Short-lived by design |
| Contractor - Base | Sponsor + Security, 2 stage | 90 days, no extension | Guest lifecycle |

### The two-policy pattern

`Eng - Prod Read` deliberately carries two policies on the same package:

- **Initial Policy** — self-service request path, requires justification and manager approval
- **Direct Assignment - Birthright** — administrator direct assignments only, not visible or
  requestable to end users

Same underlying access, two entry paths, modelled explicitly rather than by creating two
near-duplicate packages. This is the pattern to reach for when access is birthright for one
population and request-based for another.

### Separation of Duties — both directions

- **Preventive:** `Finance - AP Analyst` and `Finance - AP Approver` are configured as mutually
  incompatible access packages. A user holding one cannot request the other.
- **Detective:** a preventive control does nothing about conflicts that already exist. Proving
  that gap required deliberately manufacturing a pre-existing conflict — see Evidence.

### Certification campaigns

| Campaign | Scope | Reviewer | Non-response |
|---|---|---|---|
| A — Department groups | Dept groups | Manager | Remove access |
| B — Privileged roles | Global Administrator (via PIM) | Security | **No change + escalate** |
| C — Access packages | Package assignment policies | Package owner | Remove access |
| D — Guests | Contractor Access catalog | Self, then sponsor | Remove access |

Campaign B's non-response setting is deliberately different. Auto-removing a privileged role
because a reviewer didn't respond in time can lock out the entire admin population. Knowing when
*not* to auto-remove is the maturity signal here, not knowing how to turn auto-remove on.

---

## The headline finding — a recorded decision is not a revocation

This is the most important thing proven in this project, and it was found by testing rather
than assumed.

In Campaign B, two accounts (`breakglass1`, `breakglass2` — unused test artefacts, not the
Project 1 emergency-access accounts) were **denied** by the reviewer. Every governance surface
reported success:

- The review's **Results** page showed `Outcome: Denied`, reviewed by Abhi Admin, timestamped
- The **audit log** recorded `Activity: Deny decision`, `Status: Success`
- The review **Overview** showed 1 Approved / 2 Denied against a live, active review

And yet, in **PIM → Microsoft Entra roles → Assignments**, both denied accounts were still
listed under Global Administrator: `State: Assigned`, `Membership: Direct`, `End time:
Permanent`.

The cause is visible in the Results page itself: the **"Apply result" column was empty for every
row**. The review was still in `Active` status, so decisions had been recorded but never applied
to the resource. The `Apply` button on the review's Overview page was greyed out, only becoming
available once the review is stopped or reaches its end date.

**The takeaway:** an access review reporting "Denied — Success" describes a *decision*, not an
*outcome*. If a governance programme only checks the review's own results page, it will report
100% completion while the access it denied is still live in production. This is exactly why a
separate verification layer has to exist, and why it must check the resource rather than the
governance tool's own record-keeping.

---

## Evidence

| Item | What it shows |
|---|---|
| SoD preventive block (My Access portal) | User holding AP Analyst blocked from requesting AP Approver — `"You cannot request access as you currently have incompatible group membership(s) and/or access package assignment(s)"`, Continue button disabled |
| SoD detective scenario (4-screenshot set) | Both conflicting packages held by the same identity, SoD rule active and bidirectional, no auto-remediation |
| Two-policy package | `Eng - Prod Read` Policies list showing Initial Policy + Direct Assignment - Birthright |
| Campaign B — decisions recorded | Review with real Approve/Deny decisions, recommended actions based on sign-in recency |
| Campaign B — Results page | `Outcome: Denied` with **Apply result column blank** |
| Campaign B — audit log | `Access Reviews / Policy / Deny decision / Success` |
| **Revocation gap** | PIM Assignments showing both denied accounts still `Assigned, Permanent` |
| Config as code | `config/catalogs.json`, `config/access-packages.json` |
| Exception policy | `sod-exception-record.md` |

### Config as code

```powershell
Connect-MgGraph -Scopes "EntitlementManagement.Read.All"
Get-MgEntitlementManagementCatalog -All | ConvertTo-Json -Depth 10 | Out-File config/catalogs.json
Get-MgEntitlementManagementAccessPackage -All | ConvertTo-Json -Depth 10 | Out-File config/access-packages.json
```

The catalog export confirms `Contractor Access` carries `IsExternallyVisible: true`, matching
the design intent that only that one catalog accepts external identities.

---

## Runbook — real failures hit during this build

- **Dynamic groups expose only the Owner role in access packages.** The first attempt at
  building the AP Analyst package targeted `Dept-Finance`, a dynamic group from Project 3. Only
  `Owner` appeared in the role dropdown — never `Member` — because a package cannot add someone
  to a group whose membership is computed by a rule. Fix: static/assigned groups for anything
  an access package needs to grant.

- **Resources must be adopted into a catalog before a package can see them.** A group existing
  in the tenant is not enough; the catalog has to explicitly include it under its Resources tab.
  The symptom is a package showing "no groups in this catalog" despite the group plainly
  existing.

- **An admin-direct assignment still routes through the approval chain.** Assigning a package
  directly with `Bypass approval = No` created a pending request that sat unapproved because no
  approver was available, rather than granting access immediately. Two such requests stacked up
  before the cause was identified.

- **SoD blocks even administrator-forced assignments.** Attempting to admin-assign a conflicting
  package returned `{"accessPackageAssignments":["Finance - AP Analyst"]}` — the platform
  enforces the incompatibility rule against direct admin action too, not just self-service
  requests. This is stronger behaviour than expected and worth knowing; the initial assumption
  that admin-direct would bypass SoD was wrong.

- **An SoD rule cannot be removed while active grants exist on either package.** Deletion failed
  with `The entitlement: Finance - AP Analyst can not be deleted because there are active
  grants` — a deliberate guard preventing an admin from casually loosening a control that is
  currently protecting real assignments. Manufacturing the detective-SoD test scenario therefore
  required removing the user's assignment first, then the rule, then reassigning both packages,
  then reinstating the rule.

- **Access package reviews are not created from the general access review creator.** The
  Identity Governance → Access reviews → New flow only offers Teams+Groups and Applications.
  Package reviews live inside each package's assignment policy, under the Lifecycle tab. Two
  wrong paths were tried before finding it.

---

## Known gaps

- **The nested-group revocation case (T10) was not specifically tested.** The revocation gap was
  proven at the direct-assignment level instead, which demonstrates the same underlying
  principle. Nested-group survival remains the more subtle version of this failure and would be
  the next test to build.
- **Campaign C was configured on a subset of packages rather than all five.** The pattern is
  proven; extending it to every package is repetition rather than new evidence.
- **The revocation gap was documented rather than closed.** The `Apply` action was not run to
  completion, so the "deny → apply → access actually removed" full loop is evidenced only up to
  the gap. Closing that loop is a small remaining step.

---

## Resume line

> Built enterprise access governance in Microsoft Entra — a four-catalog access-package
> structure with multi-stage approvals, time-bound access, and segregation of duties enforced
> both preventively and detectively; ran manager, privileged-role, package-owner and guest
> certification campaigns, and proved through direct verification that a recorded "Deny" review
> decision does not by itself revoke the underlying access.

Backed by: catalog and package exports, the SoD preventive block and detective conflict
evidence, the four-screenshot revocation-gap chain, and a formal SoD exception policy.

---

## Interview questions this unlocks

1. Preventive vs detective SoD — why do you need both, and what does each one miss?
2. Your access review says 100% of decisions were applied, but the user still has access. Walk
   me through how you'd find out why.
3. Why must a policy exception have an expiry date, and what happens operationally if it
   doesn't?
4. Why would you ever set a privileged-role review to *not* auto-remove on non-response?
5. What goes in an access package versus a birthright group, and how do you decide?
6. How would you model access that's automatic for one population and request-based for
   another?
7. You need to test whether your SoD control catches pre-existing conflicts, but the platform
   won't let you create one. How do you get a valid test?
