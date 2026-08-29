# NGINX HA, Management DHCP, Health Monitoring & Web Diagnostics — Reference

**Date:** August 26-29, 2026
**Environment:** EVE-NG, FortiGate VM 7.0.12, Alpine Linux 3.24.1 (NGINX-1, NGINX-2, Web-1, Web-2, Client-1, Monitor/DHCP node)
**Objective:** Add high availability to the NGINX load balancer pair, build a centralized management/DHCP network, implement dynamic health monitoring, and add a browser-based diagnostics tool (MTR, port check, service check).

---

## Part 1 — NGINX High Availability (keepalived + VRRP)

**Design:** two identical NGINX nodes share a floating virtual IP (VIP). Whichever is healthy and has the higher priority holds the VIP; if it goes down, the other takes over automatically within seconds.

| Node | Real IP | Role |
|---|---|---|
| NGINX-1 | 10.12.1.3 | MASTER (priority 150) |
| NGINX-2 | 10.12.1.4 | BACKUP (priority 100) |
| — | **10.12.1.10** | **Floating VIP** (whichever node is MASTER holds this) |

**Build steps:**
1. Clone NGINX-1's disk to create NGINX-2 (`cp -r linux-alpine linux-alpine-nginx2`)
2. Give NGINX-2 its own hostname, machine-id, and static IPs (do NOT reuse NGINX-1's addresses)
3. Replicate NGINX-1's exact nginx config (`upstream backend_pool`, port 80 + 443 server blocks, SSL cert) onto NGINX-2
4. Install keepalived on both — **note the sub-package requirement below**
5. Configure `/etc/keepalived/keepalived.conf` on each (MASTER vs BACKUP)
6. Enable and start, verify VIP placement, test failover

**Working keepalived configs:**

NGINX-1 (MASTER):
```
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass labpass1
    }
    virtual_ipaddress {
        10.12.1.10/24
    }
}
```

NGINX-2 (BACKUP) — identical except:
```
    state BACKUP
    priority 100
```

### Issue — `keepalived` installs but `/etc/keepalived/` doesn't exist

**Symptom:** `apk add keepalived` succeeds, `apk info -e keepalived` confirms installed, but there's no config directory, no sample config, and no OpenRC service script.

**Root cause:** Alpine splits `keepalived` into sub-packages — the base package alone doesn't include the init script or sample config.

**Fix:**
```bash
apk add keepalived keepalived-openrc keepalived-sample-config
mkdir -p /etc/keepalived
```
Then write the config manually (as above).

### Verifying and Testing Failover

**Confirm VIP placement:**
```bash
ip addr show eth0   # on MASTER — should list 10.12.1.10 as secondary
```

**Test failover:**
```bash
# On NGINX-1 (current MASTER)
rc-service keepalived stop
```
Confirm NGINX-2 picks up the VIP, and traffic keeps flowing (`curl -k https://10.12.1.10` from client, repeated — round-robin continues, served entirely by NGINX-2).

**Confirmed log evidence of a clean failover + failback cycle** (`grep -i vrrp /var/log/messages`):
```
# NGINX-2 taking over
Keepalived_vrrp: (VI_1) Entering FAULT STATE
Keepalived_vrrp: Netlink reports eth0 down
Keepalived_vrrp: Netlink reports eth0 up
Keepalived_vrrp: (VI_1) Entering BACKUP STATE
Keepalived_vrrp: (VI_1) Entering MASTER STATE

# NGINX-1 reclaiming MASTER after restart (higher priority wins)
Keepalived_vrrp: (VI_1) received lower priority (100) advert from 10.12.1.3 - discarding
Keepalived_vrrp: (VI_1) Entering MASTER STATE

# Correspondingly on NGINX-2
Keepalived_vrrp: (VI_1) Master received advert from 10.12.1.2 with higher priority 150, ours 100
Keepalived_vrrp: (VI_1) Entering BACKUP STATE
```
No split-brain observed — both nodes never claimed MASTER simultaneously.

**Note on testing with curl:** don't forget `-k` (self-signed cert) when testing the VIP directly — omitting it produces a cert error (`SSL: certificate subject name 'nginx-lb' does not match target hostname`) that looks like a failure but is actually just curl correctly refusing an untrusted/mismatched cert. If curl gets far enough to receive and evaluate a certificate at all, that confirms something live answered the TLS handshake — useful diagnostic signal even without `-k`.

---

## Part 2 — Centralized Management/DHCP Network

**Design:** a dedicated Monitor/DHCP node with two interfaces — one bridged to the home network (Cloud0, internet access), one serving a new isolated management subnet (`10.99.99.0/24`) that every other lab node joins via an additional NIC. This eliminates the repeated "temporarily bridge to Cloud0" workaround used earlier in the lab for package installs.

| Node | Mgmt IP |
|---|---|
| Monitor/DHCP | 10.99.99.1 (gateway) |
| Web-1 | 10.99.99.127 (reserved) |
| Web-2 | 10.99.99.126 (reserved) |
| Client-1 | 10.99.99.77 (dynamic) |
| NGINX-1/2 | not yet connected — deferred |

**Build steps:**
1. Clone a new Alpine node for Monitor/DHCP duties, with two NICs from creation (eth0 → Cloud0, eth1 → new Mgmt-Switch)
2. `eth0`: DHCP client (gets a real address from the home router). `eth1`: static `10.99.99.1/24`, no gateway line (eth0 remains the default route)
3. Install: `dnsmasq iptables iptables-openrc curl busybox-cronie ifupdown-ng`
4. Enable IP forwarding + NAT so the mgmt subnet can reach the internet through eth0
5. Configure dnsmasq as DHCP server on eth1 only
6. Add a second/third NIC on every other node, wired to Mgmt-Switch, configured as a DHCP client

**IP forwarding + NAT:**
```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.d/99-forwarding.conf
sysctl -p /etc/sysctl.d/99-forwarding.conf

iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT

rc-update add iptables default
/etc/init.d/iptables save
```

**dnsmasq config** (`/etc/dnsmasq.conf`):
```
interface=eth1
bind-interfaces
dhcp-range=10.99.99.50,10.99.99.150,12h
dhcp-option=3,10.99.99.1
dhcp-option=6,8.8.8.8
```
`bind-interfaces` is essential — without it dnsmasq may also attempt to serve DHCP on eth0 (the actual home network), which must be avoided.

**Fixed reservations for infrastructure nodes** (so monitoring/config can reference stable IPs instead of floating leases):
```bash
cat >> /etc/dnsmasq.conf << 'EOF'
dhcp-host=<web1-mac>,10.99.99.11
dhcp-host=<web2-mac>,10.99.99.12
dhcp-host=<nginx1-mac>,10.99.99.13
dhcp-host=<nginx2-mac>,10.99.99.14
EOF
rc-service dnsmasq restart
```
(Get each MAC via `ip link show <mgmt-iface>` on that node.)

### Issue — commented-out community repo on cloned nodes

**Symptom:** `apk add keepalived` on a freshly cloned node fails with "no such package" despite confirmed internet access.

**Root cause:** `/etc/apk/repositories` had the community line commented out:
```
#http://dl-cdn.alpinelinux.org/alpine/v3.24/community
```
This was present on the very first Alpine base image, meaning **every node cloned from it inherited the same disabled repo** — worth checking on any node hitting a "no such package" error for a community-tier package.

**Fix:**
```bash
cat > /etc/apk/repositories << 'EOF'
https://dl-cdn.alpinelinux.org/alpine/v3.24/main
https://dl-cdn.alpinelinux.org/alpine/v3.24/community
EOF
apk update
```

---

## Part 3 — Dynamic Health Monitoring

**Design evolution:** started with a hardcoded target list, then rebuilt to read live targets directly from dnsmasq's lease file — new devices get monitored automatically with zero script maintenance.

**Final script** (`/opt/monitor/healthcheck.sh`):
```bash
#!/bin/sh
LOG=/var/log/healthcheck.log
LEASES=/var/lib/misc/dnsmasq.leases

while read expiry mac ip hostname clientid; do
    [ -z "$ip" ] && continue

    if ping -c 1 -W 2 "$ip" >/dev/null 2>&1; then
        if curl -s --max-time 2 -o /dev/null "http://$ip"; then
            echo "$(date '+%Y-%m-%d %H:%M:%S') UP+WEB $hostname ($ip)" >> $LOG
        else
            echo "$(date '+%Y-%m-%d %H:%M:%S') UP     $hostname ($ip)" >> $LOG
        fi
    else
        echo "$(date '+%Y-%m-%d %H:%M:%S') DOWN    $hostname ($ip)" >> $LOG
    fi
done < "$LEASES"
```

**How it works:** reads `dnsmasq.leases` line by line (`expiry mac ip hostname clientid` — dnsmasq's native format), pings each leased device, and if it responds, additionally checks HTTP. Produces three distinct states:
- `UP+WEB` — machine alive AND serving HTTP
- `UP` — machine alive but not serving HTTP (either it doesn't run a web service, like Client-1, or a web service that should be running has crashed)
- `DOWN` — machine unreachable entirely

This distinction matters operationally: "box is dead" and "box is fine but the app crashed" are different problems requiring different fixes.

**Confirmed test results:**
| Action | Result |
|---|---|
| `rc-service nginx stop` on Web-1 | `UP` (ping succeeds, HTTP fails) |
| Web-1 powered off entirely | `DOWN` (ping fails) |
| Web-1 restored | back to `UP+WEB` |

### Issue — cron never picks up a crontab edit made after crond started

**Symptom:** log entries only appeared at moments matching manual script runs, never on a clean one-minute cadence, despite `* * * * * /opt/monitor/healthcheck.sh` being correctly present in `/etc/crontabs/root`.

**Root cause:** BusyBox's `crond` reads the crontab file once at startup and does not automatically detect edits made afterward. The line was added to the file *after* crond had already started, so it was invisible to the running process. `rc-service crond start` correctly refused to start a duplicate instance (`WARNING: crond has already been started`) but this did NOT reload the file either.

**Diagnosis:**
```bash
grep -i cron /var/log/messages | tail -20   # showed crond's actual start timestamp
cat /etc/crontabs/root                       # confirmed the line WAS present
```
Comparing the crond start timestamp against when the healthcheck line was added confirmed the timing mismatch.

**Fix:**
```bash
rc-service crond restart    # NOT 'start' — must be a genuine restart to force a re-read
```

**Takeaway:** any time a crontab is edited on a system where crond is already running, restart the service — don't assume the running daemon will notice the file changed.

---

## Part 4 — Web-Based Diagnostics Tool

**Design:** a lightweight nginx + fcgiwrap setup on the Monitor node, running three CGI shell scripts (MTR, port check, HTTP service check) behind a simple browser form. Originally scoped to listen only on the mgmt subnet (`10.99.99.1:8080`) for security separation, later changed to listen on all interfaces (`0.0.0.0:8080`) to allow direct browser testing from the Windows host via the mgmt node's home-network-facing IP.

**Packages:**
```bash
apk add nginx fcgiwrap spawn-fcgi mtr busybox-extras
```

**nginx config** (`/etc/nginx/http.d/monitoring.conf`) — final working version:
```
server {
    listen 0.0.0.0:8080;
    server_name _;
    root /var/www/monitor;
    index index.html;

    location /cgi-bin/ {
        gzip off;
        fastcgi_pass unix:/run/fcgiwrap/fcgiwrap.sock;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

**The three CGI scripts** (`/var/www/monitor/cgi-bin/`) — each validates input as a strict IPv4 pattern before touching a shell command, and (for port check) a numeric-only port:

`mtr.cgi`:
```bash
#!/bin/sh
echo "Content-type: text/plain"
echo ""
TARGET=$(echo "$QUERY_STRING" | sed -n 's/^.*target=\([^&]*\).*$/\1/p' | tr -d '\n')
if ! echo "$TARGET" | grep -qE '^[0-9]{1,3}(\.[0-9]{1,3}){3}$'; then
    echo "Invalid target. Only IPv4 addresses are accepted."
    exit 0
fi
mtr -r -c 4 "$TARGET"
```

`portcheck.cgi` — same validation pattern, then:
```bash
if nc -z -w 3 "$TARGET" "$PORT" 2>/dev/null; then
    echo "Port $PORT on $TARGET is OPEN"
else
    echo "Port $PORT on $TARGET is CLOSED or unreachable"
fi
```

`servicecheck.cgi` — same validation pattern, then:
```bash
STATUS=$(curl -s -o /dev/null -w "%{http_code}" --max-time 3 "http://$TARGET")
if [ "$STATUS" = "200" ]; then
    echo "$TARGET is UP (HTTP $STATUS)"
elif [ -n "$STATUS" ] && [ "$STATUS" != "000" ]; then
    echo "$TARGET responded with HTTP $STATUS (may indicate a problem)"
else
    echo "$TARGET is DOWN or unreachable"
fi
```

**Frontend** — evolved from plain HTML forms (which opened results in a new tab) to a single-page AJAX version using `fetch()`, displaying each check's result directly beneath its own form in a `<pre>` block. See current `/var/www/monitor/index.html` on the Monitor node for the live version — key mechanism:
```javascript
async function runCheck(script, targetId, portId, outputId) {
    const target = document.getElementById(targetId).value;
    let url = `/cgi-bin/${script}.cgi?target=${encodeURIComponent(target)}`;
    if (portId) url += `&port=${encodeURIComponent(document.getElementById(portId).value)}`;
    const res = await fetch(url);
    document.getElementById(outputId).textContent = await res.text();
}
```

### Issue — fcgiwrap socket path mismatch

**Symptom:** `rc-service fcgiwrap status` reports "started," but `/run/fcgiwrap.sock` doesn't exist, and nginx can't connect to the FastCGI backend.

**Root cause:** Alpine's fcgiwrap actually uses `/run/fcgiwrap/fcgiwrap.sock` (with a subdirectory) by default — confirmed via:
```bash
cat /etc/conf.d/fcgiwrap        # showed the commented-out default: unix:/run/fcgiwrap/fcgiwrap.sock
find / -iname "fcgiwrap.sock" 2>/dev/null   # located the real path
```
The originally-written nginx config assumed the more commonly-documented `/run/fcgiwrap.sock` path, which was simply wrong for this package/version.

**Fix:** update `fastcgi_pass` in the nginx config to the confirmed real path (`unix:/run/fcgiwrap/fcgiwrap.sock`), then `nginx -s reload`.

### Issue — 403 Forbidden on CGI script requests despite correct permissions

**Symptom:** `http://.../cgi-bin/mtr.cgi?target=...` returned 403, with nothing relevant in nginx's error log (only unrelated old bind-conflict and favicon-404 entries).

**Root cause:** a path-duplication bug in the nginx config itself:
```
fastcgi_param SCRIPT_FILENAME /var/www/monitor/cgi-bin$fastcgi_script_name;
```
`$fastcgi_script_name` already resolves to the full requested path (`/cgi-bin/mtr.cgi`), so hardcoding `/var/www/monitor/cgi-bin` in front of it produced a **doubled** path: `/var/www/monitor/cgi-bin/cgi-bin/mtr.cgi` — which genuinely doesn't exist. fcgiwrap's response to a missing/unexecutable target script is 403, not 404, which is why the symptom didn't look like an obvious "file not found" issue.

**Confirmed via:**
```bash
ls -la /var/www/monitor/cgi-bin/cgi-bin/mtr.cgi
# No such file or directory — confirmed the doubled path was the issue
```

**Fix** — use `$document_root` instead of hardcoding the path a second time:
```
fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
```
Since `$document_root` already resolves to `/var/www/monitor` (from the `root` directive) and `$fastcgi_script_name` is already the full `/cgi-bin/mtr.cgi` path, concatenating them correctly produces the real path with no duplication.

**Takeaway:** a 403 from fcgiwrap with nothing informative in nginx's own error log usually means the FastCGI backend itself rejected the request — check the actual resolved script path directly on disk (`ls -la <the-exact-path-your-config-builds>`) rather than assuming nginx's log will explain it.

### Issue — nginx `listen` directive change not taking effect after `reload`

**Symptom:** changed `listen 10.99.99.1:8080` to `listen 0.0.0.0:8080` in the config file, ran `nginx -s reload`, but `netstat -tlnp | grep 8080` still showed the old bind address.

**Root cause:** changing a `listen` directive's bind address is a bigger structural change than `reload` reliably picks up in all cases — a full restart is more reliable for this specific kind of change.

**Fix:**
```bash
rc-service nginx restart    # not reload
netstat -tlnp | grep 8080   # confirm 0.0.0.0:8080 now shown
```

---

## Quick Reference — Commands From This Session

```bash
# keepalived / VRRP
ip addr show eth0                      # check which node currently holds the VIP
grep -i vrrp /var/log/messages         # failover event history

# DHCP / management network
cat /var/lib/misc/dnsmasq.leases       # currently active leases
iptables -t nat -L -n -v               # confirm NAT rules are active

# Cron
cat /etc/crontabs/root                 # confirm a job is actually registered
grep -i cron /var/log/messages         # confirm crond's actual start time & job runs
rc-service crond restart               # required after ANY crontab edit on a running system

# Web diagnostics / fcgiwrap
find / -iname "fcgiwrap.sock" 2>/dev/null    # locate the real socket path if unsure
nginx -T | grep -A 3 listen                  # confirm nginx's actual active bind config
netstat -tlnp | grep <port>                  # confirm what's really listening, vs. what the config claims
```

---

## The Pattern Worth Remembering

Two recurring themes across today's issues:

1. **A config file being correct doesn't mean the running service is using it.** Cron didn't reload after an edit; nginx didn't fully apply a `listen` change after a `reload` (needed `restart` instead). Whenever a change "should have worked" but didn't, check what the *running process* is actually doing (`netstat`, `nginx -T`, log timestamps) rather than re-reading the config file and assuming it's live.

2. **A generic error code doesn't always point at the layer you'd expect.** The 403 looked like a permissions problem (matching how 403s usually present) but was actually a path-construction bug one layer down, in fcgiwrap rather than nginx or the filesystem. When an error doesn't match its usual cause, verify the actual resolved path/value the system is using, rather than re-checking the same permissions repeatedly.
