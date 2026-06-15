# Session 2 — Command Reference

## Interface Commands

| Command | Description |
|---------|-------------|
| `set interfaces ge-0/0/0 unit 0 family inet address X.X.X.X/N` | Assign IPv4 address to physical interface |
| `set interfaces lo0 unit 0 family inet address X.X.X.X/32` | Assign loopback address |
| `delete interfaces ge-0/0/0 disable` | Re-enable an admin-down interface |
| `set interfaces ge-0/0/0 disable` | Admin-down an interface |
| `show interfaces terse` | All interfaces — one line each |
| `show interfaces ge-0/0/0` | Full detail for one interface |
| `show interfaces ge-0/0/0 detail` | Counters and error statistics |

## Routing Commands

| Command | Description |
|---------|-------------|
| `set routing-options static route X.X.X.X/N next-hop X.X.X.X` | Add static route |
| `delete routing-options static route X.X.X.X/N` | Remove static route |
| `set routing-options router-id X.X.X.X` | Set router ID (used by OSPF/BGP) |
| `show route` | Full routing table |
| `show route terse` | Compact routing table |
| `show route detail` | Full detail including next-hop and protocol |
| `show route X.X.X.X/N exact` | Lookup an exact prefix |
| `show route X.X.X.X/N longer` | All more-specific routes within a prefix |
| `show route protocol static` | Only static routes |
| `show route protocol direct` | Only connected routes |
| `show route forwarding-table` | Forwarding table (FIB) |

## Junos Route Preferences (Administrative Distance)

| Protocol | Preference | Cisco Equivalent (AD) |
|----------|------------|----------------------|
| Direct | 0 | 0 (connected) |
| Local | 0 | 0 (connected) |
| Static | 5 | 1 |
| OSPF internal | 10 | 110 |
| IS-IS Level 1 | 15 | 115 |
| IS-IS Level 2 | 18 | 115 |
| OSPF external | 150 | 110 (E2) |
| BGP | 170 | 20 (eBGP), 200 (iBGP) |

## Session 2 Wrap-Up

Static routing covers two-router scenarios but does not scale. Every static route must be manually added to every router. When a link fails, static routes to destinations behind that link turn black-hole unless you add BFD or track next-hop reachability.

Session 3 replaces static routes with **OSPF**, which discovers topology automatically and converges when links fail.
