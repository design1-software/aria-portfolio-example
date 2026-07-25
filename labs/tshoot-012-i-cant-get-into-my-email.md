# I Can't Get Into My Email

**Identity · Troubleshoot · Completed 2026-07-25 · Instructor-validated**

---

## Overview

A branch clinician reports she cannot get into her email; Outlook keeps rejecting her password. "Email is down" is almost always an identity problem. The task: split identity from the client, read the account and sign-in state from Microsoft Graph, name the cause, and fix it in scope or escalate.

## Skills Demonstrated

Identity-vs-client isolation (OWA test), Microsoft Graph PowerShell (account state, sign-in logs, MFA posture), AADSTS error-code interpretation, scoped remediation as Authentication Administrator, dual-client verification, professional close-out documentation.

## Objective

Diagnose why the user cannot sign in, isolate whether the fault is identity-side or client-side, identify the root cause from Graph evidence, remediate within the Authentication Administrator scope, and verify the fix in both web and desktop clients.

## Environment

Microsoft Entra training tenant — cloud identity with Entra ID Free licensing (no mailbox; every fault is a sign-in fault).

## Approach

1. Clarified the vague symptom — scoped "email is down" to the specific user, device, and error behavior.
2. Isolated identity from client — signed in as the affected user at Outlook on the web from a clean browser. OWA failed too, proving the fault is identity-side, not client-side.
3. Read the account state with Microsoft Graph — checked AccountEnabled, captured the AADSTS sign-in failure code, verified license and MFA posture.
4. Named the root cause from the evidence — AccountEnabled=False, AADSTS50057 (account disabled), both clients blocked, nothing client-side involved.
5. Remediated in scope — as Authentication Administrator, re-enabled the account.
6. Verified in both clients — sign-in now succeeds in Outlook on the web and the desktop client.
7. Drafted the professional close-out note.

## Evidence Requirements Met

- [x] **clarify_symptom** — Vague "email is down" scoped to real symptom, device, user
- [x] **owa_isolation_test** — OWA tried from another device — the identity-vs-client split
- [x] **account_state** — AccountEnabled / block state read with Microsoft Graph
- [x] **signin_failure_code** — AADSTS sign-in failure code captured and interpreted
- [x] **license_and_mfa_check** — License and registered authentication methods verified
- [x] **client_side_checks** — Client-side checks — skipped (OWA failed, fault is identity)
- [x] **root_cause_named** — A single root cause named from the evidence
- [x] **fix_or_escalate** — Remediated in scope (account re-enabled)
- [x] **verification** — Sign-in confirmed in both web and desktop client
- [x] **final_note** — Professional close-out note

## Evidence & Results

### 1. clarify_symptom

User: branch clinician, <user-display-name>. Device: desktop Outlook client. Symptom: password rejected on every attempt. No recent password change by the user. Started overnight — worked yesterday afternoon.

### 2. owa_isolation_test

Opened a clean browser, navigated to https://outlook.office.com, entered the user's UPN. Result: **sign-in failed** — the same rejection. This proves the fault is **identity-side** (server rejects the authentication), not client-side. If OWA had succeeded, the investigation would pivot to the desktop client.

### 3. account_state

```
DisplayName       : <user-display-name>
UserPrincipalName : <training-upn>
AccountEnabled    : False
```

The account is disabled. This is the identity fault.

### 4. signin_failure_code

```
Status           : failure
ErrorCode        : 50057
FailureReason    : The user account is disabled.
```

AADSTS50057 — the tenant is actively rejecting authentication because the account is disabled. This matches the OWA isolation result and the AccountEnabled=False read.

### 5. license_and_mfa_check

```
AssignedLicenses : (none — Entra ID Free)
```

Authentication methods registered:

```
Id                                   : <method-id>
@odata.type                          : #microsoft.graph.passwordAuthenticationMethod
```

No MFA methods registered (expected for this training account configuration). License is Entra ID Free — no mailbox, which confirms every fault is a sign-in fault, not a mail-flow issue.

### 6. client_side_checks

Skipped — OWA failed at Step 2, proving the fault is identity-side. Client-side checks are only relevant when OWA succeeds but the desktop client does not.

### 7. root_cause_named

**Root cause: account disabled (AADSTS50057).** The account's AccountEnabled property is False. Both OWA and the desktop client are blocked because the tenant rejects authentication at the identity layer. No client-side component is involved.

### 8. fix_or_escalate

Remediated in scope as Authentication Administrator:

```
Update-MgUser -UserId <training-upn> -AccountEnabled:$true
```

Confirmed:

```
AccountEnabled : True
```

The account is now enabled. (If the fault had been outside Authentication Administrator scope — e.g. a Conditional Access policy block — the action would be escalation with the AADSTS code, not a guess.)

### 9. verification

**Outlook on the web:** sign-in succeeded. User reaches inbox (empty — Entra ID Free, no mailbox, but authentication completes).

**Desktop client:** sign-in succeeded. Outlook connects and shows the account as active.

Both clients now authenticate — the same evidence that proved the fault now proves the fix.

### 10. final_note

**Summary:** Branch clinician unable to sign in to email. Outlook desktop and OWA both rejecting credentials.

**Isolation result:** OWA sign-in failed from a clean browser — fault is identity-side, not client-side.

**Evidence:** AccountEnabled=False, AADSTS50057 ("account is disabled"), no MFA methods registered, Entra ID Free license (no mailbox).

**Root cause:** Account disabled at the identity layer.

**Action taken:** Re-enabled the account (Authentication Administrator scope). AccountEnabled now True.

**Verification:** Sign-in confirmed in both Outlook on the web and the desktop client.

**Escalation decision:** None required — remediation was within scope. If the account is disabled again, escalate to the identity owner to investigate the source of the disable action (admin action, automation, or compromise).

---

## Verification & Outcome (Approved Note)

Good isolation-first approach — she proved it was identity before touching anything. Graph evidence is clean: the AccountEnabled read, the AADSTS code, and the dual-client verification all line up. The close-out note follows the professional format. Approved.

*Reviewed and approved by an ARIA instructor on 2026-07-25.*
