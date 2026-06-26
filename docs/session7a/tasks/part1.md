# Part 1 — L2circuit (Pseudowire)

## Overview

An **L2circuit** (also called a pseudowire) is a point-to-point Layer 2 connection across an MPLS backbone. The provider stretches a single Ethernet segment between two CE-facing interfaces — CE1 and CE2 behave as if there is a direct cable between them, even though their traffic crosses the entire provider backbone.

### How It Works

Two configuration elements are required on each PE:

**1. CCC encapsulation on the CE-facing interface**

Setting `encapsulation ethernet-ccc` on the PE's CE-facing interface marks it as a Circuit Cross-Connect (CCC) port. The router stops processing the arriving Ethernet frames as IP packets and instead treats them as raw Layer 2 data to be forwarded across a pseudowire. The interface has no IP address.

**2. L2circuit neighbor statement**

The `protocols l2circuit neighbor` statement identifies the remote PE (by loopback address) and assigns a **virtual-circuit-id** — a locally significant number that both ends must agree on. This triggers an automatic targeted LDP session between the two PE loopbacks. Through that targeted LDP session, each PE signals a VC label to its peer.

When PE1 receives a frame on ge-0/0/2:
1. Looks up the L2circuit for virtual-circuit-id 100 toward neighbor 10.0.0.4
2. Pushes the **VC label** (inner, identifies the pseudowire on PE2)
3. Pushes the **LDP transport label** (outer, carries the packet to PE2 via P1 and P2)
4. Forwards out ge-0/0/0 toward P1

P1 and P2 swap the outer transport label as in any MPLS-forwarded packet. They never see or process the inner VC label.

When PE2 receives the labeled packet:
1. LDP pops the outer transport label (PHP or explicit pop)
2. PE2 looks up the VC label in its LFIB
3. Strips the VC label and delivers the original Ethernet frame out ge-0/0/2 to CE2

CE2 receives an unmodified Ethernet frame — no MPLS headers, no encapsulation overhead visible to the customer device.

## Step 1: Configure L2circuit on PE1

```junos
configure

set interfaces ge-0/0/2 encapsulation ethernet-ccc
set protocols l2circuit neighbor 10.0.0.4 interface ge-0/0/2.0 virtual-circuit-id 100

commit
```

## Step 2: Configure L2circuit on PE2

```junos
configure

set interfaces ge-0/0/2 encapsulation ethernet-ccc
set protocols l2circuit neighbor 10.0.0.1 interface ge-0/0/2.0 virtual-circuit-id 100

commit
```

!!! warning "virtual-circuit-id must match on both PEs"
    PE1 uses `neighbor 10.0.0.4` (PE2's loopback) and PE2 uses `neighbor 10.0.0.1` (PE1's loopback). Both use `virtual-circuit-id 100`. If the IDs do not match, the pseudowire will not come up — the targeted LDP negotiation will reject the mismatch.

## Step 3: Verify the Pseudowire

Allow 10–20 seconds for the targeted LDP session to establish and the VC labels to be exchanged.

On **PE1**:

```junos
show l2circuit connections
```

Expected output when the pseudowire is up:

```text
Layer-2 Circuit Connections:

Legend for connection status (St)
EI -- encapsulation invalid      NP -- interface h/w not present
MM -- mtu mismatch               Dn -- down
EM -- encapsulation mismatch     VC-Dn -- Virtual circuit down
CM -- control word mismatch      Up -- operational
VM -- vlan id mismatch
CN -- circuit not provisioned    OL -- no outgoing label
OO -- out of range label         NC -- encapsulation not CCC/TCC

Legend for interface status
Up -- operational
Dn -- down

Neighbor: 10.0.0.4
    Interface                 Type  St     Time last up          # Up trans
    ge-0/0/2.0(vc 100)        rmt   Up     Jun 26 12:00:00               1
      Remote PE: 10.0.0.4, Negotiated control-word: No
      Incoming label: 299824, Outgoing label: 299840
      Local interface: ge-0/0/2.0, Status: Up, Encapsulation: ETHERNET
```

Key fields:

- **`St: Up`** — the pseudowire is operational. Any other state (Dn, VC-Dn, OL, CM) indicates a configuration or transport problem.
- **`Incoming label`** — the VC label PE2 assigned for this pseudowire and signaled to PE1. PE1 pushes this as the inner label when forwarding frames toward PE2.
- **`Outgoing label`** — the VC label PE1 assigned and signaled to PE2. PE2 pushes this when forwarding frames toward PE1.
- **`Encapsulation: ETHERNET`** — both ends agreed on Ethernet encapsulation (matching `ethernet-ccc`).
- **`Negotiated control-word: No`** — the control word (an optional 4-byte header for pseudowire sequencing and fragmentation) was not negotiated. This is the expected default for Ethernet pseudowires.

Also verify the targeted LDP session that was automatically created:

```junos
show ldp session
```

Expected — now two sessions on PE1: the original link LDP session to P1, and a new targeted LDP session to PE2's loopback:

```text
Address                          State       Connection  Hold time  I/F
10.0.0.2                         Operational Open          27        3
10.0.0.4                         Operational Open          27        1
```

The second session (to 10.0.0.4) is the targeted LDP session. It was created automatically when `protocols l2circuit neighbor 10.0.0.4` was configured.

## Step 4: Test CE-to-CE Connectivity

With the pseudowire up, CE1 and CE2 can reach each other as if directly connected on the same Ethernet segment.

```junos
CE1> ping 192.168.1.2 count 5
```

Expected — all five replies succeed:

```text
PING 192.168.1.2 (192.168.1.2): 56 data bytes
64 bytes from 192.168.1.2: icmp_seq=0 ttl=64 time=32.140 ms
64 bytes from 192.168.1.2: icmp_seq=1 ttl=60 time=41.872 ms
64 bytes from 192.168.1.2: icmp_seq=2 ttl=64 time=28.654 ms
64 bytes from 192.168.1.2: icmp_seq=3 ttl=64 time=37.221 ms
64 bytes from 192.168.1.2: icmp_seq=4 ttl=64 time=29.508 ms

--- 192.168.1.2 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 28.654/33.879/41.872/4.688 ms
```

!!! note "TTL behavior through the pseudowire"
    Unlike the CE-to-CE ping in Session 7, no `source` flag is required here. CE1 and CE2 are in the same /24 subnet — ARP resolves CE2's MAC address and the ICMP request goes directly via Layer 2. The provider backbone is invisible to CE1 and CE2 at the IP level.

    The TTL seen in replies may be 64 (if the provider core does not decrement TTL during label swap) or lower depending on the vMX tunnel TTL propagation setting. Both are acceptable.

Verify the reverse direction:

```junos
CE2> ping 192.168.1.1 count 5
```

Expected: all five replies succeed.
