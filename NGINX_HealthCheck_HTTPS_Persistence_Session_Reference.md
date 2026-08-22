# NGINX Health Checks, HTTPS Termination & Alpine Persistence — Session Reference

**Date:** August 22, 2026
**Environment:** EVE-NG, FortiGate VM 7.0.12, Alpine Linux 3.24.1 (Client, NGINX, Web-1, Web-2)
**Objective:** Add passive health-check failover and TLS termination to the FortiGate + NGINX load-balancing lab, and resolve a persistence problem that surfaced while doing it — Alpine nodes were losing all configuration on every reboot.

---

## Part 1 — Passive Health Checks

**Key clarification:** NGINX **Open Source** does not have active health checks (proactive polling of backends) — that's an NGINX Plus (paid) feature. Open-source NGINX has **passive health checks** instead: it watches real client traffic, and if a backend fails enough times, temporarily stops sending it requests. This is still a legitimate, widely-used production pattern.

**Config (`/etc/nginx/http.d/default.conf` on the NGINX node):**
```nginx
upstream backend_pool {
    server 10.10.10.2:80 max_fails=2 fail_timeout=10s;
    server 10.10.10.3:80 max_fails=2 fail_timeout=10s;
}

server {
    listen 80;
    server_name _;
    location / {
        proxy_pass http://backend_pool;
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

- `max_fails=2` / `fail_timeout=10s` — after 2 failed connection attempts, mark that backend down for 10 seconds
- `proxy_next_upstream` — defines which failure types count, and tells NGINX to retry the request on a different backend rather than just failing it

**Test procedure:**
```bash
# On Web-1 — simulate an outage
rc-service nginx stop
```
```bash
# From client, repeatedly
curl http://10.12.1.2
```
**Confirmed result:** first request(s) may be slow/fail while NGINX detects the outage; subsequent requests consistently return only Web-2. Restarting Web-1 (`rc-service nginx start`) and waiting past `fail_timeout` restored round-robin rotation.

**Log evidence confirming the mechanism actually triggered** (`/var/log/nginx/error.log` on NGINX node):
```
connect() failed (111: Connection refused) while connecting to upstream ... upstream: "http://10.10.10.2:80/"
upstream server temporarily disabled while connecting to upstream ... upstream: "http://10.10.10.2:80/"
```

---

## Part 2 — HTTPS Termination (SSL Offload)

Client connects to NGINX over HTTPS; NGINX talks plain HTTP to the backends internally — backends never need certificates at all.

**Generate a self-signed cert on the NGINX node:**
```bash
apk add openssl   # if not already installed
mkdir -p /etc/nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/nginx-selfsigned.key \
  -out /etc/nginx/ssl/nginx-selfsigned.crt \
  -subj "/C=PH/ST=Cordillera/O=NetworkLab/CN=nginx-lb"
```

**Full working config** (both HTTP and HTTPS server blocks, `/etc/nginx/http.d/default.conf`):
```nginx
upstream backend_pool {
    server 10.10.10.2:80 max_fails=2 fail_timeout=10s;
    server 10.10.10.3:80 max_fails=2 fail_timeout=10s;
}

