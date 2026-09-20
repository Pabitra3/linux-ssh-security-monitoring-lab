# SSH Security Monitoring & Investigation

## SOC Analyst L1 — SSH Command Reference

---

## 1. Overview

This document contains the SSH-specific commands and investigation procedures used in the Linux SSH Security Monitoring & Incident Investigation Lab.

The objective is to demonstrate how a SOC Analyst L1 can monitor and investigate SSH activity on a Linux server.

The investigation covers:

- SSH service verification
- SSH port verification
- SSH connectivity
- Successful authentication
- Failed authentication
- SSH log analysis
- Source IP identification
- User identification
- Session validation
- Privileged activity
- Authentication event correlation
- Basic SSH security investigation

The lab uses an isolated VirtualBox network consisting of a Kali Linux test workstation and an Ubuntu Server monitored host.

---

# 2. SSH Lab Architecture

```text
                    SOC-LAB Internal Network
                         192.168.56.0/24
                                |
              +-----------------+-----------------+
              |                                   |
              |                                   |
       Kali Linux                         Ubuntu Server
       Test Machine                       Monitored Host
       192.168.56.20                      192.168.56.10
              |                                   |
              |                                   |
              +-------- SSH / TCP 22 ------------+
```

### Source

```text
Kali Linux
192.168.56.20
```

### Destination

```text
Ubuntu Server
192.168.56.10
```

### Protocol

```text
SSH
```

### Port

```text
TCP/22
```

### Account

```text
lexa
```

---

# 3. What Is SSH?

SSH stands for:

**Secure Shell**

SSH is a network protocol used to securely access and administer remote systems.

A typical SSH connection contains:

```text
SSH Client
     ↓
Network
     ↓
SSH Server
```

In this lab:

```text
Kali Linux
    ↓
SSH
    ↓
Ubuntu Server
```

Kali acts as the client/test workstation and Ubuntu acts as the monitored SSH server.

---

# 4. Why SOC Analysts Monitor SSH

SSH is an important security telemetry source because attackers frequently target remote-access services.

Potential SSH-related threats include:

- Password guessing
- Brute-force attacks
- Credential stuffing
- Unauthorized remote access
- Account enumeration
- Compromised credentials
- Privilege escalation after login
- Persistence through unauthorized accounts or keys

A SOC analyst therefore monitors:

```text
Source IP
Username
Timestamp
Authentication result
Authentication method
Target host
Session activity
Privilege activity
```

---

# 5. Verify SSH Installation and Service

The first step is confirming that the SSH service is operational.

### Command

```bash
sudo systemctl status ssh --no-pager
```

### Purpose

This displays the current SSH service status.

Important information includes:

- Whether SSH is installed
- Whether the service is running
- Process information
- Recent service messages

### Observed State

```text
Active: active (running)
```

### SOC Interpretation

SSH was active and available for authentication.

This establishes that the monitored service was operational before investigating authentication activity.

---

# 6. Quickly Check SSH Service State

A shorter service check can be performed using:

```bash
systemctl is-active ssh
```

Expected output:

```text
active
```

### Why Use This?

This command is useful when an analyst only needs the current state of the SSH service.

Possible results include:

```text
active
inactive
failed
unknown
```

---

# 7. Start SSH Service

During the initial lab configuration, the SSH service was found inactive.

It was started using:

```bash
sudo systemctl start ssh
```

After starting it, the state was verified using:

```bash
systemctl is-active ssh
```

The final state was:

```text
active
```

### SOC Relevance

A service must be operational before meaningful service-level testing can occur.

---

# 8. Verify SSH Listening Port

SSH normally listens on TCP port 22.

The listening state was verified with:

```bash
sudo ss -lntp | grep ':22'
```

### Command Breakdown

| Option | Meaning |
|---|---|
| `ss` | Display socket information |
| `-l` | Listening sockets |
| `-n` | Show numerical addresses |
| `-t` | TCP sockets |
| `-p` | Show process information |
| `grep ':22'` | Filter for port 22 |

### Observed Result

The SSH daemon was listening on:

