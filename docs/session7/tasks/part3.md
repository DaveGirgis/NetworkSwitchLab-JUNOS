# Part 3 — RSVP-TE Intro

## Overview

LDP distributes labels hop-by-hop, following the same path the IGP selects. Every router independently computes the best path and assigns labels — the result is a label-switched path that mirrors the IS-IS shortest path. There is no global coordination, no bandwidth reservation, and no ability to steer traffic onto an explicit path that differs from the IGP result.

**RSVP-TE** (Resource Reservation Protocol — Traffic Engineering) solves these limitations. It is a signaling protocol that establishes an **LSP** (Label Switched Path) with an explicit, end-to-end setup:

1. The ingress router (PE1) sends an RSVP **PATH** message toward the egress (PE2), specifying the desired path and any bandwidth constraint
2. Each transit router (P1, P2) allocates resources and records the state
3. The egress router returns an RSVP **RESV** message back along the path, allocating labels hop-by-hop
4. The ingress router receives the RESV and the LSP is established

The result is a **signaled LSP** — every router on the path knows it exists, the exact path is fixed at the ingress, and bandwidth can be reserved.

### LDP vs RSVP-TE

| | LDP | RSVP-TE |
|-|-----|---------|
| Path | IGP shortest path | Explicit or CSPF-computed |
| Bandwidth | No reservation | Optional reservation |
| Signaling | Distributed, per-hop | End-to-end signaling from ingress |
| Failure recovery | Re-convergence with IGP | Fast Reroute (pre-computed backup, sub-50ms) |
| Complexity | Low | Higher |

In production deployments both often run simultaneously. LDP provides baseline label connectivity for all destinations. RSVP-TE LSPs are created for specific traffic-engineered paths — typically between PE routers — and BGP can be directed to prefer the RSVP LSP over the LDP path.

## Step 1: Enable IS-IS Traffic Engineering Extensions

RSVP-TE uses **CSPF** (Constrained Shortest Path First) to compute the explicit path. CSPF uses the IGP topology enriched with TE attributes — link bandwidth, TE metric, admin groups. IS-IS floods these attributes in TE extensions (TLV 22).

Enable IS-IS TE extensions on all four provider routers. This is a single command per router.

### PE1

```junos
configure

set protocols isis traffic-engineering

commit
```

### P1

```junos
configure

set protocols isis traffic-engineering

commit
```

### P2

```junos
configure

set protocols isis traffic-engineering

commit
```

### PE2

```junos
configure

set protocols isis traffic-engineering

commit
```

!!! note "IS-IS adjacencies remain stable"
    Enabling TE extensions adds new TLVs to IS-IS LSPs. Existing adjacencies are not reset — they continue to carry traffic normally. The new TLVs propagate within the next IS-IS LSP refresh cycle (default 30 minutes, or triggered immediately if you run `clear isis database purge`).

## Step 2: Enable RSVP on Provider Interfaces

RSVP needs to be enabled on every interface along the path of the LSP. Enable it on all four provider routers.

### PE1

```junos
configure

set protocols rsvp interface ge-0/0/0.0

commit
```

### P1

```junos
configure

set protocols rsvp interface ge-0/0/0.0
set protocols rsvp interface ge-0/0/1.0

commit
```

### P2

```junos
configure

set protocols rsvp interface ge-0/0/0.0
set protocols rsvp interface ge-0/0/1.0

commit
```

### PE2

```junos
configure

set protocols rsvp interface ge-0/0/0.0

commit
```

## Step 3: Create an LSP from PE1 to PE2

The LSP is defined only on the **ingress** router (PE1). The `to` address is the egress router's loopback. No explicit path or bandwidth constraint is specified here — CSPF will compute the best path using the IS-IS TE topology.

On **PE1**:

```junos
configure

set protocols mpls label-switched-path PE1-TO-PE2 to 10.0.0.4

commit
```

!!! note "The LSP is one-directional"
    `PE1-TO-PE2` only creates a path from PE1 toward PE2. To have a bidirectional engineered path, configure a second LSP on PE2 pointing to PE1. For this intro, one direction is sufficient to see the mechanism.

