# NETWORK LAB PRACTICAL GUIDE - HANDS-ON SECURITY PROJECTS
## Building Real Interview Stories Through Lab Work

---

## PROJECT 1: VULNERABILITY SCANNING & REMEDIATION
### Tool: OpenVAS (Free) or Nessus Essentials (Free Tier)

### Why This Matters for Interviews:
- Shows you can run actual security tools (not just theory)
- Demonstrates remediation process (scan → fix → re-scan → verify)
- Real findings = real stories ("I found X vulnerability, here's how I fixed it")
- Proves you understand how to improve infrastructure security

### STEP 1: INSTALL OPENVAS (Free, Open Source)

**On Linux (Ubuntu/Debian):**
```bash
# Add OpenVAS repository
sudo add-apt-repository ppa:manicho/openvas
sudo apt-get update
sudo apt-get install openvas

# Start OpenVAS services
sudo systemctl start openvas-scanner
sudo systemctl start openvas-manager
sudo systemctl start openvas-gsa  # Web interface

# Access web interface
# URL: https://localhost:9392
# Default credentials: admin / admin (change these!)
```

**Or Docker (Easiest):**
```bash
# Run OpenVAS in Docker
docker run -d \
  --name openvas \
  -p 9392:9392 \
  -e ADMIN_USERNAME=admin \
  -e ADMIN_PASSWORD=YourSecurePassword \
  greenbone/openvas:latest

# Access at https://localhost:9392
```

### STEP 2: SCAN YOUR LAB INFRASTRUCTURE

**Create scan targets in OpenVAS:**
1. **FortiGate firewall** (e.g., 10.0.0.1)
2. **NGINX load balancer** (e.g., 10.0.1.10, 10.0.1.11)
3. **Alpine boxes** (10.0.2.10, 10.0.2.11, 10.0.2.12)
4. **Backend servers** (10.0.1.100, 10.0.1.101, 10.0.1.102)

**In OpenVAS GUI:**
1. Click "Scan Management" → "Tasks"
2. Create New Task
3. Set Target: 10.0.0.0/24 (scan entire subnet)
4. Scanner: Full and fast
5. Start Scan

**Scan takes 30-60 minutes (first time)**

### STEP 3: ANALYZE FINDINGS

**Common vulnerabilities you'll find:**

| Severity | Example Findings | FortiGate | NGINX | Alpine |
|----------|------------------|-----------|-------|--------|
| **High** | SSH weak protocols | ❌ TLSv1.0 enabled | ❌ Old OpenSSL | ❌ No SSH key auth |
| **High** | Unpatched services | ❌ Old firmware | ❌ Outdated version | ❌ Old packages |
| **Medium** | Default credentials | ❓ Admin/admin | ✓ Should be OK | ❌ Likely defaults |
| **Medium** | Unnecessary services | ❓ HTTP enabled | ❌ Debug headers | ✓ Usually minimal |
| **Low** | Information disclosure | ❓ Banner grabbing | ❌ Server headers | ✓ Usually OK |

### STEP 4: DOCUMENT & FIX VULNERABILITIES

**Create a remediation spreadsheet:**

```
VULNERABILITY SCAN REPORT - Lab Network
Date: 2025-01-15
Scanner: OpenVAS
Target Scope: 10.0.0.0/24

| Severity | Host | Finding | Fix | Priority | Status |
|----------|------|---------|-----|----------|--------|
| HIGH | 10.0.0.1 (FortiGate) | TLSv1.0/1.1 enabled | Disable in admin settings | Critical | FIXED |
| HIGH | 10.0.1.10 (NGINX) | OpenSSL 1.0.2 (expired) | Update to 3.0+ | Critical | FIXED |
| MEDIUM | 10.0.2.10 (Alpine) | SSH password auth enabled | Disable, use keys only | High | FIXED |
| MEDIUM | 10.0.1.10 (NGINX) | Server header leaks version | Remove with add_header directive | Medium | FIXED |
| LOW | 10.0.0.1 | Banner grabbing possible | Cosmetic risk | Low | WONTFIX |
```

### STEP 5: FIX CRITICAL VULNERABILITIES

**Example 1: NGINX - Remove Server Header**
```nginx
# /etc/nginx/nginx.conf

# Add this to hide server version
server_tokens off;

# Remove Server header in responses
add_header Server "" always;

# Test after reload:
curl -I http://localhost
# Should NOT show: Server: nginx/1.24.0
```

**Example 2: Alpine Box - Disable SSH Password Auth**
```bash
# /etc/ssh/sshd_config

# Change these lines:
PasswordAuthentication no        # Disable password auth
PubkeyAuthentication yes          # Enable key auth only
PermitRootLogin no                # Disable root login

# Create authorized_keys for Alpine user
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "your-public-key-here" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Restart SSH
sudo service ssh restart

# Test: ssh-keygen, upload public key, verify key auth works
```

