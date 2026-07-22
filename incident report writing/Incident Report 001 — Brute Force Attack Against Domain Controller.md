====================================================================
                    SECURITY INCIDENT REPORT
====================================================================

Report ID:          IR-2026-001
Incident Title:     Brute Force Attack Against Domain Controller (DC1)
Date of Incident:   July 10, 2026
Date Reported:      July 10, 2026
Report Status:      CLOSED - CONTAINED
Severity:           HIGH (Level 10)
Category:           Credential Access - Brute Force
Analyst:            Abdimalik Yusuf Mohamud
Internship Role:    Network & Security Intern

====================================================================
1. EXECUTIVE SUMMARY
====================================================================

At approximately 09:07 UTC on July 10, 2026, Wazuh SIEM detected an
active Kerberos-based brute force attack targeting the Administrator
account on the Domain Controller (DC1.fh.local, 10.10.20.10). The
attack originated from an internal host (10.10.10.60) located in the
HQ-STAFF network segment. Wazuh's automated Active Response system
successfully engaged within seconds of detection, deploying a Windows
Firewall rule on DC1 to block the attacking IP address for the
configured cooldown period. No successful authentication occurred
against the Administrator account during this incident.

====================================================================
2. SYSTEMS AFFECTED
====================================================================

Primary Target:     DC1.fh.local (10.10.20.10)
                    - Windows Server 2025 Build 26100
                    - Role: Active Directory Domain Controller
                    - Domain: fh.local

Source of Attack:   Kali Linux workstation (10.10.10.60)
                    - HQ-STAFF network segment
                    - NOTE: Unauthorized device on internal LAN

Monitoring System:  Wazuh Manager (10.10.20.20)
                    - Wazuh v4.14.2
                    - Agent 001 (DC1) - Active

====================================================================
3. INDICATORS OF COMPROMISE (IoCs)
====================================================================

Network Indicators:
  - Source IP:      10.10.10.60
  - Destination:    10.10.20.10
  - Ports:          88 (Kerberos KDC)
  - Protocol:       TCP/UDP Kerberos AS-REQ

Attack Tool Fingerprint:
  - Tool:           Kerbrute v1.0.3
  - User-Agent:     Kerbrute-Client/Golang-Kerberos
  - Pattern:        Rapid AS-REQ requests targeting single username
                    with iterating password list

Targeted Account:
  - Username:       Administrator
  - Domain:         fh.local

Payload Wordlist Signatures (attempted passwords):
  - Password123, Password1, Company123, P@ssword1
  - Winter2024, Summer2024, Spring2024, Welcome1
  - Admin123, Admin!, Fall2024, Root123, Password!
  - qwerty123, letmein, 15422035s$

====================================================================
4. TIMELINE OF EVENTS
====================================================================

[FILL IN YOUR ACTUAL TIMESTAMPS FROM SCREENSHOTS]

09:07:08 UTC   Kerbrute initiated brute force from 10.10.10.60
09:07:08 UTC   First AS-REQ (Administrator + "Password123") reaches DC1
09:07:08 UTC   Wazuh rule 60104 fires: "Windows audit failure event"
09:07:09 UTC   Multiple 60104 events accumulate
09:07:14 UTC   Wazuh rule 60205 fires: "Multiple Windows audit
                failure events" (Level 10 correlation alert)
09:07:14 UTC   Wazuh Active Response triggered on agent 001 (DC1)
09:07:14 UTC   netsh.exe command executed on DC1
09:07:14 UTC   Windows Firewall inbound block rule created:
                "WAZUH ACTIVE RESPONSE BLOCKED IP" -> 10.10.10.60
09:07:15 UTC   Wazuh rule 92043 fires: "Netsh used to add
                firewall rule" (confirms successful containment)
09:07:15 UTC   Administrator account locked (rule 60115)
09:07:16 UTC   Kerbrute reports "USER LOCKED OUT and safe mode
                on! Aborting..." - attack aborted
09:10:14 UTC   Firewall block auto-expired (180s timeout)

====================================================================
5. DETECTION METHOD
====================================================================

Detection Layer:    Wazuh SIEM (v4.14.2)

Rules Triggered:
  Rule 60104   Level 5    Windows audit failure event
                          (fired multiple times per attempt)
  Rule 60205   Level 10   Multiple Windows audit failure events
                          (correlation rule - primary detection)
  Rule 60115   Level 9    User account locked out
                          (multiple login errors)
  Rule 657     Level 3    Active response executed
  Rule 92043   Level 10   Netsh used to add firewall rule
                          (containment confirmation)

Detection Method:   Log-based (Windows Event Channel Security)
                    - Event ID 4771 (Kerberos pre-authentication
                      failure)
                    - Correlation via same_field: srcip

