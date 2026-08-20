# FortiGate + NGINX Load Balancing Lab — Build & Troubleshooting Reference

**Date:** August 20, 2026
**Environment:** EVE-NG (VMware Workstation Pro, Windows 11 host), FortiGate VM 7.0.12, Alpine Linux 3.24.1 (x4 nodes)
**Objective:** Build a real-world-style DMZ topology — client → firewall policy → load balancer → backend web pool — using free tools only, after F5 BIG-IP VE trial licensing became unavailable for personal/non-corporate accounts.

---

## Final Topology

```
Client (Alpine, curl-capable)  --- port2 (10.11.0.0/24) --- FortiGate --- port3 (10.12.1.0/24) --- NGINX (Alpine)
                                                                                                          |
                                                                                              switch (10.10.10.0/24)
                                                                                                    /            \
                                                                                              Web-1 (10.10.10.2)  Web-2 (10.10.10.3)
```

| Node | Role | IP(s) |
|---|---|---|
| Client | Test client (curl) | `10.11.0.50/24`, gw `10.11.0.1` |
| FortiGate port2 | Client-facing (LAN) | `10.11.0.1/24` |
| FortiGate port3 | NGINX-facing (DMZ) | `10.12.1.1/24` |
| NGINX (eth0) | Load balancer, DMZ-facing | `10.12.1.2/24` |
| NGINX (eth1) | Load balancer, backend-facing | `10.10.10.1/24` |
| Web-1 | Backend web server | `10.10.10.2/24` |
| Web-2 | Backend web server | `10.10.10.3/24` |

**Result confirmed:** `curl http://10.12.1.2` from the client, repeated several times, alternates cleanly between "Response from Web-1" and "Response from Web-2" — round-robin load balancing proven end-to-end through a real firewall policy.

---

## Why NGINX Instead of F5

F5 BIG-IP VE requires a MyF5 trial registration, which — as of this lab session — **no longer accepts personal (non-corporate) email addresses**; F5's own documentation confirms personal trials are no longer offered through the standard self-service flow. Rather than chase a workaround, we pivoted to **NGINX Open Source** — genuinely free, no license/account needed, and arguably *more* representative of real-world production load balancing today than BIG-IP is in many modern shops. Core concepts (upstream pools, round-robin distribution, reverse proxying) transfer directly to F5 knowledge if/when trial access becomes available later (e.g. via a school or work email).

---

## Building Alpine Nodes in EVE-NG (from scratch, no vendor image)

Unlike FortiGate/F5, there's no pre-built qcow2 to download — Alpine is installed from its ISO like a real OS.

