# SOC INCIDENT REPORT

---

| Field | Details |
|---|---|
| **Report Title** | Brute Force Attack via RDP — Detection & Analysis |
| **Incident ID** | INC-2026-001 |
| **Report Author** | Abhishek Prajapat |
| **Role** | SOC Analyst L1 (Lab Environment) |
| **Organization** | Mini SOC Lab — Home Lab Project |
| **Date of Incident** | March 26, 2026 |
| **Report Date** | March 26, 2026 |
| **Severity** | 🔴 High |
| **Status** | Resolved |
| **SIEM Platform** | Wazuh v4.7.5 (Open Source) |
| **Framework** | MITRE ATT&CK |

---

## 1. Executive Summary

A simulated brute force attack targeting the Remote Desktop Protocol (RDP) service was successfully detected, analyzed, and documented within a controlled virtualized SOC environment. The attacker machine (Kali Linux — `192.168.56.10`) performed systematic credential-stuffing attempts against the victim machine (Windows Tiny10 — `192.168.56.102`) on port 3389. The Wazuh SIEM platform (v4.7.5) detected the attack in real time, triggered high-severity alerts, and automatically mapped the behavior to the MITRE ATT&CK framework.

This report covers the complete SOC workflow — from initial network discovery and reconnaissance through brute force identification, alert triage, event correlation, and final analysis — demonstrating practical SOC Analyst competencies across the full incident lifecycle.

**Total Events Detected:** 22 authentication failures, 0 successes  
**Attack Window:** March 26, 2026 @ 11:48 – 11:51 AM IST  
**Endpoint Agent:** DESKTOP-V6FHRQI (Agent ID: 004)

---

## 2. Environment Overview

### 2.1 Lab Infrastructure

| Component | Details |
|---|---|
| **Virtualization Platform** | Oracle VirtualBox |
| **Network Type** | Host-Only Network (Isolated) |
| **Network Name** | soc-lab |
| **Subnet** | 192.168.56.0/24 |

### 2.2 System Inventory

| Machine | Operating System | Role | IP Address | Hostname |
|---|---|---|---|---|
| Attacker | Kali Linux | Threat Actor (Red Team) | 192.168.56.10 (eth0) | kalisoc |
| Victim | Windows 10 Enterprise LTSC 2021 (10.0.19044.3324) | Target Endpoint | 192.168.56.20 / 192.168.56.102 | DESKTOP-V6FHRQI |
| SIEM Server | Ubuntu + Wazuh v4.7.5 | Security Monitoring | 192.168.56.103 (enp0s8) | wazuh-server |

> **Note:** The victim machine has two network adapters. `192.168.56.20` is the primary Host-Only adapter (Ethernet 1). `192.168.56.102` is the secondary adapter (Ethernet 2) — this was the IP targeted during Nmap scans and Hydra brute force. User account on victim: `tiny10soc`.

### 2.3 Network Interface Details

| Machine | Interface | IPv4 Address | MAC Address |
|---|---|---|---|
| Kali Linux | eth0 | 192.168.56.10 | 08:00:27:9b:45:de |
| Kali Linux | eth1 | 10.0.3.15 (NAT) | 08:00:27:9d:83:3c |
| Windows Victim | Ethernet 1 | 192.168.56.20 | — |
| Windows Victim | Ethernet 2 | 192.168.56.102 | — |
| Wazuh Server | enp0s3 | 10.0.2.15 (NAT) | 08:00:27:68:6a:2e |
| Wazuh Server | enp0s8 | 192.168.56.103 | 08:00:27:62:f5:f3 |

### 2.4 Monitoring Stack

| Component | Tool | Version |
|---|---|---|
| SIEM | Wazuh Manager | v4.7.5 |
| Log Indexing | OpenSearch | — |
| Dashboard | Wazuh Dashboard | v4.7.5 |
| Endpoint Agent | Wazuh Agent (Windows) | v4.7.5 |
| FIM Module | Wazuh Syscheck | Built-in |
| SCA Module | Wazuh SCA | CIS Win10 Benchmark v1.12.0 |

---

## 3. Incident Timeline

