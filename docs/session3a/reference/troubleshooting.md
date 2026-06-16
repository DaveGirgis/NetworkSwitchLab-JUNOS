# Session 3a — Troubleshooting

## Ping Still Fails After Enabling RSTP

**Symptom:** Pings between PCs fail even after `commit` on both switches.

**Cause / Fix:**

1. RSTP needs a few seconds to converge on first enable — wait 5–10 seconds and retry
2. Confirm RSTP is active: `show spanning-tree bridge` — if this returns empty, RSTP did not enable
3. Confirm the bridge domain has RSTP enabled: `show configuration bridge-domains VLAN10`
4. If the command syntax caused a commit error, refer to the guide corrections and re-apply

---

## `show spanning-tree bridge` Returns Nothing

**Symptom:** The command returns empty output or `no spanning tree instances found`.

**Cause:** RSTP was not successfully enabled, likely due to a syntax issue with `set bridge-domains VLAN10 protocols rstp` on this vMX code version.

**Fix:**

Try enabling RSTP at the global protocols level instead:

```junos
configure
set protocols rstp
commit
```

Then check again. If still empty, the vMX 14.1 VCP-only image may require a different approach — report the exact error to the course maintainer for a guide correction.

---

## Broadcast Storm After Adding Second Trunk

**Symptom:** After adding the second trunk (Part 0), pings hang and the vMX becomes sluggish. `show system processes` shows high CPU.

**Cause:** Expected — this is what a broadcast storm looks like without STP.

**Fix:**

1. From GNS3, right-click the second trunk link and delete or suspend it immediately
2. Enable RSTP on both switches (Part 1) before reconnecting
3. Reconnect the second trunk after RSTP is confirmed active

---

## One Trunk Port Stuck in Discarding After Failover

**Symptom:** After restoring the cut trunk link, the port remains in Discarding and does not return to Forwarding.

**Cause / Fix:**

RSTP should reconverge automatically. If it does not:

1. Check the port is not administratively disabled: `show interfaces ge-0/0/3 terse`
2. Force STP to recompute: disconnect and reconnect the link in GNS3
3. On both switches, confirm RSTP is enabled and the bridge priority is correct: `show spanning-tree bridge`

---

## Access Ports Not Showing as Edge

**Symptom:** `show spanning-tree interface` shows access ports going through Discarding → Learning → Forwarding states on each link flap.

**Cause:** Edge port configuration was not applied.

**Fix:**

```junos
configure
set protocols rstp interface ge-0/0/1 edge
set protocols rstp interface ge-0/0/2 edge
commit
```

Apply on both switches. Edge ports skip the STP state machine entirely and go directly to Forwarding.
