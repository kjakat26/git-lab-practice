# FortiGate Study Guide: VLAN, VDOM, ISP, FMG, Policy, Profiles

Interview prep notes. Each section has the facts, then a **Remember** line to make it stick.

---

## 1. VLAN

**What it is:** Splits one physical network into separate logical Layer 2 networks using 802.1Q tags.

**On FortiGate:**

- Create a **VLAN sub-interface** on a physical port (e.g. `port2`, VLAN ID 10).
- Each sub-interface has its own IP (the gateway), DHCP server, and policies.
- The switch port facing the FortiGate must be a **trunk** carrying those VLAN IDs.
- Inter-VLAN traffic is routed by the FortiGate, so every flow is subject to policy.

```
config system interface
    edit "vlan10"
        set vdom "root"
        set interface "port2"
        set vlanid 10
        set ip 192.168.10.1 255.255.255.0
        set allowaccess ping
    next
end
```

**Common issues:** switch trunk missing the VLAN ID, VLAN ID mismatch, no policy.

> **Remember:** A VLAN is a **separate room in the same building**. The FortiGate is the **security guard at the door** between rooms.

---

## 2. VDOM (Virtual Domain)

**What it is:** Splits the FortiGate into multiple independent virtual firewalls.

**Each VDOM has its own:** interfaces, routing table, policies, objects, VPNs, security profiles, admins, logs.

**Key terms:**

- **Management VDOM:** `root` by default. Handles FortiGuard updates, licensing, and the FMG connection.
- **Global settings:** firmware, HA, system-wide settings. Live outside VDOMs.
- **Inter-VDOM traffic** does not flow automatically. It needs a **vdom-link** or an external connection.

**Enable and create:**

```
config system global
    set vdom-mode multi-vdom
end

config vdom
    edit CustomerA
    next
end
```

In multi-VDOM mode the CLI splits into `config global` and `config vdom` / `edit <name>`.

**Moving an interface to a VDOM:** remove every reference first (policies, routes, DHCP), then:

```
config global
    config system interface
        edit "vlan10"
            set vdom "CustomerA"
```

> **Remember:** VDOM = **separate apartments in one building**. Each has its own front door (interfaces), house rules (policies), and mailbox (routing table). VLAN = rooms, VDOM = whole apartments.

---

## 3. VLAN vs VDOM

|  | VLAN | VDOM |
| --- | --- | --- |
| Splits | The network (Layer 2) | The firewall itself |
| Own routing table | No | Yes |
| Own admins | No | Yes |
| Overlapping IPs allowed | No | Yes |
| Typical use | Segment one organization | Multi-tenant, hard isolation |

They combine: one trunk port carries many VLANs, and each VLAN sub-interface can be assigned to a different VDOM.

> **Remember:** **VLAN splits the wire. VDOM splits the brain.**

---

## 4. When to use multiple VDOMs

**Good use cases**

1. MSP or hosting: one box, many customers (the classic case).
2. Overlapping IP ranges (mergers, multiple customers).
3. Hard separation: corporate, guest, PCI, DMZ with different admin teams.
4. Edge VDOM (internet and VPN) plus internal VDOM (LAN).
5. Branch consolidation.
6. Test VDOM on a production unit.

**Wrong choice when**

- One organization, one network (VLANs plus policies are enough).
- You only need to separate VLANs (policies already do that).
- Small model with a low VDOM limit or extra licensing.
- You don't want per-VDOM overhead (routes, policies, DNS, logging each).

**Rule of thumb**

| Need | Use |
| --- | --- |
| Separate subnets, same org | VLANs plus policies |
| Separate customers, own routing and admins | Multiple VDOMs |
| Regulatory hardware isolation | Separate physical firewalls |

> **Remember:** Ask **"how many independent firewalls do I need?"** If the answer is one, don't use VDOMs.

---

## 5. Inter-VDOM link

Connects two VDOMs inside the same FortiGate.

```
config global
    config system vdom-link
        edit "vlnk-A"
        next
    end
    config system interface
        edit "vlnk-A0"
            set vdom "root"
            set ip 10.255.0.1 255.255.255.252
        next
        edit "vlnk-A1"
            set vdom "CustomerA"
            set ip 10.255.0.2 255.255.255.252
        next
    end
end
```

- Creates **two virtual interfaces**, one per VDOM.
- Needs **routes and policies on both sides** (each VDOM filters independently).
- On NP models you may see hardware-accelerated `npu-vlink`.
- Return routing is the usual mistake: `root` needs a route back to the customer subnet.

