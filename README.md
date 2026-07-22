# FashilHack Internship — Enterprise Network, Security & SOC Lab

A two-month, self-directed internship building an enterprise multi-site network from scratch and then attacking it from a red team perspective while defending it as a SOC analyst.

**Intern:** Abdimalik Yusuf Mohamud
**Role:** Network & Security Intern
**Duration:** Month 1 (Infrastructure) + Month 2 (SOC Operations)
**Environment:** Self-directed VirtualBox lab, four Windows machines, two pfSense firewalls, one Ubuntu Wazuh SIEM, one Kali attacker


## What This Repo Demonstrates

Both sides of a modern security career in one project:

**Blue Team / Defense**
- Multi-site network design with proper VLAN segmentation
- Dual pfSense enterprise firewalls with IPsec site-to-site VPN
- Active Directory + DNS deployment
- Wazuh SIEM deployment, custom decoder and rule development, custom dashboards
- Detection engineering across Windows EventChannel + pfSense syslog
- Automated incident response via Wazuh Active Response
- Formal incident report writing (three full IR docs)
- Threat hunting with hypothesis-driven KQL queries

**Red Team / Offense**
- Kerberos-based brute force (`kerbrute`)
- SMB reconnaissance and anonymous share exploitation (`smbclient`, `netexec`)
- Lateral movement via stolen credentials from misconfigured shares
- WinRM-based post-exploitation shells (`evil-winrm`)
- Kerberoasting service accounts (`impacket-GetUserSPNs`)
- Offline hash cracking with mutation rules (`hashcat`)
- DCSync attack against a Domain Controller (`impacket-secretsdump`)
- Pass-the-hash to Domain Admin
- Network pivoting via reverse SOCKS proxy (`chisel` + `proxychains`)
- LOLBin file transfer (`certutil`)

**End state:** starting with zero credentials on an internal network, achieved full Domain Administrator on `fh.local` in one attack chain. Every step was executed against real defenses and every detection (or gap) is documented.


## Repository Structure

```
.
├── README.md                          # This file
├── INTERNSHIP.md                      # Full combined report (Months 1 + 2)
├── month-1/
│   └── fashilhack-intership-month-1.docx
├── month-2/
│   └── fashilhack-intership-month-2.docx
├── full-report/
│   └── full-intership.docx            # Combined Word document
├── incident-reports/
│   ├── IR-2026-001-brute-force.md
│   ├── IR-2026-002-anonymous-smb.md
│   └── IR-2026-003-kerberoast-dcsync.md
├── use-cases/
│   ├── active-response / brute-force  # use case 1
│   ├── lateral-movement/              # Use Case 2
│   └── privilege-escalation/          # Use Case 3
├── hunts/
│   ├── hunt-1-post-brute-force-authentication/
│   ├── hunt-2-kerberoasting-tgs-requests/
│   ├── hunt-3-domain-admins-group-changes/
│   └── hunt-4-repeated-failed-logons/
├── screenshots/                       # Evidence collected across the two months
└── diagrams/
    ├── lab-topology-month-1.png
    └── lab-topology-month-2-attacker.png
```


## The Lab

Two-site enterprise environment simulating a Headquarters and Branch office connected by an IPsec VPN.

**Headquarters (HQ)**
- `HQ_STAFF` (10.10.10.0/24) — corporate users
- `HQ_SERVERS` (10.10.20.0/24) — Active Directory + Wazuh
- `HQ_GUEST` (10.10.30.0/24) — internet-only

**Branch (BR)**
- `BR_STAFF` (10.20.10.0/24) — branch users
- `BR_GUEST` (10.20.30.0/24) — internet-only

**Key systems**
| System | IP | Role |
|---|---|---|
| pfSense-HQ | 10.10.10.1 / 10.10.20.1 | Edge firewall, IPsec VPN, syslog source |
| pfSense-BR | 10.20.10.1 | Branch firewall + VPN endpoint |
| DC1 | 10.10.20.10 | Windows Server 2025 Domain Controller (fh.local) + DNS |
| Wazuh Manager | 10.10.20.20 | Wazuh v4.14.2 on Ubuntu 24.04 |
| HQ-STAFF-01 | 10.10.10.59 | Windows 10 domain-joined workstation |
| Kali Linux | 10.10.10.60 | Attacker workstation (Month 2) |

![Lab Topology](diagrams/lab-topology-month-2-attacker.png)


## Highlights

