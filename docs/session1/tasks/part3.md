# Part 3 — Junos CLI Fundamentals

This part covers the core Junos concepts you will use in every subsequent session. Take your time here — understanding the commit model is the single most important prerequisite for working in Junos.

---

## The Two CLI Modes

Junos has two distinct CLI modes:

| Mode | Prompt | Purpose |
|------|--------|---------|
| Operational | `root>` | Read-only monitoring — `show`, `ping`, `traceroute` |
| Configuration | `root#` | Staged editing — all changes go to a candidate config |

Switch between them:

```junos
root> configure
Entering configuration mode

[edit]
root#
```

```junos
[edit]
root# exit
root>
```

---

## The Candidate Configuration

!!! note "This is the most important Junos concept"
    Every change you make in configuration mode modifies a **candidate configuration** — a draft copy that has no effect on the running router until you `commit`. This means you can make multiple changes, review them with `show | compare`, and then apply them all at once — or discard them completely with `rollback 0`.

### Example: Set the hostname

On R1's console:

```junos
root> configure

[edit]
root# set system host-name R1

[edit]
root# show | compare
[edit system]
+ host-name R1;

[edit]
root# commit
commit complete

[edit]
root# exit
R1>
```

The prompt changed to `R1>` — the hostname is now live.

### Rollback

If you make a mistake, `rollback 1` reverts to the configuration that was in place before your last `commit`:

```junos
[edit]
R1# rollback 1
load complete

[edit]
R1# commit
commit complete
```

Junos keeps up to 50 rollback generations (`rollback 0` through `rollback 49`).

---

## Configuration Hierarchy

Junos configuration is **hierarchical** — like a nested tree. You navigate it with `edit` to go deeper and `up` / `top` to go back up.

```junos
[edit]
R1# edit system

[edit system]
R1# set host-name R1

[edit system]
R1# top
```

Alternatively, you can issue the full path from anywhere:

```junos
[edit]
R1# set system host-name R1
```

Both are equivalent. Full-path `set` commands are used throughout these labs for clarity.

---

## Useful Operational Commands

| Command | Purpose |
|---------|---------|
| `show version` | Junos version and platform |
| `show interfaces terse` | All interfaces and their IP addresses |
| `show interfaces ge-0/0/0` | Detail for a single interface |
| `show route` | Routing table (all protocols) |
| `show system uptime` | System uptime |
| `ping 10.1.1.2` | ICMP ping |
| `traceroute 10.1.1.2` | Path trace |

---

## Useful Configuration Commands

| Command | Purpose |
|---------|---------|
| `configure` | Enter configuration mode |
| `set ...` | Stage a configuration change |
| `delete ...` | Stage a configuration deletion |
| `show \| compare` | Show pending changes (diff vs committed) |
| `commit` | Apply staged changes |
| `commit confirmed 5` | Apply changes; auto-rollback in 5 min if not confirmed |
| `rollback 0` | Discard all staged changes |
| `rollback 1` | Revert to the previous committed configuration |
| `show` | Show current candidate configuration at this level |
| `show \| display set` | Show candidate config in `set` command format |

---

## Task: Configure Both Routers

On **R1**:

```junos
root> configure

[edit]
root# set system host-name R1
root# set system root-authentication plain-text-password
New password: lab123
Retype new password: lab123

[edit]
root# commit
commit complete
```

On **R2** (open R2's console via GNS3):

```junos
root> configure

[edit]
root# set system host-name R2
root# set system root-authentication plain-text-password
New password: lab123
Retype new password: lab123

[edit]
root# commit
commit complete
```

!!! warning "Set a root password"
    Junos will warn you if you `commit` without setting a root password. In production this is a safety gate; in the lab it is acceptable to set `lab123` as shown above.