**Example 3: FortiGate - Disable Old TLS**
```bash
# SSH to FortiGate
ssh admin@10.0.0.1

# Disable weak protocols
config system global
  set tlsv10 disable
  set tlsv11 disable
  set ssl-min-proto-version tlsv12
end

# Verify
# Test with: openssl s_client -connect 10.0.0.1:443 -tls1
# Should fail; -tls1_2 should succeed
```

### STEP 6: RE-SCAN & VERIFY FIXES

```bash
# In OpenVAS, re-run same scan on fixed assets
# Target: 10.0.0.0/24
# Compare results: Before vs After

# Create comparison report:
Before fixes:
  - High: 3 findings
  - Medium: 5 findings
  - Low: 2 findings
  Total: 10 findings

After fixes:
  - High: 0 findings ✓
  - Medium: 2 findings (low-risk, accepted)
  - Low: 2 findings (cosmetic, accepted)
  Total: 4 findings → 60% reduction
```

### HOW TO TALK ABOUT THIS IN INTERVIEW:

> "I ran a vulnerability scan using OpenVAS against my lab network. Found several HIGH severity findings: TLSv1.0/1.1 enabled on FortiGate, outdated OpenSSL on NGINX, and password authentication enabled on Alpine servers. I remediated each: disabled old TLS protocols, updated OpenSSL, switched Alpine to SSH key-only authentication, and removed server version banners. I re-scanned to confirm fixes. This exercise taught me the difference between vulnerability detection and actual remediation—scanning is easy, but prioritizing and fixing without breaking systems is the real skill. I documented the process like a real change ticket: before state, risk assessment, remediation steps, and verification."

---

## PROJECT 2: FIREWALL RULE AUDIT & HARDENING
### Tool: FortiGate (Your lab firewall)

### Why This Matters:
- Shows "least privilege" security thinking
- Real security review experience (MSPs do this for clients)
- Identifies overly permissive rules that could be exploited
- Demonstrates technical depth (not just "apply more rules")

### STEP 1: DOCUMENT CURRENT FIREWALL RULES

**SSH to FortiGate:**
```bash
ssh admin@10.0.0.1
```

**Export current policies:**
```bash
# At FortiGate CLI
show firewall policy
# Or for specific policy:
show firewall policy 1
show firewall policy 2
show firewall policy 3
```

**Create a table of ALL policies:**

```
CURRENT FIREWALL POLICIES - Lab Network
Date: 2025-01-15
FortiGate: 10.0.0.1

| # | Name | Source | Destination | Service | Action | Notes |
|---|------|--------|-------------|---------|--------|-------|
| 1 | Client→NGINX | 10.0.2.0/24 | 10.0.1.0/24 | HTTPS(443) | ACCEPT | Allows client traffic to load balancer |
| 2 | NGINX→Backend | 10.0.1.0/24 | 10.0.1.100-102:8080 | Custom:8080 | ACCEPT | Proxying to backends |
| 3 | DMZ→WAN NAT | 10.0.1.0/24 | 0.0.0.0/0 | ALL | ACCEPT | **TOO PERMISSIVE!** |
| 4 | Admin Access | 192.168.1.0/24 | 10.0.0.0/8 | SSH, HTTPS | ACCEPT | Office access to lab |
| 5 | Implicit Deny | any | any | any | DENY | Catch-all (last rule) |
```

### STEP 2: AUDIT WITH LEAST PRIVILEGE LENS

**Review each rule for overpermissiveness:**

**BEFORE (Too Permissive):**
```
Policy 3: DMZ→WAN NAT
  Source: 10.0.1.0/24 (NGINX servers)
  Destination: 0.0.0.0/0 (ENTIRE INTERNET!)
  Service: ALL (🔴 CRITICAL ISSUE!)
  Action: ACCEPT

Why this is bad:
  - NGINX could initiate connections to ANYWHERE on the internet
  - If NGINX is compromised, attacker has free access
  - Violates least privilege (should only need what app uses)
```

**AFTER (Least Privilege):**
```
Policy 3a: DMZ→WAN (DNS)
  Source: 10.0.1.0/24 (NGINX servers)
  Destination: 8.8.8.8, 8.8.4.4 (Google DNS)
  Service: DNS (port 53 UDP)
  Action: ACCEPT
  Why: NGINX needs DNS to resolve backend hostnames

Policy 3b: DMZ→WAN (NTP)
  Source: 10.0.1.0/24
  Destination: pool.ntp.org (0.0.0.0/0 limited to NTP)
  Service: NTP (port 123 UDP)
  Action: ACCEPT
  Why: NGINX needs time sync for SSL certificates

Policy 3c: DMZ→WAN Implicit Deny
  Source: 10.0.1.0/24
  Destination: 0.0.0.0/0
  Service: ALL
  Action: DENY
  Why: Default deny anything else
```

