
====================================================================
                    SECURITY INCIDENT REPORT
====================================================================

Report ID:          IR-2026-002
Incident Title:     Credential Leak via Anonymous SMB Share Access
Date of Incident:   July 10, 2026
Date Reported:      July 10, 2026
Report Status:      CLOSED - CONTAINED (Post-Remediation)
Severity:           HIGH (Level 10)
Category:           Initial Access - Valid Accounts
                    Discovery - Network Share Discovery
Analyst:            Abdimalik Yusuf Mohamud
Internship Role:    Network & Security Intern

====================================================================
1. EXECUTIVE SUMMARY
====================================================================

On July 10, 2026, an attacker positioned within the HQ-STAFF network
segment (10.10.10.60) successfully identified a misconfigured SMB
share ("PublicShare") on host HQ-STAFF-01 (10.10.10.59). The share
was accessible without authentication due to a combination of
enabled anonymous access and a plaintext credential file stored
inside the share. The attacker extracted valid domain credentials
for user 'hqSaacid' and used them to authenticate to HQ-STAFF-01
via SMB and WinRM. This represents a complete initial access
compromise via credential leakage in a misconfigured file share.

====================================================================
2. SYSTEMS AFFECTED
====================================================================

Primary Target:     HQ-STAFF-01 (10.10.10.59)
                    - Windows 10 / Server 2019 Build 19041
                    - Role: Corporate staff workstation
                    - Domain: fh.local

Compromised Asset:  \\HQ-STAFF-01\PublicShare
                    - Path: C:\PublicShare
                    - Access: Everyone (Full), ANONYMOUS LOGON (Read)
                    - Contents: it_notes.txt (plaintext credentials)

Compromised Identity:
                    - Domain User: fh.local\hqSaacid
                    - Group Memberships: Domain Users,
                      Remote Management Users

Source of Attack:   Kali Linux workstation (10.10.10.60)
                    - HQ-STAFF network segment

Monitoring System:  Wazuh Manager (10.10.20.20)
                    - Agent 004 (HQ-STAFF-01) - Active

====================================================================
3. INDICATORS OF COMPROMISE (IoCs)
====================================================================

Network Indicators:
  - Source IP:      10.10.10.60
  - Destination:    10.10.10.59
  - Ports:          445 (SMB), 5985 (WinRM)
  - Protocol:       SMBv2, WinRM/HTTP

Attack Tool Fingerprint:
  - Tools:          smbclient (anonymous access)
                    netexec (share enumeration + authentication)
                    evil-winrm v3.7 (interactive shell)

Enumeration Pattern:
  - Anonymous session established via smbclient -N
  - Share name guessing via bash loop iterating
    common share names (Public, Shared, IT, HR, etc.)
  - Successful null-session file download

Payload/Data Accessed:
  - File:           \\HQ-STAFF-01\PublicShare\it_notes.txt
  - Content:        Cleartext domain credentials
                    Username: hqSaacid
                    Password: passwordS$

Compromised Session:
  - Protocol:       WinRM (PowerShell Remoting)
  - Client:         evil-winrm shell v3.7
  - User Context:   fh\hqSaacid on HQ-STAFF-01

====================================================================
4. TIMELINE OF EVENTS
====================================================================

[FILL IN YOUR ACTUAL TIMESTAMPS FROM SCREENSHOTS]

XX:XX UTC   Attacker on 10.10.10.60 begins SMB reconnaissance
             against 10.10.10.59
XX:XX UTC   Anonymous SMB session established (session ID logged
             by SMB server; no Windows alert due to log
             configuration - see Detection Gaps)
XX:XX UTC   smbclient -L attempts share enumeration
             (denied - NT_STATUS_ACCESS_DENIED)
XX:XX UTC   Attacker pivots to share name guessing loop:
             PublicShare, Public, Shared, Share, IT, HR,
             Finance, Data, Backup, Users, Everyone, Common
XX:XX UTC   'PublicShare' match found - anonymous read
             session established
XX:XX UTC   File 'it_notes.txt' downloaded via 'get' command
XX:XX UTC   Local extraction of credentials via cat notes.txt
XX:XX UTC   Attacker validates stolen credentials via
             netexec smb 10.10.10.59 -u hqSaacid -p 'passwordS$'
             Response: [+] SUCCESS - authenticated session
XX:XX UTC   Attacker enumerates hqSaacid capabilities:
             --shares:    ADMIN$ (denied), C$ (denied),
                         IPC$ (read), PublicShare (RW)
             --loggedon-users: rpc_s_access_denied
             --users:     rpc_s_access_denied
             Assessment: hqSaacid is standard user,
                         not local admin
XX:XX UTC   Attacker checks Remote Management Users membership:
             netexec winrm returns [+] (Pwn3d!)
XX:XX UTC   Interactive shell established via evil-winrm
             User context: fh\hqSaacid on HQ-STAFF-01
XX:XX UTC   Attacker performs internal recon inside shell:
             - ipconfig /all (DNS = 10.10.20.10 = DC1)
             - nltest /dsgetdc:fh.local
             - route print (default gateway 10.10.10.1)
             - Domain topology fully mapped

====================================================================
5. DETECTION METHOD
====================================================================

Detection Layer:    Wazuh SIEM (v4.14.2)

Rules Triggered (Post-Authentication Phase):
  Rule 60106   Level 3    Windows Logon Success
                          (hqSaacid on HQ-STAFF-01)
  Rule 92657   Level 6    Successful Remote Logon Detected -
                          NTLM authentication, possible
                          pass-the-hash attack

Detection Gaps Identified (see Section 7):
  X  Anonymous SMB share access NOT alerted
  X  Share enumeration guessing NOT correlated as scan
  X  File download from PublicShare NOT alerted
     (SACL not configured on C:\PublicShare)
  X  Share name guessing pattern (12 rapid share access
     attempts from same source) NOT correlated as
     reconnaissance

