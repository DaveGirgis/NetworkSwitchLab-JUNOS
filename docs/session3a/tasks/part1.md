# Part 1 — Add Second Trunk

With RSTP already active, adding the second trunk creates a physical loop that RSTP will immediately detect and resolve by blocking one of the redundant paths. No broadcast storm will occur.

## Step 1: Draw the second trunk in GNS3

1. Click **Add a link**
2. Click **SW1** → select **Adapter 5** (ge-0/0/3)
3. Click **SW2** → select **Adapter 5** (ge-0/0/3)

The link appears in GNS3 but carries no traffic until configured on both switches.

## Step 2: Configure Trunk 2 on SW1

```junos
configure

set interfaces ge-0/0/3 flexible-vlan-tagging
set interfaces ge-0/0/3 encapsulation flexible-ethernet-services

set interfaces ge-0/0/3 unit 10 encapsulation vlan-bridge
set interfaces ge-0/0/3 unit 10 vlan-id 10

set interfaces ge-0/0/3 unit 11 encapsulation vlan-bridge
set interfaces ge-0/0/3 unit 11 vlan-id 11

set bridge-domains VLAN10 interface ge-0/0/3.10
set bridge-domains VLAN11 interface ge-0/0/3.11

commit
```

## Step 3: Configure Trunk 2 on SW2

```junos
configure

set interfaces ge-0/0/3 flexible-vlan-tagging
set interfaces ge-0/0/3 encapsulation flexible-ethernet-services

set interfaces ge-0/0/3 unit 10 encapsulation vlan-bridge
set interfaces ge-0/0/3 unit 10 vlan-id 10

set interfaces ge-0/0/3 unit 11 encapsulation vlan-bridge
set interfaces ge-0/0/3 unit 11 vlan-id 11

set bridge-domains VLAN10 interface ge-0/0/3.10
set bridge-domains VLAN11 interface ge-0/0/3.11

commit
```

## Step 4: Verify RSTP blocked the redundant path

```junos
show spanning-tree interface
```

Expected on SW2 — one trunk port is Alternate/Discarding:

```text
Interface      Port ID    Designated    Port    State       Role
                          bridge ID     Cost
ge-0/0/0.10    128:1      1000.SW1mac   2000  Forwarding  Root
ge-0/0/3.10    128:4      8000.SW2mac   2000  Discarding  Alternate
```

```junos
show bridge domain
```

Expected: both bridge domains now list three interfaces (two trunk subunits + one access port):

```text
Routing instance        Bridge domain            VLAN ID     Interfaces
default-switch          VLAN10                   10
                                                     ge-0/0/0.10
                                                     ge-0/0/1.0
                                                     ge-0/0/3.10
default-switch          VLAN11                   11
                                                     ge-0/0/0.11
                                                     ge-0/0/2.0
                                                     ge-0/0/3.11
```

## Step 5: Confirm traffic is unaffected

From PC1, ping PC3:

```text
ping 192.168.10.2
```

Pings should succeed without interruption — RSTP converged before the loop was ever active.
