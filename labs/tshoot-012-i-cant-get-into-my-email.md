[← Back to portfolio](../README.md)

# I Can't Get Into My Email

[![Domain](https://img.shields.io/badge/Domain-Identity-1f6feb?style=flat-square)](../README.md#skills-matrix)
[![Type](https://img.shields.io/badge/Type-Troubleshoot-8250df?style=flat-square)](../README.md#the-work)
[![Evidence](https://img.shields.io/badge/Evidence-10%2F10%20met-1a7f37?style=flat-square)](#evidence)
[![Status](https://img.shields.io/badge/Status-instructor%20approved-1a7f37?style=flat-square)](#verification--outcome)

|  |  |
|---|---|
| **Lab ID** | `tshoot-012` |
| **Completed** | 2026-07-25 |
| **Environment** | Microsoft Entra training tenant — cloud identity, Entra ID Free licensing (no mailbox; every fault is a sign-in fault) |
| **Skills** | `identity-vs-client isolation` `Microsoft Graph PowerShell` `AADSTS interpretation` `MFA posture review` `scoped remediation` `close-out documentation` |

---

## The situation

A branch clinician reports she cannot get into her email; Outlook keeps rejecting her password.

**"Email is down" is almost always an identity problem.** The task: split identity from the client,
read the account and sign-in state from Microsoft Graph, name the cause, and fix it in scope or
escalate.

> [!NOTE]
> **Objective** — Diagnose why the user cannot sign in, isolate whether the fault is identity-side or
> client-side, identify the root cause from Graph evidence, remediate within the Authentication
> Administrator scope, and verify the fix in both web and desktop clients.

## Approach

```
1 · Clarify       Scope "email is down" to a specific user, device, and error behavior
2 · Isolate       Sign in as the user at OWA from a clean browser — identity or client?
3 · Read state    Graph: AccountEnabled, AADSTS failure code, license and MFA posture
4 · Name cause    AccountEnabled=False + AADSTS50057 + both clients blocked
5 · Remediate     Re-enable the account as Authentication Administrator
6 · Verify        Sign-in succeeds in OWA *and* the desktop client
7 · Document      Professional close-out note
```

> [!IMPORTANT]
> **The pivot point is step 2.** OWA failing too means the server is rejecting authentication —
> nothing on the desktop client can explain it, so no time is spent there.

---

## Evidence

| # | Requirement | What it proves | |
|---|---|---|---|
| 1 | `clarify_symptom` | Vague "email is down" scoped to real symptom, device, user | ✅ |
| 2 | `owa_isolation_test` | OWA tried from another device — the identity-vs-client split | ✅ |
| 3 | `account_state` | AccountEnabled / block state read with Microsoft Graph | ✅ |
| 4 | `signin_failure_code` | AADSTS sign-in failure code captured and interpreted | ✅ |
| 5 | `license_and_mfa_check` | License and registered authentication methods verified | ✅ |
| 6 | `client_side_checks` | Correctly skipped — OWA failed, so the fault is identity | ✅ |
| 7 | `root_cause_named` | A single root cause named from the evidence | ✅ |
| 8 | `fix_or_escalate` | Remediated in scope (account re-enabled) | ✅ |
| 9 | `verification` | Sign-in confirmed in both web and desktop client | ✅ |
| 10 | `final_note` | Professional close-out note | ✅ |

### 1 · Clarify the symptom

| | |
|---|---|
| **User** | Branch clinician, `<user-display-name>` |
| **Device** | Desktop Outlook client |
| **Symptom** | Password rejected on every attempt |
| **Recent changes** | No password change by the user |
| **Onset** | Overnight — worked yesterday afternoon |

### 2 · OWA isolation test

Opened a clean browser, navigated to `https://outlook.office.com`, entered the user's UPN.

**Result: sign-in failed** — the same rejection.

> **Reading it:** the fault is **identity-side** — the server itself rejects the authentication. Had
> OWA succeeded, the investigation would have pivoted to the desktop client instead.

### 3 · Account state

```console
DisplayName       : <user-display-name>
UserPrincipalName : <training-upn>
AccountEnabled    : False
```

> **Reading it:** the account is disabled. This is the identity fault.

### 4 · Sign-in failure code

```console
Status           : failure
ErrorCode        : 50057
FailureReason    : The user account is disabled.
```

> **Reading it:** AADSTS50057 — the tenant is actively rejecting authentication because the account
> is disabled. Three independent signals now agree: the OWA isolation result, the `AccountEnabled`
> read, and the sign-in log.

### 5 · License and MFA check

```console
AssignedLicenses : (none — Entra ID Free)
```

Registered authentication methods:

```console
Id                                   : <method-id>
@odata.type                          : #microsoft.graph.passwordAuthenticationMethod
```

> **Reading it:** no MFA methods registered (expected for this training account configuration).
> License is Entra ID Free — no mailbox, which confirms every fault here is a sign-in fault, not a
> mail-flow issue.

### 6 · Client-side checks

**Skipped — deliberately.** OWA failed at step 2, proving the fault is identity-side. Client-side
checks are only relevant when OWA succeeds but the desktop client does not.

### 7 · Root cause

> [!CAUTION]
> **Root cause: account disabled (AADSTS50057).**
>
> The account's `AccountEnabled` property is `False`. Both OWA and the desktop client are blocked
> because the tenant rejects authentication at the identity layer. No client-side component is
> involved.

### 8 · Fix or escalate

Remediated in scope as **Authentication Administrator**:

```powershell
Update-MgUser -UserId <training-upn> -AccountEnabled:$true
```

Confirmed:

```console
AccountEnabled : True
```

> **Scope discipline:** had the fault been outside Authentication Administrator scope — a Conditional
> Access policy block, for example — the correct action would be escalation *with the AADSTS code*,
> not a guess.

### 9 · Verification

| Client | Result |
|---|---|
| **Outlook on the web** | ✅ Sign-in succeeded — user reaches inbox (empty; Entra ID Free, no mailbox, but authentication completes) |
| **Desktop client** | ✅ Sign-in succeeded — Outlook connects and shows the account as active |

> **Reading it:** the same evidence that proved the fault now proves the fix.

### 10 · Final note

| | |
|---|---|
| **Summary** | Branch clinician unable to sign in to email. Outlook desktop and OWA both rejecting credentials. |
| **Isolation result** | OWA sign-in failed from a clean browser — fault is identity-side, not client-side. |
| **Evidence** | `AccountEnabled=False`, AADSTS50057 ("account is disabled"), no MFA methods registered, Entra ID Free license (no mailbox). |
| **Root cause** | Account disabled at the identity layer. |
| **Action taken** | Re-enabled the account (Authentication Administrator scope). `AccountEnabled` now `True`. |
| **Verification** | Sign-in confirmed in both Outlook on the web and the desktop client. |
| **Escalation decision** | None required — remediation was within scope. If the account is disabled again, escalate to the identity owner to investigate the source of the disable action (admin action, automation, or compromise). |

---

## Verification & outcome

> [!TIP]
> **Approved note — reviewing instructor**
>
> Good isolation-first approach — she proved it was identity before touching anything. Graph
> evidence is clean: the `AccountEnabled` read, the AADSTS code, and the dual-client verification all
> line up. The close-out note follows the professional format. **Approved.**

<sub>Reviewed and approved by an ARIA instructor on 2026-07-25.</sub>

---

<sub>[← Back to portfolio](../README.md)</sub>