```text
0.0.0.0:22
```

and:

```text
[::]:22
```

### Interpretation

SSH was listening on IPv4 and IPv6.

---

# 9. Why Port 22 Matters

TCP/22 is the default SSH port.

A listening SSH service creates a remote-access surface.

From a SOC perspective, the analyst should know:

```text
Which host?
Which service?
Which port?
Which account?
Which source?
Which authentication result?
```

This information forms the basis of an authentication investigation.

---

# 10. Test SSH Connectivity

The Kali workstation was used to connect to Ubuntu.

### Command

```bash
ssh lexa@192.168.56.10
```

This specifies:

```text
Username: lexa
Destination: 192.168.56.10
Protocol: SSH
Port: 22
```

The command establishes an SSH session with the Ubuntu server.

---

# 11. Validate the Remote Host

After successfully connecting through SSH, the remote hostname was checked.

### Command

```bash
hostname
```

Observed result:

```text
ubuntu-server
```

### Purpose

This confirms that the SSH session reached the intended server.

---

# 12. Validate the Authenticated User

The authenticated Linux user was checked using:

```bash
whoami
```

Observed result:

```text
lexa
```

### SOC Relevance

This verifies which account was used for the SSH session.

During an incident, the username is an important investigation field.

---

# 13. Successful SSH Authentication

Successful password authentication can be identified in `/var/log/auth.log`.

### Command

```bash
sudo grep -a "sshd.*Accepted password" /var/log/auth.log | tail -n 5
```

The key phrase is:

```text
Accepted password
```

A representative event is:

```text
Accepted password for lexa from 192.168.56.20
```

### Important Information

```text
Authentication:
Successful

Username:
lexa

Source IP:
192.168.56.20

Protocol:
SSH
```

---

# 14. SOC Interpretation of Successful Login

A successful authentication should not automatically be classified as malicious.

The analyst should investigate:

1. Was the account expected?
2. Was the source IP authorized?
3. Was the login time normal?
4. Was the authentication method expected?
5. What actions occurred after login?

A successful authentication becomes more suspicious when combined with other indicators.

For example:

```text
Multiple Failed Logins
        ↓
Successful Authentication
        ↓
Privileged Commands
        ↓
Suspicious Network Activity
```

This type of correlation may indicate account compromise.

---

# 15. Failed SSH Authentication

Failed authentication events can be searched using:

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password"
```

The key phrase is:

```text
Failed password
```

A representative event is:

```text
Failed password for lexa from 192.168.56.20
```

### Important Fields

| Field | Value |
|---|---|
| Result | Failed |
| Account | lexa |
| Source IP | 192.168.56.20 |
| Service | SSH |
| Port | 22 |
| Protocol | TCP |

---

# 16. Controlled Failed Authentication Test

A deliberate incorrect password was entered during a controlled SSH test.

### Command

```bash
ssh lexa@192.168.56.10
```

An incorrect password produced:

```text
Permission denied, please try again.
```

The Ubuntu SSH logs subsequently recorded the authentication failure.

This allowed the lab to validate the complete detection process:

```text
Generate Event
      ↓
SSH Server Receives Event
      ↓
Authentication Fails
      ↓
Event Written to Logs
      ↓
SOC Analyst Searches Logs
      ↓
Event Investigated
```

---

# 17. Confirm Actual Failed Events

The SSH service journal was used to identify the actual failed authentication events.

### Command

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password"
```

The investigation identified three actual failed SSH authentication events:

```text
2026-09-09 07:58:55
2026-09-09 13:09:40
2026-09-09 13:11:02
```

All three originated from:

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

# 18. Count Actual Failed SSH Events

After filtering the SSH service journal:

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password" | wc -l
```

The investigation resulted in:

```text
3
```

actual failed SSH authentication events.

---

# 19. Why Raw auth.log Counting Can Be Misleading

During investigation, commands executed with `sudo` can themselves generate authentication/audit records.

For example:

```bash
sudo grep -a "Failed password" /var/log/auth.log
```

may cause related sudo activity to appear in the authentication log.

Therefore, a raw count against `/var/log/auth.log` can produce misleading results if all matching records are treated as SSH authentication failures.

The analyst must distinguish:

```text
Actual SSH authentication event
```

from:

```text
Analyst-generated sudo/audit event
```

### Preferred Investigation

Use the SSH service journal:

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password"
```

