# Part 1 — Bridge Domains and Access Ports

A **bridge domain** in Junos is the equivalent of a VLAN on a traditional switch. It defines a Layer 2 forwarding domain identified by a VLAN ID. Interfaces added to a bridge domain forward traffic at Layer 2 within that domain only.

## Concepts

| Term | Junos equivalent | Cisco equivalent |
|------|-----------------|-----------------|
| VLAN / broadcast domain | Bridge domain | VLAN |
| Access port (untagged) | `interface-mode access` under `family bridge` | `switchport mode access` |
| Trunk port (tagged) | `flexible-vlan-tagging` + `encapsulation vlan-bridge` | `switchport mode trunk` |
| SVI / VLAN interface | IRB (Integrated Routing and Bridging) | `interface vlan X` |

## Step 1: Add VPCS nodes in GNS3

1. Drag four **VPCS** nodes onto the canvas and label them `PC1`, `PC2`, `PC3`, `PC4`
2. Connect PC1 to SW1 **Adapter 3** (ge-0/0/1)
3. Connect PC2 to SW1 **Adapter 4** (ge-0/0/2)
4. Connect PC3 to SW2 **Adapter 3** (ge-0/0/1)
5. Connect PC4 to SW2 **Adapter 4** (ge-0/0/2)
6. Do **not** connect the trunk yet — that is Part 2

## Step 2: Define bridge domains on SW1

```junos
configure

set bridge-domains VLAN10 vlan-id 10
set bridge-domains VLAN10 description "VLAN 10 segment"
set bridge-domains VLAN11 vlan-id 11
set bridge-domains VLAN11 description "VLAN 11 segment"

commit
```

Repeat on **SW2** — both switches must define the same bridge domains for the trunk in Part 2 to work.

## Step 3: Configure access ports on SW1

VLAN 10 access port for PC1 (ge-0/0/1):

```junos
configure

set interfaces ge-0/0/1 encapsulation ethernet-bridge
set interfaces ge-0/0/1 unit 0 family bridge interface-mode access
set interfaces ge-0/0/1 unit 0 family bridge vlan-id 10

commit
```

VLAN 11 access port for PC2 (ge-0/0/2):

```junos
configure

set interfaces ge-0/0/2 encapsulation ethernet-bridge
set interfaces ge-0/0/2 unit 0 family bridge interface-mode access
set interfaces ge-0/0/2 unit 0 family bridge vlan-id 11

commit
```

!!! note "No explicit bridge-domain interface statement for access ports"
    When `interface-mode access` is set with a `vlan-id`, Junos automatically associates the interface with the bridge domain that matches that VLAN ID. Adding `set bridge-domains VLAN10 interface ge-0/0/1.0` explicitly will cause a commit error. The explicit interface statement is only required for trunk subunits (Part 2).

## Step 4: Configure access ports on SW2

VLAN 10 access port for PC3 (ge-0/0/1):

```junos
configure

set interfaces ge-0/0/1 encapsulation ethernet-bridge
set interfaces ge-0/0/1 unit 0 family bridge interface-mode access
set interfaces ge-0/0/1 unit 0 family bridge vlan-id 10

commit
```

VLAN 11 access port for PC4 (ge-0/0/2):

```junos
configure

set interfaces ge-0/0/2 encapsulation ethernet-bridge
set interfaces ge-0/0/2 unit 0 family bridge interface-mode access
set interfaces ge-0/0/2 unit 0 family bridge vlan-id 11

commit
```

## Step 5: Assign IPs to VPCS nodes

Open each VPCS console (right-click > Console in GNS3) and apply:

**PC1** (SW1, VLAN 10):
```text
ip 192.168.10.1 192.168.10.254 24
```

**PC2** (SW1, VLAN 11):
```text
ip 192.168.11.1 192.168.11.254 24
```

**PC3** (SW2, VLAN 10):
```text
ip 192.168.10.2 192.168.10.254 24
```

**PC4** (SW2, VLAN 11):
```text
ip 192.168.11.2 192.168.11.254 24
```

The gateway addresses (`.254`) will not resolve until Part 3 when the IRB interface is added.

## Verify

```junos
show bridge domain
```

Expected on SW1:

```text
Routing instance        Bridge domain            VLAN ID     Interfaces
default-switch          VLAN10                   10
                                                     ge-0/0/1.0
default-switch          VLAN11                   11
                                                     ge-0/0/2.0
```

Each bridge domain shows its local access port. The trunk has not been added yet.

```junos
show interfaces ge-0/0/1 terse
show interfaces ge-0/0/2 terse
```

Expected on SW1:

```text
Interface               Admin Link Proto    Local                 Remote
ge-0/0/1                up    up
ge-0/0/1.0              up    up   Bridge
ge-0/0/2                up    up
ge-0/0/2.0              up    up   Bridge
```

At this stage, no cross-switch communication is possible — the trunk is not yet configured.
