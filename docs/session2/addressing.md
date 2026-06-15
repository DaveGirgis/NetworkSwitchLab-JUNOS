# Session 2 — Addressing Table

## Interface Addresses

| Device | Interface | IPv4 Address | Subnet | Purpose |
|--------|-----------|-------------|--------|---------|
| R1 | `ge-0/0/0` unit 0 | 10.1.12.1 | /30 | WAN link R1–R2 |
| R1 | `lo0` unit 0 | 10.0.0.1 | /32 | Loopback |
| R2 | `ge-0/0/0` unit 0 | 10.1.12.2 | /30 | WAN link R1–R2 |
| R2 | `lo0` unit 0 | 10.0.0.2 | /32 | Loopback |

## Subnet Reference

| Subnet | Purpose | Usable Hosts |
|--------|---------|-------------|
| 10.1.12.0/30 | R1 — R2 WAN link | .1 (R1), .2 (R2) |
| 10.0.0.1/32 | R1 loopback | R1 only |
| 10.0.0.2/32 | R2 loopback | R2 only |

## Static Routes

| Device | Destination | Next-hop | Purpose |
|--------|-------------|----------|---------|
| R1 | 10.0.0.2/32 | 10.1.12.2 | Reach R2 loopback |
| R2 | 10.0.0.1/32 | 10.1.12.1 | Reach R1 loopback |

!!! note "/30 for point-to-point links"
    A /30 gives you 4 addresses, 2 usable — perfect for point-to-point router links. These labs use the convention `.1` for the "left" router and `.2` for the "right" router on each link, matching the router number in the node name.
