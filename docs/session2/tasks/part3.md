# Part 3 — Default Routes & Route Summarization

## Default Routes

A default route (`0.0.0.0/0`) matches any destination not in the routing table. In a service provider network, default routes are typically used on customer edge (CE) routers pointing toward the provider.

### R2 — Add a Default Route Pointing to R1

```junos
R2> configure

[edit]
R2# set routing-options static route 0.0.0.0/0 next-hop 10.1.12.1

[edit]
R2# commit
commit complete
```

Verify it appears in the routing table:

```junos
R2> show route 0.0.0.0/0

inet.0: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0.0.0.0/0          *[Static/5] 00:00:15
                    > to 10.1.12.1 via ge-0/0/0.0
```

Remove the default route when done — in production SP networks, default routes are distributed via BGP, not configured statically on transit routers:

```junos
R2> configure

[edit]
R2# delete routing-options static route 0.0.0.0/0

[edit]
R2# commit
commit complete
```

## Qualified Next-Hops (Floating Static Routes)

Junos supports **floating static routes** — a backup route with a higher preference that activates only when the primary route disappears.

```junos
R1> configure

[edit]
R1# set routing-options static route 10.0.0.2/32 next-hop 10.1.12.2
R1# set routing-options static route 10.0.0.2/32 qualified-next-hop 10.1.12.2 preference 10

[edit]
R1# commit
commit complete
```

!!! note "Floating static routes in Junos"
    The `qualified-next-hop` with a higher preference value serves as the backup. If the primary next-hop becomes unreachable, the qualified next-hop activates. This pattern is less common once dynamic routing protocols are in place.

## Examine the Full Routing Table

```junos
R1> show route detail
```

The `detail` flag shows next-hop type, preference, metric, and age for each route. Use `show route <prefix> detail` to inspect a specific prefix.

```junos
R1> show route 10.0.0.2/32 detail

inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
10.0.0.2/32 (1 entry, 1 announced)
        *Static Preference: 5
                Next hop: 10.1.12.2 via ge-0/0/0.0, selected
                State: <Active Int>
                Age: 2:34
                Task: RT
                Announcement bits (1): 0-KRT
                AS path: I
```

## Useful Route Filtering

| Command | What it shows |
|---------|---------------|
| `show route protocol static` | Only static routes |
| `show route protocol direct` | Only directly connected routes |
| `show route 10.0.0.0/8 longer` | All routes within 10.0.0.0/8 |
| `show route 10.0.0.2/32 exact` | Only the exact prefix |
| `show route terse` | Compact one-line format per route |
