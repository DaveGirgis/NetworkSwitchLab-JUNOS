# Part 2 — MP-BGP & Route Targets

## Overview

The VRFs exist and have their CE-facing interfaces. The next step is to enable the PEs to exchange VPN routes over iBGP. Standard iBGP (from Session 6) carries IPv4 unicast prefixes in `inet.0`. To carry VPN-IPv4 prefixes — the (RD, prefix) tuples from the VRFs — the iBGP session needs an additional address family: `inet-vpn unicast`.

Adding `family inet-vpn unicast` to the iBGP group on both PEs does three things:

1. **Negotiates the capability** — during BGP OPEN, both peers advertise support for the VPN-IPv4 NLRI. If only one side has the family configured, VPN routes will not be exchanged.
2. **Creates `bgp.l3vpn.0`** — Junos creates a new BGP table specifically for VPN-IPv4 routes. VPN routes received from the remote PE land here first, then are evaluated for import into the local VRF based on route target matching.
3. **Enables route export** — VRF routes tagged with the configured `vrf-target` RT community are automatically advertised to iBGP peers in the VPN-IPv4 NLRI.

No configuration is needed on P1, P2, CE1, or CE2. The `inet-vpn unicast` family is strictly between PE routers.

### bgp.l3vpn.0 — The VPN Route Table

`bgp.l3vpn.0` is the MP-BGP Loc-RIB for VPN-IPv4 routes. Each entry has the format `RD:prefix` — for example, `65001:200:10.0.0.12/32` is CE2's loopback as originated by PE2 (with PE2's RD of 65001:200).

This table is populated after Part 3 when the CE eBGP sessions inside the VRFs are up and CE routes are being exchanged. At the end of Part 2, `bgp.l3vpn.0` may still be empty — that is expected.

## Step 1: Add inet-vpn unicast to iBGP on PE1

```junos
configure

set protocols bgp group IBGP family inet-vpn unicast

commit
```

## Step 2: Add inet-vpn unicast to iBGP on PE2

```junos
configure

set protocols bgp group IBGP family inet-vpn unicast

commit
```

## Step 3: Verify NLRI Negotiation

After committing on both PEs, the iBGP session will briefly re-negotiate capabilities. Wait ~15 seconds, then confirm both address families are active on the session.

On **PE1**:

```junos
show bgp neighbor 10.0.0.4 | match NLRI
```

Expected — `inet-vpn-unicast` appears in the negotiated families. If Session 7a (VPLS) was completed, `l2vpn` will also appear:

```text
  NLRI for restart configured on peer: inet-vpn-unicast l2vpn
  NLRI advertised by peer: inet-vpn-unicast l2vpn
  NLRI for this session: inet-vpn-unicast l2vpn
  NLRI that restart is negotiated for: inet-vpn-unicast l2vpn
  NLRI of received end-of-rib markers: inet-vpn-unicast l2vpn
  NLRI of all end-of-rib markers sent: inet-vpn-unicast l2vpn
```

The key check is that `inet-vpn-unicast` appears in "NLRI for this session". The `l2vpn` family is the Session 7a VPLS configuration carried forward on the iBGP group — it is harmless and does not affect L3 VPN operation.

If `inet-vpn-unicast` is missing entirely, confirm PE2 also has `family inet-vpn unicast` under its IBGP group and that the BGP session is `Established`.

## Step 4: Verify bgp.l3vpn.0

Check the VPN-IPv4 route table. Because the VRFs on both PEs already have direct and local routes for the CE-facing subnets, and the `vrf-target` export is immediately active, those subnet routes are already being exported and imported as VPN-IPv4 entries — even before the CE eBGP sessions are configured.

```junos
PE1> show route table bgp.l3vpn.0
```

Expected — one entry already present (PE2's CE-facing subnet exported as a VPN route):

```text
bgp.l3vpn.0: 1 destination, 1 route (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

65001:2000:172.16.2.0/30
                   *[BGP/170] 00:04:37, localpref 100, from 10.0.0.4
                      AS path: I, validation-state: unverified
                    > to 10.1.12.2 via ge-0/0/0.0, Push <vpn-label>, Push 299808
```

This confirms the VRF RT export/import mechanism is already working. The CE loopbacks will join this table in Part 3 once the CE eBGP sessions are brought up inside the VRFs.

## Step 5: Confirm the iBGP Session Is Healthy

```junos
PE1> show bgp summary
```

Expected — when multiple BGP address families are active, Junos shows per-table prefix counts on separate lines under the peer:

```text
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.l2vpn.0
                       1          1          0          0          0          0
bgp.l3vpn.0
                       1          1          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.0.4              65001         12         15       0       0        4:37 Establ
  bgp.l2vpn.0: 1/1/1/0
  VPLS-100.l2vpn.0: 1/1/1/0
  bgp.l3vpn.0: 1/1/1/0
  VPN-A.inet.0: 1/1/1/0
```

The peer shows `Establ` (not `Active`) — the session is up. `bgp.l3vpn.0: 1/1/1/0` confirms one VPN-IPv4 route is being exchanged. `VPN-A.inet.0: 1/1/1/0` confirms PE1 has imported one VPN route into its local VRF table from PE2. The session is ready for Part 3.
