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

1. In GNS3, drag two **VPCS** nodes onto the canvas
2. Label them `PC1` and `PC2`
3. Connect PC1 to SW1 **Adapter 3** (ge-0/0/1)
4. Connect PC2 to SW2 **Adapter 3** (ge-0/0/1)
5. Do **not** connect the trunk yet — that is Part 2

## Step 2: Define bridge domains on SW1

```junos
configure

set bridge-domains VLAN10 vlan-id 10
set bridge-domains VLAN10 description "PC1 segment"
set bridge-domains VLAN11 vlan-id 11
set bridge-domains VLAN11 description "PC2 segment"

commit
```

Repeat on **SW2** — both switches must define the same bridge domains for the trunk in Part 2 to work.

## Step 3: Configure the access port on SW1 (VLAN 10 → PC1)

```junos
configure

set interfaces ge-0/0/1 encapsulation ethernet-bridge
set interfaces ge-0/0/1 unit 0 family bridge interface-mode access
set interfaces ge-0/0/1 unit 0 family bridge vlan-id 10
set bridge-domains VLAN10 interface ge-0/0/1.0

commit
```

## Step 4: Configure the access port on SW2 (VLAN 11 → PC2)

```junos
configure

set interfaces ge-0/0/1 encapsulation ethernet-bridge
set interfaces ge-0/0/1 unit 0 family bridge interface-mode access
set interfaces ge-0/0/1 unit 0 family bridge vlan-id 11
set bridge-domains VLAN11 interface ge-0/0/1.0

commit
```

## Step 5: Assign IPs to VPCS nodes

Open the console for PC1 (right-click > Console in GNS3):

```text
ip 192.168.10.1 192.168.10.254 24
```

Open the console for PC2:

```text
ip 192.168.11.1 192.168.11.254 24
```

The second argument (`192.168.10.254`) is the default gateway — it won't resolve until Part 3 when the IRB interface is added.

## Verify

```junos
show bridge domain
```

Expected on SW1:

```text
Routing instance        Bridge domain            Intfs  IRB intfs  MAC ageing
default-switch          VLAN10                   1          -          300
default-switch          VLAN11                   0          -          300
```

VLAN10 shows 1 interface (ge-0/0/1.0). VLAN11 shows 0 because the PC2 access port is on SW2.

```junos
show interfaces ge-0/0/1 terse
```

Expected:

```text
Interface               Admin Link Proto    Local                 Remote
ge-0/0/1                up    up
ge-0/0/1.0              up    up   Bridge
```

At this stage, PC1 and PC2 cannot communicate — the trunk between SW1 and SW2 is not yet configured.
