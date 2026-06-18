# Session 5 — Troubleshooting

## `show isis adjacency` Returns Empty

**Symptom:** No neighbors listed after committing IS-IS configuration.

**Cause / Fix:**

1. Confirm `family iso` is on the interface:
   ```junos
   show interfaces ge-0/0/0 detail | match iso
   ```
   If no output, add `set interfaces ge-0/0/0 unit 0 family iso` and commit.

2. Confirm IS-IS is committed on the remote router — both sides must have IS-IS configured for an adjacency to form.

3. Confirm the interface is added to IS-IS:
   ```junos
   show configuration protocols isis
   ```
   The interface stanza must be present.

---

## IS-IS Adjacency Stuck in `Initializing`

**Symptom:** `show isis adjacency` shows state `Initializing` rather than `Up`.

**Cause:** IS-IS Hellos are being received from the neighbor but the adjacency has not fully formed — usually a level mismatch or area mismatch.

**Fix:**

1. Confirm both sides have `level 1 disable` — if one side runs L1/L2 and the other runs L2 only, adjacency forms at L2 but may appear inconsistent.

2. Confirm both sides are in the same area:
   ```junos
   show isis adjacency detail
   ```
   Look for `Area addresses` in the output — both neighbors must share at least one area address.

---

## IS-IS Adjacency Up but No IPv4 Routes

**Symptom:** `show isis adjacency` shows `Up` but `show route protocol isis` is empty.

**Cause:** IS-IS adjacency runs over ISO (`family iso`) but IPv4 reachability requires `family inet` to also be present on the interface. If a transit interface is missing `family inet`, IS-IS will form an adjacency but will not carry IPv4 prefix information.

**Fix:** Confirm `family inet address X.X.X.X/Y` is configured on all transit interfaces:

```junos
show interfaces ge-0/0/0 terse
```

The unit should show both `inet` and `iso` address families.

---

## Routes Present but Loopback Missing

**Symptom:** Transit subnet routes appear in `show route protocol isis` but loopback routes are missing.

**Cause:** The loopback is not added to IS-IS as a passive interface, or `family iso` with the NET address is missing from `lo0 unit 0`.

**Fix:**

```junos
show configuration interfaces lo0
```

Should include:
```junos
unit 0 {
    family inet {
        address 10.0.0.1/32;
    }
    family iso {
        address 49.0001.0100.0000.0001.00;
    }
}
```

```junos
show configuration protocols isis
```

Should include `interface lo0.0 { passive; }`.

---

## `show route protocol isis` Shows Routes But Ping Fails

**Symptom:** IS-IS routes are in the routing table but pings between loopbacks fail.

**Fix:** Confirm the routes show `*` (active) and the next-hop interface is up:

```junos
show route 10.0.0.4
```

If the route is active, run a ping with source set to the loopback:

```junos
ping 10.0.0.4 source 10.0.0.1 count 3
```

If this succeeds but the sourceless ping fails, the return path is missing — check IS-IS on the remote routers.
