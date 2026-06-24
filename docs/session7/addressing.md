# Session 7 — Addressing Table

No new IP addresses are introduced in this session. All addressing is inherited from Session 6.

## Loopback Addresses

| Device | Interface | Address | MPLS Role |
|--------|-----------|---------|-----------|
| CE1 | lo0.0 | 10.0.0.11/32 | Outside MPLS domain |
| PE1 | lo0.0 | 10.0.0.1/32 | LDP transport address; LDP LSP egress for inbound traffic |
| P1 | lo0.0 | 10.0.0.2/32 | LDP transport address; LSR loopback |
| P2 | lo0.0 | 10.0.0.3/32 | LDP transport address; LSR loopback |
| PE2 | lo0.0 | 10.0.0.4/32 | LDP transport address; LDP LSP egress for inbound traffic |
| CE2 | lo0.0 | 10.0.0.12/32 | Outside MPLS domain |

## Point-to-Point Link Addresses

| Link | Device | Interface | Address | LDP on interface? |
|------|--------|-----------|---------|-------------------|
| CE1 — PE1 | CE1 | ge-0/0/0 | 172.16.1.2/30 | No |
| CE1 — PE1 | PE1 | ge-0/0/1 | 172.16.1.1/30 | No |
| PE1 — P1 | PE1 | ge-0/0/0 | 10.1.12.1/30 | Yes |
| PE1 — P1 | P1 | ge-0/0/0 | 10.1.12.2/30 | Yes |
| P1 — P2 | P1 | ge-0/0/1 | 10.1.23.1/30 | Yes |
| P1 — P2 | P2 | ge-0/0/0 | 10.1.23.2/30 | Yes |
| P2 — PE2 | P2 | ge-0/0/1 | 10.1.34.1/30 | Yes |
| P2 — PE2 | PE2 | ge-0/0/0 | 10.1.34.2/30 | Yes |
| PE2 — CE2 | PE2 | ge-0/0/1 | 172.16.2.1/30 | No |
| PE2 — CE2 | CE2 | ge-0/0/0 | 172.16.2.2/30 | No |

## LDP Session Summary

LDP sessions form between directly connected routers on enabled interfaces. LDP uses the loopback address as its **LSR-ID** (router identifier in the LDP session).

| LDP Session | Router A | LSR-ID A | Router B | LSR-ID B |
|-------------|----------|----------|----------|----------|
| PE1 — P1 | PE1 | 10.0.0.1:0 | P1 | 10.0.0.2:0 |
| P1 — P2 | P1 | 10.0.0.2:0 | P2 | 10.0.0.3:0 |
| P2 — PE2 | P2 | 10.0.0.3:0 | PE2 | 10.0.0.4:0 |

!!! note "LSR-ID format"
    The `:0` suffix in LDP LSR-IDs (e.g., `10.0.0.1:0`) is the label space identifier. A value of `:0` indicates a platform-wide label space — all interfaces share the same label range. This is the default for all Junos routers.
