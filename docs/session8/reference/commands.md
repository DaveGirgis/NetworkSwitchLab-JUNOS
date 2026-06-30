# Session 8 — Command Reference

## VRF Configuration Commands

| Command | Description |
|---------|-------------|
| `set routing-instances <name> instance-type vrf` | Create a VRF routing instance |
| `set routing-instances <name> interface ge-0/0/1.0` | Assign CE-facing interface to the VRF |
| `set routing-instances <name> route-distinguisher <rd>` | Set the RD for VPN prefix uniqueness (format ASN:id, unique per PE per VPN) |
| `set routing-instances <name> vrf-target target:<rt>` | Set RT for both export and import (must match on all PEs in the same VPN) |
| `set routing-instances <name> vrf-table-label` | Allocate a single MPLS label for the VRF (the inner VPN label) |
| `delete protocols bgp group EBGP-CE1` | Remove the global eBGP group before migrating to VRF-based eBGP |

## MP-BGP Configuration Commands

| Command | Description |
|---------|-------------|
| `set protocols bgp group IBGP family inet-vpn unicast` | Add VPN-IPv4 address family to the iBGP session — enables bgp.l3vpn.0 |

## VRF eBGP Configuration Commands

| Command | Description |
|---------|-------------|
| `set routing-instances <name> protocols bgp group <grp> type external` | Configure eBGP group inside the VRF |
| `set routing-instances <name> protocols bgp group <grp> peer-as <as>` | Set the CE's AS number |
| `set routing-instances <name> protocols bgp group <grp> neighbor <ip>` | Add CE neighbor address |
| `set routing-instances <name> protocols bgp group <grp> neighbor <ip> as-override` | Replace customer AS in AS_PATH — required when CE sites share an ASN |
| `set routing-instances <name> protocols bgp group <grp> export ADVERTISE-VPN` | Apply export policy to advertise remote VPN routes to CE |
| `set policy-options policy-statement ADVERTISE-VPN term 1 from protocol bgp` | Match BGP routes in the VRF (imported VPN routes from remote PE) |
| `set policy-options policy-statement ADVERTISE-VPN term 1 then accept` | Accept and advertise them to the CE |

## Verification Commands

| Command | Where to run | What it shows |
|---------|-------------|---------------|
| `show route table VPN-A.inet.0` | PE | All routes in the VRF: direct, CE eBGP, and imported VPN routes |
| `show route table VPN-A.inet.0 <prefix> detail` | PE | Two-label stack detail for a specific VPN route |
| `show route table bgp.l3vpn.0` | PE | VPN-IPv4 entries in RD:prefix format from bgp.l3vpn.0 |
| `show bgp summary instance VPN-A` | PE | BGP session state for CE eBGP sessions inside the VRF |
| `show bgp neighbor <ip> \| match NLRI` | PE | Negotiated address families — confirms inet-vpn is active |
| `show route table inet.0 <prefix>` | PE | Confirm VPN routes are NOT leaking into the global table |
| `show route receive-protocol bgp <ip>` | CE | Prefixes received from PE — confirms as-override AS_PATH |
| `show route protocol bgp` | CE | All BGP routes on CE — CE1 should see CE2's loopback and vice versa |

## Acronym Reference

| Acronym | Meaning |
|---------|---------|
| L3 VPN | Layer 3 Virtual Private Network — provider routes customer IP traffic between sites |
| VRF | Virtual Routing and Forwarding — isolated routing table per customer per PE |
| RD | Route Distinguisher — 64-bit value prepended to VPN prefixes for global BGP uniqueness |
| RT | Route Target — BGP extended community controlling which VRFs import which VPN routes |
| MP-BGP | Multiprotocol BGP — BGP extension carrying address families beyond IPv4 unicast |
| VPN-IPv4 | BGP address family carrying (RD, IPv4 prefix) tuples — lives in bgp.l3vpn.0 |
| NLRI | Network Layer Reachability Information — the prefix information carried in a BGP update |
| PE | Provider Edge — the router that hosts VRFs and connects to CE routers |
| CE | Customer Edge — the customer router; unaware of the VRF on the PE |
| P | Provider core router — MPLS transit only, no VRF, no customer prefix knowledge |
| PHP | Penultimate Hop Popping — P2 pops the outer transport label before delivering to PE2 |
| VC Label | Virtual Circuit label — the inner MPLS label identifying the VRF on the egress PE |

## Session Wrap-Up

Session 8 completes the BGP/MPLS L3 VPN stack:

- **Session 4–5**: IS-IS and eBGP establish IP reachability between provider and customer
- **Session 7**: LDP builds the outer MPLS transport label path between PEs (inn inet.3)
- **Session 8**: VRFs isolate customer routes; MP-BGP carries them as VPN-IPv4; the inner VPN label identifies the VRF on the egress PE

The two-label stack ties Sessions 7 and 8 together: the outer LDP label (from Session 7, unchanged) carries VPN packets through the provider backbone; the inner VPN label (from `vrf-table-label`) is processed only by the egress PE. P1 and P2 required zero changes in Session 8 — this is the scalability property of the architecture.

In production, this same framework supports thousands of VPN customers on a single set of PE routers. Each VPN gets its own VRF, its own RD/RT, and its own label. The provider backbone remains oblivious to customer addressing.
