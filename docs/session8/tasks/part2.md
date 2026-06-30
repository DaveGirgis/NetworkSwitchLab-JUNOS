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

Expected — both `inet-unicast` and `inet-vpn` appear in the negotiated families:

```text
  NLRI for restart configured on peer: inet-unicast inet-vpn
  NLRI advertised by peer: inet-unicast inet-vpn
  NLRI for this session: inet-unicast inet-vpn
```

If `inet-vpn` is missing from "NLRI for this session", the capability was not successfully negotiated. Confirm PE2 also has `family inet-vpn unicast` configured under the IBGP group and that the BGP session is `Established`.

## Step 4: Verify bgp.l3vpn.0

At this stage the VPN-IPv4 table exists but is likely empty — no CE routes have been learned yet because the VRF eBGP sessions to CE1 and CE2 haven't been configured.

```junos
PE1> show route table bgp.l3vpn.0
```

Expected at end of Part 2 — empty or no output:

```text
bgp.l3vpn.0: 0 destinations, 0 routes (0 active, 0 holddown, 0 hidden)
```

This is correct. The VPN route table will populate in Part 3 once CE routes enter the VRFs and are redistributed to MP-BGP.

!!! note "Why bgp.l3vpn.0 may already have entries"
    If the PE's VRF already has direct routes (the CE-facing subnet) and the vrf-target export is active, Junos may have already originated those direct routes as VPN-IPv4 entries into bgp.l3vpn.0. This is fine — it means the export mechanism is working.

## Step 5: Confirm the iBGP Session Is Still Healthy

The iBGP session between PE1 and PE2 should remain established throughout this part. Adding a new address family to an established BGP session causes a brief capability re-negotiation but does not tear the session down in Junos 14.1.

```junos
PE1> show bgp summary
```

Expected — iBGP to PE2 still shows established (numeric counts, not `Active`):

```text
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.0.4              65001         52         51       0       0       22:34 0/0/0/0              0/0/0/0
```

The prefix counts are 0/0/0/0 — expected, since CE routes are not yet in the VRFs. The session itself is established.
