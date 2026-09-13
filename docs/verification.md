# Verification Reference

Commands used to confirm the mapping server lab is working, in the order worth running
them, with the output to expect. All commands are Cisco IOS-XR.

---

## Quick reference

| Goal | Command | Run on |
|------|---------|--------|
| OSPF adjacencies up | `show ospf neighbor` | all |
| Mapping configured | `show segment-routing mapping-server prefix-sid-map ipv4 detail` | R1 |
| Advertisement flooded | `show ospf database opaque-area 7.0.0.1` | R1 |
| Inter-area re-origination | `show ospf database opaque-area 7.0.0.1 self-originate` | R2, R4 |
| Mapping received and resolved | `show ospf segment-routing prefix-sid-map` | R3, R5 |
| Labels programmed | `show mpls forwarding` | R5 |
| SRGB allocated | `show mpls label table detail` | any |
| Data plane | `ping` / `traceroute` with loopback source | R5 |

---

## 1. Baseline — before the mapping server

Worth capturing so you have a "before" to compare against. On R5, with OSPF and SR up but
no mapping server yet:

```
RP/0/0/CPU0:R5#show mpls forwarding
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes
Label  Label       or ID              Interface                    Switched
------ ----------- ------------------ ------------ --------------- --------
16000  Pop         SR Adj (idx 0)     Gi0/0/0/1    10.1.45.4       0
16001  Pop         SR Adj (idx 0)     Gi0/0/0/1    10.1.45.4       0
```

Only `SR Adj` entries — adjacency SIDs, allocated locally the moment SR forwarding came
up. No `SR Pfx` entries, because no Prefix-SID has been advertised by anyone.

---

## 2. The mapping on R1

```
RP/0/0/CPU0:R1#show segment-routing mapping-server prefix-sid-map ipv4 detail
```

Confirms the range is active locally. If this is empty, the `prefix-sid-map` block did not
commit — check for a missing `address-family ipv4`.

---

## 3. The advertisement — R1, area 1

```
RP/0/0/CPU0:R1#show ospf database opaque-area 7.0.0.1

            OSPF Router with ID (1.1.1.1) (Process ID 1)

                Type-10 Opaque Link Area Link States (Area 1)

  LS age: 1249
  Options: (No TOS-capability, DC)
  LS Type: Opaque Area Link
  Link State ID: 7.0.0.1
  Opaque Type: 7
  Opaque ID: 1
  Advertising Router: 1.1.1.1
  LS Seq Number: 80000002
  Checksum: 0x2fd9
  Length: 48

    Extended Prefix Range TLV: Length: 24
      AF        : 0
      Prefix    : 1.1.1.1/32
      Range Size: 100
      Flags     : 0x0

      SID sub-TLV: Length: 8
        Flags     : 0x60
        MTID      : 0
        Algo      : 0
        SID Index : 1
```

Reading this:

- **Opaque Type 7** is the OSPF Extended Prefix Range LSA — the container SR uses to carry
  a mapping server range. The Link State ID `7.0.0.1` is `<opaque type>.0.0.<opaque id>`.
- **Type-10** means area-scoped. It floods within area 1 and no further on its own; an ABR
  must re-originate it to reach other areas.
- **Prefix / Range Size / SID Index** are the mapping itself — 100 prefixes starting at
  1.1.1.1/32, indices starting at 1.
- **Flags: 0x0** — no IA-flag. This is the original advertisement in its home area.
- **Algo: 0** — SPF. Algorithm 1 would be Strict-SPF.

Every router in area 1 now has the mapping.

---

## 4. Crossing an area boundary — R2, the ABR

```
RP/0/0/CPU0:R2#show ospf database opaque-area 7.0.0.1 self-originate

            OSPF Router with ID (1.1.1.2) (Process ID 1)

                Type-10 Opaque Link Area Link States (Area 0)

  LS age: 1750
  Link State ID: 7.0.0.1
  Opaque Type: 7
  Opaque ID: 1
  Advertising Router: 1.1.1.2
  LS Seq Number: 80000002
  Checksum: 0xaed8
  Length: 48

    Extended Prefix Range TLV: Length: 24
      AF        : 0
      Prefix    : 1.1.1.1/32
      Range Size: 100
      Flags     : 0x80

      SID sub-TLV: Length: 8
        Flags     : 0x60
        MTID      : 0
        Algo      : 0
        SID Index : 1
```

Two differences from R1's copy, and both matter:

**Advertising Router is 1.1.1.2, not 1.1.1.1.** A Type-10 LSA cannot leave its area. R2
did not forward R1's LSA — it generated a new one into area 0 carrying the same mapping.
`self-originate` is what filters the output to R2's own copy.

