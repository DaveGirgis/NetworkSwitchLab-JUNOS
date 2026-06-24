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

Expected:

```text
inet.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
10.0.0.12/32 (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Next hop type: Indirect
                Address: 0x96f0568
                Next-hop reference count: 3
                Source: 10.0.0.4
                Next hop type: Router, Next hop index: 569
                Next hop: 10.1.12.2 via ge-0/0/0.0, selected
                Label operation: Push 299808
                Label TTL action: prop-ttl
                Load balance label: Label 299808: None;
                Session Id: 0x140
                Protocol next hop: 10.0.0.4
                Indirect next hop: 0x9650110 1048574 INH Session ID: 0x143
                State: <Active Int Ext>
                Local AS: 65001 Peer AS: 65001
                Age: 22:22:05   Metric2: 1
                Validation State: unverified
                Task: BGP_65001.10.0.0.4+59859
                Announcement bits (3): 0-KRT 3-BGP_RT_Background 4-Resolve tree 4
                AS path: 65100 I
                Accepted
                Localpref: 100
                Router ID: 10.0.0.4
```

The critical lines are `Label operation: Push 299808` and `State: <Active Int Ext>`. Together they confirm:

- **`Label operation: Push 299808`** — PE1 will push label 299808 onto every packet headed for 10.0.0.12 before handing it to P1. P routers forward on the label without looking at the 10.0.0.12 destination IP.
- **`State: <Active Int Ext>`** — `Int` means the BGP next-hop (10.0.0.4) was resolved through inet.3 (the MPLS/LDP table), not inet.0. `Ext` means this is an external route (received via iBGP). Before MPLS was enabled, `Int` would not appear here.
- **`Protocol next hop: 10.0.0.4`** — the iBGP next-hop is PE2's loopback, which inet.3 maps to the label path.
- **`Label TTL action: prop-ttl`** — IP TTL is copied into the MPLS label TTL on push, so traceroute sees each hop through the MPLS domain.

## Step 4: CE-to-CE Ping

This is the payoff of Session 7. A ping from CE1 to CE2's loopback was failing at the end of Session 6 because P routers had no route to CE prefixes. With LDP running, the label path carries the packet end-to-end.

!!! warning "Always specify the loopback as source"
    The BGP export policies in this lab only advertise loopback prefixes (10.0.0.11/32 and 10.0.0.12/32), not the PE-CE link subnets. If you run `ping 10.0.0.12` without a source, Junos uses the outgoing interface address (172.16.1.2). CE2 receives the ping but has no route to 172.16.1.2 and drops the reply — 100% loss. Always use `source 10.0.0.11` so the reply travels back over the BGP-learned path.

```junos
CE1> ping 10.0.0.12 source 10.0.0.11 count 5
```

Expected — all five replies succeed:

```text
PING 10.0.0.12 (10.0.0.12): 56 data bytes
64 bytes from 10.0.0.12: icmp_seq=0 ttl=60 time=28.351 ms
64 bytes from 10.0.0.12: icmp_seq=1 ttl=60 time=47.681 ms
64 bytes from 10.0.0.12: icmp_seq=2 ttl=60 time=23.684 ms
64 bytes from 10.0.0.12: icmp_seq=3 ttl=60 time=74.369 ms
64 bytes from 10.0.0.12: icmp_seq=4 ttl=60 time=13.464 ms

--- 10.0.0.12 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 13.464/37.510/74.369/21.519 ms
```

What is happening end-to-end:

