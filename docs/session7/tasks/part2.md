# Part 2 — Label Distribution Verification

## Overview

With LDP sessions established, every provider router has exchanged label bindings for all prefixes in their routing tables. This part walks through the label tables and ends with the payoff: a CE-to-CE ping that was impossible before MPLS.

## Step 1: Inspect the LDP Label Database

The **LDP database** shows the label bindings exchanged between two LDP peers. On PE1, there is one LDP session (to P1). The database shows what labels PE1 received from P1 (input) and what labels PE1 sent to P1 (output).

```junos
PE1> show ldp database
```

Expected output:

```text
Input label database, 10.0.0.1:0--10.0.0.2:0
  Label     Prefix
 299776      10.0.0.1/32
      3      10.0.0.2/32
 299792      10.0.0.3/32
 299808      10.0.0.4/32

Output label database, 10.0.0.1:0--10.0.0.2:0
  Label     Prefix
      3      10.0.0.1/32
 299776      10.0.0.2/32
 299792      10.0.0.3/32
 299808      10.0.0.4/32
```

LDP only distributes labels for loopback (host) prefixes — the /30 link subnets do not appear.

**Input label database** — labels PE1 received from P1. These are the labels PE1 will use when pushing traffic toward P1:
- Label `299776` for 10.0.0.1/32 — P1 assigns this label for traffic it needs to send back to PE1's loopback. PE1 uses it when traffic loops back, but more relevantly P1 advertises it upstream.
- Label `3` for 10.0.0.2/32 — PHP in action. P1 sends implicit-null (label 3) for its own loopback, telling PE1 not to push a label for traffic destined to P1's loopback. PE1 forwards those packets to P1 as plain IP.
- Labels `299792` and `299808` for 10.0.0.3/32 and 10.0.0.4/32 — PE1 pushes these labels when forwarding traffic toward P2 or PE2 via P1.

**Output label database** — labels PE1 sent to P1. P1 uses these when sending traffic to PE1:
- Label `3` for 10.0.0.1/32 — PE1 sends implicit-null for its own loopback. PHP: P1 pops before delivering to PE1. PE1 receives plain IP for traffic destined to itself.
- Labels `299776`, `299792`, `299808` for the remaining loopbacks — PE1's label assignments for prefixes P1 might forward to PE1 as a transit hop.

## Step 2: Verify inet.3

The **inet.3** routing table holds MPLS-resolved next-hops computed by LDP. BGP uses inet.3 to resolve its protocol next-hops — this is the table that makes MPLS transparent to BGP.

```junos
PE1> show route table inet.3
```

Expected:

```text
inet.3: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32        *[LDP/9] 00:03:21, metric 1
                    > to 10.1.12.2 via ge-0/0/0.0
10.0.0.3/32        *[LDP/9] 00:03:21, metric 2
                    > to 10.1.12.2 via ge-0/0/0.0, Push 299792
10.0.0.4/32        *[LDP/9] 00:03:21, metric 3
                    > to 10.1.12.2 via ge-0/0/0.0, Push 299808
```

Three entries — one for each provider loopback that PE1 can reach via LDP:

- **10.0.0.2/32 (P1)** — directly adjacent; PHP applies. P1 sent implicit-null, so no label is pushed. Traffic arrives at P1 as a plain IP packet.
- **10.0.0.3/32 (P2)** — two hops. PE1 pushes label 299777. P1 receives it, swaps or pops (PHP from P2), and forwards toward P2.
- **10.0.0.4/32 (PE2)** — three hops. PE1 pushes label 299808. P1 swaps, P2 pops (PHP, as P2 is penultimate for PE2), PE2 receives a plain IP packet.

The preference value is **LDP/9** — LDP routes in inet.3 have a preference of 9, which is lower (more preferred) than IS-IS (18), ensuring LDP routes are used when resolving BGP next-hops.

!!! note "inet.3 is separate from inet.0"
    BGP next-hops are resolved in inet.3 by default. IS-IS and other IGP routes only appear in inet.0. This separation allows BGP to use the MPLS LSP (label path) while non-BGP traffic continues to use the normal IP path in inet.0.

## Step 3: Verify BGP Route Resolution with MPLS

Before MPLS, the BGP route to CE2's loopback on PE1 had no label — P routers would receive an IP packet destined for 10.0.0.12 and drop it. Now, check the route detail:

```junos
PE1> show route 10.0.0.12 detail
```

Expected (key lines highlighted):

```text
inet.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
10.0.0.12/32 (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Next hop type: Indirect
                Next-hop reference count: 1
                Source: 10.0.0.4
                Next hop: 10.1.12.2 via ge-0/0/0.0, selected
                Label-stack { 299808 }; MPLS-label: Push 299808
                Protocol next hop: 10.0.0.4
                Indirect next hop: 0x93474a0 299808 INH Session ID: 0x150001
                State: <Active Ext>
                Age: 4:07      Metric2: 3
                Task: BGP_65001.10.0.0.4+179
                AS path: 65100 I
                Localpref: 100
                Router ID: 10.0.0.4
```

