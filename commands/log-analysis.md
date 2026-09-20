# Linux SSH Log Analysis & Investigation

## SOC Analyst L1 — Command Reference and Investigation Guide

---

## 1. Overview

This document contains the commands, observations, analysis methodology, and investigation logic used during the Linux SSH Security Monitoring & Incident Investigation Lab.

The purpose of this document is to demonstrate how a SOC Analyst L1 can investigate authentication-related security events on a Linux server using native Linux security and monitoring tools.

The investigation focuses on:

- SSH service monitoring
- SSH port verification
- Authentication log analysis
- Successful authentication detection
- Failed authentication detection
- Source IP identification
- Event correlation
- Timeline construction
- Privileged activity investigation
- Network verification
- Incident classification
- MITRE ATT&CK mapping

The investigation follows a simplified SOC workflow:

```text
Monitor
   ↓
Detect
   ↓
Collect Evidence
   ↓
Analyze Logs
   ↓
Identify Source
   ↓
Correlate Events
   ↓
Investigate Activity
   ↓
Classify Incident
   ↓
Document Findings
```

The activity performed in this project was conducted in an isolated and controlled VirtualBox laboratory environment.

---

# 2. Lab Environment

## 2.1 Infrastructure

| Component | Configuration |
|---|---|
| Host Operating System | Windows |
| Virtualization | Oracle VirtualBox |
| Target Server | Ubuntu Server 26.04 LTS |
| Test/Attacker Machine | Kali Linux |
| Target Username | `lexa` |
| Ubuntu SOC-LAB IP | `192.168.56.10` |
| Kali SOC-LAB IP | `192.168.56.20` |
| Protocol | SSH |
| Port | TCP/22 |
| Virtual Network | `SOC-LAB` |
| Ubuntu Internal Interface | `enp0s8` |
| Kali Internal Interface | `eth1` |

---

## 2.2 Network Architecture

The laboratory uses two network interfaces on the virtual machines.

### NAT Network

The NAT interface provides Internet access.

Ubuntu:

```text
10.0.2.15
```

Kali:

```text
10.0.2.6
```

### Internal SOC-LAB Network

The internal network is used for communication between Kali and Ubuntu.

Ubuntu:

```text
192.168.56.10/24
```

Kali:

```text
192.168.56.20/24
```

The SSH investigation therefore uses:

```text
Source:
192.168.56.20

Destination:
192.168.56.10

Protocol:
SSH

Port:
22/TCP
```

---

# 3. Initial Network Verification

Before investigating SSH activity, network connectivity between the test machine and the monitored server was verified.

## 3.1 Display Network Interfaces

### Command

```bash
ip -br addr
```

### Purpose

This command provides a concise overview of the system's network interfaces and assigned IP addresses.

It was used on both Ubuntu and Kali to verify the SOC-LAB addresses.

### Ubuntu

The relevant interface was:

```text
enp0s8
192.168.56.10/24
```

### Kali

The relevant interface was:

```text
eth1
192.168.56.20/24
```

### SOC Relevance

A SOC analyst needs to understand the network identity of the monitored system before interpreting source and destination addresses in security logs.

---

# 4. Verify Routing

The Ubuntu routing table was checked using:

```bash
ip route
```

The SOC-LAB network was associated with:

```text
192.168.56.0/24
```

through:

```text
enp0s8
```

The NAT network was associated with the Internet-facing virtual interface.

## 4.1 Verify Kali Route to Ubuntu

The exact route used by Kali to reach the Ubuntu server was checked with:

```bash
ip route get 192.168.56.10
```

The relevant result was:

```text
192.168.56.10 dev eth1 src 192.168.56.20
```

### Interpretation

This confirms:

```text
Destination: 192.168.56.10
Interface:   eth1
Source IP:   192.168.56.20
```

This was important because the same source IP was later observed in SSH authentication logs.

---

# 5. Test Network Connectivity

Connectivity between Kali and Ubuntu was tested using ICMP.

### Command

```bash
ping -c 4 192.168.56.10
```

### Result

The Ubuntu server responded successfully to all four packets.

```text
4 packets transmitted
4 packets received
0% packet loss
```

### SOC Relevance

Basic connectivity should be confirmed before troubleshooting an application such as SSH.

