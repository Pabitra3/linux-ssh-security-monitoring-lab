# MITRE ATT&CK Mapping & Detection Logic

## 1. Overview

This document maps the SSH authentication activity observed in the **Linux SSH Security Monitoring & Incident Investigation Lab** to the **MITRE ATT&CK framework**.

The objective is to demonstrate how a SOC analyst moves from:

```text
Security Event
      ↓
Evidence
      ↓
Technique Identification
      ↓
Detection Logic
      ↓
Investigation
      ↓
Incident Classification
```

The observed activity was generated intentionally in an isolated lab using Kali Linux against Ubuntu Server.

---

# 2. Incident Context

| Attribute | Value |
|---|---|
| Target | Ubuntu Server |
| Target IP | `192.168.56.10` |
| Source/Test System | Kali Linux |
| Source IP | `192.168.56.20` |
| Account Targeted | `lexa` |
| Protocol | SSH |
| Destination Port | `22` |
| Actual Failed SSH Events | `3` |
| Classification | Controlled Security Testing |
| Severity | Low |
| Status | Closed |

The failed authentication activity was intentionally generated for SOC investigation practice.

There was **no confirmed compromise** in the lab.

---

# 3. MITRE ATT&CK Technique

## T1110 — Brute Force

The observed repeated SSH authentication failures are mapped to:

**MITRE ATT&CK Technique: T1110 — Brute Force**

The technique describes adversaries attempting to gain access to accounts by repeatedly attempting authentication.

In this lab, repeated failed SSH password authentication events provide the evidence pattern that can be associated with this technique.

### Important Context

The mapping does **not** mean that a real attacker compromised the system.

This project intentionally generated authentication failures to demonstrate how a SOC analyst could detect and investigate a behavior associated with T1110.

Therefore:

```text
ATT&CK Technique Match
        ≠
Confirmed Malicious Activity
```

The analyst must consider context before declaring an incident malicious.

---

# 4. Evidence Supporting the Mapping

The investigation identified three actual failed SSH authentication events.

Example event:

```text
2026-09-09T07:58:55.890135+00:00 ubuntu-server sshd[41403]:
Failed password for lexa from 192.168.56.20 port 44560 ssh2
```

Additional failed events were observed at:

```text
2026-09-09 13:09:40
2026-09-09 13:11:02
```

The source IP was consistently:

```text
192.168.56.20
```

The destination was the Ubuntu SSH service:

```text
192.168.56.10:22
```

---

# 5. Authentication Evidence

The primary Linux evidence sources were:

### SSH Journal

```bash
sudo journalctl -u ssh --no-pager
```

### Failed Authentication Search

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password"
```

### Successful Authentication Search

```bash
sudo journalctl -u ssh --no-pager | grep "Accepted password"
```

### Authentication Log

```bash
sudo grep -a "sshd.*Failed password" /var/log/auth.log
```

These sources provide authentication-level evidence that complements network-level information.

---

# 6. Network Correlation

The source IP was validated against the lab network configuration.

Kali:

```text
IP:        192.168.56.20
Interface: eth1
```

Ubuntu:

```text
IP:        192.168.56.10
Interface: enp0s8
```

Kali routing validation:

```bash
ip route get 192.168.56.10
```

Result:

```text
192.168.56.10 dev eth1 src 192.168.56.20
```

This confirmed that Kali used `eth1` and source IP `192.168.56.20` when reaching Ubuntu.

The correlation was:

```text
Kali
192.168.56.20
     |
     | SOC-LAB
     | TCP/22
     |
     v
Ubuntu
192.168.56.10
     |
     v
sshd
     |
     v
Authentication Log
```

---

# 7. Detection Logic

A basic SOC detection rule for SSH brute-force-like behavior can be expressed as:

```text
IF
    multiple SSH authentication failures
    occur from the same source IP
    against the same destination
    within a defined time window

THEN
    generate a security alert
```

For example:

```text
IF
    Failed SSH authentication >= 5
    from the same source IP
    within 5 minutes

THEN
    Alert: Possible SSH Brute Force
```

This is an example detection threshold, not a universal production standard.

The threshold should be tuned according to the organization's normal authentication behavior.

---

# 8. Detection Pseudocode

```text
FOR each SSH authentication failure:

    Extract:
        timestamp
        source_ip
        username
        destination_host

    Group events by:
        source_ip
        destination_host

    Count failures within a time window

    IF failure_count >= threshold:

        Generate alert

        Include:
            source IP
            target host
            username
            failure count
            first event
            latest event
```

This represents the basic logic that a SIEM or detection-engineering system could implement.

---

# 9. Example Linux Detection Command

A simple manual detection approach is:

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password"
```

This retrieves SSH authentication failures.

An analyst can then inspect:

- Timestamp
- Username
- Source IP
- Source port
- Authentication method
- Frequency of attempts

For the lab, the clean SSH journal output identified three actual failed authentication events.

---

# 10. Why Simple Log Counting Can Be Misleading

During the investigation, a raw command such as:

