# SOC Home Lab — Network Architecture

## 1. Overview

This document describes the network architecture used for the **Linux SSH Security Monitoring & Incident Investigation Lab**.

The lab uses Oracle VirtualBox and an isolated Internal Network named `SOC-LAB`. Kali Linux generates controlled SSH authentication activity against an Ubuntu Server, and the analyst investigates the resulting network and authentication logs.

## 2. Lab Architecture

```text
                         HOST WINDOWS
                              |
                       Oracle VirtualBox
                              |
                       Internal Network
                           SOC-LAB
                              |
              +---------------+---------------+
              |                               |
        KALI-ATTACKER                    UBUNTU-SERVER
        192.168.56.20                   192.168.56.10
              |                               |
              |         SSH TCP/22            |
              +------------------------------>|
                                              |
                                      OpenSSH Server
                                              |
                                  +-----------+-----------+
                                  |                       |
                              journalctl              auth.log
                                  |                       |
                                  +-----------+-----------+
                                              |
                                      SOC Investigation
                                              |
                           +------------------+------------------+
                           |                  |                  |
                       Source IP         Event Timeline      Detection
                       Analysis          Correlation         Analysis
```

## 3. Virtual Machines

| System | Role | Operating System | SOC-LAB IP |
|---|---|---|---|
| Kali-Attacker | Test workstation / attack simulation | Kali Linux | `192.168.56.20` |
| Ubuntu-Server | Monitored endpoint / SSH server | Ubuntu 26.04 LTS | `192.168.56.10` |

The Windows VM belongs to the broader home-lab environment but is not required for this SSH investigation.

## 4. Network Design

```text
Network Name: SOC-LAB
Subnet:       192.168.56.0/24
```

### Ubuntu Server

```text
Interface: enp0s8
IP:        192.168.56.10/24
Role:      Monitored SSH endpoint
```

### Kali Linux

```text
Interface: eth1
IP:        192.168.56.20/24
Role:      Security testing workstation
```

## 5. Network Segmentation

The VMs use separate network purposes.

**NAT:** provides Internet connectivity for package installation, updates, and other administrative tasks.

**SOC-LAB Internal Network:** provides isolated communication between the security-testing workstation and monitored endpoint.

```text
Internet
   |
VirtualBox NAT
   |
VM
   |
SOC-LAB Internal Network
   |
+-- Kali-Attacker
|
+-- Ubuntu-Server
```

## 6. SSH Communication Flow

```text
Kali-Attacker
192.168.56.20
       |
       | TCP/22
       | SSH authentication
       v
Ubuntu-Server
192.168.56.10
       |
       v
OpenSSH
       |
       +--> Successful authentication logs
       |
       +--> Failed authentication logs
       |
       +--> Session events
       |
       +--> Privileged activity
```

Example:

```bash
ssh lexa@192.168.56.10
```

## 7. Log Sources

### SSH service journal

```bash
sudo journalctl -u ssh --no-pager
```

Used to investigate successful and failed SSH authentication and SSH session activity.

### Authentication log

```bash
sudo grep -a "sshd" /var/log/auth.log
```

Used to investigate authentication events, source IP addresses, user activity, and sudo activity.

## 8. Source and Destination Identification

For an event such as:

```text
Failed password for lexa from 192.168.56.20
```

| Field | Value |
|---|---|
| Source IP | `192.168.56.20` |
| Source system | Kali-Attacker |
| Destination IP | `192.168.56.10` |
| Destination system | Ubuntu-Server |
| Protocol | SSH |
| Destination Port | `22` |
| Target Account | `lexa` |

The source IP represents the origin of the connection within this controlled lab.

## 9. Connectivity Verification

From Kali:

```bash
ping -c 4 192.168.56.10
```

The route was also validated:

```bash
ip route get 192.168.56.10
```

Expected relationship:

```text
192.168.56.10 dev eth1 src 192.168.56.20
```

This confirms that Kali uses `192.168.56.20` as the source address for communication with Ubuntu over `SOC-LAB`.

## 10. Security Investigation Workflow

```text
Network Event
     |
     v
Identify Source IP
     |
     v
Identify Destination
     |
     v
Identify Service/Port
     |
     v
Review Authentication Logs
     |
     v
Correlate Events
     |
     v
Build Timeline
     |
     v
Check for Successful Login
     |
     v
Review Post-Authentication Activity
     |
     v
Determine Incident Classification
     |
     v
Document Findings
```

## 11. Incident Context

During the controlled lab exercise, the Kali workstation generated failed SSH authentication attempts against the Ubuntu Server.

```text
Source:      192.168.56.20
Destination: 192.168.56.10
Service:     SSH
Port:        22
Account:     lexa
```

Three actual failed SSH authentication events were identified during the investigation.

The activity was intentionally generated as part of the authorized SOC security-testing exercise. Therefore, the source IP represents the controlled test workstation rather than a real-world malicious attacker.

## 12. Security Boundaries

This project is an isolated security laboratory.

Testing is restricted to:

```text
192.168.56.0/24
```

The SSH authentication attempts documented in this project were authorized and generated for defensive monitoring and investigation practice. No third-party systems are targeted.

## 13. Architecture-to-SOC Mapping

| Architecture Component | SOC Function |
|---|---|
| Kali-Attacker | Generates controlled security events |
| Ubuntu-Server | Generates security telemetry |
| SOC-LAB | Isolated investigation network |
| OpenSSH | Monitored service |
| `journalctl` | Event investigation |
| `/var/log/auth.log` | Authentication telemetry |
| Source IP | Investigation indicator |
| Event timeline | Correlation |
| Detection logic | Alerting concept |
| Incident report | Analyst documentation |

## 14. Future Architecture Enhancement

The current project intentionally focuses on **manual SOC investigation** rather than claiming that a SIEM has been deployed.

Possible future enhancements include:

- Centralized log collection
- Wazuh SIEM integration
- Automated SSH detection rules
- Security dashboards
- Alert severity classification
- File Integrity Monitoring
- Threat intelligence enrichment
- Automated response
- Multiple monitored endpoints

These are future enhancements and are **not represented as implemented features of the current project**.

## 15. Conclusion

The architecture provides a controlled environment for practicing a realistic SOC workflow:

**Network communication → SSH authentication → log generation → event correlation → investigation → detection → incident documentation**

The separation between the Kali testing workstation and Ubuntu monitored endpoint makes it possible to clearly identify the source, destination, service, authentication events, and investigation evidence.
