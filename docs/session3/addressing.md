# Session 3 — Addressing

## VLAN Assignment

| VLAN ID | Name | Subnet | Purpose |
|---------|------|--------|---------|
| 10 | VLAN10 | 192.168.10.0/24 | PC1 and PC3 segment |
| 11 | VLAN11 | 192.168.11.0/24 | PC2 and PC4 segment |

## Host Addresses

| Device | Switch | VLAN | IP Address | Default Gateway |
|--------|--------|------|-----------|-----------------|
| PC1 | SW1 | 10 | 192.168.10.1/24 | 192.168.10.254 (IRB, Part 3) |
| PC2 | SW1 | 11 | 192.168.11.1/24 | 192.168.11.254 (IRB, Part 3) |
| PC3 | SW2 | 10 | 192.168.10.2/24 | 192.168.10.254 (IRB, Part 3) |
| PC4 | SW2 | 11 | 192.168.11.2/24 | 192.168.11.254 (IRB, Part 3) |

## IRB Addresses (Part 3)

| Interface | Device | IP Address | VLAN |
|-----------|--------|-----------|------|
| irb.10 | SW1 | 192.168.10.254/24 | 10 |
| irb.11 | SW1 | 192.168.11.254/24 | 11 |

## Interface Roles

| Device | Interface | Adapter | Role |
|--------|-----------|---------|------|
| SW1 | ge-0/0/0 | 2 | Trunk to SW2 (VLANs 10, 11) |
| SW1 | ge-0/0/1 | 3 | Access port, VLAN 10 → PC1 |
| SW1 | ge-0/0/2 | 4 | Access port, VLAN 11 → PC2 |
| SW2 | ge-0/0/0 | 2 | Trunk to SW1 (VLANs 10, 11) |
| SW2 | ge-0/0/1 | 3 | Access port, VLAN 10 → PC3 |
| SW2 | ge-0/0/2 | 4 | Access port, VLAN 11 → PC4 |

!!! note "No routing addresses on switch interfaces"
    In Parts 1 and 2 the vMX interfaces carry only bridged traffic — no `family inet` addresses are needed. IP addresses are added only in Part 3 via IRB interfaces.
