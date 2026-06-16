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

Apply the same trunk config on SW2 (identical — both ends must match):

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

## Step 4: Test VLAN 10 reachability

From PC1, ping SW1's VLAN 10 IRB (not yet configured) — skip this for now. Instead, verify the trunk is up:

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

Expected on SW1:

```text
Routing instance        Bridge domain            Intfs  IRB intfs  MAC ageing
default-switch          VLAN10                   2          -          300
default-switch          VLAN11                   1          -          300
```

VLAN10 now shows 2 interfaces: ge-0/0/1.0 (access to PC1) and ge-0/0/0.10 (trunk to SW2).

## Step 5: Verify VLAN isolation

VLAN 10 and VLAN 11 are separate bridge domains — traffic does not cross between them. PC1 (VLAN 10) and PC2 (VLAN 11) are isolated at Layer 2 even though they share the same trunk link.

This is the core purpose of VLANs: **traffic separation on shared infrastructure**.

Inter-VLAN communication requires a Layer 3 device (a router or IRB interface). That is configured in Part 3.

!!! note "MAC learning"
    Once PC1 and PC2 generate traffic (e.g., pings after Part 3 adds IRB), you can watch MAC addresses populate:
    ```junos
    show bridge mac-table
    ```
