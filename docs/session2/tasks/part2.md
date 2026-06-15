# Part 2 — Loopbacks & Static Routes

## Configure Loopback Interfaces

Loopbacks in Junos use `lo0` (the only loopback device). Each router has a single `lo0` with one or more units.

!!! tip "Why /32 on the loopback?"
    A /32 means "this address belongs only to this router." Routing protocols advertise loopbacks as /32 host routes, making them reachable from anywhere in the network. They do not depend on any physical link being up.

### R1 — Loopback

```junos
R1> configure

[edit]
R1# set interfaces lo0 unit 0 family inet address 10.0.0.1/32

[edit]
R1# commit
commit complete
```

### R2 — Loopback

```junos
R2> configure

[edit]
R2# set interfaces lo0 unit 0 family inet address 10.0.0.2/32

[edit]
R2# commit
commit complete
```

Verify loopbacks appear in `show interfaces terse`:

```junos
R1> show interfaces lo0 terse
Interface               Admin Link Proto    Local                 Remote
lo0                     up    up
lo0.0                   up    up   inet     10.0.0.1/32
                                            127.0.0.1/8
```

## Configure Static Routes

Static routes in Junos live under `[edit routing-options static]`. The syntax is:

```
set routing-options static route <destination> next-hop <gateway>
```

### R1 — Static Route to R2 Loopback

```junos
R1> configure

[edit]
R1# set routing-options static route 10.0.0.2/32 next-hop 10.1.12.2

[edit]
R1# show | compare
[edit routing-options]
+  static {
+      route 10.0.0.2/32 next-hop 10.1.12.2;
+  }

[edit]
R1# commit
commit complete
```

### R2 — Static Route to R1 Loopback

```junos
R2> configure

[edit]
R2# set routing-options static route 10.0.0.1/32 next-hop 10.1.12.1

[edit]
R2# commit
commit complete
```

## Verify the Routing Table

```junos
R1> show route

inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.1/32        *[Direct/0] 00:03:21
                    > via lo0.0
10.0.0.2/32        *[Static/5] 00:00:42
                    > to 10.1.12.2 via ge-0/0/0.0
10.1.12.0/30       *[Direct/0] 00:05:11
                    > via ge-0/0/0.0
10.1.12.1/30       *[Local/0] 00:05:11
                      Local via ge-0/0/0.0
```

**Route types:**
- `Direct/0` — directly connected network (preference 0)
- `Local/0` — local IP address on an interface
- `Static/5` — static route (preference 5)

!!! note "Junos route preference"
    Junos uses **preference** instead of Cisco's administrative distance. Lower = preferred. Static routes have preference 5 (vs Cisco's 1). OSPF internal = 10, BGP = 170.

## Verify Loopback-to-Loopback Ping

```junos
R1> ping 10.0.0.2 count 3
PING 10.0.0.2 (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: icmp_seq=0 ttl=64 time=1.345 ms
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=1.123 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.998 ms
```

This confirms the static route is installed and traffic can reach R2's loopback.
