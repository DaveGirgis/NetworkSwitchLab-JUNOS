# Part 3 - Fault 3 (MPLS/L3VPN)

## Symptom

With Faults 1 and 2 both fixed and verified, the network runs cleanly again. A third ticket arrives - and this time it is worded almost identically to Part 2's:

> "VPN-A between CE1 and CE2 is down again. The Layer 2 service is fine."

Do not assume it is the same fault repeating. Confirm the symptom and start the diagnostic sequence from the bottom again - that repetition is deliberate. Real tickets rarely tell you which layer broke, and a good habit does not skip steps just because the last fault happened to be found at a particular layer.

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

Observed - VPLS-100 still fine:

```text
PING 192.168.1.2 (192.168.1.2): 56 data bytes
64 bytes from 192.168.1.2: icmp_seq=0 ttl=64 time=27.845 ms
64 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=34.902 ms
64 bytes from 192.168.1.2: icmp_seq=2 ttl=64 time=31.117 ms
64 bytes from 192.168.1.2: icmp_seq=3 ttl=64 time=29.660 ms
64 bytes from 192.168.1.2: icmp_seq=4 ttl=64 time=38.244 ms

--- 192.168.1.2 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
```

Same surface symptom as Part 2. This is the point of the exercise - you cannot tell from the ping alone whether this is another BGP address-family problem, a VRF problem, or something else entirely. Work the layers in order.

## Step 1: Confirm IS-IS and LDP (Still Healthy)

```junos
PE1> show isis adjacency
PE1> show route table inet.3
```

Expected and observed - both unchanged and healthy, exactly as at the end of Part 1's fix. Transport layer is not the problem.

## Step 2: Confirm BGP Session and Address Families (Still Healthy)

Given Part 2 just happened, check this layer again rather than assuming it stayed fixed - configuration can drift, and there is no guarantee only one thing changed since the last ticket.

```junos
PE1> show bgp summary
```

Observed:

```text
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.l2vpn.0
                       1          1          0          0          0          0
bgp.l3vpn.0
                       2          2          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.0.4              65001        320        318       0       2       01:15:40 Establ
  bgp.l2vpn.0: 1/1/1/0
  VPLS-100.l2vpn.0: 1/1/1/0
  bgp.l3vpn.0: 2/2/2/0
  VPN-A.inet.0: 1/1/1/0
```

The session is `Establ`, `bgp.l3vpn.0` shows 2 paths - the address family is negotiated and working at the BGP session level. But look at the last line: `VPN-A.inet.0: 1/1/1/0` - only **one** route accepted into the VRF, where the Part 0 baseline showed 2. Something is being received into `bgp.l3vpn.0` but not all of it is making it into the local VRF table.

## Step 3: Inspect bgp.l3vpn.0 Directly

```junos
PE1> show route table bgp.l3vpn.0
```

Observed:

```text
bgp.l3vpn.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

65001:1000:10.0.0.11/32
                   *[BGP/170] 04:20:11, localpref 100
                      AS path: 65100 I
                    > Indirect
65001:2000:10.0.0.12/32
                   *[BGP/170] 00:02:03, localpref 100, from 10.0.0.4
                      AS path: 65100 I
                    > to 10.1.12.2 via ge-0/0/0.0, Push <vpn-label>, Push 299808
```

Both VPN-IPv4 routes are present in `bgp.l3vpn.0` - PE1's own route (`65001:1000:...`) and PE2's route for CE2's loopback (`65001:2000:10.0.0.12/32`). The route from PE2 arrived over BGP successfully. This confirms Part 2's fault is not recurring - the address family is negotiated and PE2 is advertising the route.

Now check the VRF table itself:

```junos
PE1> show route table VPN-A.inet.0
```

Observed:

```text
VPN-A.inet.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.11/32       *[BGP/170] 04:20:11, localpref 100
                      AS path: 65100 I, validation-state: unverified
                    > to 172.16.1.2 via ge-0/0/1.0
172.16.1.0/30      *[Direct/0] 05:02:33
                    > via ge-0/0/1.0
172.16.1.1/32      *[Local/0] 05:02:33
                      Local via ge-0/0/1.0
```

**CE2's loopback (10.0.0.12/32) and PE2's link subnet (172.16.2.0/30) are both missing from the VRF table**, even though the route is clearly present in `bgp.l3vpn.0` one command ago. This is the exact signature of a route-target import mismatch: the route exists in the global VPN-IPv4 table (BGP received and accepted it), but the local VRF's import policy does not recognize it as belonging to VPN-A, so it never gets imported into `VPN-A.inet.0`.

