# Part 3 — PE-CE Routing in the VRF

## Overview

The VRFs exist and MP-BGP is ready to carry VPN routes. The final step is to configure eBGP sessions **inside the VRF** on each PE toward the CE. These sessions replace the global eBGP groups deleted in Part 1.

From CE1 and CE2's perspective nothing changes — they still peer with the same PE addresses (172.16.1.1 and 172.16.2.1). The difference is that those sessions are now handled within the VRF context, so routes learned from CE land in `VPN-A.inet.0` instead of `inet.0`.

### What Happens End-to-End

After Part 3 is complete, the full VPN path works as follows:

1. **CE1 → PE1 (eBGP in VRF)**: CE1 advertises `10.0.0.11/32` to PE1. PE1's VRF eBGP session receives it and installs it in `VPN-A.inet.0`.
2. **PE1 → PE2 (iBGP inet-vpn)**: PE1 exports `10.0.0.11/32` as a VPN-IPv4 route (`65001:100:10.0.0.11/32`) to PE2 via iBGP, tagged with RT `target:65001:100`.
3. **PE2 VRF import**: PE2 sees the RT matches its import policy and installs `10.0.0.11/32` into `VPN-A.inet.0`. The route carries PE2's VPN label as the inner label and PE2's LDP label (for the outer transport) — forming the two-label stack.
4. **PE2 → CE2 (eBGP in VRF)**: PE2 advertises `10.0.0.11/32` to CE2 via the VRF eBGP session.
5. **CE2 → PE2 → PE1 → CE1**: The reverse path follows the same logic for CE2's loopback.

### ADVERTISE-VPN Policy

The VRF eBGP group needs an export policy to know which routes to advertise to the CE. Routes imported into the VRF from the remote PE arrive with protocol `bgp` in `VPN-A.inet.0`. A policy matching `from protocol bgp` will select exactly those remote VPN routes for advertisement to the CE — while BGP's own loop prevention prevents re-advertising a route back to the peer it came from.

### as-override

