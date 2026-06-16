# Session 1 — Verification Checklist

Complete these checks before moving to Session 2.

## Platform

- [ ] GNS3 and GNS3 VM are the same version and communicate successfully
- [ ] vMX-14.1 template is created with 6 adapters and 2048 MB RAM
- [ ] Both R1 and R2 boot successfully (Junos `root@` prompt appears)

## CLI Fundamentals

- [ ] You can switch between operational mode (`>`) and configuration mode (`#`)
- [ ] `show version` shows Junos 14.1R4.8 and model `vmx`
- [ ] R1 hostname is set to `R1`, R2 hostname is set to `R2`
- [ ] Root password is configured on both routers
- [ ] You have used `show | compare` to review a change before committing

## Quick Commands

Run these on each router and confirm the output matches expectations:

```junos
show version
```
Expected: `Model: vmx`, Junos version 14.1R4.8

```junos
show version | match Hostname
```
Expected: `Hostname: R1` (or `R2`)

```junos
show interfaces terse
```
Expected: `em0`, `em1`, `lo0` listed. `ge-0/0/x` interfaces only appear after an IP address is configured — query them directly with `show interfaces ge-0/0/0 terse`