> **Remember:** A vdom-link is a **short network cable inside the box**. Two ends, so **two policies** and **two routes**.

---

## 6. Separate ISP per VDOM

**Option A: dedicated port per VDOM**

- `wan1` stays in `root` (ISP 1), `wan2` goes to `CustomerA` (ISP 2).
- Each VDOM has its own default route and NAT policy. No vdom-link needed.

```
config global
    config system interface
        edit "wan2"
            set vdom "CustomerA"
```

Then inside `CustomerA`: set the IP on `wan2`, add a static default route via `wan2`, and a policy (`vlan10` to `wan2`, NAT on).

**Option B: both ISPs on one physical port**

- A switch tags ISP 1 as VLAN 100 and ISP 2 as VLAN 200, with a single trunk to `port1`.
- Create `port1.100` and `port1.200` as VLAN sub-interfaces.
- The parent port stays in one VDOM (usually `root`), but each **sub-interface can belong to a different VDOM**.

**Watch out for**

- Single port, cable, and switch is a single point of failure, and bandwidth is shared.
- ISP VLAN IDs must not clash with other VLANs on the same port.
- The management VDOM (`root`) still needs a working internet link for FortiGuard and licensing.
- Using both ISPs for one network is SD-WAN or policy routing, not VDOMs.

> **Remember:** A port belongs to **one VDOM**, but its **VLAN sub-interfaces can be shared out**. One cable, many tagged lanes.

---

## 7. Any port can be a WAN

- "WAN" is a **role label** (`lan`, `wan`, `dmz`, `undefined`), not hardware. `port5` can be WAN.
- The role only affects GUI behavior and defaults, not forwarding.

**Real limits**

- Physical port count and speed or type (copper, SFP, SFP+).
- Internal switch ports on small models (40F, 60F) must be broken out before they're standalone.
- `mgmt` and `ha` ports are meant for management and heartbeat.
- Some ports share accelerated paths.
- A port belongs to one VDOM, unless you use VLAN sub-interfaces.

If you run out of ports: use VLAN handoff, free up the internal switch, or add a switch with trunks.

> **Remember:** **"WAN" is a name tag, not a hardware type.**

---

## 8. FortiManager (FMG) with multi-VDOM and HA

**HA challenges**

