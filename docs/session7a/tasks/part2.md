# Part 2 — VPLS (Virtual Private LAN Service)

## Overview

L2circuit in Part 1 provided a point-to-point pseudowire between exactly two sites. **VPLS** (Virtual Private LAN Service) generalizes this to a multipoint service: any number of CE sites can connect to the same virtual LAN, and all sites appear to be on the same Ethernet segment.

### How VPLS Differs from L2circuit

| | L2circuit | VPLS |
|-|-----------|------|
| Topology | Point-to-point (two sites) | Multipoint (any number of sites) |
| Signaling | Targeted LDP | BGP (l2vpn signaling) |
| MAC learning | None (transparent wire) | Full MAC table per VPLS instance |
| Flooding | All traffic forwarded | Unknown MACs flooded to all pseudowires |
| Configuration | `protocols l2circuit` | `routing-instances` type `vpls` |

### How VPLS Works in Junos

VPLS in Junos uses **BGP signaling** (sometimes called Kompella VPLS or RFC 4761). Each PE advertises its VPLS membership via iBGP using the `l2vpn signaling` address family. When PE1 receives a VPLS BGP update from PE2, it learns:

- PE2's loopback address (the pseudowire endpoint)
- The VC label PE2 has allocated for this VPLS instance

From this information, PE1 automatically builds a pseudowire to PE2, using the existing LDP transport labels for the outer label.

Within the VPLS instance, each PE runs a simple MAC learning bridge:

1. When a frame arrives from CE1 on ge-0/0/2, PE1 records `source-MAC → ge-0/0/2.0` in the VPLS MAC table
2. If the destination MAC is known, PE1 forwards directly (to the local CE port or the correct pseudowire)
3. If unknown, PE1 floods the frame out all pseudowires (split-horizon prevents flooding back onto the incoming interface)

CE1 and CE2 are completely unaware of the VPLS infrastructure. They see an Ethernet network with normal ARP and MAC learning behavior.

### Label Stack

VPLS uses the same two-label stack as L2circuit:

- **Outer label** — LDP transport label (same path through the backbone as Session 7)
- **Inner label** — VPLS VC label assigned by BGP, identifies the specific VPLS instance and source PE

P routers only process the outer label, as in Session 7.

## Step 1: Remove L2circuit Configuration

Part 2 reconfigures the same interfaces used in Part 1. Remove the L2circuit configuration first.

On **PE1**:

```junos
configure

delete interfaces ge-0/0/2 encapsulation
delete protocols l2circuit

commit
```

On **PE2**:

```junos
configure

delete interfaces ge-0/0/2 encapsulation
delete protocols l2circuit

commit
```

After committing, verify the pseudowire is gone:

```junos
PE1> show l2circuit connections
```

Expected: no output (no active L2circuit connections).

## Step 2: Add L2VPN Signaling to BGP

VPLS uses a BGP address family (`l2vpn signaling`) to distribute VPLS membership between PEs. Add this family to the existing iBGP group on both PEs.

On **PE1**:

```junos
configure

set protocols bgp group IBGP family l2vpn signaling

commit
```

On **PE2**:

```junos
configure

set protocols bgp group IBGP family l2vpn signaling

commit
```

Verify the BGP session is still up and now carries the L2VPN family:

```junos
PE1> show bgp neighbor 10.0.0.4 | match NLRI
```

Expected to show `l2vpn` in the negotiated NLRI families:

```text
  NLRI for restart configured on peer: inet-unicast l2vpn
  NLRI advertised by peer: inet-unicast l2vpn
  NLRI for this session: inet-unicast l2vpn
```

## Step 3: Configure the VPLS Routing Instance on PE1

```junos
configure

set routing-instances VPLS-100 instance-type vpls
set routing-instances VPLS-100 interface ge-0/0/2.0
set routing-instances VPLS-100 route-distinguisher 65001:100
set routing-instances VPLS-100 vrf-target target:65001:100
set routing-instances VPLS-100 protocols vpls no-tunnel-services
set routing-instances VPLS-100 protocols vpls site CE1-SITE site-identifier 1
set routing-instances VPLS-100 protocols vpls site CE1-SITE interface ge-0/0/2.0

commit
```

## Step 4: Configure the VPLS Routing Instance on PE2

```junos
configure

set routing-instances VPLS-100 instance-type vpls
set routing-instances VPLS-100 interface ge-0/0/2.0
set routing-instances VPLS-100 route-distinguisher 65001:101
set routing-instances VPLS-100 vrf-target target:65001:100
set routing-instances VPLS-100 protocols vpls no-tunnel-services
set routing-instances VPLS-100 protocols vpls site CE2-SITE site-identifier 2
set routing-instances VPLS-100 protocols vpls site CE2-SITE interface ge-0/0/2.0

commit
```