The critical line is `Label-stack { 299808 }; MPLS-label: Push 299808`. This confirms:

- BGP found the route to CE2's loopback (10.0.0.12) via iBGP from PE2 (10.0.0.4)
- BGP resolved the next-hop 10.0.0.4 using inet.3
- inet.3 says: push label 299808 to reach 10.0.0.4 via P1 (10.1.12.2)
- When PE1 forwards this packet, it will push label 299808 before handing off to P1

P1 and P2 will swap/pop the label without ever looking at the 10.0.0.12 destination IP.

## Step 4: CE-to-CE Ping

This is the payoff of Session 7. A ping from CE1 to CE2's loopback was failing at the end of Session 6 because P routers had no route to CE prefixes. With LDP running, the label path carries the packet end-to-end.

```junos
CE1> ping 10.0.0.12 count 5
```

Expected — all five replies succeed:

```text
PING 10.0.0.12 (10.0.0.12): 56 data bytes
64 bytes from 10.0.0.12: icmp_seq=0 ttl=60 time=3.2 ms
64 bytes from 10.0.0.12: icmp_seq=1 ttl=60 time=2.9 ms
64 bytes from 10.0.0.12: icmp_seq=2 ttl=60 time=3.1 ms
64 bytes from 10.0.0.12: icmp_seq=3 ttl=60 time=2.8 ms
64 bytes from 10.0.0.12: icmp_seq=4 ttl=60 time=3.0 ms

--- 10.0.0.12 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.8/3.0/3.2/0.1 ms
```

The TTL of 60 reflects packet traversal across multiple hops (started at 64, decremented by each router). What is happening end-to-end:

1. CE1 sends an ICMP packet destined for 10.0.0.12 toward its BGP next-hop PE1 (172.16.1.1)
2. PE1 has a BGP route to 10.0.0.12 resolved via inet.3 — pushes label 299808 and forwards to P1
3. P1 swaps label 299808 with its outgoing label for PE2 and forwards to P2
4. P2 performs PHP (pops the label) and forwards the bare IP packet to PE2
5. PE2 receives an IP packet, looks up 10.0.0.12 in its BGP table, and forwards out ge-0/0/1 to CE2
6. CE2 replies to CE1's IP (10.0.0.11 or 172.16.1.2), path reverses through the same LSP

!!! note "TTL propagation through MPLS"
    By default, Junos copies the IP TTL into the MPLS TTL field when pushing a label (RFC 3032 behavior). Each label swap decrements the MPLS TTL. When the label is popped, the remaining TTL is written back into the IP header. This means TTL-based traceroute will show P1 and P2 in the path as expected.

Also verify the reverse direction:

```junos
CE2> ping 10.0.0.11 count 5
```

Expected: same pattern, all five replies succeed.

## Step 5: View the MPLS Forwarding Table

The **mpls.0** table shows the local LFIB — what happens to each incoming labeled packet on this router.

```junos
P1> show route table mpls.0
```

Expected on P1 (a transit LSR):

```text
mpls.0: 6 destinations, 6 routes (6 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0                  *[MPLS/0] 2d 04:12:00, metric 1
                      Receive
1                  *[MPLS/0] 2d 04:12:00, metric 1
                      Receive
2                  *[MPLS/0] 2d 04:12:00, metric 1
                      Receive
299776             *[LDP/9] 00:08:14, metric 1
                    > to 10.1.12.1 via ge-0/0/0.0, Pop
299792             *[LDP/9] 00:08:14, metric 1
                    > to 10.1.23.2 via ge-0/0/1.0, Pop
299808             *[LDP/9] 00:08:14, metric 1
                    > to 10.1.23.2 via ge-0/0/1.0, Pop
```

Labels 0, 1, 2 are reserved (IPv4 Explicit Null, Router Alert, IPv6 Explicit Null) — any packet arriving with these labels is consumed locally.

The three LDP entries correspond to labels P1 assigned and advertised to its neighbors:
- **299776** (PE1's loopback) — traffic arriving labeled 299776 is headed for PE1. P1 pops (PHP — PE1 sent implicit-null for its own loopback) and forwards bare IP to PE1 out ge-0/0/0.0.
- **299792** (P2's loopback) — traffic arriving labeled 299792 from PE1 is headed for P2. P1 pops (PHP — P2 sent implicit-null for its own loopback to P1) and forwards bare IP to P2 out ge-0/0/1.0.
- **299808** (PE2's loopback) — traffic arriving labeled 299808 from PE1 is headed for PE2. P1 pops (PHP — P2 sent implicit-null for PE2's loopback, as P2 is the penultimate hop for PE2) and forwards bare IP to P2 out ge-0/0/1.0.

In this four-router linear topology, PHP removes the label at every hop — P1 never needs to Swap. P2 receives bare IP packets from P1 and makes the final IP forwarding decision. The key point holds: P1 makes every forwarding decision based on the incoming label, not the IP destination.
