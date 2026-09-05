# Multicast PIM-SM Lab — FRR on Alpine (3-Router Build) Reference

**Date:** September 2026
**Environment:** EVE-NG, Alpine Linux + FRRouting 10.6.1 (R1, R2, R3), Alpine MC-Source, Alpine MC-Receiver
**Objective:** Build a real, working PIM-SM multicast lab — RP, shared tree, source-specific tree, Register/SPT switchover — using genuinely lightweight nodes instead of heavy vendor router images, then prove it with real application traffic rather than synthetic/static test entries.

---

## Why FRR-on-Alpine Instead of Juniper vJunos-router

The original plan used Juniper's vJunos-router image for vendor-syntax familiarity (relevant to real work use). However:
- **Juniper vMX is now end-of-life**, no longer downloadable
- **vJunos-router requires 4 vCPU / 6144MB RAM per node** — a 3-router topology alone would need ~18GB, more than the available 16GB host

**Pivot:** use **FRRouting (FRR)** — genuine, production-grade open-source routing software (its `pimd` component implements real RFC 4601 PIM-SM, and FRR itself runs in real production networks under various vendors) — on lightweight Alpine nodes instead. Each node runs comfortably in under 512MB, making a 3-router + 2-endpoint topology entirely feasible on modest hardware.

**One real unknown going in:** whether EVE-NG's Alpine kernel supports `IP_MROUTE` (Linux kernel multicast routing) at all. **Confirmed working** — verified via `/proc/net/ip_mr_cache` and `/proc/net/ip_mr_vif` both existing with real content (including a `pimreg` virtual interface) immediately after `pimd` started on R1.

---

## Final Topology

```
MC-Source (10.1.1.2) -- R1 (1.1.1.1) -- R2/RP (2.2.2.2) -- R3 (3.3.3.3) -- MC-Receiver (10.1.4.2)
                        eth0/eth1        eth0/eth1        eth0/eth1
```

| Node | Role | Loopback | eth0 | eth1 |
|---|---|---|---|---|
| R1 | First-hop router | 1.1.1.1/32 | 10.1.1.1 (source-side) | 10.1.2.1 (link to R2) |
| R2 | RP | 2.2.2.2/32 | 10.1.2.2 (link to R1) | 10.1.3.1 (link to R3) |
| R3 | Last-hop router | 3.3.3.3/32 | 10.1.3.2 (link to R2) | 10.1.4.1 (receiver-side) |
| MC-Source | Real sender | — | 10.1.1.2 | — |
| MC-Receiver | Real receiver | — | 10.1.4.2 | — |

Multicast group used throughout: **239.1.1.1:5000**

---

## Part 1 — Building R1 (Proof of Concept)

```bash
apk update
apk add frr frr-pythontools
sed -i 's/zebra=no/zebra=yes/' /etc/frr/daemons
sed -i 's/pimd=no/pimd=yes/' /etc/frr/daemons
sed -i 's/ospfd=no/ospfd=yes/' /etc/frr/daemons
grep -E "^(zebra|pimd|ospfd)=" /etc/frr/daemons   # verify — don't trust sed silently

echo "net.ipv4.ip_forward=1" >> /etc/sysctl.d/99-forwarding.conf
sysctl -p /etc/sysctl.d/99-forwarding.conf

rc-update add frr default
rc-service frr start
```

**Kernel multicast support verification (the real go/no-go check):**
```bash
cat /proc/net/ip_mr_cache 2>&1
cat /proc/net/ip_mr_vif 2>&1
```
Confirmed: both returned real content, including a `pimreg` virtual interface — proof `pimd` successfully registered with the kernel.

**R1 final config, applied via `vtysh`:**
```
interface eth0
 ip pim
interface eth1
 ip pim
interface lo
 ip address 1.1.1.1/32
router ospf
 network 10.1.1.0/24 area 0
 network 10.1.2.0/24 area 0
 network 1.1.1.1/32 area 0
router pim
 rp 2.2.2.2 224.0.0.0/4
```

### Issue A — `ip igmp` accidentally applied to the wrong interface

**Symptom:** none functionally — caught during a manual review of `show running-config`, not from an error.

**Root cause:** `ip igmp` was applied to R1's `eth0` (source-side), but IGMP is only meaningful on a **receiver**-facing interface — a source doesn't join the group, it just sends to it. Not harmful, but doesn't match the design and adds noise.

**Fix:**
```
interface eth0
 no ip igmp
```
**Lesson:** always verify `show running-config` against the intended design after applying changes — a config can be syntactically valid and still not match intent.

---

