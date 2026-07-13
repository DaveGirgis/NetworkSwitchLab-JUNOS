# Part 0 - Project Import

## Overview

This session does not add new configuration. It starts from the exact state you had at the end of Session 8 - IS-IS across the core, MPLS/LDP transport, iBGP between PE1 and PE2 carrying `inet-vpn unicast` and `l2vpn signaling`, the VPN-A L3VPN, and the VPLS-100 L2VPN from Session 7a - and then breaks it, one layer at a time.

Part 0 has three jobs:

1. Make sure every router is running its known-good end-of-Session-8 configuration.
2. Run a quick health check across every layer so you have a "last known good" baseline to compare against once faults start appearing.
3. Receive Fault 1.

## CLI Setup

Before pasting any configuration, run these two commands at the operational prompt (`>`) on each router:

```junos
set cli screen-length 0
set cli complete-on-space off
```

Re-run these at the start of every console session - they do not persist across reconnects.

!!! tip "Pasting large config blocks"
    Use `load set terminal` in config mode instead of pasting directly at the `#` prompt. Junos buffers all input until you press **Ctrl+D**, then processes every set command at once.

    ```text
    configure
    load set terminal
    set protocols isis interface ge-0/0/1.0
    ^D
    commit
    ```

## Step 1: Restore Your Saved Configuration (If Needed)

If your GNS3 project has been running continuously since Session 8 and every router is still up, skip to Step 2 - there is nothing to restore.

If a node crashed, or you are starting this session in a new GNS3 project, restore each router from the configuration you saved at the end of Session 8:

```junos
configure
load override <path-to-saved-config>
commit
```

!!! warning "Do not skip a broken node"
    If any single router in the topology (CE1, PE1, P1, P2, PE2, CE2) fails to restore correctly, every later fault in this session will produce confusing, compounded symptoms. Confirm each router's config with `show configuration | display set | match "isis|bgp|ldp|routing-instances"` and compare against your Session 8 notes before proceeding.

If you do not have a saved configuration, work through Sessions 5-8 again to rebuild the required state. There is no shortcut - this session assumes a fully working starting point, because the entire point is to recognize what "working" looks like well enough to spot what changed.

## Step 2: Baseline Health Check

Run this sequence on every router that participates in the layer being checked. Save the output somewhere (a text file, a second terminal) - you will compare against it after each fault is injected.

### IS-IS (all four provider routers)

```junos
PE1> show isis adjacency
```

Expected - PE1 has one adjacency (P1):

```text
Interface             System         L State        Hold (secs) SNPA
ge-0/0/0.0            P1             2  Up                   24
```

```junos
P1> show isis adjacency
```

Expected - P1 has two adjacencies (PE1 and P2):

```text
Interface             System         L State        Hold (secs) SNPA
ge-0/0/0.0            PE1            2  Up                   22
ge-0/0/1.0            P2             2  Up                   19
```

```junos
P2> show isis adjacency
```

Expected - P2 has two adjacencies (P1 and PE2):

```text
Interface             System         L State        Hold (secs) SNPA
ge-0/0/0.0            P1             2  Up                   21
ge-0/0/1.0            PE2            2  Up                   26
```

```junos
PE2> show isis adjacency
```

Expected - PE2 has one adjacency (P2):

```text
Interface             System         L State        Hold (secs) SNPA
ge-0/0/0.0            P2             2  Up                   18
```

### LDP (all four provider routers)

```junos
PE1> show ldp session
```

Expected:

```text
Address                          State       Connection  Hold time  Adv. Mode
10.0.0.2                         Operational Open          28        DU
10.0.0.4                         Operational Open          26        DU
```

!!! note "Two LDP sessions on PE1"
    PE1 shows both the link LDP session to P1 (10.0.0.2) and the targeted LDP session to PE2 (10.0.0.4) carried forward from Session 7a's L2 service work. Either one or two sessions in `Operational` state is a healthy baseline in this topology; anything other than `Operational` is not.

### BGP (PE1 and PE2)

```junos
PE1> show bgp summary
```

Expected - iBGP to PE2 established, carrying multiple address families:

```text
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.l2vpn.0
                       1          1          0          0          0          0
bgp.l3vpn.0
                       2          2          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.0.4              65001        455        460       0       0     3:12:47 Establ
  bgp.l2vpn.0: 1/1/1/0
  VPLS-100.l2vpn.0: 1/1/1/0
  bgp.l3vpn.0: 2/2/2/0
  VPN-A.inet.0: 2/2/2/0
```

```junos
PE1> show bgp summary instance VPN-A
```

Expected - CE1 established inside the VRF:

```text
Instance: VPN-A
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
VPN-A.inet.0
                       2          2          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.16.1.2            65100        212        208       0       0     3:10:02 1/1/1/0              0/0/0/0
```

### VPN-A VRF Table (PE1)

```junos
PE1> show route table VPN-A.inet.0
```

Expected - four routes:

```text
VPN-A.inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.11/32       *[BGP/170] 03:10:02, localpref 100
                      AS path: 65100 I, validation-state: unverified
                    > to 172.16.1.2 via ge-0/0/1.0
10.0.0.12/32       *[BGP/170] 03:09:40, localpref 100, from 10.0.0.4
                      AS path: 65100 I, validation-state: unverified
                    > to 10.1.12.2 via ge-0/0/0.0, Push <vpn-label>, Push 299808
172.16.1.0/30      *[Direct/0] 03:20:15
                    > via ge-0/0/1.0
172.16.2.0/30      *[BGP/170] 03:09:40, localpref 100, from 10.0.0.4
                    > to 10.1.12.2 via ge-0/0/0.0, Push <vpn-label>, Push 299808
```

### VPLS-100 (PE1)

```junos
PE1> show vpls connections
```

Expected:

```text
Instance: VPLS-100
  Local site: CE1-SITE (1)
    connection-site           Type  St     Time last up          # Up trans
    2                         rmt   Up     Jul 10 09:15:00               1
      Remote PE: 10.0.0.4, Negotiated control-word: No
      Incoming label: 299856, Outgoing label: 299872
      Local interface: ge-0/0/2.0, Status: Up, Encapsulation: ETHERNET
```

### End-to-End Reachability

```junos
CE1> ping 10.0.0.12 source 10.0.0.11 count 3
```

Expected - 3/3 replies, 0% packet loss (VPN-A).

```junos
CE1> ping 192.168.1.2 count 3
```

Expected - 3/3 replies, 0% packet loss (VPLS-100).

!!! warning "Fix the baseline before continuing"
    If any command above does not match its expected output, this is not one of the three graded faults - it is leftover drift from an earlier session (a crashed node, an incomplete commit, a GNS3 restart that dropped a session). Fix it now using the troubleshooting references from Sessions 7, 7a, and 8. Every fault in Parts 1-3 assumes you are starting from a fully healthy network.

## Step 3: Fault 1 Is Injected

Your instructor (or the provided fault-injection script) now makes exactly one change to the IS-IS configuration on one router in the core. You will not be told what changed or which router.

What you will be told:

> "The provider core has an IGP problem. Start troubleshooting from Part 1."

Do not read ahead to Part 1's root-cause reveal before running the diagnostic steps yourself - the value of this session is in the process, not the answer.

Proceed to [Part 1 - Fault 1 (IGP)](part1.md).
