# Session 6 — Verification Checklist

## Topology

- [ ] CE1 and CE2 nodes added to GNS3 project and running
- [ ] CE1 Adapter 2 linked to PE1 Adapter 3 (ge-0/0/0 ↔ ge-0/0/1)
- [ ] CE2 Adapter 2 linked to PE2 Adapter 3 (ge-0/0/0 ↔ ge-0/0/1)

## Interfaces

- [ ] `PE1> show interfaces terse` — `ge-0/0/1.0` is `up up`, address `172.16.1.1/30`
- [ ] `PE2> show interfaces terse` — `ge-0/0/1.0` is `up up`, address `172.16.2.1/30`
- [ ] `CE1> show interfaces terse` — `ge-0/0/0.0` is `172.16.1.2/30`, `lo0.0` is `10.0.0.11/32`
- [ ] `CE2> show interfaces terse` — `ge-0/0/0.0` is `172.16.2.2/30`, `lo0.0` is `10.0.0.12/32`

## eBGP Sessions

- [ ] `PE1> show bgp summary` — peer `172.16.1.2` (CE1) shows `Establ`
- [ ] `PE2> show bgp summary` — peer `172.16.2.2` (CE2) shows `Establ`
- [ ] `CE1> show bgp summary` — peer `172.16.1.1` (PE1) shows `Establ`
- [ ] `CE2> show bgp summary` — peer `172.16.2.1` (PE2) shows `Establ`

## iBGP Session

- [ ] `PE1> show bgp summary` — peer `10.0.0.4` (PE2) shows `Establ`
- [ ] `PE2> show bgp summary` — peer `10.0.0.1` (PE1) shows `Establ`

## Prefix Propagation

- [ ] `PE1> show route receive-protocol bgp 172.16.1.2` — shows `10.0.0.11/32` from CE1
- [ ] `PE2> show route receive-protocol bgp 172.16.2.2` — shows `10.0.0.12/32` from CE2
- [ ] `PE1> show route receive-protocol bgp 10.0.0.4` — shows `10.0.0.12/32` from PE2 via iBGP
- [ ] `PE2> show route receive-protocol bgp 10.0.0.1` — shows `10.0.0.11/32` from PE1 via iBGP
- [ ] `CE1> show route receive-protocol bgp 172.16.1.1` — shows `10.0.0.12/32`, AS path `65001 65100`
- [ ] `CE2> show route receive-protocol bgp 172.16.2.1` — shows `10.0.0.11/32`, AS path `65001 65100`

## BGP Table

- [ ] `PE1> show route protocol bgp` — two routes: `10.0.0.11/32` (eBGP) and `10.0.0.12/32` (iBGP)
- [ ] `PE2> show route protocol bgp` — two routes: `10.0.0.12/32` (eBGP) and `10.0.0.11/32` (iBGP)

## Quick Commands

```junos
show bgp summary
```
Expected: all peers in `Establ` state

```junos
show route protocol bgp
```
Expected on PE1: `10.0.0.11/32` via ge-0/0/1 (eBGP), `10.0.0.12/32` via IS-IS next-hop (iBGP)

```junos
show bgp neighbor 10.0.0.4
```
Expected on PE1: `Type: Internal`, `State: Established`, `next-hop-self` confirmed in options
