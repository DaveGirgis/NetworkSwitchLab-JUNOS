# Session 7a — Command Reference

## L2circuit Configuration Commands

| Command | Description |
|---------|-------------|
| `set interfaces ge-0/0/2 encapsulation ethernet-ccc` | Mark the CE-facing interface as a CCC (Circuit Cross-Connect) port — no IP, raw L2 forwarding |
| `set protocols l2circuit neighbor <remote-lo0> interface ge-0/0/2.0 virtual-circuit-id <id>` | Define a pseudowire to the remote PE, identified by a shared virtual-circuit-id |
| `delete interfaces ge-0/0/2 encapsulation` | Remove CCC encapsulation (when transitioning to VPLS) |
| `delete protocols l2circuit` | Remove all L2circuit configuration |

## VPLS Configuration Commands

| Command | Description |
|---------|-------------|
| `set protocols bgp group IBGP family l2vpn signaling` | Enable L2VPN BGP address family for VPLS membership distribution |
| `set routing-instances <name> instance-type vpls` | Create a VPLS routing instance |
| `set routing-instances <name> interface ge-0/0/2.0` | Add CE-facing interface to the VPLS instance |
| `set routing-instances <name> route-distinguisher <rd>` | Unique per-PE identifier for BGP VPLS routes (format: ASN:id) |
| `set routing-instances <name> vrf-target target:<rt>` | Route target for VPLS export/import — must match on all participating PEs |
| `set routing-instances <name> protocols vpls no-tunnel-services` | Use native Ethernet forwarding (required on vMX — no physical Tunnel PIC present) |
| `set routing-instances <name> protocols vpls site <site-name> site-identifier <n>` | Name this PE's VPLS site and assign a unique site ID |
| `set routing-instances <name> protocols vpls site <site-name> interface ge-0/0/2.0` | Bind the CE interface to this VPLS site |

## Verification Commands

| Command | Where to run | What it shows |
|---------|-------------|---------------|
| `show l2circuit connections` | PE | Pseudowire state, VC labels, encapsulation (Part 1) |
| `show ldp session` | PE | All LDP sessions including targeted LDP created by l2circuit |
| `show vpls connections` | PE | VPLS pseudowire state and labels (Part 2) |
| `show vpls mac-table instance <name>` | PE | MAC addresses learned per VPLS instance |
| `show route table <name>.l2vpn.0` | PE | BGP-distributed VPLS routes |
| `show bgp neighbor <ip> \| match NLRI` | PE | Negotiated BGP address families — confirms l2vpn is active |
| `show interfaces ge-0/0/2 terse` | PE | Link state of L2 service interface |
| `show interfaces ge-0/0/1 terse` | CE | Confirm CE test interface is up with correct IP |

## Acronym Reference

| Acronym | Meaning |
|---------|---------|
| L2VPN | Layer 2 Virtual Private Network |
| L2circuit | Layer 2 Circuit — Junos name for a point-to-point pseudowire |
| CCC | Circuit Cross-Connect — Junos mechanism for raw L2 forwarding |
| VC | Virtual Circuit — the pseudowire identified by a virtual-circuit-id |
| VPLS | Virtual Private LAN Service — multipoint L2 VPN |
| RD | Route Distinguisher — per-PE unique identifier for BGP VPN routes |
| RT | Route Target — community used to control import/export of VPN routes |
| MAC | Media Access Control — Layer 2 hardware address |
| ARP | Address Resolution Protocol — maps IP addresses to MAC addresses |
| PHP | Penultimate Hop Popping — removes outer MPLS label one hop before egress |
| NLRI | Network Layer Reachability Information — BGP prefix information |

## Session Wrap-Up

After completing Session 7a, the lab has demonstrated:

- **L2circuit**: A point-to-point pseudowire using targeted LDP signaling. One virtual-circuit-id ties PE1's CCC interface to PE2's, and LDP exchanges the VC labels. CE1 and CE2 communicate on the same /24 without any IP config on the PEs.

- **VPLS**: A multipoint virtual LAN using BGP signaling. The `l2vpn signaling` address family distributes VPLS membership. Each PE runs a MAC bridge for the VPLS instance — MAC learning, flooding, and split-horizon are all handled by the PE. CE1 and CE2 again see a shared Ethernet segment, but the VPLS instance can scale to any number of PE sites.

Both services use the same two-label MPLS stack: outer transport label (LDP from Session 7) and inner VC label (signaled per-service). P1 and P2 process only the outer label and remain completely unaware of the L2 service running inside.

The next step, Session 8, uses the same MP-BGP infrastructure (iBGP + address families) but shifts to **Layer 3 VPN** — where the provider takes ownership of IP routing via VRFs, route distinguishers, and route targets applied to IPv4 prefixes rather than L2 pseudowires.
