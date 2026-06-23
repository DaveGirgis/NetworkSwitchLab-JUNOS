# Part 0 — Enable Enhanced-Ethernet Mode

vMX supports bridge domains only in `enhanced-ethernet` network-services mode. The default after initial setup is `enhanced-ip`. You must change this mode and reboot **before** any bridging configuration will take effect.

!!! danger "Reboot required"
    Changing `network-services` is a chassis-level change. The router must be rebooted immediately after committing — there is no way to apply it live. This is expected behaviour.

## CLI Setup

Before pasting any configuration, run these two commands at the operational prompt (`>`) on each switch:

```junos
set cli screen-length 0
set cli complete-on-space off
```

Re-run these at the start of every console session — they do not persist across reconnects.

!!! tip "Pasting large config blocks"
    Use `load set terminal` in config mode instead of pasting directly at the `#` prompt. Junos buffers all input until you press **Ctrl+D**, then processes every set command at once — eliminating paste-rate issues with console terminals.

    ```text
    configure
    load set terminal
    set system host-name SW1
    set interfaces ge-0/0/0 ...
    ^D
    commit
    ```

## Step 1: Set hostnames

On SW1:

```junos
configure
set system host-name SW1
set system root-authentication plain-text-password
New password: lab123
Retype new password: lab123
commit
```

On SW2:

```junos
configure
set system host-name SW2
set system root-authentication plain-text-password
New password: lab123
Retype new password: lab123
commit
```

## Step 2: Change network-services mode

Apply this on **both SW1 and SW2**:

```junos
configure
set chassis network-services enhanced-ethernet
commit
```

Expected output:

```text
commit complete
```

## Step 3: Reboot

On each router immediately after commit:

```junos
request system reboot
```

Confirm with `yes`. Wait 3–5 minutes for the router to come back up.

## Step 4: Verify the mode change

After reboot, confirm the mode:

```junos
show chassis network-services
```

Expected:

```text
Network Services Mode: Enhanced-Ethernet
```

!!! tip "If you see Enhanced-IP"
    The mode did not apply. Re-enter configuration mode, re-apply `set chassis network-services enhanced-ethernet`, commit, and reboot again. Confirm you typed the value exactly — `enhanced-ethernet` not `enhanced_ethernet`.
