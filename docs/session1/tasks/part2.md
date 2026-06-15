# Part 2 — Connect Devices & Boot

## Step 1: Draw the Link

1. In GNS3, click the **Add a link** button (cable icon in the toolbar)
2. Click **R1** — a menu pops up showing its interfaces
3. Select `ge-0/0/0 (Adapter 0)`
4. Click **R2** — select `ge-0/0/0 (Adapter 0)`

A line appears between the two nodes. The link is red (down) because the nodes are not running yet.

## Step 2: Start the Nodes

Click the green **Start all nodes** button (play icon) or right-click each node and choose **Start**.

!!! warning "Boot time: 3-5 minutes"
    vJunos-router loads a full Junos OS. The GNS3 progress indicator will show activity for several minutes. Do not open a console or attempt configuration until the node is fully booted.

## Step 3: Open the Console

Once both nodes show green links:

1. Right-click **R1** > **Console**
2. A terminal window opens — wait for the `root@` prompt

You will see boot messages scroll by, then eventually:

```
Amnesiac (ttyd0)

login:
```

Type `root` and press Enter. No password is required on a fresh boot.

```text
root@%
```

You are now in the Junos **shell** (BSD shell). Type `cli` to enter the Junos CLI:

```text
root@% cli
root>
```

The `>` prompt indicates you are in **operational mode**.

## Step 4: Verify the Node is Ready

```junos
show version
```

Expected output includes the Junos version and platform:

```text
Junos: 23.2R1.15
...
Model: vjunos-router
```

If you see the model and version, the node is fully booted and ready for configuration.

!!! tip "Console shortcut"
    In GNS3 you can also double-click a node to open its console. Middle-click closes it.
