# Part 0 — Add Second Trunk

This part adds a second trunk link between SW1 and SW2, creating a Layer 2 loop. **Do not enable RSTP yet** — the loop will be active briefly while you observe what happens without STP protection. Keep this step short.

!!! danger "Broadcast storm risk"
    As soon as the second trunk is connected and configured without STP, any broadcast (including ARP from a ping) will loop indefinitely. Your VPCS nodes may become unreachable and the vMX CPU may spike. If this happens, stop and disconnect the second trunk link from GNS3 immediately, then proceed to Part 1 before reconnecting.

## Step 1: Draw the second trunk in GNS3

1. Click **Add a link**
2. Click **SW1** → select **Adapter 5** (ge-0/0/3)
3. Click **SW2** → select **Adapter 5** (ge-0/0/3)

The link will appear in GNS3 but traffic will not flow until the trunk is configured on both switches.

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

## Step 4: Observe the loop (briefly)

With both trunks active and no STP, send one ping from PC1:

```text
ping 192.168.10.2
```

You may see the ping hang or fail as ARP traffic loops. Watch the vMX console for CPU warnings. This demonstrates why STP is necessary — proceed immediately to Part 1.

```junos
show bridge domain
```

Expected: both bridge domains now show three interfaces each (ge-0/0/0.1x, ge-0/0/1.0 or ge-0/0/2.0, and ge-0/0/3.1x).