## Part 2 — Cloning R1 into R2 and R3

```bash
# Stop R1 first for filesystem consistency
cp -r linux-alpine-r1 linux-alpine-r2
cp -r linux-alpine-r1 linux-alpine-r3
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

### Issue B — Cloned nodes inherit the source node's SAVED FRR config, not a blank slate

**Symptom:** none yet at clone time — but a real risk, since `write memory` on R1 had already persisted its config to `/etc/frr/frr.conf` on disk, meaning R2 and R3 would boot with R1's exact hostname, loopback, and OSPF networks unless explicitly overwritten.

**Fix approach:** on each clone, explicitly remove the inherited R1-specific lines before adding the correct ones:
```
hostname R2
interface lo
 no ip address 1.1.1.1/32
 ip address 2.2.2.2/32
router ospf
 no network 10.1.1.0/24 area 0
 no network 1.1.1.1/32 area 0
 network 10.1.3.0/24 area 0
 network 10.1.2.0/24 area 0
 network 2.2.2.2/32 area 0
```
(R3 configured analogously, plus adding back `ip igmp` on its receiver-facing `eth1`, which R1's clone never had.)

**Lesson:** cloning a node that already has `write memory`'d application-level config (not just OS-level network config) means you're inheriting TWO layers of prior state — the OS network config (already a known pattern from earlier in this project) AND the application's own saved config file. Both need auditing after any clone.

**One node (R3) was built fresh/manually instead of cloned**, due to an unresolved cloning quirk on that attempt — worked fine as a clean, from-scratch FRR install using the same install/enable steps as R1.

---

## Part 3 — OSPF Underlay

### Issue C — OSPF neighbor never formed

**Symptom:** `show ip ospf neighbor` returned empty on R1 despite correct-looking config on both R1 and R2.

**Root cause:** the actual EVE-NG topology link between R1 and R2 was never cabled — config was correct, but there was no physical (virtual) connection at all.

**Fix:** wire R1's `eth1` to R2's `eth0` in the EVE-NG canvas per the addressing plan, then confirm basic reachability before even looking at OSPF:
```bash
ping 10.1.2.2   # from R1
```
**Lesson:** always confirm Layer 3 reachability first when a routing protocol won't adjacency — a protocol-level symptom (no OSPF neighbor) can have a purely physical-layer cause.

### Confirmed working state
```
vtysh -c "show ip ospf neighbor"
```
All three routers reached `Full` state (R1↔R2 as DR/BDR, R2↔R3 as DR/BDR) — full convergence confirmed before moving to PIM.

---

## Part 4 — PIM-SM Configuration and Verification

### Issue D — R2 (the RP) didn't recognize itself as the RP

**Symptom:**
```
show ip pim rp-info
RP address  ...  OIF      I am RP  ...
2.2.2.2     ...  Unknown  no       ...
```
R2 owns `2.2.2.2` on its own loopback, but reported `I am RP: no` with `OIF: Unknown`.

**Root cause:** `ip pim` had been enabled on R2's `eth0`/`eth1`, but **never on `lo` itself**. `pimd` determines local RP ownership by checking whether a PIM-enabled *local interface* holds the RP address — the IP being present on `lo` wasn't sufficient without PIM also being enabled there.

**Fix:**
```
interface lo
 ip pim
```
**Confirmed after fix:**
```
RP address  ...  OIF  I am RP  ...
2.2.2.2     ...  lo   yes      ...
```

**Lesson:** enabling a protocol on the interface holding an address is not automatic just because the address exists there — this applies to loopbacks just as much as physical interfaces.

---

## Part 5 — Real Traffic: MC-Source and MC-Receiver

Rather than a static/synthetic IGMP entry, two Alpine nodes running actual Python sockets were used, so the join and the traffic are both genuinely real.

**Receiver** (`mc_receive.py`) — uses `socket.IP_ADD_MEMBERSHIP` to issue a real IGMP join:
```python
import socket, struct
MCAST_GRP = '239.1.1.1'
MCAST_PORT = 5000
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sock.bind(('', MCAST_PORT))
mreq = struct.pack("4sl", socket.inet_aton(MCAST_GRP), socket.INADDR_ANY)
sock.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP, mreq)
print(f"Listening for multicast on {MCAST_GRP}:{MCAST_PORT}...")
while True:
    data, addr = sock.recvfrom(1024)
    print(f"Received from {addr}: {data.decode()}")