This provides a more focused view of events generated by the SSH service.

---

# 20. Identify the Source IP

The source IP is critical during authentication investigations.

### Command

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password"
```

The events showed:

```text
from 192.168.56.20
```

Therefore:

```text
Source IP:
192.168.56.20
```

---

# 21. Validate the Source IP on Kali

Kali's network configuration was checked using:

```bash
ip -br addr
```

The SOC-LAB interface was:

```text
eth1
```

with:

```text
192.168.56.20/24
```

This matched the source IP recorded by the Ubuntu SSH logs.

### Correlation

```text
Kali eth1
    ↓
192.168.56.20
    ↓
SSH connection
    ↓
Ubuntu 192.168.56.10
```

### Investigation Conclusion

The source IP belonged to the known Kali test workstation in the controlled laboratory.

---

# 22. Validate the Route

The exact route from Kali to Ubuntu was verified using:

```bash
ip route get 192.168.56.10
```

The result was:

```text
192.168.56.10 dev eth1 src 192.168.56.20
```

### Interpretation

```text
Destination:
192.168.56.10

Interface:
eth1

Source:
192.168.56.20
```

This provided additional evidence linking the SSH source address to Kali.

---

# 23. Review SSH Journal

The complete SSH journal can be viewed with:

```bash
sudo journalctl -u ssh --no-pager
```

The most recent entries can be limited with:

```bash
sudo journalctl -u ssh --no-pager -n 20
```

### SOC Use

The journal can reveal:

- Authentication events
- Session creation
- Session termination
- Failed authentication
- Successful authentication
- SSH daemon activity

---

# 24. Review Authentication Log

Ubuntu's authentication log can be inspected with:

```bash
sudo tail -n 50 /var/log/auth.log
```

The complete file can also be searched using:

```bash
sudo grep -a "sshd" /var/log/auth.log
```

### SOC Use

This can help investigate:

- SSH authentication
- sudo activity
- PAM events
- User sessions
- Authentication failures
- Authentication successes

---

# 25. Correlate Successful and Failed Events

Successful and failed authentication events were correlated using:

```bash
sudo journalctl -u ssh --no-pager | grep -E "Accepted password|Failed password" | tail -n 10
```

### Purpose

This creates a simplified authentication event stream.

The analyst can compare:

```text
Accepted
```

against:

```text
Failed
```

and analyze their timestamps and source addresses.

---

# 26. Why Event Correlation Matters

Individual events may not provide enough context.

For example:

```text
One failed login
```

may simply be a user entering the wrong password.

However:

```text
20 failed logins
+
Same source IP
+
Multiple accounts targeted
+
Successful login
```

would be significantly more suspicious.

Therefore:

```text
Individual Event
      ↓
Correlation
      ↓
Context
      ↓