If connectivity fails, an SSH connection failure cannot automatically be interpreted as an authentication problem.

---

# 6. SSH Service Verification

After confirming network connectivity, the SSH service on Ubuntu was checked.

### Command

```bash
sudo systemctl status ssh --no-pager
```

### Purpose

This command verifies the state of the SSH daemon.

It provides information such as:

- Service status
- Process state
- Whether the service is running
- Recent service events

### Final Observed State

```text
Active: active (running)
```

The SSH daemon was therefore operational.

## 6.1 Quick SSH Service State Check

```bash
systemctl is-active ssh
```

Expected result:

```text
active
```

### SOC Relevance

This is useful for quickly determining whether SSH is currently active without displaying the complete systemd status output.

---

# 7. Verify SSH Listening Port

The system's listening sockets were checked using:

```bash
sudo ss -lntp | grep ':22'
```

### Purpose

This verifies whether TCP port 22 is listening.

The command uses:

- `ss` — socket statistics
- `-l` — listening sockets
- `-n` — numerical addresses and ports
- `-t` — TCP sockets
- `-p` — process information
- `grep ':22'` — filter for SSH port 22

### Observed Result

SSH was listening on:

```text
0.0.0.0:22
```

and:

```text
[::]:22
```

### Interpretation

The SSH daemon was listening on IPv4 and IPv6 addresses.

### SOC Relevance

Listening services represent potential attack surfaces. SSH is particularly important to monitor because authentication attempts can include password guessing, brute-force attempts, credential attacks, and automated scanning.

---

# 8. Establish an SSH Session

The Kali test machine was used to connect to the Ubuntu server.

### Command

```bash
ssh lexa@192.168.56.10
```

After successful authentication, the session was verified using:

```bash
hostname
```

The server returned:

```text
ubuntu-server
```

The current user was then verified:

```bash
whoami
```

The result was:

```text
lexa
```

### SOC Relevance

This establishes a known successful authentication event that can later be correlated with the server's security logs.

---

# 9. SSH Authentication Logs

Ubuntu records authentication-related activity in:

```text
/var/log/auth.log
```

SSH events can also be queried through the systemd journal.

The two primary investigation sources used in this lab were:

```text
/var/log/auth.log
```

and:

```text
journalctl -u ssh
```

---

# 10. Review Recent Authentication Logs

### Command

```bash
sudo tail -n 50 /var/log/auth.log
```

### Purpose

This displays the most recent authentication-related log entries.

The log can contain events related to:

- SSH
- sudo
- PAM authentication
- User sessions
- Authentication failures
- Authentication successes

### SOC Relevance

Authentication logs are one of the most important Linux security telemetry sources for investigating account activity.

---

# 11. Review SSH Service Logs

### Command

```bash
sudo journalctl -u ssh --no-pager -n 20
```

### Purpose

This displays the latest SSH service events recorded by systemd's journal.

Relevant information can include:

- Authentication attempts
- Successful logins
- Failed logins
- Session creation
- Session termination
- SSH daemon events

---

# 12. Review the Complete SSH Journal

For broader investigation:

```bash
sudo journalctl -u ssh --no-pager
```

This provides the SSH service event history available in the journal.

### SOC Relevance

Using the service-specific journal helps reduce unrelated system events and makes SSH investigation more focused.

---

# 13. Detect Successful SSH Authentication

Successful SSH authentication events were searched in `/var/log/auth.log`.

### Command

```bash
sudo grep -a "sshd.*Accepted password" /var/log/auth.log | tail -n 5
```

### Purpose

The command searches for successful password-based SSH authentication events.

The important keyword is:

```text
Accepted password
```

A representative event contains:

```text
Accepted password for lexa from 192.168.56.20
```

### Important Fields

| Field | Value |
|---|---|
| Authentication result | Accepted |
| Username | lexa |
| Source IP | 192.168.56.20 |
| Protocol | SSH |
| Authentication method | Password |

### SOC Interpretation

A successful authentication does not automatically mean malicious activity.

The analyst must determine:

1. Was the login expected?
2. Was the account authorized?
3. Was the source IP expected?
4. Was the authentication method legitimate?
5. What activity occurred after authentication?

---

