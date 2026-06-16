# NetworkSwitchLab-JUNOS Project Context

## Overview

**NetworkSwitchLab-JUNOS** is a JNCIS-SP lab guide series for adult professionals using vJunos-router (MX-based) images in GNS3, built with MkDocs Material and published via GitHub Pages.

**Repository:** `https://github.com/DaveGirgis/NetworkSwitchLab-JUNOS`  
**Student site:** `https://DaveGirgis.github.io/NetworkSwitchLab-JUNOS/`  
**Working directory:** `C:\Users\LocalUser\Downloads\Claude-Code\Network-SwitchLab-JUNOS`  
**Branch:** main (direct commits, no feature branches)

---

## Development Workflow

Dev server runs at `http://localhost:8000` via `mkdocs serve`. Changes commit directly to main and push without PRs — documentation repo, branching overhead is unnecessary.

---

## Session Progress

Ten sessions are planned:
- Session 1 (GNS3 + CLI) — complete ✅
- Session 2 (Interfaces + Static Routing) — complete ✅
- Session 3 (Bridging & VLANs) — complete ✅
- Session 4 (OSPF) — complete ✅ (renumbered from Session 3)
- Sessions 5–9 — pending 🔜

---

## File Structure (All Sessions)

```
docs/session{N}/
├── index.md              # Objectives, outcomes, prerequisites
├── topology.md           # Mermaid diagrams, device/link summary
├── addressing.md         # IP address tables, loopback table
├── tasks/
│   ├── part0.md          # Base config (hostname, interfaces up)
│   ├── part1.md ... partN.md
│   └── verify.md         # Verification checklist + show commands
└── reference/
    ├── commands.md       # Full command reference + wrap-up
    └── troubleshooting.md # Symptom / cause / fix sections
```

New sessions require updates to `mkdocs.yml` nav and `docs/index.md`.

---

## Hardware / Image Rules

**vMX specifics (Junos 14.1R4.8, VCP-only single VM):**
- Image file: `hda.qcow2`
- Interface naming: `ge-0/0/0` through `ge-0/0/3` (data interfaces)
- Loopback: `lo0` with `unit 0 family inet address X.X.X.X/32`
- OSPF/IS-IS/MPLS run under `[edit protocols]`
- Routing options (static routes, router-id, AS): `[edit routing-options]`
- VRFs: `[edit routing-instances <name>]` with `instance-type vrf`
- BGP: `[edit protocols bgp]` with `group` stanzas
- Firewall filters: `[edit firewall family inet filter <name>]`
- Commit model: always `commit` or `commit confirmed <minutes>`

**GNS3 setup notes:**
- vMX requires QEMU/KVM — runs inside GNS3 VM (Linux)
- RAM per node: 2048 MB; vCPUs: 1
- QEMU options: `-serial mon:stdio -nographic -M pc` (`-M pc` required — QEMU 8.x defaults to q35 which causes boot hang)
- Adapters: 6 total — Adapter 0 = em0 (mgmt), Adapter 1 = em1 (internal), Adapter 2 = ge-0/0/0, Adapter 3 = ge-0/0/1, Adapter 4 = ge-0/0/2, Adapter 5 = ge-0/0/3
- Always connect GNS3 topology links on Adapter 2+ (never Adapter 0 or 1)
- First boot: reboot required due to network-services initialization warning
- Boot time: ~3-5 minutes — warn students not to start configs until `root@%` prompt appears
- Console access: right-click > Console in GNS3; type `root` to login, then `cli` for Junos CLI

---

## Core Topology (Sessions 3–8)

Four provider routers + two customer edge routers:

```
CE1 --- PE1 --- P1 --- P2 --- PE2 --- CE2
```

**Loopbacks:**
- PE1: 10.0.0.1/32
- P1:  10.0.0.2/32
- P2:  10.0.0.3/32
- PE2: 10.0.0.4/32
- CE1: 10.0.0.11/32
- CE2: 10.0.0.12/32

**Provider p2p links (/30):**
- PE1 — P1: 10.1.12.0/30 (PE1 ge-0/0/0 = .1, P1 ge-0/0/0 = .2)
- P1 — P2: 10.1.23.0/30 (P1 ge-0/0/1 = .1, P2 ge-0/0/0 = .2)
- P2 — PE2: 10.1.34.0/30 (P2 ge-0/0/1 = .1, PE2 ge-0/0/0 = .2)
- PE1 — CE1: 172.16.1.0/30 (PE1 ge-0/0/1 = .1, CE1 ge-0/0/0 = .2)
- PE2 — CE2: 172.16.2.0/30 (PE2 ge-0/0/1 = .1, CE2 ge-0/0/0 = .2)

**BGP ASNs:**
- Provider: 65001
- Customer A: 65100

---

## Documentation Standards

- Mermaid `graph LR` diagrams only (no ASCII art)
- Plain ASCII hyphens in comments (never Unicode em dashes)
- No hidden Unicode in code blocks
- Code blocks use `junos` or `text` language tags for syntax highlighting
- Admonition types: `warning`, `note`, `tip`, `danger`

---

## Common Utilities

**Scan for hidden Unicode:**
```python
with open('docs/sessionX/tasks/partY.md', 'rb') as f:
    content = f.read()
for i, byte in enumerate(content):
    if byte > 127 or (byte < 32 and byte not in (9, 10, 13)):
        line = content[:i].count(10) + 1
        print(f'Line ~{line}: byte=0x{byte:02X} at offset {i}')
```

**Push changes:**
```bash
git add <files>
git commit -m "message"
git push origin main
```
