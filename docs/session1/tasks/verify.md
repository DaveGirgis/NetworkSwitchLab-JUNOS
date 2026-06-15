# Session 1 — Verification Checklist

Complete these checks before moving to Session 2.

## Platform

- [ ] GNS3 and GNS3 VM are the same version and communicate successfully
- [ ] vJunos-router template is created with 4 adapters and 4096 MB RAM
- [ ] Both R1 and R2 boot successfully (Junos `root@` prompt appears)

## CLI Fundamentals

- [ ] You can switch between operational mode (`>`) and configuration mode (`#`)
- [ ] `show version` shows Junos 23.2R1 (or later) and model `vjunos-router`
- [ ] R1 hostname is set to `R1`, R2 hostname is set to `R2`
- [ ] Root password is configured on both routers
- [ ] You have used `show | compare` to review a change before committing

## Quick Commands

Run these on each router and confirm the output matches expectations:

```junos
show version
```
Expected: `Model: vjunos-router`, Junos version 23.2R1+

```junos
show system information
```
Expected: hostname `R1` (or `R2`), uptime since boot

```junos
show interfaces terse
```
Expected: `ge-0/0/0` through `ge-0/0/3` listed, all `down` (no IP yet)
