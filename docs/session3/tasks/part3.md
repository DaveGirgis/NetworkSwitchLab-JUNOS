# Part 3 — OSPF Tuning & Passive Interfaces

## OSPF Metric (Cost)

By default, Junos OSPF assigns a cost of **1** to all interfaces regardless of bandwidth. In a real SP network, costs are tuned to reflect link capacity and influence traffic engineering.

### Set Interface Cost

Assign a lower cost to the P1–P2 link to simulate a higher-capacity path:

```junos
P1> configure

set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 metric 10

commit
```

```junos
P2> configure

set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 metric 10

commit
```

Verify the metric change:

```junos
PE1> show ospf route

Prefix             Path  Route      NH       Metric NextHop       Nexthop
                   Type  Type                        Interface     Address/LSP
10.0.0.1/32        Intra Router     IP            0 lo0.0
10.0.0.2/32        Intra Router     IP            1 ge-0/0/0.0    10.1.12.2
10.0.0.3/32        Intra Router     IP           11 ge-0/0/0.0    10.1.12.2
10.0.0.4/32        Intra Router     IP           12 ge-0/0/0.0    10.1.12.2
```

PE1 now shows cost 11 to reach P2 (1 to P1 + 10 on P1–P2 link).

## OSPF Hello & Dead Timers

Default timers: Hello=10s, Dead=40s. Adjust for faster convergence:

```junos
PE1> configure

set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 hello-interval 5
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 dead-interval 20

commit
```

!!! warning "Timer mismatch"
    OSPF neighbors will not form if the Hello and Dead intervals do not match on both ends of a link. If you change timers on PE1's `ge-0/0/0.0`, you must also change them on P1's `ge-0/0/0.0`. For this lab, leave timers at default values unless you want to experiment.

Restore defaults:

```junos
PE1> configure

delete protocols ospf area 0.0.0.0 interface ge-0/0/0.0 hello-interval
delete protocols ospf area 0.0.0.0 interface ge-0/0/0.0 dead-interval

commit
```

## Examine the OSPF LSDB

```junos
PE1> show ospf database

    OSPF database, Area 0.0.0.0
 Type       ID               Adv Rtr           Seq      Age  Opt  Cksum  Len
Router   10.0.0.1         10.0.0.1         0x80000004    92  0x22 0x9c1a  48
Router   10.0.0.2         10.0.0.2         0x80000005    91  0x22 0x7c29  60
Router   10.0.0.3         10.0.0.3         0x80000004    89  0x22 0x5c3a  60
Router   10.0.0.4         10.0.0.4         0x80000003    90  0x22 0x3c4b  48
```

All four **Router LSAs** (Type 1) appear. Each router generates one Router LSA describing its interfaces and neighbors.

!!! note "No Type 2 LSAs on p2p links"
    With `interface-type p2p` configured, OSPF does not elect a DR/BDR and does not generate Network LSAs (Type 2). This is the correct behavior for point-to-point links.

## Examine a Specific LSA

```junos
PE1> show ospf database router lsa-id 10.0.0.2 detail

  OSPF database, Area 0.0.0.0
 Type       ID               Adv Rtr           Seq      Age  Opt  Cksum  Len
Router   10.0.0.2         10.0.0.2         0x80000005   102  0x22 0x7c29  60
  bits 0x0, link count 4
  id 10.0.0.1, data 10.1.12.2, Type PointToPoint (1), metric 1
  id 10.1.12.0, data 255.255.255.252, Type Stub (3), metric 1
  id 10.0.0.3, data 10.1.23.1, Type PointToPoint (1), metric 10
  id 10.1.23.0, data 255.255.255.252, Type Stub (3), metric 10
```

P1's Router LSA describes:
- Two point-to-point links to PE1 and P2 (Type 1)
- Two stub networks (the /30 subnets) (Type 3)
- The loopback (10.0.0.2/32) appears as a Stub link in the LSA

## OSPF Summary

| Command | What it shows |
|---------|---------------|
| `show ospf neighbor` | Adjacency state, router ID, dead timer |
| `show ospf interface` | Interface state, area, DR/BDR, cost |
| `show ospf database` | All LSAs in the LSDB |
| `show ospf route` | OSPF-computed routes (before FIB install) |
| `show route protocol ospf` | OSPF routes in the routing table |
