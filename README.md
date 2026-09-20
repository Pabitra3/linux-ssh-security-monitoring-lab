# Linux SSH Security Monitoring & Incident Investigation Lab

A hands-on Security Operations Center (SOC) lab for monitoring, detecting, investigating, and documenting Linux SSH security events using Ubuntu Server, Kali Linux, and VirtualBox.

This project demonstrates a practical SOC Analyst L1 workflow using native Linux telemetry rather than relying on a pre-built SIEM. The investigation focuses on SSH authentication activity, failed-login detection, source-IP correlation, event timelines, false-positive analysis, MITRE ATT&CK mapping, and incident reporting.

---

## Project Overview

The objective of this lab is to demonstrate how a SOC analyst can investigate suspicious SSH authentication activity from raw Linux logs and determine:

- What happened?
- When did it happen?
- Which account was targeted?
- What was the source IP?
- What was the destination system?
- Was authentication successful or unsuccessful?
- Was there evidence of compromise?
- Is the activity malicious, benign, or authorized testing?
- What evidence supports the conclusion?
- What defensive actions should be considered?

The lab uses controlled and authorized security testing inside an isolated VirtualBox environment.

---

## SOC Investigation Scenario

A Linux Ubuntu Server is monitored for SSH authentication activity.

During the investigation, multiple failed SSH authentication attempts were observed from the Kali Linux test workstation.

The analyst correlated:

1. SSH service telemetry
2. `/var/log/auth.log`
3. `journalctl`
4. Source and destination IP addresses
5. Successful and failed authentication events
6. Privileged command activity
7. Event timestamps
8. User/account information

The final investigation identified **three actual failed SSH authentication events** originating from the controlled Kali workstation.

The activity was classified as:

> **Controlled Security Testing / Benign Authorized Activity**

No confirmed compromise was identified from the available evidence.

---

# Lab Architecture

```text
                         Windows Host
                              |
                         VirtualBox
                              |
              +---------------+---------------+
              |                               |
        Kali-Attacker                    Ubuntu-Server
        192.168.56.20                   192.168.56.10
              |                               |
              +--------- SOC-LAB ------------+
                   192.168.56.0/24
                         |
                    SSH / TCP 22
```

### Network Design

The lab uses two VirtualBox network paths:

| Network | Purpose |
|---|---|
| NAT | Internet/package access |
| SOC-LAB Internal Network | Isolated security testing and monitoring |

### Virtual Machines

| System | Role | SOC-LAB IP | Primary Function |
|---|---|---:|---|
| Ubuntu Server | Target / monitored endpoint | `192.168.56.10` | SSH service and log generation |
| Kali Linux | Test workstation | `192.168.56.20` | Controlled SSH security testing |
| Windows Host | Virtualization host | N/A | Runs VirtualBox |

---

# Technology Stack

### Operating Systems
- Ubuntu Server 26.04 LTS
- Kali Linux
- Windows host

### Virtualization
- Oracle VirtualBox

### Security / Networking Tools
- OpenSSH
- `ss`
- `ip`
- `ping`
- `journalctl`
- `grep`
- `systemctl`
- Linux authentication logs

### Security Framework
- MITRE ATT&CK
- Technique: **T1110 — Brute Force**

---

# SOC Investigation Workflow

The project follows a simplified SOC Analyst L1 investigation lifecycle:

```text
Telemetry
   ↓
Detection
   ↓
Alert Validation
   ↓
Source IP Identification
   ↓
Event Correlation
   ↓
Authentication Analysis
   ↓
False-Positive Analysis
   ↓
MITRE ATT&CK Mapping
   ↓
Incident Classification
   ↓
Incident Documentation
   ↓
Response Recommendations
```

---

# 1. Network Verification

The first stage establishes the network context of the investigation.

### Ubuntu

```bash
ip -br addr
ip route
```

Expected SOC-LAB address:

```text
192.168.56.10/24
```

### Kali

```bash
ip addr
ip route get 192.168.56.10
```

Expected source address:

```text
192.168.56.20
```

### Connectivity Test

From Kali:

```bash
ping 192.168.56.10
```

This verifies that the controlled test workstation can reach the monitored Ubuntu endpoint.

---

# 2. SSH Service Verification

The monitored endpoint runs OpenSSH.

Check the service:

```bash
systemctl is-active ssh
```

