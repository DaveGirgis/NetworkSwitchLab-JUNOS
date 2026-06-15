# Part 1 — Create a GNS3 Project & Add Devices

## Step 1: Create a New Project

1. In GNS3, select **File** > **New blank project**
2. Name it `JNCIS-SP-Session1`
3. Click **OK**

The project opens on a blank canvas.

## Step 2: Add Two vJunos-router Nodes

1. In the **Devices** panel on the left, expand **Routers**
2. Drag **vJunos-router** onto the canvas twice
3. GNS3 names them `vJunos-router-1` and `vJunos-router-2` automatically

!!! tip "Rename for clarity"
    Right-click each node, choose **Change hostname**, and rename them `R1` and `R2`. These names appear on the canvas and on the Junos console prompt after configuration.

## Step 3: Review Node Properties

Right-click `R1` and choose **Configure** to confirm:

- RAM: 4096 MB
- Adapters: 4
- The `.qcow2` disk image is attached

Repeat for `R2`.

## Step 4: Save the Project

Press **Ctrl+S** or use **File** > **Save project**. Save frequently — GNS3 project state can be lost if the VM crashes mid-session.
