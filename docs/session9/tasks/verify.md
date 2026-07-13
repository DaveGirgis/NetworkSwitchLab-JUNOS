# Session 9 - Verification Checklist

## Fault and Fix Summary

| Fault | Layer | Root Cause | Fix | Key Diagnostic Signature |
|-------|-------|------------|-----|---------------------------|
| 1 | IGP (IS-IS) | P2's `ge-0/0/0.0` interface (facing P1) pinned to `level 1 enable` / `level 2 disable`, while every other router/interface in the topology runs Level 2 only | `delete protocols isis interface ge-0/0/0.0 level 1` and `level 2` on P2, restoring the router-wide Level 2 default | Both VPN-A and VPLS-100 fail together; `show isis adjacency` shows a missing neighbor on both sides of one link; `show isis interface <if> detail` shows mismatched `Circuit type` on the two ends |
| 2 | BGP / MP-BGP | `family inet-vpn unicast` removed from PE2's iBGP group toward PE1 | `set protocols bgp group IBGP family inet-vpn unicast` on PE2 | iBGP session stays `Establ`; VPLS-100 keeps working; `show bgp neighbor <ip> \| match NLRI` is missing `inet-vpn-unicast`; `bgp.l3vpn.0` and per-peer `VPN-A.inet.0` lines disappear from `show bgp summary` |
| 3 | MPLS / L3VPN (route target) | PE2's `VPN-A` routing-instance `vrf-target` changed from `target:65001:100` to `target:65001:200` | `set routing-instances VPN-A vrf-target target:65001:100` on PE2 | BGP session and address families healthy; route present in `bgp.l3vpn.0` but missing from `VPN-A.inet.0`; VPLS-100 unaffected (separate routing-instance, separate RT) |

## Layer-by-Layer Health Check

### Physical / Interfaces

- [ ] `PE1> show interfaces terse | match ge-` - all interfaces `up up`
- [ ] `P1> show interfaces terse | match ge-` - all interfaces `up up`
- [ ] `P2> show interfaces terse | match ge-` - all interfaces `up up`
- [ ] `PE2> show interfaces terse | match ge-` - all interfaces `up up`

### IS-IS

- [ ] `PE1> show isis adjacency` - one adjacency (P1), `Up`
- [ ] `P1> show isis adjacency` - two adjacencies (PE1, P2), both `Up`
- [ ] `P2> show isis adjacency` - two adjacencies (P1, PE2), both `Up`
- [ ] `PE2> show isis adjacency` - one adjacency (P2), `Up`
- [ ] `P2> show configuration protocols isis interface ge-0/0/0.0` - no `level 1 enable` or `level 2 disable` override remains

### LDP / MPLS Transport

- [ ] `PE1> show ldp session` - session(s) `Operational`
- [ ] `PE1> show route table inet.3` - three destinations (10.0.0.2, 10.0.0.3, 10.0.0.4), 10.0.0.4/32 shows a `Push` label
- [ ] `PE2> show route table inet.3` - three destinations (10.0.0.1, 10.0.0.2, 10.0.0.3), 10.0.0.1/32 shows a `Push` label

### BGP / MP-BGP

- [ ] `PE1> show bgp summary` - peer 10.0.0.4 shows `Establ`
- [ ] `PE1> show bgp neighbor 10.0.0.4 | match NLRI` - `inet-vpn-unicast l2vpn` present on every NLRI line
- [ ] `PE2> show bgp neighbor 10.0.0.1 | match NLRI` - `inet-vpn-unicast l2vpn` present on every NLRI line
- [ ] `PE1> show configuration protocols bgp group IBGP` - `family inet-vpn unicast` and `family l2vpn signaling` both present
- [ ] `PE2> show configuration protocols bgp group IBGP` - `family inet-vpn unicast` and `family l2vpn signaling` both present

### VPN-A (L3VPN)

