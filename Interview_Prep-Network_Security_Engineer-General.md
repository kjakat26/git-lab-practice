# NETWORK SECURITY ENGINEER - INTERVIEW PREP
## MSP Role: Vulnerability Remediation & Security Hardening

---

## ROLE SUMMARY (For Your Understanding)

**What they want:** Someone who can take vulnerability scan findings → fix them → harden systems so they stay fixed

**Why it matters:** MSP manages multiple clients, needs someone who:
- Can work independently on security projects
- Understands both attack and defense
- Comfortable with repetition (same hardening applied to many clients)
- Good documentation (each client needs proof of work)
- Reliable (security mistakes hurt client reputation)

**Your advantages:**
- ✅ 13+ years network/security
- ✅ CCNP Security certified
- ✅ FortiGate firewall experience
- ✅ VPN and cloud background
- ✅ Lab project proves hands-on skills
- ✅ Can discuss CIS benchmarks (you're doing it!)
- ✅ Familiar with Qualys (vulnerability management)

---

## INTERVIEW QUESTION CATEGORIES

### CATEGORY 1: VULNERABILITY SCANNING & REMEDIATION
**(Most Important - Core Job)**

---

### Q1: "Walk me through your vulnerability scanning and remediation process"

**Why they ask:** 
- Assessing if you actually know how to scan and fix (not just theory)
- Want to see your methodology and judgment

**Your Answer:**
> "I've used Qualys VMDR extensively. My process is:

> **1. Scan Preparation:**
> - Coordinate with client/stakeholders about timing (avoid production windows)
> - Understand network topology so scan targets are correct
> - Document baseline (what systems are expected to have what)

> **2. Execute Scan:**
> - Run full scan with credentialed access (most accurate)
> - Or unauthenticated if credentialed not possible
> - For firewall/NGFW: I also run specific checks for configuration issues

> **3. Review & Prioritize:**
> - Sort by CVSS score (9.0+ = critical, fix immediately)
> - Categorize by type:
>   * Patches: OS/app updates (can often automate via Patch Management)
>   * Configuration: Hardening fixes (always manual, requires judgment)
>   * Deprecated: Systems we need to decommission anyway
> - Consider business impact: A patch that requires reboot during business hours gets scheduled, not rushed

> **4. Remediate:**
> - For patches: Approve in Qualys Patch Management, test in non-prod first, deploy in maintenance window
> - For config issues: SSH to device, implement fix per remediation guidance, test connectivity, verify no service disruption
> - Document every change: what was changed, why, what was tested

> **5. Re-Scan & Verify:**
> - Run targeted re-scan (faster than full re-scan)
> - Confirm vulnerability is gone
> - If not gone, troubleshoot: did the fix apply correctly? Do we need a different approach?

> **6. Report & Close:**
> - Show before/after results
> - Calculate: X vulnerabilities → Y vulnerabilities (Z% reduction)
> - Document any deferred/accepted risks with business approval
> - Track for future audits

> **Key judgment call:** Not all findings are equal. A missing patch is often straightforward (install and verify). A configuration change might require more thought: disabling TLS 1.0 is good security, but what if a legacy app needs it? Then you might disable it for production but keep it for that one app, or phase the change.

> Recently in my lab, I scanned my network with OpenVAS, found TLSv1.0/1.1 enabled on FortiGate (HIGH severity), implemented the fix, re-scanned to confirm. This is the same workflow I use with Qualys daily."

**Why this answer is strong:**
- Shows real Qualys experience
- Demonstrates methodology (not ad-hoc)
- Shows judgment (not everything is urgent)
- Mentions documentation (MSPs care about this)
- References lab work as current practice

---

### Q2: "Tell me about a time when a vulnerability was difficult to remediate. What made it hard? How did you handle it?"

**Why they ask:**
- Want to see problem-solving under constraints
- Real-world vulnerabilities aren't always straightforward
- Assessing maturity/judgment

**Your Answer (Example 1 - Real situation):**
> "At OpenVPN Inc., we found missing security patches on production servers, but they required reboots during business hours. Standard answer would be 'just apply the patch,' but that would impact live customer connections.

> What made it hard: Customers needed 99.99% uptime; a reboot = downtime. But not patching = security risk.

> How I handled it:
> - Analyzed the vulnerability: Was it exploitable? Was it actively being exploited? Severity?
> - For critical CVEs: Coordinated with product team for emergency maintenance window (4 AM Sunday, customers notified 1 week prior)
> - For high-severity but not critical: Tested patch in non-prod, confirmed no app breakage, scheduled phased rollout (10% → 50% → 100%)
> - For medium-severity: Could wait for next scheduled maintenance

> The key was: don't just blindly patch. Understand the business impact, prioritize accordingly, and communicate with stakeholders. We reduced vulnerability count 30% while maintaining uptime SLAs."

**Alternative (Lab Example):**
> "In my lab hardening project, I found that disabling TLS 1.0/1.1 on FortiGate was straightforward. But I discovered a legacy admin interface that only worked with TLS 1.0. 

> What made it hard: Can't just disable TLS—breaks that interface.

> How I handled it:
> - Documented the issue
> - Created two solutions: (1) Update legacy tool (preferred), (2) Create separate FortiGate interface for that tool only (workaround)
> - For a real environment, you'd probably update the tool. In lab, I implemented the workaround to show understanding of tradeoffs.

> This taught me that 'remediate vulnerabilities' isn't always a technical problem—it's a business problem too."

**Why this answer works:**
- Shows real judgment
- Considers business impact
- Communication-focused
- Not just "I ran a command"

---

### Q3: "Describe the most complex vulnerability scan you've managed. How many findings? How did you prioritize?"

**Your Answer:**
> "At Globe Telecom, I ran comprehensive security scans across IP core network devices (routers, firewalls, carrier-grade equipment). Typical scan found 50-100 findings per device, across 20-30 devices = 1000+ total findings.

> Prioritization approach:

> **Tier 1 (Fix immediately - same day):**
> - CVSS 9.0+: Critical RCE, authentication bypass, etc.
> - Actively exploited vulnerabilities (known in-the-wild)
> - Network-wide impact (affects all customers)
> Example: Missing security patch on BGP router = all traffic could be hijacked

> **Tier 2 (Fix within 1 week):**
> - CVSS 7.0-8.9: High severity, but requires local access
> - Affects limited scope (specific customer, specific service)
> Example: Weak SSH ciphers on management interface

> **Tier 3 (Fix within 1 month):**
> - CVSS 4.0-6.9: Medium severity
> - Cosmetic/information disclosure issues
> Example: Banner grabbing reveals OS version

> **Tier 4 (Track but don't fix immediately):**
> - Low severity, accepted risk
> - Would be fixed during next maintenance window
> Example: Outdated firmware on non-critical device

> Key: I tracked this in a spreadsheet: device, finding, CVSS, severity, fix date, owner. Reported to management weekly. 

> Result: After 4 weeks, we had fixed 80% of HIGH and CRITICAL findings. Remaining 20% had legitimate business reasons for deferral (documentation, stakeholder approval, planned decommissioning).

> This is the kind of project I'm ready to do at an MSP—manage large portfolios of findings across many client devices and deliver reliable, documented results."

**Why this works:**
- Shows you can handle scale
- Real prioritization methodology
> Demonstrates tracking/documentation

---

## CATEGORY 2: FIREWALL ADMINISTRATION & HARDENING

---

### Q4: "What's your experience with Next-Generation Firewalls? Which platforms?"

**Your Answer:**
> "I have 4+ years hands-on FortiGate experience (Fortinet's NGFW platform), which I consider one of the best in the market. At Globe Telecom, I managed a fleet of 50+ FortiGate units across multiple sites.

> Key things I've done with FortiGate:
> - Designed and implemented security policies from scratch (DMZ segmentation, VPN rules, IPS/DLP policies)
> - Performed security hardening: disabled weak protocols, enforced strong authentication, configured centralized logging
> - Managed FortiManager for centralized policy management across 50+ devices (60% reduction in policy deployment time)
> - Troubleshooted connectivity issues (firewall blocking legitimate traffic, policy conflicts)
> - Implemented redundancy and HA (Keepalived + VRRP for zero-downtime failover)

> I'm also familiar with the concepts on competing platforms:
> - Palo Alto (app-based policies, very granular)
> - SonicWall (good SMB option, solid threat protection)
> - Meraki (cloud-managed, easier for distributed sites)

> But FortiGate is my deep expertise. I can walk you through a complex security policy audit, explain the difference between firewall rules and app controls, and troubleshoot why a policy is working or not working."

**Follow-up they might ask:**
> "Can you give an example of a policy audit you did?"

> "Sure. I reviewed our DMZ policies and found rule 3 allowed NGINX servers to connect to ANY destination with ALL services. This violated least privilege. If NGINX was compromised, attacker had unrestricted internet access. I documented the risk, created a change ticket, implemented least-privilege policies: allow only DNS (specific servers) and NTP (specific server), deny everything else. I tested it, documented the before/after, and it's exactly the kind of audit work I'd do here."

---

### Q5: "Walk me through how you'd harden a firewall to a baseline standard"

**Why they ask:**
- This is a core part of the job
- Want to see if you have a systematic approach (vs. ad-hoc)

**Your Answer:**
> "I use industry-standard frameworks like CIS Benchmarks as a starting point. Here's my approach:

> **Phase 1: Baseline Security**
> - Change default credentials (strong password, no 'admin/admin')
> - Disable unnecessary services (HTTP admin interface → HTTPS only)
> - Update to latest stable firmware
> - Configure syslog to central server (audit trail)

> **Phase 2: Access Control**
> - Restrict admin access by IP (office networks only)
> - Require MFA or SSH keys for admin access (no passwords)
> - Limit who can make config changes (audit logging)

> **Phase 3: Network Segmentation**
> - Implement implicit deny (default reject all traffic)
> - Create specific policies for required flows (client → server, only HTTPS)
> - Remove any 'allow ALL' rules

> **Phase 4: Encryption & Protocols**
> - Disable weak TLS versions (only TLSv1.2+)
> - Disable weak ciphers
> - Disable deprecated protocols (SSLv3, etc.)

> **Phase 5: Logging & Monitoring**
> - Enable firewall logging (log denied connections)
> - Enable admin activity logging (who changed what)
> - Configure log rotation (prevent disk filling)

> **Phase 6: Threat Protection**
> - Enable IPS/IDS signatures
> - Enable DDoS protection
> - Enable antivirus/malware scanning (if NGFW supports it)

> **Phase 7: Documentation & Testing**
> - Document all changes (before/after state)
> - Test each change doesn't break legitimate traffic
> - Create a rollback plan (if something goes wrong)

> For an MSP, I'd also create a hardening checklist that gets applied consistently to all client FortiGates. This ensures every client gets the same level of protection and reduces the chance of missing something.

> I recently created exactly this—a CIS-based hardening checklist for FortiGate. Started with a configuration that was 25% compliant with CIS, applied hardening, and got to 90% compliance. Documented every step. That's the kind of repeatable standard I'd build here."

**Why this answer works:**
- Systematic (not ad-hoc)
- References industry standards (CIS)
- Includes documentation
- Mentions testing/rollback (professional)
- References your lab work

---

## CATEGORY 3: FIREWALL RULE AUDITING & LEAST PRIVILEGE

---

### Q6: "How do you identify overly permissive firewall rules? Give me an example"

**Your Answer:**
> "Overly permissive rules are a big security risk. I look for patterns like:

> **Red Flags:**
> 1. `set service 'ALL'` — allows any protocol/port
> 2. `set destination 0.0.0.0/0` — allows destination anywhere
> 3. Source or destination is too broad (`10.0.0.0/8` instead of `10.0.1.0/24`)
> 4. Rules that haven't been reviewed in years (no change log)
> 5. Orphaned rules (policy references a server that no longer exists)

> **Real Example:**
> I audited a FortiGate and found:
>   Policy 5: DMZ servers → WAN with service 'ALL'
>   Translation: 'NGINX servers can connect anywhere on the internet with any protocol'
>   Risk: If NGINX is compromised, attacker can exfiltrate data, download malware, pivot to other networks
>   Actual need: NGINX only needs DNS (port 53) and NTP (port 123)

> I created a change ticket:
>   Before: 1 rule allowing ANY service
>   After: 2 specific rules (DNS, NTP) + 1 implicit deny
>   Testing: Verified NGINX still works, but can't reach random IP:random port
>   Risk reduction: If NGINX compromised, attacker is contained

> I documented this like an MSP would:
>   - Risk assessment
>   - Before/after state
>   - Testing results
>   - Rollback plan (if something goes wrong)
>   - Approval from stakeholder

> This is exactly what I'd do here—systematically audit all clients' firewall rules, identify risks, document and implement fixes."

**Why this answer works:**
- Shows you understand least privilege
- Real example (not theoretical)
- Demonstrates proper documentation
- Explains risk/benefit tradeoff

---

### Q7: "A client's firewall has 200 rules and traffic seems slow. How would you audit and clean this up?"

**Your Answer:**
> "Rule bloat is a real problem. 200 rules is likely 150+ redundant or unused. Here's my approach:

> **Phase 1: Inventory**
> - Export all rules
> - Document purpose of each (interview stakeholders if unclear)
> - Identify duplicates (rules that do the same thing)
> - Flag rules with no recent hit count (might be unused)

> **Phase 2: Analyze Hit Counts**
> - FortiGate logs every matched rule
> - If a rule has zero hits in 6 months, it's probably not needed
> - But verify before deleting (might be fallback rule that rarely triggers)

> **Phase 3: Consolidate**
> - Combine similar rules (instead of 5 rules for client A, 1 rule for 'all clients')
> - Remove redundant rules (Rule 5 + Rule 7 do the same thing? Keep one, delete other)
> - Remove rules for decommissioned systems

> **Phase 4: Reorganize**
> - Sort rules logically (critical flows first, then less-critical, then deny-all at end)
> - Add comments (why does this rule exist? Who approved it?)
> - Make rules scannable (naming convention: 'Client-A-HTTPS-In', not 'Rule-47')

> **Result Example:**
> Before: 200 rules, confusing order, many unnamed
> After: 80 rules, well-organized, documented
> Performance gain: Firewall processes fewer rules → slightly faster (but bigger gain is maintainability)

> **Why it matters:** Fewer rules = lower chance of misconfiguration = higher security. Plus easier for future admins to understand and maintain.

> I'd approach this professionally:
> - Test changes in non-prod first (if possible)
> - Do it in maintenance window
> - Have rollback plan
> - Document impact on client
> - Report findings to management"

---

## CATEGORY 4: VPN & ZERO TRUST NETWORK ACCESS

---

### Q8: "Describe your VPN experience. Site-to-site? Remote access? VPN solutions you've used?"

**Your Answer:**
> "I have hands-on VPN experience across multiple architectures:

> **Site-to-Site VPN:**
> - At Huawei, I designed IPSec VPN tunnels connecting carrier offices across countries
> - Used BGP for dynamic routing (if one tunnel fails, traffic reroutes through backup)
> - Managed IKEv2 encryption, PFS (perfect forward secrecy), redundancy
> - Real example: Office A (10.0.1.0/24) ↔ Office B (10.0.2.0/24) connected via VPN; users transparent to the setup

> **Remote Access VPN:**
> - At OpenVPN Inc., managed remote workers connecting via OpenVPN/SSL-VPN
> - Configured certificate-based and 2FA authentication
> - Built connection pooling and load balancing across multiple VPN gateways
> - Managed HA (high availability) with Keepalived for zero-downtime failover

> **VPN Platforms:**
> - OpenVPN (open-source, very flexible, what I know deeply)
> - FortiGate built-in VPN (simple, good for SMB)
> - IPSec (Cisco, standard for site-to-site)

> **Current Evolution—Zero Trust Network Access:**
> I understand the shift from traditional VPN to ZTNA. Traditional VPN says 'if you're in the tunnel, trust everything.' ZTNA says 'verify every request: who are you, what's your device health, what do you have permission to access?'

> I recently analyzed this for my lab: compared traditional OpenVPN (client connected → access entire subnet) vs. ZTNA approach (client connected → can only access specific apps based on role). Documented the tradeoffs and created a migration plan.

> For clients still on legacy VPN, I can help migrate to modern ZTNA. For clients on ZTNA already, I can optimize policies and troubleshoot connectivity."

**If they ask "Which ZTNA solution?"**
> "I've worked with Tailscale for small deployments (easiest to set up). I'm also familiar with principles behind Okta Identity Engine, Azure AD conditional access, and enterprise solutions. Each has tradeoffs—Tailscale is simplest, enterprise solutions are more powerful but complex.

> I'm not wedded to any specific platform. What matters is understanding the principles: authentication (who are you), device compliance (is your device secure), and policy (what can you access). The tooling is secondary."

---

### Q9: "Tell me about a time you troubleshot a VPN connectivity issue"

**Why they ask:**
- VPN is critical; issues need fast resolution
- Assessing troubleshooting methodology

**Your Answer (Example):**
> "At OpenVPN Inc., a customer complained: 'My remote workers can't connect to the VPN.' Classic ambiguous report. Here's how I troubleshot:

> **Step 1: Gather Data**
> - Affected: How many users? All users or specific users?
> - When: Just started today? Happened after an update?
> - Error message: What does the client show?
> - In this case: All users, started 2 hours ago, error = 'Connection refused'

> **Step 2: Check Infrastructure**
> - Is VPN gateway up? (yes, running)
> - Is gateway listening on port? (yes, netstat shows port 443 listening)
> - Is gateway receiving client connections? (yes, firewall logs show connection attempts)

> **Step 3: Check Gateway Logs**
> - Tail -f /var/log/openvpn/gateway.log
> - Found: 'Cipher mismatch—client using AES-128, gateway configured AES-256'

> **Root Cause:** Gateway had been updated to AES-256 cipher, but clients were still using old version with AES-128. Mismatch = connection refused.

> **Fix:**
> - Downgrade gateway back to AES-128 (temporary fix, restores connectivity immediately)
> - Update all client software to AES-256
> - Schedule coordinated migration
> - Time to resolve: 5 minutes (users restored) + 1 day (all clients updated)

> **Key Lesson:** VPN issues are often configuration mismatches, not hardware. Systematic approach: check infrastructure → check logs → find mismatch → fix quickly → plan permanent solution.

> For this MSP role, this is exactly the kind of rapid troubleshooting I'd do for clients. Fast, methodical, minimal downtime."

---

## CATEGORY 5: ZERO TRUST & MODERN SECURITY CONCEPTS

---

### Q10: "What does 'Zero Trust Network Access' mean to you? How would you implement it?"

**Your Answer:**
> "Zero Trust is a fundamental shift in security philosophy.

> **Traditional Model:** 
> 'If you're inside the network perimeter, trust everything.'
> Problem: If someone gets credentials or a device is compromised, they have free access to everything.

> **Zero Trust Model:**
> 'Trust nothing by default. Verify every request: who is the user, what device are they on, is the device secure, do they have permission to access this specific resource?'

> **Key Components of ZTNA:**
> 1. **Identity Verification:** MFA (multi-factor auth), not just password
> 2. **Device Verification:** Check if device is patched, firewall on, no malware
> 3. **Fine-Grained Access:** Not 'user gets access to entire subnet,' but 'user gets access to app X on port Y only'
> 4. **Continuous Verification:** Re-check every request, not just at login
> 5. **Least Privilege:** Default deny, explicit allow for what's needed

> **Implementation Example (from my lab):**
> Traditional VPN: Client connects → has access to 10.0.0.0/24 (entire network)
> ZTNA: Client (Kenneth, laptop, patched, MFA) → can access NGINX on port 443 only

> **For this MSP:**
> - Help clients understand ZTNA (education, not just tooling)
> - Assess readiness (do they have identity provider like Azure AD?)
> - Plan migration (usually phased: ZTNA + old VPN in parallel first)
> - Implement access policies (role-based: admin → different access than user)
> - Support ongoing management (adjust policies as roles change)

> I'm not just implementing tools. I'm helping clients think about security differently. That's valuable for an MSP."

---

## CATEGORY 6: CIS BENCHMARKS & HARDENING FRAMEWORKS

---

### Q11: "What are CIS Benchmarks and why do they matter?"

**Your Answer:**
> "CIS (Center for Internet Security) Benchmarks are best-practice standards for hardening devices and systems. They're widely used because they're:

> 1. **Vendor-neutral:** Not Fortinet or Palo Alto telling you 'here's our way'—it's industry consensus on what's secure
> 2. **Evidence-based:** Each recommendation has security rationale (why do this?)
> 3. **Practical:** Not theoretical—these are real hardening steps you can apply immediately
> 4. **Audit-friendly:** Many compliance frameworks (SOC 2, ISO 27001, etc.) reference CIS

> **CIS Benchmark Levels:**
> - Level 1: Essential hardening, most organizations can implement
> - Level 2: Stricter hardening, may impact usability or require more management

> **Example—FortiGate CIS Benchmark topics:**
> - Authentication (strong passwords, MFA, limited admin accounts)
> - Encryption (disable weak TLS, enforce strong ciphers)
> - Logging (enable audit logs, send to central server)
> - Access control (least privilege policies)
> - Updates (keep firmware current)

> **Why it matters for MSP:**
> Clients ask: 'Is our firewall secure?' Without a standard, you can't answer objectively. CIS Benchmarks give you a measuring stick.

> I recently created a CIS-based hardening checklist for FortiGate:
> - Started: 25% compliant
> - Applied hardening
> - Result: 90% compliant
> - Documented every step
> - This is the kind of repeatable standard I'd build for all clients here."

---

### Q12: "How would you build a hardening standard for multiple clients with different needs?"

**Your Answer:**
> "Different clients, different needs, but you need consistency. My approach:

> **Step 1: Create a Base Standard**
> - Start with CIS Benchmark Level 1 (essential hardening everyone should have)
> - Document: 'every client firewall will have password policy X, TLS 1.2 minimum, admin access restricted'
> - This is non-negotiable for all clients

> **Step 2: Create Client Tiers**
> - Tier 1 (High security): Financial, healthcare, critical infrastructure
>   * CIS Level 2 + additional controls
> - Tier 2 (Standard): Most clients
>   * CIS Level 1 + best practices
> - Tier 3 (Budget-conscious): Small businesses with limited resources
>   * Minimum security posture (but still Level 1 base)

> **Step 3: Create Checklists**
> - For each tier, create a step-by-step checklist
> - Checklist includes: what to configure, expected result, how to test, rollback plan
> - Make it repeatable (engineer A and engineer B can apply and get same result)

> **Step 4: Document Tradeoffs**
> - Some hardening measures impact performance (logging everything is secure but adds overhead)
> - Document: 'For clients with <100 users, enable all logging. For clients with >1000 users, sample logs at 10% to reduce overhead.'

> **Step 5: Version Control & Review**
> - Your hardening standard should evolve (new CVEs discovered, new best practices)
> - Version your checklist ('FortiGate Hardening v2.3, updated Jan 2025')
> - Review annually or when significant CVEs are discovered

> **Outcome:**
> - Consistency: All clients get secure baseline
> - Efficiency: New engineer can onboard client in 2 hours (follow checklist)
> - Compliance: Checklists are audit trail (we hardened to CIS v2.0)

> This is exactly what I'd build here—a scalable, auditable approach to hardening that works for 20 clients or 200 clients."

---

## CATEGORY 7: DOCUMENTATION & CHANGE MANAGEMENT

---

### Q13: "This role involves a lot of documentation (PSA, change tickets, etc.). How do you approach documentation?"

**Why they ask:**
- MSPs live and die by documentation
- One engineer's config change needs to be understandable to the next engineer
- Audits require proof of work

**Your Answer:**
> "Documentation is critical for MSPs. I treat it as seriously as the technical work itself.

> **What I document:**
> 1. **Change Tickets** (every change to client environment)
>    - Before state: What was it? (config, settings, version)
>    - Change: What did I do and why?
>    - After state: What is it now?
>    - Testing: What did I verify?
>    - Rollback plan: If something goes wrong, how do I undo?

> 2. **Scan Reports** (vulnerability and remediation work)
>    - Scan date, scope (which systems scanned)
>    - Findings summary: X vulnerabilities, Y critical, Z high, etc.
>    - Remediation actions: What was fixed, what was deferred
>    - Re-scan results: Verification that fixes worked

> 3. **Hardening Checklists** (repeatable standards)
>    - Step-by-step instructions
>    - Expected results (after step X, firewall should show Y)
>    - Testing procedures
>    - Screenshots (before/after)

> 4. **Troubleshooting Notes** (for future reference)
>    - Problem: User complaint or alert triggered
>    - Root cause: What investigation found
>    - Solution: Steps taken to fix
>    - Prevention: How to avoid in future

> **Example from my work:**
> Firewall rule audit I did:
> - Document: Detailed risk analysis (overly permissive rules = X risk)
> - Before: 200 rules, 5 with 'allow ALL'
> - Action: Consolidated to 80 specific rules
> - Testing: Verified client apps still work
> - After: 80 rules, no 'allow ALL,' proper least privilege
> - Rollback: If something breaks, restore from backup config

> **Why this matters:**
> - For audits: 'Show me you hardened this client's firewall.' I have the ticket.
> - For client trust: 'What did you do?' I have detailed before/after.
> - For team: If I'm on vacation and another engineer needs to fix something, they can read my notes and understand.

> I use PSA systems well (Zendesk, Jira, LabTech, etc.) and keep detailed notes in each ticket. Every ticket has context: who authorized it, what was changed, how we verified it."

---

### Q14: "How do you balance speed with thoroughness in your work?"

**Why they ask:**
- MSPs are busy; clients want fast turnaround
- But mistakes (misconfigured firewall rule, premature deployment) are expensive
- Assessing your judgment

**Your Answer:**
> "It's a tension, but I err toward thoroughness with good speed.

> **High-Urgency Issues (Security Breach, Outage):**
> - Immediate action: Contain the issue (if ransomware detected, isolate network)
> - Quick verification: Test the fix works
> - Detailed documentation after (during incident, you don't have time)
> - Inform stakeholders: 'Here's what we did, detailed report tomorrow'

> **Normal Changes (Hardening, Rule Audit, Patch Deployment):**
> - Proper process: Plan → test in non-prod → execute → verify → document
> - Takes 1-2 days per client, not 1 hour
> - But eliminates mistakes later
> - Example: Hardening FortiGate might take 3 hours to plan/test/document properly. Rushing takes 30 min and breaks something. Bad ROI.

> **Routine Tasks (Software updates, routine scans):**
> - Can be faster: Update-check-document
> - Familiar process, less risk
> - Still documented but less elaborate

> **My approach:**
> For every job, I ask: 'What's the blast radius if this goes wrong?'
> - High blast radius (firewall rule affects 1000 users): thorough
> - Low blast radius (updating one VLAN on non-critical switch): faster

> MSP clients pay for both speed AND reliability. I deliver both by being smart about where to take shortcuts."

---

## CATEGORY 8: EXPERIENCE & BACKGROUND

---

### Q15: "Walk me through your background. Why are you interested in this role?"

**Your Answer:**
> "I've spent 13+ years in network security and infrastructure, moving from support roles to senior engineering. My path:

> **Early Career (Huawei Technologies, NOC):**
> - Monitored carrier-grade networks, handled alarms
> - Escalated to team leads for complex issues
> - Built strong foundation in networking fundamentals

> **Mid-Career (Globe Telecom, L3 Support):**
> - Advanced to level 3 (L3) support—solving complex issues
> - Designed network solutions (BGP, OSPF, MPLS)
> - Led team members through troubleshooting
> - Earned CCNP Security (hands-on Cisco security expertise)

> **Recent (OpenVPN Inc., Senior Engineer):**
> - Moved into cloud infrastructure (AWS, Azure, GCP)
> - Designed and managed VPN infrastructure for thousands of users
> - Worked on advanced infrastructure like load balancing, redundancy, security hardening

> **Why This Role Now:**
> I've done deep technical work in networking and security. What I'm looking for now is the ability to systematically improve security posture across multiple organizations. An MSP role offers exactly that:
> - Variety: Different clients = different networks = continuous learning
> - Impact: Hardening improves not just one company but many
> - System building: Create repeatable standards (checklists, baselines) that scale
> - Collaboration: Work with a team of engineers vs. solo

> I'm particularly interested in the vulnerability remediation aspect. I've used Qualys extensively, and I understand the workflow: scan → prioritize → remediate → verify. This role lets me do that systematically for many clients.

> I'm also excited about ZTNA. That's the direction the industry is moving, and I want to be part of helping clients make that transition. I've already analyzed it in my lab.

> So I'm looking for a role where I can bring 13+ years of hands-on experience, security certifications (CCNP), and enthusiasm for building better infrastructure. An MSP that values documentation and quality fits that perfectly."

**Why this answer works:**
- Shows progression (not just static experience)
- Articulates why you're interested (not just 'job posting looked good')
- References your relevant background
- Mentions certifications

---

## COMMON BEHAVIORAL QUESTIONS

---

### Q16: "Tell me about a time you disagreed with a decision. How did you handle it?"

**Why they ask:**
- MSPs need people who speak up if something is wrong
- But also people who are professional about disagreement

**Your Answer (Example):**
> "At Globe Telecom, management wanted to deploy a firewall update immediately, without testing. My concern: the update was known to have issues with a specific VLAN configuration we used extensively.

> What I did:
> - Researched: Found bug reports on the vendor forum (not just my guess)
> - Documented: Created a summary showing the risk
> - Proposed solution: 'Test in our development environment first (4 hours). If it works, deploy. If it breaks, stay on current version and plan next update.'
> - Presented professionally: Didn't say 'you're wrong,' said 'here's risk we should mitigate'

> Result: Management agreed to test. Testing found the issue, we waited for next patch, deployment went smoothly.

> Key: I didn't just complain. I did the research, proposed a solution, and framed it as 'how can we succeed' not 'you're wrong.' That's professional disagreement."

---

### Q17: "Tell me about a time you made a mistake. What did you learn?"

**Why they ask:**
- Everyone makes mistakes; how you handle it matters
- Do you hide it or own it?

**Your Answer (Example):**
> "Early in my career at Huawei, I made a firewall rule change in production without testing it first (rushing, bad judgment). The rule broke DNS lookups for an entire office.

> What happened:
> - Users complained: 'Can't reach anything'
> - I panicked initially
> - But I quickly rolled back the rule and DNS was restored (10-minute outage)

> What I learned:
> - Always test changes in non-prod first, even if 'this will be quick'
> - 'Quick' mistakes cost more than 'slow' correctness
> - The extra 30 minutes to test saves hours of firefighting later

> Since then:
> - I follow change management discipline: plan → test → execute → verify → document
> - I never skip the testing step, no matter the pressure
> - I'm proactive about mistakes: if something doesn't feel right, I speak up vs. hoping it works

> This is why I'm careful and thorough now. That mistake taught me the value of process."

**Why this works:**
- You owned the mistake (didn't blame others)
- Showed you learned
- Demonstrated improvement (you follow process now)

---

## CATEGORY 9: TECHNICAL DEPTH QUESTIONS

---

### Q18: "What's the difference between TCP and UDP? When would you use each in a network?"

**Your Answer:**
> "TCP and UDP are transport layer protocols with different characteristics:

> **TCP (Transmission Control Protocol):**
> - Reliable: Guarantees delivery (receiver gets every packet)
> - Ordered: Packets arrive in order sent
> - Connection-oriented: Establishes connection before sending data (handshake)
> - Overhead: Larger headers, acknowledgments
> - Use cases: Email (SMTP), web (HTTP/S), file transfer (FTP), SSH

> **UDP (User Datagram Protocol):**
> - No guarantee: Packets might be lost (doesn't care)
> - Unordered: Packets might arrive out of order
> - Connectionless: Just sends data (no handshake)
> - Low overhead: Smaller headers, faster
> - Use cases: VoIP, streaming video, online gaming, DNS, NTP

> **Real Example:**
> - Email: Must use TCP. If packet is lost, message doesn't arrive (bad!)
> - Video call: UDP acceptable. If 1 frame is lost out of 30 fps, user doesn't notice
> - Gaming: UDP. If one packet lost, game might skip slightly, but real-time communication is important

> **For Firewalls:**
> When creating rules, I specify which protocol:
> - Allow TCP 443 (HTTPS)
> - Allow UDP 53 (DNS)
> - Block UDP 445 (SMB over UDP, unusual, potential attack)

> This is why 'allow ALL services' in a firewall rule is so bad—allows TCP and UDP for everything."

---

### Q19: "Explain what VLAN is and why you'd use it"

**Your Answer:**
> "VLAN = Virtual LAN. It's a way to logically segment a network even though it uses the same physical infrastructure.

> **How it works:**
> - Physical: All devices on same switch (one broadcast domain)
> - VLAN: Divide that switch into multiple virtual networks (separate broadcast domains)
> - Example: Same physical switch handles:
>   * VLAN 1: Employees (10.0.1.0/24)
>   * VLAN 2: Guests (10.0.2.0/24)
>   * VLAN 3: Servers (10.0.3.0/24)
>   * Traffic between VLANs blocked by default (firewall must allow it)

> **Why use VLANs:**
> 1. **Security:** Guests can't see employee devices. Servers isolated from workstations.
> 2. **Broadcast reduction:** Each VLAN has its own broadcast domain (smaller overhead)
> 3. **Network management:** Easier to manage groups of devices
> 4. **Compliance:** Some standards require network segmentation (PCI requires separating payment systems)

> **For Firewalls:**
> When I design policies:
> - VLAN 1 → VLAN 2: No traffic (guests isolated)
> - VLAN 1 → VLAN 3: HTTPS only (employees can reach servers securely)
> - VLAN 3 → Internet: DNS and NTP only (servers limited)

> VLANs and firewall rules together enforce security through segmentation. A typical hardening task is: 'Make sure each VLAN only talks to what it needs.'"

---

### Q20: "What's an access list (ACL) and how do you use it?"

**Your Answer:**
> "ACL = Access Control List. It's a set of rules that control which traffic is allowed or denied on a device (router, firewall, switch).

> **Basic Example:**
> ```
> ACL 1:
>   permit TCP 10.0.1.0 0.0.0.255 any eq 443   # Allow HTTP/S from employees
>   permit UDP 10.0.1.0 0.0.0.255 any eq 53    # Allow DNS from employees
>   deny any                                     # Deny everything else (implicit)
> ```

> **Key Concepts:**
> - Evaluated top-to-bottom (first match wins)
> - Implicit deny at end (if no rule matches, traffic is blocked)
> - Specific before general (more specific rules should be higher)

> **Real Example (FortiGate):**
> I audited a client's firewall and found their ACLs were a mess:
> - 200 rules mixed together (old and new)
> - Some rules contradicted (rule 50 allows something, rule 150 denies same thing)
> - Unclear order (could fail in unexpected ways)

> I reorganized:
> - Group similar rules (all client A traffic together)
> - Ordered properly (critical flows first, deny-all at end)
> - Added comments (why does this rule exist?)
> - Tested: Verified no change in actual traffic behavior

> Result: Cleaner, easier to audit, easier to maintain.

> **MSP context:**
> This is part of the firewall rule audit I'd do for clients. ACLs aren't just about security—they're about maintainability."

---

## QUESTIONS TO ASK THEM (At end of interview)

---

### Q21: "What does success look like in this role? How is performance measured?"

> Understand what they prioritize: ticket volume, client satisfaction, security improvements, etc.

### Q22: "What's the biggest security challenge you see across your client base right now?"

> Shows you're interested in real problems, not just generic job

### Q23: "What tools do you use for vulnerability scanning? Can I work on different tools or are you locked in?"

> Important to know if you're locked into one tool (Qualys, Nessus, Rapid7, etc.)

### Q24: "What's the typical career path here? Can someone grow from this role into management or architecture?"

> Assessing if this is a dead-end or growth opportunity

### Q25: "How much US business hours overlap is required? I see minimum 4 hours listed—what does that look like practically?"

> You're in Baguio (Asia), they're US Central. Clarify overlap expectations.

---

## RED FLAGS TO AVOID

---

**Don't say:**
- ❌ "I'll just use Google if I don't know something" (Show you have knowledge base, not guessing)
- ❌ "I don't like documentation" (MSPs care deeply about this)
- ❌ "I've never actually implemented this, just studied it" (They want hands-on, not theoretical)
- ❌ "I'm only interested if it pays X amount" (Leads with money, not interest in work)
- ❌ "I'm not really interested in security, just looking for a job" (Be genuinely interested)

**Do say:**
- ✅ "I've used X tool extensively, and I'm familiar with Y as well"
- ✅ "I document all my work in detail—here's an example from my lab"
- ✅ "I recently hardened my lab network and applied CIS benchmarks"
- ✅ "I'm interested in learning new tools as long as the security principles are sound"
- ✅ "I'm excited about this role because of the variety and impact"

---

## INTERVIEW STRATEGY

**Day of Interview:**

1. **First 5 min:** Establish rapport, show enthusiasm
2. **Middle 20 min:** Let them see your expertise (specific examples, lab work)
3. **Last 10 min:** Ask good questions (shows you're interested)
4. **Throughout:** Be honest (don't exaggerate, but do highlight your strengths)

**Technical Interview (if they ask you to show something):**

- Walk them through your vulnerability scan + remediation (OpenVAS findings)
- Show your firewall rule audit documentation
- Explain your CIS hardening checklist
- Discuss your ZTNA analysis

**After Interview:**

- Send thank you email within 24 hours
- Reference specific things you discussed (shows you were paying attention)
- Reiterate interest

---

## FINAL TIPS

1. **Lead with lab work:** "I've built a lab to stay current with these exact skills..."
2. **Reference Qualys experience:** You have real vulnerability management experience—that's valuable
3. **Be specific:** "I hardened FortiGate to CIS v2.0" beats "I've done hardening"
4. **Show documentation:** MSPs live by documentation. Demonstrate you can do it well
5. **Express genuine interest:** They'll feel if you're genuinely interested in security vs. just looking for work

---

**You're a very strong candidate for this role.** 13+ years, CCNP Security, FortiGate, VPN, lab work, Qualys experience. 

Good luck! 🚀
