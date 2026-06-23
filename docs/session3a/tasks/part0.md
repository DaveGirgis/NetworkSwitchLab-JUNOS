# Part 0 — Enable RSTP

RSTP is enabled before adding the second trunk so the loop is never active without protection. This is the correct production approach — always enable spanning tree before introducing a redundant Layer 2 path.

## CLI Setup

Before pasting any configuration, run these two commands at the operational prompt (`>`) on each switch:

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
    set protocols rstp ...
    ^D
    commit
    ```

## Step 1: Enable RSTP on SW1 (root bridge)

```junos
configure

set protocols rstp bridge-priority 4096
set protocols rstp interface ge-0/0/0
set protocols rstp interface ge-0/0/1 edge
set protocols rstp interface ge-0/0/2 edge

commit
```

## Step 2: Enable RSTP on SW2

```junos
configure

set protocols rstp bridge-priority 8k
set protocols rstp interface ge-0/0/0
set protocols rstp interface ge-0/0/1 edge
set protocols rstp interface ge-0/0/2 edge

commit
```

`8k` is Junos shorthand for 8192. SW2's priority of 8192 is higher than SW1's 4096, so SW1 will win the root election once both trunks are active.

## Step 3: Verify RSTP is active

On both switches:

```junos
show spanning-tree bridge
```

Expected on SW1:

```text
STP bridge parameters:
  Context: default-switch
  Enabled protocol: RSTP
  This bridge is the root
```

Expected on SW2:

```text
  Enabled protocol: RSTP
  This bridge is not the root
```

Confirm RSTP is running on both switches before proceeding to Part 1.
