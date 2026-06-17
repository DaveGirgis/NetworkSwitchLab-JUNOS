# Part 0 — Base Configuration

Apply this base configuration to both routers. It sets the hostname and root password if you are starting from a fresh vMX boot.

## CLI Setup

Before pasting any configuration, run these two commands at the operational prompt (`>`):

```junos
set cli screen-length 0
set cli complete-on-space off
```

Re-run these at the start of every console session — they do not persist across reconnects.

## R1 — Base Config

```junos
configure

set system host-name R1
set system root-authentication plain-text-password
New password: lab123
Retype new password: lab123

commit
```

## R2 — Base Config

```junos
configure

set system host-name R2
set system root-authentication plain-text-password
New password: lab123
Retype new password: lab123

commit
```

## Verify

On each router, confirm the hostname appears in the prompt:

```text
R1>
R2>
```

If you completed Session 1 and saved your project, the hostnames are already set. Verify with:

```junos
show version | match Hostname
```

Expected:

```text
Hostname: R1
```
