# Cross-Domain Incident Response

**Identity + Endpoint + SOC · Capstone · Completed 2026-07-25 · Instructor-validated**

---

## Overview

Overnight, the SIEM raised alerts on a branch Linux server (patient-facing scheduling host) at a regional clinic network. Preliminary triage suggests the access originated from a staff Microsoft 365 account whose credential appears to have been compromised. This is one incident across three domains — identity, endpoint, and SOC — and the response must close all three.

## Skills Demonstrated

SIEM alert triage and timeline construction (Wazuh), Linux endpoint forensics (unauthorized accounts, persistence mechanisms, SUID backdoor hunting), Microsoft Graph identity investigation (account state, MFA posture, sign-in signal), cross-domain timeline correlation, containment-before-eradication across two domains, dual-domain remediation and verification, executive incident reporting for a non-technical audience.

## Objective

Investigate the SIEM alerts, trace the incident across identity and endpoint, build a correlated cross-domain timeline, contain and eradicate in both domains (identity first), verify both sides clean, and deliver a client-facing executive report.

## Environment

Wazuh SIEM (analyst read-only), assigned Linux endpoint (SSH), Microsoft Entra training tenant (Authentication Administrator, scoped to assigned training users).

## Approach

1. Anchored the timeline from the SOC — read the Wazuh dashboard for the affected host to identify which rules fired, at what level, and when. FIM alerts on /etc changes and rootcheck findings established the *what* and *when*.
2. Investigated the endpoint — followed the alerts to the host, identified the unauthorized foothold account, traced persistence mechanisms (cron beacon, attacker SSH key), and hunted for the SUID backdoor that the real-time alerts didn't catch.
3. Established the identity compromise — read the associated account via Graph: account state, MFA registration (stripped), and sign-in signal. Confirmed the attacker's identity action with no secrets in evidence.
4. Correlated all three domains — built a single timeline aligning the identity event, the endpoint foothold and persistence, and the Wazuh detections with real timestamps.
5. Contained the identity first — disabled the compromised account so the attacker cannot re-enter during endpoint cleanup.
6. Eradicated the endpoint — removed the foothold user, cron persistence, attacker key, and SUID backdoor. Verified all artifacts gone.
7. Completed identity eradication — reset the credential and initiated MFA re-registration. Verified the account is clean and functional.
8. Drafted the executive report for the client decision-maker.

## Evidence Requirements Met

- [x] **soc_alert_timeline** — Wazuh alerts for the host (rule IDs/levels + timestamps)
- [x] **endpoint_foothold** — The unauthorized account, tied to the alerts
- [x] **persistence_found** — Persistence mechanisms and the SUID backdoor
- [x] **identity_compromise** — Graph evidence: account state, sign-in signal, MFA change
- [x] **cross_domain_timeline** — One timeline aligning identity + endpoint + SOC
- [x] **endpoint_remediation** — Foothold/persistence/backdoor removed, before/after
- [x] **identity_remediation** — Account contained + credential/MFA reset, verified
- [x] **executive_report** — Client-facing impact / root cause / actions / prevention

## Evidence & Results

### 1. soc_alert_timeline

Wazuh dashboard — alerts for <lab-host> (agent <agent-id>):

```
Timestamp            Rule ID   Level   Description
2026-07-25 02:14:07  550       7       Integrity checksum changed: /etc/passwd
2026-07-25 02:14:07  554       5       File added to the system: /etc/cron.d/<hidden-filename>
2026-07-25 02:17:22  510       7       Rootcheck: SUID file found in /tmp
```

The FIM alerts on /etc (new cron file, passwd modification from the new user) anchor the timeline. The rootcheck finding flags the SUID binary. These tell me the host was touched and approximately when — they do not reveal the credential story or the full extent of persistence.

### 2. endpoint_foothold

```
$ getent passwd | awk -F: '$3>=1000'
<foothold-user>:x:1337:1337::/home/<foothold-user>:/bin/bash
```

An unauthorized local account (UID 1337) — not present in any baseline. This is the foothold the attacker created after gaining access via the compromised credential. The UID and creation time align with the Wazuh FIM alert on /etc/passwd.

### 3. persistence_found

**Persistence A — cron beacon:**
```
$ cat /etc/cron.d/<hidden-filename>
*/15 * * * * <foothold-user> /usr/bin/curl -s http://<redacted-c2>/beacon >/dev/null 2>&1
```

**Persistence B — attacker SSH key:**
```
$ cat /home/<foothold-user>/.ssh/authorized_keys
ssh-rsa AAAA<redacted>... attacker@<redacted>
```

**Backdoor — SUID binary:**
```
$ find / -xdev -perm -4000 -type f 2>/dev/null | grep tmp
/tmp/<hidden-binary>

$ file /tmp/<hidden-binary>
/tmp/<hidden-binary>: setuid ELF 64-bit LSB executable, x86-64
```

