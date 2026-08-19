# Linux Service Troubleshooting — Command Reference

A general-purpose diagnostic methodology, using the EVE-NG HTML5 console case as the working example. The pattern here applies to almost any Linux app built on **systemd services + a database + a web server** — Guacamole, GitLab, Nextcloud, Zabbix, PiHole, etc. all follow the same basic shape, so the same investigation order works even when the specific commands differ.

---

## The General Diagnostic Order (memorize this sequence, not just the commands)

1. **Is the service running at all?** (`systemctl`)
2. **Is the data layer intact?** (database tables, config files)
3. **What does the application itself say when it fails?** (application logs — this is usually where the *real* answer is)
4. **Are the actual files the app needs present on disk?** (not just "is it installed" — installed and present are different things)
5. **Reinstall/redeploy only once you know *why*** — reinstalling blindly as a first step wastes time and can mask the real cause.

This order matters because steps 1 and 4 can both report "fine" while the app is still broken — a service can be `active (running)` while serving nothing useful, and a package can show as `installed` while its files are missing. Only the logs (step 3) tell you the truth.

---

## Commands Used, Explained

### 1. `systemctl status <service>` / `systemctl list-units`

```bash
systemctl status guacd tomcat8
systemctl list-units --type=service --all | grep -iE "tomcat|guacamole"
```

**What it does:** Asks `systemd` (the process manager that starts/stops/monitors services on modern Linux) whether a service is currently running, and shows its recent log lines.

**Why we used `list-units --all` instead of guessing the name:** `systemctl status <name>` fails silently-ish (`could not be found`) if you guess wrong — and service names change between software versions (we expected `tomcat8`, the actual service was `tomcat9`). `list-units --all | grep -i <keyword>` searches *every* registered service, running or not, so you find the real name instead of assuming.

**Reusable for other apps:** Always start here, and always search broadly by keyword rather than assuming the exact service name — e.g. `systemctl list-units --all | grep -i nginx`, `| grep -i mysql`, `| grep -i docker`.

**The trap:** "active (running)" only means the process is alive — it does NOT mean the application inside it is functioning correctly. Don't stop your investigation here just because status looks green.

---

### 2. `mysql -u root --password=<pw> -e "<query>"`

```bash
mysql -u root --password=eve-ng -e "SHOW DATABASES;"
mysql -u root --password=eve-ng -e "USE guacdb; SHOW TABLES;"
```

**What it does:** Connects to the MySQL/MariaDB server as root and runs a query non-interactively (`-e` = execute and exit, instead of dropping into the `mysql>` prompt).

**Why:** `SHOW DATABASES` confirms the database exists and MySQL is reachable at all with those credentials. `SHOW TABLES` (after `USE <db>`) confirms the schema is actually complete — a common failure mode for many self-hosted apps is a partially-applied database migration, leaving some tables missing after a bad upgrade.

**Reusable for other apps:** Any app with a MySQL/PostgreSQL backend (Nextcloud, GitLab, Zabbix, WordPress) — swap `mysql` for `psql -U postgres -c "<query>"` on Postgres systems. The logic is identical: confirm the DB exists, then confirm its structure is complete before assuming corruption.

**What this step told us:** All 23 expected tables were present — this *ruled out* the popular "corrupted database" theory that older EVE-NG forums point to, saving us from chasing the wrong fix.

---

### 3. `find / -iname "<pattern>" 2>/dev/null`

```bash
find / -iname "*guacamole*.sql" 2>/dev/null
find / -iname "guacamole*.war" 2>/dev/null
find / -iname "catalina.out" 2>/dev/null
```

**What it does:** Searches the entire filesystem (`/`) for files matching a name pattern, case-insensitively (`-iname`). `2>/dev/null` discards "permission denied" error spam from directories you can't read, so only real matches show.

**Why we used this repeatedly:** Documentation (even official FAQs) often references file paths that shift between software versions. Rather than trusting a remembered/documented path, searching the live filesystem finds where things *actually* are on this specific install.

**Reusable for other apps:** This is your go-to whenever a guide says "edit file X" and X isn't where the guide says — `find / -iname "<filename>" 2>/dev/null` locates it (or proves it's genuinely missing, which is itself useful information, as it was here).

**Caveat:** Searching `/` on a large disk can be slow. If you know the rough location (e.g., `/opt`, `/var`, `/etc`), scope the search there instead: `find /opt -iname "guacamole*.war"`.

---

### 4. `tail -f <logfile>`

```bash
tail -f /var/log/tomcat9/catalina.out
```

**What it does:** Prints the last lines of a file and then keeps watching it, printing new lines as they're written — like watching a log update live.

**Why this was the turning point:** Service status and database checks both looked fine, but the *application log* showed the actual Java exception explaining exactly why deployment failed. This is almost always true in troubleshooting: infrastructure-level checks (is it running? does the DB exist?) tell you the app *could* work, but only the application's own log tells you why it *doesn't*.

**Reusable for other apps:** Nearly universal technique. Find the app's log (`find / -iname "*.log" 2>/dev/null` or check `/var/log/<appname>/`), then `tail -f` it while reproducing the problem in real time. Watching it live (rather than reading a static snapshot after the fact) makes it much easier to spot which lines correspond to your specific action.

---

### 5. `grep -B <n> "<pattern>" <file> | tail -<n>`

```bash
grep -B 60 "createMainResourceSet" /var/log/tomcat9/catalina.out | tail -80
```

