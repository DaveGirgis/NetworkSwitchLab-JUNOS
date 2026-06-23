# Session 6 — Addressing Table

## Loopback Addresses

| Device | Interface | Address | Role |
|--------|-----------|---------|------|
| CE1 | lo0.0 | 10.0.0.11/32 | Customer loopback — advertised into BGP |
| PE1 | lo0.0 | 10.0.0.1/32 | Provider edge — used as iBGP source |
| P1 | lo0.0 | 10.0.0.2/32 | Provider core — IS-IS only |
| P2 | lo0.0 | 10.0.0.3/32 | Provider core — IS-IS only |
| PE2 | lo0.0 | 10.0.0.4/32 | Provider edge — used as iBGP source |
| CE2 | lo0.0 | 10.0.0.12/32 | Customer loopback — advertised into BGP |

## Point-to-Point Link Addresses

| Link | Device | Interface | Address |
|------|--------|-----------|---------|
| CE1 — PE1 | CE1 | ge-0/0/0 | 172.16.1.2/30 |
| CE1 — PE1 | PE1 | ge-0/0/1 | 172.16.1.1/30 |
| PE1 — P1 | PE1 | ge-0/0/0 | 10.1.12.1/30 |
| PE1 — P1 | P1 | ge-0/0/0 | 10.1.12.2/30 |
| P1 — P2 | P1 | ge-0/0/1 | 10.1.23.1/30 |
| P1 — P2 | P2 | ge-0/0/0 | 10.1.23.2/30 |
| P2 — PE2 | P2 | ge-0/0/1 | 10.1.34.1/30 |
| P2 — PE2 | PE2 | ge-0/0/0 | 10.1.34.2/30 |
| PE2 — CE2 | PE2 | ge-0/0/1 | 172.16.2.1/30 |
| PE2 — CE2 | CE2 | ge-0/0/0 | 172.16.2.2/30 |

## BGP Parameters

| Parameter | Value |
|-----------|-------|
| Provider ASN | 65001 |
| Customer ASN | 65100 |
| iBGP source (PE1) | 10.0.0.1 (loopback) |
| iBGP source (PE2) | 10.0.0.4 (loopback) |
| eBGP hold time | 90s (default) |
| iBGP hold time | 90s (default) |
| Route preference | 170 (all BGP routes in Junos) |

!!! note "172.16.x.x for PE-CE links"
    The PE-CE links use 172.16.0.0/12 (RFC 1918 private) rather than 10.1.x.x to clearly distinguish customer-facing interfaces from provider core interfaces. This mirrors real SP addressing practice.
