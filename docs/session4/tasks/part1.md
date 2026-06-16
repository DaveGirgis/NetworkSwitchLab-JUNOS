# Part 1 — Interface Addressing

Configure all interfaces according to the [Addressing Table](../addressing.md). Complete all four routers before moving to Part 2.

## PE1

```junos
PE1> configure

set interfaces ge-0/0/0 unit 0 family inet address 10.1.12.1/30
set interfaces lo0 unit 0 family inet address 10.0.0.1/32
set routing-options router-id 10.0.0.1

commit
```

## P1

```junos
P1> configure

set interfaces ge-0/0/0 unit 0 family inet address 10.1.12.2/30
set interfaces ge-0/0/1 unit 0 family inet address 10.1.23.1/30
set interfaces lo0 unit 0 family inet address 10.0.0.2/32
set routing-options router-id 10.0.0.2

commit
```

## P2

```junos
P2> configure

set interfaces ge-0/0/0 unit 0 family inet address 10.1.23.2/30
set interfaces ge-0/0/1 unit 0 family inet address 10.1.34.1/30
set interfaces lo0 unit 0 family inet address 10.0.0.3/32
set routing-options router-id 10.0.0.3

commit
```

## PE2

```junos
PE2> configure

set interfaces ge-0/0/0 unit 0 family inet address 10.1.34.2/30
set interfaces lo0 unit 0 family inet address 10.0.0.4/32
set routing-options router-id 10.0.0.4

commit
```

## Verify Interface Addressing

Run `show interfaces terse` on each router and confirm all expected interfaces show `up up`:

```junos
PE1> show interfaces terse
Interface               Admin Link Proto    Local                 Remote
ge-0/0/0                up    up
ge-0/0/0.0              up    up   inet     10.1.12.1/30
lo0                     up    up
lo0.0                   up    up   inet     10.0.0.1/32
                                            127.0.0.1/8
```

```junos
P1> show interfaces terse
Interface               Admin Link Proto    Local                 Remote
ge-0/0/0                up    up
ge-0/0/0.0              up    up   inet     10.1.12.2/30
ge-0/0/1                up    up
ge-0/0/1.0              up    up   inet     10.1.23.1/30
lo0                     up    up
lo0.0                   up    up   inet     10.0.0.2/32
                                            127.0.0.1/8
```

## Verify Direct Connectivity

```junos
PE1> ping 10.1.12.2 count 3
```
Expected: 3/3 replies from P1's ge-0/0/0

```junos
P1> ping 10.1.23.2 count 3
```
Expected: 3/3 replies from P2's ge-0/0/0

```junos
P2> ping 10.1.34.2 count 3
```
Expected: 3/3 replies from PE2's ge-0/0/0

!!! warning "Verify all links before configuring OSPF"
    OSPF will not form adjacencies across a link that is not layer-3 reachable. Fix all interface/link issues now — troubleshooting OSPF is significantly harder when interface problems are also present.
