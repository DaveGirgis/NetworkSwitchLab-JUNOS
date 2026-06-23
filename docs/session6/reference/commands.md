# Session 6 — Command Reference

## BGP Configuration

| Command | Description |
|---------|-------------|
| `set routing-options autonomous-system 65001` | Set the local AS number |
| `set routing-options router-id 10.0.0.1` | Set the BGP router ID (defaults to highest loopback if not set) |
| `set protocols bgp group <name> type external` | Define an eBGP peer group |
| `set protocols bgp group <name> type internal` | Define an iBGP peer group |
| `set protocols bgp group <name> peer-as 65100` | Set the expected peer AS |
| `set protocols bgp group <name> neighbor <ip>` | Add a neighbor to the group |
| `set protocols bgp group <name> local-address 10.0.0.1` | Set the local source address (use loopback for iBGP) |
| `set protocols bgp group <name> export <policy>` | Apply an export policy to control which routes are advertised |
| `set protocols bgp group <name> neighbor <ip> as-override` | Replace the peer's AS in the AS_PATH with the local AS — required when PE advertises to same-AS CE sites |

## Routing Policy (Prefix Advertisement & Next-Hop)

| Command | Description |
|---------|-------------|
| `set policy-options policy-statement <name> term <t> from protocol direct` | Match directly connected routes |
| `set policy-options policy-statement <name> term <t> from route-filter <prefix> exact` | Match an exact prefix |
| `set policy-options policy-statement <name> term <t> then accept` | Accept (advertise) matched routes |
| `set policy-options policy-statement <name> term <t> then reject` | Reject (suppress) matched routes |
| `set policy-options policy-statement <name> term <t> from protocol bgp` | Match only BGP-learned routes (required to prevent IS-IS/direct routes leaking into iBGP) |
| `set policy-options policy-statement <name> term <t> then next-hop self` | Replace NEXT_HOP with local router address (used for iBGP next-hop-self on vMX 14.1) |

## BGP Show Commands

| Command | Description |
|---------|-------------|
| `show bgp summary` | All BGP peers, state, and prefix counts |
| `show bgp neighbor <ip>` | Full detail for one neighbor — state, timers, capabilities, prefix counts |
| `show bgp neighbor <ip> advertised-routes` | Routes currently being sent to this peer |
| `show route receive-protocol bgp <ip>` | Routes received from this peer (Adj-RIB-In) |
| `show route protocol bgp` | All BGP routes active in inet.0 |
| `show route protocol bgp detail` | BGP routes with full path attributes |
| `show route <prefix> detail` | All attributes for a specific route |
| `show bgp group` | Group configuration summary |

## BGP Acronym Reference

| Acronym | Full Name | Meaning |
|---------|-----------|---------|
| **BGP** | Border Gateway Protocol | The inter-domain routing protocol; RFC 4271 |
| **EGP** | Exterior Gateway Protocol | Generic term for inter-AS routing protocols (BGP is the only one in use) |
| **IGP** | Interior Gateway Protocol | Routing protocol within a single AS (IS-IS, OSPF) |
| **AS** | Autonomous System | A network under a single administrative domain with a consistent routing policy |
| **ASN** | Autonomous System Number | The number that identifies an AS (1–65535 for 2-byte; up to ~4 billion for 4-byte) |
| **eBGP** | External BGP | BGP session between routers in different autonomous systems |
| **iBGP** | Internal BGP | BGP session between routers in the same autonomous system |
| **NLRI** | Network Layer Reachability Information | The prefix (destination) being advertised in a BGP UPDATE message |
| **RIB** | Routing Information Base | The BGP route database |
| **Adj-RIB-In** | Adjacent RIB Inbound | Routes received from a peer before local policy is applied |
| **Adj-RIB-Out** | Adjacent RIB Outbound | Routes queued to be sent to a peer after local policy |
| **Loc-RIB** | Local RIB | Best-path routes after policy and path selection — what gets installed in inet.0 |
| **NEXT_HOP** | Next Hop | BGP path attribute — the IP address used to forward traffic toward the prefix |
| **LOCAL_PREF** | Local Preference | BGP attribute influencing outbound path selection within an AS (higher = preferred) |
| **MED** | Multi-Exit Discriminator | BGP attribute influencing which entry point a neighbor AS uses (lower = preferred) |
| **AS_PATH** | AS Path | Ordered list of ASes a route has traversed; used for loop prevention and path selection |
| **ORIGIN** | Origin | How the prefix entered BGP — IGP (i), EGP (e), or Incomplete (?) |
| **COMMUNITY** | Community | A tag attached to routes for flexible policy grouping |
| **RR** | Route Reflector | An iBGP router that re-advertises iBGP routes to clients, eliminating the full mesh requirement |
| **RRC** | Route Reflector Client | A router that peers with an RR and relies on it for iBGP route distribution |
| **TTL** | Time to Live | IP header field; eBGP uses TTL=1 by default (direct link only); iBGP uses TTL=255 (multihop) |
| **TCP** | Transmission Control Protocol | BGP runs over TCP port 179 |
| **OPEN** | BGP OPEN message | First message exchanged — carries ASN, BGP version, hold time, router ID |
| **UPDATE** | BGP UPDATE message | Carries NLRI advertisements and withdrawals |
| **KEEPALIVE** | BGP KEEPALIVE message | Sent every 1/3 of hold time to maintain the session (default every 30s) |
| **NOTIFICATION** | BGP NOTIFICATION message | Signals an error and tears down the session |
| **AFI** | Address Family Identifier | Identifies the protocol family — e.g., IPv4 (1), IPv6 (2) |
| **SAFI** | Subsequent Address Family Identifier | Sub-type within an AFI — e.g., unicast (1), VPN (128) |
| **MP-BGP** | Multi-Protocol BGP | RFC 4760 extension carrying non-IPv4 address families — used for VPNs in Session 8 |

## Session 6 Wrap-Up

BGP is now running across the full topology. CE1 and CE2 can each see the other's loopback prefix in their routing tables — the prefix has traveled eBGP to PE1, iBGP across the provider backbone, and eBGP out to CE2.

The missing piece is **forwarding**. P1 and P2 carry the traffic but have no route to CE prefixes — they only know IS-IS routes. Session 7 adds **MPLS and LDP**: P routers forward on labels, eliminating the need for them to carry BGP prefixes. When Session 7 is complete, a ping from CE1 to CE2 will succeed for the first time.
