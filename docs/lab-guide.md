# Lab Guide — Segment Routing Using a Mapping Server

Build order for the five-router IOS-XR topology. Each task is self-contained; verify
before moving on.

Devices: **R1** (mapping server, area 1) · **R2** (ABR 1↔0) · **R3** (area 0) ·
**R4** (ABR 0↔2) · **R5** (area 2).

---

## Task 1 — Hostnames, loopbacks and links

Every router gets a `/32` loopback from 1.1.1.0/24 matching its number, and `/24`
point-to-point links numbered `10.1.<near><far>.0`.

**R1**
```
hostname R1
!
interface Loopback0
 ipv4 address 1.1.1.1 255.255.255.255
!
interface GigabitEthernet0/0/0/0
 ipv4 address 10.1.12.1 255.255.255.0
 no shutdown
!
```

**R2**
```
hostname R2
!
interface Loopback0
 ipv4 address 1.1.1.2 255.255.255.255
!
interface GigabitEthernet0/0/0/0
 ipv4 address 10.1.12.2 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/0/0/1
 ipv4 address 10.1.23.2 255.255.255.0
 no shutdown
!
```

**R3**
```
hostname R3
!
interface Loopback0
 ipv4 address 1.1.1.3 255.255.255.255
!
interface GigabitEthernet0/0/0/0
 ipv4 address 10.1.34.3 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/0/0/1
 ipv4 address 10.1.23.3 255.255.255.0
 no shutdown
!
```

**R4**
```
hostname R4
!
interface Loopback0
 ipv4 address 1.1.1.4 255.255.255.255
!
interface GigabitEthernet0/0/0/0
 ipv4 address 10.1.34.4 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/0/0/1
 ipv4 address 10.1.45.4 255.255.255.0
 no shutdown
!
```

**R5**
```
hostname R5
!
interface Loopback0
 ipv4 address 1.1.1.5 255.255.255.255
!
interface GigabitEthernet0/0/0/1
 ipv4 address 10.1.45.5 255.255.255.0
 no shutdown
!
```

> IOS-XR interfaces are administratively down by default — `no shutdown` is not optional,
> and nothing is live until you `commit`.

**Verify:** `show ipv4 interface brief` — every configured interface Up/Up. Ping each
directly connected neighbour.

---

## Task 2 — Segment Routing Global Block

```
segment-routing
 global-block 16000 23999
!
```

Applied on all five routers. 16000–23999 is the IOS-XR default, so the fabric ends up
with a uniform SRGB and identical labels for a given index on every node — which makes
the `show mpls forwarding` output easy to read the first time through.

**Variation worth running afterwards:** give each router its own block —
R2 `20000 29999`, R3 `30000 39999`, R4 `40000 49999`, R5 `50000 59999`. Connectivity is
unchanged, but a traceroute now shows a different label at each hop for the same
destination. That is the clearest demonstration that the mapping server advertises a
*SID index*, and each router computes its own label as `SRGB base + index`.

Set the SRGB **before** OSPF establishes adjacencies. Changing it later requires
`clear ospf process` to reallocate labels.

**Verify:** `show segment-routing local-block inconsistencies` (should be empty) and
`show mpls label table detail | include SRGB`.

---

## Task 3 — OSPF with Segment Routing

Three lines enable SR on the process itself:

| Command | Purpose |
|---------|---------|
| `segment-routing mpls` | Enable SR extensions for this OSPF process |
| `segment-routing forwarding mpls` | Program the MPLS forwarding plane from SR |
| `network point-to-point` | Skip DR/BDR election on the Ethernet links |

**R1** — area 1 only
```
router ospf 1
 segment-routing mpls
 network point-to-point
 segment-routing forwarding mpls
 area 1
  interface Loopback0
   passive enable
  !
  interface GigabitEthernet0/0/0/0
  !
 !
!
```

**R2** — ABR, area 1 and area 0
```
router ospf 1
 segment-routing mpls
 network point-to-point
 segment-routing forwarding mpls
 area 0
  interface GigabitEthernet0/0/0/1
  !
 !
 area 1
  interface Loopback0
   passive enable
  !
  interface GigabitEthernet0/0/0/0
  !
 !
!
```

