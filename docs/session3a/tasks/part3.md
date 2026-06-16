# Part 3 — RSTP Failover Test

RSTP's main advantage over original STP is fast convergence. When the active path fails, the Alternate port transitions to Forwarding in under one second rather than 30–50 seconds.

## Step 1: Establish a baseline ping

From PC1, start a continuous ping to PC3:

```text
ping 192.168.10.2 -t
```

VPCS pings once per second by default. Confirm steady replies before proceeding.

## Step 2: Identify which trunk is active

On SW2, note which trunk port is in Forwarding state (Root role):

```junos
show spanning-tree interface
```

Note the interface name — either `ge-0/0/0.10` or `ge-0/0/3.10` will be the Root port.

## Step 3: Disconnect the active trunk in GNS3

In GNS3, right-click the **active trunk link** (the one carrying traffic) and select **Delete link** or simply **Suspend link** if available.

Watch the ping output from PC1. With RSTP:

- **Expected:** 1–3 ping failures while RSTP reconverges, then replies resume
- **If original STP:** 30–50 seconds of failures (indicates RSTP is not active)

## Step 4: Verify the Alternate port took over

On SW2:

```junos
show spanning-tree interface
```

The port that was previously Alternate/Discarding should now show Root/Forwarding. The dead trunk should show Discarding.

```junos
show bridge domain
```

The bridge domain still lists both trunk interfaces — but traffic now flows through the previously-blocked one.

## Step 5: Restore the link

In GNS3, reconnect the link you removed (SW1 Adapter 2 ↔ SW2 Adapter 2, or Adapter 5 ↔ Adapter 5 depending on which you cut).

After the link comes back:

```junos
show spanning-tree interface
```

RSTP will reassess and return the topology to its original state (SW1 ge-0/0/0 forwarding, SW2 ge-0/0/3 blocked), since the preferred path is restored.

## What you observed

| Event | Original STP (802.1D) | RSTP (802.1w) |
|-------|-----------------------|---------------|
| Failover time | 30–50 seconds | < 1 second |
| Recovery time | 30–50 seconds | < 1 second |
| Mechanism | Timer-based | Proposal/agreement handshake |

RSTP achieves fast convergence by using a **proposal/agreement** handshake between adjacent bridges instead of waiting for timers to expire. When a port is ready to forward, it proposes the change to its neighbour, which agrees and immediately transitions — no waiting.
