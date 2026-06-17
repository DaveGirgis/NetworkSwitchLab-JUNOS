# Part 2 — Port Roles and States

With RSTP converged, each port is assigned a **role** and a **state**. Understanding these is essential for troubleshooting STP-related connectivity issues.

## RSTP Port Roles

| Role (CLI) | Description |
|------------|-------------|
| **DESG** — Designated | The port on a segment providing the best path toward the root. All ports on the root bridge are Designated. |
| **ROOT** — Root | The port on a non-root bridge with the best path cost to the root bridge. One per switch. |
| **ALT** — Alternate | A blocked port providing a redundant path to the root. Transitions to Root if the active Root port fails. |

## RSTP Port States

| State (CLI) | Traffic forwarded? | Description |
|-------------|-------------------|-------------|
| **FWD** — Forwarding | Yes | Normal operation |
| **BLK** — Blocking | No | Redundant path suppressed by STP |
| **LRN** — Learning | No (data), Yes (MAC) | Transitional — building MAC table |

## Step 1: View port roles on SW1

```junos
show spanning-tree interface
```

Expected on SW1 (root bridge — all ports Designated/Forwarding):

```text
Spanning tree interface parameters for instance 0

Interface    Port ID    Designated      Designated         Port    State  Role
                         port ID        bridge ID          Cost
ge-0/0/0         128:1        128:1   4096.0005867107d0     20000  FWD    DESG
ge-0/0/1         128:2        128:2   4096.0005867107d0     20000  FWD    DESG
ge-0/0/2         128:3        128:3   4096.0005867107d0     20000  FWD    DESG
ge-0/0/3         128:4        128:4   4096.0005867107d0     20000  FWD    DESG
```

The Designated bridge ID (`4096.MAC`) confirms SW1 is the root — the priority prefix `4096` matches the configured `bridge-priority 4096`. All of SW1's ports are `DESG` because the root bridge always wins on every segment.

## Step 2: View port roles on SW2

```junos
show spanning-tree interface
```

Expected on SW2 — one trunk port will show `ALT` and `BLK`:

```text
Spanning tree interface parameters for instance 0

Interface    Port ID    Designated      Designated         Port    State  Role
                         port ID        bridge ID          Cost
ge-0/0/0         128:1        128:1   4096.0005867107d0     20000  FWD    ROOT
ge-0/0/1         128:2        128:2   8192.SW2mac           20000  FWD    DESG
ge-0/0/2         128:3        128:3   8192.SW2mac           20000  FWD    DESG
ge-0/0/3         128:4        128:4   8192.SW2mac           20000  BLK    ALT
```

`ge-0/0/0` is the **Root port** — SW2's best path to the root bridge.
`ge-0/0/3` is the **Alternate port** — the redundant trunk is blocked.

!!! note "Which trunk gets blocked?"
    Both trunks have the same port cost (20000), so the tie is broken by port ID. The higher port number (ge-0/0/3) on SW2 loses and becomes the Alternate. The actual MAC addresses in your output will differ but the Root/ALT roles should match.

## Step 3: Confirm traffic is unaffected

From PC1, ping PC3:

```text
ping 192.168.10.2
```

Traffic flows across the Root port (ge-0/0/0). The Alternate port (ge-0/0/3) is silent.