```

**Sender** (`mc_send.py`) — **`IP_MULTICAST_TTL` set explicitly to 32**, since the default (often 1) would cause packets to die at the first hop and never reach beyond the local subnet:
```python
import socket, time
MCAST_GRP = '239.1.1.1'
MCAST_PORT = 5000
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)
sock.setsockopt(socket.IPPROTO_IP, socket.IP_MULTICAST_TTL, 32)
i = 0
while True:
    sock.sendto(f"Hello multicast group! seq={i}".encode(), (MCAST_GRP, MCAST_PORT))
    i += 1
    time.sleep(2)
```

**Confirmed real join reached R3:**
```
show ip igmp groups
239.1.1.1  ...  Uptime 00:00:39
```
(A live uptime counter — proof of a genuine tracked membership, not a static entry.)

**Confirmed genuine end-to-end delivery** — receiver printed real received messages with the actual source IP and port.

---

## Part 6 — Verified Multicast Forwarding State

```
show ip mroute
```

**R1:** `10.1.1.2  239.1.1.1  SFT  PIM  eth0 → eth1`
**R2:** two entries — `(*, 239.1.1.1)` (shared tree) and `(10.1.1.2, 239.1.1.1)` (source-specific), both flagged `T`
**R3:** `10.1.1.2  239.1.1.1  ST  STAR  eth0 → eth1`

**Flag meanings confirmed in this test:**
- `S` — Sparse mode
- `F` — Register flag (traffic was, at some point, unicast-Registered to the RP)
- `T` — SPT-bit set (this router has switched to native shortest-path-tree forwarding)

---

## Part 7 — Packet Captures: Watching PIM Actually Work

**Data traffic, confirmed identical at every hop** (`tcpdump -i eth1 -n host 239.1.1.1` / `-i any`):
```
IP 10.1.1.2.45937 > 239.1.1.1.5000: UDP, length 28
```
Same packet content visible at R1, R2, and R3 — direct visual proof of multicast's "replication without duplication" model (the destination address never changes hop to hop, unlike unicast next-hop addressing).

**On R2 specifically, the switchover was caught live:**
```
pimreg In  IP 10.1.1.2... > 239.1.1.1...   (first packet — arrived via Register/pimreg)
eth0  M   IP 10.1.1.2... > 239.1.1.1...   (every packet after — arrived as native multicast)
eth1  Out IP 10.1.1.2... > 239.1.1.1...
```

**The actual Register handshake, captured directly** (`tcpdump -i eth1 -n host 2.2.2.2` on R1, timed to catch a fresh source after prior state aged out):
```
10.1.1.1 > 2.2.2.2: PIMv2, Register, length 64
10.1.1.1 > 2.2.2.2: PIMv2, Register, length 64
2.2.2.2 > 10.1.1.1: PIMv2, Register Stop, length 18
```
This is the literal mechanism behind the `F`→`T` flag transition: R1 unicast-wraps the multicast payload and sends it to the RP as a **Register** message; once R2 has built real forwarding state, it sends **Register-Stop**, and every subsequent packet switches to native multicast — confirmed here with real timestamps showing the whole handshake completing in about 6 milliseconds.

---

## Quick Reference — Verification Commands Used

```bash
# Kernel-level multicast support (run once, early)
cat /proc/net/ip_mr_cache
cat /proc/net/ip_mr_vif

# OSPF underlay
vtysh -c "show ip ospf neighbor"

# PIM control plane
vtysh -c "show ip pim neighbor"
vtysh -c "show ip pim rp-info"
vtysh -c "show ip igmp groups"

# Multicast forwarding state
vtysh -c "show ip mroute"

# Packet-level proof
tcpdump -i <iface> -n host 239.1.1.1        # data traffic
tcpdump -i <iface> -n host <rp-address>     # catches Register/Register-Stop
tcpdump -i <iface> -n proto 103             # PIM control messages generally
```

---

## Key Takeaways

1. **Open-source routing software (FRR) can fully substitute for heavy vendor router images** when the goal is protocol behavior/understanding rather than vendor-specific CLI practice — at a fraction of the resource cost.
2. **Cloning inherits saved application config, not just OS config** — anything already `write memory`'d exists on the cloned disk and must be explicitly audited/overwritten, the same discipline already learned for OS-level network config earlier in this project.
3. **A protocol not adjacency-ing doesn't always mean a protocol-level bug** — check the physical/virtual link first.
4. **Enabling a protocol on an address doesn't happen implicitly just because the address exists on an interface** — this bit both `lo`'s PIM-RP recognition and is a pattern worth remembering generally.
5. **Real, live packet captures are the strongest possible verification** — they don't just confirm a feature works, they reveal the actual mechanism (Register → Register-Stop) that a CLI `show` command can only summarize with a flag letter.
