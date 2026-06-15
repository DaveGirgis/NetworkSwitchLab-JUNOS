# Session 1 — Troubleshooting

## GNS3 VM Not Detected

**Symptom:** GNS3 shows "GNS3 VM is not running" or the VM status is red.

**Cause / Fix:**

1. Open VMware and confirm the GNS3 VM is powered on
2. In GNS3: **Edit** > **Preferences** > **GNS3 VM** — verify the VM IP appears
3. If the IP field is empty, restart the GNS3 VM and click the refresh button
4. Confirm the GNS3 app version matches the GNS3 VM version exactly

---

## vMX Node Shows Red X

**Symptom:** After starting nodes, one or both nodes show a red X instead of green.

**Cause / Fix:**

1. Right-click the node > **Show log** — look for QEMU errors
2. Most common cause: **nested virtualization not enabled** in VMware settings
   - Shut down the GNS3 VM
   - VM Settings > Processors > enable "Virtualize Intel VT-x/EPT or AMD-V/RVI"
   - Restart the GNS3 VM and try again
3. Second common cause: insufficient RAM — confirm GNS3 VM has at least 8 GB

---

## Boot Hangs at `mount /dev/vda /boot` or `mount /dev/hda1 /boot`

**Symptom:** The console shows boot messages but freezes at a mount line.

**Cause:** QEMU 8.x defaults to the `q35` machine type. vMX was built for the older `pc` (i440FX) machine type and stalls during boot when `q35` is used.

**Fix:**

1. Stop the node (right-click > **Stop**)
2. Go to **Edit** > **Preferences** > **QEMU** > **QEMU VMs**
3. Select **vMX-14.1** and click **Edit**
4. Go to the **Advanced** tab
5. Confirm **Additional settings** contains `-M pc`:

```
-serial mon:stdio -nographic -M pc
```

6. Click **OK**, then start the node again

---

## First Boot Warning: "Chassis configuration for network services has been changed"

**Symptom:** On first boot, the console prints:

```text
WARNING: Chassis configuration for network services has been changed.
A system reboot is mandatory. Please reboot the system NOW.
```

**Cause:** This is expected on a fresh image — vMX is setting its network-services mode for the first time.

**Fix:** This is a one-time initialization. Log in as `root`, type `cli`, then reboot:

```junos
request system reboot
```

Confirm with `yes`. The warning will not appear on subsequent boots of the same node instance.

---

## Console Shows No Output / Stays Blank

**Symptom:** Console opens but nothing appears.

**Cause / Fix:**

1. Wait — vMX takes 3–5 minutes to boot; a blank console is normal for the first 60–90 seconds
2. Press Enter once — this sometimes wakes a stalled console
3. If still blank after 10 minutes, right-click the node > **Stop**, then **Start** again

---

## `ge-0/0/0` Not Reachable After Connecting Nodes

**Symptom:** You connected two nodes and configured `ge-0/0/0` but pings fail.

**Cause:** The link was drawn using **Adapter 0** or **Adapter 1** instead of **Adapter 2**. vMX reserves the first two adapters for management — `ge-0/0/0` maps to Adapter 2.

**Fix:** Delete the link in GNS3, redraw it connecting **Adapter 2** on each node.

---

## Hostname Does Not Change After `commit`

**Symptom:** You set `system host-name R1` and committed, but the prompt still shows `root`.

**Fix:** Exit and re-enter the CLI:

```junos
[edit]
root# exit
root> exit
root@% cli
R1>
```