# 14. Detect Failed SSH Authentication

Failed SSH authentication attempts were investigated through the SSH service journal.

### Command

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password"
```

### Purpose

This filters SSH service logs for failed password authentication events.

The important indicator is:

```text
Failed password
```

A representative event is:

```text
Failed password for lexa from 192.168.56.20
```

### Important Fields

| Field | Meaning |
|---|---|
| Failed password | Authentication failed |
| lexa | Target account |
| 192.168.56.20 | Source IP |
| SSH | Protocol |
| Timestamp | Time of event |

---

# 15. Failed Authentication Investigation

The laboratory contained controlled failed SSH authentication attempts.

The actual failed SSH events identified through the SSH service journal were:

```text
2026-09-09 07:58:55
2026-09-09 13:09:40
2026-09-09 13:11:02
```

All three events originated from:

```text
192.168.56.20
```

and targeted:

```text
lexa
```

on:

```text
192.168.56.10
```

---

# 16. Search Failed Authentication in auth.log

The same type of event can be searched directly in the authentication log.

### Command

```bash
sudo grep -a "sshd.*Failed password" /var/log/auth.log
```

### Purpose

This searches `/var/log/auth.log` for SSH daemon failed-password events.

### SOC Relevance

Using `auth.log` provides an additional source for validating SSH authentication activity.

---

# 17. Important Log Analysis Lesson: Self-Referential Events

During the investigation, an important log-analysis issue was identified.

Commands executed using `sudo` can themselves generate authentication/audit-related records.

For example:

```bash
sudo grep -a "Failed password" /var/log/auth.log
```

may cause related sudo activity to appear in the authentication log.

Therefore, simply executing:

```bash
grep ... | wc -l
```

against `/var/log/auth.log` can produce a misleading event count if the analyst does not distinguish actual SSH daemon events from audit records generated by the investigation itself.

### SOC Lesson

A raw text match is not automatically a distinct security event.

The analyst must understand:

- Which process generated the log entry
- Which subsystem owns the event
- Whether the entry is an authentication event or an audit record
- Whether the analyst's own command created the matching record

---

# 18. Correct Method for Counting Failed SSH Events

Instead of relying on a raw `/var/log/auth.log` count, the SSH service journal was filtered directly.

### Command

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password"
```

The result showed the actual SSH authentication failures.

The confirmed count was:

```text
3 actual failed SSH authentication events
```

### SOC Lesson

A log count is only meaningful after the analyst understands what each matching record represents.

This is an important distinction between simple log searching and actual security investigation.

---

# 19. Count Failed SSH Events

After filtering the SSH service journal, the events can be counted with:

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password" | wc -l
```

The investigation produced:

```text
3
```

actual failed SSH authentication events.

---

# 20. Identify the Source IP

The source IP is one of the most important indicators in authentication investigations.

### Command

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password"
```

The failed events showed:

```text
from 192.168.56.20
```

Therefore:

```text
Source IP:
192.168.56.20
```

---

# 21. Validate the Source IP Against Kali

The Kali network configuration was checked with:

```bash
ip -br addr
```

The internal SOC-LAB interface was:

```text
eth1
```

with:

```text
192.168.56.20/24
```

This matched the source IP recorded in the Ubuntu SSH logs.

### Correlation

```text
Kali interface:
eth1

Kali IP:
192.168.56.20

Ubuntu SSH log source:
192.168.56.20
```

### Conclusion

The source IP in the SSH logs corresponded to the Kali test workstation used in the controlled lab.

---

# 22. Correlate Successful and Failed Authentication

Successful and failed SSH events were combined into a simplified event view.

### Command

```bash
sudo journalctl -u ssh --no-pager | grep -E "Accepted password|Failed password" | tail -n 10
```

### Purpose

This filters the SSH journal for two important authentication outcomes:

```text
Accepted password
```

and:

```text
Failed password
```

### SOC Value

Event correlation allows an analyst to answer questions such as:

- Did failed attempts occur?
- Were the failures repeated?
- Did authentication eventually succeed?
- Did the same source IP generate both failed and successful events?
- What was the timing between events?

---

# 23. Authentication Event Timeline

The investigation identified the following relevant authentication events.

