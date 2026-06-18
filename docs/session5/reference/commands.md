# Session 5 — Command Reference

## IS-IS Configuration

| Command | Description |
|---------|-------------|
| `set interfaces lo0 unit 0 family iso address <NET>` | Assign NET address to loopback (required for IS-IS) |
| `set interfaces ge-0/0/0 unit 0 family iso` | Enable ISO address family on transit interface |
| `set protocols isis level 1 disable` | Run Level 2 only (standard for SP backbones) |
| `set protocols isis interface ge-0/0/0.0 point-to-point` | Add interface to IS-IS as p2p link |
| `set protocols isis interface lo0.0 passive` | Advertise loopback prefix, suppress IS-IS hellos |

## IS-IS Show Commands

| Command | Description |
|---------|-------------|
| `show isis adjacency` | Neighbor summary — system name, level, state, hold timer |
| `show isis adjacency detail` | Full neighbor detail including area address and IP |
| `show isis interface` | Interface participation, DR state, metric |
| `show isis database` | All LSPs in the LSDB |
| `show isis database detail` | Full LSP content with TLVs |
| `show isis database PE1.00-00 detail` | One specific LSP by ID |
| `show isis route` | IS-IS SPF-computed routes |
| `show route protocol isis` | IS-IS routes in inet.0 |
| `show isis statistics` | Hello, LSP, and CSNP counters |

## NET Address Format

```
49.0001.0100.0000.0001.00
^   ^    ^^^^^^^^^^^^^^^^  ^
|   |    System ID         SEL (always 00)
|   Area (0001)
AFI (49 = private)
```

Derive System ID from loopback: pad each octet to 3 digits, concatenate, group in 4s.
`10.0.0.1` → `010.000.001` → `0100.0000.0001`

## IS-IS vs OSPF Reference

| Concept | OSPF Term | IS-IS Term |
|---------|-----------|------------|
| Router identifier | Router ID | NET / System ID |
| Topology unit | LSA | LSP (Link State PDU) |
| Database | LSDB | LSDB |
| Hello protocol | OSPF Hello | IS-IS Hello (IIH) |
| Backbone area | Area 0.0.0.0 | Level 2 |
| Sub-area | Non-zero area | Level 1 |
| Route preference | 10 (internal) | 18 (L2 internal) |
| Default metric | 1 | 10 |

## Session 5 Wrap-Up

IS-IS is now the IGP for your provider backbone. The outcome is identical to OSPF — all four loopbacks are reachable — but the mechanism differs: IS-IS runs over Layer 2, uses TLVs instead of typed LSAs, and is the protocol you will encounter on real SP equipment.

In Session 6 you will build **BGP** on top of this IS-IS backbone. IS-IS provides reachability between loopbacks; BGP uses those loopbacks as its peering addresses for iBGP sessions across the provider core.
