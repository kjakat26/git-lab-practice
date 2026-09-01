# Remote Diagnostics via Restricted SSH — Reference

**Date:** August 31, 2026
**Environment:** Monitor/DHCP node (Alpine), Web-1, Web-2 (EVE-NG lab)
**Objective:** Extend the browser-based diagnostics tool so MTR checks can be executed on a specific remote node (Web-1, Web-2) instead of only locally on the Monitor box — using a hardened, non-root, command-restricted SSH approach.

---

## Part 1 — The Full Request Flow (How a Check Actually Executes)

Before extending the tool, it's worth understanding exactly what happens between a browser click and rendered output — six distinct hops, each a separate process/privilege boundary:

1. **Browser click** — the form's JS `runCheck()` function builds a URL like `/cgi-bin/mtr.cgi?target=10.10.10.2&host=web1` and calls `fetch()` — an async HTTP GET, no page reload.
2. **nginx (`:8080`) receives it** — matches the `location /cgi-bin/` block, and instead of serving a static file, hands the request to a separate process via the **FastCGI protocol** over a Unix socket (`fastcgi_pass unix:/run/fcgiwrap/fcgiwrap.sock`).
3. **fcgiwrap executes the script** — translates the FastCGI request into an actual shell process invocation of `mtr.cgi`, injecting the query string as `$QUERY_STRING` and other CGI environment variables.
4. **The script does the real work** — parses `$QUERY_STRING`, validates the target as IPv4, and (new as of this session) branches on a `host` parameter to decide whether to run `mtr` locally or over SSH to a remote node.
5. **Output flows back up the same chain** — fcgiwrap captures the script's stdout, wraps it as a FastCGI response, nginx repackages it as a normal HTTP response.
6. **Browser renders it** — `await res.text()` reads the body, `outputEl.textContent = text` writes it directly into the page's `<pre>` block.

**Critical detail: the script does NOT run as you, or as `root`, or even as `nginx`.** It runs as whatever user fcgiwrap itself runs as — confirmed via `ps -eo pid,user,comm | grep fcgiwrap` to be a dedicated system account literally named `fcgiwrap` (UID 102, GID 82/`www-data`, shell `/sbin/nologin`, confirmed via `grep fcgiwrap /etc/passwd`). This privilege-separation is deliberate: nginx never executes arbitrary code directly, and fcgiwrap runs as an unprivileged dedicated account so that a bug or compromise in a script is contained, not automatically root-level.

**Checking what a specific binary is allowed to do:**
```bash
ls -la /usr/sbin/mtr-packet      # standard Unix rwx permissions (owner/group/others)
getcap /usr/sbin/mtr-packet      # Linux capabilities — e.g. cap_net_raw=ep, granted via setcap
id <username>                     # UID/GID and group memberships for any account
```
`mtr` specifically needs `cap_net_raw` (normally root-only, for opening raw sockets) — granted narrowly via `setcap cap_net_raw+ep /usr/sbin/mtr-packet` rather than running the whole web stack as root.

---

## Part 2 — Extending to Remote Execution: Design Decision

