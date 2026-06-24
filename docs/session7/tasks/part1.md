# Part 1 — Enable MPLS & LDP

## Overview

With `family mpls` in place on all provider interfaces, enable the MPLS and LDP protocols on each router. Each router needs:

1. **`protocols mpls interface <intf>`** — registers the interface with the MPLS forwarding engine
2. **`protocols ldp interface <intf>`** — enables LDP hello and label distribution on the interface
3. **`protocols ldp interface lo0.0`** — adds the loopback to LDP so LDP uses the loopback IP as its LSR-ID and transport address

LDP discovers neighbors by sending **hello** messages out enabled interfaces. When two routers see each other's hellos, they establish a TCP session (on port 646) and exchange label bindings for every prefix in their routing tables. This process happens automatically once the interfaces are enabled — no explicit neighbor configuration is required.

## Step 1: Enable MPLS & LDP on PE1

```junos
configure

set protocols mpls interface ge-0/0/0.0
set protocols ldp interface ge-0/0/0.0
set protocols ldp interface lo0.0

commit
```

## Step 2: Enable MPLS & LDP on P1

P1 has two provider-facing interfaces — both need MPLS and LDP.

```junos
configure

set protocols mpls interface ge-0/0/0.0
set protocols mpls interface ge-0/0/1.0
set protocols ldp interface ge-0/0/0.0
set protocols ldp interface ge-0/0/1.0
set protocols ldp interface lo0.0

commit
```

## Step 3: Enable MPLS & LDP on P2

```junos
configure

set protocols mpls interface ge-0/0/0.0
set protocols mpls interface ge-0/0/1.0
set protocols ldp interface ge-0/0/0.0
set protocols ldp interface ge-0/0/1.0
set protocols ldp interface lo0.0

commit
```

## Step 4: Enable MPLS & LDP on PE2

```junos
configure

set protocols mpls interface ge-0/0/0.0
set protocols ldp interface ge-0/0/0.0
set protocols ldp interface lo0.0

commit
```

## Step 5: Verify LDP Neighbors

LDP sessions should form within a few seconds of both sides being configured. Check each router in turn.

```junos
PE1> show ldp neighbor
```

Expected — one LDP neighbor (P1), identified by its loopback address:

```text
Address                             Interface       Label space ID     Hold time
10.1.12.2                         ge-0/0/0.0      10.0.0.2:0              13
```

```junos
P1> show ldp neighbor
```

Expected — two LDP neighbors (PE1 and P2):

```text
Address                             Interface       Label space ID     Hold time
10.1.12.1                         ge-0/0/0.0      10.0.0.1:0              12
10.1.23.2                         ge-0/0/1.0      10.0.0.3:0              11
```

```junos
PE2> show ldp neighbor
```

Expected — one LDP neighbor (P2):

```text
Address                             Interface       Label space ID     Hold time
10.1.34.1                         ge-0/0/1.0      10.0.0.3:0              14
```

The **Label space ID** column shows the neighbor's loopback address with `:0` (platform-wide label space). If the neighbor column is empty, wait 10–15 seconds and retry — LDP convergence takes a moment after IS-IS is already up.

!!! note "LDP transport address"
    Adding `lo0.0` to the LDP interface list tells Junos to use the loopback as the LDP transport address. This means the LDP TCP session runs between loopback addresses (10.0.0.x) rather than link addresses. If a physical link goes down but IS-IS reroutes, the LDP session survives because loopback reachability is maintained.

## Step 6: Quick Sanity Check on LDP Session

Confirm the LDP session between PE1 and P1 is fully up:

```junos
PE1> show ldp session
```

Expected:

```text
Address                          State       Connection  Hold time  Adv. Mode
10.0.0.2                         Operational Open          26        DU
```

**State: Operational** confirms the TCP session is up and labels have been exchanged. **DU** (Downstream Unsolicited) is the default LDP advertisement mode — each router advertises labels for all its routes to all neighbors without waiting to be asked.
