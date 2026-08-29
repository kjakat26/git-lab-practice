# Home Lab Infrastructure Overview — Services, Configs & Commands

**Last updated:** August 29, 2026
**Scope:** EVE-NG lab covering FortiGate, NGINX HA load balancing, DHCP/management networking, health monitoring, email alerting, and web-based diagnostics.

This is a quick-reference map of every use case built in this lab: which node runs it, what service provides it, where its config lives, and the commands to manage it day to day.

---

## 1. Network Emulation Platform

| | |
|---|---|
| **Node** | EVE-NG host (Ubuntu 22.04, VMware Workstation VM) |
| **Purpose** | Hosts the entire virtual topology — all other nodes run inside this |
| **Key directories** | `/opt/unetlab/addons/qemu/<image-folder>/` — one folder per node image (disk + ISO live here) |
| **Config files** | `/etc/netplan/*.yaml` — EVE-NG host's own network config (Ubuntu uses netplan, NOT `/etc/network/interfaces` like the Alpine guest nodes) |
| **Manage / fix permissions** | `/opt/unetlab/wrappers/unl_wrapper -a fixpermissions` — run after adding/editing any image files |
| **Web GUI backend** | `guacd` (proxy daemon) + `tomcat9` (serves the Guacamole HTML5 console webapp) |
| **Common commands** | `systemctl status guacd tomcat9`, `apt-get install --reinstall eve-ng-pro-guacamole` (fixes broken HTML5 console) |

---

## 2. Perimeter Firewall / Routing — FortiGate

| | |
|---|---|
| **Node** | FortiGate VM (FortiOS 7.0.12) |
| **Purpose** | Firewall policies between subnets, NAT, VIPs, DHCP relay point for production traffic |
| **Image location (EVE-NG host)** | `/opt/unetlab/addons/qemu/fortinet-Firewall7.0.12/virtioa.qcow2` |
| **Config access** | CLI only in this lab (`config system interface`, `config firewall policy`, `config router static`) |
| **Key interfaces** | port1 = WAN/management, port2 = client-facing LAN, port3 = DMZ (NGINX-facing) |
| **Manage commands** | `show system interface`, `show router static`, `get router info routing-table all`, `get system status` (also shows license state) |
| **License troubleshooting** | `diagnose debug vm-print-license`, `execute factoryreset2` (resets eval license, preserves interface/routing config) |

---

## 3. Load Balancing — NGINX (Active + Standby Pair)

| | |
|---|---|
| **Nodes** | NGINX-1 (10.12.1.3, MASTER), NGINX-2 (10.12.1.4, BACKUP) |
| **Purpose** | Reverse proxy + round-robin load balancing to Web-1/Web-2, SSL termination |
| **Service** | `nginx` |
| **Main config** | `/etc/nginx/http.d/default.conf` — contains the `upstream backend_pool` block + port 80/443 `server` blocks |
| **SSL cert location** | `/etc/nginx/ssl/nginx-selfsigned.crt` + `.key` |
| **Manage commands** | `nginx -t` (syntax check, always run before reload), `nginx -s reload`, `rc-service nginx restart`, `nginx -T` (dump full active config — useful for catching duplicate `server` blocks) |
| **Logs** | `/var/log/nginx/error.log` |
| **Enable at boot** | `rc-update add nginx default` |

---

## 4. High Availability — keepalived (VRRP Floating VIP)

| | |
|---|---|
| **Nodes** | NGINX-1 + NGINX-2 |
| **Purpose** | Automatic failover between the two NGINX nodes via a shared virtual IP |
| **Floating VIP** | 10.12.1.10 |
| **Service** | `keepalived` |
| **Required packages** | `keepalived keepalived-openrc keepalived-sample-config` (base package alone lacks the init script/config dir) |
| **Config file** | `/etc/keepalived/keepalived.conf` — differs only in `state` (MASTER/BACKUP) and `priority` (150/100) between the two nodes |
| **Manage commands** | `rc-service keepalived start/stop/restart`, `ip addr show eth0` (check which node currently holds the VIP) |
| **Logs / proof of failover** | `grep -i vrrp /var/log/messages` |
| **Enable at boot** | `rc-update add keepalived default` |

---

## 5. Backend Web Servers

