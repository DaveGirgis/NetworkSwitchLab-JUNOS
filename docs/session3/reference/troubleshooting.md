# Session 3 — Troubleshooting

## Bridge Domain Not Appearing After Commit

**Symptom:** `show bridge domain` returns nothing or "no bridge domains found" after configuring.

**Cause:** The vMX is still in `enhanced-ip` mode. Bridge domains only function in `enhanced-ethernet` mode.

**Fix:**
1. Check the current mode: `show chassis network-services`
2. If it shows `Enhanced-IP`, the mode change did not take effect
3. Enter config mode, re-apply `set chassis network-services enhanced-ethernet`, commit, and reboot

---

## Interface Shows `Proto: -` Instead of `Bridge`

**Symptom:** `show interfaces ge-0/0/1 terse` shows `ge-0/0/1.0 up up -` with no protocol.

**Cause:** `family bridge` was not configured on the unit, or the interface was not added to a bridge domain.

**Fix:**
```junos
configure
set interfaces ge-0/0/1 encapsulation ethernet-bridge
set interfaces ge-0/0/1 unit 0 family bridge interface-mode access
set interfaces ge-0/0/1 unit 0 family bridge vlan-id 10
set bridge-domains VLAN10 interface ge-0/0/1.0
commit
```

---

## Trunk Subunits Not Coming Up

**Symptom:** `ge-0/0/0.10` and `ge-0/0/0.11` show `down down` after configuring the trunk.

**Cause / Fix:**

1. Confirm `flexible-vlan-tagging` and `encapsulation flexible-ethernet-services` are set on the **physical** interface (ge-0/0/0), not on a unit
2. Confirm the physical link is up: `show interfaces ge-0/0/0 terse` — the parent line should show `up up`
3. Confirm the GNS3 link is connected on **Adapter 2** (ge-0/0/0) on both SW1 and SW2
4. Confirm both ends have matching VLAN IDs configured on the trunk subunits

---

## PC Ping to Gateway Fails

**Symptom:** PC1 cannot ping 192.168.10.254 after Part 3.

**Cause / Fix:**

1. Confirm IRB interface is up: `show interfaces irb terse` — irb.10 must show `up up inet`
2. Confirm `set bridge-domains VLAN10 routing-interface irb.10` was committed
3. Confirm PC1's access port (ge-0/0/1.0) is in the VLAN10 bridge domain: `show bridge domain VLAN10`
4. Confirm PC1's IP and gateway are set correctly from the VPCS console: `show ip`

---

## Inter-VLAN Ping Fails (PC1 → PC2)

**Symptom:** PC1 can ping its gateway (192.168.10.254) but cannot reach PC2 (192.168.11.1).

**Cause / Fix:**

1. Confirm irb.11 is up and has address 192.168.11.254/24
2. Confirm PC2's gateway is set to 192.168.11.254 (not 192.168.10.254)
3. Confirm SW1 has both irb.10 and irb.11 bound to their bridge domains
4. Check the routing table: `show route` — should show both 192.168.10.0/24 and 192.168.11.0/24

---

## `show bridge mac-table` is Empty

**Symptom:** No MAC addresses appear even after pinging.

**Cause:** In VCP-only mode, software forwarding may not populate the MAC table in all code paths.

**Impact:** Low — routing and bridging may still work. The MAC table is a diagnostic aid, not a requirement for forwarding in this lab.

**Workaround:** Generate traffic (several pings), then check again. If still empty, proceed — verify connectivity directly with pings rather than relying on the MAC table.
