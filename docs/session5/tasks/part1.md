# Part 1 — Verify IS-IS

## IS-IS vs OSPF — Key Differences in Output

Before running verification commands, note two important differences from OSPF:

| Property | OSPF | IS-IS |
|----------|------|-------|
| Neighbor command | `show ospf neighbor` | `show isis adjacency` |
| Database command | `show ospf database` | `show isis database` |
| Database entries | LSAs (Type 1–7) | LSPs (Link State PDUs) |
| Route preference | 10 | 18 (L2 internal) |
| Default metric | 1 per interface | 10 per interface |

The higher default metric (10) means IS-IS route costs grow faster than OSPF across multiple hops. PE1 to PE2 will show metric 30 (three hops × 10) vs OSPF's metric 3.

## Step 1: Verify Adjacency

Run on all four routers:

```junos
show isis adjacency
```

Expected on PE1:

```text
Interface             System         L State        Hold (secs) SNPA
ge-0/0/0.0           P1             2  Up                   23
```

Expected on P1 (two neighbors):

```text
Interface             System         L State        Hold (secs) SNPA
ge-0/0/0.0           PE1            2  Up                   22
ge-0/0/1.0           P2             2  Up                   27
```

**State `Up`** means the adjacency is fully established. **`L`** shows the IS-IS level — `2` confirms Level 2 only, as configured.

## Step 2: Examine the Link-State Database

```junos
PE1> show isis database
```

Expected — four LSPs, one per router:

```text
IS-IS level 2 link-state database:
  LSP ID                      Sequence  Checksum  Lifetime  Attributes
  PE1.00-00                   0x00000003  0x...      1180  L1 L2
  P1.00-00                    0x00000004  0x...      1175  L1 L2
  P2.00-00                    0x00000003  0x...      1177  L1 L2
  PE2.00-00                   0x00000002  0x...      1172  L1 L2
```

Sequence numbers, checksums, and lifetimes will differ in your output. The important check is that all four LSPs are present — each router generates exactly one LSP describing its links and reachable prefixes.

!!! note "LSP vs LSA"
    In OSPF, each router generates multiple LSAs of different types (Router LSA, Network LSA, etc.). In IS-IS, each router generates a single LSP that contains all of its information in TLV fields. The result is the same: a complete topology map distributed to all routers.

## Step 3: Verify IS-IS Routes

```junos
PE1> show route protocol isis
```

Expected — routes to all remote loopbacks and transit subnets:

```text
inet.0: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32        *[IS-IS/18] 00:02:10, metric 10
                    > to 10.1.12.2 via ge-0/0/0.0
10.0.0.3/32        *[IS-IS/18] 00:02:10, metric 20
                    > to 10.1.12.2 via ge-0/0/0.0
10.0.0.4/32        *[IS-IS/18] 00:02:10, metric 30
                    > to 10.1.12.2 via ge-0/0/0.0
10.1.23.0/30       *[IS-IS/18] 00:02:10, metric 20
                    > to 10.1.12.2 via ge-0/0/0.0
10.1.34.0/30       *[IS-IS/18] 00:02:10, metric 30
                    > to 10.1.12.2 via ge-0/0/0.0
```

The `[IS-IS/18]` tag confirms IS-IS is the source (preference 18). Metrics reflect three hops × default cost 10 = metric 30 to reach PE2's loopback.

## Step 4: Verify End-to-End Connectivity

```junos
PE1> ping 10.0.0.4 count 3
```

Expected:

```text
PING 10.0.0.4 (10.0.0.4): 56 data bytes
64 bytes from 10.0.0.4: icmp_seq=0 ttl=62 time=3.4 ms
64 bytes from 10.0.0.4: icmp_seq=1 ttl=62 time=3.1 ms
64 bytes from 10.0.0.4: icmp_seq=2 ttl=62 time=3.2 ms
```

TTL=62 confirms the packet traversed two intermediate hops (P1 and P2), same as with OSPF.

```junos
PE2> ping 10.0.0.1 count 3
```

Expected: 3/3 replies from PE1's loopback.

## Step 5: Confirm IS-IS Interface State

```junos
PE1> show isis interface
```

Expected:

```text
IS-IS interface database:
Interface             L CirID Level 1 DR    Level 2 DR    L1/L2 Metric
ge-0/0/0.0            2   0x1 Disabled      Point to Point 10/10
lo0.0                 2   0x1 Passive        Passive        0/0
```

`Point to Point` on the transit interface and `Passive` on the loopback confirms the configuration is correct.