```
[Phase 1] Reconnaissance — March 26, 2026 @ 10:12 AM IST
    └── nmap -sn 192.168.56.20       → Host discovery, MAC: 08:00:27:C3:FD:A5 (Oracle VirtualBox NIC)
    └── nmap -A 192.168.56.20        → OS: Windows 11, ports 139/445 open, SMB signing NOT required
    └── nmap -sS -Pn 192.168.56.102  → SYN scan, 998 filtered, ports 139/445 confirmed open
    └── nmap -p 3389 192.168.56.102  → @ 10:58 AM → 3389/tcp OPEN ms-wbt-server (0.62s)

[Phase 2] Pre-Attack Preparation
    └── RDP enabled on victim (simulating misconfigured endpoint)
    └── Firewall rule: netsh advfirewall firewall set rule group="remote desktop" new enable=Yes
    └── Port 3389 confirmed OPEN from Kali

[Phase 3] Brute Force Execution — March 26, 2026 @ ~11:46 AM IST
    └── Hydra launched: hydra -l administrator -P attack.txt -t 1 -W 2 rdp://192.168.56.102
    └── First event logged: Mar 26, 2026 @ 11:48:24.795
    └── Windows Event ID 4625 (Failed Logon) generated per attempt

[Phase 4] Detection & Alerting — March 26, 2026 @ 11:48–11:51 AM IST
    └── Wazuh Rule 60122 fired → Single logon failure (Level 5)
    └── Wazuh Rule 60204 fired → Multiple logon failures (Level 10)
    └── Total: 22 authentication failures detected, 0 successes
    └── Alert spike visible on SIEM dashboard timeline
    └── Last event logged: Mar 26, 2026 @ 11:51:47.079

[Phase 5] Analysis & Documentation
    └── Events correlated → Brute force pattern confirmed
    └── MITRE ATT&CK: T1110, T1078, T1531 auto-mapped by Wazuh
    └── Source IP 192.168.56.10, agent DESKTOP-V6FHRQI confirmed
    └── Incident report generated
```

---

## 4. Attack Narrative

### 4.1 Phase 1 — Reconnaissance (Nmap)

The threat actor initiated a structured multi-stage reconnaissance process to enumerate live hosts, identify services, and fingerprint the target OS.

**Tool Used:** Nmap 7.98
**Commands Executed (in order):**

```bash
# Stage 1 — Host discovery
nmap -sn 192.168.56.20

# Stage 2 — Aggressive scan: OS + service + version + script detection
nmap -A 192.168.56.20

# Stage 3 — Stealth SYN scan (no ping)
nmap -sS -Pn 192.168.56.102

# Stage 4 — Targeted RDP port check (March 26, 2026 @ 10:58 AM)
nmap -p 3389 192.168.56.102
```

**Ports Discovered:**

| Port | Protocol | Service | State | Notes |
|---|---|---|---|---|
| 139 | TCP | netbios-ssn | Open | Microsoft Windows netbios-ssn |
| 445 | TCP | microsoft-ds | Open | SMB — signing enabled but NOT required |
| 3389 | TCP | ms-wbt-server | Open | RDP — primary brute force target |

**OS & System Fingerprinting (nmap -A results):**

| Field | Value |
|---|---|
| Detected OS | Microsoft Windows 11 |
| OS CPE | cpe:/o:microsoft:windows_11 |
| MAC Address | 08:00:27:C3:FD:A5 (Oracle VirtualBox NIC) |
| SMB Signing | Enabled but NOT required (security misconfiguration) |
| SMB2 Time | 2026-03-09T22:06:21 |
| Clock Skew | 5 seconds |
| Network Distance | 1 hop |
| Scan Duration | 53.39 seconds |

**Analyst Note:** SMB signing not being required is a significant misconfiguration exposing the system to relay attacks. Port 3389 open confirmed this machine as a high-value credential attack target.

---

### 4.2 Phase 2 — Attack Surface Preparation

RDP was enabled on the victim to replicate a realistic misconfigured endpoint scenario.

**Command on Windows Victim:**
```powershell
netsh advfirewall firewall set rule group="remote desktop" new enable=Yes
```

**Verification from Kali (March 26, 2026 @ 10:58 AM):**
```bash
nmap -p 3389 192.168.56.102
# Result → 3389/tcp open  ms-wbt-server
# Host is up (0.0011s latency)
# Scan completed in 0.62 seconds
```

---

### 4.3 Phase 3 — Brute Force Execution (Hydra)

**Tool Used:** Hydra
**Attack Command:**

```bash
hydra -l administrator -P attack.txt -t 1 -W 2 rdp://192.168.56.102
```

**Flag Explanation:**

| Flag | Value | Purpose |
|---|---|---|
| `-l` | administrator | Target username (local admin account) |
| `-P` | attack.txt | Custom password wordlist |
| `-t 1` | 1 thread | Single thread — avoid RDP rate limiting / lockout |
| `-W 2` | 2 seconds | Delay between attempts |
| `rdp://` | 192.168.56.102 | Target protocol and IP (Ethernet 2 of victim) |

