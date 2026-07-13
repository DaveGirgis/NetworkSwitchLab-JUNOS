# Part 2 - Fault 2 (BGP)

## Symptom

With Fault 1 fixed and verified, the network runs cleanly for a while. Then a second ticket arrives:

> "VPN-A between CE1 and CE2 has stopped working. The Layer 2 service between them still works fine."

Start the same way - confirm the symptom precisely before touching anything.

```junos
CE1> ping 10.0.0.12 source 10.0.0.11 count 5
```

Observed:

```text
PING 10.0.0.12 (10.0.0.12): 56 data bytes

--- 10.0.0.12 ping statistics ---
5 packets transmitted, 0 packets received, 100% packet loss
```

```junos
CE1> ping 192.168.1.2 count 5
```

Observed - VPLS-100 is unaffected:

```text
PING 192.168.1.2 (192.168.1.2): 56 data bytes
64 bytes from 192.168.1.2: icmp_seq=0 ttl=64 time=33.912 ms
64 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=29.774 ms
64 bytes from 192.168.1.2: icmp_seq=2 ttl=64 time=41.203 ms
64 bytes from 192.168.1.2: icmp_seq=3 ttl=64 time=36.558 ms
64 bytes from 192.168.1.2: icmp_seq=4 ttl=64 time=30.116 ms

--- 192.168.1.2 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
```

This is the opposite pattern from Part 1. In Part 1, both services failed together - pointing at shared transport (IS-IS/LDP). Here, only VPN-A fails while VPLS-100 keeps working perfectly on the same physical path. That tells you the shared transport layer is healthy and the fault is specific to VPN-A's control plane.

## Step 1: Confirm the Transport Layer First (Do Not Skip This)

Even with a strong hypothesis, verify the layer below before assuming it is healthy - that is the discipline, not a suggestion.

```junos
PE1> show isis adjacency
```

