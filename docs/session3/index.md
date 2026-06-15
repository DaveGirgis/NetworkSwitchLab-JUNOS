# Session 3 — OSPF Single-Area

## Objectives

By the end of this session you will be able to:

- [ ] Build the four-router provider core topology in GNS3
- [ ] Configure OSPF area 0 on all provider interfaces
- [ ] Verify OSPF neighbor adjacency on all links
- [ ] Explain LSA types 1 and 2 and read the OSPF LSDB
- [ ] Configure passive interfaces to suppress OSPF hellos on loopbacks
- [ ] Summarize routes at the OSPF ABR (preview for Session 4)

## Prerequisites

- Sessions 1 and 2 complete — comfortable with Junos CLI and interface configuration
- Understand IP addressing and subnetting (/30 point-to-point links)

## Topology Overview

```mermaid
graph LR
  PE1([PE1<br/>10.0.0.1]) -- ge-0/0/0<br/>10.1.12.0/30 --- ge-0/0/0<br/>10.1.12.0/30 -- P1([P1<br/>10.0.0.2])
  P1 -- ge-0/0/1<br/>10.1.23.0/30 --- ge-0/0/0<br/>10.1.23.0/30 -- P2([P2<br/>10.0.0.3])
  P2 -- ge-0/0/1<br/>10.1.34.0/30 --- ge-0/0/0<br/>10.1.34.0/30 -- PE2([PE2<br/>10.0.0.4])
```

This is the full provider backbone used in Sessions 3–8. It has four routers: two **Provider Edge (PE)** routers that connect to customers, and two **Provider (P)** routers that form the transit core.

## Session Parts

| Part | Topic |
|------|-------|
| [Part 0](tasks/part0.md) | Build topology & base config |
| [Part 1](tasks/part1.md) | Interface addressing |
| [Part 2](tasks/part2.md) | OSPF configuration |
| [Part 3](tasks/part3.md) | OSPF tuning & passive interfaces |
| [Verification](tasks/verify.md) | Checklist |