server {
    listen 80;
    server_name _;
    location / {
        proxy_pass http://backend_pool;
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

server {
    listen 443 ssl;
    server_name _;

    ssl_certificate /etc/nginx/ssl/nginx-selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx-selfsigned.key;

    location / {
        proxy_pass http://backend_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

```bash
nginx -t
nginx -s reload
```

**FortiGate policy update** — the existing `Client-to-NGINX` policy only permitted HTTP/PING; add HTTPS:
```
config firewall policy
    edit 3
        set service "HTTP" "HTTPS" "PING"
    next
end
```

**Test:**
```bash
curl -k https://10.12.1.2
```
(`-k` skips cert trust validation — expected/correct for a self-signed cert. A real deployment would use a CA-issued certificate instead, e.g. Let's Encrypt.)

**Confirmed result:** clean round-robin alternation between Web-1 and Web-2, now over an encrypted connection to NGINX.

---

## Part 3 — The Persistence Troubleshooting Saga

This took up most of the session and is worth documenting in detail, since it wasn't one bug but a chain of several distinct issues, each masking the next.

### Issue A — `rc-update` shows nothing for `networking`

**Symptom:** `rc-update show | grep networking` returned empty, even after apparently running `rc-update add networking default` earlier — meaning networking config was never actually applied automatically at boot.

**Root cause:** the `networking` OpenRC service script itself existed (`/etc/init.d/networking` was present), but the package that makes it *functional* — **`ifupdown-ng`** — was missing. Alpine's minimal install doesn't include it by default; `/etc/network/interfaces` is just a config file, something has to actually read and apply it.

**Fix:**
```bash
apk add ifupdown-ng
rc-update add networking default
rc-update show default | grep networking
```
Confirm via the actual symlink, not just command output:
```bash
ls -la /etc/runlevels/default/ | grep -i network
```

### Issue B — Config file "saved" but vanished after reboot (root cause: `tmpfs`)

**Symptom:** Even after fixing Issue A, `/etc/network/interfaces` still didn't survive a reboot on one node.

**Diagnosis:**
```bash
mount | grep " / "
# tmpfs on / type tmpfs (rw,relatime,mode=755,inode64)
```

**Root cause:** the node's root filesystem was mounted in **RAM** (`tmpfs`), not on the actual virtual disk — meaning it was booting into Alpine's live/diskless ISO environment every time, not the installed system. *Everything* configured in that session was wiped on every reboot, not just the network config — this explained why the hostname also kept resetting to `localhost`.

**Confirmed by checking the actual disk file size on the EVE-NG host:**
```bash
ls -la /opt/unetlab/addons/qemu/linux-alpine-client/virtioa.qcow2
# 196736 bytes — essentially empty, just a qcow2 header, no real filesystem
```
A properly-installed Alpine system should be well over 100MB. This meant `setup-alpine`'s disk-install step had silently never completed for this cloned node.

**Fix — redo the disk install properly:**
```bash
setup-alpine
# Disk step: type the correct device name (confirm first with `lsblk` if available)
# When asked sys/data/none — MUST type "sys" explicitly, don't let it default past this
# Wait for actual file-copy activity, not just prompt completion
```
**Verify before powering off:**
```bash
mount | grep " / "
# should now show /dev/vda3 on / type ext4 (rw,relatime) — real disk, not tmpfs
```

### Issue C — `syslinux(no such package)` during disk install

**Symptom:** Retrying the Alpine installer's disk step threw:
```
unable to select packages:
 syslinux(no such package):
   required by: world(syslinux)
```

**Root cause:** the installer's mirror/URL step had been skipped, so `apk` had no repository configured to fetch `syslinux` (the bootloader package needed to make the installed disk actually bootable).

**Fix — don't skip the mirror step.** If already past that point, finish the disk step manually once connectivity + repos are confirmed:
```bash
ping -c 3 8.8.8.8
cat /etc/apk/repositories   # populate if empty, same as earlier sessions
apk update
setup-disk -m sys /dev/vda
```

### Issue D — `/etc/nginx/http.d/` directory doesn't exist on a freshly-installed node

**Symptom:** `cat: can't create /etc/nginx/http.d/default.conf: nonexistent directory` on Web-1 after a fresh `apk add nginx`.

**Root cause:** `find / -iname "nginx.conf"` returned nothing at all — the package wasn't actually installed (`apk info -e nginx` came back empty), despite `apk list nginx` showing it as available in the repo. The install had silently not completed, likely due to a connectivity hiccup during the `apk add` that wasn't immediately obvious.

**Fix:**
```bash
apk info -e nginx        # confirms true install state — empty output = not installed
apk add nginx            # reinstall
apk info -e nginx        # confirm it now shows installed
ls -la /etc/nginx/        # confirm config directories now exist
```

### Issue E — `vim` package unavailable

**Symptom:** `apk add vim` → no such package.

**Diagnosis:** `apk search vim` showed only plugin-style packages (`nginx-vim`, `cmake-vim`, etc. — syntax-highlighting add-ons for other tools), no standalone `vim` editor in this repo snapshot.

**Resolution:** not worth chasing — used Alpine's built-in BusyBox `vi` instead (already present, no install needed). Purely cosmetic/convenience issue, not a blocker.

---

## Persistent Configuration — Reference Template

For every Alpine node in this lab going forward, this is the confirmed-working baseline:

**1. Packages (install once per node, while temporarily internet-connected):**
```bash
apk update
apk add nginx ifupdown-ng
# add openssl + curl on the NGINX/load-balancer node specifically
```

**2. `/etc/network/interfaces`:**
```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address <node-ip>
    netmask 255.255.255.0
    gateway <gateway-ip>
```
(Omit the `gateway` line on any second interface that isn't the default route — e.g. NGINX's backend-facing `eth1`.)

```bash
rc-update add networking default
rc-service networking restart
```

**3. Hostname:**
```bash
setup-hostname <node-name>
```
(This handles both `/etc/hostname` and the live hostname in one step — equivalent manual version is `hostname <name>` + `echo "<name>" > /etc/hostname`.)

**4. Confirm nginx also starts at boot** (separate from `networking`):
```bash
rc-update add nginx default
```

**5. Always verify with a full reboot, not just a service restart:**
```bash
reboot
```
Then check:
```bash
mount | grep " / "              # must show real disk (ext4), never tmpfs
cat /etc/network/interfaces      # must still contain your config
ip addr show eth0                # must show your static IP
hostname                          # must show your custom name
```

---

## Quick Reference — Diagnostic Commands From This Session

```bash
# Confirm real disk vs live/tmpfs boot — the single most important check
mount | grep " / "

# Confirm a package is genuinely installed (more reliable than apk list)
apk info -e <package>

# Confirm boot-time service registration (check the actual symlink, not just command output)
ls -la /etc/runlevels/default/ | grep -i <service>

# Check disk file size directly from the EVE-NG host (catches incomplete installs early)
ls -la /opt/unetlab/addons/qemu/<node-folder>/virtioa.qcow2

# Finish an Alpine disk install manually if setup-alpine's wizard didn't complete it
setup-disk -m sys /dev/vda

# NGINX health-check failure evidence
tail -f /var/log/nginx/error.log
```

---

## The Pattern Worth Remembering

Nearly every issue in Part 3 traced back to the same root behavior: **a step that looked like it completed actually hadn't**, and the failure was silent rather than a clear error — `setup-alpine`'s disk step returning to a prompt without having written anything, `apk add nginx` not actually installing despite no obvious fatal error, `rc-update add` appearing to run without the symlink existing. 

**Takeaway:** after any install/config step in Alpine (or really any Linux system), verify the *result* directly — check the actual file size, the actual symlink, the actual mount type, the actual `apk info -e` status — rather than trusting that a command returning to a prompt means it succeeded. This is especially true for cloned VM disks, where a clone taken mid-setup can silently carry over an incomplete state that looks fine until you specifically check for it.