Expected and observed - unchanged, healthy (both P1 adjacencies restored from Part 1's fix remain up, and P1-P2/P2-PE2 are healthy).

```junos
PE1> show route table inet.3
```

Expected and observed - three LDP routes present, PE2 loopback resolves with a Push label. Transport is confirmed healthy. The fault is above this layer.

## Step 2: Check the iBGP Session Between PE1 and PE2

```junos
PE1> show bgp summary
```

Observed:

```text
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.l2vpn.0
                       1          1          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.0.4              65001        128        124       0       2       00:04:12 Establ
  bgp.l2vpn.0: 1/1/1/0
  VPLS-100.l2vpn.0: 1/1/1/0
```

The session to PE2 (10.0.0.4) shows `Establ` - it is up. But look carefully at what tables are listed: only `bgp.l2vpn.0` and `VPLS-100.l2vpn.0`. Compare this against the Part 0 baseline, which showed `bgp.l3vpn.0` and `VPN-A.inet.0` as additional lines under the same peer. Those lines are gone.

**This is the key lesson of this fault: the BGP session itself is Established - it did not go down. A session-state check alone would tell you everything is fine.** The problem is which address families are negotiated over that session, not whether the session exists.

## Step 3: Check Negotiated NLRI Families

```junos
PE1> show bgp neighbor 10.0.0.4 | match NLRI
```

Observed:

```text
  NLRI for restart configured on peer: l2vpn
  NLRI advertised by peer: l2vpn
  NLRI for this session: l2vpn
  NLRI that restart is negotiated for: l2vpn
  NLRI of received end-of-rib markers: l2vpn
  NLRI of all end-of-rib markers sent: l2vpn
```

`inet-vpn-unicast` is completely absent from every NLRI line. In the Part 0 baseline this same command showed `inet-vpn-unicast l2vpn` on every line. Now it shows only `l2vpn`. The `inet-vpn unicast` address family has stopped being negotiated on this session.

## Step 4: Confirm on Both Sides

An address-family mismatch requires checking both PEs - if only one side lost the family, that side is where the fault lives.

```junos
PE1> show configuration protocols bgp group IBGP
```

Observed on PE1 - unchanged from Session 8:

```text
type internal;
local-address 10.0.0.1;
family inet-vpn {
    unicast;
}
family l2vpn {
    signaling;
}
neighbor 10.0.0.4;
```

PE1's configuration is intact - it still requests `inet-vpn unicast`. Check PE2:

```junos
PE2> show configuration protocols bgp group IBGP
```

Observed on PE2:

```text
type internal;
local-address 10.0.0.4;
family l2vpn {
    signaling;
}
neighbor 10.0.0.1;
```

**Root cause found.** PE2's iBGP group no longer has `family inet-vpn unicast` configured at all - only `family l2vpn signaling` remains. Since BGP address family negotiation requires both sides to advertise support for a family during the session's capability exchange, removing it from PE2 alone is enough to drop `inet-vpn-unicast` from the session entirely, even though PE1 never changed.

This explains every symptom observed:

- The BGP session stayed `Establ` - sessions do not go down just because one address family is removed; they simply stop negotiating that family and continue with whatever families both sides still agree on (here, `l2vpn`).
- VPLS-100 kept working, because it depends only on `l2vpn signaling`, which is untouched.
- VPN-A broke completely, because `bgp.l3vpn.0` has no way to receive VPN-IPv4 routes without the `inet-vpn unicast` family, so PE1 and PE2 stop exchanging VPN-A prefixes even though the VRFs themselves are still correctly configured.

## Step 5: Fix

Restore the missing address family on PE2's iBGP group.

```junos
PE2> configure

set protocols bgp group IBGP family inet-vpn unicast

commit
```

!!! warning "Do not touch PE1"
    PE1's configuration was never wrong. Adding or re-adding `family inet-vpn unicast` on PE1 changes nothing (it is idempotent) but wastes time and, more importantly, teaches the wrong lesson - always confirm which side actually changed before editing configuration on a router that was never broken.

## Step 6: Verify the Fix

Wait about 15-30 seconds for the session to renegotiate capabilities, then recheck.

```junos
PE1> show bgp neighbor 10.0.0.4 | match NLRI
```

Expected - `inet-vpn-unicast` restored on every line:

```text
  NLRI for restart configured on peer: inet-vpn-unicast l2vpn
  NLRI advertised by peer: inet-vpn-unicast l2vpn
  NLRI for this session: inet-vpn-unicast l2vpn
  NLRI that restart is negotiated for: inet-vpn-unicast l2vpn
  NLRI of received end-of-rib markers: inet-vpn-unicast l2vpn
  NLRI of all end-of-rib markers sent: inet-vpn-unicast l2vpn
```

```junos
PE1> show bgp summary
```

Expected - `bgp.l3vpn.0` and per-peer `VPN-A.inet.0` counts restored:

```text
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.l2vpn.0
                       1          1          0          0          0          0
bgp.l3vpn.0
                       2          2          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.0.4              65001        142        139       0       2       00:07:45 Establ
  bgp.l2vpn.0: 1/1/1/0
  VPLS-100.l2vpn.0: 1/1/1/0
  bgp.l3vpn.0: 2/2/2/0
  VPN-A.inet.0: 2/2/2/0
```

```junos
PE1> show route table VPN-A.inet.0
```

Expected - four routes present again, including CE2's loopback with the two-label stack:

```text
VPN-A.inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.11/32       *[BGP/170] 03:41:02, localpref 100
                      AS path: 65100 I, validation-state: unverified
                    > to 172.16.1.2 via ge-0/0/1.0
10.0.0.12/32       *[BGP/170] 00:00:18, localpref 100, from 10.0.0.4
                      AS path: 65100 I, validation-state: unverified
                    > to 10.1.12.2 via ge-0/0/0.0, Push <vpn-label>, Push 299808
172.16.1.0/30      *[Direct/0] 03:51:15
                    > via ge-0/0/1.0
172.16.2.0/30      *[BGP/170] 00:00:18, localpref 100, from 10.0.0.4
                    > to 10.1.12.2 via ge-0/0/0.0, Push <vpn-label>, Push 299808
```

Finally, confirm end-to-end reachability:

```junos
CE1> ping 10.0.0.12 source 10.0.0.11 count 5
```

Expected - 5/5 replies, 0% packet loss.

```junos
CE1> ping 192.168.1.2 count 5
```

Expected - 5/5 replies, 0% packet loss (should never have stopped working, but confirm it still hasn't regressed).

!!! tip "Why this fault was placed second"
    This fault only makes sense to diagnose once you trust the IGP is healthy - otherwise you cannot be confident the fault is in BGP and not a symptom of a lower-layer issue. It also teaches the most important lesson in BGP troubleshooting: a session showing `Establ` is necessary but not sufficient. You must check what address families are actually negotiated, not just whether the peer relationship exists.

Proceed to [Part 3 - Fault 3 (MPLS/L3VPN)](part3.md).
