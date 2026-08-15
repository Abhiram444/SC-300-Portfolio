# Separation of Duties — Exception Record Policy & Template

**Project 4 — Access Governance** **Purpose:** define how a Separation of Duties conflict may be temporarily permitted, and what makes that permission legitimate rather than a quiet weakening of the control.

---

## Why exceptions need a formal record

A Separation of Duties rule exists because holding both sides of a conflicting pair creates fraud or error risk — for example, one person able to both create a vendor payment and approve it, with no second set of eyes.

Real business situations sometimes require breaking that rule anyway: someone covering two roles during a colleague's medical leave, a two-person team where the segregation is mathematically impossible, a month-end crunch where the approver is unreachable. Refusing every exception is not a workable governance policy — people will simply route around the control, and the organisation loses visibility entirely.

The governance principle is therefore not *"never allow an exception."* It is:

> **An exception without an expiry date is not an exception. It is a permanent policy change wearing a temporary label.**

Every exception below must carry a hard expiry, a named approver, and a compensating control. If it cannot satisfy all three, it should be refused rather than granted informally.

---

## Required fields

Every SoD exception must record all of the following. An exception missing any field is not valid and must not be granted.

| Field | Requirement | Why it matters |
| :---- | :---- | :---- |
| **Exception ID** | Unique reference (e.g. `SOD-EX-2026-001`) | Makes the exception traceable in audit and reportable as a population |
| **Identity** | UPN of the person receiving the conflicting access | Who exactly this applies to — never a group or a role, always a named person |
| **Conflicting access** | Both sides of the conflict, named explicitly (e.g. `Finance - AP Analyst` \+ `Finance - AP Approver`) | Prevents vague "finance access" exceptions that quietly expand in scope |
| **Business justification** | Free text, must state the operational reason | An exception with no stated reason cannot be re-evaluated later |
| **Requested by** | Who asked for it | Separates the requester from the approver |
| **Approved by** | Named approver, must not be the requester | Self-approved exceptions defeat the entire purpose of the control |
| **Approval date** | Date granted | Starts the clock |
| **Expiry date** | **Mandatory. No exception may be open-ended.** | The single most important field on this record |
| **Compensating control** | What additional oversight applies while the conflict exists | The risk doesn't disappear because it was approved — something must offset it |
| **Review owner** | Who must confirm the exception was actually revoked at expiry | An expiry nobody checks is not an expiry |
| **Status** | Active / Expired / Revoked early | Enables reporting on the current exception population |

---

## Compensating controls — acceptable options

An exception must pair with at least one of these. "We'll keep an eye on it" is not a compensating control.

- **Enhanced transaction monitoring** — all transactions where the individual acted on both sides of the conflict are logged and reviewed by a named third party on a defined cadence.  
- **Mandatory secondary sign-off** — a second approver outside the conflicted pair must co-sign any transaction above a defined value threshold.  
- **Reduced scope** — the conflicting access is granted at a narrower scope than normal (e.g. a single cost centre rather than the whole department) for the duration.  
- **Shortened expiry** — the exception is granted for days rather than months, forcing re-justification quickly rather than drifting.  
- **Post-hoc detective review** — the detective SoD check (`sod-detect`) is run at the end of the exception window specifically to confirm the conflict was cleared.

---

## Lifecycle

1. **Request** — requester submits the exception with business justification and proposed expiry. Open-ended requests are rejected at intake.  
2. **Risk assessment** — the reviewing party confirms the conflict is genuine, assesses impact, and selects a compensating control proportional to that impact.  
3. **Approval** — a named approver who is not the requester approves, or refuses. The approval is recorded against the Exception ID.  
4. **Implementation** — the conflicting access is granted. In Microsoft Entra this means the SoD incompatibility rule must be temporarily bypassed, which is itself a logged administrative action — that log is part of the exception record.  
5. **Monitoring** — the compensating control runs for the full duration of the exception.  
6. **Expiry** — the review owner confirms the conflicting access was actually removed. **This step must be verified against the resource, not against the exception tracker** — the same principle proven in this project's revocation testing: a record saying access ended is not the same as access having ended.  
7. **Closure** — status set to Expired (or Revoked early), with evidence of removal attached.

---

## Exception record template

EXCEPTION ID:          SOD-EX-YYYY-NNN

STATUS:                Active / Expired / Revoked early

IDENTITY:              firstname.lastname@domain

CONFLICTING ACCESS:    \[Package A\] \+ \[Package B\]

CONFLICT TYPE:         Separation of Duties — incompatible access packages

BUSINESS JUSTIFICATION:

\[Why this exception is operationally necessary. Must describe the specific

 situation, not a general convenience.\]

REQUESTED BY:          \[Name, role\]

APPROVED BY:           \[Name, role — must differ from requester\]

APPROVAL DATE:         YYYY-MM-DD

EXPIRY DATE:           YYYY-MM-DD   ← MANDATORY, no open-ended exceptions

COMPENSATING CONTROL:

\[Which control from the approved list applies, and who operates it\]

REVIEW OWNER:          \[Name — responsible for confirming removal at expiry\]

CLOSURE EVIDENCE:

\[Attached at expiry: proof the conflicting access was actually removed from

 the resource, verified directly rather than assumed from the tracker\]

---

## Worked example

EXCEPTION ID:          SOD-EX-2026-001

STATUS:                Active

IDENTITY:              helma.hassel@rabhi060gmail.onmicrosoft.com

CONFLICTING ACCESS:    Finance \- AP Analyst \+ Finance \- AP Approver

CONFLICT TYPE:         Separation of Duties — incompatible access packages

BUSINESS JUSTIFICATION:

The Finance AP Approver is on unplanned medical leave with no trained backup.

Month-end close requires approvals to continue. Helma is the only other person

trained on the AP approval process. Without this exception, month-end close

cannot complete on schedule.

REQUESTED BY:          Finance Manager

APPROVED BY:           Head of Internal Controls

APPROVAL DATE:         2026-08-15

EXPIRY DATE:           2026-08-29   (14 days — covers month-end close only)

COMPENSATING CONTROL:

Mandatory secondary sign-off: any AP transaction Helma both creates and approves

requires co-signature from the Head of Internal Controls before payment release.

All such transactions are logged and reviewed weekly.

REVIEW OWNER:          Head of Internal Controls

CLOSURE EVIDENCE:

\[To be attached 2026-08-29: screenshot of Finance \- AP Approver assignments

 confirming Helma is no longer listed, plus sod-detect output showing zero

 remaining conflicts\]

---

## Reporting requirement

The current exception population must be reportable at any time, answering:

- How many SoD exceptions are currently active?  
- How many have passed their expiry date without confirmed closure? **(This number should always be zero. If it isn't, the exception process has failed.)**  
- Which identities hold more than one active exception? (A pattern here suggests the underlying role design is wrong, not that the person needs more exceptions.)  
- What is the average exception duration, and is it trending upward? (Rising duration means exceptions are becoming permanent by stealth.)

---

## Connection to this project's findings

This policy is written directly against something proven in Project 4's revocation testing: a recorded governance decision and an actual change to access are two different things. An access review recorded a Deny decision with status "Success" in the audit log, while the underlying role assignment remained active and permanent, because the review's results had not been applied to the resource.

That is why step 6 of the lifecycle above requires verification **against the resource** rather than against the exception tracker. An exception process that trusts its own record-keeping inherits exactly the same failure mode.  
