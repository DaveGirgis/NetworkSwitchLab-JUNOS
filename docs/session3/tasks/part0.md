# Part 0 — Build Topology & Base Config

## Step 1: Create the GNS3 Project

1. In GNS3: **File** > **New blank project** > name it `JNCIS-SP-Core`
2. Drag four **vJunos-router** nodes onto the canvas
3. Rename them `PE1`, `P1`, `P2`, `PE2` (right-click > Change hostname in GNS3)

## Step 2: Draw the Links

| Link | Node A | Adapter | Node B | Adapter |
|------|--------|---------|--------|---------|
| PE1 — P1 | PE1 | 0 (ge-0/0/0) | P1 | 0 (ge-0/0/0) |
| P1 — P2 | P1 | 1 (ge-0/0/1) | P2 | 0 (ge-0/0/0) |
| P2 — PE2 | P2 | 1 (ge-0/0/1) | PE2 | 0 (ge-0/0/0) |

## Step 3: Start All Nodes

Click **Start all nodes**. Wait 3–5 minutes for all four nodes to fully boot before opening consoles.

!!! warning "RAM requirement"
    Four vJunos-router nodes require **16 GB** of RAM in the GNS3 VM. If the GNS3 VM has less than 16 GB, start the nodes two at a time and configure them in pairs.

## Step 4: Apply Base Config

Open a console on each node and apply the base configuration.

### PE1

```junos
configure

set system host-name PE1
set system root-authentication plain-text-password
New password: lab123
Retype new password: lab123

commit
```

### P1

```junos
configure

set system host-name P1
set system root-authentication plain-text-password
New password: lab123
Retype new password: lab123

commit
```

### P2

```junos
configure

set system host-name P2
set system root-authentication plain-text-password
New password: lab123
Retype new password: lab123

commit
```

### PE2

```junos
configure

set system host-name PE2
set system root-authentication plain-text-password
New password: lab123
Retype new password: lab123

commit
```

## Verify Base Config

On each router, confirm the prompt shows the correct hostname:

```text
PE1>
P1>
P2>
PE2>
```