```bash
sudo grep -a "sshd.*Failed password" /var/log/auth.log | wc -l
```

produced an unexpectedly high number.

The reason was that commands used during the investigation were themselves recorded in system logs, creating self-referential matches.

This is an important SOC lesson:

> Detection logic must distinguish the actual security event from analyst-generated log entries.

The cleaner investigation method was:

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password"
```

This allowed the actual SSH service events to be reviewed directly.

The final investigation identified:

```text
3 actual failed SSH authentication events
```

---

# 11. False Positive Considerations

Repeated failed SSH authentication does not automatically mean an attacker is performing a brute-force attack.

Possible legitimate causes include:

- User entering the wrong password
- Forgotten credentials
- Automated service using an expired password
- Administrative troubleshooting
- Misconfigured SSH clients
- Security testing
- Vulnerability scanning
- Authorized penetration testing

Therefore, an alert should be investigated using context.

---

# 12. SOC Investigation Questions

When an SSH brute-force alert fires, the analyst should ask:

### 1. Who is the source?

Check:

```text
Source IP
Hostname
Asset owner
Network segment
```

### 2. Who is being targeted?

Check:

```text
Username
Destination host
SSH service
```

### 3. How many attempts occurred?

Determine:

```text
Failure count
Frequency
Time window
```

### 4. Were any attempts successful?

Search:

```bash
sudo journalctl -u ssh --no-pager | grep "Accepted password"
```

A successful authentication after repeated failures is significantly more important than failures alone.

### 5. What happened after authentication?

If access was successful, investigate:

- Commands executed
- Privilege escalation
- File changes
- Persistence
- Network connections
- Account activity

### 6. Is the source authorized?

Determine whether the source is:

- Internal administrator
- Security scanner
- Penetration tester
- Application server
- Unknown host
- External system

---

# 13. Event Correlation

The project demonstrates correlation across several evidence sources.

## Network Evidence

```text
Source: 192.168.56.20
Destination: 192.168.56.10
Port: 22
Protocol: TCP
```

## Authentication Evidence

```text
Failed password for lexa
from 192.168.56.20
```

## Successful Authentication Evidence

Successful SSH events were also observed from the expected lab source.

## Privileged Activity

The investigation included `sudo` activity.

For example:

```text
COMMAND=/usr/bin/whoami
USER=root
```

This was analyst-generated investigation activity.

It was **not interpreted as malicious privilege escalation**.

This distinction is critical when building detection rules and investigating alerts.

---

# 14. Incident Assessment

The observed behavior matched a pattern associated with:

```text
T1110 — Brute Force
```

However, contextual analysis determined:

```text
Activity Type:
Controlled Security Testing
```

Therefore:

```text
Severity: Low
Status: Closed
Compromise: Not confirmed
```

The activity was intentionally generated as part of the lab.

---

# 15. MITRE ATT&CK Mapping Table

| Evidence | Observed Behavior | ATT&CK Mapping | Interpretation |
|---|---|---|---|
| Repeated failed SSH passwords | Multiple authentication failures | T1110 | Brute-force-like authentication behavior |
| SSH service on TCP/22 | Remote authentication service exposed | Supporting evidence | Provides the attack surface |
| Same source IP | Repeated attempts from `192.168.56.20` | Supporting evidence | Useful for correlation |
| Successful SSH events | Valid authentication occurred | Investigation evidence | Requires context and follow-up |
| `sudo` commands | Privileged commands recorded | Not automatically malicious | Analyst-generated investigation activity |

---

# 16. What a Real SOC Alert Could Look Like

```text
ALERT: Possible SSH Brute Force

Source IP:
192.168.56.20

Destination:
192.168.56.10

Destination Port:
22

Username:
lexa

Failed Attempts:
5

Time Window:
5 minutes

Detection:
Multiple SSH authentication failures

MITRE ATT&CK:
T1110 - Brute Force

Recommended Action:
Investigate source, validate authorization,
check successful authentication, and review
post-authentication activity.
```

This format is suitable for a SOC alert or SIEM detection.

---

# 17. Analyst Response Workflow

If this were a real production alert, the SOC workflow could be:

```text
Alert Generated
       ↓
Validate Alert
       ↓
Identify Source IP
       ↓
Identify Target Account
       ↓
Identify Target Host
       ↓
Review Failure Frequency
       ↓
Check Successful Logins
       ↓
Check Source Reputation / Ownership
       ↓
Review Post-Authentication Activity
       ↓
Determine Benign vs Suspicious
       ↓
Contain / Escalate if Required
       ↓
Document Investigation
       ↓
Close or Escalate Incident
```

---

# 18. Production Response Recommendations

If the source were confirmed malicious in a real environment, possible response actions could include:

1. Block or restrict the source IP where appropriate.
2. Review successful SSH authentication events.
3. Verify whether the targeted account is authorized to use SSH.
4. Reset compromised credentials if compromise is suspected.
5. Prefer SSH keys or stronger authentication mechanisms.
6. Disable unnecessary SSH exposure.
7. Apply rate limiting or account lockout controls where appropriate.
8. Centralize SSH logs in a SIEM.
9. Create alerts for repeated authentication failures.
10. Investigate activity following any successful login.

Response actions should always follow organizational procedures and authorization.

---

# 19. SIEM Detection Engineering Concept

The manual lab demonstrates the detection logic that can later be implemented in a SIEM such as **Wazuh**.

Current lab:

```text
Ubuntu SSH Logs
      ↓
