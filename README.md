# Microsoft Entra ID — Identity & Access Engineering Portfolio

## What this repository is

This is not five unrelated labs. It is one identity programme, built in stages, where each
project hands the next one something real to build on. A recruiter or interviewer should be
able to open any single project folder and see working configuration, exported config-as-code,
evidence that the control actually does what it claims, and a runbook of real failures — not
just a green checkmark. But the stronger story only shows up when you read them in order,
because P2 governs the admins P1 protects, P3 populates the identities P4 governs, and P5
plugs applications into the access model P4 already built.

The single sentence this portfolio is trying to prove to a recruiter is: **this person can
stand up and govern enterprise identity end to end** — not just click through a wizard once,
but design it, break it, fix it, and prove it with exported evidence.

---

## The environment everything lives in

| Item | Value |
|---|---|
| Tenant | `rabhi060gmail.onmicrosoft.com`, internally called IAM-LAB |
| Licensing | Entra ID P2 / Entra ID Governance |
| Population | Synthetic accounts across departments and roles — admin accounts (`abhi.admin`), test/service accounts (`pim.test`), department test users (`helma.hassel`), break-glass accounts |
| Automation | Microsoft Graph PowerShell SDK, run locally and exported to JSON per project |
| Pattern | Every control is built manually first in the Entra admin center, understood end to end, then automated or exported — never automated before it's understood |

---

## How the five projects connect

Picture the identity lifecycle of a single employee at a company that runs this exact setup.

