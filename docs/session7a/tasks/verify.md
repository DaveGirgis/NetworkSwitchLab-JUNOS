# Session 7a — Verification Checklist

## Part 0 — Baseline and CE Interfaces

- [ ] `PE1> show ldp session` — `10.0.0.2` shows `Operational` (link LDP to P1 from Session 7)
- [ ] `PE1> show route table inet.3` — three LDP routes present for 10.0.0.2, 10.0.0.3, 10.0.0.4
- [ ] `CE1> show interfaces ge-0/0/1 terse` — `ge-0/0/1.0` Up/Up with address 192.168.1.1/24
- [ ] `CE2> show interfaces ge-0/0/1 terse` — `ge-0/0/1.0` Up/Up with address 192.168.1.2/24
- [ ] `CE1> ping 192.168.1.2 count 3` — 100% packet loss (L2 service not yet configured)

## Part 1 — L2circuit

### Pseudowire State

- [ ] `PE1> show l2circuit connections` — neighbor 10.0.0.4, ge-0/0/2.0 (vc 100), State: `Up`
- [ ] `PE2> show l2circuit connections` — neighbor 10.0.0.1, ge-0/0/2.0 (vc 100), State: `Up`
- [ ] `PE1> show l2circuit connections` — shows `Incoming label` and `Outgoing label` (non-zero values)
- [ ] `PE1> show l2circuit connections` — `Encapsulation: ETHERNET`

### Targeted LDP Session

- [ ] `PE1> show ldp session` — two sessions listed: 10.0.0.2 (link LDP to P1) and 10.0.0.4 (targeted LDP to PE2)
- [ ] `PE2> show ldp session` — two sessions listed: 10.0.0.3 (link LDP to P2) and 10.0.0.1 (targeted LDP to PE1)

### End-to-End Connectivity

- [ ] `CE1> ping 192.168.1.2 count 5` — 5/5 replies, 0% packet loss
- [ ] `CE2> ping 192.168.1.1 count 5` — 5/5 replies, 0% packet loss

## Part 2 — VPLS

### L2circuit Removed

- [ ] `PE1> show l2circuit connections` — no output (L2circuit deleted)
- [ ] `PE2> show l2circuit connections` — no output

### BGP L2VPN Family

- [ ] `PE1> show bgp neighbor 10.0.0.4 | match NLRI` — `l2vpn` appears in negotiated NLRI families
- [ ] `PE2> show bgp neighbor 10.0.0.1 | match NLRI` — `l2vpn` appears in negotiated NLRI families

### VPLS Instance State

- [ ] `PE1> show vpls connections` — connection-site 2 (PE2's site), State: `Up`
- [ ] `PE2> show vpls connections` — connection-site 1 (PE1's site), State: `Up`
- [ ] `PE1> show route table VPLS-100.l2vpn.0` — two routes present (PE1's own route + BGP route from PE2)

### MAC Table

- [ ] After pinging: `PE1> show vpls mac-table instance VPLS-100` — two entries present
- [ ] After pinging: CE1's MAC shows interface `ge-0/0/2.0` (locally learned)
- [ ] After pinging: CE2's MAC shows interface `lsi.1048576` (learned via VPLS pseudowire)

### End-to-End Connectivity

- [ ] `CE1> ping 192.168.1.2 count 5` — 5/5 replies, 0% packet loss
- [ ] `CE2> ping 192.168.1.1 count 5` — 5/5 replies, 0% packet loss

## Quick Commands

```junos
show l2circuit connections
```
Expected in Part 1: neighbor 10.0.0.4, State Up on PE1.

```junos
show ldp session
```
Expected after Part 1: two sessions on each PE — one link LDP, one targeted LDP.

```junos
show vpls connections
```
Expected in Part 2: one remote connection per PE, State Up.

```junos
show vpls mac-table instance VPLS-100
```
Expected after traffic: CE1 MAC as local (DL), CE2 MAC as remote (R) on PE1.
