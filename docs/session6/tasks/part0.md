# Part 0 — Expand Topology & Base Config

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

## Step 1: Add CE Nodes to GNS3

In the `JNCIS-SP-Core` GNS3 project:

1. Drag two new **vMX-14.1** nodes onto the canvas and name them `CE1` and `CE2`
2. Draw the following links:

| Link | Node A | Adapter | Node B | Adapter |
|------|--------|---------|--------|---------|
| CE1 — PE1 | CE1 | 2 (ge-0/0/0) | PE1 | 3 (ge-0/0/1) |
| CE2 — PE2 | CE2 | 2 (ge-0/0/0) | PE2 | 3 (ge-0/0/1) |

3. Start CE1 and CE2. Wait 3–5 minutes for both to fully boot before opening consoles.

!!! warning "RAM check"
    Six vMX-14.1 nodes require 12 GB of RAM in the GNS3 VM. Confirm the GNS3 VM has at least 12 GB allocated before starting CE1 and CE2.

## Step 2: Base Config on CE1 and CE2

### CE1

```junos
configure

set system host-name CE1
set system root-authentication plain-text-password
New password: lab123
Retype new password: lab123

commit
```

### CE2

```junos
configure

set system host-name CE2
set system root-authentication plain-text-password
New password: lab123
Retype new password: lab123

commit
```

## Step 3: Configure Interfaces on CE1 and CE2

### CE1

```junos
configure

set interfaces ge-0/0/0 unit 0 family inet address 172.16.1.2/30
set interfaces lo0 unit 0 family inet address 10.0.0.11/32
set routing-options router-id 10.0.0.11

commit
```

### CE2

```junos
configure

set interfaces ge-0/0/0 unit 0 family inet address 172.16.2.2/30
set interfaces lo0 unit 0 family inet address 10.0.0.12/32
set routing-options router-id 10.0.0.12

commit
```

## Step 4: Configure PE-CE Interfaces on PE1 and PE2

PE1 and PE2 already have ge-0/0/0 configured (provider core). Add the new CE-facing interface on each.

### PE1

```junos
configure

set interfaces ge-0/0/1 unit 0 family inet address 172.16.1.1/30

commit
```

### PE2

```junos
configure

set interfaces ge-0/0/1 unit 0 family inet address 172.16.2.1/30

commit
```

## Step 5: Verify Direct Connectivity

Confirm each PE can reach its directly connected CE:

```junos
PE1> ping 172.16.1.2 count 3
```

Expected: 3/3 replies from CE1.

```junos
PE2> ping 172.16.2.2 count 3
```

Expected: 3/3 replies from CE2.

```junos
CE1> ping 172.16.1.1 count 3
```

Expected: 3/3 replies from PE1.

!!! warning "Verify links before configuring BGP"
    BGP sessions will not establish if the underlying IP connectivity is broken. Fix any interface or link issues here before moving to Part 1.
