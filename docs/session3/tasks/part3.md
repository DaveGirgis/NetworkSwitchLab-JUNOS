# Part 3 — IRB for Inter-VLAN Routing

An **IRB (Integrated Routing and Bridging)** interface is a logical Layer 3 interface attached to a bridge domain. It gives the vMX a routable IP address within a VLAN — equivalent to a Cisco SVI (`interface Vlan10`). With IRB interfaces on both VLANs, SW1 can route between them.

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
default-switch          VLAN11                   1      1          300
```

## Step 3: Test from PC1 → gateway

From the VPCS console on PC1:

```text
ping 192.168.10.254
```

Expected: replies from SW1's IRB.10.

## Step 4: Test inter-VLAN routing

From PC1, ping PC2:

```text
ping 192.168.11.1
```

Expected: replies from PC2. SW1 routes the packet from VLAN 10 to VLAN 11 through its IRB interfaces.

If the ping fails, check that PC2 has its gateway set correctly:

```text
show ip
```

PC2 should show gateway `192.168.11.254`. If not, re-apply:

```text
ip 192.168.11.1 192.168.11.254 24
```

## Step 5: Check the routing table

On SW1:

```junos
show route
```

Expected: two directly connected routes for the IRB subnets:

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
                   → SW1 ge-0/0/0.11 (bridge, VLAN11, trunk)
                   → SW2 ge-0/0/0.11 (bridge, VLAN11, trunk)
                   → SW2 ge-0/0/1.0 (bridge, VLAN11)
                   → PC2 (192.168.11.1)
```

The packet crosses from the VLAN 10 bridge domain to the VLAN 11 bridge domain via the IRB interfaces on SW1 — this is inter-VLAN routing.
