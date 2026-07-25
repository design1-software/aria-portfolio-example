[← Back to portfolio](../README.md)

# ACL Basics and Safe Traffic Filtering

[![Domain](https://img.shields.io/badge/Domain-Networking-1f6feb?style=flat-square)](../README.md#skills-matrix)
[![Type](https://img.shields.io/badge/Type-Build-8250df?style=flat-square)](../README.md#the-work)
[![Evidence](https://img.shields.io/badge/Evidence-5%2F5%20met-1a7f37?style=flat-square)](#evidence)
[![Status](https://img.shields.io/badge/Status-instructor%20approved-1a7f37?style=flat-square)](#verification--outcome)

|  |  |
|---|---|
| **Lab ID** | `net-build-005` |
| **Completed** | 2026-07-24 |
| **Environment** | EVE-NG routed lab — two Cisco routers, /30 interlinks, loopback addresses |
| **Skills** | `ACL design` `extended ACL syntax` `interface-direction binding` `baseline-before-change` `rollback planning` `Cisco IOS` |

---

## The situation

A small office wants to allow ICMP testing between two lab networks but block simulated Telnet
traffic to a protected loopback.

**The task:** apply *only* the scoped ACL — and prove the result.

> [!NOTE]
> **Objective** — Build an extended ACL that denies Telnet to the protected loopback, permits ICMP,
> and permits everything else. Apply it inbound on the correct interface, and verify allowed traffic
> still flows.

## Approach

```
1 · Baseline      Ping between loopbacks both directions — capture the "before" snapshot
2 · Design        deny Telnet to protected loopback → permit ICMP → permit all other traffic
3 · Apply         Bind inbound on the correct interface of the upstream router
4 · Capture       Access-list contents + interface binding as configuration evidence
5 · Test          ICMP ping again — confirm allowed traffic still passes
6 · Document      Before / configured / after / rollback command
```

---

## Evidence

| # | Requirement | What it proves | |
|---|---|---|---|
| 1 | `baseline` | Ping worked before the ACL was applied | ✅ |
| 2 | `acl_config` | The access-list exists and contains the correct rules | ✅ |
| 3 | `acl_application` | The ACL is bound to the right interface and direction | ✅ |
| 4 | `allowed_test` | ICMP ping still succeeds after the ACL | ✅ |
| 5 | `final_note` | A written explanation of the policy and the rollback plan | ✅ |

### 1 · Baseline

```console
<router-1>#ping <loopback-address-2>
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to <loopback-address-2>, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms

<router-2>#ping <loopback-address-1>
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to <loopback-address-1>, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

> **Reading it:** both loopbacks reachable in both directions before any ACL is applied. Without
> this snapshot, nothing after the change is provable.

### 2 · ACL configuration

```console
<router-1>#show access-lists
Extended IP access list 100
    10 deny tcp any host <loopback-address-2> eq telnet
    20 permit icmp any any
    30 permit ip any any
```

> **Reading it:** the ACL denies Telnet to the protected loopback, permits ICMP explicitly, and
> permits all other IP traffic — specific deny first, general permit after.

### 3 · ACL application

```console
<router-1>#show ip interface <interface>
  Inbound  access list is 100
  Outgoing access list is not set
```

> **Reading it:** ACL 100 is applied **inbound** on the correct interface — traffic heading toward
> the protected loopback is filtered *before* it is routed.

### 4 · Allowed-traffic test

```console
<router-1>#ping <loopback-address-2>
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to <loopback-address-2>, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

> **Reading it:** ICMP still passes after the ACL. The permit rule works and allowed traffic is
> unaffected — the filter is scoped, not blunt.

### 5 · Final note

| | |
|---|---|
| **Policy applied** | Extended ACL 100 on `<router-1> <interface>` inbound. Denies Telnet to `<loopback-address-2>`, permits ICMP and all other traffic. |
| **Before state** | Full reachability between both loopbacks (ICMP and Telnet). |
| **After state** | ICMP still passes; Telnet to the protected loopback is denied. All other traffic unaffected. |
| **Rollback** | `no access-list 100`, then `no ip access-group 100 in` on the interface. Restores full reachability immediately. |

---

## Verification & outcome

> [!TIP]
> **Approved note — reviewing instructor**
>
> Clean build. Baseline captured before any changes, ACL scoped correctly (deny specific + permit
> general), applied on the right interface in the right direction, and allowed traffic verified
> after. Rollback plan is one command. **Approved.**

<sub>Reviewed and approved by an ARIA instructor on 2026-07-24.</sub>

---

<sub>[← Back to portfolio](../README.md)</sub>