## Step 4: Check the Route Target Configuration on Both PEs

```junos
PE1> show configuration routing-instances VPN-A vrf-target
```

Observed on PE1 - unchanged:

```text
target:65001:100;
```

```junos
PE2> show configuration routing-instances VPN-A vrf-target
```

Observed on PE2:

```text
target:65001:200;
```

**Root cause found.** PE2's `VPN-A` routing instance is now using `vrf-target target:65001:200` instead of the correct `target:65001:100`. Since `vrf-target` sets both the export and import route target to the same value, this single change affects both directions:

- PE2 now **exports** VPN-A routes tagged with `target:65001:200` instead of `target:65001:100` - so PE1's VRF (still importing `target:65001:100`) does not recognize PE2's routes as belonging to VPN-A and does not import them. This is exactly what Step 3 showed.
- PE2 now **imports** only routes carrying `target:65001:200` - so PE2's own VRF will also fail to import PE1's routes (`target:65001:100`), even though those routes are correctly present in PE2's `bgp.l3vpn.0`.

The BGP session, the address families, and the underlying transport are all completely healthy. The fault is entirely inside a single line of VRF configuration on PE2.

!!! note "Why this did not affect VPLS-100"
    VPLS-100 uses its own independent `vrf-target target:65001:100` under the `VPLS-100` routing instance, configured separately in Session 7a. Changing VPN-A's route target has no effect on VPLS-100's route target - they are two distinct routing-instance stanzas, and this is exactly why the RD/RT plan in Session 8 kept them numerically distinct in the first place.

## Step 5: Fix

Correct the route target on PE2 back to the value shared with PE1.

```junos
PE2> configure

set routing-instances VPN-A vrf-target target:65001:100

commit
```

!!! warning "Confirm you are editing the right routing-instance"
    Before committing, run `show | compare` to confirm the only pending change is inside `routing-instances VPN-A` and that you have not accidentally touched `routing-instances VPLS-100`, which also uses `target:65001:100` but is a completely separate customer service.

## Step 6: Verify the Fix

```junos
PE2> show configuration routing-instances VPN-A vrf-target
```

Expected:

```text
target:65001:100;
```

```junos
PE1> show route table VPN-A.inet.0
```

Expected - four routes restored:

```text
VPN-A.inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.11/32       *[BGP/170] 04:35:02, localpref 100
                      AS path: 65100 I, validation-state: unverified
                    > to 172.16.1.2 via ge-0/0/1.0
10.0.0.12/32       *[BGP/170] 00:00:22, localpref 100, from 10.0.0.4
                      AS path: 65100 I, validation-state: unverified
                    > to 10.1.12.2 via ge-0/0/0.0, Push <vpn-label>, Push 299808
172.16.1.0/30      *[Direct/0] 05:17:24
                    > via ge-0/0/1.0
172.16.2.0/30      *[BGP/170] 00:00:22, localpref 100, from 10.0.0.4
                    > to 10.1.12.2 via ge-0/0/0.0, Push <vpn-label>, Push 299808
```

```junos
PE1> show bgp summary
```

Expected - `VPN-A.inet.0: 2/2/2/0` restored under the peer.

Finally, confirm end-to-end reachability one last time:

```junos
CE1> ping 10.0.0.12 source 10.0.0.11 count 5
```

Expected - 5/5 replies, 0% packet loss.

```junos
CE1> ping 192.168.1.2 count 5
```

Expected - 5/5 replies, 0% packet loss.

Both directions of VPN-A should also be tested:

```junos
CE2> ping 10.0.0.11 source 10.0.0.12 count 5
```

Expected - 5/5 replies, 0% packet loss.

!!! tip "Why this fault was placed last"
    This fault is the hardest of the three because every layer below it - interface, IS-IS, LDP, and the BGP session itself with its address families - is completely healthy. The only observable evidence is a route present in `bgp.l3vpn.0` but absent from `VPN-A.inet.0`. Recognizing that specific signature (present in the BGP table, missing from the VRF) as a route-target problem, rather than assuming BGP itself is broken, is the core skill this session is building toward.

All three faults are now fixed. Proceed to the [Verification Checklist](verify.md) to confirm full end-to-end health across every layer and every service.