| Timestamp | Event | Source IP | Account |
|---|---|---|---|
| 2026-09-04 14:34:53 | Successful SSH authentication | 192.168.56.20 | lexa |
| 2026-09-09 07:23:02 | Successful SSH authentication | 192.168.56.20 | lexa |
| 2026-09-09 07:58:55 | Failed SSH authentication | 192.168.56.20 | lexa |
| 2026-09-09 13:09:40 | Failed SSH authentication | 192.168.56.20 | lexa |
| 2026-09-09 13:11:02 | Failed SSH authentication | 192.168.56.20 | lexa |

The three failed events were confirmed using the SSH service journal.

---

# 24. Investigating the Relationship Between Failures and Success

One of the most important SOC questions during an authentication investigation is:

> Did a successful login occur immediately after repeated failed authentication attempts?

A suspicious sequence might look like:

```text
Failed
Failed
Failed
Accepted
```

This pattern could indicate a successful brute-force or password-guessing attempt.

In this controlled lab, the available evidence did not establish a successful authentication immediately following the three identified failed attempts.

The source IP was the known Kali test workstation.

Therefore, the events were classified as controlled security testing rather than confirmed compromise.

---

# 25. Investigate Active SSH Connections

Current TCP connections involving SSH can be checked with:

```bash
sudo ss -tnp | grep ':22'
```

### Purpose

This identifies currently active TCP connections associated with port 22.

### SOC Relevance

During a live incident, an active SSH connection may provide useful information about:

- Current source IP
- Destination
- Connection state
- Process information

An empty result does not indicate a problem.

It simply means that no matching active TCP SSH connection existed at the moment the command was executed.

---

# 26. Investigate Privileged Activity

Authentication investigations should not stop after identifying a successful login.

The analyst should also determine whether privileged commands were executed.

Linux `sudo` activity can be found in `/var/log/auth.log`.

A controlled investigation command was:

```bash
sudo grep -a "COMMAND=/usr/bin/whoami" /var/log/auth.log | tail -n 3
```

### Purpose

This searches for sudo execution of:

```text
/usr/bin/whoami
```

The relevant record contained information such as:

```text
USER=root
COMMAND=/usr/bin/whoami
```

---

# 27. Interpret Privileged Activity Correctly

The presence of:

```text
USER=root
```

does not automatically mean that an attacker escalated privileges.

Context must be considered.

In this lab, the `sudo whoami` command was deliberately executed by the analyst as part of the investigation.

The observed privilege-related activity was therefore:

```text
Analyst-generated investigation activity
```

rather than:

```text
Confirmed malicious privilege escalation
```

### SOC Lesson

Security logs must be interpreted together with:

- User identity
- Source IP
- Timestamp
- Command
- Session context
- Investigation activity
- Other correlated events

---

# 28. Verify Current User

The currently authenticated Linux user was checked using:

```bash
whoami
```

The result was:

```text
lexa
```

This confirmed the normal user context.

---

# 29. Verify Sudo Privilege Context

A controlled sudo command was executed:

```bash
sudo whoami
```

The result was:

```text
root
```

This demonstrated the distinction between:

```text
Normal user:
lexa
```

and:

```text
Privileged sudo context:
root
```

The corresponding sudo activity was visible in the authentication logs.

---

# 30. SSH Session Validation

After connecting to the server through SSH, the session was validated using:

```bash
hostname
```

Result:

```text
ubuntu-server
```

Then:

```bash
whoami
```

Result:

```text
lexa
```

This verified that the SSH session reached the intended Ubuntu server and authenticated as the expected user.

---

# 31. Controlled Failed Authentication Test

A deliberate incorrect password was entered during an SSH connection attempt.

### Command

```bash
ssh lexa@192.168.56.10
```

The incorrect credential attempt resulted in:

```text
Permission denied, please try again.
```

This generated a corresponding server-side SSH log event containing:

```text
Failed password for lexa from 192.168.56.20
```

### Purpose

This provided a known controlled test case for validating that failed SSH authentication events were being recorded correctly.

---

# 32. Why Controlled Testing Is Important

A SOC lab should generate known events so the analyst can verify that:

1. The event actually occurs.
2. The event is logged.
3. The relevant log source records it.
4. The analyst can search for it.
5. The source can be identified.
6. The event can be correlated.
7. The event can be documented.

