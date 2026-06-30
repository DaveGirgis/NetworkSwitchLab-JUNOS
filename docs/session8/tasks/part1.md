# Part 1 — VRF & Route Distinguishers

## Overview

A **VRF** (Virtual Routing and Forwarding instance) is a separate, isolated routing table within a single router. Junos calls these **routing instances**. Every interface assigned to a routing instance, and every route learned or generated through that interface, belongs exclusively to that instance — invisible to `inet.0` and to other instances.

In this part you will:

1. Remove the global eBGP groups from PE1 and PE2 — these placed customer routes into `inet.0` and are not compatible with VRF-based L3 VPN
2. Create a VRF named `VPN-A` on each PE and assign the CE-facing interface to it
3. Verify the VRF table contains the CE-facing link subnet as a direct route

The VRF parameters:

**`instance-type vrf`** — tells Junos this is a Layer 3 VPN routing instance, not a regular table or VPLS instance.

**`interface ge-0/0/1.0`** — moves the CE-facing interface into the VRF. Once assigned, this interface's routes (direct, connected) go into `VPN-A.inet.0` rather than `inet.0`.

**`route-distinguisher`** — the RD prepended to every VPN prefix this PE originates into MP-BGP. It must be unique per PE per VPN. PE1 uses `65001:100`, PE2 uses `65001:200`. The RD makes VPN prefixes globally unique in BGP so two customers using the same address space don't collide.

**`vrf-target`** — sets both the export and import route target to `target:65001:100`. Routes exported from this VRF carry that RT community. The VRF imports routes from MP-BGP that carry the same RT. Since both PEs use the same RT, they exchange each other's VPN routes automatically.

**`vrf-table-label`** — instructs Junos to allocate a single MPLS label for the entire VRF. This is the inner VPN label that appears in the two-label stack. When the egress PE receives a packet with this label, it identifies the VRF, strips the label, and performs an IP lookup inside `VPN-A.inet.0` to find the CE next-hop. Without this command the VRF will not have a label to advertise to remote PEs.

## Step 1: Remove Global eBGP Groups from PEs

The existing global eBGP groups (configured in Session 6) place CE routes directly into `inet.0`. They must be removed before the VRF takes ownership of the CE-facing interface.

On **PE1**:

```junos
configure

delete protocols bgp group EBGP-CE1

commit
```

On **PE2**:

```junos
configure

delete protocols bgp group EBGP-CE2

commit
```

!!! note "CE sessions will drop briefly"
    CE1 and CE2 will lose their BGP sessions to the PEs when these groups are deleted. This is expected. The sessions will re-establish after Part 3 when the VRF-based eBGP groups are configured.

## Step 2: Create the VRF on PE1

```junos
configure

set routing-instances VPN-A instance-type vrf
set routing-instances VPN-A interface ge-0/0/1.0
set routing-instances VPN-A route-distinguisher 65001:100
set routing-instances VPN-A vrf-target target:65001:100
set routing-instances VPN-A vrf-table-label

commit
```

## Step 3: Create the VRF on PE2

```junos
configure

set routing-instances VPN-A instance-type vrf
set routing-instances VPN-A interface ge-0/0/1.0
set routing-instances VPN-A route-distinguisher 65001:200
set routing-instances VPN-A vrf-target target:65001:100
set routing-instances VPN-A vrf-table-label

commit
```

!!! note "RD is unique per PE; RT is shared"
    PE1 uses `route-distinguisher 65001:100` and PE2 uses `65001:200` — these must differ. Both use `vrf-target target:65001:100` — this must match. The RT is what binds the two PEs into the same VPN; the RD just keeps their BGP advertisements distinct.

## Step 4: Verify the VRF Table

On **PE1**, check that the VRF routing table has been created and contains the CE-facing link subnet as a direct route:

```junos
show route table VPN-A.inet.0
```

Expected — the CE-facing link subnet appears as a direct route:

```text
VPN-A.inet.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

172.16.1.0/30      *[Direct/0] 00:00:42
                    > via ge-0/0/1.0
```

The route `172.16.1.0/30` is the directly connected subnet on the CE-facing interface, now living in the VRF rather than `inet.0`. This confirms the interface has been successfully assigned to the VRF.

Run the same check on **PE2**:

```junos
show route table VPN-A.inet.0
```

Expected:

```text
VPN-A.inet.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

172.16.2.0/30      *[Direct/0] 00:00:21
                    > via ge-0/0/1.0
```

Also confirm the global table **does not** contain the CE-facing subnets anymore:

```junos
PE1> show route table inet.0 172.16.1.0/30
```

Expected: no output. The route is now in `VPN-A.inet.0`, not `inet.0`.

## Step 5: Verify the VPN Label Was Allocated

`vrf-table-label` causes Junos to allocate an MPLS label for the VRF. Confirm the label was assigned:

```junos
PE1> show route table VPN-A.inet.0 detail | match VPN-Label
```

Expected to return no output at this stage — the VPN label appears on routes imported from the remote PE (it shows PE2's label in PE1's route table, and vice versa). The local label allocation is visible on the remote PE once MP-BGP is configured in Part 2.

Alternatively, verify the label indirectly:

```junos
PE2> show route table mpls.0 | match VPN
```

If `vrf-table-label` allocated a label correctly, it will appear in the MPLS forwarding table. The label number will vary.
