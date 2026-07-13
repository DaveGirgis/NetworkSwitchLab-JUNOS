# Session 9 - Troubleshooting

This page covers the three graded faults in this session plus the most common mistakes students make while working through them. Unlike prior sessions, the "cause" for each graded fault is intentionally not restated here in full - see Parts 1-3 for the complete diagnostic walkthrough. This page is a quick-reference for getting unstuck, not a shortcut to skip the process.

## Graded Fault 1 - IS-IS Adjacency Down on P1-P2

**Symptom:** Both `CE1> ping 10.0.0.12 source 10.0.0.11` and `CE1> ping 192.168.1.2` fail with 100% packet loss. Both services fail together.

**Where to look:** `show isis adjacency` on all four provider routers. The missing adjacency will appear on exactly one link, on both routers that terminate it.

**Common student mistake:** Jumping straight to `show bgp summary` or `show vpls connections` because the ticket mentions "VPN-A is down." Both services failing together is the tell that this is not a VPN-specific issue - always check IS-IS before BGP or VRF configuration when multiple independent services fail identically.

**Second most common mistake:** Seeing `show isis interface <if> detail` show `Adjacencies: 0` on one side and assuming the interface itself is down. Confirm with `show interfaces terse` first - in this fault, the interface is fully up. The problem is the IS-IS `Circuit type` (level) configured on the interface, not the physical link.

See [Part 1](../tasks/part1.md) for the full walkthrough.

---

## Graded Fault 2 - Missing inet-vpn unicast Address Family

**Symptom:** `CE1> ping 10.0.0.12 source 10.0.0.11` fails with 100% packet loss. `CE1> ping 192.168.1.2` (VPLS-100) succeeds normally.

**Where to look:** `show bgp summary` - the session shows `Establ`, but `bgp.l3vpn.0` and the per-peer `VPN-A.inet.0` line are missing. Confirm with `show bgp neighbor <ip> | match NLRI` - `inet-vpn-unicast` will be absent from every NLRI line.

**Common student mistake:** Seeing `Establ` in `show bgp summary` and concluding "BGP is fine" without checking which address families are actually negotiated. An established session is not the same as a session carrying every expected family. Always run the NLRI check whenever a specific VPN service fails but the underlying BGP peer relationship looks healthy.

**Second most common mistake:** Editing PE1's configuration first. Confirm which PE actually lost the `family inet-vpn unicast` statement with `show configuration protocols bgp group IBGP` on both routers before touching either one - in this fault, PE1 was never wrong, and adding the family to PE1 a second time (while harmless) does not fix anything and wastes diagnostic time.

**Third common mistake:** Assuming the fix requires restarting the BGP session (`clear bgp neighbor`). It does not. Adding the missing `family` statement and committing is enough - Junos renegotiates capabilities automatically within about 15-30 seconds without a hard session reset.

See [Part 2](../tasks/part2.md) for the full walkthrough.

---

## Graded Fault 3 - VPN-A Route Target Mismatch on PE2

**Symptom:** `CE1> ping 10.0.0.12 source 10.0.0.11` fails with 100% packet loss. `CE1> ping 192.168.1.2` (VPLS-100) succeeds normally. Symptom is identical to Fault 2 from the CE's point of view.

**Where to look:** `show bgp summary` shows the session `Establ` with `bgp.l3vpn.0` populated correctly (2 paths) - the address family is fine, unlike Fault 2. The tell is a **mismatch between two specific commands**: `show route table bgp.l3vpn.0` shows both PE1's and PE2's VPN-IPv4 routes, but `show route table VPN-A.inet.0` on PE1 is missing the route learned from PE2. A route present in the BGP table but absent from the VRF table is the signature of a route-target import failure.

**Common student mistake:** Concluding "BGP is broken again" and re-checking NLRI families (Fault 2's fix) instead of comparing the BGP table against the VRF table. If NLRI already shows `inet-vpn-unicast` correctly negotiated and `bgp.l3vpn.0` has the expected number of routes, the problem is downstream of BGP negotiation - in the VRF's route-target import policy, not the session itself.

**Second most common mistake:** Fixing the route target on the wrong PE, or fixing it under the wrong routing-instance. VPN-A and VPLS-100 both use `target:65001:100` as a coincidence of this lab's addressing plan (see [addressing.md](../addressing.md)) - always run `show | compare` before committing to confirm the pending change is scoped to `routing-instances VPN-A` and not accidentally touching `routing-instances VPLS-100`.

**Third common mistake:** Assuming `vrf-target` sets only the export policy or only the import policy. It sets both simultaneously to the same value. A mismatch on one PE breaks import in both directions on that PE - it will fail to import the other PE's routes, and the other PE will fail to import this PE's routes, even though this PE's own export is technically "working" in the sense that the route reaches the remote `bgp.l3vpn.0` table.

See [Part 3](../tasks/part3.md) for the full walkthrough.

---

## General Mistakes Across All Three Faults

### Skipping the Baseline in Part 0

**Symptom:** Confusing or contradictory results while diagnosing Fault 1, 2, or 3 that do not match this session's expected output.

**Cause:** Part 0 was skipped or its health check commands were not actually run and compared. If the network was not fully healthy before the first fault was injected, later diagnostic steps compare against a baseline that was never actually correct.

**Fix:** Return to [Part 0](../tasks/part0.md) and work through every baseline check. Do not proceed into Part 1 until every command in Part 0's Step 2 matches its expected output exactly.

### Testing Only One Service

**Symptom:** Concluding a fault is fixed after testing only `CE1> ping 10.0.0.12 source 10.0.0.11`, without also testing `ping 192.168.1.2`.

**Cause:** Some students stop testing as soon as the specific symptom in the ticket resolves. This misses two things: (1) confirming the fix didn't collaterally break the other service, and (2) building the habit of using VPLS-100 as a permanent control group, which is the fastest way to scope a future fault to "VPN-A only" versus "shared infrastructure" in under 10 seconds.

**Fix:** After every fix in this session, test both `ping 10.0.0.12 source 10.0.0.11` (VPN-A) and `ping 192.168.1.2` (VPLS-100), in both directions, before moving to the next part.

### Editing Configuration Before Forming a Hypothesis

**Symptom:** Multiple unrelated `set` and `delete` commands committed in sequence, "trying things" until the symptom resolves - but without being able to explain afterward exactly which change fixed it.

**Cause:** Troubleshooting under pressure often turns into trial-and-error editing rather than diagnosis. This can accidentally fix the graded fault while leaving an unrelated new misconfiguration in place (or accidentally introduce one), which then surfaces as a confusing new symptom in the next part.

**Fix:** Before opening configuration mode, write down (even just in your own notes) the specific command output that led you to suspect a specific line of configuration on a specific router. Make exactly one change, commit, and verify the specific show command output changed as expected before moving on.

### Forgetting `source` on VPN-A Pings

**Symptom:** `CE1> ping 10.0.0.12` (without a source) shows 100% packet loss even though every layer is actually healthy.

**Cause:** This is not a fault - it is the same behavior documented since Session 7. Only loopback prefixes are exported into VPN-A. Without `source 10.0.0.11`, Junos uses the outgoing interface address (172.16.1.2), which is not the prefix CE2 has a return route for.

**Fix:** Always test VPN-A reachability with `ping 10.0.0.12 source 10.0.0.11` (from CE1) and `ping 10.0.0.11 source 10.0.0.12` (from CE2). If you see 100% loss and every show command from Parts 1-3 looks healthy, check your ping syntax before assuming a fourth, ungraded fault exists.