!!! note "Route distinguisher vs vrf-target"
    The **route-distinguisher** must be unique per PE per VPN (65001:100 on PE1, 65001:101 on PE2). It disambiguates BGP updates when two PEs advertise membership in the same VPN.

    The **vrf-target** (route target) must be identical on both PEs (target:65001:100). It controls which PEs import each other's VPLS membership updates. Both PEs export routes tagged with `target:65001:100` and import routes carrying the same tag — this is how PE1 discovers PE2 is part of VPLS-100 and vice versa.

!!! note "no-tunnel-services"
    On vMX, VPLS requires `no-tunnel-services` to use native Ethernet interfaces for Layer 2 forwarding rather than relying on a physical Tunnel PIC (which does not exist in the virtual image). Omitting this command may cause the VPLS instance to remain down.

## Step 5: Verify the VPLS Instance

Allow 15–30 seconds for BGP to exchange VPLS membership and for the pseudowire to establish.

On **PE1**:

```junos
show vpls connections
```

Expected when the VPLS pseudowire is up:

```text
Instance: VPLS-100
  Local site: CE1-SITE (1)
    connection-site           Type  St     Time last up          # Up trans
    2                         rmt   Up     Jun 26 12:00:00               1
      Remote PE: 10.0.0.4, Negotiated control-word: No
      Incoming label: 299856, Outgoing label: 299872
      Local interface: ge-0/0/2.0, Status: Up, Encapsulation: ETHERNET
```

The `connection-site 2` entry represents the pseudowire to PE2's site (site-identifier 2). State `Up` confirms the pseudowire is active.

On **PE1**, check the BGP VPLS routes:

```junos
show route table VPLS-100.l2vpn.0
```

Expected — two routes: PE1's own locally originated route, and PE2's route received via BGP:

```text
VPLS-100.l2vpn.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

65001:100:1/96
                   *[L2VPN/170] ...
                    Indirect
65001:101:2/96
                   *[BGP/170] ...
                    > to 10.1.12.2 via ge-0/0/0.0, Push 299872, Push 299808
```

The second route shows the BGP-learned VPLS membership from PE2: two labels are being pushed — the inner VPLS VC label (299872) and the outer LDP transport label (299808) for reaching PE2.

## Step 6: Check the MAC Table

After CE1 sends traffic, PE1 will learn its MAC address.

```junos
PE1> show vpls mac-table instance VPLS-100
```

Expected after at least one frame from CE1:

```text
MAC flags (S-static MAC, D-dynamic MAC, L-locally learned, C-Control MAC,
           SE-Statistics enabled, NM-Non-configured MAC, R-Remote PE MAC)

Routing instance : VPLS-100
 Bridging domain : __VPLS-100__, VLAN : NA
   MAC                 MAC      Logical          NH     RTR
   address             flags    interface        Index  ID
   aa:bb:cc:dd:ee:11   DL       ge-0/0/2.0
```

The `DL` flags indicate this MAC was **D**ynamically learned on a **L**ocal interface. After CE1 pings CE2 in Step 7, CE2's MAC will also appear in the table on PE2 with the `DL` flag, and PE1 will learn CE2's MAC as a remote entry (flag `R`) via the VPLS pseudowire.

## Step 7: Test CE-to-CE Connectivity

The test is identical to Part 1 — CE1 pings CE2 on the 192.168.1.0/24 subnet. The path is now through the VPLS instance rather than the L2circuit, but CE1 and CE2 see no difference.

```junos
CE1> ping 192.168.1.2 count 5
```

Expected — all five replies succeed:

```text
PING 192.168.1.2 (192.168.1.2): 56 data bytes
64 bytes from 192.168.1.2: icmp_seq=0 ttl=64 time=35.410 ms
64 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=42.037 ms
64 bytes from 192.168.1.2: icmp_seq=2 ttl=64 time=31.298 ms
64 bytes from 192.168.1.2: icmp_seq=3 ttl=64 time=38.851 ms
64 bytes from 192.168.1.2: icmp_seq=4 ttl=64 time=33.762 ms

--- 192.168.1.2 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 31.298/36.272/42.037/3.759 ms
```

After the ping, check the MAC table again on PE1:

```junos
PE1> show vpls mac-table instance VPLS-100
```

Now CE2's MAC should appear as a remote entry — PE1 learned it from PE2 via the VPLS pseudowire:

```text
   MAC                 MAC      Logical          NH     RTR
   address             flags    interface        Index  ID
   aa:bb:cc:dd:ee:11   DL       ge-0/0/2.0
   aa:bb:cc:dd:ee:22   R        vtep.32769
```

The `R` flag indicates a **R**emote MAC learned via the VPLS pseudowire. Future frames destined for CE2's MAC will be forwarded directly to the pseudowire without flooding — exactly how an Ethernet switch learns and optimizes forwarding.
