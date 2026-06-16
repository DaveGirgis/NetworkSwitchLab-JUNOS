# Part 2 — Trunk Configuration

A **trunk** carries multiple VLANs over a single link using 802.1Q tagging. In Junos, a trunk port is built by enabling `flexible-vlan-tagging` on the physical interface and creating a logical subunit per VLAN with `encapsulation vlan-bridge`.

## Step 1: Draw the trunk link in GNS3

1. Click **Add a link**
2. Click **SW1** → select **Adapter 2** (ge-0/0/0)
3. Click **SW2** → select **Adapter 2** (ge-0/0/0)

## Step 2: Configure the trunk on SW1

```junos
configure

set interfaces ge-0/0/0 flexible-vlan-tagging
set interfaces ge-0/0/0 encapsulation flexible-ethernet-services

set interfaces ge-0/0/0 unit 10 encapsulation vlan-bridge
set interfaces ge-0/0/0 unit 10 vlan-id 10

set interfaces ge-0/0/0 unit 11 encapsulation vlan-bridge
set interfaces ge-0/0/0 unit 11 vlan-id 11

set bridge-domains VLAN10 interface ge-0/0/0.10
set bridge-domains VLAN11 interface ge-0/0/0.11

commit
```

## Step 3: Configure the trunk on SW2

Apply the same trunk config on SW2 (both ends must match):

```junos
configure

set interfaces ge-0/0/0 flexible-vlan-tagging
set interfaces ge-0/0/0 encapsulation flexible-ethernet-services

set interfaces ge-0/0/0 unit 10 encapsulation vlan-bridge
set interfaces ge-0/0/0 unit 10 vlan-id 10

set interfaces ge-0/0/0 unit 11 encapsulation vlan-bridge
set interfaces ge-0/0/0 unit 11 vlan-id 11

set bridge-domains VLAN10 interface ge-0/0/0.10
set bridge-domains VLAN11 interface ge-0/0/0.11

commit
```

## Step 4: Verify the trunk is up

```junos
show interfaces ge-0/0/0 terse
```

Expected on both switches:

```text
Interface               Admin Link Proto    Local                 Remote
ge-0/0/0                up    up
ge-0/0/0.10             up    up   Bridge
ge-0/0/0.11             up    up   Bridge
```

```junos
show bridge domain
```

Expected on SW1 (and mirrored on SW2):

```text
Routing instance        Bridge domain            Intfs  IRB intfs  MAC ageing
default-switch          VLAN10                   2          -          300
default-switch          VLAN11                   2          -          300
```

Each bridge domain now shows 2 interfaces: the local access port and the trunk subunit.

## Step 5: Test same-VLAN communication across the trunk

From PC1 (192.168.10.1), ping PC3 (192.168.10.2) — both are in VLAN 10:

```text
ping 192.168.10.2
```

Expected: replies from PC3. The trunk is carrying VLAN 10 between the switches.

From PC2 (192.168.11.1), ping PC4 (192.168.11.2) — both are in VLAN 11:

```text
ping 192.168.11.2
```

Expected: replies from PC4.

## Step 6: Confirm VLAN isolation

From PC1 (VLAN 10), attempt to ping PC2 (VLAN 11, 192.168.11.1):

```text
ping 192.168.11.1
```

Expected: **no reply**. PC1 and PC2 are in different bridge domains — they cannot communicate at Layer 2 even though they are connected to the same physical switch.

This is the core purpose of VLANs: **traffic separation on shared infrastructure**.

Cross-VLAN communication requires a Layer 3 device. That is added in Part 3.

!!! note "MAC learning"
    After the pings succeed, you can view learned MAC addresses per VLAN:
    ```junos
    show bridge mac-table
    ```