**The day they're hired**, Project 3's HR-driven Joiner workflow provisions their identity —
sourced attributes from an HR feed, a collision-safe username, birthright group membership by
department. The moment that identity exists, **Project 1's Conditional Access framework** is
already waiting for it: their persona (Internal employee, or Admin if they're hired into IT)
determines which of the ten CA policies apply to every sign-in they ever make — MFA, device
compliance, session limits, geo-blocking.

**If their job involves any administrative work**, they never get standing admin rights.
**Project 2's PIM configuration** means Global Administrator, or any privileged role, is
eligible-only — they activate it just-in-time, with approval and phishing-resistant MFA bound
in through Project 1's authentication-context policy (CA150). This is the direct wire between
Project 1 and Project 2: PIM activation for the most sensitive role in the tenant is itself
gated by a Conditional Access policy, not just PIM's own settings.

**Whatever access they need beyond their birthright group** — a finance system, an engineering
production environment, a contractor tool — comes from **Project 4's access packages**, not
from someone manually adding them to a group. That request goes through an approval chain,
carries an expiry, and if it conflicts with another access they hold (like AP Analyst and AP
Approver — a classic separation-of-duties violation), the request is blocked before it's
granted. If it somehow existed before the SoD rule was configured, a detective check catches
it after the fact. Recurring access reviews recertify all of this on a schedule, so access
doesn't just silently persist because nobody remembered to check.

**Every application they touch** — email, an internal tool, a SaaS platform — is meant to be
onboarded the way **Project 5** describes: federated sign-in so there's no local password to
leak, automated provisioning so an account exists the moment they're assigned, and that
assignment itself wrapped in one of Project 4's access packages so it's requestable and
reviewable like everything else, not a special case.

**The day they leave**, Project 3's Leaver workflow fires in a specific order — revoke active
sessions first, then disable the account, then strip groups, licenses, and any PIM-eligible
roles, then (per Project 5's design) deprovision their accounts in every SCIM-connected app.
A verification script checks that this actually happened, because a workflow reporting
"success" and access actually being gone are two different claims, and only one of them
deserves to be trusted without proof.

That's the whole loop: **P1 protects every sign-in → P2 protects every privileged action → P3
drives the lifecycle event → P4 governs everything requested outside the birthright default →
P5 extends that same governance model out to individual applications.** No project in this
repo is a standalone demo; each one is a layer in the same system.

---

## Project 1 — Zero Trust Conditional Access Framework

**Status: complete.** [Full README](p1-conditional-access/README.md)

The starting problem was a tenant with ad-hoc Conditional Access policies added one incident
at a time — no design, no break-glass account, no safe way to test a policy change without
risking a lockout. The fix was a persona-based framework: six personas (global/all users,
admins, internals, guests, service accounts, break-glass), ten numbered policies (CA001
through CA502) mapped against those personas, and a rollout discipline that never skips a
step — break-glass accounts excluded from everything and built first, every policy launched
in report-only mode, impact measured with the What-If tool before anything is enforced, and
only then a ringed rollout to real users.

What's actually proven with evidence: the report-only impact of a live policy (100% of
evaluated sign-ins were in report-only, non-blocking mode), What-If simulations run against
three different personas (a standard user on Windows, an admin activating a privileged role
on macOS through an authentication context, and a device-registration flow on Linux), the
exported policy set as JSON via Graph, and a full policy register documenting the blast radius
of misconfiguring each policy. The one deliberately unfinished piece is break-glass sign-in
alerting — the accounts exist and are correctly excluded from every policy, but no automated
notification fires when they're used. That's flagged, not hidden.

---

## Project 2 — Privileged Access with PIM

**Status: in progress.** [Full README](p2-pim-privileged-access/README.md)

The problem this solves is standing privileged access — accounts holding Global Administrator
or other high-impact roles permanently, "because it's easier," with no approval, no
justification, no time limit, and no review. If one of those accounts is compromised, the
tenant is gone. The fix is Privileged Identity Management: every high-privilege role becomes
eligible-only rather than permanently active, tiered by risk (Global Admin and other Tier 0
roles need approval and a short activation window; lower-tier roles need MFA and justification
but not approval), with recurring access reviews on the eligible population and — this is the
piece that ties back to Project 1 — Global Administrator activation specifically requires
phishing-resistant MFA through a Conditional Access authentication context, not just PIM's own
MFA setting.

What's proven so far: the eligible-assignments view showing a test account (`pim.test`) holding
Global Administrator as Eligible rather than a standing Active assignment — the literal
"before/after" headline artefact this project is built around. Still to capture: the activation
request flow itself with justification and approval, the audit trail showing automatic
deactivation once the time window expires, and a completed access review on the privileged role
population.

---

## Project 3 — Identity Lifecycle Automation (Joiner / Mover / Leaver)

**Status: in progress.** [Full README](p3-identity-lifecycle/README.md)

This is the backbone every other governance control depends on, because none of it works if
the underlying identity data is wrong. The scenario: new joiners provisioned by hand take a
day and get inconsistent access, leavers keep access for weeks after they're gone, and movers
accumulate every permission they've ever had across every team. The fix makes HR the single
source of truth — every attribute either sourced directly from an HR feed or derived by a
documented expression (a collision-safe username generation scheme is part of this), with
Lifecycle Workflows automating three events: Joiner (provision, assign birthright groups,
enable, welcome), Mover (remove old department access, add new), and Leaver, where the *order*
of operations is the entire point — revoke active sessions before disabling the account, because
otherwise an existing token stays valid until it naturally expires regardless of the account
being disabled.

What's captured so far: a clean Leaver workflow execution — one user processed, one success,
zero failures, five tasks completed — for a test identity (`Sharlene Brooks`). Still needed: a
deliberately engineered failure case (a joiner with no manager on file, so the Temporary Access
Pass task fails while everything else succeeds — proving the workflow degrades partially rather
than catastrophically), the Logic App custom extension callback for the Mover flow, and the
verification script output that proves a leaver's access was actually and completely removed,
not just reported as removed.

---

## Project 4 — Access Governance: Entitlement Management, Access Reviews & SoD

**Status: in progress, and the deepest evidence trail in this repo so far.**
[Full README](p4-access-governance/README.md)

This is the governance heart of the whole portfolio, and the project an interviewer targeting
IGA or SailPoint-adjacent roles will look at hardest. The problem: access granted by ticket and
free text, no catalog, no owner, no expiry, no recertification, no conflict checking — the
exact state that makes an auditor's question ("who approved this and when was it last
reviewed?") unanswerable. The fix is a governed access catalog: four catalogs as trust
boundaries (Finance Applications, Engineering Tooling, Elevated Access, Contractor Access, the
last one specifically enabled for external/guest users), five access packages built inside
them with real policies — approval chains, expiry windows, and in one case (Engineering — Prod
Read) two separate policies on the same package, one for direct/birthright assignment and one
for a self-service request path.

The centerpiece of what's actually been proven here is Separation of Duties, tested both
directions the project design calls for. **Preventively:** a test identity holding Finance —
AP Analyst tried to request the conflicting Finance — AP Approver package through the
self-service portal and was blocked outright, with the platform's own error message and a
"See details" drill-down captured as evidence. **Detectively:** because the platform's SoD
engine turned out to block conflicts even on admin-forced direct assignments — a stronger
design than expected, and worth noting as a genuine finding rather than an assumption — the
only way to create a real pre-existing-conflict scenario was to temporarily remove the SoD
link, assign both conflicting packages to the same identity, and then re-add the link. That
produced exactly the scenario the project design is built around: a real SoD violation that
exists and persists even with the preventive rule active, because the rule only screens new
requests, not the access that already existed. That's the entire argument for why a detective
check has to exist as a separate control from a preventive one, proven rather than asserted.

Still ahead: building and running the four certification campaigns (department groups,
privileged roles via PIM, the access packages themselves, and guest identities — each with
different reviewers and, deliberately, different non-response behaviour, since auto-removing a
privileged role from a non-responding reviewer is a real way to lock out an admin population),
proving revocation actually lands including the nested-group edge case the project design
singles out as a "crown jewel" test, and the JSON exports plus a formal campaign-completion
report.

---

## Project 5 — Application Integration: SSO Federation + SCIM Provisioning

**Status: not started.** [Full README](p5-app-integration-sso-scim/README.md)

This is the project that connects identity governance to the applications people actually use
day to day, and it's designed to be built last because it deliberately reuses everything before
it: Conditional Access from Project 1 gets enforced per-app, and the access packages from
Project 4 are what make an app's roles requestable and reviewable rather than assigned by hand.
The plan is to onboard three deliberately different apps — a SAML gallery app (the 80% case
most companies run), a custom OIDC app registration with SCIM 2.0 provisioning (the modern
auth-plus-automated-provisioning case), and a legacy-style header or password-SSO app (the
awkward case every real environment has at least one of).

The part of this project that actually separates a real build from a demo is deprovisioning,
not provisioning — SCIM makes it easy to create an account when someone's assigned, and it's
just as easy to leave that account orphaned when they're unassigned if the deprovisioning
behaviour isn't explicitly configured and tested. The design calls for proving that a leaver
processed through Project 3's offboarding workflow actually gets deprovisioned in a
SCIM-connected app as a downstream effect — which is the clearest possible demonstration that
these five projects are one system rather than five separate ones.

---

## What makes any project here count as evidence rather than a claim

A resume line is rejectable when it's just a sentence. It's unrejectable when it's backed by
something a hiring manager can actually open and ask about. Every project that's marked
complete or in-progress in this repo follows the same four-part standard:

1. **Working configuration** in a real tenant — not a diagram of what it would look like.
2. **Config as code** — every policy, workflow, and package exported to JSON via Microsoft
   Graph and committed to the repo. This one habit is what separates an engineer who can
   operate infrastructure from someone who can click through a portal once.
3. **Validation evidence** — screenshots of the control actually working, ideally alongside a
   test matrix showing what was checked and what the expected result was.
4. **A documented failure.** A project with no failure log in this repo is not finished, on
   purpose — a clean run proves nothing about whether the thing was actually operated. Every
   completed project here has at least one real mistake, misconfiguration, or unexpected
   platform behaviour captured honestly, because that's the part an interview question is
   actually going to probe.

---

## Repo map

```
SC-300-Portfolio/
├── README.md                              # this file
├── p1-conditional-access/
│   ├── README.md
│   ├── CA-Framework-Documentation.docx
│   ├── config/ca-policies.json
│   └── evidence screenshots
├── p2-pim-privileged-access/
│   ├── README.md  (pending)
│   └── evidence in progress
├── p3-identity-lifecycle/
│   ├── README.md  (pending)
│   └── evidence in progress
├── p4-access-governance/
│   ├── README.md  (pending)
│   └── SoD evidence set, in progress
└── p5-app-integration-sso-scim/
    └── not yet started
```

---

## Status

- [x] P1 — Conditional Access framework — **complete**
- [ ] P2 — PIM — eligible-assignment evidence captured, activation/review flow pending
- [ ] P3 — Identity lifecycle — one clean Leaver run captured, failure case and verification script pending
- [ ] P4 — Access governance — SoD preventive and detective evidence complete; campaigns, revocation proof, and exports pending
- [ ] P5 — App integration — not started, designed to be built last

This status list is deliberately honest rather than uniform. A portfolio where every box is
checked the same week it was started reads as less credible than one that shows real,
in-progress engineering — which is what identity governance work actually looks like.
