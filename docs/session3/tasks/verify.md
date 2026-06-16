# Session 3 — Verification Checklist

Complete these checks before moving to Session 4.

## Platform

- [ ] Both SW1 and SW2 show `Network Services Mode: Enhanced-Ethernet` (`show chassis network-services`)
- [ ] Both switches have VLAN10 and VLAN11 bridge domains defined (`show bridge domain`)

## Access Ports

- [ ] SW1 ge-0/0/1.0 is in VLAN10 bridge domain and shows `up up Bridge`
- [ ] SW1 ge-0/0/2.0 is in VLAN11 bridge domain and shows `up up Bridge`
- [ ] SW2 ge-0/0/1.0 is in VLAN10 bridge domain and shows `up up Bridge`
- [ ] SW2 ge-0/0/2.0 is in VLAN11 bridge domain and shows `up up Bridge`
- [ ] PC1 has IP 192.168.10.1/24 with gateway 192.168.10.254
- [ ] PC2 has IP 192.168.11.1/24 with gateway 192.168.11.254
- [ ] PC3 has IP 192.168.10.2/24 with gateway 192.168.10.254
- [ ] PC4 has IP 192.168.11.2/24 with gateway 192.168.11.254

## Trunk

- [ ] SW1 and SW2 ge-0/0/0 trunk is up with subunits .10 and .11 showing `up up Bridge`
- [ ] Both VLAN10 and VLAN11 bridge domains show 2 interfaces on each switch (access + trunk)
- [ ] PC1 (192.168.10.1) can ping PC3 (192.168.10.2) — same VLAN across trunk
- [ ] PC2 (192.168.11.1) can ping PC4 (192.168.11.2) — same VLAN across trunk
- [ ] PC1 cannot ping PC2 (different VLANs, no routing yet)

## IRB / Inter-VLAN Routing

- [ ] SW1 irb.10 shows `192.168.10.254/24` and is `up up`
- [ ] SW1 irb.11 shows `192.168.11.254/24` and is `up up`
- [ ] All four PCs can ping their default gateway (192.168.10.254 or 192.168.11.254)
- [ ] PC1 can ping PC2 (192.168.11.1) — inter-VLAN routing works
- [ ] PC3 can ping PC4 (192.168.11.2) — inter-VLAN routing works across trunk

## Quick Commands

```junos
show chassis network-services
```
Expected: `Network Services Mode: Enhanced-Ethernet`

```junos
show bridge domain
```
Expected: VLAN10 and VLAN11 each showing 2 interfaces and 1 IRB interface (after Part 3)

```junos
show bridge mac-table
```
Expected: MAC addresses for all four PCs in their respective bridge domains after traffic is generated

```junos
show interfaces irb terse
```
Expected: irb.10 and irb.11 both `up up inet`

```junos
show route
```
Expected: 192.168.10.0/24 and 192.168.11.0/24 as Direct routes via irb.10 and irb.11
