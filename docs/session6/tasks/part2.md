# Part 2 — iBGP PE-PE

## Overview

iBGP connects PE1 and PE2 within AS 65001 so that customer prefixes learned on one PE can be shared with the other. Key points before configuring:

- iBGP peers use **loopback addresses** as source and neighbor, not physical interface IPs. IS-IS already provides reachability between the loopbacks.
- A next-hop-self export policy is required: without it, PE2 would see CE1's address (172.16.1.2) as the NEXT_HOP for CE1's prefix — an address PE2 cannot reach.
- P1 and P2 are **not** iBGP peers. They are IS-IS transit routers only.

!!! note "next-hop-self on vMX 14.1"
    Junos 14.1 does not support `next-hop-self` as a direct BGP neighbor statement. Instead, a routing policy with `then next-hop self` achieves the same result and is applied as an export policy on the iBGP group.

## Step 1: Configure iBGP on PE1

```junos
configure

set protocols bgp group IBGP type internal
set protocols bgp group IBGP local-address 10.0.0.1
set protocols bgp group IBGP neighbor 10.0.0.4
set policy-options policy-statement NEXT-HOP-SELF term 1 then next-hop self
set policy-options policy-statement NEXT-HOP-SELF term 1 then accept
set protocols bgp group IBGP export NEXT-HOP-SELF

commit
```

## Step 2: Configure iBGP on PE2

```junos
configure

set protocols bgp group IBGP type internal
set protocols bgp group IBGP local-address 10.0.0.4
set protocols bgp group IBGP neighbor 10.0.0.1
set policy-options policy-statement NEXT-HOP-SELF term 1 then next-hop self
set policy-options policy-statement NEXT-HOP-SELF term 1 then accept
set protocols bgp group IBGP export NEXT-HOP-SELF

commit
```

## Step 3: Verify iBGP Session

```junos
PE1> show bgp summary
```

Expected — both eBGP (CE1) and iBGP (PE2) sessions established:

```text
Groups: 2 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0                 2          2          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.16.1.2            65100         12         11       0       0       3:24 Establ
  inet.0: 1/1/1/0
10.0.0.4              65001          8          8       0       0       1:47 Establ
  inet.0: 1/1/1/0
```

The iBGP peer (10.0.0.4) shows `1/1/1/0` — one prefix received (CE2's loopback from PE2).

## Step 4: Verify Prefix Propagation

PE1 should now have CE2's loopback via iBGP from PE2, and vice versa.

```junos
PE1> show route receive-protocol bgp 10.0.0.4
```

Expected — CE2's prefix received from PE2 via iBGP:

```text
inet.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.0.0.12/32            10.0.0.4                    100        65100 I

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
```

The next-hop is PE2's loopback (10.0.0.4) — `next-hop-self` replaced the original CE2 address. `Lclpref 100` is the default LOCAL_PREF applied to routes entering the AS via iBGP.

```junos
PE2> show route receive-protocol bgp 10.0.0.1
```

Expected — CE1's prefix received from PE1 via iBGP:

```text
inet.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.0.0.11/32            10.0.0.1                    100        65100 I

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
```

## Step 5: Verify CE Routers Receive Each Other's Prefix

PE2 will now advertise CE1's prefix (10.0.0.11/32) to CE2 via eBGP, and PE1 will advertise CE2's prefix (10.0.0.12/32) to CE1.

```junos
CE1> show route receive-protocol bgp 172.16.1.1
```

Expected — CE2's loopback received via PE1:

```text
inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.0.0.12/32            172.16.1.1                              65001 65100 I
```

Note the AS_PATH: `65001 65100` — the route passed through the provider AS (65001) before originating in AS 65100 (CE2). CE routers do not run IS-IS so no `iso.0` table appears.

```junos
CE2> show route receive-protocol bgp 172.16.2.1
```

Expected — CE1's loopback received via PE2:

```text
inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.0.0.11/32            172.16.2.1                              65001 65100 I
```

## Step 6: Confirm the Full BGP Table on PE1

```junos
PE1> show route protocol bgp
```

Expected — two BGP routes, one from each direction:

```text
inet.0: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)

10.0.0.11/32       *[BGP/170] 00:04:15, localpref 100
                    AS path: 65100 I, validation-state: unverified
                    > to 172.16.1.2 via ge-0/0/1.0
10.0.0.12/32       *[BGP/170] 00:01:30, localpref 100
                    AS path: 65100 I, validation-state: unverified
                    > to 10.1.12.2 via ge-0/0/0.0
```

10.0.0.11/32 was learned from CE1 directly (eBGP, out ge-0/0/1). 10.0.0.12/32 was learned from PE2 (iBGP, next-hop resolved via IS-IS out ge-0/0/0 toward P1).

!!! note "Why CE-to-CE ping fails without MPLS"
    CE1 has a route to CE2's loopback (10.0.0.12/32) via PE1. CE2 has a route to CE1's loopback via PE2. The BGP knowledge is complete. However, when a packet from CE1 to 10.0.0.12 reaches PE1, PE1 forwards it toward PE2 via IS-IS — the packet passes through P1 and P2. Those P routers do not have a route to 10.0.0.12 and will drop it. Session 7 (MPLS) solves this by having P routers forward on labels rather than destination IPs.
