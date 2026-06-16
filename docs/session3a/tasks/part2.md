# Part 2 — Port Roles and States

With RSTP converged, each port is assigned a **role** and a **state**. Understanding these is essential for troubleshooting STP-related connectivity issues.

## RSTP Port Roles

| Role | Description |
|------|-------------|
| **Root** | The port on a non-root bridge with the best path cost to the root bridge. One per switch (except the root bridge itself). |
| **Designated** | The port on a segment that provides the best path toward the root. The root bridge's ports are always Designated. |
| **Alternate** | A blocked port that provides a redundant path to the root. Transitions to Root if the current Root port fails. |
| **Edge** | A port connected directly to an end host (no switches beyond it). Transitions immediately to Forwarding — not part of STP calculations. |

## RSTP Port States

| State | Traffic forwarded? | Description |
|-------|-------------------|-------------|
| **Forwarding** | Yes | Normal operation |
| **Discarding** | No | Blocking — redundant path |
| **Learning** | No (data), Yes (MAC) | Transitional — building MAC table |

## Step 1: View port roles on both switches

```junos
show spanning-tree interface
```

Expected on SW1 (root bridge — all ports are Designated):

```text
Interface      Port ID    Designated    Port    State     Role
                          bridge ID     Cost
ge-0/0/0.10    128:1      1000.SW1mac   2000  Forwarding  Designated
ge-0/0/1.0     128:2      1000.SW1mac      0  Forwarding  Edge
ge-0/0/2.0     128:3      1000.SW1mac      0  Forwarding  Edge
ge-0/0/3.10    128:4      1000.SW1mac   2000  Forwarding  Designated
```

Expected on SW2 (non-root — one trunk port will be Alternate/Discarding):

```text
Interface      Port ID    Designated    Port    State       Role
                          bridge ID     Cost
ge-0/0/0.10    128:1      1000.SW1mac   2000  Forwarding  Root
ge-0/0/1.0     128:2      8000.SW2mac      0  Forwarding  Edge
ge-0/0/2.0     128:3      8000.SW2mac      0  Forwarding  Edge
ge-0/0/3.10    128:4      8000.SW2mac   2000  Discarding  Alternate
```

!!! note "Which trunk gets blocked?"
    The Alternate port is the one with the higher port cost or, if equal, the higher port ID. Since both trunks have the same cost, the higher-numbered port (ge-0/0/3) on SW2 is typically blocked. The actual output on your device may differ — what matters is that exactly one path between the switches is blocked per VLAN.

## Step 2: Confirm traffic flows on the forwarding trunk only

On SW1, check which trunk is actively forwarding:

```junos
show interfaces ge-0/0/0 terse
show interfaces ge-0/0/3 terse
```

The forwarding trunk subunits show `up up Bridge`. The blocked trunk subunit on SW2 shows `up up Bridge` too (link is up) but RSTP is discarding frames on it — the link state and STP state are independent.

## Step 3: Observe edge port behaviour

Access ports (PC-facing) are Edge ports and are never involved in STP topology calculations:

```junos
show spanning-tree interface
```

Look for `Edge` in the Role column next to ge-0/0/1.0 and ge-0/0/2.0. Edge ports go directly to Forwarding when the link comes up, which is why VPCS console access is never delayed by STP.