**Initial idea:** SSH from the Monitor box into a target node (e.g. Web-1) as `root`, using a passphrase-less key (required since a CGI script can't interactively type one).

**Rejected in favor of a hardened design**, because root-to-root key-based SSH means anyone who compromises the `fcgiwrap` account on Mon-box gets a stepping stone to root on every target node.

**Final design — least-privilege, defense in depth:**
- A dedicated, **non-root** user (`diagcheck`) on each target node, with no usable password
- SSH's `command=` restriction in `authorized_keys` — the server **always** runs a forced wrapper script, ignoring whatever the client requested; the client's original request is only available via `$SSH_ORIGINAL_COMMAND` for the wrapper to inspect
- The wrapper independently **re-validates** the target IP with its own regex — even if validation on the Monitor box's side were ever bypassed, the target node still refuses anything malformed
- `no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty` — strips away every other use SSH could be put to besides running the one forced command

**Blast radius if the key were ever stolen:** an attacker could only ever trigger `mtr` against an IP address of their choosing — no shell, no other binaries, no file access. Confirmed by testing directly (see Part 4).

---

## Part 3 — Build Steps (Per Target Node)

**1. Create the restricted user — get the shell right the first time:**
```bash
adduser -D -s /bin/ash diagcheck
mkdir -p /home/diagcheck/.ssh
chmod 700 /home/diagcheck/.ssh
passwd -u diagcheck
```
See Part 5, Issues A and B, for why both the shell choice and the `passwd -u` step are non-negotiable — both were discovered the hard way on Web-1.

**2. Write the command wrapper** (`/usr/local/bin/diag-wrapper.sh`):
```bash
#!/bin/sh
case "$SSH_ORIGINAL_COMMAND" in
    mtr\ -r\ -c\ 4\ *)
        TARGET=$(echo "$SSH_ORIGINAL_COMMAND" | awk '{print $NF}')
        if echo "$TARGET" | grep -qE '^[0-9]{1,3}(\.[0-9]{1,3}){3}$'; then
            mtr -r -c 4 "$TARGET"
        else
            echo "Rejected: invalid target"
        fi
        ;;
    *)
        echo "Rejected: command not permitted"
        ;;
esac
```
```bash
chmod +x /usr/local/bin/diag-wrapper.sh
```

**3. Install prerequisites and grant the capability:**
```bash
apk add mtr openssh libcap
rc-update add sshd default
rc-service sshd start
setcap cap_net_raw+ep /usr/sbin/mtr-packet
```

**4. Authorize the Monitor box's key with the restriction:**
```bash
cat > /home/diagcheck/.ssh/authorized_keys << 'EOF'
command="/usr/local/bin/diag-wrapper.sh",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty ssh-ed25519 AAAA...(Mon box's public key)... root@MON-DHCP
EOF
chown -R diagcheck:diagcheck /home/diagcheck/.ssh
chmod 700 /home/diagcheck/.ssh
chmod 600 /home/diagcheck/.ssh/authorized_keys
```
**Must be a single line** — the `command="..."` prefix and the key itself have to stay on one line or SSH treats the entry as malformed.

---

## Part 4 — Verification Tests (Run From the Monitor Box)

**Allowed path:**
```bash
ssh -i /etc/diag-ssh/diag_key -o StrictHostKeyChecking=accept-new -o UserKnownHostsFile=/etc/diag-ssh/known_hosts diagcheck@<target-ip> "mtr -r -c 4 10.10.10.2"
```
Confirmed: real `mtr` output returned, `HOST:` line correctly shows the target node's hostname.

**Rejected path (arbitrary command):**
```bash
ssh -i /etc/diag-ssh/diag_key -o UserKnownHostsFile=/etc/diag-ssh/known_hosts diagcheck@<target-ip> "whoami; cat /etc/shadow"
```
Confirmed: returns exactly `Rejected: command not permitted`. The SSH connection itself succeeds (valid key, valid account) — it's the forced command that refuses to act on an unrecognized request.

**Injection/piggyback attempt:**
```bash
ssh -i /etc/diag-ssh/diag_key -o UserKnownHostsFile=/etc/diag-ssh/known_hosts diagcheck@<target-ip> "mtr -r -c 4 10.10.10.2; whoami"
```
Confirmed: returns `Rejected: invalid target`. **Why:** the wrapper never re-executes `$SSH_ORIGINAL_COMMAND` as a shell command — it only pattern-matches it and extracts the *last whitespace-separated field* via `awk '{print $NF}'`. For this input, the last field is literally the string `whoami`, which fails the IPv4 regex. The wrapper builds a fresh, hardcoded `mtr -r -c 4 "$TARGET"` command rather than ever handing the original string to `eval`/`sh -c` — that distinction (parse-and-rebuild vs. re-execute) is what actually prevents injection, not the regex alone.

**Realistic worst case identified:** if an attacker crafted input so the *last field* happened to look like a valid IP (e.g. `"mtr -r -c 4 10.10.10.2; ping 8.8.8.8"` → last field `8.8.8.8`), the wrapper would run `mtr -r -c 4 8.8.8.8` — still only ever `mtr`, against an IP of their choosing. The appended command is discarded, never executed.

---

## Part 5 — Issues Hit and Resolved

### Issue A — "User diagcheck not allowed because account is locked"

**Symptom:** SSH silently fell back to a password prompt instead of accepting the key.

**Root cause:** `adduser -D` creates an account with no password set, which Alpine's sshd/PAM treats as **locked**, not as "passwordless login permitted." Confirmed via `/var/log/auth.log`: `User diagcheck not allowed because account is locked`.

**Fix:**
```bash
passwd -u diagcheck
```
Explicitly unlocks the account without setting a real password. Confirm via `cat /etc/shadow | grep diagcheck` — a `!` at the start of the password field indicates locked; it should be gone after the fix.

### Issue B — "This account is not available" despite successful key auth

**Symptom:** After fixing Issue A, SSH no longer prompted for a password, but returned `This account is not available` instead of running the wrapper.

**Root cause:** a forced `command=` in `authorized_keys` is executed **through the user's login shell** — effectively `<shell> -c "<wrapper>"`. The user was created with shell `/sbin/nologin`, whose entire job is to print that exact message and exit, ignoring any command it's asked to run. `nologin` and `command=` restrictions are fundamentally incompatible.

**Fix:** use a real shell instead:
```bash
adduser -D -s /bin/ash diagcheck   # get this right at creation time
```
This doesn't weaken the security model — the actual restriction was always coming from `command=` plus the wrapper's own validation, never from `nologin`. `ash` just needs to exist as the mechanism that launches the wrapper at all.

### Issue C — Key permission denied when triggered via the browser (but worked manually as root)

**Symptom:** Manual testing from a root shell worked perfectly; the same command via the browser produced nothing, then (after adding `2>&1` for visibility) showed `Identity file /root/.ssh/diag_key not accessible: Permission denied`.

**Root cause:** the script executes as the `fcgiwrap` system account (see Part 1), which has no read access to a key stored under `/root/.ssh/` (`-rw-------`, root-owned).

**Fix:** create a dedicated copy of the key, owned by and readable only by the account that actually needs it:
```bash
mkdir -p /etc/diag-ssh
cp /root/.ssh/diag_key /etc/diag-ssh/diag_key
chown fcgiwrap /etc/diag-ssh/diag_key
chgrp 82 /etc/diag-ssh/diag_key    # see Issue D for why numeric GID was needed
chmod 400 /etc/diag-ssh/diag_key
```
Root's own original key is left untouched and separately secured.

### Issue D — `chown fcgiwrap:fcgiwrap` fails: "unknown user/group"

**Symptom:** the user half resolved fine elsewhere, but the combined `user:group` syntax failed.

**Root cause:** `/etc/passwd` showed `fcgiwrap:x:102:82:...` — UID 102, but **GID 82**, which is not a group literally named `fcgiwrap`. Confirmed via `getent group 82` → `www-data:x:82:fcgiwrap,nginx`. There is no group named `fcgiwrap` at all; fcgiwrap's actual group is `www-data` (shared with nginx).

**Fix:** set ownership and group separately, using the numeric GID to sidestep the name lookup:
```bash
chown fcgiwrap /etc/diag-ssh/diag_key
chgrp 82 /etc/diag-ssh/diag_key
```

### Issue E — "Failed to add the host to the list of known hosts"

**Symptom:** SSH succeeded but printed a warning about being unable to write the known_hosts file.

**Root cause:** `/etc/diag-ssh/known_hosts` didn't exist yet as a file the `fcgiwrap` account could write to — SSH tried to record the newly-trusted host key there (via `StrictHostKeyChecking=accept-new`) and failed silently on the write.

**Fix:**
```bash
touch /etc/diag-ssh/known_hosts
chown fcgiwrap /etc/diag-ssh/known_hosts
chgrp 82 /etc/diag-ssh/known_hosts
```

### Issue F — `sed` "successfully" edited a config, but nothing changed

**Symptom:** ran a `sed -i 's|old|new|'` command against `mtr.cgi` expecting it to update the SSH key path and add missing options; the command returned no error, but the browser test still showed the exact old, unfixed command (`/root/.ssh/diag_key`, missing `StrictHostKeyChecking`/`UserKnownHostsFile`).

**Root cause:** `sed`'s substitute command **fails silently** if the pattern doesn't match the file's actual content exactly (whitespace, quoting, or prior edits can all cause a mismatch) — no error is printed either way, so a non-matching `sed` looks identical to a successful one from the terminal output alone.

**Fix / lesson:** after any `sed -i` edit to a script that matters, always verify with `cat`/`grep` immediately afterward rather than trusting the lack of an error message. When a `sed` pattern is even moderately complex (multi-line inserts, special characters), it's more reliable to just rewrite the whole file with `cat > file << 'EOF' ... EOF` than to debug why a substitution silently didn't apply.

### Issue G — Executable bit lost after deleting and recreating the CGI script

**Symptom:** after deleting and recreating `mtr.cgi` to fix an earlier duplicate-entry mistake, the browser test returned "Permission denied."

**Root cause:** recreating a file with `cat >` (or most editors) does not preserve any previous executable permission — a fresh file defaults to non-executable (`-rw-r--r--`), and fcgiwrap cannot execute a script it doesn't have `+x` on.

**Fix:**
```bash
chmod +x /var/www/monitor/cgi-bin/mtr.cgi
```
**Lesson:** any time a CGI script is deleted/recreated (as opposed to edited in place), re-check and reapply the executable bit — it's not implicitly retained.

---

## Final Working `mtr.cgi`

```bash
#!/bin/sh
echo "Content-type: text/plain"
echo ""
TARGET=$(echo "$QUERY_STRING" | sed -n 's/.*target=\([^&]*\).*/\1/p' | tr -d '\n')
HOST=$(echo "$QUERY_STRING" | sed -n 's/.*host=\([^&]*\).*/\1/p' | tr -d '\n')
if ! echo "$TARGET" | grep -qE '^[0-9]{1,3}(\.[0-9]{1,3}){3}$'; then
    echo "Invalid target. Only IPv4 addresses are accepted."
    exit 0
fi
case "$HOST" in
    local|"")
        mtr -r -c 4 "$TARGET"
        ;;
    web1)
        ssh -i /etc/diag-ssh/diag_key -o ConnectTimeout=5 -o StrictHostKeyChecking=accept-new -o UserKnownHostsFile=/etc/diag-ssh/known_hosts diagcheck@10.99.99.127 "mtr -r -c 4 $TARGET"
        ;;
    web2)
        ssh -i /etc/diag-ssh/diag_key -o ConnectTimeout=5 -o StrictHostKeyChecking=accept-new -o UserKnownHostsFile=/etc/diag-ssh/known_hosts diagcheck@10.99.99.126 "mtr -r -c 4 $TARGET"
        ;;
    *)
        echo "Unknown host selection."
        ;;
esac
```
(`chmod +x` applied, `2>&1` deliberately **not** present in the final version — used only temporarily during Issue C's debugging, then removed to avoid leaking internal paths/errors into browser-facing output.)

`index.html`'s MTR section dropdown now offers **Mon box (local)**, **Web-1**, and **Web-2** — confirmed working end-to-end for all three via the browser, with each remote option's output correctly showing the target node's own hostname in the `HOST:` line.

---

## Quick Reference — Commands From This Session

```bash
# Identify the real execution user of a web-facing process
ps -eo pid,user,comm | grep <process>
grep <username> /etc/passwd

# Check binary-level privileges
getcap /usr/sbin/mtr-packet
setcap cap_net_raw+ep /usr/sbin/mtr-packet

# Check account lock state
cat /etc/shadow | grep <username>    # leading ! = locked
passwd -u <username>                  # unlock without setting a password

# Resolve a GID to its actual group name
getent group <gid>

# Test a restricted SSH command exactly as the real low-privilege execution user would
su -s /bin/sh <service-user> -c "ssh -i <key> ... "

# Always verify a sed -i edit actually applied
grep -n "<expected content>" <file>
```

---

## Key Takeaways

1. **A web-facing script's real execution identity is rarely the account you're logged in as.** Always confirm with `ps -eo user,comm` before assuming permissions will "just work" the same way they did in a manual root test.
2. **`command=` forced SSH restrictions require a real shell.** `/sbin/nologin` silently defeats them — this is a subtle enough interaction that it's worth remembering as a rule, not re-deriving from first principles next time.
3. **`sed -i` fails silently on a non-match.** Never trust "no error" as proof an edit applied — verify the actual file content, every time, especially on anything security- or access-relevant.
4. **Recreating a file loses its executable bit.** Editing in place preserves permissions; deleting and recreating does not.
5. **The real security boundary in this design is the server-side wrapper's re-validation, not the client's request.** Understanding that the wrapper parses-and-rebuilds a command rather than re-executing the original string is what makes the injection test's result (`Rejected: invalid target`, not arbitrary code execution) make sense rather than feel like luck.