Time to Detection:  < 5 seconds
Time to Containment: < 6 seconds (fully automated)

====================================================================
6. RESPONSE ACTIONS TAKEN
====================================================================

Automated Actions (Wazuh Active Response):
  [X] Deployed inbound firewall block on DC1 targeting 10.10.10.60
  [X] Alert generated at Wazuh dashboard
  [X] Alert level 10 sent to alerts.log

Manual Actions Taken:
  [X] Incident evidence captured (Wazuh dashboard screenshots)
  [X] Firewall rule creation verified via
      Get-NetFirewallRule -DisplayName "WAZUH*"
  [X] Administrator account confirmed locked out
  [X] Investigation into source host (10.10.10.60) initiated

Actions NOT Taken (limitations noted):
  [ ] No SIEM alert notification to on-call (email not configured)
  [ ] Firewall block auto-expired after 180 seconds; permanent
      block requires manual intervention

====================================================================
7. ROOT CAUSE ANALYSIS
====================================================================

Primary Root Cause:
An unauthorized host on the HQ-STAFF network segment was permitted
to communicate directly with the Domain Controller on Kerberos
port 88. While this traffic is legitimate for domain-joined
workstations, no whitelist enforcement exists to restrict
who can initiate Kerberos authentication attempts.

Contributing Factors:
  1. No account lockout notification to security team
  2. No source-IP whitelisting for domain authentication
  3. Attacker used enumerated username (Administrator) which
     is a well-known account name
  4. Password complexity policy allowed common patterns
     found in public wordlists (Password123, etc.)
  5. Active Response cooldown (180s) allows retry after expiry

====================================================================
8. REMEDIATION RECOMMENDATIONS
====================================================================

Immediate (< 24 hours):
  R1. Investigate source host 10.10.10.60 - determine if
      authorized device or intrusion
  R2. Force password reset on Administrator account (already
      locked, extend lockout to indefinite)
  R3. Extend Active Response cooldown to permanent block for
      repeat offenders (rule 60205 hits from same IP twice)

Short-term (1-2 weeks):
  R4. Enable Kerberos Authentication Service auditing across
      all domain-joined systems (currently DC1-only)
  R5. Deploy Windows Firewall inbound rules restricting
      Kerberos port 88 to known workstation IP ranges only
  R6. Implement account lockout notifications via Wazuh
      email integration (rule 60115 -> email SOC team)
  R7. Rename Administrator account to a non-obvious name to
      slow username enumeration

Long-term (1-3 months):
  R8. Deploy Multi-Factor Authentication (MFA) for
      privileged accounts (Windows Hello, YubiKey, or Azure MFA)
  R9. Deploy password complexity policy enforcing:
      - Minimum 14 characters
      - Common password blacklist (Have I Been Pwned integration)
      - Passphrase encouragement
  R10. Segment HQ-STAFF network from HQ-SERVERS with
       stricter allow-list firewall rules (already implemented
       in Month 2 - verify enforcement)

====================================================================
9. LESSONS LEARNED
====================================================================

What Worked:
  + Wazuh correlation rule (60205) fired within seconds
  + Automated Active Response contained attack without human
    intervention
  + Windows audit policy enabled during Month 2 was critical
    for detection (before enabling, brute force was invisible)
  + Custom rule (60204/60205) linked to netsh Active Response
    provides real-time containment

Detection Gaps Identified:
  - Wazuh default netsh command has srcip extraction issues
    with EventChannel format on some Windows versions
    (Wazuh Issue #10022) - workaround: verify field mapping
  - Active Response cooldown does not persist repeat attackers
  - No secondary alert channel (email/SMS/webhook) for
    high-severity events

Process Improvements:
  - Add "Active Response Verification" to daily SOC checklist
    (query firewall rules for WAZUH prefix daily)
  - Create custom Wazuh dashboard tracking:
    * Active Response deployments per day
    * Top offending source IPs
    * Recurring attack sources

====================================================================
10. SUPPORTING EVIDENCE
====================================================================

Screenshots (attached separately):
  - E-01: Kerbrute execution output from source
  - E-02: Wazuh dashboard - rule 60205 detection
  - E-03: Wazuh dashboard - rule 657 Active Response
  - E-04: DC1 PowerShell - Get-NetFirewallRule showing
           WAZUH ACTIVE RESPONSE BLOCKED IP rule
  - E-05: DC1 Event Viewer - Security log 4771 events

Log Extracts:
  - /var/ossec/logs/alerts/alerts.log (60205 entries)
  - Windows Security Event Log (DC1) - Event IDs 4771, 4740

====================================================================
                        END OF REPORT
====================================================================