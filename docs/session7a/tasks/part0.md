# Part 0 — Baseline Verification & New GNS3 Links

## CLI Setup

Before pasting any configuration, run these two commands at the operational prompt (`>`) on each router:

```junos
set cli screen-length 0
set cli complete-on-space off
```

Re-run these at the start of every console session — they do not persist across reconnects.

!!! tip "Pasting large config blocks"
    Use `load set terminal` in config mode instead of pasting directly at the `#` prompt. Junos buffers all input until you press **Ctrl+D**, then processes every set command at once.

    ```text
    configure
    load set terminal
    set interfaces ge-0/0/1 unit 0 family inet address 192.168.1.1/24
    ^D
    commit
    ```

## Step 1: Verify Session 7 Baseline

L2circuit and VPLS both run over the LDP transport established in Session 7. Confirm LDP is fully operational before adding any L2 service configuration.

On **PE1**:

```junos
show ldp session
```

Expected — one session to P1, state `Operational`:

```text
Address                          State       Connection  Hold time  I/F
10.0.0.2                         Operational Open          27        3
```

```junos
show route table inet.3
```

Expected — three LDP-resolved routes with Push labels for P2 and PE2:

```text
inet.3: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32        *[LDP/9] ...
                    > to 10.1.12.2 via ge-0/0/0.0
10.0.0.3/32        *[LDP/9] ...
                    > to 10.1.12.2 via ge-0/0/0.0, Push 299792
10.0.0.4/32        *[LDP/9] ...
                    > to 10.1.12.2 via ge-0/0/0.0, Push 299808
```

!!! warning "Fix LDP before continuing"
    If LDP sessions are missing or inet.3 is empty, return to Session 7 to restore MPLS. L2circuit relies on LDP to signal pseudowire labels, and VPLS relies on LDP for the transport path.

## Step 2: Add New GNS3 Links

Stop all nodes in GNS3. Connect the following new links:

- **PE1 Adapter 4** (ge-0/0/2) to **CE1 Adapter 3** (ge-0/0/1)
- **PE2 Adapter 4** (ge-0/0/2) to **CE2 Adapter 3** (ge-0/0/1)

All six adapters on each vMX node are provisioned at boot — the interfaces already exist in Junos. After adding the GNS3 links, restart all nodes.

Wait for the full boot sequence (~3–5 minutes) before proceeding.

## Step 3: Configure CE Test Interfaces

Add IP addresses to the new CE-facing interfaces. These are the addresses CE1 and CE2 will use to test L2 connectivity through the pseudowire and VPLS.

### CE1

```junos
configure

set interfaces ge-0/0/1 unit 0 family inet address 192.168.1.1/24

commit
```

### CE2

```junos
configure

set interfaces ge-0/0/1 unit 0 family inet address 192.168.1.2/24

commit
```

## Step 4: Verify CE Interfaces Are Up

On **CE1**:

```junos
show interfaces ge-0/0/1 terse
```

Expected:

```text
Interface               Admin Link Proto    Local                 Remote
ge-0/0/1                up    up
ge-0/0/1.0              up    up   inet     192.168.1.1/24
                                   multiservice
```

On **CE2**:

```junos
show interfaces ge-0/0/1 terse
```

Expected:

```text
Interface               Admin Link Proto    Local                 Remote
ge-0/0/1                up    up
ge-0/0/1.0              up    up   inet     192.168.1.2/24
                                   multiservice
```

## Step 5: Confirm CE1 Cannot Reach CE2 Yet

At this point the physical links are up and the CE interfaces have addresses, but no L2 service is configured on the PE side. CE1 cannot reach CE2.

```junos
CE1> ping 192.168.1.2 count 3
```

Expected — 100% packet loss. The packets reach PE1 ge-0/0/2 and stop; PE1 has no L2 forwarding configured for that interface yet.

This is the expected state before Part 1. Proceed to configure L2circuit.
