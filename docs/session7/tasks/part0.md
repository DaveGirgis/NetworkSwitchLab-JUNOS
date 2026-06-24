# Part 0 — Verify Session 6 State & Add family mpls

## CLI Setup

Before pasting any configuration, run these two commands at the operational prompt (`>`) on each router:

```junos
set cli screen-length 0
set cli complete-on-space off
```

Re-run these at the start of every console session — they do not persist across reconnects.

!!! tip "Pasting large config blocks"
    Use `load set terminal` in config mode instead of pasting directly at the `#` prompt. Junos buffers all input until you press **Ctrl+D**, then processes every set command at once — eliminating paste-rate issues with console terminals.

    ```text
    configure
    load set terminal
    set protocols mpls interface ge-0/0/0.0
    ...
    ^D
    commit
    ```

## Step 1: Verify Session 6 State

Before adding MPLS, confirm IS-IS and BGP are healthy. If either is broken, MPLS will not fix it.

On **PE1**:

```junos
show isis adjacency
```

Expected — two adjacencies (P1 and no others on PE1; only ge-0/0/0 runs IS-IS):

```text
Interface             System         L State        Hold (secs) SNPA
ge-0/0/0.0           P1             2  Up                   22
```

```junos
show bgp summary
```

Expected — two BGP sessions (eBGP to CE1, iBGP to PE2), both `Establ`:

```text
Groups: 2 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0
                       2          2          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.0.4              65001         42         41       0       0       20:11 1/1/1/0              0/0/0/0
172.16.1.2            65100         89        105       0       0       44:22 1/1/1/0              0/0/0/0
```

!!! warning "Fix IS-IS or BGP issues before continuing"
    MPLS depends on IS-IS routes to compute LSPs. BGP depends on MPLS to resolve next-hops. If IS-IS adjacencies are missing or BGP sessions are down, return to Sessions 5 or 6 to restore them.

## Step 2: Add `family mpls` to Provider Interfaces

MPLS-labeled packets have a different EtherType (0x8847) than plain IP packets. Each interface that will carry MPLS-labeled traffic must have `family mpls` enabled, or the router will drop incoming labeled frames.

Add `family mpls` to the provider-facing interfaces on all four provider routers. CE-facing interfaces (ge-0/0/1 on PE1 and PE2) are **not** included — CE routers are outside the MPLS domain.

### PE1

```junos
configure

set interfaces ge-0/0/0 unit 0 family mpls

commit
```

### P1

```junos
configure

set interfaces ge-0/0/0 unit 0 family mpls
set interfaces ge-0/0/1 unit 0 family mpls

commit
```

### P2

```junos
configure

set interfaces ge-0/0/0 unit 0 family mpls
set interfaces ge-0/0/1 unit 0 family mpls

commit
```

### PE2

```junos
configure

set interfaces ge-0/0/0 unit 0 family mpls

commit
```

## Step 3: Confirm Interface Configuration

On PE1, confirm ge-0/0/0 now shows both `inet` and `mpls` families:

```junos
show interfaces ge-0/0/0 detail | match Proto
```

Expected:

```text
    Protocol inet, MTU: 1500, Generation: 154, Route table: 0
    Protocol iso, MTU: 1497, Generation: 155, Route table: 0
    Protocol mpls, MTU: 1488, Maximum labels: 3, Generation: 159,
    Protocol multiservice, MTU: Unlimited, Generation: 156, Route table: 0
```

The `mpls` line confirms the interface is ready to receive labeled frames. The MTU of 1488 accounts for the 4-byte MPLS label header reducing the usable payload space below the 1492 Ethernet MTU.

!!! note "No traffic disruption"
    Adding `family mpls` does not change how IP or IS-IS traffic is forwarded. IS-IS adjacencies and BGP sessions remain up throughout this step.
