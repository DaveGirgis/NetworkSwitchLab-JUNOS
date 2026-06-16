# Session 3a — Addressing

## IP Addressing

No new IP addresses are introduced in this session. PC addresses and IRB gateway addresses remain identical to Session 3.

| Device | VLAN | IP Address | Gateway |
|--------|------|-----------|---------|
| PC1 | 10 | 192.168.10.1/24 | 192.168.10.254 |
| PC2 | 11 | 192.168.11.1/24 | 192.168.11.254 |
| PC3 | 10 | 192.168.10.2/24 | 192.168.10.254 |
| PC4 | 11 | 192.168.11.2/24 | 192.168.11.254 |

## New Interface Roles (Session 3a additions)

| Device | Interface | Adapter | Role |
|--------|-----------|---------|------|
| SW1 | ge-0/0/3 | 5 | Trunk 2 to SW2 (VLANs 10, 11) |
| SW2 | ge-0/0/3 | 5 | Trunk 2 to SW1 (VLANs 10, 11) |

## STP Parameters

| Parameter | SW1 | SW2 |
|-----------|-----|-----|
| Bridge priority | 4096 (root) | 32768 (default) |
| Hello time | 2s (default) | 2s (default) |
| Max age | 20s (default) | 20s (default) |
| Forward delay | 15s (default) | 15s (default) |
