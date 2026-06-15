# Session 2 — Troubleshooting

## Interface Shows `up/down`

**Symptom:** `show interfaces terse` shows `ge-0/0/0 up down` — admin up, link down.

**Cause / Fix:**
1. The GNS3 link is not drawn or the remote node is not running — check the canvas
2. The remote node's interface has no cable — right-click the node in GNS3 > Show interfaces
3. The remote node is still booting — wait for the `root@` prompt

---

## Ping to Remote Interface Fails

**Symptom:** `ping 10.1.12.2` returns `Request timeout`.

**Cause / Fix:**
1. Run `show interfaces ge-0/0/0 terse` — must be `up up`
2. Confirm the correct IP address is assigned: `show interfaces ge-0/0/0.0`
3. Check the remote router has the other /30 address (`10.1.12.2`) on its `ge-0/0/0.0`
4. Confirm both routers are in the same /30 subnet — a /30 has only 2 usable hosts

---

## Static Route Not Appearing in `show route`

**Symptom:** You configured the static route but it does not appear.

**Cause / Fix:**
1. Did you `commit`? Run `show | compare` — if your route shows as `+`, you did not commit yet
2. Check the next-hop is reachable: the next-hop must be within a directly connected subnet
3. If the next-hop interface is down, Junos marks the route **hidden** — run `show route hidden` to confirm

---

## `show route hidden`

If a static route's next-hop interface goes down, the route becomes hidden (inactive). Use:

```junos
show route hidden
show route hidden detail
```

The `detail` flag shows why the route is hidden (`next-hop dead` or `interface down`).

---

## Wrong Interface Configured

**Symptom:** You configured an address on `ge-0/0/1` instead of `ge-0/0/0`.

**Fix:**

```junos
configure

delete interfaces ge-0/0/1 unit 0 family inet address 10.1.12.1/30
set interfaces ge-0/0/0 unit 0 family inet address 10.1.12.1/30

commit
```

Always verify with `show interfaces terse` after committing.
