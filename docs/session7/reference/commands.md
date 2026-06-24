# Session 7 — Command Reference

## MPLS Configuration

| Command | Description |
|---------|-------------|
| `set interfaces ge-0/0/x unit 0 family mpls` | Enable MPLS packet processing on an interface — required before MPLS protocols can forward labeled frames |
| `set protocols mpls interface ge-0/0/x.0` | Register the interface with the MPLS forwarding engine |
| `set protocols mpls label-switched-path <name> to <ip>` | Define an RSVP-TE LSP with this router as ingress, targeting the given egress loopback |
| `set protocols mpls label-switched-path <name> bandwidth <bps>` | Optional bandwidth reservation for the LSP (e.g., `10m` for 10 Mbps) |
| `set protocols mpls label-switched-path <name> primary <path-name>` | Bind an explicit named path to the LSP for traffic engineering |
| `set protocols mpls path <name> <ip> strict` | Define an explicit path hop (strict = must be directly adjacent) |

## LDP Configuration

| Command | Description |
|---------|-------------|
| `set protocols ldp interface ge-0/0/x.0` | Enable LDP on a data interface — LDP hellos are sent out this interface to discover neighbors |
| `set protocols ldp interface lo0.0` | Add loopback to LDP — sets the loopback as the LDP transport address (LSR-ID and TCP session source) |
| `set protocols ldp preference <value>` | Override LDP route preference in inet.3 (default 9) |
| `delete protocols ldp interface ge-0/0/x.0` | Disable LDP on a specific interface |

## RSVP-TE Configuration

| Command | Description |
|---------|-------------|
| `set protocols rsvp interface ge-0/0/x.0` | Enable RSVP on an interface — required on every interface along a potential LSP path |
| `set protocols rsvp interface ge-0/0/x.0 bandwidth <bps>` | Define the reservable bandwidth on this interface for CSPF |
| `set protocols isis traffic-engineering` | Enable IS-IS TE extensions (TLV 22) so CSPF has link bandwidth and metric information |
| `set protocols isis traffic-engineering family inet` | Restrict TE extensions to IPv4 (default if unspecified) |

## MPLS Show Commands

| Command | Description |
|---------|-------------|
| `show route table mpls.0` | LFIB — incoming labels and their swap/pop/receive actions on this router |
| `show route table inet.3` | MPLS-resolved next-hops — LDP and RSVP routes used by BGP for next-hop resolution |
| `show route <prefix> detail` | Full route detail including Label-stack when MPLS is in use |
| `show mpls lsp` | Summary of all MPLS LSPs (Ingress, Egress, Transit) on this router |
| `show mpls lsp detail` | Full detail for each LSP including path, labels, bandwidth |
| `show mpls interface` | MPLS-enabled interfaces and their status |

## LDP Show Commands

| Command | Description |
|---------|-------------|
| `show ldp neighbor` | Directly connected LDP neighbors discovered via hello messages |
| `show ldp session` | LDP TCP sessions and their state (Operational / OpenSent / etc.) |
| `show ldp database` | Input and output label bindings exchanged with each LDP peer |
| `show ldp database session <ip>` | Label bindings for a specific peer session |
| `show ldp route` | LDP routes computed from the label database |
| `show ldp statistics` | LDP message and session counters |

## RSVP Show Commands

| Command | Description |
|---------|-------------|
| `show rsvp session` | All RSVP sessions — Ingress (head-end), Egress (tail-end), Transit |
| `show rsvp session detail` | Full RSVP session detail including labels, bandwidth, path hops |
| `show rsvp neighbor` | Directly adjacent RSVP neighbors and hello state |
| `show rsvp interface` | RSVP-enabled interfaces and reservable bandwidth |

## MPLS Acronym Reference

| Acronym | Full Name | Meaning |
|---------|-----------|---------|
| **MPLS** | Multiprotocol Label Switching | Data-plane mechanism that forwards packets using fixed-length labels instead of IP lookups |
| **LDP** | Label Distribution Protocol | Protocol that distributes labels hop-by-hop following IGP paths; RFC 5036 |
| **RSVP-TE** | Resource Reservation Protocol — Traffic Engineering | Protocol that signals end-to-end LSPs with explicit paths and optional bandwidth reservation; RFC 3209 |
| **LSP** | Label Switched Path | The end-to-end path a labeled packet takes through the MPLS network |
| **FEC** | Forwarding Equivalence Class | A group of packets forwarded identically — typically one prefix per FEC |
| **LER** | Label Edge Router | The ingress or egress PE — pushes labels on entry, pops on exit |
| **LSR** | Label Switch Router | A transit P router — swaps incoming label for outgoing label |
| **LFIB** | Label Forwarding Information Base | Per-router table mapping incoming labels to outgoing label + interface |
| **PHP** | Penultimate Hop Popping | The second-to-last router pops the label before forwarding to the egress PE, reducing one label lookup on the PE |
| **inet.3** | MPLS Routing Table | Junos routing table holding MPLS-resolved next-hops; used by BGP for next-hop resolution |
| **mpls.0** | MPLS Forwarding Table | Junos routing table that is the LFIB — shows label swap/pop/receive entries |
| **TC** | Traffic Class | 3-bit field in MPLS label header used for QoS marking (formerly called EXP) |
| **TTL** | Time to Live | 8-bit field in MPLS label header; decremented at each hop; controls traceroute visibility through MPLS |
| **CSPF** | Constrained Shortest Path First | Algorithm used by RSVP-TE to compute the explicit LSP path considering bandwidth and TE constraints |
| **TE** | Traffic Engineering | The practice of routing traffic along non-IGP paths for bandwidth control or optimization |
| **DU** | Downstream Unsolicited | LDP advertisement mode — labels are distributed to neighbors without being requested (the default) |
| **LSR-ID** | Label Switch Router ID | The identifier for a router in the LDP domain — always the loopback address in Junos |
| **TLV** | Type-Length-Value | Encoding used by IS-IS, LDP, and RSVP to carry extensible information |
| **PATH** | RSVP PATH message | Sent by the ingress toward the egress to signal an LSP setup request |
| **RESV** | RSVP RESV message | Sent by the egress back to the ingress, allocating labels hop-by-hop |
| **FRR** | Fast Reroute | RSVP-TE mechanism that pre-computes a backup LSP path, enabling sub-50ms failover |
| **AF** | Address Family | inet.3 is used for the IPv4 address family; inet6.3 for IPv6 |

## Session 7 Wrap-Up

MPLS and LDP are now running across the provider backbone. P1 and P2 forward traffic using label swaps without any knowledge of customer prefixes. BGP next-hops are resolved via inet.3, giving every PE router a label-switched path to every other PE router. End-to-end CE-to-CE connectivity is established for the first time.

The next session (Session 8) builds directly on this foundation. **BGP/MPLS L3 VPN** uses a **two-label stack**: an outer transport label (the LDP LSP to the remote PE — exactly what this session configured) and an inner **VPN label** that identifies which customer VRF the packet belongs to. The P routers see only the outer label and remain unaware of VPN routing, exactly as they are now unaware of customer prefixes.