| | |
|---|---|
| **Nodes** | Web-1 (10.10.10.2), Web-2 (10.10.10.3) |
| **Purpose** | Serve static content, targets of NGINX's load balancing |
| **Service** | `nginx` (plain static file server config, not a proxy) |
| **Config file** | `/etc/nginx/http.d/default.conf` — `root /var/www; index index.html;` |
| **Content location** | `/var/www/index.html` |
| **Manage commands** | Same as Section 3 (`nginx -t`, `rc-service nginx restart`) |

---

## 6. Node Networking (all Alpine nodes)

| | |
|---|---|
| **Applies to** | Every Alpine node in the lab (Web-1, Web-2, NGINX-1, NGINX-2, Client-1, Monitor/DHCP) |
| **Init system** | OpenRC (NOT netplan — that's only for the Ubuntu-based EVE-NG host, see Section 1) |
| **Config file** | `/etc/network/interfaces` |
| **Required package for persistence** | `ifupdown-ng` — without it, `networking` has no functional backend even if the config file exists |
| **Manage commands** | `rc-service networking restart`, `rc-update add networking default` (register at boot — only needs running once) |
| **Verify real disk vs. live/tmpfs boot** | `mount | grep " / "` — must show a real filesystem (e.g. `ext4`), **never** `tmpfs` (tmpfs means the node is booting from ISO/live mode and nothing persists) |
| **Hostname** | `setup-hostname <name>` (handles both `/etc/hostname` and the live hostname in one step) |

---

## 7. Management / DHCP Network

| | |
|---|---|
| **Node** | Monitor/DHCP node (10.99.99.1 on eth1, mgmt subnet) |
| **Purpose** | Central DHCP leasing + internet access for all lab nodes, avoiding repeated manual bridging for package installs |
| **Services** | `dnsmasq` (DHCP server), `iptables` (NAT/routing) |
| **DHCP config** | `/etc/dnsmasq.conf` — `interface=eth1`, `dhcp-range=...`, plus `dhcp-host=<mac>,<ip>` lines for fixed reservations |
| **NAT / forwarding config** | `/etc/sysctl.d/99-forwarding.conf` (`net.ipv4.ip_forward=1`), iptables rules (MASQUERADE + FORWARD) |
| **Lease data** | `/var/lib/misc/dnsmasq.leases` — live list of every currently-leased device (also used as the target source for the health-check script, see Section 8) |
| **Manage commands** | `rc-service dnsmasq restart`, `iptables -t nat -L -n -v` (confirm NAT rules), `cat /var/lib/misc/dnsmasq.leases` |
| **Persist iptables rules** | `/etc/init.d/iptables save` (required — rules vanish on reboot otherwise) |
| **Enable at boot** | `rc-update add dnsmasq default`, `rc-update add iptables default` |

---

## 8. Health Monitoring (Dynamic, Lease-Driven)

| | |
|---|---|
| **Node** | Monitor/DHCP node |
| **Purpose** | Automatically check reachability + HTTP status of every currently-leased device |
| **Script** | `/opt/monitor/healthcheck.sh` |
| **Scheduler** | cron (`busybox-cronie` package, `crond` service) |
| **Cron entry** | `/etc/crontabs/root` → `* * * * * /opt/monitor/healthcheck.sh` (runs every minute) |
| **Output log** | `/var/log/healthcheck.log` — every check, every run (full history) |
| **Data source** | `/var/lib/misc/dnsmasq.leases` (same file DHCP maintains — no separate target list to manage) |
| **Manage commands** | `/opt/monitor/healthcheck.sh` (run manually anytime), `tail -f /var/log/healthcheck.log` |
| **IMPORTANT — after any crontab edit** | `rc-service crond restart` — a plain `start` will NOT reload an already-running crond's job list; must be a genuine restart |
| **Enable at boot** | `rc-update add crond default` |

---

## 9. Email Alerting on State Change

| | |
|---|---|
| **Node** | Monitor/DHCP node |
| **Purpose** | Send an email only when a monitored host's status actually changes (not on every routine check) |
| **Mail service** | `msmtp` (relays through Gmail SMTP) |
| **Mail config** | `/etc/msmtprc` — contains Gmail app-password credential, `chmod 600` |
| **Mail send log** | `/var/log/msmtp.log` |
| **Alert logic** | Built into `/opt/monitor/healthcheck.sh` (same script as Section 8) — compares current check result against a per-host state file |
| **State tracking files** | `/var/lib/monitor/state/<hostname>` — one small file per host holding its last known status |
| **Alert-only log** | `/var/log/alerts.log` — only state-change events, not routine checks (contrast with the full log in Section 8) |
| **Manage / test** | `echo -e "Subject: Test\n\nBody" | msmtp your-email@gmail.com` (manual send test) |

---

## 10. Web-Based Diagnostics Tool

| | |
|---|---|
| **Node** | Monitor/DHCP node |
| **Purpose** | Browser-based MTR, port check, and HTTP service check — no SSH needed for quick diagnostics |
| **Services** | `nginx` (this node's THIRD distinct nginx role — separate from Sections 3/5) + `fcgiwrap` (bridges nginx to shell scripts) |
| **nginx config** | `/etc/nginx/http.d/monitoring.conf` — listens on `0.0.0.0:8080` |
| **fcgiwrap socket** | `/run/fcgiwrap/fcgiwrap.sock` (note: subdirectory — NOT `/run/fcgiwrap.sock`, a common wrong assumption) |
| **fcgiwrap config** | `/etc/conf.d/fcgiwrap` |
| **Web root** | `/var/www/monitor/index.html` (AJAX-based single-page form) |
| **CGI scripts** | `/var/www/monitor/cgi-bin/mtr.cgi`, `portcheck.cgi`, `servicecheck.cgi` — each validates input as strict IPv4/numeric before executing any command |
| **Access URL** | `http://192.168.100.158:8080` (Monitor node's home-network-facing IP) |
| **Manage commands** | `nginx -T | grep -A 3 listen` (confirm actual active bind config), `rc-service nginx restart` (note: `reload` does NOT reliably pick up `listen` directive changes — restart is required), `rc-service fcgiwrap status` |
| **mtr permission fix (if needed)** | `setcap cap_net_raw+ep /usr/sbin/mtr-packet` |

---

## Node Summary Table

| Node | Primary Role(s) | Production IP(s) | Mgmt IP |
|---|---|---|---|
| FortiGate | Firewall/routing | port2: 10.11.0.1, port3: 10.12.1.1 | — |
| NGINX-1 | Load balancer, HA MASTER | 10.12.1.3 (+ VIP 10.12.1.10 when MASTER) | 10.99.99.13 |
| NGINX-2 | Load balancer, HA BACKUP | 10.12.1.4 | 10.99.99.14 |
| Web-1 | Backend web server | 10.10.10.2 | 10.99.99.127 |
| Web-2 | Backend web server | 10.10.10.3 | 10.99.99.126 |
| Client-1 | Test client | 10.11.0.x | 10.99.99.77 |
| Monitor/DHCP | DHCP, monitoring, alerting, diagnostics | — | 10.99.99.1 (eth1), home LAN (eth0) |

---

## Cross-Cutting Commands (apply to any Alpine node)

```bash
# Package management
apk update
apk info -e <package>              # confirm TRUE install state (more reliable than apk list)
apk search <keyword>               # find exact package names when unsure

# Service management (OpenRC)
rc-service <name> start|stop|restart|status
rc-update add <name> default       # enable at boot
rc-update show default             # list everything enabled at boot

# Networking
ip addr show <iface>
ip route show
mount | grep " / "                 # sanity check: real disk vs tmpfs/live-boot

# Diagnosing "it should be working but isn't"
netstat -tlnp | grep <port>        # what's ACTUALLY listening, vs what a config file claims
find / -iname "<filename>" 2>/dev/null   # locate real file paths when docs/assumptions are wrong
```

---

## The One Habit Worth Keeping

Across nearly every issue hit while building this lab, the common thread was: **a config file looking correct is not the same as the running service using it.** Cron needed an explicit restart after every crontab edit. nginx needed a full restart (not reload) for `listen` directive changes. A package showing `[installed]` in one listing didn't always mean `apk info -e` agreed. Whenever something "should work" and doesn't, check what the live, running process is actually doing — `netstat`, `ps aux`, `-T`/`-t` dump-and-test flags, and log files — before re-reading the config file a third time assuming it must be right.
