# Session 1 — GNS3 Platform & Junos CLI

## Objectives

By the end of this session you will be able to:

- [ ] Download and import the vMX 14.1 QEMU image into GNS3
- [ ] Create a two-router GNS3 project and boot both nodes successfully
- [ ] Navigate Junos operational mode and configuration mode
- [ ] Apply, verify, and roll back a basic configuration
- [ ] Use `show | compare` to review staged changes before committing

## Prerequisites

- GNS3 2.2.44+ and GNS3 VM installed
- Virtualization enabled in BIOS (VT-x / AMD-V)
- ~3 GB free disk space for the vMX image
- Juniper account (free) to download the image

## Topology Overview

```mermaid
graph LR
  R1([R1<br/>vMX-14.1]) ---|ge-0/0/0| R2([R2<br/>vMX-14.1])
```

Two routers connected back-to-back. No routing protocol yet — the goal is platform familiarity.

## Session Parts

| Part | Topic |
|------|-------|
| [Part 0](tasks/part0.md) | Download & import the vMX-14.1 image |
| [Part 1](tasks/part1.md) | Create a GNS3 project and add devices |
| [Part 2](tasks/part2.md) | Connect devices and boot |
| [Part 3](tasks/part3.md) | Junos CLI fundamentals |
| [Verification](tasks/verify.md) | Checklist |
