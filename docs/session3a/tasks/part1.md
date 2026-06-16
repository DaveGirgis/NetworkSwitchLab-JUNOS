# Part 1 — Enable RSTP and Root Election

RSTP (Rapid Spanning Tree Protocol, IEEE 802.1w) is enabled per bridge domain in Junos. The **root bridge** is elected based on bridge priority — the switch with the lowest priority wins. We set SW1 to priority 4096 so it always becomes root.

!!! warning "Command verification needed"
    STP configuration syntax for vMX 14.1 bridge domains is flagged as uncertain. Apply the commands below and report any commit errors so the guide can be corrected.

## Step 1: Enable RSTP on SW1 and set as root bridge

```junos
configure

set bridge-domains VLAN10 protocols rstp
set bridge-domains VLAN11 protocols rstp

set protocols rstp bridge-priority 4096

commit
```

## Step 2: Enable RSTP on SW2 (default priority)

```junos
configure

set bridge-domains VLAN10 protocols rstp
set bridge-domains VLAN11 protocols rstp

commit
```

SW2 uses the default bridge priority of 32768 — higher than SW1's 4096, so SW1 wins the root election.

## Step 3: Verify the root bridge election

On SW1:

```junos
show spanning-tree bridge
```

Expected on SW1 (root bridge):

```text
STP bridge parameters:
  Context: default-switch
  Enabled protocol: RSTP
    Root ID       : 1000.MAC-of-SW1
    Hello time    : 2 seconds
    Maximum age   : 20 seconds
    Forward delay : 15 seconds
    Message age   : 0
  This bridge is the root
```

On SW2:

```junos
show spanning-tree bridge
```

Expected on SW2:

```text
  Root ID       : 1000.MAC-of-SW1
  This bridge is not the root
    Root port     : ge-0/0/0.10 (or ge-0/0/3.10)
```

!!! note "Bridge priority in the Root ID"
    The Root ID shown is a combination of priority and MAC address. With priority 4096 (hex 0x1000), SW1's Root ID will always be lower than SW2's 0x8000 regardless of MAC address — guaranteeing SW1 wins the election.

## Step 4: Verify traffic has recovered

From PC1, ping PC3:

```text
ping 192.168.10.2
```

RSTP should have converged and the ping should succeed. If the ping still fails, wait 10 seconds and try again — RSTP convergence on first boot can take a few seconds.
