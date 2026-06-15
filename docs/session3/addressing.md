# Session 3 — Addressing Table

## Loopback Addresses

| Device | Interface | Address |
|--------|-----------|---------|
| PE1 | lo0.0 | 10.0.0.1/32 |
| P1 | lo0.0 | 10.0.0.2/32 |
| P2 | lo0.0 | 10.0.0.3/32 |
| PE2 | lo0.0 | 10.0.0.4/32 |

## Point-to-Point Link Addresses

| Link | Device | Interface | Address |
|------|--------|-----------|---------|
| PE1 — P1 | PE1 | ge-0/0/0 | 10.1.12.1/30 |
| PE1 — P1 | P1 | ge-0/0/0 | 10.1.12.2/30 |
| P1 — P2 | P1 | ge-0/0/1 | 10.1.23.1/30 |
| P1 — P2 | P2 | ge-0/0/0 | 10.1.23.2/30 |
| P2 — PE2 | P2 | ge-0/0/1 | 10.1.34.1/30 |
| P2 — PE2 | PE2 | ge-0/0/0 | 10.1.34.2/30 |

## OSPF Parameters

| Parameter | Value |
|-----------|-------|
| Process | Not named in Junos (one process per protocol instance) |
| Area | 0.0.0.0 (backbone) |
| Router IDs | PE1=10.0.0.1, P1=10.0.0.2, P2=10.0.0.3, PE2=10.0.0.4 |
| Interface type | p2p on all transit links |
| Hello interval | 10s (default) |
| Dead interval | 40s (default) |

!!! note "Junos OSPF area notation"
    Junos accepts both `area 0` and `area 0.0.0.0` — they are equivalent. This lab uses `area 0.0.0.0` for clarity.
