# Session 9 - Addressing

No new IP addresses, ASNs, RDs, or RTs are introduced in this session. Every value below is inherited unchanged from Sessions 5-8 and Session 7a. This page exists so you have one reference while troubleshooting, without needing to flip back through five prior sessions.

## Loopback Addresses

| Device | Interface | Address |
|--------|-----------|---------|
| PE1 | lo0.0 | 10.0.0.1/32 |
| P1 | lo0.0 | 10.0.0.2/32 |
| P2 | lo0.0 | 10.0.0.3/32 |
| PE2 | lo0.0 | 10.0.0.4/32 |
| CE1 | lo0.0 | 10.0.0.11/32 |
| CE2 | lo0.0 | 10.0.0.12/32 |

## Provider Point-to-Point Links (IS-IS + MPLS + LDP)

| Link | Interface | Address |
|------|-----------|---------|
| PE1 ge-0/0/0 | PE1 - P1 | 10.1.12.1/30 |
| P1 ge-0/0/0 | PE1 - P1 | 10.1.12.2/30 |
| P1 ge-0/0/1 | P1 - P2 | 10.1.23.1/30 |
| P2 ge-0/0/0 | P1 - P2 | 10.1.23.2/30 |
| P2 ge-0/0/1 | P2 - PE2 | 10.1.34.1/30 |
| PE2 ge-0/0/0 | P2 - PE2 | 10.1.34.2/30 |

!!! note "Part 1's fault lives here"
    The IGP fault in Part 1 is injected on the P1-P2 link (10.1.23.0/30). Nothing about the IP addressing changes - the fault is in the IS-IS protocol configuration, not the interface addressing. Do not "fix" this by changing IP addresses.

## CE-PE Links (VPN-A VRF, from Session 8)

| Link | Interface | Address | VRF |
|------|-----------|---------|-----|
| PE1 ge-0/0/1 | PE1 - CE1 | 172.16.1.1/30 | VPN-A |
| CE1 ge-0/0/0 | PE1 - CE1 | 172.16.1.2/30 | global |
| PE2 ge-0/0/1 | PE2 - CE2 | 172.16.2.1/30 | VPN-A |
| CE2 ge-0/0/0 | PE2 - CE2 | 172.16.2.2/30 | global |

## VPN-A Identifiers (from Session 8)

| PE | Route Distinguisher | Route Target (export) | Route Target (import) |
|----|--------------------|-----------------------|-----------------------|
| PE1 | 65001:1000 | target:65001:100 | target:65001:100 |
| PE2 | 65001:2000 | target:65001:100 | target:65001:100 |

!!! warning "Part 3's fault lives here"
    The route-target values above are the **correct** baseline values. Part 3 injects a mismatch on PE2's `vrf-target`. When you reach Part 3, do not assume the values above are still what is configured - verify with `show configuration routing-instances VPN-A vrf-target` before assuming anything.

## VPLS-100 Identifiers (from Session 7a)

| PE | Route Distinguisher | Route Target (vrf-target) |
|----|--------------------|-----------------------------|
| PE1 | 65001:100 | target:65001:100 |
| PE2 | 65001:101 | target:65001:100 |

!!! danger "VPLS-100 and VPN-A use different RD numbering but must not collide"
    VPLS-100 uses route-distinguishers `65001:100` (PE1) and `65001:101` (PE2). VPN-A uses `65001:1000` (PE1) and `65001:2000` (PE2). These were deliberately chosen in Session 8 to avoid RD collision between the two services on the same router. Both services independently use `vrf-target target:65001:100` for their own import/export - this is not a conflict because RD and RT serve different purposes, and each routing instance (VPN-A vs. VPLS-100) only imports routes belonging to its own instance type and NLRI. If you ever see VPN-A and VPLS-100 routes bleeding into each other during troubleshooting, treat that as a strong signal that a route-distinguisher or vrf-target was typed into the wrong routing-instance stanza.

## BGP Autonomous Systems

| AS | Role |
|----|------|
| 65001 | Provider (PE1, P1, P2, PE2) |
| 65100 | Customer A (CE1, CE2) |

## L2 Service Test Interfaces (VPLS-100, from Session 7a)

| Interface | Address | Purpose |
|-----------|---------|---------|
| CE1 ge-0/0/1.0 | 192.168.1.1/24 | VPLS-100 test source |
| CE2 ge-0/0/1.0 | 192.168.1.2/24 | VPLS-100 test destination |
| PE1 ge-0/0/2.0 | none | VPLS-100 access port |
| PE2 ge-0/0/2.0 | none | VPLS-100 access port |

This subnet and its associated pseudowire are used throughout this session as a control group - if VPLS-100 (192.168.1.1 to 192.168.1.2) keeps working while VPN-A (10.0.0.11 to 10.0.0.12) breaks, that is strong evidence the fault is specific to VPN-A's control plane (BGP address family or route target) rather than the shared IS-IS/LDP transport underneath both services.