### STEP 3: CREATE MSP-STYLE CHANGE TICKET

**Document like a professional security review:**

```
┌─────────────────────────────────────────────────────────────┐
│            FIREWALL POLICY AUDIT & REMEDIATION              │
│                     CHANGE TICKET #001                       │
└─────────────────────────────────────────────────────────────┘

SUMMARY:
Audit of FortiGate firewall policies identified overly permissive 
rules allowing unnecessary outbound access. Recommend restricting 
to least-privilege: only allow required services and destinations.

RISK ASSESSMENT:
Current State: HIGH RISK
  - Policy 3 allows NGINX servers to connect to ANY destination
  - If NGINX compromised, attacker has unrestricted internet access
  - Violates principle of least privilege

Proposed State: LOW RISK
  - Only allow DNS (8.8.8.8, 8.8.4.4) on port 53
  - Only allow NTP (pool.ntp.org) on port 123
  - Deny all other outbound connections
  - Attackers cannot exfiltrate data or pivot to other networks

CHANGES:

BEFORE (Policy 3):
  set srcintf "dmz"
  set dstintf "wan1"
  set srcaddr "dmz-servers" (10.0.1.0/24)
  set dstaddr "all"
  set service "ALL" ← PROBLEM!
  set action accept
  set nat enable

AFTER (Policy 3a, 3b, 3c):
  
  Policy 3a (DNS):
    set srcintf "dmz"
    set dstintf "wan1"
    set srcaddr "dmz-servers"
    set dstaddr "dns-servers" (8.8.8.8, 8.8.4.4)
    set service "DNS"
    set action accept
    set nat enable
  
  Policy 3b (NTP):
    set srcintf "dmz"
    set dstintf "wan1"
    set srcaddr "dmz-servers"
    set dstaddr "ntp-servers" (pool.ntp.org)
    set service "NTP"
    set action accept
    set nat enable
  
  Policy 3c (Implicit Deny):
    set srcintf "dmz"
    set dstintf "wan1"
    set srcaddr "dmz-servers"
    set dstaddr "all"
    set service "ALL"
    set action deny
    (No NAT—traffic is dropped)

ROLLBACK PLAN:
If issues occur:
1. Re-enable original Policy 3 (set action accept for ALL services)
2. Disable Policies 3a, 3b, 3c
3. Test connectivity
4. Investigate what service needs access
5. Add specific policy for that service

TESTING PLAN:
Before deployment:
  ✓ NGINX can still resolve DNS: nslookup google.com from NGINX
  ✓ NGINX can sync time: ntpdate -d pool.ntp.org from NGINX
  ✓ Backend connectivity unaffected: curl from NGINX to backend:8080
  ✓ Outbound to random host blocked: curl from NGINX to 8.8.4.4:53 (non-DNS) → fails

APPROVAL:
Date: 2025-01-15
Approved by: [Your Name]
Test environment: Lab Network
Status: Ready for deployment
```

### STEP 4: IMPLEMENT CHANGES

**On FortiGate CLI:**
```bash
# Backup current config first!
execute backup config tftp 10.0.0.254 lab-backup-before-audit.conf

# Remove old permissive rule
config firewall policy
  delete 3
end

# Add restrictive DNS rule
config firewall policy
  edit 31
  set name "DMZ→WAN DNS"
  set srcintf "dmz"
  set dstintf "wan1"
  set srcaddr "dmz-servers"
  set dstaddr "dns-servers"  # Previously created address object
  set service "DNS"
  set action accept
  set nat enable
  next
end

# Add restrictive NTP rule
config firewall policy
  edit 32
  set name "DMZ→WAN NTP"
  set srcintf "dmz"
  set dstintf "wan1"
  set srcaddr "dmz-servers"
  set dstaddr "ntp-servers"
  set service "NTP"
  set action accept
  set nat enable
  next
end

# Add implicit deny rule
config firewall policy
  edit 33
  set name "DMZ→WAN Implicit Deny"
  set srcintf "dmz"
  set dstintf "wan1"
  set srcaddr "dmz-servers"
  set dstaddr "all"
  set service "ALL"
  set action deny
  next
end

# Verify changes
show firewall policy
```

### STEP 5: TEST & DOCUMENT RESULTS

