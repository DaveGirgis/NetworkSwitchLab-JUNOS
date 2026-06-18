# Session 5 — Verification Checklist

## OSPF Removal

- [ ] `PE1> show ospf neighbor` — returns empty or `OSPF instance is not running`
- [ ] `P1> show ospf neighbor` — same
- [ ] No OSPF routes in `show route protocol ospf` on any router

## IS-IS Configuration

- [ ] `PE1> show isis interface` — `ge-0/0/0.0` shows `Point to Point`, `lo0.0` shows `Passive`
- [ ] `P1> show isis interface` — both `ge-0/0/0.0` and `ge-0/0/1.0` show `Point to Point`
- [ ] `show configuration protocols isis` on PE1 shows `level 1 disable` and interface stanzas

## IS-IS Adjacency

- [ ] `PE1> show isis adjacency` — one neighbor: `P1`, state `Up`, level `2`
- [ ] `P1> show isis adjacency` — two neighbors: `PE1` and `P2`, both `Up`
- [ ] `P2> show isis adjacency` — two neighbors: `P1` and `PE2`, both `Up`
- [ ] `PE2> show isis adjacency` — one neighbor: `P2`, state `Up`

## Link-State Database

- [ ] `PE1> show isis database` — four LSPs present: PE1, P1, P2, PE2

## Routing Table

- [ ] `PE1> show route protocol isis` — routes to `10.0.0.2/32`, `10.0.0.3/32`, `10.0.0.4/32` and remote /30 subnets, all tagged `[IS-IS/18]`
- [ ] `PE2> show route protocol isis` — routes to `10.0.0.1/32`, `10.0.0.2/32`, `10.0.0.3/32`

## End-to-End Connectivity

- [ ] `PE1> ping 10.0.0.4 count 5` — 5/5 replies
- [ ] `PE2> ping 10.0.0.1 count 5` — 5/5 replies

## Quick Commands

```junos
show isis adjacency
```
Expected on P1: two neighbors, both `Up`, level `2`

```junos
show isis database
```
Expected: four LSPs, one per router

```junos
show route protocol isis
```
Expected: loopbacks and /30 subnets with `[IS-IS/18]` tag
