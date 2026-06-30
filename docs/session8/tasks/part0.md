# Part 0 — Baseline Verification

## CLI Setup

Before pasting any configuration, run these two commands at the operational prompt (`>`) on each router:

```junos
set cli screen-length 0
set cli complete-on-space off
```

Re-run these at the start of every console session — they do not persist across reconnects.

!!! tip "Pasting large config blocks"
    Use `load set terminal` in config mode to paste multiple set commands at once. Press **Ctrl+D** when done.

## Step 1: Verify MPLS Is Operational

Session 8 builds directly on the LDP transport from Session 7. Confirm LDP is healthy before making any changes.

On **PE1**:

```junos
show ldp session
```

Expected — session to P1 in `Operational` state:

```text
Address                          State       Connection  Hold time  I/F
10.0.0.2                         Operational Open          27        3
```

```junos
show route table inet.3
```

Expected — three LDP-resolved routes, 10.0.0.4 shows Push label:

```text
inet.3: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32        *[LDP/9] ...
                    > to 10.1.12.2 via ge-0/0/0.0
10.0.0.3/32        *[LDP/9] ...
                    > to 10.1.12.2 via ge-0/0/0.0, Push 299792
10.0.0.4/32        *[LDP/9] ...
                    > to 10.1.12.2 via ge-0/0/0.0, Push 299808
```

!!! warning "Fix MPLS before continuing"
    If `inet.3` is empty or PE2's loopback is missing, return to Session 7 and restore LDP. The VPN label stack in Session 8 requires the outer transport label from LDP. Without it, VPN packets will not reach the egress PE.

## Step 2: Verify Existing BGP State

Session 6's BGP is still running. Confirm both eBGP (to CE1) and iBGP (to PE2) are up on PE1.

```junos
PE1> show bgp summary
```

Expected — two sessions, both established:

```text
Groups: 2 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0
                       2          2          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.0.4              65001         42         41       0       0       20:11 1/1/1/0              0/0/0/0
172.16.1.2            65100         89        105       0       0       44:22 1/1/1/0              0/0/0/0
```

Both peers show established counts. If either session shows `Active` or `Idle`, restore Session 6 BGP first.

## Step 3: Understand What Is About to Change

Before proceeding, note what the next three parts will do to the existing configuration:

| Component | Session 6 state | After Session 8 |
|-----------|-----------------|-----------------|
| PE1 `EBGP-CE1` group | Global eBGP to CE1 in `inet.0` | Deleted — replaced by VRF eBGP |
| PE2 `EBGP-CE2` group | Global eBGP to CE2 in `inet.0` | Deleted — replaced by VRF eBGP |
| PE iBGP group `IBGP` | Carries `inet-unicast` only | Gains `inet-vpn unicast` family |
| Customer routes | In `inet.0` (global table) | In `VPN-A.inet.0` (VRF table) |
| CE1 and CE2 config | eBGP to PE neighbor addresses | **Unchanged** — CEs are unaware of VRF |
| P1 and P2 config | MPLS transit | **Unchanged** |

The CE routers do not need to be reconfigured. They will see their BGP session briefly drop when the global eBGP group is removed from the PE, then come back up when the VRF-based eBGP is configured in Part 3.
