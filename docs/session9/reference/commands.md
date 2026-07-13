# Session 9 - Command Reference

## Baseline / Health Check Commands

| Command | Where to run | What it shows |
|---------|-------------|----------------|
| `show interfaces terse \| match ge-` | Any | Quick up/down state of all data interfaces |
| `show isis adjacency` | PE1, P1, P2, PE2 | IS-IS neighbor state per interface - first stop for any core reachability issue |
| `show isis interface <if> detail` | Any provider router | Per-interface IS-IS circuit type, level, timers - reveals level mismatches invisible in the adjacency summary |
| `show ldp session` | PE1, P1, P2, PE2 | LDP session state - `Operational` required for MPLS transport |
| `show ldp neighbor` | PE1, P1, P2, PE2 | LDP neighbor discovery state and label space ID |
| `show route table inet.3` | PE1, PE2 | MPLS-resolved next-hops used by BGP; confirms LDP end-to-end reachability |
| `show route table mpls.0` | Any provider router | Local LFIB - incoming label to outgoing action (Swap/Pop/Push) |

## BGP / MP-BGP Diagnostic Commands

| Command | Where to run | What it shows |
|---------|-------------|----------------|
| `show bgp summary` | PE1, PE2 | Session state (`Establ`) and per-table path counts - the first BGP-layer command to run |
| `show bgp summary instance VPN-A` | PE1, PE2 | VRF-scoped eBGP session state to the local CE |
| `show bgp neighbor <ip> \| match NLRI` | PE1, PE2 | Negotiated address families on the iBGP session - the critical check for Fault 2 |
| `show configuration protocols bgp group IBGP` | PE1, PE2 | Confirms which `family` statements are actually configured, not just negotiated |
| `show route table bgp.l3vpn.0` | PE1, PE2 | VPN-IPv4 (RD:prefix) routes received via MP-BGP - the critical check for Fault 3 |
| `show route table bgp.l2vpn.0` | PE1, PE2 | L2VPN signaling routes (VPLS-100) - the control group for comparison |
| `show route receive-protocol bgp <ip>` | CE1, CE2 | Confirms the CE actually received the expected prefix with the expected AS path |

## VRF / L3VPN Diagnostic Commands

| Command | Where to run | What it shows |
|---------|-------------|----------------|
| `show route table VPN-A.inet.0` | PE1, PE2 | The VRF's local routing table - direct, CE eBGP, and imported VPN routes |
| `show route table VPN-A.inet.0 <prefix> detail` | PE1, PE2 | Two-label stack detail (`Label operation: Push <vpn-label>, Push <transport-label>`) for a specific route |
| `show configuration routing-instances VPN-A vrf-target` | PE1, PE2 | The single most important command for Fault 3 - confirms export/import RT match |
| `show configuration routing-instances VPN-A route-distinguisher` | PE1, PE2 | Confirms RD uniqueness per PE |
| `show route table inet.0 <prefix>` | PE1, PE2 | Confirms customer prefixes are NOT leaking into the global table |

## VPLS-100 Control-Group Commands

| Command | Where to run | What it shows |
|---------|-------------|----------------|
| `show vpls connections` | PE1, PE2 | VPLS-100 pseudowire state - use as a control group to rule shared transport in or out |
| `show route table VPLS-100.l2vpn.0` | PE1, PE2 | BGP-distributed VPLS membership routes |
| `show configuration routing-instances VPLS-100 vrf-target` | PE1, PE2 | Confirms VPLS-100's independent RT was not accidentally touched while fixing VPN-A |

## Configuration Comparison Commands

| Command | Where to run | What it shows |
|---------|-------------|----------------|
| `show \| compare` | Any, in config mode | Diffs the candidate configuration against the active configuration - confirms a fix touches only the intended stanza |
| `show configuration protocols isis` | Any provider router | Full IS-IS stanza - used to spot a per-interface level override |
| `show configuration protocols bgp group IBGP` | PE1, PE2 | Full iBGP group stanza - used to spot a missing `family` statement |
| `show configuration routing-instances VPN-A` | PE1, PE2 | Full VRF stanza - used to spot an RD or RT typo |

## End-to-End Test Commands

| Command | Where to run | What it shows |
|---------|-------------|----------------|
| `ping 10.0.0.12 source 10.0.0.11 count 5` | CE1 | VPN-A end-to-end reachability (always specify `source`) |
| `ping 10.0.0.11 source 10.0.0.12 count 5` | CE2 | VPN-A reverse-direction reachability |
| `ping 192.168.1.2 count 5` | CE1 | VPLS-100 end-to-end reachability (no source needed - same subnet) |
| `ping 192.168.1.1 count 5` | CE2 | VPLS-100 reverse-direction reachability |

## Acronym Reference

| Acronym | Meaning |
|---------|---------|
| IGP | Interior Gateway Protocol - IS-IS in this lab |
| LSR-ID | Label Switch Router ID - the loopback address LDP uses to identify a peer |
| NLRI | Network Layer Reachability Information - the prefix information carried in a BGP update, scoped per address family |
| RD | Route Distinguisher - 64-bit value prepended to VPN prefixes for global BGP uniqueness |
| RT | Route Target - BGP extended community controlling which VRFs import which VPN routes |
| VRF | Virtual Routing and Forwarding - isolated routing table per customer per PE |
| MP-BGP | Multiprotocol BGP - BGP extension carrying address families beyond IPv4 unicast |
| PHP | Penultimate Hop Popping - the second-to-last router in an LSP pops the outer label |
| LFIB | Label Forwarding Information Base - the per-router table mapping incoming label to outgoing action |

## Session Wrap-Up

Session 9 closes the lab series by testing the same skill in three different forms: recognizing that a symptom (CE1 cannot reach CE2) can originate from any layer in the stack, and using a consistent, bottom-up diagnostic order to find out which one:

- **Fault 1 (IGP)** broke the foundation every other protocol depends on. The signature was symptom breadth - both VPN-A and VPLS-100 failed together, pointing straight at shared transport.
- **Fault 2 (BGP)** broke a control-plane capability while leaving the session itself up. The signature was a healthy session with a missing address family - a distinction only visible if you check NLRI negotiation, not just session state.
- **Fault 3 (MPLS/L3VPN)** broke route import while leaving BGP, the address family, and the underlying transport all completely healthy. The signature was a route present in `bgp.l3vpn.0` but absent from the VRF table - the narrowest and hardest-to-spot fault of the three.

In production, faults rarely announce their own layer. The discipline built across these three parts - confirm the symptom, start at the bottom of the stack, narrow by comparison, form a hypothesis before editing configuration, fix the smallest possible change, verify at the layer you fixed and then end-to-end - is the same discipline used on real service provider networks carrying real customer traffic.
