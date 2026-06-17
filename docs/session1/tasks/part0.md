# Part 0 — Install GNS3 & Import vMX

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

In GNS3, go to **Edit** > **Preferences** > **GNS3 VM** and set the resources:

- **RAM**: 8 GB minimum (each vMX node needs 2 GB)
- **vCPUs**: 4 or more

Click **OK** to apply. GNS3 will restart the VM with the updated resources.

!!! warning "Nested virtualization — VMware setting"
    QEMU requires hardware virtualization support. In VMware Workstation, open the GNS3 VM's settings, go to **Processors**, and enable **"Virtualize Intel VT-x/EPT or AMD-V/RVI"**. This must be set before starting the GNS3 VM.

Start the GNS3 VM and wait for it to show its IP address before launching GNS3.

## Step 3: Obtain the vMX Image

These labs use the **Juniper vMX 14.1** image (`hda.qcow2`). The image is available from:

- **Juniper Software Portal** — [support.juniper.net](https://support.juniper.net) (free account required) > Downloads > Junos Platforms > Virtual > vMX
- **Existing EVE-NG installation** — copy from `/opt/unetlab/addons/qemu/vmx-14.1R<x>.<y>/hda.qcow2`

## Step 4: Create the GNS3 QEMU Template

1. In GNS3, go to **Edit** > **Preferences** > **QEMU** > **QEMU VMs** > **New**
2. Set **Name**: `vMX-14.1`
3. Set **RAM**: `2048`
4. Leave QEMU binary as default (`qemu-system-x86_64`)
5. Click **Next**, select **New Image**, browse to your `hda.qcow2` — GNS3 uploads it to the GNS3 VM automatically
6. Click **Finish**

Then right-click the template and choose **Edit**:

| Tab | Setting | Value |
|-----|---------|-------|
| General | vCPUs | `1` |
| HDD | Disk interface | `ide` |
| Network | Adapters | `6` |
| Network | Type | `virtio-net-pci` |
| Advanced | Additional settings | `-serial mon:stdio -nographic -M pc` |
| Advanced | Use KVM if available | checked |

!!! warning "Machine type must be `pc`"
    The `-M pc` flag forces the legacy i440FX machine type. Without it, QEMU 8.x defaults to `q35` and vMX hangs at `mount /dev/vda /boot` on every boot.

!!! note "Why 6 adapters?"
    vMX reserves the first two adapters for management interfaces — Adapter 0 becomes `em0` (external management) and Adapter 1 becomes `em1` (internal, pre-configured at `172.16.0.1/16`). Data plane interfaces start at Adapter 2. Six adapters gives you `ge-0/0/0` through `ge-0/0/3` for lab use.

    | GNS3 Adapter | Junos Interface |
    |---|---|
    | Adapter 0 | `em0` (management) |
    | Adapter 1 | `em1` (internal) |
    | Adapter 2 | `ge-0/0/0` |
    | Adapter 3 | `ge-0/0/1` |
    | Adapter 4 | `ge-0/0/2` |
    | Adapter 5 | `ge-0/0/3` |

Click **OK** to save the template.

## Step 5: First Boot Procedure

vMX requires a one-time reboot on first boot to apply its network-services configuration. This only happens once per fresh image instance.

1. Drag a **vMX-14.1** node onto the canvas and start it
2. Open the console — wait 3–5 minutes for boot to complete
3. At the login prompt, type `root` and press Enter (no password on a fresh image)
4. Type `cli` to enter the Junos CLI from the BSD shell:

```text
--- JUNOS 14.1R4.8 built 2015-01-28 03:38:12 UTC
root@% cli
root>
```

5. You will see a warning during first boot:

```text
WARNING: Chassis configuration for network services has been changed.
A system reboot is mandatory. Please reboot the system NOW.
```

6. Reboot immediately:

```junos
request system reboot
```

Confirm with `yes`. After the second boot the warning will not appear again.

## Step 6: CLI Setup

After the second boot, log back in (`root` / `cli`) and run these two preferences before entering configuration mode:

```junos
set cli screen-length 0
set cli complete-on-space off
```

`screen-length 0` disables the `--More--` pager so `show` output never pauses mid-screen. `complete-on-space off` prevents the spacebar from triggering keyword completion when pasting multi-word commands.

!!! note "These do not persist across reconnects"
    Re-run both commands at the start of every console session. They apply to the current session only.