```bash
mkdir -p /opt/unetlab/addons/qemu/linux-alpine
cd /opt/unetlab/addons/qemu/linux-alpine
wget https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/x86_64/alpine-virt-3.24.1-x86_64.iso -O cdrom.iso
qemu-img create -f qcow2 virtioa.qcow2 8G
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

> **Tip:** Use Alpine's `latest-stable` URL path instead of hardcoding a version number — it always resolves to the current release, avoiding stale-link issues as Alpine ships new versions frequently.

In EVE-NG GUI: add node → type **Linux** → select `linux-alpine` image → 1 vCPU / 512MB RAM is plenty for Alpine + NGINX or a simple web server. Boot → it loads from `cdrom.iso` (empty disk) → login as `root`, no password → run `setup-alpine` and follow prompts (static IP per node's actual subnet, root password, timezone, proxy none, NTP chrony default, mirror option 1, install target = the virtual disk, mode = **sys**).

After install: `poweroff`, then on the EVE-NG host rename the ISO so future boots go straight to disk instead of re-triggering the installer:
```bash
mv /opt/unetlab/addons/qemu/linux-alpine/cdrom.iso /opt/unetlab/addons/qemu/linux-alpine/cdrom.iso.bak
```

**To clone for additional nodes (Web-1, Web-2, Client) instead of reinstalling each time:**
```bash
cd /opt/unetlab/addons/qemu/
cp -r linux-alpine linux-alpine-web1
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```
> **Important after cloning:** each clone shares the same hostname/IP/machine-id as the original disk. Fix on first boot of each clone:
> ```bash
> setup-hostname <new-name>
> setup-interfaces
> rm -f /etc/machine-id
> ```

---

## Issue 1 — EVE-NG Host: DNS Resolution Failure

**Symptom:**
```
wget: unable to resolve host address 'dl-cdn.alpinelinux.org'
```
Routing was confirmed fine (`ip route show` showed a valid default route), isolating this to DNS specifically.

**Root cause:** `/etc/resolv.conf` had no usable nameserver.

**Fix:**
```bash
echo "nameserver 8.8.8.8" >> /etc/resolv.conf
```
> If this resets after reboot, the system's DHCP client or `systemd-resolved` is likely auto-generating the file — a persistent fix requires editing the relevant netplan config instead.

---

## Issue 2 — `apk add python3` Fails: "no such package"

**Symptom:** Package installs failing on a freshly-installed Alpine node.

**Root cause (two possible, check both):**
1. Node has no internet route at all (common on backend-subnet nodes with no path out)
2. `/etc/apk/repositories` missing the `community` repo line (where many packages, including `nginx`, actually live)

**Diagnosis:**
```bash
ping -c 3 8.8.8.8          # confirms basic reachability
cat /etc/apk/repositories   # confirms repo config
```

**Fix (repo config):**
```bash
cat > /etc/apk/repositories << 'EOF'
https://dl-cdn.alpinelinux.org/alpine/v3.24/main
https://dl-cdn.alpinelinux.org/alpine/v3.24/community
EOF
apk update
```

**Fix (no internet route) — isolated backend-subnet nodes (Web-1/Web-2):**
Temporarily add a second NIC bridged to Cloud0 in EVE-NG, bring it up, install what's needed, then the node keeps the installed package even after removing/never using that interface again:
```bash
udhcpc -i eth1
apk update && apk add nginx
```

**Fix (no internet route) — DMZ-facing nodes with a real firewall path (NGINX):**
More realistic for a node that's supposed to have controlled outbound access — add a FortiGate NAT policy instead of a direct bridge:
```
config firewall policy
    edit 2
        set name "DMZ-to-WAN"
        set srcintf "port3"
        set dstintf "port1"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set service "ALL"
        set schedule "always"
        set nat enable
    next
end
```
Plus a temporary static default route on the node itself if none exists yet:
```bash
ip route add default via <gateway-ip> dev eth0
```

---

## Issue 3 — BusyBox `httpd` Applet Missing

**Symptom:** `httpd: not found` when trying to use BusyBox's built-in tiny web server (intended to avoid needing internet access at all for Web-1/Web-2).

**Root cause:** Alpine's default BusyBox build doesn't always compile in server-side applets (`httpd`, and reportedly `udhcpd` too, per Alpine's own bug tracker) — client-side tools are present, server-side ones sometimes aren't, depending on the build.

**Fix:** Confirm with `busybox --list | grep -i http` first. If absent, use Python's built-in HTTP server instead (`apk add python3`, or in this lab's case, install `nginx` directly on Web-1/Web-2 too, once temporary internet access is arranged per Issue 2).

---

## Issue 4 — `rc-service nginx start` Fails: "networking failed to start"

**Symptom:**
```
* Starting networking ...
ifquery: could not parse /etc/network/interfaces
* ERROR: networking failed to start
```

**Root cause:** `/etc/network/interfaces` **did not exist at all** on this node — likely because the `setup-alpine` installer's non-interactive defaults, combined with manual `ip addr add` commands used earlier to configure a second NIC (eth1), never actually wrote a persistent interfaces file.

```bash
cat /etc/network/interfaces
# cat: can't open '/etc/network/interfaces': No such file or directory
```

**Fix:** Rebuild the file manually with real, confirmed IPs (check with `ip addr show <iface>` first, don't guess):
```bash
cat > /etc/network/interfaces << 'EOF'
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 10.12.1.2
    netmask 255.255.255.0
    gateway 10.12.1.1

auto eth1
iface eth1 inet static
    address 10.10.10.1
    netmask 255.255.255.0
EOF
rc-service networking restart
```

**Workaround to unblock testing immediately without fixing the file first:**
```bash
nginx    # run the binary directly, bypassing rc-service's networking dependency chain
```

---

## Issue 5 — `curl http://localhost` Hangs / Times Out Despite nginx `LISTEN`ing

**Symptom:** `netstat -tlnp` confirmed nginx listening on `0.0.0.0:80` and `:::80`, worker process healthy, no firewall (`iptables`/`nft` not even installed) — yet `curl -v` hung and eventually timed out with no response, no "connection refused."