Security Assessment
```

is a fundamental SOC investigation process.

---

# 27. SSH Authentication Timeline

The investigation identified the following relevant events:

| Timestamp | Event | Source IP | Account |
|---|---|---|---|
| 2026-09-04 14:34:53 | Successful SSH authentication | 192.168.56.20 | lexa |
| 2026-09-09 07:23:02 | Successful SSH authentication | 192.168.56.20 | lexa |
| 2026-09-09 07:58:55 | Failed SSH authentication | 192.168.56.20 | lexa |
| 2026-09-09 13:09:40 | Failed SSH authentication | 192.168.56.20 | lexa |
| 2026-09-09 13:11:02 | Failed SSH authentication | 192.168.56.20 | lexa |

The three failed events were confirmed through the SSH service journal.

---

# 28. Check Active SSH Connections

Current SSH TCP connections can be investigated using:

```bash
sudo ss -tnp | grep ':22'
```

### Purpose

This shows active TCP connections involving port 22.

Potential information includes:

- Source address
- Destination address
- Connection state
- Associated process

### Important Note

If the command returns nothing, it means there was no matching active SSH TCP connection at that moment.

It does not mean SSH is broken.

The SSH service can be running while no client is currently connected.

---

# 29. Investigate Privileged Activity

After authentication, the analyst should investigate whether privileged commands were executed.

A controlled sudo event was investigated using:

```bash
sudo grep -a "COMMAND=/usr/bin/whoami" /var/log/auth.log | tail -n 3
```

The resulting record contained information such as:

```text
USER=root
COMMAND=/usr/bin/whoami
```

---

# 30. Verify Normal User Context

The current user was checked using:

```bash
whoami
```

Result:

```text
lexa
```

This represents the normal authenticated user context.

---

# 31. Verify Privileged Context

A controlled sudo command was executed:

```bash
sudo whoami
```

Result:

```text
root
```

This demonstrates:

```text
Normal User:
lexa

sudo Privileged Context:
root
```

The corresponding sudo activity was recorded in the authentication logs.

---

# 32. Important Privilege Investigation Lesson

The presence of:

```text
USER=root
```

does not automatically indicate malicious privilege escalation.

The analyst must examine the context.

In this lab, the `sudo whoami` command was deliberately executed by the analyst during the investigation.

Therefore:

```text
Observed root activity
        ↓
Analyst-generated sudo command
        ↓
Not confirmed malicious escalation
```

This demonstrates why security analysts must correlate logs with known analyst activity.

---

# 33. SSH Log Indicators

Important SSH log indicators include:

### Successful Authentication

```text
Accepted password
```

### Failed Authentication

```text
Failed password
```

### Invalid User

```text
Invalid user
```

### Session Opened

```text
session opened
```

### Session Closed

```text
session closed
```

### Authentication Failure

```text
authentication failure
```

These keywords can be used during manual investigation and later converted into SIEM detection rules.

---

# 34. Useful SSH Investigation Searches

## Successful Logins

```bash
sudo grep -a "Accepted password" /var/log/auth.log
```

## Failed Logins

```bash
sudo grep -a "Failed password" /var/log/auth.log
```

## Invalid Users

```bash
sudo grep -a "Invalid user" /var/log/auth.log
```

## SSH Authentication Failures

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password"
```

## Successful and Failed Authentication

```bash
sudo journalctl -u ssh --no-pager | grep -E "Accepted password|Failed password"
```

## SSH Sessions

```bash
sudo journalctl -u ssh --no-pager | grep -E "session opened|session closed"
```

---

# 35. Search for a Specific User

To investigate SSH activity for the `lexa` account:

```bash
sudo grep -a "lexa" /var/log/auth.log | tail -n 20
```

### SOC Purpose

This can help build an account-specific activity timeline.

---

# 36. Search by Source IP

To investigate the known lab source:

```bash
sudo grep -a "192.168.56.20" /var/log/auth.log
```

### SOC Purpose

This allows an analyst to correlate multiple authentication and authorization events associated with the same source.

---

# 37. Search SSH Logs by Time

The journal can be filtered by time.

Example:

```bash
sudo journalctl -u ssh --since "2026-09-09 07:00:00" --until "2026-09-09 09:00:00"
```

### Purpose

Time-based filtering is useful during incident investigation when an analyst already knows the approximate incident window.

---

# 38. Search Recent SSH Events

For the latest SSH events:

```bash
sudo journalctl -u ssh --since "1 hour ago"
```

### SOC Use

This can be useful during live monitoring or triage.

---

# 39. Authentication Event Correlation Model

A basic SSH investigation can be represented as:

```text
Source IP
    +
Username
    +
Timestamp
    +
Authentication Result
    +
Target Host
    +
Session Activity
    +
Privilege Activity
    ↓
Incident Assessment
```

This model prevents the analyst from making conclusions based on a single field.

---

# 40. Suspicious SSH Pattern

A potentially suspicious pattern could look like:

