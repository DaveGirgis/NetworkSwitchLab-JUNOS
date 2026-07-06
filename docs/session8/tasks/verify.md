# Session 8 — Verification Checklist

## Part 0 — Baseline

- [ ] `PE1> show ldp session` — `10.0.0.2` shows `Operational`
- [ ] `PE1> show route table inet.3` — three entries; 10.0.0.4/32 shows `Push <label>`
- [ ] `PE1> show bgp summary` — both `10.0.0.4` (iBGP) and `172.16.1.2` (eBGP) are established

## Part 1 — VRF & Route Distinguishers

### Global eBGP Removed

- [ ] `PE1> show bgp summary` — `172.16.1.2` no longer appears (EBGP-CE1 group deleted)
- [ ] `PE2> show bgp summary` — `172.16.2.2` no longer appears (EBGP-CE2 group deleted)

### VRF Created

- [ ] `PE1> show route table VPN-A.inet.0` — table exists; shows `172.16.1.0/30` as Direct route
- [ ] `PE2> show route table VPN-A.inet.0` — table exists; shows `172.16.2.0/30` as Direct route
- [ ] `PE1> show route table inet.0 172.16.1.0/30` — no output (route is now in VRF, not inet.0)
- [ ] `PE2> show route table inet.0 172.16.2.0/30` — no output

## Part 2 — MP-BGP & Route Targets

### NLRI Negotiation

- [ ] `PE1> show bgp neighbor 10.0.0.4 | match NLRI` — `inet-vpn` appears in all three NLRI lines
- [ ] `PE2> show bgp neighbor 10.0.0.1 | match NLRI` — `inet-vpn` appears in all three NLRI lines

### iBGP Session Still Up

- [ ] `PE1> show bgp summary` — `10.0.0.4` remains established (not `Active`)

## Part 3 — PE-CE Routing

### VRF BGP Sessions

- [ ] `PE1> show bgp summary instance VPN-A` — `172.16.1.2` (CE1) shows established, at least 1 prefix received
- [ ] `PE2> show bgp summary instance VPN-A` — `172.16.2.2` (CE2) shows established, at least 1 prefix received

### VRF Routing Table

- [ ] `PE1> show route table VPN-A.inet.0` — four routes present: 172.16.1.0/30 (Direct), 10.0.0.11/32 (BGP from CE1), 10.0.0.12/32 (BGP from PE2), 172.16.2.0/30 (BGP from PE2)
- [ ] `PE2> show route table VPN-A.inet.0` — four routes present: 172.16.2.0/30 (Direct), 10.0.0.12/32 (BGP from CE2), 10.0.0.11/32 (BGP from PE1), 172.16.1.0/30 (BGP from PE1)
- [ ] `PE1> show route table VPN-A.inet.0` — `10.0.0.12/32` shows `Push <label>, Push <label>` (two-label stack)

### bgp.l3vpn.0

- [ ] `PE1> show route table bgp.l3vpn.0` — two routes: `65001:1000:10.0.0.11/32` and `65001:2000:10.0.0.12/32`
- [ ] `PE2> show route table bgp.l3vpn.0` — two routes: `65001:1000:10.0.0.11/32` and `65001:2000:10.0.0.12/32`

### No Route Leaking

- [ ] `PE1> show route table inet.0 10.0.0.11` — no output
- [ ] `PE1> show route table inet.0 10.0.0.12` — no output
- [ ] `PE2> show route table inet.0 10.0.0.11` — no output
- [ ] `PE2> show route table inet.0 10.0.0.12` — no output

### CE Receives Remote Prefix

- [ ] `CE1> show route receive-protocol bgp 172.16.1.1` — `10.0.0.12/32` present with AS path `65001 65001 I`
- [ ] `CE2> show route receive-protocol bgp 172.16.2.1` — `10.0.0.11/32` present with AS path `65001 65001 I`

### End-to-End Ping

- [ ] `CE1> ping 10.0.0.12 source 10.0.0.11 count 5` — 5/5 replies, 0% packet loss
- [ ] `CE2> ping 10.0.0.11 source 10.0.0.12 count 5` — 5/5 replies, 0% packet loss

## Quick Commands

```junos
show route table VPN-A.inet.0
```
Expected on PE1: four routes including 10.0.0.12/32 with two-label Push stack.

```junos
show route table bgp.l3vpn.0
```
Expected: two VPN-IPv4 entries in `RD:prefix` format.

```junos
show bgp summary instance VPN-A
```
Expected: CE peer established with prefix counts > 0.

```junos
show route table VPN-A.inet.0 10.0.0.12 detail | match Label
```
Expected on PE1: `Label operation: Push <vpn-label>, Push 299808` — inner VPN label followed by outer LDP label.