- Form the cluster **first**, then add it as **one device**.
- Use an IP that follows the primary (not one unit's own management IP).
- After failover, the new primary must re-establish the FGFM tunnel (TCP 541).
- Direct changes on the FortiGate cause out-of-sync status.
- Installs go through the primary and sync to the secondary. Upgrades are rolling.

**Multi-VDOM challenges**

- FGFM comes from the **management VDOM**, so it needs a route to FMG.
- Each VDOM needs its own **policy package**.
- **Object name conflicts** during import (same name, different values).
- Interface mapping can differ per VDOM.
- ADOM design: splitting VDOMs across ADOMs needs advanced ADOM mode. ADOM version must match FortiOS.
- Install ordering: moving an interface between VDOMs can need two installs.
- HA, global settings, and VDOM creation are device-level, not in policy packages.
- Licensing: a VDOM can count as a managed device. Check this.

**FortiGate side**

```
config system central-management
    set type fortimanager
    set fmg "10.0.0.50"
end
```

Allow `fgfm` in the interface `allowaccess`.

**Workflow:** build HA and VDOMs, confirm `root` reaches FMG on 541, add the cluster once, import each VDOM into its own package, resolve conflicts during import, test a failover, then manage only from FMG.

> **Remember:** **Cluster first, one device, root reaches FMG, one package per VDOM, FMG is the boss.**

---

## 9. Firewall policy vs security profile

|  | Firewall policy | Security profile |
| --- | --- | --- |
| Question | Should this traffic be allowed? | Is anything bad inside it? |
| Layer | L3/L4 (plus identity) | L7 (content, apps, URLs) |
| Standalone | Yes | No, must be attached to a policy |
| Order matters | Yes, top-down first match | No |
| Reusable | One entry per rule | Shared across many policies |

**Flow**

```
Traffic -> policy match? -- no --> implicit deny
              | yes (accept)
              v
        security profiles (AV, web filter, IPS, app control)
              v
        allow / block / log
```

**Related items**

- **SSL inspection profile:** decrypts HTTPS so other profiles can see inside.
- **Profile group:** bundle of profiles attached as one.
- **Inspection mode:** flow or proxy.

**Why it shows up in FMG onboarding**

- Name conflicts (two devices with a profile called `default`).
- Built-in default profiles with custom edits.
- Profiles live in the **shared object database**. Editing one affects every policy and device using it.
- Same-named profiles per VDOM can differ.
- FortiOS version differences cause unsupported-field install errors.
- FortiGuard licensing and updates.

> **Remember:** **Policy is the bouncer** (are you on the list?). **Profile is the bag check** (what are you carrying?). No bouncer pass, no bag check.

---

## 10. Flow-based vs proxy-based

|  | Flow-based | Proxy-based |
| --- | --- | --- |
| Method | Inspects the stream inline | Terminates the session, buffers, scans fully |
| Speed | Faster, lower latency | Slower, higher latency |
| CPU and memory | Lower | Higher |
| Acceleration | Better fit for NP/CP offload | Limited |
| Scan depth | Good | More thorough |
| Features | Fewer | More (explicit proxy, ICAP, WAF, full DLP) |

```
Flow:   Client ===== stream inspected inline ===== Server
Proxy:  Client -> [FortiGate: buffer + scan] -> Server  (two TCP sessions)
```

**Where to set it:** per policy in newer FortiOS (`set inspection-mode flow` or `proxy`), per VDOM in older versions. Profiles can also carry a feature set (`flow` or `proxy`).

**FMG impact:** a proxy-only profile on a flow policy can fail install, and mixed modes across VDOMs create import conflicts. Standardize on one default mode.

**Choose**

- **Flow:** default, high throughput, branches.
- **Proxy:** explicit proxy, ICAP, WAF, strict file scanning, and enough hardware.

> **Remember:** **Flow is a security camera** (watches as things pass). **Proxy is a customs checkpoint** (stops, opens, inspects everything).

---

## 11. CLI cheat sheet

```
get system status                      # shows VDOM mode and version
config vdom / edit CustomerA           # enter a VDOM
config global                          # global settings
get router info routing-table all      # routes (per VDOM)
diagnose sniffer packet vlan10 'icmp' 4
diagnose debug flow ...                # trace policy matching
diagnose sys vdom-admin list
get hardware nic                       # port details
show system settings | grep inspection # inspection mode
```

---

## 12. Likely interview questions

**What is the difference between a VLAN and a VDOM?** VLAN segments the network at Layer 2. VDOM splits the firewall into independent virtual firewalls with their own routing, policies, and admins.

**Can two VDOMs talk to each other?** Not by default. You need a vdom-link (or an external path), plus routes and policies on both sides.

**How do you give each VDOM its own ISP?** Assign a dedicated port to each VDOM, or hand off both ISPs as VLANs on one trunk and assign each sub-interface to a different VDOM.

**Can any port be used as WAN?** Yes. WAN is only a role label. Limits are port count, type, internal switch grouping, and the one-VDOM-per-port rule.

**Which VDOM connects to FMG?** The management VDOM (`root` by default), over FGFM on TCP 541.

**What goes wrong adding an HA multi-VDOM cluster to FMG?** Adding before the cluster is formed, using a unit-specific IP, no route from the management VDOM, object name conflicts, interface mapping, and ADOM or version mismatches.

**Policy vs security profile?** The policy allows or denies traffic. The profile inspects the content of allowed traffic. A profile does nothing unless attached to an accept policy.

**Flow vs proxy?** Flow inspects the stream inline for speed. Proxy buffers and fully scans for depth and extra features, at a performance cost.

**Why would an install fail after changing a security profile?** Profile name conflict, proxy-only option on a flow policy, a field not supported by the FortiOS version, or a profile shared by other policies.

---

## 13. One-page memory tricks

| Topic | Trick |
| --- | --- |
| VLAN | Rooms in one building |
| VDOM | Separate apartments |
| VLAN vs VDOM | VLAN splits the wire, VDOM splits the brain |
| vdom-link | Short cable inside the box, two ends, two policies |
| One port, two ISPs | One cable, many tagged lanes |
| WAN port | A name tag, not a hardware type |
| FMG onboarding | Cluster first, one device, root reaches FMG, FMG is the boss |
| Policy vs profile | Bouncer vs bag check |
| Flow vs proxy | Security camera vs customs checkpoint |

**Closing tip for the interview:** when asked "when would you use X?", answer with the **need** first ("I need independent firewalls for multiple customers"), then the feature. It shows design thinking, not just command knowledge.