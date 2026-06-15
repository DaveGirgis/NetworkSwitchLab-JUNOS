# Part 0 — Install GNS3 & Import vJunos-router

## Step 1: Install GNS3 and GNS3 VM

1. Download the GNS3 all-in-one installer from [gns3.com/software](https://www.gns3.com/software/download)
2. Run the installer — select **GNS3 VM** when prompted to choose a VM type
3. Download the **GNS3 VM** `.ova` file from the same download page (version must match GNS3)
4. Import the `.ova` into VMware Workstation Player:
   - **File** > **Open** > select the `.ova`
   - Accept defaults; the VM will appear in your library
5. In GNS3, go to **Edit** > **Preferences** > **GNS3 VM** and confirm the VM is detected

!!! warning "Match versions exactly"
    The GNS3 application and GNS3 VM version numbers must match (e.g., both 2.2.44). Mismatched versions cause connection errors that are difficult to diagnose.

## Step 2: Configure GNS3 VM Resources

In VMware Workstation, edit the GNS3 VM settings:

- **RAM**: 16 GB (minimum 8 GB — each vJunos node needs 4 GB)
- **vCPUs**: 4 or more
- **Nested virtualization**: Enable "Virtualize Intel VT-x/EPT or AMD-V/RVI" (required for QEMU)

Start the GNS3 VM. Wait for it to show its IP address on the console before launching GNS3.

## Step 3: Import the vJunos-router Image

1. In GNS3, open **Edit** > **Preferences** > **QEMU** > **QEMU VMs**
2. Click **New**
3. Set **Name**: `vJunos-router`
4. Set **Qemu binary**: `/bin/qemu-system-x86_64` (auto-detected in most cases)
5. Set **RAM**: `4096`
6. Click **Next**, then select **New Image**
7. Browse to your `.qcow2` file and import it
8. Leave all other defaults and click **Finish**

## Step 4: Configure the vJunos-router Template

After import, right-click the template and choose **Edit**:

| Setting | Value |
|---------|-------|
| General > Symbol | Router (or any router icon) |
| General > Category | Routers |
| HDD > Disk image | your `.qcow2` path |
| Network > Adapters | 4 |
| Network > Type | virtio-net-pci |
| Advanced > Options | `-nographic` |

!!! note "4 adapters"
    vJunos-router maps its interfaces as `ge-0/0/0` through `ge-0/0/3`. Each GNS3 adapter corresponds to one `ge-0/0/X` interface. You need at least 4 for the full topology used in Sessions 3–8.

Click **OK** to save the template.