This is more useful than simply reading random logs without understanding how the events were generated.

---

# 33. Authentication Investigation Questions

During an SSH investigation, a SOC Analyst L1 should ask:

### Question 1 — Which host was targeted?

Answer:

```text
Ubuntu Server
192.168.56.10
```

### Question 2 — Which service was targeted?

Answer:

```text
SSH
```

### Question 3 — Which port was targeted?

Answer:

```text
TCP/22
```

### Question 4 — Which account was targeted?

Answer:

```text
lexa
```

### Question 5 — What was the source IP?

Answer:

```text
192.168.56.20
```

### Question 6 — Was the source IP known?

Answer:

```text
Yes.
It belonged to the Kali test workstation in the lab.
```

### Question 7 — Were authentication failures observed?

Answer:

```text
Yes.
Three actual failed SSH authentication events were identified.
```

### Question 8 — Was successful authentication observed?

Answer:

```text
Yes.
Successful SSH authentication events were observed from the same lab source.
```

### Question 9 — Was privilege-related activity observed?

Answer:

```text
Yes.
But the observed sudo activity was generated by the analyst during investigation.
```

### Question 10 — Was compromise confirmed?

Answer:

```text
No.
No confirmed compromise was identified.
```

---

# 34. MITRE ATT&CK Mapping

The controlled repeated authentication failures can be mapped to:

```text
T1110 — Brute Force
```

### Technique

**MITRE ATT&CK T1110 — Brute Force**

### Context

The lab simulated repeated failed authentication attempts against an SSH service.

The technique mapping is used to demonstrate how SOC analysts can associate observed behavior with an established adversary technique framework.

Because this was an authorized laboratory simulation, the mapping does not represent a real-world attacker attribution or confirmed compromise.

---

# 35. Incident Classification

Based on the investigation evidence, the activity was classified as:

```text
Classification:
Controlled Security Testing / Benign Authorized Security Testing
```

### Severity

```text
Low
```

### Status

```text
Closed
```

### Reasoning

The investigation determined that:

- The source IP was the known Kali test workstation.
- The SSH target was the controlled Ubuntu server.
- The failed authentication events were deliberately generated.
- No confirmed unauthorized access was identified.
- Privileged activity was analyst-generated.
- No evidence established compromise.

---

# 36. SOC Investigation Workflow

The investigation can be summarized as:

```text
1. Verify network connectivity
        ↓
2. Identify target host
        ↓
3. Verify SSH service
        ↓
4. Verify TCP/22 listening state
        ↓
5. Generate controlled authentication events
        ↓
6. Collect SSH logs
        ↓
7. Search for successful authentication
        ↓
8. Search for failed authentication
        ↓
9. Identify source IP
        ↓
10. Validate source IP against Kali
        ↓
11. Correlate successful and failed events
        ↓
12. Build incident timeline
        ↓
13. Investigate privilege-related activity
        ↓
14. Determine whether compromise occurred
        ↓
15. Map behavior to MITRE ATT&CK
        ↓
16. Classify and document the incident
```

---

# 37. Key Commands Reference

## Network Information

```bash
ip -br addr
```

```bash
ip route
```

```bash
ip route get 192.168.56.10
```

```bash
ping -c 4 192.168.56.10
```

## SSH Service

```bash
sudo systemctl status ssh --no-pager
```

```bash
systemctl is-active ssh
```

## SSH Listening Port

```bash
sudo ss -lntp | grep ':22'
```

## SSH Logs

```bash
sudo journalctl -u ssh --no-pager -n 20
```

```bash
sudo journalctl -u ssh --no-pager
```

## Authentication Logs

```bash
sudo tail -n 50 /var/log/auth.log
```

## Successful SSH Authentication

```bash
sudo grep -a "sshd.*Accepted password" /var/log/auth.log | tail -n 5
```

## Failed SSH Authentication

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password"
```

```bash
sudo grep -a "sshd.*Failed password" /var/log/auth.log
```

## Count Failed SSH Events

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password" | wc -l
```

## Correlate Authentication Events

```bash
sudo journalctl -u ssh --no-pager | grep -E "Accepted password|Failed password" | tail -n 10
```