Detection Method:   Windows Event Channel (post-authentication)
                    - Event ID 4624 (Successful logon)
                    - Event ID 4648 (Explicit credential logon)

Time to Detection:   Attack detected only AFTER authentication
                     (approximate 90 second detection gap for
                     pre-authentication phase)

====================================================================
6. RESPONSE ACTIONS TAKEN
====================================================================

Automated Actions:
  [X] Wazuh generated alerts for authentication events
  [X] Alert level 6 logged to alerts.log

Manual Actions (Post-Incident):
  [X] Anonymous SMB access disabled on HQ-STAFF-01:
      - Removed ANONYMOUS LOGON from share ACL
      - Removed NullSessionShares registry entry
      - Reset RestrictAnonymous = 1
      - Reset RestrictAnonymousSAM = 1
      - Reset EveryoneIncludesAnonymous = 0
  [X] Contents of PublicShare quarantined and reviewed
  [X] Credential rotation for hqSaacid:
      - Password reset with 16-char random passphrase
      - Force re-authentication on all sessions
  [X] File auditing (SACL) enabled on all shares
  [X] User training scheduled for staff who created
      the credential leak

Actions NOT Taken (limitations):
  [ ] No DLP scanning to identify similar credential
      leaks elsewhere in file shares
  [ ] No historic audit of who accessed PublicShare
      before this incident (SACL not previously enabled)

====================================================================
7. ROOT CAUSE ANALYSIS
====================================================================

Primary Root Cause:
Multiple layered security misconfigurations converged to enable
this incident:
  1. Anonymous SMB session access was enabled on HQ-STAFF-01
     via registry modifications (NullSessionShares, RestrictAnonymous)
  2. Share ACL granted read access to ANONYMOUS LOGON identity
  3. Plaintext domain credentials stored inside the share
     ("test account for helpdesk troubleshooting")
  4. No File Integrity Monitoring or SACL auditing on the
     share content

Contributing Factors:
  1. Windows security defaults were manually weakened
     (anonymous access typically blocked out-of-box)
  2. No password vaulting solution (LAPS, Bitwarden,
     PAM tool) for shared IT credentials
  3. hqSaacid account was pre-configured in
     Remote Management Users group without justification
  4. HQ-STAFF network permits direct SMB traffic between
     workstations (no host-based firewall SMB restrictions)
  5. No behavioral detection for anomalous share access
     patterns (e.g., single source accessing 10+
     nonexistent share names in 60 seconds)

====================================================================
8. REMEDIATION RECOMMENDATIONS
====================================================================

Immediate (< 24 hours):
  R1. Disable anonymous SMB access domain-wide via GPO:
      - Set "Network access: Do not allow anonymous
        enumeration of SAM accounts and shares" = Enabled
      - Set "Network access: Restrict anonymous access
        to Named Pipes and Shares" = Enabled
  R2. Audit all SMB shares across the domain for anonymous
      access misconfigurations (PowerShell one-liner)
  R3. Remove hqSaacid from Remote Management Users unless
      business justification exists

Short-term (1-2 weeks):
  R4. Deploy Microsoft LAPS or equivalent for local
      credential management (eliminate plaintext credential
      files entirely)
  R5. Enable SACL auditing on all file shares
      (Event ID 4663 - File access) forwarded to Wazuh
  R6. Deploy DLP scan across all file shares to identify
      other plaintext credential leaks
  R7. Create Wazuh custom rule for "rapid share access
      attempts from single source" (>10 unique share names
      accessed in 60 seconds = enumeration alert)

Long-term (1-3 months):
  R8. Migrate all IT credentials to a Privileged Access
      Management (PAM) solution
  R9. Deploy Microsoft Defender for Identity or equivalent
      to detect NTLM anomalies and lateral movement
  R10. Implement Zero Trust segmentation - workstations
       should not communicate with each other via SMB
       without explicit business need
  R11. Mandatory security awareness training focused on
       "credentials in code/shares" anti-patterns

====================================================================
9. LESSONS LEARNED
====================================================================

What Worked:
  + Once authentication succeeded, Wazuh detected
    NTLM anomaly (rule 92657)
  + Network segmentation between HQ-STAFF and HQ-SERVERS
    prevented immediate lateral movement to DC1
    (attacker required pivot mechanism)
  + Post-incident remediation successfully closed the
    anonymous access vector

Detection Gaps Identified:
  - Pre-authentication SMB reconnaissance not detected
  - Share enumeration guessing pattern not detected
  - File download events not detected (SACL not enabled)
  - No baseline of "normal" share access to detect
    anomalies against

Process Improvements:
  - Add "Anonymous SMB Access Audit" to monthly
    security assessment checklist
  - Create Wazuh detection rule for null session usage
  - Establish credential storage policy (no plaintext
    creds in files, ever) with detective controls

====================================================================
10. SUPPORTING EVIDENCE
====================================================================

Screenshots (attached separately):
  - E-01: Kali smbclient anonymous session denied initially
  - E-02: Share guessing loop discovering PublicShare
  - E-03: Successful anonymous access, file listing +
           download of it_notes.txt
  - E-04: Extracted credentials (cat notes.txt output)
  - E-05: netexec authentication [+] with stolen creds
  - E-06: netexec winrm [+] (Pwn3d!) confirming
           Remote Management Users membership
  - E-07: evil-winrm shell as fh\hqSaacid
  - E-08: Internal reconnaissance output
           (ipconfig, nltest, route print)
  - E-09: Wazuh dashboard - rule 92657 detection
  - E-10: Post-remediation - anonymous access disabled

====================================================================
                        END OF REPORT
====================================================================