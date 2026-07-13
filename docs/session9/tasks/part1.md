# Part 1 - Fault 1 (IGP)

## Symptom

A ticket arrives (from your instructor, or is self-reported after Part 0's injection): "Something in the core broke. CE1 cannot reach CE2 anymore." That is the entire symptom report - exactly as vague as a real Tier-1 ticket. Your job is to turn that into a root cause.

Start testing from CE1:

```junos
CE1> ping 10.0.0.12 source 10.0.0.11 count 5
```

Observed:

```text
PING 10.0.0.12 (10.0.0.12): 56 data bytes

--- 10.0.0.12 ping statistics ---
5 packets transmitted, 0 packets received, 100% packet loss
```

100% loss. VPN-A appears completely broken. Before touching BGP or VPN configuration, apply the methodology from the session index: **start at the bottom of the stack.**

## Step 1: Rule Out the VPLS-100 Control Group

Because VPLS-100 rides the same IS-IS/LDP transport but has a completely independent control plane (BGP `l2vpn signaling` vs. VPN-A's VRF), testing it tells you whether the fault is specific to VPN-A or shared across the whole transport.

```junos
CE1> ping 192.168.1.2 count 5
```

Observed:

```text
PING 192.168.1.2 (192.168.1.2): 56 data bytes

--- 192.168.1.2 ping statistics ---
5 packets transmitted, 0 packets received, 100% packet loss
```

VPLS-100 is also down. Both services fail identically. Both depend on the same LDP transport, which depends on IS-IS. This points at a shared lower layer, not a VPN-specific misconfiguration - exactly the kind of clue that should stop you from opening BGP or VRF configuration first.

## Step 2: Check Interfaces Before Protocols

```junos
PE1> show interfaces terse | match ge-
```

Expected and observed - all local interfaces show `up up`. No cabling or physical fault on PE1.

Repeat the same check quickly on P1, P2, and PE2. All interfaces report up. This rules out a physical-layer or GNS3-link fault - the problem is in a routing protocol, not a wire.

## Step 3: Check IS-IS Adjacencies on Every Provider Router

This is the first real diagnostic step, and it should be your first stop whenever a symptom spans multiple independent services.

```junos
PE1> show isis adjacency
```

Expected and observed - unchanged, still Up:

```text
Interface             System         L State        Hold (secs) SNPA
ge-0/0/0.0            P1             2  Up                   23
```

```junos
P1> show isis adjacency
```

Observed - only one adjacency instead of the expected two:

```text
Interface             System         L State        Hold (secs) SNPA
ge-0/0/0.0            PE1            2  Up                   21
```

The P1-P2 adjacency is missing entirely. Compare against P2:

```junos
P2> show isis adjacency
```

Observed - P2 also shows only one adjacency (PE2), not two:

```text
Interface             System         L State        Hold (secs) SNPA
ge-0/0/1.0            PE2            2  Up                   17
```

The IS-IS adjacency between P1 and P2 is down. This is your isolation point: the fault is on the P1-P2 link, at the IS-IS layer.

## Step 4: Narrow the Cause on the P1-P2 Link

The interface is physically up (confirmed in Step 2), so the adjacency failure is a protocol-level mismatch, not a wire problem. Check IS-IS interface-level detail on both sides.

```junos
P1> show isis interface ge-0/0/1.0 detail
```

Observed:

```text
IS-IS interface database:

ge-0/0/1.0
  Index: 3, State: 0x6, Circuit id: 0x2, Circuit type: 2
  LSP interval: 100 ms, CSNP interval: 10 s, Oper interface type: point-to-point
  Level1
    Adjacencies: 0
  Level2
    Adjacencies: 0
    Metric: 10, Priority: 64, Internal-Only
    Hello: 9 s, Hold: 27 s, Wide-Metric: TRUE
```

Zero adjacencies at both levels - nothing is inherently wrong with this snippet alone, so compare the same command on P2:

```junos
P2> show isis interface ge-0/0/0.0 detail
```

Observed:

```text
IS-IS interface database:

ge-0/0/0.0
  Index: 3, State: 0x6, Circuit id: 0x1, Circuit type: 1
  LSP interval: 100 ms, CSNP interval: 10 s, Oper interface type: point-to-point
  Level1
    Adjacencies: 0
    Metric: 10, Priority: 64, Internal-Only
  Level2
    Adjacencies: 0
```

Compare `Circuit type` on the two sides: P1's ge-0/0/1.0 shows **Circuit type: 2** (Level 2 only). P2's ge-0/0/0.0 shows **Circuit type: 1** (Level 1 only). This is the mismatch. IS-IS adjacencies form only when both sides share at least one common level. P1 is only running Level 2 on this interface; P2 is only running Level 1. There is no common level, so no adjacency forms - and Junos does not raise an alarm for this the way it would for a down interface. The link looks perfectly healthy at every layer below IS-IS.

Confirm by checking the actual configuration on P2 - this is where the fault was injected:

```junos
P2> show configuration protocols isis
```

Observed:

```text
level 1 disable;
interface ge-0/0/0.0 {
    level 1 enable;
    level 2 disable;
}
interface ge-0/0/1.0;
interface lo0.0;
```

**Root cause:** P2's `ge-0/0/0.0` interface (facing P1) has been explicitly configured with `level 1 enable` and `level 2 disable`. Every other interface and every other router in this topology runs Level 2 only (the standard baseline established in Session 5). This single interface was pinned to Level 1, which no other router in the topology runs - guaranteeing no adjacency can ever form on this link.

## Step 5: Fix

Remove the Level 1 override on P2's ge-0/0/0.0 interface so it falls back to the router-wide Level 2 default.

```junos
P2> configure

delete protocols isis interface ge-0/0/0.0 level 1
delete protocols isis interface ge-0/0/0.0 level 2

commit
```

!!! note "Why delete both lines"
    `level 1 enable` and `level 2 disable` are two separate configuration statements under the same interface stanza. Deleting only the `level 1 enable` line would leave `level 2 disable` in place and the interface would run no IS-IS level at all - still broken. Remove both so the interface inherits the router-wide `level 1 disable` / Level 2 default already configured under `protocols isis` (visible in Step 4's output as `level 1 disable;` at the top of the stanza).

## Step 6: Verify the Fix

```junos
P1> show isis adjacency
```

Expected - two adjacencies restored:

```text
Interface             System         L State        Hold (secs) SNPA
ge-0/0/0.0            PE1            2  Up                   24
ge-0/0/1.0            P2             2  Up                    9
```

```junos
P2> show isis adjacency
```

Expected - two adjacencies restored:

```text
Interface             System         L State        Hold (secs) SNPA
ge-0/0/0.0            P1             2  Up                    8
ge-0/0/1.0            PE2            2  Up                   22
```

Confirm LDP re-established across the now-healthy adjacency:

```junos
P1> show ldp session
```

Expected - both sessions Operational:

```text
Address                          State       Connection  Hold time  Adv. Mode
10.0.0.1                         Operational Open          25        DU
10.0.0.3                         Operational Open          19        DU
```

Confirm inet.3 on PE1 now resolves all three provider loopbacks:

```junos
PE1> show route table inet.3
```

Expected:

```text
inet.3: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32        *[LDP/9] 00:01:14, metric 1
                    > to 10.1.12.2 via ge-0/0/0.0
10.0.0.3/32        *[LDP/9] 00:00:52, metric 2
                    > to 10.1.12.2 via ge-0/0/0.0, Push 299792
10.0.0.4/32        *[LDP/9] 00:00:52, metric 3
                    > to 10.1.12.2 via ge-0/0/0.0, Push 299808
```

Finally, confirm both services are restored end-to-end:

```junos
CE1> ping 10.0.0.12 source 10.0.0.11 count 5
```

Expected - 5/5 replies, 0% packet loss.

```junos
CE1> ping 192.168.1.2 count 5
```

Expected - 5/5 replies, 0% packet loss.

Both VPN-A and VPLS-100 recover from the same single IS-IS fix - confirming the root cause was shared transport, not either service's individual control plane.

!!! tip "Why this fault was placed first"
    IS-IS is the foundation every other protocol in this stack depends on for loopback reachability. Breaking it first, and confirming it produces symptoms in both VPN-A and VPLS-100 simultaneously, reinforces the layering lesson before Parts 2 and 3 introduce faults that look similar on the surface (CE1 cannot reach CE2) but live entirely above a healthy IGP.

Proceed to [Part 2 - Fault 2 (BGP)](part2.md).
