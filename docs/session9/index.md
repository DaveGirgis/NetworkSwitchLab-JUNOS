# Session 9 - Capstone Troubleshooting Lab

## Overview

Sessions 1 through 8 built a complete service provider network layer by layer: interfaces and static routing, bridging, IS-IS, BGP, MPLS/LDP, Layer 2 services (VPLS-100 from Session 7a), and a BGP/MPLS L3 VPN (VPN-A from Session 8). Every protocol you configured is still running at the end of Session 8 - IS-IS adjacencies across the core, iBGP between PE1 and PE2 carrying three address families (`inet-vpn unicast`, `l2vpn signaling`, plus the underlying `inet-unicast`), LDP label distribution, and two independent customer services (VPLS-100 and VPN-A) sharing the same physical topology.

This is the capstone. There is no new configuration to add. Instead, the instructor (or a provided script) reaches into your working network and injects **three deliberate faults, one at a time**, into three different layers of the stack. Your job is not to be told what broke - it is to find out, using the same systematic, layer-by-layer diagnostic method a working SP engineer uses when a ticket says only "customer says the network is down."

## Objectives

By the end of this session you will be able to:

- [ ] Apply a structured, layer-by-layer troubleshooting methodology (physical/interface -> IGP -> BGP -> MPLS/VPN) to an unfamiliar fault
- [ ] Form a hypothesis from symptom evidence before touching configuration
- [ ] Use `show` commands to isolate a fault to a single layer, then to a single router, then to a single line of configuration
- [ ] Recognize the specific symptoms an IS-IS adjacency fault produces downstream in BGP and MPLS
- [ ] Recognize the specific symptoms a BGP/MP-BGP fault produces even when the IGP and MPLS transport are healthy
- [ ] Recognize the specific symptoms a VRF/route-target/MPLS fault produces even when IS-IS and BGP sessions are up
- [ ] Distinguish "the protocol session is down" from "the protocol session is up but not doing what it should" - the harder class of fault
- [ ] Restore full CE-to-CE reachability for both VPN-A (L3 VPN) and VPLS-100 (L2 VPN) after each fix
- [ ] Explain, in your own words, why each fault broke what it broke - not just what command fixed it

## Prerequisites

- Sessions 1-8 complete, including Session 7a (Layer 2 Services) - this session assumes VPLS-100 is running alongside VPN-A
- All six nodes (CE1, PE1, P1, P2, PE2, CE2) booted and reachable via console
- A saved copy of your working end-of-Session-8 configuration for each router (see [Part 0](tasks/part0.md) if you do not have one)
- Comfort with the show commands from Sessions 5-8: `show isis adjacency`, `show bgp summary`, `show route table inet.3`, `show route table VPN-A.inet.0`, `show ldp session`, `show vpls connections`

## Methodology: How to Troubleshoot This Session

Every fault in this session is solvable with show commands alone - no packet captures, no external tools. The discipline that matters is the **order** in which you check things. The provider network is layered, and each layer depends on the one below it:

```
Physical / interface  -->  IGP (IS-IS)  -->  MPLS (LDP)  -->  BGP  -->  L3VPN / L2VPN
```

If IS-IS is broken, BGP next-hops cannot resolve and every symptom above IS-IS will look like a BGP or VPN problem even though BGP itself is configured correctly. The methodology this session teaches:

1. **Confirm the symptom** - what exactly fails? A ping? A missing route? A down session? Write down the exact command and exact output.
2. **Start at the bottom of the stack, not the top** - check interfaces and IS-IS before you touch BGP configuration, even if the symptom looks like a BGP problem (e.g., "CE1 can't ping CE2" is often not a BGP fault at all).
3. **Narrow by comparison** - compare the same show command across multiple routers. A working P1-PE1 adjacency next to a missing P1-P2 adjacency tells you exactly where to look.
4. **Form a hypothesis before changing configuration** - state, out loud or in writing, "I believe X is misconfigured on router Y because show command Z shows W instead of the expected value." Only then open configuration mode.
5. **Fix the smallest possible change** - a single `set` or `delete`, not a re-paste of the whole stanza.
6. **Verify at the layer you fixed, then verify end-to-end** - confirm the specific show command output is now correct, then re-test CE-to-CE reachability for both VPN-A and VPLS-100.

!!! warning "Do not skip layers"
    The single most common mistake in this lab is jumping straight to `show bgp summary` or `show route table VPN-A.inet.0` because the symptom "looks like" a BGP or VPN problem. Every fault in this session is diagnosable in under five commands if you start at the bottom of the stack and work up. Starting at the top wastes time chasing a symptom instead of a cause.

## Session Parts

| Part | Fault Layer | What Breaks |
|------|-------------|--------------|
| [Part 0](tasks/part0.md) | None (baseline) | Import/restore the known-good end-of-Session-8 configuration and confirm it is healthy before any fault is injected |
| [Part 1](tasks/part1.md) | IGP (IS-IS) | An IS-IS adjacency-breaking fault on the P1-P2 link removes core loopback reachability |
| [Part 2](tasks/part2.md) | BGP / MP-BGP | An iBGP address-family fault on PE2 breaks VPN-A route exchange even though IS-IS and LDP are healthy |
| [Part 3](tasks/part3.md) | MPLS / L3VPN | A route-target mismatch on PE2's VPN-A VRF breaks VPN-A route import even though BGP sessions are up |
| [Verification](tasks/verify.md) | All layers | End-to-end checklist confirming CE1-CE2 reachability over VPN-A, VPLS-100 unaffected, and a fault/fix summary table |

Each fault must be fixed before the next is introduced. This mirrors real operations - you never get to diagnose three unrelated faults simultaneously with a clean signal for each; you fix what you find and keep moving up the stack.