**Custom Wordlist (`attack.txt`):**
```
admin
admin123
password
123456
root
```

> `rockyou.txt` was too slow due to RDP rate limiting. A targeted wordlist generated a detectable event volume within a short window.

Each failed attempt generated **Windows Security Event ID 4625**, immediately forwarded to Wazuh Manager via Agent 004 (DESKTOP-V6FHRQI).

---

## 5. Detection & Alerting

### 5.1 Windows Event Log — Event ID 4625

| Field | Value |
|---|---|
| Event ID | 4625 — An account failed to log on |
| Account Name | administrator |
| Source IP | 192.168.56.10 (Kali Linux — eth0) |
| Target IP | 192.168.56.102 (Windows Ethernet 2) |
| Logon Type | 10 (RemoteInteractive — RDP session) |
| Failure Reason | Unknown user name or bad password |
| Process | svchost.exe |
| Agent Name | DESKTOP-V6FHRQI |
| Agent ID | 004 |
| First Event | Mar 26, 2026 @ 11:48:24.795 |
| Last Event | Mar 26, 2026 @ 11:51:47.079 |
| Total Events | **22 failures / 0 successes** |

---

### 5.2 Wazuh Rules Triggered

| Rule ID | Description | Severity Level | Trigger Condition |
|---|---|---|---|
| **60122** | Logon failure — Unknown user or bad password | Level 5 — Medium | Single Event ID 4625 per attempt |
| **60204** | Multiple Windows logon failures | Level 10 — High | Repeated failures = brute-force pattern |

---

### 5.3 Alert Behavior Observed

- **22 total hits** on Event ID 4625 filter in Wazuh Security Events
- **Alert spike** visible on dashboard timeline — two bursts at ~06:00 and ~09:00 window
- **Rule escalation:** Level 5 (60122) → Level 10 (60204) as frequency increased
- **MITRE ATT&CK** auto-tagged on every event by Wazuh
- **Authentication failure counter:** 22 | **Authentication success counter:** 0

---

### 5.4 SCA Compliance Scan Result

| Field | Value |
|---|---|
| Benchmark | CIS Microsoft Windows 10 Enterprise Benchmark v1.12.0 |
| Scan Time | Mar 26, 2026 @ 06:08:15 |
| Passed | 128 checks |
| Failed | **255 checks** |
| Not Applicable | 11 checks |
| **Overall Score** | **33%** |

> A 33% CIS compliance score confirms severe endpoint misconfiguration — consistent with a machine vulnerable to brute force, RDP exposure, weak password policies, and missing lockout controls.

---

## 6. File Integrity Monitoring (FIM)

### 6.1 Configuration

| Field | Value |
|---|---|
| Module | Wazuh Syscheck |
| Monitored Path | `C:\Users\tiny10soc\Desktop` |
| Agent | DESKTOP-V6FHRQI (ID: 004) |
| User Account | tiny10soc |

### 6.2 Real Detection Results

| Timestamp (IST) | Action | File Path | Rule ID | Description | Level |
|---|---|---|---|---|---|
| Mar 26, 2026 @ 09:41:38.763 | **added** | `c:\users\tiny10soc\desktop\test.txt.txt` | 554 | File added to the system | 5 |
| Mar 26, 2026 @ 09:41:44.863 | **deleted** | `c:\users\tiny10soc\desktop\test.txt.txt` | 553 | File deleted | 7 |

**Analyst Note:** Both events were detected within 6 seconds of the action. Wazuh captured the full file path, event type, syscheck path, and rule classification in real time with zero false positives.

---

## 7. MITRE ATT&CK Mapping

All techniques below were **automatically mapped by Wazuh** and confirmed in the Security Events dashboard.

| Tactic | Technique ID | Technique Name | Observed Behavior |
|---|---|---|---|
| Reconnaissance | T1046 | Network Service Discovery | Nmap multi-stage scan — ports 139, 445, 3389 enumerated |
| Credential Access | T1110 | Brute Force | Hydra RDP — 22 failed login attempts on administrator |
| Defense Evasion | T1078 | Valid Accounts | Login attempts using real local admin account name |
| Persistence | T1078 | Valid Accounts | Wazuh auto-mapped alongside Defense Evasion |
| Privilege Escalation | T1078 | Valid Accounts | Escalation attempt via administrator credentials |
| Initial Access | T1078 | Valid Accounts | Attempted initial foothold via exposed RDP |
| Impact | T1531 | Account Access Removal | Auto-mapped by Wazuh on repeated logon failures |