Manual Analysis
      ↓
Analyst Detection
      ↓
Incident Documentation
```

Planned Level 3 architecture:

```text
Ubuntu
   ↓
Wazuh Agent
   ↓
Wazuh Manager
   ↓
Detection Rules
   ↓
Alert
   ↓
Dashboard
   ↓
SOC Analyst
```

The Level 3 implementation will allow the project to demonstrate centralized monitoring and automated detection rather than only manual log analysis.

---

# 20. Detection Engineering Improvements

A mature SSH brute-force detection should consider more than a simple failure count.

Potential enrichment includes:

### Source Reputation

Determine whether the source IP is:

- Known
- Unknown
- Internal
- External
- Previously malicious

### Account Criticality

A failed login against:

```text
root
administrator
service-account
```

may require different severity than a low-privilege test account.

### Time Pattern

Consider:

```text
5 failures in 5 minutes
50 failures in 10 minutes
100 failures in 1 hour
```

### Success After Failure

A particularly important pattern is:

```text
Multiple failures
       ↓
Successful authentication
       ↓
Post-login activity
```

This should generally receive higher investigation priority than repeated failures alone.

---

# 21. Detection Logic Example

A more complete detection model can be represented as:

```text
Failed SSH Authentication
        |
        v
Group by Source IP + Username + Destination
        |
        v
Count Events in Time Window
        |
        v
Threshold Exceeded?
       /      No   Yes
     |     |
   Stop    v
       Check Successful Login
             |
             v
       Successful Login?
          /                No          Yes
        |            |
   Medium/Low    Higher Priority
   Investigation  Investigation
```

This demonstrates how a SOC detection can evolve from simple thresholding into contextual correlation.

---

# 22. Interview Questions

## Q1. Which MITRE ATT&CK technique did you map this activity to?

I mapped the repeated failed SSH authentication behavior to **T1110 — Brute Force**.

The mapping describes the observed behavior pattern. It does not by itself prove that the activity was malicious.

## Q2. Why did you classify the incident as benign?

Because the authentication failures were intentionally generated from the Kali machine in an isolated SOC lab for security testing.

## Q3. How would you detect SSH brute force?

I would monitor failed SSH authentication events, group them by source IP and target account, apply a time-based threshold, and generate an alert when the threshold is exceeded.

## Q4. What would make the alert more serious?

A successful authentication following multiple failures, especially against a privileged or sensitive account, would increase the priority.

I would then investigate post-authentication activity and possible compromise.

## Q5. Why is context important in SOC detection?

The same technical event can have different meanings depending on context.

For example, repeated SSH failures could be:

- A legitimate user making mistakes
- An administrator troubleshooting
- Authorized penetration testing
- A malicious brute-force attempt

Therefore, detection should identify suspicious behavior while investigation determines the actual incident context.

## Q6. Why shouldn't you rely only on the number of failed logins?

Because a raw count can produce false positives and may include irrelevant or self-referential log entries.

The analyst should validate the events, source, target, timeframe, and surrounding activity.

---

# 23. Level 2 Outcome

This MITRE ATT&CK and detection-engineering exercise demonstrates practical SOC skills in:

- MITRE ATT&CK mapping
- Authentication-event analysis
- Brute-force detection concepts
- Threshold-based detection
- Time-window analysis
- Source-IP correlation
- False-positive analysis
- Alert validation
- Incident classification
- Detection engineering
- SOC investigation workflow
- Production response planning

The project now demonstrates not only how to collect logs, but also how to interpret a security behavior and translate it into a repeatable detection concept.

---

# 24. Final Conclusion

The SSH authentication activity observed in this lab provides a realistic example of a SOC investigation.

The analyst identified:

```text
Repeated SSH Authentication Failures
              ↓
Source IP Validation
              ↓
Network Correlation
              ↓
Authentication Log Analysis
              ↓
MITRE ATT&CK T1110 Mapping
              ↓
Detection Logic
              ↓
Context Validation
              ↓
Incident Classification
```

The final assessment was:

```text
Technique:
T1110 — Brute Force

Activity:
Controlled Security Testing

Actual Failed Events:
3

Source:
192.168.56.20

Target:
192.168.56.10:22

Compromise:
Not confirmed

Severity:
Low

Status:
Closed
```

This completes an important Level 2 SOC capability: converting raw Linux security events into **evidence-based detection, MITRE ATT&CK mapping, investigation, and incident assessment**.

The next major project stage is **Level 3 — Wazuh SIEM**, where these manually investigated events can be collected centrally, detected automatically, visualized in dashboards, and turned into real SOC alerts.