```bash
# From NGINX server (10.0.1.10):

# Test 1: DNS should work
$ nslookup google.com
Server: 8.8.8.8
Address: 8.8.8.8#53
Name: google.com
Address: 142.250.185.46

# Test 2: NTP should work
$ ntpdate -d pool.ntp.org
... synced successfully

# Test 3: Backend connectivity should work
$ curl http://10.0.1.100:8080
<HTML>Backend Server 1</HTML>

# Test 4: Random outbound should FAIL
$ curl http://8.8.4.4:53
curl: (7) Failed to connect to 8.8.4.4 port 53: Connection refused
✓ Correctly blocked!

# Test 5: HTTPS from client should still work
$ curl https://10.0.1.10
<HTML>NGINX Page</HTML>
✓ Still works!
```

### HOW TO TALK ABOUT THIS IN INTERVIEW:

> "I performed a firewall policy audit on my lab FortiGate. Found that the DMZ-to-WAN policy allowed ALL services to ANY destination—completely violated least privilege. An attacker who compromised NGINX could access anywhere. I created a professional change ticket documenting the risk, the specific before/after policies, rollback plan, and testing. I then implemented least-privilege rules: allow only DNS to 8.8.8.8/4.4, only NTP to pool.ntp.org, deny everything else. I tested each rule to confirm: DNS works, NTP works, backend access works, but random outbound is blocked. This exercise taught me the difference between 'it works' and 'it works securely'—and how to document security changes for production environments."

---

## PROJECT 3: CIS HARDENING CHECKLIST
### Target: FortiGate Firewall

### Why This Matters:
- CIS Benchmarks are industry standard (used by enterprises everywhere)
- Shows you know security frameworks (ISO 27001, NIST, CIS, etc.)
- Specific artifact for interviews (actual checklist + evidence)
- Proves you can apply standards to real infrastructure

### STEP 1: GET CIS FORTIGATE BENCHMARK

**Download free CIS benchmark:**
1. Go to https://www.cisecurity.org/cis-benchmarks/
2. Find "FortiGate Firewall" benchmark
3. Download PDF (free registration required)

**Or use these common hardening areas:**
```
FortiGate CIS Benchmark v2.0 (Common Areas):
1. Authentication & Access Control
2. System Configuration
3. Network Access Control
4. Logging & Monitoring
5. Security Features
```

### STEP 2: CREATE HARDENING CHECKLIST

**Document your lab FortiGate against CIS:**

