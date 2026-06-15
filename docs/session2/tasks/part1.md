# Part 1 — Physical Interface Addressing

Junos interface configuration lives at `[edit interfaces]`. Each physical interface has one or more **units** (logical sub-interfaces). For simple IPv4 routing, you use `unit 0` with `family inet`.

!!! note "Interfaces are up by default"
    Unlike Cisco IOS — where interfaces start `shutdown` — Junos interfaces are enabled by default. You do not need a `no shutdown` equivalent. If an interface is admin-down, you will see `disable` in its config; remove it with `delete interfaces ge-0/0/X disable`.

## R1 — Configure ge-0/0/0

```junos
R1> configure

[edit]
R1# set interfaces ge-0/0/0 unit 0 family inet address 10.1.12.1/30

[edit]
R1# show | compare
[edit interfaces ge-0/0/0]
+   unit 0 {
+       family inet {
+           address 10.1.12.1/30;
+       }
+   }

[edit]
R1# commit
commit complete
```

## R2 — Configure ge-0/0/0

```junos
R2> configure

[edit]
R2# set interfaces ge-0/0/0 unit 0 family inet address 10.1.12.2/30

[edit]
R2# commit
commit complete
```

## Verify Link-Layer Connectivity

From R1, ping R2's interface address:

```junos
R1> ping 10.1.12.2 count 3
PING 10.1.12.2 (10.1.12.2): 56 data bytes
64 bytes from 10.1.12.2: icmp_seq=0 ttl=64 time=1.234 ms
64 bytes from 10.1.12.2: icmp_seq=1 ttl=64 time=0.987 ms
64 bytes from 10.1.12.2: icmp_seq=2 ttl=64 time=1.102 ms
```

!!! warning "If ping fails"
    1. Run `show interfaces ge-0/0/0 terse` on both routers — both should show `up up` in the first two columns (physical state, protocol state)
    2. Check the GNS3 link is green between R1 and R2
    3. Confirm you configured `ge-0/0/0` (not `ge-0/0/1`) on both ends

## Understanding `show interfaces terse`

```junos
R1> show interfaces terse
Interface               Admin Link Proto    Local                 Remote
ge-0/0/0                up    up
ge-0/0/0.0              up    up   inet     10.1.12.1/30
ge-0/0/1                up    down
ge-0/0/2                up    down
ge-0/0/3                up    down
lo0                     up    up
lo0.0                   up    up   inet     127.0.0.1/8
```

- Column 1 (Admin): `up` = not disabled
- Column 2 (Link): `up` = physical carrier detected (link partner present)
- `ge-0/0/0.0` is the logical unit you configured — shows the IP address

`ge-0/0/1` through `ge-0/0/3` are `up/down` because no cable is connected to those ports.