CE1 and CE2 are both in AS 65100. When CE2 receives `10.0.0.11/32` from PE2, the AS_PATH contains `65100 I` (CE1's AS). BGP's loop prevention would suppress this route at CE2 (its own AS is in the path). The `as-override` knob on the PE neighbor statement replaces the customer AS in the AS_PATH with the provider AS (65001) before advertising. CE2 receives the route with AS_PATH `65001 65001 I` and accepts it.

## Step 1: Create the ADVERTISE-VPN Policy and Configure VRF eBGP on PE1

```junos
configure

set policy-options policy-statement ADVERTISE-VPN term 1 from protocol bgp
set policy-options policy-statement ADVERTISE-VPN term 1 then accept

set routing-instances VPN-A protocols bgp group EBGP-CE1 type external
set routing-instances VPN-A protocols bgp group EBGP-CE1 peer-as 65100
set routing-instances VPN-A protocols bgp group EBGP-CE1 neighbor 172.16.1.2
set routing-instances VPN-A protocols bgp group EBGP-CE1 neighbor 172.16.1.2 as-override
set routing-instances VPN-A protocols bgp group EBGP-CE1 export ADVERTISE-VPN

commit
```

## Step 2: Create the ADVERTISE-VPN Policy and Configure VRF eBGP on PE2

```junos
configure

set policy-options policy-statement ADVERTISE-VPN term 1 from protocol bgp
set policy-options policy-statement ADVERTISE-VPN term 1 then accept

set routing-instances VPN-A protocols bgp group EBGP-CE2 type external
set routing-instances VPN-A protocols bgp group EBGP-CE2 peer-as 65100
set routing-instances VPN-A protocols bgp group EBGP-CE2 neighbor 172.16.2.2
set routing-instances VPN-A protocols bgp group EBGP-CE2 neighbor 172.16.2.2 as-override
set routing-instances VPN-A protocols bgp group EBGP-CE2 export ADVERTISE-VPN

commit
```

## Step 3: Verify the VRF BGP Session

Wait ~30 seconds for the eBGP sessions to re-establish. CE1 and CE2 still have their `EBGP-PE1` / `EBGP-PE2` groups and ADVERTISE-LOOPBACK export policies from Session 6 — they do not need any changes.

On **PE1**, check the BGP session within the VRF instance:

```junos
show bgp summary instance VPN-A
```

Expected — one eBGP peer (CE1) established with one received prefix:

```text
Instance: VPN-A
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
VPN-A.inet.0
                       1          1          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.16.1.2            65100         12         10       0       0        0:48 1/1/1/0              0/0/0/0
```

The peer `172.16.1.2` (CE1) shows established with `1/1/1/0` — one prefix received (CE1's loopback). Run the same check on PE2:

```junos
show bgp summary instance VPN-A
```

Expected — CE2 established:

```text
Instance: VPN-A
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
VPN-A.inet.0
                       2          2          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.16.2.2            65100         12         10       0       0        0:50 1/1/1/0              0/0/0/0
```

## Step 4: Verify the VRF Routing Table

On **PE1**, inspect the full VRF table:

```junos
show route table VPN-A.inet.0
```

Expected — four routes: the CE-facing link subnet (direct), CE1's loopback (eBGP from CE1), CE2's loopback (imported from PE2 via iBGP), and PE2's CE-facing subnet (also imported):

```text
VPN-A.inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.11/32       *[BGP/170] 00:01:15, localpref 100
                      AS path: 65100 I, validation-state: unverified
                    > to 172.16.1.2 via ge-0/0/1.0
10.0.0.12/32       *[BGP/170] 00:00:52, localpref 100, from 10.0.0.4
                      AS path: 65100 I, validation-state: unverified
                    > to 10.1.12.2 via ge-0/0/0.0, Push <vpn-label>, Push 299808
172.16.1.0/30      *[Direct/0] 00:15:22
                    > via ge-0/0/1.0
172.16.2.0/30      *[BGP/170] 00:00:52, localpref 100, from 10.0.0.4
                    > to 10.1.12.2 via ge-0/0/0.0, Push <vpn-label>, Push 299808
```

The route `10.0.0.12/32` shows the two-label stack — `Push <vpn-label>` (inner VPN label) followed by `Push 299808` (outer LDP transport label, same as Session 7). The `<vpn-label>` is PE2's `vrf-table-label` allocation and will be a different number each session.

## Step 5: Verify bgp.l3vpn.0

On **PE1**, check the VPN-IPv4 route table:

```junos
show route table bgp.l3vpn.0
```

Expected — two VPN-IPv4 routes with their RDs:

```text
bgp.l3vpn.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

65001:1000:10.0.0.11/32
                   *[BGP/170] 00:01:15, localpref 100
                      AS path: 65100 I
                    > Indirect
65001:2000:10.0.0.12/32
                   *[BGP/170] 00:00:52, localpref 100, from 10.0.0.4
                      AS path: 65100 I
                    > to 10.1.12.2 via ge-0/0/0.0, Push <vpn-label>, Push 299808
```

The first route (`65001:1000:10.0.0.11/32`) is PE1's own locally originated VPN route — CE1's loopback with PE1's RD prepended. It shows as `Indirect` because it was originated from the VRF and redistributed into `bgp.l3vpn.0`.

The second route (`65001:2000:10.0.0.12/32`) is PE2's VPN route — CE2's loopback with PE2's RD (`65001:2000`) prepended. It was received via iBGP from PE2 and imported into VPN-A because the RT matched.

## Step 6: Inspect the Two-Label Stack in Detail

On **PE1**, view the detail for CE2's VPN route:

```junos
show route table VPN-A.inet.0 10.0.0.12 detail
```

Expected — key fields showing the two-label stack:

```text
VPN-A.inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
10.0.0.12/32 (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Next hop type: Indirect
                Source: 10.0.0.4
                Next hop: 10.1.12.2 via ge-0/0/0.0, selected
                Label operation: Push <vpn-label>, Push 299808
                Label TTL action: prop-ttl, prop-ttl
                Protocol next hop: 10.0.0.4
                State: <Active Int Ext>
                Local AS: 65001 Peer AS: 65001
                AS path: 65100 I
                VPN Label: <vpn-label>
                Localpref: 100
                Router ID: 10.0.0.4
```

The critical field is `Label operation: Push <vpn-label>, Push 299808`:

- **First Push (`<vpn-label>`)** — PE2's VPN label (allocated by `vrf-table-label` on PE2). When PE2 receives a packet carrying this label, it identifies the VPN-A VRF, strips the label, and looks up `10.0.0.12` inside `VPN-A.inet.0` to find CE2.
- **Second Push (299808)** — the LDP outer transport label, identical to Session 7. P1 and P2 process only this label. They do not see the inner VPN label.

`State: <Active Int Ext>` — `Int` confirms the protocol next-hop (PE2's loopback 10.0.0.4) was resolved through `inet.3` (LDP), exactly as in Session 7.

## Step 7: CE-to-CE Ping

The full VPN path is in place. CE1 can reach CE2 and vice versa.

!!! warning "Use the loopback as source"
    CE1's loopback (10.0.0.11/32) is the prefix exported into the VPN. If you ping without specifying a source, Junos uses the outgoing interface address (172.16.1.2). That address is in the VPN but CE2 only has a BGP route back to 10.0.0.11/32, not 172.16.1.2. Always specify `source 10.0.0.11`.

```junos
CE1> ping 10.0.0.12 source 10.0.0.11 count 5
```

Expected — all five replies:

```text
PING 10.0.0.12 (10.0.0.12): 56 data bytes
64 bytes from 10.0.0.12: icmp_seq=0 ttl=60 time=31.204 ms
64 bytes from 10.0.0.12: icmp_seq=1 ttl=60 time=44.817 ms
64 bytes from 10.0.0.12: icmp_seq=2 ttl=60 time=27.391 ms
64 bytes from 10.0.0.12: icmp_seq=3 ttl=60 time=38.644 ms
64 bytes from 10.0.0.12: icmp_seq=4 ttl=60 time=29.102 ms

--- 10.0.0.12 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 27.391/34.232/44.817/6.262 ms
```

TTL=60 — same as Session 7 (four hops: PE1→P1→P2→PE2 on the forward path, and the same in reverse). The VPN encapsulation does not add a visible hop because P routers are MPLS transit; only the label-switching hops count against TTL in this configuration.

Verify the reverse direction:

```junos
CE2> ping 10.0.0.11 source 10.0.0.12 count 5
```

Expected: all five replies succeed.

## Step 8: Confirm Routes Are Not Leaking into inet.0

A key property of L3 VPN is that customer routes stay in the VRF. Verify CE prefixes are absent from the global table on PE1:

```junos
PE1> show route table inet.0 10.0.0.11
PE1> show route table inet.0 10.0.0.12
```

Expected: no output. If either prefix appears in `inet.0`, the VRF configuration is incorrect and routes are leaking.
