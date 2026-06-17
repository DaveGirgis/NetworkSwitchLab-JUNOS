# Session 4 — Troubleshooting

## OSPF Neighbor Stuck in `Init` State

**Symptom:** `show ospf neighbor` shows state `Init` indefinitely.

**Cause:** The remote router is sending Hellos but is not including this router's IP in its Hello packet neighbor list — usually because it has not yet received a Hello from this side.

**Fix:**
1. Confirm both sides have OSPF configured on the same interface with the same area
2. Check for a firewall filter on the interface blocking UDP/OSPF — `show firewall`
3. Confirm the interface is not passive on the remote side
4. Run `show ospf interface ge-0/0/0.0` — interface should show area `0.0.0.0`, type `P2P`

---

## OSPF Neighbor Stuck in `ExStart` / `Exchange`

**Symptom:** State cycles between `ExStart` and `Exchange` and never reaches `Full`.

**Cause:** MTU mismatch between neighbors. vMX-14.1 defaults to 1500 MTU, but if you have mismatched settings, OSPF DD packet exchange fails.

**Fix:** Verify MTU on both sides:

```junos
show interfaces ge-0/0/0 detail | match MTU
```

Both should show `1500`. As a workaround in the lab:

```junos
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 ignore-mtu-mismatch
```

---

## OSPF Forms Adjacency but Routes Are Missing

**Symptom:** `show ospf neighbor` shows `Full` but `show route protocol ospf` is missing some prefixes.

**Cause / Fix:**
1. Check `show ospf database` — does the LSDB contain all four Router LSAs?
2. If an LSA is missing, the router that should have generated it may not have OSPF enabled — verify with `show ospf interface` on that router
3. Loopback not advertised: confirm `lo0.0` is in OSPF with `passive` — not just missing from OSPF entirely

---

## OSPF Adjacency Flaps

**Symptom:** Neighbor repeatedly drops and re-forms.

**Cause / Fix:**
1. Run `show ospf statistics` — look for `Dead timer` expiry events
2. Timer mismatch: both sides must have identical Hello and Dead intervals
3. CPU overload on vMX-14.1: if the GNS3 VM is overloaded, Hello packets may be delayed. Reduce to two nodes at a time if needed.

---

## `show ospf neighbor` Returns Empty

**Symptom:** No neighbors listed at all.

**Fix:**

```junos
show protocols ospf
```

If OSPF is not configured, the output will be empty or show no interfaces. Verify your configuration:

```junos
show configuration protocols ospf
```

This should show the area and interface stanzas. If empty, OSPF was not committed correctly.
