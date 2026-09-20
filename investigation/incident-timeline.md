# SSH Security Incident Timeline

## 1. Incident Overview

This investigation analyzes SSH authentication activity observed on the Ubuntu Server during the SOC lab exercise.

The activity includes both successful and failed SSH authentication attempts originating from the Kali Linux test workstation.

---

## 2. Environment

| Component | Details |
|---|---|
| Target System | Ubuntu Server |
| Target IP | 192.168.56.10 |
| Test Workstation | Kali Linux |
| Source IP | 192.168.56.20 |
| Target Account | lexa |
| Service | OpenSSH |
| Protocol | SSH |
| Destination Port | 22 |
| Network | SOC-LAB (192.168.56.0/24) |
| Virtualization | Oracle VirtualBox |

---

## 3. Incident Timeline

| Time | Event | Source | Account | Result |
|---|---|---|---|---|
| Sep 04 14:34:53 | SSH authentication | 192.168.56.20 | lexa | Successful |
| Sep 09 07:23:02 | SSH authentication | 192.168.56.20 | lexa | Successful |
| Sep 09 07:58:55 | SSH authentication | 192.168.56.20 | lexa | Failed |
| Sep 09 13:09:40 | SSH authentication | 192.168.56.20 | lexa | Failed |
| Sep 09 13:11:02 | SSH authentication | 192.168.56.20 | lexa | Failed |

> Note: The timestamps above are taken from the SSH journal evidence collected during the lab.

---

## 4. Evidence Collected

### 4.1 Network Evidence

The Ubuntu Server was configured with:

`192.168.56.10/24`

The Kali Linux test workstation was configured with:

`192.168.56.20/24`

Connectivity between the two systems was verified using ICMP ping.

### 4.2 SSH Service Evidence

The SSH service was verified as operational:

```text
Active: active (running)
```

The SSH daemon was running with the `sshd` process and listening for SSH connections on TCP port 22.

### 4.3 Successful Authentication

The SSH authentication logs recorded successful authentication for the `lexa` account from the Kali workstation:

```text
Accepted password for lexa from 192.168.56.20
```

### 4.4 Failed Authentication

Multiple failed SSH password authentication events were observed:

```text
Failed password for lexa from 192.168.56.20
```

Three failed SSH authentication events were identified during the investigation.

All three observed failures originated from `192.168.56.20`.

### 4.5 Privileged Activity

Sudo activity was reviewed through `/var/log/auth.log`.

The `lexa` account was observed executing commands through `sudo`, including:

```text
COMMAND=/usr/bin/whoami
```

The command returned:

```text
root
```

The observed privileged commands were generated during the lab and investigation process. Therefore, this activity is classified as authorized analyst activity rather than malicious privilege escalation.

---

## 5. Investigation Findings

1. The Ubuntu Server was reachable from the Kali test workstation over the isolated SOC-LAB network.
2. SSH was operational and listening on TCP port 22.
3. Successful SSH authentication events were observed for the `lexa` account.
4. Three failed SSH password authentication events were identified.
5. All observed failed authentication attempts originated from `192.168.56.20`.
6. `192.168.56.20` belongs to the Kali Linux test workstation used in this controlled lab.
7. The failed authentication activity was intentionally generated as part of the security monitoring exercise.
8. The authentication logs did not provide evidence of a successful compromise following the observed failed attempts.
9. Privileged commands observed during the investigation were analyst-generated commands executed through `sudo`.

---

## 6. Authentication Analysis

### 6.1 Failed Authentication

The failed events indicate repeated password authentication failures against the `lexa` account.

From a SOC monitoring perspective, repeated authentication failures from the same source IP are potentially suspicious and should be investigated.

### 6.2 Source Analysis

The source IP was:

`192.168.56.20`

This corresponds to the Kali Linux workstation in the lab.

### 6.3 Target Analysis

The targeted account was:

`lexa`

The authentication attempts targeted the SSH service on the Ubuntu Server.

### 6.4 Compromise Assessment

The observed failed authentication events were intentionally generated in the controlled lab.

No evidence from this exercise establishes that these failed attempts resulted in unauthorized access or account compromise.

---

## 7. Analyst Assessment

**Severity:** Low