**What it does:** `grep -B 60` finds every line matching the pattern and includes the 60 lines *before* it (Before context) — useful because Java stack traces print the root cause at the bottom, with the surrounding context above it. Piping to `tail -80` trims that down to just the most recent (last) occurrence, since the same error may have logged multiple times across restart attempts.

**Reusable for other apps:** Whenever `tail -f` shows you an error but scrolls past the full context, use `grep -B <n> "<distinctive error text>" <logfile>` to pull the complete surrounding trace instead of scrolling back manually.

---

### 6. `ls -la <path>`

```bash
ls -la /var/lib/tomcat9/webapps/ | grep -i guacamole
ls -la /opt/unetlab/html5/
```

**What it does:** Lists directory contents with full details (`-l`) including hidden files (`-a`) — file sizes, permissions, ownership, and crucially, whether something is a **symlink** (shown as `name -> target`).

**Why this mattered here:** The `-la` flag revealed that `guacamole.war` in the webapps folder wasn't a real file — it was a symlink pointing elsewhere. A plain `ls` (without `-la`) might not have made this as obvious. Following that symlink's target with a second `ls -la` on the target path is what revealed the target directory didn't exist at all.

**Reusable for other apps:** Any time a file "exists" but an app can't use it, check whether it's actually a symlink to something missing — this is a common failure pattern in packaged Linux software that uses symlinks to manage versions.

---

### 7. `apt list --installed` / `dpkg -l`

```bash
apt list --installed 2>/dev/null | grep -i eve
dpkg -l | grep -i guacamole
```

**What it does:** Both list installed packages on Debian/Ubuntu-based systems (`apt list --installed` is the modern, human-friendlier version; `dpkg -l` is the older, more detailed one — its `ii` status code means "installed and configured correctly," according to apt's own bookkeeping).

**The critical insight this revealed:** `dpkg` said `ii` (fully installed) even though we'd just proven the actual files were missing from disk. **apt's database and the real filesystem state can disagree** — apt only tracks what it *believes* it installed, not what's *actually still there*. This is exactly the kind of drift that causes "but it says it's installed!" confusion.

**Reusable for other apps:** Whenever a package-managed app is misbehaving and you've ruled out config/data issues, check `dpkg -l | grep -i <appname>` to get the exact package name and version — you'll need this for the reinstall step regardless of what broke.

---

### 8. `cat /etc/apt/sources.list.d/*.list` (or similar)

```bash
cat /etc/apt/sources.list.d/*eve* 2>/dev/null
```

**What it does:** Shows the contents of apt's repository source files — where apt actually downloads packages from for this particular software vendor.

**Why:** Before trying to reinstall a package, confirm apt actually has a source configured for it. If this file didn't exist or the URL was dead, `apt-get install --reinstall` would fail with a clear "unable to locate package" or similar — worth ruling out before troubleshooting further.

**Reusable for other apps:** Any third-party software installed via a custom apt repo (Docker, GitLab, Fortinet's own tools if they ship .deb packages) follows this same pattern — check `/etc/apt/sources.list.d/` for the vendor's repo file.

---

### 9. `apt-get update`

```bash
apt-get update
```

**What it does:** Refreshes apt's local cache of what packages/versions are *available* from all configured repos (it does not install or upgrade anything by itself).

**Why it's step one before any install/reinstall:** Without this, apt may try to reinstall using stale metadata, potentially failing with "unable to fetch" or grabbing an outdated version. Cheap to run, low risk, standard first move before any package operation.

---

### 10. `apt-get install --reinstall <package>`

```bash
apt-get install --reinstall eve-ng-pro-guacamole
```

**What it does:** Forces apt to re-download and re-extract a package's files from the repository, even though apt believes it's already installed — overwriting whatever is (or isn't) currently on disk with a fresh copy.

**Why this was the actual fix:** Since the root cause was "package metadata says installed, but files are physically missing," a normal `apt-get install` would do nothing (apt thinks there's nothing to do). `--reinstall` forces the file extraction to happen again regardless of apt's belief about current state.

**Reusable for other apps:** This is the general fix for "package/app is broken in a way that looks like missing/corrupted files, but the package manager thinks everything is fine" — applicable to any `apt`-installed software. On other package managers: `yum reinstall <package>` (RHEL/CentOS), `dnf reinstall <package>` (Fedora), `pacman -S <package>` (Arch, which reinstalls if already current).

**Bonus outcome to watch for:** In this case, the reinstall pulled a *newer* version than what was originally installed (6.5.0 vs 6.0.1), because apt always installs the latest version available in the repo unless pinned. This can occasionally introduce new behavior — worth knowing that a "reinstall" isn't always byte-for-byte identical to what was there before.

---

## The Transferable Mental Model

When any Linux service-based app "just doesn't work" with no obvious error on the surface:

```
systemctl (is it running?)
      ↓ running, but still broken
database checks (is the data intact?)
      ↓ intact, but still broken
application logs — tail -f the real log while reproducing the issue
      ↓ this tells you the ACTUAL error
find/ls -la — confirm whether the files the error references actually exist
      ↓ confirms root cause
dpkg/apt — confirm what the package manager believes vs. reality
      ↓ 
apt-get install --reinstall — fix the mismatch
```

The two biggest traps this case illustrates:
1. **"Running" ≠ "working."** Both `guacd` and `tomcat9` were happily "active (running)" the entire time the app was broken.
2. **"Installed" ≠ "files present."** `dpkg` said `ii` while the actual application directory didn't exist on disk.

Trust logs and direct filesystem inspection over status flags — status flags only tell you what the tool *believes*, not what's *true*.
