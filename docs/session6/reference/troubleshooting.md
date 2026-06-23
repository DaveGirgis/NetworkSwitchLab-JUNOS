# Session 6 — Troubleshooting

## BGP Session Stuck in `Active` State

**Symptom:** `show bgp summary` shows `Active` rather than `Establ` for a peer.

**Cause:** The router is trying to initiate a TCP connection but is not succeeding. `Active` means no session is established.

**Fix:**

1. Confirm the peer IP address is reachable:
   ```junos
   ping 172.16.1.2 count 3
   ```
   If this fails, fix the interface/link before proceeding.

2. Confirm the correct peer AS is configured on both sides — a mismatch causes BGP NOTIFICATION and session teardown:
   ```junos
   show configuration protocols bgp
   ```

3. Confirm the local AS is set:
   ```junos
   show configuration routing-options autonomous-system
   ```

4. For iBGP: confirm IS-IS has reachability to the peer loopback:
   ```junos
   ping 10.0.0.4 count 3
   ```
   iBGP peering uses loopback addresses — if IS-IS is not running or the loopback route is missing, the session cannot form.

---

## BGP Session Establishes but No Prefixes Received

**Symptom:** `show bgp summary` shows `Establ` but `0/0/0/0` prefix counts.

**Cause:** No export policy is configured on the advertising router, or the policy does not match any routes.

**Fix:**

On the CE router, confirm the export policy is applied to the BGP group:
```junos
show configuration protocols bgp group EBGP-PE1
```

Should show `export ADVERTISE-LOOPBACK`. If not, apply it:
```junos
set protocols bgp group EBGP-PE1 export ADVERTISE-LOOPBACK
commit
```

Confirm the policy matches the loopback:
```junos
show configuration policy-options policy-statement ADVERTISE-LOOPBACK
```

The route-filter should match the exact loopback prefix. Test the policy:
```junos
test policy ADVERTISE-LOOPBACK 10.0.0.11/32
```

---

## Prefix Received on PE but Not Propagated via iBGP

**Symptom:** PE1 has CE1's prefix via eBGP but PE2 does not see it via iBGP.

**Cause:** The iBGP session is not established, or there is no export policy on PE1 for iBGP (BGP requires explicit export policy in Junos to re-advertise routes).

**Fix:**

In Junos, BGP by default **does** re-advertise eBGP-learned routes to iBGP peers without an explicit export policy (unlike IS-IS or OSPF redistribution). If PE2 is not receiving the prefix:

1. Confirm the iBGP session is `Establ`:
   ```junos
   show bgp summary
   ```

2. Check what PE1 is advertising to PE2:
   ```junos
   show bgp neighbor 10.0.0.4 advertised-routes
   ```

3. If the route is not in the advertised list, check that the iBGP session has the same local AS on both sides.

---

## iBGP Route Received but Not Active (Hidden)

**Symptom:** `show route receive-protocol bgp 10.0.0.4` shows the prefix but `show route protocol bgp` does not show it as active.

**Cause:** The NEXT_HOP is unreachable — `next-hop-self` is missing or not working.

**Fix:**

Confirm `next-hop-self` is configured on the iBGP neighbor:
```junos
show configuration protocols bgp group IBGP
```

Should show `neighbor 10.0.0.4 { next-hop-self; }`.

Check what next-hop value is being received:
```junos
show route receive-protocol bgp 10.0.0.4 detail
```

Look for the `Next hop` line — if it shows a 172.16.x.x address, `next-hop-self` is not being applied on the sending side. Configure it on PE2 for its neighbor 10.0.0.1.

---

## CE-to-CE Ping Fails

**Symptom:** CE1 can reach PE1 but cannot ping CE2's loopback (10.0.0.12).

**This is expected behavior at the end of Session 6.** CE1 has a BGP route to 10.0.0.12, but the transit P routers (P1, P2) do not have a route to that prefix — they only carry IS-IS routes. The packet is forwarded from PE1 toward PE2 via IS-IS, reaches P1, and is dropped.

**Resolution:** Session 7 (MPLS + LDP) adds label switching through P routers, removing the need for P routers to carry BGP prefixes in their routing table.
