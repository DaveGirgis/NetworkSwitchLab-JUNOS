# Part 0 — Remove OSPF & Configure IS-IS

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
    set system host-name PE1
    set interfaces ge-0/0/0 ...
    ^D
    commit
    ```

## Step 1: Remove OSPF

On **all four routers** (PE1, P1, P2, PE2):

```junos
configure

delete protocols ospf

commit
```

Verify OSPF is gone:

```junos
show ospf neighbor
```

Expected: command returns empty or `OSPF instance is not running`.

!!! warning "Routes will be missing briefly"
    After deleting OSPF, the routing table loses all learned routes until IS-IS is configured and converges. This is expected — configure IS-IS on all routers before testing connectivity.

## Step 2: Configure IS-IS

IS-IS requires two things that OSPF does not:

1. **`family iso` on each interface** — IS-IS runs directly over Layer 2 and needs the ISO address family enabled on every interface it uses, including the loopback.
2. **NET address on the loopback** — the NET is IS-IS's system identifier (equivalent to OSPF router-id).

Configure each router in sequence.

### PE1

```junos
configure

set interfaces lo0 unit 0 family iso address 49.0001.0100.0000.0001.00
set interfaces ge-0/0/0 unit 0 family iso

set protocols isis level 1 disable
set protocols isis interface ge-0/0/0.0 point-to-point
set protocols isis interface lo0.0 passive

commit
```

### P1

```junos
configure

set interfaces lo0 unit 0 family iso address 49.0001.0100.0000.0002.00
set interfaces ge-0/0/0 unit 0 family iso
set interfaces ge-0/0/1 unit 0 family iso

set protocols isis level 1 disable
set protocols isis interface ge-0/0/0.0 point-to-point
set protocols isis interface ge-0/0/1.0 point-to-point
set protocols isis interface lo0.0 passive

commit
```

### P2

```junos
configure

set interfaces lo0 unit 0 family iso address 49.0001.0100.0000.0003.00
set interfaces ge-0/0/0 unit 0 family iso
set interfaces ge-0/0/1 unit 0 family iso

set protocols isis level 1 disable
set protocols isis interface ge-0/0/0.0 point-to-point
set protocols isis interface ge-0/0/1.0 point-to-point
set protocols isis interface lo0.0 passive

commit
```

### PE2

```junos
configure

set interfaces lo0 unit 0 family iso address 49.0001.0100.0000.0004.00
set interfaces ge-0/0/0 unit 0 family iso

set protocols isis level 1 disable
set protocols isis interface ge-0/0/0.0 point-to-point
set protocols isis interface lo0.0 passive

commit
```

## Step 3: Quick Sanity Check

On PE1, confirm IS-IS adjacency is forming:

```junos
show isis adjacency
```

Expected — one neighbor (P1) in state `Up`:

```text
Interface             System         L State        Hold (secs) SNPA
ge-0/0/0.0           P1             2  Up                   23
```

If the adjacency shows `Up`, proceed to Part 1 for full verification. If it is blank or stuck in another state, see the Troubleshooting guide.