**R3** — backbone only
```
router ospf 1
 segment-routing mpls
 network point-to-point
 segment-routing forwarding mpls
 area 0
  interface Loopback0
   passive enable
  !
  interface GigabitEthernet0/0/0/0
  !
  interface GigabitEthernet0/0/0/1
  !
 !
!
```

**R4** — ABR, area 0 and area 2
```
router ospf 1
 segment-routing mpls
 network point-to-point
 segment-routing forwarding mpls
 area 0
  interface GigabitEthernet0/0/0/0
  !
 !
 area 2
  interface Loopback0
   passive enable
  !
  interface GigabitEthernet0/0/0/1
  !
 !
!
```

**R5** — area 2 only
```
router ospf 1
 segment-routing mpls
 network point-to-point
 segment-routing forwarding mpls
 area 2
  interface Loopback0
   passive enable
  !
  interface GigabitEthernet0/0/0/1
  !
 !
!
```

Loopbacks are `passive enable` — advertised into OSPF, but no adjacency attempted.

**Verify before continuing.** Four adjacencies, all FULL:

```
show ospf neighbor
show route ospf
```

R5 must have routes to 1.1.1.1 through 1.1.1.4. `show mpls forwarding` at this point
shows only `SR Adj` entries — adjacency SIDs exist, but no Prefix-SIDs, because nothing
has advertised any yet. That is the expected "before" state.

---

## Task 4 — Mapping server on R1

One block, on R1 only:

```
segment-routing
 mapping-server
  prefix-sid-map
   address-family ipv4
    1.1.1.1/32 1 range 100
   !
  !
 !
!
```

The three arguments are **start prefix**, **start index**, **range size**:

| Prefix | SID index | Resulting label (SRGB 16000) |
|--------|-----------|------------------------------|
| 1.1.1.1/32 | 1 | 16001 |
| 1.1.1.2/32 | 2 | 16002 |
| 1.1.1.3/32 | 3 | 16003 |
| 1.1.1.4/32 | 4 | 16004 |
| 1.1.1.5/32 | 5 | 16005 |
| … up to 1.1.1.100/32 | 100 | 16100 |

A range of 100 covers far more than the five loopbacks in use — which is the point.
Adding a sixth router at 1.1.1.6 requires no change to the mapping server at all; it
picks up index 6 automatically.

Nothing is configured on R2–R5. They already have SR enabled, and that is all they need
to receive and act on the mapping.

**Verify on R1:**
```
show segment-routing mapping-server prefix-sid-map ipv4 detail
```

---

## Task 5 — Verify

See [`verification.md`](verification.md) for full command output. The short version, in
order:

1. **R1** — `show ospf database opaque-area 7.0.0.1`
   The Extended Prefix Range LSA exists, `Flags: 0x0`.
2. **R2** — `show ospf database opaque-area 7.0.0.1 self-originate`
   The ABR re-originated it into area 0 with the IA-flag set (`Flags: 0x80`).
3. **R5** — `show mpls forwarding`
   `SR Pfx (idx N)` entries for every loopback, learned entirely from the mapping server.
4. **R5** — `ping 1.1.1.1 source Loopback0` and `traceroute 1.1.1.1 source 1.1.1.5`
   100% success, every hop label-switched.

---

## Suggested experiments

Once the base lab works:

- **Per-router SRGB** — as described in Task 2. Re-run the traceroute and watch the label
  change per hop.
- **Shrink the range** — change `range 100` to `range 3`. R4 and R5 lose their
  Prefix-SIDs; confirm where the traffic falls back to.
- **Move the mapping server** — configure the same range on R5 instead of R1. The result
  is identical, which demonstrates that the mapping server has no forwarding role and
  need not sit at the edge of the network.
- **Two mapping servers** — configure the same range on both R1 and R5, then configure
  overlapping but conflicting ranges, and observe how the conflict is resolved.
- **Break an ABR** — shut R2's area 0 interface and confirm the mapping never reaches
  area 0 or area 2, reproducing the "LSA present locally, absent remotely" failure in the
  troubleshooting table.
