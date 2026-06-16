# Session 3 — Verification Checklist

## Topology

- [ ] Four vJunos-router nodes (PE1, P1, P2, PE2) are running in GNS3
- [ ] All three links show green (up) on the GNS3 canvas

## Interfaces

- [ ] `PE1> show interfaces terse` — `ge-0/0/0.0` is `up up`, address `10.1.12.1/30`; `lo0.0` is `up up`, address `10.0.0.1/32`
- [ ] `P1> show interfaces terse` — `ge-0/0/0.0` is `10.1.12.2/30`, `ge-0/0/1.0` is `10.1.23.1/30`, `lo0.0` is `10.0.0.2/32`
- [ ] `P2> show interfaces terse` — `ge-0/0/0.0` is `10.1.23.2/30`, `ge-0/0/1.0` is `10.1.34.1/30`, `lo0.0` is `10.0.0.3/32`
- [ ] `PE2> show interfaces terse` — `ge-0/0/0.0` is `10.1.34.2/30`, `lo0.0` is `10.0.0.4/32`

## OSPF Adjacency

- [ ] `PE1> show ospf neighbor` — one neighbor: `10.0.0.2` in state `Full`
- [ ] `P1> show ospf neighbor` — two neighbors: `10.0.0.1` and `10.0.0.3` both `Full`
- [ ] `P2> show ospf neighbor` — two neighbors: `10.0.0.2` and `10.0.0.4` both `Full`
- [ ] `PE2> show ospf neighbor` — one neighbor: `10.0.0.3` in state `Full`

## Routing Table

- [ ] `PE1> show route protocol ospf` — shows routes to `10.0.0.2/32`, `10.0.0.3/32`, `10.0.0.4/32` and the remote /30 subnets
- [ ] `PE2> show route protocol ospf` — shows routes to `10.0.0.1/32`, `10.0.0.2/32`, `10.0.0.3/32` and the remote /30 subnets

## End-to-End Connectivity

- [ ] `PE1> ping 10.0.0.4 count 5` — 5/5 replies
- [ ] `PE2> ping 10.0.0.1 count 5` — 5/5 replies
- [ ] `PE1> traceroute 10.0.0.4` — path shows P1 (`10.1.12.2` or `10.0.0.2`) then P2 (`10.1.23.2` or `10.0.0.3`) before reaching PE2

## OSPF Database

- [ ] `PE1> show ospf database` — 4 Router LSAs present (one per router), no Network LSAs