```
┌────────────────────────────────────────────────────────────┐
│        FORTIGATE HARDENING CHECKLIST (CIS v2.0)            │
│            Lab Environment: 10.0.0.1                        │
│            Date: 2025-01-15                                 │
└────────────────────────────────────────────────────────────┘

SECTION 1: AUTHENTICATION & ACCESS CONTROL
══════════════════════════════════════════════════════════════

[ ✓ ] 1.1 - Password Policy
  Requirement: Enforce strong password policy (min 12 chars, complexity)
  Current State: ❌ FAILED
    admin password: "admin123" (TOO WEAK!)
  Fix: Change to complex password
    $ set system admin 1
    $ set password MyStr0ng!Pass2025$
  Status: ✓ FIXED
  Verification: Attempted simple passwords → rejected

[ ✓ ] 1.2 - Disable Default Accounts
  Requirement: Disable or rename admin account
  Current State: ❌ FAILED
    Default "admin" account active
  Fix: Created new admin account "fortigeyadmin"
    config system admin
      edit "fortigeyadmin"
      set password [strong password]
      set trusthost 10.0.0.0 255.255.255.0
      next
    end
  Status: ✓ FIXED
  Verification: Can log in as fortigeyadmin; admin account disabled

[ ✓ ] 1.3 - Enable SSH Key Authentication
  Requirement: Use SSH keys instead of passwords
  Current State: ⚠️ PARTIAL
    Password auth enabled but keys not configured
  Fix: Add SSH public key
    config system admin
      edit "fortigeyadmin"
      set ssh-public-key1 "ssh-rsa AAAA..." 
      next
    end
  Status: ✓ FIXED
  Verification: SSH with key works; can disable password auth

[ ✗ ] 1.4 - Restrict Admin Access by IP
  Requirement: Limit admin access to trusted networks only
  Current State: ❌ FAILED
    Admin accessible from anywhere (0.0.0.0/0)
  Fix: Restrict to management network
    config system admin
      edit "fortigeyadmin"
      set trusthost 10.0.0.0 255.255.255.0  # Only from lab network
      next
    end
  Status: ✓ FIXED
  Verification: SSH from 10.0.0.0/24 → works; from external IP → blocked

[ ✗ ] 1.5 - Disable Unnecessary Admin Interfaces
  Requirement: Disable HTTP admin (use HTTPS only)
  Current State: ❌ FAILED
    Admin accessible via HTTP (port 80)
  Fix: Disable HTTP admin interface
    set admin-https-pki-required enable
    set admin-https-ssl-version tlsv1-2
    set admintimeout 15  # Auto-logout after 15 min
  Status: ✓ FIXED
  Verification: HTTP → redirects to HTTPS

SECTION 2: SYSTEM CONFIGURATION
══════════════════════════════════════════════════════════════

[ ✓ ] 2.1 - Update Firmware to Latest
  Requirement: Run latest supported firmware version
  Current State: ❌ FAILED
    Firmware: 7.0.8 (older version)
  Fix: Update to 7.4.3 (latest stable)
    Upload firmware via CLI or GUI
  Status: ✓ FIXED
  Verification: show system status | grep "Version"
    Result: 7.4.3

[ ✓ ] 2.2 - Enable Automatic Firmware Updates
  Requirement: Configure auto-update policy
  Current State: ❌ FAILED
    Auto-update disabled
  Fix: Enable automatic updates
    config system update-check
      set enable-update-schedule enable
      set update-schedule 02:00  # 2am daily
      next
    end
  Status: ✓ FIXED
  Verification: Check system update logs

[ ✓ ] 2.3 - Disable Unnecessary Services
  Requirement: Disable HTTP, SNMP, Telnet (if not needed)
  Current State: ⚠️ PARTIAL
    HTTP enabled, Telnet disabled, SNMP disabled
  Fix: Disable HTTP admin
    set admin-https-pki-required enable
  Status: ✓ FIXED

[ ✓ ] 2.4 - Enable Syslog for Audit Logs
  Requirement: Send logs to central syslog server for audit
  Current State: ❌ FAILED
    No remote logging configured
  Fix: Send logs to central server (10.0.0.254)
    config log syslogd setting
      set status enable
      set server 10.0.0.254
      set port 514
      next
    end
  Status: ✓ FIXED
  Verification: Syslog server receives logs

SECTION 3: NETWORK ACCESS CONTROL
══════════════════════════════════════════════════════════════

[ ✓ ] 3.1 - Implement Firewall Policies (Least Privilege)
  Requirement: Default deny, explicit allow for required traffic
  Current State: ✓ ALREADY DONE
    Implicit deny at end of firewall policies
  Status: ✓ VERIFIED

[ ✓ ] 3.2 - Restrict VLAN Access
  Requirement: Don't allow all VLANs to communicate
  Current State: ⚠️ PARTIAL
    VLAN 1 (Client) ↔ VLAN 2 (DMZ) ↔ VLAN 3 (Backend)
    All inter-VLAN traffic allowed
  Fix: Restrict to required connections only
    Policy: Client → DMZ (HTTPS only)
    Policy: DMZ → Backend (8080 only)
    Deny: Backend → DMZ (no reverse traffic)
  Status: ✓ FIXED

[ ✓ ] 3.3 - Disable IP Forwarding if Not Needed
  Requirement: Disable IP forwarding on unnecessary interfaces
  Current State: ✓ ALREADY CONFIGURED
    Forwarding only enabled on required interfaces
  Status: ✓ VERIFIED

SECTION 4: LOGGING & MONITORING
══════════════════════════════════════════════════════════════

[ ✓ ] 4.1 - Enable Firewall Logging
  Requirement: Log all denied connections for audit
  Current State: ❌ FAILED
    Firewall logging disabled
  Fix: Enable logging for denied connections
    config log firewall setting
      set disable-alert-on-restriction disable
      set log-policy-comment enable
      set log-policy-name enable
      next
    end
  Status: ✓ FIXED
  Verification: Check firewall log shows denied packets

[ ✓ ] 4.2 - Enable Admin Activity Logging
  Requirement: Log all admin changes for audit trail
  Current State: ⚠️ PARTIAL
    Some logging, but not comprehensive
  Fix: Enable full admin log
    config log setting
      set log-invalid-packet enable
      set log-policy-disabled-implicit-deny enable
      next
    end
  Status: ✓ FIXED

[ ✓ ] 4.3 - Set Log Retention Policy
  Requirement: Keep logs for audit period (e.g., 90 days)
  Current State: ❌ FAILED
    Local logs filling disk
  Fix: Configure log rotation and retention
    config log setting
      set log-disk-quota-size 500  # 500MB max
      set log-disk-quota-warning-threshold 80
      next
    end
  Status: ✓ FIXED

SECTION 5: SECURITY FEATURES
══════════════════════════════════════════════════════════════

[ ✓ ] 5.1 - Enable DDoS Protection
  Requirement: Enable anti-DDoS mechanisms
  Current State: ⚠️ PARTIAL
    Basic DDoS enabled, not tuned for lab
  Fix: Configure DDoS policies
    config firewall ddos-policy
      set enable enable
      next
    end
  Status: ✓ VERIFIED

[ ✓ ] 5.2 - Enable IPS/IDS
  Requirement: Enable intrusion prevention signatures
  Current State: ✓ ALREADY ENABLED
    IPS signatures updated
  Status: ✓ VERIFIED

[ ✓ ] 5.3 - Enable SSL Inspection (Optional for Lab)
  Requirement: Inspect encrypted traffic for threats
  Current State: ❌ SKIPPED
    Not needed in lab environment
  Reason: Lab is trusted; production would need this
  Status: N/A for lab

═══════════════════════════════════════════════════════════════
SUMMARY:

Before Hardening:
  ✓ Passed: 5 checks
  ✗ Failed: 12 checks
  ⚠️ Partial: 3 checks
  Score: 25/20 (vulnerable)

After Hardening:
  ✓ Passed: 18 checks
  ✗ Failed: 0 checks (all fixed!)
  ⚠️ Partial: 2 checks (acceptable for lab)
  Score: 90/20 (significantly hardened)

KEY IMPROVEMENTS:
  ✓ Strong authentication (complex password, SSH key, IP restrictions)
  ✓ Updated firmware to latest stable version
  ✓ Disabled unnecessary services (HTTP admin)
  ✓ Implemented least-privilege firewall policies
  ✓ Enabled comprehensive logging for audit trail
  ✓ Proper log retention and rotation
  
REMAINING WORK (For Production):
  - SSL inspection (needed for HTTPS traffic analysis)
  - High-availability setup (redundancy)
  - Integration with SIEM (for central log analysis)
```