## Active SSH Connections

```bash
sudo ss -tnp | grep ':22'
```

## Privileged Activity

```bash
sudo grep -a "COMMAND=/usr/bin/whoami" /var/log/auth.log | tail -n 3
```

```bash
sudo whoami
```

## Current User

```bash
whoami
```

## SSH Connection Test

```bash
ssh lexa@192.168.56.10
```

---

# 38. Command Explanation Cheat Sheet

| Command | SOC Purpose |
|---|---|
| `ip -br addr` | Identify interfaces and IP addresses |
| `ip route` | Inspect routing |
| `ip route get <IP>` | Determine route and source IP |
| `ping` | Verify basic connectivity |
| `systemctl status ssh` | Check SSH service state |
| `systemctl is-active ssh` | Quickly verify SSH state |
| `ss -lntp` | Identify listening TCP services |
| `ss -tnp` | Inspect active TCP connections |
| `journalctl -u ssh` | Investigate SSH service logs |
| `grep` | Search security logs |
| `tail` | View recent log entries |
| `wc -l` | Count filtered records |
| `whoami` | Identify current user |
| `sudo whoami` | Verify privileged context |
| `ssh` | Test SSH authentication |

---

# 39. Evidence Sources

The primary evidence sources used in the investigation were:

### SSH systemd journal

```text
journalctl -u ssh
```

### Authentication log

```text
/var/log/auth.log
```

### Network configuration

```text
ip -br addr
```

### Routing information

```text
ip route
```

### Socket information

```text
ss
```

### Service state

```text
systemctl
```

These sources provided enough information to establish the authentication timeline and investigate the observed SSH activity.

---

# 40. Investigation Findings

## Finding 1 — SSH Was Active

The SSH service was confirmed to be running.

```text
Active: active (running)
```

---

## Finding 2 — TCP/22 Was Listening

The SSH daemon was listening on port 22.

```text
0.0.0.0:22
[::]:22
```

---

## Finding 3 — Network Connectivity Was Available

Kali successfully communicated with Ubuntu over the SOC-LAB network.

```text
192.168.56.20 → 192.168.56.10
```

---

## Finding 4 — Successful Authentication Was Observed

Successful SSH authentication events were recorded for:

```text
User:
lexa

Source:
192.168.56.20
```

---

## Finding 5 — Failed Authentication Was Observed

Three actual failed SSH authentication events were identified.

Source:

```text
192.168.56.20
```

Target:

```text
192.168.56.10
```

Account:

```text
lexa
```

---

## Finding 6 — Source IP Was Validated

The source IP from the SSH logs matched the Kali SOC-LAB interface:

```text
Kali eth1:
192.168.56.20/24
```

---

## Finding 7 — Privileged Activity Was Analyst-Generated

The observed sudo activity was generated during investigation.

No malicious privilege escalation was confirmed.

---

## Finding 8 — No Confirmed Compromise

The investigation did not identify evidence sufficient to confirm compromise of the Ubuntu server.

---

# 41. Analyst Assessment

The observed SSH authentication failures represented a controlled security-testing scenario.

The activity demonstrated the type of telemetry that a SOC Analyst may encounter when monitoring SSH authentication.

The investigation showed how an analyst can move from a raw authentication event to a structured conclusion by correlating:

```text
Timestamp
+
Username
+
Source IP
+
Target Host
+
Authentication Result
+
Network Information
+
Privilege Activity
+
Session Context
```

The source IP was confirmed as the Kali test workstation, and the failed authentication attempts were intentionally generated within the isolated laboratory.

Therefore, the incident was assessed as:

```text
Low Severity
Controlled Security Testing
No Confirmed Compromise
Closed
```

---

# 42. Production SOC Recommendations

Although this lab incident was benign, the same pattern in a production environment could require further investigation.

## Authentication Monitoring

Monitor:

- Failed SSH authentication
- Successful SSH authentication
- Repeated authentication failures
- Authentication from unusual locations
- Authentication for disabled or unexpected accounts

## Source IP Monitoring

Investigate:

- External source IPs
- Repeated attempts from a single source
- Multiple usernames targeted by one source
- Known malicious IP addresses
- Geographically unusual authentication sources