### Real Segmentation, Real Pivot
pfSense-HQ was configured with strict rules that block the staff subnet from reaching the servers subnet except from a single authorized workstation (HQ-STAFF-01). This forced the Month 2 attack chain to compromise the workstation first and pivot through it via `chisel` reverse SOCKS proxy to reach the Domain Controller. This is not theater — it mirrors how real enterprise attack chains unfold and how real detection is done.

### Custom Detection Engineering
- Written custom Wazuh decoder (`local_decoder.xml`) for pfSense filterlog format that the default decoder does not match
- Written custom Wazuh correlation rules (`local_rules.xml`) — rule 100300 for individual firewall drops and rule 100301 for correlated port scan detection (8+ drops from same source in 60 seconds)
- Custom Authentication Monitoring dashboard with three visualizations

### Automated Containment That Actually Works
Wazuh Active Response configured with `netsh` command targeting rules 60204 (NTLM brute force) and 60205 (Kerberos brute force). When triggered, `netsh.exe` on the affected agent creates an inbound firewall block rule named `WAZUH ACTIVE RESPONSE BLOCKED IP` targeting the attacker's source IP. Validated end-to-end on both DC1 and HQ-STAFF-01.

### Honest About Gaps
Not everything works. Kerberoasting, DCSync, `chisel` tunneling, and `certutil` LOLBin usage were all executed successfully without triggering meaningful alerts. Every gap is documented with a concrete remediation recommendation in [Section 20 of INTERNSHIP.md](INTERNSHIP.md#20-detection-gaps-identified).


## Attack Chain Summary (Month 2)

```
Kali on HQ-STAFF subnet, zero credentials
    │
    ▼
Anonymous SMB session on HQ-STAFF-01 (share name guessing)
    │
    ▼
PublicShare discovered → download it_notes.txt
    │
    ▼
Plaintext creds: hqSaacid:passwordS$
    │
    ▼
Authenticate via WinRM → interactive shell on HQ-STAFF-01
    │
    ▼
Chisel reverse SOCKS tunnel back to Kali
    │
    ▼
Kerberoast svc_backup (via pivot) → crack Summer2024! offline
    │
    ▼
svc_backup has DCSync rights → dump entire NTDS.dit
    │
    ▼
Pass-the-hash as Administrator → NT AUTHORITY\SYSTEM on DC1
    │
    ▼
Domain Compromise (fh.local)
```

Total time from zero to Domain Admin: hours, not days. Every technique documented with tools, commands, screenshots, and detection status.

---

## Skills Demonstrated

**Networking & Infrastructure**
Enterprise firewall configuration, manual outbound NAT, IPsec VPN, packet capture analysis, routing design, Active Directory deployment, DNS troubleshooting, multi-site authentication.

**SIEM & Detection Engineering**
Wazuh deployment, custom decoders (PCRE2 regex), custom correlation rules, log source onboarding (Windows EventChannel, pfSense syslog), threat intelligence enrichment (VirusTotal), dashboard engineering.

**Incident Response**
Automated containment via Active Response, end-to-end response validation, formal incident reporting, MITRE ATT&CK mapping, root cause analysis, remediation planning.

**Offensive Security**
Kerberos attacks, SMB reconnaissance and exploitation, credential attacks (brute force, spraying, roasting), pass-the-hash, DCSync, network pivoting, LOLBins, offline password cracking.

**SOC Operations**
KQL query construction, hypothesis-driven threat hunting, detection gap analysis, purple team methodology.

## Reading Order

1. Start with this **README.md** for the overview
2. Read **[INTERNSHIP.md](INTERNSHIP.md)** for the full technical report (Months 1 + 2)
3. Deep-dive the three **[incident-reports/](incident-reports/)** for structured IR examples
4. Browse **[use-cases/](use-cases/)** and **[hunts/](hunts/)** for the working evidence
5. `full-intership.docx` is the same report as `INTERNSHIP.md` in Word format for offline reading

---

## Tools Used

**Defensive**
Wazuh SIEM v4.14.2 · pfSense CE · Active Directory · Windows Sysmon · OpenSearch Dashboards · VirusTotal

**Offensive**
Kali Linux · kerbrute · netexec (formerly crackmapexec) · impacket suite (GetUserSPNs, secretsdump, wmiexec, psexec) · evil-winrm · smbclient · nmap · hashcat · chisel · proxychains

**Infrastructure**
VirtualBox · Ubuntu 24.04 · Windows Server 2025 · Windows 10

---

## Contact

Feedback, questions, or reproducing this lab — reach out. This is a portfolio project intended to show what a serious internship in network and security engineering looks like end-to-end.