**Root cause:** **Loopback interface (`lo`) was administratively down.**
```bash
ip addr show lo
# 1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN qlen 1000
```
This traces directly back to Issue 4 — the missing `/etc/network/interfaces` file meant OpenRC's networking service never had an `auto lo` stanza to bring loopback up at boot. nginx was genuinely listening, but the interface carrying `127.0.0.1` traffic was never activated, so every SYN packet was silently dropped at the kernel level before reaching nginx — hence a timeout instead of an immediate rejection.

**Fix:**
```bash
ip link set lo up
```
Confirm: `ip addr show lo` should now show `state UNKNOWN` (Alpine's normal "up" wording for loopback) with `127.0.0.1/8` listed. This is also fixed permanently once `/etc/network/interfaces` is rebuilt (Issue 4's fix includes the `auto lo` stanza).

---

## Issue 6 — Getting a 404 Instead of Load-Balanced Response

**Symptom:** After fixing loopback, `curl http://localhost` returned a clean `404 Not Found` from nginx — not a timeout, not a 502, just "not found."

**Root cause:** `/etc/nginx/http.d/default.conf` contained **two `server` blocks** — Alpine's stock default (`return 404` for everything, explicitly marked `default_server`) stacked above the custom `upstream`/`proxy_pass` block added for this lab. Since the custom block had no matching `server_name` for `localhost`, and the stock block claimed `default_server` status, unmatched requests fell through to the 404 handler regardless of the upstream config being technically correct.

```bash
cat /etc/nginx/http.d/default.conf
# showed both the stock 404 block AND the custom upstream block in the same file
```

**Fix:** Rewrite the file with only the intended config:
```bash
cat > /etc/nginx/http.d/default.conf << 'EOF'
upstream backend_pool {
    server 10.10.10.2:80;
    server 10.10.10.3:80;
}

server {
    listen 80;
    server_name _;
    location / {
        proxy_pass http://backend_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF
nginx -t
nginx -s reload
```

> **Lesson:** always check for duplicate `default_server` declarations when adding a custom nginx site config on top of a distro's stock default — this is a common gotcha across most Linux distros' nginx packaging, not just Alpine.

---

## Issue 7 — VPCS Can't `curl`

**Symptom:** `curl` command not recognized/available on the VPCS node used as the test client.

**Root cause:** VPCS is a minimal simulated IP stack for basic reachability testing (`ping`, `trace`, `ip`, `show`) — not a real OS. It has no shell and cannot run arbitrary tools like `curl`.

**Fix:** Replace VPCS with a cloned Alpine node as the test client wherever L7 (HTTP-level) testing is needed. VPCS remains useful for quick L3 ping/reachability checks elsewhere in a topology, just not for this kind of test.

---

## Quick Reference — Diagnostic Commands Used Today

```bash
# DNS / connectivity
ping -c 3 8.8.8.8
cat /etc/resolv.conf

# Package management
cat /etc/apk/repositories
apk update && apk add <package>

# Interface / routing state
ip addr show <iface>
ip route show
ip route show table local

# Loopback specifically
ip addr show lo
ip link set lo up

# nginx diagnostics
netstat -tlnp | grep :80
ps aux | grep nginx
nginx -T                      # dumps full active config — check for duplicate server blocks
nginx -t                      # syntax check before reload
nginx -s reload
cat /var/log/nginx/error.log

# Low-level socket state (when netstat/curl behavior seems contradictory)
cat /proc/net/tcp | grep ':0050'    # hex for port 80
dmesg | tail -30
```

---

## The Pattern Worth Remembering

Today's chain of issues is a good real-world lesson in layered troubleshooting: a **missing config file** (Issue 4) silently caused a **dead loopback interface** (Issue 5), which looked like a completely unrelated "nginx isn't responding" mystery until traced back through `ip addr show lo`. Meanwhile Issue 6 (duplicate `default_server`) was a separate, unrelated bug that only became visible *after* Issue 5 was fixed — the timeout had been masking the 404 the whole time.

**Takeaway:** when a service "isn't responding" at all (timeout, not an error), suspect the network/interface layer before the application config. Once you get *any* response (even a wrong one, like a 404), that confirms the interface layer is fine and the problem has moved up the stack into application config — a good signal for narrowing where to look next.
