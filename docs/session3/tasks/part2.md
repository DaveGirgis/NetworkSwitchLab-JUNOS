# Part 2 — OSPF Configuration

## OSPF in Junos — Overview

In Junos, OSPF is configured under `[edit protocols ospf]`. You add interfaces to OSPF areas; OSPF hellos start automatically on those interfaces.

The key structural difference from Cisco IOS: Junos OSPF uses **interface-based** configuration (you specify which interfaces belong to which area), whereas IOS uses `network` statements with wildcard masks.

## PE1 — OSPF

```junos
PE1> configure

set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface lo0.0 passive

commit
```

## P1 — OSPF

```junos
P1> configure

set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface lo0.0 passive

commit
```

## P2 — OSPF

```junos
P2> configure

set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface lo0.0 passive

commit
```

## PE2 — OSPF

```junos
PE2> configure

set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface lo0.0 passive

commit
```

## Explanation of Key Options

**`interface-type p2p`** — tells OSPF this is a point-to-point link. On a p2p link:
- OSPF skips DR/BDR election (only two routers)
- Adjacency forms faster
- No Type 2 (Network) LSAs are generated

Without `interface-type p2p`, Junos defaults to **broadcast** mode and attempts DR/BDR election — which works but is slower and generates unnecessary LSAs on /30 links.

**`passive`** on `lo0.0` — OSPF will advertise the loopback prefix into the LSDB but will NOT send Hello packets on this interface. This is correct behavior: a loopback has no neighbors, so sending Hellos wastes resources.

!!! warning "Passive interface behavior"
    `passive` in Junos suppresses OSPF Hellos on an interface while still advertising its prefix. If you accidentally mark a transit interface (like `ge-0/0/0.0`) as `passive`, OSPF adjacency will never form on that link.

## Verify OSPF Neighbors

Wait ~30 seconds after committing, then:

```junos
PE1> show ospf neighbor
Address          Interface              State     ID               Pri  Dead
10.1.12.2        ge-0/0/0.0             Full      10.0.0.2         128    37
```

```junos
P1> show ospf neighbor
Address          Interface              State     ID               Pri  Dead
10.1.12.1        ge-0/0/0.0             Full      10.0.0.1         128    37
10.1.23.2        ge-0/0/1.0             Full      10.0.0.3         128    39
```

**State `Full`** means the adjacency is fully established and LSDBs are synchronized. Any other state (Init, ExStart, Exchange, Loading) indicates a problem — see the Troubleshooting guide.

## Verify OSPF Routes in the Routing Table

```junos
PE1> show route protocol ospf

inet.0: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32        *[OSPF/10] 00:02:14, metric 1
                    > to 10.1.12.2 via ge-0/0/0.0
10.0.0.3/32        *[OSPF/10] 00:02:14, metric 2
                    > to 10.1.12.2 via ge-0/0/0.0
10.0.0.4/32        *[OSPF/10] 00:02:14, metric 3
                    > to 10.1.12.2 via ge-0/0/0.0
10.1.23.0/30       *[OSPF/10] 00:02:14, metric 2
                    > to 10.1.12.2 via ge-0/0/0.0
10.1.34.0/30       *[OSPF/10] 00:02:14, metric 3
                    > to 10.1.12.2 via ge-0/0/0.0
224.0.0.5/32       *[OSPF/10] 00:02:30, metric 1
                      MultiRecv
```

PE1 can now reach all four loopbacks via OSPF. The `metric` values reflect hop count (each link adds 1 by default — see Part 3 for tuning).

## Verify End-to-End Loopback Reachability

```junos
PE1> ping 10.0.0.4 count 3
PING 10.0.0.4 (10.0.0.4): 56 data bytes
64 bytes from 10.0.0.4: icmp_seq=0 ttl=62 time=3.211 ms
64 bytes from 10.0.0.4: icmp_seq=1 ttl=62 time=2.987 ms
64 bytes from 10.0.0.4: icmp_seq=2 ttl=62 time=3.102 ms
```

PE1 can reach PE2's loopback across the full provider backbone. TTL=62 (started at 64, decremented by P1 and P2).