## Step 4: Verify RSVP Neighbors

```junos
PE1> show rsvp neighbor
```

Expected — P1 appears as an RSVP neighbor:

```text
RSVP neighbor: 1 learned
Address            Idle    Up/Dn    LastChange  HelloInt  HelloTx/Rx  MsgRx
10.1.12.2           0      1/0           0:42         9    5/5           0
```

Each router only peers RSVP with directly adjacent routers (unlike BGP which can peer to loopbacks). The `HelloTx/Rx` counter confirms hellos are flowing bidirectionally.

```junos
P1> show rsvp neighbor
```

Expected — PE1 and P2 both appear as neighbors:

```text
RSVP neighbor: 2 learned
Address            Idle    Up/Dn    LastChange  HelloInt  HelloTx/Rx  MsgRx
10.1.12.1           0      1/0           1:10         9    8/8           0
10.1.23.2           0      1/0           1:08         9    8/8           0
```

## Step 5: Verify the RSVP-TE LSP

```junos
PE1> show rsvp session
```

Expected — the LSP is active:

```text
Ingress RSVP: 1 sessions
To              From            State   Rt Style Labelin Labelout LSPname
10.0.0.4        10.0.0.1        Up       0  1 FF       -   299824 PE1-TO-PE2

Total 1 displayed, Up 1, Down 0
```

The **State: Up** confirms the LSP was successfully signaled from PE1 to PE2. **Labelout: 299824** is the label PE1 will push when forwarding traffic into this LSP. **FF** (Fixed-Filter) is the RSVP reservation style — one label is reserved per sender/destination pair.

```junos
PE1> show mpls lsp
```

Expected:

```text
Ingress LSP: 1 sessions
To              From            State Rt P     ActivePath       LSPname
10.0.0.4        10.0.0.1        Up     0 *                      PE1-TO-PE2

Total 1 displayed, Up 1, Down 0

Egress LSP: 0 sessions
Total 0 displayed, Up 0, Down 0

Transit LSP: 0 sessions
Total 0 displayed, Up 0, Down 0
```

PE1 shows one **Ingress LSP** (PE1 is the head-end). Check P1 and P2 to see them as transit LSRs:

```junos
P1> show mpls lsp
```

Expected on P1:

```text
Ingress LSP: 0 sessions
Total 0 displayed, Up 0, Down 0

Egress LSP: 0 sessions
Total 0 displayed, Up 0, Down 0

Transit LSP: 1 sessions
To              From            State   Rt Style Labelin Labelout LSPname
10.0.0.4        10.0.0.1        Up       0  1 FF  299824   299840 PE1-TO-PE2

Total 1 displayed, Up 1, Down 0
```

P1 is a **Transit LSP** — it receives traffic labeled 299824 (from PE1) and swaps it to 299840 (toward P2).

!!! note "RSVP-TE and LDP coexist"
    Both LDP and RSVP-TE are now running. The RSVP LSP is visible in inet.3 alongside LDP routes, but Junos needs explicit configuration (`traffic-engineering bgp-igp` or LSP-to-FEC binding) to steer BGP next-hops toward the RSVP LSP. Without that, BGP continues using the LDP paths from Part 2. Exploring those mechanisms is beyond this session's scope — the goal here is to understand how RSVP-TE signals and establishes a path.

## Step 6: Verify RSVP on PE2

On PE2, confirm the LSP arrives correctly:

```junos
PE2> show rsvp session
```

Expected:

```text
Egress RSVP: 1 sessions
To              From            State   Rt Style Labelin Labelout LSPname
10.0.0.4        10.0.0.1        Up       0  1 FF  3        -      PE1-TO-PE2

Total 1 displayed, Up 1, Down 0
```

PE2 appears as the **Egress** for the LSP. **Labelin: 3** is implicit-null — P2 performs PHP (pops the label before sending to PE2), so PE2 receives a plain IP packet. **Labelout: -** confirms no outgoing label (PE2 is the endpoint).
