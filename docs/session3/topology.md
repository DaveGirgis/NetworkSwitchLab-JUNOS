# Session 3 — Topology

## Diagram

```mermaid
graph LR
  PE1([PE1<br/>lo0: 10.0.0.1/32]) -- ge-0/0/0<br/>10.1.12.1 --- ge-0/0/0<br/>10.1.12.2 -- P1([P1<br/>lo0: 10.0.0.2/32])
  P1 -- ge-0/0/1<br/>10.1.23.1 --- ge-0/0/0<br/>10.1.23.2 -- P2([P2<br/>lo0: 10.0.0.3/32])
  P2 -- ge-0/0/1<br/>10.1.34.1 --- ge-0/0/0<br/>10.1.34.2 -- PE2([PE2<br/>lo0: 10.0.0.4/32])
```

## Device Summary

| Device | Role | Loopback |
|--------|------|----------|
| PE1 | Provider Edge | 10.0.0.1/32 |
| P1 | Provider Core | 10.0.0.2/32 |
| P2 | Provider Core | 10.0.0.3/32 |
| PE2 | Provider Edge | 10.0.0.4/32 |

## Link Summary

| Link | Left Device | Left Interface | Left Address | Right Device | Right Interface | Right Address |
|------|------------|---------------|-------------|-------------|----------------|--------------|
| PE1 — P1 | PE1 | ge-0/0/0 | 10.1.12.1/30 | P1 | ge-0/0/0 | 10.1.12.2/30 |
| P1 — P2 | P1 | ge-0/0/1 | 10.1.23.1/30 | P2 | ge-0/0/0 | 10.1.23.2/30 |
| P2 — PE2 | P2 | ge-0/0/1 | 10.1.34.1/30 | PE2 | ge-0/0/0 | 10.1.34.2/30 |

## GNS3 Project Setup

Create a new GNS3 project `JNCIS-SP-Core` with four vJunos-router nodes. Draw the links as follows:

| Link | Node A Port | Node B Port |
|------|-------------|-------------|
| PE1 — P1 | Adapter 0 (ge-0/0/0) | Adapter 0 (ge-0/0/0) |
| P1 — P2 | Adapter 1 (ge-0/0/1) | Adapter 0 (ge-0/0/0) |
| P2 — PE2 | Adapter 1 (ge-0/0/1) | Adapter 0 (ge-0/0/0) |

!!! warning "GNS3 adapter numbering"
    In GNS3, Adapter 0 = `ge-0/0/0`, Adapter 1 = `ge-0/0/1`, Adapter 2 = `ge-0/0/2`, Adapter 3 = `ge-0/0/3`. Always verify the mapping by running `show interfaces terse` after connecting links — the link light on the GNS3 canvas should turn green, and the interface should show `up up`.

## Notes

This four-router topology is the foundation for all remaining sessions (3–8). Save this GNS3 project as `JNCIS-SP-Core` and reuse it — you will add OSPF in Session 3, then BGP on top in Session 5, then MPLS in Session 6, and VPNs in Session 7. Each session builds on the previous committed state.