A SUID copy of bash in /tmp — privilege escalation backdoor. This was not caught by Wazuh real-time alerting (the /tmp path isn't in the FIM configuration) — found by doing the manual endpoint audit that the initial alerts pointed toward.

### 4. identity_compromise

```
DisplayName       : <user-display-name>
UserPrincipalName : <training-upn>
AccountEnabled    : True
```

Authentication methods:
```
@odata.type: #microsoft.graph.passwordAuthenticationMethod
```

MFA methods: **none registered** — the attacker stripped MFA from the account (the clear-mfa fault). With MFA removed, the compromised password alone is sufficient for access.

Sign-in signal:
```
CreatedDateTime : 2026-07-25T02:12:41Z
Status          : Success
AppDisplayName  : Microsoft Office
IPAddress       : <redacted-ip>
```

A successful sign-in from an unfamiliar IP approximately 2 minutes before the first endpoint alert — consistent with credential compromise followed by lateral movement to the Linux host.

### 5. cross_domain_timeline

| Time (UTC) | Domain | Event |
|---|---|---|
| ~02:10 | Identity | MFA stripped from the account (attacker preparation) |
| 02:12:41 | Identity | Successful sign-in from unfamiliar IP |
| 02:13–02:14 | Endpoint | Foothold user created (UID 1337), cron persistence written, SSH key planted |
| 02:14:07 | SOC | Wazuh FIM: /etc/passwd changed, /etc/cron.d/<hidden-filename> added |
| 02:14–02:17 | Endpoint | SUID backdoor placed in /tmp |
| 02:17:22 | SOC | Wazuh rootcheck: SUID file in /tmp |

**One incident:** credential compromise (identity) → foothold + persistence (endpoint) → detection (SOC). The sign-in timestamp precedes the first host modification by under two minutes. This is not three separate events — it is one attack chain.

### 6. endpoint_remediation

**Containment note:** identity was contained first (Step 7 below) before endpoint eradication began — the attacker cannot re-enter via the compromised credential during cleanup.

Eradication:
```
$ userdel -r <foothold-user>               # remove foothold user + home directory
$ rm /etc/cron.d/<hidden-filename>          # remove cron persistence
$ rm /tmp/<hidden-binary>                   # remove SUID backdoor
```

Verification (after):
```
$ getent passwd | grep <foothold-user>
(no output — user removed)

$ ls /etc/cron.d/<hidden-filename> 2>&1
ls: cannot access '/etc/cron.d/<hidden-filename>': No such file or directory

$ find / -xdev -perm -4000 -type f 2>/dev/null | grep tmp
(no output — SUID backdoor removed)
```

All three artifacts confirmed removed: foothold user, cron persistence, and SUID backdoor.

### 7. identity_remediation

**Containment (before endpoint eradication):**
```
Update-MgUser -UserId <training-upn> -AccountEnabled:$false
```

The account is disabled immediately — the attacker loses the credential path while the endpoint is cleaned.

**Eradication (after endpoint is clean):**
```
Update-MgUser -UserId <training-upn> -AccountEnabled:$true
Reset-MgUserAuthenticationMethodPassword -UserId <training-upn> ...
```

User directed to re-register MFA at the security-info portal.

**Verification:**
```
DisplayName       : <user-display-name>
AccountEnabled    : True
```

Authentication methods now include a newly registered MFA method. Sign-in succeeds with the new credential + MFA.

### 8. executive_report

**To:** Regional Clinic Network — Executive Leadership
**From:** Incident Response
**Date:** 2026-07-25
**Subject:** Security Incident — Branch Linux Server / Compromised Staff Credential

**Impact:** A staff Microsoft 365 credential was compromised and used to access a branch Linux server (patient-facing scheduling host). The attacker created an unauthorized account, installed persistence mechanisms, and planted a privilege-escalation backdoor. No evidence of data exfiltration was found, but the attacker had the access to attempt it.

**Root Cause:** The staff account's multi-factor authentication was removed (by the attacker or by a misconfiguration that allowed it), leaving the account protected only by its password. The compromised password was used to authenticate and pivot to the Linux host.

**Timeline:** MFA removed from the account → successful sign-in from an unfamiliar IP (02:12 UTC) → foothold user + persistence created on the host (02:13–02:14) → SUID backdoor placed (02:14–02:17). Total attacker dwell time before detection: approximately 5 minutes.

**Actions Taken:**
1. Contained the identity — disabled the compromised account to prevent re-entry.
2. Eradicated the endpoint — removed the unauthorized user, cron persistence, attacker SSH key, and SUID backdoor.
3. Restored the identity — re-enabled the account with a new password and re-registered MFA.
4. Verified both sides — confirmed all artifacts removed from the host; confirmed the account authenticates only with the new credential + MFA.

**Prevention Recommendations:**
1. Enforce MFA registration as a Conditional Access requirement — removal should trigger an alert and automatic account disable.
2. Monitor for new local accounts on Linux hosts (UID > 1000) via Wazuh FIM on /etc/passwd.
3. Add /tmp to the Wazuh syscheck configuration for SUID monitoring.
4. Require SSH key rotation and restrict authorized_keys to managed keys only.

---

## Verification & Outcome (Approved Note)

This is the capstone — and she ran it like a real incident. The timeline correlation is the strongest part: the identity sign-in, the endpoint artifacts, and the Wazuh detections all line up within minutes, and she built that story explicitly instead of treating them as three separate findings. Containment order was correct — identity first, then endpoint, not the other way around. Both sides verified clean. The executive report is client-ready: no command logs, clear impact statement, actionable prevention. Approved as the graduation capstone.

*Reviewed and approved by an ARIA instructor on 2026-07-25.*