Check TCP/22:

```bash
sudo ss -lntp | grep ':22'
```

The investigation confirmed that SSH was listening on TCP port 22.

---

# 3. Successful SSH Authentication

A controlled successful login was performed from Kali:

```bash
ssh lexa@192.168.56.10
```

After authentication:

```bash
hostname
whoami
```

Expected values:

```text
hostname → ubuntu-server
whoami   → lexa
```

This generated successful authentication telemetry that could later be correlated with failed authentication events.

---

# 4. Log Sources

The investigation primarily used two Linux telemetry sources.

### Authentication Log

```bash
/var/log/auth.log
```

Example search:

```bash
sudo grep -a "sshd.*Accepted password" /var/log/auth.log | tail -n 5
```

### SSH Journal

```bash
sudo journalctl -u ssh --no-pager
```

Failed authentication search:

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password"
```

These logs provided the evidence required to identify:

- timestamp
- username
- source IP
- destination service
- authentication result
- SSH process
- session information

---

# 5. Controlled Failed SSH Authentication

A wrong password was intentionally entered from the Kali test workstation.

This generated failed SSH authentication events on Ubuntu.

The relevant investigation identified three actual failed events:

| Timestamp | Result | Source IP | Target Account |
|---|---|---|---|
| 2026-09-09 07:58:55 | Failed | `192.168.56.20` | `lexa` |
| 2026-09-09 13:09:40 | Failed | `192.168.56.20` | `lexa` |
| 2026-09-09 13:11:02 | Failed | `192.168.56.20` | `lexa` |

The events were generated as part of authorized testing.

---

# 6. Event Correlation

A SOC analyst should not investigate authentication events in isolation.

The following command was used to correlate successful and failed SSH activity:

```bash
sudo journalctl -u ssh --no-pager | grep -E "Accepted password|Failed password" | tail -n 10
```

This allows the analyst to compare:

```text
Successful Authentication
        +
Failed Authentication
        +
Source IP
        +
Target Account
        +
Timestamp
```

This correlation helps determine whether failed authentication was followed by successful access.

---

# 7. Source IP Identification

The investigation identified:

```text
Source / Test Workstation:
192.168.56.20
```

```text
Destination / Monitored Server:
192.168.56.10
```

```text
Protocol:
SSH
```

```text
Destination Port:
22
```

The source IP was verified against the known lab network configuration rather than assuming that an IP address represented an external attacker.

---

# 8. Privileged Activity Analysis

The analyst also examined privileged command activity.

Example:

```bash
sudo whoami
```

Result:

```text
root
```

The corresponding authentication log contained `sudo` command telemetry showing:

```text
COMMAND=/usr/bin/whoami
USER=root
```

This was interpreted in context as analyst-generated investigation activity.

It was **not treated as evidence of malicious privilege escalation**.

This demonstrates an important SOC principle:

> A suspicious-looking event must be interpreted using surrounding context and authorization information.

---

# 9. Detection Logic

The core detection objective is to identify repeated failed SSH authentication attempts.

Basic detection:

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password"
```

A simple detection concept is:

```text
IF
    SSH authentication result = failure
AND
    source IP is identified
AND
    repeated failures occur within an investigation window
THEN
    generate an authentication anomaly for analyst review
```

The investigation then enriches the event with:

- source IP
- target IP
- username
- timestamp
- number of failures
- successful authentication after failures
- source-IP reputation/context in a production environment
- whether the activity was authorized

The project intentionally demonstrates the detection manually using Linux native telemetry.

---

# 10. False-Positive Analysis

The presence of failed SSH authentication does not automatically prove malicious activity.

The analyst checked:

### Source validation
Was the source IP part of the controlled lab?

**Yes.**

### Authorization
Was the activity intentionally generated?

**Yes.**

### Successful authentication
Was there evidence of successful access following the failed attempts?

The available investigation evidence did not show a successful authentication immediately following the failed attempts.

### Post-authentication activity
Was there evidence of suspicious activity after authentication?

No confirmed malicious post-authentication activity was identified.

### Final classification

```text
Controlled Security Testing / Benign Authorized Activity
```

This classification is based on the known lab context and observed evidence.

---

# 11. MITRE ATT&CK Mapping

The authentication activity was mapped to:

### T1110 — Brute Force

The controlled failed-login activity represents the type of authentication behavior that can be investigated under the MITRE ATT&CK Brute Force technique.

