# Email Alerting on State Change — Reference

**Date:** August 29, 2026
**Environment:** Alpine Linux Monitor/DHCP node (EVE-NG lab)
**Objective:** Extend the existing dynamic health-check monitor so it emails an alert only when a host's status actually changes, instead of requiring manual log review.

---

## Why This Was Needed

The health-check script built earlier (Part 3 of the previous session) logs every checked host's status every minute, every time — genuinely useful as a historical record, but not something anyone will proactively watch. **Alerting** means the system tells you when something changes, rather than you having to go looking. This session added that layer on top of the existing monitor, using email as the notification channel.

---

## Part 1 — msmtp Setup (Monitor Node Only)

**Important scope note:** `msmtp` only needs to be installed on the **Monitor node** — it's the only machine running the health-check script, so it's the only one that ever needs to send mail. Web-1, Web-2, NGINX-1/2, and Client-1 don't need it.

**Install:**
```bash
apk add msmtp ca-certificates
```

### Issue — `msmtp-mta` package doesn't exist

**Symptom:**
```
ERROR: unable to select packages:
  msmtp-mta (no such package):
    required by: world[msmtp-mta]
```

**Root cause:** this Alpine repo snapshot doesn't ship an `msmtp-mta` sub-package at all. Confirmed via:
```bash
apk search msmtp
# msmtp-1.8.32-r1, msmtp-doc, msmtp-lang, msmtp-openrc, msmtp-vim — no msmtp-mta
```

**Fix:** just install `msmtp` alone — it's sufficient for calling `msmtp` directly from a script (which is what the health-check script does). The `-mta` variant that exists on some distros is specifically for making msmtp act as a drop-in `sendmail` replacement for other programs to call transparently — not needed here.

### Gmail SMTP Configuration

Confirmed current as of 2026 (no changes from prior knowledge):
- Server: `smtp.gmail.com`
- Port: `587` (STARTTLS) — recommended default; `465` (SSL) also works as a fallback
- **App Password is mandatory** if 2-Step Verification is enabled (which Google effectively expects) — a raw account password will not work against Gmail's SMTP server
- Free Gmail accounts: 500 emails/day limit — irrelevant for this use case (alerts only fire on state changes, not continuously)

**Generate an App Password:** `myaccount.google.com/apppasswords` (requires 2-Step Verification already enabled on the account) → create one, name it something identifiable like "EVE-NG Lab Alerts" → copy the 16-character password immediately, it's shown only once.

**Config file** (`/etc/msmtprc`):
```
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        /var/log/msmtp.log

account        gmail
host           smtp.gmail.com
port           587
from           your-email@gmail.com
user           your-email@gmail.com
password       your-16-char-app-password

account default : gmail
```
```bash
chmod 600 /etc/msmtprc
```
(`chmod 600` matters — this file contains a plaintext credential.)

**Test send:**
```bash
echo -e "Subject: Test from EVE-NG lab\n\nThis is a test alert." | msmtp your-email@gmail.com
```
**Confirmed working** — test email received successfully. If it ever fails, check `/var/log/msmtp.log` for the specific SMTP error.

---

## Part 2 — State-Change Detection & Alerting Logic

**Mechanism:** the script now keeps a small state file per hostname (`/var/lib/monitor/state/<hostname>`), storing just its last known status string (`UP+WEB`, `UP`, or `DOWN`). Each run compares the freshly-computed current status against what's stored. An email + a dedicated alert-log entry only fire when the two differ — the routine (unchanged) case does nothing extra beyond the normal per-minute log line it already wrote.

**Guard against false first-run alerts:** if there's no previous state file yet for a host (first time it's ever been seen), the script skips alerting on that first check — otherwise every host would trigger a spurious "alert" the very first time the monitor sees it, even though nothing actually changed.

---

## Final Working Script

