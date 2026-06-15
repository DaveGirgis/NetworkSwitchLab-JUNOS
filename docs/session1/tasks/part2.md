# Part 2 — Connect Devices & Boot

## Step 1: Draw the Link

1. In GNS3, click the **Add a link** button (cable icon in the toolbar)
2. Click **R1** — a menu pops up showing its adapters
3. Select **Adapter 2** (this maps to `ge-0/0/0` inside Junos)
4. Click **R2** — select **Adapter 2**

A line appears between the two nodes.

!!! warning "Always use Adapter 2 for ge-0/0/0"
    vMX reserves Adapter 0 (`em0`) and Adapter 1 (`em1`) for internal management. Data plane interfaces start at Adapter 2. Connecting Adapter 0 or 1 between nodes will not produce `ge-0/0/0` connectivity.

    | GNS3 Adapter | Junos Interface |
    |---|---|
    | Adapter 0 | `em0` — management only |
    | Adapter 1 | `em1` — internal (172.16.0.1/16) |
    | Adapter 2 | `ge-0/0/0` |
    | Adapter 3 | `ge-0/0/1` |
    | Adapter 4 | `ge-0/0/2` |
    | Adapter 5 | `ge-0/0/3` |

## Step 2: Start the Nodes

Click the green **Start all nodes** button or right-click each node and choose **Start**.

!!! warning "Boot time: 3–5 minutes"
    vMX loads a full Junos OS. Do not open a console or attempt configuration until the node is fully booted.

## Step 3: Open the Console and Log In

Once both nodes show green links:

1. Right-click **R1** > **Console**
2. Wait for the login prompt, then type `root` and press Enter — no password required on a fresh image
3. Type `cli` to enter the Junos CLI:

```text
--- JUNOS 14.1R4.8 built 2015-01-28 03:38:12 UTC
root@% cli
root>
```

The `>` prompt indicates you are in **operational mode**.

## Step 4: Verify the Node is Ready

```junos
show version
```

Expected output:

```text
Junos: 14.1R4.8
...
Model: vmx
```

```junos
show interfaces ge-0/0/0 terse
```

Expected:

```text
Interface               Admin Link Proto    Local                 Remote
ge-0/0/0                up    up
```

If `ge-0/0/0` shows `up up` the node is ready for configuration.

!!! note "ge-0/0/x not visible in `show interfaces terse`"
    On vMX 14.1, `ge-0/0/x` interfaces do not appear in the general `show interfaces terse` output until they have an IP address configured. Query them directly with `show interfaces ge-0/0/0 terse` to confirm they exist.