Important distinction:

> Mapping an observed behavior to a MITRE ATT&CK technique does not by itself prove that a real attacker performed the activity.

The actual lab activity was authorized testing.

---

# 12. Incident Timeline

| Date / Time | Event |
|---|---|
| 2026-09-04 14:34:53 | Successful SSH authentication |
| 2026-09-09 07:23:02 | Successful SSH authentication |
| 2026-09-09 07:58:55 | Failed SSH authentication |
| 2026-09-09 13:09:40 | Failed SSH authentication |
| 2026-09-09 13:11:02 | Failed SSH authentication |

The complete investigation timeline is documented in:

[`investigation/incident-timeline.md`](investigation/incident-timeline.md)

---

# 13. Incident Report

The complete incident report documents:

- incident description
- affected system
- source IP
- target account
- evidence
- timeline
- severity
- classification
- MITRE ATT&CK mapping
- investigation results
- response recommendations

See:

[`reports/incident-report.md`](reports/incident-report.md)

---

# 14. False-Positive Investigation

Detailed false-positive analysis is available in:

[`investigation/false-positive-analysis.md`](investigation/false-positive-analysis.md)

The document explains how an analyst can distinguish:

```text
Real Security Incident
        vs
Authorized Security Testing
        vs
Normal User Error
```

---

# 15. Detection Documentation

The complete detection methodology is documented in:

[`detection/detection-logic.md`](detection/detection-logic.md)

It covers:

- detection objective
- log sources
- detection patterns
- event counting
- source-IP correlation
- successful-login correlation
- false-positive handling
- MITRE mapping
- detection limitations
- SOC investigation workflow
- future SIEM implementation

---

# 16. Network Architecture Documentation

Detailed network architecture is available in:

[`architecture/network-architecture.md`](architecture/network-architecture.md)

It documents:

- VirtualBox topology
- VM roles
- IP addresses
- NAT vs Internal Network
- SSH traffic flow
- log sources
- security boundaries
- SOC investigation workflow

---

# 17. Evidence

Screenshots are organized under:

```text
screenshots/
├── 01-network/
├── 02-connectivity/
├── 03-ssh/
├── 04-logs/
├── 05-attacks/
└── 06-detection/
```

Evidence includes:

- network configuration
- connectivity
- SSH service status
- TCP/22 listener
- successful SSH login
- successful authentication logs
- failed authentication logs
- SSH event timeline
- event correlation
- final SSH service state

---

# 18. Key Skills Demonstrated

This project demonstrates practical exposure to:

### Linux Security

- Linux command line
- systemd
- journald
- authentication logs
- file permissions
- process/service inspection

### Networking

- IPv4 addressing
- subnetting fundamentals
- routing
- internal VirtualBox networking
- TCP/22
- SSH connectivity

### SOC Operations

- alert validation
- log analysis
- event correlation
- source-IP identification
- authentication analysis
- timeline reconstruction
- false-positive analysis
- incident classification
- evidence collection
- incident documentation

### Threat Detection

- SSH authentication monitoring
- failed-login detection
- repeated authentication analysis
- MITRE ATT&CK mapping
- detection logic development

### Security Documentation

- incident timeline
- incident report
- detection documentation
- architecture documentation
- evidence organization

---

# 19. Important Investigation Commands

### Network

```bash
ip -br addr
ip route
ip route get 192.168.56.10
ping 192.168.56.10
```

### SSH Service

```bash
systemctl is-active ssh
sudo ss -lntp | grep ':22'
```

### SSH Logs

```bash
sudo journalctl -u ssh --no-pager -n 20
```

### Failed Authentication

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password"
```

### Successful Authentication

```bash
sudo grep -a "sshd.*Accepted password" /var/log/auth.log | tail -n 5
```

### Event Correlation

```bash
sudo journalctl -u ssh --no-pager | grep -E "Accepted password|Failed password" | tail -n 10
```

### Privileged Command Investigation

```bash
sudo grep -a "COMMAND=/usr/bin/whoami" /var/log/auth.log | tail -n 3
```

---

# 20. Important SOC Investigation Lesson

One of the main lessons from this lab is that **raw log analysis requires context**.

For example:

```text
Failed SSH Login
        ↓
Identify Source IP
        ↓
Identify Target Account
        ↓