1. CE1 sends an ICMP packet sourced from 10.0.0.11 (loopback), destined for 10.0.0.12, toward PE1 (172.16.1.1)
2. PE1 has a BGP route to 10.0.0.12 resolved via inet.3 — pushes label 299808 and forwards to P1
3. P1 swaps label 299808 with its outgoing label for FEC 10.0.0.4 and forwards to P2
4. P2 performs PHP (pops the label) and forwards the bare IP packet to PE2
5. PE2 receives the IP packet, looks up 10.0.0.12 in its BGP table, and forwards out ge-0/0/1 to CE2
6. CE2 sends an ICMP reply to 10.0.0.11 (CE1's loopback) — CE2 has a BGP route to 10.0.0.11 via PE2
7. The reply path reverses: CE2 → PE2 (push label for FEC 10.0.0.1) → P2 (swap) → P1 (PHP pop) → PE1 → CE1

!!! note "TTL propagation through MPLS"
    The `Label TTL action: prop-ttl` seen in Step 3 means Junos copies the IP TTL into the MPLS TTL on push. Each label swap decrements it. When the label is popped, the remaining TTL is written back into the IP header — traceroute shows P1 and P2 as hops through the MPLS domain.

Verify the reverse direction:

```junos
CE2> ping 10.0.0.11 source 10.0.0.12 count 5
```

Expected: same pattern, all five replies succeed.

## Step 5: View the MPLS Forwarding Table

The **mpls.0** table shows the local LFIB — what happens to each incoming labeled packet on this router.

```junos
P1> show route table mpls.0
```

Expected on P1 (a transit LSR):

```text
mpls.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0                  *[MPLS/0] 00:24:03, metric 1
                      to table inet.0
0(S=0)             *[MPLS/0] 00:24:03, metric 1
                      to table mpls.0
1                  *[MPLS/0] 00:24:03, metric 1
                      Receive
2                  *[MPLS/0] 00:24:03, metric 1
                      to table inet6.0
2(S=0)             *[MPLS/0] 00:24:03, metric 1
                      to table mpls.0
13                 *[MPLS/0] 00:24:03, metric 1
                      Receive
299776             *[LDP/9] 00:24:02, metric 1
                    > to 10.1.12.1 via ge-0/0/0.0, Pop
299776(S=0)        *[LDP/9] 00:24:02, metric 1
                    > to 10.1.12.1 via ge-0/0/0.0, Pop
299792             *[LDP/9] 00:23:34, metric 1
                    > to 10.1.23.2 via ge-0/0/1.0, Pop
299792(S=0)        *[LDP/9] 00:23:34, metric 1
                    > to 10.1.23.2 via ge-0/0/1.0, Pop
299808             *[LDP/9] 00:23:07, metric 1
                    > to 10.1.23.2 via ge-0/0/1.0, Swap 299808
```

**Reserved label entries (labels 0, 1, 2, 13):**

- **Label 0** (`to table inet.0`) — IPv4 Explicit Null. A packet arriving with label 0 is forwarded into the IPv4 routing table for normal IP lookup. Not the same as Receive — the packet is re-routed, not consumed.
- **Label 0(S=0)** (`to table mpls.0`) — same label but with the bottom-of-stack bit clear, meaning more labels exist below. Forwarded back into mpls.0 to process the next label in the stack.
- **Label 1** (Receive) — Router Alert. Consumed locally.
- **Label 2** (`to table inet6.0`) — IPv6 Explicit Null. Forwarded into the IPv6 routing table.
- **Label 2(S=0)** (`to table mpls.0`) — stacked variant of IPv6 Explicit Null.
- **Label 13** (Receive) — Generic Associated Channel (G-ACh), used for MPLS OAM. Consumed locally.

**`(S=0)` entries** — in MPLS, the S-bit (bottom-of-stack flag) in a label indicates whether this is the innermost label. S=1 means bottom of stack (single label or innermost). S=0 means more labels exist below this one (label stack). Junos installs separate LFIB entries for the S=0 case so it can apply the correct action when a label stack is in use — relevant for L3 VPN in Session 8 where an inner VPN label sits below the transport label.

**LDP label entries:**

- **299776 / 299776(S=0)** (PE1's loopback) — P1 pops and forwards bare IP to PE1 out ge-0/0/0.0. PHP applies: PE1 sent implicit-null for its own loopback, so P1 strips the label before delivering to PE1.
- **299792 / 299792(S=0)** (P2's loopback) — P1 pops and forwards bare IP to P2 out ge-0/0/1.0. PHP applies: P2 sent implicit-null for its own loopback, so P1 strips the label. P2 receives at its own address.
- **299808** (PE2's loopback) — P1 **swaps** to outgoing label 299808 and forwards to P2. Note the outgoing label is also 299808 — P2 independently assigned 299808 for the same FEC. Labels are locally significant; two routers assigning the same numeric value for different purposes is a coincidence. P2 will receive a packet labeled 299808 and pop it (PHP — PE2 sent implicit-null to P2 for PE2's own loopback), delivering bare IP to PE2.