**Flags is 0x80 — the IA-flag is set.** This marks the advertisement as having already
crossed an area boundary. The rule an ABR follows: *do not propagate a mapping server
advertisement received from a non-backbone area with the IA-flag already set.*

Without that rule, consider two ABRs joining the same pair of areas. ABR-A re-originates
into the backbone; ABR-B receives it, re-originates back into the non-backbone area; ABR-A
receives that and re-originates again — the advertisement circulates indefinitely. The
IA-flag terminates the cycle after exactly one boundary crossing.

R4 does the same thing at the area 0 ↔ area 2 boundary, so the mapping reaches R5.

---

## 5. Mapping received on a remote router

```
RP/0/0/CPU0:R5#show ospf segment-routing prefix-sid-map
```

Shows the ranges R5 has received and how it resolved them against prefixes in its LSDB.
This is the step between "LSA arrived" and "label programmed" — if the LSA is present but
this is empty, the prefixes are not in R5's routing table (check that loopbacks are
advertised into OSPF).

---

## 6. Programmed labels — R5

R5 sits in area 2, four hops and two area boundaries from the mapping server, with no
Prefix-SID configuration of its own:

```
RP/0/0/CPU0:R5#show mpls forwarding
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes
Label  Label       or ID              Interface                    Switched
------ ----------- ------------------ ------------ --------------- --------
16000  Pop         SR Adj (idx 0)     Gi0/0/0/1    10.1.45.4       0
16001  Pop         SR Adj (idx 0)     Gi0/0/0/1    10.1.45.4       0
16001  16001       SR Pfx (idx 1)     Gi0/0/0/1    10.1.45.4       0
16002  16002       SR Pfx (idx 2)     Gi0/0/0/1    10.1.45.4       0
16003  16003       SR Pfx (idx 3)     Gi0/0/0/1    10.1.45.4       0
16004  Pop         SR Pfx (idx 4)     Gi0/0/0/1    10.1.45.4       0
```

The `SR Pfx (idx N)` entries are the result. Each one is `SRGB base + index`:
index 1 → 16001 for 1.1.1.1, index 2 → 16002 for 1.1.1.2, and so on. Index 4 shows
`Pop` because R4 is the penultimate hop for 1.1.1.4 — standard PHP.

**With a uniform SRGB the local and outgoing labels are identical**, which makes this
output easy to read but hides the interesting part. Re-run the lab with per-router SRGBs
and the same table becomes:

```
50001  40001       SR Pfx (idx 1)     Gi0/0/0/1    10.1.45.4
50002  40002       SR Pfx (idx 2)     Gi0/0/0/1    10.1.45.4
```

R5 (SRGB 50000) allocates 50001 locally; R4 (SRGB 40000) expects 40001. Same index,
different labels — proof that the index is the global identifier and the label is purely
local.

---

## 7. Data plane

```
RP/0/0/CPU0:R5#ping 1.1.1.1 source Loopback0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 1.1.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 19/37/109 ms
```

```
RP/0/0/CPU0:R5#traceroute 1.1.1.1 source 1.1.1.5
Type escape sequence to abort.
Tracing the route to 1.1.1.1

 1  10.1.45.4 [MPLS: Label 16001 Exp 0] 79 msec  19 msec  19 msec
 2  10.1.34.3 [MPLS: Label 16001 Exp 0] 29 msec  9 msec  19 msec
 3  10.1.23.2 [MPLS: Label 16001 Exp 0] 29 msec  9 msec  19 msec
 4  10.1.12.1 29 msec  *  29 msec
```

Every hop is MPLS label-switched. The final hop has no label — R2 popped it as the
penultimate hop for 1.1.1.1.

The label is constant across hops here only because the SRGB is uniform. With per-router
SRGBs the same traceroute reads `40001`, `30001`, `20001` — one swap per hop, same
destination.

The whole path was built without configuring a single Prefix-SID outside R1.

---

## Failure signatures

| What you see | What it means |
|--------------|---------------|
| No Opaque Type 7 LSA on R1 | Mapping server never committed — re-check the `prefix-sid-map` block |
| LSA on R1, none in area 0 | ABR not re-originating; check R2 has interfaces in both areas and adjacencies are FULL |
| LSA everywhere, no `SR Pfx` entries | `segment-routing forwarding mpls` missing from the OSPF process |
| `SR Pfx` for some loopbacks only | Range size too small, or those prefixes are not in OSPF |
| `SR Adj` entries only, nothing else | Working as expected *before* Task 4 — the mapping server is not active yet |
| Labels outside the configured SRGB | SRGB changed after OSPF came up; `clear ospf process` to reallocate |
| Ping fails but labels look right | Loopback missing from OSPF, or missing `passive enable` under the area |