## Account Monitoring

Investigate:

- Privileged accounts
- Service accounts
- Disabled accounts
- Unexpected usernames
- Repeated attempts against administrative accounts

## Post-Authentication Investigation

After a suspicious successful login, review:

- Shell activity
- `sudo` activity
- New processes
- New users
- File modifications
- Network connections
- Persistence mechanisms
- Other authentication events

## SSH Hardening

Recommended production controls include:

- Use SSH keys where appropriate
- Disable password authentication where operationally feasible
- Disable direct root SSH login
- Restrict SSH access to authorized networks
- Apply firewall controls
- Use rate limiting or account lockout controls where appropriate
- Monitor authentication logs centrally
- Configure SIEM alerts

---

# 43. Example Detection Logic

A basic SOC detection concept for SSH could be:

```text
IF
multiple SSH authentication failures
occur from the same source IP
within a short period
THEN
generate a security alert
```

A stronger correlation could be:

```text
Multiple failed SSH authentications
        +
Successful SSH authentication
        +
Same source IP
        ↓
Higher-priority investigation
```

This demonstrates why correlation is more valuable than looking at individual events in isolation.

---

# 44. Level 2 SOC Capabilities Demonstrated

This project currently demonstrates the following Level 2 SOC skills:

- Linux command-line investigation
- SSH monitoring
- Authentication log analysis
- `journalctl` usage
- `auth.log` analysis
- Source IP identification
- Network validation
- Event correlation
- Timeline creation
- Privilege activity analysis
- Incident classification
- Evidence collection
- MITRE ATT&CK mapping
- Incident documentation
- Analyst reasoning
- Distinguishing benign activity from suspicious activity

---

# 45. Level 3 Future Enhancement

The next stage of the project is planned as a Level 3 SIEM implementation using Wazuh.

Potential enhancements include:

```text
Ubuntu Server
      ↓
Wazuh Agent
      ↓
Wazuh Manager
      ↓
Wazuh Indexer
      ↓
Wazuh Dashboard
```

The Level 3 implementation can provide:

- Centralized log collection
- SSH authentication alerts
- Detection rules
- Security dashboards
- File integrity monitoring
- Authentication monitoring
- MITRE ATT&CK visualization
- Alert investigation
- Security event correlation
- Detection engineering
- Automated response capabilities

The Wazuh implementation is a future enhancement and is **not part of the current Level 2 implementation**.

---

# 46. Final Conclusion

This investigation demonstrates a complete Linux SSH security monitoring and investigation workflow using native Linux tools.

The analyst successfully:

1. Verified the network.
2. Verified SSH service availability.
3. Confirmed TCP/22 was listening.
4. Established SSH connectivity.
5. Generated controlled authentication events.
6. Collected SSH security logs.
7. Identified successful authentication events.
8. Identified failed authentication events.
9. Confirmed three actual failed SSH events.
10. Identified the source IP.
11. Validated the source IP against the Kali test workstation.
12. Correlated authentication events.
13. Investigated privilege-related activity.
14. Distinguished analyst activity from malicious activity.
15. Built an incident timeline.
16. Mapped the behavior to MITRE ATT&CK T1110.
17. Assessed the incident as controlled security testing.
18. Documented the investigation.

The project therefore demonstrates the core workflow expected from an entry-level SOC analyst:

```text
MONITOR
   ↓
DETECT
   ↓
INVESTIGATE
   ↓
CORRELATE
   ↓
ASSESS
   ↓
DOCUMENT
```

The current implementation represents the **Level 2 — Interview-Ready** stage.

The next major enhancement is the **Level 3 — SIEM and Detection Engineering** stage using Wazuh.

---

## Project Information

**Project Name:**

Linux SSH Security Monitoring & Incident Investigation Lab

**Repository:**

`linux-ssh-security-monitoring-lab`

**Primary Technologies:**

- Ubuntu Server
- Kali Linux
- Oracle VirtualBox
- OpenSSH
- systemd journal
- `/var/log/auth.log`
- Linux networking tools
- MITRE ATT&CK

**Primary Security Area:**

SSH Authentication Monitoring and Incident Investigation

**Current Project Level:**

Level 2 — Interview-Ready SOC Investigation Lab
