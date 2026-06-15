# Session 2 — Topology

## Diagram

```mermaid
graph LR
  R1([R1<br/>lo0: 10.0.0.1/32]) -- ge-0/0/0<br/>10.1.12.1/30 --- ge-0/0/0<br/>10.1.12.2/30 -- R2([R2<br/>lo0: 10.0.0.2/32])
```

## Device Summary

| Device | Role | Loopback |
|--------|------|----------|
| R1 | Router | 10.0.0.1/32 |
| R2 | Router | 10.0.0.2/32 |

## Link Summary

| Link | R1 Interface | R1 Address | R2 Interface | R2 Address |
|------|-------------|------------|-------------|------------|
| R1 — R2 | ge-0/0/0 | 10.1.12.1/30 | ge-0/0/0 | 10.1.12.2/30 |

## Notes

This is the same two-router project from Session 1. Open `JNCIS-SP-Session1` in GNS3 (or create a new project `JNCIS-SP-Session2` with the same topology). The link between R1 and R2 on `ge-0/0/0` was already drawn in Session 1.

The loopback addresses (`10.0.0.X/32`) are used throughout this series as stable router identifiers — they remain reachable even if a physical link goes down. OSPF, BGP, and MPLS all reference loopbacks as router IDs.
