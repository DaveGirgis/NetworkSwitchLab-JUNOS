# Session 3 — Command Reference

## Network Services

| Command | Description |
|---------|-------------|
| `show chassis network-services` | Show current mode (Enhanced-IP or Enhanced-Ethernet) |
| `set chassis network-services enhanced-ethernet` | Enable bridge domain support (requires reboot) |

## Bridge Domain Commands

| Command | Description |
|---------|-------------|
| `show bridge domain` | List all bridge domains, interface counts, IRB status |
| `show bridge mac-table` | Show learned MAC addresses per bridge domain |
| `show bridge mac-table bridge-domain VLAN10` | MAC table for a specific domain |
| `show l2-learning global-information` | Global L2 learning state |
| `clear bridge mac-table` | Flush all learned MAC addresses |

## Interface Commands

| Command | Description |
|---------|-------------|
| `show interfaces ge-0/0/0 terse` | Physical + subunits, admin/link state |
| `show interfaces irb terse` | All IRB interfaces and their IPs |
| `show interfaces ge-0/0/0 detail` | Full counters, encapsulation type |

## Configuration Hierarchy

### Bridge domain
```junos
[edit bridge-domains VLAN10]
  vlan-id 10;
  description "...";
  interface ge-0/0/1.0;
  interface ge-0/0/0.10;
  routing-interface irb.10;   /* Part 3 only */
```

### Access port
```junos
[edit interfaces ge-0/0/1]
  encapsulation ethernet-bridge;
  unit 0 {
    family bridge {
      interface-mode access;
      vlan-id 10;
    }
  }
```

### Trunk port
```junos
[edit interfaces ge-0/0/0]
  flexible-vlan-tagging;
  encapsulation flexible-ethernet-services;
  unit 10 {
    encapsulation vlan-bridge;
    vlan-id 10;
  }
  unit 11 {
    encapsulation vlan-bridge;
    vlan-id 11;
  }
```

### IRB interface
```junos
[edit interfaces irb]
  unit 10 {
    family inet {
      address 192.168.10.254/24;
    }
  }
```

## Wrap-Up

After this session, the vMX is configured as a full Layer 2 switch with:

- Two isolated broadcast domains (VLAN 10 and VLAN 11)
- 802.1Q trunking between two switches
- IRB interfaces providing Layer 3 inter-VLAN routing

Session 4 returns to Layer 3 with OSPF — the SP core topology begins there and remains constant through Session 8.
