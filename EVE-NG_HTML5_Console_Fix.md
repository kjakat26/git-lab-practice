# EVE-NG HTML5 Console Not Responding — Troubleshooting Reference

**Date resolved:** August 19, 2026
**Environment:** EVE-NG 6.2.0-4, VMware Workstation Pro, Windows 11 host
**Symptom:** Native console works fine (accessible with admin account). HTML5 console UI loads, login prompt appears, but nothing happens after entering credentials — no error, no console session.

---

## Root Cause

The `eve-ng-pro-guacamole` package (version 6.0.1-37) was marked as **installed** in `dpkg`, but its actual deployed files were missing from disk — specifically:

- `/opt/unetlab/html5/guacamole.war` (the symlink target) did not exist
- `/var/lib/tomcat9/webapps/guacamole/` (the extracted webapp directory) did not exist

This caused Tomcat to fail deploying the Guacamole web application on every startup, while `guacd` (the proxy daemon) and `tomcat9` (the servlet container) both continued running normally — making the failure invisible from a simple `systemctl status` check. The HTML5 console frontend loaded fine (served by EVE-NG's main app), but had nothing to actually connect to on the backend, so login silently did nothing.

**Likely cause of the missing files:** package files removed or corrupted outside of `apt` (manual cleanup, disk issue, or incomplete initial install) while `dpkg`'s records were untouched.

---

## Diagnostic Procedure (in order)

### 1. Rule out node/browser-level causes first
Before touching the backend, confirm:
- The node is actually **started** (not just added to the topology)
- No **pop-up blocker** is silently blocking the HTML5 console tab
- Try a different browser (**Chrome** is most reliable for EVE-NG's Guacamole-based console)
- Console preference in EVE-NG user settings is actually set to **HTML5**, not Native

If HTML5 UI loads and shows a login screen but nothing happens after login (as opposed to nothing happening at all), move to backend diagnostics.

### 2. Try the quick fixes first
```bash
# Logout/login from the EVE-NG web GUI — resolves a surprising number of cases
```
```bash
# Generic EVE-NG permissions fix (low-risk, worth running regardless)
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

### 3. Check if this is the "known" Guacamole DB corruption issue
Older EVE-NG documentation points to a corrupted Guacamole database as the most common cause of HTML5 console failures. Check for the historical fix path:
```bash
find / -iname "*guacamole*.sql" 2>/dev/null
```
> **Note:** On EVE-NG 6.x, the old fix path `/opt/unetlab/schema/guacamole-update.sql` no longer exists. The actual schema files found were:
> ```
> /opt/unetlab/schema/guacamole-002-update.sql
> /opt/unetlab/schema/guacamole-1.0-002-create-schema.sql
> /opt/unetlab/schema/guacamole-001-create-schema.sql
> /opt/unetlab/schema/guacamole-003-create-admin-user.sql
> /opt/unetlab/schema/guacamole-1.0-001-create-schema.sql
> ```
> Don't assume the classic fix command applies as-is on newer versions — verify actual file paths first.

### 4. Verify the database itself is intact
```bash
mysql -u root --password=eve-ng -e "SHOW DATABASES;"
```
Confirm `guacdb` is present, then check its tables:
```bash
mysql -u root --password=eve-ng -e "USE guacdb; SHOW TABLES;"
```
**Result in this case:** All 23 expected `guacamole_*` tables were present and intact. This ruled out database corruption entirely — the problem was elsewhere.

### 5. Check that the relevant services are actually running
```bash
systemctl list-units --type=service --all | grep -iE "tomcat|guacamole"
```
> **Note:** Don't assume the service name — `tomcat8` was expected based on older documentation, but this version uses `tomcat9`. Always list actual service names rather than guessing.

**Result in this case:** Both `guacd.service` and `tomcat9.service` showed `active (running)` — services being "up" does not mean the deployed application inside them is functional.

### 6. Check the actual application deployment logs — this is where the real error was found
```bash
find / -iname "catalina.out" 2>/dev/null
tail -f /var/log/tomcat9/catalina.out
# (then reproduce the issue in the browser while this is running)
```

**Key log evidence:**
```
[crit] Caused by: java.lang.IllegalArgumentException: The main resource set specified
       [/var/lib/tomcat9/webapps/guacamole] is not valid
[crit]     at org.apache.catalina.webresources.StandardRoot.createMainResourceSet(...)
...
[info] Deployment of web application directory [/var/lib/tomcat9/webapps/ROOT]
       has finished in [218] ms
[info] Deployment of deployment descriptor [/etc/tomcat9/Catalina/localhost/guacamole.xml]
       has finished in [4] ms
```
A deployment "finishing" in 4ms is not a healthy sign — Tomcat gave up almost instantly because the resource path didn't exist, rather than actually starting the app.

### 7. Confirm the missing files directly
```bash
ls -la /var/lib/tomcat9/webapps/ | grep -i guacamole
find / -iname "guacamole*.war" 2>/dev/null
cat /etc/tomcat9/Catalina/localhost/guacamole.xml
```
**Result:**
```
lrwxrwxrwx 1 root root 32 Aug 19 14:22 guacamole.war -> /opt/unetlab/html5/guacamole.war
```
The `.war` in `webapps/` was only a **symlink**, pointing to `/opt/unetlab/html5/guacamole.war` — which turned out not to exist:
```bash
ls -la /opt/unetlab/html5/
# ls: cannot access '/opt/unetlab/html5/': No such file or directory
```
**This confirmed the root cause:** the entire `/opt/unetlab/html5/` directory (containing the actual Guacamole application files) was missing from disk, even though the package was marked installed.

### 8. Confirm the package is "installed" but broken
```bash
apt list --installed 2>/dev/null | grep -i eve
dpkg -l | grep -i guacamole
```
**Result:**
```
ii  eve-ng-pro-guacamole    6.0.1-37    amd64    Guacamole for UNetLab/EVE-NG
```
`ii` = apt believes it's fully installed. This mismatch between "apt thinks it's fine" and "files are missing" is the core of the bug.

---

## Resolution

### Step 1 — Confirm the EVE-NG repo is reachable
```bash
cat /etc/apt/sources.list.d/*eve* 2>/dev/null
# deb [arch=amd64] http://www.eve-ng.net/jammy jammy main
```

### Step 2 — Refresh package lists
```bash
apt-get update
```

### Step 3 — Force reinstall the broken package
```bash
apt-get install --reinstall eve-ng-pro-guacamole
```

**Output confirming success:**
```
The following packages will be upgraded:
  eve-ng-pro-guacamole
Get:1 http://www.eve-ng.net/jammy jammy/main amd64 eve-ng-pro-guacamole amd64 6.5.0-16 [36.3 MB]
Checking MySQL... done
Checking GUACDB Presence... done
Unpacking eve-ng-pro-guacamole (6.5.0-16) over (6.0.1-37) ...
Setting up eve-ng-pro-guacamole (6.5.0-16) ...
Enable services at boot... done
Staging Guacamole webapp... done
Starting Tomcat... done
Starting guacamole... done
```

Note this pulled a **newer package version** (6.5.0-16, up from the broken 6.0.1-37) directly from EVE-NG's repo, and this version deploys differently — it stages a real `.war` file directly into Tomcat's webapps folder rather than relying on the old `/opt/unetlab/html5/` symlink path. That old path being missing was a non-issue after this reinstall; it's simply not used by the newer package version.

### Step 4 — Verify the fix
```bash
ls -la /var/lib/tomcat9/webapps/ | grep -i guacamole
```
**Result:**
```
drwxr-x--- 10 tomcat tomcat     4096 Aug 19 15:44 guacamole
-rw-r--r--  1 root   root   15641879 Jun 19  2025 guacamole.war
```
A real extracted `guacamole/` directory now exists (not a broken symlink), and the `.war` is a real ~15MB file.

### Step 5 — Test
Reload the EVE-NG web GUI → right-click a running node → Console (HTML5). Connected successfully.

---

## Quick Reference — If This Happens Again

1. Check `catalina.out` first — it will show the exact deployment error immediately (`tail -f /var/log/tomcat9/catalina.out` while reproducing the issue in browser)
2. If you see `IllegalArgumentException: The main resource set specified [...] is not valid`, the Guacamole webapp files are missing/corrupted — this is a package deployment issue, not a database issue
3. Don't assume database corruption without checking `SHOW TABLES` in `guacdb` first — it wastes time chasing the wrong fix
4. The fix is almost always: `apt-get update && apt-get install --reinstall eve-ng-pro-guacamole`
5. Service names change between EVE-NG versions (`tomcat8` vs `tomcat9`) — always confirm with `systemctl list-units --all | grep -i tomcat` rather than assuming

---

## Key Diagnostic Commands (Copy-Paste Block)

```bash
# 1. Check services
systemctl list-units --type=service --all | grep -iE "tomcat|guacamole"

# 2. Check database integrity
mysql -u root --password=eve-ng -e "USE guacdb; SHOW TABLES;"

# 3. Find and tail the deployment log (run while reproducing the issue)
find / -iname "catalina.out" 2>/dev/null
tail -f /var/log/tomcat9/catalina.out

# 4. Check package status
dpkg -l | grep -i guacamole

# 5. Reinstall if broken
apt-get update
apt-get install --reinstall eve-ng-pro-guacamole
systemctl restart tomcat9
```
