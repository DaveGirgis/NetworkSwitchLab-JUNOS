# Session 3a — Verification Checklist

Complete these checks before moving to Session 4.

## Topology

- [ ] Second trunk link is drawn in GNS3 (SW1 Adapter 5 ↔ SW2 Adapter 5)
- [ ] ge-0/0/3 trunk subunits (.10 and .11) are configured on both SW1 and SW2
- [ ] `show bridge domain` shows three interfaces per VLAN bridge domain (two trunks + one access port)

## RSTP

- [ ] `show spanning-tree bridge` shows SW1 as root bridge (priority 4096)
- [ ] `show spanning-tree bridge` on SW2 confirms it is not root and identifies the Root port
- [ ] `show spanning-tree interface` on SW2 shows one trunk port as Alternate/Discarding
- [ ] `show spanning-tree interface` shows access ports as Edge/Forwarding

## Connectivity

- [ ] PC1 can ping PC3 (192.168.10.2) with STP converged
- [ ] PC2 can ping PC4 (192.168.11.2) with STP converged
- [ ] After cutting the active trunk, pings recover in under 3 seconds
- [ ] After restoring the trunk, STP returns to original topology

## Quick Commands

```junos
show spanning-tree bridge
```
Expected on SW1: `This bridge is the root`, priority 4096

```junos
show spanning-tree interface
```
Expected on SW2: one trunk port in `Discarding / Alternate`, access ports in `Forwarding / Edge`

```junos
show bridge domain
```
Expected: VLAN10 and VLAN11 each showing three interfaces (two trunk subunits + one access port)
