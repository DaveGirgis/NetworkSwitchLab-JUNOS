# Session 3a — Command Reference

## STP Show Commands

| Command | Description |
|---------|-------------|
| `show spanning-tree bridge` | Root bridge ID, priority, hello/age/forward timers |
| `show spanning-tree interface` | Port roles and states for all STP-active interfaces |
| `show spanning-tree statistics` | BPDU counters per interface |
| `show spanning-tree statistics interface ge-0/0/0.10` | BPDU stats for one interface |

## STP Configuration Hierarchy

### Enable RSTP globally (vMX 14.1)
```junos
[edit protocols rstp]
  bridge-priority 4096;   /* SW1 root — lower value = preferred */
  interface ge-0/0/0;     /* Trunk 1 — participates in STP topology */
  interface ge-0/0/3;     /* Trunk 2 — participates in STP topology */
  interface ge-0/0/1 {
    edge;                 /* access port — skip state machine, forward immediately */
  }
  interface ge-0/0/2 {
    edge;
  }
```
```junos
[edit protocols rstp]
  bridge-priority 8192;   /* SW2 non-root — use 8k shorthand in set mode */
  interface ge-0/0/0;
  interface ge-0/0/3;
```

Note: on vMX 14.1, `bridge-priority` must be specified explicitly and each trunk interface must be added individually with `set protocols rstp interface <name>`. `protocols rstp` is global; there is no per-bridge-domain STP stanza.

### Mark an interface as edge (access ports)
```junos
[edit protocols rstp]
  interface ge-0/0/1 {
    edge;
    no-root-port;
  }
```

### Set interface as point-to-point (trunk ports)
```junos
[edit protocols rstp]
  interface ge-0/0/0 {
    mode point-to-point;
  }
```

## Bridge Priority Values

STP bridge priority must be a multiple of 4096:

| Priority Value | Hex | Notes |
|---------------|-----|-------|
| 4096 | 0x1000 | Lowest common value — use for root bridge |
| 8192 | 0x2000 | Secondary root / backup root |
| 32768 | 0x8000 | Default — non-root bridges |
| 61440 | 0xF000 | Highest — never becomes root |

## RSTP Port Roles and States

| Role | State | Meaning |
|------|-------|---------|
| Root | Forwarding | Best path to root, active |
| Designated | Forwarding | Best port on this segment toward root |
| Alternate | Discarding | Redundant path, blocked |
| Edge | Forwarding | End-host port, bypasses STP states |

## Wrap-Up

After this session you have seen Layer 2 loop prevention in action. RSTP's proposal/agreement mechanism ensures fast sub-second failover — a critical property for production switching environments.

Session 4 returns to Layer 3 with OSPF on the SP core topology.