```text
Multiple Failed Authentication Attempts
                  ↓
          Same Source IP
                  ↓
          Same Target Host
                  ↓
       Successful Authentication
                  ↓
          Privileged Activity
```

This pattern should trigger deeper investigation in a production SOC.

---

# 41. Benign SSH Pattern in This Lab

The observed laboratory activity was:

```text
Known Kali Test Machine
        ↓
SSH Authentication Testing
        ↓
Controlled Failed Attempts
        ↓
Logs Generated
        ↓
SOC Investigation
        ↓
Source Validated
        ↓
No Confirmed Compromise
```

The source was known and authorized.

Therefore, the event was classified as controlled security testing.

---

# 42. MITRE ATT&CK Mapping

The repeated failed authentication behavior can be mapped to:

```text
T1110 — Brute Force
```

### Context

The laboratory simulated repeated authentication failures against SSH.

This provides practical experience mapping observed behavior to MITRE ATT&CK.

The mapping is for defensive analysis and does not imply that an actual attacker compromised the system.

---

# 43. Incident Classification

The investigated activity was classified as:

```text
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

### Target

```text
Ubuntu Server
192.168.56.10
```

### Source

```text
Kali Test Workstation
192.168.56.20
```

### Account

```text
lexa
```

### Service

```text
SSH
```

### Port

```text
22/TCP
```

---

# 44. Investigation Findings

## Finding 1 — SSH Service Active

SSH was confirmed to be running:

```text
Active: active (running)
```

## Finding 2 — SSH Port Listening

TCP/22 was listening on IPv4 and IPv6.

```text
0.0.0.0:22
[::]:22
```

## Finding 3 — Network Connectivity

Kali successfully communicated with Ubuntu.

```text
192.168.56.20 → 192.168.56.10
```

## Finding 4 — Successful Authentication

Successful SSH authentication events were observed for:

```text
lexa
```

from:

```text
192.168.56.20
```

## Finding 5 — Failed Authentication

Three actual failed SSH authentication events were identified.

## Finding 6 — Source IP Validated

The source IP matched Kali's SOC-LAB interface.

## Finding 7 — Privileged Activity

Sudo activity was observed but was analyst-generated.

## Finding 8 — No Confirmed Compromise

No confirmed compromise was identified.

---

# 45. SOC Analyst Investigation Checklist

When investigating suspicious SSH activity, use the following checklist.

## Host

- [ ] Identify target host
- [ ] Confirm target IP
- [ ] Confirm SSH service status
- [ ] Confirm port 22 status

## Authentication

- [ ] Identify username
- [ ] Identify successful logins
- [ ] Identify failed logins
- [ ] Count actual failures
- [ ] Identify authentication method

## Network

- [ ] Identify source IP
- [ ] Validate source IP
- [ ] Identify interface
- [ ] Check active connections

## Correlation

- [ ] Compare failed and successful authentication
- [ ] Check timestamps
- [ ] Check repeated attempts
- [ ] Check source consistency

## Post-Authentication

- [ ] Investigate session activity
- [ ] Investigate sudo activity
- [ ] Investigate suspicious commands
- [ ] Investigate new accounts
- [ ] Investigate persistence

## Assessment

- [ ] Determine whether activity is expected
- [ ] Determine whether compromise occurred
- [ ] Assign severity
- [ ] Map to MITRE ATT&CK
- [ ] Document evidence

---

# 46. Production SSH Monitoring Recommendations

In a production environment, an organization should monitor:

### Authentication Failures

Alert on repeated failures from:

- One source IP
- Multiple source IPs
- External addresses
- Unusual geographic locations

### Successful Logins

Monitor:

- Unexpected accounts
- Privileged accounts
- Unusual source IPs
- Unusual login times
- Successful login after repeated failures

### Privileged Activity

Monitor:

- `sudo`
- Root shell access
- New administrative users
- SSH key modifications
- Sensitive configuration changes

### Network Activity

Monitor:

- Unexpected SSH connections
- New external connections
- Unusual destinations
- Long-lived sessions

---

# 47. Example Detection Logic

A basic detection rule could be:

```text
IF
5 or more failed SSH authentication attempts
occur from the same source IP
within 5 minutes

