# Session 8 — Addressing

All IP addresses are unchanged from Sessions 4–7. Session 8 introduces VPN identifiers (RD and RT) rather than new IP addresses.

## Loopback Addresses

| Device | Interface | Address |
|--------|-----------|---------|
| PE1 | lo0.0 | 10.0.0.1/32 |
| P1 | lo0.0 | 10.0.0.2/32 |
| P2 | lo0.0 | 10.0.0.3/32 |
| PE2 | lo0.0 | 10.0.0.4/32 |
| CE1 | lo0.0 | 10.0.0.11/32 |
| CE2 | lo0.0 | 10.0.0.12/32 |

## Provider Point-to-Point Links

| Link | Interface | Address |
|------|-----------|---------|
| PE1 ge-0/0/0 | PE1 - P1 | 10.1.12.1/30 |
| P1 ge-0/0/0 | PE1 - P1 | 10.1.12.2/30 |
| P1 ge-0/0/1 | P1 - P2 | 10.1.23.1/30 |
| P2 ge-0/0/0 | P1 - P2 | 10.1.23.2/30 |
| P2 ge-0/0/1 | P2 - PE2 | 10.1.34.1/30 |
| PE2 ge-0/0/0 | P2 - PE2 | 10.1.34.2/30 |

## CE-PE Links (VPN-A VRF)

| Link | Interface | Address | VRF |
|------|-----------|---------|-----|
| PE1 ge-0/0/1 | PE1 - CE1 | 172.16.1.1/30 | VPN-A |
| CE1 ge-0/0/0 | PE1 - CE1 | 172.16.1.2/30 | global |
| PE2 ge-0/0/1 | PE2 - CE2 | 172.16.2.1/30 | VPN-A |
| CE2 ge-0/0/0 | PE2 - CE2 | 172.16.2.2/30 | global |

PE1 and PE2's CE-facing interfaces are assigned to the `VPN-A` routing instance. The physical addresses are unchanged; only their routing context moves from `inet.0` to `VPN-A.inet.0`.

## VPN-A Identifiers

| PE | Route Distinguisher | Route Target (export) | Route Target (import) |
|----|--------------------|-----------------------|-----------------------|
| PE1 | 65001:1000 | target:65001:100 | target:65001:100 |
| PE2 | 65001:2000 | target:65001:100 | target:65001:100 |

Route distinguishers are unique per PE per VPN. The route target is identical on both PEs — both export and import `target:65001:100`, which causes VPN-A routes from PE1 to be imported by PE2's VPN-A and vice versa.

## BGP Autonomous Systems

| AS | Role |
|----|------|
| 65001 | Provider (PE1, P1, P2, PE2) |
| 65100 | Customer A (CE1, CE2) |