> **Top MITRE Tactics detected (from Wazuh agent dashboard):**
> Persistence (35) | Privilege Escalation (35) | Defense Evasion (34) | Initial Access (34) | Impact (7)

---

## 8. Root Cause Analysis

| Factor | Detail |
|---|---|
| **Initial Vector** | RDP port 3389 exposed on 192.168.56.102 with no rate-limiting or lockout |
| **Vulnerability** | Weak credential policy — common passwords accepted without lockout enforcement |
| **Misconfiguration** | SMB signing not required (confirmed by nmap -A) — additional relay attack surface |
| **Compliance Gap** | 33% CIS score — 255 benchmark checks failing on victim endpoint |
| **Detection Gap** | No alert on single failure until pattern threshold triggered Rule 60204 |
| **Exposure Window** | 3 minutes 22 seconds (11:48:24.795 → 11:51:47.079) |

---

## 9. Recommendations

### Immediate Actions

| # | Recommendation | Priority |
|---|---|---|
| 1 | Disable RDP if not required — use VPN-gated access instead | 🔴 Critical |
| 2 | Enforce Account Lockout Policy (5 failed attempts = 30-min lockout) | 🔴 Critical |
| 3 | Enable Network Level Authentication (NLA) for RDP | 🔴 Critical |
| 4 | Enforce SMB signing as required (currently disabled — confirmed by Nmap) | 🔴 Critical |
| 5 | Restrict RDP access by IP using Windows Firewall inbound rules | 🟠 High |
| 6 | Rename default administrator account | 🟠 High |

### Long-Term Improvements

| # | Recommendation | Priority |
|---|---|---|
| 7 | Deploy MFA on all remote access services | 🔴 Critical |
| 8 | Improve CIS Benchmark compliance from 33% (255 checks currently failing) | 🟠 High |
| 9 | Implement SOAR to auto-block IPs on Rule 60204 trigger | 🟠 High |
| 10 | Lower Rule 60122 threshold — trigger at 3 consecutive failures | 🟡 Medium |
| 11 | Enable Threat Intelligence feeds in Wazuh | 🟡 Medium |
| 12 | Deploy honeypot RDP service as early-warning trap | 🟡 Medium |

---

## 10. Lessons Learned

1. **Two adapters, two attack surfaces** — The victim having both 192.168.56.20 and 192.168.56.102 shows how multiple interfaces on a single endpoint create unexpected exposure. Always audit all active network adapters during hardening.

2. **SIEM correlation is the real detection engine** — 22 individual Rule 60122 alerts are noise. Wazuh escalating them to Rule 60204 demonstrates behavioral detection. Without correlation, this attack requires manual pattern recognition across raw logs.

3. **nmap -A reveals more than just open ports** — OS fingerprinting, SMB signing status, clock skew, and MAC vendor info all came from one scan. This is exactly what attackers collect before striking — defenders must understand this attack surface.

4. **A 33% SCA score tells the whole story** — 255 CIS benchmark checks failed before any attack. Compliance scanning should be a baseline step in every endpoint assessment, not an afterthought.

5. **FIM catches what network tools miss** — File creation and deletion on `test.txt.txt` was detected within 6 seconds. Endpoint-level monitoring is non-negotiable in any SOC deployment.

6. **MITRE ATT&CK auto-mapping adds professional depth** — Wazuh automatically tagged T1078, T1110, T1531 across Credential Access, Defense Evasion, Persistence, Privilege Escalation, and Impact. Raw logs become actionable intelligence.

---

## 11. Analyst Sign-Off

| Field | Detail |
|---|---|
| **Analyst Name** | Abhishek Prajapat |
| **Role** | SOC Analyst L1 (Lab Environment) |
| **Certifications** | Certified SOC Analyst (CSA) \| CCNA 200-301 |
| **Institution** | Swami Keshvanand Institute of Technology, Jaipur |
| **Contact** | abhishekprajapati6350@gmail.com |
| **LinkedIn** | [linkedin.com/in/abhii-prajapati6350](https://www.linkedin.com/in/abhii-prajapati6350) |
| **GitHub** | [github.com/Abhii-prajapat](https://github.com/Abhii-prajapat) |
| **Report Status** | ✅ Final |

---

*All data in this report — IPs, timestamps, rule IDs, event counts, file paths, MAC addresses, compliance scores — is real and captured directly from the lab environment. No data has been fabricated or estimated.*
