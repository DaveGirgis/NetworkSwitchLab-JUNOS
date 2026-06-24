# Session 7 — Troubleshooting

## LDP Session Not Forming

**Symptom:** `show ldp neighbor` returns empty. `show ldp session` shows no sessions.

**Cause:** One of several issues: `family mpls` missing from the interface, LDP not enabled on the interface, IS-IS adjacency down (LDP uses IS-IS routes to reach the neighbor's loopback for the TCP session).

**Fix:**

1. Confirm `family mpls` is on the provider interface:
   ```junos
   show interfaces ge-0/0/0 detail | match family
   ```
   Must show `Protocol mpls`. If missing: `set interfaces ge-0/0/0 unit 0 family mpls`, then `commit`.

2. Confirm LDP is enabled on the interface:
   ```junos
   show configuration protocols ldp
   ```
   Must include `interface ge-0/0/0.0`. If missing, add it and commit.

3. Confirm IS-IS adjacency is up (LDP TCP sessions use loopback addresses, which are reachable via IS-IS):
   ```junos
   show isis adjacency
   ```
   If IS-IS is down, LDP cannot form — fix IS-IS first.

4. Confirm `lo0.0` is in the LDP interface list:
   ```junos
   show configuration protocols ldp interface
   ```
   If `lo0.0` is missing, add it (`set protocols ldp interface lo0.0`, commit). Without it, LDP may try to use a physical interface IP as the transport address, causing session failures if the physical link goes down.

---

## LDP Session Is `Operational` but inet.3 Has No Entries

**Symptom:** `show ldp session` shows `Operational` but `show route table inet.3` is empty.

**Cause:** LDP has formed but no label bindings have been exchanged, or the routes in inet.0 are not being distributed by LDP. This is unusual but can occur if the LDP configuration is incomplete.

**Fix:**

1. Confirm the LDP database has entries:
   ```junos
   show ldp database
   ```
   If the database is empty, LDP is not distributing labels. Check that IS-IS routes exist in inet.0:
   ```junos
   show route table inet.0 protocol isis
   ```
   LDP distributes labels for routes in inet.0. If inet.0 has no IS-IS routes, fix IS-IS first.

2. Restart LDP if the database remains empty despite correct configuration:
   ```junos
   restart ldp
   ```

---

## BGP Route Has No Label Stack (inet.3 Exists but BGP Ignores It)

**Symptom:** `show route table inet.3` shows LDP routes but `show route 10.0.0.12 detail` does not show a label-stack.

**Cause:** BGP uses inet.3 for next-hop resolution only when the BGP next-hop is reachable there. If the iBGP session is not using loopback addresses (missing `local-address`), or the LDP entry for the next-hop does not exist, BGP falls back to inet.0.

**Fix:**

1. Confirm the iBGP session uses loopback addresses:
   ```junos
   show configuration protocols bgp group IBGP
   ```
   Must show `local-address 10.0.0.1` (on PE1). If missing, add it and restart the BGP session.

2. Confirm inet.3 has an entry for the BGP next-hop (10.0.0.4 for CE2's route on PE1):
   ```junos
   show route table inet.3 10.0.0.4
   ```
   If 10.0.0.4 is missing, LDP has not distributed a label for PE2's loopback. Check LDP database and IS-IS convergence.

3. Confirm the iBGP neighbor address matches the LDP transport address:
   ```junos
   show bgp neighbor 10.0.0.4
   ```
   The neighbor must be the loopback address. If the iBGP session was configured with a physical address instead of the loopback, LDP will not be able to match the next-hop to an inet.3 entry.

---

## CE-to-CE Ping Still Failing After MPLS Is Up

**Symptom:** `show route table inet.3` has LDP entries, BGP routes show a label-stack, but `CE1> ping 10.0.0.12` fails.

**Cause and fix — check in this order:**

1. **Confirm BGP is still up** — MPLS configuration does not affect BGP, but confirm sessions did not drop during the changes:
   ```junos
   PE1> show bgp summary
   ```
   Both CE1 and PE2 sessions must be `Establ`. If BGP dropped, wait for sessions to re-establish.

2. **Confirm CE1 has the BGP route**:
   ```junos
   CE1> show route 10.0.0.12
   ```
   Must show a BGP route via 172.16.1.1 (PE1). If missing, check that the BGP export policy on CE1 is still applied and the iBGP session between PE1 and PE2 is carrying the prefix.

3. **Confirm PE2 has a route back to CE1's loopback**:
   ```junos
   PE2> show route 10.0.0.11
   ```
   The return path requires PE2 to have CE1's loopback via iBGP, and it must also be MPLS-resolved. Check `show route 10.0.0.11 detail` on PE2 for a label-stack.

4. **Check family mpls on all provider interfaces** — a missing `family mpls` on any provider interface causes labeled packets to be dropped at that hop. A traceroute from CE1 can reveal where packets stop:
   ```junos
   CE1> traceroute 10.0.0.12
   ```
   If traceroute shows a hop then stops, the next router is dropping the labeled packet. Confirm `family mpls` on the interface that connects to the dropping router.

---

## RSVP LSP Stuck in `Dn` (Down) State

**Symptom:** `show rsvp session` on PE1 shows State: `Dn`. `show mpls lsp` shows `Dn`.

**Cause:** The RSVP PATH message could not be delivered end-to-end. Common reasons:

- RSVP not enabled on a transit interface (PATH message dropped)
- IS-IS TE extensions not enabled — CSPF cannot compute a path
- The egress loopback (10.0.0.4) is not reachable via IS-IS

**Fix:**

1. Confirm RSVP is enabled on all provider interfaces on all four routers:
   ```junos
   show rsvp interface
   ```
   On P1, must show both ge-0/0/0.0 and ge-0/0/1.0. If an interface is missing, add it (`set protocols rsvp interface ge-0/0/x.0`, commit).

2. Confirm IS-IS TE extensions are active on all four routers:
   ```junos
   show isis database detail | match TE
   ```
   If no TE information appears, run `set protocols isis traffic-engineering` on the missing router and commit.

3. Confirm IS-IS reachability to PE2's loopback from PE1:
   ```junos
   ping 10.0.0.4 count 3
   ```
   If this fails, fix IS-IS first — RSVP cannot establish an LSP to an unreachable destination.

4. Check RSVP error messages in detail:
   ```junos
   show rsvp session detail
   ```
   Look for an error code in the PathErr or ResvErr fields — this identifies which router rejected the PATH message and why.

---

## RSVP Session Up but LSP Not Used for BGP

**Symptom:** `show mpls lsp` shows the LSP as Up but `show route 10.0.0.12 detail` on PE1 still shows the LDP label (not the RSVP label).

**Cause:** This is expected behavior. BGP uses the best route in inet.3. Without additional configuration, Junos prefers LDP routes over RSVP LSPs for BGP next-hop resolution.

**Fix:** To steer BGP traffic over the RSVP LSP instead of the LDP path, you would configure `traffic-engineering bgp-igp` under `protocols mpls`, which replaces LDP next-hops in inet.3 with RSVP LSPs for matching destinations. This is beyond the scope of this intro session — the RSVP LSP is verified independently via `show rsvp session` and `show mpls lsp`.