### HOW TO TALK ABOUT THIS IN INTERVIEW:

> "I created a hardening checklist based on CIS benchmarks and applied it to my FortiGate firewall. Found ~12 vulnerabilities: weak admin password, HTTP admin enabled, no SSH key auth, insufficient logging. I addressed each: enforced strong password policy, switched to SSH keys with IP-based access control, updated firmware to latest version, enabled comprehensive firewall and admin logging, configured log retention. My checklist went from 25% compliant to 90% compliant. This taught me how security frameworks aren't just theory—they're practical checklists that make infrastructure measurably more secure. The documentation (before/after state) is exactly what you'd present to a security audit."

---

## PROJECT 4: ZTNA VS TRADITIONAL VPN COMPARISON
### Conceptual Analysis + Lab Implementation

### Why This Matters:
- Shows strategic thinking about network security evolution
- RingCentral, Pinnacle, and other companies care about ZTNA (Zero Trust Network Access)
- Demonstrates you understand modern vs. legacy approaches
- Good for discussions about infrastructure transformation

### STEP 1: UNDERSTAND THE CONCEPTS

**Your Current Lab Setup (Traditional VPN):**
```
Client (1.2.3.4 remote)
    ↓ OpenVPN (encrypted tunnel)
    ↓
VPN Gateway (10.0.0.254)
    ↓ All traffic allowed? 
    ↓
NGINX Load Balancer (10.0.1.10)
    ↓ Unrestricted access to entire LAN?
    ↓
Backend Servers (10.0.1.100-102)

Trust Model: "If you're in the VPN tunnel, trust everything"
```

**ZTNA Model (What Companies Are Moving To):**
```
Client (1.2.3.4 remote)
    ↓ (Authenticate with MFA)
    ↓
Identity Provider (Okta, Azure AD)
    ↓ (Device compliance check: is laptop patched? firewall on?)
    ↓
ZTNA Gateway/Controller (Checks: User=Kenneth, Device=Laptop, Role=Admin)
    ↓ (Policy: Kenneth (admin) can access NGINX on port 443 only)
    ↓
Proxy/Gateway (Grants access ONLY to NGINX:443)
    ↓
NGINX Load Balancer (10.0.1.10:443 only)
    ↓ No access to backend, other servers, admin interfaces, etc.
    ↓
(If Kenneth needs backend access: different policy, different identity)

Trust Model: "Trust nothing by default. Verify every request."
```

### STEP 2: CREATE COMPARISON DOCUMENT

**Create a professional comparison:**

