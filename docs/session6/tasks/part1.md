# Part 1 — eBGP PE-CE

## Overview

eBGP sessions are configured between each PE router and its directly connected CE router. The peers use the physical link addresses (172.16.x.x) since eBGP by default only travels one IP hop (TTL=1).

In Junos, BGP configuration lives under `[edit protocols bgp]`. Peers are grouped — a **group** holds shared configuration (type, peer-AS, policy) and individual neighbors inherit it.

## Step 1: Configure AS Numbers

Each router must have its ASN defined before BGP can start.

### PE1 and PE2 (AS 65001)

```junos
PE1> configure

set routing-options autonomous-system 65001

commit
```

```junos
PE2> configure

set routing-options autonomous-system 65001

commit
```

### CE1 and CE2 (AS 65100)

```junos
CE1> configure

set routing-options autonomous-system 65100

commit
```

```junos
CE2> configure

set routing-options autonomous-system 65100

commit
```

## Step 2: Configure eBGP on PE1 and CE1

### PE1

```junos
configure

set protocols bgp group EBGP-CE1 type external
set protocols bgp group EBGP-CE1 peer-as 65100
set protocols bgp group EBGP-CE1 neighbor 172.16.1.2

commit
```

### CE1

```junos
configure

set protocols bgp group EBGP-PE1 type external
set protocols bgp group EBGP-PE1 peer-as 65001
set protocols bgp group EBGP-PE1 neighbor 172.16.1.1

commit
```

## Step 3: Configure eBGP on PE2 and CE2

### PE2

```junos
configure

set protocols bgp group EBGP-CE2 type external
set protocols bgp group EBGP-CE2 peer-as 65100
set protocols bgp group EBGP-CE2 neighbor 172.16.2.2

commit
```

### CE2

```junos
configure

set protocols bgp group EBGP-PE2 type external
set protocols bgp group EBGP-PE2 peer-as 65001
set protocols bgp group EBGP-PE2 neighbor 172.16.2.1

commit
```

## Step 4: Verify eBGP Sessions

Wait ~30 seconds, then check the session state:

```junos
PE1> show bgp summary
```

Expected — one eBGP peer up, no prefixes yet:

```text
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0
                       0          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.16.1.2            65100          5          5       0       0         :42 0/0/0/0              0/0/0/0
```

The peer line shows `0/0/0/0  0/0/0/0` — two count groups for inet.0 and iso.0 respectively. A session that has not formed yet shows `Active` or `Connect` instead of counts. The `0/0/0/0` with no error text confirms the session is established. Prefix counts are zero because no export policy is configured on CE1 yet.

## Step 5: Advertise CE Loopbacks into BGP

By default, BGP does not advertise any routes. An **export policy** must be configured to select which routes to send.

We will advertise each CE's loopback into BGP using a routing policy that matches the directly connected loopback prefix.

### CE1 — advertise 10.0.0.11/32

```junos
configure

set policy-options policy-statement ADVERTISE-LOOPBACK term 1 from protocol direct
set policy-options policy-statement ADVERTISE-LOOPBACK term 1 from route-filter 10.0.0.11/32 exact
set policy-options policy-statement ADVERTISE-LOOPBACK term 1 then accept
set protocols bgp group EBGP-PE1 export ADVERTISE-LOOPBACK

commit
```

### CE2 — advertise 10.0.0.12/32

```junos
configure

set policy-options policy-statement ADVERTISE-LOOPBACK term 1 from protocol direct
set policy-options policy-statement ADVERTISE-LOOPBACK term 1 from route-filter 10.0.0.12/32 exact
set policy-options policy-statement ADVERTISE-LOOPBACK term 1 then accept
set protocols bgp group EBGP-PE2 export ADVERTISE-LOOPBACK

commit
```

## Step 6: Verify Prefix Receipt on PE Routers

```junos
PE1> show route receive-protocol bgp 172.16.1.2
```

Expected — CE1's loopback received:

```text
inet.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.0.0.11/32            172.16.1.2                              65100 I

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
```

The `iso.0` table is normal — it reflects the IS-IS NET address carried by `family iso` on the PE interfaces.

```junos
PE2> show route receive-protocol bgp 172.16.2.2
```

Expected — CE2's loopback received:

```text
inet.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.0.0.12/32            172.16.2.2                              65100 I

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
```

At this point PE1 knows CE1's prefix and PE2 knows CE2's prefix. The two PE routers have not yet shared this information — that is the job of iBGP in Part 2.

!!! note "BGP route preference 170"
    In Junos all BGP routes carry a preference of 170, lower than IS-IS (18) and OSPF (10). BGP routes will only be installed in inet.0 if no better-preference route to the same prefix exists.