`/opt/monitor/healthcheck.sh`:
```bash
#!/bin/sh
LOG=/var/log/healthcheck.log
ALERTLOG=/var/log/alerts.log
LEASES=/var/lib/misc/dnsmasq.leases
STATEDIR=/var/lib/monitor/state
ALERT_EMAIL="your-email@gmail.com"

mkdir -p "$STATEDIR"

while read expiry mac ip hostname clientid; do
    [ -z "$ip" ] && continue

    if ping -c 1 -W 2 "$ip" >/dev/null 2>&1; then
        if curl -s --max-time 2 -o /dev/null "http://$ip"; then
            CURRENT="UP+WEB"
        else
            CURRENT="UP"
        fi
    else
        CURRENT="DOWN"
    fi

    echo "$(date '+%Y-%m-%d %H:%M:%S') $CURRENT $hostname ($ip)" >> $LOG

    STATEFILE="$STATEDIR/$hostname"
    PREVIOUS=""
    [ -f "$STATEFILE" ] && PREVIOUS=$(cat "$STATEFILE")

    if [ "$CURRENT" != "$PREVIOUS" ] && [ -n "$PREVIOUS" ]; then
        TIMESTAMP="$(date '+%Y-%m-%d %H:%M:%S')"
        MSG="$TIMESTAMP ALERT: $hostname ($ip) changed from $PREVIOUS to $CURRENT"
        echo "$MSG" >> $ALERTLOG

        if [ "$CURRENT" = "DOWN" ] || { [ "$CURRENT" = "UP" ] && [ "$PREVIOUS" = "UP+WEB" ]; }; then
            SUBJECT="[LAB ALERT] $hostname is DOWN or degraded"
            HEADLINE="Something needs attention."
        elif [ "$CURRENT" = "UP+WEB" ]; then
            SUBJECT="[LAB RECOVERED] $hostname is back to normal"
            HEADLINE="Good news -- this host has recovered."
        else
            SUBJECT="[LAB ALERT] $hostname state changed"
            HEADLINE="A state change was detected."
        fi

        BODY="$HEADLINE

Host:       $hostname
IP:         $ip
Previous:   $PREVIOUS
Current:    $CURRENT
Time:       $TIMESTAMP

--
Sent automatically by the EVE-NG lab health-check monitor."

        printf "Subject: %s\n\n%s\n" "$SUBJECT" "$BODY" | msmtp "$ALERT_EMAIL"
    fi

    echo "$CURRENT" > "$STATEFILE"

done < "$LEASES"
```
```bash
chmod +x /opt/monitor/healthcheck.sh
```
**Replace `your-email@gmail.com` with the real destination address before use.**

**Subject line logic:**
- `[LAB ALERT] <host> is DOWN or degraded` — fires when going fully DOWN, or dropping from `UP+WEB` to plain `UP` (service crashed but box still alive)
- `[LAB RECOVERED] <host> is back to normal` — fires when a host returns to `UP+WEB`
- `[LAB ALERT] <host> state changed` — fallback for any other transition

No changes needed to cron — it already points at this same script path (`/etc/crontabs/root`), so overwriting the file's contents is sufficient; the schedule picks up the new logic on its next run automatically.

---

## Testing

```bash
# On Web-1 — trigger a degradation
rc-service nginx stop
```
```bash
# On Monitor — run immediately rather than waiting for the next cron minute
/opt/monitor/healthcheck.sh
cat /var/log/alerts.log
```
**Confirmed:** email received with subject `[LAB ALERT] WEB1 is DOWN or degraded` (or similar depending on exact transition), formatted body with host/IP/previous/current/timestamp clearly laid out.

```bash
# On Web-1 — recover
rc-service nginx start
```
```bash
# On Monitor
/opt/monitor/healthcheck.sh
```
**Confirmed:** recovery email received with `[LAB RECOVERED]` subject.

**Also confirmed:** the automated cron-driven run (not manually triggered) correctly picked up a state change and sent the alert on its own, proving the full pipeline works unattended — which is the actual point of building this.

---

## Quick Reference

```bash
# Check msmtp's own send log if an alert email doesn't arrive
cat /var/log/msmtp.log

# Check the dedicated alert history (only state-change events, not routine checks)
cat /var/log/alerts.log

# Check current stored state for any host
cat /var/lib/monitor/state/<hostname>

# Force an immediate check without waiting for cron
/opt/monitor/healthcheck.sh
```

---

## Key Takeaway

The distinction between **monitoring** and **alerting** is exactly what got built here: the original script already told the truth every minute, but a log nobody reads isn't actionable. Adding a per-host state file to detect *transitions* — rather than re-evaluating and re-announcing the same unchanged status every cycle — is the standard pattern real monitoring systems (Zabbix, Prometheus Alertmanager, Nagios, etc.) use internally, just implemented here at the smallest possible scale with a plain shell script and a text file per host instead of a database.