```
┌──────────────────────────────────────────────────────────┐
│     ZERO TRUST NETWORK ACCESS (ZTNA)                     │
│        vs Traditional VPN                                 │
│     Analysis & Lab Implementation Plan                    │
└──────────────────────────────────────────────────────────┘

TRADITIONAL VPN (Current Lab Setup)
════════════════════════════════════════════════════════════

Architecture:
  Client → OpenVPN Tunnel → VPN Gateway → All LAN Resources

Authentication:
  - Single VPN credential (shared pre-shared key or cert)
  - No per-application authentication
  - No device health verification

Authorization:
  - Once connected, access to almost everything
  - VLAN-based segmentation (client VLAN → all servers in VLAN)
  - Coarse-grained (all or nothing)

Trust Assumption:
  "If someone has valid VPN credentials, trust them completely"

Security Model:
  ✓ Perimeter-focused (trust inside, don't trust outside)
  ✓ Good for offices with laptops VPNing in
  ✗ Bad if credentials compromised (attacker inside)
  ✗ Bad for ransomware (lateral movement = easy)
  ✗ Weak for remote/distributed teams

When It Works Well:
  - Corporate offices with few remote workers
  - Small team, high trust environment
  - Simple networks without sensitive data segmentation

When It Fails:
  - Employee laptop stolen (VPN creds = access to everything)
  - Contractor needs access to one app (gets access to all)
  - Ransomware on VPN client (spreads to all connected resources)
  - Insider threat (once in VPN, can attack other systems)

Lab Example:
  OpenVPN connects Kenneth to 10.0.0.0/24
  Kennedy can now:
    - Access NGINX (10.0.1.10) ✓
    - Access backend (10.0.1.100-102) ✓ (shouldn't be able to)
    - SSH to admin box (10.0.0.2) ✓ (shouldn't be able to)
    - Run network scans ✓ (shouldn't be able to)

Risk: 🔴 HIGH


ZERO TRUST NETWORK ACCESS (ZTNA)
════════════════════════════════════════════════════════════

Architecture:
  Client → (Auth: MFA + Device Check) → ZTNA Controller 
       → (Policy: User X can access App Y only) 
       → Conditional Access Gateway 
       → Only the specific resource (e.g., NGINX:443)

Authentication:
  - Multi-factor authentication (MFA) required
  - Identity verification (who are you?)
  - Device health check (is your laptop secure?)
  - Continuous verification (not just once at login)

Authorization:
  - Per-application access control
  - Role-based policies (admin can access X, user can access Y)
  - Attribute-based (time of day, device type, network location)
  - Dynamic (policies change based on risk score)

Trust Assumption:
  "Trust nothing. Verify everything. Every. Single. Request."

Security Model:
  ✓ Zero implicit trust (even inside the network)
  ✓ Least privilege (access only what you need)
  ✓ Good for remote/distributed workforce
  ✓ Good for protecting against insider threats
  ✓ Good for preventing lateral movement
  ✗ More complex to set up
  ✗ Slightly higher latency (verification adds time)
  ✗ Requires more infrastructure

When It Works Well:
  - Large distributed teams (remote workers worldwide)
  - High-security environments (financial, healthcare, defense)
  - Multi-tenant environments (contractors, vendors)
  - Organizations protecting sensitive data
  - Any organization needing regulatory compliance (SOC 2, HIPAA)

When It's Overkill:
  - Very small team (< 5 people)
  - All local network access
  - No sensitive data
  - High-trust environment

Lab Example (ZTNA Implementation):
  Kenneth (admin) logs in:
    1. Enters credentials → MFA (SMS code)
    2. System checks: Device=MyLaptop, OS=UpToDate, Firewall=On
    3. ZTNA Controller grants: "Kenneth can access NGINX:443"
    4. Kenneth connects to NGINX only (no backend access)
  
  Different user (contractor) logs in:
    1. Enters credentials → MFA
    2. System checks: Device=SharedLaptop, OS=Outdated → DENIED
       OR grants with restrictions: "Can access NGINX but not API"
  
  If Kenneth's device is compromised:
    1. Malware on laptop tries to scan network
    2. ZTNA controller detects suspicious behavior
    3. Access revoked in real-time
    4. Malware confined to laptop, can't pivot to network

Risk: 🟢 LOW


COMPARISON TABLE
════════════════════════════════════════════════════════════

| Aspect | Traditional VPN | ZTNA |
|--------|-----------------|------|
| **Authentication** | Credentials only | MFA + Device verification |
| **Trust Model** | Perimeter-based | Zero trust |
| **Granularity** | Coarse (subnet-level) | Fine (app-level) |
| **Access Duration** | Full session | Per-request |
| **Lateral Movement** | Easy | Difficult |
| **Credential Compromise** | Severe impact | Contained |
| **Setup Complexity** | Simple | Complex |
| **Compliance Ready** | Partial | Full |
| **Remote Work Friendly** | Medium | Excellent |
| **Cost** | Low | Medium-High |


LAB IMPLEMENTATION PLAN: MOVING FROM VPN TO ZTNA
════════════════════════════════════════════════════════════

Phase 1: Keep OpenVPN for now, add application-level access control
  - NGINX already requires credentials (login)
  - Can add MFA at NGINX level (Duo, Google Authenticator)
  - Restrict NGINX admin access by role

Phase 2: Add a ZTNA controller (open-source: Tailscale, WireGuard + policies)
  - Tailscale: Free tier available, easiest for lab
  - Replaces OpenVPN with ZTNA-like behavior
  - Can define: "Kenneth can access NGINX:443, nothing else"

Phase 3: Implement device compliance checks
  - Check if client device is patched
  - Check firewall status
  - Revoke access if device unhealthy

CONCRETE LAB STEPS:

1. Replace OpenVPN with Tailscale (5 minutes):
   $ curl -fsSL https://tailscale.com/install.sh | sh
   $ sudo tailscale up
   - Connects to Tailscale network (zero-trust by default)
   
2. Add Duo MFA to NGINX (15 minutes):
   - Install Duo UNIX module
   - Protect NGINX with Duo 2FA
   
3. Create access policies in Tailscale:
   Kenneth (admin) → Can reach 10.0.1.10:443 (NGINX only)
   Contractor (limited) → Can reach 10.0.1.10:443 (NGINX only, no API)
   Backend service → Can reach 10.0.1.100:8080 only

4. Test ZTNA:
   - Kenneth connects → accesses NGINX ✓
   - Kenneth tries backend → blocked ✗ (no policy)
   - Contractor tries admin access → blocked ✗ (no role)
   - Real zero trust in action!


HOW TO TALK ABOUT THIS IN INTERVIEW
════════════════════════════════════════════════════════════

"I've analyzed the shift from traditional VPN to Zero Trust Network 
Access. Traditional VPN trusts anyone with valid credentials, which is 
risky if credentials are compromised or device is stolen. ZTNA verifies 
EVERY request: who is the user, what's their role, is their device 
secure, do they have permission to access THIS specific resource.

In my lab, I implemented this conceptually by documenting how I'd move 
from OpenVPN (current) to Tailscale-based ZTNA (desired state). I 
outlined the policy differences: traditional VPN grants access to entire 
subnet, ZTNA grants access only to specific applications.

For RingCentral, this matters because they're migrating customers from 
legacy VPN to modern cloud access. Understanding ZTNA principles is 
critical for infrastructure that needs to support both models during 
transition."
```

