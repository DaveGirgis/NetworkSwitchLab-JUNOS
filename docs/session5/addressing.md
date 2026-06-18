# Session 5 — Addressing Table

## IP Addressing (unchanged from Session 4)

| Device | Interface | Address |
|--------|-----------|---------|
| PE1 | lo0.0 | 10.0.0.1/32 |
| P1 | lo0.0 | 10.0.0.2/32 |
| P2 | lo0.0 | 10.0.0.3/32 |
| PE2 | lo0.0 | 10.0.0.4/32 |
| PE1 | ge-0/0/0 | 10.1.12.1/30 |
| P1 | ge-0/0/0 | 10.1.12.2/30 |
| P1 | ge-0/0/1 | 10.1.23.1/30 |
| P2 | ge-0/0/0 | 10.1.23.2/30 |
| P2 | ge-0/0/1 | 10.1.34.1/30 |
| PE2 | ge-0/0/0 | 10.1.34.2/30 |

## IS-IS NET Addresses

A **NET** (Network Entity Title) is IS-IS's equivalent of a router ID. It has three parts:

```
49 . 0001 . 0100.0000.0001 . 00
^     ^      ^                ^
AFI   Area   System ID        SEL (always 00)
```

- **AFI 49** — private/lab address space (like RFC 1918 for IP)
- **Area 0001** — all four routers are in the same area
- **System ID** — derived from the loopback IP by padding each octet to 3 digits and grouping into 4-digit hex fields:
  `10.0.0.1` → `010.000.001` → `0100.0000.0001`

| Device | Loopback | NET |
|--------|----------|-----|
| PE1 | 10.0.0.1/32 | 49.0001.0100.0000.0001.00 |
| P1 | 10.0.0.2/32 | 49.0001.0100.0000.0002.00 |
| P2 | 10.0.0.3/32 | 49.0001.0100.0000.0003.00 |
| PE2 | 10.0.0.4/32 | 49.0001.0100.0000.0004.00 |

## IS-IS Parameters

| Parameter | Value |
|-----------|-------|
| Area | 49.0001 |
| Level | Level 2 only (Level 1 disabled) |
| Interface type | point-to-point on all transit links |
| Default metric | 10 per interface |
| Route preference | 18 (IS-IS L2 internal) |

!!! note "Level 2 only"
    A single-area backbone with no customer sites attached only needs Level 2. Disabling Level 1 simplifies the configuration and matches real-world SP practice where the backbone is a flat Level 2 domain.