THEN
generate a high-priority authentication alert.
```

A correlation rule could be:

```text
IF
multiple failed SSH authentications
are followed by
a successful authentication
from the same source IP

THEN
generate a higher-priority investigation alert.
```

The exact thresholds should be tuned for the organization's environment.

---

# 48. Manual Investigation vs SIEM

The current project demonstrates manual Linux investigation.

### Current Level 2

```text
Ubuntu Logs
     ↓
journalctl
     ↓
grep
     ↓
Manual Correlation
     ↓
Timeline
     ↓
Incident Report
```

### Planned Level 3

```text
Ubuntu
   ↓
Wazuh Agent
   ↓
Wazuh Manager
   ↓
Detection Rules
   ↓
Alerts
   ↓
Dashboard
   ↓
SOC Investigation
```

The Level 3 implementation will automate much of the log collection and alerting process.

---

# 49. Level 2 Skills Demonstrated

This SSH investigation demonstrates:

- Linux command-line skills
- SSH administration
- SSH security monitoring
- Authentication log analysis
- `journalctl`
- `grep`
- `systemctl`
- `ss`
- Network troubleshooting
- Source IP investigation
- Authentication correlation
- Timeline analysis
- Privilege investigation
- MITRE ATT&CK mapping
- Incident classification
- Incident documentation

These are directly relevant to an entry-level SOC Analyst role.

---

# 50. Final Investigation Summary

The SSH investigation established:

```text
Target:
Ubuntu Server 192.168.56.10

Source:
Kali Linux 192.168.56.20

Account:
lexa

Service:
SSH

Port:
22/TCP

Actual Failed Authentication Events:
3

Confirmed Compromise:
No

Incident Classification:
Controlled Security Testing

Severity:
Low

Status:
Closed

MITRE ATT&CK:
T1110 — Brute Force
```

The source IP was validated against the Kali test workstation.

The authentication events were generated within the controlled laboratory.

The privileged activity observed during investigation was analyst-generated and was not evidence of malicious privilege escalation.

---

# 51. Final SOC Workflow

The complete SSH investigation workflow used in this project was:

```text
                 SSH Security Event
                         ↓
                Identify Target
                         ↓
                Verify SSH Service
                         ↓
               Verify Port 22
                         ↓
              Generate Controlled
              Authentication Events
                         ↓
              Collect SSH Logs
                         ↓
        +----------------+----------------+
        ↓                                 ↓
 Successful Authentication        Failed Authentication
        ↓                                 ↓
 Identify User                    Identify User
        ↓                                 ↓
 Identify Source IP               Identify Source IP
        ↓                                 ↓
        +----------------+----------------+
                         ↓
                  Correlate Events
                         ↓
                 Build Timeline
                         ↓
              Investigate Privilege
                         ↓
                Assess the Incident
                         ↓
              MITRE ATT&CK Mapping
                         ↓
               Document Findings
                         ↓
                  Close Incident
```

---

# 52. Conclusion

This document demonstrates how SSH authentication activity can be investigated using native Linux tools.

The lab successfully demonstrated:

- SSH service verification
- SSH port monitoring
- Successful authentication analysis
- Failed authentication detection
- Source IP identification
- Network validation
- Event correlation
- Authentication timeline construction
- Privilege activity analysis
- Incident classification
- MITRE ATT&CK mapping
- SOC investigation methodology

The project currently represents the:

**Level 2 — Interview-Ready SOC Investigation**

stage.

The planned Level 3 enhancement is centralized SIEM monitoring using Wazuh, which will introduce centralized log collection, automated detection rules, dashboards, alerting, and additional detection engineering capabilities.

---

## Project

**Linux SSH Security Monitoring & Incident Investigation Lab**

**Repository:**

`linux-ssh-security-monitoring-lab`

**Primary Security Use Case:**

SSH Authentication Monitoring and Incident Investigation

**Current Stage:**

Level 2 — Interview-Ready SOC Analyst Lab
