# Session 8 — Troubleshooting

## VRF Table Is Empty After Part 1

**Symptom:** `show route table VPN-A.inet.0` returns no output or "0 destinations".

**Cause and fix — check in this order:**

1. **Interface not assigned to the VRF**

   Confirm `ge-0/0/1.0` appears in the VRF configuration:
   ```junos
   show configuration routing-instances VPN-A
   ```
   Must show `interface ge-0/0/1.0`. If missing, add it and commit.

2. **Interface link down**

   If the CE-facing link is physically down, no direct route will appear:
   ```junos
   show interfaces ge-0/0/1 terse
   ```
   Must show `up up`. If the link is down, check the GNS3 connection between PE and CE.

3. **Route showing in inet.0 instead of VPN-A.inet.0**

   If the interface was assigned to the VRF but `vrf-target` is missing or the VRF commit failed, the interface may still contribute to `inet.0`. Check:
   ```junos
   show route table inet.0 172.16.1.0/30
   ```
   If the route appears here, the VRF assignment did not take effect. Verify `instance-type vrf` is set and `commit` completed without errors.

---

## bgp.l3vpn.0 Has No Routes After Part 2

**Symptom:** `show route table bgp.l3vpn.0` shows 0 destinations after adding `family inet-vpn unicast`.

**Cause:** The CE eBGP sessions inside the VRF are not yet configured (this is expected at the end of Part 2). VPN routes cannot enter `bgp.l3vpn.0` until CE routes are present in the VRF. Proceed to Part 3.

If `bgp.l3vpn.0` is still empty after completing Part 3, check that:

1. The iBGP session has negotiated `inet-vpn`:
   ```junos
   show bgp neighbor 10.0.0.4 | match NLRI
   ```
   Must show `inet-vpn` in "NLRI for this session". If missing, confirm `family inet-vpn unicast` is configured on both PEs.

2. The VRF BGP session to CE is up (see next section).

---

## VRF eBGP Session Not Establishing (CE Session Down)

**Symptom:** `show bgp summary instance VPN-A` shows CE peer in `Active` or `Idle` state.

**Cause and fix:**

1. **CE still configured toward the old global session**

   CE1 is configured to peer with `172.16.1.1` (PE1's interface address). That address has not changed — it is still on PE1's ge-0/0/1.0, now inside the VRF. If CE1's session shows `Active`, the CE may still be trying to connect. Wait up to 60 seconds for the TCP session to be established from the VRF side.

2. **Group not configured inside the VRF**

   Confirm the eBGP group exists under the routing instance (not under global `protocols bgp`):
   ```junos
   show configuration routing-instances VPN-A protocols bgp
   ```
   Must show the group with `neighbor 172.16.1.2` and `peer-as 65100`. If the group is under global `protocols bgp` instead, it is not part of the VRF — delete it there and recreate it under `routing-instances VPN-A`.

3. **as-override missing**

   If CE1 and CE2 share an AS (AS 65100), `as-override` is required on the PE neighbor statement. Without it, CE2 will receive CE1's route with AS 65100 in the path and suppress it as a loop.

---

## VRF Table Missing Remote CE Prefix

**Symptom:** `show route table VPN-A.inet.0` on PE1 shows CE1's loopback (10.0.0.11/32) but not CE2's (10.0.0.12/32).

**Cause and fix — check in this order:**

1. **iBGP inet-vpn not negotiated on PE2**

   PE2 must also have `family inet-vpn unicast` on its IBGP group. If only PE1 has it, VPN routes will not be exchanged:
   ```junos
   PE2> show bgp neighbor 10.0.0.1 | match NLRI
   ```
   Must show `inet-vpn` in "NLRI for this session".

2. **vrf-target mismatch**

   PE1 and PE2 must import and export the same route target. If PE2 was configured with a different RT (e.g. `target:65001:200`), PE1 will not import PE2's routes:
   ```junos
   show configuration routing-instances VPN-A vrf-target
   ```
   Both must show `target:65001:100`. If they differ, correct and commit.

3. **CE2 not advertising its loopback**

   Confirm CE2's eBGP session is up and the ADVERTISE-LOOPBACK policy is exporting the loopback:
   ```junos
   PE2> show route receive-protocol bgp 172.16.2.2
   ```
   Must show `10.0.0.12/32`. If empty, CE2's export policy is not working — check `show configuration protocols bgp` on CE2.

4. **ADVERTISE-VPN policy not applied on PE2**

   Without this policy, PE2 will receive CE2's route in `VPN-A.inet.0` but not advertise it to the iBGP peer. Confirm:
   ```junos
   PE2> show configuration routing-instances VPN-A protocols bgp group EBGP-CE2
   ```
   Must show `export ADVERTISE-VPN`.

---

## CE-to-CE Ping Fails — 100% Loss

**Symptom:** `CE1> ping 10.0.0.12 source 10.0.0.11 count 5` returns 100% packet loss.

**Cause and fix — check in this order:**

1. **No source specified or wrong source**

   Only loopback prefixes are in the VPN. If source is omitted, Junos uses the outgoing interface address (172.16.1.2). PE2 has a route to 172.16.1.2 in the VRF (it's a direct subnet imported from PE1), but CE2 may not have a return route. Always use `source 10.0.0.11`.

2. **CE route missing from VRF table**

   Work through the "VRF Table Missing Remote CE Prefix" steps above. CE2's loopback must appear in PE1's `VPN-A.inet.0` with a two-label stack before the ping can work.

3. **CE not receiving remote loopback**

   CE1 must have a BGP route to `10.0.0.12/32`:
   ```junos
   CE1> show route 10.0.0.12
   ```
   If missing, PE1 is not advertising it to CE1 via the VRF eBGP group. Check that `export ADVERTISE-VPN` is applied on PE1's VRF eBGP group and that `10.0.0.12/32` is present as a BGP route in `VPN-A.inet.0`.

4. **LDP transport broken**

   The outer label path must be intact. Verify:
   ```junos
   PE1> show route table inet.3 10.0.0.4
   ```
   Must show a Push label entry for PE2. If missing, LDP has failed — return to Session 7 to restore it.

---

## Customer Routes Appearing in inet.0 (Route Leaking)

**Symptom:** `show route table inet.0 10.0.0.11` on PE1 shows a BGP route to CE1's loopback.

**Cause:** The global `EBGP-CE1` group from Session 6 was not deleted, so CE1 has two eBGP sessions: one in the VRF and one globally. The global session is installing routes into `inet.0`.

**Fix:**

```junos
PE1> configure
PE1# delete protocols bgp group EBGP-CE1
PE1# commit
```

After deleting the global group, verify `inet.0` no longer contains CE prefixes.
