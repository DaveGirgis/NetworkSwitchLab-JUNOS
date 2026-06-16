# Part 3 — IRB for Inter-VLAN Routing

An **IRB (Integrated Routing and Bridging)** interface is a logical Layer 3 interface attached to a bridge domain. It gives the vMX a routable IP address within a VLAN — equivalent to a Cisco SVI (`interface Vlan10`). With IRB interfaces on both VLANs, SW1 acts as the default gateway and routes traffic between VLAN 10 and VLAN 11.

## Step 1: Add IRB interfaces to SW1

```junos
configure

set interfaces irb unit 10 family inet address 192.168.10.254/24
set interfaces irb unit 11 family inet address 192.168.11.254/24

set bridge-domains VLAN10 routing-interface irb.10
set bridge-domains VLAN11 routing-interface irb.11

commit
```

## Step 2: Verify IRB interfaces are up

```junos
show interfaces irb terse
```

Expected:

```text
Interface               Admin Link Proto    Local                 Remote
irb.10                  up    up   inet     192.168.10.254/24
irb.11                  up    up   inet     192.168.11.254/24
```

```junos
show bridge domain
```

Expected: both bridge domains now show `IRB intfs: 1`:

```text
Routing instance        Bridge domain            Intfs  IRB intfs  MAC ageing
default-switch          VLAN10                   2      1          300
default-switch          VLAN11                   2      1          300
```

## Step 3: Test gateway reachability from each PC

From **PC1**:
```text
ping 192.168.10.254
```

From **PC2**:
```text
ping 192.168.11.254
```

From **PC3**:
```text
ping 192.168.10.254
```

From **PC4**:
```text
ping 192.168.11.254
```

All four should receive replies from SW1's IRB interfaces.

## Step 4: Test inter-VLAN routing

From PC1 (VLAN 10), ping PC2 (VLAN 11):

```text
ping 192.168.11.1
```

From PC3 (VLAN 10), ping PC4 (VLAN 11):

```text
ping 192.168.11.2
```

Expected: replies in both cases. SW1 routes the packets between bridge domains through its IRB interfaces.

## Step 5: Check the routing table on SW1

```junos
show route
```

Expected:

```text
inet.0: 2 destinations, 2 routes
  192.168.10.0/24     *[Direct/0] ...
  192.168.11.0/24     *[Direct/0] ...
```

## How it works

```
PC1 (192.168.10.1) → SW1 ge-0/0/1.0 (bridge, VLAN10)
                   → SW1 irb.10 (L3 gateway 192.168.10.254)
                   → SW1 irb.11 (L3 gateway 192.168.11.254)
                   → SW1 ge-0/0/2.0 (bridge, VLAN11)
                   → PC2 (192.168.11.1)

PC3 (192.168.10.2) → SW2 ge-0/0/1.0 (bridge, VLAN10)
                   → SW2 ge-0/0/0.10 (trunk, VLAN10)
                   → SW1 ge-0/0/0.10 (trunk, VLAN10)
                   → SW1 irb.10 → irb.11
                   → SW1 ge-0/0/0.11 (trunk, VLAN11)
                   → SW2 ge-0/0/0.11 (trunk, VLAN11)
                   → SW2 ge-0/0/2.0 (bridge, VLAN11)
                   → PC4 (192.168.11.2)
```

All inter-VLAN routing passes through SW1's IRB interfaces, regardless of which switch the source and destination PCs are connected to.
