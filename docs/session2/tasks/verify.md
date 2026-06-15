# Session 2 — Verification Checklist

## Interfaces

- [ ] `R1> show interfaces terse` — `ge-0/0/0.0` shows `up up inet 10.1.12.1/30`
- [ ] `R2> show interfaces terse` — `ge-0/0/0.0` shows `up up inet 10.1.12.2/30`
- [ ] `R1> show interfaces terse` — `lo0.0` shows `up up inet 10.0.0.1/32`
- [ ] `R2> show interfaces terse` — `lo0.0` shows `up up inet 10.0.0.2/32`

## Routing Table

- [ ] `R1> show route` — 4 routes: `10.0.0.1/32` (Direct), `10.0.0.2/32` (Static), `10.1.12.0/30` (Direct), `10.1.12.1/30` (Local)
- [ ] `R2> show route` — 4 routes: `10.0.0.2/32` (Direct), `10.0.0.1/32` (Static), `10.1.12.0/30` (Direct), `10.1.12.2/30` (Local)

## Connectivity

- [ ] `R1> ping 10.1.12.2 count 3` — 3/3 packets received
- [ ] `R2> ping 10.1.12.1 count 3` — 3/3 packets received
- [ ] `R1> ping 10.0.0.2 count 3` — 3/3 packets received (via static route)
- [ ] `R2> ping 10.0.0.1 count 3` — 3/3 packets received (via static route)

## Quick Verification Commands

```junos
show interfaces terse
show route
show route protocol static
show route 10.0.0.2/32 detail
```