Check Frequency
        ↓
Check Successful Authentication
        ↓
Check Post-Authentication Activity
        ↓
Validate Authorization
        ↓
Classify Event
```

A single failed login should not automatically be treated as a confirmed attack.

Similarly, a successful login from an unusual source should not automatically be treated as compromise without additional evidence.

The analyst must correlate multiple pieces of telemetry before reaching an incident classification.

---

# 21. Project Limitations

This project is intentionally designed as a focused SOC Analyst L1 investigation lab.

Current limitations include:

- No centralized SIEM deployment
- No automated alerting
- No automated response
- No production-scale log ingestion
- No external threat-intelligence enrichment
- No persistent centralized dashboard
- Manual investigation workflow

These limitations are intentional and provide a clear path for future development.

---

# 22. Future Enhancements

Potential future enhancements include:

- Wazuh or another SIEM
- centralized log collection
- automated SSH brute-force detection
- alert dashboards
- threat-intelligence enrichment
- automated incident ticket creation
- automated response workflows
- additional Linux telemetry
- Windows endpoint monitoring
- Sysmon telemetry
- network IDS integration
- detection-as-code
- MITRE ATT&CK coverage expansion

These are **future enhancements**, not components currently deployed in this project.

---

# 23. Ethical Use

All security testing in this repository was performed in an isolated, controlled VirtualBox lab using systems owned or explicitly authorized for testing.

The failed SSH authentication events were intentionally generated for security monitoring and investigation practice.

Do not use these techniques against systems without authorization.

---

# 24. Repository Structure

```text
SOC-Home-Lab/
│
├── README.md
├── INTERVIEW_GUIDE.md
│
├── architecture/
│   └── network-architecture.md
│
├── commands/
│   ├── log-analysis.md
│   ├── networking.md
│   └── ssh.md
│
├── detection/
│   └── detection-logic.md
│
├── investigation/
│   ├── incident-timeline.md
│   ├── false-positive-analysis.md
│   └── mitre-attack-mapping.md
│
├── reports/
│   └── incident-report.md
│
└── screenshots/
    ├── 01-network/
    ├── 02-connectivity/
    ├── 03-ssh/
    ├── 04-logs/
    ├── 05-attacks/
    └── 06-detection/
```

---

# 25. Interview Explanation

### 30-second version

> “I built a Linux SSH Security Monitoring and Incident Investigation Lab using Ubuntu Server, Kali Linux, and VirtualBox. I configured an isolated SOC network, monitored SSH authentication activity using journald and auth.log, generated controlled failed authentication events, identified the source and destination IPs, correlated successful and failed logins, reconstructed the incident timeline, performed false-positive analysis, mapped the behavior to MITRE ATT&CK T1110, and documented the investigation in an incident report. The current implementation is manual and focuses on demonstrating the underlying SOC investigation process.”

### If asked: “Did you use Wazuh?”

> “Not in the current implementation. I intentionally built the investigation using native Linux telemetry first so I could understand the underlying detection and investigation process. The detection logic is documented in a way that could later be implemented in a SIEM such as Wazuh.”

### If asked: “What was the attacker IP?”

> “In this lab, `192.168.56.20` was the Kali test workstation that generated the controlled failed SSH authentication events. I would not describe it as an external attacker IP because the activity was authorized testing.”

### If asked: “Was the server compromised?”

> “No confirmed compromise was identified from the available evidence. The observed failed authentication events were generated intentionally during controlled testing.”

### If asked: “What did you actually learn?”

> “The main lesson was that SOC analysis is not just finding a suspicious log line. I learned to correlate timestamps, source IPs, accounts, authentication outcomes, service logs, and post-authentication activity before classifying an event.”

---

# 26. Project Outcome

The completed lab demonstrates an end-to-end manual SOC investigation workflow:

```text
Linux Endpoint
      ↓
SSH Telemetry
      ↓
Raw Authentication Logs
      ↓
Detection
      ↓
Source IP Identification
      ↓
Event Correlation
      ↓
Timeline Reconstruction
      ↓
False-Positive Analysis
      ↓
MITRE ATT&CK Mapping
      ↓
Incident Classification
      ↓
Incident Report
```

The project is designed to demonstrate **practical SOC Analyst L1 investigation skills**, with evidence and documentation supporting each major stage.

---

## License

This project is licensed under the MIT License.
