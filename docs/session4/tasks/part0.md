# Part 0 — Build Topology & Base Config

## Step 1: Create the GNS3 Project

1. In GNS3: **File** > **New blank project** > name it `JNCIS-SP-Core`
2. Drag four **vMX-14.1** nodes onto the canvas
3. Rename them `PE1`, `P1`, `P2`, `PE2` (right-click > Change hostname in GNS3)

## Step 2: Draw the Links

| Link | Node A | Adapter | Node B | Adapter |
|------|--------|---------|--------|---------|
| PE1 — P1 | PE1 | 2 (ge-0/0/0) | P1 | 2 (ge-0/0/0) |
| P1 — P2 | P1 | 3 (ge-0/0/1) | P2 | 2 (ge-0/0/0) |
| P2 — PE2 | P2 | 3 (ge-0/0/1) | PE2 | 2 (ge-0/0/0) |

## Step 3: Start All Nodes

Click **Start all nodes**. Wait 3–5 minutes for all four nodes to fully boot before opening consoles.

!!! warning "RAM requirement"
    Four vMX-14.1 nodes require **8 GB** of RAM in the GNS3 VM (2048 MB each). If the GNS3 VM has less than 8 GB, start the nodes two at a time and configure them in pairs.

## Step 4: CLI Setup

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

## Step 5: Apply Base Config

Open a console on each node and apply the base configuration.

### PE1

```junos
configure

set system host-name PE1
set system root-authentication plain-text-password
New password: lab123
Retype new password: lab123

commit
```

### P1

```junos
configure

set system host-name P1
set system root-authentication plain-text-password
New password: lab123
Retype new password: lab123

commit
```

### P2

```junos
configure

set system host-name P2
set system root-authentication plain-text-password
New password: lab123
Retype new password: lab123

commit
```

### PE2

```junos
configure

set system host-name PE2
set system root-authentication plain-text-password
New password: lab123
Retype new password: lab123

commit
```

## Verify Base Config

On each router, confirm the prompt shows the correct hostname:

```text
PE1>
P1>
P2>
PE2>
```
