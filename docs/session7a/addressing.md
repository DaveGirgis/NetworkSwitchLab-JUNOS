# Session 7a — Addressing

All addresses from Sessions 4–7 are unchanged. This session adds one new subnet for Layer 2 service testing.

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

## CE-PE BGP Links (Sessions 5-6)

| Link | Interface | Address |
|------|-----------|---------|
| PE1 ge-0/0/1 | PE1 - CE1 | 172.16.1.1/30 |
| CE1 ge-0/0/0 | PE1 - CE1 | 172.16.1.2/30 |
| PE2 ge-0/0/1 | PE2 - CE2 | 172.16.2.1/30 |
| CE2 ge-0/0/0 | PE2 - CE2 | 172.16.2.2/30 |

## L2 Service Test Interfaces (Session 7a)

| Interface | Address | Purpose |
|-----------|---------|---------|
| CE1 ge-0/0/1.0 | 192.168.1.1/24 | L2circuit / VPLS test source |
| CE2 ge-0/0/1.0 | 192.168.1.2/24 | L2circuit / VPLS test destination |
| PE1 ge-0/0/2.0 | none | L2 pass-through (CCC / VPLS access) |
| PE2 ge-0/0/2.0 | none | L2 pass-through (CCC / VPLS access) |

CE1 and CE2 use 192.168.1.0/24 as a test subnet. Because the L2 service is transparent at Layer 2, CE1 and CE2 appear to be directly connected on the same Ethernet segment. No routing is needed between them — they are in the same /24.

!!! note "The /24 only exists at the CE layer"
    PE1 and PE2 do not have IP addresses on the L2 service interfaces and do not participate in the 192.168.1.0/24 subnet. The provider core has no awareness of this customer prefix.

## BGP Autonomous Systems

| AS | Role |
|----|------|
| 65001 | Provider (PE1, P1, P2, PE2) |
| 65100 | Customer A (CE1, CE2) |