### STEP 3: IMPLEMENT BASIC ZTNA IN LAB (Optional but Impressive)

**Install Tailscale (easiest open-source ZTNA):**

```bash
# On your lab controller
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# On each lab device
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Define access policy
# In Tailscale dashboard: Access Controls
# Example policy:
{
  "ACLs": [
    {
      "Action": "accept",
      "Principal": "autogroup:admin",
      "Resources": ["10.0.1.10:443"]  # NGINX only
    },
    {
      "Action": "accept",
      "Principal": "autogroup:members",
      "Resources": ["10.0.1.10:443"]  # Limited users also HTTPS only
    },
    {
      "Action": "deny",
      "Principal": "*",
      "Resources": ["*"]  # Deny everything else
    }
  ]
}

# Test the policy
# Kenneth connects → can reach NGINX ✓
# Kenneth tries SSH to backend → denied ✗
# ZTNA working!
```

---

## HOW TO PRESENT ALL 4 PROJECTS IN INTERVIEW

**When asked: "Tell me about a security project you've done"**

> "I've completed a comprehensive security hardening project in my lab across four areas:

> **1. Vulnerability Scanning:** Ran OpenVAS against my entire network, found HIGH severity findings (old TLS versions, unpatched services, default credentials). I remediated each: updated OpenSSL, disabled TLSv1.0/1.1, enforced SSH key authentication. Re-scanned to confirm 60% reduction in vulnerabilities.

> **2. Firewall Rule Audit:** Reviewed my FortiGate policies and found an overly permissive rule allowing NGINX servers to connect to anywhere on the internet. This violated least privilege—if NGINX was compromised, attacker had unrestricted internet access. I implemented proper policy: allow only DNS (specific servers, port 53) and NTP (specific server, port 123), deny everything else. Created an MSP-style change ticket with before/after, risk assessment, rollback plan, and testing procedures.

> **3. CIS Hardening Checklist:** Applied the CIS Fortigate benchmark to my firewall. Created a detailed checklist covering authentication, system config, network access, logging, and security features. Went from 25% compliant (vulnerable) to 90% compliant (hardened). Documented every fix.

> **4. ZTNA Analysis:** Analyzed the shift from traditional VPN (my current setup) to Zero Trust Network Access (industry direction). Documented the security improvements and created a lab implementation plan using Tailscale. This shows I understand modern security architecture, not just legacy systems.

> These four projects together demonstrate: security tools (OpenVAS), infrastructure hardening (firewall rules), compliance frameworks (CIS), and strategic thinking (ZTNA). They're not theoretical—they're real, documented, and repeatable."

---

**This is interview gold!** 🎯 You now have:
✅ Real vulnerability scan results
✅ Professional change ticket documentation
✅ CIS hardening checklist (before/after)
✅ ZTNA vs VPN comparison
✅ Concrete, specific stories to tell

Get to it! Start with the vulnerability scan (easiest to show results). Good luck! 🚀