- [ ] `PE1> show configuration routing-instances VPN-A vrf-target` - `target:65001:100`
- [ ] `PE2> show configuration routing-instances VPN-A vrf-target` - `target:65001:100`
- [ ] `PE1> show route table bgp.l3vpn.0` - two routes: `65001:1000:10.0.0.11/32` and `65001:2000:10.0.0.12/32`
- [ ] `PE2> show route table bgp.l3vpn.0` - two routes: `65001:1000:10.0.0.11/32` and `65001:2000:10.0.0.12/32`
- [ ] `PE1> show route table VPN-A.inet.0` - four routes: `172.16.1.0/30` (Direct), `10.0.0.11/32` (BGP from CE1), `10.0.0.12/32` (BGP from PE2, two-label Push stack), `172.16.2.0/30` (BGP from PE2)
- [ ] `PE2> show route table VPN-A.inet.0` - four routes: `172.16.2.0/30` (Direct), `10.0.0.12/32` (BGP from CE2), `10.0.0.11/32` (BGP from PE1, two-label Push stack), `172.16.1.0/30` (BGP from PE1)
- [ ] `PE1> show bgp summary instance VPN-A` - CE1 (172.16.1.2) established
- [ ] `PE2> show bgp summary instance VPN-A` - CE2 (172.16.2.2) established
- [ ] `CE1> show route receive-protocol bgp 172.16.1.1` - `10.0.0.12/32` present with AS path `65001 65001 I`
- [ ] `CE2> show route receive-protocol bgp 172.16.2.1` - `10.0.0.11/32` present with AS path `65001 65001 I`

### VPLS-100 (L2VPN, Unaffected Control Group)

- [ ] `PE1> show vpls connections` - connection-site 2, `Up`
- [ ] `PE2> show vpls connections` - connection-site 1, `Up`
- [ ] `PE1> show route table VPLS-100.l2vpn.0` - two routes present
- [ ] `PE1> show configuration routing-instances VPLS-100 vrf-target` - `target:65001:100` (unchanged, independent of VPN-A's RT)

### End-to-End Reachability

- [ ] `CE1> ping 10.0.0.12 source 10.0.0.11 count 5` - 5/5 replies, 0% packet loss (VPN-A)
- [ ] `CE2> ping 10.0.0.11 source 10.0.0.12 count 5` - 5/5 replies, 0% packet loss (VPN-A)
- [ ] `CE1> ping 192.168.1.2 count 5` - 5/5 replies, 0% packet loss (VPLS-100)
- [ ] `CE2> ping 192.168.1.1 count 5` - 5/5 replies, 0% packet loss (VPLS-100)

## Quick Commands

Run these four commands, in this order, any time you need a fast top-to-bottom health read of the whole stack:

```junos
show isis adjacency
```
Expected: correct number of `Up` adjacencies for that router's position in the topology (1 for PE1/PE2, 2 for P1/P2).

```junos
show ldp session
```
Expected: all sessions `Operational`.

```junos
show bgp summary
```
Expected: iBGP peer `Establ`, with `bgp.l3vpn.0` and `bgp.l2vpn.0` both listed and both showing active paths.

```junos
show route table VPN-A.inet.0
```
Expected on PE1 or PE2: four routes, including the remote CE loopback with a two-label `Push` stack.

## Reflection

Before closing out this session, answer these for yourself - the goal is retention of the method, not just the three fixes:

- For each fault, what was the very first command you ran, and was it the right place to start?
- Which fault would have been fastest to misdiagnose if you had started troubleshooting from the top of the stack (VPN-A configuration) instead of the bottom (interfaces, then IGP)?
- What is the single distinguishing symptom that told you Fault 2 was a BGP address-family problem and not a VRF problem, given that both faults produced an identical CE1-to-CE2 ping failure?
- Why did VPLS-100 serve as a useful control group throughout this session, and what would it have meant if VPLS-100 had also broken during Fault 2 or Fault 3?

This closes the JNCIS-SP lab series. You have now built, from nothing, a working IS-IS core, an MPLS/LDP transport layer, a BGP/MPLS Layer 3 VPN, a Layer 2 VPLS service, and - in this final session - practiced the diagnostic discipline needed to keep all of it running when something breaks in production.
