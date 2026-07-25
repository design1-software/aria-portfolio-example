[← Back to portfolio](../README.md)

# Cross-Domain Incident Response

[![Domain](https://img.shields.io/badge/Domain-Identity%20%2B%20Endpoint%20%2B%20SOC-1f6feb?style=flat-square)](../README.md#skills-matrix)
[![Type](https://img.shields.io/badge/Type-Capstone-d1242f?style=flat-square)](../README.md#the-work)
[![Evidence](https://img.shields.io/badge/Evidence-8%2F8%20met-1a7f37?style=flat-square)](#evidence)
[![Status](https://img.shields.io/badge/Status-instructor%20approved-1a7f37?style=flat-square)](#verification--outcome)

|  |  |
|---|---|
| **Lab ID** | `capstone-001` |
| **Completed** | 2026-07-25 |
| **Environment** | Wazuh SIEM (analyst read-only) · assigned Linux endpoint (SSH) · Microsoft Entra training tenant (Authentication Administrator, scoped to assigned training users) |
| **Skills** | `SIEM triage` `timeline construction` `Linux forensics` `persistence hunting` `SUID backdoor detection` `Microsoft Graph investigation` `cross-domain correlation` `containment sequencing` `executive reporting` |

---

## The situation

Overnight, the SIEM raised alerts on a branch Linux server — a patient-facing scheduling host at a
regional clinic network. Preliminary triage suggests the access originated from a staff Microsoft 365
account whose credential appears to have been compromised.

> [!WARNING]
> This is **one incident across three domains** — identity, endpoint, and SOC. The response has to
> close all three, in the right order.

> [!NOTE]
> **Objective** — Investigate the SIEM alerts, trace the incident across identity and endpoint, build
> a correlated cross-domain timeline, contain and eradicate in both domains (identity first), verify
> both sides clean, and deliver a client-facing executive report.

## Approach

```
1 · Anchor        Wazuh dashboard for the affected host — which rules fired, what level, when
2 · Endpoint      Follow the alerts to the host: foothold account, cron beacon, attacker SSH key,
                  and the SUID backdoor the real-time alerts missed
3 · Identity      Graph: account state, MFA registration (stripped), sign-in signal
4 · Correlate     One timeline aligning identity + endpoint + SOC on real timestamps
5 · Contain       Disable the compromised account FIRST — no re-entry during cleanup
6 · Eradicate     Remove foothold user, cron persistence, attacker key, SUID backdoor — verify gone
7 · Restore       Reset the credential, re-register MFA, verify the account clean and functional
8 · Report        Executive report for the client decision-maker
```

> [!IMPORTANT]
> **Containment order is the whole discipline here.** Cleaning the endpoint while the attacker still
> holds a working credential just gives them a host to re-take. Identity closes first.

---

## Evidence

| # | Requirement | What it proves | |
|---|---|---|---|
| 1 | `soc_alert_timeline` | Wazuh alerts for the host — rule IDs, levels, timestamps | ✅ |
| 2 | `endpoint_foothold` | The unauthorized account, tied to the alerts | ✅ |
| 3 | `persistence_found` | Persistence mechanisms and the SUID backdoor | ✅ |
| 4 | `identity_compromise` | Graph evidence: account state, sign-in signal, MFA change | ✅ |
| 5 | `cross_domain_timeline` | One timeline aligning identity + endpoint + SOC | ✅ |
| 6 | `endpoint_remediation` | Foothold / persistence / backdoor removed, before and after | ✅ |
| 7 | `identity_remediation` | Account contained + credential/MFA reset, verified | ✅ |
| 8 | `executive_report` | Client-facing impact / root cause / actions / prevention | ✅ |

### 1 · SOC alert timeline

Wazuh dashboard — alerts for `<lab-host>` (agent `<agent-id>`):

```console
Timestamp            Rule ID   Level   Description
2026-07-25 02:14:07  550       7       Integrity checksum changed: /etc/passwd
2026-07-25 02:14:07  554       5       File added to the system: /etc/cron.d/<hidden-filename>
2026-07-25 02:17:22  510       7       Rootcheck: SUID file found in /tmp
```

> **Reading it:** the FIM alerts on `/etc` anchor the timeline — a new cron file and a `passwd`
> modification. Rootcheck flags the SUID binary. These establish *the host was touched* and *roughly
> when* — they say nothing about the credential story or the full extent of persistence.

### 2 · Endpoint foothold

```console
$ getent passwd | awk -F: '$3>=1000'
<foothold-user>:x:1337:1337::/home/<foothold-user>:/bin/bash
```

> **Reading it:** an unauthorized local account (UID 1337) — not present in any baseline. This is the
> foothold created *after* access via the compromised credential. Its UID and creation time align
> with the Wazuh FIM alert on `/etc/passwd`.

### 3 · Persistence and backdoor

**Persistence A — cron beacon**

```console
$ cat /etc/cron.d/<hidden-filename>
*/15 * * * * <foothold-user> /usr/bin/curl -s http://<redacted-c2>/beacon >/dev/null 2>&1
```

**Persistence B — attacker SSH key**

```console
$ cat /home/<foothold-user>/.ssh/authorized_keys
ssh-rsa AAAA<redacted>... attacker@<redacted>
```

**Backdoor — SUID binary**

```console
$ find / -xdev -perm -4000 -type f 2>/dev/null | grep tmp
/tmp/<hidden-binary>

$ file /tmp/<hidden-binary>
/tmp/<hidden-binary>: setuid ELF 64-bit LSB executable, x86-64
```

> [!WARNING]
> **The SUID backdoor was never alerted on.** `/tmp` isn't in the FIM configuration — a SUID copy of
> bash sitting there is a privilege-escalation path that real-time alerting would have missed
> entirely. It was found by doing the manual endpoint audit the initial alerts pointed toward.
> *Alerts are where you start, not where you stop.*

### 4 · Identity compromise

```console
DisplayName       : <user-display-name>
UserPrincipalName : <training-upn>
AccountEnabled    : True
```

Authentication methods:

```console
@odata.type: #microsoft.graph.passwordAuthenticationMethod
```

> **Reading it:** **no MFA methods registered** — the attacker stripped MFA from the account. With
> MFA removed, the compromised password alone is sufficient for access.

Sign-in signal:

```console
CreatedDateTime : 2026-07-25T02:12:41Z
Status          : Success
AppDisplayName  : Microsoft Office
IPAddress       : <redacted-ip>
```

> **Reading it:** a successful sign-in from an unfamiliar IP roughly **two minutes before the first
> endpoint alert** — consistent with credential compromise followed by lateral movement to the Linux
> host.

### 5 · Cross-domain timeline

| Time (UTC) | Domain | Event |
|---|---|---|
| ~02:10 | 🔐 Identity | MFA stripped from the account (attacker preparation) |
| 02:12:41 | 🔐 Identity | Successful sign-in from unfamiliar IP |
| 02:13–02:14 | 💻 Endpoint | Foothold user created (UID 1337), cron persistence written, SSH key planted |
| 02:14:07 | 🚨 SOC | Wazuh FIM: `/etc/passwd` changed, `/etc/cron.d/<hidden-filename>` added |
| 02:14–02:17 | 💻 Endpoint | SUID backdoor placed in `/tmp` |
| 02:17:22 | 🚨 SOC | Wazuh rootcheck: SUID file in `/tmp` |

> [!CAUTION]
> **One incident, not three findings.**
> Credential compromise *(identity)* → foothold + persistence *(endpoint)* → detection *(SOC)*.
> The sign-in timestamp precedes the first host modification by under two minutes. This is a single
> attack chain.

### 6 · Endpoint remediation

> [!IMPORTANT]
> **Containment note:** identity was contained first (step 7 below) *before* endpoint eradication
> began — the attacker cannot re-enter via the compromised credential during cleanup.

Eradication:

```console
$ userdel -r <foothold-user>               # remove foothold user + home directory
$ rm /etc/cron.d/<hidden-filename>         # remove cron persistence
$ rm /tmp/<hidden-binary>                  # remove SUID backdoor
```

Verification:

```console
$ getent passwd | grep <foothold-user>
(no output — user removed)

$ ls /etc/cron.d/<hidden-filename> 2>&1
ls: cannot access '/etc/cron.d/<hidden-filename>': No such file or directory

$ find / -xdev -perm -4000 -type f 2>/dev/null | grep tmp
(no output — SUID backdoor removed)
```

> **Reading it:** all three artifacts confirmed removed — foothold user, cron persistence, SUID
> backdoor. Each removal is proven by the same command that originally found it.

### 7 · Identity remediation

**Containment** — before endpoint eradication:

```powershell
Update-MgUser -UserId <training-upn> -AccountEnabled:$false
```

> The account is disabled immediately — the attacker loses the credential path while the endpoint is
> cleaned.

**Eradication** — after the endpoint is clean:

```powershell
Update-MgUser -UserId <training-upn> -AccountEnabled:$true
Reset-MgUserAuthenticationMethodPassword -UserId <training-upn> ...
```

User directed to re-register MFA at the security-info portal.

**Verification:**

```console
DisplayName       : <user-display-name>
AccountEnabled    : True
```

> **Reading it:** authentication methods now include a newly registered MFA method. Sign-in succeeds
> with the new credential **and** MFA — the stripped-MFA condition that enabled the compromise is
> closed.

### 8 · Executive report

> **To:** Regional Clinic Network — Executive Leadership
> **From:** Incident Response
> **Date:** 2026-07-25
> **Subject:** Security Incident — Branch Linux Server / Compromised Staff Credential

**Impact**

A staff Microsoft 365 credential was compromised and used to access a branch Linux server
(patient-facing scheduling host). The attacker created an unauthorized account, installed persistence
mechanisms, and planted a privilege-escalation backdoor. No evidence of data exfiltration was found,
but the attacker had the access to attempt it.

**Root cause**

The staff account's multi-factor authentication was removed — by the attacker, or by a
misconfiguration that allowed it — leaving the account protected only by its password. The
compromised password was used to authenticate and pivot to the Linux host.

**Timeline**

MFA removed → successful sign-in from an unfamiliar IP (02:12 UTC) → foothold user and persistence
created on the host (02:13–02:14) → SUID backdoor placed (02:14–02:17). **Total attacker dwell time
before detection: approximately 5 minutes.**

**Actions taken**

| | Action |
|---|---|
| 1 | **Contained the identity** — disabled the compromised account to prevent re-entry. |
| 2 | **Eradicated the endpoint** — removed the unauthorized user, cron persistence, attacker SSH key, and SUID backdoor. |
| 3 | **Restored the identity** — re-enabled the account with a new password and re-registered MFA. |
| 4 | **Verified both sides** — confirmed all artifacts removed from the host; confirmed the account authenticates only with the new credential + MFA. |

**Prevention recommendations**

| Priority | Recommendation |
|---|---|
| High | Enforce MFA registration as a Conditional Access requirement — removal should trigger an alert and automatic account disable. |
| High | Add `/tmp` to the Wazuh syscheck configuration for SUID monitoring — this incident's backdoor went undetected there. |
| Medium | Monitor for new local accounts on Linux hosts (UID > 1000) via Wazuh FIM on `/etc/passwd`. |
| Medium | Require SSH key rotation and restrict `authorized_keys` to managed keys only. |

---

## Verification & outcome

> [!TIP]
> **Approved note — reviewing instructor**
>
> This is the capstone — and she ran it like a real incident. The timeline correlation is the
> strongest part: the identity sign-in, the endpoint artifacts, and the Wazuh detections all line up
> within minutes, and she built that story explicitly instead of treating them as three separate
> findings. Containment order was correct — identity first, then endpoint, not the other way around.
> Both sides verified clean. The executive report is client-ready: no command logs, clear impact
> statement, actionable prevention. **Approved as the graduation capstone.**

<sub>Reviewed and approved by an ARIA instructor on 2026-07-25.</sub>

---

<sub>[← Back to portfolio](../README.md)</sub>
