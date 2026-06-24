# Session 7 — Verification Checklist

## family mpls on Provider Interfaces

- [ ] `PE1> show interfaces ge-0/0/0 detail | match mpls` — shows `Protocol mpls`
- [ ] `P1> show interfaces ge-0/0/0 detail | match mpls` — shows `Protocol mpls`
- [ ] `P1> show interfaces ge-0/0/1 detail | match mpls` — shows `Protocol mpls`
- [ ] `P2> show interfaces ge-0/0/0 detail | match mpls` — shows `Protocol mpls`
- [ ] `P2> show interfaces ge-0/0/1 detail | match mpls` — shows `Protocol mpls`
- [ ] `PE2> show interfaces ge-0/0/0 detail | match mpls` — shows `Protocol mpls`

## LDP Sessions

- [ ] `PE1> show ldp session` — `10.0.0.2` shows `Operational`
- [ ] `P1> show ldp session` — both `10.0.0.1` and `10.0.0.3` show `Operational`
- [ ] `P2> show ldp session` — both `10.0.0.2` and `10.0.0.4` show `Operational`
- [ ] `PE2> show ldp session` — `10.0.0.3` shows `Operational`

## LDP Neighbors

- [ ] `PE1> show ldp neighbor` — one neighbor, label space ID `10.0.0.2:0`
- [ ] `P1> show ldp neighbor` — two neighbors, `10.0.0.1:0` and `10.0.0.3:0`
- [ ] `P2> show ldp neighbor` — two neighbors, `10.0.0.2:0` and `10.0.0.4:0`
- [ ] `PE2> show ldp neighbor` — one neighbor, label space ID `10.0.0.3:0`

## LDP Label Database

- [ ] `PE1> show ldp database` — Input labels for 10.0.0.4/32 received from P1 (a label number, not 3)
- [ ] `PE1> show ldp database` — Label `3` (implicit-null) for 10.0.0.1/32 in Input database (PHP from P1)

## inet.3 — MPLS Routing Table

- [ ] `PE1> show route table inet.3` — shows 3 destinations (10.0.0.2, 10.0.0.3, 10.0.0.4)
- [ ] `PE1> show route table inet.3` — 10.0.0.4/32 shows `Push <label>` action
- [ ] `PE2> show route table inet.3` — shows 3 destinations (10.0.0.1, 10.0.0.2, 10.0.0.3)
- [ ] `PE2> show route table inet.3` — 10.0.0.1/32 shows `Push <label>` action

## BGP Next-Hop Resolution via MPLS

- [ ] `PE1> show route 10.0.0.12 detail` — shows `Label-stack { <label> }; MPLS-label: Push <label>`
- [ ] `PE2> show route 10.0.0.11 detail` — shows `Label-stack { <label> }; MPLS-label: Push <label>`

## End-to-End CE Reachability

- [ ] `CE1> ping 10.0.0.12 count 5` — 5/5 replies, 0% packet loss
- [ ] `CE2> ping 10.0.0.11 count 5` — 5/5 replies, 0% packet loss

## MPLS Forwarding Table (P Routers)

- [ ] `P1> show route table mpls.0` — shows Swap entries (not just Receive) for labels from PE1
- [ ] `P2> show route table mpls.0` — shows Swap entries for labels from P1

## RSVP-TE (Part 3)

- [ ] `PE1> show rsvp neighbor` — shows P1 (10.1.12.2) as RSVP neighbor
- [ ] `P1> show rsvp neighbor` — shows both PE1 (10.1.12.1) and P2 (10.1.23.2)
- [ ] `PE1> show rsvp session` — one Ingress session to 10.0.0.4, State: Up
- [ ] `PE2> show rsvp session` — one Egress session from 10.0.0.1, State: Up
- [ ] `P1> show mpls lsp` — one Transit LSP for PE1-TO-PE2, State: Up
- [ ] `PE1> show mpls lsp` — Ingress LSP PE1-TO-PE2 shows Up

## Quick Commands

```junos
show ldp session
```
Expected: all sessions in `Operational` state.

```junos
show route table inet.3
```
Expected on PE1: three LDP routes with Push labels for 10.0.0.2, 10.0.0.3, 10.0.0.4.

```junos
show rsvp session
```
Expected on PE1: Ingress session to 10.0.0.4 in Up state.

```junos
show mpls lsp
```
Expected on PE1: PE1-TO-PE2 shows Up. Expected on P1/P2: Transit LSP for PE1-TO-PE2 shows Up.