**Classification:** Controlled Security Testing

**Status:** Closed

The observed failed SSH authentication events represent suspicious authentication behavior from a monitoring perspective.

However, the source was the authorized Kali test workstation and the events were intentionally generated during the laboratory exercise.

Therefore, the activity is classified as:

**Benign / Authorized Security Testing**

rather than a confirmed real-world compromise.

---

## 8. SOC Investigation Workflow

The investigation followed a basic SOC Analyst L1 workflow:

```text
Detect
  ↓
Validate
  ↓
Identify Source
  ↓
Analyze Logs
  ↓
Determine Impact
  ↓
Classify
  ↓
Document
```

### Detection

Failed SSH authentication events were identified in the Ubuntu SSH logs.

### Validation

The SSH service status and authentication logs were reviewed to confirm the events.

### Source Identification

The source IP was identified as:

`192.168.56.20`

### Account Identification

The targeted account was:

`lexa`

### Log Analysis

The following Linux security data sources were used:

- `/var/log/auth.log`
- `journalctl -u ssh`
- SSH service status
- TCP port 22 listening information

### Impact Assessment

No unauthorized modification, availability impact, or confirmed account compromise was identified during the exercise.

### Classification

The activity was classified as controlled and authorized security testing because the source system was the lab's Kali workstation.

### Closure

The incident was documented and closed after reviewing the available authentication and privilege-related evidence.

---

## 9. MITRE ATT&CK Mapping

### T1110 — Brute Force

The repeated failed password authentication attempts are mapped to:

**MITRE ATT&CK T1110 — Brute Force**

The mapping is used in the context of this controlled laboratory simulation.

The observed activity represents repeated password authentication failures rather than a confirmed compromise.

---

## 10. Impact Assessment

| Security Area | Assessment |
|---|---|
| Confidentiality | No confirmed impact |
| Integrity | No confirmed unauthorized modification |
| Availability | No impact observed |
| Authentication | Multiple failed attempts observed |
| Account Compromise | Not confirmed |
| Privilege Escalation | Not observed; sudo activity was authorized |
| Overall Impact | Low |

---

## 11. Recommended Response Actions

In a production environment, a SOC analyst could recommend:

1. Monitor the source IP for additional authentication attempts.
2. Review successful SSH logins associated with the targeted account.
3. Verify whether the targeted account is authorized to use SSH.
4. Enforce strong password policies.
5. Prefer SSH key-based authentication where appropriate.
6. Disable direct SSH access for unnecessary accounts.
7. Implement authentication rate limiting or lockout controls where appropriate.
8. Centralize SSH authentication logs in a SIEM.
9. Configure alerts for repeated authentication failures.
10. Investigate any successful login occurring after repeated authentication failures.

---

## 12. Evidence Screenshots

Relevant evidence is stored under:

```text
screenshots/
```

Key evidence includes:

- Ubuntu network configuration
- Kali network configuration
- Kali-to-Ubuntu connectivity
- SSH service status
- SSH port 22 listening
- Successful SSH login
- Successful SSH authentication logs
- Failed SSH authentication logs
- SSH event correlation
- SSH event timeline
- SSH service final verification

---

## 13. Final Conclusion

The lab successfully demonstrated an end-to-end SOC Analyst L1 workflow for investigating Linux SSH authentication activity.

The investigation covered:

- Network verification
- SSH service verification
- SSH port monitoring
- Successful authentication analysis
- Failed authentication detection
- Source IP identification
- Authentication log analysis
- Event correlation
- Privileged activity review
- Incident classification
- MITRE ATT&CK mapping
- Incident documentation

The investigation identified three failed SSH authentication events originating from the Kali test workstation at `192.168.56.20` against the `lexa` account.

Because the activity was intentionally generated within an isolated laboratory environment, it was classified as authorized security testing and no confirmed compromise was established.

---

## 14. Future Enhancement

The next stage of this project can extend the lab into a more advanced SOC environment by implementing:

- Centralized SIEM monitoring
- Automated SSH detection rules
- Security alerts
- Dashboards
- Threat intelligence enrichment
- Detection engineering
- Automated incident response

This will form the foundation for the Level 3 advanced SOC implementation.
