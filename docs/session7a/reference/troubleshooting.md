# Session 7a — Troubleshooting

## L2circuit Shows `NP` (Interface Not Present)

**Symptom:** `show l2circuit connections` shows State `NP` and the legend reads `NP -- interface h/w not present`. The targeted LDP session is `Operational` in `show ldp session`.

**Cause:** The logical unit 0 on the PE's CCC interface was never explicitly configured. Setting `encapsulation ethernet-ccc` on the physical interface is not sufficient — Junos also requires the unit to exist before it can bind the l2circuit to it.

**Fix:** Add unit 0 on both PEs and commit:

```junos
configure
set interfaces ge-0/0/2 unit 0
commit
```

Verify `show interfaces ge-0/0/2 terse` now shows `ge-0/0/2.0` as a listed unit. The l2circuit should transition to `Up` within a few seconds.

---

## L2circuit Stays in `Dn` State

**Symptom:** `show l2circuit connections` shows State `Dn` or `VC-Dn` instead of `Up`.

**Cause and fix — check in this order:**

1. **Targeted LDP session not established**

   The targeted LDP session between PE1 and PE2 loopbacks must be `Operational`:
   ```junos
   PE1> show ldp session
   ```
   If only one session appears (the link LDP to P1), the targeted LDP session to PE2 has not formed. This usually means PE2's `protocols l2circuit` has not been committed yet, or the loopback is not reachable. Confirm PE2 also has `set protocols l2circuit neighbor 10.0.0.1 interface ge-0/0/2.0 virtual-circuit-id 100`.

2. **virtual-circuit-id mismatch**

   Both PEs must configure the same virtual-circuit-id:
   ```junos
   PE1> show configuration protocols l2circuit
   PE2> show configuration protocols l2circuit
   ```
   If PE1 has `virtual-circuit-id 100` and PE2 has `virtual-circuit-id 200`, the session will show `VM -- vlan id mismatch` or stay `Dn`. Correct and commit to the same ID on both PEs.

3. **Wrong neighbor address**

   PE1 must use PE2's loopback (10.0.0.4) as the neighbor, and PE2 must use PE1's loopback (10.0.0.1). Using a physical interface address or typo will prevent the targeted LDP session from matching the L2circuit configuration.

4. **CCC encapsulation missing on one or both PEs**

   If one PE is missing `encapsulation ethernet-ccc`, the pseudowire negotiation will show `EI -- encapsulation invalid`:
   ```junos
   show configuration interfaces ge-0/0/2
   ```
   Must show `encapsulation ethernet-ccc`. If missing, add it and commit.

5. **LDP transport not functioning**

   The targeted LDP session runs over the existing LDP path. If `show route table inet.3` on PE1 does not show a Push label for PE2 loopback (10.0.0.4), LDP transport is broken — fix LDP from Session 7 first.

---

## Targeted LDP Session Shows `Operational` but L2circuit Is Still `Dn`

**Symptom:** `show ldp session` shows two sessions (link LDP + targeted LDP), both `Operational`, but `show l2circuit connections` still shows `Dn`.

**Cause:** The targeted LDP session has established but the VC label exchange failed. This typically means an encapsulation mismatch or virtual-circuit-id mismatch was detected at the L2circuit signaling layer even though the LDP transport session is healthy.

**Fix:**

Check the detail output for the specific failure reason:
```junos
show l2circuit connections detail
```

Look for a status code in the legend at the top of the output. Common codes after `Dn`:
- `MM` — MTU mismatch (unlikely in this lab with identical vMX images)
- `EM` — encapsulation mismatch (one side is not `ethernet-ccc`)
- `CM` — control-word mismatch (one side negotiated a control word, the other did not)
- `VM` — virtual-circuit-id mismatch

Resolve the mismatch on the indicated side and commit.

---

## CE1 Cannot Ping CE2 After L2circuit Shows Up

**Symptom:** `show l2circuit connections` shows State `Up` on both PEs, but `CE1> ping 192.168.1.2` returns 100% packet loss.

**Cause and fix — check in this order:**

