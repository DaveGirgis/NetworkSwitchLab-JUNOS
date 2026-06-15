# Session 1 — Troubleshooting

## GNS3 VM Not Detected

**Symptom:** GNS3 shows "GNS3 VM is not running" or the VM status is red.

**Cause / Fix:**

1. Open VMware and confirm the GNS3 VM is powered on
2. In GNS3: **Edit** > **Preferences** > **GNS3 VM** — verify the VM IP appears
3. If the IP field is empty, restart the GNS3 VM and click the refresh button
4. Confirm the GNS3 app version matches the GNS3 VM version exactly

---

## vJunos-router Node Shows Red X

**Symptom:** After starting nodes, one or both nodes show a red X instead of green.

**Cause / Fix:**

1. Right-click the node > **Show log** — look for QEMU errors
2. Most common cause: **nested virtualization not enabled** in VMware settings
   - Shut down the GNS3 VM
   - VM Settings > Processors > enable "Virtualize Intel VT-x/EPT or AMD-V/RVI"
   - Restart the GNS3 VM and try again
3. Second common cause: insufficient RAM — confirm GNS3 VM has at least 8 GB

---

## Console Shows No Output / Stays Blank

**Symptom:** Console opens but nothing appears, or the cursor blinks with no boot messages.

**Cause / Fix:**

1. Wait — vJunos-router takes 3–5 minutes to boot; a blank console is normal for the first 60–90 seconds
2. Press Enter once — this sometimes wakes a stalled console
3. If still blank after 10 minutes, right-click the node > **Stop**, then **Start** again

---

## `commit` Returns Error: "Must be authenticated"

**Symptom:**

```text
[edit]
R1# commit
error: configuration check-out failed
```

**Cause:** Another session has the configuration locked (e.g., you have two consoles open on the same node).

**Fix:** Close duplicate console windows. Only one configuration session should be active per node.

---

## Hostname Does Not Change After `commit`

**Symptom:** You set `system host-name R1` and committed, but the prompt still shows `root`.

**Cause / Fix:**

- The hostname change takes effect in **new** CLI sessions. Exit and re-enter:

```junos
[edit]
root# exit
root> exit
root@% cli
R1>
```

If you exited back to the shell (`root@%`), re-enter the CLI with `cli`.

---

## `show | compare` Shows Nothing

**Symptom:** You made changes but `show | compare` returns nothing.

**Cause:** You have not made any changes in this configuration session, OR you inadvertently ran `rollback 0` which discarded the changes.

**Fix:** Re-enter configuration mode and re-apply your changes.
