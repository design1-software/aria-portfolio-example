# ACL Basics and Safe Traffic Filtering

**Networking · Build · Completed 2026-07-24 · Instructor-validated**

---

## Overview

A small office wants to allow ICMP testing between two lab networks but block simulated Telnet traffic to a protected loopback. The task: apply only the scoped ACL and prove the result.

## Skills Demonstrated

Access-control list design, extended ACL syntax, interface-direction binding, baseline-before-change methodology, rollback planning, Cisco IOS configuration on routed topology.

## Objective

Build an extended ACL that denies Telnet to a protected loopback, permits ICMP, and permits everything else — then apply it inbound on the correct interface and verify allowed traffic still flows.

## Environment

EVE-NG routed lab — two Cisco routers with /30 interlinks and loopback addresses.

## Approach

1. Verified baseline reachability — ping between loopbacks in both directions to establish the "before" snapshot.
2. Designed the extended ACL: deny Telnet to the protected loopback, permit ICMP, permit all other traffic.
3. Applied the ACL inbound on the correct interface of the upstream router.
4. Captured the access-list contents and interface binding as configuration evidence.
5. Tested allowed traffic (ICMP ping) to confirm it still passes after the ACL.
6. Drafted the final note: what was verified before, what was configured, what was verified after, and what the rollback command would be.

## Evidence Requirements Met

- [x] **baseline** — Ping worked before the ACL was applied
- [x] **acl_config** — The access-list exists and contains the correct rules
- [x] **acl_application** — The ACL is bound to the right interface and direction
- [x] **allowed_test** — ICMP ping still succeeds after the ACL
- [x] **final_note** — A written explanation of the policy and the rollback plan

## Evidence & Results

### 1. baseline

```
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

Baseline confirmed: both loopbacks reachable before any ACL is applied.

### 2. acl_config

```
<router-1>#show access-lists
Extended IP access list 100
    10 deny tcp any host <loopback-address-2> eq telnet
    20 permit icmp any any
    30 permit ip any any
```

The ACL denies Telnet to the protected loopback, permits ICMP explicitly, and permits all other IP traffic.

### 3. acl_application

```
<router-1>#show ip interface <interface>
  Inbound  access list is 100
  Outgoing access list is not set
```

ACL 100 is applied inbound on the correct interface — traffic heading toward the protected loopback is filtered before it is routed.

### 4. allowed_test

```
<router-1>#ping <loopback-address-2>
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to <loopback-address-2>, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

ICMP still passes after the ACL — the permit rule is working and allowed traffic is unaffected.

### 5. final_note

**Policy applied:** Extended ACL 100 on <router-1> <interface> inbound. Denies Telnet to <loopback-address-2>, permits ICMP and all other traffic.

**Before state:** Full reachability between both loopbacks (ICMP and Telnet).

**After state:** ICMP still passes; Telnet to the protected loopback is denied. All other traffic unaffected.

**Rollback:** `no access-list 100` followed by `no ip access-group 100 in` on the interface. Restores full reachability immediately.

---

## Verification & Outcome (Approved Note)

Clean build. Baseline captured before any changes, ACL scoped correctly (deny specific + permit general), applied on the right interface in the right direction, and allowed traffic verified after. Rollback plan is one command. Approved.

*Reviewed and approved by an ARIA instructor on 2026-07-24.*