1. **CE interfaces not configured**

   Confirm CE1 and CE2 ge-0/0/1 are up with correct addresses:
   ```junos
   CE1> show interfaces ge-0/0/1 terse
   CE2> show interfaces ge-0/0/1 terse
   ```
   Both must show `up up` with addresses 192.168.1.1/24 and 192.168.1.2/24 respectively. If an interface is `down` or missing an address, return to Part 0 and reconfigure.

2. **GNS3 link not connected**

   Even if the pseudowire is up, traffic from CE1 will not reach it if the physical GNS3 link between CE1 and PE1 is not drawn. Stop the topology, verify both new links are connected (PE1 Adapter 4 ↔ CE1 Adapter 3, PE2 Adapter 4 ↔ CE2 Adapter 3), and restart.

3. **ARP not resolving**

   Because L2circuit is a transparent pseudowire, CE1 sends ARP for 192.168.1.2. If CE2's ARP response is not making it back, there may be a unidirectional issue. Test with:
   ```junos
   CE2> ping 192.168.1.1 count 3
   ```
   If CE2 can reach CE1 but not the reverse, the pseudowire is unidirectional — check that the CCC encapsulation and l2circuit configuration are symmetric on both PEs.

---

## VPLS Instance Stays `Dn` After Configuration

**Symptom:** `show vpls connections` shows no connections or shows State `Dn`.

**Cause and fix — check in this order:**

1. **BGP l2vpn family not negotiated**

   Verify the iBGP session is carrying the l2vpn address family:
   ```junos
   PE1> show bgp neighbor 10.0.0.4 | match NLRI
   ```
   If `l2vpn` does not appear, the BGP session did not negotiate the family. Confirm `set protocols bgp group IBGP family l2vpn signaling` is committed on both PE1 and PE2. After committing, the BGP session may need a moment to re-advertise — wait 30 seconds and re-check.

2. **vrf-target mismatch**

   The route target used for export on one PE must match the import target on the other:
   ```junos
   PE1> show configuration routing-instances VPLS-100
   PE2> show configuration routing-instances VPLS-100
   ```
   Both must have `vrf-target target:65001:100` (the same value). If one uses `target:65001:100` and the other uses `target:65001:200`, VPLS routes will not be imported.

3. **no-tunnel-services missing**

   On vMX, VPLS requires `no-tunnel-services` under `protocols vpls`:
   ```junos
   show configuration routing-instances VPLS-100 protocols vpls
   ```
   Must show `no-tunnel-services`. If missing, add it and commit. The VPLS instance will typically remain in a non-functional state without this on vMX.

4. **site-identifier conflict**

   Each PE's VPLS site must have a unique `site-identifier`. If both PEs configure `site-identifier 1`, BGP will accept the routes but the VPLS instance may not build pseudowires correctly. PE1 must use `site-identifier 1` and PE2 must use `site-identifier 2`.

5. **L2circuit config not removed**

   If `protocols l2circuit` and `encapsulation ethernet-ccc` were not deleted before configuring VPLS, ge-0/0/2 may still be in CCC mode and cannot be added to a VPLS instance. Verify:
   ```junos
   show configuration interfaces ge-0/0/2
   show configuration protocols l2circuit
   ```
   Both should return empty (no config). If l2circuit is still present, delete it and commit before re-checking VPLS.

---

## CE-to-CE Ping Works Through L2circuit but Not VPLS

**Symptom:** Part 1 (L2circuit) worked fine. After configuring VPLS in Part 2, CE1 cannot ping CE2.

**Cause:** This is almost always a VPLS misconfiguration rather than a transport issue — the underlying LDP path is still working. Work through the VPLS troubleshooting items above (vrf-target mismatch, no-tunnel-services, site-identifier conflict).

A quick sanity check: confirm the VPLS pseudowire itself is up before testing CE pings:

```junos
PE1> show vpls connections
```

If the connection-site shows `Dn`, fix the VPLS configuration first. If it shows `Up` but CE pings still fail, the issue is at the CE interface layer — check ge-0/0/1 on CE1 and CE2.
